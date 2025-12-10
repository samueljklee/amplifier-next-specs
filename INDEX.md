# Enterprise Productivity Extensions for Amplifier

> **Status**: Draft specifications for review
> **Purpose**: Comprehensive specs for tools, hooks, scenario tools, collections, and integrations targeting enterprise product teams working in large codebases.

---

## Overview

This specification set defines extensions to the Amplifier framework optimized for:

- **Product building** - Feature development, planning, release management
- **Team collaboration** - Code review, knowledge sharing, onboarding
- **Large codebase management** - Navigation, understanding, tech debt
- **Enterprise context** - Compliance, security, audit trails

---

## Architectural Note

**Why no custom orchestrators?**

After analysis, we determined that multi-step workflows and multi-agent collaboration belong at the **tool layer**, not the orchestrator layer. This follows Amplifier's "mechanism vs policy" philosophy:

- **Orchestrators** (kernel-adjacent): Simple execution mechanisms (loop-basic, loop-streaming)
- **Tools** (app layer): Workflow policies (recipes, collaborative, swarm)

The existing `amplifier-collection-recipes` already provides declarative multi-step workflows. New coordination patterns (`tool-collaborative`, `tool-swarm`) are specified as tools that spawn sub-agents.

---

## Specification Index

### Tools (Atomic Capabilities)

| Spec | Purpose | Priority |
|------|---------|----------|
| [tool-codebase-search](tools/tool-codebase-search.md) | Semantic search across code, docs, tickets | P0 |
| [tool-dependency-graph](tools/tool-dependency-graph.md) | Trace imports, call chains, impact analysis | P1 |
| [tool-architecture-map](tools/tool-architecture-map.md) | Generate/update architecture diagrams | P2 |
| [tool-tech-debt-scanner](tools/tool-tech-debt-scanner.md) | Identify anti-patterns, dead code, outdated deps | P1 |
| [tool-pr-context](tools/tool-pr-context.md) | Fetch PR, comments, linked issues, related PRs | P0 |
| [tool-slack-search](tools/tool-slack-search.md) | Search team discussions, decisions | P1 |
| [tool-jira-ops](tools/tool-jira-ops.md) | Create, update, link tickets | P0 |
| [tool-git-advanced](tools/tool-git-advanced.md) | Branch ops, history analysis, contextual blame | P1 |
| [tool-test-runner](tools/tool-test-runner.md) | Run specific tests, analyze failures | P1 |
| [tool-migration-helper](tools/tool-migration-helper.md) | Schema changes, migrations, rollback plans | P2 |
| [tool-feature-flag](tools/tool-feature-flag.md) | Check/toggle flags, analyze flag debt | P2 |
| [tool-metrics-query](tools/tool-metrics-query.md) | Query Prometheus/DataDog/etc. | P1 |

### Tools (Coordination Patterns)

These tools enable sophisticated multi-agent patterns while keeping orchestrators simple:

| Spec | Purpose | Priority |
|------|---------|----------|
| [tool-collaborative](tools/tool-collaborative.md) | Multi-agent collaboration (parallel specialists) | P1 |
| [tool-swarm](tools/tool-swarm.md) | Parallel exploration with convergence | P2 |

> **Note**: For sequential multi-step workflows, use the existing `amplifier-collection-recipes`.

### Hooks (Guardrails & Intelligence)

| Spec | Purpose | Priority |
|------|---------|----------|
| [hooks-style-enforcer](hooks/hooks-style-enforcer.md) | Auto-lint, inject feedback on violations | P1 |
| [hooks-security-scan](hooks/hooks-security-scan.md) | SAST scan, block critical vulnerabilities | P0 |
| [hooks-secret-detector](hooks/hooks-secret-detector.md) | Prevent committing secrets/keys | P0 |
| [hooks-breaking-change](hooks/hooks-breaking-change.md) | Detect API breaking changes | P1 |
| [hooks-ticket-linker](hooks/hooks-ticket-linker.md) | Auto-load JIRA context for current branch | P0 |
| [hooks-reviewer-suggester](hooks/hooks-reviewer-suggester.md) | Suggest reviewers based on code ownership | P2 |
| [hooks-standup-logger](hooks/hooks-standup-logger.md) | Summarize work for async standups | P2 |
| [hooks-knowledge-capture](hooks/hooks-knowledge-capture.md) | Extract learnings to team knowledge base | P1 |
| [hooks-audit-trail](hooks/hooks-audit-trail.md) | Immutable log for compliance | P0 |
| [hooks-pii-guardian](hooks/hooks-pii-guardian.md) | Redact PII before sending to LLM | P0 |
| [hooks-license-checker](hooks/hooks-license-checker.md) | Flag incompatible licenses | P2 |
| [hooks-approval-workflow](hooks/hooks-approval-workflow.md) | Require human approval for sensitive changes | P1 |
| [hooks-token-budget](hooks/hooks-token-budget.md) | Enforce per-session/user token limits | P1 |
| [hooks-model-router](hooks/hooks-model-router.md) | Route tasks to appropriate models | P1 |
| [hooks-usage-analytics](hooks/hooks-usage-analytics.md) | Track usage patterns for optimization | P1 |

