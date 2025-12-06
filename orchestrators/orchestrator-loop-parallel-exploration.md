# orchestrator-loop-parallel-exploration

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-loop-parallel-exploration`

## Overview

An orchestrator that explores multiple solution approaches in parallel, evaluates each against defined criteria, and presents a comparative analysis with recommendations. Enables rapid exploration of solution space by generating multiple prototypes simultaneously rather than exploring sequentially.

### Value Proposition

| Without | With |
|---------|------|
| Sequential exploration (try A, fail, try B, etc.) | Parallel exploration of 3-5 approaches simultaneously |
| Anchor on first solution | Fair evaluation of multiple alternatives |
| Miss better alternatives | Comprehensive solution space coverage |
| Subjective comparison | Structured evaluation matrix with criteria |

### Use Cases

1. **Architecture decisions**: Evaluate multiple database choices (Postgres, MongoDB, DynamoDB) in parallel
2. **API design**: Compare REST, GraphQL, gRPC with working examples
3. **Algorithm selection**: Test multiple sorting/search algorithms with benchmarks
4. **Design patterns**: Explore different implementation patterns (Strategy, Factory, etc.)
5. **Technology choices**: Compare frameworks/libraries with proof-of-concepts

---

## Contract

### Orchestrator Interface

```python
class ParallelExplorationOrchestrator:
    """
    Orchestrator that explores multiple solution approaches in parallel.
    
    Flow:
    1. Receive problem statement
    2. Generate N different approaches
    3. Spawn sub-agents to develop each approach in parallel
    4. Collect all approach results
    5. Evaluate against criteria
    6. Generate comparison matrix
    7. Recommend best approach(es)
    """
    
    async def execute(
        self,
        prompt: str,
        context: ContextManager,
        providers: dict[str, Any],
        tools: dict[str, Any],
        hooks: HookRegistry,
        coordinator: ModuleCoordinator | None = None,
    ) -> str:
        """
        Execute parallel exploration.
        
        Args:
            prompt: Problem to solve
            context: Conversation context
            providers: Available LLM providers
            tools: Available tools
            hooks: Hook registry for events
            coordinator: Module coordinator
            
        Returns:
            Comparison matrix with recommendations
        """
```

### Configuration Schema

```toml
[[orchestrators]]
module = "loop-parallel-exploration"
config = {
  # Exploration parameters
  num_approaches = 4                     # How many alternatives to explore
  approach_generation = "automatic"      # "automatic" | "user_specified"
  
  # Evaluation criteria
  evaluation_criteria = [
    "performance",
    "cost", 
    "complexity",
    "scalability",
    "maintainability"
  ]
  
  # Criteria weights (for scoring)
  weights = {
    performance = 1.0,
    cost = 0.8,
    complexity = 1.2,
    scalability = 0.9,
    maintainability = 1.0
  }
  
  # Prototype generation
  generate_prototypes = true             # Generate working examples
  prototype_depth = "proof_of_concept"   # "sketch" | "proof_of_concept" | "production_ready"
  
  # Execution
  parallel_execution = true              # Run all explorations simultaneously
  timeout_per_approach = 180             # Seconds per approach
  
  # Output control
  comparison_format = "matrix"           # "matrix" | "narrative" | "both"
  include_code_samples = true
  include_trade_off_analysis = true
}
```

### Events Emitted

| Event | When | Data |
|-------|------|------|
| `orchestrator:exploration:start` | Exploration begins | num_approaches, criteria |
| `orchestrator:approach:generated` | Approach identified | approach_id, description |
| `orchestrator:approach:start` | Approach exploration starts | approach_id |
| `orchestrator:approach:complete` | Approach done | approach_id, scores |
| `orchestrator:evaluation:complete` | All evaluated | top_approach, comparison_matrix |

---

## Architecture

### Component Diagram

```
┌───────────────────────────────────────────────────────────────┐
│          orchestrator-loop-parallel-exploration               │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌───────────────┐    ┌────────────────┐    ┌─────────────┐  │
│  │   Approach    │───▶│    Parallel    │───▶│  Evaluator  │  │
│  │  Generator    │    │    Executor    │    │             │  │
│  └───────────────┘    └────────────────┘    └──────┬──────┘  │
│                                                     │         │
│                                                     │         │
│  ┌───────────────┐    ┌────────────────┐    ┌──────▼──────┐  │
│  │  Prototype    │    │  Comparison    │    │  Scoring    │  │
│  │  Generator    │    │    Matrix      │    │   Engine    │  │
│  └───────────────┘    └────────────────┘    └─────────────┘  │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### Internal Components

