# hooks-reviewer-suggester

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-hooks-reviewer-suggester`

## Overview

Analyzes code changes and suggests optimal reviewers based on code ownership, expertise, availability, and review load balance. Helps ensure PRs get reviewed by the right people quickly.

### Value Proposition

| Without | With |
|---------|------|
| Guessing who should review | Data-driven suggestions |
| Overloading same reviewers | Balanced review load |
| Missing domain experts | Expertise matching |
| Slow review assignment | Instant suggestions |

---

## Contract

### Hook Configuration

```yaml
hooks:
  - module: hooks-reviewer-suggester
    config:
      # Data sources
      sources:
        git_blame: true                 # Use git blame for ownership
        codeowners: true                # Use CODEOWNERS file
        review_history: true            # Past review patterns
        org_chart: false                # Optional: org structure

      # Suggestion criteria
      criteria:
        ownership_weight: 0.4           # How much ownership matters
        expertise_weight: 0.3           # Past changes to similar code
        availability_weight: 0.2        # Current review load
        recency_weight: 0.1             # Recent activity in area

      # Constraints
      constraints:
        min_reviewers: 1
        max_reviewers: 3
        avoid_author: true              # Don't suggest PR author
        require_codeowner: true         # At least one CODEOWNER
        max_pending_reviews: 5          # Skip overloaded reviewers

      # Integration
      integration:
        platform: github                # github | gitlab | azure-devops
        auto_assign: false              # Auto-assign or just suggest
        notify: true                    # Notify suggested reviewers

      # Trigger
      trigger_on:
        - pr_created
        - pr_updated
        - files_changed
```

### HookResult

```python
# Suggest reviewers
HookResult(
    action="inject_context",
    context_injection="""
**Suggested Reviewers for This Change**

Based on code ownership and expertise:

1. **@alice** (Score: 0.92)
   - Primary owner of `src/payments/` (CODEOWNERS)
   - 45 commits to payment module in past 6 months
   - Current review load: 2 pending PRs

2. **@bob** (Score: 0.78)
   - Frequent contributor to `src/api/`
   - Reviewed similar changes 12 times
   - Current review load: 3 pending PRs

3. **@carol** (Score: 0.65)
   - Secondary owner of `src/payments/gateway.py`
   - Domain expertise: payment gateways
   - Current review load: 1 pending PR

**Recommendation**: Assign @alice (required: CODEOWNER) + @bob (expertise)
""",
    user_message="Suggested reviewers: @alice, @bob, @carol",
    user_message_level="info"
)
```

---

## Architecture

```python
class ReviewerSuggesterHook:
    """Suggest optimal code reviewers."""

    def __init__(self, config: ReviewerSuggesterConfig):
        self.config = config
        self.ownership_analyzer = OwnershipAnalyzer(config)
        self.load_balancer = ReviewLoadBalancer(config)
        self.scorer = ReviewerScorer(config.criteria)

    async def __call__(self, event: str, data: dict) -> HookResult:
        # Trigger on relevant events
        if not self._should_trigger(event, data):
            return HookResult(action="continue")

        # Get changed files
        changed_files = await self._get_changed_files(data)
        if not changed_files:
            return HookResult(action="continue")

        # Find candidate reviewers
        candidates = await self._find_candidates(changed_files, data)

        # Score and rank candidates
        scored = await self._score_candidates(candidates, changed_files)

        # Apply constraints
        suggestions = self._apply_constraints(scored, data)

        # Generate suggestion
        return self._generate_suggestion(suggestions, changed_files)

    async def _find_candidates(
        self,
        changed_files: list[str],
        data: dict
    ) -> list[ReviewerCandidate]:
        """Find potential reviewers from multiple sources."""
        candidates = {}

        # CODEOWNERS
        if self.config.sources.codeowners:
            codeowner_reviewers = await self.ownership_analyzer.from_codeowners(
                changed_files
            )
            for reviewer in codeowner_reviewers:
                candidates[reviewer.id] = reviewer
                candidates[reviewer.id].is_codeowner = True

        # Git blame
        if self.config.sources.git_blame:
            blame_reviewers = await self.ownership_analyzer.from_git_blame(
                changed_files
            )
            for reviewer in blame_reviewers:
                if reviewer.id in candidates:
                    candidates[reviewer.id].ownership_score += reviewer.ownership_score
                else:
                    candidates[reviewer.id] = reviewer

        # Review history
        if self.config.sources.review_history:
            history_reviewers = await self.ownership_analyzer.from_review_history(
                changed_files
            )
            for reviewer in history_reviewers:
                if reviewer.id in candidates:
                    candidates[reviewer.id].expertise_score = reviewer.expertise_score
                else:
                    candidates[reviewer.id] = reviewer

        return list(candidates.values())

    async def _score_candidates(
        self,
        candidates: list[ReviewerCandidate],
        changed_files: list[str]
    ) -> list[ScoredReviewer]:
        """Score candidates based on criteria."""
        scored = []

        for candidate in candidates:
            # Get current review load
            pending_reviews = await self.load_balancer.get_pending_count(
                candidate.id
            )

            # Calculate availability score (inverse of load)
            max_load = self.config.constraints.max_pending_reviews
            availability = 1.0 - (pending_reviews / max_load) if pending_reviews < max_load else 0

            # Calculate final score
            score = self.scorer.calculate(
                ownership=candidate.ownership_score,
                expertise=candidate.expertise_score,
                availability=availability,
                recency=candidate.recency_score
            )

            scored.append(ScoredReviewer(
                candidate=candidate,
                score=score,
                pending_reviews=pending_reviews,
                breakdown={
                    "ownership": candidate.ownership_score,
                    "expertise": candidate.expertise_score,
                    "availability": availability,
                    "recency": candidate.recency_score
                }
            ))

        # Sort by score descending
        return sorted(scored, key=lambda x: x.score, reverse=True)

    def _apply_constraints(
        self,
        scored: list[ScoredReviewer],
        data: dict
    ) -> list[ScoredReviewer]:
        """Apply constraints to suggestions."""
        filtered = []
        author = data.get("author")

        for reviewer in scored:
            # Skip author
            if self.config.constraints.avoid_author and reviewer.candidate.id == author:
                continue

            # Skip overloaded reviewers
            if reviewer.pending_reviews >= self.config.constraints.max_pending_reviews:
                continue

            filtered.append(reviewer)

        # Ensure we have required CODEOWNER
        if self.config.constraints.require_codeowner:
            has_codeowner = any(r.candidate.is_codeowner for r in filtered)
            if not has_codeowner:
                # Find first codeowner regardless of constraints
                for reviewer in scored:
                    if reviewer.candidate.is_codeowner and reviewer.candidate.id != author:
                        filtered.insert(0, reviewer)
                        break

        # Limit to max reviewers
        return filtered[:self.config.constraints.max_reviewers]


