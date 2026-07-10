# Architecture Reference

Deep-dive reference for `edit.html` and `index.html`. Read `CLAUDE.md` first for orientation; come here when you need to actually touch the code.

## File structure

```
edit.html        — the full editor: markup, styles, script, all CRUD + Publish. Private, used only by the owner.
index.html       — the read-only viewer: same visual style, no write-capable code at all. What GitHub Pages
                   serves at the root and the link sent to Dad.
data.json        — published snapshot of `parts`, written by edit.html's Publish button, read by index.html.
                   Does not exist in the repo until the first Publish.
README.md        — user-facing setup/hosting/publish instructions
gitignore        — minimal (OS cruft only)
CLAUDE.md        — Claude Code context (this doc's sibling)
ARCHITECTURE.md  — this file
DECISIONS.md     — chronological decision log
```

`edit.html` and `index.html` are two independent, fully self-contained files (each with its own inline `<style>`/`<script>`) — there is no shared JS/CSS file between them. This means constants like `THEME`/`RIG_ORDER` and small helpers (`money()`, `escapeHtml()`) are duplicated across both. That's a deliberate tradeoff: it's a small amount of duplication in exchange for each file staying independently self-contained (see `docs/DECISIONS.md` point 1).

## Data model

Everything lives in one in-memory array called `parts`, persisted as JSON (see Storage below). Each part is a plain object:

```js
{
  id: "p_xxxxxxx",        // generated via uid()
  category: "Wheel Base", // free text, but datalist-suggested per rig (see CATEGORIES)
  name: "Fanatec CSL DD",
  image: "https://...",   // direct image URL
  price: 599.99,          // number
  link: "https://...",    // store page URL
  rig: "racing",          // "racing" | "flight" | "shared"
  active: true            // true = in the active build; false = an Alternate
}
```

Notes:
- `active` defaults to `true` implicitly for any part saved before this field existed (`p.active !== false` is the check used everywhere, not `p.active === true`) — this keeps old saved data visible without a migration step.
- There is no separate "alternates" array — alternates are just parts with `active: false`. This keeps swap-in/out trivial (just flip the flag) and keeps one single source of truth.
- `category` + `rig` together define a "slot" for the swap mechanic (see below). Matching is case-insensitive and trimmed, but not fuzzy beyond that.

## Storage

`edit.html` (the editor) has two backends for its working copy, chosen automatically at load time:

```js
const IS_ARTIFACT_ENV = (typeof window.storage !== 'undefined' && window.storage !== null);
```

- **Inside a Claude.ai artifact:** `window.storage.get/set('sim-rig-parts', ..., false)` — the `false` means personal (non-shared) storage scoped to the user.
- **Everywhere else** (downloaded file, GitHub Pages, etc.): `localStorage.getItem/setItem('sim-rig-parts', ...)`.

Both paths are wrapped in try/catch in `loadParts()` / `saveParts()`; failures fall back to an empty array / a toast, never a thrown error.

**Important caveat:** `localStorage` is scoped per-browser, per-device (and per-origin/URL) — this is exactly why `edit.html`'s copy is never what Dad sees. What he sees is the separately-published `data.json` (see Publish flow below), fetched fresh by `index.html` on every load — `index.html` has no storage of its own at all.

## Publish flow (edit.html → data.json → index.html)

`edit.html` has a **Publish** button (`publishBuild()`) that pushes the current in-memory `parts` array to `data.json` in this repo via the GitHub Contents API:

