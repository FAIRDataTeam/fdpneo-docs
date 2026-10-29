# FDPneo documentation

FDPneo is an **AI-developed implementation** of the [FAIR Data Point specifications](https://specs.fairdatapoint.org): its code, tests and much of its documentation were generated with AI coding assistants under human direction and review. It is a metadata repository modelled on W3C DCAT, where communities describe datasets, data services, organisations, biobanks, patient registries, publications and other entities through their own SHACL schemas, and expose those descriptions over standard semantic-web protocols.

FDPneo consists of three independently released components:

| Component | Repository | What it is |
|---|---|---|
| **Server** | [`fdpneo-server`](https://github.com/FAIRDataTeam/fdpneo-server) | Python / FastAPI metadata server with a full W3C Linked Data Platform API, an access-controlled SPARQL endpoint, optional ODRL policy enforcement and anonymous usage metrics. |
| **Client** | [`fdpneo-client`](https://github.com/FAIRDataTeam/fdpneo-client) | Vue 3 single-page web application for browsing, searching and publishing metadata, with visual SHACL and ODRL editors and a metrics dashboard. |
| **MCP bridge** | [`mcp`](https://github.com/FAIRDataTeam/mcp) (`fdp-mcp`) | Model Context Protocol server that lets AI agents consume any spec-compliant FAIR Data Point. |

!!! warning "Status"
    FDPneo is **not** the FAIR Data Point reference implementation. It is another, AI-developed implementation of the FDP specifications, under active development; the bundled evaluation deployment is dev-credentialed and not hardened for production. The reference implementation is [FAIRDataTeam/FAIRDataPoint](https://github.com/FAIRDataTeam/FAIRDataPoint), documented at [docs.fairdatapoint.org](https://docs.fairdatapoint.org).

## Where to start

<div class="grid cards" markdown>

-   **I want to try it on my machine**

    ---

    One `docker compose` command brings up the client, the server and all backing services.

    [Local deployment](deployment/local.md)

-   **I want to run it for my organisation**

    ---

    The single-domain production stack serves everything under one HTTPS origin behind Caddy, with automatic certificates.

    [Production deployment](deployment/production.md)

-   **I want to use a running FDPneo**

    ---

    Signing in, browsing and searching, publishing records, managing schemas, policies, licenses and users.

    [Usage](usage/signing-in.md)

-   **I want to talk to it from a script or an agent**

    ---

    The REST/LDP API, SPARQL, personal access tokens, and the MCP bridge for AI agents.

    [API usage](api/index.md) · [MCP bridge](agents/mcp.md)

</div>

## Versions covered

This documentation was written against **server 0.21.1**, **client 0.9.2** and **fdp-mcp 0.3.1** (October 2026). Statements that depend on a specific version say so.

## See also

- [FAIR Data Point specifications](https://specs.fairdatapoint.org)
- [FAIR Data Point reference implementation documentation](https://docs.fairdatapoint.org)
- [Explainer videos](videos.md)
