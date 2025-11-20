# 04 - Component Deep-Dive

## 📋 Table of Contents

- [Introduction](#introduction)
- [Application Entry Points](#application-entry-points)
- [Authentication Flow](#authentication-flow)
- [Booking Creation Flow](#booking-creation-flow)
- [Data Flow Patterns](#data-flow-patterns)
- [State Management Deep-Dive](#state-management-deep-dive)
- [Navigation System](#navigation-system)
- [API Integration](#api-integration)
- [Error Handling Strategy](#error-handling-strategy)
- [Performance Optimizations](#performance-optimizations)

---

## Introduction

This document provides a **deep dive into key components** of the Compass App, exploring how they work together to create a cohesive user experience.

**What you'll learn**:
- How the app initializes and bootstraps
- Complete authentication flow with diagrams
- Step-by-step booking creation process
- Data flow patterns throughout the app
- State management strategies
- Navigation architecture
- API integration details
- Error handling approaches

**Analogy**: If the architecture documentation is like a building's blueprint, this is like a **guided tour** showing how people move through the building, how systems interact, and how everything works in practice.

---

## Application Entry Points

The Compass App has **three entry points** for different environments:

### 1. Default Entry Point

**File**: [app/lib/main.dart](../app/lib/main.dart)

```dart
import 'main_development.dart' as development;

void main() {
  // Delegate to development main
  development.main();
}
```

**Purpose**: Default entry point that delegates to development mode.

**Why?** Allows running the app without specifying a target:
```bash
flutter run  # Uses main.dart → development mode
```

---

### 2. Development Entry Point

**File**: [app/lib/main_development.dart](../app/lib/main_development.dart)

```dart
import 'package:flutter/material.dart';
import 'package:logging/logging.dart';
import 'package:provider/provider.dart';

import 'config/dependencies.dart';

void main() {
  // Configure logging
  Logger.root.level = Level.WARNING; // Less verbose for development
  Logger.root.onRecord.listen((record) {
    print('${record.level.name}: ${record.time}: ${record.message}');
  });

  runApp(
    MultiProvider(
      providers: providersLocal, // ← Local data sources
      child: const MainApp(),
    ),
  );
}

class MainApp extends StatelessWidget {
  const MainApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      title: 'Compass App',
      theme: AppTheme.lightTheme,
      darkTheme: AppTheme.darkTheme,
      themeMode: ThemeMode.system, // Auto-switch based on system
      localizationsDelegates: const [
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
        GlobalCupertinoLocalizations.delegate,
      ],
      supportedLocales: const [
        Locale('en', ''), // English
      ],
      routerConfig: router(
        context.read<AuthRepository>(),
      ),
      debugShowCheckedModeBanner: false,
    );
  }
}
```

**Characteristics**:
- ✅ Uses `providersLocal` (in-memory data)
- ✅ Logging level: WARNING (less noise)
- ✅ No authentication required (dev mode)
- ✅ Fast iteration
- ✅ No backend server needed

**Run Command**:
```bash
flutter run --target lib/main_development.dart
```

---

### 3. Staging Entry Point

**File**: [app/lib/main_staging.dart](../app/lib/main_staging.dart)

```dart
import 'package:flutter/material.dart';
import 'package:logging/logging.dart';
import 'package:provider/provider.dart';

import 'config/dependencies.dart';

void main() {
  // Configure logging
  Logger.root.level = Level.ALL; // Verbose logging for debugging
  Logger.root.onRecord.listen((record) {
    print('${record.level.name}: ${record.time}: ${record.message}');
  });

  runApp(
    MultiProvider(
      providers: providersRemote, // ← Remote API sources
      child: const MainApp(),
    ),
  );
}

// Same MainApp class as development
```

**Characteristics**:
- ✅ Uses `providersRemote` (HTTP API)
- ✅ Logging level: ALL (verbose debugging)
- ✅ Full authentication flow
- ✅ Production-like behavior
- ❌ Requires backend server running

**Run Commands**:
```bash
# Terminal 1: Start backend server
cd server
dart run

# Terminal 2: Run app
cd app
flutter run --target lib/main_staging.dart
```

---

### Bootstrap Sequence

```
┌─────────────────────────────────────────┐
│  1. main() function executes            │
│     • Configure logging                  │
│     • Set log level                      │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  2. Initialize Provider tree             │
│     • Register all dependencies          │
│     • Create repositories                │
│     • Create use cases                   │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  3. Build MaterialApp.router             │
│     • Configure themes                   │
│     • Configure localization             │
│     • Initialize router                  │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  4. Router determines initial route      │
│     • Check authentication status        │
│     • Redirect if necessary              │
│     • Navigate to initial screen         │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  5. Build initial screen                 │
│     • Create ViewModel                   │
│     • ViewModel auto-loads data          │
│     • Render UI                          │
└─────────────────────────────────────────┘
```

---

## Authentication Flow

### Overview

The Compass App supports two authentication modes:

1. **Development Mode**: Always authenticated (no login required)
2. **Staging Mode**: JWT-based authentication with login screen

---

### Authentication Architecture

```
┌────────────────────────────────────────────────────────────┐
│                   AuthRepository                            │
│                  (Abstract Interface)                       │
│                                                             │
│  Future<bool> get isAuthenticated                          │
│  Future<Result<void>> login(email, password)               │
│  Future<void> logout()                                     │
└──────────────────┬─────────────────────────────────────────┘
                   │
         ┌─────────┴──────────┐
         ↓                    ↓
┌────────────────────┐  ┌────────────────────┐
│ AuthRepositoryDev  │  │AuthRepositoryRemote│
│                    │  │                    │
│ • Always authed    │  │ • JWT tokens       │
│ • No login needed  │  │ • SharedPreferences│
│ • Instant return   │  │ • API calls        │
└────────────────────┘  └────────────────────┘
```

---

### Development Mode Authentication

**File**: [app/lib/data/repositories/auth/auth_repository_dev.dart](../app/lib/data/repositories/auth/auth_repository_dev.dart)

```dart
class AuthRepositoryDev extends ChangeNotifier implements AuthRepository {
  @override
  Future<bool> get isAuthenticated async {
    return true; // Always authenticated!
  }

  @override
  Future<Result<void>> login({
    required String email,
    required String password,
  }) async {
    // Simulate brief delay
    await Future.delayed(const Duration(milliseconds: 300));
    return Result.ok(null); // Always succeeds
  }

  @override
  Future<void> logout() async {
    // No-op
  }
}
```

**Flow**:
```
1. App starts
   ↓
2. Router checks isAuthenticated
   ↓ (returns true)
3. Navigate to HomeScreen
   ↓
4. No login screen shown
```

---

### Staging Mode Authentication

**File**: [app/lib/data/repositories/auth/auth_repository_remote.dart](../app/lib/data/repositories/auth/auth_repository_remote.dart)

```dart
class AuthRepositoryRemote extends ChangeNotifier implements AuthRepository {
  final AuthApiClient _authApiClient;
  final SharedPreferencesService _sharedPreferencesService;
  final Logger _log = Logger('AuthRepositoryRemote');

  AuthRepositoryRemote({
    required AuthApiClient authApiClient,
    required SharedPreferencesService sharedPreferencesService,
  })  : _authApiClient = authApiClient,
        _sharedPreferencesService = sharedPreferencesService;

  @override
  Future<bool> get isAuthenticated async {
    final token = await _sharedPreferencesService.fetchToken();
    final isAuth = token != null;
    _log.fine('isAuthenticated: $isAuth');
    return isAuth;
  }

  @override
  Future<Result<void>> login({
    required String email,
    required String password,
  }) async {
    _log.fine('Logging in with email: $email');

    // Call login API
    final result = await _authApiClient.login(
      email: email,
      password: password,
    );

    return result.when(
      ok: (loginResponse) async {
        // Save token to SharedPreferences
        await _sharedPreferencesService.saveToken(loginResponse.token);
        _log.info('Login successful, token saved');

        // Notify listeners (triggers router redirect)
        notifyListeners();

        return Result.ok(null);
      },
      error: (error) {
        _log.warning('Login failed: $error');
        return Result.error(error);
      },
    );
  }

  @override
  Future<void> logout() async {
    _log.info('Logging out');

    // Remove token
    await _sharedPreferencesService.saveToken(null);

    // Notify listeners (triggers redirect to login)
    notifyListeners();
  }
}
```

---

### Login Flow (Staging Mode)

**Step-by-Step Sequence**:

```
┌─────────────────────────────────────────────────────────────┐
│  1. App Starts                                              │
│     Router checks authRepository.isAuthenticated            │
│     ↓ Returns false (no token in SharedPreferences)         │
└────────────────┬────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────┐
│  2. Router Redirects to /login                              │
│     LoginScreen is displayed                                │
└────────────────┬────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────┐
│  3. User Enters Credentials                                 │
│     • Email: user@example.com                               │
│     • Password: ********                                    │
│     • Taps "Login" button                                   │
└────────────────┬────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────┐
│  4. LoginViewModel.login.execute()                          │
│     Command sets running = true                             │
│     UI shows loading spinner                                │
└────────────────┬────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────┐
│  5. AuthRepository.login()                                  │
│     Creates LoginRequest(email, password)                   │
│     POST to /login endpoint                                 │
└────────────────┬────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────┐
│  6. Server Validates Credentials                            │
│     ✓ Valid → Returns JWT token                            │
│     ✗ Invalid → Returns error                              │
└────────────────┬────────────────────────────────────────────┘
                 ↓ (Success path)
┌─────────────────────────────────────────────────────────────┐
│  7. AuthRepository Saves Token                              │
│     SharedPreferences.setString('auth_token', token)        │
│     notifyListeners() → Router re-evaluates auth            │
└────────────────┬────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────┐
│  8. Router Detects Auth Change                              │
│     isAuthenticated now returns true                        │
│     Redirects from /login to /                              │
└────────────────┬────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────┐
│  9. HomeScreen Displayed                                    │
│     User is now authenticated                               │
│     Token included in all API requests                      │
└─────────────────────────────────────────────────────────────┘
```

---

### Login Screen Implementation

**ViewModel**: [app/lib/ui/auth/view_models/login_viewmodel.dart](../app/lib/ui/auth/view_models/login_viewmodel.dart)

```dart
class LoginViewModel extends ChangeNotifier {
  final AuthRepository _authRepository;

  LoginViewModel({
    required AuthRepository authRepository,
  }) : _authRepository = authRepository {
    login = Command0(_login);
  }

  late Command0 login;

  String _email = '';
  String _password = '';

  String get email => _email;
  String get password => _password;

  set email(String value) {
    _email = value;
    notifyListeners();
  }

  set password(String value) {
    _password = value;
    notifyListeners();
  }

  bool get isValid => _email.isNotEmpty && _password.isNotEmpty;

  Future<Result<void>> _login() async {
    return await _authRepository.login(
      email: _email,
      password: _password,
    );
  }
}
```

**View**: [app/lib/ui/auth/widgets/login_screen.dart](../app/lib/ui/auth/widgets/login_screen.dart)

```dart
class LoginScreen extends StatefulWidget {
  final LoginViewModel viewModel;

  const LoginScreen({
    super.key,
    required this.viewModel,
  });

  @override
  State<LoginScreen> createState() => _LoginScreenState();
}

class _LoginScreenState extends State<LoginScreen> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(24.0),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              // Logo
              FlutterLogo(size: 100),
              SizedBox(height: 32),

              // Title
              Text(
                'Welcome to Compass',
                style: Theme.of(context).textTheme.headlineMedium,
              ),
              SizedBox(height: 32),

              // Email field
              TextField(
                decoration: InputDecoration(
                  labelText: 'Email',
                  border: OutlineInputBorder(),
                ),
                keyboardType: TextInputType.emailAddress,
                onChanged: (value) => widget.viewModel.email = value,
              ),
              SizedBox(height: 16),

              // Password field
              TextField(
                decoration: InputDecoration(
                  labelText: 'Password',
                  border: OutlineInputBorder(),
                ),
                obscureText: true,
                onChanged: (value) => widget.viewModel.password = value,
              ),
              SizedBox(height: 24),

              // Login button
              ListenableBuilder(
                listenable: widget.viewModel.login,
                builder: (context, _) {
                  return SizedBox(
                    width: double.infinity,
                    child: ElevatedButton(
                      onPressed: widget.viewModel.isValid && !widget.viewModel.login.running
                          ? _handleLogin
                          : null,
                      child: widget.viewModel.login.running
                          ? SizedBox(
                              height: 20,
                              width: 20,
                              child: CircularProgressIndicator(strokeWidth: 2),
                            )
                          : Text('Login'),
                    ),
                  );
                },
              ),

              // Error message
              ListenableBuilder(
                listenable: widget.viewModel.login,
                builder: (context, _) {
                  if (widget.viewModel.login.error != null) {
                    return Padding(
                      padding: const EdgeInsets.only(top: 16),
                      child: Text(
                        'Login failed: ${widget.viewModel.login.error}',
                        style: TextStyle(color: Colors.red),
                      ),
                    );
                  }
                  return SizedBox.shrink();
                },
              ),
            ],
          ),
        ),
      ),
    );
  }

  Future<void> _handleLogin() async {
    await widget.viewModel.login.execute();

    // No need to manually navigate - router handles it via auth state change!
  }
}
```

---

### Logout Flow

```
┌─────────────────────────────────────────┐
│  1. User Taps Logout Button             │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  2. AuthRepository.logout()             │
│     Remove token from SharedPreferences │
│     notifyListeners()                   │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  3. Router Listens to Auth Change       │
│     isAuthenticated now returns false   │
│     Redirects to /login                 │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  4. LoginScreen Displayed               │
│     User must login again               │
└─────────────────────────────────────────┘
```

**Logout Button**: [app/lib/ui/auth/widgets/logout_button.dart](../app/lib/ui/auth/widgets/logout_button.dart)

```dart
class LogoutButton extends StatelessWidget {
  const LogoutButton({super.key});

  @override
  Widget build(BuildContext context) {
    return IconButton(
      icon: const Icon(Icons.logout),
      onPressed: () => _handleLogout(context),
      tooltip: 'Logout',
    );
  }

  Future<void> _handleLogout(BuildContext context) async {
    final authRepository = context.read<AuthRepository>();
    await authRepository.logout();
    // Router automatically redirects to login
  }
}
```

---

### Router Integration

**File**: [app/lib/routing/router.dart](../app/lib/routing/router.dart)

```dart
GoRouter router(AuthRepository authRepository) {
  return GoRouter(
    initialLocation: Routes.home,
    debugLogDiagnostics: true,

    // Redirect based on authentication state
    redirect: (context, state) => _redirect(context, state, authRepository),

    // Listen to auth changes
    refreshListenable: authRepository, // ← Rebuilds router when auth changes

    routes: [
      GoRoute(
        path: Routes.login,
        builder: (context, state) {
          return LoginScreen(
            viewModel: LoginViewModel(
              authRepository: context.read(),
            ),
          );
        },
      ),
      GoRoute(
        path: Routes.home,
        builder: (context, state) {
          return HomeScreen(
            viewModel: HomeViewModel(
              bookingRepository: context.read(),
              userRepository: context.read(),
            ),
          );
        },
        // ... nested routes
      ),
    ],
  );
}

Future<String?> _redirect(
  BuildContext context,
  GoRouterState state,
  AuthRepository authRepository,
) async {
  final loggedIn = await authRepository.isAuthenticated;
  final loggingIn = state.matchedLocation == Routes.login;

  // Not logged in and not on login page → redirect to login
  if (!loggedIn && !loggingIn) {
    return Routes.login;
  }

  // Logged in but on login page → redirect to home
  if (loggedIn && loggingIn) {
    return Routes.home;
  }

  // No redirect needed
  return null;
}
```

**Key Features**:
- `refreshListenable: authRepository` → Router rebuilds when auth state changes
- `redirect` callback → Centralized redirect logic
- Automatic navigation on login/logout

---

## Booking Creation Flow

### Overview

Creating a booking is a **multi-step process** spanning 5 screens:

1. **HomeScreen**: Start new trip
2. **SearchFormScreen**: Select continent, dates, guests
3. **ResultsScreen**: Choose destination
4. **ActivitiesScreen**: Pick activities
5. **BookingScreen**: Confirm and create

**Temporary State**: User selections are stored in `ItineraryConfig` (in-memory) until final confirmation.

---

### Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│  Step 1: HomeScreen                                         │
│  • User taps "New Trip" FAB                                 │
│  • Navigate to /search                                      │
└────────────────┬────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 2: SearchFormScreen                                   │
│  • Select continent: "Europe"                               │
│  • Pick dates: June 1-10, 2025                              │
│  • Choose guests: 2                                         │
│  • Tap "Search"                                             │
│                                                             │
│  ItineraryConfig saved:                                     │
│  {                                                          │
│    continent: "Europe",                                     │
│    startDate: 2025-06-01,                                   │
│    endDate: 2025-06-10,                                     │
│    guests: 2                                                │
│  }                                                          │
│                                                             │
│  • Navigate to /results                                     │
└────────────────┬────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 3: ResultsScreen                                      │
│  • Load destinations in Europe                              │
│  • User selects "Paris, France"                             │
│  • Tap "Next"                                               │
│                                                             │
│  ItineraryConfig updated:                                   │
│  {                                                          │
│    continent: "Europe",                                     │
│    startDate: 2025-06-01,                                   │
│    endDate: 2025-06-10,                                     │
│    guests: 2,                                               │
│    destination: "paris-france"  ← Added                     │
│  }                                                          │
│                                                             │
│  • Navigate to /activities                                  │
└────────────────┬────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 4: ActivitiesScreen                                   │
│  • Load activities for Paris                                │
│  • User selects:                                            │
│    - Eiffel Tower Tour                                      │
│    - Louvre Museum                                          │
│    - Seine River Cruise                                     │
│  • Tap "Continue"                                           │
│                                                             │
│  ItineraryConfig updated:                                   │
│  {                                                          │
│    continent: "Europe",                                     │
│    startDate: 2025-06-01,                                   │
│    endDate: 2025-06-10,                                     │
│    guests: 2,                                               │
│    destination: "paris-france",                             │
│    activities: [                          ← Added           │
│      "eiffel-tower",                                        │
│      "louvre-museum",                                       │
│      "seine-river-cruise"                                   │
│    ]                                                        │
│  }                                                          │
│                                                             │
│  • Navigate to /booking                                     │
└────────────────┬────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 5: BookingScreen                                      │
│  • Display booking summary                                  │
│  • User reviews details                                     │
│  • Tap "Confirm"                                            │
│                                                             │
│  BookingCreateUseCase.createFrom(config):                   │
│    1. Fetch full Destination from ref                       │
│    2. Fetch full Activities from refs                       │
│    3. Create Booking entity                                 │
│    4. Persist to BookingRepository                          │
│                                                             │
│  Booking created:                                           │
│  {                                                          │
│    id: 123,                                                 │
│    startDate: 2025-06-01,                                   │
│    endDate: 2025-06-10,                                     │
│    destination: <full Destination object>,                  │
│    activity: [<full Activity objects>]                      │
│  }                                                          │
│                                                             │
│  • Show success message                                     │
│  • Enable "Share" button                                    │
└─────────────────────────────────────────────────────────────┘
```

---

### Step 2: Search Form (Detailed)

**ViewModel**: [app/lib/ui/search_form/view_models/search_form_viewmodel.dart](../app/lib/ui/search_form/view_models/search_form_viewmodel.dart)

```dart
class SearchFormViewModel extends ChangeNotifier {
  final ContinentRepository _continentRepository;
  final ItineraryConfigRepository _itineraryConfigRepository;
  final Logger _log = Logger('SearchFormViewModel');

  SearchFormViewModel({
    required ContinentRepository continentRepository,
    required ItineraryConfigRepository itineraryConfigRepository,
  })  : _continentRepository = continentRepository,
        _itineraryConfigRepository = itineraryConfigRepository {
    load = Command0(_load)..execute();
    updateItineraryConfig = Command0(_updateItineraryConfig);
  }

  late Command0 load;
  late Command0 updateItineraryConfig;

  // State
  List<Continent> _continents = [];
  String? _selectedContinent;
  DateTime? _startDate;
  DateTime? _endDate;
  int _guests = 1;

  // Getters
  List<Continent> get continents => _continents;
  String? get selectedContinent => _selectedContinent;
  DateTime? get startDate => _startDate;
  DateTime? get endDate => _endDate;
  int get guests => _guests;

  // Validation
  bool get valid =>
      _selectedContinent != null &&
      _startDate != null &&
      _endDate != null &&
      _guests > 0;

  // Setters
  set selectedContinent(String? value) {
    _selectedContinent = value;
    notifyListeners();
  }

  set startDate(DateTime? value) {
    _startDate = value;
    // Ensure end date is after start date
    if (_endDate != null && value != null && _endDate!.isBefore(value)) {
      _endDate = value;
    }
    notifyListeners();
  }

  set endDate(DateTime? value) {
    _endDate = value;
    notifyListeners();
  }

  set guests(int value) {
    _guests = value.clamp(1, 10); // Max 10 guests
    notifyListeners();
  }

  Future<Result<void>> _load() async {
    _log.fine('Loading continents');

    final result = await _continentRepository.getContinents();

    switch (result) {
      case Ok<List<Continent>>():
        _continents = result.value;
        _log.info('Loaded ${_continents.length} continents');
        notifyListeners();
      case Error<List<Continent>>():
        _log.warning('Failed to load continents', result.error);
    }

    return result as Result<void>;
  }

  Future<Result<void>> _updateItineraryConfig() async {
    _log.fine('Saving search criteria to ItineraryConfig');

    final config = ItineraryConfig(
      continent: _selectedContinent,
      startDate: _startDate,
      endDate: _endDate,
      guests: _guests,
    );

    final result = await _itineraryConfigRepository.setItineraryConfig(config);

    switch (result) {
      case Ok():
        _log.info('Saved ItineraryConfig');
      case Error():
        _log.warning('Failed to save ItineraryConfig', result.error);
    }

    return result;
  }
}
```

**View**: Displays continent cards, date pickers, and guest counter.

---

### Step 3: Results Screen (Detailed)

**ViewModel**: [app/lib/ui/results/view_models/results_viewmodel.dart](../app/lib/ui/results/view_models/results_viewmodel.dart)

```dart
class ResultsViewModel extends ChangeNotifier {
  final DestinationRepository _destinationRepository;
  final ItineraryConfigRepository _itineraryConfigRepository;
  final Logger _log = Logger('ResultsViewModel');

  ResultsViewModel({
    required DestinationRepository destinationRepository,
    required ItineraryConfigRepository itineraryConfigRepository,
  })  : _destinationRepository = destinationRepository,
        _itineraryConfigRepository = itineraryConfigRepository {
    search = Command0(_search)..execute();
    updateItineraryConfig = Command0(_updateItineraryConfig);
  }

  late Command0 search;
  late Command0 updateItineraryConfig;

  List<Destination> _destinations = [];
  String? _selectedDestination;

  List<Destination> get destinations => _destinations;
  String? get selectedDestination => _selectedDestination;

  set selectedDestination(String? value) {
    _selectedDestination = value;
    notifyListeners();
  }

  bool get valid => _selectedDestination != null;

  Future<Result<void>> _search() async {
    _log.fine('Searching destinations');

    // Load itinerary config
    final configResult = await _itineraryConfigRepository.getItineraryConfig();
    if (configResult is Error) {
      return configResult as Result<void>;
    }

    final config = configResult.asOk.value;

    // Load all destinations
    final destinationsResult = await _destinationRepository.getDestinations();

    return destinationsResult.when(
      ok: (allDestinations) {
        // Filter by continent
        _destinations = allDestinations.where((d) {
          return d.continent == config.continent;
        }).toList();

        _log.info('Found ${_destinations.length} destinations in ${config.continent}');
        notifyListeners();
        return Result.ok(null);
      },
      error: (error) {
        _log.warning('Failed to load destinations', error);
        return Result.error(error);
      },
    );
  }

  Future<Result<void>> _updateItineraryConfig() async {
    _log.fine('Updating ItineraryConfig with destination');

    // Load current config
    final configResult = await _itineraryConfigRepository.getItineraryConfig();
    if (configResult is Error) {
      return configResult as Result<void>;
    }

    // Update with selected destination
    final updatedConfig = configResult.asOk.value.copyWith(
      destination: _selectedDestination,
    );

    return await _itineraryConfigRepository.setItineraryConfig(updatedConfig);
  }
}
```

**Key Pattern**: `copyWith` preserves previous steps' data while adding new selection.

---

### Step 4: Activities Screen (Detailed)

**ViewModel**: [app/lib/ui/activities/view_models/activities_viewmodel.dart](../app/lib/ui/activities/view_models/activities_viewmodel.dart)

```dart
class ActivitiesViewModel extends ChangeNotifier {
  final ActivityRepository _activityRepository;
  final ItineraryConfigRepository _itineraryConfigRepository;
  final Logger _log = Logger('ActivitiesViewModel');

  ActivitiesViewModel({
    required ActivityRepository activityRepository,
    required ItineraryConfigRepository itineraryConfigRepository,
  })  : _activityRepository = activityRepository,
        _itineraryConfigRepository = itineraryConfigRepository {
    loadActivities = Command0(_loadActivities)..execute();
    saveActivities = Command0(_saveActivities);
  }

  late Command0 loadActivities;
  late Command0 saveActivities;

  List<Activity> _activities = [];
  List<String> _selectedActivities = [];

  List<Activity> get activities => _activities;
  List<String> get selectedActivities => _selectedActivities;

  // Split activities by time of day for better UX
  List<Activity> get daytimeActivities => _activities.where((a) {
        return a.timeOfDay == TimeOfDay.any ||
            a.timeOfDay == TimeOfDay.morning ||
            a.timeOfDay == TimeOfDay.afternoon;
      }).toList();

  List<Activity> get eveningActivities => _activities.where((a) {
        return a.timeOfDay == TimeOfDay.evening ||
            a.timeOfDay == TimeOfDay.night;
      }).toList();

  bool get valid => _selectedActivities.isNotEmpty;

  void addActivity(String ref) {
    if (!_selectedActivities.contains(ref)) {
      _selectedActivities.add(ref);
      _log.fine('Added activity: $ref');
      notifyListeners();
    }
  }

  void removeActivity(String ref) {
    _selectedActivities.remove(ref);
    _log.fine('Removed activity: $ref');
    notifyListeners();
  }

  bool isSelected(String ref) => _selectedActivities.contains(ref);

  Future<Result<void>> _loadActivities() async {
    _log.fine('Loading activities');

    // Load current config
    final configResult = await _itineraryConfigRepository.getItineraryConfig();
    if (configResult is Error) {
      return configResult as Result<void>;
    }

    final config = configResult.asOk.value;
    final destinationRef = config.destination;

    if (destinationRef == null) {
      return Result.error(Exception('No destination selected'));
    }

    // Load activities for destination
    final result = await _activityRepository.getByDestination(destinationRef);

    return result.when(
      ok: (activities) {
        _activities = activities;
        _log.info('Loaded ${_activities.length} activities');
        notifyListeners();
        return Result.ok(null);
      },
      error: (error) {
        _log.warning('Failed to load activities', error);
        return Result.error(error);
      },
    );
  }

  Future<Result<void>> _saveActivities() async {
    _log.fine('Saving ${_selectedActivities.length} selected activities');

    // Load current config
    final configResult = await _itineraryConfigRepository.getItineraryConfig();
    if (configResult is Error) {
      return configResult as Result<void>;
    }

    // Update with selected activities
    final updatedConfig = configResult.asOk.value.copyWith(
      activities: _selectedActivities,
    );

    return await _itineraryConfigRepository.setItineraryConfig(updatedConfig);
  }
}
```

**UI Feature**: Activities are split into "Daytime" and "Evening" sections for better organization.

---

### Step 5: Booking Confirmation (Detailed)

**ViewModel**: [app/lib/ui/booking/view_models/booking_viewmodel.dart](../app/lib/ui/booking/view_models/booking_viewmodel.dart)

```dart
class BookingViewModel extends ChangeNotifier {
  final BookingRepository _bookingRepository;
  final ItineraryConfigRepository _itineraryConfigRepository;
  final BookingCreateUseCase _bookingCreateUseCase;
  final BookingShareUseCase _bookingShareUseCase;
  final int? bookingId;
  final Logger _log = Logger('BookingViewModel');

  BookingViewModel({
    required BookingRepository bookingRepository,
    required ItineraryConfigRepository itineraryConfigRepository,
    required BookingCreateUseCase bookingCreateUseCase,
    required BookingShareUseCase bookingShareUseCase,
    this.bookingId,
  })  : _bookingRepository = bookingRepository,
        _itineraryConfigRepository = itineraryConfigRepository,
        _bookingCreateUseCase = bookingCreateUseCase,
        _bookingShareUseCase = bookingShareUseCase {
    load = Command0(_load)..execute();
    createBooking = Command0(_createBooking);
    shareBooking = Command0(_shareBooking);
  }

  late Command0 load;
  late Command0 createBooking;
  late Command0 shareBooking;

  Booking? _booking;
  Booking? get booking => _booking;

  Future<Result<void>> _load() async {
    if (bookingId != null) {
      // Load existing booking
      _log.fine('Loading existing booking $bookingId');
      final result = await _bookingRepository.getBooking(bookingId!);

      return result.when(
        ok: (booking) {
          _booking = booking;
          _log.info('Loaded booking: ${booking.destination.name}');
          notifyListeners();
          return Result.ok(null);
        },
        error: (error) {
          _log.warning('Failed to load booking', error);
          return Result.error(error);
        },
      );
    } else {
      // New booking - no load needed
      return Result.ok(null);
    }
  }

  Future<Result<void>> _createBooking() async {
    _log.fine('Creating new booking');

    // Load itinerary config
    final configResult = await _itineraryConfigRepository.getItineraryConfig();
    if (configResult is Error) {
      return configResult as Result<void>;
    }

    final config = configResult.asOk.value;

    // Use case creates booking from config
    final result = await _bookingCreateUseCase.createFrom(config);

    return result.when(
      ok: (booking) {
        _booking = booking;
        _log.info('Created booking: ${booking.destination.name}');
        notifyListeners();
        return Result.ok(null);
      },
      error: (error) {
        _log.warning('Failed to create booking', error);
        return Result.error(error);
      },
    );
  }

  Future<Result<void>> _shareBooking() async {
    if (_booking == null) {
      return Result.error(Exception('No booking to share'));
    }

    _log.fine('Sharing booking');
    return await _bookingShareUseCase.shareBooking(_booking!);
  }
}
```

**Use Case**: [app/lib/domain/use_cases/booking/booking_create_use_case.dart](../app/lib/domain/use_cases/booking/booking_create_use_case.dart)

```dart
class BookingCreateUseCase {
  final DestinationRepository _destinationRepository;
  final ActivityRepository _activityRepository;
  final BookingRepository _bookingRepository;

  Future<Result<Booking>> createFrom(ItineraryConfig config) async {
    // 1. Validate config
    if (config.destination == null) {
      return Result.error(Exception('Destination not selected'));
    }
    if (config.startDate == null || config.endDate == null) {
      return Result.error(Exception('Dates not selected'));
    }

    // 2. Fetch full destination object
    final destinationResult = await _fetchDestination(config.destination!);
    if (destinationResult is Error) {
      return destinationResult as Result<Booking>;
    }

    // 3. Fetch full activity objects
    final activitiesResult = await _activityRepository.getByDestination(
      config.destination!,
    );
    if (activitiesResult is Error) {
      return activitiesResult as Result<Booking>;
    }

    // Filter to selected activities
    final selectedActivities = activitiesResult.asOk.value
        .where((a) => config.activities.contains(a.ref))
        .toList();

    // 4. Create booking entity
    final booking = Booking(
      startDate: config.startDate!,
      endDate: config.endDate!,
      destination: destinationResult.asOk.value,
      activity: selectedActivities,
    );

    // 5. Persist booking
    final createResult = await _bookingRepository.createBooking(booking);

    return createResult.when(
      ok: (_) {
        log.info('Successfully created booking');
        return Result.ok(booking);
      },
      error: (error) {
        log.warning('Failed to create booking', error);
        return Result.error(error);
      },
    );
  }

  Future<Result<Destination>> _fetchDestination(String ref) async {
    final result = await _destinationRepository.getDestinations();

    return result.when(
      ok: (destinations) {
        try {
          final destination = destinations.firstWhere((d) => d.ref == ref);
          return Result.ok(destination);
        } catch (e) {
          return Result.error(Exception('Destination not found: $ref'));
        }
      },
      error: (error) => Result.error(error),
    );
  }
}
```

---

## Data Flow Patterns

### Pattern 1: Load Data on Screen Entry

**Used in**: HomeScreen, ResultsScreen, ActivitiesScreen, BookingScreen

**Flow**:
```
1. Router creates Screen with ViewModel
   ↓
2. ViewModel constructor creates Command0
   ↓ Command0(_load)..execute()
3. Command executes immediately
   ↓
4. ViewModel calls repository
   ↓
5. Repository returns Result<T>
   ↓
6. ViewModel updates state
   ↓ notifyListeners()
7. ListenableBuilder rebuilds
   ↓
8. UI shows data
```

**Example**:
```dart
class HomeViewModel extends ChangeNotifier {
  HomeViewModel({required BookingRepository repository}) {
    load = Command0(_load)..execute(); // Auto-execute!
  }

  late Command0 load;

  Future<Result<void>> _load() async {
    final result = await repository.getBookingsList();
    // Update state
  }
}
```

---

### Pattern 2: User Action Updates State

**Used in**: Delete booking, select continent, pick activities

**Flow**:
```
1. User interacts (tap button, select item)
   ↓
2. View calls ViewModel method or setter
   ↓
3. ViewModel updates state
   ↓ notifyListeners()
4. ListenableBuilder rebuilds
   ↓
5. UI reflects change
```

**Example**:
```dart
// In ViewModel
set selectedContinent(String? value) {
  _selectedContinent = value;
  notifyListeners(); // Trigger rebuild
}

// In View
onTap: () => viewModel.selectedContinent = 'Europe',
```

---

### Pattern 3: Multi-Step Form Flow

**Used in**: Booking creation (search → results → activities → booking)

**Flow**:
```
Step N Screen:
1. Load current ItineraryConfig
   ↓
2. User makes selection
   ↓
3. Update ItineraryConfig with copyWith
   ↓
4. Save to ItineraryConfigRepository
   ↓
5. Navigate to Step N+1

Final Step:
1. Load ItineraryConfig
   ↓
2. BookingCreateUseCase transforms config → Booking
   ↓
3. Save to BookingRepository
   ↓
4. Clear ItineraryConfig (optional)
```

**Example**:
```dart
// Step 2: Save continent selection
final config = ItineraryConfig(
  continent: selectedContinent,
  startDate: startDate,
  endDate: endDate,
  guests: guests,
);
await repository.setItineraryConfig(config);

// Step 3: Add destination to config
final currentConfig = await repository.getItineraryConfig();
final updatedConfig = currentConfig.copyWith(
  destination: selectedDestination,
);
await repository.setItineraryConfig(updatedConfig);
```

---

## State Management Deep-Dive

### ChangeNotifier Pattern

**How it works**:

1. **ViewModel extends ChangeNotifier**
   ```dart
   class HomeViewModel extends ChangeNotifier {
     List<Booking> _bookings = [];

     List<Booking> get bookings => _bookings;
   }
   ```

2. **State updates call notifyListeners()**
   ```dart
   Future<void> _load() async {
     final result = await repository.getBookings();
     _bookings = result.value;
     notifyListeners(); // ← Notify observers
   }
   ```

3. **UI listens with ListenableBuilder**
   ```dart
   ListenableBuilder(
     listenable: viewModel,
     builder: (context, _) {
       // Rebuilds when notifyListeners() is called
       return ListView(children: viewModel.bookings.map(...));
     },
   )
   ```

---

### Selective Rebuilding

**Problem**: Calling `notifyListeners()` rebuilds ALL listeners, even if only one piece of data changed.

**Solution 1**: Multiple ViewModels
```dart
class HomeViewModel extends ChangeNotifier {
  // Main data
}

class HomeAppBarViewModel extends ChangeNotifier {
  // Just user data
}

// In UI
ListenableBuilder(listenable: homeViewModel, ...)     // List
ListenableBuilder(listenable: appBarViewModel, ...)   // AppBar
```

**Solution 2**: Listen to specific Commands
```dart
// Don't rebuild entire screen
ListenableBuilder(
  listenable: viewModel, // ❌ Rebuilds on any change
  ...
)

// Rebuild only when load command changes
ListenableBuilder(
  listenable: viewModel.load, // ✅ Rebuilds only for load
  ...
)
```

---

### Command State Management

Commands automatically manage:
- `running`: Is action in progress?
- `error`: Did action fail?
- `result`: What was returned?
- `completed`: Did action succeed?

**Example UI patterns**:

**Loading State**:
```dart
ListenableBuilder(
  listenable: viewModel.load,
  builder: (context, _) {
    if (viewModel.load.running) {
      return CircularProgressIndicator();
    }
    // Show data
  },
)
```

**Error State**:
```dart
if (viewModel.load.error != null) {
  return ErrorIndicator(
    error: viewModel.load.error,
    onRetry: viewModel.load.execute,
  );
}
```

**Disabled Button While Running**:
```dart
ElevatedButton(
  onPressed: viewModel.deleteBooking.running
      ? null // Disabled
      : () => viewModel.deleteBooking.execute(id),
  child: Text('Delete'),
)
```

---

## Navigation System

### go_router Configuration

**File**: [app/lib/routing/router.dart](../app/lib/routing/router.dart)

**Features**:
- Declarative routing
- Authentication redirects
- Nested routes
- Query parameters
- Deep linking

**Route Tree**:
```
/login (LoginScreen)
/
├─ /search
├─ /results
├─ /activities
└─ /booking
   └─ /booking?id=123
```

**Navigation Methods**:

| Method | Use Case | Example |
|--------|----------|---------|
| `context.go('/path')` | Replace current route | `context.go(Routes.home)` |
| `context.push('/path')` | Add to stack | `context.push(Routes.bookingWithId(123))` |
| `context.pop()` | Go back | `context.pop()` |

---

### Deep Linking

**Supported URL formats**:
```
compass://                    → Home screen
compass://search              → Search form
compass://booking?id=123      → Specific booking
```

**Configuration** (in `AndroidManifest.xml` and `Info.plist`):
```xml
<intent-filter>
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data android:scheme="compass" />
</intent-filter>
```

---

## API Integration

### API Client Architecture

```
┌─────────────────────────────────────┐
│         BookingViewModel             │
└──────────────┬──────────────────────┘
               │ Uses
               ▼
┌─────────────────────────────────────┐
│      BookingRepositoryRemote         │
└──────────────┬──────────────────────┘
               │ Uses
               ▼
┌─────────────────────────────────────┐
│           ApiClient                  │
│  • HTTP requests                     │
│  • Auth header injection             │
│  • Error handling                    │
└──────────────┬──────────────────────┘
               │ Uses
               ▼
┌─────────────────────────────────────┐
│      AuthHeaderProvider              │
│  • Gets token from SharedPreferences │
│  • Formats Authorization header      │
└─────────────────────────────────────┘
```

---

### API Request Flow

```
1. ViewModel calls repository
   ↓
2. Repository calls apiClient.getBookings()
   ↓
3. ApiClient gets auth header
   ↓ AuthHeaderProvider.getAuthHeader()
4. Load token from SharedPreferences
   ↓
5. Format header: "Authorization: Bearer {token}"
   ↓
6. Make HTTP GET request
   ↓
7. Server validates token
   ↓
8. Server returns JSON response
   ↓
9. Parse JSON to domain models
   ↓
10. Return Result<List<Booking>>
   ↓
11. Repository returns to ViewModel
   ↓
12. ViewModel updates UI
```

---

### Error Handling in API Calls

**File**: [app/lib/data/services/api/api_client.dart](../app/lib/data/services/api/api_client.dart)

```dart
Future<Result<List<Booking>>> getBookings() async {
  try {
    final uri = Uri.parse('$baseUrl/booking');
    final headers = await _authHeaderProvider.getAuthHeader();

    final response = await _client.get(uri, headers: headers);

    // Check HTTP status
    if (response.statusCode == 401) {
      return Result.error(Exception('Unauthorized - please login'));
    }

    if (response.statusCode != 200) {
      return Result.error(
        Exception('HTTP ${response.statusCode}: ${response.body}'),
      );
    }

    // Parse JSON
    final List<dynamic> json = jsonDecode(response.body);
    final bookings = json.map((item) =>
        Booking.fromJson(item as Map<String, dynamic>)
    ).toList();

    return Result.ok(bookings);
  } on SocketException catch (e) {
    return Result.error(Exception('Network error: $e'));
  } on FormatException catch (e) {
    return Result.error(Exception('Invalid JSON: $e'));
  } on Exception catch (e) {
    return Result.error(e);
  }
}
```

**Error Types Handled**:
- HTTP errors (401, 404, 500, etc.)
- Network errors (no connection)
- JSON parsing errors
- Unexpected exceptions

---

## Error Handling Strategy

### Layer-by-Layer Error Handling

**API Client Layer**:
- Convert HTTP errors to `Result.error`
- Log errors
- Return typed errors

**Repository Layer**:
- Pass through errors from API client
- Add context-specific logging
- Transform errors if needed

**ViewModel Layer**:
- Handle errors from repositories
- Update UI state (show error message)
- Provide retry mechanism

**View Layer**:
- Display error messages
- Show retry buttons
- Provide user feedback

---

### Example: Complete Error Flow

```dart
// API Client
Future<Result<Booking>> getBooking(int id) async {
  try {
    final response = await http.get('/booking/$id');

    if (response.statusCode == 404) {
      return Result.error(Exception('Booking not found'));
    }

    return Result.ok(Booking.fromJson(response.data));
  } catch (e) {
    log.warning('API error', e);
    return Result.error(Exception('Network error: $e'));
  }
}

// Repository (pass through)
Future<Result<Booking>> getBooking(int id) async {
  return await _apiClient.getBooking(id);
}

// ViewModel
Future<Result<void>> _load() async {
  final result = await _repository.getBooking(bookingId);

  switch (result) {
    case Ok<Booking>():
      _booking = result.value;
      _error = null;

    case Error<Booking>():
      _booking = null;
      _error = result.error.toString();
      log.warning('Failed to load booking', result.error);
  }

  notifyListeners();
  return result as Result<void>;
}

// View
ListenableBuilder(
  listenable: viewModel.load,
  builder: (context, _) {
    if (viewModel.load.error != null) {
      return ErrorIndicator(
        title: 'Failed to load booking',
        message: viewModel.load.error.toString(),
        onRetry: viewModel.load.execute,
      );
    }
    // Show data
  },
)
```

---

## Performance Optimizations

### 1. Parallel Data Loading

**Example**: Load bookings and user simultaneously

```dart
Future<Result<void>> _load() async {
  // ❌ Sequential (slow)
  final bookings = await _bookingRepository.getBookingsList();
  final user = await _userRepository.getUser();

  // ✅ Parallel (fast)
  final results = await (
    _bookingRepository.getBookingsList(),
    _userRepository.getUser(),
  ).wait;

  // Process results
}
```

**Performance Gain**: ~2x faster when both requests take similar time.

---

### 2. Selective Rebuilding

**Example**: Only rebuild affected widgets

```dart
// ❌ Rebuild entire screen
ListenableBuilder(
  listenable: viewModel,
  builder: (context, _) => EntireScreen(),
)

// ✅ Rebuild only the list
Column(
  children: [
    AppBar(), // Static
    ListenableBuilder(
      listenable: viewModel,
      builder: (context, _) => BookingList(), // Only this rebuilds
    ),
  ],
)
```

---

### 3. Lazy Initialization

**Example**: Only create expensive objects when needed

```dart
// Dependency Injection
Provider(
  create: (context) => ExpensiveService(), // Created on first access
  lazy: true, // Default
)
```

---

### 4. Image Caching

**Package**: `cached_network_image`

```dart
CachedNetworkImage(
  imageUrl: destination.imageUrl,
  placeholder: (context, url) => CircularProgressIndicator(),
  errorWidget: (context, url, error) => Icon(Icons.error),
)
```

**Benefits**:
- Images cached on disk
- Automatic memory management
- Placeholder during loading

---

## Summary

This deep-dive covered:

✅ **Application Entry Points**: Three environments (default, development, staging)
✅ **Authentication Flow**: Dev mode (always authed) vs staging mode (JWT)
✅ **Booking Creation**: 5-step multi-screen flow with temporary state
✅ **Data Flow Patterns**: Load on entry, user actions, multi-step forms
✅ **State Management**: ChangeNotifier, selective rebuilding, Command pattern
✅ **Navigation**: go_router with auth redirects and deep linking
✅ **API Integration**: HTTP client with auth header injection
✅ **Error Handling**: Layer-by-layer error propagation with Result pattern
✅ **Performance**: Parallel loading, selective rebuilds, caching

**Next**: Explore [Getting Started Guide](./05-getting-started.md) for practical development workflows.
