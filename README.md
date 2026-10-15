# kubeseal-ui docs

Jekyll site for the kubeseal-ui documentation, published to GitHub Pages at
**https://kubeseal-ui.github.io/** (the repository is the user site
`kubeseal-ui.github.io`, so pages serve at the root).

The theme is a custom layout in `_layouts/default.html` plus
`assets/css/site.css`: `theme: null` in the config, so there is no theme gem and
no remote dependency, and no build step beyond the standard `jekyll-build-pages`
action. Navigation lives in `_data/`.

The palette is the product's, not this site's: every `--app-*` value in
`site.css` is a copy of the same token in `frontend/src/style.css`, and the names
are kept so a change there can be found here by the same word. Add a colour by
copying it from the app, never by picking one here.

## Structure

```text
_config.yml            Jekyll config (theme: null, markdown, permalinks)
_data/nav.yml          Sidebar navigation (order, groups, labels)
_layouts/default.html  Page shell: header, sidebar, content, footer; theme script
assets/css/site.css    The product's design tokens + component styles
*.md                   The documentation pages (front matter: title, description)
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

Colour and geometry belong in `site.css`, as tokens — a rule that hard-codes a hex
value is a second copy of the palette that no longer tracks the app. The theme
control offers light, dark, and system, and stores that *preference* under
`kubeseal-ui-theme`; `data-theme` on the root element is always the *resolved*
theme, which is what the stylesheet keys on.

## Verify locally

```bash
bundle exec jekyll serve
# or: docker run -p 4000:4000 -v "$PWD:/site" jekyll/jekyll jekyll serve
```

CI builds on push to `main` via `.github/workflows/pages.yml` (GitHub Pages,
Source: GitHub Actions).
