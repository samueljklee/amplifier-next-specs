# collection-platform-team

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-collection-platform-team`

## Overview

Specialized collection for platform engineering and infrastructure teams. Focuses on multi-service architecture, infrastructure-as-code, reliability engineering, and cross-cutting concerns.

### Value Proposition

| Without | With |
|---------|------|
| Service sprawl complexity | Unified multi-service tools |
| Manual dependency tracking | Automated service graph |
| Incident chaos | Structured response workflow |
| Config drift | Infrastructure validation |

---

## Collection Contents

```yaml
collection:
  name: platform-team
  version: "1.0.0"
  description: "Platform engineering and SRE toolkit"

  extends: enterprise-dev          # Inherits enterprise security

  # Platform-specific modules
  modules:

    # ===== TOOLS =====
    tools:
      # Multi-service operations
      - module: tool-service-graph
        config:
          discovery:
            kubernetes: true
            terraform: true
            consul: true
          include_dependencies: true
          include_health: true

      - module: tool-multi-repo-search
        config:
          repositories: "${PLATFORM_REPOS}"
          include_configs: true
          include_iac: true

      # Infrastructure
      - module: tool-terraform-ops
        config:
          workspaces: ["${TF_WORKSPACES}"]
          allowed_operations: [plan, validate, state]
          apply_require_approval: true

      - module: tool-kubernetes-ops
        config:
          contexts: ["${K8S_CONTEXTS}"]
          namespaces: ["${K8S_NAMESPACES}"]
          allowed_operations: [get, describe, logs, exec]
          blocked_operations: [delete, edit]

      - module: tool-helm-ops
        config:
          repositories: ["${HELM_REPOS}"]
          operations: [list, status, template, diff]
          install_require_approval: true

      # Observability
      - module: tool-metrics-query
        config:
          providers:
            prometheus:
              url: "${PROMETHEUS_URL}"
              queries: platform-queries.yaml
            datadog:
              api_key: "${DD_API_KEY}"
            cloudwatch:
              regions: ["${AWS_REGIONS}"]
          max_lookback: 30d            # Longer for capacity planning

      - module: tool-log-aggregator
        config:
          providers:
            elasticsearch:
              url: "${ES_URL}"
              indices: ["logs-*", "traces-*"]
            loki:
              url: "${LOKI_URL}"
          max_entries: 5000

      - module: tool-trace-analyzer
        config:
          provider: jaeger             # jaeger | zipkin | datadog
          url: "${JAEGER_URL}"
          include_dependencies: true

      # Dependencies and health
      - module: tool-dependency-graph
        config:
          scope: multi-repo
          include_infrastructure: true
          include_network: true

      - module: tool-health-checker
        config:
          services: "${PLATFORM_SERVICES}"
          checks: [http, tcp, grpc]
          include_dependencies: true

      # Database operations
      - module: tool-database-ops
        config:
          connections: "${DB_CONNECTIONS}"
          allowed_operations: [query, explain, schema]
          ddl_require_approval: true
          max_rows: 1000

    # ===== HOOKS =====
    hooks:
      # Platform-specific security
      - module: hooks-iac-validator
        config:
          validate_terraform: true
          validate_kubernetes: true
          validate_helm: true
          policies:
            - no_public_ingress
            - require_resource_limits
            - require_security_context

      - module: hooks-blast-radius
        config:
          warn_threshold: 10           # Services affected
          block_threshold: 50
          require_approval_above: 20

      # Change management
      - module: hooks-change-window
        config:
          windows:
            - name: maintenance
              cron: "0 2 * * 0"        # Sunday 2 AM
              duration: 4h
            - name: freeze
              dates: ["2024-12-20:2025-01-05"]
          production_changes: maintenance_only

      - module: hooks-rollback-ready
        config:
          require_rollback_plan: true
          verify_previous_version: true

      # Incident support
      - module: hooks-runbook-linker
        config:
          runbook_path: docs/runbooks/
          auto_suggest: true
          link_to_alerts: true

      - module: hooks-incident-auto-context
        config:
          include_recent_deploys: true
          include_related_alerts: true
          include_service_health: true
          lookback: 24h

      # Capacity and cost
      - module: hooks-resource-estimator
        config:
          estimate_cpu: true
          estimate_memory: true
          estimate_cost: true
          warn_high_cost: true

    # ===== SCENARIO TOOLS =====
    scenarios:
      - module: incident-responder
        config:
          auto_gather_context: true
          services_scope: platform

      - module: capacity-planner
        config:
          forecast_days: 90
          include_cost_projection: true

  # Platform-specific profiles
  profiles:
    # Infrastructure work
    infrastructure:
      tools:
        - tool-terraform-ops
        - tool-kubernetes-ops
        - tool-helm-ops
        - tool-service-graph
      hooks:
        - hooks-iac-validator
        - hooks-blast-radius
        - hooks-change-window
      config:
        require_approval_for:
          - "*.tf"
          - "*.yaml"           # K8s manifests
          - "values*.yaml"     # Helm values

    # Incident response
    incident:
      tools:
        - tool-metrics-query
        - tool-log-aggregator
        - tool-trace-analyzer
        - tool-service-graph
        - tool-health-checker
      hooks:
        - hooks-runbook-linker
        - hooks-incident-auto-context
      scenarios:
        - incident-responder
      config:
        elevated_permissions: true
        audit_level: verbose

    # Capacity planning
    capacity:
      tools:
        - tool-metrics-query
        - tool-service-graph
        - tool-dependency-graph
      hooks:
        - hooks-resource-estimator
      scenarios:
        - capacity-planner
      config:
        metrics_lookback: 90d

    # Service development
    service-dev:
      extends: ../enterprise-dev:development
      tools:
        - tool-service-graph
        - tool-multi-repo-search
        - tool-health-checker
      hooks:
        - hooks-blast-radius
