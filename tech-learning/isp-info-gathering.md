---
layout: default
title: "Can your ISP see what URL you browse to?"
permalink: /tech-learning/isp-info-gathering/
description: "How your ISP may know what site you are browsing to"
---
<h1>{{ page.title }}</h1>
<p class="subtitle">August 2022</p>
<img class="center" src="/images/isp-article.png" alt="ISP Snooping" title="ISP Snooping" />

Following an interview question I had recently, I decided to look more into this. It seems that your Internet Service Provider (ISP) may know a bit more about what sites you tend to visit than you may know. Here are a few of the possible ways they can do so:  

## A decent portion of the internet still runs on http
Sites running on the HTTP protocol send all requests in plaintext and hence data is not encrypted over the communication channel. This means that an ISP would have full visibility on what sites you are visiting and what you are actually doing there. In this time and age, all sites should really move to using HTTPS, however, there may be no justification on purchasing a TLS certificate for a potentially low data volume site.

## Monitoring dns
Even if a site employs the use of HTTPS, an IPS simply would be able to monitor requests made to the Domain Name System (DNS) and hence learn the name of the domain you are visiting. 

## Server name indication (sni)
SNI indicates a website’ hostname or domain name and is sometimes sent as part of the TLS handshake process. 

In HTTPS, a TLS handshake occurs first, before the HTTP conversation can begin of course. Without SNI, there is no way for the client to indicate to the server which hostname they are conversing with. As a result of this, the server may send back the TLS certificate for the wrong hostname. If the name on the certificate does not match the name the client is trying to reach, this results in an error. SNI thus adds the domain name to the TLS handshake process, so that the TLS process can reach the correct domain name and receive the right TLS certificate. 

The SNI is included in the Client Hello message which is the first step of TLS handshake, which is sent in plaintext, unless it uses a TLS Encrypted ClientHello (ECH) extension.