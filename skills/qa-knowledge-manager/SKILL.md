---
name: qa-knowledge-manager
description: Manages the autonomous QA learning cycle — runs preflight checks before a QA run (applying known fixes automatically), captures new errors into qa-knowledge.yaml after the run, and supports manual register/report modes. Invoked by qa-stability-agent as Steps 0a and 0b. Also callable standalone to inspect or update the knowledge base without running tests.
argument-hint: "[preflight | learn | register | report]"
---

# qa-knowledge-manager

Gestión del ciclo completo de autoaprendizaje del QA autónomo.  
Lee, aplica y actualiza `qa-plugin-config/qa-knowledge.yaml` en el proyecto Flutter activo.

**Invocation examples:**
- `/qa-knowledge-manager preflight` — antes del run; detecta y aplica soluciones conocidas
- `/qa-knowledge-manager learn` — tras el run; registra errores nuevos en qa-plugin-config/qa-knowledge.yaml
- `/qa-knowledge-manager register` — registro manual de un error/solución conocida
- `/qa-knowledge-manager report` — estado actual de la base de conocimiento

## Scope

This skill:
- ✅ Reads and applies entries from `qa-plugin-config/qa-knowledge.yaml` before a run (preflight).
- ✅ Scans run output for new errors and drafts entries with user confirmation (learn).
- ✅ Supports manual registration of known errors and their solutions (register).
- ✅ Reports the current state of the knowledge base (report).

This skill does NOT:
- Run tests — delegates to runner skills via `qa-stability-agent`.
- Modify plugin files — all fixes live in the project's `qa-plugin-config/qa-knowledge.yaml`, not in the plugin.
- Apply solutions silently — always confirms with the user before writing new entries.
- Block the main run on failure — both preflight and learn are non-fatal wrappers.

## When to use

- Automatically via `qa-stability-agent` (Steps 0a and 0b) — no need to invoke manually in normal runs.
- Standalone `preflight` to check environment state before a manual test session.
- Standalone `learn` to backfill `qa-plugin-config/qa-knowledge.yaml` from a run that completed without the agent.
- `register` to document a known environment fix before it is observed in a real run.
- `report` for a quick audit of what the knowledge base contains.

## When NOT to use

- As a replacement for `qa-stability-agent` — this skill only manages knowledge, not the full QA lifecycle.
- When `qa-plugin-config/qa-knowledge.yaml` does not exist and the project hasn't opted into autoaprendizaje — the skill will inform and no-op cleanly.
- For modifying plugin behavior — fixes specific to a project's environment belong in `qa-plugin-config/qa-knowledge.yaml`, not in skill files.

---

## Modos de operación

```
/qa-knowledge-manager preflight   → Verifica y aplica soluciones conocidas antes del run
/qa-knowledge-manager learn       → Analiza errores del run reciente y actualiza la base
/qa-knowledge-manager register    → Registra un nuevo error/solución manualmente
/qa-knowledge-manager report      → Muestra el estado de la base de conocimiento
```

Si no se pasa modo, el agente **pregunta** cuál ejecutar.

---

## MODO: preflight

### Objetivo
Ejecutar antes de cualquier run de QA para detectar condiciones de error conocidas
y aplicar soluciones automáticas, acortando el tiempo total de ejecución.

### Pasos

**Paso 1 — Localizar qa-knowledge.yaml**

Buscar en este orden:
1. `qa-plugin-config/qa-knowledge.yaml` en el directorio actual (raíz del proyecto Flutter)
2. Ruta indicada en `qa-plugin-config/qa-agent.yaml` bajo `knowledge.path`
3. Si no existe: informar al usuario y ofrecer crear uno desde el template

```
Si no existe qa-plugin-config/qa-knowledge.yaml:
  → Avisar: "No se encontró qa-plugin-config/qa-knowledge.yaml. Crear uno desde el template con:
     cp ~/.claude/plugins/local/qa-flutter/templates/qa-knowledge.yaml ./qa-plugin-config/qa-knowledge.yaml"
  → Continuar el run sin preflight (no bloquear)
```

**Paso 2 — Cargar entradas habilitadas**

Leer todas las entradas donde `enabled: true` y que tengan `preflight_check` no nulo.
Ordenar por categoría: `environment` primero, luego `build`, luego `ci`.

