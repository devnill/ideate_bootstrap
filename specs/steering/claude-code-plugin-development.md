# Claude Code Plugin Development Reference

**Generated**: March 7, 2026
**Source**: Official Anthropic / Claude Code documentation
**Purpose**: Comprehensive reference for building plugins, skills, agents, and tooling for Claude Code

---

## Table of Contents

1. [Core Concepts Overview](#1-core-concepts-overview)
2. [Skills System](#2-skills-system)
3. [Slash Commands](#3-slash-commands)
4. [Plugins System](#4-plugins-system)
5. [Plugin Marketplaces](#5-plugin-marketplaces)
6. [Agents and Subagents](#6-agents-and-subagents)
7. [Agent Teams](#7-agent-teams)
8. [Hooks System](#8-hooks-system)
9. [MCP (Model Context Protocol)](#9-mcp-model-context-protocol)
10. [CLAUDE.md Files](#10-claudemd-files)
11. [Settings and Configuration](#11-settings-and-configuration)
12. [Agent SDK](#12-agent-sdk)
13. [Packaging and Distribution](#13-packaging-and-distribution)
14. [Integration Patterns](#14-integration-patterns)

---

## 1. Core Concepts Overview

Claude Code is extensible through several layered systems:

| Component | What It Is | How Invoked |
|-----------|-----------|------------|
| **Skill** | Reusable Markdown-defined prompt/task | `/skill-name` or automatically by Claude |
| **Agent** | Specialized Claude instance with defined purpose | Delegation from parent Claude or Agent tool |
| **Plugin** | Packaged bundle of skills, agents, hooks, MCP servers | `/plugin install` |
| **Marketplace** | Catalog of installable plugins | `/plugin marketplace add` |
| **Hook** | Shell command run at lifecycle events | Automatic on event (PreToolUse, PostToolUse, etc.) |
| **MCP Server** | External service exposing tools/resources to Claude | Automatic tool access |
| **CLAUDE.md** | Persistent instruction file loaded at session start | Automatic context load |

All components interact: skills can invoke subagents, plugins can bundle hooks and MCP servers, hooks can block or modify tool calls, and CLAUDE.md sets the baseline context that all components inherit.

---

## 2. Skills System

### What Are Skills?

Skills are reusable, prompt-based extensions that teach Claude to perform specific tasks. They follow the [Agent Skills](https://agentskills.io) open standard with Claude-specific extensions.

- Written in Markdown with YAML frontmatter
- Invocable by users (`/skill-name`) or automatically by Claude based on context
- Support arguments and dynamic context injection
- Can run inline or in isolated subagent contexts
- Can ship with supporting files: templates, scripts, examples

### File Structure

```
.claude/skills/my-skill/
├── SKILL.md                 # Main instructions (required)
├── reference.md             # API docs, referenced from SKILL.md (optional)
├── examples.md              # Usage examples (optional)
├── templates/               # Templates Claude fills in (optional)
├── scripts/                 # Helper scripts Claude executes (optional)
└── data/                    # Supporting data files (optional)
```

### SKILL.md Format

```yaml
---
name: skill-name                          # kebab-case, lowercase, max 64 chars
description: What this does and when to use it
disable-model-invocation: true            # Optional: only user can invoke (not auto)
user-invocable: true                      # Optional: show in menu (default true)
argument-hint: "[argument description]"   # Optional: autocomplete hint
allowed-tools: Read, Grep, Glob          # Optional: limit tool access in this skill
model: sonnet                             # Optional: override default model
context: fork                             # Optional: "fork" runs in isolated subagent
agent: Explore                            # Optional: which subagent type to use
hooks:                                    # Optional: lifecycle event handlers
  SessionStart: [...]
---

# Skill Instructions

Detailed instructions. Available placeholders:
- $ARGUMENTS        - all arguments passed
- $ARGUMENTS[0]     - first argument
- $0, $1, $2        - shorthand for $ARGUMENTS[N]
- ${CLAUDE_SESSION_ID}  - current session ID
- ${CLAUDE_SKILL_DIR}   - path to skill directory
```

### Skill Scopes and Discovery

| Location | Scope | Path |
|----------|-------|------|
| Enterprise | Organization-wide | `/Library/Application Support/ClaudeCode/` (macOS) |
| Personal | All projects | `~/.claude/skills/<skill-name>/SKILL.md` |
| Project | Single project | `.claude/skills/<skill-name>/SKILL.md` |
| Plugin | When plugin enabled | `<plugin>/skills/<skill-name>/SKILL.md` (namespaced) |
| Nested | Subdirectories | `packages/frontend/.claude/skills/` |

**Precedence**: enterprise > personal > project

**Discovery**: At session start, Claude walks up from working directory discovering `.claude/skills/` directories. Subdirectory skills are lazy-loaded when Claude opens files in those directories.

### Built-in Bundled Skills

| Skill | Purpose |
|-------|---------|
| `/simplify` | Reviews changed code for reuse, quality, efficiency |
| `/batch` | Orchestrates large-scale parallel changes |
| `/debug` | Troubleshoots current session issues |
| `/loop 5m <prompt>` | Runs prompts repeatedly on an interval |
| `/claude-api` | Loads Claude API/Agent SDK reference |

### Advanced Skill Patterns

**Dynamic context injection** (`!` command syntax — runs before Claude sees the skill):

```yaml
---
name: pr-review
---

PR information:
- Diff: !`gh pr diff`
- Comments: !`gh pr view --comments`
- Files: !`gh pr diff --name-only`

Review this pull request with the above context...
```

**Running in isolated subagent** (`context: fork`):

```yaml
---
name: research-topic
description: Research a topic deeply without touching current files
context: fork
agent: Explore
allowed-tools: Bash(rg *), Read, Glob, Grep
---

Research $ARGUMENTS thoroughly...
```

**Preventing auto-invocation** (only explicit user command):

```yaml
---
name: deploy
disable-model-invocation: true
---

Deploy to production...
```

**Hidden from user menu** (Claude-only auto-invoke):

```yaml
---
name: background-context
user-invocable: false
---

Background knowledge about legacy system...
```

**Tool restriction within skills**:

```yaml
---
name: safe-explorer
allowed-tools: Read, Grep, Glob
---

Explore code in read-only mode...
```

---

## 3. Slash Commands

### Built-in Commands (Fixed Logic)

| Command | Purpose |
|---------|---------|
| `/help` | Show help |
| `/memory` | View/edit CLAUDE.md and auto memory |
| `/hooks` | Manage lifecycle hooks |
| `/plugin` | Plugin manager |
| `/mcp` | MCP server management |
| `/agents` | Subagent configuration |
| `/context` | Show context usage |
| `/cost` | Show token/cost usage |
| `/clear` | Clear session history |
| `/compact` | Compress context |
| `/init` | Generate CLAUDE.md |
| `/status` | Show status info |

### Custom Commands (Skills-Based)

Custom slash commands are now **skills**. Both old and new formats work:

```
.claude/
├── commands/
│   └── review.md           # Old format: → /review
├── skills/
│   └── review/
│       └── SKILL.md        # Preferred: → /review
```

When same name exists in both, `skills/` takes precedence.

### MCP Prompts as Commands

MCP servers can expose prompts that become slash commands:

```bash
/mcp__servername__promptname           # Basic invocation
/mcp__github__pr_review 456            # With arguments
/mcp__jira__create_issue "Bug" high    # Multiple args
```

### Plugin-Namespaced Commands

Skills in plugins get prefixed to prevent conflicts:

```bash
/my-plugin:skill-name
/coding-tools:format
```

---

## 4. Plugins System

### What Are Plugins?

Plugins are packaged collections of skills, agents, hooks, and MCP servers. They:
- **Namespace commands** to prevent conflicts (`/plugin-name:skill`)
- **Bundle related tools** cohesively
- **Enable sharing** via marketplaces
- **Version independently** with semantic versioning

### Plugin Directory Structure

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json                  # Manifest (required)
├── skills/
│   └── review/
│       ├── SKILL.md
│       └── examples.md
├── agents/
│   ├── security-reviewer.md
│   └── compliance-checker.md
├── commands/
│   └── deploy.md
├── hooks/
│   └── hooks.json
├── .mcp.json                        # MCP server configs
├── .lsp.json                        # LSP server configs
├── settings.json                    # Default settings contributed by plugin
└── README.md
```

> **Important**: Component directories live at **plugin root**, NOT inside `.claude-plugin/`. Only `plugin.json` goes inside `.claude-plugin/`.

### Plugin Manifest (`plugin.json`)

```json
{
  "name": "my-plugin",
  "description": "What this plugin does",
  "version": "1.0.0",
  "author": {
    "name": "Author Name",
    "email": "email@example.com"
  },
  "homepage": "https://github.com/user/my-plugin",
  "repository": "https://github.com/user/my-plugin",
  "license": "MIT",
  "keywords": ["productivity", "automation"],
  "category": "productivity",

  "commands": ["./commands/"],
  "agents": ["./agents/reviewer.md"],
  "hooks": "./hooks/hooks.json",
  "mcpServers": {
    "my-server": {
      "command": "${CLAUDE_PLUGIN_ROOT}/server.py"
    }
  },

  "strict": true
}
```

### Plugin Installation Scopes

| Scope | Location | Applies To | Shared? |
|-------|----------|-----------|---------|
| User | `~/.claude/plugins/` | All your projects | No (per-machine) |
| Project | `.claude/plugins/` | This project | Yes (in git) |
| Managed | System directories | All users (IT-deployed) | Yes (org-wide) |

### Plugin Testing and Development

```bash
# Local testing during development
claude --plugin-dir ./my-plugin

# Reload without restart
/reload-plugins

# Validate plugin structure
claude plugin validate ./my-plugin
```

### Path Variable

Use `${CLAUDE_PLUGIN_ROOT}` for paths relative to plugin root — important since installed plugins are cached:

```json
{
  "mcpServers": {
    "my-server": {
      "command": "${CLAUDE_PLUGIN_ROOT}/bin/server"
    }
  }
}
```

---

## 5. Plugin Marketplaces

### What Is a Marketplace?

A marketplace is a catalog (`marketplace.json`) listing plugins and their sources. Two-step process:
1. Add marketplace: `/plugin marketplace add <source>` — discovers the catalog
2. Install plugins: Choose individual plugins from the catalog

### Official Marketplace

Anthropic's official marketplace (`claude-plugins-official`) is pre-added:

```bash
/plugin install plugin-name@claude-plugins-official
```

Categories: code intelligence, external integrations (GitHub, Jira, Figma, Slack), development workflows, output styles.

**Submit to official marketplace:**
- Claude.ai: `claude.ai/settings/plugins/submit`
- Console: `platform.claude.com/plugins/submit`

### Marketplace File Schema

```json
{
  "name": "my-marketplace",
  "owner": {
    "name": "Team Name",
    "email": "team@example.com"
  },
  "metadata": {
    "description": "Marketplace for company tools",
    "version": "1.0.0",
    "pluginRoot": "./plugins"
  },
  "plugins": [
    {
      "name": "code-review",
      "source": "./plugins/review",
      "description": "AI-powered code reviews",
      "version": "2.1.0",
      "author": {"name": "Team"},
      "category": "productivity",
      "tags": ["review", "quality"],
      "license": "MIT",
      "homepage": "https://docs.example.com/review",
      "repository": "https://github.com/team/review-plugin"
    }
  ]
}
```

### Plugin Source Types

```json
"source": "./plugins/my-plugin"                          // Local relative path
"source": {
  "source": "github",
  "repo": "owner/plugin-repo",
  "ref": "v2.0.0",           // Optional: branch/tag
  "sha": "a1b2c3d4e5..."     // Optional: exact commit (recommended for security)
}
"source": {
  "source": "url",
  "url": "https://github.com/user/repo.git",
  "ref": "main",
  "sha": "a1b2c3d4..."
}
"source": {
  "source": "git-subdir",
  "url": "https://github.com/monorepo/repo.git",
  "path": "tools/plugin",
  "ref": "v1.0"
}
"source": {
  "source": "npm",
  "package": "@org/plugin",
  "version": "2.1.0",
  "registry": "https://npm.example.com"    // Optional: private registry
}
```

### Adding and Managing Marketplaces

```bash
/plugin marketplace add owner/repo-name              # GitHub
/plugin marketplace add owner/repo-name#v2.0.0       # Specific version
/plugin marketplace add https://gitlab.com/.../repo.git
/plugin marketplace add ./my-marketplace              # Local path

/plugin marketplace list                              # List all
/plugin marketplace update marketplace-name           # Refresh catalog
/plugin marketplace remove marketplace-name           # Remove + uninstall its plugins
/plugin marketplace <name> --enable-auto-update
```

### Team/Enterprise Deployment

Add to `.claude/settings.json` to pre-configure for everyone:

```json
{
  "extraKnownMarketplaces": {
    "company-tools": {
      "source": {
        "source": "github",
        "repo": "company/plugins"
      }
    }
  },
  "enabledPlugins": {
    "code-formatter@company-tools": true,
    "deployment@company-tools": true
  }
}
```

### Restricting Marketplaces (Managed)

```json
{
  "strictKnownMarketplaces": [
    {"source": "github", "repo": "approved/plugins"},
    {"source": "url", "url": "https://internal.com/marketplace.json"},
    {"source": "hostPattern", "hostPattern": "^github\\.company\\.com$"}
  ]
}
```

---

## 6. Agents and Subagents

### What Are Agents/Subagents?

Agents are specialized Claude instances spawned by the Agent tool. They run with:
- Their own system prompt defining purpose and expertise
- Restricted or expanded tool access
- Isolated context window (no parent conversation history)
- Their final message returned as the Agent tool result to parent

The **Agent tool** (formerly Task tool, still aliased) is the mechanism for spawning. Claude automatically decides when to delegate, or you can explicitly request.

### Built-in Agent Types

| Type | Model | Tools | Purpose |
|------|-------|-------|---------|
| `Explore` | Haiku (fast) | Read-only | File discovery, codebase search |
| `Plan` | Inherited | Read-only | Context gathering for plan mode |
| `general-purpose` | Inherited | All | Complex multi-step tasks |
| `statusline-setup` | Sonnet | Read, Edit | Configuration tasks |
| `claude-code-guide` | Haiku | Web, Read | Answers Claude Code questions |

### Custom Agent Definition File

**Location**: `.claude/agents/AGENT_NAME.md` (project) or `~/.claude/agents/AGENT_NAME.md` (user)

```yaml
---
name: security-reviewer              # Required. Lowercase, hyphens
description: "Expert security reviewer. Use when reviewing code for vulnerabilities,
              auth issues, injection risks, or security best practices."  # Required
tools: Read, Grep, Glob              # Optional. Comma-separated allowlist
disallowedTools: Write, Edit         # Optional. Denylist
model: sonnet                        # Optional: sonnet, opus, haiku, inherit
permissionMode: acceptEdits          # Optional: default, acceptEdits, dontAsk, bypassPermissions, plan
maxTurns: 10                         # Optional
skills:                              # Optional: preloaded skills
  - api-conventions
mcpServers:                          # Optional
  github: github                     # Reference existing config
memory: user                         # Optional: user, project, or local
background: true                     # Optional: always run in background
isolation: worktree                  # Optional: git worktree isolation
hooks:                               # Optional: lifecycle hooks
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./validate.sh"
---

# System Prompt

You are a senior security engineer. When invoked:
1. Analyze the code for OWASP Top 10 vulnerabilities
2. Check authentication and authorization flows
3. Identify injection risks (SQL, command, XSS)
4. Return a prioritized list of findings
```

### Agent Priority (Resolution Order)

| Location | Scope | Priority |
|----------|-------|----------|
| CLI `--agents` flag | Session only | 1 (highest) |
| `.claude/agents/` | Project | 2 |
| `~/.claude/agents/` | User (all projects) | 3 |
| Plugin's `agents/` | Plugin scope | 4 (lowest) |

### Foreground vs Background Agents

**Foreground** (default):
- Blocks main conversation until complete
- Permission prompts pass through to user
- Use when results needed immediately

**Background**:
- Runs concurrently
- Permission pre-approval required upfront
- Clarifying questions fail silently

```yaml
---
name: long-runner
background: true   # Always run in background
---
```

Runtime: ask Claude "run this in the background" or use Ctrl+B.

### Worktree Isolation

```yaml
---
name: parallel-worker
isolation: worktree    # Each invocation gets isolated git worktree
---
```

Creates `<repo>/.claude/worktrees/<name>/` with its own branch. Auto-cleaned if no changes.

### Tool Restrictions

```yaml
# Read-only (analysis only)
tools: Read, Grep, Glob

# Can modify code
tools: Read, Edit, Grep, Glob

# Can run commands
tools: Read, Bash, Grep

# Can spawn subagents
tools: Agent, Read, Bash

# Can only spawn specific subagents
tools: Agent(worker, researcher), Read, Bash

# Block write operations
disallowedTools: Write, Edit
```

### Agent Communication

- Parent → subagent: Only via spawn prompt string (no conversation history)
- Subagent → parent: Final message returned verbatim as tool result
- No direct peer-to-peer between subagents (use Agent Teams for that)

### Session Resumption (SDK)

```python
# Capture session ID
session_id = None
async for message in query(prompt="Read the auth module", ...):
    if hasattr(message, "session_id"):
        session_id = message.session_id

# Resume later with full context
async for message in query(
    prompt="Now find all places that call it",
    options=ClaudeAgentOptions(resume=session_id)
):
    pass
```

---

## 7. Agent Teams

### What Are Agent Teams?

Agent teams are multiple independent Claude Code sessions (teammates) coordinating on complex work via:
- **Shared task list**: `~/.claude/tasks/{team-name}/` — tasks with pending/in-progress/completed states
- **Mailbox messaging**: Peer-to-peer messages delivered automatically
- **Team lead**: Main session that spawns and coordinates teammates

> **Status**: Experimental, disabled by default

### Enabling Teams

```json
// ~/.claude/settings.json or .claude/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Or: `claude --team`

### Display Modes

| Mode | Behavior | Requirements |
|------|----------|--------------|
| `in-process` | All teammates in main terminal, Shift+Down to cycle | Works anywhere |
| `tmux` | Each teammate in separate pane | tmux or iTerm2 |
| `auto` | Split panes if in tmux, otherwise in-process | — |

```json
{ "teammateMode": "in-process" }
```

### Teammate Communication

```text
# Lead → teammate: Shift+Down to cycle and type
# Teammate → teammate:
message teammate-name: here's what I found about the auth system

# Broadcast to all:
broadcast: all teammates should review the architecture decision
```

### Team Hooks

```json
{
  "hooks": {
    "TeammateIdle": [{
      "hooks": [{"type": "command", "command": "./scripts/validate-work.sh"}]
    }],
    "TaskCompleted": [{
      "hooks": [{"type": "command", "command": "./scripts/check-quality.sh"}]
    }]
  }
}
```

Exit code 2 prevents completion and sends feedback to keep teammate working.

### When to Use Teams vs Subagents

| Aspect | Subagents | Agent Teams |
|--------|-----------|-------------|
| Architecture | Single session, isolated context | Separate sessions, independent |
| Communication | Report results back only | Peer-to-peer + shared task list |
| Token cost | Lower (results summarized) | Higher (each teammate separate) |
| Best for | Focused, self-contained tasks | Complex collaborative work |
| Setup | Easy (file-based) | Requires experimental flag |
| Nesting | Can't spawn subagents | Can't have sub-teams |

### Known Limitations

- One team per session; no nested teams; lead is fixed
- `/resume` and `/rewind` don't restore in-process teammates
- Task status can lag; shutdown can be slow
- Split panes need tmux/iTerm2 (not VS Code integrated terminal)
- Permissions set at spawn (can't set per-teammate at spawn time)

---

## 8. Hooks System

### What Are Hooks?

Hooks are shell commands (or prompts/HTTP calls) that execute at specific lifecycle events. They provide deterministic, guaranteed behavior that doesn't rely on Claude's judgment.

### Hook Events

| Event | When It Fires | Can Block? |
|-------|--------------|-----------|
| `SessionStart` | Session begins/resumes | No |
| `UserPromptSubmit` | Before processing prompt | Yes (exit 2) |
| `PreToolUse` | Before tool execution | Yes (exit 2) |
| `PermissionRequest` | Permission dialog appears | Yes |
| `PostToolUse` | After tool succeeds | No |
| `PostToolUseFailure` | After tool fails | No |
| `Notification` | Claude needs attention | No |
| `SubagentStart` | Subagent spawned | No |
| `SubagentStop` | Subagent finishes | No |
| `Stop` | Claude finishes response | Yes (exit 2 keeps working) |
| `TeammateIdle` | Team teammate idle | No |
| `TaskCompleted` | Task marked complete | Yes (exit 2) |
| `PreCompact` | Before context compaction | No |
| `SessionEnd` | Session terminates | No |

### Hook Configuration

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "async": false,
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write $FILE",
            "timeout": 30
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/validate.sh"
          }
        ]
      }
    ]
  }
}
```

### Hook Exit Codes

| Exit Code | Behavior |
|-----------|----------|
| 0 | Proceed. Stdout added to context (UserPromptSubmit/SessionStart only) |
| 2 | **Block action**. Stderr sent as feedback to Claude |
| Other | Proceed. Stderr logged (visible in verbose mode) |

### Hook Input

Hooks receive JSON on stdin:

```json
{
  "session_id": "abc123",
  "cwd": "/Users/dan/project",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test"
  }
}
```

### Hook Types

**1. Command** (shell execution):
```json
{"type": "command", "command": "./scripts/validate.sh", "timeout": 30}
```

**2. Prompt** (single LLM call for decisions):
```json
{"type": "prompt", "prompt": "Is the code production-ready? Respond with {\"ok\": true/false, \"reason\": \"...\"}"}
```

**3. Agent** (multi-turn with tool access):
```json
{"type": "agent", "prompt": "Verify tests pass before allowing stop. Run test suite.", "timeout": 120}
```

**4. HTTP** (POST to external service):
```json
{"type": "http", "url": "http://localhost:8080/hooks/validate", "headers": {"Authorization": "Bearer $API_KEY"}}
```

### Practical Hook Examples

**Block dangerous commands:**
```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')
if echo "$COMMAND" | grep -qE 'rm -rf /|DROP TABLE'; then
  echo "Blocked: dangerous command" >&2
  exit 2
fi
exit 0
```

**Auto-format on edit:**
```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{"type": "command", "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write"}]
    }]
  }
}
```

**Desktop notification when Claude needs input:**
```json
{
  "hooks": {
    "Notification": [{
      "matcher": "",
      "hooks": [{"type": "command", "command": "osascript -e 'display notification \"Claude needs input\"'"}]
    }]
  }
}
```

### Hook Configuration Locations

| Location | Scope | Shareable |
|----------|-------|-----------|
| `~/.claude/settings.json` | All your projects | No |
| `.claude/settings.json` | Single project | Yes (git) |
| `.claude/settings.local.json` | Single project, you only | No (gitignored) |
| Plugin `hooks/hooks.json` | When plugin enabled | Yes |
| Skill/Agent frontmatter | While skill/agent active | Yes |
| Managed settings | Organization-wide | Yes (IT) |

---

## 9. MCP (Model Context Protocol)

### What Is MCP?

MCP is an open standard for connecting Claude to external tools, databases, APIs, and services. MCP servers expose:
- **Tools**: Functions Claude can call directly
- **Resources**: Data Claude can reference via `@` mentions
- **Prompts**: Instructions that become slash commands (`/mcp__server__prompt`)

### Installing MCP Servers

```bash
# Remote HTTP (recommended for cloud services)
claude mcp add --transport http github https://api.githubcopilot.com/mcp/
claude mcp add --transport http stripe https://mcp.stripe.com
claude mcp add --transport http --header "Authorization: Bearer $TOKEN" myapi https://api.example.com/mcp

# Local stdio (for local processes)
claude mcp add --transport stdio airtable \
  --env AIRTABLE_API_KEY=YOUR_KEY \
  -- npx -y airtable-mcp-server

# Scoped to all projects (user scope)
claude mcp add --scope user github https://api.githubcopilot.com/mcp/

# Shared with team (project scope, goes to .mcp.json)
claude mcp add --scope project github https://api.githubcopilot.com/mcp/
```

### `.mcp.json` Configuration

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/",
      "headers": {
        "Authorization": "Bearer ${GITHUB_TOKEN}"
      }
    },
    "database": {
      "type": "stdio",
      "command": "/usr/local/bin/db-server",
      "args": ["--config", "/etc/db.json"],
      "env": {
        "DB_PASSWORD": "${DB_PASSWORD}"
      }
    }
  }
}
```

Environment variable syntax: `${VAR}` or `${VAR:-default}`

### MCP Tool Search

When MCP tools exceed context window threshold, Tool Search auto-activates:

```bash
ENABLE_TOOL_SEARCH=false claude         # Disable
ENABLE_TOOL_SEARCH=auto:5 claude        # Activate at 5% threshold
ENABLE_TOOL_SEARCH=true claude          # Force always on
```

### Popular MCP Servers

| Server | URL/Package | Transport |
|--------|------------|-----------|
| GitHub | `https://api.githubcopilot.com/mcp/` | HTTP |
| Stripe | `https://mcp.stripe.com` | HTTP |
| Sentry | `https://mcp.sentry.dev/mcp` | HTTP |
| Notion | `https://mcp.notion.com/mcp` | HTTP |
| Asana | `https://mcp.asana.com/sse` | SSE |
| Slack | Slack MCP package | HTTP |

### MCP from Plugins

Plugins can bundle MCP servers:

```json
{
  "name": "db-plugin",
  "mcpServers": {
    "postgres": {
      "command": "${CLAUDE_PLUGIN_ROOT}/servers/postgres",
      "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config.json"]
    }
  }
}
```

---

## 10. CLAUDE.md Files

### What Are CLAUDE.md Files?

Persistent Markdown instruction files loaded at session start. Unlike settings (which enforce behavior), CLAUDE.md provides guidance Claude reads and follows.

**Key insight**: Settings enforce rules; CLAUDE.md provides context and conventions.

### Locations and Precedence

| Scope | Location | Precedence |
|-------|----------|-----------|
| Managed policy | System directories | Highest (can't exclude) |
| Project root | `./CLAUDE.md` or `./.claude/CLAUDE.md` | High |
| User home | `~/.claude/CLAUDE.md` | Medium |
| Local | `./CLAUDE.local.md` | Low |
| Subdirectory | `packages/frontend/CLAUDE.md` | Lazy-loaded |

### Effective CLAUDE.md Structure

Target under 200 lines. Be specific:

```markdown
# Project Overview

Brief description of the project.

# Build and Test

- Build: `npm run build`
- Test: `npm test`
- Dev server: `npm run dev`

# Code Standards

- TypeScript, strict mode
- 2-space indentation
- Use camelCase for variables, PascalCase for classes
- Components in `src/components/`, utilities in `src/utils/`

# Git Workflow

- Feature branches from `main`
- Include issue number: `git checkout -b fix/issue-123`
- Squash commits before merging
```

### Importing Additional Files

```markdown
# Project Overview

See @README for project description and @package.json for commands.

# Coding Standards

See @STANDARDS.md for detailed guidelines and @.eslintrc for linter rules.
```

- `@README` — relative to CLAUDE.md location
- `@docs/api.md` — nested path
- `@~/.claude/shared-rules.md` — home directory reference
- Max 5 import depth levels

### Path-Specific Rules

```markdown
---
paths:
  - "src/api/**/*.ts"
  - "src/**/*.{ts,tsx}"
---

# API Development Rules

- All endpoints must include validation
- Use standard error response format
- Include OpenAPI documentation comments
```

Path-scoped rules load on-demand when matching files are opened. Rules without `paths` apply everywhere.

### Organizing with `.claude/rules/`

```
.claude/
├── CLAUDE.md                    # Main instructions, imports rules
└── rules/
    ├── code-style.md
    ├── testing.md
    ├── api-design.md
    ├── security.md
    └── frontend/
        ├── components.md
        └── styling.md
```

### Auto Memory vs CLAUDE.md

| Aspect | CLAUDE.md | Auto Memory |
|--------|-----------|------------|
| Who writes | You | Claude |
| Content | Rules, conventions, standards | Learnings, patterns, discoveries |
| Updated | Manually | Automatically by Claude |
| Loaded | Full at startup | First 200 lines at startup |

Use `/memory` to view and edit both.

---

## 11. Settings and Configuration

### Configuration Hierarchy

```
Managed (highest — admin-deployed, can't override)
  → Local (.claude/settings.local.json)
    → Project (.claude/settings.json)
      → User (~/.claude/settings.json — lowest, fallback)
```

Array settings (permissions, hooks, env vars) **merge** across scopes rather than replace.

### Configuration Files

| File | Location | Purpose |
|------|----------|---------|
| `settings.json` | `~/.claude/` or `.claude/` | Permissions, env, hooks, model |
| `settings.local.json` | `.claude/` | Personal overrides (gitignored) |
| `.claude.json` | `~/` (home) | Session state, OAuth, MCP servers |
| `.mcp.json` | Project root | MCP server definitions (shared) |
| `CLAUDE.md` | Various | Instructions and context |
| `managed-settings.json` | System dirs | IT-deployed policy |

### Core Settings Example

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",

  "model": "sonnet",
  "language": "english",

  "permissions": {
    "allow": ["Bash(npm *)", "Read(src/**)", "Edit(src/**)"],
    "ask": ["Bash(git push *)"],
    "deny": ["Read(.env*)", "WebFetch"],
    "defaultMode": "acceptEdits"
  },

  "env": {
    "NODE_ENV": "development",
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  },

  "enabledPlugins": {
    "my-plugin@marketplace": true
  },

  "teammateMode": "auto"
}
```

### Permission Syntax

```json
"allow": [
  "Bash",                      // All bash commands
  "Bash(npm run *)",           // Bash matching pattern
  "Read(src/**)",              // Read with glob
  "Agent(worker, researcher)", // Only these subagent types
  "mcp__github__*"             // All GitHub MCP tools
]
```

### Key Environment Variables

```bash
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1  # Enable agent teams
CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1  # Disable background tasks
MAX_THINKING_TOKENS=10000               # Thinking budget
ENABLE_TOOL_SEARCH=auto:5               # MCP tool search threshold
```

### Managed-Only Settings

These can **only** be set in `managed-settings.json` (IT-deployed):

- `allowManagedHooksOnly` — Block user/project hooks
- `allowManagedPermissionRulesOnly` — Lock permissions
- `allowManagedMcpServersOnly` — Restrict MCP servers
- `strictKnownMarketplaces` — Control plugin sources
- `disableBypassPermissionsMode` — Disable permission bypass

---

## 12. Agent SDK

### Overview

The Claude Agent SDK (TypeScript and Python) lets you build standalone applications with the same agentic loop as Claude Code, programmatically.

### Installation

```bash
# TypeScript
npm install @anthropic-ai/claude-agent-sdk

# Python
pip install claude-agent-sdk
```

### Core API

```typescript
// TypeScript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Your task description",
  options: {
    allowedTools: ["Read", "Bash", "Grep", "Agent"],
    agents: {
      "code-reviewer": {
        description: "Expert code reviewer for security and quality",
        prompt: "You are a senior code reviewer...",
        tools: ["Read", "Grep", "Glob"],
        model: "sonnet"
      }
    },
    permissionMode: "acceptEdits",
    mcpServers: {},
    hooks: {}
  }
})) {
  if ("result" in message) console.log(message.result);
}
```

```python
# Python
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition
import asyncio

async def main():
    async for message in query(
        prompt="Your task description",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Bash", "Agent"],
            agents={
                "reviewer": AgentDefinition(
                    description="Code reviewer",
                    prompt="You are a code reviewer...",
                    tools=["Read", "Grep"],
                    model="sonnet"
                )
            },
            permission_mode="acceptEdits"
        )
    ):
        print(message)

