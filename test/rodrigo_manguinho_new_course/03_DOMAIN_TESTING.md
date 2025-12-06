# Domain Layer Testing

## Introduction

The **Domain Layer** contains your application's core business logic. This includes:
- **Entities**: Core business objects with behavior
- **Use Cases**: Application-specific business rules
- **Value Objects**: Immutable objects defined by their values

Domain tests are the simplest because they test **pure functions** with no external dependencies (no databases, no APIs, no UI).

---

## Table of Contents

1. [What to Test in Domain Layer](#what-to-test-in-domain-layer)
2. [Entity Testing](#entity-testing)
3. [Helper Function Pattern](#helper-function-pattern)
4. [Testing Multiple Scenarios](#testing-multiple-scenarios)
5. [Edge Cases](#edge-cases)
6. [Complete Example](#complete-example)

---

## What to Test in Domain Layer

| Element | What to Test |
|---------|--------------|
| **Entity properties** | Computed properties, derived values |
| **Entity methods** | Business logic methods |
| **Validation rules** | Input validation, constraints |
| **Edge cases** | Empty values, null, boundaries |
| **Value transformations** | Formatting, calculations |

---

## Entity Testing

### The Entity Under Test

Here's a `NextEventPlayer` entity with a computed `initials` property:

```dart
final class NextEventPlayer {
  final String id;
  final String name;
  final String initials;  // Computed from name
  final String? photo;
  final String? position;
  final bool isConfirmed;
  final DateTime? confirmationDate;

  const NextEventPlayer._({
    required this.id,
    required this.name,
    required this.initials,
    required this.isConfirmed,
    this.photo,
    this.position,
    this.confirmationDate
  });

  factory NextEventPlayer({
    required String id,
    required String name,
    required bool isConfirmed,
    String? photo,
    String? position,
    DateTime? confirmationDate
  }) => NextEventPlayer._(
    id: id,
    name: name,
    photo: photo,
    position: position,
    initials: _getInitials(name),  // Computed here
    isConfirmed: isConfirmed,
    confirmationDate: confirmationDate
  );

  // The business logic we want to test
  static String _getInitials(String name) {
    final names = name.toUpperCase().trim().split(' ');
    final firstChar = names.first.split('').firstOrNull ?? '-';
    final lastChar = names.last.split('').elementAtOrNull(
      names.length == 1 ? 1 : 0
    ) ?? '';
    return '$firstChar$lastChar';
  }
}
```

### The Business Rules

The `initials` property has these rules:
1. Return first letter of first name + first letter of last name
2. If only one name, return first two letters
3. If only one letter, return just that letter
4. If empty, return "-"
5. Always uppercase
6. Ignore extra whitespace

---

## Helper Function Pattern

Instead of creating a full entity for every test, use a **helper function** to reduce boilerplate:

### Without Helper (Verbose)

```dart
test('should return initials', () {
  final player1 = NextEventPlayer(
    id: '',
    name: 'Rodrigo Manguinho',
    isConfirmed: true
  );
  expect(player1.initials, 'RM');

  final player2 = NextEventPlayer(
    id: '',
    name: 'Pedro Carvalho',
    isConfirmed: true
  );
  expect(player2.initials, 'PC');
});
```

### With Helper (Clean)

```dart
void main() {
  // Helper function - only needs the property we're testing
  String initialsOf(String name) => NextEventPlayer(
    id: '',
    name: name,
    isConfirmed: true
  ).initials;

  test('should return initials', () {
    expect(initialsOf('Rodrigo Manguinho'), 'RM');
    expect(initialsOf('Pedro Carvalho'), 'PC');
  });
}
```

### Benefits of Helper Functions

1. **Reduces boilerplate**: Don't repeat entity creation
2. **Focus on what matters**: Only pass relevant values
3. **Self-documenting**: `initialsOf('name')` is clear
4. **Easy to add cases**: Just add one line per scenario

---

## Testing Multiple Scenarios

Group related test cases in a single test when they verify the same rule:

```dart
test('should return the first letter of the first and last names', () {
  expect(initialsOf('Rodrigo Manguinho'), 'RM');
  expect(initialsOf('Pedro Carvalho'), 'PC');
  expect(initialsOf('Ingrid Mota da Silva'), 'IS');
});
```

This is acceptable because:
- All assertions test the **same rule**
- If one fails, you know which exact case failed
- Test name describes the rule, not a specific case

---

## Edge Cases

### Single Name

```dart
test('should return the first letters of the first name', () {
  expect(initialsOf('Rodrigo'), 'RO');
  expect(initialsOf('R'), 'R');
});
```

### Empty Input

```dart
test('should return "-" when name is empty', () {
  expect(initialsOf(''), '-');
});
```

### Case Insensitivity

```dart
test('should convert to uppercase', () {
  expect(initialsOf('rodrigo manguinho'), 'RM');
  expect(initialsOf('rodrigo'), 'RO');
  expect(initialsOf('r'), 'R');
});
```

### Whitespace Handling

```dart
test('should ignore extra whitespaces', () {
  expect(initialsOf('Rodrigo Manguinho '), 'RM');   // Trailing
  expect(initialsOf(' Rodrigo Manguinho'), 'RM');   // Leading
  expect(initialsOf('Rodrigo  Manguinho'), 'RM');   // Middle
  expect(initialsOf(' Rodrigo  Manguinho '), 'RM'); // All
  expect(initialsOf(' Rodrigo '), 'RO');            // Single with spaces
  expect(initialsOf(' R '), 'R');                   // One letter with spaces
  expect(initialsOf(' '), '-');                     // Only space
  expect(initialsOf('  '), '-');                    // Multiple spaces
});
```

---

## Complete Example

Here's the complete test file:

```dart
import 'package:advanced_flutter/domain/entities/next_event_player.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  // Helper function for clean, focused tests
  String initialsOf(String name) => NextEventPlayer(
    id: '',
    name: name,
    isConfirmed: true
  ).initials;

  test('should return the first letter of the first and last names', () {
    expect(initialsOf('Rodrigo Manguinho'), 'RM');
    expect(initialsOf('Pedro Carvalho'), 'PC');
    expect(initialsOf('Ingrid Mota da Silva'), 'IS');
  });

  test('should return the first letters of the first name', () {
    expect(initialsOf('Rodrigo'), 'RO');
    expect(initialsOf('R'), 'R');
  });

  test('should return "-" when name is empty', () {
    expect(initialsOf(''), '-');
  });

  test('should convert to uppercase', () {
    expect(initialsOf('rodrigo manguinho'), 'RM');
    expect(initialsOf('rodrigo'), 'RO');
    expect(initialsOf('r'), 'R');
  });

  test('should ignore extra whitespaces', () {
    expect(initialsOf('Rodrigo Manguinho '), 'RM');
    expect(initialsOf(' Rodrigo Manguinho'), 'RM');
    expect(initialsOf('Rodrigo  Manguinho'), 'RM');
    expect(initialsOf(' Rodrigo  Manguinho '), 'RM');
    expect(initialsOf(' Rodrigo '), 'RO');
    expect(initialsOf(' R '), 'R');
    expect(initialsOf(' '), '-');
    expect(initialsOf('  '), '-');
  });
}
```

---

## Test Organization Tips

### When to Use `group()`

Use groups when you have multiple related tests:

```dart
void main() {
  group('initials', () {
    String initialsOf(String name) => NextEventPlayer(
      id: '',
      name: name,
      isConfirmed: true
    ).initials;

    test('should handle full name', () { ... });
    test('should handle single name', () { ... });
    test('should handle empty', () { ... });
  });

  group('confirmationDate', () {
    test('should be null by default', () { ... });
    test('should accept a date', () { ... });
  });
}
```

### When to Keep It Simple

For single computed properties like `initials`, a flat structure is fine.

---

## Key Takeaways

| Principle | Application |
|-----------|-------------|
| **No dependencies** | Domain tests don't need mocks |
| **Helper functions** | Reduce boilerplate, focus on logic |
| **Edge cases first** | Empty, null, boundaries |
| **Multiple assertions** | OK when testing same rule |
| **Descriptive names** | `should [expected behavior]` |

---

## Domain Testing Checklist

- [ ] Test the happy path (normal input)
- [ ] Test edge cases (empty, null, boundaries)
- [ ] Test case sensitivity if relevant
- [ ] Test whitespace handling if relevant
- [ ] Test invalid inputs
- [ ] Use helper functions to reduce boilerplate
- [ ] Group related tests logically

---

## Next Steps

Now that you understand domain testing, proceed to:

**[→ Infrastructure Layer Testing](04_INFRA_TESTING.md)**: Learn how to test repositories, adapters, and mappers with mocked dependencies.