```

---

## Service Graph Tool

A key differentiator for platform teams:

```yaml
tool-service-graph:
  description: "Map and analyze service dependencies"

  capabilities:
    # Discovery
    discover_from:
      - kubernetes_services
      - terraform_state
      - consul_catalog
      - manual_config

    # Analysis
    analyze:
      - dependency_chains
      - blast_radius
      - single_points_of_failure
      - cyclic_dependencies

    # Visualization
    outputs:
      - ascii_diagram
      - mermaid
      - graphviz
      - interactive_html

  # Example output
  example_output: |
    Service Dependency Graph for: payment-service

    payment-service
    ├── auth-service (direct)
    │   └── redis-cache (direct)
    ├── database (direct)
    │   └── postgres-primary (RDS)
    ├── notification-service (async)
    │   ├── sns-topic
    │   └── email-provider (external)
    └── stripe-api (external)

    Blast Radius: 12 services
    Single Points of Failure: postgres-primary
    Cyclic Dependencies: None
```

---

## Infrastructure Validation Hook

```yaml
hooks-iac-validator:
  description: "Validate infrastructure-as-code before apply"

  policies:
    # Kubernetes
    kubernetes:
      - name: require_resource_limits
        description: "All containers must have CPU/memory limits"
        severity: error

      - name: require_security_context
        description: "Pods must have security context"
        severity: error

      - name: no_privileged_containers
        description: "No privileged containers allowed"
        severity: error

      - name: require_liveness_probe
        description: "Services must have liveness probes"
        severity: warning

    # Terraform
    terraform:
      - name: no_public_s3
        description: "S3 buckets must not be public"
        severity: error

      - name: encryption_at_rest
        description: "Databases must have encryption"
        severity: error

      - name: require_tags
        description: "Resources must have cost allocation tags"
        severity: warning

    # Helm
    helm:
      - name: no_latest_tag
        description: "Image tags must be explicit versions"
        severity: error

  example_output: |
    🔍 IaC Validation Results

    ❌ FAILED: require_resource_limits
       File: deployment.yaml
       Line: 45
       Container 'app' missing resource limits

    ⚠️ WARNING: require_liveness_probe
       File: deployment.yaml
       Line: 42
       Container 'app' missing liveness probe

    ✅ PASSED: no_privileged_containers
    ✅ PASSED: require_security_context