### Recipes (Declarative Multi-Step Workflows)

Recipes are declarative YAML specifications that define multi-step AI agent workflows. They leverage `amplifier-collection-recipes` for:
- Sequential/conditional execution with `foreach` and `parallel: true`
- Recipe composition (sub-recipes for reusable audits)
- Context isolation and checkpointing
- Agent mode switching (ANALYZE, ARCHITECT, REVIEW)

#### Main Recipes

| Recipe | Purpose | Priority |
|--------|---------|----------|
| [pr-reviewer](recipes/pr-reviewer.yaml) | Comprehensive PR review with parallel analysis | P0 |
| [incident-responder](recipes/incident-responder.yaml) | Incident analysis with multi-hypothesis diagnosis | P0 |
| [feature-planner](recipes/feature-planner.yaml) | Transform PRD to implementation plan | P1 |
| [codebase-onboarder](recipes/codebase-onboarder.yaml) | New developer onboarding guide generation | P1 |
| [tech-debt-prioritizer](recipes/tech-debt-prioritizer.yaml) | Tech debt triage with JIRA integration | P1 |

#### Reusable Sub-Recipes (audits/)

Composable audit recipes that can be used standalone or within larger workflows:

| Recipe | Purpose | Used By |
|--------|---------|---------|
| [security-audit](recipes/audits/security-audit.yaml) | Security vulnerability analysis | pr-reviewer |
| [quality-audit](recipes/audits/quality-audit.yaml) | Code quality and maintainability | pr-reviewer |
| [breaking-change-audit](recipes/audits/breaking-change-audit.yaml) | API/schema compatibility analysis | pr-reviewer |
| [architecture-audit](recipes/audits/architecture-audit.yaml) | Architectural impact assessment | pr-reviewer (thorough mode) |

> **Migration Note**: The previous scenario-tools (Python implementations) have been converted to declarative YAML recipes. This follows Amplifier's "mechanism vs policy" philosophy - complex workflows are now declarative specifications that the recipe engine executes.

#### Extended Recipes (Showcasing Advanced Features)

Additional recipes demonstrating advanced Amplifier@next capabilities:

| Recipe | Purpose | Key Features | Priority |
|--------|---------|--------------|----------|
| [release-manager](recipes/release-manager.yaml) | Coordinate release process (changelog, notes, tagging) | `foreach` over commits, parallel notifications | P1 |
| [migration-orchestrator](recipes/migration-orchestrator.yaml) | Complex migrations with validation at each step | Checkpointing, rollback support, validation gates | P1 |
| [dependency-upgrader](recipes/dependency-upgrader.yaml) | Automated dep updates with compatibility checks | `foreach` + `parallel`, test integration, rollback | P1 |
| [root-cause-analyzer](recipes/root-cause-analyzer.yaml) | Deep dive into production issues | Multi-hypothesis diagnosis, evidence weighting | P1 |
| [test-suite-analyzer](recipes/test-suite-analyzer.yaml) | Coverage gaps, flaky tests, test prioritization | Parallel analysis, metrics aggregation | P1 |
| [api-evolution-tracker](recipes/api-evolution-tracker.yaml) | Track API changes, suggest versioning, generate migrations | Temporal analysis, doc generation | P2 |
| [documentation-auditor](recipes/documentation-auditor.yaml) | Find outdated docs, doc-code consistency | Multi-source correlation, quality scoring | P2 |
| [knowledge-harvester](recipes/knowledge-harvester.yaml) | Extract knowledge from Slack/PRs/meetings | Multi-source ingestion, entity extraction | P2 |
| [compliance-checker](recipes/compliance-checker.yaml) | SOC2/GDPR/HIPAA compliance scanning | Policy-based rules, parallel audits | P2 |
| [performance-profiler](recipes/performance-profiler.yaml) | Profile app, identify bottlenecks, track over time | Metrics integration, trend analysis | P2 |

