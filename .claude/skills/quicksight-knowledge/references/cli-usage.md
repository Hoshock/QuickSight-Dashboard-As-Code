# AWS CLI Usage for QuickSight Analyses

The `aws quicksight` CLI mirrors the API surface 1:1. Every API field has a corresponding `--kebab-case` flag, plus two meta-flags (`--generate-cli-skeleton`, `--cli-input-json`) that make the dashboard-as-code workflow tractable.

Always prefer `--cli-input-json file://...` for `create-analysis`, `update-analysis`, and `update-analysis-permissions`. Passing `--definition` inline is possible but practically unusable for any real analysis (shell quoting + JSON escaping + multi-megabyte bodies).

## The Skeleton Workflow

Canonical round-trip for net-new authoring:

```bash
# 1. Generate an empty JSON skeleton for the operation
aws quicksight create-analysis --generate-cli-skeleton > create-analysis.json

# 2. Edit create-analysis.json — fill in AwsAccountId, AnalysisId, Name, Definition, Permissions

# 3. Submit it
aws quicksight create-analysis --cli-input-json file://create-analysis.json
```

The same pattern applies to `update-analysis` and `update-analysis-permissions`. AWS recommends skeleton files specifically for those three commands because their bodies are large and complex.

For a describe → edit → update round-trip, skip the skeleton generator — use the real definition as the starting point:

```bash
# 1. Read current definition
aws quicksight describe-analysis-definition \
  --aws-account-id 123456789012 \
  --analysis-id my-analysis \
  > current.json

# 2. Shape it into an update-analysis body
jq '{
  AwsAccountId: "123456789012",
  AnalysisId: .AnalysisId,
  Name: .Name,
  Definition: .Definition,
  ThemeArn: .ThemeArn
}' current.json > update-analysis.json

# 3. Edit update-analysis.json...

# 4. Submit
aws quicksight update-analysis --cli-input-json file://update-analysis.json
```

`describe-analysis-definition` returns fields at the top level (`AnalysisId`, `Name`, `Definition`, `ThemeArn`, `ResourceStatus`, `Errors`). `update-analysis` expects a request body with those fields (minus `ResourceStatus`/`Errors`) plus `AwsAccountId`. The `jq` step above does that reshape.

## Command Cheat Sheet

| API                               | CLI                                                                                                       |
| --------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `CreateAnalysis`                  | `aws quicksight create-analysis`                                                                          |
| `UpdateAnalysis`                  | `aws quicksight update-analysis`                                                                          |
| `DescribeAnalysis`                | `aws quicksight describe-analysis`                                                                        |
| `DescribeAnalysisDefinition`      | `aws quicksight describe-analysis-definition`                                                             |
| `UpdateAnalysisPermissions`       | `aws quicksight update-analysis-permissions`                                                              |
| `DescribeAnalysisPermissions`     | `aws quicksight describe-analysis-permissions`                                                            |
| `DeleteAnalysis`                  | `aws quicksight delete-analysis --aws-account-id ... --analysis-id ... [--force-delete-without-recovery]` |
| `ListAnalyses` / `SearchAnalyses` | `aws quicksight list-analyses` / `search-analyses`                                                        |

### create-analysis required flags

```bash
aws quicksight create-analysis \
  --aws-account-id 123456789012 \
  --analysis-id my-analysis \
  --name "My Analysis" \
  --definition file://definition.json \
  # OR: --source-entity file://source-entity.json (mutually exclusive)
  [--parameters file://params.json] \
  [--permissions file://permissions.json] \
  [--tags Key=team,Value=analytics] \
  [--folder-arns arn:aws:quicksight:...:folder/...] \
  [--theme-arn arn:aws:quicksight:...:theme/...] \
  [--validation-strategy Mode=STRICT]
```

Any flag that takes a structured value (`--definition`, `--source-entity`, `--parameters`, `--permissions`, `--validation-strategy`, etc.) accepts either inline JSON or `file://path`. For anything longer than one line, use `file://`.

### describe-analysis-definition

```bash
aws quicksight describe-analysis-definition \
  --aws-account-id 123456789012 \
  --analysis-id my-analysis
```

Returns the full response to stdout. Pipe to `jq` or redirect to a file.

### update-analysis-permissions

```bash
aws quicksight update-analysis-permissions \
  --aws-account-id 123456789012 \
  --analysis-id my-analysis \
  --grant-permissions file://grant.json \
  --revoke-permissions file://revoke.json
```

Both flags are optional individually, but at least one must be present.

## Meta-flags

| Flag                                     | Purpose                                                                                                                                                               |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--generate-cli-skeleton`                | Output an empty JSON template for the command's inputs. Use `--generate-cli-skeleton output` (on some commands) for the response shape.                               |
| `--cli-input-json file://...`            | Submit the command using the JSON body from a file or inline JSON string. Any flag passed explicitly on the command line overrides the corresponding key in the JSON. |
| `--output json \| yaml \| table \| text` | Output format. Default `json`.                                                                                                                                        |
| `--query '<JMESPath>'`                   | Client-side filter on the response. Useful for polling: `--query 'Analysis.Status'`.                                                                                  |
| `--no-paginate`                          | Disable pagination for `list-*` / `search-*` commands.                                                                                                                |

## Polling Pattern

`CreateAnalysis` and `UpdateAnalysis` are async. Poll `DescribeAnalysis.Status`:

