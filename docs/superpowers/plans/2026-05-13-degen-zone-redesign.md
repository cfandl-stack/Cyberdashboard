# Degen Zone Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the Polymarket bot UI in the Degen Zone with three tabbed tools: a position calculator, an OpenInsider screener table, and a CFTC COT signal report.

**Architecture:** The dashboard (`index.html`) is a single static HTML file — all new features are pure client-side JS or `fetch()` calls to static files/a public Gist. OpenInsider data flows via a new GitHub Actions workflow that commits `trades.csv` weekly. COT data flows via an existing GitHub Actions workflow in the CFTCTrade repo, extended to write a JSON file to a public GitHub Gist.

**Tech Stack:** Vanilla JS/HTML/CSS (dashboard), Python 3.11 (OpenInsider screener), GitHub Actions (automation), GitHub Gist (COT data transport)

---

## File Map

### Repo: `cfandl-stack/Cyberdashboard` (local path: this repo)

| File | Action | Responsibility |
|---|---|---|
| `index.html` | Modify | Remove Polymarket bot; add tabs, calculator, insider display, COT display |
| `.github/workflows/openinsider-screener.yml` | Create | Weekly run of OpenInsider script → commit trades.csv |
| `Openinsider_scrap/output/trades.csv` | Auto-generated | Produced by screener, read by dashboard |

### Repo: `cfandl-stack/CFTCTrade` (clone separately)

| File | Action | Responsibility |
|---|---|---|
| `src/json_output.py` | Create | `build_cot_json()` + `write_cot_json()` — maps Alert dataclasses to JSON |
| `src/main.py` | Modify | Call `write_cot_json()` after email send |
| `.github/workflows/cot_alert.yml` | Modify | Add Gist update step after script run |
| `tests/test_json_output.py` | Create | Unit tests for `build_cot_json()` |

---

## One-Time Setup (do before Task 9)

These are manual steps, not code:

1. Create a public Gist with a placeholder file named `cot_data.json`:
   ```bash
   echo '{}' > cot_data.json
   gh gist create --public --filename cot_data.json cot_data.json
   ```
   Note the Gist ID from the output URL (the long hex string).

2. Get the raw URL: `https://gist.githubusercontent.com/cfandl-stack/{GIST_ID}/raw/cot_data.json`

3. Create a GitHub Personal Access Token (PAT) with `gist` scope at https://github.com/settings/tokens

4. Add two secrets to the `cfandl-stack/CFTCTrade` repo (Settings → Secrets → Actions):
   - `GIST_TOKEN` = the PAT from step 3
   - `GIST_ID` = the Gist ID from step 1

---

## Task 1: Strip Polymarket Bot from index.html

**Repo:** `cfandl-stack/Cyberdashboard`

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Remove bot-related CSS**

In `index.html`, delete the entire block of CSS rules between `/* ── BOT STATS CARDS ────────────────────────── */` (line ~276) and `/* ── TICKER BAR ────────────────────────────── */` (line ~311). The classes to remove are: `.bot-stats`, `.bot-stat`, `.bot-stat-label`, `.bot-stat-value`, `.pnl-pos`, `.pnl-neg`, `.bot-section`, `.bot-section-head`, `.bot-table-wrap`, `.bot-table`, `.whale-tag`, `.bot-api-status`, `.bot-loading`, `.bot-error`, and the `.trading-cards`, `.tc`, `.tc-icon`, `.tc-name`, `.tc-desc`, `.tc-btn` block.

- [ ] **Step 2: Remove bot HTML from `#page-trading`**

Inside `<div id="page-trading">`, keep only:
- `.trading-page-header` div (back button + title)
- `.ticker-bar` div
- `<div class="ticker-update" id="ticker-update"></div>`

Delete everything else: `#bot-stats`, both `.bot-section` divs, `.trading-cards`, `#bot-api-status`.

Also update the `.tp-warn` text to:
```html
<div class="tp-warn">POSITION CALCULATOR // INSIDER SCREENER // COT SIGNALS</div>
```

- [ ] **Step 3: Remove bot JavaScript**

Delete from the `<script>` block:
- `const API_BASE = ...`
- `function renderPositionsTable(...)` (entire function)
- `function renderHistoryTable(...)` (entire function)
- `async function fetchBotData()` (entire function)
- `fetchBotData();`
- `setInterval(fetchBotData, 30000);`

- [ ] **Step 4: Verify page opens without errors**

