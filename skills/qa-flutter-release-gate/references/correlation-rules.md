# Root-cause correlation rules (release-gate Step 5.5)

Applied to findings AFTER severity classification. Each rule scans the finding set and, on match, emits a `CorrelatedFinding` that groups N findings under a shared root cause.

**Severity propagation:** the CorrelatedFinding's severity = `max(severity of each grouped finding)`. Correlation NEVER downgrades individual findings.

**Rule evaluation order:** rules are evaluated in the order listed. A finding may participate in multiple correlations; record all in its `correlated_with` field (list, not scalar).

---

## Rule: BACKEND_UNSTABLE

**Match:** `count(finding where layer == "e2e" AND regex_match(finding.details, /HttpException|SocketException|Connection refused|timeout.*http/i)) >= 2`

**Emit:**
```
root_cause: "Backend inestable durante el run — múltiples fallos de red en tests E2E"
severity: max(matching findings)
action: "Verificar health del backend antes de re-correr el gate. Revisar logs del servicio en health_url."
evidence: <finding IDs>
```

---

## Rule: AUTH_BROKEN

**Match:** `exists(finding where feature == "login" AND result == "FAIL") AND count(finding where feature != "login" AND layer == "e2e" AND result in {"FAIL", "PARCIAL"}) >= 1`

**Emit:**
```
root_cause: "Autenticación rota — fallos downstream por login fallido"
severity: max(matching findings — login finding included)
action: "Revisar POST /auth/login del backend. Verificar que devuelve 200 con credenciales test_user antes de re-correr."
evidence: <finding IDs>
```

---

## Rule: DEVICE_FLAKY

**Match:** `count(finding where result in {"ADB_FAIL", "TIMEOUT"}) >= 2` AND `NOT all those findings match the same error pattern` (i.e. diverse errors — if same error, it's a bug, not flakiness)

**Emit:**
```
root_cause: "Dispositivo/emulador inestable durante el run"
severity: max(matching findings)
action: "Revisar estado del emulador/device. Considerar wipe + cold boot. Ver emulator logs en /tmp/qa-bootstrap-emulator.log."
evidence: <finding IDs>
```

---

## Rule: COVERAGE_LOW_AGGREGATE

**Match:** `count(finding where result == "COVERAGE_LOW" AND feature == F) >= 2` for some feature F (multiple coverage reports for the same feature all flag low)

**Emit:**
```
root_cause: "Feature {F} sistemáticamente infra-testeada — múltiples capas bajo el floor"
severity: max(matching findings)
action: "Agregar tests en las capas faltantes de {F} (ver reportes unit + widget). Floor configurable en release_gate.coverage_floor."
evidence: <finding IDs>
```

---

## Rule: BUILD_REGRESSION

**Match:** `exists(finding where result == "COMPILE_ERROR")` AND `count(finding where result in {"FAIL", "SKIPPED"}) >= 2`

**Emit:**
```
root_cause: "Regresión de compilación bloquea el run completo"
severity: Critical (always — compile error is always blocking)
action: "Resolver el compile error antes de re-correr. Tests subsecuentes no son interpretables hasta que build compile."
evidence: <finding IDs>
```

---

## Extending

Add new rules by appending a new `## Rule: NAME` section here. No code changes required.

Guidelines:
- Rules must be static and deterministic — no ML, no heuristics based on run history.
- Match predicates should use counts, regex matches, and feature/layer/result field comparisons only.
- Severity is always `max(group)` — never introduce "correlation bonus severity".
