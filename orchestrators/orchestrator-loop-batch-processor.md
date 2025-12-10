# orchestrator-loop-batch-processor

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-loop-batch-processor`

## Overview

An orchestrator for non-interactive batch processing. No chat interface - users provide structured input (files, configs, data), the system processes them according to a defined pipeline, and returns structured output. Demonstrates Amplifier as a processing engine, not just a chatbot.

### Value Proposition

| Without | With |
|---------|------|
| Every interaction requires chat interface | Drop files, get processed results |
| User must describe what to do each time | Pre-configured pipelines execute automatically |
| One item at a time | Parallel processing of many items |
| Interactive waits | Background processing with notifications |

### Use Cases

1. **Code migration**: Process all files in a directory, apply transformation
2. **Documentation generation**: Generate docs from code across entire codebase
3. **Test generation**: Batch generate tests for all functions/classes
4. **Data transformation**: Convert formats, enrich data, extract patterns
5. **Report generation**: Process multiple inputs, produce consolidated report

### Modality Differentiator

This breaks the "chatbot" mental model by:
- **No conversation**: Input → Process → Output
- **No interactive prompts**: Pre-defined pipeline
- **Structured input/output**: Files, JSON, not natural language
- **Batch semantics**: Many items, one pipeline

---

## Contract

### Orchestrator Interface

```python
from dataclasses import dataclass
from typing import Any, Callable


@dataclass
class BatchInput:
    """Single item in a batch."""
    id: str
    data: Any
    metadata: dict[str, Any] | None = None


@dataclass
class BatchOutput:
    """Result for a single item."""
    id: str
    success: bool
    result: Any | None = None
    error: str | None = None
    metrics: dict[str, Any] | None = None


@dataclass
class BatchResult:
    """Complete batch processing result."""
    total: int
    succeeded: int
    failed: int
    outputs: list[BatchOutput]
    summary: str | None = None
    metrics: dict[str, Any] | None = None


class BatchProcessorOrchestrator:
    """
    Orchestrator for non-interactive batch processing.

    Pipeline:
    1. Receive batch input (files, data, configs)
    2. Validate and queue items
    3. Process items (parallel or sequential)
    4. Aggregate results
    5. Return structured output
    """

    async def execute(
        self,
        prompt: str,  # Unused in batch mode - pipeline is pre-configured
        context: Any,
        providers: dict[str, Any],
        tools: dict[str, Any],
        hooks: Any,
        coordinator: Any | None = None,
    ) -> str:
        """
        Execute batch processing.

        In batch mode, 'prompt' is typically a JSON-encoded BatchConfig
        or a path to a batch manifest file.

        Returns:
            JSON-encoded BatchResult
        """

    async def process_batch(
        self,
        inputs: list[BatchInput],
        pipeline: str,  # Pipeline name or inline definition
        config: dict[str, Any],
    ) -> BatchResult:
        """
        Process a batch of inputs through a pipeline.

        Args:
            inputs: Items to process
            pipeline: Which pipeline to run
            config: Pipeline configuration

        Returns:
            Aggregated batch results
        """
```

### Configuration Schema

```toml
[[orchestrators]]
module = "loop-batch-processor"
config = {
  # Processing control
  parallel = true                      # Process items in parallel
  max_concurrent = 5                   # Maximum concurrent items
  retry_failed = true                  # Retry failed items
  max_retries = 3                      # Maximum retry attempts

  # Input handling
  input_format = "files"               # "files" | "json" | "stdin" | "manifest"
  input_path = "./input/"              # Path for file-based input
  file_pattern = "**/*.py"             # Glob for file discovery

  # Pipeline definition
  pipeline = "transform"               # Named pipeline or inline
  pipelines = {
    transform = {
      steps = [
        { tool = "tool-filesystem", action = "read" },
        { prompt = "Transform this code: {content}" },
        { tool = "tool-filesystem", action = "write", path = "{output_path}" }
      ]
    },
    analyze = {
      steps = [
        { tool = "tool-filesystem", action = "read" },
        { prompt = "Analyze this: {content}" },
        { aggregate = "summary" }
      ]
    }
  }

  # Output handling
  output_format = "files"              # "files" | "json" | "stdout" | "report"
  output_path = "./output/"            # Path for file-based output

  # Aggregation
  aggregate_results = true             # Combine results at end
  aggregation_prompt = "Summarize these results: {results}"

  # Progress/notifications
  progress_hook = "hooks-batch-progress"
  notify_on_complete = true

  # Error handling
  fail_fast = false                    # Stop on first error
  error_output = "./errors/"           # Where to write errors
}
```

### Pipeline Definition

```yaml
# pipelines/code-migration.yaml
name: code-migration
description: Migrate code from pattern A to pattern B

input:
  type: files
  pattern: "**/*.py"

steps:
  - name: read
    tool: tool-filesystem
    action: read

  - name: analyze
    prompt: |
      Analyze this code for migration opportunities:
      - Old pattern: {config.old_pattern}
      - Look for: imports, function calls, class usage

      Code:
      {content}

    output: analysis

  - name: transform
    prompt: |
      Apply this migration to the code:
      - Old pattern: {config.old_pattern}
      - New pattern: {config.new_pattern}
      - Analysis: {analysis}

      Code:
      {content}

    output: transformed

  - name: write
    tool: tool-filesystem
    action: write
    content: "{transformed}"
    path: "{output_path}"

