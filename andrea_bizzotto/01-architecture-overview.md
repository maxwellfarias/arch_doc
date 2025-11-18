# Architecture Overview

## Table of Contents
- [Introduction](#introduction)
- [Tech Stack](#tech-stack)
- [Architectural Pattern](#architectural-pattern)
- [Project Structure](#project-structure)
- [Layer Responsibilities](#layer-responsibilities)
- [Why This Architecture?](#why-this-architecture)
- [Data Flow Visualization](#data-flow-visualization)

---

## Introduction

Welcome to the **Flutter E-Commerce Application** documentation! This project is a comprehensive, production-ready mobile application that demonstrates best practices in Flutter development. Whether you're a beginner learning Flutter or an experienced developer looking for architectural patterns, this project serves as an excellent reference.

### What is This App?

This is a **full-featured e-commerce application** that allows users to:
- Browse a catalog of products
- Search and filter products
- Add items to a shopping cart
- Complete purchases through checkout
- Review their order history
- Leave reviews and ratings for products
- Manage their account and authentication

### Learning Objectives

By studying this project, you'll learn:
- How to structure a large-scale Flutter application
- Clean Architecture principles in practice
- Modern state management with Riverpod 2.x
- Declarative navigation with GoRouter
- Testing strategies (unit, widget, integration tests)
- Real-world design patterns in Flutter

---

## Tech Stack

### Core Technologies

| Technology | Version | Purpose |
|------------|---------|---------|
| **Flutter** | SDK | Cross-platform UI framework |
| **Dart** | Latest | Programming language |
| **Riverpod** | 2.6.1 | State management & dependency injection |
| **GoRouter** | 15.0.0 | Declarative routing and navigation |
| **Sembast** | 3.8.3 | NoSQL local database (for offline cart) |
| **RxDart** | 0.28.0 | Reactive programming utilities |

### Development Tools

| Tool | Version | Purpose |
|------|---------|---------|
| **riverpod_generator** | 2.6.5 | Code generation for providers |
| **build_runner** | 2.4.15 | Runs code generators |
| **mocktail** | 1.0.4 | Mocking framework for tests |
| **flutter_test** | SDK | Testing framework |

### Why These Choices?

**Riverpod** was chosen for:
- Type-safe dependency injection
- Compile-time safety with code generation
- Excellent testability
- Great developer experience

**GoRouter** was chosen for:
- Declarative routing (easier to understand and maintain)
- Deep linking support
- Route guards for authentication
- Type-safe navigation

**Sembast** was chosen for:
- Simple NoSQL database for local storage
- No native dependencies
- Works on all platforms (mobile, web, desktop)
- Perfect for storing shopping cart locally

---

## Architectural Pattern

This project uses a **Feature-First Clean Architecture** with Riverpod as the state management solution.

### What is Clean Architecture?

Think of Clean Architecture like organizing a restaurant:

```
┌─────────────────────────────────────────────────────────────┐
│                         RESTAURANT                          │
├─────────────────────────────────────────────────────────────┤
│  Front of House (Presentation Layer)                       │
│  - Waiters interact with customers                         │
│  - Take orders, serve food, handle feedback                │
├─────────────────────────────────────────────────────────────┤
│  Kitchen Manager (Application Layer)                       │
│  - Coordinates between waiters and cooks                   │
│  - Orchestrates complex workflows                          │
├─────────────────────────────────────────────────────────────┤
│  Cooks & Recipes (Domain Layer)                            │
│  - Core business logic and rules                           │
│  - How to make dishes (entities & business rules)          │
├─────────────────────────────────────────────────────────────┤
│  Pantry & Suppliers (Data Layer)                           │
│  - Where ingredients come from                             │
│  - Storage and external data sources                       │
└─────────────────────────────────────────────────────────────┘
```

### The Four Layers

```
┌──────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                        │
│  (UI, Screens, Widgets, Controllers)                        │
│  - What the user sees and interacts with                    │
│  - Flutter widgets, screens, buttons                        │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ↓
┌──────────────────────────────────────────────────────────────┐
│                   APPLICATION LAYER                          │
│  (Services, Use Cases, Business Logic Orchestration)        │
│  - Coordinates operations between layers                    │
│  - Implements complex workflows                             │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ↓
┌──────────────────────────────────────────────────────────────┐
│                     DOMAIN LAYER                             │
│  (Entities, Business Rules, Domain Logic)                   │
│  - Core business logic independent of framework             │
│  - Pure Dart classes (Product, Cart, Order)                 │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ↓
┌──────────────────────────────────────────────────────────────┐
│                      DATA LAYER                              │
│  (Repositories, Data Sources, External APIs)                │
│  - Handles data persistence and retrieval                   │
│  - Databases, APIs, file storage                            │
└──────────────────────────────────────────────────────────────┘
```

### Feature-First Organization

Instead of organizing by layer type, we organize by **business features**:

```
❌ BAD (Layer-First):
lib/
├── models/           # All models together
├── views/            # All views together
├── controllers/      # All controllers together
└── services/         # All services together

✅ GOOD (Feature-First):
lib/src/features/
├── authentication/   # Everything for auth in one place
│   ├── domain/
│   ├── data/
│   ├── application/
│   └── presentation/
├── products/         # Everything for products in one place
│   ├── domain/
│   ├── data/
│   └── presentation/
└── cart/             # Everything for cart in one place
    ├── domain/
    ├── data/
    ├── application/
    └── presentation/
```

**Why Feature-First?**
- Easy to find all related code
- Features can be worked on independently
- Easy to delete a feature (just delete the folder)
- Teams can work on different features without conflicts
- New developers can focus on one feature at a time

---

## Project Structure

### High-Level Directory Tree

```
ecommerce_app/
│
├── lib/                          # Application source code
│   ├── main.dart                 # App entry point
│   └── src/
│       ├── app.dart              # Root widget (MyApp)
│       ├── constants/            # App-wide constants
│       ├── exceptions/           # Custom exceptions
│       ├── utils/                # Helper utilities
│       ├── localization/         # i18n support
│       ├── routing/              # GoRouter configuration
│       ├── common_widgets/       # Reusable UI components
│       └── features/             # Feature modules
│           ├── authentication/
│           ├── products/
│           ├── cart/
│           ├── checkout/
│           ├── orders/
│           ├── reviews/
│           └── address/
│
├── test/                         # Unit & widget tests
│   └── src/
│       ├── features/             # Tests mirroring lib structure
│       ├── goldens/              # Golden image tests
│       ├── mocks.dart            # Mock objects
│       └── robot.dart            # Test robot pattern
│
├── integration_test/             # End-to-end tests
│
├── assets/                       # Static assets
│   ├── fonts/                    # Custom fonts
│   └── products/                 # Product images
│
├── android/                      # Android platform code
├── ios/                          # iOS platform code
├── web/                          # Web platform code
├── macos/                        # macOS platform code
├── windows/                      # Windows platform code
│
└── pubspec.yaml                  # Dependencies & configuration
```

### Feature Module Structure

Each feature follows a consistent internal structure:

```
feature_name/
│
├── domain/                       # Business entities & logic
│   ├── entity.dart               # Core business object
│   └── entity_extensions.dart   # Business logic methods
│
├── data/                         # Data sources & repositories
│   ├── repository_interface.dart
│   └── repository_impl.dart
│
├── application/                  # Business logic orchestration
│   ├── service.dart              # Use cases & workflows
│   └── providers.dart            # Riverpod providers
│
└── presentation/                 # UI layer
    ├── screen_name/
    │   ├── screen.dart           # UI layout
    │   ├── controller.dart       # State management
    │   └── widgets/              # Screen-specific widgets
    │       ├── widget_1.dart
    │       └── widget_2.dart
    └── another_screen/
        └── ...
```

### Example: Cart Feature Structure

Let's look at a real example from the cart feature:

```
lib/src/features/cart/
│
├── domain/
│   ├── cart.dart                 # Cart entity (immutable)
│   ├── item.dart                 # Cart item entity
│   └── mutable_cart.dart         # Cart business logic (extensions)
│
├── data/
│   ├── local/
│   │   ├── local_cart_repository.dart      # Abstract interface
│   │   └── sembast_cart_repository.dart    # Sembast implementation
│   └── remote/
│       ├── remote_cart_repository.dart     # Abstract interface
│       └── fake_remote_cart_repository.dart # Fake implementation
│
├── application/
│   ├── cart_service.dart         # Cart business logic
│   └── cart_sync_service.dart    # Sync local/remote carts
│
└── presentation/
    ├── add_to_cart/
    │   ├── add_to_cart_widget.dart
    │   └── add_to_cart_controller.dart
    ├── shopping_cart/
    │   ├── shopping_cart_screen.dart
    │   ├── shopping_cart_item.dart
    │   └── shopping_cart_item_controller.dart
    └── cart_total/
        └── cart_total_text.dart
```

**File References:**
- Domain entities: [lib/src/features/cart/domain/cart.dart](lib/src/features/cart/domain/cart.dart)
- Repository interface: [lib/src/features/cart/data/local/local_cart_repository.dart](lib/src/features/cart/data/local/local_cart_repository.dart)
- Business logic: [lib/src/features/cart/application/cart_service.dart](lib/src/features/cart/application/cart_service.dart)
- UI: [lib/src/features/cart/presentation/shopping_cart/shopping_cart_screen.dart](lib/src/features/cart/presentation/shopping_cart/shopping_cart_screen.dart)

---

## Layer Responsibilities

### 1. Presentation Layer (UI)

**Responsibilities:**
- Display data to users
- Capture user input
- Handle UI state
- Navigate between screens
- Show loading/error states

**What it contains:**
- Screens (full-page views)
- Widgets (reusable UI components)
- Controllers (UI state management with Riverpod)

**What it DOESN'T do:**
- ❌ Direct database access
- ❌ Business logic
- ❌ Data formatting (that's domain logic)
- ❌ API calls directly

**Example:** `shopping_cart_screen.dart`
```dart
// File: lib/src/features/cart/presentation/shopping_cart/shopping_cart_screen.dart

class ShoppingCartScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Watch the cart state from a provider
    final cartValue = ref.watch(cartProvider);

    return AsyncValueWidget(
      value: cartValue,
      data: (cart) => /* Build UI with cart data */
    );
  }
}
```

### 2. Application Layer (Use Cases)

**Responsibilities:**
- Orchestrate business workflows
- Coordinate between presentation and data layers
- Implement use cases (e.g., "Add item to cart")
- Handle complex business operations

**What it contains:**
- Services (business logic coordination)
- Complex workflows
- Use case implementations

**What it DOESN'T do:**
- ❌ UI rendering
- ❌ Direct data storage (uses repositories)
- ❌ Define entities (that's domain layer)

**Example:** `cart_service.dart`
```dart
// File: lib/src/features/cart/application/cart_service.dart:15-30

class CartService {
  CartService(this.ref);
  final Ref ref;

  // Use case: Add item to cart
  Future<void> addItem(Item item) async {
    // Get current cart
    final cart = await _fetchCart();
    // Apply business logic
    final updatedCart = cart.addItem(item);
    // Persist changes
    await _setCart(updatedCart);
  }

  // Helper: Fetch cart (handles local vs remote)
  Future<Cart> _fetchCart() async {
    final user = ref.read(authStateChangesProvider).value;
    if (user != null) {
      return ref.read(remoteCartRepositoryProvider).fetchCart(user.uid);
    } else {
      return ref.read(localCartRepositoryProvider).fetchCart();
    }
  }
}
```

### 3. Domain Layer (Business Logic)

**Responsibilities:**
- Define core business entities
- Implement business rules
- Pure business logic (no dependencies on Flutter/UI)
- Immutable data structures

**What it contains:**
- Entities (Product, Cart, Order, User)
- Value objects (ProductID, OrderID)
- Business logic methods (extension methods)
- Domain exceptions

**What it DOESN'T do:**
- ❌ UI code
- ❌ Database queries
- ❌ API calls
- ❌ Navigation

**Example:** `cart.dart` and `mutable_cart.dart`
```dart
// File: lib/src/features/cart/domain/cart.dart:7-15

@immutable
class Cart {
  const Cart([this.items = const {}]);

  // Items stored as a map: productId -> Item
  final Map<String, Item> items;

  // Other methods...
}
```

```dart
// File: lib/src/features/cart/domain/mutable_cart.dart:5-20

// Extension methods add business logic to Cart entity
extension MutableCart on Cart {
  // Business rule: Adding an item increases quantity if exists
  Cart addItem(Item item) {
    final copy = Map<String, Item>.from(items);
    final currentItem = copy[item.productId];

    if (currentItem != null) {
      // Item exists: increase quantity
      copy[item.productId] = Item(
        productId: item.productId,
        quantity: currentItem.quantity + item.quantity,
      );
    } else {
      // New item: add to cart
      copy[item.productId] = item;
    }

    return Cart(copy);
  }
}
```

### 4. Data Layer (External Data)

**Responsibilities:**
- Fetch data from external sources
- Persist data locally or remotely
- Abstract data source implementation
- Handle data conversion

**What it contains:**
- Repository interfaces (abstract classes)
- Repository implementations (concrete classes)
- Data sources (APIs, databases, local storage)
- Data models (if different from domain entities)

**What it DOESN'T do:**
- ❌ Business logic
- ❌ UI code
- ❌ Navigation

**Example:** Repository interface and implementation
```dart
// File: lib/src/features/cart/data/local/local_cart_repository.dart:6-15

// Abstract interface (contract)
abstract class LocalCartRepository {
  Future<Cart> fetchCart();
  Stream<Cart> watchCart();
  Future<void> setCart(Cart cart);
}
```

```dart
// File: lib/src/features/cart/data/local/sembast_cart_repository.dart:15-40

// Concrete implementation using Sembast database
class SembastCartRepository implements LocalCartRepository {
  SembastCartRepository(this.db);
  final Database db;
  final store = StoreRef.main();

  static const cartItemsKey = 'cartItems';

  @override
  Future<Cart> fetchCart() async {
    final cartItemsJson = await store.record(cartItemsKey).get(db);
    if (cartItemsJson != null) {
      return Cart.fromJson(cartItemsJson as Map<String, dynamic>);
    } else {
      return const Cart();
    }
  }

  @override
  Stream<Cart> watchCart() {
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
    await store.record(cartItemsKey).put(db, cart.toJson());
  }
}
```

---

## Why This Architecture?

### Benefits for Beginners

1. **Separation of Concerns**
   - Each layer has one job
   - Easy to understand what code goes where
   - Reduces cognitive load

2. **Testability**
   - Each layer can be tested independently
   - Easy to mock dependencies
   - Fast unit tests

3. **Maintainability**
   - Changes are isolated to specific layers
   - Easy to find and fix bugs
   - Code is organized logically

4. **Scalability**
   - Easy to add new features
   - Features don't interfere with each other
   - Can grow from small to large apps

5. **Team Collaboration**
   - Different developers can work on different features
   - Clear boundaries reduce merge conflicts
   - Easier code reviews

### Real-World Benefits

**Example Scenario:** Changing Database

Let's say you want to switch from Sembast to SQLite. With this architecture:

```
Step 1: Create new implementation
✓ Create SqliteCartRepository implements LocalCartRepository

Step 2: Update provider
✓ Change one line in the provider

Step 3: Done!
✓ No changes needed in UI
✓ No changes needed in business logic
✓ No changes needed in domain entities
```

**Why it works:**
- The repository interface (`LocalCartRepository`) stays the same
- Only the implementation changes
- All other layers depend on the interface, not the implementation
- This is the **Dependency Inversion Principle** in action!

---

## Data Flow Visualization

### User Action to Data and Back

Let's trace what happens when a user taps "Add to Cart":

```
┌─────────────────────────────────────────────────────────────────┐
│  1. USER ACTION                                                 │
│  User taps "Add to Cart" button                                │
└────────────┬────────────────────────────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────────────────────────────┐
│  2. PRESENTATION LAYER                                          │
│  AddToCartController.addItem() is called                       │
│  - Shows loading state                                          │
│  - Calls application layer                                      │
└────────────┬────────────────────────────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────────────────────────────┐
│  3. APPLICATION LAYER                                           │
│  CartService.addItem() orchestrates the operation              │
│  - Fetches current cart                                         │
│  - Applies business logic                                       │
│  - Saves updated cart                                           │
└────────────┬────────────────────────────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────────────────────────────┐
│  4. DOMAIN LAYER                                                │
│  Cart.addItem() applies business rules                         │
│  - Checks if item exists                                        │
│  - Increases quantity or adds new item                         │
│  - Returns new immutable Cart                                   │
└────────────┬────────────────────────────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────────────────────────────┐
│  5. DATA LAYER                                                  │
│  LocalCartRepository.setCart() persists data                   │
│  - Saves to Sembast database                                    │
│  - Emits new cart via Stream                                    │
└────────────┬────────────────────────────────────────────────────┘
             │
             ↓ (Stream emission)
┌─────────────────────────────────────────────────────────────────┐
│  6. BACK TO PRESENTATION                                        │
│  UI rebuilds automatically via Riverpod                        │
│  - Cart icon shows updated count                               │
│  - Success message displayed                                    │
│  - Loading state removed                                        │
└─────────────────────────────────────────────────────────────────┘
```

### Detailed Flow Example

Here's the actual code execution flow:

#### Step 1: User Interaction
```dart
// File: lib/src/features/cart/presentation/add_to_cart/add_to_cart_widget.dart

AddToCartButton(
  productId: product.id,
  onPressed: () async {
    // Call controller
    final controller = ref.read(addToCartControllerProvider.notifier);
    await controller.addItem(productId);
  }
)
```

#### Step 2: Controller (Presentation Layer)
```dart
// File: lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart:20-30

@riverpod
class AddToCartController extends _$AddToCartController {
  @override
  FutureOr<void> build() {
    // Initial state
  }

  Future<void> addItem(ProductID productId) async {
    // Get cart service from application layer
    final cartService = ref.read(cartServiceProvider);

    // Set loading state
    state = const AsyncLoading<void>();

    // Call service and handle errors
    state = await AsyncValue.guard(() =>
      cartService.addItem(Item(productId: productId, quantity: 1))
    );
  }
}
```

#### Step 3: Service (Application Layer)
```dart
// File: lib/src/features/cart/application/cart_service.dart:45-55

class CartService {
  Future<void> addItem(Item item) async {
    // Fetch current cart (handles auth state)
    final cart = await _fetchCart();

    // Apply domain logic
    final updatedCart = cart.addItem(item);  // Domain layer method

    // Persist updated cart
    await _setCart(updatedCart);
  }
}
```

#### Step 4: Domain Logic
```dart
// File: lib/src/features/cart/domain/mutable_cart.dart:5-20

extension MutableCart on Cart {
  Cart addItem(Item item) {
    final copy = Map<String, Item>.from(items);
    // Business logic: merge or add
    copy[item.productId] = /* ... */;
    return Cart(copy);  // Return new immutable cart
  }
}
```

#### Step 5: Repository (Data Layer)
```dart
// File: lib/src/features/cart/data/local/sembast_cart_repository.dart:55-60

class SembastCartRepository implements LocalCartRepository {
  @override
  Future<void> setCart(Cart cart) async {
    // Save to database
    await store.record(cartItemsKey).put(db, cart.toJson());
    // Database automatically emits to stream watchers
  }
}
```

#### Step 6: Reactive UI Update
```dart
// File: lib/src/features/cart/presentation/shopping_cart/shopping_cart_screen.dart

Consumer(
  builder: (context, ref, child) {
    // Automatically rebuilds when cart changes
    final cart = ref.watch(cartProvider);

    return cart.when(
      data: (cart) => Text('Items: ${cart.items.length}'),
      loading: () => CircularProgressIndicator(),
      error: (error, stack) => ErrorWidget(error),
    );
  }
)
```

---

## Key Takeaways

1. **Feature-First Organization**: All related code lives together
2. **Four Clear Layers**: Domain → Data → Application → Presentation
3. **Dependency Flow**: Presentation → Application → Domain ← Data
4. **Separation of Concerns**: Each layer has specific responsibilities
5. **Testability**: Easy to test each layer independently
6. **Scalability**: Easy to add features and make changes

---

## Next Steps

Now that you understand the high-level architecture, dive deeper into:

- **[Layer Breakdown](02-layer-breakdown.md)**: Detailed explanation of each layer
- **[Design Patterns](03-design-patterns.md)**: Patterns used throughout the project
- **[State Management](04-state-management-and-navigation.md)**: Riverpod and GoRouter deep dive
- **[Features Deep-Dive](05-features-deep-dive.md)**: How each feature works
- **[Getting Started Guide](06-getting-started-guide.md)**: Practical guide to working with the code

---

**Questions or Feedback?**
This documentation is designed to help you understand and learn from this project. If you have questions or suggestions for improvement, please open an issue or submit a pull request!
