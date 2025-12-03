# codebase-onboarder

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-scenario-codebase-onboarder`

## Overview

Interactive codebase exploration and learning tool for new team members. Generates personalized learning paths, explains architecture, answers questions with full context, and tracks onboarding progress.

### Value Proposition

| Without | With |
|---------|------|
| Weeks to understand codebase | Days with guided exploration |
| Scattered documentation | Unified, contextual explanations |
| Constant interruptions to seniors | Self-service learning |
| No visibility into onboarding progress | Tracked milestones |

---

## Workflow Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Codebase Onboarding Pipeline                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐      │
│  │ Profile  │───▶│ Generate │───▶│ Explore  │───▶│ Assess   │      │
│  │ Setup    │    │ Path     │    │ & Learn  │    │ Progress │      │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘      │
│       │               │               │               │              │
│       ▼               ▼               ▼               ▼              │
│  • Role           • Learning      • Interactive   • Quiz           │
│  • Experience       modules        Q&A           • Practical       │
│  • Focus areas    • Milestones   • Code walks     challenges      │
│  • Time budget    • Resources    • Architecture  • Knowledge       │
│                                    explanations    check           │
│                                                                      │
│                    ┌───────────────┐                                │
│                    │ Progress      │◀── Throughout                  │
│                    │ Tracking      │                                │
│                    └───────────────┘                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Configuration

```yaml
scenario:
  name: codebase-onboarder
  version: "1.0.0"

  # User profile
  profile:
    collect:
      - role                         # frontend, backend, fullstack, etc.
      - experience_level             # junior, mid, senior
      - prior_technologies           # Familiar languages/frameworks
      - focus_areas                  # What they'll work on
      - time_budget                  # Hours per day for learning

  # Content generation
  content:
    # Documentation sources
    sources:
      readme_files: true
      inline_docs: true
      architecture_docs: true
      adr_files: true
      wiki: false
      confluence: false

    # Learning modules to generate
    modules:
      architecture_overview:
        priority: 1
        estimated_time: 2h
      core_concepts:
        priority: 2
        estimated_time: 3h
      development_workflow:
        priority: 3
        estimated_time: 1h
      key_components:
        priority: 4
        estimated_time: 4h
      common_patterns:
        priority: 5
        estimated_time: 2h
      testing_practices:
        priority: 6
        estimated_time: 1h

  # Interactive features
  interactive:
    code_walkthrough: true
    architecture_diagrams: true
    live_questions: true
    practical_exercises: true

  # Assessment
  assessment:
    knowledge_checks: true
    practical_challenges: true
    mentor_review_points: true

  # Progress tracking
  progress:
    storage: file                   # file | database
    path: ".amplifier/onboarding/"
    milestones: true
    time_tracking: true

  # Output
  output:
    format: interactive              # interactive | static | both
    export_pdf: false
