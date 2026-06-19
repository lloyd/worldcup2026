# World Cup 2026 — The Prediction League

A single-page data story about the Yarris × Hilaiel family group-chat World Cup 2026
prediction league: seven forecasters, eight matchdays, twenty-eight games, scored
**2 / 3 / 5** in Jake's spreadsheet.

Standings are the family's own official totals; each matchday's points reconcile
exactly to them. Everything is anchored to the 2026 World Cup group stage (Jun 11–18).

## The site

It's one self-contained file — `index.html` — with no external dependencies
(all CSS, JS, and the hand-built SVG charts are inline). Open it directly, or view it
live on GitHub Pages.

## Publishing

Two ways, both land on the **`gh-pages`** branch that Pages serves from:

- **Automatic** — push to `main` and the workflow in `.github/workflows/deploy.yml`
  redeploys `index.html` to `gh-pages`.
- **Manual** — publish the current tree to `gh-pages`:

  ```sh
  git subtree push --prefix . origin gh-pages   # if site is repo root
  ```

## Local preview

```sh
python3 -m http.server 8000   # then open http://localhost:8000
```
