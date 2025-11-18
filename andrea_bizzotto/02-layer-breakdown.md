# Layer Breakdown

## Table of Contents
- [Introduction](#introduction)
- [Domain Layer](#domain-layer)
- [Data Layer](#data-layer)
- [Application Layer](#application-layer)
- [Presentation Layer](#presentation-layer)
- [Cross-Layer Communication](#cross-layer-communication)
- [Layer Dependencies](#layer-dependencies)

---

## Introduction

This document provides an in-depth exploration of each architectural layer in the e-commerce application. We'll examine the purpose, responsibilities, and implementation details of each layer with real code examples from the project.

### The Four Layers Recap

```
Presentation ──→ Application ──→ Domain
      ↑                            ↑
      └────────── Data ────────────┘
```

**Dependency Rule:**
- Outer layers depend on inner layers
- Inner layers know nothing about outer layers
- Domain is the innermost layer (no dependencies)
- Data and Presentation both depend on Domain
- Application coordinates between layers

---

## Domain Layer

### Overview

The Domain Layer is the **heart of your application**. It contains the core business logic and rules, completely independent of any framework, UI, or database.

**Think of it as:** The recipe book in a restaurant. The recipes (business rules) don't care whether you're using a gas or electric stove (implementation details).

### Characteristics

- ✅ Pure Dart code (no Flutter dependencies)
- ✅ Framework-agnostic
- ✅ Immutable data structures
- ✅ Business rules and validation
- ✅ No external dependencies
- ✅ Highly testable

### What Goes in Domain Layer?

1. **Entities**: Core business objects
2. **Value Objects**: Type-safe identifiers and values
3. **Business Logic**: Rules and operations
4. **Domain Exceptions**: Business rule violations

### Entities

Entities represent the core business concepts in your application.

#### Example 1: Product Entity

**File:** [lib/src/features/products/domain/product.dart](lib/src/features/products/domain/product.dart:7-30)

```dart
/// Type alias for product IDs (type safety)
typedef ProductID = String;

/// Product entity - represents a product in the store
@immutable
class Product {
  const Product({
    required this.id,
    required this.imageUrl,
    required this.title,
    required this.description,
    required this.price,
    required this.availableQuantity,
    this.avgRating = 0.0,
    this.numRatings = 0,
  });

  final ProductID id;
  final String imageUrl;
  final String title;
  final String description;
  final double price;
  final int availableQuantity;
  final double avgRating;
  final int numRatings;

  // Serialization methods
  factory Product.fromJson(Map<String, dynamic> json) { /* ... */ }
  Map<String, dynamic> toJson() { /* ... */ }
}
```

**Key Points:**
- `@immutable`: Can't be changed after creation
- `typedef ProductID = String`: Type-safe ID (can't mix Product and Order IDs)
- All fields are `final`: Immutability enforced
- Serialization methods for data persistence

#### Example 2: Cart Entity

**File:** [lib/src/features/cart/domain/cart.dart](lib/src/features/cart/domain/cart.dart:7-45)

```dart
/// Shopping cart entity
@immutable
class Cart {
  const Cart([this.items = const {}]);

  /// Map of product ID to cart item
  final Map<String, Item> items;

  /// Factory constructor from JSON
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

  /// Convert to JSON
  Map<String, dynamic> toJson() {
    return {
      'items': items.map((key, value) => MapEntry(key, value.toJson())),
    };
  }

  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is Cart && mapEquals(other.items, items);
  }

  @override
  int get hashCode => items.hashCode;

  @override
  String toString() => 'Cart{items: $items}';
}
```

**Key Points:**
- Stores items as a `Map<ProductID, Item>`
- Completely immutable
- Equality based on content, not reference
- JSON serialization for persistence

#### Example 3: Item Value Object

**File:** [lib/src/features/cart/domain/item.dart](lib/src/features/cart/domain/item.dart:5-35)

```dart
/// Represents a single item in the shopping cart
@immutable
class Item {
  const Item({
    required this.productId,
    required this.quantity,
  });

  final ProductID productId;
  final int quantity;

  factory Item.fromJson(Map<String, dynamic> json) {
    return Item(
      productId: json['productId'] as String,
      quantity: json['quantity'] as int,
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'productId': productId,
      'quantity': quantity,
    };
  }

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

### Business Logic with Extension Methods

Since entities are immutable, we use **extension methods** to add behavior without modifying the entity class.

**File:** [lib/src/features/cart/domain/mutable_cart.dart](lib/src/features/cart/domain/mutable_cart.dart:5-90)

```dart
/// Extension methods that implement cart business logic
extension MutableCart on Cart {

  /// Add an item to cart (increases quantity if exists)
  Cart addItem(Item item) {
    final copy = Map<String, Item>.from(items);
    final currentItem = copy[item.productId];

    if (currentItem != null) {
      // Item already exists - increase quantity
      final newQuantity = currentItem.quantity + item.quantity;
      copy[item.productId] = Item(
        productId: item.productId,
        quantity: newQuantity,
      );
    } else {
      // New item - add to cart
      copy[item.productId] = item;
    }

    return Cart(copy);  // Return new immutable cart
  }

  /// Set item quantity (or add if doesn't exist)
  Cart setItem(Item item) {
    final copy = Map<String, Item>.from(items);
    copy[item.productId] = item;
    return Cart(copy);
  }

  /// Remove item by product ID
  Cart removeItemById(ProductID productId) {
    final copy = Map<String, Item>.from(items);
    copy.remove(productId);
    return Cart(copy);
  }
}
```

**Why Extension Methods?**
- Keeps entity classes simple
- Business logic lives near the entity
- Easy to test independently
- Maintains immutability

**Business Rules Implemented:**
1. Adding existing item → increase quantity
2. Adding new item → add to map
3. Setting item → replace or add
4. Removing item → remove from map

### Domain Calculations

**File:** [lib/src/features/cart/domain/mutable_cart.dart](lib/src/features/cart/domain/mutable_cart.dart:92-110)

```dart
extension CartItems on Cart {
  /// Get list of all items
  List<Item> toItemsList() {
    return items.values.toList();
  }

  /// Calculate total quantity of all items
  int get totalQuantity {
    return items.values.fold<int>(
      0,
      (sum, item) => sum + item.quantity,
    );
  }

  /// Calculate total price (requires products list)
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

### Other Domain Entities

#### Order Entity

**File:** [lib/src/features/orders/domain/order.dart](lib/src/features/orders/domain/order.dart:7-35)

```dart
typedef OrderID = String;

/// Represents a completed order
@immutable
class Order {
  const Order({
    required this.id,
    required this.userId,
    required this.items,
    required this.orderStatus,
    required this.orderDate,
    required this.total,
  });

  final OrderID id;
  final String userId;
  final Map<ProductID, int> items;  // productId -> quantity
  final OrderStatus orderStatus;
  final DateTime orderDate;
  final double total;

  // Serialization, equality, etc.
}

enum OrderStatus {
  confirmed,
  shipped,
  delivered,
}
```

#### Review Entity

**File:** [lib/src/features/reviews/domain/review.dart](lib/src/features/reviews/domain/review.dart:7-30)

```dart
typedef ReviewID = String;

/// Represents a product review
@immutable
class Review {
  const Review({
    required this.rating,
    required this.comment,
    required this.date,
    required this.userId,
  });

  final double rating;      // 1.0 to 5.0
  final String comment;
  final DateTime date;
  final String userId;

  // Serialization methods
}
```

---

## Data Layer

### Overview

The Data Layer handles **all external data operations**: fetching from APIs, saving to databases, reading from local storage, etc.

**Think of it as:** The warehouse and suppliers. They store and retrieve goods (data) but don't care how the kitchen (business logic) uses them.

### Characteristics

- ✅ Repository pattern
- ✅ Abstract interfaces
- ✅ Concrete implementations
- ✅ Data source abstraction
- ✅ Error handling
- ✅ Data caching

### Repository Pattern

The repository pattern provides a clean abstraction over data sources:

```
┌─────────────────────────────────────────┐
│     Application/Presentation           │
│     (Depends on interface only)        │
└──────────────┬──────────────────────────┘
               │
               ↓
┌──────────────────────────────────────────┐
│     Repository Interface                │
│     (Abstract class - contract)         │
└──────────────┬───────────────────────────┘
               │
               ↓
┌──────────────────────────────────────────┐
│     Repository Implementation           │
│     (Sembast, Firebase, REST API, etc.) │
└──────────────────────────────────────────┘
```

### Example: Local Cart Repository

#### Interface (Contract)

**File:** [lib/src/features/cart/data/local/local_cart_repository.dart](lib/src/features/cart/data/local/local_cart_repository.dart:6-15)

```dart
/// Abstract interface for local cart storage
abstract class LocalCartRepository {
  /// Fetch cart once
  Future<Cart> fetchCart();

  /// Watch cart changes (reactive)
  Stream<Cart> watchCart();

  /// Save cart
  Future<void> setCart(Cart cart);
}
```

**Why Abstract?**
- Defines the contract
- Allows multiple implementations
- Easy to mock for testing
- Dependency inversion principle

#### Implementation (Concrete)

**File:** [lib/src/features/cart/data/local/sembast_cart_repository.dart](lib/src/features/cart/data/local/sembast_cart_repository.dart:15-75)

```dart
/// Sembast implementation of local cart repository
class SembastCartRepository implements LocalCartRepository {
  SembastCartRepository(this.db);

  final Database db;  // Sembast database instance
  final store = StoreRef.main();  // Main store

  static const cartItemsKey = 'cartItems';

  @override
  Future<Cart> fetchCart() async {
    // Read from database
    final cartItemsJson = await store.record(cartItemsKey).get(db);

    if (cartItemsJson != null) {
      // Parse JSON to Cart entity
      return Cart.fromJson(cartItemsJson as Map<String, dynamic>);
    } else {
      // Return empty cart if nothing saved
      return const Cart();
    }
  }

  @override
  Stream<Cart> watchCart() {
    // Get database record
    final record = store.record(cartItemsKey);

    // Create stream that emits on changes
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
    // Convert cart to JSON and save
    await store.record(cartItemsKey).put(db, cart.toJson());
  }
}
```

**Key Points:**
- Implements the interface
- Handles Sembast-specific code
- Converts between domain entities and JSON
- Provides reactive stream of cart changes

#### Riverpod Provider

**File:** [lib/src/features/cart/data/local/local_cart_repository.dart](lib/src/features/cart/data/local/local_cart_repository.dart:17-25)

```dart
@Riverpod(keepAlive: true)
LocalCartRepository localCartRepository(Ref ref) {
  // Get Sembast database instance
  final database = ref.watch(sembastDatabaseProvider).requireValue;

  // Create and return repository
  return SembastCartRepository(database);
}

@Riverpod(keepAlive: true)
Future<Database> sembastDatabase(Ref ref) async {
  // Initialize Sembast database
  final appDocDir = await getApplicationDocumentsDirectory();
  final dbPath = join(appDocDir.path, 'default.db');
  return databaseFactoryIo.openDatabase(dbPath);
}
```

### Example: Remote Cart Repository

For authenticated users, carts are stored remotely (simulated with fake data):

#### Interface

**File:** [lib/src/features/cart/data/remote/remote_cart_repository.dart](lib/src/features/cart/data/remote/remote_cart_repository.dart:6-15)

```dart
abstract class RemoteCartRepository {
  Future<Cart> fetchCart(String uid);
  Stream<Cart> watchCart(String uid);
  Future<void> setCart(String uid, Cart cart);
}
```

#### Fake Implementation

**File:** [lib/src/features/cart/data/remote/fake_remote_cart_repository.dart](lib/src/features/cart/data/remote/fake_remote_cart_repository.dart:12-45)

```dart
class FakeRemoteCartRepository implements RemoteCartRepository {
  FakeRemoteCartRepository({this.addDelay = true});

  final bool addDelay;

  /// In-memory store per user
  final Map<String, InMemoryStore<Cart>> _carts = {};

  /// Get or create cart store for user
  InMemoryStore<Cart> _getCartRef(String uid) {
    return _carts.putIfAbsent(uid, () => InMemoryStore<Cart>(const Cart()));
  }

  @override
  Future<Cart> fetchCart(String uid) async {
    if (addDelay) {
      await Future.delayed(const Duration(seconds: 1));
    }
    return _getCartRef(uid).value;
  }

  @override
  Stream<Cart> watchCart(String uid) {
    return _getCartRef(uid).stream;
  }

  @override
  Future<void> setCart(String uid, Cart cart) async {
    if (addDelay) {
      await Future.delayed(const Duration(seconds: 1));
    }
    _getCartRef(uid).value = cart;
  }
}
```

**InMemoryStore Utility**

**File:** [lib/src/utils/in_memory_store.dart](lib/src/utils/in_memory_store.dart:5-25)

```dart
/// Simple in-memory store with reactive streams
class InMemoryStore<T> {
  InMemoryStore(T initial) : _subject = BehaviorSubject<T>.seeded(initial);

  final BehaviorSubject<T> _subject;

  /// Current value
  T get value => _subject.value;

  /// Update value (emits to stream)
  set value(T value) => _subject.add(value);

  /// Stream of value changes
  Stream<T> get stream => _subject.stream;

  void dispose() {
    _subject.close();
  }
}
```

### Other Repository Examples

#### Products Repository

**File:** [lib/src/features/products/data/fake_products_repository.dart](lib/src/features/products/data/fake_products_repository.dart:50-90)

```dart
class FakeProductsRepository {
  FakeProductsRepository({this.addDelay = true});

  final bool addDelay;

  /// Hardcoded product list
  static final List<Product> _allProducts = [
    Product(
      id: '1',
      imageUrl: 'assets/products/bruschetta-plate.jpg',
      title: 'Bruschetta',
      description: 'Italian appetizer with tomatoes',
      price: 15,
      availableQuantity: 10,
    ),
    // ... more products
  ];

  /// Get product list
  Future<List<Product>> getProductsList() async {
    if (addDelay) {
      await Future.delayed(const Duration(seconds: 2));
    }
    return _allProducts;
  }

  /// Get product stream (reactive)
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

---

## Application Layer

### Overview

The Application Layer **orchestrates business workflows** by coordinating between the presentation and data layers. It implements use cases and complex business operations.

**Think of it as:** The kitchen manager who coordinates between waiters (presentation), cooks (domain logic), and suppliers (data layer).

### Characteristics

- ✅ Implements use cases
- ✅ Coordinates multiple repositories
- ✅ Handles complex workflows
- ✅ Business logic orchestration
- ✅ Transaction management
- ✅ Error handling

### Example: Cart Service

**File:** [lib/src/features/cart/application/cart_service.dart](lib/src/features/cart/application/cart_service.dart:15-90)

```dart
class CartService {
  CartService(this.ref);

  final Ref ref;

  /// Fetch cart (handles auth state)
  Future<Cart> _fetchCart() {
    final user = ref.read(authStateChangesProvider).value;
    if (user != null) {
      // Authenticated: use remote cart
      return ref.read(remoteCartRepositoryProvider).fetchCart(user.uid);
    } else {
      // Guest: use local cart
      return ref.read(localCartRepositoryProvider).fetchCart();
    }
  }

  /// Save cart (handles auth state)
  Future<void> _setCart(Cart cart) async {
    final user = ref.read(authStateChangesProvider).value;
    if (user != null) {
      // Authenticated: save to remote
      await ref.read(remoteCartRepositoryProvider).setCart(user.uid, cart);
    } else {
      // Guest: save locally
      await ref.read(localCartRepositoryProvider).setCart(cart);
    }
  }

  /// Use case: Set item in cart
  Future<void> setItem(Item item) async {
    final cart = await _fetchCart();
    final updatedCart = cart.setItem(item);  // Domain logic
    await _setCart(updatedCart);
  }

  /// Use case: Add item to cart
  Future<void> addItem(Item item) async {
    final cart = await _fetchCart();
    final updatedCart = cart.addItem(item);  // Domain logic
    await _setCart(updatedCart);
  }

  /// Use case: Remove item from cart
  Future<void> removeItemById(ProductID productId) async {
    final cart = await _fetchCart();
    final updatedCart = cart.removeItemById(productId);  // Domain logic
    await _setCart(updatedCart);
  }
}
```

**Riverpod Provider**

```dart
@Riverpod(keepAlive: true)
CartService cartService(Ref ref) {
  return CartService(ref);
}

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

**What This Service Does:**
1. **Abstracts auth state** - Controllers don't need to know if user is logged in
2. **Coordinates repositories** - Uses local or remote based on auth
3. **Applies domain logic** - Calls Cart extension methods
4. **Handles persistence** - Saves after each operation

### Example: Cart Sync Service

When a user signs in, their local cart needs to merge with their remote cart:

**File:** [lib/src/features/cart/application/cart_sync_service.dart](lib/src/features/cart/application/cart_sync_service.dart:10-60)

```dart
class CartSyncService {
  CartSyncService(this.ref) {
    _init();
  }

  final Ref ref;
  StreamSubscription<AsyncValue<AppUser?>>? _subscription;

  void _init() {
    // Listen to auth state changes
    _subscription = ref.listen<AsyncValue<AppUser?>>(
      authStateChangesProvider,
      (previous, next) {
        final previousUser = previous?.value;
        final user = next.value;

        // User just signed in
        if (previousUser == null && user != null) {
          _moveItemsToRemoteCart(user.uid);
        }
      },
    );
  }

  /// Merge local cart into remote cart
  Future<void> _moveItemsToRemoteCart(String uid) async {
    try {
      // Get both carts
      final localCartRepository = ref.read(localCartRepositoryProvider);
      final remoteCartRepository = ref.read(remoteCartRepositoryProvider);

      final localCart = await localCartRepository.fetchCart();
      final remoteCart = await remoteCartRepository.fetchCart(uid);

      // Merge carts (local items take precedence)
      final mergedCart = _mergeCarts(localCart, remoteCart);

      // Save merged cart to remote
      await remoteCartRepository.setCart(uid, mergedCart);

      // Clear local cart
      await localCartRepository.setCart(const Cart());
    } catch (e) {
      // Handle error
      debugPrint('Error moving cart items: $e');
    }
  }

  /// Merge two carts (business logic)
  Cart _mergeCarts(Cart local, Cart remote) {
    final localItems = local.items;
    final remoteItems = remote.items;

    // Merge logic: combine items, sum quantities
    final merged = <String, Item>{...remoteItems};
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

### Example: Checkout Service

**File:** [lib/src/features/checkout/application/fake_checkout_service.dart](lib/src/features/checkout/application/fake_checkout_service.dart:15-60)

```dart
class FakeCheckoutService {
  FakeCheckoutService(this.ref);

  final Ref ref;

  /// Use case: Place order
  Future<void> placeOrder() async {
    final user = ref.read(authStateChangesProvider).value;
    if (user == null) {
      throw Exception('User must be signed in');
    }

    // Get cart
    final cart = await ref.read(cartProvider.future);
    if (cart.items.isEmpty) {
      throw Exception('Cart is empty');
    }

    // Get products for price calculation
    final products = await ref.read(productsListStreamProvider.future);
    final total = cart.totalPrice(products);

    // Create order
    final order = Order(
      id: DateTime.now().toIso8601String(),  // Simple ID generation
      userId: user.uid,
      items: cart.items.map((id, item) => MapEntry(id, item.quantity)),
      orderStatus: OrderStatus.confirmed,
      orderDate: DateTime.now(),
      total: total,
    );

    // Save order
    final ordersRepository = ref.read(ordersRepositoryProvider);
    await ordersRepository.addOrder(user.uid, order);

    // Clear cart
    final cartService = ref.read(cartServiceProvider);
    await cartService._setCart(const Cart());
  }
}
```

**This orchestrates:**
1. Check user authentication
2. Validate cart not empty
3. Calculate total price
4. Create order entity
5. Save to orders repository
6. Clear the cart

### Example: Reviews Service

**File:** [lib/src/features/reviews/application/reviews_service.dart](lib/src/features/reviews/application/reviews_service.dart:15-45)

```dart
class ReviewsService {
  ReviewsService(this.ref);

  final Ref ref;

  /// Use case: Submit review
  Future<void> submitReview({
    required ProductID productId,
    required Review review,
  }) async {
    final user = ref.read(authStateChangesProvider).value;
    if (user == null) {
      throw Exception('User must be signed in to review');
    }

    // Validate rating
    if (review.rating < 1.0 || review.rating > 5.0) {
      throw Exception('Rating must be between 1 and 5');
    }

    // Save review
    final reviewsRepository = ref.read(reviewsRepositoryProvider);
    await reviewsRepository.setReview(productId, user.uid, review);

    // Invalidate reviews cache to refresh UI
    ref.invalidate(productReviewsProvider(productId));
  }
}
```

---

## Presentation Layer

### Overview

The Presentation Layer contains all **UI code**: screens, widgets, and controllers that manage UI state.

**Think of it as:** The dining room and waiters. They present food to customers and take their orders, but don't cook the food themselves.

### Characteristics

- ✅ Flutter widgets
- ✅ UI state management
- ✅ User input handling
- ✅ Navigation
- ✅ Error and loading states
- ✅ Responsive design

### Components

1. **Screens**: Full-page views
2. **Widgets**: Reusable UI components
3. **Controllers**: UI state management with Riverpod

### Example: Product List Screen

**File:** [lib/src/features/products/presentation/products_list/products_list_screen.dart](lib/src/features/products/presentation/products_list/products_list_screen.dart:15-65)

```dart
class ProductsListScreen extends StatefulWidget {
  const ProductsListScreen({super.key});

  @override
  State<ProductsListScreen> createState() => _ProductsListScreenState();
}

class _ProductsListScreenState extends State<ProductsListScreen> {
  String _searchQuery = '';

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: HomeAppBar(
        onSearchChanged: (query) {
          setState(() {
            _searchQuery = query;
          });
        },
      ),
      body: Consumer(
        builder: (context, ref, child) {
          // Watch products from provider
          final productsValue = ref.watch(productsListSearchProvider(_searchQuery));

          // Handle different states
          return AsyncValueWidget<List<Product>>(
            value: productsValue,
            data: (products) => ProductsGrid(products: products),
          );
        },
      ),
    );
  }
}
```

**Key Points:**
- Uses `Consumer` to watch Riverpod state
- Handles search with local state (`setState`)
- `AsyncValueWidget` handles loading/error/data states

### Example: Controller

**File:** [lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart](lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart:15-50)

```dart
@riverpod
class AddToCartController extends _$AddToCartController {
  @override
  FutureOr<void> build() {
    // Initial state - nothing to do
  }

  Future<void> addItem(ProductID productId) async {
    // Get required dependencies
    final cartService = ref.read(cartServiceProvider);
    final product = ref.read(productProvider(productId));

    if (product == null) {
      // Product not found
      state = AsyncError(
        Exception('Product not found'),
        StackTrace.current,
      );
      return;
    }

    // Check available quantity
    final cart = await ref.read(cartProvider.future);
    final currentQuantity = cart.items[productId]?.quantity ?? 0;

    if (currentQuantity >= product.availableQuantity) {
      // Out of stock
      state = AsyncError(
        Exception('Not enough stock'),
        StackTrace.current,
      );
      return;
    }

    // Create item
    final item = Item(productId: productId, quantity: 1);

    // Set loading state
    state = const AsyncLoading<void>();

    // Call service and handle result
    state = await AsyncValue.guard(() => cartService.addItem(item));
  }
}
```

**Controller Pattern:**
1. Extends generated `_$AddToCartController`
2. `build()` returns initial state
3. Methods update state reactively
4. Uses `AsyncValue.guard()` for error handling
5. UI automatically rebuilds when state changes

### Example: Reactive Widget

**File:** [lib/src/features/cart/presentation/cart_total/cart_total_text.dart](lib/src/features/cart/presentation/cart_total/cart_total_text.dart:10-35)

```dart
class CartTotalText extends ConsumerWidget {
  const CartTotalText({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Watch cart
    final cartValue = ref.watch(cartProvider);

    // Watch products
    final productsValue = ref.watch(productsListStreamProvider);

    // Calculate total
    final totalValue = AsyncValue.guard(() async {
      final cart = await cartValue.when(
        data: (cart) => cart,
        loading: () => throw Exception('Loading'),
        error: (e, s) => throw e,
      );

      final products = await productsValue.when(
        data: (products) => products,
        loading: () => throw Exception('Loading'),
        error: (e, s) => throw e,
      );

      return cart.totalPrice(products);
    });

    // Display total
    return totalValue.when(
      data: (total) => Text(
        'Total: \$${total.toStringAsFixed(2)}',
        style: Theme.of(context).textTheme.headlineSmall,
      ),
      loading: () => const CircularProgressIndicator(),
      error: (e, s) => Text('Error: $e'),
    );
  }
}
```

---

## Cross-Layer Communication

### How Layers Interact

```
User Taps Button
       ↓
[Presentation] AddToCartController.addItem()
       ↓
[Application] CartService.addItem()
       ↓
[Domain] Cart.addItem() - Business logic
       ↓
[Data] LocalCartRepository.setCart()
       ↓
Database saves cart
       ↓
[Data] Stream emits new cart
       ↓
[Presentation] UI rebuilds automatically
```

### Dependency Injection with Riverpod

All layer dependencies are managed via Riverpod providers:

```dart
// Data layer provider
@Riverpod(keepAlive: true)
LocalCartRepository localCartRepository(Ref ref) {
  final db = ref.watch(sembastDatabaseProvider).requireValue;
  return SembastCartRepository(db);
}

// Application layer provider (depends on data layer)
@Riverpod(keepAlive: true)
CartService cartService(Ref ref) {
  return CartService(ref);  // Can access repositories via ref
}

// Presentation layer (depends on application layer)
@riverpod
class AddToCartController extends _$AddToCartController {
  Future<void> addItem(ProductID productId) async {
    final service = ref.read(cartServiceProvider);  // Get service
    await service.addItem(item);
  }
}
```

---

## Layer Dependencies

### Dependency Graph

```
       Domain
      ↗     ↖
   Data     Application
      ↘     ↙
    Presentation
```

### Rules

1. **Domain** depends on nothing
2. **Data** depends on Domain only
3. **Application** depends on Domain and Data
4. **Presentation** depends on all layers

### Why This Matters

**Example: Changing Database**

If you switch from Sembast to SQLite:

```
✅ Domain layer: NO CHANGES
✅ Application layer: NO CHANGES
✅ Presentation layer: NO CHANGES
✓ Data layer: Create new SqliteCartRepository
✓ Update provider to use new repository
```

Only the data layer changes!

---

## Summary

### Layer Checklist

| Layer | Contains | Doesn't Contain |
|-------|----------|-----------------|
| **Domain** | Entities, business rules, value objects | UI, database code, frameworks |
| **Data** | Repositories, data sources, persistence | UI, business logic |
| **Application** | Services, use cases, workflows | UI, direct data access |
| **Presentation** | Screens, widgets, controllers | Business logic, database access |

### Key Principles

1. **Separation of Concerns**: Each layer has one responsibility
2. **Dependency Rule**: Outer depends on inner, never the reverse
3. **Testability**: Each layer can be tested independently
4. **Flexibility**: Easy to change implementations

---

## Next Steps

- **[Design Patterns](03-design-patterns.md)**: Explore patterns used in each layer
- **[State Management](04-state-management-and-navigation.md)**: Deep dive into Riverpod
- **[Features](05-features-deep-dive.md)**: See layers working together in real features

---
