# integration-observability

> **Priority**: P2 (Medium Value)
> **Status**: Draft
> **Module**: `amplifier-integration-observability`

## Overview

Unified observability integration connecting Amplifier to monitoring platforms (Datadog, Prometheus, Grafana, New Relic, etc.). Enables AI-powered analysis of metrics, logs, and traces during incident response and proactive monitoring.

### Value Proposition

| Without | With |
|---------|------|
| Manual metric correlation | AI-driven anomaly detection |
| Siloed observability data | Unified query interface |
| Reactive incident response | Proactive issue identification |
| Context switching between tools | Single AI interface |

---

## Features

### 1. Unified Metrics Query

Query metrics across multiple platforms with natural language.

```python
# Natural language query
"Show me the error rate for payment-api in the last hour"

# Amplifier translates to appropriate platform queries:
# Datadog: avg:payment_api.error_rate{service:payment-api}.rollup(avg, 60)
# Prometheus: rate(payment_api_errors_total{service="payment-api"}[1h])
# New Relic: SELECT average(errorRate) FROM Transaction WHERE appName='payment-api'

# Returns unified response:
{
    "metric": "error_rate",
    "service": "payment-api",
    "time_range": "1h",
    "data_points": [...],
    "summary": {
        "current": 0.02,
        "average": 0.015,
        "trend": "increasing",
        "anomaly_detected": True
    }
}
```

### 2. Log Analysis

AI-powered log search and analysis.

```python
# Query logs across platforms
async with AmplifierSession(config=OBSERVABILITY_CONFIG) as session:
    result = await session.execute(
        prompt="Find errors related to database connections in the last 30 minutes",
        tools=["tool-log-query"]
    )

# Result:
"""
Found 47 database connection errors across 3 services:

**payment-api** (32 errors)
- Pattern: "Connection pool exhausted" at 14:23-14:28 UTC
- Affected endpoints: /api/payments, /api/refunds

**user-service** (12 errors)
- Pattern: "Connection timeout after 30s"
- Started after payment-api errors

**order-service** (3 errors)
- Pattern: "Unable to acquire connection"
- Isolated incidents, likely cascade effect

**Root Cause Analysis:**
The connection pool in payment-api reached capacity at 14:23 UTC.
This coincides with traffic spike (3x normal) from marketing campaign.

**Recommended Actions:**
1. Increase payment-api connection pool size
2. Add connection pool metrics alerting
3. Review query efficiency in /api/payments endpoint
"""
```

### 3. Trace Analysis

Distributed trace correlation and analysis.

```python
# Analyze slow traces
async with AmplifierSession(config=TRACE_CONFIG) as session:
    result = await session.execute(
        prompt="Why are checkout requests taking over 5 seconds?",
        tools=["tool-trace-query"]
    )

# Result with trace visualization:
"""
**Slow Checkout Analysis** (traces > 5s in last hour)

Sample trace: `trace-id-abc123`
Total duration: 7.2s

```
checkout-api    ████░░░░░░░░░░░░░░░░  1.2s (17%)
  └─ inventory  ░░░░████████████████  5.8s (81%) ← Bottleneck
      └─ db     ░░░░░░░░████████████  4.1s (57%)
  └─ payment    ░░░░░░░░░░░░░░░░░░██  0.2s (3%)
```

**Analysis:**
- Inventory service is the bottleneck (81% of total time)
- Database queries in inventory taking 4.1s average
- Query: `SELECT * FROM inventory WHERE sku IN (...)` - scanning 2M rows

**Recommendations:**
1. Add index on `inventory.sku` column
2. Implement inventory caching for frequently accessed SKUs
3. Consider pagination for bulk inventory checks
"""
```

### 4. Anomaly Detection

Proactive anomaly detection with AI analysis.

