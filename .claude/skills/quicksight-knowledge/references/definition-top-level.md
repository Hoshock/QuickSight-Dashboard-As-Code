# AnalysisDefinition — Top-Level Structure

`AnalysisDefinition` is the dashboard-as-code root object. It is passed as `Definition` on `CreateAnalysis` / `UpdateAnalysis` and returned under `Definition` by `DescribeAnalysisDefinition`.

## Full field list

```json
{
  "DataSetIdentifierDeclarations": [ ... ],
  "AnalysisDefaults":              { ... },
  "CalculatedFields":              [ ... ],
  "ColumnConfigurations":          [ ... ],
  "FilterGroups":                  [ ... ],
  "Options":                       { ... },
  "ParameterDeclarations":         [ ... ],
  "QueryExecutionOptions":         { ... },
  "Sheets":                        [ ... ],
  "StaticFiles":                   [ ... ],
  "TooltipSheets":                 [ ... ]
}
```

| Field                           | Type                             | Required | Limit        | See also                                                             |
| ------------------------------- | -------------------------------- | -------- | ------------ | -------------------------------------------------------------------- |
| `DataSetIdentifierDeclarations` | `DataSetIdentifierDeclaration[]` | ✅       | 1–50 items   | [definition-calculated-fields.md](./definition-calculated-fields.md) |
| `AnalysisDefaults`              | `AnalysisDefaults`               | ❌       | —            | below                                                                |
| `CalculatedFields`              | `CalculatedField[]`              | ❌       | 0–2000 items | [definition-calculated-fields.md](./definition-calculated-fields.md) |
| `ColumnConfigurations`          | `ColumnConfiguration[]`          | ❌       | 0–2000 items | below                                                                |
| `FilterGroups`                  | `FilterGroup[]`                  | ❌       | 0–2000 items | [definition-filters.md](./definition-filters.md)                     |
| `Options`                       | `AssetOptions`                   | ❌       | —            | below                                                                |
| `ParameterDeclarations`         | `ParameterDeclaration[]`         | ❌       | 0–400 items  | [definition-parameters.md](./definition-parameters.md)               |
| `QueryExecutionOptions`         | `QueryExecutionOptions`          | ❌       | —            | below                                                                |
| `Sheets`                        | `SheetDefinition[]`              | ❌       | 0–20 items   | [definition-sheets-layouts.md](./definition-sheets-layouts.md)       |
| `StaticFiles`                   | `StaticFile[]`                   | ❌       | 0–200 items  | below                                                                |
| `TooltipSheets`                 | `TooltipSheetDefinition[]`       | ❌       | 0–50 items   | below                                                                |

**Only `DataSetIdentifierDeclarations` is strictly required.** An analysis with an empty `Sheets` array is valid but renders as "no sheets" in the UI. In practice, any useful analysis has at least one sheet.

## AnalysisDefaults

Sets defaults applied to newly-created sheets (from the UI, not from API). Does not retroactively apply to existing sheets.

```json
{
  "DefaultNewSheetConfiguration": {
    "InteractiveLayoutConfiguration": {
      "Grid":     { "CanvasSizeOptions": { "ScreenCanvasSizeOptions": { "ResizeOption": "FIXED" | "RESPONSIVE", "OptimizedViewPortWidth": "1600px" } } },
      "FreeForm": { "CanvasSizeOptions": { "ScreenCanvasSizeOptions": { "OptimizedViewPortWidth": "1600px" } } }
    },
    "PaginatedLayoutConfiguration": {
      "SectionBased": { "CanvasSizeOptions": { "PaperCanvasSizeOptions": { "PaperSize": "US_LETTER" | "A4" | ..., "PaperOrientation": "PORTRAIT" | "LANDSCAPE", "PaperMargin": { ... } } } }
    },
    "SheetContentType": "PAGINATED" | "INTERACTIVE"
  }
}
```

`SheetContentType` is the key switch between "dashboard" (`INTERACTIVE`) and "paginated report" (`PAGINATED`) defaults.

> **⚠️ `ResizeOption: RESPONSIVE`** must not include `OptimizedViewPortWidth`. Only `FIXED` accepts it.

