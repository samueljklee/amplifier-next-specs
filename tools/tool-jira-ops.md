# tool-jira-ops

> **Priority**: P0 (Foundation)
> **Status**: Draft
> **Module**: `amplifier-module-tool-jira-ops`

## Overview

Comprehensive JIRA operations including creating, updating, querying, and linking issues. Enables workflow automation and tight integration between code work and ticket management.

### Value Proposition

| Without | With |
|---------|------|
| Switch to browser, find project, create ticket, copy ID back | `create_issue("Bug in auth", type="bug")` returns PROJ-456 |
| Manual status updates forgotten during coding flow | Automatic status transitions as work progresses |
| Lost context between code and tickets | Bidirectional linking between commits/PRs and tickets |
| Searching JIRA's clunky UI | JQL queries from your workflow |

### Use Cases

1. **Issue creation**: Create tickets from code context (bugs found, tasks identified)
2. **Status management**: Transition tickets as work progresses
3. **Query & search**: Find relevant tickets by JQL, assignee, sprint, etc.
4. **Linking**: Link issues to PRs, commits, other issues
5. **Bulk operations**: Update multiple tickets (sprint planning, grooming)
6. **Time tracking**: Log work against tickets

---

## Contract

### Tool Definition

```python
TOOL_DEFINITION = {
    "name": "jira_ops",
    "description": """
    JIRA operations: create, update, query, and link issues.

    Operations:
    - create: Create new issue (bug, story, task, etc.)
    - update: Update issue fields (status, assignee, description, etc.)
    - transition: Move issue through workflow (To Do → In Progress → Done)
    - query: Search issues using JQL
    - get: Get single issue details
    - link: Link issues together or to external resources
    - comment: Add comment to issue
    - log_work: Log time against issue

    Use for ticket management without leaving your workflow.
    """,
    "parameters": {
        "type": "object",
        "properties": {
            "operation": {
                "type": "string",
                "enum": ["create", "update", "transition", "query", "get", "link", "comment", "log_work"],
                "description": "Operation to perform"
            },
            "issue_key": {
                "type": "string",
                "description": "Issue key (e.g., PROJ-123). Required for update/transition/get/link/comment/log_work."
            },
            "project": {
                "type": "string",
                "description": "Project key for create operation"
            },
            "issue_type": {
                "type": "string",
                "enum": ["bug", "story", "task", "epic", "subtask"],
                "description": "Issue type for create operation"
            },
            "summary": {
                "type": "string",
                "description": "Issue summary/title"
            },
            "description": {
                "type": "string",
                "description": "Issue description (supports JIRA markdown)"
            },
            "fields": {
                "type": "object",
                "description": "Additional fields to set (assignee, priority, labels, custom fields, etc.)"
            },
            "transition": {
                "type": "string",
                "description": "Transition name for transition operation (e.g., 'Start Progress', 'Done')"
            },
            "jql": {
                "type": "string",
                "description": "JQL query for query operation"
            },
            "link_type": {
                "type": "string",
                "description": "Link type (blocks, is blocked by, relates to, etc.)"
            },
            "link_target": {
                "type": "string",
                "description": "Target issue key or URL for linking"
            },
            "comment": {
                "type": "string",
                "description": "Comment text for comment operation"
            },
            "time_spent": {
                "type": "string",
                "description": "Time spent for log_work (e.g., '2h', '30m', '1d')"
            },
            "max_results": {
                "type": "integer",
                "default": 20,
                "description": "Maximum results for query operation"
            }
        },
        "required": ["operation"]
    }
}
```

### Input Schema

```python
@dataclass
class JiraOpsInput:
    operation: Operation                # create | update | transition | query | get | link | comment | log_work

    # Issue identification
    issue_key: str | None = None        # Required for most ops except create/query
    project: str | None = None          # Required for create

    # Issue content
    issue_type: IssueType | None = None
    summary: str | None = None
    description: str | None = None
    fields: dict | None = None          # Additional/custom fields

    # Operation-specific
    transition: str | None = None       # For transition op
    jql: str | None = None              # For query op
    link_type: str | None = None        # For link op
    link_target: str | None = None      # For link op
    comment: str | None = None          # For comment op
    time_spent: str | None = None       # For log_work op

    # Query options
    max_results: int = 20
```

