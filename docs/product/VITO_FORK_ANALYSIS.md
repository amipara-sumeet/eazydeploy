# Vito 4.x — Fork & Productization Analysis

**Analysed:** `vitodeploy/vito` @ `8598e4d` (7 Oct 2026), default branch `4.x`, latest release `4.1.1` (21 Sep 2026)
**Stack:** Laravel 13 / PHP 8.4 / Inertia 3 / React / Tailwind / Shadcn, phpseclib for SSH, Redis + Horizon for queues
**Size:** 885 PHP files, ~69k LOC app code, 188 test files (~48k LOC tests), 447 frontend files
**License:** AGPL-3.0
**Community:** ~3.2k stars, 421 forks, ~13 open issues, 386 closed, active releases through Sep 2026

---

## 1. Verdict (TL;DR)

Vito is **not a toy and not broken**. It is a mature, broad, actively maintained self-hosted server manager. Rewriting it from scratch would cost 6–12 months and lose the integrations, UI, API, and plugin ecosystem you liked in the first place.

The real gap is **not features — it is trust and onboarding**:

1. **Deployments are not safe enough for production.** Modern deployment is still labelled beta, does not reload PHP-FPM/OPcache after switching releases, does not validate nginx config before reload, has no health gate or automatic rollback on a bad release, and can report success when the deploy actually failed.
2. **Provisioning is a one-shot linear job.** If one apt step fails (very common on budget VPS), you get a half-installed server and the only remedy is delete-and-recreate. There is no resume, per-step retry, idempotency, repair, or drift detection.
3. **The control plane is heavier than its job.** Vito uses SQLite for data but hardcodes Redis for queues and requires Horizon, a websocket process, cron, PHP-FPM and nginx. The bare-metal installer takes over `/etc/nginx/nginx.conf`, installs PHP/Node/Redis, and even enables TLS 1.0/1.1. It assumes a **dedicated, fresh VPS** — a bad fit when you point it at a Hostinger box that already has a stack on it.
4. **The Hostinger path is entirely manual.** Vito has providers for DigitalOcean, Hetzner, Vultr, Linode, AWS/Lightsail, Proxmox and "Custom" — but no Hostinger. DNS integration exists only for Cloudflare. Hostinger *does* have a full public API for VPS and DNS (67 VPS endpoints, 8 DNS endpoints), so this is an integration gap, not a platform limitation.
5. **Too many concepts before the first deploy.** Project → server → services → site → source control → branch → isolated user → env → deployment script → SSL → worker → cron. Great for control, hostile for "I just want my Laravel app live".

**Recommendation:** Fork 4.x, keep the data model/UI/SSH layer/API/plugins, and rebuild three things: **onboarding, provisioning orchestration, and the deploy engine**. Position it narrowly:

> **"Deploy Laravel to a cheap VPS with Forge-level safety and zero SSH — Hostinger first."**

MVP in **6–8 weeks** for one full-time developer; production beta in **~12 weeks**. Do **not** try to out-feature Vito.

**Licensing warning up front:** AGPL-3.0 means if you run a modified version as a network service, users must be able to get the source. You can sell hosting, support, and branded distributions; you cannot keep your modifications closed while offering it as a service. Rebrand — do not use the Vito name or logo.

---

## 2. What Vito actually is

### Runtime architecture

```
Browser (React/Inertia)
        │
Laravel app (actions, policies, API)
        │
Redis + Horizon ──► Jobs (InstallJob, DeployJob, BackupJob, …)
        │
phpseclib SSH ──► managed servers (Ubuntu 20/22/24/26)
        │
Blade-rendered bash scripts + Mustache vhost templates
        │
nginx / Caddy, PHP-FPM, MySQL/MariaDB/Postgres, Redis/Valkey, Supervisor, UFW, fail2ban
```

