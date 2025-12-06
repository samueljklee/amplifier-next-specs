# context-decision-journal

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-context-decision-journal`

## Overview

A context manager that automatically detects, extracts, and indexes architectural and product decisions from conversations. Maintains a queryable decision log with full context, rationale, alternatives considered, and temporal tracking of how decisions evolve or get revisited.

### Value Proposition

| Without | With |
|---------|------|
| "Why did we choose X?" lost in history | Complete decision record with rationale |
| Re-litigating old decisions | Quick lookup: "We decided this because..." |
| New team members missing context | Decision journal as onboarding material |
| Inconsistent decision-making | Learn from past decisions |

### Use Cases

1. **Onboarding**: New engineers/PMs quickly understand past architectural decisions
2. **Decision review**: "Why did we choose Postgres over MongoDB?" - instant answer
3. **Architecture evolution**: Track how decisions changed over time
4. **Pattern recognition**: Identify decision patterns and principles
5. **Compliance/audit**: Maintain decision trail for governance

---

## Contract

### ContextManager Interface

```python
class DecisionJournalContext:
    """
    Context manager with decision detection and indexing.
    
    Extends standard context manager with:
    - Automatic decision detection in conversations
    - Structured decision extraction
    - Queryable decision index
    - Temporal decision tracking
    """
    
    async def add_message(
        self, 
        message: dict[str, Any],
    ) -> None:
        """
        Add message and detect decisions.
        
        If decision detected:
        1. Extract decision metadata
        2. Store in decision index
        3. Link to related code/features
        """
    
    async def get_messages(self) -> list[dict[str, Any]]:
        """Get conversation messages (standard behavior)."""
    
    async def search_decisions(
        self,
        query: str,
        filters: DecisionFilters | None = None
    ) -> list[Decision]:
        """
        Search decision journal.
        
        Args:
            query: Text search query
            filters: Optional filters (by tech, date, stakeholder)
            
        Returns:
            Matching decisions with full context
        """
    
    async def get_decision_chain(
        self,
        decision_id: str
    ) -> DecisionChain:
        """
        Get evolution of a decision.
        
        Returns:
        - Original decision
        - Revisions/updates
        - Superseded decisions
        """
```

### Configuration Schema

```toml
[[contexts]]
module = "context-decision-journal"
config = {
  # Decision detection
  auto_detect = true
  detection_keywords = [
    "decided", "decision:", "let's go with", "we should",
    "agreed", "consensus", "choosing", "selected"
  ]
  detection_threshold = 0.7                 # Confidence threshold for LLM detection
  
  # Extraction
  structured_extraction = true              # Extract metadata (what, why, alternatives)
  extract_rationale = true
  extract_alternatives = true
  extract_trade_offs = true
  
  # Linking
  link_to_code = true                       # Link decisions to code files
  link_to_features = true                   # Link to features/epics
  auto_link_strategy = "file_mention"       # "file_mention" | "git_context" | "llm_inference"
  
  # Indexing
  search_index = "sqlite"                   # "memory" | "sqlite" | "elasticsearch"
  index_embeddings = true                   # Enable semantic search
  embedding_model = "text-embedding-3-small"
  
  # Storage
  persistence = true                        # Persist decisions across sessions
  storage_path = ".amplifier/decisions/"
  retention_days = 730                      # 2 years default
  
  # Querying
  enable_temporal_queries = true            # "decisions made in Q2 2024"
  enable_stakeholder_queries = true         # "decisions by @alice"
  enable_technology_queries = true          # "decisions about databases"
}
```

### Data Schema

```python
@dataclass
class Decision:
    """Structured decision record."""
    id: str                                  # UUID
    timestamp: datetime
    session_id: str
    
    # Core decision
    decision: str                            # What was decided
    context: str                             # Why was it being discussed
    rationale: str                           # Why this choice
    
    # Alternatives
    alternatives: list[Alternative]          # Other options considered
    trade_offs: TradeOffs | None            # Pros/cons analysis
    
    # Stakeholders
    stakeholders: list[str]                  # Who was involved
    decision_maker: str | None               # Final decision maker
    
    # Links
    related_code: list[str]                  # File paths affected
    related_features: list[str]              # Features/epics
    related_docs: list[str]                  # Documentation
    
    # Status
    status: DecisionStatus                   # active | superseded | revisited
    superseded_by: str | None                # Decision ID if replaced
    supersedes: str | None                   # Decision ID if this replaces another
    
    # Search
    tags: list[str]                          # ["database", "architecture"]
    keywords: list[str]                      # Extracted key terms

