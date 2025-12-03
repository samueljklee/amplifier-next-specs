# pr-reviewer

> **Priority**: P0 (Critical Path)
> **Status**: Draft
> **Module**: `amplifier-scenario-pr-reviewer`

## Overview

Comprehensive pull request review automation that analyzes code changes, checks for issues, runs tests, and provides structured feedback. Combines multiple tools and hooks into a coherent review workflow that augments human reviewers.

### Value Proposition

| Without | With |
|---------|------|
| Manual code inspection | Automated issue detection |
| Missed edge cases | Systematic coverage analysis |
| Inconsistent review quality | Standardized review checklist |
| Hours per complex PR | Minutes with human verification |

---

## Workflow Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PR Review Pipeline                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐      │
│  │ Context  │───▶│ Analysis │───▶│ Testing  │───▶│ Report   │      │
│  │ Gather   │    │ Phase    │    │ Phase    │    │ Generate │      │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘      │
│       │               │               │               │              │
│       ▼               ▼               ▼               ▼              │
│  • PR metadata    • Security     • Run tests    • Summary          │
│  • File diffs     • Code quality • Coverage     • Issues list      │
│  • Related PRs    • Breaking     • Performance  • Suggestions      │
│  • Linked issues    changes      • Linting      • Approval rec     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Configuration

```yaml
scenario:
  name: pr-reviewer
  version: "1.0.0"

  # Input
  input:
    pr_url: "${PR_URL}"                 # Required: GitHub/GitLab PR URL
    depth: standard                      # quick | standard | thorough

  # Pipeline stages
  stages:
    context:
      enabled: true
      tools:
        - tool-pr-context
        - tool-jira-ops
      config:
        fetch_linked_issues: true
        fetch_related_prs: true
        max_related: 5

    analysis:
      enabled: true
      parallel: true                     # Run analyses in parallel
      analyses:
        security:
          enabled: true
          hook: hooks-security-scan
          severity_threshold: medium

        code_quality:
          enabled: true
          tool: tool-codebase-search
          checks:
            - complexity
            - duplication
            - naming
            - documentation

        breaking_changes:
          enabled: true
          hook: hooks-breaking-change
          check_api: true
          check_schema: true

        license:
          enabled: true
          hook: hooks-license-checker

        architecture:
          enabled: true
          tool: tool-dependency-graph
          check_cycles: true
          check_boundaries: true

    testing:
      enabled: true
      tool: tool-test-runner
      config:
        run_affected: true              # Only tests affected by changes
        coverage_threshold: 80
        require_new_tests: true         # New code needs tests

    style:
      enabled: true
      hook: hooks-style-enforcer
      auto_fix: false                   # Don't auto-fix in review

  # Output
  output:
    format: markdown                    # markdown | json | github-review
    post_comment: true                  # Post as PR comment
    request_changes: auto               # auto | manual | never
    approve_threshold: 0                # 0 issues = auto approve suggestion

  # Customization
  rules:
    # File-specific rules
    paths:
      - pattern: "src/api/**"
        require_openapi_update: true
      - pattern: "migrations/**"
        require_dba_approval: true
      - pattern: "*.test.*"
        skip_coverage_check: true

    # Reviewer-specific
    expertise_match: true               # Match reviewers to changed areas
```

---

## Pipeline Implementation

