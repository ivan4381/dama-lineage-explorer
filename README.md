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
- Light/dark theme (follows the OS by default, toggleable).

## Data

The app ships with a dataset **baked into the HTML** (the `EMBEDDED` array near the
top of the `<script>` block), generated from [`Lineage_DAG_Master.csv`](./Lineage_DAG_Master.csv).
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

You can also **load your own file** from the app (CSV/TXT/XLSX/XLS, same column
layout) via "Load CSV / Excel" or drag-and-drop — it's parsed in the browser and
kept in `localStorage`, so it's remembered next time you open the file. "Reset
data" switches back to the built-in dataset above.

### Updating the built-in dataset

When `Lineage_DAG_Master.csv` changes, re-embed it into the HTML:

```js
node -e "
const fs = require('fs');
let csv = fs.readFileSync('Lineage_DAG_Master.csv', 'utf8');
if (csv.charCodeAt(0) === 0xFEFF) csv = csv.slice(1); // strip BOM
const rows = csv.split(/\r?\n/).filter(Boolean).map(l => l.split(','));
const literal = 'const EMBEDDED = ' + JSON.stringify(rows) + ';';
let html = fs.readFileSync('DAMA_ETL_Lineage_Explorer.html', 'utf8');
const nl = html.includes('\r\n') ? '\r\n' : '\n';
const lines = html.split(/\r?\n/);
const idx = lines.findIndex(l => l.startsWith('const EMBEDDED = ['));
lines[idx] = literal;
fs.writeFileSync('DAMA_ETL_Lineage_Explorer.html', lines.join(nl), 'utf8');
"
```

This assumes the CSV has no commas or quotes inside a field (true for the current
file — every row splits cleanly into 8 columns). If that stops being true, parse
with a real CSV library instead of the naive `split(',')` above.

## Tech

Plain HTML/CSS/JS, zero dependencies except the `xlsx` library (loaded from a CDN)
for reading `.xlsx`/`.xls` uploads. Everything else — parsing, the lineage graph
layout, search/filtering — is hand-rolled in the single `<script>` block.
