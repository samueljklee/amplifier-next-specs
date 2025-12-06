# context-codebase-knowledge-graph

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-context-codebase-knowledge-graph`

## Overview

A context manager that maintains a living knowledge graph of the codebase structure (files, functions, classes, dependencies, data flow) and injects relevant subgraphs into conversation context. Enables graph-based queries like "What depends on X?", "How does data flow from A to B?", and "What's the impact radius of changing Y?"

### Value Proposition

| Without | With |
|---------|------|
| Flat file view of codebase | Relationship-aware graph view |
| "What depends on this?" requires manual tracing | Graph traversal shows all dependencies instantly |
| Impact analysis is guesswork | Precise impact radius calculation |
| Dead code unknown | Graph reachability detects unused code |

### Use Cases

1. **Refactoring impact analysis**: "What breaks if I change this module?"
2. **Dependency tracing**: "What does this function call? What calls it?"
3. **Data flow analysis**: "How does user data get to the database?"
4. **Dead code detection**: "What code is unreachable from entry points?"
5. **Architecture understanding**: "Show me the call graph for authentication"

---

## Contract

### ContextManager Interface

```python
class CodebaseKnowledgeGraphContext:
    """
    Context manager with codebase graph integration.
    
    Extends standard context with:
    - Codebase graph construction and maintenance
    - Graph-based queries
    - Relevant subgraph injection into context
    - Impact analysis
    """
    
    async def add_message(
        self, 
        message: dict[str, Any],
    ) -> None:
        """
        Add message to context.
        
        If message mentions code entities:
        1. Query graph for related entities
        2. Inject relevant subgraph into context
        """
    
    async def get_messages(
        self,
        include_graph_context: bool = True
    ) -> list[dict[str, Any]]:
        """
        Get messages with optional graph context.
        
        Args:
            include_graph_context: Include relevant graph data
        """
    
    # Graph query methods
    
    async def find_dependencies(
        self,
        entity: str
    ) -> list[Dependency]:
        """Find all dependencies of an entity."""
    
    async def find_dependents(
        self,
        entity: str
    ) -> list[Dependent]:
        """Find all entities that depend on this entity."""
    
    async def trace_call_chain(
        self,
        from_func: str,
        to_func: str
    ) -> list[CallPath]:
        """Find call paths between functions."""
    
    async def calculate_impact_radius(
        self,
        changed_files: list[str]
    ) -> ImpactAnalysis:
        """Calculate impact of changing files."""
    
    async def detect_circular_dependencies(self) -> list[Cycle]:
        """Find circular dependency chains."""
    
    async def find_unused_code(
        self,
        entry_points: list[str]
    ) -> list[str]:
        """Find code unreachable from entry points."""
```

### Configuration Schema

```toml
[[contexts]]
module = "context-codebase-knowledge-graph"
config = {
  # Graph construction
  codebase_paths = ["src/", "lib/"]
  exclude_paths = ["node_modules/", "dist/", ".git/"]
  languages = ["typescript", "python", "rust"]
  
  # Graph storage
  graph_database = "neo4j://localhost:7687"  # or "sqlite", "memory"
  persistence = true
  incremental_updates = true
  
  # Indexing strategy
  index_on_startup = false                   # Build graph on first use
  watch_for_changes = true                   # Update graph on file changes
  debounce_ms = 1000                         # Debounce rapid changes
  
  # Node types to track
  track_files = true
  track_functions = true
  track_classes = true
  track_variables = true
  track_imports = true
  track_tables = true                        # Database tables
  
  # Relationship types to track
  track_imports_relations = true             # File imports file
  track_calls_relations = true               # Function calls function
  track_inherits_relations = true            # Class inherits class
  track_modifies_relations = true            # Function modifies table
  track_reads_relations = true               # Function reads table
  
  # Context injection
  auto_inject_graph = true                   # Auto-inject relevant subgraph
  max_depth = 3                              # Max traversal depth
  max_nodes = 50                             # Max nodes in subgraph
  
  # Query optimization
  cache_queries = true
  cache_ttl_seconds = 300
}
```

### Data Schema

```python
# Graph node types
@dataclass
class FileNode:
    path: str
    language: str
    lines_of_code: int
    last_modified: datetime