**Paso 3 — Ejecutar checks**

Para cada entrada con `preflight_check`:

```
Si preflight_check.type == "command":
  1. Ejecutar el comando del check
  2. Comparar resultado contra min_expected (con invert si aplica)
  3. Si se detecta la condición de error → ir a Paso 4
  4. Si no hay error → continuar al siguiente entry
```

Registrar en log interno:
```
[PREFLIGHT] K001 — check: OK (emulador disponible)
[PREFLIGHT] K002 — check: FALLO → iniciando solución automática
```

**Paso 4 — Aplicar solución (si auto_apply: true)**

```
Si solution.auto_apply == true:
  Para cada step en solution.steps:
    1. Ejecutar step.cmd (reemplazando variables de entorno si aplica)
    2. Si wait_seconds > 0: esperar ese tiempo
    3. Si el comando falla: loggear el error y continuar con el siguiente step
  
  Después del último step: re-ejecutar el preflight_check para verificar que se resolvió
  
  Si la verificación post-solución pasa:
    → Actualizar en qa-knowledge.yaml:
       stats.applied_count += 1
       stats.last_applied = fecha_actual (YYYY-MM-DD)
       stats.last_result = "success"
    → Log: "[PREFLIGHT] K002 — solución aplicada: ÉXITO"
  
  Si la verificación post-solución falla:
    → Actualizar: stats.last_result = "failure"
    → Log: "[PREFLIGHT] K002 — solución aplicada: FALLO"
    → Mostrar fallback_message al usuario
    → Preguntar si desea continuar el run de todas formas

Si solution.auto_apply == false:
  → Mostrar fallback_message al usuario
  → Preguntar: "¿Continuar el run sin resolver este problema? (s/n)"
```

**Paso 5 — Resumen preflight**

Antes de entregar el control al runner:

```
╔══════════════════════════════════════════╗
║  QA PREFLIGHT — Resumen                  ║
╠══════════════════════════════════════════╣
║  Checks ejecutados : N                   ║
║  Sin problemas     : N                   ║
║  Resueltos auto    : N                   ║
║  Requieren acción  : N (lista)           ║
╚══════════════════════════════════════════╝
```

Si hay items que "requieren acción" y el usuario eligió no continuar → salir con código 1.
Si todo OK o el usuario eligió continuar → salir con código 0.

---

## MODO: learn

### Objetivo
Analizar la salida del run reciente, identificar errores nuevos y actualizar `qa-knowledge.yaml`
con lo aprendido (errores sin solución conocida o soluciones encontradas dinámicamente).

### Pasos

**Paso 1 — Localizar el reporte del run reciente**

Buscar en `reports.output_dir` (definido en `qa-plugin-config/qa-agent.yaml`) el reporte más reciente:
- Archivos `.md` o `.json` con timestamp del run actual
- Si no existe reporte: usar el output de la sesión actual

**Paso 2 — Extraer errores del run**

Escanear el reporte/output buscando:
- Líneas con `ERROR`, `FAILED`, `Exception`, `Error:`, `fatal:`
- Mensajes de timeout, connection refused, build failed
- Exit codes distintos de 0

Para cada error encontrado:

```
¿Coincide con algún pattern de qa-knowledge.yaml?
  SÍ → marcar como "error conocido, ya registrado" (no duplicar)
  NO → agregar a lista de "errores nuevos"
```

**Paso 3 — Procesar errores nuevos**

Para cada error nuevo:

```
1. Mostrar el error al agente (contexto propio)
2. Intentar determinar:
   a. Categoría probable (environment / build / ci)
   b. Si el error fue resuelto durante el run (¿hubo un comando exitoso después?)
   c. Qué comando lo resolvió (si aplica)

3. Generar borrador de entrada para qa-knowledge.yaml:
   - id: siguiente ID disponible (p. ej. K017, K018, ...)
   - pattern: regex derivado del mensaje de error (escapar caracteres especiales)
   - category: categoría inferida
   - description: descripción legible del error
   - solution: si se identificó solución → auto_apply: true + steps
               si no → auto_apply: false + fallback_message genérico

4. IMPORTANTE — Antes de escribir, presentar el borrador al usuario:
   "Detecté el siguiente error nuevo durante el run. ¿Quieres registrarlo
    en qa-knowledge.yaml para que futuros runs lo resuelvan automáticamente?"
   [mostrar el bloque YAML del borrador]
   Opciones: (a) Registrar tal cual  (b) Editar antes de registrar  (c) Ignorar

   → Solo escribir en qa-plugin-config/qa-knowledge.yaml tras confirmación explícita del usuario
   → Con registered_by: "agent"

5. Si el usuario elige ignorar: no escribir nada. Si el error se repite en runs
   futuros, volver a preguntar (no suprimir permanentemente).
```

