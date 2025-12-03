# tool-test-runner

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-tool-test-runner`

## Overview

Intelligent test execution with failure analysis, coverage tracking, and test selection optimization. Runs specific tests, analyzes failures, and suggests fixes based on error context.

### Value Proposition

| Without | With |
|---------|------|
| Run all tests hoping to catch the right one | Smart test selection based on changed files |
| Parse test output manually | Structured failure analysis with context |
| Hunt for relevant test files | "Run tests for PaymentGateway" finds them |
| Rerun flaky tests manually | Automatic flaky test detection and retry |

### Use Cases

1. **Targeted testing**: Run tests for specific functions/classes
2. **Failure analysis**: Understand why tests failed with context
3. **Change-based testing**: Run tests affected by recent changes
4. **Coverage analysis**: Track what's tested and what's not
5. **Flaky test management**: Detect and handle flaky tests
6. **Performance tracking**: Monitor test execution times

---

## Contract

### Tool Definition

```python
TOOL_DEFINITION = {
    "name": "test_runner",
    "description": """
    Intelligent test execution and analysis.

    Operations:
    - run: Execute tests (all, specific, or affected by changes)
    - analyze: Analyze test failures with context
    - coverage: Get coverage information
    - list: List available tests
    - history: Test execution history and trends

    Supports pytest, jest, go test, and other common frameworks.
    """,
    "parameters": {
        "type": "object",
        "properties": {
            "operation": {
                "type": "string",
                "enum": ["run", "analyze", "coverage", "list", "history"],
                "description": "Operation to perform"
            },
            "target": {
                "type": "string",
                "description": "Test target: file path, test name, or symbol name"
            },
            "selection": {
                "type": "string",
                "enum": ["all", "failed", "affected", "target"],
                "default": "target",
                "description": "Test selection strategy"
            },
            "framework": {
                "type": "string",
                "enum": ["auto", "pytest", "jest", "go", "cargo"],
                "default": "auto",
                "description": "Test framework (auto-detected if not specified)"
            },
            "options": {
                "type": "object",
                "description": "Framework-specific options (verbose, parallel, etc.)"
            },
            "analyze_failures": {
                "type": "boolean",
                "default": true,
                "description": "Include failure analysis in results"
            }
        },
        "required": ["operation"]
    }
}
```

### Output Schema

```python
@dataclass
class TestRunnerOutput:
    operation: str

    # For run operation
    run_result: TestRunResult | None = None

    # For analyze operation
    failure_analysis: list[FailureAnalysis] | None = None

    # For coverage operation
    coverage: CoverageReport | None = None

    # For list operation
    tests: list[TestInfo] | None = None

    # For history operation
    history: TestHistory | None = None

@dataclass
class TestRunResult:
    success: bool
    total: int
    passed: int
    failed: int
    skipped: int
    duration_seconds: float

    # Detailed results
    test_results: list[TestResult]

    # Failure analysis (if analyze_failures=True)
    failure_analyses: list[FailureAnalysis] | None

    # Framework info
    framework: str
    command: str

@dataclass
class TestResult:
    name: str
    file_path: str
    status: str                         # passed | failed | skipped | error
    duration_seconds: float
    output: str | None                  # Captured output
    error: TestError | None             # If failed

@dataclass
class TestError:
    type: str                           # AssertionError, TypeError, etc.
    message: str
    traceback: str
    location: Location                  # File and line where error occurred

@dataclass
class FailureAnalysis:
    test_name: str
    error: TestError

    # Analysis
    failure_type: str                   # assertion | exception | timeout | fixture
    root_cause: str                     # AI-analyzed root cause
    relevant_code: list[CodeSnippet]    # Related code snippets
    suggested_fix: str | None           # If determinable
    similar_failures: list[str]         # Other tests with similar failures

@dataclass
class CoverageReport:
    total_lines: int
    covered_lines: int
    coverage_percent: float
    files: list[FileCoverage]
    uncovered_functions: list[str]      # Functions with 0% coverage

@dataclass
class FileCoverage:
    path: str
    lines: int
    covered: int
    percent: float
    uncovered_lines: list[int]          # Line numbers not covered

@dataclass
class TestInfo:
    name: str
    file_path: str
    line_number: int
    markers: list[str]                  # pytest markers, jest tags, etc.
    parameters: list[str] | None        # Parameterized test variants
