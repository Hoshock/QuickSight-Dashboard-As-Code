# Filters

Filters live in `AnalysisDefinition.FilterGroups[]`. A filter group bundles one or more filters with a shared scope (which sheets/visuals they apply to). Each filter is a union type — 8 variants.

## FilterGroup

```json
{
  "FilterGroupId":       "string",
  "CrossDataset":        "ALL_DATASETS" | "SINGLE_DATASET",
  "Filters":             [ /* Filter[] — 0–20 */ ],
  "ScopeConfiguration":  {
    "SelectedSheets": {
      "SheetVisualScopingConfigurations": [
        {
          "SheetId":    "string",
          "Scope":      "ALL_VISUALS" | "SELECTED_VISUALS",
          "VisualIds":  ["string"]
        }
      ]
    },
    "AllSheets": {}
  },
  "Status":              "ENABLED" | "DISABLED"
}
```

| Field                | Required | Notes                                                                                                                    |
| -------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------ |
| `FilterGroupId`      | ✅       | 1–512 chars, pattern `[\w\-]+`                                                                                           |
| `CrossDataset`       | ✅       | `ALL_DATASETS` applies across any dataset with a matching column; `SINGLE_DATASET` restricts to the filter's own dataset |
| `Filters`            | ✅       | 0–20 items                                                                                                               |
| `ScopeConfiguration` | ✅       | Union — one of `SelectedSheets` / `AllSheets`                                                                            |
| `Status`             | ❌       | Defaults to `ENABLED`                                                                                                    |

`ScopeConfiguration.AllSheets = {}` means the group applies to every sheet. `SelectedSheets` scopes to specific sheets and optionally specific visuals within each.

## Filter (union of 8 variants)

A single `Filter` is exactly one of:

```json
{
  "CategoryFilter":       { ... },
  "NumericEqualityFilter":{ ... },
  "NumericRangeFilter":   { ... },
  "TimeEqualityFilter":   { ... },
  "TimeRangeFilter":      { ... },
  "RelativeDatesFilter":  { ... },
  "TopBottomFilter":      { ... },
  "NestedFilter":         { ... }
}
```

All eight share three common fields: `FilterId` (required, 1–512 chars, pattern `[\w\-]+`), `Column` (required, `ColumnIdentifier`), and `DefaultFilterControlConfiguration` (optional — default control appearance when the filter is cross-sheet).

### 1. CategoryFilter — text match

```json
{
  "CategoryFilter": {
    "FilterId": "filter-region",
    "Column":   { "DataSetIdentifier": "sales", "ColumnName": "region" },
    "Configuration": {
      "FilterListConfiguration": {
        "MatchOperator":     "CONTAINS" | "DOES_NOT_CONTAIN",
        "CategoryValues":    ["APAC", "EMEA"],
        "SelectAllOptions":  "FILTER_ALL_VALUES",
        "NullOption":        "ALL_VALUES" | "NULLS_ONLY" | "NON_NULLS_ONLY"
      },
      "CustomFilterListConfiguration": {
        "MatchOperator":    "EQUALS" | "DOES_NOT_EQUAL" | "CONTAINS" | "DOES_NOT_CONTAIN" | "STARTS_WITH" | "ENDS_WITH",
        "CategoryValues":   ["..."],
        "SelectAllOptions": "FILTER_ALL_VALUES",
        "NullOption":       "ALL_VALUES" | "NULLS_ONLY" | "NON_NULLS_ONLY"
      },
      "CustomFilterConfiguration": {
        "MatchOperator":    "EQUALS" | "DOES_NOT_EQUAL" | "CONTAINS" | "DOES_NOT_CONTAIN" | "STARTS_WITH" | "ENDS_WITH",
        "CategoryValue":    "APAC",
        "ParameterName":    "SelectedRegion",
        "SelectAllOptions": "FILTER_ALL_VALUES",
        "NullOption":       "ALL_VALUES" | "NULLS_ONLY" | "NON_NULLS_ONLY"
      }
    }
  }
}
```

`Configuration` is itself a union — one of the three config variants:

| Variant                         | Use                                                                   |
| ------------------------------- | --------------------------------------------------------------------- |
| `FilterListConfiguration`       | Pick-list of specific values; coarser match operators                 |
| `CustomFilterListConfiguration` | Pick-list + wider match operators                                     |
| `CustomFilterConfiguration`     | Single value (literal or via `ParameterName`) with any match operator |

> **⚠️ `FilterListConfiguration.MatchOperator` restriction**: `FilterListConfiguration.MatchOperator` accepts only `CONTAINS` and `DOES_NOT_CONTAIN`.

> **⚠️ `SelectAllOptions` overrides `CategoryValues`**: When `FilterListConfiguration` includes `SelectAllOptions: "FILTER_ALL_VALUES"`, any `CategoryValues` in the same configuration are **ignored** — all values pass the filter. To set specific default selections, omit `SelectAllOptions` and list the desired defaults in `CategoryValues`.

### 2. NumericEqualityFilter — numeric equals / not-equals

```json
{
  "NumericEqualityFilter": {
    "FilterId":       "filter-year",
    "Column":         { "DataSetIdentifier": "sales", "ColumnName": "year" },
    "Value":          2025,
    "ParameterName":  "SelectedYear",
    "MatchOperator":  "EQUALS" | "DOES_NOT_EQUAL",
    "NullOption":     "ALL_VALUES" | "NULLS_ONLY" | "NON_NULLS_ONLY",
    "AggregationFunction": { /* optional — apply the filter to the aggregated value */ },
    "SelectAllOptions": "FILTER_ALL_VALUES"
  }
}
```

Required: `FilterId`, `Column`, `MatchOperator`, `NullOption`. Either `Value` or `ParameterName` supplies the comparand (they are mutually exclusive in effect; submitting both is allowed but `ParameterName` wins at runtime).

### 3. NumericRangeFilter — numeric range

```json
{
  "NumericRangeFilter": {
    "FilterId":          "filter-revenue-band",
    "Column":            { "DataSetIdentifier": "sales", "ColumnName": "revenue" },
    "RangeMinimum":      { "StaticValue": 1000,   "Parameter": "MinRevenue" },
    "RangeMaximum":      { "StaticValue": 100000, "Parameter": "MaxRevenue" },
    "IncludeMinimum":    true,
    "IncludeMaximum":    false,
    "NullOption":        "ALL_VALUES" | "NULLS_ONLY" | "NON_NULLS_ONLY",
    "AggregationFunction": { ... },
    "SelectAllOptions":  "FILTER_ALL_VALUES"
  }
}
```

Each of `RangeMinimum` / `RangeMaximum` is itself a union-ish (`StaticValue` literal or `Parameter` name). Required: `FilterId`, `Column`, `NullOption`.

### 4. TimeEqualityFilter — exact-date match

```json
{
  "TimeEqualityFilter": {
    "FilterId":     "filter-run-date",
    "Column":       { "DataSetIdentifier": "sales", "ColumnName": "run_date" },
    "Value":        "2024-05-03T00:00:00Z",
    "RollingDate":  { "DataSetIdentifier": "sales", "Expression": "truncDate('DD', now())" },
    "ParameterName": "SelectedDate",
    "TimeGranularity": "YEAR" | "QUARTER" | "MONTH" | "WEEK" | "DAY" | "HOUR" | "MINUTE" | "SECOND" | "MILLISECOND"
  }
}
```

**Exactly one** of `Value` / `RollingDate` / `ParameterName` may be set — mutually exclusive.

### 5. TimeRangeFilter — date/time range

