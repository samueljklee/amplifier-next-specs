# context-stakeholder-lenses

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-context-stakeholder-lenses`

## Overview

A context manager that filters and shapes conversation context based on stakeholder roles (PM, engineering, design, security, etc.). Each role sees a tailored view of the conversation with relevant information emphasized and irrelevant details filtered out, while maintaining the ability to reference cross-lens information when needed.

### Value Proposition

| Without | With |
|---------|------|
| Everyone sees all technical details | PM sees business context, engineers see code, designers see UX |
| Wading through irrelevant information | Role-specific context filtering |
| One-size-fits-all responses | Responses tailored to stakeholder needs |
| Context window filled with noise | Efficient token usage per role |

### Use Cases

1. **Cross-functional reviews**: PM, eng, and design review same feature with different context
2. **Team collaboration**: Each team member gets relevant context for their role
3. **Onboarding**: New PMs see business context, new engineers see architecture
4. **Documentation**: Generate role-specific documentation from conversations
5. **Meeting summaries**: Different summaries for technical and non-technical stakeholders

---

## Contract

### ContextManager Interface

```python
class StakeholderLensesContext:
    """
    Context manager with role-based filtering.
    
    Each stakeholder lens:
    - Filters messages by relevance to role
    - Emphasizes role-specific information
    - Hides low-relevance details
    - Maintains references to other lenses
    """
    
    async def add_message(
        self, 
        message: dict[str, Any],
        metadata: MessageMetadata | None = None
    ) -> None:
        """
        Add message to context.
        
        Args:
            message: Standard Anthropic message format
            metadata: Optional metadata (tags, relevance scores per role)
        """
    
    async def get_messages(
        self,
        role: str | None = None,
        max_tokens: int | None = None
    ) -> list[dict[str, Any]]:
        """
        Get messages for specific role lens.
        
        Args:
            role: Role to filter for (None = base context)
            max_tokens: Optional token limit
            
        Returns:
            Filtered message list for role
        """
    
    async def set_active_role(self, role: str):
        """Set the active stakeholder role for filtering."""
    
    async def get_cross_lens_reference(
        self,
        message_id: str,
        source_role: str
    ) -> dict[str, Any]:
        """Retrieve message from another role's lens."""
```

### Configuration Schema

```toml
[[contexts]]
module = "context-stakeholder-lenses"
config = {
  # Role definitions
  lenses = [
    {
      role = "product",
      includes = ["business", "user_feedback", "roadmap", "metrics", "decisions"],
      excludes = ["implementation_details", "code_snippets", "technical_architecture"],
      weight = 1.0,
      emoji = "📊"
    },
    {
      role = "engineering",
      includes = ["architecture", "tech_debt", "performance", "code", "dependencies"],
      excludes = ["marketing_copy", "design_tokens", "brand_guidelines"],
      weight = 1.0,
      emoji = "⚙️"
    },
    {
      role = "design",
      includes = ["ux", "accessibility", "design_system", "brand", "user_research"],
      excludes = ["database_schemas", "api_contracts", "infrastructure"],
      weight = 1.0,
      emoji = "🎨"
    },
    {
      role = "security",
      includes = ["security", "compliance", "authentication", "data_protection"],
      excludes = ["ui_details", "design_tokens"],
      weight = 1.2,
      emoji = "🔒"
    }
  ]
  
  # Filtering strategy
  filtering_strategy = "relevance_score"  # "relevance_score" | "tag_based" | "hybrid"
  min_relevance_threshold = 0.3           # Messages below this are filtered
  
  # Cross-lens references
  allow_cross_lens_references = true
  cross_lens_reference_format = "summary"  # "summary" | "full" | "link"
  
  # Compaction
  compaction_strategy = "per_lens"        # Compact each lens independently
  preserve_shared_decisions = true        # Important decisions visible to all lenses
  
  # Auto-tagging
  auto_tag_messages = true                # Use LLM to tag messages with topics
  auto_score_relevance = true             # Calculate relevance scores per role
}
```

### Data Schema

```python
@dataclass
class MessageMetadata:
    """Metadata for context messages."""
    tags: list[str]                      # ["business", "technical", "design"]
    relevance_scores: dict[str, float]   # {role: score} - 0.0 to 1.0
    importance: str                      # "critical" | "high" | "medium" | "low"
    is_decision: bool                    # Shared decisions visible to all lenses
    
