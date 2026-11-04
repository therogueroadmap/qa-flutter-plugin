---
name: qa-flutter-test-planner
description: Use BEFORE running QA on a Flutter feature to produce an auditable test plan covering screens, preconditions, end-to-end flows, acceptance criteria and out-of-scope risks. Generates a markdown+YAML plan in qa-plugin-config/qa-plans/<feature>.md that the runners (qa-flutter-manual-runner, qa-flutter-android-runner, qa-flutter-web-runner) can consume via --plan=<path>. NOT a test runner — it never executes tests, only plans them. NOT for unit/widget tests — use qa-flutter-unit-generator directly.
argument-hint: "<feature-name> [--auto] [--output=<path>]"
---

# qa-flutter-test-planner

Produces an auditable QA test plan for a Flutter feature **before any test is generated or executed**. You implement the `/qa-plan` command.

The output is a single artifact `qa-plugin-config/qa-plans/<feature>.md` with YAML frontmatter that downstream runners read as input. This decouples the *what to test* decision (planning) from the *how to test* mechanics (runner skills).

**Invocation examples:**
- `/qa-plan login` — interactive planning for the login feature; user reviews each section
- `/qa-plan login --auto` — generate the plan without prompts (CI/scheduled use)
- `/qa-plan checkout --output=qa-plugin-config/qa-plans/v1.2/checkout.md` — custom output path (e.g. version-scoped)

## Step 1 — Parse arguments

Extract from the invocation string:
- `FEATURE` = first positional arg, trimmed and lowercased
- `AUTO_MODE` = true if `--auto` present, false otherwise
- `OUTPUT_PATH` = value after `--output=` if present, else `qa-plugin-config/qa-plans/{FEATURE}.md`

If `FEATURE` is empty → abort:
```
⛔ Falta el nombre de la feature.
   Uso: /qa-plan <feature> [--auto] [--output=<path>]
```

If `AUTO_MODE` is true, print:
```
Generando plan en modo automático — sin interacción durante el análisis.
```

## Step 2 — Read configuration

Read `qa-plugin-config/qa-agent.yaml` from the project root using the Read tool. Extract:

| Variable | yaml path | Default |
|---|---|---|
| `PLATFORM` | `project.platform` | `"android"` |
| `ANDROID_STACK` | `project.android_stack` | `"flutter_drive"` |
| `FLUTTER_PATH` | `project.flutter.path` (if web/appium) | cwd |
| `BACKEND_TEST_URL` | `backend.test_url` | empty |
| `PLAN_DIR` | `planning.test_plan_dir` | `qa-plugin-config/qa-plans/` |

If `qa-agent.yaml` does not exist → continue with defaults but emit a warning:
```
⚠ qa-plugin-config/qa-agent.yaml no encontrado. Generando plan con valores por defecto (platform=android, stack=flutter_drive).
   El plan será válido pero los runners requerirán el yaml para ejecutarlo.
```

If `OUTPUT_PATH` is the default and `PLAN_DIR` is set, override `OUTPUT_PATH = {PLAN_DIR}/{FEATURE}.md`.

## Step 3 — Semantic discovery (multi-layer)

Reuse the three-layer resolution from `qa-flutter-manual-runner` § C.1, **generalized to return all relevant pages** (not just top candidate). See [`references/screens-discovery.md`](references/screens-discovery.md) for the full playbook.

Quick summary:

**Layer 1 — Pages.** Glob `lib/**/pages/*.dart` and `lib/**/screens/*.dart`. Score each:
- Class name contains FEATURE keywords: **+40**
- Imported provider/bloc has HTTP endpoint matching FEATURE: **+35**
- Page is a route destination in `lib/config/app_router.dart` (or `lib/router/`): **+25**

**Layer 2 — API endpoints.** Grep `lib/**/providers/*.dart`, `lib/**/bloc/*.dart`, `lib/**/services/*.dart` for `get(`, `post(`, `put(`, `delete(`, `http.*`, `dio.*`. Match URL strings against FEATURE keywords. Flag POST/PUT/DELETE as **MUTANT**.

**Layer 3 — Router.** Read the router file. For each candidate page, identify `entry_route` and reachable `next_routes` (transitions reachable from the page).

Build `SCREENS`: a deduplicated list of `{page_class, page_file, entry_route, score, endpoints[], next_routes[], MUTANT}`.

**Filtering rule:** keep all screens with `score >= 40` (configurable in `planning.score_threshold`, default 40). Screens with lower scores go to **risks / out-of-scope** (Step 7).

