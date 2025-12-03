# hooks-token-budget

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-hooks-token-budget`

## Overview

Enforce token usage limits at session, user, and project levels. Prevents runaway costs from expensive LLM operations and enables usage-based quotas.

### Value Proposition

| Without | With |
|---------|------|
| Unlimited token consumption | Controlled spending |
| Surprise LLM bills | Predictable costs |
| No per-user limits | Fair usage allocation |
| Single runaway session can be costly | Session-level caps |

---

## Contract

### Hook Configuration

```yaml
hooks:
  - module: hooks-token-budget
    config:
      # Budget levels (all optional)
      budgets:
        # Per-session limit
        session:
          max_tokens: 100000            # Total tokens per session
          max_requests: 50              # Max LLM requests per session
          warn_at_percent: 80           # Warn when 80% consumed

        # Per-user limit (daily)
        user:
          daily_tokens: 500000
          daily_requests: 200
          reset_time: "00:00"           # UTC
          warn_at_percent: 90

        # Per-project limit (monthly)
        project:
          monthly_tokens: 10000000
          monthly_cost_usd: 500.00
          warn_at_percent: 75

      # Actions when budget exceeded
      on_exceed:
        session: deny                   # deny | warn | continue
        user: deny
        project: warn

      # Cost tracking
      cost:
        track: true
        # Token costs per model (per 1M tokens)
        pricing:
          claude-3-opus: {input: 15.00, output: 75.00}
          claude-3-sonnet: {input: 3.00, output: 15.00}
          gpt-4-turbo: {input: 10.00, output: 30.00}
          gpt-4o: {input: 5.00, output: 15.00}

      # Storage for tracking
      storage:
        type: file                      # file | redis | database
        path: "${HOME}/.amplifier/budgets/"
```

### HookResult Actions

```python
# Warn approaching limit
HookResult(
    action="inject_context",
    context_injection="⚠️ Token budget: 82% consumed (82,000 / 100,000 tokens). Consider wrapping up soon.",
    user_message="Budget warning: 82% of session tokens used",
    user_message_level="warning"
)

# Deny when exceeded
HookResult(
    action="deny",
    reason="Session token budget exceeded (100,000 tokens). Start a new session to continue."
)
```

---

## Architecture

```python
class TokenBudgetHook:
    """Enforce token usage budgets."""

    def __init__(self, config: TokenBudgetConfig):
        self.config = config
        self.storage = BudgetStorage(config.storage)
        self.pricing = config.cost.pricing if config.cost.track else {}

        # Session-level tracking (in-memory)
        self.session_usage: dict[str, Usage] = {}

    async def __call__(self, event: str, data: dict) -> HookResult:
        session_id = data.get("session_id")
        user_id = data.get("user_id")

        # Pre-request: Check budgets before allowing
        if event == "provider:request":
            return await self._check_budgets(session_id, user_id, data)

        # Post-response: Track usage
        if event == "provider:response":
            await self._track_usage(session_id, user_id, data)
            return await self._check_warnings(session_id, user_id)

        return HookResult(action="continue")

    async def _check_budgets(
        self,
        session_id: str,
        user_id: str | None,
        data: dict
    ) -> HookResult:
        """Check if request would exceed any budget."""

        # Estimate tokens for this request
        estimated = self._estimate_request_tokens(data)

        # Check session budget
        if self.config.budgets.session:
            session_usage = self._get_session_usage(session_id)
            if session_usage.tokens + estimated > self.config.budgets.session.max_tokens:
                if self.config.on_exceed.session == "deny":
                    return HookResult(
                        action="deny",
                        reason=f"Session token budget exceeded ({session_usage.tokens:,} / {self.config.budgets.session.max_tokens:,})"
                    )

        # Check user budget
        if self.config.budgets.user and user_id:
            user_usage = await self.storage.get_user_usage(user_id)
            if user_usage.daily_tokens + estimated > self.config.budgets.user.daily_tokens:
                if self.config.on_exceed.user == "deny":
                    return HookResult(
                        action="deny",
                        reason=f"Daily user token budget exceeded. Resets at {self.config.budgets.user.reset_time} UTC."
                    )

        # Check project budget
        if self.config.budgets.project:
            project_usage = await self.storage.get_project_usage()
            if project_usage.monthly_tokens + estimated > self.config.budgets.project.monthly_tokens:
                if self.config.on_exceed.project == "deny":
                    return HookResult(
                        action="deny",
                        reason="Monthly project token budget exceeded."
                    )

        return HookResult(action="continue")

    async def _track_usage(
        self,
        session_id: str,
        user_id: str | None,
        data: dict
    ) -> None:
        """Track token usage from response."""
        usage = data.get("usage", {})
        input_tokens = usage.get("input_tokens", 0)
        output_tokens = usage.get("output_tokens", 0)
        total_tokens = input_tokens + output_tokens

        model = data.get("model", "unknown")
        cost = self._calculate_cost(model, input_tokens, output_tokens)

        # Update session usage
        session_usage = self._get_session_usage(session_id)
        session_usage.tokens += total_tokens
        session_usage.requests += 1
        session_usage.cost += cost

        # Update persistent storage
        if user_id:
            await self.storage.add_user_usage(user_id, total_tokens, cost)
        await self.storage.add_project_usage(total_tokens, cost)

    async def _check_warnings(
        self,
        session_id: str,
        user_id: str | None
    ) -> HookResult:
        """Check if any budget is approaching limit."""
        warnings = []

        # Session warning
        if self.config.budgets.session:
            session_usage = self._get_session_usage(session_id)
            percent = (session_usage.tokens / self.config.budgets.session.max_tokens) * 100
            if percent >= self.config.budgets.session.warn_at_percent:
                warnings.append(f"Session: {percent:.0f}% ({session_usage.tokens:,} / {self.config.budgets.session.max_tokens:,} tokens)")

        # User warning
        if self.config.budgets.user and user_id:
            user_usage = await self.storage.get_user_usage(user_id)
            percent = (user_usage.daily_tokens / self.config.budgets.user.daily_tokens) * 100
            if percent >= self.config.budgets.user.warn_at_percent:
                warnings.append(f"Daily: {percent:.0f}% of daily limit")

        if warnings:
            return HookResult(
                action="inject_context",
                context_injection=f"⚠️ Token budget warnings:\n" + "\n".join(f"- {w}" for w in warnings),
                suppress_output=True,
                user_message=f"Budget: {'; '.join(warnings)}",
                user_message_level="warning"
            )

        return HookResult(action="continue")

    def _calculate_cost(self, model: str, input_tokens: int, output_tokens: int) -> float:
        """Calculate cost for token usage."""
        if model not in self.pricing:
            return 0.0

        pricing = self.pricing[model]
        input_cost = (input_tokens / 1_000_000) * pricing["input"]
        output_cost = (output_tokens / 1_000_000) * pricing["output"]
        return input_cost + output_cost


