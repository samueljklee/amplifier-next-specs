# integration-teams-bot

> **Priority**: P2 (Enhancement)
> **Status**: Draft
> **Module**: `amplifier-teams-bot`

## Overview

Microsoft Teams bot for enterprise team collaboration with Amplifier. Query codebases in team channels, get AI assistance in meetings, share code explanations, and integrate with enterprise workflows (Azure DevOps, SharePoint).

### Value Proposition

| Without | With |
|---------|------|
| Switch to terminal for AI help | Ask in Teams where you work |
| Share screenshots of code | Interactive code explanations |
| Manual incident triage | AI-assisted investigation |
| Siloed knowledge | Team-shared AI insights |

---

## Features

### 1. Channel Queries

Ask Amplifier questions in team channels.

```
┌─────────────────────────────────────────────────────────────────────┐
│  #engineering-backend                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  👤 Alice Chen                                          10:30 AM    │
│  @Amplifier How does the payment retry logic work?                  │
│                                                                      │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                      │
│  🤖 Amplifier                                           10:30 AM    │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ **Payment Retry Logic**                                         ││
│  │                                                                  ││
│  │ The payment system uses exponential backoff for retries:        ││
│  │                                                                  ││
│  │ 1. **Initial attempt** → If fails, wait 1s                      ││
│  │ 2. **Retry 1** → If fails, wait 2s                              ││
│  │ 3. **Retry 2** → If fails, wait 4s                              ││
│  │ 4. **Final retry** → If fails, raise alert                      ││
│  │                                                                  ││
│  │ 📄 Source: `src/payments/retry.ts:45`                           ││
│  │                                                                  ││
│  │ [View Code] [Ask Follow-up] [Share to Thread]                   ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                      │
│  👤 Bob Smith                                           10:32 AM    │
│  @Amplifier What happens if all retries fail?                       │
│                                                                      │
│  🤖 Amplifier                                           10:32 AM    │
│  When all retries are exhausted:                                    │
│  1. Transaction marked as `FAILED`                                  │
│  2. Alert sent to #payments-alerts                                  │
│  3. Customer notified via email                                     │
│  ...                                                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2. Direct Message Assistant

Private AI assistance via DM.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Chat with Amplifier                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  👤 You                                                 2:15 PM     │
│  I'm getting a weird error in the auth module. Can you help?        │
│                                                                      │
│  🤖 Amplifier                                           2:15 PM     │
│  I'd be happy to help! Can you share more details:                  │
│                                                                      │
│  1. What error message are you seeing?                              │
│  2. Which file/function is causing it?                              │
│  3. Any recent changes to the auth module?                          │
│                                                                      │
│  Or you can paste the error/stacktrace directly.                    │
│                                                                      │
│  👤 You                                                 2:16 PM     │
│  ```                                                                │
│  Error: Token expired at validateToken (auth/middleware.ts:45)      │
│  ```                                                                │
│                                                                      │
│  🤖 Amplifier                                           2:16 PM     │
│  I found the issue. Looking at `auth/middleware.ts:45`:             │
│                                                                      │
│  The token expiration check uses system time which can drift.       │
│  There's a known issue (#892) about this.                           │
│                                                                      │
│  **Quick fix:**                                                     │
│  Add clock skew tolerance in `config/auth.ts`:                      │
│  ```ts                                                              │
│  clockTolerance: 30 // seconds                                      │
│  ```                                                                │
│                                                                      │
│  [View Issue #892] [Apply Fix] [Explain More]                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3. Incident Response

AI-assisted incident investigation in dedicated channels.

```
┌─────────────────────────────────────────────────────────────────────┐
│  #incident-2025-01-15-payment-outage                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  🚨 PagerDuty                                           3:00 PM     │
│  ALERT: Payment processing failure rate > 50%                       │
│  Severity: P1 | Service: payment-api | Region: us-east-1            │
│                                                                      │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                      │
│  👤 On-Call Engineer                                    3:01 PM     │
│  @Amplifier /incident analyze                                       │
│                                                                      │
│  🤖 Amplifier                                           3:01 PM     │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 🔍 **Incident Analysis**                                        ││
│  │                                                                  ││
│  │ **Timeline:**                                                   ││
│  │ • 2:55 PM - Error rate started climbing                         ││
│  │ • 2:58 PM - Stripe API latency spike detected                   ││
│  │ • 3:00 PM - Alert triggered                                     ││
│  │                                                                  ││
│  │ **Probable Cause:**                                             ││
│  │ Stripe API returning 503 errors. Retry logic exhausting         ││
│  │ but circuit breaker not triggering.                             ││
│  │                                                                  ││
│  │ **Recent Changes:**                                             ││
│  │ • PR #234 merged 2 hours ago (payment retry logic)              ││
│  │ • No config changes in last 24h                                 ││
│  │                                                                  ││
│  │ **Recommended Actions:**                                        ││
│  │ 1. Enable circuit breaker: `config/payments.ts:12`              ││
│  │ 2. Check Stripe status page                                     ││
│  │ 3. Consider rollback of PR #234                                 ││
│  │                                                                  ││
│  │ [View Metrics] [Recent PRs] [Rollback PR #234]                  ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                      │
│  👤 On-Call Engineer                                    3:02 PM     │
│  @Amplifier Show me the circuit breaker config                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4. Meeting Assistant

AI assistance during Teams meetings.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Meeting: Architecture Review - Payment System                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  [Video Call in Progress]                                           │
│                                                                      │
│  ─────────────────────────────────────────────────────────────────  │
│  Meeting Chat                                                        │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                      │
│  👤 Alice                                               2:30 PM     │
│  @Amplifier Can you show the current payment flow diagram?          │
│                                                                      │
│  🤖 Amplifier                                           2:30 PM     │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ **Payment Flow** (generated from codebase)                      ││
│  │                                                                  ││
│  │  ┌─────────┐    ┌──────────┐    ┌─────────┐                    ││
│  │  │ Request │───▶│ Validate │───▶│ Process │                    ││
│  │  └─────────┘    └──────────┘    └────┬────┘                    ││
│  │                                       │                         ││
│  │                      ┌────────────────┼────────────────┐        ││
│  │                      ▼                ▼                ▼        ││
│  │                 ┌────────┐      ┌────────┐      ┌────────┐     ││
│  │                 │ Stripe │      │ PayPal │      │ Manual │     ││
│  │                 └────────┘      └────────┘      └────────┘     ││
│  │                                                                  ││
│  │ [Share to Meeting] [Download] [Update]                          ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                      │
│  👤 Bob                                                 2:32 PM     │
│  @Amplifier What's the error rate for each payment method?          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5. Azure DevOps Integration

Connect with Azure DevOps work items and pipelines.

```
┌─────────────────────────────────────────────────────────────────────┐
│  #project-alpha                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  👤 PM                                                  11:00 AM    │
│  @Amplifier What's the status of the payment feature (Epic #45)?    │
│                                                                      │
│  🤖 Amplifier                                           11:00 AM    │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ **Epic #45: Payment System Improvements**                       ││
│  │                                                                  ││
│  │ 📊 Progress: 67% complete (8/12 items done)                     ││
│  │                                                                  ││
│  │ **Completed:**                                                  ││
│  │ ✅ #101 - Retry logic implementation                            ││
│  │ ✅ #102 - Error handling improvements                           ││
│  │ ✅ #103 - Stripe v3 upgrade                                     ││
│  │ ... 5 more                                                      ││
│  │                                                                  ││
│  │ **In Progress:**                                                ││
│  │ 🔄 #108 - Circuit breaker (@alice)                              ││
│  │ 🔄 #109 - Metrics dashboard (@bob)                              ││
│  │                                                                  ││
│  │ **Blocked:**                                                    ││
│  │ ⚠️ #110 - PayPal integration (waiting on API access)            ││
│  │                                                                  ││
│  │ **Code Status:**                                                ││
│  │ • 3 PRs open, 2 ready for review                                ││
│  │ • Pipeline: All passing ✅                                       ││
│  │                                                                  ││
│  │ [View in Azure DevOps] [PRs] [Pipeline]                         ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Teams Bot Service                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Bot Framework SDK                          │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐        │   │
│  │  │ Channel  │ │ DM       │ │ Meeting  │ │ Adaptive │        │   │
│  │  │ Handler  │ │ Handler  │ │ Handler  │ │ Cards    │        │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘        │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                    ┌─────────┴─────────┐                            │
│                    │ Amplifier Client  │                            │
│                    └─────────┬─────────┘                            │
│                              │                                       │
│         ┌────────────────────┼────────────────────┐                 │
│         ▼                    ▼                    ▼                 │
│  ┌────────────┐      ┌────────────┐      ┌────────────┐            │
│  │ Amplifier  │      │ Azure      │      │ Graph      │            │
│  │ API Server │      │ DevOps API │      │ API        │            │
│  └────────────┘      └────────────┘      └────────────┘            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Bot Commands

| Command | Description | Example |
|---------|-------------|---------|
| `@Amplifier [question]` | Ask about codebase | `@Amplifier how does auth work?` |
| `@Amplifier /explain [code]` | Explain code snippet | `@Amplifier /explain \`\`\`code\`\`\`` |
| `@Amplifier /incident analyze` | Analyze incident | In incident channel |
| `@Amplifier /pr [number]` | PR summary | `@Amplifier /pr 234` |
| `@Amplifier /work-item [id]` | Work item status | `@Amplifier /work-item 45` |
| `@Amplifier /pipeline [name]` | Pipeline status | `@Amplifier /pipeline main` |
| `@Amplifier /help` | Show commands | |

---

## Implementation

### Bot Service (Node.js)

```typescript
// src/bot.ts
import {
  ActivityHandler,
  TurnContext,
  TeamsActivityHandler,
  CardFactory
} from 'botbuilder';
import { AmplifierClient } from './amplifier-client';
import { AdaptiveCardBuilder } from './cards';
import { CommandParser } from './commands';

export class AmplifierBot extends TeamsActivityHandler {
  private amplifier: AmplifierClient;
  private cardBuilder: AdaptiveCardBuilder;
  private commandParser: CommandParser;

  constructor() {
    super();

    this.amplifier = new AmplifierClient({
      apiUrl: process.env.AMPLIFIER_API_URL,
      apiKey: process.env.AMPLIFIER_API_KEY
    });

    this.cardBuilder = new AdaptiveCardBuilder();
    this.commandParser = new CommandParser();

    // Handle messages
    this.onMessage(async (context, next) => {
      await this.handleMessage(context);
      await next();
    });

    // Handle reactions
    this.onReactionsAdded(async (context, next) => {
      await this.handleReaction(context);
      await next();
    });
  }

  async handleMessage(context: TurnContext) {
    const text = context.activity.text?.trim() || '';

    // Remove bot mention
    const cleanText = this.removeMention(text);

    // Parse command
    const { command, args } = this.commandParser.parse(cleanText);

    switch (command) {
      case 'explain':
        await this.handleExplain(context, args);
        break;

      case 'incident':
        await this.handleIncident(context, args);
        break;

      case 'pr':
        await this.handlePR(context, args);
        break;

      case 'work-item':
        await this.handleWorkItem(context, args);
        break;

      case 'help':
        await this.handleHelp(context);
        break;

      default:
        // General question
        await this.handleQuestion(context, cleanText);
    }
  }

  async handleQuestion(context: TurnContext, question: string) {
    // Send typing indicator
    await context.sendActivity({ type: 'typing' });

    // Get team/channel context
    const teamContext = this.getTeamContext(context);

    // Execute query
    const result = await this.amplifier.execute({
      prompt: question,
      context: {
        type: 'teams_query',
        team: teamContext.teamName,
        channel: teamContext.channelName,
        user: context.activity.from.name
      }
    });

    // Build response card
    const card = this.cardBuilder.queryResponse({
      question,
      response: result.response,
      sources: result.sources
    });

    await context.sendActivity({
      attachments: [CardFactory.adaptiveCard(card)]
    });
  }

  async handleIncident(context: TurnContext, args: string[]) {
    const subCommand = args[0];

    if (subCommand === 'analyze') {
      // Get channel messages for context
      const messages = await this.getChannelMessages(context);

      const result = await this.amplifier.execute({
        prompt: `Analyze this incident based on the channel discussion:\n\n${messages}`,
        profile: 'enterprise-dev:incident',
        context: {
          type: 'incident_analysis',
          channel: context.activity.channelId
        }
      });

      const card = this.cardBuilder.incidentAnalysis(result.response);

      await context.sendActivity({
        attachments: [CardFactory.adaptiveCard(card)]
      });
    }
  }

  async handlePR(context: TurnContext, args: string[]) {
    const prNumber = args[0];

    const result = await this.amplifier.execute({
      prompt: `Summarize PR #${prNumber}`,
      context: {
        type: 'pr_summary',
        pr_number: prNumber
      }
    });

    const card = this.cardBuilder.prSummary({
      number: prNumber,
      summary: result.response
    });

    await context.sendActivity({
      attachments: [CardFactory.adaptiveCard(card)]
    });
  }

  async handleWorkItem(context: TurnContext, args: string[]) {
    const workItemId = args[0];

    // Get work item from Azure DevOps
    const workItem = await this.getAzureDevOpsWorkItem(workItemId);

    // Get AI analysis
    const result = await this.amplifier.execute({
      prompt: `Analyze the status of this work item:\n\n${JSON.stringify(workItem)}`,
      context: {
        type: 'work_item_analysis'
      }
    });

    const card = this.cardBuilder.workItemStatus({
      workItem,
      analysis: result.response
    });

    await context.sendActivity({
      attachments: [CardFactory.adaptiveCard(card)]
    });
  }
}
```

### Adaptive Card Builder

```typescript
// src/cards.ts
export class AdaptiveCardBuilder {
  queryResponse(data: QueryResponseData) {
    return {
      type: 'AdaptiveCard',
      $schema: 'http://adaptivecards.io/schemas/adaptive-card.json',
      version: '1.4',
      body: [
        {
          type: 'TextBlock',
          text: data.response,
          wrap: true
        },
        {
          type: 'FactSet',
          facts: data.sources?.map(s => ({
            title: '📄 Source',
            value: s.path
          })) || []
        }
      ],
      actions: [
        {
          type: 'Action.Submit',
          title: 'Ask Follow-up',
          data: { action: 'followup', context: data }
        },
        {
          type: 'Action.OpenUrl',
          title: 'View Code',
          url: data.sources?.[0]?.url
        }
      ]
    };
  }

