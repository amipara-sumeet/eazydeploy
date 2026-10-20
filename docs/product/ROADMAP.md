# EazyDeploy — Phase-Wise Product & Engineering Plan

| | |
|---|---|
| **Version** | 0.3 (final decisions locked; awaiting go-ahead) |
| **Date** | 20 Oct 2026 |
| **Repo** | `github.com/amipara-sumeet/eazydeploy` (fork of `vitodeploy/vito`, AGPL-3.0) |
| **Base** | Vito `4.1.1` line @ `8598e4dc`, default branch `4.x` |
| **Companion doc** | `docs/product/VITO_FORK_ANALYSIS.md` (deep code/issue analysis) |
| **Status** | Internal-first build. No public release until owner sign-off. |
| **Roles** | Implementation: AI agent · Review & testing: owner |

---

## 1. Vision & positioning

**One-liner:** EazyDeploy is a self-hosted, lightweight server deployment panel — provider-agnostic, Ubuntu 26-first — that ships Laravel/PHP, Node.js, and React/static apps to production with zero-downtime, health-gated deploys and zero manual server setup.

**What it is**

- A self-hosted control panel (single VPS or Docker) that provisions and manages Linux servers over SSH.
- A deployment engine that is safe by default: atomic releases, health checks, automatic rollback, validated web-server config, no silent failures.
- Runtime-agnostic: Laravel/PHP, Node SSR (Next/Nuxt/Nest/Express), static React/Vite, Bun.
- Provider-agnostic: any Ubuntu VPS via SSH, plus first-class API integrations (Hostinger, DigitalOcean, Hetzner, Vultr, Linode, AWS).
- Web-server-agnostic: **nginx and Caddy are both first-class in v1**.
- API/CLI-first: every deploy action is automatable from day one, so CI works immediately.

**What it is not (v1 non-goals)**

- Not Kubernetes, not an autoscaler, not a Docker app host, not a CI runner, not a full PaaS.
- Not a hosting company: no managed databases, object storage, or email server.
- Not multi-region HA. Single control plane, honest about it.
- Not a public product yet. Internal use, private testing, no marketing release until the owner is satisfied.

**Positioning vs incumbents**

| | Laravel Forge | Ploi.io | Vito 4.x | **EazyDeploy** |
|---|---|---|---|---|
| Model | SaaS | SaaS | Self-hosted OSS | **Self-hosted, internal-first** |
| Pricing | Per server | Per server | Free | **Free / self-hosted** |
| Control plane | n/a | n/a | Redis + Horizon + WS | **Postgres + DB queue, no Redis/Horizon required** |
| Zero-downtime default | ✅ new sites | ✅ | ⚠️ beta, Laravel only | **✅ all runtimes** |
| Auto-rollback on bad release | ❌ notify only | ❌ | ❌ | **✅ health-gated** |
| Config validation before reload | unknown | unknown | ❌ | **✅ `nginx -t` / `caddy validate` gate** |
| Web servers | nginx | nginx (+ Apache hybrid) | nginx, Caddy beta | **nginx + Caddy first-class** |
| Node / React / Bun | Partial | ✅ PM2/Supervisor | ⚠️ Node deprecated, Bun added | **✅ PM2-first Node, static React, Bun** |
| Laravel extras | Horizon/Octane/Pulse/Reverb | partial | plugins | **✅ first-class feature pack** |
| Provisioning resumable | n/a | n/a | ❌ | **✅ step state machine** |
| API + CLI | ✅ | ✅ | API, no CLI | **✅ v1** |
| Self-hosted | ❌ | ❌ | ✅ | **✅** |

**What we deliberately do differently**

1. **Auto-rollback, not just alerting.** Forge/Ploi notify after a bad deploy; we revert it.
2. **Resumable provisioning.** A failed apt step never means "delete the server and start over".
3. **One control plane for PHP + Node + React.** No separate tools or per-runtime panels.
4. **Lightweight by default.** No Redis, Horizon, or websocket process required; Postgres is the only heavy dependency.
5. **Caddy parity.** Automatic HTTPS without certbot is a real operational win; it is not a beta afterthought.
6. **UI that is ours.** New design system and information architecture, not a reskinned Vito.

---

## 2. Key decisions & rationale

### 2.1 Licensing: what is actually possible

**Facts**

- Vito is **AGPL-3.0**. A fork is a derivative work and **must remain AGPL-3.0**. You cannot unilaterally relicense it to MIT/Apache/BSL.
- Relicensing would require permission from **all copyright holders** (the maintainer plus every contributor whose code remains), which is impractical without a CLA or a negotiated agreement.
- AGPL does **not** prevent commercial use or charging money. It requires that users interacting with the modified program over a network are offered its source. For **internal/private use**, the practical impact is minimal — your own team can access the source.
- If you later sell a hosted service, AGPL §13 applies: you must offer the corresponding source to your users. Open-core AGPL is a proven model (Grafana, Mattermost, GitLab CE).

**Options**

| Option | Feasibility | Cost | When |
|---|---|---|---|
| **Keep AGPL** (recommended now) | ✅ immediate | none | Now |
| Negotiate a dual/commercial license with the Vito maintainer | ⚠️ unlikely; contributor consent needed | legal + negotiation | Only if going commercial |
| Clean-room rewrite (new code, no copied Vito code) under Apache-2.0/MIT | ✅ legally clean | 6–12 months | Only if proprietary is mandatory |
| Open-core: AGPL core + separately licensed services communicating over APIs | ⚠️ needs legal review of AGPL boundaries | medium | Commercial phase |

**Decision:** keep AGPL-3.0 for the fork, preserve `LICENSE`, `NOTICE`, and upstream attribution, and do not strip headers. Revisit licensing only if and when a commercial release is planned. **Do not plan to "change the license" of the fork — that is not legally available.** This is not legal advice; engage a lawyer before any commercial distribution.

