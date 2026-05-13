# /improve v0.2.0 — Golden Test Corpus

This directory contains the reference fixture set for the `/improve` skill v0.2.0.
Each case documents what the subagent **should** return for a given raw input.

## Format

Each runnable case lives in a subdirectory under `tests/cases/<category>/` and
contains two files:

| File | Contents |
|------|----------|
| `input.txt` | The raw text the user would type after `/improve` |
| `expected.json` | The JSON envelope the subagent should return for that input |

`expected.json` follows the v0.2.0 JSON envelope schema. Fields whose exact
value depends on the installed capability set are annotated with a `__notes__`
array field (a JSON-legal annotation convention used throughout this corpus).

Non-runnable cases (like `malformed/retry-behavior.md`) are plain markdown
documents describing behavioral contracts that cannot be expressed as a single
input/output pair.

## Schema reference

`expected.json` conforms to the canonical envelope schema defined in
`SKILL.md` frontmatter and elaborated in:

- `references/router.md` — `route` field values and routing tie-break rules
- `references/safety.md` — `safety` field semantics and the wrapper-side regex set

Required fields and their constraints:

```
status       "execute" | "clarify" | "blocked"
route        capability name, or "none" when status != "execute"
safety       "none" or short description of destructive pattern detected
ambiguity    "none" or one-line alt route hint
question     "" when status="execute"; clarifying question or refusal reason otherwise
mode         "silent" | "verbose" | "debug"
rewrite      JSON-escaped enriched prompt; "" when status != "execute"
debug_notes  "" unless mode="debug"
```

## How to run (v0.2.0 — manual)

There is no automated test runner in this version. These fixtures are reference
documents for human review and for future automated harness construction.

To manually verify a case:

1. Copy the contents of `input.txt`.
2. Run `/improve !debug <pasted contents>` (prepend `!debug` so the envelope
   is visible in the session output).
3. Compare the actual envelope fields to `expected.json`.
4. Check `__notes__` for acceptable variations before marking a field as wrong.

## How to add new cases

1. Pick the most specific category folder from the table below.
2. Create a new subdirectory with a descriptive slug, e.g. `cases/routing/my-new-case/`.
3. Add `input.txt` (raw user input) and `expected.json` (expected envelope).
4. Add a `__notes__` field for any fields that have acceptable variation.
5. Update the case count in the summary table below.

## Categories

| Category | Cases | Purpose |
|----------|-------|---------|
| `safety-positive` | 3 | Inputs containing destructive instructions that MUST trigger the confirmation gate |
| `safety-negative` | 3 | Inputs mentioning destructive tokens as content for analysis — MUST NOT trigger |
| `routing` | 4 | Inputs with a clear expected route; validates router heuristics and tie-breaks |
| `injection` | 4 | Adversarial inputs attempting to escape the raw_input wrapper, fake the envelope, or claim elevated privileges |
| `mode` | 3 | Validates `!verbose` / `!debug` flag detection; mid-string occurrences must not parse as flags |
| `malformed` | — | Prose contract for retry behavior; no runnable input/expected pair |

## Validation notes

- `__notes__` fields in `expected.json` list acceptable variations. They are not
  part of the envelope schema and must be ignored by any future JSON validator.
- Routes marked as environment-dependent (e.g., `vague-bug-hunt`) list multiple
  acceptable values. The test passes if the actual route matches any listed value.
- `safety-negative` cases are the most critical for false-positive detection.
  A safety field of `"none"` is a hard requirement in those cases.
- The `malformed/retry-behavior.md` file documents the one-retry contract.
  The exact abort message `/improve: subagent returned malformed envelope after
  one retry. Aborting.` is normative and must match exactly in any future runner.
