---
description: Generate an auditable QA test plan (pantallas, precondiciones, flujos, criterios, riesgos) for a Flutter feature — to be reviewed and then consumed by the runners via --plan.
argument-hint: "<feature-name> [--auto] [--output=<path>]"
---

Invoke the `qa-flutter:qa-flutter-test-planner` skill with arguments: $ARGUMENTS

Follow the skill's SKILL.md exactly — start with Step 1 (parse arguments) and proceed through Step 9 (output).

If `$ARGUMENTS` is empty, abort with the usage hint.

The skill writes a markdown plan to `qa-plugin-config/qa-plans/<feature>.md` (or the path given via `--output=`). After the plan is written, suggest the next step:

```
/qa-run <feature> --plan=<path-to-plan>
```

In `--auto` mode, the skill skips per-section prompts and just writes the plan; useful for CI / scheduled refreshes of plan artifacts.
