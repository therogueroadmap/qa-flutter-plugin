# QA Test Plan Template

This is the canonical template that `qa-flutter-test-planner` substitutes when writing `qa-plugin-config/qa-plans/<feature>.md`. Placeholders are wrapped in `{ }`. Optional sections are marked `{IF ... }` / `{END IF}`.

The template MUST stay backward-compatible: the runners parse the YAML frontmatter and the section headings (`## 1.`, `## 2.`, ...) by exact match. Free-form text inside each section is preserved verbatim and never parsed by the runners.

---

```markdown
---
feature: {FEATURE}
version: {PLAN_VERSION}
generated_at: {TIMESTAMP}
generator: {GENERATOR}
platform: {PLATFORM}
android_stack: {ANDROID_STACK_OR_NULL}
data_mutating: {true|false}
blocker_risks: {N}
---

# Test Plan: {FEATURE}

## 1. Scope

{One paragraph describing what this feature does, derived from the top-scoring page's class doc-comment if present, else from the page class name and main provider name.}

**Pantallas detectadas:** {N} | **Flujos:** {happy} happy + {error} error | **Endpoints MUTANT:** {N}

## 2. Pantallas a cubrir

| # | Pantalla (clase) | Archivo | Entry route | Próximas rutas | Notas |
|---|---|---|---|---|---|
{For each screen in SCREENS, sorted by score desc:}
| {i} | `{page_class}` | `{page_file}` | `{entry_route}` | {next_routes joined by ", " or "—"} | score {score}{if MUTANT}, ⚠ MUTANT{end if} |

## 3. Precondiciones / datos requeridos

{Group by category. Use `[ ]` for blockers and `[~]` for advisory.}

### 3.1 Variables de entorno
{For each env var detected:}
- [ ] `{VAR_NAME}` — {context: where it's read, e.g. "leído por LoginPage vía String.fromEnvironment"}

### 3.2 Backend
- [ ] `backend.test_url` configurado en `qa-agent.yaml` (actual: `{BACKEND_TEST_URL or "❌ vacío"}`)
- [ ] `API_BASE_URL` en `qa-plugin-config/.env` coincide con `backend.test_url`
{If data_mutating:}
- [ ] Backend corriendo en perfil de **test** (datos descartables) — esta feature crea/modifica datos
{End if}
- [ ] Backend alcanzable: `GET {BACKEND_TEST_URL}/health` responde 200

### 3.3 Usuarios y datos seed
- [ ] Usuario de prueba con credenciales válidas (`TEST_EMAIL` / `TEST_PASSWORD`)
{For each seed requirement detected:}
- [~] {description: e.g. "Lista de transacciones con al menos 1 item para que la pantalla no muestre estado vacío"}

### 3.4 Dispositivo
{If PLATFORM == "android":}
- [ ] Dispositivo o emulador conectado (`adb devices` muestra `device`)
{For each Android permission detected:}
- [ ] Permiso `{permission}` concedido en el AVD (sino el flujo {affected_flow} fallará)
{End if}
{If PLATFORM == "web":}
- [ ] Chrome instalado y `flutter build web` funciona
- [ ] **`main()` llama `WidgetsFlutterBinding.ensureInitialized().ensureSemantics()`** antes de `runApp()` *(sin esto el a11y tree está vacío y Playwright no encuentra ningún widget — ver K001)*
{End if}

### 3.5 Dependencias nativas
{For each native dep detected from pubspec.yaml:}
- [~] `{package_name}` requiere setup adicional: {one-line note}

## 4. Flujos end-to-end

{For each flow with type=happy:}
### 4.{i}. {flow_name} (happy path)
{If data_mutating:}
> ⚠ Este flujo crea/modifica datos en el backend (MUTANT).
{End if}

**Pasos:**

> **v1.5+**: cuando `platform: web`, cada step lleva un marker `[NAV] / [INTERACT] / [INPUT] / [UNKNOWN]` derivado por el planner (Step 4.W). El driver (Section C.3) consume estos markers para generar el código Playwright correcto:
> - `[NAV]` → tap + `waitForFunction` (espera navegación).
> - `[INTERACT]` → tap + `waitForTimeout` (scroll/modal/etc).
> - `[INPUT]` → `fillByAriaLabel(label, value)`.
> - `[UNKNOWN]` → el spec emite TODO comment; el humano edita el plan a `[NAV]` o `[INTERACT]`.

{Numbered list of steps from flow.steps[]:}
{If PLATFORM == "web":}
1. [INTERACT] Abrir app → pantalla {first_screen} (auto al arrancar)
2. [INPUT] Ingresar email `'{TEST_EMAIL}'` en campo con label `'{labelText}'`
3. [NAV] Tapear botón `'{visible_text}'` → navega a {next_screen}
{Else:}
1. {step description, e.g. "Abrir app → pantalla LoginPage"}
2. {step description, e.g. "Ingresar email '{TEST_EMAIL}' en campo `Key('login.email')`"}
{End if}

{End for}

{If any error flow exists:}
### 4.E. Casos de error / borde

{For each flow with type=error:}
- **{flow_name}**: {one-line description, e.g. "Submit del formulario con email vacío → debe mostrar 'Email requerido'"}
{End for}
{End if}

## 5. Criterios de aceptación

| Flujo | Resultado esperado verificable |
|---|---|
{For each criterion in CRITERIA:}
| {flow_name} | {expected_result} |

## 6. Riesgos / fuera de scope

{If RISKS is empty:}
Sin riesgos identificados.
{Else:}
{For each risk:}
- **{severity emoji}** {description}
{End for}

**Severidad:**
- 🔴 Blocker: debe resolverse antes de ejecutar el plan
- 🟡 Warning: el plan se puede ejecutar pero el resultado puede ser parcial
- 🔵 Info: nota informativa, no afecta ejecución
{End if}

---

<!--
QA_PLAN_METADATA — los runners parsean estas líneas para optimización; no editar a mano.
plan_schema_version: 1
machine_summary: {"screens": N, "happy_flows": N, "error_flows": N, "blockers": N}
-->
```

---

## Substitution conventions

- All placeholders use `{snake_case}` or `{ALL_CAPS_FOR_CONSTANTS}`.
- Lists rendered with `{For each X:}` ... `{End for}` are 1-indexed when numbered.
- `{If condition:}` / `{End if}` blocks are stripped entirely if condition is false.
- Emojis in severity: 🔴 = blocker, 🟡 = warning, 🔵 = info.
- Frontmatter `data_mutating: true` if any flow has `data_mutating: true`.

## Why this format

- **Frontmatter is YAML** so runners can parse it with the same yaml library they already use for `qa-agent.yaml`.
- **Sections are numbered with stable headings** (`## 1.`, `## 2.`, ...) so runners locate them by `^## N\.` regex without depending on free-form section names.
- **Tables for screens / criteria** because they are structured and parseable; **markdown lists for steps** because they're human-edited.
- **Checkboxes (`[ ]`, `[~]`) for preconditions** so a human can tick them off in interactive review and a runner can refuse to start while blockers remain unchecked.
- **HTML comment metadata at the bottom** is a future hook for richer machine summaries without breaking the human-readable layout.
