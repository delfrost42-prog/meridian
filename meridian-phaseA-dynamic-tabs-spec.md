# Build spec — Phase A: convert 3 static tabs to dynamic

## For Claude Code — read this first

This converts **three** hardcoded tabs in `research_terminal.html` to load live data from Google Sheets, through the Cloudflare Worker — exactly like the Unicorn Watchlist tab that's already working. The three tabs:

1. **Institutional Investors**
2. **Political Holdings**
3. **Biotech**

This is the **same proven pattern**, repeated three times. The Unicorn tab is your reference implementation — match it.

Working rules (unchanged from the Unicorn job):
- The owner is **not a professional developer**. Explain steps in plain language.
- Work **one tab at a time**, fully finishing and testing each before starting the next. Do NOT batch all three blindly.
- **The worker is LIVE and the owner's Claude connector depends on it.** Do NOT modify or remove existing routes (`/`, `/mcp`, `/sse`), any existing tool, the JSON-RPC handling, the `CORS` object, or `sheetsGet`/`sheetsPost`. **Only ADD.**
- **Keep the worker in its existing service-worker format** (`addEventListener('fetch')`, `var`, global secrets `APPS_SCRIPT_URL` / `MERIDIAN_TOKEN`). Do NOT convert to ES modules.
- `MERIDIAN_TOKEN` stays server-side, inside `sheetsGet` only. It must never appear in the frontend or any response body.
- **Styling is load-bearing.** Reuse the existing HTML structure and CSS for each tab exactly — same `<td>` order, same classes, same markup. The dynamic table must render **indistinguishable** from the current static one. No restyling, no rebuilding.

You already added a read route in the Unicorn job: `GET /api/sheet?tab=<Tab>`, allowlisted. This spec **extends that same route's allowlist** — it does not create a new route.

---

## Pattern recap (what "convert a tab" means)

For each tab, the conversion is four steps:

1. **Read the current static tab** in `research_terminal.html`. Record its exact columns, row markup, and CSS classes. These define both the Sheet headers and the row template.
2. **Create the Sheet tab** (exact name from the table below), header row matching those columns, and **migrate the existing hardcoded rows** into it so nothing is lost. Generate a `.tsv` for the owner to paste in (headers in row 1, data below), the same way the Unicorns migration worked.
3. **Add the tab name to the worker's `/api/sheet` allowlist** and redeploy. Do not touch anything else in the worker.
4. **Convert the frontend**: replace that tab's hardcoded `<tbody>` rows with an empty `<tbody>` + JS that fetches `https://<worker-url>/api/sheet?tab=<Tab>`, reads `.rows`, and builds rows from a template that reproduces the existing markup exactly. Keep a loading state and a readable error row. Keep `filterTable()` working.

---

## The three tabs

Use these exact Sheet tab names (they become the `?tab=` value and the allowlist entry):

| Page tab | Sheet tab name | Notes |
|---|---|---|
| Institutional Investors | `Institutional` | Standard table migration. |
| Political Holdings | `Political` | Standard table migration. |
| Biotech | `Biotech` | Standard table migration. Likely has trial-stage / catalyst-date columns — preserve every column exactly as displayed. |

For each, the owner will paste the generated `.tsv` into the new Sheet tab at cell A1.

---

## Data-format caution (applies to all three)

Real data is messy. The rendering JS must **display whatever is in each cell as-is** — do not assume a value is a number, a date, or non-empty. Some cells will be blank, some will be strings like `"~$2.4B"`, `"Phase II"`, or `"TBD"`. Render them verbatim. A missing or oddly-formatted cell must never break the row or blank the tab. (This is the same robustness the Stock Tracker will need later, where `Price_At_Research` is sometimes a number, sometimes `"$30.44"`, sometimes `"TBD"`.)

---

## Per-tab acceptance criteria

For each of the three tabs, before moving to the next:

- `GET /api/sheet?tab=<Tab>` returns that tab's rows as JSON; no token in the response; tab names outside the allowlist are rejected with 403.
- `/mcp` still works (the Claude connector is unaffected) and `/` is unchanged.
- The tab renders from the Sheet and is **visually indistinguishable** from its old static version.
- Editing a row in the Sheet and reloading updates the tab.
- A worker/Sheet failure shows a readable message, not a blank or broken tab.
- `filterTable()` still works on the now-dynamic rows.

---

## Order of work

1. **Institutional** first (full four steps + test + owner confirms).
2. **Political** next (same).
3. **Biotech** last (same).

Stop after each tab for the owner to test before starting the next. If a tab's structure turns out to differ meaningfully from Unicorns, pause and describe what's different rather than guessing.

---

## Deploy reminder (owner does this)

Each worker change: Claude Code hands the owner the updated worker file; the owner opens the worker in the Cloudflare dashboard → **Edit code** → replaces the contents → **Deploy**. Keep `worker.backup.js` as the restore point. Because all three tabs use the **same** `/api/sheet` route, the worker only needs editing to extend the allowlist — ideally once, adding all three names (`Institutional`, `Political`, `Biotech`) at the same time, even though the frontend conversions happen one at a time.

---

## NOT in this spec (later phases)

- **Performance** and **Bubble Charts** — derive from existing Tickers data; separate spec once we confirm what they display.
- **News Feed** and **Options Chain** — live external API feeds, not Sheet migrations; separate effort.
- **Options Education** — intentionally stays hardcoded (embedded videos, links, teaching text). Do not convert.
- **Stock Tracker, IPO Calendar, Research Notes** — already backed by tools; their frontend conversion is a later, separate pass.
