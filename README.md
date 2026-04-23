# QuickSight Dashboard as Code — AI Agent Skill

Amazon QuickSight の Analysis を JSON 定義（`AnalysisDefinition`）で管理するための AI エージェント向けナレッジベースです。

## これは何？

[Claude Code](https://github.com/anthropics/claude-code) や [Kiro](https://kiro.dev) などの AI コーディングエージェントが QuickSight の Dashboard as Code ワークフローを実行する際に参照するスキルファイル群です。

エージェントがこのスキルを読み込むことで、以下を正確に実行できるようになります：

- `CreateAnalysis` / `UpdateAnalysis` による Analysis の作成・更新
- `AnalysisDefinition` JSON の構築（シート、ビジュアル、フィルタ、パラメータ、計算フィールド）
- 非同期バリデーションエラーの診断と修正
- `DescribeAnalysisDefinition` → 編集 → `UpdateAnalysis` のラウンドトリップ

## セットアップ

プロジェクトの `.claude/skills/` にコピーするだけです：

```bash
git clone https://github.com/Hoshock/QuickSight-Dashboard-As-Code.git
cp -r QuickSight-Dashboard-As-Code/.claude/skills/quicksight-knowledge \
  your-project/.claude/skills/
```

## ファイル構成

```
.claude/skills/quicksight-knowledge/
├── SKILL.md                              # エントリポイント（概要、制約、Gotchas）
└── references/
    ├── apis.md                           # API オペレーション（6 操作の詳細）
    ├── cli-usage.md                      # AWS CLI の使い方
    ├── workflow.md                       # ラウンドトリップワークフロー
    ├── definition-top-level.md           # AnalysisDefinition トップレベル構造
    ├── definition-sheets-layouts.md      # シートとレイアウト
    ├── definition-visuals.md             # 25 種類のビジュアルタイプ
    ├── definition-filters.md             # 8 種類のフィルタ
    ├── definition-parameters.md          # パラメータ宣言
    └── definition-calculated-fields.md   # 計算フィールドと式言語
```

## 含まれる内容

| カテゴリ            | 内容                                                                                                                                |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| API リファレンス    | `CreateAnalysis`, `UpdateAnalysis`, `DescribeAnalysis`, `DescribeAnalysisDefinition`, `UpdateAnalysisPermissions`, `DeleteAnalysis` |
| CLI ガイド          | `--cli-input-json file://` パターン、スケルトンワークフロー、ポーリングパターン                                                     |
| Definition スキーマ | トップレベル構造、シート/レイアウト、25 ビジュアルタイプ、8 フィルタタイプ、4 パラメータタイプ、計算フィールド                      |
| Verified Gotchas    | AWS ドキュメントに記載のない実装上の落とし穴（14 項目）                                                                             |

## Verified Gotchas（抜粋）

実際のデプロイで発見された、AWS ドキュメントに記載のない落とし穴です：

- **Analysis の権限は全 7 アクション必須** — 1 つでも欠けると非同期で `CREATION_FAILED`
- **LineChart の Colors 使用時は Values が 1 つまで**
- **集約済み CalculatedField に AggregationFunction を設定してはいけない**
- **SheetControlLayouts は 12 列グリッド**（メインレイアウトの 36 列ではない）
- **`parseDate` はタイムゾーンオフセットパターン非対応**
- **`SelectAllOptions` を指定すると `CategoryValues` は無視される**

全 14 項目は [SKILL.md](.claude/skills/quicksight-knowledge/SKILL.md) を参照してください。

## 前提条件

- QuickSight **Enterprise エディション**以上（`Definition` による dashboard-as-code は Standard では利用不可）
- DataSet が事前に作成済みであること（Analysis は既存の DataSet ARN を参照する）

## ライセンス

MIT
