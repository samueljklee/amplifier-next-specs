# tool-collaborative

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-tool-collaborative`

## Overview

A tool for coordinating multiple specialized agents on complex tasks. Enables parallel specialist analysis, hierarchical delegation, and synthesis of multiple perspectives - all invoked as a tool call from the main session.

### Why a Tool, Not an Orchestrator?

Multi-agent collaboration is **policy** (when to collaborate, which agents, how to synthesize) not **mechanism**. The kernel orchestrator should stay simple (loop-basic/streaming). Sophisticated coordination patterns belong at the tool layer where:

- The agent decides **when** collaboration is valuable
- Collaboration can be **mixed** with normal conversation
- Patterns can **evolve** without touching the kernel
- It **composes** naturally with other tools

### Value Proposition

| Without | With |
|---------|------|
| Single generalist perspective | Multiple specialist perspectives |
| Sequential analysis | Parallel execution |
| Manual coordination | Automated synthesis |
| Context overload | Focused context per specialist |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Main Session (loop-basic)                                      │
│                                                                 │
│  Agent: "This needs multiple perspectives..."                   │
│    │                                                            │
│    └─► tool-collaborative.execute(...)                         │
│          │                                                      │
│          ├─► spawn: security-specialist ──────┐                │
│          ├─► spawn: performance-specialist ───┼─► parallel     │
│          ├─► spawn: maintainability-specialist┘                │
│          │                                                      │
│          └─► coordinator synthesizes ──► return result         │
│                                                                 │
│  Agent: "Here's the comprehensive analysis..."                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tool Contract

```yaml
name: tool-collaborative
description: Coordinate multiple specialized agents on a task

parameters:
  task:
    type: string
    description: The task for agents to collaborate on
    required: true

  agents:
    type: array
    description: Agent configurations to involve
    required: true
    items:
      type: object
      properties:
        name:
          type: string
          description: Identifier for this agent
        role:
          type: string
          description: The specialist role (e.g., "security", "performance")
        config:
          type: object
          description: Agent config (providers, tools, system prompt)
        focus:
          type: string
          description: Specific focus area for this agent

  mode:
    type: string
    enum: [parallel, sequential, hierarchical]
    default: parallel
    description: How agents execute

  synthesis:
    type: string
    enum: [coordinator, vote, merge, best_of]
    default: coordinator
    description: How to combine agent outputs

  coordinator_config:
    type: object
    description: Config for synthesis coordinator (if synthesis=coordinator)

  context:
    type: object
    description: Shared context passed to all agents

returns:
  type: object
  properties:
    result:
      type: string
      description: Synthesized output
    contributions:
      type: array
      description: Individual agent contributions
    consensus:
      type: object
      description: Agreement/disagreement analysis
    metadata:
      type: object
      description: Execution stats (agents, timing, tokens)
```

---

## Implementation

```python
# tool.py

from amplifier_core import AmplifierSession
import asyncio
from dataclasses import dataclass

@dataclass
class Contribution:
    agent: str
    role: str
    response: str
    confidence: float | None = None
    tokens_used: int = 0


