# Beaconing Traffic Detection — Fundamentals

# 📌 Overview

This page covers the **new fundamentals** introduced in the Beaconing Traffic Detection Lab. Prior knowledge from the SSH Brute Force, Port Scan Detection, Reverse Shell Detection, Custom Log-Based IDS, and SOC Investigation Simulation labs is assumed.

> 💡 **Key Concept:** This lab shifts detection from counting events to measuring **time regularity**. The question is no longer "how many?" but "how consistent are the intervals?"
> 

---

# ✅ What You Already Know

| Tool / Concept | Covered In |
| --- | --- |
| VirtualBox / Kali / Ubuntu network setup | SSH Brute Force Lab |
| Netcat — listeners, connect mode, flags | Reverse Shell Detection Lab |
| Wireshark — packet capture and filtering | Reverse Shell Detection Lab |
| Splunk — log ingestion, basic SPL, indexes | SSH Brute Force and Port Scan Detection Labs |
| C2 concept — attacker listener, victim callback | Reverse Shell Detection Lab |
| MITRE ATT&CK — mapping techniques to tactics | All previous labs |

---

# 🆕 New Fundamentals

## 1. Beaconing as a C2 Mechanism

A reverse shell is a **one-time event** — a connection fires, a session opens. Beaconing is different. It is a **repeating, periodic callback** — malware on a victim machine checks in with a C2 server at a regular interval, even when it has nothing to do. Think of it like a heartbeat.

| Property | Reverse Shell | Beaconing |
| --- | --- | --- |
| Connection type | One-time, stays open | Repeated short check-ins |
| Session duration | Interactive, persistent | Closes after each beacon |
| Detection signal | Unusual outbound connection | Suspiciously regular intervals |
| Attacker intent | Immediate interaction | Maintain presence, issue commands later |

> 🗂️ **MITRE ATT&CK:** T1071 (Application Layer Protocol) — adversaries use standard protocols to blend beaconing traffic into normal network noise.
> 
- 💭 Why do attackers prefer beaconing over a persistent shell?
    
    A persistent shell is noisy — it keeps a connection open indefinitely, which is easy to spot in network monitoring. Beaconing lets an attacker go dormant between check-ins. Each connection lasts only a second or two, making the traffic blend into background noise. The attacker can issue commands on the next check-in without needing to maintain a live session the whole time.
    

---

## 2. Beacon Intervals and Jitter

The key detection signal is **regularity**. If a host connects to the same IP every ~30 seconds, that is suspicious — humans do not behave that way, machines do.

Attackers are aware of this. They introduce **jitter** — a small random variation added to each interval. Instead of exactly 30 seconds every time, it might be 27s, 33s, 29s, 61s.

> ⚠️ Detection logic cannot just look for *exact* intervals — it must look for **low variance** in connection timing, even when intervals are not perfectly identical.
> 
- 💭 What does jitter look like in practice?
    
    **No jitter — easy to detect:**
    
    ```jsx
    Connection 1:  t = 0s
    Connection 2:  t = 30s
    Connection 3:  t = 60s
    Connection 4:  t = 90s
    Standard deviation = 0
    ```
    
    **With jitter — harder to detect:**
    
    ```jsx
    Connection 1:  t = 0s
    Connection 2:  t = 33s
    Connection 3:  t = 61s
    Connection 4:  t = 88s
    Standard deviation ≈ 1.5
    ```
    
    In the lab you add jitter using bash's built-in `$RANDOM` variable:
    
    ```bash
    sleep $((30 + RANDOM % 10))   # random interval between 30–39 seconds
    ```
    

---

## 3. Time-Delta Analysis in Splunk

This is the **core new detection skill** in this lab. Previous labs detected events by count and threshold (e.g. >5 failed SSH attempts). Here, detection is **time-based** — you calculate the gap between successive connections and look for suspicious regularity.

### New SPL Commands

| Command | What it does |
| --- | --- |
| `streamstats` | Computes running statistics across a stream of events — used here to retrieve the previous event's timestamp per source IP |
| `eval` | Creates a calculated field — used here to subtract timestamps and produce a time delta |
| `stats stdev()` | Calculates standard deviation — low stdev means high regularity, which is the beaconing signal |
| `where` | Filters aggregated results — used here to flag IPs whose jitter falls below the threshold |

### The Full Detection Query

```
index=beaconing_detection tcp_dstport=4444
| sort ip_src _time
| streamstats last(_time) as prev_time by ip_src
| eval time_gap = _time - prev_time
| stats stdev(time_gap) as jitter by ip_src
| where jitter < 5
```

