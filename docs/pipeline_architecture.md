# Pipeline Architecture — Brand IP Analytics

> Technical reference for the 4-stage trademark data pipeline.
> For project overview and findings see [README.md](../README.md).
> For the interactive report see [report/trademark_portfolio_report.html](../report/trademark_portfolio_report.html).

---

## Overview

```
Pakistani IP Office PDF Exports
            │
            ▼
  ┌─────────────────────┐
  │   Stage 1           │
  │   PDF Extraction    │  pdfplumber · two parser versions
  └────────┬────────────┘
           │  combined_data_ISMAIL.csv
           ▼
  ┌─────────────────────┐
  │   Stage 2           │
  │   Cleaning &        │  pandas · re · 9 normalisation blocks
  │   Normalisation     │  33 status variants → 20 canonical
  └────────┬────────────┘
           │  df_clean (520 records, 9 entities)
           ▼
  ┌─────────────────────┐
  │   Stage 3           │
  │   Excel Export      │  openpyxl · 11 sheets · full formatting
  └────────┬────────────┘
           │  Trademarks_Analysis.xlsx
           ▼
  ┌─────────────────────┐
  │   Stage 4           │
  │   Visualisation     │  matplotlib · 12 charts · HTML report
  └─────────────────────┘
           │
           ▼
  charts/ (12 PNGs) + report/trademark_portfolio_report.html
```

---

## Stage 1 — PDF Extraction

### Input

Pakistani IP Office PDF exports. Each file contains a paginated trademark search results table with the following columns:

| Column | Description |
|--------|-------------|
| Selected | Checkbox column — empty, dropped |
| File No. | Pakistani trademark filing number (`1/T/XXXX/XXXXXX`) |
| Novelty date | Filing date (`DD/MM/YYYY`) |
| Mark name | Trademark name |
| Classes | Nice Classification class(es) |
| Owner name | Registrant name — raw, inconsistent |
| Status | Registry status — raw, truncated, inconsistent |
| Response | Agent response field — empty across all records, dropped |

### Challenge

The registry PDFs have three structural problems that make simple extraction fail:

**1. Continuation rows** — long field values (especially Owner name) wrap across multiple table rows. The second row has the same `File No.` column empty, making it look like a new blank record.

**2. Inconsistent column counts** — some rows have more or fewer columns than the header, causing index errors when iterating.

**3. Metadata rows** — search criteria summary rows (`"Search criteria: ..."`, `"items found"`) appear mid-table and must be filtered.

### Solution — Two Parser Versions

Both parsers are retained in the notebook to document the iterative development process.

#### Version 1 — File Number Detection

```python
file_no = row[1].strip()
if file_no:          # non-empty = new record
    ...
else:                # empty = continuation row
    for i in range(len(current_record)):
        if i < len(row) and row[i].strip():
            current_record[i] = (current_record[i] + " " + row[i]).strip()
```

Detects new records by checking whether `File No.` (column index 1) is non-empty. Simple and fast, but vulnerable to edge cases where continuation rows have partial File No. values.

#### Version 2 — 1/T Anchor Parser *(production)*

```python
file_no = row[2].strip() if len(row) > 2 else ""
if file_no.startswith("1/T"):    # Pakistani filing format anchor
    ...
```

Anchors new record detection to the `1/T` prefix of Pakistani trademark filing numbers. More robust — a continuation row can never accidentally match `1/T`. This is the version used for all downstream processing.

**Fixes applied to both versions:**

```python
# Fix 1 — reproducible file ordering
for file_name in sorted(os.listdir(folder_path)):

# Fix 2 — IndexError on continuation rows
for i in range(len(current_record)):      # iterate by parent length
    if i < len(row) and row[i]:           # boundary check before read

# Fix 3 — pandas 2.x compatibility
df = df.map(lambda x: " ".join(str(x).split()))   # replaces applymap
```

### Output

`combined_data_ISMAIL.csv` — raw extracted records, UTF-8 BOM encoded for Excel compatibility.

---

## Stage 2 — Cleaning & Normalisation

### 2A — Owner Name Normalisation

The registry returns owner names in dozens of inconsistent formats for the same entity:

```
Raw variants for a single entity:
  "ISMAIL INDUSTRIES"
  "ISMAIL INDUSTRIES LIMITED"
  "Ismail Industries Limited [PK]"
  "Trading As ISMAIL INDUSTRIES LIMITED"
  "Trading as Trading As ISMAIL INDUSTRIES LIMITED [PK]"
  "M/s. ISMAIL INDUSTRIES LIMITED [PK]"
  "Trading As ISMAIL GOUP OF INDUSTRIES [PK]"    ← typo: GOUP
```

**Nine normalisation blocks** handle all variations for all 9 entities:

```python
# Pattern design principles:

# 1. Case insensitive
r'(?i)^ismail\s+industries.*'

# 2. Prefix-agnostic (catches "Trading As", "M/s." etc.)
r'(?i).*trading\s+as\s+ismail\s+industries.*'

# 3. Optional characters for typos
r'(?i).*ismail\s+gr?oup\s+(of\s+)?industries.*'
#                    ↑           ↑
#             R optional    "OF" optional
#             catches GOUP  catches missing word

# 4. Singular/plural variants
r'(?i).*cambridge\s+garments?\s+industries.*'
#                          ↑
#                   S optional — catches both GARMENT and GARMENTS

# 5. Brand confusion via alternation
r'(?i).*(euro|furo)\s+food\s+industries.*'
#              ↑
#        catches registry typo FURO
```

**False match removal — 5-record threshold:**

```python
owner_counts = df['Owner name'].value_counts()
valid_owners = owner_counts[owner_counts >= 5].index
df_clean = df[df['Owner name'].isin(valid_owners)].copy()
```

36 records dropped — all individuals who share a name fragment with client entities (e.g. `"Mohammad Ismail, Trading As BADO PLASTICO BRUSH INDUSTRIES"`) but operate entirely unrelated businesses. The 5-record threshold is conservative and defensible — every dropped entity had 1–3 records and zero connection to the client portfolio.

### 2B — Status Normalisation

The registry truncates status strings inconsistently, producing 33 raw variants for 20 logical values:

```
Raw                                          →  Canonical
─────────────────────────────────────────────────────────
"Trademark registered"                       →  "Trademark Registered"
"Trademark registere"                        →  "Trademark Registered"
"Trademark register"                         →  "Trademark Registered"
"RTM"                                        →  "Trademark Registered"
"Opposition decisio"                         →  "Opposition Decisions"
"Opposition decision"                        →  "Opposition Decisions"
"Opposition decisions"                       →  "Opposition Decisions"
"Opposition (period) finish"                 →  "Opposition Period Finished"
"Opposition (period) finishe"                →  "Opposition Period Finished"
"Opposition (period) finished"               →  "Opposition Period Finished"
"Grace perio"                                →  "Grace Period"
"Grace period"                               →  "Grace Period"
"Examinatio"                                 →  "Under Examination"
"Examination"                                →  "Under Examination"
"Order Sectio"                               →  "Order Section"
"Abandone"                                   →  "Abandoned"
"Withdraw"                                   →  "Withdrawn"
"Reminder 33(5) is Due"                      →  "Reminder Due"
"Reminder is du"                             →  "Reminder Due"
"To be deemed withdrawn"                     →  "To Be Abandoned"
"Trademark register NCL(0-0) 30 ..."         →  "Trademark Registered"
```

**Critical implementation detail — ordering matters:**

```python
# WRONG order — partial match shadow:
if s.startswith('Trademark register'):     # catches "Trademark Registered - Rectification"
    return 'Trademark Registered'          # incorrectly normalises it

# CORRECT order — uppercase first:
if s.startswith('Trademark Registered'):   # catches full uppercase variant first
    if 'Rectif' in s:
        return 'Trademark Registered - Rectification received'
    return 'Trademark Registered'
if s.startswith('Trademark register'):     # only reaches here for lowercase variants
    return 'Trademark Registered'
```

**Clean / Grey / Red scale mapping:**

