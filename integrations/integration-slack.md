# integration-slack

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-integration-slack`

## Overview

Slack app integration providing AI-powered assistance directly within team conversations. Enables code questions, incident response, PR notifications, and interactive workflows without leaving Slack.

### Value Proposition

| Without | With |
|---------|------|
| Context switching to IDE/terminal | AI assistance in Slack |
| Manual incident notifications | Automated alerts with context |
| Separate tools for collaboration | Unified team AI interface |
| No audit trail in conversations | Integrated team knowledge |

---

## Features

### 1. Direct Message Assistant

Personal AI assistant for code questions.

```
User: @Amplifier How does our payment retry logic work?

Amplifier: Based on the codebase analysis:

**Payment Retry Logic** (`src/services/payment/retry.ts`)

The retry mechanism uses exponential backoff:

1. **Initial attempt** - Direct charge via Stripe
2. **On failure** - Wait 2^attempt seconds (max 5 attempts)
3. **Retry conditions**:
   - Network errors: Always retry
   - Rate limits: Retry with backoff
   - Card declined: No retry, notify user

```typescript
const RETRY_CONFIG = {
  maxAttempts: 5,
  baseDelay: 1000,
  maxDelay: 32000,
  retryableErrors: ['network_error', 'rate_limit']
};
```

Want me to explain any specific part?
```

### 2. Channel Integration

Team-wide AI assistance in public channels.

```
#engineering channel:

Sarah: @Amplifier What's the status of PR #1234?

Amplifier: **PR #1234: Add user authentication**

📊 **Status**: Ready for review
👤 **Author**: @john
📝 **Changes**: +423 / -89 lines across 12 files

**AI Review Summary**:
- ✅ Security: No issues found
- ⚠️ Code Quality: 2 suggestions
- ✅ Tests: Coverage increased to 87%

**Suggested Reviewers**: @alice (auth expert), @bob (security)

