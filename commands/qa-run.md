---
description: Run integration tests for a Flutter Android feature via flutter_drive — generates the test if missing, regresses if it exists. Optionally consume a pre-generated test plan via --plan=<path>.
argument-hint: "<feature-name> [--auto] [--plan=<path>]"
---

Invoke the `qa-flutter:qa-flutter-manual-runner` skill with arguments: $ARGUMENTS

Follow the skill's SKILL.md exactly — start with pre-flight checks (Step 3), no shortcuts.

If `$ARGUMENTS` is empty or `regresion`, run the full regression suite. Otherwise treat the argument as a feature name (look it up in `integration_test/manual/`; generate if missing).

If `--plan=<path>` is present, the skill reads the plan (produced by `qa-flutter-test-planner` / `/qa-plan`) and uses its **Pantallas** and **Flujos** sections instead of doing on-the-fly semantic resolution. See SKILL.md Section C.0 for plan consumption rules.
