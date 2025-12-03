# tech-debt-prioritizer

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-scenario-tech-debt-prioritizer`

## Overview

Systematic technical debt identification and prioritization workflow. Scans codebase for debt indicators, calculates business impact, estimates remediation effort, and generates prioritized action plans aligned with product goals.

### Value Proposition

| Without | With |
|---------|------|
| Invisible accumulating debt | Visible, tracked inventory |
| Arbitrary prioritization | Data-driven decisions |
| Debt addressed reactively | Strategic planning |
| No ROI visibility | Clear cost/benefit analysis |

---

## Workflow Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Tech Debt Prioritization Pipeline                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐      │
│  │  Scan    │───▶│ Analyze  │───▶│ Priorit- │───▶│ Plan     │      │
│  │          │    │ Impact   │    │ ize      │    │          │      │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘      │
│       │               │               │               │              │
│       ▼               ▼               ▼               ▼              │
│  • Code smell     • Velocity      • Business     • Sprints          │
│    detection        impact          alignment   • Resources         │
│  • Complexity     • Bug           • Cost/        • Dependencies     │
│  • Duplication      correlation     benefit    • Risks             │
│  • Dependencies   • Change risk   • Quick wins  • Tracking         │
│  • Test coverage  • Team pain     • Critical                       │
│                                                                      │
│                    ┌───────────────┐                                │
│                    │ Dashboard     │◀── Continuous                  │
│                    │ & Reporting   │                                │
│                    └───────────────┘                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Configuration

```yaml
scenario:
  name: tech-debt-prioritizer
  version: "1.0.0"

  # Scanning
  scan:
    # Code analysis
    code:
      complexity:
        enabled: true
        threshold:
          cyclomatic: 10
          cognitive: 15
      duplication:
        enabled: true
        min_tokens: 50
        min_occurrences: 2
      code_smells:
        enabled: true
        rules:
          - long_method
          - large_class
          - feature_envy
          - shotgun_surgery
          - god_object
      naming:
        enabled: true
        conventions: auto-detect

    # Architecture
    architecture:
      dependency_cycles: true
      layer_violations: true
      unstable_dependencies: true
      dead_code: true

    # Testing
    testing:
      coverage_threshold: 80
      test_quality: true
      flaky_tests: true

    # Dependencies
    dependencies:
      outdated: true
      security_vulnerabilities: true
      deprecated_usage: true
      unmaintained: 365d          # Not updated in X days

    # Documentation
    documentation:
      missing_docs: true
      stale_docs: true
      incomplete_api_docs: true

  # Impact analysis
  impact:
    # Correlations
    correlations:
      bugs_by_file: true
      change_frequency: true
      review_time: true
      deployment_failures: true

    # Sources
    sources:
      jira:
        enabled: true
        bug_labels: ["bug", "defect"]
        lookback: 180d
      github:
        enabled: true
        lookback: 365d

  # Prioritization
  prioritization:
    # Scoring weights
    weights:
      business_impact: 0.30
      remediation_effort: 0.20       # Inverse - prefer easier
      bug_correlation: 0.20
      change_risk: 0.15
      team_pain: 0.15

    # Business alignment
    alignment:
      roadmap_areas:                # Priority multiplier for roadmap areas
        - path: "src/payments/"
          multiplier: 1.5
        - path: "src/onboarding/"
          multiplier: 1.3

    # Categories
    categories:
      critical:                     # Address immediately
        threshold: 0.9
      high:                         # Next quarter
        threshold: 0.7
      medium:                       # Future planning
        threshold: 0.4
      low:                          # Track but defer
        threshold: 0.0

  # Output
  output:
    report_format: markdown
    jira_export: true
    dashboard_data: true
    tracking:
      create_epics: false
      update_existing: true
