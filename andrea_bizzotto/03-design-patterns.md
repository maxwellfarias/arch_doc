# Design Patterns

## Table of Contents
- [Introduction](#introduction)
- [Repository Pattern](#repository-pattern)
- [Service Pattern](#service-pattern)
- [Observer Pattern](#observer-pattern)
- [Provider Pattern](#provider-pattern)
- [Extension Methods Pattern](#extension-methods-pattern)
- [Value Object Pattern](#value-object-pattern)
- [Controller Pattern](#controller-pattern)
- [Robot Pattern](#robot-pattern)
- [Factory Pattern](#factory-pattern)
- [Singleton Pattern](#singleton-pattern)
- [Pattern Summary](#pattern-summary)

---

## Introduction

Design patterns are **proven solutions to common problems** in software design. They're like blueprints that you can customize to solve recurring design challenges in your code.

### Why Learn Design Patterns?

Think of design patterns like cooking techniques:
- **Sautéing** is a technique you use for many dishes
- **Folding** egg whites is a technique for making soufflés
- **Braising** is a technique for tender meats

Similarly, design patterns are techniques you apply in different situations:
- **Repository** pattern for data access
- **Observer** pattern for reactive updates
- **Factory** pattern for object creation

### Patterns in This Project

This project uses **10+ design patterns**:

1. **Repository Pattern** - Data access abstraction
2. **Service Pattern** - Business logic orchestration
3. **Observer Pattern** - Reactive data flow
4. **Provider Pattern** - Dependency injection
5. **Extension Methods** - Adding functionality to existing classes
6. **Value Object** - Type-safe identifiers
7. **Controller Pattern** - UI state management
8. **Robot Pattern** - Test automation
9. **Factory Pattern** - Object construction
10. **Singleton Pattern** - Single instance management

Let's explore each one with real examples from the codebase!

---

## Repository Pattern

### What Is It?

The Repository pattern provides a **collection-like interface** for accessing domain objects, hiding the details of data storage.

**Analogy:** Think of a library. You ask the librarian (repository) for a book. You don't need to know:
- Which shelf it's on
- How the books are organized
- Whether it's in the main library or storage

You just ask, "Get me this book" and the librarian handles the rest.

### Why Use It?

✅ **Abstraction**: Hide data source implementation details
✅ **Testability**: Easy to mock repositories in tests
✅ **Flexibility**: Switch data sources without changing business logic
✅ **Centralization**: All data access logic in one place

### Structure

```
┌──────────────────────────────────┐
│  Business Logic / Controllers    │
│  (Depends on interface only)    │
└───────────────┬──────────────────┘
                │
                ↓
┌──────────────────────────────────┐
│  Repository Interface            │
│  abstract class SomeRepository   │
└───────────────┬──────────────────┘
                │
                ↓
┌──────────────────────────────────┐
│  Repository Implementation       │
│  class ConcreteSomeRepository    │
└──────────────────────────────────┘
```

### Example 1: Local Cart Repository

#### Step 1: Define Interface (Contract)

**File:** [lib/src/features/cart/data/local/local_cart_repository.dart](lib/src/features/cart/data/local/local_cart_repository.dart:6-15)

```dart
/// Abstract interface - defines WHAT operations are available
abstract class LocalCartRepository {
  /// Fetch cart once
  Future<Cart> fetchCart();

  /// Watch cart changes (reactive stream)
  Stream<Cart> watchCart();

  /// Save cart
  Future<void> setCart(Cart cart);
}
```

**Key Points:**
- `abstract class` - can't be instantiated directly
- Only method signatures (no implementation)
- Defines the contract that all implementations must follow

#### Step 2: Implement with Sembast

**File:** [lib/src/features/cart/data/local/sembast_cart_repository.dart](lib/src/features/cart/data/local/sembast_cart_repository.dart:15-75)

```dart
/// Concrete implementation using Sembast database
class SembastCartRepository implements LocalCartRepository {
  SembastCartRepository(this.db);

  final Database db;
  final store = StoreRef.main();
  static const cartItemsKey = 'cartItems';

  @override
  Future<Cart> fetchCart() async {
    // Implementation details: reading from Sembast
    final cartItemsJson = await store.record(cartItemsKey).get(db);

    if (cartItemsJson != null) {
      return Cart.fromJson(cartItemsJson as Map<String, dynamic>);
    } else {
      return const Cart();
    }
  }

  @override
  Stream<Cart> watchCart() {
    // Implementation details: creating a stream from Sembast
    final record = store.record(cartItemsKey);

    return record.onSnapshot(db).map((snapshot) {
      if (snapshot != null) {
        return Cart.fromJson(snapshot.value as Map<String, dynamic>);
      } else {
        return const Cart();
      }
    });
  }

  @override
  Future<void> setCart(Cart cart) async {
    // Implementation details: saving to Sembast
    await store.record(cartItemsKey).put(db, cart.toJson());
  }
}
```

#### Step 3: Provide via Riverpod

**File:** [lib/src/features/cart/data/local/local_cart_repository.dart](lib/src/features/cart/data/local/local_cart_repository.dart:17-25)

```dart
@Riverpod(keepAlive: true)
LocalCartRepository localCartRepository(Ref ref) {
  final database = ref.watch(sembastDatabaseProvider).requireValue;
  return SembastCartRepository(database);
}
```

#### Step 4: Use in Application Layer

**File:** [lib/src/features/cart/application/cart_service.dart](lib/src/features/cart/application/cart_service.dart:25-35)

```dart
class CartService {
  CartService(this.ref);
  final Ref ref;

  Future<Cart> _fetchCart() {
    final user = ref.read(authStateChangesProvider).value;

    if (user != null) {
      // Use remote repository
      return ref.read(remoteCartRepositoryProvider).fetchCart(user.uid);
    } else {
      // Use local repository (depends on interface, not implementation!)
      return ref.read(localCartRepositoryProvider).fetchCart();
    }
  }
}
```

### Example 2: Products Repository

**File:** [lib/src/features/products/data/fake_products_repository.dart](lib/src/features/products/data/fake_products_repository.dart:50-120)

```dart
class FakeProductsRepository {
  FakeProductsRepository({this.addDelay = true});

  final bool addDelay;

  /// Hardcoded product data (simulates API/database)
  static final List<Product> _allProducts = [
    Product(
      id: '1',
      imageUrl: 'assets/products/bruschetta-plate.jpg',
      title: 'Bruschetta',
      description: 'Italian appetizer',
      price: 15,
      availableQuantity: 10,
    ),
    // ... more products
  ];

  /// Get all products
  Future<List<Product>> getProductsList() async {
    if (addDelay) {
      await Future.delayed(const Duration(seconds: 2));
    }
    return _allProducts;
  }

  /// Watch products (stream)
  Stream<List<Product>> watchProductsList() async* {
    if (addDelay) {
      await Future.delayed(const Duration(seconds: 2));
    }
    yield _allProducts;
  }

  /// Get single product
  Product? getProduct(String id) {
    return _allProducts.firstWhereOrNull((p) => p.id == id);
  }

  /// Search products
  Future<List<Product>> searchProducts(String query) async {
    if (addDelay) {
      await Future.delayed(const Duration(seconds: 2));
    }

    return _allProducts
        .where((p) => p.title.toLowerCase().contains(query.toLowerCase()))
        .toList();
  }
}
```

**Riverpod Provider:**

```dart
@Riverpod(keepAlive: true)
FakeProductsRepository productsRepository(Ref ref) {
  return FakeProductsRepository();
}
```

### When to Use Repository Pattern

✅ **Use when:**
- Accessing data from external sources (API, database, file system)
- You want to abstract data source implementation
- You need to switch between different data sources
- You want easy testing with mock data

❌ **Don't use when:**
- Data is purely in-memory and temporary
- The abstraction adds unnecessary complexity
- You have simple, one-off data operations

---

## Service Pattern

### What Is It?

The Service pattern encapsulates **business logic** and **orchestrates operations** across multiple repositories or entities.

**Analogy:** A restaurant kitchen manager:
- Coordinates between different stations (repositories)
- Implements recipes (business workflows)
- Manages complex orders (use cases)
- Doesn't cook directly, but orchestrates

### Why Use It?

✅ **Orchestration**: Coordinate multiple data sources
✅ **Reusability**: Business logic used by multiple controllers
✅ **Testability**: Test business logic independently of UI
✅ **Separation**: Keep controllers thin and focused

### Example 1: Cart Service

**File:** [lib/src/features/cart/application/cart_service.dart](lib/src/features/cart/application/cart_service.dart:15-90)

```dart
class CartService {
  CartService(this.ref);

  final Ref ref;

  /// Helper: Fetch cart (handles authentication state)
  Future<Cart> _fetchCart() {
    final user = ref.read(authStateChangesProvider).value;

    if (user != null) {
      // User is authenticated: use remote cart
      return ref.read(remoteCartRepositoryProvider).fetchCart(user.uid);
    } else {
      // User is guest: use local cart
      return ref.read(localCartRepositoryProvider).fetchCart();
    }
  }

  /// Helper: Save cart (handles authentication state)
  Future<void> _setCart(Cart cart) async {
    final user = ref.read(authStateChangesProvider).value;

    if (user != null) {
      await ref.read(remoteCartRepositoryProvider).setCart(user.uid, cart);
    } else {
      await ref.read(localCartRepositoryProvider).setCart(cart);
    }
  }

  /// Use Case: Set item quantity
  Future<void> setItem(Item item) async {
    final cart = await _fetchCart();
    final updatedCart = cart.setItem(item);  // Domain logic
    await _setCart(updatedCart);
  }

  /// Use Case: Add item to cart
  Future<void> addItem(Item item) async {
    final cart = await _fetchCart();
    final updatedCart = cart.addItem(item);  // Domain logic
    await _setCart(updatedCart);
  }

  /// Use Case: Remove item
  Future<void> removeItemById(ProductID productId) async {
    final cart = await _fetchCart();
    final updatedCart = cart.removeItemById(productId);  // Domain logic
    await _setCart(updatedCart);
  }
}
```

**What This Service Does:**
1. **Abstracts auth complexity**: Controllers don't care about local vs remote
2. **Coordinates repositories**: Uses correct repository based on auth state
3. **Applies business logic**: Calls domain layer methods
4. **Ensures persistence**: Saves after every operation

**Riverpod Provider:**

```dart
@Riverpod(keepAlive: true)
CartService cartService(Ref ref) {
  return CartService(ref);
}
```

### Example 2: Cart Sync Service

**File:** [lib/src/features/cart/application/cart_sync_service.dart](lib/src/features/cart/application/cart_sync_service.dart:10-80)

```dart
/// Service that syncs local cart to remote when user signs in
class CartSyncService {
  CartSyncService(this.ref) {
    _init();
  }

  final Ref ref;
  StreamSubscription<AsyncValue<AppUser?>>? _subscription;

  void _init() {
    // Listen to authentication state changes
    _subscription = ref.listen<AsyncValue<AppUser?>>(
      authStateChangesProvider,
      (previous, next) {
        final previousUser = previous?.value;
        final user = next.value;

        // User just signed in (was null, now has value)
        if (previousUser == null && user != null) {
          _moveItemsToRemoteCart(user.uid);
        }
      },
    );
  }

  /// Complex workflow: Merge local cart into remote cart
  Future<void> _moveItemsToRemoteCart(String uid) async {
    try {
      // 1. Get repositories
      final localRepo = ref.read(localCartRepositoryProvider);
      final remoteRepo = ref.read(remoteCartRepositoryProvider);

      // 2. Fetch both carts
      final localCart = await localRepo.fetchCart();
      final remoteCart = await remoteRepo.fetchCart(uid);

      // 3. Merge carts (business logic)
      final mergedCart = _mergeCarts(localCart, remoteCart);

      // 4. Save merged cart to remote
      await remoteRepo.setCart(uid, mergedCart);

      // 5. Clear local cart
      await localRepo.setCart(const Cart());
    } catch (e) {
      debugPrint('Error syncing cart: $e');
    }
  }

  /// Business logic: Merge two carts
  Cart _mergeCarts(Cart local, Cart remote) {
    final localItems = local.items;
    final remoteItems = remote.items;

    // Start with remote items
    final merged = <String, Item>{...remoteItems};

    // Add local items
    for (final item in localItems.values) {
      final existingItem = merged[item.productId];

      if (existingItem != null) {
        // Item exists in both: sum quantities
        merged[item.productId] = Item(
          productId: item.productId,
          quantity: existingItem.quantity + item.quantity,
        );
      } else {
        // Only in local: add it
        merged[item.productId] = item;
      }
    }

    return Cart(merged);
  }

  void dispose() {
    _subscription?.cancel();
  }
}
```

**This service orchestrates a complex workflow:**
1. Listens to auth state changes
2. Detects user sign-in
3. Fetches both local and remote carts
4. Merges them with business logic
5. Saves merged cart remotely
6. Clears local cart

### Example 3: Checkout Service

**File:** [lib/src/features/checkout/application/fake_checkout_service.dart](lib/src/features/checkout/application/fake_checkout_service.dart:15-70)

```dart
class FakeCheckoutService {
  FakeCheckoutService(this.ref);

  final Ref ref;

  /// Use Case: Place order
  Future<void> placeOrder() async {
    // 1. Validate user is authenticated
    final user = ref.read(authStateChangesProvider).value;
    if (user == null) {
      throw Exception('User must be signed in to place order');
    }

    // 2. Get cart
    final cart = await ref.read(cartProvider.future);
    if (cart.items.isEmpty) {
      throw Exception('Cannot place order: cart is empty');
    }

    // 3. Calculate total price
    final products = await ref.read(productsListStreamProvider.future);
    final total = cart.totalPrice(products);

    // 4. Create order entity
    final orderId = DateTime.now().toIso8601String();
    final order = Order(
      id: orderId,
      userId: user.uid,
      items: cart.items.map((id, item) => MapEntry(id, item.quantity)),
      orderStatus: OrderStatus.confirmed,
      orderDate: DateTime.now(),
      total: total,
    );

    // 5. Save order to repository
    final ordersRepository = ref.read(ordersRepositoryProvider);
    await ordersRepository.addOrder(user.uid, order);

    // 6. Clear the cart
    final cartService = ref.read(cartServiceProvider);
    await cartService._setCart(const Cart());
  }
}
```

**Orchestrates multiple steps:**
1. Authentication check
2. Cart validation
3. Price calculation (uses domain logic)
4. Order creation
5. Persistence (uses repository)
6. Cart clearing (uses cart service)

### When to Use Service Pattern

✅ **Use when:**
- Coordinating multiple repositories
- Implementing complex business workflows
- Business logic needs to be reused
- Operations span multiple entities

❌ **Don't use when:**
- Simple CRUD operations (use repository directly)
- No business logic needed
- Single repository operation

---

## Observer Pattern

### What Is It?

The Observer pattern allows objects to **notify other objects** about changes in their state. In Dart/Flutter, this is typically implemented with **Streams**.

**Analogy:** Subscribing to a YouTube channel:
- You (observer) subscribe to a channel (observable)
- When a new video is uploaded (state change)
- You get notified automatically (stream emission)
- You can react to the notification (rebuild UI)

### Why Use It?

✅ **Reactive UI**: UI updates automatically when data changes
✅ **Loose Coupling**: Observers don't need to know about each other
✅ **Real-time**: Changes propagate immediately
✅ **Declarative**: Describe what to show, not how to update

### Example 1: InMemoryStore

**File:** [lib/src/utils/in_memory_store.dart](lib/src/utils/in_memory_store.dart:5-30)

```dart
/// Simple in-memory store with reactive streams (Observer pattern)
class InMemoryStore<T> {
  InMemoryStore(T initial) : _subject = BehaviorSubject<T>.seeded(initial);

  /// BehaviorSubject is an observable (from RxDart)
  final BehaviorSubject<T> _subject;

  /// Current value
  T get value => _subject.value;

  /// Update value (notifies all observers)
  set value(T value) => _subject.add(value);

  /// Stream for observers to listen to
  Stream<T> get stream => _subject.stream;

  void dispose() {
    _subject.close();
  }
}
```

**How it works:**

```dart
// 1. Create store
final cartStore = InMemoryStore<Cart>(const Cart());

// 2. Observers subscribe
cartStore.stream.listen((cart) {
  print('Cart changed: ${cart.items.length} items');
});

// 3. Update value (observers get notified automatically)
cartStore.value = Cart({...});  // All listeners are notified!
```

### Example 2: Sembast Repository with Streams

**File:** [lib/src/features/cart/data/local/sembast_cart_repository.dart](lib/src/features/cart/data/local/sembast_cart_repository.dart:45-60)

```dart
class SembastCartRepository implements LocalCartRepository {
  @override
  Stream<Cart> watchCart() {
    final record = store.record(cartItemsKey);

    // Create stream from database snapshots (Observer pattern)
    return record.onSnapshot(db).map((snapshot) {
      if (snapshot != null) {
        return Cart.fromJson(snapshot.value as Map<String, dynamic>);
      } else {
        return const Cart();
      }
    });
  }

  @override
  Future<void> setCart(Cart cart) async {
    // When cart is saved, onSnapshot automatically notifies listeners!
    await store.record(cartItemsKey).put(db, cart.toJson());
  }
}
```

### Example 3: Cart Provider Stream

**File:** [lib/src/features/cart/application/cart_service.dart](lib/src/features/cart/application/cart_service.dart:92-105)

```dart
/// Provider that exposes cart as a stream (Observer pattern)
@Riverpod(keepAlive: true)
Stream<Cart> cart(Ref ref) {
  final user = ref.watch(authStateChangesProvider).value;

  if (user != null) {
    // Authenticated: watch remote cart
    return ref.watch(remoteCartRepositoryProvider).watchCart(user.uid);
  } else {
    // Guest: watch local cart
    return ref.watch(localCartRepositoryProvider).watchCart();
  }
}
```

### Example 4: UI Observing Cart Changes

**File:** [lib/src/features/cart/presentation/shopping_cart/shopping_cart_screen.dart](lib/src/features/cart/presentation/shopping_cart/shopping_cart_screen.dart:20-40)

```dart
class ShoppingCartScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // This widget is an OBSERVER of the cart stream
    final cartValue = ref.watch(cartProvider);

    // When cart changes, this widget automatically rebuilds!
    return AsyncValueWidget<Cart>(
      value: cartValue,
      data: (cart) => cart.items.isEmpty
          ? const EmptyPlaceholderWidget(message: 'Your cart is empty')
          : ShoppingCartItemsList(cart: cart),
    );
  }
}
```

### Flow Diagram

```
User adds item to cart
        ↓
CartService.addItem()
        ↓
Repository.setCart()
        ↓
[Database saves cart]
        ↓
Stream emits new cart ← Observer Pattern in action!
        ↓
cartProvider notifies listeners
        ↓
UI rebuilds automatically
```

### When to Use Observer Pattern

✅ **Use when:**
- UI needs to update when data changes
- Multiple parts of the app need the same data
- Real-time updates are important
- You want reactive programming

❌ **Don't use when:**
- One-time data fetching
- Data doesn't change
- Updates aren't time-sensitive

---

## Provider Pattern

### What Is It?

The Provider pattern (specifically **Riverpod** in this project) handles **dependency injection** and **state management**.

**Analogy:** A restaurant's ordering system:
- Kitchen (provider) prepares dishes
- Waiters (widgets) request dishes from kitchen
- Kitchen manages one order vs multiple orders (singleton vs factory)
- Kitchen notifies waiters when food is ready (reactive)

### Why Use It?

✅ **Dependency Injection**: Widgets get dependencies without creating them
✅ **State Management**: Share state across the app
✅ **Testability**: Easy to override providers in tests
✅ **Compile-time Safety**: Errors caught at compile time

### Provider Types in Riverpod

#### 1. Simple Provider (Singleton)

**File:** [lib/src/features/cart/application/cart_service.dart](lib/src/features/cart/application/cart_service.dart:92-96)

```dart
/// keepAlive: true = Singleton (never disposed)
@Riverpod(keepAlive: true)
CartService cartService(Ref ref) {
  return CartService(ref);
}
```

**Usage:**

```dart
final cartService = ref.read(cartServiceProvider);
```

#### 2. Stream Provider

**File:** [lib/src/features/cart/application/cart_service.dart](lib/src/features/cart/application/cart_service.dart:98-110)

```dart
/// Provides a stream of cart updates
@Riverpod(keepAlive: true)
Stream<Cart> cart(Ref ref) {
  final user = ref.watch(authStateChangesProvider).value;

  if (user != null) {
    return ref.watch(remoteCartRepositoryProvider).watchCart(user.uid);
  } else {
    return ref.watch(localCartRepositoryProvider).watchCart();
  }
}
```

**Usage:**

```dart
// Watch (rebuild on changes)
final cartValue = ref.watch(cartProvider);

// Read once
final cart = await ref.read(cartProvider.future);
```

#### 3. Family Provider (Parameterized)

**File:** [lib/src/features/products/data/fake_products_repository.dart](lib/src/features/products/data/fake_products_repository.dart:145-152)

```dart
/// Provider that takes a parameter
@riverpod
Product? product(Ref ref, ProductID id) {
  final repository = ref.watch(productsRepositoryProvider);
  return repository.getProduct(id);
}
```

**Usage:**

```dart
// Different parameter = different provider instance
final product1 = ref.watch(productProvider('1'));
final product2 = ref.watch(productProvider('2'));
```

#### 4. Notifier Provider (Stateful)

**File:** [lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart](lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart:15-50)

```dart
/// Controller with mutable state
@riverpod
class AddToCartController extends _$AddToCartController {
  @override
  FutureOr<void> build() {
    // Initial state
  }

  Future<void> addItem(ProductID productId) async {
    // Update state
    state = const AsyncLoading();

    // Perform action
    final service = ref.read(cartServiceProvider);
    state = await AsyncValue.guard(() => service.addItem(item));
  }
}
```

**Usage:**

```dart
// Read controller
final controller = ref.read(addToCartControllerProvider.notifier);

// Watch state
final state = ref.watch(addToCartControllerProvider);
```

### Dependency Graph Example

```dart
// Products provider
@Riverpod(keepAlive: true)
FakeProductsRepository productsRepository(Ref ref) {
  return FakeProductsRepository();
}

// Cart provider (depends on products)
@Riverpod(keepAlive: true)
Stream<Cart> cart(Ref ref) {
  // Can access other providers via ref
  final user = ref.watch(authStateChangesProvider).value;
  return /* ... */;
}

// Controller (depends on both)
@riverpod
class CheckoutController extends _$CheckoutController {
  @override
  FutureOr<void> build() {}

  Future<void> checkout() async {
    // Access dependencies
    final cart = await ref.read(cartProvider.future);
    final products = await ref.read(productsListStreamProvider.future);

    // Business logic
    final total = cart.totalPrice(products);
    // ...
  }
}
```

### Provider Lifecycle

```dart
@Riverpod(keepAlive: true)  // Never disposed
CartService cartService(Ref ref) {
  final service = CartService(ref);

  // Cleanup when provider is disposed
  ref.onDispose(() {
    service.dispose();
  });

  return service;
}
```

### When to Use Provider Pattern

✅ **Use when:**
- Sharing state across widgets
- Dependency injection
- Reactive updates
- Complex state management

❌ **Don't use when:**
- Local widget state (use StatefulWidget instead)
- Temporary variables
- Simple computations

---

## Extension Methods Pattern

### What Is It?

Extension methods allow you to **add functionality to existing classes** without modifying them or creating subclasses.

**Analogy:** Adding accessories to your phone:
- You don't modify the phone itself
- You attach a case, screen protector, etc.
- The accessories add functionality
- You can remove them anytime

### Why Use It?

✅ **Separation**: Keep entity classes simple
✅ **Organization**: Group related functionality
✅ **Maintainability**: Business logic near the entity
✅ **Immutability**: Add methods without modifying originals

### Example: Cart Extensions

**File:** [lib/src/features/cart/domain/mutable_cart.dart](lib/src/features/cart/domain/mutable_cart.dart:5-115)

```dart
/// Extension adds cart mutation methods
extension MutableCart on Cart {

  /// Business logic: Add item to cart
  Cart addItem(Item item) {
    // Copy current items
    final copy = Map<String, Item>.from(items);
    final currentItem = copy[item.productId];

    if (currentItem != null) {
      // Item exists: increase quantity (business rule)
      copy[item.productId] = Item(
        productId: item.productId,
        quantity: currentItem.quantity + item.quantity,
      );
    } else {
      // New item: add to cart
      copy[item.productId] = item;
    }

    // Return new immutable cart
    return Cart(copy);
  }

  /// Business logic: Set item quantity
  Cart setItem(Item item) {
    final copy = Map<String, Item>.from(items);
    copy[item.productId] = item;
    return Cart(copy);
  }

  /// Business logic: Remove item
  Cart removeItemById(ProductID productId) {
    final copy = Map<String, Item>.from(items);
    copy.remove(productId);
    return Cart(copy);
  }
}

/// Another extension for cart calculations
extension CartItems on Cart {

  /// Get items as a list
  List<Item> toItemsList() {
    return items.values.toList();
  }

  /// Calculate total quantity
  int get totalQuantity {
    return items.values.fold<int>(
      0,
      (sum, item) => sum + item.quantity,
    );
  }

  /// Calculate total price (business logic)
  double totalPrice(List<Product> productsList) {
    double total = 0.0;

    for (final item in items.values) {
      final product = productsList.firstWhere(
        (p) => p.id == item.productId,
      );
      total += product.price * item.quantity;
    }

    return total;
  }
}
```

**Usage:**

```dart
// Original Cart entity is simple
final cart = Cart({
  '1': Item(productId: '1', quantity: 2),
});

// Extension methods add functionality
final updatedCart = cart.addItem(Item(productId: '2', quantity: 1));
final quantity = cart.totalQuantity;
final price = cart.totalPrice(products);
```

### Benefits

**Without Extensions:**
```dart
// Bad: All methods in Cart class (bloated)
class Cart {
  final Map<String, Item> items;

  Cart addItem(Item item) { /* ... */ }
  Cart setItem(Item item) { /* ... */ }
  Cart removeItem(ProductID id) { /* ... */ }
  int get totalQuantity { /* ... */ }
  double totalPrice(List<Product> products) { /* ... */ }
  // ... 20 more methods
}
```

**With Extensions:**
```dart
// Good: Simple entity
class Cart {
  final Map<String, Item> items;
}

// Related functionality grouped together
extension MutableCart on Cart {
  Cart addItem(Item item) { /* ... */ }
  Cart setItem(Item item) { /* ... */ }
}

extension CartCalculations on Cart {
  int get totalQuantity { /* ... */ }
  double totalPrice(List<Product> products) { /* ... */ }
}
```

### When to Use Extensions

✅ **Use when:**
- Adding methods to existing classes (especially from packages)
- Keeping entity classes simple
- Grouping related functionality
- Maintaining immutability

❌ **Don't use when:**
- You control the class and it makes sense to add methods directly
- The extension is used only once
- It requires access to private fields

---

## Value Object Pattern

### What Is It?

Value objects are **simple, immutable types** that represent values (as opposed to entities with identity). They're often type aliases for primitive types to add type safety.

**Analogy:** Using specific measuring cups:
- Instead of "a cup of flour" (string)
- Use a "flour cup" that can only hold flour (type-safe)
- Can't accidentally use a "sugar cup" for flour

### Why Use It?

✅ **Type Safety**: Can't mix up different ID types
✅ **Self-Documenting**: Code is clearer
✅ **Refactoring**: Easy to change underlying type
✅ **Validation**: Can add validation logic

### Example: Type Aliases

**File:** [lib/src/features/products/domain/product.dart](lib/src/features/products/domain/product.dart:7-10)

```dart
/// Type alias for product IDs
typedef ProductID = String;

class Product {
  const Product({
    required this.id,  // ProductID, not just String
    // ...
  });

  final ProductID id;
  // ...
}
```

**File:** [lib/src/features/orders/domain/order.dart](lib/src/features/orders/domain/order.dart:7-10)

```dart
/// Type alias for order IDs
typedef OrderID = String;

class Order {
  const Order({
    required this.id,  // OrderID, not just String
    // ...
  });

  final OrderID id;
  // ...
}
```

### Benefits

**Without Value Objects:**
```dart
// Bad: Easy to mix up
void addToCart(String productId) { }
void viewOrder(String orderId) { }

// Error: passing order ID as product ID
addToCart(myOrderId);  // Compiles, but wrong!
```

**With Value Objects:**
```dart
// Good: Type-safe
void addToCart(ProductID productId) { }
void viewOrder(OrderID orderId) { }

// Error: won't compile!
addToCart(myOrderId);  // Type error!
```

### Example: Item Value Object

**File:** [lib/src/features/cart/domain/item.dart](lib/src/features/cart/domain/item.dart:5-40)

```dart
/// Item is a value object (immutable, value-based equality)
@immutable
class Item {
  const Item({
    required this.productId,
    required this.quantity,
  });

  final ProductID productId;
  final int quantity;

  // Value objects have value-based equality
  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;

    return other is Item &&
        other.productId == productId &&
        other.quantity == quantity;
  }

  @override
  int get hashCode => productId.hashCode ^ quantity.hashCode;
}
```

**Value-based Equality:**
```dart
final item1 = Item(productId: '1', quantity: 2);
final item2 = Item(productId: '1', quantity: 2);

print(item1 == item2);  // true (same values)
```

### When to Use Value Objects

✅ **Use when:**
- Type safety is important
- Wrapping primitive types (IDs, emails, phone numbers)
- Value-based equality needed
- Immutable data

❌ **Don't use when:**
- Identity matters more than value
- Mutable state needed
- Adds unnecessary complexity

---

## Controller Pattern

### What Is It?

Controllers manage **UI state** and **handle user interactions** using Riverpod's `AsyncNotifier` classes.

**Analogy:** TV remote control:
- Remote (controller) manages TV state
- Buttons (user actions) trigger state changes
- TV (UI) displays current state
- Remote handles communication with TV

### Why Use It?

✅ **Separation**: Business logic separate from UI
✅ **Testability**: Test controllers without widgets
✅ **Reusability**: Multiple widgets can use same controller
✅ **State Management**: Built-in loading/error states

### Example: Add to Cart Controller

**File:** [lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart](lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart:15-80)

```dart
@riverpod
class AddToCartController extends _$AddToCartController {

  /// Initial state
  @override
  FutureOr<void> build() {
    // Nothing to initialize
  }

  /// Action: Add item to cart
  Future<void> addItem(ProductID productId) async {
    // 1. Get dependencies
    final cartService = ref.read(cartServiceProvider);
    final product = ref.read(productProvider(productId));

    // 2. Validate product exists
    if (product == null) {
      state = AsyncError(
        Exception('Product not found'),
        StackTrace.current,
      );
      return;
    }

    // 3. Check available quantity
    final cart = await ref.read(cartProvider.future);
    final currentQuantity = cart.items[productId]?.quantity ?? 0;

    if (currentQuantity >= product.availableQuantity) {
      state = AsyncError(
        Exception('Not enough stock available'),
        StackTrace.current,
      );
      return;
    }

    // 4. Create item
    final item = Item(productId: productId, quantity: 1);

    // 5. Set loading state
    state = const AsyncLoading<void>();

    // 6. Call service and handle result
    state = await AsyncValue.guard(() => cartService.addItem(item));

    // State is now either AsyncData or AsyncError
  }
}
```

**Generated Provider (by riverpod_generator):**
```dart
// Auto-generated file: add_to_cart_controller.g.dart

final addToCartControllerProvider =
    AutoDisposeAsyncNotifierProvider<AddToCartController, void>(
  AddToCartController.new,
);
```

### Using the Controller in UI

**File:** [lib/src/features/cart/presentation/add_to_cart/add_to_cart_widget.dart](lib/src/features/cart/presentation/add_to_cart/add_to_cart_widget.dart:20-60)

```dart
class AddToCartWidget extends ConsumerWidget {
  const AddToCartWidget({required this.productId});

  final ProductID productId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Watch controller state
    final state = ref.watch(addToCartControllerProvider);

    return state.when(
      data: (_) => ElevatedButton(
        onPressed: () async {
          // Call controller action
          final controller = ref.read(addToCartControllerProvider.notifier);
          await controller.addItem(productId);

          // Show success message
          if (context.mounted) {
            ScaffoldMessenger.of(context).showSnackBar(
              const SnackBar(content: Text('Added to cart!')),
            );
          }
        },
        child: const Text('Add to Cart'),
      ),
      loading: () => const CircularProgressIndicator(),
      error: (error, _) => Text('Error: $error'),
    );
  }
}
```

### Controller State Lifecycle

```
Initial: AsyncData<void>(null)
         ↓
User taps button
         ↓
AsyncLoading<void>()  ← UI shows loading spinner
         ↓
Service call completes
         ↓
Success: AsyncData<void>(null)  ← UI shows button again
OR
Error: AsyncError(exception)  ← UI shows error message
```

### When to Use Controller Pattern

✅ **Use when:**
- Managing UI state (loading, success, error)
- Handling user interactions
- Need to separate UI from business logic
- State needs to be shared

❌ **Don't use when:**
- Simple local state (use StatefulWidget)
- No async operations
- State is purely derived

---

## Robot Pattern

### What Is It?

The Robot pattern creates **reusable test helpers** that encapsulate common test actions and assertions.

**Analogy:** Having a personal assistant for testing:
- Instead of you doing every step manually
- Assistant knows how to do common tasks
- You just say "sign in" and they do all the steps
- Makes tests readable and maintainable

### Why Use It?

✅ **Reusability**: Common test actions in one place
✅ **Readability**: Tests read like plain English
✅ **Maintainability**: Update one place when UI changes
✅ **Consistency**: All tests use same patterns

### Example: Main Robot

**File:** [test/src/robot.dart](test/src/robot.dart:15-50)

```dart
/// Main test robot that coordinates feature-specific robots
class Robot {
  Robot(this.tester)
      : auth = AuthRobot(tester),
        products = ProductsRobot(tester),
        cart = CartRobot(tester),
        checkout = CheckoutRobot(tester);

  final WidgetTester tester;

  // Feature-specific robots
  final AuthRobot auth;
  final ProductsRobot products;
  final CartRobot cart;
  final CheckoutRobot checkout;

  /// Pump app with provider overrides
  Future<void> pumpMyApp({
    List<Override> overrides = const [],
  }) async {
    await tester.pumpWidget(
      ProviderScope(
        overrides: overrides,
        child: const MyApp(),
      ),
    );
    await tester.pumpAndSettle();
  }
}
```

### Example: Auth Robot

**File:** [test/src/features/authentication/auth_robot.dart](test/src/features/authentication/auth_robot.dart:10-80)

```dart
class AuthRobot {
  AuthRobot(this.tester);

  final WidgetTester tester;

  /// Action: Open sign-in screen
  Future<void> openSignInScreen() async {
    final finder = find.text('Account');
    expect(finder, findsOneWidget);
    await tester.tap(finder);
    await tester.pumpAndSettle();
  }

  /// Action: Sign in with email and password
  Future<void> signInWithEmailAndPassword({
    required String email,
    required String password,
  }) async {
    // Find email field
    final emailField = find.byType(TextField).first;
    await tester.enterText(emailField, email);
    await tester.pumpAndSettle();

    // Find password field
    final passwordField = find.byType(TextField).last;
    await tester.enterText(passwordField, password);
    await tester.pumpAndSettle();

    // Tap sign-in button
    final signInButton = find.text('Sign In');
    await tester.tap(signInButton);
    await tester.pumpAndSettle();
  }

  /// Assertion: Expect signed in
  void expectSignedIn() {
    final finder = find.text('Sign Out');
    expect(finder, findsOneWidget);
  }

  /// Assertion: Expect signed out
  void expectSignedOut() {
    final finder = find.text('Sign In');
    expect(finder, findsOneWidget);
  }
}
```

### Example: Test Using Robots

**File:** [test/src/features/authentication/auth_flow_test.dart](test/src/features/authentication/auth_flow_test.dart:15-40)

```dart
void main() {
  testWidgets('Sign in flow', (tester) async {
    // Create robot
    final r = Robot(tester);

    // Override repositories with fake data
    await r.pumpMyApp(overrides: [
      authRepositoryProvider.overrideWithValue(FakeAuthRepository()),
    ]);

    // Test steps using robot (readable!)
    await r.auth.openSignInScreen();
    await r.auth.signInWithEmailAndPassword(
      email: 'test@test.com',
      password: 'test1234',
    );
    r.auth.expectSignedIn();
  });

  testWidgets('Full purchase flow', (tester) async {
    final r = Robot(tester);

    await r.pumpMyApp();

    // Readable test steps
    await r.auth.openSignInScreen();
    await r.auth.signInWithEmailAndPassword(
      email: 'test@test.com',
      password: 'test1234',
    );

    r.products.expectFindAllProductCards();
    await r.products.selectProduct();
    await r.cart.addToCart();
    r.cart.expectItemCount(1);
    await r.cart.openCart();
    await r.checkout.startCheckout();
    await r.checkout.completePayment();

    r.checkout.expectOrderConfirmation();
  });
}
```

### Benefits

**Without Robot:**
```dart
// Hard to read and maintain
testWidgets('Test', (tester) async {
  await tester.pumpWidget(MyApp());
  final emailField = find.byType(TextField).first;
  await tester.enterText(emailField, 'test@test.com');
  final passwordField = find.byType(TextField).last;
  await tester.enterText(passwordField, 'password');
  final button = find.text('Sign In');
  await tester.tap(button);
  await tester.pumpAndSettle();
  expect(find.text('Welcome'), findsOneWidget);
});
```

**With Robot:**
```dart
// Clear and maintainable
testWidgets('Test', (tester) async {
  final r = Robot(tester);
  await r.pumpMyApp();
  await r.auth.signInWithEmailAndPassword(
    email: 'test@test.com',
    password: 'password',
  );
  r.auth.expectSignedIn();
});
```

---

## Factory Pattern

### What Is It?

The Factory pattern creates objects without specifying the exact class.

### Example: fromJson Factories

**File:** [lib/src/features/cart/domain/cart.dart](lib/src/features/cart/domain/cart.dart:15-25)

```dart
class Cart {
  const Cart([this.items = const {}]);

  final Map<String, Item> items;

  /// Factory constructor
  factory Cart.fromJson(Map<String, dynamic> json) {
    final itemsJson = json['items'] as Map<String, dynamic>;

    final items = itemsJson.map(
      (key, value) => MapEntry(
        key,
        Item.fromJson(value as Map<String, dynamic>),
      ),
    );

    return Cart(items);
  }
}
```

---

## Singleton Pattern

### What Is It?

Ensures a class has only **one instance** globally.

### Example: Riverpod Providers with keepAlive

**File:** [lib/src/features/cart/application/cart_service.dart](lib/src/features/cart/application/cart_service.dart:92-96)

```dart
/// Singleton: keepAlive = true
@Riverpod(keepAlive: true)
CartService cartService(Ref ref) {
  return CartService(ref);  // Only created once
}
```

---

## Pattern Summary

| Pattern | Purpose | Example File |
|---------|---------|--------------|
| **Repository** | Abstract data access | `local_cart_repository.dart` |
| **Service** | Business logic orchestration | `cart_service.dart` |
| **Observer** | Reactive updates | `in_memory_store.dart` |
| **Provider** | Dependency injection | All `*_provider.dart` files |
| **Extension Methods** | Add functionality | `mutable_cart.dart` |
| **Value Object** | Type-safe values | `product.dart` (ProductID) |
| **Controller** | UI state management | `add_to_cart_controller.dart` |
| **Robot** | Test automation | `robot.dart` |
| **Factory** | Object creation | `fromJson` methods |
| **Singleton** | Single instance | Riverpod providers with `keepAlive` |

---

## Next Steps

- **[State Management](04-state-management-and-navigation.md)**: Deep dive into Riverpod
- **[Features](05-features-deep-dive.md)**: See patterns in action
- **[Getting Started](06-getting-started-guide.md)**: Apply these patterns yourself

---
