# hooks-usage-analytics

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-hooks-usage-analytics`

## Overview

Collects usage metrics for analysis, optimization, and reporting. Tracks patterns like model usage, tool frequency, session duration, and task types to inform decisions about AI tooling investments.

### Value Proposition

| Without | With |
|---------|------|
| No visibility into AI usage | Comprehensive analytics |
| Can't justify AI investment | ROI data for stakeholders |
| Unknown optimization opportunities | Data-driven improvements |
| No usage patterns visible | Understand team behavior |

---

## Contract

### Hook Configuration

```yaml
hooks:
  - module: hooks-usage-analytics
    config:
      # What to track
      metrics:
        sessions:
          enabled: true
          track: [duration, turns, outcome]

        tokens:
          enabled: true
          track: [input, output, by_model, by_task]

        tools:
          enabled: true
          track: [frequency, duration, success_rate]

        tasks:
          enabled: true
          classify: true                # Auto-classify task types

        costs:
          enabled: true
          pricing: "${PRICING_CONFIG}"

      # Aggregation
      aggregation:
        intervals: [hourly, daily, weekly, monthly]
        dimensions: [user, team, project, model, task_type]

      # Storage
      storage:
        type: file                      # file | database | datadog | prometheus
        path: "${HOME}/.amplifier/analytics/"

      # Privacy
      privacy:
        anonymize_users: false
        exclude_content: true           # Don't store prompts/responses
        aggregate_only: false           # Only store aggregates, not raw events

      # Export
      export:
        enabled: true
        format: json                    # json | csv | parquet
        schedule: daily
```

### Events Captured

```python
# All events are captured with standard fields:
{
    "timestamp": "2024-01-15T10:30:00Z",
    "session_id": "sess-123",
    "user_id": "alice",                 # Optional, anonymizable
    "project": "payment-service",
    "event_type": "...",
    "data": {...}
}
```

---

## Architecture

```python
class UsageAnalyticsHook:
    """Collect and aggregate usage metrics."""

    CAPTURED_EVENTS = [
        "session:start",
        "session:end",
        "provider:request",
        "provider:response",
        "tool:pre",
        "tool:post",
        "tool:error",
        "prompt:submit",
        "prompt:complete",
    ]

    def __init__(self, config: UsageAnalyticsConfig):
        self.config = config
        self.storage = AnalyticsStorage(config.storage)
        self.aggregator = MetricsAggregator(config.aggregation)
        self.classifier = TaskClassifier() if config.metrics.tasks.classify else None

        # In-memory session tracking
        self.sessions: dict[str, SessionMetrics] = {}

    async def __call__(self, event: str, data: dict) -> HookResult:
        if event not in self.CAPTURED_EVENTS:
            return HookResult(action="continue")

        # Extract metrics
        metrics = self._extract_metrics(event, data)

        # Apply privacy settings
        metrics = self._apply_privacy(metrics)

        # Store raw event (if not aggregate_only)
        if not self.config.privacy.aggregate_only:
            await self.storage.store_event(metrics)

        # Update aggregates
        await self.aggregator.update(metrics)

        # Session-level tracking
        self._update_session(event, data, metrics)

        # Always continue - analytics is observational
        return HookResult(action="continue")

    def _extract_metrics(self, event: str, data: dict) -> dict:
        """Extract relevant metrics from event."""
        base = {
            "timestamp": datetime.utcnow().isoformat(),
            "session_id": data.get("session_id"),
            "user_id": data.get("user_id"),
            "event_type": event,
        }

        # Event-specific extraction
        if event == "provider:response":
            usage = data.get("usage", {})
            base.update({
                "model": data.get("model"),
                "provider": data.get("provider"),
                "input_tokens": usage.get("input_tokens", 0),
                "output_tokens": usage.get("output_tokens", 0),
                "latency_ms": data.get("latency_ms"),
            })

            # Calculate cost
            if self.config.metrics.costs.enabled:
                base["cost_usd"] = self._calculate_cost(
                    data.get("model"),
                    usage.get("input_tokens", 0),
                    usage.get("output_tokens", 0)
                )

        elif event == "tool:post":
            base.update({
                "tool": data.get("tool"),
                "operation": data.get("operation"),
                "duration_ms": data.get("duration_ms"),
                "success": data.get("success", True),
            })

        elif event == "prompt:submit":
            if self.classifier:
                base["task_type"] = self.classifier.classify(data.get("prompt", ""))

        return base

    def _update_session(self, event: str, data: dict, metrics: dict) -> None:
        """Update session-level metrics."""
        session_id = data.get("session_id")
        if not session_id:
            return

        if event == "session:start":
            self.sessions[session_id] = SessionMetrics(
                start_time=datetime.utcnow(),
                turns=0,
                total_tokens=0,
                total_cost=0.0,
                tools_used=set(),
                models_used=set()
            )

        elif session_id in self.sessions:
            session = self.sessions[session_id]

            if event == "prompt:complete":
                session.turns += 1

            if event == "provider:response":
                session.total_tokens += metrics.get("input_tokens", 0) + metrics.get("output_tokens", 0)
                session.total_cost += metrics.get("cost_usd", 0)
                if metrics.get("model"):
                    session.models_used.add(metrics["model"])

            if event == "tool:post":
                if metrics.get("tool"):
                    session.tools_used.add(metrics["tool"])

            if event == "session:end":
                session.end_time = datetime.utcnow()
                session.duration_seconds = (session.end_time - session.start_time).total_seconds()
                # Store session summary
                asyncio.create_task(self._store_session_summary(session_id, session))