@dataclass
class LensDefinition:
    role: str                            # Role identifier
    includes: list[str]                  # Topics to include
    excludes: list[str]                  # Topics to exclude
    weight: float                        # Importance weight
    emoji: str                           # Visual identifier
    
@dataclass
class FilteredMessage:
    message: dict[str, Any]              # Original message
    relevance_score: float               # Relevance to current lens
    filtered_content: str | None         # Optional filtered version
    summary: str | None                  # Optional summary for low-relevance
```

### Events Emitted

| Event | When | Data |
|-------|------|------|
| `context:lens:activated` | Lens switched | role, previous_role |
| `context:message:tagged` | Message auto-tagged | message_id, tags |
| `context:message:filtered` | Message filtered | message_id, role, reason |
| `context:cross_lens:accessed` | Cross-lens reference | target_role, message_id |

---

## Architecture

### Component Diagram

```
┌──────────────────────────────────────────────────────────────┐
│              context-stakeholder-lenses                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐    ┌──────────────┐    ┌───────────────┐   │
│  │   Message   │───▶│   Relevance  │───▶│    Lens       │   │
│  │   Storage   │    │   Scorer     │    │   Filter      │   │
│  └─────────────┘    └──────────────┘    └───────┬───────┘   │
│         │                                        │           │
│         │                                        │           │
│  ┌──────▼──────┐    ┌──────────────┐    ┌───────▼───────┐   │
│  │   Auto      │    │  Cross-Lens  │    │  Compaction   │   │
│  │   Tagger    │    │  References  │    │   Engine      │   │
│  └─────────────┘    └──────────────┘    └───────────────┘   │
│                                                              │
└──────────────────────────────────────────────────────────────┘

Storage Structure:
┌────────────────────────────────┐
│ Base Storage                   │
│ - All messages                 │
│ - Metadata per message         │
└────────┬───────────────────────┘
         │
    ┌────┴──────┬──────────┬──────────┐
    ▼           ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│Product │ │ Eng    │ │Design  │ │Security│
│Lens    │ │Lens    │ │Lens    │ │Lens    │
│View    │ │View    │ │View    │ │View    │
└────────┘ └────────┘ └────────┘ └────────┘
 Filtered   Filtered   Filtered   Filtered
```

### Internal Components

#### 1. Message Storage

Central storage with metadata:

```python
class MessageStorage:
    """Stores all messages with metadata."""
    
    def __init__(self):
        self.messages: list[StoredMessage] = []
        self.message_index: dict[str, StoredMessage] = {}
    
    async def store(self, message: dict[str, Any], metadata: MessageMetadata) -> str:
        """Store message with metadata."""
        message_id = str(uuid.uuid4())
        
        stored = StoredMessage(
            id=message_id,
            message=message,
            metadata=metadata,
            timestamp=datetime.now()
        )
        
        self.messages.append(stored)
        self.message_index[message_id] = stored
        
        return message_id
    
    async def get_all(self) -> list[StoredMessage]:
        """Get all messages (base view)."""
        return self.messages.copy()
    
    async def get_by_id(self, message_id: str) -> StoredMessage | None:
        """Get specific message by ID."""
        return self.message_index.get(message_id)
```

#### 2. Relevance Scorer

Calculates relevance per role:

```python
class RelevanceScorer:
    """Calculates message relevance per role."""
    
    def __init__(self, lenses: list[LensDefinition]):
        self.lenses = {lens.role: lens for lens in lenses}
    
    async def score_message(
        self,
        message: dict[str, Any],
        tags: list[str]
    ) -> dict[str, float]:
        """
        Calculate relevance score per role.
        
        Scoring:
        1. Check includes: +1.0 per matching tag
        2. Check excludes: -1.0 per matching tag
        3. Normalize to 0.0-1.0 range
        """
        scores = {}
        
        for role, lens in self.lenses.items():
            score = 0.0
            
            # Positive signal from includes
            included_tags = set(tags) & set(lens.includes)
            score += len(included_tags)
            
            # Negative signal from excludes
            excluded_tags = set(tags) & set(lens.excludes)
            score -= len(excluded_tags)
            
            # Normalize (assume max 10 tags)
            normalized = max(0.0, min(1.0, (score + 5) / 10))
            
            scores[role] = normalized
        
        return scores
    
    async def score_message_with_llm(
        self,
        message: dict[str, Any],
        roles: list[str]
    ) -> dict[str, float]:
        """
        Use LLM to score relevance (more accurate but slower).
        
        Asks LLM: "On a scale of 0-10, how relevant is this message
        to a [role] stakeholder?"
        """
        prompt = f"""
Score this message's relevance to each role (0-10):

Message: {message['content'][:500]}

Roles: {', '.join(roles)}

Output as JSON: {{"role": score, ...}}
"""
        # Call LLM and parse scores
        ...
