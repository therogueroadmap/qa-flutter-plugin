# Widget test template

Use when the selected candidate is a widget in `lib/**/*_page.dart`, `lib/**/*_screen.dart`, `lib/**/*_view.dart`, or `lib/src/pages/**`.

## Analysis before generating

Read the source file. Extract:
- `WIDGET_CLASS` — main widget class name
- `WIDGET_ARGS[]` — required constructor args
- `KEYS[]` — all `const Key('...')` declarations. If the widget has fewer than 2 keyed interactive elements, emit this warning in the report:

```
⚠ {WIDGET_CLASS} tiene pocos Keys — los finders quedarán acoplados a texto. Considera añadir Keys.
```

## Template (framework-agnostic, wrapper branches by STATE_STACK)

Target: `test/{relative_path}/<widget_name>_test.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
{state-specific imports}
import 'package:{APP_PACKAGE_NAME}/{widget_import}';

{mock declarations for state container}

void main() {
  {state-specific setUp}

  setUp(() {
    TestWidgetsFlutterBinding.ensureInitialized();
  });

  Widget makeSut() {
    return {state_specific_wrapper}(
      child: MaterialApp(home: {WIDGET_CLASS}({widget_args})),
    );
  }

  testWidgets('Given initial render, Then key elements appear', (tester) async {
    tester.view.physicalSize = const Size(1080, 1920);
    tester.view.devicePixelRatio = 1.0;
    addTearDown(tester.view.resetPhysicalSize);
    addTearDown(tester.view.resetDevicePixelRatio);

    await tester.pumpWidget(makeSut());
    await tester.pumpAndSettle();

    {for each KEY:} expect(find.byKey(const Key('{key}')), findsOneWidget);
  });

  testWidgets('Given loading state, When rendered, Then progress indicator shown', (tester) async {
    {arrange loading state via mock/notifier};
    await tester.pumpWidget(makeSut());
    await tester.pump(); // intentionally no settle — still loading
    expect(find.byType(CircularProgressIndicator), findsOneWidget);
  });

  testWidgets('Given error state, Then error UI shown', (tester) async {
    {arrange error state};
    await tester.pumpWidget(makeSut());
    await tester.pumpAndSettle();
    {assert error finder};
  });

  {if GOLDEN:}
  testWidgets('Given happy path, Then matches golden', (tester) async {
    tester.view.physicalSize = const Size(1080, 1920);
    tester.view.devicePixelRatio = 1.0;
    await tester.pumpWidget(makeSut());
    await tester.pumpAndSettle();
    await expectLater(find.byType({WIDGET_CLASS}), matchesGoldenFile('goldens/{feature}.png'));
  });
}
```

## Wrappers by STATE_STACK

| Stack | Wrapper |
|---|---|
| `bloc` | `BlocProvider<{Bloc}>.value(value: mockBloc, child: ...)` |
| `riverpod` | `ProviderScope(overrides: [...], child: ...)` |
| `provider` | `ChangeNotifierProvider<{Notifier}>.value(value: fakeNotifier, child: ...)` |
| `getx` | `GetMaterialApp` + `Get.put<{Controller}>(mock)` in `setUp`; `Get.reset()` in `tearDown` |
| `setstate` | no wrapper — just `MaterialApp` |

## Principles enforced

1. **Set `physicalSize` + `devicePixelRatio`** with `addTearDown(reset)`. Prevents layout flakiness.
2. **Prefer `find.byKey`** over `find.text` when keys exist. Keys are stable across i18n and text refactors.
3. **Never `await Future.delayed(...)`.** Use `pump()` (intentional single-frame), `pumpAndSettle()` (wait for animations), or `Completer` (controlled async).
4. **Reset `debugDefaultTargetPlatformOverride = null`** at test end if the test mutated it.
5. **Cover initial/loading/error** — always at least these three states.
6. **Dispose / reset** the state container in tearDown per stack (BLoC close, Get.reset, etc.).
