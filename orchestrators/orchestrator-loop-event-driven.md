# orchestrator-loop-event-driven

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-loop-event-driven`

## Overview

An orchestrator that responds to external events rather than user prompts. The agent runs continuously (or on-demand), watching for triggers like file changes, git pushes, webhooks, or scheduled times. Demonstrates Amplifier as an automation engine, not requiring interactive input.

### Value Proposition

| Without | With |
|---------|------|
| User must initiate every interaction | Agent responds to events automatically |
| Manual triggers for CI/CD workflows | Git push triggers code review |
| Polling and manual checks | Webhook-driven instant response |
| Scheduled tasks require separate tooling | Built-in cron-like scheduling |

### Use Cases

1. **PR auto-review**: New PR triggers security/quality analysis
2. **File watcher**: Code changes trigger test generation or docs update
3. **Webhook handler**: External service events trigger processing
4. **Scheduled jobs**: Daily report generation, weekly summaries
5. **Alert responder**: Monitoring alerts trigger incident investigation

### Modality Differentiator

This breaks the "chatbot" mental model by:
- **No user prompt**: Events trigger actions
- **Continuous operation**: Runs in background
- **External integration**: Responds to real-world triggers
- **Automation-first**: Set up once, runs forever

---

## Contract

### Orchestrator Interface

```python
from dataclasses import dataclass
from typing import Any, Callable
from enum import Enum


class EventType(Enum):
    FILE_CHANGE = "file_change"
    GIT_PUSH = "git_push"
    GIT_PR = "git_pr"
    WEBHOOK = "webhook"
    SCHEDULE = "schedule"
    MANUAL = "manual"
    CUSTOM = "custom"


@dataclass
class Event:
    """External event that triggers processing."""
    type: EventType
    source: str
    payload: dict[str, Any]
    timestamp: str
    metadata: dict[str, Any] | None = None


@dataclass
class EventResult:
    """Result of processing an event."""
    event_id: str
    success: bool
    actions_taken: list[str]
    output: Any | None = None
    error: str | None = None


class EventDrivenOrchestrator:
    """
    Orchestrator that responds to external events.

    Lifecycle:
    1. Register event sources (watchers, webhooks, schedules)
    2. Wait for events
    3. Match event to handler
    4. Execute handler pipeline
    5. Report result
    6. Continue waiting
    """

    async def execute(
        self,
        prompt: str,  # Initial config or ignored in daemon mode
        context: Any,
        providers: dict[str, Any],
        tools: dict[str, Any],
        hooks: Any,
        coordinator: Any | None = None,
    ) -> str:
        """
        Start event-driven execution.

        In daemon mode, this runs indefinitely.
        In one-shot mode, processes single event and exits.
        """

    async def handle_event(
        self,
        event: Event,
        handler: str,  # Handler name or pipeline
    ) -> EventResult:
        """Process a single event through its handler."""

    def register_source(
        self,
        source_type: EventType,
        config: dict[str, Any],
        handler: str,
    ) -> None:
        """Register an event source and its handler."""

    async def start(self) -> None:
        """Start listening for events (daemon mode)."""

    async def stop(self) -> None:
        """Stop listening and cleanup."""
