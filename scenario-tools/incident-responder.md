# incident-responder

> **Priority**: P0 (Critical Path)
> **Status**: Draft
> **Module**: `amplifier-scenario-incident-responder`

## Overview

Automated incident response workflow that gathers context, analyzes symptoms, suggests root causes, and coordinates remediation. Reduces MTTR by automating the information gathering and initial diagnosis phases.

### Value Proposition

| Without | With |
|---------|------|
| Manual log diving | Automated log aggregation |
| Knowledge scattered | Relevant context surfaced |
| Slow initial diagnosis | Rapid symptom analysis |
| War room chaos | Structured response workflow |

---

## Workflow Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Incident Response Pipeline                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐      │
│  │  Alert   │───▶│ Context  │───▶│ Diagnose │───▶│ Remediate│      │
│  │ Ingest   │    │ Gather   │    │          │    │          │      │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘      │
│       │               │               │               │              │
│       ▼               ▼               ▼               ▼              │
│  • PagerDuty     • Recent deploys  • Log analysis  • Rollback?      │
│  • Datadog       • Code changes    • Error patterns• Config fix?    │
│  • Slack         • Related alerts  • Correlation   • Scale?         │
│  • Custom        • System metrics  • Root cause    • Communicate    │
│                                                                      │
│                    ┌──────────┐                                      │
│                    │ Document │◀── Throughout                        │
│                    │ & Learn  │                                      │
│                    └──────────┘                                      │
│                         │                                            │
│                         ▼                                            │
│                    • Timeline                                        │
│                    • Postmortem                                      │
│                    • Runbook update                                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Configuration

```yaml
scenario:
  name: incident-responder
  version: "1.0.0"

  # Alert sources
  alerts:
    sources:
      pagerduty:
        enabled: true
        api_key: "${PAGERDUTY_API_KEY}"
      datadog:
        enabled: true
        api_key: "${DD_API_KEY}"
        app_key: "${DD_APP_KEY}"
      slack:
        enabled: true
        channel: "#incidents"

  # Context gathering
  context:
    # Recent deployments
    deployments:
      sources:
        - github-actions
        - argocd
        - jenkins
      lookback: 24h

    # Code changes
    code:
      lookback: 48h
      focus_areas:              # Priority areas to check
        - config
        - database
        - api
        - dependencies

    # Observability
    metrics:
      provider: datadog         # datadog | prometheus | cloudwatch
      queries:
        error_rate: "sum:http.requests{status:5xx}.as_rate()"
        latency_p99: "avg:http.request.latency.p99{*}"
        cpu: "avg:system.cpu.user{*}"
        memory: "avg:system.mem.used{*}"
      lookback: 4h

    # Logs
    logs:
      provider: datadog         # datadog | elasticsearch | cloudwatch
      lookback: 2h
      max_entries: 1000
      focus_patterns:
        - "error"
        - "exception"
        - "timeout"
        - "connection refused"

    # Related incidents
    history:
      similar_lookback: 90d
      correlation_threshold: 0.7

  # Diagnosis
  diagnosis:
    # Analysis approaches
    approaches:
      log_pattern_analysis: true
      metric_correlation: true
      change_correlation: true
      similar_incident_match: true

    # Root cause categories
    categories:
      - deployment_issue
      - configuration_error
      - capacity_problem
      - dependency_failure
      - code_bug
      - external_factor

  # Remediation
  remediation:
    # Automated actions (require approval)
    actions:
      rollback:
        enabled: true
        require_approval: true
      scale:
        enabled: true
        require_approval: true
      config_revert:
        enabled: true
        require_approval: true

    # Communication
    communication:
      slack_channel: "#incidents"
      status_page: true
      stakeholder_notify: true

  # Documentation
  documentation:
    timeline: true
    auto_postmortem: true
    runbook_suggestions: true
```

---

## Pipeline Implementation