### 2.2 Plugin system: keep the primitives, drop the marketplace

**Recommendation: keep the internal registration primitives; remove remote/marketplace plugin installation from v1.**

Why:

1. **Security.** Vito plugins are PHP code executed inside a control plane that holds root SSH credentials for every managed server. A malicious or compromised plugin is a total compromise. A marketplace amplifies this.
2. **API churn.** Vito's plugin contracts changed materially in v4. Maintaining a public plugin API adds compatibility burden before our own core is stable.
3. **Scope.** Our differentiators are deploy safety, provisioning, and UX — not an ecosystem. A plugin marketplace is a product of its own.
4. **We still need extension points.** The `Register*` primitives are the cleanest way to add providers, DNS, site types, services, and notification channels without bloating core.

**Plan:** keep the registration classes as internal extension points; support **local plugins only**, disabled by default, hash-pinned, with an explicit admin enable step. No GitHub-zip installer, no marketplace UI in v1. After beta, if an ecosystem is justified, design a **sandboxed, capability-scoped** plugin model (separate process, explicit permissions) rather than in-process PHP.

### 2.3 Control-plane database: Postgres-only

**Decision: PostgreSQL is the only supported production database for the control plane. SQLite remains test-only (in-memory) for fast unit/feature tests.**

Can SQLite do the job? Yes, for a single user and modest volume, with WAL mode, `busy_timeout`, and a single writer. But:

- Vito's queue uses database-backed jobs; multiple `queue:work` processes plus web requests and metrics writes create write contention that SQLite handles poorly.
- Metrics, deployment history, audit logs, and alert events grow continuously; Postgres indexes and `JSONB` handle this better.
- Teams, API tokens, and concurrent deploys are on the roadmap; Postgres avoids a migration later.
- Postgres costs ~80–150 MB RAM. Tune `shared_buffers` (~128 MB), `work_mem` (~4 MB), and `max_connections` (~50) for a small control plane.

**Consequences:** control-plane minimum 1 GB (tuned, tight), **2 GB recommended**; Docker Compose includes Postgres; `eazydeploy:doctor` checks Postgres health; CI runs the full suite against Postgres and a fast SQLite subset. Phase 0 includes a **Postgres compatibility audit** of Vito's migrations/queries (JSON, date functions, `PRAGMA`, `strftime`, case sensitivity, `ILIKE` vs `LIKE`).

### 2.4 Other locked decisions

- **Ubuntu 26.04 LTS is the default and first-choice OS.** 24.04 remains supported, 22.04 is legacy. Hostinger 26.04 is the primary test image.
- **Forge-specific Laravel integrations are v1 scope:** Horizon, Octane, Pulse, Reverb, scheduler, queues. **Octane defaults to FrankenPHP** with `max_requests` recycling and RSS monitoring to contain memory growth.
- **PM2 is required for Node**, not optional. PM2 runs **single-process by default** (deterministic blue/green), per site user under a systemd unit; cluster mode is opt-in. systemd remains the OS-level supervisor.
- **nginx is the default web server; Caddy is one click away and first-class** in v1 (install, vhost generation, validation, reload, SSL, PHP, Node proxy, redirects, basic auth).
- **API + CLI are v1 scope**, starting in Phase 1 so CI works immediately; CLI ships as a **PHAR**.
- **Blueprints include MySQL, MariaDB, PostgreSQL, and SQLite** options.
- **React scope:** static SPA (Vite/CRA) + Next.js SSR in v1; Remix/React Router v7 SSR in v2.
- **UI:** **calm, Vercel-like** brand and design-system refresh in Phase 6, not a skin. No Vito visual identity in user-facing surfaces.
- **Private beta wave 1: 1–2 trusted developers after Phase 7**, read-only until Phase 8 security and Phase 9 backup gates pass.
- **No public release** until the owner signs off after dogfooding.

---

## 3. Supported matrix (v1 target)

| Layer | v1 | Later |
|---|---|---|
| OS | **Ubuntu 26.04 LTS (default)**, 24.04, 22.04 (legacy) | Debian 12/13, Alma/Rocky |
| Arch | x86_64, arm64 | — |
| Access | Root SSH at provision, dedicated `eazy` user after; key-only recommended | Password-auth onboarding |
| Providers | Custom (any VPS), **Hostinger**, DigitalOcean, Hetzner, Vultr, Linode, AWS/Lightsail | Proxmox, Scaleway, UpCloud, Oracle, Contabo API |
| Web servers | **nginx + Caddy (both first-class)** | Apache hybrid, OpenLiteSpeed |
| PHP | 8.1–8.5 (multi-version), FPM pools per site | 7.4, 8.0 legacy |
| Laravel pack | Horizon, Octane, Pulse, Reverb, scheduler, queues, storage links | Nightwatch, Spark, Nova installers |
| Databases | MySQL 8.4/9.x, MariaDB 11.8/12.x, PostgreSQL 17/18, SQLite | MySQL 9.7, MariaDB 12.3 |
| Cache/Queue | Redis, Valkey | Memcached, Meilisearch |
| Runtimes | Laravel, generic PHP, WordPress, **Node SSR (Next/Nuxt/Nest/Express)**, **static React/Vite/SPA**, Bun | Remix/React Router v7 SSR, Python, Go |
| Node process manager | **PM2 (required)**, supervised by systemd; per-site PM2 home | PM2 cluster presets, podman |
| PHP process manager | PHP-FPM + Supervisor for queues | — |
| Git | GitHub, GitLab, Bitbucket, custom SSH | Gitea/Forgejo first-class |
| Notifications | Email, Slack, Discord, Telegram, generic webhook | SMS, PagerDuty, status pages |
| Control-plane DB | **PostgreSQL only** | — |

**Deliberate scope decisions**

