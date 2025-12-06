# orchestrator-loop-perspective-synthesis

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-loop-perspective-synthesis`

## Overview

An orchestrator that analyzes problems from multiple stakeholder perspectives in parallel, then synthesizes insights into a cohesive recommendation. Enables cross-functional decision-making by spawning specialized sub-agents for each role (PM, engineering, design, security, etc.) and combining their analyses.

### Value Proposition

| Without | With |
|---------|------|
| One-size-fits-all analysis | Role-specific perspectives (PM, eng, design, security) |
| Sequential stakeholder consultation | Parallel analysis, faster results |
| Missing blind spots | Each role highlights different concerns |
| Vague recommendations | Clear trade-offs per stakeholder |

### Use Cases

1. **Feature decisions**: "Should we build feature X?" analyzed by PM (market), eng (feasibility), design (UX)
2. **Architecture reviews**: System design evaluated by multiple technical specialists
3. **Incident post-mortems**: Root cause from ops, eng, product, security angles
4. **Compliance assessment**: Legal, security, privacy perspectives on new feature
5. **Technical design**: Database choice evaluated by performance, cost, ops perspectives

---

## Contract

### Orchestrator Interface

```python
class PerspectiveSynthesisOrchestrator:
    """
    Orchestrator that analyzes from multiple perspectives in parallel.
    
    Flow:
    1. Receive user query/problem
    2. Spawn sub-agents for each configured perspective
    3. Execute all perspectives in parallel
    4. Detect conflicts between perspectives
    5. Synthesize unified recommendation
    6. Generate role-specific reports
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
        Execute multi-perspective analysis.
        
        Args:
            prompt: User's question or problem
            context: Conversation context
            providers: Available LLM providers
            tools: Available tools
            hooks: Hook registry for events
            coordinator: Module coordinator
            
        Returns:
            Synthesized recommendation with all perspectives
        """
```

### Configuration Schema

```toml
[[orchestrators]]
module = "loop-perspective-synthesis"
config = {
  # Perspectives to analyze
  perspectives = [
    { 
      role = "product",
      agent = "agent-product-analyst",
      weight = 1.0,
      focus_areas = ["market", "user_feedback", "roadmap", "metrics"]
    },
    { 
      role = "engineering",
      agent = "agent-architect",
      weight = 1.0,
      focus_areas = ["architecture", "tech_debt", "performance", "scale"]
    },
    { 
      role = "design",
      agent = "agent-design-system",
      weight = 0.8,
      focus_areas = ["ux", "accessibility", "design_system", "brand"]
    },
    { 
      role = "security",
      agent = "agent-security-guardian",
      weight = 1.2,
      focus_areas = ["threat_model", "compliance", "data_protection"]
    }
  ]
  
  # Synthesis strategy
  synthesis_strategy = "weighted_voting"  # "weighted_voting" | "consensus" | "veto"
  conflict_resolution = "surface"         # "surface" | "mediate" | "escalate"
  
  # Execution
  parallel_execution = true               # Run all perspectives simultaneously
  timeout_per_perspective = 120           # Seconds per perspective
  
  # Output control
  include_individual_reports = true       # Show each perspective's full analysis
  include_conflict_analysis = true        # Highlight disagreements
  generate_role_specific_summaries = true # Summary per stakeholder type
}
```

### Events Emitted

| Event | When | Data |
|-------|------|------|
| `orchestrator:perspectives:start` | Analysis begins | perspectives_count, roles |
| `orchestrator:perspective:start` | Perspective analysis starts | role, agent |
| `orchestrator:perspective:complete` | Perspective done | role, recommendation, confidence |
| `orchestrator:conflict:detected` | Disagreement found | conflicting_roles, topic |
| `orchestrator:synthesis:complete` | Synthesis done | final_recommendation, consensus_score |

---

## Architecture

### Component Diagram

```
┌────────────────────────────────────────────────────────────────┐
│           orchestrator-loop-perspective-synthesis              │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────────┐    ┌────────────────┐    ┌──────────────┐ │
│  │  Perspective   │───▶│   Parallel     │───▶│  Conflict    │ │
│  │   Spawner      │    │   Executor     │    │  Detector    │ │
│  └────────────────┘    └────────────────┘    └──────┬───────┘ │
│                                                      │         │
│                                                      │         │
│  ┌────────────────┐    ┌────────────────┐    ┌──────▼───────┐ │
│  │   Synthesis    │◀───│   Weighting    │◀───│   Report     │ │
│  │    Engine      │    │    Engine      │    │  Generator   │ │
│  └────────────────┘    └────────────────┘    └──────────────┘ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Internal Components