```

### Configuration Schema

```toml
[[orchestrators]]
module = "loop-event-driven"
config = {
  # Operating mode
  mode = "daemon"                      # "daemon" | "one-shot"

  # Event sources
  sources = [
    # File watcher
    {
      type = "file_change",
      path = "./src/",
      pattern = "**/*.py",
      events = ["modify", "create"],
      handler = "on-code-change",
      debounce_ms = 1000,              # Wait for rapid changes to settle
    },

    # Git hooks
    {
      type = "git_push",
      branches = ["main", "develop"],
      handler = "on-push",
    },
    {
      type = "git_pr",
      events = ["opened", "synchronize"],
      handler = "on-pr",
    },

    # Webhooks
    {
      type = "webhook",
      path = "/hooks/deploy",
      secret = "${WEBHOOK_SECRET}",
      handler = "on-deploy-webhook",
    },

    # Schedules (cron syntax)
    {
      type = "schedule",
      cron = "0 9 * * MON",            # Every Monday at 9am
      handler = "weekly-summary",
    },
    {
      type = "schedule",
      cron = "0 */4 * * *",            # Every 4 hours
      handler = "health-check",
    },
  ]

  # Handlers (pipelines for each event type)
  handlers = {
    on-code-change = {
      steps = [
        { prompt = "Analyze changes in {files}: what tests might need updating?" },
        { tool = "tool-filesystem", action = "write", path = "suggested-tests.md" }
      ]
    },
    on-pr = {
      recipe = "recipes/pr-reviewer"   # Use existing recipe
    },
    weekly-summary = {
      prompt = "Generate weekly summary of {repo} activity since {last_run}"
    }
  }

  # Concurrency
  max_concurrent_handlers = 3
  queue_overflow = "drop"              # "drop" | "queue" | "error"

  # Webhook server (if webhook sources used)
  webhook_port = 8080
  webhook_host = "0.0.0.0"

  # Persistence
  state_file = ".amplifier/event-state.json"
  persist_last_run = true              # Remember last run times for schedules
}
```

---

## Architecture

### Event Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    EVENT SOURCES                             │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │  File    │  │   Git    │  │ Webhook  │  │ Schedule │    │
│  │ Watcher  │  │  Hooks   │  │ Server   │  │  Timer   │    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘    │
│       │             │             │             │           │
└───────┼─────────────┼─────────────┼─────────────┼───────────┘
        │             │             │             │
        └─────────────┴──────┬──────┴─────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    EVENT ROUTER                              │
│  Match source → handler │ Filter │ Dedupe │ Queue           │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    HANDLER EXECUTOR                          │
│                                                              │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐   │
│  │  Handler 1  │     │  Handler 2  │     │  Handler 3  │   │
│  │  (PR Rev)   │     │ (File Watch)│     │  (Schedule) │   │
│  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘   │
│         │                   │                   │           │
└─────────┼───────────────────┼───────────────────┼───────────┘
          │                   │                   │
          └───────────────────┴───────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    RESULT REPORTER                           │
│  Log │ Notify │ Store │ Update Status                       │
└─────────────────────────────────────────────────────────────┘
```

### Event Source Implementations

```python
class FileWatcherSource:
    """Watch filesystem for changes."""

    async def start(self, config: dict, callback: Callable):
        watcher = Watchdog(config["path"], config["pattern"])
        watcher.on_change = lambda files: callback(Event(
            type=EventType.FILE_CHANGE,
            source=config["path"],
            payload={"files": files}
        ))
        await watcher.start()


class GitHookSource:
    """Receive git hook events."""

    async def start(self, config: dict, callback: Callable):
        # Register with git hook dispatcher
        await git_hooks.register(
            events=config.get("events", ["push"]),
            branches=config.get("branches"),
            callback=lambda data: callback(Event(
                type=EventType.GIT_PUSH,
                source="git",
                payload=data
            ))
        )


class WebhookSource:
    """HTTP webhook receiver."""

    async def start(self, config: dict, callback: Callable):
        @app.post(config["path"])
        async def webhook_handler(request):
            if not verify_signature(request, config["secret"]):
                raise HTTPException(401)

            callback(Event(
                type=EventType.WEBHOOK,
                source=config["path"],
                payload=await request.json()
            ))
            return {"status": "accepted"}


class ScheduleSource:
    """Cron-like scheduler."""

    async def start(self, config: dict, callback: Callable):
        scheduler = AsyncScheduler()
        scheduler.add_job(
            cron=config["cron"],
            func=lambda: callback(Event(
                type=EventType.SCHEDULE,
                source=config["cron"],
                payload={"scheduled_time": datetime.now()}
            ))
        )
        await scheduler.start()
```

---

## Events

```python
# Event-driven lifecycle events
"event:received"       # External event received
"event:queued"         # Event queued for processing
"event:handler:start"  # Handler begins processing
"event:handler:end"    # Handler completes
"event:dropped"        # Event dropped (queue full, filtered)

# Source lifecycle
"source:started"       # Event source started
"source:stopped"       # Event source stopped
"source:error"         # Event source error
```

---

## Examples

### PR Auto-Review Setup

```yaml
# .amplifier/profiles/pr-reviewer-daemon.md
---
profile:
  name: pr-reviewer-daemon
  extends: dev

session:
  orchestrator:
    module: loop-event-driven
    config:
      mode: daemon
      sources:
        - type: git_pr
          events: [opened, synchronize]
          handler: review-pr

      handlers:
        review-pr:
          recipe: recipes/pr-reviewer
          config:
            post_comment: true
            require_approval: false
---

# PR Review Daemon

Automatically reviews PRs when they're opened or updated.
```