@dataclass
class FunctionNode:
    name: str
    file: str
    signature: str
    line_start: int
    line_end: int

@dataclass
class ClassNode:
    name: str
    file: str
    methods: list[str]
    line_start: int

@dataclass
class TableNode:
    name: str
    database: str
    columns: list[str]

# Graph relationship types
@dataclass
class ImportsRelation:
    from_file: str
    to_file: str

@dataclass
class CallsRelation:
    from_func: str
    to_func: str
    call_count: int

@dataclass
class InheritsRelation:
    child_class: str
    parent_class: str

@dataclass
class ModifiesRelation:
    function: str
    table: str
    operation: str  # INSERT, UPDATE, DELETE

# Query results
@dataclass
class Dependency:
    entity: str
    type: str
    relationship: str

@dataclass
class ImpactAnalysis:
    directly_affected: list[str]      # Files directly importing changed files
    indirectly_affected: list[str]    # Files transitively affected
    estimated_risk: str               # "low" | "medium" | "high"
    affected_count: int
    depth_levels: dict[int, list[str]]  # {depth: [files at that depth]}
```

### Events Emitted

| Event | When | Data |
|-------|------|------|
| `context:graph:building` | Graph construction starts | codebase_path, file_count |
| `context:graph:built` | Graph construction complete | node_count, edge_count, duration_ms |
| `context:graph:updated` | Incremental update | changed_files, updated_nodes |
| `context:graph:queried` | Graph query executed | query_type, result_count |
| `context:graph:injected` | Subgraph injected into context | entity, subgraph_size |

---

## Architecture

### Component Diagram

```
┌────────────────────────────────────────────────────────────┐
│          context-codebase-knowledge-graph                  │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌──────────────┐    ┌──────────────┐    ┌────────────┐   │
│  │  Codebase    │───▶│    Graph     │───▶│   Query    │   │
│  │   Parser     │    │   Builder    │    │   Engine   │   │
│  └──────────────┘    └──────────────┘    └──────┬─────┘   │
│                                                  │         │
│                                                  │         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────▼─────┐   │
│  │   Change     │    │  Subgraph    │    │   Graph    │   │
│  │   Watcher    │    │  Extractor   │    │   Store    │   │
│  └──────────────┘    └──────────────┘    └────────────┘   │
│                                                            │
└────────────────────────────────────────────────────────────┘

Graph Schema:

  ┌─────────┐         ┌──────────┐        ┌─────────┐
  │  File   │─IMPORTS→│   File   │─DEFINES→│Function │
  └─────────┘         └──────────┘        └────┬────┘
                                               │CALLS
                                               ▼
  ┌─────────┐         ┌──────────┐        ┌─────────┐
  │ Table   │◀MODIFIES─│ Function │─CALLS─→│Function │
  └─────────┘         └──────────┘        └─────────┘
