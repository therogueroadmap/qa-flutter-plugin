# State-management test templates

Use when the selected candidate is a state-management class. Branch by `STATE_STACK` detected in pre-flight.

## BLoC / Cubit — uses `bloc_test`

Target: `test/{relative_path}/<name>_bloc_test.dart`

```dart
import 'package:bloc_test/bloc_test.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:{APP_PACKAGE_NAME}/{relative_import}';

class Mock{Dep} extends Mock implements {Dep} {}

void main() {
  late {BlocClass} sut;
  late Mock{Dep} mock{Dep};

  setUp(() {
    mock{Dep} = Mock{Dep}();
    sut = {BlocClass}({mock_injected});
  });

  tearDown(() => sut.close());

  test('Given fresh bloc, Then initial state is {InitialState}', () {
    expect(sut.state, equals({InitialState}()));
  });

  blocTest<{BlocClass}, {StateType}>(
    'Given success, When {Event} added, Then emits [Loading, Loaded]',
    build: () {
      when(() => mock{Dep}.{method}(any())).thenAnswer((_) async => {fixture});
      return sut;
    },
    act: (b) => b.add({Event}({args})),
    expect: () => [{LoadingState}(), {LoadedState}({fixture})],
    verify: (_) { verify(() => mock{Dep}.{method}(any())).called(1); },
  );

  blocTest<{BlocClass}, {StateType}>(
    'Given dependency fails, When {Event} added, Then emits [Loading, Error]',
    build: () {
      when(() => mock{Dep}.{method}(any())).thenThrow(Exception('boom'));
      return sut;
    },
    act: (b) => b.add({Event}({args})),
    expect: () => [{LoadingState}(), isA<{ErrorState}>()],
  );
}
```

## Riverpod

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:mocktail/mocktail.dart';

class Mock{Dep} extends Mock implements {Dep} {}

void main() {
  late ProviderContainer container;
  late Mock{Dep} mock{Dep};

  setUp(() {
    mock{Dep} = Mock{Dep}();
    container = ProviderContainer(overrides: [
      {depProvider}.overrideWithValue(mock{Dep}),
    ]);
  });

  tearDown(() => container.dispose());

  test('Given success, When provider read, Then returns value', () async {
    when(() => mock{Dep}.{method}(any())).thenAnswer((_) async => {fixture});
    final result = await container.read({featureProvider}.future);
    expect(result, equals({fixture}));
  });

  test('Given failure, When provider read, Then throws', () async {
    when(() => mock{Dep}.{method}(any())).thenThrow(Exception('boom'));
    expect(() => container.read({featureProvider}.future), throwsA(isA<Exception>()));
  });
}
```

**Principle — never mock the provider itself.** Always override its dependencies via `ProviderContainer(overrides: [...])`.

## Provider / ChangeNotifier

```dart
void main() {
  late {NotifierClass} sut;
  late Mock{Dep} mock{Dep};

  setUp(() {
    mock{Dep} = Mock{Dep}();
    sut = {NotifierClass}({mock_injected});
  });

  tearDown(() => sut.dispose());

  test('Given initial state, Then defaults are set', () {
    expect(sut.{prop}, equals({default_value}));
  });

  test('Given success, When {method}() called, Then state updates and notifies', () async {
    when(() => mock{Dep}.{method}(any())).thenAnswer((_) async => {fixture});
    var notifications = 0;
    sut.addListener(() => notifications++);

    await sut.{method}({args});

    expect(sut.{prop}, equals({expected}));
    expect(notifications, greaterThan(0));
  });
}
```

## GetX

```dart
void main() {
  late {ControllerClass} sut;
  late Mock{UseCase} mock{UseCase};

  setUp(() {
    mock{UseCase} = Mock{UseCase}();
    sut = {ControllerClass}({mock_injected});
  });

  tearDown(() => Get.reset());

  test('Given success, When {method}() called, Then reactive value updates', () async {
    when(() => mock{UseCase}(any())).thenAnswer((_) async => {fixture});
    await sut.{method}({args});
    expect(sut.{observable}.value, equals({fixture}));
  });
}
```

## setState

Skip this layer. Log:
```
⚠ STATE_STACK=setstate — saltando capa de estado. Los tests de widget la cubrirán.
```

## Principles across all stacks

1. **Dispose in tearDown:** `sut.close()` (BLoC), `container.dispose()` (Riverpod), `sut.dispose()` (ChangeNotifier), `Get.reset()` (GetX).
2. **Given-When-Then naming** in every test/blocTest description.
3. **Always cover both success and failure paths.**
4. **Riverpod specifically** — override dependencies, never mock the provider.
5. **Initial state test** — always verify defaults before testing transitions.
