# End-to-End SOC Investigation Simulation

## Overview

This project covers a full attack on a network system and covers the framework a SOC analyst would use to investigate and respond to the attack.

The attack simulates a full attack chain - port scanning, brute forcing an open port and deploying a reverse shell to compromise the target.
Wireshark and Splunk were then used to detect, investigate and visualise the attack, following both NIST and PICERL frameworks.

This project is suited to those interested in blue team roles whislt also providing red team exposure through hands-on technique implementation.

The fundamentals act as a learning methodology that teaches you the core knowledge required to complete the project independently. The write-up is my personal documentation of the lab I carried out.

> Note: The fundamentals section was AI-assisted and used as a structured learning resource prior to completing the lab
>

## What I Learned
>
This project gave me hands-on experience in applying the PICERL framework to respond to a real-world style attack, as well as experience in implementing the offensive techniques I had learned as a connected attack chain, rather than independently, to gain access to a target machine. Additionally, analysing TCP streams in Wireshark allowed me to observe the commands an attacker would run post-exploitation, strengthening my ability to think both as an attacker and a defender. I also became more proficient in writing SPL queries to isolate specific events and building dashboards in Splunk to reduce an analysts response time to future intrusions.
>

## Tools Used
>
 - Splunk
 - Wireshark
 - Netcat
 - Metasploit
 - Nmap
 - Hydra
 - Oracle VirtualBox
 - Kali Linux
 - Ubuntu
