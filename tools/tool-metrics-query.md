# tool-metrics-query

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-tool-metrics-query`

## Overview

Query observability platforms (Prometheus, DataDog, Grafana, CloudWatch) to get metrics, logs, and traces. Enables data-driven analysis during debugging, incident response, and performance optimization.

### Value Proposition

| Without | With |
|---------|------|
| Switch to Grafana, build query, interpret graphs | "What's the p99 latency for payment API?" → instant answer |
| Manually correlate metrics across dashboards | Single query spans metrics, logs, traces |
| Learn each platform's query language | Natural language queries translated automatically |

### Use Cases

1. **Performance analysis**: "What's the latency trend for /api/orders?"
2. **Incident investigation**: "Show error rates for the last hour"
3. **Capacity planning**: "What's the CPU utilization trend this week?"
4. **Debugging**: "Correlate this error with metrics and traces"
5. **Alerting context**: "What triggered the PagerDuty alert?"

---

## Contract

### Tool Definition

```python
TOOL_DEFINITION = {
    "name": "metrics_query",
    "description": """
    Query observability platforms for metrics, logs, and traces.

    Supports:
    - Prometheus/VictoriaMetrics (PromQL)
    - DataDog (metrics, logs, APM)
    - Grafana (as unified query layer)
    - CloudWatch (AWS metrics and logs)

    Operations:
    - query: Execute a metrics query
    - logs: Search logs
    - traces: Find distributed traces
    - alerts: Get active alerts

    Use for performance analysis, debugging, and incident investigation.
    """,
    "parameters": {
        "type": "object",
        "properties": {
            "operation": {
                "type": "string",
                "enum": ["query", "logs", "traces", "alerts"],
                "description": "Type of observability query"
            },
            "query": {
                "type": "string",
                "description": "Query in natural language or native syntax (PromQL, etc.)"
            },
            "service": {
                "type": "string",
                "description": "Service/application to query"
            },
            "time_range": {
                "type": "object",
                "properties": {
                    "start": {"type": "string"},
                    "end": {"type": "string"},
                    "relative": {"type": "string"}
                },
                "description": "Time range (e.g., {relative: '1h'} or {start: '...', end: '...'})"
            },
            "platform": {
                "type": "string",
                "enum": ["auto", "prometheus", "datadog", "grafana", "cloudwatch"],
                "default": "auto",
                "description": "Observability platform"
            },
            "aggregation": {
                "type": "string",
                "enum": ["avg", "sum", "min", "max", "p50", "p90", "p95", "p99"],
                "description": "Aggregation for metrics"
            }
        },
        "required": ["operation", "query"]
    }
}
```

### Output Schema

```python
@dataclass
class MetricsQueryOutput:
    operation: str
    platform: str
    native_query: str                   # Translated query in native syntax

    # For query operation
    metrics: MetricsResult | None = None

    # For logs operation
    logs: LogsResult | None = None

    # For traces operation
    traces: TracesResult | None = None

    # For alerts operation
    alerts: list[Alert] | None = None

@dataclass
class MetricsResult:
    series: list[TimeSeries]
    summary: MetricsSummary

@dataclass
class TimeSeries:
    labels: dict[str, str]              # {service: "api", endpoint: "/orders"}
    values: list[DataPoint]

@dataclass
class DataPoint:
    timestamp: datetime
    value: float

@dataclass
class MetricsSummary:
    min: float
    max: float
    avg: float
    current: float
    trend: str                          # increasing | decreasing | stable
    anomalies: list[Anomaly]

@dataclass
class LogsResult:
    entries: list[LogEntry]
    total_count: int
    patterns: list[LogPattern]          # Detected patterns/clusters

@dataclass
class LogEntry:
    timestamp: datetime
    level: str
    message: str
    service: str
    trace_id: str | None
    attributes: dict

@dataclass
class TracesResult:
    traces: list[Trace]
    service_map: ServiceMap | None

@dataclass
class Trace:
    trace_id: str
    root_span: Span
    duration_ms: float
    service_count: int
    error: bool

@dataclass
class Span:
    span_id: str
    service: str
    operation: str
    duration_ms: float
    status: str
    children: list[Span]

@dataclass
class Alert:
    id: str
    name: str
    status: str                         # firing | resolved
    severity: str
    message: str
    triggered_at: datetime
    labels: dict
```

---

## Architecture

### Query Translation

```python
class QueryTranslator:
    """Translate natural language to native query syntax."""

    async def translate(
        self,
        query: str,
        platform: str,
        context: QueryContext
    ) -> str:
        """
        Examples:
        - "p99 latency for payment-service"
          → histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{service="payment-service"}[5m]))

        - "error rate for /api/orders"
          → sum(rate(http_requests_total{endpoint="/api/orders",status=~"5.."}[5m]))
            / sum(rate(http_requests_total{endpoint="/api/orders"}[5m]))

        - "CPU usage trending"
          → avg(rate(container_cpu_usage_seconds_total{...}[5m])) by (pod)
        """

        # Detect if already native query
        if self._is_native_query(query, platform):
            return query

        # Use platform-specific translation
        return await self._translate_natural_language(query, platform, context)
