# 02 - Layer Breakdown

## 📋 Table of Contents

- [Introduction](#introduction)
- [Domain Layer](#domain-layer)
- [Data Layer](#data-layer)
- [Presentation Layer](#presentation-layer)
- [Utils Layer](#utils-layer)
- [Routing Layer](#routing-layer)
- [Configuration Layer](#configuration-layer)
- [Layer Communication](#layer-communication)

---

## Introduction

This document provides a **detailed breakdown of each architectural layer** in the Compass App. Each layer has specific responsibilities and follows established patterns to maintain clean separation of concerns.

**What you'll learn**:
- What each layer contains
- The responsibilities of each layer
- How components within each layer work
- Concrete code examples with file references
- Best practices for each layer

---

## Domain Layer

**Location**: [app/lib/domain/](../app/lib/domain/)

**Purpose**: Contains the core business logic and entities that are independent of any framework or external concern.

**Analogy**: Think of the domain layer as the **recipe book** for your restaurant. It defines what a "burger" is (ingredients, structure) and how to prepare it, but doesn't care whether you're cooking on a gas stove or electric grill (those are implementation details in other layers).

---

### Domain Models

**Location**: [app/lib/domain/models/](../app/lib/domain/models/)

**Purpose**: Define the core entities/objects that the application works with.

**Characteristics**:
- **Immutable**: Properties cannot change after creation
- **Business-focused**: Represent real-world concepts (Booking, Destination)
- **Framework-independent**: No Flutter or HTTP dependencies
- **Self-contained**: Include validation and business rules

---

#### 1. Booking Model

**File**: [app/lib/domain/models/booking/booking.dart](../app/lib/domain/models/booking/booking.dart)

**Purpose**: Represents a complete travel booking with all details.

**Structure**:
```dart
@freezed
class Booking with _$Booking {
  const factory Booking({
    int? id,
    required DateTime startDate,
    required DateTime endDate,
    required Destination destination,
    required List<Activity> activity,
  }) = _Booking;

  factory Booking.fromJson(Map<String, dynamic> json) =>
      _$BookingFromJson(json);
}
```

**Properties**:
| Property | Type | Description |
|----------|------|-------------|
| `id` | `int?` | Unique identifier (null for new bookings) |
| `startDate` | `DateTime` | Trip start date |
| `endDate` | `DateTime` | Trip end date |
| `destination` | `Destination` | Where the user is traveling |
| `activity` | `List<Activity>` | Selected activities for the trip |

**Key Features**:
- Uses `@freezed` annotation for code generation
- Immutable (properties are `required` and `final`)
- JSON serialization built-in
- `copyWith` method automatically generated

**Usage Example**:
```dart
// Create a new booking
final booking = Booking(
  startDate: DateTime(2025, 6, 1),
  endDate: DateTime(2025, 6, 10),
  destination: parisDestination,
  activity: [eiffelTower, louvreMuseum],
);

// Create a modified copy
final extendedBooking = booking.copyWith(
  endDate: DateTime(2025, 6, 15),
);

// Equality check (automatically generated)
print(booking == extendedBooking); // false
```

**Why immutable?**
Immutability prevents bugs from accidental modifications. If you need to change a booking, you create a new one with `copyWith`, making state changes explicit and traceable.

---

#### 2. BookingSummary Model

**File**: [app/lib/domain/models/booking/booking_summary.dart](../app/lib/domain/models/booking/booking_summary.dart)

**Purpose**: Lightweight representation of a booking for list displays.

**Structure**:
```dart
@freezed
class BookingSummary with _$BookingSummary {
  const factory BookingSummary({
    required int id,
    required String name,
    required DateTime startDate,
    required DateTime endDate,
  }) = _BookingSummary;

  factory BookingSummary.fromJson(Map<String, dynamic> json) =>
      _$BookingSummaryFromJson(json);
}
```

**Why separate from Booking?**
- **Performance**: Don't need full destination and activity data for lists
- **Network Efficiency**: API returns less data for list endpoints
- **Separation**: List view needs different data than detail view

**Usage**: The home screen displays a list of BookingSummary objects. When user taps one, the full Booking is fetched.

---

#### 3. Destination Model

**File**: [app/lib/domain/models/destination/destination.dart](../app/lib/domain/models/destination/destination.dart)

**Purpose**: Represents a travel destination.

**Structure**:
```dart
@freezed
class Destination with _$Destination {
  const factory Destination({
    required String ref,
    required String name,
    required String country,
    required String continent,
    required String knownFor,
    required List<String> tags,
    required String imageUrl,
  }) = _Destination;

  factory Destination.fromJson(Map<String, dynamic> json) =>
      _$DestinationFromJson(json);
}
```

**Properties**:
| Property | Type | Description | Example |
|----------|------|-------------|---------|
| `ref` | `String` | Unique reference identifier | `"paris-france"` |
| `name` | `String` | Display name | `"Paris"` |
| `country` | `String` | Country name | `"France"` |
| `continent` | `String` | Continent name | `"Europe"` |
| `knownFor` | `String` | Famous for | `"Art, culture, and romance"` |
| `tags` | `List<String>` | Searchable tags | `["Art", "Architecture"]` |
| `imageUrl` | `String` | Hero image URL | `"https://..."` |

**Key Feature**: The `ref` field is used to link destinations with activities. For example, all activities in Paris have `destinationRef: "paris-france"`.

**Usage Example**:
```dart
// Search destinations by continent
final europeDestinations = allDestinations.where(
  (dest) => dest.continent == 'Europe',
).toList();

// Display destination
Text(destination.name);              // "Paris"
Text(destination.knownFor);          // "Art, culture, and romance"
CachedNetworkImage(destination.imageUrl);
```

---

#### 4. Activity Model

**File**: [app/lib/domain/models/activity/activity.dart](../app/lib/domain/models/activity/activity.dart)

**Purpose**: Represents an activity available at a destination.

**Structure**:
```dart
@freezed
class Activity with _$Activity {
  const factory Activity({
    required String name,
    required String description,
    required String locationName,
    required int duration,
    required TimeOfDay timeOfDay,
    required bool familyFriendly,
    required int price,
    required String destinationRef,
    required String ref,
    required String imageUrl,
  }) = _Activity;

  factory Activity.fromJson(Map<String, dynamic> json) =>
      _$ActivityFromJson(json);
}
```

**Properties**:
| Property | Type | Description | Example |
|----------|------|-------------|---------|
| `name` | `String` | Activity name | `"Eiffel Tower Tour"` |
| `description` | `String` | Detailed description | `"Visit the iconic..."` |
| `locationName` | `String` | Specific location | `"Champ de Mars"` |
| `duration` | `int` | Duration in hours | `3` |
| `timeOfDay` | `TimeOfDay` | When to do it | `TimeOfDay.any` |
| `familyFriendly` | `bool` | Suitable for families | `true` |
| `price` | `int` | Price in dollars | `25` |
| `destinationRef` | `String` | Links to destination | `"paris-france"` |
| `ref` | `String` | Unique identifier | `"eiffel-tower"` |
| `imageUrl` | `String` | Activity image | `"https://..."` |

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

**Usage Example**:
```dart
// Filter activities by time of day
final morningActivities = allActivities.where(
  (a) => a.timeOfDay == TimeOfDay.morning || a.timeOfDay == TimeOfDay.any,
).toList();

// Filter family-friendly activities
final familyActivities = allActivities.where(
  (a) => a.familyFriendly,
).toList();

// Calculate total price
final totalPrice = selectedActivities.fold(
  0,
  (sum, activity) => sum + activity.price,
);
```

---

#### 5. Continent Model

**File**: [app/lib/domain/models/continent/continent.dart](../app/lib/domain/models/continent/continent.dart)

**Purpose**: Represents a geographic continent.

**Structure**:
```dart
@freezed
class Continent with _$Continent {
  const factory Continent({
    required String name,
    required String imageUrl,
  }) = _Continent;

  factory Continent.fromJson(Map<String, dynamic> json) =>
      _$ContinentFromJson(json);
}
```

**Properties**:
| Property | Type | Description | Example |
|----------|------|-------------|---------|
| `name` | `String` | Continent name | `"Europe"` |
| `imageUrl` | `String` | Representative image | `"https://..."` |

**Usage**: Users select a continent in the search form, which filters available destinations.

**Example**:
```dart
// Display continent selector
GridView.builder(
  itemCount: continents.length,
  itemBuilder: (context, index) {
    final continent = continents[index];
    return ContinentCard(
      name: continent.name,
      imageUrl: continent.imageUrl,
      onTap: () => selectContinent(continent.name),
    );
  },
);
```

---

#### 6. User Model

**File**: [app/lib/domain/models/user/user.dart](../app/lib/domain/models/user/user.dart)

**Purpose**: Represents a user profile.

**Structure**:
```dart
@freezed
class User with _$User {
  const factory User({
    required String name,
    required String picture,
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) =>
      _$UserFromJson(json);
}
```

**Properties**:
| Property | Type | Description | Example |
|----------|------|-------------|---------|
| `name` | `String` | User's full name | `"Sofie Bauer"` |
| `picture` | `String` | Profile picture URL | `"https://..."` |

**Usage**: Displayed in the app bar and profile sections.

---

#### 7. ItineraryConfig Model

**File**: [app/lib/domain/models/itinerary_config/itinerary_config.dart](../app/lib/domain/models/itinerary_config/itinerary_config.dart)

**Purpose**: Stores temporary user selections during the booking flow.

**Structure**:
```dart
@freezed
class ItineraryConfig with _$ItineraryConfig {
  const factory ItineraryConfig({
    String? continent,
    DateTime? startDate,
    DateTime? endDate,
    int? guests,
    String? destination,
    @Default([]) List<String> activities,
  }) = _ItineraryConfig;

  factory ItineraryConfig.fromJson(Map<String, dynamic> json) =>
      _$ItineraryConfigFromJson(json);
}
```

**Properties**:
| Property | Type | Description | Step Collected |
|----------|------|-------------|----------------|
| `continent` | `String?` | Selected continent | Search Form |
| `startDate` | `DateTime?` | Trip start | Search Form |
| `endDate` | `DateTime?` | Trip end | Search Form |
| `guests` | `int?` | Number of guests | Search Form |
| `destination` | `String?` | Destination ref | Results Screen |
| `activities` | `List<String>` | Activity refs | Activities Screen |

**Why this exists**: The booking flow spans multiple screens. ItineraryConfig accumulates selections across screens until the user confirms and creates a Booking.

**Flow**:
```
Step 1: SearchFormScreen
  ↓ Save continent, dates, guests to ItineraryConfig

Step 2: ResultsScreen
  ↓ Save destination to ItineraryConfig

Step 3: ActivitiesScreen
  ↓ Save activities to ItineraryConfig

Step 4: BookingScreen
  ↓ BookingCreateUseCase converts ItineraryConfig → Booking
  ↓ Save Booking to BookingRepository
```

**Usage Example**:
```dart
// In SearchFormViewModel
Future<void> saveSearch() async {
  final config = ItineraryConfig(
    continent: selectedContinent,
    startDate: startDate,
    endDate: endDate,
    guests: numberOfGuests,
  );
  await itineraryConfigRepository.setItineraryConfig(config);
}

// In ResultsViewModel
Future<void> selectDestination(String destRef) async {
  final currentConfig = await itineraryConfigRepository.getItineraryConfig();
  final updatedConfig = currentConfig.copyWith(
    destination: destRef,
  );
  await itineraryConfigRepository.setItineraryConfig(updatedConfig);
}

// In BookingViewModel
Future<void> createBooking() async {
  final config = await itineraryConfigRepository.getItineraryConfig();
  final result = await bookingCreateUseCase.createFrom(config);
  // Handle result
}
```

---

### Domain Use Cases

**Location**: [app/lib/domain/use_cases/](../app/lib/domain/use_cases/)

**Purpose**: Encapsulate complex business operations that involve multiple repositories or complex logic.

**When to create a use case**:
- ✅ Operation involves multiple repositories
- ✅ Complex business logic that doesn't belong in a ViewModel
- ✅ Reusable operation across multiple ViewModels
- ❌ Simple CRUD operations (use repository directly)
- ❌ UI-specific logic (belongs in ViewModel)

---

#### 1. BookingCreateUseCase

**File**: [app/lib/domain/use_cases/booking/booking_create_use_case.dart](../app/lib/domain/use_cases/booking/booking_create_use_case.dart)

**Purpose**: Creates a complete Booking entity from an ItineraryConfig.

**Responsibilities**:
1. Validate that required fields are set (dates, destination, activities)
2. Fetch full Destination object from destination reference
3. Fetch full Activity objects from activity references
4. Create Booking entity
5. Persist booking to repository

**Why a use case?**
This operation requires:
- 3 different repositories (Destination, Activity, Booking)
- Complex validation logic
- Data transformation (refs → full objects)
- Multiple async operations in sequence

Putting this in the ViewModel would make it too complex. The use case isolates this complexity.

**Code Structure**:
```dart
class BookingCreateUseCase {
  final DestinationRepository destinationRepository;
  final ActivityRepository activityRepository;
  final BookingRepository bookingRepository;

  BookingCreateUseCase({
    required this.destinationRepository,
    required this.activityRepository,
    required this.bookingRepository,
  });

  Future<Result<Booking>> createFrom(ItineraryConfig config) async {
    // 1. Validate
    if (config.destination == null) {
      return Result.error(Exception('Destination not set'));
    }
    if (config.startDate == null || config.endDate == null) {
      return Result.error(Exception('Dates not set'));
    }

    // 2. Fetch full destination
    final destinationResult = await _fetchDestination(config.destination!);
    if (destinationResult is Error) {
      return destinationResult as Result<Booking>;
    }

    // 3. Fetch activities
    final activitiesResult = await activityRepository.getByDestination(
      config.destination!,
    );
    if (activitiesResult is Error) {
      return activitiesResult as Result<Booking>;
    }

    // Filter to selected activities
    final selectedActivities = activitiesResult.asOk.value
        .where((activity) => config.activities.contains(activity.ref))
        .toList();

    // 4. Create booking
    final booking = Booking(
      startDate: config.startDate!,
      endDate: config.endDate!,
      destination: destinationResult.asOk.value,
      activity: selectedActivities,
    );

    // 5. Persist
    final createResult = await bookingRepository.createBooking(booking);
    return createResult.when(
      ok: (_) => Result.ok(booking),
      error: (error) => Result.error(error),
    );
  }

  Future<Result<Destination>> _fetchDestination(String ref) async {
    final result = await destinationRepository.getDestinations();
    return result.when(
      ok: (destinations) {
        final destination = destinations.firstWhere(
          (d) => d.ref == ref,
        );
        return Result.ok(destination);
      },
      error: (error) => Result.error(error),
    );
  }
}
```

**Usage in ViewModel**:
```dart
class BookingViewModel extends ChangeNotifier {
  final BookingCreateUseCase bookingCreateUseCase;
  final ItineraryConfigRepository itineraryConfigRepository;

  late Command0 createBooking;

  BookingViewModel({
    required this.bookingCreateUseCase,
    required this.itineraryConfigRepository,
  }) {
    createBooking = Command0(_createBooking);
  }

  Future<Result<void>> _createBooking() async {
    // Get current config
    final configResult = await itineraryConfigRepository.getItineraryConfig();
    if (configResult is Error) return configResult as Result<void>;

    // Use case handles all complexity!
    final result = await bookingCreateUseCase.createFrom(
      configResult.asOk.value,
    );

    return result.when(
      ok: (booking) {
        _booking = booking;
        notifyListeners();
        return Result.ok(null);
      },
      error: (error) => Result.error(error),
    );
  }
}
```

**Benefits**:
- ViewModel stays simple and focused on UI state
- Business logic is reusable
- Easy to test use case independently
- Clear separation of concerns

---

#### 2. BookingShareUseCase

**File**: [app/lib/domain/use_cases/booking/booking_share_use_case.dart](../app/lib/domain/use_cases/booking/booking_share_use_case.dart)

**Purpose**: Share a booking via the OS share dialog.

**Responsibilities**:
1. Format booking data into shareable text
2. Invoke OS share functionality

**Code Structure**:
```dart
typedef ShareFunction = Future<void> Function(String text);

class BookingShareUseCase {
  final ShareFunction _share;

  // Factory for production (uses share_plus package)
  factory BookingShareUseCase.withSharePlus() =>
      BookingShareUseCase._(Share.share);

  // Custom share function (for testing)
  factory BookingShareUseCase.custom(ShareFunction share) =>
      BookingShareUseCase._(share);

  BookingShareUseCase._(this._share);

  Future<Result<void>> shareBooking(Booking booking) async {
    try {
      final text = _formatBooking(booking);
      await _share(text);
      return Result.ok(null);
    } on Exception catch (e) {
      return Result.error(e);
    }
  }

  String _formatBooking(Booking booking) {
    final buffer = StringBuffer();
    buffer.writeln('Check out my trip!');
    buffer.writeln();
    buffer.writeln('Destination: ${booking.destination.name}');
    buffer.writeln('Dates: ${_formatDate(booking.startDate)} - ${_formatDate(booking.endDate)}');
    buffer.writeln();
    buffer.writeln('Activities:');
    for (final activity in booking.activity) {
      buffer.writeln('  • ${activity.name}');
    }
    return buffer.toString();
  }

  String _formatDate(DateTime date) {
    return '${date.month}/${date.day}/${date.year}';
  }
}
```

**Why a use case?**
- Encapsulates the share functionality (makes it mockable for tests)
- Formatting logic is reusable
- Clear abstraction over platform-specific code

**Usage in ViewModel**:
```dart
class BookingViewModel extends ChangeNotifier {
  final BookingShareUseCase bookingShareUseCase;

  late Command0 shareBooking;

  Future<Result<void>> _shareBooking() async {
    if (_booking == null) return Result.error(Exception('No booking'));
    return await bookingShareUseCase.shareBooking(_booking!);
  }
}
```

**Testing**:
```dart
test('BookingShareUseCase formats booking correctly', () async {
  String? sharedText;

  // Inject mock share function
  final useCase = BookingShareUseCase.custom((text) async {
    sharedText = text;
  });

  await useCase.shareBooking(testBooking);

  expect(sharedText, contains('Paris'));
  expect(sharedText, contains('Eiffel Tower'));
});
```

---

## Data Layer

**Location**: [app/lib/data/](../app/lib/data/)

**Purpose**: Handle all data access, persistence, and external communication.

**Analogy**: The data layer is like a **warehouse manager**. It knows where to get products (data) - whether from the local warehouse (cache/storage) or ordering from suppliers (API). The domain layer just asks for products without caring about the source.

---

### Repository Pattern

**Core Concept**: All repositories follow the same structure:
1. **Abstract interface** defining contract (in domain-focused location)
2. **Local implementation** for development/offline mode
3. **Remote implementation** for production/API mode

**Benefits**:
- Swap implementations without changing business logic
- Easy to mock for testing
- Consistent API across all data sources

---

### Repository Structure

Each repository follows this pattern:

```
data/repositories/[entity]/
├── [entity]_repository.dart          # Abstract interface
├── [entity]_repository_local.dart    # Local implementation
└── [entity]_repository_remote.dart   # Remote implementation
```

---

#### 1. AuthRepository

**Files**:
- Interface: [data/repositories/auth/auth_repository.dart](../app/lib/data/repositories/auth/auth_repository.dart)
- Dev: [data/repositories/auth/auth_repository_dev.dart](../app/lib/data/repositories/auth/auth_repository_dev.dart)
- Remote: [data/repositories/auth/auth_repository_remote.dart](../app/lib/data/repositories/auth/auth_repository_remote.dart)

**Purpose**: Handle user authentication.

**Interface**:
```dart
abstract class AuthRepository extends ChangeNotifier {
  Future<bool> get isAuthenticated;
  Future<Result<void>> login({
    required String email,
    required String password,
  });
  Future<void> logout();
}
```

**Why extend ChangeNotifier?**
Authentication state changes should trigger UI updates (e.g., redirect to login when logged out). ChangeNotifier makes this reactive.

---

**Development Implementation** (`AuthRepositoryDev`):

```dart
class AuthRepositoryDev extends ChangeNotifier implements AuthRepository {
  @override
  Future<bool> get isAuthenticated async => true; // Always authenticated!

  @override
  Future<Result<void>> login({
    required String email,
    required String password,
  }) async {
    return Result.ok(null); // Always succeeds
  }

  @override
  Future<void> logout() async {
    // No-op
  }
}
```

**Use Case**: Development mode where authentication would slow down testing.

---

**Remote Implementation** (`AuthRepositoryRemote`):

```dart
class AuthRepositoryRemote extends ChangeNotifier implements AuthRepository {
  final AuthApiClient _authApiClient;
  final SharedPreferencesService _sharedPreferencesService;

  AuthRepositoryRemote({
    required AuthApiClient authApiClient,
    required SharedPreferencesService sharedPreferencesService,
  })  : _authApiClient = authApiClient,
        _sharedPreferencesService = sharedPreferencesService;

  @override
  Future<bool> get isAuthenticated async {
    final token = await _sharedPreferencesService.fetchToken();
    return token != null;
  }

  @override
  Future<Result<void>> login({
    required String email,
    required String password,
  }) async {
    final result = await _authApiClient.login(
      email: email,
      password: password,
    );

    return result.when(
      ok: (response) async {
        // Save token
        await _sharedPreferencesService.saveToken(response.token);
        notifyListeners(); // Trigger UI update
        return Result.ok(null);
      },
      error: (error) => Result.error(error),
    );
  }

  @override
  Future<void> logout() async {
    await _sharedPreferencesService.saveToken(null);
    notifyListeners(); // Trigger redirect to login
  }
}
```

**Key Features**:
- Stores JWT token in SharedPreferences
- Checks token presence for authentication status
- Notifies listeners on auth state changes

---

#### 2. BookingRepository

**Files**:
- Interface: [data/repositories/booking/booking_repository.dart](../app/lib/data/repositories/booking/booking_repository.dart)
- Local: [data/repositories/booking/booking_repository_local.dart](../app/lib/data/repositories/booking/booking_repository_local.dart)
- Remote: [data/repositories/booking/booking_repository_remote.dart](../app/lib/data/repositories/booking/booking_repository_remote.dart)

**Purpose**: Manage user bookings (CRUD operations).

**Interface**:
```dart
abstract class BookingRepository {
  Future<Result<List<BookingSummary>>> getBookingsList();
  Future<Result<Booking>> getBooking(int id);
  Future<Result<void>> createBooking(Booking booking);
  Future<Result<void>> delete(int id);
}
```

---

**Local Implementation** (`BookingRepositoryLocal`):

```dart
class BookingRepositoryLocal implements BookingRepository {
  final LocalDataService _localDataService;

  // In-memory storage
  final List<Booking> _bookings = [];
  int _sequenceId = 0;

  BookingRepositoryLocal({
    required LocalDataService localDataService,
  }) : _localDataService = localDataService {
    // Create a default booking on initialization
    _createDefaultBooking();
  }

  @override
  Future<Result<List<BookingSummary>>> getBookingsList() async {
    final summaries = _bookings.map((booking) {
      return BookingSummary(
        id: booking.id!,
        name: booking.destination.name,
        startDate: booking.startDate,
        endDate: booking.endDate,
      );
    }).toList();
    return Result.ok(summaries);
  }

  @override
  Future<Result<Booking>> getBooking(int id) async {
    try {
      final booking = _bookings.firstWhere((b) => b.id == id);
      return Result.ok(booking);
    } catch (e) {
      return Result.error(Exception('Booking not found'));
    }
  }

  @override
  Future<Result<void>> createBooking(Booking booking) async {
    final bookingWithId = booking.copyWith(id: _sequenceId++);
    _bookings.add(bookingWithId);
    return Result.ok(null);
  }

  @override
  Future<Result<void>> delete(int id) async {
    _bookings.removeWhere((b) => b.id == id);
    return Result.ok(null);
  }

  Future<void> _createDefaultBooking() async {
    // Load destinations and activities
    final destinationsResult = await _localDataService.getDestinations();
    final destination = destinationsResult.asOk.value.first;

    final activitiesResult = await _localDataService.getActivities();
    final activities = activitiesResult.asOk.value
        .where((a) => a.destinationRef == destination.ref)
        .take(3)
        .toList();

    // Create default booking
    final booking = Booking(
      startDate: DateTime.now().add(Duration(days: 30)),
      endDate: DateTime.now().add(Duration(days: 37)),
      destination: destination,
      activity: activities,
    );

    await createBooking(booking);
  }
}
```

**Key Features**:
- In-memory storage (data lost on app restart)
- Auto-incrementing IDs
- Creates a default booking for testing
- Synchronous operations (wrapped in async for interface consistency)

---

**Remote Implementation** (`BookingRepositoryRemote`):

```dart
class BookingRepositoryRemote implements BookingRepository {
  final ApiClient _apiClient;

  BookingRepositoryRemote({
    required ApiClient apiClient,
  }) : _apiClient = apiClient;

  @override
  Future<Result<List<BookingSummary>>> getBookingsList() async {
    return await _apiClient.getBookings();
  }

  @override
  Future<Result<Booking>> getBooking(int id) async {
    return await _apiClient.getBooking(id);
  }

  @override
  Future<Result<void>> createBooking(Booking booking) async {
    return await _apiClient.postBooking(booking);
  }

  @override
  Future<Result<void>> delete(int id) async {
    return await _apiClient.deleteBooking(id);
  }
}
```

**Key Features**:
- Delegates to ApiClient (separation of concerns)
- Real persistence via HTTP API
- Data survives app restarts

**Comparison**:

| Feature | Local | Remote |
|---------|-------|--------|
| **Storage** | In-memory | Server database |
| **Persistence** | Lost on restart | Permanent |
| **Network** | None | Required |
| **Speed** | Instant | Network latency |
| **Use Case** | Development | Production |

---

#### 3. DestinationRepository

**Files**:
- Interface: [data/repositories/destination/destination_repository.dart](../app/lib/data/repositories/destination/destination_repository.dart)
- Local: [data/repositories/destination/destination_repository_local.dart](../app/lib/data/repositories/destination/destination_repository_local.dart)
- Remote: [data/repositories/destination/destination_repository_remote.dart](../app/lib/data/repositories/destination/destination_repository_remote.dart)

**Purpose**: Fetch available travel destinations.

**Interface**:
```dart
abstract class DestinationRepository {
  Future<Result<List<Destination>>> getDestinations();
}
```

**Local Implementation**:
```dart
class DestinationRepositoryLocal implements DestinationRepository {
  final LocalDataService _localDataService;

  DestinationRepositoryLocal({
    required LocalDataService localDataService,
  }) : _localDataService = localDataService;

  @override
  Future<Result<List<Destination>>> getDestinations() {
    return _localDataService.getDestinations();
  }
}
```

Simple delegation to LocalDataService, which loads from JSON asset.

**Remote Implementation**:
```dart
class DestinationRepositoryRemote implements DestinationRepository {
  final ApiClient _apiClient;

  DestinationRepositoryRemote({
    required ApiClient apiClient,
  }) : _apiClient = apiClient;

  @override
  Future<Result<List<Destination>>> getDestinations() {
    return _apiClient.getDestinations();
  }
}
```

Simple delegation to ApiClient.

---

#### 4. ActivityRepository

**Files**:
- Interface: [data/repositories/activity/activity_repository.dart](../app/lib/data/repositories/activity/activity_repository.dart)
- Local: [data/repositories/activity/activity_repository_local.dart](../app/lib/data/repositories/activity/activity_repository_local.dart)
- Remote: [data/repositories/activity/activity_repository_remote.dart](../app/lib/data/repositories/activity/activity_repository_remote.dart)

**Purpose**: Fetch activities available at a destination.

**Interface**:
```dart
abstract class ActivityRepository {
  Future<Result<List<Activity>>> getByDestination(String ref);
}
```

**Local Implementation**:
```dart
class ActivityRepositoryLocal implements ActivityRepository {
  final LocalDataService _localDataService;

  ActivityRepositoryLocal({
    required LocalDataService localDataService,
  }) : _localDataService = localDataService;

  @override
  Future<Result<List<Activity>>> getByDestination(String ref) async {
    final result = await _localDataService.getActivities();
    return result.when(
      ok: (activities) {
        final filtered = activities.where(
          (activity) => activity.destinationRef == ref,
        ).toList();
        return Result.ok(filtered);
      },
      error: (error) => Result.error(error),
    );
  }
}
```

**Key Feature**: Filters all activities by `destinationRef` in-memory.

**Remote Implementation**:
```dart
class ActivityRepositoryRemote implements ActivityRepository {
  final ApiClient _apiClient;

  ActivityRepositoryRemote({
    required ApiClient apiClient,
  }) : _apiClient = apiClient;

  @override
  Future<Result<List<Activity>>> getByDestination(String ref) {
    return _apiClient.getActivitiesByDestination(ref);
  }
}
```

**Key Difference**: Remote version uses a specific API endpoint that filters on the server side (more efficient for large datasets).

---

#### 5. ContinentRepository

**Files**:
- Interface: [data/repositories/continent/continent_repository.dart](../app/lib/data/repositories/continent/continent_repository.dart)
- Local: [data/repositories/continent/continent_repository_local.dart](../app/lib/data/repositories/continent/continent_repository_local.dart)
- Remote: [data/repositories/continent/continent_repository_remote.dart](../app/lib/data/repositories/continent/continent_repository_remote.dart)

**Purpose**: Fetch available continents.

**Interface**:
```dart
abstract class ContinentRepository {
  Future<Result<List<Continent>>> getContinents();
}
```

**Local Implementation**:
```dart
class ContinentRepositoryLocal implements ContinentRepository {
  final LocalDataService _localDataService;

  ContinentRepositoryLocal({
    required LocalDataService localDataService,
  }) : _localDataService = localDataService;

  @override
  Future<Result<List<Continent>>> getContinents() {
    return _localDataService.getContinents();
  }
}
```

Delegates to LocalDataService, which returns a hardcoded list.

---

#### 6. UserRepository

**Files**:
- Interface: [data/repositories/user/user_repository.dart](../app/lib/data/repositories/user/user_repository.dart)
- Local: [data/repositories/user/user_repository_local.dart](../app/lib/data/repositories/user/user_repository_local.dart)
- Remote: [data/repositories/user/user_repository_remote.dart](../app/lib/data/repositories/user/user_repository_remote.dart)

**Purpose**: Fetch current user profile.

**Interface**:
```dart
abstract class UserRepository {
  Future<Result<User>> getUser();
}
```

**Local Implementation**:
Returns a hardcoded user via LocalDataService.

**Remote Implementation**:
Fetches user from API based on authenticated session.

---

#### 7. ItineraryConfigRepository

**Files**:
- Interface: [data/repositories/itinerary_config/itinerary_config_repository.dart](../app/lib/data/repositories/itinerary_config/itinerary_config_repository.dart)
- Memory: [data/repositories/itinerary_config/itinerary_config_repository_memory.dart](../app/lib/data/repositories/itinerary_config/itinerary_config_repository_memory.dart)

**Purpose**: Store temporary booking configuration during the multi-step booking flow.

**Interface**:
```dart
abstract class ItineraryConfigRepository {
  Future<Result<ItineraryConfig>> getItineraryConfig();
  Future<Result<void>> setItineraryConfig(ItineraryConfig config);
}
```

**Memory Implementation**:
```dart
class ItineraryConfigRepositoryMemory implements ItineraryConfigRepository {
  ItineraryConfig _config = const ItineraryConfig();

  @override
  Future<Result<ItineraryConfig>> getItineraryConfig() async {
    return Result.ok(_config);
  }

  @override
  Future<Result<void>> setItineraryConfig(ItineraryConfig config) async {
    _config = config;
    return Result.ok(null);
  }
}
```

**Key Characteristics**:
- **Only one implementation**: No local/remote split
- **In-memory only**: Data lost when app closes (by design)
- **Temporary storage**: Only used during booking creation flow

**Why in-memory?**
ItineraryConfig is temporary user input. Once a Booking is created, the config is no longer needed. Persisting it would be unnecessary complexity.

---

### Data Services

**Location**: [app/lib/data/services/](../app/lib/data/services/)

**Purpose**: Actual data sources that repositories use.

---

#### 1. LocalDataService

**File**: [data/services/local/local_data_service.dart](../app/lib/data/services/local/local_data_service.dart)

**Purpose**: Load data from local JSON assets and provide hardcoded data.

**Methods**:
```dart
class LocalDataService {
  Future<Result<List<Destination>>> getDestinations();
  Future<Result<List<Activity>>> getActivities();
  Future<Result<List<Continent>>> getContinents();
  Future<Result<User>> getUser();
}
```

**Implementation Details**:

**Loading JSON**:
```dart
Future<Result<List<Destination>>> getDestinations() async {
  try {
    // Load JSON file from assets
    final data = await rootBundle.loadString(Assets.destinationsData);

    // Parse JSON
    final List<dynamic> json = jsonDecode(data);

    // Convert to Destination objects
    final destinations = json.map((item) =>
        Destination.fromJson(item as Map<String, dynamic>)
    ).toList();

    return Result.ok(destinations);
  } on Exception catch (e) {
    return Result.error(e);
  }
}
```

**Hardcoded Data**:
```dart
Future<Result<List<Continent>>> getContinents() async {
  return Result.ok([
    Continent(
      name: 'Europe',
      imageUrl: 'https://example.com/europe.jpg',
    ),
    Continent(
      name: 'Asia',
      imageUrl: 'https://example.com/asia.jpg',
    ),
    // ...
  ]);
}

Future<Result<User>> getUser() async {
  return Result.ok(User(
    name: 'Sofie Bauer',
    picture: Assets.userImage,
  ));
}
```

**Assets**:
- [assets/destinations.json](../app/assets/destinations.json) (~81 KB)
- [assets/activities.json](../app/assets/activities.json) (~2 MB)
- [assets/user.jpg](../app/assets/user.jpg)

---

#### 2. ApiClient

**File**: [data/services/api/api_client.dart](../app/lib/data/services/api/api_client.dart)

**Purpose**: HTTP client for communicating with the backend server.

**Configuration**:
```dart
class ApiClient {
  final String baseUrl;
  final http.Client _client;
  final AuthHeaderProvider _authHeaderProvider;

  ApiClient({
    required this.baseUrl,  // e.g., 'http://localhost:8080'
    required http.Client client,
    required AuthHeaderProvider authHeaderProvider,
  }) : _client = client,
       _authHeaderProvider = authHeaderProvider;
}
```

**Endpoints**:

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/continent` | List continents |
| GET | `/destination` | List destinations |
| GET | `/destination/:ref/activity` | Activities by destination |
| GET | `/booking` | User's bookings |
| GET | `/booking/:id` | Single booking |
| POST | `/booking` | Create booking |
| DELETE | `/booking/:id` | Delete booking |
| GET | `/user` | Current user profile |

**Example Method**:
```dart
Future<Result<List<Destination>>> getDestinations() async {
  try {
    final uri = Uri.parse('$baseUrl/destination');
    final headers = await _authHeaderProvider.getAuthHeader();

    final response = await _client.get(uri, headers: headers);

    if (response.statusCode != 200) {
      return Result.error(
        Exception('Failed to load destinations: ${response.statusCode}'),
      );
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

**Key Features**:
- **Authentication**: Automatically injects JWT token in headers
- **Error Handling**: Converts HTTP errors to `Result.error`
- **Logging**: Logs all requests and errors
- **Type Safety**: Returns typed domain models

**AuthHeaderProvider**:
```dart
class AuthHeaderProvider {
  final SharedPreferencesService _sharedPreferencesService;

  AuthHeaderProvider({
    required SharedPreferencesService sharedPreferencesService,
  }) : _sharedPreferencesService = sharedPreferencesService;

  Future<Map<String, String>> getAuthHeader() async {
    final token = await _sharedPreferencesService.fetchToken();
    if (token == null) return {};

    return {
      'Authorization': 'Bearer $token',
    };
  }
}
```

---

#### 3. AuthApiClient

**File**: [data/services/api/auth_api_client.dart](../app/lib/data/services/api/auth_api_client.dart)

**Purpose**: Handle login requests (separate from ApiClient because login doesn't require auth).

**Method**:
```dart
class AuthApiClient {
  final String baseUrl;
  final http.Client _client;

  AuthApiClient({
    required this.baseUrl,
    required http.Client client,
  }) : _client = client;

  Future<Result<LoginResponse>> login({
    required String email,
    required String password,
  }) async {
    try {
      final uri = Uri.parse('$baseUrl/login');
      final body = jsonEncode(LoginRequest(
        email: email,
        password: password,
      ).toJson());

      final response = await _client.post(
        uri,
        headers: {'Content-Type': 'application/json'},
        body: body,
      );

      if (response.statusCode != 200) {
        return Result.error(
          Exception('Login failed: ${response.statusCode}'),
        );
      }

      final json = jsonDecode(response.body);
      final loginResponse = LoginResponse.fromJson(json);
      return Result.ok(loginResponse);
    } on Exception catch (e) {
      return Result.error(e);
    }
  }
}
```

**DTOs**:
```dart
@freezed
class LoginRequest with _$LoginRequest {
  const factory LoginRequest({
    required String email,
    required String password,
  }) = _LoginRequest;

  factory LoginRequest.fromJson(Map<String, dynamic> json) =>
      _$LoginRequestFromJson(json);
}

@freezed
class LoginResponse with _$LoginResponse {
  const factory LoginResponse({
    required String token,
  }) = _LoginResponse;

  factory LoginResponse.fromJson(Map<String, dynamic> json) =>
      _$LoginResponseFromJson(json);
}
```

---

#### 4. SharedPreferencesService

**File**: [data/services/shared_preferences_service.dart](../app/lib/data/services/shared_preferences_service.dart)

**Purpose**: Persist authentication token locally.

**Implementation**:
```dart
class SharedPreferencesService {
  static const String _tokenKey = 'auth_token';

  Future<String?> fetchToken() async {
    final prefs = await SharedPreferences.getInstance();
    return prefs.getString(_tokenKey);
  }

  Future<void> saveToken(String? token) async {
    final prefs = await SharedPreferences.getInstance();
    if (token == null) {
      await prefs.remove(_tokenKey);
    } else {
      await prefs.setString(_tokenKey, token);
    }
  }
}
```

**Usage**:
- On login: Save token
- On logout: Remove token
- On app start: Check if token exists (determines auth state)
- On API requests: Include token in headers

---

## Presentation Layer

**Location**: [app/lib/ui/](../app/lib/ui/)

**Purpose**: Everything related to displaying information and handling user interaction.

**Organization**: Feature-based structure where each feature has its own folder.

---

### Feature Structure

Each feature follows this pattern:

```
ui/[feature]/
├── view_models/
│   └── [feature]_viewmodel.dart
└── widgets/
    ├── [feature]_screen.dart
    └── [components].dart
```

---

### Feature 1: Home

**Location**: [app/lib/ui/home/](../app/lib/ui/home/)

**Purpose**: Display user's bookings and allow navigation to create new ones.

#### HomeViewModel

**File**: [ui/home/view_models/home_viewmodel.dart](../app/lib/ui/home/view_models/home_viewmodel.dart)

**Responsibilities**:
- Load user bookings
- Load user profile
- Handle booking deletion

**Code**:
```dart
class HomeViewModel extends ChangeNotifier {
  final BookingRepository _bookingRepository;
  final UserRepository _userRepository;

  HomeViewModel({
    required BookingRepository bookingRepository,
    required UserRepository userRepository,
  })  : _bookingRepository = bookingRepository,
        _userRepository = userRepository {
    load = Command0(_load)..execute(); // Auto-load on creation
    deleteBooking = Command1(_deleteBooking);
  }

  // Commands
  late Command0 load;
  late Command1<void, int> deleteBooking;

  // State
  List<BookingSummary> _bookings = [];
  User? _user;

  // Getters
  List<BookingSummary> get bookings => _bookings;
  User? get user => _user;

  // Actions
  Future<Result<void>> _load() async {
    final results = await (
      _bookingRepository.getBookingsList(),
      _userRepository.getUser(),
    ).wait;

    switch (results.$1) {
      case Ok<List<BookingSummary>>():
        _bookings = results.$1.value;
      case Error<List<BookingSummary>>():
        log.warning('Failed to load bookings', results.$1.error);
    }

    switch (results.$2) {
      case Ok<User>():
        _user = results.$2.value;
      case Error<User>():
        log.warning('Failed to load user', results.$2.error);
    }

    notifyListeners();
    return Result.ok(null);
  }

  Future<Result<void>> _deleteBooking(int id) async {
    final result = await _bookingRepository.delete(id);

    switch (result) {
      case Ok():
        _bookings.removeWhere((b) => b.id == id);
        notifyListeners();
      case Error():
        log.warning('Failed to delete booking', result.error);
    }

    return result;
  }
}
```

**Key Patterns**:
- **Parallel Loading**: Uses `wait` to load bookings and user simultaneously
- **Command Pattern**: `load` and `deleteBooking` are Command objects
- **Auto-Execute**: `load` executes immediately on creation
- **Reactive**: Calls `notifyListeners()` after state changes

---

#### HomeScreen

**File**: [ui/home/widgets/home_screen.dart](../app/lib/ui/home/widgets/home_screen.dart)

**Structure**:
```dart
class HomeScreen extends StatefulWidget {
  final HomeViewModel viewModel;

  const HomeScreen({required this.viewModel});

  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('My Bookings'),
        actions: [
          // User profile
          ListenableBuilder(
            listenable: widget.viewModel,
            builder: (context, _) {
              final user = widget.viewModel.user;
              if (user == null) return SizedBox.shrink();

              return CircleAvatar(
                backgroundImage: AssetImage(user.picture),
              );
            },
          ),
          LogoutButton(),
        ],
      ),
      body: ListenableBuilder(
        listenable: widget.viewModel.load,
        builder: (context, _) {
          // Loading state
          if (widget.viewModel.load.running) {
            return Center(child: CircularProgressIndicator());
          }

          // Error state
          if (widget.viewModel.load.error) {
            return ErrorIndicator(
              title: 'Failed to load bookings',
              label: widget.viewModel.load.error.toString(),
              onPressed: widget.viewModel.load.execute,
            );
          }

          // Empty state
          if (widget.viewModel.bookings.isEmpty) {
            return Center(
              child: Text('No bookings yet!'),
            );
          }

          // Success state - show bookings
          return ListView.builder(
            itemCount: widget.viewModel.bookings.length,
            itemBuilder: (context, index) {
              final booking = widget.viewModel.bookings[index];
              return BookingCard(
                booking: booking,
                onTap: () => _viewBooking(booking.id),
                onDelete: () => _deleteBooking(booking.id),
              );
            },
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => context.go(Routes.search),
        child: Icon(Icons.add),
      ),
    );
  }

  void _viewBooking(int id) {
    context.push(Routes.bookingWithId(id));
  }

  Future<void> _deleteBooking(int id) async {
    await widget.viewModel.deleteBooking.execute(id);

    if (widget.viewModel.deleteBooking.completed) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Booking deleted')),
      );
    }
  }
}
```

**UI Patterns**:
- **ListenableBuilder**: Rebuilds on ViewModel changes
- **State Handling**: Shows loading, error, empty, and success states
- **Navigation**: Uses go_router for navigation
- **Feedback**: Shows SnackBar after deletion

---

### Feature 2: Search Form

**Location**: [app/lib/ui/search_form/](../app/lib/ui/search_form/)

**Purpose**: Collect trip search criteria (continent, dates, guests).

#### SearchFormViewModel

**File**: [ui/search_form/view_models/search_form_viewmodel.dart](../app/lib/ui/search_form/view_models/search_form_viewmodel.dart)

**Responsibilities**:
- Load continents
- Track form selections
- Validate form completeness
- Save to ItineraryConfig

**Code Highlights**:
```dart
class SearchFormViewModel extends ChangeNotifier {
  final ContinentRepository _continentRepository;
  final ItineraryConfigRepository _itineraryConfigRepository;

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

  // Setters with validation
  set selectedContinent(String? value) {
    _selectedContinent = value;
    notifyListeners();
  }

  set startDate(DateTime? value) {
    _startDate = value;
    // Auto-adjust end date if invalid
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
    _guests = value.clamp(1, 10);
    notifyListeners();
  }

  Future<Result<void>> _load() async {
    final result = await _continentRepository.getContinents();
    switch (result) {
      case Ok<List<Continent>>():
        _continents = result.value;
        notifyListeners();
      case Error<List<Continent>>():
        log.warning('Failed to load continents', result.error);
    }
    return result as Result<void>;
  }

  Future<Result<void>> _updateItineraryConfig() async {
    final config = ItineraryConfig(
      continent: _selectedContinent,
      startDate: _startDate,
      endDate: _endDate,
      guests: _guests,
    );
    return await _itineraryConfigRepository.setItineraryConfig(config);
  }
}
```

**Key Features**:
- **Validation**: `valid` getter checks all required fields
- **Smart Setters**: Date setters auto-adjust to maintain consistency
- **Reactive**: Each setter calls `notifyListeners()`

---

#### SearchFormScreen

**File**: [ui/search_form/widgets/search_form_screen.dart](../app/lib/ui/search_form/widgets/search_form_screen.dart)

**UI Highlights**:
```dart
class SearchFormScreen extends StatelessWidget {
  final SearchFormViewModel viewModel;

  const SearchFormScreen({required this.viewModel});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Plan Your Trip')),
      body: ListenableBuilder(
        listenable: viewModel,
        builder: (context, _) {
          return Column(
            children: [
              // Continent selector
              ContinentSelector(
                continents: viewModel.continents,
                selected: viewModel.selectedContinent,
                onSelect: (continent) {
                  viewModel.selectedContinent = continent;
                },
              ),

              // Date pickers
              DateRangePicker(
                startDate: viewModel.startDate,
                endDate: viewModel.endDate,
                onStartDateChanged: (date) {
                  viewModel.startDate = date;
                },
                onEndDateChanged: (date) {
                  viewModel.endDate = date;
                },
              ),

              // Guests counter
              GuestCounter(
                guests: viewModel.guests,
                onChanged: (count) {
                  viewModel.guests = count;
                },
              ),

              Spacer(),

              // Submit button
              ListenableBuilder(
                listenable: viewModel.updateItineraryConfig,
                builder: (context, _) {
                  return ElevatedButton(
                    onPressed: viewModel.valid ? _search : null,
                    child: viewModel.updateItineraryConfig.running
                        ? CircularProgressIndicator()
                        : Text('Search'),
                  );
                },
              ),
            ],
          );
        },
      ),
    );
  }

  Future<void> _search() async {
    await viewModel.updateItineraryConfig.execute();

    if (viewModel.updateItineraryConfig.completed) {
      context.go(Routes.results);
    }
  }
}
```

**UI Patterns**:
- **Reactive Form**: Entire form rebuilds on ViewModel changes
- **Validation**: Button disabled until form is valid
- **Loading State**: Shows spinner while saving
- **Navigation**: Goes to results screen after saving

---

### Core UI Components

**Location**: [app/lib/ui/core/](../app/lib/ui/core/)

**Purpose**: Shared UI components used across features.

#### Themes

**Location**: [ui/core/themes/](../app/lib/ui/core/themes/)

**Light Theme** ([app_theme.dart](../app/lib/ui/core/themes/app_theme.dart)):
```dart
class AppTheme {
  static ThemeData get lightTheme {
    return ThemeData(
      useMaterial3: true,
      colorScheme: ColorScheme.fromSeed(
        seedColor: Colors.blue,
        brightness: Brightness.light,
      ),
      textTheme: GoogleFonts.montserratTextTheme(),
    );
  }

  static ThemeData get darkTheme {
    return ThemeData(
      useMaterial3: true,
      colorScheme: ColorScheme.fromSeed(
        seedColor: Colors.blue,
        brightness: Brightness.dark,
      ),
      textTheme: GoogleFonts.montserratTextTheme(
        ThemeData.dark().textTheme,
      ),
    );
  }
}
```

**Features**:
- Material 3 design
- Light and dark modes
- Google Fonts (Montserrat)
- Consistent color scheme

---

#### Reusable Widgets

**Location**: [ui/core/ui/](../app/lib/ui/core/ui/)

**ErrorIndicator** ([error_indicator.dart](../app/lib/ui/core/ui/error_indicator.dart)):
```dart
class ErrorIndicator extends StatelessWidget {
  final String title;
  final String? label;
  final VoidCallback? onPressed;

  const ErrorIndicator({
    required this.title,
    this.label,
    this.onPressed,
  });

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Icon(Icons.error_outline, size: 48, color: Colors.red),
          SizedBox(height: 16),
          Text(title, style: Theme.of(context).textTheme.headlineSmall),
          if (label != null) ...[
            SizedBox(height: 8),
            Text(label!, style: Theme.of(context).textTheme.bodyMedium),
          ],
          if (onPressed != null) ...[
            SizedBox(height: 16),
            ElevatedButton(
              onPressed: onPressed,
              child: Text('Retry'),
            ),
          ],
        ],
      ),
    );
  }
}
```

**Other Reusable Components**:
- **TagChip**: Display tags/labels
- **SearchBar**: Styled search input
- **LoadingIndicator**: Consistent loading spinner
- **EmptyState**: Placeholder for empty lists

---

## Utils Layer

**Location**: [app/lib/utils/](../app/lib/utils/)

**Purpose**: Shared utilities used across the application.

---

### Result Pattern

**File**: [utils/result.dart](../app/lib/utils/result.dart)

**Purpose**: Type-safe error handling without exceptions.

**Implementation**:
```dart
sealed class Result<T> {
  const Result();

  const factory Result.ok(T value) = Ok<T>._;
  const factory Result.error(Exception error) = Error<T>._;

  R when<R>({
    required R Function(T value) ok,
    required R Function(Exception error) error,
  }) {
    return switch (this) {
      Ok<T>(:final value) => ok(value),
      Error<T>(:final error) => error(error),
    };
  }
}

class Ok<T> extends Result<T> {
  final T value;
  const Ok._(this.value);
}

class Error<T> extends Result<T> {
  final Exception error;
  const Error._(this.error);
}

// Extensions for easier access
extension ResultExtension<T> on Result<T> {
  Ok<T> get asOk => this as Ok<T>;
  Error<T> get asError => this as Error<T>;

  T? get valueOrNull => switch (this) {
    Ok<T>(:final value) => value,
    Error<T>() => null,
  };
}
```

**Usage Patterns**:

**Pattern Matching**:
```dart
final result = await repository.getBooking(id);
switch (result) {
  case Ok<Booking>(:final value):
    _booking = value;
    print('Loaded booking: ${value.destination.name}');
  case Error<Booking>(:final error):
    _error = error.toString();
    print('Failed to load: $error');
}
```

**When Method**:
```dart
final message = result.when(
  ok: (booking) => 'Loaded ${booking.destination.name}',
  error: (error) => 'Error: $error',
);
```

**Null-Safe Access**:
```dart
final booking = result.valueOrNull; // Returns null if error
```

---

### Command Pattern

**File**: [utils/command.dart](../app/lib/utils/command.dart)

**Purpose**: Encapsulate async UI actions with automatic state management.

**Implementation**:
```dart
abstract class Command<T> extends ChangeNotifier {
  bool _running = false;
  Exception? _error;
  T? _result;

  bool get running => _running;
  Exception? get error => _error;
  T? get result => _result;
  bool get completed => !_running && _error == null;

  void clearError() {
    _error = null;
    notifyListeners();
  }
}

class Command0<T> extends Command<T> {
  final Future<Result<T>> Function() _action;

  Command0(this._action);

  Future<void> execute() async {
    if (_running) return; // Prevent duplicate execution

    _running = true;
    _error = null;
    _result = null;
    notifyListeners();

    try {
      final result = await _action();
      switch (result) {
        case Ok<T>(:final value):
          _result = value;
        case Error<T>(:final error):
          _error = error;
      }
    } on Exception catch (e) {
      _error = e;
    } finally {
      _running = false;
      notifyListeners();
    }
  }
}

class Command1<T, A> extends Command<T> {
  final Future<Result<T>> Function(A) _action;

  Command1(this._action);

  Future<void> execute(A argument) async {
    if (_running) return;

    _running = true;
    _error = null;
    _result = null;
    notifyListeners();

    try {
      final result = await _action(argument);
      switch (result) {
        case Ok<T>(:final value):
          _result = value;
        case Error<T>(:final error):
          _error = error;
      }
    } on Exception catch (e) {
      _error = e;
    } finally {
      _running = false;
      notifyListeners();
    }
  }
}
```

**Features**:
- **Running State**: Tracks if command is executing
- **Error State**: Stores exception if command fails
- **Result State**: Stores successful result
- **Completed State**: True when finished successfully
- **Prevents Duplicates**: Ignores execute() while running
- **Reactive**: Extends ChangeNotifier for UI updates

**Usage Example**:
```dart
// In ViewModel
class BookingViewModel extends ChangeNotifier {
  BookingViewModel() {
    loadBooking = Command1(_loadBooking);
  }

  late Command1<Booking, int> loadBooking;

  Future<Result<Booking>> _loadBooking(int id) async {
    return await repository.getBooking(id);
  }
}

// In Widget
ListenableBuilder(
  listenable: viewModel.loadBooking,
  builder: (context, _) {
    // Loading state
    if (viewModel.loadBooking.running) {
      return CircularProgressIndicator();
    }

    // Error state
    if (viewModel.loadBooking.error != null) {
      return ErrorIndicator(
        title: 'Failed to load booking',
        label: viewModel.loadBooking.error.toString(),
        onPressed: () => viewModel.loadBooking.execute(bookingId),
      );
    }

    // Completed state
    if (viewModel.loadBooking.completed) {
      final booking = viewModel.loadBooking.result!;
      return BookingDetails(booking: booking);
    }

    // Initial state
    return SizedBox.shrink();
  },
)
```

---

## Routing Layer

**Location**: [app/lib/routing/](../app/lib/routing/)

**Purpose**: Configure navigation and routing.

---

### Router Configuration

**File**: [routing/router.dart](../app/lib/routing/router.dart)

**Purpose**: Define all routes and navigation logic.

**Implementation**:
```dart
GoRouter router(AuthRepository authRepository) {
  return GoRouter(
    initialLocation: Routes.home,
    debugLogDiagnostics: true,
    redirect: (context, state) => _redirect(context, state, authRepository),
    routes: [
      // Login route
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

      // Home route with nested routes
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
        routes: [
          // Search form
          GoRoute(
            path: Routes.searchRelative,
            builder: (context, state) {
              return SearchFormScreen(
                viewModel: SearchFormViewModel(
                  continentRepository: context.read(),
                  itineraryConfigRepository: context.read(),
                ),
              );
            },
          ),

          // Results
          GoRoute(
            path: Routes.resultsRelative,
            builder: (context, state) {
              return ResultsScreen(
                viewModel: ResultsViewModel(
                  destinationRepository: context.read(),
                  itineraryConfigRepository: context.read(),
                ),
              );
            },
          ),

          // Activities
          GoRoute(
            path: Routes.activitiesRelative,
            builder: (context, state) {
              return ActivitiesScreen(
                viewModel: ActivitiesViewModel(
                  activityRepository: context.read(),
                  itineraryConfigRepository: context.read(),
                ),
              );
            },
          ),

          // Booking
          GoRoute(
            path: Routes.bookingRelative,
            builder: (context, state) {
              final id = state.uri.queryParameters['id'];
              return BookingScreen(
                viewModel: BookingViewModel(
                  bookingRepository: context.read(),
                  itineraryConfigRepository: context.read(),
                  bookingCreateUseCase: context.read(),
                  bookingShareUseCase: context.read(),
                  bookingId: id != null ? int.parse(id) : null,
                ),
              );
            },
          ),
        ],
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

  // Not logged in → redirect to login
  if (!loggedIn && !loggingIn) {
    return Routes.login;
  }

  // Logged in but on login screen → redirect to home
  if (loggedIn && loggingIn) {
    return Routes.home;
  }

  // No redirect needed
  return null;
}
```

**Key Features**:
- **Authentication Redirects**: Auto-redirects based on auth state
- **Nested Routes**: Booking flow routes nested under home
- **Dependency Injection**: ViewModels created at route level
- **Query Parameters**: Booking route accepts optional `id` parameter
- **Debug Logging**: Enabled for development

---

### Routes Constants

**File**: [routing/routes.dart](../app/lib/routing/routes.dart)

**Purpose**: Centralized route path definitions.

**Implementation**:
```dart
abstract class Routes {
  // Absolute paths
  static const String home = '/';
  static const String login = '/login';
  static const String search = '/search';
  static const String results = '/results';
  static const String activities = '/activities';
  static const String booking = '/booking';

  // Relative paths (for nested routes)
  static const String searchRelative = 'search';
  static const String resultsRelative = 'results';
  static const String activitiesRelative = 'activities';
  static const String bookingRelative = 'booking';

  // Parameterized routes
  static String bookingWithId(int id) => '/booking?id=$id';
}
```

**Usage**:
```dart
// Navigation
context.go(Routes.search);
context.push(Routes.bookingWithId(42));

// In router
GoRoute(path: Routes.home, ...)
```

---

## Configuration Layer

**Location**: [app/lib/config/](../app/lib/config/)

**Purpose**: App-wide configuration.

---

### Dependency Injection

**File**: [config/dependencies.dart](../app/lib/config/dependencies.dart)

**Purpose**: Configure all dependencies for both environments.

**Local Configuration**:
```dart
List<SingleChildWidget> get providersLocal {
  return [
    // Data service
    Provider(
      create: (context) => LocalDataService(),
    ),

    // Repositories
    Provider(
      create: (context) => ContinentRepositoryLocal(
        localDataService: context.read(),
      ) as ContinentRepository,
    ),
    Provider(
      create: (context) => DestinationRepositoryLocal(
        localDataService: context.read(),
      ) as DestinationRepository,
    ),
    Provider(
      create: (context) => ActivityRepositoryLocal(
        localDataService: context.read(),
      ) as ActivityRepository,
    ),
    Provider(
      create: (context) => BookingRepositoryLocal(
        localDataService: context.read(),
      ) as BookingRepository,
    ),
    ChangeNotifierProvider(
      create: (context) => AuthRepositoryDev() as AuthRepository,
    ),
    Provider(
      create: (context) => UserRepositoryLocal(
        localDataService: context.read(),
      ) as UserRepository,
    ),
    Provider(
      create: (context) => ItineraryConfigRepositoryMemory()
          as ItineraryConfigRepository,
    ),

    // Use cases
    Provider(
      create: (context) => BookingCreateUseCase(
        destinationRepository: context.read(),
        activityRepository: context.read(),
        bookingRepository: context.read(),
      ),
    ),
    Provider(
      create: (context) => BookingShareUseCase.withSharePlus(),
    ),
  ];
}
```

**Remote Configuration**:
```dart
List<SingleChildWidget> get providersRemote {
  return [
    // HTTP client
    Provider(create: (context) => http.Client()),

    // Shared preferences
    Provider(create: (context) => SharedPreferencesService()),

    // Auth API client
    Provider(
      create: (context) => AuthApiClient(
        baseUrl: 'http://localhost:8080',
        client: context.read(),
      ),
    ),

    // Auth repository
    ChangeNotifierProvider(
      create: (context) => AuthRepositoryRemote(
        authApiClient: context.read(),
        sharedPreferencesService: context.read(),
      ) as AuthRepository,
    ),

    // Auth header provider
    Provider(
      create: (context) => AuthHeaderProvider(
        sharedPreferencesService: context.read(),
      ),
    ),

    // API client
    Provider(
      create: (context) => ApiClient(
        baseUrl: 'http://localhost:8080',
        client: context.read(),
        authHeaderProvider: context.read(),
      ),
    ),

    // Remote repositories
    Provider(
      create: (context) => ContinentRepositoryRemote(
        apiClient: context.read(),
      ) as ContinentRepository,
    ),
    Provider(
      create: (context) => DestinationRepositoryRemote(
        apiClient: context.read(),
      ) as DestinationRepository,
    ),
    Provider(
      create: (context) => ActivityRepositoryRemote(
        apiClient: context.read(),
      ) as ActivityRepository,
    ),
    Provider(
      create: (context) => BookingRepositoryRemote(
        apiClient: context.read(),
      ) as BookingRepository,
    ),
    Provider(
      create: (context) => UserRepositoryRemote(
        apiClient: context.read(),
      ) as UserRepository,
    ),
    Provider(
      create: (context) => ItineraryConfigRepositoryMemory()
          as ItineraryConfigRepository,
    ),

    // Use cases (same for both environments)
    Provider(
      create: (context) => BookingCreateUseCase(
        destinationRepository: context.read(),
        activityRepository: context.read(),
        bookingRepository: context.read(),
      ),
    ),
    Provider(
      create: (context) => BookingShareUseCase.withSharePlus(),
    ),
  ];
}
```

**Key Patterns**:
- **Abstract Types**: Providers return abstract repository types
- **Dependency Resolution**: `context.read()` resolves dependencies
- **Environment Switching**: Same provider keys, different implementations
- **Shared Providers**: Use cases are identical in both environments

---

## Layer Communication

### Communication Rules

**Allowed Dependencies**:
```
Presentation → Domain ✅
Presentation → Data ✅
Domain → (no dependencies) ✅
Data → Domain ✅
```

**Forbidden Dependencies**:
```
Domain → Presentation ❌
Domain → Data ❌
Data → Presentation ❌
```

---

### Example: Create Booking Flow

**Complete flow across all layers**:

```
1. USER INTERACTION (Presentation Layer)
   BookingScreen: User taps "Confirm Booking"
   ↓
   viewModel.createBooking.execute()

2. VIEWMODEL (Presentation Layer)
   BookingViewModel._createBooking()
   ↓
   Get ItineraryConfig from repository
   ↓
   Call BookingCreateUseCase

3. USE CASE (Domain Layer)
   BookingCreateUseCase.createFrom(config)
   ↓
   Fetch Destination via DestinationRepository
   ↓
   Fetch Activities via ActivityRepository
   ↓
   Create Booking entity (domain model)
   ↓
   Persist via BookingRepository

4. REPOSITORY (Data Layer)
   BookingRepository.createBooking(booking)
   ↓
   Delegate to implementation (Local or Remote)

5. SERVICE (Data Layer)
   Local: Add to in-memory list
   Remote: POST to API via ApiClient

6. RESULT PROPAGATION
   Service → Repository → Use Case → ViewModel → UI
   ↓
   Success: Navigate to booking detail screen
   Error: Show error message
```

---

## Summary

The Compass App's layered architecture provides:

✅ **Clear Separation**: Each layer has distinct responsibilities
✅ **Testability**: Easy to test each layer independently
✅ **Flexibility**: Swap implementations without changing logic
✅ **Scalability**: Add features following established patterns
✅ **Maintainability**: Organized structure makes code easy to navigate

**Next**: Explore [Design Patterns](./03-design-patterns.md) for detailed pattern implementations.
