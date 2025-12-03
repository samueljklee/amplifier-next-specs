# tool-slack-search

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-tool-slack-search`

## Overview

Search team Slack conversations to find decisions, context, and tribal knowledge. Surfaces relevant discussions that explain why code was written a certain way or captures historical context not found in documentation.

### Value Proposition

| Without | With |
|---------|------|
| "Why did we do it this way?" → Ask around, hope someone remembers | Search Slack, find the original discussion |
| Context lost when people leave | Institutional knowledge preserved and searchable |
| Decisions made in DMs never documented | Surface relevant conversations (with permissions) |
| Hours searching Slack's clunky UI | Semantic search finds relevant threads instantly |

### Use Cases

1. **Decision archaeology**: "Why did we choose Redis over Memcached?"
2. **Context recovery**: "What was discussed about the auth refactor?"
3. **Incident history**: "What happened during the outage on Jan 15?"
4. **Expert finding**: "Who knows about the payment gateway?"
5. **Meeting follow-up**: "What did we decide in the sprint planning?"

---

## Contract

### Tool Definition

```python
TOOL_DEFINITION = {
    "name": "slack_search",
    "description": """
    Search Slack conversations for context and decisions.

    Search across:
    - Public channels you have access to
    - Private channels you're a member of
    - Direct messages (your own only)
    - Threads and replies

    Use for finding decisions, context, and institutional knowledge.
    """,
    "parameters": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "Search query (natural language or keywords)"
            },
            "channels": {
                "type": "array",
                "items": {"type": "string"},
                "description": "Limit search to specific channels"
            },
            "from_user": {
                "type": "string",
                "description": "Filter by sender"
            },
            "date_range": {
                "type": "object",
                "properties": {
                    "after": {"type": "string", "format": "date"},
                    "before": {"type": "string", "format": "date"}
                },
                "description": "Filter by date range"
            },
            "has": {
                "type": "array",
                "items": {"type": "string", "enum": ["link", "file", "reaction", "thread"]},
                "description": "Filter by content type"
            },
            "include_threads": {
                "type": "boolean",
                "default": true,
                "description": "Include thread replies in search"
            },
            "max_results": {
                "type": "integer",
                "default": 20,
                "description": "Maximum results to return"
            }
        },
        "required": ["query"]
    }
}
```

### Output Schema

```python
@dataclass
class SlackSearchOutput:
    results: list[SlackMessage]
    total_count: int
    query_interpretation: str
    channels_searched: list[str]

@dataclass
class SlackMessage:
    # Message identity
    id: str                             # Slack message ID
    permalink: str                      # Direct link to message

    # Content
    text: str                           # Message text (markdown)
    user: SlackUser
    channel: SlackChannel
    timestamp: datetime

    # Thread context
    thread_ts: str | None               # Thread parent timestamp
    is_thread_parent: bool
    reply_count: int
    thread_participants: list[str]      # Users in thread

    # Rich content
    attachments: list[Attachment]
    files: list[SlackFile]
    reactions: list[Reaction]
    links: list[str]

    # Context
    thread_messages: list[SlackMessage] | None  # If is_thread_parent
    relevance_score: float

@dataclass
class SlackUser:
    id: str
    name: str
    display_name: str
    avatar_url: str | None

@dataclass
class SlackChannel:
    id: str
    name: str
    is_private: bool

@dataclass
class Attachment:
    title: str | None
    text: str | None
    type: str                           # message, file, etc.

@dataclass
class Reaction:
    name: str                           # emoji name
    count: int
    users: list[str]                    # User IDs who reacted
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      tool-slack-search                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │   Query     │───▶│   Slack     │───▶│   Result    │         │
│  │  Builder    │    │    API      │    │  Enricher   │         │
│  └─────────────┘    └─────────────┘    └─────────────┘         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Query Builder

```python
class SlackQueryBuilder:
    """Build Slack search queries from natural language."""

    def build(self, input: SlackSearchInput) -> str:
        """Convert input to Slack search query syntax."""
        parts = [input.query]

        if input.channels:
            channel_part = " ".join(f"in:{c}" for c in input.channels)
            parts.append(channel_part)

        if input.from_user:
            parts.append(f"from:{input.from_user}")

        if input.date_range:
            if input.date_range.after:
                parts.append(f"after:{input.date_range.after}")
            if input.date_range.before:
                parts.append(f"before:{input.date_range.before}")

        if input.has:
            for h in input.has:
                parts.append(f"has:{h}")

        return " ".join(parts)
```

### Result Enricher