Open `index.html` in a browser. Click DEGEN → button. Verify: ticker loads, no JS errors in console, no broken layout.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "refactor: remove Polymarket bot from Degen Zone"
```

---

## Task 2: Add Tab Navigation

**Repo:** `cfandl-stack/Cyberdashboard`

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add tab CSS to the `<style>` block**

Insert after the `/* ── TICKER BAR ────────────────────────────── */` section (after `.ticker-update` rule):

```css
/* ── DEGEN TABS ─────────────────────────────── */
.degen-tabs{
  display:flex;gap:2px;padding:4px;flex-shrink:0;
  border-bottom:1px solid var(--border);
}
.degen-tab{
  font-family:'Orbitron',sans-serif;font-size:.55rem;font-weight:700;
  letter-spacing:2px;padding:8px 20px;cursor:pointer;
  border:1px solid var(--border);background:transparent;
  color:var(--dim);transition:all .2s;
}
.degen-tab:hover{border-color:rgba(255,60,60,.4);color:var(--text);}
.degen-tab.active{
  border-color:var(--red);color:var(--red);
  background:rgba(255,60,60,.06);box-shadow:var(--glow-r);
}
.tab-panel{flex:1;overflow-y:auto;min-height:0;padding:16px;}
.tab-panel::-webkit-scrollbar{width:2px;}
.tab-panel::-webkit-scrollbar-thumb{background:var(--red);}
.degen-section-head{
  font-family:'Orbitron',sans-serif;font-size:.6rem;font-weight:700;
  color:var(--red);letter-spacing:3px;
  padding:8px 0 6px;border-bottom:1px solid rgba(255,60,60,.2);margin-bottom:16px;
}
```

- [ ] **Step 2: Add tab HTML inside `#page-trading`**

After `<div class="ticker-update" id="ticker-update"></div>`, insert:

```html
<!-- TABS -->
<div class="degen-tabs">
  <button class="degen-tab active" id="tab-btn-rechner" onclick="showTab('rechner')">⬡ RECHNER</button>
  <button class="degen-tab" id="tab-btn-insider" onclick="showTab('insider')">📊 INSIDER</button>
  <button class="degen-tab" id="tab-btn-cot" onclick="showTab('cot')">📡 COT</button>
</div>

<!-- TAB PANELS (content added in Tasks 3–5) -->
<div class="tab-panel" id="tab-rechner"></div>
<div class="tab-panel" id="tab-insider" style="display:none"></div>
<div class="tab-panel" id="tab-cot" style="display:none"></div>
```

- [ ] **Step 3: Add `showTab()` to the script block**

```javascript
// ─── DEGEN TABS ───────────────────────────────────────────
function showTab(name) {
  ['rechner','insider','cot'].forEach(t => {
    document.getElementById('tab-'+t).style.display = t===name ? 'block' : 'none';
    document.getElementById('tab-btn-'+t).classList.toggle('active', t===name);
  });
  if (name==='insider') loadInsiderData();
  if (name==='cot')     loadCotData();
}
```

- [ ] **Step 4: Verify tab switching works**

Open `index.html`, go to Degen Zone. Click all three tabs — each tab panel toggles, active tab is highlighted red, no JS errors.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add tab navigation to Degen Zone"
```

---

## Task 3: Position Calculator

**Repo:** `cfandl-stack/Cyberdashboard`

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add calculator CSS to `<style>` block**

Insert after the tab CSS added in Task 2:

```css
/* ── POSITION CALCULATOR ────────────────────── */
.calc-wrap{max-width:580px;margin:0 auto;}
.calc-direction{display:flex;gap:0;border:1px solid var(--border);margin-bottom:16px;}
.dir-btn{
  flex:1;padding:10px;font-family:'Orbitron',sans-serif;
  font-size:.55rem;font-weight:700;letter-spacing:2px;
  cursor:pointer;border:none;background:transparent;color:var(--dim);transition:all .2s;
}
.dir-btn.long.active{background:rgba(57,255,20,.1);color:var(--green);}
.dir-btn.short.active{background:rgba(255,60,60,.1);color:var(--red);}
.calc-grid{display:grid;grid-template-columns:1fr 1fr;gap:12px;margin-bottom:4px;}
.calc-field label{
  display:block;font-size:.5rem;color:var(--dim);
  letter-spacing:2px;margin-bottom:5px;
}
.calc-input{
  width:100%;background:rgba(0,245,255,.03);
  border:1px solid rgba(0,245,255,.2);color:var(--text);
  font-family:'Share Tech Mono',monospace;font-size:.95rem;
  padding:10px 12px;outline:none;transition:border-color .2s;
}
.calc-input:focus{border-color:var(--cyan);box-shadow:0 0 12px rgba(0,245,255,.08);}
.calc-input::-webkit-inner-spin-button,.calc-input::-webkit-outer-spin-button{-webkit-appearance:none;}
.calc-error{color:var(--red);font-size:.6rem;letter-spacing:1px;min-height:1.2em;margin:6px 0;}
.calc-results{display:grid;grid-template-columns:1fr 1fr 1fr;gap:12px;margin-top:12px;}
.calc-result{
  background:var(--panel);border:1px solid var(--border);
  padding:16px;text-align:center;
}
.calc-result-label{
  font-size:.48rem;color:var(--dim);
  letter-spacing:2px;text-transform:uppercase;margin-bottom:8px;
}
.calc-result-value{
  font-family:'Orbitron',sans-serif;font-size:1.3rem;
  font-weight:700;color:var(--cyan);letter-spacing:1px;
}
.calc-result.pos .calc-result-value{color:var(--green);text-shadow:var(--glow-g);}
```

- [ ] **Step 2: Fill `#tab-rechner` with calculator HTML**

Replace `<div class="tab-panel" id="tab-rechner"></div>` with:

