# Port Scan Detection Lab — Fundamentals

This page covers all the core concepts you need for the Port Scan Detection Engineering Lab. It is structured as a reference guide you can return to at any point.

---

## 1. What is a SIEM?

A **SIEM** (Security Information and Event Management) performs four jobs in sequence:

| Job | What it does |
| --- | --- |
| Collect | Gathers logs from many sources across your environment |
| Normalise | Converts different log formats into a common structure |
| Index & Store | Makes logs fast to search across large volumes |
| Search & Alert | Lets you detect patterns and fire alerts automatically |

The real value of a SIEM is **correlation** — a single firewall log line means nothing, but "one source IP touched 900 ports in 10 seconds" only becomes visible when you can aggregate across thousands of events at once.

> 💡 Splunk is one SIEM platform. The concepts here transfer to others such as Elastic, Microsoft Sentinel, and IBM QRadar.
> 

---

## 2. The TCP Three-Way Handshake

Almost every Nmap scan type is a variation on — or deliberate violation of — the normal TCP connection process. You need this cold before anything else.

A normal TCP connection uses a **three-way handshake**:

```
Client  →  SYN      →  Server      ("I want to connect")
Server  →  SYN-ACK  →  Client      ("OK, I'm ready")
Client  →  ACK      →  Server      ("Great, let's talk")
```

The key TCP flags you need to know:

| Flag | Meaning |
| --- | --- |
| SYN | Initiate a connection |
| ACK | Acknowledge a packet |
| RST | Reset / refuse a connection |
| FIN | Finish / close a connection |
| PSH | Push data immediately |
| URG | Urgent data |

---

## 3. Port States

Nmap classifies every port it probes into one of these states:

| State | Meaning |
| --- | --- |
| **Open** | Something is listening — you received a SYN-ACK |
| **Closed** | Host is reachable but nothing listening — you received RST |
| **Filtered** | No response, or ICMP unreachable — likely a firewall |
| **Unfiltered** | Port is reachable but open/closed can't be determined |

---

## 4. Nmap Scan Types

> 🗺️ Nmap is a **network mapping tool**. Its job is to tell an attacker (or a security tester) which ports are open on a target, and therefore which services might be exploitable. Each scan type achieves this differently — by manipulating TCP flags, exploiting protocol quirks, or varying timing. Understanding how each one works is what makes your detection logic accurate.
> 
- 🔹 SYN Scan (-sS) — Half-Open / Stealth Scan
    
    **What it does:**
    
    The SYN scan is the default Nmap scan when run as root. It works by deliberately never completing the TCP three-way handshake. Instead of going all the way through SYN → SYN-ACK → ACK, Nmap tears the connection down the moment it learns what it needs.
    
    ```
    Nmap   →  SYN        →  Target    Step 1: Probe — "I want to connect"
    Target →  SYN-ACK    →  Nmap      Step 2: Target reveals port is open
    Nmap   →  RST        →  Target    Step 3: Nmap slams the door shut
    ```
    
    If the port is **closed**, the target replies with RST at Step 2 instead of SYN-ACK. If **filtered** by a firewall, there is no reply at all.
    
    **Why it's called stealth:**
    
    Because the handshake never completes, the listening application on the target (e.g. a web server or SSH daemon) never receives a completed connection and therefore never logs it. Traditional application-level logs are completely blind to a SYN scan.
    
    **Why you'd use it:**
    
    - It is fast — one probe per port, no cleanup overhead
    - It avoids leaving traces in application logs
    - It is the standard first-step reconnaissance technique used by attackers mapping a target network
    - In a lab or pentest context, it's the most efficient way to quickly identify all open ports
    
    **What it looks like in iptables logs:**
    
    You will see a burst of TCP packets with the SYN flag set, each with a different DPT (destination port). Because the connection is torn down immediately, you will also see RST packets shortly after each SYN-ACK.
    
    **Limitation:** Requires root/administrator privileges to send raw packets.
    