```json
{
  "TimeRangeFilter": {
    "FilterId":           "filter-date-range",
    "Column":             { "DataSetIdentifier": "sales", "ColumnName": "order_date" },
    "RangeMinimumValue":  { "StaticValue": "2024-01-01T00:00:00Z", "RollingDate": { ... }, "Parameter": "FromDate" },
    "RangeMaximumValue":  { "StaticValue": "2025-01-01T00:00:00Z", "RollingDate": { ... }, "Parameter": "ToDate" },
    "IncludeMinimum":     true,
    "IncludeMaximum":     true,
    "NullOption":         "ALL_VALUES" | "NULLS_ONLY" | "NON_NULLS_ONLY",
    "TimeGranularity":    "YEAR" | "QUARTER" | "MONTH" | "WEEK" | "DAY" | "HOUR" | "MINUTE" | "SECOND" | "MILLISECOND",
    "ExcludePeriodConfiguration": {
      "Amount":         1,
      "Granularity":    "YEAR" | "QUARTER" | "MONTH" | "WEEK" | "DAY" | "HOUR" | "MINUTE" | "SECOND",
      "Status":         "ENABLED" | "DISABLED"
    }
  }
}
```

Required: `FilterId`, `Column`, `NullOption`. `RangeMinimumValue` / `RangeMaximumValue` each carry one of static/rolling/parameter. `ExcludePeriodConfiguration` cuts out a recent window (e.g., "exclude the last week").

### 6. RelativeDatesFilter — relative windows

```json
{
  "RelativeDatesFilter": {
    "FilterId":                "filter-last-30-days",
    "Column":                  { "DataSetIdentifier": "sales", "ColumnName": "order_date" },
    "AnchorDateConfiguration": { "AnchorOption": "NOW", "ParameterName": "Anchor" },
    "TimeGranularity":         "YEAR" | "QUARTER" | "MONTH" | "WEEK" | "DAY" | "HOUR" | "MINUTE" | "SECOND" | "MILLISECOND",
    "MinimumGranularity":      "DAY",
    "RelativeDateType":        "PREVIOUS" | "THIS" | "LAST" | "NOW" | "NEXT",
    "RelativeDateValue":       30,
    "ParameterName":           "WindowLength",
    "NullOption":              "ALL_VALUES" | "NULLS_ONLY" | "NON_NULLS_ONLY",
    "ExcludePeriodConfiguration": { ... }
  }
}
```

Required: `FilterId`, `Column`, `AnchorDateConfiguration`, `TimeGranularity`, `RelativeDateType`, `NullOption`. `RelativeDateValue` applies to `LAST` / `NEXT` only; `PREVIOUS`, `THIS`, `NOW` do not accept it.

`AnchorDateConfiguration.AnchorOption = "NOW"` anchors to query time; supplying `ParameterName` uses a DateTime parameter value as the anchor.

### 7. TopBottomFilter — top-N / bottom-N

```json
{
  "TopBottomFilter": {
    "FilterId":       "filter-top-10-products",
    "Column":         { "DataSetIdentifier": "sales", "ColumnName": "product_name" },
    "Limit":          10,
    "ParameterName":  "TopN",
    "AggregationSortConfigurations": [
      {
        "Column":             { "DataSetIdentifier": "sales", "ColumnName": "revenue" },
        "SortDirection":      "ASC" | "DESC",
        "AggregationFunction": { "NumericalAggregationFunction": { "SimpleNumericalAggregation": "SUM" } }
      }
    ],
    "TimeGranularity": "YEAR" | "QUARTER" | "MONTH" | "WEEK" | "DAY" | "HOUR" | "MINUTE" | "SECOND" | "MILLISECOND"
  }
}
```

Required: `FilterId`, `Column`, `AggregationSortConfigurations` (1–100 entries). `Limit` vs `ParameterName` — one supplies the N. A `TopBottomFilter` with `SortDirection = "DESC"` + `Limit = 10` = top-10; `ASC` = bottom-10.

### 8. NestedFilter — filter by subset

```json
{
  "NestedFilter": {
    "FilterId":         "filter-active-customers",
    "Column":           { "DataSetIdentifier": "sales", "ColumnName": "customer_id" },
    "IncludeInnerSet":  true,
    "InnerFilter": {
      "CategoryInnerFilter": {
        "Column":          { "DataSetIdentifier": "customers", "ColumnName": "status" },
        "Configuration":   { /* same variants as CategoryFilter.Configuration */ },
        "DefaultFilterControlConfiguration": { ... }
      }
    }
  }
}
```

All four fields required. `IncludeInnerSet = true` → keep rows whose `Column` appears in the inner filter's result set. `InnerFilter` is its own union; `CategoryInnerFilter` is the main variant.

