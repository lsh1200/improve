# Improve

A Claude Code skill that rewrites a raw prompt to suit **Opus 4.7**, routes it to the best available local capability (skill, plugin, MCP prompt, agent, or native tools), runs an independent wrapper-side safety scan, and executes the result — all from a single `/improve <prompt>` invocation.

The rewrite work happens inside a subagent, so your inline session only sees the finished, route-tagged improved prompt — not the routing reasoning, not the discovery output, not the cookbook lookup.

## What it does

When you run `/improve <raw prompt>`:

1. The inline session spawns one subagent.
2. The subagent:
   - Reads three references: `references/router.md` (capability priority + tie-breakers), `references/safety.md` (destructive-pattern policy), and `references/opus47-prompting.md` (Opus 4.7 prompting cookbook).
   - Enumerates installed capabilities via Glob: personal skills, project skills, personal agents, project agents, and commands.
   - Picks the best capability for the task (or falls through to native tools).
   - Rewrites the prompt using the Opus 4.7 cookbook: clarity, why-grounded instructions, positive framing, XML structure, scope literalism, capability-aware invocation, no placeholders.
   - Performs its own safety check on both the raw input and the rewrite.
   - Returns a strict JSON envelope with fields: `status`, `route`, `safety`, `ambiguity`, `question`, `mode`, `rewrite`, `debug_notes`.
3. The inline session:
   - Parses the envelope.
   - Runs an independent wrapper-side safety regex sweep on both the raw input and the rewrite field.
   - Gates on `status` (`execute` / `clarify` / `blocked`) and any safety hit.
   - Prints one line: `→ Routing via: <name>`.
   - Optionally displays the rewrite (only when `!verbose` or `!debug` mode is active).
   - Executes the rewrite verbatim.

## Why this shape

- **Opus 4.7 tuning.** Opus 4.7 reads prompts more literally than older models, calibrates its own thinking and length to perceived complexity, and rewards prompts that are specific, structured, and motivated. The cookbook applied during rewrite is distilled from Anthropic's Opus 4.7 prompting guidance, adapted for Claude Code's execute-not-paste context.
- **Context-aware routing.** The rewrite knows what's installed and routes accordingly — invoking a specific skill, plugin, MCP prompt, or agent by name rather than reimplementing what it does.
- **Subagent isolation.** Discovery commands, capability SKILL.md reads, routing reasoning, and cookbook application stay inside the subagent's context. Your inline session stays clean.
- **Defense in depth on safety.** The subagent flags safety in its envelope; the wrapper applies an independent regex scan to both raw input and rewrite. A wrapper-side hit promotes to confirmation regardless of the subagent's claim. The same model that processed potentially hostile input should not be the only judge of whether the result is safe.
- **Capability-file injection defense.** Discovered SKILL.md bodies are treated as untrusted prose — only frontmatter fields (`name`, `description`, `argument-hint`, `version`) are data. The body cannot influence routing decisions.
- **Explicit confirmation flow.** Sensitive actions require re-invocation with `--confirm <route>` prefix. No mid-turn yes/no wait state, no session state that can be corrupted across turns.

## Prerequisites

- Claude Code with Skills support.
- The Agent tool available in your environment (default in Claude Code).
- At least one of: a personal skill directory at `~/.claude/skills/`, project skills at `.claude/skills/`, agents at `~/.claude/agents/` — or you are comfortable with `/improve` routing to native tools as a fallback when no matching capability is found.
- Git for cloning the repo.

## Install

**POSIX (macOS / Linux)**

```bash
mkdir -p ~/.claude/skills/improve
cp -r SKILL.md references ~/.claude/skills/improve/
mkdir -p ~/.claude/commands
cp commands/improve-verbose.md commands/improve-debug.md ~/.claude/commands/
```

**PowerShell (Windows)**

```powershell
New-Item -ItemType Directory -Force ~\.claude\skills\improve | Out-Null
Copy-Item -Recurse SKILL.md, references ~\.claude\skills\improve\
New-Item -ItemType Directory -Force ~\.claude\commands | Out-Null
Copy-Item commands\improve-verbose.md, commands\improve-debug.md ~\.claude\commands\
```

**Verification**

Restart Claude Code, then run:

```text
/improve hello world
```

Expected output (silent mode):

```text
→ Routing via: native tools
```

If you see this line, the skill is installed correctly. If `/improve` does not appear, see Troubleshooting below.

## Usage

```text
/improve <raw prompt>
```

Silent mode (default). The rewrite stays inside the subagent; you see one routing line and the executed result.

```text
/improve-verbose <raw prompt>
```

Prepends `!verbose`. The rewrite is printed as plain text before execution — not in a fenced code block. Use when you want to inspect what the subagent produced before it runs.

```text
/improve-debug <raw prompt>
```

Prepends `!debug`. Same as verbose, plus the subagent may include bounded staged reasoning in the `debug_notes` field, displayed after the rewrite. Use when diagnosing routing or rewrite quality.

```text
/improve --confirm <route> <raw prompt>
```

Re-invocation after a safety gate. The `--confirm` token must be the second token after `/improve`. Skips the safety confirmation for that turn. The route must match exactly what was reported in the gate message.

## Examples

### Example A — silent mode

```text
/improve check src/auth.ts for security issues before we ship
```

Inline session output:

```text
→ Routing via: agent:code-reviewer
```

