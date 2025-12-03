# tool-swarm

> **Priority**: P2 (Medium Value)
> **Status**: Draft
> **Module**: `amplifier-module-tool-swarm`

## Overview

A tool for parallel exploration with convergence - spawn multiple variations of the same agent with different parameters, let them explore independently, then converge on the best result. Useful for creative tasks, optimization, and exploring solution spaces.

### Why a Tool, Not an Orchestrator?

Like `tool-collaborative`, swarm exploration is **policy** (when to explore, how many variations, how to converge) not **mechanism**. The agent decides when parallel exploration adds value.

### Value Proposition

| Without | With |
|---------|------|
| Single attempt | Multiple parallel explorations |
| Local optima | Broader solution space coverage |
| One temperature setting | Temperature sweep |
| Manual iteration | Automated best-of-n |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Main Session                                                   │
│                                                                 │
│  Agent: "Let me explore multiple approaches..."                 │
│    │                                                            │
│    └─► tool-swarm.execute(...)                                 │
│          │                                                      │
│          ├─► variation 1 (temp=0.3) ──┐                        │
│          ├─► variation 2 (temp=0.5) ──┼─► parallel             │
│          ├─► variation 3 (temp=0.7) ──┤                        │
│          ├─► variation 4 (temp=0.9) ──┘                        │
│          │                                                      │
│          └─► converge (vote/best/synthesis) ──► return         │
│                                                                 │
│  Agent: "After exploring 4 variations, the best approach..."   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tool Contract

```yaml
name: tool-swarm
description: Parallel exploration with multiple agent variations

parameters:
  task:
    type: string
    description: The task to explore
    required: true

  variations:
    type: integer
    default: 3
    description: Number of parallel variations to spawn

  vary_by:
    type: string
    enum: [temperature, prompt, model, custom]
    default: temperature
    description: What to vary across agents

  temperature_range:
    type: array
    default: [0.3, 0.5, 0.7, 0.9]
    description: Temperatures to use (if vary_by=temperature)

  prompt_variations:
    type: array
    description: Different prompts to try (if vary_by=prompt)

  convergence:
    type: string
    enum: [best_of, vote, synthesis, all]
    default: best_of
    description: How to select/combine results

  evaluation_criteria:
    type: string
    description: Criteria for judging "best" result

  base_config:
    type: object
    description: Base agent configuration (variations modify this)

returns:
  type: object
  properties:
    result:
      type: string
      description: Converged result
    all_results:
      type: array
      description: All variation results
    selection:
      type: object
      description: Why this result was selected
    metadata:
      type: object
      description: Execution stats
```

---

## Implementation

