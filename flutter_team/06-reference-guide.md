# 06 - Reference Guide

## 📋 Table of Contents

- [Introduction](#introduction)
- [Dependencies Reference](#dependencies-reference)
- [Models Reference](#models-reference)
- [Repositories Reference](#repositories-reference)
- [ViewModels Reference](#viewmodels-reference)
- [API Endpoints](#api-endpoints)
- [Routes Reference](#routes-reference)
- [Utility Classes](#utility-classes)
- [Testing Utilities](#testing-utilities)
- [Code Generation Commands](#code-generation-commands)

---

## Introduction

This is a **comprehensive reference guide** for the Compass App. Use this as a quick lookup when you need specific information about classes, methods, or configurations.

**How to use this guide**:
- Use `Cmd+F` / `Ctrl+F` to search for specific items
- Each section is organized alphabetically
- Includes file paths, method signatures, and usage examples

---

## Dependencies Reference

### Core Dependencies

| Package | Version | Purpose | Documentation |
|---------|---------|---------|---------------|
| **flutter** | SDK | UI framework | [flutter.dev](https://flutter.dev) |
| **provider** | ^6.1.2 | State management & DI | [pub.dev/packages/provider](https://pub.dev/packages/provider) |
| **go_router** | ^14.6.2 | Declarative routing | [pub.dev/packages/go_router](https://pub.dev/packages/go_router) |
| **freezed** | ^2.5.7 | Immutable models | [pub.dev/packages/freezed](https://pub.dev/packages/freezed) |
| **freezed_annotation** | ^2.4.4 | Freezed annotations | [pub.dev/packages/freezed_annotation](https://pub.dev/packages/freezed_annotation) |
| **json_annotation** | ^4.9.0 | JSON serialization annotations | [pub.dev/packages/json_annotation](https://pub.dev/packages/json_annotation) |
| **json_serializable** | ^6.9.0 | JSON code generation | [pub.dev/packages/json_serializable](https://pub.dev/packages/json_serializable) |

### UI Dependencies

| Package | Version | Purpose | Documentation |
|---------|---------|---------|---------------|
| **cached_network_image** | ^3.4.1 | Image loading and caching | [pub.dev/packages/cached_network_image](https://pub.dev/packages/cached_network_image) |
| **flutter_svg** | ^2.0.16 | SVG rendering | [pub.dev/packages/flutter_svg](https://pub.dev/packages/flutter_svg) |
| **google_fonts** | ^6.2.1 | Custom fonts | [pub.dev/packages/google_fonts](https://pub.dev/packages/google_fonts) |

### Utility Dependencies

| Package | Version | Purpose | Documentation |
|---------|---------|---------|---------------|
| **logging** | ^1.3.0 | Structured logging | [pub.dev/packages/logging](https://pub.dev/packages/logging) |
| **intl** | SDK | Internationalization | [pub.dev/packages/intl](https://pub.dev/packages/intl) |
| **collection** | ^1.18.0 | Collection utilities | [pub.dev/packages/collection](https://pub.dev/packages/collection) |
| **share_plus** | ^10.1.3 | Native share dialog | [pub.dev/packages/share_plus](https://pub.dev/packages/share_plus) |
| **shared_preferences** | ^2.3.5 | Key-value storage | [pub.dev/packages/shared_preferences](https://pub.dev/packages/shared_preferences) |

### Dev Dependencies

| Package | Version | Purpose | Documentation |
|---------|---------|---------|---------------|
| **flutter_test** | SDK | Unit testing | [flutter.dev/docs/testing](https://flutter.dev/docs/testing) |
| **integration_test** | SDK | E2E testing | [flutter.dev/docs/testing/integration-tests](https://flutter.dev/docs/testing/integration-tests) |
| **mocktail** | ^1.0.4 | Mocking library | [pub.dev/packages/mocktail](https://pub.dev/packages/mocktail) |
| **build_runner** | ^2.4.13 | Code generation runner | [pub.dev/packages/build_runner](https://pub.dev/packages/build_runner) |
| **flutter_lints** | ^5.0.0 | Lint rules | [pub.dev/packages/flutter_lints](https://pub.dev/packages/flutter_lints) |

---

## Models Reference

### Activity

**File**: [app/lib/domain/models/activity/activity.dart](../app/lib/domain/models/activity/activity.dart)

**Purpose**: Represents an activity available at a destination

**Properties**:

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | `String` | Yes | Activity name |
| `description` | `String` | Yes | Detailed description |
| `locationName` | `String` | Yes | Specific location |
| `duration` | `int` | Yes | Duration in hours |
| `timeOfDay` | `TimeOfDay` | Yes | When activity occurs |
| `familyFriendly` | `bool` | Yes | Suitable for families |
| `price` | `int` | Yes | Price in USD |
| `destinationRef` | `String` | Yes | Links to destination |
| `ref` | `String` | Yes | Unique identifier |
| `imageUrl` | `String` | Yes | Activity image |

**TimeOfDay Enum**:
```dart
enum TimeOfDay {
  any,
  morning,
  afternoon,
  evening,
  night;
}
```

**Usage**:
```dart
final activity = Activity(
  name: 'Eiffel Tower Tour',
  description: 'Visit the iconic Eiffel Tower',
  locationName: 'Champ de Mars',
  duration: 3,
  timeOfDay: TimeOfDay.any,
  familyFriendly: true,
  price: 25,
  destinationRef: 'paris-france',
  ref: 'eiffel-tower',
  imageUrl: 'https://...',
);
```

---

### Booking

**File**: [app/lib/domain/models/booking/booking.dart](../app/lib/domain/models/booking/booking.dart)

**Purpose**: Represents a complete travel booking

**Properties**:

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `id` | `int?` | No | Unique identifier (null for new) |
| `startDate` | `DateTime` | Yes | Trip start date |
| `endDate` | `DateTime` | Yes | Trip end date |
| `destination` | `Destination` | Yes | Where traveling |
| `activity` | `List<Activity>` | Yes | Selected activities |

**Usage**:
```dart
final booking = Booking(
  id: 1,
  startDate: DateTime(2025, 6, 1),
  endDate: DateTime(2025, 6, 10),
  destination: parisDestination,
  activity: [eiffelTower, louvreMuseum],
);

// Create modified copy
final extended = booking.copyWith(
  endDate: DateTime(2025, 6, 15),
);
```

---

### BookingSummary

**File**: [app/lib/domain/models/booking/booking_summary.dart](../app/lib/domain/models/booking/booking_summary.dart)

**Purpose**: Lightweight booking representation for lists

**Properties**:

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `id` | `int` | Yes | Unique identifier |
| `name` | `String` | Yes | Destination name |
| `startDate` | `DateTime` | Yes | Trip start date |
| `endDate` | `DateTime` | Yes | Trip end date |

---

### Continent

**File**: [app/lib/domain/models/continent/continent.dart](../app/lib/domain/models/continent/continent.dart)

**Purpose**: Represents a geographic continent

**Properties**:

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | `String` | Yes | Continent name |
| `imageUrl` | `String` | Yes | Representative image |

---

### Destination

**File**: [app/lib/domain/models/destination/destination.dart](../app/lib/domain/models/destination/destination.dart)

**Purpose**: Represents a travel destination

**Properties**:

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `ref` | `String` | Yes | Unique reference |
| `name` | `String` | Yes | Display name |
| `country` | `String` | Yes | Country name |
| `continent` | `String` | Yes | Continent name |
| `knownFor` | `String` | Yes | Famous for |
| `tags` | `List<String>` | Yes | Searchable tags |
| `imageUrl` | `String` | Yes | Hero image |

---

### ItineraryConfig

**File**: [app/lib/domain/models/itinerary_config/itinerary_config.dart](../app/lib/domain/models/itinerary_config/itinerary_config.dart)

**Purpose**: Temporary storage for booking flow selections

**Properties**:

| Property | Type | Required | Description | Step |
|----------|------|----------|-------------|------|
| `continent` | `String?` | No | Selected continent | Search Form |
| `startDate` | `DateTime?` | No | Trip start | Search Form |
| `endDate` | `DateTime?` | No | Trip end | Search Form |
| `guests` | `int?` | No | Number of guests | Search Form |
| `destination` | `String?` | No | Destination ref | Results |
| `activities` | `List<String>` | No | Activity refs | Activities |

---

### User

**File**: [app/lib/domain/models/user/user.dart](../app/lib/domain/models/user/user.dart)

**Purpose**: User profile information

**Properties**:

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | `String` | Yes | User's full name |
| `picture` | `String` | Yes | Profile picture URL |

---

## Repositories Reference

### ActivityRepository

**File**: [app/lib/data/repositories/activity/activity_repository.dart](../app/lib/data/repositories/activity/activity_repository.dart)

**Purpose**: Manage activities data

**Methods**:

```dart
Future<Result<List<Activity>>> getByDestination(String ref)
```
- **Description**: Get activities for a destination
- **Parameters**:
  - `ref` (String): Destination reference
- **Returns**: `Result<List<Activity>>`
- **Example**:
  ```dart
  final result = await repository.getByDestination('paris-france');
  switch (result) {
    case Ok(): print('Found ${result.value.length} activities');
    case Error(): print('Error: ${result.error}');
  }
  ```

**Implementations**:
- [ActivityRepositoryLocal](../app/lib/data/repositories/activity/activity_repository_local.dart): Loads from JSON asset
- [ActivityRepositoryRemote](../app/lib/data/repositories/activity/activity_repository_remote.dart): Fetches from API

---

### AuthRepository

**File**: [app/lib/data/repositories/auth/auth_repository.dart](../app/lib/data/repositories/auth/auth_repository.dart)

**Purpose**: Handle user authentication

**Methods**:

```dart
Future<bool> get isAuthenticated
```
- **Description**: Check if user is authenticated
- **Returns**: `Future<bool>`
- **Example**: `final loggedIn = await authRepository.isAuthenticated;`

```dart
Future<Result<void>> login({
  required String email,
  required String password,
})
```
- **Description**: Log in user
- **Parameters**:
  - `email` (String): User email
  - `password` (String): User password
- **Returns**: `Result<void>`
- **Example**:
  ```dart
  final result = await authRepository.login(
    email: 'user@example.com',
    password: 'password',
  );
  ```

```dart
Future<void> logout()
```
- **Description**: Log out current user
- **Returns**: `Future<void>`
- **Example**: `await authRepository.logout();`

**Implementations**:
- [AuthRepositoryDev](../app/lib/data/repositories/auth/auth_repository_dev.dart): Always authenticated
- [AuthRepositoryRemote](../app/lib/data/repositories/auth/auth_repository_remote.dart): JWT-based authentication

---

### BookingRepository

**File**: [app/lib/data/repositories/booking/booking_repository.dart](../app/lib/data/repositories/booking/booking_repository.dart)

**Purpose**: Manage bookings (CRUD operations)

**Methods**:

```dart
Future<Result<List<BookingSummary>>> getBookingsList()
```
- **Description**: Get list of user's bookings
- **Returns**: `Result<List<BookingSummary>>`

```dart
Future<Result<Booking>> getBooking(int id)
```
- **Description**: Get detailed booking by ID
- **Parameters**: `id` (int): Booking ID
- **Returns**: `Result<Booking>`

```dart
Future<Result<void>> createBooking(Booking booking)
```
- **Description**: Create a new booking
- **Parameters**: `booking` (Booking): Booking to create
- **Returns**: `Result<void>`

```dart
Future<Result<void>> delete(int id)
```
- **Description**: Delete a booking
- **Parameters**: `id` (int): Booking ID
- **Returns**: `Result<void>`

**Implementations**:
- [BookingRepositoryLocal](../app/lib/data/repositories/booking/booking_repository_local.dart): In-memory storage
- [BookingRepositoryRemote](../app/lib/data/repositories/booking/booking_repository_remote.dart): API-backed

---

### ContinentRepository

**File**: [app/lib/data/repositories/continent/continent_repository.dart](../app/lib/data/repositories/continent/continent_repository.dart)

**Purpose**: Fetch continents

**Methods**:

```dart
Future<Result<List<Continent>>> getContinents()
```
- **Description**: Get all continents
- **Returns**: `Result<List<Continent>>`

**Implementations**:
- [ContinentRepositoryLocal](../app/lib/data/repositories/continent/continent_repository_local.dart): Hardcoded list
- [ContinentRepositoryRemote](../app/lib/data/repositories/continent/continent_repository_remote.dart): API fetch

---

### DestinationRepository

**File**: [app/lib/data/repositories/destination/destination_repository.dart](../app/lib/data/repositories/destination/destination_repository.dart)

**Purpose**: Fetch destinations

**Methods**:

```dart
Future<Result<List<Destination>>> getDestinations()
```
- **Description**: Get all destinations
- **Returns**: `Result<List<Destination>>`

**Implementations**:
- [DestinationRepositoryLocal](../app/lib/data/repositories/destination/destination_repository_local.dart): JSON asset
- [DestinationRepositoryRemote](../app/lib/data/repositories/destination/destination_repository_remote.dart): API fetch

---

### ItineraryConfigRepository

**File**: [app/lib/data/repositories/itinerary_config/itinerary_config_repository.dart](../app/lib/data/repositories/itinerary_config/itinerary_config_repository.dart)

**Purpose**: Store temporary booking flow state

**Methods**:

```dart
Future<Result<ItineraryConfig>> getItineraryConfig()
```
- **Description**: Get current config
- **Returns**: `Result<ItineraryConfig>`

```dart
Future<Result<void>> setItineraryConfig(ItineraryConfig config)
```
- **Description**: Update config
- **Parameters**: `config` (ItineraryConfig): New config
- **Returns**: `Result<void>`

**Implementation**:
- [ItineraryConfigRepositoryMemory](../app/lib/data/repositories/itinerary_config/itinerary_config_repository_memory.dart): In-memory only

---

### UserRepository

**File**: [app/lib/data/repositories/user/user_repository.dart](../app/lib/data/repositories/user/user_repository.dart)

**Purpose**: Fetch user profile

**Methods**:

```dart
Future<Result<User>> getUser()
```
- **Description**: Get current user
- **Returns**: `Result<User>`

**Implementations**:
- [UserRepositoryLocal](../app/lib/data/repositories/user/user_repository_local.dart): Mock user
- [UserRepositoryRemote](../app/lib/data/repositories/user/user_repository_remote.dart): API fetch

---

## ViewModels Reference

### ActivitiesViewModel

**File**: [app/lib/ui/activities/view_models/activities_viewmodel.dart](../app/lib/ui/activities/view_models/activities_viewmodel.dart)

**Purpose**: Manage activity selection screen

**Commands**:
- `loadActivities` (Command0): Load activities for destination
- `saveActivities` (Command0): Save selections to config

**Properties**:
```dart
List<Activity> get activities         // All activities
List<Activity> get daytimeActivities  // Morning/afternoon
List<Activity> get eveningActivities  // Evening/night
List<String> get selectedActivities   // Selected refs
bool get valid                        // At least one selected
```

**Methods**:
```dart
void addActivity(String ref)          // Select activity
void removeActivity(String ref)       // Deselect activity
bool isSelected(String ref)           // Check if selected
```

---

### BookingViewModel

**File**: [app/lib/ui/booking/view_models/booking_viewmodel.dart](../app/lib/ui/booking/view_models/booking_viewmodel.dart)

**Purpose**: Manage booking confirmation/detail screen

**Commands**:
- `load` (Command0): Load existing booking (if ID provided)
- `createBooking` (Command0): Create booking from config
- `shareBooking` (Command0): Share booking via OS dialog

**Properties**:
```dart
Booking? get booking  // Current booking
```

---

### HomeViewModel

**File**: [app/lib/ui/home/view_models/home_viewmodel.dart](../app/lib/ui/home/view_models/home_viewmodel.dart)

**Purpose**: Manage home screen

**Commands**:
- `load` (Command0): Load bookings and user
- `deleteBooking` (Command1<void, int>): Delete a booking

**Properties**:
```dart
List<BookingSummary> get bookings  // User's bookings
User? get user                     // Current user
```

---

### LoginViewModel

**File**: [app/lib/ui/auth/view_models/login_viewmodel.dart](../app/lib/ui/auth/view_models/login_viewmodel.dart)

**Purpose**: Manage login screen

**Commands**:
- `login` (Command0): Perform login

**Properties**:
```dart
String get email       // Email input
String get password    // Password input
bool get isValid       // Both fields filled
```

**Setters**:
```dart
set email(String value)
set password(String value)
```

---

### ResultsViewModel

**File**: [app/lib/ui/results/view_models/results_viewmodel.dart](../app/lib/ui/results/view_models/results_viewmodel.dart)

**Purpose**: Manage destination results screen

**Commands**:
- `search` (Command0): Search destinations by continent
- `updateItineraryConfig` (Command0): Save selection

**Properties**:
```dart
List<Destination> get destinations        // Filtered destinations
String? get selectedDestination           // Selected ref
bool get valid                           // One selected
```

**Setters**:
```dart
set selectedDestination(String? value)
```

---

### SearchFormViewModel

**File**: [app/lib/ui/search_form/view_models/search_form_viewmodel.dart](../app/lib/ui/search_form/view_models/search_form_viewmodel.dart)

**Purpose**: Manage search form screen

**Commands**:
- `load` (Command0): Load continents
- `updateItineraryConfig` (Command0): Save search criteria

**Properties**:
```dart
List<Continent> get continents      // Available continents
String? get selectedContinent       // Selected continent
DateTime? get startDate             // Trip start
DateTime? get endDate               // Trip end
int get guests                      // Number of guests
bool get valid                      // All required fields set
```

**Setters**:
```dart
set selectedContinent(String? value)
set startDate(DateTime? value)
set endDate(DateTime? value)
set guests(int value)
```

---

## API Endpoints

### Base URL

**Development**: Not applicable (local data)
**Staging**: `http://localhost:8080`

---

### Authentication Endpoints

#### POST `/login`

**Purpose**: Authenticate user and get JWT token

**Request Body**:
```json
{
  "email": "user@example.com",
  "password": "password"
}
```

**Response** (200 OK):
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Errors**:
- 400: Invalid credentials
- 500: Server error

---

### Continent Endpoints

#### GET `/continent`

**Purpose**: Get all continents

**Headers**: `Authorization: Bearer <token>`

**Response** (200 OK):
```json
[
  {
    "name": "Europe",
    "imageUrl": "https://..."
  },
  ...
]
```

---

### Destination Endpoints

#### GET `/destination`

**Purpose**: Get all destinations

**Headers**: `Authorization: Bearer <token>`

**Response** (200 OK):
```json
[
  {
    "ref": "paris-france",
    "name": "Paris",
    "country": "France",
    "continent": "Europe",
    "knownFor": "Art, culture, and romance",
    "tags": ["Art", "Architecture"],
    "imageUrl": "https://..."
  },
  ...
]
```

---

### Activity Endpoints

#### GET `/destination/:ref/activity`

**Purpose**: Get activities for a destination

**Headers**: `Authorization: Bearer <token>`

**Parameters**:
- `ref` (path): Destination reference

**Response** (200 OK):
```json
[
  {
    "name": "Eiffel Tower Tour",
    "description": "Visit the iconic Eiffel Tower",
    "locationName": "Champ de Mars",
    "duration": 3,
    "timeOfDay": "any",
    "familyFriendly": true,
    "price": 25,
    "destinationRef": "paris-france",
    "ref": "eiffel-tower",
    "imageUrl": "https://..."
  },
  ...
]
```

---

### Booking Endpoints

#### GET `/booking`

**Purpose**: Get user's bookings

**Headers**: `Authorization: Bearer <token>`

**Response** (200 OK):
```json
[
  {
    "id": 1,
    "name": "Paris",
    "startDate": "2025-06-01T00:00:00.000Z",
    "endDate": "2025-06-10T00:00:00.000Z"
  },
  ...
]
```

---

#### GET `/booking/:id`

**Purpose**: Get detailed booking

**Headers**: `Authorization: Bearer <token>`

**Parameters**:
- `id` (path): Booking ID

**Response** (200 OK):
```json
{
  "id": 1,
  "startDate": "2025-06-01T00:00:00.000Z",
  "endDate": "2025-06-10T00:00:00.000Z",
  "destination": { ... },
  "activity": [ ... ]
}
```

**Errors**:
- 404: Booking not found

---

#### POST `/booking`

**Purpose**: Create a new booking

**Headers**: `Authorization: Bearer <token>`

**Request Body**:
```json
{
  "startDate": "2025-06-01T00:00:00.000Z",
  "endDate": "2025-06-10T00:00:00.000Z",
  "destination": { ... },
  "activity": [ ... ]
}
```

**Response** (200 OK):
```json
{
  "id": 123,
  "startDate": "2025-06-01T00:00:00.000Z",
  "endDate": "2025-06-10T00:00:00.000Z",
  "destination": { ... },
  "activity": [ ... ]
}
```

---

#### DELETE `/booking/:id`

**Purpose**: Delete a booking

**Headers**: `Authorization: Bearer <token>`

**Parameters**:
- `id` (path): Booking ID

**Response** (200 OK): Empty body

**Errors**:
- 404: Booking not found

---

### User Endpoints

#### GET `/user`

**Purpose**: Get current user profile

**Headers**: `Authorization: Bearer <token>`

**Response** (200 OK):
```json
{
  "name": "Sofie Bauer",
  "picture": "https://..."
}
```

---

## Routes Reference

### Route Paths

**File**: [app/lib/routing/routes.dart](../app/lib/routing/routes.dart)

| Route | Path | Screen | Description |
|-------|------|--------|-------------|
| `login` | `/login` | LoginScreen | User authentication |
| `home` | `/` | HomeScreen | Bookings list |
| `search` | `/search` | SearchFormScreen | Trip search form |
| `results` | `/results` | ResultsScreen | Destination selection |
| `activities` | `/activities` | ActivitiesScreen | Activity selection |
| `booking` | `/booking` | BookingScreen | New booking confirmation |
| `bookingWithId` | `/booking?id={id}` | BookingScreen | Existing booking detail |

---

### Navigation Examples

```dart
// Go to route (replace)
context.go(Routes.home);
context.go(Routes.search);

// Push route (add to stack)
context.push(Routes.bookingWithId(123));

// Go back
context.pop();

// Check current route
final currentRoute = GoRouter.of(context).location;
```

---

### Deep Links

Supported URL schemes:
```
compass://                → Home
compass://search          → Search form
compass://booking?id=123  → Specific booking
```

---

## Utility Classes

### Command

**File**: [app/lib/utils/command.dart](../app/lib/utils/command.dart)

**Purpose**: Encapsulate async actions with state management

**Command0** (no arguments):
```dart
class Command0<T> extends Command<T> {
  Command0(Future<Result<T>> Function() action);

  Future<void> execute();

  bool get running;      // Is executing?
  Exception? get error;  // Error if failed
  T? get result;         // Result if succeeded
  bool get completed;    // Succeeded?
}
```

**Command1** (one argument):
```dart
class Command1<T, A> extends Command<T> {
  Command1(Future<Result<T>> Function(A) action);

  Future<void> execute(A argument);

  // Same properties as Command0
}
```

**Usage**:
```dart
// Create command
late Command0 load;
load = Command0(_load)..execute(); // Auto-execute

// In View
ListenableBuilder(
  listenable: viewModel.load,
  builder: (context, _) {
    if (viewModel.load.running) return Spinner();
    if (viewModel.load.error != null) return ErrorWidget();
    return DataWidget();
  },
)
```

---

### Result

**File**: [app/lib/utils/result.dart](../app/lib/utils/result.dart)

**Purpose**: Type-safe error handling

**Classes**:
```dart
sealed class Result<T> {
  const factory Result.ok(T value) = Ok._;
  const factory Result.error(Exception error) = Error._;
}

class Ok<T> extends Result<T> {
  final T value;
}

class Error<T> extends Result<T> {
  final Exception error;
}
```

**Usage**:
```dart
// Return Result
Future<Result<Booking>> getBooking(int id) async {
  try {
    final booking = await api.fetch(id);
    return Result.ok(booking);
  } catch (e) {
    return Result.error(Exception(e));
  }
}

// Pattern match
final result = await repository.getBooking(id);
switch (result) {
  case Ok<Booking>(:final value):
    print('Got booking: ${value.destination.name}');
  case Error<Booking>(:final error):
    print('Error: $error');
}

// When method
final message = result.when(
  ok: (booking) => 'Loaded ${booking.destination.name}',
  error: (error) => 'Failed: $error',
);

// Null-safe access
final booking = result.valueOrNull;
```

---

## Testing Utilities

### Mocktail

**Package**: mocktail ^1.0.4

**Purpose**: Create mock objects for testing

**Usage**:
```dart
import 'package:mocktail/mocktail.dart';

// Define mock
class MockBookingRepository extends Mock implements BookingRepository {}

// In test
test('loads bookings', () async {
  final mockRepo = MockBookingRepository();

  // Stub method
  when(() => mockRepo.getBookingsList()).thenAnswer(
    (_) async => Result.ok([booking1, booking2]),
  );

  // Use mock
  final viewModel = HomeViewModel(bookingRepository: mockRepo);
  await viewModel.load.execute();

  // Verify
  expect(viewModel.bookings.length, 2);
  verify(() => mockRepo.getBookingsList()).called(1);
});
```

---

### Flutter Test

**Basic Test Structure**:
```dart
import 'package:flutter_test/flutter_test.dart';

void main() {
  group('MyClass', () {
    setUp(() {
      // Setup before each test
    });

    tearDown(() {
      // Cleanup after each test
    });

    test('description of test', () {
      // Arrange
      final instance = MyClass();

      // Act
      final result = instance.method();

      // Assert
      expect(result, expectedValue);
    });
  });
}
```

---

### Widget Testing

```dart
import 'package:flutter_test/flutter_test.dart';

void main() {
  testWidgets('shows loading indicator', (tester) async {
    // Build widget
    await tester.pumpWidget(
      MaterialApp(home: MyWidget()),
    );

    // Find widgets
    expect(find.text('Loading'), findsOneWidget);
    expect(find.byType(CircularProgressIndicator), findsOneWidget);

    // Tap button
    await tester.tap(find.byType(ElevatedButton));
    await tester.pump(); // Rebuild

    // Verify
    expect(find.text('Loaded'), findsOneWidget);
  });
}
```

---

## Code Generation Commands

### Freezed Code Generation

**One-time generation**:
```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

**Watch mode** (auto-regenerate on file changes):
```bash
flutter pub run build_runner watch --delete-conflicting-outputs
```

**Clean and regenerate**:
```bash
flutter pub run build_runner clean
flutter pub run build_runner build --delete-conflicting-outputs
```

---

### What Gets Generated

For a model like:
```dart
@freezed
class Booking with _$Booking {
  const factory Booking({...}) = _Booking;
  factory Booking.fromJson(Map<String, dynamic> json) => _$BookingFromJson(json);
}
```

**Generated files**:
- `booking.freezed.dart`: Immutable class, copyWith, equality
- `booking.g.dart`: JSON serialization

---

### When to Regenerate

Run code generation when:
- ✅ Adding a new @freezed model
- ✅ Adding/removing properties from a model
- ✅ Changing property types
- ✅ After pulling changes that modify models
- ❌ Changing UI code (not needed)
- ❌ Changing ViewModel logic (not needed)

---

## Quick Reference Tables

### Logging Levels

| Level | When to Use | Example |
|-------|-------------|---------|
| `FINE` | Detailed debug info | `_log.fine('Loading data')` |
| `INFO` | General information | `_log.info('Loaded 10 items')` |
| `WARNING` | Potential issues | `_log.warning('Failed to load', error)` |
| `SEVERE` | Serious errors | `_log.severe('Database corrupted')` |

---

### Environment Variables

| Environment | Log Level | Data Source | Auth Required |
|-------------|-----------|-------------|---------------|
| Development | WARNING | Local JSON | No |
| Staging | ALL | Remote API | Yes |

---

### File Naming Conventions

| Type | Convention | Example |
|------|-----------|---------|
| Screen | `{feature}_screen.dart` | `home_screen.dart` |
| ViewModel | `{feature}_viewmodel.dart` | `home_viewmodel.dart` |
| Repository | `{entity}_repository.dart` | `booking_repository.dart` |
| Model | `{entity}.dart` | `booking.dart` |
| Test | `{file}_test.dart` | `home_viewmodel_test.dart` |

---

## Summary

This reference guide covers:

✅ **Dependencies**: All packages with versions and links
✅ **Models**: Complete property reference for all domain models
✅ **Repositories**: All methods with signatures and examples
✅ **ViewModels**: Commands, properties, and usage
✅ **API Endpoints**: Complete HTTP API reference
✅ **Routes**: All navigation paths
✅ **Utilities**: Command and Result pattern reference
✅ **Testing**: Mocktail and Flutter test utilities
✅ **Code Generation**: When and how to regenerate code

**Bookmark this page** for quick lookups during development!

---

## Additional Resources

- [Architecture Overview](./01-architecture-overview.md)
- [Layer Breakdown](./02-layer-breakdown.md)
- [Design Patterns](./03-design-patterns.md)
- [Component Deep-Dive](./04-component-deep-dive.md)
- [Getting Started Guide](./05-getting-started.md)

**Official Documentation**:
- [Flutter Docs](https://flutter.dev/docs)
- [Dart Language Tour](https://dart.dev/guides/language/language-tour)
- [Provider Package](https://pub.dev/packages/provider)
- [go_router Package](https://pub.dev/packages/go_router)
- [Freezed Package](https://pub.dev/packages/freezed)
