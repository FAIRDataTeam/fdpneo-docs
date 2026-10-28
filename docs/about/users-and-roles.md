# Users and roles

FDPneo distinguishes four functional roles. A single person can occupy several at once, and the system maps identity-provider claims onto FDPneo authorization decisions through ODRL constraints rather than through a fixed role table.

## Functional roles

**Anonymous users** are unauthenticated consumers. They receive a request context with no identity and no roles, and the policy evaluator treats them as a single class. Anonymous access is read-only and sees only published records.

**Data consumers** browse, search and query the metadata, and download or query the underlying data distributions where policy permits. Authenticated consumers may have access to records that are not publicly visible.

**Data stewards** curate metadata. They create and edit records, manage their publication state, and may be scoped to specific containers: a steward responsible for one catalog need not have write access to others. The scoping is expressed through ODRL constraints on the relevant Offers.

**Administrators** configure the deployment. In the application they manage SHACL schemas, resource types, ODRL policies, licenses, instance settings and users. At the deployment level they choose the triple store, configure the OIDC provider, apply deployment profiles and tune system parameters.

## Realm roles in the identity provider

The bundled Keycloak realm defines two realm roles that the server reads from the access token (configured through `FDP_OIDC_ROLES_CLAIM`, by default `realm_access.roles`):

| Realm role | Grants in the application |
|---|---|
| `steward` | Create, edit and delete own records, manage their publication state, use the "My data" dashboard. |
| `admin` | Everything a steward can do, plus schemas, resource types, ODRL policies, licenses, instance settings, users and factory reset. |

What a signed-in user can do with a given record is ultimately decided by the ODRL policy the record references. The system default Offer (`system-default`) grants public read and steward modify, which is the baseline until a deployment authors its own policies.

## Development users

The evaluation stack imports an `fdp-dev` realm with three users whose passwords equal their usernames. They are development-only and publicly known.

| User | Password | Realm roles |
|---|---|---|
| `admin` | `admin` | `admin`, `steward` |
| `alice` | `alice` | `steward` |
| `bob` | `bob` | none (equivalent to anonymous) |

The production stack imports a realm template with no demo users and a single bootstrap administrator, `fdp-admin`, whose temporary password is set in the deployment's `.env` and must be changed at first login.

## Where identity lives

FDPneo stores no passwords and issues no tokens of its own. Passwords, e-mail addresses and multi-factor settings are managed in the identity provider; the client links to the provider's account console from the Profile page. See [Signing in and roles](../usage/signing-in.md) for the user-facing view and [User management](../deployment/user-management.md) for the optional admin facade.
