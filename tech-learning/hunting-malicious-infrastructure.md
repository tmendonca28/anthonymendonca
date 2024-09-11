---
layout: default
title: "Hunting Malicious Infrastructure: A Practical Guide to Identifying and Tracking C2 Servers"
permalink: /tech-learning/hunting-malicious-infrastructure/
description: "Hunting Malicious Infrastructure: C2 Servers"
---
<h1>{{ page.title }}</h1>
<p class="subtitle">August 2024</p>
<!-- <div style="text-align: center;">
    <img src="/images/hmi.jpeg" alt="hmi" title="hmi">
</div> -->

Identifying and tracking Command and Control (C2) servers is a critical skill for threat hunters and analysts. Threat actors use these servers to control malware remotely, steal sensitive data and launch further attacks. Also, MITRE has broken down generalized techniques for **Command and Control** [here](https://attack.mitre.org/tactics/TA0011/) with relation to the ATT&CK framework.  

## C2 SERVER? WHAT IS IT?
A C2 server is an external server used by threat actors to send commands and receive data from compromised systems within a victim's network. They are vital for maintaining control over the malware, exfiltrating data or issuing commands for further attacks.    

## TOOLS AND TECHNIQUES FOR C2 IDENTIFICATION
This involves a combination of OSINT tools and threat intelligence data to map out the server's infrastructure and gather more information and domains or IP addresses associated with it.  

> If you know the enemy and know yourself, you need not fear the result of a hundred battles. If you know yourself but not the enemy, for every victory gained you will also suffer a defeat. If you know neither the enemy nor yourself, you will succumb in every battle.  

A few key tools (non-exhaustive of course):
- Shodan: The 'Google' for internet-connected devices, used to identify devices exposed to the internet, including C2 servers.
- Censys: Similar to Shodan, but more focused on comprehensive scans and certificates.
- Passive DNS: Helps track historical use of domains and IP addresses.
- VirusTotal: Useful for analyzing suspicious domains, IP addresses and URLs.  

