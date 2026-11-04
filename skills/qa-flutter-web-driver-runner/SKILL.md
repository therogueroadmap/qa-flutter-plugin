---
name: qa-flutter-web-driver-runner
description: Use when running automated end-to-end QA on a Flutter web app via Playwright against Chrome (CanvasKit — único renderer en Flutter 3.29+). Sidesteps the hit-test mismatch that breaks `flutter drive -d chrome` on long flows (catalogued as K011) usando la estrategia de selectores de K012 (role + hasText + dispatchEvent + aria-label). Pre-requisito en la app Flutter — main() debe llamar `WidgetsFlutterBinding.ensureInitialized().ensureSemantics()`. Generates and runs `.spec.ts` Playwright tests, captures screenshots per step, and feeds run output to `qa-knowledge-manager learn`. NOT for Android — use `qa-flutter-manual-runner` (flutter_drive) or `qa-flutter-android-runner` (Appium). NOT for static stability checks (analyze/build/checklist) — use `qa-flutter-web-runner`. NOT for unit/widget/state tests — use `qa-flutter-unit-generator`.
argument-hint: "<feature-name> [--auto] [--plan=<path>] [--headed] [--no-bootstrap] [--skip-preflight-probe]"
---

# qa-flutter-web-driver-runner

End-to-end QA runner for Flutter web apps. Drives Chrome via Playwright against
the **HTML renderer** so widget hit-tests resolve against real DOM nodes.

**Invocation examples:**
- `/qa-web-drive login` — run existing test, or generate-and-run if new
- `/qa-web-drive login --auto` — no user interaction during the run
- `/qa-web-drive signup --plan=qa-plugin-config/qa-plans/signup.md` — plan-driven
- `/qa-web-drive login --headed` — show the browser (debug)

## Scope

This skill:
- ✅ Generates Playwright `.spec.ts` files from a plan (preferred) or from
  semantic resolution of `lib/pages` + `lib/state`.
- ✅ Runs them via `npx playwright test` against Chrome at `project.web.base_url`.
- ✅ Captures one screenshot per step + the standard Playwright trace.
- ✅ Persists passing tests under `qa-plugin-config/web-tests/`; discards on fail.
- ✅ Invokes `qa-knowledge-manager preflight` before and `learn` after.
- ✅ Emits a Markdown report under `qa-plugin-config/qa-reports/`.

This skill does NOT:
- Run tests against Android or iOS — wrong stack.
- Generate `Semantics(identifier: ...)` for the Flutter widgets — that is a
  manual instrumentation step. The skill aborts cleanly with guidance if no
  semantic identifiers are reachable.
- Auto-launch `flutter run -d chrome` — the app must already be served at
  `project.web.base_url`. Use a separate bootstrap step (or `qa-flutter-bootstrap`
  once it supports web, see ROADMAP).
- Replace `qa-flutter-web-runner` — that one is for static checks + manual
  checklists; this one is for automated browser flows.

## Why HTML renderer and not CanvasKit

`flutter drive -d chrome` defaults to CanvasKit, which paints widgets to a
single `<canvas>`. Hit-tests are computed by the test framework against
`tester.view.physicalSize`, but the actual browser viewport can differ, so
tap coordinates land on the Overlay's `RenderAbsorbPointer` instead of the
target widget. The Next-button wizard flows we saw choke at the second tap.
Full diagnostic in `qa-knowledge.yaml` entry **K011**.

The HTML renderer paints each Flutter element as a stack of `flt-*` DOM
elements, and `Semantics(identifier: 'x')` becomes `<flt-semantics id="x">`.
Playwright's hit-tests run inside the browser, so layouted offset and
viewport always agree.

## Configuration contract

From `qa-plugin-config/qa-agent.yaml`:

```yaml
project:
  platform: "web"
  flutter:
    path: "/abs/path/to/flutter/project"
  web:
    base_url: "http://localhost:4200"   # where `flutter run -d chrome` serves
    renderer: "html"                    # REQUIRED — driver mode needs HTML
  auth:                                 # optional
    email: "qa@example.com"
    password: "secret"
  driver:                               # optional block — defaults shown
    runtime: "playwright"               # only "playwright" supported in v1
    headed: false                       # set true to watch the browser
    test_timeout_seconds: 30
    retries: 0
agent:
  reports_output_dir: "qa-reports"
  web_tests_dir: "web-tests"            # relative to qa-plugin-config/
```

Resolve `CONFIG_PATH` at runtime — never hardcode:
1. `./qa-plugin-config/qa-agent.yaml` in cwd.
2. Walk up to find a `pubspec.yaml` sibling, then look for `qa-plugin-config/qa-agent.yaml` there.
3. If still missing, ask the user for the absolute path.

`PROJECT_ROOT` = directory containing `pubspec.yaml` (parent of `qa-plugin-config/`).
`QA_AGENT_DIR` = `{PROJECT_ROOT}/qa-plugin-config`.

## Step 1 — Parse arguments

Extract from invocation:
- `FEATURE` = first positional arg, trimmed and lowercased.
- `AUTO_MODE` = true if `--auto` present.
- `PLAN_PATH` = value after `--plan=` if present, else empty.
- `HEADED` = true if `--headed` present, else value of `project.driver.headed`.
- `NO_BOOTSTRAP` = true if `--no-bootstrap` present, else false. *(NEW v1.5, B.10)*
- `SKIP_PREFLIGHT_PROBE` = true if `--skip-preflight-probe` present, else false. *(NEW v1.5, B.8)*

If `AUTO_MODE` is true, print:
```
Ejecutando en modo automático — sin interacción durante el run.
```

## Step 0 — Bootstrap automático (NEW v1.5, B.10)

Esta sección decide si el driver invoca `qa-flutter-bootstrap` por su cuenta. Se ejecuta antes de Step 2 (que necesita el server arriba para Step 3+).

### 0.1 — Skip checks

- Si `NO_BOOTSTRAP == true` → log `[BOOTSTRAP] Skipped por --no-bootstrap.`, set `WE_INVOKED_BOOTSTRAP=false`, continuar a Step 2.

### 0.2 — Composition check (marker existente)

Resolver `MARKER_PATH` igual que el bootstrap: `{PROJECT_ROOT}/{reports.output_dir}/.qa-bootstrap-marker` (default `qa-plugin-config/qa-reports/.qa-bootstrap-marker`).