asyncio.run(main())
```

### AgentDefinition Fields

| Field | Type | Description |
|-------|------|-------------|
| `description` | string | **Required.** When to use this agent (Claude reads this) |
| `prompt` | string | **Required.** System prompt for the agent |
| `tools` | string[] | Allowed tools (inherits all if omitted) |
| `disallowedTools` | string[] | Tools to remove from inherited set |
| `model` | string | `"sonnet"`, `"opus"`, `"haiku"`, or `"inherit"` |
| `permissionMode` | string | `"default"`, `"acceptEdits"`, `"dontAsk"`, `"bypassPermissions"` |
| `maxTurns` | number | Max agentic turns |
| `mcpServers` | object | MCP server configurations |
| `hooks` | object | Lifecycle hooks |
| `skills` | string[] | Skills to preload |
| `memory` | string | `"user"`, `"project"`, or `"local"` |
| `background` | boolean | Always run in background |
| `isolation` | string | `"worktree"` for git worktree isolation |

### Permission Modes

| Mode | Behavior |
|------|----------|
| `default` | Show permission dialogs |
| `acceptEdits` | Auto-accept file edits, prompt for other |
| `dontAsk` | Run autonomously, skip all prompts |
| `bypassPermissions` | Skip all permission checks (dangerous) |
| `plan` | Require plan approval before executing |

### Detecting Agent Tool Use (SDK)

```python
if hasattr(message, "content") and message.content:
    for block in message.content:
        if getattr(block, "type", None) == "tool_use" and block.name in ("Task", "Agent"):
            print(f"Subagent invoked: {block.input.get('subagent_type')}")