  incidentAnalysis(analysis: string) {
    // Parse analysis into structured sections
    const sections = this.parseAnalysis(analysis);

    return {
      type: 'AdaptiveCard',
      $schema: 'http://adaptivecards.io/schemas/adaptive-card.json',
      version: '1.4',
      body: [
        {
          type: 'TextBlock',
          text: '🔍 **Incident Analysis**',
          size: 'large',
          weight: 'bolder'
        },
        {
          type: 'Container',
          items: [
            {
              type: 'TextBlock',
              text: '**Timeline**',
              weight: 'bolder'
            },
            {
              type: 'TextBlock',
              text: sections.timeline,
              wrap: true
            }
          ]
        },
        {
          type: 'Container',
          items: [
            {
              type: 'TextBlock',
              text: '**Probable Cause**',
              weight: 'bolder'
            },
            {
              type: 'TextBlock',
              text: sections.cause,
              wrap: true
            }
          ]
        },
        {
          type: 'Container',
          items: [
            {
              type: 'TextBlock',
              text: '**Recommended Actions**',
              weight: 'bolder'
            },
            {
              type: 'TextBlock',
              text: sections.actions,
              wrap: true
            }
          ]
        }
      ],
      actions: [
        {
          type: 'Action.OpenUrl',
          title: 'View Metrics',
          url: sections.metricsUrl
        },
        {
          type: 'Action.Submit',
          title: 'Rollback',
          data: { action: 'rollback', pr: sections.recentPR }
        }
      ]
    };
  }

