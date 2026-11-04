# CanvasKit fill strategy — árbol de decisión consolidado

**Para qué sirve este archivo**

Cuando estés generando un spec nuevo, editando `_helpers.ts`, o decidiendo cómo interactuar con un widget de Flutter web (CanvasKit) desde Playwright, consulta este árbol **antes de iterar**. La elección de primitiva está condicionada por cómo Flutter expone el widget al DOM accessibility tree, no por lo que el código Dart sugiere.

**Required reading antes de:**
- Editar el `_helpers.ts` del proyecto
- Generar un spec nuevo que involucre inputs o botones no triviales
- Diagnosticar un fail de fill/tap que no matcheó K013-K016 en auto-match

**Pre-requisito hard:** la app de test debe llamar `WidgetsFlutterBinding.ensureInitialized().ensureSemantics()` en `main()`. Sin esto, ningún selector encuentra nodos (ver K001 + Step 3e del driver).

---

## Árbol de decisión

### 1. ¿Qué clase de widget quieres manipular?

| Tipo | Ir a |
|---|---|
| Input de texto (TextFormField, TextField) | §2 |
| Botón / acción tappeable (ElevatedButton, FilledButton, IconButton, PopupMenuItem) | §5 |
| Dropdown / menú abierto | §6 |
| Otro (toggle, slider, custom) | §7 |

### 2. ¿El input tiene `labelText` o `decoration.labelText` definido en Dart?

| Respuesta | Acción |
|---|---|
| **Sí, hay labelText visible** | El input se serializa como `<input aria-label="...">` en el DOM permanente del overlay `flt-semantics`. → §3 |
| **No tiene labelText** (típicamente teléfono, código numérico, OTP) | El input on-demand aparece bajo `flt-text-editing-host` solo al recibir foco. → §4 |

### 3. Caso input con label → `fillInput`

```typescript
await fillInput(page, 'Email *', 'qa@example.com');
```

**Por qué funciona** (K013):
- Flutter NO bindea el `<input>` del overlay al `TextEditingController` hasta que recibe foco.
- `fillInput` hace `click({ force: true })` (bypasea el `visibility:hidden` del overlay) → Flutter attacha el text editing client → el `fill()` siguiente escribe al controller real.
- `waitForTimeout(200)` post-fill le da tiempo a Flutter a commit el value al widget state.

**Cuidado con:**
- **Orden de campos en formularios con dropdowns:** llenar dropdowns ANTES de text fields. Cambiar un dropdown puede triggerear rebuild del form que descarta inputs ya escritos.
- **Tests en modo `--headed` que escriben values inválidos para validación:** Chrome autofill puede pisarlos. Tras `fillInput`, llamar `lockInputValue(page, label, value)` (K016) para congelar el getter.

### 4. Caso input sin label → `fillUnlabeledTextbox`

```typescript
await fillUnlabeledTextbox(page, '+50688551234');
```

**Por qué funciona** (K015):
- `flt-semantics[role="textbox"]` sin `aria-label` representa el input on-demand de Flutter.
- Dispatch chain (pointerdown → mousedown → pointerup → mouseup → click) sobre ese nodo → Flutter da foco de teclado al input on-demand creado en `flt-text-editing-host`.
- `keyboard.type(value)` escribe en el `document.activeElement` (que ahora es el input on-demand), letra a letra al controller.

**Limitaciones:**
- Si hay múltiples `flt-semantics[role="textbox"]` sin aria-label en la misma pantalla, el helper agarra el **primero** (orden DOM). Para identificar uno específico, agregar `Semantics(label: 'X')` al widget en Flutter (preferido) o usar selector más específico por posición.
- No usar `fill()` aquí — no hay un input DOM permanente al cual `fill()` pueda apuntar.

### 5. Caso botón / acción tappeable → `tapButton`

```typescript
await tapButton(page, 'Siguiente');
await tapButton(page, /Solicitud/i);     // regex también funciona
```

**Por qué funciona** (K014):
- Flutter pone el texto visible **tanto en `textContent` como en `aria-label`** según el widget. `hasText()` de Playwright solo examina `textContent`, por eso falla en ~30% de los casos.
- `tapButton` busca **dual**: primero por textContent, fallback a aria-label.
- Acepta tanto `role="button"` como `role="menuitem"` (PopupMenuItem también).
- Dispatch chain (no `Locator.click()`) porque Playwright `.click()` NO dispara `onPressed` de Flutter.

**Casos donde el texto está SOLO en aria-label** (textContent vacío):
- `FilledButton.icon` — el ícono ocupa el textContent.
- `DropdownButton` sin selección — solo muestra el label en aria-label.
- `IconButton` con tooltip.
- `PopupMenuItem` (`role="menuitem"`).

**No usar `buttonByText` para tap** — solo para inspección. `buttonByText` es un Locator con `hasText`, que no cubre aria-label.

### 6. Caso dropdown / menú abierto

