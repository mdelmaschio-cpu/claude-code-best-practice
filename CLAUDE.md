# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a best practices repository for Claude Code configuration, demonstrating patterns for skills, subagents, hooks, and commands. It serves as a reference implementation rather than an application codebase.

## Key Components

### Weather System (Example Workflow)
A demonstration of two distinct skill patterns via the **Command → Agent → Skill** architecture:
- `/weather-orchestrator` command (`.claude/commands/weather-orchestrator.md`): Entry point — asks user for C/F, invokes agent, then invokes SVG skill
- `weather-agent` agent (`.claude/agents/weather-agent.md`): Fetches temperature using its preloaded `weather-fetcher` skill (agent skill pattern)
- `weather-fetcher` skill (`.claude/skills/weather-fetcher/SKILL.md`): Preloaded into agent — instructions for fetching temperature from Open-Meteo
- `weather-svg-creator` skill (`.claude/skills/weather-svg-creator/SKILL.md`): Skill — creates SVG weather card, writes `orchestration-workflow/weather.svg` and `orchestration-workflow/output.md`

Two skill patterns: agent skills (preloaded via `skills:` field) vs skills (invoked via `Skill` tool). See `orchestration-workflow/orchestration-workflow.md` for the complete flow diagram.

### Skill Definition Structure
Skills in `.claude/skills/<name>/SKILL.md` use YAML frontmatter:
- `name`: Display name and `/slash-command` (defaults to directory name)
- `description`: When to invoke (recommended for auto-discovery)
- `argument-hint`: Autocomplete hint (e.g., `[issue-number]`)
- `disable-model-invocation`: Set `true` to prevent automatic invocation
- `user-invocable`: Set `false` to hide from `/` menu (background knowledge only)
- `allowed-tools`: Tools allowed without permission prompts when skill is active
- `model`: Model to use when skill is active
- `context`: Set to `fork` to run in isolated subagent context
- `agent`: Subagent type for `context: fork` (default: `general-purpose`)
- `hooks`: Lifecycle hooks scoped to this skill

### Repository Agents
- `presentation-curator`: Manages `presentation/index.html` with 3 preloaded skills — always delegate via Agent tool (see `.claude/rules/presentation.md`)
- `weather-agent`: Fetches Dubai weather using preloaded `weather-fetcher` skill; `model: sonnet`, `memory: project`, scoped `PreToolUse` hook
- `time-agent`: Displays current time in PKT (UTC+5)
- `development-workflows-research-agent`: Reads GitHub repos, counts agents/skills/commands, reports on workflow repositories; `permissionMode: bypassPermissions`, `maxTurns: 30`

### Repository Skills
- `weather-fetcher`: Agent-preloaded skill for Open-Meteo temperature fetch (not user-invocable directly)
- `weather-svg-creator`: Creates SVG weather card and writes `orchestration-workflow/weather.svg`
- `agent-browser`: Browser automation via `agent-browser` CLI — navigate, snapshot, fill forms, click; uses `allowed-tools: Bash(agent-browser:*)`
- `time-skill` / `time-command`: Display PKT time

### Workflow Commands & Agents
Nested under `.claude/commands/workflows/best-practice/` — one command per report type:
- `workflow-claude-skills`, `workflow-claude-commands`, `workflow-claude-subagents`, `workflow-claude-settings`: Each launches a matching research agent from `.claude/agents/workflows/best-practice/` to detect documentation drift
- `.claude/commands/workflows/development-workflows.md`: Updates the DEVELOPMENT WORKFLOWS table in README.md using `development-workflows-research-agent`

### Presentation System
See `.claude/rules/presentation.md` — all presentation work is delegated to the `presentation-curator` agent.

### Hooks System
Cross-platform sound notification system in `.claude/hooks/`:
- `scripts/hooks.py`: Main handler for Claude Code hook events
- `config/hooks-config.json`: Shared team configuration
- `config/hooks-config.local.json`: Personal overrides (git-ignored)
- `sounds/`: Audio files organized by hook event (generated via ElevenLabs TTS)

Hook events configured in `.claude/settings.json`: PreToolUse, PostToolUse, PostToolUseFailure, UserPromptSubmit, Notification, Stop, SubagentStart, SubagentStop, PreCompact, PostCompact, SessionStart, SessionEnd, Setup, PermissionRequest, TeammateIdle, TaskCompleted, ConfigChange, WorktreeCreate, WorktreeRemove, InstructionsLoaded, Elicitation, ElicitationResult.

