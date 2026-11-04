# Changelog

All notable changes to this project will be documented in this file.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.6.3] — 2026-05-23

Mejora documental tras una sesión de validación interna del uso de Playwright MCP. La sesión confirmó el enfoque: Playwright MCP como herramienta exploratoria del developer/QA, ciclo `.spec.ts` del driver para regresión. Este patch documenta el contrato y abre una opción explícita en el R4 gate para invocarlo.

### Added

- **`skills/qa-flutter-web-driver-runner/references/mcp-exploration-companion.md`** — guía nueva sobre cómo usar Playwright MCP como complemento exploratorio del driver. Contiene:
  - Matriz "MCP vs driver" según situación (pantalla nueva / R4 disparado / CI gate / bug ad-hoc).
  - Workflow recomendado de 10-15 calls para mapear 1-2 pantallas.
  - Mapeo `hallazgo MCP → K# del knowledge base` (e.g. textbox sin aria-label → K015).
  - Métrica de costo orientativa medida en una sesión de validación interna (referencia para decidir si vale la pena integrar wrappers MCP en el futuro).
  - Anti-patrones (no usar MCP para regresión, no usar driver para exploración inicial).

### Changed

- **`SKILL.md` Operational directives R4 gate** — agrega opción `(e) Smoke MCP de 10-15 min` al diagnóstico del gate. Cuando el modelo lleva 3 ediciones del `_helpers.ts` sin progreso, ahora puede pausar a operar Playwright MCP contra la pantalla bloqueante en vez de seguir iterando a ciegas. Detalle del contrato en `references/mcp-exploration-companion.md`.
- **`references/canvaskit-fill-strategy.md`** — sección "Cuándo este archivo NO basta" enumera el caso "pantalla nueva sin a11y tree conocido", con pointer al companion. Refuerza que un smoke MCP upfront ahorra ~3 iteraciones del helper en casos no triviales.

### Confirmed (validación empírica)

Una sesión de validación interna reprodujo K011 (`<flutter-view> intercepts pointer events`), K013 (fill no commitea sin focus+wait+pressSeq+blur) y el caso teléfono sin aria-label (K015) contra una app Flutter web de prueba. **Las mitigaciones ya existentes en el `_helpers.ts` del driver son las correctas** — Playwright MCP las **detecta más rápido** que el ciclo spec.ts (1 vuelta vs 5 runs históricos), pero no las reemplaza.

### Migration desde v1.6.2

Cero acciones requeridas. Ningún helper, knowledge entry ni contrato del runner cambió. Esto es **mejora documental** + ampliación del R4 gate. Si el usuario quiere aprovechar el companion, debe instalar Playwright MCP user-level (instrucciones en el nuevo doc); el driver sigue funcionando sin él.

### No cambia

- `templates/qa-knowledge.yaml` — K011-K016 sin tocar.
- `_helpers.ts` template — sin cambios.
- Cualquier flujo de bootstrap, runner, planner, release-gate, learn — sin cambios.

---

## [1.6.2] — 2026-05-21

Patch crítico tras segunda validación runtime. El run reveló que **los patterns de v1.6.1 no matcheaban el error real** aunque conceptualmente debían: K013 reportó "sin matches" frente a un `TimeoutError: locator.fill ... waiting for locator('input[aria-label="Máximo por transacción (EUR)"]')` que es exactamente el caso "fill silencioso → timeout en input".

### Root cause

Los regex usaban `.*` que en la mayoría de engines (JavaScript, Python sin flag DOTALL, Node) **no matchea newlines** por default. Los stack traces de Playwright reparten la firma en 3 líneas (`TimeoutError:` / `Call log:` / `waiting for locator(...)`), entonces `TimeoutError.*locator.*input\[aria-label` falla porque el primer `.*` no puede cruzar el `\n`.

Validación empírica del bug (Node):
```
K013 (v1.6.1) match contra error real: false  ← bug
K013 (v1.6.2) match contra error real: true   ← fixed
```

### Fixed

- **K013** — `.*` → `[\s\S]*` en las 3 alternativas multi-elemento del pattern. Cross-line safe en cualquier engine.
- **K014** — `.*` → `[\s\S]*` en las 3 alternativas multi-elemento (`hasText timeout`, `TimeoutError flt-semantics button`, `tapButton Timeout`).
- **K015** — `.*` → `[\s\S]*` en todas las alternativas multi-elemento. Mantiene la restricción `(?:after|tras).*fillUnlabeled` en la alternativa indirecta porque K013 cubre el caso general; K015 sólo dispara cuando el stack menciona explícitamente fillUnlabeled.
- **K016** — sin cambios (pattern de una sola línea, no afectado).

### Changed (Section E.1)

- **Instrucción explícita sobre flag DOTALL** y advertencia contra "match mental". Cuando Node esté disponible (Step 3b ya lo valida), el SKILL.md sugiere invocar un snippet inline para compilar el regex con `'is'` flag (case-insensitive + dotall) y aplicar contra el `DRIVE_OUTPUT`. Esto previene el falso negativo observado en v1.6.1.

