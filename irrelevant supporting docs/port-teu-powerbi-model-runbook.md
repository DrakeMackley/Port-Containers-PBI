# Runbook: DOT Supply Chain Indicators CSV → Power BI Port Container Model

**Audience:** an agent reproducing this build from a fresh copy of the source file in a clean Power BI Desktop environment.
**Outcome:** a three-table star schema (one fact, two dimensions) reporting loaded and empty container volumes at nine U.S. ports, monthly, Jan 2019 onward.

---

## 1. Source characteristics

The source is `PortData.csv`, the U.S. Department of Transportation / BTS **Supply Chain and Freight Indicators** extract.

| Property | Value |
|---|---|
| Rows | ~26,400 (grows over time) |
| Columns | 12 |
| Encoding | UTF-8 (65001) |
| Delimiter | Comma |
| Distinct values in `INDICATOR` | ~44 |

Column semantics:

- `INDICATOR` — the series name; the primary record descriptor
- `NOTE` — extended description / methodology caveats for the series
- `VALUE1` + `UNITS` — the metric and its unit
- `MEASURE1` + `MEASURE1_DESCRIPTION` — the primary categorical breakdown and its label
- `MEASURE2` + `MEASURE2_DESCRIPTION` — a secondary breakdown, unused by the series in scope
- `ID`, `DATE`, `YEAR`, `SOURCE` — key, timestamp, year, attribution

**Critical formatting traits — verify these on a fresh file before proceeding:**

1. Every field is double-quoted. `NOTE` contains embedded commas, so quote handling is not optional.
2. `VALUE1` is comma-separated text (`"1,692,044"`). In the series in scope, **100% of rows** contain a thousands separator.
3. `VALUE1` is **not** integral — some values carry decimals (`"429,922.75"`).
4. `YEAR` is also comma-separated (`"2,019"`), which is why it is discarded rather than parsed.
5. `DATE` is text in the form `2019 Jan 06 12:00:00 AM`. The time component is always midnight.
6. The file mixes grains (weekly and monthly series) and units (TEUs, vessel counts, dollars, MPH, index values) in one flat table.

---

## 2. Scope decision

Include exactly these three indicators:

- `Loaded Import Containers at Select U.S. Ports in TEUs`
- `Loaded Export Containers at Select U.S. Ports in TEUs`
- `Empty Export Containers at Select U.S. Ports in TEUs`

They share grain (monthly), unit (TEUs), and dimension (`MEASURE1` = port name), which is what makes them combinable in a single fact table. Each covers 9 ports × 90 months with no gaps and no nulls.

**Explicitly exclude** `Empty Export Containers (in TEUs), Aggregated Select U.S. Ports`.

It is a pre-computed rollup of the per-port empty-export series — it matches the sum of the nine ports in 82 of 90 months, differing only by minor revisions. Its `MEASURE1` is null, so it cannot join to a port dimension, and it sits in the same column as its own components. Including it double-counts. Derive the national total in DAX with `SUM` instead.

Every other indicator in the file is rail, trucking, labor, fuel, or macroeconomic, or has a different grain. Do not mix them into this fact table.

---

## 3. Transformation sequence

Order matters. Each step exists for a stated reason.

| # | Step | Reason |
|---|---|---|
| 1 | `Csv.Document` with `QuoteStyle = QuoteStyle.Csv` | `NOTE` contains commas inside quoted strings. Without explicit quote handling, rows shift or error against the pinned `Columns = 12`. |
| 2 | `Table.PromoteHeaders` | Standard. |
| 3 | `Table.RemoveColumns` — drop `ID`, `YEAR`, `MEASURE2`, `MEASURE1_DESCRIPTION`, `MEASURE2_DESCRIPTION`, `UNITS`, `NOTE`, `SOURCE` | `ID` is `indicator_date_port` concatenated — unique across every row, making it the highest-cardinality column in the model, while duplicating information already in Date + Port + Indicator. `SOURCE`, `UNITS`, and `MEASURE1_DESCRIPTION` are single-valued across the filtered set. `MEASURE2`, `MEASURE2_DESCRIPTION`, and `NOTE` are 100% null. `YEAR` is comma-formatted text and is better derived in the date table. |
| 4 | `Table.SelectRows` — filter to the three indicators | **Must precede type conversion.** Typing first runs conversion across ~26,400 rows spanning other units and formats, risking errors from rows that are about to be discarded. Filtering first reduces the set to 2,430. |
| 5 | `Table.TransformColumnTypes` **with `"en-US"` culture** | Non-negotiable. Every `VALUE1` in scope has a thousands separator and the `DATE` format is `yyyy MMM dd hh:mm:ss tt`. Without an explicit culture the function inherits locale at runtime — the query then works in Desktop but returns **nulls, not errors**, when refreshed under different locale settings. Use `type number` for `VALUE1`, never `Int64.Type` (decimals present). |
| 6 | `Table.TransformColumns` — shorten `INDICATOR` values | The full names are unusable as column headers and slicer labels. |
| 7 | `Table.Pivot` on `INDICATOR` with `List.Sum` | Collapses 2,430 long rows to 810 wide rows, one per port-month. Date + Port is already a unique key, so the aggregation never actually aggregates — if it ever does, the grain has changed upstream and should be investigated. |
| 8 | `Table.RenameColumns` — `MEASURE1` → `Port` | Field-list legibility. |
| 9 | `Table.TransformColumnTypes` on the pivoted columns | `Table.Pivot` returns untyped columns, which can load as text and silently break aggregation. |

