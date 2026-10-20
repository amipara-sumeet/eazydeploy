# EazyDeploy — Product Docs

Working product/engineering documentation for EazyDeploy (fork of Vito 4.x, AGPL-3.0). Internal-first; no public release until owner sign-off.

| Doc | Purpose |
|---|---|
| [ROADMAP.md](./ROADMAP.md) | Phase-wise product and engineering plan (v0.3), scope, KPIs, risks, decisions |
| [VITO_FORK_ANALYSIS.md](./VITO_FORK_ANALYSIS.md) | Deep analysis of the Vito 4.x codebase, gaps, and original recommendations |

**Locked stack decisions:** Ubuntu 26.04 default · provider-agnostic · Laravel/PHP + Node + static React + Bun · PM2 single-process default (cluster opt-in) · nginx default, Caddy one click away (both first-class) · Postgres-only control plane · API + CLI in v1 · Laravel pack (Horizon/Octane/Pulse/Reverb) in v1 · Octane defaults to FrankenPHP with memory safeguards · calm, Vercel-like UI.

**Status:** planning complete (v0.3), all blocking decisions locked. No product code changes yet — awaiting owner go-ahead. Base pinned to Vito `4.1.1` line (`4.x` @ `8598e4dc`).

**Principles:** reliability over features · non-destructive automation · idempotent provisioning · measurable exit criteria · owner review gate each phase.