# Detect if running inside subagent context
if hasattr(message, "parent_tool_use_id") and message.parent_tool_use_id:
    print("Running inside a subagent")
```

Note: The Task tool was renamed to Agent in Claude Code v2.1.63. Old code using `Task(...)` still works as an alias.

---

## 13. Packaging and Distribution

### Creating a Distributable Plugin (Checklist)

1. **Structure your plugin** following the directory layout in §4
2. **Create `plugin.json`** with name, version, description, author, license
3. **Use `${CLAUDE_PLUGIN_ROOT}`** for all dynamic paths within the plugin
4. **Test locally**: `claude --plugin-dir ./my-plugin`
5. **Create a marketplace** with `marketplace.json` listing your plugin
6. **Publish to GitHub/GitLab** and share the add command

### Semantic Versioning

```
1.0.0  → Initial release
1.1.0  → New feature (backward compatible)
1.1.1  → Bug fix
2.0.0  → Breaking changes
```

### Publishing Checklist

- [ ] Version bumped in `plugin.json`
- [ ] CHANGELOG updated
- [ ] README with install instructions and examples
- [ ] All skills/agents tested locally
- [ ] No hardcoded paths or secrets
- [ ] `${CLAUDE_PLUGIN_ROOT}` used for dynamic paths
- [ ] All files properly licensed
- [ ] Submitted to marketplace (if desired)

### Plugin Caching

Installed plugins are **copied** to cache at `~/.claude/plugins/cache/`. This means:
- Plugins can reference files within their own directory
- Plugins cannot reference `../sibling-directory/file`
- Use symlinks for shared utilities across plugins

---

## 14. Integration Patterns

### Pattern: Skills + Isolated Subagent

```yaml
---
name: research-topic
description: Deep research without touching current files
context: fork
agent: Explore
allowed-tools: Bash(rg *), Read, Glob, Grep
---