If `SCREENS` is empty → abort:
```
⛔ No se encontró ninguna pantalla relacionada con '{FEATURE}' en el proyecto.
   Revisa el nombre, indica un alias en planning.aliases o crea el plan manualmente.
```

## Step 4 — Multi-screen flow inference

For each screen in `SCREENS`, walk `next_routes` up to depth 3 (configurable `planning.flow_depth`) and build flow chains.

**Algorithm:**
1. Start from each screen with `score >= 60` (entry candidates).
2. Follow `next_routes` until: (a) a leaf screen (no further nav), (b) loop detected, (c) depth limit reached.
3. Each chain is a candidate **happy path flow**: `Screen A → action → Screen B → action → Screen C`.
4. For each MUTANT endpoint touched in the chain, mark the flow as `data_mutating: true`.

**Error/edge cases:** for every form/input widget detected on a screen (Layer 1 read), add an error variant flow:
- Empty submission → expect validation message
- Invalid format (email, number) → expect specific error
- Network failure on MUTANT endpoint → expect error toast/dialog

Build `FLOWS`: list of `{name, type: "happy"|"error", steps[], data_mutating}`.

If no flows could be inferred (no router hits, no input widgets) → keep `FLOWS = []` and surface in Step 7 as risk "navegación no inferible — flujos requieren definición manual".

## Step 4.W — Web-specific widget analysis (NEW v1.5, B.2)

Ejecuta solo si `PLATFORM == "web"`. Enriquece los flows con markers `[NAV] / [INTERACT] / [INPUT] / [UNKNOWN]` para que el driver pueda generar specs correctos.

Para cada `screen` en `SCREENS`:

### 4.W.1 — Leer el archivo de la screen

```python
content = read(screen.file)
```

### 4.W.2 — Detectar inputs (TextFormField con labelText)

Regex:
```regex
TextFormField\([^)]*labelText:\s*['"]([^'"]+)['"]
```

Para cada match:
- Emitir entrada en `WIDGETS_FOUND[screen]`: `{type: "input", label: match[1], marker: "INPUT"}`.

### 4.W.3 — Detectar buttons con label visible

Regex:
```regex
(?:Filled|Elevated|Text|Outlined|Icon)Button(?:\.icon)?\(
  [^)]*?onPressed:\s*([^,)]+(?:\([^)]*\))?[^,)]*?)
  [^)]*?(?:child:\s*Text\(['"]([^'"]+)['"]\)|label:\s*Text\(['"]([^'"]+)['"]\))
```

(Flag `re.DOTALL` o equivalente — el constructor puede ocupar múltiples líneas.)

Para cada match:
- `callback_body = match[1]` (texto literal del callback, o nombre de función referenciada).
- `visible_text = match[2] or match[3]`.
- Determinar `marker` por heurísticas (ver 4.W.5).

### 4.W.4 — Detectar otros widgets tapeables (InkWell / GestureDetector)

Regex:
```regex
(?:InkWell|GestureDetector)\([^)]*onTap:\s*([^,)]+)
```

Para cada match:
- `callback_body = match[1]`.
- `visible_text` = inferir del context (ej. `Text` widget adjacent) o dejar `null`.
- Si `visible_text` es null → marker = `UNKNOWN`, agregar nota en el plan: "widget no addressable por texto visible — el humano debe agregar `Semantics(label: 'X')` o cambiar a un `*Button` con `child: Text(...)`".

### 4.W.5 — Heurísticas para determinar marker

Aplicar en orden, primer match gana:

```
1. Si callback_body contiene cualquier de:
     context.go
     context.goNamed
     Navigator.push
     Navigator.pop
     Navigator.replace
     Navigator.pushNamed
     Navigator.pushReplacement
   → marker = "NAV" (alta confianza)

2. Si callback_body contiene cualquier de:
     _scrollTo
     scrollController.animateTo
     showDialog
     showModalBottomSheet
     showBottomSheet
     Scaffold.of(...).openDrawer
     Scaffold.of(...).openEndDrawer
   → marker = "INTERACT" (alta confianza)

3. Si callback_body es referencia a función nombrada (ej. `_handlePress`, `_onLogin`):
   Leer el cuerpo de esa función en el MISMO archivo (depth max 1 — no profundizar más).
   Aplicar reglas 1 y 2 sobre ese cuerpo.
   Si encuentra → marker correspondiente.
   Si no → marker = "UNKNOWN".

4. En cualquier otro caso → marker = "UNKNOWN".
```

