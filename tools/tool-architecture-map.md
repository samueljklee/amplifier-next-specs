# tool-architecture-map

> **Priority**: P2 (Enhancement)
> **Status**: Draft
> **Module**: `amplifier-module-tool-architecture-map`

## Overview

Generate and maintain architecture diagrams from code analysis. Creates visual representations of system structure, data flows, and component relationships that stay synchronized with the actual codebase.

### Value Proposition

| Without | With |
|---------|------|
| Architecture docs always outdated | Diagrams generated from live code |
| Manual diagram maintenance | Auto-update on codebase changes |
| Inconsistent diagram styles | Consistent, templated output |
| Hours in diagramming tools | Instant generation from code analysis |

### Use Cases

1. **System overview**: Generate high-level architecture diagram
2. **Component deep-dive**: Detailed view of specific service/module
3. **Data flow visualization**: How data moves through the system
4. **Onboarding**: Quick visual understanding for new developers
5. **Documentation sync**: Keep architecture docs current

---

## Contract

### Tool Definition

```python
TOOL_DEFINITION = {
    "name": "architecture_map",
    "description": """
    Generate architecture diagrams from code analysis.

    Diagram types:
    - system: High-level system/service architecture
    - component: Internal component structure
    - data_flow: Data movement through system
    - deployment: Infrastructure/deployment view
    - sequence: Interaction sequences

    Output formats: mermaid, plantuml, dot, svg, png
    """,
    "parameters": {
        "type": "object",
        "properties": {
            "diagram_type": {
                "type": "string",
                "enum": ["system", "component", "data_flow", "deployment", "sequence"],
                "description": "Type of diagram to generate"
            },
            "scope": {
                "type": "string",
                "description": "Path or pattern to analyze (e.g., 'src/services/', 'PaymentService')"
            },
            "depth": {
                "type": "integer",
                "default": 2,
                "description": "Level of detail (1=high-level, 3=detailed)"
            },
            "output_format": {
                "type": "string",
                "enum": ["mermaid", "plantuml", "dot", "svg", "png"],
                "default": "mermaid",
                "description": "Output format"
            },
            "include_external": {
                "type": "boolean",
                "default": true,
                "description": "Include external services/dependencies"
            },
            "group_by": {
                "type": "string",
                "enum": ["module", "layer", "domain", "none"],
                "default": "module",
                "description": "How to group components"
            }
        },
        "required": ["diagram_type"]
    }
}
```

### Output Schema

```python
@dataclass
class ArchitectureMapOutput:
    diagram_type: str
    scope: str

    # Diagram content
    diagram: str                        # In requested format
    format: str

    # Metadata
    components: list[Component]
    relationships: list[Relationship]
    external_services: list[ExternalService]

    # Analysis summary
    summary: ArchitectureSummary

@dataclass
class Component:
    id: str
    name: str
    type: str                           # service | module | class | database | queue
    description: str | None
    source_path: str | None
    layer: str | None                   # api | service | data | infrastructure
    metrics: ComponentMetrics | None

@dataclass
class Relationship:
    source_id: str
    target_id: str
    type: str                           # calls | uses | stores | publishes | subscribes
    description: str | None
    async_: bool
    protocol: str | None                # http | grpc | amqp | sql

@dataclass
class ExternalService:
    name: str
    type: str                           # database | cache | queue | api | storage
    provider: str | None                # aws | gcp | azure | self-hosted

@dataclass
class ArchitectureSummary:
    total_components: int
    total_relationships: int
    layers_detected: list[str]
    patterns_detected: list[str]        # "microservices", "monolith", "event-driven"
    suggestions: list[str]              # Architecture improvement suggestions
```

---

## Architecture

### Diagram Generation Pipeline

```python
class ArchitectureMapper:
    """Generate architecture diagrams from code analysis."""

    async def generate(self, input: ArchitectureMapInput) -> ArchitectureMapOutput:
        # 1. Analyze codebase structure
        components = await self._discover_components(input.scope, input.depth)

        # 2. Analyze relationships
        relationships = await self._analyze_relationships(components)

        # 3. Detect external services
        external = await self._detect_external_services(components)

        # 4. Apply grouping/layout
        layout = self._compute_layout(components, relationships, input.group_by)

        # 5. Generate diagram
        diagram = self._render_diagram(
            components, relationships, external, layout,
            input.diagram_type, input.output_format
        )

        # 6. Analyze architecture patterns
        summary = self._analyze_architecture(components, relationships)

        return ArchitectureMapOutput(
            diagram_type=input.diagram_type,
            scope=input.scope,
            diagram=diagram,
            format=input.output_format,
            components=components,
            relationships=relationships,
            external_services=external,
            summary=summary
        )

    async def _discover_components(self, scope: str, depth: int) -> list[Component]:
        """Discover components from code structure."""
        components = []

        # Depth 1: Services/top-level modules
        if depth >= 1:
            services = await self._find_services(scope)
            components.extend(services)

        # Depth 2: Major classes/modules within services
        if depth >= 2:
            for service in services:
                modules = await self._find_modules(service.source_path)
                components.extend(modules)

        # Depth 3: Individual classes and functions
        if depth >= 3:
            for module in components:
                classes = await self._find_classes(module.source_path)
                components.extend(classes)

        return components

    def _render_diagram(
        self,
        components: list[Component],
        relationships: list[Relationship],
        external: list[ExternalService],
        layout: Layout,
        diagram_type: str,
        format: str
    ) -> str:
        """Render diagram in requested format."""
        renderers = {
            "mermaid": MermaidRenderer(),
            "plantuml": PlantUMLRenderer(),
            "dot": GraphvizRenderer(),
        }

        renderer = renderers[format]
        return renderer.render(components, relationships, external, layout, diagram_type)
```

