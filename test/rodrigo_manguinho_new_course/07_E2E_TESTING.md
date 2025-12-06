# End-to-End Testing

## Introduction

**End-to-End (E2E) tests** verify complete user flows from the UI through all layers to the data source. They:
- Use **real implementations** (not mocks) for internal components
- Only mock **external boundaries** (HTTP, file system)
- Test the **complete integration** of all layers

E2E tests give the highest confidence but are slower and more complex.

---

## Table of Contents

1. [E2E Testing Strategy](#e2e-testing-strategy)
2. [Setting Up E2E Tests](#setting-up-e2e-tests)
3. [Testing API Success Flow](#testing-api-success-flow)
4. [Testing Cache Fallback](#testing-cache-fallback)
5. [Testing Error States](#testing-error-states)
6. [Using ensureVisible](#using-ensurevisible)

---

## E2E Testing Strategy

### What to Mock vs. Use Real

| Component | Use |
|-----------|-----|
| UI (Pages, Widgets) | Real |
| Presenter | Real |
| Repositories | Real |
| Mappers | Real |
| HTTP Client | **Mock** (external boundary) |
| Cache/File System | **Mock** (external boundary) |

### Why This Approach?

- **Real components**: Verify actual integration
- **Mock boundaries**: Control external responses
- **Deterministic**: Tests are repeatable
- **Fast**: No real network/disk I/O

---

## Setting Up E2E Tests

### Complete Test Setup

```dart
import 'package:advanced_flutter/infra/api/adapters/http_adapter.dart';
import 'package:advanced_flutter/infra/api/repositories/load_next_event_api_repo.dart';
import 'package:advanced_flutter/infra/cache/adapters/cache_manager_adapter.dart';
import 'package:advanced_flutter/infra/cache/repositories/load_next_event_cache_repo.dart';
import 'package:advanced_flutter/infra/mappers/next_event_mapper.dart';
import 'package:advanced_flutter/infra/respositories/load_next_event_from_api_with_cache_fallback_repo.dart';
import 'package:advanced_flutter/main/factories/infra/mappers/next_event_mapper_factory.dart';
import 'package:advanced_flutter/presentation/rx/next_event_rx_presenter.dart';
import 'package:advanced_flutter/ui/pages/next_event_page.dart';

import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

import '../infra/api/mocks/client_spy.dart';
import '../infra/cache/mocks/cache_manager_spy.dart';
import '../mocks/fakes.dart';

void main() {
  late String responseJson;
  late String key;
  late NextEventMapper mapper;
  late ClientSpy client;
  late CacheManagerSpy cacheManager;
  late HttpAdapter httpClient;
  late CacheManagerAdapter cacheClient;
  late LoadNextEventApiRepository apiRepo;
  late LoadNextEventCacheRepository cacheRepo;
  late LoadNextEventFromApiWithCacheFallbackRepository repo;
  late NextEventRxPresenter presenter;
  late MaterialApp sut;

  // Define test data once
  setUpAll(() {
    responseJson = '''
      {
        "id": "1",
        "groupName": "Pelada Chega+",
        "date": "2024-01-11T11:10:00.000Z",
        "players": [{
          "id": "1",
          "name": "Cristiano Ronaldo",
          "position": "forward",
          "isConfirmed": true,
          "confirmationDate": "2024-01-10T11:07:00.000Z"
        }, {
          "id": "2",
          "name": "Lionel Messi",
          "position": "midfielder",
          "isConfirmed": true,
          "confirmationDate": "2024-01-10T11:08:00.000Z"
        }, {
          "id": "3",
          "name": "Dida",
          "position": "goalkeeper",
          "isConfirmed": true,
          "confirmationDate": "2024-01-10T09:10:00.000Z"
        }, {
          "id": "4",
          "name": "Romario",
          "position": "forward",
          "isConfirmed": true,
          "confirmationDate": "2024-01-10T11:10:00.000Z"
        }, {
          "id": "5",
          "name": "Claudio Gamarra",
          "position": "defender",
          "isConfirmed": false,
          "confirmationDate": "2024-01-10T13:10:00.000Z"
        }, {
          "id": "6",
          "name": "Diego Forlan",
          "position": "defender",
          "isConfirmed": false,
          "confirmationDate": "2024-01-10T14:10:00.000Z"
        }, {
          "id": "7",
          "name": "Zé Ninguém",
          "isConfirmed": false
        }, {
          "id": "8",
          "name": "Rodrigo Manguinho",
          "isConfirmed": false
        }, {
          "id": "9",
          "name": "Claudio Taffarel",
          "position": "goalkeeper",
          "isConfirmed": true,
          "confirmationDate": "2024-01-10T09:15:00.000Z"
        }]
      }
    ''';
  });

  // Wire up real components before each test
  setUp(() {
    key = anyString();

    // Real mapper (from factory)
    mapper = makeNextEventMapper();

    // Mock HTTP client (external boundary)
    client = ClientSpy();

    // Real HTTP adapter with mock client
    httpClient = HttpAdapter(client: client);

    // Real API repository
    apiRepo = LoadNextEventApiRepository(
      httpClient: httpClient,
      url: anyString(),
      mapper: mapper
    );

    // Mock cache manager (external boundary)
    cacheManager = CacheManagerSpy();

    // Real cache adapter with mock manager
    cacheClient = CacheManagerAdapter(client: cacheManager);

    // Real cache repository
    cacheRepo = LoadNextEventCacheRepository(
      cacheClient: cacheClient,
      key: key,
      mapper: mapper
    );

    // Real composite repository
    repo = LoadNextEventFromApiWithCacheFallbackRepository(
      loadNextEventFromApi: apiRepo.loadNextEvent,
      loadNextEventFromCache: cacheRepo.loadNextEvent,
      cacheClient: cacheClient,
      key: key,
      mapper: mapper
    );

    // Real presenter
    presenter = NextEventRxPresenter(nextEventLoader: repo.loadNextEvent);

    // Real page
    sut = MaterialApp(
      home: NextEventPage(presenter: presenter, groupId: anyString())
    );
  });

  // Tests go here
}
```

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    MaterialApp + Page                       │
│                        (REAL)                               │
├─────────────────────────────────────────────────────────────┤
│                     Presenter                               │
│                        (REAL)                               │
├─────────────────────────────────────────────────────────────┤
│              Composite Repository                           │
│                        (REAL)                               │
├─────────────────────────────────────────────────────────────┤
│         API Repository      │      Cache Repository         │
│            (REAL)           │           (REAL)              │
├─────────────────────────────────────────────────────────────┤
│         HTTP Adapter        │      Cache Adapter            │
│            (REAL)           │           (REAL)              │
├─────────────────────────────────────────────────────────────┤
│         ClientSpy           │      CacheManagerSpy          │
│           (MOCK)            │           (MOCK)              │
└─────────────────────────────────────────────────────────────┘
```

---

## Testing API Success Flow

When the API returns data successfully:

```dart
testWidgets('should present api data', (tester) async {
  // Configure mock to return valid JSON
  client.responseJson = responseJson;

  // Build the widget tree
  await tester.pumpWidget(sut);
  await tester.pump();  // Process initial load

  // Scroll to and verify player is visible
  await tester.ensureVisible(
    find.text('Cristiano Ronaldo', skipOffstage: false)
  );
  await tester.pump();
  expect(find.text('Cristiano Ronaldo'), findsOneWidget);

  // Verify more players
  await tester.ensureVisible(
    find.text('Lionel Messi', skipOffstage: false)
  );
  await tester.pump();
  expect(find.text('Lionel Messi'), findsOneWidget);

  await tester.ensureVisible(
    find.text('Claudio Gamarra', skipOffstage: false)
  );
  await tester.pump();
  expect(find.text('Claudio Gamarra'), findsOneWidget);
});
```

### What This Tests

1. ✅ HTTP adapter correctly parses JSON response
2. ✅ Mapper correctly transforms JSON to entities
3. ✅ Repository returns data to presenter
4. ✅ Presenter transforms entities to ViewModels
5. ✅ Page correctly displays the data
6. ✅ Complete data flow from API to UI

---

## Testing Cache Fallback

When the API fails, the app should fall back to cached data:

```dart
testWidgets('should present cache data', (tester) async {
  // Configure API to fail
  client.simulateServerError();

  // Configure cache to return valid data
  cacheManager.file.simulateResponse(responseJson);

  // Build and verify
  await tester.pumpWidget(sut);
  await tester.pump();

  // Data should come from cache
  await tester.ensureVisible(
    find.text('Cristiano Ronaldo', skipOffstage: false)
  );
  await tester.pump();
  expect(find.text('Cristiano Ronaldo'), findsOneWidget);

  await tester.ensureVisible(
    find.text('Lionel Messi', skipOffstage: false)
  );
  await tester.pump();
  expect(find.text('Lionel Messi'), findsOneWidget);

  await tester.ensureVisible(
    find.text('Claudio Gamarra', skipOffstage: false)
  );
  await tester.pump();
  expect(find.text('Claudio Gamarra'), findsOneWidget);
});
```

### What This Tests

1. ✅ API failure is handled gracefully
2. ✅ Cache fallback is triggered
3. ✅ Cache adapter correctly reads cached data
4. ✅ Same mapper works for cached JSON
5. ✅ UI displays cached data identically
6. ✅ User doesn't notice the fallback

---

## Testing Error States

When both API and cache fail:

```dart
testWidgets('should present error message', (tester) async {
  // Configure API to fail
  client.simulateServerError();

  // Configure cache to fail (invalid JSON)
  cacheManager.file.simulateInvalidResponse();

  // Build and verify error state
  await tester.pumpWidget(sut);
  await tester.pump();

  // Find and verify error message
  await tester.ensureVisible(
    find.text('Algo errado aconteceu, tente novamente.', skipOffstage: false)
  );
  await tester.pump();
  expect(
    find.text('Algo errado aconteceu, tente novamente.'),
    findsOneWidget
  );
});
```

### What This Tests

1. ✅ API failure is detected
2. ✅ Cache fallback is attempted
3. ✅ Cache failure is detected
4. ✅ Error propagates to presenter
5. ✅ Error state is displayed in UI
6. ✅ User sees appropriate error message

---

## Using ensureVisible

### Why `ensureVisible`?

Widgets in a `ListView` or `ScrollView` might be off-screen (offstage). Regular `find.text()` won't find them:

```dart
// ❌ May fail if widget is off-screen
expect(find.text('Player Name'), findsOneWidget);

// ✅ Scrolls to make widget visible first
await tester.ensureVisible(find.text('Player Name', skipOffstage: false));
await tester.pump();
expect(find.text('Player Name'), findsOneWidget);
```

### The Pattern

```dart
// 1. Find including off-screen widgets
await tester.ensureVisible(
  find.text('Widget Text', skipOffstage: false)
);

// 2. Rebuild after scrolling
await tester.pump();

// 3. Now verify (widget is on-screen)
expect(find.text('Widget Text'), findsOneWidget);
```

### `skipOffstage: false`

By default, finders skip off-stage (hidden) widgets. Use `skipOffstage: false` to include them:

```dart
// Find even if off-screen
find.text('Player Name', skipOffstage: false)
```

---

## E2E Test Scenarios Summary

| Scenario | API State | Cache State | Expected Result |
|----------|-----------|-------------|-----------------|
| API Success | ✅ Returns data | - | Show API data |
| Cache Fallback | ❌ Server error | ✅ Valid cache | Show cache data |
| Complete Failure | ❌ Server error | ❌ Invalid/empty | Show error message |

---

## Complete E2E Test Example

```dart
import 'package:advanced_flutter/infra/api/adapters/http_adapter.dart';
import 'package:advanced_flutter/infra/api/repositories/load_next_event_api_repo.dart';
import 'package:advanced_flutter/infra/cache/adapters/cache_manager_adapter.dart';
import 'package:advanced_flutter/infra/cache/repositories/load_next_event_cache_repo.dart';
import 'package:advanced_flutter/infra/mappers/next_event_mapper.dart';
import 'package:advanced_flutter/infra/respositories/load_next_event_from_api_with_cache_fallback_repo.dart';
import 'package:advanced_flutter/main/factories/infra/mappers/next_event_mapper_factory.dart';
import 'package:advanced_flutter/presentation/rx/next_event_rx_presenter.dart';
import 'package:advanced_flutter/ui/pages/next_event_page.dart';

import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

import '../infra/api/mocks/client_spy.dart';
import '../infra/cache/mocks/cache_manager_spy.dart';
import '../mocks/fakes.dart';

void main() {
  late String responseJson;
  late String key;
  late NextEventMapper mapper;
  late ClientSpy client;
  late CacheManagerSpy cacheManager;
  late HttpAdapter httpClient;
  late CacheManagerAdapter cacheClient;
  late LoadNextEventApiRepository apiRepo;
  late LoadNextEventCacheRepository cacheRepo;
  late LoadNextEventFromApiWithCacheFallbackRepository repo;
  late NextEventRxPresenter presenter;
  late MaterialApp sut;

  setUpAll(() {
    responseJson = '''
      {
        "id": "1",
        "groupName": "Team A",
        "date": "2024-01-11T11:10:00.000Z",
        "players": [
          {
            "id": "1",
            "name": "Cristiano Ronaldo",
            "position": "forward",
            "isConfirmed": true,
            "confirmationDate": "2024-01-10T11:07:00.000Z"
          }
        ]
      }
    ''';
  });

  setUp(() {
    key = anyString();
    mapper = makeNextEventMapper();
    client = ClientSpy();
    httpClient = HttpAdapter(client: client);
    apiRepo = LoadNextEventApiRepository(
      httpClient: httpClient,
      url: anyString(),
      mapper: mapper
    );
    cacheManager = CacheManagerSpy();
    cacheClient = CacheManagerAdapter(client: cacheManager);
    cacheRepo = LoadNextEventCacheRepository(
      cacheClient: cacheClient,
      key: key,
      mapper: mapper
    );
    repo = LoadNextEventFromApiWithCacheFallbackRepository(
      loadNextEventFromApi: apiRepo.loadNextEvent,
      loadNextEventFromCache: cacheRepo.loadNextEvent,
      cacheClient: cacheClient,
      key: key,
      mapper: mapper
    );
    presenter = NextEventRxPresenter(nextEventLoader: repo.loadNextEvent);
    sut = MaterialApp(
      home: NextEventPage(presenter: presenter, groupId: anyString())
    );
  });

  testWidgets('should present api data', (tester) async {
    client.responseJson = responseJson;

    await tester.pumpWidget(sut);
    await tester.pump();

    await tester.ensureVisible(
      find.text('Cristiano Ronaldo', skipOffstage: false)
    );
    await tester.pump();
    expect(find.text('Cristiano Ronaldo'), findsOneWidget);
  });

  testWidgets('should present cache data', (tester) async {
    client.simulateServerError();
    cacheManager.file.simulateResponse(responseJson);

    await tester.pumpWidget(sut);
    await tester.pump();

    await tester.ensureVisible(
      find.text('Cristiano Ronaldo', skipOffstage: false)
    );
    await tester.pump();
    expect(find.text('Cristiano Ronaldo'), findsOneWidget);
  });

  testWidgets('should present error message', (tester) async {
    client.simulateServerError();
    cacheManager.file.simulateInvalidResponse();

    await tester.pumpWidget(sut);
    await tester.pump();

    await tester.ensureVisible(
      find.text('Algo errado aconteceu, tente novamente.', skipOffstage: false)
    );
    await tester.pump();
    expect(
      find.text('Algo errado aconteceu, tente novamente.'),
      findsOneWidget
    );
  });
}
```

---

## E2E Testing Best Practices

| Practice | Description |
|----------|-------------|
| Use real components | Only mock external boundaries |
| Use factory methods | `makeNextEventMapper()` for real instances |
| Use `setUpAll` for static data | JSON responses that don't change |
| Use `setUp` for fresh instances | Reset state between tests |
| Use `ensureVisible` | For scrollable content |
| Test all paths | Success, fallback, error |

---

## When to Write E2E Tests

| Scenario | Write E2E Test? |
|----------|-----------------|
| Critical user flows | ✅ Yes |
| Complex integrations | ✅ Yes |
| Fallback behaviors | ✅ Yes |
| Simple components | ❌ Use unit/widget tests |
| Pure business logic | ❌ Use unit tests |

---

## Next Steps

Now that you understand all testing layers, proceed to:

**[→ Best Practices](08_BEST_PRACTICES.md)**: Summary of patterns and guidelines for maintainable tests.
