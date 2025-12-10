# orchestrator-loop-observer-swarm

> **Priority**: P2 (Medium Value)
> **Status**: Draft
> **Module**: `amplifier-module-loop-observer-swarm`

## Overview

An orchestrator where multiple specialized agents observe a shared work queue or artifact, each contributing from their expertise without direct user interaction. Agents work semi-autonomously, coordinating through a shared context rather than explicit delegation.

### Value Proposition

| Without | With |
|---------|------|
| Single agent perspective | Multiple expert perspectives simultaneously |
| Sequential expert consultation | Parallel observation and contribution |
| User orchestrates between agents | Agents self-coordinate around shared work |
| Explicit handoffs required | Emergent collaboration through observation |

### Use Cases

1. **Code review swarm**: Security, performance, style experts all observe PR
2. **Design review**: UX, accessibility, brand experts observe design artifacts
3. **Document review**: Technical accuracy, style, completeness experts observe docs
4. **Architecture review**: Multiple stakeholders observe design proposal
5. **Incident investigation**: Multiple specialists observe logs/metrics simultaneously

### Modality Differentiator

This breaks the "chatbot" mental model by:
- **Multiple agents**: Not one assistant, but a team
- **Observation-based**: Agents watch and contribute, not prompted
- **Emergent coordination**: No explicit task assignment
- **Shared context**: All agents see the same evolving work

---

## Contract

### Orchestrator Interface

```python
from dataclasses import dataclass
from typing import Any
from enum import Enum


class ObserverRole(Enum):
    CRITIC = "critic"           # Find problems
    SUGGESTER = "suggester"     # Propose improvements
    VALIDATOR = "validator"     # Verify correctness
    SYNTHESIZER = "synthesizer" # Combine insights


@dataclass
class Observer:
    """A specialized agent that observes shared work."""
    name: str
    role: ObserverRole
    expertise: str
    trigger_conditions: list[str]  # When to activate
    system_prompt: str


@dataclass
class Observation:
    """An observation from an agent."""
    observer: str
    type: str  # "issue" | "suggestion" | "approval" | "question"
    content: str
    confidence: float
    references: list[str]  # What part of artifact this relates to
    priority: str  # "high" | "medium" | "low"


@dataclass
class SwarmResult:
    """Aggregated result from observer swarm."""
    observations: list[Observation]
    consensus: dict[str, Any]  # Areas of agreement
    conflicts: list[dict]  # Areas of disagreement
    synthesis: str  # Combined insight
    recommendations: list[str]


class ObserverSwarmOrchestrator:
    """
    Orchestrator that coordinates multiple observer agents.

    Flow:
    1. Present artifact to swarm
    2. Each observer analyzes from their perspective
    3. Observations are shared to shared context
    4. Observers can react to each other's observations
    5. Synthesizer combines insights
    6. Result includes all perspectives
    """

    async def execute(
        self,
        prompt: str,  # Artifact to observe or task description
        context: Any,
        providers: dict[str, Any],
        tools: dict[str, Any],
        hooks: Any,
        coordinator: Any | None = None,
    ) -> str:
        """Execute observer swarm on artifact."""

    async def spawn_observers(
        self,
        artifact: str,
        observers: list[Observer],
    ) -> list[Observation]:
        """Spawn all observers to analyze artifact."""

    async def coordinate_round(
        self,
        observations: list[Observation],
        observers: list[Observer],
    ) -> list[Observation]:
        """Allow observers to react to each other."""

    async def synthesize(
        self,
        all_observations: list[Observation],
    ) -> SwarmResult:
        """Combine observations into final result."""
```

### Configuration Schema

```toml
[[orchestrators]]
module = "loop-observer-swarm"
config = {
  # Swarm composition
  observers = [
    {
      name = "security-expert",
      role = "critic",
      expertise = "security vulnerabilities, auth, injection",
      trigger_conditions = ["*.py", "auth/*", "api/*"],
      priority = 1,  # Higher priority observers go first
    },
    {
      name = "performance-expert",
      role = "critic",
      expertise = "performance, efficiency, scalability",
      trigger_conditions = ["*.py", "db/*", "api/*"],
      priority = 2,
    },
    {
      name = "ux-expert",
      role = "suggester",
      expertise = "user experience, error messages, API ergonomics",
      trigger_conditions = ["*.ts", "*.tsx", "api/*"],
      priority = 3,
    },
    {
      name = "synthesizer",
      role = "synthesizer",
      expertise = "combining perspectives, identifying consensus",
      trigger_conditions = ["always"],
      priority = 99,  # Always last
    },
  ]

  # Execution control
  parallel_observers = true           # Run observers in parallel
  max_concurrent = 4                  # Max parallel observers
  coordination_rounds = 2             # How many rounds of reaction
  require_consensus = false           # Must all observers agree?

  # Observation handling
  min_confidence = 0.6               # Minimum confidence to include
  dedupe_similar = true              # Combine similar observations
  conflict_resolution = "vote"        # "vote" | "priority" | "synthesize"

  # Output
  include_dissent = true             # Show minority opinions
  synthesis_style = "executive"       # "executive" | "detailed" | "action-items"
}
```

