# 03 - Design Patterns

## Table of Contents

1. [Introduction](#introduction)
2. [What You'll Learn](#what-youll-learn)
3. [Pattern Categories](#pattern-categories)
4. [Creational Patterns](#creational-patterns)
5. [Structural Patterns](#structural-patterns)
6. [Behavioral Patterns](#behavioral-patterns)
7. [Architectural Patterns](#architectural-patterns)
8. [Pattern Combinations](#pattern-combinations)
9. [Pattern Benefits](#pattern-benefits)
10. [Key Takeaways](#key-takeaways)
11. [Next Steps](#next-steps)

---

## Introduction

This document is a **comprehensive catalog** of all design patterns used in the Clean Flutter App. Design patterns are proven solutions to common software design problems. They provide a shared vocabulary and best practices for building maintainable, scalable applications.

**Why patterns matter:**
Imagine trying to explain how to build a house without words like "door," "window," or "foundation." Design patterns are like architectural terms for software - they give us a common language to describe solutions.

---

## What You'll Learn

By the end of this document, you will understand:

- ✅ All 15+ design patterns used in the project
- ✅ When and why each pattern is used
- ✅ How each pattern is implemented with real code
- ✅ The benefits and trade-offs of each pattern
- ✅ How patterns work together in combination
- ✅ How to apply these patterns to your own projects

---

## Pattern Categories

Design patterns fall into three main categories:

### Pattern Classification

```
┌────────────────────────────────────────────────────────┐
│ CREATIONAL PATTERNS                                    │
│ Deal with object creation                              │
│ • Factory Method                                       │
│ • Builder                                              │
│ • Singleton                                            │
└────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────┐
│ STRUCTURAL PATTERNS                                    │
│ Deal with object composition                           │
│ • Adapter                                              │
│ • Decorator                                            │
│ • Composite                                            │
│ • Facade                                               │
└────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────┐
│ BEHAVIORAL PATTERNS                                    │
│ Deal with object interaction and responsibility        │
│ • Strategy                                             │
│ • Observer                                             │
│ • Template Method                                      │
│ • Command                                              │
└────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────┐
│ ARCHITECTURAL PATTERNS                                 │
│ Deal with overall application structure                │
│ • Dependency Injection                                 │
│ • Repository                                           │
│ • ViewModel                                            │
│ • Composition Root                                     │
└────────────────────────────────────────────────────────┘
```

---

## Creational Patterns

Creational patterns deal with **object creation mechanisms**, trying to create objects in a manner suitable to the situation.

---

### 1. Factory Method Pattern

**Intent:** Define an interface for creating objects, but let subclasses decide which class to instantiate.

**Problem It Solves:**
Creating objects directly (using `new`) couples your code to specific classes. If you need to change the implementation, you have to modify all creation sites.

**Analogy:**
Think of a car factory. You order a "car" but don't specify whether it's assembled in Factory A or Factory B. The factory manager (factory method) decides the details.

---

**Implementation in Project:**

The entire `main/factories/` directory uses Factory Method pattern extensively.

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

**How it works:**
- Function returns `HttpClient` (interface)
- Creates concrete `HttpAdapter` internally
- Callers don't know about `HttpAdapter` class

**Benefits:**
```dart
// Easy to swap implementations
HttpClient makeHttpAdapter() {
  return HttpAdapter(Client());
}

// Later, change to Dio without touching callers
HttpClient makeHttpAdapter() {
  return DioAdapter(Dio());
}
```

---

**Example: Page Factory**

File: [lib/main/factories/pages/login/login_page_factory.dart](../lib/main/factories/pages/login/login_page_factory.dart)

```dart
import '../../../../ui/pages/pages.dart';
import '../../factories.dart';
import 'package:flutter/material.dart';

Widget makeLoginPage() {
  return LoginPage(makeGetxLoginPresenter());
}
```

**Used in routing:**

File: [lib/main/main.dart](../lib/main/main.dart)

```dart
GetMaterialApp(
  routes: {
    '/login': (context) => makeLoginPage(),
    '/signup': (context) => makeSignUpPage(),
    '/surveys': (context) => makeSurveysPage(),
  },
)
```

**Why it's better than:**
```dart
// ❌ BAD - tightly coupled
'/login': (context) => LoginPage(
  GetxLoginPresenter(
    authentication: RemoteAuthentication(
      httpClient: HttpAdapter(Client()),
      url: 'http://api.com/login'
    ),
    validation: ValidationComposite([...]),
    saveCurrentAccount: LocalSaveCurrentAccount(...)
  )
),

// ✅ GOOD - decoupled via factory
'/login': (context) => makeLoginPage(),
```

---

**Example: Composed Factory (Dependencies)**

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

**Dependency chain:**
```
makeLoginPage()
  └─ makeGetxLoginPresenter()
       ├─ makeRemoteAuthentication()
       │    ├─ makeAuthorizeHttpClientDecorator()
       │    │    ├─ makeHttpAdapter()
       │    │    └─ makeSecureStorageAdapter()
       │    └─ makeApiUrl()
       ├─ makeLoginValidation()
       └─ makeLocalSaveCurrentAccount()
            └─ makeSecureStorageAdapter()
```

All created via factories!

---

**Pattern Structure:**

```
┌─────────────────────────────┐
│   Creator (Factory)         │
│                             │
│  + createProduct()          │
│    return ConcreteProduct   │
└─────────────────────────────┘
              │
              │ creates
              ↓
┌─────────────────────────────┐
│   Product (Interface)       │
└─────────────────────────────┘
              ▲
              │ implements
┌─────────────────────────────┐
│   ConcreteProduct           │
└─────────────────────────────┘
```

**Benefits:**
- ✅ Decouples client code from concrete classes
- ✅ Easy to swap implementations
- ✅ Single place to change object creation
- ✅ Follows Dependency Inversion Principle

**Trade-offs:**
- ⚠️ More code (factory functions)
- ⚠️ Indirection (one more level)

---

### 2. Builder Pattern

**Intent:** Construct complex objects step by step. The same construction process can create different representations.

**Problem It Solves:**
Creating complex objects with many parameters is error-prone and hard to read, especially when many parameters are optional.

**Analogy:**
Building a custom burger at a restaurant: "I'll have a burger with cheese, no pickles, extra lettuce, and BBQ sauce." Each step adds to the burger.

---

**Implementation in Project:**

**ValidationBuilder** - Fluent API for building validation rules.

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

---

**Usage Example:**

File: [lib/main/factories/pages/login/login_validation_factory.dart](../lib/main/factories/pages/login/login_validation_factory.dart)

```dart
List<FieldValidation> makeLoginValidations() {
  return [
    ...ValidationBuilder.field('email').required().email().build(),
    ...ValidationBuilder.field('password').required().min(3).build(),
  ];
}
```

**Reads like English:**
```dart
ValidationBuilder.field('email')
  .required()
  .email()
  .build()
```

Translates to: "For the email field, it's required and must be a valid email."

---

**Expanded result:**
```dart
[
  RequiredFieldValidation('email'),
  EmailValidation('email'),
]
```

---

**More complex example (SignUp):**

File: [lib/main/factories/pages/signup/signup_validation_factory.dart](../lib/main/factories/pages/signup/signup_validation_factory.dart)

```dart
List<FieldValidation> makeSignUpValidations() {
  return [
    ...ValidationBuilder.field('name').required().min(3).build(),
    ...ValidationBuilder.field('email').required().email().build(),
    ...ValidationBuilder.field('password').required().min(3).build(),
    ...ValidationBuilder.field('passwordConfirmation')
      .required()
      .sameAs('password')
      .build(),
  ];
}
```

**Without Builder pattern:**
```dart
// ❌ Harder to read and maintain
final validations = [
  RequiredFieldValidation('name'),
  MinLengthValidation(field: 'name', size: 3),
  RequiredFieldValidation('email'),
  EmailValidation('email'),
  RequiredFieldValidation('password'),
  MinLengthValidation(field: 'password', size: 3),
  RequiredFieldValidation('passwordConfirmation'),
  CompareFieldsValidation(
    field: 'passwordConfirmation',
    fieldToCompare: 'password'
  ),
];
```

---

**Pattern Structure:**

```
┌─────────────────────────────┐
│   Builder                   │
│                             │
│  + field(name)              │
│  + required()               │
│  + email()                  │
│  + min(size)                │
│  + build()                  │
└─────────────────────────────┘
              │
              │ builds
              ↓
┌─────────────────────────────┐
│   Product                   │
│   (List<FieldValidation>)   │
└─────────────────────────────┘
```

**Key characteristics:**
- **Fluent Interface** - methods return `this` for chaining
- **Step-by-step construction** - add validations one at a time
- **Final build()** - produces the final product

**Benefits:**
- ✅ Readable, expressive code
- ✅ Easy to add new validation types
- ✅ Flexible construction process
- ✅ Follows Open/Closed Principle

**Trade-offs:**
- ⚠️ More classes/code
- ⚠️ Overkill for simple objects

---

### 3. Singleton Pattern

**Intent:** Ensure a class has only one instance and provide a global point of access to it.

**Problem It Solves:**
Some objects should exist only once in the application (e.g., configuration, cache manager).

**Analogy:**
A country has only one president at a time. No matter where you are in the country, "the president" refers to the same person.

---

**Implementation in Project:**

GetX controllers are automatically singletons per scope when using `Get.find()` or `Get.put()`.

**Implicit Singleton (GetX):**

File: [lib/presentation/presenters/getx_login_presenter.dart](../lib/presentation/presenters/getx_login_presenter.dart:10)

```dart
class GetxLoginPresenter extends GetxController
  with LoadingManager, NavigationManager, FormManager, UIErrorManager
  implements LoginPresenter {
  // ...
}
```

**When registered with GetX:**
```dart
Get.put(GetxLoginPresenter(...));

// Anywhere in the app:
final presenter = Get.find<GetxLoginPresenter>(); // Same instance!
```

---

**Classic Singleton Pattern (if implemented manually):**

```dart
class ConfigManager {
  static ConfigManager? _instance;

  ConfigManager._(); // Private constructor

  static ConfigManager get instance {
    _instance ??= ConfigManager._();
    return _instance!;
  }
}

// Usage:
final config = ConfigManager.instance;
```

---

**Pattern Structure:**

```
┌─────────────────────────────┐
│   Singleton                 │
│                             │
│  - static instance          │
│  - Singleton._()            │ Private constructor
│  + static getInstance()     │
└─────────────────────────────┘
```

**Benefits:**
- ✅ Controlled access to single instance
- ✅ Reduced memory footprint
- ✅ Global point of access

**Trade-offs:**
- ⚠️ Harder to test (global state)
- ⚠️ Potential coupling
- ⚠️ Can hide dependencies

**Best practice:**
Use Dependency Injection instead of singletons when possible. Let the DI container manage instance lifecycle.

---

## Structural Patterns

Structural patterns deal with **object composition** - how objects and classes are composed to form larger structures.

---

### 4. Adapter Pattern

**Intent:** Convert the interface of a class into another interface clients expect. Adapter lets classes work together that couldn't otherwise due to incompatible interfaces.

**Problem It Solves:**
You want to use a third-party library, but its interface doesn't match what your code expects.

**Analogy:**
Using a power adapter when traveling abroad. Your phone charger (client) expects a US plug, but the wall socket (adaptee) is European. The adapter makes them compatible.

---

**Implementation in Project:**

All classes in the `infra/` layer are adapters!

---

**Example 1: HttpAdapter**

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
        futureResponse = client.get(Uri.parse(url), headers: defaultHeaders);
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

**What it adapts:**

```
┌─────────────────────────────┐
│   Target Interface          │
│   (What Data layer expects) │
│                             │
│   abstract class HttpClient │
│     request(...)            │
└─────────────────────────────┘
              ▲
              │ implements
┌─────────────────────────────┐
│   Adapter                   │
│   HttpAdapter               │
│                             │
│   + request(...)            │
│     calls adaptee methods   │
└─────────────────────────────┘
              │
              │ uses
              ↓
┌─────────────────────────────┐
│   Adaptee                   │
│   (http package's Client)   │
│                             │
│   + post(Uri, ...)          │
│   + get(Uri, ...)           │
│   + put(Uri, ...)           │
└─────────────────────────────┘
```

**Adaptations performed:**
1. **Interface** - `request(url, method, ...)` → `post(Uri, ...)`, `get(Uri, ...)`, etc.
2. **Parameters** - `String url` → `Uri.parse(url)`
3. **Headers** - Adds JSON content-type automatically
4. **Body** - `Map body` → `jsonEncode(body)`
5. **Errors** - Status codes → `HttpError` enum
6. **Response** - JSON string → `Map`

---

**Example 2: SecureStorageAdapter**

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

**What it adapts:**

```
┌─────────────────────────────┐
│   Target Interfaces         │
│   (Data layer expects)      │
│                             │
│   SaveSecureCacheStorage    │
│   FetchSecureCacheStorage   │
│   DeleteSecureCacheStorage  │
└─────────────────────────────┘
              ▲
              │ implements
┌─────────────────────────────┐
│   Adapter                   │
│   SecureStorageAdapter      │
└─────────────────────────────┘
              │
              │ uses
              ↓
┌─────────────────────────────┐
│   Adaptee                   │
│   FlutterSecureStorage      │
│                             │
│   + write(key, value)       │
│   + read(key)               │
│   + delete(key)             │
└─────────────────────────────┘
```

Simple pass-through adapter!

---

**Pattern Structure:**

```
Client → Target Interface ← Adapter → Adaptee
          (expects)       (implements)  (uses)
```

**Benefits:**
- ✅ Isolates external dependencies
- ✅ Easy to swap libraries
- ✅ Follows Dependency Inversion Principle
- ✅ Testable (mock the interface)

**Example of swapping:**
```dart
// Change from 'http' to 'dio' package
class DioAdapter implements HttpClient {
  final Dio dio;

  Future<dynamic> request(...) async {
    // Use dio instead of http package
  }
}

// Change factory - that's it!
HttpClient makeHttpAdapter() {
  return DioAdapter(Dio());
}
```

---

### 5. Decorator Pattern

**Intent:** Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.

**Problem It Solves:**
You want to add behavior to objects without modifying their class or creating a complex inheritance hierarchy.

**Analogy:**
Decorating a Christmas tree. The tree is the base, and you add lights, ornaments, and a star on top. Each decoration adds functionality without changing the tree itself.

---

**Implementation in Project:**

**AuthorizeHttpClientDecorator** - Adds authorization to HTTP requests.

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
        headers: authorizedHeaders
      );
    } catch(error) {
      if (error is HttpError && error != HttpError.forbidden) {
        rethrow;
      } else {
        await deleteSecureCacheStorage.delete('token');
        throw HttpError.forbidden;
      }
    }
  }
}
```

---

**How it works:**

1. **Implements same interface** as the object it decorates (`HttpClient`)
2. **Wraps the original object** (stored in `decoratee`)
3. **Adds behavior before/after** delegating to wrapped object
4. **Can be stacked** - decorators can decorate other decorators

---

**Decoration flow:**

```
Request comes in
    ↓
Decorator: Fetch token from storage
    ↓
Decorator: Add token to headers
    ↓
Decorator: Delegate to wrapped HttpClient (decoratee)
    ↓
Wrapped HttpClient: Makes actual HTTP request
    ↓
Response comes back
    ↓
Decorator: Check for 403 error
    ↓
If 403: Delete token and rethrow
Otherwise: Return response
```

---

**Pattern Structure:**

```
┌─────────────────────────────┐
│   Component (Interface)     │
│   HttpClient                │
│                             │
│   + request(...)            │
└─────────────────────────────┘
         ▲           ▲
         │           │
         │           │
┌────────┴────┐  ┌──┴─────────────────────┐
│ ConcreteComp│  │ Decorator              │
│ HttpAdapter │  │ AuthorizeHttpClient... │
└─────────────┘  │                        │
                 │ - decoratee: HttpClient│
                 │ + request(...)         │
                 │   • add behavior       │
                 │   • call decoratee     │
                 └────────────────────────┘
```

---

**Usage (Factory):**

File: [lib/main/factories/http/authorize_http_client_decorator_factory.dart](../lib/main/factories/http/authorize_http_client_decorator_factory.dart)

```dart
import '../../../data/http/http.dart';
import '../../../main/decorators/decorators.dart';
import '../factories.dart';

HttpClient makeAuthorizeHttpClientDecorator() {
  return AuthorizeHttpClientDecorator(
    fetchSecureCacheStorage: makeSecureStorageAdapter(),
    deleteSecureCacheStorage: makeSecureStorageAdapter(),
    decoratee: makeHttpAdapter(),
  );
}
```

**Decoration chain:**

```
RemoteAuthentication
  ↓ uses
AuthorizeHttpClientDecorator (adds token)
  ↓ wraps
HttpAdapter (makes HTTP request)
  ↓ uses
http.Client (external package)
```

---

**Stacking decorators (example):**

```dart
HttpClient makeLoggingHttpClient() {
  return LoggingDecorator(
    decoratee: AuthorizeHttpClientDecorator(
      decoratee: HttpAdapter(Client())
    )
  );
}

// Flow:
Client → LoggingDecorator → AuthorizeDecorator → HttpAdapter
         (logs requests)     (adds auth)          (makes request)
```

---

**Benefits:**
- ✅ Add behavior without modifying original class
- ✅ More flexible than inheritance
- ✅ Supports Open/Closed Principle
- ✅ Can stack multiple decorators
- ✅ Single Responsibility - each decorator does one thing

**Trade-offs:**
- ⚠️ Can result in many small objects
- ⚠️ Harder to debug (multiple layers)

---

### 6. Composite Pattern

**Intent:** Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions uniformly.

**Problem It Solves:**
You want to work with a group of objects the same way you work with a single object.

**Analogy:**
A military organization. A soldier is a component. A squad is a composite of soldiers. A platoon is a composite of squads. You can give an order to a soldier or a platoon - both respond to the same "execute order" interface.

---

**Implementation in Project:**

Two main composites:

1. **ValidationComposite** - Combines multiple validators
2. **RemoteLoadWithLocalFallback** - Combines remote and local data sources

---

**Example 1: ValidationComposite**

File: [lib/main/composites/validation_composite.dart](../lib/main/composites/validation_composite.dart)

```dart
import '../../presentation/protocols/protocols.dart';
import '../../validation/protocols/protocols.dart';

class ValidationComposite implements Validation {
  final List<FieldValidation> validations;

  ValidationComposite(this.validations);

  ValidationError? validate({ required String field, required Map input}) {
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

**Structure:**

```
┌─────────────────────────────┐
│   Component (Interface)     │
│   Validation                │
│   + validate(field, input)  │
└─────────────────────────────┘
         ▲           ▲
         │           │
┌────────┴────┐  ┌──┴─────────────────────┐
│   Leaf      │  │   Composite            │
│ FieldValid..│  │ ValidationComposite    │
│             │  │                        │
│ • Required  │  │ - validations: List    │
│ • Email     │  │ + validate(...)        │
│ • MinLength │  │   • iterate children   │
│ • Compare   │  │   • return first error │
└─────────────┘  └────────────────────────┘
```

**How it works:**

```dart
// Create individual validators (leaves)
final validators = [
  RequiredFieldValidation('email'),
  EmailValidation('email'),
  RequiredFieldValidation('password'),
  MinLengthValidation(field: 'password', size: 3),
];

// Wrap in composite
final composite = ValidationComposite(validators);

// Validate a field - composite delegates to matching validators
composite.validate(field: 'email', input: {'email': ''});
// Runs:
//  1. RequiredFieldValidation('email').validate(...) → error!
//  2. Returns error immediately (short-circuits)
```

---

**Example 2: RemoteLoadSurveysWithLocalFallback**

File: [lib/main/composites/remote_load_surveys_with_local_fallback.dart](../lib/main/composites/remote_load_surveys_with_local_fallback.dart)

```dart
import '../../data/usecases/usecases.dart';
import '../../domain/entities/entities.dart';
import '../../domain/helpers/helpers.dart';
import '../../domain/usecases/usecases.dart';

class RemoteLoadSurveysWithLocalFallback implements LoadSurveys {
  final RemoteLoadSurveys remote;
  final LocalLoadSurveys local;

  RemoteLoadSurveysWithLocalFallback({
    required this.remote,
    required this.local
  });

  Future<List<SurveyEntity>> load() async {
    try {
      final surveys = await remote.load();
      await local.save(surveys);
      return surveys;
    } catch(error) {
      if (error == DomainError.accessDenied) {
        rethrow;
      }
      await local.validate();
      return await local.load();
    }
  }
}
```

**Structure:**

```
┌─────────────────────────────┐
│   Component (Interface)     │
│   LoadSurveys               │
│   + load()                  │
└─────────────────────────────┘
         ▲           ▲
         │           │
┌────────┴────┐  ┌──┴─────────────────────────┐
│   Leaf      │  │   Composite                │
│ RemoteLoad..│  │ RemoteLoadWith...Fallback  │
│ LocalLoad...│  │                            │
└─────────────┘  │ - remote: RemoteLoadSurveys│
                 │ - local: LocalLoadSurveys  │
                 │ + load()                   │
                 │   • try remote             │
                 │   • on fail, use local     │
                 └────────────────────────────┘
```

**How it works:**

```
User calls: composite.load()
    ↓
Try: remote.load()
  ├─ Success:
  │   ├─ Save to cache: local.save(surveys)
  │   └─ Return surveys
  └─ Failure:
      ├─ If accessDenied: rethrow (session expired)
      └─ Otherwise:
          ├─ Validate cache: local.validate()
          └─ Load from cache: local.load()
```

**Benefits:**
- Offline support (falls back to cache)
- Automatic caching (saves successful remote loads)
- Graceful degradation

---

**Pattern Benefits:**
- ✅ Treat individual and composite objects uniformly
- ✅ Easy to add new component types
- ✅ Simplifies client code

**Trade-offs:**
- ⚠️ Can make design overly general

---

### 7. Facade Pattern

**Intent:** Provide a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use.

**Problem It Solves:**
A subsystem has many complex classes and interactions. You want a simple interface to common tasks.

**Analogy:**
A car's steering wheel, pedals, and gear shift are a facade. Under the hood, turning the wheel involves power steering pumps, tie rods, steering columns, etc. The simple interface hides the complexity.

---

**Implementation in Project:**

**Presenters act as facades** for complex use case interactions.

**Example: GetxLoginPresenter**

File: [lib/presentation/presenters/getx_login_presenter.dart](../lib/presentation/presenters/getx_login_presenter.dart:62-76)

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

**What it facades:**

```
UI calls simple method: presenter.auth()

Behind the scenes, presenter coordinates:
  1. Clear previous error
  2. Show loading
  3. Call authentication use case
  4. Call save account use case
  5. Trigger navigation
  6. Handle errors
  7. Map domain errors to UI errors
  8. Hide loading
```

**Without facade (UI would need to do):**

```dart
// ❌ Complex, error-prone
try {
  setState(() => isLoading = true);
  final account = await remoteAuth.auth(params);
  await secureStorage.save(key: 'token', value: account.token);
  Navigator.pushNamed(context, '/surveys');
} on HttpError catch (e) {
  if (e == HttpError.unauthorized) {
    showError('Invalid credentials');
  } else {
    showError('Unexpected error');
  }
  setState(() => isLoading = false);
}
```

**With facade (UI does):**

```dart
// ✅ Simple, clear
presenter.auth();
```

---

**Pattern Structure:**

```
┌─────────────────────────────┐
│   Client                    │
│   (UI)                      │
└─────────────────────────────┘
              │
              │ uses
              ↓
┌─────────────────────────────┐
│   Facade                    │
│   (Presenter)               │
│                             │
│   + auth()                  │
└─────────────────────────────┘
         │
         │ coordinates
         ↓
┌─────────────────────────────────────┐
│   Subsystem Classes                 │
│   • Authentication                  │
│   • SaveCurrentAccount              │
│   • Navigation                      │
│   • Error Handling                  │
│   • Loading State                   │
└─────────────────────────────────────┘
```

**Benefits:**
- ✅ Simplifies complex subsystems
- ✅ Reduces coupling between client and subsystem
- ✅ Makes subsystem easier to use
- ✅ Promotes weak coupling

---

## Behavioral Patterns

Behavioral patterns deal with **algorithms** and the **assignment of responsibilities** between objects.

---

### 8. Strategy Pattern

**Intent:** Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

**Problem It Solves:**
You have multiple ways to perform a task and want to choose the approach at runtime.

**Analogy:**
Navigation apps offer multiple strategies: fastest route, shortest route, avoid highways, scenic route. You choose the strategy, but the navigation system (context) remains the same.

---

**Implementation in Project:**

**Validation validators** are strategies!

**Strategy Interface:**

File: [lib/validation/protocols/field_validation.dart](../lib/validation/protocols/field_validation.dart)

```dart
import '../../presentation/protocols/protocols.dart';

abstract class FieldValidation {
  String get field;
  ValidationError? validate(Map input);
}
```

---

**Concrete Strategies:**

```dart
// Strategy 1: Required Field
class RequiredFieldValidation implements FieldValidation {
  ValidationError? validate(Map input) =>
    input[field]?.isNotEmpty == true ? null : ValidationError.requiredField;
}

// Strategy 2: Email
class EmailValidation implements FieldValidation {
  ValidationError? validate(Map input) {
    final regex = RegExp(r"^[a-zA-Z0-9.!#$%&'*+/=?^_`{|}~]+@[a-zA-Z0-9]+\.[a-zA-Z]+");
    final isValid = input[field]?.isNotEmpty != true || regex.hasMatch(input[field]);
    return isValid ? null : ValidationError.invalidField;
  }
}

// Strategy 3: Minimum Length
class MinLengthValidation implements FieldValidation {
  final int size;
  ValidationError? validate(Map input) {
    final value = input[field] as String?;
    return value != null && value.length >= size
      ? null
      : ValidationError.invalidField;
  }
}

// Strategy 4: Compare Fields
class CompareFieldsValidation implements FieldValidation {
  final String fieldToCompare;
  ValidationError? validate(Map input) {
    return input[field] != null &&
           input[fieldToCompare] != null &&
           input[field] == input[fieldToCompare]
      ? null
      : ValidationError.invalidField;
  }
}
```

---

**Context (uses strategies):**

```dart
// ValidationComposite is the Context
class ValidationComposite implements Validation {
  final List<FieldValidation> validations; // Strategies!

  ValidationError? validate({ required String field, required Map input}) {
    for (final validation in validations.where((v) => v.field == field)) {
      error = validation.validate(input); // Execute strategy
      if (error != null) return error;
    }
    return null;
  }
}
```

---

**Usage:**

```dart
// Choose strategies at runtime (via factory)
final validation = ValidationComposite([
  RequiredFieldValidation('email'),
  EmailValidation('email'),
  RequiredFieldValidation('password'),
  MinLengthValidation(field: 'password', size: 3),
]);

// Context executes appropriate strategies
validation.validate(field: 'email', input: formData);
```

---

**Pattern Structure:**

```
┌─────────────────────────────┐
│   Context                   │
│   (ValidationComposite)     │
│                             │
│   - strategy: Strategy      │
│   + validate()              │
│     calls strategy.validate │
└─────────────────────────────┘
              │
              │ uses
              ↓
┌─────────────────────────────┐
│   Strategy (Interface)      │
│   FieldValidation           │
│   + validate(input)         │
└─────────────────────────────┘
              ▲
              │ implemented by
   ┌──────────┼──────────┬──────────┐
   │          │          │          │
Required   Email     MinLength  Compare
```

**Benefits:**
- ✅ Algorithms are interchangeable
- ✅ Easy to add new strategies
- ✅ Follows Open/Closed Principle
- ✅ Eliminates conditional statements

**Example without Strategy:**
```dart
// ❌ Hard to extend, violates OCP
ValidationError? validate(String type, Map input) {
  if (type == 'required') {
    return input['field']?.isNotEmpty == true ? null : error;
  } else if (type == 'email') {
    return regex.hasMatch(input['field']) ? null : error;
  } else if (type == 'minLength') {
    return input['field'].length >= 3 ? null : error;
  }
  // Adding new type requires modifying this function!
}
```

---

### 9. Observer Pattern

**Intent:** Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified automatically.

**Problem It Solves:**
You have objects that need to stay synchronized with changes in another object.

**Analogy:**
YouTube subscriptions. When a channel (subject) uploads a new video, all subscribers (observers) get notified.

---

**Implementation in Project:**

**Reactive Streams** - Presenters use GetX `Rx` observables.

**Subject (Observable):**

File: [lib/presentation/presenters/getx_login_presenter.dart](../lib/presentation/presenters/getx_login_presenter.dart:15-22)

```dart
class GetxLoginPresenter extends GetxController {
  // Observables (Subjects)
  final _emailError = Rx<UIError?>(null);
  final _passwordError = Rx<UIError?>(null);

  // Expose streams (for observers to subscribe)
  Stream<UIError?> get emailErrorStream => _emailError.stream;
  Stream<UIError?> get passwordErrorStream => _passwordError.stream;

  // Notify observers (change state)
  void validateEmail(String email) {
    _email = email;
    _emailError.value = _validateField('email'); // Triggers notification
    _validateForm();
  }
}
```

---

**Observer (Subscriber):**

File: [lib/ui/pages/login/components/email_input.dart](../lib/ui/pages/login/components/email_input.dart)

```dart
class EmailInput extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final presenter = Provider.of<LoginPresenter>(context);

    // Subscribe to stream (Observer pattern)
    return StreamBuilder<UIError?>(
      stream: presenter.emailErrorStream, // Subscribe
      builder: (context, snapshot) {
        // Automatically rebuilds when stream emits new value
        return TextFormField(
          decoration: InputDecoration(
            errorText: snapshot.hasData ? snapshot.data?.description : null,
          ),
          onChanged: presenter.validateEmail,
        );
      },
    );
  }
}
```

---

**Flow:**

```
1. User types in email field
    ↓
2. UI calls: presenter.validateEmail('test@email.com')
    ↓
3. Presenter validates and updates: _emailError.value = ...
    ↓
4. Rx<UIError?> notifies all stream subscribers
    ↓
5. StreamBuilder receives notification
    ↓
6. StreamBuilder rebuilds widget with new error
```

---

**Pattern Structure:**

```
┌─────────────────────────────┐
│   Subject (Observable)      │
│   GetxLoginPresenter        │
│                             │
│   - observers: List         │
│   + attach(observer)        │ (stream subscription)
│   + notifyObservers()       │ (value change)
└─────────────────────────────┘
              │
              │ notifies
              ↓
┌─────────────────────────────┐
│   Observer                  │
│   StreamBuilder             │
│                             │
│   + update()                │ (builder function)
└─────────────────────────────┘
```

**Multiple observers:**

```dart
// One subject, multiple observers
final emailError = Rx<UIError?>(null);

// Observer 1
StreamBuilder<UIError?>(
  stream: emailError.stream,
  builder: (_, snapshot) => Text(snapshot.data?.description ?? ''),
)

// Observer 2
StreamBuilder<UIError?>(
  stream: emailError.stream,
  builder: (_, snapshot) => Icon(
    snapshot.hasData ? Icons.error : Icons.check
  ),
)

// Change once, both observers notified
emailError.value = UIError.invalidField;
```

---

**Benefits:**
- ✅ Loose coupling (subject doesn't know observers)
- ✅ Dynamic relationships (subscribe/unsubscribe at runtime)
- ✅ Broadcast communication

**Trade-offs:**
- ⚠️ Unexpected updates
- ⚠️ Memory leaks if not unsubscribed (GetX handles this)

---

### 10. Template Method Pattern

**Intent:** Define the skeleton of an algorithm in an operation, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps without changing the algorithm's structure.

**Problem It Solves:**
You have an algorithm with steps that should be customizable, but the overall structure should remain the same.

**Analogy:**
Making coffee and tea both follow the same template:
1. Boil water (same)
2. Brew (different: coffee grounds vs tea leaves)
3. Pour in cup (same)
4. Add condiments (different: sugar/milk vs lemon)

---

**Implementation in Project:**

**Presenter mixins** define template methods.

**Example: LoadingManager Mixin (Template)**

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

**Template structure:**
```dart
// Template method
void handleLoading(BuildContext context, Stream<bool> stream) {
  stream.listen((isLoading) {
    if (isLoading) {
      showLoadingDialog(); // Step 1
    } else {
      hideLoadingDialog(); // Step 2
    }
  });
}
```

---

**Used by multiple classes:**

```dart
class LoginPage extends StatelessWidget
  with KeyboardManager, LoadingManager, UIErrorManager, NavigationManager {

  @override
  Widget build(BuildContext context) {
    return Builder(
      builder: (context) {
        handleLoading(context, presenter.isLoadingStream); // Uses template
        handleMainError(context, presenter.mainErrorStream);
        handleNavigation(presenter.navigateToStream);
        // ...
      }
    );
  }
}
```

All pages use the same loading template!

---

**Pattern Structure:**

```
┌─────────────────────────────┐
│   AbstractClass (Mixin)     │
│   LoadingManager            │
│                             │
│   + templateMethod()        │
│     • step1()               │
│     • step2()               │
│     • step3()               │
└─────────────────────────────┘
              ▲
              │ used by
   ┌──────────┼──────────┬──────────┐
   │          │          │          │
LoginPage SignUpPage SurveysPage ...
```

**Benefits:**
- ✅ Code reuse (algorithm defined once)
- ✅ Controlled extension points
- ✅ Follows Hollywood Principle ("don't call us, we'll call you")

---

### 11. Command Pattern

**Intent:** Encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests.

**Problem It Solves:**
You want to issue requests without knowing the receiver or the specifics of the request.

**Analogy:**
Restaurant ordering. You give the waiter (invoker) a written order (command object). The waiter doesn't cook - they pass the order to the kitchen (receiver).

---

**Implementation in Project:**

**Use Cases are Commands!**

**Command Interface:**

```dart
abstract class Authentication {
  Future<AccountEntity> auth(AuthenticationParams params);
}
```

---

**Concrete Command:**

```dart
class RemoteAuthentication implements Authentication {
  final HttpClient httpClient;
  final String url;

  Future<AccountEntity> auth(AuthenticationParams params) async {
    // Execute command
    final body = RemoteAuthenticationParams.fromDomain(params).toJson();
    final response = await httpClient.request(url: url, method: 'post', body: body);
    return RemoteAccountModel.fromJson(response).toEntity();
  }
}
```

---

**Invoker:**

```dart
class GetxLoginPresenter {
  final Authentication authentication; // Command

  Future<void> auth() async {
    // Invoke command
    final account = await authentication.auth(params);
  }
}
```

---

**Pattern Structure:**

```
┌─────────────────────────────┐
│   Invoker                   │
│   (Presenter)               │
│                             │
│   - command: Command        │
│   + invoke()                │
│     calls command.execute() │
└─────────────────────────────┘
              │
              │ uses
              ↓
┌─────────────────────────────┐
│   Command (Interface)       │
│   (Use Case)                │
│   + execute()               │
└─────────────────────────────┘
              ▲
              │ implements
┌─────────────────────────────┐
│   ConcreteCommand           │
│   (RemoteAuthentication)    │
│                             │
│   + execute()               │
│     calls receiver methods  │
└─────────────────────────────┘
              │
              │ uses
              ↓
┌─────────────────────────────┐
│   Receiver                  │
│   (HttpClient)              │
└─────────────────────────────┘
```

**Benefits:**
- ✅ Decouples invoker from receiver
- ✅ Easy to add new commands
- ✅ Commands can be queued, logged, undone

---

## Architectural Patterns

Patterns that deal with the overall structure of the application.

---

### 12. Dependency Injection Pattern

**Intent:** Provide dependencies from outside rather than creating them internally.

**Problem It Solves:**
Creating dependencies internally creates tight coupling and makes testing difficult.

**Analogy:**
Hotel room service. You don't cook your own food (create dependencies). The hotel (DI container) delivers prepared meals (dependencies) to your room.

---

**Implementation in Project:**

**Constructor Injection** everywhere!

**Example:**

```dart
class GetxLoginPresenter {
  final Validation validation;
  final Authentication authentication;
  final SaveCurrentAccount saveCurrentAccount;

  // Dependencies injected via constructor
  GetxLoginPresenter({
    required this.validation,
    required this.authentication,
    required this.saveCurrentAccount
  });
}
```

**Without DI (bad):**
```dart
// ❌ Creates own dependencies
class LoginPresenter {
  LoginPresenter() {
    this.validation = ValidationComposite([...]);
    this.authentication = RemoteAuthentication(
      httpClient: HttpAdapter(Client()),
      url: 'http://api.com/login'
    );
  }
}
```

**Problems:**
- Hard-coded dependencies
- Can't swap implementations
- Can't test (can't inject mocks)

---

**With DI (good):**
```dart
// ✅ Dependencies injected
class GetxLoginPresenter {
  final Validation validation;
  final Authentication authentication;

  GetxLoginPresenter({
    required this.validation,
    required this.authentication
  });
}

// Factory creates and injects
LoginPresenter makeLoginPresenter() {
  return GetxLoginPresenter(
    validation: makeValidation(),
    authentication: makeAuthentication()
  );
}

// Tests inject mocks
test('...', () {
  final presenter = GetxLoginPresenter(
    validation: MockValidation(),
    authentication: MockAuthentication()
  );
});
```

---

**Benefits:**
- ✅ Loose coupling
- ✅ Easy testing (inject mocks)
- ✅ Flexible (swap implementations)
- ✅ Follows Dependency Inversion Principle

---

### 13. Repository Pattern

**Intent:** Encapsulate data access logic and provide a collection-like interface for accessing domain objects.

**Problem It Solves:**
Business logic shouldn't know whether data comes from an API, database, or cache.

**Analogy:**
A library. You ask the librarian for a book. You don't know if it's in the main stacks, storage, or needs to be ordered. The librarian (repository) handles the details.

---

**Implementation in Project:**

**Data layer use cases act as repositories.**

**Example: RemoteLoadSurveys (Repository)**

```dart
class RemoteLoadSurveys implements LoadSurveys {
  final HttpClient httpClient;
  final String url;

  // Collection-like interface
  Future<List<SurveyEntity>> load() async {
    final response = await httpClient.request(url: url, method: 'get');
    return response.map<SurveyEntity>((json) =>
      RemoteSurveyModel.fromJson(json).toEntity()
    ).toList();
  }
}
```

**Caller doesn't know where data comes from:**

```dart
class GetxSurveysPresenter {
  final LoadSurveys loadSurveys; // Repository interface

  Future<void> loadData() async {
    final surveys = await loadSurveys.load();
    // Could be from API, cache, or both!
  }
}
```

---

**Repository with Fallback:**

```dart
class RemoteLoadSurveysWithLocalFallback implements LoadSurveys {
  final RemoteLoadSurveys remote;
  final LocalLoadSurveys local;

  Future<List<SurveyEntity>> load() async {
    try {
      return await remote.load(); // Try API
    } catch (e) {
      return await local.load(); // Fall back to cache
    }
  }
}
```

Presenter still just calls `loadSurveys.load()` - doesn't know about fallback logic!

---

**Benefits:**
- ✅ Abstracts data source
- ✅ Centralizes data access logic
- ✅ Easy to add caching, fallbacks
- ✅ Domain doesn't depend on data source

---

### 14. ViewModel Pattern

**Intent:** Prepare data specifically for display in the UI.

**Problem It Solves:**
Domain entities aren't always in the right format for UI display.

**Analogy:**
A menu at a restaurant. The kitchen (domain) works with raw ingredients. The menu (ViewModel) shows dishes in an appetizing, readable format for customers.

---

**Implementation in Project:**

**ViewModels transform entities for UI.**

**Example: SurveyViewModel**

File: [lib/ui/pages/surveys/survey_viewmodel.dart](../lib/ui/pages/surveys/survey_viewmodel.dart)

```dart
import 'package:equatable/equatable.dart';

class SurveyViewModel extends Equatable {
  final String id;
  final String question;
  final String date;
  final bool didAnswer;

  List get props => [id, question, date, didAnswer];

  SurveyViewModel({
    required this.id,
    required this.question,
    required this.date,
    required this.didAnswer,
  });
}
```

**Domain Entity:**

```dart
class SurveyEntity {
  final String id;
  final String question;
  final DateTime dateTime; // DateTime object
  final bool didAnswer;
}
```

**Transformation:**

```dart
extension SurveyEntityExtensions on SurveyEntity {
  SurveyViewModel toViewModel() => SurveyViewModel(
    id: id,
    question: question,
    date: dateTime.toFormattedString(), // DateTime → formatted String
    didAnswer: didAnswer,
  );
}
```

---

**Why separate ViewModel?**

```dart
// Domain entity uses DateTime (good for business logic)
SurveyEntity(dateTime: DateTime(2024, 1, 15))

// UI needs formatted string (good for display)
SurveyViewModel(date: "15 Jan 2024")
```

**Presenter creates ViewModels:**

```dart
class GetxSurveysPresenter {
  Future<void> loadData() async {
    final surveys = await loadSurveys.load(); // Domain entities
    surveysData = surveys
      .map((entity) => entity.toViewModel()) // Transform to ViewModels
      .toList();
  }
}
```

---

**Benefits:**
- ✅ Separates display logic from business logic
- ✅ Domain stays pure (no formatting)
- ✅ UI gets exactly what it needs
- ✅ Easy to change UI format without touching domain

---

### 15. Composition Root Pattern

**Intent:** Compose the object graph in a single location at the application's entry point.

**Problem It Solves:**
Where should object creation and wiring happen?

**Analogy:**
An orchestra conductor. Before the concert, the conductor (composition root) ensures every musician (object) has their instrument (dependencies) and knows their part. During the concert, musicians just play.

---

**Implementation in Project:**

**The entire `main/` layer is the Composition Root!**

File: [lib/main/main.dart](../lib/main/main.dart)

```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:get/get.dart';
import '../ui/components/components.dart';
import 'factories/factories.dart';

void main() {
  runApp(App());
}

class App extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    SystemChrome.setSystemUIOverlayStyle(SystemUiOverlayStyle.light);

    return GetMaterialApp(
      title: '4Dev',
      debugShowCheckedModeBanner: false,
      theme: makeAppTheme(),
      initialRoute: '/',
      getPages: [
        GetPage(name: '/', page: makeSplashPage, transition: Transition.fade),
        GetPage(name: '/login', page: makeLoginPage, transition: Transition.fadeIn),
        GetPage(name: '/signup', page: makeSignUpPage),
        GetPage(name: '/surveys', page: makeSurveysPage, transition: Transition.fadeIn),
        GetPage(
          name: '/survey_result/:survey_id',
          page: makeSurveyResultPage,
        ),
      ],
    );
  }
}
```

**All factories are called from here:**

```
App (main.dart)
  └─ makeLoginPage()
       └─ makeLoginPresenter()
            ├─ makeAuthentication()
            │    ├─ makeHttpClient()
            │    └─ makeApiUrl()
            ├─ makeValidation()
            └─ makeSaveCurrentAccount()
                 └─ makeSecureStorage()
```

**Everything is wired at startup, before any business logic runs!**

---

**Benefits:**
- ✅ Single location for all wiring
- ✅ Easy to see full dependency graph
- ✅ Application code doesn't know about composition
- ✅ Easy to swap implementations (change one factory)

---

## Pattern Combinations

Patterns rarely exist in isolation. Here's how they combine in this project:

### Combination 1: Factory + Dependency Injection

**Factory creates objects, DI wires them together.**

```dart
// Factory creates
HttpClient makeHttpClient() => HttpAdapter(Client());

// DI injects
Authentication makeAuthentication() =>
  RemoteAuthentication(
    httpClient: makeHttpClient(), // DI!
    url: makeApiUrl()
  );
```

---

### Combination 2: Decorator + Adapter + Factory

**Decorator wraps Adapter created by Factory.**

```dart
HttpClient makeAuthorizeHttpClientDecorator() {
  return AuthorizeHttpClientDecorator( // Decorator
    decoratee: makeHttpAdapter(), // Adapter (from Factory)
    fetchSecureCacheStorage: makeSecureStorage(),
    deleteSecureCacheStorage: makeSecureStorage()
  );
}
```

---

### Combination 3: Composite + Strategy + Builder

**Builder creates Strategies, Composite combines them.**

```dart
Validation makeValidation() {
  return ValidationComposite(  // Composite
    [
      ...ValidationBuilder  // Builder
        .field('email')
        .required()  // Strategy
        .email()  // Strategy
        .build(),
    ]
  );
}
```

---

### Combination 4: Repository + Composite

**Composite Repository combines Remote and Local Repositories.**

```dart
LoadSurveys makeLoadSurveys() {
  return RemoteLoadSurveysWithLocalFallback(  // Composite
    remote: makeRemoteLoadSurveys(),  // Repository
    local: makeLocalLoadSurveys()  // Repository
  );
}
```

---

## Pattern Benefits

### Why So Many Patterns?

**Each pattern solves a specific problem:**

| Pattern | Problem Solved |
|---------|---------------|
| Factory | Hide object creation complexity |
| Builder | Construct complex objects step-by-step |
| Adapter | Make incompatible interfaces work together |
| Decorator | Add behavior without modifying classes |
| Composite | Treat individual & composite objects uniformly |
| Facade | Simplify complex subsystems |
| Strategy | Make algorithms interchangeable |
| Observer | Notify dependents of state changes |
| Template Method | Define algorithm skeleton, defer steps |
| Command | Encapsulate requests as objects |
| Dependency Injection | Provide dependencies from outside |
| Repository | Abstract data source |
| ViewModel | Prepare data for display |
| Composition Root | Wire dependencies in one place |

---

### Pattern Synergy

**Patterns work together to create a flexible, maintainable architecture:**

```
Composition Root
  └─ Creates objects via Factories
       └─ Injects dependencies (DI)
            └─ Wraps with Decorators
                 └─ Uses Adapters for external libs
                      └─ Combines via Composites
                           └─ Strategies are interchangeable
                                └─ Observers react to changes
                                     └─ Commands encapsulate operations
                                          └─ Repositories abstract data
                                               └─ ViewModels format for UI
```

---

## Key Takeaways

### Pattern Categories Summary

**Creational (How objects are created):**
- Factory Method - Create via factories
- Builder - Build step-by-step
- Singleton - One instance

**Structural (How objects are composed):**
- Adapter - Convert interfaces
- Decorator - Add behavior
- Composite - Part-whole hierarchies
- Facade - Simplify subsystem

**Behavioral (How objects interact):**
- Strategy - Interchangeable algorithms
- Observer - Notify dependents
- Template Method - Algorithm skeleton
- Command - Encapsulate requests

**Architectural (Overall structure):**
- Dependency Injection - Provide dependencies
- Repository - Abstract data source
- ViewModel - Format for display
- Composition Root - Wire dependencies

---

### When to Use Each Pattern

**Use Factory when:**
- Creating objects is complex
- Want to hide concrete classes
- Need to swap implementations

**Use Builder when:**
- Objects have many optional parameters
- Want fluent, readable construction

**Use Adapter when:**
- Need to use library with incompatible interface
- Want to isolate external dependencies

**Use Decorator when:**
- Want to add behavior dynamically
- Need to stack multiple behaviors

**Use Composite when:**
- Part-whole hierarchy
- Want to treat individual & composite uniformly

**Use Strategy when:**
- Multiple algorithms for same task
- Want to choose algorithm at runtime

**Use Observer when:**
- One-to-many dependency
- Want automatic updates

**Use Repository when:**
- Abstracting data source
- Want to centralize data access

---

## Next Steps

Now that you understand all the design patterns:

1. **[Component Deep-Dive](04-component-deep-dive.md)** - See patterns in complete features
2. **[Data Flow Guide](05-data-flow-guide.md)** - Follow data through patterns
3. **[Getting Started Guide](06-getting-started-guide.md)** - Apply patterns yourself
4. **[Testing Strategy](07-testing-strategy.md)** - Test patterns effectively

---

**Documentation Version:** 1.0
**Last Updated:** 2025

---

**Further Reading:**
- "Design Patterns: Elements of Reusable Object-Oriented Software" (Gang of Four)
- "Head First Design Patterns" (Freeman & Freeman)
- "Refactoring to Patterns" (Joshua Kerievsky)