Special handling: git commits trigger `pretooluse-git-committing` sound.

## Critical Patterns

### Subagent Orchestration
Subagents **cannot** invoke other subagents via bash commands. Use the Agent tool (renamed from Task in v2.1.63; `Task(...)` still works as an alias):
```
Agent(subagent_type="agent-name", description="...", prompt="...", model="haiku")
```

Be explicit about tool usage in subagent definitions. Avoid vague terms like "launch" that could be misinterpreted as bash commands.

### Subagent Definition Structure
Subagents in `.claude/agents/*.md` use YAML frontmatter:
- `name`: Subagent identifier
- `description`: When to invoke (use "PROACTIVELY" for auto-invocation)
- `tools`: Comma-separated allowlist of tools (inherits all if omitted). Supports `Agent(agent_type)` syntax
- `disallowedTools`: Tools to deny, removed from inherited or specified list
- `model`: Model alias: `haiku`, `sonnet`, `opus`, or `inherit` (default: `inherit`)
- `permissionMode`: Permission mode (e.g., `"acceptEdits"`, `"plan"`, `"bypassPermissions"`)
- `maxTurns`: Maximum agentic turns before the subagent stops
- `skills`: List of skill names to preload into agent context
- `mcpServers`: MCP servers for this subagent (server names or inline configs)
- `hooks`: Lifecycle hooks scoped to this subagent (all hook events are supported; `PreToolUse`, `PostToolUse`, and `Stop` are the most common)
- `memory`: Persistent memory scope — `user`, `project`, or `local` (see `reports/claude-agent-memory.md`)
- `background`: Set to `true` to always run as a background task
- `effort`: Effort level override: `low`, `medium`, `high`, `max` (default: inherits from session)
- `isolation`: Set to `"worktree"` to run in a temporary git worktree
- `color`: CLI output color for visual distinction

### Configuration Hierarchy
1. **Managed** (`managed-settings.json` / MDM plist / Registry): Organization-enforced, cannot be overridden
2. Command line arguments: Single-session overrides
3. `.claude/settings.local.json`: Personal project settings (git-ignored)
4. `.claude/settings.json`: Team-shared settings
5. `~/.claude/settings.json`: Global personal defaults
6. `hooks-config.local.json` overrides `hooks-config.json`

### Disable Hooks
Set `"disableAllHooks": true` in `.claude/settings.local.json`, or disable individual hooks in `hooks-config.json`.

## Answering Best Practice Questions

When the user asks a Claude Code best practice question, **always search this repo first** (`best-practice/`, `reports/`, `tips/`, `implementation/`, and `README.md`) before relying on training knowledge or external sources. This repo is the authoritative source — only fall back to external docs or web search if the answer is not found here.

## Workflow Best Practices

From experience with this repository:

- Keep CLAUDE.md under 200 lines per file for reliable adherence
- Use commands for workflows instead of standalone agents
- Create feature-specific subagents with skills (progressive disclosure) rather than general-purpose agents
- Perform manual `/compact` at ~50% context usage
- Start with plan mode for complex tasks
- Use human-gated task list workflow for multi-step tasks
- Break subtasks small enough to complete in under 50% context

### Debugging Tips

- Use `/doctor` for diagnostics
- Run long-running terminal commands as background tasks for better log visibility
- Use browser automation MCPs (Claude in Chrome, Playwright, Chrome DevTools) for Claude to inspect console logs
- Provide screenshots when reporting visual issues

### Settings Highlights (`.claude/settings.json`)
- `outputStyle`: Sets AI response style (e.g., `"Explanatory"`)
- `plansDirectory`: Points plan/report outputs to `./reports`
- `env.CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`: Auto-compact threshold (set to `"80"`)
- `spinnerVerbs` / `spinnerTipsOverride`: Custom loading messages
- `respectGitignore: true`: Keeps gitignored files out of context

### Agent Teams
`agent-teams/` contains the prompt and output for multi-agent parallel workflows. See `implementation/claude-agent-teams-implementation.md`.

## Documentation

See `.claude/rules/markdown-docs.md` for documentation standards. Key docs:
- `best-practice/claude-subagents.md`: Subagent frontmatter, hooks, and repository agents
- `best-practice/claude-skills.md`: Skill frontmatter fields and patterns
- `best-practice/claude-commands.md`: Slash command patterns and built-in command reference
- `reports/claude-agent-memory.md`: Agent memory scopes (`user`, `project`, `local`)
- `orchestration-workflow/orchestration-workflow.md`: Weather system flow diagram
