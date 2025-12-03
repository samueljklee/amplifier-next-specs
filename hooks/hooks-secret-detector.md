# hooks-secret-detector

> **Priority**: P0 (Foundation)
> **Status**: Draft
> **Module**: `amplifier-module-hooks-secret-detector`

## Overview

Specialized hook focused exclusively on secret detection. Lightweight, fast, and critical for preventing credential leaks. Runs on every file write with minimal overhead.

### Value Proposition

| Without | With |
|---------|------|
| Secrets accidentally committed | Hard block before write |
| Security scanning is slow | Optimized for speed, runs on every write |
| Generic scanners miss secrets | Purpose-built for credential detection |

### Key Difference from hooks-security-scan

- **hooks-secret-detector**: Fast, focused, always-on for secret detection
- **hooks-security-scan**: Comprehensive but heavier SAST + secrets + dependencies

Use both together: secret-detector as first line of defense, security-scan for deeper analysis.

---

## Contract

### Hook Configuration

```yaml
hooks:
  - module: hooks-secret-detector
    config:
      # Always block secrets (no severity threshold)
      action: deny                      # deny | warn

      # Built-in pattern categories
      detect:
        aws: true
        gcp: true
        azure: true
        github: true
        openai: true
        stripe: true
        slack: true
        database: true
        private_keys: true
        generic_secrets: true

      # Custom patterns
      custom_patterns:
        - pattern: "COMPANY_API_[A-Z0-9]{32}"
          name: "Company Internal API Key"

      # Exclusions
      exclude:
        files:
          - "**/*.test.*"
          - "**/fixtures/**"
          - "**/mocks/**"
        patterns:
          - "test_api_key"
          - "example_"
          - "dummy_"
          - "fake_"

      # Allow specific files to contain secrets (use with extreme caution)
      allowed_files:
        - ".env.example"                # Only example values
```

### HookResult

```python
# Always deny on secret detection
HookResult(
    action="deny",
    reason="""
🚨 SECRET DETECTED - File write blocked

**AWS Secret Access Key** found at line 15:
```
aws_secret_access_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

**How to fix:**
1. Remove the secret from your code
2. Use environment variables:
   ```python
   import os
   aws_secret = os.environ.get('AWS_SECRET_ACCESS_KEY')
   ```
3. If this was intentional test data, prefix with 'test_' or 'example_'

