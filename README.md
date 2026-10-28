# FDPneo documentation

Source of the FDPneo documentation site, built with [MkDocs](https://www.mkdocs.org/)
and the [Material](https://squidfunk.github.io/mkdocs-material/) theme and
published on Read the Docs.

The site documents the three FDPneo repositories together:

- [`fdpneo-server`](https://github.com/FAIRDataTeam/fdpneo-server), the FastAPI metadata server
- [`fdpneo-client`](https://github.com/FAIRDataTeam/fdpneo-client), the Vue 3 web client
- [`mcp`](https://github.com/FAIRDataTeam/mcp) (`fdp-mcp`), the Model Context Protocol bridge for agents

## Local preview

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve          # http://127.0.0.1:8000
mkdocs build --strict # fails on broken links and missing pages
```

## Layout

```
mkdocs.yml          site configuration and navigation
.readthedocs.yaml   Read the Docs build configuration
requirements.txt    Python dependencies for the build
docs/               the pages (Markdown)
docs/assets/        screenshots, diagrams and the logo
```

## Keeping it current

The pages consolidate material that also lives in the repositories (READMEs,
`client/docs/`, `server/docs/`, `mcp/docs/`). When one of those changes, update
the matching page here. Version-specific statements name the version they were
verified against.

## License

MIT, matching the FDPneo repositories.
