# About FDPneo

FDPneo is an **AI-developed** FAIR Data Point: a FAIR-aligned metadata repository that implements the [FAIR Data Point (FDP) specifications](https://specs.fairdatapoint.org). It is modelled on W3C DCAT, uses user-defined SHACL schemas to describe any kind of resource a community wishes to describe, enforces access control with W3C ODRL policies, and exposes everything through a full W3C Linked Data Platform (LDP) API and a SPARQL endpoint.

## An AI-developed implementation

FDPneo is an experiment in building a complete FAIR infrastructure component with AI coding assistants. The architecture, the decision records, the server and client code, the test suites and the deployment stacks were generated with AI under human direction, with a human setting the goals, reviewing the design decisions and validating the result against the FDP specifications. This is stated openly so that adopters can weigh it: the code base is large and was produced quickly, it has been audited for security more than once, and it is still experimental.

!!! warning "Not the reference implementation"
    FDPneo is separate from the [FAIR Data Point reference implementation](https://github.com/FAIRDataTeam/FAIRDataPoint). The core stack runs and can be brought up with one command, but parts of the design are still target state, and the bundled evaluation deployment is not hardened for production.

## What an FDP does

A FAIR Data Point is a metadata repository. It serves descriptions of datasets, data services, organisations, biobanks, patient registries, scientific publications, methodologies, semantic artefacts, and anything else a community wishes to describe through its own SHACL schemas, over standard semantic-web protocols. Consumers browse and search the metadata through a REST API (Linked Data Platform), run SPARQL queries against it, and reach the underlying open-access data distributions where policy permits. Every record is dereferenceable Linked Data, so a record's URL is its identifier.

## Features

**Metadata management.** Records follow the DCAT hierarchy (a Catalog contains Datasets, which contain Distributions), and a deployment can define further resource types at runtime. Records are validated against the SHACL schema registered for their type. Each record moves through a publication lifecycle of draft, published and archived; only published records are visible to anonymous users.

**User-defined schemas.** Administrators author and publish SHACL shapes, either as Turtle in a text-first editor or in a visual node-based editor with a live Turtle preview. Shapes are versioned at a stable IRI. Resource types and their child-link relationships are registered against published shapes, and new types appear across the application without a rebuild.

**Access control with ODRL.** Access to records is governed by ODRL Offers in an FDP-specific profile (Permissions and Prohibitions). A record opts into a policy through `dct:rights`, and the server's policy decision point enforces it. Offers are versioned, and grants materialise Agreements that reference the Offer version in force at grant time, giving an audit trail. A guided ODRL editor in the client composes Offers within the profile.

**Licenses as managed documents.** Reusable license documents (for example CC BY 4.0 or CC0) are curated once and referenced from records through `dct:license`. Licenses are descriptive and are not access policy.

**Search and SPARQL.** Free-text search with facets built from the live index, saved searches, and a SPARQL 1.1 playground. The SPARQL endpoint is exposed by the FDPneo API, not by the triple store directly, and access control is enforced at the query layer.

**External identity.** FDPneo has no internal user database. Authentication is delegated to any OIDC-compliant identity provider (Keycloak is bundled for evaluation); the server only validates bearer tokens. An optional user-management facade lets administrators invite users and assign roles through the identity provider's admin API.

**Personal access tokens.** Signed-in users create API keys for scripts and machine clients, and revoke them at any time.

**Anonymous-by-design metrics.** Usage metrics are stripped of personally identifying data at ingress, so the metrics module is structurally incapable of producing user-identifying reports. A dashboard shows views and downloads per record and schema, SPARQL query counts, unique visitors per day, and, when a GeoIP database is configured, geography.

**Deployment profiles.** A community bundles its schemas, default ODRL Offers, container hierarchy and seed records into a versioned, validated profile that bootstraps a new deployment. The bundled default profile provides the standard FDP/DCAT types.

**Backup, restore and migration.** The `fdp backup` commands dump every named graph with a manifest, restore a dump faithfully, or import records into another deployment under its own base IRI.

**Agent access.** The separate `fdp-mcp` bridge exposes the public surface of any spec-compliant FDP (FDPneo or the reference implementation) as Model Context Protocol tools, so AI agents can browse, search and query a repository and cite records by IRI. See [MCP bridge](../agents/mcp.md).

**White-labelling.** The client's colours, logo, favicon and organisation name are configured at container start-up through environment variables, authored with an in-app Appearance editor.

## Key design choices

- Python and FastAPI for the server, Vue 3 and TypeScript for the client, in separate repositories with independent release cycles.
- External OIDC for authentication, with no internal user database.
- A modular monolith with bounded contexts for the metadata provider, security enforcer, metrics gatherer and simple data provider.
- PostgreSQL for operational state (metrics, search index, authorization cache, audit, API keys) and a pluggable triple store, reached over SPARQL 1.1 Protocol, for RDF metadata. GraphDB is the recommended default.
- One named graph per metadata record, which reduces per-record access control to set membership over graph IRIs and lets LDP `PATCH` map directly onto SPARQL Update.
- Full LDP, including `PATCH` for partial updates.
- An ODRL profile limited to Permissions and Prohibitions, with versioned Offers and materialised Agreements.
- Anonymous-by-design metrics, GDPR-safe by construction rather than by policy.

The rationale behind each choice is recorded in the [architecture decision records](../development/adrs.md).

## Relationship to the reference implementation

FDPneo preserves the FDP's core mission and conforms to the published specifications for metadata structure, vocabulary and content negotiation. Compared with the reference implementation it drops internal user management in favour of OIDC federated identity, separates server and client into independent repositories, replaces partial LDP support with full LDP including `PATCH`, and adds ODRL-based access control enforcement, anonymous usage metrics and a simple data provider for open distributions.

Some things are intentionally out of scope for the first version. There is no FDP-to-FDP federation (cross-repository query belongs to the separate FAIR Discovery aggregation product), no access-controlled data distribution, no property-level access control within a record, and no ODRL Duty enforcement.

## Security

The security posture is documented in the repositories: a secure-development guide and a deployment-hardening checklist in `fdpneo-server/docs/security/`, and security audit reports. The evaluation stack ships development credentials that are publicly known and must be rotated before any real deployment; see [Production deployment](../deployment/production.md).

## License

FDPneo is released under the [MIT License](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/LICENSE), matching the FDP reference implementation. Contributions require a DCO sign-off.