---

## Architecture

### Swarm Execution Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    ARTIFACT INPUT                            │
│  Code │ Design │ Document │ Architecture │ Logs             │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    OBSERVER SPAWNER                          │
│  Select observers based on artifact type and triggers       │
└────────────────────────┬────────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┬──────────────┐
          ▼              ▼              ▼              ▼
     ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
     │Security │    │  Perf   │    │   UX    │    │  Style  │
     │ Expert  │    │ Expert  │    │ Expert  │    │ Expert  │
     └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘
          │              │              │              │
          └──────────────┴──────┬──────┴──────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────┐
│                    SHARED OBSERVATION POOL                   │
│  All observations visible to all observers                  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼ (Coordination Rounds)
┌─────────────────────────────────────────────────────────────┐
│                    REACTION PHASE                            │
│  Observers can respond to each other's observations         │
│  "I agree with Security Expert's point about X..."          │
│  "Performance concern overrides UX suggestion because..."   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    SYNTHESIS                                 │
│  Combine observations │ Resolve conflicts │ Prioritize      │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    SWARM RESULT                              │
│  All observations │ Consensus │ Conflicts │ Recommendations │
└─────────────────────────────────────────────────────────────┘
```

### Observer Agents

Each observer is a specialized agent with:

```python
class ObserverAgent:
    """Individual observer in the swarm."""

    def __init__(self, config: Observer, providers: dict, tools: dict):
        self.config = config
        self.providers = providers
        self.tools = tools

    async def observe(self, artifact: str) -> list[Observation]:
        """Generate observations about the artifact."""

        prompt = f"""
        You are a {self.config.expertise} expert.
        Your role is to {self.config.role.value}.

        Analyze this artifact and provide observations:
        - For issues: describe the problem, severity, and location
        - For suggestions: describe the improvement and benefit
        - For approvals: confirm what looks good
        - For questions: ask clarifying questions

        Artifact:
        {artifact}
        """

        response = await self.providers["default"].complete(...)
        return parse_observations(response, observer=self.config.name)

    async def react(
        self,
        artifact: str,
        other_observations: list[Observation]
    ) -> list[Observation]:
        """React to other observers' observations."""

        prompt = f"""
        You are a {self.config.expertise} expert.

        Other experts have made these observations:
        {format_observations(other_observations)}

        Original artifact:
        {artifact}

        React to the other observations:
        - Agree or disagree with specific points
        - Add nuance from your expertise
        - Identify conflicts between recommendations
        - Support or challenge priorities
        """

        response = await self.providers["default"].complete(...)
        return parse_reactions(response, observer=self.config.name)
```

---

## Observer Patterns

### Code Review Swarm

```yaml
observers:
  - name: security-auditor
    role: critic
    expertise: |
      Security vulnerabilities:
      - Injection attacks (SQL, command, XSS)
      - Authentication/authorization flaws
      - Secrets exposure
      - Input validation
    system_prompt: |
      You are a security auditor. Your job is to find security vulnerabilities.
      Be thorough but avoid false positives. Rate severity: critical/high/medium/low.

  - name: performance-analyst
    role: critic
    expertise: |
      Performance concerns:
      - Algorithm complexity
      - Database query efficiency
      - Memory usage
      - Concurrency issues
    system_prompt: |
      You are a performance analyst. Find inefficiencies and bottlenecks.
      Suggest concrete improvements with expected impact.

  - name: maintainability-reviewer
    role: suggester
    expertise: |
      Code quality:
      - Readability
      - Test coverage
      - Documentation
      - Design patterns
    system_prompt: |
      You are a code quality advocate. Suggest improvements for maintainability.
      Balance perfectionism with pragmatism.

  - name: code-synthesizer
    role: synthesizer
    expertise: Combining technical feedback into actionable summary
    system_prompt: |
      You synthesize feedback from multiple experts.
      Prioritize: security > correctness > performance > style.
      Identify which feedback is essential vs nice-to-have.
