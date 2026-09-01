# Semantic Model Documentation — `port-teu-powerbi`

## Overview

| Property | Value |
|---|---|
| Model name | `Model` (database `port-teu-powerbi.SemanticModel`) |
| Compatibility level | 1606 |
| Culture | en-US |
| Storage mode | Import (all tables) |
| Time intelligence (auto date/time) | Disabled (`__PBI_TimeIntelligenceEnabled = 0`) — an explicit `DimDate` table is used instead |
| Tables | `Query Measures` (measure holder), `PortContainers` (fact), `DimDate` (dimension), `DimPort` (dimension) |

This model tracks weekly container traffic — in Twenty-foot Equivalent Units (TEUs) — at a set of major U.S. ports, sourced from a MARAD/CBP public dataset.

---

## Table relationships

    DimPort ||--o{ PortContainers : "Port = Port"
    DimDate ||--o{ PortContainers : "Date = DATE"

    DimPort {
        string Port "dimension key"
        string Coast "derived: West / Gulf / East"
        double Latitude
        double Longitude
    }

    DimDate {
        dateTime Date "dimension key"
        int64 Year
        int64 MonthNum
        string MonthName "sorted by MonthNum"
        string YearMonth
        string Quarter
    }

    PortContainers {
        dateTime DATE FK
        string Port FK
        double LoadedImportTEU
        double LoadedExportTEU
        double EmptyExportTEU
    }

Both relationships are the TMDL default: **many-to-one, single-direction cross-filtering, active**.

| Relationship | From (many) | To (one) |
|---|---|---|
| `AutoDetected_925a7a81…` | `PortContainers[Port]` | `DimPort[Port]` |
| `efa842c6-c99d…` | `PortContainers[DATE]` | `DimDate[Date]` |

`Query Measures` is a standalone, hidden-column table with no relationships — it exists purely to hold all DAX measures together in the Fields list. It has no data rows (its Power Query source returns an empty string).

---

## Data sources (inferred from Power Query)

### Parameter: `FilePath`
A text parameter holding the absolute path to the single source file:
```
C:\Users\drake\OneDrive\4Archive\Documents\2026\PBI\PortData.csv
```
> This is a local, machine-specific absolute path. Anyone else opening this .pbip (or moving the file) will need to update the parameter before refresh will succeed.

### `PortContainers` (fact table)
Source: `PortData.csv`, loaded via `Csv.Document(File.Contents(FilePath), …)`. The raw file is a long/narrow MARAD ("Maritime Administration") extract with one row per indicator/port/week, containing columns such as `ID`, `DATE`, `YEAR`, `INDICATOR`, `MEASURE1` (port name), `VALUE1`, `SOURCE`, plus notes and units that are discarded.

Transformation steps applied:
1. **Promote headers** from the first row.
2. **Remove columns** not needed for reporting (`MEASURE2`, descriptions, `UNITS`, `NOTE`, `YEAR`, `ID`, `SOURCE`).
3. **Filter rows** down to exactly three `INDICATOR` values:
   - "Empty Export Containers at Select U.S. Ports in TEUs"
   - "Loaded Export Containers at Select U.S. Ports in TEUs"
   - "Loaded Import Containers at Select U.S. Ports in TEUs"
4. **Type conversion**: `DATE` → datetime, `VALUE1` → number.
5. **Shorten indicator labels** to `LoadedImportTEU`, `LoadedExportTEU`, `EmptyExportTEU`.
6. **Pivot** the shortened indicator column so each becomes its own numeric column (aggregated with `List.Sum`), turning the long file into one row per date/port.
7. **Rename** `MEASURE1` → `Port`.
8. Re-type the three pivoted measure columns as numbers.

Result: one row per `DATE` + `Port`, with three TEU measure columns — this is the model's fact table.

### `DimPort` (dimension table)
Not sourced directly from the CSV file as a lookup — instead it is **derived from `PortContainers`**:
1. Take the distinct list of `Port` values already loaded into `PortContainers`.
2. Add a `Coast` column via hardcoded business logic: West coast ports (Los Angeles, Long Beach, Oakland, Sea‑Tac), Gulf (Houston), else East.
3. Add `Latitude`/`Longitude` via a hardcoded lookup record of coordinates, keyed by port name.

This means `DimPort` will only ever contain ports that actually appear in the source file, but the coast/coordinate logic is hardcoded and must be updated manually if a new port appears in a future data refresh.

### `DimDate` (dimension table)
Not sourced from any file — it is a **fully calculated calendar table** spanning `2019-01-01` to `2026-12-31` (one row per day), with `Year`, `MonthNum`, `MonthName` (sorted by `MonthNum`), `YearMonth`, and `Quarter` columns all derived via `Date.*` functions in M.

### `Query Measures`
No external source; its Power Query returns an empty string. It exists solely as a container for the model's measures.

---

## Measures

All measures live in the `Query Measures` table.

### `Rows`
```dax
Rows = COUNTROWS(PortContainers)
```
A simple row count of the fact table, used mainly for diagnostics (e.g., confirming the data loaded and filter context is behaving as expected) rather than for reporting.

### `Loaded Imports`
```dax
Loaded Imports = SUM(PortContainers[LoadedImportTEU])
```
Total inbound container volume (TEUs) that ports received as loaded (non-empty) cargo, for whatever ports/dates are in filter context.

### `Loaded Exports`
```dax
Loaded Exports = SUM(PortContainers[LoadedExportTEU])
```
Total outbound container volume (TEUs) shipped out as loaded cargo.

### `Empty Exports`
```dax
Empty Exports = SUM(PortContainers[EmptyExportTEU])
```
Total volume (TEUs) of **empty** containers shipped out — a classic trade-imbalance indicator: high empty-export volume signals more containers arriving full than are needed to carry outbound loaded goods.

### `Total Outbound`
```dax
Total Outbound = [Loaded Exports] + [Empty Exports]
```
All outbound container movement, loaded and empty combined — the total number of containers leaving a port regardless of whether they carry cargo.

### `Empty Share`
```dax
Empty Share = DIVIDE([Empty Exports], [Total Outbound])
```
The percentage of outbound containers that leave empty. A rising share is a common proxy for trade imbalance (more goods coming in than going out), and `DIVIDE` guards against a divide-by-zero blank when there's no outbound activity.

### `Import/Export Ratio`
```dax
Import/Export Ratio = DIVIDE([Loaded Imports], [Loaded Exports])
```
How many TEUs of loaded imports arrive for every TEU of loaded exports shipped. A ratio greater than 1 means imports outweigh exports.

### `Import/Export Ratio Display`
```dax
Import/Export Ratio Display = CONCATENATE(FORMAT([Import/Export Ratio], "Fixed"), ":1")
```
Formats the ratio above as a readable "X.XX:1" string for card/label visuals, so it reads naturally (e.g., "2.35:1") rather than as a raw decimal.

### `Loaded Imports Negative`
```dax
Loaded Imports Negative = -[Loaded Imports]
```
The imports total negated. Typically used to plot imports on the opposite side of a diverging/butterfly chart from exports (e.g., a back-to-back bar chart comparing inbound vs. outbound volume).

### `Total TEU`
```dax
Total TEU = [Loaded Imports] + [Total Outbound]
```
Grand total container throughput at a port: loaded imports plus all outbound movement (loaded exports + empty exports). Represents overall port activity/volume.

### `Loaded Imports YoY %`
```dax
Loaded Imports YoY % =
VAR Prior = CALCULATE([Loaded Imports], DATEADD(DimDate[Date], -12, MONTH))
RETURN DIVIDE([Loaded Imports] - Prior, Prior)
```
Year-over-year percentage change in loaded imports, comparing the current period to the same period 12 months earlier. Answers "is import volume growing or shrinking compared to last year?"

### `Imports 3M Avg`
```dax
Imports 3M Avg =
CALCULATE(
    AVERAGEX(VALUES(DimDate[YearMonth]), [Loaded Imports]),
    DATESINPERIOD(DimDate[Date], MAX(DimDate[Date]), -3, MONTH))
```
A rolling 3-month average of loaded imports, averaged across distinct calendar months in the trailing 3-month window. Smooths out weekly volatility to reveal the underlying trend.

### `Imports Index 2019=100`
```dax
Imports Index 2019 100 =
VAR Base =
    CALCULATE(
        AVERAGEX(VALUES(DimDate[YearMonth]), [Loaded Imports]),
        ALL(DimDate), DimDate[Year] = 2019
    )
RETURN DIVIDE([Loaded Imports], Base) * 100
```
An index of import volume relative to the 2019 monthly average baseline (2019 = 100), ignoring any date filters when computing the baseline. Lets users see growth/decline relative to pre-pandemic-era volumes regardless of what date range is currently selected — a value of 130 means imports are running 30% above the 2019 average.

### `Share of National Imports`
```dax
Share of National Imports = DIVIDE([Loaded Imports], CALCULATE([Loaded Imports], ALL(DimPort)))
```
What percentage of total (all-port) loaded import volume a given port (or selected set of ports) accounts for — removes any port filter for the denominator so each port can be compared against the national total.

---

## Row-level security

**No row-level security (RLS) roles are defined in this model.** There are no `role` objects in the TMDL definition and no security-role table filters. All users with access to this semantic model see all rows in `PortContainers`, `DimPort`, and `DimDate`.

> Note: the `en-US.tmdl` culture file contains Q&A **linguistic modeling** entries labeled `"Roles"` (e.g., `dim_port`, `port_container`) — these define natural-language synonyms/relationships for Power BI Q&A and are unrelated to row-level security.

---

## Conventions and notes

- Measures favor `DIVIDE()` over `/` throughout, avoiding divide-by-zero errors as blanks.
- `PortContainers`, `DimPort`, and `DimDate` are linked by two single-direction, many-to-one relationships (see diagram above); there is no bidirectional filtering in the model.
- `DimDate[MonthName]` is sorted by `DimDate[MonthNum]` so month names display in calendar order rather than alphabetically.
- `Query Measures[Query Measures]` is a hidden placeholder column with no reporting purpose beyond satisfying the table's need for at least one column.
