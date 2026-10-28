# Configuration

FDPneo is configured through environment variables. The server reads them (or a `.env` file) through pydantic-settings; the client container reads them at start-up and writes its `config.js`. The compose stacks set the important ones for you; this page is the reference for when you need to go beyond them.

## Server

Copy `.env.example` from the server repository and adjust. Settings are grouped by prefix.

| Prefix | Purpose | Key variables |
|---|---|---|
| (top level) | Core | `ENVIRONMENT` (`development`, `staging`, `production`), `BASE_URL` (the public URL that mints resource IRIs), `POSTGRES_DSN`, `FDP_NAMESPACE`, `EXPOSE_API_DOCS` |
| (top level) | Persistent identifiers | `IDENTIFIER_BASE` (persistent namespace records are minted under, for example a W3ID prefix, decoupled from `BASE_URL`), `FDP_PID_*` for W3ID automation |
| `FDP_TRIPLESTORE_` | RDF store | `QUERY_ENDPOINT`, `UPDATE_ENDPOINT`, `GRAPH_STORE_ENDPOINT`, `USERNAME`, `PASSWORD`, capability flags |
| `FDP_OIDC_` | Authentication | `ISSUER`, `AUDIENCE`, `ROLES_CLAIM`, `GROUPS_CLAIM`, `JWKS_CACHE_TTL_SECONDS` |
| `FDP_CORS_` | Browser access | `ALLOW_ORIGINS` (comma-separated or JSON array), `ALLOW_CREDENTIALS` |
| `FDP_IDP_ADMIN_` | User management | `CLIENT_ID`, `CLIENT_SECRET` (enables `/users`), `BASE_URL`, `REALM` |
| `FDP_METRICS_` | Metrics pipeline | `ENABLED`, `UNIQUE_VISITOR_COUNTING`, `GEOIP_DATABASE_PATH`, roll-up cadence, `ROLLUP_IN_PROCESS` |
| `FDP_SEARCH_` | Search index | indexing options |
| `FDP_API_KEYS_` | Personal access tokens | token policy |
| `FDP_SCHEMA_SYNC_` | Schema synchronisation | refresh of remote-sourced schemas |
| `FDP_DATA_` | Simple data provider | `DOWNLOAD_MODE` (`redirect` or `stream`), proxy timeout and size limit |
| `FDP_PROFILE_` | Deployment profile | `AUTO_APPLY`, `PATH` |
| `FDP_INDEX_PING_` | FDP Index announcement | `TARGETS`, `INTERVAL_SECONDS` |
| `FDP_REMOTE_LABELS_` | External label resolution | `ENABLED`, `ALLOWED_HOSTS` |
| `FDP_RATELIMIT_` | Rate limiting behind a proxy | `TRUST_FORWARDED_FOR` |

After editing the configuration, restart the server. After upgrading, also run `fdp db migrate` (the container entrypoint does this automatically).

### Triple store

Any SPARQL 1.1 Protocol endpoint works. For GraphDB:

| Variable | Example | Notes |
|---|---|---|
| `FDP_TRIPLESTORE_QUERY_ENDPOINT` | `http://localhost:7200/repositories/fdp` | SPARQL query endpoint (required) |
| `FDP_TRIPLESTORE_UPDATE_ENDPOINT` | `http://localhost:7200/repositories/fdp/statements` | SPARQL update endpoint (required) |
| `FDP_TRIPLESTORE_GRAPH_STORE_ENDPOINT` | `http://localhost:7200/repositories/fdp/rdf-graphs/service` | Graph Store Protocol endpoint (optional, used for fast graph ingest and replace) |
| `FDP_TRIPLESTORE_USERNAME` / `_PASSWORD` | | Set when the store requires authentication (enable in production) |
| `FDP_TRIPLESTORE_SUPPORTS_REPOSITORY_MANAGEMENT` | `true` | GraphDB-only capability flag |
| `FDP_TRIPLESTORE_SUPPORTS_NAMED_GRAPH_SYNC` | `true` | Capability flag |

### PostgreSQL

| Variable | Example |
|---|---|
| `POSTGRES_DSN` | `postgresql+asyncpg://fdp:fdp@localhost:5432/fdp` |

PostgreSQL holds the metrics pipeline, the full-text search index, the authorization cache, the audit log, saved SPARQL queries and API-key hashes. If it is down, search and the steward dashboard fail even though browsing still works.

### OIDC

