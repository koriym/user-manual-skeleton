# Add Pagefind Search to Jekyll Documentation Site

This skill adds client-side full-text search to an existing Jekyll-based documentation site using [Pagefind](https://pagefind.app/).

## When to Use

- The site is built with Jekyll and served as static HTML (GitHub Pages, etc.)
- The site has multiple documentation pages (markdown) that users need to search
- You want a modern search UX (magnifier icon → modal) without external services (Algolia, etc.)

## Prerequisites

- Jekyll 4.x
- Node.js (for Pagefind CLI in CI and local development)
- A layout file that wraps the main content (e.g., `_layouts/docs-*.html`)

## Implementation Steps

### 1. Mark the content area for indexing

Add `data-pagefind-body` to the `<article>` or `<main>` element in your documentation layout. This tells Pagefind to index only the content, excluding navigation and sidebars.

```html
<!-- _layouts/docs-en.html -->
<article id="article" class="markdown-body" data-pagefind-body>
    {{ content }}
</article>
```

### 2. Add the search UI to the header

In your header include, load the Pagefind Component UI assets and place the modal trigger.

```html
<!-- _includes/header.html -->
{% if page.category == 'Manual' %}
<link href="/pagefind/pagefind-component-ui.css" rel="stylesheet">
<script src="/pagefind/pagefind-component-ui.js" type="module"></script>
{% endif %}
```

Place the trigger where you want the search icon (e.g., next to language switcher). Keep the row visible at every breakpoint so small screens can search too — only the language switcher is desktop-only, since small screens usually have their own switcher in the sidebar navigation.

```html
<ul class="navbar-nav ml-md-auto align-self-end align-self-md-auto d-flex flex-row language-list">
    <li class="nav-item">
        <pagefind-modal-trigger></pagefind-modal-trigger>
    </li>
    <!-- Language switcher: desktop only -->
    <li class="d-none d-md-block">
        <a href="{{ page.permalink | replace: '/ja/', '/en/' }}">En</a>
    </li>
    <li class="d-none d-md-block">
        <a href="{{ page.permalink | replace: '/en/', '/ja/' }}">Ja</a>
    </li>
</ul>
<pagefind-modal></pagefind-modal>
```

Notes on the responsive classes:

- `align-self-end` right-aligns the row on small screens, where a `navbar` collapses to `flex-column` and `ml-auto` has no effect on the cross axis.
- `align-self-md-auto` restores the default above the `md` breakpoint so the desktop layout is untouched.
- `d-flex flex-row` overrides `.navbar-nav`'s `flex-direction: column`.
- Do **not** add `align-items-center` to this list — it shifts the icon a few pixels out of line with the `En` / `Ja` links. Plain default alignment already matches them.

**Do not** append `{# ... #}` comments in Jekyll templates — that is Twig syntax, not Liquid, and it renders literally into the page. Use `<!-- ... -->` instead.

### 3. Customize the trigger to show only a magnifier icon

Add CSS to hide the default text and shortcut, showing only the icon:

```css
/* css/your-site.css */
pagefind-modal-trigger .pf-trigger-text,
pagefind-modal-trigger .pf-trigger-shortcut {
    display: none !important;
}

pagefind-modal-trigger {
    --pf-input-height: 1.5rem;  /* match your nav item height */
}

pagefind-modal-trigger .pf-trigger-btn {
    display: inline-flex !important;
    align-items: center !important;
    justify-content: center !important;
    border: none !important;
    background: transparent !important;
    padding: 0 0.5rem !important;
    box-shadow: none !important;
}

pagefind-modal-trigger .pf-trigger-btn:hover {
    background: rgba(0, 0, 0, 0.05) !important;
    border-radius: 0.25rem;
}
```

### 4. Configure Jekyll to keep the search index

Add `pagefind` to `keep_files` in `_config.yml` so Jekyll does not delete the index during watch rebuilds:

```yaml
# _config.yml
keep_files:
  - "pagefind"
```

### 5. Update the local development script

Build the site and generate the index once before starting the watch server:

```bash
#!/bin/bash
# bin/serve_local.sh
bundle exec jekyll build
if command -v npx >/dev/null 2>&1; then
  npx --yes pagefind@latest --site _site || echo "Warning: pagefind index generation failed; search disabled."
else
  echo "Note: npx not found; skipping Pagefind search index."
fi
bundle exec jekyll serve --watch
```

### 6. Add CI/CD workflow for GitHub Pages

Create `.github/workflows/pages.yml`:

```yaml
name: Deploy Jekyll site to Pages

on:
  push:
    branches: ["master"]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-22.04
    permissions:
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          persist-credentials: false

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Build with Jekyll
        run: bundle exec jekyll build --baseurl "${{ steps.pages.outputs.base_path }}"
        env:
          JEKYLL_ENV: production

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Build search index
        run: npx --yes pagefind@latest --site _site

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3

  deploy:
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-22.04
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### 7. Remove legacy full-page concatenation (optional)

If you have a script that generates a single-page manual (e.g., `merge_md_files.rb` → `1page.md`), consider removing it:

- Search replaces the need for humans to browse a giant concatenated page
- AI tools should use `llms.txt` / `llms-full.txt` instead
- Update navigation links and `.gitignore` accordingly

## Multilingual Support

Pagefind automatically detects the `<html lang>` attribute and builds separate indexes per language. Ensure your layouts set the correct `lang`:

```html
<html lang="{% if page.layout == 'docs-ja' %}ja{% else %}en{% endif %}">
```

This gives you zero-config language-separated search: Japanese pages search only Japanese content, English pages search only English content.

## Design Decisions

| Decision | Rationale |
|---|---|
| **Modal UI** | Same pattern as VitePress, Docusaurus, GitHub Docs |
| **Magnifier icon only** | Standard, compact, accessible (aria-label retained) |
| **Bold highlight** | Standard (Google, VitePress); background color optional |
| **No `1page.md`** | Search + `llms.txt` cover human and AI use cases better |
| **CI-only index generation** | Local dev generates once on serve start; CI regenerates on every deploy |
| **Responsive trigger** | Shown at all breakpoints; language switcher stays desktop-only to avoid duplication |

## Verification

1. Run `./bin/serve_local.sh`
2. Open a manual page in the browser
3. Confirm the magnifier icon appears in the header
4. Click it and search for a keyword
5. Confirm results are language-appropriate and matched words are bold
6. Repeat at a mobile viewport (e.g. 390x844) and confirm:
   - the icon is still visible and right-aligned,
   - the modal opens full width and search works,
   - the page has no horizontal scroll (`document.body.scrollWidth === document.body.clientWidth`).
