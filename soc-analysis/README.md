# SOC Analysis — Investigations & Incident Reports
>**AI assistance note**:  This page was produced using AI.


Written security work from two different environments. The difference between them
changes what each document can prove, so it is stated per document rather than left
for the reader to work out.

**Lab casework** — alert investigations from my self-hosted detection environment.
Each case documents a real alert worked end to end: initial assessment →
investigation → verification → disposition → recommendation. The emphasis is on
process and defensible reasoning, not just the verdict — every conclusion is backed
by evidence gathered on the affected asset, and that evidence still exists.

**Incident reports** — incidents from production environments I operated,
reconstructed after the fact. Where remediation destroyed the evidence, the report
says so and states what it cannot establish.

Neither substitutes for the other. The lab work has evidence and no stakes. The
production incident had stakes and no surviving evidence.

> **Environment note — lab casework:** case studies from a personal homelab
> security-monitoring lab (Wazuh SIEM, with honeypot telemetry feeding it), not a
> production SOC. Hosts and data are my own.
>
> **Environment note — incident reports:** a production host I operated for a
> business since closed. Anonymized. No logs or artifacts survive.

## Cases

| # | Alert | Source | Disposition | Key skill demonstrated |
|---|-------|--------|-------------|------------------------|
| [001](case-001.md) | "Trojaned version" of `/usr/bin/md5sum` | Wazuh rootcheck (rule 510) | False positive (verified) | Asset scoping; independent package-integrity verification; rule-tuning recommendation |

## Incident reports

| Report | Environment | Evidence available |
|---|---|---|
| [2025-09 host compromise](incident-report-2025-09-host-compromise.md) | Production host, anonymized. Business since closed. | None surviving — the host was wiped during my own remediation. Timeline derived from a third-party record, verified, withheld for privacy. |

## How I work a case

Applies to the lab casework above. The 2025 incident predates this environment and
is documented as a retrospective, not worked to this process.

1. **Read the alert fully** — the firing signature and metadata, not just the title.
2. **Assess context** — severity, frequency, affected asset, what the rule actually matches on.
3. **Form a hypothesis** — true positive, false positive, or needs escalation.
4. **Verify on the affected asset** — using an independent trust chain where possible,
   never taking the suspect's word for itself.
5. **Disposition and recommend** — document the finding and, where relevant, propose
   tuning or process changes to reduce future noise.

## Framework alignment

Lab cases map to **NIST CSF 2.0**, primarily the **Detect** function, and — where the
rule carries them — compliance references such as PCI DSS and GDPR. The documented
review-and-disposition process also serves as evidence for a Logging & Monitoring
policy: the same work read through a governance (GRC) lens.

The incident report sits in **Respond** and **Recover** instead, and follows the
standard phase structure: detection, scope assessment, containment, eradication,
recovery, corrective action, lessons learned.

## On provenance

Each document states its own scope, its limits, and whether the work it describes
was done with AI assistance. Where a conclusion is hindsight rather than reasoning
held at the time, it is labeled as hindsight.

---

_Part of [homelab-and-portfolio](../README.md). Environment build details:
[Proxmox host](../proxmox) · [Wazuh SIEM over WireGuard](../wazuh)._