- **Para abrir el dropdown:** `tapButton(page, '<label del dropdown>')` (caso §5, el aria-label del DropdownButton).
- **Para seleccionar un item:** los items aparecen como `flt-semantics[role="menuitem"]`. `tapButton(page, '<texto del item>')` los cubre por el mismo dual search.
- Si el dropdown tiene scroll interno: scroll al item con `page.evaluate(() => document.querySelector('flt-semantics[role="menuitem"][aria-label*="X"]').scrollIntoView())` antes del tap.

### 7. Caso no cubierto (toggle, slider, custom)

Cuando no hay primitiva ya construida:

1. **Inspeccionar el DOM**: abrir DevTools, buscar el widget en el a11y tree, identificar `role` + atributos.
2. **Tentar `page.evaluate()` directo**:
   ```typescript
   await page.evaluate(() => {
     const el = document.querySelector('flt-semantics[role="checkbox"][aria-label*="Acepto"]');
     // dispatch chain estándar...
   });
   ```
3. **Documentar como K-new**: si la estrategia funciona, registrarla con `qa-knowledge-manager learn` o agregarla manualmente al `qa-knowledge.yaml` del proyecto. Patrón estructural + fallback_message con el snippet.
4. **Si después de 30 min de exploración no anda → consultar Operational directives (R4 hypothesis gate)** del SKILL.md — no iterar a ciegas.

---

## Matriz de primitivas → cuándo usar qué

| Necesidad | Primitiva | Por qué (K#) |
|---|---|---|
| Escribir en input con label visible (TextFormField con labelText) | `fillInput(page, label, value)` | Click-prime attacha el text editing client (K013) |
| Escribir en input sin label (teléfono, OTP) | `fillUnlabeledTextbox(page, value)` | Dispatch + keyboard.type al activeElement (K015) |
| Tap a botón / menuitem por texto | `tapButton(page, text)` | Dual textContent + aria-label, dispatch chain (K014) |
| Bypass Chrome autofill en tests `--headed` de validación | `lockInputValue(page, label, value)` tras fillInput | Congela el getter del `<input>` (K016) |
| Esperar a que Flutter levante y emita a11y tree | `openApp(page, baseUrl)` | networkidle + waitFor flt-semantics |
| Capturar screenshot del paso actual | `takeStepShot(page, info, label)` | Adjunta al reporte de Playwright |
| Inspección visual de un botón (no tap) | `buttonByText(page, text)` | Locator legacy con hasText. Solo inspección — NO para click. |
| Caso ad-hoc no cubierto | `page.evaluate()` directo + registrar K-new | Siempre documentar antes de iterar (R4) |

---

## Anti-patrones a evitar

| Anti-patrón | Por qué falla | En su lugar |
|---|---|---|
| `await page.click('flt-semantics[role="button"]')` | Playwright `.click()` no dispara `onPressed` de Flutter | `tapButton` con dispatch chain |
| `await page.locator('input').fill(value)` sin click previo | Si Flutter no bindeó el input al controller, el fill es silencioso | `fillInput` con click-prime |
| Buscar botones solo con `hasText` | No cubre los botones con ícono o dropdowns sin selección | `tapButton` con dual search |
| Usar `Semantics(identifier: 'X')` en Dart esperando que aparezca en DOM | `identifier` solo lo usa `SemanticsTester` interno (test framework de Flutter) | `Semantics(label: 'X')` se serializa como `aria-label` |
| `await page.waitForTimeout(2000)` para "que Flutter se asiente" | Brittle, no determinista | `await locator.waitFor({ state: 'attached' })` después de la acción que debe materializar el nodo |
| Probar 5 estrategias seguidas sin documentar cuál y por qué cada una falla | Pierde tiempo + el modelo no aprende | Después de 1-2 fallos, consultar este árbol + K013-K016. Después de 3 → activar R4 gate |

---

## Cuándo este archivo NO basta

- Casos nuevos no documentados aquí ni en K013-K016 → modo exploración con tope de tiempo. Si pasas de 30 min sin progreso, R4 te detiene.
- Issues estructurales de la app de test (ej. `ensureSemantics()` faltante) → eso es K001 + Step 3e del driver, no este árbol.
- Backend down / build inválido → eso es Step 3c / K002, no este árbol.
- Pantalla nueva sin a11y tree conocido y no sabes qué primitiva va a aplicar → un smoke Playwright MCP en 10-15 calls te confirma roles + selectores antes de codificar el spec. Ver `mcp-exploration-companion.md`. **Pedirlo upfront ahorra ~3 iteraciones del `_helpers.ts`** en casos no triviales.

---

## Referencias cruzadas

- `templates/qa-knowledge.yaml` — entradas K013, K014, K015, K016 (fuentes ejecutables de los snippets).
- Origen empírico de las primitivas: una sesión de validación interna.
- `skills/qa-flutter-web-driver-runner/SKILL.md` Section C.4 — código completo del `_helpers.ts` template.
- `skills/qa-flutter-web-driver-runner/SKILL.md` Operational directives — gate R4 que se dispara cuando este árbol no fue suficiente.