```

#### 3. Lens Filter

Filters messages for specific role:

```python
class LensFilter:
    """Filters messages for specific stakeholder lens."""
    
    def __init__(
        self, 
        lens: LensDefinition,
        min_relevance: float = 0.3
    ):
        self.lens = lens
        self.min_relevance = min_relevance
    
    async def filter_messages(
        self,
        messages: list[StoredMessage]
    ) -> list[FilteredMessage]:
        """
        Filter messages for this lens.
        
        Rules:
        1. Always include messages with is_decision=True
        2. Include messages with relevance >= threshold
        3. Summarize borderline messages (0.2-0.3 relevance)
        4. Exclude low-relevance messages (< 0.2)
        """
        filtered = []
        
        for stored in messages:
            relevance = stored.metadata.relevance_scores.get(self.lens.role, 0.0)
            
            # Always include decisions
            if stored.metadata.is_decision:
                filtered.append(FilteredMessage(
                    message=stored.message,
                    relevance_score=1.0,
                    filtered_content=None,
                    summary=None
                ))
                continue
            
            # High relevance - include fully
            if relevance >= self.min_relevance:
                filtered.append(FilteredMessage(
                    message=stored.message,
                    relevance_score=relevance,
                    filtered_content=None,
                    summary=None
                ))
            
            # Borderline - include as summary
            elif relevance >= (self.min_relevance - 0.1):
                summary = await self._summarize_for_lens(stored.message)
                filtered.append(FilteredMessage(
                    message=stored.message,
                    relevance_score=relevance,
                    filtered_content=None,
                    summary=summary
                ))
            
            # Low relevance - exclude
            # (but keep reference for cross-lens lookups)
        
        return filtered
    
    async def _summarize_for_lens(self, message: dict[str, Any]) -> str:
        """Create brief summary for borderline messages."""
        content = message.get("content", "")
        
        # Simple truncation or LLM-based summarization
        if len(content) < 100:
            return content
        
        # LLM summarization
        prompt = f"""
Summarize this message in one sentence for a {self.lens.role} stakeholder:

{content}

One-sentence summary:
"""
        # Returns brief summary
        ...
```

#### 4. Auto-Tagger

Automatically tags messages:

```python
class AutoTagger:
    """Automatically tags messages with topic categories."""
    
    KEYWORD_MAP = {
        "business": ["revenue", "market", "user", "customer", "roadmap"],
        "technical": ["code", "architecture", "database", "api", "performance"],
        "design": ["ux", "ui", "component", "design system", "accessibility"],
        "security": ["auth", "security", "encryption", "vulnerability", "compliance"],
        # ... more mappings
    }
    
    async def tag_message(self, message: dict[str, Any]) -> list[str]:
        """
        Auto-tag message with topic categories.
        
        Strategy:
        1. Keyword matching (fast, simple)
        2. LLM tagging (accurate, slower) - optional
        """
        content = message.get("content", "").lower()
        tags = set()
        
        # Keyword matching
        for category, keywords in self.KEYWORD_MAP.items():
            if any(keyword in content for keyword in keywords):
                tags.add(category)
        
        # Optional: LLM enhancement
        if self.use_llm_tagging and len(tags) < 2:
            llm_tags = await self._llm_tag(message)
            tags.update(llm_tags)
        
        return list(tags)
    
    async def _llm_tag(self, message: dict[str, Any]) -> list[str]:
        """Use LLM to tag message."""
        prompt = f"""
Categorize this message with relevant tags:

{message['content'][:500]}

Available categories: business, technical, design, security, product, 
infrastructure, compliance, user_research, marketing

Tags (comma-separated):
"""
        # Returns tags
        ...
