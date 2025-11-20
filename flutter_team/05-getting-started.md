# 05 - Getting Started Guide

## 📋 Table of Contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Initial Setup](#initial-setup)
- [Running the Application](#running-the-application)
- [Project Navigation](#project-navigation)
- [Common Development Tasks](#common-development-tasks)
- [Adding New Features](#adding-new-features)
- [Testing Guide](#testing-guide)
- [Debugging Tips](#debugging-tips)
- [Code Style Guidelines](#code-style-guidelines)
- [Troubleshooting](#troubleshooting)

---

## Introduction

Welcome to the Compass App development guide! This document will help you **get up and running quickly** and become productive with the codebase.

**Who is this for?**
- New developers joining the project
- Flutter developers learning Clean Architecture
- Contributors wanting to add features
- Anyone exploring the codebase

**What you'll learn**:
- How to set up your development environment
- How to run the app in different modes
- How to navigate the codebase effectively
- How to add new features following established patterns
- How to test your changes
- How to debug common issues

---

## Prerequisites

### Required Software

| Software | Minimum Version | Installation |
|----------|----------------|--------------|
| **Flutter SDK** | 3.5.0 | [flutter.dev/docs/get-started/install](https://flutter.dev/docs/get-started/install) |
| **Dart SDK** | 3.5.0 | Included with Flutter |
| **IDE** | Any | VS Code (recommended) or Android Studio |
| **Git** | 2.0+ | [git-scm.com](https://git-scm.com/) |

### Optional (for staging mode)

| Software | Version | Purpose |
|----------|---------|---------|
| **Dart SDK** (standalone) | 3.5.0+ | Run backend server |

### IDE Plugins (Recommended)

**For VS Code**:
- Flutter (by Dart Code)
- Dart (by Dart Code)
- Error Lens (helpful error display)
- GitLens (Git integration)

**For Android Studio**:
- Flutter plugin
- Dart plugin

---

### Verify Installation

Run these commands to verify your setup:

```bash
# Check Flutter version
flutter --version

# Check that Flutter is properly configured
flutter doctor

# Check Dart version
dart --version
```

**Expected output** (Flutter):
```
Flutter 3.5.0 • channel stable • https://github.com/flutter/flutter.git
Framework • revision xxxxx
Engine • revision xxxxx
Tools • Dart 3.5.0 • DevTools 2.x.x
```

**Expected output** (flutter doctor):
```
[✓] Flutter (Channel stable, 3.5.0, on macOS/Linux/Windows)
[✓] Android toolchain - develop for Android devices
[✓] Xcode - develop for iOS and macOS (macOS only)
[✓] Chrome - develop for the web
[✓] Android Studio / VS Code
[✓] Connected device
```

If you see any ❌ or ⚠️, follow the suggested fixes.

---

## Initial Setup

### Step 1: Clone the Repository

```bash
# Clone the repository
git clone <repository-url>
cd compass_app

# Check repository structure
ls -la
```

**Expected structure**:
```
compass_app/
├── app/          # Flutter application
├── server/       # Backend server (optional)
├── docs/         # Documentation
└── README.md
```

---

### Step 2: Install Dependencies

```bash
# Navigate to app directory
cd app

# Install Flutter packages
flutter pub get
```

**What this does**:
- Downloads all packages listed in `pubspec.yaml`
- Creates `.dart_tool/` directory
- Updates `pubspec.lock` with resolved versions

**Expected output**:
```
Running "flutter pub get" in app...
Resolving dependencies...
Got dependencies!
```

---

### Step 3: Run Code Generation

The app uses **Freezed** to generate code for immutable models. You need to generate this code before running the app.

```bash
# Generate code (one-time)
flutter pub run build_runner build --delete-conflicting-outputs
```

**What this does**:
- Generates `*.freezed.dart` files for models
- Generates `*.g.dart` files for JSON serialization
- Deletes conflicting outputs if any

**Expected output**:
```
[INFO] Generating build script...
[INFO] Generating build script completed, took 2.1s

[INFO] Creating build script snapshot...
[INFO] Creating build script snapshot completed, took 3.4s

[INFO] Initializing inputs
[INFO] Building new asset graph...
[INFO] Building new asset graph completed, took 1.2s

[INFO] Checking for unexpected pre-existing outputs...
[INFO] Succeeded after 5.7s with 220 outputs (440 actions)
```

**Generated files**:
- `lib/domain/models/booking/booking.freezed.dart`
- `lib/domain/models/booking/booking.g.dart`
- And many more...

---

### Step 4: Verify Setup

Run a quick check to ensure everything is set up correctly:

```bash
# Run Flutter analyze (checks for code issues)
flutter analyze

# Run tests
flutter test
```

**Expected output** (analyze):
```
Analyzing app...
No issues found!
```

**Expected output** (test):
```
00:02 +50: All tests passed!
```

If you see errors, check the [Troubleshooting](#troubleshooting) section.

---

## Running the Application

### Development Mode (Recommended for Daily Work)

**Characteristics**:
- ✅ Local data (no server needed)
- ✅ Fast iteration
- ✅ No authentication required
- ✅ Sample data pre-loaded

**Run Command**:
```bash
flutter run --target lib/main_development.dart
```

**Or in VS Code**:
1. Open `lib/main_development.dart`
2. Press `F5` or click "Run" → "Start Debugging"

**Expected behavior**:
- App launches
- Shows home screen with a sample booking
- No login screen

---

### Staging Mode (For Testing API Integration)

**Characteristics**:
- ✅ Remote API data
- ✅ Full authentication flow
- ✅ Production-like behavior
- ❌ Requires backend server

**Run Commands**:

**Terminal 1** (Start backend server):
```bash
cd server
dart run
```

**Expected output**:
```
Server listening on http://localhost:8080
```

**Terminal 2** (Run app):
```bash
cd app
flutter run --target lib/main_staging.dart
```

**Expected behavior**:
- App launches
- Shows login screen
- Can log in with test credentials
- Data fetched from server

**Test Credentials** (check server code for actual credentials):
```
Email: user@example.com
Password: password
```

---

### Hot Reload & Hot Restart

**Hot Reload** (recommended for UI changes):
- Press `r` in terminal
- Or save file in VS Code (auto hot-reload)
- Fast (~1 second)
- Preserves app state

**Hot Restart** (for logic changes):
- Press `R` in terminal
- Or click restart icon in IDE
- Slower (~3 seconds)
- Resets app state

**When to use each**:
- **Hot Reload**: Changing UI, colors, layouts
- **Hot Restart**: Changing models, business logic, initialization code

---

### Platform Selection

**Run on specific platform**:

```bash
# iOS simulator (macOS only)
flutter run -d ios

# Android emulator
flutter run -d android

# Chrome web browser
flutter run -d chrome

# macOS desktop (macOS only)
flutter run -d macos

# Windows desktop (Windows only)
flutter run -d windows

# Linux desktop (Linux only)
flutter run -d linux
```

**List available devices**:
```bash
flutter devices
```

---

## Project Navigation

### "I want to understand the overall architecture"

**Read**:
1. [docs/01-architecture-overview.md](./01-architecture-overview.md) - Big picture
2. [docs/02-layer-breakdown.md](./02-layer-breakdown.md) - Layer details

**Explore**:
- [app/lib/domain/](../app/lib/domain/) - Business logic
- [app/lib/data/](../app/lib/data/) - Data access
- [app/lib/ui/](../app/lib/ui/) - User interface

---

### "I want to understand a specific feature"

**Example: Booking feature**

**Steps**:
1. Start with the **Screen**: [app/lib/ui/booking/widgets/booking_screen.dart](../app/lib/ui/booking/widgets/booking_screen.dart)
   - See what's displayed to the user

2. Check the **ViewModel**: [app/lib/ui/booking/view_models/booking_viewmodel.dart](../app/lib/ui/booking/view_models/booking_viewmodel.dart)
   - See business logic and state management

3. Find the **Repository**: [app/lib/data/repositories/booking/booking_repository.dart](../app/lib/data/repositories/booking/booking_repository.dart)
   - See data access interface

4. Check **Implementations**:
   - Local: [app/lib/data/repositories/booking/booking_repository_local.dart](../app/lib/data/repositories/booking/booking_repository_local.dart)
   - Remote: [app/lib/data/repositories/booking/booking_repository_remote.dart](../app/lib/data/repositories/booking/booking_repository_remote.dart)

5. Look at **Models**: [app/lib/domain/models/booking/booking.dart](../app/lib/domain/models/booking/booking.dart)
   - See data structure

---

### "I want to find where X is implemented"

**Use Case: Find where bookings are deleted**

**Method 1: IDE Search**

In VS Code:
1. Press `Cmd+Shift+F` (macOS) or `Ctrl+Shift+F` (Windows/Linux)
2. Search for `delete` in `booking`
3. Find `deleteBooking` in `HomeViewModel`

In Android Studio:
1. Press `Cmd+Shift+F` (macOS) or `Ctrl+Shift+F` (Windows/Linux)
2. Enter search term
3. Browse results

**Method 2: Follow the Flow**

1. Find the UI button: Search for "Delete" text
2. Find the handler: Look for `onDelete` or similar
3. Find the ViewModel method: Follow the method call
4. Find the repository: Check what the ViewModel calls

---

### "I want to understand data flow"

**Example: How bookings are loaded**

**Trace the flow**:

```
1. HomeScreen displays bookings
   File: app/lib/ui/home/widgets/home_screen.dart
   ↓

2. Data comes from HomeViewModel
   File: app/lib/ui/home/view_models/home_viewmodel.dart
   Method: _load()
   ↓

3. ViewModel calls BookingRepository
   File: app/lib/data/repositories/booking/booking_repository.dart
   Method: getBookingsList()
   ↓

4. Repository implementation (Local or Remote)
   Local: app/lib/data/repositories/booking/booking_repository_local.dart
   Remote: app/lib/data/repositories/booking/booking_repository_remote.dart
   ↓

5. Data source
   Local: In-memory list
   Remote: ApiClient → HTTP GET /booking
```

**Tip**: Use VS Code's "Go to Definition" (`F12`) to jump between files.

---

### "I want to see all routes"

**File**: [app/lib/routing/router.dart](../app/lib/routing/router.dart)

**Visual map**:
```
/login          → LoginScreen
/               → HomeScreen
  /search       → SearchFormScreen
  /results      → ResultsScreen
  /activities   → ActivitiesScreen
  /booking      → BookingScreen (new)
  /booking?id=X → BookingScreen (existing)
```

**Navigate in code**:
- Route definitions: [router.dart](../app/lib/routing/router.dart)
- Route paths: [routes.dart](../app/lib/routing/routes.dart)

---

## Common Development Tasks

### Task 1: Add a New Field to a Model

**Example**: Add `notes` field to Booking model

**Step 1**: Update the model

**File**: [app/lib/domain/models/booking/booking.dart](../app/lib/domain/models/booking/booking.dart)

```dart
@freezed
class Booking with _$Booking {
  const factory Booking({
    int? id,
    required DateTime startDate,
    required DateTime endDate,
    required Destination destination,
    required List<Activity> activity,
    String? notes, // ← Add this
  }) = _Booking;

  factory Booking.fromJson(Map<String, dynamic> json) =>
      _$BookingFromJson(json);
}
```

**Step 2**: Regenerate code

```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

**Step 3**: Update UI to display/edit the field

**File**: [app/lib/ui/booking/widgets/booking_screen.dart](../app/lib/ui/booking/widgets/booking_screen.dart)

```dart
// Display notes
if (booking.notes != null)
  Text('Notes: ${booking.notes}'),
```

**Step 4**: Test

```bash
flutter test
flutter run
```

---

### Task 2: Add a New Repository Method

**Example**: Add `search(query)` method to DestinationRepository

**Step 1**: Add method to interface

**File**: [app/lib/data/repositories/destination/destination_repository.dart](../app/lib/data/repositories/destination/destination_repository.dart)

```dart
abstract class DestinationRepository {
  Future<Result<List<Destination>>> getDestinations();

  // Add this
  Future<Result<List<Destination>>> search(String query);
}
```

**Step 2**: Implement in Local

**File**: [app/lib/data/repositories/destination/destination_repository_local.dart](../app/lib/data/repositories/destination/destination_repository_local.dart)

```dart
class DestinationRepositoryLocal implements DestinationRepository {
  // Existing methods...

  @override
  Future<Result<List<Destination>>> search(String query) async {
    final result = await _localDataService.getDestinations();

    return result.when(
      ok: (destinations) {
        final filtered = destinations.where((d) {
          return d.name.toLowerCase().contains(query.toLowerCase()) ||
              d.country.toLowerCase().contains(query.toLowerCase());
        }).toList();
        return Result.ok(filtered);
      },
      error: (error) => Result.error(error),
    );
  }
}
```

**Step 3**: Implement in Remote

**File**: [app/lib/data/repositories/destination/destination_repository_remote.dart](../app/lib/data/repositories/destination/destination_repository_remote.dart)

```dart
class DestinationRepositoryRemote implements DestinationRepository {
  // Existing methods...

  @override
  Future<Result<List<Destination>>> search(String query) async {
    return await _apiClient.searchDestinations(query);
  }
}
```

**Step 4**: Update ApiClient (for remote)

**File**: [app/lib/data/services/api/api_client.dart](../app/lib/data/services/api/api_client.dart)

```dart
Future<Result<List<Destination>>> searchDestinations(String query) async {
  try {
    final uri = Uri.parse('$baseUrl/destination?q=$query');
    final headers = await _authHeaderProvider.getAuthHeader();
    final response = await _client.get(uri, headers: headers);

    if (response.statusCode != 200) {
      return Result.error(Exception('Search failed'));
    }

    final List<dynamic> json = jsonDecode(response.body);
    final destinations = json.map((item) =>
        Destination.fromJson(item as Map<String, dynamic>)
    ).toList();

    return Result.ok(destinations);
  } on Exception catch (e) {
    return Result.error(e);
  }
}
```

**Step 5**: Use in ViewModel

```dart
Future<Result<void>> _search(String query) async {
  final result = await _destinationRepository.search(query);

  switch (result) {
    case Ok<List<Destination>>():
      _destinations = result.value;
      notifyListeners();
    case Error<List<Destination>>():
      // Handle error
  }
}
```

---

### Task 3: Add a New Command to ViewModel

**Example**: Add `refresh` command to HomeViewModel

**File**: [app/lib/ui/home/view_models/home_viewmodel.dart](../app/lib/ui/home/view_models/home_viewmodel.dart)

```dart
class HomeViewModel extends ChangeNotifier {
  HomeViewModel({...}) {
    load = Command0(_load)..execute();
    deleteBooking = Command1(_deleteBooking);
    refresh = Command0(_refresh); // ← Add this
  }

  late Command0 load;
  late Command1<void, int> deleteBooking;
  late Command0 refresh; // ← Add this

  Future<Result<void>> _refresh() async {
    // Clear cache if any
    _bookings.clear();
    notifyListeners();

    // Reload data
    return await _load();
  }
}
```

**Use in View**:

```dart
RefreshIndicator(
  onRefresh: () async {
    await viewModel.refresh.execute();
  },
  child: ListView(...),
)
```

---

### Task 4: Change Theme Colors

**File**: [app/lib/ui/core/themes/app_theme.dart](../app/lib/ui/core/themes/app_theme.dart)

```dart
class AppTheme {
  static ThemeData get lightTheme {
    return ThemeData(
      useMaterial3: true,
      colorScheme: ColorScheme.fromSeed(
        seedColor: Colors.green, // ← Change from blue to green
        brightness: Brightness.light,
      ),
      textTheme: GoogleFonts.montserratTextTheme(),
    );
  }
}
```

**Hot reload** to see changes immediately!

---

### Task 5: Add Logging

**Import logger**:
```dart
import 'package:logging/logging.dart';
```

**Create logger**:
```dart
class MyViewModel extends ChangeNotifier {
  final Logger _log = Logger('MyViewModel');

  Future<void> _load() async {
    _log.fine('Loading data');

    final result = await repository.getData();

    switch (result) {
      case Ok():
        _log.info('Loaded ${result.value.length} items');
      case Error():
        _log.warning('Failed to load data', result.error);
    }
  }
}
```

**Log Levels**:
- `FINE`: Detailed debug info
- `INFO`: General information
- `WARNING`: Potential issues
- `SEVERE`: Serious errors

**View logs**: Check terminal or IDE debug console.

---

## Adding New Features

### Feature: Add a "Favorites" System

Let's walk through adding a complete new feature step-by-step.

---

#### Step 1: Create Domain Model

**File**: `app/lib/domain/models/favorite/favorite.dart`

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'favorite.freezed.dart';
part 'favorite.g.dart';

@freezed
class Favorite with _$Favorite {
  const factory Favorite({
    required int id,
    required String destinationRef,
    required DateTime createdAt,
  }) = _Favorite;

  factory Favorite.fromJson(Map<String, dynamic> json) =>
      _$FavoriteFromJson(json);
}
```

**Generate code**:
```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

---

#### Step 2: Create Repository Interface

**File**: `app/lib/data/repositories/favorite/favorite_repository.dart`

```dart
import 'package:flutter/foundation.dart';
import '../../../domain/models/favorite/favorite.dart';
import '../../../utils/result.dart';

abstract class FavoriteRepository {
  Future<Result<List<Favorite>>> getFavorites();
  Future<Result<void>> addFavorite(String destinationRef);
  Future<Result<void>> removeFavorite(int id);
  Future<Result<bool>> isFavorite(String destinationRef);
}
```

---

#### Step 3: Implement Local Repository

**File**: `app/lib/data/repositories/favorite/favorite_repository_local.dart`

```dart
import 'package:logging/logging.dart';
import '../../../domain/models/favorite/favorite.dart';
import '../../../utils/result.dart';
import 'favorite_repository.dart';

class FavoriteRepositoryLocal implements FavoriteRepository {
  final Logger _log = Logger('FavoriteRepositoryLocal');

  final List<Favorite> _favorites = [];
  int _sequenceId = 0;

  @override
  Future<Result<List<Favorite>>> getFavorites() async {
    _log.fine('Getting ${_favorites.length} favorites');
    return Result.ok(List.from(_favorites));
  }

  @override
  Future<Result<void>> addFavorite(String destinationRef) async {
    _log.fine('Adding favorite: $destinationRef');

    final favorite = Favorite(
      id: _sequenceId++,
      destinationRef: destinationRef,
      createdAt: DateTime.now(),
    );

    _favorites.add(favorite);
    return Result.ok(null);
  }

  @override
  Future<Result<void>> removeFavorite(int id) async {
    _log.fine('Removing favorite: $id');
    _favorites.removeWhere((f) => f.id == id);
    return Result.ok(null);
  }

  @override
  Future<Result<bool>> isFavorite(String destinationRef) async {
    final exists = _favorites.any((f) => f.destinationRef == destinationRef);
    return Result.ok(exists);
  }
}
```

---

#### Step 4: Implement Remote Repository

**File**: `app/lib/data/repositories/favorite/favorite_repository_remote.dart`

```dart
import 'package:logging/logging.dart';
import '../../../domain/models/favorite/favorite.dart';
import '../../../utils/result.dart';
import '../../services/api/api_client.dart';
import 'favorite_repository.dart';

class FavoriteRepositoryRemote implements FavoriteRepository {
  final ApiClient _apiClient;
  final Logger _log = Logger('FavoriteRepositoryRemote');

  FavoriteRepositoryRemote({
    required ApiClient apiClient,
  }) : _apiClient = apiClient;

  @override
  Future<Result<List<Favorite>>> getFavorites() async {
    _log.fine('Fetching favorites from API');
    return await _apiClient.getFavorites();
  }

  @override
  Future<Result<void>> addFavorite(String destinationRef) async {
    _log.fine('Adding favorite via API: $destinationRef');
    return await _apiClient.postFavorite(destinationRef);
  }

  @override
  Future<Result<void>> removeFavorite(int id) async {
    _log.fine('Removing favorite via API: $id');
    return await _apiClient.deleteFavorite(id);
  }

  @override
  Future<Result<bool>> isFavorite(String destinationRef) async {
    final result = await getFavorites();
    return result.when(
      ok: (favorites) {
        final exists = favorites.any((f) => f.destinationRef == destinationRef);
        return Result.ok(exists);
      },
      error: (error) => Result.error(error),
    );
  }
}
```

---

#### Step 5: Register in Dependency Injection

**File**: [app/lib/config/dependencies.dart](../app/lib/config/dependencies.dart)

**Local**:
```dart
List<SingleChildWidget> get providersLocal {
  return [
    // ... existing providers

    // Favorite repository
    Provider(
      create: (context) => FavoriteRepositoryLocal() as FavoriteRepository,
    ),
  ];
}
```

**Remote**:
```dart
List<SingleChildWidget> get providersRemote {
  return [
    // ... existing providers

    // Favorite repository
    Provider(
      create: (context) => FavoriteRepositoryRemote(
        apiClient: context.read(),
      ) as FavoriteRepository,
    ),
  ];
}
```

---

#### Step 6: Create ViewModel

**File**: `app/lib/ui/favorites/view_models/favorites_viewmodel.dart`

```dart
import 'package:flutter/foundation.dart';
import 'package:logging/logging.dart';
import '../../../data/repositories/favorite/favorite_repository.dart';
import '../../../domain/models/favorite/favorite.dart';
import '../../../utils/command.dart';
import '../../../utils/result.dart';

class FavoritesViewModel extends ChangeNotifier {
  final FavoriteRepository _favoriteRepository;
  final Logger _log = Logger('FavoritesViewModel');

  FavoritesViewModel({
    required FavoriteRepository favoriteRepository,
  }) : _favoriteRepository = favoriteRepository {
    load = Command0(_load)..execute();
    toggleFavorite = Command1(_toggleFavorite);
  }

  late Command0 load;
  late Command1<void, String> toggleFavorite;

  List<Favorite> _favorites = [];
  List<Favorite> get favorites => _favorites;

  Future<Result<void>> _load() async {
    _log.fine('Loading favorites');

    final result = await _favoriteRepository.getFavorites();

    switch (result) {
      case Ok<List<Favorite>>():
        _favorites = result.value;
        _log.info('Loaded ${_favorites.length} favorites');
        notifyListeners();
      case Error<List<Favorite>>():
        _log.warning('Failed to load favorites', result.error);
    }

    return result as Result<void>;
  }

  Future<Result<void>> _toggleFavorite(String destinationRef) async {
    final isFavResult = await _favoriteRepository.isFavorite(destinationRef);

    if (isFavResult is Error) {
      return isFavResult as Result<void>;
    }

    final isFav = isFavResult.asOk.value;

    if (isFav) {
      // Remove favorite
      final favorite = _favorites.firstWhere(
        (f) => f.destinationRef == destinationRef,
      );
      final result = await _favoriteRepository.removeFavorite(favorite.id);

      if (result is Ok) {
        _favorites.removeWhere((f) => f.destinationRef == destinationRef);
        notifyListeners();
      }

      return result;
    } else {
      // Add favorite
      final result = await _favoriteRepository.addFavorite(destinationRef);

      if (result is Ok) {
        await _load(); // Reload to get new favorite with ID
      }

      return result;
    }
  }

  bool isFavorite(String destinationRef) {
    return _favorites.any((f) => f.destinationRef == destinationRef);
  }
}
```

---

#### Step 7: Create UI

**File**: `app/lib/ui/favorites/widgets/favorites_screen.dart`

```dart
import 'package:flutter/material.dart';
import '../view_models/favorites_viewmodel.dart';

class FavoritesScreen extends StatefulWidget {
  final FavoritesViewModel viewModel;

  const FavoritesScreen({
    super.key,
    required this.viewModel,
  });

  @override
  State<FavoritesScreen> createState() => _FavoritesScreenState();
}

class _FavoritesScreenState extends State<FavoritesScreen> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('My Favorites'),
      ),
      body: ListenableBuilder(
        listenable: widget.viewModel.load,
        builder: (context, _) {
          if (widget.viewModel.load.running) {
            return const Center(child: CircularProgressIndicator());
          }

          if (widget.viewModel.load.error != null) {
            return Center(
              child: Text('Error: ${widget.viewModel.load.error}'),
            );
          }

          if (widget.viewModel.favorites.isEmpty) {
            return const Center(
              child: Text('No favorites yet!'),
            );
          }

          return ListView.builder(
            itemCount: widget.viewModel.favorites.length,
            itemBuilder: (context, index) {
              final favorite = widget.viewModel.favorites[index];
              return ListTile(
                title: Text(favorite.destinationRef),
                subtitle: Text(
                  'Added ${_formatDate(favorite.createdAt)}',
                ),
                trailing: IconButton(
                  icon: const Icon(Icons.delete),
                  onPressed: () => widget.viewModel.toggleFavorite.execute(
                    favorite.destinationRef,
                  ),
                ),
              );
            },
          );
        },
      ),
    );
  }

  String _formatDate(DateTime date) {
    return '${date.month}/${date.day}/${date.year}';
  }
}
```

---

#### Step 8: Add Route

**File**: [app/lib/routing/routes.dart](../app/lib/routing/routes.dart)

```dart
abstract class Routes {
  static const String home = '/';
  static const String login = '/login';
  static const String favorites = '/favorites'; // ← Add this
  // ... other routes
}
```

**File**: [app/lib/routing/router.dart](../app/lib/routing/router.dart)

```dart
GoRoute(
  path: Routes.home,
  builder: (context, state) => HomeScreen(...),
  routes: [
    // ... existing routes

    // Favorites route
    GoRoute(
      path: Routes.favoritesRelative, // 'favorites'
      builder: (context, state) {
        return FavoritesScreen(
          viewModel: FavoritesViewModel(
            favoriteRepository: context.read(),
          ),
        );
      },
    ),
  ],
),
```

---

#### Step 9: Add Navigation

**In HomeScreen**: Add a favorites button

```dart
AppBar(
  actions: [
    IconButton(
      icon: const Icon(Icons.favorite),
      onPressed: () => context.push(Routes.favorites),
    ),
  ],
)
```

---

#### Step 10: Test

```bash
# Run tests
flutter test

# Run app
flutter run --target lib/main_development.dart
```

**Test manually**:
1. Open app
2. Tap favorites icon in app bar
3. Should see empty favorites screen
4. Go back and add some destinations to favorites (you'd need to add favorite buttons to destinations)
5. Check that favorites appear in favorites screen

---

## Testing Guide

### Running Tests

**All tests**:
```bash
flutter test
```

**Specific test file**:
```bash
flutter test test/ui/home/home_viewmodel_test.dart
```

**With coverage**:
```bash
flutter test --coverage
```

**Integration tests**:
```bash
# Local data
flutter test integration_test/app_local_data_test.dart

# Remote data (requires server)
flutter test integration_test/app_server_data_test.dart
```

---

### Writing a ViewModel Test

**Example**: Test HomeViewModel

**File**: `test/ui/home/home_viewmodel_test.dart`

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:compass_app/data/repositories/booking/booking_repository.dart';
import 'package:compass_app/data/repositories/user/user_repository.dart';
import 'package:compass_app/ui/home/view_models/home_viewmodel.dart';
import 'package:compass_app/domain/models/booking/booking_summary.dart';
import 'package:compass_app/domain/models/user/user.dart';
import 'package:compass_app/utils/result.dart';

// Mock classes
class MockBookingRepository extends Mock implements BookingRepository {}
class MockUserRepository extends Mock implements UserRepository {}

void main() {
  group('HomeViewModel', () {
    late MockBookingRepository mockBookingRepo;
    late MockUserRepository mockUserRepo;

    setUp(() {
      mockBookingRepo = MockBookingRepository();
      mockUserRepo = MockUserRepository();
    });

    test('loads bookings and user on initialization', () async {
      // Arrange
      final bookings = [
        BookingSummary(
          id: 1,
          name: 'Paris',
          startDate: DateTime(2025, 6, 1),
          endDate: DateTime(2025, 6, 10),
        ),
      ];

      final user = User(name: 'Test User', picture: 'user.jpg');

      when(() => mockBookingRepo.getBookingsList()).thenAnswer(
        (_) async => Result.ok(bookings),
      );

      when(() => mockUserRepo.getUser()).thenAnswer(
        (_) async => Result.ok(user),
      );

      // Act
      final viewModel = HomeViewModel(
        bookingRepository: mockBookingRepo,
        userRepository: mockUserRepo,
      );

      // Wait for load command to complete
      await Future.delayed(Duration(milliseconds: 100));

      // Assert
      expect(viewModel.bookings, bookings);
      expect(viewModel.user, user);
      expect(viewModel.load.completed, isTrue);
    });

    test('handles error when loading bookings fails', () async {
      // Arrange
      when(() => mockBookingRepo.getBookingsList()).thenAnswer(
        (_) async => Result.error(Exception('Network error')),
      );

      when(() => mockUserRepo.getUser()).thenAnswer(
        (_) async => Result.ok(User(name: 'Test', picture: 'test.jpg')),
      );

      // Act
      final viewModel = HomeViewModel(
        bookingRepository: mockBookingRepo,
        userRepository: mockUserRepo,
      );

      await Future.delayed(Duration(milliseconds: 100));

      // Assert
      expect(viewModel.bookings, isEmpty);
      expect(viewModel.load.error, isNotNull);
    });

    test('deleteBooking removes booking from list', () async {
      // Arrange
      final bookings = [
        BookingSummary(
          id: 1,
          name: 'Paris',
          startDate: DateTime(2025, 6, 1),
          endDate: DateTime(2025, 6, 10),
        ),
        BookingSummary(
          id: 2,
          name: 'London',
          startDate: DateTime(2025, 7, 1),
          endDate: DateTime(2025, 7, 10),
        ),
      ];

      when(() => mockBookingRepo.getBookingsList()).thenAnswer(
        (_) async => Result.ok(bookings),
      );

      when(() => mockUserRepo.getUser()).thenAnswer(
        (_) async => Result.ok(User(name: 'Test', picture: 'test.jpg')),
      );

      when(() => mockBookingRepo.delete(1)).thenAnswer(
        (_) async => Result.ok(null),
      );

      final viewModel = HomeViewModel(
        bookingRepository: mockBookingRepo,
        userRepository: mockUserRepo,
      );

      await Future.delayed(Duration(milliseconds: 100));

      // Act
      await viewModel.deleteBooking.execute(1);

      // Assert
      expect(viewModel.bookings.length, 1);
      expect(viewModel.bookings.first.id, 2);
      expect(viewModel.deleteBooking.completed, isTrue);
    });
  });
}
```

---

## Debugging Tips

### Tip 1: Use Flutter DevTools

**Launch DevTools**:
```bash
flutter pub global activate devtools
flutter pub global run devtools
```

**Or** in VS Code: `Cmd+Shift+P` → "Flutter: Open DevTools"

**Features**:
- **Inspector**: View widget tree, examine properties
- **Timeline**: Performance profiling
- **Memory**: Track memory usage
- **Network**: Monitor HTTP requests
- **Logging**: View all log messages

---

### Tip 2: Add Breakpoints

**In VS Code**:
1. Click in the gutter next to line number
2. Red dot appears
3. Run in debug mode (`F5`)
4. App pauses when breakpoint is hit
5. Inspect variables, step through code

---

### Tip 3: Print Debugging

```dart
// Quick debugging
print('Bookings loaded: ${bookings.length}');

// Better: Use logging
final _log = Logger('MyClass');
_log.info('Bookings loaded: ${bookings.length}');
_log.warning('Failed to load', error);
```

---

### Tip 4: Check Network Requests

**In DevTools**:
1. Open Network tab
2. Run app in staging mode
3. See all HTTP requests
4. Check request/response bodies
5. Verify authentication headers

---

### Tip 5: Widget Inspector

**In DevTools Inspector**:
1. Select widget in tree
2. See all properties
3. Check constraints, size, padding
4. Toggle debug paint (shows layout boxes)
5. Toggle slow animations

---

## Code Style Guidelines

### Follow Existing Patterns

**ViewModels**:
```dart
// ✅ Good
class MyViewModel extends ChangeNotifier {
  final MyRepository _repository;
  late Command0 load;

  MyViewModel({required MyRepository repository})
      : _repository = repository {
    load = Command0(_load)..execute();
  }

  Future<Result<void>> _load() async { ... }
}

// ❌ Bad
class MyViewModel {
  MyRepository repository; // Missing underscore, not final
  void load() { ... } // Not using Command pattern
}
```

---

### Naming Conventions

| Type | Convention | Example |
|------|-----------|---------|
| **Classes** | PascalCase | `HomeViewModel` |
| **Files** | snake_case | `home_viewmodel.dart` |
| **Variables** | camelCase | `selectedContinent` |
| **Private** | _underscore | `_repository` |
| **Constants** | camelCase | `defaultGuests` |

---

### Import Order

```dart
// 1. Dart imports
import 'dart:async';

// 2. Flutter imports
import 'package:flutter/material.dart';

// 3. Package imports
import 'package:logging/logging.dart';
import 'package:provider/provider.dart';

// 4. Relative imports
import '../../../domain/models/booking.dart';
import '../../utils/result.dart';
```

---

## Troubleshooting

### Issue: "Could not find a file named pubspec.yaml"

**Solution**:
```bash
# Make sure you're in the app directory
cd app
flutter pub get
```

---

### Issue: "Target of URI doesn't exist" after adding model

**Solution**:
```bash
# Regenerate Freezed code
flutter pub run build_runner build --delete-conflicting-outputs
```

---

### Issue: "Provider not found"

**Solution**:
- Check that provider is registered in `config/dependencies.dart`
- Ensure you're using `context.read<Type>()` with correct type
- Verify you're calling it within `MultiProvider` tree

---

### Issue: "Hot reload didn't work"

**Solution**:
- Try hot restart (`R` in terminal)
- For model changes, always restart
- Check terminal for error messages

---

### Issue: Server not starting (staging mode)

**Solution**:
```bash
# Check that port 8080 is not in use
lsof -i :8080

# Kill process if needed
kill -9 <PID>

# Restart server
cd server
dart run
```

---

### Issue: Tests failing with "Provider not found"

**Solution**:
- Mock all dependencies
- Don't use real Provider in unit tests
- Use dependency injection directly

---

## Summary

You now know how to:

✅ **Set up** the development environment
✅ **Run** the app in development and staging modes
✅ **Navigate** the codebase effectively
✅ **Add** new fields, methods, and commands
✅ **Create** complete new features
✅ **Test** your code with unit and integration tests
✅ **Debug** issues using various tools
✅ **Follow** code style guidelines
✅ **Troubleshoot** common problems

**Next Steps**:
- Explore [Reference Guide](./06-reference-guide.md) for comprehensive API documentation
- Try adding a small feature on your own
- Read through existing code to understand patterns
- Join team discussions and code reviews

**Happy coding! 🚀**