@dataclass
class Alternative:
    """Alternative option that was considered."""
    option: str
    pros: list[str]
    cons: list[str]
    why_not_chosen: str | None

@dataclass
class TradeOffs:
    """Structured pros/cons."""
    pros: list[str]
    cons: list[str]
    risks: list[str]
    assumptions: list[str]

@dataclass
class DecisionChain:
    """Evolution of a decision over time."""
    original: Decision
    revisions: list[Decision]
    current: Decision
    
enum DecisionStatus:
    ACTIVE = "active"                        # Current decision
    SUPERSEDED = "superseded"                # Replaced by newer decision
    REVISITED = "revisited"                  # Reconsidered but kept
```

### Events Emitted

| Event | When | Data |
|-------|------|------|
| `context:decision:detected` | Decision detected | decision_id, confidence |
| `context:decision:extracted` | Metadata extracted | decision_id, decision_summary |
| `context:decision:indexed` | Added to index | decision_id |
| `context:decision:superseded` | Decision replaced | old_decision_id, new_decision_id |
| `context:decision:searched` | Search performed | query, result_count |

---

## Architecture

### Component Diagram

```
┌──────────────────────────────────────────────────────────────┐
│              context-decision-journal                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐    ┌──────────────┐    ┌───────────────┐   │
│  │  Decision   │───▶│  Metadata    │───▶│    Decision   │   │
│  │  Detector   │    │  Extractor   │    │     Index     │   │
│  └─────────────┘    └──────────────┘    └───────┬───────┘   │
│                                                  │           │
│                                                  │           │
│  ┌─────────────┐    ┌──────────────┐    ┌───────▼───────┐   │
│  │   Code      │    │   Temporal   │    │    Search     │   │
│  │   Linker    │    │   Tracker    │    │    Engine     │   │
│  └─────────────┘    └──────────────┘    └───────────────┘   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Internal Components

#### 1. Decision Detector

Identifies decision points in conversations:

```python
class DecisionDetector:
    """Detects when decisions are made in conversations."""
    
    DECISION_KEYWORDS = [
        "decided", "decision:", "let's go with", "we should",
        "agreed", "consensus", "choosing", "selected", "approved",
        "going with", "will use", "settled on"
    ]
    
    def __init__(self, threshold: float = 0.7):
        self.threshold = threshold
    
    async def detect(
        self,
        message: dict[str, Any],
        conversation_context: list[dict[str, Any]]
    ) -> DetectionResult:
        """
        Detect if message contains a decision.
        
        Detection strategy:
        1. Keyword matching (fast, high recall)
        2. LLM confirmation (slower, high precision)
        3. Context analysis (is this really a decision?)
        """
        content = message.get("content", "")
        
        # Fast keyword check
        has_keyword = any(
            keyword in content.lower()
            for keyword in self.DECISION_KEYWORDS
        )
        
        if not has_keyword:
            return DetectionResult(is_decision=False, confidence=0.0)
        
        # LLM confirmation
        confidence = await self._llm_confirm(message, conversation_context)
        
        return DetectionResult(
            is_decision=confidence >= self.threshold,
            confidence=confidence,
            decision_snippet=self._extract_snippet(content)
        )
    
    async def _llm_confirm(
        self,
        message: dict[str, Any],
        context: list[dict[str, Any]]
    ) -> float:
        """Use LLM to confirm this is a decision."""
        
        context_str = self._format_context(context[-5:])  # Last 5 messages
        
        prompt = f"""
Analyze if this message contains a finalized decision:

Recent context:
{context_str}

Current message:
{message['content']}

Is this a decision being made (not just a proposal or discussion)?

Answer with a confidence score (0.0-1.0) and brief explanation:
- 1.0: Definitely a decision
- 0.5: Possibly a decision
- 0.0: Not a decision

Format: score|explanation
"""
        
        # Parse LLM response
        response = await self._call_llm(prompt)
        score, explanation = self._parse_response(response)
        
        return score
```

