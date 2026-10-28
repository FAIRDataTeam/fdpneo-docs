# MCP bridge (fdp-mcp)

`fdp-mcp` is a standalone [Model Context Protocol](https://modelcontextprotocol.io) (MCP) server that makes **any spec-compliant FAIR Data Point** consumable by AI agents. Point it at an FDP, connect an MCP client (Claude, an IDE agent, or any other), and converse with the repository's metadata, grounded in real records and cited by IRI. It lives in its own repository, [FAIRDataTeam/mcp](https://github.com/FAIRDataTeam/mcp), and is released independently.

It works on the network that exists. The bridge consumes only the public, specified FDP surface (LDP, the SPARQL endpoint, the search API, SHACL shapes), so it works against FDPneo **and** the Java reference implementation with no server-side changes. It is read-only by default, speaks only to the configured FDP's origin, adds no authorization of its own, and never fabricates: a capability the target FDP lacks degrades to a structured `unsupported` result.

```
MCP client  ──MCP──▶  fdp-mcp  ──HTTP (public surface only)──▶  FAIR Data Point
(Claude, …)                     LDP GET · /sparql · search · SHACL shapes
```

## Start the bridge

The only required setting is `FDP_BASE_URL`:

```bash
docker run --rm -p 8080:8000 \
  -e FDP_BASE_URL=https://your-fdp.example.org \
  ghcr.io/fairdatateam/fdp-mcp:main
```

The bridge serves the streamable-HTTP MCP transport at `http://localhost:8080/mcp` and a liveness probe at `http://localhost:8080/healthz`. At the time of writing the image is private on GHCR; `docker login ghcr.io` with a token that has `read:packages`, or build it from the repository (`docker build -t fdp-mcp:main .`). From a source checkout, `uv sync` then `FDP_BASE_URL=… uv run fdp-mcp`.

In the FDPneo compose stacks the bridge is a service behind the `mcp` profile: `--profile mcp` on the `up` command serves it at `http://localhost:8002/mcp` (evaluation) or `https://PUBLIC_HOST/mcp` (production). See [Local deployment](../deployment/local.md) and [Production deployment](../deployment/production.md).

## Connect a client

**Claude Desktop** launches MCP servers as local subprocesses over stdio. Add to `claude_desktop_config.json` (Settings, Developer, Edit Config):

```json
{
  "mcpServers": {
    "fdp": {
      "command": "docker",
      "args": [
        "run", "--rm", "-i",
        "-e", "FDP_BASE_URL=https://your-fdp.example.org",
        "-e", "FDP_MCP_TRANSPORT=stdio",
        "ghcr.io/fairdatateam/fdp-mcp:main"
      ]
    }
  }
}
```

**Streamable-HTTP clients** connect to the bridge URL, for example `http://localhost:8080/mcp`. **MCP Inspector** (`npx @modelcontextprotocol/inspector`) lists the tools and lets you call them by hand, which is a quick way to confirm a target FDP before wiring up a chat client.

## Forwarding an API key

The bridge is anonymous by default and sees exactly what the FDP shows the public. To see restricted records, a streamable-HTTP client supplies your FDP API key (FDPneo mints them with an `fdpk_` prefix) as `Authorization: Bearer <key>` on the MCP session. The bridge forwards it verbatim on that session's requests only, holds it in memory, and never stores, caches or logs it. The FDP's own authorization decides what you can see.

## The tools

| Tool | Ask it… |
|---|---|
| `fdp_info` | "What is this repository, who runs it, and what kinds of records does it hold?" Kinds and optional endpoints are discovered at runtime. |
| `list_records` | "List the catalogs or datasets, optionally under a given parent." Paged summaries, IRIs always present. |
| `get_record` | "Show the full metadata for this IRI." Framed JSON-LD by default, raw RDF on request, with signposting and containment links and the record's `etag`. |
| `get_schema` | "What fields does a dataset here have, and what do they mean?" |
| `search` | "Find records about X." Optional; degrades if the FDP has no search API. |
| `sparql_query` | "Run this read-only SPARQL query." Update and `SERVICE` forms are refused before any network call. |
| `get_access_conditions` | "May I use this, under what licence, and whom do I contact?" Resolves `dct:license`, ODRL `dct:rights`, `dct:accessRights` and contact points, and reports `determinable: false` when the metadata is silent. |
| `list_data_services` | "What services (for example query endpoints) does this repository expose?" |

### Writing (opt-in)

With `FDP_MCP_ENABLE_WRITE=true` the bridge also registers `create_record`, `update_record` (replace or SPARQL-update patch, `if_match` mandatory), `delete_record` and `set_record_state` (`DRAFT`, `PUBLISHED`, `ARCHIVED`). Write tools refuse an anonymous session before any network call, so they need a forwarded credential over streamable-HTTP (`stdio` is always anonymous). Every write is decided by the FDP: SHACL violations come back as `validation_failed`, a stale `if_match` as `precondition_failed`, and a `401` or `403` as `upstream_unauthorized`.

## Configuration

| Variable | Default | Purpose |
|---|---|---|
| `FDP_BASE_URL` | required | Base URL of the target FDP. The bridge speaks only to this origin. |
| `FDP_MCP_TRANSPORT` | `streamable-http` | `streamable-http` (network sidecar) or `stdio` (desktop-client subprocess). |
| `FDP_MCP_HOST`, `FDP_MCP_PORT` | `127.0.0.1`, `8000` | Bind address and port (the image binds `0.0.0.0`). |
| `FDP_MCP_LOG_LEVEL` | `info` | Log level; credentials are always redacted. |
| `FDP_MCP_FDP_TIMEOUT_SECONDS` | `30` | Per-request timeout for FDP calls. |
| `FDP_MCP_FDP_MAX_RESPONSE_BYTES` | `10485760` | Response-size cap. |
| `FDP_MCP_FDP_SPARQL_PATH`, `FDP_MCP_FDP_SEARCH_PATH` | `/fdp-api/sparql`, `/fdp-api/search` | Endpoint path overrides; an endpoint advertised in the FDP's root record is preferred when left at default. |
| `FDP_MCP_ENABLE_WRITE` | `false` | Register the write tools. |
| `FDP_MCP_MAX_CONCURRENT_SESSIONS`, `FDP_MCP_SESSION_IDLE_TIMEOUT_SECONDS`, `FDP_MCP_SESSION_LIFETIME_TIMEOUT_SECONDS` | `100`, `300`, `3600` | Session limits for streamable-HTTP. |

There is no credential setting and no egress allow-list by design.

## A three-minute demo

1. Ask the agent what datasets the repository holds about a topic, which are openly licensed, and how to get access to the restricted ones. It answers from `search` or `list_records`, `get_record` and `get_access_conditions`, citing IRIs.
2. Ask a question that needs the schema, such as which registries record a particular field. The agent reads the SHACL shape with `get_schema` and then queries with `sparql_query`.
3. Restart the bridge with `FDP_BASE_URL` pointing at a reference-implementation FDP on the existing network and have the same conversation. Where the second FDP lacks a capability, the tool degrades explicitly and the agent routes around it.

## Troubleshooting

- A tool returns `unsupported`: the target FDP does not expose that capability (often search on the reference implementation). Use the suggested `alternative`.
- `not_found` for a record you know exists: a spec-compliant FDP returns `404` both for missing records and for ones hidden from you. Supplying an API key may reveal it.
- `sparql_query` returns no rows for a non-empty repository: FDPneo projects records into named graphs, so the default graph is often empty. Match named graphs explicitly: `SELECT ?s ?p ?o WHERE { GRAPH ?g { ?s ?p ?o } }`.
- `curl http://localhost:8080/healthz` should return `{"status":"ok"}`.

The tool contract is published in the repository as [`docs/mcp-tool-surface.md`](https://github.com/FAIRDataTeam/mcp/blob/main/docs/mcp-tool-surface.md); the motivation is in the server's [agent consumption vision](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/architecture/agent-consumption-vision.md) and ADR-0018.
