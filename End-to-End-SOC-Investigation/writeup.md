# 📝 End-to-End SOC Investigation — Project Write-up

**Project:** End-to-End SOC Investigation  
**Course:** MSci Cyber Security — Lab Project  
**Tools:** VirtualBox · Kali Linux · Ubuntu Server · Splunk Enterprise · Wireshark · MITRE ATT&CK Navigator

📖 Read alongside: [🔐 Fundamentals] for theory

> **AI Disclosure:** The structure, layout and formatting of this write-up were designed with the assistance of AI. All technical work, practical work, analysis and written content are my own.

---

## 🗺️ Lab Environment

### Network Architecture

The lab runs entirely on a single Windows host using VirtualBox. A host-only network (192.168.102.x) isolates all traffic within the machine — no traffic reaches the real home network at any point.

| Machine | Role | IP Address |
|---|---|---|
| Windows Host | Runs Splunk Enterprise · gateway for host-only network | 192.168.102.1 |
| Kali Linux VM | Attacker · simulates threat actor activity | 192.168.102.4 |
| Ubuntu Server VM | Victim · target of simulated attack chain | 192.168.102.3 |

---

## 🚨 Phase 1 — Alert Triage

### Initial Alert

A Splunk alert fired indicating anomalous activity on the Ubuntu server. The alert was triggered by a threshold-based saved search detecting more than 10 failed SSH login attempts from a single IP within a 5-minute window.

**Alert Details:**

| Field | Value |
|---|---|
| Alert Name | SSH Brute Force Threshold Exceeded |
| Trigger Time | 2026-06-10 22:14:03 |
| Source IP | 192.168.102.4 |
| Destination IP | 192.168.102.3 |
| Failed Attempts | 407 |
| Severity | Critical |

**Triage Decision:** Escalate to Tier 2 investigation. Volume of failures combined with a subsequent successful login indicates a likely completed brute force attack.

**Figure 1 — Initial Splunk Alert**

> Screenshot: Splunk triggered alert showing SSH Brute Force Threshold Exceeded, source 192.168.102.4, 407 failed attempts, severity Critical

---

## 🔍 Phase 2 — Log Correlation & Evidence Gathering

### 2.1 Failed Authentication Analysis

```spl
index=main sourcetype=linux_secure "Failed password"
| rex field=_raw "from (?P<src_ip>\d+\.\d+\.\d+\.\d+)"
| stats count as failed_attempts by src_ip
| sort -failed_attempts
```

**Result:**

| src_ip | failed_attempts | Classification |
|---|---|---|
| 192.168.102.4 | 407 | 🔴 CRITICAL |
| 192.168.102.3 | 3 | 🟡 LOW |

407 failed attempts from a single source IP in a short window confirms automated attack behaviour. No legitimate user generates this volume of failures.

**Figure 2 — Failed Authentication SPL Results**

> Screenshot: Splunk table showing 407 failed attempts from 192.168.102.4

### 2.2 Successful Login Correlation

```spl
index=main sourcetype=linux_secure "Accepted password"
| rex field=_raw "for (?P<username>\w+) from (?P<src_ip>\d+\.\d+\.\d+\.\d+)"
| table _time, username, src_ip
| sort _time
```

**Result:** `Accepted password for s3rvic from 192.168.102.4 port 49318 ssh2`

This is the critical finding. The same IP that generated 407 failures subsequently achieved a successful authentication. This combination is definitive evidence of a completed brute force attack.

**Figure 3 — Successful Login Event**

> Screenshot: Splunk event showing Accepted password for s3rvic from 192.168.102.4 — confirming breach

### 2.3 Post-Login Activity Timeline

```spl
index=main sourcetype=linux_secure
| rex field=_raw "for (?P<username>\w+) from (?P<src_ip>\d+\.\d+\.\d+\.\d+)"
| where src_ip="192.168.102.4"
| table _time, _raw
| sort _time
```

Events logged after the successful login were correlated against Wireshark packet captures to reconstruct post-compromise activity. Network traffic analysis confirmed command execution and lateral movement indicators.

**Figure 4 — Post-Login Activity Timeline**

> Screenshot: Splunk timeline showing sequence of events from 192.168.102.4 after successful authentication

**Figure 5 — Wireshark Packet Capture**

> Screenshot: Wireshark capture showing SSH session from 192.168.102.4 to 192.168.102.3 following the brute force sequence

---

## 🔎 Phase 3 — Threat Hunting

### IOC Expansion

Following confirmation of the initial breach, a wider threat hunt was conducted to identify any additional indicators of compromise, lateral movement, or persistence mechanisms.

**Hunt Query — Unusual Process Execution:**

```spl
index=main sourcetype=linux_secure
| rex field=_raw "session opened for user (?P<username>\w+)"
| stats count by username
| where count > 0
```

**Hunt Query — Privilege Escalation Indicators:**

