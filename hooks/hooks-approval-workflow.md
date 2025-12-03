# hooks-approval-workflow

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-hooks-approval-workflow`

## Overview

Dynamic approval gates for sensitive operations beyond the kernel's built-in approval system. Enables sophisticated approval logic based on operation context, time of day, affected resources, and organizational policies.

### Value Proposition

| Without | With |
|---------|------|
| Binary allow/deny decisions | Context-aware approval logic |
| Same rules for all operations | Different rules for dev/staging/prod |
| Manual approval tracking | Automated approval workflows |
| No escalation paths | Multi-level approval for high-risk operations |

---

## Contract

### Hook Configuration

```yaml
hooks:
  - module: hooks-approval-workflow
    config:
      # Approval rules (evaluated in order)
      rules:
        # Production file protection
        - name: production_files
          match:
            tool: filesystem
            operation: write
            path_patterns:
              - "production/**"
              - "prod/**"
              - "**/config/production.*"
          action: require_approval
          approval:
            prompt: "Write to production file: {file_path}?"
            timeout: 300
            default: deny

        # Database migrations
        - name: database_migrations
          match:
            tool: migration_helper
            operation: execute
          action: require_approval
          approval:
            prompt: "Execute database migration: {migration_name}?"
            options: ["Run", "Dry-run first", "Cancel"]
            require_reason: true

        # Destructive operations
        - name: destructive_ops
          match:
            tool: "*"
            operation: delete
          action: require_approval
          approval:
            prompt: "Delete {resource_type}: {resource_id}?"
            default: deny

        # After-hours changes
        - name: after_hours
          match:
            time:
              outside: "09:00-18:00"
              timezone: "America/New_York"
          action: require_approval
          approval:
            prompt: "After-hours change. Proceed?"

        # High-cost operations
        - name: expensive_llm_calls
          match:
            event: provider:request
            conditions:
              estimated_tokens: "> 50000"
          action: require_approval
          approval:
            prompt: "Large LLM request (~{estimated_tokens} tokens, ~${estimated_cost}). Continue?"

      # Default behavior for non-matched operations
      default_action: continue

      # Approval caching
      caching:
        enabled: true
        ttl_seconds: 300                # Cache approvals for 5 minutes
        scope: session                  # session | operation
```

### HookResult Actions

```python
# Request approval from user
HookResult(
    action="ask_user",
    approval_prompt="Write to production/config.yaml?\n\nChanges:\n- Updated API endpoint\n- Changed timeout from 30s to 60s",
    approval_options=["Allow", "Deny", "Show diff"],
    approval_timeout=300.0,
    approval_default="deny"
)

# After approval received, continue or deny
HookResult(action="continue")  # If approved
HookResult(action="deny", reason="User denied production file write")  # If denied
```

---

## Architecture

```python
class ApprovalWorkflowHook:
    """Dynamic approval gates based on configurable rules."""

    def __init__(self, config: ApprovalWorkflowConfig):
        self.config = config
        self.rules = [ApprovalRule(r) for r in config.rules]
        self.cache = ApprovalCache(config.caching) if config.caching.enabled else None
        self.pending_approvals: dict[str, PendingApproval] = {}

    async def __call__(self, event: str, data: dict) -> HookResult:
        # Check if any rule matches
        for rule in self.rules:
            if rule.matches(event, data):
                return await self._handle_rule(rule, event, data)

        return HookResult(action="continue")

    async def _handle_rule(
        self,
        rule: ApprovalRule,
        event: str,
        data: dict
    ) -> HookResult:
        # Check cache first
        cache_key = self._compute_cache_key(rule, data)
        if self.cache:
            cached = self.cache.get(cache_key)
            if cached:
                if cached.approved:
                    return HookResult(action="continue")
                else:
                    return HookResult(action="deny", reason=cached.reason)

        # Build approval prompt
        prompt = rule.format_prompt(data)
        options = rule.options or ["Allow", "Deny"]

        # Request approval
        return HookResult(
            action="ask_user",
            approval_prompt=prompt,
            approval_options=options,
            approval_timeout=rule.timeout,
            approval_default=rule.default
        )

    def _compute_cache_key(self, rule: ApprovalRule, data: dict) -> str:
        """Compute cache key based on rule scope."""
        if self.config.caching.scope == "session":
            return f"{rule.name}"
        else:  # operation scope
            return f"{rule.name}:{hash(frozenset(data.items()))}"


