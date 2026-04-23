# Parameters

Analysis parameters are typed named variables declared at the top level and referenced from filters, visual titles, URL actions, and calculated fields. They're the primary mechanism for interactivity beyond raw filters.

`ParameterDeclaration` is a union of four variants — one per supported data type.

## ParameterDeclaration (union of 4)

```json
{
  "StringParameterDeclaration":   { ... },
  "IntegerParameterDeclaration":  { ... },
  "DecimalParameterDeclaration":  { ... },
  "DateTimeParameterDeclaration": { ... }
}
```

Every variant shares this field shape:

| Field                     | Required | Notes                                                                      |
| ------------------------- | -------- | -------------------------------------------------------------------------- |
| `Name`                    | ✅       | 1–2048 chars, pattern `^[a-zA-Z0-9]+$` (alphanumeric only — no `_` or `-`) |
| `ParameterValueType`      | ✅       | `SINGLE_VALUED` or `MULTI_VALUED`                                          |
| `DefaultValues`           | ❌       | Type-specific default values object                                        |
| `ValueWhenUnset`          | ❌       | Behavior when no value and no default is selected                          |
| `MappedDataSetParameters` | ❌       | 0–150 items — links analysis parameter to dataset parameters               |

`MULTI_VALUED` parameters accept arrays of values (used with multi-select UI controls). `SINGLE_VALUED` accepts one — and `DefaultValues` must contain at most one value in that case.

### StringParameterDeclaration

```json
{
  "StringParameterDeclaration": {
    "Name":               "SelectedRegion",
    "ParameterValueType": "SINGLE_VALUED",
    "DefaultValues":      { "StaticValues": ["APAC"], "DynamicValue": { "DefaultValueColumn": { ... }, "UserNameColumn": { ... }, "GroupNameColumn": { ... } } },
    "ValueWhenUnset":     { "ValueWhenUnsetOption": "RECOMMENDED_VALUE", "CustomValue": "ALL" },
    "MappedDataSetParameters": [
      { "DataSetIdentifier": "sales", "DataSetParameterName": "RegionParam" }
    ]
  }
}
```

### IntegerParameterDeclaration

```json
{
  "IntegerParameterDeclaration": {
    "Name":               "TopN",
    "ParameterValueType": "SINGLE_VALUED",
    "DefaultValues":      { "StaticValues": [10], "DynamicValue": { ... } },
    "ValueWhenUnset":     { "ValueWhenUnsetOption": "NULL" },
    "MappedDataSetParameters": [ ... ]
  }
}
```

### DecimalParameterDeclaration

```json
{
  "DecimalParameterDeclaration": {
    "Name":               "Threshold",
    "ParameterValueType": "SINGLE_VALUED",
    "DefaultValues":      { "StaticValues": [0.95], "DynamicValue": { ... } },
    "ValueWhenUnset":     { "ValueWhenUnsetOption": "RECOMMENDED_VALUE", "CustomValue": 0.5 },
    "MappedDataSetParameters": [ ... ]
  }
}
```

### DateTimeParameterDeclaration

```json
{
  "DateTimeParameterDeclaration": {
    "Name":               "AsOfDate",
    "TimeGranularity":    "YEAR" | "QUARTER" | "MONTH" | "WEEK" | "DAY" | "HOUR" | "MINUTE" | "SECOND" | "MILLISECOND",
    "DefaultValues":      {
      "StaticValues":  ["2024-05-03T00:00:00.000Z"],
      "RollingDate":   { "DataSetIdentifier": "sales", "Expression": "truncDate('MM', now())" },
      "DynamicValue":  { ... }
    },
    "ValueWhenUnset":     { "ValueWhenUnsetOption": "RECOMMENDED_VALUE", "CustomValue": "2023-11-14T00:00:00.000Z" },
    "MappedDataSetParameters": [ ... ]
  }
}
```

> `DateTimeParameterDeclaration` has no `ParameterValueType` field. DateTime parameters are implicitly single-valued.

`DateTimeParameterDeclaration` is the only variant with a `TimeGranularity` field and a `RollingDate` default option (dynamically re-evaluated at query time).

> **⚠️ DateTime values must be ISO 8601 strings** (e.g., `"2024-05-03T00:00:00.000Z"`), not epoch integers. This applies to `StaticValues`, `CustomValue` in `ValueWhenUnset`, and DateTime filter values.

## DefaultValues shapes

Each `DefaultValues` variant is itself a union of sources:

| Source         | What                                                                                                         |
| -------------- | ------------------------------------------------------------------------------------------------------------ |
| `StaticValues` | Hardcoded literal(s). Array of values (1 item for `SINGLE_VALUED`, 1+ for `MULTI_VALUED`).                   |
| `DynamicValue` | Look the default up in a dataset column. Optionally keyed by current username / group for per-user defaults. |
| `RollingDate`  | (`DateTimeParameterDeclaration` only.) Expression evaluated at query time (e.g., `truncDate('MM', now())`).  |

> **⚠️ `RollingDate.Expression` uses a restricted expression syntax.** Only a subset of functions is available — primarily `truncDate`, `now`, `addDateTime`, and `dateDiff`. The syntax differs from `CalculatedField` expressions: for example, `addDateTime` in a `RollingDate.Expression` uses the same function signature but the expression is evaluated in a different context (no column references, no parameters). Complex expressions that work in calculated fields may fail with `CALCULATION_EXPRESSION_SYNTAX_ERROR` in `RollingDate`. Keep rolling date expressions simple — e.g., `truncDate('DD', now())`, `addDateTime(-7, 'DD', now())`.