### Output Schema

```python
@dataclass
class JiraOpsOutput:
    success: bool
    operation: str

    # For create/update/get operations
    issue: JiraIssue | None = None

    # For query operation
    issues: list[JiraIssue] | None = None
    total_count: int | None = None

    # For transition operation
    available_transitions: list[Transition] | None = None

    # Operation metadata
    message: str | None = None

@dataclass
class JiraIssue:
    key: str                            # PROJ-123
    id: str                             # Internal ID
    self_url: str                       # API URL

    # Core fields
    summary: str
    description: str | None
    issue_type: str
    status: str
    priority: str | None

    # People
    assignee: str | None
    reporter: str

    # Dates
    created: datetime
    updated: datetime
    resolved: datetime | None
    due_date: date | None

    # Organization
    project: str
    labels: list[str]
    components: list[str]
    sprint: str | None
    epic: str | None                    # Epic key if linked

    # Links
    links: list[IssueLink]
    subtasks: list[str]                 # Subtask keys
    parent: str | None                  # Parent key if subtask

    # Custom fields (project-specific)
    custom_fields: dict[str, Any]

    # Computed
    url: str                            # Browser URL

@dataclass
class IssueLink:
    type: str                           # "blocks", "is blocked by", etc.
    direction: str                      # "inward" | "outward"
    target_key: str
    target_summary: str
    target_status: str

@dataclass
class Transition:
    id: str
    name: str                           # "Start Progress", "Done", etc.
    to_status: str                      # Target status name
```

### Events Emitted

| Event | When | Data |
|-------|------|------|
| `tool:jira:start` | Operation begins | operation, issue_key, project |
| `tool:jira:api_call` | JIRA API call | endpoint, method, duration_ms |
| `tool:jira:created` | Issue created | issue_key, project, issue_type |
| `tool:jira:updated` | Issue updated | issue_key, fields_changed |
| `tool:jira:transitioned` | Status changed | issue_key, from_status, to_status |
| `tool:jira:complete` | Operation done | operation, success |
| `tool:jira:error` | Operation failed | error_type, message |

---

## Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        tool-jira-ops                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │  Operation  │───▶│    JIRA     │───▶│   Result    │         │
│  │  Dispatcher │    │   Client    │    │   Mapper    │         │
│  └─────────────┘    └──────┬──────┘    └─────────────┘         │
│                            │                                    │
│                            ▼                                    │
│                     ┌─────────────┐                             │
│                     │   Field     │                             │
│                     │  Resolver   │                             │
│                     └─────────────┘                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Internal Components

#### 1. Operation Dispatcher

```python
class OperationDispatcher:
    """Routes operations to appropriate handlers."""

    async def dispatch(self, input: JiraOpsInput) -> JiraOpsOutput:
        handlers = {
            "create": self._handle_create,
            "update": self._handle_update,
            "transition": self._handle_transition,
            "query": self._handle_query,
            "get": self._handle_get,
            "link": self._handle_link,
            "comment": self._handle_comment,
            "log_work": self._handle_log_work,
        }

        handler = handlers.get(input.operation)
        if not handler:
            raise ValueError(f"Unknown operation: {input.operation}")

        return await handler(input)

    async def _handle_create(self, input: JiraOpsInput) -> JiraOpsOutput:
        # Validate required fields
        if not input.project:
            raise ValueError("project required for create")
        if not input.summary:
            raise ValueError("summary required for create")

        # Resolve field IDs (custom fields, etc.)
        fields = await self.field_resolver.resolve(input.project, {
            "project": {"key": input.project},
            "issuetype": {"name": input.issue_type or "Task"},
            "summary": input.summary,
            "description": input.description,
            **(input.fields or {})
        })

        # Create issue
        result = await self.client.create_issue(fields)

        return JiraOpsOutput(
            success=True,
            operation="create",
            issue=self.result_mapper.map_issue(result),
            message=f"Created {result['key']}"
        )

    async def _handle_transition(self, input: JiraOpsInput) -> JiraOpsOutput:
        if not input.issue_key:
            raise ValueError("issue_key required for transition")

        # Get available transitions
        transitions = await self.client.get_transitions(input.issue_key)

        if not input.transition:
            # Just return available transitions
            return JiraOpsOutput(
                success=True,
                operation="transition",
                available_transitions=[
                    Transition(id=t["id"], name=t["name"], to_status=t["to"]["name"])
                    for t in transitions
                ]
            )

        # Find matching transition
        transition_id = None
        for t in transitions:
            if t["name"].lower() == input.transition.lower():
                transition_id = t["id"]
                break

        if not transition_id:
            available = [t["name"] for t in transitions]
            raise ValueError(f"Transition '{input.transition}' not available. Available: {available}")

        # Execute transition
        await self.client.transition_issue(input.issue_key, transition_id)

        # Get updated issue
        issue = await self.client.get_issue(input.issue_key)

        return JiraOpsOutput(
            success=True,
            operation="transition",
            issue=self.result_mapper.map_issue(issue),
            message=f"Transitioned {input.issue_key} to {input.transition}"
        )
```