class MetricsAggregator:
    """Aggregate metrics across dimensions."""

    async def update(self, metrics: dict) -> None:
        """Update all relevant aggregations."""
        timestamp = datetime.fromisoformat(metrics["timestamp"])

        for interval in self.config.intervals:
            bucket = self._get_bucket(timestamp, interval)

            for dimension in self.config.dimensions:
                key = self._get_dimension_key(metrics, dimension)
                if key:
                    await self._increment_counters(bucket, dimension, key, metrics)

    async def _increment_counters(
        self,
        bucket: str,
        dimension: str,
        key: str,
        metrics: dict
    ) -> None:
        """Increment counters for this bucket/dimension/key."""
        counter_key = f"{bucket}:{dimension}:{key}"

        # Token counters
        if "input_tokens" in metrics:
            await self.storage.increment(f"{counter_key}:input_tokens", metrics["input_tokens"])
            await self.storage.increment(f"{counter_key}:output_tokens", metrics["output_tokens"])

        # Request counters
        if metrics["event_type"] == "provider:response":
            await self.storage.increment(f"{counter_key}:requests", 1)

        # Cost counters
        if "cost_usd" in metrics:
            await self.storage.increment_float(f"{counter_key}:cost_usd", metrics["cost_usd"])
```

---

## Metrics Schema

### Session Metrics

```python
@dataclass
class SessionMetrics:
    session_id: str
    user_id: str | None
    project: str | None

    start_time: datetime
    end_time: datetime | None
    duration_seconds: float

    turns: int                          # User/assistant exchanges
    total_tokens: int
    total_cost: float

    models_used: set[str]
    tools_used: set[str]
    task_types: list[str]               # Classified task types

    outcome: str | None                 # success | abandoned | error
```

### Aggregated Metrics

```python
@dataclass
class AggregatedMetrics:
    period: str                         # "2024-01-15", "2024-W03", "2024-01"
    dimension: str                      # user | team | project | model
    dimension_value: str

    # Counts
    sessions: int
    requests: int
    tool_calls: int

    # Tokens
    input_tokens: int
    output_tokens: int
    total_tokens: int

    # Costs
    total_cost_usd: float
    avg_cost_per_session: float

    # Performance
    avg_latency_ms: float
    p95_latency_ms: float

    # Usage patterns
    top_tools: list[tuple[str, int]]    # [(tool, count), ...]
    top_models: list[tuple[str, int]]
    top_task_types: list[tuple[str, int]]
```

---

## Examples

### Example 1: Daily Summary

```json
{
    "period": "2024-01-15",
    "dimension": "team",
    "dimension_value": "platform",

    "sessions": 45,
    "requests": 892,
    "tool_calls": 234,

    "input_tokens": 1250000,
    "output_tokens": 450000,
    "total_tokens": 1700000,

    "total_cost_usd": 52.30,
    "avg_cost_per_session": 1.16,

    "avg_latency_ms": 1250,
    "p95_latency_ms": 3200,

    "top_tools": [
        ["filesystem:read", 89],
        ["codebase_search", 45],
        ["bash", 34]
    ],
    "top_models": [
        ["claude-3-sonnet", 650],
        ["claude-3-opus", 242]
    ],
    "top_task_types": [
        ["code_generation", 28],
        ["debugging", 12],
        ["analysis", 5]
    ]
}
```

### Example 2: User Leaderboard

```json
{
    "period": "2024-01",
    "dimension": "user",
    "leaderboard": [
        {"user": "alice", "sessions": 120, "cost": 245.50, "tokens": 5200000},
        {"user": "bob", "sessions": 95, "cost": 189.20, "tokens": 4100000},
        {"user": "carol", "sessions": 78, "cost": 156.80, "tokens": 3400000}
    ]
}
```

---

## Visualization / Export

### Dashboard Data Points

- **Time series**: Tokens, costs, sessions over time
- **Breakdowns**: By model, tool, task type, user
- **Trends**: Week-over-week changes
- **Alerts**: Unusual spikes in usage/cost

### Export Formats

```python
# CSV export
async def export_csv(period: str) -> str:
    """Export metrics as CSV."""
    data = await storage.get_period_metrics(period)
    return csv_format(data)

# JSON export for dashboards
async def export_json(period: str) -> dict:
    """Export metrics as JSON."""
    return await storage.get_period_metrics(period)

# Prometheus metrics
async def export_prometheus() -> str:
    """Export as Prometheus metrics."""
    metrics = await storage.get_current_metrics()
    return prometheus_format(metrics)
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `metrics.*.enabled` | bool | true | Enable metric category |
| `aggregation.intervals` | list | [daily] | Aggregation intervals |
| `aggregation.dimensions` | list | [user,project] | Aggregation dimensions |
| `storage.type` | string | "file" | Storage backend |
| `privacy.anonymize_users` | bool | false | Hash user IDs |
| `privacy.exclude_content` | bool | true | Don't store prompts |
| `privacy.aggregate_only` | bool | false | Only store aggregates |
| `export.schedule` | string | "daily" | Export schedule |

---

## Security Considerations

- User IDs can be anonymized
- Prompt content excluded by default
- Aggregates can be used instead of raw events
- Storage should be access-controlled

---

## Dependencies

### Required
- `aiofiles` - File operations

### Optional
- `prometheus_client` - Prometheus export
- `datadog` - DataDog integration
- `pandas` - Data analysis

---

## Open Questions

1. **Retention**: How long to keep raw events vs aggregates?
2. **Real-time**: Stream metrics to dashboards?
3. **Alerting**: Integrate with alerting systems?
4. **Benchmarking**: Compare teams/projects?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
