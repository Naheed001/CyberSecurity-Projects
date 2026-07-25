# Beaconing Traffic Detection Lab

## Overview

This project covers simulating periodic C2 traffic and building time-based detection logic to identify beaconing behaviour. Rather than detecting events by count or threshold, detection is achieved by calculating the variance between successive connection intervals to flag suspiciously regular machine-driven traffic.

The fundamentals act as a learning methodology that teaches you the core knowledge required to complete the project independently. The write-up is my personal documentation of the lab I carried out.

> Note: The fundamentals section was AI-assisted and used as a structured learning resource prior to completing the lab
>

## What I Learned
>
Unlike the reverse shell lab, where a single outbound connection was the detection signal, beaconing taught me that the threat can hide itself in the regularity of intervals between connections rather than the connections themselves. This project also gave me hands-on experince with using time-delta analysis as a detection methodology - a fundamentally different approach to the count and threshold logic used in previous labs. Rather than simply counting events, I calculated variance across connection timestamps to measure how consistently the traffic arrived.
>

## Tools Used
>
 - Splunk
 - Netcat
 - Wireshark
 - tshark
 - Oracle VirtualBox
 - Kali Linux
 - Ubuntu
 - Python3
