# Changelog

Each repository keeps its own changelog in the [Keep a Changelog](https://keepachangelog.com/) format, and releases are tagged independently. This page points at them and summarises the versions this documentation was written against.

| Component | Current version | Changelog |
|---|---|---|
| Server | 0.17.0 (2026-09-09) | [`fdpneo-server/CHANGELOG.md`](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/CHANGELOG.md) |
| Client | 0.6.6 (2026-09-09) | [`fdpneo-client/CHANGELOG.md`](https://github.com/FAIRDataTeam/fdpneo-client/blob/main/CHANGELOG.md) |
| MCP bridge | 0.3.1 (2026-09-02) | [`mcp/CHANGELOG.md`](https://github.com/FAIRDataTeam/mcp/blob/main/CHANGELOG.md) |

## Server 0.17.0

**Security.** Request-derived IRIs can no longer reach SPARQL unchecked: the LDP router rejects any path carrying a character forbidden in a SPARQL IRI reference with a plain `404` before authorization or the store is touched, and the storage adapter validates graph IRIs again before interpolating. Triple-store errors surface as a structured `502` (`fdp.upstream_error`) instead of a bare `500`.

**Fixed.** Per-record metrics join again: events are recorded against the canonical record IRI, independent of proxy configuration, and `fdp metrics rebase-resources --from http://<host>` re-keys historic rows. Operational probes (`readyz`, `info`, `config`) no longer count as content in the metrics. `401` responses carry an RFC 6750 `WWW-Authenticate` challenge so clients can tell a dead token from missing authentication.

**Changed.** The bundled compose stacks roll metrics up in-process (`FDP_METRICS_ROLLUP_IN_PROCESS=true`), so raw rows carrying the visitor hash are aggregated and pruned as ADR-0002 intends. The production stack sets `FORWARDED_ALLOW_IPS=*` so the server trusts Caddy's forwarded headers.

Earlier 0.16 releases corrected the FDP Ontology namespace to `https://w3id.org/fdp/fdp-o#` and migrated stored vocabulary (ADR-0026), and made FDP Index ping targets manageable at runtime (ADR-0025).

## Client 0.6.6

An expired session now ends instead of looping: a failed silent token renewal clears the session, idempotent reads are replayed once without the token so public content still loads, writes are never re-sent, and the header shows a "Session expired, sign in again" prompt. The metrics dashboard shows `4xx` and `5xx` responses as separate indicators.

## MCP bridge 0.3.1

Dual-accept of the FDP-O namespace: the bridge reads publication state and state-transition affordances in both the legacy and the corrected namespace, so it works against servers before and after 0.16.