### Note on the pivot value list

Hardcode `{"LoadedImportTEU", "LoadedExportTEU", "EmptyExportTEU"}` rather than deriving it with `List.Distinct`. This yields deterministic column order and causes the query to fail loudly if BTS renames a series, instead of quietly producing an unexpected column.

### Note on `DATE` typing

`DATE` is left as `datetime`. Every value is midnight, so converting to `type date` via `Table.TransformColumns(prev, {{"DATE", DateTime.Date, type date}})` is safe and marginally cleaner. If you convert it, the date dimension's `Date` column must also be `type date`. **A datetime-to-date relationship loads without complaint and then drops non-matching rows** — keep both sides on the same type either way.

M identifiers are case-sensitive. If the column is `DATE`, the step must reference `"DATE"`, not `"Date"`.

---

## 4. Build order, naming, and the file path

### 4.1 Build order

Queries must be created in this order. `DimPort` reads from `PortContainers`, so the fact query must exist and be correctly named first.

1. `FilePath` (parameter)
2. `PortContainers` (fact)
3. `DimPort` (references `PortContainers`)
4. `DimDate` (independent — order relative to the others does not matter)

### 4.2 Query naming — required, not cosmetic

Power Query names a new query after its source file, so importing `PortData.csv` produces a query named **`PortData`**. The names in this runbook are load-bearing: in M, a query name is a global identifier, and `DimPort` resolves `PortContainers` by that name. If the fact query is still called `PortData`, `DimPort` fails with an identifier-not-found error.

After creating the fact query, rename it to `PortContainers` in the Queries pane (right-click → Rename, or edit the Name box in Query Settings) **before** creating `DimPort`.

If you prefer to keep the default name, that is fine — but then change the first line of `DimPort` to match:

```m
Source = Table.Distinct(Table.SelectColumns(PortData, {"Port"})),
```

A query name containing spaces or punctuation must be escaped in a reference as `#"Query Name"`. Avoid spaces to keep references clean.

### 4.3 The file path

Create a Power Query parameter rather than hardcoding the path. This isolates the one environment-specific value in the whole solution, so a fresh environment requires exactly one edit in a known location instead of a search through query code.

**Home → Manage Parameters → New Parameter:**

| Field | Value |
|---|---|
| Name | `FilePath` |
| Type | Text |
| Suggested Values | Any value |
| Current Value | the absolute path to `PortData.csv` on this machine |

Example current value: `C:\Users\<user>\OneDrive\Documents\PBI\PortData.csv`

The path must point at a file that exists before any query is evaluated — Power Query resolves `File.Contents` eagerly, so a placeholder throws immediately on refresh rather than at load time.

To confirm the parameter is set correctly before building anything on it, create a throwaway blank query with `= File.Contents(FilePath)` and check it returns binary content rather than a `DataSource.Error`. Delete it afterward.

## 4.4 Query: `PortContainers` (fact table)