```html
<div class="tab-panel" id="tab-rechner">
  <div class="calc-wrap">
    <div class="degen-section-head">// POSITIONS-RECHNER</div>
    <div class="calc-direction">
      <button class="dir-btn long active" id="dir-long" onclick="setDirection('long')">LONG</button>
      <button class="dir-btn short" id="dir-short" onclick="setDirection('short')">SHORT</button>
    </div>
    <div class="calc-grid">
      <div class="calc-field">
        <label>KONTOKAPITAL ($)</label>
        <input class="calc-input" type="number" id="calc-kapital" placeholder="10000" min="0" oninput="calcPositions()">
      </div>
      <div class="calc-field">
        <label>RISIKO (%)</label>
        <input class="calc-input" type="number" id="calc-risiko" placeholder="2" min="0" step="0.1" oninput="calcPositions()">
      </div>
      <div class="calc-field">
        <label>ENTRY-PREIS ($)</label>
        <input class="calc-input" type="number" id="calc-entry" placeholder="100" min="0" oninput="calcPositions()">
      </div>
      <div class="calc-field">
        <label>STOP LOSS ($)</label>
        <input class="calc-input" type="number" id="calc-sl" placeholder="95" min="0" oninput="calcPositions()">
      </div>
    </div>
    <div id="calc-error" class="calc-error"></div>
    <div class="calc-results">
      <div class="calc-result">
        <div class="calc-result-label">HEBEL</div>
        <div class="calc-result-value" id="result-hebel">—</div>
      </div>
      <div class="calc-result">
        <div class="calc-result-label">MARGIN</div>
        <div class="calc-result-value" id="result-margin">—</div>
      </div>
      <div class="calc-result pos">
        <div class="calc-result-label">POSITIONSGRÖSSE</div>
        <div class="calc-result-value" id="result-pos">—</div>
      </div>
    </div>
  </div>
</div>
```

- [ ] **Step 3: Add calculator JavaScript**

Add to the script block:

```javascript
// ─── POSITION CALCULATOR ──────────────────────────────────
let calcDirection = 'long';

function setDirection(dir) {
  calcDirection = dir;
  document.getElementById('dir-long').classList.toggle('active', dir==='long');
  document.getElementById('dir-short').classList.toggle('active', dir==='short');
  calcPositions();
}

function calcPositions() {
  const kapital = parseFloat(document.getElementById('calc-kapital').value);
  const risiko  = parseFloat(document.getElementById('calc-risiko').value);
  const entry   = parseFloat(document.getElementById('calc-entry').value);
  const sl      = parseFloat(document.getElementById('calc-sl').value);
  const errEl   = document.getElementById('calc-error');

  const clear = () => {
    document.getElementById('result-hebel').textContent  = '—';
    document.getElementById('result-margin').textContent = '—';
    document.getElementById('result-pos').textContent    = '—';
  };

  if (isNaN(kapital) || isNaN(risiko) || isNaN(entry) || isNaN(sl)) {
    errEl.textContent = ''; clear(); return;
  }
  if (entry <= 0 || sl <= 0 || kapital <= 0 || risiko <= 0 || risiko > 100) {
    errEl.textContent = 'Ungültiger Wert — alle Felder müssen > 0 sein'; clear(); return;
  }
  if (calcDirection === 'long' && sl >= entry) {
    errEl.textContent = 'Long: Stop Loss muss unter Entry liegen'; clear(); return;
  }
  if (calcDirection === 'short' && sl <= entry) {
    errEl.textContent = 'Short: Stop Loss muss über Entry liegen'; clear(); return;
  }

  errEl.textContent = '';
  const margin = kapital * risiko / 100;
  const hebel  = entry / Math.abs(entry - sl);
  const pos    = margin * hebel;
  const fmt    = n => '$' + n.toLocaleString('en-US', {minimumFractionDigits:2, maximumFractionDigits:2});

  document.getElementById('result-hebel').textContent  = hebel.toFixed(1) + '×';
  document.getElementById('result-margin').textContent = fmt(margin);
  document.getElementById('result-pos').textContent    = fmt(pos);
}
```

- [ ] **Step 4: Verify calculator logic manually**

Open `index.html` → DEGEN → RECHNER tab. Enter:
- Kapital: 10000 · Risiko: 2 · Entry: 100 · Stop Loss: 95 · Direction: LONG

Expected output: **Hebel: 20.0×** · **Margin: $200.00** · **Positionsgröße: $4,000.00**

Also test: SL = 105 with LONG → red error message. SL = 95 with SHORT → red error message.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add position calculator to Degen Zone (Rechner tab)"
```

---

## Task 4: OpenInsider Tab

**Repo:** `cfandl-stack/Cyberdashboard`

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add insider table CSS**

Insert into `<style>` block after calculator CSS:

```css
/* ── INSIDER TABLE ───────────────────────────── */
.insider-bar{
  display:flex;align-items:center;justify-content:space-between;margin-bottom:12px;
}
.insider-title{
  font-family:'Orbitron',sans-serif;font-size:.6rem;
  font-weight:700;color:var(--green);letter-spacing:3px;
}
.btn-g{background:rgba(57,255,20,.08);border-color:var(--green);color:var(--green);}
.btn-g:hover{background:rgba(57,255,20,.18);box-shadow:var(--glow-g);}
.insider-table-wrap{overflow-x:auto;}
.insider-table{width:100%;border-collapse:collapse;font-size:.68rem;}
.insider-table th{
  font-family:'Orbitron',sans-serif;font-size:.48rem;font-weight:700;
  color:var(--dim);letter-spacing:2px;text-transform:uppercase;
  text-align:left;padding:8px 10px;
  border-bottom:1px solid rgba(57,255,20,.15);background:rgba(57,255,20,.03);
}
.insider-table td{
  padding:7px 10px;border-bottom:1px solid rgba(255,255,255,.03);
  color:var(--text);white-space:nowrap;
}
.insider-table tr:hover td{background:rgba(57,255,20,.03);}
.insider-ticker{color:var(--green);font-weight:700;}
.insider-value{color:var(--green);font-weight:700;}
.insider-delta{color:var(--yellow);}
.insider-dim{color:var(--dim);font-size:.62rem;}
.insider-footer{
  font-size:.5rem;color:var(--dim);letter-spacing:1px;
  text-align:right;padding:6px 4px;
}
```

- [ ] **Step 2: Fill `#tab-insider` with HTML**

