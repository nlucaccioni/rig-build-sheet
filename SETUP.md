# Setup

Detailed setup/hosting steps — for the person maintaining this repo, or for reference if you're cloning it for your own build.

## Hosting on GitHub Pages

1. Push this repo to GitHub (see commands below if starting fresh).
2. In the repo, go to **Settings → Pages**.
3. Under **Build and deployment**, set **Source** to "Deploy from a branch."
4. Pick the `main` branch and the `/ (root)` folder, then **Save**.
5. GitHub will publish `index.html` at `https://<your-username>.github.io/<repo-name>/` within a minute or two — that's the read-only link to share.

`edit.html` is reachable the same way, at `.../edit.html` — keep that link private.

## Setting up Publish (one-time)

The editor needs a GitHub personal access token to push `data.json` on your behalf.

1. Go to [github.com/settings/personal-access-tokens/new](https://github.com/settings/personal-access-tokens/new) (fine-grained tokens).
2. Under **Repository access**, choose "Only select repositories" and pick this repo.
3. Under **Permissions → Repository permissions**, set **Contents** to **Read and write**. Leave everything else as "No access."
4. Generate the token and copy it.
5. Open `edit.html` and click **Publish** — you'll be prompted to paste the token once. It's saved only in your browser's local storage, never committed to the repo.

If the token is ever rejected or you want to rotate it, use the "change token" link next to the Publish button.

## A note on data storage

- Working edits live in `edit.html`'s browser `localStorage`, on whatever device you're using to edit.
- Clicking **Publish** is what makes the current build visible to anyone with the viewer link. Until then, changes are only visible to the editor.
- The viewer (`index.html`) has no storage of its own — it just fetches the latest published `data.json` every time it loads.

## Cloning this for your own build

The repo/owner/branch the Publish button pushes to (and the viewer fetches from) are hardcoded constants near the top of each file's `<script>`:

```js
const GITHUB_OWNER = 'nlucaccioni';
const GITHUB_REPO = 'rig-build-sheet';
const GITHUB_BRANCH = 'main';
```

Update these in both `edit.html` and `index.html` to point at your own fork/repo, then follow the hosting and Publish setup steps above.

## Setup commands (from this folder, starting a repo from scratch)

```bash
git init
git add .
git commit -m "Initial commit: rig build sheet"
git branch -M main
git remote add origin https://github.com/<your-username>/<repo-name>.git
git push -u origin main
```