class BudgetStorage:
    """Persistent storage for budget tracking."""

    async def get_user_usage(self, user_id: str) -> UserUsage:
        """Get user's current usage, resetting if new day."""
        data = await self._read_user_file(user_id)

        # Check if we need to reset (new day)
        if data and data["date"] != self._today():
            data = None

        if not data:
            return UserUsage(daily_tokens=0, daily_cost=0.0)

        return UserUsage(**data)

    async def add_user_usage(self, user_id: str, tokens: int, cost: float) -> None:
        """Add usage to user's daily total."""
        usage = await self.get_user_usage(user_id)
        usage.daily_tokens += tokens
        usage.daily_cost += cost
        await self._write_user_file(user_id, {
            "date": self._today(),
            "daily_tokens": usage.daily_tokens,
            "daily_cost": usage.daily_cost
        })
```

---

## Examples

### Example 1: Session Limit Warning

```python
# After 82,000 tokens in session with 100,000 limit:
{
    "action": "inject_context",
    "context_injection": "⚠️ Token budget warnings:\n- Session: 82% (82,000 / 100,000 tokens)",
    "user_message": "Budget: Session: 82% (82,000 / 100,000 tokens)",
    "user_message_level": "warning"
}
```

### Example 2: Session Limit Exceeded

```python
# Request would exceed session limit:
{
    "action": "deny",
    "reason": "Session token budget exceeded (100,000 / 100,000 tokens). Start a new session to continue."
}
```

### Example 3: Cost Tracking

```python
# After response with usage data:
# Model: claude-3-sonnet
# Input: 5,000 tokens, Output: 2,000 tokens
# Cost: (5000/1M * $3) + (2000/1M * $15) = $0.015 + $0.030 = $0.045
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `budgets.session.max_tokens` | int | none | Session token limit |
| `budgets.session.max_requests` | int | none | Session request limit |
| `budgets.user.daily_tokens` | int | none | Daily per-user limit |
| `budgets.project.monthly_tokens` | int | none | Monthly project limit |
| `budgets.*.warn_at_percent` | int | 80 | Warning threshold |
| `on_exceed.*` | string | "deny" | Action when exceeded |
| `cost.track` | bool | true | Track costs |
| `cost.pricing` | dict | {} | Model pricing |

---

## Storage Schema

```json
// User usage file: ~/.amplifier/budgets/users/{user_id}.json
{
    "date": "2024-01-15",
    "daily_tokens": 45000,
    "daily_cost": 1.35,
    "daily_requests": 23
}

// Project usage file: ~/.amplifier/budgets/project.json
{
    "month": "2024-01",
    "monthly_tokens": 2500000,
    "monthly_cost": 125.50,
    "monthly_requests": 1250
}
```

---

## Security Considerations

- Budget data stored securely
- User quotas prevent abuse
- Cost tracking enables chargeback

---

## Dependencies

### Required
- `aiofiles` - Async file operations

### Optional
- `redis` - For distributed budget tracking

---

## Open Questions

1. **Distributed tracking**: How to share budgets across instances?
2. **Quota inheritance**: Project limits vs team limits vs user limits?
3. **Alerting**: Notify admins when project budget is low?
4. **Rollover**: Allow unused quota to roll over?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
