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