### Lección aprendida

Los patterns de auto-match son **datos ejecutables**, no documentación. Confiar en match mental del LLM es propenso a falsos negativos en outputs multi-línea. v1.6.2 codifica el principio: regex se valida con regex engine real, siempre que sea posible.

### Migration desde v1.6.1

Cero acciones requeridas. Los patterns del template del plugin se cargan automáticamente la próxima vez que se invoque el driver con los nuevos regex cross-line safe.

---

## [1.6.1] — 2026-05-20

Patch tras validación runtime de v1.6.0 contra una app Flutter web de prueba. El run identificó dos limitaciones del auto-match de Section E.1 que dejaban dormida la mitad del valor de v1.6.

### Fixed (A — auto-match consulta AMBOS knowledge files)

- **Section E.1 ahora une AMBOS knowledge files** (proyecto + plugin template) con dedup por `id` (proyecto gana en colisión). Antes el contrato decía "proyecto, fallback al plugin" → si el proyecto tenía `qa-knowledge.yaml` con sus entries learn-eadas (K001-K003 en el proyecto), las K013-K016 del template del plugin **nunca se evaluaban**.
- **Visibility logs** muestran cuántas entries vienen de cada source (`count_project`, `count_plugin_only`). Cada match imprime `source: project` o `source: plugin` para que el usuario sepa dónde modificar la entry si hace falta.
- **Section E.1 imprime una línea de visibility incluso cuando no hay matches** (`Auto-match consultó N entries — sin matches`) en vez de skip silencioso, así el usuario confirma que la sección corrió y tiene cobertura.

### Changed (B — patterns indirectos K013/K014/K015)

Los patterns originales matcheaban el síntoma inmediato del helper (ej. "no textbox found") pero no el síntoma indirecto cuando el helper aparentaba funcionar pero el value no se commiteaba (ej. timeout esperando el input del paso siguiente del wizard). El caso observado fue exactamente este: `fillUnlabeledTextbox` no escribió, el form quedó con un mensaje de campo obligatorio en el campo de teléfono, el siguiente locator timeouteó. Ninguna K# matcheó.

- **K013** — pattern agrega `TimeoutError.*locator.*input\[aria-label.*\*?\"\]`. Cubre "el helper aparentó funcionar pero el siguiente step espera un input que nunca aparece". Fallback_message extendido con sección "Si este match fue INDIRECTO" con 3 hipótesis a validar.
- **K014** — pattern agrega `TimeoutError.*flt-semantics\[role=.button|tapButton.*Timeout`. Cubre "el tap aparentó funcionar pero onPressed no disparó". Fallback con guía adicional para distinguir [INTERACT] vs [NAV] del plan.
- **K015** — pattern agrega `(?:[Oo]bligatori[oa]|[Rr]equired).*(?:Tel[eé]fono|Phone)|fillUnlabeledTextbox.*Timeout` y el TimeoutError correlacionado. Cubre exactamente el síntoma observado en el wizard de prueba. Fallback con 3 hipótesis específicas: selector agarró otro nodo, foco se perdió antes de keyboard.type, inputFormatter rechaza el value.
- **K016** — sin cambios (autofill es específico, no necesita variante indirecta).

### Referencias

- Run que motivó el patch: un reporte de validación runtime (sección "Knowledge auto-match" reportó "Ninguna entrada matcheó" porque K013-K016 no se evaluaban).

### Migration desde v1.6.0

Cero acciones requeridas. El cambio es transparente — la próxima invocación del driver evalúa automáticamente AMBOS knowledge files con los patterns ampliados.

---

## [1.6.0] — 2026-05-20

Iteration safety: cierra los 3 gaps operativos críticos identificados en una sesión de validación interna que NO eran de modelo de selectores (ya cubiertos en v1.5.1 con K013-K016) sino de **proceso de iteración**. Sin cambios en runtime de Playwright, helpers TypeScript ni knowledge entries.

### Added

- **Section E.1 — Post-failure knowledge auto-match ** — tras `RESULT != PASS`, el driver compara `DRIVE_OUTPUT` contra el `pattern` regex de cada entry K001-K016 en `qa-knowledge.yaml`. Si hay match, imprime el `fallback_message` literal ANTES de Section F. Convierte las knowledge entries de documentación pasiva en alerta activa al fallar.
- **Operational directives — Iteration safety (R4)** — hypothesis validation gate al final del SKILL.md del driver web. Counter `helpers_edits_without_progress` mantenido durante la sesión. Tras 3 ediciones consecutivas de `_helpers.ts` sin que ningún test que fallaba ahora pase, el skill PAUSA con dump diagnóstico (diff acumulado + estado de tests + hipótesis ensayadas) y **NO reanuda sin input explícito del usuario**. Protege contra el patrón de pérdida de tiempo iterando sobre una hipótesis no verificada.
- **`skills/qa-flutter-web-driver-runner/references/canvaskit-fill-strategy.md` (R5)** — decision tree consolidado: cuándo usar `fillInput` / `fillUnlabeledTextbox` / `tapButton` / `lockInputValue`. Matriz de primitivas. Anti-patrones. Referenciado desde Step 3, Section C.0 y Section C.4 del SKILL.md como required reading antes de editar helpers o generar specs con inputs/botones.

