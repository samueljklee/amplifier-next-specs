# hooks-standup-logger

> **Priority**: P2 (Nice to Have)
> **Status**: Draft
> **Module**: `amplifier-module-hooks-standup-logger`

## Overview

Automatically captures work activity from AI sessions and generates concise standup summaries. Enables async standups by providing clear, structured updates of what was accomplished, in progress, and blocked.

### Value Proposition

| Without | With |
|---------|------|
| Manual standup note-taking | Auto-generated summaries |
| Forgetting what you worked on | Complete activity capture |
| Inconsistent update formats | Structured, readable updates |
| Time spent writing updates | Focus on actual work |

---

## Contract

### Hook Configuration

```yaml
hooks:
  - module: hooks-standup-logger
    config:
      # What to capture
      capture:
        file_changes: true              # Files created/modified
        tool_usage: true                # Tools invoked and results
        decisions: true                 # Key decisions made
        blockers: true                  # Issues encountered
        completions: true               # Tasks completed

      # Summarization
      summary:
        format: structured              # structured | narrative | bullet
        max_length: 500                 # Max chars per summary
        include_metrics: true           # Include time/token stats
        group_by: task                  # task | time | file

      # Storage
      storage:
        type: file                      # file | notion | slack | jira
        path: "~/.amplifier/standups/"
        daily_rollup: true              # Combine sessions into daily log

      # Integration
      integration:
        auto_post: false                # Auto-post to channel
        slack_channel: "#team-standups"
        schedule: "09:00"               # When to post daily rollup

      # Privacy
      privacy:
        exclude_patterns:               # Don't mention these files
          - "*.env"
          - "*secret*"
        anonymize_paths: false          # Hash sensitive paths
```

### HookResult

```python
# At session end, generate summary
HookResult(
    action="inject_context",
    context_injection="",  # No injection needed
    user_message="""
📋 Session Summary (45 min)

**Completed:**
- Fixed payment timeout bug in gateway.py
- Added retry logic with exponential backoff
- Updated tests (3 new test cases)

**In Progress:**
- Refactoring error handling in api/routes.py

**Files Changed:** 4 files, +127/-34 lines

Logged to: ~/.amplifier/standups/2024-01-15.md
""",
    user_message_level="info"
)
```

---

## Architecture

