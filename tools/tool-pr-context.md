# tool-pr-context

> **Priority**: P0 (Foundation)
> **Status**: Draft
> **Module**: `amplifier-module-tool-pr-context`

## Overview

Fetches comprehensive context about pull requests including the PR itself, all comments, linked issues, CI status, related PRs, and relevant code context. Provides everything needed to understand, review, or continue work on a PR.

### Value Proposition

| Without | With |
|---------|------|
| Open GitHub, read PR, click through to issues, check CI, find related PRs | Single call returns complete PR context |
| Context switching between browser and terminal | Full context in your workflow |
| Missing context from linked issues and prior discussions | Complete picture with cross-references |
| Manual correlation of PR with tickets | Automatic JIRA/GitHub issue linkage |

### Use Cases

1. **PR review**: Get full context before reviewing code
2. **Continue work**: Understand where a PR left off, what feedback exists
3. **Impact analysis**: See what other PRs touch related code
4. **Status check**: Quick summary of PR state, blockers, approvals
5. **Historical lookup**: "What was in PR #123 and why?"

---

## Contract

### Tool Definition

```python
TOOL_DEFINITION = {
    "name": "pr_context",
    "description": """
    Fetches comprehensive context about a pull request including:
    - PR details (title, description, author, status, branch info)
    - All comments and review threads
    - Linked issues (JIRA, GitHub Issues)
    - CI/CD status and logs
    - Related PRs (same files, same branch, dependencies)
    - Code diff with surrounding context

    Use when you need to understand, review, or work on a PR.
    """,
    "parameters": {
        "type": "object",
        "properties": {
            "pr_identifier": {
                "type": "string",
                "description": "PR number, URL, or branch name"
            },
            "repo": {
                "type": "string",
                "description": "Repository (owner/repo). Defaults to current repo."
            },
            "include": {
                "type": "array",
                "items": {
                    "type": "string",
                    "enum": ["comments", "reviews", "ci", "linked_issues", "related_prs", "diff", "files"]
                },
                "default": ["comments", "reviews", "ci", "linked_issues", "diff"],
                "description": "What context to include"
            },
            "diff_context_lines": {
                "type": "integer",
                "default": 5,
                "description": "Lines of context around diff hunks"
            }
        },
        "required": ["pr_identifier"]
    }
}
```

### Input Schema

```python
@dataclass
class PRContextInput:
    pr_identifier: str                  # PR number, URL, or branch name
    repo: str | None = None             # owner/repo, defaults to current
    include: list[str] = field(default_factory=lambda: [
        "comments", "reviews", "ci", "linked_issues", "diff"
    ])
    diff_context_lines: int = 5
```

### Output Schema

