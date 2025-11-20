# 01 - Architecture Overview

## 📋 Table of Contents

- [Introduction](#introduction)
- [Executive Summary](#executive-summary)
- [High-Level System Design](#high-level-system-design)
- [Architectural Patterns](#architectural-patterns)
- [Project Structure](#project-structure)
- [Environment Configurations](#environment-configurations)
- [Technology Stack](#technology-stack)
- [Key Architectural Decisions](#key-architectural-decisions)
- [Architecture Benefits](#architecture-benefits)

---

## Introduction

Welcome to the **Compass App** architecture documentation! This document provides a comprehensive overview of how the application is designed and structured.

**What is Compass App?**
Compass App is a travel itinerary booking application built with Flutter. It allows users to search for destinations, select activities, and create complete travel bookings. Think of it as a simplified version of apps like TripAdvisor or Airbnb Experiences, but focused on educational purposes to demonstrate Flutter best practices.

**Why is this documentation important?**
Understanding the architecture helps you:
- Navigate the codebase confidently
- Make changes without breaking existing functionality
- Follow established patterns when adding new features
- Understand how different parts of the app communicate

---

## Executive Summary

### Quick Facts

| Metric | Value |
|--------|-------|
| **Total Lines of Code** | ~9,155 lines |
| **Number of Dart Files** | 110 files |
| **Architectural Pattern** | Clean Architecture + MVVM |
| **State Management** | Provider + ChangeNotifier |
| **Navigation** | go_router (declarative) |
| **Code Generation** | Freezed (models) + json_serializable |
| **Backend** | Dart Shelf server (optional) |

### Core Architecture

The Compass App follows **Clean Architecture** principles with three distinct layers:

```
┌─────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                    │
│           (What the user sees and interacts with)        │
│     Screens, Widgets, ViewModels, UI Components          │
└──────────────────┬──────────────────────────────────────┘
                   │
                   │ Uses models and repositories
                   │
┌──────────────────▼──────────────────────────────────────┐
│                     DOMAIN LAYER                         │
│              (Business rules and entities)               │
│      Models, Use Cases, Repository Interfaces            │
└──────────────────┬──────────────────────────────────────┘
                   │
                   │ Defines contracts
                   │
┌──────────────────▼──────────────────────────────────────┐
│                      DATA LAYER                          │
│            (Where data comes from and goes to)           │
│   Repositories, Services, API Clients, Local Storage     │
└─────────────────────────────────────────────────────────┘
```

**Analogy**: Think of building a house:
- **Presentation Layer** = Interior design (what people see and interact with)
- **Domain Layer** = Blueprint (the rules and structure)
- **Data Layer** = Foundation and utilities (water, electricity - the infrastructure)

---

## High-Level System Design

### The Three-Layer Architecture

#### 1. Presentation Layer (UI)

**Purpose**: Display information to users and handle their interactions.

**Location**: `app/lib/ui/`

**Responsibilities**:
- Render screens and widgets
- Handle user input (taps, text entry, etc.)
- Display loading states and errors
- Navigate between screens
- Format data for display

**Key Components**:
- **Screens**: Full-page views (HomeScreen, BookingScreen, etc.)
- **ViewModels**: Business logic for each screen (MVVM pattern)
- **Widgets**: Reusable UI components (buttons, cards, etc.)

**Example**: The [HomeScreen](app/lib/ui/home/widgets/home_screen.dart) displays a list of user bookings. It doesn't know where the bookings come from - it just displays them.

---

#### 2. Domain Layer (Business Logic)

**Purpose**: Define the core business rules and entities of the application.

**Location**: `app/lib/domain/`

**Responsibilities**:
- Define what a "Booking" is, what a "Destination" is, etc.
- Contain business logic (e.g., "How to create a booking from user selections")
- Define interfaces (contracts) that other layers must follow
- Stay independent of frameworks and UI

**Key Components**:
- **Models**: Data structures (Booking, Destination, Activity, etc.)
- **Use Cases**: Complex business operations (BookingCreateUseCase)

**Example**: The [Booking](app/lib/domain/models/booking/booking.dart) model defines what information makes up a booking (dates, destination, activities). This definition is used everywhere in the app.

**Analogy**: The domain layer is like a recipe book. It defines what ingredients you need (models) and how to prepare a dish (use cases), but it doesn't care whether you're cooking in a gas or electric oven (data source).

---

#### 3. Data Layer (Data Access)

**Purpose**: Handle all data operations - fetching, storing, and managing data.

**Location**: `app/lib/data/`

**Responsibilities**:
- Fetch data from APIs or local storage
- Save data to databases or files
- Transform data between formats (JSON ↔ Dart objects)
- Implement the interfaces defined in the domain layer
- Handle network errors and retries

**Key Components**:
- **Repositories**: Implement data access contracts
- **Services**: Actual data sources (API client, local JSON files)
- **API Models**: Data transfer objects (DTOs)

**Example**: [BookingRepository](app/lib/data/repositories/booking/booking_repository.dart) has two implementations:
- **BookingRepositoryLocal**: Stores bookings in memory (for development)
- **BookingRepositoryRemote**: Stores bookings on a server via HTTP API (for production)

**Analogy**: The data layer is like a librarian. You ask for a book (data), and the librarian knows whether to get it from the local shelf (local storage) or request it from another library (API).

---

### Layer Communication Rules

**Important principle**: Layers can only communicate in one direction:

```
Presentation Layer
       ↓ (can use)
   Domain Layer
       ↓ (can use)
    Data Layer
```

**What this means**:
- ✅ UI can use domain models and call repositories
- ✅ Domain defines interfaces; data layer implements them
- ❌ Domain layer CANNOT import UI code
- ❌ Data layer CANNOT import UI code

**Why?** This keeps business logic independent of UI frameworks. You could replace Flutter with React Native, and the domain layer wouldn't change!

---

## Architectural Patterns

The Compass App implements several well-known design patterns:

### 1. Clean Architecture

**What it is**: A way of organizing code into layers with clear boundaries and dependencies.

**Benefits**:
- Business logic is testable without UI
- Easy to swap data sources (local vs. API)
- Code is organized and predictable

**Where to learn more**: See [docs/02-layer-breakdown.md](./02-layer-breakdown.md)

---

### 2. MVVM (Model-View-ViewModel)

**What it is**: A pattern for organizing UI code.

**Components**:
- **Model**: Data structures (Booking, Destination)
- **View**: Flutter widgets (HomeScreen)
- **ViewModel**: Business logic for the view (HomeViewModel)

**Benefits**:
- Separates UI from business logic
- Easy to test business logic
- Reactive UI updates automatically

**Example**:
```dart
// Model (domain/models/booking/booking.dart)
class Booking {
  final DateTime startDate;
  final DateTime endDate;
  final Destination destination;
  final List<Activity> activity;
}

// ViewModel (ui/home/view_models/home_viewmodel.dart)
class HomeViewModel extends ChangeNotifier {
  List<BookingSummary> _bookings = [];
  List<BookingSummary> get bookings => _bookings;

  Future<void> loadBookings() async {
    _bookings = await repository.getBookingsList();
    notifyListeners(); // Tells the View to update
  }
}

// View (ui/home/widgets/home_screen.dart)
class HomeScreen extends StatelessWidget {
  Widget build(BuildContext context) {
    return ListenableBuilder(
      listenable: viewModel,
      builder: (context, _) {
        return ListView(
          children: viewModel.bookings.map((booking) =>
            BookingCard(booking: booking)
          ).toList(),
        );
      },
    );
  }
}
```

**Analogy**: Think of a restaurant:
- **Model**: The actual food
- **ViewModel**: The waiter who takes orders and brings food
- **View**: The menu and table presentation

---

### 3. Repository Pattern

**What it is**: An abstraction that hides where data comes from.

**Benefits**:
- UI doesn't need to know if data is local or from an API
- Easy to switch data sources
- Consistent interface for data access

**Example**:
```dart
// Abstract interface (data/repositories/booking/booking_repository.dart)
abstract class BookingRepository {
  Future<Result<List<BookingSummary>>> getBookingsList();
}

// Local implementation (development)
class BookingRepositoryLocal implements BookingRepository {
  @override
  Future<Result<List<BookingSummary>>> getBookingsList() async {
    // Returns data from memory
  }
}

// Remote implementation (production)
class BookingRepositoryRemote implements BookingRepository {
  @override
  Future<Result<List<BookingSummary>>> getBookingsList() async {
    // Fetches data from API
  }
}

// Usage in ViewModel - doesn't know which implementation!
class HomeViewModel {
  final BookingRepository repository; // Could be either!

  Future<void> load() async {
    final result = await repository.getBookingsList();
    // Handle result
  }
}
```

---

### 4. Dependency Injection

**What it is**: A way to provide dependencies to components instead of having them create their own.

**Benefits**:
- Easy to test (inject mock objects)
- Flexible (swap implementations)
- Clear dependencies

**Implementation**: Uses the **Provider** package.

**Example**:
```dart
// Configuration (config/dependencies.dart)
List<SingleChildWidget> get providersLocal {
  return [
    Provider(create: (context) => LocalDataService()),
    Provider(
      create: (context) => BookingRepositoryLocal(
        localDataService: context.read(),
      ) as BookingRepository,
    ),
  ];
}

// Usage in widget
final repository = context.read<BookingRepository>();
```

**Analogy**: Instead of a chef going to the market to buy ingredients (creating dependencies), someone delivers the ingredients to the kitchen (injecting dependencies). This makes it easy to test recipes with different ingredient sources.

---

### 5. Command Pattern

**What it is**: Encapsulates UI actions as objects with state (running, completed, error).

**Benefits**:
- Track action progress (loading states)
- Prevent duplicate executions
- Consistent error handling

**Example**:
```dart
// In ViewModel
class HomeViewModel extends ChangeNotifier {
  HomeViewModel() {
    load = Command0(_loadBookings)..execute(); // Auto-execute
  }

  late Command0 load;

  Future<Result<void>> _loadBookings() async {
    final result = await repository.getBookingsList();
    // Handle result
  }
}

// In UI
ListenableBuilder(
  listenable: viewModel.load,
  builder: (context, _) {
    if (viewModel.load.running) {
      return CircularProgressIndicator();
    }
    if (viewModel.load.error) {
      return ErrorIndicator(viewModel.load.error);
    }
    // Show data
  },
)
```

---

### 6. Result Pattern

**What it is**: Type-safe error handling without exceptions.

**Benefits**:
- Forces error handling (compiler checks)
- No forgotten try-catch blocks
- Clear success/failure paths

**Example**:
```dart
// Result type (utils/result.dart)
sealed class Result<T> {
  const factory Result.ok(T value) = Ok._;
  const factory Result.error(Exception error) = Error._;
}

// Usage
final result = await repository.getBooking(id);
switch (result) {
  case Ok<Booking>():
    _booking = result.value; // Type-safe access
  case Error<Booking>():
    _error = result.error;
}
```

---

## Project Structure

### High-Level Folder Organization

```
compass_app/
├── app/                          # Flutter application
│   ├── lib/                     # Dart source code
│   ├── test/                    # Unit tests
│   ├── integration_test/        # Integration tests
│   ├── assets/                  # Images, JSON data files
│   └── pubspec.yaml            # Dependencies
│
├── server/                       # Backend server (optional)
│   ├── lib/                     # Server source code
│   └── pubspec.yaml            # Server dependencies
│
└── docs/                         # Documentation (you are here!)
```

---

### Detailed `app/lib/` Structure

```
app/lib/
├── main.dart                    # Main entry point (delegates to main_development.dart)
├── main_development.dart        # Development mode (local data)
├── main_staging.dart           # Staging mode (remote API)
│
├── config/                      # Application configuration
│   ├── dependencies.dart       # Dependency injection setup
│   └── assets.dart             # Asset path constants
│
├── domain/                      # Business logic layer
│   ├── models/                 # Domain entities
│   │   ├── activity/          # Activity model
│   │   ├── booking/           # Booking models
│   │   ├── continent/         # Continent model
│   │   ├── destination/       # Destination model
│   │   ├── itinerary_config/  # Temporary booking config
│   │   └── user/              # User model
│   │
│   └── use_cases/              # Business use cases
│       └── booking/           # Booking-related use cases
│           ├── booking_create_use_case.dart
│           └── booking_share_use_case.dart
│
├── data/                        # Data access layer
│   ├── repositories/           # Repository implementations
│   │   ├── activity/          # ActivityRepository
│   │   ├── auth/              # AuthRepository
│   │   ├── booking/           # BookingRepository
│   │   ├── continent/         # ContinentRepository
│   │   ├── destination/       # DestinationRepository
│   │   ├── itinerary_config/  # ItineraryConfigRepository
│   │   └── user/              # UserRepository
│   │
│   └── services/               # Data sources
│       ├── api/               # HTTP API client
│       ├── local/             # Local JSON data service
│       └── shared_preferences_service.dart
│
├── ui/                          # Presentation layer
│   ├── home/                   # Home feature
│   │   ├── view_models/       # HomeViewModel
│   │   └── widgets/           # HomeScreen + components
│   │
│   ├── search_form/            # Search feature
│   │   ├── view_models/       # SearchFormViewModel
│   │   └── widgets/           # SearchFormScreen + components
│   │
│   ├── results/                # Results feature
│   │   ├── view_models/       # ResultsViewModel
│   │   └── widgets/           # ResultsScreen + components
│   │
│   ├── activities/             # Activities feature
│   │   ├── view_models/       # ActivitiesViewModel
│   │   └── widgets/           # ActivitiesScreen + components
│   │
│   ├── booking/                # Booking feature
│   │   ├── view_models/       # BookingViewModel
│   │   └── widgets/           # BookingScreen + components
│   │
│   ├── auth/                   # Authentication feature
│   │   ├── view_models/       # LoginViewModel
│   │   └── widgets/           # LoginScreen, LogoutButton
│   │
│   └── core/                   # Shared UI components
│       ├── localization/      # Internationalization
│       ├── themes/            # Light/dark themes
│       └── ui/                # Reusable widgets
│
├── routing/                     # Navigation
│   ├── router.dart            # go_router configuration
│   └── routes.dart            # Route path constants
│
└── utils/                       # Utilities
    ├── command.dart           # Command pattern implementation
    └── result.dart            # Result pattern implementation
```

---

### Feature-Based Organization

Notice how the `ui/` folder is organized by **features** (home, search_form, booking, etc.) rather than by technical role (all screens in one folder, all view models in another).

**Benefits**:
- Related code is co-located
- Easy to find everything about a feature
- Features are more modular and independent

**Example**: Everything related to the booking feature is in [ui/booking/](app/lib/ui/booking/):
```
ui/booking/
├── view_models/
│   └── booking_viewmodel.dart     # Business logic
└── widgets/
    ├── booking_screen.dart         # Main screen
    ├── booking_header.dart         # Header component
    └── booking_body.dart           # Body component
```

If you want to modify the booking feature, you know exactly where to go!

---

## Environment Configurations

The Compass App supports **two environments** with different data sources:

### 1. Development Environment (Local Data)

**Entry Point**: [main_development.dart](app/lib/main_development.dart)

**Data Source**: Local JSON files in [assets/](app/assets/)

**Characteristics**:
- ✅ No server required
- ✅ Fast iteration
- ✅ No authentication needed
- ✅ Consistent test data
- ❌ Can't test real API integration

**Use Case**: Day-to-day development, UI work, offline testing

**How to Run**:
```bash
flutter run --target lib/main_development.dart
```

**Configuration**:
```dart
// main_development.dart
void main() {
  Logger.root.level = Level.WARNING;

  runApp(
    MultiProvider(
      providers: providersLocal,  // ← Uses local data
      child: const MainApp(),
    ),
  );
}
```

---

### 2. Staging Environment (Remote API)

**Entry Point**: [main_staging.dart](app/lib/main_staging.dart)

**Data Source**: HTTP API running on `localhost:8080`

**Characteristics**:
- ✅ Tests real API integration
- ✅ Full authentication flow
- ✅ Production-like behavior
- ❌ Requires backend server running
- ❌ Slower than local data

**Use Case**: Testing API integration, authentication flows, production scenarios

**How to Run**:
```bash
# Terminal 1: Start server
cd server
dart run

# Terminal 2: Run app
cd app
flutter run --target lib/main_staging.dart
```

**Configuration**:
```dart
// main_staging.dart
void main() {
  Logger.root.level = Level.ALL;

  runApp(
    MultiProvider(
      providers: providersRemote,  // ← Uses API
      child: const MainApp(),
    ),
  );
}
```

---

### Environment Switching Architecture

The magic happens in [config/dependencies.dart](app/lib/config/dependencies.dart):

```dart
// Local configuration
List<SingleChildWidget> get providersLocal {
  return [
    Provider(create: (context) => LocalDataService()),
    Provider(
      create: (context) => BookingRepositoryLocal(
        localDataService: context.read(),
      ) as BookingRepository,  // ← Returns as abstract type
    ),
    // ... other local providers
  ];
}

// Remote configuration
List<SingleChildWidget> get providersRemote {
  return [
    ChangeNotifierProvider(
      create: (context) => AuthRepositoryRemote(...),
    ),
    Provider(
      create: (context) => BookingRepositoryRemote(
        apiClient: context.read(),
      ) as BookingRepository,  // ← Returns as abstract type
    ),
    // ... other remote providers
  ];
}
```

**Key Insight**: Both configurations provide the same abstract types (BookingRepository, DestinationRepository, etc.), but with different implementations. The rest of the app doesn't know which implementation it's using!

**Analogy**: It's like having two power sources for your home - solar panels or the grid. Your appliances don't care which one provides electricity; they just plug into the same outlet.

---

## Technology Stack

### Core Technologies

| Technology | Version | Purpose |
|------------|---------|---------|
| **Flutter** | 3.5.0+ | UI framework |
| **Dart** | 3.5.0+ | Programming language |
| **go_router** | 14.6.2 | Declarative navigation |
| **Provider** | 6.1.2 | State management & DI |
| **Freezed** | 2.5.7 | Immutable models |
| **json_serializable** | 6.9.0 | JSON serialization |

---

### Key Packages

#### State Management & Architecture
- **provider**: Dependency injection and state management
- **freezed**: Code generation for immutable data classes
- **freezed_annotation**: Annotations for Freezed
- **json_annotation**: Annotations for JSON serialization

#### Navigation
- **go_router**: Modern declarative routing with deep linking

#### UI Components
- **cached_network_image**: Efficient image loading and caching
- **flutter_svg**: SVG image support
- **google_fonts**: Custom typography

#### Utilities
- **share_plus**: Native share dialog integration
- **shared_preferences**: Persistent key-value storage
- **intl**: Internationalization and formatting
- **logging**: Structured logging
- **collection**: Advanced collection utilities

#### Testing
- **flutter_test**: Unit testing framework
- **integration_test**: E2E testing
- **mocktail**: Mocking library for tests
- **mocktail_image_network**: Mock network images in tests

---

### Backend Stack

The optional backend server uses:

| Technology | Purpose |
|------------|---------|
| **Dart Shelf** | HTTP server framework |
| **shelf_router** | Request routing |
| **Freezed** | Shared models with app |

**Why Dart for backend?**
Using Dart for both frontend and backend allows sharing models and serialization logic. The [Booking model](app/lib/domain/models/booking/booking.dart) is used identically in both!

---

## Key Architectural Decisions

### 1. Why Clean Architecture?

**Decision**: Organize code into three layers (Presentation, Domain, Data)

**Reasoning**:
- **Testability**: Business logic can be tested without UI
- **Flexibility**: Easy to swap data sources (local ↔ API)
- **Maintainability**: Clear organization makes code easier to understand
- **Scalability**: New features follow established patterns

**Trade-offs**:
- ⚠️ More files and folders (higher initial complexity)
- ⚠️ More abstraction (learning curve for beginners)
- ✅ Long-term maintainability outweighs initial complexity

---

### 2. Why MVVM Instead of BLoC?

**Decision**: Use MVVM (ViewModel + ChangeNotifier) instead of BLoC pattern

**Reasoning**:
- **Simplicity**: Easier for beginners to understand
- **Less Boilerplate**: No need for events and states
- **Direct**: ViewModels directly expose properties
- **Sufficient**: Meets all state management needs

**Comparison**:

| Aspect | MVVM | BLoC |
|--------|------|------|
| Learning Curve | Lower | Higher |
| Boilerplate | Less | More |
| Testability | Excellent | Excellent |
| Ecosystem | Provider | flutter_bloc |
| Use Case | Small-medium apps | Large apps, complex state |

**When to use BLoC instead**: Very large apps with complex state transitions, teams already familiar with BLoC, apps requiring time-travel debugging.

---

### 3. Why go_router Instead of Navigator 1.0?

**Decision**: Use go_router for navigation

**Reasoning**:
- **Declarative**: Define routes upfront, not imperatively
- **Deep Linking**: Built-in URL support
- **Type Safety**: Compile-time route validation
- **Redirect Logic**: Auth redirects built-in
- **Modern**: Recommended by Flutter team

**Example Benefit**: Authentication redirects are handled declaratively:
```dart
GoRouter(
  redirect: (context, state) async {
    final loggedIn = await authRepository.isAuthenticated;
    if (!loggedIn) return Routes.login;  // Auto-redirect!
    return null;
  },
)
```

---

### 4. Why Freezed for Models?

**Decision**: Use Freezed code generation for domain models

**Reasoning**:
- **Immutability**: Prevents accidental mutations
- **copyWith**: Easy to create modified copies
- **Equality**: Automatic == and hashCode
- **JSON Serialization**: Integrates with json_serializable
- **Union Types**: Support for sealed classes

**Example**:
```dart
@freezed
class Booking with _$Booking {
  const factory Booking({
    required int id,
    required DateTime startDate,
    required DateTime endDate,
    required Destination destination,
    required List<Activity> activity,
  }) = _Booking;

  factory Booking.fromJson(Map<String, dynamic> json) =>
      _$BookingFromJson(json);
}

// Generated code provides:
// - Immutable properties
// - booking.copyWith(endDate: newDate)
// - booking1 == booking2
// - Booking.fromJson(json) and booking.toJson()
```

---

### 5. Why Repository Pattern?

**Decision**: Abstract all data access behind repository interfaces

**Reasoning**:
- **Testability**: Easy to inject mock repositories
- **Flexibility**: Swap local ↔ remote without changing business logic
- **Consistency**: Uniform API for all data access
- **Separation**: Data layer isolated from domain layer

**Real-world example**: During development, use local data. In production, use API. Same code in ViewModels!

---

### 6. Why Command Pattern?

**Decision**: Encapsulate async actions in Command objects

**Reasoning**:
- **Loading States**: Automatic `running`, `error`, `completed` tracking
- **Prevent Duplicates**: Can't execute while already running
- **Consistent UI**: Same pattern for all async actions
- **Error Handling**: Centralized error management

**Before (without Command)**:
```dart
class HomeViewModel extends ChangeNotifier {
  bool _loading = false;
  String? _error;

  bool get loading => _loading;
  String? get error => _error;

  Future<void> loadBookings() async {
    if (_loading) return;  // Manual duplicate prevention
    _loading = true;
    _error = null;
    notifyListeners();

    try {
      final result = await repository.getBookingsList();
      switch (result) {
        case Ok(): _bookings = result.value;
        case Error(): _error = result.error.toString();
      }
    } finally {
      _loading = false;
      notifyListeners();
    }
  }
}
```

**After (with Command)**:
```dart
class HomeViewModel extends ChangeNotifier {
  late Command0 load;

  HomeViewModel() {
    load = Command0(_loadBookings)..execute();
  }

  Future<Result<void>> _loadBookings() async {
    final result = await repository.getBookingsList();
    // Command handles running/error/completed automatically!
    return result;
  }
}
```

Much cleaner! 🎉

---

### 7. Why Result Pattern Instead of Exceptions?

**Decision**: Use `Result<T>` type for error handling instead of throwing exceptions

**Reasoning**:
- **Compile-Time Safety**: Compiler forces you to handle errors
- **Explicit**: Clear which methods can fail
- **No Surprises**: No forgotten try-catch blocks
- **Pattern Matching**: Elegant error handling with switch

**Before (with exceptions)**:
```dart
Future<Booking> getBooking(int id) async {
  // Might throw! Caller may forget try-catch
  final response = await http.get('/booking/$id');
  if (response.statusCode != 200) {
    throw Exception('Failed to load booking');
  }
  return Booking.fromJson(response.data);
}

// Caller - easy to forget error handling!
final booking = await repository.getBooking(id);
```

**After (with Result)**:
```dart
Future<Result<Booking>> getBooking(int id) async {
  final response = await http.get('/booking/$id');
  if (response.statusCode != 200) {
    return Result.error(Exception('Failed to load booking'));
  }
  return Result.ok(Booking.fromJson(response.data));
}

// Caller - MUST handle both cases!
final result = await repository.getBooking(id);
switch (result) {
  case Ok<Booking>():
    _booking = result.value;
  case Error<Booking>():
    _error = result.error;  // Compiler forces error handling!
}
```

---

## Architecture Benefits

### 1. Testability

**What**: Easy to write unit tests for business logic

**How**:
- ViewModels don't depend on UI framework
- Repositories can be mocked
- Use cases are pure functions

**Example Test**:
```dart
test('HomeViewModel loads bookings successfully', () async {
  // Arrange
  final mockRepo = MockBookingRepository();
  when(() => mockRepo.getBookingsList()).thenAnswer(
    (_) async => Result.ok([booking1, booking2]),
  );

  final viewModel = HomeViewModel(bookingRepository: mockRepo);

  // Act
  await viewModel.load.execute();

  // Assert
  expect(viewModel.bookings, [booking1, booking2]);
  expect(viewModel.load.completed, isTrue);
});
```

See [test/ui/home/](app/test/ui/home/) for real examples.

---

### 2. Maintainability

**What**: Easy to modify and extend code

**How**:
- Clear layer boundaries
- Feature-based organization
- Consistent patterns throughout
- Self-documenting code structure

**Example**: To add a new booking feature:
1. Add method to BookingRepository interface
2. Implement in BookingRepositoryLocal and BookingRepositoryRemote
3. Add command to BookingViewModel
4. Update BookingScreen UI

Each step is clear and isolated!

---

### 3. Scalability

**What**: Easy to add new features without breaking existing ones

**How**:
- New features follow existing patterns
- Dependencies are injected, not hardcoded
- Layers are independent

**Example**: Adding a "Favorites" feature:
```
1. Create domain/models/favorite/favorite.dart
2. Create data/repositories/favorite/favorite_repository.dart
3. Implement local and remote versions
4. Create ui/favorites/ with ViewModel and Screen
5. Add route in routing/router.dart
6. Register in config/dependencies.dart
```

Existing booking, search, and activities features are unaffected!

---

### 4. Flexibility

**What**: Easy to swap implementations

**How**:
- Repository pattern abstracts data sources
- Dependency injection makes swapping easy
- Environment configurations pre-configured

**Examples**:
- Swap local data → API by changing one line (`providersLocal` → `providersRemote`)
- Add Firebase backend by creating new repository implementations
- Add GraphQL instead of REST without changing ViewModels

---

### 5. Developer Experience

**What**: Pleasant development workflow

**How**:
- Hot reload works perfectly (stateless architecture)
- Clear error messages (Result pattern)
- Consistent patterns reduce decision fatigue
- Code generation reduces boilerplate

**Workflow Example**:
1. Modify UI in `widgets/` → Hot reload → See changes
2. Modify ViewModel logic → Hot reload → Test immediately
3. Modify model → Run code generation → Continue

No full rebuild needed! ⚡

---

## Architecture Visualization

### Complete System Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│                            COMPASS APP                                  │
│                     Travel Itinerary Platform                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │
        ┌───────────────────────────┴───────────────────────────┐
        │                                                       │
        ▼                                                       ▼
┌───────────────────┐                                 ┌───────────────────┐
│   DEVELOPMENT     │                                 │     STAGING       │
│   ENVIRONMENT     │                                 │   ENVIRONMENT     │
│                   │                                 │                   │
│ • Local JSON data │                                 │ • Remote API      │
│ • No auth         │                                 │ • Full auth       │
│ • Fast iteration  │                                 │ • Production-like │
└─────────┬─────────┘                                 └─────────┬─────────┘
          │                                                     │
          │                                                     │
          └──────────────────┬──────────────────────────────────┘
                             │
                             │
┌────────────────────────────▼────────────────────────────────────────────┐
│                      PRESENTATION LAYER                                 │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Feature Modules                              │   │
│  │                                                                 │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐           │   │
│  │  │  Home   │  │ Search  │  │ Results │  │Activity │           │   │
│  │  │ Screen  │→ │  Form   │→ │ Screen  │→ │ Screen  │           │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘           │   │
│  │       ↓            ↓             ↓            ↓                 │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐           │   │
│  │  │  Home   │  │ Search  │  │ Results │  │Activity │           │   │
│  │  │VwModel  │  │VwModel  │  │VwModel  │  │VwModel  │           │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘           │   │
│  │                                                                 │   │
│  │                          ↓                                      │   │
│  │                  ┌─────────────┐                                │   │
│  │                  │   Booking   │                                │   │
│  │                  │   Screen    │                                │   │
│  │                  └─────────────┘                                │   │
│  │                        ↓                                        │   │
│  │                  ┌─────────────┐                                │   │
│  │                  │   Booking   │                                │   │
│  │                  │  ViewModel  │                                │   │
│  │                  └─────────────┘                                │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                  Authentication                                 │   │
│  │  ┌─────────┐              ┌─────────┐                           │   │
│  │  │  Login  │              │ Logout  │                           │   │
│  │  │ Screen  │              │ Button  │                           │   │
│  │  └─────────┘              └─────────┘                           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Core UI Components                           │   │
│  │  • Themes (Light/Dark)  • Widgets  • Localization              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                │ Uses
                                │
┌───────────────────────────────▼─────────────────────────────────────────┐
│                         DOMAIN LAYER                                    │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      Domain Models                              │   │
│  │                                                                 │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │   │
│  │  │   Booking   │  │ Destination │  │  Activity   │             │   │
│  │  │             │  │             │  │             │             │   │
│  │  │ • id        │  │ • ref       │  │ • name      │             │   │
│  │  │ • dates     │  │ • name      │  │ • duration  │             │   │
│  │  │ • dest      │  │ • country   │  │ • price     │             │   │
│  │  │ • activity  │  │ • imageUrl  │  │ • timeOfDay │             │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘             │   │
│  │                                                                 │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │   │
│  │  │  Continent  │  │    User     │  │  Itinerary  │             │   │
│  │  │             │  │             │  │   Config    │             │   │
│  │  │ • name      │  │ • name      │  │             │             │   │
│  │  │ • imageUrl  │  │ • picture   │  │ (temp state)│             │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘             │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                       Use Cases                                 │   │
│  │                                                                 │   │
│  │  ┌────────────────────────────────────────────────────────┐     │   │
│  │  │  BookingCreateUseCase                                  │     │   │
│  │  │  • Orchestrates booking creation                       │     │   │
│  │  │  • Validates required data                             │     │   │
│  │  │  • Coordinates multiple repositories                   │     │   │
│  │  └────────────────────────────────────────────────────────┘     │   │
│  │                                                                 │   │
│  │  ┌────────────────────────────────────────────────────────┐     │   │
│  │  │  BookingShareUseCase                                   │     │   │
│  │  │  • Formats booking for sharing                         │     │   │
│  │  │  • Integrates with OS share dialog                     │     │   │
│  │  └────────────────────────────────────────────────────────┘     │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                │ Implements
                                │
┌───────────────────────────────▼─────────────────────────────────────────┐
│                          DATA LAYER                                     │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Repository Implementations                   │   │
│  │                                                                 │   │
│  │  ┌──────────────────────┐         ┌──────────────────────┐     │   │
│  │  │ BookingRepository    │         │ DestinationRepository│     │   │
│  │  │                      │         │                      │     │   │
│  │  │ • Local: in-memory   │         │ • Local: JSON        │     │   │
│  │  │ • Remote: HTTP API   │         │ • Remote: HTTP API   │     │   │
│  │  └──────────────────────┘         └──────────────────────┘     │   │
│  │                                                                 │   │
│  │  ┌──────────────────────┐         ┌──────────────────────┐     │   │
│  │  │ ActivityRepository   │         │ AuthRepository       │     │   │
│  │  │                      │         │                      │     │   │
│  │  │ • Local: JSON        │         │ • Dev: always auth   │     │   │
│  │  │ • Remote: HTTP API   │         │ • Remote: JWT        │     │   │
│  │  └──────────────────────┘         └──────────────────────┘     │   │
│  │                                                                 │   │
│  │  ┌──────────────────────┐         ┌──────────────────────┐     │   │
│  │  │ ContinentRepository  │         │ UserRepository       │     │   │
│  │  │                      │         │                      │     │   │
│  │  │ • Local: hardcoded   │         │ • Local: mock        │     │   │
│  │  │ • Remote: HTTP API   │         │ • Remote: HTTP API   │     │   │
│  │  └──────────────────────┘         └──────────────────────┘     │   │
│  │                                                                 │   │
│  │  ┌──────────────────────────────────────────────────────┐      │   │
│  │  │ ItineraryConfigRepository (in-memory only)           │      │   │
│  │  │ • Stores temporary booking selections                │      │   │
│  │  └──────────────────────────────────────────────────────┘      │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      Data Services                              │   │
│  │                                                                 │   │
│  │  ┌─────────────────────┐        ┌─────────────────────┐        │   │
│  │  │ LocalDataService    │        │    ApiClient        │        │   │
│  │  │                     │        │                     │        │   │
│  │  │ • Loads JSON assets │        │ • HTTP client       │        │   │
│  │  │ • Parses data       │        │ • Auth headers      │        │   │
│  │  │ • Mock user         │        │ • Error handling    │        │   │
│  │  └─────────────────────┘        └─────────────────────┘        │   │
│  │                                                                 │   │
│  │  ┌─────────────────────┐        ┌─────────────────────┐        │   │
│  │  │  AuthApiClient      │        │ SharedPreferences   │        │   │
│  │  │                     │        │                     │        │   │
│  │  │ • Login endpoint    │        │ • Token storage     │        │   │
│  │  │ • JWT tokens        │        │ • Persistence       │        │   │
│  │  └─────────────────────┘        └─────────────────────┘        │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                       CROSS-CUTTING CONCERNS                            │
│                                                                         │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐      │
│  │    Routing       │  │  Dependency      │  │     Utils        │      │
│  │   (go_router)    │  │   Injection      │  │                  │      │
│  │                  │  │   (Provider)     │  │ • Result<T>      │      │
│  │ • Auth redirects │  │                  │  │ • Command        │      │
│  │ • Deep linking   │  │ • providersLocal │  │ • Logging        │      │
│  │ • Type-safe      │  │ • providersRemote│  │                  │      │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

The Compass App demonstrates a **well-architected Flutter application** with:

✅ **Clean Architecture**: Three layers with clear boundaries
✅ **MVVM Pattern**: Separation of UI and business logic
✅ **Repository Pattern**: Flexible data access
✅ **Dependency Injection**: Provider-based IoC
✅ **Command Pattern**: Consistent async action handling
✅ **Result Pattern**: Type-safe error handling
✅ **Environment Support**: Easy switching between local and remote data
✅ **Feature-Based Structure**: Related code co-located
✅ **Comprehensive Testing**: Unit and integration tests
✅ **Modern Navigation**: go_router with deep linking
✅ **Immutable Models**: Freezed code generation

---

## Next Steps

Now that you understand the overall architecture, explore the detailed documentation:

1. **[Layer Breakdown](./02-layer-breakdown.md)** - Deep dive into each layer
2. **[Design Patterns](./03-design-patterns.md)** - Pattern implementations with examples
3. **[Component Deep-Dive](./04-component-deep-dive.md)** - Key components and data flows
4. **[Getting Started](./05-getting-started.md)** - Practical guide for developers
5. **[Reference Guide](./06-reference-guide.md)** - Complete API and component reference

---

## Questions?

If you're new to the project:
- Start with the [Getting Started Guide](./05-getting-started.md)
- Review code examples in this document
- Look at real implementations in the [app/lib/](../app/lib/) folder

If you're modifying the architecture:
- Understand the [design patterns](./03-design-patterns.md) first
- Follow established conventions
- Update this documentation!

---

**Happy coding! 🚀**