### 4.W.6 — Persistir resultados

`WIDGETS_FOUND[screen]` ahora es una lista de:
- `{type: "input", label, marker: "INPUT"}`
- `{type: "button", visible_text, marker: "NAV"|"INTERACT"|"UNKNOWN"}`
- `{type: "tappable", visible_text|null, marker: "NAV"|"INTERACT"|"UNKNOWN"}`

Esta lista se usa en Step 5 (preconditions, para agregar el blocker de ensureSemantics) y Step 6 (Section 4 — Flujos, para inyectar los markers en cada step).

## Step 5 — Preconditions extraction

Inspect each screen and its providers to detect what the test will need at runtime. See [`references/preconditions-checklist.md`](references/preconditions-checklist.md) for the canonical list.

Detect:

| Source | What to extract |
|---|---|
| `String.fromEnvironment('X')` calls in pages/providers | Required env vars (e.g. `TEST_EMAIL`, `TEST_PASSWORD`, `API_BASE_URL`) |
| HTTP base URL constants | `BACKEND_TEST_URL` must be reachable |
| MUTANT endpoints (POST/PUT/DELETE) | Flag plan as **data-mutating**; recommend test-profile backend |
| `AndroidManifest.xml` `<uses-permission>` | Device permissions required (CAMERA, LOCATION, etc.) — only for Android |
| `pubspec.yaml` packages with native deps | `firebase_*`, `google_maps_*`, `permission_handler` → flag as setup needed |
| Auth-protected routes (router guards) | Test user must be valid + active |
| Seed data inferred from page (e.g. lists requiring items) | Listar items mínimos requeridos |

Build `PRECONDITIONS`: list of `{category, description, blocker: true|false}`. `blocker: true` items must be checked off before the runner can succeed.

**Para `PLATFORM == "web"`**, agregar siempre como blocker (independiente de las heurísticas):
- `{category: "Dispositivo", description: "main() llama WidgetsFlutterBinding.ensureInitialized().ensureSemantics() antes de runApp() (sin esto el a11y tree está vacío y Playwright no encuentra widgets)", blocker: true}`.

## Step 6 — Acceptance criteria

**Para `PLATFORM == "web"`**, cada step del flow ya tiene un marker `[NAV] / [INTERACT] / [INPUT] / [UNKNOWN]` derivado de `WIDGETS_FOUND[screen]` (ver Step 4.W). Los emit en Section 4 del plan-template como prefijo del step (`N. [MARKER] descripción`).

For each flow in `FLOWS`, derive a verifiable expected result by inspecting the **terminal screen** of the chain:

- If terminal screen contains a `SnackBar`, `Dialog`, or `Toast` widget → "Aparece mensaje '{text}'"
- If terminal screen is a different page than start → "Navegación a '{terminal_route}' completa"
- If terminal screen renders state from a provider → "El provider '{name}' tiene estado '{success_state}'"
- For error flows → "Aparece validación '{validation_message}'" o "Permanece en '{current_route}' sin avanzar"

Build `CRITERIA`: list of `{flow_name, expected_result}`.

## Step 7 — Risks / out-of-scope

Compile risks from earlier steps:

1. **Screens excluded by score threshold** (Step 3): list each with `score < 40` and reason "score bajo — posible falso positivo".
2. **Flows not inferable** (Step 4): if no flows were derived, add risk.
3. **MUTANT flows without test backend configured**: if `BACKEND_TEST_URL` is empty AND any flow is `data_mutating`, add **blocker risk** "datos de producción en riesgo — configurar backend.test_url antes de ejecutar".
4. **Native deps requiring real device**: surface from Step 5.
5. **Multiple pages for same feature**: if `SCREENS` has more than 3 entries, suggest splitting the plan into sub-features.

Build `RISKS`: list of `{description, severity: "blocker"|"warning"|"info"}`.

## Step 8 — Render plan

Apply [`references/plan-template.md`](references/plan-template.md) substituting the data from Steps 3–7. Compute:

- `TIMESTAMP` = current UTC datetime as ISO-8601 (e.g. `2026-05-04T14-30-00Z`)
- `PLAN_VERSION` = `1` (schema version)
- `GENERATOR` = `qa-flutter-test-planner@1.0`

**Interactive mode (AUTO_MODE = false):**

Print each section to the user as it's built and ask:
```
Sección {N} — {nombre}: {resumen}
¿OK / Editar / Quitar? [O/E/Q]
```
Apply the user's choice before moving to the next section. After all sections, write `OUTPUT_PATH`.