```python
class PRReviewerScenario:
    """Comprehensive PR review automation."""

    def __init__(self, config: PRReviewerConfig):
        self.config = config
        self.tools = ToolRegistry()
        self.hooks = HookRegistry()

    async def run(self, pr_url: str) -> ReviewReport:
        """Execute full review pipeline."""

        # Stage 1: Gather Context
        context = await self._gather_context(pr_url)

        # Stage 2: Run Analyses (parallel)
        analyses = await self._run_analyses(context)

        # Stage 3: Run Tests
        test_results = await self._run_tests(context)

        # Stage 4: Style Check
        style_results = await self._check_style(context)

        # Stage 5: Generate Report
        report = await self._generate_report(
            context, analyses, test_results, style_results
        )

        # Stage 6: Post Results
        if self.config.output.post_comment:
            await self._post_comment(pr_url, report)

        return report

    async def _gather_context(self, pr_url: str) -> PRContext:
        """Stage 1: Gather all relevant context."""

        # Get PR details
        pr_context_tool = self.tools.get("tool-pr-context")
        pr_data = await pr_context_tool.execute({
            "pr_url": pr_url,
            "include_diff": True,
            "include_comments": True,
            "include_reviews": True
        })

        # Get linked issues
        jira_tool = self.tools.get("tool-jira-ops")
        linked_issues = []

        for issue_ref in pr_data.get("linked_issues", []):
            issue = await jira_tool.execute({
                "action": "get",
                "issue_key": issue_ref
            })
            linked_issues.append(issue)

        # Get related PRs
        related_prs = await pr_context_tool.execute({
            "action": "find_related",
            "pr_url": pr_url,
            "max_results": self.config.stages.context.config.max_related
        })

        return PRContext(
            pr=pr_data,
            linked_issues=linked_issues,
            related_prs=related_prs,
            changed_files=pr_data["changed_files"],
            diff=pr_data["diff"]
        )

    async def _run_analyses(self, context: PRContext) -> AnalysisResults:
        """Stage 2: Run all configured analyses in parallel."""

        tasks = []
        results = AnalysisResults()

        if self.config.stages.analysis.analyses.security.enabled:
            tasks.append(self._analyze_security(context))

        if self.config.stages.analysis.analyses.code_quality.enabled:
            tasks.append(self._analyze_quality(context))

        if self.config.stages.analysis.analyses.breaking_changes.enabled:
            tasks.append(self._analyze_breaking(context))

        if self.config.stages.analysis.analyses.license.enabled:
            tasks.append(self._analyze_licenses(context))

        if self.config.stages.analysis.analyses.architecture.enabled:
            tasks.append(self._analyze_architecture(context))

        # Run in parallel
        completed = await asyncio.gather(*tasks, return_exceptions=True)

        # Collect results
        for result in completed:
            if isinstance(result, Exception):
                results.errors.append(str(result))
            else:
                results.merge(result)

        return results

    async def _analyze_security(self, context: PRContext) -> SecurityAnalysis:
        """Run security analysis on changed files."""

        security_hook = self.hooks.get("hooks-security-scan")
        issues = []

        for file in context.changed_files:
            result = await security_hook({
                "file_path": file.path,
                "content": file.new_content,
                "diff": file.diff
            })

            if result.findings:
                issues.extend(result.findings)

        # Filter by severity threshold
        threshold = self.config.stages.analysis.analyses.security.severity_threshold
        filtered = [i for i in issues if i.severity >= threshold]

        return SecurityAnalysis(
            issues=filtered,
            passed=len([i for i in filtered if i.severity == "high"]) == 0
        )

    async def _analyze_breaking(self, context: PRContext) -> BreakingAnalysis:
        """Check for breaking changes."""

        breaking_hook = self.hooks.get("hooks-breaking-change")
        changes = []

        for file in context.changed_files:
            if file.old_content:  # Existing file modified
                result = await breaking_hook.analyze(
                    file.path,
                    file.old_content,
                    file.new_content
                )
                changes.extend(result.breaking_changes)

        return BreakingAnalysis(
            changes=changes,
            requires_version_bump=len(changes) > 0
        )

    async def _run_tests(self, context: PRContext) -> TestResults:
        """Stage 3: Run relevant tests."""

        test_tool = self.tools.get("tool-test-runner")

        # Determine affected tests
        affected = await test_tool.execute({
            "action": "find_affected",
            "changed_files": [f.path for f in context.changed_files]
        })

        # Run tests
        results = await test_tool.execute({
            "action": "run",
            "test_files": affected,
            "coverage": True
        })

        # Check coverage for new code
        if self.config.stages.testing.config.require_new_tests:
            new_code_coverage = self._calculate_new_code_coverage(
                context, results.coverage
            )
            results.new_code_coverage = new_code_coverage

        return results

    async def _generate_report(
        self,
        context: PRContext,
        analyses: AnalysisResults,
        tests: TestResults,
        style: StyleResults
    ) -> ReviewReport:
        """Stage 5: Generate comprehensive review report."""

        # Collect all issues
        all_issues = []
        all_issues.extend(analyses.security.issues)
        all_issues.extend(analyses.breaking.changes)
        all_issues.extend(analyses.quality.issues)
        all_issues.extend(style.issues)

        # Determine recommendation
        blocking_issues = [i for i in all_issues if i.blocking]
        recommendation = self._determine_recommendation(
            blocking_issues, tests, analyses
        )

        # Generate summary using LLM
        summary = await self._generate_summary(context, all_issues, tests)

        return ReviewReport(
            pr_url=context.pr["url"],
            summary=summary,
            recommendation=recommendation,
            issues=all_issues,
            test_results=tests,
            analyses=analyses,
            suggested_reviewers=await self._suggest_reviewers(context)
        )

    def _determine_recommendation(
        self,
        blocking_issues: list,
        tests: TestResults,
        analyses: AnalysisResults
    ) -> ReviewRecommendation:
        """Determine review recommendation."""

        if blocking_issues:
            return ReviewRecommendation(
                action="request_changes",
                reason=f"{len(blocking_issues)} blocking issues found",
                confidence=0.9
            )

        if not tests.passed:
            return ReviewRecommendation(
                action="request_changes",
                reason="Tests failing",
                confidence=0.95
            )

        if analyses.security.issues:
            return ReviewRecommendation(
                action="comment",
                reason="Security findings require review",
                confidence=0.7
            )

        # All checks passed
        return ReviewRecommendation(
            action="approve",
            reason="All automated checks passed",
            confidence=0.8
        )
```