  prSummary(data: PRSummaryData) {
    return {
      type: 'AdaptiveCard',
      $schema: 'http://adaptivecards.io/schemas/adaptive-card.json',
      version: '1.4',
      body: [
        {
          type: 'TextBlock',
          text: `**PR #${data.number}**`,
          size: 'large'
        },
        {
          type: 'TextBlock',
          text: data.summary,
          wrap: true
        }
      ],
      actions: [
        {
          type: 'Action.OpenUrl',
          title: 'View PR',
          url: `https://github.com/org/repo/pull/${data.number}`
        },
        {
          type: 'Action.Submit',
          title: 'Approve',
          data: { action: 'approve_pr', pr: data.number }
        }
      ]
    };
  }
}
```

---

## Deployment

### Azure Bot Service

```yaml
# azure-pipelines.yml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

stages:
  - stage: Build
    jobs:
      - job: BuildBot
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '18.x'

          - script: |
              npm ci
              npm run build
            displayName: 'Build'

          - task: ArchiveFiles@2
            inputs:
              rootFolderOrFile: '$(System.DefaultWorkingDirectory)'
              includeRootFolder: false
              archiveFile: '$(Build.ArtifactStagingDirectory)/bot.zip'

          - publish: $(Build.ArtifactStagingDirectory)/bot.zip
            artifact: bot

  - stage: Deploy
    jobs:
      - deployment: DeployBot
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'Azure-Sub'
                    appName: 'amplifier-teams-bot'
                    package: '$(Pipeline.Workspace)/bot/bot.zip'
