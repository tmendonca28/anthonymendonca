---
layout: default
title: "Detection Validation as Code: Building a Closed-Loop Testing Pipeline"
permalink: /tech-learning/detection-validation-pipeline/
description: "A practical guide to building a CI/CD pipeline that automatically executes adversary techniques, queries your SIEM, and validates that detection rules actually fire - before and after every change."
---

<h1>{{ page.title }}</h1>
<p class="subtitle">November 2025 &middot; {% include reading-time.html %}</p>

Detection engineering teams produce KQL, SPL, and Sigma rules at pace. They test manually, once, when the rule is first written. Then the rule sits in the SIEM for months or years and nobody checks whether it still works. Log schemas change. Data connectors break. Normalisation updates shift field names. The rule that passed validation in January silently stops firing in March, and nobody knows until the next tabletop exercise - or the next breach.

This article walks through a detection validation pipeline that closes that loop: a system that automatically executes adversary techniques, waits for log ingestion, queries your SIEM, and records pass or fail. Tied to a CI/CD workflow, it means every detection rule change triggers a test run, and every week your entire detection library is re-validated against live log sources.

This article focuses on the architecture, the reasoning behind key design decisions, and what a real enterprise deployment looks like.

## The Core Problem

A detection rule is a hypothesis: if an attacker does X, we will see Y in the logs. Most teams validate that hypothesis once, at rule creation, then never again. The hypothesis is assumed to hold indefinitely.

It does not. Four things break detection rules without anyone touching the rule itself:

- **Log source changes.** A connector update renames a field. `InitiatingProcessAccountName` becomes `AccountName`. The rule returns zero results indefinitely with no error, no alert, and no visibility.
- **Normalisation drift.** ASIM schema versions evolve. A rule written against raw tables may behave differently after normalisation is enabled or updated across the workspace.
- **Exclusion creep.** Tuning changes accumulate over time and silently swallow true positives alongside the false ones they were meant to suppress. Nobody reviews the cumulative effect.
- **Infrastructure changes.** A new EDR rollout, a Sysmon config update, or a log forwarding change alters what data reaches the SIEM in the first place.

The result is a detection library that looks complete on paper but is partially broken in practice. The gap between "we have a rule for this" and "we know the rule works right now" is one of the most expensive and least visible problems in enterprise SOC programmes.

## Where Existing Approaches Fall Short

The obvious response is Breach and Attack Simulation (BAS). Platforms like AttackIQ, SafeBreach, and Cymulate automate technique execution and provide coverage reporting. They solve a real problem, but they introduce their own limitations in an enterprise context:

**They are black-box.** You cannot see exactly what the platform executed, precisely when it ran, or how it determined pass or fail. Audit teams and detection engineers want transparent logic they can verify and reproduce.

**They are decoupled from your detection code.** BAS platforms have no concept of your rule repository. When you update a KQL file in Git and merge the PR, nothing re-validates the rule automatically. The CI/CD integration simply does not exist.

**They are expensive relative to their footprint.** A Fortune 50 client paying six figures annually for a BAS subscription is still only running scheduled campaigns, not continuous validation. The value proposition weakens when you realise the same closed-loop logic can be built on open source tooling and integrated directly into your existing developer workflow.

Atomic Red Team solves the execution side but has no query loop. You run the test, then manually check the SIEM. That works for ad-hoc purple teaming and not much else at scale.

The gap is the integration layer: something that connects technique execution to SIEM validation and surfaces the result in the same tooling your engineers already use.

## What Closed-Loop Validation Actually Means

The loop has three stages:

1. **Execute.** Run the adversary technique in a controlled lab environment and record the exact timestamp.
2. **Query.** After a configurable ingestion wait, run the detection rule against the SIEM scoped to the post-execution window.
3. **Record.** Store pass or fail with evidence - alert count, run time, technique identifier - in a structured format that feeds both issue tracking and a coverage dashboard.

The loop is closed because the output feeds directly back into your development process. A failing rule surfaces as a labelled GitHub Issue assigned to the detection engineering team. A PR that breaks a detection fails its CI check before it merges. Detection quality becomes a gate, not an afterthought.

```
Detection rule updated (PR) or scheduled weekly run
              |
              v
    GitHub Actions workflow triggers
              |
              v
    Self-hosted runner on lab Windows VM
      - Executes Atomic Red Team test
      - Records exact run timestamp
              |
              | Logs forward to Sentinel workspace
              v
    Azure Log Analytics API
      - Query runs after configurable ingestion wait
      - Detection rule scoped to post-execution window
              |
              v
    PASS (rows > 0) or FAIL (rows == 0)
              |
              v
    Result stored in structured JSON artifact
    GitHub Issue opened for new failures
    ATT&CK Navigator coverage layer updated
```

