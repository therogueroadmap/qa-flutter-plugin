---
name: qa-flutter-manual-runner
description: Use when running functional QA on a Flutter Android app via flutter_drive (integration_test package). Covers regression of existing integration tests and generation of new ones from semantic resolution of lib/src/pages and lib/src/providers (or from a pre-generated plan via --plan). Requires a real Android device or emulator and a running backend. NOT for Appium-based testing — use qa-flutter-android-runner. NOT for unit/widget/state tests — use qa-flutter-unit-generator. NOT for Flutter web — use qa-flutter-web-runner.
argument-hint: "<feature-name> [--auto] [--plan=<path>]"
---

# qa-flutter-manual-runner

Functional QA runner for Flutter Android apps. You implement the `/qa-run` command.

**Invocation examples:**
- `/qa-run login` — test login (existing test) or generate+persist if new
- `/qa-run login --auto` — same, no user interaction
- `/qa-run login --plan=qa-plugin-config/qa-plans/login.md` — use a pre-generated plan from `qa-flutter-test-planner`
- `/qa-run regresion` — run full regression suite sequentially
- `/qa-run regresion --auto` — full suite, CI mode

## Step 1 — Parse arguments

Extract from the invocation string:
- `FEATURE` = first positional arg (text before any `--flag`), trimmed and lowercased
- `AUTO_MODE` = true if `--auto` present, false otherwise
- `PLAN_PATH` = value after `--plan=` if present, else empty

If AUTO_MODE is true, print immediately:
```
Ejecutando en modo automático — sin interacción durante el run.
```

If `PLAN_PATH` is set, print:
```
Usando plan: {PLAN_PATH}
```
The plan will be loaded in Section C.0 before any semantic resolution.

## Step 2 — Read configuration

Read `qa-plugin-config/qa-agent.yaml` from the project root using the Read tool. Extract:

| Variable | yaml path |
|---|---|
| `DEVICE_ID` | `device.id` |
| `APP_PACKAGE` | `device.app_package` |
| `TEST_TIMEOUT` | `timeouts.test_seconds` |
| `ADB_RETRY_COUNT` | `timeouts.adb_retry_count` |
| `ADB_RETRY_WAIT` | `timeouts.adb_retry_wait_seconds` |
| `RESET_BEFORE_EACH` | `execution.reset_app_before_each` |
| `BACKEND_TEST_URL` | `backend.test_url` |
| `REPORTS_DIR` | `reports.output_dir` |
| `PLAN_DIR` | `planning.test_plan_dir` (optional, default `qa-plugin-config/qa-plans/`) |
| `REQUIRE_PLAN` | `planning.require_plan` (optional, default `false`) |

If `PLAN_PATH` was empty in Step 1 AND `planning.test_plan_dir` is configured, set `PLAN_PATH = {PLAN_DIR}/{FEATURE}.md` if that file exists. Otherwise leave `PLAN_PATH` empty.

If `REQUIRE_PLAN` is true AND `PLAN_PATH` is empty AND `FEATURE != "regresion"` → abort:
```
⛔ planning.require_plan está activado y no existe plan para '{FEATURE}'.
   Genera el plan primero: /qa-plan {FEATURE}
```

If `qa-plugin-config/qa-agent.yaml` does not exist → abort:
```
⛔ qa-plugin-config/qa-agent.yaml no encontrado en la raíz del proyecto.
   Crea el archivo en qa-plugin-config/ con la configuración del dispositivo y backend.
```

**Read `pubspec.yaml` once here** — extract `name:` → `PUBSPEC_NAME` and `version:` → `PUBSPEC_VERSION`.
Both values are reused in Section C.3.2, E, and F. Do NOT read `pubspec.yaml` again in those sections.

## Step 3 — Pre-flight checks

Execute in order. Each failure aborts the run without executing any tests.

### 3.1 — Verify .env complete and secure

Read `.env` from the project root. Extract `TEST_EMAIL`, `TEST_PASSWORD`, `API_BASE_URL`.

If `.env` missing or any field empty → abort:
```
⛔ .env incompleto. Campos requeridos: TEST_EMAIL, TEST_PASSWORD, API_BASE_URL.
   Copia .env.example a .env y rellena los valores.
```