### Why this scope (subset of the originally planned scope)

El bundle de iteration safety y portabilidad anunciado en v1.5.1 incluía 8 items (B.1, B.4–B.9, directivas meta). El subset implementado en v1.6.0 son los 3 items que protegen contra el patrón de pérdida de tiempo observado: knowledge dormido, iteración sobre hipótesis no verificada, y dispersión del criterio de selección. Los items restantes (templates externalizados, storageState, JSON artifact, portabilidad Windows, retry policy, R6/R7) quedan para v1.7+.

### Migration desde v1.5.1

Cero acciones requeridas. Las directivas nuevas (auto-match, hypothesis gate) son del flujo del skill — operan automáticamente desde la próxima invocación de `/qa-web-drive`. El archivo `references/canvaskit-fill-strategy.md` se consume al ejecutar el skill.

### Referencias


- Origen empírico: una sesión de validación interna.

---

## [1.5.1] — 2026-05-20

Hotfix de modelo de selectores tras la validación contra una app Flutter web de prueba. El `fillByAriaLabel` del v1.5.0 (solo `input.fill()`) falla en wizards con múltiples campos y dropdowns porque Flutter no bindea el input permanente al `TextEditingController` hasta recibir foco — el `fill()` se va sin attach. Se descubrieron también gaps en la búsqueda de botones (textContent vs aria-label), inputs sin label, y autofill de Chrome.

### Changed (reescritura de helpers)

- **`fillInput(page, label, value)` (NUEVO, reemplaza `fillByAriaLabel`)** — aplica `click({ force: true })` antes del `fill()` para attachear el text editing client de Flutter al input permanente. Sin esto el valor no llega al `TextEditingController` en flujos no-triviales. `fillByAriaLabel` queda como alias de `fillInput` por back-compat. **K013**.
- **`tapButton(page, text)` (REESCRITO)** — búsqueda dual: matchea por `textContent` O `aria-label`. Antes solo examinaba `textContent`; fallaba con botones de ícono, dropdowns sin selección, y PopupMenuItems. También acepta `flt-semantics[role="menuitem"]` además de `[role="button"]`. **K014**.
- **`fillUnlabeledTextbox(page, value)` (NUEVO)** — para inputs sin `aria-label` (típico de teléfonos sin labelText). Estrategia: blur + `dispatchEvent` chain sobre el primer `flt-semantics[role="textbox"]` sin `aria-label` + `keyboard.type(value)`. **K015**.
- **`lockInputValue(page, label, value)` (NUEVO)** — congela el getter del input para que Chrome autofill (en modo `--headed`) no pise el value tras `fillInput()`. Útil en tests de validación que escriben values inválidos a propósito. **K016**.
- **`buttonByText(page, text)` (sin cambios pero documentado como legacy)** — solo para inspección, no para tap; usar `tapButton` que cubre `aria-label`.

### Added (knowledge entries)

- **K013** — `templates/qa-knowledge.yaml`: fill strategy click-prime + fill.
- **K014** — Flutter CanvasKit button text dual search (textContent + aria-label).
- **K015** — Unlabeled textbox: dispatch + keyboard.type.
- **K016** — Chrome autofill interference (headed mode) + lockInputValue workaround.
- **README tabla de selectores** expandida con referencias K# explícitas.

### Why not v1.6.0

Cambio acotado a `_helpers.ts` del driver SKILL (template) + knowledge entries + docs. No introduce arquitectura nueva ni feature; cierra un gap de modelo descubierto en runtime real. `1.6.0` queda reservado para el siguiente bundle (templates externalizados, storageState, JSON artifact, portabilidad Windows, ubicación node_modules, retry policy, directivas meta del plugin para validación de hipótesis y early-abort).

### Migration desde v1.5.0

- **Tests/specs generados por el driver v1.5.0** siguen funcionando porque `fillByAriaLabel` es alias de `fillInput` en v1.5.1. Recomendado: regenerar el spec con `/qa-web-drive {feature}` para que use `fillInput` explícito y los helpers nuevos.
- **`_helpers.ts` ya personalizado en `web-tests/`** queda intacto (el driver NO sobrescribe desde v1.4.1). Si quieres adoptar los helpers nuevos, copia el template del driver SKILL.md Section C.4 a tu archivo manualmente, o bórralo para que el driver lo regenere.

---

## [1.5.0] — 2026-05-20

Madurez del driver web post-validación runtime del v1.4.x. Cierra 4 gaps con respaldo de evidencia empírica contra una app Flutter web de prueba: UX standalone, detección temprana de problemas estructurales, planner alineado a CanvasKit, validación de plans en el driver.

