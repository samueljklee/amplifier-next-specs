# hooks-style-enforcer

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-hooks-style-enforcer`

## Overview

Automatically runs linters and formatters on code as it's written, injecting feedback for the agent to fix issues immediately. Ensures all generated code meets team style standards without manual intervention.

### Value Proposition

| Without | With |
|---------|------|
| Linting issues found in CI | Fixed during generation |
| Style debates in PR reviews | Automated enforcement |
| Inconsistent code style | Consistent from creation |

---

## Contract

### Hook Configuration

```yaml
hooks:
  - module: hooks-style-enforcer
    config:
      # Linters by language
      linters:
        python:
          enabled: true
          tools:
            - name: ruff
              command: "ruff check --fix {file}"
              auto_fix: true
            - name: black
              command: "black --check {file}"
              auto_fix_command: "black {file}"
            - name: mypy
              command: "mypy {file}"
              auto_fix: false

        typescript:
          enabled: true
          tools:
            - name: eslint
              command: "eslint {file}"
              auto_fix_command: "eslint --fix {file}"
            - name: prettier
              command: "prettier --check {file}"
              auto_fix_command: "prettier --write {file}"

        go:
          enabled: true
          tools:
            - name: gofmt
              command: "gofmt -l {file}"
              auto_fix_command: "gofmt -w {file}"
            - name: golint
              command: "golint {file}"

      # Behavior
      behavior:
        run_on: post                    # pre | post (after file write)
        auto_fix: true                  # Auto-apply fixes when possible
        inject_feedback: true           # Tell agent about remaining issues
        fail_on_error: false            # Don't block writes on lint errors

      # Feedback settings
      feedback:
        max_issues: 10                  # Limit issues in feedback
        include_suggestions: true       # Include fix suggestions
        severity_threshold: warning     # error | warning | info
```

### HookResult Actions

```python
# After auto-fixing, inject remaining issues
HookResult(
    action="inject_context",
    context_injection="""
Style check results for src/payments/gateway.py:

**Auto-fixed:**
- Formatted with black
- Fixed 3 ruff issues (unused imports, line length)

**Remaining issues (please fix):**
1. Line 42: mypy error: Argument 1 to "process" has incompatible type "str"; expected "int"
2. Line 67: mypy error: Missing return statement

Please address the type errors above.
""",
    user_message="Style: auto-fixed 4 issues, 2 type errors need attention",
    user_message_level="warning"
)
```

---

## Architecture

```python
class StyleEnforcerHook:
    """Enforce code style with linters and formatters."""

    def __init__(self, config: StyleEnforcerConfig):
        self.config = config
        self.linters = self._initialize_linters()

    async def __call__(self, event: str, data: dict) -> HookResult:
        # Only process file writes
        if event != "tool:post" or not self._is_code_write(data):
            return HookResult(action="continue")

        file_path = data.get("file_path", "")
        language = self._detect_language(file_path)

        if not language or language not in self.config.linters:
            return HookResult(action="continue")

        # Run linters
        results = await self._run_linters(file_path, language)

        # Auto-fix if configured
        auto_fixed = []
        if self.config.behavior.auto_fix:
            auto_fixed = await self._apply_auto_fixes(file_path, language)

        # Collect remaining issues
        remaining = [r for r in results if not r.auto_fixed]

        # Filter by severity
        remaining = self._filter_by_severity(remaining)

        # Generate feedback
        if auto_fixed or remaining:
            return self._generate_feedback(file_path, auto_fixed, remaining)

        return HookResult(action="continue")

    async def _run_linters(self, file_path: str, language: str) -> list[LintResult]:
        """Run all configured linters for language."""
        results = []
        linter_config = self.config.linters[language]

        for tool in linter_config.tools:
            command = tool.command.format(file=file_path)
            try:
                proc = await asyncio.create_subprocess_shell(
                    command,
                    stdout=asyncio.subprocess.PIPE,
                    stderr=asyncio.subprocess.PIPE
                )
                stdout, stderr = await proc.communicate()

                if proc.returncode != 0:
                    issues = self._parse_output(tool.name, stdout.decode(), stderr.decode())
                    results.extend(issues)
            except Exception as e:
                # Log but don't fail on linter errors
                pass

        return results

    async def _apply_auto_fixes(self, file_path: str, language: str) -> list[str]:
        """Apply auto-fixes from formatters."""
        fixed = []
        linter_config = self.config.linters[language]

        for tool in linter_config.tools:
            if tool.auto_fix and tool.auto_fix_command:
                command = tool.auto_fix_command.format(file=file_path)
                try:
                    proc = await asyncio.create_subprocess_shell(command)
                    await proc.wait()
                    if proc.returncode == 0:
                        fixed.append(tool.name)
                except Exception:
                    pass

        return fixed

    def _generate_feedback(
        self,
        file_path: str,
        auto_fixed: list[str],
        remaining: list[LintResult]
    ) -> HookResult:
        """Generate feedback for agent."""
        lines = [f"Style check results for {file_path}:", ""]

        if auto_fixed:
            lines.append("**Auto-fixed:**")
            for tool in auto_fixed:
                lines.append(f"- Applied {tool}")
            lines.append("")

        if remaining:
            lines.append(f"**Remaining issues ({len(remaining)}):**")
            for i, issue in enumerate(remaining[:self.config.feedback.max_issues], 1):
                lines.append(f"{i}. Line {issue.line}: {issue.tool}: {issue.message}")
                if self.config.feedback.include_suggestions and issue.suggestion:
                    lines.append(f"   Suggestion: {issue.suggestion}")
            lines.append("")
            lines.append("Please fix these issues.")

        severity = "warning" if any(r.severity == "error" for r in remaining) else "info"

        return HookResult(
            action="inject_context",
            context_injection="\n".join(lines),
            user_message=f"Style: {len(auto_fixed)} auto-fixed, {len(remaining)} need attention",
            user_message_level=severity
        )

    def _parse_output(self, tool: str, stdout: str, stderr: str) -> list[LintResult]:
        """Parse linter output into structured results."""
        # Each linter has different output format
        parsers = {
            "ruff": self._parse_ruff,
            "black": self._parse_black,
            "mypy": self._parse_mypy,
            "eslint": self._parse_eslint,
            "prettier": self._parse_prettier,
        }

        parser = parsers.get(tool, self._parse_generic)
        return parser(stdout, stderr)

    def _parse_ruff(self, stdout: str, stderr: str) -> list[LintResult]:
        """Parse ruff output."""
        results = []
        for line in stdout.split("\n"):
            # Format: file.py:10:5: E501 Line too long
            match = re.match(r".*:(\d+):\d+: (\w+) (.+)", line)
            if match:
                results.append(LintResult(
                    tool="ruff",
                    line=int(match.group(1)),
                    code=match.group(2),
                    message=match.group(3),
                    severity="warning",
                    auto_fixable=match.group(2) not in ["E999"]  # Syntax errors not auto-fixable
                ))
        return results