```bash
while true; do
  status=$(aws quicksight describe-analysis \
    --aws-account-id 123456789012 \
    --analysis-id my-analysis \
    --query 'Analysis.Status' --output text)
  case "$status" in
    *_SUCCESSFUL) echo "ok"; break ;;
    *_FAILED)
      aws quicksight describe-analysis \
        --aws-account-id 123456789012 \
        --analysis-id my-analysis \
        --query 'Analysis.Errors'
      exit 1
      ;;
    *_IN_PROGRESS) sleep 3 ;;
    *) echo "unexpected: $status"; exit 1 ;;
  esac
done
```

Typical run times: create/update completes in seconds for small analyses, but complex analyses with many visuals can take 30–60 s. Don't hardcode a fixed sleep — poll.

On `*_FAILED`, the one-liner above dumps `Analysis.Errors`. To turn those `ViolatedEntities[].Path` entries into the actual broken nodes of your local JSON submission, see [workflow.md §6](./workflow.md).

## Handling large `describe-analysis-definition` responses

Responses for non-trivial analyses routinely exceed 10 MB. The AWS CLI handles the network transfer fine, but common downstream issues:

```bash
# Safe baseline: redirect to file, then query locally
aws quicksight describe-analysis-definition \
  --aws-account-id 123456789012 \
  --analysis-id my-analysis \
  --no-cli-pager \
  --output json > current.json

jq '.Definition' current.json > definition.json      # Just the definition body
jq '.Errors'     current.json                        # Any validation issues
```

Common failures and fixes:

- **Terminal pager truncation** — The CLI by default pipes large responses through `less` (or `$AWS_PAGER`). If the shell shows a truncated file when redirecting, pass `--no-cli-pager` (or set `AWS_PAGER=""` in the environment) to disable it.
- **Shell pipe stall** — Piping directly into `jq` can hang on some shells if `jq` blocks on output. Prefer writing to a file first, then running `jq` against the file.
- **`jq` loads the whole doc** — For responses approaching 100 MB, `jq` may be slow. Use `jq -c` or stream with `jq --stream` if extracting a single path.
- **Corrupted output in CI logs** — `--output json` is safe; `--output table` / `text` can mangle large nested structures. Stick to `json` for definitions.
- **HTTP client buffers (SDK users)** — Some SDK HTTP clients have default response buffer limits < 10 MB. Increase the limit explicitly if using the SDK directly rather than the CLI.

The AWS CLI itself does not truncate or chunk — the response is one JSON blob. Pagination does not apply to this operation.

## Common Gotchas

- **`file://` vs `fileb://`**: QuickSight bodies are text JSON, so use `file://`. `fileb://` treats the file as raw bytes and base64-encodes it — wrong for this use case.
- **Windows paths**: `file://C:\path\to.json` requires forward-slashes or double-backslashes depending on shell; on Git Bash use `file:///c/path/to.json`.
- **Response size**: `describe-analysis-definition` responses can be > 10 MB. The AWS CLI handles this fine; SDK users may hit default HTTP buffer limits. See the "Handling large `describe-analysis-definition` responses" section above for the concrete recipe.
- **`AnalysisId` is case-sensitive** and appears in the QuickSight console URL. If you create `My-Analysis` and try to describe `my-analysis`, you get 404.
- **`Name` must be passed to `update-analysis` every call** even if unchanged. Omitting it is a 400.
- **Validation errors show up in `Errors[].ViolatedEntities[].Path`** as JSON-pointer-style paths (e.g., `/Sheets/0/Visuals/2/ChartConfiguration/FieldWells`). These point into the submitted `Definition`; use them to jump straight to the broken node.
- **`--cli-input-json` + explicit flags**: When both are present, the explicit flag wins. Handy for overriding `--aws-account-id` without regenerating the JSON.
- **Region**: QuickSight is regional. The CLI picks up region from `AWS_REGION` / `~/.aws/config` like any other service; analyses are not global.
- **Identity**: QuickSight has its own user registry (per-namespace). IAM permissions alone are not enough — the calling identity must be mapped to a QuickSight user with the right permissions.
- **Two error phases**: Schema violations (wrong field names, invalid enum values) return `InvalidParameterValueException` immediately (HTTP 400) — fix the JSON and resubmit. Semantic errors (column type mismatches, broken expressions, missing references) are async — the call returns 200/202 but `DescribeAnalysis` shows `*_FAILED` with `Errors[]`. Always poll even after a successful response.
- **Recovery from `CREATION_FAILED`**: The AnalysisId is consumed. Run `delete-analysis --force-delete-without-recovery` to free it, then re-run `create-analysis`.

## `--generate-cli-skeleton` vs the real schema

The skeleton emits a **shape** with empty strings and zero-length arrays for every field. It does not enforce "one-of" constraints — for example, it will emit both `Definition: {}` and `SourceEntity: {}` on a `create-analysis` skeleton, but submitting both is a 400. Delete the one you're not using before submitting.

## CLI Reference URLs

- `create-analysis`: https://docs.aws.amazon.com/cli/latest/reference/quicksight/create-analysis.html
- `update-analysis`: https://docs.aws.amazon.com/cli/latest/reference/quicksight/update-analysis.html
- `describe-analysis`: https://docs.aws.amazon.com/cli/latest/reference/quicksight/describe-analysis.html
- `describe-analysis-definition`: https://docs.aws.amazon.com/cli/latest/reference/quicksight/describe-analysis-definition.html
- `update-analysis-permissions`: https://docs.aws.amazon.com/cli/latest/reference/quicksight/update-analysis-permissions.html
- CLI skeleton guide: https://docs.aws.amazon.com/quicksight/latest/developerguide/cli-skeletons.html
