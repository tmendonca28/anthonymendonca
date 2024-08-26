---
layout: default
title: "Hunting Malicious Infrastructure: A Practical Guide to Identifying and Tracking C2 Servers"
permalink: /tech-learning/hunting-malicious-infrastructure/
description: "Hunting Malicious Infrastructure: A Practical Guide to Identifying and Tracking C2 Servers"
---
<h1>{{ page.title }}</h1>
<p class="subtitle">August 2024</p>
<!-- <div style="text-align: center;">
    <img src="/images/hmi.jpeg" alt="hmi" title="hmi">
</div> -->

Identifying and tracking Command and Control (C2) servers is a critical skill for threat hunters and analysts. Threat actors use these servers to control malware remotely, steal sensitive data and launch further attacks. Also, MITRE has broken down generalized techniques for **Command and Control** [here](https://attack.mitre.org/tactics/TA0011/) with relation to the ATT&CK framework.  

## What is a C2 Server?
A C2 server is an external server used by threat actors to send commands and receive data from compromised systems within a victim's network. They are vital for maintaining control over the malware, exfiltrating data or issuing commands for further attacks.  
### Example
Think of a botnet, where each compromised machine connects to a C2 server for instructions. This can be used to send payloads to infected systems or initiate a DDoS attack against targets. Understanding how these servers operate is useful in threat hunting.