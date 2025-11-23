# Design Patterns - Comprehensive Guide

## Table of Contents
- [Introduction](#introduction)
- [Architectural Patterns](#architectural-patterns)
- [Creational Patterns](#creational-patterns)
- [Structural Patterns](#structural-patterns)
- [Behavioral Patterns](#behavioral-patterns)
- [SOLID Principles in Action](#solid-principles-in-action)

---

## Introduction

This project demonstrates **10+ design patterns** used in professional software development. Each pattern solves a specific problem and makes code more maintainable, testable, and flexible.

### What is a Design Pattern?

**Analogy**: Design patterns are like **recipes** in cooking. Just as recipes provide proven solutions for making dishes, design patterns provide proven solutions for common programming problems.

### Why Learn Patterns?

- ✅ **Common vocabulary**: Communicate with other developers ("This uses the Adapter pattern")
- ✅ **Proven solutions**: Don't reinvent the wheel
- ✅ **Better code**: More maintainable and understandable
- ✅ **Career growth**: Expected knowledge in professional development

---

## Architectural Patterns

These are high-level patterns that define the overall structure of the application.

---

### 1. Clean Architecture

**Category**: Architectural Pattern

**What it solves**: How to organize code so it's maintainable, testable, and independent of frameworks

**Real-world analogy**: Building a house with clear floors—foundation, plumbing, furniture, and decoration. Each floor has a specific purpose and can be changed without affecting others.

#### Implementation in This Project

**Structure**:
```
Application
    ├─ Domain Layer (Core business logic)
    ├─ Infrastructure Layer (External communication)
    ├─ Presentation Layer (UI business logic)
    └─ UI Layer (Widgets)
```

**Files**: Entire project structure

**Key Rules**:
1. Dependencies point inward (toward domain)
2. Domain has no dependencies
3. Each layer has clear responsibilities

#### Code Example

```dart
// ✅ GOOD: Infrastructure depends on Domain
// File: lib/infra/api/repositories/load_next_event_api_repo.dart
import 'package:fluttter_avancado_tdd_clean_arch/domain/entities/next_event.dart';

class LoadNextEventApiRepository {
  Future<NextEvent> loadNextEvent({required String groupId}) async {
    // Uses domain entity
  }
}

// ❌ BAD: Domain depending on Infrastructure (FORBIDDEN!)
// File: lib/domain/entities/next_event.dart
import 'package:http/http.dart';  // External dependency in domain - NO!
```

**Benefits**:
- ✅ Testable (each layer tested independently)
- ✅ Framework-independent (domain can run anywhere)
- ✅ Flexible (easy to swap implementations)

**Reference**: [01-ARCHITECTURE-OVERVIEW.md](01-ARCHITECTURE-OVERVIEW.md)

---

### 2. MVP (Model-View-Presenter)

**Category**: Architectural Pattern

**What it solves**: How to separate UI from business logic

**Real-world analogy**: Like a **restaurant**:
- **Model**: The kitchen (where food is prepared)
- **View**: The dining area (what customers see)
- **Presenter**: The waiter (coordinates between kitchen and customers)

#### Implementation in This Project

**Files**:
- Model (Domain): [lib/domain/entities/next_event.dart](fluttter_avancado_tdd_clean_arch/lib/domain/entities/next_event.dart)
- View (UI): [lib/ui/pages/next_event_page.dart](fluttter_avancado_tdd_clean_arch/lib/ui/pages/next_event_page.dart)
- Presenter: [lib/presentation/rx/next_event_rx_presenter.dart](fluttter_avancado_tdd_clean_arch/lib/presentation/rx/next_event_rx_presenter.dart)

#### How It Works

```
┌─────────────┐         ┌──────────────┐         ┌───────────────┐
│    View     │◄────────│  Presenter   │────────►│    Model      │
│  (Passive)  │ Updates │  (Mediator)  │ Fetches │ (Data/Logic)  │
│             │         │              │         │               │
└─────────────┘         └──────────────┘         └───────────────┘
       │                        ▲
       │ User Actions           │
       └────────────────────────┘
```

#### Code Example

**View** (Passive - just displays):
```dart
// File: lib/ui/pages/next_event_page.dart:42-58
@override
Widget build(BuildContext context) {
  return StreamBuilder<NextEventViewModel>(
    stream: widget.presenter.nextEventStream,  // Listens to presenter
    builder: (context, snapshot) {
      if (snapshot.hasError) return ErrorLayout(...);

      final viewModel = snapshot.data!;
      return ListView(
        children: [
          ListSection(title: 'GOALKEEPERS', items: viewModel.goalkeepers),
          // ... more sections
        ],
      );
    },
  );
}
```

**Presenter** (Mediator - coordinates):
```dart
// File: lib/presentation/rx/next_event_rx_presenter.dart:15-32
final class NextEventRxPresenter implements NextEventPresenter {
  final nextEventSubject = BehaviorSubject<NextEventViewModel>();

  Future<void> loadNextEvent({required String groupId}) async {
    try {
      final event = await nextEventLoader(groupId: groupId);  // Load from model
      nextEventSubject.add(_mapEvent(event));  // Transform and notify view
    } catch (error) {
      nextEventSubject.addError(error);
    }
  }

  NextEventViewModel _mapEvent(NextEvent event) {
    // Filter, sort, categorize for UI
    return NextEventViewModel(...);
  }
}
```

**Model** (Domain entity):
```dart
// File: lib/domain/entities/next_event.dart:3-11
final class NextEvent {
  final String groupName;
  final DateTime date;
  final List<NextEventPlayer> players;
}
```

**Benefits**:
- ✅ View is testable (just widgets, no logic)
- ✅ Presenter is testable (no Flutter dependencies)
- ✅ Clear separation of concerns

**Testing**:
- View test: [test/ui/pages/next_event_page_test.dart](fluttter_avancado_tdd_clean_arch/test/ui/pages/next_event_page_test.dart)
- Presenter test: [test/presentation/rx/next_event_rx_presenter_test.dart](fluttter_avancado_tdd_clean_arch/test/presentation/rx/next_event_rx_presenter_test.dart)

---

## Creational Patterns

Patterns that deal with object creation.

---

### 3. Factory Pattern

**Category**: Creational Pattern

**What it solves**: How to create objects without exposing creation logic

**Real-world analogy**: When you order a burger at McDonald's, you don't know how it's made—you just ask for it and receive it ready.

#### Use Case 1: Entity Creation with Business Logic

**File**: [lib/domain/entities/next_event_player.dart](fluttter_avancado_tdd_clean_arch/lib/domain/entities/next_event_player.dart)

```dart
final class NextEventPlayer {
  final String name;
  final String initials;  // Auto-generated

  // Private constructor (users can't call this directly)
  const NextEventPlayer._({
    required this.name,
    required this.initials,
    // ... other fields
  });

  // Public factory constructor
  factory NextEventPlayer({
    required String name,
    // ... other fields
  }) => NextEventPlayer._(
        name: name,
        initials: _getInitials(name),  // 🎯 Business logic hidden
        // ... other fields
      );

  // Business rule: generate initials
  static String _getInitials(String name) {
    final names = name.toUpperCase().trim().split(' ');
    final firstChar = names.first.split('').firstOrNull ?? '-';
    final lastChar = names.last.split('').elementAtOrNull(
      names.length == 1 ? 1 : 0,
    ) ?? '';
    return '$firstChar$lastChar';
  }
}

// Usage:
final player = NextEventPlayer(name: 'Maxwell Farias');
print(player.initials);  // "MF" - auto-generated!
```

**Why this is powerful**:
- ✅ User can't create player with wrong initials
- ✅ Business rule encapsulated
- ✅ Initials always consistent

#### Use Case 2: Dependency Injection Factories

**File**: [lib/main/factories/ui/pages/next_event_page_factory.dart](fluttter_avancado_tdd_clean_arch/lib/main/factories/ui/pages/next_event_page_factory.dart)

```dart
Widget makeNextEventPage() {
  final repo = makeLoadNextEventFromApiWithCacheFallbackRepository();
  final presenter = NextEventRxPresenter(
    nextEventLoader: repo.loadNextEvent,
  );
  return NextEventPage(
    presenter: presenter,
    groupId: 'valid_id',
  );
}
```

**Chain of factories**:
```
makeNextEventPage()
  └─> makeLoadNextEventFromApiWithCacheFallbackRepository()
      ├─> makeHttpAdapter()
      ├─> makeNextEventMapper()
      │   └─> makeNextEventPlayerMapper()
      └─> makeCacheManagerAdapter()
```

**Benefits**:
- ✅ Centralized object creation
- ✅ Easy to change implementations
- ✅ Clear dependency graph

**All Factory Files**: [lib/main/factories/](fluttter_avancado_tdd_clean_arch/lib/main/factories/)

---

### 4. Singleton Pattern (Implicit)

**Category**: Creational Pattern

**What it solves**: Ensure only one instance of a class exists

**Real-world analogy**: There's only one CEO of a company—not multiple.

#### Implementation

**File**: [lib/presentation/rx/next_event_rx_presenter.dart](fluttter_avancado_tdd_clean_arch/lib/presentation/rx/next_event_rx_presenter.dart)

```dart
final class NextEventRxPresenter implements NextEventPresenter {
  final nextEventSubject = BehaviorSubject<NextEventViewModel>();
  final isBusySubject = BehaviorSubject<bool>();

  // Created once by factory, shared across page lifecycle
}
```

**Managed by factory**:
```dart
// File: lib/main/factories/ui/pages/next_event_page_factory.dart
Widget makeNextEventPage() {
  final presenter = NextEventRxPresenter(...);  // One instance
  return NextEventPage(presenter: presenter);   // Injected
}
```

**Note**: This is implicit singleton (one per page instance), not global singleton.

---

## Structural Patterns

Patterns that deal with object composition and relationships.

---

### 5. Adapter Pattern

**Category**: Structural Pattern

**What it solves**: How to make incompatible interfaces work together

**Real-world analogy**: Like a **power adapter** for your phone when traveling—it adapts the wall socket (external interface) to your phone charger (your code).

#### Use Case 1: HTTP Client Adapter

**Problem**: We don't want our repositories to depend directly on the `http` package.

**Solution**: Create an adapter that wraps the package.

**File**: [lib/infra/api/adapters/http_adapter.dart](fluttter_avancado_tdd_clean_arch/lib/infra/api/adapters/http_adapter.dart)

```dart
// External package we want to wrap
import 'package:http/http.dart' as http;

// Our interface (what repositories need)
abstract interface class HttpGetClient {
  Future<dynamic> get({
    required String url,
    Json? headers,
    Json? params,
    Json? queryString,
  });
}

// Adapter: Makes http package work with our interface
final class HttpAdapter implements HttpGetClient {
  final http.Client client;  // External dependency

  const HttpAdapter({required this.client});

  @override
  Future<dynamic> get({
    required String url,
    Json? headers,
    Json? params,
    Json? queryString,
  }) async {
    final response = await client.get(
      _buildUri(url: url, params: params, queryString: queryString),
      headers: _buildHeaders(url: url, headers: headers),
    );
    return _handleResponse(response);
  }

  // Adapter-specific logic: URL building, header setup, error handling
}
```

**Before Adapter** (tightly coupled):
```dart
class LoadNextEventApiRepository {
  final http.Client client;  // Direct dependency on external package!

  Future<NextEvent> loadNextEvent() async {
    final response = await client.get(Uri.parse('http://...'));
    // ... handle response
  }
}
```

**After Adapter** (loosely coupled):
```dart
class LoadNextEventApiRepository {
  final HttpGetClient httpClient;  // Depends on interface, not package!

  Future<NextEvent> loadNextEvent() async {
    final json = await httpClient.get(url: 'http://...');
    return mapper.toDto(json);
  }
}
```

**Benefits**:
- ✅ Repository doesn't know about `http` package
- ✅ Easy to swap (use Dio, fetch, etc.)
- ✅ Easy to test (mock interface)
- ✅ Centralized error handling

#### Use Case 2: Cache Manager Adapter

**File**: [lib/infra/cache/adapters/cache_manager_adapter.dart](fluttter_avancado_tdd_clean_arch/lib/infra/cache/adapters/cache_manager_adapter.dart)

```dart
import 'package:flutter_cache_manager/flutter_cache_manager.dart';

// Our interfaces
abstract interface class CacheGetClient {
  Future<dynamic> get({required String key});
}

abstract interface class CacheSaveClient {
  Future<void> save({required String key, required dynamic value});
}

// Adapter: Wraps flutter_cache_manager package
final class CacheManagerAdapter
    implements CacheGetClient, CacheSaveClient {
  final BaseCacheManager client;  // External dependency

  const CacheManagerAdapter({required this.client});

  @override
  Future<dynamic> get({required String key}) async {
    final info = await client.getFileFromCache(key);
    if (info?.validTill.isBefore(DateTime.now()) != false) return null;

    final data = await info!.file.readAsString();
    return jsonDecode(data);
  }

  @override
  Future<void> save({required String key, required dynamic value}) async {
    await client.putFile(
      key,
      utf8.encode(jsonEncode(value)),
      fileExtension: 'json',
    );
  }
}
```

**Benefits**: Same as HTTP adapter—isolates external dependency.

**Testing**:
- HTTP Adapter: [test/infra/api/adapters/http_client_test.dart](fluttter_avancado_tdd_clean_arch/test/infra/api/adapters/http_client_test.dart)
- Cache Adapter: [test/infra/cache/adapters/cache_manager_adapter_test.dart](fluttter_avancado_tdd_clean_arch/test/infra/cache/adapters/cache_manager_adapter_test.dart)

---

### 6. Repository Pattern

**Category**: Structural Pattern

**What it solves**: How to abstract data source details from business logic

**Real-world analogy**: Like a **librarian**—you ask for a book, and the librarian finds it. You don't care if it's in the basement, on the shelf, or needs to be ordered.

#### Basic Repository

**File**: [lib/infra/api/repositories/load_next_event_api_repo.dart](fluttter_avancado_tdd_clean_arch/lib/infra/api/repositories/load_next_event_api_repo.dart)

```dart
final class LoadNextEventApiRepository {
  final HttpGetClient httpClient;
  final String url;
  final DtoMapper<NextEvent> mapper;

  const LoadNextEventApiRepository({
    required this.httpClient,
    required this.url,
    required this.mapper,
  });

  Future<NextEvent> loadNextEvent({required String groupId}) async {
    final event = await httpClient.get(
      url: url,
      params: {"groupId": groupId},
    );
    return mapper.toDto(event);  // JSON → Domain entity
  }
}
```

**What presenter sees**:
```dart
// Presenter doesn't know:
// - Where data comes from (API? Database? File?)
// - How it's fetched (HTTP? GraphQL?)
// - How JSON is parsed

final event = await repository.loadNextEvent(groupId: 'abc');
// Just gets NextEvent entity - simple!
```

**Benefits**:
- ✅ Presenter doesn't know about data sources
- ✅ Easy to swap (mock in tests, use different API)
- ✅ Centralized data fetching logic

#### Composite Repository

**File**: [lib/infra/repositories/load_next_event_from_api_with_cache_fallback_repo.dart](fluttter_avancado_tdd_clean_arch/lib/infra/repositories/load_next_event_from_api_with_cache_fallback_repo.dart)

```dart
final class LoadNextEventFromApiWithCacheFallbackRepository {
  final LoadNextEventRepository loadNextEventFromApi;
  final LoadNextEventRepository loadNextEventFromCache;
  final CacheSaveClient cacheClient;
  final JsonMapper<NextEvent> mappper;
  final String key;

  Future<NextEvent> loadNextEvent({required String groupId}) async {
    try {
      // Try API first
      final event = await loadNextEventFromApi(groupId: groupId);

      // Save to cache
      await cacheClient.save(
        key: '$key:$groupId',
        value: mappper.toJson(event),
      );

      return event;
    } on SessionExpiredError {
      rethrow;  // Don't catch auth errors
    } catch (error) {
      // Fallback to cache
      return loadNextEventFromCache(groupId: groupId);
    }
  }
}
```

**Strategy**:
1. Try API (fresh data)
2. Save to cache on success
3. Fallback to cache on error
4. Always propagate auth errors

**This combines two patterns**: Repository + Strategy!

**Testing**: [test/infra/repositories/load_next_event_from_api_with_cache_fallback_repo_test.dart](fluttter_avancado_tdd_clean_arch/test/infra/repositories/load_next_event_from_api_with_cache_fallback_repo_test.dart)

---

### 7. Composite Pattern (with Mixins)

**Category**: Structural Pattern

**What it solves**: How to compose complex interfaces from simpler ones

**Real-world analogy**: Like **LEGO blocks**—combine small pieces to build complex structures.

#### Implementation with Mixins

**File**: [lib/infra/mappers/mapper.dart](fluttter_avancado_tdd_clean_arch/lib/infra/mappers/mapper.dart)

```dart
// Small, focused interfaces
abstract interface class DtoMapper<Dto> {
  Dto toDto(Json json);
}

abstract interface class JsonMapper<Dto> {
  Json toJson(Dto dto);
}

// Bidirectional mapper combines both
abstract interface class Mapper<Dto>
    implements DtoMapper<Dto>, JsonMapper<Dto> {}

// Mixin: Adds list conversion capability
mixin DtoListMapper<Dto> implements DtoMapper<Dto> {
  List<Dto> toDtoList(dynamic arr) =>
      arr.map<Dto>((item) => toDto(item)).toList();
}

// Mixin: Adds JSON array conversion
mixin JsonArrMapper<Dto> implements JsonMapper<Dto> {
  JsonArr toJsonArr(List<Dto> list) =>
      list.map((dto) => toJson(dto)).toList();
}

// Composite: Combines both mixins
abstract base class ListMapper<Dto>
    with DtoListMapper<Dto>, JsonArrMapper<Dto> {}
```

**Usage**:
```dart
// File: lib/infra/mappers/next_event_player_mapper.dart
final class NextEventPlayerMapper extends ListMapper<NextEventPlayer> {
  // Only implement core methods
  @override
  NextEventPlayer toDto(dynamic json) => NextEventPlayer(...);

  @override
  Json toJson(NextEventPlayer player) => {...};

  // Get these for free from mixins:
  // - toDtoList()
  // - toJsonArr()
}

// Now we can use:
final players = mapper.toDtoList(jsonArray);  // Inherited from mixin!
```

**Benefits**:
- ✅ Code reuse (list logic defined once)
- ✅ Flexible composition
- ✅ Interface segregation (use only what you need)

**Files**:
- Interface: [lib/infra/mappers/mapper.dart](fluttter_avancado_tdd_clean_arch/lib/infra/mappers/mapper.dart)
- Implementation: [lib/infra/mappers/next_event_player_mapper.dart](fluttter_avancado_tdd_clean_arch/lib/infra/mappers/next_event_player_mapper.dart)

---

## Behavioral Patterns

Patterns that deal with communication between objects.

---

### 8. Observer Pattern (Streams)

**Category**: Behavioral Pattern

**What it solves**: How to notify multiple objects when something changes

**Real-world analogy**: Like **subscribing to a YouTube channel**—when a new video is uploaded, all subscribers get notified automatically.

#### Implementation with RxDart

**File**: [lib/presentation/rx/next_event_rx_presenter.dart](fluttter_avancado_tdd_clean_arch/lib/presentation/rx/next_event_rx_presenter.dart)

```dart
final class NextEventRxPresenter implements NextEventPresenter {
  // Subjects are observable streams you can add data to
  final nextEventSubject = BehaviorSubject<NextEventViewModel>();
  final isBusySubject = BehaviorSubject<bool>();

  // Expose streams (read-only) to observers
  @override
  Stream<NextEventViewModel> get nextEventStream => nextEventSubject.stream;

  @override
  Stream<bool> get isBusyStream => isBusySubject.stream;

  // When data changes, notify all observers
  Future<void> loadNextEvent({required String groupId}) async {
    try {
      final event = await nextEventLoader(groupId: groupId);
      nextEventSubject.add(_mapEvent(event));  // 📢 Notify observers!
    } catch (error) {
      nextEventSubject.addError(error);  // 📢 Notify of error!
    }
  }

  @override
  void dispose() {
    nextEventSubject.close();  // Stop notifications
    isBusySubject.close();
  }
}
```

**Observer (UI subscribing)**:
```dart
// File: lib/ui/pages/next_event_page.dart
@override
void initState() {
  super.initState();

  // Observer 1: Listen to data stream
  StreamBuilder<NextEventViewModel>(
    stream: widget.presenter.nextEventStream,  // Subscribe
    builder: (context, snapshot) {
      // Automatically rebuilds when new data arrives
    },
  );

  // Observer 2: Listen to loading stream
  _isBusySubscription = widget.presenter.isBusyStream.listen((isBusy) {
    isBusy ? showLoading() : hideLoading();
  });
}
```

**Flow**:
```
Presenter                          UI (Observer)
    │                                  │
    │  nextEventSubject.add(data) ────►│ StreamBuilder rebuilds
    │                                  │
    │  isBusySubject.add(true) ────────►│ Show loading dialog
    │                                  │
    │  isBusySubject.add(false) ───────►│ Hide loading dialog
```

**Benefits**:
- ✅ UI automatically updates when data changes
- ✅ Multiple observers can listen to same stream
- ✅ Decoupled (presenter doesn't know about UI)

**RxDart's BehaviorSubject**:
- Remembers last value
- New subscribers immediately receive last value
- Perfect for state management

**Testing**: [test/presentation/rx/next_event_rx_presenter_test.dart](fluttter_avancado_tdd_clean_arch/test/presentation/rx/next_event_rx_presenter_test.dart)

---

### 9. Strategy Pattern

**Category**: Behavioral Pattern

**What it solves**: How to switch between different algorithms/approaches at runtime

**Real-world analogy**: Like **choosing a route to work**—you might take the highway (fast but toll), local roads (slower but free), or public transport (cheapest). Same goal, different strategies.

#### Implementation: Cache Fallback Strategy

**File**: [lib/infra/repositories/load_next_event_from_api_with_cache_fallback_repo.dart](fluttter_avancado_tdd_clean_arch/lib/infra/repositories/load_next_event_from_api_with_cache_fallback_repo.dart)

```dart
final class LoadNextEventFromApiWithCacheFallbackRepository {
  final LoadNextEventRepository loadNextEventFromApi;    // Strategy 1
  final LoadNextEventRepository loadNextEventFromCache;  // Strategy 2

  Future<NextEvent> loadNextEvent({required String groupId}) async {
    try {
      // Strategy 1: Try API first (preferred)
      return await loadNextEventFromApi(groupId: groupId);
    } on SessionExpiredError {
      rethrow;
    } catch (error) {
      // Strategy 2: Fallback to cache
      return await loadNextEventFromCache(groupId: groupId);
    }
  }
}
```

**Different strategies**:

| Strategy | When Used | Benefit |
|----------|-----------|---------|
| API | Has internet | Fresh data |
| Cache | No internet / API fails | Offline support |

**Both strategies share same interface**:
```dart
typedef LoadNextEventRepository = Future<NextEvent> Function({
  required String groupId,
});
```

**Benefits**:
- ✅ Strategies are interchangeable
- ✅ Easy to add new strategies (e.g., local database)
- ✅ Each strategy can be tested independently

**Testing**: [test/infra/repositories/load_next_event_from_api_with_cache_fallback_repo_test.dart](fluttter_avancado_tdd_clean_arch/test/infra/repositories/load_next_event_from_api_with_cache_fallback_repo_test.dart)

---

### 10. Sealed Class Pattern (Dart 3)

**Category**: Behavioral Pattern (Type-Safe Error Handling)

**What it solves**: How to define a closed set of types with exhaustive checking

**Real-world analogy**: Like **traffic lights**—can only be red, yellow, or green. No other colors allowed, and you must handle all three cases.

#### Implementation

**File**: [lib/domain/entities/errors.dart](fluttter_avancado_tdd_clean_arch/lib/domain/entities/errors.dart)

```dart
// Sealed class: Can only be implemented in this file
sealed class DomainError {
  String description();
}

// Specific error types
final class UnexpectedError implements DomainError {
  @override
  String description() => 'Algo inesperado aconteceu';
}

final class SessionExpiredError implements DomainError {
  @override
  String description() => 'Sua sessão expirou';
}
```

**Usage with exhaustive checking**:
```dart
String handleError(DomainError error) {
  // Compiler enforces handling ALL cases (no default needed!)
  return switch (error) {
    UnexpectedError() => 'Try again',
    SessionExpiredError() => 'Please login',
    // If you add new error type, compiler forces you to handle it
  };
}
```

**Comparison with Enum**:

```dart
// ❌ Old way: Enum (limited)
enum DomainError {
  unexpected,
  sessionExpired,
}

// ✅ New way: Sealed class (powerful)
sealed class DomainError {
  String description();  // Can have abstract methods!
}

class UnexpectedError implements DomainError {
  final String? details;  // Can have different properties!

  @override
  String description() => 'Error: ${details ?? 'unknown'}';
}
```

**Benefits of Sealed Classes**:
- ✅ Exhaustive checking (compiler enforces handling all cases)
- ✅ Each subclass can have different properties
- ✅ Can have abstract methods
- ✅ More flexible than enums
- ✅ Type-safe

**Why sealed?**
- Cannot be instantiated directly
- Can only be implemented in same file
- Compiler knows ALL possible implementations

**Usage in HTTP Adapter**:
```dart
// File: lib/infra/api/adapters/http_adapter.dart:66-71
T _handleResponse<T>(Response response) {
  return switch (response.statusCode) {
    200 => jsonDecode(response.body),
    401 => throw SessionExpiredError(),  // Type-safe error
    _ => throw UnexpectedError(),
  };
}
```

**Usage in UI**:
```dart
// File: lib/ui/pages/next_event_page.dart:50-54
if (snapshot.hasError) {
  return ErrorLayout(
    message: (snapshot.error as DomainError).description(),
    onRetry: () => widget.presenter.loadNextEvent(...),
  );
}
```

---

## SOLID Principles in Action

SOLID is an acronym for 5 design principles that make software more maintainable.

---

### S - Single Responsibility Principle

**Principle**: A class should have only ONE reason to change.

**Real-world analogy**: A chef should cook, not also clean, serve, and manage inventory. Each person has one job.

#### Examples

✅ **GOOD**: Each class has one responsibility

```dart
// File: lib/infra/api/adapters/http_adapter.dart
final class HttpAdapter implements HttpGetClient {
  // ONLY responsibility: Wrap HTTP client
  Future<dynamic> get({required String url, ...}) async {
    final response = await client.get(...);
    return _handleResponse(response);
  }
}

// File: lib/infra/mappers/next_event_mapper.dart
final class NextEventMapper implements Mapper<NextEvent> {
  // ONLY responsibility: Map NextEvent
  NextEvent toDto(dynamic json) => NextEvent(...);
  Json toJson(NextEvent event) => {...};
}

// File: lib/infra/api/repositories/load_next_event_api_repo.dart
final class LoadNextEventApiRepository {
  // ONLY responsibility: Load from API
  Future<NextEvent> loadNextEvent({required String groupId}) async {
    final json = await httpClient.get(...);
    return mapper.toDto(json);
  }
}
```

❌ **BAD**: God class with multiple responsibilities

```dart
// Anti-pattern: One class doing everything
class EventService {
  Future<NextEvent> loadNextEvent() async {
    // HTTP logic
    final response = await http.get(...);

    // JSON parsing
    final json = jsonDecode(response.body);

    // Entity mapping
    final event = NextEvent(...);

    // Caching logic
    await cacheManager.save(...);

    // UI logic
    final viewModel = _categorizePlayersForUI(event);

    return event;
  }
}
// This has 5 reasons to change! (HTTP, JSON, mapping, cache, UI)
```

**Benefits**:
- ✅ Easy to test (focused tests)
- ✅ Easy to understand (clear purpose)
- ✅ Easy to modify (change doesn't ripple)

---

### O - Open/Closed Principle

**Principle**: Open for extension, closed for modification.

**Real-world analogy**: Electrical outlets—you can plug in new devices (extension) without rewiring the house (modification).

#### Example: Mapper Interfaces

**File**: [lib/infra/mappers/mapper.dart](fluttter_avancado_tdd_clean_arch/lib/infra/mappers/mapper.dart)

```dart
// Base interfaces (closed for modification)
abstract interface class DtoMapper<Dto> {
  Dto toDto(Json json);
}

abstract interface class JsonMapper<Dto> {
  Json toJson(Dto dto);
}

// Can extend by creating new implementations (open for extension)
final class NextEventMapper implements Mapper<NextEvent> {
  // New mapper without modifying base interfaces
}

final class NextEventPlayerMapper extends ListMapper<NextEventPlayer> {
  // Another mapper without modifying base interfaces
}

// Can add UserMapper, ProductMapper, etc. without changing interfaces!
```

**Benefits**:
- ✅ Add new features without modifying existing code
- ✅ Less risk of breaking existing functionality
- ✅ Respects "Don't touch working code" principle

---

### L - Liskov Substitution Principle

**Principle**: Subtypes must be substitutable for their base types.

**Real-world analogy**: Any car (SUV, sedan, sports car) should work with a gas station. You shouldn't need a special gas station for each car type.

#### Example: Repository Substitution

```dart
// Base contract (function signature)
typedef LoadNextEventRepository = Future<NextEvent> Function({
  required String groupId,
});

// Implementation 1: API
final class LoadNextEventApiRepository {
  Future<NextEvent> loadNextEvent({required String groupId}) async {
    // Fetch from API
  }
}

// Implementation 2: Cache
final class LoadNextEventCacheRepository {
  Future<NextEvent> loadNextEvent({required String groupId}) async {
    // Fetch from cache
  }
}

// Both can be used interchangeably!
final LoadNextEventRepository repo1 = LoadNextEventApiRepository(...).loadNextEvent;
final LoadNextEventRepository repo2 = LoadNextEventCacheRepository(...).loadNextEvent;

// Composite uses both without caring about implementation
final class LoadNextEventFromApiWithCacheFallbackRepository {
  final LoadNextEventRepository loadNextEventFromApi;
  final LoadNextEventRepository loadNextEventFromCache;

  Future<NextEvent> loadNextEvent({required String groupId}) async {
    try {
      return await loadNextEventFromApi(groupId: groupId);  // Substitute!
    } catch (e) {
      return await loadNextEventFromCache(groupId: groupId);  // Substitute!
    }
  }
}
```

**Benefits**:
- ✅ Implementations are interchangeable
- ✅ Easy to mock for testing
- ✅ Flexible composition

---

### I - Interface Segregation Principle

**Principle**: Clients shouldn't depend on interfaces they don't use.

**Real-world analogy**: A printer should have different buttons for different users—admins get advanced controls, regular users get simple ones. Don't force everyone to learn all controls.

#### Example 1: Separate Cache Interfaces

**Files**:
- [lib/infra/cache/clients/cache_get_client.dart](fluttter_avancado_tdd_clean_arch/lib/infra/cache/clients/cache_get_client.dart)
- [lib/infra/cache/clients/cache_save_client.dart](fluttter_avancado_tdd_clean_arch/lib/infra/cache/clients/cache_save_client.dart)

```dart
// ✅ GOOD: Separate interfaces
abstract interface class CacheGetClient {
  Future<dynamic> get({required String key});
}

abstract interface class CacheSaveClient {
  Future<void> save({required String key, required dynamic value});
}

// Classes implement only what they need
final class LoadNextEventCacheRepository {
  final CacheGetClient cacheClient;  // Only needs read!
}

final class LoadNextEventFromApiWithCacheFallbackRepository {
  final CacheSaveClient cacheClient;  // Only needs write!
}

// Adapter implements both (because it can do both)
final class CacheManagerAdapter
    implements CacheGetClient, CacheSaveClient {
  // Implements both
}
```

❌ **BAD**: One big interface

```dart
// Forces all clients to implement both methods
abstract interface class CacheClient {
  Future<dynamic> get({required String key});
  Future<void> save({required String key, required dynamic value});
}

// What if a class only needs to read? Still must implement save!
final class ReadOnlyRepository implements CacheClient {
  Future<dynamic> get({required String key}) => ...;

  Future<void> save(...) => throw UnimplementedError();  // Useless method!
}
```

#### Example 2: Mapper Segregation

**File**: [lib/infra/mappers/mapper.dart](fluttter_avancado_tdd_clean_arch/lib/infra/mappers/mapper.dart)

```dart
// Small, focused interfaces
abstract interface class DtoMapper<Dto> {
  Dto toDto(Json json);
}

abstract interface class JsonMapper<Dto> {
  Json toJson(Dto dto);
}

// Repositories only need DtoMapper (not JsonMapper)
final class LoadNextEventApiRepository {
  final DtoMapper<NextEvent> mapper;  // Only needs toDto()!
}

// Composite repository needs JsonMapper
final class LoadNextEventFromApiWithCacheFallbackRepository {
  final JsonMapper<NextEvent> mapper;  // Only needs toJson()!
}

// Full mapper implements both
final class NextEventMapper implements Mapper<NextEvent> {
  // Implements both when needed
}
```

**Benefits**:
- ✅ Classes only depend on methods they use
- ✅ Easier to understand (smaller interfaces)
- ✅ More flexible composition

---

### D - Dependency Inversion Principle

**Principle**: Depend on abstractions, not concretions.

**Real-world analogy**: Your phone charger uses a USB interface—you don't need a different charger for each phone model. The charger depends on the USB standard (abstraction), not specific phones (concretions).

#### Example 1: Presenter Depends on Function Signature

**File**: [lib/presentation/rx/next_event_rx_presenter.dart](fluttter_avancado_tdd_clean_arch/lib/presentation/rx/next_event_rx_presenter.dart)

```dart
// ✅ GOOD: Depends on abstraction (function signature)
final class NextEventRxPresenter implements NextEventPresenter {
  final Future<NextEvent> Function({required String groupId}) nextEventLoader;

  NextEventRxPresenter({required this.nextEventLoader});

  Future<void> loadNextEvent({required String groupId}) async {
    final event = await nextEventLoader(groupId: groupId);
    nextEventSubject.add(_mapEvent(event));
  }
}

// Can inject ANY function that matches signature
final presenter1 = NextEventRxPresenter(
  nextEventLoader: apiRepo.loadNextEvent,
);

final presenter2 = NextEventRxPresenter(
  nextEventLoader: cacheRepo.loadNextEvent,
);

final presenter3 = NextEventRxPresenter(
  nextEventLoader: mockRepo.loadNextEvent,  // Test double!
);
```

❌ **BAD**: Depends on concrete class

```dart
// Tightly coupled to specific repository
final class NextEventRxPresenter {
  final LoadNextEventApiRepository repository;  // Concrete class!

  Future<void> loadNextEvent({required String groupId}) async {
    final event = await repository.loadNextEvent(groupId: groupId);
    // Can ONLY use API repository, can't swap or test easily
  }
}
```

#### Example 2: Repository Depends on Interface

**File**: [lib/infra/api/repositories/load_next_event_api_repo.dart](fluttter_avancado_tdd_clean_arch/lib/infra/api/repositories/load_next_event_api_repo.dart)

```dart
// ✅ GOOD: Depends on interface
final class LoadNextEventApiRepository {
  final HttpGetClient httpClient;  // Interface!
  final DtoMapper<NextEvent> mapper;  // Interface!

  // Can inject any implementation
}

// In production
final repo = LoadNextEventApiRepository(
  httpClient: HttpAdapter(client: Client()),
  mapper: NextEventMapper(...),
);

// In tests
final repo = LoadNextEventApiRepository(
  httpClient: MockHttpClient(),
  mapper: MockMapper(),
);
```

**Dependency Inversion Diagram**:
```
High-Level Module (Presenter)
        │
        │ depends on
        ↓
    Abstraction (Function signature / Interface)
        ▲
        │ implements
        │
Low-Level Module (Repository)
```

**Benefits**:
- ✅ High-level code doesn't depend on low-level details
- ✅ Easy to swap implementations
- ✅ Easy to test (inject mocks)
- ✅ Flexible and maintainable

---

## Pattern Summary Table

| Pattern | Category | Problem Solved | Files |
|---------|----------|----------------|-------|
| **Clean Architecture** | Architectural | Organize code in layers | Entire project |
| **MVP** | Architectural | Separate UI from logic | Presentation + UI layers |
| **Factory** | Creational | Object creation | `lib/main/factories/`, Entity constructors |
| **Singleton** | Creational | One instance | Presenter (implicit) |
| **Adapter** | Structural | Wrap external libraries | `lib/infra/*/adapters/` |
| **Repository** | Structural | Abstract data sources | `lib/infra/*/repositories/` |
| **Composite** | Structural | Combine interfaces | `lib/infra/mappers/mapper.dart` |
| **Observer** | Behavioral | Notify on changes | RxDart Streams |
| **Strategy** | Behavioral | Switch algorithms | API with cache fallback |
| **Sealed Class** | Behavioral | Type-safe error handling | `lib/domain/entities/errors.dart` |

---

## Learning Path

### Beginner Level
1. Start with **Factory Pattern** (entity creation)
2. Understand **Repository Pattern** (data abstraction)
3. Learn **MVP Pattern** (separate UI from logic)

### Intermediate Level
4. Study **Adapter Pattern** (isolate dependencies)
5. Master **Observer Pattern** (reactive programming with Streams)
6. Apply **SOLID principles** (especially SRP and DIP)

### Advanced Level
7. Explore **Clean Architecture** (full system design)
8. Understand **Strategy Pattern** (algorithm selection)
9. Use **Sealed Classes** (type-safe error handling)
10. Master **Composite Pattern** with mixins

---

## Key Takeaways

✅ **Patterns are tools**, not rules—use when they solve real problems
✅ **SOLID principles** guide good design decisions
✅ **Dependency Inversion** enables testability and flexibility
✅ **Each layer** demonstrates multiple patterns working together
✅ **Clean Architecture** is the foundation that enables all other patterns

---

## Next Steps

Continue learning:
- **[Component Deep-Dive](04-COMPONENT-DEEP-DIVE.md)**: See patterns in action with data flow
- **[Getting Started Guide](05-GETTING-STARTED-GUIDE.md)**: Apply patterns yourself
- **Practice**: Try adding new features following these patterns