If `BACKEND_TEST_URL` is empty or missing from `qa-agent.yaml` → abort:
```
⛔ backend.test_url no está configurado en qa-agent.yaml.
   Define la URL del entorno de test para proteger datos de producción.
```

If `API_BASE_URL` does not contain `BACKEND_TEST_URL` → abort:
```
⛔ API_BASE_URL no coincide con el entorno de test configurado.
   Esperado (backend.test_url): {BACKEND_TEST_URL}
   Encontrado (API_BASE_URL):   {API_BASE_URL}
   Ajusta API_BASE_URL en .env antes de continuar.
```

### 3.2 — Health check backend

Attempt in order via Bash:

**Attempt 1:**
```bash
curl -s -o /dev/null -w "%{http_code}" --max-time 10 {API_BASE_URL}/health
```
If output is `200` → backend available, continue to 3.3.

**Attempt 2** (if attempt 1 is not `200`):
```bash
curl -s -o /dev/null -w "%{http_code}" --max-time 10 \
  -X POST {API_BASE_URL}/login \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"{TEST_EMAIL}\",\"password\":\"{TEST_PASSWORD}\"}"
```
If output is `200` or `401` → backend available, continue to 3.3.

If connection error or timeout → abort:
```
⛔ Backend no disponible en {API_BASE_URL}. Verifica conectividad.
```

### 3.3 — Verify / launch device

Read `autonomous.device.boot_avd` from `qa-agent.yaml`.

Run via Bash:
```bash
adb devices
```

**Case A — marker file exists (bootstrap skill ran before).** Read `{REPORTS_DIR}/.qa-bootstrap-marker` (YAML). Use the `device.id` field. Skip AVD launch; device is ready.

**Case B — no marker, but `autonomous.device.boot_avd` is set in yaml:**
1. If `DEVICE_ID` appears in `adb devices` with status `device` → continue to 3.4.
2. If not, launch the explicit AVD:
   ```bash
   emulator -avd {autonomous.device.boot_avd} -no-snapshot-load &
   ```
3. Poll boot for `autonomous.device.boot_timeout_seconds` seconds (default 90):
   ```bash
   adb wait-for-device
   adb shell getprop sys.boot_completed
   ```
4. On timeout:
   ```
   ⛔ AVD '{boot_avd}' no booteó en {N}s.
   ```

**Case C — no marker AND no `autonomous.device.boot_avd` (legacy fallback):**
1. Run: `emulator -list-avds`. If empty → abort.
2. Take the first AVD name alphabetically.
3. Emit warning:
   ```
   ⚠ Sin autonomous.device.boot_avd en qa-agent.yaml; usando el primer AVD alfabético ({avd_name}).
     Fragil en CI — define autonomous.device.boot_avd para runs reproducibles.
   ```
4. Launch with `emulator -avd {avd_name} -no-snapshot-load &`, poll boot up to 90s.
5. On timeout → abort with same message as Case B.

All cases: after success, `DEVICE_ID` is the id visible in `adb devices`.

### 3.4 — Verify / create test_runner.dart

Check if `integration_test/fixtures/test_runner.dart` exists.

If it does not exist → create it:
```dart
import 'package:integration_test/integration_test.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
}
```
Set flag `RUNNER_AUTOCREATED = true`.

## Step 4 — Route by feature

Evaluate in order:

1. If `FEATURE == "regresion"` → **[Section A]**
2. Else if `integration_test/manual/{FEATURE}_test.dart` exists → **[Section B]**
3. Else → **[Section C]**

---

## Section A — Regression Suite

List all `*.dart` files in `integration_test/manual/` (exclude `.gitkeep`). Sort alphabetically. Each filename without `_test.dart` is the feature slug.

Print:
```
Iniciando suite de regresión — {N} features: {slug1}, {slug2}, ...
```

For each feature slug, **sequentially**:
1. If `RESET_BEFORE_EACH` is true:
   ```bash
   adb -s {DEVICE_ID} shell pm clear {APP_PACKAGE}
   ```
   Wait 2 seconds.
2. Set `TARGET_FILE = integration_test/manual/{slug}_test.dart`.
3. Execute → **[Section D]** → returns `RESULT`, `DRIVE_OUTPUT`, `DURATION`.
4. Generate report → **[Section E]** with `MODE_LABEL = "regresion"`.

