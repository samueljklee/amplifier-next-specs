# orchestrator-loop-iterative-refinement

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-module-loop-iterative-refinement`

## Overview

An orchestrator that manages iterative refinement workflows with structured feedback loops, version tracking, and convergence detection. Enables design sprint-style iteration where each cycle produces a versioned artifact, captures feedback, and tracks what changed and why.

### Value Proposition

| Without | With |
|---------|------|
| Manual iteration tracking, lost context between versions | Automatic version tracking with delta visualization |
| "What changed since last time?" requires memory | Side-by-side diffs with rationale for each change |
| Feedback scattered across messages | Structured feedback correlation to specific changes |
| No convergence signal | Automatic detection when iterations stabilize |

### Use Cases

1. **Design system component iteration**: Designer reviews and refines component specs through multiple rounds
2. **API design refinement**: Architecture evolves based on stakeholder feedback
3. **Feature specification**: PM iterates on PRD with eng/design input
4. **Code refactoring**: Large refactors broken into reviewable iterations
5. **Documentation writing**: Technical docs refined with SME feedback

---

## Contract

### Orchestrator Interface

```python
class IterativeRefinementOrchestrator:
    """
    Orchestrator that manages iterative refinement with feedback tracking.
    
    Each iteration:
    1. Generate or refine artifact
    2. Checkpoint version
    3. Present for review (optional pause)
    4. Capture feedback
    5. Assess convergence
    6. Continue or conclude
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
        Execute iterative refinement loop.
        
        Args:
            prompt: Initial user request
            context: Conversation context
            providers: Available LLM providers
            tools: Available tools
            hooks: Hook registry for events
            coordinator: Module coordinator for capabilities
            
        Returns:
            Final refined artifact with evolution history
        """
```

### Configuration Schema

```toml
[[orchestrators]]
module = "loop-iterative-refinement"
config = {
  # Iteration control
  max_iterations = 10                    # Maximum refinement cycles
  min_iterations = 1                     # Minimum cycles before allowing convergence
  
  # Convergence detection
  convergence_threshold = 0.9            # 0.0-1.0: fraction of feedback addressed
  auto_converge = true                   # Stop when threshold reached
  
  # Review flow
  pause_for_review = true                # Pause between iterations for human review
  review_prompt = "Review iteration {n}. Provide feedback or approve:"
  
  # Version tracking
  track_deltas = true                    # Calculate and show diffs between versions
  delta_format = "semantic"              # "semantic" | "line-by-line" | "summary"
  track_rationale = true                 # Capture "why" for each change
  
  # Feedback correlation
  correlate_changes_to_feedback = true   # Map changes back to feedback items
  feedback_scoring = true                # Score how well each feedback was addressed
  
  # Output control
  show_evolution_history = true          # Include complete history in final output
  checkpoint_storage = "memory"          # "memory" | "file" | "database"
}
```

### Events Emitted

| Event | When | Data |
|-------|------|------|
| `orchestrator:iteration:start` | Iteration begins | iteration_number, total_iterations |
| `orchestrator:iteration:complete` | Iteration done | iteration_number, artifact_id, changes_count |
| `orchestrator:review:pause` | Paused for review | iteration_number, artifact_preview |
| `orchestrator:review:resume` | Review complete | iteration_number, feedback_items |
| `orchestrator:convergence:detected` | Convergence reached | convergence_score, iterations_completed |
| `orchestrator:refinement:complete` | All iterations done | final_artifact_id, total_iterations, evolution_summary |

---

## Architecture

### Component Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│           orchestrator-loop-iterative-refinement                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────┐    ┌────────────────┐    ┌───────────────┐  │
│  │   Iteration    │───▶│    Version     │───▶│  Convergence  │  │
│  │   Controller   │    │    Tracker     │    │   Detector    │  │
│  └────────┬───────┘    └────────┬───────┘    └───────┬───────┘  │
│           │                     │                    │           │
│           │                     │                    │           │
│  ┌────────▼───────┐    ┌───────▼────────┐    ┌──────▼────────┐  │
│  │    Feedback    │    │     Delta      │    │   Evolution   │  │
│  │   Correlator   │    │   Calculator   │    │   Historian   │  │
│  └────────────────┘    └────────────────┘    └───────────────┘  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Internal Components

#### 1. Iteration Controller

Manages the refinement loop lifecycle:

```python
@dataclass
class Iteration:
    number: int
    artifact: str                    # Current version of work
    artifact_id: str                 # UUID for this version
    feedback: list[FeedbackItem]     # Feedback to address
    changes: list[Change]            # Changes made this iteration
    timestamp: datetime
    convergence_score: float         # 0.0-1.0

