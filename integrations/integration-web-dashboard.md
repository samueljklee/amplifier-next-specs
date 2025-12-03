# integration-web-dashboard

> **Priority**: P1 (High Value)
> **Status**: Draft
> **Module**: `amplifier-web-dashboard`

## Overview

Web-based dashboard for managing Amplifier: collections, profiles, sessions, analytics, and interactive chat. Provides visual management without requiring CLI knowledge. Built on top of the API Server.

### Value Proposition

| Without | With |
|---------|------|
| CLI-only management | Visual point-and-click interface |
| Manual profile editing | Profile builder with live preview |
| Log file inspection | Real-time analytics dashboard |
| Single-session terminal | Multi-session workspace |

---

## Features

### 1. Interactive Chat Workspace

Multi-panel chat interface with session management.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Amplifier Dashboard                          [user@email.com] [⚙️] │
├─────────────┬───────────────────────────────────────────────────────┤
│             │                                                       │
│  Sessions   │  Session: Code Review #42                    [Profile]│
│  ─────────  │  ───────────────────────────────────────────────────  │
│             │                                                       │
│  ● Active   │  User: Explain the payment processing flow           │
│    #42      │                                                       │
│    #38      │  Assistant: Based on my analysis of the codebase:    │
│             │                                                       │
│  ○ Recent   │  1. **Entry Point** (src/api/payments.ts:45)         │
│    #35      │     - Receives POST /api/payments                    │
│    #31      │                                                       │
│             │  2. **Validation** (src/services/PaymentValidator.ts)│
│  + New      │     - Card validation                                │
│             │     - Fraud detection                                │
│             │                                                       │
│             │  [Referenced Files]                                   │
│             │  • payments.ts • PaymentValidator.ts                  │
│             │                                                       │
│             │  ─────────────────────────────────────────────────── │
│             │  [Type your message...]                    [Send] [📎]│
│             │                                                       │
└─────────────┴───────────────────────────────────────────────────────┘
```

### 2. Collection Manager

Visual interface for browsing, installing, and managing collections.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Collections                                    [+ Install] [🔄]    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Installed (4)                                                      │
│  ──────────────────────────────────────────────────────────────────│
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ 📦 enterprise-dev                                    v1.2.0   │ │
│  │    Enterprise development tools and profiles                  │ │
│  │    Profiles: 3  Agents: 5  Tools: 8                          │ │
│  │    [View] [Update Available] [Remove]                         │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ 📦 foundation                                        v2.0.0   │ │
│  │    Core profiles and base configurations                      │ │
│  │    Profiles: 2  Agents: 3  Tools: 0                          │ │
│  │    [View] [Up to date]                                        │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  Available (12)                                          [Browse →] │
│  ──────────────────────────────────────────────────────────────────│
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3. Profile Builder

Visual profile editor with inheritance visualization.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Profile Builder: my-custom-profile                        [Save]   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Extends: [foundation:base ▼]                                       │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ Inheritance Chain                                               ││
│  │ foundation:base → enterprise-dev:development → my-custom        ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                     │
│  Providers                                                          │
│  ─────────────────────────────────────────────────────────────────  │
│  ┌──────────────────────────┐  ┌──────────────────────────┐        │
│  │ 🤖 provider-anthropic    │  │ + Add Provider           │        │
│  │    Model: claude-sonnet  │  │                          │        │
│  │    Temperature: 0.7      │  │                          │        │
│  │    [Configure] [Remove]  │  │                          │        │
│  └──────────────────────────┘  └──────────────────────────┘        │
│                                                                     │
│  Tools                                                              │
│  ─────────────────────────────────────────────────────────────────  │
│  [x] filesystem   [x] bash   [x] web   [ ] task   [+ Add Tool]     │
│                                                                     │
│  Hooks                                                              │
│  ─────────────────────────────────────────────────────────────────  │
│  [x] logging   [x] security-scan   [ ] audit-trail   [+ Add Hook]  │
│                                                                     │
│  Preview YAML                                              [Copy]   │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ extends: foundation:base                                        ││
│  │ providers:                                                      ││
│  │   - module: provider-anthropic                                  ││
│  │     config:                                                     ││
│  │       model: claude-sonnet-4-5                                  ││
│  └─────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
```

### 4. Analytics Dashboard

