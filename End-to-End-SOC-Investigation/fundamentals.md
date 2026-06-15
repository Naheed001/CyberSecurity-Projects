# 🔐 End-to-End SOC Investigation — Fundamentals

**Project:** End-to-End SOC Investigation | **Course:** MSCi Cyber Security — Year 1

This page contains all the foundational theory you need before conducting a full SOC investigation. Each chapter builds on the last — read them in order.

---

## 🏢 Chapter 1: The Security Operations Centre (SOC)

**Why this matters first:** Before you can investigate anything, you need to understand the environment you're working in.

### What is a SOC?

A Security Operations Centre (SOC) is a centralised team and facility where security analysts monitor, detect, analyse, and respond to cybersecurity incidents. It is the operational hub of an organisation's defensive security posture.

### SOC Tiers

| Tier | Role | Responsibilities |
|---|---|---|
| Tier 1 — Alert Analyst | First responder | Monitor dashboards, triage incoming alerts, basic investigation, escalate |
| Tier 2 — Incident Responder | Deep investigator | Correlate events, perform threat hunting, determine scope and impact |
| Tier 3 — Threat Hunter | Proactive defender | Hunt for advanced threats, develop new detection rules, analyse malware |

### Key SOC Metrics

- **MTTA (Mean Time to Acknowledge)** — How quickly an analyst picks up an alert
- **MTTD (Mean Time to Detect)** — How quickly the SOC identifies a real incident
- **MTTC (Mean Time to Contain)** — How quickly the threat is contained after detection
- **False Positive Rate** — Percentage of alerts that are not genuine threats

> ⚠️ Alert fatigue is one of the biggest challenges in a SOC. Analysts who receive too many false positives begin to miss real threats. Good detection engineering minimises this.

---

## 📊 Chapter 2: SIEM Fundamentals

**Why this matters:** The SIEM is the SOC analyst's primary investigation tool.

### What is a SIEM?

A **Security Information and Event Management (SIEM)** system aggregates logs from across an organisation's infrastructure — servers, firewalls, endpoint agents, network devices — into a single searchable platform. It enables:

- Real-time alert generation
- Historical log investigation
- Correlation of events across multiple data sources
- Dashboards and visualisation for pattern recognition

### Splunk Architecture

| Component | Role |
|---|---|
| Universal Forwarder | Lightweight agent installed on endpoints. Monitors log files and ships new entries to the indexer in real time |
| Indexer | Receives, parses, and stores log data. Extracts timestamps, hosts, and source types automatically |
| Search Head | The web UI (port 8000). Where analysts write SPL queries, build dashboards, and configure alerts |

### SPL (Search Processing Language)

SPL works as a pipeline — each command passes its output to the next:

```spl
index=main sourcetype=linux_secure "Failed password"
| rex field=_raw "from (?P<src_ip>\d+\.\d+\.\d+\.\d+)"
| stats count as failed_attempts by src_ip
| sort -failed_attempts
| where failed_attempts > 10
```

**Breaking down the pipeline:**

1. `index=main sourcetype=linux_secure "Failed password"` — filter logs to SSH failures
2. `| rex "from (?P<src_ip>...)"` — extract source IP using a regex named capture group
3. `| stats count by src_ip` — aggregate: count events grouped by IP
4. `| sort -failed_attempts` — sort highest first
5. `| where failed_attempts > 10` — threshold filter

---

## 🔍 Chapter 3: The SOC Investigation Lifecycle

Every SOC investigation follows a structured lifecycle. Understanding this prevents missed steps and ensures evidence integrity.

### Phase 1 — Alert Triage

The first question an analyst asks: **Is this a true positive or a false positive?**

**Triage checklist:**
- What triggered the alert? (Rule name, query, threshold)
- What is the source and destination of the activity?
- Is the source IP internal or external? Known or unknown?
- Has this alert fired before? What was the outcome?
- What is the business context? (Is this a server that should have SSH open?)

**Triage decisions:**

