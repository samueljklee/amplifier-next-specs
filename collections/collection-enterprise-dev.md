# collection-enterprise-dev

> **Priority**: P0 (Critical Path)
> **Status**: Draft
> **Module**: `amplifier-collection-enterprise-dev`

## Overview

A comprehensive collection of tools, hooks, and configurations for enterprise software development teams. Provides security-first defaults, compliance features, and integrations with common enterprise tools.

### Value Proposition

| Without | With |
|---------|------|
| Manual module assembly | Pre-configured enterprise stack |
| Security afterthought | Security-first defaults |
| Compliance gaps | Built-in audit and compliance |
| Integration complexity | Ready-to-use enterprise integrations |

---

## Collection Contents

```yaml
collection:
  name: enterprise-dev
  version: "1.0.0"
  description: "Enterprise-grade AI development toolkit"

  # Included modules
  modules:

    # ===== PROVIDERS =====
    providers:
      - module: provider-anthropic
        config:
          default_model: claude-sonnet-4-5
          max_tokens: 8192

      - module: provider-azure-openai    # Enterprise Azure option
        config:
          api_version: "2024-02-15-preview"
          deployment_name: "${AZURE_OPENAI_DEPLOYMENT}"

    # ===== TOOLS =====
    tools:
      # Core development
      - module: tool-filesystem
        config:
          allowed_paths: ["${WORKSPACE}"]
          audit_all_writes: true

      - module: tool-bash
        config:
          allowed_commands: ["git", "npm", "python", "docker"]
          blocked_commands: ["rm -rf", "curl|sh"]
          timeout: 30s

      # Enterprise tools (from specs)
      - module: tool-codebase-search
        config:
          include_semantic: true
          include_docs: true
          include_tickets: true

      - module: tool-pr-context
        config:
          platform: github-enterprise
          include_reviews: true

      - module: tool-jira-ops
        config:
          server: "${JIRA_SERVER}"
          project_filter: ["${JIRA_PROJECTS}"]
          allowed_operations: [read, create, update, link]

      - module: tool-git-advanced
        config:
          enable_blame: true
          enable_bisect: false          # Potentially dangerous
          audit_commands: true

      - module: tool-test-runner
        config:
          frameworks: [pytest, jest, go-test]
          coverage_threshold: 80

      - module: tool-dependency-graph
        config:
          languages: [python, typescript, go]
          include_external: true

      - module: tool-slack-search
        config:
          channels: ["${ALLOWED_SLACK_CHANNELS}"]
          exclude_dm: true

      - module: tool-metrics-query
        config:
          providers: [datadog, prometheus]
          max_lookback: 7d

    # ===== HOOKS =====
    hooks:
      # Security (P0 - Always enabled)
      - module: hooks-secret-detector
        config:
          action: deny                   # Block secrets
          patterns: enterprise           # Enterprise patterns
          audit: true

      - module: hooks-security-scan
        config:
          scan_on_write: true
          severity_threshold: medium
          block_high: true

      - module: hooks-pii-guardian
        config:
          redact_patterns: [email, phone, ssn, credit_card]
          action: redact                 # Redact before sending to LLM

      - module: hooks-license-checker
        config:
          policy:
            allowed: [MIT, Apache-2.0, BSD-*]
            prohibited: [GPL-*, AGPL-*]
          action: deny

      # Compliance
      - module: hooks-audit-trail
        config:
          storage: "${AUDIT_LOG_PATH}"
          include_all_events: true
          retention_days: 365

      - module: hooks-approval-workflow
        config:
          require_approval_for:
            - "production/*"
            - "*.env*"
            - "secrets/*"
          escalation_channel: "#security-approvals"

      # Quality
      - module: hooks-style-enforcer
        config:
          auto_fix: true
          tools:
            python: [ruff, black, mypy]
            typescript: [eslint, prettier]

      - module: hooks-breaking-change
        config:
          check_api: true
          check_database: true
          action: warn

      # Productivity
      - module: hooks-ticket-linker
        config:
          platforms: [jira, github]
          auto_load_context: true

      - module: hooks-reviewer-suggester
        config:
          use_codeowners: true
          max_reviewers: 3

      # Observability
      - module: hooks-logging
        config:
          level: info
          format: json
          include_token_usage: true
          include_latency: true

      - module: hooks-usage-analytics
        config:
          track_costs: true
          track_tokens: true
          aggregation: [user, team, project]

    # ===== ORCHESTRATORS =====
    orchestrators:
      default: loop-streaming
      options:
        - loop-streaming
        - loop-basic

    # ===== CONTEXTS =====
    contexts:
      default: context-persistent
      config:
        storage: "${STATE_PATH}"
        max_history: 100

  # Collection-level configuration
  config:
    # Security defaults
    security:
      require_approval_for_production: true
      block_secrets: true
      audit_all_operations: true

    # Enterprise integrations
    integrations:
      sso:
        provider: okta
        required: true
      vault:
        enabled: true
        path: "secret/amplifier"

    # Defaults
    defaults:
      timeout: 120s
      max_tokens: 8192
      temperature: 0.3
```

