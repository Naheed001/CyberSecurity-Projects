# End-to-End SOC Investigation Simulation

## Overview

This project simulates a full attack on a network system and covers the framework a SOC analyst would use to investigate and respond to the attack.

The attack consists of a port scan followed by a brute force attack on an open port. After exploitation of the port, the attacker runs a reverse shell to compromise the target system. Wireshark was used to detect the reverse shell connection by identifying any suspicious outbound traffic, whilst Splunk was used as another source of evidence - ingesting logs from the Ubuntu target machine as well as creating a dashboard to visualise the attack timeline. Wireshark was also used to follow the TCP stream of the packets created during the reverse shell connection so that a defender could see what commands the attacker run and what their next move would be.

The investigation follows both the NIST and PICERL (Preparation, Identification, Containment, Eradication, Recovery, Lesson Learnt) frameworks to simulate how a SOC analyst would respond to a real-world attack. It acts as a solid project for those interested in blue team roles, whilst also providing red team exposure through simulating a full attack chain on a system.

The fundamentals act as a learning methodology that teaches you the core knowledge required to complete the project independently. The write-up is my personal documentation of the lab I carried out.

> Note: The fundamentals section was AI-assisted and used as a structured learning resource prior to completing the lab
>

## What I Learned
>

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
