---
layout: default
title: "Active Directory: Attack Paths and Privilege Escalation"
permalink: /tech-learning/active-directory-attack-paths/
description: "The offensive side of Active Directory: trust relationships, privileged groups, GPO abuse, ACL misconfigurations, and the named attack techniques that chain them together."
---

<h1>{{ page.title }}</h1>
<p class="subtitle">September 2025 &middot; {% include reading-time.html %}</p>

This is the second part of my Active Directory security notes. [Part one](/tech-learning/active-directory-primer/) covers the foundations - AD architecture, objects, Domain Controllers, and the Kerberos and NTLM authentication flows that underpin every named attack. This part covers the attack surface: how trust relationships create lateral movement paths, which privileged groups attackers actually target, how GPOs become a persistence mechanism, and how ACL misconfigurations chain silently into full domain compromise.

## Trusts: Lateral Movement Between Domains

Trusts allow users in one domain to access resources in another.

| Trust Type | Description |
|------------|-------------|
| **One-way** | Domain A trusts B: users in B can access A's resources, not vice versa |
| **Two-way** | Both domains trust each other |
| **Transitive** | Trust extends automatically: if A trusts B and B trusts C, A transitively trusts C |
| **Non-transitive** | Trust is limited to the two explicit domains only |

All domains in a forest have automatic two-way transitive trusts with each other. External (cross-forest) trusts require explicit setup.

> Trust relationships define lateral movement paths. A compromised account in a child domain may have a route to the parent domain's Domain Admins. Mapping trusts is one of the first actions both attackers and defenders should perform.

## Privileged Groups: What Attackers Are Hunting

"Path to Domain Admin" is meaningless unless you know what you are actually targeting. These are the high-value groups:

| Group | Scope | Why Attackers Want It |
|-------|-------|-----------------------|
| **Domain Admins** | Per domain | Full control of the domain - the primary target |
| **Enterprise Admins** | Forest-wide | Full control of the entire forest - exists only in the root domain |
| **Schema Admins** | Forest-wide | Can modify the AD schema - rarely needed, extremely dangerous if abused |
| **Backup Operators** | Per domain | Can back up and restore files including NTDS.dit - effectively Domain Admin |
| **Account Operators** | Per domain | Can create and modify accounts - useful for persistence |
| **Server Operators** | Per domain | Can log on to DCs interactively - often overlooked |

> A common mistake: defenders over-focus on Domain Admins and overlook groups like Backup Operators or Account Operators. An attacker in either group has a reliable path to full domain compromise. BloodHound maps these paths automatically.

## Group Policy Objects: The Double-Edged Sword

**Group Policy Objects (GPOs)** are the primary mechanism for pushing configuration across a domain. They apply to sites, domains, or OUs - meaning a single GPO can configure thousands of machines simultaneously. Settings include password policies, software deployment, login scripts, firewall rules, and more.

**Defensively**, GPOs enforce security baselines - disabling NTLM, restricting local admin rights, configuring audit logging, deploying certificates.

**Offensively**, GPOs are a persistence and lateral movement vector:
- Any account with **write access to a GPO** can modify it to execute arbitrary scripts on every machine it applies to
- Write access to an OU can allow an attacker to link a malicious GPO to that OU
- GPO misconfigurations are regularly surfaced by BloodHound

> If an attacker has GenericWrite on a GPO linked to Domain Controllers, they can push a script that runs on every DC the next time policy refreshes (default: every 90 minutes). That is full domain compromise through configuration management - no exploit needed.

## ACLs and Object Permissions: The Hidden Attack Surface

Every object in AD - users, groups, OUs, GPOs - has an **Access Control List (ACL)** defining who can do what to it. These permissions are separate from group membership and are frequently misconfigured.

Key permissions attackers look for:

| Permission | What It Allows |
|------------|----------------|
| **GenericAll** | Full control of the object - read, write, delete, own |
| **GenericWrite** | Write to any non-protected attribute - can set an SPN (enables Kerberoasting) |
| **WriteDACL** | Modify the object's ACL - attacker can grant themselves GenericAll |
| **WriteOwner** | Take ownership of the object - ownership implies control |
| **ForceChangePassword** | Reset a user's password without knowing the current one |
| **AddMember** | Add members to a group - escalate by adding yourself to Domain Admins |

These permissions chain together silently. A realistic path:

1. Low-privilege account has `GenericWrite` on a service account - set an SPN, Kerberoast it
2. Crack the service account's password - it has `WriteDACL` on Domain Admins - grant yourself `AddMember`
3. Add yourself to Domain Admins - full domain compromise, no exploits used

This chaining of ACL misconfigurations is exactly what **BloodHound** visualises as attack paths. LDAP queries expose these relationships by default to any authenticated domain user - which is why enumeration is so powerful before a single exploit is run.

## Why AD Is Every Attacker's Favourite Target

| Technique | What Is Abused | One-Liner |
|-----------|----------------|-----------|
| **Kerberoasting** | Service account SPNs | Request a Service Ticket for any SPN, crack the service account's password offline |
| **AS-REP Roasting** | Pre-auth disabled accounts | Get an AS-REP without authenticating, crack the hash offline |
| **Pass-the-Hash** | NTLM authentication | Use a captured hash directly to authenticate - no cracking needed |
| **Pass-the-Ticket** | Kerberos TGTs / STs | Steal and reuse a ticket from memory |
| **Overpass-the-Hash** | NTLM hash to Kerberos | Use an NTLM hash to request a Kerberos TGT |
| **DCSync** | AD replication protocol | Impersonate a DC to pull password hashes via DRS - requires replication privileges |
| **Golden Ticket** | krbtgt hash | Forge a TGT for any user with any privileges - survives most password resets |
| **Silver Ticket** | Service account hash | Forge a ST for a specific service - stealthier, no DC contact |
| **GPO Abuse** | GPO write permissions | Push malicious config or scripts to any machine the GPO applies to |
| **ACL Abuse** | Misconfigured object permissions | Chain permissions like GenericWrite and WriteDACL to escalate without exploits |
| **BloodHound** | AD relationships via LDAP | Graph-based enumeration that maps all attack paths to Domain Admin |
| **LDAP Enumeration** | Default read access | Any authenticated user can query users, groups, SPNs, ACLs, and trusts |

## Key Takeaways

- AD is the backbone of Windows enterprise environments. Centralised identity means centralised target.
- The Domain Controller holds everything. **NTDS.dit is the crown jewel.**
- Kerberos is the foundation of most named attacks. TGTs, STs, krbtgt, and SPNs unlock all of it.
- NTLM hashes are credentials. Cracking to plaintext is optional.
- Trust relationships define blast radius. The forest is the real security boundary, not the domain.
- Privileged groups beyond Domain Admins (Backup Operators, Account Operators) are regularly overlooked and reliably abused.
- GPOs are simultaneously the defender's most powerful tool and a dangerous attack surface.
- ACL misconfigurations chain silently into full domain compromise. BloodHound finds them.
- Any authenticated user can enumerate most of AD by default. Attackers do not need exploits to gather intelligence.