Replace `<div class="tab-panel" id="tab-insider" style="display:none"></div>` with:

```html
<div class="tab-panel" id="tab-insider" style="display:none">
  <div class="insider-bar">
    <span class="insider-title">// INSIDER SCREENER</span>
    <button class="btn btn-g" onclick="loadInsiderData(true)">↺ RELOAD</button>
  </div>
  <div id="insider-container"><div class="bot-loading">TAB ÖFFNEN ZUM LADEN…</div></div>
  <div class="insider-footer" id="insider-footer"></div>
</div>
```

- [ ] **Step 3: Add CSV parser and insider render functions to script block**

```javascript
// ─── CSV PARSER ───────────────────────────────────────────
function parseCSVLine(line) {
  const result = []; let cur = ''; let inQ = false;
  for (let i = 0; i < line.length; i++) {
    const ch = line[i];
    if (ch === '"')         { inQ = !inQ; }
    else if (ch === ',' && !inQ) { result.push(cur); cur = ''; }
    else                    { cur += ch; }
  }
  result.push(cur);
  return result;
}
function parseCSV(text) {
  const lines = text.trim().split('\n').filter(l => l.trim());
  if (lines.length < 2) return [];
  const headers = parseCSVLine(lines[0]);
  return lines.slice(1).map(line => {
    const vals = parseCSVLine(line);
    const obj = {};
    headers.forEach((h, i) => obj[h.trim()] = (vals[i] || '').trim());
    return obj;
  });
}

// ─── INSIDER TABLE ────────────────────────────────────────
let insiderLoaded = false;

function fmtValue(v) {
  const n = parseFloat(v);
  if (isNaN(n) || n === 0) return '–';
  if (n >= 1_000_000) return '$' + (n / 1_000_000).toFixed(1) + 'M';
  if (n >= 1_000)     return '$' + (n / 1_000).toFixed(0) + 'K';
  return '$' + n.toFixed(0);
}

function renderInsiderTable(rows) {
  let html = `<div class="insider-table-wrap"><table class="insider-table">
    <thead><tr>
      <th>DATUM</th><th>TICKER</th><th>COMPANY</th><th>INSIDER</th>
      <th>TITEL</th><th>WERT</th><th>Δ POS</th><th>GRUND</th>
    </tr></thead><tbody>`;
  rows.forEach(r => {
    const delta   = parseFloat(r.delta_own);
    const deltaStr= isNaN(delta) ? '–' : '+' + delta.toFixed(0) + '%';
    const company = (r.company||'').length > 28 ? r.company.substring(0,28)+'…' : (r.company||'–');
    html += `<tr>
      <td>${(r.trade_date||'').substring(0,10)}</td>
      <td><span class="insider-ticker">${r.ticker||'–'}</span></td>
      <td>${company}</td>
      <td>${r.insider||'–'}</td>
      <td class="insider-dim">${r.title||'–'}</td>
      <td class="insider-value">${fmtValue(r.value)}</td>
      <td class="insider-delta">${deltaStr}</td>
      <td class="insider-dim">${r.materiality_reason||'–'}</td>
    </tr>`;
  });
  html += '</tbody></table></div>';
  return html;
}

function loadInsiderData(force = false) {
  if (insiderLoaded && !force) return;
  const container = document.getElementById('insider-container');
  const footer    = document.getElementById('insider-footer');
  container.innerHTML = '<div class="bot-loading">LADE INSIDER-DATEN…</div>';
  footer.textContent  = '';

  fetch('Openinsider_scrap/output/trades.csv')
    .then(r => { if (!r.ok) throw new Error('HTTP ' + r.status); return r.text(); })
    .then(text => {
      const rows = parseCSV(text).filter(r => r.ticker);
      if (!rows.length) {
        container.innerHTML = '<div class="bot-loading">Keine Trades nach Filterung</div>';
        return;
      }
      const sorted = rows
        .sort((a,b) => parseFloat(b.value||0) - parseFloat(a.value||0))
        .slice(0, 50);
      container.innerHTML = renderInsiderTable(sorted);
      const lastRow = rows[rows.length - 1];
      footer.textContent = `Zuletzt aktualisiert: ${lastRow.exported_at||'–'} · ${rows.length} Trades nach Filterung`;
      insiderLoaded = true;
    })
    .catch(e => {
      container.innerHTML = `<div class="bot-error">⚠ DATEN NICHT VERFÜGBAR — trades.csv nicht gefunden (${e.message})</div>`;
    });
}
```

