# Playwright MCP como complemento exploratorio del driver

**Para qué sirve este archivo**

El `qa-flutter-web-driver-runner` sigue el ciclo Playwright tradicional (genera `.spec.ts`, ejecuta, lee report, itera helpers). Ese ciclo es óptimo para **regresión**: 1 vez codificado, se ejecuta sin overhead. Para **exploración** — entender el a11y tree de una pantalla nueva, decidir qué primitiva del `_helpers.ts` aplica, validar selectores — el ciclo spec.ts es caro: 5+ runs en wizards complejos (sesión 2026-05-20).

**Playwright MCP** (`@playwright/mcp` de Microsoft) es complementario: el modelo opera el browser directo desde Claude Code, captura el a11y tree inline, prueba selectores en 1 vuelta sin generar archivos. **No reemplaza el driver** — lo complementa en la fase exploratoria.

**Required reading antes de:**
- Generar un spec nuevo en un wizard / pantalla nunca antes mapeada (potencial ahorro de iteraciones).
- Reanudar tras el R4 gate cuando 3 ediciones del `_helpers.ts` no hicieron pasar el test (la opción "validar a11y tree antes de iterar más").
- Diagnosticar un fail "indirecto" K013/K014/K015 (auto-match disparó pero la mitigación no resolvió).

---

## Cuándo usar MCP vs cuándo usar el driver tradicional

| Situación | Herramienta | Razón |
|---|---|---|
| Pantalla nueva, a11y tree desconocido | **MCP primero**, luego driver | Confirmar que `ensureSemantics()` está activo y qué roles emite, antes de codificar selectores. |
| Spec ya existe, falla en CI / regresión | **Driver** | Reproducir el fail tal cual; MCP es ad-hoc, no determinista en suites. |
| R4 gate disparó (3 ediciones sin progreso) | **MCP para validar hipótesis** | La opción (d) del gate ahora incluye "smoke MCP de 10 min antes de continuar". Si el a11y tree contradice tu hipótesis, no sigas iterando ese helper. |
| Bug reportado por usuario sobre 1 paso específico | **MCP** | Reproduce en vivo, captura screenshot + console + a11y tree en 5-10 calls. |
| Validar que el plugin v1.x.y mejora el ciclo | **MCP** | Smoke comparativo del wizard, mide tool calls antes/después del cambio del helper. |
| CI gate / release gate (`qa-flutter-release-gate`) | **Driver** | Reproducible, sin tooling extra que el CI no tiene. |

**Regla de oro:** MCP es del developer/QA en su máquina; el driver es del agente autónomo (CI, release gate, stability run). No mezclar.

---

## Instalación / disponibilidad

Playwright MCP se registra user-level (no tocando el repo del plugin):

```bash
# user-level — disponible en cualquier proyecto del usuario
claude mcp add playwright npx @playwright/mcp -- --user --no-sandbox
# (o el comando equivalente según versión del CLI)
```

Comprobar con `/mcp` en Claude Code — debe aparecer `playwright` con health `✓ Connected` y tools `browser_*` (~40).

**No requerido para el plugin:** el driver no asume Playwright MCP. Si el usuario lo tiene, gana exploración rápida; si no, el ciclo spec.ts tradicional sigue funcionando idéntico.

---

## Workflow exploratorio recomendado (10-15 calls típicas para 1-2 pantallas)

### Paso 0 — Bootstrap del proyecto

Igual que para el driver: la app debe estar levantada en `localhost:<port>`. Lanzar `qa-flutter-bootstrap --up` si no está corriendo. **Asegurar que se aplique el perfil QA** (`autonomous.web.build_flags: ["--dart-define-from-file=dart_define/qa.json"]` en `qa-agent.yaml`) para que el backend QA reciba las requests con los headers correctos.

### Paso 1 — Smoke a11y tree

```
browser_navigate("http://localhost:4200/")
browser_wait_for(time: 3)         # Flutter levanta CanvasKit
browser_snapshot()
```

Lo que se valida en 3 calls:
1. **¿`ensureSemantics()` está activo?** Si el snapshot solo emite `generic` / `text` sin `textbox` / `button` / `menuitem` → falta el binding (pre-requisito hard del driver, ver K001 + Step 3e).
2. **¿Los roles ARIA son ricos?** Si hay `textbox "Nombre *"`, `button "Ciudad *"`, `progressbar "1"` → la app está bien instrumentada y los selectores del `_helpers.ts` van a funcionar.
3. **¿Hay textboxes sin aria-label?** El snapshot los muestra como `textbox [ref=eXX]` (sin nombre) → caso K015 (`fillUnlabeledTextbox`).

### Paso 2 — Probar primitivas inline

Para cada widget no trivial:

| Necesidad | Llamada MCP |
|---|---|
| Click en botón con texto visible | `browser_click(target: "<ref>")` — si timeoutea con `flutter-view intercepts pointer events` → K011, aplicar mitigación K012 vía `browser_evaluate` con `dispatchEvent` |
| Llenar textbox con aria-label | `browser_type(target: "<ref>", text: "...", slowly: true)` — equivale a `pressSequentially`. **Click primero para tomar foco, luego type slowly.** |
| Llenar textbox sin aria-label | `browser_type(target: "<ref>", text: "...", slowly: true)` — MCP infiere `input[type="tel"]` o similar automáticamente |
| Verificar value commiteado | `browser_evaluate(() => Array.from(document.querySelectorAll('input')).map(i => ({label: i.getAttribute('aria-label'), value: i.value})))` |
| Capturar console errors | `browser_console_messages(level: "warning")` |

### Paso 3 — Mapear MCP-findings → K# del knowledge base

Para que el smoke MCP alimente al driver, **documentar lo que se encontró** antes de codificar el spec. Patrón:

