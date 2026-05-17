---
layout: default
title: "Standardising Detections with Sigma and sigma-cli"
permalink: /tech-learning/sigma-cli-detection-standardisation/
description: "A practical guide to converting SIEM-specific detection rules into portable Sigma format and using sigma-cli to generate KQL, SPL, and other query languages from a single source of truth."
---

<h1>{{ page.title }}</h1>
<p class="subtitle">December 2025 &middot; {% include reading-time.html %}</p>

Most detection engineering teams write rules in the query language of whatever SIEM they happen to be running. KQL for Sentinel, SPL for Splunk, Lucene for Elastic. The rule works, the alert fires, the job is done. The problem surfaces when the organisation adds a second SIEM, migrates platforms, acquires a company on a different stack, or wants to contribute rules to a community that does not share their vendor.

At that point, every rule needs to be rewritten. The logic is the same. The detection intent is the same. But the syntax is incompatible, the field names differ, and the person doing the conversion has to understand both query languages well enough to produce an accurate translation. At scale, this is expensive and error-prone.

Sigma solves this. It is a vendor-neutral, YAML-based rule format that describes what a detection does rather than how to express it in a specific query language. Write the rule once in Sigma. Use sigma-cli to generate KQL for Sentinel, SPL for Splunk, or Lucene for Elastic from the same source file.

This article walks through what Sigma is, how sigma-cli works, and how to take an existing KQL detection rule and standardise it as a portable Sigma rule. Screenshots of the tooling in practice will follow as a separate update once the detection library is built out.

## What Sigma Is

Sigma is an open detection rule format created by Florian Roth and Thomas Patzke. Think of it as YARA for log-based detections: a structured, readable format that captures the intent of a detection without binding it to a specific platform.

A Sigma rule describes three things: the log source the detection targets, the field values that constitute a match, and the condition that determines when to alert. Everything else - the query syntax, the table names, the field name conventions - is handled by the conversion tooling at output time.