- [ ] **Step 4: Verify insider tab with local CSV**

Open `index.html` from a local web server (not `file://` — fetch requires HTTP). Use:
```bash
cd "C:/Users/Dell/Documents/Claude_Code_test/Code_privat/dashboard"
python -m http.server 8080
```
Then open http://localhost:8080. Go to Degen Zone → INSIDER tab. Should show 41 trades sorted by value, with Annovis Bio ($1.5M) and Blackrock ECAT ($4.2M) near the top.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add OpenInsider screener tab to Degen Zone"
```

---

## Task 5: COT Report Tab

**Repo:** `cfandl-stack/Cyberdashboard`

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add COT CSS to `<style>` block**

Insert after insider CSS:

```css
/* ── COT REPORT ──────────────────────────────── */
.cot-signal-card{
  background:rgba(57,255,20,.05);border:1px solid rgba(57,255,20,.3);
  box-shadow:0 0 12px rgba(57,255,20,.08);padding:20px;margin-bottom:12px;
}
.cot-signal-title{
  font-family:'Orbitron',sans-serif;font-size:.9rem;font-weight:700;
  color:var(--green);letter-spacing:3px;margin-bottom:6px;
}
.cot-etf{
  display:inline-block;padding:3px 10px;
  border:1px solid var(--green);color:var(--green);
  font-family:'Orbitron',sans-serif;font-size:.6rem;font-weight:700;
  letter-spacing:2px;margin-bottom:10px;
}
.cot-hold{font-size:.62rem;color:var(--dim);letter-spacing:1px;margin-bottom:12px;}
.cot-hold strong{color:var(--text);}
.cot-detail{width:100%;border-collapse:collapse;font-size:.65rem;}
.cot-detail td{padding:4px 8px;border-bottom:1px solid rgba(255,255,255,.04);}
.cot-detail td:first-child{color:var(--dim);letter-spacing:.5px;}
.cot-detail td:last-child{color:var(--text);text-align:right;}
.cot-no-signal{
  text-align:center;padding:32px;
  font-family:'Orbitron',sans-serif;font-size:.6rem;
  color:var(--dim);letter-spacing:2px;
}
.cot-status-table{width:100%;border-collapse:collapse;font-size:.68rem;margin-top:16px;}
.cot-status-table th{
  font-family:'Orbitron',sans-serif;font-size:.48rem;font-weight:700;
  color:var(--dim);letter-spacing:2px;text-align:left;padding:8px 10px;
  border-bottom:1px solid rgba(255,230,0,.15);background:rgba(255,230,0,.03);
}
.cot-status-table td{
  padding:7px 10px;border-bottom:1px solid rgba(255,255,255,.03);color:var(--text);
}
```

- [ ] **Step 2: Add `COT_GIST_URL` constant and fill `#tab-cot`**

At the top of the `<script>` block, add:

```javascript
const COT_GIST_URL = 'REPLACE_WITH_GIST_RAW_URL';
```

Replace `<div class="tab-panel" id="tab-cot" style="display:none"></div>` with:

```html
<div class="tab-panel" id="tab-cot" style="display:none">
  <div id="cot-container"><div class="bot-loading">TAB ÖFFNEN ZUM LADEN…</div></div>
</div>
```

- [ ] **Step 3: Add COT render and fetch functions to script block**

```javascript
// ─── COT REPORT ───────────────────────────────────────────
let cotLoaded = false;

function renderCotReport(data) {
  let html = '';

  if (data.buy_signals && data.buy_signals.length > 0) {
    data.buy_signals.forEach(sig => {
      const com = sig.commercial || {};
      const tr  = sig.trend      || {};
      html += `<div class="cot-signal-card">
        <div class="cot-signal-title">🟢 BUY — ${sig.commodity}</div>
        <div class="cot-etf">${sig.etf} (2× ETF)</div>
        <div class="cot-hold">Halten bis: <strong>${sig.hold_until}</strong> (4 Wochen)</div>
        <table class="cot-detail">
          <tr><td>Commercial Long (aktuell)</td><td>${(com.current_long||0).toLocaleString()}</td></tr>
          <tr><td>52W Prior Max</td>            <td>${(com.prior_52w_max||0).toLocaleString()}</td></tr>
          <tr><td>COT Report Datum</td>         <td>${com.report_date||'–'}</td></tr>
          <tr><td>Aktueller Kurs</td>           <td>$${tr.latest_close??'–'}</td></tr>
          <tr><td>50-Day SMA</td>               <td>$${tr.sma50??'–'}</td></tr>
          <tr><td>200-Day SMA</td>              <td>$${tr.sma200??'–'}</td></tr>
        </table>
      </div>`;
    });
  } else {
    html += '<div class="cot-no-signal">KEINE BUY-SIGNALE DIESE WOCHE</div>';
  }

  if (data.full_status && data.full_status.length > 0) {
    html += `<div class="degen-section-head" style="margin-top:16px">// FULL STATUS OVERVIEW</div>
      <table class="cot-status-table"><thead><tr>
        <th>COMMODITY</th><th>COMMERCIAL</th><th>TREND</th><th>COMBINED</th>
      </tr></thead><tbody>`;
    data.full_status.forEach(s => {
      const yn  = v => v ? '✅ YES' : '❌ NO';
      const buy = s.combined ? '<strong style="color:var(--green)">🟢 BUY</strong>' : '—';
      html += `<tr><td>${s.commodity}</td><td>${yn(s.commercial_signal)}</td><td>${yn(s.trend_signal)}</td><td>${buy}</td></tr>`;
    });
    html += '</tbody></table>';
  }

  const gen = data.generated_at ? data.generated_at.substring(0,16).replace('T',' ') : '–';
  html += `<div class="insider-footer">COT Report: ${data.report_date||'–'} · Generiert: ${gen} UTC</div>`;
  return html;
}

function loadCotData(force = false) {
  if (cotLoaded && !force) return;
  const container = document.getElementById('cot-container');
  container.innerHTML = '<div class="bot-loading">LADE COT-SIGNALE…</div>';

  fetch(COT_GIST_URL)
    .then(r => { if (!r.ok) throw new Error('HTTP ' + r.status); return r.json(); })
    .then(data => {
      container.innerHTML = renderCotReport(data);
      cotLoaded = true;
    })
    .catch(e => {
      container.innerHTML = `<div class="bot-error">⚠ COT-DATEN NICHT VERFÜGBAR (${e.message})<br>
        <button class="btn btn-r" style="margin-top:12px" onclick="loadCotData(true)">↺ RETRY</button></div>`;
    });
}
```