class CollaborativeTool:
    """Tool for multi-agent collaboration."""

    name = "tool-collaborative"

    async def execute(
        self,
        task: str,
        agents: list[dict],
        mode: str = "parallel",
        synthesis: str = "coordinator",
        coordinator_config: dict | None = None,
        context: dict | None = None
    ) -> dict:
        """Execute collaborative task."""

        # Phase 1: Execute agents
        if mode == "parallel":
            contributions = await self._parallel_execution(task, agents, context)
        elif mode == "sequential":
            contributions = await self._sequential_execution(task, agents, context)
        else:
            contributions = await self._hierarchical_execution(task, agents, context)

        # Phase 2: Synthesize
        if synthesis == "coordinator":
            result = await self._coordinator_synthesis(
                task, contributions, coordinator_config
            )
        elif synthesis == "vote":
            result = await self._voting_synthesis(contributions)
        elif synthesis == "merge":
            result = await self._merge_synthesis(contributions)
        else:  # best_of
            result = await self._best_of_synthesis(contributions)

        return {
            "result": result,
            "contributions": [c.__dict__ for c in contributions],
            "consensus": self._analyze_consensus(contributions),
            "metadata": {
                "agents_count": len(agents),
                "mode": mode,
                "synthesis": synthesis,
                "total_tokens": sum(c.tokens_used for c in contributions)
            }
        }

    async def _parallel_execution(
        self,
        task: str,
        agents: list[dict],
        context: dict | None
    ) -> list[Contribution]:
        """Execute all agents in parallel."""

        async def run_agent(agent_def: dict) -> Contribution:
            config = self._build_agent_config(agent_def)

            prompt = self._build_agent_prompt(
                task=task,
                role=agent_def.get("role", "specialist"),
                focus=agent_def.get("focus"),
                context=context
            )

            async with AmplifierSession(config=config) as session:
                response = await session.execute(prompt)

                return Contribution(
                    agent=agent_def["name"],
                    role=agent_def.get("role", "specialist"),
                    response=response.text,
                    tokens_used=response.usage.total_tokens
                )

        contributions = await asyncio.gather(*[
            run_agent(agent) for agent in agents
        ])

        return list(contributions)

    async def _sequential_execution(
        self,
        task: str,
        agents: list[dict],
        context: dict | None
    ) -> list[Contribution]:
        """Execute agents sequentially, each seeing previous results."""

        contributions = []
        accumulated_context = context or {}

        for agent_def in agents:
            config = self._build_agent_config(agent_def)

            # Include previous contributions in context
            prompt = self._build_agent_prompt(
                task=task,
                role=agent_def.get("role", "specialist"),
                focus=agent_def.get("focus"),
                context=accumulated_context,
                previous_contributions=contributions
            )

            async with AmplifierSession(config=config) as session:
                response = await session.execute(prompt)

                contribution = Contribution(
                    agent=agent_def["name"],
                    role=agent_def.get("role", "specialist"),
                    response=response.text,
                    tokens_used=response.usage.total_tokens
                )
                contributions.append(contribution)

                # Add to accumulated context
                accumulated_context[f"{agent_def['name']}_analysis"] = response.text

        return contributions

    async def _hierarchical_execution(
        self,
        task: str,
        agents: list[dict],
        context: dict | None
    ) -> list[Contribution]:
        """Coordinator delegates subtasks to specialists."""

        # First agent is coordinator
        coordinator = agents[0]
        specialists = agents[1:]

        # Coordinator decomposes task
        decomposition = await self._decompose_task(
            task, coordinator, specialists, context
        )

        # Specialists execute their subtasks in parallel
        specialist_tasks = []
        for assignment in decomposition["assignments"]:
            specialist = next(
                (s for s in specialists if s["name"] == assignment["agent"]),
                None
            )
            if specialist:
                specialist_tasks.append(
                    self._run_specialist(assignment["subtask"], specialist, context)
                )

        specialist_contributions = await asyncio.gather(*specialist_tasks)

        # Coordinator contribution includes orchestration
        coordinator_contribution = Contribution(
            agent=coordinator["name"],
            role="coordinator",
            response=decomposition["plan"],
            tokens_used=decomposition["tokens_used"]
        )

        return [coordinator_contribution] + list(specialist_contributions)

    async def _coordinator_synthesis(
        self,
        task: str,
        contributions: list[Contribution],
        coordinator_config: dict | None
    ) -> str:
        """Use coordinator agent to synthesize contributions."""

        config = coordinator_config or self._default_coordinator_config()

        prompt = f"""
You are synthesizing contributions from multiple specialist agents.

## Original Task
{task}

## Specialist Contributions

{self._format_contributions(contributions)}

## Your Task
Synthesize these perspectives into a unified, coherent response:
1. Identify key agreements across specialists
2. Highlight important disagreements or tensions
3. Resolve conflicts with reasoned judgment
4. Provide actionable conclusions
5. Note any gaps or areas needing further analysis

Provide a comprehensive synthesis that leverages the strengths of each perspective.
"""

        async with AmplifierSession(config=config) as session:
            response = await session.execute(prompt)
            return response.text

    async def _voting_synthesis(
        self,
        contributions: list[Contribution]
    ) -> str:
        """Synthesize by identifying consensus and majority views."""

        # Extract key points from each contribution
        # Identify overlapping conclusions
        # Weight by confidence if available

        consensus_points = self._extract_consensus(contributions)
        disagreements = self._extract_disagreements(contributions)

        return f"""
## Consensus ({len(consensus_points)} points)
{self._format_points(consensus_points)}

## Divergent Views
{self._format_disagreements(disagreements)}

## Majority Conclusion
{self._determine_majority(contributions)}
"""

    async def _merge_synthesis(
        self,
        contributions: list[Contribution]
    ) -> str:
        """Simple merge of all contributions."""

        sections = []
        for c in contributions:
            sections.append(f"### {c.agent} ({c.role})\n\n{c.response}")

        return "\n\n---\n\n".join(sections)

    def _build_agent_prompt(
        self,
        task: str,
        role: str,
        focus: str | None,
        context: dict | None,
        previous_contributions: list[Contribution] | None = None
    ) -> str:
        """Build prompt for specialist agent."""

        prompt_parts = [f"## Task\n{task}"]

        if focus:
            prompt_parts.append(f"## Your Focus\n{focus}")

        if context:
            prompt_parts.append(f"## Context\n{self._format_context(context)}")

        if previous_contributions:
            prompt_parts.append(
                f"## Previous Analysis\n{self._format_contributions(previous_contributions)}"
            )

        prompt_parts.append(f"""
## Instructions
You are a {role} specialist. Analyze the task from your expert perspective.
Provide specific, actionable insights based on your expertise.
Be thorough but focused on your domain.
""")

        return "\n\n".join(prompt_parts)

    def _format_contributions(self, contributions: list[Contribution]) -> str:
        """Format contributions for synthesis prompt."""

        formatted = []
        for c in contributions:
            formatted.append(f"### {c.agent} ({c.role})\n{c.response}")
        return "\n\n".join(formatted)

    def _analyze_consensus(self, contributions: list[Contribution]) -> dict:
        """Analyze agreement/disagreement across contributions."""

        # Simple heuristic - could be enhanced with AI analysis
        return {
            "agents_count": len(contributions),
            "has_consensus": len(contributions) <= 1,  # Placeholder
            "agreement_score": None,  # Could compute similarity
            "key_agreements": [],
            "key_disagreements": []
        }

    def _default_coordinator_config(self) -> dict:
        """Default config for synthesis coordinator."""

        return {
            "session": {
                "orchestrator": "loop-basic",
                "context": "context-simple"
            },
            "providers": [{
                "module": "provider-anthropic",
                "source": "git+https://github.com/microsoft/amplifier-module-provider-anthropic@main",
                "config": {
                    "model": "claude-sonnet-4-5",
                    "temperature": 0.3  # Balanced for synthesis
                }
            }]
        }
