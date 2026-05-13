# Degen Zone Redesign — Design Spec
**Date:** 2026-05-13
**Status:** Approved

## Goal

Replace the existing Degen Zone (Polymarket bot UI) with three focused tools:
1. A leveraged position calculator (client-side)
2. An OpenInsider trade screener (data via GitHub Actions → CSV in repo)
3. A CFTC COT signal report (data via GitHub Actions → GitHub Gist)

The ticker bar ($BTC/$ETH/$PEPE/$QUBIC) stays at the top. Everything else is replaced.

---

## Architecture

```
CFTCTrade Repo (GitHub Actions, every Friday 21:00 UTC)
  └── python -m src.main
        ├── sends email (unchanged)
        └── writes cot_data.json → GitHub Gist (public)   ← NEW

Cyberdashboard Repo (GitHub Actions, every Monday 07:00 UTC)
  └── python Openinsider_scrap/main.py
        └── commits Openinsider_scrap/output/trades.csv   ← NEW

index.html (browser, static)
  └── #page-trading (Degen Zone)
        ├── ticker-bar (unchanged)
        └── 3 Tabs
              ├── RECHNER  → pure JS, no fetch
              ├── INSIDER  → fetch('Openinsider_scrap/output/trades.csv')
              └── COT      → fetch(GIST_RAW_URL) → cot_data.json
```

---

## Part 1 — index.html Changes

### Remove
- `.bot-stats` grid (4 stat cards)
- `#positions-container` and surrounding `.bot-section`
- `#history-container` and surrounding `.bot-section`
- `.trading-cards` grid (Polymarket/TradingView/Hyperliquid/CoinMarketCap links)
- `#bot-api-status`
- All Polymarket bot JS: `renderPositionsTable()`, `renderHistoryTable()`, `fetchBotData()`, `setInterval(fetchBotData,30000)`, `API_BASE` constant
- CSS classes: `.bot-stats`, `.bot-stat`, `.bot-section`, `.bot-section-head`, `.bot-table-wrap`, `.bot-table`, `.whale-tag`, `.bot-api-status`, `.bot-loading`, `.bot-error`, `.tc`, `.trading-cards`

### Keep
- `.trading-page-header` (back button + title)
- `.ticker-bar` + `.ticker-update` + all ticker JS + `COINS` array

### Add
- Tab navigation bar directly below ticker
- Three tab panel divs: `#tab-rechner`, `#tab-insider`, `#tab-cot`
- Tab switching JS: `showTab(name)`
- New CSS for tabs, calculator, insider table, COT report

---

## Part 2 — Position Calculator (Tab: RECHNER)

### Inputs
| Field | ID | Example |
|---|---|---|
| Kontokapital ($) | `calc-kapital` | 10000 |
| Risiko (%) | `calc-risiko` | 2 |
| Entry-Preis ($) | `calc-entry` | 100 |
| Stop Loss ($) | `calc-sl` | 95 |

Long/Short toggle (`#calc-direction`): affects validation only (SL < Entry for Long, SL > Entry for Short).

### Outputs (live, recalculated on every input event)
| Output | Formula | Example |
|---|---|---|
| Hebel | `Entry / |Entry − SL|` | 20× |
| Margin | `Kapital × Risiko / 100` | $200 |
| Positionsgröße | `Margin × Hebel` | $4,000 |

### Validation
- SL ≤ 0 or Entry ≤ 0: show "Ungültiger Preis"
- Long: SL ≥ Entry → "Stop Loss muss unter Entry liegen"
- Short: SL ≤ Entry → "Stop Loss muss über Entry liegen"
- Any invalid state: outputs show "—"

---

## Part 3 — OpenInsider Tab (Tab: INSIDER)

### Data Source
File: `Openinsider_scrap/output/trades.csv` (relative to `index.html`)

Fetched via `fetch()` on tab open (cached until page reload). Parsed client-side with a minimal CSV parser.

### Table Columns
`Datum` · `Ticker` · `Company` · `Insider` · `Titel` · `Wert ($)` · `Δ Position` · `Grund`

Sorted by `value` descending. Max 50 rows shown.

Footer: `Zuletzt aktualisiert: {exported_at from last row}` · `{N} Trades nach Filterung`

### Error States
- Fetch fails: "⚠ DATEN NICHT VERFÜGBAR — trades.csv nicht gefunden"
- Empty CSV: "Keine Trades nach Filterung"
- Loading: "LADE INSIDER-DATEN..."