```python
# Configure anomaly detection
config = {
    "anomaly_detection": {
        "enabled": True,
        "metrics": [
            "error_rate",
            "latency_p99",
            "throughput",
            "cpu_usage",
            "memory_usage"
        ],
        "sensitivity": "medium",  # low | medium | high
        "notification_channel": "slack:#observability-alerts"
    }
}

# Automatic anomaly report:
"""
🔍 **Anomaly Detected** - payment-api

**Metric:** latency_p99
**Current:** 2.3s (normal: 0.4s)
**Started:** 14:15 UTC (25 minutes ago)
**Severity:** High

**Correlated Events:**
- 14:12 UTC: Deployment `payment-api@v2.4.1`
- 14:13 UTC: Config change in Redis cluster
- 14:14 UTC: Traffic spike +40%

**AI Analysis:**
The latency increase correlates strongly with the Redis config change.
The new configuration reduced connection pool from 100 to 20.

**Suggested Investigation:**
1. Check Redis connection metrics
2. Review config change in `redis-cluster-config@v1.2.3`
3. Consider rollback if latency doesn't recover

[View Dashboard](link) | [Start Investigation](link)
"""
```

### 5. Dashboard Generation

AI-generated dashboards based on service architecture.

```python
# Generate dashboard from service description
async with AmplifierSession(config=DASHBOARD_CONFIG) as session:
    result = await session.execute(
        prompt="Create a dashboard for our payment processing service",
        context={
            "service": "payment-api",
            "dependencies": ["postgres", "redis", "stripe-api"],
            "key_metrics": ["throughput", "error_rate", "latency"]
        }
    )

# Result: Dashboard configuration
{
    "title": "Payment API Dashboard",
    "panels": [
        {
            "title": "Request Throughput",
            "type": "timeseries",
            "query": "rate(payment_api_requests_total[5m])"
        },
        {
            "title": "Error Rate",
            "type": "gauge",
            "query": "rate(payment_api_errors_total[5m]) / rate(payment_api_requests_total[5m])",
            "thresholds": [0.01, 0.05]
        },
        {
            "title": "P99 Latency",
            "type": "timeseries",
            "query": "histogram_quantile(0.99, payment_api_latency_bucket)"
        },
        {
            "title": "Dependency Health",
            "type": "status",
            "targets": ["postgres", "redis", "stripe-api"]
        }
    ]
}
```

### 6. Incident Context Enrichment

Automatically enrich incident data with observability context.

```python
# When incident is triggered, gather observability context
async def enrich_incident_context(incident: dict) -> dict:
    """Gather comprehensive observability context for incident."""

    service = incident["service"]
    start_time = incident["started_at"]

    context = {}

    # Get metrics around incident start
    context["metrics"] = await query_metrics(
        service=service,
        start=start_time - timedelta(minutes=30),
        end=start_time + timedelta(minutes=10)
    )

    # Get error logs
    context["error_logs"] = await query_logs(
        service=service,
        level="error",
        start=start_time - timedelta(minutes=5),
        limit=100
    )

    # Get recent traces with errors
    context["error_traces"] = await query_traces(
        service=service,
        status="error",
        start=start_time - timedelta(minutes=5),
        limit=20
    )

    # Get recent deployments
    context["deployments"] = await query_deployments(
        service=service,
        start=start_time - timedelta(hours=2)
    )

    # Get correlated alerts
    context["alerts"] = await query_alerts(
        service=service,
        start=start_time - timedelta(minutes=10)
    )

    return context
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Observability Integration                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Unified Query Layer                       │   │
│  │  • Natural language → Platform-specific queries              │   │
│  │  • Result normalization and aggregation                      │   │
│  └───────────────────────────┬─────────────────────────────────┘   │
│                              │                                      │
│     ┌────────────────────────┼────────────────────────┐            │
│     ▼                        ▼                        ▼             │
│  ┌──────────┐          ┌──────────┐          ┌──────────┐          │
│  │ Datadog  │          │Prometheus│          │ Grafana  │          │
│  │ Adapter  │          │ Adapter  │          │ Adapter  │          │
│  └──────────┘          └──────────┘          └──────────┘          │
│                                                                      │
│  ┌──────────┐          ┌──────────┐          ┌──────────┐          │
│  │ New Relic│          │   Loki   │          │  Jaeger  │          │
│  │ Adapter  │          │ Adapter  │          │ Adapter  │          │
│  └──────────┘          └──────────┘          └──────────┘          │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Analysis Engine                           │   │
│  │  • Anomaly detection  • Correlation  • Root cause analysis   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Platform Adapters

### Datadog Adapter

```python
# adapters/datadog.py

from datadog_api_client import ApiClient, Configuration
from datadog_api_client.v1.api import metrics_api, logs_api

