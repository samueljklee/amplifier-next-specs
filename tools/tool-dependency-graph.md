# tool-dependency-graph

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-tool-dependency-graph`

## Overview

Analyzes and visualizes code dependencies at multiple levels: imports, function calls, class inheritance, and module relationships. Essential for impact analysis, refactoring planning, and understanding complex codebases.

### Value Proposition

| Without | With |
|---------|------|
| Manual tracing through files to find callers | `get_callers("PaymentGateway.process")` → instant list |
| Risky refactoring without knowing impact | "What breaks if I change this?" → complete dependency tree |
| Circular dependency surprises at runtime | Static analysis catches cycles before they cause problems |
| Architecture documentation always stale | Live dependency graphs always current |

### Use Cases

1. **Impact analysis**: "What code is affected if I change this function?"
2. **Refactoring safety**: "Can I safely rename/move this class?"
3. **Dead code detection**: "What code has no callers?"
4. **Architecture visualization**: "Show me the dependency structure of this module"
5. **Circular dependency detection**: "Are there any circular imports?"
6. **Coupling analysis**: "Which modules are most tightly coupled?"

---

## Contract

### Tool Definition

```python
TOOL_DEFINITION = {
    "name": "dependency_graph",
    "description": """
    Analyze code dependencies: imports, calls, inheritance, and module relationships.

    Operations:
    - get_dependents: What depends on this symbol/file? (callers, importers)
    - get_dependencies: What does this symbol/file depend on? (callees, imports)
    - get_graph: Full dependency graph for a scope
    - find_cycles: Detect circular dependencies
    - find_dead_code: Find code with no dependents
    - coupling_analysis: Measure coupling between modules

    Use for impact analysis, refactoring, and architecture understanding.
    """,
    "parameters": {
        "type": "object",
        "properties": {
            "operation": {
                "type": "string",
                "enum": ["get_dependents", "get_dependencies", "get_graph", "find_cycles", "find_dead_code", "coupling_analysis"],
                "description": "Analysis operation to perform"
            },
            "target": {
                "type": "string",
                "description": "Symbol (function, class) or file path to analyze"
            },
            "scope": {
                "type": "string",
                "description": "Limit analysis to path pattern (e.g., 'src/', '**/*.py')"
            },
            "depth": {
                "type": "integer",
                "default": 3,
                "description": "Maximum depth for recursive analysis"
            },
            "include_external": {
                "type": "boolean",
                "default": false,
                "description": "Include external library dependencies"
            },
            "dependency_types": {
                "type": "array",
                "items": {
                    "type": "string",
                    "enum": ["import", "call", "inheritance", "type_reference", "all"]
                },
                "default": ["all"],
                "description": "Types of dependencies to include"
            },
            "output_format": {
                "type": "string",
                "enum": ["tree", "list", "graph", "mermaid"],
                "default": "tree",
                "description": "Output format for results"
            }
        },
        "required": ["operation"]
    }
}
```

### Input Schema

```python
@dataclass
class DependencyGraphInput:
    operation: Operation                # get_dependents | get_dependencies | get_graph | find_cycles | find_dead_code | coupling_analysis
    target: str | None = None           # Symbol or file path
    scope: str | None = None            # Path pattern to limit analysis
    depth: int = 3                       # Max recursion depth
    include_external: bool = False       # Include third-party dependencies
    dependency_types: list[str] = field(default_factory=lambda: ["all"])
    output_format: str = "tree"          # tree | list | graph | mermaid
```

### Output Schema

```python
@dataclass
class DependencyGraphOutput:
    operation: str
    target: str | None

    # For get_dependents/get_dependencies
    dependencies: list[Dependency] | None = None

    # For get_graph
    graph: DependencyGraph | None = None

    # For find_cycles
    cycles: list[Cycle] | None = None

    # For find_dead_code
    dead_code: list[DeadCode] | None = None

    # For coupling_analysis
    coupling: CouplingAnalysis | None = None

    # Formatted output
    formatted: str | None = None         # Formatted per output_format

    # Stats
    stats: AnalysisStats

@dataclass
class Dependency:
    source: Symbol                       # Where the dependency originates
    target: Symbol                       # What it depends on
    type: str                            # import | call | inheritance | type_reference
    location: Location                   # File and line
    context: str | None                  # Code snippet showing the dependency

@dataclass
class Symbol:
    name: str                            # Function/class/module name
    qualified_name: str                  # Full path (module.Class.method)
    kind: str                            # function | class | module | variable
    file_path: str
    line_number: int

