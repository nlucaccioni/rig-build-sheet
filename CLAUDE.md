# Rig Build Sheet — Claude Code Context

This file orients Claude Code on the project. Deeper reference docs live alongside it:
- `ARCHITECTURE.md` — data model, code structure, how each feature works
- `DECISIONS.md` — chronological log of decisions made and why (including rejected approaches)

## What this is

A single-page tool for a personal project: planning and pricing out two sim rig builds (racing + flight) for the user's father. Tracks parts with name, image, price, and store link across:
- **Racing Rig**
- **Flight Rig**
- **Shared Peripherals** (monitors, VR headset, KVM, mouse/keyboard, etc. — used by both rigs)
- **Alternates** (dream upgrades or cheaper swaps, not currently in the active build, that can be swapped in)
- **Full Build** — read-only combined total across all active parts

It's built as two self-contained HTML files — no build step, no framework, no dependencies to install. Started as a Claude.ai artifact, now living in a real GitHub repo (`nlucaccioni/rig-build-sheet`) hosted via GitHub Pages.

## Current state

- ✅ Core CRUD for parts, per rig, with images/price/links (`edit.html`)
- ✅ Auto price/name/image lookup from a pasted store link (no API keys — scrapes public schema.org/Open Graph data via free CORS proxies) (`edit.html`)
- ✅ Alternates tab with swap-into-active-build mechanic (`edit.html`)
- ✅ Full Build tab (combined total, active parts only) — in both files
- ✅ Dual storage in `edit.html`: `window.storage` when running as a Claude.ai artifact, falls back to `localStorage` otherwise
- ✅ Repo is **public** (confirmed with user: no secrets/API keys in the code, so public is fine) hosted via GitHub Pages
- ✅ **Editor/Viewer split (Option A) implemented** — see below

## Editor/Viewer split (shipped)

The tool is now two files instead of one, solving both "Dad shouldn't be able to edit" and "Dad's browser has no localStorage of his own":

- **`edit.html`** — the private editor, full CRUD as before, used only by the user. Has a **Publish** button that pushes the current `parts` array to `data.json` in this repo via the GitHub Contents API, authenticated with a personal access token the user provides (fine-grained PAT, scoped to just this repo, Contents: read/write only). The token lives only in `edit.html`'s own `localStorage` — never committed to the repo. Committed to the public repo itself (confirmed with user): safe because publishing requires a real token with push access to this specific repo, which only the user has — the code's visibility grants no capability by itself.
- **`index.html`** — the read-only viewer, the file GitHub Pages serves at the root and the link sent to Dad. Fetches the published `data.json` from `raw.githubusercontent.com` (no auth needed, repo is public) and renders it read-only — same visual style (rig tabs, Alternates read-only, Full Build totals) but **no add/edit/remove/swap UI at all in the shipped code**. Because the viewer literally contains no write capability, it doesn't matter how many people have the link.
- Tradeoff accepted: publishing is a manual step (click "Publish" after making changes) rather than instantly live. Small UX cost for real security.
- Full technical detail in `ARCHITECTURE.md` → "Publish flow"; decision record in `DECISIONS.md` point 9. Step-by-step setup (GitHub Pages hosting, generating the PAT, cloning for a different repo) lives in `SETUP.md`, not `README.md` — `README.md` is now the general, public-facing description of what the tool is, not a setup walkthrough.
- The user has generated a PAT, published successfully, and `data.json` exists in the repo with real build data — this is fully live, not just built.

## Known limitations to keep in mind / keep surfacing to the user

- **The auto price/image lookup (`fetchDetails`, in `edit.html`) is not perfectly reliable.** It scrapes public schema.org JSON-LD and Open Graph meta tags via free, third-party CORS proxies (no API keys, nothing paid). Manufacturer/retailer sites with good SEO (Fanatec, Thrustmaster, Logitech G, Honeycomb, VKB, WinWing, Best Buy) tend to work well. Amazon and other heavily bot-protected sites often fail (blocked proxy traffic, or price rendered via JS not present in raw HTML). Manual entry is always available as a fallback and the UI is explicit when lookup fails rather than guessing.
- **The "swap into rig" mechanic in Alternates matches by rig + category name** (case-insensitive, trimmed). If a user labels the same slot inconsistently (e.g. "Wheel Base" vs "Wheel-Base"), the swap won't find a match to demote and will just add the alternate as a second active part instead of replacing anything. Worth keeping category names consistent, or making this matching fuzzier in the future.
- **GitHub Pages + private repos:** confirmed with the user that GitHub Pages only publishes from private repos on paid plans (Pro/Team/Enterprise). Free personal accounts require a public repo for Pages. Since this codebase has no secrets, the user chose to make the repo public rather than pay or use a third-party host.
- **Publishing is manual, not automatic.** Editing in `edit.html` never changes what Dad sees until the Publish button is clicked — if the user reports "I made a change but the shared link still shows the old build," check whether they published.

## Conventions used in the code

- Vanilla JS, no framework, no build tooling — keep it that way unless the user asks to change the stack.
- Two self-contained HTML files (`edit.html`, `index.html`), each with `<style>` and `<script>` inline, no shared JS/CSS file between them — a deliberate choice to keep both independently "just open the file" simple, accepting some small constant/helper duplication in exchange (see `DECISIONS.md` point 9). If introducing a shared file or a build step, confirm with the user first.
- Dark, telemetry/dashboard-inspired visual theme with per-rig accent colors (racing = amber/red, flight = cyan, shared = purple, alternates = gold). See `ARCHITECTURE.md` for the full token system if extending the visual design.