```python
@dataclass
class PRContext:
    # Core PR info
    pr: PRDetails

    # Included based on 'include' parameter
    comments: list[Comment] | None
    reviews: list[Review] | None
    ci_status: CIStatus | None
    linked_issues: list[LinkedIssue] | None
    related_prs: list[RelatedPR] | None
    diff: Diff | None
    files: list[FileChange] | None

    # Computed summaries
    summary: PRSummary

@dataclass
class PRDetails:
    number: int
    title: str
    description: str
    author: str
    state: str                          # open | closed | merged
    draft: bool
    base_branch: str
    head_branch: str
    created_at: datetime
    updated_at: datetime
    merged_at: datetime | None
    url: str

    # Approval status
    approvals: int
    changes_requested: int
    pending_reviewers: list[str]

@dataclass
class Comment:
    id: str
    author: str
    body: str
    created_at: datetime
    updated_at: datetime | None
    reply_to: str | None                # Parent comment ID if reply
    reactions: dict[str, int]           # emoji -> count

    # For line comments
    file_path: str | None
    line_number: int | None
    diff_hunk: str | None

@dataclass
class Review:
    id: str
    author: str
    state: str                          # approved | changes_requested | commented
    body: str | None
    submitted_at: datetime
    comments: list[Comment]             # Review-specific comments

@dataclass
class CIStatus:
    overall: str                        # success | failure | pending | neutral
    checks: list[CICheck]

@dataclass
class CICheck:
    name: str
    status: str                         # success | failure | pending | skipped
    conclusion: str | None
    url: str | None
    started_at: datetime | None
    completed_at: datetime | None
    summary: str | None                 # Brief failure reason if failed

@dataclass
class LinkedIssue:
    source: str                         # "github" | "jira"
    key: str                            # Issue number or JIRA key
    title: str
    status: str
    url: str
    relationship: str                   # "closes" | "references" | "related"

@dataclass
class RelatedPR:
    number: int
    title: str
    author: str
    state: str
    relationship: str                   # "same_files" | "same_branch" | "dependency" | "dependent"
    overlap_files: list[str] | None     # Files touched by both PRs

@dataclass
class Diff:
    stats: DiffStats
    files: list[FileDiff]

@dataclass
class FileDiff:
    path: str
    status: str                         # added | modified | deleted | renamed
    additions: int
    deletions: int
    hunks: list[DiffHunk]

@dataclass
class PRSummary:
    """Computed summary for quick understanding."""
    one_liner: str                      # "Add retry logic to payment gateway"
    change_type: str                    # feature | bugfix | refactor | docs | test
    risk_level: str                     # low | medium | high
    blockers: list[str]                 # What's blocking merge
    ready_to_merge: bool
    key_changes: list[str]              # Bullet points of main changes
    outstanding_threads: int            # Unresolved comment threads
```

### Events Emitted

| Event | When | Data |
|-------|------|------|
| `tool:pr_context:start` | Fetch begins | pr_identifier, repo, include |
| `tool:pr_context:github_fetch` | GitHub API call | endpoint, duration_ms |
| `tool:pr_context:jira_fetch` | JIRA API call | issue_key, duration_ms |
| `tool:pr_context:complete` | Fetch done | pr_number, sections_fetched |
| `tool:pr_context:error` | Fetch failed | error_type, message |

---

## Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                       tool-pr-context                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │  Identifier │───▶│   Context   │───▶│   Summary   │         │
│  │   Resolver  │    │   Fetcher   │    │  Generator  │         │
│  └─────────────┘    └──────┬──────┘    └─────────────┘         │
│                            │                                    │
│         ┌──────────────────┼──────────────────┐                │
│         ▼                  ▼                  ▼                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │   GitHub    │    │    JIRA     │    │   Local     │         │
│  │   Client    │    │   Client    │    │    Git      │         │
│  └─────────────┘    └─────────────┘    └─────────────┘         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Internal Components

#### 1. Identifier Resolver

```python
class IdentifierResolver:
    """
    Resolves various PR identifier formats to canonical form.

    Supports:
    - PR number: "123" or "#123"
    - URL: "https://github.com/owner/repo/pull/123"
    - Branch: "feature/add-retry" (finds PR for branch)
    - Current: "current" or "." (PR for current branch)
    """

    async def resolve(self, identifier: str, repo: str | None) -> ResolvedPR:
        # Detect format
        if identifier in ("current", "."):
            branch = await self._get_current_branch()
            return await self._find_pr_for_branch(branch, repo)

        if identifier.startswith("http"):
            return self._parse_url(identifier)

        if identifier.replace("#", "").isdigit():
            pr_number = int(identifier.replace("#", ""))
            return ResolvedPR(number=pr_number, repo=repo or await self._detect_repo())

        # Assume branch name
        return await self._find_pr_for_branch(identifier, repo)
```

#### 2. Context Fetcher

