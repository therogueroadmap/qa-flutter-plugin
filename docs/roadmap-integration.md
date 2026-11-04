# Roadmap — autonomous integration

Living document that tracks what's needed to make the qa-flutter plugin **truly autonomous** — i.e., runnable unattended by a scheduler or CI, from zero-state (no backend running, no device ready) to a signed release-gate verdict.

## Current state — what works autonomously today

| Capability | Stack | Status |
|---|---|---|
| E2E regression over existing tests | flutter_drive (`manual-runner`) | ✅ if backend + device are already up |
| E2E regression via Appium | Appium (`android-runner`) | ✅ Appium stack has `bootstrap.py` / `teardown.py` — handles backend + Appium server + Flutter build |
| Unit/widget generation and execution | `unit-generator` | ✅ fully autonomous, no infra deps |
| Web stability checklist | `web-runner` | ✅ autonomous (local `flutter build web`) |
| Platform/stack routing | `qa-stability-agent` | ✅ reads yaml and delegates |
| Severity classification + GO/NO-GO | `release-gate` | ✅ reads reports and classifies |
| CI trigger via `claude -p` | All | ✅ with exit codes |

## Gaps blocking full autonomy

The gaps below prevent scheduled/unattended runs from completing without human intervention.

---

### G1 — Backend bootstrap for the `flutter_drive` stack 🔴 Critical

**Problem.** `qa-flutter-manual-runner` does a pre-flight health check on the backend (`GET /health` or `POST /login`). If the backend is down, it aborts. In an autonomous run (nightly cron, CI), nobody has started the backend, so the gate always fails.

**Appium stack doesn't have this gap** — its `bootstrap.py` starts the backend as part of the setup. The flutter_drive stack was designed assuming the user manually starts the backend before invoking the runner.

**Proposal — add `autonomous.backend` to `qa-agent.yaml`:**

```yaml
autonomous:
  backend:
    start_cmd: "./gradlew bootRun --args='--spring.profiles.active=test'"
    start_cwd: "../my-backend-api"
    health_url: "http://localhost:8080/api/health"
    ready_timeout_seconds: 120
    stop_cmd: "pkill -f 'gradlew bootRun'"     # optional
    stop_timeout_seconds: 30
```

**Where to implement:** extend `qa-stability-agent` body with a new Step 0 "Autonomous bootstrap" that runs before routing. Logic:

1. If invoked in auto mode (`--auto` or scheduled) AND `autonomous.backend` is present:
   - Check `health_url` — if already 200, skip (backend already up).
   - Otherwise run `start_cmd` in background from `start_cwd`.
   - Poll `health_url` every 2s up to `ready_timeout_seconds`.
   - On timeout → abort with NO-GO and reason "backend did not boot".
2. Register `stop_cmd` to run on agent exit (success or failure).
3. Proceed with routing.

**Design decisions to make:**
- Who owns the backend lifecycle? If the backend is a separate repo, the gate shouldn't assume it's a Flutter sibling. `start_cwd` makes it explicit.
- Should `stop_cmd` run if the backend was already up when we started? Probably not — don't stop what we didn't start. Track a `WE_STARTED_IT` flag.
- Timeouts — Spring Boot starts in ~30s, Rails in ~10s, heavy stacks in 60-90s. Default `ready_timeout_seconds: 120` covers most cases.

**Security note:** `start_cmd` runs arbitrary shell. The yaml is user-authored and lives in the repo — trust model is already the same as any dev script. Still, document in the skill that this runs as the user and should never invoke anything requiring secrets not already in the environment.