- 🔹 Connect Scan (-sT) — Full TCP Handshake Scan
    
    **What it does:**
    
    The Connect scan is the fallback when Nmap doesn't have root privileges. Instead of crafting raw packets, it asks the operating system to make a real TCP connection on its behalf using the standard `connect()` system call — the same call a browser or any normal application uses.
    
    ```
    Nmap   →  SYN        →  Target    Step 1: Probe
    Target →  SYN-ACK    →  Nmap      Step 2: Port is open
    Nmap   →  ACK        →  Target    Step 3: Handshake completes (full connection!)
    Nmap   →  FIN        →  Target    Step 4: Nmap closes the connection
    Target →  ACK        →  Nmap      Step 5: Target acknowledges close
    ```
    
    For a **closed** port, the target replies RST at Step 2 and no connection forms.
    
    **Why you'd use it:**
    
    - When you don't have root privileges and cannot send raw packets
    - It is reliable and works on every operating system without special configuration
    - Results are identical to a SYN scan in terms of what ports are found
    
    **The key difference from SYN scan:**
    
    Because the connection fully establishes, the application on the target **does** see and log the connection. A web server, SSH daemon, or any listening service will record that something connected and immediately disconnected. This makes it noisier and more detectable than a SYN scan.
    
    It also produces more packets per port (5–6 instead of 2–3), so your iptables logs will be flooded with ACK packets from the established connections, which can make the raw log harder to read.
    
    **What it looks like in iptables logs:**
    
    You will see a mix of SYN packets (the probes) and many ACK packets (from established connections exchanging data). The DPT values change across events as Nmap cycles through ports.
    
- 🔹 UDP Scan (-sU) — User Datagram Protocol Scan
    
    **What it does:**
    
    UDP is a completely different transport protocol from TCP. It has no handshake, no connection state, and no guaranteed delivery. Nmap probes UDP ports by sending a bare UDP packet and then waiting to see what comes back.
    
    ```
    Nmap   →  UDP packet  →  Target:port    Step 1: Probe
    
    If port is OPEN:    No response (UDP has no acknowledgement mechanism)
    If port is CLOSED:  Target sends ICMP "Port Unreachable" back to Nmap
    If FILTERED:        No response (same as open — ambiguous)
    ```
    
    This creates a problem: silence means either open or filtered. Nmap has to wait for a timeout before concluding a port is open, which is why UDP scans are extremely slow.
    
    **Why you'd use it:**
    
    - Many important services run on UDP rather than TCP — DNS (port 53), SNMP (port 161), DHCP (port 67/68), NTP (port 123), and VoIP services
    - An attacker who only scans TCP will completely miss these services
    - In a real pentest, a UDP scan is essential for a complete picture of the target's attack surface
    
    **Why it's slow:**
    
    Because there is no handshake to confirm receipt, Nmap must wait for a timeout on every non-responding port. With 1000 ports to scan and each waiting several seconds for a timeout, a full UDP scan can take hours.
    
    **What it looks like in iptables logs:**
    
    You will see UDP packets with `PROTO=UDP` and varying DPT values. Notably, unlike TCP scans, there will be no SYN, ACK, or RST flags — just the bare protocol and port information.
    
- 🔹 FIN Scan (-sF), NULL Scan (-sN), Xmas Scan (-sX) — RFC Exploit Scans
    
    **What they do:**
    
    These three scans all exploit the same quirk buried in the original TCP specification (RFC 793). The spec states:
    
    - A **closed** port **must** reply with RST to any unexpected packet
    - An **open** port **should silently ignore** packets that don't make sense in the context of an established connection
    
    Nmap exploits this by sending packets with unusual or impossible flag combinations that no legitimate connection would ever use:
    
    | Scan | Flags sent | What it sends |
    | --- | --- | --- |
    | FIN (-sF) | FIN only | "I want to close a connection" with no connection open |
    | NULL (-sN) | No flags | A completely empty TCP header |
    | Xmas (-sX) | FIN + PSH + URG | Multiple flags that should never appear together |
    
    **How to interpret the response:**
    
    - **No response** → port is likely **open** (it silently ignored the nonsensical packet)
    - **RST received** → port is **closed**
    - **No response** → could also mean **filtered** — making results ambiguous
    
    **Why you'd use them:**
    
    - They are specifically designed to slip past **simple stateless firewalls** that only block SYN packets
    - A firewall that blocks SYN (to prevent connections) may allow FIN or NULL packets through because they don't look like new connection attempts
    - They can reveal open ports that a SYN scan would show as filtered
    - Useful for understanding whether a firewall is stateless (simpler) or stateful (smarter)
    
    **Critical limitation:**
    
    These scans **do not work reliably against Windows targets**. Windows does not follow the RFC 793 behaviour — it replies RST to everything regardless of whether the port is open or closed, making all ports appear closed.
    
    **What they look like in iptables logs:**
    
    You will see TCP packets with unusual flag combinations. A FIN scan produces `FIN URGP=0`, a NULL scan produces no flags at all, and an Xmas scan produces `FIN PSH URG`. These are immediately recognisable as abnormal — no legitimate application would send packets like these.
    
