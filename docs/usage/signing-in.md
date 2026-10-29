# Signing in and roles

Anyone can browse a FAIR Data Point without an account. Signing in unlocks what your role allows.

![The browse page before signing in](../assets/screenshots/01-browse.webp)

## Signing in

Click **Sign in** (or the avatar) in the top-right corner. You are redirected to the identity provider (Keycloak in the bundled stacks), authenticate there, and are sent back to the application. FDPneo never sees your password.

![The browse page after signing in](../assets/screenshots/02-browse-signed-in.webp)

In the evaluation stack, sign in as `admin` / `admin` (administrator and steward), `alice` / `alice` (steward) or `bob` / `bob` (no role). In a production deployment, use the account your administrator created.

## What each role can do

| Role | Can |
|---|---|
| **Anonymous** (not signed in) | Browse and search **published** records, run SPARQL queries, view metrics. |
| **Steward** | All of the above, plus create, edit and delete their own records, manage publication state, and use the "My data" dashboard. |
| **Admin** | Everything, plus manage SHACL schemas, resource types, ODRL policies, licenses, instance settings, users, and factory reset. |

In the default open mode these roles are all that matters. When the deployment runs in ODRL mode, what you may do with a specific record is decided by the ODRL policy it references; see [Access policies](policies.md).

## Your profile

Your identity, roles and a **Manage account** link to the identity provider are under **avatar, Profile**. Passwords and e-mail addresses are changed in the identity provider's account console, not in FDPneo.

![The profile page](../assets/screenshots/15-profile.webp)

## Account menu

Signed-in users find their pages in two places. The header tabs show **Browse** and **Search** to everyone, and **Schemas**, **Policies** and **Metrics** to administrators. The avatar menu holds **My metadata** (stewards), **Licenses**, **Resource types**, **Settings**, **Backup & Restore**, **Appearance** and **Users** (administrators, the last one only when the user-management façade is enabled), and **Profile** and **Access tokens** for everyone. The **About** page in the footer shows the server version, the applied profile and the enabled features.

![The about page](../assets/screenshots/17-about.webp)
