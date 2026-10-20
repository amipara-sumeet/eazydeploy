# EazyDeploy — Phase-Wise Product & Engineering Plan

| | |
|---|---|
| **Version** | 0.1 (draft for review) |
| **Date** | 20 Oct 2026 |
| **Repo** | `github.com/amipara-sumeet/eazydeploy` (fork of `vitodeploy/vito`, AGPL-3.0) |
| **Base** | Vito `4.1.1` line @ `8598e4dc`, default branch `4.x` |
| **Companion doc** | `docs/product/VITO_FORK_ANALYSIS.md` (deep code/issue analysis) |
| **Status** | Awaiting sign-off before implementation starts |

---

## 1. Vision & positioning

**One-liner:** EazyDeploy is a self-hosted, lightweight server deployment panel — provider-agnostic, Ubuntu-first — that ships Laravel/PHP, Node.js, and React/static apps to production with zero-downtime, health-gated deploys and zero manual server setup.

**What it is**

- A self-hosted control panel (single small VPS or Docker) that provisions and manages Linux servers over SSH.
- A deployment engine that is safe by default: atomic releases, health checks, automatic rollback, validated config, no silent failures.
- A runtime-agnostic platform: PHP/Laravel, Node SSR (Next/Nuxt/Nest/Express), static React/Vite, Bun later.
- Provider-agnostic: any Ubuntu VPS via SSH, plus first-class API integrations for common clouds (Hostinger, DigitalOcean, Hetzner, Vultr, Linode, AWS).

**What it is not (v1 non-goals)**

- Not Kubernetes, not an autoscaler, not a Docker app host, not a CI runner, not a full PaaS.
- Not a hosting company: no managed databases, no object storage, no email server.
- Not multi-region HA. Single control plane, honest about it.
- Not a Forge/Ploi clone at feature parity on day one. We pick the 20% that matters and do it better.

**Target users**

1. Solo developers and small agencies on $5–20/mo VPSs.
2. Teams that already manage servers and want a self-hosted panel with no per-site pricing.
3. Cost-sensitive users on providers like Hostinger/Contabo who are currently stuck doing everything by hand.

**Positioning vs incumbents**

| | Laravel Forge | Ploi.io | Vito 4.x | **EazyDeploy** |
|---|---|---|---|---|
| Model | SaaS | SaaS | Self-hosted OSS | **Self-hosted OSS** |
| Pricing | Per server | Per server | Free | **Free / self-hosted** |
| Lightweight control plane | n/a (SaaS) | n/a (SaaS) | Heavy (Redis+Horizon+WS) | **SQLite + DB queue, 1 GB target** |
| Zero-downtime default | ✅ new sites | ✅ | ⚠️ beta, Laravel only | **✅ all runtimes** |
| Auto-rollback on bad release | ❌ (notify only) | ❌ | ❌ | **✅ health-gated** |
| Config validation before reload | unknown | unknown | ❌ | **✅ `nginx -t` gate** |
| Node / React / Bun | Partial (Node, Next/Nuxt) | ✅ Node (PM2/Supervisor) | ⚠️ Node deprecated, Bun added | **✅ first-class** |
| Provisioning resumable | n/a | n/a | ❌ | **✅ step state machine** |
| Provider breadth | 8+ | 8+ incl. Hostinger | 8 | **8+ incl. Hostinger** |
| Self-hosted | ❌ | ❌ | ✅ | **✅** |
| Audit log / granular permissions | Partial | ✅ | ⚠️ | **✅** (Phase 7) |

---

## 2. Product principles

1. **Reliability over features.** A deploy that cannot be verified is a failed deploy. Never mark a release active until it passes checks.
2. **Zero-surprise automation.** Never overwrite host config that EazyDeploy does not own. Never destroy user data to make a step pass.
3. **Idempotent by construction.** Every script can run twice with the same result. Every provisioning step can resume.
4. **Measurable quality.** Each phase has numeric exit criteria; no phase ships on vibes.
5. **Narrow and excellent.** Support Ubuntu 22.04/24.04 + nginx + the three runtime families first. Fail fast and clearly elsewhere.
6. **Open source, upstream-friendly.** Keep diffs small and rebaseable; upstream generic fixes where possible.
7. **Self-hosted means the user owns everything.** No phone-home, no license server, no hidden SaaS dependency.