**Feature Showcase:**

| Feature | Demonstrated By |
|---------|-----------------|
| `foreach` + `parallel` | dependency-upgrader, release-manager |
| Checkpointing/Resumability | migration-orchestrator |
| Multi-hypothesis Reasoning | root-cause-analyzer, incident-responder |
| Recipe Composition | pr-reviewer (uses audit sub-recipes) |
| Conditional Steps | all recipes with `condition:` |
| Mode Switching | feature-planner (ANALYZE → ARCHITECT → REVIEW) |
| Multi-source Ingestion | knowledge-harvester, root-cause-analyzer |
| Evidence Collection | compliance-checker, root-cause-analyzer |

### Collections (Packaged Expertise)

| Spec | Purpose | Priority |
|------|---------|----------|
| [collection-enterprise-dev](collections/collection-enterprise-dev.md) | Core enterprise development | P0 |
| [collection-platform-team](collections/collection-platform-team.md) | Platform/infrastructure teams | P1 |
| [collection-product-team](collections/collection-product-team.md) | Product management integration | P2 |

### Integrations (Enterprise Ecosystem)

#### Core Infrastructure

| Spec | Purpose | Priority |
|------|---------|----------|
| [integration-api-server](integrations/integration-api-server.md) | HTTP REST/GraphQL API server (enables all other integrations) | P0 |
| [integration-github-actions](integrations/integration-github-actions.md) | CI/CD integration | P0 |
| [integration-git-hooks](integrations/integration-git-hooks.md) | Pre-commit, commit-msg, pre-push automation | P1 |

#### Developer Tools

| Spec | Purpose | Priority |
|------|---------|----------|
| [integration-tauri-desktop](integrations/integration-tauri-desktop.md) | **Amplifier Desktop** - Native Tauri app with streaming UI | P0 |
| [integration-vscode](integrations/integration-vscode.md) | VS Code IDE integration | P1 |
| [integration-browser-extension](integrations/integration-browser-extension.md) | GitHub/GitLab PR review, code explanation in browser | P1 |
| [integration-electron-app](integrations/integration-electron-app.md) | Standalone desktop app with offline support | P2 |
| [integration-obsidian-plugin](integrations/integration-obsidian-plugin.md) | Knowledge management, code-to-docs | P2 |

#### Team Communication

| Spec | Purpose | Priority |
|------|---------|----------|
| [integration-slack](integrations/integration-slack.md) | Slack bot integration | P1 |
| [integration-teams-bot](integrations/integration-teams-bot.md) | Microsoft Teams bot with Azure DevOps | P2 |

#### Dashboards & Monitoring

| Spec | Purpose | Priority |
|------|---------|----------|
| [integration-web-dashboard](integrations/integration-web-dashboard.md) | Web-based management UI | P1 |
| [integration-observability](integrations/integration-observability.md) | Grafana/DataDog dashboards | P2 |
| [integration-mobile-companion](integrations/integration-mobile-companion.md) | iOS/Android mobile app | P2 |

---

## Priority Legend

- **P0**: Foundation - Build first, enables other work
- **P1**: High value - Significant productivity gain
- **P2**: Enhancement - Nice to have, can defer

---

## Specification Template

Each spec follows this structure:

```markdown
# [Component Name]

## Overview
Brief description and value proposition

## Contract
Interface definition (inputs, outputs, events)

## Architecture
Internal design, dependencies, data flow

## Configuration
Available options and defaults

## Examples
Usage examples and common patterns

## Security Considerations
Permissions, data handling, risks

## Testing Strategy
How to verify correctness

## Open Questions
Decisions needing input
```

---

## Review Process

1. Review specs in priority order (P0 → P1 → P2)
2. Mark decisions in "Open Questions" sections
3. Iterate until spec is approved
4. Implementation follows approved spec
