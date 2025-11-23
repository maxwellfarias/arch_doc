# Getting Started Guide - Beginner's Roadmap

## Table of Contents
- [Welcome](#welcome)
- [Prerequisites](#prerequisites)
- [Setting Up Your Environment](#setting-up-your-environment)
- [Running the Project](#running-the-project)
- [Understanding the Codebase](#understanding-the-codebase)
- [Reading Code Walkthrough](#reading-code-walkthrough)
- [Running Tests](#running-tests)
- [Adding a New Feature](#adding-a-new-feature)
- [Common Patterns & Conventions](#common-patterns--conventions)
- [Troubleshooting](#troubleshooting)
- [Learning Path](#learning-path)

---

## Welcome

Welcome to your journey into professional Flutter development! This guide will help you:
- Set up and run the project
- Navigate the codebase confidently
- Understand the architecture hands-on
- Add your first feature following Clean Architecture
- Master TDD (Test-Driven Development)

**Time investment**: Plan 4-6 hours to complete this guide thoroughly.

---

## Prerequisites

### Required Knowledge

**Beginner Level** (Must have):
- ✅ Basic Dart syntax (variables, functions, classes)
- ✅ Basic Flutter (StatelessWidget, StatefulWidget, build method)
- ✅ Understanding of async/await
- ✅ Git basics (clone, commit, push)

**Intermediate Level** (Helpful but not required):
- 📖 Object-Oriented Programming concepts
- 📖 JSON and REST APIs
- 📖 Unit testing basics

**Don't worry if you don't know**:
- ❌ Clean Architecture (you'll learn it here!)
- ❌ Design patterns (covered in docs)
- ❌ RxDart (we'll explain)
- ❌ TDD methodology (you'll practice)

### Required Software

| Software | Version | Download |
|----------|---------|----------|
| **Flutter** | 3.0+ | https://flutter.dev/docs/get-started/install |
| **Dart** | 3.0+ | Included with Flutter |
| **Node.js** | 18+ | https://nodejs.org |
| **Git** | Latest | https://git-scm.com |
| **IDE** | Any | VS Code (recommended) or Android Studio |

### Recommended VS Code Extensions

- Flutter (Dart-Code.flutter)
- Dart (Dart-Code.dart-code)
- Error Lens (usernamehw.errorlens)
- Better Comments (aaron-bond.better-comments)

---

## Setting Up Your Environment

### Step 1: Clone the Repository

```bash
# Navigate to your projects folder
cd ~/Documents/projects

# Clone the repository (replace with your repo URL)
git clone <repository-url>
cd fluttter_avancado_tdd

# Navigate to the Flutter project
cd fluttter_avancado_tdd_clean_arch
```

### Step 2: Install Flutter Dependencies

```bash
# Get all Flutter packages
flutter pub get

# Verify installation
flutter doctor
```

**Expected output**: All checks should pass or have green checkmarks.

### Step 3: Set Up the Backend (Mock API)

The project includes a Node.js Express server for development.

```bash
# Navigate to backend folder
cd backend

# Install Node.js dependencies
npm install

# Start the server
npm start
```

**Expected output**:
```
Server running at http://localhost:8080
```

**Keep this terminal open** while developing!

### Step 4: Verify Backend

Open your browser and navigate to:
```
http://localhost:8080/api/groups/valid_id/next_event
```

You should see JSON data with soccer event information.

---

## Running the Project

### Option 1: Run on iOS Simulator (macOS only)

```bash
# Open iOS Simulator
open -a Simulator

# Run Flutter app
flutter run
```

### Option 2: Run on Android Emulator

```bash
# List available devices
flutter devices

# Start Android emulator (replace with your device name)
flutter emulators --launch <emulator_id>

# Run Flutter app
flutter run
```

### Option 3: Run on Chrome (Web)

```bash
flutter run -d chrome
```

### Expected Result

You should see the app with:
- App bar: "Próximo Jogo"
- Loading spinner initially
- Then lists of players categorized by:
  - DENTRO - GOLEIROS (Confirmed goalkeepers)
  - DENTRO - JOGADORES (Confirmed field players)
  - FORA (Players who cannot attend)
  - DÚVIDA (Undecided players)

### Pull to Refresh

Try pulling down on the list—you should see a loading dialog while data refreshes.

---

## Understanding the Codebase

### Project Structure Overview

```
fluttter_avancado_tdd_clean_arch/
├── lib/
│   ├── domain/          # 🎯 Start here: Business entities
│   ├── infra/           # 🔧 Data fetching & conversion
│   ├── presentation/    # 🧠 UI business logic
│   ├── ui/              # 🎨 Flutter widgets
│   └── main/            # 🏭 Dependency injection
│
├── test/                # 🧪 Mirror of lib/ with tests
│
├── backend/             # 🖥️ Mock API server
│
└── pubspec.yaml         # 📦 Dependencies
```

### File Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| **Entities** | `snake_case.dart` | `next_event_player.dart` |
| **Repositories** | `action_source_repo.dart` | `load_next_event_api_repo.dart` |
| **Adapters** | `library_adapter.dart` | `http_adapter.dart` |
| **Presenters** | `screen_presenter.dart` | `next_event_presenter.dart` |
| **Pages** | `screen_page.dart` | `next_event_page.dart` |
| **Components** | `component_name.dart` | `player_photo.dart` |
| **Factories** | `thing_factory.dart` | `http_adapter_factory.dart` |
| **Tests** | `same_name_test.dart` | `next_event_player_test.dart` |

### Import Organization

**Order** (recommended):
1. Dart SDK imports
2. Flutter imports
3. External package imports
4. Project imports

**Example**:
```dart
// Dart SDK
import 'dart:convert';

// Flutter
import 'package:flutter/material.dart';

// External packages
import 'package:rxdart/rxdart.dart';
import 'package:http/http.dart';

// Project
import 'package:fluttter_avancado_tdd_clean_arch/domain/entities/next_event.dart';
```

---

## Reading Code Walkthrough

Let's read through a complete feature to understand how everything connects.

### Walkthrough: "Display Next Event"

Follow this sequence to understand data flow:

#### Step 1: Entry Point

**File**: [lib/main/main.dart](fluttter_avancado_tdd_clean_arch/lib/main/main.dart)

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: makeNextEventPage(),  // 👈 Start reading here
    );
  }
}
```

**Question to ask**: "What does `makeNextEventPage()` do?"

---

#### Step 2: Page Factory

**File**: [lib/main/factories/ui/pages/next_event_page_factory.dart](fluttter_avancado_tdd_clean_arch/lib/main/factories/ui/pages/next_event_page_factory.dart)

```dart
Widget makeNextEventPage() {
  final repo = makeLoadNextEventFromApiWithCacheFallbackRepository();
  final presenter = NextEventRxPresenter(
    nextEventLoader: repo.loadNextEvent,
  );
  return NextEventPage(
    presenter: presenter,
    groupId: 'valid_id',
  );
}
```

**Questions**:
1. "What is the composite repository?" → Click into `makeLoadNextEventFromApiWithCacheFallbackRepository()`
2. "What does the presenter do?" → Open `NextEventRxPresenter`
3. "What does the page do?" → Open `NextEventPage`

**Exercise**: Trace each factory call to see what objects are created.

---

#### Step 3: Domain Entity

**File**: [lib/domain/entities/next_event_player.dart](fluttter_avancado_tdd_clean_arch/lib/domain/entities/next_event_player.dart)

```dart
final class NextEventPlayer {
  final String name;
  final String initials;  // 🎯 Auto-generated

  factory NextEventPlayer({required String name, ...}) =>
      NextEventPlayer._(
        name: name,
        initials: _getInitials(name),  // 👈 Business logic
        ...
      );

  static String _getInitials(String name) {
    // Read this method to understand the logic
  }
}
```

**Exercise**:
1. Find the test file: [test/domain/entities/next_event_player_test.dart](fluttter_avancado_tdd_clean_arch/test/domain/entities/next_event_player_test.dart)
2. Read the test cases to understand expected behavior
3. Try to predict the output for "John Doe" before checking the code

---

#### Step 4: HTTP Adapter

**File**: [lib/infra/api/adapters/http_adapter.dart](fluttter_avancado_tdd_clean_arch/lib/infra/api/adapters/http_adapter.dart)

**Read in this order**:
1. `get()` method - Entry point
2. `_buildUri()` - See how URLs are constructed
3. `_buildHeaders()` - Default headers
4. `_handleResponse()` - Status code handling

**Exercise**: Add a `print()` statement in `get()` to log the final URI:
```dart
Future<dynamic> get({...}) async {
  final uri = _buildUri(url: url, params: params, queryString: queryString);
  print('HTTP GET: $uri');  // 👈 Add this
  ...
}
```

Run the app and watch the console—you'll see the actual URL being called!

---

#### Step 5: Mapper

**File**: [lib/infra/mappers/next_event_player_mapper.dart](fluttter_avancado_tdd_clean_arch/lib/infra/mappers/next_event_player_mapper.dart)

```dart
final class NextEventPlayerMapper extends ListMapper<NextEventPlayer> {
  @override
  NextEventPlayer toDto(dynamic json) => NextEventPlayer(
    id: json['id'],
    name: json['name'],
    // ... read each field mapping
  );
}
```

**Exercise**:
1. Print the JSON being mapped:
```dart
@override
NextEventPlayer toDto(dynamic json) {
  print('Mapping player: ${json['name']}');  // 👈 Add this
  return NextEventPlayer(...);
}
```
2. Run the app—you'll see each player being mapped!

---

#### Step 6: Presenter

**File**: [lib/presentation/rx/next_event_rx_presenter.dart](fluttter_avancado_tdd_clean_arch/lib/presentation/rx/next_event_rx_presenter.dart)

**Read in this order**:
1. Constructor - See what's injected
2. `loadNextEvent()` - Main method
3. `_mapEvent()` - How categorization works
4. `_mapPlayer()` - Entity → ViewModel

**Exercise**: Add logging to understand categorization:
```dart
NextEventViewModel _mapEvent(NextEvent event) {
  final doubt = event.players
      .where((player) => player.confirmationDate == null)
      .sortedBy((player) => player.name)
      .map(_mapPlayer)
      .toList();

  print('Doubt players: ${doubt.length}');  // 👈 Add this

  return NextEventViewModel(doubt: doubt, ...);
}
```

---

#### Step 7: UI Page

**File**: [lib/ui/pages/next_event_page.dart](fluttter_avancado_tdd_clean_arch/lib/ui/pages/next_event_page.dart)

**Read in this order**:
1. `initState()` - What happens when page opens
2. `build()` - StreamBuilder setup
3. Error handling - `snapshot.hasError`
4. Data display - ListView with sections
5. `dispose()` - Cleanup

**Exercise**: Understand StreamBuilder states:
```dart
builder: (context, snapshot) {
  print('ConnectionState: ${snapshot.connectionState}');  // 👈 Add this
  print('Has data: ${snapshot.hasData}');
  print('Has error: ${snapshot.hasError}');

  if (snapshot.connectionState != ConnectionState.active) {
    return const Center(child: CircularProgressIndicator());
  }
  ...
}
```

---

## Running Tests

### Run All Tests

```bash
# From project root
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

### Understanding Test Output

**Example output**:
```
00:01 +1: test/domain/entities/next_event_player_test.dart: should return the first letter of the first and last names
00:02 +2: test/domain/entities/next_event_player_test.dart: should ignore extra whitespaces
00:03 +14: All tests passed!
```

**Legend**:
- `+1`, `+2`: Number of tests passed
- `00:01`: Time elapsed
- `All tests passed!`: Success!

### Reading a Test

**File**: [test/domain/entities/next_event_player_test.dart](fluttter_avancado_tdd_clean_arch/test/domain/entities/next_event_player_test.dart)

```dart
test('should return the first letter of the first and last names', () {
  // Arrange (setup)
  const name = "Maxwell Farias";

  // Act (execute)
  final initials = initialsOf(name);

  // Assert (verify)
  expect(initials, 'MF');
});
```

**Pattern**: Arrange → Act → Assert (AAA)

**Exercise**: Add your own test case:
```dart
test('should handle my name', () {
  // Change to your name!
  expect(initialsOf("Your Name"), 'YN');
});
```

Run the test and see if it passes!

---

## Adding a New Feature

Let's add a new feature: **Display event date in a friendly format**

Currently, the date isn't shown on the screen. We'll add it following Clean Architecture.

### Step 1: Define Requirements

**User Story**: As a user, I want to see when the next event is scheduled.

**Acceptance Criteria**:
- Date shown in format: "11 de Janeiro de 2024"
- Displayed below the app bar
- Only shown if event has a date

### Step 2: Write Test First (TDD)

**Create test file**: `test/presentation/rx/next_event_rx_presenter_test.dart`

Find the test and add:

```dart
test('should include event date in ViewModel', () async {
  // Arrange
  final event = NextEvent(
    groupName: 'Test Group',
    date: DateTime(2024, 1, 11),
    players: [],
  );
  nextEventLoader.output = event;

  // Act
  await sut.loadNextEvent(groupId: groupId);

  // Assert
  sut.nextEventStream.listen((viewModel) {
    expect(viewModel.eventDate, DateTime(2024, 1, 11));
  });
});
```

**Run test**: `flutter test` → It will fail (property doesn't exist yet)

### Step 3: Add Property to ViewModel

**File**: [lib/presentation/presenters/next_event_presenter.dart](fluttter_avancado_tdd_clean_arch/lib/presentation/presenters/next_event_presenter.dart)

```dart
final class NextEventViewModel {
  final DateTime eventDate;  // 👈 Add this
  final List<NextEventPlayerViewModel> goalkeepers;
  final List<NextEventPlayerViewModel> players;
  final List<NextEventPlayerViewModel> out;
  final List<NextEventPlayerViewModel> doubt;

  const NextEventViewModel({
    required this.eventDate,  // 👈 Add this
    this.goalkeepers = const [],
    this.players = const [],
    this.out = const [],
    this.doubt = const [],
  });
}
```

### Step 4: Update Presenter Mapping

**File**: [lib/presentation/rx/next_event_rx_presenter.dart](fluttter_avancado_tdd_clean_arch/lib/presentation/rx/next_event_rx_presenter.dart)

```dart
NextEventViewModel _mapEvent(NextEvent event) => NextEventViewModel(
      eventDate: event.date,  // 👈 Add this
      doubt: event.players.where(...).toList(),
      out: event.players.where(...).toList(),
      goalkeepers: event.players.where(...).toList(),
      players: event.players.where(...).toList(),
    );
```

**Run test again**: `flutter test` → Should pass now! ✅

### Step 5: Create Date Formatter Component

**Create file**: `lib/ui/components/event_date.dart`

```dart
import 'package:flutter/material.dart';
import 'package:intl/intl.dart';

final class EventDate extends StatelessWidget {
  final DateTime date;

  const EventDate({super.key, required this.date});

  String _formatDate() {
    // Format: "11 de Janeiro de 2024"
    final formatter = DateFormat('dd \'de\' MMMM \'de\' yyyy', 'pt_BR');
    return formatter.format(date);
  }

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.all(16.0),
      child: Text(
        _formatDate(),
        style: const TextStyle(
          fontSize: 18,
          fontWeight: FontWeight.bold,
        ),
        textAlign: TextAlign.center,
      ),
    );
  }
}
```

### Step 6: Add to Page

**File**: [lib/ui/pages/next_event_page.dart](fluttter_avancado_tdd_clean_arch/lib/ui/pages/next_event_page.dart)

```dart
@override
Widget build(BuildContext context) {
  return Scaffold(
    appBar: AppBar(
      title: const Text('Próximo Jogo'),
      centerTitle: true,
    ),
    body: StreamBuilder<NextEventViewModel>(
      stream: widget.presenter.nextEventStream,
      builder: (context, snapshot) {
        if (snapshot.connectionState != ConnectionState.active) {
          return const Center(child: CircularProgressIndicator());
        }

        if (snapshot.hasError) {
          return buildErrorLayout(snapshot.error as DomainError);
        }

        final viewModel = snapshot.data!;
        return RefreshIndicator(
          onRefresh: () async => widget.presenter.loadNextEvent(
            groupId: widget.groupId,
            isReload: true,
          ),
          child: ListView(
            padding: const EdgeInsets.all(20),
            children: [
              EventDate(date: viewModel.eventDate),  // 👈 Add this
              if (viewModel.goalkeepers.isNotEmpty)
                ListSection(
                  title: 'DENTRO - GOLEIROS',
                  items: viewModel.goalkeepers,
                ),
              // ... rest of sections
            ],
          ),
        );
      },
    ),
  );
}
```

### Step 7: Add intl Package

**File**: `pubspec.yaml`

```yaml
dependencies:
  flutter:
    sdk: flutter
  # ... existing dependencies
  intl: ^0.18.0  # 👈 Add this
```

Run: `flutter pub get`

### Step 8: Write Widget Test

**Create file**: `test/ui/components/event_date_test.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:fluttter_avancado_tdd_clean_arch/ui/components/event_date.dart';

void main() {
  testWidgets('should display formatted date', (tester) async {
    // Arrange
    final date = DateTime(2024, 1, 11);
    final sut = MaterialApp(
      home: Scaffold(
        body: EventDate(date: date),
      ),
    );

    // Act
    await tester.pumpWidget(sut);

    // Assert
    expect(find.text('11 de Janeiro de 2024'), findsOneWidget);
  });
}
```

**Run test**: `flutter test test/ui/components/event_date_test.dart`

### Step 9: Run the App

```bash
flutter run
```

You should now see the event date displayed below the app bar!

### Step 10: Commit Your Changes

```bash
git add .
git commit -m "feat: display event date in friendly format

- Add eventDate to NextEventViewModel
- Create EventDate component with pt_BR formatting
- Add widget and presenter tests
- Update NextEventPage to display date"
```

**Congratulations!** You just added a feature following:
- ✅ Clean Architecture
- ✅ TDD (Test-Driven Development)
- ✅ Proper separation of concerns
- ✅ Professional commit message

---

## Common Patterns & Conventions

### Pattern 1: Creating a New Entity

```dart
// 1. Define in domain layer
// lib/domain/entities/my_entity.dart
final class MyEntity {
  final String id;
  final String name;

  const MyEntity({
    required this.id,
    required this.name,
  });
}

// 2. Create mapper
// lib/infra/mappers/my_entity_mapper.dart
final class MyEntityMapper implements Mapper<MyEntity> {
  @override
  MyEntity toDto(dynamic json) => MyEntity(
    id: json['id'],
    name: json['name'],
  );

  @override
  Json toJson(MyEntity entity) => {
    'id': entity.id,
    'name': entity.name,
  };
}

// 3. Write tests
// test/infra/mappers/my_entity_mapper_test.dart
test('should convert JSON to entity', () {
  final json = {'id': '1', 'name': 'Test'};
  final entity = sut.toDto(json);
  expect(entity.id, '1');
  expect(entity.name, 'Test');
});
```

### Pattern 2: Creating a New Repository

```dart
// 1. Define in infra layer
// lib/infra/api/repositories/my_repository.dart
final class MyRepository {
  final HttpGetClient httpClient;
  final String url;
  final DtoMapper<MyEntity> mapper;

  const MyRepository({
    required this.httpClient,
    required this.url,
    required this.mapper,
  });

  Future<MyEntity> loadData({required String id}) async {
    final json = await httpClient.get(
      url: url,
      params: {"id": id},
    );
    return mapper.toDto(json);
  }
}

// 2. Create factory
// lib/main/factories/infra/repositories/my_repository_factory.dart
MyRepository makeMyRepository() {
  return MyRepository(
    httpClient: makeHttpAdapter(),
    url: 'http://localhost:8080/api/my-endpoint/:id',
    mapper: makeMyEntityMapper(),
  );
}

// 3. Write tests
// test/infra/repositories/my_repository_test.dart
test('should load data from API', () async {
  final data = await sut.loadData(id: '123');
  expect(httpClient.url, contains('123'));
  expect(data, isA<MyEntity>());
});
```

### Pattern 3: Creating a New Component

```dart
// 1. Create component
// lib/ui/components/my_component.dart
final class MyComponent extends StatelessWidget {
  final String data;

  const MyComponent({super.key, required this.data});

  @override
  Widget build(BuildContext context) {
    return Text(data);
  }
}

// 2. Write widget test
// test/ui/components/my_component_test.dart
testWidgets('should display data', (tester) async {
  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: MyComponent(data: 'Test'),
      ),
    ),
  );

  expect(find.text('Test'), findsOneWidget);
});
```

### Commit Message Convention

Follow this format:

```
<type>: <short description>

<optional longer description>

<optional footer>
```

**Types**:
- `feat`: New feature
- `fix`: Bug fix
- `refactor`: Code refactoring
- `test`: Add/update tests
- `docs`: Documentation
- `style`: Code style changes (formatting)
- `chore`: Build process, dependencies

**Examples**:
```
feat: add player filtering by position

Allow users to filter players by their position (goalkeeper, defender, etc.)

Closes #123
```

```
fix: correct initials generation for single names

Previously, single-word names would throw an error. Now defaults to first two letters.
```

---

## Troubleshooting

### Problem: "Waiting for another flutter command to release the startup lock"

**Solution**:
```bash
killall -9 dart
flutter clean
flutter pub get
```

### Problem: Backend not responding

**Check**:
1. Is the backend server running? (`npm start` in `backend/`)
2. Is it on the correct port? (Should be 8080)
3. Try accessing in browser: `http://localhost:8080/api/groups/valid_id/next_event`

**Solution**:
```bash
cd backend
killall node  # Stop all Node processes
npm start     # Restart
```

### Problem: "Error: Connection refused"

**For iOS Simulator**:
Change `localhost` to `127.0.0.1` in:
- [lib/main/factories/infra/api/repositories/load_next_event_api_repo_factory.dart](fluttter_avancado_tdd_clean_arch/lib/main/factories/infra/api/repositories/load_next_event_api_repo_factory.dart)

```dart
LoadNextEventApiRepository makeLoadNextEventApiRepository() {
  return LoadNextEventApiRepository(
    httpClient: makeHttpAdapter(),
    url: 'http://127.0.0.1:8080/api/groups/:groupId/next_event',  // Changed
    mapper: makeNextEventMapper(),
  );
}
```

**For Android Emulator**:
Use `10.0.2.2` instead of `localhost`:

```dart
url: 'http://10.0.2.2:8080/api/groups/:groupId/next_event',
```

### Problem: Tests failing with "Stream has already been listened to"

**Reason**: BehaviorSubject can have multiple listeners, but you might be subscribing incorrectly.

**Solution**: Use `stream` property, not the subject directly:
```dart
// ❌ Wrong
presenter.nextEventSubject.listen(...)

// ✅ Correct
presenter.nextEventStream.listen(...)
```

### Problem: "setState called after dispose"

**Reason**: Stream subscription not cancelled.

**Solution**: Always cancel in dispose:
```dart
@override
void dispose() {
  _subscription.cancel();  // Add this
  widget.presenter.dispose();
  super.dispose();
}
```

### Problem: Hot reload not working

**Solution**:
1. Try hot restart: `r` in terminal or `Shift + R`
2. If still broken: `flutter clean && flutter run`

---

## Learning Path

### Week 1: Foundations

**Goals**:
- ✅ Run the project successfully
- ✅ Understand the 4 layers (Domain, Infra, Presentation, UI)
- ✅ Read through one complete flow (entry point to UI)

**Tasks**:
1. Read [01-ARCHITECTURE-OVERVIEW.md](01-ARCHITECTURE-OVERVIEW.md)
2. Follow the "Reading Code Walkthrough" section
3. Add `print()` statements to trace data flow
4. Run all tests and understand test output

**Quiz yourself**:
- What is the entry point of the application?
- What are the 4 layers and their responsibilities?
- Where is business logic located?
- Where is JSON parsing done?

---

### Week 2: Deep Dive

**Goals**:
- ✅ Understand each design pattern used
- ✅ Trace data flow from API to UI
- ✅ Read and understand all tests

**Tasks**:
1. Read [02-LAYER-BREAKDOWN.md](02-LAYER-BREAKDOWN.md)
2. Read [03-DESIGN-PATTERNS.md](03-DESIGN-PATTERNS.md)
3. Trace a complete request-response cycle
4. Study test files for each layer

**Quiz yourself**:
- What is the Adapter pattern and where is it used?
- How does the Repository pattern work?
- What is MVP and how is it implemented?
- How do streams work for UI updates?

---

### Week 3: Hands-On

**Goals**:
- ✅ Add a new feature following TDD
- ✅ Write tests for new code
- ✅ Understand component interactions

**Tasks**:
1. Follow "Adding a New Feature" section
2. Add another feature on your own (e.g., search players by name)
3. Read [04-COMPONENT-DEEP-DIVE.md](04-COMPONENT-DEEP-DIVE.md)
4. Write tests before code (TDD)

**Quiz yourself**:
- How do you add a new property to a ViewModel?
- Where should filtering logic go?
- How do you test a repository?
- What is the correct test pattern (AAA)?

---

### Week 4: Mastery

**Goals**:
- ✅ Refactor existing code
- ✅ Improve test coverage
- ✅ Understand all patterns deeply

**Tasks**:
1. Refactor a component (improve code quality)
2. Add tests for edge cases
3. Experiment with different implementations (e.g., Bloc instead of RxDart)
4. Review SOLID principles in codebase

**Quiz yourself**:
- Can you explain each SOLID principle with examples from the code?
- How would you add a new data source (e.g., local database)?
- How would you handle pagination?
- How would you add user authentication?

---

## Practice Exercises

### Exercise 1: Add Player Search

**Goal**: Add a search bar to filter players by name

**Steps**:
1. Add `searchQuery` to `NextEventViewModel`
2. Update presenter to filter players based on query
3. Add `TextField` widget to page
4. Connect TextField changes to presenter
5. Write tests

### Exercise 2: Add Player Avatars

**Goal**: Show player photos in a circular avatar

**Current**: Shows initials or image
**New**: Always show image with initials as fallback

**Steps**:
1. Update `PlayerPhoto` component
2. Handle loading states (shimmer effect)
3. Handle errors (show initials)
4. Write widget tests

### Exercise 3: Add Offline Indicator

**Goal**: Show banner when using cached data

**Steps**:
1. Add `isFromCache` property to ViewModel
2. Update composite repository to track source
3. Add banner widget to page
4. Write tests

### Exercise 4: Add Pull-to-Refresh Loading

**Goal**: Show inline loading indicator during refresh

**Current**: Shows modal dialog
**New**: Show progress bar below app bar

**Steps**:
1. Update `isBusyStream` logic
2. Replace dialog with `LinearProgressIndicator`
3. Update widget tests

---

## Next Steps

### Continue Learning

1. **Experiment**: Try modifying the code—break things and fix them!
2. **Read tests**: Tests are excellent documentation
3. **Ask questions**: Use code comments to understand "why"
4. **Build your own**: Create a similar project from scratch

### Resources

- **Flutter Documentation**: https://flutter.dev/docs
- **Dart Language Tour**: https://dart.dev/guides/language/language-tour
- **Clean Architecture**: Book by Robert C. Martin
- **RxDart**: https://pub.dev/packages/rxdart
- **Testing Flutter Apps**: https://flutter.dev/docs/testing

### Join the Community

- Flutter Discord: https://discord.gg/flutter
- r/FlutterDev: https://reddit.com/r/FlutterDev
- Stack Overflow: Tag your questions with `flutter`

---

## Conclusion

You now have:
- ✅ A running Flutter project with Clean Architecture
- ✅ Understanding of the 4-layer architecture
- ✅ Knowledge of design patterns in action
- ✅ Experience with TDD methodology
- ✅ Ability to add new features professionally

**Remember**: Professional development is about:
- Writing maintainable code
- Testing thoroughly
- Separating concerns
- Following conventions
- Continuous learning

**Keep practicing, keep building, and welcome to professional Flutter development!** 🚀

---

## Quick Reference Card

### Common Commands

```bash
# Run app
flutter run

# Run tests
flutter test

# Hot reload
r

# Hot restart
R

# Clean build
flutter clean

# Get dependencies
flutter pub get

# Check Flutter setup
flutter doctor

# Run backend
cd backend && npm start
```

### File Locations Cheat Sheet

| What | Where |
|------|-------|
| Entities | `lib/domain/entities/` |
| Repositories | `lib/infra/*/repositories/` |
| Adapters | `lib/infra/*/adapters/` |
| Mappers | `lib/infra/mappers/` |
| Presenters | `lib/presentation/` |
| Pages | `lib/ui/pages/` |
| Components | `lib/ui/components/` |
| Factories | `lib/main/factories/` |
| Tests | `test/` (mirrors `lib/`) |

### Testing Quick Commands

```bash
# Run all tests
flutter test

# Run specific test
flutter test path/to/test_file.dart

# Run with coverage
flutter test --coverage

# Watch mode (re-run on changes)
flutter test --watch
```

Happy coding! 🎉
