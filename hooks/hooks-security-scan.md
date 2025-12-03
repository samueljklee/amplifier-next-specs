# hooks-security-scan

> **Priority**: P0 (Foundation)
> **Status**: Draft
> **Module**: `amplifier-module-hooks-security-scan`

## Overview

Real-time security scanning of code as it's written. Detects vulnerabilities, secrets, and security anti-patterns before they're committed. Can block dangerous operations or inject warnings for the agent to address.

### Value Proposition

| Without | With |
|---------|------|
| Security issues found in PR review | Caught immediately as code is written |
| Secrets committed accidentally | Blocked before file write |
| Security debt accumulates | Issues fixed in the same session |

### Trigger Events

| Event | Action | Purpose |
|-------|--------|---------|
| `tool:pre` (file write) | Scan content before write | Block if critical vulnerability |
| `tool:post` (file write) | Scan and report | Inject feedback for agent to fix |
| `session:start` | Scan existing codebase | Baseline security context |

---

## Contract

### Hook Configuration

```yaml
hooks:
  - module: hooks-security-scan
    config:
      # Scan modes
      mode: block                       # block | warn | report
      severity_threshold: high          # Block/warn at this level and above

      # Scanners to enable
      scanners:
        secrets: true                   # API keys, passwords, tokens
        sast: true                      # Static analysis (SQL injection, XSS, etc.)
        dependencies: true              # Known vulnerable dependencies
        patterns: true                  # Custom anti-patterns

      # Secret detection
      secrets:
        patterns:
          - "aws_access_key_id"
          - "api[_-]?key"
          - "password\\s*=\\s*[\"'][^\"']+[\"']"
          - "-----BEGIN.*PRIVATE KEY-----"
        allow_patterns:
          - "test_api_key"
          - "example_password"

      # SAST rules
      sast:
        rules:
          - sql_injection
          - xss
          - path_traversal
          - command_injection
          - insecure_deserialization

      # Actions
      actions:
        critical:
          action: deny
          message: "Critical security vulnerability detected"
        high:
          action: inject_context
          message: "Security issue found - please fix"
        medium:
          action: inject_context
          suppress_output: true
        low:
          action: continue
```

### HookResult Actions

```python
# Block dangerous writes
HookResult(
    action="deny",
    reason="AWS credentials detected in file. Remove before saving."
)

# Inject feedback for agent to fix
HookResult(
    action="inject_context",
    context_injection="""
Security scan found issues in your code:

1. **SQL Injection (HIGH)** at line 42:
   ```python
   query = f"SELECT * FROM users WHERE id = {user_id}"
   ```
   Fix: Use parameterized queries

2. **Hardcoded secret (CRITICAL)** at line 15:
   ```python
   API_KEY = "sk_live_..."
   ```
   Fix: Use environment variable

Please fix these issues before proceeding.
""",
    user_message="Found 2 security issues",
    user_message_level="warning"
)
```

---

## Architecture

### Scanner Components

```python
class SecurityScanHook:
    """Real-time security scanning hook."""

    def __init__(self, config: SecurityScanConfig):
        self.config = config
        self.scanners = []

        if config.scanners.secrets:
            self.scanners.append(SecretScanner(config.secrets))
        if config.scanners.sast:
            self.scanners.append(SASTScanner(config.sast))
        if config.scanners.dependencies:
            self.scanners.append(DependencyScanner())
        if config.scanners.patterns:
            self.scanners.append(PatternScanner(config.patterns))

    async def __call__(self, event: str, data: dict) -> HookResult:
        if event == "tool:pre" and self._is_file_write(data):
            return await self._scan_before_write(data)

        if event == "tool:post" and self._is_file_write(data):
            return await self._scan_after_write(data)

        return HookResult(action="continue")

    async def _scan_before_write(self, data: dict) -> HookResult:
        content = data.get("content", "")
        file_path = data.get("file_path", "")

        findings = await self._scan(content, file_path)

        # Block critical issues
        critical = [f for f in findings if f.severity == "critical"]
        if critical and self.config.mode == "block":
            return HookResult(
                action="deny",
                reason=self._format_critical_findings(critical)
            )

        return HookResult(action="continue")

    async def _scan_after_write(self, data: dict) -> HookResult:
        content = data.get("content", "")
        file_path = data.get("file_path", "")

        findings = await self._scan(content, file_path)

        if not findings:
            return HookResult(action="continue")

        # Filter by threshold
        reportable = [f for f in findings
                     if self._severity_meets_threshold(f.severity)]

        if not reportable:
            return HookResult(action="continue")

        return HookResult(
            action="inject_context",
            context_injection=self._format_findings(reportable),
            user_message=f"Security scan: {len(reportable)} issue(s) found",
            user_message_level="warning" if any(f.severity in ("high", "critical") for f in reportable) else "info"
        )

    async def _scan(self, content: str, file_path: str) -> list[Finding]:
        findings = []
        for scanner in self.scanners:
            scanner_findings = await scanner.scan(content, file_path)
            findings.extend(scanner_findings)
        return findings
```

### Secret Scanner

