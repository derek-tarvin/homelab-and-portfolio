# Wazuh SIEM Deployment

## Objective
Deploy a Wazuh SIEM (manager + indexer + dashboard) and ingest telemetry
from a real, internet-facing production Linux host.

## Status: 🚧 In progress
- ✅ Wazuh 4.14 all-in-one manager deployed on Ubuntu 24.04 (Proxmox VM)
- ✅ Dashboard reachable, clean-install snapshot taken
- 🚧 Agent-to-manager connection: securing the link between a remote VPS
  and the home-LAN manager (see Design note)

## Design note — connecting a remote agent without exposing the SIEM
The manager lives on a private home LAN; the monitored host is a public
VPS. Rather than expose the manager's enrollment ports to the internet,
the agent connects to the manager over a private encrypted tunnel — the
SIEM is never publicly reachable. [expand once the tunnel is built]

## What this demonstrates
SIEM stand-up, agent enrollment, and secure architecture of the
agent-to-manager path — attack surface considered, not just "it works."

## What's next
Green agent reporting real VPS telemetry → screenshots (sanitized) →
detection rules against live traffic.