```python
class IncidentResponderScenario:
    """Automated incident response workflow."""

    def __init__(self, config: IncidentConfig):
        self.config = config
        self.tools = ToolRegistry()
        self.timeline = Timeline()

    async def respond(self, alert: Alert) -> IncidentReport:
        """Execute incident response pipeline."""

        # Initialize incident
        incident = await self._initialize_incident(alert)
        self.timeline.add("Incident started", alert.summary)

        # Stage 1: Gather Context
        context = await self._gather_context(incident)
        self.timeline.add("Context gathered", f"{len(context.items)} items")

        # Stage 2: Diagnose
        diagnosis = await self._diagnose(incident, context)
        self.timeline.add("Diagnosis complete", diagnosis.summary)

        # Stage 3: Suggest Remediation
        remediation = await self._suggest_remediation(diagnosis)
        self.timeline.add("Remediation suggested", remediation.summary)

        # Stage 4: Coordinate Response
        if self.config.remediation.communication.slack_channel:
            await self._post_status(incident, diagnosis, remediation)

        # Generate report
        return await self._generate_report(
            incident, context, diagnosis, remediation
        )

    async def _gather_context(self, incident: Incident) -> IncidentContext:
        """Stage 1: Gather all relevant context."""

        context = IncidentContext()

        # Parallel context gathering
        tasks = [
            self._get_recent_deploys(incident),
            self._get_recent_changes(incident),
            self._get_metrics(incident),
            self._get_logs(incident),
            self._get_similar_incidents(incident),
            self._get_related_alerts(incident),
        ]

        results = await asyncio.gather(*tasks, return_exceptions=True)

        context.deployments = results[0] if not isinstance(results[0], Exception) else []
        context.code_changes = results[1] if not isinstance(results[1], Exception) else []
        context.metrics = results[2] if not isinstance(results[2], Exception) else {}
        context.logs = results[3] if not isinstance(results[3], Exception) else []
        context.similar_incidents = results[4] if not isinstance(results[4], Exception) else []
        context.related_alerts = results[5] if not isinstance(results[5], Exception) else []

        return context

    async def _get_recent_deploys(self, incident: Incident) -> list[Deployment]:
        """Get recent deployments."""
        deployments = []

        for source in self.config.context.deployments.sources:
            if source == "github-actions":
                runs = await self._query_github_actions(incident)
                deployments.extend(runs)
            elif source == "argocd":
                syncs = await self._query_argocd(incident)
                deployments.extend(syncs)

        # Sort by time, most recent first
        deployments.sort(key=lambda d: d.timestamp, reverse=True)

        return deployments

    async def _get_logs(self, incident: Incident) -> list[LogEntry]:
        """Fetch relevant logs."""
        metrics_tool = self.tools.get("tool-metrics-query")

        # Build log query based on incident
        query = self._build_log_query(incident)

        logs = await metrics_tool.execute({
            "action": "query_logs",
            "provider": self.config.context.logs.provider,
            "query": query,
            "start_time": incident.started_at - timedelta(
                hours=int(self.config.context.logs.lookback.rstrip('h'))
            ),
            "end_time": datetime.utcnow(),
            "limit": self.config.context.logs.max_entries
        })

        # Filter for relevant entries
        filtered = self._filter_relevant_logs(logs, incident)

        return filtered

    async def _diagnose(
        self,
        incident: Incident,
        context: IncidentContext
    ) -> Diagnosis:
        """Stage 2: Analyze and diagnose."""

        hypotheses = []

        # Approach 1: Log pattern analysis
        if self.config.diagnosis.approaches.log_pattern_analysis:
            log_hypothesis = await self._analyze_log_patterns(context.logs)
            if log_hypothesis:
                hypotheses.append(log_hypothesis)

        # Approach 2: Metric correlation
        if self.config.diagnosis.approaches.metric_correlation:
            metric_hypothesis = await self._analyze_metric_correlation(
                context.metrics, incident
            )
            if metric_hypothesis:
                hypotheses.append(metric_hypothesis)

        # Approach 3: Change correlation
        if self.config.diagnosis.approaches.change_correlation:
            change_hypothesis = await self._correlate_changes(
                context.deployments, context.code_changes, incident
            )
            if change_hypothesis:
                hypotheses.append(change_hypothesis)

        # Approach 4: Similar incident matching
        if self.config.diagnosis.approaches.similar_incident_match:
            similar_hypothesis = await self._match_similar_incidents(
                context.similar_incidents, incident
            )
            if similar_hypothesis:
                hypotheses.append(similar_hypothesis)

        # Synthesize diagnosis
        diagnosis = await self._synthesize_diagnosis(hypotheses, context)

        return diagnosis

    async def _analyze_log_patterns(self, logs: list[LogEntry]) -> Hypothesis | None:
        """Analyze logs for error patterns."""

        # Extract error patterns
        error_patterns = defaultdict(int)
        error_examples = defaultdict(list)

        for log in logs:
            if log.level in ("error", "fatal"):
                pattern = self._extract_pattern(log.message)
                error_patterns[pattern] += 1
                if len(error_examples[pattern]) < 3:
                    error_examples[pattern].append(log)

        if not error_patterns:
            return None

        # Find dominant pattern
        dominant = max(error_patterns, key=error_patterns.get)
        count = error_patterns[dominant]

        # Analyze pattern
        analysis = await self._analyze_error_pattern(
            dominant, error_examples[dominant]
        )

        return Hypothesis(
            category=analysis.category,
            description=analysis.description,
            confidence=min(0.9, count / 100),  # More occurrences = higher confidence
            evidence=[
                f"Error pattern occurred {count} times",
                f"First occurrence: {error_examples[dominant][0].timestamp}",
                f"Example: {error_examples[dominant][0].message[:200]}"
            ],
            suggested_actions=analysis.suggested_actions
        )

    async def _correlate_changes(
        self,
        deployments: list[Deployment],
        code_changes: list[CodeChange],
        incident: Incident
    ) -> Hypothesis | None:
        """Correlate incident with recent changes."""

        # Find changes in time window before incident
        window_start = incident.started_at - timedelta(hours=4)

        suspect_deploys = [
            d for d in deployments
            if window_start <= d.timestamp <= incident.started_at
        ]

        if not suspect_deploys:
            return None

        # Most recent deployment is prime suspect
        prime_suspect = suspect_deploys[0]

        # Analyze what changed
        changes = await self._analyze_deployment_changes(prime_suspect)

        return Hypothesis(
            category="deployment_issue",
            description=f"Incident correlates with deployment {prime_suspect.id}",
            confidence=0.7,
            evidence=[
                f"Deployment at {prime_suspect.timestamp}",
                f"Incident started at {incident.started_at}",
                f"Time between: {incident.started_at - prime_suspect.timestamp}",
                f"Changes: {', '.join(changes.summary)}"
            ],
            suggested_actions=[
                f"Consider rollback to {prime_suspect.previous_version}",
                "Review deployment changes for issues",
                "Check deployment health metrics"
            ],
            metadata={
                "deployment_id": prime_suspect.id,
                "previous_version": prime_suspect.previous_version,
                "changes": changes
            }
        )

    async def _suggest_remediation(self, diagnosis: Diagnosis) -> Remediation:
        """Stage 3: Suggest remediation actions."""

        actions = []

        for hypothesis in diagnosis.hypotheses:
            if hypothesis.category == "deployment_issue":
                if hypothesis.confidence >= 0.6:
                    actions.append(RemediationAction(
                        type="rollback",
                        description="Roll back to previous version",
                        command=f"kubectl rollout undo deployment/{hypothesis.metadata['deployment']}",
                        risk="low",
                        require_approval=True
                    ))

            elif hypothesis.category == "capacity_problem":
                actions.append(RemediationAction(
                    type="scale",
                    description="Scale up affected service",
                    command=f"kubectl scale deployment/{hypothesis.metadata['service']} --replicas=+2",
                    risk="low",
                    require_approval=True
                ))

            elif hypothesis.category == "configuration_error":
                actions.append(RemediationAction(
                    type="config_revert",
                    description="Revert configuration change",
                    risk="medium",
                    require_approval=True
                ))

        # Add communication actions
        actions.append(RemediationAction(
            type="communicate",
            description="Update status page",
            risk="none",
            require_approval=False
        ))

        return Remediation(
            primary_action=actions[0] if actions else None,
            alternative_actions=actions[1:],
            communication_plan=await self._generate_comms_plan(diagnosis)
        )

    async def _post_status(
        self,
        incident: Incident,
        diagnosis: Diagnosis,
        remediation: Remediation
    ) -> None:
        """Post status update to Slack."""

        slack_tool = self.tools.get("tool-slack-search")

        message = self._format_status_message(incident, diagnosis, remediation)

        await slack_tool.execute({
            "action": "post",
            "channel": self.config.remediation.communication.slack_channel,
            "message": message,
            "thread_ts": incident.slack_thread_ts  # Continue in thread if exists
        })
```

