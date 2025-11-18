# Getting Started Guide

## Table of Contents
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Project Setup](#project-setup)
- [Understanding the Codebase](#understanding-the-codebase)
- [Following a Feature End-to-End](#following-a-feature-end-to-end)
- [How to Add a New Feature](#how-to-add-a-new-feature)
- [Common Tasks](#common-tasks)
- [Working with Code Generation](#working-with-code-generation)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)
- [Learning Resources](#learning-resources)

---

## Introduction

This guide will help you **get started** working with this Flutter e-commerce project. Whether you're:
- A beginner learning Flutter and clean architecture
- An experienced developer exploring this codebase
- Someone looking to add features or fix bugs

This guide has you covered!

### What You'll Learn

1. How to set up and run the project
2. How to navigate the codebase effectively
3. How to trace a feature from UI to data layer
4. How to add new features following project patterns
5. Common tasks and workflows
6. Testing strategies

---

## Prerequisites

### Required Knowledge

**Minimum:**
- Basic Dart programming
- Basic Flutter widgets (StatelessWidget, StatefulWidget)
- Understanding of async/await

**Recommended:**
- Flutter state management concepts
- Object-oriented programming
- Git basics

**Nice to Have:**
- Clean Architecture concepts
- Riverpod experience
- Testing experience

### Required Tools

1. **Flutter SDK** (latest stable version)
   ```bash
   flutter --version
   # Should show Flutter 3.x or higher
   ```

2. **Dart SDK** (comes with Flutter)
   ```bash
   dart --version
   ```

3. **IDE** (choose one):
   - Visual Studio Code with Flutter extension
   - Android Studio with Flutter plugin
   - IntelliJ IDEA with Flutter plugin

4. **Git**
   ```bash
   git --version
   ```

---

## Project Setup

### Step 1: Clone the Repository

```bash
git clone <repository-url>
cd ecommerce_app
```

### Step 2: Install Dependencies

```bash
flutter pub get
```

This downloads all packages listed in `pubspec.yaml`.

### Step 3: Generate Code

The project uses code generation for Riverpod providers:

```bash
# One-time generation
dart run build_runner build --delete-conflicting-outputs

# Or watch mode (auto-regenerates on file changes)
dart run build_runner watch --delete-conflicting-outputs
```

**What this does:**
- Generates `*.g.dart` files from `@riverpod` annotations
- Creates provider code automatically
- Must be run whenever you add/modify providers

### Step 4: Run the App

```bash
# Run on connected device/emulator
flutter run

# Or run on specific device
flutter devices  # List devices
flutter run -d <device-id>

# Or run on Chrome (web)
flutter run -d chrome
```

### Step 5: Run Tests

```bash
# Run all tests
flutter test

# Run specific test file
flutter test test/src/features/cart/domain/mutable_cart_test.dart

# Run with coverage
flutter test --coverage
```

---

## Understanding the Codebase

### Where to Start?

When exploring a new codebase, start from the **entry point** and work outward:

#### 1. Start with main.dart

**File:** [lib/main.dart](lib/main.dart)

```dart
void main() {
  runApp(const ProviderScope(child: MyApp()));
}
```

**Key observations:**
- `ProviderScope` wraps the app (required for Riverpod)
- `MyApp` is the root widget

#### 2. Look at MyApp (Root Widget)

**File:** [lib/src/app.dart](lib/src/app.dart:10-35)

```dart
class MyApp extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Get router from provider
    final goRouter = ref.watch(goRouterProvider);

    return MaterialApp.router(
      routerConfig: goRouter,
      // Theme, localization, etc.
    );
  }
}
```

**Key observations:**
- Uses `ConsumerWidget` to watch providers
- Uses `MaterialApp.router` with GoRouter
- Router configuration is in a provider

#### 3. Explore Router Configuration

**File:** [lib/src/routing/app_router.dart](lib/src/routing/app_router.dart:15-150)

```dart
@Riverpod(keepAlive: true)
GoRouter goRouter(Ref ref) {
  return GoRouter(
    initialLocation: '/',
    routes: [
      GoRoute(
        path: '/',
        name: AppRoute.home.name,
        builder: (context, state) => const ProductsListScreen(),
        // Nested routes...
      ),
    ],
  );
}
```

**Key observations:**
- All routes defined here
- Initial route is `/` (home = products list)
- Routes use named constants from `AppRoute` enum

#### 4. Find the Home Screen

**File:** [lib/src/features/products/presentation/products_list/products_list_screen.dart](lib/src/features/products/presentation/products_list/products_list_screen.dart:15-50)

```dart
class ProductsListScreen extends StatefulWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: HomeAppBar(/* search functionality */),
      body: Consumer(
        builder: (context, ref, child) {
          // Watch products provider
          final productsValue = ref.watch(productsListSearchProvider(_searchQuery));

          // Build UI based on state
          return AsyncValueWidget(/* ... */);
        },
      ),
    );
  }
}
```

**Key observations:**
- Uses `Consumer` to watch providers
- Provider name: `productsListSearchProvider`
- Uses `AsyncValueWidget` to handle loading/error/data states

#### 5. Trace Provider Back to Source

**Provider chain:**
```
productsListSearchProvider (presentation)
         ↓
productsRepositoryProvider (data)
         ↓
FakeProductsRepository (data implementation)
         ↓
_allProducts (hardcoded data)
```

### Directory Navigation Map

```
lib/src/
│
├── app.dart                    # START HERE - Root widget
│
├── routing/                    # Navigation configuration
│   └── app_router.dart         # All routes defined here
│
├── features/                   # Main features
│   ├── authentication/         # User sign-in/sign-out
│   ├── products/              # Product catalog
│   ├── cart/                  # Shopping cart
│   ├── checkout/              # Purchase flow
│   ├── orders/                # Order history
│   └── reviews/               # Product reviews
│
├── common_widgets/            # Reusable UI components
│   ├── async_value_widget.dart    # Handle AsyncValue states
│   ├── responsive_center.dart     # Responsive layout
│   └── ...
│
├── constants/                 # App constants
│   └── app_sizes.dart         # Spacing, sizes
│
├── exceptions/                # Custom exceptions
│   └── app_exception.dart     # Error handling
│
└── utils/                     # Utilities
    └── in_memory_store.dart   # Reactive in-memory storage
```

### Feature Structure Pattern

**Every feature follows this pattern:**

```
feature_name/
├── domain/           # What it is (entities, business rules)
├── data/             # Where it comes from (repositories, APIs)
├── application/      # What to do with it (services, use cases)
└── presentation/     # How to show it (screens, widgets, controllers)
```

### File Naming Conventions

| Type | Convention | Example |
|------|-----------|---------|
| Screen | `*_screen.dart` | `products_list_screen.dart` |
| Controller | `*_controller.dart` | `add_to_cart_controller.dart` |
| Repository | `*_repository.dart` | `fake_products_repository.dart` |
| Service | `*_service.dart` | `cart_service.dart` |
| Widget | Descriptive name | `product_card.dart` |
| Entity | Entity name | `product.dart`, `cart.dart` |
| Provider | Same as function | `cart_service.dart` contains `cartServiceProvider` |

---

## Following a Feature End-to-End

Let's trace the **"Add to Cart"** feature from UI to database and back!

### Step 1: Find the UI (Presentation Layer)

**File:** [lib/src/features/cart/presentation/add_to_cart/add_to_cart_widget.dart](lib/src/features/cart/presentation/add_to_cart/add_to_cart_widget.dart:20-50)

```dart
class AddToCartWidget extends ConsumerWidget {
  final ProductID productId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(addToCartControllerProvider);

    return state.when(
      data: (_) => ElevatedButton(
        onPressed: () {
          // 1. USER ACTION: Tap button
          final controller = ref.read(addToCartControllerProvider.notifier);
          controller.addItem(productId);
        },
        child: Text('Add to Cart'),
      ),
      loading: () => CircularProgressIndicator(),
      error: (error, _) => Text('Error: $error'),
    );
  }
}
```

**What happens here:**
- User taps button
- Controller's `addItem()` method is called
- UI shows loading/success/error based on state

### Step 2: Follow to Controller

**File:** [lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart](lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart:20-60)

```dart
@riverpod
class AddToCartController extends _$AddToCartController {
  @override
  FutureOr<void> build() {}

  Future<void> addItem(ProductID productId) async {
    // 2. CONTROLLER: Manage UI state

    // Get dependencies
    final cartService = ref.read(cartServiceProvider);

    // Set loading state
    state = const AsyncLoading();

    // 3. Call service
    state = await AsyncValue.guard(() async {
      final item = Item(productId: productId, quantity: 1);
      await cartService.addItem(item);
    });
  }
}
```

**What happens here:**
- Controller gets `CartService` from provider
- Sets loading state (UI shows spinner)
- Calls service
- Updates state based on result

### Step 3: Follow to Service (Application Layer)

**File:** [lib/src/features/cart/application/cart_service.dart](lib/src/features/cart/application/cart_service.dart:45-60)

```dart
class CartService {
  final Ref ref;

  Future<void> addItem(Item item) async {
    // 4. SERVICE: Orchestrate business logic

    // Fetch current cart (handles auth state)
    final cart = await _fetchCart();

    // 5. Apply domain logic
    final updatedCart = cart.addItem(item);

    // 6. Save cart
    await _setCart(updatedCart);
  }

  Future<Cart> _fetchCart() {
    final user = ref.read(authStateChangesProvider).value;
    if (user != null) {
      return ref.read(remoteCartRepositoryProvider).fetchCart(user.uid);
    } else {
      return ref.read(localCartRepositoryProvider).fetchCart();
    }
  }

  Future<void> _setCart(Cart cart) async {
    final user = ref.read(authStateChangesProvider).value;
    if (user != null) {
      await ref.read(remoteCartRepositoryProvider).setCart(user.uid, cart);
    } else {
      await ref.read(localCartRepositoryProvider).setCart(cart);
    }
  }
}
```

**What happens here:**
- Service fetches current cart (abstracts auth state)
- Calls domain logic to add item
- Saves updated cart

### Step 4: Follow to Domain Logic

**File:** [lib/src/features/cart/domain/mutable_cart.dart](lib/src/features/cart/domain/mutable_cart.dart:5-30)

```dart
extension MutableCart on Cart {
  Cart addItem(Item item) {
    // 5. DOMAIN: Business rules

    final copy = Map<String, Item>.from(items);
    final currentItem = copy[item.productId];

    if (currentItem != null) {
      // Item exists: increase quantity (business rule!)
      copy[item.productId] = Item(
        productId: item.productId,
        quantity: currentItem.quantity + item.quantity,
      );
    } else {
      // New item: add to cart
      copy[item.productId] = item;
    }

    return Cart(copy);  // Return new immutable cart
  }
}
```

**What happens here:**
- Pure business logic (no dependencies!)
- Returns new Cart (immutable)
- Business rule: if item exists, increase quantity

### Step 5: Follow to Repository (Data Layer)

**File:** [lib/src/features/cart/data/local/sembast_cart_repository.dart](lib/src/features/cart/data/local/sembast_cart_repository.dart:55-65)

```dart
class SembastCartRepository implements LocalCartRepository {
  final Database db;

  @override
  Future<void> setCart(Cart cart) async {
    // 6. REPOSITORY: Save to database

    // Convert to JSON and save
    await store.record(cartItemsKey).put(db, cart.toJson());

    // Stream automatically emits new value!
  }
}
```

**What happens here:**
- Cart converted to JSON
- Saved to Sembast database
- Database stream emits new cart

### Step 6: Follow Back to UI (Reactive Update)

**File:** [lib/src/features/cart/presentation/shopping_cart/shopping_cart_screen.dart](lib/src/features/cart/presentation/shopping_cart/shopping_cart_screen.dart:20-40)

```dart
class ShoppingCartScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // 7. UI: Watch cart provider
    final cartValue = ref.watch(cartProvider);

    // Automatically rebuilds when cart changes!
    return cartValue.when(
      data: (cart) => Text('${cart.items.length} items'),
      loading: () => CircularProgressIndicator(),
      error: (error, _) => Text('Error: $error'),
    );
  }
}
```

**What happens here:**
- UI watches `cartProvider`
- Provider watches database stream
- When cart changes, UI rebuilds automatically!

### Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────┐
│ 1. USER TAPS "Add to Cart" button                      │
│    (add_to_cart_widget.dart)                           │
└────────────────┬────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 2. CONTROLLER: addItem()                               │
│    - Sets loading state                                │
│    - Calls CartService                                 │
│    (add_to_cart_controller.dart)                       │
└────────────────┬────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 3. SERVICE: addItem()                                  │
│    - Fetches current cart                              │
│    - Calls domain logic                                │
│    - Saves updated cart                                │
│    (cart_service.dart)                                 │
└────────────────┬────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 4. DOMAIN: cart.addItem()                              │
│    - Applies business rules                            │
│    - Returns new immutable cart                        │
│    (mutable_cart.dart)                                 │
└────────────────┬────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 5. REPOSITORY: setCart()                               │
│    - Saves to database                                 │
│    - Stream emits new cart                             │
│    (sembast_cart_repository.dart)                      │
└────────────────┬────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 6. PROVIDER: cartProvider emits                        │
│    - All listeners notified                            │
└────────────────┬────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 7. UI: Automatically rebuilds                          │
│    - Cart icon badge updates                           │
│    - Shopping cart screen updates                      │
│    - Cart total updates                                │
└─────────────────────────────────────────────────────────┘
```

---

## How to Add a New Feature

Let's add a **"Favorites"** feature step-by-step!

### Step 1: Plan the Feature

**Requirements:**
- Users can mark products as favorites
- View list of favorite products
- Store favorites locally

**Architecture:**
```
favorites/
├── domain/
│   └── favorite.dart          # Favorite entity
├── data/
│   └── local_favorites_repository.dart
├── application/
│   └── favorites_service.dart
└── presentation/
    ├── favorites_button/
    └── favorites_list/
```

### Step 2: Create Domain Layer

**File:** `lib/src/features/favorites/domain/favorite.dart`

```dart
import 'package:flutter/foundation.dart';
import '../../products/domain/product.dart';

/// Favorite entity
@immutable
class Favorite {
  const Favorite({
    required this.productId,
    required this.addedAt,
  });

  final ProductID productId;
  final DateTime addedAt;

  // Serialization
  factory Favorite.fromJson(Map<String, dynamic> json) {
    return Favorite(
      productId: json['productId'] as String,
      addedAt: DateTime.parse(json['addedAt'] as String),
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'productId': productId,
      'addedAt': addedAt.toIso8601String(),
    };
  }

  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is Favorite && other.productId == productId;
  }

  @override
  int get hashCode => productId.hashCode;
}

/// Favorites collection
@immutable
class Favorites {
  const Favorites([this.items = const []]);

  final List<Favorite> items;

  // Business logic via extensions
}

extension MutableFavorites on Favorites {
  /// Add to favorites
  Favorites add(Favorite favorite) {
    final copy = List<Favorite>.from(items);

    // Don't add duplicates
    if (!copy.any((f) => f.productId == favorite.productId)) {
      copy.add(favorite);
    }

    return Favorites(copy);
  }

  /// Remove from favorites
  Favorites remove(ProductID productId) {
    final copy = List<Favorite>.from(items);
    copy.removeWhere((f) => f.productId == productId);
    return Favorites(copy);
  }

  /// Check if product is favorited
  bool isFavorite(ProductID productId) {
    return items.any((f) => f.productId == productId);
  }
}
```

### Step 3: Create Data Layer

**File:** `lib/src/features/favorites/data/local_favorites_repository.dart`

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:sembast/sembast.dart';
import '../domain/favorite.dart';
import '../../products/domain/product.dart';

part 'local_favorites_repository.g.dart';

/// Abstract interface
abstract class LocalFavoritesRepository {
  Future<Favorites> fetchFavorites();
  Stream<Favorites> watchFavorites();
  Future<void> setFavorites(Favorites favorites);
}

/// Sembast implementation
class SembastFavoritesRepository implements LocalFavoritesRepository {
  SembastFavoritesRepository(this.db);

  final Database db;
  final store = StoreRef.main();
  static const favoritesKey = 'favorites';

  @override
  Future<Favorites> fetchFavorites() async {
    final json = await store.record(favoritesKey).get(db);
    if (json != null) {
      final items = (json as List)
          .map((item) => Favorite.fromJson(item as Map<String, dynamic>))
          .toList();
      return Favorites(items);
    }
    return const Favorites();
  }

  @override
  Stream<Favorites> watchFavorites() {
    return store.record(favoritesKey).onSnapshot(db).map((snapshot) {
      if (snapshot != null) {
        final items = (snapshot.value as List)
            .map((item) => Favorite.fromJson(item as Map<String, dynamic>))
            .toList();
        return Favorites(items);
      }
      return const Favorites();
    });
  }

  @override
  Future<void> setFavorites(Favorites favorites) async {
    final json = favorites.items.map((f) => f.toJson()).toList();
    await store.record(favoritesKey).put(db, json);
  }
}

/// Provider
@Riverpod(keepAlive: true)
LocalFavoritesRepository localFavoritesRepository(Ref ref) {
  final db = ref.watch(sembastDatabaseProvider).requireValue;
  return SembastFavoritesRepository(db);
}

@Riverpod(keepAlive: true)
Stream<Favorites> favorites(Ref ref) {
  return ref.watch(localFavoritesRepositoryProvider).watchFavorites();
}
```

### Step 4: Create Application Layer

**File:** `lib/src/features/favorites/application/favorites_service.dart`

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
import '../domain/favorite.dart';
import '../data/local_favorites_repository.dart';
import '../../products/domain/product.dart';

part 'favorites_service.g.dart';

class FavoritesService {
  FavoritesService(this.ref);

  final Ref ref;

  /// Toggle favorite
  Future<void> toggleFavorite(ProductID productId) async {
    final repository = ref.read(localFavoritesRepositoryProvider);
    final favorites = await repository.fetchFavorites();

    final updated = favorites.isFavorite(productId)
        ? favorites.remove(productId)
        : favorites.add(Favorite(
            productId: productId,
            addedAt: DateTime.now(),
          ));

    await repository.setFavorites(updated);
  }

  /// Add to favorites
  Future<void> addFavorite(ProductID productId) async {
    final repository = ref.read(localFavoritesRepositoryProvider);
    final favorites = await repository.fetchFavorites();

    final updated = favorites.add(Favorite(
      productId: productId,
      addedAt: DateTime.now(),
    ));

    await repository.setFavorites(updated);
  }

  /// Remove from favorites
  Future<void> removeFavorite(ProductID productId) async {
    final repository = ref.read(localFavoritesRepositoryProvider);
    final favorites = await repository.fetchFavorites();

    final updated = favorites.remove(productId);

    await repository.setFavorites(updated);
  }
}

@Riverpod(keepAlive: true)
FavoritesService favoritesService(Ref ref) {
  return FavoritesService(ref);
}
```

### Step 5: Create Presentation Layer

**File:** `lib/src/features/favorites/presentation/favorite_button/favorite_button.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../products/domain/product.dart';
import '../../data/local_favorites_repository.dart';
import '../../application/favorites_service.dart';

class FavoriteButton extends ConsumerWidget {
  const FavoriteButton({required this.productId, super.key});

  final ProductID productId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final favoritesValue = ref.watch(favoritesProvider);

    return favoritesValue.when(
      data: (favorites) {
        final isFavorite = favorites.isFavorite(productId);

        return IconButton(
          icon: Icon(
            isFavorite ? Icons.favorite : Icons.favorite_border,
            color: isFavorite ? Colors.red : null,
          ),
          onPressed: () {
            final service = ref.read(favoritesServiceProvider);
            service.toggleFavorite(productId);
          },
        );
      },
      loading: () => IconButton(
        icon: Icon(Icons.favorite_border),
        onPressed: null,
      ),
      error: (_, __) => IconButton(
        icon: Icon(Icons.error),
        onPressed: null,
      ),
    );
  }
}
```

### Step 6: Run Code Generation

```bash
dart run build_runner build --delete-conflicting-outputs
```

This generates:
- `local_favorites_repository.g.dart`
- `favorites_service.g.dart`

### Step 7: Add to Product Card

**File:** `lib/src/features/products/presentation/products_list/product_card.dart`

```dart
// Add favorite button to product card
class ProductCard extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Card(
      child: Column(
        children: [
          // ... existing code

          // Add favorite button
          Positioned(
            top: 8,
            right: 8,
            child: FavoriteButton(productId: product.id),
          ),
        ],
      ),
    );
  }
}
```

### Step 8: Add Favorites Screen

Create screen to show all favorites, add route, etc.

**You've just added a complete feature following the project's architecture!**

---

## Common Tasks

### Task 1: Adding a New Screen

1. Create screen file in appropriate feature folder
2. Add route to `app_router.dart`
3. Navigate using `context.goNamed()` or `context.go()`

**Example:**

```dart
// 1. Create screen
class NewScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('New Screen')),
      body: Center(child: Text('Hello!')),
    );
  }
}