```

### Internal Components

#### 1. Codebase Parser

Parses code to extract entities and relationships:

```python
class CodebaseParser:
    """Parses codebase to extract graph nodes and edges."""
    
    def __init__(self, languages: list[str]):
        self.parsers = {
            lang: self._init_parser(lang)
            for lang in languages
        }
    
    def _init_parser(self, language: str):
        """Initialize TreeSitter parser for language."""
        import tree_sitter
        
        language_lib = tree_sitter.Language(
            f'build/{language}.so',
            language
        )
        
        parser = tree_sitter.Parser()
        parser.set_language(language_lib)
        
        return parser
    
    async def parse_file(self, file_path: Path) -> FileParseResult:
        """
        Parse file and extract entities and relationships.
        
        Returns:
        - Nodes: functions, classes, variables
        - Edges: calls, imports, inherits
        """
        
        language = self._detect_language(file_path)
        parser = self.parsers.get(language)
        
        if not parser:
            return FileParseResult.empty()
        
        # Read and parse file
        content = file_path.read_text()
        tree = parser.parse(bytes(content, "utf8"))
        
        # Extract entities
        nodes = []
        edges = []
        
        # Extract functions
        functions = self._extract_functions(tree, file_path)
        nodes.extend(functions)
        
        # Extract classes
        classes = self._extract_classes(tree, file_path)
        nodes.extend(classes)
        
        # Extract imports
        imports = self._extract_imports(tree, file_path)
        edges.extend(imports)
        
        # Extract function calls
        calls = self._extract_calls(tree, file_path)
        edges.extend(calls)
        
        return FileParseResult(nodes=nodes, edges=edges)
    
    def _extract_functions(
        self,
        tree: tree_sitter.Tree,
        file_path: Path
    ) -> list[FunctionNode]:
        """Extract function definitions."""
        
        functions = []
        
        # Query for function definitions
        query = """
        (function_declaration
          name: (identifier) @func_name
          parameters: (formal_parameters) @params
        ) @function
        """
        
        matches = self._query_tree(tree, query)
        
        for match in matches:
            func_node = match["function"]
            name = match["func_name"].text.decode()
            
            functions.append(FunctionNode(
                name=name,
                file=str(file_path),
                signature=self._extract_signature(match),
                line_start=func_node.start_point[0],
                line_end=func_node.end_point[0]
            ))
        
        return functions
    
    def _extract_calls(
        self,
        tree: tree_sitter.Tree,
        file_path: Path
    ) -> list[CallsRelation]:
        """Extract function calls."""
        
        calls = []
        
        # Query for call expressions
        query = """
        (call_expression
          function: (identifier) @callee
        ) @call
        """
        
        matches = self._query_tree(tree, query)
        
        # Track which function contains this call
        current_function = None  # Determined by context
        
        for match in matches:
            callee = match["callee"].text.decode()
            
            if current_function:
                calls.append(CallsRelation(
                    from_func=current_function,
                    to_func=callee,
                    call_count=1
                ))
        
        return calls
```

#### 2. Graph Builder

Constructs and maintains graph:

```python
class GraphBuilder:
    """Builds and maintains codebase knowledge graph."""
    
    def __init__(self, graph_store: GraphStore):
        self.graph_store = graph_store
        self.parser = CodebaseParser(["typescript", "python", "rust"])
    
    async def build_graph(self, codebase_paths: list[Path]) -> GraphStats:
        """Build complete graph from codebase."""
        
        stats = GraphStats()
        
        # Find all source files
        files = []
        for path in codebase_paths:
            files.extend(self._find_source_files(path))
        
        stats.files_found = len(files)
        
        # Parse all files in parallel
        parse_tasks = [
            self.parser.parse_file(file)
            for file in files
        ]
        
        results = await asyncio.gather(*parse_tasks)
        
        # Build graph
        for file, result in zip(files, results):
            # Add file node
            await self.graph_store.add_node(FileNode(
                path=str(file),
                language=self._detect_language(file),
                lines_of_code=self._count_lines(file),
                last_modified=file.stat().st_mtime
            ))
            
            # Add entity nodes
            for node in result.nodes:
                await self.graph_store.add_node(node)
                stats.nodes_added += 1
            
            # Add relationships
            for edge in result.edges:
                await self.graph_store.add_edge(edge)
                stats.edges_added += 1
        
        return stats
    
    async def update_file(self, file_path: Path):
        """Incrementally update graph for changed file."""
        
        # Remove existing nodes/edges for this file
        await self.graph_store.remove_nodes_by_file(str(file_path))
        
        # Re-parse file
        result = await self.parser.parse_file(file_path)
        
        # Add updated nodes/edges
        for node in result.nodes:
            await self.graph_store.add_node(node)
        
        for edge in result.edges:
            await self.graph_store.add_edge(edge)
