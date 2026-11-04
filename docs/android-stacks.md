# Android E2E stacks — decision matrix

The qa-flutter plugin supports **two parallel stacks** for Android E2E testing. This document defines when to use each one.

Both run on real Android devices/emulators, both produce Markdown reports under `qa-reports/`. The difference is **how** they drive the app.

> **Default since qa-flutter-bootstrap (release X.Y):** `flutter_drive` is the default when `project.android_stack` is omitted. It has the lowest friction (no Python / no Appium server) and is the recommended path for teams starting with the plugin. Use Appium only when the `flutter_drive` column in the decision matrix below says ❌ for a requirement you need.
>
> Both stacks integrate with the autonomous orchestrator via `qa-flutter-bootstrap` — the yaml contract is unified in `autonomous.*`. See [README — Autonomous execution](../README.md#autonomous-execution).

## Stacks

### Stack A — `flutter_drive` (skill: `qa-flutter-manual-runner`)

- Uses the Flutter `integration_test` package + `flutter drive` as the runner.
- Test code is Dart, lives in `integration_test/manual/<feature>_test.dart`.
- Drives the app via Flutter's widget tree (`find.byKey`, `find.byType`).
- Runs inside the Flutter build — no external test framework.
- Bootstrap is a pre-flight: `.env` check, backend `/health`, ADB, `test_runner.dart`.

### Stack B — Appium (skills: `qa-flutter-android-runner` + `qa-flutter-android-tester`)

- Uses Appium UiAutomator2 driving an installed APK.
- Test steps are declarative JSON (`{action, key, text}`) executed by Python (`appium_runner.py`).
- Drives the app via Android accessibility + Flutter semantics bridge.
- Bootstrap via a user-supplied companion `bootstrap.py` / `teardown.py` (starts backend, launches Flutter build, boots Appium). The plugin does not ship these scripts; see [`references/appium-bootstrap-contract.md`](references/appium-bootstrap-contract.md).
- Requires a companion `qa-agent/` directory with Python scripts outside the Flutter project.

## Decision matrix

Pick the row that best matches your context. If multiple apply, column-weighted wins.

| Requirement / constraint | `flutter_drive` (manual-runner) | Appium (android-runner) |
|---|---|---|
| Dart test code you can commit and maintain | ✅ **pick this** | ❌ JSON steps only |
| Zero external dependencies (no Python, no Appium server) | ✅ **pick this** | ❌ needs Python + Appium |
| Access to Flutter widget state (`tester`, internal providers) | ✅ **pick this** | ❌ black-box |
| Gestos nativos complejos (multi-touch, swipe con velocidad, long-press coords) | ⚠️ limitado | ✅ **pick this** |
| Flujos multi-app (permission dialogs, intents a otras apps, deep links desde Chrome) | ❌ | ✅ **pick this** |
| Tests sobre APK pre-compilado (staging, release builds) | ❌ re-compila por cada run | ✅ **pick this** |
| Velocidad de feedback por run | ✅ **pick this** (30-90s) | ❌ 2-5min con bootstrap |
| CI sin display ni servidor Appium | ✅ **pick this** | ⚠️ requiere Appium headless |
| Tests que crean datos reales en el backend | ✅ tiene MUTANT tagging nativo | ⚠️ sin tagging — hay que gestionarlo manualmente |
| Generación automática de test nuevo desde semantic resolution del código | ✅ **pick this** (Section C de manual-runner) | ❌ no tiene generador |
| Equipo no familiarizado con Dart pero sí con Appium/Selenium | ❌ | ✅ **pick this** |
| Recuperación automática por expiración de token | ✅ **pick this** (retry nativo) | ❌ sin retry |
| Paralelización entre features | ❌ secuencial | ⚠️ secuencial por limitación del orquestador actual |
| Reusa keys declarados en widgets (`Key('btn-submit')`) | ✅ directamente | ✅ via semantics bridge |
| Portabilidad a iOS en el futuro | ⚠️ requiere adaptación | ✅ Appium soporta iOS con el mismo stack |

## Regla corta

```
¿Tests en Dart que lees/editas como código? ───── flutter_drive
¿Gestos nativos, multi-app o APK staging? ─────── Appium
¿Duda? ──────────────────────────────────────────  flutter_drive (menor fricción)
```

## Indicar la elección en `qa-agent.yaml`

```yaml
project:
  platform: "android"
  android_stack: "flutter_drive"   # o "appium"
```

`qa-stability-agent` lee este campo y rutea automáticamente. Si se omite, el default es `flutter_drive` (menor fricción, menos dependencias).

## Caminos de migración

### De Appium a flutter_drive

Si hoy usas Appium y quieres migrar:
1. Añadir `integration_test` al `pubspec.yaml`.
2. Usar `/qa-run <feature>` con `manual-runner` — genera el `_test.dart` desde el código.
3. Mantener el `steps-{feature}.json` viejo como referencia para validar paridad.
4. Descartar `qa-agent/scripts/*.py` cuando todos los tests estén migrados.

### De flutter_drive a Appium

Si hoy usas flutter_drive y necesitas Appium (ej. agregaste un flujo multi-app):
1. Instalar la companion `qa-agent/` directory con los scripts Python.
2. Mantener los tests flutter_drive para regresión rápida y los Appium para los flujos que no se pueden cubrir con flutter_drive.
3. En `qa-agent.yaml`, setear `android_stack: "appium"` pero invocar `manual-runner` directamente para los tests que sigan siendo flutter_drive. `qa-stability-agent` sólo rutea a uno — la convivencia es a mano.

## Anti-patrones

- **Duplicar un test en ambos stacks.** Elige uno por feature. Si necesitas ambos para la misma feature, es señal de que la feature está mal acotada.
- **Usar Appium "por si acaso" cuando flutter_drive alcanza.** Suma setup innecesario: servidor Appium, Python deps, bootstrap más lento.
- **Usar flutter_drive para flujos con permission dialogs nativos.** `flutter drive` no puede tocar UI fuera del canvas de Flutter — vas a chocar con esto rápido.
