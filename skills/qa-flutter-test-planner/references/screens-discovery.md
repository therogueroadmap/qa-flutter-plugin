# Screens discovery — multi-layer playbook

This is the canonical procedure used by `qa-flutter-test-planner` Step 3 to find screens relevant to a feature in a Flutter project. It is a **generalization** of `qa-flutter-manual-runner` § C.1 that returns *all relevant screens* (for multi-screen plans) rather than a single top candidate (which the runner needed for single-test generation).

The runners (`qa-flutter-manual-runner`, `qa-flutter-android-runner`, `qa-flutter-web-runner`) keep their existing logic intact. This document exists so the planner mirrors their conventions — when a project changes its layout (e.g. moves pages from `lib/src/pages/` to `lib/features/<x>/presentation/`), update both this playbook and the runner sections C.1.

## Inputs

- `FEATURE` — lowercase slug
- `PLATFORM` — `"android"` or `"web"`
- Project root — discovered via `pubspec.yaml` walk-up from `qa-agent.yaml`'s directory

## Conventions Flutter projects in scope

Plans are generated against projects that follow at least one of these layouts:

| Layout | Pages directory | Providers/Bloc directory | Router |
|---|---|---|---|
| Classic (manual-runner default) | `lib/src/pages/`, `lib/core/` | `lib/src/providers/`, `lib/src/bloc/` | `lib/config/app_router.dart` |
| Feature-first | `lib/features/<x>/presentation/pages/` | `lib/features/<x>/data/`, `lib/features/<x>/state/` | `lib/router/` or `lib/app_router.dart` |
| Web (web-runner default) | `lib/pages/` | `lib/state/`, `lib/notifiers/`, `lib/services/` | `lib/config/` |

The planner tries all layouts in order; the first that yields ≥1 candidate wins. If none match, it asks the user where pages live (interactive mode) or falls back to a recursive `lib/**/*.dart` glob (auto mode).

## Layer 1 — Pages

### What to read

Glob (in order, stop at first non-empty result):
1. `lib/src/pages/**/*.dart` + `lib/core/**/*.dart`
2. `lib/features/**/presentation/pages/**/*.dart` + `lib/features/**/presentation/screens/**/*.dart`
3. `lib/pages/**/*.dart` + `lib/screens/**/*.dart`

For each `.dart` file, parse:
- All `class XxxPage extends ...` and `class XxxScreen extends ...` declarations
- All `import` statements pointing to `lib/**/providers/`, `lib/**/bloc/`, `lib/**/state/`, `lib/**/services/`

### Scoring rules

For each `(file, class)` pair, compute a 0–100 score:

| Signal | Points |
|---|---|
| Class name (snake-cased) contains FEATURE keywords | **+40** |
| File path (snake-cased) contains FEATURE keywords | **+15** |
| Imports a provider/bloc whose name contains FEATURE keywords | **+20** |
| Imported provider has an HTTP method call to URL containing FEATURE | **+35** (Layer 2 backref) |
| Class is a `StatefulWidget` or `ConsumerStatefulWidget` (real screen, not pure UI) | **+10** |
| Class appears as a `MaterialPageRoute(builder: (_) => XxxPage())` or `GoRoute(builder: ...)` target | **+25** |

Cap at 100. Keep all candidates with score ≥ 40 (configurable via `planning.score_threshold`).

### Keyword extraction from FEATURE

Split FEATURE on `-`, `_`, ` `. Filter stop-words (`the`, `a`, `de`, `el`, `la`, `un`, `una`). Lowercase. Compare case-insensitively, stem-aware (e.g. "registro" matches both "Register" and "Registration", "login" matches "logIn" and "LogInPage").

## Layer 2 — API endpoints

### What to grep

In order:
1. `lib/src/providers/**/*.dart` + `lib/src/bloc/**/*.dart`
2. `lib/features/**/data/**/*.dart`
3. `lib/services/**/*.dart` + `lib/notifiers/**/*.dart`