class DatadogAdapter:
    """Adapter for Datadog APIs."""

    def __init__(self, config: DatadogConfig):
        self.config = config
        self.api_config = Configuration()
        self.api_config.api_key["apiKeyAuth"] = config.api_key
        self.api_config.api_key["appKeyAuth"] = config.app_key

    async def query_metrics(
        self,
        query: str,
        start: datetime,
        end: datetime
    ) -> MetricsResult:
        """Query Datadog metrics."""

        with ApiClient(self.api_config) as client:
            api = metrics_api.MetricsApi(client)
            response = api.query_metrics(
                query=query,
                _from=int(start.timestamp()),
                to=int(end.timestamp())
            )

        return self._normalize_metrics(response)

    async def query_logs(
        self,
        query: str,
        start: datetime,
        end: datetime,
        limit: int = 100
    ) -> LogsResult:
        """Query Datadog logs."""

        with ApiClient(self.api_config) as client:
            api = logs_api.LogsApi(client)
            response = api.list_logs(
                body={
                    "filter": {
                        "query": query,
                        "from": start.isoformat(),
                        "to": end.isoformat()
                    },
                    "page": {"limit": limit}
                }
            )

        return self._normalize_logs(response)

    def translate_query(self, natural_query: str, context: dict) -> str:
        """Translate natural language to Datadog query."""

        # Use AI to translate
        # This would call Amplifier to do the translation
        pass

    def _normalize_metrics(self, response) -> MetricsResult:
        """Normalize Datadog metrics to unified format."""

        return MetricsResult(
            source="datadog",
            data_points=[
                DataPoint(
                    timestamp=datetime.fromtimestamp(p[0] / 1000),
                    value=p[1]
                )
                for series in response.series
                for p in series.pointlist
            ],
            metadata={
                "query": response.query,
                "unit": response.series[0].unit if response.series else None
            }
        )
```

### Prometheus Adapter

```python
# adapters/prometheus.py

import aiohttp

class PrometheusAdapter:
    """Adapter for Prometheus API."""

    def __init__(self, config: PrometheusConfig):
        self.config = config
        self.base_url = config.url

    async def query_metrics(
        self,
        query: str,
        start: datetime,
        end: datetime,
        step: str = "1m"
    ) -> MetricsResult:
        """Query Prometheus metrics."""

        async with aiohttp.ClientSession() as session:
            async with session.get(
                f"{self.base_url}/api/v1/query_range",
                params={
                    "query": query,
                    "start": start.timestamp(),
                    "end": end.timestamp(),
                    "step": step
                }
            ) as response:
                data = await response.json()

        return self._normalize_metrics(data)

    async def query_instant(self, query: str) -> MetricsResult:
        """Query current metric value."""

        async with aiohttp.ClientSession() as session:
            async with session.get(
                f"{self.base_url}/api/v1/query",
                params={"query": query}
            ) as response:
                data = await response.json()

        return self._normalize_instant(data)

    def translate_query(self, natural_query: str, context: dict) -> str:
        """Translate natural language to PromQL."""

        # Common translations
        translations = {
            "error_rate": 'rate({service}_errors_total{{service="{service}"}}[{window}])',
            "latency_p99": 'histogram_quantile(0.99, rate({service}_latency_bucket{{service="{service}"}}[{window}]))',
            "throughput": 'rate({service}_requests_total{{service="{service}"}}[{window}])',
        }

        # Would use AI for complex translations
        pass

    def _normalize_metrics(self, data: dict) -> MetricsResult:
        """Normalize Prometheus response to unified format."""

        result = data.get("data", {}).get("result", [])
        if not result:
            return MetricsResult(source="prometheus", data_points=[])

        return MetricsResult(
            source="prometheus",
            data_points=[
                DataPoint(
                    timestamp=datetime.fromtimestamp(float(p[0])),
                    value=float(p[1]),
                    labels=series.get("metric", {})
                )
                for series in result
                for p in series.get("values", [])
            ],
            metadata={
                "result_type": data.get("data", {}).get("resultType")
            }
        )
```

### Grafana Adapter

```python
# adapters/grafana.py