```

### Platform Adapters

```python
class PrometheusAdapter:
    """Adapter for Prometheus/VictoriaMetrics."""

    async def query(self, promql: str, time_range: TimeRange) -> MetricsResult:
        # Query range for time series
        result = await self.client.query_range(
            query=promql,
            start=time_range.start,
            end=time_range.end,
            step=self._calculate_step(time_range)
        )

        series = self._parse_matrix(result)
        summary = self._compute_summary(series)

        return MetricsResult(series=series, summary=summary)


class DataDogAdapter:
    """Adapter for DataDog."""

    async def query_metrics(self, query: str, time_range: TimeRange) -> MetricsResult:
        result = await self.client.query_timeseries(
            query=query,
            start=int(time_range.start.timestamp()),
            end=int(time_range.end.timestamp())
        )
        return self._parse_result(result)

    async def search_logs(self, query: str, time_range: TimeRange) -> LogsResult:
        result = await self.client.logs_list(
            filter_query=query,
            filter_from=time_range.start.isoformat(),
            filter_to=time_range.end.isoformat()
        )
        return self._parse_logs(result)
```

---

## Configuration

```yaml
tool-metrics-query:
  # Platform configurations
  platforms:
    prometheus:
      enabled: true
      url: "${PROMETHEUS_URL}"
      auth:
        type: basic                     # basic | bearer | none
        username: "${PROMETHEUS_USER}"
        password: "${PROMETHEUS_PASS}"

    datadog:
      enabled: true
      api_key: "${DD_API_KEY}"
      app_key: "${DD_APP_KEY}"
      site: "datadoghq.com"             # or datadoghq.eu, etc.

    grafana:
      enabled: true
      url: "${GRAFANA_URL}"
      api_key: "${GRAFANA_API_KEY}"

    cloudwatch:
      enabled: true
      region: "us-east-1"
      # Uses AWS credentials from environment

  # Query settings
  query:
    default_time_range: "1h"
    max_data_points: 1000
    timeout_seconds: 30

  # Service discovery
  services:
    # Map friendly names to label selectors
    payment-service:
      prometheus: {service: "payment-api"}
      datadog: {service: "payment-api"}
    auth-service:
      prometheus: {service: "auth-api"}

  # Translation hints
  translation:
    # Common metric name mappings
    latency: "http_request_duration_seconds"
    errors: "http_requests_total{status=~'5..'}"
    requests: "http_requests_total"
```

---

## Examples

### Example 1: Latency Query

```python
# Input
{
    "operation": "query",
    "query": "p99 latency for payment-service",
    "time_range": {"relative": "1h"}
}

# Output
{
    "operation": "query",
    "platform": "prometheus",
    "native_query": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service=\"payment-api\"}[5m])) by (le))",
    "metrics": {
        "series": [
            {
                "labels": {"service": "payment-api"},
                "values": [
                    {"timestamp": "2024-01-15T10:00:00Z", "value": 0.245},
                    {"timestamp": "2024-01-15T10:05:00Z", "value": 0.251},
                    ...
                ]
            }
        ],
        "summary": {
            "min": 0.198,
            "max": 0.312,
            "avg": 0.247,
            "current": 0.251,
            "trend": "stable",
            "anomalies": []
        }
    }
}
```

### Example 2: Log Search with Correlation

```python
# Input
{
    "operation": "logs",
    "query": "error payment timeout",
    "service": "payment-service",
    "time_range": {"relative": "30m"}
}

# Output
{
    "operation": "logs",
    "platform": "datadog",
    "logs": {
        "entries": [
            {
                "timestamp": "2024-01-15T10:42:15Z",
                "level": "error",
                "message": "Payment gateway timeout after 30s",
                "service": "payment-api",
                "trace_id": "abc123",
                "attributes": {
                    "user_id": "u_456",
                    "payment_id": "pay_789",
                    "gateway": "stripe"
                }
            }
        ],
        "total_count": 23,
        "patterns": [
            {
                "pattern": "Payment gateway timeout after *",
                "count": 20,
                "first_seen": "2024-01-15T10:30:00Z"
            }
        ]
    }
}
```

### Example 3: Active Alerts

```python
# Input
{
    "operation": "alerts",
    "service": "payment-service"
}

# Output
{
    "operation": "alerts",
    "alerts": [
        {
            "id": "alert_123",
            "name": "PaymentServiceHighLatency",
            "status": "firing",
            "severity": "warning",
            "message": "p99 latency > 500ms for 5 minutes",
            "triggered_at": "2024-01-15T10:35:00Z",
            "labels": {"service": "payment-api", "endpoint": "/process"}
        }
    ]
}
```

---

## Security Considerations

- Accesses sensitive operational data
- API keys stored securely
- Query results may reveal infrastructure details

### Permissions

```python
REQUIRED_CAPABILITIES = [
    "network:prometheus",     # If Prometheus enabled
    "network:datadog",        # If DataDog enabled
    "network:grafana",        # If Grafana enabled
    "network:aws",            # If CloudWatch enabled
]
```

---

## Dependencies

### Required
- `httpx` - HTTP client

### Optional (per platform)
- `prometheus-api-client` - Prometheus
- `datadog-api-client` - DataDog
- `boto3` - CloudWatch

---

## Open Questions

1. **Query validation**: Should we validate queries before execution?
2. **Result caching**: Cache recent query results?
3. **Alerting integration**: Support creating/modifying alerts?
4. **Dashboard linking**: Link to relevant dashboards in results?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