#### 2. Metadata Extractor

Extracts structured decision data:

```python
class MetadataExtractor:
    """Extracts structured metadata from decision conversations."""
    
    async def extract(
        self,
        decision_message: dict[str, Any],
        conversation_context: list[dict[str, Any]]
    ) -> Decision:
        """
        Extract structured decision data.
        
        Extracts:
        - What was decided
        - Why (rationale)
        - What alternatives were considered
        - Who decided
        - What it relates to
        """
        
        # Build extraction prompt with conversation context
        prompt = self._build_extraction_prompt(
            decision_message,
            conversation_context
        )
        
        # Get structured extraction from LLM
        raw_extraction = await self._call_llm(prompt)
        
        # Parse into Decision object
        decision = self._parse_extraction(raw_extraction)
        
        # Enrich with additional metadata
        decision = await self._enrich_decision(decision, conversation_context)
        
        return decision
    
    def _build_extraction_prompt(
        self,
        message: dict[str, Any],
        context: list[dict[str, Any]]
    ) -> str:
        """Build prompt for structured extraction."""
        
        context_str = self._format_context(context[-10:])
        
        return f"""
Extract structured decision information:

Conversation context:
{context_str}

Decision message:
{message['content']}

Extract the following in JSON format:
{{
  "decision": "What was decided (one sentence)",
  "context": "Why was this being discussed",
  "rationale": "Why was this choice made",
  "alternatives": [
    {{
      "option": "Alternative that was considered",
      "pros": ["advantage 1", "advantage 2"],
      "cons": ["disadvantage 1"],
      "why_not_chosen": "Reason not selected"
    }}
  ],
  "trade_offs": {{
    "pros": ["advantage of chosen option"],
    "cons": ["disadvantage of chosen option"],
    "risks": ["potential risks"],
    "assumptions": ["assumptions made"]
  }},
  "stakeholders": ["@person1", "@person2"],
  "decision_maker": "@person or null",
  "tags": ["technology", "domain"]
}}

Be specific and extract actual information from the conversation.
"""
    
    async def _enrich_decision(
        self,
        decision: Decision,
        context: list[dict[str, Any]]
    ) -> Decision:
        """Enrich decision with additional context."""
        
        # Extract stakeholders from conversation
        decision.stakeholders = self._extract_stakeholders(context)
        
        # Extract related code/files mentioned
        decision.related_code = self._extract_file_mentions(context)
        
        # Generate keywords for search
        decision.keywords = await self._generate_keywords(decision)
        
        return decision
    
    def _extract_file_mentions(
        self,
        context: list[dict[str, Any]]
    ) -> list[str]:
        """Extract file paths mentioned in conversation."""
        import re
        
        file_pattern = r'([a-zA-Z0-9_/.-]+\.(py|js|ts|go|java|rb|rs|md))'
        
        files = set()
        for msg in context:
            content = msg.get("content", "")
            matches = re.findall(file_pattern, content)
            files.update(match[0] for match in matches)
        
        return list(files)
```

#### 3. Decision Index

Searchable decision storage:

```python
class DecisionIndex:
    """Searchable index of decisions."""
    
    def __init__(
        self,
        storage_path: Path,
        enable_embeddings: bool = True
    ):
        self.storage_path = storage_path
        self.db = self._init_database()
        self.embeddings = EmbeddingIndex() if enable_embeddings else None
    
    def _init_database(self) -> sqlite3.Connection:
        """Initialize SQLite database for decisions."""
        
        conn = sqlite3.connect(self.storage_path / "decisions.db")
        
        conn.execute("""
            CREATE TABLE IF NOT EXISTS decisions (
                id TEXT PRIMARY KEY,
                timestamp DATETIME,
                session_id TEXT,
                decision TEXT,
                context TEXT,
                rationale TEXT,
                alternatives JSON,
                trade_offs JSON,
                stakeholders JSON,
                decision_maker TEXT,
                related_code JSON,
                related_features JSON,
                status TEXT,
                superseded_by TEXT,
                supersedes TEXT,
                tags JSON,
                keywords JSON
            )
        """)
        
        conn.execute("""
            CREATE VIRTUAL TABLE IF NOT EXISTS decisions_fts
            USING fts5(decision, context, rationale, content='decisions')
        """)
        
        return conn
    
    async def store(self, decision: Decision):
        """Store decision in index."""
        
        # Store in SQLite
        self.db.execute("""
            INSERT INTO decisions VALUES (
                :id, :timestamp, :session_id, :decision, :context,
                :rationale, :alternatives, :trade_offs, :stakeholders,
                :decision_maker, :related_code, :related_features,
                :status, :superseded_by, :supersedes, :tags, :keywords
            )
        """, self._decision_to_dict(decision))
        
        # Store in FTS
        self.db.execute("""
            INSERT INTO decisions_fts (rowid, decision, context, rationale)
            SELECT rowid, decision, context, rationale FROM decisions
            WHERE id = ?
        """, (decision.id,))
        
        self.db.commit()
        
        # Store embedding for semantic search
        if self.embeddings:
            text = f"{decision.decision} {decision.context} {decision.rationale}"
            await self.embeddings.add(decision.id, text)
    
    async def search(
        self,
        query: str,
        filters: DecisionFilters | None = None,
        limit: int = 10
    ) -> list[Decision]:
        """
        Search decisions.
        
        Search strategies:
        1. Semantic search (if embeddings enabled)
        2. Full-text search (SQLite FTS)
        3. Filtered search (by tags, dates, stakeholders)
        """
        
        results = []
        
        # Semantic search
        if self.embeddings:
            semantic_results = await self.embeddings.search(query, k=limit)
            results.extend(semantic_results)
        
        # Full-text search
        fts_results = self.db.execute("""
            SELECT id FROM decisions_fts
            WHERE decisions_fts MATCH ?
            LIMIT ?
        """, (query, limit)).fetchall()
        
        # Deduplicate and load full decisions
        decision_ids = list(set(
            r[0] for r in fts_results
        ) | set(r.id for r in results))
        
        decisions = [
            self._load_decision(did)
            for did in decision_ids[:limit]
        ]
        
        # Apply filters
        if filters:
            decisions = self._apply_filters(decisions, filters)
        
        return decisions
    
    async def get_chain(self, decision_id: str) -> DecisionChain:
        """Get decision evolution chain."""
        
        decision = self._load_decision(decision_id)
        
        # Find all related decisions
        revisions = []
        current = decision
        
        # Walk forward (superseded by)
        while current.superseded_by:
            next_decision = self._load_decision(current.superseded_by)
            revisions.append(next_decision)
            current = next_decision
        
        # Walk backward (supersedes)
        original = decision
        while original.supersedes:
            prev_decision = self._load_decision(original.supersedes)
            original = prev_decision
        
        return DecisionChain(
            original=original,
            revisions=revisions,
            current=current
        )
```

#### 4. Temporal Tracker

Tracks decision evolution:

```python
class TemporalTracker:
    """Tracks how decisions evolve over time."""
    
    async def link_superseding_decision(
        self,
        old_decision_id: str,
        new_decision_id: str,
        reason: str
    ):
        """Link new decision as superseding old one."""
        
        # Update old decision
        await self.index.update(old_decision_id, {
            "status": DecisionStatus.SUPERSEDED,
            "superseded_by": new_decision_id
        })
        
        # Update new decision
        await self.index.update(new_decision_id, {
            "supersedes": old_decision_id
        })
    
    async def detect_revisited_decisions(
        self,
        current_discussion: str,
        existing_decisions: list[Decision]
    ) -> list[Decision]:
        """
        Detect if current discussion revisits past decisions.
        
        Uses semantic similarity to find related decisions.
        """
        
        if not self.embeddings:
            return []
        
        # Semantic search for similar decisions
        similar = await self.embeddings.search(
            current_discussion,
            k=5
        )
        
        # Filter by similarity threshold
        revisited = [
            d for d in similar
            if d.similarity_score > 0.7
        ]
        
        return revisited
```

### Data Flow

