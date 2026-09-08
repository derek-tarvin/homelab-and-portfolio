# Cowrie Honeypot (Vesta)
> **AI assistance note**:  In this case, I leaned on Claude to help me build out this setup and this document.  While spinning up a VPS is something I have done many times, this particular configuration was new.
## What it is

As part of the homelab, a second VPS (named Vesta) was spun up on DigitalOcean to act as a honeypot — 1 vCPU, 1GB RAM, 25GB storage. Spun up 07/21/2026, running Ubuntu 26.04 with Cowrie as the actual honeypot. A Wazuh agent monitors Vesta and feeds data back to the Wazuh manager (Fornax, a VM on Proxmox) via a dedicated VPN tunnel.

## Architecture — and the decision

While adding Vesta to the existing VPN would have worked, it would have exposed Hestia — a production server — to an intentionally compromised system. That was not an acceptable risk.

Segregating Vesta onto a second VPN, linked separately to the Wazuh manager, limits exposure to a system that's low-value and easily replaced or restored.

Since Cowrie listens on port 22 for SSH, the real admin SSH port was moved to a random port number.

**WireGuard configuration:**

- **Fornax**
  - UFW only opens specific ports for the VPN, restricted to connections from `10.10.10.1` and `10.10.20.1`.
  - 2 WireGuard interfaces:
    - `wg0.conf` — interface `10.10.10.2/24`, AllowedIPs limited to `10.10.10.1/32`, Endpoint set to Hestia's public IP.
    - `wg1.conf` — interface `10.10.20.2/24`, AllowedIPs limited to `10.10.20.1/32`, Endpoint set to Vesta's public IP.
- **Vesta**
  - UFW enabled 07/25/2026, default deny inbound. It allows Cowrie's listener,
    the WireGuard port the manager dials into, and a relocated admin SSH port —
    and nothing else. That last part is the point: on a honeypot, any open port
    other than Cowrie's is a path to compromising the box that never touches
    Cowrie, and therefore never reaches the SIEM.
  - 1 WireGuard interface:
    - `wg0.conf` — interface `10.10.20.1/24`, AllowedIPs limited to `10.10.20.2/32`.
- **Hestia**
  - UFW only opens specific ports for the VPN.
  - Interface `10.10.10.1/24`, AllowedIPs limited to `10.10.10.2/32`.

This creates two interfaces on Fornax, each scoped to a single peer IP. Hestia and Vesta each limit their own communication to their specific interface on Fornax — a specific, point-to-point link in both directions, not a shared network.

> Note: each WireGuard interface is tied to a specific port via `ListenPort` in each `wg*.conf`. Port numbers left out for security.

## What's running

- **Hestia** — production VPS, Ubuntu 22.04.
- **Vesta** — DigitalOcean VPS, Ubuntu 26.04.
- **Fornax** — VM on Proxmox, Ubuntu 26.04.

## Detection rules

Wazuh ships nothing for Cowrie out of the box — the ruleset below was written with AI assistance. Levels were set so the constant noise of SSH connection attempts wouldn't drown out the events that actually matter; connects sit at level 3, and the loud levels are reserved for post-auth behavior and payload retrieval. (See [Three failures](#three-failures) below for how these rules ended up shaped this way.)

```xml
<group name="cowrie,honeypot,">

  <rule id="100100" level="0">
    <decoded_as>json</decoded_as>
    <field name="eventid">^cowrie\.</field>
    <description>Cowrie honeypot event (base)</description>
  </rule>

  <rule id="100101" level="3">
    <if_sid>100100</if_sid>
    <field name="eventid">cowrie.session.connect</field>
    <description>Cowrie: SSH session opened from $(src_ip)</description>
  </rule>

  <rule id="100102" level="5">
    <if_sid>100100</if_sid>
    <field name="eventid">cowrie.login.failed</field>
    <description>Cowrie: failed login $(username)/$(password) from $(src_ip)</description>
  </rule>

  <rule id="100103" level="10">
    <if_sid>100100</if_sid>
    <field name="eventid">cowrie.login.success</field>
    <description>Cowrie: ACCEPTED login $(username)/$(password) from $(src_ip)</description>
  </rule>

  <rule id="100104" level="10">
    <if_sid>100100</if_sid>
    <field name="eventid">cowrie.command.input</field>
    <description>Cowrie: post-auth command from $(src_ip): $(input)</description>
  </rule>

  <rule id="100105" level="12">
    <if_sid>100100</if_sid>
    <field name="eventid">cowrie.session.file_download</field>
    <description>Cowrie: attacker downloaded a payload from $(src_ip)</description>
  </rule>

  <rule id="100110" level="10" frequency="8" timeframe="120">
    <if_matched_sid>100102</if_matched_sid>
    <same_field>src_ip</same_field>
    <description>Cowrie: SSH brute-force / credential spray from $(src_ip)</description>
  </rule>

</group>
```

## Three failures

**1. Crossed WireGuard keys.** During setup, the public keys of Vesta and Fornax were swapped, which prevented the VPN from establishing a secure connection.

**2. UFW on Fornax was documented but never enabled.** This was remedied by changing UFW's status to active. This could have been a larger issue on a public-facing server — but since Fornax is a VM on a Proxmox host that's also behind its own firewall, the actual exposure was minimal.

**3. A green dashboard was hiding a dead detection pipeline.** Once connected to the Wazuh manager, the dashboard showed 2 active agents and no errors. But since Vesta was internet-facing, port scanners hitting 22 should have shown up eventually — and the dashboard showed no traffic at all. Opening an SSH connection to Vesta and deliberately trying to fail a login still granted access to the honeypot, exactly as designed. This confirmed the VPN link was up and data was flowing, but not that the detection layer was working.

Investigating further: a custom decoder (`cowrie-json`) had been built to catch Cowrie's log entries and hand them to the rules above — but testing showed it never caught anything. Running a sample log entry through `wazuh-logtest` confirmed why: Wazuh's built-in `json` decoder was claiming the line first. `cowrie-json` lost the race on sequencing, not on the accuracy of its match — the default `json` decoder is broad enough to catch the same lines and gets evaluated first. Rather than trying to force `cowrie-json` ahead of a core decoder, the fix moved to the rules layer instead: rules were rewritten to match on JSON-decoded events with an `eventid` field containing `cowrie`, so `cowrie-json` never needed to win at all.

A follow-up test using fictitious credentials logged in successfully, as Cowrie is designed to allow. Checking the Wazuh dashboard confirmed rules `100101`, `100103`, and `100104` fired correctly. Rule `100102` did not fire — because Cowrie accepted these particular credentials rather than rejecting them, not because no login can fail. Confirming `100102` would require testing a credential combination Cowrie's `userdb.txt` explicitly denies.

**Thesis:** a green agent, a loaded ruleset, and no errors is indistinguishable from a working detection pipeline and a completely silent one. Only running a known-good event through `wazuh-logtest` and reading which decoder actually claimed it told the two apart.

## Open items

- Let the honeypot soak and gather data.
- Return to Wazuh after a week and check for warnings.
- If a sufficient body of data is gathered, start analyzing for patterns, vulnerabilities, and hardening.

## What's next

- Data analysis once the honeypot has soaked long enough to produce meaningful data.
- Review for specific target files and start making it harder for attackers to reach them.
- Retune rule `100110`. Its 8-failures-in-120-seconds window was set from an assumption, and it has still never fired on live traffic — measuring the actual inter-attempt intervals in the capture is the way to fix it. See the [Wazuh SIEM write-up](../wazuh/README.md) for the current state of the ingest chain and its known gaps.