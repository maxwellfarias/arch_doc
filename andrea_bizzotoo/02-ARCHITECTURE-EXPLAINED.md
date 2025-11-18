# Architecture Explained - Clean Architecture in Flutter

## Table of Contents
- [What is Software Architecture?](#what-is-software-architecture)
- [Why Clean Architecture?](#why-clean-architecture)
- [The Four Layers](#the-four-layers)
- [Layer Details and Examples](#layer-details-and-examples)
- [Dependency Rules](#dependency-rules)
- [Data Flow Through Layers](#data-flow-through-layers)
- [Feature-First Organization](#feature-first-organization)
- [Real-World Analogy](#real-world-analogy)
- [Benefits of This Architecture](#benefits-of-this-architecture)
- [Common Pitfalls to Avoid](#common-pitfalls-to-avoid)

---

## What is Software Architecture?

Think of software architecture like the blueprint for a building. Just as a building has:
- A **foundation** (strong and stable)
- **Structural walls** (organize rooms and spaces)
- **Plumbing and electrical** (hidden but essential)
- **Interior design** (what people see and interact with)

An app has layers that organize code by responsibility. **Good architecture** makes your code:
- ✅ **Easier to understand** - You know where to find things
- ✅ **Easier to test** - Each piece can be tested independently
- ✅ **Easier to change** - Modify one part without breaking others
- ✅ **Easier to scale** - Add features without creating chaos

---

## Why Clean Architecture?

**Clean Architecture** is a pattern created by Robert C. Martin (Uncle Bob) that separates concerns into layers with strict rules about dependencies.

### The Problem Without Clean Architecture

Imagine a simple Flutter app without layers:

```dart
// ❌ BAD: Everything mixed together
class ProductScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // UI mixed with business logic and data access
    final response = await http.get('https://api.example.com/products');
    final products = jsonDecode(response.body);
    final filteredProducts = products.where((p) => p['price'] < 100);

    return ListView.builder(
      itemCount: filteredProducts.length,
      itemBuilder: (context, index) {
        return ListTile(title: Text(filteredProducts[index]['name']));
      },
    );
  }
}
```

**Problems:**
- 🔴 Can't test business logic without the UI
- 🔴 Can't test without making real HTTP calls
- 🔴 Changing API format breaks the UI
- 🔴 Can't reuse logic in other screens
- 🔴 Hard to understand what the code does

### The Solution: Separation of Concerns

Clean Architecture separates code into layers, each with a single responsibility:

```
┌─────────────────────────────────────────────┐
│         PRESENTATION LAYER (UI)             │  ← What users see
│  Widgets, Screens, Controllers              │
└─────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────┐
│      APPLICATION LAYER (Business Logic)     │  ← Coordinates actions
│  Services, Use Cases                        │
└─────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────┐
│         DATA LAYER (Data Access)            │  ← Gets and saves data
│  Repositories, Data Sources                 │
└─────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────┐
│       DOMAIN LAYER (Business Entities)      │  ← Core business rules
│  Models, Entities (Pure Dart)               │
└─────────────────────────────────────────────┘
```

---

## The Four Layers

This project uses **four distinct layers**. Let's understand each one:

### 1. Domain Layer (Core)
**Location:** `lib/src/features/[feature]/domain/`

**Responsibility:** Represents the business entities and rules.

**Contains:**
- Pure Dart classes (no Flutter imports!)
- Business entities (Product, Cart, Order, User, etc.)
- No logic about where data comes from or how it's displayed

**Analogy:** The "soul" of your business - the fundamental concepts that exist regardless of technology.

**Example Files:**
- [lib/src/features/products/domain/product.dart](../lib/src/features/products/domain/product.dart)
- [lib/src/features/cart/domain/cart.dart](../lib/src/features/cart/domain/cart.dart)
- [lib/src/features/orders/domain/order.dart](../lib/src/features/orders/domain/order.dart)

### 2. Data Layer
**Location:** `lib/src/features/[feature]/data/`

**Responsibility:** Handles data access and persistence.

**Contains:**
- Repositories (abstract interfaces and implementations)
- Data sources (local database, remote API, cache)
- Data transfer objects (DTOs) for API responses
- Data mapping/serialization logic

**Analogy:** The "librarian" - knows where all the books (data) are stored and how to retrieve them.

**Example Files:**
- [lib/src/features/products/data/fake_products_repository.dart](../lib/src/features/products/data/fake_products_repository.dart)
- [lib/src/features/cart/data/local/sembast_cart_repository.dart](../lib/src/features/cart/data/local/sembast_cart_repository.dart)
- [lib/src/features/authentication/data/fake_auth_repository.dart](../lib/src/features/authentication/data/fake_auth_repository.dart)

### 3. Application Layer
**Location:** `lib/src/features/[feature]/application/`

**Responsibility:** Orchestrates the business logic.

**Contains:**
- Services that coordinate multiple repositories
- Complex business operations
- Use cases (single-purpose actions)
- Application-specific logic

**Analogy:** The "manager" - coordinates between different departments to get work done.

**Example Files:**
- [lib/src/features/cart/application/cart_service.dart](../lib/src/features/cart/application/cart_service.dart)
- [lib/src/features/cart/application/cart_sync_service.dart](../lib/src/features/cart/application/cart_sync_service.dart)
- [lib/src/features/checkout/application/fake_checkout_service.dart](../lib/src/features/checkout/application/fake_checkout_service.dart)

### 4. Presentation Layer
**Location:** `lib/src/features/[feature]/presentation/`

**Responsibility:** Everything the user sees and interacts with.

**Contains:**
- Screens and pages
- Widgets (buttons, forms, lists)
- Controllers (state management)
- UI-specific logic (validation, formatting)

**Analogy:** The "storefront" - what customers see and interact with.

**Example Files:**
- [lib/src/features/products/presentation/products_list/products_list_screen.dart](../lib/src/features/products/presentation/products_list/products_list_screen.dart)
- [lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart](../lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart)
- [lib/src/features/authentication/presentation/sign_in/email_password_sign_in_screen.dart](../lib/src/features/authentication/presentation/sign_in/email_password_sign_in_screen.dart)

---

## Layer Details and Examples

### Domain Layer - Deep Dive

The domain layer contains your **business entities** - the "things" your app deals with.

#### Example: Product Entity

**File:** [lib/src/features/products/domain/product.dart](../lib/src/features/products/domain/product.dart)

```dart
/// Represents a product in the e-commerce catalog
class Product {
  const Product({
    required this.id,
    required this.imageUrl,
    required this.title,
    required this.description,
    required this.price,
    required this.availableQuantity,
    this.avgRating = 0,
    this.numRatings = 0,
  });

  // Unique identifier
  final ProductID id;

  // Display information
  final String imageUrl;
  final String title;
  final String description;

  // Pricing and inventory
  final double price;
  final int availableQuantity;

  // Social proof
  final double avgRating;
  final int numRatings;

  // Computed property (business logic in domain)
  String get numRatingsLabel => '($numRatings)';
}

// Type alias for clarity and type safety
typedef ProductID = String;
```

**Key Characteristics:**
- ✅ **Immutable** - Uses `const` constructor, all fields are `final`
- ✅ **Pure Dart** - No Flutter imports
- ✅ **Self-contained** - All product-related data in one place
- ✅ **Type-safe** - Uses `ProductID` type alias instead of raw String

#### Example: Cart Entity with Business Logic

**File:** [lib/src/features/cart/domain/cart.dart](../lib/src/features/cart/domain/cart.dart)

```dart
/// Represents a shopping cart containing items
class Cart {
  const Cart([this.items = const {}]);

  // Map of ProductID to quantity
  final Map<ProductID, int> items;

  // Computed properties (read-only)
  int get length => items.length;

  List<ProductID> get productIds => items.keys.toList();

  int get itemsCount => items.values.fold(0, (sum, qty) => sum + qty);

  // Serialization
  Map<String, dynamic> toJson() => {'items': items};

  factory Cart.fromJson(Map<String, dynamic> json) {
    return Cart(Map<ProductID, int>.from(json['items']));
  }
}
```

**Business Logic via Extension:**

```dart
/// Extension methods for cart mutations
/// (Keeps Cart class immutable while allowing operations)
extension MutableCart on Cart {
  /// Returns a NEW cart with the item set to the specified quantity
  Cart setItem(Item item) {
    final copy = Map<ProductID, int>.from(items);
    copy[item.productId] = item.quantity;
    return Cart(copy);
  }

  /// Returns a NEW cart with the item's quantity increased
  Cart addItem(Item item) {
    final copy = Map<ProductID, int>.from(items);
    copy.update(
      item.productId,
      (existingQty) => item.quantity + existingQty,
      ifAbsent: () => item.quantity,
    );
    return Cart(copy);
  }

  /// Returns a NEW cart with all items added
  Cart addItems(List<Item> itemsToAdd) {
    var result = this;
    for (final item in itemsToAdd) {
      result = result.addItem(item);
    }
    return result;
  }

  /// Returns a NEW cart without the specified product
  Cart removeItemById(ProductID productId) {
    final copy = Map<ProductID, int>.from(items);
    copy.remove(productId);
    return Cart(copy);
  }
}
```

**Why Immutable?**
- ✅ Predictable - Can't be modified accidentally
- ✅ Thread-safe - Safe to use across async operations
- ✅ Easier to test - Given same input, always same output
- ✅ Time-travel debugging - Can store history of states

---

### Data Layer - Deep Dive

The data layer **abstracts where data comes from**. The rest of the app doesn't care if data is from:
- A remote API
- Local database
- In-memory cache
- Mock/fake data

#### Example: Repository Interface

```dart
/// Abstract interface for cart data access
abstract class LocalCartRepository {
  Future<Cart> fetchCart();
  Stream<Cart> watchCart();
  Future<void> setCart(Cart cart);
}
```

This is a **contract** - any class implementing this must provide these methods.

#### Example: Sembast Implementation

**File:** [lib/src/features/cart/data/local/sembast_cart_repository.dart](../lib/src/features/cart/data/local/sembast_cart_repository.dart)

```dart
/// Local cart repository using Sembast database
class SembastCartRepository implements LocalCartRepository {
  SembastCartRepository(this.db);

  final Database db;
  final store = StoreRef.main();
  static const cartItemsKey = 'cartItems';

  /// Fetch cart from local database
  @override
  Future<Cart> fetchCart() async {
    final cartJson = await store.record(cartItemsKey).get(db) as String?;
    if (cartJson != null) {
      return Cart.fromJson(json.decode(cartJson));
    }
    return const Cart();
  }

  /// Watch cart changes as a stream
  @override
  Stream<Cart> watchCart() {
    return store.record(cartItemsKey).onSnapshot(db).map((snapshot) {
      if (snapshot != null) {
        return Cart.fromJson(json.decode(snapshot.value as String));
      } else {
        return const Cart();
      }
    });
  }

  /// Save cart to local database
  @override
  Future<void> setCart(Cart cart) {
    return store.record(cartItemsKey).put(db, json.encode(cart.toJson()));
  }

  /// Factory to create repository with default database
  static Future<SembastCartRepository> makeDefault() async {
    return SembastCartRepository(await createDatabase());
  }

  static Future<Database> createDatabase() async {
    final appDir = await getApplicationDocumentsDirectory();
    return databaseFactoryIo.openDatabase('${appDir.path}/default.db');
  }
}
```

**Key Points:**
- ✅ Implements the interface contract
- ✅ Handles database-specific code (Sembast)
- ✅ Converts between Cart entities and JSON
- ✅ Provides streams for reactive updates
- ✅ Can be swapped with a different implementation

#### Example: Fake Repository (for demos)

**File:** [lib/src/features/products/data/fake_products_repository.dart](../lib/src/features/products/data/fake_products_repository.dart)

```dart
/// Fake products repository using in-memory data
class FakeProductsRepository {
  FakeProductsRepository({this.addDelay = true});

  final bool addDelay;

  // In-memory store with reactive stream
  final _products = InMemoryStore<List<Product>>(List.from(kTestProducts));

  /// Get products list synchronously
  List<Product> getProductsList() => _products.value;

  /// Get a single product by ID
  Product? getProduct(String id) {
    return _getProduct(_products.value, id);
  }

  /// Fetch products (async, with optional delay to simulate network)
  Future<List<Product>> fetchProductsList() async {
    return Future.value(_products.value);
  }

  /// Watch products as a stream
  Stream<List<Product>> watchProductsList() => _products.stream;

  /// Watch a single product as a stream
  Stream<Product?> watchProduct(String id) {
    return watchProductsList().map((products) => _getProduct(products, id));
  }

  /// Search products by query string
  Future<List<Product>> searchProducts(String query) async {
    final productsList = await fetchProductsList();
    return productsList
        .where((product) =>
            product.title.toLowerCase().contains(query.toLowerCase()))
        .toList();
  }

  /// Helper to find product by ID
  static Product? _getProduct(List<Product> products, String id) {
    try {
      return products.firstWhere((product) => product.id == id);
    } catch (e) {
      return null;
    }
  }

  /// Cleanup
  void dispose() => _products.close();
}
```

**InMemoryStore Pattern:**

**File:** [lib/src/utils/in_memory_store.dart](../lib/src/utils/in_memory_store.dart)

```dart
/// A simple in-memory store backed by RxDart's BehaviorSubject
class InMemoryStore<T> {
  InMemoryStore(T initial) : _subject = BehaviorSubject<T>.seeded(initial);

  final BehaviorSubject<T> _subject;

  /// Stream of value changes
  Stream<T> get stream => _subject.stream;

  /// Current value
  T get value => _subject.value;

  /// Update value (triggers stream event)
  set value(T value) => _subject.add(value);

  /// Cleanup
  void close() => _subject.close();
}
```

This gives us **reactive data** - when the value changes, all listeners are notified automatically!

---

### Application Layer - Deep Dive

The application layer contains **business logic** that coordinates between multiple repositories or performs complex operations.

#### Example: Cart Service

**File:** [lib/src/features/cart/application/cart_service.dart](../lib/src/features/cart/application/cart_service.dart)

```dart
/// Service for managing cart operations
/// Coordinates between local and remote cart repositories
class CartService {
  CartService(this.ref);
  final Ref ref;

  /// Fetch cart based on authentication status
  Future<Cart> _fetchCart() {
    final user = ref.read(authRepositoryProvider).currentUser;
    if (user != null) {
      // Logged in: use remote cart
      return ref.read(remoteCartRepositoryProvider).fetchCart(user.uid);
    } else {
      // Guest: use local cart
      return ref.read(localCartRepositoryProvider).fetchCart();
    }
  }

  /// Save cart to appropriate repository
  Future<void> _setCart(Cart cart) async {
    final user = ref.read(authRepositoryProvider).currentUser;
    if (user != null) {
      await ref.read(remoteCartRepositoryProvider).setCart(user.uid, cart);
    } else {
      await ref.read(localCartRepositoryProvider).setCart(cart);
    }
  }

  /// Set item to specific quantity
  Future<void> setItem(Item item) async {
    final cart = await _fetchCart();
    final updated = cart.setItem(item);
    await _setCart(updated);
  }

  /// Add item (increases quantity if already in cart)
  Future<void> addItem(Item item) async {
    final product = await ref.read(productsRepositoryProvider).fetchProduct(item.productId);
    if (product == null) {
      throw ProductNotFoundException();
    }

    final cart = await _fetchCart();
    final updated = cart.addItem(item);
    await _setCart(updated);
  }

  /// Remove item from cart
  Future<void> removeItemById(ProductID productId) async {
    final cart = await _fetchCart();
    final updated = cart.removeItemById(productId);
    await _setCart(updated);
  }
}
```

**Why a Service Layer?**
- ✅ **Encapsulates complex logic** - Hides the auth-based cart switching
- ✅ **Coordinates multiple repositories** - Auth + Cart repositories
- ✅ **Validates business rules** - Checks product exists before adding
- ✅ **Reusable** - Can be called from multiple UI screens

#### Example: Cart Sync Service

**File:** [lib/src/features/cart/application/cart_sync_service.dart](../lib/src/features/cart/application/cart_sync_service.dart)

```dart
/// Service that automatically syncs local cart to remote when user signs in
class CartSyncService {
  CartSyncService(this.ref) {
    _init();
  }

  final Ref ref;

  void _init() {
    // Listen for auth state changes
    ref.listen<AsyncValue<AppUser?>>(
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

  /// Move all items from local cart to remote cart
  Future<void> _moveItemsToRemoteCart(String uid) async {
    try {
      // 1. Get current carts
      final localCartRepository = ref.read(localCartRepositoryProvider);
      final remoteCartRepository = ref.read(remoteCartRepositoryProvider);

      final localCart = await localCartRepository.fetchCart();
      final remoteCart = await remoteCartRepository.fetchCart(uid);

      // 2. Calculate items to add (respecting inventory limits)
      final localItemsToAdd = await _getLocalItemsToAdd(localCart, remoteCart);

      // 3. Merge into remote cart
      final updatedRemoteCart = remoteCart.addItems(localItemsToAdd);
      await remoteCartRepository.setCart(uid, updatedRemoteCart);

      // 4. Clear local cart
      await localCartRepository.setCart(const Cart());
    } catch (e) {
      // Error handling...
      ref.read(errorLoggerProvider).logError(e, StackTrace.current);
    }
  }

  /// Calculate which items can be added (considering available quantities)
  Future<List<Item>> _getLocalItemsToAdd(Cart localCart, Cart remoteCart) async {
    // Complex business logic here...
    // Checks inventory, merges quantities, etc.
  }
}
```

**Why This Service Exists:**
- ✅ **Automatic behavior** - User doesn't trigger this manually
- ✅ **Complex orchestration** - Merges two carts with inventory checks
- ✅ **Side effect management** - Happens in response to auth changes
- ✅ **Error handling** - Gracefully handles failures

---

### Presentation Layer - Deep Dive

The presentation layer is everything the user **sees and interacts with**.

#### Example: Product List Screen

**File:** [lib/src/features/products/presentation/products_list/products_list_screen.dart](../lib/src/features/products/presentation/products_list/products_list_screen.dart)

```dart
/// Screen showing list of all products
class ProductsListScreen extends StatelessWidget {
  const ProductsListScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Products'),
        actions: const [ShoppingCartIcon()],
      ),
      body: CustomScrollView(
        slivers: [
          const ProductsSearchBar(),
          Consumer(
            builder: (context, ref, child) {
              // Watch the products provider
              final productsListValue = ref.watch(productsListStreamProvider);

              // Handle loading/error/data states
              return AsyncValueWidget<List<Product>>(
                value: productsListValue,
                data: (products) => ProductsLayoutGrid(
                  itemCount: products.length,
                  itemBuilder: (_, index) {
                    final product = products[index];
                    return ProductCard(product: product);
                  },
                ),
              );
            },
          ),
        ],
      ),
    );
  }
}
```

**Key Presentation Patterns:**

1. **Consumer Widget** - Listens to Riverpod providers
2. **AsyncValueWidget** - Handles loading/error/data states
3. **Separation** - Screen composes smaller widgets

#### Example: Controller for User Actions

**File:** [lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart](../lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart)

```dart
/// Controller for adding items to cart
@riverpod
class AddToCartController extends _$AddToCartController {
  @override
  FutureOr<void> build() {
    // Initial state is "ready" (no loading)
  }

  /// Add item to cart
  Future<void> addItem(ProductID productId) async {
    // Get dependencies
    final cartService = ref.read(cartServiceProvider);
    final quantity = ref.read(itemQuantityControllerProvider);
    final item = Item(productId: productId, quantity: quantity);

    // Set loading state
    state = const AsyncLoading<void>();

    // Perform action with error handling
    state = await AsyncValue.guard(() => cartService.addItem(item));

    // If successful, reset quantity to 1
    if (!state.hasError) {
      ref.read(itemQuantityControllerProvider.notifier).updateQuantity(1);
    }
  }
}
```

**Controller Pattern Benefits:**
- ✅ **Separates UI from logic** - Widget just calls controller
- ✅ **Handles async states** - Loading, success, error
- ✅ **Testable** - Can test without UI
- ✅ **Reusable** - Multiple widgets can use same controller

---

## Dependency Rules

The most important rule in Clean Architecture: **Dependencies point inward**.

```
┌──────────────────────────────────────────────┐
│  PRESENTATION (can depend on all layers)     │
│  ↓ (knows about)                             │
└──────────────────────────────────────────────┘
┌──────────────────────────────────────────────┐
│  APPLICATION (can depend on Data & Domain)   │
│  ↓ (knows about)                             │
└──────────────────────────────────────────────┘
┌──────────────────────────────────────────────┐
│  DATA (can depend on Domain only)            │
│  ↓ (knows about)                             │
└──────────────────────────────────────────────┘
┌──────────────────────────────────────────────┐
│  DOMAIN (depends on NOTHING!)                │
│  ← Pure business logic                       │
└──────────────────────────────────────────────┘
```

### What This Means:

✅ **Domain** can import: Nothing (pure Dart)
✅ **Data** can import: Domain models
✅ **Application** can import: Domain models, Data repositories
✅ **Presentation** can import: Everything

❌ **Domain** CANNOT import Data, Application, or Presentation
❌ **Data** CANNOT import Application or Presentation
❌ **Application** CANNOT import Presentation

### Why These Rules?

**Example Violation:**
```dart
// ❌ BAD: Domain importing from Data layer
class Product {
  void saveToDatabase() {
    SembastDatabase.save(this);  // Domain knows about database!
  }
}
```

**Problems:**
- Can't test Product without database
- Can't switch to different database
- Domain logic tied to infrastructure

**Correct Approach:**
```dart
// ✅ GOOD: Domain is pure
class Product {
  const Product({...});
  // No save method - that's the repository's job!
}

// Repository handles persistence
class ProductsRepository {
  Future<void> saveProduct(Product product) {
    return database.save(product);
  }
}
```

---

## Data Flow Through Layers

Let's trace a complete user action through all layers:

### Scenario: User Adds Product to Cart

```
┌─────────────────────────────────────────────────────────┐
│ 1. USER TAPS "ADD TO CART" BUTTON                       │
│    Location: Presentation Layer (UI Widget)             │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ 2. WIDGET CALLS CONTROLLER                              │
│    File: add_to_cart_widget.dart                        │
│    Code: controller.addItem(productId)                  │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ 3. CONTROLLER PROCESSES ACTION                          │
│    Layer: Presentation                                  │
│    File: add_to_cart_controller.dart                    │
│    - Sets loading state                                 │
│    - Calls CartService                                  │
│    - Handles errors                                     │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ 4. SERVICE ORCHESTRATES BUSINESS LOGIC                  │
│    Layer: Application                                   │
│    File: cart_service.dart                              │
│    - Checks if user is authenticated                    │
│    - Validates product exists                           │
│    - Fetches current cart                               │
│    - Updates cart (immutably)                           │
│    - Saves updated cart                                 │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ 5. REPOSITORY HANDLES DATA PERSISTENCE                  │
│    Layer: Data                                          │
│    File: sembast_cart_repository.dart                   │
│    - Converts Cart entity to JSON                       │
│    - Saves to Sembast database                          │
│    - Emits stream event (cart changed!)                │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ 6. STREAM UPDATE FLOWS BACK UP                          │
│    - Repository stream emits new cart                   │
│    - Riverpod provider detects change                   │
│    - All watching widgets notified                      │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ 7. UI REBUILDS AUTOMATICALLY                            │
│    - Cart icon badge shows new count                    │
│    - Cart screen shows new item                         │
│    - "Add to Cart" button resets                        │
└─────────────────────────────────────────────────────────┘
```

### Code Flow Example

**Step 1-2: UI Widget**
```dart
// Presentation layer
ElevatedButton(
  onPressed: () {
    ref.read(addToCartControllerProvider.notifier).addItem(productId);
  },
  child: Text('Add to Cart'),
)
```

**Step 3: Controller**
```dart
// Presentation layer
Future<void> addItem(ProductID productId) async {
  final item = Item(productId: productId, quantity: 1);
  state = const AsyncLoading();
  state = await AsyncValue.guard(() =>
    ref.read(cartServiceProvider).addItem(item)  // Call service
  );
}
```

**Step 4: Service**
```dart
// Application layer
Future<void> addItem(Item item) async {
  // Validate
  final product = await ref.read(productsRepositoryProvider).fetchProduct(item.productId);
  if (product == null) throw ProductNotFoundException();

  // Get current state
  final cart = await _fetchCart();  // Uses repository

  // Apply business logic
  final updated = cart.addItem(item);  // Domain logic

  // Persist
  await _setCart(updated);  // Uses repository
}
```

**Step 5: Repository**
```dart
// Data layer
Future<void> setCart(Cart cart) async {
  final json = cart.toJson();  // Domain entity → JSON
  await store.record(cartItemsKey).put(db, json);  // Persist
}

Stream<Cart> watchCart() {
  return store.record(cartItemsKey).onSnapshot(db).map((snapshot) {
    return Cart.fromJson(snapshot.value);  // JSON → Domain entity
  });
}
```

**Step 6-7: Reactive Update**
```dart
// Presentation layer
Consumer(
  builder: (context, ref, child) {
    final cart = ref.watch(cartProvider);  // Auto-rebuilds!
    return CartBadge(count: cart.itemsCount);
  },
)
```

---

## Feature-First Organization

This project organizes code by **feature** rather than by **type**.

### Traditional (Type-First) ❌

```
lib/
├── models/
│   ├── product.dart
│   ├── cart.dart
│   ├── order.dart
│   └── user.dart
├── repositories/
│   ├── products_repository.dart
│   ├── cart_repository.dart
│   └── auth_repository.dart
├── services/
│   ├── cart_service.dart
│   └── checkout_service.dart
└── screens/
    ├── products_screen.dart
    ├── cart_screen.dart
    └── checkout_screen.dart
```

**Problems:**
- Have to jump between many folders
- Hard to see what a feature does
- Can't easily extract or reuse features

### Feature-First (This Project) ✅

```
lib/src/features/
├── products/
│   ├── domain/
│   │   └── product.dart
│   ├── data/
│   │   └── fake_products_repository.dart
│   └── presentation/
│       ├── products_list_screen.dart
│       └── product_screen.dart
│
├── cart/
│   ├── domain/
│   │   └── cart.dart
│   ├── data/
│   │   └── sembast_cart_repository.dart
│   ├── application/
│   │   ├── cart_service.dart
│   │   └── cart_sync_service.dart
│   └── presentation/
│       ├── shopping_cart_screen.dart
│       └── add_to_cart_widget.dart
│
└── checkout/
    ├── application/
    │   └── fake_checkout_service.dart
    └── presentation/
        └── checkout_screen.dart
```

**Benefits:**
- ✅ All cart code in one place
- ✅ Easy to understand feature scope
- ✅ Can move/extract features easily
- ✅ Clear boundaries between features
- ✅ Teams can work on different features independently

---

## Real-World Analogy

Think of this architecture like a **restaurant**:

### Domain Layer = Menu Items
- The dishes you serve (Burger, Pizza, Salad)
- Exist regardless of how they're made or served
- Core business concepts

### Data Layer = Kitchen & Storage
- Where ingredients are stored (refrigerator, pantry)
- How food is prepared and retrieved
- Abstracts the "how" from the "what"

### Application Layer = Chef/Manager
- Coordinates the whole operation
- Combines ingredients following recipes
- Handles special requests
- Manages timing and order

### Presentation Layer = Waiters & Dining Room
- What customers see and interact with
- Takes orders
- Delivers food
- Handles customer communication

**Example Flow: Customer Orders Burger**

1. **Presentation**: Waiter takes order
2. **Application**: Manager coordinates (checks inventory, assigns chef, plans timing)
3. **Data**: Kitchen retrieves ingredients from storage
4. **Domain**: The burger itself (the concept and recipe)
5. **Data**: Kitchen prepares burger
6. **Application**: Manager ensures quality and completion
7. **Presentation**: Waiter serves burger to customer

Each layer has a clear job, and you can change one without affecting others (new waiter, new chef, new menu item, new storage system).

---

## Benefits of This Architecture

### 1. Testability
```dart
// Can test domain logic without UI or database
test('Cart.addItem increases quantity', () {
  final cart = Cart({'product-1': 2});
  final updated = cart.addItem(Item(productId: 'product-1', quantity: 3));
  expect(updated.items['product-1'], 5);
});

// Can test service with mock repositories
test('CartService uses remote cart when logged in', () async {
  // Setup mocks...
  final service = CartService(mockRef);
  await service.addItem(item);
  verify(() => mockRemoteRepo.setCart(any(), any()));
});
```

### 2. Flexibility
- Want to switch from Sembast to Hive? Just change the Data layer!
- Want to add a real API? Implement a new repository!
- Want to redesign UI? Presentation layer is isolated!

### 3. Scalability
- Add features without affecting existing ones
- Multiple teams can work in parallel
- Code is organized and predictable

### 4. Maintainability
- Easy to find where things are
- Changes are localized to one layer
- Clear separation of concerns

### 5. Reusability
- Services can be used by multiple screens
- Repositories can serve multiple features
- Domain models are pure and portable

---

## Common Pitfalls to Avoid

### ❌ Pitfall 1: Skipping Layers
```dart
// BAD: Widget directly accessing repository
class ProductScreen extends ConsumerWidget {
  Widget build(context, ref) {
    final products = ref.watch(productsRepositoryProvider).getProducts();
    // ...
  }
}
```

**Fix:** Always go through proper layers (use services or providers).

### ❌ Pitfall 2: Domain Knowing About Flutter
```dart
// BAD: Domain importing Flutter
import 'package:flutter/material.dart';

class Product {
  Color get priceColor => price > 100 ? Colors.red : Colors.green;
}
```

**Fix:** Keep domain pure Dart. UI logic belongs in presentation.

### ❌ Pitfall 3: Business Logic in Widgets
```dart
// BAD: Complex logic in widget
class CartScreen extends StatelessWidget {
  Widget build(context) {
    final total = cart.items.entries.map((e) {
      final product = products.firstWhere((p) => p.id == e.key);
      return product.price * e.value;
    }).reduce((a, b) => a + b);
    // ...
  }
}
```

**Fix:** Move to domain entity, service, or controller.

### ❌ Pitfall 4: Tight Coupling
```dart
// BAD: Directly creating dependencies
class CartService {
  final repo = SembastCartRepository();  // Hardcoded!
}
```

**Fix:** Use dependency injection (Riverpod providers).

---

## Summary

Clean Architecture with feature-first organization provides:

✅ **Clear separation** of concerns across layers
✅ **Testable** code at every level
✅ **Flexible** - easy to swap implementations
✅ **Scalable** - grows with your app
✅ **Maintainable** - easy to find and change code
✅ **Team-friendly** - multiple developers can work in parallel

**The Four Layers:**
1. **Domain** - Pure business entities (Product, Cart, Order)
2. **Data** - Repositories and data sources
3. **Application** - Services and business logic
4. **Presentation** - UI screens, widgets, controllers

**The Golden Rule:**
> Dependencies point inward. Domain knows nothing. Presentation knows everything.

---

## What's Next?

Now that you understand the architecture, it's time to learn **how state flows through these layers** using **Riverpod**.

Head to **[03-RIVERPOD-COMPLETE-GUIDE.md](03-RIVERPOD-COMPLETE-GUIDE.md)** to master:
- What Riverpod is and why it's powerful
- All provider types explained
- Reading vs watching providers
- Dependency injection patterns
- Code generation
- And much more!

Ready to level up? Let's go! 🚀