| Decision | Meaning |
|---|---|
| Close — False Positive | Alert fired but no malicious activity detected |
| Escalate to Tier 2 | Suspicious activity requiring deeper investigation |
| Immediate Containment | Confirmed active threat requiring urgent response |

### Phase 2 — Investigation & Evidence Collection

The goal is to reconstruct exactly what happened using log evidence.

**Key questions to answer:**
- What was the first malicious event? (Initial access)
- What did the attacker do after gaining access? (Post-compromise activity)
- What accounts, systems, and data were affected?
- Is the attacker still present?

**Evidence sources:**

| Source | What it tells you |
|---|---|
| auth.log | Authentication successes and failures on Linux systems |
| /var/log/syslog | General system activity |
| Network packet capture (Wireshark) | Exact bytes transferred, protocol-level detail |
| Firewall logs | Allowed and blocked connections |
| Endpoint logs | Process execution, file system changes |

### Phase 3 — Threat Hunting

Threat hunting is **proactive** — you are looking for threats that haven't triggered an alert yet.

A threat hunt starts with a **hypothesis**:

> "If an attacker successfully brute-forced SSH credentials, they may have attempted privilege escalation or installed a persistence mechanism. Let me check for sudo commands and crontab modifications from the compromised account."

**Common hunt queries:**

```spl
# Privilege escalation attempts
index=main sourcetype=linux_secure ("sudo" OR "su ")
| table _time, host, _raw

# New cron jobs (persistence)
index=main "crontab"
| table _time, host, _raw

# Outbound connections to unusual ports
index=main sourcetype=network
| where dest_port NOT IN (80, 443, 22, 53)
| stats count by dest_ip, dest_port
| sort -count
```

### Phase 4 — Root Cause Analysis

Root cause analysis answers: **Why did this happen, and how do we prevent recurrence?**

**RCA framework — 5 Whys:**

1. Why was the account compromised? → Weak password
2. Why was a weak password in use? → No password policy enforced
3. Why was no password policy enforced? → No PAM configuration
4. Why was PAM not configured? → Security hardening checklist not followed
5. Why was the hardening checklist not followed? → No formal onboarding security review process

The root cause is the last "why" — the process gap, not just the technical failure.

### Phase 5 — Incident Reporting

Every investigation ends with a structured report. This serves multiple purposes:

- Provides evidence for management and legal teams
- Enables post-incident review and lessons learned
- Creates a record for regulatory compliance (e.g. GDPR Article 33)
- Builds organisational knowledge about attacker TTPs

**Standard incident report fields:**

| Field | Purpose |
|---|---|
| Incident ID | Unique identifier for tracking |
| Severity | Critical / High / Medium / Low |
| Detection Time | When the alert first fired |
| Containment Time | When the threat was neutralised |
| Attack Vector | How the attacker gained access |
| Affected Assets | Which systems, accounts, and data were impacted |
| MITRE ATT&CK Mapping | Standardised classification of techniques used |
| Remediation Actions | What was done to fix the issue |
| Recommendations | What should be changed to prevent recurrence |

---

## 🎯 Chapter 4: MITRE ATT&CK Framework

**Why this matters:** MITRE ATT&CK is the industry-standard language for describing attacker behaviour. SOC analysts use it to classify techniques and communicate findings clearly.

### What is MITRE ATT&CK?

MITRE ATT&CK is a globally-accessible knowledge base of adversary tactics and techniques based on real-world observations. It is organised into a matrix:

- **Tactics** — The *why*: the adversary's goal (e.g. Initial Access, Persistence, Privilege Escalation)
- **Techniques** — The *how*: the method used to achieve the tactic (e.g. T1110 — Brute Force)
- **Sub-techniques** — More specific variants (e.g. T1110.001 — Password Guessing)

### Key Tactics for This Project

| Tactic | ID | Relevance |
|---|---|---|
| Reconnaissance | TA0043 | Port scanning before the brute force |
| Initial Access | TA0001 | Gaining the first foothold via SSH |
| Credential Access | TA0006 | Stealing/cracking credentials (brute force) |
| Execution | TA0002 | Running commands after gaining access |
| Persistence | TA0003 | Mechanisms to maintain access (check for these in the hunt) |
| Privilege Escalation | TA0004 | Attempting to gain higher privileges |

