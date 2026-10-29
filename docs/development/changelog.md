# Changelog

Each repository keeps its own changelog in the [Keep a Changelog](https://keepachangelog.com/) format, and releases are tagged independently. This page points at them and summarises the versions this documentation was written against.

| Component | Current version | Changelog |
|---|---|---|
| Server | 0.21.1 (2026-10-29) | [`fdpneo-server/CHANGELOG.md`](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/CHANGELOG.md) |
| Client | 0.9.2 (2026-10-29) | [`fdpneo-client/CHANGELOG.md`](https://github.com/FAIRDataTeam/fdpneo-client/blob/main/CHANGELOG.md) |
| MCP bridge | 0.3.1 (2026-09-02) | [`mcp/CHANGELOG.md`](https://github.com/FAIRDataTeam/mcp/blob/main/CHANGELOG.md) |

## Server 0.18 to 0.21

**0.21.1.** The first signed, multi-architecture release images (`linux/amd64` and `linux/arm64`). Apple Silicon and ARM servers no longer need to build locally. The host updater verifies release signatures before it changes anything. See [Updates](../deployment/updates.md).

**0.20.0.** Update notifications: a daily check for newer server and client releases, shown to administrators (ADR-0028). Version-tagged releases with upgrade metadata, and per-component image tags (`SERVER_IMAGE_TAG`, `CLIENT_IMAGE_TAG`). Full-deployment backup and restore, encrypted with a passphrase, from the command line, over HTTP and with `backup.sh` and `restore.sh` for moving a bundled stack (ADR-0029, see [Backup and restore](../deployment/backup-and-restore.md)). The optional host updater (ADR-0030). The evaluation stack keeps Keycloak's data in a `keycloak-data` volume. Migrations `0011` and `0012` run automatically.

**0.19.0.** Current service versions in the bundled stacks: PostgreSQL 18, Keycloak 26.8, GraphDB 10.8.14, Caddy 2.11. The server image runs Python 3.15. **Upgrading** an existing stack needs two one-off scripts, `upgrade-postgres-18.sh` and `fix-keycloak-audience.sh` (see [Local deployment](../deployment/local.md#upgrading-a-stack-from-before-server-019)).

**0.18.0.** Open access by default, ODRL enforcement opt-in (ADR-0027, see [Access policies](../usage/policies.md#access-control-mode)). **Upgrading:** a deployment that relies on ODRL policies must set `FDP_ACCESS_CONTROL_MODE=odrl`. Also a dependency refresh that clears all known advisories.

## Client 0.7 to 0.9

**0.9.2.** Signed, multi-architecture images. **0.9.0** (needs server 0.20) adds the update banner, the *Updates* panel in Instance settings, *Update to X* through the host updater, and updater tokens. **0.8.0** upgrades major dependencies and pins DOMPurify past a critical advisory. **0.7.0** adds the access-control form in Instance settings and the open-mode notice on the Policies page, and fixes the policy and license editors against server 0.16 and later.

## Server 0.17.0

**Security.** Request-derived IRIs can no longer reach SPARQL unchecked: the LDP router rejects any path carrying a character forbidden in a SPARQL IRI reference with a plain `404` before authorization or the store is touched, and the storage adapter validates graph IRIs again before interpolating. Triple-store errors surface as a structured `502` (`fdp.upstream_error`) instead of a bare `500`.

**Fixed.** Per-record metrics join again: events are recorded against the canonical record IRI, independent of proxy configuration, and `fdp metrics rebase-resources --from http://<host>` re-keys historic rows. Operational probes (`readyz`, `info`, `config`) no longer count as content in the metrics. `401` responses carry an RFC 6750 `WWW-Authenticate` challenge so clients can tell a dead token from missing authentication.

**Changed.** The bundled compose stacks roll metrics up in-process (`FDP_METRICS_ROLLUP_IN_PROCESS=true`), so raw rows carrying the visitor hash are aggregated and pruned as ADR-0002 intends. The production stack sets `FORWARDED_ALLOW_IPS=*` so the server trusts Caddy's forwarded headers.

Earlier 0.16 releases corrected the FDP Ontology namespace to `https://w3id.org/fdp/fdp-o#` and migrated stored vocabulary (ADR-0026), and made FDP Index ping targets manageable at runtime (ADR-0025).

## Client 0.6.6

An expired session now ends instead of looping: a failed silent token renewal clears the session, idempotent reads are replayed once without the token so public content still loads, writes are never re-sent, and the header shows a "Session expired, sign in again" prompt. The metrics dashboard shows `4xx` and `5xx` responses as separate indicators.

## MCP bridge 0.3.1

Dual-accept of the FDP-O namespace: the bridge reads publication state and state-transition affordances in both the legacy and the corrected namespace, so it works against servers before and after 0.16.