@dataclass
class FeedbackItem:
    id: str
    source: str                      # "user" | "agent" | "automated"
    content: str
    priority: str                    # "must" | "should" | "could"
    addressed: bool
    iteration_addressed: int | None

@dataclass
class Change:
    description: str
    rationale: str
    addresses_feedback: list[str]    # Feedback IDs
    diff: str | None

class IterationController:
    """Manages iteration lifecycle and flow control."""
    
    def __init__(self, config: IterativeRefinementConfig):
        self.config = config
        self.iterations: list[Iteration] = []
        self.current_iteration = 0
    
    async def should_continue(self) -> bool:
        """Determine if another iteration is needed."""
        if self.current_iteration >= self.config.max_iterations:
            return False
        
        if self.current_iteration < self.config.min_iterations:
            return True
        
        if self.config.auto_converge:
            score = await self.convergence_detector.check()
            if score >= self.config.convergence_threshold:
                return False
        
        return True
    
    async def execute_iteration(self, prompt: str, context: ContextManager) -> Iteration:
        """Execute one refinement iteration."""
        self.current_iteration += 1
        
        # Get pending feedback
        pending_feedback = self._get_pending_feedback()
        
        # Build iteration prompt
        iteration_prompt = self._build_iteration_prompt(
            iteration_num=self.current_iteration,
            base_prompt=prompt,
            feedback=pending_feedback,
            previous_artifact=self._get_latest_artifact()
        )
        
        # Execute with standard loop (delegate to loop-basic)
        result = await self._execute_standard_loop(iteration_prompt, context)
        
        # Create iteration record
        iteration = Iteration(
            number=self.current_iteration,
            artifact=result,
            artifact_id=str(uuid.uuid4()),
            feedback=pending_feedback,
            changes=[],  # Extracted in post-processing
            timestamp=datetime.now(),
            convergence_score=0.0
        )
        
        # Post-process: extract changes, calculate deltas
        iteration = await self._post_process_iteration(iteration)
        
        self.iterations.append(iteration)
        return iteration
    
    def _build_iteration_prompt(
        self, 
        iteration_num: int, 
        base_prompt: str,
        feedback: list[FeedbackItem],
        previous_artifact: str | None
    ) -> str:
        """Build prompt for this iteration."""
        if iteration_num == 1:
            return base_prompt
        
        prompt_parts = [
            f"# Iteration {iteration_num}",
            "",
            "## Previous Version",
            previous_artifact,
            "",
            "## Feedback to Address",
        ]
        
        for item in feedback:
            priority_marker = {
                "must": "🔴 MUST FIX",
                "should": "🟡 SHOULD FIX",
                "could": "🟢 NICE TO HAVE"
            }[item.priority]
            
            prompt_parts.append(f"{priority_marker}: {item.content}")
        
        prompt_parts.extend([
            "",
            "## Task",
            "Refine the previous version based on the feedback above.",
            "For each change, explain WHY you made it and which feedback it addresses.",
            "",
            base_prompt
        ])
        
        return "\n".join(prompt_parts)