---

## 3. Supported matrix (v1 target)

| Layer | v1 (beta) | Later |
|---|---|---|
| OS | Ubuntu 22.04, 24.04 | Debian 12/13, Ubuntu 26.04, Alma/Rocky |
| Arch | x86_64, arm64 | — |
| Access | Root SSH at provision, dedicated `eazy` user after; key-only recommended | Password-auth onboarding |
| Providers | Custom (any VPS), Hostinger, DigitalOcean, Hetzner, Vultr, Linode, AWS/Lightsail | Proxmox, Scaleway, UpCloud, Oracle, Contabo API |
| Web server | nginx | Caddy (keep beta), nginx+Apache hybrid |
| PHP | 8.1–8.5 (multi-version), FPM pools per site | 7.4, 8.0 legacy |
| Databases | MySQL 8.4, MariaDB 11.8, PostgreSQL 17/18, SQLite | MySQL 9.x, MariaDB 12.x |
| Cache/Queue | Redis, Valkey | Memcached, Meilisearch |
| Runtimes | Laravel, generic PHP, WordPress, **Node SSR (Next/Nuxt/Nest/Express)**, **static React/Vite/SPA**, Bun | Python, Go, Docker apps |
| Process manager | systemd (primary), Supervisor (PHP queues) | PM2, podman |
| Git | GitHub, GitLab, Bitbucket, custom SSH | Gitea/Forgejo first-class |
| Notifications | Email, Slack, Discord, Telegram, generic webhook | SMS, PagerDuty |

**Deliberate scope decisions**

- **systemd, not PM2, for Node by default.** No extra runtime dependency, cgroup limits, journald logs, `Restart=always`, and it survives reboots. PM2 remains an optional install for cluster mode (later).
- **Static React is a first-class site type.** Build in release, symlink `current`, nginx serves files — zero downtime with no process management.
- **Node SSR uses blue/green ports.** Build in a release, start the new process on the idle port, health check, switch nginx upstream, drain and stop the old process. This is the only honest way to do zero-downtime Node without a cluster manager.
- **Laravel is a first-class citizen, not the only citizen.** Forge-specific integrations (Horizon, Octane, Pulse, Reverb) are recipes/blueprints, not hard requirements.

---

## 4. Architecture

### 4.1 Control plane (slim profile — default)

```
Browser (React/Inertia)
        │
Laravel 13 app (actions, policies, API)
        │
Database queue driver  ──► queue:work (2 workers) + schedule:work
        │
phpseclib SSH ──► managed Ubuntu servers
```

- **Data:** SQLite (default, WAL) or Postgres/MySQL for larger installs.
- **Queue:** `database` driver by default; Redis + Horizon become an opt-in "performance" profile.
- **Scheduler:** `schedule:work` in the worker container; no host cron required in Docker mode.
- **Realtime:** polling by default; websocket server optional, never required for correctness.
- **Footprint target:** < 400 MB RAM steady state, works on a 1 GB VPS alongside a small app.
- **Deployment:** Docker Compose (`app`, `worker`, `scheduler` — one image, three commands) or the non-destructive installer. No host nginx/PHP/Redis writes in Docker mode.

### 4.2 Managed server layout

```
/home/<site-user>/<domain>/
├── current -> releases/<ts>        # atomic switch point
├── releases/<ts>/                  # per-deploy build
├── source/                         # optional canonical clone (classic mode)
├── shared/                         # .env, storage, sqlite, uploads
└── logs/
```

- One Linux user per site (optional isolation, default on for new sites).
- nginx vhost per site; PHP-FPM pool per site for PHP; systemd unit per Node app (`eazy-<site>@blue/ @green`).
- Sudoers **allowlist** in `/etc/sudoers.d/eazydeploy` (reload nginx, reload php-fpm, supervisorctl, systemctl for own unit) — never `NOPASSWD:ALL`.
- No writes outside owned paths; every generated file is tracked so `uninstall` and drift repair are possible.

### 4.3 Core new state machines

