# RUNBOOK — Caddy secret rotation (Namecheap API key + CrowdSec bouncer key)

Applies to: `dmz01` (proxies group), managed by `main.yml` play
*"Configure Caddy Reverse Proxy"* rendering `Caddyfile-dmz.j2`.

**Why this exists:** the live Caddyfile, plus `Caddyfile.bak` and
`Caddyfile-2026-01-11` in the same volume, carried the Namecheap API key and the
CrowdSec bouncer key in plaintext. Treat **every previously deployed key as
exposed** and rotate. (refs apigban/dmz01#8)

**What this runbook rotates:**

| Secret | Where it lives | Revocation mechanism |
|---|---|---|
| Namecheap API key | Namecheap console → Profile → Tools → Namecheap API Access | Resetting the key in the console invalidates the old key account-wide (one key per account) |
| CrowdSec LAPI bouncer key | `cscli bouncers` inside the `crowdsec` container | `cscli bouncers delete <name>` removes the DB entry; the old key stops authenticating |

**Critical constraint:** the Namecheap key is **IP-pinned** — it only works from
`namecheap_client_ip` (dmz01's public IP), which must stay whitelisted in the
Namecheap console. Rotation must preserve that pin, otherwise DNS-01 renewals
fail and TLS lapses when cached certs expire (~30 days of LE grace).

All host commands below run **on dmz01, as `contsvc`**, unless noted.

---

## 0. Preconditions

- **KNOWN BLOCKER (verify first):** as of 2026-11-07, caddy's DNS-01 attempts fail with
  `namecheap api returned error: Cannot complete this command as this domain is not using
  proper DNS servers` — the `apigban.com` zone is currently delegated to non-Namecheap
  nameservers (the public site is fronted by Cloudflare). DNS-01 via the Namecheap API
  will fail with this error **regardless of key freshness** until either the zone is
  re-delegated to Namecheap's nameservers, or caddy's DNS provider is switched to the
  one actually serving the zone. Rotating the key is still required (exposure), but a
  green `certificate obtained successfully` additionally needs this fixed.
- `git pull` the latest `webhost-playbook` on your Ansible control host.
- Confirm caddy + crowdsec containers are healthy:
  ```bash
  podman ps --format '{{.Names}} | {{.Status}}' | grep -E 'caddy|crowdsec'
  ```
- Snapshot the current cert dates for later comparison:
  ```bash
  for d in alain.apigban.com pingvin.apigban.com; do
    echo -n "$d: "; echo | openssl s_client -connect "$d:443" -servername "$d" 2>/dev/null \
      | openssl x509 -noout -dates
  done
  ```

## 1. Mint the new CrowdSec bouncer key

The key is displayed **exactly once** — capture it immediately into your
password manager, never into files, chat, or commit messages.

```bash
podman exec crowdsec cscli bouncers add caddy-ansible
# API key for 'caddy-ansible': <NEW_CROWDSEC_KEY>   <- shown once, copy it now
```

Both the old and the new key are valid from this moment (zero downtime); the
new key is simply unused until caddy is redeployed. Smoke-test it:

```bash
curl -sS -o /dev/null -w 'new key: %{http_code}\n' \
  -H "X-Api-Key: <NEW_CROWDSEC_KEY>" http://localhost:8080/v1/decisions
# expect: new key: 200   (403 would mean a bad key)
```

## 2. Rotate the Namecheap API key (console, preserves the IP pin)

1. Log in to Namecheap → **Profile → Tools → Namecheap API Access → MANAGE**.
2. **Before** resetting: confirm the **Whitelisted IPs** list still contains
   dmz01's public IP (`namecheap_client_ip` from `inventory/secrets.yml`).
   Resetting the key does **not** clear the whitelist — verify anyway; the new
   key is useless without the pin.
3. Reset/regenerate the API key per the console's *Resetting the API key*
   flow. Copy the new key into your password manager.
4. Smoke-test the new key **from dmz01** (the whitelisted IP) *before*
   deploying, using a read-only command:
   ```bash
   curl -sS 'https://api.namecheap.com/xml.response?ApiUser=<namecheap_user>&ApiKey=<NEW_NAMECHEAP_KEY>&UserName=<namecheap_user>&Command=namecheap.domains.check&ClientIp=<namecheap_client_ip>&DomainList=apigban.com'
   ```
   Expect `Status="OK"` and `Available="true"` for the domain. `Status="ERROR"`
   with an IP/whitelist error means the pin is broken — stop and fix step 2.2.
5. Note: from this moment the **old** Namecheap key is dead. Caddy only calls
   the Namecheap API during DNS-01 renewals, so nothing breaks immediately —
   proceed to step 3 **immediately** to shrink the window in which the on-disk
   Caddyfile holds a dead key.

## 3. Vault the new values

On the Ansible control host (never on dmz01, never in git):

```bash
cd webhost-playbook
ansible-vault edit inventory/secrets.yml        # or create from the sample first
# update:
#   namecheap_api_key: <NEW_NAMECHEAP_KEY>
#   crowdsec_api_key:  <NEW_CROWDSEC_KEY>
# (namecheap_email / namecheap_user / namecheap_client_ip / crowdsec_api_url unchanged)
```

`inventory/secrets.yml` must be the whole-file-encrypted vault. The play's
preflight assert **fails on purpose** if any Caddy secret is missing or still a
`XXXX…` placeholder — that guard exists so caddy never gets rendered with
broken keys. Never bypass it by `-e` overriding values on the CLI.

## 4. Deploy

```bash
ansible-navigator run --eei localhost/role-dev-ee -m stdout \
  main.yml -i inventory/inventory.yml \
  --vault-id prod@prompt
# or plain ansible-playbook with --vault-id, per repo README
```

The play now:
1. asserts real secrets are present (preflight),
2. renders `Caddyfile-dmz.j2` to a **staging** file inside the caddy volume
   (`0640 contsvc:contsvc`) — the live path is not touched yet,
3. runs `caddy validate` inside the container against the **staged** file — a
   broken render fails the block, the staged copy is deleted, and the live
   Caddyfile (and the running caddy / reboot crontab) keep the old config,
4. atomically renames the validated staging file over the live path (`mv`
   within the volume, preserving `0640 contsvc:contsvc`), and only then
   restarts caddy via the handler,
5. deletes the plaintext-key fossils (`caddy_caddyfile_fossils` in
   `inventory/group_vars/all/all.yml`); the reload runs only if a validated
   config was actually promoted.

Preview the fossil purge without touching anything:
`ansible-playbook main.yml -i inventory/inventory.yml --vault-id prod@prompt --check`

## 5. Post-deploy verification (all checks must pass before revocation)

```bash
# 5a. caddy restarted cleanly and reloaded the new config
podman ps --format '{{.Names}} {{.Status}}' | grep caddy
podman logs caddy --since 10m 2>&1 | grep -iE 'error|invalid' | grep -v 'notice' || true

# 5b. crowdsec plugin authenticates with the NEW key (LAPI pull is fresh)
podman exec crowdsec cscli bouncers list
#   'caddy-ansible' must show a recent "Last API Pull" (ticker_interval is 15s)

# 5c. no auth failures in caddy logs (bad-key 403s would show here)
podman logs caddy --since 15m 2>&1 | grep -iE 'access forbidden|unauthorized|crowdsec.*(403|error)' || echo OK

# 5d. sites still 200 (note: alain goes through Cloudflare edge — see 5f note)
curl -sSI https://alain.apigban.com   | head -1     # expect HTTP/2 200
curl -sSI https://pingvin.apigban.com | head -1     # expect HTTP/2 200

# 5e. certs still valid (fresh DNS-01 proof comes from 5f or the next renewal)
for d in alain.apigban.com pingvin.apigban.com; do
  echo -n "$d: "; echo | openssl s_client -connect "$d:443" -servername "$d" 2>/dev/null \
    | openssl x509 -noout -dates -issuer
done
```

**5f. Optional but recommended — force a fresh DNS-01 issuance with the new
Namecheap key** (otherwise proof arrives at the next renewal, ~60 days):

> NOTE: this only proves anything once the zone-delegation blocker in step 0 is
> resolved. Also, `https://alain.apigban.com` is served through Cloudflare — a
> public 200 there proves the edge, not this host. Verify caddy itself via its
> own logs and the LAPI checks below.

```bash
# CAREFUL: production TLS. Do it in a quiet minute; brief downtime per restart.
podman stop caddy
podman exec caddy ls /data/caddy/certificates 2>/dev/null || \
  ls ~/.local/share/containers/storage/volumes/caddy_data/_data/caddy/certificates/*/
# move (do NOT delete) the LE cert storage for reissue:
mv ~/.local/share/containers/storage/volumes/caddy_data/_data/caddy/certificates \
   ~/.local/share/containers/storage/volumes/caddy_data/_data/caddy/certificates.pre-rotation
podman start caddy
podman logs -f caddy 2>&1 | grep -E 'challenge_type|certificate obtained successfully'
# expect per site:  "challenge_type":"dns-01"  then "certificate obtained successfully"
```

If issuance fails with a Namecheap/whitelist error, the pin or the new key is
wrong: restore the moved directory (`mv … .pre-rotation` back), fix step 2,
re-deploy.

## 6. Revoke the old keys — only after ALL of step 5 passes

Revocation order: **new key proven working first, old key killed second.**

```bash
# 6a. CrowdSec: delete the OLD bouncer entry (kills the old key)
podman exec crowdsec cscli bouncers list          # identify the old entry name
podman exec crowdsec cscli bouncers delete <old-bouncer-name>
podman exec crowdsec cscli bouncers list          # only 'caddy-ansible' remains, Last API Pull fresh

# 6b. confirm the old crowdsec key now 403s (substitute the OLD key from your
#     password manager; do not paste it anywhere persistent)
curl -sS -o /dev/null -w 'old key: %{http_code}\n' \
  -H "X-Api-Key: <OLD_CROWDSEC_KEY>" http://localhost:8080/v1/decisions
# expect: old key: 403
```

- The old **Namecheap** key needs no further action: resetting it in step 2
  already invalidated it (Namecheap keeps one active API key per account).

## 7. Close out

1. Fossils gone from the volume:
   ```bash
   ls ~/.local/share/containers/storage/volumes/caddy_etc/_data/
   # expect only: Caddyfile  (plus corazawaf/)
   ```
2. Update the dmz01 secrets/ops documentation with the new key locations and
   rotation date (operator action in the **dmz01 repo** — do not store the
   values themselves, only references).
3. Record the rotation date in your password manager entries for both keys.

## Rollback

- Repo plumbing (template/tasks) is two-way: `git revert` and re-run — the
  vaulted values in `inventory/secrets.yml` stay authoritative either way.
- If caddy breaks post-deploy: the pre-deploy `Caddyfile` content is not
  kept by Ansible, so roll **forward**: fix the vault value and re-run
  `main.yml` (the play re-renders + validates + reloads). If only cert storage
  was moved in step 5f, move it back and `podman restart caddy`.