```python
class ContextFetcher:
    """
    Orchestrates fetching from multiple sources based on 'include' param.
    """

    async def fetch(self, pr: ResolvedPR, include: list[str]) -> PRContext:
        # Always fetch core PR details
        pr_details = await self.github.get_pr(pr.repo, pr.number)

        # Parallel fetch of included sections
        tasks = {}

        if "comments" in include:
            tasks["comments"] = self._fetch_comments(pr)
        if "reviews" in include:
            tasks["reviews"] = self._fetch_reviews(pr)
        if "ci" in include:
            tasks["ci"] = self._fetch_ci_status(pr)
        if "linked_issues" in include:
            tasks["linked_issues"] = self._fetch_linked_issues(pr, pr_details)
        if "related_prs" in include:
            tasks["related_prs"] = self._fetch_related_prs(pr)
        if "diff" in include:
            tasks["diff"] = self._fetch_diff(pr)
        if "files" in include:
            tasks["files"] = self._fetch_files(pr)

        results = await asyncio.gather(*tasks.values(), return_exceptions=True)

        return PRContext(
            pr=pr_details,
            **dict(zip(tasks.keys(), results))
        )

    async def _fetch_linked_issues(self, pr: ResolvedPR, details: PRDetails) -> list[LinkedIssue]:
        """Extract and fetch linked issues from PR description and commits."""
        # Parse PR description for issue references
        github_refs = self._parse_github_refs(details.description)
        jira_refs = self._parse_jira_refs(details.description)

        # Also check commit messages
        commits = await self.github.get_pr_commits(pr.repo, pr.number)
        for commit in commits:
            github_refs.extend(self._parse_github_refs(commit.message))
            jira_refs.extend(self._parse_jira_refs(commit.message))

        # Fetch issue details
        issues = []

        for ref in set(github_refs):
            issue = await self.github.get_issue(pr.repo, ref.number)
            issues.append(LinkedIssue(
                source="github",
                key=f"#{ref.number}",
                title=issue.title,
                status=issue.state,
                url=issue.url,
                relationship=ref.relationship
            ))

        for ref in set(jira_refs):
            if self.jira:
                issue = await self.jira.get_issue(ref.key)
                issues.append(LinkedIssue(
                    source="jira",
                    key=ref.key,
                    title=issue.summary,
                    status=issue.status,
                    url=issue.url,
                    relationship=ref.relationship
                ))

        return issues
```

#### 3. Summary Generator

```python
class SummaryGenerator:
    """
    Generates human-readable summaries from raw PR context.
    """

    def generate(self, context: PRContext) -> PRSummary:
        return PRSummary(
            one_liner=self._generate_one_liner(context),
            change_type=self._classify_change_type(context),
            risk_level=self._assess_risk(context),
            blockers=self._identify_blockers(context),
            ready_to_merge=self._check_merge_readiness(context),
            key_changes=self._extract_key_changes(context),
            outstanding_threads=self._count_unresolved_threads(context)
        )

    def _identify_blockers(self, context: PRContext) -> list[str]:
        blockers = []

        # CI failures
        if context.ci_status and context.ci_status.overall == "failure":
            failed = [c.name for c in context.ci_status.checks if c.status == "failure"]
            blockers.append(f"CI failing: {', '.join(failed)}")

        # Changes requested
        if context.pr.changes_requested > 0:
            blockers.append(f"{context.pr.changes_requested} reviewer(s) requested changes")

        # Unresolved threads
        unresolved = self._count_unresolved_threads(context)
        if unresolved > 0:
            blockers.append(f"{unresolved} unresolved comment thread(s)")

        # Missing approvals
        if context.pr.approvals < 1:
            blockers.append("No approvals yet")

        # Draft status
        if context.pr.draft:
            blockers.append("PR is still in draft")

        return blockers

    def _assess_risk(self, context: PRContext) -> str:
        """Assess risk level based on changes."""
        risk_score = 0

        if context.diff:
            # Large changes
            if context.diff.stats.total_changes > 500:
                risk_score += 2
            elif context.diff.stats.total_changes > 200:
                risk_score += 1

            # Many files
            if len(context.diff.files) > 20:
                risk_score += 1

            # Critical file patterns
            critical_patterns = ["migration", "auth", "payment", "security", "config"]
            for file in context.diff.files:
                if any(p in file.path.lower() for p in critical_patterns):
                    risk_score += 1
                    break

        if risk_score >= 3:
            return "high"
        elif risk_score >= 1:
            return "medium"
        return "low"
```

