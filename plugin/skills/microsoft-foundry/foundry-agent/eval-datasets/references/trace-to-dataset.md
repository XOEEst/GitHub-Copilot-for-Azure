# Trace-to-Dataset Pipeline — Harvest Production Traces as Test Cases

Extract production traces from App Insights using KQL, transform them into evaluation dataset format, and persist as versioned datasets. This is the core workflow for turning real-world agent failures into reproducible test cases.

## ⛔ Do NOT

- Do NOT upload datasets to blob storage or call `evaluation_dataset_create` — this MCP tool is not ready.
- Do NOT generate SAS URLs. Local JSONL + `inputData` is the only supported path.

## Prerequisites

- App Insights resource resolved (see [trace skill](../../trace/trace.md) Before Starting)
- Agent name and project endpoint available in `.env`
- Time range confirmed with user (default: last 7 days)

> 💡 **Run all KQL queries** using **`monitor_resource_log_query`** (Azure MCP tool) against the App Insights resource. This is preferred over delegating to the `azure-kusto` skill.

## Overview

```
App Insights traces
    │
    ▼
[1] KQL Harvest Query (filter by error/latency/eval score)
    │
    ▼
[2] Schema Transform (trace → JSONL format)
    │
    ▼
[3] Human Review (show candidates, let user approve/edit/reject)
    │
    ▼
[4] Persist Dataset (local JSONL files)
```

## Step 1 — Choose a Harvest Template

Select the appropriate KQL template based on user intent. These templates mirror common LangSmith "run rules" but offer more power through KQL's query language.

### Error Harvest — Failed Traces

Captures all traces where the agent returned errors. Equivalent to LangSmith's `eq(error, True)` run rule.

```kql
dependencies
| where timestamp > ago(7d)
| where success == false
| where isnotempty(customDimensions["gen_ai.operation.name"])
| where customDimensions["gen_ai.agent.name"] == "<agent-name>"
    or customDimensions["azure.ai.agentserver.agent_name"] == "<agent-name>"
| extend
    conversationId = tostring(customDimensions["gen_ai.conversation.id"]),
    responseId = tostring(customDimensions["gen_ai.response.id"]),
    operation = tostring(customDimensions["gen_ai.operation.name"]),
    model = tostring(customDimensions["gen_ai.request.model"]),
    errorType = tostring(customDimensions["error.type"]),
    inputTokens = toint(customDimensions["gen_ai.usage.input_tokens"]),
    outputTokens = toint(customDimensions["gen_ai.usage.output_tokens"])
| summarize
    errorCount = count(),
    errors = make_set(errorType, 5),
    firstSeen = min(timestamp),
    lastSeen = max(timestamp)
    by conversationId, responseId, operation, model
| order by lastSeen desc
| take 100
```

### Low-Eval Harvest — Traces with Poor Evaluation Scores

Captures traces where evaluator scores fell below a threshold. Equivalent to LangSmith's `and(eq(feedback_key, "quality"), lt(feedback_score, 0.3))` run rule.

```kql
let lowEvalResponses = customEvents
| where timestamp > ago(7d)
| where name == "gen_ai.evaluation.result"
| extend
    score = todouble(customDimensions["gen_ai.evaluation.score.value"]),
    evalName = tostring(customDimensions["gen_ai.evaluation.name"]),
    responseId = tostring(customDimensions["gen_ai.response.id"]),
    conversationId = tostring(customDimensions["gen_ai.conversation.id"])
| where score < <threshold>
| project responseId, conversationId, evalName, score;
lowEvalResponses
| join kind=inner (
    dependencies
    | where timestamp > ago(7d)
    | where isnotempty(customDimensions["gen_ai.response.id"])
    | extend responseId = tostring(customDimensions["gen_ai.response.id"])
) on responseId
| extend
    operation = tostring(customDimensions["gen_ai.operation.name"]),
    model = tostring(customDimensions["gen_ai.request.model"]),
    inputTokens = toint(customDimensions["gen_ai.usage.input_tokens"]),
    outputTokens = toint(customDimensions["gen_ai.usage.output_tokens"])
| project timestamp, conversationId, responseId, evalName, score, operation, model, duration
| order by score asc
| take 100
```

> 💡 **Tip:** Replace `<threshold>` with the pass threshold from your evaluator config. Common values: `3.0` for 1–5 ordinal scales, `0.5` for 0–1 continuous scales.

### Latency Harvest — Slow Responses

Captures traces where response latency exceeds a threshold. Equivalent to LangSmith's `gt(latency, 5000)` run rule.

```kql
dependencies
| where timestamp > ago(7d)
| where duration > <threshold_ms>
| where isnotempty(customDimensions["gen_ai.operation.name"])
| where customDimensions["gen_ai.agent.name"] == "<agent-name>"
    or customDimensions["azure.ai.agentserver.agent_name"] == "<agent-name>"
| extend
    conversationId = tostring(customDimensions["gen_ai.conversation.id"]),
    responseId = tostring(customDimensions["gen_ai.response.id"]),
    operation = tostring(customDimensions["gen_ai.operation.name"]),
    model = tostring(customDimensions["gen_ai.request.model"]),
    inputTokens = toint(customDimensions["gen_ai.usage.input_tokens"]),
    outputTokens = toint(customDimensions["gen_ai.usage.output_tokens"])
| summarize
    avgDuration = avg(duration),
    maxDuration = max(duration),
    spanCount = count()
    by conversationId, responseId, operation, model
| order by maxDuration desc
| take 100
```

