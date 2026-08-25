# STATUS — resume here

**Frozen 2026-08-25.** This is the single "pick it back up" file. Read it first, then
`README.md` → `docs/BUILD.md`.

## Stage: planning complete — paused before building

- The full design is **done and durable**: 5 ADRs + 4 research files + the view prototype, all in
  this repo. Planned with `/wayfinder` (map: https://github.com/pepearayao/calendar-wiz/issues/3).
- **No application code written yet.** Deployment is decided ([ADR 0005](docs/adr/0005-deployment.md))
  but not yet set up on the server.

## What's already decided — do not redo

Everything is in `docs/adr/` (indexed in `README.md`):

- **ADR 0001** unified store + two-way sync engine · Kotlin/Ktor · SQLite/SQLDelight.
- **ADR 0002** event data model (recurrence, timezones, dedup).
- **ADR 0003** busy-mirroring.
- **ADR 0004** booking rules · host **`book.pepearayao.com`** (`/{handle}/{context}/{duration}`).
- **ADR 0005** deployment · right-sized Compose + Caddy(443) + Watchtower on `hermes-server`
  (no Swarm; must not disturb `munich-housing-notifier`).

Research: `docs/research/` (existing tools, Proton, Google/Microsoft, email-invite auto-add).
Prototype: https://claude.ai/code/artifact/3751fb68-58d9-47ae-98c3-f5685191ae5e

## Next actions when you resume (in order)

1. *(optional, needs the accounts)* Run the two account checks — [#10 Proton reads](https://github.com/pepearayao/calendar-wiz/issues/10), [#11 Teams tenant](https://github.com/pepearayao/calendar-wiz/issues/11). They refine, they don't block.
2. **Scaffold the deploy files** (per ADR 0005): `app/Dockerfile`, `docker-compose.yml` (app + Caddy + Watchtower), `Caddyfile`, `.github/workflows/{ci,deploy}.yml`.
3. **Build Phase 0** (`docs/BUILD.md`): Ktor + SQLite/SQLDelight + `GoogleAdapter` (read) → unified view.
4. Then Phases 1–5 in `docs/BUILD.md`.

## Open items (owner)

- **DNS** — `A book.pepearayao.com → 5.161.43.242` — *user is adding it.*
- **ghcr image = private** — needs a `read:packages` token at `/home/deploy/.docker/config.json` — *at deploy time.*
- **2 GB swapfile on Hermes** — add before the first deploy (changes the box; confirm first).
- **Email/DKIM sending domain** — needed at **Phase 2** (the iMIP write channel).

## Resuming

- **With an agent:** point it at this file + `README.md`. Everything else follows from the docs.
- **The map:** `/wayfinder https://github.com/pepearayao/calendar-wiz/issues/3` — 9/11 tickets closed; open: #10, #11.

## Hermes reminders

- SSH is WireGuard-only. Bring the tunnel up first: `sudo wg-quick up ~/.wireguard/server.conf`, then `ssh hermes-agent`.
- `hermes-server` = **5.161.43.242** (public) / **10.0.3.1** over WireGuard. Small box: **1.9 GB RAM, no swap.**
- **Do not break `munich-housing-notifier`** — it owns port **80**; calendar-wiz uses **443**.