The self-hosted runner is an important detail. Running the GitHub Actions runner directly on the lab Windows VM means atomic tests execute locally with no SSH tunnelling or RDP required. The runner has the same context as the machine, which means it picks up domain membership, local credentials for cleanup steps, and the full Sysmon event stream without any forwarding configuration specific to the pipeline.

## The Rule Manifest

The pipeline needs to know which atomic test maps to which detection rule. A YAML manifest per rule handles this, stored alongside the KQL files in the detection repository:

```yaml
# detections/credential-access/t1003-001-lsass-access.yaml
name: "Credential Dumping via LSASS Memory Access"
technique: T1003.001
severity: high
kql_file: t1003-001-lsass-access.kql
atomic_tests:
  - technique: T1003.001
    test_number: 1
  - technique: T1003.001
    test_number: 2
owner: detection-engineering
ingestion_wait: 300
environment: workstation
requires_approval: false
```

A few fields worth explaining:

`ingestion_wait` is per-rule rather than global. Sysmon process events typically land in a well-tuned Sentinel workspace within 60-90 seconds. Network events, cloud audit logs, and some security data connector sources take significantly longer. Setting a global five-minute wait for every rule wastes pipeline time on fast sources and causes false failures on slow ones.

`environment` tags which lab tier the test requires. Not every technique runs safely on a standalone workstation. Active Directory techniques need a domain controller and joined workstation. Cloud techniques need a dedicated test subscription. The scheduler dispatches based on this tag.

`requires_approval` flags techniques that are too destructive or too noisy for automatic scheduling. These are skipped on the weekly sweep and only run when a human manually triggers the workflow with explicit technique selection. Ransomware simulation and domain replication fall into this category for most enterprise clients.

The KQL file is the detection rule exactly as it lives in Sentinel - no modifications, no test-specific logic. If the rule works in the pipeline, it is the rule that works in production. Any divergence between the two is itself a bug worth surfacing.

## The Validation Loop

When the pipeline runs a rule, the validator does four things in sequence.

**Execute and timestamp.** The atomic test runs on the lab VM and the exact UTC timestamp is recorded before execution begins. This timestamp becomes the lower bound for the Sentinel query - anything generated before it is irrelevant and filtered out.

**Wait with retry logic.** Rather than sleeping a fixed duration, the validator polls with exponential backoff: query at two minutes, four minutes, eight minutes. The first non-zero result terminates the loop. This is meaningfully faster than a fixed five-minute sleep for rules with fast-ingesting log sources, and it avoids false failures caused by ingestion lag on slower sources.

**Query the workspace.** The detection rule KQL is submitted to the Log Analytics API with a `TimeGenerated >= timestamp` filter appended. If rows come back, the detection fired. The row count is stored alongside the pass/fail status because alert volume trending upward over weekly runs is itself a signal - a rule generating 300 alerts where it used to generate one is not passing cleanly; it is misfiring broadly.

**Record and surface.** Results write to a structured JSON artifact retained for 90 days. Failed rules that do not already have an open issue trigger automatic issue creation with the rule name, technique, test number, and run timestamp. The duplicate check matters in practice - without it, a detection that fails every Monday for a month generates four identical issues.

## Detection Drift: The Scheduled Run

The weekly sweep is the most operationally valuable component of the pipeline. Every detection rule re-validated against live log sources on a schedule, not just when someone thinks to check.

Detection drift - previously passing rules that silently stop working - is pervasive in large enterprises. The organisations most exposed are the ones with the most rules, because rule volume creates the illusion of coverage while the actual validated detection rate quietly degrades.

The weekly run catches:

- A Sysmon config change that dropped an event category the rule depended on
- A MDE connector update that renamed or removed a field
- A new ASIM schema version that shifted table structure
- A tuning exclusion added during an incident response that was too broad and never reviewed
- A log forwarding change that broke coverage for a specific endpoint tier

Without automation, none of these surface until a red team exercise or an incident. With the pipeline, they surface Monday morning as a labelled issue with enough information for a detection engineer to investigate and close within the day. The mean time to detect a broken detection drops from months to days.

## Enterprise Deployment Considerations

For a Fortune 50 deployment, the base architecture holds but several implementation details change.

**Isolated lab workspace.** Do not run atomic tests against the production Sentinel workspace. Build a dedicated lab workspace and use the Sentinel API or Terraform to sync detection rules from production to lab on a schedule. Validation runs against lab; results surface in production tooling. This eliminates any risk of test activity polluting production alert queues or triggering automated response workflows.