**Status:** ✅ Implemented 2026-04-23 via new skill `qa-flutter-bootstrap` (Approach 2). Backend lifecycle configured under `autonomous.backend.*` in `qa-agent.yaml`. Design note: single `start_cmd` (deferred multi-service per open question #1); `stop_cmd` optional with `kill {pid}` fallback.

---

### G2 — Device bootstrap for the `flutter_drive` stack 🟠 High

**Problem.** `manual-runner` Section 3.3 already handles emulator boot ("if no device → launch first AVD"). But the device name is inferred, which breaks if multiple AVDs exist. And for CI where no AVD may exist at all, there's no recovery.

**Proposal — add `autonomous.device` to `qa-agent.yaml`:**

```yaml
autonomous:
  device:
    boot_avd: "Pixel_5_API_33"               # explicit AVD to boot
    boot_timeout_seconds: 90
    create_if_missing: false                 # if true, create the AVD before booting (heavy)
    create_spec:
      system_image: "system-images;android-33;google_apis;x86_64"
      device_profile: "pixel_5"
```

**Where to implement:** `manual-runner` Section 3.3 already has the skeleton — extend to:
- Read `autonomous.device.boot_avd` to pick the specific AVD (not "first alphabetical").
- If `create_if_missing` and the AVD doesn't exist, create it via `avdmanager` + `sdkmanager`. Emit a warning that first run takes longer.
- Respect `boot_timeout_seconds` (currently hardcoded to 90).

**Status:** ✅ Implemented 2026-04-23. `autonomous.device.boot_avd` + `boot_timeout_seconds` exposed in `qa-agent.yaml`. Manual-runner Section 3.3 honors the explicit AVD when configured, falls back to legacy "first alphabetical" with warning when not. `create_if_missing` deferred (spec decision 6 = A).

---

### G3 — Notifications on NO-GO 🟠 High

**Problem.** Release-gate emits a markdown report to disk. In an autonomous nightly run, nobody reads it unless actively looking. A NO-GO needs to page the team.

**Options — from simplest to most integrated:**

1. **Webhook in the skill.** `release_gate.notify_on_nogo` already reserved in yaml. Implement as a POST to a URL with the verdict + report link.
2. **Claude Code hook.** Use a `Stop` or `PostToolUse` hook in `~/.claude/settings.json` to detect `RELEASE_GATE_VERDICT=NO-GO` in output and fire a shell command (e.g., `curl slack webhook`). Zero skill changes, but requires per-user hook setup.
3. **External CI notification.** The CI pipeline that invokes `claude -p "/qa-release-gate --auto"` reads the exit code and fires its own notification (GitHub Actions has built-in Slack integrations). Cleanest separation of concerns.

**Recommendation:** start with (3) — CI already has mature notification systems. Add (1) as a fallback for users running via `anthropic-skills:schedule` without an external CI.

**Status:** yaml field reserved, no implementation. Document clearly that notification is out-of-scope today.

---

### G4 — Historical trend / delta view 🟡 Medium

**Problem.** Each gate run is independent. Users can't answer: "are we trending worse? which features regressed from last release?".

**Proposal — `release-gate` emits a machine-readable artifact alongside the markdown:**

```
{reports_output_dir}/
  release-gate-v1.2.3-2026-04-23T03-00.md        ← human report
  release-gate-v1.2.3-2026-04-23T03-00.json      ← machine-readable summary
```

The JSON contains: version, timestamp, verdict, counts per severity, full findings list with stable IDs.

A second skill or external tool can diff two JSONs to produce a trend report:
- Findings that newly appeared (regressions)
- Findings that disappeared (fixes)
- Severity escalations/de-escalations

**Minimal v1:** just emit the JSON. Defer the diff tool until the data exists.

**Status:** ✅ Implemented 2026-04-23. Schema v1.0 emitted alongside the `.md` at `release-gate-{VERSION}-{TIMESTAMP}.json`. Includes findings with stable IDs, correlations (G4 + root-cause), infrastructure lifecycle summary, coverage, thresholds. Diff tooling deferred (YAGNI — data exists now for future).

---

### G5 — Waiver approval workflow 🟡 Medium

**Problem.** Waivers in `release_gate.waivers[]` have no approval record. Anyone with repo access can silently downgrade a High finding. For regulated contexts, this is insufficient.

**Proposal — separate the waiver from the yaml:**

```yaml
release_gate:
  waivers_file: ".qa-waivers.yaml"     # tracked in git, requires PR review to change
```

The `waivers_file` is a separate artifact whose edits trigger code review via normal PR flow. The gate reads it identically to the inline array.

**Alternative — signed waivers.** Each waiver carries a commit SHA where it was added and the author. Gate report lists them. Audit is "who added this waiver, when".

**Status:** not implemented. Low priority unless compliance requires it.

---

### G6 — Scheduled-run idempotency 🟡 Medium

**Problem.** If a scheduled run at 03:00 takes 45 minutes and the next scheduler fires at 03:30, two runs can overlap. Two runs both running `pm clear` on the same emulator is chaos.

**Proposal — pidfile-style lock:**

At the start of `qa-stability-agent` or `release-gate`:
```
lock_file = {reports_output_dir}/.qa-lock
if lock_file exists and its pid is alive → abort with "Another run in progress since {timestamp}"
else → write current pid + timestamp, register cleanup on exit
```

**Status:** ✅ Implemented 2026-04-23 via marker file which doubles as run lock. Located at `{reports.output_dir}/.qa-bootstrap-marker`. Stale detection via `pid_ward` liveness check. Cross-machine coordination remains out of scope (open question: local lock only).

---

### G8 — Planning artifact before execution 🟠 High

**Problem.** Until v1.2 the runners did *semantic resolution* on the fly inside `qa-flutter-manual-runner` § C.1 to decide what to test. That logic only handles single-page features and gives the QA no chance to review the scope before tests are generated and executed. Multi-screen flows (checkout, onboarding, password recovery → reset) are missed entirely, preconditions surface only at runtime as cryptic failures, and there is no auditable artifact justifying the coverage decisions for compliance / release-gate reviews.

**Proposal — extract planning into its own skill that produces a markdown+YAML artifact:**

```
/qa-plan login            → qa-plans/login.md
                            (pantallas, precondiciones, flujos, criterios, riesgos)
/qa-run login --plan=qa-plans/login.md
                          → runner consumes the plan instead of guessing scope
```

Plans are committed alongside the code; reports stay in `reports.output_dir` (gitignored).

**Where to implement:** new skill `qa-flutter-test-planner` reuses Layer 1+2+3 logic from manual-runner § C.1 generalized to N screens. Runners gain a `--plan=<path>` flag that, when present, replaces their on-the-fly resolution. Stability agent gains a Step 2.5 that auto-injects the flag based on `planning.test_plan_dir`.

**Backward compatibility.** `planning:` block in yaml is optional; without it, runners behave exactly as in v1.1.

**Status:** ✅ Implemented 2026-05-04 via new skill `qa-flutter-test-planner` + `--plan=<path>` flag on all three runners + Step 2.5 in `qa-stability-agent`. Templates live under `skills/qa-flutter-test-planner/references/`. Plan format is markdown body with YAML frontmatter (schema v1).

---

### G7 — Cost control for unattended Claude runs 🔵 Low

**Problem.** `claude -p "/qa-release-gate --auto"` consumes tokens for every run. Nightly cron = 30+ runs per month. The gate itself is cheap (reads reports and classifies), but the delegated skills dispatch sub-agents.

**Proposal — track token cost per run in the JSON artifact (G4):**
```json
{
  "tokens_used": 84000,
  "duration_seconds": 2700,
  "subagents_dispatched": 7
}
```
Surface aggregates in a dashboard. If cost is a concern, move to hybrid: run the actual tests via non-LLM tooling (already the case for `flutter test` / `flutter drive` — only the classification needs Claude) and invoke Claude only for the gate's synthesis step.

**Status:** not implemented. Cost per run today is low (~$1-3); only matters at scale.

---

## Priority summary

| Gap | Priority | Effort | Unlocks |
|---|---|---|---|
| G1 — Backend bootstrap | 🔴 Critical | Medium | True autonomy for flutter_drive stack |
| G2 — Device bootstrap | 🟠 High | Low | CI runs from cold state |
| G3 — Notifications | 🟠 High | Low (via CI) | Actionable NO-GO |
| G4 — JSON artifact + trend | 🟡 Medium | Low | Historical view |
| G5 — Waiver workflow | 🟡 Medium | Low | Audit / compliance |
| G6 — Run idempotency | 🟡 Medium | Low | Safe high-frequency schedules |
| G7 — Cost tracking | 🔵 Low | Low | Scale |
| G8 — Planning artifact | 🟠 High | Medium | Multi-screen scope, auditable coverage decisions |

## Suggested order of implementation

1. **G1 (backend bootstrap)** — hard prerequisite for scheduled runs on flutter_drive. Without this, autonomy is a nominal feature.
2. **G2 (device bootstrap config)** — small extension to what exists, big reliability gain.
3. **G4 (JSON artifact)** — cheap; unblocks historical tooling later.
4. **G3 (notifications via CI)** — document the pattern; users wire it up.
5. **G6 (idempotency)** — add when first collision is reported.
6. **G5 (waiver workflow)** — add when compliance demands it.
7. **G7 (cost tracking)** — add when cost becomes a concern.

## Non-goals (explicitly out of scope)

- **Replacing `flutter test` / `flutter drive` with a pure-Claude runner.** Native tooling is faster, deterministic, and well-known. Claude's role is orchestration + synthesis, not execution of individual tests.
- **ML-driven severity classification.** Static rules in `severity-rules.md` are predictable and auditable. An ML model would be a black box for compliance.
- **Deploying the app.** The gate emits GO/NO-GO; the deployment system reads it. Never couple the two.
- **Managing backend test data.** MUTANT tagging + test-profile backend handle isolation. The gate verifies the guard is in place; it doesn't clean data.

## Open questions

1. **Should `autonomous.backend.start_cmd` support multiple backends?** Some apps talk to 2-3 services. An array of services with ordered start might be needed. Defer until the single-backend case works.
2. **Should the gate ever auto-waive flaky tests?** E.g., "if a test passed on the previous run and fails now, retry once". Tempting but masks real regressions. Vote: no, keep the gate honest; let the underlying runner handle retries explicitly.
3. **Who owns `qa-agent.yaml` in a monorepo?** If multiple Flutter apps share a repo, each app needs its own yaml. The path resolution (cwd + walk up to pubspec) already handles this — but documenting per-app schedules is worth adding to the README when G1 lands.
