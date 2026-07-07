# Dunder Mifflin Dataset Dictionary

## Layout

```
data/
├── clean/          # canonical warehouse tables — tidy, ready to analyze
│   ├── branches.csv                 (11)
│   ├── employees.csv                (500)
│   ├── products.csv                 (119)
│   ├── customers.csv                (2,500)
│   ├── sales_orders.csv             (72,000)      order headers
│   ├── sales_order_lines.csv        (215,485)     ← large time-series fact table
│   ├── sales_targets.csv            (1,637)       quarterly quota vs actual
│   └── ai_readiness_survey.csv      (473)         one row per active employee
├── ml/             # machine-learning layer (invented paper mill)
│   ├── machines.csv                 (48)
│   ├── production_runs.csv          (94,307)
│   ├── machine_telemetry.csv        (262,944)     ← large sensor table
│   ├── maintenance_events.csv       (3,867)       preventive + failures
│   └── equipment_failure_dataset.csv (87,648)     ← model-ready, one row per machine-day
├── raw_sources/    # intentionally-messy exports for the pipeline module
│   ├── crm_customer_export.csv      (2,189)
│   ├── erp_orders_export.csv        (66,096)
│   └── regional_manager_tracker.csv (600)
└── excel_starter/  # small, Excel-sized slice (Scranton, 2022) — see below
    ├── (same clean tables, scoped)           ≤ 17,520 rows each
    ├── equipment_failure_dataset.csv         (17,520)
    ├── machine_telemetry_sample.csv          (73)  ← one failure, for a line chart
    └── (scoped crm / erp / regional_manager exports)
```

---

## One-line "why it's in there" index

