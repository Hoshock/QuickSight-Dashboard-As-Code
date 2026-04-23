# Dashboard-as-Code Workflow

End-to-end recipes for managing a QuickSight Analysis as JSON. These recipes assume CLI usage with a local definition file; adapt for SDK calls by treating `file://x.json` as a parsed object in the request body.

## 1. Bootstrap: create net-new from a definition

Use when an analysis does not exist yet.

```bash
# 1. Start from a skeleton
aws quicksight create-analysis --generate-cli-skeleton > create-analysis.json

# 2. Edit the file
#    - AwsAccountId:             "123456789012"
#    - AnalysisId:                "sales-overview"
#    - Name:                      "Sales Overview"
#    - delete the SourceEntity block (using Definition)
#    - fill Definition with DataSetIdentifierDeclarations + Sheets
#    - fill Permissions — grant ALL 7 actions for the author/owner principal:
#        quicksight:DescribeAnalysis, quicksight:DescribeAnalysisPermissions,
#        quicksight:UpdateAnalysis, quicksight:DeleteAnalysis,
#        quicksight:QueryAnalysis, quicksight:RestoreAnalysis,
#        quicksight:UpdateAnalysisPermissions
#      Omitting ANY action (commonly DescribeAnalysisPermissions) causes CREATION_FAILED.

# 3. Submit
aws quicksight create-analysis --cli-input-json file://create-analysis.json

# 4. Poll until terminal status
while :; do
  s=$(aws quicksight describe-analysis \
    --aws-account-id 123456789012 --analysis-id sales-overview \
    --query 'Analysis.Status' --output text)
  echo "status=$s"
  case "$s" in
    *_SUCCESSFUL) break ;;
    *_FAILED)
      aws quicksight describe-analysis \
        --aws-account-id 123456789012 --analysis-id sales-overview \
        --query 'Analysis.Errors' > errors.json
      cat errors.json; exit 1 ;;
    *_IN_PROGRESS) sleep 3 ;;
  esac
done
```

On `*_FAILED`, inspect `Errors[].ViolatedEntities[].Path` — each is a JSON-pointer-style path into your submitted `Definition`.

## 2. Read → Edit → Update: iterate on an existing analysis

The canonical dashboard-as-code loop.

```bash
ACCOUNT=123456789012
AID=sales-overview

# 1. Export the current definition
aws quicksight describe-analysis-definition \
  --aws-account-id $ACCOUNT --analysis-id $AID \
  > current.json

# 2. Reshape into an update-analysis request body
jq --arg acc "$ACCOUNT" '{
  AwsAccountId: $acc,
  AnalysisId: .AnalysisId,
  Name: .Name,
  ThemeArn: .ThemeArn,
  Definition: .Definition
}' current.json > update-analysis.json

# 3. Make your edits to update-analysis.json
#    (Examples: add a visual, tweak a filter, adjust a calculated field)

# 4. Submit
aws quicksight update-analysis --cli-input-json file://update-analysis.json

# 5. Poll (same as bootstrap)
```

`Name` must be sent on every `UpdateAnalysis` call, even unchanged. `Definition` is what actually changes.

Tip: commit `current.json` (or a normalized version) to version control so diffs show exactly what changed between revisions.

### Normalizing for diffs

Exported definitions often include server-assigned IDs and ordering that can produce noisy diffs. A stable normalization:

```bash
jq '.Definition' current.json \
  | jq -S .                                 # sort keys
  > current-normalized.json
```

`jq -S` gives deterministic key ordering. Array ordering is still significant (layout `ColumnIndex`/`RowIndex`, not position, drives placement — but the API preserves input order on round-trip).

## 3. Permissions-only change

```bash
cat > grant.json <<EOF
[
  { "Principal": "arn:aws:quicksight:us-east-1:123456789012:user/default/alice",
    "Actions":   ["quicksight:DescribeAnalysis", "quicksight:DescribeAnalysisPermissions", "quicksight:UpdateAnalysis", "quicksight:DeleteAnalysis", "quicksight:QueryAnalysis", "quicksight:RestoreAnalysis", "quicksight:UpdateAnalysisPermissions"] }
]
EOF

aws quicksight update-analysis-permissions \
  --aws-account-id 123456789012 \
  --analysis-id sales-overview \
  --grant-permissions file://grant.json
```

