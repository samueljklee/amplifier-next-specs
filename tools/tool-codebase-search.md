# tool-codebase-search

> **Priority**: P0 (Foundation)
> **Status**: Draft
> **Module**: `amplifier-module-tool-codebase-search`

## Overview

Semantic and structural search across codebases, documentation, and linked systems. Goes beyond grep/ripgrep by understanding code structure, natural language queries, and cross-referencing multiple data sources.

### Value Proposition

| Without | With |
|---------|------|
| `grep -r "auth timeout"` → 500 results, mostly noise | "Where do we handle authentication timeouts?" → 3 relevant files with context |
| Manual cross-referencing between code, docs, tickets | Single query spans code + docs + tickets + Slack |
| Lost tribal knowledge | "Why was this implemented this way?" → Links to original PR, discussion, decision |

### Use Cases

1. **Code discovery**: "Find all API endpoints that require admin permissions"
2. **Impact analysis**: "What code paths touch the user's email address?"
3. **Historical context**: "Why does this function have this weird edge case?"
4. **Onboarding**: "How does the payment flow work?"
5. **Debugging**: "Where is this error message defined and when is it thrown?"

---

## Contract

### Tool Definition

```python
TOOL_DEFINITION = {
    "name": "codebase_search",
    "description": """
    Search across the codebase, documentation, and linked systems using natural language
    or structured queries. Returns relevant code snippets, files, and cross-references.

    Use for:
    - Finding code by description ("authentication handling")
    - Understanding code relationships ("what calls this function")
    - Historical context ("why was this changed")
    - Cross-system search (code + docs + tickets)
    """,
    "parameters": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "Natural language query or structured search"
            },
            "scope": {
                "type": "string",
                "enum": ["code", "docs", "tickets", "discussions", "all"],
                "default": "all",
                "description": "Limit search to specific sources"
            },
            "file_patterns": {
                "type": "array",
                "items": {"type": "string"},
                "description": "Glob patterns to filter files (e.g., ['*.py', 'src/**'])"
            },
            "search_type": {
                "type": "string",
                "enum": ["semantic", "structural", "literal", "hybrid"],
                "default": "hybrid",
                "description": "Search strategy"
            },
            "include_context": {
                "type": "boolean",
                "default": true,
                "description": "Include surrounding code context and cross-references"
            },
            "max_results": {
                "type": "integer",
                "default": 10,
                "description": "Maximum results to return"
            }
        },
        "required": ["query"]
    }
}
```

### Input Schema

```python
@dataclass
class CodebaseSearchInput:
    query: str                          # Natural language or structured query
    scope: Scope = Scope.ALL            # code | docs | tickets | discussions | all
    file_patterns: list[str] | None     # Glob patterns to filter
    search_type: SearchType = HYBRID    # semantic | structural | literal | hybrid
    include_context: bool = True        # Include surrounding context
    max_results: int = 10               # Max results
```

### Output Schema

```python
@dataclass
class SearchResult:
    source_type: str                    # "code" | "doc" | "ticket" | "discussion"
    location: str                       # File path or URL
    snippet: str                        # Matched content
    relevance_score: float              # 0.0 - 1.0
    context: ResultContext | None       # Surrounding context if requested
    cross_references: list[CrossRef]    # Related items

@dataclass
class ResultContext:
    before: str                         # Lines before match
    after: str                          # Lines after match
    file_summary: str                   # What this file does
    related_symbols: list[str]          # Functions/classes in same file

@dataclass
class CrossRef:
    ref_type: str                       # "calls" | "called_by" | "imports" | "ticket" | "pr" | "doc"
    target: str                         # Target location
    description: str                    # Human-readable description

@dataclass
class CodebaseSearchOutput:
    results: list[SearchResult]
    query_interpretation: str           # How the query was understood
    search_stats: SearchStats           # Performance metrics
    suggestions: list[str]              # Related queries to try
```

### Events Emitted