### Mermaid Renderer

```python
class MermaidRenderer:
    """Render diagrams in Mermaid format."""

    def render_system(
        self,
        components: list[Component],
        relationships: list[Relationship],
        external: list[ExternalService]
    ) -> str:
        lines = ["graph TB"]

        # Add subgraphs for groupings
        groups = self._group_components(components)
        for group_name, group_components in groups.items():
            lines.append(f"    subgraph {group_name}")
            for comp in group_components:
                shape = self._get_shape(comp.type)
                lines.append(f"        {comp.id}{shape}")
            lines.append("    end")

        # Add external services
        if external:
            lines.append("    subgraph External")
            for ext in external:
                lines.append(f"        {ext.name}[({ext.name})]")
            lines.append("    end")

        # Add relationships
        for rel in relationships:
            arrow = self._get_arrow(rel.type, rel.async_)
            label = f"|{rel.type}|" if rel.type else ""
            lines.append(f"    {rel.source_id} {arrow}{label} {rel.target_id}")

        return "\n".join(lines)

    def _get_shape(self, component_type: str) -> str:
        shapes = {
            "service": "[{}]",          # Rectangle
            "database": "[({})]",       # Cylinder
            "queue": "{{{}}}",          # Hexagon
            "api": "([{}])",            # Stadium
            "module": "({})",           # Rounded
        }
        return shapes.get(component_type, "[{}]")
```

---

## Examples

### Example 1: System Architecture Diagram

```python
# Input
{
    "diagram_type": "system",
    "scope": "src/services/",
    "depth": 1,
    "output_format": "mermaid"
}

# Output
{
    "diagram_type": "system",
    "diagram": """graph TB
    subgraph Services
        api-gateway[API Gateway]
        user-service[User Service]
        payment-service[Payment Service]
        order-service[Order Service]
    end
    subgraph Data
        postgres[(PostgreSQL)]
        redis[(Redis)]
    end
    subgraph External
        stripe([Stripe API])
        sendgrid([SendGrid])
    end

    api-gateway -->|REST| user-service
    api-gateway -->|REST| order-service
    order-service -->|gRPC| payment-service
    payment-service -->|HTTPS| stripe
    user-service --> postgres
    payment-service --> postgres
    user-service --> redis
    order-service -->|email| sendgrid""",
    "format": "mermaid",
    "summary": {
        "total_components": 4,
        "layers_detected": ["api", "service", "data"],
        "patterns_detected": ["microservices", "api-gateway"],
        "suggestions": [
            "Consider adding circuit breakers for external service calls",
            "payment-service has high coupling with order-service"
        ]
    }
}
```

### Example 2: Component Diagram

```python
# Input
{
    "diagram_type": "component",
    "scope": "src/services/payment/",
    "depth": 2
}

# Output
{
    "diagram": """graph TB
    subgraph payment-service
        subgraph API Layer
            PaymentController
            WebhookController
        end
        subgraph Service Layer
            PaymentService
            RefundService
            WebhookProcessor
        end
        subgraph Data Layer
            PaymentRepository
            TransactionRepository
        end
    end

    PaymentController --> PaymentService
    WebhookController --> WebhookProcessor
    PaymentService --> PaymentRepository
    PaymentService --> RefundService
    RefundService --> TransactionRepository"""
}
```

---

## Configuration

```yaml
tool-architecture-map:
  # Analysis settings
  analysis:
    # Patterns to identify services
    service_patterns:
      - "**/services/*/"
      - "**/apps/*/"
      - "**/microservices/*/"

    # Patterns to identify modules
    module_patterns:
      - "**/*_service.py"
      - "**/*_controller.py"
      - "**/*_repository.py"

    # Layer detection
    layers:
      api: ["controller", "handler", "endpoint", "router"]
      service: ["service", "usecase", "interactor"]
      data: ["repository", "dao", "model", "entity"]
      infrastructure: ["client", "adapter", "gateway"]

  # External service detection
  external_services:
    # Detect from import patterns
    patterns:
      stripe: ["stripe"]
      aws: ["boto3", "aws_sdk"]
      redis: ["redis", "aioredis"]
      postgres: ["psycopg", "asyncpg", "sqlalchemy"]

  # Output settings
  output:
    default_format: "mermaid"
    include_metrics: false              # Include LOC, complexity, etc.
```

---

## Security Considerations

- Read-only analysis of codebase
- May reveal system architecture in outputs
- No external network access required

---

## Dependencies

### Required
- `tree-sitter` - Code parsing

### Optional
- `graphviz` - For DOT/SVG/PNG output
- `plantuml` - For PlantUML rendering

---

## Open Questions

1. **Real-time updates**: Should diagrams update as files change?
2. **Diagram storage**: Where to persist generated diagrams?
3. **Interactive diagrams**: Support for clickable/zoomable output?
4. **Custom styling**: Allow custom themes/colors?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