#### 2. Field Resolver

```python
class FieldResolver:
    """
    Resolves human-friendly field names to JIRA field IDs.
    Handles custom fields and project-specific configurations.
    """

    def __init__(self, client: JiraClient):
        self.client = client
        self._field_cache: dict[str, dict] = {}

    async def resolve(self, project: str, fields: dict) -> dict:
        """Convert user-provided fields to JIRA API format."""

        # Get field definitions for project
        if project not in self._field_cache:
            self._field_cache[project] = await self._load_fields(project)

        field_defs = self._field_cache[project]
        resolved = {}

        for key, value in fields.items():
            # Standard field
            if key in STANDARD_FIELDS:
                resolved[key] = self._resolve_standard(key, value)

            # Custom field by name
            elif key in field_defs:
                field_id = field_defs[key]["id"]
                resolved[field_id] = self._resolve_custom(field_defs[key], value)

            # Custom field by ID
            elif key.startswith("customfield_"):
                resolved[key] = value

            else:
                raise ValueError(f"Unknown field: {key}")

        return resolved

    def _resolve_standard(self, key: str, value: Any) -> Any:
        """Handle standard field formatting."""

        # Assignee/reporter: accept username string
        if key in ("assignee", "reporter"):
            if isinstance(value, str):
                return {"name": value}
            return value

        # Priority: accept name string
        if key == "priority":
            if isinstance(value, str):
                return {"name": value}
            return value

        # Labels: ensure list
        if key == "labels":
            if isinstance(value, str):
                return [value]
            return value

        return value
```

#### 3. JIRA Client

```python
class JiraClient:
    """
    Low-level JIRA REST API client.
    Handles authentication, rate limiting, and error handling.
    """

    def __init__(self, config: JiraConfig):
        self.base_url = config.base_url.rstrip("/")
        self.auth = (config.username, config.api_token) if config.username else None
        self.headers = {
            "Content-Type": "application/json",
            "Accept": "application/json"
        }
        if config.api_token and not config.username:
            self.headers["Authorization"] = f"Bearer {config.api_token}"

    async def create_issue(self, fields: dict) -> dict:
        return await self._request("POST", "/rest/api/2/issue", json={"fields": fields})

    async def update_issue(self, key: str, fields: dict) -> None:
        await self._request("PUT", f"/rest/api/2/issue/{key}", json={"fields": fields})

    async def get_issue(self, key: str, expand: list[str] | None = None) -> dict:
        params = {}
        if expand:
            params["expand"] = ",".join(expand)
        return await self._request("GET", f"/rest/api/2/issue/{key}", params=params)

    async def search(self, jql: str, max_results: int = 50, fields: list[str] | None = None) -> dict:
        body = {
            "jql": jql,
            "maxResults": max_results,
            "fields": fields or ["*all"]
        }
        return await self._request("POST", "/rest/api/2/search", json=body)

    async def get_transitions(self, key: str) -> list[dict]:
        result = await self._request("GET", f"/rest/api/2/issue/{key}/transitions")
        return result["transitions"]

    async def transition_issue(self, key: str, transition_id: str, fields: dict | None = None) -> None:
        body = {"transition": {"id": transition_id}}
        if fields:
            body["fields"] = fields
        await self._request("POST", f"/rest/api/2/issue/{key}/transitions", json=body)

    async def add_comment(self, key: str, body: str) -> dict:
        return await self._request("POST", f"/rest/api/2/issue/{key}/comment", json={"body": body})

    async def add_worklog(self, key: str, time_spent: str, comment: str | None = None) -> dict:
        body = {"timeSpent": time_spent}
        if comment:
            body["comment"] = comment
        return await self._request("POST", f"/rest/api/2/issue/{key}/worklog", json=body)

    async def create_link(self, link_type: str, inward_key: str, outward_key: str) -> None:
        body = {
            "type": {"name": link_type},
            "inwardIssue": {"key": inward_key},
            "outwardIssue": {"key": outward_key}
        }
        await self._request("POST", "/rest/api/2/issueLink", json=body)

    async def _request(self, method: str, path: str, **kwargs) -> Any:
        url = f"{self.base_url}{path}"
        async with httpx.AsyncClient() as client:
            response = await client.request(
                method, url,
                auth=self.auth,
                headers=self.headers,
                **kwargs
            )

            if response.status_code >= 400:
                error_body = response.json() if response.content else {}
                raise JiraAPIError(
                    status_code=response.status_code,
                    messages=error_body.get("errorMessages", []),
                    errors=error_body.get("errors", {})
                )

            if response.content:
                return response.json()
            return None
```