- [ ] **Step 4: Test COT tab with mock data (before Gist exists)**

Create a temp file `output/cot-test.json` with this content:
```json
{
  "generated_at": "2026-05-09T21:00:00Z",
  "report_date": "2026-05-09",
  "signal_count": 1,
  "buy_signals": [{"commodity":"Crude Oil (WTI)","etf":"UCO","hold_until":"2026-06-06","commercial":{"current_long":933442,"prior_52w_max":903255,"report_date":"2026-05-05"},"trend":{"latest_close":95.42,"sma50":94.48,"sma200":69.79}}],
  "full_status": [
    {"commodity":"Gold","commercial_signal":false,"trend_signal":false,"combined":false},
    {"commodity":"Silver","commercial_signal":false,"trend_signal":true,"combined":false},
    {"commodity":"Crude Oil (WTI)","commercial_signal":true,"trend_signal":true,"combined":true}
  ]
}
```

Temporarily change `COT_GIST_URL` to `'output/cot-test.json'`. Open http://localhost:8080, go to COT tab. Verify: green BUY card for Crude Oil, full status table with ✅/❌, footer with dates.

Then restore `COT_GIST_URL = 'REPLACE_WITH_GIST_RAW_URL'` and delete the temp file.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add COT report tab to Degen Zone"
```

---

## Task 6: OpenInsider GitHub Actions Workflow

**Repo:** `cfandl-stack/Cyberdashboard`

**Files:**
- Create: `.github/workflows/openinsider-screener.yml`

- [ ] **Step 1: Create workflow file**

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

      - name: Install Python dependencies
        run: pip install -r Openinsider_scrap/requirements.txt

      - name: Install Playwright browser
        run: playwright install chromium --with-deps

      - name: Run OpenInsider screener
        run: python Openinsider_scrap/main.py

      - name: Commit updated trades.csv
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add Openinsider_scrap/output/trades.csv
          git diff --staged --quiet || git commit -m "chore: update insider trades [skip ci]"
          git push
```

- [ ] **Step 2: Check Openinsider_scrap/requirements.txt has playwright**

```bash
cat Openinsider_scrap/requirements.txt
```

If `playwright` is missing, add it:
```
playwright
pandas
```

- [ ] **Step 3: Commit and push workflow**

```bash
git add .github/workflows/openinsider-screener.yml Openinsider_scrap/requirements.txt
git commit -m "ci: add weekly OpenInsider screener GitHub Actions workflow"
git push
```

- [ ] **Step 4: Trigger workflow manually to verify**

Go to https://github.com/cfandl-stack/Cyberdashboard/actions → select "OpenInsider Screener" → "Run workflow". Watch the run complete. Verify that a new commit appears updating `Openinsider_scrap/output/trades.csv`.

---

## Task 7: CFTCTrade — `src/json_output.py` with Tests

**Repo:** `cfandl-stack/CFTCTrade`  
**Pre-requisite:** Clone the repo locally: `gh repo clone cfandl-stack/CFTCTrade`

**Files:**
- Create: `src/json_output.py`
- Create: `tests/test_json_output.py`

- [ ] **Step 1: Write the failing tests first**

Create `tests/test_json_output.py`:

