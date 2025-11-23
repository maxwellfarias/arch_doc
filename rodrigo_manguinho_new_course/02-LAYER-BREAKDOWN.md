# Layer Breakdown - Detailed Explanation of Each Layer

## Table of Contents
- [Domain Layer](#domain-layer)
- [Infrastructure Layer](#infrastructure-layer)
- [Presentation Layer](#presentation-layer)
- [UI Layer](#ui-layer)
- [Composition Root](#composition-root)

---

## Domain Layer

**Location**: `lib/domain/`

**Analogy**: Think of this as the **rulebook** of your application—the fundamental truths that never change regardless of technology.

### Purpose

The Domain layer contains:
- Pure business entities (the "nouns" of your business)
- Business rules (how things work)
- Domain errors (what can go wrong)

### Key Characteristics

✅ **No dependencies** - Uses only pure Dart (no Flutter, no external packages)
✅ **Highly testable** - No framework required to test
✅ **Framework agnostic** - Could be used in a web app, mobile app, or server
✅ **Immutable entities** - Data cannot be changed after creation

---

### Entities

#### 1. NextEvent Entity

**File**: [lib/domain/entities/next_event.dart](fluttter_avancado_tdd_clean_arch/lib/domain/entities/next_event.dart)

```dart
final class NextEvent {
  final String groupName;
  final DateTime date;
  final List<NextEventPlayer> players;

  const NextEvent({
    required this.groupName,
    required this.date,
    required this.players,
  });
}
```

**What it represents**: A sports event (soccer game) with players

**Key points**:
- `final` class means it cannot be extended (sealed for concrete implementation)
- All fields are `final` (immutable)
- `const` constructor allows compile-time constants
- No business logic (pure data container)

**Why immutable?**
- Predictable behavior (no hidden changes)
- Thread-safe (safe for concurrent operations)
- Easy to test (no state mutations)
- Prevents bugs (can't accidentally modify)

---

#### 2. NextEventPlayer Entity

**File**: [lib/domain/entities/next_event_player.dart](fluttter_avancado_tdd_clean_arch/lib/domain/entities/next_event_player.dart)

```dart
final class NextEventPlayer {
  final String id;
  final String name;
  final String initials;        // ⚡ Auto-generated from name
  final bool? isConfirmed;
  final String? position;       // 'goalkeeper', 'defender', etc.
  final String? photo;
  final DateTime? confirmationDate;

  // Private constructor (cannot be called directly)
  const NextEventPlayer._({
    required this.id,
    required this.name,
    required this.initials,
    this.isConfirmed,
    this.position,
    this.photo,
    this.confirmationDate,
  });

  // Public factory constructor (calls private constructor)
  factory NextEventPlayer({
    required String id,
    required String name,
    bool? isConfirmed,
    String? position,
    String? photo,
    DateTime? confirmationDate,
  }) =>
      NextEventPlayer._(
        id: id,
        name: name,
        initials: _getInitials(name),  // 🎯 Business logic!
        isConfirmed: isConfirmed,
        position: position,
        photo: photo,
        confirmationDate: confirmationDate,
      );

  // Business logic: Generate initials from name
  static String _getInitials(String name) {
    final names = name.toUpperCase().trim().split(' ');
    final firstChar = names.first.split('').firstOrNull ?? '-';
    final lastChar = names.last.split('').elementAtOrNull(
      names.length == 1 ? 1 : 0,
    ) ?? '';
    return '$firstChar$lastChar';
  }
}
```

**Examples of initials generation**:
- "Maxwell Farias" → "MF" (first letter of first + first letter of last)
- "John" → "JO" (first two letters if only one name)
- "  " → "-" (handles empty/whitespace names)

**Why this design?**

1. **Factory Pattern**: Users call `NextEventPlayer(...)` not knowing about `_getInitials()`
2. **Encapsulation**: Initials generation is internal detail
3. **Consistency**: Impossible to create player with wrong initials
4. **Business Rule in Domain**: Logic belongs with the entity, not scattered

**Educational Note**: This is an example of "smart" domain entity (not completely anemic). The initials generation is a business rule that belongs in the domain.

---

#### 3. Domain Errors

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

**What is a Sealed Class?** (Dart 3 feature)

A sealed class:
- Cannot be instantiated directly
- Can only be implemented in the same file
- Compiler knows ALL possible implementations

**Benefits**:
```dart
// Compiler enforces exhaustive checking (no default case needed!)
String handleError(DomainError error) {
  return switch (error) {
    UnexpectedError() => 'Try again',
    SessionExpiredError() => 'Please login',
    // If you add new error, compiler forces you to handle it!
  };
}
```

**Why sealed instead of enum?**
- Each error can have different properties
- Each error can have different methods
- More flexible than enums
- Still type-safe and exhaustive

---

### Domain Layer Summary

**Files**:
- [lib/domain/entities/next_event.dart](fluttter_avancado_tdd_clean_arch/lib/domain/entities/next_event.dart)
- [lib/domain/entities/next_event_player.dart](fluttter_avancado_tdd_clean_arch/lib/domain/entities/next_event_player.dart)
- [lib/domain/entities/errors.dart](fluttter_avancado_tdd_clean_arch/lib/domain/entities/errors.dart)

**Key Patterns**:
- Immutable entities
- Factory pattern for entity creation
- Sealed classes for type-safe errors
- Business rules encapsulated in entities

**Testing**: See [test/domain/entities/next_event_player_test.dart](fluttter_avancado_tdd_clean_arch/test/domain/entities/next_event_player_test.dart)

---

## Infrastructure Layer

**Location**: `lib/infra/`

**Analogy**: Think of this as the **delivery service**—it fetches packages (data) from warehouses (APIs, databases) and brings them to your door (application).

### Purpose

The Infrastructure layer handles:
- Communication with external services (REST APIs, databases)
- Data transformation (JSON ↔ Domain entities)
- Caching strategies
- Low-level implementation details

### Key Characteristics

✅ **Depends only on Domain** - Uses domain entities
✅ **Abstracts external dependencies** - Wraps third-party libraries
✅ **Converts data formats** - JSON to domain entities and vice versa
✅ **Implements data fetching strategies** - API, cache, combinations

---

### Sub-Layer 1: Clients (Interfaces)

These define contracts for external services.

#### HttpGetClient Interface

**File**: [lib/infra/api/clients/http_get_client.dart](fluttter_avancado_tdd_clean_arch/lib/infra/api/clients/http_get_client.dart)

```dart
typedef Json = Map<String, dynamic>;

abstract interface class HttpGetClient {
  Future<dynamic> get({
    required String url,
    Json? headers,
    Json? params,
    Json? queryString,
  });
}
```

**Why an interface?**
- Repository doesn't care about HTTP implementation details
- Easy to mock for testing
- Can swap implementations (e.g., use Dio instead of http package)

#### CacheGetClient & CacheSaveClient Interfaces

**Files**:
- [lib/infra/cache/clients/cache_get_client.dart](fluttter_avancado_tdd_clean_arch/lib/infra/cache/clients/cache_get_client.dart)
- [lib/infra/cache/clients/cache_save_client.dart](fluttter_avancado_tdd_clean_arch/lib/infra/cache/clients/cache_save_client.dart)

```dart
abstract interface class CacheGetClient {
  Future<dynamic> get({required String key});
}

abstract interface class CacheSaveClient {
  Future<void> save({required String key, required dynamic value});
}
```

**Why separate Get and Save?**
- **Interface Segregation Principle** (SOLID)
- Some classes only need to read, others only write
- Don't force classes to implement unused methods

---

### Sub-Layer 2: Adapters (Wrappers)

Adapters wrap external libraries to implement the client interfaces.

#### HttpAdapter

**File**: [lib/infra/api/adapters/http_adapter.dart](fluttter_avancado_tdd_clean_arch/lib/infra/api/adapters/http_adapter.dart)

```dart
import 'package:http/http.dart';  // External package

final class HttpAdapter implements HttpGetClient {
  final Client client;

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

  // Build URI with parameter replacement and query string
  Uri _buildUri({required String url, Json? params, Json? queryString}) {
    // Replace :paramName in URL with actual values
    // Example: "/groups/:groupId/event" with {groupId: "123"}
    //       → "/groups/123/event"
    url = params?.keys.fold(
          url,
          (result, key) =>
              result.replaceFirst(':$key', params[key]?.toString() ?? ''),
        ).removeSuffix('/') ?? url;

    // Add query string
    // Example: {page: "1", limit: "10"} → "?page=1&limit=10"
    url = queryString?.keys
          .fold(
            '$url?',
            (result, key) => '$result$key=${queryString[key]}&',
          )
          .removeSuffix('&') ?? url;

    return Uri.parse(url);
  }

  // Build headers with defaults
  Map<String, String> _buildHeaders({required String url, Json? headers}) {
    final defaultHeaders = {
      'content-type': 'application/json',
      'accept': 'application/json',
    };
    return {...defaultHeaders, ...?headers?.cast<String, String>()};
  }

  // Handle HTTP response and convert status codes to errors
  T _handleResponse<T>(Response response) {
    return switch (response.statusCode) {
      200 => jsonDecode(response.body),
      401 => throw SessionExpiredError(),
      _ => throw UnexpectedError(),
    };
  }
}
```

**What this adapter does**:

1. **URL Parameter Replacement**: `/groups/:groupId` → `/groups/123`
2. **Query String Building**: `{page: 1}` → `?page=1`
3. **Default Headers**: Adds JSON content-type and accept
4. **Status Code Handling**: Converts HTTP errors to domain errors
5. **JSON Decoding**: Parses response body automatically

**Why wrap the http package?**
- ✅ Repositories don't depend on external package
- ✅ Easier testing (mock interface, not package)
- ✅ Centralized error handling
- ✅ Can swap HTTP libraries without changing repositories

---

#### CacheManagerAdapter

**File**: [lib/infra/cache/adapters/cache_manager_adapter.dart](fluttter_avancado_tdd_clean_arch/lib/infra/cache/adapters/cache_manager_adapter.dart)

```dart
import 'package:flutter_cache_manager/flutter_cache_manager.dart';

final class CacheManagerAdapter
    implements CacheGetClient, CacheSaveClient {
  final BaseCacheManager client;

  const CacheManagerAdapter({required this.client});

  @override
  Future<dynamic> get({required String key}) async {
    final info = await client.getFileFromCache(key);

    // Check if cache is valid and file exists
    if (info?.validTill.isBefore(DateTime.now()) != false ||
        !await info!.file.exists()) {
      return null;
    }

    // Read and parse JSON file
    final data = await info.file.readAsString();
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

**What this adapter does**:

1. **Cache Validation**: Checks expiration date
2. **File Existence Check**: Ensures file wasn't deleted
3. **JSON Encoding/Decoding**: Stores as JSON files
4. **Null Safety**: Returns null if cache invalid/missing

---

### Sub-Layer 3: Mappers (Transformers)

Mappers convert between JSON (from APIs) and Domain entities.

#### Mapper Interfaces

**File**: [lib/infra/mappers/mapper.dart](fluttter_avancado_tdd_clean_arch/lib/infra/mappers/mapper.dart)

```dart
typedef Json = Map<String, dynamic>;
typedef JsonArr = List<Json>;

// Convert JSON to Domain entity
abstract interface class DtoMapper<Dto> {
  Dto toDto(Json json);
}

// Convert Domain entity to JSON
abstract interface class JsonMapper<Dto> {
  Json toJson(Dto dto);
}

// Bidirectional mapper
abstract interface class Mapper<Dto>
    implements DtoMapper<Dto>, JsonMapper<Dto> {}

// Mixin for list conversion (uses toDto internally)
mixin DtoListMapper<Dto> implements DtoMapper<Dto> {
  List<Dto> toDtoList(dynamic arr) =>
      arr.map<Dto>((item) => toDto(item)).toList();
}

// Mixin for JSON array conversion (uses toJson internally)
mixin JsonArrMapper<Dto> implements JsonMapper<Dto> {
  JsonArr toJsonArr(List<Dto> list) =>
      list.map((dto) => toJson(dto)).toList();
}

// Base class that combines both mixins
abstract base class ListMapper<Dto>
    with DtoListMapper<Dto>, JsonArrMapper<Dto> {}
```

**Why this complex structure?**

1. **Interface Segregation**: Classes can implement only what they need
2. **Mixins for Reuse**: `toDtoList()` logic defined once, reused everywhere
3. **Composition over Inheritance**: Flexible combination of capabilities

**Example usage**:
```dart
// If mapper extends ListMapper<Player>:
final players = mapper.toDtoList(jsonArray);  // Automatically maps all items
```

---

#### NextEventPlayerMapper

**File**: [lib/infra/mappers/next_event_player_mapper.dart](fluttter_avancado_tdd_clean_arch/lib/infra/mappers/next_event_player_mapper.dart)

```dart
final class NextEventPlayerMapper extends ListMapper<NextEventPlayer> {
  @override
  NextEventPlayer toDto(dynamic json) => NextEventPlayer(
        id: json['id'],
        name: json['name'],
        isConfirmed: json['isConfirmed'],
        position: json['position'],
        photo: json['photo'],
        confirmationDate: DateTime.tryParse(json['confirmationDate'] ?? ''),
      );

  @override
  Json toJson(NextEventPlayer player) => {
        'id': player.id,
        'name': player.name,
        'isConfirmed': player.isConfirmed,
        'position': player.position,
        'photo': player.photo,
        'confirmationDate': player.confirmationDate?.toIso8601String(),
      };
}
```

**What it does**:
- `toDto()`: JSON → NextEventPlayer entity
- `toJson()`: NextEventPlayer → JSON
- `toDtoList()`: JSON array → List<NextEventPlayer> (inherited from mixin)
- `toJsonArr()`: List<NextEventPlayer> → JSON array (inherited from mixin)

**Example**:
```dart
// Input JSON
{
  "id": "1",
  "name": "Cristiano Ronaldo",
  "isConfirmed": true,
  "position": "forward",
  "photo": "http://...",
  "confirmationDate": "2024-01-10T10:00:00.000Z"
}

// After toDto()
NextEventPlayer(
  id: "1",
  name: "Cristiano Ronaldo",
  initials: "CR",  // Auto-generated by entity!
  isConfirmed: true,
  position: "forward",
  photo: "http://...",
  confirmationDate: DateTime(2024, 1, 10, 10, 0, 0)
)
```

---

#### NextEventMapper

**File**: [lib/infra/mappers/next_event_mapper.dart](fluttter_avancado_tdd_clean_arch/lib/infra/mappers/next_event_mapper.dart)

```dart
final class NextEventMapper implements Mapper<NextEvent> {
  final DtoListMapper<NextEventPlayer> playerMapper;

  const NextEventMapper({required this.playerMapper});

  @override
  NextEvent toDto(dynamic json) => NextEvent(
        groupName: json['groupName'],
        date: DateTime.parse(json['date']),
        players: playerMapper.toDtoList(json['players']),  // Delegate to player mapper
      );

  @override
  Json toJson(NextEvent event) => {
        'groupName': event.groupName,
        'date': event.date.toIso8601String(),
        'players': playerMapper.toJsonArr(event.players),  // Delegate to player mapper
      };
}
```

**Dependency Injection**: Notice `playerMapper` is injected! This mapper doesn't create its own player mapper—it receives one.

**Why?**
- ✅ Single Responsibility (doesn't know how to map players)
- ✅ Testable (can inject mock player mapper)
- ✅ Reusable (player mapper used elsewhere)

---

### Sub-Layer 4: Repositories

Repositories coordinate data fetching and use adapters/mappers.

#### LoadNextEventApiRepository

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
      params: {"groupId": groupId},  // Replaces :groupId in URL
    );
    return mapper.toDto(event);  // JSON → NextEvent entity
  }
}
```

**Responsibilities**:
1. ✅ Call HTTP client with correct URL
2. ✅ Pass parameters
3. ✅ Delegate JSON parsing to mapper
4. ✅ Return domain entity

**What it does NOT do**:
- ❌ Know about HTTP implementation
- ❌ Parse JSON itself
- ❌ Handle UI concerns

**Testing**: See [test/infra/api/repositories/load_next_event_api_repo_test.dart](fluttter_avancado_tdd_clean_arch/test/infra/api/repositories/load_next_event_api_repo_test.dart)

---

#### LoadNextEventCacheRepository

**File**: [lib/infra/cache/repositories/load_next_event_cache_repo.dart](fluttter_avancado_tdd_clean_arch/lib/infra/cache/repositories/load_next_event_cache_repo.dart)

```dart
final class LoadNextEventCacheRepository {
  final CacheGetClient cacheClient;
  final String key;
  final Mapper<NextEvent> mapper;

  const LoadNextEventCacheRepository({
    required this.cacheClient,
    required this.key,
    required this.mapper,
  });

  Future<NextEvent> loadNextEvent({required String groupId}) async {
    final json = await cacheClient.get(key: '$key:$groupId');
    if (json == null) throw UnexpectedError();
    return mapper.toDto(json);
  }
}
```

**Key difference from API repository**:
- Uses cache key instead of URL
- Returns error if cache miss (no data)
- Otherwise same structure (delegate to mapper)

---

#### LoadNextEventFromApiWithCacheFallbackRepository (Composite)

**File**: [lib/infra/repositories/load_next_event_from_api_with_cache_fallback_repo.dart](fluttter_avancado_tdd_clean_arch/lib/infra/repositories/load_next_event_from_api_with_cache_fallback_repo.dart)

```dart
final class LoadNextEventFromApiWithCacheFallbackRepository {
  final LoadNextEventRepository loadNextEventFromApi;
  final LoadNextEventRepository loadNextEventFromCache;
  final CacheSaveClient cacheClient;
  final JsonMapper<NextEvent> mappper;
  final String key;

  const LoadNextEventFromApiWithCacheFallbackRepository({
    required this.loadNextEventFromApi,
    required this.loadNextEventFromCache,
    required this.cacheClient,
    required this.mappper,
    required this.key,
  });

  Future<NextEvent> loadNextEvent({required String groupId}) async {
    try {
      // Step 1: Try loading from API
      final event = await loadNextEventFromApi(groupId: groupId);

      // Step 2: Save to cache for next time
      await cacheClient.save(
        key: '$key:$groupId',
        value: mappper.toJson(event),
      );

      // Step 3: Return fresh data
      return event;
    } on SessionExpiredError {
      // Don't catch session expired—user needs to login
      rethrow;
    } catch (error) {
      // Step 4: On any other error, try cache
      return loadNextEventFromCache(groupId: groupId);
    }
  }
}
```

**Strategy**:
1. ✅ Try API first (fresh data preferred)
2. ✅ Save to cache on success
3. ✅ Fallback to cache on failure
4. ✅ Always rethrow SessionExpiredError (user must re-authenticate)

**Why this design?**
- **Offline support**: App works without internet
- **Performance**: Faster subsequent loads
- **User experience**: Never show error if cached data exists
- **Composition**: Combines two strategies (API + Cache)

**Testing**: See [test/infra/repositories/load_next_event_from_api_with_cache_fallback_repo_test.dart](fluttter_avancado_tdd_clean_arch/test/infra/repositories/load_next_event_from_api_with_cache_fallback_repo_test.dart)

---

### Infrastructure Layer Summary

**Structure**:
```
infra/
├── api/
│   ├── adapters/       # HttpAdapter (wraps http package)
│   ├── clients/        # HttpGetClient interface
│   └── repositories/   # LoadNextEventApiRepository
├── cache/
│   ├── adapters/       # CacheManagerAdapter (wraps flutter_cache_manager)
│   ├── clients/        # CacheGetClient, CacheSaveClient interfaces
│   └── repositories/   # LoadNextEventCacheRepository
├── mappers/            # JSON ↔ Entity conversion
│   ├── mapper.dart     # Interfaces and mixins
│   ├── next_event_mapper.dart
│   └── next_event_player_mapper.dart
├── repositories/       # Composite repositories
│   └── load_next_event_from_api_with_cache_fallback_repo.dart
└── types/              # Type aliases (Json, JsonArr)
```

**Key Patterns**:
- Adapter Pattern (wrap external libraries)
- Repository Pattern (abstract data sources)
- Interface Segregation (small, focused interfaces)
- Dependency Injection (all dependencies injected)
- Mixin Composition (reusable list mapping logic)

---

## Presentation Layer

**Location**: `lib/presentation/`

**Analogy**: Think of this as a **personal assistant** who takes raw information and organizes it perfectly for your needs.

### Purpose

The Presentation layer:
- Manages UI state (loading, loaded, error)
- Transforms domain entities into UI-ready ViewModels
- Handles UI-specific business logic (filtering, sorting, categorizing)
- Provides reactive streams for UI updates

### Key Characteristics

✅ **Depends only on Domain** - No infrastructure, no UI widgets
✅ **No Flutter imports** - Testable without Flutter framework
✅ **Uses RxDart** - Reactive programming with streams
✅ **MVP Pattern** - Model-View-Presenter architecture

---

### Presenter Interface

**File**: [lib/presentation/presenters/next_event_presenter.dart](fluttter_avancado_tdd_clean_arch/lib/presentation/presenters/next_event_presenter.dart)

```dart
abstract interface class NextEventPresenter {
  Stream<NextEventViewModel> get nextEventStream;
  Stream<bool> get isBusyStream;

  void dispose();
  Future<void> loadNextEvent({required String groupId, bool isReload = false});
}
```

**Why an interface?**
- ✅ UI depends on contract, not implementation
- ✅ Easy to mock for widget tests
- ✅ Can create different implementations (Bloc, Cubit, etc.)

---

### ViewModels

**File**: [lib/presentation/presenters/next_event_presenter.dart](fluttter_avancado_tdd_clean_arch/lib/presentation/presenters/next_event_presenter.dart) (same file)

```dart
final class NextEventViewModel {
  final List<NextEventPlayerViewModel> goalkeepers;  // Confirmed goalkeepers
  final List<NextEventPlayerViewModel> players;      // Confirmed field players
  final List<NextEventPlayerViewModel> out;          // Cannot attend
  final List<NextEventPlayerViewModel> doubt;        // Undecided

  const NextEventViewModel({
    this.goalkeepers = const [],
    this.players = const [],
    this.out = const [],
    this.doubt = const [],
  });
}

final class NextEventPlayerViewModel {
  final String name;
  final String initials;
  final String? photo;
  final String? position;
  final bool? isConfirmed;

  const NextEventPlayerViewModel({
    required this.name,
    required this.initials,
    this.photo,
    this.position,
    this.isConfirmed,
  });
}
```

**Why ViewModels are different from Entities?**

| Aspect | Entity (Domain) | ViewModel (Presentation) |
|--------|----------------|--------------------------|
| Purpose | Business data | UI data |
| Location | Domain layer | Presentation layer |
| Contains | All business fields | Only fields UI needs |
| Example | `id`, `confirmationDate` | Just display fields |

**Example**:
- Entity has `id` (needed by backend)—ViewModel doesn't (UI doesn't show it)
- Entity has `confirmationDate`—ViewModel doesn't need it (already sorted)
- ViewModel groups players by category—Entity has flat list

**Why ViewModels in same file as Presenter?**

From the code comments:
> "ViewModels are specific to this screen and unlikely to be reused elsewhere, since they are designed specifically for the UI they serve. Therefore, the professor prefers to keep both the presenter and the viewModel in the same file."

---

### RxDart Presenter Implementation

**File**: [lib/presentation/rx/next_event_rx_presenter.dart](fluttter_avancado_tdd_clean_arch/lib/presentation/rx/next_event_rx_presenter.dart)

```dart
import 'package:rxdart/rxdart.dart';

final class NextEventRxPresenter implements NextEventPresenter {
  final Future<NextEvent> Function({required String groupId}) nextEventLoader;

  // Subjects are streams that you can add data to
  final nextEventSubject = BehaviorSubject<NextEventViewModel>();
  final isBusySubject = BehaviorSubject<bool>();

  NextEventRxPresenter({required this.nextEventLoader});

  // Expose streams (not subjects) to UI
  @override
  Stream<NextEventViewModel> get nextEventStream => nextEventSubject.stream;

  @override
  Stream<bool> get isBusyStream => isBusySubject.stream;

  @override
  Future<void> loadNextEvent({
    required String groupId,
    bool isReload = false,
  }) async {
    try {
      // Show loading indicator only on refresh (not initial load)
      if (isReload) isBusySubject.add(true);

      // Load data using injected function
      final event = await nextEventLoader(groupId: groupId);

      // Transform domain entity to ViewModel and emit
      nextEventSubject.add(_mapEvent(event));
    } catch (error) {
      // Emit error through stream
      nextEventSubject.addError(error);
    } finally {
      // Hide loading indicator
      if (isReload) isBusySubject.add(false);
    }
  }

  @override
  void dispose() {
    nextEventSubject.close();
    isBusySubject.close();
  }

  // UI-specific business logic: categorize and sort players
  NextEventViewModel _mapEvent(NextEvent event) => NextEventViewModel(
        // DOUBT: Players who haven't decided (sorted alphabetically)
        doubt: event.players
            .where((player) => player.confirmationDate == null)
            .sortedBy((player) => player.name)
            .map(_mapPlayer)
            .toList(),

        // OUT: Players who cannot attend (sorted by confirmation time)
        out: event.players
            .where((player) =>
                player.confirmationDate != null &&
                player.isConfirmed == false)
            .sortedBy((player) => player.confirmationDate!)
            .map(_mapPlayer)
            .toList(),

        // GOALKEEPERS: Confirmed goalkeepers (sorted by confirmation time)
        goalkeepers: event.players
            .where((player) =>
                player.confirmationDate != null &&
                player.isConfirmed == true &&
                player.position == 'goalkeeper')
            .sortedBy((player) => player.confirmationDate!)
            .map(_mapPlayer)
            .toList(),

        // PLAYERS: Confirmed field players (sorted by confirmation time)
        players: event.players
            .where((player) =>
                player.confirmationDate != null &&
                player.isConfirmed == true &&
                player.position != 'goalkeeper')
            .sortedBy((player) => player.confirmationDate!)
            .map(_mapPlayer)
            .toList(),
      );

  NextEventPlayerViewModel _mapPlayer(NextEventPlayer player) =>
      NextEventPlayerViewModel(
        name: player.name,
        initials: player.initials,
        photo: player.photo,
        position: player.position,
        isConfirmed: player.isConfirmed,
      );
}
```

**Key Concepts**:

1. **BehaviorSubject**: A special stream that:
   - Remembers last value
   - Sends last value to new subscribers immediately
   - Perfect for state management

2. **Function Injection**: Instead of injecting a repository interface, injects a function
   ```dart
   final Future<NextEvent> Function({required String groupId}) nextEventLoader;
   ```
   **Why?** Simpler than creating UseCase classes. The function signature IS the contract.

3. **UI-Specific Logic in Presenter**: Filtering and sorting happen here, not in domain
   **Why?** Different UIs might need different sorting. Mobile shows by confirmation time; web might show alphabetically.

4. **Error Handling**: Errors emitted through stream (`nextEventSubject.addError(error)`)
   **Why?** UI listens to one stream and handles both data and errors

5. **Loading State**: Separate `isBusyStream` for loading indicators
   **Why?** Keeps loading state separate from data state

**Testing**: See [test/presentation/rx/next_event_rx_presenter_test.dart](fluttter_avancado_tdd_clean_arch/test/presentation/rx/next_event_rx_presenter_test.dart)

---

### Presentation Layer Summary

**Files**:
- [lib/presentation/presenters/next_event_presenter.dart](fluttter_avancado_tdd_clean_arch/lib/presentation/presenters/next_event_presenter.dart) - Interface & ViewModels
- [lib/presentation/rx/next_event_rx_presenter.dart](fluttter_avancado_tdd_clean_arch/lib/presentation/rx/next_event_rx_presenter.dart) - RxDart implementation

**Key Patterns**:
- MVP (Model-View-Presenter)
- Observer Pattern (Streams)
- Dependency Inversion (function injection)
- ViewModel Pattern (UI-specific data)

**Responsibilities**:
- ✅ Load data via injected function
- ✅ Transform domain entities to ViewModels
- ✅ Filter and categorize for UI
- ✅ Manage loading state
- ✅ Handle errors
- ✅ Provide reactive streams

---

## UI Layer

**Location**: `lib/ui/`

**Analogy**: Think of this as the **storefront**—everything the customer sees and interacts with.

### Purpose

The UI layer:
- Renders visual components (Flutter widgets)
- Listens to presenter streams
- Displays data to users
- Captures user input
- Triggers presenter actions

### Key Characteristics

✅ **Depends only on Presentation** - No business logic
✅ **Flutter widgets only** - StatefulWidget, StatelessWidget
✅ **Passive** - Just displays data from presenter
✅ **Reactive** - Rebuilds when streams emit new data

---

### Main Page

#### NextEventPage

**File**: [lib/ui/pages/next_event_page.dart](fluttter_avancado_tdd_clean_arch/lib/ui/pages/next_event_page.dart)

```dart
final class NextEventPage extends StatefulWidget {
  final NextEventPresenter presenter;
  final String groupId;

  const NextEventPage({
    super.key,
    required this.presenter,
    required this.groupId,
  });

  @override
  State<NextEventPage> createState() => _NextEventPageState();
}

class _NextEventPageState extends State<NextEventPage> {
  late final StreamSubscription<bool> _isBusySubscription;

  @override
  void initState() {
    super.initState();

    // Load data when page opens
    widget.presenter.loadNextEvent(groupId: widget.groupId);

    // Listen to loading state to show/hide spinner dialog
    _isBusySubscription = widget.presenter.isBusyStream.listen((isBusy) {
      isBusy ? showLoading(context) : Navigator.of(context).maybePop();
    });
  }

  @override
  void dispose() {
    // CRITICAL: Cancel subscription to prevent memory leak
    _isBusySubscription.cancel();

    // CRITICAL: Dispose presenter to close streams
    widget.presenter.dispose();

    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Próximo Jogo'),
        centerTitle: true,
      ),
      body: StreamBuilder<NextEventViewModel>(
        stream: widget.presenter.nextEventStream,
        builder: (context, snapshot) {
          // Show spinner while waiting for first data
          if (snapshot.connectionState != ConnectionState.active) {
            return const Center(child: CircularProgressIndicator());
          }

          // Show error screen if stream emits error
          if (snapshot.hasError) {
            return ErrorLayout(
              message: (snapshot.error as DomainError).description(),
              onRetry: () => widget.presenter.loadNextEvent(
                groupId: widget.groupId,
              ),
            );
          }

          // Display data
          final viewModel = snapshot.data!;
          return RefreshIndicator(
            onRefresh: () async => widget.presenter.loadNextEvent(
              groupId: widget.groupId,
              isReload: true,
            ),
            child: ListView(
              padding: const EdgeInsets.all(20),
              children: [
                if (viewModel.goalkeepers.isNotEmpty)
                  ListSection(
                    title: 'DENTRO - GOLEIROS',
                    items: viewModel.goalkeepers,
                  ),
                if (viewModel.players.isNotEmpty)
                  ListSection(
                    title: 'DENTRO - JOGADORES',
                    items: viewModel.players,
                  ),
                if (viewModel.out.isNotEmpty)
                  ListSection(
                    title: 'FORA',
                    items: viewModel.out,
                  ),
                if (viewModel.doubt.isNotEmpty)
                  ListSection(
                    title: 'DÚVIDA',
                    items: viewModel.doubt,
                  ),
              ],
            ),
          );
        },
      ),
    );
  }
}
```

**Key Points**:

1. **StreamSubscription Management**:
   ```dart
   late final StreamSubscription<bool> _isBusySubscription;

   _isBusySubscription = widget.presenter.isBusyStream.listen(...);

   @override
   void dispose() {
     _isBusySubscription.cancel();  // Prevent memory leak!
     super.dispose();
   }
   ```
   **Why?** StreamSubscriptions don't automatically cancel when widget dies. Must manually cancel!

2. **Presenter Disposal**:
   ```dart
   @override
   void dispose() {
     widget.presenter.dispose();  // Close all streams
     super.dispose();
   }
   ```
   **Why?** Presenter holds stream controllers that must be closed.

3. **ConnectionState Check**:
   ```dart
   if (snapshot.connectionState != ConnectionState.active) {
     return const Center(child: CircularProgressIndicator());
   }
   ```
   **Why?**
   - `waiting`: Stream exists but no data yet
   - `active`: Stream has emitted at least once
   - Once active, stays active (even during reload)

4. **Error Handling**:
   ```dart
   if (snapshot.hasError) {
     return ErrorLayout(
       message: (snapshot.error as DomainError).description(),
       onRetry: () => widget.presenter.loadNextEvent(groupId: widget.groupId),
     );
   }
   ```
   Error is a `DomainError` (sealed class), so we can call `.description()`

5. **Pull-to-Refresh**:
   ```dart
   RefreshIndicator(
     onRefresh: () async => widget.presenter.loadNextEvent(
       groupId: widget.groupId,
       isReload: true,  // Shows loading dialog
     ),
     child: ListView(...),
   )
   ```

6. **Conditional Rendering**:
   ```dart
   if (viewModel.goalkeepers.isNotEmpty)
     ListSection(title: 'GOALKEEPERS', items: viewModel.goalkeepers),
   ```
   Only show sections that have data.

**Testing**: See [test/ui/pages/next_event_page_test.dart](fluttter_avancado_tdd_clean_arch/test/ui/pages/next_event_page_test.dart)

---

### Reusable Components

#### ListSection

**File**: [lib/ui/components/list_section.dart](fluttter_avancado_tdd_clean_arch/lib/ui/components/list_section.dart)

```dart
final class ListSection extends StatelessWidget {
  final String title;
  final List<NextEventPlayerViewModel> items;

  const ListSection({
    super.key,
    required this.title,
    required this.items,
  });

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text(title, style: context.textStyles.titleSmall),
        const SizedBox(height: 10),
        ...items.map((item) => PlayerListItem(item: item)),
        const SizedBox(height: 20),
      ],
    );
  }
}
```

**Responsibility**: Groups players under a title (e.g., "GOALKEEPERS")

---

#### PlayerPhoto

**File**: [lib/ui/components/player_photo.dart](fluttter_avancado_tdd_clean_arch/lib/ui/components/player_photo.dart)

```dart
final class PlayerPhoto extends StatelessWidget {
  final String initials;
  final String? photo;

  const PlayerPhoto({
    super.key,
    required this.initials,
    this.photo,
  });

  @override
  Widget build(BuildContext context) {
    return CircleAvatar(
      radius: 25,
      foregroundImage: photo != null ? NetworkImage(photo!) : null,
      child: photo == null ? Text(initials) : null,
    );
  }
}
```

**Logic**:
- If photo URL exists → show image
- If no photo → show initials ("MF")

**Testing**: See [test/ui/components/player_photo_test.dart](fluttter_avancado_tdd_clean_arch/test/ui/components/player_photo_test.dart)

---

#### PlayerPosition

**File**: [lib/ui/components/player_position.dart](fluttter_avancado_tdd_clean_arch/lib/ui/components/player_position.dart)

```dart
final class PlayerPosition extends StatelessWidget {
  final String? position;

  const PlayerPosition({super.key, this.position});

  String buildPosition() => switch (position) {
        'goalkeeper' => 'Goleiro',
        'defender' => 'Zagueiro',
        'midfielder' => 'Meia',
        'forward' => 'Atacante',
        _ => 'Gandula',  // Default for null or unknown
      };

  @override
  Widget build(BuildContext context) {
    return Text(
      buildPosition(),
      style: context.textStyles.bodySmall,
    );
  }
}
```

**Pattern**: Dart 3 switch expressions (concise, exhaustive)

**Testing**: See [test/ui/components/player_position_test.dart](fluttter_avancado_tdd_clean_arch/test/ui/components/player_position_test.dart)

---

#### PlayerStatus

**File**: [lib/ui/components/player_status.dart](fluttter_avancado_tdd_clean_arch/lib/ui/components/player_status.dart)

```dart
final class PlayerStatus extends StatelessWidget {
  final bool? isConfirmed;

  const PlayerStatus({super.key, this.isConfirmed});

  Color getColor() => switch (isConfirmed) {
        true => Colors.teal,        // Confirmed → green
        false => Colors.pink,       // Cannot attend → red
        null => Colors.blueGrey,    // Undecided → gray
      };

  @override
  Widget build(BuildContext context) {
    return Container(
      height: 12,
      width: 12,
      decoration: BoxDecoration(
        shape: BoxShape.circle,
        color: getColor(),
      ),
    );
  }
}
```

**Visual indicator**: Small colored circle showing player status

**Testing**: See [test/ui/components/player_status_test.dart](fluttter_avancado_tdd_clean_arch/test/ui/components/player_status_test.dart)

---

### UI Layer Summary

**Structure**:
```
ui/
├── pages/
│   └── next_event_page.dart       # Main screen (StatefulWidget)
└── components/
    ├── list_section.dart          # Groups players
    ├── player_list_item.dart      # Individual player row
    ├── player_photo.dart          # Avatar with initials fallback
    ├── player_position.dart       # Position label
    └── player_status.dart         # Status indicator circle
```

**Key Patterns**:
- MVP View (passive, just displays)
- StreamBuilder (reactive UI)
- Component composition (small, reusable widgets)
- Dart 3 pattern matching (switch expressions)

**Responsibilities**:
- ✅ Display data from ViewModels
- ✅ Listen to presenter streams
- ✅ Handle user interactions (tap, refresh)
- ✅ Manage stream subscriptions
- ✅ Dispose resources properly

**Critical Rules**:
- ⚠️ Always cancel StreamSubscriptions in `dispose()`
- ⚠️ Always dispose presenters in `dispose()`
- ⚠️ Never put business logic in widgets

---

## Composition Root

**Location**: `lib/main/`

**Analogy**: Think of this as the **assembly line** where all parts come together to build the final product.

### Purpose

The Composition Root:
- Creates all objects
- Wires dependencies
- Assembles the application
- Entry point for the app

### Key Characteristics

✅ **Manual dependency injection** - No frameworks (no get_it, no provider)
✅ **Factory pattern** - Functions that create and wire objects
✅ **Only place where layers meet** - All other code follows dependency rule

---

### Main Entry Point

**File**: [lib/main/main.dart](fluttter_avancado_tdd_clean_arch/lib/main/main.dart)

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: makeNextEventPage(),  // Factory creates page with all dependencies
    );
  }
}
```

---

### Factory Functions

Factories follow a pattern: `make[ClassName]()`

#### Page Factory

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

**What it does**: Creates page → injects presenter → presenter gets repository function

---

#### Repository Factory

**File**: [lib/main/factories/infra/repositories/load_next_event_from_api_with_cache_fallback_repo_factory.dart](fluttter_avancado_tdd_clean_arch/lib/main/factories/infra/repositories/load_next_event_from_api_with_cache_fallback_repo_factory.dart)

```dart
LoadNextEventFromApiWithCacheFallbackRepository
    makeLoadNextEventFromApiWithCacheFallbackRepository() {
  return LoadNextEventFromApiWithCacheFallbackRepository(
    cacheClient: makeCacheManagerAdapter(),
    key: 'next_event',
    loadNextEventFromApi: makeLoadNextEventApiRepository().loadNextEvent,
    loadNextEventFromCache: makeLoadNextEventCacheRepository().loadNextEvent,
    mappper: makeNextEventMapper(),
  );
}
```

**Dependencies**:
- Cache adapter
- API repository
- Cache repository
- Mapper

---

#### API Repository Factory

**File**: [lib/main/factories/infra/api/repositories/load_next_event_api_repo_factory.dart](fluttter_avancado_tdd_clean_arch/lib/main/factories/infra/api/repositories/load_next_event_api_repo_factory.dart)

```dart
LoadNextEventApiRepository makeLoadNextEventApiRepository() {
  return LoadNextEventApiRepository(
    httpClient: makeHttpAdapter(),
    url: 'http://localhost:8080/api/groups/:groupId/next_event',
    mapper: makeNextEventMapper(),
  );
}
```

---

#### Adapter Factories

**File**: [lib/main/factories/infra/api/adapters/http_adapter_factory.dart](fluttter_avancado_tdd_clean_arch/lib/main/factories/infra/api/adapters/http_adapter_factory.dart)

```dart
HttpAdapter makeHttpAdapter() {
  return HttpAdapter(client: Client());
}
```

**File**: [lib/main/factories/infra/cache/adapters/cache_manager_adapter_factory.dart](fluttter_avancado_tdd_clean_arch/lib/main/factories/infra/cache/adapters/cache_manager_adapter_factory.dart)

```dart
CacheManagerAdapter makeCacheManagerAdapter() {
  return CacheManagerAdapter(client: DefaultCacheManager());
}
```

---

#### Mapper Factories

**File**: [lib/main/factories/infra/mappers/next_event_player_mapper_factory.dart](fluttter_avancado_tdd_clean_arch/lib/main/factories/infra/mappers/next_event_player_mapper_factory.dart)

```dart
NextEventPlayerMapper makeNextEventPlayerMapper() {
  return NextEventPlayerMapper();
}
```

**File**: [lib/main/factories/infra/mappers/next_event_mapper_factory.dart](fluttter_avancado_tdd_clean_arch/lib/main/factories/infra/mappers/next_event_mapper_factory.dart)

```dart
NextEventMapper makeNextEventMapper() {
  return NextEventMapper(
    playerMapper: makeNextEventPlayerMapper(),
  );
}
```

---

### Complete Dependency Graph

```
makeNextEventPage()
  ├─> makeLoadNextEventFromApiWithCacheFallbackRepository()
  │     ├─> makeLoadNextEventApiRepository()
  │     │     ├─> makeHttpAdapter()
  │     │     │     └─> Client() [from http package]
  │     │     └─> makeNextEventMapper()
  │     │           └─> makeNextEventPlayerMapper()
  │     ├─> makeLoadNextEventCacheRepository()
  │     │     ├─> makeCacheManagerAdapter()
  │     │     │     └─> DefaultCacheManager() [from flutter_cache_manager]
  │     │     └─> makeNextEventMapper()
  │     │           └─> makeNextEventPlayerMapper()
  │     ├─> makeCacheManagerAdapter()
  │     └─> makeNextEventMapper()
  └─> NextEventRxPresenter(nextEventLoader: repo.loadNextEvent)
```

---

### Why Manual Dependency Injection?

**Advantages**:
- ✅ Full control over object creation
- ✅ Easy to debug (no magic)
- ✅ Compile-time safety
- ✅ No runtime reflection
- ✅ Simple to understand

**Disadvantages**:
- ❌ More boilerplate
- ❌ Manual wiring

**Alternative**: Could use `get_it`, `provider`, or `riverpod` for DI

---

### Composition Root Summary

**Structure**:
```
main/
├── main.dart                      # App entry point
└── factories/
    ├── infra/
    │   ├── api/
    │   │   ├── adapters/
    │   │   │   └── http_adapter_factory.dart
    │   │   └── repositories/
    │   │       ├── load_next_event_api_repo_factory.dart
    │   │       └── load_next_event_cache_repo_factory.dart
    │   ├── cache/
    │   │   └── adapters/
    │   │       └── cache_manager_adapter_factory.dart
    │   ├── mappers/
    │   │   ├── next_event_mapper_factory.dart
    │   │   └── next_event_player_mapper_factory.dart
    │   └── repositories/
    │       └── load_next_event_from_api_with_cache_fallback_repo_factory.dart
    └── ui/
        └── pages/
            └── next_event_page_factory.dart
```

**Pattern**: Factory Method Pattern

**Responsibility**: Create and wire all objects

---

## Summary

### Layer Comparison Table

| Layer | Location | Depends On | Responsibility | Examples |
|-------|----------|------------|----------------|----------|
| **Domain** | `lib/domain/` | Nothing | Business entities & rules | NextEvent, NextEventPlayer |
| **Infrastructure** | `lib/infra/` | Domain | Data fetching & conversion | HttpAdapter, Repositories, Mappers |
| **Presentation** | `lib/presentation/` | Domain | UI state & ViewModels | NextEventRxPresenter, ViewModels |
| **UI** | `lib/ui/` | Presentation | Widgets & user interaction | NextEventPage, Components |
| **Main** | `lib/main/` | All layers | Dependency injection | Factories |

---

### Key Takeaways

✅ **Domain**: Pure business logic, no dependencies
✅ **Infrastructure**: Handles external world (APIs, databases)
✅ **Presentation**: Prepares data for UI, manages state
✅ **UI**: Displays data, captures input
✅ **Main**: Assembles everything together

✅ **Each layer is independently testable**
✅ **Dependencies flow inward (toward domain)**
✅ **Clear separation of concerns**

---

## Next Steps

Continue learning:
- **[Design Patterns](03-DESIGN-PATTERNS.md)**: Deep dive into patterns used
- **[Component Deep-Dive](04-COMPONENT-DEEP-DIVE.md)**: See how everything connects
- **[Getting Started Guide](05-GETTING-STARTED-GUIDE.md)**: Start coding
