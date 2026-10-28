# Smart Children Stories (حكايات ذكية)

A free, bilingual (Arabic RTL / English LTR) story platform for children and their families, built on Cloudflare Pages, Pages Functions, and D1.

## Product guardrails

- All artwork is 2D only. No 3D anywhere.
- Private by default: libraries, worlds, galleries, and unpublished videos are never indexed and never reachable by other users.
- Phase 1 is free. Payments and public-library publishing stay behind disabled feature flags (`payments_enabled`, `public_library_publishing`).
- While payments are disabled each account may create at most 3 generations per UTC day. Failed jobs are not counted.
- Children are represented only by fictional characters and nicknames. We never ask for home address, school, phone number, or real photos.
- Free-resource-first: stories will be composed from a license-verified 2D asset library and code transitions before any paid generative call.

Stage 1 delivers the foundation only: authentication, sessions, quota accounting, feature flags, and the bilingual shell. Story generation itself is not implemented yet (`POST /api/generations` returns `501 feature_disabled`).

## Stack

| Layer | Choice |
| --- | --- |
| Hosting | Cloudflare Pages (built from GitHub) |
| API | Pages Functions (`functions/`), shared router in `src/server/` |
| Database | Cloudflare D1 (migrations in `migrations/`) |
| Storage (later) | Cloudflare R2 |
| Async (later) | Cloudflare Queues / Workflows |
| Bot protection | Cloudflare Turnstile (server-verified, off locally) |
| Client | Framework-free TypeScript SPA (Vite) |
| Tests | Vitest + `@cloudflare/vitest-plugin` (Workers runtime, real D1) |

## Prerequisites

- Node.js 20+ (developed on Node 24 LTS) and npm.
- No Cloudflare account is required for local development.

## Local setup

```bash
npm install
npm run db:migrate:local   # applies migrations to .wrangler/state
npm run dev                # builds, then serves http://127.0.0.1:8788
```

`npm run dev` builds the client into `dist/` and runs `wrangler pages dev dist`, which serves both the static shell and the Pages Functions.

Optional environment overrides: copy `.dev.vars.example` to `.dev.vars`. Locally Turnstile runs in `off` mode and `ENVIRONMENT=development` (see `wrangler.toml`).

For a faster client loop while the Pages server is already running on port 8788:

```bash
npm run dev:web            # Vite on :5173, proxies /api to :8788
```

## Commands

| Command | Purpose |
| --- | --- |
| `npm run dev` | Build and run the full local stack on :8788 |
| `npm run dev:web` | Vite dev server for the client only |
| `npm run build` | Production build into `dist/` |
| `npm run typecheck` | TypeScript for client and server configs |
| `npm test` | Vitest suite (Workers runtime + D1) |
| `npm run db:migrate:local` | Apply D1 migrations locally |
| `npm run db:migrate:remote` | Apply D1 migrations to the remote database |
| `npm run deploy` | Build and deploy with `wrangler pages deploy` |

## Configuration

`wrangler.toml` holds non-secret variables and the D1 binding:

- `ENVIRONMENT` — `production` enables strict secret checks.
- `TURNSTILE_MODE` — `off` (local) or `enforced` (all auth requests must include a valid Turnstile token).
- `PASSWORD_ITERATIONS` — PBKDF2-SHA256 iterations, clamped to 1,000–1,000,000, default 100,000.
- `PRIVACY_SALT` — salt for hashing IPs and rate-limit identifiers. Required in production.

Secrets (`TURNSTILE_SECRET`, `TURNSTILE_SITE_KEY`, `PRIVACY_SALT`) must be set as Pages secrets and must never be committed. `.dev.vars` is gitignored.

The `database_id` in `wrangler.toml` is a stable local development UUID. Replace it with the real D1 ID before any remote command.

## Project layout

```
functions/            Pages Functions entrypoints and shell middleware
migrations/           D1 schema and seed migrations
public/               Static assets, _routes.json, _headers, robots.txt
src/client/           SPA views, router, API client, i18n, components
src/server/           Router, auth, sessions, CSRF, quota, flags, validation
src/shared/           Types and bilingual message catalog
test/                 Vitest suites and helpers
docs/                 Stage reports
```

## Tests

```bash
npm run typecheck
npm test
```

The suite runs against a real D1 instance via the Workers Vitest plugin and applies migrations automatically. It covers security headers, authentication, cookie handling, CSRF and origin checks, rate limiting, quota accounting (including refunds and concurrency), feature-flag gating, and Arabic/English message parity.

## Deployment handoff (not performed in Stage 1)

Deployment was intentionally deferred. When credentials and a repository are available:

1. `wrangler d1 create storyproject` and put the returned ID into `wrangler.toml`.
2. `npm run db:migrate:remote`.
3. Create a Pages project connected to the GitHub repository.
   - Build command: `npm run build`
   - Build output directory: `dist`
4. Add secrets: `TURNSTILE_SECRET`, `TURNSTILE_SITE_KEY`, `PRIVACY_SALT`.
5. Set `ENVIRONMENT=production` and `TURNSTILE_MODE=enforced` for the production environment.
6. Review `public/_headers`, `public/_routes.json`, and `public/robots.txt` after the first deploy.

## Known limits

See `docs/stage-1-report.md`. In short: no story generation, no email verification or password reset, no R2/Queues yet, and PBKDF2 cost may exceed the Workers free-tier CPU budget until it is tuned or the project moves to a paid plan.
