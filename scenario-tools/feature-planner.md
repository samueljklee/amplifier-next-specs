# feature-planner

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-scenario-feature-planner`

## Overview

Collaborative feature planning workflow that transforms requirements into actionable technical specifications. Analyzes codebase architecture, identifies integration points, estimates complexity, and generates implementation plans with tasks.

### Value Proposition

| Without | With |
|---------|------|
| Requirements lost in translation | Structured technical specs |
| Unclear integration points | Automated architecture analysis |
| Inaccurate estimates | Data-driven complexity scoring |
| Scattered planning docs | Integrated planning workflow |

---

## Workflow Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Feature Planning Pipeline                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐      │
│  │ Require- │───▶│ Analysis │───▶│ Design   │───▶│ Plan     │      │
│  │ ments    │    │          │    │          │    │ Output   │      │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘      │
│       │               │               │               │              │
│       ▼               ▼               ▼               ▼              │
│  • PRD/spec       • Codebase     • Components   • Tech spec        │
│  • User stories     analysis     • APIs         • Tasks            │
│  • Constraints    • Integration  • Data models  • Estimates        │
│  • Acceptance       points       • Sequences    • Risks            │
│    criteria       • Dependencies • Trade-offs   • JIRA tickets     │
│                                                                      │
│                        ┌───────────────┐                            │
│                        │ Human Review  │◀── At each stage           │
│                        │ & Iteration   │                            │
│                        └───────────────┘                            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Configuration

```yaml
scenario:
  name: feature-planner
  version: "1.0.0"

  # Input
  input:
    requirements:
      source: markdown              # markdown | jira | notion | confluence
      path: "${REQUIREMENTS_PATH}"

  # Analysis stage
  analysis:
    codebase:
      enabled: true
      tools:
        - tool-codebase-search
        - tool-architecture-map
        - tool-dependency-graph
      scope:
        include:
          - "src/**"
          - "api/**"
        exclude:
          - "**/*.test.*"
          - "node_modules/**"

    patterns:
      enabled: true
      detect:
        - existing_implementations
        - coding_patterns
        - api_conventions
        - data_models

  # Design stage
  design:
    outputs:
      component_diagram: true
      sequence_diagrams: true
      api_spec: true
      data_model: true

    constraints:
      max_components: 10
      prefer_existing_patterns: true
      backwards_compatible: true

  # Planning stage
  planning:
    task_breakdown:
      granularity: medium          # fine | medium | coarse
      include_tests: true
      include_docs: true

    estimation:
      method: complexity_points    # complexity_points | hours | story_points
      confidence_threshold: 0.7

    risk_analysis:
      enabled: true
      categories:
        - technical_risk
        - integration_risk
        - scope_risk
        - dependency_risk

  # Output
  output:
    formats:
      - tech_spec_markdown
      - jira_tickets
      - architecture_diagram

    jira:
      project: "${JIRA_PROJECT}"
      epic_type: Epic
      story_type: Story
      task_type: Task
      create_tickets: false        # false = dry run