```

---

## Collaboration Modes

### Parallel Mode

All agents execute simultaneously, then results synthesized.

```yaml
# Best for: Independent specialist perspectives
mode: parallel

agents:
  - name: security-reviewer
    role: security
    focus: "vulnerabilities, auth, injection"

  - name: performance-reviewer
    role: performance
    focus: "complexity, caching, queries"

  - name: maintainability-reviewer
    role: maintainability
    focus: "patterns, coupling, readability"

synthesis: coordinator
```

### Sequential Mode

Agents execute in order, each seeing previous contributions.

```yaml
# Best for: Building analysis, refinement chains
mode: sequential

agents:
  - name: analyzer
    role: analysis
    focus: "understand the problem"

  - name: designer
    role: design
    focus: "propose solution based on analysis"

  - name: critic
    role: critique
    focus: "evaluate design, find gaps"

  - name: refiner
    role: refinement
    focus: "improve design based on critique"

synthesis: merge  # Keep the chain visible
```

### Hierarchical Mode

First agent coordinates, delegates to specialists.

```yaml
# Best for: Complex tasks needing decomposition
mode: hierarchical

agents:
  - name: coordinator
    role: coordinator
    focus: "decompose task, assign to specialists, synthesize"

  - name: backend-specialist
    role: backend
    focus: "API, database, services"

  - name: frontend-specialist
    role: frontend
    focus: "UI, state, components"

  - name: infra-specialist
    role: infrastructure
    focus: "deployment, scaling, monitoring"

