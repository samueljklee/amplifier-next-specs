# hooks-model-router

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-hooks-model-router`

## Overview

Intelligent routing of LLM requests to appropriate models based on task complexity, cost optimization, latency requirements, and capabilities. Routes simple tasks to cheaper models while preserving quality for complex work.

### Value Proposition

| Without | With |
|---------|------|
| All tasks use same expensive model | Smart routing saves 50-70% on costs |
| Simple tasks wait for slow models | Fast models for simple tasks |
| One-size-fits-all | Task-appropriate model selection |

---

## Contract

### Hook Configuration

```yaml
hooks:
  - module: hooks-model-router
    config:
      # Available models with capabilities
      models:
        high_capability:
          provider: anthropic
          model: claude-3-opus
          capabilities: [reasoning, coding, analysis, creative]
          cost_per_1k: 0.015
          latency_ms: 2000

        balanced:
          provider: anthropic
          model: claude-3-sonnet
          capabilities: [reasoning, coding, analysis, creative]
          cost_per_1k: 0.003
          latency_ms: 1000

        fast:
          provider: anthropic
          model: claude-3-haiku
          capabilities: [simple, formatting, extraction]
          cost_per_1k: 0.00025
          latency_ms: 500

        code_specialized:
          provider: openai
          model: gpt-4-turbo
          capabilities: [coding, debugging]
          cost_per_1k: 0.01
          latency_ms: 1500

      # Routing rules
      routing:
        # Task-based routing
        tasks:
          simple_questions:
            patterns: ["what is", "define", "list", "when was"]
            route_to: fast

          code_generation:
            patterns: ["write.*function", "implement", "create.*class"]
            route_to: balanced

          complex_reasoning:
            patterns: ["analyze", "compare.*and.*contrast", "evaluate"]
            route_to: high_capability

          debugging:
            patterns: ["fix.*bug", "debug", "why.*error"]
            route_to: code_specialized

        # Token-based routing
        tokens:
          - max_input: 1000
            route_to: fast
          - max_input: 10000
            route_to: balanced
          - max_input: 100000
            route_to: high_capability

        # Override: always use specific model for certain contexts
        overrides:
          - context_contains: "IMPORTANT"
            route_to: high_capability
          - context_contains: "quick question"
            route_to: fast

      # Fallback behavior
      fallback:
        model: balanced
        on_error: retry_with_fallback

      # Logging
      log_routing_decisions: true
```

### HookResult

```python
# Route to different model
HookResult(
    action="modify",
    data={
        **original_data,
        "model": "claude-3-haiku",
        "provider_config": {"model": "claude-3-haiku-20240307"}
    },
    user_message="Routed to fast model (simple task)"
)
```

---

## Architecture

```python
class ModelRouterHook:
    """Route requests to appropriate models."""

    def __init__(self, config: ModelRouterConfig):
        self.config = config
        self.models = {name: ModelInfo(**info) for name, info in config.models.items()}
        self.classifier = TaskClassifier(config.routing.tasks)

    async def __call__(self, event: str, data: dict) -> HookResult:
        if event != "provider:request":
            return HookResult(action="continue")

        # Determine best model
        selected_model = await self._route_request(data)

        # If no change needed, continue
        if selected_model is None or selected_model == data.get("model"):
            return HookResult(action="continue")

        # Modify request to use selected model
        model_info = self.models[selected_model]
        return HookResult(
            action="modify",
            data={
                **data,
                "model": model_info.model,
                "provider": model_info.provider,
            },
            user_message=f"Routed to {selected_model} model"
        )

    async def _route_request(self, data: dict) -> str | None:
        """Determine which model to use."""
        messages = data.get("messages", [])

        # Check overrides first
        for override in self.config.routing.overrides:
            if self._matches_override(messages, override):
                return override["route_to"]

        # Classify task
        last_message = messages[-1].get("content", "") if messages else ""
        task_type = self.classifier.classify(last_message)

        if task_type:
            task_config = self.config.routing.tasks.get(task_type)
            if task_config:
                return task_config["route_to"]

        # Token-based routing
        estimated_tokens = self._estimate_tokens(messages)
        for rule in self.config.routing.tokens:
            if estimated_tokens <= rule["max_input"]:
                return rule["route_to"]

        # Fallback
        return self.config.fallback.model


