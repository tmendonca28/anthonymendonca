---
layout: default
title: "Hunting Malicious Infrastructure: A Practical Guide to Identifying and Tracking C2 Servers"
permalink: /tech-learning/hunting-malicious-infrastructure/
description: "Hunting Malicious Infrastructure: C2 Servers"
---
<h1>{{ page.title }}</h1>
<p class="subtitle">September 2024</p>
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

For instance, let's say the CTI team provides you with the malicious domain abc.xyz which is tagged as an IOC, the first step would be to check on a site like **dnsdumpster** the passive DNS records. You can then look into each of the resultant domains that pop up. This can be done using an online tool like **VirusTotal** or **AbuseIPDB**.  

## HUNTING A REAL-WORLD C2 SERVER
I enjoy understanding exactly 'how' things are done as I derive the most value from learning in this way.  
Here is the scenario: A suspicious domain - asupermaliciousdomain\[.\]com - has been flagged in your organization's network or provided to you by your CTI team. You suspect (never trust always verify 😉) it's likely linked to a C2 server. Here's how I would go about investigating it:  

### 1. PASSIVE DNS LOOKUP
Perform a Passive DNS lookup on asupermaliciousdomain\[.\]com and find that the domain has resolved to 2 IP addresses as well as find that it has historically been linked to other domains. Note these down for later where you can perform a Reverse DNS lookup and see if it points to other potentially malicious domains. Interestingly, the domain was created recently, which is also cause for suspicion.  

### 2. SHODAN INVESTIGATION
Next, pop the IP addresses into Shodan to check what services are running on them. You find that 1 IP address is hosting a web server running an unusual service on port 8080 and the other IP address is running an FTP server.  
It is useful to look for the following in Shodan:  
- Open ports that could be used for malicious communication e.g. 8080, custom ports.  
- Services that should not be exposed to the internet e.g. database services, telnet etc.  

### 3. CERTIFICATE AND METADATA ANALYSIS
Use Censys to check if any there are any SSL certificates are associated with the initial 2 IP addresses. You find both IPs share a self-signed certificate with the Common Name (CN) cn-supermalicious-cert. This is a valuable pivot point for us because the shared certificate suggests that these servers are part of the same malicious infrastructure.  

### 4. PIVOTING AND EXPANDING THE SEARCH
With the new certificate info, we can now pivot to search for other IP addresses that may be using the same SSL certificate. This helps in understanding and mapping out a broader network of related  C2 servers that the threat actor might be using.  
What can we pivot on?  
- Cross-referencing IP addresses and certificates.
- Searching for other domains linked to the same IP (reverse DNS lookup) or certificate.
- Investigating additional infrastructure linked to shared registration data or WHOIS info.  

Through the above process, you discover asupermaliciousdomain\[.\]com is part of a larger C2 infrastructure that spans multiple IP addresses and hosts services meant to control botnet activity.  

## SETTING UP DETECTION RULES
Once you identify the malicious infrastructure, the next step would be to set up detection rules that will help monitor for future use of these servers or similar tactics. The detection engineering piece in typical fusion SOCs should be done by the detection engineering or security engineering team.  
**YARA Rule Example:**  
[YARA rules](https://yara.readthedocs.io/en/stable/writingrules.html) are used to identify malware based on patterns in their structure or behavior. For example. if the malware contacting domain asupermaliciousdomain\[.\]com uses specific communication patterns or payload structures, you could write YARA rule to detect it.  
```
rule DetectMaliciousInfrastructure {
    meta:
        description = "Detects communication with malicious C2 server"
        author = "Anthony Mendonca"
    strings:
        $domain = "asupermaliciousdomain.com"
    condition:
        any of them
}
```
  
**Suricate Rule Example:**
[Suricata](https://docs.suricata.io/en/latest/rules/index.html) can be used for real-time network monitoring. Here's an example rule to detect traffic to one of the malicious IP addresses identified earlier:  
```
alert http any any -> xxx.xxx.x.xxx any (msg:"Malicious C2 Traffic Detected"; sid:100001;)
```  
This rule monitors network traffic for HTTP communication to the IP address associated with the C2 server.  

## MITIGATION AND PREVENTION
Once malicious infrastructure is identified, we need to mitigate against it. This might include:  
- Block identified domains and IP addresses at network perimeter (firewalls, DNS denylists etc.)
- Monitoring for further activity through enhanced vigilance based on detection rules developed using the SIEM tool, YARA or Suricata.
- Collaborating with external organizations to take down the infrastructure (ISPs) or share intelligence (ISACs) with other teams.  
