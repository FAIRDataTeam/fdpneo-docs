# Components

An FDPneo deployment is five processes, plus an optional sixth for agent access. The client and server are stateless; all durable state lives in the triple store and PostgreSQL.

| Component | Technology | Default port (evaluation stack) | Role |
|---|---|---|---|
| **Client** (`fdpneo-client`) | Vue 3, TypeScript, Vite, served by nginx | `5173` | The browser user interface. Talks to the API and to the identity provider directly. |
| **Server** (`fdpneo-server`) | Python 3.12+ (image: 3.15), FastAPI, uvicorn | `8000` | REST/LDP API and SPARQL endpoint. Validates bearer tokens and enforces access control, open by default or ODRL policies when enabled. |
| **Triple store** | Ontotext GraphDB 10.8 (default) | internal | Holds all RDF: metadata records, schemas, policies, licenses, resource definitions. |
| **PostgreSQL** | PostgreSQL 18 | internal | Operational state: metrics, search index, authorization cache, audit log, saved queries, API-key hashes, job state. |
| **Keycloak** | Keycloak 26 | `8081` | OIDC identity provider. FDPneo has no internal user database. |
| **MCP bridge** (`fdp-mcp`, optional) | Python | `8002` | Model Context Protocol server exposing the public FDP surface to AI agents. |

```
                 ┌────────────┐     OIDC login (Authorization Code + PKCE)
   Browser ──────│   client   │──────────────────────────────► Keycloak
                 └─────┬──────┘                                      ▲
                       │ REST/LDP + SPARQL (Bearer token)            │ validates
                       ▼                                             │ tokens
                 ┌────────────┐         ┌──────────────┐            │
                 │   server   │────────►│   GraphDB    │ (RDF)      │
                 │            │────────►│  PostgreSQL  │ (state) ───┘
                 └────────────┘         └──────────────┘
                       ▲
                 ┌─────┴──────┐
   AI agent ─────│  fdp-mcp   │  public FDP surface only (LDP, /sparql, search, shapes)
                 └────────────┘
```

## Triple store

The server is triple-store-agnostic: any SPARQL 1.1 Protocol endpoint works, and Fuseki and Oxigraph are tested alternatives. GraphDB is the recommended default for its administrative interface and scalability, and it is the one wired into the bundled compose files, where an init container creates the `fdp` repository automatically on first boot. Every metadata record lives in its own named graph, with sibling meta and audit graphs.

## PostgreSQL

PostgreSQL holds everything that is not RDF. If it is down, search and the steward dashboard fail even though browsing still works. Migrations are run with `fdp db migrate`, which the container entrypoint does automatically.

## Server

The server is a modular monolith with bounded contexts for identity, metadata, policy, access (SPARQL), data, metrics and storage, plus a shared kernel. It exposes two HTTP surfaces, the REST/LDP API and the SPARQL endpoint, that share authentication, authorization and storage. In the bundled stacks the API is mounted under `/fdp-api`, and record IRIs are minted under `BASE_URL`.

## Client

The client is a static single-page application. It authenticates with the identity provider directly (Authorization Code flow with PKCE); the server is not in the authentication path beyond validating tokens. Its runtime configuration (API URL, public origin, branding) is injected into `config.js` by the container entrypoint at start-up, so one image serves any deployment.

## Keycloak

Keycloak is bundled as the identity provider. The evaluation stack runs it in development mode with a pre-imported `fdp-dev` realm; the production stack runs it in production mode against its own database with a realm template that contains no demo users. Any other OIDC-compliant provider can be used instead.

## Reverse proxy

The production stack puts Caddy in front of everything so that the client, the API and Keycloak share one HTTPS origin, with certificates obtained automatically. An nginx variant is provided for hosts that already run nginx. The proxy splits browser traffic from machine traffic on content negotiation: requests preferring `text/html` go to the client, and everything else (Turtle, JSON-LD, `*/*`) dereferences straight to the server, so a record IRI serves a web page to a person and RDF to a program.

See [Architecture](architecture.md) for the server internals and [Deployment](../deployment/local.md) for how the pieces are wired together.
