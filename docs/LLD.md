# PageDrop — Low-Level Design

Adapts the shipped contract-reviewer's Worker+Supabase+credits architecture. Simpler than most siblings: a single synchronous LLM call plus a Cloudflare Pages API deploy — no VPS/async pattern needed.

## Architecture

```
Browser (static frontend, Workers Assets)
  │  POST /api/sites { business_description, slug }
  ▼
Cloudflare Worker (TypeScript, ESM)
  ├─ /config.js
  ├─ /api/sites          (POST) → verify JWT, spend_credit, generate HTML,
  │                                deploy to Cloudflare Pages, store result
  ├─ /api/sites          (GET)  → list user's sites
  ├─ /api/sites/:id      (GET)  → single site status + live URL
  └─ /api/sites/:id/renew-hosting (POST) → spend_credit, extend hosting_paid_until
  │
  ├─ Supabase (plain fetch, service-role key): GoTrue JWT verify, PostgREST
  │     for sites/profiles
  │
  ├─ LLM provider (switchable): anthropic (claude-sonnet-4-6) | deepseek (deepseek-chat)
  │     → single-file HTML+Tailwind-CDN string, no code execution/sandbox
  │
  └─ Cloudflare Pages API (Direct Upload, branch-aliased deployment)
       → https://{slug}.{pages_project}.pages.dev
```

Request flow: user submits a business description → Worker verifies JWT → checks slug uniqueness → `spend_credit(user_id)` → LLM generates a single self-contained HTML file (Tailwind via CDN script tag, no build step) → Worker POSTs the file to the Cloudflare Pages API as a branch deployment aliased to the slug → on success, site row updated `status='deployed'` with the live URL; on any failure (generation error, slug collision at deploy time, Pages API error), `refund_credit` and `status='failed'`.

Because generation is one LLM call and deployment is one API call — no ffmpeg, no chunking, no multi-minute subprocess — the whole request completes well under Cloudflare's ~100s cap. This is a **synchronous** route, unlike the async-VPS siblings; no `/__announce` or VPS callback plumbing is needed for v1.

## Data model

```sql
create table public.profiles (
  user_id uuid primary key references auth.users(id) on delete cascade,
  credits int not null default 1,
  created_at timestamptz not null default now()
);
alter table public.profiles enable row level security;
create policy "read own profile" on public.profiles for select using (auth.uid() = user_id);

create table public.sites (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  business_description text not null,
  slug text not null unique,
  status text not null default 'generating'
    check (status in ('generating','deployed','failed')),
  live_url text,
  pages_deployment_id text,
  hosting_paid_until timestamptz not null default (now() + interval '30 days'),
  model text,
  error text,
  created_at timestamptz not null default now()
);
alter table public.sites enable row level security;
create policy "read own sites" on public.sites for select using (auth.uid() = user_id);
create index sites_user_created on public.sites (user_id, created_at desc);
create unique index sites_slug_idx on public.sites (slug);

-- Reused verbatim from the reference (flat -1 spend, no parameterization needed —
-- generation and hosting-renewal are both billed as "1 credit", priced to the
-- user via credit-pack rupee amounts, not by changing the RPC):
create or replace function public.spend_credit(p_user uuid) returns boolean
language plpgsql security definer set search_path = public as $$
begin
  update profiles set credits = credits - 1 where user_id = p_user and credits > 0;
  return found;
end $$;

create or replace function public.refund_credit(p_user uuid) returns void
language plpgsql security definer set search_path = public as $$
begin
  update profiles set credits = credits + 1 where user_id = p_user;
end $$;
revoke execute on function public.spend_credit(uuid) from public, anon, authenticated;
revoke execute on function public.refund_credit(uuid) from public, anon, authenticated;
grant execute on function public.spend_credit(uuid), public.refund_credit(uuid) to service_role;
```

State machine: `generating → deployed | failed`. Hosting renewal doesn't change `status`; it only pushes `hosting_paid_until` forward and spends a credit.

## API routes

