# Fora Site

Public documentation site for Fora.

The first public pass is intentionally selective: it publishes user-facing manuals and a concise home page, while internal architecture notes remain in the main Fora workspace.

## Local preview

```powershell
pip install -r requirements.txt
mkdocs serve
```

## Publish

GitHub Pages is built by `.github/workflows/pages.yml` after pushing to `main`.
