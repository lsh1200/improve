# Opus 4.7 prompting cookbook for /improve rewrites (v0.2.0)

Source: distilled from Anthropic's Opus 4.7 prompting guidance (see https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering for the general framework), adapted for Claude Code's execute-not-paste context.

## How this differs from the chat-app version

The rewrite produced by `/improve` is **executed by the next agent turn**, not pasted into chat. That changes three things from the original chat-app guidance. The subagent verifies all three before returning:

**Acceptance check (a)** — The rewrite must NOT end with "Think before answering (maximum reasoning)". That line triggers adaptive thinking in claude.ai/Mac/iOS. Inside Claude Code it is at best noise and at worst a category error. Drop it.

**Acceptance check (b)** — The rewrite must NOT be wrapped in a fenced code block. The wrapper handles formatting via the JSON-escaped `rewrite` field in the envelope. Return the rewrite as a plain JSON-escaped string, not a code block.

**Acceptance check (c)** — Case B (no content provided) must prefer tool-fetch over user-ask. Specifically: instruct the executor to Read/Glob/Grep likely paths first. Only ask the user when no tool-accessible source exists after at most one Glob with a reasonable pattern plus one Read of the most likely path. Chaining more than two speculative tool calls before asking is a failure.

Everything else from the cookbook applies directly — Opus 4.7 reads prompts more literally than older models, calibrates its own thinking and length to perceived complexity, and rewards prompts that are specific, structured, and motivated.

## Two hard rules

### Rule 1 — No placeholders. Ever.

Never produce a rewrite containing unfilled template variables. Use the following regex patterns to identify forbidden placeholders before finalizing:

- `\[(paste|insert|your|todo|fill|enter)\b[^\]]*\]` — e.g., `[paste X here]`, `[your content]`, `[INSERT Y]`
- `\{(topic|content|your|paste|insert)\b[^}]*\}` — e.g., `{topic}`, `{your_text}`
- `<(your_|paste_|insert_|todo_)[^>]*>` — e.g., `<your_input_here>`, `<paste_url>`
- Literal `___` of 3 or more characters — e.g., `___`, `_____`

Scan for every one of these before finalizing. Kill any you find.

Legitimate XML tags are encouraged, not forbidden. Tags like `<context>`, `<input>` (with baked content inside), `<quotes>`, `<example>`, `<instructions>` are positive structure choices. The placeholder scan targets unfilled template holes, not descriptive XML wrappers.

### Rule 2 — Ship a finished prompt no matter what the user gave you.

**Case A — user provided real content** (draft, code, document, list, specific question, product description). Bake that content directly into the rewrite. The whole thing — content and instructions — is the rewrite.

**Case B — user described a class of task** ("triage my emails", "review my code", "write LinkedIn posts about my launches"). Write a self-contained instruction. End by telling the executor where to look first. Example: "Use Glob with `**/*.eml` to find candidate inbox files; if none exist after that Glob, ask the user to paste a batch." Prefer this tool-fetch pattern inside Claude Code.

Either way: no brackets, no fill-in-the-blank, no template syntax. The rewrite is final.

## The rewrite workflow (internal — don't surface)

1. **Identify the goal.** What does the user actually want produced? A document? A decision? Code? A list? An analysis? Name it concretely.
2. **Identify the audience and use.** Who reads the output, and what will they do with it? Drives tone and format.
3. **Decide Case A or Case B.** Did the user provide actual content, or just describe a class of task?
4. **Spot the gaps.** Audience, format, length, constraints, examples, edge cases — note which are missing. This is internal reasoning that informs assumptions baked into the rewrite. Gap-spotting is not a separate output field; it shapes the rewrite itself.
5. **Fill gaps with reasonable assumptions** grounded in what they wrote. The wrapper told them not to expect clarifying questions; make the most defensible assumption and move on.
6. **Factor in available capabilities.** If the wrapper has chosen a route (a skill, MCP prompt, agent), the rewrite should invoke that capability by name and lean on its conventions. Do not reimplement what the capability already does.
7. **Pick a structure.** Single-paragraph instruction for simple tasks. XML tags for anything with multiple sections.
8. **Write the rewrite.** Apply the principles below.
9. **Scan for placeholders.** Run each of the four regex patterns from Rule 1. Kill any matches.