#### 1. Approach Generator

Generates alternative solution approaches:

```python
@dataclass
class Approach:
    id: str                          # UUID
    name: str                        # "Approach A: PostgreSQL"
    description: str                 # High-level description
    rationale: str                   # Why this approach?
    key_characteristics: list[str]   # Defining features
    prototype: str | None            # Code/config example
    evaluation_scores: dict[str, float] | None  # Criteria scores

class ApproachGenerator:
    """Generates diverse solution approaches for exploration."""
    
    def __init__(self, num_approaches: int = 4):
        self.num_approaches = num_approaches
    
    async def generate_approaches(
        self,
        problem: str,
        context: ContextManager
    ) -> list[Approach]:
        """
        Generate diverse approaches to solve the problem.
        
        Uses LLM to brainstorm alternatives, aiming for:
        - Different paradigms (SQL vs NoSQL, REST vs GraphQL)
        - Different trade-offs (performance vs simplicity)
        - Different implementation strategies
        """
        prompt = f"""
You are exploring solutions to this problem:

{problem}

Generate {self.num_approaches} different approaches. Each approach should:
1. Take a fundamentally different strategy
2. Have clear pros and cons
3. Be viable (not strawman alternatives)

For each approach, provide:
- Name: Brief identifier
- Description: What is this approach?
- Rationale: Why would someone choose this?
- Key characteristics: What defines this approach?

Output as JSON array.
"""
        
        # Get LLM to generate approaches
        result = await self._call_llm(prompt, context)
        approaches = self._parse_approaches(result)
        
        # Assign UUIDs
        for approach in approaches:
            approach.id = str(uuid.uuid4())
        
        return approaches
    
    def _parse_approaches(self, llm_output: str) -> list[Approach]:
        """Parse LLM output into Approach objects."""
        # Extract JSON, create Approach objects
        ...
```

#### 2. Parallel Executor

Executes exploration of all approaches simultaneously:

```python
class ParallelExecutor:
    """Executes exploration of approaches in parallel."""
    
    async def explore_all(
        self,
        approaches: list[Approach],
        problem: str,
        context: ContextManager,
        coordinator: ModuleCoordinator,
        config: ExplorationConfig
    ) -> list[Approach]:
        """Explore all approaches in parallel."""
        
        # Create exploration tasks
        tasks = [
            self._explore_approach(approach, problem, context, coordinator, config)
            for approach in approaches
        ]
        
        # Execute in parallel
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # Filter errors
        explored_approaches = []
        for i, result in enumerate(results):
            if isinstance(result, Exception):
                logger.error(f"Approach {approaches[i].name} failed: {result}")
            else:
                explored_approaches.append(result)
        
        return explored_approaches
    
    async def _explore_approach(
        self,
        approach: Approach,
        problem: str,
        context: ContextManager,
        coordinator: ModuleCoordinator,
        config: ExplorationConfig
    ) -> Approach:
        """Explore a single approach in depth."""
        
        # Build exploration prompt
        exploration_prompt = f"""
# Approach: {approach.name}

## Description
{approach.description}

## Original Problem
{problem}

## Task
Develop this approach in detail:

1. **Architecture**: How would this work?
2. **Implementation**: {"Generate working code example" if config.generate_prototypes else "Describe implementation"}
3. **Pros**: What are the advantages?
4. **Cons**: What are the disadvantages?
5. **Evaluation**: Score this approach on:
   {chr(10).join(f"   - {criterion}: 0-10 scale" for criterion in config.evaluation_criteria)}

Be specific and realistic. This will be compared to other approaches.
"""
        
        # Use task tool to spawn exploration
        result = await coordinator.call_tool("task", {
            "agent": "developer-expertise:zen-architect",  # Or configurable agent
            "instruction": exploration_prompt
        })
        
        # Parse result and update approach
        return self._parse_exploration_result(result, approach, config)
    
    def _parse_exploration_result(
        self,
        result: str,
        approach: Approach,
        config: ExplorationConfig
    ) -> Approach:
        """Parse exploration result and extract scores."""
        
        # Extract prototype code (if generated)
        if config.generate_prototypes:
            approach.prototype = self._extract_code(result)
        
        # Extract evaluation scores
        scores = {}
        for criterion in config.evaluation_criteria:
            score = self._extract_score(result, criterion)
            scores[criterion] = score
        
        approach.evaluation_scores = scores
        
        return approach
```

#### 3. Evaluator & Scoring Engine

