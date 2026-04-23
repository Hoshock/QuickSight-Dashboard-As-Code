# Visuals

A `Visual` is a union type — each entry in `SheetDefinition.Visuals` has exactly one of 25 keys. The key name identifies the visual type.

## Common Envelope

Every variant shares the same structural envelope, differing only in the nested `ChartConfiguration` shape:

```json
{
  "<VisualTypeKey>Visual": {
    "VisualId":              "string",
    "Title":                 { "Visibility": "VISIBLE" | "HIDDEN", "FormatText": { "PlainText": "..." } | { "RichText": "..." } },
    "Subtitle":              { "Visibility": "VISIBLE" | "HIDDEN", "FormatText": { "PlainText": "..." } },
    "ChartConfiguration":    { /* per-visual — see links below */ },
    "Actions":               [ /* VisualCustomAction[] — max 10 */ ],
    "ColumnHierarchies":     [ /* drill-down hierarchies — max 2 */ ],
    "VisualContentAltText":  "string"
  }
}
```

| Field                  | Required                              | Constraint                                                                              |
| ---------------------- | ------------------------------------- | --------------------------------------------------------------------------------------- |
| `VisualId`             | ✅                                    | 1–512 chars, pattern `[\w\-]+`. Must match the `ElementId` used in the sheet's `Layout` |
| `Title`                | ❌                                    | —                                                                                       |
| `Subtitle`             | ❌                                    | —                                                                                       |
| `ChartConfiguration`   | ❌ (but empty visual will not render) | Per-visual shape                                                                        |
| `Actions`              | ❌                                    | Max 10 custom actions                                                                   |
| `ColumnHierarchies`    | ❌                                    | Max 2                                                                                   |
| `VisualContentAltText` | ❌                                    | 1–1024 chars                                                                            |

## The 25 Visual Types

Union key ⇒ visual on screen. Link to the corresponding AWS API page for the `ChartConfiguration` schema.