---

## Configuration

```yaml
tool-jira-ops:
  # Connection
  base_url: "${JIRA_URL}"               # https://company.atlassian.net

  # Authentication (choose one)
  # Option 1: API token (Atlassian Cloud)
  username: "${JIRA_USERNAME}"          # email for cloud
  api_token: "${JIRA_API_TOKEN}"

  # Option 2: Personal Access Token (Server/Data Center)
  # api_token: "${JIRA_PAT}"            # PAT without username

  # Option 3: OAuth (advanced)
  # oauth:
  #   consumer_key: "..."
  #   access_token: "..."
  #   ...

  # Default project
  default_project: "PROJ"               # Used when project not specified

  # Field mappings (project-specific custom field names)
  field_aliases:
    story_points: "customfield_10001"
    sprint: "customfield_10002"
    team: "customfield_10003"

  # Workflow shortcuts
  workflow_aliases:
    start: "Start Progress"
    done: "Done"
    review: "Ready for Review"
    block: "Blocked"

  # Query defaults
  query:
    default_fields:
      - summary
      - status
      - assignee
      - priority
      - created
      - updated
    max_results: 50

  # Behavior
  behavior:
    # Auto-assign to current user on transition to "In Progress"
    auto_assign_on_start: true

    # Add comment with PR link when linking
    comment_on_link: true

    # Validate transitions before attempting
    validate_transitions: true
```

---

## Examples

### Example 1: Create Bug from Code Context

```python
# Input
{
    "operation": "create",
    "project": "PROJ",
    "issue_type": "bug",
    "summary": "NullPointerException in PaymentGateway.process()",
    "description": "Found while reviewing PR #456.\n\n{code:java}\n// Line 142 in PaymentGateway.java\nuser.getEmail().toLowerCase()  // user can be null\n{code}\n\nReproduction: Call process() with null user.",
    "fields": {
        "priority": "High",
        "labels": ["backend", "payments"],
        "components": ["payment-service"]
    }
}

# Output
{
    "success": true,
    "operation": "create",
    "issue": {
        "key": "PROJ-789",
        "summary": "NullPointerException in PaymentGateway.process()",
        "status": "To Do",
        "priority": "High",
        "url": "https://company.atlassian.net/browse/PROJ-789"
    },
    "message": "Created PROJ-789"
}
```

### Example 2: Transition with Auto-Assign

```python
# Input
{
    "operation": "transition",
    "issue_key": "PROJ-789",
    "transition": "Start Progress"
}

# Output
{
    "success": true,
    "operation": "transition",
    "issue": {
        "key": "PROJ-789",
        "status": "In Progress",
        "assignee": "current-user"  // Auto-assigned
    },
    "message": "Transitioned PROJ-789 to Start Progress"
}
```