```

#### 5. Cross-Lens References

Enable viewing other lenses:

```python
class CrossLensReference:
    """Handle cross-lens references."""
    
    def __init__(self, storage: MessageStorage):
        self.storage = storage
    
    async def get_reference(
        self,
        message_id: str,
        target_lens: str,
        format: str = "summary"
    ) -> dict[str, Any]:
        """
        Get message from another lens's perspective.
        
        Formats:
        - "summary": Brief summary
        - "full": Complete message
        - "link": Reference pointer
        """
        stored = await self.storage.get_by_id(message_id)
        if not stored:
            return None
        
        if format == "summary":
            return {
                "type": "cross_lens_reference",
                "source_lens": target_lens,
                "message_id": message_id,
                "summary": f"[From {target_lens}] {stored.message['content'][:100]}..."
            }
        elif format == "full":
            return {
                "type": "cross_lens_reference",
                "source_lens": target_lens,
                "message": stored.message
            }
        else:  # link
            return {
                "type": "cross_lens_reference",
                "source_lens": target_lens,
                "message_id": message_id,
                "reference": f"See {target_lens} lens for details"
            }
```

### Data Flow

```
Message Added
    │
    ▼
┌─────────────────────┐
│ Auto-Tag Message    │
│ tags: ["technical", │
│        "architecture"]
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Score Relevance     │
│ product: 0.2        │
│ engineering: 0.9    │
│ design: 0.3         │
│ security: 0.4       │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Store with Metadata │
│ (central storage)   │
└──────────┬──────────┘
           │
    ┌──────┴──────┬──────────┬──────────┐
    ▼             ▼          ▼          ▼
┌─────────┐  ┌─────────┐ ┌────────┐ ┌────────┐
│Product  │  │Engineer │ │Design  │ │Security│
│Lens     │  │Lens     │ │Lens    │ │Lens    │
│         │  │         │ │        │ │        │
│Score:0.2│  │Score:0.9│ │Score:  │ │Score:  │
│→Exclude │  │→Include │ │0.3     │ │0.4     │
│         │  │ (full)  │ │→Summary│ │→Include│
└─────────┘  └─────────┘ └────────┘ └────────┘
```

---

## Examples

### Example 1: Feature Discussion

```yaml
# Message 1: Feature request
User (PM): "We need PDF export. Users requested it 247 times."

Auto-tags: ["business", "user_feedback", "product"]
Relevance scores:
  product: 0.95 (user feedback, feature request)
  engineering: 0.60 (feature to build)
  design: 0.40 (UI needed)
  security: 0.20 (low relevance)

Product Lens: ✓ Include (high relevance)
Engineering Lens: ✓ Include (moderate relevance)
Design Lens: ✓ Include as summary
Security Lens: ✗ Exclude (below threshold)

# Message 2: Technical implementation
Developer: "We'll use Puppeteer for rendering. Need 200MB Docker image..."

Auto-tags: ["technical", "implementation", "infrastructure"]
Relevance scores:
  product: 0.20 (implementation detail)
  engineering: 1.00 (technical detail)
  design: 0.30 (affects design delivery)
  security: 0.50 (Docker image security)

Product Lens: ✗ Exclude (technical detail)
Engineering Lens: ✓ Include (highly relevant)
Design Lens: ✓ Include as summary ("Implementation uses Puppeteer")
Security Lens: ✓ Include (Docker security relevant)

# Message 3: Design concern
Designer: "Export modal needs format picker (PDF, PNG, SVG)"

Auto-tags: ["design", "ux", "component"]
Relevance scores:
  product: 0.60 (feature scope)
  engineering: 0.50 (interface to implement)
  design: 1.00 (design specification)
  security: 0.10 (low relevance)

Product Lens: ✓ Include (affects feature scope)
Engineering Lens: ✓ Include (needs implementation)
Design Lens: ✓ Include (full detail)
Security Lens: ✗ Exclude

# Message 4: DECISION
Team: "DECISION: Build PDF export in Q2 with security review"

Metadata: is_decision=True
Relevance scores: N/A (always included)

All Lenses: ✓ Include (decisions visible to all)

# Context retrieval examples

## Product Manager perspective
get_messages(role="product") returns:
1. "We need PDF export. Users requested it 247 times."
2. (Summary) "Export modal needs format picker"
3. "DECISION: Build PDF export in Q2 with security review"

## Engineering perspective
get_messages(role="engineering") returns:
1. "We need PDF export. Users requested it 247 times."
2. "We'll use Puppeteer for rendering. Need 200MB Docker image..."
3. "Export modal needs format picker (PDF, PNG, SVG)"
4. "DECISION: Build PDF export in Q2 with security review"

## Design perspective
get_messages(role="design") returns:
1. (Summary) "PDF export requested by users"
2. (Summary) "Implementation uses Puppeteer"
3. "Export modal needs format picker (PDF, PNG, SVG)"
4. "DECISION: Build PDF export in Q2 with security review"
```

### Example 2: Cross-Lens Reference

```python
# Designer asks: "What did engineering say about the timeline?"
# System detects query about engineering lens