| Key                   | Description                                                    | API Ref                                                                                                                   |
| --------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `BarChartVisual`      | Bar chart. Supports horizontal/vertical, stacked, 100% stacked | [`API_BarChartVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_BarChartVisual.html)           |
| `BoxPlotVisual`       | Box plot for distribution analysis                             | [`API_BoxPlotVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_BoxPlotVisual.html)             |
| `ComboChartVisual`    | Bar + line combined                                            | [`API_ComboChartVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_ComboChartVisual.html)       |
| `CustomContentVisual` | Embed iframe / image / HTML block                              | [`API_CustomContentVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_CustomContentVisual.html) |
| `EmptyVisual`         | Placeholder — no rendering                                     | [`API_EmptyVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_EmptyVisual.html)                 |
| `FilledMapVisual`     | Choropleth map                                                 | [`API_FilledMapVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_FilledMapVisual.html)         |
| `FunnelChartVisual`   | Funnel chart                                                   | [`API_FunnelChartVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_FunnelChartVisual.html)     |
| `GaugeChartVisual`    | Single-metric gauge                                            | [`API_GaugeChartVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_GaugeChartVisual.html)       |
| `GeospatialMapVisual` | Point / heat map on geography                                  | [`API_GeospatialMapVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_GeospatialMapVisual.html) |
| `HeatMapVisual`       | 2-axis heat map                                                | [`API_HeatMapVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_HeatMapVisual.html)             |
| `HistogramVisual`     | Histogram with bins                                            | [`API_HistogramVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_HistogramVisual.html)         |
| `InsightVisual`       | NLG narrative / auto-insight                                   | [`API_InsightVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_InsightVisual.html)             |
| `KPIVisual`           | Single-metric KPI with trend/comparison                        | [`API_KPIVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_KPIVisual.html)                     |
| `LayerMapVisual`      | Multi-layer geospatial map                                     | [`API_LayerMapVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_LayerMapVisual.html)           |
| `LineChartVisual`     | Line / area chart (stacked, 100%)                              | [`API_LineChartVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_LineChartVisual.html)         |
| `PieChartVisual`      | Pie / donut                                                    | [`API_PieChartVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_PieChartVisual.html)           |
| `PivotTableVisual`    | Pivot table with collapsible rows/columns                      | [`API_PivotTableVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_PivotTableVisual.html)       |
| `PluginVisual`        | Custom plugin visual                                           | [`API_PluginVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_PluginVisual.html)               |
| `RadarChartVisual`    | Radar / spider chart                                           | [`API_RadarChartVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_RadarChartVisual.html)       |
| `SankeyDiagramVisual` | Sankey flow diagram                                            | [`API_SankeyDiagramVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_SankeyDiagramVisual.html) |
| `ScatterPlotVisual`   | Scatter / bubble plot                                          | [`API_ScatterPlotVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_ScatterPlotVisual.html)     |
| `TableVisual`         | Flat tabular data                                              | [`API_TableVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_TableVisual.html)                 |
| `TreeMapVisual`       | Rectangle tree map                                             | [`API_TreeMapVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_TreeMapVisual.html)             |
| `WaterfallVisual`     | Waterfall chart                                                | [`API_WaterfallVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_WaterfallVisual.html)         |
| `WordCloudVisual`     | Word cloud                                                     | [`API_WordCloudVisual.html`](https://docs.aws.amazon.com/quicksight/latest/APIReference/API_WordCloudVisual.html)         |

## FieldWell Pattern

All data-bound visuals funnel data through a `FieldWells` sub-object inside `ChartConfiguration`. `FieldWells` is a union type keyed by visual style (e.g., a bar chart has `BarChartAggregatedFieldWells`). Each well slot holds an array of field references (`MeasureField` / `DimensionField` — see below).

Schematically:

```json
{
  "ChartConfiguration": {
    "FieldWells": {
      "<VisualStyle>AggregatedFieldWells": {
        "Category":  [ /* DimensionField[] */ ],
        "Values":    [ /* MeasureField[]   */ ],
        "Colors":    [ /* DimensionField[] */ ],
        "SmallMultiples": [ /* DimensionField[] */ ]
      }
    },
    "SortConfiguration":     { ... },
    "CategoryAxis":          { ... },
    "PrimaryYAxisDisplayOptions": { ... },
    "Legend":                { ... },
    "DataLabels":            { ... },
    "Tooltip":               { ... },
    "ReferenceLines":        [ ... ],
    "ContributionAnalysisDefaults": [ ... ],
    "Interactions":          { "ContextMenuOption": { "AvailabilityStatus": "ENABLED" | "DISABLED" }, "VisualMenuOption": { ... } }
  }
}
```

Slot names (`Category`, `Values`, `Colors`, ...) vary per visual. The table below lists the primary `FieldWells` envelope and its slot names for the most common visual types. For any visual not listed, consult the per-visual AWS doc page linked above.

### Cross-cutting conventions

- **`FieldId`** is a string unique within a visual. Pick any stable convention — `<dataset>.<column>` for simple cases, `<dataset>.<column>.<agg>` when the same column appears in multiple wells with different aggregations (e.g., `sales.revenue.sum` vs. `sales.revenue.avg`). `FieldId` is what `SortConfiguration`, `DataLabels`, `ReferenceLines`, etc. reference — keep it stable if you later re-author a visual.
- Use a `CalculatedMeasureField` for **table calculations** (pivot table in-cell calculations with inline expressions). To use a top-level `CalculatedField` as a measure, reference it via `NumericalMeasureField`, `CategoricalMeasureField`, or `DateMeasureField` with `Column.ColumnName` set to the calculated field's `Name`.
- **Use a CalculatedField as a dimension**: reference it as a `Column` in a `CategoricalDimensionField` / `DateDimensionField` / `NumericalDimensionField` with `{DataSetIdentifier, ColumnName: <CalculatedField.Name>}` — `CalculatedField.Name` is the column name within the dataset namespace.

### FieldWells cheatsheet (common visuals)

| Visual                | `FieldWells` envelope                                                                 | Slot names (dim = DimensionField, meas = MeasureField)                                                                      |
| --------------------- | ------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `BarChartVisual`      | `BarChartAggregatedFieldWells`                                                        | `Category` (dim) · `Values` (meas) · `Colors` (dim) · `SmallMultiples` (dim)                                                |
| `LineChartVisual`     | `LineChartAggregatedFieldWells`                                                       | `Category` (dim) · `Values` (meas) · `Colors` (dim) · `SmallMultiples` (dim)                                                |
| `ComboChartVisual`    | `ComboChartAggregatedFieldWells`                                                      | `Category` (dim) · `BarValues` (meas) · `LineValues` (meas) · `Colors` (dim)                                                |
| `PieChartVisual`      | `PieChartAggregatedFieldWells`                                                        | `Category` (dim) · `Values` (meas) · `SmallMultiples` (dim)                                                                 |
| `KPIVisual`           | `KPIFieldWells`                                                                       | `Values` (meas) · `TargetValues` (meas) · `TrendGroups` (dim) — **`TargetValues` and `TrendGroups` are mutually exclusive** |
| `GaugeChartVisual`    | `GaugeChartFieldWells`                                                                | `Values` (meas) · `TargetValues` (meas)                                                                                     |
| `TableVisual`         | `TableAggregatedFieldWells` (or `TableUnaggregatedFieldWells`)                        | Aggregated: `GroupBy` (dim) · `Values` (meas). Unaggregated: `Values` (unaggregated `UnaggregatedField[]`)                  |
| `PivotTableVisual`    | `PivotTableAggregatedFieldWells`                                                      | `Rows` (dim) · `Columns` (dim) · `Values` (meas)                                                                            |
| `HeatMapVisual`       | `HeatMapAggregatedFieldWells`                                                         | `Rows` (dim) · `Columns` (dim) · `Values` (meas)                                                                            |
| `ScatterPlotVisual`   | `ScatterPlotCategoricallyAggregatedFieldWells` or `ScatterPlotUnaggregatedFieldWells` | `Category` (dim) · `XAxis` (meas) · `YAxis` (meas) · `Size` (meas) · `Label` (dim)                                          |
| `HistogramVisual`     | `HistogramAggregatedFieldWells`                                                       | `Values` (meas, **no AggregationFunction** — raw column only)                                                               |
| `TreeMapVisual`       | `TreeMapAggregatedFieldWells`                                                         | `Groups` (dim) · `Sizes` (meas) · `Colors` (meas)                                                                           |
| `WaterfallVisual`     | `WaterfallChartAggregatedFieldWells`                                                  | `Categories` (dim) · `Values` (meas) · `Breakdowns` (dim)                                                                   |
| `FunnelChartVisual`   | `FunnelChartAggregatedFieldWells`                                                     | `Category` (dim) · `Values` (meas)                                                                                          |
| `WordCloudVisual`     | `WordCloudAggregatedFieldWells`                                                       | `GroupBy` (dim) · `Size` (meas)                                                                                             |
| `BoxPlotVisual`       | `BoxPlotAggregatedFieldWells`                                                         | `GroupBy` (dim) · `Values` (meas, **no AggregationFunction**)                                                               |
| `SankeyDiagramVisual` | `SankeyDiagramAggregatedFieldWells`                                                   | `Source` (dim) · `Destination` (dim) · `Weight` (meas)                                                                      |
| `RadarChartVisual`    | `RadarChartAggregatedFieldWells`                                                      | `Category` (dim) · `Color` (dim) · `Values` (meas)                                                                          |
| `FilledMapVisual`     | `FilledMapAggregatedFieldWells`                                                       | `Geospatial` (dim) · `Values` (meas)                                                                                        |
| `InsightVisual`       | n/a                                                                                   | Uses `InsightConfiguration.Computations` instead of FieldWells                                                              |
| `CustomContentVisual` | n/a                                                                                   | Uses `ChartConfiguration.ContentConfiguration` (`ContentUrl`, `ContentType`)                                                |
| `EmptyVisual`         | n/a                                                                                   | No FieldWells — placeholder only                                                                                            |

Treat slot names as ground truth for "what role a field plays in this visual." Putting a MeasureField into a slot that expects a DimensionField (or vice versa) is a common cause of `UPDATE_FAILED` on deploy — `Errors[].ViolatedEntities[].Path` will point at the offending slot.

## DimensionField / MeasureField

Both are union types. A DimensionField wraps a non-aggregated column (category axis), and a MeasureField wraps an aggregated column (value axis).

### DimensionField

```json
{
  "CategoricalDimensionField": {
    "FieldId":           "sales.region",
    "Column":            { "DataSetIdentifier": "sales", "ColumnName": "region" },
    "HierarchyId":       "string",
    "FormatConfiguration": { /* StringFormatConfiguration */ }
  },
  "DateDimensionField": {
    "FieldId":           "sales.order_date.month",
    "Column":            { "DataSetIdentifier": "sales", "ColumnName": "order_date" },
    "DateGranularity":   "YEAR" | "QUARTER" | "MONTH" | "WEEK" | "DAY" | "HOUR" | "MINUTE" | "SECOND" | "MILLISECOND",
    "HierarchyId":       "string",
    "FormatConfiguration": { /* DateTimeFormatConfiguration */ }
  },
  "NumericalDimensionField": {
    "FieldId":           "sales.units",
    "Column":            { "DataSetIdentifier": "sales", "ColumnName": "units" },
    "HierarchyId":       "string",
    "FormatConfiguration": { /* NumberFormatConfiguration */ }
  }
}
```

### MeasureField

```json
{
  "CategoricalMeasureField": {
    "FieldId":            "sales.region_count",
    "Column":             { "DataSetIdentifier": "sales", "ColumnName": "region" },
    "AggregationFunction": "COUNT" | "DISTINCT_COUNT",
    "FormatConfiguration": { /* StringFormatConfiguration */ }
  },
  "DateMeasureField": {
    "FieldId":            "sales.last_order",
    "Column":             { "DataSetIdentifier": "sales", "ColumnName": "order_date" },
    "AggregationFunction": "COUNT" | "DISTINCT_COUNT" | "MIN" | "MAX",
    "FormatConfiguration": { /* DateTimeFormatConfiguration */ }
  },
  "NumericalMeasureField": {
    "FieldId":            "sales.revenue",
    "Column":             { "DataSetIdentifier": "sales", "ColumnName": "revenue" },
    "AggregationFunction": {
      "SimpleNumericalAggregation": "SUM" | "AVERAGE" | "MIN" | "MAX" | "COUNT" | "DISTINCT_COUNT" | "VAR" | "VARP" | "STDEV" | "STDEVP" | "MEDIAN" | "PERCENTILE",
      "PercentileAggregation":      { "PercentileValue": 95 }
    },
    "FormatConfiguration": { /* NumberFormatConfiguration */ }
  },
  "CalculatedMeasureField": {
    "FieldId":    "sales.revenue_yoy",
    "Expression": "(sum({revenue}) - periodOverPeriodLastValue(sum({revenue}), {order_date}, YEAR)) / ..."
  }
}
```

`FieldId` is arbitrary within the visual — used by `SortConfiguration`, `DataLabels`, `ReferenceLines` to reference this field. A common convention is `<dataset>.<column>`.

> **⚠️ MeasureField type must match column data type**

| Column data type      | Use this MeasureField     | AggregationFunction shape                                        |
| --------------------- | ------------------------- | ---------------------------------------------------------------- |
| `STRING`              | `CategoricalMeasureField` | Plain string: `"COUNT"` or `"DISTINCT_COUNT"`                    |
| `DATETIME`            | `DateMeasureField`        | Plain string: `"COUNT"`, `"DISTINCT_COUNT"`, `"MIN"`, or `"MAX"` |
| `INTEGER` / `DECIMAL` | `NumericalMeasureField`   | Object: `{ "SimpleNumericalAggregation": "SUM" }`                |

Using the wrong MeasureField type causes `COLUMN_TYPE_INCOMPATIBLE` at deploy time. This is the most common visual deploy failure.

Note the **inconsistent AggregationFunction shapes**: `CategoricalMeasureField` and `DateMeasureField` use a plain string, while `NumericalMeasureField` uses an object with `SimpleNumericalAggregation` (or `PercentileAggregation`). Mixing these shapes is a common error.

To use a top-level `CalculatedField` as a measure, reference it via the MeasureField type matching the calculated field's **output** data type — e.g., a calculated field that outputs a number uses `NumericalMeasureField` with `Column.ColumnName` set to the calculated field's `Name`.

> **⚠️ Aggregated CalculatedFields: omit AggregationFunction.** If the top-level `CalculatedField` expression already contains aggregate functions (e.g., `avg({error})`, `sum({x}) / sum({y})`), reference it via `NumericalMeasureField` **without** `AggregationFunction`. Setting `AggregationFunction` on an already-aggregated calculated field causes async `INVALID_COLUMN_AGGREGATION` at deploy time.

## VisualCustomAction

Attached to a visual via `Actions[]`. Triggers an operation on click/hover/filter.

```json
{
  "CustomActionId":  "drill-to-detail",
  "Name":            "Drill to detail",
  "Status":          "ENABLED" | "DISABLED",
  "Trigger":         "DATA_POINT_CLICK" | "DATA_POINT_MENU",
  "ActionOperations": [
    { "FilterOperation":    { "SelectedFieldsConfiguration": { ... }, "TargetVisualsConfiguration": { ... } } },
    { "NavigationOperation": { "LocalNavigationConfiguration": { "TargetSheetId": "detail" } } },
    { "SetParametersOperation": { "ParameterValueConfigurations": [ ... ] } },
    { "URLOperation": { "URLTemplate": "https://...?x=<<$selectedFieldValue>>", "URLTarget": "NEW_TAB" | "SAME_TAB" | "NEW_WINDOW" } }
  ]
}
```

`ActionOperations` supports multiple operations chained in a single action.

## ColumnHierarchy

Declares a drill-down hierarchy used by a visual (max 2 per visual).

```json
{
  "DateTimeHierarchy":      { "HierarchyId": "time-drill", "DrillDownFilters": [ ... ] },
  "ExplicitHierarchy":      { "HierarchyId": "geo-drill",  "Columns": [ { "DataSetIdentifier": "...", "ColumnName": "country" }, { "DataSetIdentifier": "...", "ColumnName": "region" }, { "DataSetIdentifier": "...", "ColumnName": "city" } ], "DrillDownFilters": [ ... ] },
  "PredefinedHierarchy":    { "HierarchyId": "auto-time",  "Columns": [ ... ], "DrillDownFilters": [ ... ] }
}
```

A DimensionField references a hierarchy via its `HierarchyId`.

## ChartConfiguration — commonly-used options beyond FieldWells

Every aggregated visual's `ChartConfiguration` carries the same recurring options. The key ones — enough to cover 80% of layout, sort, and label authoring:

### Orientation (bar / funnel / waterfall)

```json
{ "Orientation": "HORIZONTAL" | "VERTICAL" }
```

Top-level field on `BarChartConfiguration`, `FunnelChartConfiguration`, `WaterfallChartConfiguration`. Controls whether categories run along the X or Y axis. Default is `VERTICAL` for bar charts.

### SortConfiguration

Most visuals accept a `SortConfiguration` block. The shape varies slightly per visual, but the common patterns:

```json
// Bar / Line / Pie / Radar / Funnel (category-axis sort)
"SortConfiguration": {
  "CategorySort": [
    {
      "FieldSort": { "FieldId": "sales.revenue.sum", "Direction": "ASC" | "DESC" },
      "ColumnSort": { "SortBy": { "DataSetIdentifier": "sales", "ColumnName": "region" }, "Direction": "ASC" | "DESC", "AggregationFunction": { ... } }
    }
  ],
  "CategoryItemsLimit": { "ItemsLimit": 10, "OtherCategories": "INCLUDE" | "EXCLUDE" },
  "ColorItemsLimit":                   { "ItemsLimit": 10, "OtherCategories": "INCLUDE" | "EXCLUDE" },
  "SmallMultiplesSort":              [ { "FieldSort": { ... } } ],
  "SmallMultiplesLimitConfiguration": { ... }
}

// Table (per-field sort)
"SortConfiguration": {
  "RowSort":  [ { "FieldSort": { "FieldId": "...", "Direction": "ASC" | "DESC" } } ],
  "PaginationConfiguration": { "PageSize": 50, "PageNumber": 1 }
}

// Pivot
"SortConfiguration": {
  "FieldSortOptions": [ { "FieldId": "...", "SortBy": { "Field": { ... }, "Column": { ... }, "DataPath": { ... } } ] }
}
```

Each entry is a union — pick `FieldSort` for "sort by a field already in the visual" or `ColumnSort` for "sort by a column not in the visual (with its own aggregation)." `FieldId` must match one used in the visual's field wells.

### Legend

```json
"Legend": {
  "Visibility": "VISIBLE" | "HIDDEN",
  "Title":    { "Visibility": "VISIBLE" | "HIDDEN", "CustomLabel": "...", "FontConfiguration": { ... } },
  "Position": "AUTO" | "RIGHT" | "BOTTOM" | "TOP",
  "Width":   "200px",
  "Height":  "100px"
}
```

### DataLabels

```json
"DataLabels": {
  "Visibility": "VISIBLE" | "HIDDEN",
  "CategoryLabelVisibility": "VISIBLE" | "HIDDEN",
  "MeasureLabelVisibility":  "VISIBLE" | "HIDDEN",
  "Position":      "INSIDE" | "OUTSIDE" | "TOP" | "BOTTOM" | "LEFT" | "RIGHT" | "MIDDLE_CENTER",
  "Overlap":       "DISABLE_OVERLAP" | "ENABLE_OVERLAP",
  "LabelContent":  "VALUE" | "PERCENT" | "VALUE_AND_PERCENT",
  "LabelFontConfiguration": { ... },
  "LabelColor":    "#RRGGBB",
  "DataLabelTypes": [ { "FieldLabelType": { "FieldId": "...", "Visibility": "VISIBLE" | "HIDDEN" } } ]
}
```

### Tooltip

```json
"Tooltip": {
  "TooltipVisibility": "VISIBLE" | "HIDDEN",
  "SelectedTooltipType": "BASIC" | "DETAILED",
  "FieldBasedTooltip": {
    "AggregationVisibility": "VISIBLE" | "HIDDEN",
    "TooltipTitleType":      "NONE" | "PRIMARY_VALUE",
    "TooltipFields":         [ { "FieldTooltipItem": { "FieldId": "...", "Label": "...", "Visibility": "..." } } ]
  }
}
```

### Axis display options (bar / line / scatter / combo)

```json
"CategoryAxis":            { "Visibility": "VISIBLE" | "HIDDEN", "AxisLabelOptions": { ... }, "TickLabelOptions": { ... }, "GridLineVisibility": "VISIBLE" | "HIDDEN", "AxisLineVisibility": "VISIBLE" | "HIDDEN" },
"PrimaryYAxisDisplayOptions":  { "AxisOptions": { ... }, "MissingDataConfigurations": [ ... ] },
"SecondaryYAxisDisplayOptions": { ... }
```

**Axis label options** — visual-type-specific field names for custom axis labels:

| Visual type   | Category axis labels   | Value axis labels          |
| ------------- | ---------------------- | -------------------------- |
| `LineChart`   | `XAxisLabelOptions`    | `PrimaryYAxisLabelOptions` |
| `BarChart`    | `CategoryLabelOptions` | `ValueLabelOptions`        |
| `ComboChart`  | `CategoryLabelOptions` | `PrimaryYAxisLabelOptions` |
| `ScatterPlot` | `XAxisLabelOptions`    | `YAxisLabelOptions`        |

Each accepts the same shape:

```json
"XAxisLabelOptions": {
  "Visibility": "VISIBLE" | "HIDDEN",
  "SortIconVisibility": "VISIBLE" | "HIDDEN",
  "AxisLabelOptions": [
    {
      "CustomLabel": "Date",
      "ApplyTo": {
        "FieldId": "sales.order_date",
        "Column": { "DataSetIdentifier": "sales", "ColumnName": "order_date" }
      }
    }
  ]
}
```

`ApplyTo.FieldId` must match a `FieldId` used in the visual's field wells. Use `<<$ParameterName>>` in `CustomLabel` for dynamic axis labels (e.g., `"<<$SelectedMetric>>"`).

> **⚠️ Field name varies by visual type.** Using `CategoryLabelOptions` on a `LineChartVisual` (which expects `XAxisLabelOptions`) causes `InvalidParameterValueException`. Always check the visual-type-specific field name.

### ReferenceLines

```json
"ReferenceLines": [
  {
    "Status": "ENABLED" | "DISABLED",
    "DataConfiguration": {
      "StaticConfiguration": { "Value": 1000 },
      "DynamicConfiguration": {
        "Column":    { "DataSetIdentifier": "sales", "ColumnName": "revenue" },
        "MeasureAggregationFunction": { "NumericalAggregationFunction": { "SimpleNumericalAggregation": "AVERAGE" } },
        "Calculation":                { "CalculationType": "AVERAGE" | "MEDIAN" | ... }
      }
    },
    "StyleConfiguration": { "Pattern": "SOLID" | "DASHED" | "DOTTED", "Color": "#RRGGBB" },
    "LabelConfiguration": { "ValueLabelConfiguration": { ... }, "CustomLabelConfiguration": { "CustomLabel": "Target" }, "FontConfiguration": { ... }, "FontColor": "#...", "HorizontalPosition": "LEFT" | "CENTER" | "RIGHT", "VerticalPosition": "ABOVE" | "BELOW" }
  }
]
```

### Interactions

```json
"Interactions": {
  "ContextMenuOption": { "AvailabilityStatus": "ENABLED" | "DISABLED" },
  "VisualMenuOption":  { "AvailabilityStatus": "ENABLED" | "DISABLED" }
}
```

## Practical Notes

- **Every visual on a sheet must have a matching element in the sheet's layout.** `VisualId` ↔ `Layout.Elements[].ElementId` with `ElementType = "VISUAL"`.
- **`ChartConfiguration` is optional** on the envelope, but a visual without it renders as empty. Author it.
- **For most chart types, the only hard-required config is `FieldWells`.** Axis/legend/data-label options have sensible defaults.
- **`TableVisual` and `PivotTableVisual` have very different `FieldWells` shapes** from aggregated charts (GroupBy / Values for tables; Rows / Columns / Values for pivots) — don't copy envelope between types.
- **`CustomContentVisual` needs `ContentUrl` + `ContentType`** (`IMAGE` or `OTHER_EMBEDDABLE_CONTENT`) in `ChartConfiguration.ContentConfiguration`; it does not take field wells.
- **`EmptyVisual` is intentional** — used to reserve a slot in the layout for a visual that will be defined later. Not usually hand-authored; produced by the UI when you drag "blank visual".