---

## Configuration

```yaml
tool-pr-context:
  # GitHub configuration
  github:
    auth_token: "${GITHUB_TOKEN}"
    api_url: "https://api.github.com"      # or GitHub Enterprise URL

    # Rate limiting
    rate_limit_buffer: 100                  # Keep this many requests in reserve

    # Caching
    cache_ttl_seconds: 60                   # Cache PR data briefly

  # JIRA configuration (optional)
  jira:
    enabled: true
    base_url: "${JIRA_URL}"
    auth_token: "${JIRA_TOKEN}"

    # Issue key patterns to recognize
    key_patterns:
      - "[A-Z]{2,10}-\\d+"                  # Standard JIRA key format

  # Default behavior
  defaults:
    include:
      - comments
      - reviews
      - ci
      - linked_issues
      - diff
    diff_context_lines: 5

  # Related PR detection
  related_prs:
    # How to find related PRs
    strategies:
      - same_files                          # PRs touching same files
      - same_branch_prefix                  # feature/X-* branches
      - linked_issues                       # PRs linked to same issues

    # Limits
    max_related: 5
    lookback_days: 30                       # Only PRs from last 30 days

  # Summary generation
  summary:
    # Risk assessment thresholds
    high_risk_lines: 500
    medium_risk_lines: 200
    critical_paths:
      - "migrations/"
      - "auth/"
      - "security/"
      - "payments/"
```

---

## Examples

### Example 1: Current Branch PR Context

```python
# Input
{
    "pr_identifier": "current"
}

# Output
{
    "pr": {
        "number": 456,
        "title": "Add retry logic to payment gateway",
        "description": "Implements exponential backoff retry for payment API calls.\n\nCloses JIRA-4523\nRelated to #451",
        "author": "alice",
        "state": "open",
        "draft": false,
        "base_branch": "main",
        "head_branch": "feature/payment-retry",
        "approvals": 1,
        "changes_requested": 0,
        "pending_reviewers": ["bob", "carol"]
    },
    "comments": [
        {
            "id": "c1",
            "author": "bob",
            "body": "Should we add a circuit breaker too?",
            "created_at": "2024-01-15T10:30:00Z",
            "file_path": "src/payments/gateway.py",
            "line_number": 42
        }
    ],
    "reviews": [
        {
            "id": "r1",
            "author": "carol",
            "state": "approved",
            "body": "LGTM! Nice use of tenacity.",
            "submitted_at": "2024-01-15T11:00:00Z"
        }
    ],
    "ci_status": {
        "overall": "success",
        "checks": [
            {"name": "tests", "status": "success"},
            {"name": "lint", "status": "success"},
            {"name": "security-scan", "status": "success"}
        ]
    },
    "linked_issues": [
        {
            "source": "jira",
            "key": "JIRA-4523",
            "title": "Payment failures during peak hours",
            "status": "In Progress",
            "relationship": "closes"
        },
        {
            "source": "github",
            "key": "#451",
            "title": "Improve payment reliability",
            "status": "open",
            "relationship": "related"
        }
    ],
    "summary": {
        "one_liner": "Adds retry logic with exponential backoff to payment gateway",
        "change_type": "feature",
        "risk_level": "medium",
        "blockers": ["1 unresolved comment thread(s)"],
        "ready_to_merge": false,
        "key_changes": [
            "New retry decorator in payments/retry.py",
            "Applied to PaymentGateway.process_payment()",
            "Config for max retries and backoff"
        ],
        "outstanding_threads": 1
    }
}
```

### Example 2: PR Status Quick Check

