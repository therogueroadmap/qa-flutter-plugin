---
name: qa-flutter-release-gate
description: Use as a go/no-go production gate for a Flutter release — runs the full QA stack (via qa-stability-agent), classifies every finding by production impact (Critical/High/Medium/Low), and emits a binary GO/NO-GO verdict against configurable thresholds. Use before promoting a build to staging or prod, as a scheduled nightly health check, or as a CI step before publishing artifacts. NOT for exploratory testing — use qa-flutter-manual-runner directly. NOT for code review — use superpowers:requesting-code-review.
argument-hint: "[--threshold=strict|normal|lenient] [--version=<tag>] [--auto]"
---

# qa-flutter-release-gate

Production-readiness gate. Wraps `qa-stability-agent`, classifies findings by impact, and emits a binary GO/NO-GO verdict with a prioritized correction list.

**Invocation examples:**
- `/qa-release-gate` — use default threshold (`normal`), tag the run with current git HEAD
- `/qa-release-gate --threshold=strict --version=v1.2.3` — strict gate before a named release
- `/qa-release-gate --auto` — unattended; for CI and scheduled runs

## Scope

This skill:
- ✅ Orchestrates a full QA pass (delegates to `qa-stability-agent`).
- ✅ Parses all resulting reports.
- ✅ Classifies findings per severity rules.
- ✅ Applies the active threshold profile.
- ✅ Emits `release-gate-{version}-{timestamp}.md` with GO/NO-GO and correction list.
- ✅ Returns a non-zero exit signal when NO-GO (for CI integration).

This skill does NOT:
- Run tests itself — delegates to the stability agent.
- Modify code or config — read-only gate.
- Open PRs, merge, or deploy — gate only.
- Send notifications — emits the report; notification is a hook concern.

## When to use

- Before promoting a build from staging to production.
- As a nightly scheduled health check (via `anthropic-skills:schedule`).
- As a CI step after build + before artifact publish.
- Before tagging a release.

## When NOT to use

- For exploratory or dev-loop testing — use `/qa-run` or `/qa-unit` directly.
- When implementation is still partial — the gate will noisy-fail and the report is not actionable.
- For code review — use `superpowers:requesting-code-review`.
- Without `qa-plugin-config/qa-agent.yaml` — the underlying agent needs it; this skill does not generate one.

## Severity classification

All rules live in [references/severity-rules.md](references/severity-rules.md). Load that file when classifying.

Summary buckets: 🔴 Critical, 🟠 High, 🟡 Medium, 🔵 Low. Threshold profiles: `strict`, `normal` (default), `lenient`.

---

## Step -1 — Autonomous bootstrap (mandatory, runs before everything)

Invoke the `qa-flutter-bootstrap` skill via the Skill tool with args: `--up --caller=release-gate`.

