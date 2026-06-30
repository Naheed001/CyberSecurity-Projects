# Custom Log-Based Detection Script — Fundamentals

# 📌 Overview

This page covers the **new fundamentals** introduced in the Custom Log-Based Intrusion Detection Script lab. Prior knowledge from the previous labs is assumed.

---

# ✅ What You Already Know

| Tool / Concept | Covered In |
| --- | --- |
| Basic Splunk ingestion and SPL searching | SSH Brute Force and Port Scan Labs and End-to-End SOC Investigation Simulation |
| Linux log locations and structure | All previous labs |
| Hydra — generating brute force traffic | SSH Brute Force Detection Lab and End-to-End SOC Investigation Simulation |
| MITRE ATT&CK — mapping techniques to tactics | All previous labs |
| General SOC detection workflow | End-to-End SOC Investigation Simulation |

---

# 🆕 New Fundamentals

## 1. Python Log Parsing

Reading a log file in Python means opening it and processing it **line by line**. The key concept is streaming reads — iterating rather than loading the entire file into memory at once.

```python
with open('/var/log/auth.log', 'r') as f:
    for line in f:
        # process each line
```

The `with` block handles closing the file safely — even if an error occurs. Think of it as reading a book one page at a time rather than memorising the whole thing first.

- 💭 Why stream line by line instead of loading the whole file?
    
    In production environments, log files can be **gigabytes** in size. Loading an entire file into memory would crash or severely slow down a script. Line-by-line streaming processes each entry and discards it, keeping memory usage constant regardless of file size.
    

---

## 2. Regular Expressions (Regex)

A log line contains structured information embedded in a plain text string. Regex lets you **extract specific parts using patterns** — like pulling an IP address out of a noisy line.

### Key Regex Patterns

| Pattern | Meaning |
| --- | --- |
| `\d+` | One or more digits |
| `(\d+\.\d+\.\d+\.\d+)` | Capture group matching an IPv4 address |
| `re.search()` | Find a match anywhere in a string |
| `match.group(0)` | The entire match including literal text |
| `match.group(1)` | First capture group only — what's inside `()` |

### Kali-Specific Log Format

> ⚠️ **Important:** Kali Linux uses `systemd-journald` instead of traditional `auth.log`. SSH logs must be exported via `sudo journalctl -u ssh`. The IP regex pattern differs from standard references.
> 

Kali's log lines use `rhost=` instead of `from`:

```
sshd-session[478538]: pam_unix(sshd:auth): authentication failure; rhost=127.0.0.1 user=kali
```

So the correct regex pattern is:

```python
import re

match = re.search(r'rhost=(\d+\.\d+\.\d+\.\d+)', line)
if match:
    ip = match.group(1)  # → "127.0.0.1"
```