| Table | Purpose |
|---|---|
| `provisioning_steps` | Per-server, per-step state: `pending/running/succeeded/failed/skipped`, attempts, log, error, timestamps. Enables resume/retry/repair |
| `deployment_phases` | Per-deployment phases with timings and logs; powers the timeline UI and failure diagnosis |
| `health_checks` | Site probes: URL, status, latency, region, checked_at; powers uptime, SSL, alerts, rollback decisions |
| `alert_rules` / `alert_events` | Thresholds, channels, dedup, recovery notices |
| `audit_logs` | Actor, action, target, IP, result, link to SSH log |

### 4.4 Deploy engine (spec)

Phases, each with timeout, log, and retry policy:

```
lock → prepare → fetch → build → link → migrate? → switch → reload → verify → activate → cleanup
                                     │
                             health gate fails ──► rollback (switch back, reload, notify)
```

- **lock:** single explicit lock per site (DB row + on-server lock file); duplicate triggers are queued or rejected with a clear message.
- **fetch:** `git fetch --prune && git reset --hard origin/$BRANCH && git clean -fd` — never `git pull` on a dirty tree.
- **build:** raw build script from the site (defaults per runtime); per-phase timeout; output streamed to the deploy log.
- **link:** shared paths symlinked (`.env`, `storage`, SQLite, uploads).
- **switch:** `ln -sfn` to the new release + reload PHP-FPM (PHP) or upstream switch (Node) — validated before it happens.
- **verify:** HTTP probe with correct `Host`, 3 attempts / 2 s apart, 2xx/3xx/401/403 = healthy, 5xx/timeout = unhealthy. Optional per-site health path.
- **activate:** only after verify passes. Marks deployment active, restarts workers, prunes releases beyond retention.
- **rollback:** symlink back, reload FPM/upstream, mark failed, alert. Never activate the failed release.
- **nginx safety:** write to temp → `nginx -t` → atomic move + reload; on failure restore previous file and abort with a diff.

### 4.5 Runtime adapters

| Runtime | Build | Activate | Process | Health |
|---|---|---|---|---|
| Laravel / PHP | composer, npm, artisan | symlink + FPM reload | FPM pool + Supervisor queues | HTTP probe |
| WordPress / PHP | composer/npm optional | symlink + FPM reload | FPM pool | HTTP probe |
| Node SSR | npm/pnpm/yarn/bun install + build | start new on idle port, switch upstream, drain old | systemd template unit | HTTP probe on new port + public URL |
| Static React/Vite | npm install + build | symlink | none (nginx serves `current/dist`) | HTTP probe |
| Bun | bun install + build | same as Node | systemd | HTTP probe |

---

## 5. Phase plan

> Estimates assume **one senior full-stack developer** working full-time. "Weeks" are sequential; several phases can overlap with a second pair of hands. Total to public beta: **~18 weeks (1 dev) / ~10–12 weeks (2 devs)**.

### Phase 0 — Foundation & fork hygiene (Week 1)

**Objective:** make the fork safe to build on: reproducible, tested, and syncable with upstream.

**Scope**

- Pin the base commit; add `UPSTREAM.md` with rebase procedure and sync cadence (monthly, or on security releases).
- Branch strategy: `main` = released state, `develop` = integration, `feat/*`, `fix/*`; tags mirror releases.
- CI: Pint, PHPStan, Pest, ESLint/typecheck, frontend build; add `bash -n` + ShellCheck over rendered SSH scripts; fail on new architecture-rule violations.
- Reproducible dev/e2e environment (docker compose with app + worker + sqlite; separate e2e harness in Phase 1).
- `eazydeploy:doctor` command: PHP extensions, APP_KEY, DB writability, queue worker heartbeat, scheduler heartbeat, storage permissions, disk space, optional Redis/ws state. Human + JSON output.
- Control-plane backup/restore script (DB dump + storage) and an upgrade script that dumps before migrating.
- Security quick wins: TLS 1.2+ in generated configs; idempotent `create-user`; stop overwriting `/root/.ssh/authorized_keys`; targeted sudoers (with legacy `NOPASSWD` behind an explicit flag); installer no longer rewrites host `nginx.conf` by default.
- Decision: keep upstream "Vito" strings internally until the rebrand phase (keeps rebases trivial). Repo name already `eazydeploy`.

**Deliverables:** green CI, doctor command, backup/restore, hardened installer defaults, `UPSTREAM.md`.

