# Skills, Agents & Plugins Design

**Status:** Design Complete
**Scope:** Desktop implementation (extensible to CLI)

---

## Overview

This document defines the architecture for Skills, Agents, and Plugins systems that extend amplifier capabilities while maintaining kernel stability.

### Design Principles
1. **Kernel Stability** - amplifier-core remains unchanged
2. **Modular Loading** - Skills/Agents/Plugins loaded as modules
3. **User Control** - All features togglable
4. **Claude Compatibility** - Same formats as Claude Code
5. **Local First** - All data stored locally

---

## Skills System

A skill is a reusable capability that extends AI functionality through natural language instructions. Skills are **model-invoked** (Claude decides when to use them).

### Directory Structure
```
~/.amplifier/skills/              # User-level (all projects)
  └── my-skill/
      ├── SKILL.md                # Required: Skill definition
      ├── scripts/                # Optional: Executable scripts
      ├── references/             # Optional: Documentation
      └── assets/                 # Optional: Templates, files

.amplifier/skills/                # Project-level (this project only)
  └── project-skill/
      └── SKILL.md
```

### SKILL.md Format
```markdown
---
name: skill-name
description: Brief description of what this skill does
allowed-tools: "Read, Write, Bash"  # Optional: Tool restrictions
version: 1.0.0
---

# Skill Name

## Instructions
Provide clear, step-by-step guidance for Claude.

## Examples
Show concrete examples of using this skill.
```

### Implementation

**Backend:** `sidecar/skills_manager.py`
```python
class Skill:
    name: str
    description: str
    skill_md: str
    allowed_tools: list[str]
    scripts: list[str]
    references: list[str]
    location: 'user' | 'project'
    enabled: bool

class SkillsManager:
    async def load_skills(project_path: str | None) -> list[Skill]
    async def enable_skill(name: str) -> None
    async def disable_skill(name: str) -> None
```

**Frontend:** Skills loaded and injected into system prompt when enabled.

---

## Agents System

An agent is a specialized AI persona with focused capabilities. Agents use **user-invoked** activation via `/agent` command.

### Directory Structure
```
~/.amplifier/agents/              # User-level
  └── my-agent.md

.amplifier/agents/                # Project-level
  └── project-agent.md
```

### Agent Format
```markdown
---
name: agent-name
description: What this agent specializes in
model: claude-opus-4-5-20251101  # Optional: Override model
tools: [read_file, write_file]   # Optional: Tool allowlist
temperature: 0.7                 # Optional: Override temp
---

# Agent Name

## Role
Define the agent's specialized role and expertise.

## Approach
Describe how the agent should approach problems.

## Constraints
List any limitations or boundaries.
```

### Activation
```
/agent code-reviewer    # Switch to code reviewer agent
/agent                  # Return to default assistant
```

---

## Plugins System

Plugins extend functionality with external integrations (MCP servers, API connections, etc.).

### Plugin Types
1. **MCP Plugins** - Model Context Protocol servers
2. **API Plugins** - External API integrations
3. **Tool Plugins** - Custom tool implementations

### Configuration
```yaml
# ~/.amplifier/settings.yaml
plugins:
  mcp:
    filesystem:
      command: "npx"
      args: ["-y", "@modelcontextprotocol/server-filesystem", "~"]
    github:
      command: "npx"
      args: ["-y", "@modelcontextprotocol/server-github"]
      env:
        GITHUB_TOKEN: ${GITHUB_TOKEN}
```

---

## Integration with amplifier-core

### Hook Points
```python
# Skills injected via system prompt hook
hooks.register("prompt:pre", inject_skills_prompt)

# Agent switching via command hook
hooks.register("command:pre", handle_agent_command)

# Plugin tools registered via mount
coordinator.mount("tools", mcp_tool, name=f"mcp_{server}_{tool}")
```

### Module Loading
Skills/Agents use the existing `LocalLoader` infrastructure:
```python
# Skills treated as loadable modules
loader.register("skill-*", SkillModule)
loader.register("agent-*", AgentModule)
```

---

## Feature Matrix

| Feature | Skills | Agents | Plugins |
|---------|--------|--------|---------|
| Activation | Model-invoked | User-invoked | Always active |
| Scope | User/Project | User/Project | Global |
| Tool Restrictions | Yes | Yes | No |
| Model Override | No | Yes | No |
| Custom Scripts | Yes | No | Yes |
| Reference Docs | Yes | No | No |

---

## Implementation Status

### Desktop
- [x] Skills manager basic implementation
- [x] Skills UI (enable/disable)
- [x] MCP plugins integration
- [ ] Agents system
- [ ] Agent switching UI
- [ ] Skills auto-discovery

### CLI
- [ ] Skills support
- [ ] Agents support
- [ ] Plugin system

---

## References

- Claude Code skills format: https://docs.anthropic.com/claude/docs/skills
- MCP specification: https://modelcontextprotocol.io/