After all features: generate summary → **[Section F]**.

---

## Section B — Single Regression Test

1. If `RESET_BEFORE_EACH` is true:
   ```bash
   adb -s {DEVICE_ID} shell pm clear {APP_PACKAGE}
   ```
   Wait 2 seconds.
2. Set `TARGET_FILE = integration_test/manual/{FEATURE}_test.dart`.
3. Execute → **[Section D]** → returns `RESULT`, `DRIVE_OUTPUT`, `DURATION`.
4. Generate report → **[Section E]** with `MODE_LABEL = "regresion individual"`.

---

## Section C — New Feature Test

### C.0 — Plan-driven path (skip C.1–C.2 if plan provided)

If `PLAN_PATH` (set in Step 1 or 2) is non-empty:

1. Read `PLAN_PATH` with the Read tool.
2. Verify the YAML frontmatter has `feature: {FEATURE}` (case-insensitive). If mismatched → abort:
   ```
   ⛔ Plan {PLAN_PATH} corresponde a feature '{plan_feature}', no a '{FEATURE}'.
   ```
3. Parse the markdown:
   - **Section 2 — Pantallas a cubrir** (table) → list of `(page_class, page_file, entry_route)`. Treat each row as one candidate. This **replaces** Layer 1+3 of C.1.
   - **Section 3 — Precondiciones** → list of `(description, blocker)`. For each `[ ]` (blocker) item, before continuing verify it is satisfied. If not, abort listing the unmet blockers (in interactive mode, ask the user to confirm; in `--auto` mode, abort).
   - **Section 4 — Flujos end-to-end** → list of flows. Each flow's steps become test step seeds for C.3.2 generation.
4. Set `SELECTED` = the candidates from Section 2 (already deduplicated by the planner).
5. Skip C.1 and C.2 entirely. Jump to **C.3** for generation.

If `PLAN_PATH` is empty, fall through to C.1 (legacy on-the-fly resolution — unchanged behavior).

### C.1 — Semantic resolution (three layers)

**IMPORTANT — Grep before Read:** Do NOT read all dart files upfront. Use Grep to identify
candidate files first, then Read only those files. This avoids loading irrelevant source
files into context (the main source of unnecessary token usage in this skill).

**Layer 1 — Pages**

Step 1a — Grep for class/file name matches:
```bash
grep -rl "{FEATURE}" lib/src/pages/ lib/core/ --include="*.dart"
```
Step 1b — Grep for FEATURE keywords in class declarations:
```bash
grep -rl "class.*{FEATURE}" lib/src/pages/ lib/core/ --include="*.dart" -i
```
Merge both lists (deduplicated) → `CANDIDATE_FILES`.

If `CANDIDATE_FILES` is empty → fallback: Glob all `*.dart` in `lib/src/pages/` but Read
only the first 50 lines of each (enough to get the class name and imports) using `limit: 50`.

Step 1c — Read only `CANDIDATE_FILES` (full read). For each, assign a score 0–100:
- Class name contains keywords from FEATURE: **+40**
- Associated provider/bloc has HTTP endpoint matching FEATURE keywords: **+35**
- Page appears as route destination in `lib/config/app_router.dart`: **+25**

**Layer 2 — API endpoints**

Search `lib/src/providers/` and `lib/src/bloc/` (use Grep) for HTTP method calls (`get(`, `post(`, `put(`, `delete(`, `http.get`, `http.post`, etc.) and URL strings containing FEATURE keywords. Flag POST/PUT/DELETE endpoints as MUTANT. Associate endpoints with the page that imports that provider/bloc.

**Layer 3 — Router**

Read `lib/config/app_router.dart`. For each candidate, identify entry route and exit routes.

Build `CANDIDATES`: each entry = `{page_file, page_class, provider_file, provider_class, score, endpoints[], entry_route, MUTANT}`.

If `CANDIDATES` is empty → abort:
```
⛔ No se encontró implementación para '{FEATURE}'.
   Revisa el nombre o indica el archivo manualmente.
```

### C.2 — Resolve candidates by mode

**Interactive mode (AUTO_MODE = false):**

