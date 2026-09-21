# Bir Söz landing and public API gateway

Production site: https://www.birsoz.kz (birsoz.kz redirects to www).
Vercel project: `birsoz`, team `lukivan8s-projects`.

`vercel.json` adds external rewrites for the explicitly listed public API routes:

Browser → www.birsoz.kz (Vercel) → api.lukivan8.com (Cloudflare) → Tunnel → Perry.

These are server-side rewrites, not browser redirects. The original API origin,
DNS, tunnel, payloads and URLs remain available to existing extensions. The site
HTML and existing downloads are unchanged. The authenticated dashboard, its nested assets/forms/export routes, login/logout and privacy page are also proxied. Administrative `/api/` routes remain available only at the original API origin.

All `/api/*`, dashboard, login/logout and privacy responses bypass Vercel rewrite caching, with no-store headers.
Organization-specific catalog requests must retain X-Bir-Organization-Code;
conditional catalog requests must retain If-None-Match and the upstream ETag.

## Dashboard

Open https://www.birsoz.kz/dashboard. The backend requires the existing team
account and accepts the exact https://www.birsoz.kz Origin alongside its original
origin. Requests with missing/untrusted Origin are still rejected for writes.
The proxy preserves relative redirects, cookies, form bodies and HTMX headers.
Host-only Secure/HttpOnly/SameSite=Strict cookies keep each domain's login separate.
The original https://api.lukivan8.com/dashboard remains available.

Before reverting the backend origin allowlist, roll back these dashboard rewrites
to avoid presenting a login form that cannot submit through the new domain.

## Future extension integration

Candidate origins are `https://api.lukivan8.com` and `https://www.birsoz.kz`.
Probe `GET /api/health` with credentials omitted, cache disabled and a bounded
request timeout. Accept only HTTP 200 JSON exactly identifying
`{"ok":true,"service":"bir-soz-api","protocolVersion":1}`; a successful HTML
block page is not a healthy API. The probe itself must reach Perry.

Health measures reachability, not authorization or database readiness. Handle
actual request failures too. Preserve event IDs when retrying analytics (backend
deduplicates). Do not automatically replay non-idempotent requests without
checking their semantics. Add both origins to extension host_permissions when
implementing selection; no installed extension is switched by this deployment.

Both paths share the backend and Tunnel. This protects against client-network
reachability problems, not backend or Cloudflare-wide outages.

## Deployment and rollback

Vercel builds Git branches as previews and main as production. Validate preview
health, catalog/ETag, malformed POST validation and preflight before merging to
main. Keep the existing site and downloaded artifacts byte-identical.

Previous production before the gateway:
`dpl_HKALwguT9cqFN9aE3eDsXqG4FJkF` / commit `e9ab7f78b07db8560c70965265a169697e6eadf7`.

If needed, use Vercel Instant Rollback to that deployment, then revert the gateway
commit in Git to keep later deployments consistent. Old extensions keep using
the original API during either rollout or rollback.