A `DynamicValue` configuration:

```json
{
  "DefaultValueColumn": {
    "DataSetIdentifier": "users",
    "ColumnName": "default_region"
  },
  "UserNameColumn": { "DataSetIdentifier": "users", "ColumnName": "email" },
  "GroupNameColumn": { "DataSetIdentifier": "users", "ColumnName": "team" }
}
```

At query time, QuickSight joins the caller's QuickSight username (or group) against `UserNameColumn` / `GroupNameColumn` and returns the value in `DefaultValueColumn`.

## ValueWhenUnset

Controls what happens when no default is set and the user has not picked a value.

```json
{ "ValueWhenUnsetOption": "RECOMMENDED_VALUE", "CustomValue": "ALL" }
// OR
{ "ValueWhenUnsetOption": "NULL" }
```

- `RECOMMENDED_VALUE` — use `CustomValue` as a fallback. **`CustomValue` is required** when this option is set.
- `NULL` — pass through as a null parameter value (filters may then select all or nothing depending on `NullOption`). **Do not include `CustomValue`** with this option.

> **⚠️ `ValueWhenUnsetOption` and `CustomValue` are coupled, not independent.** Use `CustomValue` only with `RECOMMENDED_VALUE`. Sending `CustomValue` with `NULL` (or sending `RECOMMENDED_VALUE` without `CustomValue`) causes `InvalidParameterValueException`.

`CustomValue`'s type matches the parameter type (string, integer, decimal, DateTime as ISO 8601 string).

## MappedDataSetParameters

Bridges an **analysis parameter** to a **dataset parameter**. Dataset parameters (declared on the dataset, not the analysis) are typed variables inside SPICE-managed query SQL. Mapping lets a single analysis parameter change what SQL hits each dataset.

```json
{ "DataSetIdentifier": "sales", "DataSetParameterName": "RegionParam" }
```

Up to 150 mappings per analysis parameter (one per dataset).

### MULTI_VALUED constraints

- `MULTI_VALUED` parameters cannot be used with `CustomFilterConfiguration` (which accepts a single `ParameterName` for a single value). Use `FilterListConfiguration` or `CustomFilterListConfiguration` instead.
- `DefaultValues.StaticValues` can contain multiple values for `MULTI_VALUED`; must contain at most 1 for `SINGLE_VALUED`.

## Referencing a parameter

From filters: any filter with a `ParameterName` field points to the parameter by `Name`.

From visual titles / text boxes / URL actions: use the syntax `<<$ParameterName>>` inside the content string. Example:

```json
{
  "Title": {
    "Visibility": "VISIBLE",
    "FormatText": { "PlainText": "Sales for <<$SelectedRegion>>" }
  }
}
```

From calculated fields: use `${ParameterName}` inside the expression, with dataset columns in `{}`. Example:

```json
{
  "DataSetIdentifier": "sales",
  "Name": "SalesInWindow",
  "Expression": "ifelse({order_date} >= ${FromDate} AND {order_date} <= ${ToDate}, {revenue}, 0)"
}
```

## End-to-end: binding a parameter to a filter

The common pattern: declare a parameter, wire a `CategoryFilter.CustomFilterConfiguration.ParameterName` to it, and optionally expose it as a `ParameterControl` for the viewer.

```json
// 1. ParameterDeclaration (AnalysisDefinition.ParameterDeclarations[])
{
  "StringParameterDeclaration": {
    "Name": "SelectedRegion",
    "ParameterValueType": "SINGLE_VALUED",
    "DefaultValues": { "StaticValues": ["APAC"] }
  }
}

// 2. FilterGroup.Filters[] uses ParameterName to bind
{
  "CategoryFilter": {
    "FilterId": "f-region",
    "Column": { "DataSetIdentifier": "sales", "ColumnName": "region" },
    "Configuration": {
      "CustomFilterConfiguration": {
        "MatchOperator": "EQUALS",
        "ParameterName": "SelectedRegion",
        "NullOption": "NON_NULLS_ONLY"
      }
    }
  }
}

// 3. ParameterControl on a sheet (optional — makes the parameter user-editable)
// SheetDefinition.ParameterControls[]:
{
  "Dropdown": {
    "ParameterControlId": "pc-region",
    "Title": "Region",
    "SourceParameterName": "SelectedRegion",
    "SelectableValues": { "Values": ["APAC", "EMEA", "NA"] }
  }
}
```

Without step 3, `SelectedRegion` still has a value (the default, or a value passed via `Parameters` on `CreateAnalysis` / `UpdateAnalysis`, or via URL query string `?p.SelectedRegion=EMEA`). Step 3 adds the in-sheet UI that lets a viewer change it interactively.

## ParameterControl (reminder)

UI rendering for a parameter is defined on each sheet via `SheetDefinition.ParameterControls[]`. See [definition-sheets-layouts.md](./definition-sheets-layouts.md). A parameter with no parameter control has no visible UI — values can still be set via URL query strings or `Parameters` on `CreateAnalysis`/`UpdateAnalysis`.

## Hard limits

| Limit                                     | Value                                                 |
| ----------------------------------------- | ----------------------------------------------------- |
| `ParameterDeclarations` per analysis      | 0–400                                                 |
| `MappedDataSetParameters` per declaration | 0–150                                                 |
| `Name` length                             | 1–2048                                                |
| `Name` pattern                            | `^[a-zA-Z0-9]+$` — no underscores, hyphens, or spaces |