| Event | When | Data |
|-------|------|------|
| `tool:codebase_search:start` | Search begins | query, scope, search_type |
| `tool:codebase_search:index_hit` | Index lookup | index_type, hit_count |
| `tool:codebase_search:complete` | Search done | result_count, duration_ms |
| `tool:codebase_search:error` | Search failed | error_type, message |

---

## Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     tool-codebase-search                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │   Query     │───▶│   Search    │───▶│   Result    │         │
│  │  Analyzer   │    │  Executor   │    │  Ranker     │         │
│  └─────────────┘    └──────┬──────┘    └─────────────┘         │
│                            │                                    │
│         ┌──────────────────┼──────────────────┐                │
│         ▼                  ▼                  ▼                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │   Code      │    │    Doc      │    │  External   │         │
│  │   Index     │    │   Index     │    │  Connectors │         │
│  └─────────────┘    └─────────────┘    └─────────────┘         │
│         │                  │                  │                 │
│         ▼                  ▼                  ▼                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │  TreeSitter │    │  Markdown   │    │ JIRA/Slack  │         │
│  │  + Embeddings│   │  + Embeddings│   │    APIs     │         │
│  └─────────────┘    └─────────────┘    └─────────────┘         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Internal Components

#### 1. Query Analyzer

Interprets natural language queries into structured search operations:

```python
class QueryAnalyzer:
    """
    Transforms natural language into search operations.

    Examples:
    - "authentication timeout" → semantic search for concept
    - "function handle_auth" → structural search for symbol
    - "JIRA-123" → literal search + external lookup
    - "where do we validate emails" → hybrid (semantic + structural)
    """

    def analyze(self, query: str) -> AnalyzedQuery:
        # Detect query intent
        intent = self._classify_intent(query)  # discovery | impact | history | debug

        # Extract search terms
        terms = self._extract_terms(query)

        # Determine optimal search strategy
        strategy = self._select_strategy(intent, terms)

        return AnalyzedQuery(intent, terms, strategy)
```

#### 2. Code Index

Maintains searchable index of codebase:

```python
class CodeIndex:
    """
    Multi-layer code index:
    - Symbol index (TreeSitter): functions, classes, variables
    - Semantic index (embeddings): conceptual similarity
    - Literal index (trigram): exact text matching
    - Graph index: call relationships, imports
    """

    def __init__(self, root_path: Path, config: IndexConfig):
        self.symbol_index = SymbolIndex(root_path)      # TreeSitter-based
        self.semantic_index = SemanticIndex(root_path)  # Embedding-based
        self.literal_index = LiteralIndex(root_path)    # Trigram-based
        self.graph_index = GraphIndex(root_path)        # Relationship-based

    async def search(self, query: AnalyzedQuery) -> list[CodeMatch]:
        # Execute appropriate indexes based on query strategy
        results = []

        if query.strategy.use_semantic:
            results.extend(await self.semantic_index.search(query.terms))

        if query.strategy.use_structural:
            results.extend(await self.symbol_index.search(query.terms))

        if query.strategy.use_literal:
            results.extend(await self.literal_index.search(query.terms))

        # Enrich with relationships
        for result in results:
            result.relationships = await self.graph_index.get_relationships(result.symbol)

        return results
```

#### 3. External Connectors

Integrates with external systems:

```python
class ExternalConnectors:
    """
    Connectors for external data sources.
    Each connector implements SearchableSource interface.
    """

    def __init__(self, config: ConnectorsConfig):
        self.connectors = {}

        if config.jira:
            self.connectors['jira'] = JiraConnector(config.jira)
        if config.slack:
            self.connectors['slack'] = SlackConnector(config.slack)
        if config.confluence:
            self.connectors['confluence'] = ConfluenceConnector(config.confluence)
        if config.github:
            self.connectors['github'] = GitHubConnector(config.github)

    async def search(self, query: AnalyzedQuery, sources: list[str]) -> list[ExternalMatch]:
        tasks = [
            self.connectors[source].search(query)
            for source in sources
            if source in self.connectors
        ]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        return flatten_valid(results)
```

#### 4. Result Ranker

Combines and ranks results from all sources:

