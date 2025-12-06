# Infrastructure Layer Testing

## Introduction

The **Infrastructure Layer** handles external concerns like:
- **Repositories**: Data access abstraction
- **Adapters**: Interface adapters for external libraries
- **Mappers**: Data transformation between layers
- **Clients**: Low-level HTTP/Cache operations

This layer requires **mocking external dependencies** while testing the logic within.

---

## Table of Contents

1. [Repository Testing](#repository-testing)
2. [Adapter Testing](#adapter-testing)
3. [Mapper Testing](#mapper-testing)
4. [Error Handling Patterns](#error-handling-patterns)
5. [Cache Testing](#cache-testing)
6. [Composite Repository Pattern](#composite-repository-pattern)

---

## Repository Testing

Repositories abstract data access. Test that they:
- Call dependencies with correct parameters
- Transform data correctly
- Handle errors appropriately

### Example: API Repository Test

```dart
import 'package:advanced_flutter/domain/entities/errors.dart';
import 'package:advanced_flutter/domain/entities/next_event.dart';
import 'package:advanced_flutter/infra/api/repositories/load_next_event_api_repo.dart';

import 'package:flutter_test/flutter_test.dart';

import '../../../mocks/fakes.dart';
import '../../mocks/mapper_spy.dart';
import '../mocks/http_get_client_spy.dart';

void main() {
  late String groupId;
  late String url;
  late HttpGetClientSpy httpClient;
  late MapperSpy<NextEvent> mapper;
  late LoadNextEventApiRepository sut;

  setUp(() {
    groupId = anyString();
    url = anyString();
    httpClient = HttpGetClientSpy();
    mapper = MapperSpy(toDtoOutput: anyNextEvent());
    sut = LoadNextEventApiRepository(
      httpClient: httpClient,
      url: url,
      mapper: mapper
    );
  });

  test('should call HttpClient with correct input', () async {
    await sut.loadNextEvent(groupId: groupId);

    expect(httpClient.url, url);
    expect(httpClient.params, { 'groupId': groupId });
    expect(httpClient.callsCount, 1);
  });

  test('should return NextEvent on success', () async {
    final event = await sut.loadNextEvent(groupId: groupId);

    expect(mapper.toDtoIntput, httpClient.response);
    expect(mapper.toDtoIntputCallsCount, 1);
    expect(event, mapper.toDtoOutput);
  });

  test('should rethrow on error', () async {
    final error = Error();
    httpClient.error = error;

    final future = sut.loadNextEvent(groupId: groupId);

    expect(future, throwsA(error));
  });

  test('should throw UnexpectedError on null response', () async {
    httpClient.response = null;

    final future = sut.loadNextEvent(groupId: groupId);

    expect(future, throwsA(isA<UnexpectedError>()));
  });
}
```

### Key Testing Points for Repositories

| What to Test | Why |
|--------------|-----|
| Correct URL/endpoint | Ensures correct API is called |
| Correct parameters | Ensures data is passed correctly |
| Call count | Prevents multiple calls |
| Data transformation | Mapper is used correctly |
| Error propagation | Errors are handled/rethrown |
| Null handling | Edge cases are covered |

---

## Adapter Testing

Adapters wrap external libraries and handle low-level operations.

### HTTP Adapter Test

The HTTP adapter handles:
- URL parameter substitution
- Query string building
- Header management
- Response parsing
- Error mapping

```dart
import 'package:advanced_flutter/domain/entities/errors.dart';
import 'package:advanced_flutter/infra/api/adapters/http_adapter.dart';

import 'package:flutter_test/flutter_test.dart';

import '../../../mocks/fakes.dart';
import '../mocks/client_spy.dart';

void main() {
  late ClientSpy client;
  late HttpAdapter sut;
  late String url;

  setUp(() {
    client = ClientSpy();
    client.responseJson = '''
      {
        "key1": "value1",
        "key2": "value2"
      }
    ''';
    url = anyString();
    sut = HttpAdapter(client: client);
  });

  group('get', () {
    test('should request with correct method', () async {
      await sut.get(url: url);

      expect(client.method, 'get');
      expect(client.callsCount, 1);
    });

    test('should request with correct url', () async {
      await sut.get(url: url);

      expect(client.url, url);
    });

    test('should request with default headers', () async {
      await sut.get(url: url);

      expect(client.headers?['content-type'], 'application/json');
      expect(client.headers?['accept'], 'application/json');
    });

    test('should append headers', () async {
      await sut.get(url: url, headers: {
        'h1': 'value1',
        'h2': 'value2',
        'h3': 123
      });

      expect(client.headers?['content-type'], 'application/json');
      expect(client.headers?['accept'], 'application/json');
      expect(client.headers?['h1'], 'value1');
      expect(client.headers?['h2'], 'value2');
      expect(client.headers?['h3'], '123');
    });
  });
}
```

### URL Parameter Substitution Tests

```dart
test('should request with correct params', () async {
  url = 'http://anyurl.com/:p1/:p2/:p3';

  await sut.get(url: url, params: { 'p1': 'v1', 'p2': 'v2', 'p3': 123 });

  expect(client.url, 'http://anyurl.com/v1/v2/123');
});

test('should request with optional param', () async {
  url = 'http://anyurl.com/:p1/:p2';

  await sut.get(url: url, params: { 'p1': 'v1', 'p2': null });

  expect(client.url, 'http://anyurl.com/v1');
});

test('should request with invalid params', () async {
  url = 'http://anyurl.com/:p1/:p2';

  await sut.get(url: url, params: { 'p3': 'v3' });

  expect(client.url, 'http://anyurl.com/:p1/:p2');
});
```

### Query String Tests

```dart
test('should request with correct queryStrings', () async {
  await sut.get(url: url, queryString: {
    'q1': 'v1',
    'q2': 'v2',
    'q3': 123
  });

  expect(client.url, '$url?q1=v1&q2=v2&q3=123');
});

test('should request with correct queryStrings and params', () async {
  url = 'http://anyurl.com/:p3/:p4';

  await sut.get(
    url: url,
    queryString: { 'q1': 'v1', 'q2': 'v2' },
    params: { 'p3': 'v3', 'p4': 'v4' }
  );

  expect(client.url, 'http://anyurl.com/v3/v4?q1=v1&q2=v2');
});
```

### HTTP Status Code Error Tests

```dart
test('should throw UnexpectedError on 400', () async {
  client.simulateBadRequestError();

  final future = sut.get(url: url);

  expect(future, throwsA(isA<UnexpectedError>()));
});

test('should throw SessionExpiredError on 401', () async {
  client.simulateUnauthorizedError();

  final future = sut.get(url: url);

  expect(future, throwsA(isA<SessionExpiredError>()));
});

test('should throw UnexpectedError on 403', () async {
  client.simulateForbiddenError();

  final future = sut.get(url: url);

  expect(future, throwsA(isA<UnexpectedError>()));
});

test('should throw UnexpectedError on 404', () async {
  client.simulateNotFoundError();

  final future = sut.get(url: url);

  expect(future, throwsA(isA<UnexpectedError>()));
});

test('should throw UnexpectedError on 500', () async {
  client.simulateServerError();

  final future = sut.get(url: url);

  expect(future, throwsA(isA<UnexpectedError>()));
});
```

### Response Parsing Tests

```dart
test('should return a Map', () async {
  final data = await sut.get(url: url);

  expect(data?['key1'], 'value1');
  expect(data?['key2'], 'value2');
});

test('should return a List', () async {
  client.responseJson = '''
    [{
      "key": "value1"
    }, {
      "key": "value2"
    }]
  ''';

  final data = await sut.get(url: url);

  expect(data?[0]['key'], 'value1');
  expect(data?[1]['key'], 'value2');
});

test('should return a Map with List', () async {
  client.responseJson = '''
    {
      "key1": "value1",
      "key2": [{
        "key": "value1"
      }, {
        "key": "value2"
      }]
    }
  ''';

  final data = await sut.get(url: url);

  expect(data?['key1'], 'value1');
  expect(data?['key2'][0]['key'], 'value1');
  expect(data?['key2'][1]['key'], 'value2');
});

test('should return null on 200 with empty response', () async {
  client.responseJson = '';

  final data = await sut.get(url: url);

  expect(data, isNull);
});

test('should return null on 204', () async {
  client.simulateNoContent();

  final data = await sut.get(url: url);

  expect(data, isNull);
});
```

---

## Mapper Testing

Mappers transform data between layers. Test both directions:
- `toDto()`: JSON → Entity
- `toJson()`: Entity → JSON

### Mapper Test Example

```dart
import 'package:advanced_flutter/domain/entities/next_event_player.dart';
import 'package:advanced_flutter/infra/mappers/next_event_player_mapper.dart';
import 'package:flutter_test/flutter_test.dart';

import '../../mocks/fakes.dart';

void main() {
  late NextEventPlayerMapper sut;

  setUp(() {
    sut = NextEventPlayerMapper();
  });

  test('should map to dto', () {
    final json = {
      'id': anyString(),
      'name': anyString(),
      'position': anyString(),
      'photo': anyString(),
      'confirmationDate': '2024-08-29T11:00:00.000',
      'isConfirmed': anyBool()
    };

    final dto = sut.toDto(json);

    expect(dto.id, json['id']);
    expect(dto.name, json['name']);
    expect(dto.position, json['position']);
    expect(dto.photo, json['photo']);
    expect(dto.confirmationDate, DateTime(2024, 8, 29, 11, 0));
    expect(dto.isConfirmed, json['isConfirmed']);
  });

  test('should map to dto with empty fields', () {
    final json = {
      'id': anyString(),
      'name': anyString(),
      'isConfirmed': anyBool()
    };

    final dto = sut.toDto(json);

    expect(dto.id, json['id']);
    expect(dto.name, json['name']);
    expect(dto.position, isNull);
    expect(dto.photo, isNull);
    expect(dto.confirmationDate, isNull);
    expect(dto.isConfirmed, json['isConfirmed']);
  });

  test('should map to json', () {
    final dto = NextEventPlayer(
      id: anyString(),
      name: anyString(),
      isConfirmed: anyBool(),
      photo: anyString(),
      position: anyString(),
      confirmationDate: DateTime(2024, 8, 29, 13, 0)
    );

    final json = sut.toJson(dto);

    expect(json['id'], dto.id);
    expect(json['name'], dto.name);
    expect(json['position'], dto.position);
    expect(json['photo'], dto.photo);
    expect(json['confirmationDate'], '2024-08-29T13:00:00.000');
    expect(json['isConfirmed'], dto.isConfirmed);
  });

  test('should map to json with empty fields', () {
    final dto = NextEventPlayer(
      id: anyString(),
      name: anyString(),
      isConfirmed: anyBool()
    );

    final json = sut.toJson(dto);

    expect(json['id'], dto.id);
    expect(json['name'], dto.name);
    expect(json['position'], isNull);
    expect(json['photo'], isNull);
    expect(json['confirmationDate'], isNull);
    expect(json['isConfirmed'], dto.isConfirmed);
  });
}
```

### Mapper Testing Checklist

- [ ] Test mapping with all fields populated
- [ ] Test mapping with optional fields missing
- [ ] Test date format conversions
- [ ] Test both directions (toDto and toJson)
- [ ] Verify field-by-field mapping

---

## Cache Testing

### Cache Adapter Test with Groups

```dart
import 'package:advanced_flutter/domain/entities/errors.dart';
import 'package:advanced_flutter/infra/cache/adapters/cache_manager_adapter.dart';

import 'package:flutter_test/flutter_test.dart';

import '../../../mocks/fakes.dart';
import '../mocks/cache_manager_spy.dart';

void main() {
  late String key;
  late CacheManagerSpy client;
  late CacheManagerAdapter sut;

  setUp(() {
    key = anyString();
    client = CacheManagerSpy();
    sut = CacheManagerAdapter(client: client);
  });

  group('get', () {
    test('should call getFileFromCache with correct input', () async {
      await sut.get(key: key);

      expect(client.key, key);
      expect(client.getFileFromCacheCallsCount, 1);
    });

    test('should return null if FileInfo is empty', () async {
      client.simulateEmptyFileInfo();

      final json = await sut.get(key: key);

      expect(json, isNull);
    });

    test('should return null if cache is old', () async {
      client.simulateCacheOld();

      final json = await sut.get(key: key);

      expect(json, isNull);
    });

    test('should call file.exists only once', () async {
      await sut.get(key: key);

      expect(client.file.existsCallsCount, 1);
    });

    test('should return null if file is empty', () async {
      client.file.simulateFileEmpty();

      final json = await sut.get(key: key);

      expect(json, isNull);
    });

    test('should call file.readAsString only once', () async {
      await sut.get(key: key);

      expect(client.file.readAsStringCallsCount, 1);
    });

    test('should return null if cache is invalid', () async {
      client.file.simulateInvalidResponse();

      final json = await sut.get(key: key);

      expect(json, isNull);
    });

    test('should return json if cache is valid', () async {
      client.file.simulateResponse('''
        {
          "key1": "value1",
          "key2": "value2"
        }
      ''');

      final json = await sut.get(key: key);

      expect(json['key1'], 'value1');
      expect(json['key2'], 'value2');
    });

    test('should return null if file.readAsString fails', () async {
      client.file.simulateReadAsStringError();

      final json = await sut.get(key: key);

      expect(json, isNull);
    });

    test('should return null if file.exists fails', () async {
      client.file.simulateExistsError();

      final json = await sut.get(key: key);

      expect(json, isNull);
    });

    test('should return null if getFileFromCache fails', () async {
      client.simulateGetFileFromCacheError();

      final json = await sut.get(key: key);

      expect(json, isNull);
    });
  });

  group('save', () {
    late Map value;

    setUp(() {
      value = {
        'key1': anyString(),
        'key2': anyIsoDate(),
        'key3': anyBool(),
        'key4': anyInt()
      };
    });

    test('should call putFile with correct input', () async {
      await sut.save(key: key, value: value);

      expect(client.key, key);
      expect(client.fileExtension, 'json');
      expect(client.fileBytesDecoded, value);
      expect(client.putFileCallsCount, 1);
    });

    test('should throw UnexpectedError when putFile fails', () async {
      client.simulatePutFileError();

      final future = sut.save(key: key, value: value);

      expect(future, throwsA(isA<UnexpectedError>()));
    });
  });
}
```

---

## Composite Repository Pattern

The composite repository combines multiple data sources with fallback:

```dart
import 'package:advanced_flutter/domain/entities/errors.dart';
import 'package:advanced_flutter/domain/entities/next_event.dart';
import 'package:advanced_flutter/infra/respositories/load_next_event_from_api_with_cache_fallback_repo.dart';

import 'package:flutter_test/flutter_test.dart';

import '../../mocks/fakes.dart';
import '../cache/mocks/cache_save_client_mock.dart';
import '../mocks/load_next_event_repo_spy.dart';
import '../mocks/mapper_spy.dart';

void main() {
  late String groupId;
  late String key;
  late LoadNextEventRepositorySpy apiRepo;
  late LoadNextEventRepositorySpy cacheRepo;
  late CacheSaveClientMock cacheClient;
  late MapperSpy<NextEvent> mapper;
  late LoadNextEventFromApiWithCacheFallbackRepository sut;

  setUp(() {
    groupId = anyString();
    key = anyString();
    apiRepo = LoadNextEventRepositorySpy();
    cacheRepo = LoadNextEventRepositorySpy();
    cacheClient = CacheSaveClientMock();
    mapper = MapperSpy(toDtoOutput: anyNextEvent());
    sut = LoadNextEventFromApiWithCacheFallbackRepository(
      key: key,
      cacheClient: cacheClient,
      loadNextEventFromApi: apiRepo.loadNextEvent,
      loadNextEventFromCache: cacheRepo.loadNextEvent,
      mapper: mapper
    );
  });

  // Test primary source (API)
  test('should load event data from api repo', () async {
    await sut.loadNextEvent(groupId: groupId);

    expect(apiRepo.groupId, groupId);
    expect(apiRepo.callsCount, 1);
  });

  // Test caching behavior
  test('should save event data from api on cache', () async {
    await sut.loadNextEvent(groupId: groupId);

    expect(cacheClient.key, '$key:$groupId');
    expect(cacheClient.value, mapper.toJsonOutput);
    expect(mapper.toJsonIntput, apiRepo.output);
    expect(mapper.toJsonCallsCount, 1);
  });

  // Test success path
  test('should return api data on success', () async {
    final event = await sut.loadNextEvent(groupId: groupId);

    expect(event, apiRepo.output);
  });

  // Test non-recoverable error (should NOT fallback)
  test('should rethrow api error when its SessionExpiredError', () async {
    apiRepo.error = SessionExpiredError();

    final future = sut.loadNextEvent(groupId: groupId);

    expect(future, throwsA(apiRepo.error));
  });

  // Test fallback to cache
  test('should load event data from cache repo when api fails', () async {
    apiRepo.error = Error();

    await sut.loadNextEvent(groupId: groupId);

    expect(cacheRepo.groupId, groupId);
    expect(cacheRepo.callsCount, 1);
  });

  // Test cache data returned
  test('should return cache data when api fails', () async {
    apiRepo.error = Error();

    final event = await sut.loadNextEvent(groupId: groupId);

    expect(event, cacheRepo.output);
  });

  // Test both sources fail
  test('should rethrow cache error when api and cache fails', () async {
    apiRepo.error = Error();
    cacheRepo.error = Error();

    final future = sut.loadNextEvent(groupId: groupId);

    expect(future, throwsA(cacheRepo.error));
  });
}
```

### Composite Repository Test Scenarios

| Scenario | Expected Behavior |
|----------|-------------------|
| API succeeds | Return API data, save to cache |
| API fails (recoverable) | Fallback to cache |
| API fails (SessionExpired) | Rethrow, no fallback |
| API and Cache fail | Rethrow cache error |

---

## Error Handling Patterns

### Throwing Specific Errors

```dart
test('should rethrow api error when its SessionExpiredError', () async {
  apiRepo.error = SessionExpiredError();

  final future = sut.loadNextEvent(groupId: groupId);

  expect(future, throwsA(apiRepo.error));
});
```

### Throwing Error Types

```dart
test('should throw UnexpectedError on null response', () async {
  httpClient.response = null;

  final future = sut.loadNextEvent(groupId: groupId);

  expect(future, throwsA(isA<UnexpectedError>()));
});
```

### Rethrowing Original Errors

```dart
test('should rethrow on error', () async {
  final error = Error();
  httpClient.error = error;

  final future = sut.loadNextEvent(groupId: groupId);

  expect(future, throwsA(error));  // Same instance
});
```

---

## Infrastructure Testing Best Practices

| Practice | Description |
|----------|-------------|
| Use `group()` | Organize related tests (get, save, etc.) |
| Test all paths | Success, error, edge cases |
| Verify inputs | Check parameters passed to dependencies |
| Check call counts | Prevent duplicate calls |
| Test transformations | Verify data is mapped correctly |
| Use simulation methods | `simulate*()` for clean error setup |

---

## Next Steps

Now that you understand infrastructure testing, proceed to:

**[→ Presentation Layer Testing](05_PRESENTATION_TESTING.md)**: Learn how to test reactive presenters with RxDart streams.