```

---

## Pipeline Implementation

```python
class CodebaseOnboarderScenario:
    """Interactive codebase onboarding experience."""

    def __init__(self, config: OnboarderConfig):
        self.config = config
        self.tools = ToolRegistry()
        self.progress = OnboardingProgress()

    async def start_onboarding(self, user_id: str) -> OnboardingSession:
        """Initialize onboarding for a new user."""

        # Stage 1: Collect Profile
        profile = await self._collect_profile(user_id)

        # Stage 2: Analyze Codebase
        codebase_info = await self._analyze_codebase()

        # Stage 3: Generate Learning Path
        learning_path = await self._generate_learning_path(profile, codebase_info)

        # Stage 4: Start Interactive Session
        session = OnboardingSession(
            user_id=user_id,
            profile=profile,
            learning_path=learning_path,
            codebase_info=codebase_info,
            progress=self.progress.load(user_id)
        )

        return session

    async def _collect_profile(self, user_id: str) -> UserProfile:
        """Stage 1: Collect user profile through interactive questions."""

        questions = [
            ProfileQuestion(
                id="role",
                text="What's your primary role?",
                options=["Frontend", "Backend", "Full-stack", "DevOps", "Data/ML", "Other"]
            ),
            ProfileQuestion(
                id="experience",
                text="What's your experience level?",
                options=["Junior (0-2 years)", "Mid (2-5 years)", "Senior (5+ years)"]
            ),
            ProfileQuestion(
                id="languages",
                text="Which languages/frameworks are you already familiar with?",
                multi_select=True,
                options=["Python", "JavaScript/TypeScript", "Go", "Java", "React", "Node.js", "Other"]
            ),
            ProfileQuestion(
                id="focus",
                text="What area will you primarily work on?",
                options=await self._get_codebase_areas()
            ),
            ProfileQuestion(
                id="time",
                text="How much time can you dedicate to onboarding daily?",
                options=["1 hour", "2 hours", "4 hours", "Full day"]
            )
        ]

        # Collect answers (interactive or from config)
        answers = await self._collect_answers(questions)

        return UserProfile(
            user_id=user_id,
            role=answers["role"],
            experience=answers["experience"],
            familiar_with=answers["languages"],
            focus_area=answers["focus"],
            daily_time_budget=self._parse_time(answers["time"])
        )

    async def _analyze_codebase(self) -> CodebaseInfo:
        """Stage 2: Analyze codebase structure and content."""

        info = CodebaseInfo()

        # Get architecture
        arch_tool = self.tools.get("tool-architecture-map")
        info.architecture = await arch_tool.execute({
            "action": "generate",
            "include_descriptions": True
        })

        # Get key components
        search_tool = self.tools.get("tool-codebase-search")

        # Find main entry points
        info.entry_points = await search_tool.execute({
            "query": "main entry point",
            "file_patterns": ["main.*", "index.*", "app.*"]
        })

        # Find core modules
        info.core_modules = await self._identify_core_modules()

        # Collect documentation
        info.documentation = await self._collect_documentation()

        # Identify patterns
        info.patterns = await self._identify_patterns()

        # Get tech stack
        info.tech_stack = await self._identify_tech_stack()

        return info

    async def _generate_learning_path(
        self,
        profile: UserProfile,
        codebase: CodebaseInfo
    ) -> LearningPath:
        """Stage 3: Generate personalized learning path."""

        modules = []

        # Module 1: Architecture Overview (always first)
        modules.append(await self._generate_architecture_module(codebase))

        # Module 2: Core Concepts (tailored to tech stack)
        modules.append(await self._generate_concepts_module(
            codebase, profile.familiar_with
        ))

        # Module 3: Development Workflow
        modules.append(await self._generate_workflow_module(codebase))

        # Module 4: Focus Area Deep Dive
        if profile.focus_area:
            modules.append(await self._generate_focus_module(
                codebase, profile.focus_area
            ))

        # Module 5: Key Components
        modules.append(await self._generate_components_module(
            codebase, profile.focus_area
        ))

        # Module 6: Common Patterns
        modules.append(await self._generate_patterns_module(
            codebase, profile.experience
        ))

        # Module 7: Testing Practices
        modules.append(await self._generate_testing_module(codebase))

        # Adjust based on time budget
        learning_path = LearningPath(
            modules=modules,
            estimated_total_time=sum(m.estimated_time for m in modules),
            daily_schedule=self._create_schedule(modules, profile.daily_time_budget)
        )

        return learning_path

    async def _generate_architecture_module(
        self,
        codebase: CodebaseInfo
    ) -> LearningModule:
        """Generate architecture overview module."""

        sections = []

        # High-level overview
        sections.append(Section(
            title="System Overview",
            content=await self._generate_overview(codebase),
            type="reading",
            estimated_time=timedelta(minutes=15)
        ))

        # Architecture diagram
        sections.append(Section(
            title="Architecture Diagram",
            content=self._render_architecture_diagram(codebase.architecture),
            type="visual",
            estimated_time=timedelta(minutes=10)
        ))

        # Key services/components
        for component in codebase.architecture.main_components[:5]:
            sections.append(Section(
                title=f"Component: {component.name}",
                content=await self._explain_component(component),
                type="reading",
                code_references=component.key_files,
                estimated_time=timedelta(minutes=10)
            ))

        # Interactive exploration
        sections.append(Section(
            title="Explore the Architecture",
            content="Let's walk through the codebase interactively.",
            type="interactive",
            exercise=ArchitectureExploration(codebase),
            estimated_time=timedelta(minutes=30)
        ))

        # Knowledge check
        sections.append(Section(
            title="Architecture Quiz",
            type="quiz",
            questions=await self._generate_arch_quiz(codebase),
            estimated_time=timedelta(minutes=10)
        ))

        return LearningModule(
            id="architecture",
            title="Architecture Overview",
            description="Understand the high-level structure of the codebase",
            sections=sections,
            estimated_time=sum(s.estimated_time for s in sections, timedelta())
        )

    async def answer_question(
        self,
        session: OnboardingSession,
        question: str
    ) -> Answer:
        """Answer a question about the codebase with full context."""

        # Search for relevant code
        search_tool = self.tools.get("tool-codebase-search")
        relevant_code = await search_tool.execute({
            "query": question,
            "search_type": "semantic",
            "max_results": 10
        })

        # Get architecture context
        arch_context = self._get_relevant_architecture(
            question, session.codebase_info.architecture
        )

        # Check documentation
        doc_context = self._search_documentation(
            question, session.codebase_info.documentation
        )

        # Generate answer with full context
        answer = await self._generate_answer(
            question=question,
            code_context=relevant_code,
            architecture_context=arch_context,
            documentation_context=doc_context,
            user_profile=session.profile,
            learning_progress=session.progress
        )

        # Track question for learning insights
        session.questions_asked.append(QuestionRecord(
            question=question,
            topic=answer.topic,
            timestamp=datetime.utcnow()
        ))

        return answer

    async def run_code_walkthrough(
        self,
        session: OnboardingSession,
        starting_point: str
    ) -> CodeWalkthrough:
        """Interactive code walkthrough from a starting point."""

        walkthrough = CodeWalkthrough(starting_point=starting_point)

        # Load starting file
        current_file = await self._load_file(starting_point)

        # Explain the file
        explanation = await self._explain_file(
            current_file, session.profile, session.codebase_info
        )

        walkthrough.add_step(WalkthroughStep(
            file=starting_point,
            explanation=explanation,
            highlights=explanation.key_lines
        ))

        # Follow imports/dependencies
        dependencies = await self._get_file_dependencies(current_file)

        walkthrough.suggested_next = [
            SuggestedExploration(
                file=dep.path,
                reason=dep.relationship,
                relevance=dep.relevance_score
            )
            for dep in dependencies[:5]
        ]

        return walkthrough