```python
class SecretScanner:
    """Detect secrets and credentials in code."""

    DEFAULT_PATTERNS = [
        # AWS
        (r'AKIA[0-9A-Z]{16}', 'AWS Access Key ID'),
        (r'aws_secret_access_key\s*=\s*["\'][^"\']+["\']', 'AWS Secret Key'),

        # Generic
        (r'api[_-]?key\s*[=:]\s*["\'][^"\']{10,}["\']', 'API Key'),
        (r'password\s*[=:]\s*["\'][^"\']+["\']', 'Hardcoded Password'),
        (r'secret\s*[=:]\s*["\'][^"\']{10,}["\']', 'Hardcoded Secret'),

        # Private keys
        (r'-----BEGIN\s+(?:RSA\s+)?PRIVATE\s+KEY-----', 'Private Key'),

        # Tokens
        (r'ghp_[a-zA-Z0-9]{36}', 'GitHub Personal Access Token'),
        (r'sk-[a-zA-Z0-9]{48}', 'OpenAI API Key'),
        (r'xox[baprs]-[0-9a-zA-Z]{10,}', 'Slack Token'),
    ]

    async def scan(self, content: str, file_path: str) -> list[Finding]:
        findings = []

        for pattern, name in self.patterns:
            matches = re.finditer(pattern, content, re.IGNORECASE)
            for match in matches:
                # Skip if in allow list
                if self._is_allowed(match.group(), file_path):
                    continue

                line_num = content[:match.start()].count('\n') + 1
                findings.append(Finding(
                    type="secret",
                    severity="critical",
                    title=f"{name} detected",
                    location=f"{file_path}:{line_num}",
                    code_snippet=self._get_snippet(content, match),
                    recommendation="Remove secret and use environment variables"
                ))

        return findings
```

### SAST Scanner

```python
class SASTScanner:
    """Static Application Security Testing."""

    RULES = {
        "sql_injection": {
            "patterns": [
                r'execute\s*\(\s*["\'].*%s',
                r'execute\s*\(\s*f["\']',
                r'cursor\.execute\s*\(\s*["\'].*\+',
            ],
            "severity": "high",
            "message": "Potential SQL injection",
            "fix": "Use parameterized queries"
        },
        "xss": {
            "patterns": [
                r'innerHTML\s*=\s*[^"\'`]',
                r'document\.write\s*\(',
                r'dangerouslySetInnerHTML',
            ],
            "severity": "high",
            "message": "Potential XSS vulnerability",
            "fix": "Sanitize user input before rendering"
        },
        "command_injection": {
            "patterns": [
                r'os\.system\s*\(',
                r'subprocess\.\w+\s*\([^)]*shell\s*=\s*True',
                r'eval\s*\(',
                r'exec\s*\(',
            ],
            "severity": "critical",
            "message": "Potential command injection",
            "fix": "Avoid shell=True, sanitize inputs, use subprocess with list args"
        },
        "path_traversal": {
            "patterns": [
                r'open\s*\([^)]*\+',
                r'Path\s*\([^)]*\+',
            ],
            "severity": "high",
            "message": "Potential path traversal",
            "fix": "Validate and sanitize file paths"
        }
    }

    async def scan(self, content: str, file_path: str) -> list[Finding]:
        findings = []

        # Determine applicable rules based on file type
        applicable_rules = self._get_applicable_rules(file_path)

        for rule_name, rule in applicable_rules.items():
            for pattern in rule["patterns"]:
                matches = re.finditer(pattern, content)
                for match in matches:
                    line_num = content[:match.start()].count('\n') + 1
                    findings.append(Finding(
                        type="sast",
                        severity=rule["severity"],
                        title=rule["message"],
                        location=f"{file_path}:{line_num}",
                        code_snippet=self._get_snippet(content, match),
                        recommendation=rule["fix"]
                    ))

        return findings
```

---

## Examples

### Example 1: Block Secret Commit

```python
# Event: tool:pre (Write file)
# Data: {"file_path": "src/config.py", "content": "API_KEY = 'sk_live_...'"}

# Hook returns:
{
    "action": "deny",
    "reason": """
Cannot write file: Critical security issue detected.

**OpenAI API Key detected** at line 1:
```python
API_KEY = 'sk_live_...'
```

Remove the secret and use an environment variable instead:
```python
import os
API_KEY = os.environ.get('OPENAI_API_KEY')
```
"""
}

# Result: File write is blocked, agent sees denial reason
```

### Example 2: Inject Fix Suggestions

```python
# Event: tool:post (Write file completed)
# Data: {"file_path": "src/db.py", "content": "...SQL with string formatting..."}

# Hook returns:
{
    "action": "inject_context",
    "context_injection": """
Security scan found issues in src/db.py:

**SQL Injection (HIGH)** at line 42:
```python
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
```
**Fix**: Use parameterized queries:
```python
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
```

Please fix this issue.
""",
    "user_message": "Security: 1 high severity issue in src/db.py",
    "user_message_level": "warning"
}

# Result: Agent sees feedback, fixes the code in next turn
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `mode` | string | "warn" | block, warn, or report only |
| `severity_threshold` | string | "medium" | Minimum severity to act on |
| `scanners.secrets` | bool | true | Enable secret detection |
| `scanners.sast` | bool | true | Enable SAST scanning |
| `scanners.dependencies` | bool | false | Enable dependency scanning |
| `secrets.patterns` | list | (defaults) | Additional regex patterns |
| `secrets.allow_patterns` | list | [] | Patterns to ignore (e.g., test data) |

---

## Security Considerations

- Scans may reveal sensitive patterns (handled carefully)
- Allow patterns should be restricted to test/example data
- SAST rules have false positive potential - tune per project

---

## Dependencies

### Required
- `re` - Regex matching (stdlib)

### Optional
- `semgrep` - Advanced SAST rules
- `bandit` - Python-specific security scanning
- `safety` - Dependency vulnerability scanning

---

## Open Questions

1. **SAST depth**: Include tree-sitter parsing for more accurate analysis?
2. **Custom rules**: Allow project-specific security rules?
3. **Remediation automation**: Auto-fix simple issues?
4. **Integration**: Connect to enterprise security tools?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
