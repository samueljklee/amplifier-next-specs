# tool-git-advanced

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-tool-git-advanced`

## Overview

Advanced git operations beyond basic add/commit/push. Provides contextual blame, history analysis, branch management, and intelligent merge conflict assistance. Designed for complex codebase navigation and workflow automation.

### Value Proposition

| Without | With |
|---------|------|
| `git blame` shows who, not why | Contextual blame links to PR, ticket, and discussion |
| Manual branch cleanup | Intelligent stale branch detection and cleanup |
| Merge conflicts are painful | AI-assisted conflict resolution with context |
| History is hard to navigate | Semantic search through commit history |

### Use Cases

1. **Contextual blame**: "Who changed this and why?" with full context
2. **History search**: "When did we change the auth timeout?"
3. **Branch management**: Clean up stale branches, find related branches
4. **Conflict resolution**: Understand both sides of a conflict with context
5. **Bisect assistance**: Help find which commit introduced a bug
6. **Cherry-pick planning**: Identify commits to backport

---

## Contract

### Tool Definition

```python
TOOL_DEFINITION = {
    "name": "git_advanced",
    "description": """
    Advanced git operations with contextual information.

    Operations:
    - blame: Contextual blame with PR/ticket links
    - history: Search commit history semantically
    - branches: List, analyze, and manage branches
    - conflicts: Analyze merge conflicts with context
    - bisect: Assisted bisection to find bug introduction
    - cherry_pick_plan: Plan cherry-picks for backporting
    - diff_analysis: Analyze changes between refs

    Use for understanding code history and managing complex git workflows.
    """,
    "parameters": {
        "type": "object",
        "properties": {
            "operation": {
                "type": "string",
                "enum": ["blame", "history", "branches", "conflicts", "bisect", "cherry_pick_plan", "diff_analysis"],
                "description": "Git operation to perform"
            },
            "file_path": {
                "type": "string",
                "description": "File path for blame/history operations"
            },
            "line_range": {
                "type": "string",
                "description": "Line range for blame (e.g., '10-20', '42')"
            },
            "query": {
                "type": "string",
                "description": "Search query for history operation"
            },
            "ref": {
                "type": "string",
                "description": "Git ref (branch, tag, commit) for operations"
            },
            "base_ref": {
                "type": "string",
                "description": "Base ref for diff/comparison operations"
            },
            "include_context": {
                "type": "boolean",
                "default": true,
                "description": "Include PR/ticket context in results"
            },
            "max_results": {
                "type": "integer",
                "default": 20,
                "description": "Maximum results for search operations"
            }
        },
        "required": ["operation"]
    }
}
```

### Output Schema

```python
@dataclass
class GitAdvancedOutput:
    operation: str

    # For blame
    blame_results: list[BlameEntry] | None = None

    # For history
    commits: list[EnrichedCommit] | None = None

    # For branches
    branches: BranchAnalysis | None = None

    # For conflicts
    conflicts: list[ConflictAnalysis] | None = None

    # For diff_analysis
    diff: DiffAnalysis | None = None

@dataclass
class BlameEntry:
    line_start: int
    line_end: int
    content: str                        # The actual code lines

    # Git info
    commit_sha: str
    author: str
    author_date: datetime
    commit_message: str

    # Context (if include_context=True)
    pr_number: int | None
    pr_title: str | None
    ticket_key: str | None
    ticket_title: str | None
    discussion_summary: str | None      # AI-summarized context

@dataclass
class EnrichedCommit:
    sha: str
    short_sha: str
    author: str
    author_email: str
    date: datetime
    message: str
    body: str | None

    # Stats
    files_changed: int
    insertions: int
    deletions: int

    # Context
    pr_number: int | None
    ticket_keys: list[str]
    tags: list[str]                     # Tags pointing to this commit

    # Relevance (for search)
    relevance_score: float | None

@dataclass
class BranchAnalysis:
    current: str
    local_branches: list[BranchInfo]
    remote_branches: list[BranchInfo]
    stale_branches: list[BranchInfo]    # No commits in X days
    merged_branches: list[BranchInfo]   # Already merged to main

@dataclass
class BranchInfo:
    name: str
    last_commit_date: datetime
    last_commit_sha: str
    last_commit_message: str
    ahead_of_main: int
    behind_main: int
    author: str                         # Most recent committer
    pr_status: str | None               # open | merged | closed | none

@dataclass
class ConflictAnalysis:
    file_path: str
    conflict_type: str                  # content | rename | delete
    ours_summary: str                   # What our side changed
    theirs_summary: str                 # What their side changed
    suggested_resolution: str | None    # AI suggestion if clear
    context: ConflictContext

@dataclass
class ConflictContext:
    ours_commits: list[str]             # Commits on our side
    theirs_commits: list[str]           # Commits on their side
    common_ancestor: str
    ours_pr: int | None
    theirs_pr: int | None
