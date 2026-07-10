# Rig Build Sheet

A single-page tool for planning and pricing out sim racing and flight sim rig builds — parts, prices, images, and store links, with a dedicated space for shared peripherals and "alternate" parts (dream upgrades or cheaper swaps) you can slot into the active build later.

## Two files, two audiences

- **`edit.html`** — the full editor: add/edit/remove parts, swap alternates in, price/image lookup, and a **Publish** button. Private — this is for whoever is planning the build.
- **`index.html`** — a read-only viewer with the same look, no editing controls at all. This is what's shared with anyone just following along with the build (e.g. the person it's being built for).

The two are connected by a published `data.json` snapshot: nothing an editor changes is visible in the viewer until they click **Publish**.

## Features

- Separate tabs for **Racing Rig**, **Flight Rig**, and **Shared Peripherals**
- **Alternates** tab for dream/budget options, with one-click swap into the active build (editor only — read-only in the viewer)
- **Full Build** tab showing the combined total across everything currently active
- **Find price & image** button that reads a product page's own listing data (schema.org / Open Graph tags) — no API key required
- **Publish** button to push the current build to the public viewer

## Running it

No build step, no dependencies to install — just open the file.

- **To edit:** open `edit.html` in a browser.
- **To view:** open `index.html`.

## Setup, hosting, and cloning this for your own build

See [SETUP.md](SETUP.md) for hosting on GitHub Pages, one-time Publish setup (GitHub token), and what to change if you're cloning this repo for your own build.