- 💭 What does each line actually do?
    
    
    | Line | Purpose |
    | --- | --- |
    | `index=beaconing_detection tcp_dstport=4444` | Filter to only connections on port 4444 |
    | `sort ip_src _time` | Order events chronologically per source IP — required before streamstats |
    | `streamstats last(_time) as prev_time by ip_src` | For each event, grab the timestamp of the previous connection from the same IP |
    | `eval time_gap = _time - prev_time` | Calculate the gap in seconds between successive connections |
    | `stats stdev(time_gap) as jitter by ip_src` | Measure how consistent those gaps are — low variance = suspicious |
    | `where jitter < 5` | Flag hosts whose connection intervals are suspiciously regular |

> 🔍 **Field name gotcha:** Splunk converts dots to underscores when ingesting CSV files. So `ip.src` becomes `ip_src`, `tcp.dstport` becomes `tcp_dstport`, and so on. Always verify field names by expanding an event before writing your query.
> 

---

## 4. Simulating Beaconing with a Bash Loop

Rather than a one-shot Netcat command, beaconing is simulated using a `while` loop in bash with a `sleep` delay — mimicking what real malware does.

### Basic Beaconing Script (No Jitter)

```bash
#!/bin/bash
while true
do
    nc -w 1 192.168.102.4 4444
    sleep 30
done
```

### Extended Script (With Jitter)

```bash
#!/bin/bash
while true
do
    nc -w 1 192.168.102.4 4444
    sleep $((30 + RANDOM % 10))
done
```

| Component | Purpose |
| --- | --- |
| `#!/bin/bash` | Shebang — tells the OS to use the bash interpreter at `/bin/bash` |
| `while true` | Loops forever — `true` always evaluates to exit code 0 |
| `nc -w 1` | Closes the connection after 1 second — prevents Netcat hanging and blocking the loop |
| `sleep 30` | Pauses for 30 seconds between beacons |
| `$RANDOM` | Built-in bash variable — generates a random number between 0 and 32767, no import needed |
- 💭 Why does -w 1 matter so much here?
    
    Without `-w 1`, Netcat connects to the listener and then just waits — it holds the connection open with nothing to send or receive. This means `sleep 30` is never reached, and the loop never repeats. The `-w 1` timeout forces Netcat to close the connection after one second of inactivity, allowing execution to continue to `sleep 30` and then loop again. This is what creates the periodic beaconing rhythm.
    

---

## 5. Lab Architecture

| Machine | IP Address | Role | Key Command |
| --- | --- | --- | --- |
| Kali Linux | 192.168.102.4 | C2 Listener | `nc -l -k -v -p 4444` |
| Ubuntu | 192.168.102.3 | Victim | `./beacon.sh` |
| Windows Host | 192.168.102.1 | Splunk / Analysis | SPL detection query |

> 🎯 **Startup order matters:** Start the Kali listener first, then Wireshark, then the Ubuntu beaconing script. If the script runs before the listener is ready, the first beacon will fail and the timing evidence will be incomplete.
> 

---

## 6. Toolchain and Data Pipeline

| Tool | Purpose in This Lab |
| --- | --- |
| **Netcat** | C2 listener on Kali (`-l -k -v`); periodic connect-back on Ubuntu |
| **Bash** | Beaconing loop script using `while true` and `sleep` |
| **Wireshark** | Captures periodic TCP SYN packets on port 4444 |
| **tshark** | Converts the pcap to CSV for Splunk ingestion |
| **Python HTTP server** | Serves the CSV from Kali so Windows can download it (`python3 -m http.server 8000`) |
| **Splunk** | Time-delta analysis and beaconing detection via SPL |

### tshark Conversion Command

```bash
tshark -r beacon.pcap -T fields -e frame.time -e ip.src -e ip.dst -e tcp.dstport -E header=y -E separator=, > beacon.csv
```

---

# 🧠 Summary — New Concepts at a Glance

| Concept | Why It Matters |
| --- | --- |
| Beaconing as a C2 mechanism | Enables attacker persistence without a live session — harder to spot than a reverse shell |
| Beacon intervals and jitter | Regularity is the detection signal; attackers add jitter to evade time-based rules |
| `streamstats` in SPL | Enables running calculations across ordered event streams — essential for time-delta logic |
| `stdev()` in SPL | Low standard deviation = suspicious regularity = likely automated beaconing |
| Bash loop simulation | Mimics real malware behaviour using `while true`, `nc -w 1`, and `sleep` |
| Splunk field name conversion | Dots become underscores in CSV-ingested fields — always verify before querying |