### Relevant Techniques

| Technique | ID | Description |
|---|---|---|
| Brute Force: Password Guessing | T1110.001 | Systematically attempting passwords against a service |
| Valid Accounts | T1078 | Using stolen credentials to authenticate legitimately |
| Command and Scripting Interpreter | T1059 | Executing commands via shell after login |
| Scheduled Task/Job: Cron | T1053.003 | Persistence via crontab (hunt for this) |

---

## 🚨 Chapter 5: Detection Logic

**Why this matters:** Understanding how detections work helps you write better queries and understand why alerts fire.

### Signal vs. Noise

Not all events are detectable on their own. Detection relies on identifying **patterns** that deviate from baseline behaviour.

| Single Event | Pattern |
|---|---|
| 1 failed SSH login | 400+ failed SSH logins in 2 minutes from one IP |
| A user runs sudo | A user runs sudo 30 seconds after first login, at 2am |
| A file is modified | 50 files are modified in 30 seconds (ransomware) |

### Detection Pyramid (Pyramid of Pain)

The Pyramid of Pain describes how difficult it is for an attacker to change each type of indicator:

```
        /\
               /  \   TTPs (hardest to change — target these)
                     /----\
                          / Tools \
                              /----------\
                                 /  Domain Names \
                                   /----------------\
                                    /    IP Addresses   \
                                    /--------------------\
                                           File Hashes    (easiest to change — least valuable)
                                           ```

                                           SIEM-based detection should target **TTPs** (Tactics, Techniques, Procedures) rather than IP addresses or file hashes alone, because attackers change IPs and hashes trivially but cannot easily change their fundamental techniques.

                                           ### Threshold Tuning

                                           Threshold-based alerts require careful tuning:

                                           | Threshold Too Low | Threshold Too High |
                                           |---|---|
                                           | High false positive rate | Misses real attacks |
                                           | Alert fatigue | Late detection |
                                           | Wastes analyst time | Underestimates attacker speed |

                                           A good starting point for SSH brute force: **10+ failures from a single IP in a 5-minute window**. Adjust based on observed baseline in your environment.

                                           ---

                                           ## 📋 Chapter 6: Containment & Remediation

                                           **Why this matters:** Detection without response is incomplete. A SOC analyst must know what to do when a threat is confirmed.

                                           ### Containment Principles

                                           **Contain first, investigate second.** If an active attacker is present, stopping further damage takes priority over preserving perfect forensic evidence.

                                           **Containment options (escalating severity):**

                                           | Action | When to use |
                                           |---|---|
                                           | Reset compromised account password | Confirmed credential theft, no active session |
                                           | Lock compromised account | Active session suspected |
                                           | Block source IP at firewall | Active external attacker |
                                           | Isolate affected host from network | Active compromise, malware suspected |
                                           | Full system rebuild | Confirmed persistent compromise |

                                           ### Remediation vs. Containment

                                           | Term | Meaning |
                                           |---|---|
                                           | Containment | Stop the immediate bleeding — prevent further damage now |
                                           | Eradication | Remove the threat — delete malware, close the access vector |
                                           | Recovery | Restore normal operations — patch, rebuild, verify clean |
                                           | Lessons Learned | Post-incident review — update detections, processes, and training |

                                           This four-phase approach comes from the **NIST Incident Response Framework (SP 800-61)** — the industry standard for incident response.

                                           ### SSH Hardening Checklist

                                           For this specific project's attack vector:

                                           - [ ] Disable SSH password authentication (`PasswordAuthentication no` in sshd_config)
                                           - [ ] Use key-based authentication only
                                           - [ ] Restrict SSH access to specific IPs via firewall rules
                                           - [ ] Deploy fail2ban (auto-ban IPs after N failures)
                                           - [ ] Enable SSH login notifications
                                           - [ ] Run SSH on a non-standard port (security through obscurity — not sufficient alone)
                                           - [ ] Disable root login via SSH (`PermitRootLogin no`)
