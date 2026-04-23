# Datasets, Columns, and Calculated Fields

These three shapes underpin every data-facing reference inside an `AnalysisDefinition`. Any field well, filter, sort config, or drill hierarchy refers to a column via `ColumnIdentifier`; every column identifier names a dataset declared in `DataSetIdentifierDeclarations`.

## DataSetIdentifierDeclaration

Required top-level entry. Declares that a dataset is usable in this analysis and assigns it a short identifier for internal references.

```json
{
  "Identifier": "sales",
  "DataSetArn": "arn:aws:quicksight:us-east-1:123456789012:dataset/sales-2025"
}
```

| Field        | Required | Constraint                                                                                                               |
| ------------ | -------- | ------------------------------------------------------------------------------------------------------------------------ |
| `Identifier` | ✅       | 1–2048 chars. Conventionally the dataset name. Used wherever a `DataSetIdentifier` is needed elsewhere in the definition |
| `DataSetArn` | ✅       | Valid QuickSight dataset ARN                                                                                             |

Per-analysis limits: 1–50 declarations. Every dataset referenced anywhere in the definition must have a matching declaration — the API will 400 on dangling references.

The `Identifier` is stable within the analysis definition; renaming it (without updating every downstream `ColumnIdentifier.DataSetIdentifier`) breaks the definition.

## ColumnIdentifier

Used wherever the definition refers to a specific column.

```json
{ "DataSetIdentifier": "sales", "ColumnName": "revenue" }
```

| Field               | Required | Constraint                                                                        |
| ------------------- | -------- | --------------------------------------------------------------------------------- |
| `DataSetIdentifier` | ✅       | 1–2048 chars. Must match one of `DataSetIdentifierDeclarations[].Identifier`      |
| `ColumnName`        | ✅       | 1–128 chars. The column name as it appears in the dataset schema — case-sensitive |

## CalculatedField

Analysis-scoped computed columns. Defined once at the top level, referenced from visuals and filters like any other column.

```json
{
  "DataSetIdentifier": "sales",
  "Name": "revenue_per_unit",
  "Expression": "sum({revenue}) / sum({units})"
}
```

| Field               | Required | Constraint                                                                                                                        |
| ------------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `DataSetIdentifier` | ✅       | 1–2048 chars. The dataset the expression is scoped to (columns in the expression must come from this dataset)                     |
| `Name`              | ✅       | 1–128 chars. Used as the column name in `ColumnIdentifier` (`{ "DataSetIdentifier": "sales", "ColumnName": "revenue_per_unit" }`) |
| `Expression`        | ✅       | 1–32000 chars. QuickSight expression language                                                                                     |

Per-analysis limit: 0–2000 calculated fields.

### Expression Language

QuickSight uses a SQL-ish expression language with functions grouped into:

- **Aggregate**: `sum`, `avg`, `count`, `distinct_count`, `min`, `max`, `median`, `stdev`, `stdevp`, `var`, `varp`, `percentile`, `percentileCont`, `percentileDisc`
- **Conditional**: `ifelse`, `coalesce`, `isNotNull`, `isNull`, `nullIf`, `in`, `notIn`
- **String**: `concat`, `contains`, `endsWith`, `startsWith`, `left`, `right`, `substring`, `replace`, `split`, `strlen`, `toLower`, `toUpper`, `trim`, `locate`, `parseInt`, `parseDecimal`
- **Numeric**: `abs`, `ceil`, `floor`, `round`, `exp`, `log`, `ln`, `sqrt`, `mod`, `decimalToInt`, `intToDecimal`
- **Date**: `now`, `truncDate`, `addDateTime`, `dateDiff`, `extract`, `formatDate`, `parseDate`, `epochDate`
- **Window/LAC-W** (Level-Aware Calculations – Window): `sumOver`, `avgOver`, `minOver`, `maxOver`, `countOver`, `denseRank`, `rank`, `firstValue`, `lastValue`, `lag`, `lead`, `runningSum`, `runningAvg`, `windowSum`, `windowAvg`, `percentOfTotal`, `percentileOver`
- **LAC-A** (Level-Aware Calculations – Aggregate): `sumIf`, `countIf`, `avgIf`, `minIf`, `maxIf`, `distinctCountIf`, `periodOverPeriodDifference`, `periodOverPeriodPercentage`, `periodToDateSum`, `periodToDateAvg`, `periodToDateMin`, `periodToDateMax`, `periodToDateCount`

Full function reference: https://docs.aws.amazon.com/quicksight/latest/user/working-with-calculated-fields.html