Si el marker existe y es parseable:
- Leer `caller` field.
- Si `caller != "qa-web-drive"` → log `[BOOTSTRAP] Reusing UP by '{caller}' (composition).`, set `WE_INVOKED_BOOTSTRAP=false`, continuar a Step 2.
- Si `caller == "qa-web-drive"` y `pid_ward` alive → abort:
  ```
  ⛔ Marker existente del mismo caller 'qa-web-drive' (pid {pid_ward}). Posible run paralelo.
     Si es stale, borra {MARKER_PATH} a mano y reintenta.
  ```

### 0.3 — Invocar bootstrap

Si el marker no existe (o pid era dead y se limpió), invocar `qa-flutter-bootstrap` vía Skill tool con argumentos `--up --caller=qa-web-drive`.

Parsear stdout:
- `QA_BOOTSTRAP_STATUS=UP` → continue, `WE_INVOKED_BOOTSTRAP=true`.
- `QA_BOOTSTRAP_STATUS=ALREADY_UP_BY={x}` → continue, `WE_INVOKED_BOOTSTRAP=false` (raro post-0.2 pero posible).
- Otro / exit≠0 → abort:
  ```
  ⛔ qa-flutter-bootstrap --up falló (caller=qa-web-drive).
     Detalle: {stdout/stderr resumido}.
     Para correr el driver sin bootstrap (server ya levantado):
       /qa-web-drive {feature} --no-bootstrap
  ```

Registrar `WE_INVOKED_BOOTSTRAP` para reusar en Step Final.

## Step 2 — Read configuration

Read `qa-plugin-config/qa-agent.yaml`. Extract every field from the contract above.
If `project.platform` ≠ `"web"` → abort: wrong platform, suggest `qa-flutter-manual-runner`.

Read `pubspec.yaml` once. Extract `name:` → `PUBSPEC_NAME` and `version:` → `PUBSPEC_VERSION`.
Reuse in Section F. Do NOT read `pubspec.yaml` again.

## Step 3 — Pre-flight checks

> **Required reading antes de editar `_helpers.ts` o generar specs con inputs / botones:** [`references/canvaskit-fill-strategy.md`](references/canvaskit-fill-strategy.md). Este archivo consolida el árbol de decisión de fill/tap (qué primitiva usar según el tipo de widget). Consultarlo evita iterar a ciegas.

### 3a — Knowledge preflight (non-fatal)
Invoke `qa-knowledge-manager preflight`. Continue regardless of outcome — preflight
only auto-applies known fixes; it does not gate the run.

### 3b — Validate Node + Playwright
Run via Bash:
```bash
node --version
npx --yes playwright --version 2>&1
```
Si `node` falta → abort. Si Playwright falta → abort:
```
⛔ Playwright no instalado. Ver "QA web con browser automation" en el README para el quick start de instalación.
```

### 3c — Validar Flutter web app is up (red de seguridad)
```bash
curl -sf -o /dev/null -w "%{http_code}" --max-time 5 "{project.web.base_url}"
```
Esperar `200`. **Comportamiento**:

- Si responde `200` → continuar normal.
- Si NO responde → warn (no abort). El `qa-flutter-bootstrap --up` debió validar esto antes; si esta línea dispara, hay race entre bootstrap y driver, o el usuario invocó el driver standalone sin levantar el server.
  ```
  ⚠ base_url no responde aunque el bootstrap debió garantizar UP. Posible race condition o invocación standalone.
    Si invocaste el driver directo (no por /qa-stability-agent), arranca el server:
      /qa-bootstrap --up
    o, alternativamente, levanta la app a mano:
      cd {project.flutter.path} && flutter run -d chrome --web-port={port_from_base_url}
    Continuando con los siguientes checks; los tests pueden fallar si el server no aparece en breve.
  ```
- Reintenta el check una vez tras 3 segundos. Si sigue sin responder → abort:
  ```
  ⛔ base_url sigue sin responder. Investiga el marker del bootstrap o levanta el server manualmente.
  ```

### 3d — Renderer (informativo, NO bloquea)

> **Nota v1.4.1**: la flag `--web-renderer` fue removida en Flutter 3.29; CanvasKit es el único renderer. Esta skill está alineada a CanvasKit. El campo `project.web.renderer` en el yaml se mantiene como informativo (back-compat) pero **no se valida** ni se usa para bifurcar comportamiento.

(Step 3d previo —chequeo de `CanvasKit` vs `HTML` con curl al `main.dart.js`— eliminado en v1.4.1.)

### 3e — Pre-flight a11y probe (NEW v1.5, B.8)

> **Cambio v1.5**: en v1.4.x el chequeo de a11y tree era "informativo" y se delegaba al primer test del usuario (que fallaba con timeout genérico de 20s). v1.5 ejecuta un **Playwright probe propio** (~5-8s) antes del spec del usuario para abort con mensaje específico si la app no llama `ensureSemantics()`.

Pre-requisito en la app Flutter: `main()` debe invocar
```dart
WidgetsFlutterBinding.ensureInitialized().ensureSemantics();
```
Sin esto, el a11y tree está vacío y los selectores K012 (role + hasText + aria-label) no encuentran nada.

#### Comportamiento

Si `SKIP_PREFLIGHT_PROBE == true` (flag escape hatch) → log `[PREFLIGHT] Skipped a11y probe.`, continuar a Step 4.

Si no:

1. **Crear archivo temporal** `{QA_AGENT_DIR}/{web_tests_dir}/_preflight.spec.ts`:

   ```typescript
   import { test, expect } from '@playwright/test';

   const BASE_URL = process.env.BASE_URL ?? 'http://localhost:4200';

   test('pre-flight: a11y tree emitted', async ({ page }) => {
     await page.goto(BASE_URL, { waitUntil: 'networkidle', timeout: 10_000 });
     await page.locator('flt-glass-pane').waitFor({ state: 'attached', timeout: 8_000 });
     const count = await page.locator('flt-semantics').count();
     expect(count, 'flt-semantics nodes emitted').toBeGreaterThan(0);
     console.log(`A11Y_TREE_OK count=${count}`);
   });
   ```

2. **Ejecutar** vía Bash con timeout duro 25s:

   ```bash
   cd {QA_AGENT_DIR}/{web_tests_dir} && \
     BASE_URL={project.web.base_url} \
     npx playwright test _preflight.spec.ts --reporter=list 2>&1
   ```

3. **Cleanup**: borrar `_preflight.spec.ts` en `finally` (siempre, pase o falle).

