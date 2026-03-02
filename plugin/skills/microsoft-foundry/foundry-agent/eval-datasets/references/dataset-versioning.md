# Dataset Versioning — Version Management & Tagging

Manage dataset versions with naming conventions, tagging, and version pinning for reproducible evaluations. This workflow formalizes dataset lifecycle management using existing MCP tools and local conventions.

## Naming Convention

Use the pattern `<agent-name>-<source>-v<N>`:

| Component | Values | Example |
|-----------|--------|---------|
| `<agent-name>` | Agent name from `.env` | `support-bot` |
| `<source>` | `traces`, `synthetic`, `manual`, `combined` | `traces` |
| `v<N>` | Incremental version number | `v3` |

**Full examples:**
- `support-bot-traces-v1` — first dataset from trace harvesting
- `support-bot-synthetic-v2` — second synthetic dataset
- `support-bot-combined-v5` — fifth dataset combining traces + manual examples

## Tagging Conventions

Tags are stored in `datasets/manifest.json` alongside dataset metadata:

| Tag | Meaning | When to Apply |
|-----|---------|---------------|
| `baseline` | Reference dataset for comparison | When establishing a new evaluation baseline |
| `prod` | Dataset used for current production evaluation | After successful deployment |
| `canary` | Dataset for canary/staging evaluation | During staged rollout |
| `regression-<date>` | Dataset that caught a regression | When a regression is detected |
| `deprecated` | Dataset no longer in active use | When replaced by a newer version |

## Version Pinning

Pin evaluations to a specific dataset version to ensure reproducible, comparable results:

### Local Pinning (JSONL Datasets)

When using local JSONL files, reference the exact filename in evaluation runs:

```
datasets/support-bot-traces-v3.jsonl  ← pinned by filename
```

Pass the contents via `inputData` parameter in **`evaluation_agent_batch_eval_create`**.

### Server-Side Pinning

When using **`evaluation_dataset_create`**, set `datasetVersion` explicitly:

| Parameter | Required | Description |
|-----------|----------|-------------|
| `projectEndpoint` | ✅ | Azure AI Project endpoint |
| `datasetContentUri` | ✅ | Blob storage SAS URL |
| `datasetName` | Optional | Dataset name |
| `datasetVersion` | Optional | Version string (e.g., `"3"`) |

Use **`evaluation_dataset_get`** to retrieve a specific version:

| Parameter | Required | Description |
|-----------|----------|-------------|
| `projectEndpoint` | ✅ | Azure AI Project endpoint |
| `datasetName` | Optional | Dataset name to retrieve |
| `datasetVersion` | Optional | Version to retrieve (default: `"1"`) |

## Manifest File

Track all dataset versions, tags, and lineage in `datasets/manifest.json`:

```json
{
  "datasets": [
    {
      "name": "support-bot-traces-v1",
      "file": "support-bot-traces-v1.jsonl",
      "version": "1",
      "tag": "deprecated",
      "source": "trace-harvest",
      "harvestRule": "error",
      "timeRange": "2025-01-01 to 2025-01-07",
      "exampleCount": 32,
      "createdAt": "2025-01-08T10:00:00Z",
      "evalRunIds": ["run-abc-123"]
    },
    {
      "name": "support-bot-traces-v2",
      "file": "support-bot-traces-v2.jsonl",
      "version": "2",
      "tag": "baseline",
      "source": "trace-harvest",
      "harvestRule": "error+latency",
      "timeRange": "2025-01-15 to 2025-01-21",
      "exampleCount": 47,
      "createdAt": "2025-01-22T10:00:00Z",
      "evalRunIds": ["run-def-456", "run-ghi-789"]
    },
    {
      "name": "support-bot-traces-v3",
      "file": "support-bot-traces-v3.jsonl",
      "version": "3",
      "tag": "prod",
      "source": "trace-harvest",
      "harvestRule": "error+latency+low-eval",
      "timeRange": "2025-02-01 to 2025-02-07",
      "exampleCount": 63,
      "createdAt": "2025-02-08T10:00:00Z",
      "evalRunIds": []
    }
  ]
}
```

## Creating a New Version

1. **Check existing versions**: Read `datasets/manifest.json` to find the latest version number
2. **Increment version**: Use `v<N+1>` as the new version
3. **Create dataset**: Via [Trace-to-Dataset](trace-to-dataset.md) or manual JSONL creation
4. **Update manifest**: Add the new entry with metadata
5. **Tag appropriately**: Apply `baseline`, `prod`, or other tags as needed
6. **Deprecate old**: Optionally mark previous versions as `deprecated`

## Comparing Versions

To understand how a dataset evolved between versions:

```bash
# Count examples per version
wc -l datasets/support-bot-traces-v*.jsonl

# Diff example queries between versions
jq -r '.query' datasets/support-bot-traces-v2.jsonl | sort > /tmp/v2-queries.txt
jq -r '.query' datasets/support-bot-traces-v3.jsonl | sort > /tmp/v3-queries.txt
diff /tmp/v2-queries.txt /tmp/v3-queries.txt
```

## Next Steps

- **Organize into splits** → [Dataset Organization](dataset-organization.md)
- **Run evaluation with pinned version** → [observe skill Step 2](../../observe/references/evaluate-step.md)
- **Track lineage** → [Eval Lineage](eval-lineage.md)