Research $ARGUMENTS thoroughly:
1. Find relevant files with grep/glob
2. Read and analyze contents
3. Synthesize findings
```

### Pattern: MCP + Skills

```yaml
---
name: github-pr-review
description: Review a GitHub PR using MCP tools
allowed-tools: mcp__github__*, Read, Edit
---

Review PR $ARGUMENTS:
1. Get diff and comments via GitHub MCP
2. Analyze changes for quality and security
3. Suggest specific improvements
```

### Pattern: Hooks + Auto-Formatting (Plugin-Bundled)

```json
{
  "name": "formatting-plugin",
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{"type": "command", "command": "${CLAUDE_PLUGIN_ROOT}/format.sh"}]
    }]
  }
}
```

### Pattern: Agent Team for Parallel Review

```text
Create an agent team to review PR #142:
- Security reviewer: auth, injection, data exposure
- Performance reviewer: query efficiency, caching, N+1
- Coverage reviewer: test completeness, edge cases
```

Each reviewer works in parallel; lead synthesizes findings.

### Pattern: Chaining Subagents

```text
Use the security-reviewer to find vulnerabilities in auth.ts,
then use the optimizer agent to propose fixes for the issues found.
```

Results from first subagent inform the second.

### Pattern: Hook as Quality Gate

```bash
#!/bin/bash
# .claude/hooks/quality-gate.sh
# Blocks Claude from stopping until tests pass