```bash
# Start the daemon
amplifier daemon --profile pr-reviewer-daemon

# Or as a service
amplifier daemon --profile pr-reviewer-daemon --background --pid-file /var/run/amplifier-pr.pid
```

### File Watcher for Docs

```yaml
sources:
  - type: file_change
    path: "./src/"
    pattern: "**/*.py"
    events: [modify, create]
    debounce_ms: 2000
    handler: update-docs

handlers:
  update-docs:
    steps:
      - tool: tool-filesystem
        action: read
        path: "{changed_files}"

      - prompt: |
          These files were modified:
          {files_content}

          Update the corresponding documentation in ./docs/
          to reflect any API changes.

      - tool: tool-filesystem
        action: write
```

### Webhook Integration

```python
# External service calls webhook when deployment completes
# Amplifier validates signature and triggers handler

# curl -X POST http://localhost:8080/hooks/deploy \
#   -H "X-Signature: sha256=..." \
#   -d '{"environment": "production", "version": "1.2.3"}'

handlers:
  on-deploy:
    steps:
      - prompt: |
          Deployment completed:
          - Environment: {payload.environment}
          - Version: {payload.version}

          Generate release notes and update changelog.
```

### Scheduled Reports

```yaml
sources:
  - type: schedule
    cron: "0 9 * * MON"  # Monday 9am
    handler: weekly-summary

handlers:
  weekly-summary:
    steps:
      - tool: tool-git-advanced
        action: log
        since: "1 week ago"

      - prompt: |
          Generate a weekly summary for the team:

          Commits: {git_log}

          Include:
          - Key changes
          - Notable PRs merged
          - Upcoming work

      - tool: tool-slack
        action: post
        channel: "#team-updates"
        message: "{summary}"
```

---

## CLI Integration

```bash
# Start event daemon
amplifier daemon --profile my-profile

# Start with specific sources only
amplifier daemon --profile my-profile --sources file_change,schedule

# One-shot mode (process single event and exit)
amplifier event --type webhook --payload '{"action": "test"}'

# List registered sources
amplifier daemon status

# Trigger manual event
amplifier daemon trigger --handler weekly-summary

# View event history
amplifier daemon history --limit 50
```

---

## Integration with Hooks

```python
# Filter events before processing
@hook("event:received")
async def filter_events(event, data):
    """Skip events from certain sources."""
    if data["event"].payload.get("bot"):
        return HookResult(action="skip", reason="Ignoring bot events")
    return HookResult(action="continue")

# Notify on completion
@hook("event:handler:end")
async def notify_completion(event, data):
    """Send notification when handler completes."""
    if data["result"].success:
        await send_slack(f"Handler {data['handler']} completed successfully")
    else:
        await send_pagerduty(f"Handler {data['handler']} failed: {data['result'].error}")
```

---

## State Management

```python
@dataclass
class EventState:
    """Persisted state for event-driven orchestrator."""

    # Last run times for schedules
    last_run: dict[str, datetime]

    # Processed event IDs (for deduplication)
    processed_events: set[str]

    # Queue state (for restart recovery)
    pending_queue: list[Event]

    # Handler state (for long-running handlers)
    handler_state: dict[str, Any]
```

---

## Testing Strategy

```python
class TestEventDrivenOrchestrator:
    async def test_file_watcher_triggers_handler(self):
        """File change triggers correct handler."""

    async def test_debounce_rapid_changes(self):
        """Rapid file changes are debounced."""

    async def test_webhook_validates_signature(self):
        """Webhook rejects invalid signatures."""

    async def test_schedule_fires_on_time(self):
        """Scheduled events fire at correct times."""

    async def test_concurrent_handler_limit(self):
        """Max concurrent handlers is respected."""

    async def test_queue_overflow_handling(self):
        """Queue overflow is handled per config."""

    async def test_state_persistence(self):
        """State survives restart."""
```

---

## Open Questions

1. **Exactly-once semantics**: How to guarantee an event is processed exactly once across restarts?
2. **Dead letter queue**: Where do failed events go for retry/inspection?
3. **Event sourcing**: Should all events be logged for replay/audit?
4. **Multi-node**: How to distribute event processing across multiple Amplifier instances?
5. **Backpressure**: How to handle event flood without losing events?