- **PM2 per site user, not a global root daemon.** `PM2_HOME=/home/<site>/.pm2`, a systemd unit generated per user, `pm2 save` on boot. This avoids cross-site interference and root-owned Node processes.
- **Node SSR uses blue/green ports.** Build in a release, start the new PM2 app on the idle port, health check, switch the web-server upstream, drain and stop the old app. **PM2 runs single-process by default**; cluster mode is opt-in per site.
- **Static React is a first-class site type.** Build in release, symlink `current`, serve files directly — zero downtime with no process management.
- **Caddy and nginx share a webserver contract.** Every generated config is validated before reload; SSL is abstracted (certbot for nginx, automatic HTTPS for Caddy, custom certs for both).
- **Octane is a special runtime adapter, FrankenPHP-first.** Zero-downtime symlink switching is disabled for Octane sites; deploys use `octane:reload` with health checks, per Forge's own guidance. Because Octane keeps the app in memory, set `octane.max_requests` (worker recycling), monitor RSS, and optionally schedule a graceful reload above a memory threshold. RoadRunner is an option; Swoole is later.

---

## 4. Architecture

### 4.1 Control plane (default)

```
Browser (React/Inertia)
        │
Laravel 13 app (actions, policies, API, CLI backend)
        │
PostgreSQL  ──► database queue ──► queue:work (2) + schedule:work
        │
phpseclib SSH ──► managed Ubuntu servers
```

- **Data:** PostgreSQL (required). SQLite only for tests.
- **Queue:** `database` driver; Redis + Horizon optional for large installs, never required.
- **Scheduler:** `schedule:work` in the worker container; no host cron required in Docker mode.
- **Realtime:** polling by default; websocket optional, never required for correctness.
- **Footprint target:** < 500 MB RAM steady state with Postgres on a 2 GB VPS; 1 GB possible with tuning.
- **Deployment:** Docker Compose (`app`, `worker`, `scheduler`, `postgres`) or the non-destructive installer.

### 4.2 Managed server layout

```
/home/<site-user>/<domain>/
├── current -> releases/<ts>        # atomic switch point
├── releases/<ts>/                  # per-deploy build
├── source/                         # optional canonical clone (classic mode)
├── shared/                         # .env, storage, sqlite, uploads
└── logs/
```

- One Linux user per site (isolation default on for new sites).
- nginx vhost or Caddy site block per site; PHP-FPM pool per site for PHP; PM2 app + systemd unit per Node site.
- Sudoers **allowlist** in `/etc/sudoers.d/eazydeploy` (reload nginx/caddy, reload php-fpm, supervisorctl, systemctl for own unit) — never `NOPASSWD:ALL`.
- Every generated file is tracked so uninstall and drift repair are possible.

### 4.3 Webserver abstraction

```php
interface Webserver {
    public function install(): void;
    public function validate(string $config): void;   // nginx -t | caddy validate
    public function apply(Site $site, string $config): void; // temp → validate → atomic move → reload
    public function reload(): void;
    public function ssl(Site $site): void;            // certbot | automatic | custom
    public function upstream(Site $site, string $host, int $port): void;
}
```

- **nginx:** `nginx -t` gate, certbot for Let's Encrypt, PHP via FastCGI socket, Node via `proxy_pass`.
- **Caddy:** `caddy validate --config`, automatic HTTPS, `php_fastcgi`, `reverse_proxy`, basic auth, redirects.
- **Both:** vhost preview, custom templates, domain/alias changes, force-SSL, websocket proxying (Reverb), rollback on invalid config.

### 4.4 Core state machines

| Table | Purpose |
|---|---|
| `provisioning_steps` | Per-server, per-step state: `pending/running/succeeded/failed/skipped`, attempts, log, error, timestamps. Enables resume/retry/repair |
| `deployment_phases` | Per-deployment phases with timings and logs; powers the timeline UI and failure diagnosis |
| `health_checks` | Site probes: URL, status, latency, region, checked_at; powers uptime, SSL, alerts, rollback decisions |
| `alert_rules` / `alert_events` | Thresholds, channels, dedup, recovery notices |
| `audit_logs` | Actor, action, target, IP, result, link to SSH log |

### 4.5 Deploy engine

```
lock → prepare → fetch → build → link → migrate? → switch → reload → verify → activate → cleanup
                                     │
                             health gate fails ──► rollback (switch back, reload, notify)
```

- **lock:** one explicit lock per site (DB row + server lock file); duplicate triggers are queued or rejected with a clear message.
- **fetch:** `git fetch --prune && git reset --hard origin/$BRANCH && git clean -fd` — never `git pull` on a dirty tree.
- **build:** raw build script from the site (defaults per runtime); per-phase timeout; output streamed to the deploy log.
- **link:** shared paths symlinked (`.env`, `storage`, SQLite, uploads).
- **switch:** `ln -sfn` + FPM reload (PHP) or upstream switch (Node) — validated before it happens.
- **verify:** HTTP probe with correct `Host`, 3 attempts / 2 s apart; 2xx/3xx/401/403 = healthy, 5xx/timeout = unhealthy.
- **activate:** only after verify passes; marks deployment active, restarts workers, prunes releases.
- **rollback:** switch back, reload, mark failed, alert. Never activate the failed release.
- **config safety:** write temp → `nginx -t`/`caddy validate` → atomic move + reload; restore previous file on failure.

### 4.6 Runtime adapters

| Runtime | Build | Activate | Process | Health |
|---|---|---|---|---|
| Laravel / PHP | composer, npm, artisan | symlink + FPM reload | FPM pool + Supervisor queues | HTTP probe |
| Laravel Octane | composer, npm, artisan | `octane:reload` (no symlink ZDD) | Octane under Supervisor | HTTP probe |
| WordPress / PHP | composer/npm optional | symlink + FPM reload | FPM pool | HTTP probe |
| Node SSR | npm/pnpm/yarn/bun install + build | start new PM2 app on idle port, switch upstream, drain old | PM2 per site user + systemd | HTTP probe on new port + public URL |
| Static React/Vite | npm install + build | symlink | none (web server serves `current/dist`) | HTTP probe |
| Bun | bun install + build | same as Node | PM2 or systemd | HTTP probe |