Evaluates approaches against criteria:

```python
class ScoringEngine:
    """Scores and ranks approaches based on evaluation criteria."""
    
    def __init__(self, weights: dict[str, float]):
        self.weights = weights
    
    def score_approaches(
        self,
        approaches: list[Approach],
        criteria: list[str]
    ) -> list[ScoredApproach]:
        """
        Score and rank approaches.
        
        Scoring:
        1. Normalize scores to 0-1 range
        2. Apply weights per criterion
        3. Calculate weighted average
        4. Rank by total score
        """
        scored = []
        
        for approach in approaches:
            # Calculate weighted score
            total_score = 0.0
            max_possible = 0.0
            
            for criterion in criteria:
                score = approach.evaluation_scores.get(criterion, 0.0)
                weight = self.weights.get(criterion, 1.0)
                
                # Normalize to 0-1 (assuming scores are 0-10)
                normalized_score = score / 10.0
                
                total_score += normalized_score * weight
                max_possible += weight
            
            # Final score (0-1 range)
            final_score = total_score / max_possible if max_possible > 0 else 0.0
            
            scored.append(ScoredApproach(
                approach=approach,
                total_score=final_score,
                criterion_scores=approach.evaluation_scores,
                rank=0  # Assigned after sorting
            ))
        
        # Sort by score (descending)
        scored.sort(key=lambda x: x.total_score, reverse=True)
        
        # Assign ranks
        for i, scored_approach in enumerate(scored):
            scored_approach.rank = i + 1
        
        return scored
    
    def identify_best_approach(
        self,
        scored_approaches: list[ScoredApproach],
        threshold: float = 0.1
    ) -> tuple[ScoredApproach, list[ScoredApproach]]:
        """
        Identify best approach and close alternatives.
        
        Returns:
        - Best approach (highest score)
        - Close alternatives (within threshold of best)
        """
        if not scored_approaches:
            raise ValueError("No approaches to evaluate")
        
        best = scored_approaches[0]
        
        # Find alternatives within threshold
        alternatives = [
            a for a in scored_approaches[1:]
            if (best.total_score - a.total_score) <= threshold
        ]
        
        return best, alternatives
```

#### 4. Comparison Matrix Generator

Generates structured comparison:

```python
class ComparisonMatrix:
    """Generates comparison matrix for approaches."""
    
    def generate_matrix(
        self,
        scored_approaches: list[ScoredApproach],
        criteria: list[str]
    ) -> str:
        """
        Generate comparison matrix table.
        
        Format:
        | Approach | Performance | Cost | Complexity | ... | Total Score | Rank |
        |----------|-------------|------|------------|-----|-------------|------|
        | A: Redis | 9/10        | 6/10 | 7/10       | ... | 0.85        | 1    |
        | B: CDN   | 8/10        | 8/10 | 6/10       | ... | 0.82        | 2    |
        """
        
        # Build header
        header = ["Approach"] + criteria + ["Total Score", "Rank"]
        
        # Build rows
        rows = []
        for scored in scored_approaches:
            row = [scored.approach.name]
            
            # Add criterion scores
            for criterion in criteria:
                score = scored.criterion_scores.get(criterion, 0.0)
                row.append(f"{score:.1f}/10")
            
            # Add total score and rank
            row.append(f"{scored.total_score:.2f}")
            row.append(str(scored.rank))
            
            rows.append(row)
        
        # Format as markdown table
        return self._format_table(header, rows)
    
    def generate_trade_off_analysis(
        self,
        scored_approaches: list[ScoredApproach],
        criteria: list[str]
    ) -> str:
        """
        Generate prose explanation of trade-offs.
        
        Example:
        "Approach A excels in performance (9/10) but is more expensive (6/10).
        Approach B offers better cost efficiency (8/10) at the expense of 
        slightly lower performance (8/10)."
        """
        analysis = []
        
        # Find best/worst per criterion
        for criterion in criteria:
            scores = [
                (a.approach.name, a.criterion_scores.get(criterion, 0.0))
                for a in scored_approaches
            ]
            scores.sort(key=lambda x: x[1], reverse=True)
            
            best_name, best_score = scores[0]
            worst_name, worst_score = scores[-1]
            
            analysis.append(
                f"**{criterion.title()}**: {best_name} leads ({best_score:.1f}/10), "
                f"{worst_name} trails ({worst_score:.1f}/10)"
            )
        
        return "\n".join(analysis)
```

### Data Flow