4. **Parsear resultado**:
   - Exit 0 y stdout contiene `A11Y_TREE_OK count=` → continue a Step 4.
   - Exit ≠ 0 y stdout contiene `flt-semantics nodes emitted: expected 0 to be greater than 0` → abort:
     ```
     ⛔ A11y tree vacío en {base_url}. La app no llamó ensureSemantics().

        En lib/main.dart, antes de runApp(), agrega:
          WidgetsFlutterBinding.ensureInitialized().ensureSemantics();

        Luego rebuild + retry:
          flutter build web {build_flags}
          /qa-web-drive {feature}

        Detalle: qa-plugin-config/qa-knowledge.yaml → K001.
     ```
   - Exit ≠ 0 por timeout o error de navegación → abort:
     ```
     ⛔ Pre-flight probe inconclusive: {error resumido}.
        Verifica que {base_url} responde y que no hay errores de consola.
        Para saltear el probe (no recomendado):
          /qa-web-drive {feature} --skip-preflight-probe
     ```

## Step 4 — Route by feature

Evaluate in order:
1. If `FEATURE == "regresion"` → **[Section A]**
2. Else if `{QA_AGENT_DIR}/{web_tests_dir}/{FEATURE}.spec.ts` exists → **[Section B]**
3. Else → **[Section C]**

---

## Section A — Regression suite

List `{QA_AGENT_DIR}/{web_tests_dir}/*.spec.ts`. Sort alphabetically.
For each: run via **[Section E]** sequentially. Aggregate results.
Generate summary via **[Section F]** with `MODE_LABEL = "regresion"`.

## Section B — Single regression

Set `TARGET_FILE = {QA_AGENT_DIR}/{web_tests_dir}/{FEATURE}.spec.ts`.
Run via **[Section E]**. Report via **[Section F]** with `MODE_LABEL = "regresion individual"`.

## Section C — New feature test

### C.0 — Plan-driven path
If `PLAN_PATH` is non-empty:
1. Read plan. Verify YAML frontmatter `feature: {FEATURE}` AND `platform: web`.
2. **Run Section C.0.1 (plan validation)** — abort si hard-reject.
3. Parse Section 4 (flujos) → ordered list of step directives con markers.
4. **Antes de generar el spec en C.3**, consultar [`references/canvaskit-fill-strategy.md`](references/canvaskit-fill-strategy.md) para mapear cada marker `[INPUT]` / `[INTERACT]` a la primitiva correcta (`fillInput` / `fillUnlabeledTextbox` / `tapButton` / etc).
5. Skip C.1 and C.2 (no semantic resolution needed). Jump to **C.3**.

If empty, fall through to C.1.

### C.0.1 — Plan validation (NEW v1.5, B.3)

Ejecuta solo si `PLAN_PATH` no está vacío.

#### Frontmatter validation

- Require `plan.frontmatter.platform == "web"`:
  - Fail → abort:
    ```
    ⛔ Plan declara platform='{x}' pero el yaml dice 'web'. Mismatch.
       Regenerá el plan con /qa-plan {feature}.
    ```
- Require `plan.frontmatter.feature == FEATURE_NAME`:
  - Fail → warn:
    ```
    ⚠ Plan feature='{x}' difiere del argumento '{FEATURE}'. Procediendo.
    ```

#### Anti-pattern scan (hard reject)

Buscar en TODO el texto del plan (no solo Section 4):

- `hits = grep(plan_text, r"Semantics\(identifier:|semanticsById|tapBySemantics|fillText\(page,")` → si hits > 0:
  ```
  ⛔ Plan menciona patrones obsoletos: {matched strings}.
     Estos no funcionan en Flutter ≥3.29 (CanvasKit only):
       - Semantics(identifier:) no se serializa al DOM.
       - semanticsById / tapBySemantics / fillText eran helpers v1.4.0 deprecated.

     Regenerá el plan con /qa-plan {FEATURE} (planner v1.5+ emite K012 strategy).
  ```

#### Identifier-with-dots scan (hard reject)

- `matches = regex_find(plan_text, r"identifier:\s*['\"]([\w-]+\.[\w.-]+)['\"]")` → si matches:
  ```
  ⛔ Identifiers con dots detectados: {matches}.
     CSS no soporta dots en selectores `[id="..."]` sin escape.
     Usa dashes: 'field-profile-email' en vez de 'field.profile.email'.
  ```

#### Key(...) scan (warn)

- `hits = grep(plan_text, r"\bKey\(['\"]")` → si hits > 0:
  ```
  ⚠ Plan menciona Key('...'). Keys no se serializan al DOM en CanvasKit.
     Si quieres un test que use Keys, este es el plugin equivocado (usa flutter_drive
     para Android). Procediendo con K012 strategy (role + hasText + aria-label).
  ```

#### UNKNOWN markers scan (warn)

- `unknowns = count_occurrences(plan_text, r"\[UNKNOWN\]")` → si unknowns > 0:
  ```
  ⚠ Plan tiene {N} steps con marker [UNKNOWN]. El planner no pudo determinar
     si la acción navega o solo es interacción local. El spec generado va a
     emitir TODO comments en esos pasos para que los corrijas a mano.
     Para evitar: edita el plan reemplazando [UNKNOWN] con [NAV] o [INTERACT]
     en los flujos correspondientes.
  ```

#### ensureSemantics precondition check (warn)

- Si la Section 3 del plan no contiene "ensureSemantics":
  ```
  ⚠ Plan Section 3 no menciona ensureSemantics() como precondition.
     El pre-flight (Step 3e) lo va a validar igual al runtime.
  ```

Si pasó todos los hard-rejects → continue a C.3.

### C.1 — Semantic resolution (lightweight)
**Grep before Read.** Identify candidate page/state files:
```bash
grep -rl "{FEATURE}" lib/pages/ lib/state/ --include="*.dart"
```
Read at most the top 3 candidates (full file). For each, score:
- Class name matches FEATURE: **+40**
- Route in `lib/config/app_router.dart` contains FEATURE: **+25**
- HTTP call (POST/PUT/DELETE) involving FEATURE keyword: **+35** AND mark MUTANT.

### C.2 — Resolve mode
**Interactive (AUTO_MODE=false):** present numbered candidates, wait for choice.
**Automatic (AUTO_MODE=true):** pick top-score; deduplicate by `(page, state)`.

