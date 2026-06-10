# Reverse Shell Network Detection

## Overview

This project covers detection and analysis of reverse shell connections over a network. It uses both Metasploit and Netcat to simulate a reverse shell attack and Wireshark to detect anomalous outbound connections that are characteristic of command-and-control (C2) traffic. It acts as a solid intermediate-level project for those interested in pursuing SOC analyst or pentesting related roles, as it develops both red-team and blue-team skillsets.

The fundamentals act as a learning methodology that teaches you the core knowledge required to complete the project independently, as well as the ethics surrounding it. This project is not to be performed outside of isolated lab environments due to the legal and ethical implications. The write-up is my personal documentation of the lab I carried out.

> Note: The fundamentals section was AI-assisted and used as a structured learning resource prior to completing the lab
>

## What I Learned
>
This project built my practical knowledge of how reverse shells operate and how attackers leverage them to establish persistent C2 channels. It also strengthened my ability to behave like a defender, analysing packets in Wireshark to identify suspicious outbound connections indicative of command-and-control activity (MITRE ATT&CK: TA0011).

I simulated reverse shell attacks using both Netcat and Metasploit, deploying an msfvenom payload with a Metasploit listener. Following initial access to the target system, I experimented with privilege escalation to explore the capabilities of Meterpreter - gaining a deeper understanding of how much control an attacker can achieve post-exploitation and the difficulty in detection and prevention of such attacks.
>

## Tools Used
>
- Metasploit Framework / msfvenom
- Netcat (traditional)
- Wireshark
- Oracle VirtualBox
- Kali Linux
- Ubuntu
