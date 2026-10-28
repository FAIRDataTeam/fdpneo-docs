# Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| **"Server is unreachable" on save** | A CORS or origin mismatch, or a server error returned without CORS headers. Confirm the three origins agree (client public origin, `FDP_CORS_ALLOW_ORIGINS`, Keycloak redirect URIs; see [Configuration](../deployment/configuration.md)); check the server logs and `GET /fdp-api/readyz`. |
| **Login loops or "invalid redirect"** | The Keycloak client's redirect URIs do not include your client origin. Add it in the realm. In the evaluation stack, the client must be opened at exactly `http://localhost:5173`. |
| **Search returns nothing or errors** | PostgreSQL is down or the index is empty. Check PostgreSQL; run `fdp search reindex`. |
| **"policy denies modify" when creating a record** | The root access policy is not resolving. Ensure the profile is applied (`fdp profile info`) and `GET /fdp-api/policies/system-default` returns `200` with state `PUBLISHED`. |
| **A record you just wrote looks empty when fetched** | Drafts are not served anonymously. Fetch it with a bearer token, or publish it. |
| **No "Users" entry in the menu** | The user-management façade is not configured. Set `FDP_IDP_ADMIN_CLIENT_ID` and `FDP_IDP_ADMIN_CLIENT_SECRET` ([User management](../deployment/user-management.md)). |
| **Per-record metrics are empty after moving to HTTPS** | Metrics recorded before server 0.17 were keyed by the internal URL. Re-key once with `fdp metrics rebase-resources --from http://<host>` ([Production deployment](../deployment/production.md)). |
| **Certificate errors or login broken on a new production host** | DNS does not yet point at the host, or ports 80/443 are blocked, so Caddy cannot obtain a certificate. Fix the DNS and firewall; `tls internal` only helps to test routing. |

For deployment-level checks, the server exposes `GET /fdp-api/healthz` (liveness), `GET /fdp-api/readyz` (readiness) and `GET /fdp-api/info` (version). Issues can be reported in the [server](https://github.com/FAIRDataTeam/fdpneo-server/issues) and [client](https://github.com/FAIRDataTeam/fdpneo-client/issues) repositories.