class ApprovalRule:
    """Single approval rule with matching and formatting."""

    def __init__(self, config: dict):
        self.name = config["name"]
        self.match_config = config["match"]
        self.action = config["action"]
        self.approval_config = config.get("approval", {})

        # Compile matchers
        self.matchers = self._compile_matchers()

    def matches(self, event: str, data: dict) -> bool:
        """Check if rule matches current event/data."""
        for matcher in self.matchers:
            if not matcher.matches(event, data):
                return False
        return True

    def _compile_matchers(self) -> list[Matcher]:
        matchers = []

        if "tool" in self.match_config:
            matchers.append(ToolMatcher(self.match_config["tool"]))

        if "operation" in self.match_config:
            matchers.append(OperationMatcher(self.match_config["operation"]))

        if "path_patterns" in self.match_config:
            matchers.append(PathMatcher(self.match_config["path_patterns"]))

        if "event" in self.match_config:
            matchers.append(EventMatcher(self.match_config["event"]))

        if "time" in self.match_config:
            matchers.append(TimeMatcher(self.match_config["time"]))

        if "conditions" in self.match_config:
            matchers.append(ConditionMatcher(self.match_config["conditions"]))

        return matchers

    def format_prompt(self, data: dict) -> str:
        """Format approval prompt with data."""
        template = self.approval_config.get("prompt", "Approve this operation?")
        return template.format(**data)


class PathMatcher:
    """Match file paths against patterns."""

    def __init__(self, patterns: list[str]):
        self.patterns = [re.compile(fnmatch.translate(p)) for p in patterns]

    def matches(self, event: str, data: dict) -> bool:
        file_path = data.get("file_path", "")
        return any(p.match(file_path) for p in self.patterns)


class TimeMatcher:
    """Match based on time of day."""

    def __init__(self, config: dict):
        self.outside = config.get("outside")  # "09:00-18:00"
        self.timezone = pytz.timezone(config.get("timezone", "UTC"))

    def matches(self, event: str, data: dict) -> bool:
        now = datetime.now(self.timezone)
        current_time = now.time()

        if self.outside:
            start, end = self._parse_time_range(self.outside)
            # Match if OUTSIDE the range
            return not (start <= current_time <= end)

        return True
```

---

## Rule Matching

### Match Criteria

| Criterion | Description | Example |
|-----------|-------------|---------|
| `tool` | Tool name (glob) | `"filesystem"`, `"*"` |
| `operation` | Operation type | `"write"`, `"delete"`, `"execute"` |
| `path_patterns` | File path patterns | `["production/**", "*.prod.*"]` |
| `event` | Event type | `"provider:request"`, `"tool:pre"` |
| `time.outside` | Time range | `"09:00-18:00"` |
| `time.timezone` | Timezone | `"America/New_York"` |
| `conditions` | Custom conditions | `{"estimated_tokens": "> 50000"}` |

### Condition Operators

- `"="` - Equals
- `"!="` - Not equals
- `">"` - Greater than
- `"<"` - Less than
- `">="` - Greater or equal
- `"<="` - Less or equal
- `"contains"` - String contains
- `"matches"` - Regex match

---

## Examples

### Example 1: Production File Protection

```yaml
rules:
  - name: production_files
    match:
      tool: filesystem
      operation: write
      path_patterns:
        - "production/**"
        - "**/config/prod.*"
    action: require_approval
    approval:
      prompt: |
        ⚠️ Production file change detected

        File: {file_path}

        Are you sure you want to modify this production file?
      options: ["Allow", "Deny"]
      default: deny
```

### Example 2: Multi-Option Approval

```yaml
rules:
  - name: database_migration
    match:
      tool: migration_helper
    action: require_approval
    approval:
      prompt: |
        Database migration: {migration_name}

        Changes:
        {changes_summary}

        What would you like to do?
      options:
        - "Run migration"
        - "Dry-run first"
        - "Show full SQL"
        - "Cancel"
```

### Example 3: Cost-Based Approval

```yaml
rules:
  - name: expensive_operations
    match:
      event: provider:request
      conditions:
        estimated_cost: "> 1.00"
    action: require_approval
    approval:
      prompt: |
        💰 Expensive LLM operation

        Estimated cost: ${estimated_cost}
        Estimated tokens: {estimated_tokens}

        Continue?
      default: deny
```

### Example 4: After-Hours Warning

```yaml
rules:
  - name: after_hours
    match:
      tool: "*"
      operation: write
      time:
        outside: "09:00-18:00"
        timezone: "America/Los_Angeles"
    action: require_approval
    approval:
      prompt: |
        🌙 After-hours change

        It's currently outside business hours.
        This change may not receive timely review.

        Proceed anyway?
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `rules` | list | [] | Approval rules |
| `default_action` | string | "continue" | Action when no rule matches |
| `caching.enabled` | bool | false | Cache approvals |
| `caching.ttl_seconds` | int | 300 | Cache TTL |
| `caching.scope` | string | "session" | Cache scope |

---

## Security Considerations

- Rules evaluated server-side, can't be bypassed
- Approval cache prevents prompt fatigue attacks
- Default-deny recommended for sensitive operations
- Audit log should capture all approval decisions

---

## Dependencies

### Required
- `fnmatch` - Pattern matching (stdlib)

### Optional
- `pytz` - Timezone handling

---

## Open Questions

1. **Escalation**: Support multi-level approval (manager approval)?
2. **Delegation**: Allow pre-approved operations for specific users?
3. **Time limits**: Approval valid for specific time window?
4. **Notifications**: Notify team on certain approvals?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