INPUT=$(cat)
EVENT=$(echo "$INPUT" | jq -r '.hook_event_name')

if [ "$EVENT" = "Stop" ]; then
  npm test --silent 2>/dev/null
  if [ $? -ne 0 ]; then
    echo "Tests are failing. Fix them before stopping." >&2
    exit 2  # Keep Claude working
  fi
fi
exit 0
```

```json
{
  "hooks": {
    "Stop": [{"matcher": "", "hooks": [{"type": "command", "command": ".claude/hooks/quality-gate.sh"}]}]
  }
}
```

### Pattern: CLAUDE.md + Skills for Project Onboarding

**CLAUDE.md**: General project context, architecture, build commands

**`.claude/skills/onboard/SKILL.md`**: Step-by-step onboarding workflow that reads the codebase and produces a personalized overview for new developers

### Pattern: Enterprise Deployment

```
managed-settings.json (IT-deployed):
├── strictKnownMarketplaces: [approved sources]
├── allowManagedMcpServersOnly: true
└── MCP servers: [company-approved tools]

.claude/settings.json (team-shared in git):
├── permissions: [project-specific allowlist]
├── hooks: [formatting, linting]
└── extraKnownMarketplaces: {company-tools: ...}

~/.claude/settings.json (per-developer):
└── Personal preferences within allowed scope
```

---

## Quick Reference

### Create a Skill

```bash
mkdir -p .claude/skills/my-skill
cat > .claude/skills/my-skill/SKILL.md << 'EOF'
---
name: my-skill
description: What this skill does and when to invoke it
tools: Read, Grep, Glob
---

