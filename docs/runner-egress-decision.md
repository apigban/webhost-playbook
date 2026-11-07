# Runner egress boundary — decision record (apigban/dmz01#13)

**Status:** implemented in `runner-egress.yml` (this repo). Deploys to dmz01 by the operator.
**Related:** dmz01#13 (ticket), PR #1 `feat/legacy-monitoring-plane` (dmz01#9 — firewalld ownership precedent).

---

## 1. Context and goal

Eleven GitHub Actions runners run on dmz01 as systemd units
(`actions.runner.apigban-<repo>.dmz01-<name>.service`, repos: austiles, brs,
dns-blocklist-stats, familyFinance, fitData, hermes-agent-config, project-atlas,
spell-capybara, suburb-scout, Super-Decision-Tree, tasker). Workflow job code
executes on the host as a single dedicated account and can currently reach
everything that account can: the LAN-facing tunnel routes, the legacy
monitoring plane, and loopback services on the host itself. Workflow secrets
(e.g. `OPENCODE_AUTH_JSON`) are readable by job code by design of the GitHub
Actions model. Goal: job-code egress is restricted to what CI legitimately
needs and is blocked from internal ranges and host services — or an
explicit accepted-risk record if that is not feasible.

This document records: the verified host facts, the research, the chosen
mechanism, the ownership split with firewalld, and the residual risks.

## 2. Verified host facts (2026-11-06/07, read-only inspection)