### Added

- **`/qa-web-drive` standalone con bootstrap automático (B.10)** — el driver invoca `qa-flutter-bootstrap --up/--down` por dentro cuando es invocación raíz (caller = `qa-web-drive`). Composition rule vía marker file: si existe marker con caller distinto, skip Step 0 (outer caller posee lifecycle). Flag `--no-bootstrap` opt-out.
- **Pre-flight a11y probe (B.8)** — Step 3e del driver ahora ejecuta un mini Playwright headless (~5-8s) antes del spec del usuario: abre `base_url`, espera `flt-glass-pane`, cuenta `flt-semantics`. Si count == 0 → abort con fix exacto (sin esperar 20s timeout). Flag escape hatch `--skip-preflight-probe`.
- **Planner híbrido para `platform: web` (B.2)** — `qa-flutter-test-planner` agrega Step 4.W que enumera widgets con labels visibles (`TextFormField.labelText`, button child/label) y aplica heurísticas regex a cada `onPressed`/`onTap` para emitir markers `[NAV]` / `[INTERACT]` / `[INPUT]` / `[UNKNOWN]` en Section 4 del plan. Profundidad max 1 al seguir callbacks a funciones nombradas.
- **Driver valida plan antes de generar spec (B.3)** — nueva Section C.0.1 del driver: rechaza `Semantics(identifier:)` references, identifiers con dots; warna sobre `Key('...')`, UNKNOWN markers sin resolver, ausencia de `ensureSemantics()` precondition.
- **Section 3 del plan-template** incluye precondition `ensureSemantics()` como blocker cuando `platform: web`.
- **`preconditions-checklist.md`** agrega fila `ensureSemantics() en main` como blocker para web.
- **README** sub-secciones: "Invocación standalone con bootstrap automático (v1.5+)", "Pre-flight a11y probe (v1.5+)", "Markers en plans web (v1.5+)".

### Changed

- **`qa-flutter-web-driver-runner` Section C.3** — ahora consume markers `[NAV]/[INTERACT]/[INPUT]/[UNKNOWN]` del plan para emitir código Playwright correcto. Plans pre-v1.5 sin markers se tratan como `[INTERACT]` por default (back-compat con warn).
- **`qa-flutter-web-driver-runner` Step 3e** — degradado el chequeo de a11y de "delegar al primer test" a "Playwright probe propio en pre-flight". Más rápido al fail (5-8s vs 20s) y mensaje específico.
- **`commands/qa-web-drive.md`** — `argument-hint` extendido con `--no-bootstrap` y `--skip-preflight-probe`.
- **Plugin manifest** — `1.4.2` → `1.5.0`.

### Migration desde v1.4.x

| Situación | Acción |
|---|---|
| Proyecto con flujo v1.4.2 funcionando (ej. una app Flutter web de prueba) | Ninguna. `--no-bootstrap` es opcional; plans pre-v1.5 sin markers se tratan como `[INTERACT]` (back-compat). |
| Plans pre-v1.5 con steps en formato libre | Funcionan. El driver warna que markers están implícitos; recomienda regenerar el plan con `/qa-plan {feature}` para markers explícitos. |
| Proyectos que invocan `/qa-web-drive` standalone (sin orquestador) | El driver ahora levanta bootstrap automático. Si el server ya está arriba a mano, agregar `--no-bootstrap` o limpiar el marker antes. |
| Plans con `Semantics(identifier:)` mencionado | El driver Section C.0.1 va a rechazarlos (hard reject). Regenerar el plan con `/qa-plan {feature}` (planner v1.5+ emite K012 strategy). |

### Roadmap

- **v1.5.0 — Madurez del driver web (parcial)** ✅ cerrado en este release.
- **v1.6+ (futuro)**: templates externalizados (B.4), `storageState` reutilizable (B.5), JSON artifact del driver (B.6), portabilidad Windows-nativo (B.7), contrato de ubicación de `node_modules` (B.1), retry policy (B.9).

---

## [1.4.2] — 2026-05-20

Cierre del gap operacional descubierto durante validación de v1.4.1 contra una app Flutter web de prueba. El bootstrap construía con `flutter build web` puro, sin propagar los `--dart-define-from-file` que la app necesita para apuntar al backend local (en lugar de producción). Resultado: el wizard quedaba bloqueado en "Cargando..." esperando respuesta del backend. **K002 (auto-registrado por el agente) ahora se resuelve declarativamente** en el yaml.

### Added

- **`autonomous.web.build_flags: [string]`** — array de flags que el bootstrap pasa literalmente a `flutter build web` durante U-W.3. Casos típicos:
  - `["--dart-define-from-file=dart_define/dev.json"]`
  - `["--dart-define=API_BASE_URL=http://localhost:8080", "--dart-define=ENV=dev"]`
  - `["--no-tree-shake-icons"]`
  Default: array vacío (sin flags = comportamiento v1.4.1).
