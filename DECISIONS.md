# Decision Log

Chronological record of the meaningful decisions made building this tool, including approaches that were tried and abandoned. Useful context for *why* the code looks the way it does, not just what it does.

## 1. Single self-contained HTML file, no framework, no build step

Started as a Claude.ai artifact (which only supports single HTML/React files anyway), and stayed that way deliberately even after moving toward GitHub Pages — it means the tool can always be opened by just double-clicking the file, no `npm install`, no build pipeline. Keep this constraint unless the user explicitly asks to change it.

## 2. Price/image lookup: started with the Anthropic API, moved to pure client-side scraping

**First approach:** used `fetch()` to `api.anthropic.com/v1/messages` with the `web_search_20250305` tool, asking the model to search for and return price/image/name as JSON.
- Worked fine *inside* the Claude.ai artifact sandbox (which proxies that specific call).
- **Broke once the file was downloaded and opened locally** — outside the artifact sandbox, that origin (`null` for a local file) gets a hard CORS rejection from `api.anthropic.com`. This was reported by the user via a console screenshot.
- Initial fix attempt: detect `IS_ARTIFACT_ENV` and disable/relabel the button outside Claude.ai with an explanatory message. This worked but meant the feature only worked in one environment.

**User asked directly: is there a way to do this without any API at all (open source / no API usage)?** This reframed the goal from "make the existing approach degrade gracefully" to "remove the dependency entirely."

**Final approach:** scrape the product page's own structured data client-side:
- Fetch raw HTML via a chain of free, keyless public CORS proxies (needed only because browsers block a page from reading another origin's HTML directly — this is a browser security rule, not something specific to Anthropic).
- Parse `schema.org` JSON-LD first (most reliable, meant for search engines), fall back to Open Graph/Twitter meta tags.
- No API keys, no server of ours involved, works identically in Claude.ai and standalone.

**One bug along the way:** an intermediate version tried a *direct* `fetch()` to the retailer's page plus an `allorigins.win` proxy fallback for images specifically (while still keeping the Anthropic API call for price). This broke the "Find price & image" button entirely, again due to artifact-sandbox CORS restrictions on arbitrary outbound domains. That intermediate step was reverted before rebuilding the fully-scraped, Anthropic-API-free version described above.

**Known limitation, communicated to the user repeatedly:** this approach can't reliably get prices from sites that render price via client-side JS (not present in raw HTML) or that actively block scraper/proxy traffic — Amazon is the most common failure case in practice. Manufacturer/retailer sites with good SEO metadata (Fanatec, Thrustmaster, Logitech G, Honeycomb, VKB, WinWing, Best Buy) tend to work well.

## 3. Image mismatch fix: prefer the page's own preview image, don't let search guess

Early version of the AI-search-based lookup once returned a wrong/mismatched image for a yoke. Root cause: asking the model to "find an image" without constraint let it return a generic/wrong-variant photo. Fixed (in the pre-scraping version) by explicitly instructing it to prefer the page's own social-share preview image and to return null rather than guess if it couldn't confirm a match. This constraint carried forward conceptually into the scraping approach, which now reads the actual page's own `og:image`/JSON-LD image directly rather than inferring one — a strictly better fix since it's not a guess at all.

## 4. Storage: environment detection, not a single universal method

`window.storage` (Claude.ai artifacts) and `localStorage` (everywhere else) are fundamentally different APIs with different scoping. Rather than picking one and breaking the other environment, the code detects which is available (`IS_ARTIFACT_ENV`) at load time and calls the matching one, with the same `parts` array as the in-memory model either way. This was driven directly by the user hitting a real CORS error after downloading the file and expecting their data to persist.

## 5. Alternates as `active: false`, not a separate array/list

Considered keeping alternates in a separate array from active parts. Rejected in favor of a single `parts` array with an `active` boolean, because:
- The swap mechanic (alternate ↔ active) is then just flipping a flag on existing objects, not moving data between two collections.
- One source of truth avoids sync bugs between two arrays.
- Old data saved before this feature existed has no `active` field at all — treating `undefined` as "active" (checking `!== false` rather than `=== true`) meant zero migration was needered for existing users' saved builds.

## 6. "Compare" renamed to "Full Build"

The user pointed out the tab wasn't actually comparing the two rigs against each other — it was showing the combined total of everything being bought. Renamed for accuracy, and the grand total copy was updated to explicitly note alternates are excluded, since that distinction now matters (it didn't before Alternates existed).