```
Hallazgos MCP wizard <feature>:
- e96/e159/e226 = textbox "Nombre *"   → fillInput  (K013)
- e100/e163/e230 = button "Ciudad *"           → tapButton + dispatch (K014)
- e102/e164/e231 = button "Categoría *" → tapButton + dispatch (K014, hit-test K011)
- e116/e177/e244 = textbox (sin aria-label, type=tel) → fillUnlabeledTextbox (K015)
- e123/e184/e251 = button "Siguiente"        → tapButton  (K014)
```

Esta tabla es el insumo directo para el spec.ts: cada entry tiene la primitiva del `_helpers.ts` que se va a usar. **Cero iteración a ciegas**.

### Paso 4 — Cerrar el browser

```
browser_close()
```

MCP no deja procesos. El http.server del bootstrap sigue corriendo para el driver.

---

## Lo que NO hace MCP (y por qué el driver sigue siendo necesario)

| Limitación de MCP | Implicación para el driver |
|---|---|
| Cada click puede invalidar refs (Flutter rerendera) | Re-snapshot frecuente cuesta tool calls. Spec.ts no sufre porque usa selectores estables (aria-label, role). |
| MCP no infiere dependencias inter-step (Step 5 depende de Step 3) | `qa-flutter-test-planner` sigue siendo necesario para mapear pre-condiciones; MCP solo prueba lo que se le pide. |
| MCP no genera reports HTML reproducibles | El driver genera `qa-reports/*.md` consumibles por `qa-knowledge-manager learn`. |
| MCP no se ejecuta en CI / release gate | Solo el driver (autónomo, sin browser real). MCP es del developer en su máquina. |
| Sesión MCP no aprende — no actualiza `qa-knowledge.yaml` | El driver sí (`qa-knowledge-manager learn` al final). |

→ Por eso el enfoque recomendado es combinar: MCP para exploración + driver para regression. **El plugin no integra MCP; lo documenta como complemento del usuario.**

---

## Métrica de costo orientativa (sesión de validación interna)

Smoke contra un wizard multi-paso de una app Flutter web de prueba:

| Magnitud | Valor observado | Comentario |
|---|---|---|
| Tool calls para mapear Step 1 (12 widgets) | ~10 | navigate + wait + snapshot + 7 clicks/fills + evaluate |
| Tool calls para confirmar K011 + mitigación K012 | 2 | click fallido + evaluate con dispatchEvent |
| Tool calls para confirmar K013 generalizado (fill loss tras rerender) | 1 | evaluate dumpeando todos los `<input>.value` |
| Extrapolación a wizard completo sin wrappers MCP | cientos de calls | extrapolación lineal |
| Extrapolación con `clickViaSemantics` + `flutterFill` + `iterateAriaDuplicates` inline | aprox. la mitad | factor 2x del baseline de una app DOM-native |

**Interpretación:** el MCP exploratorio es rentable hasta ~2-3 pantallas / wizard parcial. A partir de 4+ pantallas el costo cumulativo supera al de codificar el spec.ts directo. Usar MCP **para entender, no para correr regresión**.

---

## Anti-patrones

| Anti-patrón | Por qué falla | En su lugar |
|---|---|---|
| Usar MCP para correr la suite completa de regresión | El costo de tool calls es 5-10x el del spec.ts; resultado no es reproducible en CI | Driver tradicional |
| Generar spec sin smoke MCP previo en pantalla nueva con widgets no triviales | Probabilidad alta de iterar 3+ ediciones del `_helpers.ts` antes de descubrir un selector roto | 10-15 calls MCP upfront |
| MCP smoke en CI / release gate | Tooling no disponible, browser real no determinista | Driver |
| Iterar MCP a ciegas en una pantalla compleja (>20 widgets) | Costo cumulativo supera al spec.ts; el modelo se pierde | Para ese caso, generar spec.ts y dejarse guiar por los fails |
| Asumir que MCP "resuelve" K011/K013/K015 | NO. MCP los **detecta** rápido pero las **mitigaciones siguen siendo las del `_helpers.ts`** | Smoke MCP → mapear hallazgos a K# → codificar spec con `fillInput`/`tapButton`/`fillUnlabeledTextbox` |
| No aplicar el perfil QA al bootstrap antes del smoke MCP | El backend recibe requests con env de prod o sin header de signup → errores espurios que parecen K# pero son de configuración | Confirmar `dart_define/qa.json` aplicado en el último build (ver `qa-agent.yaml.autonomous.web.build_flags`) |

---

## Cuándo este archivo NO basta

- **Claude for Chrome (extensión)** — para cuando MCP headless no reproduce el fail (race conditions con scheduler real, autofill, sesión persistente). **No instalado por defecto** — una sesión de validación interna concluyó que solo una minoría de los casos observados lo justifica; preferir resolver con MCP + helpers antes.
- **Bugs estructurales del a11y tree** — si MCP smoke revela `generic` pelado pese a `ensureSemantics()`, es K001 del driver, no de este archivo: agregar `Semantics(label: 'X')` en código Dart.
- **Bugs del backend** — MCP no diagnostica HTTP/RPC. Usar `browser_network_requests` (otra tool MCP) o DevTools Network del browser real.

---

## Referencias cruzadas

- `references/canvaskit-fill-strategy.md` — árbol de decisión de primitivas (siguiente paso tras un smoke MCP).
- `templates/qa-knowledge.yaml` — K011 (hit-test), K012 (dispatch chain), K013 (fillInput), K014 (tapButton), K015 (fillUnlabeledTextbox), K016 (lockInputValue).
- `SKILL.md` — Operational directives R4 gate, opción (d) ahora apunta a este archivo.
