---
layout: default
title: "Technical Information Gathering using theHarvester"
permalink: /anthonymendonca/tech-learning/tech-info-gathering-theharvester/
description: "Getting up and running with theHarvester for OSINT."
---
<h1>{{ page.title }}</h1>
<p class="subtitle">August 2022</p>

## Intro
It allows one to gather bother technical and people information on a given target.  
Information gathering is super important during a red teaming exercise as it can prove extremely useful during exploitation. A good example of this is finding a few (forgotten) insecure client resources (like servers) that could be potentially easier to get into than the main client's site.  
[theHarvester](https://www.kali.org/tools/theharvester/) automates this process and aids in information gathering in a heartbeat.  

The first step in a red teaming exercise is generally collection of information about a given target. This is usually the Recon phase within the [Kill Chain](https://www.lockheedmartin.com/en-us/capabilities/cyber/cyber-kill-chain.html)

Search engines like Google, DuckDuckGo, Bing and Yandex can potentially be used for this purpose. These can be used to find email addresses, subdomains and sometimes sensitive files related to said target. They can also be used to map out an organization's hierarchy thereby allowing for social engineering attacks, for example, spear phishing C-level or Director level individuals within the company.

DNS queries and port scans can also be run against servers of the target so as to get a better understanding on them.

This is sadly manual querying, and would hence take an extremely long amount of time to search for all the various parts explained above. This is where theHarvester steps in; it can automate a lot of the aforementioned searches and save us time.

It is a tool for OSINT (Open Source Intelligence) gathering and assists in determining a company's external threat landscape on the Internet. The tool gathers emails, names, subdomains, IPs and URLs using a myriad of data sources. It can be used to perform both intrusive and non-intrusive information gathering. Being open source, it can be catered to a specific organization's needs. 

The source code can be found [here](https://github.com/laramies/theHarvester)  as well as the installation instructions for use.

To get up and running, run the following:
```console
    git clone https://github.com/laramies/theHarvester
```
Then to install all the libraries required:
```console
    pip3 install -r requirements.txt
```
The _cd_ into theHarvester folder and run:
```console
    python3 ./Harvester.py
```

## Finding Subdomains and IP addresses
These are extremely important attack avenues.
An example of enumerating the subdomains for a given domain is as seen below:
```console
    python3 ./theHarvester.py -d <domain> -l <limit> -b <data-sources>
```
Interesting finds can be looked into further with an _nmap_ command.
We can also find information about people working in a specific organisation using the same command, but focusing our data sources to LinkedIn or Twitter.  

theHarvester tool can be easily expanded and integrated with other tools. For instance, we could run any emails we find in our results to check against exposed password databases.  
It can be utilized by an organization to reduce their sensitive information footprint on the Internet by focusing on (stale) DNS records, email addresses and old websites that could be exploited by malicious actors. 