1. **Token:** `getGithubToken()` reads a personal access token from `localStorage` (`rig-github-token`), prompting once via `prompt()` if absent. The token never touches the repo — it only ever lives in the editor's own browser storage. `changeGithubToken()` lets the user rotate it later.
2. **Read current sha:** `GET /repos/{owner}/{repo}/contents/data.json?ref={branch}` — GitHub's Contents API requires the file's current blob `sha` to update it (a 404 here just means "first publish," handled by omitting `sha` from the next call rather than erroring).
3. **Write:** `PUT /repos/{owner}/{repo}/contents/data.json` with a base64-encoded body (`b64EncodeUnicode()` — a UTF-8-safe wrapper around `btoa`, needed since part names can contain non-ASCII characters that plain `btoa` can't handle) and the `sha` from step 2 when updating an existing file.
4. **Result handling:** 401 clears the stored token (it's presumed invalid) and asks the user to re-enter it; 403 surfaces a permissions message; anything else surfaces a generic retry toast. On success, a `rig-last-published` timestamp is saved to `localStorage` and shown next to the Publish button.

`GITHUB_OWNER` / `GITHUB_REPO` / `GITHUB_BRANCH` / `DATA_PATH` are constants at the top of `edit.html`'s script — update these if the repo is ever renamed or forked.

`index.html` reads the published data with a plain, unauthenticated `fetch()`:

```js
const DATA_URL = `https://raw.githubusercontent.com/${GITHUB_OWNER}/${GITHUB_REPO}/${GITHUB_BRANCH}/${DATA_PATH}`;
```

A `?t=Date.now()` cache-buster is appended since `raw.githubusercontent.com` can serve a briefly stale cached copy otherwise. If the fetch fails or 404s (nothing published yet), the viewer shows a plain "No build has been published yet" message instead of an empty/broken page.

## View/nav state

Three pieces of global state drive what's rendered:

```js
let activeRig = "racing";      // which of racing/flight/shared is showing, only relevant when currentView === 'rig'
let currentView = "rig";       // 'rig' | 'alternates' | 'fullbuild'
let editingId = null;          // id of the part being edited, or null when adding
let modalIsAlternate = false;  // whether the open modal is adding/editing an Alternate vs an active part
```

Top-level nav functions (each clears tab-active state, swaps the `--accent`/`--accent-dim` CSS vars for the rig's theme color, swaps the ribbon decoration, and shows/hides the right view container):

- `setRig(rig)` → shows `#rig-view`, calls `render()`
- `showAlternatesView()` → shows `#alternates-view`, calls `renderAlternates()`
- `showFullBuild()` → shows `#fullbuild-view`, calls `renderFullBuild()`

## Rendering functions

Both files implement `render()` / `renderAlternates()` / `renderFullBuild()` with the same filtering logic; `index.html`'s versions are stripped of every write-capable element (no dashed "Add" card, no edit/remove/swap/demote buttons — just the card, its data, and the store link).

- `render()` — the main per-rig grid. Filters `parts` to `p.rig === activeRig && p.active !== false`. Shows Total/Count gauges, then part cards (in `edit.html`, plus an "Add Part" dashed card).
- `renderAlternates()` — filters `parts` to `p.active === false` (across all rigs). Each card shows a rig-colored badge; in `edit.html` also a "⇄ Swap into rig" button (see Swap mechanic below) and an "Add Alternate" dashed card. In `index.html` the card is informational only.
- `renderFullBuild()` — filters `parts` to `p.active !== false`, grouped by `RIG_ORDER = ["racing","flight","shared"]`, one column per rig with its own subtotal, plus a grand total banner across all three. **Alternates are deliberately excluded** from this total since they're not real spend yet. Identical in both files.

The remaining sections below — Modal, Swap mechanic, Auto price/image lookup — describe `edit.html` only; none of that code exists in `index.html`.

## Modal (Add/Edit) — edit.html only

One shared modal (`#overlay` / `.modal`) handles four flows, distinguished by `modalIsAlternate` and `editingId`:

| Function | modalIsAlternate | editingId | Effect on save |
|---|---|---|---|
| `openAdd()` | false | null | new part, `rig: activeRig`, `active: true` |
| `openEdit(id)` | false | id | update existing active part |
| `openAddAlternate()` | true | null | new part, `rig: <#f-altrig value>`, `active: false` |
| `openEditAlternate(id)` | true | id | update existing alternate, rig editable via `#f-altrig` |

`populateDatalist(rig)` refreshes the category `<datalist>` suggestions to match `CATEGORIES[rig]`. When adding/editing an Alternate, the "For rig" `<select id="f-altrig">` is shown, and its `onchange` handler re-calls `populateDatalist()` so suggestions stay relevant to the currently chosen rig.

`savePart()` reads all the form fields, builds a `data` object, branches on `modalIsAlternate` for `rig`/`active`, then either merges into the existing part (`editingId` set) or pushes a new one. Calls `renderAlternates()` or `render()` depending on which flow was active.

## Swap mechanic (Alternates ↔ active build) — edit.html only

`swapAlternateIn(id)`:
1. Finds the alternate part by id.
2. Looks for an existing **active** part with the same `rig` and same `category` (case-insensitive, trimmed) — takes the *first* match only.
3. Sets the alternate's `active = true`.
4. If a match was found, sets *that* part's `active = false` (demoting it to an Alternate) — a clean one-for-one swap.
5. If no match, just activates the alternate with no demotion (toast reflects which case happened).

`demoteToAlternate(id)` is the reverse manual action, available from the main rig grid — sets `active = false` on an active part without deleting it, so it reappears in the Alternates tab.

## Auto price/image lookup (`fetchDetails`) — edit.html only

No API keys, no Anthropic API dependency (removed — see `docs/DECISIONS.md` for why). Pure client-side scraping:

1. `fetchPageHtml(url)` tries a chain of free, keyless CORS proxies (`CORS_PROXIES` array — currently `corsproxy.io`, `api.allorigins.win`, `api.codetabs.com`) in order, each with a timeout, until one returns usable HTML (`fetchWithTimeout`).
2. `parseProductFromHtml(html, baseUrl)` parses the HTML with `DOMParser` and looks for, in priority order:
   - `schema.org` JSON-LD (`<script type="application/ld+json">`) with `@type: "Product"` — reads `name`, `image` (string, array, or `{url}` object), and `offers.price`/`offers.lowPrice`.
   - Falls back to Open Graph / Twitter meta tags (`og:image`, `og:title`, `og:price:amount`, `itemprop="price"`, etc.) for whatever JSON-LD didn't supply.
3. Image URLs are resolved to absolute via `absUrl()`. Price strings are stripped of currency symbols/commas and parsed as a float.
4. UI (`setFetchStatus`) always tells the user what happened — found price/image/name, or a clear "couldn't find/reach" message — never silently guesses or fails silently.

Known weak points: sites that render price via client-side JS (nothing in the raw HTML), and sites that actively block proxy/bot traffic (Amazon is the most common failure case). This is inherent to the no-API-key/no-server-backend constraint and has been communicated clearly to the user as an expected tradeoff.

## Visual design tokens

Dark, telemetry/dashboard-inspired theme. Key CSS custom properties on `:root`, swapped at the `body` level per active view:

```
--racing: #e8542a   (amber/red)
--flight: #4fb8e8   (cyan)
--shared: #a78bfa   (purple)
--alt:    #e8c547   (gold, for Alternates)
```

Each has a `-dim` variant for borders/backgrounds. `--accent`/`--accent-dim` are the "current" pair, swapped via `document.body.style.setProperty()` in each nav function. Typography: Rajdhani (display/headers), JetBrains Mono (numeric/technical values), Inter (body).
