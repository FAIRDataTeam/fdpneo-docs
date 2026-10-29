# Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| **"Server is unreachable" on save** | A CORS or origin mismatch, or a server error returned without CORS headers. Confirm the three origins agree (client public origin, `FDP_CORS_ALLOW_ORIGINS`, Keycloak redirect URIs; see [Configuration](../deployment/configuration.md)); check the server logs and `GET /fdp-api/readyz`. |
| **Login loops or "invalid redirect"** | The Keycloak client's redirect URIs do not include your client origin. Add it in the realm. In the evaluation stack, the client must be opened at exactly `http://localhost:5173`. |
| **Search returns nothing or errors** | PostgreSQL is down or the index is empty. Check PostgreSQL; run `fdp search reindex`. |
| **"no matching manifest for linux/arm64" when pulling** | `main` is an amd64-only development build. Set `IMAGE_TAG=latest` (or a release version) in `.env`; releases are multi-architecture. |
| **`postgres` exits with "old database(s) found" after an upgrade** | Server 0.19 moved to PostgreSQL 18. Run `upgrade-postgres-18.sh` from the stack directory (see [Local deployment](../deployment/local.md#upgrading-a-stack-from-before-server-019)). |
| **Every sign-in fails with "invalid bearer token" after an upgrade** | Keycloak 26 no longer emits the `fdp` audience from the old realm. Run `fix-keycloak-audience.sh` from the stack directory. |
| **Writes return `503` with `Retry-After`** | A full backup or restore is running over HTTP. Reads continue; retry when it finishes. |
| **Policies have no effect, every published record is public** | The deployment runs in open mode, the default. Switch *Access control* to *ODRL policies* under avatar, Settings, or set `FDP_ACCESS_CONTROL_MODE=odrl`. |
| **"policy denies modify" when creating a record** | In ODRL mode, the root access policy is not resolving. Ensure the profile is applied (`fdp profile info`) and `GET /fdp-api/policies/system-default` returns `200` with state `PUBLISHED`. |
| **A record you just wrote looks empty when fetched** | Drafts are not served anonymously. Fetch it with a bearer token, or publish it. |
| **Users page says "Couldn't load users"** (API answers `502`, log says the IdP admin call returned `401`) | The service-account token was minted at one Keycloak origin and the Admin API called at another. Point `FDP_IDP_ADMIN_BASE_URL` at the same origin as `FDP_OIDC_ISSUER` ([User management](../deployment/user-management.md)). |
| **Policy or license editor: Validate says the document "is not an odrl:Offer" (or has no title) and Publish does nothing** | The client built the document's subject IRI as `<base>/policies/<id>` while the server (0.16 and later) stores managed documents under `<base>/fdp-api/policies/<id>` and `<base>/fdp-api/licenses/<id>`. Client 0.6.6 and earlier have this mismatch. Upgrade the client to 0.7.0 or later. |
| **No "Users" entry in the menu** | The user-management façade is not configured. Set `FDP_IDP_ADMIN_CLIENT_ID` and `FDP_IDP_ADMIN_CLIENT_SECRET` ([User management](../deployment/user-management.md)). |
| **Per-record metrics are empty after moving to HTTPS** | Metrics recorded before server 0.17 were keyed by the internal URL. Re-key once with `fdp metrics rebase-resources --from http://<host>` ([Production deployment](../deployment/production.md)). |
| **Certificate errors or login broken on a new production host** | DNS does not yet point at the host, or ports 80/443 are blocked, so Caddy cannot obtain a certificate. Fix the DNS and firewall; `tls internal` only helps to test routing. |

For deployment-level checks, the server exposes `GET /fdp-api/healthz` (liveness), `GET /fdp-api/readyz` (readiness) and `GET /fdp-api/info` (version). Issues can be reported in the [server](https://github.com/FAIRDataTeam/fdpneo-server/issues) and [client](https://github.com/FAIRDataTeam/fdpneo-client/issues) repositories.