Call is synchronous — response carries the full post-update permission set.

## 4. Full round-trip (from scratch to iterating)

```
┌─────────────────────────────┐
│ Author initial Definition   │──┐
│ (by hand or from skeleton)  │  │
└─────────────────────────────┘  │
                                 ▼
┌─────────────────────────────┐
│ CreateAnalysis              │
│    Definition=...           │
│    Permissions=...          │
└─────────┬───────────────────┘
          │ returns CREATION_IN_PROGRESS
          ▼
┌─────────────────────────────┐
│ Poll DescribeAnalysis       │
│    until *_SUCCESSFUL       │◀──┐
└─────────┬───────────────────┘   │
          │                       │
          ▼                       │
┌─────────────────────────────┐   │
│ DescribeAnalysisDefinition  │   │
│   (capture canonical JSON)  │   │
└─────────┬───────────────────┘   │
          │                       │
          ▼                       │
┌─────────────────────────────┐   │
│ Edit Definition in place    │   │
└─────────┬───────────────────┘   │
          │                       │
          ▼                       │
┌─────────────────────────────┐   │
│ UpdateAnalysis              │   │
│    Definition=edited        │   │
└─────────┬───────────────────┘   │
          │ returns UPDATE_IN_PROGRESS
          └───────────────────────┘

(independent, anytime)
┌─────────────────────────────┐
│ UpdateAnalysisPermissions   │
│   (grants/revokes)          │
└─────────────────────────────┘
```

## 5. Common Failure Modes

| Symptom                                                       | Likely cause                                                                                                                                                                                                  |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ResourceExistsException` on `create-analysis`                | `AnalysisId` is taken. Use `update-analysis` instead.                                                                                                                                                         |
| `ResourceNotFoundException`                                   | Wrong case in `AnalysisId`; or dataset / theme ARN does not exist (or is in a different region); or caller has no `DescribeAnalysis` permission (QuickSight surfaces auth failures as 404 for some resources) |
| `UnsupportedUserEditionException`                             | Account is on QuickSight Standard — dashboard-as-code via `Definition` requires Enterprise                                                                                                                    |
| `CREATION_FAILED` / `UPDATE_FAILED` with non-empty `Errors[]` | Read `ViolatedEntities[].Path` — each Path is a JSON Pointer into the submitted `Definition`                                                                                                                  |
| `InvalidParameterValueException` at submission time           | **Synchronous** (HTTP 400). Wrong enum value, wrong field name, structural violation (e.g., `Definition` + `SourceEntity`). Fix JSON and resubmit — no polling needed.                                        |
| Visual renders but shows "No data"                            | Likely dataset column name mismatch — `ColumnIdentifier.ColumnName` is case-sensitive against the live dataset                                                                                                |
| UI shows correct analysis, API returns stale definition       | `describe-analysis-definition` reflects the last successful save; a failed update leaves the stored definition at the previous successful version                                                             |
| `CREATION_FAILED` with consumed AnalysisId                    | `CreateAnalysis` returns `ResourceExistsException`, `UpdateAnalysis` also fails. Run `delete-analysis --force-delete-without-recovery`, then re-run `CreateAnalysis`                                          |

## 6. Inspecting a Failure — `DescribeAnalysis` is the primary diagnostic

**This is how you find out what's wrong with a broken definition.** QuickSight validation is asynchronous — the `CreateAnalysis` / `UpdateAnalysis` HTTP response almost always says `*_IN_PROGRESS` and does **not** contain the root cause. The real error surfaces on `DescribeAnalysis` once the resource reaches `*_FAILED`.

Baseline recipe:

```bash
# After UpdateAnalysis / CreateAnalysis reached *_FAILED:
aws quicksight describe-analysis \
  --aws-account-id 123456789012 \
  --analysis-id sales-overview \
  --query 'Analysis.Errors' > errors.json