Patterns to match:
- `(?:http|dio|client)\.(get|post|put|delete|patch)\s*\(` — direct HTTP calls
- `\.(get|post|put|delete|patch)\s*\(\s*['"]([^'"]+)['"]` — capture URL string

### Classification

For each match:
- **Verb**: `GET` → read-only; `POST`/`PUT`/`PATCH`/`DELETE` → **MUTANT** (creates/modifies backend data).
- **Endpoint URL**: extract the path string. Match against FEATURE keywords case-insensitive.
- **Parent class**: walk up to the enclosing class declaration; that's the provider/bloc that owns the call.

### Backref to Layer 1

For each Layer 1 candidate page, find the import line that pulls in the provider class identified above. Mark the page as a consumer of that endpoint. Use this backref to award the **+35** bonus in Layer 1 scoring.

## Layer 3 — Router

### What to read

Try in order:
1. `lib/config/app_router.dart`
2. `lib/router/router.dart` + `lib/router/app_router.dart`
3. `lib/app_router.dart` + `lib/main.dart` (router defined inline)

### What to extract

Parse the file for route definitions. Common patterns:

**GoRouter:**
```dart
GoRoute(path: '/login', builder: (_, __) => LoginPage())
GoRoute(path: '/home', builder: ..., routes: [GoRoute(path: 'profile', ...)])
```

**Named routes:**
```dart
'/login': (_) => LoginPage(),
'/home': (_) => HomePage(),
```

**MaterialApp.routes / onGenerateRoute:**
```dart
case '/login': return MaterialPageRoute(builder: (_) => LoginPage());
```

For each route definition produce: `{path, page_class, file}`.

### Build transitions

For each candidate page from Layer 1:
- `entry_route` = the path mapped to that page
- `next_routes` = paths reachable from this page (search the page's source for `Navigator.push*`, `context.go(...)`, `context.push(...)`, `Navigator.pushReplacement*`)

Resolve `next_routes` by URL string match against the route registry. Cap at depth 3 to keep flow inference bounded.

## Output

The planner consolidates Layers 1+2+3 into a `SCREENS` list:

```yaml
- page_class: LoginPage
  page_file: lib/src/pages/auth/login_page.dart
  entry_route: /login
  next_routes: ["/home", "/forgot-password"]
  score: 95
  endpoints:
    - {verb: POST, url: /api/auth/login, mutant: true}
  MUTANT: true
- page_class: HomePage
  page_file: lib/src/pages/home_page.dart
  entry_route: /home
  next_routes: ["/profile", "/transactions"]
  score: 60
  endpoints: []
  MUTANT: false
```

Sort descending by `score`. Pass to Step 4 (flow inference) which walks `next_routes`.

## Edge cases

| Situation | Handling |
|---|---|
| Project has no router file | Layer 3 returns empty. `next_routes` for all candidates = `[]`. Step 4 will produce no flows; Step 7 (risks) records "navegación no inferible". |
| Multiple pages match equally well | Keep all above threshold. The plan will list them separately and downstream runner can test each. |
| Provider used by 5+ pages | Avoid double-counting; the +35 bonus is awarded once per (page, endpoint) pair. |
| FEATURE matches no Dart classes | Try aliases declared in `qa-agent.yaml` under `planning.aliases.{feature}: [synonym1, synonym2]`. If still empty → abort Step 3. |
| Generated `*.g.dart` / `*.freezed.dart` files | Skip — these are codegen artifacts, not source pages. |
| Test files (`test/**`, `integration_test/**`) | Skip — we're discovering production code, not test code. |

## When to update this playbook

If a project's layout deviates from the supported set, update:
1. The "Layouts in scope" table above
2. The Layer 1 glob order
3. The Layer 2 grep targets
4. `qa-flutter-manual-runner/SKILL.md` § C.1 (so the on-the-fly resolution stays consistent)

Do not let this playbook and § C.1 drift apart — both must accept the same project layouts.
