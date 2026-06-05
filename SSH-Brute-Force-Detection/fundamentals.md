# 🔐 SSH Brute Force Detection — Fundamentals

> **Project:** SSH Brute Force Detection | **Course:** MSCi Cyber Security — Year 1
> 

> This page contains all the foundational theory you need before building your SSH brute force detector. Each chapter builds on the last — read them in order.
> 

---

# 📡 Chapter 1: Networking Basics

> **Why this matters first:** SSH is a network protocol. Without understanding how computers communicate, the attack and detection concepts won't click.
> 

### Key Concepts

**IP Addresses** — Unique identifiers for every device on a network, like a postal address.

- Example: `192.168.1.5`
- In your lab: your Kali attack machine and Ubuntu target machine each have their own IP

**Ports** — Numbered "doors" on a single machine (up to 65,535).

- Different services listen on different ports
- ⚠️ **Port 22** = where SSH lives
- When attackers scan for vulnerable machines, they're looking for open port 22

**TCP (Transmission Control Protocol)** — The transport protocol SSH uses.

- Before data flows, TCP performs a **3-way handshake**: `SYN → SYN-ACK → ACK`
- Every SSH connection starts this way — and every one gets logged

---

# 🖥️ Chapter 2: SSH Explained

> SSH (Secure Shell) lets you remotely control another machine over a network — fully encrypted. It replaced Telnet, which sent passwords in plain text that anyone on the network could intercept.
> 

### Two Authentication Methods

- 🔑 Password Authentication
    
    You type a username and password. Simple, but:
    
    - Every attempt generates a log entry
    - No limit on how many times an attacker can try (unless configured)
    - **This is what your project targets**
- 🗝️ Key-Based Authentication
    
    A cryptographic key pair system:
    
    - The server stores your **public key**
    - You prove ownership with your **private key** locally
    - Practically impossible to brute force
    - Real sysadmins disable password auth entirely

> [!warning]
> 

> **Important config file:** `/etc/ssh/sshd_config`
> 

> For your lab, leave `PasswordAuthentication yes` on the target machine to make it intentionally vulnerable.
> 

---

# ⚔️ Chapter 3: Brute Force Attacks

A brute force attack **systematically tries credentials until one works**. There are four main variants:

- 1️⃣ Pure Brute Force
    
    Tries every possible character combination. Impractical on long passwords — the number of combinations grows exponentially.
    
- 2️⃣ Dictionary Attack
    
    Tries a list of real-world common passwords.
    
    🔥 **The rockyou.txt wordlist** — 14 million leaked real-world passwords, ships with Kali Linux. Makes this approach frighteningly effective.
    
- 3️⃣ Credential Stuffing
    
    Uses username/password pairs from **previous data breaches**. Effective because people reuse passwords across multiple services.
    
- 4️⃣ Password Spray
    
    Tries **one common password across many usernames**. Designed to stay under per-account lockout thresholds and avoid detection.
    

### 🛠️ The Tool You'll Use: Hydra

```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt -t 4 ssh://192.168.1.100
```

> [!tip]
> 

> Each failed attempt in Hydra = one log entry on the target. Run this for 60 seconds and you'll have hundreds of evidence entries in `auth.log` — exactly what you need for Splunk.
> 

---

# 📋 Chapter 4: Linux Logs

Every SSH attempt writes to:

- **Ubuntu/Debian:** `/var/log/auth.log`
- **CentOS/RHEL:** `/var/log/secure`

### Reading a Log Entry

```
Jun 15 14:23:01 webserver sshd[12345]: Failed password for root from 10.0.0.50 port 54231 ssh2
Jun 15 14:23:02 webserver sshd[12346]: Failed password for invalid user admin from 10.0.0.50 port 54232 ssh2
```