---

## Profile Hierarchy

The collection provides layered profiles for different use cases:

```yaml
profiles:
  # Base enterprise profile (always applied)
  base:
    description: "Core enterprise security and compliance"
    hooks:
      - hooks-secret-detector
      - hooks-security-scan
      - hooks-audit-trail
      - hooks-pii-guardian
      - hooks-logging

  # Development profile (extends base)
  development:
    extends: base
    description: "Day-to-day development work"
    tools:
      - tool-filesystem
      - tool-bash
      - tool-codebase-search
      - tool-git-advanced
      - tool-test-runner
    hooks:
      - hooks-style-enforcer
      - hooks-ticket-linker

  # Review profile (extends base)
  review:
    extends: base
    description: "Code review and PR work"
    tools:
      - tool-pr-context
      - tool-codebase-search
      - tool-dependency-graph
    hooks:
      - hooks-breaking-change
      - hooks-reviewer-suggester
      - hooks-license-checker

  # Incident profile (extends base)
  incident:
    extends: base
    description: "Incident response"
    tools:
      - tool-metrics-query
      - tool-git-advanced
      - tool-slack-search
    hooks:
      - hooks-approval-workflow   # For production changes
    config:
      timeout: 300s               # Longer timeout for investigation

  # Full profile (all modules)
  full:
    extends: base
    description: "All enterprise modules"
    tools: all
    hooks: all
```

---

## Installation

```bash
# Install collection
amplifier collection install enterprise-dev

# Use specific profile
amplifier --profile enterprise-dev:development

# Or set as default
amplifier config set default-collection enterprise-dev
amplifier config set default-profile development
```

---

## Configuration Requirements

### Environment Variables

```bash
# Required
WORKSPACE=/path/to/project
AUDIT_LOG_PATH=/var/log/amplifier/audit
STATE_PATH=~/.amplifier/state

# Enterprise integrations
JIRA_SERVER=https://company.atlassian.net
JIRA_PROJECTS=PROJ,PLAT,INFRA
ALLOWED_SLACK_CHANNELS=engineering,platform,incidents

# Optional (for specific modules)
AZURE_OPENAI_ENDPOINT=https://company.openai.azure.com
AZURE_OPENAI_DEPLOYMENT=gpt-4
DD_API_KEY=xxx
DD_APP_KEY=xxx
```

### Secret Management

```yaml
# Vault integration for secrets
vault:
  enabled: true
  address: "${VAULT_ADDR}"
  auth_method: kubernetes       # or token, ldap, oidc
  secret_paths:
    anthropic: "secret/ai/anthropic"
    azure: "secret/ai/azure-openai"
    jira: "secret/integrations/jira"
    github: "secret/integrations/github"
```

---

## Security Posture

### Default Security Controls

| Control | Default | Configurable |
|---------|---------|--------------|
| Secret detection | Block | Yes (warn mode) |
| Security scanning | Block high severity | Yes |
| PII redaction | Redact before LLM | Yes |
| License checking | Block GPL | Yes |
| Audit logging | All events | Log level only |
| Production approval | Required | No (enterprise) |

