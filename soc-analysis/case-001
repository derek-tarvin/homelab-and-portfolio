# Case 001 — Rootcheck "Trojaned Binary" Alert on `/usr/bin/md5sum`

> **Environment note:** This is a triage case study from a personal homelab
> security-monitoring environment, not a production SOC. Hosts and data are my own.
> The intent is to document a real alert-investigation process end to end.

**Analyst:** Derek Tarvin
**Date:** 2026-07-20
**Alert source:** Wazuh 4.x (rootcheck), self-hosted manager
**Disposition:** False Positive (verified)
**Time to disposition:** ~30 min

---

## Environment

| Host | Role | Wazuh component |
|---|---|---|
| `ubuntu-wazuh` | Ubuntu Server 24.04 VM (home LAN) | Manager (agent id `000`) |
| `hestia` | Production VPS | Enrolled agent |

The manager also monitors itself via the built-in local agent, which reports under
**agent id `000`**. This detail is load-bearing for the investigation (see below).

---

## The Alert

Wazuh **rule 510** (rootcheck host-based anomaly detection), level **7** (medium),
fired on the manager's local agent:

```json
{
  "agent":   { "name": "ubuntu-wazuh", "id": "000" },
  "manager": { "name": "ubuntu-wazuh" },
  "data":    { "file": "/usr/bin/md5sum", "title": "Trojaned version of file detected." },
  "rule": {
    "id": "510",
    "level": 7,
    "firedtimes": 20,
    "description": "Host-based anomaly detection event (rootcheck).",
    "groups": ["ossec", "rootcheck"],
    "pci_dss": ["10.6.1"],
    "gdpr": ["IV_35.7.d"]
  },
  "location": "rootcheck",
  "full_log": "Trojaned version of file '/usr/bin/md5sum' detected. Signature used: 'bash|^/bin/sh|file\\.h|proc\\.h|/dev/[^cfhlnrstuw]|^/bin/.*sh' (Generic).",
  "timestamp": "2026-07-20T23:23:55.649+0000"
}
```

## Why it warrants investigation

A trojaned system binary is a legitimate rootkit indicator: an attacker replaces a
core utility (here, `md5sum`) with a malicious version to hide activity or subvert
integrity checks. Treated as a true positive on a production host, this is an incident.

## Investigation

**1. Read the firing signature, not just the title.**
The `full_log` shows the matched signature is marked `(Generic)`:

```
'bash|^/bin/sh|file\.h|proc\.h|/dev/[^cfhlnrstuw]|^/bin/.*sh'
```

This is a broad heuristic that flags files containing common strings found in known
rootkits (`bash`, `/bin/sh`, `proc.h`, etc.). It is **not** a match against a specific
known-malware hash. A legitimate coreutils binary like `md5sum` contains many of these
strings as normal compiled content, so a generic string match here is a known
false-positive pattern for Wazuh rootcheck against standard distro binaries.

**2. Weigh the context signals.**
- `firedtimes: 20` — recurring on a regular cadence, consistent with a benign standing
  signature rather than a live intrusion (a real compromise does not politely re-alert
  20 times on schedule).
- Level 7 (medium) — Wazuh itself is not treating this as critical.

**3. Identify the affected asset — the pivotal step.**
The alert's `agent id` is `000`, which is the **manager's** local agent — so the
flagged file lives on **`ubuntu-wazuh`**. My initial verification was run on `hestia`
(the VPS) because that was the session I was in. `dpkg -V coreutils` returned clean
there — but that verifies the *wrong host*. Correct answer, invalid evidence. I caught
the mismatch between the shell hostname (`hestia`) and the alert's host (`ubuntu-wazuh`)
and re-ran verification on the correct asset.

**4. Verify integrity on the correct host (`ubuntu-wazuh`), independent trust chain.**

```bash
sudo dpkg -V coreutils      # (empty output = all package files match shipped hashes)
md5sum /usr/bin/md5sum
```

`dpkg -V` validates the file against the package manager's integrity database — a trust
chain separate from the (suspect) `md5sum` binary itself, so the suspect is not vouching
for itself. Empty output confirms `/usr/bin/md5sum` is the unmodified distro binary.

## Disposition

**False positive.** The alert is documented Wazuh rootcheck behavior: a generic rootkit
string signature matching legitimate content in the standard coreutils `md5sum` binary.
Package integrity verified intact on the alerting host via an independent mechanism.

## Recommendation

- Tune rule 510 / add a verified-path exception for confirmed coreutils binaries to
  reduce recurring noise (~20 fires/24h from this one signature).
- Retain the tuning rationale as documented justification rather than silently
  suppressing the rule.

## Key lesson

**Asset scoping is part of the investigation, not a formality.** The alert metadata
names the affected host; verifying anywhere else can produce a *correct disposition
backed by invalid evidence* — which passes review and rots quietly. Confirm you are
investigating on the actual alerting asset before trusting your own check. In a
multi-host setup (VPS agent → home-LAN manager), this is a live risk, not a hypothetical.

---

## Framework mapping (governance view)

| Framework | Mapping |
|---|---|
| NIST CSF 2.0 | **Detect (DE)** — security event analysis; improving detection processes via tuning |
| PCI DSS | 10.6.1 (log review) — carried on the rule |
| Process evidence | Supports a Logging & Monitoring policy: documented alert review, verification, and disposition |