```

#### 2. Version Tracker

Tracks artifact versions and changes:

```python
class VersionTracker:
    """Tracks versions and computes deltas between iterations."""
    
    def __init__(self):
        self.versions: dict[str, str] = {}  # artifact_id -> content
    
    def store_version(self, artifact_id: str, content: str):
        """Store a version."""
        self.versions[artifact_id] = content
    
    async def compute_delta(
        self, 
        from_artifact_id: str, 
        to_artifact_id: str,
        format: str = "semantic"
    ) -> Delta:
        """Compute changes between versions."""
        from_content = self.versions[from_artifact_id]
        to_content = self.versions[to_artifact_id]
        
        if format == "semantic":
            return await self._semantic_diff(from_content, to_content)
        elif format == "line-by-line":
            return self._line_diff(from_content, to_content)
        else:  # summary
            return self._summary_diff(from_content, to_content)
    
    async def _semantic_diff(self, old: str, new: str) -> SemanticDelta:
        """Use LLM to identify semantic changes."""
        # Ask LLM to identify what changed conceptually
        prompt = f"""
        Compare these two versions and identify semantic changes:
        
        VERSION A:
        {old}
        
        VERSION B:
        {new}
        
        List changes as: [Added|Modified|Removed] - <description>
        """
        # Returns structured list of semantic changes
        ...
    
    def _line_diff(self, old: str, new: str) -> LineDelta:
        """Traditional unified diff."""
        import difflib
        diff = difflib.unified_diff(
            old.splitlines(keepends=True),
            new.splitlines(keepends=True),
            lineterm=''
        )
        return LineDelta(lines=list(diff))
```

#### 3. Feedback Correlator

Links changes back to feedback items:

```python
class FeedbackCorrelator:
    """Correlates changes to feedback items and scores addressing."""
    
    async def correlate_changes(
        self,
        changes: list[Change],
        feedback: list[FeedbackItem]
    ) -> CorrelationResult:
        """
        Determine which changes address which feedback.
        Uses LLM to understand if a change addresses feedback.
        """
        correlations = []
        
        for feedback_item in feedback:
            addressing_changes = []
            
            for change in changes:
                # Check if change explicitly mentions feedback
                if feedback_item.id in change.addresses_feedback:
                    addressing_changes.append(change)
                    continue
                
                # Use LLM to infer if change addresses feedback
                is_addressing = await self._infer_correlation(
                    feedback_item, 
                    change
                )
                if is_addressing:
                    addressing_changes.append(change)
            
            correlation = FeedbackCorrelation(
                feedback_item=feedback_item,
                changes=addressing_changes,
                fully_addressed=len(addressing_changes) > 0,
                quality_score=self._score_addressing_quality(
                    feedback_item, 
                    addressing_changes
                )
            )
            correlations.append(correlation)
        
        return CorrelationResult(correlations=correlations)
    
    async def _infer_correlation(
        self,
        feedback: FeedbackItem,
        change: Change
    ) -> bool:
        """Use LLM to determine if change addresses feedback."""
        prompt = f"""
        Feedback: {feedback.content}
        Change: {change.description} - {change.rationale}
        
        Does this change address the feedback? (yes/no)
        """
        # Returns boolean
        ...
    
    def _score_addressing_quality(
        self,
        feedback: FeedbackItem,
        changes: list[Change]
    ) -> float:
        """Score 0.0-1.0: how well was feedback addressed."""
        if not changes:
            return 0.0
        
        # Simple heuristic: more changes = better (could be LLM-based)
        return min(1.0, len(changes) * 0.5)