### Compliance Features

- **SOC 2**: Audit trail, access controls, encryption
- **GDPR**: PII detection and redaction
- **HIPAA**: PHI patterns in PII guardian
- **PCI-DSS**: Credit card detection, audit logs

---

## Enterprise Integrations

### SSO Integration

```yaml
sso:
  provider: okta                # okta | azure-ad | onelogin
  client_id: "${OKTA_CLIENT_ID}"
  issuer: "${OKTA_ISSUER}"
  scopes: [openid, profile, groups]

  # Map groups to permissions
  group_mapping:
    "Engineering": ["development", "review"]
    "Security": ["development", "review", "incident"]
    "Platform": ["development", "review", "incident", "admin"]
```

### SIEM Integration

```yaml
siem:
  enabled: true
  provider: splunk              # splunk | elastic | datadog
  endpoint: "${SPLUNK_HEC_ENDPOINT}"

  # Events to forward
  events:
    - "audit:*"
    - "security:*"
    - "approval:*"

  # Format
  format: json
  include_context: false        # Don't send prompt content
```

---

## Usage Examples

### Example 1: Daily Development

```bash
# Start development session
amplifier --profile enterprise-dev:development

# Tools available:
# - Codebase search (semantic + docs + tickets)
# - Git operations (with audit)
# - Test runner
# - File operations (workspace only)

# Hooks active:
# - Secret detection (blocks)
# - Style enforcement (auto-fix)
# - Ticket linking (auto-context)
# - Security scanning (warns)
```

### Example 2: Code Review

```bash
# Review a PR
amplifier --profile enterprise-dev:review

> Review PR #1234

# Automatically:
# - Fetches PR context
# - Checks for breaking changes
# - Scans for security issues
# - Suggests reviewers
# - Checks license compliance
```

### Example 3: Incident Response

```bash
# Incident mode with elevated permissions
amplifier --profile enterprise-dev:incident

> Investigate payment failures

# Available:
# - Metrics queries (up to 7d lookback)
# - Git history analysis
# - Slack search for context
# - Production file access (with approval)
```

---

## Customization

### Extending the Collection

```yaml
# company-config.yaml
extends: enterprise-dev

# Add company-specific modules
modules:
  tools:
    - module: tool-internal-api
      config:
        endpoint: https://api.internal.company.com

  hooks:
    - module: hooks-company-policy
      config:
        policy_file: ./company-policy.yaml

# Override defaults
config:
  security:
    additional_secret_patterns:
      - name: internal_api_key
        pattern: "COMPANY_[A-Z0-9]{32}"

  integrations:
    ticketing:
      platform: internal        # Use internal system instead of Jira
```

### Disabling Modules

```yaml
# For specific use case, disable certain hooks
profiles:
  sandbox:
    extends: development
    disabled:
      - hooks-approval-workflow   # No approvals in sandbox
      - hooks-audit-trail         # No audit in sandbox
    config:
      security:
        require_approval_for_production: false
```

---

## Metrics & Reporting

### Built-in Dashboards

The collection provides data for:

- **Usage Dashboard**: Tokens, costs, sessions by team/user
- **Security Dashboard**: Blocked secrets, vulnerabilities found
- **Compliance Dashboard**: Approval rates, audit coverage
- **Productivity Dashboard**: Time saved, code generated

### Export Formats

```bash
# Export usage data
amplifier analytics export --format csv --period month

# Export audit logs
amplifier audit export --format json --days 30

# Generate compliance report
amplifier compliance report --standard soc2
```

---

## Dependencies

### Required External Services

- Git repository (GitHub Enterprise / GitLab / Azure DevOps)
- Ticketing system (Jira / Azure Boards)
- Secret management (Vault / AWS Secrets Manager)

### Optional Services

- Observability (Datadog / Prometheus / New Relic)
- Chat (Slack / Teams)
- SSO (Okta / Azure AD)

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