## Options (AssetOptions)

Analysis-wide options. All fields are optional.

```json
{
  "CustomActionDefaults":     { ... },
  "ExcludedDataSetArns":      ["arn:aws:quicksight:...:dataset/..."],
  "QBusinessInsightsStatus":  "ENABLED" | "DISABLED",
  "Timezone":                 "America/New_York",
  "WeekStart":                "SUNDAY" | "MONDAY" | ... | "SATURDAY"
}
```

- `Timezone` — IANA name. Affects how `DateTime` fields are aggregated across visuals.
- `WeekStart` — affects week-level time granularity.
- `ExcludedDataSetArns` — suppress specific datasets from being used despite being declared.
- `QBusinessInsightsStatus` — toggle Q Business integration for this analysis.

## QueryExecutionOptions

```json
{ "QueryExecutionMode": "AUTO" | "MANUAL" }
```

`MANUAL` defers query execution until the user clicks Apply. Default is `AUTO`.

## ColumnConfiguration

Analysis-level default formatting for a column. Applied wherever the column appears unless overridden by a visual-local configuration.

```json
{
  "Column": { "DataSetIdentifier": "...", "ColumnName": "..." },
  "FormatConfiguration": {
    "StringFormatConfiguration":   { ... },
    "NumberFormatConfiguration":   { ... },
    "DateTimeFormatConfiguration": { ... }
  },
  "Role": "DIMENSION" | "MEASURE",
  "ColorsConfiguration": { "CustomColors": [ { "FieldValue": "...", "Color": "#RRGGBB", "SpecialValue": "EMPTY" | "NULL" | "OTHER" } ] }
}
```

Exactly one of the three `*FormatConfiguration` variants applies, matching the column type. `Role` overrides the column's auto-inferred role for this analysis.

## StaticFile

Assets (images, icons) uploaded with the analysis. Referenced from `SheetImage` / `CustomContentVisual`.

```json
{
  "ImageStaticFile": {
    "StaticFileId": "my-logo",
    "Source": {
      "UrlOptions": { "Url": "https://..." },
      "S3Options": { "BucketName": "...", "ObjectKey": "..." }
    }
  },
  "SpatialStaticFile": {
    /* GeoJSON / shapefile for GeospatialMapVisual */
  }
}
```

Union type — one-of `ImageStaticFile` / `SpatialStaticFile`.

## TooltipSheetDefinition

A sheet whose sole purpose is to render inside another visual's tooltip.

```json
{ "SheetId": "tooltip-1" /* ... SheetDefinition fields ... */ }
```

Referenced from a visual's tooltip configuration by `SheetId`.

## Practical structure

A minimal-but-functional definition looks like this:

```json
{
  "DataSetIdentifierDeclarations": [
    {
      "Identifier": "sales",
      "DataSetArn": "arn:aws:quicksight:us-east-1:123456789012:dataset/sales-ds"
    }
  ],
  "Sheets": [
    {
      "SheetId": "overview",
      "Name": "Overview",
      "ContentType": "INTERACTIVE",
      "Layouts": [
        {
          "Configuration": {
            "GridLayout": {
              "Elements": [
                {
                  "ElementId": "kpi-1",
                  "ElementType": "VISUAL",
                  "ColumnIndex": 0,
                  "ColumnSpan": 12,
                  "RowIndex": 0,
                  "RowSpan": 4
                }
              ]
            }
          }
        }
      ],
      "Visuals": [
        {
          "KPIVisual": {
            "VisualId": "kpi-1",
            "Title": {
              "Visibility": "VISIBLE",
              "FormatText": { "PlainText": "Total Sales" }
            },
            "ChartConfiguration": {
              "FieldWells": {
                "Values": [
                  {
                    "NumericalMeasureField": {
                      "FieldId": "sales.revenue.sum",
                      "Column": {
                        "DataSetIdentifier": "sales",
                        "ColumnName": "revenue"
                      },
                      "AggregationFunction": {
                        "SimpleNumericalAggregation": "SUM"
                      }
                    }
                  }
                ]
              }
            }
          }
        }
      ]
    }
  ]
}
```

