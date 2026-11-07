# Decision record: retire vs rebind the legacy monitoring plane (dmz01)

- **Status**: recorded (defaults applied in this repo); operator can override at deploy time
- **Ticket**: apigban/dmz01#9
- **Decision context**: dmz01 operator manual, ADR-0001 (*Grafana Cloud as monitoring backend*)
- **Knob**: `legacy_monitoring_enabled` in `inventory/group_vars/all/all.yml` (default `false`)
- **Executor**: `legacy-monitoring.yml` (targets `dmz01`)

## Context

dmz01 runs two monitoring planes:

1. **Alloy → Grafana Cloud** (`alloy.service`) — the active pipeline and system of record (ADR-0001).
2. **Legacy local containers** (rootless podman under `contsvc`) — pre-Alloy prometheus (`0.0.0.0:9100`, `--web.enable-admin-api` on, scrape jobs commented out) and grafana (`0.0.0.0:3000`, static admin password). Both bind to **all interfaces**: every device on the tailnet can reach an unauthenticated admin API and a statically-passworded dashboard. The 2026-11-06 audit also found firewalld carrying `9101/tcp` and `19122/udp` with **no listener behind them** — stale entries that make the audit story lie.

## Options

| | A — Retire (default) | B — Rebind |
|---|---|---|
| Containers | stopped + removed | re-created bound to `127.0.0.1` only |
| Prometheus admin API | moot (gone) | flag removed (`--web.enable-admin-api` off) |
| firewalld | `3000/tcp`, `9100/tcp` removed | also kept closed (loopback needs no rule) |
| Stale `9101/tcp`, `19122/udp` | removed | removed |
| Bootstrap crontab | starts only caddy/crowdsec/opencode | keeps starting both (loopback-bound) |
| Data volumes (`prometheus`, `grafana`) | **preserved** unless `legacy_monitoring_purge_volumes: true` | preserved |
| Tailnet exposure | eliminated | eliminated (loopback only) |
| Attack surface kept on host | none | two long-running containers with static grafana auth |

## Decision

**Default: Option A (retire).** Rationale:

- ADR-0001 names Grafana Cloud as the system of record and calls the local stack "legacy … not the alerting surface"; its `node` scrape jobs are commented out, so it monitors almost nothing.
- Retiring removes the only unauthenticated-mutating surface (`--web.enable-admin-api` exposes `/api/*/admin/tsdb/delete_series` and snapshot endpoints to anyone who can reach :9100) rather than shrinking it.
- Every running container is RAM/CPU spent on a shared 4 OCPU host for zero active use.
- Fewer long-running things = fewer things for the audit to re-explain next time.

**Override**: set `legacy_monitoring_enabled: true` (host_var/group_var or `-e`) if the operator wants local-only dashboards kept; the playbook then rebinds both containers to loopback and drops the admin API flag. This is recorded in the same knob — one deploy-time choice, no fork in the codebase.

## Safety / blast radius

- **Two-way door.** Container removal is reversible: the named podman volumes `prometheus` (config + TSDB) and `grafana` (data + `grafana.ini`) are **preserved** by the retire path. A later `legacy_monitoring_enabled: true` run re-creates both containers against the preserved volumes with identical mounts (`type=volume` + `subpath`, matching the audited originals). Only `legacy_monitoring_purge_volumes: true` (default `false`) destroys data, and it is opt-in.
- **Firewalld is asymmetric-safe**: only the four monitoring/stale ports are touched; `51820/udp` (wireguard) and `41641/udp` (tailscale) are left alone; `https`, `ssh`, `dhcpv6-client` services and the `192.168.2.0/24` source are untouched.
- **Crontab**: single Ansible cron job under the existing `#Ansible: start podman containers` marker; `caddy → crowdsec → opencode` order preserved in both modes.

## Post-deploy verification (operator)

From a **second tailnet device**:

```
curl -m 5 http://<dmz01-tailnet-ip>:3000/    # expect: refused / timeout
curl -m 5 http://<dmz01-tailnet-ip>:9100/    # expect: refused / timeout
```

On dmz01 (read-only):

```
crontab -l                                   # job matches chosen mode
sudo firewall-cmd --list-ports --zone=public # no 3000/tcp, 9100/tcp, 9101/tcp, 19122/udp
podman ps                                    # (retire) no prometheus/grafana; caddy/crowdsec/speakr intact
```

## References

- dmz01 docs: `adr/0001-grafana-cloud-monitoring-backend.md`, `monitoring.md`
- Prometheus security model — `--web.enable-admin-api` enables unauthenticated mutating endpoints: <https://prometheus.io/docs/operating/security/>
- podman-run(1) `--publish`: hostIP omitted binds **all** host addresses; explicit `127.0.0.1` binds loopback only: <https://docs.podman.io/en/latest/markdown/podman-run.1.html>
- podman-run(1) `--mount type=volume,…,subpath=`: <https://docs.podman.io/en/latest/markdown/podman-run.1.html>
- ansible.posix.firewalld module (permanent + immediate, `state: disabled`): <https://docs.ansible.com/projects/ansible/latest/collections/ansible/posix/firewalld_module.html>