```

#### 3. Graph Store

Persists and queries graph:

```python
class GraphStore:
    """Graph database abstraction."""
    
    def __init__(self, backend: str = "neo4j"):
        self.backend = backend
        self.conn = self._connect(backend)
    
    async def add_node(self, node: Node):
        """Add node to graph."""
        
        if self.backend == "neo4j":
            await self._neo4j_add_node(node)
        elif self.backend == "sqlite":
            await self._sqlite_add_node(node)
    
    async def add_edge(self, edge: Relation):
        """Add edge to graph."""
        # Implementation depends on backend
        ...
    
    async def query(
        self,
        cypher_query: str,
        params: dict | None = None
    ) -> list[dict]:
        """Execute Cypher query (or SQL equivalent)."""
        
        if self.backend == "neo4j":
            result = await self.conn.execute_read(
                lambda tx: tx.run(cypher_query, params or {})
            )
            return [record.data() for record in result]
        
        # Translate to SQL for sqlite backend
        ...
    
    # Common query patterns
    
    async def find_dependencies(self, entity: str) -> list[str]:
        """Find all dependencies of entity."""
        
        query = """
        MATCH (e {name: $entity})-[r:IMPORTS|CALLS|INHERITS]->(dep)
        RETURN dep.name as dependency
        """
        
        results = await self.query(query, {"entity": entity})
        return [r["dependency"] for r in results]
    
    async def find_dependents(self, entity: str) -> list[str]:
        """Find all entities that depend on this entity."""
        
        query = """
        MATCH (dependent)-[r:IMPORTS|CALLS|INHERITS]->(e {name: $entity})
        RETURN dependent.name as dependent
        """
        
        results = await self.query(query, {"entity": entity})
        return [r["dependent"] for r in results]
    
    async def find_call_paths(
        self,
        from_func: str,
        to_func: str,
        max_depth: int = 5
    ) -> list[list[str]]:
        """Find call paths between functions."""
        
        query = """
        MATCH path = (start:Function {name: $from})-[:CALLS*1..${max_depth}]->(end:Function {name: $to})
        RETURN [node in nodes(path) | node.name] as path
        LIMIT 10
        """
        
        results = await self.query(query, {
            "from": from_func,
            "to": to_func,
            "max_depth": max_depth
        })
        
        return [r["path"] for r in results]
    
    async def calculate_impact(
        self,
        changed_files: list[str],
        max_depth: int = 5
    ) -> dict[int, list[str]]:
        """Calculate impact radius (BFS from changed files)."""
        
        query = """
        MATCH (changed:File)
        WHERE changed.path IN $changed_files
        CALL {
          WITH changed
          MATCH path = (changed)<-[:IMPORTS*1..${max_depth}]-(dependent:File)
          RETURN dependent.path as file, length(path) as depth
        }
        RETURN depth, collect(DISTINCT file) as files
        ORDER BY depth
        """
        
        results = await self.query(query, {
            "changed_files": changed_files,
            "max_depth": max_depth
        })
        
        return {r["depth"]: r["files"] for r in results}
```

#### 4. Subgraph Extractor

Extracts relevant subgraph for context:

```python
class SubgraphExtractor:
    """Extracts relevant subgraph for conversation context."""
    
    def __init__(self, graph_store: GraphStore):
        self.graph_store = graph_store
    
    async def extract_relevant_subgraph(
        self,
        entities: list[str],
        max_depth: int = 2,
        max_nodes: int = 50
    ) -> Subgraph:
        """
        Extract subgraph relevant to entities.
        
        Strategy:
        1. Start from mentioned entities
        2. Expand outward (BFS) up to max_depth
        3. Stop at max_nodes
        4. Format for context injection
        """
        
        subgraph = Subgraph(nodes=[], edges=[])
        visited = set()
        queue = [(entity, 0) for entity in entities]
        
        while queue and len(subgraph.nodes) < max_nodes:
            entity, depth = queue.pop(0)
            
            if entity in visited or depth > max_depth:
                continue
            
            visited.add(entity)
            
            # Get node
            node = await self.graph_store.get_node(entity)
            if node:
                subgraph.nodes.append(node)
            
            # Get edges
            edges = await self.graph_store.get_edges(entity)
            subgraph.edges.extend(edges)
            
            # Expand to neighbors (if within depth)
            if depth < max_depth:
                neighbors = await self.graph_store.get_neighbors(entity)
                queue.extend((n, depth + 1) for n in neighbors)
        
        return subgraph
    
    def format_for_context(self, subgraph: Subgraph) -> str:
        """Format subgraph for injection into context."""
        
        sections = []
        
        # Files
        files = [n for n in subgraph.nodes if isinstance(n, FileNode)]
        if files:
            sections.append("# Files")
            for file in files:
                sections.append(f"- {file.path} ({file.lines_of_code} LOC)")
        
        # Functions
        functions = [n for n in subgraph.nodes if isinstance(n, FunctionNode)]
        if functions:
            sections.append("\n# Functions")
            for func in functions:
                sections.append(f"- {func.name} in {func.file}:{func.line_start}")
        
        # Relationships
        sections.append("\n# Relationships")
        for edge in subgraph.edges[:20]:  # Limit to 20 edges
            sections.append(f"- {edge.from_entity} {edge.relation_type} {edge.to_entity}")
        
        return "\n".join(sections)
