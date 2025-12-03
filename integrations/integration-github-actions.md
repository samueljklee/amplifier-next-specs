# integration-github-actions

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-integration-github-actions`

## Overview

GitHub Actions integration for running Amplifier in CI/CD pipelines. Provides pre-built actions for code review, PR analysis, documentation generation, and custom AI-powered workflows.

### Value Proposition

| Without | With |
|---------|------|
| Manual AI-assisted reviews | Automated PR feedback |
| No CI/CD AI integration | Native Actions support |
| Complex setup | Drop-in action |
| Token management | Secure secrets handling |

---

## Actions Available

### 1. amplifier/pr-review-action

Automated PR review with configurable depth.

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

      - uses: amplifier/pr-review-action@v1
        with:
          # Required
          github-token: ${{ secrets.GITHUB_TOKEN }}
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}

          # Optional configuration
          depth: standard              # quick | standard | thorough
          checks:
            security: true
            code-quality: true
            breaking-changes: true
            test-coverage: true

          # Output
          post-comment: true
          request-changes: auto        # auto | manual | never
          fail-on-critical: true

          # Custom prompts (optional)
          custom-instructions: |
            Pay special attention to:
            - SQL injection vulnerabilities
            - Authentication bypasses
            - Our coding standards in CONTRIBUTING.md
```

### 2. amplifier/code-analysis-action

Deep code analysis without PR context.

```yaml
- uses: amplifier/code-analysis-action@v1
  with:
    anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}

    # What to analyze
    paths:
      - src/
      - lib/

    # Analysis types
    analysis:
      - security
      - complexity
      - architecture
      - dependencies

    # Output
    output-format: sarif           # sarif | json | markdown
    output-file: analysis.sarif

    # Thresholds
    fail-on-high-severity: true
    complexity-threshold: 15
```

### 3. amplifier/doc-generator-action

Generate or update documentation.

```yaml
- uses: amplifier/doc-generator-action@v1
  with:
    anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}

    # What to document
    source-paths:
      - src/
    output-path: docs/

    # Generation options
    mode: update                   # generate | update | verify
    types:
      - api-reference
      - code-comments
      - readme-sections

    # Git operations
    create-commit: true
    commit-message: "docs: auto-update documentation"
```

### 4. amplifier/custom-action

Run custom Amplifier scenarios.

```yaml
- uses: amplifier/custom-action@v1
  with:
    anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}

    # Custom scenario
    scenario: ./scenarios/tech-debt-scan.yaml

    # Or inline prompt
    prompt: |
      Analyze the codebase for architectural improvements.
      Focus on the payment module.

    # Configuration
    profile: enterprise-dev
    collection: enterprise-dev

    # Output
    output-file: ${{ github.workspace }}/analysis.md
```

---

## Architecture

```yaml
# Action structure
action.yml:
  name: "Amplifier PR Review"
  description: "AI-powered code review"

  inputs:
    github-token:
      required: true
    anthropic-api-key:
      required: true
    depth:
      default: "standard"
    # ... more inputs

  runs:
    using: "docker"
    image: "ghcr.io/amplifier/actions:v1"
    env:
      GITHUB_TOKEN: ${{ inputs.github-token }}
      ANTHROPIC_API_KEY: ${{ inputs.anthropic-api-key }}

# Docker image contents
FROM python:3.11-slim

# Install Amplifier
RUN pip install amplifier-cli amplifier-collection-enterprise

# Install action runner
COPY entrypoint.py /entrypoint.py
ENTRYPOINT ["python", "/entrypoint.py"]
```

---

## Implementation