class GrafanaAdapter:
    """Adapter for Grafana API."""

    def __init__(self, config: GrafanaConfig):
        self.config = config
        self.base_url = config.url
        self.api_key = config.api_key

    async def query_dashboard(
        self,
        dashboard_uid: str,
        panel_id: int,
        start: datetime,
        end: datetime
    ) -> MetricsResult:
        """Query specific dashboard panel."""

        headers = {"Authorization": f"Bearer {self.api_key}"}

        async with aiohttp.ClientSession() as session:
            async with session.get(
                f"{self.base_url}/api/dashboards/uid/{dashboard_uid}",
                headers=headers
            ) as response:
                dashboard = await response.json()

        # Find panel and execute its queries
        panel = self._find_panel(dashboard, panel_id)
        return await self._execute_panel_queries(panel, start, end)

    async def create_dashboard(
        self,
        config: DashboardConfig
    ) -> dict:
        """Create a new Grafana dashboard."""

        headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json"
        }

        dashboard_json = self._build_dashboard(config)

        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.base_url}/api/dashboards/db",
                headers=headers,
                json={"dashboard": dashboard_json}
            ) as response:
                return await response.json()

    async def list_dashboards(
        self,
        folder_id: int | None = None,
        tag: str | None = None
    ) -> list[dict]:
        """List available dashboards."""

        headers = {"Authorization": f"Bearer {self.api_key}"}
        params = {}
        if folder_id:
            params["folderIds"] = folder_id
        if tag:
            params["tag"] = tag

        async with aiohttp.ClientSession() as session:
            async with session.get(
                f"{self.base_url}/api/search",
                headers=headers,
                params=params
            ) as response:
                return await response.json()
```

---

## Unified Query Interface

```python
# query/unified.py

class UnifiedObservabilityQuery:
    """Unified interface for querying observability data."""

    def __init__(self, adapters: dict[str, Adapter]):
        self.adapters = adapters

    async def query_metrics(
        self,
        natural_query: str,
        context: QueryContext
    ) -> UnifiedResult:
        """Query metrics using natural language."""

        # Parse natural query
        parsed = await self._parse_query(natural_query, context)

        # Execute on relevant platforms
        results = []
        for platform in parsed.target_platforms:
            adapter = self.adapters.get(platform)
            if adapter:
                platform_query = adapter.translate_query(
                    natural_query, context.to_dict()
                )
                result = await adapter.query_metrics(
                    platform_query,
                    parsed.start_time,
                    parsed.end_time
                )
                results.append(result)

        # Aggregate and normalize
        return self._aggregate_results(results)

    async def query_logs(
        self,
        natural_query: str,
        context: QueryContext
    ) -> UnifiedResult:
        """Query logs using natural language."""

        parsed = await self._parse_query(natural_query, context)

        results = []
        for platform in self._get_log_platforms():
            adapter = self.adapters.get(platform)
            if adapter and hasattr(adapter, "query_logs"):
                result = await adapter.query_logs(
                    parsed.translated_query,
                    parsed.start_time,
                    parsed.end_time
                )
                results.append(result)

        return self._aggregate_log_results(results)

    async def query_traces(
        self,
        trace_id: str | None = None,
        service: str | None = None,
        operation: str | None = None,
        start: datetime | None = None,
        end: datetime | None = None,
        status: str | None = None
    ) -> list[Trace]:
        """Query distributed traces."""

        results = []
        for platform in self._get_trace_platforms():
            adapter = self.adapters.get(platform)
            if adapter and hasattr(adapter, "query_traces"):
                result = await adapter.query_traces(
                    trace_id=trace_id,
                    service=service,
                    operation=operation,
                    start=start,
                    end=end,
                    status=status
                )
                results.extend(result)

        return self._deduplicate_traces(results)

    async def _parse_query(
        self,
        natural_query: str,
        context: QueryContext
    ) -> ParsedQuery:
        """Parse natural language query into structured form."""

        # Use AI to parse the query
        async with AmplifierSession(config=QUERY_PARSER_CONFIG) as session:
            result = await session.execute(
                prompt=f"""Parse this observability query:
                Query: {natural_query}
                Context: {context.to_dict()}

                Extract:
                - Target metrics/logs/traces
                - Time range
                - Filters (service, environment, etc.)
                - Aggregations
                """,
            )

        return ParsedQuery.from_ai_response(result)