```spl
index=main sourcetype=linux_secure ("sudo" OR "su " OR "COMMAND")
| rex field=_raw "USER=(?P<target_user>\w+)"
| table _time, host, target_user, _raw
| sort _time
```

**Hunt Findings:**

| Indicator | Finding | Severity |
|---|---|---|
| Lateral movement | No evidence of movement to additional hosts | 🟢 Clear |
| Privilege escalation | No sudo commands logged post-compromise | 🟢 Clear |
| Persistence mechanism | No crontab modifications or new SSH keys added | 🟢 Clear |
| Data exfiltration | No outbound connections outside host-only network | 🟢 Clear |

The threat hunt confirmed the compromise was contained to the initial brute force and SSH session. No evidence of further attacker activity was identified.

**Figure 6 — Threat Hunt Results**

> Screenshot: Splunk search results showing no lateral movement or escalation activity beyond initial authentication

---

## 🎯 Phase 4 — Root Cause Analysis

### Attack Chain Reconstruction

| Step | Action | Evidence Source |
|---|---|---|
| 1 | Attacker (192.168.102.4) initiates SSH dictionary attack using Hydra | auth.log — 407 Failed password entries |
| 2 | rockyou.txt wordlist used; password `password1` found on attempt 28 | Hydra output |
| 3 | Successful authentication as user `s3rvic` | auth.log — Accepted password event |
| 4 | Interactive SSH session established | Wireshark — TCP session on port 22 |
| 5 | Session closed — no further malicious activity observed | auth.log — session closed event |

### Root Cause

The root cause of this incident was the use of a weak, commonly known password (`password1`) for the `s3rvic` account, combined with SSH password authentication being enabled on an internet-accessible service. The password appears in the top 100 entries of the rockyou.txt wordlist, making it trivially crackable.

---

## 📋 Phase 5 — Incident Report

### Incident Summary

| Field | Value |
|---|---|
| Incident ID | INC-2026-001 |
| Severity | Critical |
| Status | Resolved |
| Detection Time | 2026-06-10 22:14:03 |
| Containment Time | 2026-06-10 22:31:00 |
| Analyst | Naheed001 |

### Timeline

| Time | Event |
|---|---|
| 22:10:00 | Hydra brute force attack initiated from 192.168.102.4 |
| 22:13:41 | 407th failed authentication attempt logged |
| 22:14:03 | Splunk alert fires — SSH Brute Force Threshold Exceeded |
| 22:14:10 | Password `password1` cracked — successful login as s3rvic |
| 22:16:00 | Analyst begins Tier 2 investigation |
| 22:28:00 | Threat hunt completed — no lateral movement confirmed |
| 22:31:00 | Incident contained — account locked, password reset |

### Containment & Remediation Actions

- Account `s3rvic` password reset immediately
- SSH password authentication disabled — key-based auth enforced
- Firewall rule added to restrict SSH access to authorised IPs only
- fail2ban deployed to auto-ban IPs after 5 failed attempts
- Alert threshold reduced from 10 to 5 failures per 5-minute window

---

## 🗂️ MITRE ATT&CK Mapping

| Field | Value |
|---|---|
| Tactic | Initial Access / Credential Access |
| Technique | T1110 — Brute Force |
| Sub-technique | T1110.001 — Password Guessing |
| Tactic (Post-Auth) | Execution |
| Technique (Post-Auth) | T1059 — Command and Scripting Interpreter |

**Figure 7 — MITRE ATT&CK Navigator**

> Screenshot: ATT&CK Navigator heatmap with T1110.001 and T1059 highlighted for this incident

---

## ✅ Summary

### Key Findings

| Finding | Detail |
|---|---|
| 🔴 Password cracked | `password1` found in 28 attempts using rockyou.txt |
| 🔴 Attack volume | 407 failed SSH attempts logged in auth.log |
| 🔴 Confirmed breach | 192.168.102.4 achieved successful login as s3rvic |
| 🟢 Contained | No lateral movement, escalation, or persistence detected |
| ✅ Detection working | Splunk alert fired within 4 minutes of attack start |
| ✅ Full timeline | Complete attack chain reconstructed from log evidence |
| ✅ Remediated | Password reset, key-auth enforced, fail2ban deployed |

### Production Improvements

📋 What would change in a real environment?

- **SOAR integration** — Automate containment actions (account lock, IP block) via a playbook triggered directly by the Splunk alert, reducing MTTC from minutes to seconds
- **EDR visibility** — An endpoint detection and response tool on the victim host would provide process-level telemetry, enabling detection of post-compromise activity invisible to auth.log
- **Threat intelligence enrichment** — Enrich source IPs against feeds like AbuseIPDB or VirusTotal at triage time to immediately identify known malicious infrastructure
- **UEBA baseline** — User and Entity Behaviour Analytics would flag the s3rvic account's first-ever login from an external IP as anomalous, even before the failure threshold was reached
- **Geo-IP visualisation** — In a production Splunk instance, attacker IPs would be plotted on a world map. Login attempts from unexpected countries are an immediate red flag
