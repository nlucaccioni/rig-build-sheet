# Rig Build Sheet — Claude Code Context

This file orients Claude Code on the project. Deeper reference docs live in `docs/`:
- `docs/ARCHITECTURE.md` — data model, code structure, how each feature works
- `docs/DECISIONS.md` — chronological log of decisions made and why (including rejected approaches)

## What this is

A single-page tool for a personal project: planning and pricing out two sim rig builds (racing + flight) for the user's father. Tracks parts with name, image, price, and store link across:
- **Racing Rig**
- **Flight Rig**
- **Shared Peripherals** (monitors, VR headset, KVM, mouse/keyboard, etc. — used by both rigs)
- **Alternates** (dream upgrades or cheaper swaps, not currently in the active build, that can be swapped in)
- **Full Build** — read-only combined total across all active parts

It's built as a single self-contained `index.html` — no build step, no framework, no dependencies to install. Started as a Claude.ai artifact, now being moved to a real GitHub repo + Claude Code so it can be hosted on GitHub Pages and sent to the user's father as a link.

## Current state (as of handoff)

- ✅ Core CRUD for parts, per rig, with images/price/links
- ✅ Auto price/name/image lookup from a pasted store link (no API keys — scrapes public schema.org/Open Graph data via free CORS proxies)
- ✅ Alternates tab with swap-into-active-build mechanic
- ✅ Full Build tab (combined total, active parts only)
- ✅ Dual storage: `window.storage` when running as a Claude.ai artifact, falls back to `localStorage` otherwise (so it works both in Claude.ai and as a standalone file / GitHub Pages site)
- ✅ Repo scaffolding: `index.html`, `README.md`, `.gitignore` — intended to be a **public** repo (confirmed with user: no secrets/API keys in the code, so public is fine) hosted via GitHub Pages

## Immediate next step (not yet built — this is where to pick up)

**Problem to solve:** the user wants to send their dad a link to *view* the build, but doesn't want "anyone with the link" able to *edit* it. Right now the single `index.html` is fully editable by anyone who loads it, and since GitHub Pages is static hosting, there's no built-in access control.

**Two options were discussed with the user; they were about to choose when the session ended:**

### Option A — Editor + read-only Viewer split (recommended, genuinely secure)
- Keep the current full-featured tool as a private **editor** (e.g. `edit.html`), used only by the user.
- Add a "Publish" action to the editor that pushes a snapshot of the `parts` array to a `data.json` file in the repo via the **GitHub Contents API**, authenticated with a personal access token the user provides (fine-grained PAT, scoped to just this repo, Contents: read/write only). The token lives only in the editor's `localStorage` on the user's own machine — never committed to the repo.
- Build a separate, stripped-down **viewer** (`index.html`, the file GitHub Pages serves at the root, i.e. the link sent to Dad) that fetches the published `data.json` (via `raw.githubusercontent.com`, no auth needed since the repo is public) and renders it read-only — same visual style (rig tabs, Full Build totals) but **no add/edit/remove/swap UI at all in the shipped code**. Because the viewer literally contains no write capability, it doesn't matter how many people have the link — it's not a permissions check, it's an absence of the feature.
- Tradeoff: publishing is a manual step (click "Publish" after making changes) rather than instantly live. This is a small UX cost in exchange for actual security.

### Option B — Single page, PIN-gated editing (simpler, weaker — was explicitly flagged to the user as not real security)
- One page, same as today, but a PIN prompt gates the add/edit/remove UI.
- Trivially bypassable via browser dev tools or "view source" since the check runs client-side. Only useful as a deterrent against a casual accidental click, not against anyone who looks even slightly.

**The user leaned toward Option A during the conversation but had not given final confirmation before switching to Claude Code.** Confirm with them which direction to build before starting, unless they've already told you.

## Known limitations to keep in mind / keep surfacing to the user

- **localStorage is per-browser, per-device.** If the user builds out the whole rig on their own machine, their dad opening the same link on his computer sees a *blank* sheet — localStorage isn't shared between people or devices. This is exactly why the Option A viewer/editor split matters — it also solves this by giving Dad a page that reads *published* data rather than his own local storage.
- **The auto price/image lookup (`fetchDetails`) is not perfectly reliable.** It scrapes public schema.org JSON-LD and Open Graph meta tags via free, third-party CORS proxies (no API keys, nothing paid). Manufacturer/retailer sites with good SEO (Fanatec, Thrustmaster, Logitech G, Honeycomb, VKB, WinWing, Best Buy) tend to work well. Amazon and other heavily bot-protected sites often fail (blocked proxy traffic, or price rendered via JS not present in raw HTML). Manual entry is always available as a fallback and the UI is explicit when lookup fails rather than guessing.
- **The "swap into rig" mechanic in Alternates matches by rig + category name** (case-insensitive, trimmed). If a user labels the same slot inconsistently (e.g. "Wheel Base" vs "Wheel-Base"), the swap won't find a match to demote and will just add the alternate as a second active part instead of replacing anything. Worth keeping category names consistent, or making this matching fuzzier in the future.
- **GitHub Pages + private repos:** confirmed with the user that GitHub Pages only publishes from private repos on paid plans (Pro/Team/Enterprise). Free personal accounts require a public repo for Pages. Since this codebase has no secrets, the user chose to make the repo public rather than pay or use a third-party host.

## Conventions used in the code

- Vanilla JS, no framework, no build tooling — keep it that way unless the user asks to change the stack.
- Single HTML file with `<style>` and `<script>` inline — this was a deliberate choice for artifact/portability simplicity. If splitting into multiple files/a real build step, confirm with the user first since it changes the "just open index.html" simplicity they've valued so far.
- Dark, telemetry/dashboard-inspired visual theme with per-rig accent colors (racing = amber/red, flight = cyan, shared = purple, alternates = gold). See `docs/ARCHITECTURE.md` for the full token system if extending the visual design.