```

---

## Pipeline Implementation

```python
class TechDebtPrioritizerScenario:
    """Systematic tech debt identification and prioritization."""

    def __init__(self, config: TechDebtConfig):
        self.config = config
        self.tools = ToolRegistry()

    async def run(self) -> TechDebtReport:
        """Execute full tech debt analysis pipeline."""

        # Stage 1: Scan Codebase
        debt_items = await self._scan_codebase()

        # Stage 2: Analyze Impact
        for item in debt_items:
            item.impact = await self._analyze_impact(item)

        # Stage 3: Prioritize
        prioritized = await self._prioritize(debt_items)

        # Stage 4: Generate Plan
        plan = await self._generate_plan(prioritized)

        # Stage 5: Output
        return await self._generate_report(prioritized, plan)

    async def _scan_codebase(self) -> list[DebtItem]:
        """Stage 1: Scan for all tech debt indicators."""

        debt_items = []

        # Code analysis
        if self.config.scan.code.complexity.enabled:
            complexity_issues = await self._scan_complexity()
            debt_items.extend(complexity_issues)

        if self.config.scan.code.duplication.enabled:
            duplication_issues = await self._scan_duplication()
            debt_items.extend(duplication_issues)

        if self.config.scan.code.code_smells.enabled:
            smell_issues = await self._scan_code_smells()
            debt_items.extend(smell_issues)

        # Architecture analysis
        if self.config.scan.architecture.dependency_cycles:
            cycle_issues = await self._scan_dependency_cycles()
            debt_items.extend(cycle_issues)

        if self.config.scan.architecture.dead_code:
            dead_code_issues = await self._scan_dead_code()
            debt_items.extend(dead_code_issues)

        # Testing analysis
        coverage_issues = await self._scan_test_coverage()
        debt_items.extend(coverage_issues)

        # Dependency analysis
        dep_issues = await self._scan_dependencies()
        debt_items.extend(dep_issues)

        # Deduplicate and merge related issues
        debt_items = self._consolidate_issues(debt_items)

        return debt_items

    async def _scan_complexity(self) -> list[DebtItem]:
        """Scan for code complexity issues."""

        debt_tool = self.tools.get("tool-tech-debt-scanner")

        results = await debt_tool.execute({
            "scan_type": "complexity",
            "thresholds": {
                "cyclomatic": self.config.scan.code.complexity.threshold.cyclomatic,
                "cognitive": self.config.scan.code.complexity.threshold.cognitive
            }
        })

        items = []
        for result in results:
            items.append(DebtItem(
                id=f"complexity-{hash(result['file'] + str(result['line']))}",
                type="complexity",
                title=f"High complexity in {result['function']}",
                description=f"Cyclomatic: {result['cyclomatic']}, Cognitive: {result['cognitive']}",
                location=Location(
                    file=result["file"],
                    line_start=result["line"],
                    line_end=result["line_end"]
                ),
                severity=self._complexity_severity(result),
                raw_data=result
            ))

        return items

    async def _scan_duplication(self) -> list[DebtItem]:
        """Scan for code duplication."""

        debt_tool = self.tools.get("tool-tech-debt-scanner")

        results = await debt_tool.execute({
            "scan_type": "duplication",
            "min_tokens": self.config.scan.code.duplication.min_tokens,
            "min_occurrences": self.config.scan.code.duplication.min_occurrences
        })

        items = []
        for dup in results:
            items.append(DebtItem(
                id=f"dup-{hash(str(dup['locations']))}",
                type="duplication",
                title=f"Duplicated code ({dup['occurrences']} occurrences)",
                description=f"{dup['tokens']} tokens duplicated across {dup['occurrences']} locations",
                locations=[Location(**loc) for loc in dup["locations"]],
                severity=self._duplication_severity(dup),
                raw_data=dup
            ))

        return items

    async def _scan_dependencies(self) -> list[DebtItem]:
        """Scan for dependency issues."""

        dep_tool = self.tools.get("tool-dependency-graph")
        items = []

        # Outdated dependencies
        if self.config.scan.dependencies.outdated:
            outdated = await dep_tool.execute({
                "action": "check_outdated"
            })

            for dep in outdated:
                severity = "high" if dep["major_versions_behind"] > 1 else "medium"
                items.append(DebtItem(
                    id=f"outdated-{dep['name']}",
                    type="outdated_dependency",
                    title=f"Outdated: {dep['name']}",
                    description=f"Current: {dep['current']}, Latest: {dep['latest']}",
                    severity=severity,
                    metadata={
                        "package": dep["name"],
                        "current": dep["current"],
                        "latest": dep["latest"],
                        "breaking_changes": dep.get("breaking_changes", [])
                    }
                ))

        # Security vulnerabilities
        if self.config.scan.dependencies.security_vulnerabilities:
            vulns = await dep_tool.execute({
                "action": "security_audit"
            })

            for vuln in vulns:
                items.append(DebtItem(
                    id=f"vuln-{vuln['id']}",
                    type="security_vulnerability",
                    title=f"Security: {vuln['package']} ({vuln['severity']})",
                    description=vuln["description"],
                    severity="critical" if vuln["severity"] == "high" else vuln["severity"],
                    metadata=vuln
                ))

        return items

    async def _analyze_impact(self, item: DebtItem) -> ImpactAnalysis:
        """Stage 2: Analyze impact of a debt item."""

        impact = ImpactAnalysis()

        # Bug correlation
        if self.config.impact.correlations.bugs_by_file and item.location:
            jira_tool = self.tools.get("tool-jira-ops")
            bugs = await jira_tool.execute({
                "action": "search",
                "jql": f'labels in ({",".join(self.config.impact.sources.jira.bug_labels)}) '
                       f'AND text ~ "{item.location.file}"'
            })
            impact.bug_count = len(bugs)
            impact.bug_correlation = min(len(bugs) / 10, 1.0)  # Normalize

        # Change frequency
        if self.config.impact.correlations.change_frequency and item.location:
            git_tool = self.tools.get("tool-git-advanced")
            history = await git_tool.execute({
                "action": "file_history",
                "file": item.location.file,
                "days": 365
            })
            impact.change_frequency = len(history)
            impact.change_risk = self._calculate_change_risk(history)

        # Team pain (from PR review times)
        if self.config.impact.correlations.review_time and item.location:
            pr_tool = self.tools.get("tool-pr-context")
            prs = await pr_tool.execute({
                "action": "search",
                "file": item.location.file
            })
            if prs:
                avg_review_time = sum(pr["review_hours"] for pr in prs) / len(prs)
                impact.team_pain = min(avg_review_time / 24, 1.0)  # Normalize

        # Business impact (based on area)
        impact.business_impact = self._calculate_business_impact(item)

        return impact

    def _calculate_business_impact(self, item: DebtItem) -> float:
        """Calculate business impact score."""

        base_impact = {
            "security_vulnerability": 1.0,
            "complexity": 0.5,
            "duplication": 0.3,
            "outdated_dependency": 0.4,
            "test_coverage": 0.5,
            "dead_code": 0.2
        }.get(item.type, 0.3)

        # Apply roadmap multiplier
        if item.location:
            for area in self.config.prioritization.alignment.roadmap_areas:
                if item.location.file.startswith(area["path"]):
                    base_impact *= area["multiplier"]
                    break

        return min(base_impact, 1.0)

    async def _prioritize(self, items: list[DebtItem]) -> list[PrioritizedDebt]:
        """Stage 3: Score and prioritize debt items."""

        weights = self.config.prioritization.weights
        prioritized = []

        for item in items:
            # Calculate composite score
            score = (
                weights.business_impact * item.impact.business_impact +
                weights.remediation_effort * (1 - item.estimated_effort / 40) +  # Inverse
                weights.bug_correlation * item.impact.bug_correlation +
                weights.change_risk * item.impact.change_risk +
                weights.team_pain * item.impact.team_pain
            )

            # Determine category
            category = self._categorize_score(score)

            prioritized.append(PrioritizedDebt(
                item=item,
                score=score,
                category=category,
                score_breakdown={
                    "business_impact": item.impact.business_impact,
                    "effort_score": 1 - item.estimated_effort / 40,
                    "bug_correlation": item.impact.bug_correlation,
                    "change_risk": item.impact.change_risk,
                    "team_pain": item.impact.team_pain
                }
            ))

        # Sort by score descending
        prioritized.sort(key=lambda x: x.score, reverse=True)

        return prioritized

    async def _generate_plan(
        self,
        prioritized: list[PrioritizedDebt]
    ) -> RemediationPlan:
        """Stage 4: Generate actionable remediation plan."""

        plan = RemediationPlan()

        # Quick wins (high score, low effort)
        plan.quick_wins = [
            p for p in prioritized
            if p.score >= 0.6 and p.item.estimated_effort <= 4
        ][:5]

        # Critical items (must address)
        plan.critical = [
            p for p in prioritized
            if p.category == "critical"
        ]

        # Sprint candidates (next 2 weeks)
        sprint_capacity = 40  # Story points
        plan.next_sprint = self._fit_to_capacity(
            [p for p in prioritized if p.category in ("critical", "high")],
            sprint_capacity
        )

        # Quarterly roadmap
        plan.quarterly = self._plan_quarterly(prioritized)

        # Track improvements
        plan.expected_improvements = self._calculate_improvements(plan)

        return plan

    def _fit_to_capacity(
        self,
        items: list[PrioritizedDebt],
        capacity: int
    ) -> list[PrioritizedDebt]:
        """Fit items into sprint capacity."""

        selected = []
        remaining_capacity = capacity

        for item in items:
            if item.item.estimated_effort <= remaining_capacity:
                selected.append(item)
                remaining_capacity -= item.item.estimated_effort

        return selected