### 4.7 Laravel feature pack (v1)

| Feature | What we do |
|---|---|
| **Horizon** | One-click install, Supervisor program, dashboard auth gate, restart on deploy, metrics link |
| **Octane** | **FrankenPHP default** (RoadRunner optional), Supervisor process, `octane:reload` deploy mode, ZDD disabled with warning, `max_requests` recycling + RSS watch to contain memory growth |
| **Pulse** | Composer install, migrations, dashboard authorization, optional prune schedule |
| **Reverb** | Install, env/config, Supervisor process, websocket proxy in nginx/Caddy, TLS, restart on deploy |
| **Scheduler** | One cron entry per site, overlap protection, timezone, logs |
| **Queues** | Supervisor workers per queue, balance/processes, restart on deploy |
| **Storage** | `storage:link`, shared `storage/`, permissions verification |

### 4.8 API, deploy hooks, and CLI (v1)

- **API:** keep Vito's REST API + OpenAPI; add `POST /deploy`, `POST /rollback`, `GET /deployments`, `GET /health`, env pull/push, site/server reads.
- **Deploy hook:** per-site signed token, HMAC signature, branch/commit query params, rate limiting, replay protection.
- **CLI (`eazy`):** `login`, `servers`, `sites`, `deploy`, `rollback`, `logs`, `env:pull`, `env:push`, `status`. Thin client over the API; distributed as a PHAR (or a small standalone binary later).
- **CI examples:** GitHub Actions, GitLab CI, Bitbucket Pipelines in docs; deploy from CI is a Phase 1 acceptance test.

---

## 5. Phase plan

> Estimates are **engineering effort** for one senior developer. The AI agent implements; the owner reviews and tests. Calendar time depends on review/test cadence — assume **12–16 weeks to private beta** with steady feedback, against ~26 effort-weeks (128 working days).

### Phase 0 — Foundation & fork hygiene (Week 1)

**Objective:** make the fork safe to build on: reproducible, tested, Postgres-ready, and syncable with upstream.

**Scope**

- Pin the base (`4.1.1`), create `develop`, add `UPSTREAM.md` + sync script (monthly + security releases).
- CI: Pint, PHPStan, Pest, ESLint/typecheck, frontend build, `bash -n` + ShellCheck over rendered SSH scripts, architecture tests.
- **Postgres switch + compatibility audit:** migrations, JSON columns, date functions, `PRAGMA`/`strftime`, case sensitivity, `LIKE`/`ILIKE`, queue locking; CI matrix Postgres (full) + SQLite (fast subset).
- **Ubuntu 26.04 CI image** and smoke tests; verify PHP 8.5, Postgres 18, MySQL/MariaDB availability, and Hostinger-like images.
- `eazydeploy:doctor`: PHP extensions, APP_KEY, Postgres health, queue worker heartbeat, scheduler heartbeat, storage permissions, disk, webserver state.
- Control-plane backup/restore script (Postgres dump + storage) and upgrade script with pre-migration dump.
- Security quick wins: TLS 1.2+ in generated configs; idempotent `create-user`; stop overwriting `/root/.ssh/authorized_keys`; targeted sudoers (legacy `NOPASSWD` behind an explicit flag); installer does not rewrite host nginx by default.
- Branding basics: `APP_NAME=EazyDeploy`, logo/wordmark, favicon, login page; keep internal code namespaces for rebase cost.
- License hygiene: keep `LICENSE`/`NOTICE`, add upstream attribution, no header stripping.
- Plugin decision: disable marketplace/remote install; keep registration primitives; local-only plugin loading behind an admin flag (no UI yet).

**Deliverables:** green CI, Postgres-running app, Ubuntu 26 image, doctor, backup/restore, hardened installer, basic brand.

**Exit criteria:** fresh clone → `composer install && php artisan test` green; full suite on Postgres; doctor detects 8+ seeded failures; app boots on Ubuntu 26; upgrade script restores from backup in a drill.

**Estimate:** 6 working days.

---

### Phase 1 — Deploy Safety Engine + Webserver Abstraction + API/CLI (Weeks 2–5)

**Objective:** trustworthy deploys on both nginx and Caddy, automatable from CI from day one.

**Scope**

1. **Phase state machine** (`deployment_phases`): phases with timeouts, logs, cancel; timeline UI.
2. **Deploy lock:** DB + server lock; duplicate triggers rejected/queued; visible reason.
3. **Git hygiene:** `fetch + reset --hard + clean -fd` (no `-x`); branch-change safety; dirty-tree reporting.
4. **PHP-FPM reload / OPcache reset** after switch and rollback (configurable, default on).
5. **Webserver contract + implementations:** nginx and Caddy; temp write → validate (`nginx -t` / `caddy validate`) → atomic move → reload; restore previous on failure. Covers create/update vhost, domains, SSL, redirects, basic auth.
6. **Health gate + auto-rollback:** probe after switch; on failure revert, reload, mark failed, notify; never activate a failed release; fix `failed()` re-activation.
7. **Migration safety:** optional pre-migration snapshot; migration lock; explicit warning about non-reversible data changes.
8. **Release retention + disk guard:** configurable history; block below free-space threshold.
9. **Post-deploy checks:** workers restarted, scheduler present, `storage:link`, config/route cache, SSL valid.
10. **API + deploy hook + CLI:** deploy/rollback/status/logs/env endpoints; signed hooks; `eazy` CLI; OpenAPI; CI examples.
11. **Node/static groundwork:** runtime-neutral phases so Phase 5 plugs in without engine changes.

**Deliverables:** state machine, lock, health gate, nginx+Caddy adapters, API/CLI, timeline UI, E2E harness.

