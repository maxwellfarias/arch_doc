# Flutter Testing Tutorial

A comprehensive guide to testing Flutter applications following Clean Architecture principles.

## 📚 Tutorial Contents

| # | Chapter | Description |
|---|---------|-------------|
| 1 | [Testing Overview](01_TESTING_OVERVIEW.md) | Introduction, project structure, test types, dependencies |
| 2 | [Mocking Strategies](02_MOCKING_STRATEGIES.md) | Spy pattern, Mock pattern, Fakes, RxDart stream mocking |
| 3 | [Domain Testing](03_DOMAIN_TESTING.md) | Testing pure business logic, entities, helper functions |
| 4 | [Infrastructure Testing](04_INFRA_TESTING.md) | Repositories, adapters, mappers, error handling |
| 5 | [Presentation Testing](05_PRESENTATION_TESTING.md) | Reactive presenters, stream testing, ViewModels |
| 6 | [UI Testing](06_UI_TESTING.md) | Widget tests, component tests, page tests, interactions |
| 7 | [E2E Testing](07_E2E_TESTING.md) | End-to-end integration tests, full flow verification |
| 8 | [Best Practices](08_BEST_PRACTICES.md) | Patterns, guidelines, common pitfalls |
| 9 | [Quick Reference](09_QUICK_REFERENCE.md) | Cheatsheet for assertions, matchers, utilities |

## 🎯 Learning Path

### Beginner
Start with chapters 1-3 to understand the basics:
- How tests are organized
- Basic mocking concepts
- Testing pure functions

### Intermediate
Continue with chapters 4-6:
- Infrastructure layer testing with dependencies
- Reactive presenter testing
- Widget and UI testing

### Advanced
Complete with chapters 7-9:
- End-to-end integration testing
- Best practices and patterns
- Quick reference for daily use

## 🏗️ Project Architecture

This tutorial is based on a Flutter project following **Clean Architecture**:

```
┌─────────────────────────────────────┐
│            UI Layer                 │
│      (Pages, Components)            │
├─────────────────────────────────────┤
│       Presentation Layer            │
│    (Presenters, ViewModels)         │
├─────────────────────────────────────┤
│      Infrastructure Layer           │
│ (Repositories, Adapters, Mappers)   │
├─────────────────────────────────────┤
│          Domain Layer               │
│      (Entities, Use Cases)          │
└─────────────────────────────────────┘
```

## 🧪 Test Types Covered

| Type | Speed | Confidence | Coverage |
|------|-------|------------|----------|
| **Unit Tests** | ⚡ Fast | Medium | Individual functions/classes |
| **Widget Tests** | 🏃 Medium | Medium-High | UI components |
| **E2E Tests** | 🐢 Slow | High | Complete user flows |

## 🔧 Key Patterns Demonstrated

- **AAA Pattern** (Arrange-Act-Assert)
- **Spy Pattern** for verification
- **Fake Data Generation** for robust tests
- **Stream Testing** with RxDart
- **Network Image Mocking**
- **Pull-to-Refresh Testing**
- **Cache Fallback Testing**

## 📦 Dependencies Used

```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  file: ^7.0.1
  network_image_mock: ^2.1.1
  flutter_lints: ^5.0.0
```

## 🚀 Quick Start

1. Read the [Testing Overview](01_TESTING_OVERVIEW.md)
2. Understand [Mocking Strategies](02_MOCKING_STRATEGIES.md)
3. Follow the progressive chapters
4. Use the [Quick Reference](09_QUICK_REFERENCE.md) as a cheatsheet

## 📖 Source Project

This tutorial analyzes the test structure from the `advanced-flutter` project by Rodrigo Manguinho, demonstrating professional testing patterns in a real-world application.

---

Happy Testing! 🎉