@dataclass
class Location:
    file_path: str
    line_number: int
    column: int | None = None

@dataclass
class DependencyGraph:
    nodes: list[GraphNode]
    edges: list[GraphEdge]
    root: str | None                     # Root node if tree structure

@dataclass
class GraphNode:
    id: str                              # Unique identifier
    symbol: Symbol
    metadata: dict                       # Additional info (size, complexity, etc.)

@dataclass
class GraphEdge:
    source_id: str
    target_id: str
    type: str
    weight: int = 1                      # Number of dependencies of this type

@dataclass
class Cycle:
    path: list[str]                      # Symbols in the cycle
    type: str                            # import | call
    severity: str                        # error | warning (import cycles more severe)

@dataclass
class DeadCode:
    symbol: Symbol
    kind: str                            # function | class | module | variable
    reason: str                          # "No callers found", "Only test references", etc.
    confidence: float                    # 0-1 confidence this is truly dead

@dataclass
class CouplingAnalysis:
    module_pairs: list[ModuleCoupling]
    metrics: CouplingMetrics

@dataclass
class ModuleCoupling:
    module_a: str
    module_b: str
    coupling_score: float                # 0-1, higher = more coupled
    dependency_count: int
    bidirectional: bool                  # Both depend on each other

@dataclass
class CouplingMetrics:
    average_coupling: float
    max_coupling: float
    highly_coupled_pairs: int            # Pairs above threshold
    suggested_refactors: list[str]       # Suggestions to reduce coupling

@dataclass
class AnalysisStats:
    files_analyzed: int
    symbols_found: int
    dependencies_found: int
    analysis_time_ms: int
```

### Events Emitted

| Event | When | Data |
|-------|------|------|
| `tool:dependency_graph:start` | Analysis begins | operation, target, scope |
| `tool:dependency_graph:parsing` | File parsing | file_path, symbols_found |
| `tool:dependency_graph:complete` | Analysis done | stats |
| `tool:dependency_graph:error` | Analysis failed | error_type, message |

---

## Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    tool-dependency-graph                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │   Symbol    │───▶│   Graph     │───▶│   Output    │         │
│  │  Resolver   │    │   Builder   │    │  Formatter  │         │
│  └─────────────┘    └──────┬──────┘    └─────────────┘         │
│                            │                                    │
│         ┌──────────────────┼──────────────────┐                │
│         ▼                  ▼                  ▼                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │   Python    │    │ TypeScript  │    │    Other    │         │
│  │   Analyzer  │    │  Analyzer   │    │  Analyzers  │         │
│  └─────────────┘    └─────────────┘    └─────────────┘         │
│         │                  │                  │                 │
│         ▼                  ▼                  ▼                 │
│  ┌─────────────────────────────────────────────────┐           │
│  │              TreeSitter Parsers                 │           │
│  └─────────────────────────────────────────────────┘           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Internal Components

#### 1. Language-Specific Analyzers

```python
class PythonAnalyzer:
    """
    Python-specific dependency analysis using TreeSitter and AST.
    """

    def __init__(self):
        self.parser = TreeSitterParser("python")

    async def analyze_file(self, file_path: Path) -> FileAnalysis:
        content = file_path.read_text()
        tree = self.parser.parse(content)

        analysis = FileAnalysis(file_path=str(file_path))

        # Extract symbols (functions, classes, variables)
        analysis.symbols = self._extract_symbols(tree, file_path)

        # Extract imports
        analysis.imports = self._extract_imports(tree)

        # Extract function calls
        analysis.calls = self._extract_calls(tree)

        # Extract inheritance
        analysis.inheritance = self._extract_inheritance(tree)

        # Extract type hints
        analysis.type_refs = self._extract_type_references(tree)

        return analysis

    def _extract_imports(self, tree: Tree) -> list[ImportDep]:
        """Extract all import statements."""
        imports = []

        # import x, import x.y
        for node in self._find_nodes(tree, "import_statement"):
            module = self._get_module_name(node)
            imports.append(ImportDep(
                module=module,
                names=None,  # Imports whole module
                location=self._get_location(node)
            ))

        # from x import y, z
        for node in self._find_nodes(tree, "import_from_statement"):
            module = self._get_from_module(node)
            names = self._get_imported_names(node)
            imports.append(ImportDep(
                module=module,
                names=names,
                location=self._get_location(node)
            ))

        return imports

    def _extract_calls(self, tree: Tree) -> list[CallDep]:
        """Extract all function/method calls."""
        calls = []

        for node in self._find_nodes(tree, "call"):
            callee = self._resolve_callee(node)
            if callee:
                calls.append(CallDep(
                    caller=self._get_enclosing_function(node),
                    callee=callee,
                    location=self._get_location(node)
                ))

        return calls
