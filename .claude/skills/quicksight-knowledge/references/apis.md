# QuickSight Analysis API Operations

Full reference for the five API operations in the dashboard-as-code workflow. All operations are scoped to an AWS account via path parameter `AwsAccountId`.

## Common Path Parameters

| Parameter      | Type   | Constraint                                                                  |
| -------------- | ------ | --------------------------------------------------------------------------- |
| `AwsAccountId` | String | Fixed length 12, pattern `^[0-9]{12}$`                                      |
| `AnalysisId`   | String | 1–512 chars, pattern `[\w\-]+`; case-sensitive; appears in the analysis URL |

## Common Error Codes

These codes are returned by every operation in this file (each operation lists additional operation-specific codes below).

| HTTP | Error                             | Meaning                                                                                                                                                                                                                                                                                       |
| ---- | --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 400  | `InvalidParameterValueException`  | **Synchronous** rejection (HTTP 400). Covers: malformed request body, invalid path parameters, wrong enum values, unrecognized field names, structural constraint violations (e.g., sending both `Definition` and `SourceEntity`). Distinct from async errors in `DescribeAnalysis.Errors[]`. |
| 403  | `UnsupportedUserEditionException` | Operation not available on current QuickSight edition (dashboard-as-code requires Enterprise)                                                                                                                                                                                                 |
| 404  | `ResourceNotFoundException`       | Analysis or referenced resource (dataset, theme, template) missing                                                                                                                                                                                                                            |
| 409  | `ConflictException`               | Resource is in a state that prevents the change (e.g., `*_IN_PROGRESS`)                                                                                                                                                                                                                       |
| 429  | `ThrottlingException`             | Rate-limited; retry with backoff                                                                                                                                                                                                                                                              |
| 500  | `InternalFailureException`        | Service-side failure; retry                                                                                                                                                                                                                                                                   |

`AccessDeniedException` (HTTP 401) is **not** returned uniformly by every operation — only `DescribeAnalysis` and `DescribeAnalysisDefinition` document it. The mutating operations (`CreateAnalysis`, `UpdateAnalysis`, `UpdateAnalysisPermissions`) surface authorization failures through other codes (typically 404 `ResourceNotFoundException` for missing-resource-or-no-permission or 400 `InvalidParameterValueException`).

---

## 1. CreateAnalysis

Create a new analysis. Returns asynchronously — poll `DescribeAnalysis` for final status.

**HTTP**

```
POST /accounts/{AwsAccountId}/analyses/{AnalysisId}
Content-Type: application/json
```

**Request body**

| Field                | Type                   | Required  | Notes                                                                                                                          |
| -------------------- | ---------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `Name`               | String                 | ✅        | 1–2048 chars                                                                                                                   |
| `Definition`         | `AnalysisDefinition`   | ⚠️ one-of | Full dashboard-as-code body. Mutually exclusive with `SourceEntity`                                                            |
| `SourceEntity`       | `AnalysisSourceEntity` | ⚠️ one-of | `{ SourceTemplate: { Arn, DataSetReferences: [{ DataSetArn, DataSetPlaceholder }] } }`. Mutually exclusive with `Definition`   |
| `Parameters`         | `Parameters`           | ❌        | Override values: `StringParameters`, `IntegerParameters`, `DecimalParameters`, `DateTimeParameters`, each `[{ Name, Values }]` |
| `Permissions`        | `ResourcePermission[]` | ❌        | 1–64 items; `[{ Principal, Actions[] }]`. Omit to create with no explicit grants                                               |
| `Tags`               | `Tag[]`                | ❌        | 1–200 items                                                                                                                    |
| `FolderArns`         | `String[]`             | ❌        | Max 1 ARN — the folder the analysis is created inside                                                                          |
| `ThemeArn`           | String                 | ❌        | Theme must be accessible to the principal                                                                                      |
| `ValidationStrategy` | `ValidationStrategy`   | ❌        | `{ Mode: "STRICT" \| "LENIENT" }` — `LENIENT` downgrades definition errors to warnings where supported                         |

**Either `Definition` or `SourceEntity` must be present.** Sending both is a 400.

**Response (200)**