```

---

## Pipeline Implementation

```python
class FeaturePlannerScenario:
    """Transform requirements into implementation plans."""

    def __init__(self, config: FeaturePlannerConfig):
        self.config = config
        self.tools = ToolRegistry()

    async def plan(self, requirements_path: str) -> FeaturePlan:
        """Execute feature planning pipeline."""

        # Stage 1: Parse Requirements
        requirements = await self._parse_requirements(requirements_path)

        # Stage 2: Analyze Codebase
        analysis = await self._analyze_codebase(requirements)

        # Stage 3: Design Solution
        design = await self._design_solution(requirements, analysis)

        # Stage 4: Generate Plan
        plan = await self._generate_plan(requirements, analysis, design)

        # Stage 5: Output
        return await self._generate_outputs(plan)

    async def _parse_requirements(self, path: str) -> Requirements:
        """Stage 1: Parse and structure requirements."""

        content = Path(path).read_text()

        # Extract structured requirements using LLM
        parsed = await self._extract_requirements(content)

        return Requirements(
            title=parsed["title"],
            summary=parsed["summary"],
            user_stories=parsed["user_stories"],
            acceptance_criteria=parsed["acceptance_criteria"],
            constraints=parsed["constraints"],
            non_functional=parsed["non_functional_requirements"],
            out_of_scope=parsed["out_of_scope"]
        )

    async def _analyze_codebase(self, requirements: Requirements) -> CodebaseAnalysis:
        """Stage 2: Analyze codebase for implementation context."""

        analysis = CodebaseAnalysis()

        # Search for related code
        search_tool = self.tools.get("tool-codebase-search")

        # Extract key concepts from requirements
        concepts = await self._extract_concepts(requirements)

        for concept in concepts:
            results = await search_tool.execute({
                "query": concept,
                "search_type": "semantic",
                "max_results": 10
            })
            analysis.related_code[concept] = results

        # Get architecture map
        arch_tool = self.tools.get("tool-architecture-map")
        analysis.architecture = await arch_tool.execute({
            "action": "generate",
            "format": "json"
        })

        # Identify integration points
        analysis.integration_points = await self._find_integration_points(
            requirements, analysis.related_code, analysis.architecture
        )

        # Get dependency information
        dep_tool = self.tools.get("tool-dependency-graph")
        analysis.dependencies = await dep_tool.execute({
            "paths": [ip.file_path for ip in analysis.integration_points]
        })

        # Detect patterns
        analysis.patterns = await self._detect_patterns(analysis.related_code)

        return analysis

    async def _find_integration_points(
        self,
        requirements: Requirements,
        related_code: dict,
        architecture: Architecture
    ) -> list[IntegrationPoint]:
        """Identify where new feature integrates with existing code."""

        integration_points = []

        # Analyze with LLM
        prompt = f"""
        Given these requirements:
        {requirements.summary}

        And this existing architecture:
        {json.dumps(architecture.components)}

        Identify integration points where the new feature should connect.
        For each point, specify:
        - Component name
        - File path
        - Type (API, Event, Database, Service)
        - Why this integration is needed
        """

        result = await self._llm_analyze(prompt)

        for point in result["integration_points"]:
            integration_points.append(IntegrationPoint(
                component=point["component"],
                file_path=point["file_path"],
                integration_type=point["type"],
                rationale=point["rationale"],
                existing_code=related_code.get(point["component"], [])
            ))

        return integration_points

    async def _design_solution(
        self,
        requirements: Requirements,
        analysis: CodebaseAnalysis
    ) -> Design:
        """Stage 3: Design the solution."""

        design = Design()

        # Generate component design
        design.components = await self._design_components(requirements, analysis)

        # Generate API design (if needed)
        if self._needs_api(requirements):
            design.api = await self._design_api(requirements, analysis)

        # Generate data model (if needed)
        if self._needs_data_model(requirements):
            design.data_model = await self._design_data_model(requirements, analysis)

        # Generate sequence diagrams
        design.sequences = await self._design_sequences(
            requirements, design.components, analysis
        )

        # Identify trade-offs
        design.trade_offs = await self._identify_trade_offs(design, requirements)

        return design

    async def _design_components(
        self,
        requirements: Requirements,
        analysis: CodebaseAnalysis
    ) -> list[ComponentDesign]:
        """Design new/modified components."""

        # Use existing patterns
        patterns = analysis.patterns

        prompt = f"""
        Design components for this feature:
        {requirements.summary}

        Integration points:
        {json.dumps([ip.to_dict() for ip in analysis.integration_points])}

        Existing patterns to follow:
        {json.dumps(patterns)}

        For each component, specify:
        - Name and purpose
        - Responsibilities
        - Interfaces (methods/APIs)
        - Dependencies
        - New vs modification of existing
        """

        result = await self._llm_design(prompt)

        components = []
        for comp in result["components"]:
            components.append(ComponentDesign(
                name=comp["name"],
                purpose=comp["purpose"],
                responsibilities=comp["responsibilities"],
                interfaces=comp["interfaces"],
                dependencies=comp["dependencies"],
                is_new=comp["is_new"],
                modifies=comp.get("modifies"),
                file_path=comp.get("file_path")
            ))

        return components

    async def _generate_plan(
        self,
        requirements: Requirements,
        analysis: CodebaseAnalysis,
        design: Design
    ) -> ImplementationPlan:
        """Stage 4: Generate implementation plan."""

        plan = ImplementationPlan(
            feature=requirements.title,
            summary=requirements.summary
        )

        # Generate tasks
        plan.tasks = await self._generate_tasks(design, analysis)

        # Estimate complexity
        plan.estimates = await self._estimate_complexity(plan.tasks, analysis)

        # Analyze risks
        plan.risks = await self._analyze_risks(requirements, design, analysis)

        # Determine milestones
        plan.milestones = await self._define_milestones(plan.tasks)

        # Calculate dependencies between tasks
        plan.task_dependencies = await self._calculate_task_dependencies(plan.tasks)

        return plan

    async def _generate_tasks(
        self,
        design: Design,
        analysis: CodebaseAnalysis
    ) -> list[Task]:
        """Break down design into tasks."""

        tasks = []

        # Tasks for each component
        for component in design.components:
            component_tasks = await self._tasks_for_component(component, analysis)
            tasks.extend(component_tasks)

        # API tasks
        if design.api:
            api_tasks = await self._tasks_for_api(design.api)
            tasks.extend(api_tasks)

        # Data model tasks
        if design.data_model:
            data_tasks = await self._tasks_for_data_model(design.data_model)
            tasks.extend(data_tasks)

        # Testing tasks
        if self.config.planning.task_breakdown.include_tests:
            test_tasks = await self._generate_test_tasks(tasks)
            tasks.extend(test_tasks)

        # Documentation tasks
        if self.config.planning.task_breakdown.include_docs:
            doc_tasks = await self._generate_doc_tasks(design)
            tasks.extend(doc_tasks)

        return tasks

    async def _estimate_complexity(
        self,
        tasks: list[Task],
        analysis: CodebaseAnalysis
    ) -> Estimates:
        """Estimate task complexity."""

        estimates = Estimates()

        for task in tasks:
            # Factors affecting complexity
            factors = {
                "code_familiarity": self._calculate_familiarity(
                    task, analysis.patterns
                ),
                "integration_complexity": self._calculate_integration_complexity(
                    task, analysis.integration_points
                ),
                "technical_uncertainty": task.uncertainty,
                "dependency_count": len(task.dependencies)
            }

            # Calculate complexity score
            complexity = self._calculate_complexity_score(factors)

            estimates.task_estimates[task.id] = TaskEstimate(
                complexity_points=complexity,
                confidence=self._estimate_confidence(factors),
                factors=factors
            )

        # Total estimate
        estimates.total = sum(e.complexity_points for e in estimates.task_estimates.values())
        estimates.confidence = min(e.confidence for e in estimates.task_estimates.values())

        return estimates

    async def _analyze_risks(
        self,
        requirements: Requirements,
        design: Design,
        analysis: CodebaseAnalysis
    ) -> list[Risk]:
        """Analyze implementation risks."""

        risks = []

        # Technical risks
        if any(c.is_new for c in design.components):
            risks.append(Risk(
                category="technical_risk",
                title="New component complexity",
                description="New components may have unforeseen complexity",
                likelihood="medium",
                impact="medium",
                mitigation="Prototype new components early, get architecture review"
            ))

        # Integration risks
        if len(analysis.integration_points) > 3:
            risks.append(Risk(
                category="integration_risk",
                title="Multiple integration points",
                description=f"Feature touches {len(analysis.integration_points)} integration points",
                likelihood="medium",
                impact="high",
                mitigation="Create integration tests early, coordinate with component owners"
            ))

        # Scope risks
        if len(requirements.user_stories) > 5:
            risks.append(Risk(
                category="scope_risk",
                title="Large scope",
                description="Feature has many user stories that may expand",
                likelihood="high",
                impact="medium",
                mitigation="Prioritize MVP stories, defer nice-to-haves"
            ))

        # Dependency risks
        external_deps = [d for d in analysis.dependencies if d.is_external]
        if external_deps:
            risks.append(Risk(
                category="dependency_risk",
                title="External dependencies",
                description=f"Feature depends on {len(external_deps)} external services",
                likelihood="medium",
                impact="high",
                mitigation="Create mocks/stubs, have fallback behavior"
            ))

        return risks