// 2. Add to router
GoRoute(
  path: '/new',
  name: AppRoute.newScreen.name,
  builder: (context, state) => NewScreen(),
)

// 3. Navigate
context.goNamed(AppRoute.newScreen.name);
```

### Task 2: Creating a New Provider

1. Add `@riverpod` annotation
2. Define function with `Ref` parameter
3. Run code generation

**Example:**

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'my_provider.g.dart';

@riverpod
String greeting(Ref ref) {
  return 'Hello, World!';
}

// Then run:
// dart run build_runner build --delete-conflicting-outputs
```

### Task 3: Modifying Business Logic

1. Find the domain layer extension methods
2. Add new method or modify existing
3. Update service if needed
4. Update UI if needed

**Example: Add "clear cart" functionality:**

```dart
// 1. Add to domain layer
extension MutableCart on Cart {
  Cart clear() {
    return const Cart();  // Empty cart
  }
}

// 2. Add to service
class CartService {
  Future<void> clearCart() async {
    await _setCart(const Cart());
  }
}

// 3. Add to controller
@riverpod
class CartController extends _$CartController {
  Future<void> clear() async {
    final service = ref.read(cartServiceProvider);
    state = const AsyncLoading();
    state = await AsyncValue.guard(() => service.clearCart());
  }
}

// 4. Use in UI
ElevatedButton(
  onPressed: () {
    ref.read(cartControllerProvider.notifier).clear();
  },
  child: Text('Clear Cart'),
)
```