```
Conversation Message
    │
    ▼
┌────────────────────┐
│ Decision Detector  │
│ - Keywords match?  │
│ - LLM confirms?    │
└──────┬─────────────┘
       │ (Decision detected)
       ▼
┌────────────────────┐
│ Metadata Extractor │
│ - What decided?    │
│ - Why?             │
│ - Alternatives?    │
│ - Stakeholders?    │
└──────┬─────────────┘
       │
       ▼
┌────────────────────┐
│ Code Linker        │
│ - Extract files    │
│ - Link features    │
└──────┬─────────────┘
       │
       ▼
┌────────────────────┐
│ Decision Index     │
│ - SQLite storage   │
│ - FTS index        │
│ - Embedding index  │
└────────────────────┘

Later Query:
"Why did we choose X?"
    │
    ▼
┌────────────────────┐
│ Search Engine      │
│ - Semantic search  │
│ - FTS search       │
│ - Filter results   │
└──────┬─────────────┘
       │
       ▼
   Decision(s)
   with full context
```

---

## Examples

### Example 1: Decision Detection and Extraction

```yaml
# Conversation
User: "Should we use REST or GraphQL for the new API?"
Agent: "Let's evaluate both options..."
[10 messages discussing trade-offs]
User: "Based on this, I think we should go with REST for now."
Agent: "Agreed. REST gives us simplicity and we can add GraphQL later if needed."

# Decision Detection
Message: "Agreed. REST gives us simplicity..."
Keywords matched: ["Agreed"]
LLM confirmation: 0.92 (high confidence)
→ Decision detected!

# Metadata Extraction
Decision extracted:
{
  "id": "dec-789",
  "timestamp": "2024-03-15T14:30:00Z",
  "decision": "Use REST API instead of GraphQL for new API",
  "context": "Choosing API architecture for new service",
  "rationale": "Team has REST expertise, simpler to implement, client needs are straightforward. Can add GraphQL later if requirements change.",
  "alternatives": [
    {
      "option": "GraphQL",
      "pros": ["Flexible queries", "Single endpoint", "Type safety"],
      "cons": ["Learning curve", "Overhead for simple queries", "Caching complexity"],
      "why_not_chosen": "Team lacks GraphQL experience, current requirements don't justify complexity"
    },
    {
      "option": "gRPC",
      "pros": ["Performance", "Type safety", "Streaming"],
      "cons": ["Browser support limited", "Tooling not familiar"],
      "why_not_chosen": "Primarily HTTP clients, no need for streaming yet"
    }
  ],
  "trade_offs": {
    "pros": ["Simpler implementation", "Team expertise", "Easier caching"],
    "cons": ["Over-fetching", "Multiple round-trips for complex queries"],
    "risks": ["May need to migrate to GraphQL later if requirements change"],
    "assumptions": ["Client needs remain simple", "Performance acceptable with caching"]
  },
  "stakeholders": ["@alice (architect)", "@bob (tech lead)", "@carol (PM)"],
  "decision_maker": "@alice",
  "related_code": ["src/api/routes/", "docs/api-design.md"],
  "tags": ["architecture", "api", "rest", "graphql"],
  "status": "active"
}

# Indexed and searchable
```

### Example 2: Decision Search

```yaml
# Query (6 months later)
New engineer: "Why aren't we using GraphQL?"

# Search
search_decisions("GraphQL")

# Results
[
  {
    "id": "dec-789",
    "decision": "Use REST API instead of GraphQL",
    "context": "Choosing API architecture for new service",
    "rationale": "Team has REST expertise, simpler to implement...",
    "timestamp": "2024-03-15",
    "stakeholders": ["@alice", "@bob", "@carol"],
    "status": "active"
  }
]

# Agent Response
"We decided on REST in March 2024. Here's the full context:

**Decision**: Use REST API instead of GraphQL

**Made by**: @alice (architect), with @bob and @carol

**Rationale**: 
- Team had REST expertise (faster to ship)
- Client needs were simple (no complex queries needed)
- Could revisit later if requirements change

**Alternatives Considered**:
- GraphQL: Flexible but steeper learning curve
- gRPC: High performance but limited browser support

**Current Status**: Active (6 months old)

**Related Code**: src/api/routes/, docs/api-design.md

**Should we revisit?** 
Current context suggests:
- We now have 15 endpoints (growing complexity)
- Mobile team requesting flexible queries
- GraphQL expertise improved (new hires)

Consider scheduling a review meeting to reassess."
```