| Variable | Example | Notes |
|---|---|---|
| `FDP_OIDC_ISSUER` | `http://localhost:8080/realms/fdp-dev` | The realm issuer URL (required) |
| `FDP_OIDC_AUDIENCE` | `fdp` | Expected `aud` claim (required) |
| `FDP_OIDC_ROLES_CLAIM` | `realm_access.roles` | Dot-path to the roles array in the token |
| `FDP_OIDC_GROUPS_CLAIM` | `groups` | Dot-path to the groups array |

The bundled Keycloak realms define the realm roles `admin` and `steward`, a public `fdp-client` client (Authorization Code with PKCE), a confidential `fdp-server` client with a service account for the user-management facade, and an audience mapper that stamps `fdp` into the access token. Any other OIDC provider can be used if it is configured equivalently.

### Running the server process in production

The development command is `uv run fastapi dev`. The container image runs uvicorn; if you run the process yourself, the ASGI application is `fdpneo_server.main:app`:

```bash
uv run uvicorn fdpneo_server.main:app --host 0.0.0.0 --port 8000 --workers 4
```

Put it behind a TLS-terminating reverse proxy and set `FORWARDED_ALLOW_IPS` to the proxy's address so that `X-Forwarded-Proto` and `X-Forwarded-For` are honoured.

## Client

The client is a static single-page application. Build-time configuration is baked in from `.env` (`VITE_*`):

- `VITE_FDP_API_URL`, the server's public URL.
- `VITE_PUBLIC_ORIGIN`, the origin the application is served at.
- `VITE_OIDC_AUTHORITY`, `VITE_OIDC_CLIENT_ID`, `VITE_OIDC_ROLES_CLAIM`, fallbacks; the live values come from the server's `GET /fdp-api/config` at start-up.

```bash
npm install
npm run build        # dist/ (static files)
npm run preview      # optional: preview the production build locally
```

Serve `dist/` from any static host configured for single-page-application history fallback (rewrite unknown paths to `index.html`).

### Container image and runtime configuration

The client also ships as an nginx image whose entrypoint writes `config.js` and the Content Security Policy from environment variables at start-up, so one image serves any deployment without a rebuild.

| Variable | Purpose |
|---|---|
| `FDP_API_URL` | Absolute API origin (default `/`). Also added to the CSP `connect-src`. |
| `FDP_PUBLIC_ORIGIN` | The origin this application is served at (OIDC redirect base). |
| `FDP_IDP_ORIGIN` | OIDC provider origin, added to the CSP `connect-src` (optional). |
| `FDP_BRANDING` | Look and feel (white-labelling): JSON with the colour palette, logo, favicon and organisation name. Validated at start-up. |
| `FDP_BRANDING_ORG_NAME`, `FDP_BRANDING_LOGO_URL`, `FDP_BRANDING_LOGO_URL_DARK`, `FDP_BRANDING_FAVICON_URL`, `FDP_BRANDING_FAVICON_URL_DARK` | Single-string identity overrides; each overrides the matching key in `FDP_BRANDING`. |
| `FDP_BRANDING_IMG_ORIGIN` | Extra origins for the CSP `img-src`, only if the logo or favicon is hosted off-origin. |

Branding mistakes cannot break the client. Invalid JSON in `FDP_BRANDING` is logged to the container logs and the client boots with the default look; semantic mistakes (unknown keys, values that are not CSS colours) are explained in the browser console and shown live in the Appearance editor.

The easiest way to author `FDP_BRANDING` is the in-app **Appearance** editor at `/appearance` (no sign-in required; it is a client-side preview and export tool): adjust colours, logo and favicon with live preview, switch the export to Docker env, and copy the generated line into your compose file.

```yaml
services:
  client:
    image: ghcr.io/fairdatateam/fdpneo-client:main
    environment:
      FDP_API_URL: https://api.fdp.example
      FDP_PUBLIC_ORIGIN: https://fdp.example
      FDP_IDP_ORIGIN: https://idp.example
      FDP_BRANDING: '{"orgName":"Acme Data","logoUrl":"/branding/logo.svg","theme":{"--accent":"#7a1f2b","--signal":"#0d6e6e"}}'
```

Look and feel is a client concern; metadata schemas are a server concern. Branding only authors the client's own `config.js`. Custom schemas are durable server state seeded through the deployment profile (`fdp profile apply`, selected with `FDP_PROFILE_*`) and managed afterwards in the application.

## The origin coupling

When client, API and identity provider are on different origins, three settings must agree or logins and writes fail: `VITE_PUBLIC_ORIGIN` / `FDP_PUBLIC_ORIGIN` on the client, `FDP_CORS_ALLOW_ORIGINS` on the server, and the redirect URIs of the Keycloak client. The single-domain production stack avoids the issue by putting all three behind one origin.