class OwnershipAnalyzer:
    """Analyze code ownership from multiple sources."""

    async def from_codeowners(self, files: list[str]) -> list[ReviewerCandidate]:
        """Get owners from CODEOWNERS file."""
        codeowners_path = self._find_codeowners()
        if not codeowners_path:
            return []

        rules = self._parse_codeowners(codeowners_path)
        owners = set()

        for file in files:
            for pattern, file_owners in rules:
                if self._matches_pattern(file, pattern):
                    owners.update(file_owners)

        return [
            ReviewerCandidate(id=owner, ownership_score=1.0, is_codeowner=True)
            for owner in owners
        ]

    async def from_git_blame(self, files: list[str]) -> list[ReviewerCandidate]:
        """Analyze git blame for ownership."""
        author_lines = defaultdict(int)
        total_lines = 0

        for file in files:
            blame = await self._run_git_blame(file)
            for line in blame:
                author_lines[line.author] += 1
                total_lines += 1

        return [
            ReviewerCandidate(
                id=author,
                ownership_score=lines / total_lines if total_lines > 0 else 0
            )
            for author, lines in author_lines.items()
        ]

    async def from_review_history(self, files: list[str]) -> list[ReviewerCandidate]:
        """Find reviewers who have reviewed similar files."""
        # Query past reviews touching these files
        past_reviews = await self._query_review_history(files)

        reviewer_counts = defaultdict(int)
        for review in past_reviews:
            reviewer_counts[review.reviewer] += 1

        max_reviews = max(reviewer_counts.values()) if reviewer_counts else 1
        return [
            ReviewerCandidate(
                id=reviewer,
                expertise_score=count / max_reviews
            )
            for reviewer, count in reviewer_counts.items()
        ]


class ReviewerScorer:
    """Calculate weighted reviewer scores."""

    def __init__(self, criteria: dict):
        self.ownership_weight = criteria.get("ownership_weight", 0.4)
        self.expertise_weight = criteria.get("expertise_weight", 0.3)
        self.availability_weight = criteria.get("availability_weight", 0.2)
        self.recency_weight = criteria.get("recency_weight", 0.1)

    def calculate(
        self,
        ownership: float,
        expertise: float,
        availability: float,
        recency: float
    ) -> float:
        """Calculate weighted score."""
        return (
            ownership * self.ownership_weight +
            expertise * self.expertise_weight +
            availability * self.availability_weight +
            recency * self.recency_weight
        )