cat errors.json
```

`errors.json` will look like:

```json
[
  {
    "Type": "COLUMN_GEOGRAPHIC_ROLE_MISMATCH",
    "Message": "...",
    "ViolatedEntities": [
      {
        "Path": "/Sheets/0/Visuals/3/BarChartVisual/ChartConfiguration/FieldWells/BarChartAggregatedFieldWells/Values/0"
      }
    ]
  }
]
```

Each `Path` is a JSON-Pointer-style path into the submitted `Definition`. Common `Type` values:

| `Type`                                                                   | What it typically means                                                                             |
| ------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------- |
| `SOURCE_NOT_FOUND`                                                       | Referenced dataset / template / theme ARN not found                                                 |
| `DATA_SET_NOT_FOUND`                                                     | `DataSetArn` in `DataSetIdentifierDeclarations` is wrong or inaccessible                            |
| `COLUMN_TYPE_MISMATCH`                                                   | Column used in a well/filter of the wrong data type                                                 |
| `COLUMN_GEOGRAPHIC_ROLE_MISMATCH`                                        | Geo visual used a column without a geographic role                                                  |
| `COLUMN_REPLACEMENT_MISSING`                                             | A column the definition references was dropped from the dataset                                     |
| `PARAMETER_VALUE_INCOMPATIBLE`                                           | Parameter default type doesn't match declaration                                                    |
| `PARAMETER_TYPE_INVALID` / `PARAMETER_NOT_FOUND`                         | Filter / visual refers to a parameter name not declared                                             |
| `COLUMN_REFERENCE_NOT_FOUND`                                             | `ColumnIdentifier` points at `{DataSetIdentifier, ColumnName}` that doesn't exist                   |
| `CALCULATION_EXPRESSION_INVALID` / `CALCULATION_EXPRESSION_SYNTAX_ERROR` | `CalculatedField.Expression` failed to parse                                                        |
| `INTERNAL_FAILURE`                                                       | Service-side; usually retryable                                                                     |
| `COLUMN_TYPE_INCOMPATIBLE`                                               | MeasureField type doesn't match column data type (e.g., CategoricalMeasureField on DATETIME column) |

To jump straight to the broken node in your local submission JSON:

```bash
jq -r '.[].ViolatedEntities[].Path' errors.json | while read path; do
  echo "--- $path ---"
  python3 -c "
import json
doc = json.load(open('update-analysis.json'))
node = doc['Definition']
for p in '$path'.lstrip('/').split('/'):
    node = node[int(p)] if p.isdigit() else node[p]
print(json.dumps(node, indent=2)[:500])
"
done
```

If the analysis has been successfully saved at least once, `DescribeAnalysisDefinition` also returns the same `Errors[]` alongside the last-saved `Definition` — useful when the failure happened on a later update and you want to diff the on-disk version against the in-service one.

**Practical rule**: any time `DescribeAnalysis.Status` is `*_FAILED`, read `Analysis.Errors` _first_. Don't guess from the submitted JSON.

## 7. Rollback

There is no dedicated rollback operation. To revert:

1. Keep a copy of the last-known-good `Definition` (commit it to source control).
2. On regression, `UpdateAnalysis` with the known-good payload.

`DeleteAnalysis` enters a recycle-bin state for 30 days; `RestoreAnalysis` reverses it. Deletion is not part of the dashboard-as-code iteration loop and is out of scope here.

## 8. Referencing a Managed Definition from Code

When treating a `Definition` as source-controlled code, a few conventions help:

- **Strip account/region-specific ARNs** at export time; interpolate them back at apply time. This keeps the source portable.
- **Keep `AnalysisId` and `Name` out of the definition file** — pass them as CLI flags or as templating variables. Definitions are otherwise reusable across environments.
- **Generate `DataSetIdentifierDeclarations` from a manifest** listing dataset ARN per environment — swap at deploy time.
- **Sort `ParameterDeclarations`, `CalculatedFields`, `FilterGroups`, and `Sheets` by ID in the stored file**. Round-tripped definitions from `DescribeAnalysisDefinition` may come back in a different order; a deterministic order keeps diffs clean.