class TaskClassifier:
    """Classify tasks for routing."""

    def __init__(self, task_configs: dict):
        self.patterns = {}
        for task_name, config in task_configs.items():
            self.patterns[task_name] = [
                re.compile(p, re.IGNORECASE)
                for p in config["patterns"]
            ]

    def classify(self, text: str) -> str | None:
        """Classify text into task type."""
        scores = {}

        for task_name, patterns in self.patterns.items():
            score = sum(1 for p in patterns if p.search(text))
            if score > 0:
                scores[task_name] = score

        if scores:
            return max(scores, key=scores.get)
        return None
```

---

## Routing Strategies

### 1. Task-Based Routing

| Task Type | Patterns | Model |
|-----------|----------|-------|
| Simple Q&A | "what is", "define", "when" | Haiku |
| Code generation | "write function", "implement" | Sonnet |
| Complex analysis | "analyze", "evaluate" | Opus |
| Debugging | "fix bug", "debug" | GPT-4 |

### 2. Token-Based Routing

| Input Size | Model | Rationale |
|------------|-------|-----------|
| < 1K tokens | Haiku | Simple, short tasks |
| 1K - 10K | Sonnet | Medium complexity |
| 10K+ | Opus | Long context needs capability |

### 3. Capability-Based Routing

```python
# Route based on required capabilities
CAPABILITY_REQUIREMENTS = {
    "mathematical_reasoning": ["high_capability"],
    "code_review": ["balanced", "code_specialized"],
    "simple_extraction": ["fast"],
    "creative_writing": ["high_capability", "balanced"],
}
```

---

## Examples

### Example 1: Simple Question → Fast Model

```python
# Input message: "What is the capital of France?"
# Classification: simple_questions
# Routing decision: fast (claude-3-haiku)

{
    "action": "modify",
    "data": {
        "model": "claude-3-haiku-20240307",
        "provider": "anthropic"
    },
    "user_message": "Routed to fast model (simple task)"
}
```

### Example 2: Complex Analysis → High Capability

```python
# Input message: "Analyze the trade-offs between microservices and monolithic architectures for our e-commerce platform"
# Classification: complex_reasoning
# Routing decision: high_capability (claude-3-opus)

{
    "action": "modify",
    "data": {
        "model": "claude-3-opus-20240229",
        "provider": "anthropic"
    },
    "user_message": "Routed to high-capability model (complex analysis)"
}
```

### Example 3: Override Takes Priority

```python
# Input message: "IMPORTANT: Quick question - what time is it in Tokyo?"
# Override matched: "IMPORTANT" → high_capability
# Despite being a simple question, override wins

{
    "action": "modify",
    "data": {
        "model": "claude-3-opus-20240229"
    },
    "user_message": "Routed to high-capability model (override: IMPORTANT)"
}
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `models.*` | dict | {} | Available models |
| `routing.tasks` | dict | {} | Task-based routing rules |
| `routing.tokens` | list | [] | Token-based routing rules |
| `routing.overrides` | list | [] | Override rules |
| `fallback.model` | string | - | Default model |
| `log_routing_decisions` | bool | true | Log decisions |

---

## Cost Impact

Typical cost savings with intelligent routing:

| Scenario | Without Routing | With Routing | Savings |
|----------|----------------|--------------|---------|
| 50% simple tasks | $100 | $35 | 65% |
| 30% code tasks | $100 | $55 | 45% |
| Mixed workload | $100 | $45 | 55% |

---

## Security Considerations

- Routing decisions may affect output quality
- Override rules should be carefully configured
- Sensitive tasks should not be routed to less capable models

---

## Dependencies

### Required
- `re` - Pattern matching (stdlib)

### Optional
- None

---

## Open Questions

1. **Learning**: Should routing learn from outcomes?
2. **User override**: Allow users to force specific models?
3. **Quality monitoring**: Track quality per routing decision?
4. **A/B testing**: Compare routing strategies?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