**Exit criteria:** fresh clone → `composer install && php artisan test` green; CI green on PR; doctor correctly detects at least 6 seeded failure conditions; upstream rebase test performed once.

**Estimate:** 5 working days. **Depends on:** nothing.

---

### Phase 1 — Deploy Safety Engine (Weeks 2–4)

**Objective:** make deployments trustworthy. This is the phase that solves the original pain.

**Scope**

1. **Phase state machine** (`deployment_phases`): wrap the existing deploy pipeline in explicit phases with timeouts, logs, and cancel support. Timeline UI on the site page.
2. **Deploy lock:** DB-backed per-site lock + on-server lock file; reject/queue duplicate triggers (webhook + manual + API) with a clear message.
3. **Git hygiene:** replace `git stash/pull` with `fetch + reset --hard + clean -fd` in modern deploys; keep untracked ignored files (no `-x`) to protect shared data.
4. **PHP-FPM reload / OPcache reset** after switch and after rollback (configurable per site; default on). Graceful `systemctl reload phpX.Y-fpm`.
5. **nginx validation gate:** temp write → `nginx -t` → atomic move + reload; restore previous config on failure. Apply to create-vhost, update-vhost, and domain changes.
6. **Health gate + auto-rollback:** probe after switch; on failure revert symlink, reload, mark failed, notify. Never activate a failed release; fix the current `failed()` behaviour that re-activates failed deployments.
7. **Migration safety:** optional pre-migration DB snapshot; migration lock; explicit warning that code rollback does not roll back data; optional snapshot restore action.
8. **Release retention + disk guard:** enforce history with pre-flight disk check; block deploys below a configurable free-space threshold.
9. **Verification of post-deploy state:** workers restarted, scheduler present, `storage:link`, config/route cache, SSL still valid.
10. **Node/static groundwork:** runtime-neutral phase definitions so Phase 5 plugs in without touching the engine.

**Deliverables**

- New tables/models/actions: `deployment_phases`, deploy lock, health probe action, rollback action.
- New/updated scripts: `reload-php-fpm`, vhost test-and-apply, snapshot/restore, probe.
- Updated `DeployJob`, `RollbackJob`, `Nginx` service, modern-deployment scripts.
- Deploy timeline UI + failure reason + logs.
- Tests: feature tests with `SSH::fake()` for each phase and failure path; a real E2E harness (Ubuntu 24 VM/container with sshd + nginx + php-fpm) that deploys a fixture Laravel app, publishes v2, breaks v3, and asserts zero 5xx and successful auto-rollback.

**Exit criteria**

- 100 consecutive deploys on the E2E harness with no OPcache/FPM degradation; p95 < 60 s for a small app.
- Injected 5xx release → automatic rollback < 30 s, zero user-visible failed requests.
- Invalid vhost → rejected, other sites unaffected.
- Duplicate deploy triggers → exactly one execution.
- Failed deploy never shows as active.

**Estimate:** 15 working days. **Depends on:** Phase 0.

---

### Phase 2 — Provisioning Reliability & Preflight Doctor (Weeks 5–6)

**Objective:** make server onboarding boring and resumable, on any Ubuntu VPS.

**Scope**

1. **`provisioning_steps` state machine:** refactor `InstallServer::install()` into idempotent steps (preflight, system update, dependencies, per-service install, security baseline, finalize). Each step: attempts, backoff, per-step log, resumable.
2. **Preflight checks:** distro/version/arch, RAM/disk, existing web/db services, ports 80/443 (and configured SSH port), IPv6, DNS resolution, cloud firewall hints, SSH auth mode, sudo availability, time sync, provider metadata. Block with a human-readable fix list before touching the server.
3. **Apt/npm robustness:** lock waiting (`DPkg::Lock::Timeout`), retries with jitter, noninteractive everywhere, `--force-confold` preserved; detect and report held/broken packages.
4. **Idempotent user & sudoers:** `useradd -m` when missing, append-only authorized_keys, scoped sudoers allowlist, no MD5 password hashes, optional password auth, rollback of partially created users.
5. **Resume / Repair / Drift:** UI actions to resume from the failed step, re-run a single step, or reconcile the whole server (services, vhosts, pools, firewall) against desired state. `servers:reconcile` command for the scheduler.
6. **Actionable failures:** every step failure maps to a message + suggested fix, with the log excerpt attached. Notifications on failure.
7. **Service install hardening:** version-availability checks per OS, PPA/repo setup validated, Postgres/MySQL failure modes handled (ports, data dir, auth), uninstall command per service.