| Feature | File | Serves |
|---|---|---|
| Salary skew + CEO outlier | employees | mean vs median |
| One column per variable type | products | variable typing |
| Linear price ~ weight + type | products | dummy-coded regression |
| Trend + Q4 + summer + 2020 shock + Stamford close | sales_order_lines | forecasting & storytelling |
| Renamed cols / bad dates / dupes / currency strings | raw_sources | Power Query clean & combine |
| Wrong hand-entered totals | regional_manager_tracker | single source of truth |
| Star schema on surrogate keys | clean/* | SQL ↔ Excel |
| ~7% positive label | equipment_failure_dataset | accuracy is misleading |
| Pre-failure sensor ramp | telemetry / *_roll7 | signal a simple model can learn |
| Vibration→pressure lead shift over years | equipment_failure_dataset | concept drift / deploy degradation |
| Comfort↔usage corr, dept variation, barriers | ai_readiness_survey | ADKAR diagnosis |


## `data/clean/` — canonical warehouse

### branches.csv
Grain: one row per branch (incl. Corporate HQ). Key: `branch_id`.

| column | type | notes |
|---|---|---|
| branch_id | Identifier | PK |
| branch_name | Nominal | Scranton, Stamford, … |
| city, state | Nominal | |
| region | Nominal | Northeast / Midwest / Corporate |
| open_date | Date | |
| close_date | Date | populated only for Stamford (closes 2020-08-31) |
| sq_ft | Continuous | |
| warehouse_flag | Boolean | branch has a warehouse |
| manager_employee_id | Identifier | FK → employees.employee_id |

### employees.csv
Grain: one row per employee. Key: `employee_id`. Canon roster first, then filler.

| column | type | notes |
|---|---|---|
| employee_id | Identifier | PK |
| first_name, last_name | Nominal | |
| branch_id | Identifier | FK → branches |
| department | **Nominal** | Sales, Accounting, Warehouse, Management, Corporate, … |
| job_title | Nominal | |
| level | **Ordinal** | 1 (entry) … 6 (exec) |
| hire_date | Date | |
| termination_date | Date | null unless status = Terminated |
| employment_status | Nominal | Active / Terminated |
| salary | **Continuous** | right-skewed; exec outliers (CEO = $410k) |
| bonus_eligible | Boolean | |
| manager_id | Identifier | FK → employees (self-reference) |
| email | Identifier | |

### products.csv
Grain: one row per SKU. Key: `product_id`. Carries one column of **each** variable type.

| column | type | notes |
|---|---|---|
| product_id | Identifier | PK |
| sku | Identifier | |
| product_name | Nominal | human-readable |
| paper_type | **Nominal** | Copy Paper, Cardstock, Glossy Photo, … |
| weight_gsm | **Continuous** | grams/m² — regression predictor |
| ream_size | **Discrete** | 250 / 500 / 1000 sheets |
| brand_tier | **Ordinal** | Economy < Standard < Premium |
| color | Nominal | |
| recycled_flag | Boolean | |
| unit_cost | Continuous | |
| list_price | Continuous | ≈ 0.045·weight_gsm + type-offset + tier-offset + noise |

### customers.csv
Grain: one row per customer. Key: `customer_id`. First 15 are canon (Vance Refrigeration, …).

| column | type | notes |
|---|---|---|
| customer_id | Identifier | PK |
| customer_name | Nominal | |
| industry | Nominal | |
| segment | **Ordinal** | SMB < Mid-Market < Enterprise |
| city, state, region | Nominal | |
| account_owner_employee_id | Identifier | FK → employees (a Sales rep) |
| credit_rating | Ordinal | A / B / C / D |
| signup_date | Date | |
| active_flag | Boolean | |

### sales_orders.csv
Grain: one row per order (header). Key: `order_id`.

| column | type | notes |
|---|---|---|
| order_id | Identifier | PK |
| order_date | Date | drives the time series |
| customer_id | Identifier | FK → customers |
| branch_id | Identifier | FK → branches |
| sales_rep_employee_id | Identifier | FK → employees |
| channel | Nominal | Field Sales / Phone / Online |
| order_status | Nominal | Fulfilled / Returned / Cancelled |
| ship_date | Date | null when Cancelled |
| discount_pct | Continuous | 0–0.15 |

### sales_order_lines.csv  *(large fact table)*
Grain: one row per product line on an order. Key: `order_line_id`. Join to orders on `order_id`.

| column | type | notes |
|---|---|---|
| order_line_id | Identifier | PK |
| order_id | Identifier | FK → sales_orders |
| product_id | Identifier | FK → products |
| quantity | **Discrete** | right-skewed (whale orders) |
| unit_price | Continuous | list_price after discount |
| line_total | Continuous | unit_price × quantity |
| unit_cost | Continuous | |
| margin | Continuous | line_total − unit_cost·quantity |

### sales_targets.csv
Grain: one row per rep × year × quarter. Compound key.

| column | type | notes |
|---|---|---|
| branch_id | Identifier | FK → branches |
| employee_id | Identifier | FK → employees |
| year | Ordinal | |
| quarter | Ordinal | 1–4 |
| quota | Continuous | |
| actual_revenue | Continuous | sum of fulfilled line_total |
| attainment_pct | Continuous | 100 × actual / quota |

### ai_readiness_survey.csv
Grain: one row per **active** employee. Key: `response_id`. Join on `employee_id`.
All `q_*` items are 5-point Likert (**Ordinal**, treated as interval for regression);
`freq_*` items are usage frequency 1 (never) … 5 (daily).

| column | type | notes |
|---|---|---|
| response_id | Identifier | PK |
| employee_id | Identifier | FK → employees |
| department, level | Nominal / Ordinal | denormalized for convenience |
| submitted_date | Date | |
| q_pipeline_confidence … q_lead_change_mgmt | **Ordinal** (Likert 1–5) | confidence/agreement items mirroring the needs survey |
| freq_ai_email, freq_ai_reports, freq_ai_code, freq_ai_review | **Ordinal** (1–5) | AI-tool usage frequency |
| primary_barrier | **Nominal** | Unclear value / Skill gap / Trust in output / Time pressure / Data privacy / None |

---

## `data/ml/` — machine-learning layer (paper mill)

### machines.csv
Grain: one row per machine. Key: `machine_id`.

| column | type | notes |
|---|---|---|
| machine_id | Identifier | PK |
| machine_name | Nominal | |
| machine_type | **Nominal** | Paper Machine / Winder / Sheeter / Cutter / Wrapper |
| facility | Nominal | Scranton Mill / Nashua Converting |
| install_date | Date | |
| manufacturer, model | Nominal | |
| rated_capacity_units_hr | Discrete | |
| criticality | **Ordinal** | 1 (low) … 5 (critical) |

### machine_telemetry.csv  *(large sensor table)*
Grain: one reading per machine every 8 hours. Key: `reading_id`. Join on `machine_id`.
Sensor values **ramp abnormally in the 7 days before a failure** (the learnable signal).

| column | type | notes |
|---|---|---|
| reading_id | Identifier | PK |
| machine_id | Identifier | FK → machines |
| timestamp | Date/Time | |
| temperature_c, vibration_mm_s, pressure_psi, roller_speed_rpm, humidity_pct, motor_current_a | **Continuous** | sensor channels |

### production_runs.csv
Grain: one row per production run. Key: `run_id`.

| column | type | notes |
|---|---|---|
| run_id | Identifier | PK |
| machine_id | Identifier | FK → machines |
| product_id | Identifier | FK → products |
| shift | Nominal | Day / Swing / Night |
| operator_employee_id | Identifier | FK → employees |
| start_time, end_time | Date/Time | |
| units_produced | Discrete | |
| scrap_units | Discrete | rises near failures |

### maintenance_events.csv
Grain: one row per maintenance/failure event. Key: `event_id`.

| column | type | notes |
|---|---|---|
| event_id | Identifier | PK |
| machine_id | Identifier | FK → machines |
| event_date | Date | |
| event_type | **Nominal** | Preventive / Failure |
| failure_mode | Nominal | null for preventive |
| downtime_hours | Continuous | right-skewed |
| repair_cost | Continuous | right-skewed |

### equipment_failure_dataset.csv  *(model-ready)*
Grain: one row per machine × day. Derived from telemetry + production + maintenance.
This is the CSV Rachel's module starts from. **Target: `failed_within_7_days`.**

| column | type | notes |
|---|---|---|
| machine_id | Identifier | FK → machines |
| date | Date | |
| temp_mean, vib_mean, pres_mean, cur_mean, hum_mean | Continuous | that day's sensor means |
| *_roll7, *_roll7_std | Continuous | 7-day rolling mean/std features |
| units_produced, scrap_units | Discrete | that day's production (0 if idle) |
| machine_type | Nominal | |
| criticality | Ordinal | |
| rated_capacity_units_hr | Discrete | |
| machine_age_days | Discrete | date − install_date |
| days_since_maintenance | Discrete | `999` sentinel before first record |
| **failed_within_7_days** | Boolean (0/1) | **label**; ~7% positive (imbalanced) |

---

## `data/raw_sources/` — messy exports (clean these!)

These are dirtied views of entities that also live in `clean/`. Column names, formats, and
quality are intentionally inconsistent — the exercise is to combine, clean, and reconcile
them, then find where they disagree with the canonical tables.

### crm_customer_export.csv
Customers as a "CRM" dump. Corruptions: mixed-case / whitespace-padded `Account Name`,
mixed phone formats, missing `Acct ID`/`Industry`/`Segment`/`City`, **~3% duplicate rows**.
Renamed columns: `Account Name, Acct ID, Industry, Segment, Phone, City, State, Created`
(`Created` is `MM/DD/YYYY`). Reconcile `Acct ID` → `customers.customer_id`.

### erp_orders_export.csv
Orders as an "ERP" dump. Corruptions: `MM/DD/YYYY` dates (vs ISO elsewhere), `AMT` as a
currency **string** (`"$1,234.56"`), uppercase `STATUS`, blank `SHIP_DT`, **~2% duplicates**.
Columns: `ORDER_NO, CUST_ID, BRANCH, ORDER_DT, SHIP_DT, STATUS, AMT`. `ORDER_NO` embeds the
`order_id` (`ORD0001234`).

### regional_manager_tracker.csv
A hand-kept Scranton spreadsheet. Corruptions: **typo'd product & client names** (need fuzzy
matching back to canonical IDs), free text in `Qty` (`"~50"`, `"a dozen"`), 2-digit dates,
and **`Total $` values that disagree with the canonical order-line totals for ~15% of rows**
— the "single source of truth" violation. Columns: `Date, Client, Rep, Product, Qty,
Total $, Notes`.