### C.3 — Generate test

`TEST_FILE = {QA_AGENT_DIR}/{web_tests_dir}/{FEATURE}.spec.ts`

#### Markers consumption (v1.5, B.3)

Para cada step en Section 4 del plan (formato `N. [MARKER] descripción`), emitir código según el marker:

| Marker | Código emitido |
|---|---|
| `[NAV]` | `await tapButton(page, '{visible_text}');`<br>`await page.waitForFunction(() => /* nav esperada */, undefined, { timeout: 10_000 });`<br>`await takeStepShot(page, info, '{step_label}');` |
| `[INTERACT]` | `await tapButton(page, '{visible_text}');`<br>`await page.waitForTimeout(800);  // scroll / modal / etc`<br>`await takeStepShot(page, info, '{step_label}');` |
| `[INPUT]` | `await fillInput(page, '{label}', '{value}');` *(v1.5.1: click-prime + fill — ver K013)* |
| `[UNKNOWN]` | `// TODO: el planner marcó este step como UNKNOWN.`<br>`// Texto: "{step_text}".`<br>`// Decide: tapButton + waitForFunction (navega) o tapButton + waitForTimeout (interacción local).` |
| **(sin marker, back-compat)** | Trata como `[INTERACT]`. Emite warn en stdout: `⚠ Step '{step_text}' sin marker; tratando como [INTERACT] (back-compat). Vuelve a generar el plan para markers explícitos.` |

**Helpers avanzados (v1.5.1)** — disponibles pero no emitidos automáticamente por C.3; el humano los usa al editar el spec generado:

- `fillUnlabeledTextbox(page, value)` — para inputs sin `aria-label` (típico de teléfonos o campos numéricos sin labelText en Flutter). Ver K015.
- `lockInputValue(page, label, value)` — para tests de validación donde Chrome autofill puede pisar el value tras `fillInput()`. Ver K016.

#### Translate plan steps

Translate los steps usando los helpers de `_helpers.ts` (creado en el primer run si falta — ver C.4):

```typescript
// {TEST_FILE} — generated by qa-flutter-web-driver-runner on {timestamp}
{if MUTANT:}// qa:creates-backend-data — corre solo contra un backend de test.
import { test, expect } from '@playwright/test';
import { openApp, buttonByText, tapButton, fillByAriaLabel, takeStepShot } from './_helpers';

const BASE_URL = process.env.BASE_URL || '{project.web.base_url}';
const EMAIL = process.env.TEST_EMAIL || '{auth.email or "qa@example.com"}';

test.describe('{FEATURE} — QA web suite', () => {
  test('happy path', async ({ page }, info) => {
    await openApp(page, BASE_URL);
    // [steps generated from plan/page analysis, one per line]
    // Example shape (Flutter 3.29+ / CanvasKit):
    //   await fillByAriaLabel(page, 'Email *', EMAIL);
    //   await takeStepShot(page, info, 'profile-filled');
    //   await tapButton(page, 'Siguiente');
    //   await expect(buttonByText(page, 'Confirmar')).toBeAttached();
  });

  test('error case — validación', async ({ page }, info) => {
    await openApp(page, BASE_URL);
    // [error steps from plan Section 4.E or generated]
  });
});
```

### C.4 — Ensure helper module exists

**Si `{QA_AGENT_DIR}/{web_tests_dir}/_helpers.ts` ya existe → NO sobrescribir.** Asumir que el proyecto tiene su versión personalizada (común en proyectos que ya invirtieron en QA web). Si quieres regenerar, borrarlo a mano antes de invocar.

> **Antes de editar `_helpers.ts` (custom o template) por cualquier razón**, leer [`references/canvaskit-fill-strategy.md`](references/canvaskit-fill-strategy.md). Codifica qué primitiva usar según el widget. Editar el helper sin haberlo consultado es exactamente el patrón que causó pérdida de tiempo en una sesión de validación interna.

Si está ausente, crear el siguiente helper alineado a Flutter ≥3.29 (CanvasKit + a11y tree). Ver K012 para el detalle del modelo.

