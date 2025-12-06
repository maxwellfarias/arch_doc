# Testing Best Practices

## Introduction

This document summarizes the best practices and patterns demonstrated throughout this project. Following these guidelines will help you write maintainable, reliable, and readable tests.

---

## Table of Contents

1. [Test Organization](#test-organization)
2. [Naming Conventions](#naming-conventions)
3. [The AAA Pattern](#the-aaa-pattern)
4. [Mocking Best Practices](#mocking-best-practices)
5. [Dependency Injection](#dependency-injection)
6. [Test Isolation](#test-isolation)
7. [Error Handling](#error-handling)
8. [Async Testing](#async-testing)
9. [Widget Testing](#widget-testing)
10. [Common Pitfalls](#common-pitfalls)

---

## Test Organization

### Mirror Source Structure

```
lib/
├── domain/entities/
├── infra/repositories/
├── presentation/presenters/
└── ui/pages/

test/
├── domain/entities/           # Same structure
├── infra/repositories/
├── presentation/presenters/
└── ui/pages/
```

### Group Related Tests

```dart
void main() {
  group('get', () {
    test('should return data', () { ... });
    test('should handle error', () { ... });
  });

  group('save', () {
    test('should save data', () { ... });
    test('should handle error', () { ... });
  });
}
```

### Centralize Mocks and Fakes

```
test/
├── mocks/
│   └── fakes.dart              # Shared fake data
├── domain/mocks/
│   └── next_event_loader_spy.dart
├── infra/mocks/
│   └── http_client_spy.dart
└── presentation/mocks/
    └── presenter_spy.dart
```

---

## Naming Conventions

### Test Names

Use descriptive names that explain expected behavior:

```dart
// ✅ Good - Clear intent
test('should return the first letter of first and last names', () {});
test('should throw UnexpectedError when response is null', () {});
test('should emit loading state before fetching data', () {});
test('should call repository with correct groupId', () {});

// ❌ Bad - Vague or technical
test('test initials', () {});
test('null check', () {});
test('loading', () {});
test('repository test', () {});
```

**Format**: `should [expected behavior] [when condition]`

### SUT Naming

Always name the system under test `sut`:

```dart
late MyRepository sut;  // Clear, consistent
late MyRepository repository;  // ❌ Avoid
late MyRepository repo;  // ❌ Avoid
```

### Spy/Mock Naming

Name by role, not type:

```dart
late HttpGetClientSpy httpClient;  // ✅ Clear role
late HttpGetClientSpy spy;  // ❌ Too generic
late HttpGetClientSpy httpGetClientSpy;  // ❌ Redundant
```

---

## The AAA Pattern

Every test follows **Arrange, Act, Assert**:

```dart
test('should return data on success', () async {
  // ARRANGE - Set up test conditions
  httpClient.response = { 'name': 'Test' };

  // ACT - Execute the code under test
  final result = await sut.loadData();

  // ASSERT - Verify the results
  expect(result.name, 'Test');
});
```

### Keep Arrange in setUp()

```dart
late HttpClientSpy httpClient;
late MyRepository sut;

setUp(() {
  httpClient = HttpClientSpy();
  sut = MyRepository(client: httpClient);
});

test('should call client', () async {
  // Arrange is in setUp()

  // Act
  await sut.loadData();

  // Assert
  expect(httpClient.callsCount, 1);
});
```

---

## Mocking Best Practices

### Use Spies for Verification

```dart
final class MySpy {
  int callsCount = 0;     // Track calls
  String? capturedInput;  // Capture inputs
  String output = '';     // Configure output
  Error? error;           // Simulate errors

  String call(String input) {
    capturedInput = input;
    callsCount++;
    if (error != null) throw error!;
    return output;
  }
}
```

### Use Simulation Methods

```dart
void simulateServerError() => statusCode = 500;
void simulateUnauthorizedError() => statusCode = 401;
void simulateEmptyResponse() => response = null;
```

### Throw UnimplementedError for Unused Methods

```dart
@override
Future<void> unusedMethod() => throw UnimplementedError();
```

This catches accidental usage of methods you didn't expect to be called.

### Use Random Fake Data

```dart
// ✅ Random data - catches edge cases
final groupId = anyString();
final date = anyDate();

// ❌ Hardcoded - may pass by accident
final groupId = 'group123';
final date = DateTime(2024, 1, 1);
```

---

## Dependency Injection

### Constructor Injection

```dart
class MyRepository {
  final HttpClient client;
  final Mapper mapper;

  MyRepository({
    required this.client,
    required this.mapper,
  });
}
```

### Function Injection

```dart
class MyPresenter {
  final Future<Data> Function({required String id}) loadData;

  MyPresenter({required this.loadData});
}

// In test
final spy = LoadDataSpy();
final sut = MyPresenter(loadData: spy.call);
```

### Benefits for Testing

- Easy to inject mocks
- No need for dependency injection frameworks
- Clear dependencies

---

## Test Isolation

### Fresh Instances in setUp()

```dart
setUp(() {
  // Fresh mocks for each test
  httpClient = HttpClientSpy();
  mapper = MapperSpy();
  sut = MyRepository(client: httpClient, mapper: mapper);
});
```

### Don't Share State Between Tests

```dart
// ❌ Bad - state leaks between tests
String sharedGroupId = '';

test('test 1', () {
  sharedGroupId = 'abc';
});

test('test 2', () {
  // sharedGroupId is still 'abc'!
});

// ✅ Good - fresh state each time
late String groupId;

setUp(() {
  groupId = anyString();
});
```

### Reset Spy State if Needed

```dart
final class MySpy {
  void reset() {
    callsCount = 0;
    capturedInput = null;
    error = null;
  }
}
```

---

## Error Handling

### Test Specific Error Types

```dart
test('should throw UnexpectedError', () async {
  httpClient.response = null;

  final future = sut.loadData();

  expect(future, throwsA(isA<UnexpectedError>()));
});
```

### Test Error Propagation

```dart
test('should rethrow original error', () async {
  final error = Error();
  httpClient.error = error;

  final future = sut.loadData();

  expect(future, throwsA(error));  // Same instance
});
```

### Test Non-Recoverable Errors

```dart
test('should rethrow SessionExpiredError (no fallback)', () async {
  apiRepo.error = SessionExpiredError();

  final future = sut.loadData();

  // Should NOT fallback to cache
  expect(future, throwsA(isA<SessionExpiredError>()));
  expect(cacheRepo.callsCount, 0);
});
```

---

## Async Testing

### Use async/await

```dart
test('should load data', () async {
  final result = await sut.loadData();
  expect(result, isNotNull);
});
```

### Test Stream Emissions

```dart
test('should emit in order', () async {
  expectLater(sut.stream, emitsInOrder([true, false]));

  await sut.doAction();
});
```

### Use @Timeout

```dart
@Timeout(Duration(seconds: 1)) library;

// Prevents hanging tests
```

### Set Up Stream Expectations First

```dart
// ✅ Correct - listen before action
expectLater(sut.stream, emits(value));
await sut.action();

// ❌ Wrong - might miss emission
await sut.action();
expectLater(sut.stream, emits(value));
```

---

## Widget Testing

### Always Wrap with MaterialApp

```dart
await tester.pumpWidget(
  const MaterialApp(home: MyWidget())
);
```

### Use pump() vs pumpAndSettle()

```dart
// Simple state change
presenter.emit();
await tester.pump();

// Animation/transition
await tester.flingFrom(...);
await tester.pumpAndSettle();
```

### Mock Network Images

```dart
testWidgets('test with image', (tester) async {
  mockNetworkImagesFor(() async {
    await tester.pumpWidget(sut);
    // ...
  });
});
```

### Use ensureVisible for Scrollable Content

```dart
await tester.ensureVisible(
  find.text('Item', skipOffstage: false)
);
await tester.pump();
expect(find.text('Item'), findsOneWidget);
```

---

## Common Pitfalls

### 1. Not Using setUp()

```dart
// ❌ Bad - duplicated setup
test('test 1', () {
  final spy = MySpy();
  final sut = MyClass(spy);
  // ...
});

test('test 2', () {
  final spy = MySpy();  // Duplicated!
  final sut = MyClass(spy);
  // ...
});

// ✅ Good
late MySpy spy;
late MyClass sut;

setUp(() {
  spy = MySpy();
  sut = MyClass(spy);
});
```

### 2. Hardcoded Test Data

```dart
// ❌ Bad - may pass by accident
expect(result.id, '123');

// ✅ Good - random ensures real matching
final expectedId = anyString();
spy.output = Data(id: expectedId);
final result = await sut.load();
expect(result.id, expectedId);
```

### 3. Missing Error Cases

```dart
// ❌ Incomplete - only tests success
test('should return data', () async {
  final result = await sut.load();
  expect(result, isNotNull);
});

// ✅ Complete - tests all paths
test('should return data on success', () async { ... });
test('should throw on error', () async { ... });
test('should handle null response', () async { ... });
```

### 4. Testing Implementation Details

```dart
// ❌ Bad - testing how, not what
test('should call _privateMethod', () { ... });

// ✅ Good - testing behavior
test('should return formatted name', () { ... });
```

### 5. Not Verifying Call Count

```dart
// ❌ Bad - might be called multiple times
test('should call api', () async {
  await sut.load();
  expect(api.wasCalled, isTrue);
});

// ✅ Good - verifies single call
test('should call api once', () async {
  await sut.load();
  expect(api.callsCount, 1);
});
```

### 6. Forgetting to pump()

```dart
// ❌ Bad - widget not rebuilt
presenter.emit();
expect(find.text('New'), findsOneWidget);  // Fails!

// ✅ Good
presenter.emit();
await tester.pump();  // Rebuild
expect(find.text('New'), findsOneWidget);
```

### 7. Missing @Timeout for Streams

```dart
// ❌ Bad - may hang forever
test('should emit', () async {
  expectLater(sut.stream, emits(value));
  // If stream never emits, test hangs
});

// ✅ Good - fails after 1 second
@Timeout(Duration(seconds: 1)) library;
```

---

## Testing Pyramid

```
        /\
       /  \     E2E Tests (few)
      /    \    - Complete user flows
     /      \   - Real components + mock boundaries
    /--------\
   /          \   Widget Tests (some)
  /            \  - UI components
 /              \ - User interactions
/----------------\
        |         |   Unit Tests (many)
        |         |   - Domain logic
        |         |   - Mappers, repositories
        |         |   - Fast, isolated
-------------------
```

### Test Distribution

| Type | Quantity | Speed | Confidence |
|------|----------|-------|------------|
| Unit | Many | Fast | Low-Medium |
| Widget | Some | Medium | Medium |
| E2E | Few | Slow | High |

---

## Checklist

### Before Writing Tests
- [ ] Identify the behavior to test
- [ ] Determine the test type (unit, widget, E2E)
- [ ] Plan the happy path and error cases

### Writing Tests
- [ ] Use AAA pattern
- [ ] Use descriptive names
- [ ] Use setUp() for common setup
- [ ] Use random fake data
- [ ] Test error handling
- [ ] Verify call counts

### After Writing Tests
- [ ] Run tests to ensure they pass
- [ ] Check test names are clear
- [ ] Remove redundant tests
- [ ] Verify edge cases are covered

---

## Next Steps

For a quick reference of testing utilities and patterns:

**[→ Quick Reference](09_QUICK_REFERENCE.md)**: Cheatsheet for common assertions and utilities.
