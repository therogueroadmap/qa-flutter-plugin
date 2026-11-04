---
name: qa-flutter-android-runner
description: Use when running full QA suite on a Flutter Android app that uses the Appium stack (qa-agent.yaml with appium.device_name, backend Python scripts). Requires the qa-agent/ companion directory with bootstrap.py and teardown.py. Optionally consumes a pre-generated plan via --plan=<path> instead of parsing a free-form objective. NOT for flutter_drive-based testing — use qa-flutter-manual-runner. NOT for unit/widget tests — use qa-flutter-unit-generator. NOT for Flutter web — use qa-flutter-web-runner.
argument-hint: "<test objective> [--plan=<path>]"
---

# qa-flutter-android-runner

Orchestrates a full QA test run against a Flutter Android app.

Invoked as: `/qa-flutter-android-runner "<test objective>"`

Example: `/qa-flutter-android-runner "test el flujo de registro y login de usuario"`

## Configuration

Resolve `CONFIG_PATH` at runtime — **never hardcode**:

1. Check `./qa-plugin-config/qa-agent.yaml` in the current working directory.
2. If absent, walk up from cwd looking for a `pubspec.yaml` sibling, then look for `qa-plugin-config/qa-agent.yaml` there.
3. If still not found, ask the user: `"¿Cuál es la ruta absoluta al archivo qa-plugin-config/qa-agent.yaml?"`.

`QA_AGENT_DIR` = directory containing the resolved `CONFIG_PATH` (= `{PROJECT_ROOT}/qa-plugin-config`).
`PROJECT_ROOT` = parent of `QA_AGENT_DIR` (= directory containing `pubspec.yaml`).

Under `QA_AGENT_DIR` there must exist `scripts/bootstrap.py` and `scripts/teardown.py` for the Appium stack (at `qa-plugin-config/scripts/`). If missing → abort with instructions to install the qa-agent companion scripts inside `qa-plugin-config/`.

## Execution — follow every step in order

### Step 1: Read configuration

Use the Read tool to read CONFIG_PATH. Extract:
- `agent.max_features_per_run`
- `agent.agent_max_execution_seconds`
- `agent.reports_output_dir`
- `project.backend.path`
- `project.flutter.path`

**Placeholder check:** If `project.backend.path` contains "REEMPLAZAR" or is empty, ask the user:
> "¿Cuál es la ruta absoluta al proyecto backend (e.g. my-backend-api)?"
Wait for the response and store it as `BACKEND_PATH`.

If `project.flutter.path` contains "REEMPLAZAR" or is empty, ask the user:
> "¿Cuál es la ruta absoluta al proyecto Flutter?"
Wait for the response and store it as `FLUTTER_PATH`.

These values are used in memory for this run only — the yaml file is NOT modified.

### Step 2: Bootstrap — start services

Use TaskCreate (or equivalent) to create a task "Bootstrap services" and mark it in_progress.

Invoke the `qa-flutter-bootstrap` skill via the Skill tool with args: `--up --caller=android-runner`.

Parse stdout:
- `QA_BOOTSTRAP_STATUS=UP` → bootstrap successful. Read `QA_BOOTSTRAP_MARKER` path; parse the marker YAML; extract `device.id` as `APPIUM_DEVICE_ID` for downstream use. Mark task completed. Continue to Step 3.
- `QA_BOOTSTRAP_STATUS=ALREADY_UP_BY={caller}` → another skill owns lifecycle. Read the existing marker; extract `device.id` same way. Mark task completed. Continue to Step 3.
- Anything else → abort:
  - Print the stderr/error from bootstrap.
  - Print: "Bootstrap failed. Aborting test run. Check services and retry."
  - Mark task completed with FAIL label.
  - STOP. Do not dispatch sub-agents. Do not run Step 6 either (`--up` did not complete).

**Note:** `qa-flutter-bootstrap` internally detects `project.android_stack == "appium"` and delegates to `scripts/bootstrap.py` in the companion qa-agent directory. The contract that `bootstrap.py` must honor is documented in [docs/references/appium-bootstrap-contract.md](../../docs/references/appium-bootstrap-contract.md).

**Deprecated path:** Invoking `python scripts/bootstrap.py qa-plugin-config/qa-agent.yaml` directly is still functional during the current minor release as a fallback — if the user's `bootstrap.py` does not yet support the `--marker` flag, `qa-flutter-bootstrap` will synthesize a marker from the legacy `bootstrap-status.json`. This fallback will be removed in the next minor release.

### Step 3: Parse test objective into feature list

**Plan-driven path (preferred):** If the invocation contains `--plan=<path>`, OR `qa-plugin-config/qa-agent.yaml` has `planning.test_plan_dir` set and a plan exists for the requested feature, load the plan instead of parsing the free-form objective:

1. Read the plan file. Verify YAML frontmatter `feature: {expected}`.
2. Use **Section 2 — Pantallas a cubrir** to derive the feature list (one entry per row).
3. Use **Section 4 — Flujos** to enrich the prompt sent to each `qa-flutter-android-tester` sub-agent in Step 4 — pass the flow steps as the test scenario instead of letting the sub-agent infer them.
4. Skip the free-form parsing rules below.