#### 1. Perspective Spawner

Spawns specialized sub-agents for each role:

```python
@dataclass
class PerspectiveDefinition:
    role: str                        # "product", "engineering", etc.
    agent: str                       # Agent/profile to use
    weight: float                    # Importance weight (0.0-2.0)
    focus_areas: list[str]          # Areas this perspective focuses on
    prompt_template: str | None      # Custom prompt per perspective

@dataclass
class PerspectiveResult:
    role: str
    recommendation: str              # Main recommendation
    rationale: str                   # Reasoning
    confidence: float                # 0.0-1.0
    concerns: list[str]              # Risks/concerns raised
    questions: list[str]             # Questions for other perspectives
    metadata: dict[str, Any]         # Role-specific data

class PerspectiveSpawner:
    """Spawns and manages sub-agents for each perspective."""
    
    def __init__(self, perspectives: list[PerspectiveDefinition]):
        self.perspectives = perspectives
    
    async def spawn_all(
        self, 
        query: str,
        context: ContextManager,
        coordinator: ModuleCoordinator
    ) -> list[PerspectiveResult]:
        """Spawn all perspectives in parallel."""
        tasks = [
            self._spawn_perspective(p, query, context, coordinator)
            for p in self.perspectives
        ]
        
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # Filter out errors, log them
        valid_results = []
        for i, result in enumerate(results):
            if isinstance(result, Exception):
                logger.error(f"Perspective {self.perspectives[i].role} failed: {result}")
            else:
                valid_results.append(result)
        
        return valid_results
    
    async def _spawn_perspective(
        self,
        perspective: PerspectiveDefinition,
        query: str,
        context: ContextManager,
        coordinator: ModuleCoordinator
    ) -> PerspectiveResult:
        """Spawn a single perspective analysis."""
        # Build role-specific prompt
        perspective_prompt = self._build_perspective_prompt(
            perspective, 
            query
        )
        
        # Use task tool to spawn sub-agent with specific perspective
        result = await coordinator.call_tool("task", {
            "agent": perspective.agent,
            "instruction": perspective_prompt
        })
        
        # Parse result into structured format
        return self._parse_result(result, perspective.role)
    
    def _build_perspective_prompt(
        self,
        perspective: PerspectiveDefinition,
        query: str
    ) -> str:
        """Build a role-specific prompt."""
        template = perspective.prompt_template or self._default_template(perspective.role)
        
        prompt = f"""
# Your Role: {perspective.role.upper()} Perspective

You are analyzing this question from the {perspective.role} perspective.

## Focus Areas
{chr(10).join(f"- {area}" for area in perspective.focus_areas)}

## Query
{query}

## Task
Analyze this question from your role's perspective. Provide:

1. **Recommendation**: What should we do?
2. **Rationale**: Why do you recommend this?
3. **Concerns**: What risks or issues do you see?
4. **Questions**: What do you need to know from other perspectives?
5. **Confidence**: How confident are you? (0-100%)

Be specific and focus on your domain. Other perspectives will cover other areas.
"""
        return prompt.strip()
```

#### 2. Conflict Detector

Identifies disagreements between perspectives:

