# QA Autónomo v2 — Estado del Proyecto y Guía de Instalación

> **Estado:** el plugin `qa-flutter` está en construcción. Este documento describe el estado actual y las brechas pendientes; los componentes y la configuración pueden cambiar entre versiones. Para instalar el plugin, sigue la sección [Installation](../README.md#installation) del README.

---

## Parte 1: Revisión del Estado del Proyecto

### Resumen ejecutivo

El plugin `qa-flutter` tiene una base técnica madura. El stack de QA está completo para ejecuciones asistidas (manual runner, unit tests, release gate), pero **no es verdaderamente autónomo** en el stack `flutter_drive`: un run nocturno o de CI desde estado frío no puede completarse sin intervención humana porque el backend y el dispositivo Android deben estar arriba de antemano.

---

### Componentes del plugin

| Artefacto | Ruta | Estado |
|---|---|---|
| `qa-stability-agent` | `agents/qa-stability-agent.md` | ✅ v2.1 — incluye ciclo de autoaprendizaje + Step 2.5 planning |
| `qa-flutter-test-planner` | `skills/qa-flutter-test-planner/` | ✅ Nuevo — genera planes de test auditables por feature (`/qa-plan`) |
| `qa-flutter-manual-runner` | `skills/qa-flutter-manual-runner/SKILL.md` | Disponible |
| `qa-flutter-android-runner` | `skills/qa-flutter-android-runner/SKILL.md` | Disponible |
| `qa-flutter-android-tester` | `skills/qa-flutter-android-tester/SKILL.md` | Disponible |
| `qa-flutter-unit-generator` | `skills/qa-flutter-unit-generator/SKILL.md` + `references/` | Disponible — dividido en main + 3 templates |
| `qa-flutter-web-runner` | `skills/qa-flutter-web-runner/SKILL.md` | Disponible |
| `qa-flutter-release-gate` | `skills/qa-flutter-release-gate/SKILL.md` + `references/severity-rules.md` | Disponible — emite veredicto GO/NO-GO |
| `qa-knowledge-manager` | `skills/qa-knowledge-manager/SKILL.md` | ✅ Nuevo — autoaprendizaje (preflight / learn / register / report) |
| `README.md` | raíz del plugin | Documentación principal del plugin |
| `docs/android-stacks.md` | matriz de decisión | ✅ Appium vs flutter_drive completa |

---

### Capacidades autónomas actuales

| Capacidad | Stack | Estado |
|---|---|---|
| Regresión E2E sobre tests existentes | flutter_drive | ✅ Si backend + dispositivo ya están arriba |
| Regresión E2E vía Appium | Appium | ✅ Requiere un companion externo (`bootstrap.py` / `teardown.py`) que aporta el usuario; ver [`docs/references/appium-bootstrap-contract.md`](references/appium-bootstrap-contract.md) |
| Generación y ejecución de unit/widget tests | unit-generator | ✅ Completamente autónomo |
| Checklist de estabilidad web | web-runner | ✅ Autónomo (`flutter build web`) |
| Ruteo por plataforma y stack | qa-stability-agent | ✅ Lee yaml y delega |
| Clasificación de severidad + GO/NO-GO | release-gate | ✅ Lee reportes y clasifica |
| Trigger de CI vía `claude -p` | Todos | ✅ Con exit codes |
| **Preflight de errores conocidos** | qa-knowledge-manager | ✅ Detecta y resuelve condiciones antes del run |
| **Autoaprendizaje post-run** | qa-knowledge-manager | ✅ Registra errores nuevos y sus soluciones |
| **Base de conocimiento versionada** | qa-knowledge.yaml | ✅ En el proyecto Flutter, editada por agente y usuario |
| **Planificación de tests por feature** | qa-flutter-test-planner | ✅ Genera plan auditable (pantallas, flujos, criterios, riesgos); auto-inyectado por stability-agent |

---

### Brechas pendientes (gaps para autonomía completa)

| # | Brecha | Prioridad | Desbloquea |
|---|---|---|---|
| **G1** | Bootstrap del backend para stack `flutter_drive` | 🔴 Crítico | Autonomía real en runs nocturnos/CI |
| G2 | Bootstrap configurable del dispositivo (AVD explícito) | 🟠 Alto | Runs desde estado frío en CI |
| G3 | Notificaciones en NO-GO | 🟠 Alto | Alertas accionables |
| G4 | Artefacto JSON + vista histórica | 🟡 Medio | Tendencias y regresiones |
| G5 | Workflow de aprobación de waivers | 🟡 Medio | Compliance/auditoría |
| G6 | Idempotencia de runs (pidfile lock) | 🟡 Medio | Schedules frecuentes seguros |
| G7 | Tracking de costo por run | 🔵 Bajo | Visibilidad a escala |
| ~~G8~~ | ~~Autoaprendizaje — preflight + registro de errores~~ | ~~🟠 Alto~~ | ✅ **Implementado** en v2.1 |

**Orden de implementación recomendado:** G1 → G2 → G4 → G3 → G6 → G5 → G7

---

### Próximo paso concreto (G1)

Añadir `autonomous.backend` en `qa-agent.yaml` y extender `qa-stability-agent` con un "Step 0 — Autonomous bootstrap":

```yaml
autonomous:
  backend:
    start_cmd: "./gradlew bootRun --args='--spring.profiles.active=test'"
    start_cwd: "../my-backend-api"
    health_url: "http://localhost:8080/api/health"
    ready_timeout_seconds: 120
    stop_cmd: "pkill -f 'gradlew bootRun'"
    stop_timeout_seconds: 30
  device:
    boot_avd: "Pixel_5_API_33"
    boot_timeout_seconds: 90
    create_if_missing: false
```

---

---

## Parte 2: Guía de Instalación Paso a Paso

> Esta guía cubre instalación nueva **y** actualización desde versiones anteriores del plugin.

---

### Requisitos previos

Antes de instalar, verificar que el entorno cumple lo siguiente:

| Herramienta | Versión mínima | Verificar con |
|---|---|---|
| Flutter SDK | 3.10+ | `flutter --version` |
| Dart SDK | 3.0+ | `dart --version` |
| Android SDK / `adb` | API 30+ | `adb --version` |
| Claude Code CLI | última | `claude --version` |

---

### Opción A: Instalación nueva

#### Paso 1 — Clonar el plugin en la carpeta local de Claude

```bash
# Ruta estándar para plugins locales de Claude Code
cd %USERPROFILE%\.claude\plugins\local

# Clonar el repositorio
git clone https://github.com/therogueroadmap/qa-flutter-plugin qa-flutter
```

En Linux/macOS:

```bash
cd ~/.claude/plugins/local
git clone https://github.com/therogueroadmap/qa-flutter-plugin qa-flutter
```

#### Paso 2 — Verificar la estructura del plugin

```
qa-flutter/
├── agents/
│   └── qa-stability-agent.md
├── skills/
│   ├── qa-flutter-manual-runner/
│   ├── qa-flutter-android-runner/
│   ├── qa-flutter-android-tester/
│   ├── qa-flutter-unit-generator/
│   ├── qa-flutter-web-runner/
│   ├── qa-flutter-release-gate/
│   └── qa-flutter-test-planner/       ← nuevo
├── docs/
│   ├── android-stacks.md
│   └── roadmap-integration.md
└── README.md
```

En el proyecto Flutter (no en el plugin) también aparece ahora:

```
<proyecto-flutter>/
└── qa-plugin-config/
    ├── qa-agent.yaml
    ├── qa-knowledge.yaml
    ├── qa-plans/                      ← nuevo — planes generados por /qa-plan
    │   ├── login.md
    │   └── checkout.md
    └── qa-reports/
```

> `qa-plugin-config/qa-plans/` debe commitearse — es input auditable del proceso de QA, no output.
> `qa-plugin-config/qa-reports/` se gitignora normalmente.

Si alguna de estas carpetas falta, el plugin está incompleto. Verificar el `git clone` o revisar ramas disponibles.

#### Paso 3 — Configurar `qa-plugin-config/qa-agent.yaml` en tu proyecto Flutter

En la raíz de tu proyecto Flutter (junto a `pubspec.yaml`), crear el directorio `qa-plugin-config/` y dentro el archivo `qa-agent.yaml`:

```yaml
project:
  platform: "android"          # "android" | "web"
  android_stack: "flutter_drive"  # "flutter_drive" | "appium"

device:
  id: emulator-5554
  app_package: com.example.tuapp

backend:
  test_url: "http://localhost:8080/api"

reports:
  output_dir: qa-plugin-config/qa-reports

unit:
  test_root: test
  coverage_target: 80

release_gate:
  threshold: normal              # "strict" | "normal" | "lenient"
  critical_features:
    - login
    - checkout
  coverage_floor: 70
  waivers: []

post_run:
  include_unit: true

# Bloque planning — opcional; habilita qa-flutter-test-planner
planning:
  enabled: false                 # cambiar a true para activar
  test_plan_dir: "qa-plugin-config/qa-plans/"
  require_plan: false            # si true, aborta el run si falta un plan
```

> **Elegir el stack correcto:** Si tus tests están en Dart (`integration_test/`) usa `flutter_drive`. Si usas scripts Python/Appium, usa `appium`. En caso de duda, `flutter_drive` es la opción de menor fricción.

#### Paso 4 — Verificar que el plugin está disponible

Desde Claude Code, ejecutar un comando del plugin, por ejemplo:

```
/qa-plan <feature>
```

Si el comando se reconoce y genera (o propone) un plan para la feature, el plugin está cargado.

#### Paso 5 — Primer run de prueba (smoke test)

```bash
# Desde la raíz de tu proyecto Flutter
claude -p "/qa-run login --dry-run"
```

Si el dry-run responde con la configuración leída del `qa-plugin-config/qa-agent.yaml`, la instalación es correcta.

---

### Opción B: Actualización desde versión anterior

#### Paso 1 — Hacer backup de tu `qa-plugin-config/qa-agent.yaml`

```bash
cp qa-plugin-config/qa-agent.yaml qa-plugin-config/qa-agent.yaml.bak
```

#### Paso 2 — Hacer pull de la última versión del plugin

```bash
cd %USERPROFILE%\.claude\plugins\local\qa-flutter   # Windows
# o
cd ~/.claude/plugins/local/qa-flutter               # Linux/macOS

git fetch origin
git pull origin main
```

Si hay cambios locales que entran en conflicto:

```bash
git stash
git pull origin main
git stash pop
```

#### Paso 3 — Verificar cambios en el esquema de configuración

Revisar si el `CHANGELOG.md` o `docs/roadmap-integration.md` indica nuevos campos en `qa-plugin-config/qa-agent.yaml`.

En la versión actual, los campos **nuevos** disponibles son:

```yaml
# NUEVO — bloque planning (qa-flutter-test-planner)
planning:
  enabled: false
  test_plan_dir: "qa-plugin-config/qa-plans/"
  require_plan: false
```

Si el campo `planning` no existe en tu `qa-agent.yaml` actual, no es necesario añadirlo — es retrocompatible (`enabled: false` por defecto).

Los siguientes campos son reservados para próximas versiones (G1/G2):

```yaml
# RESERVADOS — añadir cuando hagas upgrade a G1/G2
autonomous:
  backend:
    start_cmd: ""
    start_cwd: ""
    health_url: ""
    ready_timeout_seconds: 120
    stop_cmd: ""
    stop_timeout_seconds: 30
  device:
    boot_avd: ""
    boot_timeout_seconds: 90
    create_if_missing: false
```

Si el campo `autonomous` no existe en tu `qa-plugin-config/qa-agent.yaml` actual, no es necesario añadirlo todavía — es retrocompatible.

#### Paso 4 — Comparar tu `qa-plugin-config/qa-agent.yaml` con el nuevo esquema

```bash
diff qa-plugin-config/qa-agent.yaml.bak qa-plugin-config/qa-agent.yaml
```

Si el archivo no cambió (solo se actualiza el plugin, no el yaml), no hay nada que migrar.

#### Paso 5 — Reiniciar Claude Code

Después de un `git pull`, Claude Code debe recargar el plugin:

```bash
claude --reload-plugins
```

O simplemente cerrar y volver a abrir la sesión de Cowork/Claude Code.

#### Paso 6 — Verificar que los skills siguen disponibles

```
/qa-plan <feature>
```

Si el comando se reconoce, los skills del plugin siguen disponibles tras la actualización.

#### Paso 7 — Smoke test post-upgrade

```bash
claude -p "/qa-run login --dry-run"
```

Si el resultado es el mismo que antes del upgrade (sin errores nuevos), la actualización fue exitosa.

---

### Configurar el autoaprendizaje (qa-knowledge.yaml)

El autoaprendizaje requiere un archivo `qa-plugin-config/qa-knowledge.yaml` en el proyecto Flutter.

#### Paso 1 — Copiar el template al proyecto

```bash
# Desde la raíz de tu proyecto Flutter
cp ~/.claude/plugins/local/qa-flutter/templates/qa-knowledge.yaml ./qa-plugin-config/qa-knowledge.yaml
```

El template incluye 7 entradas pre-configuradas para los errores más comunes (emulador no iniciado, puerto ocupado, build de Gradle fallido, etc.).

#### Paso 2 — Revisar y ajustar las entradas del template

Abrir `qa-plugin-config/qa-knowledge.yaml` y verificar:
- `preflight_check.command` es compatible con el entorno del proyecto
- `solution.steps` usan las rutas y comandos correctos para el proyecto
- El nombre del AVD en `K001` coincide con el AVD del proyecto (`QA_AVD_NAME`)

#### Paso 3 — Opcional: ajustar la URL del backend en K003

```yaml
# En K003, cambiar la URL del health check si es distinta a la default
preflight_check:
  command: "curl -sf http://localhost:TU_PUERTO/tu/health -o /dev/null; echo $?"
```

O bien, usar la variable de entorno:

```bash
export QA_BACKEND_HEALTH="http://localhost:9090/api/ping"
```

#### Paso 4 — Versionar el archivo

```bash
git add qa-plugin-config/qa-knowledge.yaml
git commit -m "chore: agregar base de conocimiento QA (qa-knowledge.yaml)"
```

El archivo debe versionarse con el proyecto. Cada error resuelto y registrado por el agente o por el equipo queda en el historial de git.

#### Comandos de gestión de conocimiento

```bash
# Ver estado de la base de conocimiento
claude -p "/qa-knowledge-manager report"

# Registrar manualmente un error y su solución
claude -p "/qa-knowledge-manager register"

# Solo ejecutar el preflight (sin correr tests)
claude -p "/qa-knowledge-manager preflight"

# Run completo con preflight + learn automático
claude -p "/qa-stability-agent"
```

---

### Solución de problemas comunes

| Síntoma | Causa probable | Solución |
|---|---|---|
| `skill not found: qa-flutter-manual-runner` | Plugin no instalado | Verificar la instalación según la sección Installation del README |
| `qa-agent.yaml not found` | YAML no creado o path incorrecto | Debe estar en `qa-plugin-config/qa-agent.yaml`, en la raíz del proyecto Flutter |
| Backend timeout en pre-flight | Backend no levantado | Levantar el backend manualmente antes del run (G1 aún no implementado) |
| `No devices found` | Emulador no arriba | Lanzar el emulador con `emulator -avd Pixel_5_API_33` |
| `flutter test` falla al instalar | Versión de Flutter incompatible | `flutter upgrade` y verificar `flutter doctor` |
| Skills desaparecen tras `git pull` | Conflict en settings.json | Verificar que el path del plugin sigue siendo correcto |
| `qa-knowledge.yaml not found` | Template no copiado al proyecto | `cp ~/.claude/plugins/local/qa-flutter/templates/qa-knowledge.yaml ./qa-plugin-config/` |
| Preflight aplica solución pero el error persiste | Comando de solución incorrecto para el entorno | Editar `solution.steps` en la entrada correspondiente o añadir `auto_apply: false` |
| El agente registra demasiados falsos positivos | Patrón regex muy genérico | Editar el `pattern` de la entrada para que sea más específico |
| `⚠️ PLANES FALTANTES` al correr el agente | `planning.enabled: true` pero no se generaron planes | Correr `/qa-plan <feature>` para cada feature listada, luego reintentar |
| El agente aborta con NO-GO por plan faltante | `planning.require_plan: true` y feature sin plan | Igual: correr `/qa-plan <feature>` primero, o cambiar `require_plan: false` |

---

### Referencia rápida de comandos

```bash
# Run de QA completo con autoaprendizaje (recomendado)
claude -p "/qa-stability-agent"

# Run de QA completo sin interacción (CI)
claude -p "/qa-stability-agent --auto"

# Solo el preflight de conocimiento (verificar entorno antes del run)
claude -p "/qa-knowledge-manager preflight"

# Ver estado de la base de conocimiento
claude -p "/qa-knowledge-manager report"

# Registrar manualmente un error/solución
claude -p "/qa-knowledge-manager register"

# Solo unit tests
claude -p "/qa-flutter-unit-generator"

# Release gate (GO/NO-GO)
claude -p "/qa-flutter-release-gate"

# Run Android con flutter_drive
claude -p "/qa-flutter-manual-runner"

# Run Android con Appium
claude -p "/qa-flutter-android-runner"

# Generar plan de test para una feature (interactivo)
claude -p "/qa-plan login"

# Generar plan sin prompts (CI / batch)
claude -p "/qa-plan login --auto"

# Correr con un plan específico (sin stability-agent)
claude -p "/qa-run login --plan=qa-plans/login.md"
```

---

*Documento actualizado: 2026-05-02 — Roadmap v2.1+planning: G8 (autoaprendizaje) implementado; qa-flutter-test-planner integrado (Step 2.5). 7 gaps pendientes, G1 sigue siendo crítico.*