**Tests:** feature tests with `SSH::fake()` per phase/failure path; E2E on Ubuntu 26 with both nginx and Caddy; traffic-loop test (v1 → v2 → broken v3); CI deploy via GitHub Actions.

**Exit criteria:**

- 100 consecutive deploys on both webservers with no OPcache/FPM degradation; p95 < 60 s for a small app.
- Injected 5xx release → auto-rollback < 30 s, zero user-visible failed requests.
- Invalid config → rejected, other sites unaffected.
- Duplicate triggers → exactly one execution.
- Failed deploy never shows active.
- `curl` deploy hook from GitHub Actions succeeds; `eazy deploy` works.

**Estimate:** 20 working days. **Depends on:** Phase 0.

---

### Phase 2 — Provisioning Reliability & Preflight Doctor (Weeks 6–7)

**Objective:** make server onboarding boring and resumable on any Ubuntu VPS.

**Scope**

1. **`provisioning_steps` state machine:** refactor `InstallServer::install()` into idempotent steps (preflight, update, dependencies, per-service install, security baseline, finalize); attempts, backoff, per-step logs, resume.
2. **Preflight checks:** distro/version/arch, RAM/disk, existing web/db services, ports 80/443 and SSH port, IPv6, DNS, cloud firewall hints, SSH auth mode, sudo, time sync, provider metadata. Block with a fix list before touching the server.
3. **Apt robustness:** `DPkg::Lock::Timeout`, retries with jitter, noninteractive, `--force-confold`, held/broken package detection.
4. **Idempotent user/sudoers:** `useradd -m` when missing, append-only authorized keys, scoped sudoers, no MD5 passwords, optional password auth, rollback of partial users.
5. **Resume / Repair / Drift:** resume from failed step, re-run one step, reconcile server (services, vhosts, pools, firewall, webserver config); `servers:reconcile` command.
6. **Actionable failures:** every step maps to a message + suggested fix + log excerpt; notifications.
7. **Service install hardening:** version availability per OS (Ubuntu 26 first), PPA/repo validation, MySQL/Postgres failure modes, uninstall per service, both webservers.
8. **Hostinger provider + DNS:** VPS create/list/OS templates/regions/SSH keys/firewall; DNS A/AAAA/CNAME with validation and snapshots — as one provider among many.

**Exit criteria:** fresh Ubuntu 26 VPS end-to-end; kill control plane mid-provision → resume with no duplicates; every step run twice → no changes; preflight blocks seeded bad hosts with actionable messages; first-try success ≥ 98% on Ubuntu 26/24 x86_64; Hostinger create + deploy works.

**Estimate:** 12 working days. **Depends on:** Phase 0 (can run parallel with Phase 1 with two workstreams).

---

### Phase 3 — Postgres Runtime, Slim Profile & Non-destructive Installer (Weeks 8–9)

**Objective:** a lightweight, reliable control plane that never fights the host.

**Scope**

1. **Postgres production profile:** tuned config, connection pooling guidance, backup/restore, health checks, migration upgrade path; remove SQLite as a supported production option.
2. **Queue profile:** `database` queue, 2 workers + `schedule:work`; Redis/Horizon optional; queue stall detection.
3. **Realtime optional:** polling fallback; websocket only when enabled.
4. **Docker delivery:** single image, `app`/`worker`/`scheduler`/`postgres` services, healthchecks, resource hints, no default credentials, persistent volumes, versioned images, upgrade with backup + rollback.
5. **Non-destructive installer:** writes only its own compose/env/data dir; never host nginx/PHP/Redis; legacy installer behind an explicit flag.
6. **Performance tuning:** OPcache/FPM sizing by RAM, Postgres `shared_buffers`/`work_mem`, log rotation, queue restart policy.
7. **Self-monitoring:** control-plane health endpoint, worker/scheduler heartbeat, queue-stalled alert, disk warnings.

**Exit criteria:** on 2 GB Ubuntu 26: idle RAM < 500 MB (Postgres included), small deploy peak < 900 MB; worker kill → auto-restart, no lost jobs; fresh Docker install < 3 min; backup/restore drill passes.

**Estimate:** 12 working days. **Depends on:** Phases 0–1.

---

### Phase 4 — Onboarding UX, Blueprints & Laravel Feature Pack (Weeks 10–12)

**Objective:** from repo to production in one screen, with the Laravel features people actually use.

**Scope**

1. **One-screen deploy wizard:** server (new/existing) → repo/branch → domain → runtime → database (MySQL/MariaDB/Postgres/SQLite) → SSL → deploy. Defaults from repo detection.
2. **Repo detection:** Laravel, Symfony, WordPress, generic PHP, Next/Nuxt/Nest/Express, Vite/CRA static, Bun; package manager, build command, output dir, start command, PHP/Node version.
3. **Blueprints:** Laravel + (MySQL|MariaDB|Postgres|SQLite) + Redis/Valkey + Horizon + scheduler; Laravel API; Laravel Octane; Laravel Reverb; WordPress; Next.js SSR; React SPA; Express/Nest API; static HTML.
4. **Laravel feature pack:** Horizon, Octane (FrankenPHP default), Pulse, Reverb install/manage flows; scheduler; queues; storage link; maintenance mode. Octane ships with `max_requests` recycling, RSS monitoring, and an optional memory-threshold reload.
5. **Env editor safety:** raw editor default for existing files; structured mode validated; no quote corruption; diff + history + rollback.
6. **Import existing app:** adopt a server/path without redeploying; detect runtime, vhost, workers, cron, SSL; dry-run plan.
7. **Provider breadth:** unified provider connection flow (tokens, SSH keys, regions, validation); Hostinger, DO, Hetzner, Vultr, Linode, AWS; custom SSH fallback.
8. **Zero-downtime by default** for new sites; preflight summary of exactly what will change.