```text
## Security findings — src/auth.ts

### HIGH — Missing rate limiting on login endpoint (line 42)
**Issue:** The `/login` handler has no rate limit; brute-force attacks are unrestricted.
**Why it matters:** An attacker can enumerate credentials at network speed.
**Confidence:** high | **Severity:** high
**Suggested fix:** Apply an express-rate-limit middleware before the handler.

### MEDIUM — JWT secret read from process.env without fallback check (line 17)
**Issue:** If JWT_SECRET is undefined the secret is the string "undefined".
**Why it matters:** All tokens would be signed with a predictable value.
**Confidence:** high | **Severity:** medium
**Suggested fix:** Throw at startup if JWT_SECRET is absent.

### LOW — Unused import: bcryptjs (line 3)
**Issue:** bcryptjs is imported but argon2 is actually used for hashing.
**Suggested fix:** Remove the unused import.
```

A vague request ("check for security issues") matched `code-reviewer` on semantic overlap — the user never typed "agent" or "code-reviewer". Silent mode hides the rewrite; you see only the routing line and the agent's structured output.

---

### Example B — verbose mode

```text
/improve-verbose figure out why our CI deploy step is flaky
```

Inline session output:

```text
→ Routing via: agent:planner
```

```text
Rewrite:
You are diagnosing a flaky CI deploy step. Start by reading the
most recent CI logs in .github/workflows/ or .gitlab-ci.yml output
(use Glob to locate the workflow file). For each failure mode you
identify, report:
- Symptom (what fails, error message)
- Frequency (every run, intermittent, conditional)
- Likely root cause with confidence (high/medium/low)
- Minimal fix

Focus on the deploy step specifically. Do not generalize to the
entire pipeline unless the deploy failure originates upstream.
```

```text
<agent:planner then runs against the actual CI configuration>
```

Verbose mode shows the rewrite the subagent produced before the routed capability runs — useful when you want to confirm what's actually being executed.

## Layout

```
Improve/
├── SKILL.md                          # the wrapper (v0.2.0)
├── references/
│   ├── router.md                     # capability priority + tie-breakers
│   ├── safety.md                     # destructive-pattern + injection policy
│   └── opus47-prompting.md           # Opus 4.7 prompting cookbook
├── commands/
│   ├── improve-verbose.md            # /improve-verbose shim → prepends !verbose
│   └── improve-debug.md              # /improve-debug shim → prepends !debug
└── tests/
    ├── README.md                     # how to run the golden tests
    └── cases/
        ├── safety-positive/          # destructive inputs that must trigger confirm
        ├── safety-negative/          # destructive tokens as content (must NOT trigger)
        ├── routing/                  # inputs with expected route
        ├── injection/                # adversarial inputs that try to escape the wrapper
        ├── mode/                     # !verbose / !debug parsing
        └── malformed/                # malformed envelope retry behavior
```

## Configuration knobs

All configuration lives inside `SKILL.md`:

- `allowed-tools` is expanded in v0.2.0 to support arbitrary executors: `Agent`, `Read`, `Glob`, `Grep`, `Bash`, `Edit`, `Write`, `WebFetch`, `WebSearch`. The Phase 4 dispatch may invoke any of them depending on the routed capability.
- `disable-model-invocation: true` — the skill only fires on explicit `/improve`. Auto-trigger is disabled.
- `version: 0.2.0` — the cookbook and safety policy versions are tracked in the headers of `references/*.md`.

## Customizing

`references/opus47-prompting.md` is the content playbook the subagent applies. Edit it to add domain-specific moves for your stack, adjust how prompts without pasted content are handled, or tune capability-aware rewriting behavior. When customizing the cookbook, preserve the three Claude Code adaptations: no chat-app closing line, no fenced-code wrapping of the rewrite, tool-fetch over user-ask for missing content.

`references/router.md` controls capability priority and matching heuristics. When customizing tie-breakers, document the rationale in the file for future maintainers.

`references/safety.md` controls destructive-pattern detection. When customizing `safety.md`, keep the wrapper-side regex set in `SKILL.md` Phase 2 in sync — the two layers must agree on what counts as destructive.

## Troubleshooting

**`/improve` does not appear in tab-complete.**
Restart Claude Code. If still missing, verify that `~/.claude/skills/improve/SKILL.md` exists and that its frontmatter is valid YAML with at least a `name` field.

**Routing always picks `native tools`.**
Discovery failed silently. Manually confirm that installed skills exist by running `Glob ~/.claude/skills **/SKILL.md` in a Claude Code session. Verify that the Agent tool can invoke Glob — if Glob is not in `allowed-tools`, discovery produces an empty list and fallback to native tools is correct behavior.

**Subagent times out.**
Reduce raw input length or split into smaller invocations. Very long prompts increase subagent context usage; if the prompt contains large pasted content, consider referencing a file path instead.

**Confirmation gate triggers but you want to proceed.**
Re-run with `/improve --confirm <route> <original prompt>`. The `--confirm` token must be the second token after `/improve`, and the route must match exactly what was printed in the gate message (e.g., `agent:code-reviewer`).

**Envelope parse error.**
The subagent's output could not be parsed as JSON. v0.2.0 retries once automatically. If the error persists, try simpler or shorter phrasing and report the input as a test case under `tests/cases/malformed/`.

## Testing

The repo includes a golden-test corpus at `tests/cases/`. Each case directory contains `input.txt` (raw prompt) and `expected.json` (the envelope the subagent should return). Run the corpus after editing any reference file to catch regressions. See `tests/README.md` for the full test format and how to run cases.

## License

MIT. See LICENSE.