```typescript
// _helpers.ts — Flutter web (CanvasKit + a11y tree) helpers para Playwright
//
// Modelo (validado empíricamente contra una app Flutter web de prueba, Flutter 3.38, v1.5.1):
//   - Pre-req en main(): WidgetsFlutterBinding.ensureInitialized().ensureSemantics()
//   - Botones / menuitems: flt-semantics[role="button"|"menuitem"]. Flutter pone el
//     texto visible tanto en textContent COMO en aria-label según el widget;
//     tapButton hace búsqueda DUAL (ver K014).
//   - Click sobre flt-semantics: cadena dispatchEvent (pointer+mouse) — Playwright
//     .click() NO dispara onPressed de Flutter.
//   - Inputs con aria-label: input[aria-label="<label>"] en el DOM permanente.
//     Flutter NO bindea ese input al TextEditingController hasta que recibe foco.
//     Click-prime (click force) attacha el text editing client; el fill() siguiente
//     SÍ llega al controller (ver K013).
//   - Inputs SIN aria-label: flt-semantics[role="textbox"] sin aria-label.
//     dispatch chain sobre ese nodo + keyboard.type — el foco está activo, el value
//     entra al controller letra a letra (ver K015).
//   - Chrome autofill en headed mode puede pisar el value tras fill(); lockInputValue
//     congela el value para tests de validación (ver K016).
// Ver K012, K013, K014, K015, K016 en qa-knowledge.yaml.

import type { Locator, Page, TestInfo } from '@playwright/test';

/** Espera a que Flutter inflate y emita el accessibility tree. */
export async function openApp(page: Page, baseUrl: string): Promise<void> {
  await page.goto(baseUrl, { waitUntil: 'networkidle' });
  await page.locator('flt-glass-pane').waitFor({ state: 'attached', timeout: 20_000 });
  await page.locator('flt-semantics').first().waitFor({ state: 'attached', timeout: 20_000 });
  await page.waitForLoadState('networkidle');
}

/** Locator a un botón Flutter por su texto visible (legacy/inspección).
 *  NO usar para click — `hasText` solo examina textContent. Para taps reales
 *  usar `tapButton` que también busca por aria-label. */
export const buttonByText = (page: Page, text: string | RegExp): Locator =>
  page.locator('flt-semantics[role="button"]', { hasText: text }).first();

/** Click sobre botón / menuitem Flutter por texto visible.
 *  Búsqueda DUAL: matchea por textContent O aria-label (Flutter pone el texto
 *  en uno u otro según el widget — un botón con ícono típicamente lo pone en
 *  aria-label sin textContent). Dispatch chain (pointer+mouse) porque
 *  Playwright .click() NO llega al onPressed de Flutter. */
export async function tapButton(page: Page, text: string | RegExp): Promise<void> {
  const src = typeof text === 'string' ? text : text.source;
  const matched = await page.evaluate((textSrc: string) => {
    const re = new RegExp(textSrc, 'i');
    const all = Array.from(document.querySelectorAll('flt-semantics[role="button"], flt-semantics[role="menuitem"]'));
    const el = all.find((n) => re.test(n.textContent ?? '')) ??
               all.find((n) => re.test((n as HTMLElement).getAttribute('aria-label') ?? ''));
    if (!el) return false;
    const r = (el as HTMLElement).getBoundingClientRect();
    const opts: PointerEventInit = {
      bubbles: true, cancelable: true,
      clientX: r.x + r.width / 2, clientY: r.y + r.height / 2, button: 0,
    };
    for (const type of ['pointerdown', 'mousedown', 'pointerup', 'mouseup', 'click']) {
      el.dispatchEvent(
        new (type.startsWith('pointer') ? PointerEvent : MouseEvent)(type, opts),
      );
    }
    return true;
  }, src);
  if (!matched) throw new Error(`tapButton: no flt-semantics[role="button"|"menuitem"] matched "${src}" (textContent or aria-label)`);
}

/** Llena un input identificado por aria-label (el labelText del TextFormField).
 *  Click-prime + fill: el click({ force: true }) hace que Flutter attache un text
 *  editing client al input permanente; el fill() siguiente escribe en ese input
 *  y Flutter lo lee como value del TextEditingController. Sin el click-prime,
 *  fill() puede no llegar al controller en escenarios con foco previo activo
 *  (ej. otro field perdió el foco y dejó la conexión en estado limbo). Ver K013. */
export async function fillInput(page: Page, label: string, value: string): Promise<void> {
  const input = page.locator(`input[aria-label="${label}"], textarea[aria-label="${label}"]`).first();
  await input.waitFor({ state: 'attached', timeout: 15_000 });
  await input.click({ force: true });
  await page.waitForTimeout(80);
  await input.fill(value);
  await page.waitForTimeout(200);
}

/** DEPRECATED en v1.5.1 — usar `fillInput` que aplica click-prime. Alias por back-compat. */
export const fillByAriaLabel = fillInput;

/** Llena un input sin aria-label (típicamente teléfono o campos numéricos
 *  donde Flutter no expone labelText al DOM). Estrategia: dispatch chain
 *  sobre el primer `flt-semantics[role="textbox"]` SIN aria-label visible →
 *  Flutter da foco de teclado al input on-demand → keyboard.type() escribe
 *  letra a letra (no hay locator ambiguity porque keyboard usa el activeElement
 *  del browser). Ver K015. */
export async function fillUnlabeledTextbox(page: Page, value: string): Promise<void> {
  await page.evaluate(() => (document.activeElement as HTMLElement | null)?.blur());
  await page.waitForTimeout(100);
  const found = await page.evaluate(() => {
    const target = Array.from(
      document.querySelectorAll<HTMLElement>('flt-semantics[role="textbox"]'),
    ).find((el) => !el.getAttribute('aria-label'));
    if (!target) return false;
    const r = target.getBoundingClientRect();
    const opts: PointerEventInit = {
      bubbles: true, cancelable: true,
      clientX: r.x + r.width / 2, clientY: r.y + r.height / 2, button: 0,
    };
    for (const type of ['pointerdown', 'mousedown', 'pointerup', 'mouseup', 'click']) {
      target.dispatchEvent(
        new (type.startsWith('pointer') ? PointerEvent : MouseEvent)(type, opts),
      );
    }
    return true;
  });
  if (!found) throw new Error('fillUnlabeledTextbox: no flt-semantics[role="textbox"] without aria-label found');
  await page.waitForTimeout(200);
  await page.keyboard.type(value);
  await page.waitForTimeout(200);
}

/** Congela el value de un input para que Chrome autofill (en modo headed) no lo
 *  pise después del fill(). Útil para tests de validación donde escribimos
 *  intencionalmente un value inválido y el browser quiere autofillearlo con uno
 *  válido. DEBE llamarse DESPUÉS de fillInput() — si se llama antes, Playwright
 *  ve el value locked al chequear pre-fill y skipea el CDP call. Ver K016. */
export async function lockInputValue(page: Page, label: string, value: string): Promise<void> {
  await page.locator(`input[aria-label="${label}"]`).evaluate(
    (el: HTMLInputElement, locked: string) => {
      Object.defineProperty(el, 'value', {
        get: () => locked,
        set: (_v: string) => { /* block Chrome autofill writes */ },
        configurable: true,
      });
      el.addEventListener('input', (e: Event) => {
        const ie = e as InputEvent;
        if (ie.inputType === 'insertReplacementText' || ie.inputType === 'insertFromAutofill') {
          e.stopImmediatePropagation();
        }
      }, true);
    },
    value,
  );
}

/** Captura screenshot del paso actual y la adjunta al reporte de Playwright. */
export async function takeStepShot(page: Page, info: TestInfo, label: string): Promise<void> {
  const path = `${info.outputDir}/${label}.png`;
  await page.screenshot({ path, fullPage: false });
  await info.attach(label, { path, contentType: 'image/png' });
}
```

### C.5 — Execute
Set `TARGET_FILE = TEST_FILE`. Run via **[Section E]**.

### C.6 — Persist or discard
- PASS → keep, `MODE_LABEL = "nueva feature → persistido"`
- FAIL/ERROR/TIMEOUT → `rm TEST_FILE`, `MODE_LABEL = "nueva feature → descartado"`

### C.7 — Report
Call **[Section F]**.

---

## Section E — Execute Playwright

Record `START_TIME`.

Generate a one-off `playwright.config.ts` under `{QA_AGENT_DIR}/{web_tests_dir}/`
if missing (idempotent):
```typescript
import { defineConfig } from '@playwright/test';
export default defineConfig({
  testDir: '.',
  timeout: {project.driver.test_timeout_seconds | default 30} * 1000,
  retries: {project.driver.retries | default 0},
  use: {
    headless: !{HEADED},
    baseURL: '{project.web.base_url}',
    trace: 'retain-on-failure',
    screenshot: 'only-on-failure',
    video: 'off',
  },
});
```