**Deliverables:** provisioning state machine, preflight Action + UI, resume/repair actions, reconciler command, failure catalogue (initial 15–20 known failure modes).

**Exit criteria:** provision a fresh Ubuntu 24 VPS end-to-end; kill the control plane mid-provision → resume produces no duplicates; run every step twice → no errors/changes; preflight blocks each of the seeded bad hosts with actionable messages; first-try success ≥ 98% across the E2E matrix (Ubuntu 22/24 × amd64).

**Estimate:** 10 working days. **Depends on:** Phase 0 (independent of Phase 1, can run in parallel with a second dev).

---

### Phase 3 — Lightweight Runtime & Non-destructive Installer (Weeks 7–8)

**Objective:** the control plane must run comfortably on a 1 GB VPS and never fight the host.

**Scope**

1. **Queue profiles:** add `database` connections for `default`/`ssh`; make Redis optional. Default profile uses `queue:work` + `schedule:work`. Horizon becomes an opt-in profile for larger installs.
2. **Realtime optional:** polling fallback for live updates/deploy logs; websocket process only enabled when desired.
3. **Docker delivery:** single image, three services (`app`, `worker`, `scheduler`), healthchecks, resource hints, no hardcoded credentials, persistent volumes, clean upgrade path (backup → migrate → rollback). Publish versioned images.
4. **Non-destructive installer:** only writes its own compose/env/data dir. Never touches host nginx/PHP/Redis. Legacy installer kept behind an explicit flag and clearly labelled as dedicated-host-only.
5. **Performance tuning:** OPcache/FPM worker sizing based on RAM, SQLite WAL + busy timeout, Horizon off, log rotation, `queue:work` limits and restart policy.
6. **Self-monitoring:** control-plane health endpoint, worker/scheduler heartbeat checks, alert when queue is stalled, storage/disk warnings.

**Deliverables:** slim compose, installer script, queue config profiles, health endpoint, tuning defaults.

**Exit criteria:** on a 1 vCPU / 1 GB Ubuntu 24 VM: idle RAM < 400 MB, a deployment of a small app completes with peak RAM < 700 MB, control plane survives a worker kill (auto-restart, no lost jobs), fresh install < 3 minutes.

**Estimate:** 10 working days. **Depends on:** Phase 0; benefits from Phase 1/2 being done.

---

### Phase 4 — Onboarding UX & Blueprints (Weeks 9–10)

**Objective:** from "I have a repo" to "it's live and safe" in one screen, with no SSH.

**Scope**

1. **One-screen deploy wizard:** provider/server (new or existing) → repo + branch → domain → runtime → database → SSL. Everything else uses defaults derived from repo detection.
2. **Repo detection:** Laravel (composer.json + artisan), Symfony, WordPress, generic PHP, Next/Nuxt/Nest/Express, Vite/CRA static, Bun. Detect package manager (lockfile), build command, output dir, start command, required PHP/Node version.
3. **Blueprints:** one-click stacks — "Laravel + MySQL + Redis + Horizon + scheduler", "Laravel API", "WordPress + cache", "Next.js SSR", "React SPA", "Node worker". Blueprint = sites + services + workers + cron + env template + deploy script.
4. **Env editor safety:** raw editor by default for existing files; validated structured mode; no quote corruption; diff preview; rollback to previous env.
5. **Import existing app:** point at a server/path, detect runtime, adopt it (vhost, releases or in-place, worker, cron, SSL) without redeploying or breaking it.
6. **Provider breadth UI:** unified provider connection flow (tokens, SSH keys, regions) with validation; Hostinger included as the first new integration (VPS + DNS), provider-agnostic beneath.
7. **Zero-downtime by default** for new sites; the wizard explains what it will change before it changes anything (summary screen).

**Deliverables:** wizard, detection service, blueprints, env editor, import flow, Hostinger provider + DNS.

**Exit criteria:** 5 different real repositories (Laravel, WordPress, Next SSR, Vite React, Express API) deployed from scratch with zero SSH in < 10 minutes each; import of an existing Laravel app with no downtime and no config loss.

