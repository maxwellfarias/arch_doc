# Domain and Data Layers - The Foundation of Clean Architecture

## Table of Contents
- [Introduction](#introduction)
- [Domain Layer Deep Dive](#domain-layer-deep-dive)
- [Immutability Explained](#immutability-explained)
- [Extension Methods for Business Logic](#extension-methods-for-business-logic)
- [Type Aliases for Type Safety](#type-aliases-for-type-safety)
- [Data Layer Deep Dive](#data-layer-deep-dive)
- [Repository Pattern](#repository-pattern)
- [Local Storage with Sembast](#local-storage-with-sembast)
- [In-Memory Store Pattern](#in-memory-store-pattern)
- [Fake Repositories for Development](#fake-repositories-for-development)
- [Data Serialization](#data-serialization)
- [Testing Domain and Data Layers](#testing-domain-and-data-layers)
- [Best Practices](#best-practices)

---

## Introduction

The **Domain** and **Data** layers form the **foundation** of this application. Understanding these layers is crucial because:

- **Domain Layer** = Your business entities (the "what")
- **Data Layer** = How you access and persist data (the "where")

Together, they create a **solid, testable foundation** that the rest of the app builds upon.

### Layer Relationship

```
┌──────────────────────────────────────┐
│     APPLICATION LAYER                │
│  (Uses domain entities)              │
│  (Calls repositories)                │
└──────────────────────────────────────┘
              ↓  ↓
┌──────────────┐  ┌──────────────────┐
│  DOMAIN      │  │  DATA            │
│  (Entities)  │←─│  (Repositories)  │
│              │  │  (Data Sources)  │
└──────────────┘  └──────────────────┘
```

---

## Domain Layer Deep Dive

The domain layer contains **pure business entities** - the core concepts of your application.

### Characteristics of Domain Entities

✅ **Pure Dart** - No Flutter imports
✅ **Immutable** - Uses `const` constructors and `final` fields
✅ **Self-contained** - All related data in one place
✅ **Serializable** - Can convert to/from JSON
✅ **Business logic** - Via methods or extensions

### Example 1: Product Entity

**File:** [lib/src/features/products/domain/product.dart](../lib/src/features/products/domain/product.dart)

```dart
/// Product entity representing an item in the catalog
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

  // Computed property (read-only)
  String get numRatingsLabel => '($numRatings)';

  // Equality and hashCode for value comparison
  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is Product && other.id == id;
  }

  @override
  int get hashCode => id.hashCode;

  @override
  String toString() => 'Product(id: $id, title: $title, price: $price)';
}

// Type alias for clarity and type safety
typedef ProductID = String;
```

**Key Points:**

1. **Type Alias**: `ProductID` is clearer than raw `String`
2. **Const Constructor**: Enables compile-time constants
3. **Final Fields**: Cannot be modified after creation
4. **Computed Property**: `numRatingsLabel` derives from `numRatings`
5. **Equality**: Based on `id` (business key)
6. **No Setters**: Immutable by design

### Example 2: Cart Entity

**File:** [lib/src/features/cart/domain/cart.dart](../lib/src/features/cart/domain/cart.dart)

```dart
/// Shopping cart containing product items and quantities
class Cart {
  const Cart([this.items = const {}]);

  /// Map of ProductID to quantity
  final Map<ProductID, int> items;

  // Computed properties
  int get length => items.length;

  List<ProductID> get productIds => items.keys.toList();

  int get itemsCount => items.values.fold(0, (sum, qty) => sum + qty);

  // Serialization
  Map<String, dynamic> toJson() => {
        'items': items,
      };

  factory Cart.fromJson(Map<String, dynamic> json) {
    return Cart(Map<ProductID, int>.from(json['items']));
  }

  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is Cart && _mapEquals(other.items, items);
  }

  @override
  int get hashCode => items.hashCode;

  @override
  String toString() => 'Cart(items: $items)';
}

// Deep equality for maps
bool _mapEquals<K, V>(Map<K, V>? a, Map<K, V>? b) {
  if (a == null) return b == null;
  if (b == null || a.length != b.length) return false;
  if (identical(a, b)) return true;
  for (final key in a.keys) {
    if (!b.containsKey(key) || b[key] != a[key]) {
      return false;
    }
  }
  return true;
}
```

**Key Points:**

1. **Default Parameter**: Empty cart by default
2. **Computed Properties**: Derived from `items` map
3. **Serialization**: Converts to/from JSON
4. **Deep Equality**: Compares map contents, not reference
5. **Business Logic**: See extension methods below

### Example 3: Item (Cart Item)

**File:** [lib/src/features/cart/domain/item.dart](../lib/src/features/cart/domain/item.dart)

```dart
/// Represents a product with a quantity in the cart
class Item {
  const Item({
    required this.productId,
    required this.quantity,
  });

  final ProductID productId;
  final int quantity;

  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is Item &&
        other.productId == productId &&
        other.quantity == quantity;
  }

  @override
  int get hashCode => Object.hash(productId, quantity);

  @override
  String toString() => 'Item(productId: $productId, quantity: $quantity)';
}
```

**Simple Value Object:**
- Two properties
- Immutable
- Value equality
- Used to represent cart items

### Example 4: Order Entity

**File:** [lib/src/features/orders/domain/order.dart](../lib/src/features/orders/domain/order.dart)

```dart
enum OrderStatus { confirmed, shipped, delivered }

/// Order entity representing a completed purchase
class Order {
  const Order({
    required this.id,
    required this.userId,
    required this.items,
    required this.orderStatus,
    required this.orderDate,
    required this.total,
  });

  final String id;
  final String userId;
  final Map<ProductID, int> items;
  final OrderStatus orderStatus;
  final DateTime orderDate;
  final double total;

  // Serialization
  Map<String, dynamic> toJson() => {
        'id': id,
        'userId': userId,
        'items': items,
        'orderStatus': orderStatus.name,
        'orderDate': orderDate.toIso8601String(),
        'total': total,
      };

  factory Order.fromJson(Map<String, dynamic> json) {
    return Order(
      id: json['id'] as String,
      userId: json['userId'] as String,
      items: Map<ProductID, int>.from(json['items'] as Map),
      orderStatus: OrderStatus.values.byName(json['orderStatus'] as String),
      orderDate: DateTime.parse(json['orderDate'] as String),
      total: (json['total'] as num).toDouble(),
    );
  }

  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is Order && other.id == id;
  }

  @override
  int get hashCode => id.hashCode;

  @override
  String toString() => 'Order(id: $id, status: $orderStatus, total: $total)';
}
```

**Enum Usage:**
- `OrderStatus` enum for type-safe status
- Serializes as string (`confirmed`, `shipped`, `delivered`)
- Deserializes with `byName`

### Example 5: Review Entity

**File:** [lib/src/features/reviews/domain/review.dart](../lib/src/features/reviews/domain/review.dart)

```dart
/// Product review with rating and comment
class Review {
  const Review({
    required this.rating,
    required this.comment,
    required this.date,
  });

  final double rating;  // 1.0 to 5.0
  final String comment;
  final DateTime date;

  Map<String, dynamic> toJson() => {
        'rating': rating,
        'comment': comment,
        'date': date.toIso8601String(),
      };

  factory Review.fromJson(Map<String, dynamic> json) {
    return Review(
      rating: (json['rating'] as num).toDouble(),
      comment: json['comment'] as String,
      date: DateTime.parse(json['date'] as String),
    );
  }

  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is Review &&
        other.rating == rating &&
        other.comment == comment &&
        other.date == date;
  }

  @override
  int get hashCode => Object.hash(rating, comment, date);

  @override
  String toString() => 'Review(rating: $rating, date: $date)';
}
```

### Example 6: User Entity

**File:** [lib/src/features/authentication/domain/app_user.dart](../lib/src/features/authentication/domain/app_user.dart)

```dart
/// User entity representing an authenticated user
class AppUser {
  const AppUser({
    required this.uid,
    this.email,
  });

  final String uid;
  final String? email;

  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is AppUser && other.uid == uid;
  }

  @override
  int get hashCode => uid.hashCode;

  @override
  String toString() => 'AppUser(uid: $uid, email: $email)';
}
```

**Nullable Field:**
- `email` is optional (nullable)
- Some auth methods might not provide email

---

## Immutability Explained

Immutability is a **core principle** in this architecture. Let's understand why.

### What is Immutability?

**Immutable** = Once created, cannot be changed.

```dart
// Immutable (this project)
final cart = Cart({'product-1': 2});
// cart.items['product-2'] = 1;  // ERROR: Can't modify!

// To "change" it, create a new one
final updatedCart = cart.addItem(Item(productId: 'product-2', quantity: 1));
```

### Why Immutability?

#### 1. **Predictable State**

```dart
void processCart(Cart cart) {
  // With mutable objects
  someFunction(cart);  // Did it modify cart? Who knows!

  // With immutable objects
  final newCart = someFunction(cart);  // Original cart unchanged!
}
```

#### 2. **Thread Safety**

```dart
// Immutable objects can be shared across async operations safely
final cart = Cart({'product-1': 2});

// Both can read cart without conflicts
await Future.wait([
  operation1(cart),
  operation2(cart),
]);
```

#### 3. **Time-Travel Debugging**

```dart
// Can store history of states
final history = <Cart>[
  Cart(),                              // Empty
  Cart({'product-1': 1}),             // Added product-1
  Cart({'product-1': 1, 'product-2': 2}),  // Added product-2
];

// Can undo/redo easily
final previousState = history[history.length - 2];
```

#### 4. **Easier Testing**

```dart
test('addItem increases quantity', () {
  final cart = Cart({'product-1': 2});
  final updated = cart.addItem(Item(productId: 'product-1', quantity: 3));

  // Original unchanged
  expect(cart.items['product-1'], 2);

  // New cart has updated quantity
  expect(updated.items['product-1'], 5);
});
```

#### 5. **Flutter Rebuilds**

```dart
// Flutter compares objects to decide if rebuild is needed
final oldCart = Cart({'product-1': 2});
final newCart = Cart({'product-1': 2});

// With proper equality, Flutter knows these are the same
oldCart == newCart  // true - no rebuild needed
```

### How to Work with Immutable Objects

Instead of modifying, you **create new instances** with changes:

```dart
// ❌ Mutable approach (not used in this project)
class MutableCart {
  Map<String, int> items = {};

  void addItem(String id, int qty) {
    items[id] = qty;  // Modifies in place
  }
}

// ✅ Immutable approach (this project)
class Cart {
  const Cart([this.items = const {}]);
  final Map<String, int> items;

  // Returns NEW cart with item added
  Cart addItem(Item item) {
    final copy = Map<String, int>.from(items);
    copy[item.productId] = item.quantity;
    return Cart(copy);  // New instance!
  }
}
```

---

## Extension Methods for Business Logic

Extension methods add functionality to existing classes **without modifying them**.

### Why Extensions for Domain Logic?

1. **Keep entity classes simple** - Just data
2. **Organize logic** - Group related operations
3. **Maintain immutability** - Return new instances

### Example: Cart Extensions

**File:** [lib/src/features/cart/domain/cart.dart](../lib/src/features/cart/domain/cart.dart)

```dart
/// Extension methods for cart mutations
extension MutableCart on Cart {
  /// Sets item to exact quantity (replaces existing)
  Cart setItem(Item item) {
    final copy = Map<ProductID, int>.from(items);
    copy[item.productId] = item.quantity;
    return Cart(copy);
  }

  /// Adds to existing quantity (or creates new entry)
  Cart addItem(Item item) {
    final copy = Map<ProductID, int>.from(items);
    copy.update(
      item.productId,
      (existingQty) => item.quantity + existingQty,
      ifAbsent: () => item.quantity,
    );
    return Cart(copy);
  }

  /// Adds multiple items at once
  Cart addItems(List<Item> itemsToAdd) {
    var result = this;
    for (final item in itemsToAdd) {
      result = result.addItem(item);
    }
    return result;
  }

  /// Removes item completely from cart
  Cart removeItemById(ProductID productId) {
    final copy = Map<ProductID, int>.from(items);
    copy.remove(productId);
    return Cart(copy);
  }
}
```

**Usage:**

```dart
// Start with empty cart
final cart = Cart();

// Add item (returns new cart)
final cart2 = cart.addItem(Item(productId: 'product-1', quantity: 2));

// Add another item (returns new cart)
final cart3 = cart2.addItem(Item(productId: 'product-2', quantity: 1));

// Remove item (returns new cart)
final cart4 = cart3.removeItemById('product-1');

// Original cart unchanged!
print(cart.items);  // {}
print(cart4.items);  // {'product-2': 1}
```

**Pattern Benefits:**
- ✅ Clear API (`addItem`, `removeItem`)
- ✅ Chainable operations
- ✅ Original never modified
- ✅ Easy to test

### Example: Address Extensions

**File:** [lib/src/features/address/domain/address.dart](../lib/src/features/address/domain/address.dart)

```dart
class Address {
  const Address({
    required this.addressLine1,
    this.addressLine2,
    required this.city,
    required this.zip,
    required this.country,
  });

  final String addressLine1;
  final String? addressLine2;
  final String city;
  final String zip;
  final String country;

  // Serialization in extension
  Map<String, dynamic> toJson() => {
        'addressLine1': addressLine1,
        'addressLine2': addressLine2,
        'city': city,
        'zip': zip,
        'country': country,
      };
}

extension AddressX on Address {
  factory AddressX.fromJson(Map<String, dynamic> json) {
    return Address(
      addressLine1: json['addressLine1'] as String,
      addressLine2: json['addressLine2'] as String?,
      city: json['city'] as String,
      zip: json['zip'] as String,
      country: json['country'] as String,
    );
  }
}
```

---

## Type Aliases for Type Safety

Type aliases make code more **readable** and **type-safe**.

### Example: Product ID

```dart
// Instead of:
String productId = '123';
void fetchProduct(String id) { }

// Use type alias:
typedef ProductID = String;

ProductID productId = '123';
void fetchProduct(ProductID id) { }
```

**Benefits:**

1. **Self-documenting**: `ProductID` is clearer than `String`
2. **Type safety**: Can't accidentally pass wrong type of string
3. **Refactoring**: Change type in one place

### Example: Multiple ID Types

```dart
typedef ProductID = String;
typedef UserID = String;
typedef OrderID = String;

// Compiler catches mistakes
void processOrder(OrderID orderId, ProductID productId) {
  // ...
}

// This would fail at compile time:
// processOrder(productId, orderId);  // Types are swapped!
```

**Used Throughout Project:**

**File:** [lib/src/features/products/domain/product.dart](../lib/src/features/products/domain/product.dart)
```dart
typedef ProductID = String;
```

**File:** [lib/src/features/cart/domain/item.dart](../lib/src/features/cart/domain/item.dart)
```dart
// Reuses ProductID from products domain
final ProductID productId;
```

---

## Data Layer Deep Dive

The data layer **abstracts data access** from the rest of the app.

### Responsibilities

✅ Fetch data from sources (API, database, cache)
✅ Store data (local database, remote server)
✅ Convert between data formats (JSON ↔ Domain entities)
✅ Handle data errors
✅ Provide streams for reactive updates

### Organization

```
features/[feature]/data/
├── local/                    # Local data sources
│   └── sembast_cart_repository.dart
├── remote/                   # Remote data sources
│   └── remote_cart_repository.dart
└── fake_products_repository.dart  # Fake implementation
```

---

## Repository Pattern

The **Repository Pattern** abstracts the data source behind an interface.

### Abstract Interface (Contract)

```dart
/// Interface defining cart data operations
abstract class LocalCartRepository {
  Future<Cart> fetchCart();
  Stream<Cart> watchCart();
  Future<void> setCart(Cart cart);
}
```

**Benefits:**
- ✅ Rest of app doesn't know about implementation
- ✅ Easy to swap implementations
- ✅ Easy to test with mocks

### Implementation Example

**File:** [lib/src/features/cart/data/local/sembast_cart_repository.dart](../lib/src/features/cart/data/local/sembast_cart_repository.dart)

```dart
/// Sembast implementation of LocalCartRepository
class SembastCartRepository implements LocalCartRepository {
  SembastCartRepository(this.db);

  final Database db;
  final store = StoreRef.main();
  static const cartItemsKey = 'cartItems';

  @override
  Future<Cart> fetchCart() async {
    final cartJson = await store.record(cartItemsKey).get(db) as String?;
    if (cartJson != null) {
      return Cart.fromJson(json.decode(cartJson));
    }
    return const Cart();
  }

  @override
  Stream<Cart> watchCart() {
    return store.record(cartItemsKey).onSnapshot(db).map((snapshot) {
      if (snapshot != null && snapshot.value != null) {
        return Cart.fromJson(json.decode(snapshot.value as String));
      } else {
        return const Cart();
      }
    });
  }

  @override
  Future<void> setCart(Cart cart) async {
    return store.record(cartItemsKey).put(db, json.encode(cart.toJson()));
  }

  /// Factory to create with default database
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

1. **Implements interface**: Must provide all methods
2. **Database instance**: Passed via constructor (DI)
3. **JSON serialization**: Converts Cart ↔ JSON string
4. **Stream support**: `onSnapshot` for reactive updates
5. **Factory method**: Async initialization pattern

---

## Local Storage with Sembast

Sembast is a **NoSQL database** for Flutter - simple and powerful.

### Why Sembast?

✅ **Works everywhere**: iOS, Android, Web, Desktop
✅ **NoSQL**: Store JSON documents
✅ **Reactive**: Streams for data changes
✅ **Lightweight**: No native dependencies
✅ **Type-safe**: Strongly typed API

### Sembast Basics

```dart
// 1. Open database
final db = await databaseFactoryIo.openDatabase('my_database.db');

// 2. Create a store reference
final store = StoreRef.main();

// 3. Write data
await store.record('key').put(db, 'value');

// 4. Read data
final value = await store.record('key').get(db);

// 5. Watch data (stream)
store.record('key').onSnapshot(db).listen((snapshot) {
  print('Value changed: ${snapshot?.value}');
});

// 6. Delete data
await store.record('key').delete(db);
```

### Complete Cart Repository Implementation

Let's trace the full implementation:

**1. Database Creation**

```dart
static Future<Database> createDatabase() async {
  // Get app documents directory
  final appDir = await getApplicationDocumentsDirectory();

  // Create/open database file
  return databaseFactoryIo.openDatabase('${appDir.path}/default.db');
}
```

**2. Fetch Cart**

```dart
Future<Cart> fetchCart() async {
  // Get JSON string from database
  final cartJson = await store.record(cartItemsKey).get(db) as String?;

  if (cartJson != null) {
    // Deserialize JSON → Cart
    return Cart.fromJson(json.decode(cartJson));
  }

  // Default: empty cart
  return const Cart();
}
```

**3. Save Cart**

```dart
Future<void> setCart(Cart cart) async {
  // Serialize Cart → JSON
  final cartJson = json.encode(cart.toJson());

  // Save to database
  await store.record(cartItemsKey).put(db, cartJson);
}
```

**4. Watch Cart (Stream)**

```dart
Stream<Cart> watchCart() {
  return store.record(cartItemsKey).onSnapshot(db).map((snapshot) {
    if (snapshot != null && snapshot.value != null) {
      // Parse JSON from snapshot
      return Cart.fromJson(json.decode(snapshot.value as String));
    } else {
      return const Cart();
    }
  });
}
```

**Data Flow:**

```
App Layer
   ↓ setCart(cart)
Repository
   ↓ cart.toJson()
Domain Entity → Map
   ↓ json.encode()
Map → JSON String
   ↓ store.record().put()
Sembast Database
   ↓ onSnapshot()
Stream Emits
   ↓ json.decode()
JSON String → Map
   ↓ Cart.fromJson()
Map → Domain Entity
   ↓
UI Updates
```

### Testing with In-Memory Database

```dart
// For tests, use in-memory database (doesn't persist)
final db = await databaseFactoryMemory.openDatabase('test.db');
final repository = SembastCartRepository(db);

// Test operations
await repository.setCart(testCart);
final cart = await repository.fetchCart();
expect(cart, testCart);
```

---

## In-Memory Store Pattern

For **fake repositories**, the project uses an in-memory reactive store.

### InMemoryStore Implementation

**File:** [lib/src/utils/in_memory_store.dart](../lib/src/utils/in_memory_store.dart)

```dart
/// An in-memory store backed by RxDart's BehaviorSubject
class InMemoryStore<T> {
  InMemoryStore(T initial) : _subject = BehaviorSubject<T>.seeded(initial);

  final BehaviorSubject<T> _subject;

  /// Stream of value changes
  Stream<T> get stream => _subject.stream;

  /// Current value
  T get value => _subject.value;

  /// Update value (notifies all listeners)
  set value(T value) => _subject.add(value);

  /// Cleanup
  void close() => _subject.close();
}
```

**How it Works:**

1. **BehaviorSubject**: RxDart stream that remembers last value
2. **Seeded**: Initialized with a value
3. **Reactive**: Emitting new value notifies all listeners
4. **Synchronous read**: Can get current value without async

**Usage Example:**

```dart
// Create store with initial value
final store = InMemoryStore<List<Product>>(initialProducts);

// Read current value
final products = store.value;

// Update value (all listeners notified)
store.value = newProducts;

// Listen to changes
store.stream.listen((products) {
  print('Products changed: ${products.length}');
});

// Cleanup when done
store.close();
```

---

## Fake Repositories for Development

This project uses **fake repositories** to simulate backend behavior without a real API.

### Benefits

✅ **No backend needed**: Develop UI independently
✅ **Instant responses**: No network delay
✅ **Predictable data**: Controlled test data
✅ **Easy to switch**: Replace with real implementation later

### Example: Fake Products Repository

**File:** [lib/src/features/products/data/fake_products_repository.dart](../lib/src/features/products/data/fake_products_repository.dart)

```dart
class FakeProductsRepository {
  FakeProductsRepository({this.addDelay = true});

  final bool addDelay;

  // In-memory store with test products
  final _products = InMemoryStore<List<Product>>(
    List.from(kTestProducts),
  );

  /// Get all products synchronously
  List<Product> getProductsList() {
    return _products.value;
  }

  /// Get single product
  Product? getProduct(String id) {
    return _getProduct(_products.value, id);
  }

  /// Fetch products (async with optional delay)
  Future<List<Product>> fetchProductsList() async {
    if (addDelay) {
      await Future.delayed(const Duration(seconds: 1));
    }
    return Future.value(_products.value);
  }

  /// Watch products as stream
  Stream<List<Product>> watchProductsList() {
    return _products.stream;
  }

  /// Watch single product
  Stream<Product?> watchProduct(String id) {
    return watchProductsList().map((products) => _getProduct(products, id));
  }

  /// Search products
  Future<List<Product>> searchProducts(String query) async {
    final lowercaseQuery = query.toLowerCase();
    final productsList = await fetchProductsList();

    return productsList
        .where((product) =>
            product.title.toLowerCase().contains(lowercaseQuery))
        .toList();
  }

  static Product? _getProduct(List<Product> products, String id) {
    try {
      return products.firstWhere((product) => product.id == id);
    } catch (e) {
      return null;
    }
  }

  void dispose() => _products.close();
}
```

**Test Products:**

**File:** [lib/src/constants/test_products.dart](../lib/src/constants/test_products.dart)

```dart
const kTestProducts = [
  Product(
    id: '1',
    imageUrl: 'assets/products/bruschetta-plate.jpg',
    title: 'Bruschetta',
    description: 'A delicious appetizer',
    price: 15.00,
    availableQuantity: 100,
    avgRating: 4.5,
    numRatings: 23,
  ),
  Product(
    id: '2',
    imageUrl: 'assets/products/salmon.jpg',
    title: 'Salmon',
    description: 'Fresh grilled salmon',
    price: 25.00,
    availableQuantity: 50,
    avgRating: 4.8,
    numRatings: 45,
  ),
  // ... 12 more products
];
```

### Example: Fake Auth Repository

**File:** [lib/src/features/authentication/data/fake_auth_repository.dart](../lib/src/features/authentication/data/fake_auth_repository.dart)

```dart
class FakeAuthRepository {
  final _authState = InMemoryStore<AppUser?>(null);

  Stream<AppUser?> authStateChanges() => _authState.stream;
  AppUser? get currentUser => _authState.value;

  Future<void> signInWithEmailAndPassword(String email, String password) async {
    await Future.delayed(const Duration(seconds: 1));

    if (password == 'wrong') {
      throw WrongPasswordException();
    }

    // Create fake user
    _authState.value = AppUser(uid: email.split('@')[0], email: email);
  }

  Future<void> createUserWithEmailAndPassword(
    String email,
    String password,
  ) async {
    await Future.delayed(const Duration(seconds: 1));

    if (_authState.value != null) {
      throw EmailAlreadyInUseException();
    }

    // Create fake user
    _authState.value = AppUser(uid: email.split('@')[0], email: email);
  }

  Future<void> signOut() async {
    _authState.value = null;
  }

  void dispose() => _authState.close();
}
```

**Simulates:**
- ✅ Network delay
- ✅ Auth errors (wrong password, email in use)
- ✅ Auth state stream
- ✅ Sign in/out behavior

---

## Data Serialization

Converting between domain entities and JSON.

### Pattern 1: toJson / fromJson

```dart
class Order {
  const Order({
    required this.id,
    required this.userId,
    required this.items,
    required this.orderStatus,
    required this.orderDate,
    required this.total,
  });

  final String id;
  final String userId;
  final Map<ProductID, int> items;
  final OrderStatus orderStatus;
  final DateTime orderDate;
  final double total;

  // Serialize to JSON
  Map<String, dynamic> toJson() => {
        'id': id,
        'userId': userId,
        'items': items,
        'orderStatus': orderStatus.name,  // Enum to string
        'orderDate': orderDate.toIso8601String(),  // DateTime to string
        'total': total,
      };

  // Deserialize from JSON
  factory Order.fromJson(Map<String, dynamic> json) {
    return Order(
      id: json['id'] as String,
      userId: json['userId'] as String,
      items: Map<ProductID, int>.from(json['items'] as Map),
      orderStatus: OrderStatus.values.byName(json['orderStatus'] as String),
      orderDate: DateTime.parse(json['orderDate'] as String),
      total: (json['total'] as num).toDouble(),
    );
  }
}
```

**Type Conversions:**

| Dart Type | JSON Type | Conversion |
|-----------|-----------|------------|
| `String` | `string` | Direct |
| `int` | `number` | `as int` |
| `double` | `number` | `.toDouble()` |
| `bool` | `boolean` | Direct |
| `DateTime` | `string` | `toIso8601String()` / `DateTime.parse()` |
| `Enum` | `string` | `.name` / `.byName()` |
| `Map` | `object` | `Map.from()` |
| `List` | `array` | `List.from()` |

### Pattern 2: Nested Serialization

```dart
class OrderWithProducts {
  final Order order;
  final List<Product> products;

  Map<String, dynamic> toJson() => {
        'order': order.toJson(),  // Nested object
        'products': products.map((p) => p.toJson()).toList(),  // List of objects
      };

  factory OrderWithProducts.fromJson(Map<String, dynamic> json) {
    return OrderWithProducts(
      order: Order.fromJson(json['order']),
      products: (json['products'] as List)
          .map((p) => Product.fromJson(p))
          .toList(),
    );
  }
}
```

### Handling Nulls

```dart
class Address {
  final String addressLine1;
  final String? addressLine2;  // Nullable

  Map<String, dynamic> toJson() => {
        'addressLine1': addressLine1,
        'addressLine2': addressLine2,  // Can be null
      };

  factory Address.fromJson(Map<String, dynamic> json) {
    return Address(
      addressLine1: json['addressLine1'] as String,
      addressLine2: json['addressLine2'] as String?,  // Nullable cast
    );
  }
}
```

---

## Testing Domain and Data Layers

Domain and data layers are **highly testable** because they have minimal dependencies.

### Testing Domain Entities

```dart
import 'package:test/test.dart';

void main() {
  group('Cart', () {
    test('empty cart has zero items', () {
      const cart = Cart();
      expect(cart.items, isEmpty);
      expect(cart.itemsCount, 0);
    });

    test('addItem adds new item', () {
      const cart = Cart();
      final updated = cart.addItem(Item(productId: 'p1', quantity: 2));

      expect(updated.items['p1'], 2);
      expect(cart.items, isEmpty);  // Original unchanged
    });

    test('addItem increases existing quantity', () {
      final cart = Cart({'p1': 2});
      final updated = cart.addItem(Item(productId: 'p1', quantity: 3));

      expect(updated.items['p1'], 5);
    });

    test('removeItemById removes item', () {
      final cart = Cart({'p1': 2, 'p2': 1});
      final updated = cart.removeItemById('p1');

      expect(updated.items, {'p2': 1});
    });

    test('serialization round-trip', () {
      final cart = Cart({'p1': 2, 'p2': 3});
      final json = cart.toJson();
      final deserialized = Cart.fromJson(json);

      expect(deserialized, cart);
    });
  });
}
```

### Testing Repositories

**File:** [test/src/features/cart/data/local/sembast_cart_repository_test.dart](../test/src/features/cart/data/local/sembast_cart_repository_test.dart)

```dart
void main() {
  late Database db;
  late SembastCartRepository repository;

  setUp(() async {
    // Use in-memory database for tests
    db = await databaseFactoryMemory.openDatabase('test.db');
    repository = SembastCartRepository(db);
  });

  tearDown(() async {
    await db.close();
  });

  test('fetchCart returns empty cart by default', () async {
    final cart = await repository.fetchCart();
    expect(cart, const Cart());
  });

  test('setCart persists cart', () async {
    final cart = Cart({'p1': 2});

    await repository.setCart(cart);
    final fetched = await repository.fetchCart();

    expect(fetched, cart);
  });

  test('watchCart emits updates', () async {
    final cart1 = Cart({'p1': 1});
    final cart2 = Cart({'p1': 2});

    // Listen to stream
    final stream = repository.watchCart();
    expectLater(
      stream,
      emitsInOrder([
        const Cart(),  // Initial
        cart1,
        cart2,
      ]),
    );

    // Make changes
    await repository.setCart(cart1);
    await repository.setCart(cart2);
  });
}
```

---

## Best Practices

### Domain Layer

✅ **Keep entities pure** - No Flutter imports
✅ **Use const constructors** - Enable compile-time constants
✅ **Make fields final** - Enforce immutability
✅ **Implement equality** - Based on business key
✅ **Add toString** - Helpful for debugging
✅ **Use type aliases** - `ProductID` instead of `String`
✅ **Computed properties** - Derive from existing fields
✅ **Extension methods** - Keep entity class simple

### Data Layer

✅ **Abstract interfaces** - Define contracts
✅ **Dependency injection** - Pass dependencies via constructor
✅ **Handle nulls gracefully** - Return empty/default values
✅ **Provide streams** - For reactive updates
✅ **Separate concerns** - Local vs remote repositories
✅ **Test with mocks** - Easy to test with fake implementations
✅ **Proper serialization** - Handle all edge cases

### ❌ Common Mistakes

❌ **Mutable domain entities** - Leads to bugs
❌ **Business logic in repositories** - Should be in services
❌ **Tight coupling** - Repository shouldn't know about UI
❌ **Forgetting to close streams** - Memory leaks
❌ **Poor error handling** - Always handle errors gracefully

---

## Summary

The **Domain and Data layers** are the foundation:

**Domain Layer:**
- ✅ Pure Dart entities
- ✅ Immutable by design
- ✅ Extension methods for business logic
- ✅ Type-safe with aliases
- ✅ Easy to test

**Data Layer:**
- ✅ Repository pattern
- ✅ Sembast for local storage
- ✅ InMemoryStore for fake data
- ✅ Reactive with streams
- ✅ Clean separation

These layers create a **rock-solid foundation** that the application and presentation layers build upon.

---

## What's Next?

Now that you understand the foundation, let's explore the **Application and Presentation layers**.

Head to **[06-APPLICATION-AND-PRESENTATION-LAYERS.md](06-APPLICATION-AND-PRESENTATION-LAYERS.md)** to learn:
- Service layer patterns
- Business logic orchestration
- Controller patterns
- UI composition
- State management in practice
- And much more!

Ready to see how the upper layers work? Let's continue! 🚀