The Sigma project is hosted at [github.com/SigmaHQ/sigma](https://github.com/SigmaHQ/sigma) and ships a community library of thousands of rules covering the MITRE ATT&CK matrix. sigma-cli is the official conversion tool, replacing the older sigmac utility. Both the rule format and the tooling are actively maintained.

## Setting Up sigma-cli

sigma-cli installs via pip. The base package handles the conversion engine; backends for specific SIEMs install separately:

```bash
pip install sigma-cli
```

Install the backends you need. For Microsoft Sentinel and Microsoft 365 Defender:

```bash
pip install pySigma-backend-microsoft365defender
```

For Splunk:

```bash
pip install pySigma-backend-splunk
```

For Elastic:

```bash
pip install pySigma-backend-opensearch
```

Once installed, verify the available backends and pipelines with:

```bash
sigma list backends
sigma list pipelines
```

Pipelines are the field mapping layer between Sigma's generic field names and the specific field names your SIEM uses. The same backend can have multiple pipelines depending on whether you are querying raw Windows event logs, Sysmon events, or normalised ASIM tables. Getting the pipeline right matters: the wrong pipeline produces a query that compiles correctly but queries the wrong fields.

## Anatomy of a Sigma Rule

A Sigma rule is a YAML file with a defined structure. The sections that matter most for detection engineering:

```yaml
title: Credential Dumping via LSASS Memory Access
id: a9945f81-b27a-4e97-9e2e-3b5d4e6f7a88
status: test
description: Detects attempts to access LSASS process memory for credential dumping
references:
  - https://attack.mitre.org/techniques/T1003/001/
author: Anthony Mendonca
date: 2025/11/01
modified: 2026/01/15
logsource:
  category: process_access
  product: windows
detection:
  selection:
    TargetImage|endswith: '\lsass.exe'
    GrantedAccess|contains:
      - '0x1010'
      - '0x1410'
      - '0x143A'
  condition: selection
falsepositives:
  - Legitimate security tooling accessing LSASS
  - Some EDR products
level: high
tags:
  - attack.credential_access
  - attack.t1003.001
```

The sections worth understanding in detail:

**`id`** is a UUID that uniquely identifies this rule. Generate one per rule. This is what lets tooling track rules across format conversions and version history.

**`status`** reflects the rule's maturity. `test` means it exists but has not been validated in production. `stable` means it has been validated and is expected to produce reliable results. `experimental` means the detection logic is speculative. Use these honestly - they communicate trust level to anyone consuming the rule.

**`logsource`** is where Sigma abstracts away platform specifics. Rather than naming a specific table, you declare a category (`process_access`, `process_creation`, `network_connection`) and a product (`windows`, `linux`, `aws`). sigma-cli maps this to the correct table in your SIEM via the pipeline. The `process_access` category maps to Sysmon Event ID 10, which captures the access mask and target process name that LSASS credential dumping tools use.

**`detection`** is the rule logic. It consists of named selection groups and a condition that combines them. Modifiers like `|endswith`, `|contains`, `|startswith`, and `|re` apply transformations to field values before matching.

**`condition`** uses selection group names with boolean logic. `selection` means the named group must match. `selection1 and not filter` means both must be true. `1 of selection*` means any selection group prefixed with `selection` can match.

**`tags`** maps to MITRE ATT&CK. Use the `attack.tNNNN` format to link techniques. This feeds coverage reporting tools including the ATT&CK Navigator layer output.

## Translating a KQL Rule to Sigma: A Worked Example

Take this KQL detection for LSASS access written for Microsoft Sentinel:

```kql
SecurityEvent
| where EventID == 4656
| where ObjectName has "lsass"
| where AccessMask in ("0x1010", "0x1410", "0x143A")
| project TimeGenerated, Account, ProcessName, ObjectName, AccessMask
```

The translation to Sigma is not a mechanical conversion. It requires understanding what the rule is detecting and expressing that intent using Sigma's field model.

The first decision is the log source. This rule uses `SecurityEvent` with Event ID 4656, which is an object handle request. The Sysmon equivalent for detecting LSASS access is Event ID 10 (ProcessAccess), which maps to Sigma's `process_access` category. Sysmon Event ID 10 captures `TargetImage` and `GrantedAccess` directly, making the detection more reliable and the Sigma mapping cleaner. If your environment does not have Sysmon, you stay with `SecurityEvent` and use the `windows` product with a specific `definition` noting the event ID dependency.

The Sigma version using Sysmon is the example above. The field names - `TargetImage` and `GrantedAccess` - match Sysmon's field model, which sigma-cli's pipeline maps correctly to your SIEM's table schema.

For the `SecurityEvent`-based version without Sysmon:

```yaml
logsource:
  product: windows
  service: security
detection:
  selection:
    EventID: 4656
    ObjectName|contains: 'lsass'
    AccessMask|contains:
      - '0x1010'
      - '0x1410'
      - '0x143A'
  condition: selection
```

The `service: security` declaration tells sigma-cli to map this to the Windows Security event log rather than the generic `process_access` category.

## Using sigma-cli to Convert Rules

With the rule written, sigma-cli generates the SIEM-specific query:

```bash
# Convert to Microsoft Sentinel KQL (via M365 Defender backend)
sigma convert -t microsoft365defender -p microsoft365defender lsass-access.yml

# Convert to Splunk SPL
sigma convert -t splunk lsass-access.yml

# Convert with Sysmon pipeline for Sentinel via windows backend
sigma convert -t microsoft365defender -p sysmon lsass-access.yml
```

To convert an entire directory of rules at once:

```bash
sigma convert -t microsoft365defender -p microsoft365defender detections/ -r
```

The `-r` flag recurses through subdirectories. The output is a batch of queries you can pipe to a file, import to your SIEM, or feed into a CI/CD pipeline.

To validate rule syntax before conversion:

```bash
sigma check lsass-access.yml
```

This catches structural errors, missing required fields, and invalid modifier usage before the rule reaches your detection library or your SIEM.

## Building a Sigma Detection Library

A Sigma library is a directory of YAML files organised by MITRE ATT&CK tactic. This structure makes it straightforward to reason about coverage and to run batch conversions scoped to a specific tactic:

```
detections/
  credential-access/
    t1003-001-lsass-access.yml
    t1003-002-sam-registry-dump.yml
    t1110-001-brute-force.yml
  defense-evasion/
    t1055-001-process-injection-dll.yml
    t1562-001-security-tool-disabled.yml
  lateral-movement/
    t1021-001-rdp-login.yml
    t1021-002-smb-admin-share.yml
  initial-access/
    t1078-001-valid-accounts-local.yml
    t1566-001-spearphishing-attachment.yml
```

Each file follows the naming convention `tNNNN-NNN-descriptive-slug.yml`, which ties every rule to its ATT&CK technique and makes the library self-documenting. Running a batch conversion for a specific SIEM is a single command that produces a deployable query set from the full library.

This structure also integrates cleanly with the detection validation pipeline. Each Sigma rule can have a corresponding manifest that links it to an Atomic Red Team test, making the library the single source of truth for both detection logic and validation coverage.

## CI/CD Integration

The natural extension of a Sigma library is generating SIEM-specific queries as part of a CI/CD pipeline. When a rule is updated in Git and a PR is merged, a GitHub Actions workflow converts the updated rule to KQL and deploys it to Sentinel via the Sentinel API. No manual copy-paste. No version drift between the source rule and what is running in production.

The workflow structure mirrors the detection validation pipeline covered in the previous article. The key additional step is the sigma-cli conversion stage, which runs before deployment and fails the pipeline if any rule fails validation:

```yaml
- name: Validate and convert rules
  run: |
    sigma check detections/ -r
    sigma convert -t microsoft365defender -p microsoft365defender detections/ -r -o queries/sentinel/
```

Failing `sigma check` on a PR blocks the merge. The converted queries in `queries/sentinel/` are what the deployment step pushes to the SIEM.

## What Sigma Cannot Express

Sigma is well-suited to field-match detections. It is not designed for stateful, multi-event, or mathematically complex logic. A few cases where KQL capabilities exceed what Sigma can represent:

**Aggregation and thresholds.** Brute force detection counting failed logins per user over a window is expressible in Sigma using the `near` condition, but the result across backends is inconsistent. Complex aggregations are better kept as native SIEM queries alongside your Sigma library, clearly documented as SIEM-specific.

**Joins and lookups.** A detection that correlates two event types (e.g., a process launch followed by a network connection from the same process) can be approximated in Sigma using `near`, but proper join logic is a KQL or SPL construct. Write these natively.

**Haversine-based detections.** The impossible travel detection described elsewhere on this site uses trigonometric functions in KQL. Sigma has no equivalent. These stay as KQL.

The practical approach is a hybrid library: Sigma for everything that fits the field-match model (which is the majority of ATT&CK technique coverage), and SIEM-native rules documented separately for the cases that require platform-specific logic.

## Development Notes

**Generate UUIDs for every rule from the start.** It is tempting to skip this and fill them in later. Do not. UUID continuity matters once you are tracking rules across versions, correlating them with validation results, or contributing rules to the SigmaHQ community library.

**Validate before you convert.** `sigma check` is fast and catches the majority of errors that would otherwise produce a confusing conversion failure. Make it a habit to run validation before committing rule changes.

**Pin sigma-cli versions in your CI environment.** The tool and its backends update regularly. A backend update that changes field mappings can silently alter the output of your conversion pipeline. Pin versions in your requirements file and upgrade deliberately.

**Test the converted output.** sigma-cli producing a valid KQL query does not mean the query detects what the Sigma rule describes. Run the converted query against your SIEM and confirm the results match your expectation before treating the rule as validated. This is where the detection validation pipeline earns its value: the sigma-cli conversion and the atomic test run sit in the same workflow, so a rule that converts and validates is a rule you can trust.

**Field mapping gaps are real.** Not all Sigma field names map cleanly to all backends. If sigma-cli produces a query with an unmapped field (it will warn you), you either need a custom pipeline that adds the mapping or you need to rethink the rule's logsource. The SigmaHQ documentation on field mappings per backend is the reference to reach for here.

## MITRE ATT&CK Mapping

| Sigma Component | Purpose |
|-----------------|---------|
| `logsource` | Abstracts the log table - backend handles the mapping |
| `detection` | Field-match logic expressed in vendor-neutral field names |
| `tags` | ATT&CK technique linkage for coverage reporting |
| `status` | Communicates rule maturity and production readiness |
| `id` (UUID) | Persistent identifier across format conversions |
| sigma-cli backend | Translates Sigma field names to target SIEM schema |
| sigma-cli pipeline | Applies environment-specific field mappings on top of the backend |