**Exit criteria:** 6 repositories (Laravel, WordPress, Next SSR, Vite React, Express API, static) deployed from scratch with zero SSH in < 10 min each; all four DB options validated; Horizon/Octane/Pulse/Reverb blueprints deploy successfully; Octane survives a 24-hour soak with RSS growth < 10% after worker recycling; import of an existing Laravel app with no downtime.

**Estimate:** 15 working days. **Depends on:** Phases 1–3.

---

### Phase 5 — Runtime Breadth: Node, React, Bun, PM2 (Weeks 13–15)

**Objective:** first-class non-PHP runtimes with the same safety guarantees.

**Scope**

1. **Node service:** install/manage Node versions (mise or nvm-style), per-site pinning; install PM2 as required; per-site `PM2_HOME`.
2. **PM2 integration:** per-site PM2 daemon under a systemd unit (`pm2-<user>`), `pm2 save`/`resurrect`, log streaming, restart/stop, memory restart limits, cluster mode option.
3. **Node SSR site type:** build/start/stop; blue/green PM2 apps on alternating ports; health check; upstream switch; drain old; env; logs.
4. **Static React site type:** Vite/CRA/Next static export; output dir; SPA fallback; cache headers; optional prerender hook.
5. **Bun site type:** install, build, run; same engine.
6. **Monorepo support:** app root relative to repo, workspaces-aware install/build, per-site package manager.
7. **Process management UI:** status, restart, stop, logs, resource usage; PM2 cluster presets.
8. **Docs/templates:** Next SSR, Vite React, Express/Nest, Bun; `VITE_*` env at build time; SSR hydration pitfalls; cache busting.
9. **Deferred to v2:** Remix/React Router v7 SSR.

**Exit criteria:** Next.js SSR deploy with zero downtime (traffic loop shows no 5xx/resets); React SPA atomic deploy; per-site Node version switch without sibling impact; PM2 crash → auto-restart + alert; memory limit enforced; PM2 survives reboot.

**Estimate:** 15 working days. **Depends on:** Phases 1, 3, 4.

---

### Phase 6 — Brand & UI Refresh (Weeks 16–17)

**Objective:** a product that feels like EazyDeploy, not a Vito fork: **calm, Vercel-like, high-contrast, data-dense without noise.**

**Scope**

1. **Design system:** typography, color tokens, spacing, iconography, dark mode, motion; custom logo/wordmark; remove Vito assets and strings from user-facing surfaces. Calm surfaces, restrained accent color, generous whitespace around dense tables, no marketing fluff.
2. **App shell:** sidebar/topbar, command palette (Cmd+K), breadcrumbs, global search, notifications center, empty states, skeletons, toasts.
3. **Dashboard:** fleet health, recent deploys, failing sites, resource cards, SSL/disk warnings, quick actions.
4. **Site page:** deploy timeline, health history, logs, env, workers, SSL, domains — dense but readable.
5. **Onboarding polish:** wizard visual pass, progress, error recovery, help text.
6. **Responsive + accessibility:** WCAG AA contrast, keyboard paths, focus states, mobile layouts.
7. **Visual regression:** Playwright screenshots for key pages; design review checklist.
8. **Constraint:** backend untouched; keep changes in `resources/js`, theme tokens, and user-facing config to limit upstream rebase conflicts.

**Exit criteria:** no visible Vito branding/assets; design review sign-off; Lighthouse accessibility ≥ 95 on key pages; mobile usable at 390 px; screenshot suite green.

**Estimate:** 12 working days. **Depends on:** Phase 4 (flows stable).

---

### Phase 7 — Observability, Alerts & Status (Weeks 18–19)

**Objective:** know when something breaks before users do.

**Scope**

1. **Uptime + latency probes** per site (path, interval, expected status), history, p95; internal probe from the server.
2. **SSL expiry** per domain, renewal verification, renewal-failure alerts.
3. **Resource monitors:** CPU, RAM, disk, inode, load; thresholds and recovery.
4. **Alert rules & routing:** email, Slack, Discord, Telegram, webhook; dedup, flapping protection, recovery notices, digest.
5. **Deploy notifications:** success/failure/duration/commit with links; CI webhook payloads.
6. **Site status page** (optional per project): public page from probe history; custom domain; incident notes.
7. **Log access:** tail nginx/Caddy/PHP/Node/deploy logs; search; no SSH.

**Exit criteria:** induced outage alerts < 2 min and resolves; dedup under flapping; SSL < 14 days warns; disk threshold warns; no duplicate notifications; log tail works for all runtimes.

**Estimate:** 12 working days. **Depends on:** Phases 1, 5.

---

### Phase 8 — Security, Permissions & Audit (Weeks 20–21)

**Objective:** safe to expose to a team and to the internet.

**Scope**

1. **Scoped sudoers:** verify allowlist covers every operation; remove `NOPASSWD:ALL` default; document residual risk.
2. **SSH hardening (opt-in, verified):** disable password/root login only after a second verified session; auto-revert on failure.
3. **Audit log:** actor, action, target, IP, result, SSH log link; append-only; export; UI filters.
4. **Permissions:** project/server-scoped roles (owner/admin/deployer/viewer), enforced in policies + API keys; deny-by-default tests.
5. **Secrets hygiene:** redaction in logs/notifications; APP_KEY rotation; encrypted-at-rest verification.
6. **Supply chain:** signed control-plane releases/checksums; local plugin hash pinning; dependency audit; CVE runbook.
7. **Hardening defaults:** rate limits on deploy hooks/login, 2FA enforcement option, session policy, security headers in vhosts.
8. **Threat model + pen-test** of the control-plane API and SSH boundary.

**Exit criteria:** cross-project/server access tests deny; every privileged action audited; hardening cannot lock out the operator (simulated failure reverts); scans clean or documented; pen-test findings triaged.