- 🔹 ACK Scan (-sA) — Firewall Mapping Scan
    
    **What it does:**
    
    The ACK scan is fundamentally different from all the others — it does **not** find open ports. Instead, its purpose is to map the firewall rules protecting a target.
    
    Nmap sends TCP packets with only the ACK flag set. An ACK packet looks like it belongs to an already-established connection. Firewalls treat it differently depending on whether they are stateless or stateful:
    
    ```
    Nmap  →  ACK  →  Target
    
    Stateless firewall: Lets it through (doesn't track connection state)
    Stateful firewall:  Blocks it (knows there was no prior SYN — this ACK is orphaned)
    ```
    
    | Response from target | What it means |
    | --- | --- |
    | RST received | Port is **unfiltered** — the packet reached the host |
    | No response / ICMP unreachable | Port is **filtered** — blocked by a stateful firewall |
    
    **Why you'd use it:**
    
    - To understand the firewall architecture protecting a target before launching further attacks
    - To determine whether a firewall is stateless (simpler, easier to evade) or stateful (more sophisticated)
    - In a pentest, this helps an attacker decide which scan types will be effective against the target's defences
    - It can reveal which ports a firewall actively filters versus which it simply ignores
    
    **Important:** Because it never finds open ports, an ACK scan is always used **alongside** other scan types, not as a standalone technique.
    
    **What it looks like in iptables logs:**
    
    You will see TCP packets with `ACK URGP=0` and varying DPT values — but no SYN flag, which immediately marks them as unusual since legitimate ACK packets would only appear as part of an established connection.
    
- 🔹 Timing Options (-T0 to -T5) — Speed and Stealth Control
    
    **What they do:**
    
    Timing templates control how fast Nmap sends probes. Every scan type above can be combined with a timing flag to make it faster (noisier, easier to detect) or slower (quieter, harder to detect).
    
    | Flag | Name | Behaviour | Detection implication |
    | --- | --- | --- | --- |
    | -T0 | Paranoid | One probe every 5 minutes | Virtually undetectable by rate-based rules |
    | -T1 | Sneaky | One probe every 15 seconds | Very hard to detect with short time windows |
    | -T2 | Polite | Slowed to reduce network load | Hard to detect, scan takes minutes |
    | -T3 | Normal | Default adaptive timing | Detectable with a reasonable threshold |
    | -T4 | Aggressive | Fast, assumes a reliable network | Easily detected — very high event rate |
    | -T5 | Insane | Maximum speed, may drop packets | Immediately obvious in any log source |
    
    **Why timing matters for detection:**
    
    Your SPL query uses `dc(DPT)` across **all time** to count distinct ports. But in a real production environment, your query would run over a fixed time window (e.g. last 5 minutes). A `-T0` or `-T1` scan deliberately spreads its probes so slowly that any fixed time window would only capture a handful of ports — well under any threshold you set. This is the core challenge of detecting slow scans and why there is no perfect answer for the time window setting.
    

---

## 5. The Key Detection Pattern

> 🎯 **Regardless of scan type, the universal signature of a port scan is: one source IP producing an unusually high number of distinct destination port values in a short time window.**
> 

Two sub-patterns:

- **Vertical scan** — one source, one target, many ports
- **Horizontal scan (sweep)** — one source, many targets, same port

Your detection query focuses on the vertical scan pattern using `dc(DPT)` — distinct count of destination ports per source IP.

---

## 6. Data Sources — Why This Differs from SSH Brute Force

This is the most important conceptual difference from the SSH Brute Force lab.

| Lab | What generates the log | Where the log lives |
| --- | --- | --- |
| SSH Brute Force | Failed authentication attempts | `/var/log/auth.log` |
| Port Scan Detection | Network connection attempts | `/var/log/syslog` (via iptables) |

