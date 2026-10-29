# Architecture decision records

The rationale for the major architectural choices is recorded as Architecture Decision Records in the server repository, in a lightweight Nygard-style format (status, context, decision, alternatives considered, consequences). ADRs are numbered sequentially and are immutable once accepted; superseding decisions are recorded as new ADRs that reference the ones they replace. The client keeps its own, independently numbered series.

## Server ADRs

| # | Title | Status |
|---|---|---|
| [0001](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0001-modular-monolith.md) | Modular monolith over microservices | Accepted |
| [0002](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0002-anonymous-metrics.md) | Anonymous-by-design metrics pipeline | Accepted |
| [0003](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0003-fixed-postgres-for-operational-state.md) | Fixed Postgres for operational state | Accepted |
| [0004](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0004-sparql-access-via-named-graph-projection.md) | SPARQL access control via named-graph projection | Accepted |
| [0005](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0005-triple-store-pluggability.md) | Pluggable triple store via SPARQL 1.1 Protocol | Accepted |
| [0006](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0006-odrl-profile-permission-prohibition.md) | ODRL profile: Permissions and Prohibitions | Accepted |
| [0007](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0007-one-graph-per-record.md) | One named graph per metadata record | Accepted |
| [0008](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0008-full-ldp-with-patch.md) | Full LDP implementation including PATCH | Accepted |
| [0009](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0009-runtime-resource-definitions.md) | Runtime-mutable resource definitions stored as RDF | Accepted |
| [0010](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0010-metadata-publication-state.md) | Metadata publication state (draft, published, archived) | Accepted |
| [0011](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0011-api-keys.md) | API keys for machine-to-machine access | Accepted |
| [0012](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0012-first-class-odrl-policy-and-license-documents.md) | First-class ODRL policy and license documents | Accepted |
| [0013](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0013-user-management-facade.md) | User-management façade over the IdP Admin API | Accepted |
| [0014](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0014-persistent-identifiers.md) | Persistent identifiers: base/serving split, dual model, W3ID | Accepted |
| [0015](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0015-form-grouping-presentation-overlay.md) | Cross-schema form grouping via a presentation overlay | Proposed |
| [0016](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0016-backup-restore-migration.md) | Faithful backup/restore and instance migration | Proposed |
| [0017](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0017-alternative-identifiers-and-signposting.md) | Structured alternative identifiers and FAIR Signposting | Proposed |
| [0018](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0018-agent-consumption-mcp-server.md) | Agent consumption via a standalone MCP sidecar (`fdp-mcp`) | Accepted |
| [0019](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0019-record-schema-binding-and-versioning.md) | Self-describing record–schema binding and schema versioning | Accepted |
| [0020](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0020-product-distributions.md) | Products as distributions: module manifests and composed deployments | Proposed |
| [0021](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0021-fair-discovery-product.md) | FAIR Discovery, the aggregation product: name, tiers, and validation posture | Proposed |
| [0022](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0022-in-band-affordance-advertisement.md) | In-band affordance advertisement: closing the HATEOAS gap | Proposed |
| [0023](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0023-downstream-composition-seams.md) | Downstream composition seams on `create_app` | Accepted |
| [0024](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0024-mcp-opt-in-write-tools.md) | Opt-in write tools in the MCP sidecar (`fdp-mcp`) | Accepted |
| [0025](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0025-runtime-index-targets.md) | Runtime-managed FDP Index ping targets | Accepted |
| [0026](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0026-fdp-o-namespace-correction.md) | Correct the FDP Ontology namespace and migrate stored vocabulary | Accepted |
| [0027](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0027-open-access-default-odrl-opt-in.md) | Open access by default, ODRL evaluation opt-in | Accepted |
| [0029](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/0029-full-deployment-backup.md) | Full-deployment backup for moving and restoring an instance | Accepted |

The [ADR index](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/adr/README.md) in the repository is authoritative.

## Client ADRs

| # | Title |
|---|---|
| [0001](https://github.com/FAIRDataTeam/fdpneo-client/blob/main/docs/adr/0001-catalogrecord-consumption.md) | CatalogRecord consumption |

## Related design documents

- [Architecture document](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/architecture/README.md)
- [Agent consumption vision](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/architecture/agent-consumption-vision.md)
- [Discovery](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/architecture/discovery.md) and [platform vision alignment](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/architecture/platform-vision-alignment.md)
- [Roadmap v1.x](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/roadmap-v1.x.md)
- Conformance notes: [LDP](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/conformance/ldp-conformance.md), [Signposting](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/conformance/signposting-conformance.md), [agent-ready](https://github.com/FAIRDataTeam/fdpneo-server/blob/main/docs/conformance/agent-ready.md)