```python
# tool.py

from amplifier_core import AmplifierSession
import asyncio
from dataclasses import dataclass

@dataclass
class ExplorationResult:
    variation_id: int
    parameters: dict
    response: str
    score: float | None = None
    tokens_used: int = 0


class SwarmTool:
    """Tool for parallel exploration with convergence."""

    name = "tool-swarm"

    async def execute(
        self,
        task: str,
        variations: int = 3,
        vary_by: str = "temperature",
        temperature_range: list[float] | None = None,
        prompt_variations: list[str] | None = None,
        convergence: str = "best_of",
        evaluation_criteria: str | None = None,
        base_config: dict | None = None
    ) -> dict:
        """Execute parallel exploration."""

        # Generate variation configs
        variation_configs = self._generate_variations(
            base_config=base_config or self._default_config(),
            vary_by=vary_by,
            count=variations,
            temperature_range=temperature_range or [0.3, 0.5, 0.7, 0.9],
            prompt_variations=prompt_variations
        )

        # Execute all variations in parallel
        results = await self._parallel_explore(task, variation_configs)

        # Converge on result
        if convergence == "best_of":
            final = await self._best_of_convergence(
                results, evaluation_criteria or "quality and correctness"
            )
        elif convergence == "vote":
            final = await self._voting_convergence(results)
        elif convergence == "synthesis":
            final = await self._synthesis_convergence(task, results)
        else:  # all
            final = self._all_results(results)

        return {
            "result": final["result"],
            "all_results": [r.__dict__ for r in results],
            "selection": final.get("selection", {}),
            "metadata": {
                "variations_count": len(results),
                "vary_by": vary_by,
                "convergence": convergence,
                "total_tokens": sum(r.tokens_used for r in results)
            }
        }

    def _generate_variations(
        self,
        base_config: dict,
        vary_by: str,
        count: int,
        temperature_range: list[float],
        prompt_variations: list[str] | None
    ) -> list[dict]:
        """Generate variation configurations."""

        variations = []

        if vary_by == "temperature":
            # Use specified temperatures or generate range
            temps = temperature_range[:count]
            for i, temp in enumerate(temps):
                config = self._deep_copy(base_config)
                config["providers"][0]["config"]["temperature"] = temp
                config["_variation"] = {"id": i, "temperature": temp}
                variations.append(config)

        elif vary_by == "prompt":
            # Use different prompt framings
            for i, prompt_mod in enumerate(prompt_variations[:count]):
                config = self._deep_copy(base_config)
                config["_variation"] = {"id": i, "prompt_modifier": prompt_mod}
                config["_prompt_modifier"] = prompt_mod
                variations.append(config)

        elif vary_by == "model":
            # Use different models
            models = ["claude-sonnet-4-5", "claude-opus-4"]
            for i, model in enumerate(models[:count]):
                config = self._deep_copy(base_config)
                config["providers"][0]["config"]["model"] = model
                config["_variation"] = {"id": i, "model": model}
                variations.append(config)

        return variations

    async def _parallel_explore(
        self,
        task: str,
        variation_configs: list[dict]
    ) -> list[ExplorationResult]:
        """Execute all variations in parallel."""

        async def run_variation(config: dict) -> ExplorationResult:
            variation_info = config.pop("_variation", {"id": 0})
            prompt_modifier = config.pop("_prompt_modifier", None)

            # Modify prompt if needed
            actual_task = task
            if prompt_modifier:
                actual_task = f"{prompt_modifier}\n\n{task}"

            async with AmplifierSession(config=config) as session:
                response = await session.execute(actual_task)

                return ExplorationResult(
                    variation_id=variation_info["id"],
                    parameters=variation_info,
                    response=response.text,
                    tokens_used=response.usage.total_tokens
                )

        results = await asyncio.gather(*[
            run_variation(config) for config in variation_configs
        ])

        return list(results)

    async def _best_of_convergence(
        self,
        results: list[ExplorationResult],
        criteria: str
    ) -> dict:
        """Select best result using evaluator."""

        # Use AI to evaluate and select best
        evaluation_prompt = f"""
Evaluate these {len(results)} responses and select the best one.

Evaluation Criteria: {criteria}

## Responses

{self._format_results_for_eval(results)}

## Your Task
1. Score each response 1-10 based on the criteria
2. Explain your reasoning for each score
3. Select the best response
4. Explain why it's the best

Respond with:
- scores: [list of scores]
- best_index: (0-indexed)
- reasoning: explanation
"""

        async with AmplifierSession(config=self._evaluator_config()) as session:
            eval_response = await session.execute(evaluation_prompt)

        # Parse evaluation (simplified - would use structured output)
        best_idx = self._parse_best_index(eval_response.text, len(results))
        best_result = results[best_idx]

        # Update scores
        scores = self._parse_scores(eval_response.text, len(results))
        for i, result in enumerate(results):
            result.score = scores[i] if i < len(scores) else None

        return {
            "result": best_result.response,
            "selection": {
                "method": "best_of",
                "selected_variation": best_result.variation_id,
                "parameters": best_result.parameters,
                "score": best_result.score,
                "reasoning": eval_response.text
            }
        }

    async def _voting_convergence(
        self,
        results: list[ExplorationResult]
    ) -> dict:
        """Identify consensus across variations."""

        # Extract key points from each result
        # Find overlapping conclusions
        # Return majority view

        voting_prompt = f"""
Analyze these {len(results)} responses for consensus.

{self._format_results_for_eval(results)}

Identify:
1. Points all/most responses agree on
2. Points with disagreement
3. The majority conclusion

Synthesize a response that represents the consensus view.
"""

        async with AmplifierSession(config=self._evaluator_config()) as session:
            consensus = await session.execute(voting_prompt)

        return {
            "result": consensus.text,
            "selection": {
                "method": "vote",
                "variations_analyzed": len(results)
            }
        }

    async def _synthesis_convergence(
        self,
        task: str,
        results: list[ExplorationResult]
    ) -> dict:
        """Synthesize best elements from all variations."""

        synthesis_prompt = f"""
Original task: {task}

You have {len(results)} different approaches to this task:

{self._format_results_for_eval(results)}

Create an improved response that:
1. Takes the best elements from each approach
2. Addresses weaknesses found in individual responses
3. Synthesizes into a coherent, superior result

Provide the synthesized response.
"""

        async with AmplifierSession(config=self._evaluator_config()) as session:
            synthesis = await session.execute(synthesis_prompt)

        return {
            "result": synthesis.text,
            "selection": {
                "method": "synthesis",
                "variations_synthesized": len(results)
            }
        }

    def _format_results_for_eval(self, results: list[ExplorationResult]) -> str:
        """Format results for evaluation prompt."""

        formatted = []
        for r in results:
            formatted.append(
                f"### Variation {r.variation_id} ({r.parameters})\n{r.response}"
            )
        return "\n\n---\n\n".join(formatted)

    def _evaluator_config(self) -> dict:
        """Config for evaluation/synthesis agent."""

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
                    "temperature": 0.2  # Low temp for evaluation
                }
            }]
        }

    def _default_config(self) -> dict:
        """Default base config for variations."""

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
                    "temperature": 0.5
                }
            }]
        }
```