A SYN scan **never reaches an application and never authenticates** — so auth logs are completely blind to it. You need network-layer telemetry that captures every packet the kernel sees, regardless of whether a connection completes.

In this lab you achieve this by adding an `iptables LOG` rule:

```bash
sudo iptables -I INPUT -j LOG --log-prefix "IPTABLES: " --log-level 4
```

This writes a log line for every inbound packet to `/var/log/syslog`, which the Splunk Universal Forwarder then ships to Splunk Enterprise.

### Key fields in iptables log lines

| Field | Meaning | Used in query |
| --- | --- | --- |
| `SRC` | Source IP address | `by SRC` |
| `DST` | Destination IP address | Context / direction filter |
| `DPT` | Destination port | `dc(DPT)` |
| `SPT` | Source port | Context |
| `PROTO` | Protocol (TCP / UDP / ICMP) | Scan type filtering |

---

## 7. Splunk & SPL Fundamentals

### How Splunk organises data

- **Index** — a labelled bucket of events (e.g. `index=main`)
- **Sourcetype** — the format of the data, tells Splunk how to parse fields (e.g. `sourcetype=syslog`)
- **Source** — the file or input the data came from
- **Host** — the machine that produced the event
- **Fields** — named values extracted from events (e.g. `SRC`, `DPT`)

### The SPL pipe model

SPL reads left to right. The `|` character passes the output of one stage into the next — exactly like a Unix pipe.

```
index=main sourcetype=syslog IPTABLES
| stats dc(DPT) as unique_ports by SRC
| where unique_ports > 50
```

Read this as: filter to IPTABLES events → count distinct ports per source IP → keep only those over the threshold.

### Key SPL commands for this lab

| Command | What it does |
| --- | --- |
| `search` | Filters events (implicit first stage) |
| `stats` | Aggregates events into summary rows |
| `dc(field)` | Counts distinct unique values of a field |
| `count` | Counts total number of events |
| `by` | Groups the aggregation |
| `where` | Filters aggregated results by a condition |
| `bin` / `timechart` | Buckets events into time windows |
| `eval` | Computes new fields |
| `match(field, regex)` | Tests a field against a regex pattern |

### Regex in SPL where clauses

| Symbol | Meaning | Example |
| --- | --- | --- |
| `^` | Start of string | `^127\.` matches anything starting with 127. |
| `$` | End of string | `\.1$` matches strings ending in .1 |
| `\.` | Literal dot (escaped) | Prevents `.` matching any character |
| `NOT match()` | Exclude pattern matches | `NOT match(SRC, "^127\.")` |

---

## 8. The Final Detection Query

```
index=main sourcetype=syslog IPTABLES
| stats dc(__) as unique_ports by __
| where __ > __
    AND NOT match(__, "^127\.")
    AND SRC!="192.168.102.1"
```

**What each part does:**

- `IPTABLES` — keyword filter, only events from the iptables LOG rule
- `dc(DPT) as unique_ports` — counts how many distinct ports each source touched
- `by SRC` — one result row per source IP
- `where unique_ports > 300` — threshold to separate scanners from background noise
- `NOT match(SRC, "^127\.")` — excludes all loopback addresses
- `SRC!="192.168.102.1"` — excludes the Windows host (known false positive)

---

## 9. False Positives

A false positive is a legitimate source that triggers your detection rule despite not being a real threat.

| Source | Why it looks like a scan | How to handle it |
| --- | --- | --- |
| Loopback (127.x.x.x) | Internal DNS resolver touching many ports | Exclude via regex |
| Windows host (192.168.102.1) | Background OS traffic | Exclude by IP |
| Vulnerability scanners (Nessus, Qualys) | Authorised scanning tools | Allowlist by IP |
| NAT gateways / load balancers | Many users sharing one IP | Raise threshold |
| Monitoring / uptime tools | Polling many ports regularly | Allowlist by IP |

### Tuning levers

- **Raise the threshold** — reduces sensitivity to slow/noisy legitimate traffic
- **Narrow the time window** — tightens the detection to burst activity only
- **Allowlist known-good sources** — explicitly exclude authorised scanners
- **Direction filter** — restrict to external-to-internal traffic only

---