**Estimate:** 12 working days. **Depends on:** Phases 2–5.

---

### Phase 9 — Backups, DR, Docs & Private Beta (Weeks 22–23)

**Objective:** recoverable, upgradeable, documented, and ready for sustained internal use.

**Scope**

1. **Database backups:** schedules, S3-compatible/B2/Drive/Dropbox, compression + encryption, verification via scratch restore, retention, one-click restore, alerts.
2. **File backups:** site files + shared paths, excludes, retention, restore.
3. **Pre-deploy snapshots** integrated with Phase 1.
4. **Control-plane DR:** self-backup (Postgres + storage), documented restore, versioned upgrade with rollback.
5. **Migration from Vito:** importer/guide for projects, servers, sites (best effort, documented limits).
6. **Docs:** install, quickstart per runtime, troubleshooting, architecture, API, CLI, upgrade, security.
7. **Release process:** versioning, changelog, signed tags, beta/stable channels, opt-in telemetry only.
8. **Performance pass:** N+1 audit, query budgets, indexes, frontend bundle budget.
9. **Private beta:** owner-run checklist, 2 weeks of dogfooding, incident log, fix list. The first external wave is **1–2 trusted developers**, tentatively after Phase 7; give them read-only access until the Phase 8 security and Phase 9 backup gates pass, then full write access.

**Exit criteria:** restore DB + files to a clean environment in < 15 min via UI + docs; upgrade N-1 → N with backup and one-command rollback; runbook rehearsed; 2-week dogfood with ≥ 99% deploy success and no data loss.

**Estimate:** 12 working days. **Depends on:** Phases 0–8.

---

### Phase 10 — Commercial Readiness (optional, post-sign-off)

Only if the owner decides to open it up after private use.

- Billing/subscriptions, multi-tenant organizations, SSO (OIDC/SAML), SCIM.
- Whitelabel, status pages as a product, plugin marketplace (sandboxed), MCP server, mobile PWA, CLI SDK.
- Hosted control-plane offering with AGPL §13 source-offer compliance.

**Decision gate:** 4–6 weeks of internal use, ≥ 99% deploy success, no data-loss incidents, and a deliberate licensing decision (open-core AGPL vs clean-room rewrite).

---

## 6. Parity map (Forge/Ploi → our phases)

| Capability | Forge | Ploi | EazyDeploy |
|---|---|---|---|
| Zero-downtime default | ✅ | ✅ | **P1** |
| Health checks | ✅ notify | ✅ uptime | **P1 + auto-rollback** |
| Deploy retention/shared paths | ✅ | ✅ | **P1** |
| Push-to-deploy + hooks | ✅ | ✅ | **P1** |
| Deploy API + CLI | ✅ | ✅ | **P1** |
| nginx + Caddy | nginx only | nginx (+Apache) | **P1/P4** |
| Resumable provisioning | n/a | n/a | **P2** |
| Preflight doctor | n/a | n/a | **P2** |
| Hostinger provider/DNS | ❌ | ✅ provider | **P2** |
| Postgres control plane | n/a | n/a | **P0/P3** |
| Blueprints + DB choices | partial | ✅ | **P4** |
| Horizon/Octane/Pulse/Reverb | ✅ | partial | **P4** |
| Node + PM2 | partial | ✅ | **P5** |
| Static React/Next SSR | partial | partial | **P5** |
| Uptime/SSL/disk alerts | ✅ paid | ✅ | **P7** |
| Status pages | ❌ | ✅ | **P7** |
| Audit log + granular roles | partial | ✅ | **P8** |
| Backups + restore | ✅ | ✅ | **P9** |
| Teams | ✅ | ✅ | **P8/P10** |
| Plugin marketplace | ❌ | ✅ | **P10 (sandboxed), not v1** |
| MCP/AI | ❌ | ✅ | **P10** |

---

## 7. Cross-cutting workstreams

| Workstream | Cadence | Notes |
|---|---|---|
| **Testing** | Every phase | Pest unit/feature (`SSH::fake`), shell contract (`bash -n`, ShellCheck, BATS), E2E Ubuntu 26 VM with nginx **and** Caddy, Postgres CI, nightly matrix |
| **Upstream sync** | Monthly + security releases | Rebase `develop` on `upstream/4.x`; upstream generic fixes; resolve conflicts in small batches; UI refresh late to avoid churn |
| **Documentation** | Every phase | Docs are part of done, not a phase |
| **Security** | Continuous | Dependency/secret scans, threat model, pen-test before beta, CVE runbook |
| **Licensing compliance** | Continuous | Preserve AGPL notices; no relicensing of fork code; legal review before commercial |
| **Quality gates** | Every PR | Pint, PHPStan, Pest, ESLint, typecheck, architecture tests, shell checks |
| **Owner review** | Every phase | Demo + test checklist; sign-off is the phase gate |

---

## 8. KPIs & definition of "production grade"

| Metric | Target |
|---|---|
| Deploy success rate (non-code failures excluded) | ≥ 99% |
| p95 deploy duration, small app | < 60 s |
| Zero-downtime violations | 0 |
| Auto-rollback time | < 30 s |
| First-try provisioning success (Ubuntu 26/24) | ≥ 98% |
| Control-plane idle RAM (2 GB profile) | < 500 MB |
| Fresh control-plane install (Docker) | < 3 min |
| Node process recovery after crash | < 10 s, automatic |
| Octane RSS growth after 24 h soak | < 10% with worker recycling |
| Alert latency (site down) | < 2 min |
| Backup restore to clean environment | < 15 min |
| Audit coverage of privileged actions | 100% |
| UI accessibility (key pages) | Lighthouse ≥ 95 |

---

