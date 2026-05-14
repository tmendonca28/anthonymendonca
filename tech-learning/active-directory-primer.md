---
layout: default
title: "Active Directory: A Security Primer"
permalink: /tech-learning/active-directory-primer/
description: "A foundational primer on Active Directory architecture, Kerberos authentication, and why it is the primary target in Windows enterprise environments."
---

<h1>{{ page.title }}</h1>
<p class="subtitle">June 2025 &middot; {% include reading-time.html %}</p>

I wrote this as a personal reference while working on both sides of Active Directory - securing it for clients during defensive engagements and attacking it during adversary emulation exercises. Kerberos in particular is something I kept having to re-derive from scratch, so this article is my attempt to write it down clearly enough to actually stick. If you also find yourself forgetting which ticket is encrypted with which hash, or confusing the TGT with the Service Ticket, this is for you.

This first part covers the foundations: AD architecture, the objects that make it up, the Domain Controller's role, and the authentication protocols behind every named attack. Part two covers the attack surface - trusts, privileged groups, GPO abuse, ACL misconfigurations, and the techniques that chain them together.

## What Is Active Directory and Why Does It Matter?

Active Directory (AD) is Microsoft's directory service for managing Windows environments. At its core it does one thing really well: centralised identity and access management. Every user, computer, and resource in a Windows enterprise authenticates through it. That centralisation is what makes it powerful for administrators and what makes it the single most valuable target in any Windows network.

If you control AD, you control the network. That is not hyperbole. It is why "path to Domain Admin" is the goal of virtually every enterprise penetration test.

AD uses **Single Sign-On (SSO)** - log in once, access everything your account is permitted to reach. This is done through Kerberos (the primary protocol) and NTLM (the legacy fallback).

## Objects: The Building Blocks

Everything stored in AD is an **object** - a record with attributes. The most security-relevant ones:

| Object | Notes |
|--------|-------|
| **Users** | Human accounts and service accounts |
| **Computers** | Machine accounts; every domain-joined machine has one |
| **Groups** | Collections of users/computers; control access to resources |
| **OUs (Organisational Units)** | Containers for organising objects; Group Policy is applied here |
| **Domain Controllers** | The servers running AD DS |
| **Contacts** | External users, not security principals, cannot authenticate |
| **Shared Folders** | Pointers to network shares |

> **Service accounts** are regular user objects configured to run applications or services. They often have elevated privileges, weak passwords, and rarely rotate credentials. They are prime targets.

## Architecture: Forests, Trees, and Domains

AD is hierarchical. From top to bottom:

```
Forest
├── Tree A
│   ├── Domain
│   │   ├── OU
│   │   │   └── Users / Computers / Groups
│   │   └── OU
│   │       └── Users / Computers / Groups
│   └── Child Domain
└── Tree B
    └── Domain
```

- **Forest**: the true security boundary. Objects in different forests cannot interact without an explicit trust. A new AD deployment starts a new forest.
- **Tree**: one or more domains sharing a contiguous DNS namespace.
- **Domain**: the core management unit. Has its own policies, account database, and at least one Domain Controller.
- **OU (Organisational Unit)**: an organisational container. Group Policy Objects (GPOs) are applied here and admin rights can be delegated here. An OU is *not* a security boundary.

> **Security note:** The forest is the real boundary, not the domain. Compromising one domain in a forest often provides a path to every other domain in it. Cross-forest trust misconfigurations extend that blast radius further.

## Domain Controllers: The Brain

Domain Controllers (DCs) run **Active Directory Domain Services (AD DS)** - the database storing every account, password hash, group membership, and policy. Every authentication request flows through a DC.

Key facts:
- The **PDC Emulator** (one of five FSMO roles) handles password changes and authentication failures. It is the authoritative source for time synchronisation, which Kerberos depends on.
- DCs store **NTDS.dit** - the AD database file containing every user's password hash. Extracting this file is often the endgame of an AD attack.
- Multiple DCs exist for redundancy and stay in sync via the **Directory Replication Service (DRS)**. The replication protocol itself can be abused - I cover DCSync in part two.

## Authentication: Kerberos and NTLM

This section is the foundation for understanding every named attack covered in part two.

### Kerberos