```

---

## Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                      tool-git-advanced                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │  Operation  │───▶│    Git      │───▶│  Context    │         │
│  │  Dispatcher │    │   Engine    │    │  Enricher   │         │
│  └─────────────┘    └──────┬──────┘    └──────┬──────┘         │
│                            │                  │                 │
│                            ▼                  ▼                 │
│                     ┌─────────────┐    ┌─────────────┐         │
│                     │   GitPython │    │   GitHub    │         │
│                     │   / libgit2 │    │   Client    │         │
│                     └─────────────┘    └─────────────┘         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Internal Components

#### 1. Blame with Context

```python
class ContextualBlame:
    """Blame with PR and ticket context."""

    async def blame(
        self,
        file_path: str,
        line_range: tuple[int, int] | None = None,
        include_context: bool = True
    ) -> list[BlameEntry]:
        # Get raw blame
        raw_blame = await self.git.blame(file_path, line_range)

        # Group consecutive lines with same commit
        entries = self._group_blame_entries(raw_blame)

        if include_context:
            # Enrich with PR/ticket context
            for entry in entries:
                entry.pr_number, entry.pr_title = await self._find_pr_for_commit(entry.commit_sha)
                entry.ticket_key, entry.ticket_title = await self._find_ticket_for_commit(entry.commit_sha)

                # Generate discussion summary if we have context
                if entry.pr_number or entry.ticket_key:
                    entry.discussion_summary = await self._summarize_context(entry)

        return entries

    async def _find_pr_for_commit(self, sha: str) -> tuple[int | None, str | None]:
        """Find PR that introduced this commit."""
        # Check if commit message references PR
        commit = await self.git.get_commit(sha)
        pr_match = re.search(r'#(\d+)', commit.message)
        if pr_match:
            pr_num = int(pr_match.group(1))
            pr = await self.github.get_pr(pr_num)
            return pr_num, pr.title

        # Search GitHub for PR containing this commit
        prs = await self.github.search_prs_for_commit(sha)
        if prs:
            return prs[0].number, prs[0].title

        return None, None
```

#### 2. History Search

```python
class HistorySearch:
    """Semantic search through commit history."""

    async def search(
        self,
        query: str,
        file_path: str | None = None,
        max_results: int = 20
    ) -> list[EnrichedCommit]:
        # Get commit log
        commits = await self.git.log(
            path=file_path,
            max_count=1000  # Search through recent history
        )

        # Score commits by relevance
        scored = []
        for commit in commits:
            score = self._score_commit(commit, query)
            if score > 0.3:  # Minimum relevance threshold
                scored.append((commit, score))

        # Sort by relevance
        scored.sort(key=lambda x: x[1], reverse=True)

        # Enrich top results
        results = []
        for commit, score in scored[:max_results]:
            enriched = await self._enrich_commit(commit)
            enriched.relevance_score = score
            results.append(enriched)

        return results

    def _score_commit(self, commit: Commit, query: str) -> float:
        """Score commit relevance to query."""
        query_lower = query.lower()
        score = 0.0

        # Message match
        if query_lower in commit.message.lower():
            score += 0.5
        elif any(word in commit.message.lower() for word in query_lower.split()):
            score += 0.3

        # File path match (if commit touches relevant files)
        for file in commit.files:
            if query_lower in file.lower():
                score += 0.2
                break

        # Ticket reference match
        ticket_match = re.search(r'([A-Z]+-\d+)', commit.message)
        if ticket_match and query_lower in ticket_match.group(1).lower():
            score += 0.4

        return min(score, 1.0)
```

#### 3. Conflict Analysis

```python
class ConflictAnalyzer:
    """Analyze and assist with merge conflicts."""

    async def analyze_conflicts(self) -> list[ConflictAnalysis]:
        # Get list of conflicted files
        conflicted = await self.git.get_conflicted_files()

        analyses = []
        for file_path in conflicted:
            analysis = await self._analyze_file_conflict(file_path)
            analyses.append(analysis)

        return analyses

    async def _analyze_file_conflict(self, file_path: str) -> ConflictAnalysis:
        # Get conflict markers and content
        content = await self.git.get_file_with_conflicts(file_path)
        ours, theirs, base = self._parse_conflict_regions(content)

        # Get commits involved
        ours_commits = await self.git.log(f"MERGE_HEAD..HEAD", path=file_path)
        theirs_commits = await self.git.log(f"HEAD..MERGE_HEAD", path=file_path)

        # Summarize changes
        ours_summary = self._summarize_changes(base, ours)
        theirs_summary = self._summarize_changes(base, theirs)

        # Try to suggest resolution
        suggestion = await self._suggest_resolution(ours, theirs, base, ours_summary, theirs_summary)

        return ConflictAnalysis(
            file_path=file_path,
            conflict_type=self._detect_conflict_type(content),
            ours_summary=ours_summary,
            theirs_summary=theirs_summary,
            suggested_resolution=suggestion,
            context=ConflictContext(
                ours_commits=[c.sha for c in ours_commits],
                theirs_commits=[c.sha for c in theirs_commits],
                common_ancestor=await self.git.merge_base("HEAD", "MERGE_HEAD"),
                ours_pr=await self._find_pr_for_branch("HEAD"),
                theirs_pr=await self._find_pr_for_branch("MERGE_HEAD")
            )
        )

    async def _suggest_resolution(
        self,
        ours: str,
        theirs: str,
        base: str,
        ours_summary: str,
        theirs_summary: str
    ) -> str | None:
        """Suggest resolution if changes don't truly conflict."""

        # Case 1: One side is superset of other
        if theirs in ours:
            return f"Keep ours - already includes their changes"
        if ours in theirs:
            return f"Keep theirs - already includes our changes"

        # Case 2: Changes to different parts (textual conflict but logical merge possible)
        if self._changes_are_independent(ours, theirs, base):
            return f"Both changes are independent - combine both"

        # Case 3: Need manual resolution
        return None