Use when the filter condition depends on a _different dataset_ than the dataset being filtered.

## Common Enum Cheatsheet

### Gotchas

- **`DefaultFilterControlConfiguration.ControlOptions`**: `DefaultFilterControlConfiguration.ControlOptions` keys: `DefaultDropdownOptions`, `DefaultListOptions`, `DefaultDateTimePickerOptions`, `DefaultSliderOptions`, `DefaultTextFieldOptions`, `DefaultTextAreaOptions`, `DefaultRelativeDateTimeOptions`.
- **`FilterListConfiguration` + `DefaultDropdownOptions`**: When using `FilterListConfiguration`, the `DefaultDropdownOptions` (or `DefaultListOptions`) must **not** contain `SelectableValues` — the selectable values come from the filter's own configuration. Including them causes `InvalidParameterValueException`.
- **Cross-sheet filter controls**: When a `FilterGroup.ScopeConfiguration` is `AllSheets` (or spans multiple sheets), any `FilterControl` on a sheet that references that filter **must** use the `CrossSheet` variant (`{ "CrossSheet": { "FilterControlId": "...", "SourceFilterId": "..." } }`). Using a sheet-local control type (e.g., `Dropdown`) with a multi-sheet filter causes `InvalidParameterValueException`.
- **`CustomFilterConfiguration`**: `CategoryValue`, `ParameterName`, and `SelectAllOptions` are mutually exclusive — set exactly one.

| Enum                              | Values                                                                                 |
| --------------------------------- | -------------------------------------------------------------------------------------- |
| `NullOption`                      | `ALL_VALUES`, `NULLS_ONLY`, `NON_NULLS_ONLY`                                           |
| `MatchOperator` (numeric/time eq) | `EQUALS`, `DOES_NOT_EQUAL`                                                             |
| `MatchOperator` (category custom) | `EQUALS`, `DOES_NOT_EQUAL`, `CONTAINS`, `DOES_NOT_CONTAIN`, `STARTS_WITH`, `ENDS_WITH` |
| `SelectAllOptions`                | `FILTER_ALL_VALUES`                                                                    |
| `TimeGranularity`                 | `YEAR`, `QUARTER`, `MONTH`, `WEEK`, `DAY`, `HOUR`, `MINUTE`, `SECOND`, `MILLISECOND`   |
| `RelativeDateType`                | `PREVIOUS`, `THIS`, `LAST`, `NOW`, `NEXT`                                              |
| `CrossDataset`                    | `ALL_DATASETS`, `SINGLE_DATASET`                                                       |

## AggregationFunction

Referenced by multiple filters. Same shape as in `MeasureField`:

```json
{
  "NumericalAggregationFunction":   { "SimpleNumericalAggregation": "SUM" | "AVERAGE" | "MIN" | "MAX" | "COUNT" | "DISTINCT_COUNT" | "VAR" | "VARP" | "STDEV" | "STDEVP" | "MEDIAN" | "PERCENTILE", "PercentileAggregation": { "PercentileValue": 95 } },
  "CategoricalAggregationFunction": "COUNT" | "DISTINCT_COUNT",
  "DateAggregationFunction":        "COUNT" | "DISTINCT_COUNT" | "MIN" | "MAX"
}
```

When a filter declares `AggregationFunction`, it filters on the post-aggregation value (e.g., "SUM(revenue) > 10000 per region") rather than the raw row value.

### DefaultFilterControlConfiguration — complete structure

```json
{
  "DefaultFilterControlConfiguration": {
    "Title": "Region",
    "ControlOptions": {
      "DefaultDropdownOptions": {
        "Type": "SINGLE_SELECT",
        "DisplayOptions": { "TitleOptions": { "Visibility": "VISIBLE" } }
      }
    }
  }
}
```

Valid `ControlOptions` keys: `DefaultDateTimePickerOptions`, `DefaultDropdownOptions`, `DefaultListOptions`, `DefaultSliderOptions`, `DefaultTextFieldOptions`, `DefaultTextAreaOptions`, `DefaultRelativeDateTimeOptions`.
