# Settings, metrics and maintenance

## Instance settings

Open **avatar, Settings** as an administrator.

![The settings page](../assets/screenshots/13-settings.webp)

Settings are per-key configuration values the server exposes: branding texts, search-filter configuration, form autocomplete sources, the access-control mode, and others. Keys with a structured form can also be edited as JSON, and every value is validated server-side before it is stored.

**Access control** chooses between *Open access*, the default, where everyone can read every published record and only the writer roles may create, edit or delete, and *ODRL policies*, where each record's access policy decides. The writer roles are edited here too. *Reset to default* returns to the value set by `FDP_ACCESS_CONTROL_MODE`. See [Access policies](policies.md#access-control-mode).

**Updates** shows the running and latest server and web-client versions, the last check and a *Check now* button. With the optional host updater connected, it also offers *Update to X* and lists recent update requests. **Update check** switches the daily release check off. See [Updates](../deployment/updates.md).

Look and feel (colours, logo, favicon, organisation name) is configured on the client container instead, authored with the **Appearance** editor at `/appearance`; see [Configuration](../deployment/configuration.md).

## Metrics dashboard

Open the **Metrics** tab in the header. Stewards see metrics for the resources they own; administrators see the whole instance.

![The metrics dashboard](../assets/screenshots/14-metrics.webp)

The dashboard shows a summary, time series, top resources and, when the server has a GeoIP database configured, the geography of visitors. The underlying metrics are anonymous by design: identifying data is dropped when a request is recorded, visitors are counted with a daily-rotated hash, query text is never stored and no analytics cookies are set. The dashboard can therefore never answer "what did user X do", only "how is this instance used".

## Access tokens

Open **avatar, Access tokens** as any signed-in user.

![The access-tokens page](../assets/screenshots/16-tokens.webp)

Create a personal access token to call the API from scripts and tools, and revoke it at any time. Copy the token when it is created: the server stores only a hash of it. See [API usage](../api/index.md). Administrators can also mint an *Updater token*, which carries only the updater role and works only for the optional host updater.

## Backup and restore

Open **avatar, Backup & Restore** as an administrator to back up the records or the whole deployment, and to restore an archive. See [Backup and restore](../deployment/backup-and-restore.md).

## Factory reset

An administrator can **factory reset** the instance from the settings area (or with `POST /fdp-api/admin/reset`). This re-applies the deployment profile and restores schemas and policies to the profile baseline. It is destructive; back up first (see [Operations](../deployment/operations.md)).