**Automatic mode (AUTO_MODE = true):**

Skip prompts. Render the full plan and write `OUTPUT_PATH` directly.

Before writing, ensure parent directory exists:
```bash
mkdir -p "$(dirname OUTPUT_PATH)"
```

If a plan already exists at `OUTPUT_PATH`:
- Interactive: ask `"Ya existe un plan en {OUTPUT_PATH}. ¿Sobrescribir? [s/N]"`. Default no.
- Auto: increment version → bump frontmatter `version: N+1` and overwrite. Print: `⚠ Plan existente sobrescrito (version {N+1}).`

## Step 9 — Output

Print to stdout (machine-parseable):
```
QA_PLAN_PATH={OUTPUT_PATH}
QA_PLAN_FEATURE={FEATURE}
QA_PLAN_SCREENS_COUNT={len(SCREENS)}
QA_PLAN_FLOWS_COUNT={len(FLOWS)}
QA_PLAN_BLOCKER_RISKS={count of risks with severity=blocker}
```

If `QA_PLAN_BLOCKER_RISKS > 0`, also print:
```
⚠ El plan tiene {N} riesgos bloqueantes. Resuélvelos antes de invocar a un runner con --plan={OUTPUT_PATH}.
```

Print human summary:
```
Plan generado: {OUTPUT_PATH}
Pantallas: {N} | Flujos: {happy + error} | Precondiciones: {N} ({N_blocker} bloqueantes) | Riesgos: {N}

Próximo paso:
  /qa-run {FEATURE} --plan={OUTPUT_PATH}
```

---

## When NOT to use

- You're generating unit/widget tests — use `qa-flutter-unit-generator` directly (unit tests don't need multi-screen planning).
- You only want to **run** an existing test — use `/qa-run <feature>` (the runner without `--plan` does on-the-fly resolution as it always has).
- The feature isn't implemented yet — the planner reads code; without code there's nothing to plan against.
- The feature spans multiple repos / micro-frontends — this skill scopes to a single Flutter project tree.

## Common mistakes

| Mistake | Fix |
|---|---|
| Running the planner and immediately a runner without reading the plan | Defeats the purpose. The plan is a **review artifact** — read it (or have a human read it) before invoking `--plan=<path>`. |
| Generating a plan when the feature has no router entry | The planner will report 0 flows; this is honest output, not a bug. Add the route or define flows manually in the plan markdown. |
| Sobrescribir un plan editado a mano sin querer | El paso 8 pregunta antes de sobrescribir en modo interactivo. En modo `--auto` bumpea version y deja el anterior visible en git. Commit los planes para no perder ediciones manuales. |
| Tratar el plan como código generado y no editarlo | El plan es markdown editable a propósito. Refinarlo manualmente es parte del flujo. Los runners parsean la frontmatter y las secciones bien-formadas; texto adicional se ignora. |
| Confundir `planning.test_plan_dir` con `reports.output_dir` | Planes son input (commiteables, vivos). Reportes son output (ephemeros, generalmente .gitignored). Mantenerlos en directorios separados. |

## Plan consumption by runners

Once a plan exists, runners can be invoked with `--plan=<path>`:

```
/qa-run login --plan=qa-plugin-config/qa-plans/login.md
```

The runner reads the plan's frontmatter for routing hints, the **Pantallas a cubrir** section to know which page files to analyze, the **Precondiciones** section as additional pre-flight checks, and the **Flujos** section to drive test generation (replacing the on-the-fly semantic resolution in Section C.1–C.2 of `qa-flutter-manual-runner`).

If `qa-agent.yaml` declares `planning.require_plan: true`, runners abort when invoked without `--plan` and no plan exists in `planning.test_plan_dir/{feature}.md`.

## Decision — when to plan vs. when to run directly

| Situation | Recommendation |
|---|---|
| First time touching a feature | **Plan first.** Review screens/flows/risks before generating tests. |
| Regression on a stable feature with existing test | Run directly (`/qa-run <feature>`). Plan would be redundant. |
| Multi-screen flow (checkout, onboarding) | **Plan first.** Single-page resolution misses the chain. |
| Critical feature listed in `release_gate.critical_features` | **Plan first.** Auditable artifact justifies coverage decisions. |
| Quick smoke test during dev | Run directly. |
| CI/nightly stability gate | Use plans if available; runners fall back to on-the-fly resolution otherwise. |
