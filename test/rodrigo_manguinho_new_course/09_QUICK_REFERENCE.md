# Quick Reference Cheatsheet

## Fake Data Utilities

```dart
// Primitives
int anyInt([int max = 999999999]) => Random().nextInt(max);
String anyString() => anyInt().toString();
bool anyBool() => Random().nextBool();
DateTime anyDate() => DateTime.fromMillisecondsSinceEpoch(anyInt());
String anyIsoDate() => anyDate().toIso8601String();

// Collections
Json anyJson() => { anyString(): anyString() };
JsonArr anyJsonArr() => List.generate(anyInt(5), (index) => anyJson());

// Entities
NextEvent anyNextEvent() => NextEvent(
  groupName: anyString(),
  date: anyDate(),
  players: anyNextEventPlayerList()
);
NextEventPlayer anyNextEventPlayer() => NextEventPlayer(
  id: anyString(),
  name: anyString(),
  isConfirmed: anyBool()
);
```

---

## Common Assertions

### Value Assertions

```dart
expect(value, equals(expected));
expect(value, isNull);
expect(value, isNotNull);
expect(value, isTrue);
expect(value, isFalse);
expect(value, isEmpty);
expect(value, isNotEmpty);
```

### Type Assertions

```dart
expect(value, isA<String>());
expect(value, isA<NextEvent>());
expect(value, isA<UnexpectedError>());
```

### Collection Assertions

```dart
expect(list.length, 3);
expect(list, contains(item));
expect(list, containsAll([a, b, c]));
expect(map, containsKey('key'));
expect(map, containsValue('value'));
```

### Error Assertions

```dart
expect(future, throwsA(isA<Error>()));
expect(future, throwsA(specificError));
expect(future, throwsA(isA<UnexpectedError>()));
expect(() => func(), throwsA(isA<Exception>()));
```

---

## Stream Assertions

```dart
// Single emission
expectLater(stream, emits(value));
expectLater(stream, emits(isA<Type>()));

// Error emission
expectLater(stream, emitsError(error));
expectLater(stream, emitsError(isA<Error>()));

// Sequence
expectLater(stream, emitsInOrder([true, false]));
expectLater(stream, emitsInOrder([
  isA<Loading>(),
  isA<Success>(),
]));

// No emission
stream.listen(neverCalled);
```

---

## Widget Finders

```dart
// By content
find.text('Hello')
find.text('Hello', skipOffstage: false)

// By type
find.byType(CircularProgressIndicator)
find.byType(Container)

// By key
find.byKey(const Key('my_key'))
find.byKey(const ValueKey('id'))

// By icon
find.byIcon(Icons.add)

// By widget instance
find.byWidget(myWidget)

// Combinators
find.ancestor(of: find.text('A'), matching: find.byType(Card))
find.descendant(of: find.byType(Card), matching: find.text('A'))
```

---

## Widget Matchers

```dart
findsOneWidget        // Exactly 1
findsNothing          // Exactly 0
findsNWidgets(3)      // Exactly 3
findsExactly(3)       // Exactly 3 (alias)
findsWidgets          // 1 or more
findsAtLeastNWidgets(2)  // 2 or more
```

---

## Widget Tester Methods

```dart
// Build widget tree
await tester.pumpWidget(widget);

// Trigger rebuild
await tester.pump();
await tester.pump(Duration(milliseconds: 100));

// Wait for animations
await tester.pumpAndSettle();
await tester.pumpAndSettle(Duration(milliseconds: 100));

// User interactions
await tester.tap(finder);
await tester.longPress(finder);
await tester.enterText(finder, 'text');
await tester.drag(finder, Offset(0, 100));

// Gestures
await tester.flingFrom(Offset(50, 100), Offset(0, 400), 800);

// Scrolling
await tester.scrollUntilVisible(finder, 100);
await tester.ensureVisible(finder);

// Get widgets
tester.widget<Widget>(finder);
tester.widgetList<Widget>(finder);
tester.firstWidget<Widget>(finder);
```

---

## Spy Pattern Template

```dart
final class MySpy implements MyInterface {
  // Track calls
  int callsCount = 0;

  // Capture inputs
  String? capturedInput;

  // Configure output
  dynamic output;

  // Simulate errors
  Error? error;

  // Simulation methods
  void simulateError() => error = Error();
  void simulateEmpty() => output = null;

  @override
  Future<dynamic> myMethod(String input) async {
    capturedInput = input;
    callsCount++;
    if (error != null) throw error!;
    return output;
  }
}
```

---

## Mock Pattern Template

```dart
final class MyMock implements MyInterface {
  String? capturedKey;
  dynamic capturedValue;

  @override
  Future<void> save({required String key, required dynamic value}) async {
    capturedKey = key;
    capturedValue = value;
  }
}
```

