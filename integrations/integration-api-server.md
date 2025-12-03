# integration-api-server

> **Priority**: P0 (Foundation)
> **Status**: Draft
> **Module**: `amplifier-api-server`

## Overview

HTTP REST/GraphQL API server wrapping Amplifier capabilities. Enables any integration - web apps, automation platforms (Zapier, n8n), custom applications, and third-party tools. The foundation that enables all other integrations.

### Why P0?

This is foundational infrastructure. Once an API server exists:
- Web Dashboard can call it
- Browser Extension can call it
- Mobile apps can call it
- Any third-party integration becomes possible

### Value Proposition

| Without | With |
|---------|------|
| CLI-only access | HTTP API for any client |
| Local execution only | Remote/hosted execution |
| Manual integrations | Zapier/n8n/webhook automation |
| Single-user | Multi-tenant capable |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         API Server                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ REST API     │  │ GraphQL API  │  │ WebSocket    │              │
│  │ /v1/*        │  │ /graphql     │  │ /ws          │              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
│         │                 │                 │                       │
│         └─────────────────┼─────────────────┘                       │
│                           ▼                                         │
│                  ┌─────────────────┐                                │
│                  │ Request Router  │                                │
│                  └────────┬────────┘                                │
│                           │                                         │
│         ┌─────────────────┼─────────────────┐                       │
│         ▼                 ▼                 ▼                       │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐                  │
│  │ Auth       │   │ Rate       │   │ Usage      │                  │
│  │ Middleware │   │ Limiting   │   │ Tracking   │                  │
│  └────────────┘   └────────────┘   └────────────┘                  │
│                           │                                         │
│                           ▼                                         │
│                  ┌─────────────────┐                                │
│                  │ Amplifier Core  │                                │
│                  │ (Session Mgmt)  │                                │
│                  └─────────────────┘                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## API Design

### REST API

#### Sessions

```bash
# Create session
POST /v1/sessions
{
  "profile": "enterprise-dev:development",
  "context": {
    "project": "my-project",
    "branch": "feature/auth"
  }
}

Response:
{
  "session_id": "sess_abc123",
  "created_at": "2025-01-15T10:30:00Z",
  "profile": "enterprise-dev:development",
  "status": "active"
}

# Execute prompt
POST /v1/sessions/{session_id}/execute
{
  "prompt": "Explain the authentication flow",
  "stream": false
}

Response:
{
  "request_id": "req_xyz789",
  "response": "The authentication flow works as follows...",
  "usage": {
    "prompt_tokens": 150,
    "completion_tokens": 450,
    "total_tokens": 600
  },
  "tool_calls": [
    {"tool": "codebase-search", "query": "auth", "results": 12}
  ]
}

# Stream execution
POST /v1/sessions/{session_id}/execute
{
  "prompt": "Generate a detailed report",
  "stream": true
}

Response: Server-Sent Events
event: token
data: {"content": "The"}

event: token
data: {"content": " report"}

event: tool_call
data: {"tool": "search", "status": "started"}

event: complete
data: {"usage": {...}}

# Get session history
GET /v1/sessions/{session_id}/messages

# Close session
DELETE /v1/sessions/{session_id}
```

#### Profiles & Collections

```bash
# List profiles
GET /v1/profiles

Response:
{
  "profiles": [
    {"name": "foundation:base", "source": "bundled"},
    {"name": "enterprise-dev:development", "source": "collection"}
  ]
}

# List collections
GET /v1/collections

# Install collection
POST /v1/collections
{
  "source": "git+https://github.com/org/amplifier-collection-custom@main"
}
```

#### Tools

```bash
# List available tools
GET /v1/tools

# Execute tool directly (no session)
POST /v1/tools/{tool_name}/execute
{
  "input": {...}
}
```

### GraphQL API

```graphql
type Query {
  sessions: [Session!]!
  session(id: ID!): Session
  profiles: [Profile!]!
  collections: [Collection!]!
  tools: [Tool!]!
}

type Mutation {
  createSession(profile: String!, context: JSON): Session!
  execute(sessionId: ID!, prompt: String!): ExecutionResult!
  closeSession(sessionId: ID!): Boolean!
  installCollection(source: String!): Collection!
}

type Subscription {
  executionStream(sessionId: ID!, requestId: ID!): StreamEvent!
}

type Session {
  id: ID!
  profile: String!
  status: SessionStatus!
  messages: [Message!]!
  createdAt: DateTime!
}

type ExecutionResult {
  requestId: ID!
  response: String!
  usage: Usage!
  toolCalls: [ToolCall!]!
}

type StreamEvent {
  type: EventType!
  content: String
  toolCall: ToolCall
  usage: Usage
}
```

### WebSocket API

For real-time streaming and bidirectional communication:

```javascript
// Connect
ws = new WebSocket('wss://api.example.com/ws');

// Authenticate
ws.send(JSON.stringify({
  type: 'auth',
  token: 'api_key_xxx'
}));

// Create session
ws.send(JSON.stringify({
  type: 'session.create',
  profile: 'enterprise-dev:development'
}));

// Execute with streaming
ws.send(JSON.stringify({
  type: 'execute',
  session_id: 'sess_abc123',
  prompt: 'Analyze this code...'
}));

// Receive events
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  switch(data.type) {
    case 'token':
      // Streaming token
      break;
    case 'tool_call':
      // Tool execution update
      break;
    case 'complete':
      // Execution complete
      break;
  }
};
```

---

## Authentication & Authorization

### Authentication Methods

```yaml
auth:
  methods:
    # API Key (simple)
    - type: api_key
      header: X-API-Key

    # Bearer Token (JWT)
    - type: bearer
      issuer: https://auth.example.com
      audience: amplifier-api

    # OAuth2
    - type: oauth2
      provider: github
      scopes: [read:user, repo]
```

### Authorization Model

```yaml
authorization:
  # Role-based access
  roles:
    admin:
      - "*"
    developer:
      - "sessions:*"
      - "profiles:read"
      - "tools:execute"
    readonly:
      - "sessions:read"
      - "profiles:read"

  # Resource-level permissions
  resources:
    sessions:
      - create
      - read
      - execute
      - delete
    profiles:
      - read
      - write
    collections:
      - read
      - install
      - remove
```

---

## Rate Limiting & Quotas

```yaml
rate_limiting:
  # Per-user limits
  default:
    requests_per_minute: 60
    tokens_per_day: 100000
    concurrent_sessions: 5

  # Tier-based
  tiers:
    free:
      requests_per_minute: 10
      tokens_per_day: 10000
    pro:
      requests_per_minute: 100
      tokens_per_day: 500000
    enterprise:
      requests_per_minute: 1000
      tokens_per_day: unlimited

  # Response headers
  headers:
    X-RateLimit-Limit: "60"
    X-RateLimit-Remaining: "45"
    X-RateLimit-Reset: "1705312800"
```

---

## Implementation

### Server Setup (FastAPI)

```python
# server.py
from fastapi import FastAPI, Depends, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from amplifier_core import AmplifierSession
from .auth import verify_api_key, get_current_user
from .sessions import SessionManager
from .rate_limit import RateLimiter

app = FastAPI(
    title="Amplifier API",
    version="1.0.0",
    description="HTTP API for Amplifier AI Agent Framework"
)

# Middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"]
)

session_manager = SessionManager()
rate_limiter = RateLimiter()


@app.post("/v1/sessions")
async def create_session(
    request: CreateSessionRequest,
    user: User = Depends(get_current_user)
):
    """Create a new Amplifier session."""

    # Check rate limits
    await rate_limiter.check(user.id, "sessions:create")

    # Create session
    session = await session_manager.create(
        user_id=user.id,
        profile=request.profile,
        context=request.context
    )

    return SessionResponse(
        session_id=session.id,
        created_at=session.created_at,
        profile=request.profile,
        status="active"
    )


@app.post("/v1/sessions/{session_id}/execute")
async def execute(
    session_id: str,
    request: ExecuteRequest,
    user: User = Depends(get_current_user)
):
    """Execute a prompt in a session."""

    session = await session_manager.get(session_id, user.id)
    if not session:
        raise HTTPException(404, "Session not found")

    # Check rate limits
    await rate_limiter.check(user.id, "sessions:execute")

    if request.stream:
        return StreamingResponse(
            session.execute_stream(request.prompt),
            media_type="text/event-stream"
        )
    else:
        result = await session.execute(request.prompt)
        return ExecutionResponse(
            request_id=result.request_id,
            response=result.response,
            usage=result.usage,
            tool_calls=result.tool_calls
        )


@app.get("/v1/profiles")
async def list_profiles(user: User = Depends(get_current_user)):
    """List available profiles."""

    from amplifier_profiles import ProfileLoader

    loader = ProfileLoader(search_paths=get_search_paths(user))
    profiles = loader.list_profiles()

    return {"profiles": profiles}
```

### Session Manager

```python
# sessions.py
from amplifier_core import AmplifierSession
from amplifier_profiles import ProfileLoader
import asyncio
from typing import AsyncIterator

class SessionManager:
    """Manages Amplifier sessions for API users."""

    def __init__(self):
        self.sessions: dict[str, ManagedSession] = {}
        self.cleanup_task = asyncio.create_task(self._cleanup_loop())

    async def create(
        self,
        user_id: str,
        profile: str,
        context: dict | None = None
    ) -> ManagedSession:
        """Create a new managed session."""

        # Load profile
        loader = ProfileLoader(search_paths=self._get_paths(user_id))
        profile_config = loader.load_profile(profile)

        # Create Amplifier session
        amplifier_session = AmplifierSession(config=profile_config.to_mount_plan())
        await amplifier_session.__aenter__()

        # Wrap in managed session
        session = ManagedSession(
            id=generate_session_id(),
            user_id=user_id,
            amplifier_session=amplifier_session,
            created_at=datetime.utcnow()
        )

        self.sessions[session.id] = session
        return session

    async def get(self, session_id: str, user_id: str) -> ManagedSession | None:
        """Get session if owned by user."""

        session = self.sessions.get(session_id)
        if session and session.user_id == user_id:
            session.last_accessed = datetime.utcnow()
            return session
        return None

    async def close(self, session_id: str, user_id: str) -> bool:
        """Close a session."""

        session = await self.get(session_id, user_id)
        if session:
            await session.amplifier_session.__aexit__(None, None, None)
            del self.sessions[session_id]
            return True
        return False

    async def _cleanup_loop(self):
        """Clean up idle sessions."""

        while True:
            await asyncio.sleep(60)

            now = datetime.utcnow()
            idle_threshold = timedelta(minutes=30)

            for session_id, session in list(self.sessions.items()):
                if now - session.last_accessed > idle_threshold:
                    await self.close(session_id, session.user_id)


class ManagedSession:
    """Wrapper around AmplifierSession with metadata."""

    def __init__(
        self,
        id: str,
        user_id: str,
        amplifier_session: AmplifierSession,
        created_at: datetime
    ):
        self.id = id
        self.user_id = user_id
        self.amplifier_session = amplifier_session
        self.created_at = created_at
        self.last_accessed = created_at
        self.messages: list[dict] = []

    async def execute(self, prompt: str) -> ExecutionResult:
        """Execute prompt and return result."""

        self.messages.append({"role": "user", "content": prompt})

        response = await self.amplifier_session.execute(prompt)

        self.messages.append({"role": "assistant", "content": response.text})

        return ExecutionResult(
            request_id=generate_request_id(),
            response=response.text,
            usage=response.usage,
            tool_calls=response.tool_calls
        )

    async def execute_stream(self, prompt: str) -> AsyncIterator[str]:
        """Execute prompt with streaming."""

        self.messages.append({"role": "user", "content": prompt})

        full_response = ""

        async for event in self.amplifier_session.execute_stream(prompt):
            if event.type == "token":
                full_response += event.content
                yield f"event: token\ndata: {json.dumps({'content': event.content})}\n\n"
            elif event.type == "tool_call":
                yield f"event: tool_call\ndata: {json.dumps(event.data)}\n\n"

        self.messages.append({"role": "assistant", "content": full_response})

        yield f"event: complete\ndata: {json.dumps({'usage': event.usage})}\n\n"
```

---

## Deployment Options

### Docker

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "amplifier_api_server:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: amplifier-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: amplifier-api
  template:
    metadata:
      labels:
        app: amplifier-api
    spec:
      containers:
      - name: api
        image: amplifier-api-server:latest
        ports:
        - containerPort: 8000
        env:
        - name: AMPLIFIER_API_KEY
          valueFrom:
            secretKeyRef:
              name: amplifier-secrets
              key: api-key
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
---
apiVersion: v1
kind: Service
metadata:
  name: amplifier-api
spec:
  selector:
    app: amplifier-api
  ports:
  - port: 80
    targetPort: 8000
  type: LoadBalancer
```

---

## Configuration

```yaml
# config.yaml
server:
  host: 0.0.0.0
  port: 8000
  workers: 4

api:
  version: v1
  base_path: /v1

auth:
  type: api_key  # api_key | bearer | oauth2

rate_limiting:
  enabled: true
  backend: redis  # memory | redis
  redis_url: redis://localhost:6379

sessions:
  max_per_user: 10
  idle_timeout: 30m
  max_duration: 24h

logging:
  level: info
  format: json

metrics:
  enabled: true
  endpoint: /metrics

cors:
  allowed_origins: ["*"]
  allowed_methods: ["GET", "POST", "DELETE"]
```

---

## SDK Generation

The API is designed for automatic SDK generation:

```bash
# Generate OpenAPI spec
amplifier-api-server openapi > openapi.json

# Generate TypeScript SDK
npx openapi-typescript-codegen --input openapi.json --output ./sdk/typescript

# Generate Python SDK
openapi-python-client generate --path openapi.json --output ./sdk/python
```

---

## Events

| Event | Description | Data |
|-------|-------------|------|
| `api:request` | API request received | method, path, user_id |
| `api:response` | API response sent | status, duration_ms |
| `api:error` | API error occurred | error, status |
| `session:created` | Session created via API | session_id, user_id |
| `session:executed` | Prompt executed | session_id, tokens |
| `rate_limit:exceeded` | Rate limit hit | user_id, limit |

---

## Open Questions

1. **Multi-tenancy**: Shared infrastructure vs isolated?
2. **Billing integration**: How to track usage for billing?
3. **Webhook support**: Async notifications for long-running tasks?
4. **File uploads**: Support for file context (documents, images)?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