Real-time usage analytics and insights.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Analytics                                    [Last 7 days ▼] [🔄]  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│  │   1,247     │ │   45.2K     │ │   $127.50   │ │   2.3s      │   │
│  │  Sessions   │ │   Tokens    │ │   Cost      │ │   Avg Time  │   │
│  │   ↑ 12%     │ │   ↑ 8%      │ │   ↓ 5%      │ │   ↓ 15%     │   │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘   │
│                                                                     │
│  Usage Over Time                                                    │
│  ─────────────────────────────────────────────────────────────────  │
│  │                                                                  │
│  │     ╭──╮                       ╭────╮                           │
│  │  ╭──╯  ╰──╮               ╭────╯    ╰──╮                        │
│  │──╯        ╰───────────────╯            ╰────                    │
│  │                                                                  │
│  └──────────────────────────────────────────────────────────────── │
│    Mon   Tue   Wed   Thu   Fri   Sat   Sun                         │
│                                                                     │
│  Top Profiles                    │  Top Tools                       │
│  ────────────────────────────── │  ──────────────────────────────  │
│  1. enterprise-dev:dev    42%   │  1. codebase-search      35%     │
│  2. foundation:base       28%   │  2. filesystem           28%     │
│  3. custom:review         15%   │  3. bash                 22%     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 5. Session History & Search

Browse and search past sessions.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Session History                              [Search...] [Filter]  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Today                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ #47 • 10:30 AM • enterprise-dev:development                     ││
│  │ "How does the payment processing work?"                         ││
│  │ 12 messages • 4.5K tokens • [Resume] [Export]                   ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ #46 • 9:15 AM • foundation:base                                 ││
│  │ "Generate unit tests for UserService"                           ││
│  │ 8 messages • 2.1K tokens • [Resume] [Export]                    ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                     │
│  Yesterday                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ #45 • 4:45 PM • enterprise-dev:review                           ││
│  │ "Review PR #234 for security issues"                            ││
│  │ 15 messages • 8.2K tokens • [Resume] [Export]                   ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                     │
│  [Load More]                                                        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Web Dashboard                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    React/Next.js Frontend                     │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐        │   │
│  │  │ Chat     │ │Collection│ │ Profile  │ │Analytics │        │   │
│  │  │ Workspace│ │ Manager  │ │ Builder  │ │Dashboard │        │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘        │   │
│  │                           │                                   │   │
│  │                    ┌──────┴──────┐                           │   │
│  │                    │ API Client  │                           │   │
│  │                    │ (SDK)       │                           │   │
│  │                    └──────┬──────┘                           │   │
│  └───────────────────────────┼──────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│                    ┌─────────────────┐                              │
│                    │ Amplifier API   │                              │
│                    │ Server          │                              │
│                    └─────────────────┘                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

```yaml
frontend:
  framework: Next.js 14
  language: TypeScript
  styling: Tailwind CSS
  state: Zustand or React Query
  components: shadcn/ui
  charts: Recharts

api_client:
  type: Generated from OpenAPI
  websocket: Native WebSocket

deployment:
  options:
    - Vercel (recommended)
    - Docker + nginx
    - Static export + CDN
```

---

## Implementation

### Pages Structure

```
app/
├── page.tsx                 # Landing / redirect to dashboard
├── login/
│   └── page.tsx             # Authentication
├── dashboard/
│   ├── page.tsx             # Main dashboard (analytics)
│   ├── chat/
│   │   ├── page.tsx         # Chat workspace
│   │   └── [sessionId]/
│   │       └── page.tsx     # Specific session
│   ├── collections/
│   │   ├── page.tsx         # Collection manager
│   │   └── [name]/
│   │       └── page.tsx     # Collection details
│   ├── profiles/
│   │   ├── page.tsx         # Profile list
│   │   ├── new/
│   │   │   └── page.tsx     # Profile builder
│   │   └── [name]/
│   │       └── page.tsx     # Edit profile
│   ├── history/
│   │   └── page.tsx         # Session history
│   └── settings/
│       └── page.tsx         # User settings
└── api/
    └── [...proxy]/
        └── route.ts         # API proxy (optional)
```

### Chat Component

