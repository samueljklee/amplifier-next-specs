# collection-product-team

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-collection-product-team`

## Overview

Optimized collection for product development teams building user-facing features. Focuses on feature development workflow, product quality, user feedback integration, and rapid iteration.

### Value Proposition

| Without | With |
|---------|------|
| Context scattered across tools | Unified product context |
| Feature specs lost in docs | Living specifications |
| Manual acceptance testing | Automated validation |
| Slow feedback loops | Rapid iteration support |

---

## Collection Contents

```yaml
collection:
  name: product-team
  version: "1.0.0"
  description: "Product development toolkit"

  extends: enterprise-dev          # Inherits enterprise security

  # Product-specific modules
  modules:

    # ===== TOOLS =====
    tools:
      # Product context
      - module: tool-product-context
        config:
          sources:
            jira:
              project: "${JIRA_PROJECT}"
              issue_types: [Story, Bug, Epic]
            figma:
              team_id: "${FIGMA_TEAM}"
              auto_fetch_designs: true
            confluence:
              space: "${CONFLUENCE_SPACE}"
              labels: [prd, spec, design]
            analytics:
              provider: amplitude
              project: "${AMPLITUDE_PROJECT}"

      - module: tool-user-feedback
        config:
          sources:
            zendesk:
              enabled: true
              views: ["${ZENDESK_VIEWS}"]
            intercom:
              enabled: true
            app_reviews:
              ios: "${APP_STORE_ID}"
              android: "${PLAY_STORE_ID}"
          sentiment_analysis: true
          auto_categorize: true

      - module: tool-feature-flag
        config:
          provider: launchdarkly     # launchdarkly | split | flagsmith
          project: "${LD_PROJECT}"
          environments: [development, staging, production]
          allowed_operations: [read, create, toggle]
          rollout_require_approval: true

      - module: tool-analytics-query
        config:
          providers:
            amplitude:
              api_key: "${AMPLITUDE_API_KEY}"
              project: "${AMPLITUDE_PROJECT}"
            mixpanel:
              token: "${MIXPANEL_TOKEN}"
          max_lookback: 90d

      - module: tool-a11y-checker
        config:
          standards: [WCAG2.1-AA]
          check_on_preview: true

      # Core development (from enterprise-dev)
      - module: tool-codebase-search
        config:
          include_figma: true
          include_specs: true

      - module: tool-test-runner
        config:
          frameworks: [jest, playwright, cypress]
          include_e2e: true
          visual_regression: true

      - module: tool-pr-context
        config:
          include_linked_issues: true
          include_figma_links: true
          include_analytics_impact: true

    # ===== HOOKS =====
    hooks:
      # Product quality
      - module: hooks-acceptance-validator
        config:
          source: jira                # Read AC from JIRA stories
          auto_check: true
          link_to_tests: true

      - module: hooks-design-diff
        config:
          figma_integration: true
          warn_on_deviation: true
          tolerance: 5px

      - module: hooks-i18n-checker
        config:
          check_hardcoded_strings: true
          check_missing_translations: true
          default_locale: en
          required_locales: [en, es, fr, de, ja]

      - module: hooks-a11y-validator
        config:
          check_on_component_write: true
          standards: [WCAG2.1-AA]
          severity: warning

      # Feature development
      - module: hooks-feature-context
        config:
          auto_load:
            - linked_story
            - acceptance_criteria
            - design_specs
            - related_feedback

      - module: hooks-flag-awareness
        config:
          detect_flag_usage: true
          warn_stale_flags: true
          stale_threshold: 30d

      # User impact
      - module: hooks-user-impact
        config:
          estimate_affected_users: true
          check_analytics_events: true
          warn_missing_tracking: true

      - module: hooks-rollout-guard
        config:
          require_staged_rollout: true
          stages: [1%, 10%, 50%, 100%]
          auto_rollback_threshold:
            error_rate_increase: 10%
            conversion_drop: 5%

      # Collaboration
      - module: hooks-design-handoff
        config:
          check_figma_status: true
          require_design_approval: true
          track_implementation_status: true

    # ===== SCENARIO TOOLS =====
    scenarios:
      - module: feature-planner
        config:
          include_user_impact: true
          include_analytics_plan: true
          output_format: [tech_spec, jira_tickets, test_plan]

      - module: pr-reviewer
        config:
          check_acceptance_criteria: true
          check_design_match: true
          check_analytics_events: true
          suggest_qa_steps: true

  # Product team profiles
  profiles:
    # Feature development
    feature:
      description: "Building new features"
      tools:
        - tool-product-context
        - tool-codebase-search
        - tool-feature-flag
        - tool-test-runner
      hooks:
        - hooks-feature-context
        - hooks-acceptance-validator
        - hooks-design-diff
        - hooks-i18n-checker
        - hooks-a11y-validator
        - hooks-user-impact

    # Bug fixing
    bugfix:
      description: "Fixing user-reported issues"
      tools:
        - tool-user-feedback
        - tool-analytics-query
        - tool-codebase-search
        - tool-test-runner
      hooks:
        - hooks-feature-context
        - hooks-user-impact

    # Feature launch
    launch:
      description: "Rolling out features"
      tools:
        - tool-feature-flag
        - tool-analytics-query
        - tool-a11y-checker
      hooks:
        - hooks-rollout-guard
        - hooks-flag-awareness
        - hooks-user-impact
      config:
        require_staged_rollout: true

    # Design implementation
    design:
      description: "Implementing designs"
      tools:
        - tool-product-context       # Figma access
        - tool-codebase-search
        - tool-a11y-checker
      hooks:
        - hooks-design-diff
        - hooks-design-handoff
        - hooks-a11y-validator
        - hooks-i18n-checker