```

---

## Data Sources

### CODEOWNERS Integration

```python
# Parse CODEOWNERS file
# .github/CODEOWNERS or CODEOWNERS

# Example CODEOWNERS:
# * @default-team
# /src/payments/ @alice @bob
# /src/api/ @api-team
# *.py @python-experts

def _parse_codeowners(path: str) -> list[tuple[str, list[str]]]:
    """Parse CODEOWNERS file into rules."""
    rules = []
    with open(path) as f:
        for line in f:
            line = line.strip()
            if not line or line.startswith("#"):
                continue
            parts = line.split()
            pattern = parts[0]
            owners = [p.lstrip("@") for p in parts[1:]]
            rules.append((pattern, owners))
    return rules
```

### Git Blame Analysis

```python
async def _run_git_blame(file: str) -> list[BlameLine]:
    """Run git blame and parse output."""
    proc = await asyncio.create_subprocess_exec(
        "git", "blame", "--line-porcelain", file,
        stdout=asyncio.subprocess.PIPE
    )
    stdout, _ = await proc.communicate()

    # Parse porcelain format
    lines = []
    current = {}
    for line in stdout.decode().split("\n"):
        if line.startswith("author "):
            current["author"] = line[7:]
        elif line.startswith("author-mail "):
            current["email"] = line[12:].strip("<>")
        elif line.startswith("\t"):
            # Content line - commit current
            if current:
                lines.append(BlameLine(**current))
                current = {}
    return lines
```

### Review History Query

```python
async def _query_review_history(files: list[str]) -> list[PastReview]:
    """Query GitHub/GitLab for past reviews."""
    # GitHub GraphQL query
    query = """
    query($files: [String!]!) {
        repository(owner: $owner, name: $repo) {
            pullRequests(first: 100, states: MERGED) {
                nodes {
                    files(first: 100) {
                        nodes { path }
                    }
                    reviews(first: 10) {
                        nodes {
                            author { login }
                            state
                        }
                    }
                }
            }
        }
    }
    """
    # Filter for PRs touching these files
    # Return reviewers who approved
```

---

## Examples

### Example 1: New PR Created

```python
# PR created touching src/payments/gateway.py and src/api/routes.py

# Hook analyzes:
# - CODEOWNERS: @alice owns /src/payments/, @api-team owns /src/api/
# - Git blame: @bob has 60% of lines in gateway.py, @carol has 30%
# - Review history: @alice reviewed 15 PRs in payments, @dave reviewed 8

# Scoring:
# @alice: ownership=1.0, expertise=0.9, availability=0.8, recency=0.7 → 0.91
# @bob: ownership=0.6, expertise=0.5, availability=1.0, recency=0.9 → 0.71
# @api-team/member: ownership=0.5, expertise=0.3, availability=0.6 → 0.44

# Result:
"""
Suggested Reviewers:
1. @alice (CODEOWNER) - Primary owner, high expertise
2. @bob - Significant contributor to gateway.py
"""
```

### Example 2: Load Balancing in Action

```python
# Normally @alice would be top suggestion, but she has 5 pending reviews

# Hook adjusts:
# @alice: availability=0.0 (at max load)
# Final score drops significantly

# Result suggests @bob instead:
"""
Suggested Reviewers:
1. @bob - Top available expert (Score: 0.78)
   Note: @alice (CODEOWNER) currently overloaded (5 pending reviews)
2. @carol - Secondary expert (Score: 0.65)
"""
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `sources.*` | bool | true | Data sources to use |
| `criteria.*_weight` | float | varies | Scoring weights |
| `constraints.min_reviewers` | int | 1 | Minimum suggestions |
| `constraints.max_reviewers` | int | 3 | Maximum suggestions |
| `constraints.max_pending_reviews` | int | 5 | Load threshold |
| `integration.auto_assign` | bool | false | Auto-assign reviewers |

---

## Security Considerations

- Requires read access to git history
- May need API tokens for review history
- Respects repository visibility settings
- No sensitive code content exposed

---

## Dependencies

### Required
- `gitpython` - Git operations

### Optional
- `PyGithub` - GitHub API
- `python-gitlab` - GitLab API

---

## Open Questions

1. **Team rotation**: Should we enforce reviewer rotation for knowledge sharing?
2. **Expertise decay**: Should expertise scores decay over time?
3. **Cross-team reviews**: Encourage reviews from outside the immediate team?
4. **Learning mode**: Track suggestion accuracy and improve?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