```python
from src.config import Commodity
from src.json_output import build_cot_json
from src.signals import Alert


def _alert(name: str, etf: str, commercial: bool, trend: bool,
           com: dict | None = None, tr: dict | None = None) -> Alert:
    commodity = Commodity(name, "000000", "XX=F", etf)
    return Alert(
        commodity=commodity,
        commercial_signal=commercial,
        trend_signal=trend,
        combined_signal=commercial and trend,
        commercial_details=com or {},
        trend_details=tr or {},
    )


class TestBuildCotJson:
    def test_top_level_keys_present(self):
        result = build_cot_json([], [])
        for key in ("generated_at", "report_date", "signal_count", "buy_signals", "full_status"):
            assert key in result

    def test_signal_count_matches_actionable(self):
        actionable = [_alert("Crude Oil (WTI)", "UCO", True, True)]
        result = build_cot_json(actionable, actionable)
        assert result["signal_count"] == 1

    def test_zero_signals(self):
        result = build_cot_json([], [_alert("Gold", "UGL", False, False)])
        assert result["signal_count"] == 0
        assert result["buy_signals"] == []

    def test_buy_signal_commodity_and_etf(self):
        alert = _alert("Crude Oil (WTI)", "UCO", True, True)
        result = build_cot_json([alert], [alert])
        sig = result["buy_signals"][0]
        assert sig["commodity"] == "Crude Oil (WTI)"
        assert sig["etf"] == "UCO"

    def test_buy_signal_commercial_fields(self):
        cd = {"current_value": 933442, "previous_max": 903255, "current_date": "2026-05-05"}
        alert = _alert("Crude Oil (WTI)", "UCO", True, True, com=cd)
        result = build_cot_json([alert], [alert])
        com = result["buy_signals"][0]["commercial"]
        assert com["current_long"] == 933442
        assert com["prior_52w_max"] == 903255
        assert com["report_date"] == "2026-05-05"

    def test_buy_signal_trend_fields(self):
        td = {"latest_close": 95.42, "sma_50": 94.48, "sma_200": 69.79}
        alert = _alert("Crude Oil (WTI)", "UCO", True, True, tr=td)
        result = build_cot_json([alert], [alert])
        tr = result["buy_signals"][0]["trend"]
        assert tr["latest_close"] == 95.42
        assert tr["sma50"] == 94.48
        assert tr["sma200"] == 69.79

    def test_full_status_length_matches_all_alerts(self):
        alerts = [
            _alert("Gold", "UGL", False, False),
            _alert("Crude Oil (WTI)", "UCO", True, True),
        ]
        result = build_cot_json([alerts[1]], alerts)
        assert len(result["full_status"]) == 2

    def test_full_status_signals_correct(self):
        alerts = [
            _alert("Gold", "UGL", False, False),
            _alert("Crude Oil (WTI)", "UCO", True, True),
        ]
        result = build_cot_json([alerts[1]], alerts)
        gold = next(s for s in result["full_status"] if s["commodity"] == "Gold")
        assert gold["commercial_signal"] is False
        assert gold["trend_signal"] is False
        assert gold["combined"] is False
        oil = next(s for s in result["full_status"] if s["commodity"] == "Crude Oil (WTI)")
        assert oil["combined"] is True

    def test_generated_at_parseable(self):
        from datetime import datetime
        result = build_cot_json([], [])
        datetime.strptime(result["generated_at"], "%Y-%m-%dT%H:%M:%SZ")

    def test_hold_until_is_4_weeks_from_today(self):
        from datetime import date, timedelta
        result = build_cot_json([_alert("Crude Oil (WTI)", "UCO", True, True)], [])
        hold = result["buy_signals"][0]["hold_until"]
        expected = (date.today() + timedelta(weeks=4)).isoformat()
        assert hold == expected
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd CFTCTrade
pytest tests/test_json_output.py -v
```

Expected: `ModuleNotFoundError: No module named 'src.json_output'`

- [ ] **Step 3: Implement `src/json_output.py`**

```python
from __future__ import annotations

import json
from datetime import date, datetime, timedelta

from src.config import HOLD_WEEKS
from src.signals import Alert


def build_cot_json(actionable: list[Alert], all_alerts: list[Alert]) -> dict:
    now = datetime.utcnow()
    hold_until = (date.today() + timedelta(weeks=HOLD_WEEKS)).isoformat()

    buy_signals = []
    for alert in actionable:
        cd = alert.commercial_details
        td = alert.trend_details
        buy_signals.append({
            "commodity": alert.commodity.name,
            "etf": alert.commodity.leveraged_etf,
            "hold_until": hold_until,
            "commercial": {
                "current_long": cd.get("current_value"),
                "prior_52w_max": cd.get("previous_max"),
                "report_date": cd.get("current_date"),
            },
            "trend": {
                "latest_close": td.get("latest_close"),
                "sma50": td.get("sma_50"),
                "sma200": td.get("sma_200"),
            },
        })

    full_status = [
        {
            "commodity": a.commodity.name,
            "commercial_signal": a.commercial_signal,
            "trend_signal": a.trend_signal,
            "combined": a.combined_signal,
        }
        for a in all_alerts
    ]

    return {
        "generated_at": now.strftime("%Y-%m-%dT%H:%M:%SZ"),
        "report_date": date.today().isoformat(),
        "signal_count": len(actionable),
        "buy_signals": buy_signals,
        "full_status": full_status,
    }


def write_cot_json(
    actionable: list[Alert],
    all_alerts: list[Alert],
    path: str = "cot_data.json",
) -> None:
    data = build_cot_json(actionable, all_alerts)
    with open(path, "w") as f:
        json.dump(data, f, indent=2)
```

- [ ] **Step 4: Run tests again — all should pass**

```bash
pytest tests/test_json_output.py -v
```

Expected: 10 tests PASSED