**Regla clave — Nunca modificar archivos del plugin:**

```
Si durante el análisis el agente identifica que la corrección podría aplicarse
en un archivo del plugin (bootstrap, skill, agent):

  → NO modificar ningún archivo dentro del directorio del plugin
  → En su lugar, crear o actualizar la entrada en qa-knowledge.yaml
  → Incluir en fallback_message una nota explicando que la corrección
     es específica de este proyecto/entorno y vive aquí intencionalmente:

     fallback_message: |
       [Corrección de infraestructura — específica de este proyecto]
       Este fix no está en el plugin para preservar actualizaciones futuras.
       Está aquí en qa-knowledge.yaml donde viaja con el proyecto.
       Ejecutar manualmente: <comando>
```

Ejemplo de situación concreta:
- El emulador no alcanza `localhost` del host → la solución es `adb reverse`
- La sugerencia de añadirlo al bootstrap del plugin es incorrecta porque:
  - El puerto puede variar por proyecto
  - El plugin es genérico y compartido
  - `qa-knowledge.yaml` es el lugar correcto: específico del proyecto, versionado con él

**Paso 4 — Actualizar stats de entradas existentes aplicadas**

Si durante el run se aplicaron soluciones conocidas (información del modo preflight):
- Verificar si `stats.last_result` ya fue actualizado en el preflight
- Si no: actualizarlo ahora con el resultado real del run

**Paso 5 — Resumen del aprendizaje**

```
╔══════════════════════════════════════════════╗
║  QA LEARN — Resumen de aprendizaje           ║
╠══════════════════════════════════════════════╣
║  Errores en el run        : N                ║
║  Ya registrados           : N                ║
║  Nuevos registrados       : N (IDs: K0xx...) ║
║  Con solución automática  : N                ║
║  Sin solución (manual)    : N                ║
╚══════════════════════════════════════════════╝

qa-knowledge.yaml actualizado.
```

---

## MODO: register

### Objetivo
Permitir al usuario o al agente registrar manualmente un error conocido con su solución,
sin necesidad de que ocurra durante un run real.

### Pasos

**Paso 1 — Recopilar información**

Solicitar (o recibir como argumentos):

```
Error a registrar:
  Descripción: _______________
  Mensaje/patrón de error (texto o regex): _______________
  Categoría [environment / build / ci]: _______________

Solución:
  ¿Tiene solución automática? [s/n]: ___
  Si sí → comandos (uno por línea):
    Comando 1: ___
    Comando 2: ___
  Si no → mensaje para el operador: _______________
```

**Paso 2 — Asignar ID y construir entrada**

```yaml
- id: "K0XX"                          # próximo ID disponible
  registered_at: "YYYY-MM-DD"
  registered_by: "user"
  category: "<categoría>"
  description: "<descripción>"
  enabled: true
  pattern: "<patrón>"
  preflight_check: null               # el usuario puede editarlo después
  solution:
    auto_apply: <true|false>
    type: "command"                   # o "manual"
    steps:
      - cmd: "<comando>"
        wait_seconds: 5
        description: "<descripción del step>"
    fallback_message: |
      <mensaje de fallback>
  stats:
    applied_count: 0
    last_applied: null
    last_result: null
```

**Paso 3 — Insertar en qa-plugin-config/qa-knowledge.yaml**

Insertar antes del marcador `# [AGENT_ENTRIES_START]`.

**Paso 4 — Confirmar**

```
✅ Entrada K0XX registrada en qa-plugin-config/qa-knowledge.yaml
   Descripción: <descripción>
   Solución automática: <sí/no>
   La solución se aplicará en el próximo preflight si se detecta el patrón.
```

---

## MODO: report

### Objetivo
Dar visibilidad del estado actual de la base de conocimiento.

### Output esperado