```

#### 2. Graph Builder

```python
class GraphBuilder:
    """
    Builds dependency graph from file analyses.
    Handles cross-file resolution and graph algorithms.
    """

    def __init__(self, analyses: list[FileAnalysis]):
        self.analyses = {a.file_path: a for a in analyses}
        self.symbol_index = self._build_symbol_index()

    def get_dependents(self, target: str, depth: int, types: list[str]) -> list[Dependency]:
        """Find all symbols that depend on target."""
        target_symbol = self._resolve_symbol(target)
        if not target_symbol:
            raise ValueError(f"Symbol not found: {target}")

        dependents = []
        visited = set()

        self._find_dependents_recursive(
            target_symbol, depth, types, dependents, visited
        )

        return dependents

    def get_dependencies(self, target: str, depth: int, types: list[str]) -> list[Dependency]:
        """Find all symbols that target depends on."""
        target_symbol = self._resolve_symbol(target)
        if not target_symbol:
            raise ValueError(f"Symbol not found: {target}")

        dependencies = []
        visited = set()

        self._find_dependencies_recursive(
            target_symbol, depth, types, dependencies, visited
        )

        return dependencies

    def find_cycles(self, scope: str | None = None) -> list[Cycle]:
        """Detect circular dependencies."""
        # Build directed graph
        graph = self._build_adjacency_graph(scope)

        # Find strongly connected components (Tarjan's algorithm)
        sccs = self._find_sccs(graph)

        # SCCs with more than one node are cycles
        cycles = []
        for scc in sccs:
            if len(scc) > 1:
                cycles.append(Cycle(
                    path=scc,
                    type=self._determine_cycle_type(scc),
                    severity="error" if self._is_import_cycle(scc) else "warning"
                ))

        return cycles

    def find_dead_code(self, scope: str | None = None) -> list[DeadCode]:
        """Find symbols with no dependents."""
        dead = []

        for symbol in self._get_symbols_in_scope(scope):
            # Skip entry points and special methods
            if self._is_entry_point(symbol):
                continue

            dependents = self.get_dependents(symbol.qualified_name, depth=1, types=["all"])

            # Filter out test-only references
            non_test_dependents = [d for d in dependents if not self._is_test_file(d.source.file_path)]

            if not non_test_dependents:
                confidence = 1.0
                reason = "No callers found"

                # Lower confidence if there are test references
                if dependents:
                    confidence = 0.7
                    reason = "Only test references"

                # Lower confidence for certain patterns (callbacks, plugins)
                if self._might_be_dynamic(symbol):
                    confidence = 0.5
                    reason += " (may be used dynamically)"

                dead.append(DeadCode(
                    symbol=symbol,
                    kind=symbol.kind,
                    reason=reason,
                    confidence=confidence
                ))

        return dead

    def coupling_analysis(self, scope: str | None = None) -> CouplingAnalysis:
        """Analyze module coupling."""
        modules = self._get_modules_in_scope(scope)
        pairs = []

        for i, mod_a in enumerate(modules):
            for mod_b in modules[i+1:]:
                # Count dependencies between modules
                a_to_b = self._count_dependencies(mod_a, mod_b)
                b_to_a = self._count_dependencies(mod_b, mod_a)

                if a_to_b > 0 or b_to_a > 0:
                    total = a_to_b + b_to_a
                    # Normalize by module sizes
                    coupling_score = self._calculate_coupling_score(mod_a, mod_b, total)

                    pairs.append(ModuleCoupling(
                        module_a=mod_a,
                        module_b=mod_b,
                        coupling_score=coupling_score,
                        dependency_count=total,
                        bidirectional=a_to_b > 0 and b_to_a > 0
                    ))

        # Sort by coupling score
        pairs.sort(key=lambda p: p.coupling_score, reverse=True)

        return CouplingAnalysis(
            module_pairs=pairs,
            metrics=self._calculate_metrics(pairs)
        )