```m
let
  Source = Csv.Document(File.Contents(FilePath), [Delimiter = ",", QuoteStyle = QuoteStyle.Csv, Columns = 12, Encoding = 65001]),
  #"Promoted headers" = Table.PromoteHeaders(Source, [PromoteAllScalars = true]),
  #"Removed Columns" = Table.RemoveColumns(#"Promoted headers", {"MEASURE2", "MEASURE1_DESCRIPTION", "MEASURE2_DESCRIPTION", "UNITS", "NOTE", "YEAR", "ID", "SOURCE"}),
  #"Filtered Rows" = Table.SelectRows(#"Removed Columns", each ([INDICATOR] = "Empty Export Containers at Select U.S. Ports in TEUs" or [INDICATOR] = "Loaded Export Containers at Select U.S. Ports in TEUs" or [INDICATOR] = "Loaded Import Containers at Select U.S. Ports in TEUs")),
  #"Changed column type" = Table.TransformColumnTypes(#"Filtered Rows", {{"DATE", type datetime}, {"INDICATOR", type text}, {"MEASURE1", type text}, {"VALUE1", type number}}, "en-US"),
  #"Shortened indicator" = Table.TransformColumns(#"Changed column type", {{"INDICATOR", each
      if Text.StartsWith(_, "Loaded Import") then "LoadedImportTEU"
      else if Text.StartsWith(_, "Loaded Export") then "LoadedExportTEU"
      else "EmptyExportTEU", type text}}),
  #"Pivoted indicator" = Table.Pivot(#"Shortened indicator", {"LoadedImportTEU", "LoadedExportTEU", "EmptyExportTEU"}, "INDICATOR", "VALUE1", List.Sum),
  #"Renamed port" = Table.RenameColumns(#"Pivoted indicator", {{"MEASURE1", "Port"}}),
  #"Typed measures" = Table.TransformColumnTypes(#"Renamed port", {{"LoadedImportTEU", type number}, {"LoadedExportTEU", type number}, {"EmptyExportTEU", type number}})
in
  #"Typed measures"
```

Result: `DATE`, `Port`, `LoadedImportTEU`, `LoadedExportTEU`, `EmptyExportTEU`.

---

## 5. Query: `DimDate`

```m
let
  Start = #date(2019, 1, 1),
  End = #date(2026, 12, 31),
  Dates = List.Dates(Start, Duration.Days(End - Start) + 1, #duration(1, 0, 0, 0)),
  ToTable = Table.FromList(Dates, Splitter.SplitByNothing(), {"Date"}),
  Typed = Table.TransformColumnTypes(ToTable, {{"Date", type date}}),
  Added = Table.AddColumn(Typed, "Year", each Date.Year([Date]), Int64.Type),
  Added2 = Table.AddColumn(Added, "MonthNum", each Date.Month([Date]), Int64.Type),
  Added3 = Table.AddColumn(Added2, "MonthName", each Date.ToText([Date], "MMM", "en-US"), type text),
  Added4 = Table.AddColumn(Added3, "YearMonth", each Date.ToText([Date], "yyyy-MM", "en-US"), type text),
  Added5 = Table.AddColumn(Added4, "Quarter", each "Q" & Text.From(Date.QuarterOfYear([Date])), type text)
in
  Added5
```

Extend `End` beyond the data range deliberately — it prevents time intelligence measures from breaking at the boundary. Sort `MonthName` by `MonthNum` in the model, and mark the table as a date table.

---

## 6. Query: `DimPort`

Reference `PortContainers` rather than duplicating the source query, so the file is read once.

This query will not evaluate until the fact query has been renamed to `PortContainers` (see 4.2).

```m
let
  Source = Table.Distinct(Table.SelectColumns(PortContainers, {"Port"})),
  #"Added coast" = Table.AddColumn(Source, "Coast", each
      if List.Contains({"Los Angeles", "Long Beach", "Oakland", "Sea-Tac"}, [Port]) then "West"
      else if [Port] = "Houston" then "Gulf"
      else "East", type text)
in
  #"Added coast"
```

Nine rows. This is also where latitude and longitude belong if a map visual is wanted — `NY-NJ`, `Sea-Tac`, and `Port of VA` will not geocode reliably from their names alone.

The nine ports: Charleston, Houston, Long Beach, Los Angeles, NY-NJ, Oakland, Port of VA, Savannah, Sea-Tac.

---

## 7. Model

| Relationship | Cardinality | Direction |
|---|---|---|
| `DimDate[Date]` → `PortContainers[DATE]` | One-to-many | Single |
| `DimPort[Port]` → `PortContainers[Port]` | One-to-many | Single |

