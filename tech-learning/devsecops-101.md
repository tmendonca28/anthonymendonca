---
layout: default
title: "A foray into DevSecOps"
permalink: /tech-learning/devsecops-101/
description: "Starting my journey in DevSecOps"
---
<h1>{{ page.title }}</h1>
<p class="subtitle">September 2022</p>
<img class="center" src="/images/devsecops-article.png" alt="ISP Snooping" title="ISP Snooping" />

In September 2022, I decided to enroll in the [Certified DevSecOps Professional](https://www.practical-devsecops.com/certified-devsecops-professional/) course to learn a bit more about DevSecOps as well as gain real-world technical skills to incorporate security into a CI/CD pipeline. I supplemented this learning with tonnes of freely available internet resources and decided to document my learning to keep myself accountable.

## devsecops terms
Before we jump in, here are a few terms that will constantly recur when learning about DevSecOps.

| Term | Definition |
| ----------- | ----------- |
| **SAST** | Static Application Security Testing: A white-box testing method that analyses source code to identify security issues. Examples of tools are Fortify, Veracode SAST and SonarQube. |
| **SCA** | Software Composition Analysis: Code scanning to provide visibility into open-source (3rd party libray) components used in development, for example, libraries used. An example is Snyk. |
| **DAST** | Dynamic Application Security Testing: A black-box testing method whereby dynamic tests are conducted on a given application. Examples of tools are OWASP ZAP and Veracode DAST. |
| **IAST** | Interactive Application Security Testing: A testing method that scans specific code workflows. Examples of tools are Seeker IAST and Acunetix. |
| **IAC** | Infrastructure as Code: A method to create and configure(or destroy) infrastructure using code definition. Examples of tools are AWS CloudFormation, Ansible, Chef, Puppet and Terraform|

## what is devsecops?
As defined by Gartner, DevSecOps is the integration of security into emerging agile IT and DevOps development as seamlessly and as transparently as possible. Ideally, this is done without reducing the agility of speed of developers or requiring them to leave their development toolchain environment.   
This concept is sometimes referred to as *shifting security left*, which simply implies introducing security at the early stages of the SDLC i.e. during design and implementation phases.  
A myriad of tools can be utilised within DevSecOps as seen below:
- During **Dev**elopment
    - Git Secrets
    - Security plugins in VSCode, Intellij or your IDE of choice 😉
    - Trufflehog
- As **Sec**urity add-ons
    - Code quality tools like Sonarqube
    - SAST security tools like Fortify, Veracode SAST and Checkmarx
    - DAST security tools like OWASP ZAP, WebInspect and Veracode DAST
    - IAC security tools like Cloudformation, Terraform
    - Container-based security like Aqua and Qualys Container Security
- During **Op**eration**s**
    - Build pipeline tools like Gitlab, GitHub Actions and GCP Cloudbuild
    - Cloud security posture management tools like Aqua CSPM
    - Container registry scanning tools like Aqua and AWS Registry Scanning
    - Infrastructure scanning tools like Nessus
    - Cloud security tools like Azure Defender


> And hence, **DevSecOps** was born.

