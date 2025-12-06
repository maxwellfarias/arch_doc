# Presentation Layer Testing

## Introduction

The **Presentation Layer** manages UI state and business logic presentation. This includes:
- **Presenters**: Coordinate data loading and state management
- **ViewModels**: Transform domain entities for display
- **Streams**: Reactive state management (using RxDart)

Testing this layer requires understanding **reactive streams** and **state transitions**.

---

## Table of Contents

1. [Presenter Testing Basics](#presenter-testing-basics)
2. [Stream Testing with RxDart](#stream-testing-with-rxdart)
3. [The @Timeout Annotation](#the-timeout-annotation)
4. [Testing Stream Emissions](#testing-stream-emissions)
5. [ViewModel Testing](#viewmodel-testing)
6. [Negative Stream Assertions](#negative-stream-assertions)

---

## Presenter Testing Basics

Presenters coordinate between the domain layer and UI. They typically:
- Load data from use cases/repositories
- Transform data into ViewModels
- Emit state through streams

### Test Setup

```dart
@Timeout(Duration(seconds: 1)) library;

import 'package:advanced_flutter/presentation/rx/next_event_rx_presenter.dart';
import 'package:advanced_flutter/presentation/viewmodels/next_event_viewmodel.dart';

import 'package:flutter_test/flutter_test.dart';

import '../../domain/mocks/next_event_loader_spy.dart';
import '../../mocks/fakes.dart';

void main() {
  late NextEventLoaderSpy nextEventLoader;
  late String groupId;
  late NextEventRxPresenter sut;

  setUp(() {
    nextEventLoader = NextEventLoaderSpy();
    groupId = anyString();
    sut = NextEventRxPresenter(nextEventLoader: nextEventLoader.call);
  });

  // Tests go here
}
```

### Basic Presenter Test

```dart
test('should get event data', () async {
  await sut.loadNextEvent(groupId: groupId);

  expect(nextEventLoader.callsCount, 1);
  expect(nextEventLoader.groupId, groupId);
});
```

---

## Stream Testing with RxDart

RxDart provides powerful stream capabilities. The presenter exposes:
- `nextEventStream`: Emits loaded data or errors
- `isBusyStream`: Emits loading state (true/false)

### Stream Matchers

| Matcher | Usage |
|---------|-------|
| `emits(value)` | Stream emits a single value |
| `emitsInOrder([...])` | Stream emits values in sequence |
| `emitsError(error)` | Stream emits an error |
| `neverEmits(value)` | Stream never emits a value |

### Testing Success Streams

```dart
test('should emit correct events on load with success', () async {
  // Assert that isBusyStream never emits
  sut.isBusyStream.listen(neverCalled);

  // Assert that nextEventStream emits a NextEventViewModel
  expectLater(sut.nextEventStream, emits(isA<NextEventViewModel>()));

  // Act
  await sut.loadNextEvent(groupId: groupId);
});
```

### Testing Error Streams

```dart
test('should emit correct events on load with error', () async {
  // Arrange
  nextEventLoader.error = Error();

  // Assert stream emits the error
  expectLater(sut.nextEventStream, emitsError(nextEventLoader.error));

  // Assert isBusy never emits (not a reload)
  sut.isBusyStream.listen(neverCalled);

  // Act
  await sut.loadNextEvent(groupId: groupId);
});
```

---

## The @Timeout Annotation

Stream tests can hang indefinitely if expectations aren't met. Use `@Timeout` to fail fast:

```dart
@Timeout(Duration(seconds: 1)) library;
```

This annotation:
- Applies to all tests in the file
- Fails tests that take longer than 1 second
- Prevents indefinite hangs from stream issues

### Why Use @Timeout?

Without `@Timeout`, if a stream never emits what you expect, the test waits forever:

```dart
// This would hang forever if nextEventStream never emits
expectLater(sut.nextEventStream, emits(isA<NextEventViewModel>()));
```

With `@Timeout(Duration(seconds: 1))`, it fails after 1 second with a timeout error.

---

## Testing Stream Emissions

### Testing Reload State Sequence

When reloading, the `isBusyStream` should emit `true` then `false`:

```dart
test('should emit correct events on reload with success', () async {
  // Assert isBusy emits true, then false
  expectLater(sut.isBusyStream, emitsInOrder([true, false]));

  // Assert data is emitted
  expectLater(sut.nextEventStream, emits(isA<NextEventViewModel>()));

  // Act - reload
  await sut.loadNextEvent(groupId: groupId, isReload: true);
});
```

### Testing Reload with Error

```dart
test('should emit correct events on reload with error', () async {
  // Arrange
  nextEventLoader.error = Error();

  // Assert error is emitted
  expectLater(sut.nextEventStream, emitsError(nextEventLoader.error));

  // Assert busy state sequence (still true/false even on error)
  expectLater(sut.isBusyStream, emitsInOrder([true, false]));

  // Act
  await sut.loadNextEvent(groupId: groupId, isReload: true);
});
```

### Understanding `expectLater`

`expectLater` is used for async assertions. It returns a `Future` that completes when the expectation is met:

```dart
// Sets up the expectation (doesn't block)
expectLater(sut.nextEventStream, emits(isA<NextEventViewModel>()));

// Triggers the stream emission
await sut.loadNextEvent(groupId: groupId);
```

**Important**: Call `expectLater` BEFORE triggering the action, so it's listening when the event is emitted.

---

## Negative Stream Assertions

### The `neverCalled` Pattern

To assert that a stream should NOT emit anything:

```dart
test('should emit correct events on load with success', () async {
  // isBusy should NOT emit during initial load (only on reload)
  sut.isBusyStream.listen(neverCalled);

  expectLater(sut.nextEventStream, emits(isA<NextEventViewModel>()));

  await sut.loadNextEvent(groupId: groupId);
});
```

`neverCalled` is a function that fails the test if it's ever invoked:

```dart
// If isBusyStream emits ANY value, this test fails
sut.isBusyStream.listen(neverCalled);
```

### When to Use `neverCalled`

| Scenario | Use `neverCalled` When |
|----------|------------------------|
| Initial load | `isBusy` shouldn't emit |
| Cached data | Loading indicator shouldn't show |
| Silent update | No UI state change expected |

---

## ViewModel Testing

ViewModels transform domain entities for display. Test:
- Mapping from entities
- Filtering logic
- Sorting logic

### ViewModel Mapping Test

```dart
test('should map player without confirmation date', () async {
  final player = NextEventPlayer(
    id: anyString(),
    name: anyString(),
    isConfirmed: anyBool(),
    photo: anyString(),
    position: anyString(),
    confirmationDate: null
  );

  final viewmodel = NextEventPlayerViewModel.fromEntity(player);

  expect(viewmodel.name, player.name);
  expect(viewmodel.initials, player.initials);
  expect(viewmodel.isConfirmed, null);
  expect(viewmodel.photo, player.photo);
  expect(viewmodel.position, player.position);
});

test('should map player with confirmation date', () async {
  final player = NextEventPlayer(
    id: anyString(),
    name: anyString(),
    isConfirmed: anyBool(),
    photo: anyString(),
    position: anyString(),
    confirmationDate: anyDate()
  );

  final viewmodel = NextEventPlayerViewModel.fromEntity(player);

  expect(viewmodel.name, player.name);
  expect(viewmodel.initials, player.initials);
  expect(viewmodel.isConfirmed, player.isConfirmed);
  expect(viewmodel.photo, player.photo);
  expect(viewmodel.position, player.position);
});
```

### ViewModel Filtering and Sorting Tests

```dart
test('should build doubt list sorted by name', () async {
  final doubt = NextEventPlayerViewModel.mapDoubtPlayers([
    NextEventPlayer(id: anyString(), name: 'C', isConfirmed: anyBool()),
    NextEventPlayer(id: anyString(), name: 'A', isConfirmed: anyBool()),
    NextEventPlayer(id: anyString(), name: 'B', isConfirmed: anyBool(), confirmationDate: anyDate()),
    NextEventPlayer(id: anyString(), name: 'D', isConfirmed: anyBool())
  ]);

  expect(doubt.length, 3);  // Only those without confirmationDate
  expect(doubt[0].name, 'A');
  expect(doubt[1].name, 'C');
  expect(doubt[2].name, 'D');
});
```

### Testing Sorting by Date

```dart
test('should build out list sorted by confirmation date', () async {
  final out = NextEventPlayerViewModel.mapOutPlayers([
    NextEventPlayer(
      id: anyString(), name: 'C', isConfirmed: false,
      confirmationDate: DateTime(2024, 1, 1, 10)
    ),
    NextEventPlayer(
      id: anyString(), name: 'A', isConfirmed: anyBool()
    ),
    NextEventPlayer(
      id: anyString(), name: 'B', isConfirmed: true,
      confirmationDate: DateTime(2024, 1, 1, 11)
    ),
    NextEventPlayer(
      id: anyString(), name: 'E', isConfirmed: false,
      confirmationDate: DateTime(2024, 1, 1, 9)
    ),
    NextEventPlayer(
      id: anyString(), name: 'D', isConfirmed: false,
      confirmationDate: DateTime(2024, 1, 1, 12)
    )
  ]);

  expect(out.length, 3);  // Only isConfirmed: false
  expect(out[0].name, 'E');  // Earliest date
  expect(out[1].name, 'C');
  expect(out[2].name, 'D');  // Latest date
});
```

### Testing Complex Filtering

```dart
test('should build goalkeepers list sorted by confirmation date', () async {
  final goalkeepers = NextEventPlayerViewModel.mapGoalkeepers([
    NextEventPlayer(
      id: anyString(), name: 'C', isConfirmed: true,
      confirmationDate: DateTime(2024, 1, 1, 10),
      position: 'goalkeeper'
    ),
    NextEventPlayer(
      id: anyString(), name: 'A', isConfirmed: anyBool()
    ),
    NextEventPlayer(
      id: anyString(), name: 'B', isConfirmed: true,
      confirmationDate: DateTime(2024, 1, 1, 11),
      position: 'defender'
    ),
    NextEventPlayer(
      id: anyString(), name: 'E', isConfirmed: false,
      confirmationDate: DateTime(2024, 1, 1, 9)
    ),
    NextEventPlayer(
      id: anyString(), name: 'D', isConfirmed: true,
      confirmationDate: DateTime(2024, 1, 1, 12)
    ),
    NextEventPlayer(
      id: anyString(), name: 'F', isConfirmed: true,
      confirmationDate: DateTime(2024, 1, 1, 8),
      position: 'goalkeeper'
    )
  ]);

  expect(goalkeepers.length, 2);  // Only position: 'goalkeeper' and isConfirmed: true
  expect(goalkeepers[0].name, 'F');  // Earliest date
  expect(goalkeepers[1].name, 'C');
});

test('should build players list sorted by confirmation date', () async {
  final players = NextEventPlayerViewModel.mapInPlayers([
    NextEventPlayer(
      id: anyString(), name: 'C', isConfirmed: true,
      confirmationDate: DateTime(2024, 1, 1, 10),
      position: 'goalkeeper'
    ),
    NextEventPlayer(
      id: anyString(), name: 'A', isConfirmed: anyBool()
    ),
    NextEventPlayer(
      id: anyString(), name: 'B', isConfirmed: true,
      confirmationDate: DateTime(2024, 1, 1, 11),
      position: 'defender'
    ),
    NextEventPlayer(
      id: anyString(), name: 'E', isConfirmed: false,
      confirmationDate: DateTime(2024, 1, 1, 9)
    ),
    NextEventPlayer(
      id: anyString(), name: 'D', isConfirmed: true,
      confirmationDate: DateTime(2024, 1, 1, 12)
    ),
    NextEventPlayer(
      id: anyString(), name: 'F', isConfirmed: true,
      confirmationDate: DateTime(2024, 1, 1, 8),
      position: 'goalkeeper'
    )
  ]);

  expect(players.length, 2);  // isConfirmed: true but NOT goalkeeper
  expect(players[0].name, 'B');  // Sorted by date
  expect(players[1].name, 'D');
});
```

---

## Complete Presenter Test Example

```dart
@Timeout(Duration(seconds: 1)) library;

import 'package:advanced_flutter/presentation/rx/next_event_rx_presenter.dart';
import 'package:advanced_flutter/presentation/viewmodels/next_event_viewmodel.dart';

import 'package:flutter_test/flutter_test.dart';

import '../../domain/mocks/next_event_loader_spy.dart';
import '../../mocks/fakes.dart';

void main() {
  late NextEventLoaderSpy nextEventLoader;
  late String groupId;
  late NextEventRxPresenter sut;

  setUp(() {
    nextEventLoader = NextEventLoaderSpy();
    groupId = anyString();
    sut = NextEventRxPresenter(nextEventLoader: nextEventLoader.call);
  });

  test('should get event data', () async {
    await sut.loadNextEvent(groupId: groupId);

    expect(nextEventLoader.callsCount, 1);
    expect(nextEventLoader.groupId, groupId);
  });

  test('should emit correct events on reload with error', () async {
    nextEventLoader.error = Error();

    expectLater(sut.nextEventStream, emitsError(nextEventLoader.error));
    expectLater(sut.isBusyStream, emitsInOrder([true, false]));

    await sut.loadNextEvent(groupId: groupId, isReload: true);
  });

  test('should emit correct events on load with error', () async {
    nextEventLoader.error = Error();

    expectLater(sut.nextEventStream, emitsError(nextEventLoader.error));
    sut.isBusyStream.listen(neverCalled);

    await sut.loadNextEvent(groupId: groupId);
  });

  test('should emit correct events on reload with success', () async {
    expectLater(sut.isBusyStream, emitsInOrder([true, false]));
    expectLater(sut.nextEventStream, emits(isA<NextEventViewModel>()));

    await sut.loadNextEvent(groupId: groupId, isReload: true);
  });

  test('should emit correct events on load with success', () async {
    sut.isBusyStream.listen(neverCalled);
    expectLater(sut.nextEventStream, emits(isA<NextEventViewModel>()));

    await sut.loadNextEvent(groupId: groupId);
  });
}
```

---

## Presentation Testing Best Practices

| Practice | Description |
|----------|-------------|
| Use `@Timeout` | Prevent hanging tests |
| `expectLater` before action | Set up stream expectations first |
| `neverCalled` for negatives | Assert streams don't emit |
| Test state sequences | `emitsInOrder` for loading states |
| Separate ViewModel tests | Test mapping logic independently |

---

## Stream Testing Checklist

- [ ] Add `@Timeout` annotation
- [ ] Test success emissions
- [ ] Test error emissions
- [ ] Test loading state sequence (reload)
- [ ] Test that loading state doesn't emit (initial load)
- [ ] Use `expectLater` BEFORE triggering action
- [ ] Use `neverCalled` for negative assertions

---

## Next Steps

Now that you understand presentation testing, proceed to:

**[→ UI/Widget Testing](06_UI_TESTING.md)**: Learn how to test Flutter widgets and pages.