class TechDebtScanner:
    """Scanner for various tech debt indicators."""

    async def scan_code_smells(
        self,
        rules: list[str]
    ) -> list[dict]:
        """Detect code smells."""

        findings = []

        for rule in rules:
            if rule == "long_method":
                findings.extend(await self._find_long_methods())
            elif rule == "large_class":
                findings.extend(await self._find_large_classes())
            elif rule == "god_object":
                findings.extend(await self._find_god_objects())
            elif rule == "shotgun_surgery":
                findings.extend(await self._find_shotgun_surgery())

        return findings

    async def _find_god_objects(self) -> list[dict]:
        """Find god objects (classes that do too much)."""

        # Use AST analysis
        findings = []

        for file in self._get_source_files():
            classes = await self._parse_classes(file)

            for cls in classes:
                # God object indicators
                indicators = {
                    "method_count": len(cls.methods),
                    "dependency_count": len(cls.dependencies),
                    "loc": cls.lines_of_code,
                    "responsibility_count": self._estimate_responsibilities(cls)
                }

                if (indicators["method_count"] > 20 or
                    indicators["dependency_count"] > 10 or
                    indicators["responsibility_count"] > 3):
                    findings.append({
                        "type": "god_object",
                        "file": file,
                        "class": cls.name,
                        "indicators": indicators,
                        "suggestion": f"Consider splitting {cls.name} into smaller, focused classes"
                    })

        return findings
