# 01 - Architecture Overview

## Table of Contents

1. [Introduction](#introduction)
2. [What You'll Learn](#what-youll-learn)
3. [Project Overview](#project-overview)
4. [Clean Architecture Fundamentals](#clean-architecture-fundamentals)
5. [Project Structure](#project-structure)
6. [Layer Hierarchy](#layer-hierarchy)
7. [SOLID Principles in Practice](#solid-principles-in-practice)
8. [Technology Stack](#technology-stack)
9. [Architecture Benefits](#architecture-benefits)
10. [Quick Reference](#quick-reference)
11. [Key Takeaways](#key-takeaways)
12. [Next Steps](#next-steps)

---

## Introduction

Welcome to the **Clean Flutter App** documentation! This project is a production-ready Flutter application that demonstrates **Clean Architecture** principles with a focus on testability, maintainability, and scalability.

**What is this app?**
ForDev (4Dev) is a survey application for developers where users can:
- Login and create accounts
- View available surveys
- Vote on survey questions
- See real-time survey results

But more importantly, this project serves as a **reference implementation** of professional Flutter architecture that you can learn from and apply to your own projects.

---

## What You'll Learn

By studying this documentation, you will understand:

- ✅ How Clean Architecture works in Flutter applications
- ✅ The purpose and responsibility of each architectural layer
- ✅ How to organize code for maximum testability and maintainability
- ✅ How different design patterns work together in a real application
- ✅ How data flows through the application layers
- ✅ Why separating concerns makes your code better
- ✅ How to scale a Flutter app to handle complex features

**Prerequisites:**
Basic knowledge of Flutter and Dart. If you're familiar with StatefulWidget, State management basics, and async/await, you're ready to dive in!

---

## Project Overview

### Project Statistics

```
📦 Project: ForDev (4Dev) - Enquetes para Programadores
📱 Platform: Flutter (iOS, Android, Web)
🏗️  Architecture: Clean Architecture + TDD
📊 Total Dart Files: 178 files
📝 Total Lines of Code: ~2,705 lines
🧪 Test Files: 32 test files
🎯 Test-Driven Development: Yes
🌐 Null Safety: Full migration
```

### Core Features

1. **Authentication System**
   - User login with email/password
   - User registration (sign up)
   - Secure token storage
   - Session management

2. **Survey System**
   - Load and display surveys
   - Vote on survey questions
   - View real-time results
   - Offline support with local cache

3. **User Experience**
   - Internationalization (i18n) support
   - Loading states
   - Error handling
   - Smooth navigation

---

## Clean Architecture Fundamentals

### What is Clean Architecture?

**Simple Analogy:**
Think of Clean Architecture like building a house with separate rooms, each with a specific purpose:

- The **living room** (UI) is where guests interact
- The **kitchen** (Business Logic) is where the real work happens
- The **pantry** (Data Storage) is where ingredients are kept
- The **grocery store** (External APIs) is where you get new ingredients

**Key Point:** Each room has a clear purpose, and you can renovate one room without destroying the others!

### The Core Idea

Clean Architecture separates your application into **independent layers** where:

1. **Inner layers don't know about outer layers** (Dependency Rule)
2. **Business logic is independent** of frameworks, UI, and databases
3. **Everything is testable** without external dependencies
4. **You can easily swap implementations** (e.g., change from REST to GraphQL)

### The Dependency Rule

**The Golden Rule of Clean Architecture:**

> Source code dependencies must point **inward** toward higher-level policies.

```
┌─────────────────────────────────────────┐
│  UI Layer (Outer)                       │  ◄─── Depends on
│  ┌───────────────────────────────────┐  │       Presentation
│  │ Presentation Layer                │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │ Domain Layer (Inner/Core)   │  │  │  ◄─── Pure Business Logic
│  │  │                             │  │  │       (No dependencies!)
│  │  └─────────────────────────────┘  │  │
│  │                                   │  │
│  └───────────────────────────────────┘  │
│                                         │
└─────────────────────────────────────────┘

         ↑                          ↑
         │                          │
    Data Layer                  Infra Layer
  (Implements Domain)         (External Adapters)
```

**What this means in practice:**

- UI knows about Presenters, but Presenters don't know about UI
- Presenters know about Use Cases, but Use Cases don't know about Presenters
- Use Cases define interfaces, but don't know about their implementations
- Data layer implements interfaces defined in Domain layer

---

## Project Structure

### High-Level Directory Structure

```
clean-flutter-app/
├── lib/
│   ├── domain/          # 🎯 Business Logic Core (Framework Independent)
│   ├── data/            # 💾 Data Handling & Use Case Implementations
│   ├── infra/           # 🔌 External Library Adapters
│   ├── presentation/    # 🎨 Presentation Logic (Separated from UI)
│   ├── ui/              # 📱 Flutter Widgets & Visual Components
│   ├── validation/      # ✅ Form Validation Logic
│   ├── main/            # 🏭 Dependency Injection & Composition Root
│   └── main.dart        # 🚀 Application Entry Point
├── test/                # 🧪 Mirror of lib/ structure with tests
├── requirements/        # 📋 BDD Specs & Use Case Documentation
└── docs/                # 📚 This documentation!
```

### Layer-by-Layer Overview

#### 1. **Domain Layer** (`lib/domain/`)

**Purpose:** Contains pure business logic with **zero dependencies** on Flutter or external libraries.

**Analogy:** This is like the blueprint of your house - it describes *what* needs to be done, not *how* to do it.

```
lib/domain/
├── entities/           # Business objects (Account, Survey, SurveyResult)
├── usecases/           # Business operations (Login, LoadSurveys, Vote)
└── helpers/            # Domain-specific helpers (DomainError)
```

**Key Files:**
- [domain/entities/account_entity.dart](../lib/domain/entities/account_entity.dart) - User account model
- [domain/usecases/authentication.dart](../lib/domain/usecases/authentication.dart) - Login contract
- [domain/helpers/domain_error.dart](../lib/domain/helpers/domain_error.dart) - Business errors

**Example:**

```dart
// This is a USE CASE interface (contract)
// It says "I need authentication" but doesn't care HOW it's done
abstract class Authentication {
  Future<AccountEntity> auth(AuthenticationParams params);
}
```

#### 2. **Data Layer** (`lib/data/`)

**Purpose:** Implements domain use cases and handles data operations.

**Analogy:** This is the construction crew that actually builds according to the blueprint.

```
lib/data/
├── usecases/           # Concrete implementations (RemoteAuth, LocalAuth)
├── models/             # Data models with JSON conversion
├── http/               # HTTP client interface
└── cache/              # Cache storage interface
```

**Key Responsibility:** Transform external data (JSON, cache) into domain entities.

**Example:**

```dart
// This IMPLEMENTS the authentication use case
class RemoteAuthentication implements Authentication {
  Future<AccountEntity> auth(AuthenticationParams params) async {
    // Makes HTTP request, converts JSON to entity
  }
}
```

#### 3. **Infrastructure Layer** (`lib/infra/`)

**Purpose:** Adapters for external libraries (HTTP clients, storage, etc.)

**Analogy:** These are the specialized tools and equipment the construction crew uses.

```
lib/infra/
├── http/               # HTTP adapter wrapping 'http' package
└── cache/              # Storage adapters wrapping storage libraries
```

**Key Pattern:** **Adapter Pattern** - wraps third-party libraries so the rest of the app doesn't depend on them directly.

**Example:**

```dart
// Wraps the 'http' package so we can easily swap it later
class HttpAdapter implements HttpClient {
  final http.Client client;
  // Adapts http package to our interface
}
```

#### 4. **Presentation Layer** (`lib/presentation/`)

**Purpose:** Presentation logic separated from UI (business logic for screens).

**Analogy:** This is the interior designer who decides how each room should look and behave.

```
lib/presentation/
├── presenters/         # Presenter implementations using GetX
├── protocols/          # Presentation contracts (Validation)
├── mixins/             # Reusable presenter behaviors
└── helpers/            # Presentation utilities
```

**Key Pattern:** Presenters expose **streams** that UI listens to (unidirectional data flow).

**Example:**

```dart
class GetxLoginPresenter {
  Stream<UIError?> get emailErrorStream => _emailError.stream;
  Stream<bool> get isLoadingStream => _isLoading.stream;

  Future<void> auth() async {
    // Calls use case, updates streams
  }
}
```

#### 5. **UI Layer** (`lib/ui/`)

**Purpose:** Flutter widgets and visual components.

**Analogy:** This is what guests actually see and interact with - the furniture and decorations.

```
lib/ui/
├── pages/              # Application screens (Login, Surveys, etc.)
├── components/         # Reusable widgets (Buttons, Headers, etc.)
├── helpers/            # UI utilities (i18n, theme)
└── mixins/             # UI mixins (Keyboard, Loading, Navigation)
```

**Key Principle:** UI is **dumb** - it only listens to presenter streams and displays data.

**Example:**

```dart
// UI just listens and rebuilds
StreamBuilder<UIError?>(
  stream: presenter.emailErrorStream,
  builder: (context, snapshot) {
    return Text(snapshot.data?.description ?? '');
  },
)
```

#### 6. **Validation Layer** (`lib/validation/`)

**Purpose:** Standalone validation logic (not tied to any specific layer).

**Analogy:** Quality control inspector who checks everything meets standards.

```
lib/validation/
├── validators/         # Email, Required, MinLength, Compare validators
└── protocols/          # Validation interfaces
```

#### 7. **Main Layer** (`lib/main/`)

**Purpose:** **Composition Root** - where all dependencies are wired together.

**Analogy:** The construction manager who assembles the right team for each job.

```
lib/main/
├── factories/          # Factory methods to create objects
│   ├── pages/          # Page factories with all dependencies
│   ├── usecases/       # Use case factories
│   └── http/           # HTTP client factories
├── builders/           # Builder patterns (ValidationBuilder)
├── composites/         # Composite patterns (Fallback strategies)
└── decorators/         # Decorator patterns (Authorization)
```

**Key Principle:** This is the **ONLY** layer that knows about all other layers.

---

## Layer Hierarchy

### The Dependency Flow

**Visual Representation:**

```
┌──────────────────────────────────────────────────────────┐
│                    MAIN LAYER                             │
│              (Composition Root / DI)                      │
│                                                           │
│  Knows about ALL layers and wires them together          │
└──────────────────────────────────────────────────────────┘
                           │
                           │ creates & injects
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
         ↓                 ↓                 ↓
┌─────────────────┐ ┌─────────────┐ ┌──────────────────┐
│   UI LAYER      │ │ VALIDATION  │ │ INFRASTRUCTURE   │
│   (Widgets)     │ │ (Validators)│ │   (Adapters)     │
│                 │ │             │ │                  │
│  Depends on:    │ │  Depends on:│ │   Implements:    │
│  - Presentation │ │  - Nothing  │ │   - Data Layer   │
└─────────────────┘ └─────────────┘ └──────────────────┘
         │                                    ↑
         │                                    │
         ↓                                    │
┌─────────────────┐                           │
│ PRESENTATION    │                           │
│ (Presenters)    │                           │
│                 │                           │
│  Depends on:    │                           │
│  - Domain       │                           │
│  - Validation   │                           │
└─────────────────┘                           │
         │                                    │
         │                                    │
         ↓                                    │
┌─────────────────┐                           │
│  DOMAIN LAYER   │ ←─────────────────────────┘
│  (Business      │
│   Logic Core)   │ ┌──────────────────┐
│                 │ │   DATA LAYER     │
│  Depends on:    │ │ (Implementations)│
│  - NOTHING!     │ │                  │
│                 │ │   Implements:    │
│  Defines:       │ │   - Domain Layer │
│  - Entities     │ └──────────────────┘
│  - Use Cases    │          ↑
│  - Interfaces   │          │
└─────────────────┘          │
         ↑                   │
         └───────────────────┘
          implements contracts
```

### Dependency Inversion in Action

**Traditional Architecture (BAD):**

```
High-Level Module (Business Logic)
        ↓ depends on
Low-Level Module (Database Implementation)
```

❌ Problem: Business logic depends on implementation details!

**Clean Architecture (GOOD):**

```
High-Level Module (Business Logic)
        ↓ defines interface
    Interface (Use Case)
        ↑ implements
Low-Level Module (Database Implementation)
```

✅ Solution: Both depend on abstraction! Implementation details are pluggable.

### Layer Communication Rules

| Layer | Can Use | Cannot Use |
|-------|---------|------------|
| **Domain** | Nothing | Everything else |
| **Data** | Domain interfaces | Presentation, UI, Infra |
| **Infra** | Data interfaces | Domain, Presentation, UI |
| **Presentation** | Domain, Validation | UI, Data, Infra |
| **UI** | Presentation | Domain, Data, Infra |
| **Validation** | Nothing | Everything else |
| **Main** | Everything | Nothing (it's the root) |

---

## SOLID Principles in Practice

This project is a masterclass in **SOLID principles**. Let's see each principle in action:

### 1. Single Responsibility Principle (SRP)

**Definition:** A class should have only one reason to change.

**Example in Project:**

```dart
// ✅ GOOD: Each class has ONE responsibility

// Responsible ONLY for HTTP communication
class HttpAdapter implements HttpClient { }

// Responsible ONLY for converting JSON to Model
class RemoteAccountModel { }

// Responsible ONLY for authentication logic
class RemoteAuthentication implements Authentication { }

// Responsible ONLY for storing tokens
class SecureStorageAdapter implements SecureCacheStorage { }
```

**File:** [infra/http/http_adapter.dart](../lib/infra/http/http_adapter.dart)
**File:** [data/models/remote_account_model.dart](../lib/data/models/remote_account_model.dart)

---

### 2. Open/Closed Principle (OCP)

**Definition:** Open for extension, closed for modification.

**Example in Project:**

**Adding a new validator without modifying existing code:**

```dart
// ✅ Existing code remains unchanged
class ValidationComposite implements Validation {
  final List<FieldValidation> validations;

  ValidationComposite(this.validations);

  ValidationError? validate({required String field, required Map input}) {
    // Iterates through all validations
  }
}

// Just ADD a new validator, don't modify ValidationComposite!
class PasswordStrengthValidation implements FieldValidation {
  ValidationError? validate(Map input) {
    // New validation logic
  }
}
```

**File:** [main/composites/validation_composite.dart](../lib/main/composites/validation_composite.dart)

---

### 3. Liskov Substitution Principle (LSP)

**Definition:** Subtypes must be substitutable for their base types.

**Example in Project:**

```dart
// ✅ Any implementation of Authentication can be used
abstract class Authentication {
  Future<AccountEntity> auth(AuthenticationParams params);
}

// Can substitute RemoteAuthentication with LocalAuthentication
class RemoteAuthentication implements Authentication { }
class MockAuthentication implements Authentication { } // For tests!

// Presenter doesn't care WHICH implementation
class GetxLoginPresenter {
  final Authentication authentication;

  GetxLoginPresenter({required this.authentication});

  Future<void> auth() async {
    // Works with ANY Authentication implementation!
    await authentication.auth(params);
  }
}
```

**File:** [presentation/presenters/getx_login_presenter.dart](../lib/presentation/presenters/getx_login_presenter.dart)

---

### 4. Interface Segregation Principle (ISP)

**Definition:** Clients shouldn't depend on interfaces they don't use.

**Example in Project:**

```dart
// ✅ GOOD: Small, focused interfaces

// Instead of one giant "Storage" interface, we have:
abstract class SaveSecureCacheStorage {
  Future<void> save({required String key, required String value});
}

abstract class FetchSecureCacheStorage {
  Future<String> fetch(String key);
}

abstract class DeleteSecureCacheStorage {
  Future<void> delete(String key);
}

// Classes only implement what they need!
class SecureStorageAdapter implements
  SaveSecureCacheStorage,
  FetchSecureCacheStorage,
  DeleteSecureCacheStorage {
  // Implements all three
}

// But a SaveCurrentAccount only needs SaveSecureCacheStorage!
class LocalSaveCurrentAccount implements SaveCurrentAccount {
  final SaveSecureCacheStorage saveSecureCacheStorage;
  // Only depends on what it uses
}
```

**File:** [data/cache/save_secure_cache_storage.dart](../lib/data/cache/save_secure_cache_storage.dart)

---

### 5. Dependency Inversion Principle (DIP)

**Definition:** Depend on abstractions, not concretions.

**Example in Project:**

```dart
// ❌ BAD (not in this project):
class LoginPresenter {
  final RemoteAuthentication auth; // Depends on concrete class!
}

// ✅ GOOD (how this project does it):
class GetxLoginPresenter {
  final Authentication authentication; // Depends on abstraction!

  // Can receive ANY implementation
  GetxLoginPresenter({required this.authentication});
}

// In Main layer (DI):
GetxLoginPresenter makeLoginPresenter() {
  return GetxLoginPresenter(
    authentication: makeRemoteAuthentication(), // Concrete injected here
  );
}
```

**File:** [main/factories/pages/login/login_presenter_factory.dart](../lib/main/factories/pages/login/login_presenter_factory.dart)

---

## Technology Stack

### Core Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| **flutter** | SDK | UI framework |
| **get** | 4.3.8 | State management & routing |
| **provider** | 6.0.1 | Dependency injection for widgets |
| **equatable** | 2.0.3 | Value equality for entities |
| **http** | 0.13.3 | HTTP client |
| **flutter_secure_storage** | 4.2.1 | Secure token storage |
| **localstorage** | 4.0.0+1 | Local data caching |
| **carousel_slider** | 4.0.0 | UI carousel for surveys |
| **intl** | 0.17.0 | Internationalization |

### Development Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| **test** | 1.17.10 | Unit testing |
| **mocktail** | 0.1.4 | Mocking framework |
| **faker** | 2.0.0 | Test data generation |
| **network_image_mock** | 2.0.1 | Mock network images |

### Why These Choices?

**GetX for State Management:**
- Lightweight and performant
- Excellent for reactive streams
- Built-in dependency injection
- Simple navigation

**Provider for UI:**
- Makes presenters available to widget tree
- Works well with GetX
- Standard in Flutter ecosystem

**Mocktail for Testing:**
- Modern successor to Mockito
- Null-safe
- Better error messages
- Type-safe mocking

**Equatable:**
- Simplifies entity comparison
- Essential for testing
- Reduces boilerplate

---

## Architecture Benefits

### Why Clean Architecture?

Let's explore the benefits with real examples from this project:

#### 1. **Testability**

**Benefit:** Every layer can be tested independently.

**Example:**

```dart
// Test Presenter WITHOUT real API
test('Should call Authentication with correct values', () async {
  final authentication = MockAuthentication();
  final presenter = GetxLoginPresenter(authentication: authentication);

  await presenter.auth();

  verify(() => authentication.auth(any())).called(1);
});
```

**Why it works:** Presenter depends on `Authentication` interface, not concrete implementation!

**Test File:** [test/presentation/presenters/getx_login_presenter_test.dart](../test/presentation/presenters/getx_login_presenter_test.dart)

---

#### 2. **Maintainability**

**Benefit:** Changes are localized to specific layers.

**Example:**

Want to switch from `http` package to `dio`?

```dart
// ONLY change this file:
class DioAdapter implements HttpClient {
  final Dio dio;
  // New implementation
}

// Everything else remains unchanged!
// ✅ No changes to Domain layer
// ✅ No changes to Data layer
// ✅ No changes to Presentation layer
// ✅ No changes to UI layer
```

---

#### 3. **Scalability**

**Benefit:** Easy to add new features without breaking existing code.

**Example:**

Adding a new feature (e.g., "Forgot Password"):

```
1. Add use case interface in Domain (ForgotPassword)
2. Implement in Data layer (RemoteForgotPassword)
3. Create presenter in Presentation layer
4. Build UI in UI layer
5. Wire together in Main layer

✅ Zero impact on existing features!
```

---

#### 4. **Flexibility**

**Benefit:** Easy to swap implementations.

**Example - Offline Mode:**

```dart
// Composite pattern provides fallback
class RemoteLoadSurveysWithLocalFallback implements LoadSurveys {
  final RemoteLoadSurveys remote;
  final LocalLoadSurveys local;

  Future<List<SurveyEntity>> load() async {
    try {
      return await remote.load(); // Try remote first
    } catch (error) {
      return await local.load(); // Fall back to cache
    }
  }
}
```

**File:** [main/composites/remote_load_surveys_with_local_fallback.dart](../lib/main/composites/remote_load_surveys_with_local_fallback.dart)

---

#### 5. **Team Collaboration**

**Benefit:** Multiple developers can work on different layers simultaneously.

**Example:**

```
Developer A: Works on UI (pages/login_page.dart)
Developer B: Works on API integration (data/usecases/remote_authentication.dart)
Developer C: Writes tests (test/presentation/...)

✅ No conflicts because layers are independent!
```

---

#### 6. **Framework Independence**

**Benefit:** Business logic doesn't depend on Flutter.

**Example:**

```dart
// This code has ZERO Flutter dependencies
// Could run in a Dart CLI app, web server, etc.
class RemoteAuthentication implements Authentication {
  final HttpClient httpClient;
  final String url;

  Future<AccountEntity> auth(AuthenticationParams params) async {
    final body = RemoteAuthenticationParams.fromDomain(params).toJson();
    final response = await httpClient.request(url: url, method: 'post', body: body);
    return RemoteAccountModel.fromJson(response).toEntity();
  }
}
```

**File:** [data/usecases/authentication/remote_authentication.dart](../lib/data/usecases/authentication/remote_authentication.dart)

---

### Trade-offs

**Every architecture has trade-offs. Let's be honest:**

| Benefit | Trade-off |
|---------|-----------|
| ✅ Testability | ⚠️ More files to create |
| ✅ Maintainability | ⚠️ More upfront planning |
| ✅ Scalability | ⚠️ Steeper learning curve |
| ✅ Flexibility | ⚠️ More abstraction layers |

**Is it worth it?**

- ✅ For small apps (< 5 screens): Maybe overkill
- ✅ For medium apps (5-20 screens): Definitely worth it
- ✅ For large apps (20+ screens): Absolutely essential!
- ✅ For learning purposes: Invaluable!

---

## Quick Reference

### Common Questions Answered

#### "Where do I put...?"

| What | Where | Why |
|------|-------|-----|
| Business entity | `domain/entities/` | Core business model |
| Business rule | `domain/usecases/` | Business operation |
| API call | `data/usecases/` | Data retrieval |
| JSON parsing | `data/models/` | Data transformation |
| HTTP client | `infra/http/` | External library adapter |
| Screen logic | `presentation/presenters/` | Presentation logic |
| Widget | `ui/pages/` or `ui/components/` | Visual component |
| Form validation | `validation/validators/` | Validation logic |
| Dependency wiring | `main/factories/` | Composition root |

#### "How do I...?"

| Task | Answer | Reference |
|------|--------|-----------|
| Add a new page | Create in UI, Presenter, Factory | See: [Getting Started](06-getting-started-guide.md) |
| Make an API call | Create use case in Data layer | See: [Layer Breakdown](02-layer-breakdown.md) |
| Add validation | Create validator, use ValidationBuilder | See: [Design Patterns](03-design-patterns.md) |
| Show loading state | Use LoadingManager mixin | See: [Component Deep-Dive](04-component-deep-dive.md) |
| Navigate to screen | Emit navigation event in presenter | See: [Data Flow](05-data-flow-guide.md) |
| Test a presenter | Mock dependencies, verify calls | See: [Testing Strategy](07-testing-strategy.md) |

---

### Architecture Decision Tree

**When adding a new feature, follow this decision tree:**

```
┌─────────────────────────┐
│  New Feature Request    │
└───────────┬─────────────┘
            │
            ↓
┌─────────────────────────────────┐
│ Does it involve business logic? │
└──────┬──────────────────┬───────┘
       │ Yes              │ No
       ↓                  ↓
┌──────────────┐   ┌─────────────┐
│ Add to Domain│   │ Add to UI   │
└──────┬───────┘   └─────────────┘
       │
       ↓
┌────────────────────────────┐
│ Does it need external data?│
└─────┬─────────────┬────────┘
      │ Yes         │ No
      ↓             ↓
┌─────────────┐  ┌────────────────┐
│Add Data/Infra│  │Use existing data│
└─────┬────────┘  └────────────────┘
      │
      ↓
┌──────────────────┐
│ Add Presentation │
└────────┬─────────┘
         │
         ↓
┌──────────────┐
│   Add UI     │
└──────┬───────┘
       │
       ↓
┌──────────────────┐
│ Wire in Main     │
└──────────────────┘
```

---

### Layer Cheat Sheet

**Quick reminder of layer purposes:**

```
┌─────────────────────────────────────────────────────────────┐
│ DOMAIN: "What" the business does                            │
│ - Entities, Use Cases, Business Rules                       │
│ - Zero dependencies                                         │
│ - Framework agnostic                                        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ DATA: "How" to get/store data                               │
│ - Use case implementations                                  │
│ - Data models & transformations                             │
│ - Defines protocols for Infra layer                         │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ INFRA: Adapters for external libraries                      │
│ - HTTP clients, Storage, etc.                               │
│ - Implements Data protocols                                 │
│ - Wraps third-party packages                                │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ PRESENTATION: Screen logic                                  │
│ - Presenters with reactive streams                          │
│ - Input validation coordination                             │
│ - State management                                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ UI: Visual components                                       │
│ - Pages, Widgets, Components                                │
│ - Listens to presenter streams                              │
│ - "Dumb" - no business logic                                │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ VALIDATION: Form validation                                 │
│ - Validators (Required, Email, etc.)                        │
│ - Standalone, reusable                                      │
│ - Used by Presentation layer                                │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ MAIN: Dependency injection                                  │
│ - Factories for creating objects                            │
│ - Composition root                                          │
│ - Knows about all layers                                    │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

### What Makes This Architecture Special

1. **Separation of Concerns**
   - Each layer has ONE job
   - Changes are isolated
   - Easy to understand and modify

2. **Dependency Inversion**
   - Inner layers define contracts
   - Outer layers implement
   - Business logic is independent

3. **Testability**
   - Every component is testable
   - Dependencies are mockable
   - Tests run fast (no external dependencies)

4. **Maintainability**
   - Clear structure
   - Easy to find things
   - Safe to refactor

5. **Scalability**
   - Add features without breaking existing code
   - Multiple developers can work in parallel
   - Framework-independent core

### The Big Picture

**Think of the architecture like a well-organized company:**

- **Domain Layer** = CEO (defines business strategy)
- **Data Layer** = Departments (execute strategy)
- **Infra Layer** = Tools & Equipment (resources used)
- **Presentation Layer** = Managers (coordinate work)
- **UI Layer** = Customer Service (interact with users)
- **Main Layer** = HR (assigns people to jobs)

Each has a clear role, and they communicate through well-defined interfaces!

---

## Next Steps

Now that you understand the overall architecture, dive deeper into specific topics:

1. **[Layer Breakdown](02-layer-breakdown.md)** - Detailed explanation of each layer
2. **[Design Patterns](03-design-patterns.md)** - All patterns used in the project
3. **[Component Deep-Dive](04-component-deep-dive.md)** - How key features work
4. **[Data Flow Guide](05-data-flow-guide.md)** - Follow data through the app
5. **[Getting Started Guide](06-getting-started-guide.md)** - Build your first feature
6. **[Testing Strategy](07-testing-strategy.md)** - Write effective tests

---

### Recommended Reading Order

**For Beginners:**
1. This document (Architecture Overview) ✅ You are here!
2. [Getting Started Guide](06-getting-started-guide.md) - Hands-on tutorial
3. [Data Flow Guide](05-data-flow-guide.md) - See it in action
4. [Layer Breakdown](02-layer-breakdown.md) - Deep dive into layers

**For Experienced Developers:**
1. This document (Architecture Overview) ✅ You are here!
2. [Design Patterns](03-design-patterns.md) - Pattern catalog
3. [Component Deep-Dive](04-component-deep-dive.md) - Implementation details
4. [Testing Strategy](07-testing-strategy.md) - Testing approach

---

### Additional Resources

**Clean Architecture:**
- 📘 "Clean Architecture" by Robert C. Martin (Uncle Bob)
- 🎥 [Clean Architecture Talk by Uncle Bob](https://www.youtube.com/watch?v=o_TH-Y78tt4)

**Flutter Architecture:**
- 📘 [Flutter Architecture Samples](https://github.com/brianegan/flutter_architecture_samples)
- 📘 [Reso Coder's Flutter TDD Clean Architecture](https://resocoder.com/flutter-clean-architecture-tdd/)

**SOLID Principles:**
- 📘 [SOLID Principles in Flutter](https://medium.com/flutter-community/solid-principles-in-flutter-3c76b7bc98f0)

---

## Glossary

**Key Terms:**

- **Entity** - A domain object that represents a business concept
- **Use Case** - A business operation or action
- **Repository** - An abstraction for data access
- **Presenter** - Coordinates between UI and business logic
- **ViewModel** - Data structure optimized for UI display
- **Adapter** - Wraps an external library to match an interface
- **Factory** - Creates and configures objects
- **Dependency Injection** - Providing dependencies from outside
- **Composition Root** - Where dependency injection happens
- **Stream** - A sequence of asynchronous events

---

**Documentation Version:** 1.0
**Last Updated:** 2025
**Author:** Clean Flutter App Team

---

**Questions or suggestions?** Open an issue or contribute to improve this documentation!

**Happy coding!** 🚀