```

### Data Flow

```
File Change Detected
    │
    ▼
┌────────────────────┐
│ Re-parse File      │
│ - Extract entities │
│ - Extract relations│
└──────┬─────────────┘
       │
       ▼
┌────────────────────┐
│ Update Graph       │
│ - Remove old nodes │
│ - Add new nodes    │
│ - Update edges     │
└──────┬─────────────┘
       │
       ▼
   Graph Updated

Conversation Message:
"Refactor auth module"
    │
    ▼
┌────────────────────┐
│ Detect Entity      │
│ "auth module"      │
└──────┬─────────────┘
       │
       ▼
┌────────────────────┐
│ Query Graph        │
│ - Find dependencies│
│ - Calculate impact │
└──────┬─────────────┘
       │
       ▼
┌────────────────────┐
│ Extract Subgraph   │
│ - auth + deps      │
│ - Max 50 nodes     │
└──────┬─────────────┘
       │
       ▼
┌────────────────────┐
│ Inject into Context│
│ (as system message)│
└────────────────────┘
```

---

## Examples

### Example 1: Dependency Analysis

```yaml
# User asks
"What depends on the User model?"

# Graph query
find_dependents("User")

# Results
Dependencies on User model:
- src/api/users.py: Imports User model
  - createUser() reads User
  - updateUser() modifies User
- src/api/auth.py: Imports User model
  - authenticate() reads User
- src/services/email.py: Imports User
  - sendWelcomeEmail() reads User.email
- tests/test_users.py: Imports User (test code)

Direct dependents: 4 files
Indirect dependents: 12 files (via imports)
Impact: Medium

# Injected into context
---
Codebase Context: User Model Dependencies

## Direct Dependencies (4)
- src/api/users.py (IMPORTS User, MODIFIES users table)
- src/api/auth.py (IMPORTS User, READS users table)
- src/services/email.py (IMPORTS User)
- tests/test_users.py (IMPORTS User)

## Call Chains
- User.save() ← createUser() ← POST /api/users
- User.authenticate() ← authenticate() ← POST /api/auth/login

## Impact Estimate
Changing User model will affect:
- 4 files directly
- 12 files indirectly
- Risk: MEDIUM (auth flow impacted)
---
```

### Example 2: Impact Analysis for Refactoring

```yaml
# User asks
"I want to refactor the authentication module. What will break?"

# Graph queries
changed_files = ["src/auth/middleware.py", "src/auth/tokens.py"]
impact = calculate_impact_radius(changed_files)

# Results
Impact Analysis: Authentication Module Refactor

## Direct Impact (Depth 1) - 4 files
Files that directly import auth modules:
- src/api/routes/users.py (imports auth/middleware)
- src/api/routes/admin.py (imports auth/middleware)
- src/services/email.py (imports auth/tokens)
- src/background/jobs/cleanup.py (imports auth/sessions)

## Indirect Impact (Depth 2) - 14 files
Files that depend on direct dependents:
- src/api/routes/posts.py → users.py → auth
- src/api/routes/comments.py → users.py → auth
- [12 more files...]

## Database Impact
Tables modified by auth modules:
- users table (auth/sessions.py MODIFIES)
- refresh_tokens table (auth/tokens.py MODIFIES)
- Migration 003_add_auth.sql needs review

## Call Chains
Critical paths through auth:
1. POST /api/users → users.create() → auth.hashPassword() → bcrypt.hash()
2. POST /api/login → auth.authenticate() → auth.verifyPassword()
3. Middleware: ALL protected routes → auth.middleware.verify()

## Risk Assessment
- Files affected: 18 total
- Critical paths: 3 (all authentication flows)
- Database changes: 2 tables
- **Estimated Risk: HIGH** (auth is central, widely used)

