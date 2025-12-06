# Mocking Strategies in Flutter Testing

## Introduction

Mocking is essential for isolating the code under test from its dependencies. This project demonstrates three main mocking strategies:

1. **Spy** - Tracks calls, captures parameters, and can be configured to return values or throw errors
2. **Mock** - Simple capture of values without complex behavior
3. **Fake** - Generates random but valid test data

Understanding when and how to use each strategy is crucial for writing effective tests.

---

## Table of Contents

1. [The Spy Pattern](#the-spy-pattern)
2. [The Mock Pattern](#the-mock-pattern)
3. [Fake Data Generation](#fake-data-generation)
4. [RxDart Stream Mocking](#rxdart-stream-mocking)
5. [Specialized Spies](#specialized-spies)
6. [When to Use Each Pattern](#when-to-use-each-pattern)

---

## The Spy Pattern

A **Spy** is the most powerful mocking strategy. It:
- ✅ Tracks how many times it was called
- ✅ Captures input parameters
- ✅ Can return configured output
- ✅ Can simulate errors
- ✅ Has helper methods for common scenarios

### Basic Spy Structure

```dart
final class NextEventLoaderSpy {
  // 1. Track call count
  int callsCount = 0;

  // 2. Capture input parameters
  String? groupId;

  // 3. Configure error simulation
  Error? error;

  // 4. Configure return value
  NextEvent output = NextEvent(
    groupName: anyString(),
    date: anyDate(),
    players: []
  );

  // 5. Helper method for specific scenarios
  void simulatePlayers(List<NextEventPlayer> players) {
    output = NextEvent(
      groupName: anyString(),
      date: anyDate(),
      players: players
    );
  }

  // 6. The method that mimics the real implementation
  Future<NextEvent> call({ required String groupId }) async {
    this.groupId = groupId;       // Capture input
    callsCount++;                  // Track call
    if (error != null) throw error!;  // Throw if configured
    return output;                 // Return configured output
  }
}
```

### Using the Spy in Tests

```dart
void main() {
  late NextEventLoaderSpy loader;
  late MyPresenter sut;

  setUp(() {
    loader = NextEventLoaderSpy();
    sut = MyPresenter(loader: loader.call);
  });

  test('should call loader with correct groupId', () async {
    await sut.loadEvent(groupId: 'group-123');

    expect(loader.groupId, 'group-123');  // Verify input
    expect(loader.callsCount, 1);         // Verify call count
  });

  test('should return data from loader', () async {
    loader.output = NextEvent(
      groupName: 'My Team',
      date: DateTime(2024, 1, 1),
      players: []
    );

    final result = await sut.loadEvent(groupId: 'any');

    expect(result.groupName, 'My Team');
  });

  test('should handle error from loader', () async {
    loader.error = Error();  // Configure error

    final future = sut.loadEvent(groupId: 'any');

    expect(future, throwsA(isA<Error>()));
  });
}
```

---

### HTTP Client Spy

A more complex spy for HTTP operations:

```dart
final class HttpGetClientSpy implements HttpGetClient {
  // Track inputs
  String? url;
  int callsCount = 0;
  Json? params;
  Json? queryString;
  Json? headers;

  // Configurable output
  dynamic response = anyJson();
  Error? error;

  @override
  Future<dynamic> get({
    required String url,
    Json? headers,
    Json? params,
    Json? queryString
  }) async {
    // Capture all inputs
    this.url = url;
    this.params = params;
    this.queryString = queryString;
    this.headers = headers;
    callsCount++;

    // Error or success
    if (error != null) throw error!;
    return response;
  }
}
```

### Testing with HTTP Client Spy

```dart
test('should call HttpClient with correct input', () async {
  await sut.loadNextEvent(groupId: 'abc123');

  expect(httpClient.url, 'https://api.example.com/events');
  expect(httpClient.params, { 'groupId': 'abc123' });
  expect(httpClient.callsCount, 1);
});

test('should return mapped data on success', () async {
  httpClient.response = { 'name': 'Team A', 'date': '2024-01-01' };

  final event = await sut.loadNextEvent(groupId: 'any');

  expect(event.groupName, 'Team A');
});
```

---

### Raw HTTP Client Spy with Simulation Methods

For low-level HTTP client testing:

```dart
final class ClientSpy implements Client {
  String? method;
  String? url;
  int callsCount = 0;
  Map<String, String>? headers;
  String responseJson = '';
  int statusCode = 200;

  // Simulation helper methods
  void simulateNoContent() => statusCode = 204;
  void simulateBadRequestError() => statusCode = 400;
  void simulateUnauthorizedError() => statusCode = 401;
  void simulateForbiddenError() => statusCode = 403;
  void simulateNotFoundError() => statusCode = 404;
  void simulateServerError() => statusCode = 500;

  @override
  Future<Response> get(Uri url, {Map<String, String>? headers}) async {
    method = 'get';
    callsCount++;
    this.url = url.toString();
    this.headers = headers;
    return Response(responseJson, statusCode);
  }

  // Other HTTP methods throw UnimplementedError
  // to catch accidental usage
  @override
  Future<Response> post(Uri url, {...}) {
    throw UnimplementedError();
  }

  // ... other methods
}
```

### Using Simulation Methods

```dart
test('should throw SessionExpiredError on 401', () async {
  client.simulateUnauthorizedError();  // Much cleaner!

  final future = sut.get(url: 'http://api.com');

  expect(future, throwsA(isA<SessionExpiredError>()));
});

test('should throw ServerError on 500', () async {
  client.simulateServerError();

  final future = sut.get(url: 'http://api.com');

  expect(future, throwsA(isA<ServerError>()));
});
```

---

## The Mock Pattern

A **Mock** is simpler than a Spy. It just captures values without tracking calls or simulating errors:

```dart
final class CacheSaveClientMock implements CacheSaveClient {
  String? key;
  dynamic value;

  @override
  Future<void> save({ required String key, required dynamic value }) async {
    this.key = key;
    this.value = value;
  }
}
```

### When to Use Mock vs Spy

| Use Mock When | Use Spy When |
|---------------|--------------|
| Only need to capture values | Need to track call count |
| Method has no return value | Need to return different values |
| Don't need error simulation | Need to simulate errors |
| Simple verification | Complex verification scenarios |

### Mock Usage Example

```dart
test('should save data to cache with correct key', () async {
  await sut.saveEvent(event);

  expect(cacheMock.key, 'events_123');
  expect(cacheMock.value, { 'name': 'Team A' });
});
```

---

## Fake Data Generation

**Fakes** generate random but valid test data. This project uses a centralized `fakes.dart` file:

```dart
import 'dart:math';

import 'package:advanced_flutter/domain/entities/next_event.dart';
import 'package:advanced_flutter/domain/entities/next_event_player.dart';
import 'package:advanced_flutter/infra/types/json.dart';

// Primitive fakes
int anyInt([int max = 999999999]) => Random().nextInt(max);
String anyString() => anyInt().toString();
bool anyBool() => Random().nextBool();
DateTime anyDate() => DateTime.fromMillisecondsSinceEpoch(anyInt());
String anyIsoDate() => anyDate().toIso8601String();

// JSON fakes
Json anyJson() => { anyString(): anyString() };
JsonArr anyJsonArr() => List.generate(anyInt(5), (index) => anyJson());

// Domain entity fakes
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

List<NextEventPlayer> anyNextEventPlayerList() =>
  List.generate(anyInt(5), (index) => anyNextEventPlayer());
```

### Why Use Random Fake Data?

1. **Avoids false positives**: Hardcoded values might accidentally pass tests
2. **Tests edge cases**: Random values catch unexpected issues
3. **Cleaner tests**: No need to invent test data
4. **Self-documenting**: `anyString()` clearly indicates "any valid string works"

### Fake Usage Examples

```dart
setUp(() {
  groupId = anyString();  // Random group ID
  key = anyString();      // Random cache key

  // Spy with random output
  apiRepo = LoadNextEventRepositorySpy();
  apiRepo.output = anyNextEvent();
});

test('should call repository with groupId', () async {
  final testGroupId = anyString();  // Fresh random value

  await sut.loadEvent(groupId: testGroupId);

  expect(repo.groupId, testGroupId);  // Exact match
});
```

---

## RxDart Stream Mocking

For reactive presenters using RxDart, spies need to include `BehaviorSubject` for stream control:

```dart
import 'package:rxdart/rxdart.dart';

final class NextEventPresenterSpy implements NextEventPresenter {
  // Standard spy properties
  int callsCount = 0;
  String? groupId;
  bool? isReload;

  // BehaviorSubjects for stream control
  var nextEventSubject = BehaviorSubject<NextEventViewModel>();
  var isBusySubject = BehaviorSubject<bool>();

  // Expose streams
  @override
  Stream<NextEventViewModel> get nextEventStream => nextEventSubject.stream;

  @override
  Stream<bool> get isBusyStream => isBusySubject.stream;

  // Helper methods to emit values
  void emitNextEvent([NextEventViewModel? viewModel]) {
    nextEventSubject.add(viewModel ?? const NextEventViewModel());
  }

  void emitNextEventWith({
    List<NextEventPlayerViewModel> goalkeepers = const [],
    List<NextEventPlayerViewModel> players = const [],
    List<NextEventPlayerViewModel> out = const [],
    List<NextEventPlayerViewModel> doubt = const []
  }) {
    nextEventSubject.add(NextEventViewModel(
      goalkeepers: goalkeepers,
      players: players,
      out: out,
      doubt: doubt
    ));
  }

  void emitError() {
    nextEventSubject.addError(Error());
  }

  void emitIsBusy([bool isBusy = true]) {
    isBusySubject.add(isBusy);
  }

  // Standard interface implementation
  @override
  Future<void> loadNextEvent({
    required String groupId,
    bool isReload = false
  }) async {
    callsCount++;
    this.groupId = groupId;
    this.isReload = isReload;
  }
}
```

### Using Stream Spy in Widget Tests

```dart
testWidgets('should show loading indicator initially', (tester) async {
  await tester.pumpWidget(sut);

  expect(find.byType(CircularProgressIndicator), findsOneWidget);
});

testWidgets('should hide loading on success', (tester) async {
  await tester.pumpWidget(sut);

  presenter.emitNextEvent();  // Emit success
  await tester.pump();        // Rebuild widget

  expect(find.byType(CircularProgressIndicator), findsNothing);
});

testWidgets('should display error message on error', (tester) async {
  await tester.pumpWidget(sut);

  presenter.emitError();  // Emit error
  await tester.pump();

  expect(find.text('Error loading data'), findsOneWidget);
});

testWidgets('should show busy indicator on reload', (tester) async {
  presenter.emitNextEvent();  // Initial load
  await tester.pumpWidget(sut);
  await tester.pump();

  presenter.emitIsBusy(true);  // Start reload
  await tester.pump();

  expect(find.byType(RefreshIndicator), findsOneWidget);
});
```

---

## Specialized Spies

### Cache Manager Spy

For testing cache operations with `flutter_cache_manager`:

```dart
final class CacheManagerSpy implements BaseCacheManager {
  // Track calls
  int getFileFromCacheCallsCount = 0;
  int putFileCallsCount = 0;

  // Capture inputs
  String? key;
  String? fileExtension;
  dynamic fileBytesDecoded;

  // Mock file for file operations
  FileSpy file = FileSpy();

  // Private state for simulation
  bool _isFileInfoEmpty = false;
  DateTime _validTill = DateTime.now().add(const Duration(seconds: 2));
  Error? _getFileFromCachError;
  Error? _putFileError;

  // Simulation methods
  void simulateEmptyFileInfo() => _isFileInfoEmpty = true;
  void simulateCacheOld() =>
    _validTill = DateTime.now().subtract(const Duration(seconds: 2));
  void simulateGetFileFromCacheError() => _getFileFromCachError = Error();
  void simulatePutFileError() => _putFileError = Error();

  @override
  Future<FileInfo?> getFileFromCache(
    String key,
    {bool ignoreMemCache = false}
  ) async {
    getFileFromCacheCallsCount++;
    this.key = key;
    if (_getFileFromCachError != null) throw _getFileFromCachError!;
    return _isFileInfoEmpty
      ? null
      : FileInfo(file, FileSource.Cache, _validTill, '');
  }

  @override
  Future<File> putFile(
    String url,
    Uint8List fileBytes,
    {String? key, String? eTag, Duration maxAge = const Duration(days: 30),
     String fileExtension = 'file'}
  ) async {
    putFileCallsCount++;
    this.key = url;
    this.fileExtension = fileExtension;
    fileBytesDecoded = jsonDecode(utf8.decode(fileBytes));
    if (_putFileError != null) throw _putFileError!;
    return file;
  }

  // Unimplemented methods throw errors to catch accidental usage
  @override
  Future<void> dispose() => throw UnimplementedError();
  // ... other methods
}
```

### File Spy

For testing file operations:

```dart
final class FileSpy implements File {
  int existsCallsCount = 0;
  int readAsStringCallsCount = 0;

  bool _fileExists = true;
  String _response = '{}';
  Error? _readAsStringError;
  Error? _existsError;

  // Simulation methods
  void simulateFileEmpty() => _fileExists = false;
  void simulateReadAsStringError() => _readAsStringError = Error();
  void simulateExistsError() => _existsError = Error();
  void simulateInvalidResponse() => _response = 'invalid_json';
  void simulateResponse(String response) => _response = response;

  @override
  Future<bool> exists() async {
    existsCallsCount++;
    if (_existsError != null) throw _existsError!;
    return _fileExists;
  }

  @override
  Future<String> readAsString({Encoding encoding = utf8}) async {
    readAsStringCallsCount++;
    if (_readAsStringError != null) throw _readAsStringError!;
    return _response;
  }

  // ... other methods throw UnimplementedError
}
```

---

### Mapper Spy

For testing data transformation:

```dart
final class MapperSpy<Dto> implements Mapper<Dto> {
  // Input tracking
  Json? toDtoIntput;
  int toDtoIntputCallsCount = 0;
  int toJsonCallsCount = 0;
  Dto? toJsonIntput;

  // Configurable outputs
  Dto toDtoOutput;
  Json toJsonOutput = anyJson();

  MapperSpy({required this.toDtoOutput});

  @override
  Dto toDto(Json json) {
    toDtoIntput = json;
    toDtoIntputCallsCount++;
    return toDtoOutput;
  }

  @override
  Json toJson(Dto dto) {
    toJsonIntput = dto;
    toJsonCallsCount++;
    return toJsonOutput;
  }
}
```

---

## When to Use Each Pattern

| Pattern | Use Case | Complexity |
|---------|----------|------------|
| **Fake** | Generate random test data | Low |
| **Mock** | Simple value capture, no return needed | Low |
| **Spy** | Full control: tracking, returns, errors | Medium |
| **Stream Spy** | Reactive components with BehaviorSubject | High |

### Decision Tree

```
Do you need to track how many times it was called?
├── No → Do you need to capture values?
│         ├── No → Use real implementation or nothing
│         └── Yes → Use Mock
└── Yes → Do you need to simulate errors or control output?
          ├── No → Use simple Spy
          └── Yes → Use Spy with simulation methods
                    └── Is it a Stream? → Use Stream Spy with BehaviorSubject
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Spy** | Tracks calls, captures inputs, configurable output and errors |
| **Mock** | Simple value capture without behavior |
| **Fake** | Generates random valid test data |
| **BehaviorSubject** | Enables stream control in RxDart spies |
| **Simulation Methods** | `simulate*()` methods for common scenarios |
| **UnimplementedError** | Catches accidental usage of unneeded methods |

---

## Next Steps

Now that you understand mocking strategies, proceed to:

**[→ Domain Layer Testing](03_DOMAIN_TESTING.md)**: Learn how to test pure business logic in entities.
