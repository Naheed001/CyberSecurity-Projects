# Web Attack Detection in SIEM

## Overview

This project covers simulating common web application attacks against a vulnerable target and then building Splunk detection logic to catch them. The lab covers SQL Injection and Reflected XSS against DVWA, with Path Traversal and Command Injection as attempted tasks as part of a further challenge. Detection was built using SPL's regex, rex, stats and bin commands, relying on three distinct signals to determine malicious activity. These signals being suspicious parameters, error-rate spikes and encoded-payload density. By building around these signals, I was able to create SPL queries that encompassed a broader range of attacks, including ones that were never tested against the target.

The fundamentals act as a learning methodology that teaches you the core knowledge required to complete the project independently. The write-up is my personal documentation of the lab I carried out.

> Note: The fundamentals section was AI-assisted and used as a structured learning resource prior to completing the lab
>

## What I Learned
>
This project shifted my detection logic from the network layer to the application layer - instead of auth events or packet-level signatures, the evidence lived entirely in HTTP requests and their encoding. I also gained an understanding of how to adapt my SPL queries so that they may detect attacks that I have yet to test, reflecting how a real SOC analyst has to plan for attacks that have yet to occur, not just the ones they have experienced. Building on this, I learned how encoding is contextual to the layer being built - the same browser request encoded some characters and left others raw depending on whether they were structurally significant to the URL or the HTML response and I also came to understand that a browser's address bar is not a reliable record of what was actually sent.
>

## Tools Used
>
 - Splunk
 - DVWA
 - Apache
 - PHP
 - MariaDB
 - Oracle VirtualBox
 - Kali
 - Ubuntu 