### Task 4: Adding Validation

Add validation in the controller before calling service:

```dart
@riverpod
class SignUpController extends _$SignUpController {
  Future<void> signUp(String email, String password) async {
    // Validation
    if (email.isEmpty) {
      state = AsyncError(Exception('Email is required'), StackTrace.current);
      return;
    }

    if (password.length < 6) {
      state = AsyncError(
        Exception('Password must be at least 6 characters'),
        StackTrace.current,
      );
      return;
    }

    // Proceed with sign up
    state = const AsyncLoading();
    // ...
  }
}
```

---

## Working with Code Generation

### When to Run Code Generation

Run `dart run build_runner build --delete-conflicting-outputs` when:
- ✅ You add a new `@riverpod` annotation
- ✅ You modify a provider function signature
- ✅ You add a new provider parameter
- ✅ You get errors about missing `.g.dart` files

### Watch Mode for Development

```bash
# Run in a separate terminal
dart run build_runner watch --delete-conflicting-outputs
```

This automatically regenerates code when you save files.

### Understanding Generated Files

**Your code:**
```dart
@riverpod
String greeting(Ref ref) {
  return 'Hello!';
}
```

**Generated code** (`*.g.dart`):
```dart
final greetingProvider = Provider<String>(
  (ref) => greeting(ref),
  name: r'greetingProvider',
);
```

