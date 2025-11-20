# 03 - Design Patterns

## 📋 Table of Contents

- [Introduction](#introduction)
- [Repository Pattern](#repository-pattern)
- [MVVM Pattern](#mvvm-pattern)
- [Command Pattern](#command-pattern)
- [Result Pattern](#result-pattern)
- [Dependency Injection](#dependency-injection)
- [Factory Pattern](#factory-pattern)
- [Strategy Pattern](#strategy-pattern)
- [Observer Pattern](#observer-pattern)
- [Builder Pattern](#builder-pattern)
- [Pattern Combinations](#pattern-combinations)

---

## Introduction

Design patterns are **proven solutions to common problems** in software design. They're like templates you can customize to solve recurring design challenges in your code.

**Why patterns matter**:
- **Common Language**: Developers instantly understand "Repository Pattern" vs explaining the whole concept
- **Proven Solutions**: Patterns have been tested and refined over decades
- **Maintainability**: Code following patterns is easier to understand and modify
- **Scalability**: Patterns make it easier to add features without breaking existing code

The Compass App demonstrates **9 key design patterns** working together to create a robust, maintainable architecture.

**Analogy**: Design patterns are like carpentry techniques. Just as a carpenter knows when to use a dovetail joint vs. a mortise and tenon, developers know when to use the Repository Pattern vs. the Strategy Pattern.

---

## Repository Pattern

### Overview

**Purpose**: Provide an abstraction layer between the domain logic and data sources.

**Problem Solved**: Your app needs data from various sources (database, API, cache, local files). Without a repository, business logic gets cluttered with data access code and becomes tightly coupled to specific data sources.

**Solution**: Create a repository interface that hides data source details. Business logic uses the interface without knowing where data comes from.

**Analogy**: Think of a library. You ask the librarian (repository) for a book. You don't need to know if it's on the shelf, in storage, or being ordered from another library. The librarian handles those details.

---

### Implementation in Compass App

#### Step 1: Define Repository Interface

**File**: [app/lib/data/repositories/booking/booking_repository.dart](../app/lib/data/repositories/booking/booking_repository.dart)

```dart
/// Abstract repository defining booking data operations
abstract class BookingRepository {
  /// Get list of user's bookings (lightweight summaries)
  Future<Result<List<BookingSummary>>> getBookingsList();

  /// Get detailed booking by ID
  Future<Result<Booking>> getBooking(int id);

  /// Create a new booking
  Future<Result<void>> createBooking(Booking booking);

  /// Delete a booking by ID
  Future<Result<void>> delete(int id);
}
```

**Key Points**:
- Abstract class (or interface)
- Returns `Result<T>` for type-safe error handling
- Domain-focused (uses domain models, not DTOs)
- No implementation details

---

#### Step 2: Local Implementation (Development)

**File**: [app/lib/data/repositories/booking/booking_repository_local.dart](../app/lib/data/repositories/booking/booking_repository_local.dart)

```dart
/// Local implementation storing bookings in memory
class BookingRepositoryLocal implements BookingRepository {
  final LocalDataService _localDataService;
  final Logger _log = Logger('BookingRepositoryLocal');

  // In-memory storage
  final List<Booking> _bookings = [];
  int _sequenceId = 0;

  BookingRepositoryLocal({
    required LocalDataService localDataService,
  }) : _localDataService = localDataService {
    // Initialize with a default booking
    _createDefaultBooking();
  }

  @override
  Future<Result<List<BookingSummary>>> getBookingsList() async {
    _log.fine('Getting bookings list (${_bookings.length} bookings)');

    // Convert full bookings to summaries
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
    _log.fine('Getting booking $id');

    try {
      final booking = _bookings.firstWhere((b) => b.id == id);
      return Result.ok(booking);
    } catch (e) {
      _log.warning('Booking $id not found');
      return Result.error(Exception('Booking not found'));
    }
  }

  @override
  Future<Result<void>> createBooking(Booking booking) async {
    _log.fine('Creating booking for ${booking.destination.name}');

    // Assign ID if not present
    final bookingWithId = booking.id == null
        ? booking.copyWith(id: _sequenceId++)
        : booking;

    _bookings.add(bookingWithId);
    _log.info('Created booking ${bookingWithId.id}');

    return Result.ok(null);
  }

  @override
  Future<Result<void>> delete(int id) async {
    _log.fine('Deleting booking $id');

    final removed = _bookings.removeWhere((b) => b.id == id);
    if (removed == 0) {
      _log.warning('Booking $id not found for deletion');
      return Result.error(Exception('Booking not found'));
    }

    _log.info('Deleted booking $id');
    return Result.ok(null);
  }

  /// Create a default booking for demonstration
  Future<void> _createDefaultBooking() async {
    _log.fine('Creating default booking');

    // Load sample destination
    final destinationsResult = await _localDataService.getDestinations();
    if (destinationsResult is Error) return;
    final destination = destinationsResult.asOk.value.first;

    // Load sample activities
    final activitiesResult = await _localDataService.getActivities();
    if (activitiesResult is Error) return;
    final activities = activitiesResult.asOk.value
        .where((a) => a.destinationRef == destination.ref)
        .take(3)
        .toList();

    // Create booking
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

**Characteristics**:
- ✅ In-memory storage (fast, no persistence)
- ✅ Auto-incrementing IDs
- ✅ Creates sample data
- ✅ Comprehensive logging
- ❌ Data lost on app restart

---

#### Step 3: Remote Implementation (Production)

**File**: [app/lib/data/repositories/booking/booking_repository_remote.dart](../app/lib/data/repositories/booking/booking_repository_remote.dart)

```dart
/// Remote implementation using HTTP API
class BookingRepositoryRemote implements BookingRepository {
  final ApiClient _apiClient;
  final Logger _log = Logger('BookingRepositoryRemote');

  BookingRepositoryRemote({
    required ApiClient apiClient,
  }) : _apiClient = apiClient;

  @override
  Future<Result<List<BookingSummary>>> getBookingsList() async {
    _log.fine('Fetching bookings list from API');
    return await _apiClient.getBookings();
  }

  @override
  Future<Result<Booking>> getBooking(int id) async {
    _log.fine('Fetching booking $id from API');
    return await _apiClient.getBooking(id);
  }

  @override
  Future<Result<void>> createBooking(Booking booking) async {
    _log.fine('Creating booking on server: ${booking.destination.name}');
    return await _apiClient.postBooking(booking);
  }

  @override
  Future<Result<void>> delete(int id) async {
    _log.fine('Deleting booking $id on server');
    return await _apiClient.deleteBooking(id);
  }
}
```

**Characteristics**:
- ✅ Persistent storage (server database)
- ✅ Real network operations
- ✅ Requires authentication
- ❌ Network latency
- ❌ Requires backend server running

---

#### Step 4: Using the Repository

**In ViewModel** ([app/lib/ui/home/view_models/home_viewmodel.dart](../app/lib/ui/home/view_models/home_viewmodel.dart)):

```dart
class HomeViewModel extends ChangeNotifier {
  final BookingRepository _bookingRepository; // Abstract type!
  final Logger _log = Logger('HomeViewModel');

  HomeViewModel({
    required BookingRepository bookingRepository, // Dependency injection
    required UserRepository userRepository,
  })  : _bookingRepository = bookingRepository,
        _userRepository = userRepository {
    load = Command0(_load)..execute();
    deleteBooking = Command1(_deleteBooking);
  }

  List<BookingSummary> _bookings = [];
  List<BookingSummary> get bookings => _bookings;

  late Command0 load;
  late Command1<void, int> deleteBooking;

  Future<Result<void>> _load() async {
    _log.fine('Loading bookings');

    // Repository call - don't know if it's local or remote!
    final result = await _bookingRepository.getBookingsList();

    switch (result) {
      case Ok<List<BookingSummary>>():
        _bookings = result.value;
        _log.info('Loaded ${_bookings.length} bookings');
      case Error<List<BookingSummary>>():
        _log.warning('Failed to load bookings', result.error);
    }

    notifyListeners();
    return Result.ok(null);
  }

  Future<Result<void>> _deleteBooking(int id) async {
    _log.fine('Deleting booking $id');

    final result = await _bookingRepository.delete(id);

    switch (result) {
      case Ok():
        _bookings.removeWhere((b) => b.id == id);
        _log.info('Deleted booking $id');
        notifyListeners();
      case Error():
        _log.warning('Failed to delete booking $id', result.error);
    }

    return result;
  }
}
```

**Key Insight**: `HomeViewModel` depends on `BookingRepository` (abstract type), not on `BookingRepositoryLocal` or `BookingRepositoryRemote`. It works with either!

---

### Benefits

1. **Testability**: Easy to inject a mock repository
   ```dart
   test('HomeViewModel loads bookings', () {
     final mockRepo = MockBookingRepository();
     when(() => mockRepo.getBookingsList()).thenAnswer(
       (_) async => Result.ok([booking1, booking2]),
     );

     final viewModel = HomeViewModel(bookingRepository: mockRepo);
     await viewModel.load.execute();

     expect(viewModel.bookings, [booking1, booking2]);
   });
   ```

2. **Flexibility**: Swap implementations by changing dependency injection config
   ```dart
   // Development
   Provider(
     create: (context) => BookingRepositoryLocal(...) as BookingRepository,
   )

   // Production
   Provider(
     create: (context) => BookingRepositoryRemote(...) as BookingRepository,
   )
   ```

3. **Maintainability**: Business logic isolated from data access details

4. **Scalability**: Add new implementations (e.g., Firebase) without changing ViewModels

---

### When to Use

✅ **Use Repository Pattern when**:
- You have multiple data sources (API, cache, database)
- You want to swap data sources easily
- You need to test business logic without real data sources
- You want to hide data access complexity

❌ **Don't use Repository Pattern when**:
- You have a single, simple data source that won't change
- You're building a prototype or throwaway code
- The abstraction adds more complexity than it removes

---

## MVVM Pattern

### Overview

**Purpose**: Separate presentation logic from UI code to improve testability and maintainability.

**Problem Solved**: When business logic is mixed with UI code, it's hard to test and reuse. Changes to UI break business logic and vice versa.

**Solution**: Split into three components:
- **Model**: Data structures (domain models)
- **View**: UI components (Flutter widgets)
- **ViewModel**: Presentation logic and state management

**Analogy**: Think of a restaurant:
- **Model**: The actual food (data)
- **ViewModel**: The waiter (manages orders, brings food to tables)
- **View**: The dining room presentation (how food is arranged on plates, table settings)

---

### MVVM Components

```
┌─────────────────────────────────────────┐
│                  VIEW                    │
│         (Flutter Widgets)                │
│                                          │
│  • Displays data                         │
│  • Handles user input                    │
│  • Triggers ViewModel commands           │
│  • Listens to ViewModel changes          │
└──────────────┬───────────────────────────┘
               │
               │ Observes (ListenableBuilder)
               │ Invokes commands
               │
┌──────────────▼───────────────────────────┐
│              VIEWMODEL                   │
│      (ChangeNotifier)                    │
│                                          │
│  • Holds UI state                        │
│  • Exposes commands                      │
│  • Contains presentation logic           │
│  • Notifies View of changes              │
└──────────────┬───────────────────────────┘
               │
               │ Uses
               │
┌──────────────▼───────────────────────────┐
│               MODEL                      │
│         (Domain Entities)                │
│                                          │
│  • Booking, Destination, Activity        │
│  • Business rules                        │
│  • Independent of UI                     │
└──────────────────────────────────────────┘
```

---

### Implementation: Home Feature

#### Model

**File**: [app/lib/domain/models/booking/booking_summary.dart](../app/lib/domain/models/booking/booking_summary.dart)

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

**Responsibility**: Represent booking data structure. No logic, just data.

---

#### ViewModel

**File**: [app/lib/ui/home/view_models/home_viewmodel.dart](../app/lib/ui/home/view_models/home_viewmodel.dart)

```dart
class HomeViewModel extends ChangeNotifier {
  final BookingRepository _bookingRepository;
  final UserRepository _userRepository;
  final Logger _log = Logger('HomeViewModel');

  HomeViewModel({
    required BookingRepository bookingRepository,
    required UserRepository userRepository,
  })  : _bookingRepository = bookingRepository,
        _userRepository = userRepository {
    // Initialize commands
    load = Command0(_load)..execute(); // Auto-execute on creation
    deleteBooking = Command1(_deleteBooking);
  }

  // Commands (user-triggered actions)
  late Command0 load;
  late Command1<void, int> deleteBooking;

  // Private state
  List<BookingSummary> _bookings = [];
  User? _user;

  // Public getters (View reads these)
  List<BookingSummary> get bookings => _bookings;
  User? get user => _user;

  /// Load bookings and user profile
  Future<Result<void>> _load() async {
    _log.fine('Loading home screen data');

    // Parallel loading for performance
    final results = await (
      _bookingRepository.getBookingsList(),
      _userRepository.getUser(),
    ).wait;

    // Handle bookings result
    switch (results.$1) {
      case Ok<List<BookingSummary>>():
        _bookings = results.$1.value;
        _log.info('Loaded ${_bookings.length} bookings');
      case Error<List<BookingSummary>>():
        _log.warning('Failed to load bookings', results.$1.error);
    }

    // Handle user result
    switch (results.$2) {
      case Ok<User>():
        _user = results.$2.value;
        _log.info('Loaded user: ${results.$2.value.name}');
      case Error<User>():
        _log.warning('Failed to load user', results.$2.error);
    }

    notifyListeners(); // Tell View to rebuild
    return Result.ok(null);
  }

  /// Delete a booking
  Future<Result<void>> _deleteBooking(int id) async {
    _log.fine('Deleting booking $id');

    final result = await _bookingRepository.delete(id);

    switch (result) {
      case Ok():
        // Update local state
        _bookings.removeWhere((b) => b.id == id);
        _log.info('Deleted booking $id');
        notifyListeners(); // Tell View to rebuild
      case Error():
        _log.warning('Failed to delete booking $id', result.error);
    }

    return result;
  }
}
```

**Responsibilities**:
- ✅ Hold UI state (`_bookings`, `_user`)
- ✅ Expose commands (`load`, `deleteBooking`)
- ✅ Coordinate repositories
- ✅ Notify View of changes
- ❌ **NO** UI code (no widgets, no `BuildContext`)
- ❌ **NO** platform-specific code

---

#### View

**File**: [app/lib/ui/home/widgets/home_screen.dart](../app/lib/ui/home/widgets/home_screen.dart)

```dart
class HomeScreen extends StatefulWidget {
  final HomeViewModel viewModel;

  const HomeScreen({
    super.key,
    required this.viewModel,
  });

  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('My Trips'),
        actions: [
          // User profile (reactive to ViewModel changes)
          ListenableBuilder(
            listenable: widget.viewModel,
            builder: (context, _) {
              final user = widget.viewModel.user;
              if (user == null) return const SizedBox.shrink();

              return Padding(
                padding: const EdgeInsets.all(8.0),
                child: CircleAvatar(
                  backgroundImage: AssetImage(user.picture),
                ),
              );
            },
          ),
          const LogoutButton(),
        ],
      ),
      body: ListenableBuilder(
        listenable: widget.viewModel.load, // Listen to load command
        builder: (context, _) {
          // Loading state
          if (widget.viewModel.load.running) {
            return const Center(
              child: CircularProgressIndicator(),
            );
          }

          // Error state
          if (widget.viewModel.load.error != null) {
            return ErrorIndicator(
              title: 'Failed to load bookings',
              label: widget.viewModel.load.error.toString(),
              onPressed: widget.viewModel.load.execute,
            );
          }

          // Empty state
          if (widget.viewModel.bookings.isEmpty) {
            return Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Icon(Icons.luggage, size: 64, color: Colors.grey),
                  SizedBox(height: 16),
                  Text(
                    'No trips yet!',
                    style: Theme.of(context).textTheme.headlineSmall,
                  ),
                  SizedBox(height: 8),
                  Text('Start planning your adventure'),
                ],
              ),
            );
          }

          // Success state - show bookings
          return ListView.builder(
            itemCount: widget.viewModel.bookings.length,
            itemBuilder: (context, index) {
              final booking = widget.viewModel.bookings[index];
              return BookingListTile(
                booking: booking,
                onTap: () => _viewBooking(booking.id),
                onDelete: () => _confirmDelete(booking),
              );
            },
          );
        },
      ),
      floatingActionButton: FloatingActionButton.extended(
        onPressed: () => context.go(Routes.search),
        icon: const Icon(Icons.add),
        label: const Text('New Trip'),
      ),
    );
  }

  void _viewBooking(int id) {
    context.push(Routes.bookingWithId(id));
  }

  Future<void> _confirmDelete(BookingSummary booking) async {
    final confirmed = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Delete Trip?'),
        content: Text('Are you sure you want to delete ${booking.name}?'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('Cancel'),
          ),
          TextButton(
            onPressed: () => Navigator.pop(context, true),
            child: const Text('Delete'),
          ),
        ],
      ),
    );

    if (confirmed == true) {
      await widget.viewModel.deleteBooking.execute(booking.id);

      if (widget.viewModel.deleteBooking.completed && mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Deleted ${booking.name}')),
        );
      }
    }
  }
}
```

**Responsibilities**:
- ✅ Render UI based on ViewModel state
- ✅ Handle user interactions
- ✅ Trigger ViewModel commands
- ✅ Show dialogs and navigation
- ❌ **NO** business logic
- ❌ **NO** direct repository access

---

### Data Flow

**Loading Data**:
```
1. View created
   ↓
2. ViewModel created (in router)
   ↓
3. ViewModel.load command auto-executes
   ↓
4. ViewModel calls repositories
   ↓
5. ViewModel updates state (_bookings, _user)
   ↓
6. ViewModel calls notifyListeners()
   ↓
7. ListenableBuilder rebuilds
   ↓
8. View displays data
```

**User Action** (Delete booking):
```
1. User taps delete button
   ↓
2. View shows confirmation dialog
   ↓
3. User confirms
   ↓
4. View calls viewModel.deleteBooking.execute(id)
   ↓
5. ViewModel calls repository.delete(id)
   ↓
6. ViewModel updates _bookings (remove deleted)
   ↓
7. ViewModel calls notifyListeners()
   ↓
8. ListenableBuilder rebuilds
   ↓
9. View shows updated list
   ↓
10. View shows success SnackBar
```

---

### Benefits

1. **Testability**: ViewModel can be tested without UI
   ```dart
   test('HomeViewModel loads bookings', () async {
     final viewModel = HomeViewModel(
       bookingRepository: mockBookingRepo,
       userRepository: mockUserRepo,
     );

     await viewModel.load.execute();

     expect(viewModel.bookings.length, 2);
     expect(viewModel.user?.name, 'Test User');
   });
   ```

2. **Separation of Concerns**: Business logic separate from UI code

3. **Reusability**: Same ViewModel could be used in different Views (mobile vs tablet layout)

4. **Maintainability**: Changes to business logic don't affect UI structure

---

### MVVM vs Other Patterns

| Pattern | ViewModel | State Updates | Learning Curve |
|---------|-----------|---------------|----------------|
| **MVVM** | ChangeNotifier | notifyListeners() | Low |
| **BLoC** | Bloc | Events → States | Medium |
| **Redux** | Store | Actions → Reducers | High |
| **Riverpod** | Provider | StateNotifier | Medium |

**Compass App uses MVVM because**:
- ✅ Simpler than BLoC for app's complexity level
- ✅ Well-supported by Flutter (ChangeNotifier built-in)
- ✅ Good balance of structure and simplicity

---

## Command Pattern

### Overview

**Purpose**: Encapsulate async operations with automatic state management (running, error, result).

**Problem Solved**: Managing loading states, errors, and preventing duplicate operations is repetitive and error-prone when done manually in every ViewModel method.

**Solution**: Wrap async operations in Command objects that automatically track state.

**Analogy**: Think of ordering food at a restaurant:
- **Command**: Your order ticket
- **Running**: Kitchen is preparing your order
- **Completed**: Food is ready
- **Error**: Kitchen ran out of ingredients
- **Prevent Duplicates**: You can't order the same dish twice while it's being prepared

---

### Implementation

**File**: [app/lib/utils/command.dart](../app/lib/utils/command.dart)

#### Base Command Class

```dart
/// Base class for all commands
abstract class Command<T> extends ChangeNotifier {
  bool _running = false;
  Exception? _error;
  T? _result;

  /// Is the command currently executing?
  bool get running => _running;

  /// Did the command fail? Contains the exception if so.
  Exception? get error => _error;

  /// Result of the command (if completed successfully)
  T? get result => _result;

  /// Did the command complete successfully?
  bool get completed => !_running && _error == null && _result != null;

  /// Clear any error state
  void clearError() {
    _error = null;
    notifyListeners();
  }
}
```

---

#### Command0 (No Arguments)

```dart
/// Command with no arguments
class Command0<T> extends Command<T> {
  final Future<Result<T>> Function() _action;

  Command0(this._action);

  /// Execute the command
  Future<void> execute() async {
    // Prevent duplicate execution
    if (_running) {
      return;
    }

    // Set running state
    _running = true;
    _error = null;
    _result = null;
    notifyListeners();

    try {
      // Execute the action
      final result = await _action();

      // Handle result
      switch (result) {
        case Ok<T>(:final value):
          _result = value;
        case Error<T>(:final error):
          _error = error;
      }
    } on Exception catch (e) {
      // Catch any exceptions not wrapped in Result
      _error = e;
    } finally {
      // Clear running state
      _running = false;
      notifyListeners();
    }
  }
}
```

**Usage**:
```dart
// In ViewModel
late Command0 load;

HomeViewModel() {
  load = Command0(_load)..execute(); // Create and auto-execute
}

Future<Result<void>> _load() async {
  // Load data
  final result = await repository.getData();
  return result;
}

// In View
ListenableBuilder(
  listenable: viewModel.load,
  builder: (context, _) {
    if (viewModel.load.running) {
      return CircularProgressIndicator();
    }
    if (viewModel.load.error != null) {
      return ErrorIndicator(error: viewModel.load.error);
    }
    // Show data
  },
)
```

---

#### Command1 (One Argument)

```dart
/// Command with one argument
class Command1<T, A> extends Command<T> {
  final Future<Result<T>> Function(A) _action;

  Command1(this._action);

  /// Execute the command with an argument
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

**Usage**:
```dart
// In ViewModel
late Command1<void, int> deleteBooking;

HomeViewModel() {
  deleteBooking = Command1(_deleteBooking);
}

Future<Result<void>> _deleteBooking(int id) async {
  return await repository.delete(id);
}

// In View
ElevatedButton(
  onPressed: () => viewModel.deleteBooking.execute(bookingId),
  child: viewModel.deleteBooking.running
      ? CircularProgressIndicator()
      : Text('Delete'),
)
```

---

### Real-World Example: Delete Booking

**Without Command Pattern** (verbose, error-prone):

```dart
class HomeViewModel extends ChangeNotifier {
  bool _deleting = false;
  String? _deleteError;

  bool get deleting => _deleting;
  String? get deleteError => _deleteError;

  Future<void> deleteBooking(int id) async {
    // Manual duplicate prevention
    if (_deleting) return;

    // Manual state management
    _deleting = true;
    _deleteError = null;
    notifyListeners();

    try {
      final result = await _bookingRepository.delete(id);

      switch (result) {
        case Ok():
          _bookings.removeWhere((b) => b.id == id);
        case Error(:final error):
          _deleteError = error.toString();
      }
    } catch (e) {
      _deleteError = e.toString();
    } finally {
      _deleting = false;
      notifyListeners();
    }
  }
}

// In View - manual state handling
if (viewModel.deleting) {
  return CircularProgressIndicator();
}
if (viewModel.deleteError != null) {
  return Text('Error: ${viewModel.deleteError}');
}
```

**With Command Pattern** (clean, automatic):

```dart
class HomeViewModel extends ChangeNotifier {
  late Command1<void, int> deleteBooking;

  HomeViewModel() {
    deleteBooking = Command1(_deleteBooking);
  }

  Future<Result<void>> _deleteBooking(int id) async {
    final result = await _bookingRepository.delete(id);

    if (result is Ok) {
      _bookings.removeWhere((b) => b.id == id);
      notifyListeners();
    }

    return result;
  }
}

// In View - automatic state handling
ListenableBuilder(
  listenable: viewModel.deleteBooking,
  builder: (context, _) {
    if (viewModel.deleteBooking.running) {
      return CircularProgressIndicator();
    }
    if (viewModel.deleteBooking.error != null) {
      return Text('Error: ${viewModel.deleteBooking.error}');
    }
    return ElevatedButton(
      onPressed: () => viewModel.deleteBooking.execute(id),
      child: Text('Delete'),
    );
  },
)
```

**Lines of code saved**: ~15 lines per async operation!

---

### Benefits

1. **Automatic State Management**: `running`, `error`, `result` tracked automatically
2. **Duplicate Prevention**: Can't execute while already running
3. **Consistent UI**: Same pattern for all async operations
4. **Less Boilerplate**: No manual state flags
5. **Reactive**: Extends ChangeNotifier for automatic UI updates

---

### Command Patterns in Compass App

| ViewModel | Command0 (no args) | Command1 (one arg) |
|-----------|--------------------|--------------------|
| **HomeViewModel** | `load` | `deleteBooking(id)` |
| **SearchFormViewModel** | `load`, `updateItineraryConfig` | - |
| **ResultsViewModel** | `search`, `updateItineraryConfig` | - |
| **ActivitiesViewModel** | `loadActivities`, `saveActivities` | - |
| **BookingViewModel** | `load`, `createBooking`, `shareBooking` | - |
| **LoginViewModel** | `login` | - |

---

## Result Pattern

### Overview

**Purpose**: Type-safe error handling without exceptions.

**Problem Solved**: Traditional exception handling has issues:
- Easy to forget try-catch blocks
- No compile-time enforcement
- Exceptions can be thrown from anywhere
- Stack traces often unhelpful

**Solution**: Return a `Result<T>` type that explicitly represents success or failure.

**Analogy**: Think of a delivery service:
- **Traditional exceptions**: Package might arrive, might get lost, you don't know until you check
- **Result pattern**: Delivery always returns a receipt: either "Package Delivered" or "Delivery Failed: [reason]"

---

### Implementation

**File**: [app/lib/utils/result.dart](../app/lib/utils/result.dart)

```dart
/// Represents the result of an operation that can succeed or fail
sealed class Result<T> {
  const Result();

  /// Create a successful result
  const factory Result.ok(T value) = Ok<T>._;

  /// Create a failed result
  const factory Result.error(Exception error) = Error<T>._;

  /// Pattern match on the result
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

/// Successful result containing a value
final class Ok<T> extends Result<T> {
  final T value;
  const Ok._(this.value);
}

/// Failed result containing an error
final class Error<T> extends Result<T> {
  final Exception error;
  const Error._(this.error);
}

/// Extension methods for convenience
extension ResultExtension<T> on Result<T> {
  /// Cast to Ok (unsafe - use with switch statement)
  Ok<T> get asOk => this as Ok<T>;

  /// Cast to Error (unsafe - use with switch statement)
  Error<T> get asError => this as Error<T>;

  /// Get value or null
  T? get valueOrNull => switch (this) {
    Ok<T>(:final value) => value,
    Error<T>() => null,
  };

  /// Get error or null
  Exception? get errorOrNull => switch (this) {
    Ok<T>() => null,
    Error<T>(:final error) => error,
  };

  /// Is this a successful result?
  bool get isOk => this is Ok<T>;

  /// Is this a failed result?
  bool get isError => this is Error<T>;
}
```

---

### Usage Patterns

#### Pattern 1: Switch Expression (Recommended)

```dart
final result = await repository.getBooking(id);

switch (result) {
  case Ok<Booking>(:final value):
    // Success path - compiler enforces value extraction
    _booking = value;
    print('Loaded booking: ${value.destination.name}');
    notifyListeners();

  case Error<Booking>(:final error):
    // Error path - compiler enforces error handling
    _error = error.toString();
    print('Failed to load booking: $error');
    notifyListeners();
}
```

**Benefits**:
- ✅ Compiler ensures both cases are handled
- ✅ Type-safe value extraction
- ✅ Clear success/error paths

---

#### Pattern 2: When Method

```dart
final result = await repository.getBooking(id);

final message = result.when(
  ok: (booking) => 'Loaded ${booking.destination.name}',
  error: (error) => 'Failed: $error',
);

print(message);
```

**Benefits**:
- ✅ Functional style
- ✅ Returns a value from both branches
- ✅ Concise for simple transformations

---

#### Pattern 3: Value or Null

```dart
final result = await repository.getBooking(id);
final booking = result.valueOrNull;

if (booking != null) {
  print('Loaded: ${booking.destination.name}');
} else {
  print('Failed to load');
}
```

**Use Case**: When you don't need detailed error information.

---

### Real-World Example: Booking Creation

**File**: [app/lib/domain/use_cases/booking/booking_create_use_case.dart](../app/lib/domain/use_cases/booking/booking_create_use_case.dart)

```dart
class BookingCreateUseCase {
  final DestinationRepository _destinationRepository;
  final ActivityRepository _activityRepository;
  final BookingRepository _bookingRepository;

  BookingCreateUseCase({
    required DestinationRepository destinationRepository,
    required ActivityRepository activityRepository,
    required BookingRepository bookingRepository,
  })  : _destinationRepository = destinationRepository,
        _activityRepository = activityRepository,
        _bookingRepository = bookingRepository;

  /// Create a booking from itinerary config
  Future<Result<Booking>> createFrom(ItineraryConfig config) async {
    // Validate config
    if (config.destination == null) {
      return Result.error(Exception('Destination not selected'));
    }
    if (config.startDate == null || config.endDate == null) {
      return Result.error(Exception('Dates not selected'));
    }

    // Fetch destination
    final destinationResult = await _fetchDestination(config.destination!);
    switch (destinationResult) {
      case Ok<Destination>(:final value):
        final destination = value;

      case Error<Destination>(:final error):
        return Result.error(error); // Propagate error
    }

    // Fetch activities
    final activitiesResult = await _activityRepository.getByDestination(
      config.destination!,
    );

    switch (activitiesResult) {
      case Ok<List<Activity>>(:final value):
        final allActivities = value;

        // Filter to selected activities
        final selectedActivities = allActivities
            .where((a) => config.activities.contains(a.ref))
            .toList();

        // Create booking
        final booking = Booking(
          startDate: config.startDate!,
          endDate: config.endDate!,
          destination: destinationResult.asOk.value,
          activity: selectedActivities,
        );

        // Persist booking
        final createResult = await _bookingRepository.createBooking(booking);

        return createResult.when(
          ok: (_) => Result.ok(booking),
          error: (error) => Result.error(error),
        );

      case Error<List<Activity>>(:final error):
        return Result.error(error);
    }
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

**Error Propagation Chain**:
```
1. ItineraryConfig validation fails
   → Result.error('Destination not selected')

2. Destination fetch fails
   → Result.error(API error)
   → Propagated to caller

3. Activity fetch fails
   → Result.error(API error)
   → Propagated to caller

4. Booking creation fails
   → Result.error(API error)
   → Propagated to caller

5. All succeed
   → Result.ok(booking)
```

---

### Comparison with Exceptions

**Traditional Exception Handling**:

```dart
// Easy to forget try-catch
Future<Booking> getBooking(int id) async {
  final response = await http.get('/booking/$id');

  if (response.statusCode != 200) {
    throw Exception('Failed to load'); // Might be uncaught!
  }

  return Booking.fromJson(response.data);
}

// Caller might forget error handling
final booking = await repository.getBooking(id); // 💣 Might throw!
print(booking.destination.name);
```

**Result Pattern**:

```dart
// Error handling is part of return type
Future<Result<Booking>> getBooking(int id) async {
  final response = await http.get('/booking/$id');

  if (response.statusCode != 200) {
    return Result.error(Exception('Failed to load'));
  }

  return Result.ok(Booking.fromJson(response.data));
}

// Compiler forces error handling
final result = await repository.getBooking(id);
switch (result) {
  case Ok<Booking>(:final value):
    print(value.destination.name); // ✅ Safe
  case Error<Booking>(:final error):
    print('Error: $error'); // ✅ Required
}
```

---

### Benefits

1. **Compile-Time Safety**: Compiler enforces error handling
2. **Explicit**: Method signature shows it can fail
3. **No Surprises**: No hidden exceptions
4. **Composable**: Easy to chain operations
5. **Testable**: Easy to test both success and error paths

---

### When to Use

✅ **Use Result Pattern for**:
- Repository methods
- Use cases
- Any operation that can fail
- API calls
- File operations

❌ **Don't use Result for**:
- Programming errors (use assertions or regular exceptions)
- Truly exceptional situations (system crashes)
- Local operations that can't fail

---

## Dependency Injection

### Overview

**Purpose**: Provide dependencies to components instead of having them create their own.

**Problem Solved**: When classes create their own dependencies:
- Hard to test (can't inject mocks)
- Tight coupling (class knows concrete types)
- Difficult to change implementations

**Solution**: Dependencies are "injected" from outside, typically through constructors.

**Analogy**: Think of a chef in a restaurant:
- **Without DI**: Chef goes to the market, buys ingredients, carries them back (creates dependencies)
- **With DI**: Ingredients are delivered to the kitchen (injected)

The chef focuses on cooking, not sourcing ingredients!

---

### Implementation in Compass App

**Package Used**: `provider` (^6.1.2)

**Configuration File**: [app/lib/config/dependencies.dart](../app/lib/config/dependencies.dart)

---

#### Local Environment Configuration

```dart
/// Dependencies for local development (no server required)
List<SingleChildWidget> get providersLocal {
  return [
    // === DATA SERVICES ===

    // Local data service (loads JSON assets)
    Provider(
      create: (context) => LocalDataService(),
    ),

    // === REPOSITORIES ===

    // Continent repository
    Provider(
      create: (context) => ContinentRepositoryLocal(
        localDataService: context.read<LocalDataService>(),
      ) as ContinentRepository, // ← Return as abstract type
    ),

    // Destination repository
    Provider(
      create: (context) => DestinationRepositoryLocal(
        localDataService: context.read<LocalDataService>(),
      ) as DestinationRepository,
    ),

    // Activity repository
    Provider(
      create: (context) => ActivityRepositoryLocal(
        localDataService: context.read<LocalDataService>(),
      ) as ActivityRepository,
    ),

    // Booking repository
    Provider(
      create: (context) => BookingRepositoryLocal(
        localDataService: context.read<LocalDataService>(),
      ) as BookingRepository,
    ),

    // Auth repository (always authenticated in dev)
    ChangeNotifierProvider(
      create: (context) => AuthRepositoryDev() as AuthRepository,
    ),

    // User repository
    Provider(
      create: (context) => UserRepositoryLocal(
        localDataService: context.read<LocalDataService>(),
      ) as UserRepository,
    ),

    // Itinerary config repository (in-memory)
    Provider(
      create: (context) => ItineraryConfigRepositoryMemory()
          as ItineraryConfigRepository,
    ),

    // === USE CASES ===

    // Booking create use case
    Provider(
      create: (context) => BookingCreateUseCase(
        destinationRepository: context.read<DestinationRepository>(),
        activityRepository: context.read<ActivityRepository>(),
        bookingRepository: context.read<BookingRepository>(),
      ),
    ),

    // Booking share use case
    Provider(
      create: (context) => BookingShareUseCase.withSharePlus(),
    ),
  ];
}
```

**Key Points**:
- ✅ Dependencies resolved via `context.read<T>()`
- ✅ Repositories returned as abstract types
- ✅ Use cases inject repository abstractions
- ✅ Order matters (dependencies must be registered before consumers)

---

#### Remote Environment Configuration

```dart
/// Dependencies for remote/staging (requires backend server)
List<SingleChildWidget> get providersRemote {
  return [
    // === HTTP CLIENT ===

    Provider(
      create: (context) => http.Client(),
    ),

    // === SHARED PREFERENCES ===

    Provider(
      create: (context) => SharedPreferencesService(),
    ),

    // === AUTH API CLIENT ===

    Provider(
      create: (context) => AuthApiClient(
        baseUrl: 'http://localhost:8080',
        client: context.read<http.Client>(),
      ),
    ),

    // === AUTH REPOSITORY ===

    ChangeNotifierProvider(
      create: (context) => AuthRepositoryRemote(
        authApiClient: context.read<AuthApiClient>(),
        sharedPreferencesService: context.read<SharedPreferencesService>(),
      ) as AuthRepository, // Same abstract type as dev!
    ),

    // === AUTH HEADER PROVIDER ===

    Provider(
      create: (context) => AuthHeaderProvider(
        sharedPreferencesService: context.read<SharedPreferencesService>(),
      ),
    ),

    // === API CLIENT ===

    Provider(
      create: (context) => ApiClient(
        baseUrl: 'http://localhost:8080',
        client: context.read<http.Client>(),
        authHeaderProvider: context.read<AuthHeaderProvider>(),
      ),
    ),

    // === REMOTE REPOSITORIES ===

    Provider(
      create: (context) => ContinentRepositoryRemote(
        apiClient: context.read<ApiClient>(),
      ) as ContinentRepository, // Same abstract type as local!
    ),

    Provider(
      create: (context) => DestinationRepositoryRemote(
        apiClient: context.read<ApiClient>(),
      ) as DestinationRepository,
    ),

    Provider(
      create: (context) => ActivityRepositoryRemote(
        apiClient: context.read<ApiClient>(),
      ) as ActivityRepository,
    ),

    Provider(
      create: (context) => BookingRepositoryRemote(
        apiClient: context.read<ApiClient>(),
      ) as BookingRepository,
    ),

    Provider(
      create: (context) => UserRepositoryRemote(
        apiClient: context.read<ApiClient>(),
      ) as UserRepository,
    ),

    Provider(
      create: (context) => ItineraryConfigRepositoryMemory()
          as ItineraryConfigRepository,
    ),

    // === USE CASES (same as local) ===

    Provider(
      create: (context) => BookingCreateUseCase(
        destinationRepository: context.read<DestinationRepository>(),
        activityRepository: context.read<ActivityRepository>(),
        bookingRepository: context.read<BookingRepository>(),
      ),
    ),

    Provider(
      create: (context) => BookingShareUseCase.withSharePlus(),
    ),
  ];
}
```

**Key Differences from Local**:
- ❌ No `LocalDataService`
- ✅ Has `http.Client`, `ApiClient`, `AuthApiClient`
- ✅ Has `SharedPreferencesService` for token storage
- ✅ Has `AuthHeaderProvider` for authenticated requests
- ✅ Remote repository implementations
- ✅ Same abstract types as local!

---

### Using Dependencies

#### In Main Entry Point

**Development** ([main_development.dart](../app/lib/main_development.dart)):
```dart
void main() {
  Logger.root.level = Level.WARNING;
  Logger.root.onRecord.listen((record) => print(record.message));

  runApp(
    MultiProvider(
      providers: providersLocal, // ← Local dependencies
      child: const MainApp(),
    ),
  );
}
```

**Staging** ([main_staging.dart](../app/lib/main_staging.dart)):
```dart
void main() {
  Logger.root.level = Level.ALL;
  Logger.root.onRecord.listen((record) => print(record.message));

  runApp(
    MultiProvider(
      providers: providersRemote, // ← Remote dependencies
      child: const MainApp(),
    ),
  );
}
```

---

#### In Router (Creating ViewModels)

**File**: [app/lib/routing/router.dart](../app/lib/routing/router.dart)

```dart
GoRoute(
  path: Routes.home,
  builder: (context, state) {
    // Create ViewModel with injected dependencies
    return HomeScreen(
      viewModel: HomeViewModel(
        bookingRepository: context.read<BookingRepository>(), // Injected!
        userRepository: context.read<UserRepository>(),       // Injected!
      ),
    );
  },
),
```

**Key Points**:
- `context.read<BookingRepository>()` gets the registered implementation
- In dev environment: Returns `BookingRepositoryLocal`
- In staging environment: Returns `BookingRepositoryRemote`
- ViewModel doesn't know which implementation it gets!

---

### Benefits

1. **Testability**: Easy to inject mocks
   ```dart
   test('HomeViewModel loads bookings', () {
     final mockRepo = MockBookingRepository();
     final viewModel = HomeViewModel(
       bookingRepository: mockRepo, // Injected mock
       userRepository: mockUserRepo,
     );
     // Test ViewModel with mock
   });
   ```

2. **Flexibility**: Swap implementations by changing config
   ```dart
   // Change one line to use different environment
   providers: providersLocal  // or providersRemote
   ```

3. **Loose Coupling**: ViewModels depend on abstractions, not concretions
   ```dart
   // ViewModel depends on interface
   final BookingRepository _repository; // Abstract type

   // Not on implementation
   final BookingRepositoryLocal _repository; // ❌ Tight coupling
   ```

4. **Single Source of Truth**: All dependencies configured in one place

---

### Provider Types

| Provider Type | Use Case | Example |
|---------------|----------|---------|
| **Provider** | Immutable services, stateless | Repositories, Use Cases |
| **ChangeNotifierProvider** | Mutable state, notifies listeners | AuthRepository |
| **FutureProvider** | Async initialization | (Not used in Compass App) |
| **StreamProvider** | Realtime data streams | (Not used in Compass App) |

---

## Factory Pattern

### Overview

**Purpose**: Provide an interface for creating objects without specifying their exact class.

**Problem Solved**: When you need to create objects but don't want to hardcode the class name, or when object creation logic is complex.

**Solution**: Use factory methods or classes to encapsulate object creation.

---

### Implementation Examples

#### Factory Method in Use Case

**File**: [app/lib/domain/use_cases/booking/booking_share_use_case.dart](../app/lib/domain/use_cases/booking/booking_share_use_case.dart)

```dart
typedef ShareFunction = Future<void> Function(String text);

class BookingShareUseCase {
  final ShareFunction _share;

  // Private constructor
  BookingShareUseCase._(this._share);

  /// Factory for production (uses share_plus package)
  factory BookingShareUseCase.withSharePlus() {
    return BookingShareUseCase._(Share.share);
  }

  /// Factory for testing (custom share function)
  factory BookingShareUseCase.custom(ShareFunction share) {
    return BookingShareUseCase._(share);
  }

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
    buffer.writeln('Check out my trip to ${booking.destination.name}!');
    buffer.writeln();
    buffer.writeln('Dates: ${_formatDate(booking.startDate)} - '
        '${_formatDate(booking.endDate)}');
    buffer.writeln();
    buffer.writeln('Activities:');
    for (final activity in booking.activity) {
      buffer.writeln('  • ${activity.name}');
    }
    return buffer.toString();
  }

  String _formatDate(DateTime date) {
    return DateFormat.yMMMd().format(date);
  }
}
```

**Usage**:

**Production**:
```dart
Provider(
  create: (context) => BookingShareUseCase.withSharePlus(),
)
```

**Testing**:
```dart
test('BookingShareUseCase formats booking correctly', () async {
  String? sharedText;

  final useCase = BookingShareUseCase.custom((text) async {
    sharedText = text; // Capture shared text
  });

  await useCase.shareBooking(testBooking);

  expect(sharedText, contains('Paris'));
  expect(sharedText, contains('Eiffel Tower'));
});
```

**Benefits**:
- ✅ Production code uses real share dialog
- ✅ Tests use mock that captures shared text
- ✅ Same interface, different implementations

---

#### Factory via Dependency Injection

**Implicit Factory Pattern**: The dependency injection configuration acts as a factory:

```dart
// Factory for local environment
List<SingleChildWidget> get providersLocal {
  return [
    Provider(
      create: (context) => BookingRepositoryLocal(...) as BookingRepository,
      //      ^^^^^^ Factory method
    ),
  ];
}

// Factory for remote environment
List<SingleChildWidget> get providersRemote {
  return [
    Provider(
      create: (context) => BookingRepositoryRemote(...) as BookingRepository,
      //      ^^^^^^ Factory method (different implementation!)
    ),
  ];
}
```

**Usage** (same code works with both factories):
```dart
final repository = context.read<BookingRepository>();
// Gets BookingRepositoryLocal in dev
// Gets BookingRepositoryRemote in staging
```

---

## Strategy Pattern

### Overview

**Purpose**: Define a family of algorithms, encapsulate each one, and make them interchangeable.

**Problem Solved**: When you have multiple ways to perform an operation and want to switch between them at runtime.

**Solution**: Define an interface for the operation, create multiple implementations, and select one at runtime.

---

### Implementation in Compass App

**The Repository Pattern IS a Strategy Pattern!**

Each repository interface defines a strategy for data access:

```dart
// Strategy interface
abstract class BookingRepository {
  Future<Result<List<BookingSummary>>> getBookingsList();
  // ...
}

// Strategy 1: In-memory storage
class BookingRepositoryLocal implements BookingRepository {
  @override
  Future<Result<List<BookingSummary>>> getBookingsList() async {
    return Result.ok(_bookings.map((b) => /* convert to summary */).toList());
  }
}

// Strategy 2: API storage
class BookingRepositoryRemote implements BookingRepository {
  @override
  Future<Result<List<BookingSummary>>> getBookingsList() async {
    return await _apiClient.getBookings();
  }
}

// Context (uses strategy)
class HomeViewModel {
  final BookingRepository _repository; // Strategy interface

  Future<Result<void>> _load() async {
    // Uses strategy without knowing which implementation
    final result = await _repository.getBookingsList();
    // ...
  }
}
```

**Strategy Selection** (via dependency injection):

```dart
// Select strategy at app startup
runApp(
  MultiProvider(
    providers: USE_LOCAL_DATA ? providersLocal : providersRemote,
    //                           ^^^^^^^^^^^^^^   ^^^^^^^^^^^^^^
    //                           Strategy 1       Strategy 2
    child: MainApp(),
  ),
);
```

---

### Benefits

1. **Runtime Flexibility**: Change strategy without code changes
2. **Open/Closed Principle**: Add new strategies without modifying existing code
3. **Testability**: Easy to test each strategy independently

---

## Observer Pattern

### Overview

**Purpose**: Define a one-to-many dependency so that when one object changes state, all dependents are notified.

**Problem Solved**: How to update UI when data changes without tight coupling.

**Solution**: Use ChangeNotifier and ListenableBuilder.

---

### Implementation

**Observable (Subject)**: ViewModels extend `ChangeNotifier`

```dart
class HomeViewModel extends ChangeNotifier {
  //                   ^^^^^^^^^^^^^^^^ Observable

  List<BookingSummary> _bookings = [];

  List<BookingSummary> get bookings => _bookings;

  Future<Result<void>> _load() async {
    final result = await _repository.getBookingsList();

    switch (result) {
      case Ok():
        _bookings = result.value;
        notifyListeners(); // ← Notify all observers!
      case Error():
        // Handle error
    }
  }
}
```

**Observer (Listener)**: Widgets use `ListenableBuilder`

```dart
class HomeScreen extends StatefulWidget {
  final HomeViewModel viewModel;
  // ...
}

class _HomeScreenState extends State<HomeScreen> {
  @override
  Widget build(BuildContext context) {
    return ListenableBuilder(
      listenable: widget.viewModel, // ← Observe this object
      builder: (context, _) {
        // This rebuilds when viewModel.notifyListeners() is called
        return ListView.builder(
          itemCount: widget.viewModel.bookings.length,
          itemBuilder: (context, index) {
            return BookingCard(booking: widget.viewModel.bookings[index]);
          },
        );
      },
    );
  }
}
```

---

### Flow

```
1. User triggers action (e.g., delete booking)
   ↓
2. View calls ViewModel method
   viewModel.deleteBooking.execute(id)
   ↓
3. ViewModel updates state
   _bookings.removeWhere((b) => b.id == id)
   ↓
4. ViewModel notifies listeners
   notifyListeners()
   ↓
5. All ListenableBuilders observing this ViewModel rebuild
   ↓
6. UI shows updated data
```

---

### Multiple Observers

One ViewModel can have multiple observers:

```dart
// Observer 1: Booking list
ListenableBuilder(
  listenable: viewModel,
  builder: (context, _) => BookingList(bookings: viewModel.bookings),
)

// Observer 2: Booking count
ListenableBuilder(
  listenable: viewModel,
  builder: (context, _) => Text('${viewModel.bookings.length} trips'),
)

// Both rebuild when viewModel.notifyListeners() is called!
```

---

## Builder Pattern

### Overview

**Purpose**: Separate object construction from its representation, allowing the same construction process to create different representations.

**Problem Solved**: Constructors with many parameters become unwieldy. Immutable objects need a way to create modified copies.

**Solution**: Use Freezed to generate builders and `copyWith` methods.

---

### Implementation with Freezed

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
  }) = _Booking;

  factory Booking.fromJson(Map<String, dynamic> json) =>
      _$BookingFromJson(json);
}
```

**Generated Code** (by Freezed):

```dart
// Constructor
Booking booking = Booking(
  startDate: DateTime(2025, 6, 1),
  endDate: DateTime(2025, 6, 10),
  destination: parisDestination,
  activity: selectedActivities,
);

// copyWith (builder method)
Booking extendedBooking = booking.copyWith(
  endDate: DateTime(2025, 6, 15), // Only change end date
  // All other properties copied automatically
);

// Equality and hashCode
print(booking1 == booking2); // Uses generated == operator

// toString
print(booking); // Readable debug output
```

---

### Benefits

1. **Immutability**: All properties are final
2. **Easy Copying**: `copyWith` method for creating modified copies
3. **Equality**: Automatic `==` and `hashCode`
4. **JSON Serialization**: Built-in `toJson` and `fromJson`
5. **Type Safety**: All properties are type-checked

---

### Real-World Usage

**Creating a new booking**:
```dart
final booking = Booking(
  startDate: config.startDate!,
  endDate: config.endDate!,
  destination: selectedDestination,
  activity: selectedActivities,
);
```

**Adding an ID after creation**:
```dart
final bookingWithId = booking.copyWith(id: 123);
```

**Updating itinerary config step-by-step**:
```dart
// Step 1: Search form
ItineraryConfig config = ItineraryConfig(
  continent: 'Europe',
  startDate: DateTime(2025, 6, 1),
  endDate: DateTime(2025, 6, 10),
  guests: 2,
);

// Step 2: Add destination
config = config.copyWith(destination: 'paris-france');

// Step 3: Add activities
config = config.copyWith(activities: ['eiffel-tower', 'louvre']);
```

---

## Pattern Combinations

The real power comes from combining patterns!

### Combination 1: MVVM + Repository + Observer

```
┌──────────────────────┐
│   View (Observer)    │
│  ListenableBuilder   │
└──────────┬───────────┘
           │ Observes
           ▼
┌──────────────────────┐
│  ViewModel (MVVM)    │
│  ChangeNotifier      │
└──────────┬───────────┘
           │ Uses
           ▼
┌──────────────────────┐
│ Repository (Strategy)│
│  Abstract Interface  │
└──────────────────────┘
```

**Example**: HomeScreen observes HomeViewModel, which uses BookingRepository

---

### Combination 2: Command + Result + Observer

```
┌──────────────────────┐
│        View          │
└──────────┬───────────┘
           │ Listens to Command
           ▼
┌──────────────────────┐
│  Command (Pattern)   │
│  • running           │
│  • error             │
│  • result            │
└──────────┬───────────┘
           │ Wraps
           ▼
┌──────────────────────┐
│  Result<T> (Pattern) │
│  • Ok(value)         │
│  • Error(exception)  │
└──────────────────────┘
```

**Example**: Delete button listens to deleteBooking command, shows spinner while running, shows error if failed

---

### Combination 3: Repository + Strategy + Factory

```
┌──────────────────────────────┐
│   Factory (DI Config)        │
│  providersLocal or Remote    │
└──────────┬───────────────────┘
           │ Creates
           ▼
┌──────────────────────────────┐
│  Repository (Strategy)       │
│  Local or Remote             │
└──────────────────────────────┘
```

**Example**: Dependency injection acts as factory, creating appropriate repository strategy

---

## Summary

The Compass App demonstrates **9 design patterns** working together:

| Pattern | Purpose | Implementation |
|---------|---------|----------------|
| **Repository** | Abstract data access | BookingRepository with Local/Remote implementations |
| **MVVM** | Separate UI from logic | ViewModels + Widgets + Models |
| **Command** | Encapsulate async actions | Command0, Command1 classes |
| **Result** | Type-safe error handling | Result<T> with Ok/Error |
| **Dependency Injection** | Provide dependencies | Provider package + config |
| **Factory** | Object creation abstraction | Factory methods in use cases |
| **Strategy** | Swappable algorithms | Repository implementations |
| **Observer** | Notify on state changes | ChangeNotifier + ListenableBuilder |
| **Builder** | Flexible object construction | Freezed-generated builders |

**Next**: Explore [Component Deep-Dive](./04-component-deep-dive.md) for detailed component interactions.