```python
class StandupLoggerHook:
    """Capture and summarize work for standups."""

    CAPTURED_EVENTS = [
        "session:start",
        "session:end",
        "tool:post",
        "prompt:submit",
        "prompt:complete",
    ]

    def __init__(self, config: StandupLoggerConfig):
        self.config = config
        self.storage = StandupStorage(config.storage)
        self.summarizer = WorkSummarizer(config.summary)

        # Session activity tracking
        self.current_session: SessionActivity | None = None

    async def __call__(self, event: str, data: dict) -> HookResult:
        if event == "session:start":
            self._start_tracking(data)
            return HookResult(action="continue")

        if event == "session:end":
            return await self._finalize_session(data)

        if event in self.CAPTURED_EVENTS:
            self._track_activity(event, data)

        return HookResult(action="continue")

    def _start_tracking(self, data: dict) -> None:
        """Initialize session tracking."""
        self.current_session = SessionActivity(
            session_id=data.get("session_id"),
            start_time=datetime.utcnow(),
            activities=[],
            files_changed=set(),
            tools_used={},
            decisions=[],
            blockers=[]
        )

    def _track_activity(self, event: str, data: dict) -> None:
        """Track activity during session."""
        if not self.current_session:
            return

        if event == "tool:post":
            self._track_tool_usage(data)
        elif event == "prompt:submit":
            self._extract_intent(data)
        elif event == "prompt:complete":
            self._extract_outcomes(data)

    def _track_tool_usage(self, data: dict) -> None:
        """Track tool invocations."""
        tool = data.get("tool", "")

        # Track file changes
        if tool in ("filesystem:write", "filesystem:edit"):
            file_path = data.get("file_path", "")
            if not self._should_exclude(file_path):
                self.current_session.files_changed.add(file_path)
                self.current_session.activities.append(Activity(
                    type="file_change",
                    description=f"Modified {Path(file_path).name}",
                    timestamp=datetime.utcnow(),
                    metadata={"path": file_path, "operation": "write"}
                ))

        # Track tool usage counts
        tool_name = tool.split(":")[0] if ":" in tool else tool
        self.current_session.tools_used[tool_name] = (
            self.current_session.tools_used.get(tool_name, 0) + 1
        )

    def _extract_intent(self, data: dict) -> None:
        """Extract user intent from prompts."""
        prompt = data.get("prompt", "")

        # Look for task markers
        if any(marker in prompt.lower() for marker in ["fix", "bug", "error"]):
            self.current_session.activities.append(Activity(
                type="task_start",
                description=f"Working on: {self._summarize_prompt(prompt)}",
                timestamp=datetime.utcnow()
            ))

    def _extract_outcomes(self, data: dict) -> None:
        """Extract outcomes from responses."""
        response = data.get("response", "")

        # Look for completion indicators
        completion_patterns = [
            r"(fixed|resolved|completed|done|finished)",
            r"(implemented|added|created|updated)",
        ]

        for pattern in completion_patterns:
            if re.search(pattern, response.lower()):
                self.current_session.activities.append(Activity(
                    type="completion",
                    description=self._extract_completion(response),
                    timestamp=datetime.utcnow()
                ))
                break

        # Look for blockers
        blocker_patterns = [
            r"(blocked|can't|unable to|need access|waiting for)",
            r"(error|failed|issue|problem)",
        ]

        for pattern in blocker_patterns:
            if re.search(pattern, response.lower()):
                self.current_session.blockers.append(
                    self._extract_blocker(response)
                )
                break

    async def _finalize_session(self, data: dict) -> HookResult:
        """Generate and store session summary."""
        if not self.current_session:
            return HookResult(action="continue")

        self.current_session.end_time = datetime.utcnow()
        self.current_session.duration = (
            self.current_session.end_time - self.current_session.start_time
        )

        # Generate summary
        summary = await self.summarizer.summarize(self.current_session)

        # Store summary
        await self.storage.store(summary)

        # Reset tracking
        session = self.current_session
        self.current_session = None

        return HookResult(
            action="continue",
            user_message=self._format_user_message(summary),
            user_message_level="info"
        )

    def _format_user_message(self, summary: SessionSummary) -> str:
        """Format summary for user display."""
        duration = summary.duration.total_seconds() / 60

        msg = [f"📋 Session Summary ({duration:.0f} min)", ""]

        if summary.completed:
            msg.append("**Completed:**")
            for item in summary.completed[:5]:
                msg.append(f"- {item}")
            msg.append("")

        if summary.in_progress:
            msg.append("**In Progress:**")
            for item in summary.in_progress[:3]:
                msg.append(f"- {item}")
            msg.append("")

        if summary.blockers:
            msg.append("**Blockers:**")
            for item in summary.blockers[:3]:
                msg.append(f"- ⚠️ {item}")
            msg.append("")

        if summary.files_changed:
            msg.append(f"**Files Changed:** {len(summary.files_changed)} files")

        return "\n".join(msg)


class WorkSummarizer:
    """Summarize session activity into standup format."""

    async def summarize(self, session: SessionActivity) -> SessionSummary:
        """Generate structured summary from activities."""

        # Group activities by type
        completions = [a for a in session.activities if a.type == "completion"]
        in_progress = [a for a in session.activities if a.type == "task_start"]

        # Deduplicate and clean
        completed_items = list(set(c.description for c in completions))
        in_progress_items = list(set(
            p.description for p in in_progress
            if p.description not in completed_items
        ))

        return SessionSummary(
            session_id=session.session_id,
            date=session.start_time.date(),
            duration=session.duration,
            completed=completed_items,
            in_progress=in_progress_items,
            blockers=session.blockers,
            files_changed=list(session.files_changed),
            tools_used=session.tools_used,
            metrics=SessionMetrics(
                duration_minutes=session.duration.total_seconds() / 60,
                files_changed=len(session.files_changed),
                tool_calls=sum(session.tools_used.values())
            )
        )


class StandupStorage:
    """Store standup summaries."""

    async def store(self, summary: SessionSummary) -> None:
        """Store summary to configured backend."""
        if self.config.type == "file":
            await self._store_file(summary)
        elif self.config.type == "notion":
            await self._store_notion(summary)
        elif self.config.type == "slack":
            await self._post_slack(summary)

    async def _store_file(self, summary: SessionSummary) -> None:
        """Store as markdown file."""
        date_str = summary.date.isoformat()
        path = Path(self.config.path).expanduser() / f"{date_str}.md"

        # Append to daily file
        content = self._format_markdown(summary)

        async with aiofiles.open(path, "a") as f:
            await f.write(content + "\n\n---\n\n")

    def _format_markdown(self, summary: SessionSummary) -> str:
        """Format summary as markdown."""
        lines = [
            f"## Session: {summary.session_id[:8]}",
            f"**Time:** {summary.duration.total_seconds() / 60:.0f} minutes",
            "",
        ]

        if summary.completed:
            lines.append("### Completed")
            for item in summary.completed:
                lines.append(f"- {item}")
            lines.append("")

        if summary.in_progress:
            lines.append("### In Progress")
            for item in summary.in_progress:
                lines.append(f"- {item}")
            lines.append("")

        if summary.blockers:
            lines.append("### Blockers")
            for item in summary.blockers:
                lines.append(f"- ⚠️ {item}")
            lines.append("")

        if summary.files_changed:
            lines.append("### Files Changed")
            for f in summary.files_changed[:10]:
                lines.append(f"- `{f}`")

        return "\n".join(lines)

    async def get_daily_rollup(self, date: date) -> list[SessionSummary]:
        """Get all summaries for a date."""
        path = Path(self.config.path).expanduser() / f"{date.isoformat()}.md"
        if not path.exists():
            return []
        # Parse and return summaries
        ...
```

