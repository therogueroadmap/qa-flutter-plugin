# Preconditions checklist — canonical list

`qa-flutter-test-planner` Step 5 inspects the project to populate the **Precondiciones** section of the plan. This document is the canonical reference for what counts as a precondition, where each is detected from, and whether it is a **blocker** (`[ ]` — must be checked off before the runner can succeed) or **advisory** (`[~]` — runner can proceed but result may be partial).

## Categories

### 1. Variables de entorno (env vars)

| Precondition | Detection source | Severity |
|---|---|---|
| `TEST_EMAIL` set in `qa-plugin-config/.env` | `String.fromEnvironment('TEST_EMAIL')` in any page or provider | **Blocker** if any auth-protected route exists |
| `TEST_PASSWORD` set in `qa-plugin-config/.env` | Same as above | **Blocker** if `TEST_EMAIL` is blocker |
| `API_BASE_URL` set in `qa-plugin-config/.env` | `String.fromEnvironment('API_BASE_URL')` in providers | **Blocker** always |
| Custom env var (e.g. `FEATURE_FLAG_X`) | Any other `String.fromEnvironment('X')` call | **Blocker** if no default value, **Advisory** otherwise |

### 2. Backend

| Precondition | Detection source | Severity |
|---|---|---|
| `backend.test_url` configured in `qa-agent.yaml` | Read yaml | **Blocker** if any HTTP call exists in providers |
| `API_BASE_URL` matches `backend.test_url` | Cross-check `qa-plugin-config/.env` vs yaml | **Blocker** — production-data protection |
| Backend reachable (`GET {test_url}/health` returns 200) | Implicit; runner pre-flight enforces | **Blocker** |
| Backend running in **test profile** (not production) | Inferred from `backend.test_url` containing `localhost`, `test`, `staging` | **Blocker** if plan is `data_mutating: true` |

### 3. Usuarios y datos seed

| Precondition | Detection source | Severity |
|---|---|---|
| Test user with valid credentials exists | Auth-protected routes detected (router guard, `isAuthenticated` check) | **Blocker** if any flow starts at protected route |
| Test user has role/permission X | Role check in provider (e.g. `if (user.role != 'admin')`) | **Blocker** if flow requires the role |
| Seed data — non-empty list | Page renders empty state (`if (items.isEmpty) return EmptyState()`) AND a flow expects the loaded state | **Advisory** — flow may silently fall into empty branch |
| Seed data — specific record (e.g. transaction with id 42) | Hardcoded id in flow steps | **Blocker** if step references the id |

### 4. Dispositivo (Android only)

| Precondition | Detection source | Severity |
|---|---|---|
| Device or emulator connected (`adb devices` shows `device`) | Implicit; runner Section 3.3 enforces | **Blocker** |
| AVD with API level X exists | `pubspec.yaml` `flutter:` section / `android/app/build.gradle` `minSdkVersion` | **Advisory** — runner aborts if missing |
| Permission CAMERA granted | `<uses-permission android:name="android.permission.CAMERA"/>` in `AndroidManifest.xml` AND a flow uses camera | **Blocker** for affected flow |
| Permission ACCESS_FINE_LOCATION granted | Same pattern | **Blocker** for affected flow |
| Permission RECORD_AUDIO / READ_EXTERNAL_STORAGE / etc. | Same pattern | **Blocker** for affected flow |
| Notification permission (Android 13+) | `permission_handler` package + flow shows notification | **Advisory** |

### 5. Dispositivo (Web only)

| Precondition | Detection source | Severity |
|---|---|---|
| Chrome installed | Implicit; web-runner Step 5 (`flutter build web`) needs it | **Blocker** |
| `flutter build web` succeeds | web-runner Step 5 | **Blocker** |
| Web base URL reachable | `project.web.base_url` from yaml; CORS reachable | **Blocker** for E2E flows |
| Auth credentials provided in yaml (`project.auth`) | Yaml `auth.email` / `auth.password` | **Blocker** if flows are auth-gated |
| `ensureSemantics()` en `main()` | `grep "ensureSemantics" lib/main.dart` (NEW v1.5) | **Blocker** para `platform: web` — sin esto el a11y tree está vacío y Playwright no encuentra widgets. Fix: agregar `WidgetsFlutterBinding.ensureInitialized().ensureSemantics()` antes de `runApp()`. Ver K001 en qa-knowledge.yaml. |

### 6. Dependencias nativas

For each package in `pubspec.yaml` matching the table below, surface as **Advisory** in Step 5 with a one-line setup note:

| Package | Setup note |
|---|---|
| `firebase_*` | `flutterfire configure` ejecutado; `google-services.json` presente en `android/app/` |
| `google_maps_flutter` | API key en `AndroidManifest.xml` (`com.google.android.geo.API_KEY`) |
| `permission_handler` | Permisos declarados en `AndroidManifest.xml` y `Info.plist` |
| `local_auth` | `usesCleartextTraffic` y configuración biométrica del device |
| `image_picker` | Permisos READ_MEDIA_IMAGES (Android 13+), galería con al menos 1 imagen seed |
| `camera` | Permiso CAMERA + dispositivo con cámara funcional (emuladores web no la tienen) |
| `flutter_secure_storage` | Keychain disponible (Android: Keystore; iOS: Keychain) |
| `webview_flutter` | URL externa alcanzable; CORS no bloquea |

### 7. Flags / feature toggles

If the project uses a feature-flag system (e.g. Firebase Remote Config, GrowthBook, env var `FF_*`), and the feature being planned is gated behind one:

- **Blocker**: flag must be enabled for the test user/environment.
- Surface in plan as: `[ ] Feature flag '{flag_name}' habilitado en backend de test`.

Detection: search for `RemoteConfig.getValue('X')`, `growthbook.isOn('X')`, `dotenv.env['FF_X']` in pages/providers related to the feature.

## Severity rules summary

| Marker | Meaning | Runner behavior |
|---|---|---|
| `[ ]` (blocker) | Must be true before runner can proceed | If the runner is invoked with `--plan` and any blocker is unchecked AND `planning.require_plan_blockers_resolved: true`, abort with the list of unmet blockers. Default behavior: warn but proceed. |
| `[~]` (advisory) | Runner can proceed; result may be partial or noisy | Runner logs the advisory and continues. Surfaced in the report's "Pendientes para corrección" section if the test fails. |

## How the planner decides severity

For each precondition, severity is computed from:
1. The detection rule above (default).
2. Whether any flow in the plan **depends on it being true** (escalates to blocker).
3. The yaml `planning.precondition_severity_overrides` map (user override).

Example override in `qa-agent.yaml`:
```yaml
planning:
  precondition_severity_overrides:
    "Permiso CAMERA": "advisory"   # downgrade to advisory; user accepts the risk
    "Seed: lista no vacía": "blocker"  # upgrade because we know the empty path is broken
```

## When to extend this checklist

Add a new row when:
- A common pattern of test failure traces back to an unstated precondition (e.g. timezone of the device, locale, dark mode).
- A new package is widely adopted by the project family and needs setup notes.
- A new env var becomes mandatory across features.

Do **not** add: project-specific preconditions (e.g. "user '`qa-test-42`' must exist"). Those go in the plan markdown manually, not in this canonical list.