```

#### 4. Convergence Detector

Determines when iteration can stop:

```python
class ConvergenceDetector:
    """Detects when iterations have converged."""
    
    def __init__(self, threshold: float = 0.9):
        self.threshold = threshold
    
    async def check_convergence(
        self,
        iterations: list[Iteration],
        all_feedback: list[FeedbackItem]
    ) -> ConvergenceScore:
        """
        Check if iteration has converged.
        
        Convergence signals:
        1. High percentage of feedback addressed
        2. Small deltas between recent iterations
        3. No critical feedback outstanding
        """
        if len(iterations) < 2:
            return ConvergenceScore(score=0.0, reason="Too few iterations")
        
        # Signal 1: Feedback addressed
        feedback_score = self._feedback_addressed_score(all_feedback)
        
        # Signal 2: Delta shrinking
        delta_score = await self._delta_shrinking_score(iterations[-3:])
        
        # Signal 3: No blockers
        blocker_score = self._no_blockers_score(all_feedback)
        
        # Weighted average
        overall_score = (
            feedback_score * 0.5 +
            delta_score * 0.3 +
            blocker_score * 0.2
        )
        
        converged = overall_score >= self.threshold
        
        return ConvergenceScore(
            score=overall_score,
            converged=converged,
            reason=self._explain_score(feedback_score, delta_score, blocker_score)
        )
    
    def _feedback_addressed_score(self, feedback: list[FeedbackItem]) -> float:
        """What fraction of feedback has been addressed."""
        if not feedback:
            return 1.0
        
        addressed = sum(1 for f in feedback if f.addressed)
        return addressed / len(feedback)
    
    async def _delta_shrinking_score(self, recent_iterations: list[Iteration]) -> float:
        """Are changes getting smaller (convergence signal)?"""
        if len(recent_iterations) < 2:
            return 0.0
        
        # Compare delta sizes (smaller = converging)
        deltas = [len(it.changes) for it in recent_iterations]
        
        # If deltas are decreasing, score high
        if all(deltas[i] >= deltas[i+1] for i in range(len(deltas)-1)):
            return 1.0
        
        # Otherwise score by how small the latest delta is
        latest_delta = deltas[-1]
        return max(0.0, 1.0 - (latest_delta / 10))  # Assume 10+ changes = big delta
```

### Data Flow

```
User Prompt
    │
    ▼
┌─────────────────────┐
│ Iteration 1: Create │
│ - Generate initial  │
│ - Store version v1  │
└──────────┬──────────┘
           │
           ▼
    ┌─────────────┐
    │ Pause for   │─── Optional: Wait for human review
    │   Review    │
    └──────┬──────┘
           │
           ▼ (Feedback captured)
┌─────────────────────┐
│ Iteration 2: Refine │
│ - Apply feedback    │
│ - Compute delta     │
│ - Store version v2  │
│ - Correlate changes │
└──────────┬──────────┘
           │
           ▼
    ┌─────────────────┐
    │ Check           │
    │ Convergence     │
    └────┬───────┬────┘
         │       │
    No   │       │ Yes
         │       │
         ▼       ▼
    [Continue]  [Done]
                 │
                 ▼
         ┌───────────────┐
         │ Generate      │
         │ Evolution     │
         │ Report        │
         └───────────────┘
```

---

## Configuration

```yaml
# Full configuration example
orchestrators:
  - module: loop-iterative-refinement
    config:
      # Iteration control
      max_iterations: 10
      min_iterations: 1
      
      # Convergence
      convergence_threshold: 0.9
      auto_converge: true
      convergence_signals:
        - feedback_addressed
        - delta_shrinking
        - no_critical_issues
      
      # Review workflow
      pause_for_review: true
      review_prompt: |
        ## Iteration {iteration_number} Review
        
        {artifact_preview}
        
        Please provide feedback:
        - What's working well?
        - What needs improvement?
        - Any blockers or must-fix issues?
        
        Type 'approve' when satisfied.
      
      # Version tracking
      track_deltas: true
      delta_format: semantic              # semantic | line-by-line | summary
      track_rationale: true
      store_intermediate_versions: true
      
      # Feedback management
      correlate_changes_to_feedback: true
      feedback_scoring: true
      feedback_priorities:
        must: 1.0                         # Weight for "must fix" feedback
        should: 0.7                       # Weight for "should fix"
        could: 0.3                        # Weight for "nice to have"
      
      # Output formatting
      show_evolution_history: true
      evolution_format: detailed          # detailed | summary | minimal
      include_all_versions: false         # Include full text of all versions
      
      # Storage
      checkpoint_storage: memory          # memory | file | database
      checkpoint_path: .amplifier/iterations/