```python
# entrypoint.py - Action entry point

import os
import json
import asyncio
from pathlib import Path

from amplifier import AmplifierSession
from github import Github

async def main():
    """Main action entry point."""

    # Load inputs
    inputs = load_inputs()

    # Initialize GitHub client
    gh = Github(os.environ["GITHUB_TOKEN"])
    repo = gh.get_repo(os.environ["GITHUB_REPOSITORY"])

    # Get PR context
    pr = None
    if "pull_request" in os.environ.get("GITHUB_EVENT_NAME", ""):
        event = json.loads(Path(os.environ["GITHUB_EVENT_PATH"]).read_text())
        pr = repo.get_pull(event["pull_request"]["number"])

    # Initialize Amplifier
    config = build_config(inputs)

    async with AmplifierSession(config=config) as session:
        # Execute based on action type
        if inputs["action_type"] == "pr-review":
            result = await run_pr_review(session, pr, inputs)
        elif inputs["action_type"] == "code-analysis":
            result = await run_code_analysis(session, inputs)
        elif inputs["action_type"] == "doc-generator":
            result = await run_doc_generator(session, inputs)
        else:
            result = await run_custom(session, inputs)

    # Process output
    await process_output(result, gh, repo, pr, inputs)


async def run_pr_review(
    session: AmplifierSession,
    pr: PullRequest,
    inputs: dict
) -> ReviewResult:
    """Run PR review scenario."""

    # Load PR context
    diff = pr.get_files()
    comments = pr.get_comments()

    # Build review prompt
    prompt = build_review_prompt(pr, diff, comments, inputs)

    # Execute review
    result = await session.execute(prompt)

    return ReviewResult(
        summary=result.response,
        issues=extract_issues(result),
        recommendation=extract_recommendation(result)
    )


async def process_output(
    result: Result,
    gh: Github,
    repo: Repository,
    pr: PullRequest | None,
    inputs: dict
) -> None:
    """Process action output."""

    # Post PR comment
    if inputs.get("post_comment") and pr:
        comment_body = format_comment(result)
        pr.create_issue_comment(comment_body)

    # Request changes if configured
    if inputs.get("request_changes") == "auto" and pr:
        if result.has_blocking_issues:
            pr.create_review(
                body=result.summary,
                event="REQUEST_CHANGES"
            )
        elif result.recommendation == "approve":
            pr.create_review(
                body=result.summary,
                event="APPROVE"
            )

    # Set outputs
    set_output("review-result", json.dumps(result.to_dict()))
    set_output("has-issues", str(result.has_issues).lower())

    # Fail action if critical issues
    if inputs.get("fail_on_critical") and result.has_critical_issues:
        print(f"::error::Critical issues found in review")
        exit(1)

    # Write SARIF if code analysis
    if inputs.get("output_format") == "sarif":
        sarif = convert_to_sarif(result)
        Path(inputs["output_file"]).write_text(json.dumps(sarif))

        # Upload to GitHub Code Scanning
        upload_sarif(gh, repo, sarif)


def format_comment(result: ReviewResult) -> str:
    """Format result as GitHub comment."""

    lines = [
        "## 🤖 Amplifier Code Review",
        "",
        f"**Recommendation**: {result.recommendation_emoji} {result.recommendation}",
        "",
        "### Summary",
        result.summary,
        "",
    ]

    if result.issues:
        lines.extend([
            "### Issues Found",
            "",
        ])

        for issue in result.issues:
            emoji = {"critical": "🔴", "high": "🟠", "medium": "🟡", "low": "🟢"}[issue.severity]
            lines.append(f"- {emoji} **{issue.title}** ({issue.file}:{issue.line})")
            lines.append(f"  {issue.description}")
            lines.append("")

    lines.extend([
        "---",
        "*Generated by [Amplifier](https://github.com/microsoft/amplifier)*"
    ])

    return "\n".join(lines)
```

---

## Configuration Reference

### PR Review Action

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `github-token` | string | Yes | - | GitHub token |
| `anthropic-api-key` | string | Yes | - | Anthropic API key |
| `depth` | string | No | "standard" | Review depth |
| `checks.security` | bool | No | true | Security check |
| `checks.code-quality` | bool | No | true | Quality check |
| `checks.breaking-changes` | bool | No | true | Breaking changes |
| `checks.test-coverage` | bool | No | true | Coverage check |
| `post-comment` | bool | No | true | Post as comment |
| `request-changes` | string | No | "auto" | Request changes behavior |
| `fail-on-critical` | bool | No | true | Fail on critical |
| `custom-instructions` | string | No | "" | Custom review instructions |

### Outputs

| Output | Description |
|--------|-------------|
| `review-result` | JSON result object |
| `has-issues` | Whether issues were found |
| `recommendation` | approve/request_changes/comment |
| `issue-count` | Number of issues |

---

## Security

### Secret Handling

```yaml
# Required secrets (set in repo settings)
secrets:
  ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

# Optional: Use GitHub App for enhanced permissions
secrets:
  APP_ID: ${{ secrets.AMPLIFIER_APP_ID }}
  APP_PRIVATE_KEY: ${{ secrets.AMPLIFIER_APP_PRIVATE_KEY }}

# Token permissions
permissions:
  contents: read
  pull-requests: write
  security-events: write  # For SARIF upload
```

### Rate Limiting

```yaml
# Built-in rate limiting
rate-limiting:
  max-reviews-per-hour: 10
  max-tokens-per-review: 50000
  concurrent-reviews: 3
```

---

## Example Workflows

### Complete PR Pipeline

```yaml
name: PR Pipeline
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    outputs:
      has-issues: ${{ steps.review.outputs.has-issues }}

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - id: review
        uses: amplifier/pr-review-action@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          depth: thorough
          fail-on-critical: false  # Don't fail yet

      - name: Upload SARIF
        if: always()
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: review-results.sarif

  security-gate:
    needs: review
    if: needs.review.outputs.has-issues == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Security Review Required
        run: |
          echo "::warning::AI review found issues - security team review required"
          # Could notify security team via Slack, etc.
```

### Scheduled Analysis

```yaml
name: Weekly Code Health
on:
  schedule:
    - cron: '0 8 * * 1'  # Monday 8 AM

jobs:
  analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: amplifier/code-analysis-action@v1
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          paths: [src/]
          analysis: [security, complexity, architecture]
          output-format: markdown
          output-file: reports/weekly-health.md

      - name: Create Issue
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('reports/weekly-health.md', 'utf8');
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `Weekly Code Health Report - ${new Date().toISOString().split('T')[0]}`,
              body: report,
              labels: ['code-health', 'automated']
            });
```

---

## Self-Hosted Runner Support

```yaml
# For enterprise with self-hosted runners
jobs:
  review:
    runs-on: self-hosted
    container:
      image: ghcr.io/amplifier/actions:v1
      credentials:
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    steps:
      - uses: amplifier/pr-review-action@v1
        with:
          # Use Azure OpenAI instead of Anthropic
          provider: azure-openai
          azure-endpoint: ${{ secrets.AZURE_OPENAI_ENDPOINT }}
          azure-api-key: ${{ secrets.AZURE_OPENAI_KEY }}
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