**Free-form path (fallback, no plan):** Given the argument passed to `/qa-flutter-android-runner`, identify the individual features to test.

Rules:
- Extract concrete feature names as lowercase slugs (e.g. "signup", "login", "password-recovery")
- If more features are identified than `max_features_per_run`, take the first N and note the rest as "not executed" in the summary
- If the objective is ambiguous, test the most specific features you can infer

Print the feature list to the user before dispatching sub-agents. If a plan was loaded, also print:
```
Usando plan: {PLAN_PATH} ({N} pantallas, {N} flujos)
```

### Step 4: Dispatch sub-agents — one per feature, strictly sequential

For EACH feature in the list (do NOT dispatch more than one at a time):

1. Use TaskCreate to create a task "Test feature: {feature_name}" and mark it in_progress.

2. Record the start time.

3. Use the Agent tool to dispatch the qa-flutter-android-tester skill with this exact prompt:
```
CONFIG_PATH={CONFIG_PATH}
FEATURE_NAME={feature_name}
QA_AGENT_DIR={QA_AGENT_DIR}
BACKEND_PATH={BACKEND_PATH}
FLUTTER_PATH={FLUTTER_PATH}

Follow the qa-flutter-android-tester skill instructions exactly.
```

4. If the sub-agent does not complete within `agent_max_execution_seconds` seconds:
   - Use TaskUpdate to mark the task as completed with a TIMEOUT note.
   - Create a timeout placeholder report at `{QA_AGENT_DIR}/{reports_output_dir}/{timestamp}-{feature_name}.md`:
     ```
     # QA Report — Feature: {feature_name}
     **Resultado:** ❌ TIMEOUT — agent exceeded {agent_max_execution_seconds}s
     ```
   - Mark the task completed with label "TIMEOUT".

5. On success: the sub-agent returns the path to the report file. Read that file and store its content.

6. Mark the task completed.

7. WAIT for this sub-agent to fully complete before starting the next one.

### Step 5: Generate summary report

Current datetime formatted as YYYY-MM-DDTHH-MM = `{timestamp}`.

Write `{QA_AGENT_DIR}/{reports_output_dir}/{timestamp}-summary.md`:

```markdown
# QA Summary — {timestamp}
**Features solicitadas:** {total} | **Ejecutadas:** {executed} | **No ejecutadas:** {skipped}

| Feature | Resultado | Duración | Errores principales |
|---------|-----------|----------|---------------------|
{one row per executed feature, values from each feature report}

## Features no ejecutadas (límite max_features_per_run)
{list, or "Ninguna." if all features were tested}

## Errores para planificación
{numbered list: one actionable item per FAIL or TIMEOUT feature}
{if all passed: "Sin errores. Todos los tests pasaron."}
```

### Step 6: Teardown — stop services

Invoke the `qa-flutter-bootstrap` skill via the Skill tool with args: `--down --caller=android-runner`.

Parse stdout:
- `QA_BOOTSTRAP_STATUS=DOWN` → OK.
- `QA_BOOTSTRAP_STATUS=SKIPPED_NOT_OWNER=<owner>` → OK (composition — outer caller handles teardown).
- `QA_BOOTSTRAP_STATUS=NO_MARKER` → OK (Step 2 did not complete — nothing to tear down).
- `QA_BOOTSTRAP_STATUS=TEARDOWN_FAILED` → log warning. The summary report's footer will note this.

Do not change the overall run outcome based on teardown status — this step is cleanup, not validation.

### Step 7: Final output

Print:
```
Test run complete.
Reports directory: {QA_AGENT_DIR}/{reports_output_dir}/
Summary: {QA_AGENT_DIR}/{reports_output_dir}/{timestamp}-summary.md
```

---

## When NOT to use

- The project uses `flutter drive` / `integration_test` without Appium — use `qa-flutter-manual-runner`.
- The project has no companion `qa-agent/` directory with `scripts/bootstrap.py`, `scripts/teardown.py`, `scripts/appium_runner.py` — those scripts are prerequisite. See [docs/references/appium-bootstrap-contract.md](../../docs/references/appium-bootstrap-contract.md) for the contract those scripts must honor.
- You only need unit or widget tests — use `qa-flutter-unit-generator`.
- The feature isn't implemented yet — `bootstrap.py` will launch but tests will find no UI.

## Common mistakes

| Mistake | Fix |
|---|---|
| Dispatching sub-agents in parallel | Spec is explicit: strictly sequential. Parallel breaks the Appium session. |
| Skipping the bootstrap-status.json check | Half-started services cause confusing errors down the line — always check. |
| Passing hardcoded `CONFIG_PATH` | Resolve via cwd + walk-up, not a static constant. |
| Forgetting to mark the task `completed` on TIMEOUT | The runner's summary reads task states; a lingering `in_progress` corrupts the report. |
| Running without `project.backend.path` resolved | Sub-agents need git hash from the backend repo for the report header. |

## Decision — when to pick this vs `qa-flutter-manual-runner`

See [docs/android-stacks.md](../../docs/android-stacks.md) for the full decision matrix. Short version: pick this when you need gestos nativos, multi-app flows, or are running against a pre-compiled APK.
