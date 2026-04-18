# Admin/API Split Deployment

This project now supports role-based deployment so you can split the admin site and the public API without introducing a separate VPS.

## Goal

- Keep the admin/dashboard on `https://cpa.boy00720.qzz.io`
- Move API traffic to `https://api.boy00720.qzz.io`
- Run two independent app services
- Reuse the same PostgreSQL and Redis instances

## Roles

Set `APP_ROLE` on each deployment:

- `APP_ROLE=admin`
  - Serves the embedded frontend
  - Serves `/api/v1/*` admin/user/auth/payment endpoints
  - Does **not** expose `/v1/*`, `/v1beta/*`, `/responses`, `/chat/completions`
- `APP_ROLE=api`
  - Serves `/v1/*`, `/v1beta/*`, `/responses`, `/chat/completions`
  - Does **not** serve the embedded frontend or `/api/v1/*`
- `APP_ROLE=all`
  - Backward-compatible single-service mode

## Background workers

`admin` (and `all`) keeps the control-plane workers such as:

- token refresh
- dashboard aggregation
- usage cleanup
- scheduled test runner
- backup scheduler
- ops aggregation / cleanup / scheduled reports

`api` keeps the gateway-facing runtime workers such as:

- scheduler snapshot refresh
- concurrency stale-slot cleanup
- user message queue cleanup

This avoids running backup/test/token-refresh jobs on the API-only service while still keeping request-path caches warm on the API service.

## Cloudflare / ClawCloud setup

1. Keep `cpa.boy00720.qzz.io` bound to the admin service.
2. Create `api.boy00720.qzz.io` and bind it to the API service.
3. On the API service, allow the admin origin in `cors.allowed_origins`, for example:

```yaml
cors:
  allowed_origins:
    - "https://cpa.boy00720.qzz.io"
  allow_credentials: true
```

4. Keep `server.frontend_url` pointing at the admin domain so emails and redirects still land on the admin site.

## CC Switch cutover

After the API service is live:

- Base URL: `https://api.boy00720.qzz.io`
- Usage URL: `https://api.boy00720.qzz.io/v1/usage`

## Verification checklist

- `https://cpa.boy00720.qzz.io/login` loads
- `https://cpa.boy00720.qzz.io/dashboard` loads
- `https://api.boy00720.qzz.io/v1/models` responds
- `https://api.boy00720.qzz.io/v1/usage` responds with a valid API key
- `https://cpa.boy00720.qzz.io/v1/models` returns `404`
- `https://api.boy00720.qzz.io/login` returns `404`