Kerberos is the default authentication protocol in modern AD. It is **ticket-based**: instead of sending your password to every resource you want to access, you prove your identity once to a trusted third party - the **Key Distribution Center (KDC)**, which runs on the DC - and receive tickets you present to services.

Three parties are always involved: the client, the KDC (comprising the Authentication Service and the Ticket Granting Service), and the target service. **Passwords never travel over the wire.** Identity is proved entirely through encryption: if you can decrypt something encrypted with your hash, you must be you.

#### Pre-Authentication

By default, when your client sends an authentication request it includes a timestamp encrypted with your password hash. The KDC decrypts it and if it checks out, you are authenticated. This is **pre-authentication**.

> When pre-authentication is *disabled* on an account (a misconfiguration), the KDC responds to any authentication request without requiring proof. The response is still encrypted with the user's hash, but now anyone can request it, take that encrypted blob offline, and crack it. That is **AS-REP Roasting**.

#### The Six-Step Flow

<div class="article-img">
  <img src="/images/kerberos-flow.png" alt="Kerberos six-step authentication flow between Client (Alice), KDC (on DC), and Service">
</div>

#### Three Things to Lock In

**The service never calls home to the KDC.**
The service decrypts the Service Ticket using its own hash - if it decrypts cleanly, the ticket is valid. This offline validation is why **Silver Tickets** work: an attacker with the service account's hash can forge a ticket the service accepts with zero DC involvement and no logs generated.

**The TGT is encrypted with the krbtgt account's hash.**
Only the KDC knows this hash. If an attacker obtains it, they can forge arbitrary TGTs for any user, with any group memberships, valid for any duration. This is the **Golden Ticket** attack. The only fix is resetting the krbtgt password *twice* (AD retains the previous hash for replication).

**Any authenticated user can request a Service Ticket for any SPN.**
A **Service Principal Name (SPN)** is the identifier an account registers when running a service (e.g. `MSSQLSvc/dbserver.corp.local`). Requesting a ticket for an SPN is normal Kerberos behaviour, but it means any domain user receives a blob encrypted with the service account's hash and can crack it offline. This is **Kerberoasting**.

#### Kerberos in 30 Seconds

| Step | What Happens | Why It Matters |
|------|--------------|----------------|
| AS-REQ / AS-REP | Prove identity to KDC, receive TGT | TGT encrypted with krbtgt hash - forge it = Golden Ticket |
| TGS-REQ / TGS-REP | Trade TGT for a Service Ticket | ST encrypted with svc hash - any user can request it = Kerberoasting |
| AP-REQ | Present ST to the service | Service validates offline - forge ST with svc hash = Silver Ticket |
| Pre-auth disabled | KDC gives AS-REP without proof | Encrypted blob crackable offline = AS-REP Roasting |

### NTLM: The Legacy Fallback

NTLM is the older challenge-response protocol. It is used when Kerberos is unavailable - for example when accessing resources by IP address instead of hostname, off-domain systems, or legacy applications.

<div class="article-img">
  <img src="/images/ntlm-flow.png" alt="NTLM challenge-response authentication flow between Client and Server">
</div>

NTLM hashes can be captured over the network (e.g. with Responder), dumped from memory (lsass), or extracted from the SAM/NTDS.dit database, and **used directly to authenticate without cracking**. This is **Pass-the-Hash**. In NTLM, the hash *is* the credential. Cracking it to plaintext is optional.

## AD Components Worth Knowing

| Component | Purpose | Security Relevance |
|-----------|---------|-------------------|
| **AD DS** | The core directory database | Stores all credentials and policy |
| **AD CS** (Certificate Services) | PKI built into AD | Heavily abused; ESC misconfigs allow privilege escalation |
| **AD FS** (Federation Services) | SSO and claims-based auth for external apps | Token forgery if misconfigured |
| **AD LDS** | Lightweight LDAP-compatible directory | Less common in attacks |
| **AD RMS** | Rights management for documents | Rarely targeted directly |

> **AD CS deserves special attention.** It is a fully featured PKI bolted into most AD environments, often with insecure default certificate templates. The ESC1-ESC13 vulnerability classes (from SpecterOps research) have made AD CS a primary post-exploitation target in modern engagements.

Part two covers the attack surface: trust relationships, privileged groups, GPO abuse, ACL misconfigurations, and a full breakdown of the named AD attack techniques.