**Don't edit generated files!** They'll be overwritten.

---

## Testing

### Test Structure

Tests mirror the `lib/` structure:

```
test/
└── src/
    ├── features/
    │   ├── authentication/
    │   │   └── auth_flow_test.dart
    │   ├── cart/
    │   │   └── cart_test.dart
    │   └── ...
    ├── robot.dart        # Test helpers
    └── mocks.dart        # Mock objects
```

### Writing a Simple Test

**File:** `test/src/features/cart/domain/mutable_cart_test.dart`

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:ecommerce_app/src/features/cart/domain/cart.dart';
import 'package:ecommerce_app/src/features/cart/domain/item.dart';
import 'package:ecommerce_app/src/features/cart/domain/mutable_cart.dart';

void main() {
  group('Cart', () {
    test('addItem adds new item', () {
      // Arrange
      const cart = Cart();
      const item = Item(productId: '1', quantity: 1);

      // Act
      final result = cart.addItem(item);

      // Assert
      expect(result.items.length, 1);
      expect(result.items['1']?.quantity, 1);
    });

    test('addItem increases quantity if item exists', () {
      // Arrange
      final cart = Cart({'1': Item(productId: '1', quantity: 1)});
      const item = Item(productId: '1', quantity: 1);

      // Act
      final result = cart.addItem(item);

      // Assert
      expect(result.items.length, 1);
      expect(result.items['1']?.quantity, 2);
    });
  });
}
```

### Using the Robot Pattern

**File:** `test/src/features/purchase_flow_test.dart`

```dart
import 'package:flutter_test/flutter_test.dart';
import 'robot.dart';

