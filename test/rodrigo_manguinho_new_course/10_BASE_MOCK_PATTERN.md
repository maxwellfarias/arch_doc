# Base Mock Pattern

## Introduction

This document explains how to create a reusable base mock class that reduces boilerplate and eliminates the need for third-party mocking libraries. This pattern leverages Dart's `noSuchMethod` to create flexible, lightweight mocks.

---

## Table of Contents

1. [The Problem](#the-problem)
2. [The Solution: BaseMock](#the-solution-basemock)
3. [How It Works](#how-it-works)
4. [Implementation](#implementation)
5. [Usage Patterns](#usage-patterns)
6. [Real-World Examples](#real-world-examples)
7. [Best Practices](#best-practices)
8. [Comparison with Third-Party Libraries](#comparison-with-third-party-libraries)

---

## The Problem

When writing tests, you often need to mock interfaces with many methods, but you only care about a few of them:

```dart
abstract interface class UserRepository {
  Future<User> getUser(String id);
  Future<List<User>> getAllUsers();
  Future<void> saveUser(User user);
  Future<void> deleteUser(String id);
  Future<void> updateUser(User user);
  Stream<User> watchUser(String id);
}
```

Without a base mock, you'd need to implement **every method**:

```dart
// ❌ Verbose - must implement all methods
class UserRepositorySpy implements UserRepository {
  @override
  Future<User> getUser(String id) async => User();

  @override
  Future<List<User>> getAllUsers() => throw UnimplementedError();

  @override
  Future<void> saveUser(User user) => throw UnimplementedError();

  @override
  Future<void> deleteUser(String id) => throw UnimplementedError();

  @override
  Future<void> updateUser(User user) => throw UnimplementedError();

  @override
  Stream<User> watchUser(String id) => throw UnimplementedError();
}
```

---

## The Solution: BaseMock

Create a base class that handles unimplemented methods automatically:

```dart
class BaseMock {
  @override
  dynamic noSuchMethod(Invocation invocation) => null;
}
```

Now your mock only needs to implement the methods you actually use:

```dart
// ✅ Concise - only implement what you need
class UserRepositorySpy extends BaseMock implements UserRepository {
  User? output;

  @override
  Future<User> getUser(String id) async => output ?? User();
}
```

---

## How It Works

### The `noSuchMethod` Mechanism

When Dart calls a method that doesn't exist on an object, it invokes `noSuchMethod`:

```dart
abstract interface class Calculator {
  int add(int a, int b);
  int subtract(int a, int b);
  int multiply(int a, int b);
}

class CalculatorMock extends BaseMock implements Calculator {
  @override
  int add(int a, int b) => a + b;

  // subtract() and multiply() are NOT implemented
  // → noSuchMethod is called, returns null
}

void main() {
  final calc = CalculatorMock();

  print(calc.add(2, 3));      // → 5 (implemented)
  print(calc.subtract(5, 2)); // → null (noSuchMethod)
  print(calc.multiply(3, 4)); // → null (noSuchMethod)
}
```

### The Invocation Object

The `Invocation` parameter contains details about the method call:

| Property | Description | Example |
|----------|-------------|---------|
| `memberName` | Method name as Symbol | `#getUser` |
| `positionalArguments` | List of positional args | `['user123']` |
| `namedArguments` | Map of named args | `{#force: true}` |
| `isMethod` | Is it a method call? | `true` |
| `isGetter` | Is it a getter? | `false` |
| `isSetter` | Is it a setter? | `false` |

---

## Implementation

### Basic BaseMock

```dart
/// Base class for creating lightweight mocks.
/// Unimplemented methods return null instead of throwing.
class BaseMock {
  @override
  dynamic noSuchMethod(Invocation invocation) => null;
}
```

### Enhanced BaseMock with Call Tracking

```dart
/// Enhanced base mock with call tracking capabilities.
class BaseMock {
  /// Stores all invocations of unimplemented methods.
  final List<Invocation> invocations = [];

  @override
  dynamic noSuchMethod(Invocation invocation) {
    invocations.add(invocation);
    return null;
  }

  /// Checks if a method was called.
  /// Usage: `expect(mock.wasCalled(#methodName), isTrue)`
  bool wasCalled(Symbol methodName) {
    return invocations.any((i) => i.memberName == methodName);
  }

  /// Counts how many times a method was called.
  /// Usage: `expect(mock.callCount(#methodName), 2)`
  int callCount(Symbol methodName) {
    return invocations.where((i) => i.memberName == methodName).length;
  }

  /// Gets the arguments from the last call to a method.
  List<dynamic>? getLastArgs(Symbol methodName) {
    final calls = invocations.where((i) => i.memberName == methodName);
    return calls.isEmpty ? null : calls.last.positionalArguments;
  }

  /// Resets all tracked invocations.
  void resetCalls() => invocations.clear();
}
```

---

## Usage Patterns

### Pattern 1: Simple Mock (Output Only)

When you only need to return a value:

```dart
class UserLoaderMock extends BaseMock implements UserLoader {
  User? output;

  @override
  Future<User> load(String id) async => output ?? User();
}

// Usage
test('should display user name', () async {
  final loader = UserLoaderMock();
  loader.output = User(name: 'John');

  final result = await loader.load('any');

  expect(result.name, 'John');
});
```

### Pattern 2: Spy (Track Inputs and Outputs)

When you need to verify inputs and control outputs:

```dart
class UserLoaderSpy extends BaseMock implements UserLoader {
  // Track calls
  int callsCount = 0;

  // Capture inputs
  String? capturedId;

  // Configure output
  User? output;

  // Simulate errors
  Error? error;

  @override
  Future<User> load(String id) async {
    capturedId = id;
    callsCount++;
    if (error != null) throw error!;
    return output ?? User();
  }

  // Helper methods
  void simulateError() => error = Error();
}

// Usage
test('should call loader with correct id', () async {
  final spy = UserLoaderSpy();
  spy.output = User(name: 'John');

  await sut.loadUser('user123');

  expect(spy.capturedId, 'user123');
  expect(spy.callsCount, 1);
});
```

### Pattern 3: Mock with Multiple Methods

When you need to implement several methods:

```dart
class HttpClientSpy extends BaseMock implements HttpClient {
  // GET
  int getCallsCount = 0;
  String? getUrl;
  dynamic getResponse;
  Error? getError;

  // POST
  int postCallsCount = 0;
  String? postUrl;
  dynamic postBody;
  dynamic postResponse;
  Error? postError;

  @override
  Future<dynamic> get(String url) async {
    getUrl = url;
    getCallsCount++;
    if (getError != null) throw getError!;
    return getResponse;
  }

  @override
  Future<dynamic> post(String url, {dynamic body}) async {
    postUrl = url;
    postBody = body;
    postCallsCount++;
    if (postError != null) throw postError!;
    return postResponse;
  }

  // Helpers
  void simulateGetError() => getError = Error();
  void simulatePostError() => postError = Error();
}
```

### Pattern 4: Using Call Tracking for Unimplemented Methods

When you want to verify calls without implementing:

```dart
class AnalyticsMock extends BaseMock implements Analytics {
  // No methods implemented - all tracked via noSuchMethod
}

// Usage
test('should track page view', () async {
  final analytics = AnalyticsMock();

  await sut.onPageLoad();

  // Verify trackPageView was called (even though not implemented)
  expect(analytics.wasCalled(#trackPageView), isTrue);
  expect(analytics.callCount(#trackPageView), 1);
});
```

---

## Real-World Examples

### Example 1: HTTP Client Spy

Based on the project's `ClientSpy`:

```dart
final class ClientSpy extends BaseMock implements Client {
  String? responseJson;
  int statusCode = 200;
  Uri? capturedUrl;
  int callsCount = 0;

  void simulateServerError() => statusCode = 500;
  void simulateUnauthorized() => statusCode = 401;

  @override
  Future<Response> get(Uri url, {Map<String, String>? headers}) async {
    capturedUrl = url;
    callsCount++;
    return Response(responseJson ?? '', statusCode);
  }
}
```

### Example 2: Cache Manager Spy

Based on the project's cache testing:

```dart
final class CacheManagerSpy extends BaseMock implements BaseCacheManager {
  final FileSpy file = FileSpy();
  String? capturedKey;
  int callsCount = 0;

  @override
  Future<FileInfo> downloadFile(
    String url, {
    String? key,
    // ... other params
  }) async {
    callsCount++;
    capturedKey = key;
    return FileInfo(file, FileSource.Cache, DateTime.now(), url);
  }
}

final class FileSpy extends BaseMock implements File {
  String? content;
  Error? error;

  void simulateResponse(String json) => content = json;
  void simulateInvalidResponse() => error = Error();

  @override
  Future<String> readAsString({Encoding encoding = utf8}) async {
    if (error != null) throw error!;
    return content ?? '';
  }
}
```

### Example 3: Presenter Spy with Streams

For testing UI components:

```dart
final class NextEventPresenterSpy extends BaseMock implements NextEventPresenter {
  int loadCallsCount = 0;
  String? capturedGroupId;

  final dataSubject = BehaviorSubject<NextEventViewModel>();
  final loadingSubject = BehaviorSubject<bool>();
  final errorSubject = BehaviorSubject<String?>();

  @override
  Stream<NextEventViewModel> get nextEventStream => dataSubject.stream;

  @override
  Stream<bool> get isLoadingStream => loadingSubject.stream;

  @override
  Stream<String?> get errorStream => errorSubject.stream;

  @override
  Future<void> loadNextEvent({required String groupId}) async {
    capturedGroupId = groupId;
    loadCallsCount++;
  }

  // Emission helpers for tests
  void emitData(NextEventViewModel vm) => dataSubject.add(vm);
  void emitLoading([bool loading = true]) => loadingSubject.add(loading);
  void emitError(String message) => errorSubject.addError(message);

  void dispose() {
    dataSubject.close();
    loadingSubject.close();
    errorSubject.close();
  }
}
```

---

## Best Practices

### 1. Always Use Random Fake Data

```dart
// ✅ Good - random data catches edge cases
test('should call with correct id', () async {
  final groupId = anyString();

  await sut.load(groupId: groupId);

  expect(spy.capturedGroupId, groupId);
});

// ❌ Bad - hardcoded may pass by accident
test('should call with correct id', () async {
  await sut.load(groupId: '123');

  expect(spy.capturedGroupId, '123');
});
```

### 2. Verify Call Counts

```dart
// ✅ Good - verifies exact number of calls
expect(spy.callsCount, 1);

// ❌ Bad - doesn't catch duplicate calls
expect(spy.wasCalled, isTrue);
```

### 3. Use Simulation Helper Methods

```dart
// ✅ Good - clear intent
spy.simulateServerError();
spy.simulateEmptyResponse();
spy.simulateUnauthorized();

// ❌ Bad - implementation details exposed
spy.statusCode = 500;
spy.response = null;
```

### 4. Throw UnimplementedError for Methods That Should Never Be Called

```dart
class UserRepoSpy extends BaseMock implements UserRepository {
  @override
  Future<User> getUser(String id) async => User();

  // If delete should never be called in this test scenario
  @override
  Future<void> deleteUser(String id) => throw UnimplementedError();
}
```

### 5. Reset State Between Tests

```dart
late MySpy spy;

setUp(() {
  spy = MySpy();  // Fresh instance each test
});

// Or if reusing:
tearDown(() {
  spy.resetCalls();
});
```

### 6. Group Related Properties

```dart
class HttpClientSpy extends BaseMock implements HttpClient {
  // Group by method
  // --- GET ---
  int getCallsCount = 0;
  String? getUrl;
  dynamic getResponse;

  // --- POST ---
  int postCallsCount = 0;
  String? postUrl;
  dynamic postResponse;
}
```

---

## Comparison with Third-Party Libraries

### Without BaseMock (Using Mockito)

```dart
// pubspec.yaml
dev_dependencies:
  mockito: ^5.0.0
  build_runner: ^2.0.0

// test file
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';

@GenerateMocks([UserRepository])
void main() {
  late MockUserRepository mockRepo;

  setUp(() {
    mockRepo = MockUserRepository();
  });

  test('should return user', () async {
    when(mockRepo.getUser(any))
        .thenAnswer((_) async => User(name: 'John'));

    final result = await mockRepo.getUser('123');

    expect(result.name, 'John');
    verify(mockRepo.getUser('123')).called(1);
  });
}

// Requires running: dart run build_runner build
```

### With BaseMock (No Dependencies)

```dart
// No additional dependencies needed

class UserRepositorySpy extends BaseMock implements UserRepository {
  User? output;
  String? capturedId;
  int callsCount = 0;

  @override
  Future<User> getUser(String id) async {
    capturedId = id;
    callsCount++;
    return output ?? User();
  }
}

void main() {
  late UserRepositorySpy spy;

  setUp(() {
    spy = UserRepositorySpy();
  });

  test('should return user', () async {
    spy.output = User(name: 'John');

    final result = await spy.getUser('123');

    expect(result.name, 'John');
    expect(spy.capturedId, '123');
    expect(spy.callsCount, 1);
  });
}
```

### Comparison Table

| Aspect | BaseMock | Mockito |
|--------|----------|---------|
| Dependencies | None | mockito, build_runner |
| Code Generation | Not needed | Required |
| Build Step | Not needed | `dart run build_runner build` |
| Learning Curve | Simple | Moderate |
| Type Safety | Full | Full |
| Flexibility | High | High |
| Boilerplate | Minimal | Minimal |
| IDE Support | Full | Full |
| Debugging | Easy | Moderate |

---

## File Organization

Place the `BaseMock` class in a shared location:

```
test/
├── mocks/
│   ├── base_mock.dart          # ← BaseMock class
│   └── fakes.dart              # Fake data utilities
├── domain/mocks/
│   └── user_loader_spy.dart
├── infra/mocks/
│   ├── http_client_spy.dart
│   └── cache_manager_spy.dart
└── presentation/mocks/
    └── presenter_spy.dart
```

### base_mock.dart

```dart
/// Base class for creating lightweight mocks without third-party dependencies.
///
/// Usage:
/// ```dart
/// class MySpy extends BaseMock implements MyInterface {
///   @override
///   String myMethod() => 'mocked';
///   // Other methods return null via noSuchMethod
/// }
/// ```
class BaseMock {
  final List<Invocation> invocations = [];

  @override
  dynamic noSuchMethod(Invocation invocation) {
    invocations.add(invocation);
    return null;
  }

  bool wasCalled(Symbol methodName) =>
      invocations.any((i) => i.memberName == methodName);

  int callCount(Symbol methodName) =>
      invocations.where((i) => i.memberName == methodName).length;

  void resetCalls() => invocations.clear();
}
```

---

## Summary

The `BaseMock` pattern provides:

1. **Zero dependencies** - No third-party packages needed
2. **Minimal boilerplate** - Only implement methods you use
3. **Full type safety** - Dart's type system still works
4. **Easy debugging** - Simple, readable code
5. **Flexible tracking** - Optional call verification
6. **Fast tests** - No code generation step

Use this pattern consistently across your tests for maintainable, lightweight mocking.

---

## Next Steps

- **[Quick Reference](09_QUICK_REFERENCE.md)**: Cheatsheet for common test patterns
- **[Best Practices](08_BEST_PRACTICES.md)**: General testing guidelines