- **README → "Pre-validación de backend con `build_flags`"** — sub-sección nueva en "QA web con browser automation" con tabla de casos típicos y notas sobre multi-ambiente.

### Changed

- **`autonomous.web.build_cache` key** — pasó de `pubspec_lock_sha256` a `build_cache_key`. El nuevo valor es sha256 de `pubspec.lock + sorted(build_flags joined)`. Permite que cambios en `build_flags` (ej. dev → staging) invaliden el cache sin tocar `pubspec.lock` ni `rm -rf build/web`.
- **Marker schema** — campo `web.pubspec_lock_sha256` reemplazado por `web.build_cache_key`. Markers v1.4.x se consideran inválidos en v1.4.2 (no tienen el campo) y disparan rebuild en el primer `--up` post-upgrade.
- **Plugin manifest** — `1.4.1` → `1.4.2`.

### Why not v1.5.0

Este cambio es backward-compatible (sin `build_flags` declarados → comportamiento v1.4.1 intacto) y cierra un gap operacional encontrado en validación de v1.4.1, no introduce un cambio mayor de arquitectura. `1.5.0` queda reservado para el bundle de madurez del driver web (planner emite labels CanvasKit, validación de plan en driver, `storageState`, JSON artifact, portabilidad Windows, contrato de ubicación de `node_modules`, recovery patterns).

### Migration desde v1.4.0 / v1.4.1

Si tu proyecto usa `--dart-define-from-file` o `--dart-define=...` para builds locales:

```yaml
# qa-plugin-config/qa-agent.yaml
autonomous:
  web:
    build_flags:
      - "--dart-define-from-file=dart_define/dev.json"
      # o las flags que sea que pases a `flutter build web` a mano
```

Sin esto, el primer `--up` post-upgrade rebuilea sin flags (igual que v1.4.1).

---

## [1.4.1] — 2026-05-20

Hotfix de modelo: el v1.4.0 (y la versión previa del skill) asumía que `flutter run -d chrome --web-renderer html` era posible y que `Semantics(identifier: 'X')` se serializaba como `<flt-semantics id="X">`. Ambas asunciones son falsas en **Flutter 3.29+**: la flag `--web-renderer` fue removida (CanvasKit es el único renderer) y `Semantics(identifier:)` solo lo consume `SemanticsTester` interno, nunca llega al DOM. Adicionalmente, `page.click()` de Playwright sobre `flt-semantics[role="button"]` NO dispara `onPressed` de Flutter — hay que despacharlos con cadena `dispatchEvent()` desde JS. Hotfix descubierto empíricamente contra una app Flutter web de prueba.

### Changed (correcciones)

- **`qa-flutter-bootstrap` Section UP-WEB / U-W.3** — `flutter build web --web-renderer html` → `flutter build web` (la flag ya no existe).
- **`qa-flutter-web-driver-runner` Step 3d** — eliminado el check "Detect renderer / abort si CanvasKit". CanvasKit es el target.
- **`qa-flutter-web-driver-runner` Step 3e** — el check de "Semantics emission" ya no busca `flt-semantics[id=...]`. Ahora verifica que el a11y tree emite cualquier `flt-semantics` (signo de que `ensureSemantics()` fue llamado en `main()`).
- **`qa-flutter-web-driver-runner` Section C.3 (scaffold del spec)** — el ejemplo del spec generado usa `tapButton(page, '...')` y `fillByAriaLabel(page, '...', '...')` en lugar de `tapBySemantics` / `fillText` por identifier.
- **`qa-flutter-web-driver-runner` Section C.4 (helpers)** — `_helpers.ts` reescrito alineado a CanvasKit: `openApp`, `buttonByText`, `tapButton` (con dispatchEvent chain), `fillByAriaLabel`, `takeStepShot`. **El skill ya no sobrescribe `_helpers.ts` existente** (proyectos con helpers personalizados quedan intactos).
- **`qa-knowledge.yaml` → K011** — la mitigación #2 deja de sugerir `--web-renderer html`; redirige a K012.
- **`qa-knowledge.yaml` → K012 (NUEVO)** — patrón de uso que documenta la estrategia de selectores correcta para Playwright + Flutter web CanvasKit (role + hasText + dispatchEvent + aria-label + flt-text-editing-host).
- **README "QA web con browser automation"** — quick start actualizado: pre-requisito `WidgetsFlutterBinding.ensureInitialized().ensureSemantics()` en `main()`, tabla con estrategia K012, sección "Lo que NO funciona" con las asunciones obsoletas.
- **Plugin manifest** — `1.4.0` → `1.4.1`.

### No cambia

- Routing por `project.web.mode` (default `driver`) — sigue igual.
- Bootstrap U-W.1, U-W.2, U-W.4..U-W.7 — sin cambios.
- `commands/qa-web-drive.md` — sin cambios.
- `qa-stability-agent` Step 2/3 — sin cambios.
- Marker schema — sin cambios.

### Compatibilidad