```

### App Manifest

```json
// manifest.json
{
  "$schema": "https://developer.microsoft.com/json-schemas/teams/v1.14/MicrosoftTeams.schema.json",
  "manifestVersion": "1.14",
  "version": "1.0.0",
  "id": "{{BOT_ID}}",
  "packageName": "com.amplifier.teams",
  "developer": {
    "name": "Microsoft",
    "websiteUrl": "https://amplifier.dev",
    "privacyUrl": "https://amplifier.dev/privacy",
    "termsOfUseUrl": "https://amplifier.dev/terms"
  },
  "name": {
    "short": "Amplifier",
    "full": "Amplifier AI Assistant"
  },
  "description": {
    "short": "AI-powered code assistant for Teams",
    "full": "Query your codebase, get AI assistance, and collaborate with your team using Amplifier."
  },
  "icons": {
    "outline": "outline.png",
    "color": "color.png"
  },
  "accentColor": "#6366f1",
  "bots": [
    {
      "botId": "{{BOT_ID}}",
      "scopes": ["personal", "team", "groupchat"],
      "supportsFiles": false,
      "isNotificationOnly": false,
      "commandLists": [
        {
          "scopes": ["personal", "team", "groupchat"],
          "commands": [
            {
              "title": "help",
              "description": "Show available commands"
            },
            {
              "title": "explain",
              "description": "Explain code snippet"
            },
            {
              "title": "pr",
              "description": "Get PR summary"
            },
            {
              "title": "incident",
              "description": "Analyze incident"
            }
          ]
        }
      ]
    }
  ],
  "permissions": ["identity", "messageTeamMembers"],
  "validDomains": ["amplifier.dev", "*.amplifier.dev"]
}
```

---

## Configuration

```yaml
# Bot configuration
bot:
  app_id: ${AZURE_BOT_APP_ID}
  app_password: ${AZURE_BOT_APP_PASSWORD}

