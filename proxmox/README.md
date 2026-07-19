# Proxmox VE Homelab Host

## Objective
Stand up a bare-metal type-1 hypervisor to host a security lab: a SIEM,
an attacker VM, and a monitored Windows endpoint — running multiple
isolated VMs on a single physical machine.

## Hardware
- Mini PC, Intel N95, 16 GB RAM, NVMe storage
- Proxmox VE installed on bare metal (no host OS overhead)

## Design decisions
- **Proxmox over a Windows host:** ~1–2 GB hypervisor overhead vs 4 GB+
  for a desktop OS — on a 16 GB box, RAM is the binding constraint.
  Type-1 virtualization is also the standard   for this kind of infrastructure.
- **Sequential, not concurrent, workloads:** at 16 GB the lab runs its
  components in stages (SIEM first, attacker and victim VMs added as
  needed) rather than all at once — a deliberate resource-planning
  tradeoff, documented rather than discovered.

## First VM: Wazuh manager
- Ubuntu Server 26.04 LTS — 6 GB RAM, 2 vCPU, 64 GB disk
- Full VM (not LXC) to avoid OpenSearch container issues
- Snapshotted immediately post-install for fast rollback

## What's next
Attacker (Kali) and monitored-endpoint (Windows 11) VMs as the
detection-lab phase comes online.
