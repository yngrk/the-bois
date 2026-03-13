# AI Dev Team

A multi-agent development setup for Claude Code. You talk to a **Manager** agent in VSCode; the manager breaks down your request and spawns specialized **subagents** that work on the project — in parallel when possible.

---

## How It Works

### The Manager

When you open this project in VSCode with the Claude Code extension, `CLAUDE.md` is automatically loaded. This turns Claude into the **Engineering Manager** — your single point of contact. You give it high-level instructions; it handles the rest.

The manager's job is to:
1. Understand your request
2. Break it into discrete tasks
3. Spawn the right subagents (in parallel when tasks are independent)
4. Collect and integrate their outputs
5. Report results back to you

You never talk directly to subagents. Everything goes through the manager.

### Subagents

Subagents are defined in `.claude/agents/`. Each is a Claude Code instance with a specific role, toolset, and system prompt. The manager spawns them on demand using Claude Code's built-in `Agent` tool.

| Agent | Role | Tools |
|-------|------|-------|
| `coder` | Implements features, fixes bugs, refactors code | Read, Edit, Write, Bash, Glob, Grep |
| `reviewer` | Reviews code quality and correctness (read-only) | Read, Glob, Grep |
| `tester` | Writes and runs tests, diagnoses failures | Read, Edit, Write, Bash, Glob, Grep |
| `researcher` | Investigates codebases, APIs, and documentation | Read, Glob, Grep, WebSearch, WebFetch |
| `architect` | Designs system structure and creates implementation plans | Read, Glob, Grep |

### Parallel Work & Worktrees

When tasks are independent, the manager spawns multiple agents **simultaneously** (multiple `Agent` tool calls in a single response). Each runs concurrently and returns results independently.

When agents need to write files at the same time, the manager uses `isolation: "worktree"` — each agent gets its own isolated git branch (a git worktree) so they don't conflict. Worktrees with changes are preserved for review; empty ones are cleaned up automatically.

This is why the project is a git repository — worktree isolation requires it.

---

## Project Structure

```
ai-dev-team/
├── CLAUDE.md                  # Manager persona — loaded automatically by Claude Code
└── .claude/
    └── agents/
        ├── coder.md           # Coder subagent definition
        ├── reviewer.md        # Reviewer subagent definition
        ├── tester.md          # Tester subagent definition
        ├── researcher.md      # Researcher subagent definition
        └── architect.md       # Architect subagent definition (uses Opus model)
```

Each file in `.claude/agents/` uses YAML frontmatter to configure the agent:

```markdown
---
name: coder
description: When the manager should spawn this agent (Claude reads this to decide)
tools: Read, Edit, Write, Bash, Glob, Grep
model: sonnet
---

System prompt for the agent goes here.
```

---

## Usage

### Prerequisites

- [Claude Code](https://marketplace.visualstudio.com/items?itemName=anthropic.claude-code) VSCode extension installed
- An Anthropic API key configured

### Opening the Project

Open this folder in VSCode. Claude Code will load `CLAUDE.md` and activate the manager persona automatically. Start a new conversation.

### Giving Orders

Talk to the manager naturally. Examples:

**Simple task — single agent:**
> "Fix the null pointer bug in `src/auth.ts`."

The manager spawns a `coder` agent with the relevant context.

**Multi-step task — sequential agents:**
> "Add input validation to the login form, then review it."

The manager spawns a `coder`, waits for it to finish, then spawns a `reviewer`.

**Complex task — parallel agents:**
> "Build a REST API for user authentication."

The manager might spawn an `architect` to design it, then simultaneously spawn a `coder` (to implement) and a `tester` (to write tests), each in their own worktree.

**Research first:**
> "What's the best way to add rate limiting to our Express app? Then implement it."

The manager spawns a `researcher` first, gets the findings, then spawns a `coder` with those findings as context.

### Customizing Agents

To change what a subagent does, edit its file in `.claude/agents/`. Key fields:

- **`description`** — Claude reads this to decide when to spawn the agent. Be specific.
- **`tools`** — Comma-separated list of tools the agent is allowed to use.
- **`model`** — `sonnet` (default), `opus` (smarter, slower), or `haiku` (faster, lighter).
- **`maxTurns`** — Maximum number of steps the agent can take (default: unlimited).

### Adding New Agent Types

Create a new file in `.claude/agents/`:

```markdown
---
name: devops
description: Manages CI/CD pipelines, Docker configs, and deployment scripts.
tools: Read, Edit, Write, Bash, Glob, Grep
model: sonnet
---

You are a DevOps Agent on an AI dev team...
```

The manager will automatically know it exists and can spawn it when relevant.

---

## Pixel Agents (Optional Visual Layer)

[Pixel Agents](https://marketplace.visualstudio.com/items?itemName=pablodelucca.pixel-agents) is a VSCode extension that visualizes each Claude Code agent as an animated pixel art character in a virtual office. Characters animate based on what tools they're using — typing when editing files, reading when searching, idle when waiting.

It works by watching Claude Code's JSONL transcript files passively, with no changes needed to this setup. Each spawned subagent appears as its own character automatically.
