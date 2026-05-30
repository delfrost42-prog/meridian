# Build spec — convert 3 tool-backed tabs to live display

## For Claude Code — read this first

This converts **three** tabs in `index.html` from hardcoded HTML to live data, reading through the Cloudflare Worker's `/api/sheet?tab=<Tab>` route — exactly like the Unicorn / Institutional / Political / Biotech conversions already done.

**The key difference that makes these the easiest yet: there is NO Sheet migration.** These three tabs already have their data in the Sheet AND working MCP tools. Their data lives in the `Tickers`, `IPO`, and `Notes` Sheet tabs right now. So this is **frontend-only** plus one allowlist edit. Do not create Sheet tabs, do not migrate rows, do not write data.

The three tabs:

| Page tab | Existing Sheet tab | Existing MCP tools (do not touch) |
|---|---|---|
| Stock Tracker | `Tickers` | get_portfolio, add_ticker, update_ticker, delete_ticker |
| IPO Calendar | `IPO` | get_ipo_calendar, add_ipo, update_ipo, delete_ipo |
| Research Notes | `Notes` | get_research_notes, add_research_note |

Working rules (unchanged from prior conversions):
- The owner is **not a professional developer**. Explain steps plainly.
- Work **one tab at a time**; finish and let the owner test before the next.
- **The worker is LIVE and the owner's Claude connector depends on it.** Do NOT modify or remove existing routes, tools, JSON-RPC handling, `CORS`, or `sheetsGet`/`sheetsPost`. The ONLY worker change is adding three tab names to the `ALLOWED_TABS` allowlist (step below).
- **Keep the worker in its service-worker format.** No ES module conversion.
- `MERIDIAN_TOKEN` stays server-side. Never in the frontend or any response.
- **Styling is load-bearing.** Reuse each tab's existing HTML structure and CSS exactly — same columns, classes, markup. The dynamic version must render **indistinguishable** from the current static one. The live file is **`index.html`**.

---

## One-time worker change (do first)

Add all three Sheet-tab names to `ALLOWED_TABS`. The current line is:

```javascript
var ALLOWED_TABS = ['Unicorns', 'Institutional', 'Political', 'Biotech']
```

Change it to:

```javascript
var ALLOWED_TABS = ['Unicorns', 'Institutional', 'Political', 'Biotech', 'Tickers', 'IPO', 'Notes']
```

That is the ONLY worker edit. The owner will paste the updated worker into the Cloudflare dashboard (Edit code → replace → Deploy), keeping a backup. After deploy, the read routes for all three become available.

---

## Per-tab frontend conversion (repeat 3×)

For each tab, in `index.html`:

1. **Read the current static tab** and record its exact columns, row markup, and CSS classes — this is the row template to reproduce.
2. Replace the hardcoded `<tbody>` rows with an empty `<tbody>` + JS that fetches `https://<worker-url>/api/sheet?tab=<Tab>`, reads `.rows`, and builds one row per record using a template that reproduces the existing markup **exactly**.
3. Show a **loading** state while fetching and a readable **error** row on failure — never silently blank the tab.
4. Keep `filterTable()` (and any per-tab filters like the IPO status filter or Stock Tracker person filter) working on the dynamically built rows.
5. Verify by loading the live page: the tab shows its data and looks identical to before.

---

## CRITICAL: Stock Tracker has messy data — handle defensively

The `Tickers` data is real and inconsistent. The rendering JS must **display each cell's value exactly as-is** and never assume type or presence:
- `Price_At_Research` is sometimes a number (`24.5`), sometimes a string (`"$30.44"`), sometimes `"TBD"`.
- Some rows are missing fields entirely (e.g. COST has no Sector).
- Render whatever is there verbatim; blank if absent. A missing/oddly-typed cell must NEVER break the row or blank the tab.

Do NOT coerce, reformat, or validate these values — just display them. (Same robustness applies to all three tabs, but Stock Tracker is where it bites.)

---

## Order of work

1. **Research Notes** first — simplest structure, good warm-up.
2. **IPO Calendar** next — has a status filter to preserve.
3. **Stock Tracker** last — the messy-data one; take care with the defensive rendering above.

Stop after each for the owner to test before the next.

---

## Acceptance criteria (per tab)

- The tab renders from the Sheet via `/api/sheet?tab=<Tab>` and is visually indistinguishable from the old static version.
- `/mcp` still works (connector unaffected); `/` unchanged.
- Editing a row in the Sheet (or via an existing MCP tool) and reloading updates the tab.
- A worker/Sheet failure shows a readable message, not a blank/broken tab.
- Existing filters still work on the dynamic rows.
- Stock Tracker: messy/missing values render without breaking anything.

---

## NOT in this spec

- No Sheet migration, no new Sheet tabs, no data writing — the data is already there.
- No new MCP tools — these tabs already have them.
- Do not touch: Performance, Bubble Charts (derive from Tickers — separate), News Feed, Options Chain (live API feeds — separate), Options Education (stays hardcoded).
- Do not modify existing worker routes/tools/helpers beyond the single `ALLOWED_TABS` line.
