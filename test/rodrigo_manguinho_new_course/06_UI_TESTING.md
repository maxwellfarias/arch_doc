# UI/Widget Testing

## Introduction

**Widget Tests** verify that Flutter UI components render correctly and respond to user interactions. This includes:
- **Component Tests**: Individual widgets in isolation
- **Page Tests**: Complete screens with mocked presenters
- **Interaction Tests**: User gestures and actions

Widget tests are faster than integration tests but more comprehensive than unit tests.

---

## Table of Contents

1. [Widget Testing Basics](#widget-testing-basics)
2. [Component Testing](#component-testing)
3. [Network Image Mocking](#network-image-mocking)
4. [Page Testing](#page-testing)
5. [Testing User Interactions](#testing-user-interactions)
6. [Pull-to-Refresh Testing](#pull-to-refresh-testing)
7. [Widget Finding Strategies](#widget-finding-strategies)

---

## Widget Testing Basics

### The `testWidgets` Function

Widget tests use `testWidgets` instead of `test`:

```dart
testWidgets('should display text', (tester) async {
  await tester.pumpWidget(const MaterialApp(home: MyWidget()));

  expect(find.text('Hello'), findsOneWidget);
});
```

### The WidgetTester

The `tester` parameter provides methods to:
- Build widgets (`pumpWidget`)
- Trigger rebuilds (`pump`, `pumpAndSettle`)
- Simulate interactions (`tap`, `fling`)
- Find widgets (`find`)

### Always Wrap with MaterialApp

Widgets need a MaterialApp ancestor for themes and navigation:

```dart
// ✅ Correct
await tester.pumpWidget(const MaterialApp(home: MyWidget()));

// ❌ Wrong - will throw errors
await tester.pumpWidget(const MyWidget());
```

---

## Component Testing

### Simple Status Component Test

```dart
import 'package:advanced_flutter/ui/components/player_status.dart';

import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  testWidgets('should present green status', (tester) async {
    await tester.pumpWidget(
      const MaterialApp(home: PlayerStatus(isConfirmed: true))
    );

    final decoration = tester.firstWidget<Container>(
      find.byType(Container)
    ).decoration as BoxDecoration;

    expect(decoration.color, Colors.teal);
  });

  testWidgets('should present red status', (tester) async {
    await tester.pumpWidget(
      const MaterialApp(home: PlayerStatus(isConfirmed: false))
    );

    final decoration = tester.firstWidget<Container>(
      find.byType(Container)
    ).decoration as BoxDecoration;

    expect(decoration.color, Colors.pink);
  });

  testWidgets('should present grey status', (tester) async {
    await tester.pumpWidget(
      const MaterialApp(home: PlayerStatus(isConfirmed: null))
    );

    final decoration = tester.firstWidget<Container>(
      find.byType(Container)
    ).decoration as BoxDecoration;

    expect(decoration.color, Colors.blueGrey);
  });
}
```

### Testing Widget Properties

To access widget properties:

```dart
// Get the first widget of a type
final container = tester.firstWidget<Container>(find.byType(Container));

// Access properties
final decoration = container.decoration as BoxDecoration;
expect(decoration.color, Colors.teal);
```

### Position/Label Component Test

```dart
import 'package:advanced_flutter/ui/components/player_position.dart';

import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  testWidgets('should handle goalkeeper position', (tester) async {
    await tester.pumpWidget(
      const MaterialApp(home: PlayerPosition(position: 'goalkeeper'))
    );

    expect(find.text('Goleiro'), findsOneWidget);
  });

  testWidgets('should handle defender position', (tester) async {
    await tester.pumpWidget(
      const MaterialApp(home: PlayerPosition(position: 'defender'))
    );

    expect(find.text('Zagueiro'), findsOneWidget);
  });

  testWidgets('should handle midfielder position', (tester) async {
    await tester.pumpWidget(
      const MaterialApp(home: PlayerPosition(position: 'midfielder'))
    );

    expect(find.text('Meia'), findsOneWidget);
  });

  testWidgets('should handle forward position', (tester) async {
    await tester.pumpWidget(
      const MaterialApp(home: PlayerPosition(position: 'forward'))
    );

    expect(find.text('Atacante'), findsOneWidget);
  });

  testWidgets('should handle positionless', (tester) async {
    await tester.pumpWidget(
      const MaterialApp(home: PlayerPosition(position: null))
    );

    expect(find.text('Gandula'), findsOneWidget);
  });
}
```

---

## Network Image Mocking

Network images fail in tests because there's no real network. Use `network_image_mock`:

### Add Dependency

```yaml
dev_dependencies:
  network_image_mock: ^2.1.1
```

### Usage

```dart
import 'package:advanced_flutter/ui/components/player_photo.dart';

import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:network_image_mock/network_image_mock.dart';

void main() {
  testWidgets('should present initials when there is no photo', (tester) async {
    await tester.pumpWidget(
      const MaterialApp(home: PlayerPhoto(initials: 'RO', photo: null))
    );

    expect(find.text('RO'), findsOneWidget);
  });

  testWidgets('should hide initials when there is photo', (tester) async {
    // Wrap with mockNetworkImagesFor
    mockNetworkImagesFor(() async {
      await tester.pumpWidget(
        const MaterialApp(
          home: PlayerPhoto(initials: 'RO', photo: 'http://any-url.com')
        )
      );

      expect(find.text('RO'), findsNothing);
    });
  });
}
```

### When to Use `mockNetworkImagesFor`

| Scenario | Use Mock |
|----------|----------|
| Widget has `Image.network` | Yes |
| Widget has `CachedNetworkImage` | Yes |
| Widget only uses local assets | No |
| Photo URL is null | No |

---

## Page Testing

Page tests verify complete screens with mocked presenters.

### Test Setup

```dart
import 'package:advanced_flutter/presentation/viewmodels/next_event_player_viewmodel.dart';
import 'package:advanced_flutter/ui/components/player_photo.dart';
import 'package:advanced_flutter/ui/components/player_position.dart';
import 'package:advanced_flutter/ui/components/player_status.dart';
import 'package:advanced_flutter/ui/pages/next_event_page.dart';

import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

import '../../mocks/fakes.dart';
import '../../presentation/mocks/next_event_presenter_spy.dart';

void main() {
  late NextEventPresenterSpy presenter;
  late String groupId;
  late Widget sut;

  setUp(() {
    presenter = NextEventPresenterSpy();
    groupId = anyString();
    sut = MaterialApp(
      home: NextEventPage(presenter: presenter, groupId: groupId)
    );
  });

  // Tests go here
}
```

### Testing Initial Load

```dart
testWidgets('should load event data on page init', (tester) async {
  await tester.pumpWidget(sut);

  expect(presenter.callsCount, 1);
  expect(presenter.groupId, groupId);
  expect(presenter.isReload, false);
});
```

### Testing Loading State

```dart
testWidgets('should present spinner while data is loading', (tester) async {
  await tester.pumpWidget(sut);

  expect(find.byType(CircularProgressIndicator), findsOneWidget);
});

testWidgets('should hide spinner on load success', (tester) async {
  await tester.pumpWidget(sut);
  expect(find.byType(CircularProgressIndicator), findsOneWidget);

  presenter.emitNextEvent();  // Emit success
  await tester.pump();        // Trigger rebuild

  expect(find.byType(CircularProgressIndicator), findsNothing);
});

testWidgets('should hide spinner on load error', (tester) async {
  await tester.pumpWidget(sut);
  expect(find.byType(CircularProgressIndicator), findsOneWidget);

  presenter.emitError();  // Emit error
  await tester.pump();

  expect(find.byType(CircularProgressIndicator), findsNothing);
});
```

### Testing Data Display

```dart
testWidgets('should present goalkeepers section', (tester) async {
  await tester.pumpWidget(sut);

  presenter.emitNextEventWith(goalkeepers: [
    NextEventPlayerViewModel(name: 'Rodrigo', initials: anyString()),
    NextEventPlayerViewModel(name: 'Rafael', initials: anyString()),
    NextEventPlayerViewModel(name: 'Pedro', initials: anyString())
  ]);
  await tester.pump();

  // Check section header
  expect(find.text('DENTRO - GOLEIROS'), findsOneWidget);
  expect(find.text('3'), findsOneWidget);  // Player count

  // Check player names
  expect(find.text('Rodrigo'), findsOneWidget);
  expect(find.text('Rafael'), findsOneWidget);
  expect(find.text('Pedro'), findsOneWidget);

  // Check components are rendered
  expect(find.byType(PlayerPosition), findsExactly(3));
  expect(find.byType(PlayerStatus), findsExactly(3));
  expect(find.byType(PlayerPhoto), findsExactly(3));
});
```

### Testing Empty State

```dart
testWidgets('should hide all sections', (tester) async {
  await tester.pumpWidget(sut);

  presenter.emitNextEvent();  // Empty viewmodel
  await tester.pump();

  expect(find.text('DENTRO - GOLEIROS'), findsNothing);
  expect(find.text('DENTRO - JOGADORES'), findsNothing);
  expect(find.text('FORA'), findsNothing);
  expect(find.text('DÚVIDA'), findsNothing);
  expect(find.byType(PlayerPosition), findsNothing);
  expect(find.byType(PlayerStatus), findsNothing);
  expect(find.byType(PlayerPhoto), findsNothing);
});
```

### Testing Error State

```dart
testWidgets('should present error message on load error', (tester) async {
  await tester.pumpWidget(sut);

  presenter.emitError();
  await tester.pump();

  // Sections hidden
  expect(find.text('DENTRO - GOLEIROS'), findsNothing);
  expect(find.text('DENTRO - JOGADORES'), findsNothing);
  expect(find.text('FORA'), findsNothing);
  expect(find.text('DÚVIDA'), findsNothing);

  // Error UI shown
  expect(find.text('Algo errado aconteceu, tente novamente.'), findsOneWidget);
  expect(find.text('RECARREGAR'), findsOneWidget);
});
```

---

## Testing User Interactions

### Tap Interaction

```dart
testWidgets('should load event data on reload click', (tester) async {
  await tester.pumpWidget(sut);

  // Initial load
  expect(presenter.callsCount, 1);
  expect(presenter.groupId, groupId);
  expect(presenter.isReload, false);

  // Show error state with reload button
  presenter.emitError();
  await tester.pump();

  // Tap reload button
  await tester.tap(find.text('RECARREGAR'));

  // Verify reload was called
  expect(presenter.callsCount, 2);
  expect(presenter.groupId, groupId);
  expect(presenter.isReload, true);
});
```

### Testing Busy State

```dart
testWidgets('should handle spinner on page busy event', (tester) async {
  await tester.pumpWidget(sut);

  // Show error state
  presenter.emitError();
  await tester.pump();

  // Emit busy state
  presenter.emitIsBusy();
  await tester.pump();
  expect(find.byType(CircularProgressIndicator), findsOneWidget);

  // Emit not busy
  presenter.emitIsBusy(false);
  await tester.pump();
  expect(find.byType(CircularProgressIndicator), findsNothing);
});
```

---

## Pull-to-Refresh Testing

### Using `flingFrom`

```dart
testWidgets('should load event data on pull to refresh', (tester) async {
  await tester.pumpWidget(sut);

  // Initial state
  expect(presenter.callsCount, 1);
  expect(presenter.groupId, groupId);
  expect(presenter.isReload, false);

  // Load data first (refresh needs content to pull)
  presenter.emitNextEvent();
  await tester.pump();

  // Simulate pull-to-refresh gesture
  await tester.flingFrom(
    const Offset(50, 100),   // Start position
    const Offset(0, 400),    // Drag direction (down)
    800                       // Speed
  );
  await tester.pumpAndSettle();  // Wait for animations

  // Verify reload was triggered
  expect(presenter.callsCount, 2);
  expect(presenter.groupId, groupId);
  expect(presenter.isReload, true);
});
```

### Understanding `flingFrom`

```dart
await tester.flingFrom(
  const Offset(50, 100),  // Starting point (x, y)
  const Offset(0, 400),   // Direction and distance
  800                     // Velocity (pixels/second)
);
```

- Start at `(50, 100)` - near top of screen
- Move `(0, 400)` - 400 pixels down
- Speed `800` - fast enough to trigger refresh

---

## Widget Finding Strategies

### Common Finders

| Finder | Usage |
|--------|-------|
| `find.text('Hello')` | Find by text content |
| `find.byType(Widget)` | Find by widget type |
| `find.byKey(Key('id'))` | Find by key |
| `find.byIcon(Icons.add)` | Find by icon |
| `find.byWidget(widget)` | Find exact widget instance |
| `find.ancestor()` | Find ancestor of widget |
| `find.descendant()` | Find descendant of widget |

### Common Matchers

| Matcher | Usage |
|---------|-------|
| `findsOneWidget` | Exactly one match |
| `findsNothing` | No matches |
| `findsNWidgets(n)` | Exactly n matches |
| `findsExactly(n)` | Same as findsNWidgets |
| `findsAtLeastNWidgets(n)` | At least n matches |
| `findsWidgets` | One or more matches |

### Examples

```dart
// Find by text
expect(find.text('Hello'), findsOneWidget);

// Find by type
expect(find.byType(CircularProgressIndicator), findsNothing);

// Find exact count
expect(find.byType(PlayerStatus), findsExactly(3));

// Find by key
expect(find.byKey(const Key('reload_button')), findsOneWidget);
```

---

## `pump` vs `pumpAndSettle`

| Method | Use When |
|--------|----------|
| `pump()` | Trigger single frame rebuild |
| `pump(duration)` | Advance by specific time |
| `pumpAndSettle()` | Wait for all animations to complete |

### When to Use Each

```dart
// Simple state change
presenter.emitNextEvent();
await tester.pump();  // One frame is enough

// Animation (pull-to-refresh, page transitions)
await tester.flingFrom(...);
await tester.pumpAndSettle();  // Wait for animation
```

---

## Complete Page Test Example

```dart
import 'package:advanced_flutter/presentation/viewmodels/next_event_player_viewmodel.dart';
import 'package:advanced_flutter/ui/components/player_photo.dart';
import 'package:advanced_flutter/ui/components/player_position.dart';
import 'package:advanced_flutter/ui/components/player_status.dart';
import 'package:advanced_flutter/ui/pages/next_event_page.dart';

import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

import '../../mocks/fakes.dart';
import '../../presentation/mocks/next_event_presenter_spy.dart';

void main() {
  late NextEventPresenterSpy presenter;
  late String groupId;
  late Widget sut;

  setUp(() {
    presenter = NextEventPresenterSpy();
    groupId = anyString();
    sut = MaterialApp(
      home: NextEventPage(presenter: presenter, groupId: groupId)
    );
  });

  testWidgets('should load event data on page init', (tester) async {
    await tester.pumpWidget(sut);

    expect(presenter.callsCount, 1);
    expect(presenter.groupId, groupId);
    expect(presenter.isReload, false);
  });

  testWidgets('should present spinner while data is loading', (tester) async {
    await tester.pumpWidget(sut);

    expect(find.byType(CircularProgressIndicator), findsOneWidget);
  });

  testWidgets('should hide spinner on load success', (tester) async {
    await tester.pumpWidget(sut);

    presenter.emitNextEvent();
    await tester.pump();

    expect(find.byType(CircularProgressIndicator), findsNothing);
  });

  testWidgets('should present players section', (tester) async {
    await tester.pumpWidget(sut);

    presenter.emitNextEventWith(players: [
      NextEventPlayerViewModel(name: 'Rodrigo', initials: 'RO'),
      NextEventPlayerViewModel(name: 'Rafael', initials: 'RA'),
      NextEventPlayerViewModel(name: 'Pedro', initials: 'PE')
    ]);
    await tester.pump();

    expect(find.text('DENTRO - JOGADORES'), findsOneWidget);
    expect(find.text('3'), findsOneWidget);
    expect(find.text('Rodrigo'), findsOneWidget);
    expect(find.text('Rafael'), findsOneWidget);
    expect(find.text('Pedro'), findsOneWidget);
  });

  testWidgets('should load event data on pull to refresh', (tester) async {
    await tester.pumpWidget(sut);

    presenter.emitNextEvent();
    await tester.pump();

    await tester.flingFrom(const Offset(50, 100), const Offset(0, 400), 800);
    await tester.pumpAndSettle();

    expect(presenter.callsCount, 2);
    expect(presenter.isReload, true);
  });
}
```

---

## Widget Testing Best Practices

| Practice | Description |
|----------|-------------|
| Wrap with MaterialApp | Required for Material widgets |
| Use `mockNetworkImagesFor` | For network images |
| `pump()` for state changes | Single frame rebuild |
| `pumpAndSettle()` for animations | Wait for completion |
| Test all states | Loading, success, error, empty |
| Test interactions | Tap, scroll, refresh |

---

## Next Steps

Now that you understand widget testing, proceed to:

**[→ End-to-End Testing](07_E2E_TESTING.md)**: Learn how to test complete user flows.