```python
STATUS_MAPPING = {
    # CLEAN — active, registered, in progress positively
    'Trademark Registered':       {'scale': 'Clean'},
    'Certificate Printed':        {'scale': 'Clean'},
    'Ready To Be Registered':     {'scale': 'Clean'},
    'Under Examination':          {'scale': 'Clean'},
    'Acceptance Publication':     {'scale': 'Clean'},
    'Reception':                  {'scale': 'Clean'},

    # GREY — pending, unclear, action may be required
    'Grace Period':               {'scale': 'Grey'},
    'Hearing':                    {'scale': 'Grey'},
    'Reminder Due':               {'scale': 'Grey'},
    'Order Section':              {'scale': 'Grey'},
    'Awaiting Opposition':        {'scale': 'Grey'},

    # RED — at risk, lapsed, or actively contested
    'Opposition Decisions':                          {'scale': 'Red'},
    'Opposition Period Finished':                    {'scale': 'Red'},
    'Abandoned':                                     {'scale': 'Red'},
    'To Be Abandoned':                               {'scale': 'Red'},
    'Withdrawn':                                     {'scale': 'Red'},
    'Migrated':                                      {'scale': 'Red'},
    'Refused':                                       {'scale': 'Red'},
    'Removed':                                       {'scale': 'Red'},
    'Trademark Registered - Rectification received': {'scale': 'Red'},
}
```

### Kernel-restart Safety

`normalise_status()` and `STATUS_MAPPING` are defined at the top of Stage 3's notebook cell — not in a separate cell. This ensures that `split_dataframe()`, which calls `normalise_status()` internally, never fails with `NameError` regardless of cell execution order.

---

## Stage 3 — Excel Export

### Workbook Structure

```
Trademarks_Analysis.xlsx
│
├── Master_All_Data          ← all 520 records + Source_Sheet column
├── Ismail Industries        ← 420 records
├── Ismail Group             ← 40 records
├── Agrolet Chemicals        ← 14 records
├── Euro Food                ← 13 records
├── Abid Industries          ← 9 records
├── Union Thread             ← 8 records
├── Cambridge Garments       ← 6 records
├── Union Textile            ← 5 records
├── Ideal Food               ← 5 records
└── Unique Mark Names        ← cross-entity mark frequency analysis
```

### Per-Entity Sheet Layout

Each entity sheet contains three blocks stacked vertically:

```
Row 1–4   ┌──────────────────────────────────┐
          │  SUMMARY BLOCK                   │  Sheet Name / Total / Statuses / Owner
          └──────────────────────────────────┘
Row 6     ┌──────────────────────────────────┐
          │  STATUS BREAKDOWN                │  Section header (amber)
          ├──────────────────────────────────┤
Row 8+    │  Status | Count | Percentage     │  Normalised status counts
          └──────────────────────────────────┘
Row N     ┌──────────────────────────────────┐
          │  DETAILED TRADEMARK DATA         │  Section header (amber)
          ├──────────────────────────────────┤
Row N+2+  │  Full record table               │  All columns including Status_Clean
          └──────────────────────────────────┘
```

### Colour Scheme

| Element | Colour | Hex |
|---------|--------|-----|
| Column headers | Dark blue | `#366092` |
| Summary block | Light blue | `#D9E1F2` |
| Status breakdown header | Green | `#70AD47` |
| Section titles | Amber | `#FFC000` |
| Unique Marks — col 1 | Light green | `#E2EFDA` |
| Unique Marks — col 2 | Light blue | `#DDEEFF` |
| Unique Marks — col 3 | Light yellow | `#FFF2CC` |

### Status_Clean Column

All Excel sheets use `Status_Clean` (normalised values) for status breakdowns, not the raw `Status` column. This ensures the workbook reflects the 20 canonical statuses rather than the 33 raw variants.

---

## Stage 4 — Visualisation

### Chart Design Principles

**Colour encoding is consistent across all charts:**
- 🟢 Green family (`#70AD47` base) → Clean statuses
- ⚫ Grey family (`#7F7F7F` base) → Grey statuses
- 🔴 Red family (`#C00000` base) → Red statuses
- 🔵 Steel blue (`#369bff`) → Rest of portfolio (context slice in donuts)

Within each scale group, 7 distinct shades are cycled so adjacent bars of the same scale are visually distinguishable.

**No axis clutter** — all charts remove y-axis ticks and spines. Count labels are placed directly on bars. Legends replace axis labels where possible.

### Chart 01 — Overall Status Distribution

- Type: Vertical bar chart
- Data: All 520 records, 20 normalised status categories, sorted descending
- Legend: Three separate framed boxes (CLEAN / GREY / RED) below the chart
- Each box uses coloured border matching its scale group

### Chart 02 — Portfolio Health by Entity

- Type: Stacked horizontal bar chart
- Data: Clean / Grey / Red counts per entity
- Features: Count labels inside segments (threshold ≥ 8), `n=total` at bar end, % annotation in dominant segment
- Entities ordered by total record count (largest at top)