Present:
```
Para '{FEATURE}' encontré estos candidatos:
1. {page_class} ({page_file}) — {entry_route} | {endpoint_summary} [score: {N}]
2. ...
¿Cuál cubre el flujo que quieres testear? (número)
```
Wait for user response. Set `SELECTED` to chosen entry.

**Automatic mode (AUTO_MODE = true):**

Sort `CANDIDATES` by score descending. Maintain set `COVERED` of `(page_class, provider_class)` pairs.
For each candidate: if pair in `COVERED` → mark OMITIDO; else mark A_EJECUTAR and add to `COVERED`.

Print deduplication table:
```
Candidatos evaluados para '{FEATURE}':
| Candidato | Score | Página | Provider | Decisión |
|-----------|-------|--------|----------|----------|
```

Set `SELECTED` = all A_EJECUTAR entries.

### C.3 — Generate, execute, persist per candidate

For each candidate in `SELECTED`, **sequentially**:

**C.3.1 — Filename**
- Single candidate: `TEST_FILE = integration_test/manual/{FEATURE}_test.dart`
- Multiple candidates: `TEST_FILE = integration_test/manual/{FEATURE}_{page_slug}_test.dart`

**C.3.2 — Generate test**

Analyze `page_file` and `provider_file` to identify navigation path, widget finders (Keys preferred, else `find.text`, `find.byType`, `find.widgetWithText`), happy path steps, and at least one error case.

**If `PLAN_PATH` was loaded in C.0:** use the flows declared in Section 4 of the plan as the source of truth for steps. Each plan step is a Spanish-language directive (e.g. "Ingresar TEST_EMAIL en Key('login.email')"); translate it into the corresponding `tester.enterText` / `tester.tap` / `await tester.pumpAndSettle()` Dart calls. Error flows from Section 4.E become the `error case` test groups. Page-source analysis still runs to resolve finder details (Keys, widget types) but does not invent new steps.

**Finder pitfalls (verificado 2026-04-24):**
- `ElevatedButton.icon(...)` retorna un subtipo privado (`_ElevatedButtonWithIcon`). `find.byType(ElevatedButton)` devuelve 0 matches aunque el botón esté en pantalla. Lo mismo aplica a `.icon` de `TextButton`, `OutlinedButton`, `FilledButton`.
- Para botones creados con `.icon`: preferir `find.text('{label}')` o `find.byIcon(Icons.{name})`. Evitar `find.widgetWithText(ElevatedButton, ...)` y `find.ancestor(matching: find.byType(ElevatedButton))`.
- Si necesitas el widget completo (no solo hijo): usar `find.ancestor(of: find.text('{label}'), matching: find.byWidgetPredicate((w) => w is ButtonStyleButton))` — match por superclase pública.
- Regla general: si un widget se instancia vía constructor factory con nombre (`.icon`, `.small`, etc.), asumir que el tipo runtime es privado hasta verificar lo contrario.

Use `PUBSPEC_NAME` extracted in Step 2 as `APP_PACKAGE`. Do NOT read `pubspec.yaml` again here.

Write `TEST_FILE`:
```dart
{if MUTANT: // qa:creates-backend-data — este test crea datos reales en el backend.}
{if MUTANT: // Recomendado: ejecutar con API_BASE_URL apuntando al perfil de test del backend.}
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:{APP_PACKAGE}/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  const email = String.fromEnvironment('TEST_EMAIL');
  const password = String.fromEnvironment('TEST_PASSWORD');

  group('{FEATURE} — QA Suite', () {
    testWidgets('happy path', (tester) async {
      app.main();
      await tester.pumpAndSettle();
      // [steps generated from page analysis]
    });

    testWidgets('error case — validación', (tester) async {
      app.main();
      await tester.pumpAndSettle();
      // [error steps generated from page analysis]
    });
  });
}
```

**C.3.3 — Reset app**
If `RESET_BEFORE_EACH`: `adb -s {DEVICE_ID} shell pm clear {APP_PACKAGE}`, wait 2s.

**C.3.4 — Execute**
Set `TARGET_FILE = TEST_FILE`. Execute → **[Section D]**.