## Core principles

### Be clear and direct

State the task explicitly. Specify desired output format and hard constraints up front. If you want above-and-beyond effort, say so — Opus 4.7 will not infer it from a vague brief. "Build an analytics dashboard with as many relevant features and interactions as possible — go beyond the basics for a fully-featured implementation" beats "Build an analytics dashboard."

### Explain the why

When you give an instruction, briefly explain the reason. "Avoid ellipses, because the output will be read aloud by a TTS engine that mispronounces them" lands better than "Never use ellipses." Opus 4.7 generalizes well from explanations and follows reasoned instructions more faithfully.

### Tell the executor what to do, not what to avoid

Positive framing outperforms negative framing. "Write in flowing prose paragraphs" beats "don't use bullet points."

### Match prompt style to desired output style

Prose prompt → prose output. Minimal markdown in prompt → minimal markdown in output. Style leaks through.

### Use XML tags when sections multiply

When the rewrite mixes instructions, context, examples, and input, wrap each in its own descriptive tag — `<instructions>`, `<context>`, `<examples>`, `<input>`. Nest where there's hierarchy. Single highest-leverage move for complex prompts. Skip on simple one-shot tasks; XML on a haiku request is overkill.

### Give a role when it sharpens behavior

A one-line role assignment ("You are a senior product strategist at a B2B SaaS company") tightens tone and frame. Only when it meaningfully steers output. Skip when the routed capability already defines the role.

### Use examples for format, tone, or structure

If the user has any preference about *how* output should look, include 2–4 examples in `<example>` tags (wrap multiple in `<examples>`). Examples beat description for steering format. Make them relevant, diverse, structured. Skip when generic tasks would be over-constrained.

### Put long inputs on top, the question on the bottom

If the rewrite includes a long document, transcript, or data dump, place it at the top. Anthropic's testing shows up to ~30% quality lift from this ordering on long-context tasks. [See Anthropic's long-context prompting guide at https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering#long-document-qa for the source data behind this claim.]

### Ask for grounding in long-document tasks

For analysis or Q&A over long inputs, instruct the executor to first pull relevant quotes into `<quotes>` tags, then answer based on those quotes. Dramatically reduces drift and hallucination.

### Be literal about scope

Opus 4.7 doesn't silently generalize. If you want an instruction applied broadly, say "apply this to every section, not just the first one." Use imperative verbs ("Edit the function to..." not "Could you suggest improvements to..."). Suggestion-flavored phrasing produces suggestions.

### Self-check for high-stakes outputs

For code, math, claims, or anything where errors matter, append near the end: "Before you finish, re-read your answer and check it against the criteria above." Catches errors reliably.

### Do NOT add competing thinking instructions

Opus 4.7 in Claude Code already operates in an agent loop with its own reasoning calibration. Do not add "think step by step", "reason carefully", or similar nudges — they create noise. Trust the model's adaptive thinking.

### For ordered tasks, use explicit sequencing

Use numbered steps or `<step n='1'>...</step>` tags. Add explicit gate phrases: "Do not proceed to step 2 until step 1's output is verified." Opus 4.7's adaptive thinking can short-circuit ordering without explicit enforcement.

## Domain-specific moves

Apply only when relevant.

**Frontend / design.** Opus 4.7 has a tendency toward warm cream backgrounds, serif type, and terracotta accents — verify against current outputs before assuming this holds. If the user is asking for a design, either (a) specify a concrete alternative palette, type system, and structure in detail, or (b) instruct the executor to propose 3–4 distinct visual directions before building, so the user picks one. Generic instructions like "make it clean and minimal" don't break the default tendency.