- 💭 Why use group(1) and not group(0)?
    
    `group(0)` returns the **entire match** including the literal prefix — e.g. `rhost=127.0.0.1`. `group(1)` returns only the **first capture group** (what's inside the parentheses) — e.g. `127.0.0.1`. We use `group(1)` because we only want the IP address itself, not the `rhost=` text attached to it.
    

---

## 3. Threshold-Based Detection Logic

Once IPs are extracted, count failures per IP using a **dictionary** and fire an alert when a threshold is crossed.

```python
failed_attempts = {}

failed_attempts[ip] = failed_attempts.get(ip, 0) + 1

if failed_attempts[ip] >= THRESHOLD:
    # fire alert
```

### Dual Severity Levels

| Attempts | Severity |
| --- | --- |
| 5 – 9 | MEDIUM |
| 10+ | HIGH |

```python
if failed_attempts[ip] >= THRESHOLD and ip not in alerted_ips:
    severity = "HIGH"
elif failed_attempts[ip] >= 5 and failed_attempts[ip] < THRESHOLD and ip not in alerted_ips:
    severity = "MEDIUM"
```

- 💭 Why >= and not >?
    
    Using `>` instead of `>=` means an IP with **exactly 10 attempts skips the HIGH threshold entirely** and falls through incorrectly. This is a boundary condition bug — small logic errors like `>` vs `>=` can silently skew detection. Always test at the exact threshold value to verify behaviour.
    

---

## 4. Duplicate Alert Prevention with Sets

Without deduplication, every attempt **after** the threshold fires a new alert — flooding analysts with thousands of duplicate events for the same IP. This is called **alert fatigue**.

A Python **set** solves this. Unlike a list, a set stores only unique values and checks membership far faster.

```python
alerted_ips = set()

if failed_attempts[ip] >= THRESHOLD and ip not in alerted_ips:
    alerted_ips.add(ip)  # Mark this IP as already alerted
    # fire alert once
```

- 💭 Why use a set instead of a list?
    
    To check if an IP is in a list, Python scans every element one by one — slow at scale. A set uses hashing, so membership checks are near-instant regardless of how many IPs it contains. In a real IDS processing thousands of IPs, this difference is significant.
    

---

## 5. Structured JSON Alerts

Rather than printing a plain text message, a SIEM expects **structured data** it can index and query. JSON is the standard format.

```python
from datetime import datetime
import json

alert = {
    "timestamp": datetime.now().strftime("%Y-%m-%dT%H:%M:%S"),
    "alert_type": "brute_force",
    "source_ip": ip,
    "failed_attempts": failed_attempts[ip],
    "severity": "HIGH"
}
```

Built as a Python `dict` first, then forwarded directly as JSON via the `requests` library.

- 💭 Why structured JSON instead of plain text?
    
    Plain text like `"Alert: brute force from 127.0.0.1"` can't be queried, filtered, or aggregated in a SIEM. JSON gives every field a name — Splunk can then search `source_ip="127.0.0.1"`, aggregate by `severity`, or chart `failed_attempts` over time. Structure is what turns a log line into actionable intelligence.
    

---

## 6. Splunk HTTP Event Collector (HEC)

HEC lets your script **push alerts into Splunk programmatically** via HTTP POST — rather than Splunk watching a log file passively.

```
POST http://<splunk_ip>:8088/services/collector/event
Authorization: Splunk <your_HEC_token>
Content-Type: application/json

{"event": { ...your alert... }}
```

### Python Implementation

```python
import requests

headers = {"Authorization": f"Splunk {SPLUNK_TOKEN}"}
payload = {"event": alert}
requests.post(SPLUNK_URL, headers=headers, json=payload)
```

- 💭 Why does SSL being enabled break the connection?
    
    Our script sends to `http://` (unencrypted). If Splunk HEC has SSL enabled, it expects `https://` (encrypted). The protocol mismatch causes the connection to be refused. In a production environment you would use SSL with a proper certificate — in a lab, disabling it removes the certificate complexity.
    

---

# 🌐 Network Architecture

| Host | IP | Role |
| --- | --- | --- |
| Windows Host | `192.168.102.1` | Splunk SIEM |
| Kali Linux VM | `192.168.102.4` | IDS Script |
| HEC Port | `8088` | Alert ingestion endpoint |

---

# 🔁 The Full IDS Pipeline

```
Hydra Brute Force Attack
         ↓
systemd-journald (SSH logs)
         ↓
journalctl → exported to auth.log
         ↓
ids.py — Python Script
  (regex → count → threshold → dedup)
         ↓
JSON Alert
         ↓
Splunk HEC (port 8088)
         ↓
Indexed & Searchable in Splunk ✅
```

---

# 📦 Libraries Used

| Library | Purpose | Built-in? |
| --- | --- | --- |
| `re` | Regular expressions | ✅ Yes |
| `json` | JSON serialisation | ✅ Yes |
| `datetime` | Timestamps | ✅ Yes |
| `requests` | HTTP POST to Splunk HEC | ❌ `sudo apt install python3-requests` |

---

# 🧠 Summary — New Concepts at a Glance

| Concept | Why It Matters |
| --- | --- |
| Python log parsing (streaming) | Memory-efficient processing of large log files |
| Regex with capture groups | Extracts structured data from unstructured log lines |
| Kali uses journald, not auth.log | Export with `journalctl` — regex pattern uses `rhost=` not `from` |
| Dictionary-based counting | Tracks failure counts per IP across all log lines |
| Sets for deduplication | Prevents alert fatigue — fires once per IP, not once per attempt |
| Structured JSON alerts | Makes alerts indexable, queryable, and actionable in Splunk |
| Splunk HEC | Programmatic alert ingestion — script pushes directly to SIEM |
| Firewall scope restriction | Defence in depth — only Kali's IP can reach port 8088 |