**C.3.5 — Persist or discard**
- PASS → keep file, set `MODE_LABEL = "nueva feature → persistido"`
- FAIL/ERROR/TIMEOUT → delete file (`rm TEST_FILE`), set `MODE_LABEL = "nueva feature → descartado"`

**C.3.6 — Report**
Call **[Section E]**. If MUTANT, include "Datos creados en backend" section.

---

## Section D — Execute flutter drive

Record `START_TIME`. Set `TOKEN_RETRY = false`.

Run via Bash (timeout = `TEST_TIMEOUT` seconds):
```bash
flutter drive \
  --driver=integration_test/fixtures/test_runner.dart \
  --target={TARGET_FILE} \
  --dart-define=TEST_EMAIL={TEST_EMAIL} \
  --dart-define=TEST_PASSWORD={TEST_PASSWORD} \
  --dart-define=API_BASE_URL={API_BASE_URL} \
  -d {DEVICE_ID}
```

Capture full stdout+stderr as `DRIVE_OUTPUT`. Record `EXIT_CODE`.

**Determine RESULT:**

| Condition | RESULT |
|---|---|
| EXIT_CODE == 0, no `FAILED` in output | PASS |
| EXIT_CODE == 0, `FAILED` present | PARCIAL |
| EXIT_CODE != 0, compile error in output | COMPILE_ERROR |
| Process killed by timeout | TIMEOUT |
| ADB connection lost (after retries) | ADB_FAIL → abort entire run |
| Other EXIT_CODE != 0 | ERROR |

**Token expiry detection** (check before finalizing RESULT):

Search `DRIVE_OUTPUT` for: `Sesión expirada`, `session expired`, `CierreSecionException`, `AuthExpirationException`, or login-screen widgets appearing in non-login steps.

If detected AND `TOKEN_RETRY == false`:
1. `adb -s {DEVICE_ID} shell pm clear {APP_PACKAGE}`, wait 2s
2. Re-run the exact same flutter drive command
3. Set `TOKEN_RETRY = true`
4. If token expiry detected again → RESULT = TOKEN_RETRY_FAIL

**ADB connection lost:**
Retry `ADB_RETRY_COUNT` times with `ADB_RETRY_WAIT` seconds between. If still failing → RESULT = ADB_FAIL → abort entire run.

`DURATION` = elapsed seconds since `START_TIME`.

Return: `RESULT`, `DRIVE_OUTPUT`, `DURATION`, `TOKEN_RETRY`.

---

## Section E — Individual Report

`TIMESTAMP` = current datetime as `YYYY-MM-DDTHH-MM`.
Use `PUBSPEC_NAME` → `APP_PACKAGE` and `PUBSPEC_VERSION` → `APP_VERSION` (read once in Step 2). Do NOT re-read `pubspec.yaml`.
Parse `DRIVE_OUTPUT` for step results where possible.

Result emoji: PASS → ✅ PASS | PARCIAL → ⚠ PARCIAL | else → ❌ {RESULT}

Write to `{REPORTS_DIR}/{TIMESTAMP}-{FEATURE}.md`:

```markdown
# QA Report — Feature: {FEATURE}
**Fecha:** {TIMESTAMP} | **Duración:** {DURATION}s | **Resultado:** {RESULT_EMOJI}
**App:** {APP_PACKAGE} v{APP_VERSION} | **Dispositivo:** {DEVICE_ID}
**Modo:** {MODE_LABEL}
{if TOKEN_RETRY:} **Nota:** Hubo reintento por expiración de token.
{if RUNNER_AUTOCREATED:} **Nota:** integration_test/fixtures/test_runner.dart creado automáticamente.

## Objetivo
{one sentence describing what this test validates}

## Pasos ejecutados
| # | Descripción | Resultado | Duración |
|---|-------------|-----------|----------|
{rows parsed from DRIVE_OUTPUT, or "No se pudieron parsear pasos individuales — ver output completo."}

{if RESULT != PASS:}
## Diagnóstico

**Error principal:** {first line matching one of: ^Error: .*, ^Exception: .*, ^FlutterError: .*, ^FAILED: .* — else "No se detectó error estructurado"}

**Stack trace relevante (top 8 frames):**
```
{up to 8 lines of DRIVE_OUTPUT matching these patterns, in order of appearance:
 - ^#\d+\s+.*               (Dart stack frame)
 - package:\S+\.dart:\d+:\d+ (location ref — prefer lib/ over package:flutter/)
 filter out frames containing 'dart:async-patch' or 'dart:async/zone'}
