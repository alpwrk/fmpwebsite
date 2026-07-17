# fmpwebsite

The landing page for **fmp — Federated Music Protocol**. What fmp actually is,
how it works, and where it stands is explained on the site itself — read it
there, that's what the page is for.

## What's in this repo

| File | Purpose |
|---|---|
| `index.html` | The entire site — a single static page |
| `style.css` | The stylesheet |
| `fmpv1.md` | The whitepaper (linked from the site, licensed CC BY-SA 4.0) |

## Tech

Deliberately minimal:

- Plain HTML + CSS. No JavaScript, no frameworks, no web fonts, no build step,
  no dependencies.
- Brutalist monospace design, `72ch` max width, automatic dark mode via
  `prefers-color-scheme`.
- ASCII diagrams instead of images — the page loads with exactly two requests.

## Running it locally

There is nothing to build. Either open `index.html` directly in a browser, or
serve the directory:

```sh
python -m http.server
# → http://localhost:8000
```

## Deploying

It's static files — any static host works (GitHub Pages, nginx, a potato).
Copy the three files somewhere a web server can reach and you're done.

## Editing

- Content lives in `index.html`, one `<section>` per topic.
- Visual changes go in `style.css`; colors are CSS custom properties in
  `:root` (light) and the `prefers-color-scheme: dark` block.
- Keep the no-JS/no-dependency constraint — it's the point.