This secret pattern is commonly exploited within minutes of exposure.
"""
)
```

---

## Architecture

```python
class SecretDetectorHook:
    """Lightweight, fast secret detection hook."""

    # Pre-compiled patterns for speed
    PATTERNS = {
        "aws_access_key": re.compile(r'AKIA[0-9A-Z]{16}'),
        "aws_secret_key": re.compile(r'(?i)aws_secret_access_key\s*[=:]\s*["\']?([A-Za-z0-9/+=]{40})["\']?'),
        "github_token": re.compile(r'ghp_[a-zA-Z0-9]{36}'),
        "github_oauth": re.compile(r'gho_[a-zA-Z0-9]{36}'),
        "openai_key": re.compile(r'sk-[a-zA-Z0-9]{48}'),
        "stripe_live": re.compile(r'sk_live_[a-zA-Z0-9]{24,}'),
        "stripe_test": re.compile(r'sk_test_[a-zA-Z0-9]{24,}'),  # Still warn
        "slack_token": re.compile(r'xox[baprs]-[0-9a-zA-Z]{10,}'),
        "private_key": re.compile(r'-----BEGIN\s+(?:RSA\s+|EC\s+|DSA\s+)?PRIVATE\s+KEY-----'),
        "generic_api_key": re.compile(r'(?i)api[_-]?key\s*[=:]\s*["\']([a-zA-Z0-9]{20,})["\']'),
        "generic_secret": re.compile(r'(?i)(?:secret|password|passwd|pwd)\s*[=:]\s*["\']([^"\']{8,})["\']'),
    }

    def __init__(self, config: SecretDetectorConfig):
        self.config = config
        self.patterns = self._build_patterns()
        self.exclusion_patterns = [re.compile(p) for p in config.exclude.patterns]

    async def __call__(self, event: str, data: dict) -> HookResult:
        # Only process file writes
        if event != "tool:pre" or not self._is_file_write(data):
            return HookResult(action="continue")

        file_path = data.get("file_path", "")
        content = data.get("content", "")

        # Check if file is excluded
        if self._is_excluded_file(file_path):
            return HookResult(action="continue")

        # Scan for secrets
        secrets = self._detect_secrets(content, file_path)

        if not secrets:
            return HookResult(action="continue")

        # Block the write
        return HookResult(
            action="deny",
            reason=self._format_denial(secrets, file_path)
        )

    def _detect_secrets(self, content: str, file_path: str) -> list[DetectedSecret]:
        secrets = []
        lines = content.split('\n')

        for line_num, line in enumerate(lines, 1):
            for pattern_name, pattern in self.patterns.items():
                match = pattern.search(line)
                if match:
                    # Check if excluded by pattern
                    if self._is_excluded_value(match.group()):
                        continue

                    secrets.append(DetectedSecret(
                        type=pattern_name,
                        line_number=line_num,
                        line_content=line.strip(),
                        match=match.group()
                    ))

        return secrets

    def _is_excluded_value(self, value: str) -> bool:
        """Check if the matched value is in exclusion list."""
        for pattern in self.exclusion_patterns:
            if pattern.search(value):
                return True
        return False

    def _format_denial(self, secrets: list[DetectedSecret], file_path: str) -> str:
        """Format denial message with actionable guidance."""
        lines = ["🚨 SECRET DETECTED - File write blocked\n"]

        for secret in secrets:
            lines.append(f"**{self._friendly_name(secret.type)}** at line {secret.line_number}:")
            # Mask most of the secret
            masked = self._mask_secret(secret.match)
            lines.append(f"```\n{masked}\n```\n")

        lines.append("**How to fix:**")
        lines.append("1. Remove the secret from your code")
        lines.append("2. Use environment variables instead")
        lines.append("3. If this is test data, prefix with 'test_' or 'example_'")

        return "\n".join(lines)

    def _mask_secret(self, secret: str) -> str:
        """Mask secret for display, showing only first/last few chars."""
        if len(secret) <= 8:
            return "*" * len(secret)
        return secret[:4] + "*" * (len(secret) - 8) + secret[-4:]
```

---

## Secret Pattern Coverage

| Category | Patterns | Examples |
|----------|----------|----------|
| **AWS** | Access Key ID, Secret Key, Session Token | `AKIA...`, `aws_secret_access_key=...` |
| **GCP** | Service Account Key, API Key | `AIza...`, JSON key files |
| **Azure** | Storage Key, Connection String | `DefaultEndpointsProtocol=...` |
| **GitHub** | PAT, OAuth, App Token | `ghp_...`, `gho_...`, `ghs_...` |
| **OpenAI** | API Key | `sk-...` |
| **Stripe** | Live/Test Keys | `sk_live_...`, `rk_live_...` |
| **Slack** | Bot, User, App Tokens | `xoxb-...`, `xoxp-...` |
| **Database** | Connection Strings | `postgres://user:pass@...` |
| **Private Keys** | RSA, EC, DSA | `-----BEGIN PRIVATE KEY-----` |
| **Generic** | API keys, passwords, secrets | `api_key=...`, `password=...` |

---

## Examples

### Example 1: AWS Key Detection

```python
# Attempting to write:
# src/config.py containing:
# AWS_ACCESS_KEY = "AKIAIOSFODNN7EXAMPLE"

# Hook returns:
{
    "action": "deny",
    "reason": """
🚨 SECRET DETECTED - File write blocked

**AWS Access Key ID** at line 1:
```
AKIA************MPLE
```

**How to fix:**
1. Remove the secret from your code
2. Use environment variables instead:
   ```python
   import os
   AWS_ACCESS_KEY = os.environ.get('AWS_ACCESS_KEY_ID')
   ```
3. If this is test data, use AWS's documented example keys

AWS credentials are actively scanned on GitHub and can be exploited within minutes.
"""
}
```

### Example 2: Test Data Exclusion

```python
# Attempting to write:
# tests/fixtures/config.py containing:
# test_api_key = "test_sk_1234567890"

# Hook returns:
{
    "action": "continue"
}

# Allowed because:
# 1. File is in tests/fixtures/ (excluded path)
# 2. Value starts with "test_" (excluded pattern)
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `action` | string | "deny" | deny or warn |
| `detect.*` | bool | true | Enable specific pattern categories |
| `custom_patterns` | list | [] | Additional patterns to detect |
| `exclude.files` | list | [] | Glob patterns for files to skip |
| `exclude.patterns` | list | [] | Value patterns to ignore |
| `allowed_files` | list | [] | Files that may contain secrets |

---

## Security Considerations

- Patterns must be kept updated as providers change formats
- False positives should be handled via exclusions, not disabled detection
- Masked output prevents exposure in logs
- `allowed_files` should be used very sparingly

---

## Dependencies

### Required
- `re` - Regex matching (stdlib)

### Optional
- None (intentionally minimal)

---

## Open Questions

1. **Pattern updates**: How to distribute pattern updates?
2. **Entropy analysis**: Add entropy-based detection for unknown patterns?
3. **Git integration**: Also scan git history for past leaks?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
