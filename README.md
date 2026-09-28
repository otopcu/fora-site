# Fora Site

Public documentation site for Fora: https://otopcu.github.io/fora-site/

The site is selective: it publishes the home page and user-facing manuals, while internal architecture notes stay in
the main Fora workspace.

## Source

The site is maintained in the Fora workspace under `docs/fora-site/` and pushed to this repository with
`git subtree push --prefix=docs/fora-site fora-site main`. Commits made directly here must be pulled back with
`git subtree pull` before the next push. The full procedure is in the Fora wiki, `Fora/ReleaseManagement.md`
("Documentation and Web Page Publishing").

## Local preview

```powershell
python -m venv .venv
.venv\Scripts\python -m pip install -r requirements.txt
.venv\Scripts\python -m mkdocs serve
```

## Publish

GitHub Pages is built by `.github/workflows/pages.yml` after every push to `main`, or when the workflow is started
manually.