### GitHub Actions Workflow
New file: `.github/workflows/openinsider-screener.yml`

```yaml
name: OpenInsider Screener
on:
  schedule:
    - cron: '0 7 * * 1'   # Every Monday 07:00 UTC
  workflow_dispatch:
jobs:
  run-screener:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install -r Openinsider_scrap/requirements.txt
      - run: playwright install chromium --with-deps
      - run: python Openinsider_scrap/main.py
      - name: Commit updated trades.csv
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add Openinsider_scrap/output/trades.csv
          git diff --staged --quiet || git commit -m "chore: update insider trades [skip ci]"
          git push
```

---

## Part 4 — COT Report Tab (Tab: COT)

### Data Source
Public GitHub Gist, URL stored as constant `COT_GIST_URL` in `index.html`.

Fetched on tab open. JSON schema (from existing CFTCTrade spec):

```json
{
  "generated_at": "2026-05-09T21:00:00Z",
  "report_date": "2026-05-09",
  "signal_count": 1,
  "buy_signals": [
    {
      "commodity": "Crude Oil (WTI)",
      "etf": "UCO",
      "hold_until": "2026-06-06",
      "commercial": { "current_long": 933442, "prior_52w_max": 903255, "report_date": "2026-05-05" },
      "trend": { "latest_close": 95.42, "sma50": 94.48, "sma200": 69.79 }
    }
  ],
  "full_status": [
    { "commodity": "Gold", "commercial_signal": false, "trend_signal": false, "combined": false }
  ]
}
```

### Display
**Active BUY signals** (if `signal_count > 0`): one card per signal with green glow, commodity name, ETF badge, hold-until date, commercial + trend detail table.

**Full status table**: all commodities, columns: Commodity · Commercial · Trend · Combined (✅/❌).

**Footer**: `COT Report: {report_date}` · `Generiert: {generated_at}`

**Fallback**: "Keine Buy-Signale diese Woche" + full status table still shown.

### Error States
- Fetch fails: "⚠ COT-DATEN NICHT VERFÜGBAR" + retry button
- Loading: "LADE COT-SIGNALE..."

### CFTCTrade Repo Changes
Three files to add/modify in `cfandl-stack/CFTCTrade`:

**`src/json_output.py`** (new): `build_cot_json(actionable, all_alerts) -> dict` — maps Alert dataclasses to the JSON schema above.

**`src/main.py`** (modified): after `send_alert_email(...)`, call `build_cot_json()` and write `cot_data.json` to working directory.

**`.github/workflows/cot_alert.yml`** (modified): new step after "Run COT alert script":
```yaml
- name: Update Gist
  env:
    GIST_TOKEN: ${{ secrets.GIST_TOKEN }}
    GIST_ID: ${{ secrets.GIST_ID }}
  run: gh gist edit $GIST_ID cot_data.json
  continue-on-error: true
```

---

## One-Time Setup (manual)

1. Create public Gist: `gh gist create --public --filename cot_data.json` with placeholder `{}` → note the Gist ID
2. Create GitHub PAT with `gist` scope at github.com/settings/tokens
3. Add secrets to `cfandl-stack/CFTCTrade`: `GIST_TOKEN` + `GIST_ID`
4. Add `COT_GIST_URL` constant to `index.html` (raw Gist URL)
5. Add `.superpowers/` to `.gitignore` in dashboard repo

---

## Affected Repos

| Repo | Changes |
|---|---|
| `cfandl-stack/Cyberdashboard` | `index.html` rewrite (Degen Zone), new `.github/workflows/openinsider-screener.yml` |
| `cfandl-stack/CFTCTrade` | New `src/json_output.py`, modify `src/main.py` + `cot_alert.yml` |

---

## Error Handling Summary

| Scenario | Behavior |
|---|---|
| Gist update fails in workflow | `continue-on-error: true` — email still sent |
| Dashboard Gist fetch fails | "⚠ COT-DATEN NICHT VERFÜGBAR" + retry button |
| trades.csv not found | "⚠ DATEN NICHT VERFÜGBAR" message |
| trades.csv empty | "Keine Trades nach Filterung" |
| Invalid calculator inputs | Outputs show "—" + inline validation message |
| OpenInsider script finds 0 trades | Empty CSV → empty state message |

---

## Out of Scope (for now)
- Trade Journal (explicitly deferred by user)
- Automatic CSV refresh interval in dashboard (manual re-open of tab is sufficient)
- Authentication for Gist (public Gist is sufficient)
