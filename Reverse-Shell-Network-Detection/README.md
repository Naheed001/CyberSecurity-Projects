# Reverse Shell Network Detection

## Overview

This project covers detection and analysis of reverse shell connections over a network. It uses Metasploit to simulate a reverse shell attack and Splunk to detect anomalous outbound connections that are characteristic of command-and-control (C2) traffic. It acts as a solid intermediate-level project for those interested in pursuing SOC analyst or blue team roles

The fundamentals act as a learning methodology that teaches you the core knowledge required to complete the project independently. The write-up is my personal documentation of the lab I carried out

Note: The fundamentals section was AI-assisted and used as a structured learning resource prior to completing the lab

## What I Learned

Through this project, I gained a comprehensive understanding of how reverse shells operate at the network level and how attackers use them to establish persistent command-and-control channels. I simulated a reverse shell attack using Metasploit (msfvenom payload + Metasploit listener) and developed detection logic in Splunk to identify the characteristic outbound connection patterns. This project deepened my understanding of the difference between bind shells and reverse shells, NAT traversal, and the challenge of detecting encrypted C2 traffic.

## Tools Used
- Splunk
- Metasploit Framework / msfvenom
- Oracle VirtualBox
- Kali Linux
- Ubuntu