amplifier:
  api_url: https://api.amplifier.example.com
  api_key: ${AMPLIFIER_API_KEY}
  default_profile: enterprise-dev:teams

azure_devops:
  organization: myorg
  project: myproject
  pat: ${AZURE_DEVOPS_PAT}

features:
  channel_queries: true
  dm_assistant: true
  meeting_assistant: true
  incident_response: true
  azure_devops: true

permissions:
  # Who can use the bot
  allowed_teams: ["*"]  # or specific team IDs
  admin_users: ["user1@company.com"]
```

---

## Events

| Event | Description | Data |
|-------|-------------|------|
| `teams:query_received` | Query in channel/DM | team, channel, user |
| `teams:response_sent` | Response delivered | type, tokens |
| `teams:command_executed` | Slash command used | command, args |
| `teams:incident_analyzed` | Incident analysis done | channel, findings |
| `teams:pr_action` | PR action taken | action, pr_number |

---

## Security

```yaml
security:
  # Azure AD authentication
  auth:
    tenant_id: ${AZURE_AD_TENANT_ID}
    validate_issuer: true

  # Bot-specific
  bot:
    validate_activity: true
    allowed_callers: ["*.botframework.com"]

  # Data handling
  data:
    log_queries: true
    redact_sensitive: true
    retention_days: 30
```

---

## Open Questions

1. **Multi-tenant**: Support for multiple Azure AD tenants?
2. **Meeting transcripts**: Integrate with Teams meeting transcripts?
3. **Proactive messages**: Notify teams about CI/CD events?
4. **Message threading**: Reply in threads vs new messages?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