```python
@dataclass
class Conflict:
    topic: str
    conflicting_roles: list[str]
    positions: dict[str, str]        # role -> position
    severity: str                    # "minor" | "moderate" | "critical"

class ConflictDetector:
    """Detects conflicts between perspective recommendations."""
    
    async def detect_conflicts(
        self,
        results: list[PerspectiveResult]
    ) -> list[Conflict]:
        """
        Identify conflicts between perspectives.
        
        Conflicts occur when:
        1. Direct contradictions (build vs don't build)
        2. Different priorities (cost vs quality)
        3. Unresolved concerns between roles
        """
        conflicts = []
        
        # Check for direct contradictions
        recommendations = [r.recommendation for r in results]
        if self._has_contradictions(recommendations):
            conflicts.append(
                self._create_contradiction_conflict(results)
            )
        
        # Check for concern overlaps
        concern_conflicts = await self._analyze_concern_conflicts(results)
        conflicts.extend(concern_conflicts)
        
        # Check for priority misalignment
        priority_conflicts = self._detect_priority_conflicts(results)
        conflicts.extend(priority_conflicts)
        
        return conflicts
    
    def _has_contradictions(self, recommendations: list[str]) -> bool:
        """Detect if recommendations contradict each other."""
        # Simple heuristic: look for opposing keywords
        has_build = any("build" in r.lower() or "implement" in r.lower() 
                       for r in recommendations)
        has_dont_build = any("don't" in r.lower() or "defer" in r.lower() 
                            for r in recommendations)
        
        return has_build and has_dont_build
    
    async def _analyze_concern_conflicts(
        self,
        results: list[PerspectiveResult]
    ) -> list[Conflict]:
        """Use LLM to identify concern conflicts."""
        # Ask LLM to identify where concerns conflict
        prompt = f"""
        Analyze these perspectives for conflicting concerns:
        
        {self._format_results(results)}
        
        List conflicts where one role's concern conflicts with another's recommendation.
        """
        # Returns structured conflicts
        ...
```

#### 3. Synthesis Engine

Combines perspectives into unified recommendation:

```python
class SynthesisEngine:
    """Synthesizes multiple perspectives into cohesive recommendation."""
    
    def __init__(self, strategy: str = "weighted_voting"):
        self.strategy = strategy
    
    async def synthesize(
        self,
        results: list[PerspectiveResult],
        conflicts: list[Conflict],
        weights: dict[str, float]
    ) -> SynthesisResult:
        """
        Synthesize perspectives based on configured strategy.
        """
        if self.strategy == "weighted_voting":
            return await self._weighted_synthesis(results, weights)
        elif self.strategy == "consensus":
            return await self._consensus_synthesis(results)
        elif self.strategy == "veto":
            return await self._veto_synthesis(results)
        else:
            raise ValueError(f"Unknown strategy: {self.strategy}")
    
    async def _weighted_synthesis(
        self,
        results: list[PerspectiveResult],
        weights: dict[str, float]
    ) -> SynthesisResult:
        """
        Weighted voting: higher weight = more influence.
        
        Example:
        - Security (weight 1.2): Don't build
        - Product (weight 1.0): Build
        - Design (weight 0.8): Build
        
        Weighted scores:
        - Don't build: 1.2
        - Build: 1.8
        
        → Recommendation: Build (but highlight security concerns)
        """
        # Calculate weighted scores per recommendation
        recommendation_scores = {}
        
        for result in results:
            weight = weights.get(result.role, 1.0)
            score = result.confidence * weight
            
            rec = self._normalize_recommendation(result.recommendation)
            recommendation_scores[rec] = recommendation_scores.get(rec, 0) + score
        
        # Top recommendation
        top_rec = max(recommendation_scores, key=recommendation_scores.get)
        consensus_score = recommendation_scores[top_rec] / sum(recommendation_scores.values())
        
        # Collect dissenting perspectives
        dissenters = [
            r for r in results 
            if self._normalize_recommendation(r.recommendation) != top_rec
        ]
        
        return SynthesisResult(
            recommendation=top_rec,
            consensus_score=consensus_score,
            supporting_roles=[r.role for r in results if self._normalize_recommendation(r.recommendation) == top_rec],
            dissenting_roles=[r.role for r in dissenters],
            synthesis_rationale=await self._generate_rationale(results, top_rec),
            role_breakdowns=results
        )
    
    async def _consensus_synthesis(
        self,
        results: list[PerspectiveResult]
    ) -> SynthesisResult:
        """
        Consensus: Only recommend if all (or majority) agree.
        """
        recommendations = [
            self._normalize_recommendation(r.recommendation) 
            for r in results
        ]
        
        # Check for unanimous agreement
        if len(set(recommendations)) == 1:
            return SynthesisResult(
                recommendation=recommendations[0],
                consensus_score=1.0,
                supporting_roles=[r.role for r in results],
                dissenting_roles=[],
                synthesis_rationale="Unanimous agreement across all perspectives"
            )
        
        # No consensus - highlight disagreement
        return SynthesisResult(
            recommendation="No consensus reached",
            consensus_score=0.0,
            supporting_roles=[],
            dissenting_roles=[r.role for r in results],
            synthesis_rationale="Perspectives disagree. See individual analyses.",
            needs_escalation=True
        )
    
    async def _veto_synthesis(
        self,
        results: list[PerspectiveResult]
    ) -> SynthesisResult:
        """
        Veto: Any critical concern blocks recommendation.
        """
        # Check for critical concerns (typically from security, legal, compliance)
        blocking_concerns = [
            r for r in results
            if any(c.lower().startswith("critical") or c.lower().startswith("blocker") 
                   for c in r.concerns)
        ]
        
        if blocking_concerns:
            return SynthesisResult(
                recommendation="Blocked by critical concerns",
                consensus_score=0.0,
                supporting_roles=[],
                dissenting_roles=[r.role for r in blocking_concerns],
                synthesis_rationale=f"Blocked by {', '.join(r.role for r in blocking_concerns)}",
                blocking_concerns=[c for r in blocking_concerns for c in r.concerns]
            )
        
        # No blocks - synthesize normally
        return await self._weighted_synthesis(results, {r.role: 1.0 for r in results})
```

#### 4. Report Generator

Generates role-specific summaries:

```python
class ReportGenerator:
    """Generates reports tailored to different stakeholders."""
    
    def generate_full_report(
        self,
        synthesis: SynthesisResult,
        conflicts: list[Conflict]
    ) -> str:
        """Generate comprehensive report with all perspectives."""
        sections = [
            self._executive_summary(synthesis),
            self._recommendation_section(synthesis),
            self._perspective_breakdowns(synthesis.role_breakdowns),
            self._conflict_analysis(conflicts),
            self._next_steps(synthesis)
        ]
        
        return "\n\n".join(sections)
    
    def generate_role_specific_summary(
        self,
        role: str,
        synthesis: SynthesisResult
    ) -> str:
        """Generate summary for specific role (PM, eng, design)."""
        # Emphasize information relevant to this role
        
        if role == "product":
            return self._product_summary(synthesis)
        elif role == "engineering":
            return self._engineering_summary(synthesis)
        elif role == "design":
            return self._design_summary(synthesis)
        else:
            return self._generic_summary(synthesis)
    
    def _executive_summary(self, synthesis: SynthesisResult) -> str:
        """High-level summary for decision-makers."""
        return f"""
# Executive Summary

**Recommendation**: {synthesis.recommendation}
**Consensus Level**: {synthesis.consensus_score:.0%}
**Supporting**: {', '.join(synthesis.supporting_roles)}
**Concerns**: {', '.join(synthesis.dissenting_roles) if synthesis.dissenting_roles else 'None'}

{synthesis.synthesis_rationale}
"""
```