---

## Presenter Spy with Streams

```dart
final class MyPresenterSpy implements MyPresenter {
  int callsCount = 0;
  var dataSubject = BehaviorSubject<Data>();
  var loadingSubject = BehaviorSubject<bool>();

  @override
  Stream<Data> get dataStream => dataSubject.stream;

  @override
  Stream<bool> get loadingStream => loadingSubject.stream;

  void emitData([Data? data]) => dataSubject.add(data ?? Data());
  void emitError() => dataSubject.addError(Error());
  void emitLoading([bool loading = true]) => loadingSubject.add(loading);

  @override
  Future<void> load() async {
    callsCount++;
  }
}
```

---

## Test Setup Templates

### Unit Test

```dart
void main() {
  late MySpy spy;
  late MyClass sut;

  setUp(() {
    spy = MySpy();
    sut = MyClass(dependency: spy);
  });

  test('should do something', () async {
    // Arrange
    spy.output = 'value';

    // Act
    final result = await sut.doSomething();

    // Assert
    expect(result, 'value');
    expect(spy.callsCount, 1);
  });
}
```

### Widget Test

```dart
void main() {
  late MyPresenterSpy presenter;
  late Widget sut;

  setUp(() {
    presenter = MyPresenterSpy();
    sut = MaterialApp(home: MyPage(presenter: presenter));
  });

  testWidgets('should display data', (tester) async {
    await tester.pumpWidget(sut);

    presenter.emitData();
    await tester.pump();

    expect(find.text('Data'), findsOneWidget);
  });
}
```

### E2E Test

```dart
void main() {
  late ClientSpy client;
  late MyRealComponent sut;

  setUp(() {
    client = ClientSpy();  // Only mock external boundary
    sut = buildRealSystem(client: client);
  });

  testWidgets('should show api data', (tester) async {
    client.responseJson = '{"data": "value"}';

    await tester.pumpWidget(MaterialApp(home: sut));
    await tester.pump();

    expect(find.text('value'), findsOneWidget);
  });
}
```

---

## Common Test Patterns

### Verify Input

```dart
test('should call with correct input', () async {
  await sut.load(id: 'abc123');

  expect(spy.capturedId, 'abc123');
  expect(spy.callsCount, 1);
});
```

### Verify Output

```dart
test('should return mapped data', () async {
  spy.output = Data(name: 'Test');

  final result = await sut.load();

  expect(result.name, 'Test');
});
```

### Verify Error

```dart
test('should throw on error', () async {
  spy.error = Error();

  final future = sut.load();

  expect(future, throwsA(isA<Error>()));
});
```

### Verify Stream Sequence

```dart
test('should emit loading sequence', () async {
  expectLater(sut.loadingStream, emitsInOrder([true, false]));

  await sut.load();
});
```

### Verify Widget State

```dart
testWidgets('should hide spinner on success', (tester) async {
  await tester.pumpWidget(sut);
  expect(find.byType(Spinner), findsOneWidget);

  presenter.emitData();
  await tester.pump();

  expect(find.byType(Spinner), findsNothing);
});
```

---

## CLI Commands

```bash
# Run all tests
flutter test

# Run specific file
flutter test test/path/to/test.dart

# Run with coverage
flutter test --coverage

# Run in watch mode
flutter test --watch

# Run with verbose output
flutter test --reporter expanded

# Run specific test by name
flutter test --plain-name "should return data"
```

---

## Annotations

```dart
// Timeout for entire file
@Timeout(Duration(seconds: 1)) library;

// Skip test
@skip
test('not implemented', () {});

// Skip with reason
@Skip('Waiting for API fix')
test('broken test', () {});

// Tags for filtering
@Tags(['integration'])
void main() { ... }
```

---

## Package Dependencies

```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  file: ^7.0.1              # File system mocking
  network_image_mock: ^2.1.1 # Network image mocking
  flutter_lints: ^5.0.0     # Linting rules
```

---

## Quick Debugging

```dart
// Print during test
debugPrint('Value: $value');

// Dump widget tree
debugDumpApp();

// Find widget details
final widget = tester.widget<MyWidget>(finder);
debugPrint('Widget: $widget');

// Check render tree
debugDumpRenderTree();
```

---

## Remember

1. **Test behavior, not implementation**
2. **Use random data to avoid false positives**
3. **Always verify call counts**
4. **Set up stream expectations before actions**
5. **Use @Timeout for stream tests**
6. **Wrap widgets in MaterialApp**
7. **pump() for state, pumpAndSettle() for animations**
8. **Mock only external boundaries in E2E tests**