```

---

## Product Context Tool

Unified view of all product information:

```yaml
tool-product-context:
  description: "Unified product context from all sources"

  sources:
    jira:
      # Pull story details
      fetch:
        - summary
        - description
        - acceptance_criteria
        - linked_issues
        - comments
        - attachments

    figma:
      # Pull design context
      fetch:
        - frames
        - components
        - design_tokens
        - comments
        - version_history

    confluence:
      # Pull specifications
      fetch:
        - prd_documents
        - technical_specs
        - decision_logs

    analytics:
      # Pull usage data
      fetch:
        - related_events
        - funnel_data
        - user_segments

  example_output: |
    📋 Product Context: PROJ-1234 "Add dark mode toggle"

    📝 Story Details:
    - Epic: User Preferences (PROJ-100)
    - Sprint: Sprint 23
    - Priority: High
    - Story Points: 5

    ✅ Acceptance Criteria:
    1. Toggle visible in settings menu
    2. Preference persists across sessions
    3. System preference detection
    4. Smooth transition animation

    🎨 Design:
    - Figma: Dark Mode Settings (v3.2)
    - Components: Toggle, ColorScheme
    - Design Tokens: colors.dark.*

    📊 Analytics Context:
    - Related events: settings_opened, theme_changed
    - Current dark mode usage: 34% of users
    - Feature requests: 156 in past 90 days

    🔗 Related:
    - PRD: confluence.com/dark-mode-spec
    - Tech Spec: confluence.com/dark-mode-tech
    - Related bugs: PROJ-890, PROJ-912
```

---

## Acceptance Criteria Validator

```yaml
hooks-acceptance-validator:
  description: "Validate implementation against acceptance criteria"

  process:
    # 1. Extract AC from story
    extract_from:
      - jira_description
      - jira_custom_field
      - linked_confluence

    # 2. Match to tests
    test_mapping:
      auto_detect: true
      patterns:
        - "test.*should.*{criterion}"
        - "it.*{criterion}"

    # 3. Check coverage
    report:
      - criteria_with_tests
      - criteria_without_tests
      - suggested_test_cases

  example_output: |
    ✅ Acceptance Criteria Validation

    Story: PROJ-1234 "Add dark mode toggle"

    Coverage Report:
    ✅ AC1: "Toggle visible in settings menu"
       → test/settings.spec.ts:45 "should show dark mode toggle"

    ✅ AC2: "Preference persists across sessions"
       → test/settings.spec.ts:67 "should persist preference"

    ⚠️ AC3: "System preference detection"
       → No matching test found
       💡 Suggested: Add test for prefers-color-scheme detection

    ✅ AC4: "Smooth transition animation"
       → test/theme.spec.ts:23 "should animate theme transition"

    Coverage: 3/4 (75%)
    Action Required: Add test for AC3
```

---

## Design Diff Hook

```yaml
hooks-design-diff:
  description: "Compare implementation to Figma designs"

  process:
    # 1. Detect component changes
    trigger_on:
      - "src/components/**/*.tsx"
      - "src/components/**/*.css"

    # 2. Find linked Figma
    figma_linking:
      by_comment: true              # Look for Figma URLs in code
      by_story: true                # Check linked JIRA story
      by_name: true                 # Match component names

    # 3. Compare
    comparison:
      visual_diff: true
      spacing_check: true
      color_check: true
      typography_check: true
      responsive_check: true

    # 4. Report
    tolerance:
      spacing: 4px
      color: 5%                     # Color difference
      font_size: 2px

  example_output: |
    🎨 Design Comparison: Button.tsx

    Figma Frame: "Primary Button" (v3.2)

    Differences Found:
    ⚠️ Padding: Expected 16px, found 14px (diff: 2px)
    ⚠️ Border radius: Expected 8px, found 6px (diff: 2px)
    ✅ Colors: Match
    ✅ Typography: Match
    ✅ Hover state: Match

    Overall: 2 minor deviations
    Design approval: @sarah (designer) notified

    💡 Tip: Use design tokens for consistent spacing
