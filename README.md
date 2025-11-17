# Courtyard AI Public Documentation

Public customer-facing documentation for Courtyard AI integration guides.

**Note**: Internal documentation has been moved to a separate private repository: [courtyard-docs-internal](https://github.com/courtyard-ai/courtyard-docs-internal)

## Structure

- `docs-public/` - Public customer-facing documentation (Hebrew with RTL support)
- `mkdocs-public.yml` - Configuration for public docs site

## Local Development

### Prerequisites

```bash
pip install mkdocs-material
```

### Preview Docs

```bash
mkdocs serve -f mkdocs-public.yml
```

Visit: http://127.0.0.1:8000

## Deployment

Public docs are automatically deployed to GitHub Pages on push to main branch.

**Live Site**: https://courtyard-ai.github.io/courtyard-docs/

### GitHub Pages Setup

The repository is configured to deploy from the `gh-pages-public` branch automatically via GitHub Actions.

Optionally set custom domain: `docs.thecourtyard.ai`

## Adding New Documentation

1. Add markdown files to `docs-public/`
2. Update navigation in `mkdocs-public.yml`
3. Commit and push - automatically deploys

## License

Public documentation is available for customer use.