```python
# Input
{
    "pr_identifier": "789",
    "include": ["ci", "reviews"]
}

# Output (minimal, fast)
{
    "pr": {
        "number": 789,
        "title": "Update dependencies",
        "state": "open",
        "approvals": 0,
        "changes_requested": 1,
        "pending_reviewers": ["security-team"]
    },
    "reviews": [
        {
            "author": "security-bot",
            "state": "changes_requested",
            "body": "Found 2 vulnerabilities in updated packages"
        }
    ],
    "ci_status": {
        "overall": "failure",
        "checks": [
            {"name": "security-scan", "status": "failure", "summary": "CVE-2024-1234 in lodash"}
        ]
    },
    "summary": {
        "blockers": [
            "CI failing: security-scan",
            "1 reviewer(s) requested changes"
        ],
        "ready_to_merge": false
    }
}
```

---

## Security Considerations

### Data Access

- **GitHub API**: Uses provided token, respects repository permissions
- **JIRA API**: Uses provided token, respects project permissions
- **Local git**: Reads from current repository only

### Sensitive Data

- **PR descriptions**: May contain sensitive information
- **Comments**: May contain sensitive discussions
- **Diff content**: May contain sensitive code

### Permissions Model

```python
REQUIRED_CAPABILITIES = [
    "network:github",       # GitHub API access
]

OPTIONAL_CAPABILITIES = [
    "network:jira",         # JIRA API access (if configured)
    "filesystem:read",      # Local git operations
]
```

### Risk Mitigations

| Risk | Mitigation |
|------|------------|
| Token exposure | Tokens never logged or returned in output |
| Unauthorized repo access | API calls fail if token lacks permissions |
| Rate limiting | Built-in rate limit awareness, graceful degradation |

---

## Testing Strategy

### Unit Tests

```python
def test_identifier_resolver_handles_pr_number():
    resolver = IdentifierResolver()
    result = await resolver.resolve("123", "owner/repo")
    assert result.number == 123
    assert result.repo == "owner/repo"

def test_identifier_resolver_handles_url():
    resolver = IdentifierResolver()
    result = await resolver.resolve(
        "https://github.com/owner/repo/pull/456",
        None
    )
    assert result.number == 456
    assert result.repo == "owner/repo"

def test_summary_identifies_blockers():
    context = PRContext(
        pr=PRDetails(approvals=0, changes_requested=1, ...),
        ci_status=CIStatus(overall="failure", ...),
        ...
    )
    summary = SummaryGenerator().generate(context)
    assert len(summary.blockers) >= 2
    assert not summary.ready_to_merge
```

### Integration Tests

```python
async def test_fetches_real_pr_context(mock_github):
    """Test against mock GitHub API."""
    mock_github.setup_pr(number=123, title="Test PR", ...)

    tool = PRContextTool(config)
    result = await tool.execute({"pr_identifier": "123"})

    assert result.pr.number == 123
    assert result.pr.title == "Test PR"

async def test_handles_linked_jira_issues(mock_github, mock_jira):
    """Test JIRA integration."""
    mock_github.setup_pr(description="Fixes PROJ-123")
    mock_jira.setup_issue(key="PROJ-123", summary="Bug")

    result = await tool.execute({"pr_identifier": "1", "include": ["linked_issues"]})

    assert len(result.linked_issues) == 1
    assert result.linked_issues[0].key == "PROJ-123"
```

---

## Dependencies

### Required

- `httpx` - Async HTTP client for API calls
- `PyGithub` or direct API - GitHub operations

### Optional

- `jira` - JIRA API client (if JIRA enabled)
- `gitpython` - Local git operations

---

## Open Questions

1. **Diff size limits**: Should we truncate large diffs?
   - Option A: Always return full diff (complete but potentially huge)
   - Option B: Truncate with indication (smaller but lossy)
   - Option C: Return stats only for large diffs, full for small

2. **Comment threading**: How to represent nested comment threads?
   - Option A: Flat list with reply_to references
   - Option B: Nested structure
   - Option C: Both (flat + thread groupings)

3. **CI log access**: Should we fetch full CI logs or just status?
   - Full logs useful for debugging but can be very large
   - Could offer as separate follow-up query

4. **Webhook vs polling**: For real-time updates, should we support webhooks?
   - Currently: Point-in-time fetch
   - Future: Could add streaming updates

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