| Route | Method | Auth | Behavior | Failure modes |
|---|---|---|---|---|
| `/config.js` | GET | none | Public Supabase config | — |
| `/api/sites` | POST | JWT | Slug uniqueness check → spend credit → generate HTML → deploy to Pages → store result | 401 no JWT; 402 no credits; 409 slug taken; 500 generation/deploy error → refund + `status=failed` |
| `/api/sites` | GET | JWT | List caller's sites | 401 no JWT |
| `/api/sites/:id` | GET | JWT | Single site status/URL, RLS-scoped | 401 no JWT; 404 not found/not owner |
| `/api/sites/:id/renew-hosting` | POST | JWT | Spend 1 credit, `hosting_paid_until += 30 days` | 401 no JWT; 402 no credits; 404 not found/not owner |

## LLM strategy

Provider choice per the reference: **anthropic (`claude-sonnet-4-6`)** default, **deepseek (`deepseek-chat`)** fallback for cost. Single synchronous call — no chunking, no async dispatch.

Prompt strategy: system prompt instructs the model to output one complete, self-contained HTML document — inline `<style>` where needed, Tailwind loaded via the CDN `<script src="https://cdn.tailwindcss.com"></script>` tag (no build step, no bundler, no code execution required to render it), semantic sections (hero, offering, contact/CTA), and copy grounded only in the business description the user provided (no invented claims, credentials, or testimonials). Output forced to strict JSON wrapping the HTML string plus metadata, to keep parsing uniform with the other products in the fleet:

```json
{
  "title": "string",
  "html": "string",
  "meta_description": "string"
}
```

No-raw-quotes-in-JSON-strings rule applies (same as the reference) since the HTML itself contains many double quotes — the prompt must instruct the model to use single quotes for HTML attributes inside the generated markup, or the Worker must HTML-decode/escape carefully before storing; the repair fallback (cheap model re-ask) is the safety net for malformed JSON.

Cost per operation: a landing page (~500 input tokens, ~3-4k output tokens for a full HTML document) on `claude-sonnet-4-6` is roughly Rs4-6 at current API pricing; deepseek-chat is a fraction of that. At Rs299/page this leaves very large margin even before hosting revenue.

## Frontend pages

`index.html` (pitch + example gallery), `app.html` (description form + slug picker + site list), `site.html` (single site status, live URL, renew-hosting button), `pricing.html`.

## Error handling / credits flow

Credit is spent before generation begins (after the slug uniqueness check, so a doomed request never charges for a slug that was always going to collide) and refunded on any failure downstream — LLM error, malformed JSON that survives the repair fallback, or a Cloudflare Pages API error. `status='failed'` carries a user-facing `error` string. There is no stale-pending sweep needed since the request is synchronous and always resolves to `deployed` or `failed` within the same HTTP call.

## Integrations and launch gates

**Cloudflare Pages API access is required to launch** — a Pages project must exist (`pagedrop-sites` or similar) with an API token scoped to `Pages:Edit`; this is account configuration, not a third-party OAuth review, so it doesn't block launch beyond setup time. No other external integration (no payment gateway integration beyond manual UPI, no OAuth) blocks v1. Custom-domain mapping for hosted pages (via Cloudflare for SaaS custom hostnames) is explicitly a **post-v1** feature — v1 ships `pages.dev` branch-subdomain URLs only.

## Security notes

RLS restricts `sites`/`profiles` reads to the owner; all writes go through the Worker's service-role key. `spend_credit`/`refund_credit` are SECURITY DEFINER, revoked from `anon`/`authenticated`. Business-description input is length-capped and stripped of any HTML/script before being sent to the LLM as plain text (defense in depth, even though the LLM output — not the user input — is what gets rendered as a live page). The generated HTML itself is treated as trusted output only after it round-trips through the Worker's JSON schema validation; no user-supplied HTML is ever deployed verbatim. The Cloudflare Pages API token is a Worker secret, never exposed to the client.

## Out of scope for v1

Page editing/regeneration beyond a fresh generate; custom domains; multi-page sites; image generation/upload (text-only landing pages, using stock gradients/placeholder visuals in the generated CSS, not real photos); contact-form backend (mailto/tel links only); analytics; A/B variants; WebContainers-based live in-browser editing (explicitly avoided, see Risks).