class InteractiveExplorer:
    """Interactive code exploration features."""

    async def explain_function(
        self,
        file_path: str,
        function_name: str,
        profile: UserProfile
    ) -> FunctionExplanation:
        """Explain a specific function in detail."""

        # Load function code
        code = await self._extract_function(file_path, function_name)

        # Generate explanation based on experience level
        explanation = await self._generate_explanation(
            code=code,
            experience_level=profile.experience,
            familiar_technologies=profile.familiar_with
        )

        return FunctionExplanation(
            name=function_name,
            purpose=explanation.purpose,
            parameters=explanation.parameters,
            return_value=explanation.return_value,
            algorithm=explanation.algorithm,
            complexity=explanation.complexity,
            related_functions=explanation.related,
            example_usage=explanation.examples
        )

    async def trace_data_flow(
        self,
        starting_point: str,
        data_name: str
    ) -> DataFlowTrace:
        """Trace how data flows through the system."""

        trace = DataFlowTrace(data=data_name)

        # Use dependency graph to trace
        dep_tool = self.tools.get("tool-dependency-graph")

        # Find where data originates
        sources = await dep_tool.execute({
            "action": "find_sources",
            "symbol": data_name,
            "start_file": starting_point
        })

        trace.sources = sources

        # Find transformations
        transforms = await dep_tool.execute({
            "action": "find_transforms",
            "symbol": data_name
        })

        trace.transformations = transforms

        # Find consumers
        consumers = await dep_tool.execute({
            "action": "find_consumers",
            "symbol": data_name
        })

        trace.consumers = consumers

        # Generate visual
        trace.diagram = self._generate_flow_diagram(sources, transforms, consumers)

        return trace


