# API usage

Everything the web client does goes through the FDPneo HTTP API, and you can use it directly from scripts, pipelines and other services. There are two surfaces: the **Linked Data Platform (LDP)** API, where records live at their own IRIs, and the **management API** under `/fdp-api/`, which also hosts the SPARQL endpoint, search, metrics and administration.

The machine-readable OpenAPI specification is always served at `/fdp-api/openapi.json`. The interactive Swagger UI at `/fdp-api/docs` and ReDoc at `/fdp-api/redoc` are served when `EXPOSE_API_DOCS=true` or in development mode; the evaluation stack enables them.

## Obtaining a token

Two kinds of bearer token work.

**OIDC access tokens.** Any client can run the OIDC Authorization Code flow with PKCE against the identity provider, or the Client Credentials flow for service-to-service use, and send the resulting JWT as `Authorization: Bearer …`. The server validates it against the provider's published keys.

**Personal access tokens (API keys).** Sign in to the client, open **avatar, Access tokens**, and create a token. FDPneo mints keys with an `fdpk_` prefix and stores only a hash. Use it the same way:

```bash
export KEY=fdpk_...
curl -H "Authorization: Bearer $KEY" http://localhost:8000/fdp-api/me/dashboard
```

Keys can be listed and revoked at `/fdp-api/me/api-keys`.

## Reading records

A record's URL is its identifier, and it dereferences with content negotiation:

```bash
# the repository root
curl -H 'Accept: text/turtle' http://localhost:8000/
# a catalog, as JSON-LD
curl -H 'Accept: application/ld+json' http://localhost:8000/catalog/<id>
```

Supported serialisations include Turtle, JSON-LD, RDF/XML and N-Triples. Containers list their members with LDP containment triples, and large containers are paged (`/fdp-api/catalog/<id>/page/<childPrefix>`). `/fdp-api/<type>/<id>/expanded` returns a record together with its children; `/fdp-api/<type>/spec` returns the SHACL shape the type is validated against.

Anonymous requests see published records only. With a bearer token you also see drafts you own or may read.

## Writing records

Records are created with LDP `POST` of RDF (Turtle by default) to the type's collection, naming the parent with `dct:isPartOf` in the body (a `Slug` header suggests the identifier suffix), or with `PUT` to a chosen IRI:

```bash
curl -i -X POST -H "Authorization: Bearer $KEY" -H "Content-Type: text/turtle" \
  -H "Slug: my-dataset" --data-binary @dataset.ttl http://localhost:8000/dataset
# HTTP/1.1 201 Created
# etag: "c8787f62…"
# location: http://localhost:8000/dataset/my-dataset
# fdp-metadata-state: DRAFT
```

where `dataset.ttl` uses `<>` as the subject and links the parent:

```turtle
@prefix dcat: <http://www.w3.org/ns/dcat#> .
@prefix dct:  <http://purl.org/dc/terms/> .
<> a dcat:Dataset ;
    dct:title "Noise levels around Enschede Airport, 2025" ;
    dct:publisher <https://example.org/org/twente-monitoring-network> ;
    dct:isPartOf <http://localhost:8000/catalog/environmental-monitoring> .
```

The body is validated against the type's SHACL shape; violations come back as a structured error. The server fills in the record's meta-metadata (creator, timestamps, version) itself.

**Publication state.** A new record starts as `DRAFT`: the create returns `201` with an `FDP-Metadata-State` header saying so, and the record is `404` to everyone except its owner and administrators until published. Either publish it afterwards:

```bash
curl -X POST -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"to":"PUBLISHED"}' http://localhost:8000/fdp-api/dataset/<id>/state
# {"record":"http://localhost:8000/dataset/<id>","from_state":"DRAFT","to_state":"PUBLISHED"}
```

or create it visible in one step with `Prefer: publication-state=PUBLISHED` on the creating request. States are `DRAFT`, `PUBLISHED` and `ARCHIVED`.

**Updates and deletes** (`PUT`, `PATCH`, `DELETE` of an existing record) require `If-Match` with the record's current ETag, which every `GET` returns, so concurrent edits cannot silently overwrite each other. `PATCH` takes `application/sparql-update` scoped to the record:

```bash
curl -X PATCH -H "Authorization: Bearer $KEY" -H "If-Match: \"<etag>\"" \
  -H "Content-Type: application/sparql-update" \
  --data 'PREFIX dct: <http://purl.org/dc/terms/>
          DELETE { ?s dct:title ?t } INSERT { ?s dct:title "New title" } WHERE { ?s dct:title ?t }' \
  http://localhost:8000/dataset/<id>
```

Whether a write is allowed depends on the access-control mode. In open mode, the default, the caller needs one of the writer roles (`admin` or `steward` unless configured otherwise); in ODRL mode the record's policy decides. A denied request returns `403`. The mode itself is the setting `access.control` (`GET` or, as an administrator, `PUT /fdp-api/settings/access.control`).

## SPARQL

`GET` or `POST` `/fdp-api/sparql` implements the SPARQL 1.1 Protocol. Queries are rewritten to the named graphs the caller may read, `SERVICE` clauses are rejected, and updates require authentication. Anonymous callers see exactly the published metadata. Every record lives in its own named graph and the default graph is empty, so match inside `GRAPH ?g { … }`.

```bash
curl -H 'Accept: application/sparql-results+json' \
  --data-urlencode 'query=SELECT ?s WHERE { GRAPH ?g { ?s a <http://www.w3.org/ns/dcat#Dataset> } } LIMIT 10' \
  http://localhost:8000/fdp-api/sparql
```

## Search

`POST /fdp-api/search` takes a free-text query and facet filters and returns policy-aware results. Signed-in users manage saved queries at `/fdp-api/me/saved-queries`.

## Management endpoints

| Area | Endpoints |
|---|---|
| Self-description | `GET /fdp-api/config`, `/info`, `/healthz`, `/readyz` |
| Schemas | `/fdp-api/schemas`, `/fdp-api/schemas/{id}`, `/{id}/{version}`, `/{id}/validate` |
| Resource types | `/fdp-api/resource-definitions`, `/{slug}` |
| Policies and licenses | `/fdp-api/policies`, `/fdp-api/licenses`, each with `/{id}` and `/{id}/validate` |
| Settings | `/fdp-api/settings`, `/{key}` |
| Users (when enabled) | `/fdp-api/users` |
| Metrics | `/fdp-api/metrics/summary`, `/timeseries/daily`, `/top-resources`, `/geography` |
| Data | `GET /fdp-api/data/{distribution_id}` serves or redirects to an open distribution; `/sparql` under it queries a SPARQL data service |
| Administration | `POST /fdp-api/admin/reset`, `/fdp-api/admin/backup/dump` (`?scope=full` with a passphrase for an encrypted full backup), `/restore`, `/jobs/{id}` |
| Updates | `GET /fdp-api/admin/updates`, `POST /fdp-api/admin/updates/check`, `/fdp-api/admin/updates/requests` (admin), `/fdp-api/admin/updates/agent/*` (updater token only) |
| Profiles | `GET /fdp-api/profiles`, `/{id}`, `/{id}/{version}` |

The OpenAPI specification is the authoritative list; the client's TypeScript types are generated from it.

## Agents

If you want an AI agent rather than a script to use the repository, the [MCP bridge](../agents/mcp.md) wraps this public surface as Model Context Protocol tools.