```json
{
  "AnalysisId": "string",
  "Arn": "string",
  "CreationStatus": "CREATION_IN_PROGRESS",
  "RequestId": "string"
}
```

`CreationStatus` ∈ `CREATION_IN_PROGRESS | CREATION_SUCCESSFUL | CREATION_FAILED | UPDATE_IN_PROGRESS | UPDATE_SUCCESSFUL | UPDATE_FAILED | DELETED`. The response almost always returns `CREATION_IN_PROGRESS`; terminal status is reached asynchronously and must be polled via `DescribeAnalysis`.

**Additional errors**

| HTTP | Error                                                                                                        |
| ---- | ------------------------------------------------------------------------------------------------------------ |
| 409  | `ResourceExistsException` — an analysis with that `AnalysisId` already exists (use `UpdateAnalysis` instead) |
| 409  | `LimitExceededException` — account-level analysis quota hit                                                  |

---

## 2. UpdateAnalysis

Update the name, definition, or theme of an existing analysis. Does **not** modify permissions.

**HTTP**

```
PUT /accounts/{AwsAccountId}/analyses/{AnalysisId}
Content-Type: application/json
```

**Request body**

| Field                | Type                   | Required  | Notes                                                                                                                                                                                                |
| -------------------- | ---------------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Name`               | String                 | ✅        | Required even if unchanged — 1–2048 chars                                                                                                                                                            |
| `Definition`         | `AnalysisDefinition`   | ⚠️ one-of | Mutually exclusive with `SourceEntity`                                                                                                                                                               |
| `SourceEntity`       | `AnalysisSourceEntity` | ⚠️ one-of | Mutually exclusive with `Definition`                                                                                                                                                                 |
| `Parameters`         | `Parameters`           | ❌        | Parameter override values                                                                                                                                                                            |
| `ThemeArn`           | String                 | ❌        | To remove the theme, pass an empty string is **not** supported — the field is optional, omit to leave unchanged on some SDKs, but the API replaces the whole analysis representation, so be explicit |
| `ValidationStrategy` | `ValidationStrategy`   | ❌        | `STRICT` (default) or `LENIENT`                                                                                                                                                                      |

**Response (200)**

```json
{
  "AnalysisId": "string",
  "Arn": "string",
  "UpdateStatus": "UPDATE_IN_PROGRESS",
  "RequestId": "string"
}
```

`UpdateStatus` uses the same enum as `CreationStatus`. Poll `DescribeAnalysis` for terminal state.

**Additional errors**

| HTTP | Error                                                                                                                   |
| ---- | ----------------------------------------------------------------------------------------------------------------------- |
| 409  | `ResourceExistsException` — rarely surfaced for updates (resource name collision in a rename scenario on some editions) |

**Key differences vs. CreateAnalysis**

- `POST` vs `PUT`
- `Name` is required (cannot omit even for no-op rename)
- `Permissions`, `Tags`, `FolderArns` are **not** accepted — manage those via `UpdateAnalysisPermissions`, `TagResource`/`UntagResource`, and `UpdateFolderPermissions` respectively

---

## 3. DescribeAnalysis

Returns metadata + status. Does **not** return the full `Definition` — use `DescribeAnalysisDefinition` for that.

**HTTP**

```
GET /accounts/{AwsAccountId}/analyses/{AnalysisId}
```

**Response (200)**

```json
{
  "Analysis": {
    "AnalysisId": "string",
    "Arn": "string",
    "Name": "string",
    "Status": "CREATION_SUCCESSFUL",
    "CreatedTime": 1700000000,
    "LastUpdatedTime": 1700000000,
    "DataSetArns": ["string"],
    "ThemeArn": "string",
    "Sheets": [
      { "SheetId": "string", "Name": "string", "Images": [ ... ] }
    ],
    "Errors": [
      {
        "Type": "string",
        "Message": "string",
        "ViolatedEntities": [ { "Path": "string" } ]
      }
    ]
  },
  "RequestId": "string"
}
```

Notable fields:

- `Status` — poll until it leaves `*_IN_PROGRESS`. `Analysis.Errors[]` surfaces validation failures from the async definition check
- `Errors[]` — populated on `*_FAILED`; `ViolatedEntities[].Path` is a JSON pointer into the definition (e.g., `/Sheets/0/Visuals/3/BarChartVisual/ChartConfiguration/FieldWells`)
- `DataSetArns` — flat list of datasets referenced anywhere in the analysis
- `Sheets[].Images` — sheet images only; full sheet content is **not** here

Use this API for lightweight polling. Do not use it to reconstruct the analysis definition.

### Primary use: deploy-time error diagnosis

`CreateAnalysis` and `UpdateAnalysis` validate the submitted `Definition` asynchronously. The 200 response on those calls does **not** mean the definition is valid — it means it was accepted for processing. The **terminal status + `Errors[]` returned by `DescribeAnalysis` is where validation failures appear** (missing required fields, unknown enum values, references to non-existent parameters/columns/visual IDs, invalid expressions, etc.).

Always call `DescribeAnalysis` after create/update, check `Status`, and if it is `CREATION_FAILED` / `UPDATE_FAILED`, read `Errors[]`. See [workflow.md §6](./workflow.md) for the recipe and the common `Errors[].Type` values.

---

## 4. DescribeAnalysisDefinition

Returns the full dashboard-as-code JSON. This is the canonical "read" side of the round-trip.

**HTTP**

```
GET /accounts/{AwsAccountId}/analyses/{AnalysisId}/definition
```

**Response (200)**

```json
{
  "AnalysisId": "string",
  "Name": "string",
  "ResourceStatus": "CREATION_SUCCESSFUL",
  "ThemeArn": "string",
  "Definition": {
    /* full AnalysisDefinition — see definition-top-level.md */
  },
  "Errors": [
    {
      "Type": "string",
      "Message": "string",
      "ViolatedEntities": [{ "Path": "string" }]
    }
  ],
  "RequestId": "string"
}
```

The `Definition` object returned here is **round-trippable** — feed it (unmodified) back into `UpdateAnalysis.Definition` and the analysis is unchanged, apart from refreshed timestamps.

Caveats:

- Returns the last-saved definition, including definitions with `*_FAILED` status. Inspect `Errors[]` before trusting the payload.
- Very large. Page size is unlimited but responses can exceed 10 MB on complex analyses — budget for this in streaming/CLI buffers.

---

## 5. UpdateAnalysisPermissions

Grant or revoke resource-level access. Independent of `UpdateAnalysis`.

**HTTP**

```
PUT /accounts/{AwsAccountId}/analyses/{AnalysisId}/permissions
Content-Type: application/json
```

**Request body**

```json
{
  "GrantPermissions": [
    {
      "Principal": "arn:aws:quicksight:...:user/default/alice",
      "Actions": ["quicksight:DescribeAnalysis"]
    }
  ],
  "RevokePermissions": [
    {
      "Principal": "arn:aws:quicksight:...:group/default/legacy",
      "Actions": ["quicksight:UpdateAnalysis"]
    }
  ]
}
```

Both arrays are optional (max 100 items each), but sending an empty body is a 400 — include at least one of the two. Both sides accept the same `ResourcePermission` shape.

Valid QuickSight analysis action strings:

- `quicksight:DescribeAnalysis`
- `quicksight:UpdateAnalysis`
- `quicksight:DeleteAnalysis`
- `quicksight:QueryAnalysis`
- `quicksight:RestoreAnalysis`
- `quicksight:DescribeAnalysisPermissions`
- `quicksight:UpdateAnalysisPermissions`

> **⚠️ The permission set is validated as a whole.** `CreateAnalysis` requires the `Actions` array to be the **complete set of all seven** actions — omitting any single action (commonly `RestoreAnalysis` or `DescribeAnalysisPermissions`) causes `CREATION_FAILED` with `VALIDATION_ERROR` at the **async** phase, not at submission time. `UpdateAnalysisPermissions` also enforces the full set. You cannot grant a subset (e.g., read-only with just `DescribeAnalysis` + `QueryAnalysis`). Every principal on an Analysis must have all seven actions.

**Note on action namespaces.** QuickSight resource permissions use action strings that overlap with — but are **not identical to** — the IAM action list (the IAM Service Authorization Reference). For example, `quicksight:QueryAnalysis` / `quicksight:QueryDashboard` are valid for QuickSight resource-level `Actions` (as documented by `UpdateAnalysisPermissions` / `UpdateDashboardPermissions`) but do not appear in the IAM-policy action list — they're QuickSight-internal grants, not IAM actions. If authoring an IAM policy rather than a `ResourcePermission`, consult the IAM Service Authorization Reference instead. Some SDKs accept the short form (e.g., `DescribeAnalysis` without prefix), but the `quicksight:` prefix is the canonical form.

**Response (200)**

```json
{
  "AnalysisArn": "string",
  "AnalysisId": "string",
  "Permissions": [{ "Principal": "string", "Actions": ["string"] }],
  "RequestId": "string"
}
```

`Permissions` reflects the **full** post-update permission set, not a diff.

**Additional errors**

| HTTP | Error                                                                                       |
| ---- | ------------------------------------------------------------------------------------------- |
| 409  | `LimitExceededException` — total permissions on the analysis would exceed the account limit |

---

## 6. DeleteAnalysis

Delete an analysis. Primarily used for recovering from `CREATION_FAILED`.

**HTTP**

```
DELETE /accounts/{AwsAccountId}/analyses/{AnalysisId}
```

**Request parameters**

| Parameter                    | Type    | Required | Notes                                                                                                                                                                                                            |
| ---------------------------- | ------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ForceDeleteWithoutRecovery` | Boolean | ❌       | Default `false`. When `true`, bypasses the 30-day recycle bin and permanently deletes immediately. **Required for recovering from `CREATION_FAILED`** — without it, the AnalysisId remains occupied for 30 days. |
| `RecoveryWindowInDays`       | Long    | ❌       | 7–30. Days before permanent deletion. Ignored when `ForceDeleteWithoutRecovery` is `true`.                                                                                                                       |