- **Control plane storage:** SQLite by default (`config/database.php`), but **queues are hardcoded to Redis** (`config/queue.php` — both `default` and `ssh` connections use the `redis` driver). Horizon is therefore mandatory.
- **Realtime:** custom Ratchet websocket server (`php artisan ws:serve`) on port 8085, nginx-proxied at `/ws/`.
- **Scheduling:** Laravel scheduler via cron; 13 scheduled tasks including metrics every minute, server checks every 5 minutes, backups every minute, auto-updates every minute.
- **Secrets:** provider credentials, SSH passwords, SSL keys, database passwords, worker env, plugin/API tokens are encrypted casts with `APP_KEY`. Good baseline.
- **Auth:** Fortify, 2FA, roles (user/admin), projects for scoping, API keys with read/write and project scope.

### Feature surface (as of 4.1.1)

| Area | What exists |
|---|---|
| Server providers | AWS, DigitalOcean, Hetzner, Lightsail, Linode, Proxmox, Vultr, Custom |
| OS | Ubuntu 20.04, 22.04, 24.04, 26.04 only |
| Web servers | nginx, Caddy (beta) |
| PHP | 5.6 → 8.5, multiple versions per server, isolated FPM pools per site |
| Databases | MySQL 8.4/9.7, MariaDB 10.11–12.3, PostgreSQL 15–18 |
| Other services | Redis, Valkey, Supervisor, UFW, fail2ban, WireGuard, GoAccess, Vito monitoring agent |
| Site types | Laravel, PHP, WordPress, PHPMyAdmin, Node, Bun, load balancer, blank |
| Deploy | Classic script deploy; "modern" releases + symlink (beta, Laravel only, no Octane) |
| Ops | SSL (Let's Encrypt/custom/CSR), backups (S3/Dropbox/etc.), cron, workers, firewall, monitoring metrics, notifications, server logs |
| Automation | Workflows, API + Swagger, plugins (providers, DNS, site types, services, notification channels, workflow actions, views) |
| DNS | Cloudflare only |

### Where the extension points are

- `app/Plugins/Register*` — providers, DNS, storage, source control, notification channels, services, site types, site features, server features, workflow actions, console commands, views.
- `app/SiteTypes/*` — new application archetypes.
- `app/Services/*` — new services with install/manage handlers.
- `app/Actions/*` + `app/Jobs/*` — orchestration, but **not pluggable**.

That last point matters: plugins can add integrations, but they **cannot fix the deploy engine, provisioning, or the installer**. Those require a fork.

---

## 3. What Vito gets right (keep it)

1. **Breadth with a coherent model.** Project → Server → Site → Deployment is a good domain model, and the UI is consistent.
2. **Real SSH orchestration.** phpseclib, per-command logs, `set -e`/`pipefail`, aliases, env injection, isolated Linux users per site.
3. **Isolated users done properly (for new sites).** `create-isolated-user.blade.php` is idempotent, reasserts permissions, and adds keys without duplication. The main `create-user` path is not — see below.
4. **Modern deployment data model.** Releases, `current` symlink, shared resources (`.env`, `storage`), configurable history, rollback to any release. The bones are right.
5. **Security baseline.** Encrypted secrets, 2FA, API key scoping, GitHub webhook HMAC verification, UFW rule backup/restore around changes, SSL expiry checks.
6. **Operational features most competitors charge for.** Backups, monitoring, workers, cron, firewall, DNS, notifications, API, workflows.
7. **Test culture.** ~48k LOC of tests for ~69k LOC of app code is unusually good for this category.
8. **Plugin surface.** Genuinely useful for adding Hostinger, DNS providers, and site presets without forking core.

---

## 4. Where it falls short (with evidence)

### 4.1 Control-plane install & lifecycle

| Problem | Evidence | Impact |
|---|---|---|
| Installer takes over the host | `scripts/install.sh` rewrites `/etc/nginx/nginx.conf`, sets `user vito`, removes `sites-enabled/default`, installs PHP/Node/Redis/Supervisor, disables `needrestart` | Cannot coexist with an existing app/panel. On Hostinger, this is the most likely source of "errors" if Vito was installed on the same VPS as the app |
| Insecure TLS defaults | `ssl_protocols TLSv1 TLSv1.1 TLSv1.2` in the generated nginx.conf | Fails security scanners; TLS 1.0/1.1 are deprecated |
| No transactional upgrade | `scripts/update.sh` does `git reset --hard`, `git clean -fd`, `composer install`, `migrate --force`, restart workers — no DB backup, no rollback, no signature/checksum verification | A bad release or migration can brick the control plane with no undo |
| Heavy runtime for small VPS | Redis + Horizon + websocket + scheduler + PHP-FPM + nginx; Horizon `maxProcesses=3` per queue by default | 1–2 GB VPS runs hot; open issues #1213–#1216 document unbounded SSH jobs and scheduler pressure |
| Docker image is dev-grade | `docker/docker-compose.yml` has hardcoded `APP_KEY`, `PASSWORD: "password"`, no healthcheck, no resource limits; supervisord runs everything in one container | Not a production deployment story |
| Silent dependency failures | If `ws:serve`, cron, or Horizon dies, UI stops updating / schedules stop; little self-monitoring | Hard to diagnose, especially for non-experts |

### 4.2 Server onboarding & provisioning

| Problem | Evidence | Impact |
|---|---|---|
| No preflight | Custom provider trusts "root SSH + Ubuntu"; no checks for distro/arch/disk/RAM, existing nginx/apache/mysql, ports 80/443, IPv6, cloud firewall, DNS | Failures surface 10 steps later, after the server is half-modified |
| Destructive user setup | `create-user.blade.php`: `useradd` without `-m`, `openssl passwd -1` (MD5), appends `NOPASSWD:ALL` to sudoers every run, and for custom servers **overwrites `/root/.ssh/authorized_keys`** | Non-idempotent; repeated installs corrupt sudoers; can remove the operator's own root key |
| Name collision | Creating a `vito` user fails if one already exists (fixed for isolated users, not the main user); #1103 documents the reverse false-positive guard | Re-provisioning or shared hosts break |
| One-shot linear install | `app/Actions/Server/InstallServer.php` runs 5→100% in one job; no per-step checkpoint, retry, or resume | Any transient failure = delete server and start over |
| No apt lock handling | `upgrade.blade.php` calls `apt-get` without waiting on locks | #1205: "Could not get lock … held by process apt-get", fails at 15% |
| Narrow OS matrix | Ubuntu only (20–26) | No Debian, Alma, Rocky, Amazon Linux; many budget providers ship Debian |
| Missing provider/DNS integrations | No Hostinger, no Hetzner DNS, no Route53, no DigitalOcean DNS; Cloudflare only | Manual DNS for most users |
| Service install fragility | MySQL 8.4 and PostgreSQL install failures reported (#1101, #1190); ondrej PPA dependency for PHP | Provisioning success rate varies by image/region |

### 4.3 Deploy engine (the most serious gap)

| Problem | Evidence | Impact |
|---|---|---|
| No PHP-FPM reload / OPcache reset after release switch | `resources/views/ssh/modern-deployment/release.blade.php` only does `ln -sfn`; `DeployJob` never reloads FPM. Open issue #1273 documents an outage after 7 releases as OPcache filled with old releases | **Production outage risk that grows with every deploy** |
| No nginx config validation | `create-vhost.blade.php` and `Nginx::updateVHost()` write the file and immediately `service nginx reload`; no `nginx -t` | One bad vhost takes down **every site** on the server |
| No health gate / auto-rollback | Deploy marks finished and activates immediately after the script exits; rollback only on script failure, and `failed()` re-activates the failed deployment | A release that "deploys" but returns 500 stays live; #1202 reports failed deploys marked active |
| Rollback is symlink-only | `RollbackJob` / docs: no scripts run, no FPM reload, no cache clear | DB migrations are not rolled back; app caches may point at the wrong release |
| Git state can diverge | Modern deploy uses `git pull` in `link-resources.blade.php`; #1256 shows a branch change leaves the repo divergent forever, requiring manual `git reset --hard` | Deploys fail indefinitely until someone SSHes in — exactly the manual effort you want to avoid |
| No deploy lock across triggers | `UniqueQueue` helps per site, but webhook/manual/API/retry paths are not a single explicit lock with cancellation | Concurrent/duplicate deploys, confusing states |
| No phase timeouts or cancellation | One long `runScript` per phase; queue timeout only | Hung composer/npm/migration blocks the deploy queue |
| Modern deployment is beta and narrow | Laravel only, no Octane, shared-resource links, manual path updates for workers/cron | The "safe" path is the one least battle-tested |

### 4.4 Security posture

- **Recent fixed vulnerabilities** show the authorization surface is complex: #1267 cross-project command execution, #1218 authenticated arbitrary file read via env path override, #1231 API key project-scope bypass. All fixed — but this is a moving target that needs continuous audit, not a one-time review.
- **Passwordless sudo** for the deploy user (`NOPASSWD:ALL`) is broader than needed. Targeted sudoers entries (`systemctl reload nginx`, `systemctl reload php*-fpm`, etc.) would materially reduce blast radius.
- **Installer executes remote code** (`curl | bash`) with no pinning or signature verification.
- **Plugin supply chain:** plugins are installed from GitHub zips; no signing/verification model.
- **No audit log** of who ran what command on which server.
- **No SSO/SAML/SCIM**, no fine-grained resource permissions (#861 open).

### 4.5 Operability

- No built-in uptime/SLO checks or alerting beyond notification channels (#863 "Monitoring alerts" open).
- Metrics are collected every minute **per server over SSH**, with no global concurrency bound (#1215).
- Backups exist but corruption/empty-upload bugs have been reported (#1071, #1073); no restore drill/verification workflow.
- Control plane is single-node SQLite; no HA story (acceptable for the target market, but should be stated).
- Debugging relies on Horizon + log viewer; no unified cross-server log search or deploy timeline.

### 4.6 UX / manual effort

- No guided "import existing Laravel app" flow. You assemble it concept by concept.
- No blueprints ("Laravel + Horizon + scheduler + Redis + MySQL", "WordPress + cache", etc.).
- DNS is manual unless Cloudflare.
- `.env` editor has corrupted quoted values (#1227) — a terrifying class of bug for a deploy tool.
- No staging → production promotion, preview environments, or PR deploys.
- No drift detection between desired state and actual server state (vhost generation is opt-in, but services/files drift silently).

---

## 5. Why Hostinger specifically felt painful

Given your description ("too much manual effort, few errors"), the likely failure chain is:

1. **Control plane placement.** If Vito was installed on the same Hostinger VPS as the Laravel app, `install.sh` rewrites nginx, installs its own PHP/Node/Redis, and binds `/home/vito/vito`. That alone can break the existing site. Even on a separate VPS, a 1–2 GB Hostinger plan is tight for Vito + Redis + Horizon + MySQL.
2. **Custom provider onboarding.** Hostinger VPS is Ubuntu with root SSH, but Vito's flow assumes a pristine box. Preinstalled packages, an existing `vito` user, apt locks, or a provider firewall blocking 80/443 cause failures at 15–45% with little actionable guidance.
3. **Destructive key handling.** For custom servers, Vito rewrites `/root/.ssh/authorized_keys` with its own key. If you rely on your own key/password, this is surprising at best, lockout-adjacent at worst.
4. **DNS is manual.** Hostinger nameservers are not integrated, so every domain needs hand-created A/AAAA/CNAME records, then a wait, then SSL issuance.
5. **SSL fragility.** `certbot --nginx` mutates the vhost; Vito also rewrites vhosts. Renewal issues have been reported (#1058). Any mismatch in server_name/DNS and issuance fails after the site is already "created".
6. **Deploy surprises.** After all that, the first deploy can fail on git divergence, isolated-user key reuse (#1272), or a false "active" state (#1202), and the modern path can degrade PHP-FPM over time (#1273).
7. **Too many knobs.** PHP version, web directory, isolated user, deployment script, env, workers, cron, firewall — each a place to get it wrong, with no "golden path" defaults.

**Conclusion:** the pain is structural, not a single bug. Fixing it means owning the onboarding and deploy engine, which is exactly what a focused fork can do.

---

## 6. Build options

| Option | Effort | Control | Risk | Verdict |
|---|---|---|---|---|
| **A. Fork 4.x and harden** | 6–8 wks MVP, 3–6 months v1 | High | Upstream drift; AGPL; security liability | **Recommended**, with a narrow thesis |
| **B. Plugin-only on upstream** | 2–4 wks | Low | Cannot fix installer, provisioning, deploy engine, or queue model | Good complement, not sufficient alone |
| **C. Greenfield (Go single binary + React)** | 6–12 months | Total | Rebuild all integrations/UI; no ecosystem | Only if you want a commercial product and can staff it |

**Recommended: A + B.** Fork for the core fixes; ship Hostinger/DNS/presets as plugins where possible; upstream generic fixes to reduce divergence.

A greenfield Go control plane is genuinely attractive long-term (single binary, embedded SQLite, no Redis/Horizon, trivial install), but it is a different company-sized bet. Do not start there unless you have 6–12 months and a team.

---

## 7. Product thesis

**Working name:** pick a neutral brand (e.g. "Bowline", "Gangway", "Harbor" is taken by CNCF). Never ship as "Vito".

**ICP:** solo developers and small agencies running Laravel on $5–20/month VPSs, especially Hostinger/Hetzner/Contabo/DigitalOcean, who want Forge/Ploi ergonomics without per-site pricing.

**Promise:** *Connect a repo, pick a domain, get a production Laravel app in under 10 minutes — with deploys that cannot take the site down.*

### The three pillars

#### 1. Trustworthy deploys
- Explicit deploy state machine with phases: `prepare → fetch → build → link → migrate → switch → reload → verify → activate | rollback`.
- **Health gate:** after switching `current`, probe the app (HTTP from the server with the correct Host header + optional path) and require N consecutive 2xx/3xx; otherwise auto-revert symlink, reload FPM, alert.
- **`nginx -t` before every reload.** Write to a temp file, test, then atomically move + reload; on failure keep the old config and show the diff.
- **Reload PHP-FPM / reset OPcache after switch and rollback.** Graceful reload, per PHP version.
- **Git hygiene:** `git fetch --prune && git reset --hard origin/$BRANCH && git clean -fd` in a dedicated clone; never `git pull` in a dirty tree.
- **Migration safety:** pre-deploy DB snapshot (configurable), migration lock, `migrate --force --isolated`, explicit warning that code rollback does not roll back data; optional snapshot restore.
- **Deploy lock + cancellation + phase timeouts**, with a live timeline in the UI.
- **Release pruning + disk guard** (warn/block before disk fills).
- **Post-deploy checks:** queue workers restarted, scheduler present, storage link, config/route cache, SSL expiry, app URL responds.

#### 2. Zero-manual onboarding
- **Hostinger provider:** create VPS via API (OS template, region, plan, SSH key, firewall), poll until running, then provision.
- **Hostinger DNS:** create/update A/AAAA/CNAME, validate, snapshot/rollback.
- **Preflight doctor:** distro, arch, RAM/disk, existing web/db services, ports, IPv6, DNS resolution, provider firewall, SSH auth mode, sudo, time sync. Blocks with actionable messages before touching anything.
- **One-screen "Deploy Laravel" flow:** repo URL + branch + domain + PHP version + DB choice. Everything else defaulted.
- **Import existing app:** detect Laravel/PHP/Node, existing `.env`, `public/`, storage, queue/scheduler usage; propose a migration plan.
- **Blueprints:** Laravel + MySQL + Redis + Horizon + scheduler; WordPress + cache; Node/Bun; API-only.
- **Automatic DNS + SSL** once the provider is connected.

#### 3. Boring operations
- **Slim control plane profile:** SQLite + **database queue** + `queue:work` + `schedule:work` in one container; Redis/Horizon/websocket **optional**. Target: runs comfortably on 1 GB.
- **Resumable provisioning:** `provisioning_steps` table with state, attempts, logs, and idempotent scripts; "Resume" and "Repair server" actions.
- **Reconciler:** periodic drift check (services, vhosts, PHP pools, firewall, SSL) with one-click fix.
- **Alerts that matter:** deploy failed, site down, SSL < 14 days, disk > 85%, backup failed, server unreachable. Email/Slack/Discord/Telegram.
- **Control-plane backup/restore + upgrade with rollback** (DB dump before migrate).
- **Audit log:** who did what, when, on which server, with command output link.

### Non-goals (for v1)
Kubernetes, autoscaling, Docker app hosting, CI runners, multi-cloud networking, a full PaaS. Stay narrow and excellent.

---

## 8. Technical plan

### Keep
- Domain model: `Project`, `Server`, `Site`, `Service`, `Deployment`, `Ssl`, `Database`, `DatabaseUser`, `Worker`, `CronJob`, `Backup`, `ServerLog`.
- SSH layer (`app/SSH`), Blade script rendering, Mustache vhost templates, isolated users.
- Service handlers and site types (extend, don't rewrite).
- React/Inertia UI shell, API + OpenAPI, plugin registry.
- Encrypted casts and 2FA.

### Replace / add

| Area | Change |
|---|---|
| Queue | Add `database` queue connections; make Redis optional; replace Horizon with `queue:work` + `schedule:work` in the default profile. Keep Horizon as an optional "large install" mode |
| Realtime | Make websocket optional; fall back to polling/Inertia reloads; never let a dead ws process look like a dead app |
| Installer | New non-destructive Docker Compose (app, worker, scheduler, optional ws/redis) + a `curl` installer that **only** writes its own compose/env and does not touch host nginx. Bare-metal mode becomes opt-in and isolated |
| Provisioning | New `provisioning_steps` + state machine; idempotent scripts; retries with jittered backoff; apt lock wait (`-o DPkg::Lock::Timeout=300`); per-step logs; resume/repair |
| Deploy | New phase state machine + lock + timeouts + health gate + rollback; FPM reload; nginx test; git reset; migration snapshot; timeline UI |
| Security | Targeted sudoers instead of `NOPASSWD:ALL`; no root key clobbering; opt-in SSH hardening with a verified second session before disabling password auth; audit log; signed release artifacts; plugin hash pinning |
| Hostinger | New `ServerProvider` + `DNSProvider` using the official API (PHP SDK available); region/OS template mapping; firewall + snapshot integration |
| Observability | Uptime probes, per-site health history, alert rules, deploy metrics (duration, failure rate), control-plane self-health |

### New schema (indicative)
- `provisioning_steps(id, server_id, key, status, attempts, started_at, finished_at, log_id, error)`
- `deployment_phases(id, deployment_id, key, status, started_at, finished_at, log_id, error)`
- `health_checks(id, site_id, url, status_code, latency_ms, checked_at)`
- `audit_logs(id, user_id, project_id, server_id, action, subject_type, subject_id, meta, ip, created_at)`
- `alert_rules` / `alert_events`

### Deploy state machine (pseudo)
```
LOCK(site)
  prepare   -> verify disk, git remote, PHP version, maintenance mode (optional)
  fetch     -> git fetch --prune; git reset --hard origin/$BRANCH; git clean -fd
  build     -> composer install --no-dev; npm ci; npm run build
  link      -> symlink .env, storage, sqlite, other shared paths
  migrate   -> optional DB snapshot; php artisan migrate --force --isolated
  switch    -> ln -sfn releases/$ts current
  reload    -> systemctl reload php$VER-fpm
  verify    -> HTTP probe (3x, 2s apart) + optional smoke script
  activate  -> mark active; restart workers; prune old releases
  on_fail   -> ln -sfn previous current; reload fpm; mark failed; alert
UNLOCK
```
Key rule: **never mark `active` until `verify` passes.** The current code does the opposite.

### Testing strategy (this is what "production grade" means)
- Unit: vhost generators, env parser, state machines.
- **Shell contract tests:** render every Blade script and run `bash -n`; ShellCheck; BATS tests for idempotency (run twice, assert no duplicate users/sudoers/keys).
- **Golden-path E2E:** ephemeral Ubuntu 24 VM/container with sshd; provision from scratch; deploy a fixture Laravel app; assert HTTP 200; deploy v2; break v3; assert auto-rollback; run a curl loop throughout and assert zero 5xx.
- **Chaos:** kill worker mid-deploy, drop network, hold apt lock, fill disk, expire SSL, revoke SSH key. Assert resume/repair and actionable errors.
- **Upgrade tests:** N-1 → N with DB dump/restore and rollback.
- **Security:** authz matrix tests (project scoping), secret leakage scans, dependency audit, signed release verification.

---

## 9. Definition of "production grade" (acceptance criteria)

1. Fresh Hostinger Ubuntu 24 VPS → Laravel app live with SSL in **< 10 minutes, zero SSH**.
2. 100 consecutive deploys with **no OPcache/FPM degradation** and p95 deploy < 60 s.
3. Bad release (5xx) → **automatic rollback in < 30 s**, zero user-visible downtime.
4. Invalid nginx config → **rejected before reload**, no impact on other sites.
5. Kill the control plane at any provisioning step → **resume without duplicate resources**.
6. No destructive changes to an existing host; preflight blocks with a fix list.
7. Every shell step idempotent; `doctor --fix` repairs drift.
8. Alerts fire for down site, failed deploy, SSL < 14 days, disk > 85%, failed backup.
9. Control-plane upgrade with automatic DB backup and one-command rollback.
10. Audit trail for every privileged action.

---

## 10. Roadmap (1 senior full-stack dev)

### Phase 0 — Fork baseline (week 1)
- Fork at `4.1.1`, rebrand strings/assets, keep AGPL notices, add `UPSTREAM.md`.
- CI: Pint, PHPStan, Pest, ESLint, `bash -n`/ShellCheck, build.
- Reproducible production Compose (app, worker, scheduler) with healthchecks, no default creds, persistent volumes.
- `vito:doctor` command (PHP/DB/Redis/queue/ws/scheduler/storage permissions).
- Security quick wins: TLS 1.2+, no root key clobbering, idempotent `create-user`, targeted sudoers, installer no longer overwrites host nginx by default.
- Control-plane backup/restore + upgrade script with DB dump.

### Phase 1 — Deploy safety pack (weeks 2–4)
- Phase state machine + lock + cancellation + timeouts.
- FPM reload/OPcache reset; `nginx -t` gate; git reset; release pruning guard.
- Health gate + auto-rollback; fix false-success activation (#1202 class).
- Migration snapshot + warning; worker restart verification.
- Deploy timeline UI + notifications.
- E2E harness + chaos tests.

### Phase 2 — Hostinger + onboarding (weeks 4–6)
- Hostinger VPS provider (create, OS template, region, SSH key, firewall, poll).
- Hostinger DNS provider (A/AAAA/CNAME, snapshots, validation).
- Preflight doctor with actionable failures.
- One-screen Laravel deploy wizard + blueprints.
- Import existing Laravel app flow.

### Phase 3 — Provisioning reliability (weeks 6–8)
- `provisioning_steps` state machine, idempotent scripts, retries, apt lock wait.
- Resume + repair + drift reconciler.
- Idempotency tests for every step.
- Slim runtime profile (database queue, optional Redis/Horizon/ws).

### Phase 4 — Ops & polish (weeks 9–12)
- Uptime/SSL/disk/deploy alerts; alert routing.
- Audit log; fine-grained permissions (at least project/server scope).
- Backup verification + restore drill.
- Docs, migration guide from Vito, demo video, pricing/support model.
- Security review + pen-test pass.

### Ongoing
- Monthly upstream rebase; contribute generic fixes upstream first.
- Keep a `patches/` or `upstream/` branch to make rebases tractable.

---

## 11. Risks & mitigations

| Risk | Mitigation |
|---|---|
| **AGPL** forces source disclosure for network use | Accept open-core: sell hosting/support/branding, not closed features. Or clean-room rewrite if proprietary is non-negotiable |
| Upstream moves fast (4.0 June → 4.1.1 Sept) | Fork at tags, keep diffs modular, upstream generic fixes, monthly rebase, avoid UI rewrites |
| Managing root SSH = high blast radius | Threat model, targeted sudo, audit log, 2FA enforcement, signed releases, security review before beta |
| Combinatorial support matrix | Support **Ubuntu 24.04 + Hostinger + 2 providers** in v1; fail fast and clearly elsewhere |
| Scope creep into a full PaaS | Enforce non-goals; ship the 3 pillars only |
| Solo maintainer burnout | Narrow thesis, plugins for integrations, community PRs, paid support only after beta |

---

## 12. First 10 working days (concrete)

1. **Fork & baseline:** `gh repo fork vitodeploy/vito`, branch `hardened`, pin `4.1.1`, add `UPSTREAM.md` + rebase script. Rebrand only what is user-visible.
2. **CI green:** run Pint/PHPStan/Pest; add ShellCheck + `bash -n` over rendered scripts; add a Docker E2E skeleton.
3. **Installer isolation:** new `deploy/` compose (app/worker/scheduler), healthchecks, no host nginx writes; keep legacy installer behind `--legacy`.
4. **`vito:doctor`:** checks PHP extensions, DB, queue, scheduler heartbeat, ws, storage perms, APP_KEY, disk; JSON + human output.
5. **Provisioning step table + wrapper:** introduce the table and wrap the 5 existing install steps; no behavior change yet, just observability/resume scaffolding.
6. **Deploy safety PR #1:** `nginx -t` before reload + FPM reload after release + `git reset --hard origin/$BRANCH`.
7. **Deploy safety PR #2:** health gate + auto-rollback + never-activate-until-verified.
8. **E2E harness:** Ubuntu 24 container/VM, fixture Laravel app, deploy v1/v2, break v3, assert rollback + zero 5xx.
9. **Hostinger spike:** authenticate, list data centers/OS templates, create a throwaway VPS, destroy it; document API quirks.
10. **Write the v1 spec** (this document → tickets) and demo the safety pack on a real Hostinger VPS.

---

## 13. Decisions needed before code

1. **Strategy:** fork-and-harden (recommended), plugin-only, or greenfield?
2. **Scope:** Hostinger-only Laravel first, or provider-agnostic Laravel/PHP from day one?
3. **Commercial model:** self-hosted open source, paid hosted service, or open-core? (Determines AGPL handling and how much to invest in multi-tenancy/billing.)
4. **Control-plane target:** run on the same VPS as apps (must be non-destructive) or a separate small control VPS (can be heavier)?
5. **Team:** solo or more developers? (Sets the roadmap to 8 vs 16 weeks.)

---

## Appendix — Useful references

- `scripts/install.sh`, `scripts/update.sh` — control-plane lifecycle
- `app/Actions/Server/InstallServer.php` — linear provisioning
- `app/Actions/Webserver/*`, `app/Services/Webserver/Nginx.php` — vhost generation/reload
- `app/Jobs/Site/DeployJob.php`, `resources/views/ssh/modern-deployment/*` — deploy/rollback
- `resources/views/ssh/os/create-user.blade.php` — user/sudo/key setup
- `app/Console/Kernel.php` — scheduler load
- `config/queue.php` — hardcoded Redis queues
- `docs/4.x/sites/modern-deployment.md` — beta caveats
- Open issues: #1202, #1256, #1273, #1272, #1205, #1103, #1101, #1190, #1058, #1227, #1213–#1216, #863, #861
- Fixed security issues: #1267, #1218, #1231
- Hostinger API: <https://developers.hostinger.com/> (VPS: 67 endpoints; DNS: 8 endpoints; PHP SDK available)