### Data Flow

```
User Query: "Should we build feature X?"
    │
    ▼
┌────────────────────────────────────────┐
│ Spawn Perspectives (Parallel)          │
├────────────────────────────────────────┤
│ → Product Agent: Market analysis       │
│ → Engineering Agent: Feasibility       │
│ → Design Agent: UX impact              │
│ → Security Agent: Threat model         │
└────────┬───────────────────────────────┘
         │ (All complete)
         ▼
┌────────────────────────────────────────┐
│ Collect Results                        │
│ - Product: Build (conf: 0.9)           │
│ - Engineering: Build, needs SRE (0.7)  │
│ - Design: Build, 8 weeks (0.8)         │
│ - Security: Build, review req'd (0.6)  │
└────────┬───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ Detect Conflicts                       │
│ - Moderate: Timeline (eng vs product)  │
│ - Minor: Security review process       │
└────────┬───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ Synthesize (Weighted Voting)           │
│ Weights: Product=1.0, Eng=1.0,         │
│          Design=0.8, Security=1.2      │
│                                        │
│ Result: BUILD (consensus: 0.75)        │
└────────┬───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ Generate Reports                       │
│ - Full report (all perspectives)       │
│ - PM summary (business focus)          │
│ - Eng summary (technical focus)        │
│ - Design summary (UX focus)            │
└────────────────────────────────────────┘
```

---

## Examples

### Example 1: Feature Decision

```yaml
# Query
"Should we add real-time collaboration to our editor?"

# Spawned Perspectives (parallel)

## Product Perspective
Role: product
Agent: agent-product-analyst
Result:
  Recommendation: Build in Q2
  Rationale: |
    - Market: Top 3 feature request (247 users)
    - Competition: Figma, Google Docs have it
    - Revenue: Unblocks 3 enterprise deals ($450K ARR)
    - Risk: Delays Q1 roadmap
  Concerns:
    - Delays other priorities
    - Complex, may take longer than estimated
  Confidence: 0.85

## Engineering Perspective
Role: engineering
Agent: agent-architect
Result:
  Recommendation: Build, but hire SRE first
  Rationale: |
    - Architecture: Need WebSocket infra (6 weeks)
    - Tech debt: State management refactor needed (2 weeks)
    - Scale: 10K concurrent = $5K/month infra
    - Ops: No on-call experience with WebSockets
  Concerns:
    - Operational complexity
    - Hiring timeline unknown
  Confidence: 0.70

## Design Perspective
Role: design
Agent: agent-design-system
Result:
  Recommendation: Build, budget 8 weeks for design
  Rationale: |
    - UX: Need conflict resolution UI
    - Components: Cursors, avatars, presence (4 new)
    - Accessibility: Real-time updates for screen readers
    - Risk: Can't half-ship, UX must be polished
  Concerns:
    - 8 weeks design is optimistic
    - Need design system expansion
  Confidence: 0.80

## Security Perspective
Role: security
Agent: agent-security-guardian
Result:
  Recommendation: Build with security review
  Rationale: |
    - Threat: User impersonation, data leakage
    - Compliance: GDPR for shared sessions
    - Audit: Need activity logs for enterprise
  Concerns:
    - Critical: Security review before launch
    - Moderate: Pen test required
  Confidence: 0.60

# Conflict Detection
Conflicts:
  - Topic: Timeline
    Roles: [product, engineering]
    Severity: moderate
    Details: Product wants Q1, engineering needs Q2
    
  - Topic: Prerequisites
    Roles: [engineering]
    Severity: moderate
    Details: Engineering blocks on SRE hire

# Synthesis (Weighted Voting)
Weights:
  product: 1.0
  engineering: 1.0
  design: 0.8
  security: 1.2

Scores:
  "Build": 1.0 + 1.0 + 0.8 = 2.8
  "Build with conditions": 1.2 = 1.2
  Total: 4.0
  
Final Recommendation: Build (consensus: 70%)

# Generated Report

## Executive Summary
**Recommendation**: Build real-time collaboration feature
**Consensus**: 70% (weighted agreement)
**Timeline**: Q2 target (not Q1)
**Prerequisites**: Hire SRE, complete security review

All perspectives agree the feature should be built, but disagree on timing
and prerequisites.

## Recommendation Details

**Build the feature with these conditions:**
1. Target Q2 (not Q1) - allows time for infrastructure
2. Hire SRE before starting - mitigates operational risk
3. Security review required - addresses compliance concerns
4. Budget 8 weeks for design - ensures quality UX

**Estimated Investment:**
- Engineering: 12 weeks (6 infra + 6 feature)
- Design: 8 weeks
- Infrastructure: $60K/year
- Prerequisites: 1 SRE hire

**Expected Return:**
- Unlocks $450K ARR in enterprise deals
- Top user-requested feature (247 requests)
- Competitive parity with Figma, Google Docs

## Perspective Breakdowns
[Full details from each perspective]

## Conflicts & Trade-offs
1. **Timeline** (Product vs Engineering)
   - Product wants Q1 for enterprise deals
   - Engineering needs Q2 for proper infrastructure
   - Resolution: Q2 target, communicate delay to prospects

2. **Prerequisites** (Engineering)
   - SRE hire needed before starting
   - Hiring timeline: 1-2 months
   - Mitigation: Start recruiting now

## Next Steps
1. [ ] Approve Q2 timeline
2. [ ] Start SRE recruitment
3. [ ] Schedule security review meeting
4. [ ] Allocate design team capacity
```

