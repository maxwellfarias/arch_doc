# Component Deep-Dive - Data Flow & Interactions

## Table of Contents
- [Introduction](#introduction)
- [Complete Data Flow](#complete-data-flow)
- [Component Interactions](#component-interactions)
- [Key Implementation Details](#key-implementation-details)
- [Error Handling Flow](#error-handling-flow)
- [Offline Support Strategy](#offline-support-strategy)
- [Testing Strategy](#testing-strategy)

---

## Introduction

This document provides a deep dive into how components interact, how data flows through the application, and the implementation details that make everything work together.

### What You'll Learn

- 🔄 Complete request-response lifecycle
- 🧩 How components communicate
- 📊 Data transformation at each layer
- ❌ Error handling and recovery
- 📡 Offline support with caching
- 🧪 Testing approaches

---

## Complete Data Flow

Let's trace a complete user interaction from tap to screen update.

### Scenario: User Opens the App

```
┌─────────────────────────────────────────────────────────────────┐
│                      1. APPLICATION START                       │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ↓
              ┌────────────────────────────────┐
              │  main() → runApp(MyApp())      │
              └────────────────────────────────┘
                               │
                               ↓
              ┌────────────────────────────────┐
              │  makeNextEventPage()           │
              │  - Creates dependencies        │
              │  - Wires everything together   │
              └────────────────────────────────┘
                               │
                               ↓
┌─────────────────────────────────────────────────────────────────┐
│                    2. DEPENDENCY CONSTRUCTION                   │
└─────────────────────────────────────────────────────────────────┘
                               │
        ┌──────────────────────┴───────────────────────┐
        ↓                                              ↓
┌──────────────────┐                       ┌──────────────────────┐
│ HTTP Components  │                       │  Cache Components    │
│ ─────────────    │                       │  ────────────────    │
│ • HttpAdapter    │                       │ • CacheAdapter       │
│ • ApiRepository  │                       │ • CacheRepository    │
│ • Mapper         │                       │ • Mapper             │
└──────────────────┘                       └──────────────────────┘
        │                                              │
        └──────────────────────┬───────────────────────┘
                               ↓
              ┌────────────────────────────────┐
              │  Composite Repository          │
              │  (API + Cache Fallback)        │
              └────────────────────────────────┘
                               │
                               ↓
              ┌────────────────────────────────┐
              │  NextEventRxPresenter          │
              │  (receives repo function)      │
              └────────────────────────────────┘
                               │
                               ↓
              ┌────────────────────────────────┐
              │  NextEventPage                 │
              │  (receives presenter)          │
              └────────────────────────────────┘
                               │
                               ↓
┌─────────────────────────────────────────────────────────────────┐
│                      3. PAGE INITIALIZATION                     │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  NextEventPage.initState()                 │
      │  ─────────────────────────                 │
      │  1. Subscribe to isBusyStream              │
      │  2. Call presenter.loadNextEvent()         │
      └────────────────────────────────────────────┘
                               │
                               ↓
┌─────────────────────────────────────────────────────────────────┐
│                    4. DATA LOADING BEGINS                       │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Presenter.loadNextEvent()                 │
      │  ─────────────────────────                 │
      │  Calls: nextEventLoader(groupId: 'abc')    │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Composite Repository                      │
      │  ─────────────────────                     │
      │  Try: Load from API                        │
      └────────────────────────────────────────────┘
                               │
                               ↓
┌─────────────────────────────────────────────────────────────────┐
│                       5. HTTP REQUEST                           │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  ApiRepository.loadNextEvent()             │
      │  ─────────────────────────────             │
      │  Calls: httpClient.get(url, params)        │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  HttpAdapter.get()                         │
      │  ──────────────────                        │
      │  1. Build URI (:groupId → 'abc')           │
      │  2. Add headers (JSON)                     │
      │  3. Execute: client.get(uri)               │
      │  4. Handle response                        │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Backend Server                            │
      │  ──────────────                            │
      │  Returns JSON:                             │
      │  {                                         │
      │    "groupName": "Pelada Chega+",           │
      │    "date": "2024-01-11...",                │
      │    "players": [...]                        │
      │  }                                         │
      └────────────────────────────────────────────┘
                               │
                               ↓
┌─────────────────────────────────────────────────────────────────┐
│                     6. DATA TRANSFORMATION                      │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  HttpAdapter._handleResponse()             │
      │  ──────────────────────────────            │
      │  Status 200 → jsonDecode(body)             │
      │  Returns: Map<String, dynamic>             │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  ApiRepository receives JSON               │
      │  ────────────────────────────              │
      │  Calls: mapper.toDto(json)                 │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  NextEventMapper.toDto()                   │
      │  ────────────────────────                  │
      │  1. Extract groupName, date                │
      │  2. Call playerMapper.toDtoList(players)   │
      │  3. Create NextEvent entity                │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  NextEventPlayerMapper.toDto()             │
      │  (for each player)                         │
      │  ──────────────────────────────            │
      │  1. Extract id, name, position, etc.       │
      │  2. Call NextEventPlayer factory           │
      │  3. Factory auto-generates initials        │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Domain Entity Created!                    │
      │  ───────────────────────                   │
      │  NextEvent(                                │
      │    groupName: "Pelada Chega+",             │
      │    date: DateTime(2024, 1, 11),            │
      │    players: [                              │
      │      NextEventPlayer(                      │
      │        name: "Cristiano Ronaldo",          │
      │        initials: "CR", // auto-generated   │
      │        ...                                 │
      │      ),                                    │
      │      ...                                   │
      │    ]                                       │
      │  )                                         │
      └────────────────────────────────────────────┘
                               │
                               ↓
┌─────────────────────────────────────────────────────────────────┐
│                       7. CACHE SAVE                             │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Composite Repository                      │
      │  ─────────────────────                     │
      │  API succeeded! Save to cache:             │
      │  cacheClient.save(                         │
      │    key: 'next_event:abc',                  │
      │    value: mapper.toJson(event)             │
      │  )                                         │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  CacheManagerAdapter.save()                │
      │  ───────────────────────────               │
      │  1. Convert entity → JSON                  │
      │  2. Encode to UTF-8                        │
      │  3. Save as file: 'next_event_abc.json'    │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Return NextEvent to Presenter             │
      └────────────────────────────────────────────┘
                               │
                               ↓
┌─────────────────────────────────────────────────────────────────┐
│                   8. PRESENTER PROCESSING                       │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Presenter._mapEvent(event)                │
      │  ───────────────────────────               │
      │  Transform domain entity → ViewModel       │
      │                                            │
      │  1. Filter players by status:              │
      │     - Confirmed goalkeepers                │
      │     - Confirmed field players              │
      │     - Cannot attend (out)                  │
      │     - Undecided (doubt)                    │
      │                                            │
      │  2. Sort each category:                    │
      │     - Confirmed: by confirmation date      │
      │     - Doubt: alphabetically by name        │
      │                                            │
      │  3. Map to PlayerViewModel (trim fields)   │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  NextEventViewModel Created!               │
      │  ────────────────────────────              │
      │  NextEventViewModel(                       │
      │    goalkeepers: [player1, player2],        │
      │    players: [player3, player4, ...],       │
      │    out: [player8],                         │
      │    doubt: [player9, player10]              │
      │  )                                         │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Emit to Stream                            │
      │  ──────────────                            │
      │  nextEventSubject.add(viewModel)           │
      └────────────────────────────────────────────┘
                               │
                               ↓
┌─────────────────────────────────────────────────────────────────┐
│                       9. UI UPDATE                              │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  StreamBuilder Receives Data               │
      │  ────────────────────────────              │
      │  snapshot.connectionState = active         │
      │  snapshot.hasData = true                   │
      │  snapshot.data = viewModel                 │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Widget Tree Built                         │
      │  ─────────────────                         │
      │  ListView(                                 │
      │    ListSection(                            │
      │      title: 'DENTRO - GOLEIROS',           │
      │      items: viewModel.goalkeepers          │
      │    ),                                      │
      │    ListSection(                            │
      │      title: 'DENTRO - JOGADORES',          │
      │      items: viewModel.players              │
      │    ),                                      │
      │    ...                                     │
      │  )                                         │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Each PlayerListItem Renders               │
      │  ────────────────────────────              │
      │  • PlayerPhoto (with initials/image)       │
      │  • Player name                             │
      │  • PlayerPosition (translated)             │
      │  • PlayerStatus (colored circle)           │
      └────────────────────────────────────────────┘
                               │
                               ↓
┌─────────────────────────────────────────────────────────────────┐
│                    10. SCREEN DISPLAYED! 🎉                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Component Interactions

Let's examine key interactions in detail.

### Interaction 1: Factory Creates Dependencies

**File**: [lib/main/factories/ui/pages/next_event_page_factory.dart](fluttter_avancado_tdd_clean_arch/lib/main/factories/ui/pages/next_event_page_factory.dart)

```dart
Widget makeNextEventPage() {
  // Step 1: Create composite repository
  final repo = makeLoadNextEventFromApiWithCacheFallbackRepository();

  // Step 2: Create presenter with repository function
  final presenter = NextEventRxPresenter(
    nextEventLoader: repo.loadNextEvent,  // Function injection!
  );

  // Step 3: Create page with presenter
  return NextEventPage(
    presenter: presenter,
    groupId: 'valid_id',
  );
}
```

**Expanded factory chain**:

```dart
LoadNextEventFromApiWithCacheFallbackRepository
    makeLoadNextEventFromApiWithCacheFallbackRepository() {
  return LoadNextEventFromApiWithCacheFallbackRepository(
    // API repository
    loadNextEventFromApi: makeLoadNextEventApiRepository().loadNextEvent,

    // Cache repository
    loadNextEventFromCache: makeLoadNextEventCacheRepository().loadNextEvent,

    // Cache client for saving
    cacheClient: makeCacheManagerAdapter(),

    // Mapper for cache conversion
    mappper: makeNextEventMapper(),

    key: 'next_event',
  );
}

LoadNextEventApiRepository makeLoadNextEventApiRepository() {
  return LoadNextEventApiRepository(
    httpClient: makeHttpAdapter(),
    url: 'http://localhost:8080/api/groups/:groupId/next_event',
    mapper: makeNextEventMapper(),
  );
}

HttpAdapter makeHttpAdapter() {
  return HttpAdapter(client: Client());  // http package
}

NextEventMapper makeNextEventMapper() {
  return NextEventMapper(
    playerMapper: makeNextEventPlayerMapper(),
  );
}
```

**Result**: Fully wired object graph ready to use!

---

### Interaction 2: URL Building with Parameters

**File**: [lib/infra/api/adapters/http_adapter.dart](fluttter_avancado_tdd_clean_arch/lib/infra/api/adapters/http_adapter.dart:23-40)

```dart
Uri _buildUri({required String url, Json? params, Json? queryString}) {
  // Step 1: Replace path parameters
  // Input: "http://localhost:8080/api/groups/:groupId/next_event"
  // Params: {"groupId": "abc123"}

  url = params?.keys.fold(
    url,
    (result, key) =>
        result.replaceFirst(':$key', params[key]?.toString() ?? ''),
  ).removeSuffix('/') ?? url;

  // After step 1: "http://localhost:8080/api/groups/abc123/next_event"

  // Step 2: Add query string
  // QueryString: {"page": "1", "limit": "10"}

  url = queryString?.keys
      .fold(
        '$url?',
        (result, key) => '$result$key=${queryString[key]}&',
      )
      .removeSuffix('&') ?? url;

  // After step 2: "http://localhost:8080/api/groups/abc123/next_event?page=1&limit=10"

  return Uri.parse(url);
}
```

**Example walkthrough**:

```dart
// Input
url = "http://api.com/groups/:groupId/events/:eventId"
params = {"groupId": "123", "eventId": "456"}
queryString = {"sort": "date", "order": "asc"}

// Step 1: Replace :groupId
url = "http://api.com/groups/123/events/:eventId"

// Step 1: Replace :eventId
url = "http://api.com/groups/123/events/456"

// Step 2: Add ?sort=date
url = "http://api.com/groups/123/events/456?sort=date&"

// Step 2: Add order=asc
url = "http://api.com/groups/123/events/456?sort=date&order=asc&"

// Step 2: Remove trailing &
url = "http://api.com/groups/123/events/456?sort=date&order=asc"

// Final URI
Uri.parse("http://api.com/groups/123/events/456?sort=date&order=asc")
```

**Benefits**:
- ✅ Clean API in repository: `get(url: url, params: {...})`
- ✅ Flexible for any URL pattern
- ✅ Handles edge cases (no params, no query string)

---

### Interaction 3: JSON to Entity Transformation

**Step-by-step transformation**:

#### Step 1: HTTP Response (Raw JSON)

```json
{
  "id": "1",
  "groupName": "Pelada Chega+",
  "date": "2024-01-11T11:10:00.000Z",
  "players": [
    {
      "id": "1",
      "name": "Cristiano Ronaldo",
      "isConfirmed": true,
      "position": "forward",
      "photo": "https://example.com/photo.jpg",
      "confirmationDate": "2024-01-10T10:00:00.000Z"
    },
    {
      "id": "2",
      "name": "Iker Casillas",
      "isConfirmed": true,
      "position": "goalkeeper",
      "photo": null,
      "confirmationDate": "2024-01-09T15:30:00.000Z"
    },
    {
      "id": "3",
      "name": "Maxwell Farias",
      "isConfirmed": null,
      "position": "midfielder",
      "photo": null,
      "confirmationDate": null
    }
  ]
}
```

#### Step 2: HttpAdapter Decodes JSON

**File**: [lib/infra/api/adapters/http_adapter.dart](fluttter_avancado_tdd_clean_arch/lib/infra/api/adapters/http_adapter.dart:66-71)

```dart
T _handleResponse<T>(Response response) {
  return switch (response.statusCode) {
    200 => jsonDecode(response.body),  // Returns Map<String, dynamic>
    401 => throw SessionExpiredError(),
    _ => throw UnexpectedError(),
  };
}
```

**Result**: `Map<String, dynamic>` (JSON as Dart map)

#### Step 3: NextEventMapper Converts to Entity

**File**: [lib/infra/mappers/next_event_mapper.dart](fluttter_avancado_tdd_clean_arch/lib/infra/mappers/next_event_mapper.dart:8-14)

```dart
@override
NextEvent toDto(dynamic json) => NextEvent(
  groupName: json['groupName'],  // "Pelada Chega+"
  date: DateTime.parse(json['date']),  // DateTime(2024, 1, 11, 11, 10)
  players: playerMapper.toDtoList(json['players']),  // Delegate to player mapper
);
```

#### Step 4: NextEventPlayerMapper Converts Each Player

**File**: [lib/infra/mappers/next_event_player_mapper.dart](fluttter_avancado_tdd_clean_arch/lib/infra/mappers/next_event_player_mapper.dart:6-16)

```dart
@override
NextEventPlayer toDto(dynamic json) => NextEventPlayer(
  id: json['id'],  // "1"
  name: json['name'],  // "Cristiano Ronaldo"
  isConfirmed: json['isConfirmed'],  // true
  position: json['position'],  // "forward"
  photo: json['photo'],  // "https://..."
  confirmationDate: DateTime.tryParse(json['confirmationDate'] ?? ''),
);
```

#### Step 5: Entity Factory Auto-Generates Initials

**File**: [lib/domain/entities/next_event_player.dart](fluttter_avancado_tdd_clean_arch/lib/domain/entities/next_event_player.dart:21-33)

```dart
factory NextEventPlayer({
  required String name,  // "Cristiano Ronaldo"
  ...
}) => NextEventPlayer._(
      name: name,
      initials: _getInitials(name),  // 🎯 "CR"
      ...
    );

static String _getInitials(String name) {
  final names = name.toUpperCase().trim().split(' ');
  // names = ["CRISTIANO", "RONALDO"]

  final firstChar = names.first.split('').firstOrNull ?? '-';
  // firstChar = "C"

  final lastChar = names.last.split('').elementAtOrNull(
    names.length == 1 ? 1 : 0,
  ) ?? '';
  // lastChar = "R"

  return '$firstChar$lastChar';  // "CR"
}
```

#### Final Result: Domain Entity

```dart
NextEvent(
  groupName: "Pelada Chega+",
  date: DateTime(2024, 1, 11, 11, 10, 0),
  players: [
    NextEventPlayer(
      id: "1",
      name: "Cristiano Ronaldo",
      initials: "CR",  // ✨ Auto-generated!
      isConfirmed: true,
      position: "forward",
      photo: "https://example.com/photo.jpg",
      confirmationDate: DateTime(2024, 1, 10, 10, 0, 0),
    ),
    NextEventPlayer(
      id: "2",
      name: "Iker Casillas",
      initials: "IC",  // ✨ Auto-generated!
      isConfirmed: true,
      position: "goalkeeper",
      photo: null,
      confirmationDate: DateTime(2024, 1, 9, 15, 30, 0),
    ),
    NextEventPlayer(
      id: "3",
      name: "Maxwell Farias",
      initials: "MF",  // ✨ Auto-generated!
      isConfirmed: null,
      position: "midfielder",
      photo: null,
      confirmationDate: null,
    ),
  ],
)
```

---

### Interaction 4: Entity to ViewModel Transformation

**File**: [lib/presentation/rx/next_event_rx_presenter.dart](fluttter_avancado_tdd_clean_arch/lib/presentation/rx/next_event_rx_presenter.dart:34-73)

```dart
NextEventViewModel _mapEvent(NextEvent event) {
  // Input: Domain entity with flat list of players
  // Output: ViewModel with categorized, sorted lists

  return NextEventViewModel(
    // Category 1: DOUBT (haven't decided)
    // Filter: No confirmation date
    // Sort: Alphabetically by name
    doubt: event.players
        .where((player) => player.confirmationDate == null)
        .sortedBy((player) => player.name)
        .map(_mapPlayer)
        .toList(),

    // Category 2: OUT (cannot attend)
    // Filter: Confirmed = false
    // Sort: By confirmation date (who declined first)
    out: event.players
        .where((player) =>
            player.confirmationDate != null &&
            player.isConfirmed == false)
        .sortedBy((player) => player.confirmationDate!)
        .map(_mapPlayer)
        .toList(),

    // Category 3: GOALKEEPERS (confirmed)
    // Filter: Confirmed = true AND position = goalkeeper
    // Sort: By confirmation date (who confirmed first)
    goalkeepers: event.players
        .where((player) =>
            player.confirmationDate != null &&
            player.isConfirmed == true &&
            player.position == 'goalkeeper')
        .sortedBy((player) => player.confirmationDate!)
        .map(_mapPlayer)
        .toList(),

    // Category 4: FIELD PLAYERS (confirmed)
    // Filter: Confirmed = true AND position ≠ goalkeeper
    // Sort: By confirmation date (who confirmed first)
    players: event.players
        .where((player) =>
            player.confirmationDate != null &&
            player.isConfirmed == true &&
            player.position != 'goalkeeper')
        .sortedBy((player) => player.confirmationDate!)
        .map(_mapPlayer)
        .toList(),
  );
}

NextEventPlayerViewModel _mapPlayer(NextEventPlayer player) =>
    NextEventPlayerViewModel(
      name: player.name,
      initials: player.initials,
      photo: player.photo,
      position: player.position,
      isConfirmed: player.isConfirmed,
      // Note: id and confirmationDate not included (UI doesn't need them)
    );
```

**Example transformation**:

```dart
// Input: Flat list
[
  Player("Casillas", confirmed: true, position: "goalkeeper", date: Jan 9),
  Player("Ronaldo", confirmed: true, position: "forward", date: Jan 10),
  Player("Maxwell", confirmed: null, position: "midfielder", date: null),
  Player("Messi", confirmed: false, position: "forward", date: Jan 8),
]

// Output: Categorized, sorted ViewModel
NextEventViewModel(
  goalkeepers: [
    PlayerVM("Casillas"),  // Confirmed goalkeeper, Jan 9
  ],
  players: [
    PlayerVM("Ronaldo"),   // Confirmed forward, Jan 10
  ],
  out: [
    PlayerVM("Messi"),     // Declined, Jan 8
  ],
  doubt: [
    PlayerVM("Maxwell"),   // Undecided (sorted alphabetically)
  ],
)
```

**Why this logic is in Presenter**:

From code comments:
> "The presenter is responsible for manipulating data from the API so that information is displayed appropriately on screen. This is the ideal place for this adaptation. If this manipulation happened in higher layers (UseCase, Repository) it would limit my API to a specific mobile screen."

**Example**: Web version might show all players in a single table, sorted differently. The domain entity supports both—only the presenter changes.

---

### Interaction 5: Stream-Based UI Updates

**Presenter emits data**:

**File**: [lib/presentation/rx/next_event_rx_presenter.dart](fluttter_avancado_tdd_clean_arch/lib/presentation/rx/next_event_rx_presenter.dart:20-30)

```dart
Future<void> loadNextEvent({required String groupId, bool isReload = false}) async {
  try {
    if (isReload) isBusySubject.add(true);  // 📢 Emit: loading started

    final event = await nextEventLoader(groupId: groupId);

    nextEventSubject.add(_mapEvent(event));  // 📢 Emit: data ready
  } catch (error) {
    nextEventSubject.addError(error);  // 📢 Emit: error occurred
  } finally {
    if (isReload) isBusySubject.add(false);  // 📢 Emit: loading finished
  }
}
```

**UI listens to streams**:

**File**: [lib/ui/pages/next_event_page.dart](fluttter_avancado_tdd_clean_arch/lib/ui/pages/next_event_page.dart)

```dart
@override
void initState() {
  super.initState();

  // Load data on page open
  widget.presenter.loadNextEvent(groupId: widget.groupId);

  // Listen to loading state (shows/hides dialog)
  _isBusySubscription = widget.presenter.isBusyStream.listen((isBusy) {
    isBusy ? showLoading(context) : Navigator.of(context).maybePop();
  });
}

@override
Widget build(BuildContext context) {
  return StreamBuilder<NextEventViewModel>(
    stream: widget.presenter.nextEventStream,  // 👂 Listen to data
    builder: (context, snapshot) {
      // State 1: Waiting for first data
      if (snapshot.connectionState != ConnectionState.active) {
        return const Center(child: CircularProgressIndicator());
      }

      // State 2: Error occurred
      if (snapshot.hasError) {
        return ErrorLayout(
          message: (snapshot.error as DomainError).description(),
          onRetry: () => widget.presenter.loadNextEvent(groupId: widget.groupId),
        );
      }

      // State 3: Data ready
      final viewModel = snapshot.data!;
      return ListView(
        children: [
          if (viewModel.goalkeepers.isNotEmpty)
            ListSection(title: 'DENTRO - GOLEIROS', items: viewModel.goalkeepers),
          if (viewModel.players.isNotEmpty)
            ListSection(title: 'DENTRO - JOGADORES', items: viewModel.players),
          // ...
        ],
      );
    },
  );
}
```

**Stream lifecycle**:

```
Timeline:
─────────────────────────────────────────────────────────────────>

T0: Page opens
    └─> StreamBuilder created
        └─> connectionState = waiting (show spinner)

T1: loadNextEvent() called
    └─> Presenter starts loading

T2: Data arrives
    └─> nextEventSubject.add(viewModel)
        └─> StreamBuilder receives data
            └─> connectionState = active
                └─> Build UI with data

T3: User pulls to refresh
    └─> isBusySubject.add(true)
        └─> _isBusySubscription triggered
            └─> Show loading dialog

T4: New data arrives
    └─> nextEventSubject.add(newViewModel)
        └─> StreamBuilder rebuilds with new data
        └─> isBusySubject.add(false)
            └─> Hide loading dialog

T5: Page closes
    └─> dispose() called
        └─> _isBusySubscription.cancel()
        └─> presenter.dispose()
            └─> Streams closed
```

**Critical: Stream Subscription Management**

**File**: [lib/ui/pages/next_event_page.dart](fluttter_avancado_tdd_clean_arch/lib/ui/pages/next_event_page.dart:20-36)

```dart
late final StreamSubscription<bool> _isBusySubscription;

@override
void initState() {
  super.initState();
  _isBusySubscription = widget.presenter.isBusyStream.listen(...);
}

@override
void dispose() {
  // ⚠️ CRITICAL: Cancel subscription to prevent memory leak!
  _isBusySubscription.cancel();

  // ⚠️ CRITICAL: Dispose presenter to close streams!
  widget.presenter.dispose();

  super.dispose();
}
```

**Why StreamSubscription must be cancelled**:

From code comments:
> "The listener remains active because it's managed by StreamSubscription, which doesn't directly depend on the widget lifecycle. To avoid leaks, cancel the listener in the dispose method."

**Memory leak example (if not cancelled)**:
```
Page opens → Subscription created → Listen to stream
Page closes → Widget destroyed → Subscription STILL ACTIVE! 💥
              Memory leak → Stream keeps sending updates to dead widget
```

---

## Key Implementation Details

### Detail 1: Initials Generation Business Logic

**File**: [lib/domain/entities/next_event_player.dart](fluttter_avancado_tdd_clean_arch/lib/domain/entities/next_event_player.dart:44-54)

```dart
static String _getInitials(String name) {
  final names = name.toUpperCase().trim().split(' ');
  final firstChar = names.first.split('').firstOrNull ?? '-';
  final lastChar = names.last.split('').elementAtOrNull(
    names.length == 1 ? 1 : 0,
  ) ?? '';
  return '$firstChar$lastChar';
}
```

**Test cases**:

| Input | Process | Output |
|-------|---------|--------|
| "Maxwell Farias" | ["MAXWELL", "FARIAS"] → "M" + "F" | "MF" |
| "John" | ["JOHN"] → "J" + "O" (2nd char) | "JO" |
| "A B C D" | ["A", "B", "C", "D"] → "A" + "D" (last) | "AD" |
| "  " | [] → "-" + "" | "-" |
| "José" | ["JOSÉ"] → "J" + "O" | "JO" |

**Why in domain?**

✅ **Business rule**: Initials format is a business decision
✅ **Consistency**: Always correct, can't create player with wrong initials
✅ **Reusable**: Any layer can use entity with initials

**Testing**: [test/domain/entities/next_event_player_test.dart](fluttter_avancado_tdd_clean_arch/test/domain/entities/next_event_player_test.dart)

---

### Detail 2: Generic Response Handling

**File**: [lib/infra/api/adapters/http_adapter.dart](fluttter_avancado_tdd_clean_arch/lib/infra/api/adapters/http_adapter.dart:66-71)

```dart
T _handleResponse<T>(Response response) {
  return switch (response.statusCode) {
    200 => jsonDecode(response.body),
    401 => throw SessionExpiredError(),
    _ => throw UnexpectedError(),
  };
}
```

**Why generic `<T>`?**

From code comments:
> "jsonDecode returns dynamic, which can be Map or List. If the generic type isn't specified when calling 'get', Dart will infer it based on the receiver."

**Example**:

```dart
// Repository expects Map
Future<dynamic> get(...) async {
  final response = await client.get(...);
  return _handleResponse(response);  // T inferred as Map from assignment
}

// In repository:
final json = await httpClient.get(...);  // json is dynamic
final event = mapper.toDto(json);  // mapper expects Map - works!

// If response was a List:
final list = await httpClient.get(...);  // Inferred as List
final events = list.map((item) => mapper.toDto(item));  // Works!
```

**Status code handling**:
- **200**: Success → decode and return JSON
- **401**: Unauthorized → throw `SessionExpiredError` (must re-login)
- **Other**: Unexpected → throw `UnexpectedError` (generic error)

---

### Detail 3: ConnectionState Handling

**File**: [lib/ui/pages/next_event_page.dart](fluttter_avancado_tdd_clean_arch/lib/ui/pages/next_event_page.dart:44-47)

```dart
if (snapshot.connectionState != ConnectionState.active) {
  return const Center(child: CircularProgressIndicator());
}
```

**ConnectionState values**:

| State | Meaning | When |
|-------|---------|------|
| `none` | No stream connected | Never (we always have stream) |
| `waiting` | Stream exists, no data yet | First load (before data arrives) |
| `active` | Stream has emitted at least once | After first data arrives |
| `done` | Stream closed | Never (we don't close stream until dispose) |

**Why check for `active`**:

From code comments:
> "Once the stream receives data for the first time, the connection state remains active, so if a new call is made, the CircularProgressIndicator won't naturally be displayed."

**Timeline**:

```
T0: StreamBuilder created
    └─> connectionState = waiting
        └─> Show CircularProgressIndicator

T1: First data arrives
    └─> connectionState = active
        └─> Show data (hide spinner)

T2: User refreshes
    └─> connectionState STILL active! (doesn't go back to waiting)
        └─> Can't use connectionState to show loading
        └─> Solution: Separate isBusyStream for loading dialog
```

---

### Detail 4: Mixin Composition for List Mapping

**File**: [lib/infra/mappers/mapper.dart](fluttter_avancado_tdd_clean_arch/lib/infra/mappers/mapper.dart:16-20)

```dart
mixin DtoListMapper<Dto> implements DtoMapper<Dto> {
  List<Dto> toDtoList(dynamic arr) =>
      arr.map<Dto>((item) => toDto(item)).toList();
}
```

**How it works**:

```dart
// Step 1: Define mixin with generic implementation
mixin DtoListMapper<Dto> implements DtoMapper<Dto> {
  // Uses toDto() from implementing class
  List<Dto> toDtoList(dynamic arr) =>
      arr.map<Dto>((item) => toDto(item)).toList();
}

// Step 2: Extend ListMapper which uses mixin
final class NextEventPlayerMapper extends ListMapper<NextEventPlayer> {
  // Only implement core method
  @override
  NextEventPlayer toDto(dynamic json) => NextEventPlayer(...);

  // Get toDtoList() for free!
}

// Step 3: Use in NextEventMapper
final class NextEventMapper implements Mapper<NextEvent> {
  final DtoListMapper<NextEventPlayer> playerMapper;

  @override
  NextEvent toDto(dynamic json) => NextEvent(
    groupName: json['groupName'],
    date: DateTime.parse(json['date']),
    players: playerMapper.toDtoList(json['players']),  // 🎯 Calls mixin method!
  );
}
```

**Benefits**:
- ✅ DRY: List logic defined once
- ✅ Reusable: Any mapper gets list conversion
- ✅ Type-safe: Generics ensure correct types

---

## Error Handling Flow

### Error Scenario 1: API Fails, Cache Available

```
┌─────────────────────────────────────────────────────────────────┐
│              User Opens App (No Internet)                       │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Presenter.loadNextEvent()                 │
      │  Calls: nextEventLoader(groupId: 'abc')    │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Composite Repository                      │
      │  Try: loadNextEventFromApi()               │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  ApiRepository → HttpAdapter → Client      │
      │  ❌ Network error thrown                   │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Composite Repository                      │
      │  Catch block: loadNextEventFromCache()     │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  CacheRepository → CacheAdapter            │
      │  ✅ Returns cached NextEvent               │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Presenter receives data                   │
      │  Maps to ViewModel                         │
      │  Emits to stream                           │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  UI displays cached data                   │
      │  (User doesn't see error!)                 │
      └────────────────────────────────────────────┘
```

**Code**:

**File**: [lib/infra/repositories/load_next_event_from_api_with_cache_fallback_repo.dart](fluttter_avancado_tdd_clean_arch/lib/infra/repositories/load_next_event_from_api_with_cache_fallback_repo.dart:17-38)

```dart
Future<NextEvent> loadNextEvent({required String groupId}) async {
  try {
    // Try API
    final event = await loadNextEventFromApi(groupId: groupId);
    await cacheClient.save(...);
    return event;
  } on SessionExpiredError {
    // Always rethrow auth errors (user must login)
    rethrow;
  } catch (error) {
    // Any other error → try cache
    return loadNextEventFromCache(groupId: groupId);
  }
}
```

---

### Error Scenario 2: Session Expired

```
┌─────────────────────────────────────────────────────────────────┐
│              User Makes Request (Token Expired)                 │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  HttpAdapter._handleResponse()             │
      │  Status code: 401                          │
      │  throw SessionExpiredError()               │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Composite Repository                      │
      │  Catches SessionExpiredError               │
      │  → rethrow (don't use cache!)              │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Presenter catches error                   │
      │  nextEventSubject.addError(error)          │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  StreamBuilder                             │
      │  snapshot.hasError = true                  │
      │  snapshot.error = SessionExpiredError()    │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  ErrorLayout displays:                     │
      │  "Sua sessão expirou"                      │
      │  [Retry button]                            │
      └────────────────────────────────────────────┘
```

**Why rethrow SessionExpiredError?**

User MUST re-authenticate. Showing cached data would be misleading—user thinks they're logged in when they're not.

---

### Error Scenario 3: No Cache Available

```
┌─────────────────────────────────────────────────────────────────┐
│        User Opens App (First Time, No Internet)                 │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Composite Repository                      │
      │  Try: API fails (no internet)              │
      │  Catch: loadNextEventFromCache()           │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  CacheRepository                           │
      │  cacheClient.get() returns null            │
      │  throw UnexpectedError()                   │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  Presenter catches error                   │
      │  nextEventSubject.addError(error)          │
      └────────────────────────────────────────────┘
                               │
                               ↓
      ┌────────────────────────────────────────────┐
      │  ErrorLayout displays:                     │
      │  "Algo inesperado aconteceu"               │
      │  [Retry button]                            │
      └────────────────────────────────────────────┘
```

**Code**:

**File**: [lib/infra/cache/repositories/load_next_event_cache_repo.dart](fluttter_avancado_tdd_clean_arch/lib/infra/cache/repositories/load_next_event_cache_repo.dart:13-19)

```dart
Future<NextEvent> loadNextEvent({required String groupId}) async {
  final json = await cacheClient.get(key: '$key:$groupId');
  if (json == null) throw UnexpectedError();  // No cache
  return mapper.toDto(json);
}
```

---

## Offline Support Strategy

### Cache Strategy: API-First with Fallback

**Goal**: Always try fresh data, fallback to cache if unavailable

**Implementation**:

```dart
// File: lib/infra/repositories/load_next_event_from_api_with_cache_fallback_repo.dart
Future<NextEvent> loadNextEvent({required String groupId}) async {
  try {
    // Step 1: Try API (fresh data preferred)
    final event = await loadNextEventFromApi(groupId: groupId);

    // Step 2: Save to cache (for next time)
    await cacheClient.save(
      key: '$key:$groupId',
      value: mappper.toJson(event),
    );

    // Step 3: Return fresh data
    return event;
  } on SessionExpiredError {
    // Don't catch auth errors
    rethrow;
  } catch (error) {
    // Step 4: Fallback to cache on any other error
    return loadNextEventFromCache(groupId: groupId);
  }
}
```

### Cache Validation

**File**: [lib/infra/cache/adapters/cache_manager_adapter.dart](fluttter_avancado_tdd_clean_arch/lib/infra/cache/adapters/cache_manager_adapter.dart:11-23)

```dart
@override
Future<dynamic> get({required String key}) async {
  final info = await client.getFileFromCache(key);

  // Check 1: Cache expired?
  if (info?.validTill.isBefore(DateTime.now()) != false) {
    return null;
  }

  // Check 2: File exists?
  if (!await info!.file.exists()) {
    return null;
  }

  // Read and parse
  final data = await info.file.readAsString();
  return jsonDecode(data);
}
```

**Validations**:
1. ✅ Cache has expiration date
2. ✅ File existence verified
3. ✅ Returns null if invalid (triggers API fetch)

### Offline Support Scenarios

| Scenario | API | Cache | Result |
|----------|-----|-------|--------|
| **First load, online** | ✅ Success | ❌ Empty | Fresh data from API, saved to cache |
| **Second load, online** | ✅ Success | ✅ Available | Fresh data from API, cache updated |
| **Load, offline** | ❌ Fails | ✅ Available | Cached data (offline support!) |
| **First load, offline** | ❌ Fails | ❌ Empty | Error shown (retry button) |
| **Load, expired cache** | ❌ Fails | ❌ Expired | Error shown |
| **Session expired** | 401 | ✅ Available | Error shown (cache NOT used) |

---

## Testing Strategy

### Test Pyramid

```
                    ┌─────────┐
                    │   E2E   │  1 test (full integration)
                    └─────────┘
                  ┌─────────────┐
                  │   Widget    │  5 tests (UI components)
                  └─────────────┘
              ┌───────────────────┐
              │  Presenter        │  1 test (business logic)
              └───────────────────┘
          ┌─────────────────────────┐
          │    Repository           │  8 tests (data layer)
          └─────────────────────────┘
      ┌───────────────────────────────┐
      │       Entity                  │  1 test (domain logic)
      └───────────────────────────────┘
```

### Testing Without Mockito: Custom Spies

**Example Spy**:

**File**: [test/infra/mocks/load_next_event_repo_spy.dart](fluttter_avancado_tdd_clean_arch/test/infra/mocks/load_next_event_repo_spy.dart)

```dart
final class LoadNextEventRepositorySpy {
  String? groupId;
  int callsCount = 0;
  NextEvent output = anyNextEvent();
  Object? error;

  Future<NextEvent> loadNextEvent({required String groupId}) async {
    this.groupId = groupId;
    callsCount++;
    if (error != null) throw error!;
    return output;
  }
}
```

**Usage in test**:

```dart
test('should load event from api repository', () async {
  // Arrange
  final apiRepo = LoadNextEventRepositorySpy();
  final sut = LoadNextEventFromApiWithCacheFallbackRepository(
    loadNextEventFromApi: apiRepo.loadNextEvent,
    ...
  );

  // Act
  await sut.loadNextEvent(groupId: 'abc');

  // Assert
  expect(apiRepo.groupId, 'abc');
  expect(apiRepo.callsCount, 1);
});
```

**Benefits**:
- ✅ Full control over behavior
- ✅ Easy to understand (no magic)
- ✅ Inspect calls: `groupId`, `callsCount`
- ✅ Simulate errors: `error = Error()`

---

## Summary

### Data Flow Summary

1. **Factory** creates all dependencies
2. **Page** initializes, calls presenter
3. **Presenter** requests data via injected function
4. **Composite Repository** tries API, falls back to cache
5. **HTTP Adapter** builds URL, executes request
6. **Mapper** converts JSON → Domain entity
7. **Entity Factory** auto-generates business fields (initials)
8. **Cache** saves data for offline use
9. **Presenter** filters/sorts into ViewModel
10. **Stream** emits ViewModel to UI
11. **StreamBuilder** rebuilds with data
12. **Components** render individual pieces

### Key Patterns in Action

- ✅ **Clean Architecture**: Each layer has clear responsibility
- ✅ **MVP**: Presenter mediates between domain and UI
- ✅ **Repository**: Abstracts data sources
- ✅ **Adapter**: Isolates external dependencies
- ✅ **Factory**: Wires dependencies
- ✅ **Observer**: Streams for reactive UI
- ✅ **Strategy**: API-first with cache fallback

### Critical Implementation Points

1. **Always cancel StreamSubscriptions** in dispose()
2. **Always dispose presenters** to close streams
3. **ConnectionState.active** persists after first data
4. **SessionExpiredError** always propagates (never use cache)
5. **Initials generation** is business logic (in domain)
6. **UI-specific filtering/sorting** is in presenter
7. **Mappers are injected** for reusability

---

## Next Steps

- **[Getting Started Guide](05-GETTING-STARTED-GUIDE.md)**: Start coding with this architecture
- **Practice**: Try adding a new feature following these patterns
- **Experiment**: Modify data flow to understand interactions