### Example 3: Decision Chain

```yaml
# Original decision (March 2024)
Decision: "Use PostgreSQL for primary database"
Rationale: "ACID guarantees, team expertise"

# Revisited (September 2024)
Discussion: "Should we migrate to DynamoDB for better scale?"
[Discussion happens]
Decision: "Keep PostgreSQL, add Redis for caching"
Supersedes: dec-123 (partial revision)

# Query decision chain
get_decision_chain("dec-123")

# Returns
{
  "original": {
    "decision": "Use PostgreSQL",
    "timestamp": "2024-03-15",
    "status": "revisited"
  },
  "revisions": [
    {
      "decision": "Keep PostgreSQL, add Redis for caching",
      "timestamp": "2024-09-20",
      "rationale": "Scaling issues addressed with caching, migration cost not justified",
      "supersedes": "dec-123"
    }
  ],
  "current": { /* latest decision */ }
}
```

---

## Configuration

```yaml
# Full configuration example
contexts:
  - module: context-decision-journal
    config:
      # Detection
      auto_detect: true
      detection_keywords:
        - decided
        - decision:
        - "let's go with"
        - we should
        - agreed
        - consensus
      detection_threshold: 0.7
      llm_confirmation: true            # Use LLM to confirm
      
      # Extraction
      structured_extraction: true
      extract_rationale: true
      extract_alternatives: true
      extract_trade_offs: true
      extract_stakeholders: true
      
      # Linking
      link_to_code: true
      link_to_features: true
      auto_link_strategy: file_mention  # Look for file mentions
      
      # Indexing
      search_index: sqlite
      index_embeddings: true
      embedding_model: text-embedding-3-small
      enable_fts: true                  # Full-text search
      
      # Storage
      persistence: true
      storage_path: .amplifier/decisions/
      retention_days: 730               # 2 years
      
      # Querying
      enable_temporal_queries: true
      enable_stakeholder_queries: true
      enable_technology_queries: true
      
      # Evolution tracking
      track_superseded_decisions: true
      detect_revisited_decisions: true
```

---

## Security Considerations

### Data Storage
- **Persistence**: Decisions stored on disk (configurable retention)
- **Sensitive decisions**: May contain confidential information
- **Access control**: Decisions inherit session permissions

### Privacy
- **Stakeholder attribution**: Names extracted from conversations
- **Code links**: File paths exposed in decisions

```python
REQUIRED_CAPABILITIES = [
    "context:read",
    "context:write",
    "filesystem:read",              # For storage
]
```

---

## Testing Strategy

### Unit Tests

```python
def test_decision_detector_detects_keywords():
    detector = DecisionDetector()
    message = {"content": "We've decided to use PostgreSQL"}
    result = detector.detect(message, [])
    assert result.is_decision

def test_metadata_extractor_extracts_alternatives():
    extractor = MetadataExtractor()
    # Test extraction from sample conversation
    ...

def test_decision_index_search():
    index = DecisionIndex(storage_path=Path("/tmp/test"))
    # Store decisions
    # Search and verify results
    ...
```

### Integration Tests

```python
async def test_full_decision_lifecycle():
    context = DecisionJournalContext(config)
    
    # Add conversation leading to decision
    await context.add_message({"content": "Should we use X or Y?"})
    await context.add_message({"content": "Let's go with X"})
    
    # Search for decision
    results = await context.search_decisions("use X")
    assert len(results) == 1
    assert "X" in results[0].decision
```

---

## Dependencies

### Required
- `amplifier-core`
- `sqlite3` (stdlib)

### Optional
- `openai` - For embeddings
- `tiktoken` - Token counting

---

## Open Questions

1. **Decision confidence**: Should we track confidence/certainty of decisions?
2. **Decision categories**: Should we categorize decisions (technical, product, design)?
3. **Automatic linking**: How aggressive should automatic code linking be?
4. **Notification**: Should stakeholders be notified when their decisions are superseded?
5. **Export**: Should decisions be exportable to Markdown/Confluence/etc.?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
