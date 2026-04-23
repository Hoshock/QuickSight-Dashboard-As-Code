---
name: quicksight-knowledge
description: |
  What: Amazon QuickSight Analysis "dashboard as code" reference — API operations, AWS CLI usage, and the AnalysisDefinition schema.
  Use when: Creating, updating, or reading a QuickSight analysis as a JSON definition (CreateAnalysis, UpdateAnalysis, DescribeAnalysis, DescribeAnalysisDefinition, UpdateAnalysisPermissions) via `aws quicksight` CLI or SDK calls.
---

# QuickSight Dashboard-as-Code Knowledge

This skill provides the reference material an agent needs to author, diff, or regenerate an Amazon QuickSight **Analysis** from a JSON definition. QuickSight exposes an analysis as a single `AnalysisDefinition` object — that object _is_ the dashboard as code.

## Prerequisite: datasets exist before the analysis

`AnalysisDefinition.DataSetIdentifierDeclarations[].DataSetArn` must point at a real, accessible `AWS::QuickSight::DataSet` at the moment `CreateAnalysis` / `UpdateAnalysis` runs.

## Mental Model

An analysis has two orthogonal concerns that must both be managed:

1. **Content** — the `AnalysisDefinition` JSON: sheets, visuals, filters, parameters, calculated fields. Mutated via `CreateAnalysis` / `UpdateAnalysis`.
2. **Access** — the ResourcePermission list. Mutated via `UpdateAnalysisPermissions`.

The two are independent API paths; updating the definition does **not** reset permissions, and vice versa.

An analysis can be authored two ways:

- **From a `SourceEntity`** (pointing at a Template ARN + dataset overrides). The user wires a template, QuickSight clones it. The definition is not sent by the caller.
- **From a `Definition`** (the full `AnalysisDefinition` JSON). This is the "dashboard as code" path — the caller owns the full schema.

`SourceEntity` and `Definition` are mutually exclusive on both `CreateAnalysis` and `UpdateAnalysis`. For dashboard-as-code workflows, always use `Definition`.

## Lifecycle (dashboard-as-code)

```
DescribeAnalysisDefinition ──▶ Definition JSON ──▶ edit ──▶ UpdateAnalysis (Definition=...)
                                                           │
                                    CreateAnalysis ◀───────┘  (if net-new analysis)
                                    DescribeAnalysis ─▶ Status / Errors (poll after create/update)
```

`CreateAnalysis` and `UpdateAnalysis` return asynchronously (`CREATION_IN_PROGRESS` / `UPDATE_IN_PROGRESS`). Poll `DescribeAnalysis` until `Status` is `*_SUCCESSFUL` or `*_FAILED`.

## Deploy-time errors: two-phase validation

QuickSight validates definitions in **two phases**:

1. **Synchronous (HTTP 400)** — Schema violations (wrong field names, invalid enums, structural constraint violations). Fix the JSON and resubmit.
2. **Asynchronous (poll `DescribeAnalysis`)** — Semantic errors (column references, expression syntax, dataset accessibility, permission validation). Read `Analysis.Errors[].ViolatedEntities[].Path` to locate the broken node.

After `CREATION_FAILED`, the `AnalysisId` is consumed — you must `delete-analysis --force-delete-without-recovery` before re-creating.

See [workflow.md](./references/workflow.md) §5 for failure modes and §6 for the inspection recipe.

## Reference Index

Load the file that matches the current subtask. Do not preload everything — these are long.

| Topic                        | File                                                                            | Read when...                                                                                                                                                                                                   |
| ---------------------------- | ------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| API operations               | [apis.md](./references/apis.md)                                                 | Calling `CreateAnalysis`, `UpdateAnalysis`, `DescribeAnalysis`, `DescribeAnalysisDefinition`, `UpdateAnalysisPermissions`, or `DeleteAnalysis` — request/response shape, required/optional fields, error codes |
| AWS CLI usage                | [cli-usage.md](./references/cli-usage.md)                                       | Shelling out to `aws quicksight ...` — how to pass a large definition via `--cli-input-json file://`, the skeleton workflow, common gotchas                                                                    |
| AnalysisDefinition top-level | [definition-top-level.md](./references/definition-top-level.md)                 | Assembling the outer `AnalysisDefinition` object — limits on sheets/parameters/calculated fields, `AnalysisDefaults`, `Options`, `QueryExecutionOptions`                                                       |
| Sheets + Layouts             | [definition-sheets-layouts.md](./references/definition-sheets-layouts.md)       | Defining sheets, placing visuals on a grid / free-form canvas / section-based report                                                                                                                           |
| Visual types                 | [definition-visuals.md](./references/definition-visuals.md)                     | Choosing or authoring a visual — all 25 visual types, common envelope, links to per-visual schemas                                                                                                             |
| Filters                      | [definition-filters.md](./references/definition-filters.md)                     | Defining a `FilterGroup` — all 8 filter types with their required/optional fields                                                                                                                              |
| Parameters                   | [definition-parameters.md](./references/definition-parameters.md)               | Declaring `ParameterDeclarations` — the 4 typed variants, default values, dataset mapping                                                                                                                      |
| Calculated fields + columns  | [definition-calculated-fields.md](./references/definition-calculated-fields.md) | Referencing dataset columns (`ColumnIdentifier`), declaring datasets (`DataSetIdentifierDeclaration`), writing `CalculatedField` expressions                                                                   |
| Dashboard-as-code workflow   | [workflow.md](./references/workflow.md)                                         | Designing the round-trip (describe → edit → update), async polling, diffs, permission handling                                                                                                                 |

## Hard Constraints (cross-cutting)

These are the constants most commonly hit when authoring. Full constraint tables live in the per-topic files.