## Recommendations
1. Update 4 direct imports (check API contracts)
2. Run test suite: pytest tests/auth/**
3. Check for breaking changes in middleware signature
4. Review migration 003_add_auth for schema dependencies
5. Consider feature flag for gradual rollout

# Injected subgraph into context
---
Codebase Context: Authentication Module Structure

[Full graph with 50 most relevant nodes showing structure]
---
```

### Example 3: Call Path Tracing

```yaml
# User asks
"How does a user login request get to the database?"

# Graph query
trace_call_chain("POST /api/login", "database.execute")

# Results
Call Path: Login Request → Database

Path 1 (most common):
POST /api/login → routes/auth.login()
                 → auth.authenticate()
                 → User.findByEmail()
                 → database.query()
                 → database.execute()

Path 2 (with caching):
POST /api/login → routes/auth.login()
                 → auth.authenticate()
                 → cache.get() [cache miss]
                 → User.findByEmail()
                 → database.query()
                 → database.execute()

## Data Flow
1. Request: { email, password }
2. Validation: auth.validateCredentials()
3. Query: SELECT * FROM users WHERE email = ?
4. Password Check: bcrypt.compare()
5. Token Generation: auth.generateToken()
6. Response: { token, user }

## Tables Accessed
- users (READ: email, password_hash)
- refresh_tokens (WRITE: new token)
```

---

## Configuration

```yaml
# Full configuration example
contexts:
  - module: context-codebase-knowledge-graph
    config:
      # Source paths
      codebase_paths:
        - src/
        - lib/
      exclude_paths:
        - node_modules/
        - dist/
        - .git/
        - "**/*.test.ts"
      languages:
        - typescript
        - python
        - rust
      
      # Graph storage
      graph_database: neo4j://localhost:7687  # or sqlite, memory
      persistence: true
      incremental_updates: true
      
      # Indexing
      index_on_startup: false
      watch_for_changes: true
      debounce_ms: 1000
      
      # Node types
      track_files: true
      track_functions: true
      track_classes: true
      track_variables: true
      track_tables: true
      
      # Relationships
      track_imports_relations: true
      track_calls_relations: true
      track_inherits_relations: true
      track_modifies_relations: true
      track_reads_relations: true
      
      # Context injection
      auto_inject_graph: true
      max_depth: 3
      max_nodes: 50
      injection_format: markdown
      
      # Performance
      cache_queries: true
      cache_ttl_seconds: 300
      parallel_parsing: true
      max_parse_workers: 4
```

---

## Security Considerations

### Data Access
- **Codebase access**: Reads all source files
- **Graph data**: Complete codebase structure exposed
- **Sensitive paths**: May reveal internal architecture

### Resource Usage
- **Graph construction**: CPU/memory intensive
- **Graph storage**: Disk space for persistence
- **Query performance**: Complex graph queries may be slow

```python
REQUIRED_CAPABILITIES = [
    "context:read",
    "context:write",
    "filesystem:read",              # Read codebase
    "filesystem:watch",             # Watch for changes
]
```

---

## Testing Strategy

### Unit Tests

```python
def test_parser_extracts_functions():
    parser = CodebaseParser(["python"])
    result = parser.parse_file(Path("test.py"))
    assert len(result.nodes) > 0
    assert any(isinstance(n, FunctionNode) for n in result.nodes)

def test_graph_store_finds_dependencies():
    store = GraphStore("memory")
    # Add nodes and edges
    deps = store.find_dependencies("User")
    assert "auth.py" in deps
```

### Integration Tests

```python
async def test_full_graph_construction():
    context = CodebaseKnowledgeGraphContext(config)
    
    # Build graph
    await context.build_graph()
    
    # Query
    deps = await context.find_dependencies("User")
    assert len(deps) > 0
    
    # Calculate impact
    impact = await context.calculate_impact_radius(["src/user.py"])
    assert impact.directly_affected > 0
```

---

## Dependencies

### Required
- `tree-sitter` + language grammars - Code parsing
- `neo4j` or `sqlite3` - Graph storage

### Optional
- `watchdog` - File watching
- `networkx` - Graph algorithms (if not using Neo4j)

---

## Open Questions

1. **Graph granularity**: Track variable-level dependencies or just function-level?
2. **Cross-language support**: How to handle calls between languages (Python → TypeScript)?
3. **Database schema tracking**: Extract schema from migrations vs runtime inspection?
4. **Performance at scale**: How to handle codebases with 100K+ files?
5. **Semantic understanding**: Should we use LLM to understand code semantics beyond syntax?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