```python
class ResultEnricher:
    """Enrich search results with thread context."""

    async def enrich(
        self,
        messages: list[dict],
        include_threads: bool
    ) -> list[SlackMessage]:
        enriched = []

        for msg in messages:
            slack_msg = self._parse_message(msg)

            # Fetch thread if parent and include_threads
            if include_threads and slack_msg.is_thread_parent and slack_msg.reply_count > 0:
                slack_msg.thread_messages = await self._fetch_thread(
                    slack_msg.channel.id,
                    slack_msg.thread_ts
                )

            enriched.append(slack_msg)

        return enriched

    async def _fetch_thread(
        self,
        channel_id: str,
        thread_ts: str
    ) -> list[SlackMessage]:
        """Fetch all replies in a thread."""
        replies = await self.slack.conversations_replies(
            channel=channel_id,
            ts=thread_ts
        )
        return [self._parse_message(r) for r in replies["messages"][1:]]  # Skip parent
```

---

## Configuration

```yaml
tool-slack-search:
  # Authentication
  auth:
    # User token with search:read scope
    user_token: "${SLACK_USER_TOKEN}"

    # Or bot token (more limited)
    # bot_token: "${SLACK_BOT_TOKEN}"

  # Search settings
  search:
    # Default channels to search (if none specified)
    default_channels:
      - general
      - engineering
      - incidents

    # Channels to never search
    excluded_channels:
      - random
      - social

    # Include archived channels
    include_archived: false

  # Results
  results:
    max_results: 50
    include_threads: true
    max_thread_messages: 20

  # Rate limiting
  rate_limit:
    requests_per_minute: 20
```

---

## Examples

### Example 1: Decision Search

```python
# Input
{
    "query": "Redis vs Memcached decision",
    "channels": ["engineering", "architecture"],
    "has": ["thread"],
    "max_results": 5
}

# Output
{
    "results": [
        {
            "id": "msg123",
            "permalink": "https://workspace.slack.com/archives/C123/p1234567890",
            "text": "We need to decide on caching. Options are Redis and Memcached. Let's discuss pros/cons.",
            "user": {"name": "alice", "display_name": "Alice Chen"},
            "channel": {"name": "architecture", "is_private": false},
            "timestamp": "2023-11-15T10:30:00Z",
            "is_thread_parent": true,
            "reply_count": 12,
            "thread_messages": [
                {
                    "text": "Redis gives us persistence and pub/sub which we'll need for the notification system",
                    "user": {"name": "bob"}
                },
                {
                    "text": "Agreed. Let's go with Redis. I'll document this in the ADR.",
                    "user": {"name": "alice"}
                }
            ],
            "relevance_score": 0.95
        }
    ],
    "total_count": 3,
    "query_interpretation": "Searching for 'Redis vs Memcached decision' in #architecture, #engineering"
}
```

### Example 2: Incident Context

```python
# Input
{
    "query": "outage payment gateway",
    "channels": ["incidents"],
    "date_range": {"after": "2024-01-01", "before": "2024-01-31"}
}

# Output
{
    "results": [
        {
            "text": ":rotating_light: INCIDENT: Payment gateway timeout errors spiking",
            "channel": {"name": "incidents"},
            "timestamp": "2024-01-15T03:45:00Z",
            "reply_count": 45,
            "thread_messages": [
                {"text": "Root cause: Database connection pool exhaustion"},
                {"text": "Fix deployed. Monitoring."},
                {"text": "Resolved. Postmortem scheduled for tomorrow."}
            ],
            "reactions": [
                {"name": "eyes", "count": 8},
                {"name": "white_check_mark", "count": 5}
            ]
        }
    ]
}
```

---

## Security Considerations

### Data Access
- Only searches channels user has access to
- User token required (more access than bot token)
- DMs only searchable by the user themselves

### Sensitive Data
- Messages may contain sensitive information
- Results should be treated with same confidentiality as Slack itself

### Permissions
```python
REQUIRED_CAPABILITIES = [
    "network:slack",        # Slack API access
]

# Required Slack scopes
SLACK_SCOPES = [
    "search:read",          # Search messages
    "channels:read",        # List channels
    "groups:read",          # Private channels
    "im:read",              # Direct messages
    "users:read",           # User info
]
```

---

## Dependencies

### Required
- `slack-sdk` - Official Slack Python SDK

### Optional
- None

---

## Open Questions

1. **Bot vs User token**: User tokens have more access but require OAuth flow
2. **Caching**: Should we cache search results? For how long?
3. **Summarization**: Should we AI-summarize long threads?
4. **Privacy**: How to handle searches that might surface sensitive DMs?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
