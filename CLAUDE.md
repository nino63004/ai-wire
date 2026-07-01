# CLAUDE.md

Guidance for Claude Code (and other AI assistants) working in this repository.

## What this is

AI Wire is a single-file static web app: a live-updating feed of AI news. The
entire application — markup, styles, and logic — lives in one file:

- `index.html` — the whole app (HTML + inline `<style>` + inline `<script>`)

There is no build system, no package.json, no server, no framework, and no
test suite. Opening `index.html` directly in a browser (or serving it as a
static file) is the entire deployment story.

## How it works

1. On load, `init()` renders the category tabs and calls `fetchStories()`.
2. `fetchStories()` builds a prompt asking for the top 10 AI news stories
   (optionally scoped to a selected category) and calls `callClaude()`.
3. `callClaude()` sends a request directly from the browser to the Anthropic
   Messages API (`https://api.anthropic.com/v1/messages`), using model
   `claude-sonnet-4-20250514` with the built-in `web_search_20250305` tool so
   the model can search the web for current stories.
4. The model is instructed to return **only** a raw JSON array of story
   objects (`headline`, `summary`, `category`, `source`, `url`, `timestamp`,
   `urgent`). The response is parsed with a regex (`/\[[\s\S]*\]/`) to strip
   any stray text, then `JSON.parse`d.
5. Stories are stored in `allStories`, filtered by category/search text into
   `filteredStories`, and rendered into `.stories-grid` as cards.

State is all in-memory module-level variables (`allStories`,
`filteredStories`, `selectedCategory`, `isLoading`) — no framework, no state
library.

## The API key model (important, security-sensitive)

- The Anthropic API key is collected via a browser `prompt()` on first use
  and persisted in `localStorage` (`aiWireApiKey`).
- The key is sent as a client-side header (`x-api-key`) directly from the
  browser to `api.anthropic.com`. This means **any user's API key is fully
  exposed to that user's browser/devtools/localStorage**, and this pattern
  only works because Anthropic's API allows browser-based CORS requests.
- Do not "fix" this into a backend proxy unless explicitly asked — it's a
  deliberate (if security-naive) design choice for a client-only demo app.
  If asked to hide the key or add a backend, flag the tradeoff and confirm
  the user actually wants a server component before adding one.
- There is currently no way to clear/change a stored key from the UI other
  than clearing `localStorage` manually.

## Conventions

- **Single-file discipline**: keep everything in `index.html` unless the
  user explicitly asks to split into multiple files. Don't introduce a
  bundler, framework, or package.json speculatively.
- **Vanilla JS only**: no imports, no dependencies. Functions are attached
  via inline `onclick`/`oninput` handlers and global `function` declarations.
- **Styling**: CSS custom properties defined once in `:root` (see the color
  palette — cream/terra/brown/sage/rose/amber tokens). Reuse existing tokens
  (`var(--terra)`, `var(--brown-light)`, etc.) instead of hardcoding new
  colors. Fonts are `Playfair Display` (serif, headings) and `Inter` (sans,
  body), loaded from Google Fonts.
- **XSS safety**: all story content rendered into the DOM goes through
  `escapeHtml()` first (see `renderStories()`). Any new code that injects
  model-provided or user-provided strings into `innerHTML` must do the same
  — the `story.url` used in the `href` is currently the one field *not*
  escaped, so be careful if extending link handling.
- **Section comment banners**: the script is organized with `// ── Section
  Name ─...` banner comments (Configuration, State, API Call, Fetch Stories,
  Filter Stories, Render Stories, UI Updates, Category selection, Init).
  Follow this pattern when adding new sections rather than scattering
  unrelated logic.
- Categories are defined once in the `CATEGORIES` array and drive both the
  tab UI and the prompt sent to the model — add new categories there rather
  than hardcoding strings elsewhere.

## Testing / verifying changes

There's no test suite or build step. To verify changes:

- Open `index.html` directly in a browser, or serve it locally, e.g.:
  ```bash
  python3 -m http.server 8000
  ```
  then visit `http://localhost:8000/index.html`.
- You'll need a real Anthropic API key (`sk-ant-...`) with web search enabled
  to exercise the live fetch; the prompt() dialog will ask for it on first
  load. Check the browser console/network tab for errors when debugging API
  or parsing failures.
- Since UI logic lives inline in `index.html`, manually test: category
  filtering, the search box, the refresh button/spinner state, and the
  empty/error states (e.g. by temporarily entering a bad API key).

## Git workflow

- Repository has two branches historically used: `main` and feature branches
  like `claude/claude-md-docs-j86r7u`. Commit history is small and linear
  (see `git log`); keep commit messages short and descriptive, consistent
  with existing messages (e.g. "Add API key handling for Claude API calls").
