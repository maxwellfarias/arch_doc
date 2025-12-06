# Flutter Testing Tutorial: Complete Guide

## Introduction

Welcome to this comprehensive Flutter testing tutorial! This guide is based on a real-world Flutter project that follows **Clean Architecture** principles and demonstrates professional testing strategies used in production applications.

By the end of this tutorial, you will understand:
- How to structure tests in a Flutter application
- Different types of tests (Unit, Widget, Integration, E2E)
- Mocking strategies (Spies, Mocks, Fakes)
- Testing reactive streams with RxDart
- Widget testing techniques
- Best practices for maintainable tests

---

## Table of Contents

1. **[Testing Overview](01_TESTING_OVERVIEW.md)** (You are here)
   - Project Architecture
   - Test Types
   - Dependencies Setup

2. **[Mocking Strategies](02_MOCKING_STRATEGIES.md)**
   - Spy Pattern
   - Mock Pattern
   - Fake Data Generation
   - RxDart Stream Mocking

3. **[Domain Layer Testing](03_DOMAIN_TESTING.md)**
   - Testing Pure Business Logic
   - Entity Testing
   - Helper Functions Pattern

4. **[Infrastructure Layer Testing](04_INFRA_TESTING.md)**
   - Repository Testing
   - Adapter Testing (HTTP, Cache)
   - Mapper Testing
   - Error Handling

5. **[Presentation Layer Testing](05_PRESENTATION_TESTING.md)**
   - Reactive Presenter Testing
   - ViewModel Testing
   - Stream Assertions

6. **[UI/Widget Testing](06_UI_TESTING.md)**
   - Component Testing
   - Page Testing
   - Gesture Simulation
   - Network Image Mocking

7. **[End-to-End Testing](07_E2E_TESTING.md)**
   - Integration Testing
   - Full Flow Verification
   - Fallback Behavior Testing

8. **[Best Practices](08_BEST_PRACTICES.md)**
   - Summary of Patterns
   - Common Pitfalls
   - Guidelines

9. **[Quick Reference](09_QUICK_REFERENCE.md)**
   - Cheatsheet
   - Common Assertions
   - Utility Functions

---

## Project Architecture Overview

This project follows **Clean Architecture** with four main layers:

```
┌─────────────────────────────────────────────────────────────┐
│                         UI Layer                            │
│              (Pages, Components, Widgets)                   │
├─────────────────────────────────────────────────────────────┤
│                    Presentation Layer                       │
│                (Presenters, ViewModels)                     │
├─────────────────────────────────────────────────────────────┤
│                   Infrastructure Layer                      │
│         (Repositories, Adapters, Mappers, Clients)         │
├─────────────────────────────────────────────────────────────┤
│                      Domain Layer                           │
│                  (Entities, Use Cases)                      │
└─────────────────────────────────────────────────────────────┘
```

### Layer Responsibilities

| Layer | Responsibility | What We Test |
|-------|---------------|--------------|
| **Domain** | Business logic, entities | Pure functions, entity behavior |
| **Infrastructure** | Data access, API calls, caching | Repositories, adapters, mappers |
| **Presentation** | State management, UI logic | Presenters, streams, viewmodels |
| **UI** | Visual components | Widget rendering, user interactions |

---

## Project Test Structure

```
test/
├── domain/
│   ├── entities/
│   │   └── next_event_player_test.dart    # Entity tests
│   └── mocks/
│       └── next_event_loader_spy.dart     # Domain mocks
├── infra/
│   ├── api/
│   │   ├── adapters/
│   │   │   └── http_adapter_test.dart     # HTTP adapter tests
│   │   ├── clients/
│   │   │   └── authorized_http_get_client_test.dart
│   │   ├── mocks/
│   │   │   ├── client_spy.dart
│   │   │   └── http_get_client_spy.dart
│   │   └── repositories/
│   │       └── load_next_event_api_repo_test.dart
│   ├── cache/
│   │   ├── adapters/
│   │   │   └── cache_manager_adapter_test.dart
│   │   ├── mocks/
│   │   │   ├── cache_get_client_spy.dart
│   │   │   ├── cache_manager_spy.dart
│   │   │   ├── cache_save_client_mock.dart
│   │   │   └── file_spy.dart
│   │   └── repositories/
│   │       └── load_next_event_cache_repo_test.dart
│   ├── mappers/
│   │   ├── next_event_mapper_test.dart
│   │   └── next_event_player_mapper_test.dart
│   ├── mocks/
│   │   ├── list_mapper_spy.dart
│   │   ├── load_next_event_repo_spy.dart
│   │   └── mapper_spy.dart
│   └── repositories/
│       └── load_next_event_from_api_with_cache_fallback_repo_test.dart
├── presentation/
│   ├── mocks/
│   │   └── next_event_presenter_spy.dart
│   ├── rx/
│   │   └── next_event_rx_presenter_test.dart
│   └── viewmodels/
│       └── next_event_player_viewmodel_test.dart
├── ui/
│   ├── components/
│   │   ├── player_photo_test.dart
│   │   ├── player_position_test.dart
│   │   └── player_status_test.dart
│   └── pages/
│       └── next_event_page_test.dart
├── e2e/
│   └── next_event_page_test.dart          # End-to-end tests
└── mocks/
    └── fakes.dart                          # Shared fake data utilities
```