---

## Report Format

### Markdown Output

```markdown
# PR Review: Add payment retry logic (#1234)

## Summary

This PR adds exponential backoff retry logic to the payment gateway.
Changes look good overall with a few minor suggestions.

**Recommendation: ✅ Approve** (with comments)

---

## Analysis Results

### Security (✅ Passed)
No security issues detected.

### Breaking Changes (⚠️ Warning)
- API endpoint `POST /payments` has new required field `idempotency_key`
  - **Suggestion**: Make field optional for backward compatibility

### Code Quality (✅ Passed)
- Complexity: Within limits
- Duplication: None detected
- Test coverage: 87% (threshold: 80%)

### License Check (✅ Passed)
No new dependencies with restricted licenses.

---

## Test Results

| Suite | Passed | Failed | Skipped |
|-------|--------|--------|---------|
| Unit  | 45     | 0      | 2       |
| Integration | 12 | 0   | 0       |

**Coverage**: 87% overall, 92% for new code

---

## Issues (3)

### 🔴 High Priority

1. **Missing error handling** (src/payments/gateway.py:142)
   ```python
   # Current
   response = await client.post(url, data)

   # Suggested
   try:
       response = await client.post(url, data)
   except TimeoutError:
       raise PaymentTimeoutError(...)
   ```

### 🟡 Medium Priority

2. **Consider adding type hints** (src/payments/retry.py:28)
   Function `calculate_backoff` lacks return type annotation.

### 🟢 Low Priority

3. **Documentation update needed** (README.md)
   New retry behavior should be documented.

---

## Suggested Reviewers

Based on code ownership and expertise:
- **@alice** - Primary owner of payments module
- **@bob** - Recent contributor to gateway code

---

*Generated by PR Reviewer v1.0.0*
```

---

## Integration with CI/CD

### GitHub Actions

```yaml
name: PR Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run PR Review
        uses: amplifier/pr-reviewer-action@v1
        with:
          pr-url: ${{ github.event.pull_request.html_url }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          depth: standard
          post-comment: true
```

### GitLab CI

```yaml
pr-review:
  stage: review
  script:
    - amplifier run pr-reviewer --pr-url $CI_MERGE_REQUEST_IID
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `input.depth` | string | "standard" | Review depth |
| `stages.*.enabled` | bool | true | Enable/disable stages |
| `output.format` | string | "markdown" | Output format |
| `output.post_comment` | bool | true | Post as PR comment |
| `output.request_changes` | string | "auto" | When to request changes |
| `rules.expertise_match` | bool | true | Match reviewers to areas |

---

## Dependencies

### Required Tools
- `tool-pr-context` - PR data fetching
- `tool-test-runner` - Test execution
- `tool-codebase-search` - Code analysis

### Required Hooks
- `hooks-security-scan` - Security analysis
- `hooks-breaking-change` - Breaking change detection
- `hooks-license-checker` - License compliance
- `hooks-style-enforcer` - Style checking

### External
- GitHub/GitLab API access
- CI/CD integration (optional)

---

## Open Questions

1. **ML enhancement**: Train on past reviews to improve suggestions?
2. **Custom rules**: Support organization-specific review rules?
3. **Review history**: Track review patterns and outcomes?
4. **Batch review**: Review multiple related PRs together?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
