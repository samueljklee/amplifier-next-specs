# hooks-ticket-linker

> **Priority**: P0 (Foundation)
> **Status**: Draft
> **Module**: `amplifier-module-hooks-ticket-linker`

## Overview

Automatically loads JIRA/GitHub issue context based on the current git branch. Injects ticket information into the agent's context at session start, ensuring work is always connected to its tracking ticket.

### Value Proposition

| Without | With |
|---------|------|
| Manually look up ticket for context | Automatic context from branch name |
| Forget to link work to tickets | Always-on ticket awareness |
| Context switching to find requirements | Requirements injected at session start |
| "What was I working on?" | Full ticket context on resume |

---

## Contract

### Hook Configuration

```yaml
hooks:
  - module: hooks-ticket-linker
    config:
      # Ticket systems
      jira:
        enabled: true
        base_url: "${JIRA_URL}"
        auth_token: "${JIRA_TOKEN}"
        project_keys: ["PROJ", "PLATFORM", "SEC"]

      github:
        enabled: true
        auth_token: "${GITHUB_TOKEN}"

      # Branch patterns to extract ticket
      branch_patterns:
        - "^(?P<key>[A-Z]+-\\d+)"           # PROJ-123-description
        - "^feature/(?P<key>[A-Z]+-\\d+)"   # feature/PROJ-123-description
        - "^fix/(?P<key>[A-Z]+-\\d+)"       # fix/PROJ-123-description
        - "^(?P<key>\\d+)-"                 # 123-description (GitHub issue)

      # What to include in context
      include:
        summary: true
        description: true
        acceptance_criteria: true
        comments: false                     # Can be verbose
        linked_issues: true
        status: true
        assignee: true

      # Context injection
      context:
        role: system                        # system | user
        max_length: 2000                    # Truncate if too long
```

### HookResult

```python
# At session:start, inject ticket context
HookResult(
    action="inject_context",
    context_injection="""
## Current Work Context

**Ticket**: PROJ-456 - Add retry logic to payment gateway
**Status**: In Progress
**Assignee**: alice

### Description
Implement exponential backoff retry for payment API calls to handle gateway timeouts during peak load.

### Acceptance Criteria
- [ ] Retry up to 3 times with exponential backoff
- [ ] Log each retry attempt
- [ ] Add circuit breaker for repeated failures
- [ ] Update metrics dashboard

### Related Issues
- PROJ-400: Payment gateway timeout errors (parent)
- PROJ-455: Add payment metrics dashboard
""",
    context_injection_role="system",
    suppress_output=True,
    user_message="Loaded context for PROJ-456"
)
```

---

## Architecture

```python
class TicketLinkerHook:
    """Auto-load ticket context at session start."""

    def __init__(self, config: TicketLinkerConfig):
        self.config = config
        self.jira = JiraClient(config.jira) if config.jira.enabled else None
        self.github = GitHubClient(config.github) if config.github.enabled else None
        self.patterns = [re.compile(p) for p in config.branch_patterns]

    async def __call__(self, event: str, data: dict) -> HookResult:
        if event != "session:start":
            return HookResult(action="continue")

        # Get current branch
        branch = await self._get_current_branch()
        if not branch:
            return HookResult(action="continue")

        # Extract ticket key from branch name
        ticket_key = self._extract_ticket_key(branch)
        if not ticket_key:
            return HookResult(action="continue")

        # Fetch ticket details
        ticket = await self._fetch_ticket(ticket_key)
        if not ticket:
            return HookResult(
                action="continue",
                user_message=f"Could not load ticket {ticket_key}"
            )

        # Format context
        context = self._format_context(ticket)

        return HookResult(
            action="inject_context",
            context_injection=context,
            context_injection_role=self.config.context.role,
            suppress_output=True,
            user_message=f"Loaded context for {ticket_key}: {ticket.summary[:50]}..."
        )

    def _extract_ticket_key(self, branch: str) -> str | None:
        """Extract ticket key from branch name using patterns."""
        for pattern in self.patterns:
            match = pattern.search(branch)
            if match:
                return match.group("key")
        return None

    async def _fetch_ticket(self, key: str) -> Ticket | None:
        """Fetch ticket from appropriate system."""
        # Try JIRA first (if key looks like JIRA format)
        if self.jira and re.match(r'[A-Z]+-\d+', key):
            try:
                return await self.jira.get_issue(key)
            except Exception:
                pass

        # Try GitHub (if numeric)
        if self.github and key.isdigit():
            try:
                return await self.github.get_issue(int(key))
            except Exception:
                pass

        return None

    def _format_context(self, ticket: Ticket) -> str:
        """Format ticket as context string."""
        lines = [
            "## Current Work Context",
            "",
            f"**Ticket**: {ticket.key} - {ticket.summary}",
            f"**Status**: {ticket.status}",
        ]

        if self.config.include.assignee and ticket.assignee:
            lines.append(f"**Assignee**: {ticket.assignee}")

        if self.config.include.description and ticket.description:
            lines.extend(["", "### Description", ticket.description])

        if self.config.include.acceptance_criteria and ticket.acceptance_criteria:
            lines.extend(["", "### Acceptance Criteria"])
            for criterion in ticket.acceptance_criteria:
                status = "x" if criterion.done else " "
                lines.append(f"- [{status}] {criterion.text}")

        if self.config.include.linked_issues and ticket.linked_issues:
            lines.extend(["", "### Related Issues"])
            for linked in ticket.linked_issues:
                lines.append(f"- {linked.key}: {linked.summary} ({linked.relationship})")

        # Truncate if too long
        context = "\n".join(lines)
        if len(context) > self.config.context.max_length:
            context = context[:self.config.context.max_length] + "\n\n[Truncated]"

        return context
```