- [ ] **Step 5: Run full test suite to confirm no regressions**

```bash
pytest -v
```

Expected: all existing tests plus 10 new ones pass.

- [ ] **Step 6: Commit**

```bash
git add src/json_output.py tests/test_json_output.py
git commit -m "feat: add build_cot_json() to produce dashboard-ready JSON"
```

---

## Task 8: CFTCTrade — Modify `src/main.py`

**Repo:** `cfandl-stack/CFTCTrade`

**Files:**
- Modify: `src/main.py`

- [ ] **Step 1: Add import and call to `write_cot_json`**

Replace the entire content of `src/main.py` with:

```python
import logging
import sys

from src.config import load_email_config
from src.email_alert import send_alert_email
from src.json_output import write_cot_json
from src.signals import evaluate_all_commodities, get_actionable_alerts

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
)
logger = logging.getLogger(__name__)


def main() -> int:
    try:
        logger.info("Starting COT trading alert")

        email_config = load_email_config()
        logger.info("Email config loaded (recipient: %s)", email_config.recipient_email)

        all_alerts = evaluate_all_commodities()
        actionable = get_actionable_alerts(all_alerts)

        logger.info(
            "Evaluation complete: %d/%d commodities triggered",
            len(actionable),
            len(all_alerts),
        )

        send_alert_email(email_config, actionable, all_alerts)

        write_cot_json(actionable, all_alerts)
        logger.info("Wrote cot_data.json (%d buy signals)", len(actionable))

        logger.info("Done")
        return 0

    except Exception:
        logger.exception("Fatal error in COT alert script")
        return 1


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 2: Run full test suite**

```bash
pytest -v
```

Expected: all tests pass.

- [ ] **Step 3: Commit**

```bash
git add src/main.py
git commit -m "feat: write cot_data.json after email send in main.py"
```

---

## Task 9: CFTCTrade — Add Gist Update Step to Workflow

**Repo:** `cfandl-stack/CFTCTrade`

**Pre-requisite:** One-Time Setup completed (GIST_TOKEN and GIST_ID secrets set in repo).

**Files:**
- Modify: `.github/workflows/cot_alert.yml`

- [ ] **Step 1: Add Gist update step**

Replace the content of `.github/workflows/cot_alert.yml` with:

```yaml
name: "COT Trading Alert"

on:
  schedule:
    - cron: '0 21 * * 5'
  workflow_dispatch:

jobs:
  run-alert:
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run COT alert script
        env:
          SENDER_EMAIL: ${{ secrets.SENDER_EMAIL }}
          SENDER_PASSWORD: ${{ secrets.SENDER_PASSWORD }}
          RECIPIENT_EMAIL: ${{ secrets.RECIPIENT_EMAIL }}
        run: python -m src.main

      - name: Update Gist with COT data
        env:
          GH_TOKEN: ${{ secrets.GIST_TOKEN }}
          GIST_ID: ${{ secrets.GIST_ID }}
        run: gh gist edit $GIST_ID cot_data.json
        continue-on-error: true
```

- [ ] **Step 2: Commit and push**

```bash
git add .github/workflows/cot_alert.yml
git commit -m "ci: push cot_data.json to Gist after weekly alert run"
git push
```

- [ ] **Step 3: Trigger manually and verify Gist**

Go to https://github.com/cfandl-stack/CFTCTrade/actions → "COT Trading Alert" → "Run workflow".

After it completes, open the Gist URL. Verify `cot_data.json` contains valid JSON with `generated_at`, `signal_count`, `full_status`.

- [ ] **Step 4: Wire up dashboard with real Gist URL**

In `index.html`, replace:
```javascript
const COT_GIST_URL = 'REPLACE_WITH_GIST_RAW_URL';
```
with the actual raw Gist URL:
```javascript
const COT_GIST_URL = 'https://gist.githubusercontent.com/cfandl-stack/{GIST_ID}/raw/cot_data.json';
```

- [ ] **Step 5: Final end-to-end test**

Open http://localhost:8080, go to Degen Zone → COT tab. Verify real data loads from the Gist.

- [ ] **Step 6: Commit final dashboard URL update and push**

```bash
cd ../dashboard   # back in Cyberdashboard repo
git add index.html
git commit -m "feat: wire COT tab to live Gist URL"
git push
```

---

## Self-Review Against Spec

| Spec Requirement | Task |
|---|---|
| Remove Polymarket bot (stats, positions, history, links, JS) | Task 1 |
| Keep ticker bar | Task 1 (keep) |
| Tab navigation: RECHNER / INSIDER / COT | Task 2 |
| Position calculator with Margin/Hebel/Positionsgröße formulas | Task 3 |
| Long/Short validation | Task 3 |
| OpenInsider CSV fetch + table display | Task 4 |
| `exported_at` footer + row count | Task 4 |
| OpenInsider GitHub Actions workflow (Monday 07:00 UTC) | Task 6 |
| COT tab with BUY signal card + full status table | Task 5 |
| `src/json_output.py` with `build_cot_json()` | Task 7 |
| `src/main.py` calls `write_cot_json()` after email | Task 8 |
| `cot_alert.yml` Gist update step with `continue-on-error` | Task 9 |
| One-time Gist + secrets setup documented | Setup section |