## 9. Risk register

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Upstream drift makes rebases painful | High | Medium | Small modular diffs; upstream-first; monthly sync; UI/brand refresh late |
| Security incident (root SSH management) | Medium | Critical | Threat model, scoped sudo, audit, signed releases, pen-test, CVE process |
| Scope creep (Forge+Ploi parity) | High | High | Non-goals, phase exit criteria, owner sign-off |
| **AGPL constrains future licensing** | High | High | Keep AGPL now; open-core or clean-room rewrite only if commercial; legal review |
| **Caddy parity gaps** (SSL, redirects, websockets) | Medium | Medium | Shared webserver contract; E2E both servers; feature matrix tests |
| **PM2 multi-user isolation** | Medium | High | Per-site `PM2_HOME`, systemd unit per user, no global root daemon, tests |
| **Postgres overhead on small VPS** | Medium | Medium | Tuned config, 2 GB recommendation, external DB option, monitoring |
| **Octane/FrankenPHP memory growth** | Medium | High | `max_requests` worker recycling, RSS monitoring, threshold reload, 24 h soak test, per-site restart policy |
| Node blue/green complexity | Medium | High | Prototype in P1, traffic-loop tests, PM2 drain, rollback |
| Provider API changes | Medium | Medium | Thin adapters, recorded fixtures, custom SSH fallback |
| E2E VM cost/flakiness | Medium | Medium | Containerized harness + nightly cloud smoke |
| Solo reviewer bottleneck | High | Medium | Phase-sized PRs, demo checklists, automated tests before review |
| UI divergence increases rebase cost | Medium | Medium | Backend untouched; design tokens; late refresh; isolated `resources/js` changes |

---

## 10. Decisions log

| # | Decision | Status |
|---|---|---|
| 1 | Fork Vito 4.x and harden rather than greenfield | ✅ |
| 2 | Repo/brand `eazydeploy`; internal-first, no public release yet | ✅ |
| 3 | Provider-agnostic, Ubuntu-first; **Ubuntu 26.04 default**, 24.04 supported | ✅ |
| 4 | Laravel/PHP + Node SSR + static React + Bun in v1; Remix/React Router v7 SSR in v2 | ✅ |
| 5 | **PM2 required** for Node, per-site, supervised by systemd | ✅ |
| 6 | **Postgres-only** control plane; SQLite tests only | ✅ |
| 7 | Database queue by default; Redis/Horizon optional | ✅ |
| 8 | **nginx + Caddy both first-class** in v1 | ✅ |
| 9 | **Forge Laravel pack in v1**: Horizon, Octane, Pulse, Reverb, scheduler, queues | ✅ |
| 10 | **API + CLI in v1**, deploy hooks from Phase 1 | ✅ |
| 11 | Blueprints include MySQL, MariaDB, PostgreSQL, SQLite | ✅ |
| 12 | Auto-rollback on failed health check | ✅ |
| 13 | Deliberately reload PHP-FPM after release (OPcache realpath accumulation) | ✅ |
| 14 | Keep AGPL for the fork; no unilateral relicensing | ✅ |
| 15 | Keep plugin registration primitives; **remove marketplace/remote install** in v1 | ✅ |
| 16 | Full brand/UI refresh in Phase 6; backend untouched | ✅ |
| 17 | Keep upstream code namespaces until rebrand phase; user-facing brand now | ✅ |
| 18 | Octane uses `octane:reload`, ZDD disabled | ✅ |
| 19 | `eazy` CLI as PHAR in v1; standalone binary deferred | ✅ |
| 20 | **nginx default** for new sites; Caddy one click away, both first-class | ✅ |
| 21 | **PM2 single-process + blue/green default**; cluster mode opt-in per site | ✅ |
| 22 | **Octane defaults to FrankenPHP**, with `max_requests`, RSS watch, threshold reload; RoadRunner optional | ✅ |
| 23 | UI direction: **calm, Vercel-like**, high-contrast, data-dense without noise | ✅ |
| 24 | Private beta wave 1: **1–2 trusted developers after Phase 7** (read-only until Phase 8/9 gates) | ✅ |

---

## 11. Immediate next 10 working days

1. Create `develop`; pin `4.1.1`; add `UPSTREAM.md` + sync script.
2. Switch local/dev/CI to Postgres; run the compatibility audit; fix migrations/queries; add CI matrix (Postgres full, SQLite fast).
3. Add Ubuntu 26.04 to CI and verify package/service availability (PHP 8.5, Postgres 18, MySQL/MariaDB).
4. Green CI + shell linting + architecture tests.
5. `eazydeploy:doctor` + control-plane backup/restore + upgrade drill.
6. Harden `create-user` (idempotent, scoped sudoers, no root key clobbering) and generated TLS defaults.
7. Introduce `deployment_phases` + timeline scaffolding.
8. Replace `git pull` with `fetch/reset --hard`; add FPM reload after release/rollback.
9. Implement the webserver contract; add `nginx -t` and `caddy validate` gates for both servers.
10. Add health probe + auto-rollback; fix failed-deploy activation.
11. Add deploy API endpoint + signed hook + `eazy deploy`; verify from GitHub Actions.
12. Stand up the E2E harness (Ubuntu 26 + nginx + Caddy + Postgres) and traffic-loop test.
13. Demo: Laravel on nginx, static React on Caddy, broken third release auto-rolls back.

---

## 12. Resolved decisions (formerly open questions)

| Question | Decision |
|---|---|
| Caddy default vs nginx default | **nginx default** for familiarity; Caddy one click away and fully tested |
| PM2 mode | **Single process + blue/green default**; cluster opt-in |
| CLI distribution | **PHAR in v1**; standalone binary later if needed |
| Octane server | **FrankenPHP default**; RoadRunner optional; memory safeguarded via `max_requests` + RSS watch |
| Design direction | **Calm, Vercel-like**; high-contrast, dense data tables, no marketing fluff |
| Private beta audience | **1–2 trusted developers** after Phase 7; read-only until security/backup gates |

No blocking questions. Awaiting owner go-ahead to start Phase 0.