```

**Log relevante (últimas 20-30 líneas filtradas):**
```
{last 30 lines of DRIVE_OUTPUT that match patterns:
 - (?i)error|exception|failed|warning
 - ^(E|F)/
 if matches < 5 → fallback to tail -40 of raw DRIVE_OUTPUT with header "filtering produced insufficient lines".}
```

**Context budget for DRIVE_OUTPUT analysis:** Limit to the last 300 lines of DRIVE_OUTPUT when
searching for errors and stack traces. Full output beyond that rarely adds diagnostic value and
consumes context unnecessarily. If a specific error reference requires earlier context, search
by line number rather than loading everything.
{end if}

## Pendientes para corrección
{one actionable sentence per failed step, or "Sin pendientes."}

{if MUTANT:}
## Datos creados en backend
{list of POST/PUT/DELETE endpoints called}
```

Print path to report file.

---

## Section F — Summary Report (regresion mode only)

`TIMESTAMP` = current datetime as `YYYY-MM-DDTHH-MM`.
Use `PUBSPEC_NAME` → `APP_PACKAGE` and `PUBSPEC_VERSION` → `APP_VERSION` (already set in Step 2). Do NOT re-read `pubspec.yaml`.

Write to `{REPORTS_DIR}/{TIMESTAMP}-summary.md`:

```markdown
# QA Regression Summary — {TIMESTAMP}
**App:** {APP_PACKAGE} v{APP_VERSION} | **Dispositivo:** {DEVICE_ID} | **Features ejecutadas:** {N}

| Feature | Resultado | Duración | Fallos principales |
|---------|-----------|----------|--------------------|
| {slug} | {RESULT_EMOJI} | {DURATION}s | {first FAIL description or "—"} |

## Pendientes para planificación
{numbered list per FAIL or PARCIAL feature, or "Sin pendientes. Todos los tests pasaron."}
```

Print:
```
Suite completa. Resumen: {REPORTS_DIR}/{TIMESTAMP}-summary.md
```

---

## When NOT to use

- The project uses the Appium stack — use `qa-flutter-android-runner`.
- You need unit, widget, or state-management tests — use `qa-flutter-unit-generator`.
- You're on Flutter web — use `qa-flutter-web-runner`.
- The feature creates backend data and there is no test-profile backend configured (`backend.test_url`) — pre-flight will abort, and rightly so. Configure the test backend first.
- There is no running emulator/device and no AVD available — the skill will abort in Step 3.3.

## Common mistakes

| Mistake | Fix |
|---|---|
| `.env` has `API_BASE_URL` pointing to production | Pre-flight 3.1 guards against this — don't bypass by editing the check. Point to the test backend. |
| Skipping `test_runner.dart` creation | Pre-flight creates it automatically; if absent, `flutter drive` fails opaquely. |
| Running multiple instances in parallel on the same device | `flutter drive` locks the device. Use separate AVDs if truly parallel is needed. |
| Persisting a test that was generated but never passed | Section C.3.5 discards on FAIL — don't override that. A broken test in `integration_test/manual/` pollutes the regression suite. |
| Forgetting the MUTANT tag on data-creating tests | Without `// qa:creates-backend-data`, future test runners can't opt-in to data isolation. Add it when POST/PUT/DELETE is involved. |
| Relying on `Future.delayed` inside a test | Use `pumpAndSettle` or explicit `Completer`. Delayed waits cause flakiness on slower CI runners. |
| `find.byType(ElevatedButton)` para botones `.icon` | `ElevatedButton.icon` es un subtipo privado — `byType` no matchea. Usar `find.text(label)` o `find.byIcon(...)`. Ver detalle en Section C.3.2 → "Finder pitfalls". |

## Decision — when to pick this vs `qa-flutter-android-runner`

See [docs/android-stacks.md](../../docs/android-stacks.md) for the full decision matrix. Short version: this is the default for Android. Pick the Appium runner only when you need gestos nativos, multi-app flows, or pre-compiled APK testing.
