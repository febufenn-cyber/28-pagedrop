# PageDrop — Build Plan

TDD, one task per focused session. Files touched, interfaces produced, test to write first, done-criteria.

## T1 — Repo scaffold + Supabase schema

Files: `package.json`, `tsconfig.json`, `wrangler.toml`, `supabase/migrations/0001_init.sql`, `.dev.vars.example`, `.gitignore`.
Interfaces: none (schema only) — `profiles`/`sites` tables, RLS, `spend_credit(p_user uuid)`, `refund_credit(p_user uuid)` exactly per LLD.
Done: `supabase db push` applies cleanly; `wrangler dev` boots.

## T2 — Supabase client + site types (`src/supa.ts`, `src/site.ts`)

Files: `src/supa.ts`, `src/site.ts`, `test/supa.test.ts`.
Interfaces: `export interface Site { id; userId; slug; status; liveUrl; hostingPaidUntil; ... }`; `insertSite(...)`, `getSite(id)`, `listSites(userId)`, `updateSite(id, patch)`, `slugExists(slug): Promise<boolean>`, `spendCredit(userId)`, `refundCredit(userId)`.
Test first: mock `fetch`; assert `slugExists` queries PostgREST with the right filter and `spendCredit` handles the insufficient-balance case.
Done: `vitest run test/supa.test.ts` green, no live network calls.

## T3 — Slug generation utility (`src/slug.ts`)

Files: `src/slug.ts`, `test/slug.test.ts`.
Interfaces: `slugify(businessName: string): string` (lowercase, hyphenated, DNS-label-safe, truncated to Cloudflare Pages' branch-alias length limit).
Test first: table-test inputs with spaces, punctuation, unicode, and overlength names; assert output is always a valid DNS label.
Done: unit tests green, including an edge case for a name that slugifies to an empty string (must fall back to a random suffix).

## T4 — HTML generation provider (`src/providers/anthropic.ts`, `src/providers/deepseek.ts`)

Files: `src/providers/anthropic.ts`, `src/providers/deepseek.ts`, `src/prompt/system.ts`, `test/providers/anthropic.test.ts`, `test/providers/deepseek.test.ts`.
Interfaces: `generateSite(opts: { description: string; apiKey: string; fetchFn?: typeof fetch }): Promise<{ title: string; html: string; metaDescription: string; model: string }>`, implemented identically for both providers matching the reference's provider function shape.
Test first: mock `fetch` with a canned JSON response; assert parsed shape; add a malformed-JSON case to exercise the repair fallback.
Done: unit tests green for both providers, including the repair-fallback path.

## T5 — Cloudflare Pages deploy client (`src/pages.ts`)

Files: `src/pages.ts`, `test/pages.test.ts`.
Interfaces: `deployToPages(opts: { slug: string; html: string; projectName: string; apiToken: string; accountId: string; fetchFn?: typeof fetch }): Promise<{ url: string; deploymentId: string }>` — POSTs a branch-aliased Direct Upload deployment to the Pages API.
Test first: mock `fetch`; assert multipart/form-data request shape and correct branch-to-URL mapping (`{slug}.{project}.pages.dev`).
Done: unit test green; a `PagesDeployError` case is tested for non-2xx responses.

## T6 — Route handlers (`src/handlers.ts`, `src/router.ts`)

Files: `src/handlers.ts`, `src/router.ts`, `test/router.test.ts`, `test/handlers.test.ts`.
Interfaces: `route(req, deps): Promise<Response|null>` with the LLD's route table; `handleCreateSite` (slug check → spend → generate → deploy → update, refund on any failure), `handleListSites`, `handleGetSite`, `handleRenewHosting`.
Test first: for `handleCreateSite`, mock a slug collision (assert 409, no credit spent) and a deploy failure after credit spend (assert refund called).
Done: `vitest run` green across router + handlers; every failure mode in the LLD's API table has a corresponding test.

## T7 — Worker entrypoint (`src/worker.ts`)

Files: `src/worker.ts`.
Interfaces: default export `{ fetch(req, env, ctx) }` wiring `route()` to real `env` bindings — no scheduled handler needed (no async sweep required for a fully synchronous product).
Done: `wrangler dev` serves `/config.js` and rejects unauthenticated `/api/sites` with 401.

## T8 — Frontend (`public/*.html`)

Files: `public/index.html`, `public/app.html`, `public/site.html`, `public/pricing.html`.
Interfaces: none (static, supabase-js CDN + fetch to `/api/*`).
Test first: none (manual/visual) — but the generate form must show a clear slug-availability check before submit.
Done: manual click-through against `wrangler dev` completes description → generate → live URL in one flow.

## T9 — Deploy + live smoke test + launch checklist

Files: `scripts/smoke.ts`, production `wrangler.toml` vars, secrets via `wrangler secret put`.
Interfaces: `scripts/smoke.ts` runs against the deployed Worker: creates a site with a unique test slug, asserts `status='deployed'` and that the returned `live_url` actually resolves (fetches it, checks 200).
Done: `wrangler deploy` succeeds; `npm run smoke` passes; launch checklist confirmed — pricing page shows Rs299/page + Rs99/mo, Cloudflare Pages project created and `CF_PAGES_API_TOKEN`/`CF_ACCOUNT_ID` set, `ANTHROPIC_API_KEY`/`DEEPSEEK_API_KEYS` and `SUPABASE_SERVICE_ROLE_KEY` set via `wrangler secret put`, no `.dev.vars` committed.
