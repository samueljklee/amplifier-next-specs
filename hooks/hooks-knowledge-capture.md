# hooks-knowledge-capture

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-hooks-knowledge-capture`

## Overview

Automatically captures learnings, decisions, and insights from AI sessions and stores them in a team knowledge base. Transforms ephemeral conversations into persistent organizational knowledge.

### Value Proposition

| Without | With |
|---------|------|
| Knowledge lost after session ends | Insights captured automatically |
| "We figured this out before..." | Searchable knowledge base |
| Tribal knowledge in people's heads | Documented and shared |
| Repeated problem-solving | Learn from past sessions |

---

## Contract

### Hook Configuration

```yaml
hooks:
  - module: hooks-knowledge-capture
    config:
      # What to capture
      capture:
        decisions: true                 # "Let's use X approach"
        learnings: true                 # "TIL...", "I learned..."
        solutions: true                 # Problem-solution pairs
        code_patterns: true             # Reusable code patterns
        gotchas: true                   # "Watch out for...", "Don't forget..."

      # Knowledge base storage
      storage:
        type: markdown                  # markdown | notion | confluence | custom
        path: "docs/knowledge-base/"
        organize_by: topic              # topic | date | project

      # Extraction settings
      extraction:
        min_significance: 0.7           # Confidence threshold
        require_confirmation: false     # Ask before saving
        deduplicate: true               # Skip if similar exists

      # Categories/tags
      categories:
        - architecture
        - debugging
        - performance
        - security
        - testing
        - deployment
```

### HookResult

```python
# Capture knowledge at session end
HookResult(
    action="inject_context",
    context_injection="",  # No injection needed
    suppress_output=True,
    user_message="Captured 3 learnings to knowledge base"
)

# Or ask for confirmation
HookResult(
    action="ask_user",
    approval_prompt="""
Knowledge capture found these insights:

1. **Decision**: Use exponential backoff for payment retries (max 3 attempts)
2. **Gotcha**: Payment gateway returns 200 even on soft failures - check response body
3. **Pattern**: Retry decorator pattern for idempotent operations

Save to knowledge base?
""",
    approval_options=["Save all", "Save selected", "Skip"]
)
```

---

## Architecture

```python
class KnowledgeCaptureHook:
    """Capture and store knowledge from sessions."""

    def __init__(self, config: KnowledgeCaptureConfig):
        self.config = config
        self.storage = self._create_storage(config.storage)
        self.extractor = KnowledgeExtractor(config.extraction)
        self.session_messages: list[dict] = []

    async def __call__(self, event: str, data: dict) -> HookResult:
        # Collect messages during session
        if event in ("prompt:submit", "prompt:complete"):
            self._collect_message(event, data)
            return HookResult(action="continue")

        # Extract knowledge at session end
        if event == "session:end":
            return await self._extract_and_save()

        return HookResult(action="continue")

    def _collect_message(self, event: str, data: dict) -> None:
        """Collect messages for later analysis."""
        if event == "prompt:submit":
            self.session_messages.append({
                "role": "user",
                "content": data.get("prompt", "")
            })
        elif event == "prompt:complete":
            self.session_messages.append({
                "role": "assistant",
                "content": data.get("response", "")
            })

    async def _extract_and_save(self) -> HookResult:
        """Extract knowledge and save to storage."""
        if not self.session_messages:
            return HookResult(action="continue")

        # Extract knowledge items
        items = await self.extractor.extract(self.session_messages)

        if not items:
            return HookResult(action="continue")

        # Deduplicate if configured
        if self.config.extraction.deduplicate:
            items = await self._deduplicate(items)

        if not items:
            return HookResult(action="continue")

        # Save or request confirmation
        if self.config.extraction.require_confirmation:
            return self._request_confirmation(items)
        else:
            await self._save_items(items)
            return HookResult(
                action="continue",
                user_message=f"Captured {len(items)} knowledge items"
            )


