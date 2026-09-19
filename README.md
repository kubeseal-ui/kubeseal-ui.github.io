# kubeseal-ui docs

Jekyll site for the kubeseal-ui documentation, published to GitHub Pages at
**https://kubeseal-ui.github.io/** (the repository is the user site
`kubeseal-ui.github.io`, so pages serve at the root).

The theme is a custom `jekyll-theme` layout in `_layouts/default.html` plus
`assets/css/site.css`: no remote theme dependency, no build step beyond the
standard `jekyll-build-pages` action. Navigation, theme tokens, and the docs
index live in `_data/`.

## Structure

```text
_config.yml            Jekyll config (theme, markdown, permalinks)
_data/nav.yml          Sidebar navigation (order, groups, labels)
_layouts/default.html  Page shell: header, sidebar, content, footer
assets/css/site.css    Design tokens + component styles (light/dark)
*.md                   The documentation pages (front matter: title, nav order)
```

## Editing

Add a page: create `mypage.md` with front matter and register it in
`_data/nav.yml`:

```markdown
---
title: My page
description: One-line summary for the index.
---
content
```

The sidebar order follows `nav.yml`; pages without a nav entry land under
"Reference".

## Verify locally

```bash
bundle exec jekyll serve
# or: docker run -p 4000:4000 -v "$PWD:/site" jekyll/jekyll jekyll serve
```

CI builds on push to `main` via `.github/workflows/pages.yml` (GitHub Pages,
Source: GitHub Actions).
