# qa-flutter-test-planner

QA test planner for Flutter projects, invocable from Claude Code.

Generates an auditable, human-editable test plan covering screens, preconditions, end-to-end flows, acceptance criteria, and risks — **before** any test is generated or executed by the runners.

---

## What it does

A single command — `/qa-plan <feature>` — produces a markdown+YAML plan at `qa-plugin-config/qa-plans/<feature>.md` that:

1. Lists which **screens** are touched by the feature (multi-screen, not just the entry page).
2. Documents **required preconditions**: env vars, backend state, test users, seed data, device permissions.
3. Infers **end-to-end flows**: happy path navigations + error/edge cases.
4. Defines **acceptance criteria**: verifiable expected results per flow.
5. Flags **risks and out-of-scope** items: low-confidence detections, missing routes, MUTANT data exposure.

The plan is consumed by the runners (`/qa-run`, `/qa-flutter-android-runner`, `/qa-flutter-web-runner`) via `--plan=<path>`.

---

## Why a separate planning step

Before this skill, the runners did on-the-fly *semantic resolution* (manual-runner § C.1) to decide what to test. That works for single-page features but:

- Misses multi-screen flows (checkout, onboarding, password recovery → reset).
- Gives no chance to **review** the test scope before tests are generated and executed.
- Doesn't surface preconditions until they fail at runtime (missing env var, empty backend).
- Leaves no auditable artifact for compliance or release-gate justification.

A standalone planning skill produces a **versionable, reviewable, editable** artifact and lets the runners stay focused on execution.

---

## Usage

### Generate a plan (interactive)

```
/qa-plan login
```

The skill walks through screens, preconditions, flows, criteria, and risks, asking "OK / Editar / Quitar?" after each section. Final plan written to `qa-plugin-config/qa-plans/login.md`.

### Generate a plan (auto / CI)

```
/qa-plan login --auto
```

No prompts. Useful before nightly stability runs where you want a fresh plan checked in alongside the report.

### Custom output path

```
/qa-plan checkout --output=qa-plugin-config/qa-plans/v1.2/checkout.md
```

Useful for version-scoping plans (e.g. one set per release).

### Then run with the plan

```
/qa-run login --plan=qa-plugin-config/qa-plans/login.md
```

The runner skips its on-the-fly resolution and uses the plan as input.

---

## Plan artifact structure

```markdown
---
feature: login
version: 1
generated_at: 2026-05-04T14-30-00Z
generator: qa-flutter-test-planner@1.0
platform: android
android_stack: flutter_drive
data_mutating: true
blocker_risks: 0
---

# Test Plan: login

## 1. Scope
...

## 2. Pantallas a cubrir
| # | Pantalla | Archivo | Entry route | Próximas rutas | Notas |
| 1 | LoginPage | lib/src/pages/auth/login_page.dart | /login | /home, /forgot-password | score 95, ⚠ MUTANT |
...

## 3. Precondiciones / datos requeridos
### 3.1 Variables de entorno
- [ ] TEST_EMAIL
- [ ] TEST_PASSWORD
### 3.2 Backend
- [ ] backend.test_url configurado
- [ ] Backend en perfil de test (esta feature crea datos)
...

## 4. Flujos end-to-end
### 4.1 Login exitoso (happy path)
1. Abrir app → LoginPage
2. Ingresar TEST_EMAIL en Key('login.email')
3. Ingresar TEST_PASSWORD en Key('login.password')
4. Tap en Key('login.submit')
5. Esperar navegación a /home
### 4.E. Casos de error
- Email vacío → mensaje 'Email requerido'
- Credenciales inválidas → mensaje 'Usuario o contraseña incorrectos'

## 5. Criterios de aceptación
| Flujo | Resultado esperado |
| Login exitoso | Navegación a /home, AuthProvider.isAuthenticated == true |
...

## 6. Riesgos / fuera de scope
- 🔵 Recuperación de contraseña detectada como ruta alcanzable pero excluida del scope (feature aparte)
```

---

## Configuration in `qa-plugin-config/qa-agent.yaml`

```yaml
planning:
  enabled: true                       # default false → backward-compatible
  test_plan_dir: "qa-plugin-config/qa-plans/"  # where plans are read/written
  require_plan: false                 # if true, runners abort without --plan
  score_threshold: 40                 # minimum Layer 1 score to include a screen
  flow_depth: 3                       # max router transitions per flow
  aliases:                            # FEATURE → synonyms for keyword matching
    "checkout": ["pago", "payment"]
    "registro": ["signup", "register"]
  precondition_severity_overrides:
    "Permiso CAMERA": "advisory"
```

All fields are optional. With `planning:` block absent, the planner uses defaults (`qa-plugin-config/qa-plans/`, threshold 40, depth 3, no aliases, no overrides).

---

## How runners consume the plan

When invoked with `--plan=<path>`:

| Runner | What it reads from the plan |
|---|---|
| `qa-flutter-manual-runner` (`/qa-run`) | Section 2 (pantallas) replaces Section C.1 semantic resolution. Section 3 (precondiciones) augments pre-flight checks. Section 4 (flujos) drives test generation in C.3. |
| `qa-flutter-android-runner` (Appium) | Section 2 → `feature_list`; Section 4 → test scenarios passed to `qa-flutter-android-tester` sub-agent. |
| `qa-flutter-web-runner` | Section 2 → impacted-areas (Step 6); Section 4 → checklist test cases (Step 7). |

Without `--plan`, runners fall back to their existing on-the-fly logic. **Backward-compatible by design.**

---

## Auto-injection by `qa-stability-agent`

If `planning.enabled: true` in `qa-agent.yaml`, the stability agent automatically looks for plans matching the features in the implementation summary and passes `--plan=<path>` to the runner. No manual `--plan` flag needed for auto-delegated runs.

If a feature has no plan and `planning.require_plan: true`, the agent aborts with NO-GO and reason "missing plan for feature `{name}`".

---

## Planning vs. running — when to do which

| Situation | Plan first? |
|---|---|
| First time touching a feature | ✅ Yes |
| Multi-screen flow (checkout, onboarding) | ✅ Yes |
| Critical feature in `release_gate.critical_features` | ✅ Yes |
| Regression on stable feature with existing test | ❌ No — run directly |
| Quick smoke during dev | ❌ No |
| CI/nightly stability gate | ✅ If plans exist; runners auto-fall-back if not |

---

## File structure summary

```
<project-root>/
  qa-plugin-config/
    qa-agent.yaml                        # add `planning:` block
    qa-plans/                            # commit these — they're living docs
      login.md
      checkout.md
      onboarding.md
    qa-reports/                          # outputs (gitignored)
  ...
```

> Plans are **input** to the QA process and should be committed.
> Reports are **output** and are typically `.gitignored`.