**Response (200)**

```json
{
  "AnalysisId": "string",
  "Arn": "string",
  "DeletionTime": "timestamp",
  "RequestId": "string"
}
```

**Recovery from CREATION_FAILED:**

```bash
# AnalysisId is consumed after CREATION_FAILED — CreateAnalysis returns ResourceExistsException,
# UpdateAnalysis also fails. Must delete first:
aws quicksight delete-analysis \
  --aws-account-id 123456789012 --analysis-id my-analysis \
  --force-delete-without-recovery
# Then re-run CreateAnalysis with the fixed definition.
```

---

## Companion: DescribeAnalysisPermissions

Read the current permission set. Often paired with `UpdateAnalysisPermissions` to verify grants after an update or to introspect access before editing.

**HTTP**

```
GET /accounts/{AwsAccountId}/analyses/{AnalysisId}/permissions
```

**Response (200)**

```json
{
  "AnalysisArn": "string",
  "AnalysisId": "string",
  "Permissions": [{ "Principal": "string", "Actions": ["string"] }],
  "RequestId": "string"
}
```

No request body. Same error set as `DescribeAnalysis`.

---

## Shared Shapes

### `ResourcePermission`

```json
{ "Principal": "string", "Actions": ["string"] }
```

`Principal` is an ARN of a QuickSight user, group, or namespace. `Actions` is an array of action strings (see the list above for analysis-valid actions).

### `ValidationStrategy`

```json
{ "Mode": "STRICT" | "LENIENT" }
```

`LENIENT` allows some categories of definition-validation errors to become warnings; still returns `*_FAILED` for structural errors.

### `Parameters`

```json
{
  "StringParameters": [{ "Name": "string", "Values": ["string"] }],
  "IntegerParameters": [{ "Name": "string", "Values": [0] }],
  "DecimalParameters": [{ "Name": "string", "Values": [0.0] }],
  "DateTimeParameters": [
    { "Name": "string", "Values": ["2023-11-14T00:00:00.000Z"] }
  ]
}
```

These are runtime **override** values for parameters declared in `AnalysisDefinition.ParameterDeclarations`. They seed the analysis with specific defaults on create/update; they do not change the declarations themselves.
