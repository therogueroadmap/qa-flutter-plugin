# Contributing to qa-flutter

## Repo layout

```
.claude-plugin/plugin.json      ← plugin manifest (name, version, author)
agents/                         ← orchestrator sub-agents
  qa-stability-agent.md
commands/                       ← slash-command wrappers
  qa-run.md                     ← /qa-run <feature>
  qa-unit.md                    ← /qa-unit <feature>
  qa-release-gate.md            ← /qa-release-gate
  qa-bootstrap.md               ← /qa-bootstrap --up | --down
skills/                         ← skill definitions (model-invoked)
  qa-flutter-bootstrap/         ← autonomous lifecycle (backend + device)
  qa-flutter-unit-generator/    ← unit/widget/state generator
  qa-flutter-manual-runner/     ← flutter_drive integration runner
  qa-flutter-android-runner/    ← Appium integration runner
  qa-flutter-android-tester/    ← single-feature Appium tester (sub-agent)
  qa-flutter-web-runner/        ← Flutter web stability runner
  qa-flutter-release-gate/      ← GO/NO-GO production gate
docs/
  android-stacks.md             ← flutter_drive vs Appium decision matrix
  roadmap-integration.md        ← gap status G1-G7
  references/                   ← bootstrap contracts, etc.
```

## Adding a new platform runner

1. Create `skills/qa-flutter-<platform>-runner/SKILL.md` with frontmatter:
   ```yaml
   ---
   name: qa-flutter-<platform>-runner
   description: Use when running QA tests against a Flutter <Platform> app. [trigger conditions]
   argument-hint: "<test objective>"
   ---
   ```
2. Follow the step structure of `qa-flutter-manual-runner` (for driver-based) or `qa-flutter-android-runner` (for external orchestrator) as reference.
3. Register the new runner in `agents/qa-stability-agent.md` — add a branch to the platform routing logic.
4. Update `README.md` skills table and `CHANGELOG.md`.
5. If the runner needs bootstrap, teach `qa-flutter-bootstrap` about it via a new section; do **not** duplicate lifecycle code per-runner.

## Adding a new skill that participates in the release gate

1. Make sure its per-feature report follows the [Diagnóstico schema](skills/qa-flutter-manual-runner/SKILL.md#section-e--individual-report) (error principal + stack trace + log relevante).
2. If it emits findings that should feed correlation rules, add a rule to `skills/qa-flutter-release-gate/references/correlation-rules.md`.
3. Document severity mapping in `skills/qa-flutter-release-gate/references/severity-rules.md`.

## Conventions

- **No emojis** in skill bodies or agent steps (project convention; OK in CHANGELOG and docs)
- **English** for technical identifiers and documentation; Spanish acceptable in report templates if the downstream audience is Spanish-speaking (current default)
- **No hardcoded absolute paths** — skills must resolve paths at runtime (cwd → walk up to `pubspec.yaml` → ask user)
- **Frontmatter `description`** must start with "Use when..." and describe trigger conditions, not the workflow
- **Reports concise, not verbose** — stack traces ≤ 10 frames, log tail ≤ 30 lines, JSON excerpts ≤ 5 lines
- **Composition rule** — if a skill can invoke another lifecycle-owning skill, the outermost caller owns lifecycle; nested callers must detect existing marker and skip

## PR flow

1. Fork the repository
2. Create a branch: `feat/<runner-or-skill-name>` or `fix/<issue-description>`
3. Make your changes
4. Verify no hardcoded paths:
   ```bash
   grep -rEn "[A-Z]:[/\\\\]|/home/[a-zA-Z]" skills/ agents/
   ```
5. Run the CI checks locally if possible (see `.github/workflows/validate-plugin.yml`)
6. Open a PR describing which skill or agent you changed and why; link to the relevant `docs/roadmap-integration.md` gap if applicable

## Reporting bugs

Use the [bug report template](.github/ISSUE_TEMPLATE/bug_report.md). Include:

- Platform (Android flutter_drive / Android Appium / Web)
- Plugin version from `plugin list` output
- Your `qa-agent.yaml` with sensitive values redacted
- The exact error or unexpected behavior
- Relevant report file from `qa-reports/` or `test/docs/QA_REPORTS/`
