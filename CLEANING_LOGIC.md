# Zededa HRIS — Rippling Data Cleaning Logic
**Source:** Rippling HR Export (Excel)
**Last Updated:** March 17, 2026

---

## Input Files
| File | Purpose |
|------|---------|
| `160439_HR_Rippling_Report_-_for_Urja_3-2026.xlsx` | Rippling HR export (source data) |
| `wise_fx_all_to_USD.xlsx` | FX conversion rates (all currencies → USD) |

---

## Source Column Reference
| Column Name | Description |
|-------------|-------------|
| Employee | Full name |
| Rippling profile number | Unique ID in Rippling |
| Title | Job title |
| Employment type name | Classification (Salaried full-time, Hourly, etc.) |
| Start date | Employment start date |
| Last work date | Termination date; blank if active |
| Department | Employee's department |
| Work location | Office location or "Remote" |
| Base compensation - Currency | Currency code for base comp |
| Base compensation | Base salary/pay amount |
| On-target commission - Currency | Currency code for OTC |
| On-target commission | On-target commission amount |
| Target annual bonus percent | Bonus % (e.g., 0.15 = 15%) |
| Grade Level | Compensation grade (numeric) |
| Operating Title | Internal/standardized title |
| Work location - City | City |
| Work location - State | State |
| Work location - Country | Country |

---

## Conversion File Format (`wise_fx_all_to_USD.xlsx`)
| Column | Description |
|--------|-------------|
| `date` | Date of the exchange rate |
| `source_currency` | Currency being converted FROM (e.g., AED, INR) |
| `target_currency` | Always USD |
| `concat` | Excel serial date + currency code (helper key, not used in Python) |
| `exchange_rate` | Multiplier to convert source → USD |

**Lookup logic:** Filter where `source_currency == <currency_code>` AND `date <= as_of_date`, then pick the most recent row.

**Fallback date:** Configured as `FALLBACK_DATE` in the notebook (default: `2026-03-16`). Update manually when today's rate is not yet in the file.

---

## Cleaning Steps

### Step 1 — Remove Contractors
- **Column:** `Employment type name`
- **Action:** Drop all rows where value equals `Contractor`

### Step 2 — Remove Blank Rippling Profile Numbers
- **Column:** `Rippling profile number`
- **Action:** Drop rows where value is NaN or empty string

### Step 3 — Fill Blanks in Department and Work Location
- **Columns:** `Department`, `Work location`
- **Action:** Replace NaN or empty string with `Unclassified`

### Step 4 — Clean Work Location State
- **Column:** `Work location - State`
- **Keep state when:**
  - `Work location - Country` = `United States`, OR
  - `Work location - Country` = `India` AND `Work location - State` = `Karnataka`
- **All other records:** Set state to blank

### Step 5 — Normalize Grade Level
- **Column:** `Grade Level`
- **Action:** Strip `.0` decimal → `5.0 → 5`, `7.0 → 7`
- **Method:** `int(float(val))`

### Step 6 — Currency Conversion

#### 6a — Determine As-Of Date
| Employee Status | As-Of Date |
|-----------------|------------|
| Active (blank Last work date) | `AS_OF_DATE` from config cell (update before each run) |
| Terminated (has Last work date) | Value in `Last work date` column |

#### 6b — FX Lookup
- Convert the as-of date to an Excel serial number: `(date - date(1899,12,30)).days`
- Build the concat key: `str(serial) + currency_code` (e.g. `"46097INR"`)
- Exact-match lookup on `concat` column in `wise_fx_all_to_USD.xlsx`
- USD → no lookup needed, rate = 1.0

#### 6c — Convert Base Compensation to USD
| Condition | Formula |
|-----------|---------|
| Currency = USD + amount ≥ 500 | No conversion, use as-is |
| Currency = USD + amount < 500 + type in ANNUALIZE_TYPES | `amount × 30 × 52` |
| Non-USD + amount < 500 + type in ANNUALIZE_TYPES | `amount × 30 × 52 × rate` |
| Non-USD, all other | `amount × rate` |

**Currency used:** `Base compensation - Currency`

#### 6d — Convert On-Target Commission to USD
- If `On-target commission - Currency` = USD → use as-is
- Otherwise → `On-target commission × rate`
- **Currency used:** `On-target commission - Currency`

#### 6e — Target Annual Bonus Percent
- Use the `Target annual bonus percent` value **directly** — no currency conversion
- Just apply Step 6f rule below

#### 6f — ABP Zero Value Rule
- Applies to all employee types
- If `Target annual bonus percent` = 0 or blank → replace with `0.1`

---

### Step 7 — Assign Base Category Name
| Condition | Value |
|-----------|-------|
| Rippling profile number IN (235, 277, 284, 291) | `60010 - Salary and Wages - EOR` |
| All other employees | `60020 - Salary and Wages` |

Note: Compare Rippling profile number as string (strip `.0` from float IDs).

---

## Output Column Mapping (25 Columns)
| # | Output Column | Source | Rule |
|---|---------------|--------|------|
| 1 | Emp ID | Rippling profile number | As text; strip `.0` |
| 2 | Req ID | — | Always blank |
| 3 | Req opening ID | — | Always blank |
| 4 | Name (masked) | — | Literal: `FirstName` |
| 5 | Job Title | Title | Direct |
| 6 | Start Date | Start date | mm/dd/yyyy |
| 7 | Termination Date | Last work date | mm/dd/yyyy; blank if active |
| 8 | Department | Department | Step 3 applied |
| 9 | Location | Work location | Step 3 applied |
| 10 | Base Category Name | — | Step 7 |
| 11 | Bonus Category Name | — | Hardcoded: `60030 - Bonus` |
| 12 | Annual Base Pay | Base compensation | Step 6c; 4 decimal places |
| 13 | Bonus % | Target annual bonus percent | Step 6e+6f; 4 decimal places |
| 14 | Annualized Stock Comp | — | Always blank |
| 15 | Annualized Sales Comp | On-target commission | Step 6d; 4 decimal places |
| 16 | Grade | Grade Level | Step 5 integer |
| 17 | Supervisor | — | Always blank |
| 18 | Work City | Work location - City | Direct |
| 19 | Work State | Work location - State | Step 4 applied |
| 20 | Work Country | Work location - Country | Direct |
| 21 | Effective Start Date | — | Always blank |
| 22 | Effective End Date | — | Always blank |
| 23 | Employee Type | Employment type name | Direct; contractors removed in Step 1 |
| 24 | Hire Type | — | Always blank |
| 25 | Termination Type | — | Always blank |

---

## Edge Cases & Notes
- **EOR IDs** (235, 277, 284, 291) are hard-coded — verify these remain correct each run
- **Annualization** applies to Hourly, Part-time, Temporary, Intern **only when base pay < 500** (assumed to be hourly rate, not annual)
- **Bonus % = 0 or blank** → always replaced with `0.1` for all employee types
- **FX fallback:** If today's rate is missing, manually update `FALLBACK_DATE` in the notebook config cell
- **Grade Level:** Stored as float in Excel (e.g., `5.0`); cast to `int` in output
- **Rippling profile number** may be read as float by pandas (e.g., `235.0`); always convert to string and strip `.0`
