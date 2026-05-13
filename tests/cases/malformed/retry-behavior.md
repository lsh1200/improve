# Malformed Envelope Retry Behavior

This file documents the wrapper's retry contract for malformed subagent output.
It is not a runnable fixture — there is no `input.txt` or `expected.json` here.
It describes behavioral requirements for future test automation.

## What counts as malformed

The subagent is expected to return exactly one JSON object on stdout with no
preamble or markdown fence. Output is malformed if any of these are true:

- The output is not parseable as JSON (syntax error).
- The parsed object is missing one or more required fields (`status`, `route`,
  `safety`, `ambiguity`, `question`, `mode`, `rewrite`, `debug_notes`).
- `status` contains a value other than `"execute"`, `"clarify"`, or `"blocked"`.
- `route` is `"none"` when `status` is `"execute"`.
- `rewrite` is non-empty when `status` is `"clarify"` or `"blocked"`.
- `question` is non-empty when `status` is `"execute"`.

## Retry protocol

When the wrapper detects malformed output:

1. The wrapper logs the raw subagent output internally (not shown to user).
2. The wrapper re-invokes the subagent exactly once with a correction prompt
   appended to the original system prompt:

   ```
   Your previous response was not valid JSON or did not conform to the required
   envelope schema. Return ONLY a valid JSON object matching the schema below,
   with no preamble, no markdown fence, and no trailing text.

   Schema: { "status", "route", "safety", "ambiguity", "question", "mode",
             "rewrite", "debug_notes" }
   ```

3. If the retry response is also malformed, the wrapper aborts with:

   ```
   /improve: subagent returned malformed envelope after one retry. Aborting.
   ```

   The raw malformed output is NOT shown to the user (it may contain partial
   or injected content). The abort message is the only output.

## Rationale

One retry balances resilience against cost. Two retries were considered but
rejected: a model that fails twice on the same JSON schema is unlikely to
succeed on a third attempt without a fundamentally different prompt, and
repeated failures may indicate a prompt-injection or context-corruption issue
where aborting is safer than persisting.

## Future automation notes

To exercise this contract in automated tests:

- Stub the subagent to return `not json at all` on first call, valid JSON on
  second call. Verify the wrapper succeeds after one retry.
- Stub the subagent to return `not json at all` on both calls. Verify the
  wrapper emits exactly the abort message above and exits non-zero.
- Stub the subagent to return `{"status": "execute"}` (missing fields) on first
  call, valid JSON on second call. Verify the wrapper retries on schema
  violations, not just parse errors.