```

---

## Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                       tool-test-runner                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │    Test     │───▶│  Framework  │───▶│  Result     │         │
│  │  Selector   │    │   Runner    │    │  Analyzer   │         │
│  └─────────────┘    └──────┬──────┘    └─────────────┘         │
│                            │                                    │
│         ┌──────────────────┼──────────────────┐                │
│         ▼                  ▼                  ▼                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │   pytest    │    │    jest     │    │   go test   │         │
│  │   adapter   │    │   adapter   │    │   adapter   │         │
│  └─────────────┘    └─────────────┘    └─────────────┘         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Internal Components

#### 1. Test Selector

```python
class TestSelector:
    """Select tests to run based on strategy."""

    async def select(
        self,
        strategy: str,
        target: str | None,
        framework: str
    ) -> list[str]:
        if strategy == "all":
            return []  # Empty = run all

        if strategy == "target":
            return await self._select_for_target(target, framework)

        if strategy == "affected":
            return await self._select_affected()

        if strategy == "failed":
            return await self._select_previously_failed()

    async def _select_for_target(self, target: str, framework: str) -> list[str]:
        """Find tests for a target symbol or file."""

        # If target is a test file, return it
        if self._is_test_file(target):
            return [target]

        # If target is a symbol (class/function), find related tests
        symbol_tests = await self._find_tests_for_symbol(target)
        if symbol_tests:
            return symbol_tests

        # If target is a source file, find tests that import it
        if Path(target).exists():
            return await self._find_tests_importing(target)

        # Search by name pattern
        return await self._find_tests_by_pattern(target)

    async def _select_affected(self) -> list[str]:
        """Select tests affected by recent changes."""
        # Get changed files
        changed = await self.git.get_changed_files()

        affected_tests = set()
        for file_path in changed:
            # Direct test file changes
            if self._is_test_file(file_path):
                affected_tests.add(file_path)
            else:
                # Find tests that depend on this file
                tests = await self._find_tests_importing(file_path)
                affected_tests.update(tests)

        return list(affected_tests)
```

#### 2. Framework Adapters

```python
class PytestAdapter:
    """Adapter for pytest."""

    async def run(
        self,
        tests: list[str],
        options: dict
    ) -> TestRunResult:
        # Build command
        cmd = ["pytest", "--tb=short", "-v"]

        if options.get("parallel"):
            cmd.extend(["-n", "auto"])
        if options.get("coverage"):
            cmd.extend(["--cov", "--cov-report=json"])
        if options.get("markers"):
            cmd.extend(["-m", options["markers"]])

        cmd.extend(tests)

        # Run pytest
        result = await self._run_command(cmd)

        # Parse output
        return self._parse_result(result)

    def _parse_result(self, result: ProcessResult) -> TestRunResult:
        """Parse pytest output into structured result."""
        # Parse pytest's output format
        # Also look for pytest-json-report output if available
        ...

class JestAdapter:
    """Adapter for Jest."""

    async def run(self, tests: list[str], options: dict) -> TestRunResult:
        cmd = ["npx", "jest", "--json"]

        if tests:
            cmd.extend(["--testPathPattern", "|".join(tests)])
        if options.get("coverage"):
            cmd.append("--coverage")
        if options.get("watch") is False:
            cmd.append("--watchAll=false")

        result = await self._run_command(cmd)
        return self._parse_json_result(result)
```

#### 3. Result Analyzer

```python
class ResultAnalyzer:
    """Analyze test failures."""

    async def analyze_failures(
        self,
        results: list[TestResult]
    ) -> list[FailureAnalysis]:
        failures = [r for r in results if r.status == "failed"]
        analyses = []

        for failure in failures:
            analysis = await self._analyze_failure(failure)
            analyses.append(analysis)

        return analyses

    async def _analyze_failure(self, failure: TestResult) -> FailureAnalysis:
        error = failure.error

        # Classify failure type
        failure_type = self._classify_failure(error)

        # Get relevant code context
        relevant_code = await self._get_relevant_code(error)

        # Analyze root cause
        root_cause = await self._analyze_root_cause(
            failure, error, relevant_code
        )

        # Generate fix suggestion if possible
        suggested_fix = await self._suggest_fix(
            failure_type, error, relevant_code
        )

        # Find similar failures
        similar = await self._find_similar_failures(error)

        return FailureAnalysis(
            test_name=failure.name,
            error=error,
            failure_type=failure_type,
            root_cause=root_cause,
            relevant_code=relevant_code,
            suggested_fix=suggested_fix,
            similar_failures=similar
        )

    def _classify_failure(self, error: TestError) -> str:
        """Classify the type of test failure."""
        if "AssertionError" in error.type:
            return "assertion"
        if "Timeout" in error.type or "timeout" in error.message.lower():
            return "timeout"
        if "fixture" in error.message.lower():
            return "fixture"
        return "exception"

    async def _suggest_fix(
        self,
        failure_type: str,
        error: TestError,
        relevant_code: list[CodeSnippet]
    ) -> str | None:
        """Suggest a fix based on failure analysis."""

        # Common assertion fixes
        if failure_type == "assertion":
            if "assertEqual" in error.message or "==" in error.message:
                # Extract expected vs actual
                return self._suggest_assertion_fix(error)

        # Fixture issues
        if failure_type == "fixture":
            if "not found" in error.message.lower():
                return f"Fixture not defined. Add @pytest.fixture or import from conftest.py"

        # Type errors often have clear fixes
        if "TypeError" in error.type:
            return self._suggest_type_error_fix(error)

        return None
