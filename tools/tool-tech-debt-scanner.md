# tool-tech-debt-scanner

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-tool-tech-debt-scanner`

## Overview

Identify and categorize technical debt across the codebase. Detects anti-patterns, outdated dependencies, dead code, code duplication, complexity hotspots, and TODO/FIXME comments. Provides prioritized recommendations for debt reduction.

### Value Proposition

| Without | With |
|---------|------|
| Tech debt accumulates invisibly | Quantified, visible debt inventory |
| "We'll fix it later" → never fixed | Prioritized backlog with impact estimates |
| Outdated deps discovered in incidents | Proactive vulnerability and freshness scanning |
| Gut feeling about code quality | Data-driven quality metrics |

### Use Cases

1. **Debt inventory**: "What tech debt exists in our codebase?"
2. **Sprint planning**: "What debt should we tackle this sprint?"
3. **Risk assessment**: "What's our exposure from outdated dependencies?"
4. **Refactoring targets**: "Where are the complexity hotspots?"
5. **Quality gates**: "Does this PR increase tech debt?"

---

## Contract

### Tool Definition

```python
TOOL_DEFINITION = {
    "name": "tech_debt_scanner",
    "description": """
    Scan codebase for technical debt and quality issues.

    Categories:
    - dependencies: Outdated, vulnerable, or deprecated deps
    - dead_code: Unused functions, classes, imports
    - complexity: High cyclomatic complexity, long functions
    - duplication: Duplicated code blocks
    - todos: TODO, FIXME, HACK comments
    - patterns: Anti-patterns, code smells
    - tests: Missing or inadequate test coverage

    Use for debt prioritization, sprint planning, and quality monitoring.
    """,
    "parameters": {
        "type": "object",
        "properties": {
            "scope": {
                "type": "string",
                "description": "Path pattern to scan (default: entire repo)"
            },
            "categories": {
                "type": "array",
                "items": {
                    "type": "string",
                    "enum": ["dependencies", "dead_code", "complexity", "duplication", "todos", "patterns", "tests", "all"]
                },
                "default": ["all"],
                "description": "Debt categories to scan"
            },
            "severity_threshold": {
                "type": "string",
                "enum": ["low", "medium", "high", "critical"],
                "default": "low",
                "description": "Minimum severity to report"
            },
            "include_metrics": {
                "type": "boolean",
                "default": true,
                "description": "Include code metrics in output"
            },
            "compare_to": {
                "type": "string",
                "description": "Git ref to compare against (for delta analysis)"
            }
        }
    }
}
```

### Output Schema

```python
@dataclass
class TechDebtScanOutput:
    scope: str
    scan_time: datetime
    categories_scanned: list[str]

    # Findings by category
    dependencies: DependencyDebt | None
    dead_code: DeadCodeDebt | None
    complexity: ComplexityDebt | None
    duplication: DuplicationDebt | None
    todos: TodoDebt | None
    patterns: PatternDebt | None
    tests: TestDebt | None

    # Summary
    summary: DebtSummary
    recommendations: list[Recommendation]

    # Delta (if compare_to specified)
    delta: DebtDelta | None

@dataclass
class DependencyDebt:
    outdated: list[OutdatedDep]
    vulnerable: list[VulnerableDep]
    deprecated: list[DeprecatedDep]
    unused: list[UnusedDep]

@dataclass
class OutdatedDep:
    name: str
    current_version: str
    latest_version: str
    age_days: int
    breaking_changes: bool
    update_effort: str                  # low | medium | high

@dataclass
class VulnerableDep:
    name: str
    version: str
    cve_ids: list[str]
    severity: str                       # low | medium | high | critical
    fixed_in: str | None
    description: str

@dataclass
class DeadCodeDebt:
    unused_functions: list[UnusedSymbol]
    unused_classes: list[UnusedSymbol]
    unused_imports: list[UnusedImport]
    unused_files: list[str]
    total_dead_lines: int

