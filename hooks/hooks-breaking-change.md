# hooks-breaking-change

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-hooks-breaking-change`

## Overview

Detects potential breaking changes in API contracts, database schemas, and public interfaces. Warns developers before they accidentally break downstream consumers.

### Value Proposition

| Without | With |
|---------|------|
| Breaking changes discovered by consumers | Detected before merge |
| "We didn't know that was public API" | Clear visibility into contracts |
| Versioning afterthought | Version bump reminders |

---

## Contract

### Hook Configuration

```yaml
hooks:
  - module: hooks-breaking-change
    config:
      # What to analyze
      analyze:
        api_endpoints: true             # REST/GraphQL endpoints
        function_signatures: true       # Public function changes
        database_schemas: true          # Schema migrations
        config_files: true              # Configuration formats
        proto_files: true               # Protocol buffers

      # Detection rules
      rules:
        api:
          # Breaking: removing endpoint, changing required params
          - type: endpoint_removed
            severity: error
          - type: required_param_added
            severity: error
          - type: response_field_removed
            severity: error
          # Non-breaking: adding optional param, new endpoint
          - type: optional_param_added
            severity: info
          - type: endpoint_added
            severity: info

        functions:
          # Public functions (exported)
          - type: signature_changed
            scope: public
            severity: error
          - type: function_removed
            scope: public
            severity: error
          # Private functions (not exported)
          - type: signature_changed
            scope: private
            severity: info

        database:
          - type: column_removed
            severity: error
          - type: column_type_changed
            severity: error
          - type: not_null_added
            severity: warning

      # Action on detection
      on_breaking_change:
        action: inject_context          # inject_context | deny | warn
        require_acknowledgment: true
```

### HookResult

```python
# Detected breaking change
HookResult(
    action="inject_context",
    context_injection="""
⚠️ **Breaking Change Detected**

File: src/api/users.py

**Changes:**
1. **BREAKING**: Removed endpoint `DELETE /api/users/{id}`
   - This endpoint may have consumers depending on it
   - Consider deprecation period before removal

2. **BREAKING**: Added required parameter `email` to `POST /api/users`
   - Existing clients will fail without this parameter
   - Consider making optional with default, or versioning API

**Recommendations:**
- Add `@deprecated` decorator with removal date
- Create v2 endpoint with new signature
- Update API changelog

Do you want to proceed with these breaking changes?
""",
    user_message="Breaking changes detected in users.py API",
    user_message_level="warning"
)
```

---

## Architecture

```python
class BreakingChangeHook:
    """Detect breaking changes in code."""

    def __init__(self, config: BreakingChangeConfig):
        self.config = config
        self.analyzers = self._initialize_analyzers()

    async def __call__(self, event: str, data: dict) -> HookResult:
        if event != "tool:post" or not self._is_code_write(data):
            return HookResult(action="continue")

        file_path = data.get("file_path", "")
        new_content = data.get("content", "")

        # Get previous content from git
        old_content = await self._get_previous_content(file_path)
        if old_content is None:
            return HookResult(action="continue")  # New file

        # Analyze for breaking changes
        changes = await self._analyze_changes(file_path, old_content, new_content)

        if not changes:
            return HookResult(action="continue")

        # Filter by severity
        breaking = [c for c in changes if c.severity in ("error", "warning")]

        if not breaking:
            return HookResult(action="continue")

        return self._generate_feedback(file_path, breaking)

    async def _analyze_changes(
        self,
        file_path: str,
        old_content: str,
        new_content: str
    ) -> list[ChangeDetection]:
        """Analyze changes using appropriate analyzer."""
        changes = []

        for analyzer in self.analyzers:
            if analyzer.can_analyze(file_path):
                analyzer_changes = await analyzer.analyze(
                    file_path, old_content, new_content
                )
                changes.extend(analyzer_changes)

        return changes


class APIAnalyzer:
    """Analyze REST API changes."""

    async def analyze(
        self,
        file_path: str,
        old_content: str,
        new_content: str
    ) -> list[ChangeDetection]:
        changes = []

        # Extract endpoints from both versions
        old_endpoints = self._extract_endpoints(old_content)
        new_endpoints = self._extract_endpoints(new_content)

        # Check for removed endpoints
        for endpoint in old_endpoints:
            if endpoint not in new_endpoints:
                changes.append(ChangeDetection(
                    type="endpoint_removed",
                    severity="error",
                    location=file_path,
                    description=f"Endpoint removed: {endpoint.method} {endpoint.path}",
                    old_value=str(endpoint),
                    new_value=None
                ))

        # Check for signature changes
        for old_ep in old_endpoints:
            new_ep = self._find_endpoint(new_endpoints, old_ep.method, old_ep.path)
            if new_ep:
                param_changes = self._compare_parameters(old_ep, new_ep)
                changes.extend(param_changes)

        return changes

    def _extract_endpoints(self, content: str) -> list[Endpoint]:
        """Extract API endpoints from code."""
        endpoints = []

        # Flask/FastAPI style decorators
        patterns = [
            r'@app\.(?:get|post|put|delete|patch)\(["\']([^"\']+)["\']',
            r'@router\.(?:get|post|put|delete|patch)\(["\']([^"\']+)["\']',
        ]

        for pattern in patterns:
            for match in re.finditer(pattern, content):
                # Extract method and path
                # Parse function signature for parameters
                endpoints.append(self._parse_endpoint(match, content))

        return endpoints