```

---

## Configuration

```yaml
tool-test-runner:
  # Framework detection
  frameworks:
    # Auto-detection patterns
    detect:
      pytest:
        - "pytest.ini"
        - "pyproject.toml:pytest"
        - "conftest.py"
      jest:
        - "jest.config.js"
        - "package.json:jest"
      go:
        - "go.mod"
        - "*_test.go"

  # Default options per framework
  defaults:
    pytest:
      verbose: true
      tb: "short"
      parallel: false
    jest:
      verbose: true
      coverage: false

  # Test selection
  selection:
    # Patterns for test files
    test_file_patterns:
      - "**/test_*.py"
      - "**/*_test.py"
      - "**/*.test.ts"
      - "**/*.spec.ts"
      - "**/*_test.go"

    # Exclude patterns
    exclude_patterns:
      - "**/node_modules/**"
      - "**/.venv/**"

  # Failure analysis
  analysis:
    enabled: true
    max_relevant_files: 5
    include_similar_failures: true

  # History tracking
  history:
    enabled: true
    retention_days: 30
    track_flaky: true
```

---

## Examples

### Example 1: Run Tests for Symbol

```python
# Input
{
    "operation": "run",
    "target": "PaymentGateway",
    "selection": "target",
    "analyze_failures": true
}

# Output
{
    "operation": "run",
    "run_result": {
        "success": false,
        "total": 8,
        "passed": 7,
        "failed": 1,
        "skipped": 0,
        "duration_seconds": 2.34,
        "test_results": [...],
        "failure_analyses": [
            {
                "test_name": "test_payment_gateway_timeout",
                "error": {
                    "type": "AssertionError",
                    "message": "Expected retry count 3, got 2",
                    "location": {"file_path": "tests/test_payments.py", "line_number": 45}
                },
                "failure_type": "assertion",
                "root_cause": "PaymentGateway.process() only retries twice. The retry decorator has max_attempts=2 but test expects 3.",
                "relevant_code": [
                    {
                        "file": "src/payments/gateway.py",
                        "line": 42,
                        "code": "@retry(max_attempts=2)  # Should be 3?"
                    }
                ],
                "suggested_fix": "Update max_attempts to 3 in gateway.py:42 or update test expectation to 2"
            }
        ],
        "framework": "pytest",
        "command": "pytest tests/test_payments.py -k PaymentGateway -v"
    }
}
```

### Example 2: Run Affected Tests

```python
# Input
{
    "operation": "run",
    "selection": "affected"
}

# Output
{
    "operation": "run",
    "run_result": {
        "success": true,
        "total": 12,
        "passed": 12,
        "failed": 0,
        "duration_seconds": 4.56,
        "framework": "pytest",
        "command": "pytest tests/test_auth.py tests/test_users.py -v"
    }
}
```

### Example 3: Coverage Report

```python
# Input
{
    "operation": "coverage",
    "target": "src/payments/"
}

# Output
{
    "operation": "coverage",
    "coverage": {
        "total_lines": 450,
        "covered_lines": 387,
        "coverage_percent": 86.0,
        "files": [
            {
                "path": "src/payments/gateway.py",
                "lines": 120,
                "covered": 115,
                "percent": 95.8,
                "uncovered_lines": [45, 67, 89, 90, 91]
            }
        ],
        "uncovered_functions": [
            "PaymentGateway._handle_timeout",
            "PaymentGateway._legacy_fallback"
        ]
    }
}
```

---

## Security Considerations

- Executes test commands (potential code execution)
- Reads test files and source code
- May expose test data in output

### Permissions

```python
REQUIRED_CAPABILITIES = [
    "filesystem:read",      # Read test files
    "process:execute",      # Run test commands
]
```

---

## Dependencies

### Required
- `pytest` (for Python projects)
- `jest` (for JavaScript/TypeScript projects)

### Optional
- `pytest-cov` - Coverage reporting
- `pytest-xdist` - Parallel execution
- `pytest-json-report` - Structured output

---

## Open Questions

1. **Framework support**: Which frameworks beyond pytest/jest/go?
2. **Parallel execution**: Default parallel or sequential?
3. **Coverage thresholds**: Should we enforce minimum coverage?
4. **Test generation**: Should this tool suggest missing tests?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