@dataclass
class UnusedSymbol:
    name: str
    file_path: str
    line_number: int
    lines_of_code: int
    confidence: float                   # 0-1, lower if might be used dynamically

@dataclass
class ComplexityDebt:
    hotspots: list[ComplexityHotspot]
    long_functions: list[LongFunction]
    deep_nesting: list[DeepNesting]
    average_complexity: float

@dataclass
class ComplexityHotspot:
    file_path: str
    function_name: str
    cyclomatic_complexity: int
    cognitive_complexity: int
    lines: int
    recommendation: str

@dataclass
class DuplicationDebt:
    duplicates: list[CodeDuplicate]
    total_duplicated_lines: int
    duplication_percentage: float

@dataclass
class CodeDuplicate:
    locations: list[DuplicateLocation]
    lines: int
    similarity: float                   # 0-1
    suggested_extraction: str | None

@dataclass
class TodoDebt:
    todos: list[TodoItem]
    by_age: dict[str, int]              # "< 1 month": 5, "1-6 months": 10, ...
    by_author: dict[str, int]

@dataclass
class TodoItem:
    type: str                           # TODO | FIXME | HACK | XXX
    text: str
    file_path: str
    line_number: int
    author: str | None                  # From git blame
    age_days: int | None
    linked_issue: str | None            # If references ticket

@dataclass
class PatternDebt:
    anti_patterns: list[AntiPattern]
    code_smells: list[CodeSmell]

@dataclass
class AntiPattern:
    name: str                           # "God Class", "Spaghetti Code", etc.
    file_path: str
    description: str
    severity: str
    fix_suggestion: str

@dataclass
class TestDebt:
    untested_functions: list[str]
    low_coverage_files: list[FileCoverage]
    missing_test_files: list[str]
    flaky_tests: list[str]

@dataclass
class DebtSummary:
    total_items: int
    by_severity: dict[str, int]
    by_category: dict[str, int]
    estimated_effort_hours: float
    debt_score: float                   # 0-100, lower is better
    trend: str                          # improving | stable | degrading

@dataclass
class Recommendation:
    priority: int                       # 1 = highest
    category: str
    title: str
    description: str
    effort: str
    impact: str
    affected_items: list[str]
```

---

## Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    tool-tech-debt-scanner                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐       │
│  │              Scanner Orchestrator                    │       │
│  └─────────────────────────────────────────────────────┘       │
│         │           │           │           │                   │
│         ▼           ▼           ▼           ▼                   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐              │
│  │   Dep   │ │  Dead   │ │  Dup    │ │ Pattern │ ...          │
│  │ Scanner │ │  Code   │ │ Finder  │ │ Checker │              │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘              │
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐       │
│  │           Prioritization Engine                      │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Scanner Components

```python
class DependencyScanner:
    """Scan for dependency-related tech debt."""

    async def scan(self, scope: str) -> DependencyDebt:
        # Find package manifests
        manifests = await self._find_manifests(scope)

        results = DependencyDebt(
            outdated=[], vulnerable=[], deprecated=[], unused=[]
        )

        for manifest in manifests:
            if manifest.type == "python":
                deps = await self._scan_python_deps(manifest)
            elif manifest.type == "npm":
                deps = await self._scan_npm_deps(manifest)
            # ... other package managers

            results.outdated.extend(deps.outdated)
            results.vulnerable.extend(deps.vulnerable)

        return results

    async def _scan_python_deps(self, manifest: Manifest) -> DependencyScanResult:
        # Check for outdated
        outdated = []
        for dep, version in manifest.dependencies.items():
            latest = await self._get_latest_version(dep, "pypi")
            if latest and version < latest:
                outdated.append(OutdatedDep(
                    name=dep,
                    current_version=str(version),
                    latest_version=str(latest),
                    age_days=self._calculate_age(version, latest),
                    breaking_changes=self._has_breaking_changes(version, latest),
                    update_effort=self._estimate_effort(dep, version, latest)
                ))

        # Check for vulnerabilities (via safety, pip-audit, etc.)
        vulnerable = await self._check_vulnerabilities(manifest)

        return DependencyScanResult(outdated=outdated, vulnerable=vulnerable)