| Field                                      | Constraint                                                           |
| ------------------------------------------ | -------------------------------------------------------------------- |
| `AwsAccountId` (path)                      | 12 digits, `^[0-9]{12}$`                                             |
| `AnalysisId` (path)                        | 1–512 chars, `[\w\-]+` — case-sensitive, appears in the analysis URL |
| `Name`                                     | 1–2048 chars                                                         |
| `Definition.DataSetIdentifierDeclarations` | **Required** when using `Definition`, 1–50 items                     |
| `Definition.Sheets`                        | 0–20 items                                                           |
| `Definition.ParameterDeclarations`         | 0–400 items                                                          |
| `Definition.CalculatedFields`              | 0–2000 items                                                         |
| `Definition.FilterGroups`                  | 0–2000 items                                                         |
| `SheetDefinition.Visuals`                  | 0–75 items per sheet                                                 |
| `SheetDefinition.Layouts`                  | **Exactly 1** item                                                   |
| `FilterGroup.Filters`                      | 0–20 items per group                                                 |
| `GridLayoutConfiguration.Elements`         | 0–430 items                                                          |

## Verified Gotchas (from implementation)

Lessons learned from actual dashboard-as-code deployments. These are not in the AWS docs and cause silent failures or confusing errors. Items marked with a reference file have full detail there; the summary here is for quick scanning.

### Analysis Permissions: all 7 actions required, no subsets

Every principal must have all 7 actions — omitting any (commonly `RestoreAnalysis` or `DescribeAnalysisPermissions`) causes async `CREATION_FAILED` with `VALIDATION_ERROR`. See [apis.md](./references/apis.md) §5 for the full action list.

### LineChart with Colors: max 1 Value field

When `LineChartAggregatedFieldWells.Colors` is populated, `Values` accepts **at most 1** field. Submitting 2+ Values with Colors causes synchronous `InvalidParameterValueException`.

Workaround: restructure the data so a single measure column holds all values and a dimension column distinguishes the series. For example, to show both planned and actual values on the same chart, add rows with `series_label = 'Actual'` and `value = actual_value` to the dataset, then use `value` as the single Value and `series_label` as Colors.

### Aggregated CalculatedFields: omit AggregationFunction

Already-aggregated calculated fields (e.g., `avg({error})`) must be referenced via `NumericalMeasureField` **without** `AggregationFunction`. Setting it causes async `INVALID_COLUMN_AGGREGATION`. See [definition-visuals.md](./references/definition-visuals.md) and [definition-calculated-fields.md](./references/definition-calculated-fields.md).

### LAC-W nested expressions: PRE_AGG_CALCULATION_LEVEL_MISMATCH

Nesting LAC-W functions like `sumOver(minOver(...))` causes async `PRE_AGG_CALCULATION_LEVEL_MISMATCH`. Workaround: pre-compute the intermediate value in the DataSet (`CreateColumnsOperation`) or flatten to a single LAC-W level.

### SheetControlLayouts: 12-column grid, max ColumnSpan 6

`SheetControlLayouts` uses a **12-column grid** (not 36). `ColumnSpan` max is 6. Exceeding causes synchronous `InvalidParameterValueException`. See [definition-sheets-layouts.md](./references/definition-sheets-layouts.md).

### S3 DataSource InputColumns: all STRING type

`S3Source.InputColumns` must all have `Type: "STRING"`. Cast via `CastColumnTypeOperation` in `LogicalTableMap.DataTransforms`.

### DataSet update-data-set: no Permissions parameter

`update-data-set` does not accept `Permissions`. Use `update-data-set-permissions` as a separate call.

### parseDate: no timezone offset pattern support

`parseDate` does not support `XXX` pattern for timezone offsets. Strip the offset with `substring` first, then adjust with `addDateTime`. See [definition-calculated-fields.md](./references/definition-calculated-fields.md) § Date function constraints.

### formatDate: limited pattern support

`"yyyy-MM"` is not supported; `formatDate` on `addDateTime` results may error. Use `toString` + `substring` for reliable display formatting. See [definition-calculated-fields.md](./references/definition-calculated-fields.md) § Date function constraints.

### SelectAllOptions overrides CategoryValues in FilterListConfiguration

`SelectAllOptions: "FILTER_ALL_VALUES"` causes `CategoryValues` to be **ignored**. Omit `SelectAllOptions` to use specific defaults. See [definition-filters.md](./references/definition-filters.md).

### Calculated field for dynamic granularity (single-visual approach)

Use a row-level calculated field with `ifelse` to switch the X-axis value based on a parameter (e.g., `display_key = ifelse(${granularity} = 'daily', substring({event_time}, 1, 10), ...)`). This avoids multiple visuals with conditional visibility and works with GridLayout.

### String-column range filter: calculated field + CategoryFilter pattern

`CategoryFilter` has no `>=`/`<=` operators. Workaround: create a calculated field like `range_flag = ifelse({hour_slot} >= ${slot_from} AND {hour_slot} <= ${slot_to}, 'in', 'out')`, then filter on `range_flag = 'in'` via `CustomFilterConfiguration`.

### DateTime display format control: string calculated field + CategoricalDimensionField

`DateDimensionField` with `DateGranularity` does not give full control over display format. Workaround: create a string calculated field (e.g., `concat(substring(toString({col}), 1, 10), ' ', substring(toString({col}), 12, 5))`) and use it as a `CategoricalDimensionField`.

## Editions

Some operations return `UnsupportedUserEditionException` on the Standard edition. Dashboard-as-code via `Definition` requires Enterprise edition (or higher). If the caller gets a 403 with that error code, check the account's QuickSight edition before debugging further.