Parse stdout:
- `QA_BOOTSTRAP_STATUS=UP` → continue.
- `QA_BOOTSTRAP_STATUS=ALREADY_UP_BY={caller}` → continue (someone else owns; we're a guest).
- Anything else or non-zero exit → **emit immediate NO-GO**:
  - Skip Steps 1–7 entirely.
  - Write a minimal `.md` report with a single Critical finding "bootstrap failed: {error}".
  - Write a minimal `.json` artifact per schema v1.0 with `infrastructure.bootstrap.teardown_status = "SKIPPED_NO_UP"` and the single Critical finding.
  - Run Step 9 (teardown) — it will no-op via `NO_MARKER` since `--up` did not complete.
  - Exit 1.

Record the outcome (`UP` or `ALREADY_UP_BY=<caller>`) for use in Step 7's "Infrastructure lifecycle" line.

## Step 1 — Parse arguments

- `THRESHOLD` = value of `--threshold=...`; default = `release_gate.threshold` from yaml or `normal`. Valid: `strict`, `normal`, `lenient`.
- `VERSION` = value of `--version=...`; default = short git HEAD via `git rev-parse --short HEAD`, or `"untagged"` if outside a git repo.
- `AUTO_MODE` = true if `--auto` present.

If `THRESHOLD` invalid → abort:
```
⛔ --threshold inválido. Valores: strict, normal, lenient.
```

## Step 2 — Read `qa-plugin-config/qa-agent.yaml` and gate config

Resolve `qa-plugin-config/qa-agent.yaml` (cwd → walk up → ask user). Extract:

| Field | yaml path | Default |
|---|---|---|
| `CRITICAL_FEATURES` | `release_gate.critical_features` | `["login", "auth", "payment", "checkout"]` |
| `COVERAGE_FLOOR` | `release_gate.coverage_floor` | `70` |
| `WAIVERS` | `release_gate.waivers` | `[]` |
| `YAML_THRESHOLD` | `release_gate.threshold` | (none → use CLI default `normal`) |
| `REPORTS_DIR` | `reports.output_dir` or `agent.reports_output_dir` | per stack |

`--threshold` CLI flag **overrides** `release_gate.threshold` in yaml.

Filter `WAIVERS` — drop any whose `expires` date is in the past. Log dropped waivers:
```
⚠ Waiver expirado ignorado: {finding_id} (expiró {expires})
```

## Step 3 — Run the full QA stack

Invoke `qa-stability-agent` via the Skill tool. Pass no special args — the agent reads the same yaml.

Capture all report paths the agent returns:
- `main_report` — from the E2E runner
- `unit_reports[]` — from unit-generator runs (if `post_run.include_unit == true`)

If the agent itself aborts (e.g., yaml missing, no device), **that is a pre-run failure**, not a gate result. Record it and emit a NO-GO report with all findings under Critical.

## Step 4 — Parse reports and extract findings

For each report in `{main_report}` + `{unit_reports[]}`:

### 4.1 — Read the report

Use the Read tool. Parse the markdown structure.

### 4.2 — Extract findings

A "finding" is any non-PASS outcome. Build a list of `FINDING` records:

```
{
  id: "<feature>-<layer>-<step_or_test_id>",
  source_report: "<path>",
  feature: "<feature slug>",
  layer: "e2e" | "unit" | "widget" | "state" | "analyze" | "build",
  result: "FAIL" | "PARCIAL" | "COMPILE_ERROR" | "TIMEOUT" | "ERROR" | "COVERAGE_LOW",
  description: "<human text from the report>",
  details: "<any error message, step index, etc.>"
}
```

Finding ids must be stable across runs so waivers can match.

### 4.3 — Pull coverage numbers

From unit reports, extract per-layer `LINE_COVERAGE` and the aggregate. If any feature in `CRITICAL_FEATURES` has `LINE_COVERAGE < COVERAGE_FLOOR`, emit a synthetic `COVERAGE_LOW` finding.

## Step 5 — Classify findings by severity

Load [references/severity-rules.md](references/severity-rules.md). For each `FINDING`, walk the rules in order; the first matching rule determines severity.

Apply waivers **after** classification: if a finding's `id` matches an active waiver, downgrade `High → Medium` and record the waiver in the report. Critical is never downgradable.

Produce bucket counts:
```
CRITICAL = N
HIGH = N (of which {waived_count} waived from higher)
MEDIUM = N
LOW = N
```

## Step 5.5 — Root-cause correlation

Load [references/correlation-rules.md](references/correlation-rules.md). For each rule in the order listed:

1. Evaluate the rule's match predicate against the current `FINDINGS` list.
2. On match, produce a `CorrelatedFinding` record:
   ```
   {
     id: <rule name, e.g. "BACKEND_UNSTABLE">
     root_cause: <rule's text>
     severity: max(severity of each matched finding)
     evidence: [<finding id>, ...]
     action: <rule's text>
   }
   ```
3. Append to list `CORRELATIONS`.
4. For each evidence finding, append `<rule name>` to its `correlated_with` field (list, may contain 0..N rule names).

Correlation NEVER modifies a finding's classified severity — it only groups + labels.

After all rules evaluated → `CORRELATIONS` is the full list. Pass it to Step 6 (threshold check, which reads only individual finding severities, not correlations) and Step 7 (report emission).

## Step 6 — Apply threshold → GO/NO-GO

| Profile | GO if |
|---|---|
| `strict` | `CRITICAL == 0` AND `HIGH == 0` AND `MEDIUM ≤ 2` |
| `normal` | `CRITICAL == 0` AND `HIGH == 0` |
| `lenient` | `CRITICAL == 0` |

Set `VERDICT = "GO"` or `"NO-GO"`.

Additionally compute `EXIT_CODE`:
- `GO` → 0
- `NO-GO` → 1

(The CLI caller — CI step — reads the exit code to fail the pipeline.)

## Step 7 — Emit the release-gate report

`TIMESTAMP` = current datetime as `YYYY-MM-DDTHH-MM`.

### 7.1 — Markdown report

Write `{REPORTS_DIR}/release-gate-{VERSION}-{TIMESTAMP}.md` with the structure below.

```markdown
# Release Gate — {VERSION}
**Fecha:** {TIMESTAMP} | **Threshold:** {THRESHOLD} | **Veredicto:** {🟢 GO | 🔴 NO-GO}

## Resumen

| Severidad | Count | Bloquea GO |
|-----------|-------|------------|
| 🔴 Critical | {N} | Sí (todos los profiles) |
| 🟠 High | {N} | {Sí en strict/normal, No en lenient} |
| 🟡 Medium | {N} | {Sí en strict si >2, No en normal/lenient} |
| 🔵 Low | {N} | No |

{if NO-GO:}
**Razón del NO-GO:** {specific rule triggered}
{end if}

## Análisis de causa raíz

{if CORRELATIONS empty:} Sin correlaciones detectadas. Ver hallazgos individuales abajo.
{else: for each correlation in CORRELATIONS:}
### {correlation.root_cause}
- **Severidad:** {correlation.severity}
- **Acción:** {correlation.action}
- **Evidencia:** {comma-separated finding IDs}
{end for}

## Hallazgos por severidad

### 🔴 Critical ({N})
{for each Critical finding:}
- **[{id}]** ({layer}/{feature}) — {description}
  {if finding.correlated_with not empty:} ↑ parte de: {comma-separated rule names}
  - Reporte: [{source_report}]({source_report})
  - Detalle: {details}
{end for}
{if none: "Sin hallazgos críticos."}

### 🟠 High ({N}) — same format as Critical with correlated_with marker

### 🟡 Medium ({N}) — same format, truncate to first 10 with "... y N más" if overflow

### 🔵 Low ({N}) — summary count only

## Waivers aplicados

| Finding ID | Motivo | Expira |
|------------|--------|--------|
{for each active waiver that matched a finding}
{if none: "Sin waivers activos."}

## Acciones requeridas

{numbered list, ordered by severity, one per Critical + High finding:}
1. **[Critical]** {action}
...

{if VERDICT == GO and no Criticals/Highs:} **Sin acciones bloqueantes.** Medium/Low opcionales para siguiente sprint.

## Próximos pasos

{if GO:}
- ✅ La build {VERSION} cumple el gate `{THRESHOLD}`.
- Si se promueve a prod: registrar este reporte como evidencia del gate.
- Medium/Low: trackear en el backlog.

{if NO-GO:}
- ❌ La build {VERSION} NO cumple el gate `{THRESHOLD}`.
- Bloquear la promoción hasta resolver las acciones listadas.
- Re-correr `/qa-release-gate --version={VERSION}` tras aplicar las correcciones.

## Reportes subyacentes
- Main: [{main_report}]({main_report})
{for each unit_report:} - Unit: [{unit_report}]({unit_report})

## Infrastructure lifecycle

{one line:}
{if Step -1 == UP:} Bootstrap: OK | Teardown: {DOWN | TEARDOWN_FAILED — ver marker file}
{if Step -1 == ALREADY_UP_BY=X:} Bootstrap: deferred to {X} | Teardown: deferred to {X}
{if Step -1 == failed:} Bootstrap: FAILED — ver Step -1 output | Teardown: {n/a | NO_MARKER}
```

### 7.2 — JSON artifact

Write `{REPORTS_DIR}/release-gate-{VERSION}-{TIMESTAMP}.json` with this schema:

```json
{
  "schema_version": "1.0",
  "run": {
    "version": "{VERSION}",
    "threshold": "{THRESHOLD}",
    "started_at": "<run start ISO8601>",
    "finished_at": "<TIMESTAMP as ISO8601>",
    "duration_seconds": <int>,
    "caller": "release-gate",
    "verdict": "GO|NO-GO",
    "exit_code": <0 or 1>
  },
  "infrastructure": {
    "bootstrap": {
      "backend_started": <bool from marker.backend.we_started_it or null>,
      "device_started": <bool from marker.device.we_started_it or null>,
      "teardown_status": "OK|TEARDOWN_FAILED|SKIPPED_NO_UP|DEFERRED_TO_{caller}"
    }
  },
  "findings": [
    {
      "id": "<stable id>",
      "feature": "<feature slug>",
      "layer": "e2e|unit|widget|state|analyze|build",
      "result": "FAIL|PARCIAL|COMPILE_ERROR|TIMEOUT|ERROR|COVERAGE_LOW",
      "severity": "Critical|High|Medium|Low",
      "source_report": "<relative path>",
      "error_excerpt": "<3-5 lines>",
      "correlated_with": ["RULE_NAME", ...]
    }
  ],
  "correlations": [
    {
      "id": "<rule name>",
      "root_cause": "<text>",
      "severity": "Critical|High|Medium|Low",
      "evidence": ["<finding id>", ...],
      "action": "<text>"
    }
  ],
  "waivers_applied": [ { "id": "<finding id>", "reason": "<text>", "expires": "<date>" } ],
  "coverage": {
    "aggregate_line": <int|null>,
    "per_feature": { "<feature>": <int> }
  },
  "thresholds": {
    "active_profile": "strict|normal|lenient",
    "triggered_rule": "<text if NO-GO, empty if GO>"
  }
}
```

Omit optional fields when empty rather than emitting `null` (except `aggregate_line` which is meaningful when null = "no unit coverage in this run").

### 7.3 — stdout

Print to stdout in AUTO_MODE (additions to the existing Step 8 output):
```
RELEASE_GATE_VERDICT={VERDICT}
RELEASE_GATE_REPORT_MD={md path}
RELEASE_GATE_REPORT_JSON={json path}
RELEASE_GATE_CRITICAL={N}
RELEASE_GATE_HIGH={N}
RELEASE_GATE_CORRELATIONS={count}
RELEASE_GATE_EXIT={EXIT_CODE}
```

## Step 8 — Output

Print to stdout in `AUTO_MODE`:
```
RELEASE_GATE_VERDICT={VERDICT}
RELEASE_GATE_REPORT={path}
RELEASE_GATE_CRITICAL={N}
RELEASE_GATE_HIGH={N}
RELEASE_GATE_EXIT={EXIT_CODE}
```

In interactive mode, also print a summary:
```
{🟢 GO | 🔴 NO-GO} — {VERSION} — threshold: {THRESHOLD}
Critical: {N} | High: {N} | Medium: {N} | Low: {N}
Reporte: {path}
```

End the run with exit code `EXIT_CODE` (0 for GO, 1 for NO-GO). CI callers fail the pipeline on non-zero.

## Step 9 — Autonomous teardown (mandatory, always runs)

**Runs even after NO-GO, early abort, or Step -1 failure.**

Invoke `qa-flutter-bootstrap` skill via Skill tool with args: `--down --caller=release-gate`.

Parse stdout — do NOT change the verdict or exit code based on teardown result:
- `QA_BOOTSTRAP_STATUS=DOWN` → OK.
- `QA_BOOTSTRAP_STATUS=NO_MARKER` → OK (nothing to tear down; consistent with Step -1 failure path).
- `QA_BOOTSTRAP_STATUS=SKIPPED_NOT_OWNER=<owner>` → OK (guest run, owner handles).
- `QA_BOOTSTRAP_STATUS=TEARDOWN_FAILED` → log warning prominently. Marker file remains with `teardown_failed: true`.

Update the "Infrastructure lifecycle" line in the already-written `.md` report (in-place edit) and the `infrastructure.bootstrap.teardown_status` in the already-written `.json` file.

End of skill execution.

---

## Scheduling (autonomous routine)

This skill is designed to be invoked autonomously. Two supported mechanisms:

### Option A — `anthropic-skills:schedule`

Creates a cron task that fires `/qa-release-gate --auto` on an interval.

Example — nightly run at 03:00 local time:
```
user: schedule '/qa-release-gate --auto --threshold=normal' every day at 03:00
```

The `schedule` skill creates the task. Report lands in `{REPORTS_DIR}/` the next morning.

### Option B — external CI (GitHub Actions, Task Scheduler, cron)

```bash
claude -p "/qa-release-gate --auto --threshold=strict --version=${VERSION}" \
  --output-format json
```

Read `RELEASE_GATE_EXIT` from output; fail the CI job if non-zero.

### Option C — pre-release manual

```
user: correr release gate para v1.2.3
```

Or directly: `/qa-release-gate --version=v1.2.3`.

---

## Integration with plan execution

At the end of a plan that implements a release, add a final task:

```markdown
- [ ] Run release gate
  - Invoke `/qa-release-gate --threshold=normal`
  - Persist the report path in the plan's Completion section
```

On NO-GO, treat the final task as failed and surface the report to the user before the plan is considered complete.

---

## Common mistakes

| Mistake | Fix |
|---|---|
| Using `lenient` as default threshold | Defeats the gate. Reserve for hot-fix branches with explicit justification. |
| Waivers without `expires` date | Reject silently — every waiver must expire. Re-evaluate when the date hits. |
| Running the gate mid-implementation | Report will be noisy; findings are not actionable. Gate is for post-implementation / pre-release. |
| Ignoring Medium for too long | Medium doesn't block but accumulates risk. Track them in a ticket system. |
| Hardcoding new severity rules in this skill | Extend via `release_gate.critical_features` in yaml. The rules file is the contract. |
| Re-running the gate without resolving a flagged Critical | The second run will NO-GO again. Fix root cause, not the gate. |

## Limitations

- **No automatic notification** — the skill emits the report; integrating with Slack/email is a hook concern (use Claude Code hooks or a CI step).
- **No waiver audit trail** — waivers in yaml don't have an approval record. Enforce via PR review.
- **Severity rules are not ML-driven** — static rules encoded in the reference file. Edit that file or the yaml config to adjust.
- **No historical trend** — each run is independent. Track deltas externally (report diff tool, dashboard).
- **Relies on `qa-stability-agent`** — if the agent misbehaves, this gate is as good as the agent.
