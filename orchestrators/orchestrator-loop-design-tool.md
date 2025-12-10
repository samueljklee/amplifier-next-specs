# orchestrator-loop-design-tool

> **Priority**: P1 (High Value - Alex's Domain)
> **Status**: Draft
> **Module**: `amplifier-module-loop-design-tool`

## Overview

An orchestrator optimized for design tool workflows where the interface is primarily visual/structured rather than conversational. The LLM operates on structured design data (components, layouts, styles) and returns structured modifications, enabling rich visual interfaces with AI-powered manipulation.

### Value Proposition

| Without | With |
|---------|------|
| Chat-based design feedback | Direct manipulation with AI assistance |
| Describe changes in prose | Structured intent → structured output |
| Manual implementation of suggestions | AI generates design tokens, not descriptions |
| Generic coding assistant | Design-aware operations and constraints |

### Use Cases

1. **Component generation**: "Create a button component with these variants"
2. **Layout assistance**: "Arrange these elements in a responsive grid"
3. **Style generation**: "Generate a color palette for this brand"
4. **Design system expansion**: "Create icon variants matching this style"
5. **Accessibility auditing**: "Check this component for WCAG compliance"

### Modality Differentiator

This breaks the "chatbot" mental model by:
- **Structured I/O**: Design tokens in, design tokens out
- **Visual-first**: Interface shows design, not chat
- **Constrained domain**: Design system aware
- **Direct manipulation**: Click/drag triggers AI, not typing

---

## Contract

### Orchestrator Interface

```python
from dataclasses import dataclass
from typing import Any
from enum import Enum


class DesignOperation(Enum):
    GENERATE = "generate"       # Create new design element
    MODIFY = "modify"           # Change existing element
    ANALYZE = "analyze"         # Inspect/audit element
    SUGGEST = "suggest"         # Propose alternatives
    VALIDATE = "validate"       # Check against constraints


@dataclass
class DesignElement:
    """A design element (component, style, layout, etc.)."""
    id: str
    type: str  # "component" | "style" | "layout" | "token" | "icon"
    data: dict[str, Any]
    metadata: dict[str, Any] | None = None


@dataclass
class DesignIntent:
    """User's design intent - structured, not prose."""
    operation: DesignOperation
    target: DesignElement | None  # Element to modify (if applicable)
    parameters: dict[str, Any]  # Operation-specific parameters
    constraints: list[str] | None = None  # Design system constraints
    context: dict[str, Any] | None = None  # Surrounding context


@dataclass
class DesignResult:
    """Result of design operation - structured output."""
    operation: DesignOperation
    success: bool
    elements: list[DesignElement]  # New or modified elements
    changes: list[dict] | None = None  # Change descriptions
    suggestions: list[dict] | None = None  # Alternative approaches
    issues: list[dict] | None = None  # Problems found


class DesignToolOrchestrator:
    """
    Orchestrator for design tool workflows.

    Flow:
    1. Receive structured design intent
    2. Load design system context
    3. Execute design operation
    4. Return structured design output
    5. Optionally iterate on feedback
    """

    async def execute(
        self,
        prompt: str,  # JSON-encoded DesignIntent
        context: Any,
        providers: dict[str, Any],
        tools: dict[str, Any],
        hooks: Any,
        coordinator: Any | None = None,
    ) -> str:
        """Execute design operation, return JSON DesignResult."""

    async def process_intent(
        self,
        intent: DesignIntent,
        design_system: dict[str, Any],
    ) -> DesignResult:
        """Process a single design intent."""

    async def validate_output(
        self,
        result: DesignResult,
        design_system: dict[str, Any],
    ) -> list[dict]:
        """Validate output against design system constraints."""
```

### Configuration Schema

```toml
[[orchestrators]]
module = "loop-design-tool"
config = {
  # Design system
  design_system = "./design-system.json"  # Design tokens, components, etc.
  component_library = "./components/"      # Existing components

  # Constraints
  enforce_design_system = true            # Must use design tokens
  allow_custom_tokens = false             # Can create new tokens
  accessibility_level = "AA"               # WCAG compliance level

  # Output format
  output_format = "figma"                  # "figma" | "css" | "react" | "tokens"
  include_variants = true                  # Generate component variants
  include_responsive = true                # Generate responsive versions

  # Generation
  temperature = 0.3                        # Lower for consistency
  max_suggestions = 3                      # Alternative designs to generate

  # Iteration
  allow_refinement = true                  # Support iterative refinement
  max_refinement_rounds = 5

  # Validation
  validate_accessibility = true
  validate_brand = true
  validate_consistency = true
}
```

---

## Architecture

### Design Operation Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    DESIGN INTENT                             │
│  Operation: generate/modify/analyze/suggest/validate        │
│  Target: component/style/layout                             │
│  Parameters: structured operation data                      │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    DESIGN CONTEXT LOADER                     │
│  Design System │ Component Library │ Brand Guidelines       │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    INTENT PROCESSOR                          │
│  Parse intent │ Map to design operations │ Load references  │
└────────────────────────┬────────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┬──────────────┐
          ▼              ▼              ▼              ▼
     ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
     │Generate │    │ Modify  │    │ Analyze │    │Validate │
     │  Path   │    │  Path   │    │  Path   │    │  Path   │
     └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘
          │              │              │              │
          └──────────────┴──────┬──────┴──────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────┐
│                    OUTPUT VALIDATOR                          │
│  Design System Compliance │ Accessibility │ Brand           │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    DESIGN RESULT                             │
│  Structured output │ Design tokens │ Code │ Suggestions     │
└─────────────────────────────────────────────────────────────┘
```

### Design System Context

```json
{
  "tokens": {
    "colors": {
      "primary": { "50": "#eff6ff", "500": "#3b82f6", "900": "#1e3a8a" },
      "neutral": { "50": "#fafafa", "500": "#737373", "900": "#171717" }
    },
    "spacing": {
      "xs": "4px", "sm": "8px", "md": "16px", "lg": "24px", "xl": "32px"
    },
    "typography": {
      "fontFamily": { "sans": "Inter", "mono": "JetBrains Mono" },
      "fontSize": { "sm": "14px", "base": "16px", "lg": "18px" }
    },
    "borderRadius": {
      "sm": "4px", "md": "8px", "lg": "12px", "full": "9999px"
    }
  },
  "components": {
    "Button": {
      "variants": ["primary", "secondary", "ghost"],
      "sizes": ["sm", "md", "lg"],
      "states": ["default", "hover", "active", "disabled"]
    }
  },
  "constraints": {
    "minContrastRatio": 4.5,
    "minTouchTarget": "44px",
    "maxLineLength": "75ch"
  }
}
```

---

## Operations

### GENERATE - Create New Elements

```python
# Intent
{
    "operation": "generate",
    "parameters": {
        "type": "component",
        "name": "Card",
        "description": "A card component for displaying content",
        "variants": ["default", "elevated", "outlined"],
        "props": ["title", "description", "image", "actions"]
    },
    "constraints": ["use design tokens", "WCAG AA"]
}

# Result
{
    "operation": "generate",
    "success": true,
    "elements": [
        {
            "id": "card-default",
            "type": "component",
            "data": {
                "name": "Card",
                "variant": "default",
                "styles": {
                    "backgroundColor": "var(--color-neutral-50)",
                    "borderRadius": "var(--radius-md)",
                    "padding": "var(--spacing-md)",
                    "boxShadow": "var(--shadow-sm)"
                },
                "slots": [
                    { "name": "image", "type": "image" },
                    { "name": "title", "type": "text", "style": "heading-sm" },
                    { "name": "description", "type": "text", "style": "body" },
                    { "name": "actions", "type": "slot" }
                ]
            }
        },
        // ... elevated and outlined variants
    ]
}
```

### MODIFY - Change Existing Elements

```python
# Intent
{
    "operation": "modify",
    "target": {
        "id": "button-primary",
        "type": "component",
        "data": { /* existing button */ }
    },
    "parameters": {
        "changes": [
            { "property": "borderRadius", "value": "full" },
            { "property": "add_variant", "value": "icon-only" }
        ]
    }
}

# Result
{
    "operation": "modify",
    "success": true,
    "elements": [{ /* modified button */ }],
    "changes": [
        { "property": "borderRadius", "from": "md", "to": "full" },
        { "type": "variant_added", "name": "icon-only" }
    ]
}
```

### ANALYZE - Inspect Elements

```python
# Intent
{
    "operation": "analyze",
    "target": { /* component to analyze */ },
    "parameters": {
        "checks": ["accessibility", "consistency", "brand"]
    }
}

# Result
{
    "operation": "analyze",
    "success": true,
    "issues": [
        {
            "type": "accessibility",
            "severity": "error",
            "message": "Contrast ratio 3.2:1 below required 4.5:1",
            "element": "button-text",
            "suggestion": "Use color-primary-700 instead of color-primary-500"
        },
        {
            "type": "consistency",
            "severity": "warning",
            "message": "Border radius differs from design system standard",
            "element": "card-container"
        }
    ]
}
```

### SUGGEST - Propose Alternatives

```python
# Intent
{
    "operation": "suggest",
    "target": { /* current design */ },
    "parameters": {
        "goal": "improve visual hierarchy",
        "count": 3
    }
}

# Result
{
    "operation": "suggest",
    "success": true,
    "suggestions": [
        {
            "id": "suggestion-1",
            "description": "Increase heading size and add more spacing",
            "elements": [{ /* modified elements */ }],
            "rationale": "Larger headings create clearer hierarchy..."
        },
        // ... more suggestions
    ]
}
```

---

## UI Integration Patterns

### React Integration

```tsx
// Design tool component using Amplifier backend
function DesignCanvas({ designSystem }) {
  const [selection, setSelection] = useState(null);

  async function handleGenerateVariant() {
    const result = await amplifier.design({
      operation: "suggest",
      target: selection,
      parameters: {
        goal: "create alternative color schemes",
        count: 3
      }
    });

    // Result is structured - render directly
    showSuggestionPanel(result.suggestions);
  }

  async function handleAccessibilityCheck() {
    const result = await amplifier.design({
      operation: "analyze",
      target: selection,
      parameters: { checks: ["accessibility"] }
    });

    // Structured issues - highlight in UI
    highlightIssues(result.issues);
  }

  return (
    <Canvas>
      <ToolBar>
        <Button onClick={handleGenerateVariant}>
          Generate Variants
        </Button>
        <Button onClick={handleAccessibilityCheck}>
          Check Accessibility
        </Button>
      </ToolBar>
      <DesignSurface selection={selection} />
    </Canvas>
  );
}
```

### Figma Plugin Integration

```typescript
// Figma plugin using Amplifier for design operations
figma.ui.onmessage = async (msg) => {
  if (msg.type === "generate-component") {
    const selection = figma.currentPage.selection[0];

    const result = await amplifier.design({
      operation: "generate",
      parameters: {
        type: "component",
        based_on: serializeFigmaNode(selection),
        variants: msg.variants
      },
      constraints: ["match existing style"]
    });

    // Apply structured result to Figma
    for (const element of result.elements) {
      const node = createFigmaNode(element);
      figma.currentPage.appendChild(node);
    }
  }
};
```

---

## Events

```python
# Design operation events
"design:intent:receive"   # Intent received
"design:context:load"     # Design system loaded
"design:operation:start"  # Operation beginning
"design:operation:end"    # Operation complete
"design:validate"         # Validation running

# Element events
"design:element:create"   # New element created
"design:element:modify"   # Element modified
"design:element:suggest"  # Suggestions generated

# Issue events
"design:issue:found"      # Validation issue found
"design:issue:fixed"      # Issue resolved
```

---

## Examples

### CLI Usage

```bash
# Generate component from description
amplifier design generate \
  --type component \
  --name "SearchInput" \
  --description "Search input with icon and clear button" \
  --design-system ./tokens.json

# Analyze component for issues
amplifier design analyze \
  --input ./components/Button.json \
  --checks accessibility,consistency

# Generate color palette
amplifier design generate \
  --type palette \
  --base-color "#3b82f6" \
  --variants "light,dark,colorblind-safe"

# Suggest layout improvements
amplifier design suggest \
  --input ./layouts/dashboard.json \
  --goal "improve information hierarchy"
```

### Programmatic Usage

```python
from amplifier_core import AmplifierSession

session = AmplifierSession(profile="design-tool")

# Generate a component
result = await session.design(
    operation="generate",
    parameters={
        "type": "component",
        "name": "Avatar",
        "sizes": ["sm", "md", "lg"],
        "variants": ["image", "initials", "icon"]
    }
)

# Iterate on feedback
refined = await session.design(
    operation="modify",
    target=result.elements[0],
    parameters={
        "feedback": "make the initials variant more prominent"
    }
)
```

---

## Design System Enforcement

```python
class DesignSystemValidator:
    """Validates output against design system constraints."""

    def validate(self, element: DesignElement, system: dict) -> list[dict]:
        issues = []

        # Check token usage
        for prop, value in element.data.get("styles", {}).items():
            if not self.is_valid_token(value, system):
                issues.append({
                    "type": "token_violation",
                    "property": prop,
                    "value": value,
                    "suggestion": self.suggest_token(prop, value, system)
                })

        # Check accessibility
        if element.type == "component":
            a11y_issues = self.check_accessibility(element, system)
            issues.extend(a11y_issues)

        # Check consistency
        similar = self.find_similar_components(element, system)
        if similar:
            consistency_issues = self.check_consistency(element, similar)
            issues.extend(consistency_issues)

        return issues
```

---

## Testing Strategy

```python
class TestDesignToolOrchestrator:
    async def test_generate_uses_design_tokens(self):
        """Generated elements use design system tokens."""

    async def test_modify_preserves_constraints(self):
        """Modifications respect design system constraints."""

    async def test_analyze_finds_accessibility_issues(self):
        """Analysis catches WCAG violations."""

    async def test_suggest_provides_alternatives(self):
        """Suggestions are valid and different."""

    async def test_output_format_correct(self):
        """Output matches requested format (Figma/CSS/React)."""

    async def test_validation_enforcement(self):
        """Invalid designs are rejected or fixed."""
```

---

## Open Questions

1. **Design format standardization**: What's the canonical format for design elements?
2. **Real-time collaboration**: How to handle multiple designers editing?
3. **Version control**: How to track design evolution?
4. **Brand learning**: Can the system learn brand style from examples?
5. **Code generation**: How tight is the design-to-code pipeline?