> 💡 **Tip:** Replace `<threshold_ms>` with the latency threshold in milliseconds. Common values: `5000` (5s), `10000` (10s), `30000` (30s).

### Combined Harvest — Multi-Criteria Filter

Combines multiple filters in a single query. Equivalent to LangSmith's compound rule: `and(gt(latency, 2000), eq(error, true), has(tags, "prod"))`.

```kql
dependencies
| where timestamp > ago(7d)
| where customDimensions["gen_ai.agent.name"] == "<agent-name>"
    or customDimensions["azure.ai.agentserver.agent_name"] == "<agent-name>"
| where isnotempty(customDimensions["gen_ai.operation.name"])
| where success == false or duration > <threshold_ms>
| extend
    conversationId = tostring(customDimensions["gen_ai.conversation.id"]),
    responseId = tostring(customDimensions["gen_ai.response.id"]),
    operation = tostring(customDimensions["gen_ai.operation.name"]),
    model = tostring(customDimensions["gen_ai.request.model"]),
    errorType = tostring(customDimensions["error.type"]),
    inputTokens = toint(customDimensions["gen_ai.usage.input_tokens"]),
    outputTokens = toint(customDimensions["gen_ai.usage.output_tokens"])
| summarize
    errorCount = countif(success == false),
    avgDuration = avg(duration),
    maxDuration = max(duration),
    spanCount = count()
    by conversationId, responseId, operation, model
| order by errorCount desc, maxDuration desc
| take 100
```

### Sampling — Control Dataset Size

Add `| sample <N>` or `| take <N>` to any harvest query to control the number of traces extracted. Equivalent to LangSmith's `sampling_rate` parameter.

```kql
// Random sample of 50 traces from the harvest
... | sample 50

// Top 50 most recent traces
... | order by timestamp desc | take 50

// Stratified sample: 20 errors + 20 slow + 10 low-eval
// Run each harvest separately and combine
```

## Step 2 — Schema Transform

Transform harvested traces into JSONL dataset format. Each line in the JSONL file must contain:

| Field | Required | Source |
|-------|----------|--------|
| `query` | ✅ | User input from the trace (reconstruct from `gen_ai.content.prompt` events) |
| `response` | Optional | Agent output from the trace |
| `context` | Optional | Tool results or retrieved documents from the trace |
| `ground_truth` | Optional | Expected correct answer (add during curation) |
| `metadata` | Optional | Source info: `{"source": "trace", "conversationId": "...", "harvestRule": "error"}` |

### Extracting Input/Output from Traces

After harvesting conversation-level summaries, drill into each conversation to extract actual input/output:

```kql
dependencies
| where customDimensions["gen_ai.conversation.id"] == "<conversation-id>"
| where customDimensions["gen_ai.operation.name"] in ("execute_agent", "chat", "create_response")
| extend
    responseId = tostring(customDimensions["gen_ai.response.id"]),
    operation = tostring(customDimensions["gen_ai.operation.name"])
| order by timestamp asc
| take 10
```

Then extract the prompt and completion events for each span. Save extracted data to a local JSONL file:

```
datasets/<agent-name>-traces-candidates-<date>.jsonl
```

## Step 3 — Human Review (Curation)

> ⚠️ **MANDATORY:** Never auto-commit harvested traces to a dataset. Always show candidates to the user first.

Present the harvested candidates as a table:

| # | Conversation ID | Error Type | Duration | Eval Score | Query (preview) |
|---|----------------|------------|----------|------------|----------------|
| 1 | conv-abc-123 | TimeoutError | 12.3s | 2.0 | "How do I reset my..." |
| 2 | conv-def-456 | None | 8.7s | 1.5 | "What's the status of..." |
| 3 | conv-ghi-789 | ValidationError | 0.4s | 3.0 | "Can you help me with..." |

Ask the user:
- *"Which candidates should I include in the dataset? (all / select by number / filter by criteria)"*
- *"Would you like to add ground_truth reference answers for any of these?"*
- *"What should I name this dataset version?"*

## Step 4 — Persist Dataset (Local JSONL)

Save approved candidates to `datasets/<agent-name>-<source>-v<N>.jsonl`:

```json
{"query": "How do I reset my password?", "context": "User account management", "metadata": {"source": "trace", "conversationId": "conv-abc-123", "harvestRule": "error"}}
{"query": "What's the status of my order?", "response": "...", "ground_truth": "Order #12345 shipped on...", "metadata": {"source": "trace", "conversationId": "conv-def-456", "harvestRule": "latency"}}
```

### Update Manifest

After persisting, update `datasets/manifest.json` with lineage information:

```json
{
  "datasets": [
    {
      "name": "support-bot-traces-v3",
      "file": "support-bot-traces-v3.jsonl",
      "version": "3",
      "source": "trace-harvest",
      "harvestRule": "error+latency",
      "timeRange": "2025-02-01 to 2025-02-07",
      "exampleCount": 47,
      "createdAt": "2025-02-08T10:00:00Z",
      "reviewedBy": "user"
    }
  ]
}
```

## Next Steps

After creating a dataset:
- **Run evaluation** → [observe skill Step 2](../../observe/references/evaluate-step.md)
- **Version and tag** → [Dataset Versioning](dataset-versioning.md)
- **Organize into splits** → [Dataset Organization](dataset-organization.md)
