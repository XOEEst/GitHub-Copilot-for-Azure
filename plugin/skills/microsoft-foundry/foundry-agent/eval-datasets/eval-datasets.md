# Evaluation Datasets — Trace-to-Dataset Pipeline & Lifecycle Management

Manage the full lifecycle of evaluation datasets for Foundry agents — from harvesting production traces into test datasets, through versioning and organization, to evaluation trending and regression detection. This skill closes the gap between **production observability** and **evaluation quality** by turning real-world agent failures into reproducible test cases.

## When to Use This Skill

USE FOR: create dataset from traces, harvest traces into dataset, build test dataset, dataset versioning, version my dataset, tag dataset, pin dataset version, organize datasets, dataset splits, curate test cases, review trace candidates, evaluation trending, metrics over time, eval regression, regression detection, compare evaluations over time, dataset comparison, evaluation lineage, trace to dataset pipeline, annotation review, production traces to test cases.

> ⚠️ **DO NOT manually run** KQL queries to extract datasets or call `evaluation_dataset_create` **without reading this skill first.** This skill defines the correct trace extraction patterns, schema transformation, versioning conventions, and quality gates that raw tools do not enforce.

> 💡 **Tip:** This skill complements the [observe skill](../observe/observe.md) (eval-driven optimization loop) and the [trace skill](../trace/trace.md) (production trace analysis). Use this skill when you need to **bridge traces and evaluations** — turning production data into test cases and tracking evaluation quality over time.

## Quick Reference

| Property | Value |
|----------|-------|
| MCP server | `foundry-mcp` |
| Key MCP tools | `evaluation_dataset_create`, `evaluation_dataset_get`, `evaluation_get`, `evaluation_comparison_create`, `evaluation_comparison_get` |
| Azure services | Application Insights (KQL), Azure Blob Storage |
| Prerequisites | Agent deployed, App Insights connected (see [trace skill](../trace/trace.md)) |
| Artifact paths | `datasets/`, `results/`, `evaluators/` |

## Entry Points

| User Intent | Start At |
|-------------|----------|
| "Create dataset from production traces" / "Harvest traces" | [Trace-to-Dataset Pipeline](references/trace-to-dataset.md) |
| "Version my dataset" / "Tag dataset" / "Pin dataset version" | [Dataset Versioning](references/dataset-versioning.md) |
| "Organize my datasets" / "Dataset splits" / "Filter datasets" | [Dataset Organization](references/dataset-organization.md) |
| "Review trace candidates" / "Curate test cases" | [Dataset Curation](references/dataset-curation.md) |
| "Show eval metrics over time" / "Evaluation trending" | [Eval Trending](references/eval-trending.md) |
| "Did my agent regress?" / "Regression detection" | [Eval Regression](references/eval-regression.md) |
| "Compare datasets" / "Experiment comparison" / "A/B test" | [Dataset Comparison](references/dataset-comparison.md) |
| "Trace my evaluation lineage" / "Audit eval history" | [Eval Lineage](references/eval-lineage.md) |

## Before Starting — Detect Current State

1. Check `.env` for `AZURE_AI_PROJECT_ENDPOINT`, `AZURE_AI_AGENT_NAME`, and `APPLICATIONINSIGHTS_CONNECTION_STRING`
2. If App Insights is missing, resolve via [trace skill](../trace/trace.md) (Before Starting section)
3. Check `datasets/` for existing datasets and `results/` for evaluation history
4. Check if `evaluation_dataset_get` returns any server-side datasets
5. Route to the appropriate entry point based on user intent

## The Foundry Flywheel

This skill enables a closed-loop improvement cycle where production failures become regression tests:

```
Production Agent → [1] Trace (App Insights + OTel)
                → [2] Harvest (KQL extraction)
                → [3] Curate (human review)
                → [4] Dataset (versioned, tagged)
                → [5] Evaluate (batch eval)
                → [6] Analyze (trending + regression)
                → [7] Compare (version diff)
                → [8] Deploy → back to [1]
```

Each cycle makes the test suite harder and more representative. Production failures from release N become regression tests for release N+1.

## Behavioral Rules

1. **Always show KQL queries.** Before executing any trace extraction query, display it in a code block. Never run queries silently.
2. **Scope to time ranges.** Always include a time range in KQL queries (default: last 7 days for trace harvesting). Ask user for the range if not specified.
3. **Require human review.** Never auto-commit harvested traces to a dataset without showing candidates to the user first. The curation step is mandatory.
4. **Use versioning conventions.** Follow the naming pattern `<agent-name>-<source>-v<N>` (e.g., `support-bot-traces-v3`).
5. **Persist artifacts.** Save datasets to `datasets/`, evaluation results to `results/`, and track lineage in `datasets/manifest.json`.
6. **Confirm before overwriting.** If a dataset version already exists, warn the user and ask for confirmation before replacing.

## Related Skills

| User Intent | Skill |
|-------------|-------|
| "Run an evaluation" / "Optimize my agent" | [observe skill](../observe/observe.md) |
| "Search traces" / "Analyze failures" / "Latency analysis" | [trace skill](../trace/trace.md) |
| "Deploy my agent" | [deploy skill](../deploy/deploy.md) |
| "Debug container issues" | [troubleshoot skill](../troubleshoot/troubleshoot.md) |