class ProgressTracker:
    """Track onboarding progress and milestones."""

    async def update_progress(
        self,
        user_id: str,
        module_id: str,
        section_id: str,
        completion: float
    ) -> ProgressUpdate:
        """Update learning progress."""

        progress = self.load(user_id)

        # Update section progress
        if module_id not in progress.modules:
            progress.modules[module_id] = ModuleProgress(module_id=module_id)

        progress.modules[module_id].sections[section_id] = completion

        # Calculate module completion
        module_completion = self._calculate_module_completion(
            progress.modules[module_id]
        )
        progress.modules[module_id].completion = module_completion

        # Check milestones
        new_milestones = self._check_milestones(progress)
        for milestone in new_milestones:
            progress.milestones_achieved.append(milestone)

        # Update total progress
        progress.total_completion = self._calculate_total_completion(progress)
        progress.time_spent += self._session_time()

        self.save(user_id, progress)

        return ProgressUpdate(
            module_completion=module_completion,
            total_completion=progress.total_completion,
            new_milestones=new_milestones,
            next_recommended=self._recommend_next(progress)
        )

    def generate_report(self, user_id: str) -> OnboardingReport:
        """Generate progress report."""

        progress = self.load(user_id)

        return OnboardingReport(
            user_id=user_id,
            started=progress.started_at,
            total_completion=progress.total_completion,
            time_spent=progress.time_spent,
            modules=[
                ModuleReport(
                    module_id=m.module_id,
                    completion=m.completion,
                    quiz_scores=m.quiz_scores,
                    challenges_completed=m.challenges_completed
                )
                for m in progress.modules.values()
            ],
            milestones=progress.milestones_achieved,
            topics_explored=progress.topics_explored,
            questions_asked=len(progress.questions),
            strengths=self._identify_strengths(progress),
            areas_for_focus=self._identify_focus_areas(progress)
        )
```

---

## Learning Module Examples

### Architecture Overview

```markdown
# Architecture Overview

## System Structure

This codebase follows a **microservices architecture** with the following
key components:

### API Gateway (`/services/gateway/`)
The entry point for all client requests. Handles:
- Authentication/authorization
- Rate limiting
- Request routing

### User Service (`/services/users/`)
Manages user accounts, profiles, and authentication.

### Payment Service (`/services/payments/`)
Handles all payment processing with Stripe integration.

[View Architecture Diagram]

---

## Interactive Exploration

Let's explore the architecture together. I'll guide you through the
main entry points and how requests flow through the system.

**Starting point**: `services/gateway/src/index.ts`

> "This is the API gateway entry point. When a request comes in,
> it first goes through the authentication middleware (line 45),
> then gets routed to the appropriate service (line 67)."

Would you like to:
1. Dive deeper into the authentication flow
2. See how a payment request is processed
3. Explore the user service
4. Ask a question

---

## Knowledge Check

1. What is the primary role of the API Gateway?
   - [ ] Database management
   - [x] Request routing and authentication
   - [ ] Payment processing

2. Where would you find the code that handles user registration?
   - [ ] services/gateway/
   - [x] services/users/
   - [ ] services/payments/
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `profile.collect` | list | [all] | Profile info to collect |
| `content.sources.*` | bool | varies | Documentation sources |
| `interactive.code_walkthrough` | bool | true | Enable walkthroughs |
| `assessment.knowledge_checks` | bool | true | Enable quizzes |
| `progress.storage` | string | "file" | Progress storage |

---

## Dependencies

### Required Tools
- `tool-codebase-search` - Code search and analysis
- `tool-architecture-map` - Architecture visualization
- `tool-dependency-graph` - Code flow analysis

### Optional
- Documentation platforms (Confluence, Notion)
- LMS integrations

---

## Open Questions

1. **Team integration**: Assign mentors based on onboarding progress?
2. **Gamification**: Add achievements/badges for motivation?
3. **Adaptive learning**: Adjust difficulty based on performance?
4. **Content updates**: Auto-update when codebase changes?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