```

---

## User Impact Analysis

```yaml
hooks-user-impact:
  description: "Estimate user impact of changes"

  analysis:
    # Affected users
    user_estimation:
      from_analytics: true
      events_to_check:
        - page_views
        - feature_usage
        - api_calls

    # Impact assessment
    impact_factors:
      - user_count
      - session_frequency
      - revenue_correlation
      - support_ticket_volume

  example_output: |
    👥 User Impact Analysis

    Change: PaymentForm.tsx modifications

    Estimated Impact:
    ├── Daily Active Users: ~45,000 (12% of DAU)
    ├── Monthly Transactions: ~120,000
    ├── Revenue Touch: $2.3M/month flows through
    └── Support Tickets: 34/month related to payments

    Risk Assessment: HIGH
    - Core revenue path
    - High user visibility

    Recommendations:
    1. ✅ Feature flag rollout (staged)
    2. ✅ A/B test conversion impact
    3. ✅ Monitor error rates closely
    4. ⚠️ Consider QA sign-off
```

---

## Rollout Guard

```yaml
hooks-rollout-guard:
  description: "Ensure safe feature rollouts"

  stages:
    # Progressive rollout
    - name: canary
      percentage: 1%
      duration: 1h
      metrics:
        error_rate: < 0.1%
        latency_p99: < 500ms

    - name: early_adopters
      percentage: 10%
      duration: 24h
      metrics:
        error_rate: < 0.5%
        conversion: > -2%

    - name: wide_release
      percentage: 50%
      duration: 48h
      metrics:
        error_rate: < 1%
        conversion: > -1%

    - name: full_release
      percentage: 100%
      metrics:
        stable

  auto_rollback:
    triggers:
      - error_rate_spike: 10%
      - conversion_drop: 5%
      - latency_increase: 100%

    actions:
      - disable_flag
      - notify_team
      - create_incident

  example_output: |
    🚀 Rollout Guard: dark-mode-enabled

    Current Stage: early_adopters (10%)
    Duration: 18h of 24h required

    Metrics:
    ✅ Error rate: 0.12% (threshold: < 0.5%)
    ✅ Latency p99: 234ms (threshold: < 500ms)
    ⚠️ Conversion: -1.8% (threshold: > -2%)

    Status: MONITORING
    Next stage: wide_release (50%) in 6h

    ⚠️ Note: Conversion slightly down - monitor closely
```

---

## Feature Flag Awareness

```yaml
hooks-flag-awareness:
  description: "Track and manage feature flags in code"

  detection:
    # Find flag usage
    patterns:
      - "launchDarkly.variation"
      - "useFeatureFlag"
      - "isEnabled"

    # Track flag lifecycle
    lifecycle:
      - created
      - in_development
      - testing
      - rolling_out
      - fully_enabled
      - cleanup_needed

  warnings:
    stale_flags:
      threshold: 30d
      message: "Flag {name} has been at 100% for {days} days - consider cleanup"

    unused_flags:
      threshold: 14d
      message: "Flag {name} not referenced in code - may be obsolete"

  example_output: |
    🚩 Feature Flag Status

    Active in this file: src/components/Header.tsx

    1. dark-mode-enabled (IN_DEVELOPMENT)
       Created: 5 days ago
       Rollout: 10%
       Owner: @alice

    2. new-navigation (STALE ⚠️)
       Created: 45 days ago
       Rollout: 100% for 32 days
       💡 Consider removing flag and cleaning up code

    3. redesigned-header (FULLY_ENABLED)
       Rollout: 100%
       All environments enabled

    Cleanup candidates: 2 flags
```

---

## Usage Examples

### Example 1: Feature Development

```bash
amplifier --profile product-team:feature

> Implement PROJ-1234 dark mode toggle

# Automatically loads:
# - Story details and AC
# - Figma designs
# - Related analytics
# - Existing theme code

# As you code:
# - Validates against AC
# - Compares to Figma
# - Checks i18n
# - Validates a11y
```

### Example 2: Bug Investigation

```bash
amplifier --profile product-team:bugfix

> Investigate checkout failures from user feedback

# Automatically:
# - Pulls related support tickets
# - Queries analytics for error patterns
# - Finds related code
# - Suggests root causes
```

### Example 3: Feature Launch

```bash
amplifier --profile product-team:launch

> Roll out dark mode feature

# Guides through:
# - Staged rollout setup
# - Metrics monitoring
# - A/B test configuration
# - Rollback planning
```

---

## Dependencies

### Required
- JIRA/Linear for stories
- Analytics platform (Amplitude/Mixpanel)
- Feature flag service

### Optional
- Figma API access
- Support platform (Zendesk/Intercom)
- Visual testing (Percy/Chromatic)

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