```

---

## Configuration

```yaml
tool-git-advanced:
  # Git settings
  git:
    repo_path: "."                      # Repository root

  # Context enrichment
  context:
    enabled: true
    github_token: "${GITHUB_TOKEN}"     # For PR lookups
    jira_token: "${JIRA_TOKEN}"         # For ticket lookups

    # Ticket patterns to recognize in commit messages
    ticket_patterns:
      - "[A-Z]{2,10}-\\d+"              # JIRA-style

  # History search
  history:
    max_commits_to_search: 1000
    relevance_threshold: 0.3

  # Branch management
  branches:
    stale_days: 30                      # Days without commits = stale
    protected_branches:
      - main
      - master
      - develop
      - release/*

  # Conflict resolution
  conflicts:
    enable_suggestions: true
    ai_assisted: true                   # Use AI for complex conflicts
```

---

## Examples

### Example 1: Contextual Blame

```python
# Input
{
    "operation": "blame",
    "file_path": "src/payments/gateway.py",
    "line_range": "42-50",
    "include_context": true
}

# Output
{
    "operation": "blame",
    "blame_results": [
        {
            "line_start": 42,
            "line_end": 48,
            "content": "@retry(max_attempts=3)\nasync def process_payment(...):\n    ...",
            "commit_sha": "abc123",
            "author": "alice",
            "author_date": "2024-01-15T10:30:00Z",
            "commit_message": "Add retry logic to payment gateway\n\nCloses JIRA-4523",
            "pr_number": 892,
            "pr_title": "Add retry logic to payment gateway",
            "ticket_key": "JIRA-4523",
            "ticket_title": "Payment failures during peak hours",
            "discussion_summary": "Added retry with exponential backoff after 5% payment failures during Black Friday. Team decided on 3 retries max to avoid customer confusion."
        }
    ]
}
```

### Example 2: History Search

```python
# Input
{
    "operation": "history",
    "query": "authentication timeout",
    "max_results": 5
}

# Output
{
    "operation": "history",
    "commits": [
        {
            "sha": "def456",
            "short_sha": "def456",
            "author": "bob",
            "date": "2024-01-10T14:20:00Z",
            "message": "Increase auth session timeout to 30 minutes\n\nPer JIRA-4400",
            "files_changed": 2,
            "insertions": 5,
            "deletions": 2,
            "pr_number": 845,
            "ticket_keys": ["JIRA-4400"],
            "relevance_score": 0.92
        },
        {
            "sha": "ghi789",
            "author": "carol",
            "date": "2023-12-15T09:00:00Z",
            "message": "Fix authentication timeout race condition",
            "relevance_score": 0.78
        }
    ]
}
```

### Example 3: Conflict Analysis

```python
# Input
{
    "operation": "conflicts"
}

# Output
{
    "operation": "conflicts",
    "conflicts": [
        {
            "file_path": "src/config/settings.py",
            "conflict_type": "content",
            "ours_summary": "Added PAYMENT_TIMEOUT setting",
            "theirs_summary": "Added AUTH_TIMEOUT setting",
            "suggested_resolution": "Both changes are independent - combine both",
            "context": {
                "ours_commits": ["abc123"],
                "theirs_commits": ["def456"],
                "common_ancestor": "base00",
                "ours_pr": 900,
                "theirs_pr": 901
            }
        }
    ]
}
```

---

## Security Considerations

- Reads git history (may contain sensitive commit messages)
- Connects to GitHub API (uses provided token)
- Does not modify repository (read-only operations)
- Credentials never logged or returned

---

## Dependencies

### Required
- `gitpython` - Git operations
- `httpx` - HTTP client for GitHub API

### Optional
- `libgit2` / `pygit2` - Alternative git backend (faster for large repos)

---

## Open Questions

1. **Git provider support**: GitHub-specific or support GitLab/Bitbucket?
2. **Large repo performance**: How to handle repos with millions of commits?
3. **Conflict resolution**: How much AI assistance for complex conflicts?
4. **Branch operations**: Should we support branch deletion/creation?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