- **Flutter < 3.29 (HTML renderer todavía disponible)**: el v1.4.1 NO usa `--web-renderer html`, así que Flutter usa su default (auto-elige según target). Para esos proyectos sigue funcionando porque el flag no es ya requerido por el driver — la app emite a11y tree de la misma forma en HTML renderer.
- **Flutter ≥ 3.29 (solo CanvasKit)**: el v1.4.1 funciona; el v1.4.0 fallaba en U-W.3 con "unknown flag".

---

## [1.4.0] — 2026-05-19

Playwright integrado al flujo orquestado. El skill `qa-flutter-web-driver-runner` (presente desde antes de este release) ahora es ciudadano de primera clase: tiene slash command propio, `qa-stability-agent` lo rutea automáticamente, y `qa-flutter-bootstrap` levanta la app web por su cuenta. **Breaking change** — el default de routing para `platform: web` cambió de `qa-flutter-web-runner` (checklist) a `qa-flutter-web-driver-runner` (Playwright). Ver Migration.

### Added

- **`/qa-web-drive` slash command** — wrapper de `qa-flutter-web-driver-runner`. Acepta `<feature>`, `--auto`, `--plan=<path>`, `--headed`. Igual UX que `/qa-run`.
- **`project.web.mode` campo en `qa-agent.yaml`** — `"driver"` (default) o `"checklist"`. Decide qué runner usa `qa-stability-agent` para `platform: web`.
- **`autonomous.web` bloque en `qa-agent.yaml`** — `build_cache` (default `true`), `build_timeout_seconds` (default 180), `server_port` (default: parsea de `base_url`), `server_ready_timeout_seconds` (default 30).
- **`Section UP-WEB` / `Section DOWN-WEB` en `qa-flutter-bootstrap`** — corre `flutter build web --web-renderer html` (con cache por sha256 de `pubspec.lock`), levanta `python -m http.server` en el puerto extraído de `base_url`, hace polling de `base_url` hasta 200, mata el server limpio en teardown, preserva `build/web` para cachear el próximo run.
- **Bloque `web:` en el marker file** — campos `we_started_it`, `server_pid`, `port`, `build_cached`, `pubspec_lock_sha256`. Composition rule heredada (el outermost caller posee el lifecycle).

### Changed

- **`qa-stability-agent` Step 2** — `platform == "web"` ya no enruta hardcoded a `qa-flutter-web-runner`. Rutea según `project.web.mode` (default `driver` → driver, `checklist` → web-runner, cualquier otro valor → abort).
- **`qa-stability-agent` Step 3** — agrega rama para `qa-flutter-web-driver-runner` con `--plan=<path>` (consume planes con frontmatter `platform: web`).
- **`qa-flutter-bootstrap` Step 3 — Branch by MODE** — extendido para enrutar a Section UP-WEB / DOWN-WEB cuando `project.platform == "web"`.
- **`qa-flutter-bootstrap` regla de `pid_ward`** — ahora también considera `web.server_pid`.
- **`qa-flutter-web-driver-runner` Step 3c** — degradado de abort a warn (el bootstrap es source of truth de UP; este check queda como red de seguridad para detectar races).
- **`qa-flutter-web-driver-runner` Step 3b** — la instrucción de instalación de Playwright se mueve al README (sección nueva); SKILL.md solo referencia.
- **Plugin manifest** — versión bumpeada a `1.4.0`.

### Migration

Proyectos en `v1.3.x` con `project.platform: "web"`:

- **Si ya tienes Playwright instalado** y la app expone `Semantics(identifier:)` en los widgets críticos → ninguna acción. El próximo run aprovecha el flujo orquestado completo.
- **Si NO tienes Playwright instalado** y prefieres mantener el comportamiento checklist anterior:

  ```yaml
  # qa-plugin-config/qa-agent.yaml
  project:
    web:
      mode: "checklist"   # ← agregar esta línea para preservar v1.3.x
  ```

- **Para migrar al driver** desde checklist:

  ```bash
  cd {flutter_project} && npm init -y
  npm install -D @playwright/test
  npx playwright install chromium
  ```

  Luego envolver los widgets críticos con `Semantics(identifier: 'field-...')` (dashes, no dots). Confirmar que el build usa `--web-renderer html`. Ver "QA web con browser automation" en el README para el quick start completo.

### Roadmap

- **v1.4 — Playwright integrado al flujo orquestado** ✅ cerrado en este release.

---

## [1.3.1] — 2026-05-04

Centralize all QA configuration files under `qa-plugin-config/` in the target Flutter project. **Breaking change** — existing projects must migrate files (see Migration guide in README).

### Changed