### Chart 03 — Entity Portfolio Share (3×3 Donut Grid)

- Type: 3 rows × 3 columns of donut charts
- Data: Per-entity Clean/Grey/Red segments + rest-of-portfolio context slice
- Centre text: Raw record count + % of total portfolio + "of portfolio"
- Annotation below each donut: `C:N  G:N  R:N`
- Card background: `#e1edff` with `#00070f` border

### Charts 04–12 — Entity Dashboards

Each dashboard is a 2-row combined figure:

```
┌─────────────────────────────────────────────────────┐
│  Row 1 (height ratio 2.2)                           │
│  Status barh — full width                           │
│  Legends (CLEAN / GREY / RED) inside right          │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────┴──────────────────────────────────┐
│  Row 2 (height ratio 1.0)                           │
│  [1/3] Donut     │  [2/3] Top N mark names bar     │
│  Clean/Grey/Red  │  Blue gradient — darkest = most  │
│  + legend        │  filed                           │
└──────────────────┴──────────────────────────────────┘
```

Subtitle bar below main title: `Total Records | Unique Marks | Filing Range | C: G: R:`

`top_n = min(10, len(df_entity['Mark name'].value_counts()))` — adapts to smaller entities (5–8 records) without crashing.

### HTML Report

`report/trademark_portfolio_report.html` is a **fully self-contained** single-file report. All 12 chart PNGs are embedded as base64 strings:

```python
def img_to_base64(filepath):
    with open(filepath, 'rb') as f:
        return base64.b64encode(f.read()).decode('utf-8')

# Embedded in HTML as:
f'<img src="data:image/png;base64,{img_b64}">'
```

**Report sections:**
- Header with 6 portfolio stat boxes (dark blue gradient)
- Sticky navigation bar (Overview / Pipeline / Entities / Dashboards / Notes)
- Confidentiality notice banner
- 4-stage pipeline architecture cards
- Entity summary table (9 rows, Clean/Grey/Red counts, health indicator)
- 3 portfolio overview charts
- 9 entity dashboard cards with sector tags and stat pills
- Methodology and classification notes

File size: approximately 3–5 MB (varies with chart DPI).

---

## Data Flow Summary

```
Input:    N PDF files in folder_path/
          ↓
Step 1:   pdfplumber extracts table rows
          ↓
Step 2:   Continuation rows merged into parent records
          ↓
Step 3:   Metadata rows filtered (ignore_keywords)
          ↓
Step 4:   DataFrame constructed, Response column dropped
          → combined_data_ISMAIL.csv (556 rows)
          ↓
Step 5:   9 owner name normalisation blocks applied
          ↓
Step 6:   5+ record threshold filter
          → df_clean (520 rows, 9 entities)
          ↓
Step 7:   Status normalisation applied (normalise_status)
          → Status_Clean column added to all entity dataframes
          ↓
Step 8:   Split into df1–df9 + df_others
          ↓
Step 9:   Excel export — 11 sheets
          → Trademarks_Analysis.xlsx
          ↓
Step 10:  12 charts generated
          → charts/*.png
          ↓
Step 11:  HTML report generated with embedded charts
          → report/trademark_portfolio_report.html
```

---

## Known Limitations

**PDF dependency:** The pipeline depends on `pdfplumber` successfully extracting structured tables. PDFs that are scanned images rather than text-layer PDFs will return empty tables. All source PDFs in this project are text-layer exports from the IP Office portal.

**Status coverage:** `normalise_status()` returns `f'Other: {s[:40]}'` for any unrecognised status string. A verification cell (Stage 4A) confirms zero `Other:` entries in this dataset. Future datasets with new status variants will require additional normalisation blocks.

**Threshold sensitivity:** The 5-record minimum for entity inclusion is a judgement call. Reducing to 3 would include additional entities; increasing to 10 would exclude Cambridge Garments, Union Textile, and Ideal Food. The current threshold is documented and can be adjusted in a single line.

**Windows path:** `folder_path = r"D:\ISMAIL INDUSTRIES"` is hardcoded for the original development environment. Update to your local PDF folder path before running Stage 1.

---

*For project overview, findings, and visualisations see [README.md](../README.md)*
*For the interactive portfolio report see [report/trademark_portfolio_report.html](../report/trademark_portfolio_report.html)*