```

#### 3. Output Formatter

```python
class OutputFormatter:
    """Format analysis results for different output formats."""

    def format(self, result: Any, format: str) -> str:
        formatters = {
            "tree": self._format_tree,
            "list": self._format_list,
            "graph": self._format_graph_json,
            "mermaid": self._format_mermaid,
        }
        return formatters[format](result)

    def _format_tree(self, dependencies: list[Dependency]) -> str:
        """Format as indented tree."""
        lines = []

        def add_node(dep: Dependency, depth: int):
            indent = "  " * depth
            prefix = "├── " if depth > 0 else ""
            lines.append(f"{indent}{prefix}{dep.target.name} ({dep.type})")
            lines.append(f"{indent}    └── {dep.location.file_path}:{dep.location.line_number}")

        for dep in dependencies:
            add_node(dep, 0)

        return "\n".join(lines)

    def _format_mermaid(self, graph: DependencyGraph) -> str:
        """Format as Mermaid diagram."""
        lines = ["graph TD"]

        # Add nodes
        for node in graph.nodes:
            label = f"{node.symbol.name}"
            lines.append(f"    {node.id}[{label}]")

        # Add edges
        for edge in graph.edges:
            arrow = "-->" if edge.type == "call" else "-.->|{edge.type}|"
            lines.append(f"    {edge.source_id} {arrow} {edge.target_id}")

        return "\n".join(lines)
```

---

## Configuration

```yaml
tool-dependency-graph:
  # Analysis settings
  analysis:
    # Languages to analyze (auto-detected if not specified)
    languages:
      - python
      - typescript
      - javascript

    # File patterns to include
    include_patterns:
      - "**/*.py"
      - "**/*.ts"
      - "**/*.tsx"
      - "**/*.js"
      - "**/*.jsx"

    # Patterns to exclude
    exclude_patterns:
      - "node_modules/**"
      - ".venv/**"
      - "**/__pycache__/**"
      - "**/dist/**"
      - "**/*.test.*"
      - "**/*.spec.*"

    # External dependencies
    include_external: false
    external_packages:
      # Packages to always exclude from analysis
      exclude:
        - "typing"
        - "typing_extensions"
        - "builtins"

  # Graph building
  graph:
    # Maximum depth for recursive analysis
    default_depth: 3
    max_depth: 10

    # Entry points (roots for dead code detection)
    entry_points:
      patterns:
        - "**/main.py"
        - "**/cli.py"
        - "**/__main__.py"
        - "**/index.ts"
      functions:
        - "main"
        - "cli"
        - "app"

    # Symbols to exclude from dead code detection
    exclude_from_dead_code:
      patterns:
        - "__*__"           # Dunder methods
        - "test_*"          # Test functions
        - "*_test"
      decorators:
        - "@app.route"      # Flask routes
        - "@pytest.fixture" # Pytest fixtures
        - "@click.command"  # CLI commands

  # Output
  output:
    default_format: "tree"

    # Mermaid diagram settings
    mermaid:
      direction: "TD"       # TB, TD, LR, RL
      max_nodes: 50         # Limit for readability

  # Caching
  cache:
    enabled: true
    ttl_seconds: 300
    invalidate_on_file_change: true
```

---

## Examples

### Example 1: Impact Analysis

```python
# Input
{
    "operation": "get_dependents",
    "target": "PaymentGateway.process",
    "depth": 2,
    "output_format": "tree"
}

# Output
{
    "operation": "get_dependents",
    "target": "PaymentGateway.process",
    "dependencies": [
        {
            "source": {"name": "OrderService.checkout", "file_path": "src/orders/service.py"},
            "target": {"name": "PaymentGateway.process", "file_path": "src/payments/gateway.py"},
            "type": "call",
            "location": {"file_path": "src/orders/service.py", "line_number": 89}
        },
        {
            "source": {"name": "SubscriptionRenewer.renew", "file_path": "src/subscriptions/renewer.py"},
            "target": {"name": "PaymentGateway.process", "file_path": "src/payments/gateway.py"},
            "type": "call",
            "location": {"file_path": "src/subscriptions/renewer.py", "line_number": 45}
        }
    ],
    "formatted": "PaymentGateway.process\n├── OrderService.checkout (call)\n│   └── src/orders/service.py:89\n├── SubscriptionRenewer.renew (call)\n│   └── src/subscriptions/renewer.py:45",
    "stats": {
        "files_analyzed": 45,
        "symbols_found": 234,
        "dependencies_found": 2,
        "analysis_time_ms": 156
    }
}
```

### Example 2: Circular Dependency Detection

```python
# Input
{
    "operation": "find_cycles",
    "scope": "src/",
    "dependency_types": ["import"]
}