class TaskGenerator:
    """Generate implementation tasks from design."""

    async def tasks_for_component(
        self,
        component: ComponentDesign,
        analysis: CodebaseAnalysis,
        granularity: str
    ) -> list[Task]:
        """Generate tasks for a component."""

        tasks = []

        if component.is_new:
            # New component tasks
            tasks.append(Task(
                title=f"Create {component.name} component",
                description=f"Implement {component.purpose}",
                type="implementation",
                component=component.name,
                acceptance_criteria=[
                    f"Implements: {', '.join(component.responsibilities)}",
                    "Follows existing code patterns",
                    "Has unit tests"
                ]
            ))

            if granularity in ("fine", "medium"):
                # Add interface tasks
                for interface in component.interfaces:
                    tasks.append(Task(
                        title=f"Implement {component.name}.{interface.name}",
                        description=interface.description,
                        type="implementation",
                        component=component.name,
                        parent=tasks[0].id
                    ))

        else:
            # Modification tasks
            tasks.append(Task(
                title=f"Modify {component.modifies} for {component.name}",
                description=f"Add {component.purpose} to existing component",
                type="modification",
                component=component.modifies,
                file_path=component.file_path
            ))

        return tasks
```

---

## Output Formats

### Technical Specification (Markdown)

```markdown
# Technical Specification: User Notification Preferences

