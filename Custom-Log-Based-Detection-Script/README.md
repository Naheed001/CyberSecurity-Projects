# Custom Log-Based Detection Script

## Overview

This project covers building a custom intrusion detection script from scratch, moving detection logic out of Splunk and into Python to understand how brute force detection works under the hood. The script identifies brute force patterns using regex and threshold-based logic and forwards structured alerts to Splunk using the HTTP Event Collector. Hydra was used to generate real attack traffic so that the script may forward real data to Splunk.

The fundamentals act as a learning methodology that teaches you the core knowledge required to complete the project independently. The write-up is my personal documentation of the lab I carried out.

> Note: The fundamentals section was AI-assisted and used as a structured learning resource prior to completing the lab
>

## What I Learned
>
This project gave me hands-on experience translating detection logic that I had previously only configured through Splunk's SPL into raw python, deepening my understanding of what lies under the surface of threshold-based detection. In addition to this, Kali's use of systemd-journald rather than tradition flat log files taught me to verify environment specific behaviour rather than assume textbook log formats will apply.
>

## Tools Used
>
 - Splunk
 - Hydra
 - Oracle VirtualBox
 - Kali Linux
 - Python3