**Tiered execution environments.** A single Windows workstation covers a significant portion of ATT&CK coverage, but not all of it. Network-based techniques need a victim and an attacker VM on the same segment. Active Directory techniques need a domain controller and a joined workstation. Cloud techniques need Azure or AWS test accounts with appropriate IAM scope. Each environment tier runs as a separate self-hosted runner pool, and the manifest `environment` tag drives dispatch.

**ITSM integration.** GitHub Issues work for engineering-led teams. In a Fortune 50 SOC environment, failed detections typically need to flow into ServiceNow as change requests or Jira as backlog items with priority ratings and SLA assignments. The issue-creation step in the workflow is a single API call. Swapping the target is the work of an afternoon, and the evidence bundle - atomic output, query response, timestamp - attaches to the ticket automatically.

**Approval gates for high-risk techniques.** Some techniques require human sign-off before running. The `requires_approval` flag in the manifest provides the hook. The scheduler skips these on automated runs; a separate manual workflow dispatch requires a named approver to confirm before execution. This is essential for techniques that touch domain replication, write to audit logs, or modify shared infrastructure.

**Evidence collection for compliance.** Audit teams at large enterprises need a complete trail: proof the test ran, proof the detection fired (or did not), and a timestamp tying both together. The validator collects the atomic test output, the raw Log Analytics query response, and the full result into an evidence bundle stored as a workflow artifact. Ninety-day retention covers most internal and external audit windows.

**Workspace health check.** Before each validation run, the pipeline checks that the lab workspace is actively receiving data by querying for recent heartbeat events. If the lab VM's heartbeat is absent, the run aborts rather than producing a sweep of false failures. Treating infrastructure health as a precondition to validation prevents a VM restart or network blip from generating hundreds of spurious issues overnight.

## Coverage Output

Results feed two downstream consumers: issue tracking and a coverage layer for MITRE ATT&CK Navigator. The coverage layer is a JSON file that colours each technique on the ATT&CK matrix based on validation status - green for rules that passed, red for rules that failed, amber for execution errors. This is the artefact you present to an engineering director or a CISO.

The distinction the layer makes visible is important: green means the rule exists and was validated successfully this week. Grey means the technique is not covered. Red means coverage was claimed but the rule is currently broken. For most large enterprises, the first time they run this, red is more prevalent than they expected.

The layer also exposes a subtler problem: techniques with rules in the repository but no manifest entry. These are rules that have never been tested by the pipeline at all. Surfacing them as a distinct category drives the team to either write the atomic mapping or acknowledge that the rule has never been validated.

## Development Notes

A few things that matter when building this for real.

**Start with three rules.** Resist building the coverage dashboard before the core loop works end to end. Pick three rules with known-good Atomic Red Team coverage, validate the execute-wait-query cycle works correctly for each, then expand. The architecture above does not need to change once the loop is solid.

**The runner environment requires careful preparation.** Antivirus on the lab VM will block Atomic Red Team tests. MDE in block mode will intervene before telemetry is generated. On the lab host specifically, AV exclusions and audit-mode EDR are the right configuration - the goal is telemetry generation, not defence. Keep this clearly documented and separated from production machine policy.

**Not everything has an atomic.** Atomic Red Team covers a substantial portion of the ATT&CK matrix but not all of it. For techniques without existing atomics, the right answer is writing your own: a script that performs the action, generates the expected telemetry, and cleans up after itself. These live in a `custom-atomics/` directory in the same repository and are referenced from manifests with a `source: custom` field. This is where offensive security depth translates directly into detection engineering value - understanding what a technique actually does at the system level is necessary to write a realistic simulation of it.

**Alert count is a secondary signal.** Track volume in the results over time, not just pass or fail. A rule that generates one alert per test run is behaving as expected. A rule that starts generating three hundred is misfiring broadly - a tuning regression as significant as an outright failure. Alert count trending across the weekly sweep gives you an early warning of coverage quality degrading before it becomes a missed detection.

**Schema stability matters.** Define the results JSON schema before building the dashboard or any downstream tooling. Changing the schema later means migrating historical artifacts or losing continuity in coverage trending. The minimal useful schema is: rule name, technique, test number, run timestamp, status, and alert count.

## MITRE ATT&CK Mapping

| Component | What It Validates |
|-----------|-------------------|
| Rule manifest | Technique-to-rule mapping exists and is maintained |
| Atomic execution | Lab environment generates telemetry for the technique |
| Query pass | Detection rule fires on current log schema and field names |
| Scheduled sweep | Detection continues to pass after infrastructure or config changes |
| Coverage layer | Visual representation of validated vs. unvalidated technique coverage |

The pipeline does not prove a detection is correct for every variant of a technique. It proves the rule fires for the specific atomic test that represents it. That is a narrower claim - and an honest one. Document it as such when presenting coverage results to stakeholders. The goal is not a perfect heatmap; it is a heatmap you can defend.