## Overview

Allow users to customize their notification preferences across email,
push, and in-app channels.

## Requirements Summary

### User Stories
1. As a user, I want to enable/disable notification channels
2. As a user, I want to set quiet hours
3. As a user, I want per-notification-type preferences

### Constraints
- Must be backwards compatible with existing notifications
- Must support GDPR data export

---

## Architecture

### Components

#### 1. NotificationPreferenceService (New)
**Purpose**: Manage user notification preferences
**Location**: `src/services/notification-preferences/`

**Interfaces**:
- `getPreferences(userId): Preferences`
- `updatePreferences(userId, updates): Preferences`
- `checkShouldNotify(userId, type, channel): boolean`

**Dependencies**:
- UserService
- NotificationService (existing, to be modified)

#### 2. NotificationService (Modification)
**Changes**: Add preference checking before sending
**Location**: `src/services/notifications/notification-service.ts`

### Data Model

```sql
CREATE TABLE notification_preferences (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  channel VARCHAR(50),           -- email, push, in_app
  notification_type VARCHAR(100),
  enabled BOOLEAN DEFAULT true,
  quiet_hours_start TIME,
  quiet_hours_end TIME,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);
```

### API

```yaml
/api/v1/users/{userId}/notification-preferences:
  GET:
    summary: Get user notification preferences
    responses:
      200:
        schema: NotificationPreferences

  PATCH:
    summary: Update notification preferences
    body:
      schema: NotificationPreferenceUpdate
```

---

## Implementation Plan

### Milestone 1: Core Preference Management (Week 1)
- [ ] Create NotificationPreferenceService
- [ ] Create database migration
- [ ] Implement CRUD API endpoints
- [ ] Add unit tests

### Milestone 2: Integration (Week 2)
- [ ] Modify NotificationService to check preferences
- [ ] Add quiet hours logic
- [ ] Integration tests

### Milestone 3: UI & Polish (Week 3)
- [ ] Settings UI component
- [ ] Notification preview
- [ ] Documentation

---

## Estimates

| Task | Complexity | Confidence |
|------|------------|------------|
| NotificationPreferenceService | 5 points | 85% |
| Database migration | 2 points | 95% |
| API endpoints | 3 points | 90% |
| NotificationService modification | 3 points | 80% |
| UI components | 5 points | 75% |
| **Total** | **18 points** | **80%** |

---

## Risks

### Medium Risk: NotificationService Modification
**Impact**: Could affect existing notification delivery
**Mitigation**: Feature flag, comprehensive integration tests

### Low Risk: Performance
**Impact**: Additional DB query per notification
**Mitigation**: Cache preferences, batch queries
```

### JIRA Export

```python
# Generated JIRA tickets
{
    "epic": {
        "project": "PROJ",
        "type": "Epic",
        "summary": "User Notification Preferences",
        "description": "Allow users to customize notification preferences..."
    },
    "stories": [
        {
            "type": "Story",
            "summary": "User can enable/disable notification channels",
            "description": "As a user, I want to...",
            "acceptance_criteria": "...",
            "story_points": 5
        },
        # ...
    ],
    "tasks": [
        {
            "type": "Task",
            "summary": "Create NotificationPreferenceService",
            "parent": "PROJ-123",  # Story key
            "description": "Implement service with CRUD operations",
            "complexity_points": 5
        },
        # ...
    ]
}
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `analysis.scope.include` | list | ["src/**"] | Paths to analyze |
| `design.constraints.backwards_compatible` | bool | true | Require backwards compat |
| `planning.task_breakdown.granularity` | string | "medium" | Task detail level |
| `planning.estimation.method` | string | "complexity_points" | Estimation method |
| `output.jira.create_tickets` | bool | false | Create actual tickets |

---

## Dependencies

### Required Tools
- `tool-codebase-search` - Code analysis
- `tool-architecture-map` - Architecture visualization
- `tool-dependency-graph` - Dependency analysis

### Optional Integrations
- JIRA API
- Confluence API
- Notion API

---

## Open Questions

1. **Template library**: Provide templates for common feature types?
2. **Historical data**: Use past estimates to improve accuracy?
3. **Collaboration**: Support multi-user planning sessions?
4. **Integration**: Deeper IDE integration for viewing plans?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