```python
class ResultRanker:
    """
    Ranks and deduplicates results across all sources.
    Uses multiple signals for relevance scoring.
    """

    def rank(self, results: list[SearchResult], query: AnalyzedQuery) -> list[SearchResult]:
        for result in results:
            result.relevance_score = self._compute_score(result, query)

        # Sort by relevance
        ranked = sorted(results, key=lambda r: r.relevance_score, reverse=True)

        # Deduplicate (same content from different indexes)
        deduped = self._deduplicate(ranked)

        # Add cross-references between results
        enriched = self._add_cross_references(deduped)

        return enriched

    def _compute_score(self, result: SearchResult, query: AnalyzedQuery) -> float:
        """
        Scoring factors:
        - Text similarity (semantic or literal match quality)
        - Recency (recently modified files ranked higher)
        - Centrality (highly connected code ranked higher)
        - Query-type match (structural query + function match = boost)
        """
        score = result.base_score
        score *= self._recency_factor(result)
        score *= self._centrality_factor(result)
        score *= self._query_type_factor(result, query)
        return score
```

### Data Flow

```
User Query
    │
    ▼
┌─────────────────┐
│ Query Analyzer  │ ── Classify intent, extract terms, select strategy
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Search Executor │ ── Fan out to appropriate indexes/connectors
└────────┬────────┘
         │
    ┌────┴────┬──────────┬──────────┐
    ▼         ▼          ▼          ▼
┌───────┐ ┌───────┐ ┌─────────┐ ┌────────┐
│ Code  │ │ Doc   │ │ JIRA    │ │ Slack  │
│ Index │ │ Index │ │Connector│ │Connector│
└───┬───┘ └───┬───┘ └────┬────┘ └───┬────┘
    │         │          │          │
    └────┬────┴──────────┴──────────┘
         │
         ▼
┌─────────────────┐
│  Result Ranker  │ ── Combine, rank, dedupe, enrich
└────────┬────────┘
         │
         ▼
    Final Results
```

---

## Configuration

```yaml
# Module configuration
tool-codebase-search:
  # Indexing configuration
  index:
    root_paths:
      - "."                           # Current repo
      - "../shared-libs"              # Shared libraries
    exclude_patterns:
      - "node_modules/**"
      - ".git/**"
      - "**/*.min.js"

    # Index types to build
    indexes:
      symbol: true                    # TreeSitter symbol index
      semantic: true                  # Embedding-based semantic index
      literal: true                   # Trigram literal index
      graph: true                     # Call graph index

    # Embedding configuration (for semantic search)
    embeddings:
      model: "text-embedding-3-small" # OpenAI embedding model
      chunk_size: 500                 # Tokens per chunk
      chunk_overlap: 50               # Overlap between chunks

    # Incremental update settings
    update:
      watch: true                     # Watch for file changes
      debounce_ms: 1000               # Debounce rapid changes

  # External connectors
  connectors:
    jira:
      enabled: true
      base_url: "${JIRA_URL}"
      auth_token: "${JIRA_TOKEN}"
      project_keys: ["PROJ", "PLATFORM"]

    slack:
      enabled: true
      auth_token: "${SLACK_TOKEN}"
      channels: ["#engineering", "#incidents"]

    github:
      enabled: true
      auth_token: "${GITHUB_TOKEN}"
      include_prs: true
      include_issues: true
      include_discussions: true

  # Search behavior
  search:
    default_scope: "all"
    default_max_results: 10
    context_lines_before: 5
    context_lines_after: 5

    # Ranking weights
    ranking:
      recency_weight: 0.2
      centrality_weight: 0.3
      match_quality_weight: 0.5

    # Performance limits
    timeout_ms: 10000
    max_files_to_scan: 10000

  # Caching
  cache:
    enabled: true
    ttl_seconds: 300                  # Cache results for 5 minutes
    max_entries: 1000
```

---

## Examples

### Example 1: Natural Language Code Discovery