synthesis: coordinator  # Coordinator synthesizes at end
```

---

## Synthesis Strategies

| Strategy | Description | Best For |
|----------|-------------|----------|
| `coordinator` | Dedicated agent synthesizes | Complex integration, nuanced judgment |
| `vote` | Identify consensus/majority | Clear yes/no decisions |
| `merge` | Concatenate all contributions | Preserving all perspectives |
| `best_of` | Select highest quality/confidence | When one answer needed |

---

## Usage Examples

### PR Review with Multiple Perspectives

```python
result = await tool_collaborative.execute(
    task=f"Review PR #{pr_number}: {pr_description}\n\nDiff:\n{diff}",
    agents=[
        {
            "name": "security-reviewer",
            "role": "security",
            "config": {
                "providers": [{
                    "module": "provider-anthropic",
                    "config": {"temperature": 0.1}
                }]
            },
            "focus": "Security vulnerabilities, auth issues, injection risks"
        },
        {
            "name": "quality-reviewer",
            "role": "code-quality",
            "config": {
                "providers": [{
                    "module": "provider-anthropic",
                    "config": {"temperature": 0.2}
                }]
            },
            "focus": "Code patterns, maintainability, SOLID principles"
        },
        {
            "name": "test-reviewer",
            "role": "testing",
            "focus": "Test coverage, edge cases, test quality"
        }
    ],
    mode="parallel",
    synthesis="coordinator"
)
```

### Architecture Decision with Debate

```python
result = await tool_collaborative.execute(
    task="Should we use microservices or monolith for the new payment system?",
    agents=[
        {
            "name": "microservices-advocate",
            "role": "advocate",
            "focus": "Argue FOR microservices architecture"
        },
        {
            "name": "monolith-advocate",
            "role": "advocate",
            "focus": "Argue FOR monolith architecture"
        },
        {
            "name": "pragmatist",
            "role": "evaluator",
            "focus": "Evaluate both arguments, consider team context"
        }
    ],
    mode="sequential",  # Each sees previous arguments
    synthesis="coordinator",
    coordinator_config={
        "providers": [{
            "module": "provider-anthropic",
            "config": {
                "model": "claude-opus-4",  # Strong reasoning for decision
                "temperature": 0.3
            }
        }]
    },
    context={
        "team_size": 8,
        "timeline": "6 months",
        "scale_requirements": "10k requests/day initially"
    }
)
```

---

## Events

| Event | Description | Data |
|-------|-------------|------|
| `tool:collaborative:start` | Collaboration started | task, agents, mode |
| `tool:collaborative:agent:start` | Agent execution started | agent, role |
| `tool:collaborative:agent:complete` | Agent finished | agent, tokens |
| `tool:collaborative:synthesis:start` | Synthesis started | strategy |
| `tool:collaborative:complete` | Collaboration complete | agents_count, total_tokens |

---

## Configuration

```yaml
tools:
  - module: tool-collaborative
    source: git+https://github.com/microsoft/amplifier-module-tool-collaborative@main
    config:
      # Defaults
      default_mode: parallel
      default_synthesis: coordinator

      # Limits
      max_agents: 5
      max_parallel: 3
      agent_timeout: 300s

      # Coordinator defaults
      coordinator:
        model: claude-sonnet-4-5
        temperature: 0.3
```

---

## Comparison with Recipes

| Aspect | tool-collaborative | recipes |
|--------|-------------------|---------|
| **Pattern** | Parallel specialists | Sequential steps |
| **Agents** | Multiple simultaneous | One per step |
| **Context flow** | Shared to all | Accumulated step-by-step |
| **Best for** | Multiple perspectives | Linear workflows |
| **Synthesis** | Built-in coordinator | Final step handles it |

Both are tools that spawn sub-agents. Use recipes for workflows, collaborative for multi-perspective analysis.

---

## Dependencies

### Required
- amplifier-core (AmplifierSession for sub-agents)
- asyncio (parallel execution)

### Optional
- None

---

## Open Questions

1. **Confidence scoring**: Should agents report confidence levels?
2. **Disagreement resolution**: Automated resolution or surface to user?
3. **Cost optimization**: Route simpler subtasks to cheaper models?
4. **Caching**: Cache specialist results for similar tasks?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification (converted from orchestrator to tool) |
