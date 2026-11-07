# Web Infrastructure Automation

An Ansible playbook for automating the deployment and configuration of web infrastructure components.

## Purpose

This playbook automates three main tasks:
1. Configures web hosts with base configuration and security hardening
2. Deploys static files to their designated paths
3. Sets up an external reverse proxy with domain-specific routing rules

## Quick Start

1. Set up configuration:
   ```bash
   # Copy and configure secrets
   cp inventory/secrets.yml.sample inventory/secrets.yml
   
   # Install required roles
   ansible-galaxy install -r requirements.yml -p roles
   ```

2. Run the playbook:
   ```bash
   ansible-navigator run --eei localhost/role-dev-ee -m stdout main.yml -i inventory/inventory.yml
   ```

## Components

### Web Host Configuration
- Base RHEL 9.3 setup
- OS hardening
- Network configuration
- Static file deployment paths

### Reverse Proxy Features
- Automated SSL/TLS via Namecheap DNS
- Load balancing with health checks
- WAF with CrowdSec integration
- Per-domain logging
- Custom routing rules

## Security

- OS hardening via devsec.hardening role
- Web Application Firewall (Coraza)
- CrowdSec integration
- Automated SSL/TLS management
- Separate access logs per domain

### Runner egress boundary (dmz01)

`runner-egress.yml` restricts what GitHub Actions job code on dmz01 can reach:
the `github-runner` user is blocked from LAN/WireGuard/tailscale ranges and the
host's own services, while GitHub, package registries, and DNS keep working.
Deployed as a separate nftables table (firewalld is not touched); **audit mode
is the default** — promote with `-e runner_egress_enforce=true` after reviewing
`nft list chain inet runner_egress runner_policy` and
`journalctl -k --grep 'egress-audit'`. Design, research citations, ownership
record, and residual risks: `docs/runner-egress-decision.md`. Secrets-exposure
audit of the 11 runner repos: `docs/runner-secrets-audit.md`. Ready-to-commit
verification workflows: `files/runner-egress-tests/`.