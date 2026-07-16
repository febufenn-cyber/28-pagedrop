# PageDrop

Describe your business in a sentence or two, get a live landing page on its own subdomain in about 60 seconds — no builder, no drag-and-drop.

## The problem

Local businesses and freelancers need a landing page fast but don't want to learn a website builder or pay an agency for a single page. PageDrop turns a short description straight into a deployed page.

## Target buyer

Local businesses and freelancers (salons, tutors, consultants, small shops) who need one credible page online now, not a full site build.

## Pricing hypothesis

Rs299 per generated page (one-time), Rs99/month hosting after the page is live.

Status: planned — not yet built (50-SaaS challenge #28)

## Stack

Cloudflare Worker (TypeScript, ESM) for API + static frontend, single synchronous LLM call per page (no VPS needed — generation is fast enough for Cloudflare's request cap). Supabase for auth/Postgres/RLS. Deploy target for generated pages is Cloudflare Pages (Direct Upload API), not a custom sandbox.

## How to continue this build

Nothing is built yet. Read `docs/LLD.md` for the architecture and data model, then `docs/PLAN.md` for the ordered, TDD task list, and execute tasks in order. `CLAUDE.md` points at the private reference implementation for the shared Worker+Supabase+credits conventions this product reuses.

## Risks / constraints

- **Do not use bolt.diy or any WebContainers-based in-browser code execution** — WebContainers requires a commercial StackBlitz license for production/paid use. PageDrop instead generates a single self-contained HTML file (Tailwind via CDN `<script>` tag, no build step, no sandboxed code execution) directly from the LLM.
- **Deployment is via the Cloudflare Pages API** (Direct Upload, branch-aliased deployments), giving each generated page a `{slug}.<project>.pages.dev` URL — not a custom domain or wildcard-DNS setup in v1.
- Slugs must be unique per site; a collision blocks generation rather than silently overwriting an existing live page.
- This is a marketing page generator, not a legal or compliance tool — no claims-checking on business copy the LLM produces; the user is responsible for what they publish.