**Estimate:** 10 working days. **Depends on:** Phases 1–3.

---

### Phase 5 — Runtime Breadth: Node, React, Bun (Weeks 11–13)

**Objective:** first-class non-PHP runtimes with the same safety guarantees.

**Scope**

1. **Node.js service:** install/manage Node versions (mise or nvm-style), per-site version pinning, no global version conflicts.
2. **Node SSR site type:** build/start/stop, port allocation, systemd template units (`eazy-<site>@blue|green`), health path, env, logs in the UI, blue/green activation via nginx upstream switch.
3. **Static React site type:** Vite/CRA/Next static export, output dir, SPA fallback (`try_files`), cache headers, optional prerender hook.
4. **Bun site type:** install + build + run, same engine.
5. **Monorepo support:** app root relative to repo root, workspaces-aware install/build, per-site package manager.
6. **Process management UI:** status, restart, stop, logs, resource usage; PM2 optional install for cluster mode (documented, not default).
7. **Node deploy scripts + shared paths** (`.env`, uploads, sqlite), worker daemons (queue workers for Node queues), and cron.
8. **React-specific docs and templates:** API URL handling, `VITE_*` env injection at build time, SSR hydration pitfalls, cache busting.

**Deliverables:** Node service + site type, static React site type, Bun, monorepo support, process UI, docs.

**Exit criteria:** Next.js SSR deploy with zero downtime (traffic loop shows no 5xx and no connection resets); React SPA deploy is atomic; Node version switch per site without affecting siblings; memory limits enforced via systemd; crash → auto-restart + alert.

**Estimate:** 15 working days. **Depends on:** Phase 1 (engine), Phase 3 (runtime installs), Phase 4 (UX).

---

### Phase 6 — Observability, Alerts & Status (Weeks 14–15)

**Objective:** know when something breaks before users do.

**Scope**

1. **Uptime + latency probes:** control-plane HTTP checks per site (path, interval, expected status), history and p95; probe from the server itself for internal checks.
2. **SSL expiry monitor:** already partly present; add per-domain expiry, renewal verification, and renewal-failure alerts.
3. **Resource monitors:** CPU, RAM, disk, inode, load; thresholds per server; alert on breach and recovery.
4. **Alert rules & routing:** email, Slack, Discord, Telegram, generic webhook; per-rule channels; dedup, flapping protection, recovery notifications. Scheduled digest option.
5. **Deploy notifications:** success/failure/duration/commit, with links; CI webhook payloads (Forge-compatible shape plus our own).
6. **Site status page (optional, per project):** public page generated from probe history; custom domain; incident notes.
7. **Log access:** tail nginx/PHP/Node logs and deploy logs from the UI; search within a log; no SSH needed.

**Deliverables:** probe engine + scheduler jobs, alert rules UI, channels, status pages, log viewer upgrades.

**Exit criteria:** induced site outage alerts within 2 minutes and resolves on recovery; alert dedup verified under flapping; SSL < 14 days warns; disk threshold warns; no duplicate notifications.

**Estimate:** 10 working days. **Depends on:** Phase 1 (health probes), Phase 4 (UI patterns).

---

### Phase 7 — Security, Permissions & Audit (Weeks 16–17)

**Objective:** safe to expose to teams and to the internet.

**Scope**

1. **Scoped sudoers:** validate the allowlist works for every supported operation; remove any remaining `NOPASSWD:ALL` default; document the residual risk.
2. **SSH hardening (opt-in, verified):** disable password auth and root login only after a second verified session; automatic revert if the verification session fails.
3. **Audit log:** actor, action, target, IP, timestamp, result, link to SSH output; immutable-ish (append-only table), exportable; UI filters.
4. **Permissions:** project/server-scoped roles (owner, admin, deployer, viewer), enforcement in policies + API keys; deny-by-default tests.
5. **Secrets hygiene:** redaction of known secret patterns in logs and notifications; encrypted-at-rest verified; APP_KEY rotation procedure.
6. **Supply chain:** signed release artifacts/checksums for the control plane; plugin hash pinning + review flow; dependency audit in CI; CVE response runbook.
7. **Hardening guidance + defaults:** rate limits on deploy hooks and login, 2FA enforcement option, session policies, security headers in generated vhosts.
8. **Threat model document** and a pen-test pass against the control plane API.