---

## Branch Pattern Examples

| Branch Name | Pattern | Extracted Key |
|-------------|---------|---------------|
| `PROJ-123-add-retry` | `^(?P<key>[A-Z]+-\\d+)` | PROJ-123 |
| `feature/PROJ-456-new-ui` | `^feature/(?P<key>[A-Z]+-\\d+)` | PROJ-456 |
| `fix/SEC-789-xss-patch` | `^fix/(?P<key>[A-Z]+-\\d+)` | SEC-789 |
| `123-github-issue` | `^(?P<key>\\d+)-` | 123 |
| `bugfix/PLATFORM-100` | `^bugfix/(?P<key>[A-Z]+-\\d+)` | PLATFORM-100 |

---

## Examples

### Example 1: JIRA Ticket Context

```python
# Session starts on branch: feature/PROJ-456-payment-retry

# Hook fetches JIRA ticket and returns:
{
    "action": "inject_context",
    "context_injection": """
## Current Work Context

**Ticket**: PROJ-456 - Add retry logic to payment gateway
**Status**: In Progress
**Assignee**: alice

### Description
Implement exponential backoff retry for payment API calls to handle gateway timeouts during peak load. This addresses the 5% payment failure rate observed during Black Friday.

### Acceptance Criteria
- [ ] Retry up to 3 times with exponential backoff
- [ ] Log each retry attempt with correlation ID
- [ ] Add circuit breaker for repeated failures
- [ ] Update Grafana dashboard with retry metrics
- [ ] Add unit tests for retry logic

### Related Issues
- PROJ-400: Payment gateway timeout errors (parent epic)
- PROJ-455: Add payment metrics dashboard (blocks)
""",
    "context_injection_role": "system",
    "suppress_output": true,
    "user_message": "Loaded context for PROJ-456: Add retry logic to payment gateway"
}
```

### Example 2: GitHub Issue Context

```python
# Session starts on branch: 789-fix-auth-bug

# Hook fetches GitHub issue #789 and returns:
{
    "action": "inject_context",
    "context_injection": """
## Current Work Context

**Issue**: #789 - Session timeout not respecting remember-me setting
**Status**: Open
**Assignee**: bob
**Labels**: bug, priority:high, auth

### Description
Users with "remember me" checked are still being logged out after 30 minutes. Expected behavior is 30 day session.

### Linked PRs
- #785: Initial remember-me implementation (merged)
""",
    "user_message": "Loaded context for #789"
}
```

### Example 3: No Ticket Found

```python
# Session starts on branch: main

# Hook returns (no injection, just continue):
{
    "action": "continue"
}

# No ticket pattern matched in branch name
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `jira.enabled` | bool | false | Enable JIRA integration |
| `jira.base_url` | string | - | JIRA instance URL |
| `jira.auth_token` | string | - | JIRA API token |
| `jira.project_keys` | list | [] | Valid project keys |
| `github.enabled` | bool | false | Enable GitHub integration |
| `branch_patterns` | list | (defaults) | Regex patterns for extraction |
| `include.*` | bool | varies | What ticket fields to include |
| `context.role` | string | "system" | Injection role |
| `context.max_length` | int | 2000 | Max context length |

---

## Security Considerations

- API tokens stored securely in environment
- Ticket content may contain sensitive information
- Context injection respects session permissions

---

## Dependencies

### Required
- `httpx` - HTTP client for API calls

### Optional
- `jira` - Official JIRA client
- `PyGithub` - GitHub API client

---

## Open Questions

1. **Multiple tickets**: What if branch references multiple tickets?
2. **Caching**: Cache ticket data to avoid repeated API calls?
3. **Updates**: Re-fetch if ticket was updated during session?
4. **Custom fields**: Support JIRA custom fields?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