```

---

## Analysis Engine

### Anomaly Detection

```python
# analysis/anomaly.py

class AnomalyDetector:
    """Detect anomalies in metrics data."""

    def __init__(self, config: AnomalyConfig):
        self.config = config
        self.baseline_window = config.baseline_window
        self.sensitivity = config.sensitivity

    async def detect(
        self,
        metric_data: MetricsResult,
        context: dict
    ) -> list[Anomaly]:
        """Detect anomalies in metric data."""

        anomalies = []

        # Statistical anomaly detection
        stats_anomalies = self._detect_statistical_anomalies(metric_data)
        anomalies.extend(stats_anomalies)

        # Pattern-based detection
        pattern_anomalies = self._detect_pattern_anomalies(metric_data)
        anomalies.extend(pattern_anomalies)

        # AI-enhanced detection for complex patterns
        ai_anomalies = await self._detect_ai_anomalies(metric_data, context)
        anomalies.extend(ai_anomalies)

        # Deduplicate and rank
        return self._rank_anomalies(anomalies)

    def _detect_statistical_anomalies(
        self,
        data: MetricsResult
    ) -> list[Anomaly]:
        """Detect anomalies using statistical methods."""

        values = [p.value for p in data.data_points]
        if len(values) < 10:
            return []

        mean = statistics.mean(values)
        stdev = statistics.stdev(values)

        threshold_multiplier = {
            "low": 3,
            "medium": 2.5,
            "high": 2
        }[self.sensitivity]

        threshold = mean + (stdev * threshold_multiplier)

        anomalies = []
        for point in data.data_points:
            if point.value > threshold:
                anomalies.append(Anomaly(
                    timestamp=point.timestamp,
                    metric=data.metadata.get("metric_name"),
                    value=point.value,
                    expected_range=(mean - stdev, mean + stdev),
                    severity=self._calculate_severity(point.value, mean, stdev),
                    detection_method="statistical"
                ))

        return anomalies

    async def _detect_ai_anomalies(
        self,
        data: MetricsResult,
        context: dict
    ) -> list[Anomaly]:
        """Use AI to detect complex anomaly patterns."""

        async with AmplifierSession(config=ANOMALY_CONFIG) as session:
            result = await session.execute(
                prompt=f"""Analyze this metric data for anomalies:

                Data: {data.to_summary()}
                Context: {context}

                Look for:
                - Sudden spikes or drops
                - Trend changes
                - Seasonal pattern violations
                - Correlation anomalies with other metrics

                Return structured anomaly data.
                """,
            )

        return self._parse_ai_anomalies(result)


class CorrelationAnalyzer:
    """Analyze correlations between metrics and events."""

    async def correlate_with_deployments(
        self,
        anomaly: Anomaly,
        service: str
    ) -> list[Correlation]:
        """Find deployments that correlate with anomaly."""

        # Get deployments in time window
        deployments = await self.deployment_adapter.get_deployments(
            service=service,
            start=anomaly.timestamp - timedelta(hours=2),
            end=anomaly.timestamp
        )

        correlations = []
        for deployment in deployments:
            time_diff = (anomaly.timestamp - deployment.timestamp).total_seconds()
            if 0 < time_diff < 3600:  # Within 1 hour after deployment
                correlations.append(Correlation(
                    event_type="deployment",
                    event=deployment,
                    time_offset=time_diff,
                    confidence=self._calculate_confidence(time_diff)
                ))

        return correlations

    async def correlate_with_config_changes(
        self,
        anomaly: Anomaly,
        service: str
    ) -> list[Correlation]:
        """Find config changes that correlate with anomaly."""

        config_changes = await self.config_adapter.get_changes(
            service=service,
            start=anomaly.timestamp - timedelta(hours=1),
            end=anomaly.timestamp
        )

        correlations = []
        for change in config_changes:
            time_diff = (anomaly.timestamp - change.timestamp).total_seconds()
            if 0 < time_diff < 1800:  # Within 30 minutes
                correlations.append(Correlation(
                    event_type="config_change",
                    event=change,
                    time_offset=time_diff,
                    confidence=self._calculate_confidence(time_diff)
                ))

        return correlations
```

### Root Cause Analysis

```python
# analysis/root_cause.py

