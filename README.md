# QuickSight Dashboard as Code — AI Agent Skill

Knowledge base for AI coding agents to manage Amazon QuickSight Analyses as JSON definitions (`AnalysisDefinition`).

## What is this?

A skill file set that AI coding agents like [Claude Code](https://github.com/anthropics/claude-code) and [Kiro](https://kiro.dev) reference when executing QuickSight Dashboard as Code workflows.

Loading this skill enables an agent to:

- Create and update Analyses via `CreateAnalysis` / `UpdateAnalysis`
- Build `AnalysisDefinition` JSON (sheets, visuals, filters, parameters, calculated fields)
- Diagnose and fix async validation errors
- Execute the `DescribeAnalysisDefinition` → edit → `UpdateAnalysis` round-trip

## Setup

Copy into your project's `.claude/skills/`:

```bash
git clone https://github.com/Hoshock/QuickSight-Dashboard-As-Code.git
cp -r QuickSight-Dashboard-As-Code/.claude/skills/quicksight-knowledge \
  your-project/.claude/skills/
```

## File Structure

```
.claude/skills/quicksight-knowledge/
├── SKILL.md                              # Entry point (overview, constraints, gotchas)
└── references/
    ├── apis.md                           # API operations (6 operations in detail)
    ├── cli-usage.md                      # AWS CLI usage
    ├── workflow.md                       # Round-trip workflow
    ├── definition-top-level.md           # AnalysisDefinition top-level structure
    ├── definition-sheets-layouts.md      # Sheets and layouts
    ├── definition-visuals.md             # 25 visual types
    ├── definition-filters.md             # 8 filter types
    ├── definition-parameters.md          # Parameter declarations
    └── definition-calculated-fields.md   # Calculated fields and expression language
```

## Contents

| Category          | Coverage                                                                                                                            |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| API Reference     | `CreateAnalysis`, `UpdateAnalysis`, `DescribeAnalysis`, `DescribeAnalysisDefinition`, `UpdateAnalysisPermissions`, `DeleteAnalysis` |
| CLI Guide         | `--cli-input-json file://` pattern, skeleton workflow, polling pattern                                                              |
| Definition Schema | Top-level structure, sheets/layouts, 25 visual types, 8 filter types, 4 parameter types, calculated fields                          |
| Verified Gotchas  | 14 undocumented pitfalls discovered through real deployments                                                                        |

## Verified Gotchas (excerpt)

Pitfalls not covered in the AWS documentation, discovered through actual deployments:

- **Analysis permissions require all 7 actions** — omitting any causes async `CREATION_FAILED`
- **LineChart with Colors: max 1 Value field**
- **Aggregated CalculatedFields must omit AggregationFunction**
- **SheetControlLayouts uses a 12-column grid** (not 36 like the main layout)
- **`parseDate` does not support timezone offset patterns**
- **`SelectAllOptions` overrides `CategoryValues`**

See [SKILL.md](.claude/skills/quicksight-knowledge/SKILL.md) for all 14 items.

## Prerequisites

- QuickSight **Enterprise edition** or higher (Dashboard as Code via `Definition` is not available on Standard)
- DataSets must be created beforehand (Analyses reference existing DataSet ARNs)

## License

MIT
