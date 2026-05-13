# Improve

A Claude Code skill that rewrites a raw prompt to suit **Opus 4.7**, routes it to the best available local capability (skill, plugin, MCP prompt, agent, or native tools), and executes it — all driven by a single `/improve <prompt>` invocation.

The rewrite work happens inside a subagent, so your inline session only sees the finished, route-tagged improved prompt — not the routing reasoning, not the discovery output, not the cookbook lookup.

## What it does

When you run `/improve <raw prompt>`:

1. The inline session spawns one subagent.
2. The subagent:
   - Reads three references (routing logic, safety policy, Opus 4.7 prompting cookbook).
   - Discovers your installed skills, plugin skills, MCP prompts, agents, and commands.
   - Picks the best capability for the task (or falls through to native tools).
   - Rewrites the prompt using the Opus 4.7 cookbook (clarity, why-grounded instructions, positive framing, XML structure, scope literalism, capability-aware invocation, no placeholders).
   - Scans both the raw input and the rewrite for destructive patterns (`rm -rf`, `git reset --hard`, `DROP TABLE`, force-push, infra delete, etc.) and side-effectful capabilities (`deploy`, `publish`, `send`).
   - Returns a strict structured envelope: `ROUTE: / SAFETY: / AMBIGUITY: / ---REWRITE-BEGIN--- ... ---REWRITE-END---`.
3. The inline session:
   - Gates on safety (asks for `yes/no` confirmation if anything destructive was matched).
   - Prints one line: `→ Routing via: <name>`.
   - Optionally displays the rewrite (only when `!verbose` or `!debug` is set).
   - Executes the rewrite verbatim.

## Why this shape

- **Opus 4.7 tuning.** Opus 4.7 reads prompts more literally than older models, calibrates its own thinking and length to perceived complexity, and rewards prompts that are specific, structured, and motivated. The cookbook applied during rewrite is distilled from Anthropic's own Opus 4.7 prompting guidance, adapted for Claude Code's execute-not-paste context (no chat-app closing line, no `[paste here]` placeholders, capability-aware invocation).
- **Context-aware routing.** The rewrite knows what's installed and routes accordingly — invoking a specific skill / plugin / MCP / agent by name rather than reimplementing what it does.
- **Subagent isolation.** The discovery commands, capability SKILL.md reads, routing reasoning, and cookbook application all stay inside the subagent's context. Your inline session stays clean and only sees the finished prompt + the executed result.
- **Safety + injection defense.** Destructive shell, git, database, infrastructure patterns and semantic verbs (`nuke`, `wipe`, etc.) require explicit confirmation. The raw input is wrapped as data, not instructions — directives inside it cannot override the wrapper policy.

## Install

```bash
# 1. Copy the skill into your Claude Code skills directory
mkdir -p ~/.claude/skills/improve
cp -r SKILL.md references ~/.claude/skills/improve/

# 2. (Optional) Copy the verbose/debug slash command shims
mkdir -p ~/.claude/commands
cp commands/improve-verbose.md commands/improve-debug.md ~/.claude/commands/
```

That's it. Restart Claude Code (or wait for skill discovery to pick it up) and `/improve` is live.

## Usage

```text
/improve <raw prompt>
```

Default mode: silent. The rewrite stays inside the subagent; you see one routing line and the executed result.

```text
/improve-verbose <raw prompt>
```

Prepends `!verbose` — the rewrite is printed in a fenced block before execution. Use when you want to inspect what the subagent produced.

```text
/improve-debug <raw prompt>
```

Prepends `!debug` — same as verbose, plus the subagent is permitted to use bounded staged reasoning (critique-revise) inside the rewrite when the routed capability benefits from it.

### Example

```text
/improve write me a haiku about coffee
```

Inline session output:
```text
→ Routing via: native tools
<haiku>
```

```text
/improve-verbose review my python code for bugs
```

Inline session output:
```text
→ Routing via: agent:python-reviewer
```
```text
You're going to review Python code I share with you. Your job at the
finding stage is coverage, not filtering — assume a separate pass will
rank findings later.

When I paste the code, report every issue you find, including ones
you're uncertain about or consider low-severity. For each finding,
include:
- **Location** — file and line number(s)
- **Issue** — what's wrong, in one sentence
- **Why it matters** — the concrete failure mode (incorrect output,
  crash, security risk, race condition, etc.)
- **Confidence** — high / medium / low
- **Severity** — high / medium / low
- **Suggested fix** — a minimal change that addresses the issue

...
```
```text
<the python-reviewer agent then runs against your code>
```

## Layout

```
Improve/
├── SKILL.md                          # the skill itself (thin wrapper around the subagent)
├── references/
│   ├── router.md                     # capability priority + matching heuristics
│   ├── safety.md                     # destructive-pattern + side-effect policy
│   └── opus47-prompting.md           # Opus 4.7 prompting cookbook
└── commands/
    ├── improve-verbose.md            # /improve-verbose shim → prepends !verbose
    └── improve-debug.md              # /improve-debug shim → prepends !debug
```

## Configuration knobs

All inside `SKILL.md`:

- `allowed-tools` is intentionally limited to `Agent` and `Read`. The post-skill agent loop has full tool access for executing the rewrite.
- `disable-model-invocation: true` means the skill only fires when you explicitly type `/improve`. It will not auto-trigger from descriptive matches.

## Customizing the cookbook

`references/opus47-prompting.md` is the content playbook the subagent applies. Edit it to:

- Add domain-specific moves for your stack.
- Adjust how Case B (no content provided) routes — current default prefers "instruct the executor to fetch via tools" over "ask the user to paste content."
- Tune capability-aware rewriting behavior.

`references/router.md` controls capability priority and matching. `references/safety.md` controls destructive-pattern detection.

## License

MIT. See LICENSE.
