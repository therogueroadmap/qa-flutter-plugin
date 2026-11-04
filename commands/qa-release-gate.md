---
description: Production GO/NO-GO release gate — runs the full QA stack via qa-stability-agent, classifies findings, emits a binary verdict and JSON artifact.
argument-hint: "[--threshold=strict|normal|lenient] [--version=vX.Y.Z] [--auto]"
---

Invoke the `qa-flutter:qa-flutter-release-gate` skill with arguments: $ARGUMENTS

Follow the skill's SKILL.md exactly — Step -1 (bootstrap `--up`) → Step 7 (report + JSON) → Step 9 (teardown). Never skip the teardown step, even on failure.

Default `--threshold` is `normal` if omitted. `--version` is optional; if absent, the gate uses the timestamp as the run identifier.
