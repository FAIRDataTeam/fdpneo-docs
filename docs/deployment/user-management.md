# User management

FDPneo has no user database of its own. Users, passwords, e-mail addresses and realm roles live in the identity provider. The administrator **Users** screen in the client (invite users, assign roles, enable or disable, delete) is therefore a façade over the identity provider's admin API, implemented on the server as the `/fdp-api/users` endpoints (ADR-0013). It is **off by default** and only works with Keycloak.

## Enabling the façade

The server needs a confidential client with a service account that may call the Keycloak Admin API. Set both:

```env
FDP_IDP_ADMIN_CLIENT_ID=fdp-server
FDP_IDP_ADMIN_CLIENT_SECRET=<the client's secret>
# FDP_IDP_ADMIN_BASE_URL and FDP_IDP_ADMIN_REALM are derived from FDP_OIDC_ISSUER when unset
```

When configured, `GET /fdp-api/config` reports `features.user_management: true`, the `/users` endpoints come alive, and the client shows the Users entry in the account menu. If the credentials are unset, the façade returns `503` and the client hides the feature.

Both bundled stacks set this up.

!!! note "Keycloak in development mode and the Admin API"
    In `start-dev` mode Keycloak derives the token issuer from the host it is called on. The server mints the service-account token at the public issuer (`http://localhost:8081/realms/fdp-dev`), so the Admin API must be called at that same origin: `FDP_IDP_ADMIN_BASE_URL` has to point at the public Keycloak URL, not at the internal `http://keycloak:8080`, or every façade call fails with `502` and "IdP admin GET /users returned HTTP 401". The evaluation compose file was corrected accordingly in October 2026. The evaluation stack uses the `fdp-server` client of the `fdp-dev` realm with the development secret `fdp-server-dev-secret`; the production stack imports the realm and hands the server the same `FDP_SERVER_CLIENT_SECRET` from `.env`, so the two sides stay in sync.

The service account is least-privilege: `view-users`, `query-users`, `manage-users` and `view-realm`. When you create your own realm, create an equivalent confidential client, grant it these realm-management roles and supply a strong secret.

## Managing users without the façade

Users can always be managed directly in the Keycloak admin console: create the user in the realm, set credentials, and assign the `admin` or `steward` realm role. Users with no realm role can sign in but have the same rights as an anonymous visitor.

## Guard rails

The façade prevents administrators from locking themselves out: you cannot delete yourself, remove your own `admin` role or disable your own account. Invited users receive an e-mail from the identity provider to set a password and verify their address; no password ever passes through FDPneo.

See [Users](../usage/users.md) for the administrator's view of the screen.