```
User Problem: "Design a caching strategy"
    │
    ▼
┌─────────────────────────────────────────┐
│ Generate Approaches                     │
│ → A: Redis in-memory cache              │
│ → B: CDN edge caching                   │
│ → C: Application-level cache            │
│ → D: Database query cache               │
└────────┬────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│ Explore All Approaches (Parallel)       │
├─────────────────────────────────────────┤
│ Approach A: Redis                       │
│ - Architecture: Centralized cache       │
│ - Prototype: Redis config + code        │
│ - Scores: perf=9, cost=6, complex=7     │
├─────────────────────────────────────────┤
│ Approach B: CDN                         │
│ - Architecture: Edge distribution       │
│ - Prototype: CDN rules + headers        │
│ - Scores: perf=8, cost=8, complex=6     │
├─────────────────────────────────────────┤
│ [... C and D ...]                       │
└────────┬────────────────────────────────┘
         │ (All complete)
         ▼
┌─────────────────────────────────────────┐
│ Score & Rank                            │
│ Weights: perf=1.0, cost=0.8, complex=1.2│
│                                         │
│ A: Redis → 0.85 (rank 1)                │
│ B: CDN   → 0.82 (rank 2)                │
│ C: App   → 0.75 (rank 3)                │
│ D: DB    → 0.60 (rank 4)                │
└────────┬────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│ Generate Comparison Matrix              │
│ + Trade-off Analysis                    │
│ + Recommendation                        │
└─────────────────────────────────────────┘
```

---

## Examples

### Example 1: Caching Strategy

```yaml
# Input
"Design a caching strategy for our API"

# Generated Approaches (4)

Approach A: Redis In-Memory Cache
Description: Centralized Redis cache with TTL
Rationale: Fast, mature, horizontal scale
Characteristics: [Centralized, Network-based, Configurable TTL]

Approach B: CDN Edge Caching
Description: Cache at CDN edge locations
Rationale: Geographic distribution, offload origin
Characteristics: [Distributed, HTTP-level, Edge computing]

Approach C: Application-Level Cache
Description: In-process cache (in-memory map)
Rationale: Zero network latency, simple
Characteristics: [Local, Fast, No shared state]

Approach D: Database Query Cache
Description: PostgreSQL query result cache
Rationale: Transparent, built-in, no new infra
Characteristics: [Database-level, Automatic, Limited control]

# Exploration Results (parallel)

## Approach A: Redis
Architecture:
- Redis cluster with read replicas
- Cache-aside pattern (lazy loading)
- TTL-based expiration

Prototype:
```python
import redis
client = redis.Redis(host='localhost', decode_responses=True)

def get_user(user_id):
    # Check cache
    cached = client.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)
    
    # Cache miss - fetch from DB
    user = db.query_user(user_id)
    client.setex(f"user:{user_id}", 3600, json.dumps(user))
    return user
```

Scores:
- Performance: 9/10 (sub-ms reads)
- Cost: 6/10 ($200/month for cluster)
- Complexity: 7/10 (need cluster ops)
- Scalability: 9/10 (horizontal scale)
- Maintainability: 7/10 (new dependency)

## Approach B: CDN Edge Caching
Architecture:
- CloudFlare/Fastly edge caching
- Cache-Control headers
- Purge API for invalidation

Prototype:
```python
# API response headers
@app.route('/api/users/<id>')
def get_user(id):
    user = db.query_user(id)
    response = jsonify(user)
    response.headers['Cache-Control'] = 'public, max-age=3600'
    response.headers['Vary'] = 'Authorization'
    return response