void main() {
  testWidgets('Complete purchase flow', (tester) async {
    final r = Robot(tester);

    await r.pumpMyApp();

    // Sign in
    await r.auth.openSignInScreen();
    await r.auth.signInWithEmailAndPassword(
      email: 'test@test.com',
      password: 'test1234',
    );
    r.auth.expectSignedIn();

    // Add to cart
    r.products.expectFindAllProductCards();
    await r.products.selectProduct();
    await r.cart.addToCart();
    r.cart.expectItemCount(1);

    // Checkout
    await r.cart.openCart();
    await r.checkout.startCheckout();
    await r.checkout.completePayment();

    // Verify
    r.checkout.expectOrderConfirmation();
  });
}
```

### Running Tests

```bash
# All tests
flutter test

# Specific file
flutter test test/src/features/cart/domain/mutable_cart_test.dart

# With coverage
flutter test --coverage

# Watch mode
flutter test --watch
```

---

## Troubleshooting

### Issue: "No provider found for..."

**Problem:** Trying to read a provider that doesn't exist or isn't in scope.

**Solution:**
1. Make sure you ran code generation: `dart run build_runner build`
2. Check that the provider file has `part 'file_name.g.dart';`
3. Ensure `ProviderScope` wraps your app in `main.dart`

### Issue: Generated files out of sync

**Problem:** Code works but you get analyzer errors about `.g.dart` files.

**Solution:**
```bash
dart run build_runner clean
dart run build_runner build --delete-conflicting-outputs
```

### Issue: "Bad state: No element"

**Problem:** Using `firstWhere` without `orElse` when element might not exist.

**Solution:**
```dart
// Bad
final product = products.firstWhere((p) => p.id == id);  // Throws if not found