---

## Security Considerations

### Sub-agent Spawning
- **Isolation**: Each perspective gets isolated context
- **Resource limits**: Timeout per perspective prevents runaway agents
- **Error handling**: Failed perspectives don't block synthesis

### Data Access
```python
REQUIRED_CAPABILITIES = [
    "orchestrator:execute",
    "tool:task:spawn",           # Spawn sub-agents
    "context:read",
    "context:write",
]
```

---

## Testing Strategy

### Unit Tests

```python
def test_conflict_detector_identifies_contradictions():
    results = [
        PerspectiveResult(role="pm", recommendation="Build now"),
        PerspectiveResult(role="security", recommendation="Don't build")
    ]
    detector = ConflictDetector()
    conflicts = detector.detect_conflicts(results)
    assert len(conflicts) > 0
    assert "Build" in conflicts[0].topic

def test_weighted_synthesis_respects_weights():
    results = [
        PerspectiveResult(role="pm", recommendation="Build", confidence=1.0),
        PerspectiveResult(role="security", recommendation="Don't", confidence=1.0)
    ]
    weights = {"pm": 1.0, "security": 2.0}
    engine = SynthesisEngine("weighted_voting")
    result = engine.synthesize(results, [], weights)
    assert "Don't" in result.recommendation  # Security weight wins
```

### Integration Tests

```python
async def test_full_perspective_synthesis():
    orchestrator = PerspectiveSynthesisOrchestrator(config)
    result = await orchestrator.execute(
        prompt="Should we migrate to microservices?",
        context=mock_context
    )
    # Should have multiple perspectives
    assert len(orchestrator.perspectives_results) >= 3
    # Should have synthesis
    assert "recommendation" in result.lower()
```

---

## Dependencies

### Required
- `amplifier-core`
- `amplifier-module-tool-task` - For spawning sub-agents

### Optional
- `amplifier-collection-recipes` - For agent profiles

---

## Open Questions

1. **Dynamic perspective selection**: Should perspectives be auto-selected based on query?
2. **Async review**: How to handle long-running perspective analyses?
3. **Perspective plugins**: Should perspectives be pluggable/configurable?
4. **Cross-perspective dialogue**: Should perspectives be able to ask each other questions?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