| Field | Meaning |
| --- | --- |
| `Jun 15 14:23:01` | Timestamp |
| `webserver` | Hostname |
| `sshd[12345]` | Process name + ID |
| `Failed password` | Event type |
| `root` | Username attempted |
| `10.0.0.50` | Source IP (attacker) |
| `port 54231` | Source port |

> `"Failed password for invalid user"` is especially telling — it means the attacker is **guessing usernames too**.
> 

### Successful Login Log

```
Jun 15 14:25:10 webserver sshd[12400]: Accepted password for john from 192.168.1.5 port 49201 ssh2
```

### Manual Analysis Commands

```bash
# Watch logs live
tail -f /var/log/auth.log

# Count failures grouped by source IP
grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn
```

> Understanding this pipeline helps you understand what your SPL query is doing in Splunk — they're solving the same problem, just differently.
> 

---

# 🔍 Chapter 5: Splunk Fundamentals

### The Three Components

| Component | Role |
| --- | --- |
| **Universal Forwarder** | Installed on your Linux target VM. Watches `auth.log` and ships new lines to Splunk in real time |
| **Indexer** | Receives data, parses fields (timestamp, host, message), stores in a searchable index |
| **Search Head** | The web UI at `http://localhost:8000` — where you write queries, build dashboards, and set alerts |

### SPL (Search Processing Language)

SPL works like a **pipeline** — each command feeds into the next:

```
index=main sourcetype=linux_secure "Failed password"
| rex "from (?P<src_ip>\d+\.\d+\.\d+\.\d+)"
| stats count by src_ip
| sort -count
| where count > 10
```

- 📖 Breaking down the query step by step
    1. `index=main sourcetype=linux_secure "Failed password"` — find all logs matching the phrase
    2. `| rex "from (?P<src_ip>...)"` — extract the source IP using regex
    3. `| stats count by src_ip` — count occurrences grouped by IP
    4. `| sort -count` — sort highest failure count first
    5. `| where count > 10` — filter to only IPs with more than 10 failures

> **Alerts** in Splunk are saved searches that run on a schedule. If results exceed a threshold (e.g. 20+ failures in 5 minutes), Splunk fires an action — email, webhook, or script.
> 

---

# 🚨 Chapter 6: Detection Logic

> [!warning]
> 

> **Key insight:** You're not detecting individual failed logins — everyone mistypes a password. You're detecting **patterns**.
> 

### Detection Signals — Ranked by Severity

| Severity | Signal | Reason |
| --- | --- | --- |
| 🔴 **High** | 20+ failed logins from a single IP in under 60 seconds | A real user doesn't fail 20 times a minute |
| 🔴 **High** | Multiple distinct usernames from the same IP | Real users know their own username |
| 🔴 **Critical** | Many failures → successful login from same IP | The attack worked |
| 🟡 **Medium** | Root login at 3am | Suspicious even without prior failures |

### The Full Detection Query

```
index=main sourcetype=linux_secure ("Failed password" OR "Invalid user")
| rex field=_raw "from (?P<src_ip>\d+\.\d+\.\d+\.\d+)"
| rex field=_raw "for (?:invalid user )?(?P<username>\w+)"
| bucket _time span=1m
| stats count as failed_attempts dc(username) as unique_users values(username) as tried_users by _time, src_ip
| where failed_attempts > 20 OR unique_users > 5
| eval severity=if(failed_attempts>100,"critical","high")
| sort -_time
```

- 📖 Key SPL functions explained
    - `bucket _time span=1m` — groups events into 1-minute windows for rate analysis
    - `dc(username)` — counts **distinct** usernames attempted (dc = distinct count)
    - `eval` — creates a calculated field; here it assigns a severity label based on volume

### Dashboard Panels to Build

1. **📈 Timeline Chart** — failed logins over time (visually see the attack spike)
2. **🗺️ Geo Map** — source IPs plotted by geographic location
3. **🏆 Top Attackers Table** — IPs ranked by failure count + severity label
4. **🔔 Alert Log** — list of triggered alerts with timestamps

---