// Good
final product = products.firstWhereOrNull((p) => p.id == id);  // Returns null
```

### Issue: UI not updating

**Problem:** Changed data but UI doesn't rebuild.

**Checklist:**
1. ✅ Using `ref.watch()` not `ref.read()`?
2. ✅ Widget extends `ConsumerWidget` or uses `Consumer`?
3. ✅ Provider is stream/future provider?
4. ✅ Data actually changed (check with debugger)?

### Issue: Navigation not working

**Problem:** `context.go()` or `context.goNamed()` does nothing.

**Solution:**
1. Check route is defined in `app_router.dart`
2. Check route name matches `AppRoute` enum
3. Check all required path parameters provided
4. Check redirect logic isn't preventing navigation

---

## Best Practices

### 1. Follow the Layer Rules

❌ **Don't:**
- Import Flutter in domain layer
- Call repository directly from presentation
- Put business logic in widgets

✅ **Do:**
- Keep domain pure Dart
- Use services to coordinate
- Keep widgets dumb (just UI)

### 2. Use Immutable Data

❌ **Don't:**
```dart
class Cart {
  Map<String, Item> items = {};  // Mutable!

  void addItem(Item item) {
    items[item.productId] = item;  // Mutating state
  }
}
```

✅ **Do:**
```dart
@immutable
class Cart {
  const Cart([this.items = const {}]);