### Example 3: Query My Sprint Work

```python
# Input
{
    "operation": "query",
    "jql": "assignee = currentUser() AND sprint in openSprints() ORDER BY priority DESC",
    "max_results": 10
}

# Output
{
    "success": true,
    "operation": "query",
    "issues": [
        {
            "key": "PROJ-456",
            "summary": "Implement retry logic",
            "status": "In Progress",
            "priority": "High"
        },
        {
            "key": "PROJ-457",
            "summary": "Add unit tests for auth",
            "status": "To Do",
            "priority": "Medium"
        }
    ],
    "total_count": 2
}
```

### Example 4: Link PR to Issue

```python
# Input
{
    "operation": "link",
    "issue_key": "PROJ-456",
    "link_type": "is implemented by",
    "link_target": "https://github.com/company/repo/pull/123"
}

# Output
{
    "success": true,
    "operation": "link",
    "message": "Linked PROJ-456 to PR #123"
}
```

### Example 5: Get Available Transitions

```python
# Input
{
    "operation": "transition",
    "issue_key": "PROJ-456"
    // No transition specified - returns available options
}

# Output
{
    "success": true,
    "operation": "transition",
    "available_transitions": [
        {"id": "21", "name": "Done", "to_status": "Done"},
        {"id": "31", "name": "Blocked", "to_status": "Blocked"},
        {"id": "41", "name": "Ready for Review", "to_status": "In Review"}
    ]
}
```

---

## Security Considerations

### Authentication

- API tokens stored securely (environment variables or secret manager)
- Tokens never logged or returned in output
- Support for multiple auth methods (basic, PAT, OAuth)

### Permissions

- Tool respects JIRA project permissions
- Operations fail gracefully if user lacks permission
- No privilege escalation possible

### Audit

- All operations logged with user context
- JIRA maintains its own audit log

### Risk Mitigations

| Risk | Mitigation |
|------|------------|
| Credential exposure | Tokens from env vars, never in config files |
| Unauthorized modifications | API enforces JIRA permissions |
| Data leakage | Issue content may contain sensitive data - same as viewing in JIRA |
| Rate limiting | Built-in rate limit handling, exponential backoff |

---

## Testing Strategy

### Unit Tests

```python
def test_field_resolver_handles_standard_fields():
    resolver = FieldResolver(mock_client)
    result = await resolver.resolve("PROJ", {
        "assignee": "alice",
        "priority": "High"
    })
    assert result["assignee"] == {"name": "alice"}
    assert result["priority"] == {"name": "High"}

def test_operation_dispatcher_validates_required_fields():
    dispatcher = OperationDispatcher(...)
    with pytest.raises(ValueError, match="project required"):
        await dispatcher.dispatch(JiraOpsInput(operation="create", summary="Test"))
```

### Integration Tests

```python
async def test_create_and_transition_flow(mock_jira_server):
    tool = JiraOpsTool(config)

    # Create
    create_result = await tool.execute({
        "operation": "create",
        "project": "TEST",
        "summary": "Integration test issue"
    })
    assert create_result.success
    issue_key = create_result.issue.key

    # Transition
    transition_result = await tool.execute({
        "operation": "transition",
        "issue_key": issue_key,
        "transition": "Start Progress"
    })
    assert transition_result.success
    assert transition_result.issue.status == "In Progress"
```

---

## Dependencies

### Required

- `httpx` - Async HTTP client
- `jira` (optional) - Can use official JIRA library or direct REST

### Optional

- None

---

## Open Questions

1. **Bulk operations**: Should we support bulk create/update?
   - Useful for sprint planning automation
   - API supports it but adds complexity

2. **Webhook integration**: Should this tool support receiving JIRA webhooks?
   - Would enable reactive workflows
   - Significantly increases scope

3. **Custom workflow support**: How to handle project-specific workflows?
   - Currently: Discover transitions dynamically
   - Could: Pre-configure workflow mappings

4. **Attachment support**: Should we support uploading attachments?
   - Useful for screenshots, logs
   - Requires file handling

5. **Board/Sprint operations**: Should we include board-level operations?
   - Sprint management, backlog ordering
   - Different API (Agile API vs Issue API)

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