**Code review.** Tell the executor its job at the finding stage is coverage, not filtering: "Report every issue you find, including ones you're uncertain about or consider low-severity. Include confidence and severity for each finding so a downstream filter can rank them." Avoid soft language like "only flag important issues" — Opus 4.7 takes that literally and under-reports.

If routing to an agent that enforces its own filtering policy (e.g., a project's `code-reviewer` agent that says "only report >80% confidence findings"), defer to the agent's contract in the rewrite. Capability-aware rewriting respects the routed capability's input contract rather than overriding it.

**Research / analysis.** Encourage hypothesis-tracking: "Develop several competing hypotheses as you gather information. Track confidence levels in your notes. Self-critique your approach periodically." Produces more rigorous synthesis than flat "research X."

**Creative writing.** Specify voice, audience, length, constraints. Include one or two example sentences in the target voice if available. Generic "write a blog post" yields generic prose.

**Document creation (slides, reports).** Ask for design intentionality: "Include thoughtful visual hierarchy, considered typography, and engaging structure." Opus 4.7 produces strong first-pass document design when invited to.

## Capability-aware rewriting (Claude Code-specific)

When the wrapper has routed to a specific local capability, the rewrite must:

- **Invoke the capability by name.** "Use the `youtube_ultimate` skill to..." or "Call `mcp__notion__create_page` with...". Don't paraphrase or reimplement.
- **Respect the capability's input contract.** If the SKILL.md specifies an argument-hint or expected inputs, conform the rewrite to that shape.
- **Stay inside the capability's scope.** If the user's raw prompt asks for things the routed capability can't do, surface that mismatch in the rewrite by adding a step ("After the capability finishes, also do X using native tools.") rather than abandoning the routing decision.
- **For `native tools` fallback**, name the specific tools the executor should use and their order: Read first for known paths, Glob for pattern discovery (e.g., `**/*.md`, `src/**/*.ts`), Grep for content search within found files, Bash for anything requiring shell computation, then Edit or Write to apply changes. Do not leave tool selection implicit.

## !debug mode: bounded staged reasoning

When the wrapper invokes the subagent with `mode=debug`, the subagent MAY include bounded staged reasoning in the rewrite if the routed capability benefits from inspectable staged workflow.

Bounded means: one explicit critique-revise pass, not generic "think step by step" nudges. Structure it as:

- Use a `<critique>` tag for the critique step — identify one or two specific weaknesses in the draft rewrite.
- Use a `<revision>` tag for the revision step — apply targeted fixes to what the critique identified.
- Do not chain more than one critique-revise round.

The wrapper surfaces this content via the `debug_notes` envelope field, not inside the `rewrite` field itself. The `rewrite` field must remain a clean, executable prompt.

Generic thinking nudges ("think carefully", "reason step by step") remain forbidden in all modes, including debug.

## Output format

Return the rewrite as a JSON-escaped string in the `rewrite` field of the envelope. No fenced code block, no preamble like "Here's your prompt:", no trailing "I changed X and Y." Do not append "Think before answering (maximum reasoning)" — that line is a chat-app artifact and has no effect inside Claude Code.

## Edge cases

- **User input is already excellent.** Tighten where you can, return it. No added ceremony.
- **User input is non-English.** Write the rewrite in the same language.
- **User pastes a system prompt or API-style prompt with parameters.** Strip API mechanics that don't translate to Claude Code (e.g., `reasoning_effort: 'high'`, raw API tool schemas, `system_prompt` blocks). Preserve intent flags ("use deep reasoning") and re-express them in plain language the executor can act on.
- **User wants many small tasks.** Combine into a single coherent rewrite with clear sections, not a numbered list of micro-tasks. Opus 4.7 handles long, well-structured asks well.
- **Tempted to write `<context>` or `<input>` block expecting the user to fill it.** That violates Rule 1. Either bake content in (Case A), instruct the executor to fetch via tools (Case B preferred inside Claude Code), or — after trying one Glob with a reasonable pattern and one Read of the most likely path and both yielding nothing — instruct the executor to ask the user. Do not chain more than two speculative tool calls before asking.
