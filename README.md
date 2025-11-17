# Courtyard AI Documentation

This repository contains both internal and public-facing documentation for Courtyard AI.

## Structure

- `docs-internal/` - Internal technical documentation (for team members only)
- `docs-public/` - Public customer-facing documentation
- `mkdocs-internal.yml` - Configuration for internal docs site
- `mkdocs-public.yml` - Configuration for public docs site

## Local Development

### Prerequisites

```bash
pip install mkdocs-material
```

### Preview Internal Docs

```bash
mkdocs serve -f mkdocs-internal.yml
```

Visit: http://127.0.0.1:8000

### Preview Public Docs

```bash
mkdocs serve -f mkdocs-public.yml
```

Visit: http://127.0.0.1:8000

## Deployment

- **Internal docs** are automatically deployed to GitHub Pages on the `gh-pages-internal` branch
- **Public docs** are automatically deployed to GitHub Pages on the `gh-pages-public` branch

### GitHub Pages Setup

After pushing to GitHub:

1. Go to repository **Settings** → **Pages**
2. For internal docs:
   - Source: Deploy from a branch
   - Branch: `gh-pages-internal`
   - Optionally set custom domain: `docs-internal.thecourtyard.ai`
3. For public docs (you'll need to set this up in a separate GitHub Pages configuration or separate repo):
   - Source: Deploy from a branch
   - Branch: `gh-pages-public`
   - Optionally set custom domain: `docs.thecourtyard.ai`

## Adding New Documentation

### Internal Docs

1. Add markdown files to `docs-internal/`
2. Update navigation in `mkdocs-internal.yml`
3. Commit and push - automatically deploys

### Public Docs

1. Add markdown files to `docs-public/`
2. Update navigation in `mkdocs-public.yml`
3. Commit and push - automatically deploys

## License

Internal documentation is proprietary. Public documentation is available for customer use.
