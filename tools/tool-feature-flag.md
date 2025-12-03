# tool-feature-flag

> **Priority**: P2 (Enhancement)
> **Status**: Draft
> **Module**: `amplifier-module-tool-feature-flag`

## Overview

Manage feature flags across the codebase. Check flag status, toggle flags, analyze flag usage and debt, and manage flag lifecycle from creation to cleanup.

### Value Proposition

| Without | With |
|---------|------|
| Check multiple dashboards for flag status | "Is dark-mode enabled for 10% of users?" → instant answer |
| Old flags accumulate forever | Automated flag debt detection and cleanup reminders |
| Flag changes require dashboard access | Toggle flags from your workflow |
| Unknown flag coverage | See exactly which code paths are flagged |

### Use Cases

1. **Flag status**: Check if a flag is enabled and for whom
2. **Flag management**: Create, toggle, archive flags
3. **Debt detection**: Find stale flags ready for cleanup
4. **Code analysis**: Find all code paths controlled by a flag
5. **Impact assessment**: What happens if we toggle this flag?

---

## Contract

### Tool Definition

```python
TOOL_DEFINITION = {
    "name": "feature_flag",
    "description": """
    Manage feature flags and analyze flag usage.

    Operations:
    - status: Check flag status and targeting
    - toggle: Enable/disable flag or change targeting
    - list: List all flags with status
    - analyze: Find flag usage in code
    - debt: Identify stale flags for cleanup
    - create: Create new flag
    - archive: Archive/delete flag

    Supports: LaunchDarkly, Split, Unleash, ConfigCat, custom systems.
    """,
    "parameters": {
        "type": "object",
        "properties": {
            "operation": {
                "type": "string",
                "enum": ["status", "toggle", "list", "analyze", "debt", "create", "archive"],
                "description": "Operation to perform"
            },
            "flag_key": {
                "type": "string",
                "description": "Flag key/name"
            },
            "environment": {
                "type": "string",
                "default": "production",
                "description": "Environment (production, staging, development)"
            },
            "targeting": {
                "type": "object",
                "description": "Targeting rules for toggle operation"
            },
            "include_code_references": {
                "type": "boolean",
                "default": false,
                "description": "Include code locations using this flag"
            }
        },
        "required": ["operation"]
    }
}
```

### Output Schema

```python
@dataclass
class FeatureFlagOutput:
    operation: str

    # For status
    flag_status: FlagStatus | None = None

    # For list
    flags: list[FlagSummary] | None = None

    # For analyze
    code_references: list[CodeReference] | None = None

    # For debt
    stale_flags: list[StaleFlag] | None = None

@dataclass
class FlagStatus:
    key: str
    name: str
    description: str | None
    kind: str                           # boolean | multivariate | string | number

    # Status per environment
    environments: dict[str, EnvironmentStatus]

    # Metadata
    created_at: datetime
    updated_at: datetime
    created_by: str
    tags: list[str]

    # Usage stats
    evaluation_count_24h: int | None
    unique_users_24h: int | None

    # Code references (if requested)
    code_references: list[CodeReference] | None

@dataclass
class EnvironmentStatus:
    enabled: bool
    targeting: TargetingRules | None
    percentage_rollout: float | None    # 0-100 for percentage rollouts
    default_value: Any

@dataclass
class TargetingRules:
    rules: list[TargetingRule]
    fallthrough: Any                    # Default when no rules match

@dataclass
class TargetingRule:
    description: str
    clauses: list[TargetClause]         # AND conditions
    variation: Any                      # Value when rule matches

@dataclass
class TargetClause:
    attribute: str                      # user attribute to check
    operator: str                       # equals | contains | startsWith | in | etc.
    values: list[Any]

@dataclass
class CodeReference:
    file_path: str
    line_number: int
    code_snippet: str
    context: str                        # function/class containing the reference
    type: str                           # check | variation | default

@dataclass
class FlagSummary:
    key: str
    name: str
    enabled_environments: list[str]
    age_days: int
    evaluation_count_7d: int
    stale_score: float                  # 0-1, higher = more stale

@dataclass
class StaleFlag:
    key: str
    name: str
    age_days: int
    last_modified: datetime
    reasons: list[str]                  # Why it's considered stale
    code_references: list[CodeReference]
    recommendation: str                 # "remove", "review", "keep"
```

---

## Architecture

### Platform Adapters

