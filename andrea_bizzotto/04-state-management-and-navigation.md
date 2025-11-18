# State Management and Navigation

## Table of Contents
- [Introduction](#introduction)
- [Riverpod Fundamentals](#riverpod-fundamentals)
- [Provider Types](#provider-types)
- [Code Generation](#code-generation)
- [State Management Patterns](#state-management-patterns)
- [Navigation with GoRouter](#navigation-with-gorouter)
- [Real-World Examples](#real-world-examples)

---

## Introduction

State management and navigation are two critical aspects of any Flutter application. This project uses:

- **Riverpod 2.x** for state management and dependency injection
- **GoRouter 15.x** for declarative routing and navigation

### What is State Management?

**State** is any data that can change over time in your app:
- User is signed in or out
- Items in shopping cart
- Current screen being displayed
- Loading/error states

**State Management** is how you:
- Store this data
- Update it when things change
- Make UI react to changes
- Share data across widgets

**Analogy:** Think of state like a scoreboard at a sports game:
- The scoreboard shows current state (score, time)
- Players' actions update the state
- Everyone can see the changes in real-time
- The scoreboard is in one place, but visible everywhere

---

## Riverpod Fundamentals

### What is Riverpod?

Riverpod is a **reactive state management** library for Flutter. It's a complete rewrite of Provider with:
- Compile-time safety
- Better testing support
- No BuildContext needed
- Code generation support

### Core Concepts

#### 1. Providers

Providers are objects that **provide values** to your widgets.

```dart
// Simple provider
@riverpod
String greeting(Ref ref) {
  return 'Hello, World!';
}

// Usage
final greeting = ref.watch(greetingProvider);
```

#### 2. Ref

`Ref` is how you **interact with other providers**.

```dart
@riverpod
String fullGreeting(Ref ref) {
  // Read another provider
  final greeting = ref.read(greetingProvider);
  return '$greeting from Riverpod!';
}
```

#### 3. Watch vs Read

**Watch**: Rebuild when value changes (reactive)
```dart
// Widget rebuilds when cart changes
final cart = ref.watch(cartProvider);
```

**Read**: Get value once (no rebuilding)
```dart
// Get value once, don't watch for changes
final service = ref.read(cartServiceProvider);
```

**Listen**: React to changes with a callback
```dart
ref.listen<AsyncValue<Cart>>(
  cartProvider,
  (previous, next) {
    // Do something when cart changes
    print('Cart updated!');
  },
);
```

#### 4. AsyncValue

`AsyncValue<T>` represents **async data** with loading/error states.

```dart
AsyncValue<Cart> cartValue = ref.watch(cartProvider);

cartValue.when(
  data: (cart) => Text('${cart.items.length} items'),
  loading: () => CircularProgressIndicator(),
  error: (error, stack) => Text('Error: $error'),
);
```

---

## Provider Types

### 1. Simple Provider

Returns a **constant or computed value**.

**Example:** Products Repository Provider

**File:** [lib/src/features/products/data/fake_products_repository.dart](lib/src/features/products/data/fake_products_repository.dart:140-145)

```dart
@Riverpod(keepAlive: true)
FakeProductsRepository productsRepository(Ref ref) {
  return FakeProductsRepository();
}
```

**Characteristics:**
- `keepAlive: true` = Never disposed (singleton)
- Created once, lives forever
- No parameters

**Usage:**
```dart
final repository = ref.read(productsRepositoryProvider);
```

### 2. Stream Provider

Returns a **Stream** for reactive data.

**Example:** Cart Provider

**File:** [lib/src/features/cart/application/cart_service.dart](lib/src/features/cart/application/cart_service.dart:98-110)

```dart
@Riverpod(keepAlive: true)
Stream<Cart> cart(Ref ref) {
  // Watch auth state
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

**Characteristics:**
- Returns `Stream<T>`
- Emits values over time
- UI automatically updates on new values

**Usage:**
```dart
// Watch as AsyncValue
final cartValue = ref.watch(cartProvider);

// Or get Future (waits for first value)
final cart = await ref.read(cartProvider.future);
```

**UI with Stream Provider:**
```dart
Widget build(BuildContext context, WidgetRef ref) {
  final cartValue = ref.watch(cartProvider);

  return cartValue.when(
    data: (cart) => Text('${cart.items.length} items'),
    loading: () => CircularProgressIndicator(),
    error: (error, _) => Text('Error: $error'),
  );
}
```

### 3. Future Provider

Returns a **Future** for async operations.

**Example:** Products List Provider

**File:** [lib/src/features/products/data/fake_products_repository.dart](lib/src/features/products/data/fake_products_repository.dart:147-155)

```dart
@Riverpod(keepAlive: true)
Stream<List<Product>> productsListStream(Ref ref) {
  final repository = ref.watch(productsRepositoryProvider);
  return repository.watchProductsList();
}

@riverpod
Future<List<Product>> productsListFuture(Ref ref) {
  final repository = ref.watch(productsRepositoryProvider);
  return repository.getProductsList();
}
```

**Usage:**
```dart
// Watch (auto-refresh)
final productsValue = ref.watch(productsListFutureProvider);

// Read once
final products = await ref.read(productsListFutureProvider.future);
```

### 4. Family Provider (Parameterized)

Providers that take **parameters**.

**Example:** Product by ID

**File:** [lib/src/features/products/data/fake_products_repository.dart](lib/src/features/products/data/fake_products_repository.dart:157-162)

```dart
@riverpod
Product? product(Ref ref, ProductID id) {
  final repository = ref.watch(productsRepositoryProvider);
  return repository.getProduct(id);
}
```

**Characteristics:**
- Takes parameters after `Ref`
- Different parameter = different provider instance
- Cached per parameter value

**Usage:**
```dart
// Different IDs = different providers
final product1 = ref.watch(productProvider('1'));
final product2 = ref.watch(productProvider('2'));
```

**Example:** Product Reviews by Product ID

**File:** [lib/src/features/reviews/data/fake_reviews_repository.dart](lib/src/features/reviews/data/fake_reviews_repository.dart:80-88)

```dart
@riverpod
Future<List<Review>> productReviews(Ref ref, ProductID productId) async {
  final user = ref.watch(authStateChangesProvider).value;
  final repository = ref.watch(reviewsRepositoryProvider);

  return repository.fetchReviews(productId);
}
```

**Usage:**
```dart
// Watch reviews for specific product
final reviewsValue = ref.watch(productReviewsProvider('product-123'));
```

### 5. Notifier Provider (Stateful)

For **mutable state** managed by a class.

**Example:** Add to Cart Controller

**File:** [lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart](lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart:15-80)

```dart
@riverpod
class AddToCartController extends _$AddToCartController {
  @override
  FutureOr<void> build() {
    // Initial state
  }

  Future<void> addItem(ProductID productId) async {
    // Get dependencies
    final cartService = ref.read(cartServiceProvider);
    final product = ref.read(productProvider(productId));

    // Validate
    if (product == null) {
      state = AsyncError(Exception('Product not found'), StackTrace.current);
      return;
    }

    // Check stock
    final cart = await ref.read(cartProvider.future);
    final currentQty = cart.items[productId]?.quantity ?? 0;

    if (currentQty >= product.availableQuantity) {
      state = AsyncError(Exception('Out of stock'), StackTrace.current);
      return;
    }

    // Create item
    final item = Item(productId: productId, quantity: 1);

    // Set loading state
    state = const AsyncLoading<void>();

    // Perform action
    state = await AsyncValue.guard(() => cartService.addItem(item));
  }
}
```

**Characteristics:**
- Extends generated `_$ClassName`
- `build()` returns initial state
- Methods update `state` field
- State is `AsyncValue<T>`

**Usage:**
```dart
// Read controller instance
final controller = ref.read(addToCartControllerProvider.notifier);

// Call methods
await controller.addItem('product-id');

// Watch state
final state = ref.watch(addToCartControllerProvider);
```

**UI Integration:**
```dart
Widget build(BuildContext context, WidgetRef ref) {
  final state = ref.watch(addToCartControllerProvider);

  return state.when(
    data: (_) => ElevatedButton(
      onPressed: () {
        ref.read(addToCartControllerProvider.notifier).addItem(productId);
      },
      child: Text('Add to Cart'),
    ),
    loading: () => CircularProgressIndicator(),
    error: (error, _) => Text('Error: $error'),
  );
}
```

---

## Code Generation

### Why Code Generation?

Riverpod 2.x uses **code generation** to:
- Reduce boilerplate
- Improve type safety
- Auto-generate provider code
- Catch errors at compile time

### Setup

**File:** [pubspec.yaml](pubspec.yaml:20-35)

```yaml
dependencies:
  flutter_riverpod: 2.6.1
  riverpod_annotation: 2.6.1

dev_dependencies:
  build_runner: 2.4.15
  riverpod_generator: 2.6.5
  riverpod_lint: 2.6.5
  custom_lint: 0.7.5
```

### Running Code Generation

```bash
# One-time generation
dart run build_runner build --delete-conflicting-outputs

# Watch mode (auto-regenerates)
dart run build_runner watch --delete-conflicting-outputs
```

### Annotation: @riverpod

**Basic Provider:**
```dart
@riverpod
String greeting(Ref ref) {
  return 'Hello!';
}

// Generates:
// final greetingProvider = Provider<String>(...);
```

**Async Provider:**
```dart
@riverpod
Future<List<Product>> products(Ref ref) async {
  return await fetchProducts();
}

// Generates:
// final productsProvider = FutureProvider<List<Product>>(...);
```

**Stream Provider:**
```dart
@riverpod
Stream<Cart> cart(Ref ref) {
  return cartStream();
}

// Generates:
// final cartProvider = StreamProvider<Cart>(...);
```

**Family Provider:**
```dart
@riverpod
Product? product(Ref ref, String id) {
  return getProduct(id);
}

// Generates:
// final productProvider = Provider.family<Product?, String>(...);
```

**Notifier:**
```dart
@riverpod
class Counter extends _$Counter {
  @override
  int build() => 0;

  void increment() => state++;
}

// Generates:
// final counterProvider = NotifierProvider<Counter, int>(...);
```

### Annotation: @Riverpod(keepAlive: true)

**Singleton Provider:**
```dart
@Riverpod(keepAlive: true)
CartService cartService(Ref ref) {
  return CartService(ref);
}

// Never disposed, lives for entire app lifetime
```

### Generated Files

For `cart_service.dart`, generator creates `cart_service.g.dart`:

```dart
// GENERATED CODE - DO NOT MODIFY BY HAND

part of 'cart_service.dart';

// **************************************************************************
// RiverpodGenerator
// **************************************************************************

String _$cartServiceHash() => r'abc123...';

@ProviderFor(cartService)
final cartServiceProvider = Provider<CartService>(
  (ref) => cartService(ref),
  name: r'cartServiceProvider',
  debugGetCreateSourceHash: _$cartServiceHash,
);

typedef CartServiceRef = ProviderRef<CartService>;

String _$cartHash() => r'def456...';

@ProviderFor(cart)
final cartProvider = StreamProvider<Cart>(
  (ref) => cart(ref),
  name: r'cartProvider',
  debugGetCreateSourceHash: _$cartHash,
);

typedef CartRef = StreamProviderRef<Cart>;
```

### Part Directive

**Every file with @riverpod needs:**

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'file_name.g.dart';  // Generated file

@riverpod
String example(Ref ref) {
  return 'Example';
}
```

---

## State Management Patterns

### Pattern 1: Repository → Provider

**Data Layer:**
```dart
// Repository
class FakeProductsRepository {
  Future<List<Product>> getProductsList() async {
    return _products;
  }
}

// Provider
@Riverpod(keepAlive: true)
FakeProductsRepository productsRepository(Ref ref) {
  return FakeProductsRepository();
}
```

**Usage:**
```dart
final repository = ref.read(productsRepositoryProvider);
final products = await repository.getProductsList();
```

### Pattern 2: Service → Provider

**Application Layer:**
```dart
// Service
class CartService {
  CartService(this.ref);
  final Ref ref;

  Future<void> addItem(Item item) async {
    // Business logic
  }
}

// Provider
@Riverpod(keepAlive: true)
CartService cartService(Ref ref) {
  return CartService(ref);
}
```

**Usage:**
```dart
final service = ref.read(cartServiceProvider);
await service.addItem(item);
```

### Pattern 3: Stream → Provider → UI

**Complete Flow:**

```dart
// 1. Repository exposes stream
class LocalCartRepository {
  Stream<Cart> watchCart() {
    return _cartStream;
  }
}

// 2. Provider wraps stream
@Riverpod(keepAlive: true)
Stream<Cart> cart(Ref ref) {
  return ref.watch(localCartRepositoryProvider).watchCart();
}

// 3. UI watches provider
Widget build(BuildContext context, WidgetRef ref) {
  final cartValue = ref.watch(cartProvider);

  return cartValue.when(
    data: (cart) => Text('${cart.items.length} items'),
    loading: () => CircularProgressIndicator(),
    error: (error, _) => Text('Error: $error'),
  );
}
```

**Benefits:**
- Data changes → Stream emits → Provider notifies → UI rebuilds
- Fully reactive
- No manual updates needed

### Pattern 4: Controller for UI State

**Presentation Layer:**

```dart
// 1. Controller manages state
@riverpod
class SignInController extends _$SignInController {
  @override
  FutureOr<void> build() {}

  Future<void> signIn(String email, String password) async {
    state = const AsyncLoading();

    final authRepo = ref.read(authRepositoryProvider);
    state = await AsyncValue.guard(
      () => authRepo.signInWithEmailAndPassword(email, password),
    );
  }
}

// 2. UI uses controller
Widget build(BuildContext context, WidgetRef ref) {
  final state = ref.watch(signInControllerProvider);

  return state.when(
    data: (_) => SignInButton(
      onPressed: () {
        ref.read(signInControllerProvider.notifier)
          .signIn(email, password);
      },
    ),
    loading: () => CircularProgressIndicator(),
    error: (error, _) => ErrorWidget(error),
  );
}
```

### Pattern 5: Combining Multiple Providers

**Example:** Cart Total (needs cart + products)

**File:** [lib/src/features/cart/presentation/cart_total/cart_total_text.dart](lib/src/features/cart/presentation/cart_total/cart_total_text.dart:10-45)

```dart
class CartTotalText extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Watch both providers
    final cartValue = ref.watch(cartProvider);
    final productsValue = ref.watch(productsListStreamProvider);

    // Combine them
    return cartValue.when(
      data: (cart) {
        return productsValue.when(
          data: (products) {
            final total = cart.totalPrice(products);
            return Text('\$${total.toStringAsFixed(2)}');
          },
          loading: () => CircularProgressIndicator(),
          error: (e, _) => Text('Error: $e'),
        );
      },
      loading: () => CircularProgressIndicator(),
      error: (e, _) => Text('Error: $e'),
    );
  }
}
```

**Better approach with helper provider:**

```dart
@riverpod
Future<double> cartTotal(Ref ref) async {
  final cart = await ref.watch(cartProvider.future);
  final products = await ref.watch(productsListStreamProvider.future);
  return cart.totalPrice(products);
}

// UI
Widget build(BuildContext context, WidgetRef ref) {
  final totalValue = ref.watch(cartTotalProvider);

  return totalValue.when(
    data: (total) => Text('\$${total.toStringAsFixed(2)}'),
    loading: () => CircularProgressIndicator(),
    error: (e, _) => Text('Error: $e'),
  );
}
```

### Pattern 6: Invalidating Providers

Force a provider to refresh:

```dart
// After submitting a review
await reviewsRepository.setReview(productId, userId, review);

// Invalidate to refresh
ref.invalidate(productReviewsProvider(productId));

// Next time UI reads it, it will refetch
```

---

## Navigation with GoRouter

### What is GoRouter?

GoRouter is a **declarative routing** package for Flutter that provides:
- Type-safe navigation
- Deep linking
- Route guards (redirects)
- Nested navigation
- URL-based routing

### Setup

**File:** [pubspec.yaml](pubspec.yaml:15-18)

```yaml
dependencies:
  go_router: 15.0.0
```

### Router Configuration

**File:** [lib/src/routing/app_router.dart](lib/src/routing/app_router.dart:15-150)

```dart
@Riverpod(keepAlive: true)
GoRouter goRouter(Ref ref) {
  final authRepository = ref.watch(authRepositoryProvider);

  return GoRouter(
    initialLocation: '/',
    debugLogDiagnostics: true,

    // Redirect logic (route guards)
    redirect: (context, state) {
      final isLoggedIn = authRepository.currentUser != null;
      final path = state.uri.path;

      // Redirect signed-in users away from sign-in page
      if (isLoggedIn && path == '/signIn') {
        return '/';
      }

      // Redirect guests away from protected pages
      if (!isLoggedIn && (path == '/account' || path == '/orders')) {
        return '/';
      }

      return null;  // No redirect
    },

    // Refresh router when auth state changes
    refreshListenable: GoRouterRefreshStream(
      authRepository.authStateChanges(),
    ),

    routes: [
      // Home route
      GoRoute(
        path: '/',
        name: AppRoute.home.name,
        builder: (context, state) => const ProductsListScreen(),

        // Nested routes
        routes: [
          // Product details
          GoRoute(
            path: 'product/:id',
            name: AppRoute.product.name,
            builder: (context, state) {
              final productId = state.pathParameters['id']!;
              return ProductScreen(productId: productId);
            },

            // Nested: leave review
            routes: [
              GoRoute(
                path: 'review',
                name: AppRoute.leaveReview.name,
                pageBuilder: (context, state) {
                  final productId = state.pathParameters['id']!;
                  return MaterialPage(
                    fullscreenDialog: true,
                    child: LeaveReviewScreen(productId: productId),
                  );
                },
              ),
            ],
          ),

          // Cart
          GoRoute(
            path: 'cart',
            name: AppRoute.cart.name,
            builder: (context, state) => const ShoppingCartScreen(),

            // Nested: checkout
            routes: [
              GoRoute(
                path: 'checkout',
                name: AppRoute.checkout.name,
                builder: (context, state) => const CheckoutScreen(),
              ),
            ],
          ),

          // Orders (protected)
          GoRoute(
            path: 'orders',
            name: AppRoute.orders.name,
            builder: (context, state) => const OrdersListScreen(),
          ),

          // Account (protected)
          GoRoute(
            path: 'account',
            name: AppRoute.account.name,
            builder: (context, state) => const AccountScreen(),
          ),

          // Sign in
          GoRoute(
            path: 'signIn',
            name: AppRoute.signIn.name,
            builder: (context, state) => const EmailPasswordSignInScreen(),
          ),
        ],
      ),
    ],

    // Error handling
    errorBuilder: (context, state) => const NotFoundScreen(),
  );
}
```

### Route Names Enum

**File:** [lib/src/routing/app_router.dart](lib/src/routing/app_router.dart:5-15)

```dart
enum AppRoute {
  home,
  product,
  leaveReview,
  cart,
  checkout,
  orders,
  account,
  signIn,
}
```

### Navigation Methods

#### 1. go() - Replace Current Route

```dart
// Navigate to cart
context.go('/cart');

// Navigate to product
context.goNamed(
  AppRoute.product.name,
  pathParameters: {'id': productId},
);
```

#### 2. push() - Push New Route

```dart
// Push sign-in screen
context.push('/signIn');

// Push with parameters
context.pushNamed(
  AppRoute.leaveReview.name,
  pathParameters: {'id': productId},
);
```

#### 3. pop() - Go Back

```dart
// Go back
context.pop();

// Go back with result
context.pop(true);
```

### Refresh on Auth State

**File:** [lib/src/routing/go_router_refresh_stream.dart](lib/src/routing/go_router_refresh_stream.dart:5-20)

```dart
/// Notifies GoRouter to refresh when stream emits
class GoRouterRefreshStream extends ChangeNotifier {
  GoRouterRefreshStream(Stream<dynamic> stream) {
    _subscription = stream.asBroadcastStream().listen(
      (dynamic _) => notifyListeners(),
    );
  }

  late final StreamSubscription<dynamic> _subscription;

  @override
  void dispose() {
    _subscription.cancel();
    super.dispose();
  }
}
```

**How it works:**
1. Auth state changes (user signs in/out)
2. Stream emits new value
3. `GoRouterRefreshStream` calls `notifyListeners()`
4. GoRouter re-runs `redirect` logic
5. User is redirected if needed

### Route Guards (Redirects)

**Protected Routes:**
```dart
redirect: (context, state) {
  final isLoggedIn = authRepository.currentUser != null;
  final path = state.uri.path;

  // Only logged-in users can access these
  if (!isLoggedIn && (path == '/orders' || path == '/account')) {
    return '/';  // Redirect to home
  }

  // Logged-in users don't need sign-in page
  if (isLoggedIn && path == '/signIn') {
    return '/';
  }

  return null;  // No redirect needed
}
```

### Deep Linking

GoRouter automatically handles deep links:

```
# URL format
myapp://product/123

# Maps to route
GoRoute(
  path: 'product/:id',
  builder: (context, state) {
    final id = state.pathParameters['id'];  // '123'
    return ProductScreen(productId: id);
  },
)
```

### Nested Routes

**Example: Product → Review**

```dart
GoRoute(
  path: 'product/:id',  // /product/123
  builder: (context, state) => ProductScreen(...),

  routes: [
    GoRoute(
      path: 'review',  // /product/123/review
      builder: (context, state) => LeaveReviewScreen(...),
    ),
  ],
)
```

**Navigation:**
```dart
// From anywhere
context.go('/product/123/review');

// Or with named routes
context.goNamed(
  AppRoute.leaveReview.name,
  pathParameters: {'id': '123'},
);
```

---

## Real-World Examples

### Example 1: Add to Cart Flow

**1. User taps "Add to Cart" button**

**File:** [lib/src/features/cart/presentation/add_to_cart/add_to_cart_widget.dart](lib/src/features/cart/presentation/add_to_cart/add_to_cart_widget.dart:20-50)

```dart
class AddToCartWidget extends ConsumerWidget {
  const AddToCartWidget({required this.productId});

  final ProductID productId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(addToCartControllerProvider);

    return state.when(
      data: (_) => ElevatedButton(
        onPressed: () async {
          // Get controller
          final controller = ref.read(addToCartControllerProvider.notifier);

          // Call action
          await controller.addItem(productId);

          // Show feedback
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

**2. Controller handles action**

**File:** [lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart](lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart:30-55)

```dart
@riverpod
class AddToCartController extends _$AddToCartController {
  @override
  FutureOr<void> build() {}

  Future<void> addItem(ProductID productId) async {
    // 1. Set loading state
    state = const AsyncLoading();

    // 2. Get dependencies
    final cartService = ref.read(cartServiceProvider);

    // 3. Perform action
    state = await AsyncValue.guard(() async {
      final item = Item(productId: productId, quantity: 1);
      await cartService.addItem(item);
    });

    // 4. State is now AsyncData (success) or AsyncError (failure)
  }
}
```

**3. Service coordinates**

**File:** [lib/src/features/cart/application/cart_service.dart](lib/src/features/cart/application/cart_service.dart:45-55)

```dart
class CartService {
  Future<void> addItem(Item item) async {
    // Fetch current cart
    final cart = await _fetchCart();

    // Apply business logic
    final updatedCart = cart.addItem(item);

    // Save
    await _setCart(updatedCart);
  }
}
```

**4. Repository persists**

**File:** [lib/src/features/cart/data/local/sembast_cart_repository.dart](lib/src/features/cart/data/local/sembast_cart_repository.dart:55-60)

```dart
@override
Future<void> setCart(Cart cart) async {
  await store.record(cartItemsKey).put(db, cart.toJson());
  // Stream automatically emits new cart
}
```

**5. UI updates automatically**

```dart
// Cart icon widget
class CartIcon extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final cartValue = ref.watch(cartProvider);

    return cartValue.when(
      data: (cart) => Badge(
        label: Text('${cart.items.length}'),
        child: Icon(Icons.shopping_cart),
      ),
      loading: () => Icon(Icons.shopping_cart),
      error: (_, __) => Icon(Icons.error),
    );
  }
}
```

**Flow:**
```
User taps button
     ↓
Controller sets loading state → UI shows spinner
     ↓
Service performs business logic
     ↓
Repository saves to database
     ↓
Stream emits new cart
     ↓
cartProvider notifies listeners
     ↓
All widgets watching cart rebuild → Badge updates!
```

### Example 2: Authentication Flow

**1. User signs in**

```dart
@riverpod
class EmailPasswordSignInController extends _$EmailPasswordSignInController {
  @override
  FutureOr<void> build() {}

  Future<void> signIn(String email, String password) async {
    state = const AsyncLoading();

    final authRepo = ref.read(authRepositoryProvider);

    state = await AsyncValue.guard(
      () => authRepo.signInWithEmailAndPassword(email, password),
    );
  }
}
```

**2. Auth repository updates state**

```dart
class FakeAuthRepository {
  final _authState = InMemoryStore<AppUser?>(null);

  Stream<AppUser?> authStateChanges() => _authState.stream;

  Future<void> signInWithEmailAndPassword(String email, String password) async {
    // Validate credentials
    // ...

    // Update auth state
    _authState.value = AppUser(uid: email, email: email);
  }
}
```

**3. Auth provider emits new value**

```dart
@Riverpod(keepAlive: true)
Stream<AppUser?> authStateChanges(Ref ref) {
  final authRepository = ref.watch(authRepositoryProvider);
  return authRepository.authStateChanges();
}
```

**4. GoRouter refresh triggered**

```dart
GoRouter(
  refreshListenable: GoRouterRefreshStream(
    authRepository.authStateChanges(),
  ),
  redirect: (context, state) {
    final isLoggedIn = authRepository.currentUser != null;

    // User just signed in → redirect from sign-in page
    if (isLoggedIn && state.uri.path == '/signIn') {
      return '/';
    }

    return null;
  },
)
```

**5. Cart sync triggered**

```dart
class CartSyncService {
  void _init() {
    ref.listen<AsyncValue<AppUser?>>(
      authStateChangesProvider,
      (previous, next) {
        final wasSignedOut = previous?.value == null;
        final isSignedIn = next.value != null;

        // User just signed in
        if (wasSignedOut && isSignedIn) {
          _moveItemsToRemoteCart(next.value!.uid);
        }
      },
    );
  }
}
```

**Complete reactive flow!**

---

## Summary

### Riverpod Key Concepts

| Concept | Purpose | Example |
|---------|---------|---------|
| **Provider** | Provide values | `@riverpod String name(Ref ref)` |
| **Ref** | Access providers | `ref.watch(cartProvider)` |
| **Watch** | Reactive updates | `ref.watch(cartProvider)` |
| **Read** | One-time read | `ref.read(cartServiceProvider)` |
| **Listen** | Side effects | `ref.listen(provider, callback)` |
| **AsyncValue** | Async states | `.when(data: , loading: , error:)` |
| **Notifier** | Stateful logic | `class Controller extends _$Controller` |
| **Family** | Parameterized | `product(Ref ref, String id)` |

### GoRouter Key Concepts

| Concept | Purpose | Example |
|---------|---------|---------|
| **GoRoute** | Define route | `path: '/cart'` |
| **go()** | Navigate (replace) | `context.go('/cart')` |
| **push()** | Navigate (stack) | `context.push('/signIn')` |
| **pop()** | Go back | `context.pop()` |
| **redirect** | Route guards | Protected routes |
| **refreshListenable** | Auto-refresh | On auth change |
| **pathParameters** | Route params | `/:id` |

---

## Next Steps

- **[Features Deep-Dive](05-features-deep-dive.md)**: See these patterns in complete features
- **[Getting Started](06-getting-started-guide.md)**: Apply what you've learned

---