class DeadCodeScanner:
    """Detect unused code."""

    async def scan(self, scope: str) -> DeadCodeDebt:
        # Build dependency graph
        graph = await self._build_dependency_graph(scope)

        # Find unreachable nodes
        entry_points = await self._find_entry_points(scope)
        reachable = self._compute_reachable(graph, entry_points)

        unused_functions = []
        unused_classes = []

        for symbol in graph.symbols:
            if symbol.id not in reachable:
                confidence = self._compute_confidence(symbol)

                item = UnusedSymbol(
                    name=symbol.name,
                    file_path=symbol.file_path,
                    line_number=symbol.line_number,
                    lines_of_code=symbol.lines,
                    confidence=confidence
                )

                if symbol.kind == "function":
                    unused_functions.append(item)
                elif symbol.kind == "class":
                    unused_classes.append(item)

        # Find unused imports
        unused_imports = await self._find_unused_imports(scope)

        return DeadCodeDebt(
            unused_functions=unused_functions,
            unused_classes=unused_classes,
            unused_imports=unused_imports,
            unused_files=self._find_unused_files(reachable),
            total_dead_lines=sum(s.lines_of_code for s in unused_functions + unused_classes)
        )


class PrioritizationEngine:
    """Prioritize debt items for remediation."""

    def prioritize(self, debt: TechDebtScanOutput) -> list[Recommendation]:
        recommendations = []

        # Critical vulnerabilities first
        for vuln in debt.dependencies.vulnerable:
            if vuln.severity == "critical":
                recommendations.append(Recommendation(
                    priority=1,
                    category="dependencies",
                    title=f"Critical vulnerability in {vuln.name}",
                    description=f"{vuln.description}. CVE: {', '.join(vuln.cve_ids)}",
                    effort="low" if vuln.fixed_in else "high",
                    impact="critical",
                    affected_items=[vuln.name]
                ))

        # High-impact complexity hotspots
        for hotspot in sorted(debt.complexity.hotspots,
                             key=lambda h: h.cyclomatic_complexity,
                             reverse=True)[:5]:
            recommendations.append(Recommendation(
                priority=2,
                category="complexity",
                title=f"Refactor {hotspot.function_name}",
                description=f"Cyclomatic complexity: {hotspot.cyclomatic_complexity}",
                effort="medium",
                impact="high",
                affected_items=[f"{hotspot.file_path}:{hotspot.function_name}"]
            ))

        # Large dead code blocks
        significant_dead = [d for d in debt.dead_code.unused_functions
                          if d.lines_of_code > 50 and d.confidence > 0.8]
        if significant_dead:
            recommendations.append(Recommendation(
                priority=3,
                category="dead_code",
                title=f"Remove {len(significant_dead)} unused functions",
                description=f"~{sum(d.lines_of_code for d in significant_dead)} lines of dead code",
                effort="low",
                impact="medium",
                affected_items=[d.name for d in significant_dead]
            ))

        return sorted(recommendations, key=lambda r: r.priority)
```

---

## Configuration

```yaml
tool-tech-debt-scanner:
  # Scanning settings
  scan:
    # File patterns
    include:
      - "**/*.py"
      - "**/*.ts"
      - "**/*.js"
    exclude:
      - "**/node_modules/**"
      - "**/.venv/**"
      - "**/dist/**"

  # Dependency scanning
  dependencies:
    # Vulnerability databases
    vuln_sources:
      - safety           # Python
      - npm-audit        # npm
      - snyk             # Multi-language

    # Freshness thresholds
    outdated_thresholds:
      minor_days: 90     # Consider outdated after 90 days
      major_days: 180

  # Complexity thresholds
  complexity:
    cyclomatic_warn: 10
    cyclomatic_error: 20
    cognitive_warn: 15
    cognitive_error: 25
    function_length_warn: 50
    function_length_error: 100

  # Dead code detection
  dead_code:
    # Entry points to consider as roots
    entry_points:
      - "main"
      - "cli"
      - "**/test_*.py"
      - "**/*_test.py"

    # Patterns that might indicate dynamic usage
    dynamic_patterns:
      - "@app.route"
      - "@pytest.fixture"
      - "__getattr__"

  # Duplication detection
  duplication:
    min_lines: 6           # Minimum lines to consider duplicate
    min_tokens: 50         # Minimum tokens
    similarity_threshold: 0.9

  # TODO scanning
  todos:
    patterns:
      - "TODO"
      - "FIXME"
      - "HACK"
      - "XXX"
      - "TECHNICAL_DEBT"
    # Extract ticket references
    ticket_patterns:
      - "[A-Z]+-\\d+"      # JIRA-style
