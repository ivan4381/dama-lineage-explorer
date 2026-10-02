# DAMA ETL Lineage Explorer

A single-file, no-build HTML app for exploring Airflow **DAG → task → script → table**
lineage. Open `DAMA_ETL_Lineage_Explorer.html` in a browser — no server, no install.

**[Open the app](./DAMA_ETL_Lineage_Explorer.html)** (or double-click the file locally).

## What it does

- **Explore**: search across DAGs, tasks, bash runners, scripts and tables; filter by
  entity type, technology (SQL Server, Hive, BigQuery, Excel, MySQL, CSV, Azure SQL,
  SAP BW, Website, GCS, …) and DAG scope.
- **Lineage graph**: pick any entity and see its upstream/downstream chain, with
  adjustable depth (direct, 2 hops, 3 hops, or the full chain), zoom/pan, and a
  back button to retrace your steps.
- **Rows view**: the underlying lineage rows as a searchable, paginated table.
- **Details panel**: facts and a collapsible tree of related entities (tasks,
  scripts, DAGs, tables, bash runners) for whatever you've focused.
- **Shareable links**: the focused entity is kept in the URL (`#focus=...`), so a
  link can be sent straight to a specific table or DAG; browser back/forward works.
- **Ask AI** (bring your own Google Gemini key): ask questions about the focused
  entity's lineage. Open the *Ask AI* tab, paste your Gemini API key, click
  *Load models* and pick one. The browser calls Gemini directly — no backend. Each question sends the
  focused entity's lineage at the current depth plus its rows, not the whole dataset;
  for anything else the model calls in-browser search tools (`search_entities`,
  `get_lineage`, `search_rows`) that return only the matching parts. Each reply
  shows its token usage (hover for input/output and the tools used).
  The key is kept in `sessionStorage` (cleared when the tab closes) unless
  *Remember key* is ticked, which moves it to `localStorage`.
- Light/dark theme (follows the OS by default, toggleable).

## Data

The app ships with a dataset **baked into the HTML** (the `EMBEDDED` array near the
top of the `<script>` block), generated from [`Lineage_DAG_Master_Complete.csv`](./Lineage_DAG_Master_Complete.csv).
Opening the file needs nothing else — the dataset loads instantly, no fetch, no CORS.

Columns (header names are matched loosely, so close variants also work):

| Column | Meaning |
|---|---|
| `DAG Name` | Airflow DAG |
| `DAG Task Id` | Task within the DAG |
| `Bash File Name` | Bash runner script that runs the task |
| `Python Script File Name` | Python script that does the actual ETL |
| `Source` / `Source Type` | Table/path read, and its technology |
| `Target Table Path` / `Target Table Type` | Table/path written, and its technology |
| `Owner` | Owner of the task (optional; shown in the Details panel) |
| `Task Description` | What the task does (optional; shown in the Details panel) |

You can also **load your own file** from the app (CSV/TXT/XLSX/XLS, same column
layout) via "Load CSV / Excel" or drag-and-drop — it's parsed in the browser and
kept in `localStorage`, so it's remembered next time you open the file. "Reset
data" switches back to the built-in dataset above.

### Updating the built-in dataset

When `Lineage_DAG_Master_Complete.csv` changes, re-embed it into the HTML. Task
descriptions contain quoted commas, so this reuses the app's own CSV parser
(`parseCSV`) instead of a naive `split(',')`:

```js
node -e "
const fs = require('fs');
const CSV = 'Lineage_DAG_Master_Complete.csv', HTML = 'DAMA_ETL_Lineage_Explorer.html';
let html = fs.readFileSync(HTML, 'utf8');
const src = html.slice(html.indexOf('function parseCSV('), html.indexOf('function parseWorkbook('));
const parseCSV = new Function(src + '; return parseCSV;')();
const rows = parseCSV(fs.readFileSync(CSV, 'utf8')).filter(r => r.some(c => c.trim()));
const nl = html.includes('\r\n') ? '\r\n' : '\n';
const lines = html.split(/\r?\n/);
const idx = lines.findIndex(l => l.startsWith('const EMBEDDED = ['));
lines[idx] = 'const EMBEDDED = ' + JSON.stringify(rows).split('</').join('<' + String.fromCharCode(92) + '/') + ';'; // never close the <script> early
lines[idx + 1] = 'const EMBEDDED_NAME = ' + JSON.stringify(CSV) + ';';
fs.writeFileSync(HTML, lines.join(nl), 'utf8');
"
```

## Tech

Plain HTML/CSS/JS, zero dependencies except the `xlsx` library (loaded from a CDN)
for reading `.xlsx`/`.xls` uploads. Everything else — parsing, the lineage graph
layout, search/filtering — is hand-rolled in the single `<script>` block.