Hide `PortContainers[DATE]` and `PortContainers[Port]` from report view so users slice on the dimension tables.

Base measures:

```dax
Loaded Imports = SUM(PortContainers[LoadedImportTEU])
Loaded Exports = SUM(PortContainers[LoadedExportTEU])
Empty Exports  = SUM(PortContainers[EmptyExportTEU])

Total Outbound = [Loaded Exports] + [Empty Exports]
Empty Share    = DIVIDE([Empty Exports], [Total Outbound])
Import/Export Ratio = DIVIDE([Loaded Imports], [Loaded Exports])
```

---

## 8. Validation checkpoints

After load, confirm all of the following. Figures are for the file as extracted through **June 2026**; if the source file is newer, row counts and totals will be higher, but the row/column structure and null count must still hold.

**Structure**

- 810 rows, 5 columns in `PortContainers`
- Zero nulls in every column
- 9 distinct ports, 90 distinct months
- Date range 2019-01-01 through 2026-06-01

**Values** — first row sorted by date then port:

| DATE | Port | LoadedImportTEU | LoadedExportTEU | EmptyExportTEU |
|---|---|---|---|---|
| 2019-01-01 | Charleston | 88,107 | 63,750 | 45,712 |

**Column totals across all 810 rows:**

| Column | Total |
|---|---|
| LoadedImportTEU | 171,876,731.25 |
| LoadedExportTEU | 75,913,801.85 |
| EmptyExportTEU | 97,788,374.85 |

The decimal places in these totals are the real test — if they match exactly, comma parsing and culture handling are both correct.

**Failure diagnosis:**

| Symptom | Cause |
|---|---|
| 810 rows but nulls in TEU columns | Culture argument missing or wrong; commas not parsed |
| More than 810 rows | Pivot did not collapse; grain changed upstream, or the aggregate indicator was not excluded |
| Fewer than 810 rows | Filter list wrong, or source file truncated |
| Row count near 2,430 | Pivot step missing |
| Errors on the type-change step | Filter placed after typing, or `Int64.Type` used on `VALUE1` |
| `DimPort` errors: identifier `PortContainers` not found | Fact query still has its default name (`PortData`); rename it, or update the reference in `DimPort` |
| `DataSource.Error` on any query | `FilePath` parameter is unset, still a placeholder, or points at a moved file |
| Totals off by a wide margin | Aggregate rollup indicator included in the filter |

---

## 9. Sanity check on the output

Empty containers should run near 70% of total outbound volume at Los Angeles, Long Beach, and NY-NJ, and roughly 25–35% at Houston, Oakland, and Port of VA. Import-to-export ratios should span about 1:1 at Houston to about 4:1 at Los Angeles.

This is the structural trade imbalance — large import gateways ship empties back to Asia while export-oriented ports move loaded cargo. If the report does not show this spread, the transformation is wrong somewhere.

---

## 10. Deployment

`File.Contents` against a local or OneDrive-synced path is a local filesystem reference. A published report **will not refresh in the Power BI Service** without an on-premises data gateway.

For unattended refresh, replace the `Source` step with `SharePoint.Files` or the OneDrive web connector and authenticate with an organizational account. All downstream steps are unchanged.

Because the path lives in the `FilePath` parameter rather than in query code, it can also be overridden per environment from the dataset settings in the Service without editing the model.

---

## 11. Out of scope but adjacent

If the model is later extended, these indicators are also maritime but require separate fact tables — they differ in grain, unit, or dimension:

- `Capacity of Containerships Calling at U.S. Ports (in TEUs)` — weekly, national. Note a methodology break: the series switched from IHS AIS data to CBP Vessel Management System data, which counts only vessels entering for an official reason. The level drops at the switch; this is not a real decline.
- `Number of Containerships Anchored off U.S. Ports` — weekly, by East/Gulf/West region, 2021+
- `Number of Containerships Awaiting Berths at all U.S. Ports` — weekly, 2021+. Contains an `All U.S. Container Ports` total in the same column as its components; same double-count hazard as the aggregate empty-export series.
- `Containerized Imports / Exports at U.S. Ports in TEUs` — monthly, national, PIERS-sourced. Different source than the per-port BTS series, so the two will not reconcile exactly.
- `Freight Rates in $ per 40ft Container` (two series) — monthly, dollars
- `Downbound Grain Barge Rates` — inland waterway, by river point, dollars per ton