---

## Status Message Format

```markdown
🚨 **Incident Update: Payment Processing Errors**

**Status**: Investigating
**Severity**: P1
**Duration**: 23 minutes

---

### Summary
Payment API returning 500 errors for ~15% of requests since 14:32 UTC.

### Initial Diagnosis
**Most Likely Cause**: Recent deployment (deploy-abc123 at 14:28 UTC)
**Confidence**: 75%

**Evidence**:
- Error spike began 4 minutes after deployment
- Errors concentrated in new payment validation code
- No infrastructure anomalies detected

### Suggested Actions
1. ⚡ **Recommended**: Rollback to previous version (v2.3.4)
2. 🔧 Alternative: Hot-fix validation logic
3. 📞 Notify payment team

### Timeline
- 14:28 - Deployment deploy-abc123 completed
- 14:32 - First errors detected
- 14:35 - Alert triggered (PagerDuty)
- 14:37 - Incident responder activated
- 14:45 - Initial diagnosis complete

---
🤖 *Generated by Incident Responder*
```

---

## Postmortem Generation

```python
class PostmortemGenerator:
    """Generate postmortem from incident data."""

    async def generate(
        self,
        incident: Incident,
        timeline: Timeline,
        diagnosis: Diagnosis,
        remediation_taken: RemediationAction
    ) -> Postmortem:
        """Generate structured postmortem."""

        return Postmortem(
            title=f"Postmortem: {incident.title}",
            date=incident.started_at.date(),
            duration=incident.resolved_at - incident.started_at,
            severity=incident.severity,

            summary=await self._generate_summary(incident, diagnosis),

            impact=Impact(
                users_affected=incident.metrics.users_affected,
                requests_failed=incident.metrics.requests_failed,
                revenue_impact=incident.metrics.revenue_impact,
                sla_breach=incident.metrics.sla_breach
            ),

            timeline=timeline.entries,

            root_cause=RootCause(
                category=diagnosis.primary_hypothesis.category,
                description=diagnosis.primary_hypothesis.description,
                evidence=diagnosis.primary_hypothesis.evidence
            ),

            resolution=Resolution(
                action_taken=remediation_taken.description,
                time_to_resolve=incident.resolved_at - incident.started_at,
                verified_by=remediation_taken.verified_by
            ),

            lessons_learned=await self._extract_lessons(incident, diagnosis),

            action_items=await self._generate_action_items(incident, diagnosis),

            appendix=Appendix(
                logs=incident.key_logs,
                metrics=incident.key_metrics,
                related_links=incident.related_links
            )
        )

    async def _generate_action_items(
        self,
        incident: Incident,
        diagnosis: Diagnosis
    ) -> list[ActionItem]:
        """Generate follow-up action items."""

        items = []

        # Prevention items
        if diagnosis.primary_hypothesis.category == "deployment_issue":
            items.append(ActionItem(
                type="prevention",
                title="Improve deployment canary analysis",
                description="Add automated canary metrics checking",
                priority="high",
                owner=None  # To be assigned
            ))

        # Detection items
        if incident.time_to_detect > timedelta(minutes=5):
            items.append(ActionItem(
                type="detection",
                title="Improve alerting thresholds",
                description=f"Current TTD was {incident.time_to_detect}, target is <5min",
                priority="medium"
            ))

        # Documentation items
        items.append(ActionItem(
            type="documentation",
            title="Update runbook for this failure mode",
            description=f"Add steps for {diagnosis.primary_hypothesis.category}",
            priority="low"
        ))

        return items
```

---

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `context.deployments.lookback` | string | "24h" | Deploy history window |
| `context.logs.lookback` | string | "2h" | Log query window |
| `context.metrics.lookback` | string | "4h" | Metrics window |
| `diagnosis.approaches.*` | bool | true | Enable diagnosis approaches |
| `remediation.actions.*.require_approval` | bool | true | Require human approval |

---

## Dependencies

### Required Tools
- `tool-metrics-query` - Observability data
- `tool-git-advanced` - Change correlation
- `tool-slack-search` - Communication

### External Integrations
- PagerDuty / OpsGenie API
- Datadog / Prometheus
- GitHub / GitLab
- Slack

---

## Open Questions

1. **Automated remediation**: How much automation without human approval?
2. **ML diagnosis**: Train on past incidents for better pattern matching?
3. **Runbook integration**: Link to and execute runbooks?
4. **Multi-service correlation**: Handle distributed system incidents?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