class FunctionSignatureAnalyzer:
    """Analyze public function signature changes."""

    async def analyze(
        self,
        file_path: str,
        old_content: str,
        new_content: str
    ) -> list[ChangeDetection]:
        changes = []

        # Parse both versions
        old_functions = self._extract_functions(old_content)
        new_functions = self._extract_functions(new_content)

        # Check public functions only
        for name, old_func in old_functions.items():
            if not self._is_public(name, old_func):
                continue

            if name not in new_functions:
                changes.append(ChangeDetection(
                    type="function_removed",
                    severity="error",
                    location=f"{file_path}:{old_func.line}",
                    description=f"Public function removed: {name}",
                ))
            else:
                new_func = new_functions[name]
                sig_changes = self._compare_signatures(old_func, new_func)
                if sig_changes:
                    changes.append(ChangeDetection(
                        type="signature_changed",
                        severity="error",
                        location=f"{file_path}:{new_func.line}",
                        description=f"Signature changed for {name}: {sig_changes}",
                    ))

        return changes

    def _is_public(self, name: str, func: FunctionInfo) -> bool:
        """Determine if function is public."""
        # Python: doesn't start with _
        if name.startswith("_"):
            return False
        # Check for @public decorator or __all__ export
        return True
```

---

## Change Detection Types

### API Changes

| Type | Severity | Description |
|------|----------|-------------|
| `endpoint_removed` | error | Endpoint deleted |
| `required_param_added` | error | New required parameter |
| `response_field_removed` | error | Field removed from response |
| `return_type_changed` | error | Return type changed |
| `optional_param_added` | info | New optional parameter |
| `endpoint_added` | info | New endpoint |

### Function Signature Changes

| Type | Severity | Description |
|------|----------|-------------|
| `function_removed` | error | Public function removed |
| `param_removed` | error | Parameter removed |
| `param_type_changed` | error | Parameter type changed |
| `return_type_changed` | error | Return type changed |
| `param_added_required` | error | Required param added |
| `param_added_optional` | info | Optional param added |

### Database Schema Changes

| Type | Severity | Description |
|------|----------|-------------|
| `column_removed` | error | Column dropped |
| `column_type_changed` | error | Column type changed |
| `not_null_added` | warning | NOT NULL constraint added |
| `table_removed` | error | Table dropped |
| `column_added_nullable` | info | Nullable column added |

---

## Examples

### Example 1: API Endpoint Removed

```python
# Old code:
@app.delete("/api/users/{id}")
def delete_user(id: int):
    ...

# New code: (endpoint removed)

# Hook detects and warns:
"""
⚠️ **Breaking Change Detected**

**BREAKING**: Endpoint removed: DELETE /api/users/{id}
- Consumers using this endpoint will receive 404

**Recommendation:**
- Add deprecation notice before removal
- Provide migration path in documentation
"""
```

### Example 2: Function Signature Change

```python
# Old:
def process_payment(amount: float, currency: str) -> bool:
    ...

# New:
def process_payment(amount: float, currency: str, idempotency_key: str) -> PaymentResult:
    ...

# Hook detects:
"""
⚠️ **Breaking Change Detected**

1. **BREAKING**: New required parameter `idempotency_key`
   - Existing callers will fail with TypeError

2. **BREAKING**: Return type changed from `bool` to `PaymentResult`
   - Callers expecting boolean will break

**Recommendations:**
- Make `idempotency_key` optional with default
- Keep backward-compatible return for transition period
"""
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `analyze.*` | bool | true | What to analyze |
| `rules.<category>` | list | (defaults) | Detection rules |
| `on_breaking_change.action` | string | "inject_context" | Action on detection |
| `on_breaking_change.require_acknowledgment` | bool | true | Require user ack |

---

## Security Considerations

- Reads file content and git history
- No external network access
- Analysis is local and fast

---

## Dependencies

### Required
- `tree-sitter` - Code parsing
- `gitpython` - Git history access

### Optional
- Language-specific parsers

---

## Open Questions

1. **Semantic versioning**: Auto-suggest version bump?
2. **Changelog**: Auto-generate changelog entries?
3. **Consumer notification**: Integrate with deprecation notices?
4. **Custom contracts**: Support custom API contract formats?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
