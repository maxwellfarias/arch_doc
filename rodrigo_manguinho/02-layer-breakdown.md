# 02 - Layer Breakdown

## Table of Contents

1. [Introduction](#introduction)
2. [What You'll Learn](#what-youll-learn)
3. [Layer Overview](#layer-overview)
4. [Domain Layer](#domain-layer)
5. [Data Layer](#data-layer)
6. [Infrastructure Layer](#infrastructure-layer)
7. [Presentation Layer](#presentation-layer)
8. [UI Layer](#ui-layer)
9. [Validation Layer](#validation-layer)
10. [Main Layer (Composition Root)](#main-layer-composition-root)
11. [Layer Interaction Patterns](#layer-interaction-patterns)
12. [Key Takeaways](#key-takeaways)
13. [Next Steps](#next-steps)

---

## Introduction

This document provides an **in-depth exploration** of each architectural layer in the Clean Flutter App. We'll examine the purpose, structure, and implementation details of every layer, with concrete code examples from the actual codebase.

**Why layers matter:**
Layers create boundaries that isolate concerns, making your code easier to understand, test, and modify. Think of layers like floors in a building - each floor has a specific purpose and communicates with other floors through well-defined interfaces (elevators and stairs).

---

## What You'll Learn

By the end of this document, you will understand:

- ✅ The exact purpose and responsibility of each layer
- ✅ What types of code belong in each layer
- ✅ How layers communicate with each other
- ✅ The file structure and organization within each layer
- ✅ Real code examples from the project
- ✅ Common patterns used in each layer
- ✅ What NOT to do in each layer (anti-patterns)

---

## Layer Overview

### The Seven Layers

```
┌─────────────────────────────────────────────────────────┐
│  Layer            │  Responsibility                      │
├─────────────────────────────────────────────────────────┤
│  1. Domain        │  Business logic & entities           │
│  2. Data          │  Data operations & use cases         │
│  3. Infrastructure│  External library adapters           │
│  4. Presentation  │  Screen logic & state management     │
│  5. UI            │  Visual components & widgets         │
│  6. Validation    │  Form validation rules               │
│  7. Main          │  Dependency injection & composition  │
└─────────────────────────────────────────────────────────┘
```

### Layer Dependency Map

```
        MAIN
         │
    ┌────┴────┐
    │         │
   UI    INFRASTRUCTURE
    │         │
    │         │
PRESENTATION  │
    │         │
    └────┬────┘
         │
    ┌────┴────┐
    │         │
  DOMAIN   VALIDATION
         (No dependencies)
```

---

## Domain Layer

### Purpose

The **Domain Layer** is the **heart** of your application. It contains:
- Pure business logic
- Business entities
- Business rules (use cases)
- Zero dependencies on frameworks or external libraries

**Analogy:** This is like the blueprint and business plan for a restaurant - it defines what dishes you serve (entities) and how you prepare them (use cases), but doesn't care about the specific kitchen equipment (frameworks) you use.

---

### Directory Structure

```
lib/domain/
├── entities/
│   ├── account_entity.dart
│   ├── survey_entity.dart
│   ├── survey_answer_entity.dart
│   └── survey_result_entity.dart
├── usecases/
│   ├── authentication.dart
│   ├── add_account.dart
│   ├── load_current_account.dart
│   ├── save_current_account.dart
│   ├── load_surveys.dart
│   ├── load_survey_result.dart
│   └── save_survey_result.dart
├── helpers/
│   └── domain_error.dart
└── entities.dart (barrel file)
```

---

### Entities

**What are entities?**
Entities are **business objects** that represent core concepts in your application domain.

**Key characteristics:**
- Extend `Equatable` for value-based equality
- Contain only data (no business logic)
- Immutable (all fields are `final`)
- Framework-independent (pure Dart)

**Example: AccountEntity**

File: [lib/domain/entities/account_entity.dart](../lib/domain/entities/account_entity.dart)

```dart
import 'package:equatable/equatable.dart';

class AccountEntity extends Equatable {
  final String token;

  List get props => [token];

  AccountEntity({ required this.token });
}
```

**Why Equatable?**
```dart
// Without Equatable
final account1 = AccountEntity(token: 'abc123');
final account2 = AccountEntity(token: 'abc123');
print(account1 == account2); // false ❌ (different instances)

// With Equatable
final account1 = AccountEntity(token: 'abc123');
final account2 = AccountEntity(token: 'abc123');
print(account1 == account2); // true ✅ (same values)
```

This is essential for:
- Testing (comparing expected vs actual)
- State management (detecting changes)
- Collections (using entities as map keys)

---

**Example: SurveyEntity**

File: [lib/domain/entities/survey_entity.dart](../lib/domain/entities/survey_entity.dart)

```dart
class SurveyEntity extends Equatable {
  final String id;
  final String question;
  final DateTime dateTime;
  final bool didAnswer;

  List get props => [id, question, dateTime, didAnswer];

  SurveyEntity({
    required this.id,
    required this.question,
    required this.dateTime,
    required this.didAnswer,
  });
}
```

**Note:** All entities are read-only value objects. To "modify" an entity, you create a new instance.

---

### Use Cases

**What are use cases?**
Use cases define **what the application can do** - they are business operations.

**Key characteristics:**
- Abstract classes (interfaces/contracts)
- Define method signatures, not implementations
- Named after business actions (verb + noun)
- Return domain entities or primitives

**Example: Authentication Use Case**

File: [lib/domain/usecases/authentication.dart](../lib/domain/usecases/authentication.dart)

```dart
import '../entities/entities.dart';
import 'package:equatable/equatable.dart';

abstract class Authentication {
  Future<AccountEntity> auth(AuthenticationParams params);
}

class AuthenticationParams extends Equatable {
  final String email;
  final String secret;

  List get props => [email, secret];

  AuthenticationParams({ required this.email, required this.secret });
}
```

**Key points:**
1. `Authentication` is **abstract** - it's a contract, not an implementation
2. The method `auth` is the business operation
3. `AuthenticationParams` is a **value object** (also Equatable)
4. Returns `AccountEntity` (domain entity)
5. Uses `Future` because authentication is async

---

**Example: LoadSurveys Use Case**

File: [lib/domain/usecases/load_surveys.dart](../lib/domain/usecases/load_surveys.dart)

```dart
import '../entities/entities.dart';

abstract class LoadSurveys {
  Future<List<SurveyEntity>> load();
}
```

**Even simpler!** No parameters needed - just load the surveys.

---

**Example: SaveSurveyResult Use Case**

File: [lib/domain/usecases/save_survey_result.dart](../lib/domain/usecases/save_survey_result.dart)

```dart
import '../entities/entities.dart';
import 'package:equatable/equatable.dart';

abstract class SaveSurveyResult {
  Future<SurveyResultEntity> save({required String answer});
}
```

**Pattern:** Always return domain entities, accept domain parameters.

---

### Domain Helpers

**DomainError Enum**

File: [lib/domain/helpers/domain_error.dart](../lib/domain/helpers/domain_error.dart)

```dart
enum DomainError {
  unexpected,
  invalidCredentials,
  emailInUse,
  accessDenied
}
```

**Purpose:** Define business-level errors. These are errors that make sense in the business context, not technical errors.

**Mapping:**
- `unexpected` → Something went wrong (500 errors, network issues)
- `invalidCredentials` → Wrong email/password
- `emailInUse` → Email already registered
- `accessDenied` → User doesn't have permission (401/403)

---

### Domain Layer Rules

**✅ DO:**
- Define business entities
- Define use case interfaces
- Define business errors
- Use pure Dart (no Flutter dependencies)
- Keep entities immutable

**❌ DON'T:**
- Import Flutter packages
- Import external libraries (except Equatable)
- Implement use cases here (that's Data layer's job)
- Put UI logic here
- Put database/API logic here

---

### Domain Layer Benefits

**1. Testability**
```dart
// Test business logic without ANY external dependencies
test('Should throw invalidCredentials on 401', () {
  // No need for HTTP client, database, or UI
});
```

**2. Portability**
```dart
// Use the same domain logic in:
// - Flutter mobile app
// - Flutter web app
// - Dart CLI tool
// - Dart backend server
```

**3. Clarity**
```dart
// Reading a use case interface tells you EXACTLY what the app does
abstract class Authentication { ... }
abstract class LoadSurveys { ... }
abstract class SaveSurveyResult { ... }
```

---

## Data Layer

### Purpose

The **Data Layer** provides **concrete implementations** of domain use cases. It:
- Implements business operations defined in Domain
- Handles data retrieval and storage
- Transforms external data to domain entities
- Defines protocols for Infrastructure layer

**Analogy:** If Domain is the blueprint, Data is the construction crew that actually builds according to the blueprint.

---

### Directory Structure

```
lib/data/
├── usecases/
│   ├── authentication/
│   │   ├── remote_authentication.dart
│   │   └── local_save_current_account.dart
│   ├── load_surveys/
│   │   ├── remote_load_surveys.dart
│   │   └── local_load_surveys.dart
│   └── ... (other use cases)
├── models/
│   ├── remote_account_model.dart
│   ├── remote_survey_model.dart
│   ├── local_survey_model.dart
│   └── ... (other models)
├── http/
│   ├── http_client.dart
│   └── http_error.dart
└── cache/
    ├── cache_storage.dart
    ├── save_secure_cache_storage.dart
    ├── fetch_secure_cache_storage.dart
    └── delete_secure_cache_storage.dart
```

---

### Use Case Implementations

**Naming Convention:**
- `Remote*` - Fetches data from API
- `Local*` - Stores/retrieves data locally

**Example: RemoteAuthentication**

File: [lib/data/usecases/authentication/remote_authentication.dart](../lib/data/usecases/authentication/remote_authentication.dart)

```dart
import '../../../domain/entities/entities.dart';
import '../../../domain/helpers/helpers.dart';
import '../../../domain/usecases/usecases.dart';
import '../../http/http.dart';
import '../../models/models.dart';

class RemoteAuthentication implements Authentication {
  final HttpClient httpClient;
  final String url;

  RemoteAuthentication({
    required this.httpClient,
    required this.url
  });

  Future<AccountEntity> auth(AuthenticationParams params) async {
    final body = RemoteAuthenticationParams.fromDomain(params).toJson();
    try {
      final httpResponse = await httpClient.request(
        url: url,
        method: 'post',
        body: body
      );
      return RemoteAccountModel.fromJson(httpResponse).toEntity();
    } on HttpError catch(error) {
      throw error == HttpError.unauthorized
        ? DomainError.invalidCredentials
        : DomainError.unexpected;
    }
  }
}
```

**Key points:**

1. **Implements domain interface** (`implements Authentication`)
2. **Constructor injection** - receives dependencies
3. **Depends on abstractions** (`HttpClient`, not concrete `HttpAdapter`)
4. **Error mapping** - `HttpError` → `DomainError`
5. **Data transformation** - JSON → Model → Entity

**Data flow:**
```
1. Receives AuthenticationParams (domain)
2. Converts to RemoteAuthenticationParams (data)
3. Serializes to JSON
4. Makes HTTP request via HttpClient abstraction
5. Receives JSON response
6. Parses to RemoteAccountModel
7. Converts to AccountEntity (domain)
8. Returns to caller
```

---

**RemoteAuthenticationParams (Helper Class)**

```dart
class RemoteAuthenticationParams {
  final String email;
  final String password;

  RemoteAuthenticationParams({
    required this.email,
    required this.password
  });

  factory RemoteAuthenticationParams.fromDomain(AuthenticationParams params) =>
    RemoteAuthenticationParams(email: params.email, password: params.secret);

  Map toJson() => {'email': email, 'password': password};
}
```

**Why a separate class?**
- Domain uses `secret` (generic term)
- API expects `password` (specific term)
- Separation of concerns: domain doesn't know about JSON

---

### Data Models

**What are models?**
Models are **adapters** between external data formats and domain entities.

**Key characteristics:**
- Know how to parse external data (JSON, database, etc.)
- Can convert to/from domain entities
- Handle data validation and error cases

**Example: RemoteAccountModel**

File: [lib/data/models/remote_account_model.dart](../lib/data/models/remote_account_model.dart)

```dart
import '../../domain/entities/entities.dart';
import '../http/http.dart';

class RemoteAccountModel {
  final String accessToken;

  RemoteAccountModel({ required this.accessToken });

  factory RemoteAccountModel.fromJson(Map json) {
    if (!json.containsKey('accessToken')) {
      throw HttpError.invalidData;
    }
    return RemoteAccountModel(accessToken: json['accessToken']);
  }

  AccountEntity toEntity() => AccountEntity(token: accessToken);
}
```

**Methods explained:**

1. **`fromJson(Map json)`** - Factory constructor
   - Parses JSON from API
   - Validates data (throws if invalid)
   - Creates model instance

2. **`toEntity()`** - Converter
   - Transforms model to domain entity
   - Maps field names (accessToken → token)

**Data transformation chain:**
```
JSON (API) → Model (Data layer) → Entity (Domain layer)
```

---

**Example: RemoteSurveyModel (More Complex)**

File: [lib/data/models/remote_survey_model.dart](../lib/data/models/remote_survey_model.dart)

```dart
import '../../domain/entities/entities.dart';
import '../http/http.dart';

class RemoteSurveyModel {
  final String id;
  final String question;
  final String date;
  final bool didAnswer;

  RemoteSurveyModel({
    required this.id,
    required this.question,
    required this.date,
    required this.didAnswer,
  });

  factory RemoteSurveyModel.fromJson(Map json) {
    if (!json.keys.toSet().containsAll(['id', 'question', 'date', 'didAnswer'])) {
      throw HttpError.invalidData;
    }
    return RemoteSurveyModel(
      id: json['id'],
      question: json['question'],
      date: json['date'],
      didAnswer: json['didAnswer'],
    );
  }

  SurveyEntity toEntity() => SurveyEntity(
    id: id,
    question: question,
    dateTime: DateTime.parse(date),
    didAnswer: didAnswer,
  );
}
```

**Note the transformation:**
- `date` (String in JSON) → `dateTime` (DateTime in entity)
- Model validates ALL required fields exist

---

**Example: LocalSurveyModel (Bidirectional)**

File: [lib/data/models/local_survey_model.dart](../lib/data/models/local_survey_model.dart)

```dart
import '../../domain/entities/entities.dart';

class LocalSurveyModel {
  final String id;
  final String question;
  final String date;
  final bool didAnswer;

  LocalSurveyModel({
    required this.id,
    required this.question,
    required this.date,
    required this.didAnswer,
  });

  factory LocalSurveyModel.fromJson(Map json) {
    if (!json.keys.toSet().containsAll(['id', 'question', 'date', 'didAnswer'])) {
      throw Exception('Invalid data');
    }
    return LocalSurveyModel(
      id: json['id'],
      question: json['question'],
      date: json['date'],
      didAnswer: json['didAnswer'],
    );
  }

  factory LocalSurveyModel.fromEntity(SurveyEntity entity) =>
    LocalSurveyModel(
      id: entity.id,
      question: entity.question,
      date: entity.dateTime.toIso8601String(),
      didAnswer: entity.didAnswer,
    );

  SurveyEntity toEntity() => SurveyEntity(
    id: id,
    question: question,
    dateTime: DateTime.parse(date),
    didAnswer: didAnswer,
  );

  Map toJson() => {
    'id': id,
    'question': question,
    'date': date,
    'didAnswer': didAnswer,
  };
}
```

**Why bidirectional?**
- `fromJson()` - Load from cache
- `fromEntity()` - Convert entity for caching
- `toEntity()` - Convert cached data to entity
- `toJson()` - Serialize for storage

**Use case:**
```dart
// Loading from cache
final json = await cache.fetch('surveys');
final model = LocalSurveyModel.fromJson(json);
final entity = model.toEntity();

// Saving to cache
final entity = // ... from API
final model = LocalSurveyModel.fromEntity(entity);
final json = model.toJson();
await cache.save('surveys', json);
```

---

### Data Layer Protocols

**What are protocols?**
Protocols are **interfaces** that define how Data layer communicates with Infrastructure layer.

**Example: HttpClient**

File: [lib/data/http/http_client.dart](../lib/data/http/http_client.dart)

```dart
abstract class HttpClient {
  Future<dynamic> request({
    required String url,
    required String method,
    Map? body,
    Map? headers,
  });
}
```

**Purpose:**
- Data layer DEFINES the interface
- Infrastructure layer IMPLEMENTS it
- Data layer doesn't know about `http` package or `dio`

**Benefits:**
```dart
// Easy to swap implementations
class HttpAdapter implements HttpClient { } // Using 'http' package
class DioAdapter implements HttpClient { }  // Using 'dio' package
class MockHttpClient implements HttpClient { } // For testing
```

---

**Example: HttpError**

File: [lib/data/http/http_error.dart](../lib/data/http/http_error.dart)

```dart
enum HttpError {
  badRequest,
  unauthorized,
  forbidden,
  notFound,
  serverError,
  invalidData
}
```

**Error hierarchy:**
```
Infrastructure Layer → HttpError (Data layer)
Data Layer → DomainError (Domain layer)
Domain Layer → UIError (UI layer)
```

Each layer only knows about its own errors!

---

**Example: Cache Storage Protocols**

File: [lib/data/cache/save_secure_cache_storage.dart](../lib/data/cache/save_secure_cache_storage.dart)

```dart
abstract class SaveSecureCacheStorage {
  Future<void> save({
    required String key,
    required String value
  });
}
```

File: [lib/data/cache/fetch_secure_cache_storage.dart](../lib/data/cache/fetch_secure_cache_storage.dart)

```dart
abstract class FetchSecureCacheStorage {
  Future<String> fetch(String key);
}
```

**Why separate interfaces?**
- **Interface Segregation Principle** (ISP)
- Classes only depend on methods they use
- Easier to mock in tests
- More flexible composition

```dart
// Class only needs to save? Depend only on SaveSecureCacheStorage
class LocalSaveCurrentAccount {
  final SaveSecureCacheStorage saveSecureCacheStorage;
  // Doesn't need fetch or delete
}

// Class needs all operations? Implement all interfaces
class SecureStorageAdapter implements
  SaveSecureCacheStorage,
  FetchSecureCacheStorage,
  DeleteSecureCacheStorage {
  // Implements all three
}
```

---

### Data Layer Rules

**✅ DO:**
- Implement domain use cases
- Create models for data transformation
- Define protocols for infrastructure
- Map errors between layers
- Handle data validation

**❌ DON'T:**
- Import Flutter packages
- Import concrete infrastructure implementations
- Put business logic here (belongs in Domain)
- Put UI logic here (belongs in Presentation/UI)
- Directly use external libraries (use protocols)

---

## Infrastructure Layer

### Purpose

The **Infrastructure Layer** provides **adapters** for external libraries and frameworks. It:
- Wraps third-party packages
- Implements Data layer protocols
- Isolates external dependencies
- Makes it easy to swap implementations

**Analogy:** These are the specific tools and equipment your construction crew uses - hammers, saws, drills. You can swap brands without changing how you build.

---

### Directory Structure

```
lib/infra/
├── http/
│   └── http_adapter.dart
└── cache/
    ├── secure_storage_adapter.dart
    └── local_storage_adapter.dart
```

---

### HTTP Adapter

**Example: HttpAdapter**

File: [lib/infra/http/http_adapter.dart](../lib/infra/http/http_adapter.dart)

```dart
import '../../data/http/http.dart';
import 'package:http/http.dart';
import 'dart:convert';

class HttpAdapter implements HttpClient {
  final Client client;

  HttpAdapter(this.client);

  Future<dynamic> request({
    required String url,
    required String method,
    Map? body,
    Map? headers
  }) async {
    final defaultHeaders = headers?.cast<String, String>() ?? {}..addAll({
      'content-type': 'application/json',
      'accept': 'application/json'
    });
    final jsonBody = body != null ? jsonEncode(body) : null;
    var response = Response('', 500);
    Future<Response>? futureResponse;

    try {
      if (method == 'post') {
        futureResponse = client.post(
          Uri.parse(url),
          headers: defaultHeaders,
          body: jsonBody
        );
      } else if (method == 'get') {
        futureResponse = client.get(
          Uri.parse(url),
          headers: defaultHeaders
        );
      } else if (method == 'put') {
        futureResponse = client.put(
          Uri.parse(url),
          headers: defaultHeaders,
          body: jsonBody
        );
      }
      if (futureResponse != null) {
        response = await futureResponse.timeout(Duration(seconds: 10));
      }
    } catch(error) {
      throw HttpError.serverError;
    }
    return _handleResponse(response);
  }

  dynamic _handleResponse(Response response) {
    switch (response.statusCode) {
      case 200: return response.body.isEmpty ? null : jsonDecode(response.body);
      case 204: return null;
      case 400: throw HttpError.badRequest;
      case 401: throw HttpError.unauthorized;
      case 403: throw HttpError.forbidden;
      case 404: throw HttpError.notFound;
      default: throw HttpError.serverError;
    }
  }
}
```

**Key responsibilities:**

1. **Implements `HttpClient` protocol** (from Data layer)
2. **Wraps `http` package** - uses `Client` from `package:http`
3. **Sets default headers** - JSON content-type and accept
4. **Serializes body** - converts Map to JSON string
5. **Handles HTTP methods** - POST, GET, PUT
6. **10-second timeout** - prevents hanging requests
7. **Maps status codes to errors** - 401 → unauthorized, etc.
8. **Deserializes response** - JSON string to Map

**Adapter Pattern in action:**
```dart
// Data layer depends on abstraction
abstract class HttpClient {
  Future<dynamic> request(...);
}

// Infrastructure provides concrete implementation
class HttpAdapter implements HttpClient {
  final http.Client client; // Wraps external library
}

// Easy to swap!
class DioAdapter implements HttpClient {
  final Dio dio; // Different library, same interface
}
```

---

### Cache Adapters

**Example: SecureStorageAdapter**

File: [lib/infra/cache/secure_storage_adapter.dart](../lib/infra/cache/secure_storage_adapter.dart)

```dart
import '../../data/cache/cache.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class SecureStorageAdapter implements
  SaveSecureCacheStorage,
  FetchSecureCacheStorage,
  DeleteSecureCacheStorage {

  final FlutterSecureStorage secureStorage;

  SecureStorageAdapter({ required this.secureStorage });

  Future<void> save({ required String key, required String value }) async {
    await secureStorage.write(key: key, value: value);
  }

  Future<String?> fetch(String key) async {
    return await secureStorage.read(key: key);
  }

  Future<void> delete(String key) async {
    await secureStorage.delete(key: key);
  }
}
```

**Key points:**

1. **Implements THREE protocols** - Save, Fetch, Delete
2. **Wraps `flutter_secure_storage`** package
3. **Simple pass-through** - delegates to wrapped library
4. **Type-safe** - enforces String keys and values

**Usage:**
```dart
// For sensitive data (tokens, passwords)
final adapter = SecureStorageAdapter(
  secureStorage: FlutterSecureStorage()
);

await adapter.save(key: 'token', value: 'abc123');
final token = await adapter.fetch('token');
await adapter.delete('token');
```

---

**Example: LocalStorageAdapter**

File: [lib/infra/cache/local_storage_adapter.dart](../lib/infra/cache/local_storage_adapter.dart)

```dart
import '../../data/cache/cache.dart';
import 'package:localstorage/localstorage.dart';
import 'dart:convert';

class LocalStorageAdapter implements CacheStorage {
  final LocalStorage localStorage;

  LocalStorageAdapter({ required this.localStorage });

  Future<void> save({ required String key, required dynamic value }) async {
    await localStorage.deleteItem(key);
    await localStorage.setItem(key, value);
  }

  Future<dynamic> fetch(String key) async {
    return await localStorage.getItem(key);
  }

  Future<void> delete(String key) async {
    await localStorage.deleteItem(key);
  }
}
```

**Difference from SecureStorageAdapter:**
- **SecureStorage** - Encrypted, for sensitive data (tokens)
- **LocalStorage** - Unencrypted, for cacheable data (surveys)

**Usage:**
```dart
// For non-sensitive cached data
final adapter = LocalStorageAdapter(
  localStorage: LocalStorage('fordev')
);

await adapter.save(key: 'surveys', value: surveysJson);
final surveys = await adapter.fetch('surveys');
```

---

### Infrastructure Layer Rules

**✅ DO:**
- Implement Data layer protocols
- Wrap external libraries
- Handle framework-specific details
- Convert between external formats and protocol types

**❌ DON'T:**
- Contain business logic
- Import Domain layer directly
- Depend on other infrastructure implementations
- Import Presentation or UI layers

---

## Presentation Layer

### Purpose

The **Presentation Layer** contains the **logic for screens** - it's the middleman between UI and business logic. It:
- Coordinates user interactions
- Manages screen state
- Validates user input
- Transforms data for display
- Emits events to UI

**Analogy:** Presenters are like restaurant managers - they take orders from customers (UI), coordinate with the kitchen (use cases), and present the finished dishes back to customers.

---

### Directory Structure

```
lib/presentation/
├── presenters/
│   ├── getx_login_presenter.dart
│   ├── getx_signup_presenter.dart
│   ├── getx_splash_presenter.dart
│   ├── getx_surveys_presenter.dart
│   └── getx_survey_result_presenter.dart
├── protocols/
│   └── validation.dart
├── mixins/
│   ├── form_manager.dart
│   ├── loading_manager.dart
│   ├── navigation_manager.dart
│   ├── session_manager.dart
│   └── ui_error_manager.dart
└── helpers/
    └── survey_result_entity_extensions.dart
```

---

### Presenters

**What are presenters?**
Presenters implement the **business logic for a screen**. They:
- Receive user input from UI
- Call domain use cases
- Manage screen state
- Expose state via reactive streams
- Emit navigation events

**All presenters in this project:**
- Extend `GetxController` (for lifecycle management)
- Use **mixins** for common behaviors
- Expose **streams** for state changes
- Depend on **domain abstractions**

---

**Example: GetxLoginPresenter (Complete)**

File: [lib/presentation/presenters/getx_login_presenter.dart](../lib/presentation/presenters/getx_login_presenter.dart)

```dart
import '../../ui/helpers/helpers.dart';
import '../../ui/pages/pages.dart';
import '../../domain/helpers/helpers.dart';
import '../../domain/usecases/usecases.dart';
import '../protocols/protocols.dart';
import '../mixins/mixins.dart';

import 'package:get/get.dart';

class GetxLoginPresenter extends GetxController
  with LoadingManager, NavigationManager, FormManager, UIErrorManager
  implements LoginPresenter {

  final Validation validation;
  final Authentication authentication;
  final SaveCurrentAccount saveCurrentAccount;

  final _emailError = Rx<UIError?>(null);
  final _passwordError = Rx<UIError?>(null);

  String? _email;
  String? _password;

  Stream<UIError?> get emailErrorStream => _emailError.stream;
  Stream<UIError?> get passwordErrorStream => _passwordError.stream;

  GetxLoginPresenter({
    required this.validation,
    required this.authentication,
    required this.saveCurrentAccount
  });

  void validateEmail(String email) {
    _email = email;
    _emailError.value = _validateField('email');
    _validateForm();
  }

  void validatePassword(String password) {
    _password = password;
    _passwordError.value = _validateField('password');
    _validateForm();
  }

  UIError? _validateField(String field) {
    final formData = {
      'email': _email,
      'password': _password,
    };
    final error = validation.validate(field: field, input: formData);
    switch (error) {
      case ValidationError.invalidField: return UIError.invalidField;
      case ValidationError.requiredField: return UIError.requiredField;
      default: return null;
    }
  }

  void _validateForm() {
    isFormValid = _emailError.value == null
      && _passwordError.value == null
      && _email != null
      && _password != null;
  }

  Future<void> auth() async {
    try {
      mainError = null;
      isLoading = true;
      final account = await authentication.auth(
        AuthenticationParams(email: _email!, secret: _password!)
      );
      await saveCurrentAccount.save(account);
      navigateTo = '/surveys';
    } on DomainError catch (error) {
      switch (error) {
        case DomainError.invalidCredentials:
          mainError = UIError.invalidCredentials;
          break;
        default:
          mainError = UIError.unexpected;
          break;
      }
      isLoading = false;
    }
  }

  void goToSignUp() {
    navigateTo = '/signup';
  }
}
```

**Let's break this down:**

---

**1. Dependencies (Constructor Injection)**

```dart
final Validation validation;
final Authentication authentication;
final SaveCurrentAccount saveCurrentAccount;

GetxLoginPresenter({
  required this.validation,
  required this.authentication,
  required this.saveCurrentAccount
});
```

- All dependencies are **abstractions** (interfaces)
- Injected via constructor (Dependency Injection)
- Presenter doesn't create dependencies (Single Responsibility)

---

**2. State Management with Reactive Streams**

```dart
final _emailError = Rx<UIError?>(null);
final _passwordError = Rx<UIError?>(null);

Stream<UIError?> get emailErrorStream => _emailError.stream;
Stream<UIError?> get passwordErrorStream => _passwordError.stream;
```

- `Rx<T>` is a GetX reactive variable
- `_emailError.value = newValue` triggers stream update
- UI listens to `.stream` and rebuilds when values change

**Pattern: Unidirectional Data Flow**
```
User Input → Presenter Method → Update Rx Variable → Stream Emits → UI Rebuilds
```

---

**3. Mixins for Common Behaviors**

```dart
class GetxLoginPresenter extends GetxController
  with LoadingManager, NavigationManager, FormManager, UIErrorManager
```

**What each mixin provides:**

- **LoadingManager** - `isLoading` state
- **NavigationManager** - `navigateTo` events
- **FormManager** - `isFormValid` state
- **UIErrorManager** - `mainError` state

**Mixin example: FormManager**

File: [lib/presentation/mixins/form_manager.dart](../lib/presentation/mixins/form_manager.dart)

```dart
import 'package:get/get.dart';

mixin FormManager on GetxController {
  final _isFormValid = false.obs;
  Stream<bool> get isFormValidStream => _isFormValid.stream;

  set isFormValid(bool value) => _isFormValid.value = value;
}
```

**Benefits:**
- Code reuse across all presenters
- Consistent state management patterns
- Easy to test (mock mixins)

---

**4. Input Validation**

```dart
void validateEmail(String email) {
  _email = email;
  _emailError.value = _validateField('email');
  _validateForm();
}

UIError? _validateField(String field) {
  final formData = {'email': _email, 'password': _password};
  final error = validation.validate(field: field, input: formData);
  switch (error) {
    case ValidationError.invalidField: return UIError.invalidField;
    case ValidationError.requiredField: return UIError.requiredField;
    default: return null;
  }
}
```

**Flow:**
1. UI calls `validateEmail(email)`
2. Presenter stores email
3. Calls validation service
4. Maps ValidationError → UIError
5. Updates stream (UI rebuilds)

---

**5. Form Validation**

```dart
void _validateForm() {
  isFormValid = _emailError.value == null
    && _passwordError.value == null
    && _email != null
    && _password != null;
}
```

**Form is valid when:**
- No email error
- No password error
- Both fields have values

**UI can disable login button:**
```dart
StreamBuilder<bool>(
  stream: presenter.isFormValidStream,
  builder: (context, snapshot) {
    return ElevatedButton(
      onPressed: snapshot.data == true ? presenter.auth : null,
      child: Text('Login'),
    );
  },
)
```

---

**6. Business Operation (Authentication)**

```dart
Future<void> auth() async {
  try {
    mainError = null;
    isLoading = true;
    final account = await authentication.auth(
      AuthenticationParams(email: _email!, secret: _password!)
    );
    await saveCurrentAccount.save(account);
    navigateTo = '/surveys';
  } on DomainError catch (error) {
    switch (error) {
      case DomainError.invalidCredentials:
        mainError = UIError.invalidCredentials;
        break;
      default:
        mainError = UIError.unexpected;
        break;
    }
    isLoading = false;
  }
}
```

**Flow:**
1. Clear previous error
2. Show loading (UI shows spinner)
3. Call authentication use case
4. Save account (token) locally
5. Navigate to surveys page
6. On error: Map DomainError → UIError, hide loading

**Error mapping:**
```
DomainError.invalidCredentials → UIError.invalidCredentials
DomainError.* → UIError.unexpected
```

---

**7. Navigation**

```dart
void goToSignUp() {
  navigateTo = '/signup';
}
```

**From `auth()` method:**
```dart
navigateTo = '/surveys';
```

**How it works:**
- `navigateTo` is from NavigationManager mixin
- Setting value emits event on stream
- UI mixin listens and navigates

---

### Presentation Mixins Deep Dive

**LoadingManager**

File: [lib/presentation/mixins/loading_manager.dart](../lib/presentation/mixins/loading_manager.dart)

```dart
import 'package:get/get.dart';

mixin LoadingManager on GetxController {
  final _isLoading = false.obs;
  Stream<bool> get isLoadingStream => _isLoading.stream;

  set isLoading(bool value) => _isLoading.value = value;
}
```

**Usage:**
```dart
isLoading = true; // Show spinner
await longOperation();
isLoading = false; // Hide spinner
```

---

**NavigationManager**

File: [lib/presentation/mixins/navigation_manager.dart](../lib/presentation/mixins/navigation_manager.dart)

```dart
import 'package:get/get.dart';

mixin NavigationManager on GetxController {
  final _navigateTo = RxString('');
  Stream<String> get navigateToStream => _navigateTo.stream.where((route) => route.isNotEmpty);

  set navigateTo(String value) => _navigateTo.value = value;
}
```

**Usage:**
```dart
navigateTo = '/surveys'; // Triggers navigation
```

---

**UIErrorManager**

File: [lib/presentation/mixins/ui_error_manager.dart](../lib/presentation/mixins/ui_error_manager.dart)

```dart
import '../../ui/helpers/helpers.dart';
import 'package:get/get.dart';

mixin UIErrorManager on GetxController {
  final _mainError = Rx<UIError?>(null);
  Stream<UIError?> get mainErrorStream => _mainError.stream;

  set mainError(UIError? value) => _mainError.value = value;
}
```

**Usage:**
```dart
mainError = UIError.invalidCredentials; // Show error message
```

---

### Presentation Layer Rules

**✅ DO:**
- Implement presenter interfaces
- Use reactive streams for state
- Depend on domain abstractions
- Map domain errors to UI errors
- Use mixins for common behaviors

**❌ DON'T:**
- Import concrete use case implementations
- Contain business logic (that's Domain's job)
- Import Flutter widgets (use protocols)
- Directly access infrastructure layer

---

## UI Layer

### Purpose

The **UI Layer** contains **visual components** - everything the user sees and interacts with. It:
- Displays data from presenters
- Captures user input
- Listens to presenter streams
- Rebuilds when state changes
- Contains zero business logic

**Analogy:** The UI is the dining room of a restaurant - beautifully decorated, welcoming to customers, but all the real work happens in the kitchen (presenters/use cases).

---

### Directory Structure

```
lib/ui/
├── pages/
│   ├── splash/
│   ├── login/
│   ├── signup/
│   ├── surveys/
│   └── survey_result/
├── components/
│   ├── app.dart
│   ├── error_message.dart
│   ├── headline1.dart
│   ├── login_header.dart
│   ├── reload_screen.dart
│   └── spinner_dialog.dart
├── helpers/
│   ├── errors/
│   │   └── ui_error.dart
│   └── i18n/
│       ├── resources.dart
│       └── strings/
│           └── pt_br.dart
└── mixins/
    ├── keyboard_manager.dart
    ├── loading_manager.dart
    ├── navigation_manager.dart
    ├── session_manager.dart
    └── ui_error_manager.dart
```

---

### Pages

**Example: LoginPage (Complete)**

File: [lib/ui/pages/login/login_page.dart](../lib/ui/pages/login/login_page.dart)

```dart
import '../../components/components.dart';
import '../../helpers/helpers.dart';
import '../../mixins/mixins.dart';
import './components/components.dart';
import './login.dart';

import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

class LoginPage extends StatelessWidget
  with KeyboardManager, LoadingManager, UIErrorManager, NavigationManager {

  final LoginPresenter presenter;

  LoginPage(this.presenter);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Builder(
        builder: (context) {
          handleLoading(context, presenter.isLoadingStream);
          handleMainError(context, presenter.mainErrorStream);
          handleNavigation(presenter.navigateToStream, clear: true);

          return GestureDetector(
            onTap: () => hideKeyboard(context),
            child: SingleChildScrollView(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.stretch,
                children: <Widget>[
                  LoginHeader(),
                  Headline1(text: R.string.login),
                  Padding(
                    padding: EdgeInsets.all(32),
                    child: ListenableProvider(
                      create: (_) => presenter,
                      child: Form(
                        child: Column(
                          children: <Widget>[
                            EmailInput(),
                            Padding(
                              padding: EdgeInsets.only(top: 8, bottom: 32),
                              child: PasswordInput(),
                            ),
                            LoginButton(),
                            TextButton.icon(
                              onPressed: presenter.goToSignUp,
                              icon: Icon(Icons.person),
                              label: Text(R.string.addAccount)
                            )
                          ],
                        ),
                      ),
                    ),
                  )
                ],
              ),
            ),
          );
        },
      ),
    );
  }
}
```

**Key points:**

---

**1. UI Mixins**

```dart
with KeyboardManager, LoadingManager, UIErrorManager, NavigationManager
```

**What each provides:**

- **KeyboardManager** - `hideKeyboard(context)` method
- **LoadingManager** - `handleLoading(context, stream)` method
- **UIErrorManager** - `handleMainError(context, stream)` method
- **NavigationManager** - `handleNavigation(stream)` method

---

**2. Stream Listeners (Side Effects)**

```dart
handleLoading(context, presenter.isLoadingStream);
handleMainError(context, presenter.mainErrorStream);
handleNavigation(presenter.navigateToStream, clear: true);
```

**These are called on every build** but use stream listeners internally to trigger side effects:
- Show/hide loading dialog
- Show error snackbar
- Navigate to new page

**LoadingManager Mixin:**

File: [lib/ui/mixins/loading_manager.dart](../lib/ui/mixins/loading_manager.dart)

```dart
import '../components/components.dart';
import 'package:flutter/material.dart';

mixin LoadingManager {
  void handleLoading(BuildContext context, Stream<bool> stream) {
    stream.listen((isLoading) {
      if (isLoading) {
        showDialog(
          context: context,
          barrierDismissible: false,
          builder: (context) => SpinnerDialog(),
        );
      } else {
        if (Navigator.canPop(context)) {
          Navigator.of(context).pop();
        }
      }
    });
  }
}
```

---

**3. Provider for Presenter Access**

```dart
ListenableProvider(
  create: (_) => presenter,
  child: Form(...)
)
```

**Why Provider?**
- Makes presenter available to child widgets
- Child widgets can access via `Provider.of<LoginPresenter>(context)`
- Avoids passing presenter through constructor chain

---

**4. Child Components**

```dart
EmailInput(),
PasswordInput(),
LoginButton(),
```

**Each component is a separate widget** that accesses presenter from Provider.

**Example: EmailInput**

File: [lib/ui/pages/login/components/email_input.dart](../lib/ui/pages/login/components/email_input.dart)

```dart
import '../../../helpers/helpers.dart';
import '../login.dart';

import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

class EmailInput extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final presenter = Provider.of<LoginPresenter>(context);

    return StreamBuilder<UIError?>(
      stream: presenter.emailErrorStream,
      builder: (context, snapshot) {
        return TextFormField(
          decoration: InputDecoration(
            labelText: R.string.email,
            errorText: snapshot.hasData ? snapshot.data?.description : null,
            icon: Icon(Icons.email),
          ),
          keyboardType: TextInputType.emailAddress,
          onChanged: presenter.validateEmail,
        );
      },
    );
  }
}
```

**Flow:**
1. Get presenter from Provider
2. Listen to `emailErrorStream`
3. Show error message if present
4. Call `presenter.validateEmail` on change

---

**Example: LoginButton**

File: [lib/ui/pages/login/components/login_button.dart](../lib/ui/pages/login/components/login_button.dart)

```dart
import '../../../helpers/helpers.dart';
import '../login.dart';

import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

class LoginButton extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final presenter = Provider.of<LoginPresenter>(context);

    return StreamBuilder<bool>(
      stream: presenter.isFormValidStream,
      builder: (context, snapshot) {
        return ElevatedButton(
          onPressed: snapshot.data == true ? presenter.auth : null,
          child: Text(R.string.enter.toUpperCase()),
        );
      },
    );
  }
}
```

**Smart button:**
- Disabled when form invalid (`onPressed: null`)
- Enabled when form valid
- Calls `presenter.auth()` when tapped

---

### UI Components

**Reusable widgets used across pages.**

**Example: Headline1**

File: [lib/ui/components/headline1.dart](../lib/ui/components/headline1.dart)

```dart
import 'package:flutter/material.dart';

class Headline1 extends StatelessWidget {
  final String text;

  const Headline1({required this.text});

  @override
  Widget build(BuildContext context) {
    return Text(
      text,
      textAlign: TextAlign.center,
      style: Theme.of(context).textTheme.headline1,
    );
  }
}
```

---

**Example: SpinnerDialog**

File: [lib/ui/components/spinner_dialog.dart](../lib/ui/components/spinner_dialog.dart)

```dart
import 'package:flutter/material.dart';

class SpinnerDialog extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Dialog(
      child: Padding(
        padding: EdgeInsets.all(16),
        child: Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            CircularProgressIndicator(),
            SizedBox(width: 16),
            Text('Aguarde...'),
          ],
        ),
      ),
    );
  }
}
```

---

### UI Helpers

**UIError Enum**

File: [lib/ui/helpers/errors/ui_error.dart](../lib/ui/helpers/errors/ui_error.dart)

```dart
import '../i18n/i18n.dart';

enum UIError {
  requiredField,
  invalidField,
  invalidCredentials,
  emailInUse,
  unexpected
}

extension UIErrorExtensions on UIError {
  String get description {
    switch (this) {
      case UIError.requiredField: return R.string.msgRequiredField;
      case UIError.invalidField: return R.string.msgInvalidField;
      case UIError.invalidCredentials: return R.string.msgInvalidCredentials;
      case UIError.emailInUse: return R.string.msgEmailInUse;
      default: return R.string.msgUnexpectedError;
    }
  }
}
```

**Usage:**
```dart
final error = UIError.invalidCredentials;
print(error.description); // "Credenciais inválidas"
```

---

### Internationalization (i18n)

**Resource System**

File: [lib/ui/helpers/i18n/resources.dart](../lib/ui/helpers/i18n/resources.dart)

```dart
import 'strings/strings.dart';

class R {
  static Translations string = PtBr();
}
```

**Translation Interface**

File: [lib/ui/helpers/i18n/strings/translations.dart](../lib/ui/helpers/i18n/strings/translations.dart)

```dart
abstract class Translations {
  String get msgEmailInUse;
  String get msgInvalidCredentials;
  String get msgInvalidField;
  String get msgRequiredField;
  String get msgUnexpectedError;
  // ... more strings
}
```

**Portuguese Implementation**

File: [lib/ui/helpers/i18n/strings/pt_br.dart](../lib/ui/helpers/i18n/strings/pt_br.dart)

```dart
class PtBr implements Translations {
  String get msgEmailInUse => 'Esse e-mail já está em uso.';
  String get msgInvalidCredentials => 'Credenciais inválidas.';
  String get msgInvalidField => 'Campo inválido';
  String get msgRequiredField => 'Campo obrigatório';
  String get msgUnexpectedError => 'Algo errado aconteceu. Tente novamente em breve.';
  // ... more strings
}
```

**Usage:**
```dart
import 'helpers/i18n/i18n.dart';

Text(R.string.login);
Text(R.string.msgInvalidCredentials);
```

**Adding a new language:**
```dart
// 1. Create en_us.dart
class EnUs implements Translations {
  String get msgEmailInUse => 'This email is already in use.';
  // ...
}

// 2. Update resources.dart
class R {
  static Translations string = EnUs(); // or PtBr()
}
```

---

### UI Layer Rules

**✅ DO:**
- Display data from presenters
- Listen to streams and rebuild
- Capture user input
- Use i18n for all user-facing strings
- Extract reusable components

**❌ DON'T:**
- Contain business logic
- Call use cases directly
- Import Data or Infrastructure layers
- Make HTTP requests or database calls
- Perform calculations or transformations

**The UI should be "dumb"** - it only displays what presenters tell it to display.

---

## Validation Layer

### Purpose

The **Validation Layer** contains **form validation logic** as a separate concern. It:
- Validates form fields
- Returns validation errors
- Is completely independent (can be used anywhere)
- Supports composition (multiple validators per field)

**Analogy:** Validators are like quality control inspectors - each checks for one specific issue.

---

### Directory Structure

```
lib/validation/
├── validators/
│   ├── required_field_validation.dart
│   ├── email_validation.dart
│   ├── min_length_validation.dart
│   └── compare_fields_validation.dart
└── protocols/
    ├── field_validation.dart
    └── validation_error.dart
```

---

### Validation Protocols

**FieldValidation Interface**

File: [lib/validation/protocols/field_validation.dart](../lib/validation/protocols/field_validation.dart)

```dart
import '../../presentation/protocols/protocols.dart';

abstract class FieldValidation {
  String get field;
  ValidationError? validate(Map input);
}
```

**All validators implement this interface.**

---

**ValidationError Enum**

File: [lib/presentation/protocols/validation.dart](../lib/presentation/protocols/validation.dart)

```dart
enum ValidationError {
  requiredField,
  invalidField
}
```

---

### Validators

**Example: RequiredFieldValidation**

File: [lib/validation/validators/required_field_validation.dart](../lib/validation/validators/required_field_validation.dart)

```dart
import '../../presentation/protocols/protocols.dart';
import '../protocols/protocols.dart';

import 'package:equatable/equatable.dart';

class RequiredFieldValidation extends Equatable implements FieldValidation {
  final String field;

  List get props => [field];

  RequiredFieldValidation(this.field);

  ValidationError? validate(Map input) =>
    input[field]?.isNotEmpty == true ? null : ValidationError.requiredField;
}
```

**Logic:**
- Field value exists and is not empty? → `null` (valid)
- Otherwise → `ValidationError.requiredField`

**Usage:**
```dart
final validator = RequiredFieldValidation('email');
final error = validator.validate({'email': ''});
print(error); // ValidationError.requiredField

final error2 = validator.validate({'email': 'test@test.com'});
print(error2); // null (valid)
```

---

**Example: EmailValidation**

File: [lib/validation/validators/email_validation.dart](../lib/validation/validators/email_validation.dart)

```dart
import '../../presentation/protocols/protocols.dart';
import '../protocols/protocols.dart';

import 'package:equatable/equatable.dart';

class EmailValidation extends Equatable implements FieldValidation {
  final String field;

  List get props => [field];

  EmailValidation(this.field);

  ValidationError? validate(Map input) {
    final regex = RegExp(
      r"^[a-zA-Z0-9.a-zA-Z0-9.!#$%&'*+-/=?^_`{|}~]+@[a-zA-Z0-9]+\.[a-zA-Z]+"
    );
    final isValid = input[field]?.isNotEmpty != true || regex.hasMatch(input[field]);
    return isValid ? null : ValidationError.invalidField;
  }
}
```

**Logic:**
- Field is empty? → `null` (valid, because required is separate concern)
- Field matches email regex? → `null` (valid)
- Otherwise → `ValidationError.invalidField`

---

**Example: MinLengthValidation**

File: [lib/validation/validators/min_length_validation.dart](../lib/validation/validators/min_length_validation.dart)

```dart
import '../../presentation/protocols/protocols.dart';
import '../protocols/protocols.dart';

import 'package:equatable/equatable.dart';

class MinLengthValidation extends Equatable implements FieldValidation {
  final String field;
  final int size;

  List get props => [field, size];

  MinLengthValidation({required this.field, required this.size});

  ValidationError? validate(Map input) {
    final value = input[field] as String?;
    return value != null && value.length >= size
      ? null
      : ValidationError.invalidField;
  }
}
```

**Usage:**
```dart
final validator = MinLengthValidation(field: 'password', size: 3);
validator.validate({'password': 'ab'}); // ValidationError.invalidField
validator.validate({'password': 'abc'}); // null (valid)
```

---

**Example: CompareFieldsValidation**

File: [lib/validation/validators/compare_fields_validation.dart](../lib/validation/validators/compare_fields_validation.dart)

```dart
import '../../presentation/protocols/protocols.dart';
import '../protocols/protocols.dart';

import 'package:equatable/equatable.dart';

class CompareFieldsValidation extends Equatable implements FieldValidation {
  final String field;
  final String fieldToCompare;

  List get props => [field, fieldToCompare];

  CompareFieldsValidation({
    required this.field,
    required this.fieldToCompare
  });

  ValidationError? validate(Map input) {
    return input[field] != null &&
           input[fieldToCompare] != null &&
           input[field] == input[fieldToCompare]
      ? null
      : ValidationError.invalidField;
  }
}
```

**Usage:**
```dart
final validator = CompareFieldsValidation(
  field: 'passwordConfirmation',
  fieldToCompare: 'password'
);

validator.validate({
  'password': 'abc123',
  'passwordConfirmation': 'abc123'
}); // null (valid)

validator.validate({
  'password': 'abc123',
  'passwordConfirmation': 'different'
}); // ValidationError.invalidField
```

---

### Validation Composition

**ValidationComposite** (in Main layer)

File: [lib/main/composites/validation_composite.dart](../lib/main/composites/validation_composite.dart)

```dart
import '../../presentation/protocols/protocols.dart';
import '../../validation/protocols/protocols.dart';

class ValidationComposite implements Validation {
  final List<FieldValidation> validations;

  ValidationComposite(this.validations);

  ValidationError? validate({required String field, required Map input}) {
    ValidationError? error;
    for (final validation in validations.where((v) => v.field == field)) {
      error = validation.validate(input);
      if (error != null) {
        return error;
      }
    }
    return error;
  }
}
```

**How it works:**
1. Receives list of all validators
2. When validating a field, filters validators for that field
3. Runs each validator in order
4. Returns first error found (or null if all pass)

**Usage:**
```dart
final composite = ValidationComposite([
  RequiredFieldValidation('email'),
  EmailValidation('email'),
  RequiredFieldValidation('password'),
  MinLengthValidation(field: 'password', size: 3),
]);

// Validates email field
composite.validate(
  field: 'email',
  input: {'email': '', 'password': 'abc'}
); // ValidationError.requiredField

composite.validate(
  field: 'email',
  input: {'email': 'invalid', 'password': 'abc'}
); // ValidationError.invalidField

composite.validate(
  field: 'email',
  input: {'email': 'test@test.com', 'password': 'abc'}
); // null (valid)
```

---

### Validation Layer Rules

**✅ DO:**
- Create single-purpose validators
- Extend Equatable (for testing)
- Keep validators pure (no side effects)
- Return null for valid, error for invalid

**❌ DON'T:**
- Import Flutter packages
- Import UI, Presentation, Data, or Infra layers
- Perform side effects (logging, analytics, etc.)
- Depend on external services

---

## Main Layer (Composition Root)

### Purpose

The **Main Layer** is where **everything comes together**. It:
- Creates all dependencies
- Wires them together (Dependency Injection)
- Implements design patterns (Factory, Builder, Decorator, Composite)
- Is the ONLY layer that knows about all other layers

**Analogy:** This is like HR in a company - it knows about all departments and assigns the right people to the right jobs.

---

### Directory Structure

```
lib/main/
├── factories/
│   ├── pages/
│   │   ├── login/
│   │   │   ├── login_page_factory.dart
│   │   │   ├── login_presenter_factory.dart
│   │   │   └── login_validation_factory.dart
│   │   ├── signup/
│   │   ├── splash/
│   │   ├── surveys/
│   │   └── survey_result/
│   ├── usecases/
│   │   ├── authentication_factory.dart
│   │   ├── save_current_account_factory.dart
│   │   ├── load_current_account_factory.dart
│   │   └── ... (other use cases)
│   ├── http/
│   │   ├── api_url_factory.dart
│   │   ├── http_client_factory.dart
│   │   └── authorize_http_client_decorator_factory.dart
│   └── cache/
│       ├── secure_storage_adapter_factory.dart
│       └── local_storage_adapter_factory.dart
├── builders/
│   └── validation_builder.dart
├── composites/
│   ├── validation_composite.dart
│   ├── remote_load_surveys_with_local_fallback.dart
│   └── remote_load_survey_result_with_local_fallback.dart
├── decorators/
│   └── authorize_http_client_decorator.dart
└── main.dart
```

---

### Factory Pattern

**Factories create and configure objects.**

**Example: HTTP Client Factory**

File: [lib/main/factories/http/http_client_factory.dart](../lib/main/factories/http/http_client_factory.dart)

```dart
import '../../../data/http/http.dart';
import '../../../infra/http/http.dart';

import 'package:http/http.dart';

HttpClient makeHttpAdapter() {
  final client = Client();
  return HttpAdapter(client);
}
```

**Simple factory:**
- Creates `http.Client`
- Wraps in `HttpAdapter`
- Returns as `HttpClient` interface

---

**Example: Remote Authentication Factory**

File: [lib/main/factories/usecases/authentication_factory.dart](../lib/main/factories/usecases/authentication_factory.dart)

```dart
import '../../../domain/usecases/usecases.dart';
import '../../../data/usecases/usecases.dart';
import '../http/http.dart';

Authentication makeRemoteAuthentication() {
  return RemoteAuthentication(
    httpClient: makeAuthorizeHttpClientDecorator(),
    url: makeApiUrl('login'),
  );
}
```

**Composed factory:**
- Calls other factories for dependencies
- Configures with specific URL
- Returns as domain interface

---

**Example: Login Presenter Factory**

File: [lib/main/factories/pages/login/login_presenter_factory.dart](../lib/main/factories/pages/login/login_presenter_factory.dart)

```dart
import '../../../../presentation/presenters/presenters.dart';
import '../../../../ui/pages/pages.dart';
import '../../factories.dart';

LoginPresenter makeGetxLoginPresenter() => GetxLoginPresenter(
  authentication: makeRemoteAuthentication(),
  validation: makeLoginValidation(),
  saveCurrentAccount: makeLocalSaveCurrentAccount()
);
```

**High-level factory:**
- Composes multiple dependencies
- Returns presenter ready to use
- All dependencies are abstractions

---

**Example: Login Page Factory**

File: [lib/main/factories/pages/login/login_page_factory.dart](../lib/main/factories/pages/login/login_page_factory.dart)

```dart
import '../../../../ui/pages/pages.dart';
import '../../factories.dart';

import 'package:flutter/material.dart';

Widget makeLoginPage() {
  return LoginPage(makeGetxLoginPresenter());
}
```

**Page factory:**
- Creates presenter
- Passes to page
- Returns Flutter widget

**Used in routing:**
```dart
GetMaterialApp(
  routes: {
    '/login': (context) => makeLoginPage(),
  },
)
```

---

### Builder Pattern

**ValidationBuilder** provides a **fluent API** for building validations.

File: [lib/main/builders/validation_builder.dart](../lib/main/builders/validation_builder.dart)

```dart
import '../../validation/protocols/protocols.dart';
import '../../validation/validators/validators.dart';

class ValidationBuilder {
  static ValidationBuilder? _instance;
  String fieldName;
  List<FieldValidation> validations = [];

  ValidationBuilder._(this.fieldName);

  static ValidationBuilder field(String fieldName) {
    _instance = ValidationBuilder._(fieldName);
    return _instance!;
  }

  ValidationBuilder required() {
    validations.add(RequiredFieldValidation(fieldName));
    return this;
  }

  ValidationBuilder email() {
    validations.add(EmailValidation(fieldName));
    return this;
  }

  ValidationBuilder min(int size) {
    validations.add(MinLengthValidation(field: fieldName, size: size));
    return this;
  }

  ValidationBuilder sameAs(String fieldToCompare) {
    validations.add(CompareFieldsValidation(
      field: fieldName,
      fieldToCompare: fieldToCompare
    ));
    return this;
  }

  List<FieldValidation> build() => validations;
}
```

**Fluent API pattern:**
```dart
ValidationBuilder.field('email')
  .required()
  .email()
  .build();

// Returns:
[
  RequiredFieldValidation('email'),
  EmailValidation('email'),
]
```

---

**Example: Login Validation Factory**

File: [lib/main/factories/pages/login/login_validation_factory.dart](../lib/main/factories/pages/login/login_validation_factory.dart)

```dart
import '../../../../presentation/protocols/protocols.dart';
import '../../../../validation/protocols/protocols.dart';
import '../../../builders/builders.dart';
import '../../../composites/composites.dart';

Validation makeLoginValidation() {
  return ValidationComposite(makeLoginValidations());
}

List<FieldValidation> makeLoginValidations() {
  return [
    ...ValidationBuilder.field('email').required().email().build(),
    ...ValidationBuilder.field('password').required().min(3).build(),
  ];
}
```

**Reads like English:**
- Email field is required and must be email
- Password field is required and minimum 3 characters

**Expands to:**
```dart
[
  RequiredFieldValidation('email'),
  EmailValidation('email'),
  RequiredFieldValidation('password'),
  MinLengthValidation(field: 'password', size: 3),
]
```

---

### Decorator Pattern

**AuthorizeHttpClientDecorator** adds authorization to HTTP requests.

File: [lib/main/decorators/authorize_http_client_decorator.dart](../lib/main/decorators/authorize_http_client_decorator.dart)

```dart
import '../../data/cache/cache.dart';
import '../../data/http/http.dart';

class AuthorizeHttpClientDecorator implements HttpClient {
  final FetchSecureCacheStorage fetchSecureCacheStorage;
  final DeleteSecureCacheStorage deleteSecureCacheStorage;
  final HttpClient decoratee;

  AuthorizeHttpClientDecorator({
    required this.fetchSecureCacheStorage,
    required this.deleteSecureCacheStorage,
    required this.decoratee,
  });

  Future<dynamic> request({
    required String url,
    required String method,
    Map? body,
    Map? headers,
  }) async {
    try {
      final token = await fetchSecureCacheStorage.fetch('token');
      final authorizedHeaders = headers ?? {}
        ..addAll({'x-access-token': token});
      return await decoratee.request(
        url: url,
        method: method,
        body: body,
        headers: authorizedHeaders,
      );
    } catch (error) {
      if (error == HttpError.forbidden) {
        await deleteSecureCacheStorage.delete('token');
      }
      rethrow;
    }
  }
}
```

**How it works:**
1. Fetch token from secure storage
2. Add token to headers (`x-access-token`)
3. Delegate to wrapped `HttpClient`
4. If response is 403 (forbidden), delete token and rethrow
5. Otherwise, rethrow any other errors

**Usage:**
```dart
HttpClient makeAuthorizeHttpClientDecorator() {
  return AuthorizeHttpClientDecorator(
    fetchSecureCacheStorage: makeSecureStorageAdapter(),
    deleteSecureCacheStorage: makeSecureStorageAdapter(),
    decoratee: makeHttpAdapter(), // Wraps base HTTP client
  );
}
```

**Decorator pattern benefits:**
- Adds behavior without modifying original class
- Can stack multiple decorators
- Follows Open/Closed Principle

---

### Composite Pattern

**RemoteLoadSurveysWithLocalFallback** tries remote first, falls back to local cache.

File: [lib/main/composites/remote_load_surveys_with_local_fallback.dart](../lib/main/composites/remote_load_surveys_with_local_fallback.dart)

```dart
import '../../domain/entities/entities.dart';
import '../../domain/helpers/helpers.dart';
import '../../domain/usecases/usecases.dart';
import '../../data/usecases/usecases.dart';

class RemoteLoadSurveysWithLocalFallback implements LoadSurveys {
  final RemoteLoadSurveys remote;
  final LocalLoadSurveys local;

  RemoteLoadSurveysWithLocalFallback({
    required this.remote,
    required this.local,
  });

  Future<List<SurveyEntity>> load() async {
    try {
      final surveys = await remote.load();
      await local.save(surveys);
      return surveys;
    } catch (error) {
      if (error == DomainError.accessDenied) {
        rethrow;
      }
      await local.validate();
      return await local.load();
    }
  }
}
```

**Flow:**
```
1. Try remote.load()
   ├─ Success:
   │   ├─ Save to local cache
   │   └─ Return surveys
   └─ Failure:
       ├─ If accessDenied: rethrow (session expired)
       └─ Otherwise:
           ├─ Validate local cache
           └─ Load from local cache
```

**Benefits:**
- Offline support
- Faster subsequent loads (cached)
- Graceful degradation

**Usage:**
```dart
LoadSurveys makeRemoteLoadSurveysWithLocalFallback() {
  return RemoteLoadSurveysWithLocalFallback(
    remote: makeRemoteLoadSurveys(),
    local: makeLocalLoadSurveys(),
  );
}
```

---

### Main Layer Rules

**✅ DO:**
- Create all objects here
- Wire dependencies together
- Know about all layers
- Use design patterns (Factory, Builder, Decorator, Composite)

**❌ DON'T:**
- Contain business logic
- Contain presentation logic
- Contain UI code
- This layer should be almost entirely factories and wiring

---

## Layer Interaction Patterns

### Request Flow Example: User Login

**Complete flow from button tap to navigation:**

```
1. UI Layer (LoginPage)
   └─ User taps "Login" button
   └─ Calls: presenter.auth()

2. Presentation Layer (GetxLoginPresenter)
   └─ Sets isLoading = true
   └─ Calls: authentication.auth(params)

3. Domain Layer (Authentication interface)
   └─ Defines contract

4. Data Layer (RemoteAuthentication)
   └─ Converts params to JSON
   └─ Calls: httpClient.request(...)

5. Infrastructure Layer (HttpAdapter via Decorator)
   └─ AuthorizeHttpClientDecorator:
       └─ Fetches token from secure storage
       └─ Adds to headers
   └─ HttpAdapter:
       └─ Makes actual HTTP request
       └─ Parses response

6. Back through Data Layer
   └─ RemoteAuthentication:
       └─ Converts JSON to RemoteAccountModel
       └─ Converts Model to AccountEntity
       └─ Returns to Presenter

7. Back to Presentation Layer
   └─ GetxLoginPresenter:
       └─ Calls: saveCurrentAccount.save(account)
       └─ Sets navigateTo = '/surveys'
       └─ Sets isLoading = false

8. UI Layer reacts
   └─ isLoadingStream emits: hide spinner
   └─ navigateToStream emits: navigate to surveys
```

---

### Error Flow Example: Invalid Credentials

```
1-5. Same as above

6. Infrastructure Layer
   └─ HTTP response: 401 Unauthorized
   └─ Throws: HttpError.unauthorized

7. Data Layer
   └─ Catches: HttpError.unauthorized
   └─ Throws: DomainError.invalidCredentials

8. Presentation Layer
   └─ Catches: DomainError.invalidCredentials
   └─ Sets mainError = UIError.invalidCredentials
   └─ Sets isLoading = false

9. UI Layer
   └─ mainErrorStream emits
   └─ Shows Snackbar: "Credenciais inválidas"
```

---

### Dependency Flow Summary

```
┌─────────────────────────────────────────────┐
│ MAIN creates everything:                    │
│                                             │
│ HttpAdapter httpAdapter = HttpAdapter(...)  │
│ ↓                                           │
│ AuthorizeHttpClientDecorator decorator =    │
│   AuthorizeHttpClientDecorator(httpAdapter) │
│ ↓                                           │
│ RemoteAuth auth = RemoteAuth(decorator)     │
│ ↓                                           │
│ Presenter presenter = Presenter(auth)       │
│ ↓                                           │
│ LoginPage page = LoginPage(presenter)       │
└─────────────────────────────────────────────┘

All dependencies flow from MAIN → UI
All calls flow from UI → Domain → Data → Infra
All data flows back: Infra → Data → Domain → Presentation → UI
```

---

## Key Takeaways

### Layer Responsibilities Summary

| Layer | What It Does | What It Doesn't Do |
|-------|--------------|-------------------|
| **Domain** | Defines business logic | Doesn't implement it |
| **Data** | Implements use cases | Doesn't make HTTP calls |
| **Infra** | Wraps external libs | Doesn't contain business logic |
| **Presentation** | Coordinates screen logic | Doesn't render UI |
| **UI** | Displays and captures | Doesn't process data |
| **Validation** | Validates input | Doesn't know about UI |
| **Main** | Wires everything | Doesn't contain logic |

---

### Dependency Rules Summary

**Golden Rule:** Dependencies point INWARD toward Domain

```
✅ UI → Presentation ✅
✅ Presentation → Domain ✅
✅ Data → Domain ✅
✅ Infra → Data ✅

❌ Domain → Anything ❌
❌ Data → Presentation ❌
❌ Infra → Presentation ❌
```

---

### Benefits Achieved

1. **Testability** - Each layer tests independently
2. **Maintainability** - Changes localized to one layer
3. **Flexibility** - Easy to swap implementations
4. **Clarity** - Clear responsibility for each layer
5. **Scalability** - Easy to add new features
6. **Team Collaboration** - Multiple developers work in parallel

---

## Next Steps

Now that you understand each layer in depth:

1. **[Design Patterns](03-design-patterns.md)** - Deep dive into patterns used
2. **[Component Deep-Dive](04-component-deep-dive.md)** - See complete features in action
3. **[Data Flow Guide](05-data-flow-guide.md)** - Follow data through the app
4. **[Getting Started Guide](06-getting-started-guide.md)** - Build your own feature
5. **[Testing Strategy](07-testing-strategy.md)** - Learn testing approaches

---

**Documentation Version:** 1.0
**Last Updated:** 2025

---

**Questions or suggestions?** Open an issue or contribute to improve this documentation!