---

## Summary Formats

### Structured Format (Default)

```markdown
## Session: abc123
**Time:** 45 minutes

### Completed
- Fixed payment timeout bug in gateway.py
- Added retry logic with exponential backoff
- Updated tests (3 new test cases)

### In Progress
- Refactoring error handling

### Files Changed
- `src/payments/gateway.py`
- `src/payments/retry.py`
- `tests/test_gateway.py`
```

### Narrative Format

```markdown
## Session Summary

Spent 45 minutes working on the payment timeout bug. Fixed the issue
by adding exponential backoff retry logic to the gateway. Added 3 new
test cases to cover the retry scenarios. Started refactoring the error
handling in the API routes but didn't complete it.

Changed 3 files in the payments module.
```

### Bullet Format

```markdown
- [45 min] Payment timeout bug
  - ✅ Fixed gateway.py timeout
  - ✅ Added retry logic
  - ✅ Added tests
  - 🔄 Error handling refactor (in progress)
```

---

## Daily Rollup

```python
class DailyRollupGenerator:
    """Generate daily standup from session summaries."""

    async def generate(self, date: date) -> DailyRollup:
        """Combine session summaries into daily rollup."""
        summaries = await self.storage.get_daily_rollup(date)

        all_completed = []
        all_in_progress = []
        all_blockers = []
        all_files = set()
        total_duration = timedelta()

        for summary in summaries:
            all_completed.extend(summary.completed)
            all_in_progress.extend(summary.in_progress)
            all_blockers.extend(summary.blockers)
            all_files.update(summary.files_changed)
            total_duration += summary.duration

        # Remove duplicates, prioritize completions
        in_progress_final = [
            item for item in set(all_in_progress)
            if item not in all_completed
        ]

        return DailyRollup(
            date=date,
            sessions=len(summaries),
            total_duration=total_duration,
            completed=list(set(all_completed)),
            in_progress=in_progress_final,
            blockers=list(set(all_blockers)),
            files_changed=list(all_files)
        )
```

---

## Examples

### Example 1: Bug Fix Session

```python
# Session activities:
# 1. User: "Fix the payment timeout bug"
# 2. Read files, found issue
# 3. Modified gateway.py
# 4. Added retry logic
# 5. Ran tests, fixed failures
# 6. Session end

# Generated summary:
"""
📋 Session Summary (32 min)

**Completed:**
- Fixed payment timeout bug in gateway.py
- Implemented exponential backoff retry
- Added 3 test cases for retry scenarios

**Files Changed:** 3 files
- src/payments/gateway.py
- src/payments/retry.py
- tests/test_gateway.py

Logged to: ~/.amplifier/standups/2024-01-15.md
"""
```

### Example 2: Session with Blockers

```python
# Session had issues:

# Generated summary:
"""
📋 Session Summary (25 min)

**In Progress:**
- Updating database schema for user profiles

**Blockers:**
- ⚠️ Need DBA approval for migration
- ⚠️ Missing access to production database

**Files Changed:** 1 file
- migrations/0042_user_profile.py

Logged to: ~/.amplifier/standups/2024-01-15.md
"""
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `capture.*` | bool | true | What to capture |
| `summary.format` | string | "structured" | Summary format |
| `summary.max_length` | int | 500 | Max summary length |
| `storage.type` | string | "file" | Storage backend |
| `storage.daily_rollup` | bool | true | Combine into daily |
| `integration.auto_post` | bool | false | Auto-post to Slack |

---

## Security Considerations

- Excludes sensitive file patterns by default
- Can anonymize paths if needed
- No code content stored, only summaries
- Local storage by default

---

## Dependencies

### Required
- `aiofiles` - File operations

### Optional
- `slack_sdk` - Slack integration
- `notion-client` - Notion integration

---

## Open Questions

1. **AI summarization**: Use LLM to generate better summaries?
2. **Team aggregation**: Combine team standups into digest?
3. **Metrics tracking**: Track productivity trends over time?
4. **Smart grouping**: Better task grouping algorithms?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