[View PR](https://github.com/org/repo/pull/1234) | [Full Review](link)
```

### 3. Incident Response Bot

Automated incident assistance.

```
#incidents channel:

🚨 **Incident Alert** (triggered by PagerDuty)
Service: payment-api
Status: HIGH_ERROR_RATE
Started: 5 minutes ago

Amplifier: 🔍 **Investigating incident...**

**Initial Analysis**:
- Error rate spiked from 0.1% to 15% at 14:23 UTC
- Affected endpoint: POST /api/payments
- Recent deployment: `payment-api@v2.3.1` (deployed 14:20 UTC)

**Likely Cause**:
Database connection pool exhaustion. `payment-api@v2.3.1`
increased pool size from 10 to 50, but RDS max_connections is 100.

**Recommended Actions**:
1. 🔄 Rollback to `payment-api@v2.3.0`
2. 📊 Monitor error rate recovery
3. 🔧 Increase RDS max_connections before redeploying

React with ✅ to execute rollback, ❌ to dismiss.
```

### 4. PR Notifications

Smart PR lifecycle notifications.

```yaml
# Notification triggers
notifications:
  pr_opened:
    channel: "#code-review"
    include_ai_summary: true
    mention_suggested_reviewers: true

  pr_review_completed:
    channel: "#code-review"
    include_ai_summary: true

  pr_merged:
    channel: "#deployments"
    include_changelog_entry: true

  ci_failed:
    channel: "#ci-alerts"
    include_ai_diagnosis: true
```

```
#code-review channel:

📬 **New PR: Implement rate limiting** (#1456)
Author: @alice | +156 / -23 | 4 files

**AI Summary**: Adds Redis-based rate limiting to API endpoints.
Implements token bucket algorithm with configurable limits per user tier.

**Key Changes**:
- New `RateLimiter` middleware
- Redis integration for distributed counting
- Configuration via environment variables

**Suggested Reviewers**: @bob (API), @carol (Redis)

[Review PR](link) | [View Changes](link)
```

### 5. Slash Commands

Interactive commands for common workflows.

```
/amplifier review https://github.com/org/repo/pull/1234
/amplifier search "payment retry logic"
/amplifier explain src/services/auth.ts
/amplifier incident start "API latency spike"
/amplifier deploy staging payment-api
```

### 6. Interactive Workflows

Button-based workflows for approvals and actions.

```
Amplifier: 🚀 **Deployment Request**

Service: `payment-api`
Version: `v2.3.2`
Environment: `staging`
Requested by: @john

**Pre-deployment Checks**:
- ✅ Tests passing
- ✅ Security scan clean
- ✅ No breaking changes detected

[Approve] [Reject] [Request Changes]

---

@alice clicked [Approve]

Amplifier: ✅ Deployment approved by @alice
Deploying `payment-api@v2.3.2` to staging...

Deployment complete!
🔗 https://staging.example.com/api/health
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Slack Integration                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │
│  │ Event       │    │ Slash       │    │ Interactive │             │
│  │ Handler     │    │ Commands    │    │ Components  │             │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘             │
│         │                  │                  │                     │
│         └──────────────────┼──────────────────┘                     │
│                            ▼                                        │
│                   ┌─────────────────┐                               │
│                   │ Request Router  │                               │
│                   └────────┬────────┘                               │
│                            │                                        │
│         ┌──────────────────┼──────────────────┐                    │
│         ▼                  ▼                  ▼                     │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐              │
│  │ Context     │   │ Amplifier   │   │ Response    │              │
│  │ Builder     │   │ Session     │   │ Formatter   │              │
│  └─────────────┘   └─────────────┘   └─────────────┘              │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Notification Service (webhooks, scheduled, event-driven)       │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Implementation

### Event Handler

```python
# handlers/events.py

from slack_bolt import App
from slack_bolt.adapter.socket_mode import SocketModeHandler
from amplifier import AmplifierSession

app = App(token=os.environ["SLACK_BOT_TOKEN"])

@app.event("app_mention")
async def handle_mention(event, say, client):
    """Handle @Amplifier mentions in channels."""

    user = event["user"]
    channel = event["channel"]
    text = event["text"]
    thread_ts = event.get("thread_ts", event["ts"])

    # Remove bot mention from text
    query = remove_mention(text)

    # Build context from thread history
    context = await build_thread_context(client, channel, thread_ts)

    # Get channel-specific configuration
    config = get_channel_config(channel)

    # Execute with Amplifier
    async with AmplifierSession(config=config) as session:
        result = await session.execute(
            prompt=query,
            context={
                "slack_channel": channel,
                "slack_user": user,
                "thread_context": context
            }
        )

    # Format and send response
    blocks = format_response_blocks(result)
    await say(blocks=blocks, thread_ts=thread_ts)


@app.event("message")
async def handle_dm(event, say):
    """Handle direct messages to the bot."""

    # Only handle DMs
    if event.get("channel_type") != "im":
        return

    user = event["user"]
    text = event["text"]

    # Get user-specific context
    user_context = await get_user_context(user)

    # Execute with Amplifier
    async with AmplifierSession(config=DM_CONFIG) as session:
        result = await session.execute(
            prompt=text,
            context={
                "user": user,
                "user_projects": user_context.get("projects"),
                "user_repositories": user_context.get("repositories")
            }
        )

    # Send response
    await say(text=result.response)
```

### Slash Commands

```python
# handlers/commands.py

@app.command("/amplifier")
async def handle_command(ack, command, respond):
    """Handle /amplifier slash commands."""

    await ack()  # Acknowledge immediately

    text = command["text"]
    user = command["user_id"]
    channel = command["channel_id"]

    # Parse command
    parts = text.split(maxsplit=1)
    action = parts[0] if parts else "help"
    args = parts[1] if len(parts) > 1 else ""

    # Route to handler
    handlers = {
        "review": handle_review_command,
        "search": handle_search_command,
        "explain": handle_explain_command,
        "incident": handle_incident_command,
        "deploy": handle_deploy_command,
        "help": handle_help_command
    }

    handler = handlers.get(action, handle_unknown_command)
    await handler(respond, user, channel, args)


async def handle_review_command(respond, user, channel, args):
    """Handle /amplifier review <pr_url>"""

    pr_url = args.strip()

    if not pr_url:
        await respond("Usage: `/amplifier review <pr_url>`")
        return

    # Show loading state
    await respond({
        "response_type": "in_channel",
        "text": f"🔍 Analyzing PR: {pr_url}..."
    })

    # Execute PR review
    async with AmplifierSession(config=PR_REVIEW_CONFIG) as session:
        result = await session.execute(
            prompt="Review this PR",
            context={"pr_url": pr_url}
        )

    # Format as Slack blocks
    blocks = format_pr_review_blocks(result)

    await respond({
        "response_type": "in_channel",
        "blocks": blocks
    })


async def handle_incident_command(respond, user, channel, args):
    """Handle /amplifier incident <action> [args]"""

    parts = args.split(maxsplit=1)
    action = parts[0] if parts else "help"
    details = parts[1] if len(parts) > 1 else ""

    if action == "start":
        # Create incident channel
        incident_id = generate_incident_id()
        incident_channel = await create_incident_channel(incident_id, details)

        # Start incident response workflow
        async with AmplifierSession(config=INCIDENT_CONFIG) as session:
            result = await session.execute(
                prompt=f"Start incident response: {details}",
                context={
                    "incident_id": incident_id,
                    "reported_by": user,
                    "description": details
                }
            )

        await respond({
            "response_type": "in_channel",
            "blocks": [
                {
                    "type": "section",
                    "text": {
                        "type": "mrkdwn",
                        "text": f"🚨 Incident `{incident_id}` created\n"
                               f"Channel: <#{incident_channel}>"
                    }
                }
            ]
        })
```

### Interactive Components

```python
# handlers/interactions.py

@app.action("approve_deployment")
async def handle_deployment_approval(ack, body, client):
    """Handle deployment approval button click."""

    await ack()

    user = body["user"]["id"]
    deployment_id = body["actions"][0]["value"]

    # Verify user has approval permission
    if not await can_approve_deployment(user, deployment_id):
        await client.chat_postEphemeral(
            channel=body["channel"]["id"],
            user=user,
            text="❌ You don't have permission to approve this deployment."
        )
        return

    # Update message to show approval
    await client.chat_update(
        channel=body["channel"]["id"],
        ts=body["message"]["ts"],
        blocks=format_approved_deployment_blocks(deployment_id, user)
    )

    # Execute deployment
    async with AmplifierSession(config=DEPLOY_CONFIG) as session:
        result = await session.execute(
            prompt="Execute approved deployment",
            context={
                "deployment_id": deployment_id,
                "approved_by": user
            }
        )

    # Post deployment result
    await client.chat_postMessage(
        channel=body["channel"]["id"],
        thread_ts=body["message"]["ts"],
        blocks=format_deployment_result_blocks(result)
    )


@app.action("rollback_deployment")
async def handle_rollback(ack, body, client):
    """Handle rollback button click during incident."""

    await ack()

    user = body["user"]["id"]
    rollback_info = json.loads(body["actions"][0]["value"])

    # Require confirmation for production
    if rollback_info["environment"] == "production":
        await show_rollback_confirmation_modal(client, body, rollback_info)
        return

    # Execute rollback
    await execute_rollback(client, body, rollback_info, user)
```

### Notification Service

```python
# services/notifications.py

class SlackNotificationService:
    """Send notifications to Slack channels."""

    def __init__(self, client: WebClient, config: NotificationConfig):
        self.client = client
        self.config = config

    async def notify_pr_opened(self, pr_data: dict) -> None:
        """Notify channel of new PR with AI summary."""

        channel = self.config.get_channel("pr_opened")
        if not channel:
            return

        # Generate AI summary if enabled
        summary = None
        if self.config.include_ai_summary:
            async with AmplifierSession(config=SUMMARY_CONFIG) as session:
                summary = await session.execute(
                    prompt="Summarize this PR for team review",
                    context={"pr": pr_data}
                )

        # Build notification blocks
        blocks = self._build_pr_opened_blocks(pr_data, summary)

        # Send notification
        await self.client.chat_postMessage(
            channel=channel,
            blocks=blocks
        )

    async def notify_incident(
        self,
        incident: dict,
        analysis: dict | None = None
    ) -> str:
        """Notify incidents channel and return message ts for threading."""

        channel = self.config.get_channel("incidents")

        blocks = self._build_incident_blocks(incident, analysis)

        response = await self.client.chat_postMessage(
            channel=channel,
            blocks=blocks
        )

        return response["ts"]

    async def notify_ci_failure(self, build_data: dict) -> None:
        """Notify of CI failure with AI diagnosis."""

        channel = self.config.get_channel("ci_failed")
        if not channel:
            return

        # Generate AI diagnosis
        diagnosis = None
        if self.config.include_ai_diagnosis:
            async with AmplifierSession(config=CI_DIAGNOSIS_CONFIG) as session:
                diagnosis = await session.execute(
                    prompt="Diagnose this CI failure",
                    context={"build": build_data}
                )

        blocks = self._build_ci_failure_blocks(build_data, diagnosis)

        await self.client.chat_postMessage(
            channel=channel,
            blocks=blocks
        )

    def _build_pr_opened_blocks(
        self,
        pr_data: dict,
        summary: dict | None
    ) -> list[dict]:
        """Build Slack blocks for PR opened notification."""

        blocks = [
            {
                "type": "header",
                "text": {
                    "type": "plain_text",
                    "text": f"📬 New PR: {pr_data['title']}"
                }
            },
            {
                "type": "section",
                "fields": [
                    {"type": "mrkdwn", "text": f"*Author:* <@{pr_data['author_slack']}>"},
                    {"type": "mrkdwn", "text": f"*Changes:* +{pr_data['additions']} / -{pr_data['deletions']}"},
                    {"type": "mrkdwn", "text": f"*Files:* {pr_data['changed_files']}"},
                    {"type": "mrkdwn", "text": f"*Branch:* `{pr_data['head_branch']}`"}
                ]
            }
        ]

        if summary:
            blocks.append({
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": f"*AI Summary:*\n{summary.response[:500]}"
                }
            })

        # Suggested reviewers
        if pr_data.get("suggested_reviewers"):
            reviewers = ", ".join(
                f"<@{r['slack_id']}>" for r in pr_data["suggested_reviewers"]
            )
            blocks.append({
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": f"*Suggested Reviewers:* {reviewers}"
                }
            })

        blocks.append({
            "type": "actions",
            "elements": [
                {
                    "type": "button",
                    "text": {"type": "plain_text", "text": "Review PR"},
                    "url": pr_data["url"],
                    "action_id": "view_pr"
                },
                {
                    "type": "button",
                    "text": {"type": "plain_text", "text": "AI Review"},
                    "action_id": "request_ai_review",
                    "value": json.dumps({"pr_url": pr_data["url"]})
                }
            ]
        })

        return blocks
```

### Context Builder

```python
# services/context.py

async def build_thread_context(
    client: WebClient,
    channel: str,
    thread_ts: str
) -> dict:
    """Build context from Slack thread history."""

    # Get thread messages
    response = await client.conversations_replies(
        channel=channel,
        ts=thread_ts,
        limit=20
    )

    messages = response.get("messages", [])

    # Extract relevant context
    context = {
        "thread_summary": summarize_thread(messages),
        "participants": extract_participants(messages),
        "mentioned_files": extract_file_mentions(messages),
        "mentioned_prs": extract_pr_mentions(messages),
        "mentioned_issues": extract_issue_mentions(messages)
    }

    return context


async def get_user_context(user_id: str) -> dict:
    """Get user-specific context for personalized responses."""

    # Get user's recent activity
    recent_prs = await get_user_recent_prs(user_id)
    recent_commits = await get_user_recent_commits(user_id)
    assigned_issues = await get_user_assigned_issues(user_id)

    # Get user's team and repositories
    team_info = await get_user_team(user_id)

    return {
        "recent_prs": recent_prs,
        "recent_commits": recent_commits,
        "assigned_issues": assigned_issues,
        "team": team_info.get("team"),
        "repositories": team_info.get("repositories"),
        "role": team_info.get("role")
    }
```

---

## Configuration

### App Manifest

```yaml
# manifest.yaml
display_information:
  name: Amplifier
  description: AI-powered code assistant
  background_color: "#4A154B"

features:
  bot_user:
    display_name: Amplifier
    always_online: true
  slash_commands:
    - command: /amplifier
      description: AI-powered code assistance
      usage_hint: "[review|search|explain|incident|deploy] [args]"

oauth_config:
  scopes:
    bot:
      - app_mentions:read
      - channels:history
      - channels:read
      - chat:write
      - commands
      - files:read
      - groups:history
      - groups:read
      - im:history
      - im:read
      - im:write
      - reactions:read
      - reactions:write
      - users:read
      - users:read.email

settings:
  event_subscriptions:
    bot_events:
      - app_mention
      - message.im
      - message.channels
      - reaction_added
  interactivity:
    is_enabled: true
  socket_mode_enabled: true
```

### Integration Configuration

```yaml
# .amplifier/integrations/slack.yaml
slack:
  enabled: true

  # Authentication
  auth:
    bot_token: ${SLACK_BOT_TOKEN}
    app_token: ${SLACK_APP_TOKEN}  # For socket mode
    signing_secret: ${SLACK_SIGNING_SECRET}

  # Channel mappings
  channels:
    code_review: "#code-review"
    incidents: "#incidents"
    deployments: "#deployments"
    ci_alerts: "#ci-alerts"
    general_support: "#engineering"

  # Feature configuration
  features:
    direct_messages: true
    channel_mentions: true
    slash_commands: true
    interactive_components: true

    # AI features per channel type
    ai_summaries:
      pr_notifications: true
      incident_alerts: true
      ci_failures: true

    # Incident response
    incidents:
      auto_create_channel: true
      channel_prefix: "inc-"
      invite_oncall: true
      post_runbook_links: true

  # Notification rules
  notifications:
    pr_opened:
      enabled: true
      channel: code_review
      include_ai_summary: true
      mention_suggested_reviewers: true
      min_lines_changed: 10  # Skip trivial PRs

    pr_merged:
      enabled: true
      channel: deployments
      include_changelog: true

    ci_failed:
      enabled: true
      channel: ci_alerts
      include_ai_diagnosis: true
      only_main_branch: true

    incident_triggered:
      enabled: true
      channel: incidents
      auto_investigate: true

  # Rate limiting
  rate_limits:
    mentions_per_minute: 30
    slash_commands_per_minute: 60
    notifications_per_minute: 20

  # User mapping
  user_mapping:
    source: ldap  # ldap | github | manual
    github_to_slack:
      johndoe: U01234567
      janedoe: U01234568
```

---

## Security

### Access Control

```python
# security/access.py

class SlackAccessControl:
    """Control access to Amplifier features in Slack."""

    def __init__(self, config: AccessConfig):
        self.config = config

    async def can_use_feature(
        self,
        user_id: str,
        feature: str,
        channel_id: str
    ) -> bool:
        """Check if user can use feature in channel."""

        # Get user info
        user_groups = await self.get_user_groups(user_id)
        channel_type = await self.get_channel_type(channel_id)

        # Check feature-specific rules
        rules = self.config.get_feature_rules(feature)

        # Allow if user is in allowed groups
        if rules.allowed_groups:
            if not any(g in user_groups for g in rules.allowed_groups):
                return False

        # Check channel restrictions
        if rules.allowed_channels:
            if channel_id not in rules.allowed_channels:
                return False

        # Check channel type restrictions
        if rules.allowed_channel_types:
            if channel_type not in rules.allowed_channel_types:
                return False

        return True

    async def can_approve_deployment(
        self,
        user_id: str,
        deployment: dict
    ) -> bool:
        """Check if user can approve deployment."""

        environment = deployment.get("environment")
        service = deployment.get("service")

        # Get approval rules for environment
        rules = self.config.get_approval_rules(environment)

        # Check if user is in approvers list
        user_groups = await self.get_user_groups(user_id)

        for required_group in rules.required_groups:
            if required_group in user_groups:
                return True

        # Check service-specific owners
        if service in rules.service_owners:
            if user_id in rules.service_owners[service]:
                return True

        return False
```

### Audit Logging

```python
# security/audit.py

class SlackAuditLogger:
    """Log all Slack interactions for audit."""

    async def log_interaction(
        self,
        interaction_type: str,
        user_id: str,
        channel_id: str,
        details: dict
    ) -> None:
        """Log Slack interaction."""

        await self.emit_event("slack:interaction", {
            "type": interaction_type,
            "user_id": user_id,
            "channel_id": channel_id,
            "timestamp": datetime.utcnow().isoformat(),
            "details": details
        })

    async def log_deployment_approval(
        self,
        user_id: str,
        deployment_id: str,
        approved: bool,
        reason: str | None = None
    ) -> None:
        """Log deployment approval decision."""

        await self.emit_event("slack:deployment_approval", {
            "user_id": user_id,
            "deployment_id": deployment_id,
            "approved": approved,
            "reason": reason,
            "timestamp": datetime.utcnow().isoformat()
        })
```

---

## Response Formatting

### Block Builders

```python
# formatters/blocks.py

def format_code_block(code: str, language: str = "") -> dict:
    """Format code as Slack code block."""
    return {
        "type": "section",
        "text": {
            "type": "mrkdwn",
            "text": f"```{language}\n{code}\n```"
        }
    }


def format_file_reference(file_path: str, line: int | None = None) -> str:
    """Format file reference with optional line number."""
    if line:
        return f"`{file_path}:{line}`"
    return f"`{file_path}`"


def format_error_response(error: str, suggestion: str | None = None) -> list[dict]:
    """Format error response blocks."""
    blocks = [
        {
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": f"❌ {error}"
            }
        }
    ]

    if suggestion:
        blocks.append({
            "type": "context",
            "elements": [
                {
                    "type": "mrkdwn",
                    "text": f"💡 {suggestion}"
                }
            ]
        })

    return blocks


def truncate_for_slack(text: str, max_length: int = 3000) -> str:
    """Truncate text to fit Slack's limits."""
    if len(text) <= max_length:
        return text
    return text[:max_length - 20] + "\n\n... (truncated)"
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