The pieces: declare one dataset, declare one sheet with one layout pinning one visual to a grid position, define the visual. That's the minimum viable dashboard-as-code.

## Worked end-to-end example

A slightly larger example that actually deploys — one sheet, one KPI, one bar chart, one region filter, one parameter. Save as `definition.json` and submit via `aws quicksight create-analysis --aws-account-id ... --analysis-id sales-demo --name "Sales Demo" --definition file://definition.json`.

```json
{
  "DataSetIdentifierDeclarations": [
    {
      "Identifier": "sales",
      "DataSetArn": "arn:aws:quicksight:us-east-1:123456789012:dataset/sales-ds"
    }
  ],
  "ParameterDeclarations": [
    {
      "StringParameterDeclaration": {
        "Name": "SelectedRegion",
        "ParameterValueType": "SINGLE_VALUED",
        "DefaultValues": { "StaticValues": ["APAC"] }
      }
    }
  ],
  "FilterGroups": [
    {
      "FilterGroupId": "fg-region",
      "CrossDataset": "SINGLE_DATASET",
      "Status": "ENABLED",
      "Filters": [
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
      ],
      "ScopeConfiguration": { "AllSheets": {} }
    }
  ],
  "Sheets": [
    {
      "SheetId": "overview",
      "Name": "Overview",
      "ContentType": "INTERACTIVE",
      "Layouts": [
        {
          "Configuration": {
            "GridLayout": {
              "Elements": [
                {
                  "ElementId": "kpi-total",
                  "ElementType": "VISUAL",
                  "ColumnIndex": 0,
                  "ColumnSpan": 12,
                  "RowIndex": 0,
                  "RowSpan": 4
                },
                {
                  "ElementId": "bar-by-product",
                  "ElementType": "VISUAL",
                  "ColumnIndex": 0,
                  "ColumnSpan": 36,
                  "RowIndex": 4,
                  "RowSpan": 10
                }
              ]
            }
          }
        }
      ],
      "Visuals": [
        {
          "KPIVisual": {
            "VisualId": "kpi-total",
            "Title": {
              "Visibility": "VISIBLE",
              "FormatText": { "PlainText": "Total Sales (<<$SelectedRegion>>)" }
            },
            "ChartConfiguration": {
              "FieldWells": {
                "Values": [
                  {
                    "NumericalMeasureField": {
                      "FieldId": "sales.revenue.sum",
                      "Column": {
                        "DataSetIdentifier": "sales",
                        "ColumnName": "revenue"
                      },
                      "AggregationFunction": {
                        "SimpleNumericalAggregation": "SUM"
                      }
                    }
                  }
                ]
              }
            }
          }
        },
        {
          "BarChartVisual": {
            "VisualId": "bar-by-product",
            "Title": {
              "Visibility": "VISIBLE",
              "FormatText": { "PlainText": "Revenue by Product" }
            },
            "ChartConfiguration": {
              "FieldWells": {
                "BarChartAggregatedFieldWells": {
                  "Category": [
                    {
                      "CategoricalDimensionField": {
                        "FieldId": "sales.product",
                        "Column": {
                          "DataSetIdentifier": "sales",
                          "ColumnName": "product"
                        }
                      }
                    }
                  ],
                  "Values": [
                    {
                      "NumericalMeasureField": {
                        "FieldId": "sales.revenue.sum.bar",
                        "Column": {
                          "DataSetIdentifier": "sales",
                          "ColumnName": "revenue"
                        },
                        "AggregationFunction": {
                          "SimpleNumericalAggregation": "SUM"
                        }
                      }
                    }
                  ]
                }
              },
              "SortConfiguration": {
                "CategorySort": [
                  {
                    "FieldSort": {
                      "FieldId": "sales.revenue.sum.bar",
                      "Direction": "DESC"
                    }
                  }
                ]
              },
              "Orientation": "VERTICAL"
            }
          }
        }
      ]
    }
  ]
}
```

This example exercises: dataset declaration, parameter, filter bound to parameter, KPI with parameterized title, bar chart with a sorted dimension, grid layout pinning two visuals.