  final Map<String, Item> items;  // Immutable
}

extension MutableCart on Cart {
  Cart addItem(Item item) {
    final copy = Map.from(items);
    copy[item.productId] = item;
    return Cart(copy);  // Return new instance
  }
}
```

### 3. Handle Errors Properly

❌ **Don't:**
```dart
Future<void> addItem(Item item) async {
  await service.addItem(item);  // Unhandled errors!
}
```

✅ **Do:**
```dart
Future<void> addItem(Item item) async {
  state = const AsyncLoading();

  state = await AsyncValue.guard(() async {
    await service.addItem(item);
  });

  // AsyncValue automatically captures errors
}
```

### 4. Use Providers for Dependencies

❌ **Don't:**
```dart
class MyWidget extends StatelessWidget {
  final ProductsRepository repository = ProductsRepository();  // Creating directly

  @override
  Widget build(BuildContext context) {
    // ...
  }
}
```

✅ **Do:**
```dart
class MyWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final repository = ref.watch(productsRepositoryProvider);  // From provider
    // ...
  }
}
```

### 5. Keep Controllers Thin

❌ **Don't:**
```dart
class AddToCartController extends _$AddToCartController {
  Future<void> addItem(ProductID productId) async {
    // Fetching data
    final localRepo = ref.read(localCartRepositoryProvider);
    final cart = await localRepo.fetchCart();

    // Business logic
    final copy = Map.from(cart.items);
    copy[productId] = Item(productId: productId, quantity: 1);
    final updated = Cart(copy);

    // Saving
    await localRepo.setCart(updated);
  }
}
```

✅ **Do:**
```dart
class AddToCartController extends _$AddToCartController {
  Future<void> addItem(ProductID productId) async {
    // Delegate to service
    final service = ref.read(cartServiceProvider);
    state = const AsyncLoading();
    state = await AsyncValue.guard(() => service.addItem(item));
  }
}
```

---

## Learning Resources

### Official Documentation

- **Flutter**: https://flutter.dev/docs
- **Riverpod**: https://riverpod.dev
- **GoRouter**: https://pub.dev/packages/go_router
- **Sembast**: https://pub.dev/packages/sembast

### Architecture & Patterns

- **Clean Architecture**: "Clean Architecture" by Robert C. Martin
- **Domain-Driven Design**: "Domain-Driven Design" by Eric Evans
- **Flutter Architecture**: https://codewithandrea.com/articles/flutter-app-architecture/

### Video Tutorials

- **Riverpod Course**: https://codewithandrea.com/courses/complete-flutter-development-bootcamp-with-riverpod/
- **Flutter Architecture**: https://www.youtube.com/c/ResoCoder
- **Flutter Testing**: https://www.youtube.com/c/FilledStacks

### Practice Projects

1. Start by modifying this project
2. Add new features (favorites, wish list, product comparisons)
3. Build a similar app in a different domain (social media, todo, etc.)
4. Contribute to open-source Flutter projects

---

## Summary

You now have everything you need to:
- ✅ Set up and run the project
- ✅ Navigate the codebase
- ✅ Understand how features work
- ✅ Add new features
- ✅ Write tests
- ✅ Follow best practices

### Next Steps

1. **Explore**: Run the app and click around
2. **Trace**: Pick a feature and trace it through all layers
3. **Modify**: Change something small (button text, color)
4. **Add**: Create a simple new feature (like favorites above)
5. **Test**: Write tests for your new feature
6. **Share**: Show others what you built!

---

**Happy Coding!** 🚀

If you get stuck, remember:
- Check the other documentation files
- Read the code comments
- Use the debugger
- Ask questions in the community

The best way to learn is by doing. Start small, experiment, and build something awesome!
