# Routing logic for `improve` skill

## Capability buckets and priority

1. **Personal skills** — `~/.claude/skills/<name>/SKILL.md`. Highest priority for exact domain match. Personal wins over project when names collide.
2. **Project skills** — `.claude/skills/<name>/SKILL.md` in cwd or ancestors. Use when the task is project-scoped.
3. **Plugin skills** — invoke as `plugin-name:skill-name`. NEVER assume plugin filesystem layout; rely on Claude Code's native discovery. The `~/.claude/plugins/` snapshot is a hint, not a registry.
4. **MCP prompts** — recognizable by `mcp__<server>__<prompt>` prefix. Do not crawl MCP from directories; they surface through `/help` and system init messages.
5. **Agents** — `~/.claude/agents/` (personal) or `.claude/agents/` (project). Prefer agents when the task benefits from isolated context or forked subagent execution.
6. **Native tools** — Read, Write, Edit, Bash, Grep, Glob, WebFetch, WebSearch, Task. Fallback when no capability matches.

## Matching heuristics

- Match by SKILL.md `description` field, not just `name`. A skill named `commit` might describe "stage and commit changes to git" — match based on semantic overlap with the raw input.
- For domain ambiguity (e.g., raw prompt mentions "email" and both a personal skill and an MCP Gmail connector exist), prefer the more specific/recent capability. If unclear, ask ONE clarifying question in verbose/debug mode, or pick personal skill in silent mode.
- If the raw input mentions a capability by name explicitly (e.g., "use youtube_ultimate to..."), route there without further evaluation, subject to safety checks.

## Output announcement format

Always exactly one line, with an arrow prefix:

```
→ Routing via: youtube_ultimate
→ Routing via: everything-claude-code:code-review  
→ Routing via: mcp__notion__create_page
→ Routing via: agent:research-agent
→ Routing via: native tools
```

Never add explanation, framework names, or routing rationale in silent mode.

## When to ask a clarifying question

Ask exactly ONE question, only if:
- Raw input is genuinely ambiguous AND
- Conversation history does not resolve it AND  
- Multiple capabilities match with no clear winner

Do not ask clarifying questions for:
- Prompts with sufficient context
- Prompts that match a single capability unambiguously
- Prompts that clearly fall through to native tools