class RootCauseAnalyzer:
    """Analyze root cause of incidents using observability data."""

    async def analyze(
        self,
        incident: dict,
        context: ObservabilityContext
    ) -> RootCauseReport:
        """Perform root cause analysis."""

        # Gather all relevant data
        metrics = context.metrics
        logs = context.error_logs
        traces = context.error_traces
        deployments = context.deployments
        config_changes = context.config_changes

        # Use AI for root cause analysis
        async with AmplifierSession(config=RCA_CONFIG) as session:
            result = await session.execute(
                prompt=f"""Analyze this incident and determine root cause:

                Incident: {incident}

                Metrics (around incident time):
                {self._summarize_metrics(metrics)}

                Error Logs:
                {self._summarize_logs(logs)}

                Error Traces:
                {self._summarize_traces(traces)}

                Recent Deployments:
                {self._summarize_deployments(deployments)}

                Recent Config Changes:
                {self._summarize_config_changes(config_changes)}

                Provide:
                1. Most likely root cause
                2. Evidence supporting this conclusion
                3. Alternative hypotheses
                4. Recommended remediation steps
                5. Preventive measures for the future
                """,
            )

        return self._parse_rca_result(result)

    def _summarize_metrics(self, metrics: dict) -> str:
        """Summarize metrics for AI analysis."""

        summaries = []
        for metric_name, data in metrics.items():
            trend = self._calculate_trend(data)
            anomalies = self._find_anomalies(data)
            summaries.append(
                f"{metric_name}: trend={trend}, anomalies={len(anomalies)}"
            )
        return "\n".join(summaries)
```

---

## Configuration

```yaml
# .amplifier/integrations/observability.yaml
observability:
  enabled: true

  # Platform configurations
  platforms:
    datadog:
      enabled: true
      api_key: ${DATADOG_API_KEY}
      app_key: ${DATADOG_APP_KEY}
      site: "datadoghq.com"  # or datadoghq.eu

    prometheus:
      enabled: true
      url: "http://prometheus.monitoring.svc:9090"
      # Optional authentication
      auth:
        type: bearer
        token: ${PROMETHEUS_TOKEN}

    grafana:
      enabled: true
      url: "https://grafana.example.com"
      api_key: ${GRAFANA_API_KEY}

    loki:
      enabled: true
      url: "http://loki.monitoring.svc:3100"

    jaeger:
      enabled: true
      url: "http://jaeger.monitoring.svc:16686"

  # Anomaly detection configuration
  anomaly_detection:
    enabled: true
    sensitivity: medium  # low | medium | high
    baseline_window: 7d  # Historical data for baseline
    metrics:
      - name: "error_rate"
        threshold: 0.05  # 5% error rate
        alert_channel: "slack:#alerts"
      - name: "latency_p99"
        threshold: 2.0   # 2 seconds
        alert_channel: "slack:#alerts"
      - name: "throughput"
        change_threshold: 50  # 50% change
        alert_channel: "slack:#alerts"

  # Dashboard generation
  dashboards:
    auto_generate: true
    output_platform: grafana
    folder: "Amplifier Generated"
    refresh_interval: "30s"

  # Query settings
  query:
    default_time_range: 1h
    max_time_range: 7d
    max_data_points: 1000
    cache_ttl: 60s

  # Correlation settings
  correlation:
    deployment_window: 2h      # Look back 2 hours for deployments
    config_change_window: 1h   # Look back 1 hour for config changes
    alert_window: 30m          # Look for related alerts

  # Export settings
  export:
    enabled: true
    formats:
      - json
      - csv
    retention: 30d
```

---

## Tools

### tool-metrics-query

```yaml
# tools/tool-metrics-query.yaml
name: tool-metrics-query
description: Query metrics from observability platforms

parameters:
  query:
    type: string
    description: Natural language query or platform-specific query
    required: true
  platform:
    type: string
    enum: [datadog, prometheus, grafana, auto]
    default: auto
  start_time:
    type: string
    description: Start time (ISO8601 or relative like "1h ago")
    default: "1h ago"
  end_time:
    type: string
    description: End time
    default: "now"
  aggregation:
    type: string
    enum: [avg, sum, min, max, count]
    default: avg