**Deliverables:** audit log, scoped permissions, hardening actions, security docs, threat model, CI security checks.

**Exit criteria:** cross-project and cross-server access tests all deny; every privileged action appears in the audit log with output link; hardening flow cannot lock the operator out (simulated failure reverts); dependency/secret scans clean or documented.

**Estimate:** 10 working days. **Depends on:** Phases 2–4.

---

### Phase 8 — Backups, DR & Ops Polish (Weeks 18–19)

**Objective:** recoverable, upgradeable, documented — the difference between a tool and a product.

**Scope**

1. **Database backups:** schedules, providers (S3-compatible, Backblaze B2, Google Drive, Dropbox), compression + encryption, verification (restore into a scratch DB), retention, one-click restore, failure alerts.
2. **File backups:** site files + shared paths, excludes, retention, restore.
3. **Pre-deploy snapshots:** integrate with Phase 1 migration safety.
4. **Control-plane DR:** automated self-backup (DB + storage), documented restore, versioned upgrade with rollback, upgrade notices.
5. **Migration from Vito:** import script/guide for projects, servers, sites (best effort, documented limits).
6. **Docs site:** install, quickstart per runtime, troubleshooting, architecture, API, upgrade, security.
7. **Release process:** versioning, changelog, signed tags, beta/stable channels, upgrade telemetry (opt-in only).
8. **Performance pass:** N+1 audit, query budgets, index review, frontend bundle budget.

**Deliverables:** backup engine + restore UI, DR runbooks, Vito migration guide, documentation, release pipeline.

**Exit criteria:** restore a site DB and files into a clean environment within 15 minutes using only the UI + docs; control-plane upgrade N-1 → N with automatic backup and one-command rollback; documented disaster runbook rehearsed once.

**Estimate:** 10 working days. **Depends on:** Phases 0–7.

---

### Phase 9 — Team & Commercial Readiness (optional, post-beta)

Only if the private deployment proves useful and you decide to open it up.

- Billing/subscription plumbing (Stripe/Paddle) or donation/support model.
- Multi-tenant organizations, SSO (OIDC/SAML), SCIM.
- Whitelabel, status pages as a product, custom domains.
- Public marketplace for plugins/blueprints, review process.
- MCP server for AI assistants (Ploi parity), mobile-friendly PWA, CLI + SDK.
- Hosted control-plane offering (AGPL §13 compliance: source offer to network users).

**Decision gate:** after 4–6 weeks of self-hosted internal use with ≥ 95% deploy success and no data-loss incidents.

---

## 6. Cross-cutting workstreams

| Workstream | Cadence | Notes |
|---|---|---|
| **Testing** | Every phase | Unit + feature (Pest, `SSH::fake`), shell contract (`bash -n`, ShellCheck, BATS), E2E VM harness, nightly matrix on Ubuntu 22/24 |
| **Upstream sync** | Monthly + security releases | Rebase `develop` on `upstream/4.x`; generic fixes upstreamed; conflicts resolved in small batches |
| **Documentation** | Every phase | Docs are part of the definition of done, not a phase |
| **Security** | Continuous | Dependency audit in CI, secret scan, threat model updates, pen-test before beta |
| **Quality gates** | Every PR | Pint, PHPStan, Pest, ESLint, typecheck, architecture tests, shell checks |
| **Accessibility & UX** | Every phase | Keyboard paths, empty states, error messages with fixes, destructive-action confirmations |
| **Telemetry** | Opt-in only | Never phone home by default; crash/deploy stats only with explicit consent |

---

## 7. KPIs & definition of "production grade"

| Metric | Target |
|---|---|
| Deploy success rate (non-code failures excluded) | ≥ 99% |
| p95 deploy duration, small app | < 60 s |
| Zero-downtime violations during deploys | 0 |
| Auto-rollback time on bad release | < 30 s |
| First-try provisioning success (supported matrix) | ≥ 98% |
| Control-plane idle RAM (1 GB VPS profile) | < 400 MB |
| Fresh control-plane install (Docker) | < 3 min |
| Site restart/recovery after crash | < 10 s, automatic |
| Alert latency (site down) | < 2 min |
| Backup restore to clean environment | < 15 min |
| Audit coverage of privileged actions | 100% |

