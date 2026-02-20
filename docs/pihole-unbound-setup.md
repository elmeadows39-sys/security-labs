# Pi-hole + Unbound: Recursive DNS Setup

## Overview
Pi-hole handles network-wide DNS filtering and ad blocking. Unbound runs alongside it as a recursive DNS resolver, eliminating reliance on upstream providers like Google (8.8.8.8) or Cloudflare (1.1.1.1) for privacy and control.

## Why This Setup
Using a third-party DNS provider means your DNS queries are logged by that provider. Unbound resolves DNS recursively by querying root servers directly, keeping DNS traffic internal to the lab.

## Architecture
Client Devices → Pi-hole (filtering) → Unbound (recursive resolver) → Root DNS Servers

## Deployment
- Both services deployed as separate LXCs on Proxmox VE
- Pi-hole: LXC 101 — 192.168.1.225
- Unbound: LXC 102 — 192.168.1.84
- Pi-hole configured to use Unbound as its sole upstream resolver

## Configuration
Pi-hole upstream DNS set to Unbound's IP (192.168.1.84) on port 5335.
Unbound access restricted to local subnet (192.168.1.0/24) only — external queries rejected.

## Result
All DNS queries on the network are filtered by Pi-hole and resolved recursively by Unbound with no third-party DNS provider involved.

## Future Improvements
- Enable DNSSEC validation on Unbound for cryptographic verification of DNS responses
- Note: DNSSEC requires careful testing to avoid breaking DNS resolution — plan a maintenance window
