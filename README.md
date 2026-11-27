# UNICEF WASH Access Dashboard (2000–2025)

An end-to-end data pipeline and interactive Power BI dashboard analysing
access to **basic, limited, and no water service** in schools across
global regions, income groups, and UNICEF/WHO reporting frameworks.

---

## 1. Project Overview

This project takes a raw UNESCO Institute for Statistics (UIS) CSV extract,
cleans it into a tidy schema, loads it into PostgreSQL, and connects it to
Power BI for live dashboards.

**Headline story:** Basic water coverage in schools improved meaningfully
from 2010 to 2025, driven almost entirely by gains in Central/Southern and
Eastern/South-Eastern Asia — while Sub-Saharan Africa and fragile states
remain roughly where they were a decade ago.

---

## 2. Repository Contents

| File | Description |
|---|---|
| `data1.1.csv` | Raw source file (multi-header, ~2000 trailing empty columns) |
| `data_clean.csv` | Cleaned, tidy output — one row per `(region_grouping, region, year)` |
| `clean.ipynb` | Jupyter notebook performing all cleaning steps |
| `load_postgres.ipynb` | Notebook that pushes `data_clean` into PostgreSQL |
| `dashboard.pbix` | Power BI report (DirectQuery against PostgreSQL) |
| `README.md` | This file |

---

## 3. Data Sources & Citation

> UNESCO Institute for Statistics (UIS). *Water, sanitation and hygiene
> (WASH) in schools — global and regional time series, 2000–2025.*
> Extracted and cleaned from `data1.1.csv` → `data_clean.csv`.
> UIS, Montreal, 2025.

---

## 4. Schema

### 4.1 Identifier columns

| Column | Type | Description |
|---|---|---|
| `region_grouping` | TEXT | Which region scheme the row belongs to |
| `REGION` | TEXT | Region name within that scheme |
| `Year` | INT | Calendar year (2000–2025) |

**Region groupings present:**

- `SDG REGIONS` — 8 regions
- `OTHER REGIONAL GROUPINGS` — 5 regions (e.g. Landlocked, LDCs, SIDS, Fragile)
- `INCOME GROUPINGS` — 4 regions (Low, Lower-middle, Upper-middle, High)
- `UNICEF PROGRAMME REGIONS` — 8 regions
- `UNICEF REPORTING REGIONS` — ~7 regions
- `WHO REGIONS` — 6 regions
- `WORLD` — global aggregate

### 4.2 Context columns

| Column | Description |
|---|---|
| `School age-population (thousands)` | School-age population |
| `% urban` | Urbanisation rate |
| `% pre-primary` / `% primary` / `% secondary` | Enrolment share by level |

### 4.3 Water-service metrics (18 columns)

Format: `{SCOPE}__{SERVICE}`

| SCOPE | SERVICE |
|---|---|
| `TOTAL` | `basic`, `limited`, `none` |
| `URBAN` | `basic`, `limited`, `none` |
| `RURAL` | `basic`, `limited`, `none` |
| `PRE-PRIMARY` | `basic`, `limited`, `none` |
| `PRIMARY` | `basic`, `limited`, `none` |
| `SECONDARY` | `basic`, `limited`, `none` |

All values are percentages. **Caveat:** `<1` and `>99` are stored as text
sentinels (see §6).

---

## 5. Pipeline

### 5.1 Cleaning (`clean.ipynb`)

1. Read raw CSV with 3-row header (`header=None`, `keep_default_na=False`).
2. Drop columns empty in both header rows.
3. Forward-fill top header (TOTAL, URBAN, …) and pair with sub-header
   (Basic / Limited / No water) → MultiIndex.
4. Flatten to `SCOPE__service` names.
5. Detect section markers (`SDG REGIONS`, `INCOME GROUPINGS`, …, `Key:`) —
   tag every data row with its current `region_grouping`.
6. Split data vs. markers using file order + Year sanity check.
7. Clean numeric strings (remove thin spaces, commas).
8. Reorder columns and export `data_clean.csv`.

### 5.2 Loading (`load_postgres.ipynb`)

1. Create DB `education_data` in PostgreSQL.
2. Connect via SQLAlchemy (`postgresql+psycopg2`).
3. Rename columns to snake_case.
4. `to_sql(..., if_exists="replace", method="multi", chunksize=1000)`.
5. Create index on `(region_grouping, year)`.

### 5.3 Dashboard

Power BI Desktop → Get Data → PostgreSQL → **DirectQuery** mode →
select `public.education_water_access`.

---

## 6. Known Data Limitations

| Issue | Impact | Mitigation |
|---|---|---|
| **Sparsity pre-2010** | Mostly TOTAL-only; urban/rural/school-level blanks | Use `TOTAL__basic` for 2000–2025 trend; scope breakdowns to 2011+ (2017+ for school levels) |
| **`<1` / `>99` sentinels** | Columns stored as TEXT in Postgres | Display as strings in dashboard; do not average |
| **Overlapping aggregates** | Income groups vs SDG regions double-count population | Always filter to one `region_grouping` at a time |
| **Pre-primary data desert** | Several regions have no pre-primary figures | Treat blank as "not reported", not zero |

---

## 7. Setup Instructions

### 7.1 Prerequisites

- Python 3.10+
- PostgreSQL 15+ (tested on 18)
- Power BI Desktop

### 7.2 Install Python dependencies

```bash
pip install pandas sqlalchemy psycopg2-binary