---

## 8. Risk register

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Upstream drift makes rebases painful | High | Medium | Small modular diffs, upstream-first for generic fixes, monthly sync, avoid mass rebranding until late |
| Security incident (root SSH management) | Medium | Critical | Threat model, scoped sudo, audit log, signed releases, pen-test, fast CVE process |
| Scope creep (trying to match Forge+Ploi fully) | High | High | Enforce non-goals, phase exit criteria, say no to feature requests outside the three pillars |
| Node zero-downtime complexity (blue/green, draining) | Medium | High | Prototype early in Phase 1 groundwork, E2E traffic-loop tests, systemd-first |
| Provider API changes (Hostinger etc.) | Medium | Medium | Thin provider adapters, contract tests against recorded fixtures, graceful degradation to custom SSH |
| E2E VM harness cost/flakiness | Medium | Medium | Containerized systemd harness + nightly cloud VM smoke; deterministic fixtures |
| Solo-maintainer bandwidth | High | High | Phased plan, plugins for integrations, community contributions after beta |
| AGPL constraints if commercialized | Medium | High | Decide open-core before Phase 9; source-offer compliance built into hosted plans |
| WordPress/PHP legacy edge cases | Medium | Low | Support matrix, clear "unsupported" messaging, plugin escape hatch |

---

## 9. Decisions log

| # | Decision | Status |
|---|---|---|
| 1 | Fork Vito 4.x and harden rather than greenfield | ✅ decided |
| 2 | Repo renamed to `eazydeploy`; final brand TBD | ✅ decided |
| 3 | Provider-agnostic, Ubuntu-first; Hostinger is one provider among many | ✅ decided |
| 4 | First-class support for Laravel/PHP, Node SSR, static React, Bun | ✅ decided |
| 5 | systemd as default Node process manager; PM2 optional later | ✅ decided |
| 6 | Slim profile: SQLite + database queue, Redis/Horizon optional | ✅ decided |
| 7 | Deliberately reload PHP-FPM after release switch (differs from Forge docs) due to OPcache accumulation | ✅ decided |
| 8 | Auto-rollback on failed health check (Forge only notifies) | ✅ decided |
| 9 | Keep upstream "Vito" strings until rebrand phase to keep rebases cheap | ✅ decided |
| 10 | Internal/private use first; commercial decision after 4–6 weeks of dogfooding | ✅ decided |
| 11 | API/CLI in v1 scope or Phase 6+? | ⏳ open |
| 12 | Database engine for larger installs: Postgres-first vs MySQL-first | ⏳ open |
| 13 | Caddy as first-class alternative to nginx in v1 or later? | ⏳ open |
| 14 | Whether to keep Vito's plugin system as-is or slim it | ⏳ open |

---

## 10. Immediate next 10 working days

1. Tag and pin the base (`4.1.1`); create `develop`; add `UPSTREAM.md` + sync script.
2. Get CI green on the fork; add shell-script linting and the architecture tests.
3. Introduce `deployment_phases` migration + model + timeline scaffolding (no behaviour change).
4. Replace `git pull` with `fetch/reset --hard` in modern deployment; add FPM reload after release and rollback.
5. Add the `nginx -t` gate for vhost writes and reloads.
6. Add the health probe action and wire auto-rollback; fix failed-deploy activation.
7. Stand up the E2E harness (Ubuntu 24 + sshd + nginx + php-fpm) and the traffic-loop test.
8. Implement `eazydeploy:doctor` and control-plane backup/restore.
9. Harden `create-user` (idempotent, scoped sudoers, no root key clobbering).
10. Demo: two real deploys on two different providers (one Laravel, one static React) with a deliberately broken third release rolling back automatically.

---

## 11. Open questions for review

1. **Node process manager:** confirm systemd-first (recommended) vs PM2-first.
2. **React scope:** static SPA + Next SSR in v1, or also Remix/React Router v7 SSR?
3. **Database default for the control plane:** SQLite (recommended) vs Postgres for all installs.
4. **API/CLI timing:** build the deploy API + CLI in Phase 1–2 (enables CI immediately) or later?
5. **Brand:** stay `eazydeploy` or pick a distinct final name before public release.
6. **Beta audience:** private only, or open to a small group of testers after Phase 6?