```

---

## Supported Linters

| Language | Tool | Auto-fix | Purpose |
|----------|------|----------|---------|
| Python | ruff | Yes | Fast linter (replaces flake8, isort) |
| Python | black | Yes | Code formatter |
| Python | mypy | No | Type checker |
| TypeScript | eslint | Yes | Linter |
| TypeScript | prettier | Yes | Formatter |
| Go | gofmt | Yes | Formatter |
| Go | golint | No | Linter |
| Rust | rustfmt | Yes | Formatter |
| Rust | clippy | Partial | Linter |

---

## Examples

### Example 1: Python Auto-Fix + Type Errors

```python
# Agent writes file with style issues and type errors
# Hook runs ruff, black, mypy

# After auto-fix:
"""
Style check results for src/calculator.py:

**Auto-fixed:**
- Applied black (formatting)
- Applied ruff (removed unused import, fixed line length)

**Remaining issues (2):**
1. Line 15: mypy: Argument "x" to "add" has incompatible type "str"; expected "int"
2. Line 23: mypy: Missing return type annotation for function "calculate"

Please fix these type errors.
"""

# Agent addresses type errors in next turn
```

### Example 2: All Issues Auto-Fixed

```python
# Only formatting issues, all auto-fixable

{
    "action": "inject_context",
    "context_injection": """
Style check results for src/utils.py:

**Auto-fixed:**
- Applied prettier (formatting)
- Applied eslint (3 issues fixed)

All issues resolved automatically.
""",
    "user_message": "Style: auto-fixed 4 issues",
    "user_message_level": "info"
}
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `linters.<lang>.enabled` | bool | true | Enable for language |
| `linters.<lang>.tools` | list | [] | Tools to run |
| `behavior.run_on` | string | "post" | When to run |
| `behavior.auto_fix` | bool | true | Auto-apply fixes |
| `behavior.inject_feedback` | bool | true | Inject remaining issues |
| `feedback.max_issues` | int | 10 | Max issues to show |
| `feedback.severity_threshold` | string | "warning" | Min severity |

---

## Security Considerations

- Linters execute on written files (code execution risk minimal)
- Auto-fix modifies files (intended behavior)
- Tool commands should be hardcoded, not user-configurable

---

## Dependencies

### Required
- Language-specific linters installed in environment

### Optional
- None (linters are external tools)

---

## Open Questions

1. **Linter conflicts**: What if black and ruff disagree?
2. **Performance**: Skip linting for large files?
3. **Custom rules**: Support project-specific lint configs?
4. **Pre vs post**: Should we lint before write to prevent bad code?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
