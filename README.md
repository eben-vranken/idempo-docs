# idempo-docs

Documentation site for [**idempo**](https://github.com/eben-vranken/idempo), the
framework-agnostic HTTP idempotency middleware for Go. Built with
[MkDocs](https://www.mkdocs.org/) and the
[Material theme](https://squidfunk.github.io/mkdocs-material/), and deployed to
GitHub Pages.

The content mirrors the idempo library source — the README, the `idempo.go`
middleware, the `Store` interface doc comments, and the in-memory / Redis /
Postgres backends — and is the canonical target for the RFC 9457 error `type`
URLs the middleware emits (`/errors/<code>/`).

## Run it locally

You need Python 3. Install the dependencies and start the live-reloading dev
server:

```sh
pip install -r requirements.txt
mkdocs serve
```

Then open <http://127.0.0.1:8000/>. The server rebuilds on every save.

To produce a static build (what CI deploys), run:

```sh
mkdocs build --strict
```

`--strict` turns warnings (such as a broken internal link) into errors, matching
the CI build.

## Project layout

```
mkdocs.yml                 # site config, theme, and nav
requirements.txt           # mkdocs-material
docs/
  index.md                 # Overview
  getting-started.md
  configuration.md
  behavior.md
  storage/                 # in-memory, Redis, Postgres, custom backend
  errors/                  # one page per error type → /errors/<code>/
.github/workflows/deploy.yml   # build + deploy to GitHub Pages
```

`mkdocs.yml` sets `use_directory_urls: true`, so pages resolve at clean paths
like `/errors/conflict/`.

## How Pages deployment works

`.github/workflows/deploy.yml` runs on every push to `main` (and can be
triggered manually from the Actions tab). It:

1. checks out the repo and sets up Python,
2. installs `requirements.txt`,
3. runs `mkdocs build --strict` into `site/`,
4. uploads `site/` as a Pages artifact and deploys it with
   `actions/deploy-pages`.

The workflow uses the official GitHub Pages deployment flow (the
`upload-pages-artifact` / `deploy-pages` actions with `pages: write` and
`id-token: write` permissions). No `gh-pages` branch is involved.

### One-time repo setup

In the repository settings, under **Settings → Pages**, set **Source** to
**GitHub Actions**. After that, every push to `main` publishes the site to
`https://eben-vranken.github.io/idempo-docs/`.

> **Note on the site URL.** `mkdocs.yml` sets
> `site_url: https://eben-vranken.github.io/idempo-docs/`, which assumes this is
> a *project* Pages site served under the `/idempo-docs/` path. If you instead
> serve it from a custom domain or a user/organization site (served at the
> domain root), update `site_url` to match so canonical links and the sitemap
> are correct.
