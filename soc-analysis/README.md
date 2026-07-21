# SOC Analysis — Triage Case Studies

Alert investigations from my self-hosted detection environment. Each case documents
a real alert worked end to end: initial assessment → investigation → verification →
disposition → recommendation. The emphasis is on process and defensible reasoning,
not just the verdict — every conclusion is backed by evidence gathered on the
affected asset.

> **Environment note:** These are case studies from a personal homelab
> security-monitoring lab (Wazuh SIEM, with Suricata/Zeek and honeypot sources
> coming online), not a production SOC. Hosts and data are my own.

## Cases

| # | Alert | Source | Disposition | Key skill demonstrated |
|---|-------|--------|-------------|------------------------|
| [001](case-001.md) | "Trojaned version" of `/usr/bin/md5sum` | Wazuh rootcheck (rule 510) | False positive (verified) | Asset scoping; independent package-integrity verification; rule-tuning recommendation |

## How I work a case

1. **Read the alert fully** — the firing signature and metadata, not just the title.
2. **Assess context** — severity, frequency, affected asset, what the rule actually matches on.
3. **Form a hypothesis** — true positive, false positive, or needs escalation.
4. **Verify on the affected asset** — using an independent trust chain where possible,
   never taking the suspect's word for itself.
5. **Disposition and recommend** — document the finding and, where relevant, propose
   tuning or process changes to reduce future noise.

## Framework alignment

Each case maps to **NIST CSF 2.0** (primarily the **Detect** function) and, where the
rule carries them, compliance references such as PCI DSS and GDPR. The documented
review-and-disposition process also serves as evidence for a Logging & Monitoring
policy — the same work read through a governance (GRC) lens.

---

_Part of [homelab-and-portfolio](../README.md). Environment build details:
[Proxmox host](../proxmox) · [Wazuh SIEM over WireGuard](../wazuh)._