returns:
  type: object
  properties:
    data_points:
      type: array
      description: Time series data
    summary:
      type: object
      description: Statistical summary
    anomalies:
      type: array
      description: Detected anomalies
```

### tool-log-query

```yaml
# tools/tool-log-query.yaml
name: tool-log-query
description: Query logs from observability platforms

parameters:
  query:
    type: string
    description: Natural language or platform-specific log query
    required: true
  platform:
    type: string
    enum: [datadog, loki, cloudwatch, auto]
    default: auto
  start_time:
    type: string
    default: "30m ago"
  end_time:
    type: string
    default: "now"
  limit:
    type: integer
    default: 100
  level:
    type: string
    enum: [debug, info, warn, error, fatal, all]
    default: all

returns:
  type: object
  properties:
    logs:
      type: array
      description: Log entries
    patterns:
      type: array
      description: Detected patterns
    summary:
      type: object
      description: Log analysis summary
```

### tool-trace-query

```yaml
# tools/tool-trace-query.yaml
name: tool-trace-query
description: Query distributed traces

parameters:
  trace_id:
    type: string
    description: Specific trace ID to retrieve
  service:
    type: string
    description: Filter by service name
  operation:
    type: string
    description: Filter by operation name
  status:
    type: string
    enum: [ok, error, all]
    default: all
  min_duration:
    type: string
    description: Minimum trace duration (e.g., "1s", "500ms")
  start_time:
    type: string
    default: "1h ago"
  end_time:
    type: string
    default: "now"
  limit:
    type: integer
    default: 20

returns:
  type: object
  properties:
    traces:
      type: array
      description: Trace data with spans
    analysis:
      type: object
      description: Trace analysis (bottlenecks, patterns)
```

---

## Security

### Credentials Management

```yaml
# Secrets should be managed via environment variables or secret manager
observability:
  platforms:
    datadog:
      api_key: ${DATADOG_API_KEY}       # From environment
      app_key: ${DATADOG_APP_KEY}

    prometheus:
      auth:
        type: bearer
        token_secret: vault://secret/prometheus/token  # From Vault
```

### Access Control

```python
# security/access.py

class ObservabilityAccessControl:
    """Control access to observability data."""

    def __init__(self, config: AccessConfig):
        self.config = config

    async def can_query(
        self,
        user: str,
        query_type: str,
        scope: dict
    ) -> bool:
        """Check if user can execute query."""

        # Get user's allowed scopes
        user_scopes = await self.get_user_scopes(user)

        # Check service access
        if scope.get("service"):
            if scope["service"] not in user_scopes.get("services", []):
                return False

        # Check environment access
        if scope.get("environment"):
            if scope["environment"] not in user_scopes.get("environments", []):
                return False

        # Check query type permissions
        allowed_queries = user_scopes.get("query_types", ["metrics", "logs"])
        if query_type not in allowed_queries:
            return False

        return True

    async def redact_sensitive_data(
        self,
        data: dict,
        user: str
    ) -> dict:
        """Redact sensitive data based on user permissions."""

        # Redact PII from logs if user doesn't have PII access
        if not await self.has_pii_access(user):
            data = self._redact_pii(data)

        # Redact secrets
        data = self._redact_secrets(data)

        return data
```

---

## Events

| Event | Description | Data |
|-------|-------------|------|
| `observability:query:start` | Query started | query, platforms |
| `observability:query:complete` | Query completed | query, results, duration |
| `observability:anomaly:detected` | Anomaly detected | metric, value, severity |
| `observability:correlation:found` | Correlation found | anomaly, correlated_event |
| `observability:rca:complete` | Root cause analysis complete | incident, root_cause |
| `observability:dashboard:created` | Dashboard generated | dashboard_id, platform |

---

## Dependencies

### Required
- Platform-specific API clients (datadog-api-client, prometheus-api-client, etc.)
- aiohttp for async HTTP requests
- Statistics libraries for anomaly detection

### Optional
- Machine learning libraries for advanced anomaly detection
- Visualization libraries for dashboard generation

---

## Open Questions

1. **Data retention**: How long to cache observability data locally?
2. **Cost management**: Rate limiting for expensive platform queries?
3. **Multi-tenancy**: Support multiple observability stacks per environment?
4. **Alerting integration**: Integrate with PagerDuty/OpsGenie for anomaly alerts?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