class KnowledgeExtractor:
    """Extract knowledge items from conversation."""

    # Patterns indicating knowledge
    DECISION_PATTERNS = [
        r"let's use",
        r"we('ll| will) go with",
        r"decided to",
        r"the approach (is|will be)",
        r"choosing .* because",
    ]

    LEARNING_PATTERNS = [
        r"(TIL|I learned|we learned)",
        r"turns out",
        r"discovered that",
        r"found out",
        r"realized",
    ]

    GOTCHA_PATTERNS = [
        r"watch out for",
        r"don't forget",
        r"gotcha",
        r"caveat",
        r"be careful",
        r"important to note",
        r"heads up",
    ]

    SOLUTION_PATTERNS = [
        r"(the |this )?(fix|solution) (is|was)",
        r"solved (it |this )by",
        r"fixed (it |this )by",
        r"the answer (is|was)",
    ]

    async def extract(self, messages: list[dict]) -> list[KnowledgeItem]:
        """Extract knowledge items from messages."""
        items = []

        # Concatenate conversation
        full_text = "\n".join(m["content"] for m in messages)

        # Pattern-based extraction
        items.extend(self._extract_by_patterns(messages))

        # LLM-based extraction for deeper insights
        items.extend(await self._extract_with_llm(messages))

        # Filter by significance
        items = [i for i in items if i.significance >= self.config.min_significance]

        return items

    def _extract_by_patterns(self, messages: list[dict]) -> list[KnowledgeItem]:
        """Extract using pattern matching."""
        items = []

        for message in messages:
            content = message["content"]

            # Check each category
            for pattern in self.DECISION_PATTERNS:
                matches = re.finditer(pattern, content, re.IGNORECASE)
                for match in matches:
                    context = self._get_context(content, match)
                    items.append(KnowledgeItem(
                        type="decision",
                        content=context,
                        significance=0.8
                    ))

            for pattern in self.GOTCHA_PATTERNS:
                matches = re.finditer(pattern, content, re.IGNORECASE)
                for match in matches:
                    context = self._get_context(content, match)
                    items.append(KnowledgeItem(
                        type="gotcha",
                        content=context,
                        significance=0.85
                    ))

        return items

    async def _extract_with_llm(self, messages: list[dict]) -> list[KnowledgeItem]:
        """Use LLM to extract deeper insights."""
        # Use a small, fast model for extraction
        prompt = f"""
Analyze this conversation and extract key knowledge items:

{self._format_conversation(messages)}

Extract:
1. Decisions made and their rationale
2. Learnings or discoveries
3. Solutions to problems encountered
4. Gotchas or caveats mentioned
5. Reusable patterns or approaches

Format as JSON array with: type, content, category, significance (0-1)
"""
        # Call LLM and parse response
        # ...
```

---

## Knowledge Item Schema

```python
@dataclass
class KnowledgeItem:
    type: str                           # decision | learning | gotcha | solution | pattern
    content: str                        # The knowledge content
    category: str | None                # architecture | debugging | etc.
    significance: float                 # 0-1 confidence score

    # Metadata
    source_session: str | None
    timestamp: datetime
    related_files: list[str]
    tags: list[str]

    # For solutions
    problem: str | None                 # What problem this solves
    solution: str | None                # How it solves it
```

---

## Storage Formats

### Markdown (Default)

```markdown
# Knowledge Base

## Decisions

### Use exponential backoff for payment retries
**Date**: 2024-01-15
**Category**: Architecture
**Context**: Payment gateway integration

Decided to implement exponential backoff with max 3 retries for payment processing.
Rationale: Gateway has intermittent timeouts during peak load.

**Related files**: src/payments/gateway.py

---

## Gotchas

### Payment gateway returns 200 on soft failures
**Date**: 2024-01-15
**Category**: Debugging

Watch out: The payment gateway returns HTTP 200 even when payment fails.
Must check `response.body.success` field, not just HTTP status.

---
```

### Notion Integration

```python
class NotionStorage:
    """Store knowledge in Notion database."""

    async def save(self, item: KnowledgeItem) -> None:
        await self.notion.pages.create(
            parent={"database_id": self.database_id},
            properties={
                "Title": {"title": [{"text": {"content": item.content[:100]}}]},
                "Type": {"select": {"name": item.type}},
                "Category": {"select": {"name": item.category}},
                "Date": {"date": {"start": item.timestamp.isoformat()}},
                "Tags": {"multi_select": [{"name": t} for t in item.tags]},
            },
            children=self._format_content_blocks(item)
        )
```

---

## Examples

### Example 1: Decision Capture

```python
# From session conversation:
# User: "Should we use Redis or Memcached for caching?"
# Assistant: "Given your need for persistence and pub/sub, let's use Redis..."

# Extracted:
{
    "type": "decision",
    "content": "Use Redis over Memcached for caching",
    "category": "architecture",
    "problem": "Choosing caching solution",
    "solution": "Redis - provides persistence and pub/sub needed for notifications",
    "significance": 0.9
}
```

### Example 2: Gotcha Capture

```python
# From session:
# Assistant: "...watch out for timezone handling. The API returns UTC but
# the database stores local time, so you need to convert..."

# Extracted:
{
    "type": "gotcha",
    "content": "API returns UTC, database stores local time - convert at boundary",
    "category": "debugging",
    "tags": ["timezone", "api", "database"],
    "significance": 0.85
}
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `capture.*` | bool | true | What to capture |
| `storage.type` | string | "markdown" | Storage backend |
| `storage.path` | string | "docs/kb/" | Path for markdown |
| `extraction.min_significance` | float | 0.7 | Confidence threshold |
| `extraction.require_confirmation` | bool | false | Ask before saving |
| `extraction.deduplicate` | bool | true | Skip duplicates |

---

## Security Considerations

- May capture sensitive information from sessions
- Storage should be access-controlled
- Consider PII filtering before storage

---

## Dependencies

### Required
- `aiofiles` - File operations

### Optional
- `notion-client` - Notion integration
- `atlassian-python-api` - Confluence integration

---

## Open Questions

1. **Confirmation flow**: Always ask, never ask, or smart selection?
2. **Cross-session**: Link related knowledge across sessions?
3. **Retrieval**: Should this hook also inject relevant knowledge?
4. **Quality**: How to ensure captured knowledge is accurate?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
