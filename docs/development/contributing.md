# Contributing

FDPneo is developed in three repositories under the [FAIRDataTeam](https://github.com/FAIRDataTeam) organisation: [`fdpneo-server`](https://github.com/FAIRDataTeam/fdpneo-server), [`fdpneo-client`](https://github.com/FAIRDataTeam/fdpneo-client) and [`mcp`](https://github.com/FAIRDataTeam/mcp). Each has its own `CONTRIBUTING.md`, `CHANGELOG.md` and issue tracker.

## License and sign-off

All three projects are licensed under the MIT License. By contributing you agree that your contributions are licensed under the same terms.

Every commit must be signed off, certifying the [Developer Certificate of Origin](https://developercertificate.org/):

```bash
git commit -s
```

This appends a `Signed-off-by: Your Name <you@example.org>` trailer from your git identity. Pull requests with unsigned commits fail CI; fix with `git rebase --signoff HEAD~<n>` and `git push --force-with-lease`.

## Process

1. Open or comment on an issue before large changes.
2. Fork, branch, and keep pull requests focused.
3. Make sure tests and linters pass locally before opening the pull request.

## Working on the server

Set up the [development environment](../deployment/development.md), then read the server's developer documentation in `docs/dev-docs/`, which takes a new engineer from a fresh clone to a change that respects the architecture: system context, the modular monolith and its import rules, code organisation, the request lifecycle, the key processes, the data model, contributing, and backup and restore. When the developer guide and the architecture document or an ADR disagree, the architecture document and the ADRs win.

The quality gate is `ruff` (lint and format), `pyright` in strict mode and `pytest`:

```bash
uv run ruff check . && uv run ruff format --check . && uv run pyright && uv run pytest
```

Tests are organised as a pyramid: fast unit tests without I/O, integration tests on testcontainers-backed GraphDB and PostgreSQL, contract tests for OpenAPI conformance, and conformance tests against the FDP specifications and the LDP test suite.

Changes to the public HTTP API change the OpenAPI specification, and the client then needs `npm run generate-api`; flag this in the pull request. Architectural changes are recorded as a new [ADR](adrs.md); existing ADRs are immutable once accepted and are superseded rather than edited.

## Working on the client

The client is Vue 3 with the Composition API, strict TypeScript 5, Vite, Pinia, TanStack Query, PrimeVue and Vue Flow. Unit tests are Vitest specs colocated with their sources (`*.spec.ts`); end-to-end smoke tests use Playwright. The `CLAUDE.md` in the repository records the current conventions.

## Working on the MCP bridge

The bridge is Python, managed with uv. Its tool contract is implementation-agnostic and published in `docs/mcp-tool-surface.md`; the code base is that contract's reference implementation. Capabilities the bridge cannot implement cleanly over the public FDP surface are recorded in `docs/fdp-api-gaps.md` and fed back to the server roadmap.

## Security

Each repository has a `SECURITY.md` with the disclosure process and a secure-development guide under `docs/security/`. Please report vulnerabilities privately as described there rather than in a public issue.
