# Architecture Overview - Flutter Advanced TDD & Clean Architecture

## Table of Contents
- [Introduction](#introduction)
- [What is Clean Architecture?](#what-is-clean-architecture)
- [High-Level System Design](#high-level-system-design)
- [The 4-Layer Architecture](#the-4-layer-architecture)
- [Project Structure](#project-structure)
- [Technology Stack](#technology-stack)
- [Dependency Flow](#dependency-flow)
- [Why This Architecture?](#why-this-architecture)

---

## Introduction

Welcome to the **Flutter Advanced TDD & Clean Architecture** project! This is an educational Flutter application that demonstrates professional software engineering practices used in real-world production applications.

### What Does This App Do?

The application displays information about a sports event (soccer game) showing:
- Players who confirmed attendance (separated by goalkeepers and field players)
- Players who cannot attend
- Players who haven't decided yet

**But the real value isn't what it does—it's HOW it's built!** This project is a learning resource demonstrating:
- Clean Architecture principles
- Test-Driven Development (TDD)
- SOLID design principles
- Advanced Flutter patterns
- Professional code organization

---

## What is Clean Architecture?

### The Building Analogy

Imagine you're building a house with multiple floors:

```
┌─────────────────────────────────────┐
│  4th Floor: UI (Beautiful Rooms)    │  ← What users see
├─────────────────────────────────────┤
│  3rd Floor: Presentation (Furniture)│  ← How data is arranged for display
├─────────────────────────────────────┤
│  2nd Floor: Infrastructure (Pipes)  │  ← How we get water/electricity
├─────────────────────────────────────┤
│  1st Floor: Domain (Foundation)     │  ← Core business rules
└─────────────────────────────────────┘
```

**Key Principle**: Upper floors depend on lower floors, but lower floors never depend on upper floors!

- You can **redecorate a room** (change UI) without touching the plumbing (infrastructure)
- You can **replace the pipes** (switch from one API to another) without changing the foundation (business rules)
- The **foundation** (domain) stands alone—it doesn't care about the floors above

This separation makes your code:
- **Testable**: Each layer can be tested independently
- **Maintainable**: Changes in one layer don't break others
- **Flexible**: You can swap implementations easily
- **Understandable**: Clear responsibilities for each part

---

## High-Level System Design

### The Application Flow

Here's how the app works from a user's perspective:

```
User Opens App
      ↓
Shows Loading Spinner
      ↓
Fetches Event Data from API
      ↓
Saves Data to Cache (for offline use)
      ↓
Processes & Categorizes Players
      ↓
Displays Player Lists on Screen
      ↓
User Can Pull-to-Refresh
```

### Behind the Scenes Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        USER INTERFACE                        │
│  (Flutter Widgets: Buttons, Lists, Images)                   │
└────────────────────┬─────────────────────────────────────────┘
                     │ User Actions (Tap, Scroll, Refresh)
                     ↓
┌──────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                        │
│  (Manages UI State, Formats Data for Display)                │
│  - Receives user actions                                     │
│  - Requests data                                             │
│  - Transforms data into UI-friendly format                   │
│  - Sends updates to UI via Streams                           │
└────────────────────┬─────────────────────────────────────────┘
                     │ Requests: "Get next event"
                     ↓
┌──────────────────────────────────────────────────────────────┐
│                   INFRASTRUCTURE LAYER                       │
│  (Handles All External Communication)                        │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐     │
│  │ API          │  │ Cache         │  │ Mappers      │     │
│  │ Repository   │  │ Repository    │  │ (JSON ↔ Data)│     │
│  └──────┬───────┘  └───────┬───────┘  └──────────────┘     │
└─────────┼──────────────────┼────────────────────────────────┘
          │                  │
          ↓                  ↓
   ┌──────────┐       ┌──────────┐
   │ REST API │       │ Local    │
   │ (Backend)│       │ Storage  │
   └──────────┘       └──────────┘
                     ↑
                     │ All layers use these
┌──────────────────────────────────────────────────────────────┐
│                       DOMAIN LAYER                           │
│  (Pure Business Logic - No Dependencies)                     │
│  - Event entity (what is an event?)                          │
│  - Player entity (what is a player?)                         │
│  - Business rules (how to generate initials?)                │
│  - Errors (what can go wrong?)                               │
└──────────────────────────────────────────────────────────────┘
```

---

## The 4-Layer Architecture

### Layer 1: Domain (The Core)

**Location**: `lib/domain/`

**Think of it as**: The heart of your application—pure business logic

**Contains**:
- **Entities**: The main "things" in your app (Event, Player)
- **Business Rules**: Logic that defines how your business works
- **Errors**: Things that can go wrong in your business

**Key Characteristics**:
- ✅ No dependencies on other layers
- ✅ No Flutter imports
- ✅ No external packages (pure Dart)
- ✅ Fully testable without any framework

**Example**:
```dart
// lib/domain/entities/next_event_player.dart
final class NextEventPlayer {
  final String name;
  final String initials;  // Auto-generated: "Maxwell Farias" → "MF"
  final bool? isConfirmed;
  final String? position;
}
```

---

### Layer 2: Infrastructure (The Workers)

**Location**: `lib/infra/`

**Think of it as**: The layer that does the "dirty work" of fetching and storing data

**Contains**:
- **Repositories**: Coordinate data fetching from different sources
- **Adapters**: Wrap external libraries (HTTP client, Cache manager)
- **Mappers**: Convert JSON ↔ Domain entities
- **Clients**: Define interfaces for external services

**Key Characteristics**:
- ✅ Depends only on Domain layer
- ✅ Uses external packages (http, cache_manager)
- ✅ Handles all data source communication
- ✅ Converts external data to domain entities

**Example**:
```dart
// lib/infra/api/repositories/load_next_event_api_repo.dart
class LoadNextEventApiRepository {
  final HttpGetClient httpClient;
  final DtoMapper<NextEvent> mapper;

  Future<NextEvent> loadNextEvent({required String groupId}) async {
    final json = await httpClient.get(url: url, params: {"groupId": groupId});
    return mapper.toDto(json);  // JSON → NextEvent entity
  }
}
```

---

### Layer 3: Presentation (The Organizer)

**Location**: `lib/presentation/`

**Think of it as**: The smart assistant that prepares data specifically for the UI

**Contains**:
- **Presenters**: Coordinate data loading and UI updates
- **ViewModels**: UI-specific data structures
- **Business Logic for UI**: Filtering, sorting, categorizing

**Key Characteristics**:
- ✅ Depends only on Domain layer
- ✅ No Flutter widgets (testable without Flutter)
- ✅ Uses RxDart Streams for reactive updates
- ✅ Transforms domain entities to ViewModels

**Example**:
```dart
// lib/presentation/rx/next_event_rx_presenter.dart
final class NextEventRxPresenter implements NextEventPresenter {
  final nextEventSubject = BehaviorSubject<NextEventViewModel>();

  Future<void> loadNextEvent({required String groupId}) async {
    final event = await nextEventLoader(groupId: groupId);
    nextEventSubject.add(_mapEvent(event));  // Transform & emit
  }

  NextEventViewModel _mapEvent(NextEvent event) {
    // Filter players into categories: goalkeepers, players, out, doubt
    // Sort each category
    // Return UI-ready ViewModel
  }
}
```

---

### Layer 4: UI (The Face)

**Location**: `lib/ui/`

**Think of it as**: Everything the user sees and interacts with

**Contains**:
- **Pages**: Full screens (StatefulWidget)
- **Components**: Reusable widgets (ListSection, PlayerPhoto, etc.)

**Key Characteristics**:
- ✅ Depends on Presentation layer
- ✅ Contains only Flutter widgets
- ✅ Listens to Presenter's streams
- ✅ Displays data, handles user input

**Example**:
```dart
// lib/ui/pages/next_event_page.dart
final class NextEventPage extends StatefulWidget {
  final NextEventPresenter presenter;

  @override
  Widget build(BuildContext context) {
    return StreamBuilder<NextEventViewModel>(
      stream: presenter.nextEventStream,
      builder: (context, snapshot) {
        final viewModel = snapshot.data!;
        return ListView(
          children: [
            ListSection(title: 'GOALKEEPERS', items: viewModel.goalkeepers),
            ListSection(title: 'PLAYERS', items: viewModel.players),
            // ... more sections
          ],
        );
      },
    );
  }
}
```

---

### Composition Root (The Assembler)

**Location**: `lib/main/`

**Think of it as**: The factory that builds and connects all pieces

**Contains**:
- **main.dart**: Application entry point
- **Factories**: Functions that create and wire dependencies

**Key Characteristics**:
- ✅ Only place where all layers come together
- ✅ Manual dependency injection (no framework)
- ✅ Creates object graph

**Example**:
```dart
// lib/main/factories/ui/pages/next_event_page_factory.dart
Widget makeNextEventPage() {
  final repo = makeLoadNextEventFromApiWithCacheFallbackRepository();
  final presenter = NextEventRxPresenter(
    nextEventLoader: repo.loadNextEvent
  );
  return NextEventPage(presenter: presenter, groupId: 'valid_id');
}
```

---

## Project Structure

### Complete Directory Tree

```
fluttter_avancado_tdd/
├── fluttter_avancado_tdd_clean_arch/           # Main Flutter project
│   │
│   ├── lib/                                     # Application source code
│   │   │
│   │   ├── domain/                              # 🎯 LAYER 1: Business Logic
│   │   │   └── entities/
│   │   │       ├── errors.dart                  # Domain errors (Sealed)
│   │   │       ├── next_event.dart              # Event entity
│   │   │       └── next_event_player.dart       # Player entity
│   │   │
│   │   ├── infra/                               # 🔧 LAYER 2: Data Management
│   │   │   ├── api/                             # HTTP communication
│   │   │   │   ├── adapters/
│   │   │   │   │   └── http_adapter.dart        # Wraps 'http' package
│   │   │   │   ├── clients/
│   │   │   │   │   └── http_get_client.dart     # HTTP client interface
│   │   │   │   └── repositories/
│   │   │   │       └── load_next_event_api_repo.dart
│   │   │   │
│   │   │   ├── cache/                           # Local storage
│   │   │   │   ├── adapters/
│   │   │   │   │   └── cache_manager_adapter.dart
│   │   │   │   ├── clients/
│   │   │   │   │   ├── cache_get_client.dart
│   │   │   │   │   └── cache_save_client.dart
│   │   │   │   └── repositories/
│   │   │   │       └── load_next_event_cache_repo.dart
│   │   │   │
│   │   │   ├── mappers/                         # JSON ↔ Entity conversion
│   │   │   │   ├── mapper.dart                  # Mapper interfaces
│   │   │   │   ├── next_event_mapper.dart
│   │   │   │   └── next_event_player_mapper.dart
│   │   │   │
│   │   │   ├── repositories/                    # Composite repositories
│   │   │   │   └── load_next_event_from_api_with_cache_fallback_repo.dart
│   │   │   │
│   │   │   └── types/                           # Type definitions
│   │   │       └── json.dart
│   │   │
│   │   ├── presentation/                        # 🧠 LAYER 3: UI Business Logic
│   │   │   ├── presenters/
│   │   │   │   └── next_event_presenter.dart    # Interface & ViewModels
│   │   │   └── rx/
│   │   │       └── next_event_rx_presenter.dart # RxDart implementation
│   │   │
│   │   ├── ui/                                  # 🎨 LAYER 4: User Interface
│   │   │   ├── components/                      # Reusable widgets
│   │   │   │   ├── list_section.dart
│   │   │   │   ├── player_photo.dart
│   │   │   │   ├── player_position.dart
│   │   │   │   └── player_status.dart
│   │   │   └── pages/
│   │   │       └── next_event_page.dart         # Main screen
│   │   │
│   │   └── main/                                # 🏭 Composition Root
│   │       ├── main.dart                        # App entry point
│   │       └── factories/                       # Dependency injection
│   │           ├── infra/
│   │           │   ├── api/
│   │           │   ├── cache/
│   │           │   ├── mappers/
│   │           │   └── repositories/
│   │           └── ui/
│   │               └── pages/
│   │
│   ├── test/                                    # 🧪 Test suite (mirrors lib/)
│   │   ├── domain/
│   │   ├── infra/
│   │   ├── presentation/
│   │   ├── ui/
│   │   ├── e2e/                                 # End-to-end tests
│   │   └── mocks/                               # Test utilities
│   │
│   ├── backend/                                 # 🖥️ Mock Express.js server
│   │   ├── index.js                             # REST API
│   │   └── package.json
│   │
│   ├── android/                                 # Android platform
│   ├── ios/                                     # iOS platform
│   ├── web/                                     # Web platform
│   ├── linux/                                   # Linux platform
│   ├── macos/                                   # macOS platform
│   ├── windows/                                 # Windows platform
│   │
│   └── pubspec.yaml                             # Dependencies
│
└── README.md                                    # Project documentation
```

---

## Technology Stack

### Core Technologies

| Technology | Version | Purpose | Layer |
|------------|---------|---------|-------|
| **Flutter** | Latest | Cross-platform UI framework | UI |
| **Dart** | 3.0+ | Programming language | All |
| **RxDart** | ^0.28.0 | Reactive state management | Presentation |
| **HTTP** | ^1.2.2 | REST API communication | Infrastructure |
| **Flutter Cache Manager** | ^3.4.1 | Local cache storage | Infrastructure |

### Utility Libraries

| Library | Purpose | Where Used |
|---------|---------|------------|
| **dartx** | Dart extensions (`.sortedBy()`, `.removeSuffix()`) | Infrastructure, Presentation |
| **awesome_flutter_extensions** | UI context extensions | UI |
| **network_image_mock** | Mock network images in tests | Tests |

### Development Tools

| Tool | Purpose |
|------|---------|
| **flutter_test** | Widget & unit testing framework |
| **flutter_lints** | Dart/Flutter linting rules |

### Backend (Development)

| Technology | Purpose |
|-----------|---------|
| **Node.js + Express** | Mock REST API server |

---

## Dependency Flow

### The Dependency Rule

**Core Principle**: Dependencies always point INWARD (toward the domain)

```
┌─────────────────────────────────────────────────────┐
│                    UI Layer                         │
│  (Depends on: Presentation)                         │
└────────────────────┬────────────────────────────────┘
                     │ depends on
                     ↓
┌─────────────────────────────────────────────────────┐
│              Presentation Layer                     │
│  (Depends on: Domain only)                          │
└────────────────────┬────────────────────────────────┘
                     │ depends on
    ┌────────────────┴────────────────┐
    ↓                                 ↓
┌──────────────────────────┐  ┌──────────────────────┐
│  Infrastructure Layer    │  │    Domain Layer      │
│  (Depends on: Domain)    │←─│  (Depends on: NONE)  │
└──────────────────────────┘  └──────────────────────┘
```

### What This Means

✅ **UI** can import from: Presentation
✅ **Presentation** can import from: Domain
✅ **Infrastructure** can import from: Domain
❌ **Domain** NEVER imports from other layers

### Why This Matters

1. **Domain is Portable**: Pure business logic can run anywhere (mobile, web, server)
2. **Easy Testing**: Test each layer independently
3. **Flexible Implementation**: Swap infrastructure without touching business logic
4. **Clear Boundaries**: Prevents circular dependencies

### Example

```dart
// ✅ ALLOWED: UI imports Presentation
import 'package:fluttter_avancado_tdd_clean_arch/presentation/presenters/next_event_presenter.dart';

// ✅ ALLOWED: Presentation imports Domain
import 'package:fluttter_avancado_tdd_clean_arch/domain/entities/next_event.dart';

// ✅ ALLOWED: Infrastructure imports Domain
import 'package:fluttter_avancado_tdd_clean_arch/domain/entities/next_event.dart';

// ❌ FORBIDDEN: Domain imports Infrastructure
import 'package:fluttter_avancado_tdd_clean_arch/infra/api/adapters/http_adapter.dart';
// This would create coupling!
```

---

## Why This Architecture?

### Benefits for Learning

1. **Clear Structure**: Easy to find where things belong
2. **Testability**: Each piece can be tested in isolation
3. **Real-World Patterns**: Used in professional applications
4. **SOLID Principles**: See them in action, not just theory
5. **Best Practices**: Industry-standard approach

### Benefits for Production

1. **Maintainability**: Changes are localized and safe
2. **Scalability**: Add features without breaking existing code
3. **Team Collaboration**: Clear boundaries for team members
4. **Flexibility**: Swap implementations (e.g., switch from REST to GraphQL)
5. **Quality**: Comprehensive testing at every layer

### Comparison with Simple Architecture

#### Simple Approach (Not Clean Architecture)
```
lib/
├── screens/
│   └── home_screen.dart        # UI + API calls + business logic + state
├── models/
│   └── player.dart             # Just data classes
└── api/
    └── api_service.dart        # All HTTP logic
```

**Problems**:
- Business logic mixed with UI
- Hard to test (need to mock widgets)
- Difficult to reuse logic
- Tightly coupled components

#### Clean Architecture Approach (This Project)
```
lib/
├── domain/        # Pure business logic (reusable, testable)
├── infra/         # Data sources (swappable)
├── presentation/  # UI logic (testable without Flutter)
└── ui/            # Widgets only (simple, focused)
```

**Benefits**:
- Clear separation of concerns
- Easy to test each part
- Reusable business logic
- Loosely coupled components

---

## Next Steps

Now that you understand the overall architecture, dive deeper into:

1. **[Layer Breakdown](02-LAYER-BREAKDOWN.md)**: Detailed explanation of each layer
2. **[Design Patterns](03-DESIGN-PATTERNS.md)**: Patterns used throughout the project
3. **[Component Deep-Dive](04-COMPONENT-DEEP-DIVE.md)**: How components interact
4. **[Getting Started Guide](05-GETTING-STARTED-GUIDE.md)**: Begin working with the code

---

## Key Takeaways

✅ **Clean Architecture** separates concerns into 4 distinct layers
✅ **Domain** is the core—pure business logic with no dependencies
✅ **Infrastructure** handles external communication (API, cache)
✅ **Presentation** prepares data for the UI (filtering, sorting, formatting)
✅ **UI** displays data and handles user interaction
✅ **Dependency flow** always points inward (toward domain)
✅ **This structure** makes code testable, maintainable, and flexible

Welcome to professional Flutter development! 🚀