```python
class LaunchDarklyAdapter:
    """Adapter for LaunchDarkly."""

    async def get_flag(self, key: str) -> FlagStatus:
        flag = await self.client.get_flag(self.project_key, key)

        environments = {}
        for env_key, env_config in flag.environments.items():
            environments[env_key] = EnvironmentStatus(
                enabled=env_config.on,
                targeting=self._parse_targeting(env_config.rules),
                percentage_rollout=self._extract_percentage(env_config.fallthrough),
                default_value=env_config.off_variation
            )

        return FlagStatus(
            key=flag.key,
            name=flag.name,
            description=flag.description,
            kind=flag.kind,
            environments=environments,
            created_at=flag.creation_date,
            updated_at=flag.last_modified,
            tags=flag.tags
        )

    async def toggle(self, key: str, environment: str, enabled: bool) -> None:
        patch = [{"op": "replace", "path": f"/environments/{environment}/on", "value": enabled}]
        await self.client.patch_flag(self.project_key, key, patch)


class CodeAnalyzer:
    """Find flag usage in code."""

    async def find_references(self, flag_key: str) -> list[CodeReference]:
        references = []

        # Search patterns for common flag SDKs
        patterns = [
            f'variation\\(["\\']{flag_key}["\\'',      # variation("flag")
            f'is_enabled\\(["\\']{flag_key}["\\'',     # is_enabled("flag")
            f'get_flag\\(["\\']{flag_key}["\\'',       # get_flag("flag")
            f'flag\\(["\\']{flag_key}["\\'',           # flag("flag")
            f'["\\']{flag_key}["\\'\\s*\\]',           # flags["flag"]
        ]

        for pattern in patterns:
            matches = await self._grep(pattern)
            for match in matches:
                references.append(CodeReference(
                    file_path=match.file_path,
                    line_number=match.line_number,
                    code_snippet=match.line,
                    context=await self._get_context(match),
                    type=self._classify_reference(match.line)
                ))

        return references


class FlagDebtAnalyzer:
    """Identify stale flags for cleanup."""

    async def analyze(self) -> list[StaleFlag]:
        flags = await self.adapter.list_flags()
        stale = []

        for flag in flags:
            reasons = []
            stale_score = 0.0

            # Age-based staleness
            if flag.age_days > 180:
                reasons.append(f"Flag is {flag.age_days} days old")
                stale_score += 0.3

            # No recent evaluations
            if flag.evaluation_count_7d == 0:
                reasons.append("No evaluations in last 7 days")
                stale_score += 0.3

            # Fully rolled out (100% on)
            if self._is_fully_rolled_out(flag):
                reasons.append("Flag is 100% enabled everywhere")
                stale_score += 0.2

            # Fully rolled back (100% off)
            if self._is_fully_rolled_back(flag):
                reasons.append("Flag is disabled everywhere")
                stale_score += 0.2

            # No code references
            refs = await self.code_analyzer.find_references(flag.key)
            if not refs:
                reasons.append("No code references found")
                stale_score += 0.2

            if stale_score > 0.4:
                stale.append(StaleFlag(
                    key=flag.key,
                    name=flag.name,
                    age_days=flag.age_days,
                    last_modified=flag.updated_at,
                    reasons=reasons,
                    code_references=refs,
                    recommendation=self._recommend_action(stale_score, refs)
                ))

        return sorted(stale, key=lambda f: -f.age_days)
```

---

## Examples

### Example 1: Check Flag Status

```python
# Input
{
    "operation": "status",
    "flag_key": "new-checkout-flow",
    "include_code_references": true
}

# Output
{
    "flag_status": {
        "key": "new-checkout-flow",
        "name": "New Checkout Flow",
        "description": "Redesigned checkout experience with fewer steps",
        "kind": "boolean",
        "environments": {
            "production": {
                "enabled": true,
                "percentage_rollout": 25.0,
                "targeting": {
                    "rules": [
                        {
                            "description": "Beta users",
                            "clauses": [{"attribute": "beta", "operator": "equals", "values": [true]}],
                            "variation": true
                        }
                    ],
                    "fallthrough": "percentage"
                }
            },
            "staging": {
                "enabled": true,
                "percentage_rollout": 100.0
            }
        },
        "evaluation_count_24h": 45230,
        "unique_users_24h": 12500,
        "code_references": [
            {
                "file_path": "src/checkout/flow.py",
                "line_number": 42,
                "code_snippet": "if flag_client.variation('new-checkout-flow', user):",
                "context": "CheckoutController.process"
            }
        ]
    }
}
```

### Example 2: Find Stale Flags

```python
# Input
{
    "operation": "debt"
}

# Output
{
    "stale_flags": [
        {
            "key": "holiday-banner-2023",
            "name": "Holiday Banner 2023",
            "age_days": 412,
            "last_modified": "2023-01-15T10:00:00Z",
            "reasons": [
                "Flag is 412 days old",
                "No evaluations in last 7 days",
                "Flag is disabled everywhere"
            ],
            "code_references": [
                {
                    "file_path": "src/components/Banner.tsx",
                    "line_number": 15,
                    "code_snippet": "const showHoliday = useFlag('holiday-banner-2023')"
                }
            ],
            "recommendation": "remove"
        },
        {
            "key": "payment-v2",
            "name": "Payment V2",
            "age_days": 245,
            "reasons": [
                "Flag is 245 days old",
                "Flag is 100% enabled everywhere"
            ],
            "recommendation": "remove (clean up code references first)"
        }
    ]
}
```

---

## Configuration

```yaml
tool-feature-flag:
  # Platform configuration
  platform: launchdarkly              # launchdarkly | split | unleash | configcat

  launchdarkly:
    api_key: "${LD_API_KEY}"
    project_key: "default"

  # Code analysis
  code_analysis:
    include_patterns:
      - "**/*.py"
      - "**/*.ts"
      - "**/*.tsx"
      - "**/*.js"
    exclude_patterns:
      - "**/node_modules/**"
      - "**/*.test.*"

  # Staleness thresholds
  staleness:
    age_warning_days: 90
    age_error_days: 180
    no_evaluation_days: 7
```

---

## Security Considerations

- Requires API access to feature flag platform
- Can toggle flags (affects production behavior)
- Should require approval for production toggles

---

## Dependencies

### Required
- Platform-specific SDK (launchdarkly-api, etc.)

### Optional
- None

---

## Open Questions

1. **Toggle approval**: Require approval for production toggles?
2. **Audit logging**: Log all flag operations?
3. **Automated cleanup**: Auto-archive stale flags?
4. **Cross-environment sync**: Help sync targeting across environments?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