output:
  type: files
  preserve_structure: true

aggregation:
  prompt: |
    Summarize the migration:
    - Files processed: {total}
    - Files modified: {modified}
    - Changes made: {changes}
```

---

## Architecture

### Processing Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    BATCH INPUT                               │
│  Files │ JSON │ Stdin │ Manifest                            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    INPUT VALIDATOR                           │
│  Schema validation │ File existence │ Format checks         │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    WORK QUEUE                                │
│  Priority │ Dependencies │ Parallelization                  │
└────────────────────────┬────────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
     ┌─────────┐    ┌─────────┐    ┌─────────┐
     │ Worker  │    │ Worker  │    │ Worker  │
     │   1     │    │   2     │    │   3     │
     └────┬────┘    └────┬────┘    └────┬────┘
          │              │              │
          └──────────────┼──────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    RESULT AGGREGATOR                         │
│  Collect │ Summarize │ Report │ Metrics                     │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    BATCH OUTPUT                              │
│  Files │ JSON │ Stdout │ Report                             │
└─────────────────────────────────────────────────────────────┘
```

### Worker Execution

Each worker processes one item through the pipeline:

```python
async def process_item(
    item: BatchInput,
    pipeline: Pipeline,
    providers: dict,
    tools: dict,
) -> BatchOutput:
    """Process single item through pipeline."""

    context = {"input": item.data, "metadata": item.metadata}

    for step in pipeline.steps:
        if step.tool:
            # Tool execution
            result = await tools[step.tool].execute({
                "action": step.action,
                **interpolate(step.params, context)
            })
            context[step.output or "result"] = result.output

        elif step.prompt:
            # LLM call
            prompt = interpolate(step.prompt, context)
            response = await providers["default"].complete(
                ChatRequest(messages=[{"role": "user", "content": prompt}])
            )
            context[step.output or "result"] = response.content

    return BatchOutput(
        id=item.id,
        success=True,
        result=context.get("result"),
    )
```

---

## Events

```python
# Batch lifecycle events
"batch:start"       # Batch processing begins
"batch:item:start"  # Single item begins processing
"batch:item:end"    # Single item completes (success or error)
"batch:progress"    # Progress update (n of total)
"batch:end"         # Batch processing completes

# Hook integration
"batch:item:before" # Before processing - can filter/modify
"batch:item:after"  # After processing - can transform result
```

---

## Examples

### CLI Usage

```bash
# Process all Python files
amplifier batch \
  --pipeline code-migration \
  --input "./src/**/*.py" \
  --output "./migrated/" \
  --config '{"old_pattern": "requests", "new_pattern": "httpx"}'

# Process from manifest
amplifier batch --manifest ./batch-job.yaml

# Watch mode - process new files as they appear
amplifier batch --pipeline analyze --input "./uploads/" --watch
```

### Programmatic Usage

```python
from amplifier_core import AmplifierSession

session = AmplifierSession(profile="batch-processor")

result = await session.batch_process(
    inputs=[
        BatchInput(id="1", data={"file": "src/main.py"}),
        BatchInput(id="2", data={"file": "src/utils.py"}),
    ],
    pipeline="code-migration",
    config={"old_pattern": "print", "new_pattern": "logging"}
)

print(f"Processed {result.succeeded}/{result.total}")
```

### Pipeline as Amplifier Recipe

```yaml
# Batch processing can use recipes as pipelines
name: batch-test-generation
type: batch

input:
  type: files
  pattern: "src/**/*.py"
  exclude: ["*_test.py", "test_*.py"]

pipeline:
  recipe: recipes/generate-tests  # Reuse existing recipe

output:
  type: files
  path: "tests/"
  naming: "{basename}_test.py"
```

---

## Integration with Hooks

```python
# Progress tracking hook
@hook("batch:progress")
async def track_progress(event, data):
    """Report progress to external system."""
    await notify_slack(
        f"Batch {data['batch_id']}: {data['completed']}/{data['total']}"
    )

# Item filtering hook
@hook("batch:item:before")
async def filter_items(event, data):
    """Skip certain items."""
    if data["item"].metadata.get("skip"):
        return HookResult(action="skip")
    return HookResult(action="continue")

# Result transformation hook
@hook("batch:item:after")
async def transform_result(event, data):
    """Enrich results with additional data."""
    data["output"].metrics = calculate_metrics(data["output"].result)
    return HookResult(action="continue", data=data)
```

---

## Testing Strategy

```python
class TestBatchProcessor:
    async def test_processes_all_items(self):
        """All items in batch are processed."""

    async def test_parallel_processing(self):
        """Items process concurrently up to max_concurrent."""

    async def test_retry_on_failure(self):
        """Failed items are retried up to max_retries."""

    async def test_aggregation(self):
        """Results are aggregated correctly."""

    async def test_fail_fast_mode(self):
        """Processing stops on first error when fail_fast=true."""

    async def test_progress_events(self):
        """Progress hooks receive accurate counts."""
```

---

## Open Questions

1. **Checkpointing**: Should batch progress be checkpointed for resume on failure?
2. **Dependency ordering**: Should items be able to declare dependencies on other items?
3. **Dynamic batching**: Should pipeline be able to split/merge items mid-processing?
4. **Resource limits**: How to handle memory/API limits for large batches?
5. **Result streaming**: Should results be streamed as they complete or only at end?
