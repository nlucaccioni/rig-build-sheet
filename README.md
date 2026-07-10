# Rig Build Sheet

A single-page tool for planning and pricing out sim racing and flight sim rig builds — parts, prices, images, and store links, with a dedicated space for shared peripherals and "alternate" parts (dream upgrades or cheaper swaps) you can slot into the active build later.

## Two files, two audiences

- **`edit.html`** — the full editor: add/edit/remove parts, swap alternates in, price/image lookup. This is for you only. Never send this link to anyone you don't want editing the build.
- **`index.html`** — a read-only viewer with the same look, no editing controls at all. This is the file GitHub Pages serves at the root, and the link you send Dad.

The two are connected by a `data.json` file in this repo: `edit.html` has a **Publish** button that pushes a snapshot of your current build to `data.json` via the GitHub API; `index.html` fetches that file and renders it. Nothing is live until you click Publish.

## Features

- Separate tabs for **Racing Rig**, **Flight Rig**, and **Shared Peripherals**
- **Alternates** tab for dream/budget options, with one-click swap into the active build (editor only — read-only in the viewer)
- **Full Build** tab showing the combined total across everything currently active
- **Find price & image** button that reads a product page's own listing data (schema.org / Open Graph tags) — no API key required
- **Publish** button to push your current build to the public viewer

## Running it

No build step, no dependencies to install — just open the file.

- **To edit:** open `edit.html` in a browser (locally, or via `https://<your-username>.github.io/<repo-name>/edit.html` once hosted).
- **To view:** open `index.html`, or use the Pages link once hosted.

## Hosting on GitHub Pages

1. Push this repo to GitHub (see commands below).
2. In the repo, go to **Settings → Pages**.
3. Under **Build and deployment**, set **Source** to "Deploy from a branch."
4. Pick the `main` branch and the `/ (root)` folder, then **Save**.
5. GitHub will publish `index.html` at `https://<your-username>.github.io/<repo-name>/` within a minute or two — that's the read-only link to send Dad.

## Setting up Publish (one-time)

The editor needs a GitHub personal access token to push `data.json` on your behalf.

1. Go to [github.com/settings/personal-access-tokens/new](https://github.com/settings/personal-access-tokens/new) (fine-grained tokens).
2. Under **Repository access**, choose "Only select repositories" and pick this repo.
3. Under **Permissions → Repository permissions**, set **Contents** to **Read and write**. Leave everything else as "No access."
4. Generate the token and copy it.
5. Open `edit.html` and click **Publish** — you'll be prompted to paste the token once. It's saved only in your browser's local storage, never committed to the repo.

If the token is ever rejected or you want to rotate it, use the "change token" link next to the Publish button.

## A note on data storage

- Your working edits live in `edit.html`'s browser `localStorage`, on whatever device you're using to edit — same as before.
- Clicking **Publish** is what makes your current build visible to anyone with the viewer link (including Dad). Until you publish, changes are only visible to you.
- The viewer (`index.html`) has no storage of its own — it just fetches the latest published `data.json` every time it loads.

## Setup commands (from this folder)

```bash
git init
git add .
git commit -m "Initial commit: rig build sheet"
git branch -M main
git remote add origin https://github.com/<your-username>/<repo-name>.git
git push -u origin main
```