Run via Bash (timeout = `test_timeout_seconds + 30`):
```bash
cd {QA_AGENT_DIR}/{web_tests_dir} && \
  TEST_EMAIL={auth.email} TEST_PASSWORD={auth.password} BASE_URL={base_url} \
  npx playwright test {basename(TARGET_FILE)} --reporter=list 2>&1
```

Capture full stdout+stderr as `DRIVE_OUTPUT`. Record `EXIT_CODE`.

**Determine RESULT:**

| Condition | RESULT |
|---|---|
| EXIT_CODE == 0, no `failed` in summary | PASS |
| EXIT_CODE == 0, some specs failed | PARCIAL |
| EXIT_CODE != 0, compile/TS error in output | COMPILE_ERROR |
| Process killed by timeout | TIMEOUT |
| Other EXIT_CODE != 0 | ERROR |

`DURATION` = seconds since `START_TIME`.

**Tracking de progreso (R4):** si `RESULT == PASS` y este test estaba en la lista de fallas previas, agregar su nombre a `tests_passing_baseline` y resetear `helpers_edits_without_progress = 0`. Si `RESULT != PASS` y esta sección viene de una iteración tras una edición de `_helpers.ts`, mantener el contador (ya fue incrementado al editar). Ver [Operational directives](#operational-directives--iteration-safety-new-v16-spec-c-r4) para el resto del contrato.

Return `RESULT`, `DRIVE_OUTPUT`, `DURATION`.

---

## Section E.1 — Post-failure knowledge auto-match (NEW v1.6)

**When to run:** sólo si `RESULT ∈ {PARCIAL, ERROR, TIMEOUT, COMPILE_ERROR}`. Si `RESULT == PASS`, skip y pasa directamente a Section F.

**Por qué existe esta sección:** las knowledge entries (K001-K016) tienen `pattern` regex que matchea errores conocidos. Sin esta auto-match step, las entries son documentación pasiva — sólo útiles si un humano las lee proactivamente. Activarlas inline tras cada falla evita iterar sobre problemas ya resueltos.

**Steps:**

1. **Resolver AMBOS knowledge paths (UNIÓN, no fallback exclusivo):**
   - `PROJECT_KNOWLEDGE` = `{QA_AGENT_DIR}/qa-knowledge.yaml` (proyecto — entries `learn`-eadas por el usuario)
   - `PLUGIN_KNOWLEDGE` = `templates/qa-knowledge.yaml` del clone del plugin (entries curadas — incluye K013-K016 del modelo de selectores CanvasKit, K011-K012 de la estrategia base)
   - **El cambio importante** (v1.6.1): el plugin yaml NO es fallback — siempre se carga junto con el del proyecto. Antes solo se consultaba uno, dejando K013-K016 dormidas si el proyecto no las había learn-eado.
   - Si ninguno de los dos existe → skip con log: `[KNOWLEDGE-MATCH] No knowledge files found; skipping.`

2. **Cargar entries de ambos archivos y deduplicar por `id`:**
   - Parsear ambos YAMLs. Construir un dict `entries_by_id`.
   - Cargar primero las del plugin (`PLUGIN_KNOWLEDGE`), luego las del proyecto (`PROJECT_KNOWLEDGE`) → en colisión gana el proyecto (override del usuario tiene prioridad sobre el template).
   - Filtrar `enabled: true` con `pattern` no vacío.
   - Llevar contadores `count_project` y `count_plugin_only` para el log de visibilidad.

3. **Match contra `DRIVE_OUTPUT`:** para cada entry resultante, compilar el `pattern` como regex case-insensitive **DOTALL** (el dot debe matchear newlines) y aplicar contra `DRIVE_OUTPUT` (stdout + stderr completos).

   **Reglas críticas para evaluar los regex correctamente:**

   - Los stack traces de Playwright tienen MÚLTIPLES líneas (TimeoutError / Call log / waiting for...). Un regex con `.*` puede NO matchear si el motor no tiene flag DOTALL activo. Los patterns del template **ya usan `[\s\S]*` en lugar de `.*`** para ser cross-line safe en cualquier engine (fix v1.6.2).
   - **NO hagas "match mental"** — confiar en que el LLM evalúe regex contra outputs largos es propenso a falsos negativos (caso real: v1.6.1 reportó "sin matches" cuando K013 sí matcheaba). Cuando esté disponible, **invocar Node.js inline** para compilar el regex real:

     ```bash
     node -e "
     const fs = require('fs');
     const output = fs.readFileSync('/tmp/drive_output.txt', 'utf8');
     const entries = require('js-yaml').load(fs.readFileSync('PATH_TO_YAML', 'utf8'));
     for (const e of entries.knowledge_base) {
       if (!e.enabled || !e.pattern) continue;
       const re = new RegExp(e.pattern, 'is');  // i=case-insensitive, s=dotall
       if (re.test(output)) console.log('MATCH:', e.id, '-', e.description);
     }
     "
     ```

     Si Node no está disponible (raro post-Step 3b), usar Python equivalente con `re.IGNORECASE | re.DOTALL`. En último caso, evaluar mentalmente PERO con la convención: tratar `.` como si matcheara cualquier carácter incluyendo `\n`, y validar dos veces.

   - **Capturar cuál entry matcheó** (id) para imprimir luego.

4. **Si hay al menos un match**, imprimir bloque destacado:

```
─── KNOWLEDGE AUTO-MATCH ───
El error matchea {N} entrada(s) conocida(s) de un total de {TOTAL} evaluadas
(proyecto: {count_project}, plugin: {count_plugin_only}).
Antes de proponer fixes o iterar, aplicar lo siguiente:

[K0XX] (source: project|plugin) {description}
{fallback_message literal completo}

[K0YY] (source: project|plugin) {description}
{fallback_message literal completo}
─── END AUTO-MATCH ───

Nota: si un match parece indirecto (el error visible no es el causal directo
sino un síntoma posterior — ej. timeout esperando elemento del siguiente
paso tras un fill helper que aparentó funcionar), el fallback_message sigue
siendo el cierre correcto: revisar el snippet del helper apuntado.

Si después de aplicar lo anterior el problema persiste, registrar por qué
el patrón no aplicó (genera contexto para una nueva entrada de knowledge).
Si esta es la N-ésima iteración sobre el mismo error, consultar
Operational directives — Iteration safety.
```

5. **Si no hay matches**, imprimir UNA línea de visibility (no skip silencioso):

```
[KNOWLEDGE-MATCH] Auto-match consultó {TOTAL} entries (proyecto: {count_project}, plugin: {count_plugin_only}) — sin matches contra el output actual.
```

Esto confirma al usuario que la sección corrió y le da contexto de cobertura. Si `TOTAL` es chico, sugiere que hay knowledge entries del plugin que no se cargaron por error.

**Reglas duras:**

- **No omitir** esta sección bajo "el error parece obvio" — el auto-match es barato (~50 entries × regex) y la prevención de un ciclo de iteración fallida vale el costo.
- **Imprimir el `fallback_message` literal**, no parafrasear. El texto fue escrito con cuidado y suele incluir snippets de código accionables.
- **Marcar el `source`** en cada match (`project` o `plugin`) — esto le dice al usuario si la entrada viene de su yaml local o del template del plugin, lo cual cambia cómo modificarla.
- Si una entry matchea pero su `solution.auto_apply == true`, agregar nota: *"K0XX tiene auto-apply; correr `qa-knowledge-manager apply K0XX` antes de iterar manualmente."*

---

## Section F — Report

`TIMESTAMP` = current datetime `YYYY-MM-DDTHH-MM`.
Use `PUBSPEC_NAME` → `APP_PACKAGE` and `PUBSPEC_VERSION` → `APP_VERSION` (set in Step 2).

Result emoji: PASS → ✅ PASS | PARCIAL → ⚠ PARCIAL | else → ❌ {RESULT}.

Write `{QA_AGENT_DIR}/{reports_output_dir}/{TIMESTAMP}-{FEATURE}-web.md`:

```markdown
# QA Web Drive Report — {FEATURE}
**Fecha:** {TIMESTAMP} | **Duración:** {DURATION}s | **Resultado:** {RESULT_EMOJI}
**App:** {APP_PACKAGE} v{APP_VERSION} | **Renderer:** html | **Base URL:** {base_url}
**Modo:** {MODE_LABEL}

## Objetivo
{one sentence describing the validated flow}

## Pasos ejecutados
| # | Spec | Test | Resultado | Duración |
|---|------|------|-----------|----------|
{parse from Playwright list-reporter output, one row per test}

{if RESULT != PASS:}
## Diagnóstico

**Error principal:** {first non-stack line matching ^Error|^Failed|expect.*to|TimeoutError}

**Stack relevante (top 5):**
```
{filter Playwright stack — keep frames from {web_tests_dir}/ first, drop node_modules}
```

**Capturas:** {list of attached screenshots from playwright-report/, with relative paths}

**Trace:** abrir con `npx playwright show-trace {QA_AGENT_DIR}/{web_tests_dir}/test-results/.../trace.zip`
{end if}

{if MUTANT:}
## Datos creados en backend
{list any POST/PUT/DELETE endpoints called — extract from .spec.ts}
{end if}

## Pendientes para corrección
{one actionable sentence per failed test, or "Sin pendientes."}
```

Print path to the report.

---

## Step Final — Teardown automático (NEW v1.5, B.10)

**Ejecutar siempre al final, incluso si Section E / F / Z fallaron.** No cambia el `RESULT`.

- Si `WE_INVOKED_BOOTSTRAP == true` (set en Step 0) → invocar `qa-flutter-bootstrap` vía Skill tool con `--down --caller=qa-web-drive`. Parsear stdout pero no cambiar el RESULT del run.
  - `QA_BOOTSTRAP_STATUS=DOWN` → log `[BOOTSTRAP] Teardown OK.`
  - `QA_BOOTSTRAP_STATUS=TEARDOWN_FAILED` → log warning prominente: `[BOOTSTRAP] Teardown failed; marker queda con teardown_failed:true. Investigar antes del próximo run.`
- Si `WE_INVOKED_BOOTSTRAP == false` → skip (composition: el outer caller posee el lifecycle).

End of skill execution.

## Section Z — Knowledge learn (always after Section F)

Invoke `qa-knowledge-manager learn` with `DRIVE_OUTPUT` as input. The manager will:
1. Scan for new error signatures not already in `qa-knowledge.yaml`.
2. In interactive mode: propose entries and ask the user to confirm/edit.
3. In `--auto` mode: append drafts under `[AGENT_ENTRIES_START]` flagged `registered_by: "agent"` and `enabled: false` for review.

This step is non-fatal — failure to learn does not change the run's RESULT.

---

## Operational directives — Iteration safety (NEW v1.6, R4)

Esta sección codifica las directivas operativas que protegen al modelo (y al usuario) de quedar atrapado en iteraciones improductivas. Son reglas **de proceso** que viven en la sesión del skill, no en código persistente.

### Hypothesis validation gate (R4)

**Origen:** en una sesión de validación interna se perdió mucho tiempo iterando sobre una hipótesis errónea de `fill()` antes de validar contra el comportamiento real. La directiva siguiente cierra ese patrón.

**Contrato:**

- **Estado mantenido durante la sesión del skill:**
  - `helpers_edits_without_progress`: contador entero, inicializado en `0` cuando el skill arranca.
  - `tests_passing_baseline`: set de nombres de tests que pasaron al menos una vez durante esta sesión.

- **Eventos que modifican el estado:**

  | Evento | Efecto |
  |---|---|
  | Editar `_helpers.ts` (template o custom del proyecto) | `helpers_edits_without_progress += 1` |
  | Reescribir un spec `.ts` sobre el mismo error tras una falla previa | `helpers_edits_without_progress += 1` |
  | Un test que estaba fallando ahora pasa | `helpers_edits_without_progress = 0`; agregar a `tests_passing_baseline` |
  | Un test nuevo (no en baseline) pasa por primera vez | `helpers_edits_without_progress = 0`; agregar a baseline |
  | El skill termina (Step Final) | Estado se descarta — la próxima invocación arranca limpio |

- **Threshold de disparo:** `helpers_edits_without_progress >= 3` → **PAUSA DURA**.

**Acción al disparar la pausa:**

1. **Detener** cualquier flujo de iteración en curso. No proponer un fix más. No editar otro archivo. No correr otro test.

2. **Imprimir bloque diagnóstico:**

```
─── HYPOTHESIS VALIDATION GATE (R4) ───
3 ediciones consecutivas de _helpers.ts (o reescrituras del spec) sin que ningún test que fallaba ahora pase.

Diff acumulado de las últimas 3 ediciones:
{salida de `git diff HEAD~3 -- _helpers.ts` o equivalente; si no hay git, mostrar resumen textual de cada cambio}

Estado de tests:
  Pasando (baseline):    {lista de tests en tests_passing_baseline}
  Fallando ahora:        {lista de tests que fallaron en el último run}
  Nunca pasaron:         {lista de tests que nunca fueron observados PASS}

Hipótesis ensayadas (resumen 1 línea por edición):
  1. {descripción 1}
  2. {descripción 2}
  3. {descripción 3}

DETENIDO. Antes de continuar, el usuario debe elegir:
  (a) Continuar con la misma línea de hipótesis (justificar por qué este intento es diferente)
  (b) Cambiar de enfoque (proponer hipótesis alternativa con razonamiento)
  (c) Validar manualmente (correr el spec en `--headed` con DevTools abiertos)
  (d) Consultar references/canvaskit-fill-strategy.md y K013-K016 antes de seguir
  (e) Smoke MCP de 10-15 min antes de iterar más (Playwright MCP exploratorio)
      → ver references/mcp-exploration-companion.md
─── END R4 GATE ───
```

3. **Esperar input explícito del usuario.** No reanudar bajo ningún concepto sin que el usuario indique qué camino tomar. **Esta es una regla dura.** No vale "el usuario probablemente quiere que siga" — el costo de pausar es bajo (un mensaje), el costo de no pausar fue alto en una sesión de validación interna.

4. **Tras input del usuario:** resetear `helpers_edits_without_progress = 0` (el ciclo arranca de nuevo con la nueva hipótesis). Si el usuario elige (d), antes de continuar, leer el archivo de referencia y/o las K* completas. Si elige (e), el usuario opera Playwright MCP en su sesión actual de Claude Code para mapear el a11y tree real de la pantalla bloqueante; el smoke MCP no consume tiempo del driver — solo informa la siguiente hipótesis.

### Interacción con Section E.1

- Section E.1 dispara en cada falla (auto-match contra knowledge).
- R4 gate dispara cada 3 ediciones sin progreso.
- Las dos son **complementarias**: E.1 ofrece soluciones conocidas inline; R4 detiene el loop cuando E.1 (o las hipótesis propias) no resuelven.

### Diferencia con R6 (early-abort hard)

R6 está pendiente para v1.7. R6 sería un kill duro sin opción de continuar (después de N fallos consecutivos del mismo test). R4 es más informado: pausa con contexto y devuelve control al usuario. R4 cubre el caso más común (modelo iterando sobre hipótesis) sin la dureza de R6.

### Cuándo NO aplica el gate

- Iteraciones sobre **un spec recién creado** que nunca pasó — esperable que requiera varios intentos. Solo cuenta como "edición sin progreso" cuando hay tests previamente pasantes que no mejoran y el spec actual sigue fallando con el mismo error.
- Cambios en archivos ajenos a `_helpers.ts` y specs (ej. ajustar `playwright.config.ts`, agregar `Semantics()` en código Dart) — no incrementan el contador.
- Pausa por parte del usuario (ej. Ctrl-C, /goal change) — el contador se conserva pero no se dispara el gate hasta el siguiente intento.

---

## When NOT to use

- The project targets Android — use `qa-flutter-manual-runner` (flutter_drive, recommended default) or `qa-flutter-android-runner` (Appium).
- You only need analyze + build + manual checklist — use `qa-flutter-web-runner`.
- You need unit / widget / state tests — use `qa-flutter-unit-generator`.
- The Flutter app is configured with CanvasKit and you cannot switch — skill aborts at Step 3d. Either switch to HTML renderer (see K011), or fall back to manual QA via `qa-flutter-web-runner`.
- The widget tree has no `Semantics(identifier:)` instrumentation and you are not willing to add it — skill warns at Step 3e but proceeds with brittle role/text selectors; expect flakiness.

## Common mistakes

| Mistake | Fix |
|---|---|
| Asumir que `--web-renderer html` está disponible | Esa flag fue removida en Flutter 3.29. CanvasKit es el único renderer. Esta skill está alineada — no la uses en el comando de build. |
| Asumir que `Semantics(identifier: 'X')` se serializa al DOM | NO. El `identifier` solo lo usa `SemanticsTester` interno. Usa la estrategia de K012: role + hasText para botones, aria-label para inputs. |
| Olvidar `ensureSemantics()` en main() de la app | Sin esto, el a11y tree está vacío y ningún selector encuentra nodos. Pre-requisito hard, no negociable. Step 3e lo detecta y aborta. |
| Usar Playwright `.click()` sobre `flt-semantics[role="button"]` | NO dispara `onPressed` de Flutter. Usa `tapButton(page, '...')` del helper, que despacha la cadena `pointerdown/mousedown/pointerup/mouseup/click` desde JS. |
| Running multiple Playwright tests in parallel against the same dev server | Flutter dev server (`flutter run`) puede racear. Si usas el bootstrap nuevo (`flutter build web` + http.server), `workers > 1` funciona sin race. |
| Relying on `await page.waitForTimeout(...)` for "Flutter to settle" | Brittle. Espera por un atributo que debe existir post-acción: `await buttonByText(page, 'Confirmar').waitFor({ state: 'attached' })`. |
| Persisting a generated test that never passed | Section C.6 discards on fail — do not override. A broken test in `web-tests/` pollutes the regression suite. |
| Calling this skill without Section 3a knowledge preflight | The preflight applies known environment fixes; skipping it costs a debug cycle every time. |
| Treating `flutter drive -d chrome` as equivalent | It is not. See K011 — long flows fail there with hit-test offsets falling on the Navigator Overlay. This skill exists because of that. |

## Decision — vs `qa-flutter-web-runner`

| | `qa-flutter-web-runner` | `qa-flutter-web-driver-runner` |
|---|---|---|
| What it produces | Markdown checklist + build status | Executed `.spec.ts` + reports + screenshots |
| Renderer required | Any | HTML only |
| Needs Node/Playwright | No | Yes |
| Needs `Semantics(identifier:)` in app | No | Strongly recommended |
| When to use | Post-implementation smoke, manual QA prep | Regression, CI gate, repeatable E2E |
| Speed | Seconds | Tens of seconds to minutes per spec |