- **All skills and agents** — config resolution updated from project root (`qa-agent.yaml`, `qa-knowledge.yaml`, `qa-plans/`, `qa-reports/`) to `qa-plugin-config/` subfolder. `.env` stays at the project root (it is a project-level file, not plugin-specific). Walk-up logic preserved: cwd → walk up to `pubspec.yaml` sibling → ask user.
- **`qa-flutter-bootstrap`** — introduces `PROJECT_ROOT` / `YAML_DIR` distinction. `start_cwd` resolves relative to `PROJECT_ROOT` (parent of `qa-plugin-config/`) for backward-compatible relative paths. Bootstrap marker path formula: `{PROJECT_ROOT}/{reports.output_dir}/.qa-bootstrap-marker`.
- **`qa-flutter-android-runner` / `qa-flutter-web-runner`** — `QA_AGENT_DIR` now points to `{PROJECT_ROOT}/qa-plugin-config`; Appium companion scripts expected at `qa-plugin-config/scripts/`.
- **`qa-flutter-manual-runner`** — `.env` read from project root (unchanged). `PLAN_DIR` default: `qa-plugin-config/qa-plans/`. Reports default: `qa-plugin-config/qa-reports/`.
- **`qa-flutter-test-planner`** — plan output default: `qa-plugin-config/qa-plans/<feature>.md`. `PLAN_DIR` default: `qa-plugin-config/qa-plans/`.
- **`qa-flutter-unit-generator`** — `REPORTS_DIR` default: `qa-plugin-config/qa-reports`.
- **`qa-flutter-release-gate`** — reads `qa-plugin-config/qa-agent.yaml`.
- **`qa-knowledge-manager`** — knowledge file path: `qa-plugin-config/qa-knowledge.yaml`.
- **`qa-stability-agent`** — all internal references updated to `qa-plugin-config/` paths.
- **`templates/qa-knowledge.yaml`** — install comment updated to `qa-plugin-config/`.
- **`commands/qa-plan.md`** — output path updated.
- **README** — Quick start, configuration, troubleshooting, and migration guide updated for new structure.
- **Plugin manifest** — version bumped to `1.3.1`.

### Migration

Projects upgrading from `≤ 1.30` must run:

```bash
mkdir -p qa-plugin-config
mv qa-agent.yaml    qa-plugin-config/
mv qa-knowledge.yaml qa-plugin-config/    # if present
mv qa-plans/        qa-plugin-config/     # if present
# Update reports.output_dir in qa-agent.yaml to: qa-plugin-config/qa-reports
# .env stays at the project root — do NOT move it
```

Then update `.gitignore`:
```
.env
qa-plugin-config/qa-reports/
```

---

## [1.30] — 2026-05-04

QA planning layer. Closes roadmap gap **G8** (planning artifact). Backward-compatible — runners keep their existing on-the-fly behavior when no plan is provided.

### Added

- **`qa-flutter-test-planner` skill** — produces an auditable, human-editable QA test plan at `qa-plans/<feature>.md` covering: scope, pantallas a cubrir, precondiciones / datos requeridos, flujos end-to-end (happy + error), criterios de aceptación, riesgos / fuera de scope. Markdown body with YAML frontmatter that downstream runners parse. Invokable as `/qa-plan <feature>` (interactive) or `/qa-plan <feature> --auto` (CI / scheduled refresh).
- **`/qa-plan` slash command** routing to the planner skill.
- **`planning:` block in `qa-agent.yaml`** — `enabled`, `test_plan_dir` (default `qa-plans/`), `require_plan`, `score_threshold`, `flow_depth`, `aliases`, `precondition_severity_overrides`. All optional.
- **References under the planner skill** — `plan-template.md` (canonical artifact format), `screens-discovery.md` (multi-layer semantic resolution playbook generalized from manual-runner § C.1), `preconditions-checklist.md` (canonical list of preconditions with detection sources and severity rules).

### Changed

- **`qa-flutter-manual-runner`** — `--plan=<path>` flag added. New Section C.0 reads the plan when provided and skips the on-the-fly resolution in C.1–C.2; C.3.2 uses plan flow steps as test step seeds.
- **`qa-flutter-android-runner`** — Step 3 uses plan's Section 2 (pantallas) as feature list and Section 4 (flujos) as test scenarios passed to the `qa-flutter-android-tester` sub-agent. Falls back to free-form parsing without `--plan`.
- **`qa-flutter-web-runner`** — Step 6 uses plan's Section 2 instead of git-diff analysis when `--plan` is provided. Step 7 checklist seeded from plan flows + preconditions.
- **`qa-stability-agent`** — new Step 2.5 resolves plans for each feature in the implementation summary. Auto-injects `--plan=<path>` to the routed runner. Honors `planning.require_plan: true` by aborting with NO-GO when plans are missing.
- **`commands/qa-run.md`** — accepts `--plan=<path>` and documents it.
- **README** — new "Planificación de QA" section documenting when to plan, configuration, plan vs. report distinction. Quick start updated to suggest `/qa-plan` first.
- **Plugin manifest** — version bumped to `1.30`.

### Roadmap

- **G8 — Planning artifact** ✅ closed in this release.

## [1.2.0] — 2026-05-02

Self-learning cycle (autoaprendizaje) and skill performance improvements.

### Added