# Retrieve from engineering lens
engineering_message = await context.get_cross_lens_reference(
    message_id="msg-123",
    source_role="engineering"
)

# Returns summary format
{
    "type": "cross_lens_reference",
    "source_lens": "engineering",
    "summary": "[From engineering] Implementation needs 12 weeks: 6 for infrastructure, 6 for feature development"
}
```

---

## Configuration

```yaml
# Full configuration example
contexts:
  - module: context-stakeholder-lenses
    config:
      # Lens definitions
      lenses:
        - role: product
          includes: [business, user_feedback, roadmap, metrics, decisions]
          excludes: [implementation_details, code_snippets, technical_architecture]
          weight: 1.0
          emoji: "📊"
          
        - role: engineering
          includes: [architecture, tech_debt, performance, code, dependencies]
          excludes: [marketing_copy, design_tokens, brand_guidelines]
          weight: 1.0
          emoji: "⚙️"
          
        - role: design
          includes: [ux, accessibility, design_system, brand, user_research]
          excludes: [database_schemas, api_contracts, infrastructure]
          weight: 1.0
          emoji: "🎨"
          
        - role: security
          includes: [security, compliance, authentication, data_protection]
          excludes: [ui_details, design_tokens]
          weight: 1.2
          emoji: "🔒"
      
      # Filtering
      filtering_strategy: hybrid          # relevance_score | tag_based | hybrid
      min_relevance_threshold: 0.3
      summary_threshold: 0.2              # Summarize messages between 0.2-0.3
      
      # Auto-processing
      auto_tag_messages: true
      auto_score_relevance: true
      use_llm_scoring: false              # Keyword-based by default (faster)
      
      # Cross-lens
      allow_cross_lens_references: true
      cross_lens_reference_format: summary
      
      # Compaction
      compaction_strategy: per_lens
      preserve_shared_decisions: true
      max_tokens_per_lens: 150000
      
      # Base context (no lens)
      include_base_context: true          # Allow viewing unfiltered context
```

---

## Security Considerations

### Data Isolation
- **Lens isolation**: Each lens sees filtered view only
- **Cross-lens access**: Controlled via configuration
- **Shared decisions**: Flagged messages visible to all roles

### Privacy
- **Role-based filtering**: Prevents accidental exposure of sensitive info
- **Metadata privacy**: Relevance scores not exposed to users

```python
REQUIRED_CAPABILITIES = [
    "context:read",
    "context:write",
]
```

---

## Testing Strategy

### Unit Tests

```python
def test_relevance_scorer_calculates_scores():
    scorer = RelevanceScorer(lenses=[...])
    scores = scorer.score_message(
        message={"content": "Deploy to production"},
        tags=["technical", "infrastructure"]
    )
    assert scores["engineering"] > 0.5
    assert scores["product"] < 0.5

def test_lens_filter_excludes_low_relevance():
    filter = LensFilter(lens=product_lens, min_relevance=0.3)
    messages = [
        StoredMessage(metadata=MessageMetadata(relevance_scores={"product": 0.9})),
        StoredMessage(metadata=MessageMetadata(relevance_scores={"product": 0.1}))
    ]
    filtered = filter.filter_messages(messages)
    assert len(filtered) == 1  # Only high-relevance included
```

### Integration Tests

```python
async def test_full_lens_filtering():
    context = StakeholderLensesContext(config)
    
    # Add messages
    await context.add_message({"role": "user", "content": "User wants feature X"})
    await context.add_message({"role": "assistant", "content": "Technical: Use database Y"})
    
    # Product lens should see first, not second
    product_messages = await context.get_messages(role="product")
    assert len(product_messages) == 1
    assert "feature X" in product_messages[0]["content"]
    
    # Engineering lens should see both
    eng_messages = await context.get_messages(role="engineering")
    assert len(eng_messages) == 2
```

---

## Dependencies

### Required
- `amplifier-core`

### Optional
- `tiktoken` - Token counting per lens

---

## Open Questions

1. **Dynamic lens creation**: Should users be able to define custom lenses?
2. **LLM scoring frequency**: Always use LLM or only for ambiguous messages?
3. **Lens switching UX**: How should users switch between lenses in CLI/UI?
4. **Historical lens views**: Should past conversations be re-filtered when lenses change?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
