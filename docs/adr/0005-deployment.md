# ADR 0005 — Deployment on Hermes

- Status: Accepted (with open items)
- Date: 2026-08-25
- Decides: how calendar-wiz is hosted, exposed, and auto-updated on the `hermes-server` box.
- Builds on: [ADR 0001](./0001-aggregation-architecture.md) (one Ktor process + SQLite), [ADR 0004](./0004-booking-rules.md) (public booking pages need TLS).

## Context — Hermes, inspected read-only (2026-08-25)

`hermes-server`, public IP **5.161.43.242**, a small shared VPS. It already runs
**munich-housing-notifier**, which must not be disrupted.

- Docker **29.5.2**, **Swarm inactive**; the `deploy` user is in the `docker` group.
- Apps live at `/home/deploy/repos/<app>/` as **plain `docker compose`** projects, env in a host `.env`.
- munich **binds `0.0.0.0:80`** directly and carries a shared **`autoheal`** sidecar
  (`willfarrell/autoheal`) that restarts any container labelled `autoheal=true`.
- **Port 443 is free; port 80 is taken.** No reverse proxy on the box.
- SSH is reachable **only over WireGuard** (`10.0.3.1:22`).
- ⚠️ **RAM = 1.9 GB, no swap, ~726 MB free**; disk 38 GB at 48%. **RAM is the binding constraint.**

## Decision — right-sized, not rondia's stack

rondia's deploy (Docker **Swarm** + **Traefik** + `almir/webhook` + wildcard certs) is built for a
multi-service, multi-tenant Django app. calendar-wiz is **one Kotlin/Ktor process + one SQLite file**,
single user, low volume. Match the deploy to that, and stay off munich's toes.

1. **Plain Docker Compose, no Swarm.** App at `/home/deploy/repos/calendar-wiz/`, `restart: unless-stopped`.
2. **Build in CI, deploy from a registry — never build on the box.** A Gradle/Kotlin build would OOM
   a 1.9 GB box. GitHub Actions builds the image and pushes to `ghcr.io/pepearayao/calendar-wiz`;
   Hermes only **pulls**.
3. **Auto-deploy on merge via Watchtower** (Hermes pulls; no inbound webhook, no SSH key in CI — which
   fits SSH being WireGuard-only). Push to `main` → CI builds+pushes → Watchtower on Hermes sees the
   new tag and restarts calendar-wiz.
   - **Scope it so it can never touch munich:** `WATCHTOWER_LABEL_ENABLE=true` + the enable label
     (`com.centurylinklabs.watchtower.enable=true`) on **calendar-wiz only**.
   - **The image is private (decided).** A ghcr **read token** (classic PAT with `read:packages`, or
     a fine-grained token) must live at `/home/deploy/.docker/config.json` and be mounted into
     Watchtower; the same login lets the box pull manually. Without it Watchtower **401s silently and
     never updates** — the dangerous failure mode.
4. **TLS on 443 only, leaving port 80 to munich.** Add **Caddy** bound to 443, reverse-proxying the
   calendar-wiz host → the Ktor container.
   - Caddy grabs `:80` by default (redirects + HTTP-01). **Disable that** (`auto_https disable_redirects`
     or an explicit `http_port` override) and use **TLS-ALPN-01** on 443 — **or** **DNS-01** on the
     `pepearayao.com` zone (needs no inbound port; the safer option if DNS is on a supported provider).
   - **Verify against current Caddy docs before applying: a misconfig that grabs `:80` breaks munich.**
   - **Host = `book.pepearayao.com` (decided); challenge = TLS-ALPN-01 on 443.** Needs only an
     `A` record `book.pepearayao.com → 5.161.43.242` and inbound 443 (free). DNS-01 stays a fallback
     if inbound ACME on 443 ever fails.
5. **Memory discipline — the main risk.**
   - **Add a 2 GB swapfile on Hermes _before_ the first deploy.** There is currently zero swap.
   - JVM ceiling is an **absolute `-Xmx256m`**, *not* `MaxRAMPercentage` — percentage-of-total would
     hand the JVM ~950 MB against ~726 MB free and OOM-kill munich.
   - Caddy ~40 MB, Watchtower ~15 MB. Watch `docker stats` after first deploy. (A GraalVM **native
     image** would cut the app to ~50 MB RAM — deferred; it adds build complexity.)
6. **Self-healing by reuse.** Label calendar-wiz `autoheal=true` to reuse munich's existing autoheal
   sidecar (one sidecar watches the whole daemon by label). **Known coupling:** that sidecar lives in
   munich's compose project, so `docker compose down` on munich also stops healing calendar-wiz.
   Accept the reuse; documented here. (Alternative: give calendar-wiz its own ~15 MB autoheal.)
7. **Data + backups.** SQLite on a bind volume under `/home/deploy/repos/calendar-wiz/data`; a nightly
   `cron` copies the file **off-box**. No database server.
8. **Secrets.** Host `.env` at `/home/deploy/repos/calendar-wiz/.env` (matches munich), never in the
   repo — OAuth client secrets, the Proton share URL, SMTP/DKIM creds, the ghcr token.
9. **Keep tests-on-PR** (`ci.yml`).

## What this adds

- **calendar-wiz repo:** `app/Dockerfile` (slim JRE base), `docker-compose.yml` (app + Caddy +
  Watchtower), a `Caddyfile`, `.github/workflows/{ci.yml, deploy.yml}` (deploy.yml = build + push
  only), `.env.example`.
- **Hermes:** `/home/deploy/repos/calendar-wiz/` checkout, the host `.env`, the swapfile, ghcr auth,
  and a DNS record.

## Open items — resolve before the first deploy

- **Hostname — decided: `book.pepearayao.com`** (booking URLs `book.pepearayao.com/{handle}/{context}/{duration}`,
  the `/agenda` segment dropped). The user is adding the `A` record → 5.161.43.242.
- **ghcr image — decided: private.** Needs a `read:packages` token at `/home/deploy/.docker/config.json`,
  mounted into Watchtower (decision 3).
- **Swap** — a 2 GB swapfile on Hermes, still to be added before first deploy (decision 5).
- **Email / DKIM sending domain** for the iMIP invite writes (SPF/DKIM/DMARC) — still open; from the map fog.

## Consequences

- Merge → CI build (a few minutes) → Watchtower pull → live. Zero manual steps, exactly the ask.
- munich stays untouched: a different port (443 vs 80), a scoped Watchtower, and a reused-but-noted
  autoheal.
- It's one small box; **RAM is the ceiling** — the swapfile plus `-Xmx256m` are what keep both apps
  alive.
