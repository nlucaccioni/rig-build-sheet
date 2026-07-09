# Rig Build Sheet

A single-page tool for planning and pricing out sim racing and flight sim rig builds — parts, prices, images, and store links, with a dedicated space for shared peripherals and "alternate" parts (dream upgrades or cheaper swaps) you can slot into the active build later.

## Features

- Separate tabs for **Racing Rig**, **Flight Rig**, and **Shared Peripherals**
- **Alternates** tab for dream/budget options, with one-click swap into the active build
- **Full Build** tab showing the combined total across everything currently active
- **Find price & image** button that reads a product page's own listing data (schema.org / Open Graph tags) — no API key required
- Everything saves automatically to your browser's local storage

## Running it

This is a single self-contained HTML file — no build step, no dependencies to install.

- **Locally:** just open `index.html` in a browser.
- **On GitHub Pages:** see below.

## Hosting on GitHub Pages

1. Push this repo to GitHub (see commands below).
2. In the repo, go to **Settings → Pages**.
3. Under **Build and deployment**, set **Source** to "Deploy from a branch."
4. Pick the `main` branch and the `/ (root)` folder, then **Save**.
5. GitHub will publish the site at `https://<your-username>.github.io/<repo-name>/` within a minute or two — that's the link to send.

## A note on data storage

Parts are saved in the browser's `localStorage`, scoped to whoever's browser is viewing the page. That means:

- Your build data lives in *your* browser, on *your* device.
- If your dad opens the same link on his own computer, he'll see a blank build sheet, not your data — localStorage isn't shared between devices or people.
- If you want him to see the build you've put together, either build it out from a device he'll also use, or let me know and we can add an export/import feature so you can hand him a snapshot.

## Setup commands (from this folder)

```bash
git init
git add .
git commit -m "Initial commit: rig build sheet"
git branch -M main
git remote add origin https://github.com/<your-username>/<repo-name>.git
git push -u origin main
```