- **`qa-knowledge-manager` skill** — autoaprendizaje cycle that wraps every QA run with two transparent steps: `preflight` (applies known environment fixes before the run) and `learn` (captures new errors after the run and updates `qa-knowledge.yaml` with user confirmation). Also exposes `register` (manual fix entry) and `report` (knowledge base audit) modes.
- **`templates/qa-knowledge.yaml`** — starter template for the knowledge file. Copy to project root and commit with the project; if absent, the orchestrator behaves exactly as v1.1.x.

### Changed

- **`qa-stability-agent` → v2.1** — adds Steps 0a (knowledge preflight) and 0b (knowledge learn) as non-fatal wrappers around the existing run. New flags: `--auto` (CI non-interactive), `--dry-run` (simulate plan without running tests), `--skip-preflight`, `--skip-learn`. Verdict semantics updated: `GO` / `NO-GO` / `CONDITIONAL` with exit codes 0 / 1 / 2 respectively. Fully backwards-compatible: without `qa-knowledge.yaml` the agent is identical to v2.0.
- **`qa-flutter-manual-runner`** — Grep-before-Read optimization in the semantic resolution phase; `pubspec.yaml` is now read once in Step 2 and reused in Sections C and E, eliminating redundant reads per feature.
- **`qa-flutter-unit-generator`** — Grep-before-Read filter applied before full file reads in Section A (repository/bloc/widget candidates), reducing unnecessary context load on large codebases.

## [1.1.1] — 2026-04-24

Usability patch — slash commands and CLI permission docs.

### Added

- **`commands/` directory** with four slash commands wrapping the most-used skills: `/qa-run`, `/qa-unit`, `/qa-release-gate`, `/qa-bootstrap`. Previously the skills could only be invoked via natural language or `claude -p` prompts — now they appear in the slash menu.
- **README — "CLI permissions in unattended mode"** section explaining the `.claude/settings.local.json` allowlist needed when running `claude -p` (non-interactive) so the orchestrator does not hang on the first denied tool call.

### Notes

- No skill or agent behavior changed in this release. Existing skills and tags continue to work; the slash commands are additive convenience.

## [1.1.0] — 2026-04-23

Autonomous orchestrator cohort. Closes roadmap gaps G1, G2, G4, G6 and upgrades report quality.

### Added

- **`qa-flutter-bootstrap` skill** — unified `--up` / `--down` lifecycle for backend + Android device. Writes a marker file at `<reports-dir>/.qa-bootstrap-marker` that doubles as lock (G6) and teardown state. Idempotent; survives crashes via PID-ward health checks.
- **`qa-flutter-unit-generator` skill** — unit / widget / state test generator and runner (fast layer of the pyramid, no device required).
- **`qa-flutter-manual-runner` skill** — `/qa-run` command for `flutter_drive` + `integration_test` based E2E runs. New default Android stack.
- **`qa-flutter-release-gate` skill** — `/qa-release-gate` GO/NO-GO production gate wrapping `qa-stability-agent`; emits JSON artifact alongside Markdown report (G4, schema v1.0); applies 5-rule root-cause correlation over findings.
- **`autonomous.*` block in `qa-agent.yaml`** — declarative backend/device bootstrap consumed by `qa-flutter-bootstrap`.
- **Diagnóstico sections in per-feature reports** — error principal, stack trace top 8 frames, filtered log tail.
- **Appium bootstrap contract v1** — unified marker-based protocol for companion `scripts/bootstrap.py` / `teardown.py`. See [`docs/references/appium-bootstrap-contract.md`](docs/references/appium-bootstrap-contract.md). Legacy fallback retained one minor for migration.

### Changed

- **`qa-stability-agent`** — Step 0 (invoke `qa-flutter-bootstrap --up`) + Step 5 (mandatory teardown). Runner delegation unchanged.
- **`qa-flutter-android-runner`** — migrates bootstrap invocation to `qa-flutter-bootstrap --up --caller=android-runner`; deprecates direct `scripts/bootstrap.py` invocation.
- **Default Android stack is now `flutter_drive`** — Appium remains supported as opt-in via `project.android_stack: "appium"`. Decision rationale in [`docs/android-stacks.md`](docs/android-stacks.md).

### Gaps deferred

- G3 — Notifications on NO-GO (out of plugin scope; handle via CI notification step)
- G5 — Waiver approval workflow (compliance-driven, future)
- G7 — Cost tracking per run (future)

## [1.0.0] — 2026-04-18

### Added

- `qa-flutter:qa-stability-agent` — orchestrator agent that auto-detects Flutter platform and routes to the correct runner
- `qa-flutter-android-runner` — Appium-based QA runner for Flutter Android apps
- `qa-flutter-android-tester` — single-feature Appium tester (sub-agent of android-runner)
- `qa-flutter-web-runner` — stability checker for Flutter web apps; runs `flutter analyze`, `flutter test`, `flutter build web`, git diff analysis, and generates a functional test checklist
- Generic configuration: all hardcoded paths removed; skills prompt users for paths at runtime