```

---

## Examples

### Example 1: Design Component Iteration

```yaml
# Initial prompt
"Design a data table component for our design system"

# Iteration 1: Initial design
Agent generates:
"""
# DataTable Component

## Props
- data: array of objects
- columns: column definitions
- onRowClick: callback

## Variants
- default: standard table
- compact: dense spacing
"""

# Pause for review
User feedback:
"Needs sorting, filtering, and column resize"

# Iteration 2: Add features
Agent refines:
"""
# DataTable Component (v2)

## Props
- data: array of objects
- columns: column definitions  
- onRowClick: callback
+ sortable: boolean
+ filterable: boolean
+ resizable: boolean

## Features
+ Sorting: Click column header to sort
+ Filtering: Filter panel above table
+ Resize: Drag column borders
"""

Changes:
- Added: sortable, filterable, resizable props [Addresses: "needs sorting, filtering, column resize"]
- Added: Sorting feature with header click [Addresses: "needs sorting"]
- Added: Filtering feature with filter panel [Addresses: "needs filtering"]
- Added: Resize feature with drag borders [Addresses: "needs column resize"]

# Pause for review
User feedback:
"Sorting looks good, but filter UX unclear"

# Iteration 3: Refine filtering
Agent refines:
"""
[Previous content]

## Filtering Details
- Filter button in header opens panel
- Panel shows filter inputs per column
- Presets: "Clear All", "Save Filter"
- Visual: Shows active filter count badge
"""

Changes:
- Modified: Filter feature with detailed UX [Addresses: "filter UX unclear"]
- Added: Filter presets for common actions
- Added: Active filter indicator

# Convergence check
Feedback addressed: 100% (3/3 items)
Delta size: Small (1 feature refined)
Critical issues: None
Convergence score: 0.95

→ Iteration complete!

# Final output
"""
# DataTable Component - Evolution Report

## Final Specification
[Full spec]

## Iteration History
Iteration 1: Initial design (baseline)
Iteration 2: Added sorting, filtering, resize (+3 features)
Iteration 3: Refined filter UX (+1 detail)

## Feedback Tracking
✓ Needs sorting → Addressed in iteration 2
✓ Needs filtering → Addressed in iteration 2, refined in 3
✓ Needs column resize → Addressed in iteration 2
✓ Filter UX unclear → Addressed in iteration 3

## Design Decisions
- Sorting: Chose click-to-sort (standard pattern)
- Filtering: Chose panel over inline (more space for complex filters)
- Resize: Chose drag borders (familiar to users)
"""
```

### Example 2: API Design Refinement

```python
# Initial prompt
"Design a REST API for user permissions"

# Iteration 1
{
    "artifact": """
    POST /api/v1/users/{id}/permissions
    GET /api/v1/users/{id}/permissions
    DELETE /api/v1/users/{id}/permissions/{permissionId}
    """,
    "iteration": 1
}

# Feedback
[
    {"content": "Need bulk operations", "priority": "should"},
    {"content": "Missing permission inheritance", "priority": "must"}
]

# Iteration 2
{
    "artifact": """
    POST /api/v1/users/{id}/permissions (bulk support via array)
    GET /api/v1/users/{id}/permissions?include=inherited
    DELETE /api/v1/users/{id}/permissions (bulk support)
    POST /api/v1/permissions/inherit
    """,
    "iteration": 2,
    "changes": [
        {
            "description": "Added bulk support to POST/DELETE",
            "addresses_feedback": ["Need bulk operations"]
        },
        {
            "description": "Added ?include=inherited query param",
            "addresses_feedback": ["Missing permission inheritance"]
        },
        {
            "description": "Added /inherit endpoint for inheritance rules",
            "addresses_feedback": ["Missing permission inheritance"]
        }
    ]
}