```python
# Input
{
    "query": "Where do we validate user email addresses?",
    "scope": "code"
}

# Output
{
    "results": [
        {
            "source_type": "code",
            "location": "src/validators/email.py:42",
            "snippet": "def validate_email(email: str) -> ValidationResult:\n    \"\"\"Validates email format and domain.\"\"\"\n    ...",
            "relevance_score": 0.95,
            "context": {
                "file_summary": "Email validation utilities",
                "related_symbols": ["validate_email", "EmailValidator", "VALID_DOMAINS"]
            },
            "cross_references": [
                {"ref_type": "called_by", "target": "src/api/users.py:create_user", "description": "Called during user registration"},
                {"ref_type": "called_by", "target": "src/api/settings.py:update_email", "description": "Called when updating email"}
            ]
        },
        {
            "source_type": "code",
            "location": "src/models/user.py:78",
            "snippet": "@validator('email')\ndef email_must_be_valid(cls, v):\n    ...",
            "relevance_score": 0.88,
            "context": {...},
            "cross_references": [...]
        }
    ],
    "query_interpretation": "Searching for email validation logic in code",
    "suggestions": ["Where do we send verification emails?", "Email validation rules"]
}
```

### Example 2: Historical Context Search

```python
# Input
{
    "query": "Why was retry logic added to payment processing?",
    "scope": "all",
    "search_type": "semantic"
}

# Output
{
    "results": [
        {
            "source_type": "ticket",
            "location": "JIRA-4523",
            "snippet": "Payment failures during peak hours due to gateway timeouts. Adding retry with exponential backoff.",
            "relevance_score": 0.92,
            "cross_references": [
                {"ref_type": "pr", "target": "PR #892", "description": "Implementation PR"},
                {"ref_type": "doc", "target": "docs/payments/reliability.md", "description": "Documentation"}
            ]
        },
        {
            "source_type": "discussion",
            "location": "slack://engineering/p1699234567",
            "snippet": "We're seeing 5% payment failures during sales events. Gateway is timing out under load...",
            "relevance_score": 0.85
        },
        {
            "source_type": "code",
            "location": "src/payments/gateway.py:156",
            "snippet": "# Added retry logic per JIRA-4523\n@retry(max_attempts=3, backoff=exponential)\nasync def process_payment(...):",
            "relevance_score": 0.78
        }
    ],
    "query_interpretation": "Searching for historical context about payment retry implementation"
}
```

### Example 3: Impact Analysis

```python
# Input
{
    "query": "What code uses the User.email field?",
    "search_type": "structural"
}

# Output
{
    "results": [
        {
            "source_type": "code",
            "location": "src/models/user.py:23",
            "snippet": "email: EmailStr = Field(...)",
            "relevance_score": 1.0,
            "cross_references": [
                {"ref_type": "read_by", "target": "src/api/users.py:45", "description": "User profile endpoint"},
                {"ref_type": "read_by", "target": "src/notifications/email.py:12", "description": "Email sender"},
                {"ref_type": "written_by", "target": "src/api/users.py:89", "description": "Update email endpoint"},
                {"ref_type": "validated_by", "target": "src/validators/email.py:42", "description": "Email validator"}
            ]
        }
    ],
    "query_interpretation": "Structural analysis of User.email field usage",
    "search_stats": {
        "files_analyzed": 234,
        "references_found": 47,
        "duration_ms": 156
    }
}
```

---

## Security Considerations

### Data Access

- **Code access**: Reads files within configured root paths only
- **External systems**: Uses configured credentials, respects API permissions
- **Cached data**: Cached in memory/disk per configuration, cleared on session end

### Sensitive Data

- **Query logging**: Queries logged to session events (may contain sensitive terms)
- **Results**: May contain sensitive code snippets
- **Credentials**: External connector credentials stored securely, never logged

### Permissions Model

```python
REQUIRED_CAPABILITIES = [
    "filesystem:read",      # Read codebase files
    "network:external",     # Connect to external APIs (if connectors enabled)
]

# Optional capabilities based on connectors
OPTIONAL_CAPABILITIES = {
    "jira": "network:jira",
    "slack": "network:slack",
    "github": "network:github",
}
```