### Date function constraints

**`parseDate`** does not support the `XXX` pattern for timezone offsets (e.g., `+09:00`, `+00:00`). To parse an ISO 8601 string with offset, strip the offset first with `substring`:

```text
parseDate(substring({datetime}, 1, 19), "yyyy-MM-dd'T'HH:mm:ss")
```

Then apply timezone adjustment manually: `addDateTime(9, 'HH', parseDate(...))`.

**`formatDate`** does not support all Java SimpleDateFormat patterns:

- `"yyyy-MM"` is not supported — use `substring(formatDate({col}, "yyyy-MM-dd"), 1, 7)`.
- `formatDate` on the result of `addDateTime` may produce "unsupported date" errors. Using `toString` + `substring` is more reliable for display formatting:
  ```text
  concat(substring(toString({adjusted_time}), 1, 10), ' ', substring(toString({adjusted_time}), 12, 5))
  ```

### Expression Syntax

- Column references use curly braces: `{column_name}` — or `{calculated_field_name}` for another calculated field in the same dataset.
- Parameter references use `${ParameterName}`.
- String literals use single quotes: `'hello'`.
- Booleans: `true`, `false`.
- Null: `NULL`.
- Case-insensitive function names, but column names inside `{}` match dataset casing.

### Examples

```text
# Simple aggregate
sum({revenue})

# Ratio
sum({revenue}) / distinct_count({customer_id})

# Conditional
ifelse({status} = 'shipped', {amount}, 0)

# Parameter-driven
ifelse({order_date} >= ${FromDate} AND {order_date} <= ${ToDate}, {revenue}, NULL)

# Date truncation + YoY
(sum({revenue}) - periodOverPeriodLastValue(sum({revenue}), {order_date}, YEAR)) / periodOverPeriodLastValue(sum({revenue}), {order_date}, YEAR)

# Window running total
runningSum(sum({revenue}), [{order_date} ASC])
```

### Aggregation context

QuickSight distinguishes between aggregated and non-aggregated expressions. A calculated field used as a **measure** (in a value field well, or referenced by `CalculatedMeasureField`) must be aggregate-valued — either the whole expression is aggregated (`sum({x})`) or QuickSight auto-aggregates in certain visual types.

A calculated field used as a **dimension** must be row-level (no aggregate functions).

Mixing aggregated and row-level references in one expression is generally a definition error — QuickSight rejects the definition.

### LAC-W fields cannot be re-aggregated in visuals

LAC-W calculated fields (e.g., `runningSum`, `sumOver`) are already aggregated. Referencing them in a visual via `NumericalMeasureField` with an `AggregationFunction` causes `INVALID_COLUMN_AGGREGATION`. Either use `CalculatedMeasureField` with the expression inline, or restructure as a row-level calculated field and aggregate in the visual.

### Output type determines MeasureField/DimensionField wrapper

When referencing a calculated field in a visual's field well, the wrapper type must match the calculated field's **output** data type:

| Calculated field output                     | Use as dimension            | Use as measure            |
| ------------------------------------------- | --------------------------- | ------------------------- |
| String (e.g., `ifelse(...)` returning text) | `CategoricalDimensionField` | `CategoricalMeasureField` |
| Number (e.g., `sum({x}) / sum({y})`)        | `NumericalDimensionField`   | `NumericalMeasureField`   |
| DateTime (e.g., `truncDate(...)`)           | `DateDimensionField`        | `DateMeasureField`        |

Mismatch causes `COLUMN_TYPE_INCOMPATIBLE` at deploy time.

## Cross-Dataset Calculated Fields

Not supported directly. A `CalculatedField` is scoped to exactly one dataset. To combine data across datasets, create a dataset with a join in Data Prep (outside the analysis) and reference that composite dataset.

## ColumnConfiguration

Not a calculated field, but related — `ColumnConfiguration` at the top level of `AnalysisDefinition` supplies default formatting/role for a column (raw or calculated) wherever it appears in visuals. See [definition-top-level.md](./definition-top-level.md).

```json
{
  "Column": { "DataSetIdentifier": "sales", "ColumnName": "revenue_per_unit" },
  "Role": "MEASURE",
  "FormatConfiguration": {
    "NumberFormatConfiguration": {
      "FormatConfiguration": {
        "CurrencyDisplayFormatConfiguration": {
          "Prefix": "$",
          "DecimalPlacesConfiguration": { "DecimalPlaces": 2 }
        }
      }
    }
  }
}
```

Applying `ColumnConfiguration` saves you from repeating format options in every visual that uses the column.