```
╔═══════════════════════════════════════════════════════════════╗
║  QA KNOWLEDGE REPORT — qa-plugin-config/qa-knowledge.yaml     ║
║  Actualizado: YYYY-MM-DD  |  Total entradas: N                ║
╠═══════════════════════════════════════════════════════════════╣
║  Por categoría:                                               ║
║    environment : N entradas  (N con auto_apply)               ║
║    build       : N entradas  (N con auto_apply)               ║
║    ci          : N entradas  (N con auto_apply)               ║
╠═══════════════════════════════════════════════════════════════╣
║  Top 3 soluciones más aplicadas:                              ║
║    1. K002 — Puerto 8080 ocupado          (N veces, N% éxito) ║
║    2. K001 — Emulador no iniciado         (N veces, N% éxito) ║
║    3. K004 — flutter pub get fallido      (N veces, N% éxito) ║
╠═══════════════════════════════════════════════════════════════╣
║  Entradas sin solución automática (requieren atención):       ║
║    - K003: Backend no disponible                              ║
║    - K006: Variable de entorno faltante                       ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## Common mistakes

| Mistake | Fix |
|---|---|
| Running `preflight` expecting it to fix infrastructure provisioning (AVD creation, SDK install) | Out of scope — preflight only applies solutions already in `qa-knowledge.yaml`. Provision infra separately. |
| Setting `auto_apply: true` for destructive or non-idempotent commands | Reserve `auto_apply: true` for safe, idempotent commands. Use `auto_apply: false` + `fallback_message` for anything requiring human judgement. |
| Adding fixes that reference plugin-internal file paths | Fixes that depend on the plugin's own structure belong in the plugin. Only project-specific environment fixes belong in `qa-knowledge.yaml`. |
| Letting `learn` accumulate entries without reviewing them | Each new entry requires explicit user confirmation. An unreviewed `auto_apply: true` entry could silently run harmful commands on the next preflight. |
| Expecting `learn` to autonomously resolve root causes | `learn` identifies and documents; it does not resolve. Root cause analysis remains the user's responsibility. |

---

## Reglas de escritura en qa-knowledge.yaml

- Siempre preservar comentarios existentes en el archivo
- Nunca eliminar entradas existentes; si una solución dejó de ser válida, añadir `enabled: false`
- Al insertar entradas del agente, añadir comentario con fecha:
  ```yaml
  # Registrado automáticamente por el agente — YYYY-MM-DD
  - id: "K0XX"
    ...
  ```
- Actualizar `updated_at` en la cabecera del YAML tras cada modificación
- Usar comillas dobles para strings que contienen caracteres especiales o regex

---

## Variables de entorno reconocidas

| Variable | Uso | Default |
|---|---|---|
| `QA_AVD_NAME` | Nombre del AVD a lanzar | `Pixel_5_API_33` |
| `QA_BACKEND_HEALTH` | URL del health check del backend | `http://localhost:8080/api/health` |
| `QA_KNOWLEDGE_PATH` | Ruta alternativa a qa-knowledge.yaml | `./qa-plugin-config/qa-knowledge.yaml` |

---

## Integración con qa-stability-agent

El agente principal llama a este skill como dos pasos envolventes:

```
[qa-stability-agent]
  └─ Step 0a: /qa-knowledge-manager preflight   ← ANTES del run
  └─ Step 1..N: runners existentes              ← run normal
  └─ Step 0b: /qa-knowledge-manager learn       ← DESPUÉS del run
```

Si el preflight retorna exit code 1 (problema bloqueante no resuelto),
`qa-stability-agent` debe detenerse y reportar el bloqueo al usuario.

---

## Limitations

- **Knowledge is project-scoped** — `qa-knowledge.yaml` travels with the project repo, not the plugin. Sharing fixes across projects requires manual copy.
- **Pattern matching is regex-based** — `learn` derives patterns from error messages. Complex or environment-specific errors may produce overly broad regexes; always review before confirming.
- **No conflict resolution** — if two entries have overlapping patterns, both will trigger in preflight. Order them intentionally and use `enabled: false` to deactivate superseded entries.
- **`learn` requires a recent report** — if the run produced no report file (e.g., bootstrap failed before any runner ran), `learn` falls back to session output, which may be incomplete.
- **Stats are advisory only** — `applied_count` and `last_result` are updated best-effort. Do not use them as production SLA metrics.