---

## Variation Strategies

### Temperature Sweep

Explore creativity spectrum:

```python
result = await tool_swarm.execute(
    task="Write a tagline for our new AI product",
    vary_by="temperature",
    temperature_range=[0.2, 0.5, 0.8, 1.0],
    convergence="best_of",
    evaluation_criteria="creativity, memorability, brand fit"
)
```

### Prompt Variations

Explore different framings:

```python
result = await tool_swarm.execute(
    task="Explain quantum computing",
    vary_by="prompt",
    prompt_variations=[
        "Explain like I'm a 5-year-old:",
        "Explain for a software engineer:",
        "Explain for a physics PhD:",
        "Explain using only analogies:"
    ],
    convergence="all"  # Keep all for different audiences
)
```

### Model Comparison

Compare model capabilities:

```python
result = await tool_swarm.execute(
    task="Solve this complex reasoning problem...",
    vary_by="model",
    convergence="best_of",
    evaluation_criteria="correctness and reasoning clarity"
)
```

---

## Convergence Strategies

| Strategy | Description | Best For |
|----------|-------------|----------|
| `best_of` | Evaluate and select best | Clear quality criteria |
| `vote` | Find consensus view | Factual tasks |
| `synthesis` | Combine best elements | Creative tasks |
| `all` | Return all results | Need multiple versions |

---

## Usage Examples

### Creative Exploration

```python
# Generate multiple creative options
result = await tool_swarm.execute(
    task="Design a logo concept for an eco-friendly tech startup",
    variations=5,
    vary_by="temperature",
    temperature_range=[0.5, 0.7, 0.9, 1.0, 1.2],
    convergence="all",  # Present all options to user
)

# User picks their favorite from result["all_results"]
```

### Solution Optimization

```python
# Find best solution approach
result = await tool_swarm.execute(
    task=f"Optimize this function for performance:\n\n{code}",
    variations=4,
    vary_by="prompt",
    prompt_variations=[
        "Focus on algorithmic complexity:",
        "Focus on memory efficiency:",
        "Focus on parallelization:",
        "Focus on caching strategies:"
    ],
    convergence="synthesis",  # Combine best optimizations
    evaluation_criteria="performance improvement, code clarity, correctness"
)
```

---

## Events

| Event | Description | Data |
|-------|-------------|------|
| `tool:swarm:start` | Swarm started | task, variations, vary_by |
| `tool:swarm:variation:start` | Variation started | variation_id, parameters |
| `tool:swarm:variation:complete` | Variation finished | variation_id, tokens |
| `tool:swarm:convergence:start` | Convergence started | method |
| `tool:swarm:complete` | Swarm complete | selected, total_tokens |

---

## Configuration

```yaml
tools:
  - module: tool-swarm
    source: git+https://github.com/microsoft/amplifier-module-tool-swarm@main
    config:
      # Defaults
      default_variations: 3
      default_convergence: best_of

      # Limits
      max_variations: 10
      max_parallel: 5
      variation_timeout: 120s

      # Default temperature range
      temperature_range: [0.3, 0.5, 0.7, 0.9]
```

---

## Comparison with tool-collaborative

| Aspect | tool-swarm | tool-collaborative |
|--------|------------|-------------------|
| **Agents** | Same agent, different params | Different specialist agents |
| **Goal** | Explore solution space | Multiple perspectives |
| **Variation** | Parameters (temp, prompt) | Roles and expertise |
| **Best for** | Creative, optimization | Analysis, review |

---

## Open Questions

1. **Adaptive stopping**: Stop early if consensus emerges?
2. **Cost vs exploration**: Auto-tune variation count by task complexity?
3. **Learning**: Track which temperatures/prompts work best for task types?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification (converted from orchestrator to tool) |