```

---

## Report Format

```markdown
# Technical Debt Report

**Generated**: 2024-01-15
**Scan Coverage**: 1,234 files analyzed
**Total Debt Items**: 156

## Executive Summary

| Category | Count | Estimated Effort |
|----------|-------|------------------|
| Critical | 8     | 45 hours        |
| High     | 23    | 120 hours       |
| Medium   | 67    | 280 hours       |
| Low      | 58    | 200 hours       |

**Total Estimated Effort**: 645 hours (~16 weeks)

### Key Findings

1. **Security vulnerabilities** in 3 dependencies require immediate attention
2. **Payment module** has highest complexity debt (affects roadmap priority)
3. **Test coverage** below threshold in 12 modules

---

## Quick Wins (High Impact, Low Effort)

### 1. Update vulnerable dependency: lodash
**Effort**: 1 hour | **Impact Score**: 0.95

Current: 4.17.15 → Latest: 4.17.21
- Fixes prototype pollution vulnerability (CVE-2021-23337)
- No breaking changes

```bash
npm update lodash
```

### 2. Extract duplicated validation logic
**Effort**: 2 hours | **Impact Score**: 0.82

Found in:
- `src/api/users.ts:45`
- `src/api/payments.ts:67`
- `src/api/orders.ts:89`

Suggested: Create `src/utils/validation.ts`

---

## Critical Items

### CRIT-1: SQL Injection Vulnerability
**Location**: `src/db/queries.ts:123`
**Severity**: Critical
**Bug Correlation**: 0 (not yet exploited)

Raw SQL string concatenation detected. Must use parameterized queries.

**Current Code**:
```typescript
const query = `SELECT * FROM users WHERE id = '${userId}'`;
```

**Remediation**:
```typescript
const query = 'SELECT * FROM users WHERE id = $1';
await db.query(query, [userId]);
```

---

## Sprint Recommendations

### Next Sprint (40 points capacity)

| Item | Points | Score | Category |
|------|--------|-------|----------|
| CRIT-1: SQL Injection | 2 | 0.98 | Critical |
| CRIT-2: Auth bypass | 4 | 0.95 | Critical |
| HIGH-1: Payment complexity | 8 | 0.85 | High |
| HIGH-2: Duplicate handlers | 4 | 0.78 | High |

**Total**: 18 points (45% capacity)

**Remaining capacity** for feature work or additional debt.

---

## Quarterly Roadmap

### Q1 Focus: Security & Stability
- All critical security items
- Payment module refactoring
- Test coverage improvement

### Q2 Focus: Architecture
- Dependency cycle resolution
- Dead code removal
- Documentation updates

---

## Trends

### Debt Accumulation (6 months)
```
Jan: ████████ 89 items
Feb: █████████ 102 items
Mar: ██████████ 118 items
Apr: ███████████ 134 items
May: ███████████ 145 items
Jun: ████████████ 156 items
```

**Trend**: +12% monthly accumulation
**Recommendation**: Allocate 20% sprint capacity to debt reduction

---

## Appendix

### Methodology
- Complexity: Cyclomatic + Cognitive complexity analysis
- Duplication: Token-based detection (min 50 tokens)
- Impact: Weighted score based on bugs, changes, team pain

### Configuration Used
- Complexity threshold: Cyclomatic > 10, Cognitive > 15
- Coverage threshold: 80%
- Lookback period: 180 days for correlation analysis
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `scan.code.complexity.threshold.*` | int | varies | Complexity thresholds |
| `scan.code.duplication.min_tokens` | int | 50 | Min duplication size |
| `impact.sources.jira.lookback` | string | "180d" | Bug correlation period |
| `prioritization.weights.*` | float | varies | Scoring weights |

---

## Dependencies

### Required Tools
- `tool-tech-debt-scanner` - Code analysis
- `tool-dependency-graph` - Dependency analysis
- `tool-git-advanced` - Change history

### Optional
- `tool-jira-ops` - Bug correlation
- `tool-pr-context` - Review time analysis

---

## Open Questions

1. **Baseline tracking**: Track debt trends over time?
2. **Team allocation**: Suggest who should work on what?
3. **Automated fixing**: Auto-remediate simple issues?
4. **CI integration**: Block PRs that increase debt?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