---

## Types of Tests

### 1. Unit Tests
Test individual functions, methods, or classes in isolation.

**Example**: Testing that an entity correctly calculates initials from a name.

```dart
test('should return the first letter of the first and last names', () {
  expect(initialsOf('Rodrigo Manguinho'), 'RM');
});
```

### 2. Widget Tests
Test individual Flutter widgets or small widget trees.

**Example**: Testing that a status indicator shows the correct color.

```dart
testWidgets('should present green status', (tester) async {
  await tester.pumpWidget(
    const MaterialApp(home: PlayerStatus(isConfirmed: true))
  );
  final decoration = tester.firstWidget<Container>(
    find.byType(Container)
  ).decoration as BoxDecoration;
  expect(decoration.color, Colors.teal);
});
```

### 3. Integration Tests
Test multiple components working together.

**Example**: Testing repository with real mapper but mocked HTTP client.

### 4. End-to-End Tests
Test complete user flows from UI to data layer.

**Example**: Testing that a page displays data from API, with cache fallback on error.

---

## Testing Dependencies

Add these to your `pubspec.yaml`:

```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  file: ^7.0.1              # File system mocking
  flutter_lints: ^5.0.0     # Code quality
  network_image_mock: ^2.1.1 # Network image mocking for widgets
```

### What Each Dependency Provides

| Package | Purpose |
|---------|---------|
| `flutter_test` | Core testing framework (testWidgets, expect, find, etc.) |
| `file` | Mock file system operations for cache testing |
| `network_image_mock` | Prevents network image loading errors in tests |
| `flutter_lints` | Enforces code quality standards |

---

## Core Testing Concepts

### The AAA Pattern (Arrange-Act-Assert)

Every test follows this structure:

```dart
test('should return correct data', () async {
  // ARRANGE - Set up test conditions
  final sut = MyClass(dependency: mock);
  mock.response = expectedData;

  // ACT - Execute the code under test
  final result = await sut.doSomething();

  // ASSERT - Verify the results
  expect(result, expectedData);
});
```

### SUT (System Under Test)

The class or function being tested is always named `sut`:

```dart
late MyRepository sut;  // Clear naming convention

setUp(() {
  sut = MyRepository(client: mockClient);
});
```

### setUp() for Test Initialization

Use `setUp()` to initialize fresh instances before each test:

```dart
void main() {
  late MyClass sut;
  late MockDependency mock;

  setUp(() {
    mock = MockDependency();
    sut = MyClass(dependency: mock);
  });

  test('test 1', () { /* uses fresh sut and mock */ });
  test('test 2', () { /* uses fresh sut and mock */ });
}
```

---

## Test Naming Convention

Use descriptive names that explain the expected behavior:

```dart
// ✅ Good - Clear intent
test('should return the first letter of the first and last names', () {});
test('should throw UnexpectedError when response is null', () {});
test('should emit loading state before fetching data', () {});

// ❌ Bad - Vague or technical
test('test initials', () {});
test('null check', () {});
test('loading', () {});
```

**Format**: `should [expected behavior] [when condition]`

---

## Running Tests

### Run All Tests
```bash
flutter test
```

### Run Specific Test File
```bash
flutter test test/domain/entities/next_event_player_test.dart
```

### Run Tests with Coverage
```bash
flutter test --coverage
```

### Run Tests in Watch Mode (with hot reload)
```bash
flutter test --watch
```

---

## Next Steps

Now that you understand the testing structure and core concepts, proceed to:

**[→ Mocking Strategies](02_MOCKING_STRATEGIES.md)**: Learn how to create Spies, Mocks, and Fakes to isolate your tests.

---

## Summary

| Concept | Description |
|---------|-------------|
| **Clean Architecture** | Separates concerns into Domain, Infra, Presentation, UI layers |
| **AAA Pattern** | Arrange, Act, Assert structure for all tests |
| **SUT** | System Under Test - the class being tested |
| **setUp()** | Initializes fresh test instances |
| **Test Naming** | `should [behavior] [condition]` format |