# Output
{
    "operation": "find_cycles",
    "cycles": [
        {
            "path": ["src/models/user.py", "src/services/auth.py", "src/models/session.py", "src/models/user.py"],
            "type": "import",
            "severity": "error"
        }
    ],
    "formatted": "Circular import detected:\n  src/models/user.py\n  → src/services/auth.py\n  → src/models/session.py\n  → src/models/user.py (cycle)",
    "stats": {...}
}
```

### Example 3: Coupling Analysis

```python
# Input
{
    "operation": "coupling_analysis",
    "scope": "src/"
}

# Output
{
    "operation": "coupling_analysis",
    "coupling": {
        "module_pairs": [
            {
                "module_a": "src/orders",
                "module_b": "src/payments",
                "coupling_score": 0.85,
                "dependency_count": 23,
                "bidirectional": true
            },
            {
                "module_a": "src/users",
                "module_b": "src/auth",
                "coupling_score": 0.72,
                "dependency_count": 18,
                "bidirectional": false
            }
        ],
        "metrics": {
            "average_coupling": 0.34,
            "max_coupling": 0.85,
            "highly_coupled_pairs": 2,
            "suggested_refactors": [
                "Consider extracting shared types between src/orders and src/payments",
                "src/payments depends heavily on src/orders internals - consider interface extraction"
            ]
        }
    }
}
```

### Example 4: Mermaid Architecture Diagram

```python
# Input
{
    "operation": "get_graph",
    "scope": "src/api/",
    "depth": 1,
    "output_format": "mermaid"
}

# Output
{
    "operation": "get_graph",
    "graph": {...},
    "formatted": "graph TD\n    api_users[users.py]\n    api_orders[orders.py]\n    api_payments[payments.py]\n    svc_user[UserService]\n    svc_order[OrderService]\n    api_users --> svc_user\n    api_orders --> svc_order\n    api_orders --> svc_user\n    api_payments --> svc_order"
}
```

---

## Security Considerations

### Data Access

- Reads source files within configured scope
- Does not execute code, only static analysis
- No network access required

### Risks

- Large codebases may consume significant memory
- Analysis results may reveal code structure

### Permissions

```python
REQUIRED_CAPABILITIES = [
    "filesystem:read",      # Read source files
]
```

---

## Testing Strategy

### Unit Tests

```python
def test_python_analyzer_extracts_imports():
    analyzer = PythonAnalyzer()
    code = "from foo.bar import baz, qux"
    analysis = analyzer.analyze_string(code)

    assert len(analysis.imports) == 1
    assert analysis.imports[0].module == "foo.bar"
    assert analysis.imports[0].names == ["baz", "qux"]

def test_cycle_detection_finds_cycles():
    builder = GraphBuilder([
        FileAnalysis("a.py", imports=[ImportDep("b")]),
        FileAnalysis("b.py", imports=[ImportDep("c")]),
        FileAnalysis("c.py", imports=[ImportDep("a")]),
    ])

    cycles = builder.find_cycles()
    assert len(cycles) == 1
    assert set(cycles[0].path) == {"a.py", "b.py", "c.py"}
```

### Integration Tests

```python
async def test_real_codebase_analysis():
    tool = DependencyGraphTool(config)
    result = await tool.execute({
        "operation": "get_dependents",
        "target": "test_function",
        "scope": "tests/fixtures/"
    })

    assert result.stats.files_analyzed > 0
    # Known fixture has specific dependencies
    assert len(result.dependencies) == 3
```

---

## Dependencies

### Required

- `tree-sitter` - Language parsing
- `tree-sitter-python` - Python grammar
- `tree-sitter-typescript` - TypeScript grammar
- `tree-sitter-javascript` - JavaScript grammar

### Optional

- Additional language grammars as needed

---

## Open Questions

1. **Language support priority**: Which languages beyond Python/TypeScript?
   - Go, Rust, Java are common enterprise languages
   - Each requires grammar and analyzer implementation

2. **Dynamic dependencies**: How to handle dynamic imports/calls?
   - `importlib.import_module()`, `getattr()`, etc.
   - Could warn about detected dynamic patterns

3. **Cross-repository analysis**: Should we support analyzing dependencies across repos?
   - Useful for microservices
   - Significantly increases complexity

4. **Incremental updates**: How to efficiently update graph on file changes?
   - Full rebuild vs incremental
   - Trade-off: complexity vs performance

5. **Visual output**: Should we generate SVG/PNG diagrams?
   - Mermaid can be rendered by many tools
   - Direct image generation adds dependencies

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