| Fact | Value |
| --- | --- |
| OS / kernel | Oracle Linux 9.8, aarch64 |
| firewalld | running, default zone `public` (iface `enp0s6`, source `192.168.2.0/24`), `trusted` zone on `tailscale0`; **nftables backend active** (`table inet firewalld` present in ruleset; firewalld 1.3.4) |
| nftables / iptables | nftables 1.0.9, iptables-nft 1.8.10; separate `ip filter`/`nat`/`mangle` tables exist (tailscale-era iptables-nft state); docker engine **inactive** |
| Runner units | all 11 units: `User=github-runner` (verified via `systemctl show … -p User`); live `Runner.Listener`/`Runner.Worker` processes run as UID **974** |
| github-runner privileges | no sudo/sudoers entries, **not in wheel**, no supplementary groups (`github-runner:x:974:`) |
| Host addresses | `enp0s6` 10.20.1.68/24 (DMZ subnet), `wg0` 10.200.0.1/32 (peer 10.200.0.253), `tailscale0` 100.96.131.75/32, v6 ULA `fd7a:115c:a1e0::e101:8365/128`; **no global v6** |
| Routed internal ranges | 192.168.2.0/24, 172.21.5.0/24, 172.16.5.0/24 via `wg0`; 10.20.1.0/24 on-link |
| DNS | `/etc/resolv.conf` is **tailscale-managed**: `100.100.100.100` (MagicDNS, inside 100.64.0.0/10) and `fd7a:115c:a1e0::53` (inside fc00::/7); systemd-resolved stubs 127.0.0.53/54 also listen |
| Host services reachable from local processes | agent server `:4096` (0.0.0.0), t3 `:3773` (0.0.0.0), speakr `:8899` (127.0.0.1 via pasta), legacy grafana `:3000` / prometheus `:9100` (`*` via rootlessport — retire/rebind pending PR #1) |

Two facts drive the design:

1. **All eleven runners share one dedicated user** (`github-runner`, UID 974),
   and that user cannot escalate (no sudo, no wheel). UID-based socket
   filtering is therefore viable, and the runner cannot change its own UID to
   escape the filter.
2. **DNS lives inside the ranges we must block.** A blanket CGNAT/ULA drop
   would silently kill name resolution for every CI job. The design needs an
   explicit, port-scoped DNS carve-out (Section 6).

## 3. Research findings

### (a) firewalld cannot match traffic by originating UID

firewalld's zone/rich-language model matches source address, destination
address, port, protocol, service, and icmp-block — there is no socket-UID
match. This is a known, open upstream gap: [firewalld RFE #725 "socket UID and
GID matching"](https://github.com/firewalld/firewalld/issues/725) (open since
2020, label `blocked` — it also requires upstream nftables support work; a
firewalld collaborator confirms "What's missing is `skuid` and `skgid`
matching"). The historical workaround — `firewall-cmd --direct` passthrough
with iptables `-m owner --uid-owner` — is **deprecated and unsupported on the
nftables backend**: RHEL 9's [Configuring firewalls and packet filters]
(https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_firewalls_and_packet_filters/using-and-configuring-firewalld_firewall-packet-filters)
states *"firewalld with nftables back end does not support passing custom
nftables rules to firewalld, using the --direct option"*. dmz01 runs
firewalld 1.3.4 with the nftables backend active, so the direct-interface path
is closed.

### (b) The sanctioned alternative: a separate nftables table with `meta skuid`

nftables has supported per-UID socket matching since long before RHEL 9:
`meta skuid` (UID of the socket owner, output path) is documented nftables
syntax ([nftables wiki](https://wiki.nftables.org/wiki-nftables/index.php/Configuring_chains),
[kb examples](https://kb.axx.no/nftables-cheatsheet/)). Crucially, nftables
tables are **namespaces that coexist**: *"With the nftables backend, firewalld
does not assume complete control over the host's firewalling. Since nftables
allows multiple 'namespaces' via tables"* ([ArchWiki:
Firewalld](https://wiki.archlinux.org/title/Firewalld)), and Red Hat
documents running nftables rules alongside firewalld as long as each service
manages its own tables ([RHEL 9: Getting started with
nftables](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_firewalls_and_packet_filters/getting-started-with-nftables_firewall-packet-filters)).

The interaction semantics that make this safe are formal in the
[nftables wiki](https://wiki.nftables.org/wiki-nftables/index.php/Configuring_chains):
packets traverse **every** base chain attached to a hook in priority order;
*"an accept verdict … isn't necessarily final. However, the same is not true
of packets that are subjected to a drop verdict … drops take immediate
effect, with no further rules or chains being evaluated."* A drop-only,
`policy accept` table on the output hook — at chain priority `filter` (0),
deliberately distinct from firewalld's own output chain (see below), where an
equal-priority tie would have meant registration-order evaluation — therefore:

- drops runner-UID packets to internal destinations regardless of what
  firewalld decided. firewalld's nft table **does** define an output base
  chain on this host: `filter_OUTPUT` (hook `output`, priority `filter + 10`
  = 10, policy `accept`). That does not weaken this boundary: a `drop` here
  is final across **all** base chains on the hook, so a packet we drop is
  never judged by firewalld's chain at all;
- never bypasses firewalld in the other direction either: the only `accept`
  verdicts we issue are the port-scoped DNS carve-outs inside our own chain,
  and an `accept` in one base chain is **not** final — traversal continues
  into firewalld's `filter_OUTPUT`, which keeps the final say (its policy is
  accept, so today a carved-out DNS packet proceeds; any future firewalld
  output rule would still apply to runner traffic). Neither chain can
  override the other's drops, whichever order they run in;
- survives firewalld start/reload/restart (firewalld only manages its own
  table), and vice versa: firewalld never flushes foreign tables.

The iptables `-m owner` via a dedicated chain was rejected because on this
host it would mean iptables-nft interacting with a firewalld-managed system —
exactly the double-management pattern the deprecated `--direct` interface
existed to gate-keep, with two rule grammars and two persistence stories. One
nft table + one systemd oneshot loader is smaller and fully owned by this
repo.

### (c) UID filtering is valid for these systemd service users

`systemctl show <unit> -p User` verified for all 11 units: `User=github-runner`
(one shared account, UID 974), and `ps` confirms the live `Runner.Listener` /
`Runner.Worker` processes run with UID 974. Job processes are children of
`Runner.Worker` and inherit that UID; `meta skuid` matches the socket
owner's UID, so **all** job-originated sockets — including those made by
rootless podman's slirp4netns/pasta helpers, which run as the user — carry
skuid 974 and are filtered. (Docker is inactive on this host, so there is no
root-UID bridge path that would escape the filter.)

The design additionally refuses to run if the resolved UID is 0 or matches a
protected operator account (assertion in the play) — a templating or naming
mistake cannot lock the operator out.

### (d) Prior art

- [GitHub docs: self-hosted runners reference](https://docs.github.com/en/actions/reference/runners/self-hosted-runners) — required domains (Section 5 below) and the note that runner hosts "must be able to make outbound HTTPS connections over port 443".
- [GitHub docs: security hardening](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions) — for self-hosted runners, asks: *"Does the machine have network access to sensitive services? For example, Azure or AWS metadata services."* Our block list includes 169.254.0.0/16 for exactly this reason.
- [step-security/harden-runner](https://github.com/step-security/harden-runner) — commercial prior art: per-job egress allowlisting/monitoring for GitHub Actions runners.
- [Ken Muse: Restricting IP Access on GitHub(-Hosted) Runners](https://www.kenmuse.com/blog/restricting-ip-access-on-github-hosted-runners/) — iptables/DNS-level egress restriction prior art.
- [Sysdig: threat actors using self-hosted runners as backdoors](https://www.sysdig.com/blog/how-threat-actors-are-using-self-hosted-github-actions-runners-as-backdoors) — the threat context for this ticket.

## 4. Decision

**Implemented: UID-scoped internal-destination blocklist in a separate
nftables table, audit-first rollout.** Full domain/IP **allowlisting was
rejected as fragile** on this architecture: GitHub requires wildcard domains
(`*.blob.core.windows.net`, `*.githubusercontent.com`) whose addresses are
unpublished and rotating (GitHub's docs note the records "might change in the
future"), npm/PyPI publish no stable IP ranges at all, and IP-range
allowlisting would break on every registry CDN change. A domain-allowlist
without an HTTP(S) egress proxy is not enforceable at the IP layer. The
ticket's stated requirement — runner jobs cannot reach LAN ranges and host
services while GitHub + registries + DNS keep working — is fully met by the
blocklist shape, with none of that fragility. A proxy-based allowlist remains
a documented future hardening step (Section 8).

Deliberate non-goals of this change:

- **No firewalld state is touched.** The boundary is a separate
  `inet runner_egress` table loaded by its own systemd oneshot
  (`runner-egress-nft.service` → `nft -f /etc/nftables.d/runner-egress.nft`).
- **No workflow changes.** Test jobs are provided as ready-to-commit specs
  (`files/runner-egress-tests/`); the operator commits them to a runner repo.

### Rules table (UID 974, `github-runner` only)

| Traffic from job code (UID 974) | AUDIT mode (default) | ENFORCE mode |
| --- | --- | --- |
| udp/tcp **53** → 100.100.100.100, fd7a:115c:a1e0::53, 127.0.0.53, 127.0.0.54 | **ALLOW** (carve-out, port-scoped) | **ALLOW** |
| 127.0.0.0/8 — host loopback services (:4096, :3773, :8899, :3000, :9100 …) | counter + log | **DROP** |
| 10.0.0.0/8 — DMZ subnet 10.20.1.0/24, wg host/peer 10.200.0.x | counter + log | **DROP** |
| 172.16.0.0/12 — wg routes 172.21.5.0/24 (web hosts), 172.16.5.0/24 (wg peer) | counter + log | **DROP** |
| 192.168.0.0/16 — LAN range incl. 192.168.2.0/24 | counter + log | **DROP** |
| 100.64.0.0/10 — tailscale CGNAT (peers + this host's 100.96.131.75) except the :53 carve-out | counter + log | **DROP** |
| 169.254.0.0/16 — link-local + cloud metadata | counter + log | **DROP** |
| 0.0.0.0/8, 224.0.0.0/4, 255.255.255.255/32 — bogons | counter + log | **DROP** |
| ::1/128, fc00::/7 (tailscale ULA incl. host v6), fe80::/10 | counter + log | **DROP** |
| Everything else (github.com, api.github.com, artifacts/caches, npmjs, PyPI, gemini API, Discord webhook, z.ai …) | pass | **pass** (untouched by this boundary; firewalld's `filter_OUTPUT` on this hook is policy accept, so the overall verdict stays accept) |
| **All other UIDs** (operator, ansible, web services, root) | **not inspected** | **not inspected** |

Notes: there is **no** `ct state established` exception for internal ranges —
a job cannot keep a pre-existing internal connection alive through a deploy.
Internal-name resolution still succeeds (DNS carve-out) but the resulting
connections are dropped; that leaks internal *names* to jobs, not reachability.

## 5. Legit CI egress reference (what the boundary must keep working)

From [GitHub's self-hosted runner requirements](https://docs.github.com/en/actions/reference/runners/self-hosted-runners), essential + used-by-these-workflows:
`github.com`, `api.github.com`, `*.actions.githubusercontent.com`,
`codeload.github.com`, `results-receiver.actions.githubusercontent.com`,
`*.blob.core.windows.net` (artifacts/caches), `objects*.githubusercontent.com`
(runner updates), `ghcr.io`/`*.pkg.github.com` (packages), plus DNS and the
workflow-level endpoints the audit found: `registry.npmjs.org`,
`pypi.org`, `generativelanguage.googleapis.com` (agy jobs), the Z.AI/opencode
model API (`OPENCODE_MODEL` plane), `downloads.rclone.org` + R2 endpoint
(familyFinance recompute — GitHub-hosted runner, unaffected), `discord.com`
webhooks (public SaaS). None are internal; all pass untouched.

## 6. DNS carve-out rationale

`/etc/resolv.conf` is tailscale-managed (verified on host; Tailscale's
resolvconf overwrite). The system resolver is `100.100.100.100` — **inside**
100.64.0.0/10 — with `fd7a:115c:a1e0::53` (inside fc00::/7) as v6 secondary,
and systemd-resolved stubs on 127.0.0.53/54 as the local fallback. The
carve-out allows **udp/tcp destination port 53 only** to those four addresses;
everything else to those IPs is still dropped (test job A's last step proves
the port scoping). If the host's DNS story changes, the carve-out list is a
play variable — one-line change, re-run.

## 7. Firewalld ownership record (acceptance criterion)

The ticket asks that firewalld ownership be confirmed and recorded. Confirmed
(evidence in PR #1's analysis, re-verified 2026-11-07):

- On `main` today, **neither the firewalld config nor the bootstrap crontab
  has an in-repo owner** — they were applied ad-hoc. PR #1
  (`feat/legacy-monitoring-plane`) establishes the first recorded firewalld
  owner (`legacy-monitoring.yml`).
- **This PR deliberately adds zero firewalld state** and instead records the
  split below. Runner egress is *not* a firewalld concern by capability
  (no UID matching — Section 3a) and by policy (zones describe inbound
  posture; this is an outbound, per-UID boundary).

| Surface | Recorded owner (after PR #1 + this PR merge) |
| --- | --- |
| firewalld zones/ports (public/trusted) on dmz01 | `legacy-monitoring.yml` (PR #1) + galaxy roles (`rhel-9.3-base` installs firewalld; `devbox-role`/`hugo_webhost` own their ports) |
| Bootstrap podman crontab on dmz01 | `legacy-monitoring.yml` (PR #1) |
| `inet runner_egress` nft table, `/etc/nftables.d/runner-egress.nft`, `runner-egress-nft.service` | `runner-egress.yml` (this PR) |
| `/etc/resolv.conf` | tailscale (host-managed; carve-outs keyed to it here) |
| Caddyfile + Caddy container | `main.yml` second play (pre-existing) |

## 8. Residual risks / accepted-risk register

1. **Public-internet exfiltration remains possible.** The boundary is
   internal-only by design: a compromised job can still talk to public hosts
   (including the `DISCORD_WEBHOOK_URL` — a public endpoint). Accepting a
   blocklist here; full allowlisting requires an HTTP(S) egress proxy (future
   hardening: squid/mitm-style forward proxy + `meta skuid` allow-only-to-proxy).
2. **UID escape requires privilege escalation.** github-runner has no sudo,
   is not in wheel, has no supplementary groups (verified). Remaining paths
   (setuid-root binaries requiring auth, kernel exploits) imply host
   compromise independent of this boundary. Accepted.
3. **Structured-JSON secrets weaken redaction** (GitHub's own guidance:
   don't store JSON blobs as secrets — `OPENCODE_AUTH_JSON` is exactly that
   shape). See `docs/runner-secrets-audit.md` for per-repo dispositions;
   compensations: rotation, scratch-HOME provisioning, restrictive
   `OPENCODE_CONFIG_CONTENT` permissions in the workflows.
4. **Internal-name leak.** Jobs can resolve internal hostnames via MagicDNS;
   connections are dropped. Reachability is the defended asset.
5. **Host-address drift.** Host services bound to 0.0.0.0 are currently
   covered only because every host address falls in a blocked range
   (10.20.1.68 → 10/8; 100.96.131.75 → 100.64/10; fd7a:… → fc00::/7). If
   enp0s6 ever carries a public address, host-unicast access from jobs would
   open. Test job B probes host-unicast targets, so drift is caught; remedy
   would be an explicit host-address drop set in the template.
6. **Operator `nft flush ruleset`** removes the boundary (and everything
   else). Remedy: `systemctl restart runner-egress-nft` or re-run the play.
   firewalld restart does **not** affect the table.
7. **Future CI needing internal egress** (e.g. a deploy job hitting an
   internal host) will fail visibly in enforce mode; audit-mode counters and
   this register document how to carve out deliberately — via a play variable,
   i.e. an infrastructure decision, not a workflow edit. That friction is
   intended.

## 9. Rollout and verification (operator)

1. Deploy play (default **AUDIT**): nothing is dropped; watch
   `nft list chain inet runner_egress runner_policy` counters and
   `journalctl -k --grep 'egress-audit'` while normal CI runs. Expect only
   internal-range attempts (scanners, misconfigured tools) to appear.
2. Commit the two test workflows (`files/runner-egress-tests/`) to one
   runner repo and run them:
   - Test job A (legit egress): must be **GREEN in audit mode** — proves the
     carve-outs keep CI functional.
   - Test job B (internal block): expected **RED in audit mode** (targets
     still reachable) — it exists to prove the *negative* once enforced.
3. Promote: re-run with `-e runner_egress_enforce=true`. Re-run test jobs:
   A stays green, B turns green. Rollout complete.
4. Rollback: `-e runner_egress_state=absent` (or
   `systemctl disable --now runner-egress-nft.service`), then re-run test
   job A to confirm CI unaffected.

## 10. Merge order

Merge PR #1 first. The two changes are functionally independent (this PR
touches no firewalld state and no Caddyfile; its only group_vars addition is
the two `runner_egress_*` knob defaults, which do not interact with PR #1's
knobs); the only shared surface is a README section (adjacency). If PR #1's
docs land first, this branch rebases trivially. No coordination needed on
rules: PR #1's zone-port removals and this output-hook table never touch the
same state.
