---
description: Run /improve with !debug — shows the rewrite in a fenced block before executing; allows bounded staged reasoning.
argument-hint: <raw prompt>
---

Invoke the `improve` skill via the Skill tool. Pass these arguments exactly, with `!debug` prepended so the skill's Phase 2 rewrite is printed and bounded critique-revise patterns are permitted:

!debug $ARGUMENTS