## 7. GitHub Pages hosting: public repo chosen over private + paid plan or third-party host

Confirmed via web search: GitHub Pages only publishes from *private* repos on paid plans (Pro/Team/Enterprise) — a free personal account requires the repo be public for Pages to work at all. Presented the user three options (make repo public / upgrade to Pro / host elsewhere like Netlify or Vercel while keeping the repo private). Since the codebase has no API keys or secrets in it (point 2 above resolved that dependency entirely), the user chose the simplest free option: a public repo.

## 8. Viewer/Editor split for access control (in progress at handoff)

Once the repo is public and hosted on GitHub Pages, the natural next question was: if the page can read *and* write shared data, "anyone with the link" and "the person editing" are the same set of people. The user asked specifically how to let their dad view the build without letting arbitrary visitors change it.

Two options were laid out:
- **Option A (recommended):** split into a private Editor (full CRUD, used only by the user, publishes snapshots to a `data.json` in the repo via the GitHub Contents API using the user's own personal access token, kept only in their browser) and a public read-only Viewer (the page GitHub Pages actually serves at the root — fetches the published `data.json` and renders it with zero write-capable code shipped). This is real security: the viewer can't be exploited into editing because the ability doesn't exist in its code, not because of a permission check that could be bypassed.
- **Option B:** a client-side PIN gate on the single existing page. Explicitly flagged to the user as *not* real security (trivially bypassed via dev tools/view-source), just a deterrent against an accidental click.

The user was leaning toward Option A but the session ended (switch to Claude Code) before final confirmation. **Confirm which direction before building** — see `CLAUDE.md` → "Immediate next step" for the full technical plan for Option A.

## 9. Option A implemented: editor/viewer split, no shared JS/CSS file

Confirmed with the user (in Claude Code) to proceed with Option A. Two sub-decisions were confirmed directly rather than assumed:

- **Alternates are shown in the viewer, read-only** (card + rig badge + store link, no swap button) rather than omitted — the user wanted Dad to see dream upgrades/budget options under consideration, not just the active build.
- **`edit.html` is committed to the public repo**, not gitignored. This is safe under the same reasoning as making the repo public in the first place (point 7): the file being visible grants no capability by itself. Publishing requires a real GitHub personal access token with push access to *this specific repo*, which only the user has — the code's visibility isn't the security boundary, GitHub's own token permissions are. Kept it committed (rather than local-only) so it's backed up and version-controlled like everything else.

**Implementation:**
- `index.html` (old, full-featured) renamed to `edit.html` via `git mv`, then extended with a **Publish** button that pushes the current `parts` array to `data.json` in this repo via the GitHub Contents API (GET for current `sha`, then PUT with base64-encoded content). The token is requested once via `prompt()` and stored in the editor's own `localStorage` (`rig-github-token`) — never written to any file.
- A new `index.html` was built from scratch as the read-only viewer: fetches `data.json` from `raw.githubusercontent.com` (cache-busted), and contains zero write-capable code — no modal, no `CATEGORIES`, no scraping/CORS-proxy code, no `localStorage` at all.
- **No shared JS/CSS file was introduced between the two.** Each stays fully self-contained per the project's original "no build step, no framework" convention (point 1) — a small amount of constant/helper duplication (`THEME`, `RIG_ORDER`, `money()`, `escapeHtml()`) was accepted in exchange for keeping "just open the file" true for both.
- Considered using `token <token>` (classic PAT header format) vs `Bearer <token>` for the GitHub API `Authorization` header — went with `Bearer`, which is what GitHub's current docs recommend and what fine-grained tokens expect.
