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

### Legacy Monitoring Plane (dmz01)

`legacy-monitoring.yml` retires (default) or loopback-rebinds the pre-Alloy local
prometheus/grafana containers and removes their firewalld ports plus the
listener-free `19122/udp` entry. `9101/tcp` is **not** stale — `node_exporter`
listens on it — and is only closed when you set
`legacy_monitoring_remove_nodeexporter_firewall_port=true` after confirming
nothing scrapes node_exporter over that port. The decision knob is
`legacy_monitoring_enabled` (group_var, default `false` = retire; data volumes
preserved). Rationale and blast radius: [docs/legacy-monitoring-decision.md](docs/legacy-monitoring-decision.md).

```bash
ansible-navigator run --eei localhost/role-dev-ee -m stdout legacy-monitoring.yml -i inventory/inventory.yml
# override to loopback-rebind instead of retire:
ansible-navigator run --eei localhost/role-dev-ee -m stdout legacy-monitoring.yml -i inventory/inventory.yml -e legacy_monitoring_enabled=true
```

## Security

- OS hardening via devsec.hardening role
- Web Application Firewall (Coraza)
- CrowdSec integration
- Automated SSL/TLS management
- Separate access logs per domain