```

### Design Review Swarm

```yaml
observers:
  - name: ux-researcher
    role: critic
    expertise: User experience, usability, user flows
    system_prompt: Evaluate from the user's perspective

  - name: accessibility-expert
    role: critic
    expertise: WCAG compliance, screen readers, color contrast
    system_prompt: Ensure design is accessible to all users

  - name: brand-guardian
    role: validator
    expertise: Brand guidelines, visual consistency, tone
    system_prompt: Verify alignment with brand standards

  - name: engineering-liaison
    role: suggester
    expertise: Implementation feasibility, component reuse
    system_prompt: Flag implementation challenges early
```

---

## Events

```python
# Swarm lifecycle
"swarm:start"              # Swarm begins
"swarm:observer:spawn"     # Observer agent created
"swarm:observation:add"    # New observation added to pool
"swarm:round:start"        # Coordination round begins
"swarm:round:end"          # Coordination round ends
"swarm:synthesis:start"    # Synthesis begins
"swarm:end"                # Swarm completes

# Observation events
"observation:conflict"     # Conflicting observations detected
"observation:consensus"    # Observers agree on point
"observation:escalate"     # High-priority observation
```

---

## Examples

### CLI Usage

```bash
# Review a PR with observer swarm
amplifier swarm review \
  --artifact "$(git diff main)" \
  --observers security,performance,style

# Review design artifact
amplifier swarm review \
  --artifact ./designs/new-feature.figma \
  --observers ux,accessibility,brand

# Custom swarm config
amplifier swarm review \
  --config ./swarm-config.yaml \
  --artifact ./proposal.md
```

### Programmatic Usage

```python
from amplifier_core import AmplifierSession

session = AmplifierSession(profile="observer-swarm")

result = await session.run_swarm(
    artifact=pr_diff,
    observers=["security", "performance", "style"],
    coordination_rounds=2,
)

print(f"Consensus items: {len(result.consensus)}")
print(f"Conflicts: {len(result.conflicts)}")
print(f"\nRecommendations:")
for rec in result.recommendations:
    print(f"  - {rec}")
```

### Integration with PR Review

```yaml
# Combined with event-driven orchestrator
sources:
  - type: git_pr
    events: [opened, synchronize]
    handler: swarm-review

handlers:
  swarm-review:
    orchestrator: loop-observer-swarm
    config:
      observers: [security, performance, style]
      coordination_rounds: 1
      synthesis_style: "action-items"

    post_action:
      tool: tool-github
      action: comment
      body: "{synthesis}"
```

---

## Conflict Resolution

```python
class ConflictResolver:
    """Resolve conflicting observations."""

    async def resolve_by_vote(
        self,
        conflicting: list[Observation]
    ) -> Observation:
        """Most observers win."""

    async def resolve_by_priority(
        self,
        conflicting: list[Observation]
    ) -> Observation:
        """Highest priority observer wins."""

    async def resolve_by_synthesis(
        self,
        conflicting: list[Observation]
    ) -> str:
        """LLM synthesizes a resolution."""

        prompt = f"""
        These expert observations conflict:

        {format_conflicts(conflicting)}

        Synthesize a resolution that:
        1. Acknowledges each perspective
        2. Identifies the core tension
        3. Recommends a path forward
        4. Notes trade-offs
        """
```

---

## Testing Strategy

```python
class TestObserverSwarm:
    async def test_all_observers_contribute(self):
        """Each observer produces observations."""

    async def test_parallel_execution(self):
        """Observers run in parallel."""

    async def test_reaction_rounds(self):
        """Observers react to each other."""

    async def test_conflict_detection(self):
        """Conflicting observations are identified."""

    async def test_synthesis_includes_all(self):
        """Final synthesis includes all perspectives."""

    async def test_consensus_tracking(self):
        """Agreement between observers is tracked."""
```

---

## Open Questions

1. **Observer weighting**: Should some observers have more influence than others?
2. **Dynamic spawning**: Can observers request additional specialists?
3. **Iterative refinement**: Can swarm re-observe after artifact is modified?
4. **Cost control**: How to limit tokens when many observers are active?
5. **Observer memory**: Should observers remember previous reviews?
