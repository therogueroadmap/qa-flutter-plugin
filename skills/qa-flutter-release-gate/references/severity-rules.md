# Severity classification rules

Every QA finding is mapped to one of four severity buckets. GO/NO-GO verdict derives from counts per bucket + the active threshold profile.

## Severity buckets

| Bucket | Icon | Production impact | Default action |
|---|---|---|---|
| **Critical** | 🔴 | App unusable, data loss, security breach, payment broken | Block GO, no waiver |
| **High** | 🟠 | Core feature broken, high-traffic flow degraded, build red | Block GO unless explicit waiver in yaml |
| **Medium** | 🟡 | Secondary feature failing, regression in edge case, coverage well below target | Warn; does not block by default |
| **Low** | 🔵 | Minor flakiness, analyzer warnings, cosmetic issues, coverage slightly below target | Info only |

## Classification rules (applied in order)

For each finding across all QA reports, apply the first matching rule:

### Critical

- E2E test fails on a feature listed under `release_gate.critical_features` in `qa-agent.yaml`.
- E2E test fails on login, auth, payment, or checkout (hardcoded defaults if not overridden).
- Compile error (`COMPILE_ERROR`) on any integration test.
- Flutter build fails (`BUILD_FAILED`).
- Crash on app startup observed in any report.
- `flutter analyze` emits any `error •` (not warnings).
- Pre-flight aborted with `API_BASE_URL` pointing to production (guard triggered).
- Unit test `PARCIAL` on a class under `lib/**/security/**`, `lib/**/auth/**`, or `lib/**/payment/**`.

### High

- E2E test fails on any other feature not in `critical_features`.
- `flutter test` suite reports 1+ failing unit tests in a module with coverage > 60%.
- Integration `TOKEN_RETRY_FAIL` (re-auth loop exhausted).
- `TIMEOUT` on any E2E test (even if secondary).
- Coverage for a critical feature below `release_gate.coverage_floor` (default 70%).
- ADB connection lost during a run (stability signal).

### Medium

- Unit test `PARCIAL` on non-critical classes.
- Coverage below `unit.coverage_target` by more than 10 percentage points.
- `flutter analyze` emits warnings on files touched in the current diff.
- A MUTANT-tagged test ran without a test-profile backend configured.
- Widget test reports "pocos Keys" warning on a page under active development.

### Low

- `flutter analyze` warnings on files NOT touched in the current diff.
- Coverage below target by 10 percentage points or less.
- A test reported `TOKEN_RETRY` (single retry, recovered) — not `TOKEN_RETRY_FAIL`.
- Golden tests skipped because no `--golden` flag was passed.
- `RUNNER_AUTOCREATED` note (the skill auto-created `test_runner.dart` — usually fine, confirm it was committed).

## Threshold profiles

The active profile determines the GO/NO-GO rule. Set via `--threshold=` flag or `release_gate.threshold` in yaml. Default: `normal`.

| Profile | GO if | NO-GO if |
|---|---|---|
| `strict` | 0 Critical AND 0 High AND ≤2 Medium | Any Critical or High, or >2 Medium |
| `normal` (default) | 0 Critical AND 0 High | Any Critical or High |
| `lenient` | 0 Critical | Any Critical |

**Critical is always blocking** across all profiles. Waivers for High exist only in `lenient`, or via explicit `release_gate.waivers[]` in yaml (see below).

## Explicit waivers

To waive a specific High finding (e.g., known flaky test under investigation), declare in `qa-agent.yaml`:

```yaml
release_gate:
  waivers:
    - finding_id: "login-e2e-step-3"    # match against report's step id or test name
      reason: "Known intermittent network timeout — tracked in TICKET-123"
      expires: "2026-05-31"              # ISO date; after this, waiver is ignored
```

A waived High is downgraded to Medium for GO/NO-GO purposes but still appears in the report under "Waivers applied".

## Yaml configuration

Full `release_gate` block:

```yaml
release_gate:
  threshold: "normal"                    # strict | normal | lenient
  critical_features:                     # features whose E2E failure is Critical
    - "login"
    - "payment"
    - "checkout"
  coverage_floor: 70                     # minimum coverage for critical features (%)
  notify_on_nogo:                        # optional — webhook/channel (plugin-specific)
    slack: "#qa-release"
  waivers:
    - finding_id: "..."
      reason: "..."
      expires: "YYYY-MM-DD"
```

All fields optional. Omitting the block applies defaults: `threshold=normal`, `critical_features=[login, auth, payment, checkout]`, `coverage_floor=70`, no waivers.

## Anti-patterns

- **Using `lenient` by default.** Defeats the purpose of a gate — only for hot-fix branches with explicit justification.
- **Adding permanent waivers.** Every waiver needs an `expires` date. Expired waivers are automatically ignored.
- **Hardcoding new severity rules in the skill.** Extensions go in `release_gate` config; the rules above are the contract.
- **Treating Medium/Low as "fix later" forever.** They should show up in a tracking system; the report flags the count but does not enforce.
