# qa-flutter-manual-runner

Functional QA regression suite for Flutter Android apps, invocable from Claude Code.

---

## What it does

A single command — `/qa-run <feature>` — covers three scenarios:

| Invocación | Comportamiento |
|---|---|
| `/qa-run login` | Corre el test existente (regresión), o genera y persiste uno nuevo si no existe |
| `/qa-run login --auto` | Igual, sin interacción con el usuario |
| `/qa-run regresion` | Corre todos los tests en `integration_test/manual/` en secuencia |
| `/qa-run regresion --auto` | Suite completa sin interacción (modo CI) |

---

## Prerequisites

- Flutter SDK `^3.10.0`
- ADB instalado y en el PATH
- Android emulator o dispositivo físico conectado
- Python 3 (para validación YAML — `python -c "import yaml"`)
- Claude Code CLI

---

## Setup (one-time, per project)

### 1. Instalar el plugin

Clonar o copiar `qa-flutter-plugin/` en la raíz del proyecto Flutter. Verificar que el directorio tiene la estructura:

```
qa-flutter-plugin/
  .claude-plugin/
    plugin.json
  skills/
    qa-flutter-manual-runner/
      SKILL.md
```

### 2. Crear `qa-plugin-config/qa-agent.yaml`

Crear el directorio `qa-plugin-config/` en la raíz del proyecto y copiar esta plantilla:

```yaml
device:
  id: emulator-5554              # ID del dispositivo ADB (adb devices)
  app_package: com.example.app   # applicationId del build.gradle

timeouts:
  test_seconds: 120              # Timeout por test antes de kill
  adb_retry_count: 2             # Reintentos ante pérdida de ADB
  adb_retry_wait_seconds: 3

execution:
  parallel: false                # Siempre false en v1
  reset_app_before_each: true    # pm clear antes de cada test

backend:
  test_url: "http://localhost:8080/api"  # REQUERIDO — URL del entorno de test
                                         # API_BASE_URL en .env debe coincidir

reports:
  output_dir: qa-plugin-config/qa-reports
```

> **`backend.test_url` es obligatorio.** El skill no ejecuta si este campo está vacío.
> El pre-flight verifica que `API_BASE_URL` en `.env` coincida exactamente con este valor,
> protegiendo datos de producción ante errores de configuración.

### 3. Crear `.env`

```
API_BASE_URL=http://localhost:8080/api
TEST_EMAIL=qa-user@test.com
TEST_PASSWORD=your-test-password
```

> `.env` nunca se commitea. Agregar a `.gitignore` si no está ya.

### 4. Agregar a `.gitignore`

```
.env
qa-plugin-config/qa-reports/
```

### 5. Añadir `integration_test` a `pubspec.yaml`

```yaml
dev_dependencies:
  integration_test:
    sdk: flutter
  flutter_test:
    sdk: flutter
```

Luego: `flutter pub get`

### 6. Crear el entry point de flutter drive

Crear `integration_test/fixtures/test_runner.dart`:

```dart
import 'package:integration_test/integration_test.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
}
```

---

## Usage

### Testear una feature existente (regresión individual)

```
/qa-run login
```

El skill busca `integration_test/manual/login_test.dart`. Si existe, lo ejecuta con `flutter drive` y genera un informe en `qa-plugin-config/qa-reports/`.

### Testear una feature nueva

```
/qa-run crear-transaccion
```

Si no existe `integration_test/manual/crear-transaccion_test.dart`, el skill:

1. Analiza semánticamente `lib/src/pages/` y `lib/src/providers/` para encontrar la implementación
2. Presenta candidatos (en modo interactivo) o elige automáticamente (en modo `--auto`)
3. Genera un test Dart con happy path + caso de error
4. Lo ejecuta con credenciales reales via `--dart-define`
5. Si PASS → persiste el test en `integration_test/manual/`
6. Si FAIL → descarta el archivo y genera el informe con los hallazgos

### Suite de regresión completa

```
/qa-run regresion
```

Ejecuta todos los `*_test.dart` en `integration_test/manual/` en orden alfabético. Al final genera un `summary.md` consolidado.

### Modo automático (CI / scheduler)

```
/qa-run regresion --auto
```

Sin interacción. Para invocar desde GitHub Actions, Task Scheduler o cron:

```bash
claude -p "/qa-run regresion --auto" --output-format json
```

---

## Pre-flight checks

Antes de ejecutar cualquier test, el skill verifica en este orden:

1. **`.env` completo y seguro** — campos requeridos presentes, `API_BASE_URL` coincide con `backend.test_url`
2. **Backend disponible** — `GET /health`, si no existe: `POST /login` con las credenciales del `.env`
3. **Dispositivo activo** — si no hay dispositivo, intenta levantar el primer AVD disponible (timeout 90s)
4. **`test_runner.dart` existe** — lo crea automáticamente si falta

Cualquier falla no recuperable aborta el run con mensaje claro antes de ejecutar un solo test.

---

## Generated test structure

Los tests generados usan credenciales via `--dart-define`, nunca hardcodeadas:

```dart
// qa:creates-backend-data  ← presente si el test llama POST/PUT/DELETE
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:com.example.app/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  const email = String.fromEnvironment('TEST_EMAIL');
  const password = String.fromEnvironment('TEST_PASSWORD');

  group('login — QA Suite', () {
    testWidgets('happy path', (tester) async {
      app.main();
      await tester.pumpAndSettle();
      // pasos generados por análisis de la página
    });

    testWidgets('error case — validación', (tester) async {
      app.main();
      await tester.pumpAndSettle();
      // caso de error generado
    });
  });
}
```

---

## Report format

Cada test genera un informe individual en `qa-plugin-config/qa-reports/`:

```
YYYY-MM-DDTHH-MM-<feature>.md      ← informe por feature
YYYY-MM-DDTHH-MM-summary.md        ← solo en modo regresion
```

El informe incluye: resultado (✅ PASS / ⚠ PARCIAL / ❌ FAIL), duración, versión de la app, pasos ejecutados, y pendientes para corrección si hay fallos.

---

## Error recovery

| Escenario | Comportamiento |
|---|---|
| Token expirado mid-test | `pm clear` + reintento automático (máx. 1 vez) |
| Test supera timeout | Kill del proceso, informe con ❌ TIMEOUT, continúa la suite |
| App crash | Captura últimas líneas de output, informe ❌ ERROR, continúa |
| Error de compilación del test generado | Descarta el archivo, informe ❌ COMPILE_ERROR, continúa |
| ADB pierde conexión | Reintentos configurables; si persiste → aborta el run completo |
| Backend caído | Aborta en pre-flight antes de tocar el emulador |

---

## Backend isolation (recommended)

Para tests que crean datos (marcados con `// qa:creates-backend-data`), se recomienda que el backend corra con un perfil de test sobre base de datos en memoria:

```yaml
# application-test.yml (Spring Boot)
spring:
  datasource:
    url: jdbc:h2:mem:qa_testdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
```

Con esta configuración, `API_BASE_URL` apunta al backend en perfil `test` y todos los datos creados se descartan al detener el servidor.

---

## File structure summary

```
<project-root>/
  .env                                   ← credenciales (excluido de git)
  qa-plugin-config/
    qa-agent.yaml                        ← config QA
    qa-reports/                          ← informes generados (excluidos de git)
  integration_test/
    fixtures/
      test_runner.dart                   ← entry point de flutter drive
    manual/
      login_test.dart                    ← tests persistidos (commiteados)
      checkout_test.dart
      ...
  qa-flutter-plugin/
    skills/
      qa-flutter-manual-runner/
        SKILL.md                         ← este skill
```