Instructions for $ARGUMENTS...
EOF
```

### Create an Agent

```bash
cat > .claude/agents/my-agent.md << 'EOF'
---
name: my-agent
description: Expert agent. Use when you need specialized analysis of X.
tools: Read, Grep, Glob
model: sonnet
---

You are an expert in X. When invoked:
1. Analyze the provided code
2. Return prioritized findings
EOF
```

### Create a Plugin

```bash
mkdir -p my-plugin/.claude-plugin my-plugin/skills/my-skill
echo '{"name":"my-plugin","version":"1.0.0","description":"...","license":"MIT"}' > my-plugin/.claude-plugin/plugin.json
# Add skills, agents, hooks, .mcp.json as needed
claude --plugin-dir ./my-plugin   # Test
```

### Create a Marketplace

```json
{
  "name": "my-marketplace",
  "plugins": [{
    "name": "my-plugin",
    "source": {"source": "github", "repo": "owner/my-plugin"},
    "version": "1.0.0"
  }]
}
```

```bash
/plugin marketplace add owner/my-marketplace-repo
/plugin install my-plugin@my-marketplace
```

---

## Sources

- Claude Code Documentation: `https://code.claude.com/docs/`
- Skills: `https://code.claude.com/docs/en/skills.md`
- Plugins: `https://code.claude.com/docs/en/plugins.md`
- Marketplaces: `https://code.claude.com/docs/en/plugin-marketplaces.md`
- Hooks: `https://code.claude.com/docs/en/hooks-guide.md`
- Subagents: `https://code.claude.com/docs/en/sub-agents.md`
- Agent Teams: `https://code.claude.com/docs/en/agent-teams.md`
- MCP: `https://code.claude.com/docs/en/mcp.md`
- Settings: `https://code.claude.com/docs/en/settings.md`
- Agent SDK: `https://platform.claude.com/docs/en/agent-sdk/overview.md`