```tsx
// components/chat/ChatWorkspace.tsx
"use client";

import { useState, useEffect, useRef } from "react";
import { useSession } from "@/hooks/useSession";
import { MessageList } from "./MessageList";
import { ChatInput } from "./ChatInput";
import { SessionSidebar } from "./SessionSidebar";
import { ProfileSelector } from "./ProfileSelector";

export function ChatWorkspace() {
  const [sessionId, setSessionId] = useState<string | null>(null);
  const [messages, setMessages] = useState<Message[]>([]);
  const [isStreaming, setIsStreaming] = useState(false);
  const messagesEndRef = useRef<HTMLDivElement>(null);

  const { createSession, executePrompt, sessions } = useSession();

  const handleNewSession = async (profile: string) => {
    const session = await createSession(profile);
    setSessionId(session.id);
    setMessages([]);
  };

  const handleSendMessage = async (content: string) => {
    if (!sessionId) return;

    // Add user message
    setMessages((prev) => [...prev, { role: "user", content }]);
    setIsStreaming(true);

    // Stream response
    let assistantMessage = "";
    for await (const event of executePrompt(sessionId, content)) {
      if (event.type === "token") {
        assistantMessage += event.content;
        setMessages((prev) => {
          const updated = [...prev];
          const lastIdx = updated.length - 1;
          if (updated[lastIdx]?.role === "assistant") {
            updated[lastIdx].content = assistantMessage;
          } else {
            updated.push({ role: "assistant", content: assistantMessage });
          }
          return updated;
        });
      }
    }

    setIsStreaming(false);
  };

  return (
    <div className="flex h-screen">
      {/* Sidebar */}
      <SessionSidebar
        sessions={sessions}
        activeSessionId={sessionId}
        onSelectSession={setSessionId}
        onNewSession={() => setSessionId(null)}
      />

      {/* Main chat area */}
      <div className="flex-1 flex flex-col">
        {/* Header */}
        <div className="border-b p-4 flex justify-between items-center">
          <h2>
            {sessionId ? `Session #${sessionId.slice(0, 8)}` : "New Session"}
          </h2>
          <ProfileSelector
            onSelect={handleNewSession}
            disabled={!!sessionId}
          />
        </div>

        {/* Messages */}
        <div className="flex-1 overflow-y-auto p-4">
          <MessageList messages={messages} isStreaming={isStreaming} />
          <div ref={messagesEndRef} />
        </div>

        {/* Input */}
        <ChatInput
          onSend={handleSendMessage}
          disabled={!sessionId || isStreaming}
          placeholder={sessionId ? "Type your message..." : "Select a profile to start"}
        />
      </div>
    </div>
  );
}
```

### Collection Manager Component

```tsx
// components/collections/CollectionManager.tsx
"use client";

import { useCollections } from "@/hooks/useCollections";
import { CollectionCard } from "./CollectionCard";
import { InstallModal } from "./InstallModal";
import { useState } from "react";

export function CollectionManager() {
  const { collections, install, remove, checkUpdates } = useCollections();
  const [showInstall, setShowInstall] = useState(false);

  const handleInstall = async (source: string) => {
    await install(source);
    setShowInstall(false);
  };

  return (
    <div className="p-6">
      <div className="flex justify-between items-center mb-6">
        <h1 className="text-2xl font-bold">Collections</h1>
        <div className="flex gap-2">
          <button
            onClick={() => checkUpdates()}
            className="btn btn-secondary"
          >
            Check Updates
          </button>
          <button
            onClick={() => setShowInstall(true)}
            className="btn btn-primary"
          >
            + Install Collection
          </button>
        </div>
      </div>

      <section className="mb-8">
        <h2 className="text-lg font-semibold mb-4">
          Installed ({collections.length})
        </h2>
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
          {collections.map((collection) => (
            <CollectionCard
              key={collection.name}
              collection={collection}
              onRemove={() => remove(collection.name)}
            />
          ))}
        </div>
      </section>

      {showInstall && (
        <InstallModal
          onInstall={handleInstall}
          onClose={() => setShowInstall(false)}
        />
      )}
    </div>
  );
}
```

---

## Authentication

Integrates with API server authentication:

```tsx
// lib/auth.ts
import { signIn, signOut, useSession } from "next-auth/react";

export const authConfig = {
  providers: [
    // GitHub OAuth
    GitHubProvider({
      clientId: process.env.GITHUB_ID,
      clientSecret: process.env.GITHUB_SECRET,
    }),
    // API Key auth
    CredentialsProvider({
      name: "API Key",
      credentials: {
        apiKey: { label: "API Key", type: "password" }
      },
      async authorize(credentials) {
        // Validate API key against Amplifier API
        const res = await fetch(`${API_URL}/v1/auth/validate`, {
          headers: { "X-API-Key": credentials.apiKey }
        });
        if (res.ok) {
          return await res.json();
        }
        return null;
      }
    })
  ]
};
```

---

## Configuration

```yaml
# Dashboard configuration
dashboard:
  title: "Amplifier Dashboard"
  logo: "/logo.svg"

  api:
    url: "https://api.amplifier.example.com"
    # Or local
    # url: "http://localhost:8000"

  features:
    chat: true
    collections: true
    profiles: true
    analytics: true
    history: true

  theme:
    primary: "#6366f1"
    mode: "system"  # light | dark | system

  auth:
    providers: ["github", "api_key"]
    required: true
```

---

## Deployment

### Vercel (Recommended)

```bash
# Deploy to Vercel
vercel deploy

# Environment variables
NEXT_PUBLIC_API_URL=https://api.amplifier.example.com
NEXTAUTH_SECRET=xxx
GITHUB_ID=xxx
GITHUB_SECRET=xxx
```

### Docker

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

EXPOSE 3000
CMD ["node", "server.js"]
```

---

## Open Questions

1. **Self-hosted vs Cloud**: Should we offer a hosted version?
2. **Offline mode**: Support for local-only without API server?
3. **Team features**: Shared sessions, team profiles?
4. **Plugin system**: Allow dashboard extensions?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