```

---

## Blast Radius Analysis

```yaml
hooks-blast-radius:
  description: "Analyze impact of changes on dependent services"

  analysis:
    # Direct impacts
    direct:
      - services_calling_this
      - services_this_calls
      - shared_databases
      - shared_queues

    # Indirect impacts
    indirect:
      - cascading_dependencies
      - feature_flag_consumers
      - config_consumers

  thresholds:
    warning: 10                # Warn if >10 services affected
    approval: 20               # Require approval if >20
    block: 50                  # Block if >50

  example_output: |
    🎯 Blast Radius Analysis: payment-service changes

    Direct Impact (5 services):
    ├── checkout-service (calls payment API)
    ├── refund-service (calls payment API)
    ├── reporting-service (reads payment DB)
    ├── notification-service (listens to events)
    └── admin-dashboard (calls payment API)

    Indirect Impact (7 services):
    ├── mobile-app → checkout-service
    ├── web-app → checkout-service
    ├── customer-support → admin-dashboard
    └── ... (4 more)

    Total Blast Radius: 12 services
    Risk Level: MEDIUM

    ⚠️ Recommend: Deploy during maintenance window
```

---

## Incident Response Workflow

```yaml
incident-responder:
  # Platform-optimized configuration
  config:
    # Auto-gather platform context
    auto_context:
      recent_deployments:
        lookback: 24h
        all_services: true        # Not just affected service

      service_health:
        include_dependencies: true
        depth: 3                  # 3 levels of dependencies

      metrics:
        queries:
          - error_rate
          - latency_p99
          - saturation
          - traffic

      alerts:
        providers: [pagerduty, datadog]
        lookback: 4h

    # Platform-specific diagnosis
    diagnosis:
      approaches:
        - service_dependency_analysis
        - deployment_correlation
        - resource_exhaustion
        - cascading_failure

    # Remediation options
    remediation:
      actions:
        - type: rollback
          scope: single_service
          require_approval: true

        - type: scale
          scope: single_service
          limits:
            max_replicas: 10

        - type: circuit_breaker
          scope: dependency
          require_approval: false

        - type: traffic_shift
          scope: cluster
          require_approval: true
```

---

## Change Window Enforcement

```yaml
hooks-change-window:
  description: "Enforce change windows for production"

  windows:
    # Regular maintenance
    maintenance:
      schedule: "0 2 * * 0"       # Sunday 2 AM UTC
      duration: 4h
      allowed:
        - infrastructure
        - config
        - deployments

    # Feature releases
    release:
      schedule: "0 10 * * 2,4"   # Tue/Thu 10 AM UTC
      duration: 2h
      allowed:
        - deployments
      blocked:
        - database_migrations

    # Freeze periods
    freeze:
      dates:
        - start: "2024-11-28"
          end: "2024-12-02"
          reason: "Black Friday"
        - start: "2024-12-20"
          end: "2025-01-05"
          reason: "Holiday freeze"
      allowed:
        - hotfixes                # With P1 approval
      approval_required: p1_manager

  enforcement:
    production:
      outside_window: block
      message: |
        🚫 Change blocked: Outside maintenance window

        Next maintenance window: Sunday 2:00 AM UTC
        Or request emergency change approval from P1 manager

    staging:
      outside_window: warn
```

---

## Usage Examples

### Example 1: Infrastructure Change

```bash
amplifier --profile platform-team:infrastructure

> Review this Terraform change for the payment-service database

# Automatically:
# 1. Validates Terraform syntax
# 2. Checks IaC policies
# 3. Analyzes blast radius
# 4. Checks change window
# 5. Requires approval if significant
```

### Example 2: Incident Investigation

```bash
amplifier --profile platform-team:incident

> payment-service is returning 500 errors

# Automatically:
# 1. Gathers recent deployments (all services)
# 2. Pulls error logs from affected services
# 3. Checks service dependencies
# 4. Correlates with metrics
# 5. Suggests runbooks
# 6. Analyzes blast radius of potential fixes
```

### Example 3: Capacity Planning

```bash
amplifier --profile platform-team:capacity

> Forecast capacity needs for Q2

# Uses:
# - 90-day metrics history
# - Service dependency graph
# - Current resource utilization
# - Growth projections
# Produces:
# - Resource forecasts
# - Cost projections
# - Scaling recommendations
```

---

## Dependencies

### Required
- Kubernetes cluster access
- Terraform state backend
- Observability stack (Prometheus/Datadog)
- Log aggregation (ES/Loki)

### Optional
- Service mesh (Istio/Linkerd)
- Tracing (Jaeger/Zipkin)
- Consul service discovery

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