# Convergence: Score 1.0 (all feedback addressed)
```

---

## Security Considerations

### Data Storage

- **Iteration checkpoints**: May contain sensitive work-in-progress
- **Feedback**: User feedback may contain sensitive context
- **Versioning**: All versions stored until session end

### Access Control

```python
REQUIRED_CAPABILITIES = [
    "orchestrator:execute",      # Execute orchestration loops
    "context:read",              # Read conversation context
    "context:write",             # Write iteration checkpoints
]
```

### Risk Mitigations

| Risk | Mitigation |
|------|------------|
| Large version history consuming memory | Configurable storage backend (memory/file/database) |
| Sensitive iterations persisted | Clear checkpoints on session end |
| Feedback leaking to other users | Feedback scoped to session |

---

## Testing Strategy

### Unit Tests

```python
def test_iteration_controller_respects_max_iterations():
    controller = IterationController(max_iterations=3)
    assert controller.should_continue() == True  # iteration 1
    # ... execute 3 iterations
    assert controller.should_continue() == False  # max reached

def test_convergence_detector_identifies_high_score():
    detector = ConvergenceDetector(threshold=0.9)
    feedback = [
        FeedbackItem(id="1", addressed=True),
        FeedbackItem(id="2", addressed=True),
        FeedbackItem(id="3", addressed=False)
    ]
    score = detector._feedback_addressed_score(feedback)
    assert score == 0.67  # 2/3

def test_feedback_correlator_links_changes():
    correlator = FeedbackCorrelator()
    feedback = FeedbackItem(id="1", content="Add sorting")
    change = Change(
        description="Added sort",
        addresses_feedback=["1"]
    )
    # Should link change to feedback
    ...
```

### Integration Tests

```python
async def test_full_iteration_flow():
    """Test complete iteration cycle."""
    orchestrator = IterativeRefinementOrchestrator(config)
    
    result = await orchestrator.execute(
        prompt="Design a button component",
        context=mock_context
    )
    
    # Should have multiple iterations
    assert len(orchestrator.iterations) > 1
    
    # Should track changes
    assert all(it.changes for it in orchestrator.iterations[1:])
    
    # Should converge
    assert orchestrator.iterations[-1].convergence_score > 0.9
```

---

## Implementation Notes

### Delegation to Standard Loop

The iterative refinement orchestrator **delegates** to the standard `loop-basic` for each iteration's execution. It provides:
- **Pre-processing**: Build iteration-specific prompt with feedback
- **Post-processing**: Extract changes, compute deltas, correlate feedback

This keeps the orchestrator focused on **iteration management** rather than reimplementing execution logic.

### LLM Usage

The orchestrator uses the LLM for:
1. **Semantic diff**: Understanding what changed conceptually
2. **Change extraction**: Parsing change descriptions from agent responses
3. **Feedback correlation**: Inferring if a change addresses feedback

### Performance Considerations

- **Checkpoint storage**: Memory mode is fast but limited; file mode scales better
- **Delta computation**: Semantic diffs require LLM calls (slower but higher quality)
- **Convergence detection**: Runs after each iteration (overhead acceptable)

---

## Dependencies

### Required
- `amplifier-core` - Core session and module infrastructure
- `amplifier-module-loop-basic` - Delegates iteration execution

### Optional
- `difflib` - Line-by-line diffs (stdlib)
- `tiktoken` - Token counting for large artifacts

---

## Open Questions

1. **Feedback format**: Should feedback be structured (schema) or free-form text?
   - Option A: Structured (easier to correlate, requires template)
   - Option B: Free-form (flexible, harder to correlate)

2. **Convergence tuning**: Should convergence be project-configurable?
   - Some projects may want stricter convergence (critical systems)
   - Others may want looser (creative work)

3. **Parallel iterations**: Should we support exploring multiple refinement paths simultaneously?
   - Would enable A/B comparison of different approaches
   - Increases complexity significantly

4. **Human-in-the-loop**: How to best pause for review?
   - Option A: Block orchestrator, wait for user input
   - Option B: Emit event, allow async review submission
   - Option C: Interactive review UI (requires integration)

5. **Version persistence**: Should intermediate versions be saved long-term?
   - Useful for audit trails but increases storage
   - Could offer configurable retention policy

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