```

---

## Examples

### Example 1: Full Scan

```python
# Input
{
    "scope": "src/",
    "categories": ["all"],
    "severity_threshold": "medium"
}

# Output
{
    "dependencies": {
        "outdated": [
            {
                "name": "requests",
                "current_version": "2.25.0",
                "latest_version": "2.31.0",
                "age_days": 450,
                "breaking_changes": false,
                "update_effort": "low"
            }
        ],
        "vulnerable": [
            {
                "name": "pyyaml",
                "version": "5.3",
                "cve_ids": ["CVE-2020-14343"],
                "severity": "critical",
                "fixed_in": "5.4",
                "description": "Arbitrary code execution via yaml.load()"
            }
        ]
    },
    "dead_code": {
        "unused_functions": [
            {
                "name": "legacy_auth_handler",
                "file_path": "src/auth/legacy.py",
                "line_number": 45,
                "lines_of_code": 78,
                "confidence": 0.95
            }
        ],
        "total_dead_lines": 234
    },
    "complexity": {
        "hotspots": [
            {
                "file_path": "src/payments/processor.py",
                "function_name": "process_payment",
                "cyclomatic_complexity": 32,
                "cognitive_complexity": 45,
                "lines": 156,
                "recommendation": "Split into smaller functions by payment type"
            }
        ]
    },
    "todos": {
        "todos": [
            {
                "type": "FIXME",
                "text": "This is a race condition waiting to happen",
                "file_path": "src/cache/manager.py",
                "line_number": 89,
                "author": "alice",
                "age_days": 342
            }
        ],
        "by_age": {"< 1 month": 3, "1-6 months": 12, "> 6 months": 8}
    },
    "summary": {
        "total_items": 47,
        "by_severity": {"critical": 1, "high": 5, "medium": 18, "low": 23},
        "by_category": {"dependencies": 8, "dead_code": 12, "complexity": 5, "todos": 22},
        "estimated_effort_hours": 40,
        "debt_score": 67,
        "trend": "stable"
    },
    "recommendations": [
        {
            "priority": 1,
            "category": "dependencies",
            "title": "Critical vulnerability in pyyaml",
            "description": "CVE-2020-14343: Arbitrary code execution",
            "effort": "low",
            "impact": "critical"
        },
        {
            "priority": 2,
            "category": "complexity",
            "title": "Refactor process_payment",
            "description": "Cyclomatic complexity: 32 (threshold: 20)",
            "effort": "medium",
            "impact": "high"
        }
    ]
}
```

---

## Security Considerations

- Reads source code and dependencies
- Vulnerability data fetched from external sources
- Scan results may reveal security weaknesses

---

## Dependencies

### Required
- `tree-sitter` - Code analysis
- `radon` - Python complexity (or language-specific tools)

### Optional
- `safety` - Python vulnerability scanning
- `npm-audit` - npm vulnerability scanning
- `jscpd` - Cross-language duplication detection

---

## Open Questions

1. **Historical tracking**: Should we store scan history for trend analysis?
2. **CI integration**: Block PRs that increase debt beyond threshold?
3. **Custom rules**: Allow project-specific anti-pattern definitions?
4. **Effort estimation**: How to improve effort estimates?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