### Risk Mitigations

| Risk | Mitigation |
|------|------------|
| Exposing sensitive code in results | Results filtered by session's file access permissions |
| Credential leakage | Credentials never included in results or logs |
| Excessive resource use | Configurable timeouts and file limits |
| Stale cache serving wrong data | Cache keyed by query + file hashes, invalidated on changes |

---

## Testing Strategy

### Unit Tests

```python
# Query analyzer tests
def test_query_analyzer_detects_semantic_intent():
    analyzer = QueryAnalyzer()
    result = analyzer.analyze("where do we handle authentication")
    assert result.intent == QueryIntent.DISCOVERY
    assert result.strategy.use_semantic == True

def test_query_analyzer_detects_structural_intent():
    analyzer = QueryAnalyzer()
    result = analyzer.analyze("function validate_email")
    assert result.intent == QueryIntent.SYMBOL_LOOKUP
    assert result.strategy.use_structural == True

# Result ranking tests
def test_ranker_prioritizes_recent_files():
    ...

def test_ranker_deduplicates_same_content():
    ...
```

### Integration Tests

```python
# Full search flow tests
async def test_semantic_search_finds_relevant_code():
    tool = CodebaseSearchTool(test_config)
    await tool.build_index(TEST_REPO_PATH)

    result = await tool.execute({
        "query": "email validation",
        "scope": "code"
    })

    assert len(result.results) > 0
    assert "validate" in result.results[0].snippet.lower()
    assert "email" in result.results[0].snippet.lower()

async def test_external_connector_integration():
    # Requires mock JIRA/Slack servers
    ...
```

### Performance Tests

```python
def test_search_performance_under_load():
    """Search should complete within timeout on large codebase."""
    # 100k file synthetic codebase
    result = tool.execute({"query": "authentication"})
    assert result.search_stats.duration_ms < 10000

def test_index_incremental_update_performance():
    """Index update should be fast for single file changes."""
    ...
```

---

## Implementation Notes

### Index Storage

- Symbol index: SQLite database with FTS5 for text search
- Semantic index: Vector database (could use Chroma, FAISS, or SQLite with vector extension)
- Graph index: SQLite with adjacency list tables
- All indexes stored in `.amplifier/indexes/` directory

### Incremental Updates

- File watcher monitors for changes
- On change: re-index only affected files
- Graph updates propagate to dependent files
- Embedding regeneration batched for efficiency

### Embedding Strategy

- Code chunked by logical units (functions, classes) when possible
- Fall back to token-based chunking for long files
- Metadata (file path, language, symbols) included in embedding context

---

## Dependencies

### Required

- `tree-sitter` + language grammars - Code parsing
- `tiktoken` - Token counting for chunking
- `aiosqlite` - Async SQLite for indexes

### Optional (based on configuration)

- `openai` - For embeddings (or alternative embedding provider)
- `chromadb` or `faiss` - For vector search (alternative to SQLite vectors)
- `jira` - JIRA connector
- `slack-sdk` - Slack connector
- `PyGithub` - GitHub connector

---

## Open Questions

1. **Embedding provider**: Should we support multiple embedding providers or standardize on one?
   - Option A: OpenAI embeddings only (simpler, consistent quality)
   - Option B: Pluggable providers (flexibility, cost control)

2. **Index persistence**: Where should indexes live?
   - Option A: Project-local (`.amplifier/indexes/`) - isolated but duplicated
   - Option B: User-level cache (`~/.amplifier/indexes/`) - shared but complex invalidation

3. **Real-time vs batch indexing**: Should we support real-time indexing for very large codebases?
   - Option A: Always real-time (simpler UX, may be slow)
   - Option B: Background batch with staleness indicator

4. **Cross-repo search**: Should a single search span multiple repositories?
   - Option A: Single repo per search (simpler)
   - Option B: Multi-repo with repo prefixes (more powerful)

5. **Result format**: How much context to include by default?
   - Trade-off: More context = more useful but more tokens consumed

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