```

Scores:
- Performance: 8/10 (CDN latency)
- Cost: 8/10 ($50/month)
- Complexity: 6/10 (simpler than Redis)
- Scalability: 10/10 (global edge network)
- Maintainability: 8/10 (managed service)

[... Approaches C and D ...]

# Evaluation Matrix

| Approach | Performance | Cost | Complexity | Scalability | Maintainability | Total | Rank |
|----------|-------------|------|------------|-------------|-----------------|-------|------|
| A: Redis | 9/10 | 6/10 | 7/10 | 9/10 | 7/10 | 0.85 | 1 |
| B: CDN   | 8/10 | 8/10 | 6/10 | 10/10 | 8/10 | 0.82 | 2 |
| C: App   | 10/10 | 10/10 | 9/10 | 4/10 | 9/10 | 0.75 | 3 |
| D: DB    | 6/10 | 9/10 | 8/10 | 5/10 | 8/10 | 0.60 | 4 |

Weights applied: performance=1.0, cost=0.8, complexity=1.2, scalability=0.9, maintainability=1.0

# Trade-off Analysis

**Performance**: Application-level cache leads (10/10, zero latency), Database cache trails (6/10)
**Cost**: Application and Database caches are cheapest (no infra), Redis most expensive (cluster required)
**Complexity**: Application cache is simplest (9/10), Redis cluster is most complex (7/10)
**Scalability**: CDN excels (10/10, global), Application cache limited (4/10, single-instance)
**Maintainability**: Application and Database caches easiest (no ops), Redis requires cluster management

# Recommendation

**Primary Choice: Approach A (Redis)**
- Score: 0.85/1.0
- Best for: High-traffic APIs needing shared cache across instances
- Trade-off: Higher cost and complexity, but superior scalability

**Close Alternative: Approach B (CDN)**
- Score: 0.82/1.0 (within 0.03 of best)
- Consider if: API is mostly public, read-heavy, geographically distributed users
- Trade-off: Less control over cache invalidation

**Use Case Guidance:**
- Start with **Application-level cache** (C) if: Small app, single instance, simplicity critical
- Use **Redis** (A) if: Multi-instance, shared state, high traffic
- Use **CDN** (B) if: Public API, read-heavy, global users
- Avoid **Database cache** (D): Lowest scores, limited control
```

---

## Configuration

```yaml
# Full configuration example
orchestrators:
  - module: loop-parallel-exploration
    config:
      # Exploration
      num_approaches: 4
      approach_generation: automatic
      approach_diversity: high          # Encourage different paradigms
      
      # Evaluation criteria
      evaluation_criteria:
        - performance
        - cost
        - complexity
        - scalability
        - maintainability
        - security                      # Optional: add domain-specific criteria
      
      # Weights (customize per project)
      weights:
        performance: 1.0
        cost: 0.8
        complexity: 1.2                 # Emphasize simplicity
        scalability: 0.9
        maintainability: 1.0
        security: 1.5                   # High priority for security
      
      # Prototype generation
      generate_prototypes: true
      prototype_depth: proof_of_concept
      prototype_format: code            # code | pseudocode | diagram
      
      # Execution
      parallel_execution: true
      timeout_per_approach: 180
      agent_per_approach: developer-expertise:zen-architect
      
      # Output
      comparison_format: both           # matrix and narrative
      include_code_samples: true
      include_trade_off_analysis: true
      include_recommendation: true
      recommendation_style: decisive    # decisive | options | conditional
```

---

## Security Considerations

### Resource Limits
- **Timeout per approach**: Prevents runaway explorations
- **Max approaches**: Configurable limit (default 4, max 10)

### Code Generation
- **Prototype code**: May execute code if depth = production_ready
- **Validation**: Generated code should be reviewed before use

```python
REQUIRED_CAPABILITIES = [
    "orchestrator:execute",
    "tool:task:spawn",
    "context:read",
]
```

---

## Testing Strategy

### Unit Tests

```python
def test_approach_generator_creates_diverse_approaches():
    generator = ApproachGenerator(num_approaches=3)
    approaches = generator.generate_approaches("Design a cache")
    assert len(approaches) == 3
    # Should have different strategies
    assert len(set(a.name for a in approaches)) == 3

def test_scoring_engine_applies_weights():
    engine = ScoringEngine(weights={"perf": 2.0, "cost": 1.0})
    approach = Approach(
        evaluation_scores={"perf": 5, "cost": 5}
    )
    score = engine.score_approaches([approach], ["perf", "cost"])
    # Performance weighted 2x should dominate
    assert score[0].total_score > 0.5
```

### Integration Tests

```python
async def test_full_parallel_exploration():
    orchestrator = ParallelExplorationOrchestrator(config)
    result = await orchestrator.execute(
        prompt="Design an API authentication system",
        context=mock_context
    )
    # Should explore multiple approaches
    assert "Approach A" in result
    assert "Approach B" in result
    # Should have comparison matrix
    assert "| Approach |" in result
    # Should have recommendation
    assert "Recommendation" in result
```

---

## Dependencies

### Required
- `amplifier-core`
- `amplifier-module-tool-task`

### Optional
- `tabulate` - For pretty tables

---

## Open Questions

1. **Approach diversity**: How to ensure approaches are sufficiently different?
2. **Domain-specific criteria**: Should criteria be customizable per domain (web, ML, infra)?
3. **Interactive exploration**: Allow user to add approaches mid-exploration?
4. **Cost estimation**: Should we estimate actual costs ($$) vs relative scoring?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
