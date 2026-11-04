---
description: Bring up or tear down the QA infrastructure (backend + Android device) declared in qa-agent.yaml. Use --down to recover from a crashed run.
argument-hint: "--up | --down [--caller=<name>]"
---

Invoke the `qa-flutter:qa-flutter-bootstrap` skill with arguments: $ARGUMENTS

Follow the skill's SKILL.md exactly. The skill is idempotent — `--down` only tears down what the marker says was started, and handles dead PIDs gracefully.

Typical use cases:
- `--up` — start backend + verify device manually before invoking a runner.
- `--down` — recover from a SIGKILL or power loss that left a stale marker at `<reports-dir>/.qa-bootstrap-marker`.

Most users do not invoke this directly; `qa-flutter-release-gate` and `qa-stability-agent` already wrap it.
