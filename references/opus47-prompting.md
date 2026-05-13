# Opus 4.7 prompting cookbook (for `/improve` rewrites)

This is the content playbook the rewrite subagent applies. Source: distilled from SKILL-47 (Opus 4.7 prompt optimizer), adapted for Claude Code's execute-not-paste context.

## How this differs from the chat-app version

The rewrite produced by `/improve` is **executed by the next agent turn**, not pasted into chat. That changes three things from the original SKILL-47:

1. **Do NOT append** "Think before answering (maximum reasoning)". That line triggers adaptive thinking in claude.ai/Mac/iOS. Inside Claude Code, it is at best noise and at worst a category error. Drop it.
2. **Do NOT wrap the rewrite in a fenced code block.** The wrapper handles formatting. Return the rewrite as plain text between the markers the wrapper expects.
3. **Soften Case B's "ask the user" reflex.** Inside Claude Code the executor has tool access — it can often Read, Glob, or Grep for missing inputs itself rather than punting to the user. Prefer "instruct the executor to look in `<likely path>` first; if not found, ask" over "ask the user to paste it."

Everything else from the cookbook applies directly — Opus 4.7 reads prompts more literally than older models, calibrates its own thinking and length to perceived complexity, and rewards prompts that are specific, structured, and motivated.

## Two hard rules

### Rule 1 — No placeholders. Ever.

Never produce a rewrite containing `[paste X here]`, `[your content]`, `{topic}`, `<your_input_here>`, `[INSERT Y]`, `___`, or any other template variable the executor is expected to fill. The executor must be able to act on the rewrite as-is. If content is missing, either bake what you have in, or instruct the executor to fetch / Read / ask — never leave brackets.

Before finalizing, scan your rewrite for `[`, `{`, or `<...your...>`-style placeholders. Kill any you find.

### Rule 2 — Ship a finished prompt no matter what the user gave you.

**Case A — user provided real content** (draft, code, document, list, specific question, product description). Bake that content directly into the rewrite. The whole thing — content and instructions — is the rewrite.

**Case B — user described a class of task** ("triage my emails", "review my code", "write LinkedIn posts about my launches"). Write a self-contained instruction. End by either:
- Telling the executor where to look first ("Read the most recent `.eml` files in `./inbox/`; if none, ask the user to paste a batch") — prefer this inside Claude Code, and
- Falling back to asking the user only when no tool-accessible source exists.

Either way: no brackets, no fill-in-the-blank, no template syntax. The rewrite is final.

## The rewrite workflow (internal — don't surface)

1. **Identify the goal.** What does the user actually want produced? A document? A decision? Code? A list? An analysis? Name it concretely.
2. **Identify the audience and use.** Who reads the output, and what will they do with it? Drives tone and format.
3. **Decide Case A or Case B.** Did the user provide actual content, or just describe a class of task?
4. **Spot the gaps.** Audience, format, length, constraints, examples, edge cases — note which are missing.
5. **Fill gaps with reasonable assumptions** grounded in what they wrote. The wrapper told them not to expect clarifying questions; make the most defensible assumption and move on.
6. **Factor in available capabilities.** If the wrapper has chosen a route (a skill, MCP prompt, agent), the rewrite should invoke that capability by name and lean on its conventions. Do not reimplement what the capability already does.
7. **Pick a structure.** Single-paragraph instruction for simple tasks. XML tags for anything with multiple sections.
8. **Write the rewrite.** Apply the principles below.
9. **Scan for brackets.** Kill any placeholders.

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
If the rewrite includes a long document, transcript, or data dump, place it at the top. Anthropic's testing shows up to ~30% quality lift from this ordering on long-context tasks.

### Ask for grounding in long-document tasks
For analysis or Q&A over long inputs, instruct the executor to first pull relevant quotes into `<quotes>` tags, then answer based on those quotes. Dramatically reduces drift and hallucination.

### Be literal about scope
Opus 4.7 doesn't silently generalize. If you want an instruction applied broadly, say "apply this to every section, not just the first one." Use imperative verbs ("Edit the function to..." not "Could you suggest improvements to..."). Suggestion-flavored phrasing produces suggestions.

### Self-check for high-stakes outputs
For code, math, claims, or anything where errors matter, append near the end: "Before you finish, re-read your answer and check it against the criteria above." Catches errors reliably.

### Do NOT add competing thinking instructions
Opus 4.7 in Claude Code already operates in an agent loop with its own reasoning calibration. Do not add "think step by step", "reason carefully", or similar nudges — they create noise. Trust the model's adaptive thinking.

## Domain-specific moves

Apply only when relevant.

**Frontend / design.** Opus 4.7 has a strong default house style (warm cream backgrounds, serif type, terracotta accents) that's wrong for most products. If the user is asking for a design, either (a) specify a concrete alternative palette, type system, and structure in detail, or (b) instruct the executor to propose 3–4 distinct visual directions before building, so the user picks one. Generic instructions like "make it clean and minimal" don't break the default.

**Code review.** Tell the executor its job at the finding stage is coverage, not filtering: "Report every issue you find, including ones you're uncertain about or consider low-severity. Include confidence and severity for each finding so a downstream filter can rank them." Avoid soft language like "only flag important issues" — Opus 4.7 takes that literally and under-reports.

**Research / analysis.** Encourage hypothesis-tracking: "Develop several competing hypotheses as you gather information. Track confidence levels in your notes. Self-critique your approach periodically." Produces more rigorous synthesis than flat "research X."

**Creative writing.** Specify voice, audience, length, constraints. Include one or two example sentences in the target voice if available. Generic "write a blog post" yields generic prose.

**Document creation (slides, reports).** Ask for design intentionality: "Include thoughtful visual hierarchy, considered typography, and engaging structure." Opus 4.7 produces strong first-pass document design when invited to.

## Capability-aware rewriting (Claude Code-specific)

When the wrapper has routed to a specific local capability, the rewrite must:

- **Invoke the capability by name.** "Use the `youtube_ultimate` skill to..." or "Call `mcp__notion__create_page` with...". Don't paraphrase or reimplement.
- **Respect the capability's input contract.** If the SKILL.md specifies an argument-hint or expected inputs, conform the rewrite to that shape.
- **Stay inside the capability's scope.** If the user's raw prompt asks for things the routed capability can't do, surface that mismatch in the rewrite by adding a step ("After the capability finishes, also do X using native tools.") rather than abandoning the routing decision.
- **For `native tools` fallback**, name the specific tools the executor should use (Read / Glob / Grep / Bash / Edit / Write) and the order.

## Output format for the rewrite

Return plain text. No fenced code block. No preamble like "Here's your prompt:". No trailing "I changed X and Y."

Do NOT append "Think before answering (maximum reasoning)" — that line is for the chat app, not Claude Code.

## Edge cases

- **User input is already excellent.** Tighten where you can, return it. No added ceremony.
- **User input is non-English.** Write the rewrite in the same language.
- **User pastes a system prompt or API-style prompt with parameters.** Strip API-only mechanics (effort levels, thinking config, tool definitions), translate intent into a single executable Claude Code prompt.
- **User wants many small tasks.** Combine into a single coherent rewrite with clear sections, not a numbered list of micro-tasks. Opus 4.7 handles long, well-structured asks well.
- **Tempted to write `<context>` or `<input>` block expecting the user to fill it.** Don't. That's Rule 1. Either bake content in (Case A), instruct the executor to fetch via tools (Case B preferred inside Claude Code), or — last resort — instruct the executor to ask the user.
