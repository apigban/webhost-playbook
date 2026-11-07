# Workflow secrets-exposure audit — 11 runner repos on dmz01 (apigban/dmz01#13)

**Audited:** 2026-11-07, via the GitHub API (`.github/workflows` contents of all
11 repos). 30 workflow files reviewed. **No secret values appear anywhere in
this document — names only**, per ticket instruction.

**Exposure model:** any repo/organization secret referenced by a workflow that
runs on a dmz01 runner (`runs-on: [self-hosted, agy]`) is readable by that
job's code as environment/context data — GitHub executes the job as the
dedicated `github-runner` user (UID 974) on dmz01, so job code has full access
to every secret the workflow carries. This is inherent to the Actions model
([GitHub docs](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions):
"Any user with write access to your repository has read access to all secrets
configured in your repository"); the runner-egress boundary
(`runner-egress.yml`) limits where job code can *send* what it reads — it
cannot and does not limit what it can read.

## Headline findings

1. **Three secrets are exposed to dmz01 job code** (beyond the automatic
   `GITHUB_TOKEN`): `OPENCODE_AUTH_JSON` (7 repos), `DISCORD_WEBHOOK_URL`
   (4 repos), and the per-job `GITHUB_TOKEN` itself (all self-hosted jobs).
2. **No workflow running on dmz01 references internal-network endpoints** —
   no `git.home.apigban.com`, no RFC1918/CGNAT/tailscale addresses, no
   localhost service ports. All job egress is public (GitHub, npm/PyPI,
   Gemini API, Z.AI model API, Discord). **The egress boundary therefore
   breaks no existing CI.**
3. **No `pull_request_target` triggers anywhere** — no fork-privileged
   workflows. `permissions:` blocks are minimal (`contents: read`,
   `pull-requests: write`, or `{}` with job-level scoping).
4. Secrets used only by **GitHub-hosted** runners (`ubuntu-latest` /
   `ubuntu-24.04`) are **not** exposed to dmz01 job code at all: R2 keys
   (familyFinance), `PROJECT_SYNC_TOKEN` (hermes-agent-config, suburb-scout).
   They are out of scope for this boundary but listed for completeness.
5. `OPENCODE_AUTH_JSON` is a **structured JSON blob** secret — GitHub's
   hardening guidance explicitly warns against this shape because log
   redaction depends on exact-value matching and structured blobs defeat it
   ("Never use structured data as a secret"). The workflows mitigate by
   writing it to a scratch `$RUNNER_TEMP` home with `chmod 600` and never
   echoing it, but redaction risk remains. **Disposition: accepted with
   rotation; consider splitting into individual registered secrets** per
   GitHub guidance.

## Per-repo audit table

| Repo | Workflow (runner) | Secrets referenced | Exposed to dmz01 job code? | Disposition |
| --- | --- | --- | --- | --- |
| austiles | opencode-review.yml (self-hosted agy) | GITHUB_TOKEN, OPENCODE_AUTH_JSON | **Yes** | Egress boundary limits exfil routes to public hosts; auth.json → scratch `$RUNNER_TEMP` home, mode 600, never logged; rotate on schedule |
| austiles | test.yml (ubuntu-latest) | none | No | — |
| brs | opencode-review.yml (self-hosted agy) | GITHUB_TOKEN, OPENCODE_AUTH_JSON | **Yes** | As austiles |
| dns-blocklist-stats | opencode-review.yml (self-hosted agy) | GITHUB_TOKEN, OPENCODE_AUTH_JSON | **Yes** | As austiles |
| dns-blocklist-stats | test.yml (ubuntu-latest) | none | No | — |
| familyFinance | opencode-review.yml (self-hosted agy) | GITHUB_TOKEN, OPENCODE_AUTH_JSON | **Yes** | As austiles; workflow carries explicit prompt-injection provenance guard for AGENTS.md/opencode.json content |
| familyFinance | recompute.yml (ubuntu-latest) | R2_ACCESS_KEY_ID, R2_ACCOUNT_ID, R2_SECRET_ACCESS_KEY | No (GitHub-hosted runner) | Out of boundary scope; least-privilege R2 token recommended |
| familyFinance | test.yml (ubuntu-latest) | none | No | — |
| fitData | agy-review.yml (self-hosted agy) | GITHUB_TOKEN | **Yes** | Token scoped `contents: read`, `pull-requests: write`; API-only review, no checkout |
| hermes-agent-config | agy-review.yml (self-hosted agy) | GITHUB_TOKEN | **Yes** | As fitData |
| hermes-agent-config | agy-triage.yml (self-hosted agy) | GITHUB_TOKEN | **Yes** | As fitData |
| hermes-agent-config | agy-healthcheck.yml (self-hosted agy) | DISCORD_WEBHOOK_URL | **Yes** | Webhook is a public endpoint — egress boundary does not (and should not) block it; rotation requires a Discord action. Accepted |
| hermes-agent-config | sync-project-status.yml (ubuntu-latest) | PROJECT_SYNC_TOKEN | No (GitHub-hosted runner) | Out of boundary scope |
| hermes-agent-config | test.yml (ubuntu-latest) | none | No | — |
| project-atlas | opencode-review.yml (self-hosted agy) | GITHUB_TOKEN, OPENCODE_AUTH_JSON | **Yes** | As austiles |
| project-atlas | verify.yml (ubuntu-latest) | none | No | — |
| spell-capybara | opencode-review.yml (self-hosted agy) | GITHUB_TOKEN, OPENCODE_AUTH_JSON | **Yes** | As austiles |
| spell-capybara | test.yml (ubuntu-24.04) | none | No | — |
| suburb-scout | agy-review.yml (self-hosted agy) | GITHUB_TOKEN | **Yes** | As fitData |
| suburb-scout | agy-triage.yml (self-hosted agy) | GITHUB_TOKEN | **Yes** | As fitData |
| suburb-scout | agy-healthcheck.yml (self-hosted agy) | DISCORD_WEBHOOK_URL | **Yes** | As hermes-agent-config |
| suburb-scout | sync-project-status.yml (ubuntu-latest) | PROJECT_SYNC_TOKEN | No (GitHub-hosted runner) | Out of boundary scope |
| suburb-scout | test.yml (ubuntu-latest) | none | No | — |
| Super-Decision-Tree | opencode-review.yml (self-hosted agy) | GITHUB_TOKEN, OPENCODE_AUTH_JSON | **Yes** | As austiles |
| Super-Decision-Tree | agy-triage.yml (self-hosted agy) | GITHUB_TOKEN | **Yes** | As fitData |
| Super-Decision-Tree | agy-healthcheck.yml (self-hosted agy) | DISCORD_WEBHOOK_URL | **Yes** | As hermes-agent-config |
| Super-Decision-Tree | test.yml (ubuntu-latest) | none | No | — |
| tasker | agy-review.yml (self-hosted agy) | GITHUB_TOKEN | **Yes** | As fitData |
| tasker | agy-triage.yml (self-hosted agy) | GITHUB_TOKEN | **Yes** | As fitData |
| tasker | agy-healthcheck.yml (self-hosted agy) | DISCORD_WEBHOOK_URL | **Yes** | As hermes-agent-config |

## GITHUB_TOKEN note

Every self-hosted job uses `GITHUB_TOKEN` (either `secrets.GITHUB_TOKEN` or
the `github.token` context) — automatically provisioned per run, scoped by
each workflow's `permissions:` block (observed: `contents: read`,
`pull-requests: write`, or empty at workflow level with job-level scoping).
A leaked token is short-lived and single-scope. No changes needed; noted so
the boundary's value is clear: the *long-lived* credentials above are the
assets the egress rule protects by limiting where job code can send them.

## Compensating controls matrix

| Control | Status |
| --- | --- |
| Runner egress boundary (internal ranges + host services unreachable from job code) | This PR — audit mode default, enforce on promotion |
| No fork-privileged triggers (`pull_request_target`) | Verified absent in all 30 workflows |
| Minimal `permissions:` scopes | Present in all audited workflows |
| Secret values never logged by workflows | Verified in review (auth.json written to scratch HOME, chmod 600; no `echo`/`set -x` of secret values observed) |
| Rotation cadence for OPENCODE_AUTH_JSON / DISCORD_WEBHOOK_URL | **Operator action — recommend establishing** |
| Split OPENCODE_AUTH_JSON into individually registered secrets | **Operator action — recommend evaluating** per GitHub structured-secret guidance |
