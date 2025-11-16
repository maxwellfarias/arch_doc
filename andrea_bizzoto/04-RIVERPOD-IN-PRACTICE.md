# Riverpod in Practice - Real Implementations from the Project

## Table of Contents
- [Overview](#overview)
- [Authentication Flow with Riverpod](#authentication-flow-with-riverpod)
- [Product Catalog Implementation](#product-catalog-implementation)
- [Shopping Cart Architecture](#shopping-cart-architecture)
- [Checkout Process](#checkout-process)
- [Order Management](#order-management)
- [Reviews System](#reviews-system)
- [Advanced Patterns](#advanced-patterns)
- [Performance Optimization](#performance-optimization)
- [Error Handling](#error-handling)
- [Best Practices Summary](#best-practices-summary)

---

## Overview

In the previous guide, we learned **what** Riverpod is and **how** it works. Now, let's see **real implementations** from this e-commerce project.

We'll explore:
- How providers are organized in each feature
- How they interact with each other
- Real-world patterns and solutions
- Performance considerations
- Testing strategies

---

## Authentication Flow with Riverpod

Authentication is a perfect example of Riverpod's power. Let's see how auth state flows through the entire app.

### The Provider Chain

```
authRepositoryProvider (singleton)
        ↓
authStateChangesProvider (stream of user)
        ↓
Multiple features watch this
```

### 1. Auth Repository Provider

**File:** [lib/src/features/authentication/data/fake_auth_repository.dart](../lib/src/features/authentication/data/fake_auth_repository.dart)

```dart
/// Singleton auth repository (lives for app lifetime)
@Riverpod(keepAlive: true)
FakeAuthRepository authRepository(Ref ref) {
  final auth = FakeAuthRepository();
  // Cleanup when app closes
  ref.onDispose(() => auth.dispose());
  return auth;
}
```

**Why keepAlive?**
- Auth repository should exist throughout the app lifetime
- Multiple features depend on it
- Expensive to recreate

### 2. Auth State Stream Provider

```dart
/// Stream of authentication state changes
@Riverpod(keepAlive: true)
Stream<AppUser?> authStateChanges(Ref ref) {
  final authRepository = ref.watch(authRepositoryProvider);
  return authRepository.authStateChanges();
}
```

**Key Points:**
- ✅ Emits whenever user signs in/out
- ✅ Other providers can watch this for reactive behavior
- ✅ Returns `AppUser?` (null when signed out)

### 3. Sign-In Controller

**File:** [lib/src/features/authentication/presentation/sign_in/email_password_sign_in_controller.dart](../lib/src/features/authentication/presentation/sign_in/email_password_sign_in_controller.dart)

```dart
@riverpod
class EmailPasswordSignInController extends _$EmailPasswordSignInController {
  @override
  FutureOr<void> build() {
    // Initial state: not loading
  }

  Future<void> submit({
    required String email,
    required String password,
    required EmailPasswordSignInFormType formType,
  }) async {
    // Set loading state (UI shows spinner)
    state = const AsyncLoading();

    // Get auth repository
    final authRepository = ref.read(authRepositoryProvider);

    // Perform async operation with automatic error handling
    state = await AsyncValue.guard(() async {
      if (formType == EmailPasswordSignInFormType.signIn) {
        await authRepository.signInWithEmailAndPassword(email, password);
      } else {
        await authRepository.createUserWithEmailAndPassword(email, password);
      }
    });
  }
}
```

**Pattern Highlights:**
- ✅ `AsyncLoading` → UI shows loading spinner
- ✅ `AsyncValue.guard` → Automatically catches errors
- ✅ `ref.read` → One-time read (not watching for changes)
- ✅ Result is `AsyncData<void>` (success) or `AsyncError` (failure)

### 4. UI Integration

**File:** [lib/src/features/authentication/presentation/sign_in/email_password_sign_in_screen.dart](../lib/src/features/authentication/presentation/sign_in/email_password_sign_in_screen.dart)

```dart
class EmailPasswordSignInScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Listen for errors and show dialog
    ref.listen<AsyncValue<void>>(
      emailPasswordSignInControllerProvider,
      (_, state) => state.showAlertDialogOnError(context),
    );

    // Watch the state for loading indicator
    final state = ref.watch(emailPasswordSignInControllerProvider);

    return Scaffold(
      body: FormWidget(
        onSubmit: state.isLoading
            ? null  // Disable button while loading
            : () {
                // Call controller
                ref
                    .read(emailPasswordSignInControllerProvider.notifier)
                    .submit(
                      email: emailController.text,
                      password: passwordController.text,
                      formType: formType,
                    );
              },
      ),
    );
  }
}
```

**UI Patterns:**
- ✅ `ref.listen` → Show error dialogs (side effects)
- ✅ `ref.watch` → Disable button while loading
- ✅ `ref.read` → Call controller methods
- ✅ Separation: UI doesn't know about auth implementation

### 5. Reactive Navigation

**File:** [lib/src/routing/app_router.dart](../lib/src/routing/app_router.dart)

```dart
@Riverpod(keepAlive: true)
GoRouter goRouter(Ref ref) {
  final authRepository = ref.watch(authRepositoryProvider);

  return GoRouter(
    // Navigation refreshes when auth state changes!
    refreshListenable: GoRouterRefreshStream(
      authRepository.authStateChanges(),
    ),
    redirect: (context, state) {
      final isLoggedIn = authRepository.currentUser != null;
      final path = state.uri.path;

      if (isLoggedIn) {
        // Redirect away from sign-in if already logged in
        if (path == '/signIn') {
          return '/';
        }
      } else {
        // Redirect to home if trying to access protected routes
        if (path == '/account' || path == '/orders') {
          return '/';
        }
      }
      return null;
    },
    routes: [...],
  );
}
```

**Reactive Routing:**
- ✅ Router watches auth state
- ✅ Auto-redirects when user signs in/out
- ✅ Route guards based on authentication

### Complete Auth Flow Diagram

```
User taps "Sign In"
        ↓
EmailPasswordSignInController.submit()
        ↓
Sets state = AsyncLoading (UI shows spinner)
        ↓
Calls authRepository.signInWithEmailAndPassword()
        ↓
FakeAuthRepository updates _authState stream
        ↓
authStateChangesProvider emits new user
        ↓
REACTIVE EFFECTS:
  1. GoRouter detects change → redirects to home
  2. Cart sync service detects change → syncs cart
  3. Account screen shows user info
  4. Navigation rebuilds
        ↓
Controller state = AsyncData (success)
        ↓
UI button re-enables
```

---

## Product Catalog Implementation

The product catalog demonstrates **computed providers** and **search functionality**.

### Provider Architecture

```
productsRepositoryProvider (singleton)
        ↓
productsListStreamProvider (all products)
        ↓
productsSearchQueryStateProvider (user's search query)
        ↓
productsListSearchProvider(query) (filtered products)
        ↓
productsSearchResultsProvider (current search results)
```

### 1. Products Repository

**File:** [lib/src/features/products/data/fake_products_repository.dart](../lib/src/features/products/data/fake_products_repository.dart)

```dart
@Riverpod(keepAlive: true)
FakeProductsRepository productsRepository(Ref ref) {
  return FakeProductsRepository();
}

@Riverpod(keepAlive: true)
Stream<List<Product>> productsListStream(Ref ref) {
  final productsRepository = ref.watch(productsRepositoryProvider);
  return productsRepository.watchProductsList();
}

@riverpod
Future<List<Product>> productsList(Ref ref) {
  final productsRepository = ref.watch(productsRepositoryProvider);
  return productsRepository.fetchProductsList();
}
```

**Pattern: Multiple Accessors**
- `Stream` → For reactive UI (auto-updates)
- `Future` → For one-time fetch
- Both use the same repository

### 2. Single Product Provider (Family)

```dart
@riverpod
Stream<Product?> product(Ref ref, ProductID id) {
  final productsRepository = ref.watch(productsRepositoryProvider);
  return productsRepository.watchProduct(id);
}
```

**Family Pattern:**
- Each product ID creates a separate provider instance
- Cached independently
- Auto-disposes when no longer watched

**Usage:**
```dart
// Different instances!
final product1 = ref.watch(productProvider('product-1'));
final product2 = ref.watch(productProvider('product-2'));
```

### 3. Search Implementation

**File:** [lib/src/features/products/presentation/products_list/products_search_bar.dart](../lib/src/features/products/presentation/products_list/products_search_bar.dart)

```dart
/// Simple state for search query
final productsSearchQueryStateProvider = StateProvider<String>((ref) {
  return '';
});
```

**Search Provider (with parameters):**

```dart
@riverpod
Future<List<Product>> productsListSearch(Ref ref, String query) {
  final productsRepository = ref.watch(productsRepositoryProvider);
  return productsRepository.searchProducts(query);
}
```

**Computed Search Results:**

```dart
@riverpod
Future<List<Product>> productsSearchResults(Ref ref) {
  final searchQuery = ref.watch(productsSearchQueryStateProvider);
  return ref.watch(productsListSearchProvider(searchQuery).future);
}
```

**Pattern Explanation:**
1. User types in search bar
2. `productsSearchQueryStateProvider` updates
3. `productsSearchResultsProvider` detects change (watches query)
4. Calls `productsListSearchProvider(newQuery)`
5. UI rebuilds with new results

### 4. Products List Screen

**File:** [lib/src/features/products/presentation/products_list/products_list_screen.dart](../lib/src/features/products/presentation/products_list/products_list_screen.dart)

```dart
class ProductsListScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Products'),
        actions: const [ShoppingCartIcon()],  // Watches cart count!
      ),
      body: CustomScrollView(
        slivers: [
          const ProductsSearchBar(),  // Updates search query
          Consumer(
            builder: (context, ref, child) {
              // Watches product stream
              final productsListValue = ref.watch(productsListStreamProvider);

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

### 5. Shopping Cart Icon Badge

```dart
class ShoppingCartIcon extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Computed provider: counts items in cart
    final cartItemsCount = ref.watch(cartItemsCountProvider);

    return Stack(
      children: [
        IconButton(
          icon: Icon(Icons.shopping_cart),
          onPressed: () => context.pushNamed(AppRoute.cart.name),
        ),
        if (cartItemsCount > 0)
          Positioned(
            right: 4,
            top: 4,
            child: CircleBadge(
              label: '$cartItemsCount',
            ),
          ),
      ],
    );
  }
}
```

**Reactive Badge:**
- Watches `cartItemsCountProvider`
- Auto-updates when cart changes
- No manual state management needed!

---

## Shopping Cart Architecture

The cart is the most complex feature, demonstrating **advanced Riverpod patterns**.

### Provider Hierarchy

```
authStateChangesProvider
        ↓
localCartRepositoryProvider ──┐
remoteCartRepositoryProvider ─┤
        ↓                      │
cartServiceProvider ───────────┘
        ↓
cartProvider (stream)
        ↓
cartItemsCountProvider (computed)
        ↓
addToCartControllerProvider (actions)
```

### 1. Repository Providers

**File:** [lib/src/features/cart/data/local/local_cart_repository.dart](../lib/src/features/cart/data/local/local_cart_repository.dart)

```dart
/// Local cart repository (Sembast)
/// Must be overridden at app startup with actual instance
@Riverpod(keepAlive: true)
LocalCartRepository localCartRepository(Ref ref) {
  throw UnimplementedError();
}
```

**File:** [lib/src/features/cart/data/remote/remote_cart_repository.dart](../lib/src/features/cart/data/remote/remote_cart_repository.dart)

```dart
/// Remote cart repository (fake implementation)
@Riverpod(keepAlive: true)
RemoteCartRepository remoteCartRepository(Ref ref) {
  return FakeRemoteCartRepository();
}
```

**Override Pattern in main.dart:**

```dart
void main() async {
  // Create actual local repository
  final localCartRepository = await SembastCartRepository.makeDefault();

  final container = ProviderContainer(
    overrides: [
      // Override the placeholder with real implementation
      localCartRepositoryProvider.overrideWithValue(localCartRepository),
    ],
  );

  runApp(UncontrolledProviderScope(
    container: container,
    child: const MyApp(),
  ));
}
```

**Why This Pattern?**
- ✅ Can't create Sembast DB synchronously
- ✅ Must initialize before app starts
- ✅ Override pattern allows async initialization

### 2. Cart Service (Orchestration)

**File:** [lib/src/features/cart/application/cart_service.dart](../lib/src/features/cart/application/cart_service.dart)

```dart
class CartService {
  CartService(this.ref);
  final Ref ref;

  /// Fetch cart based on auth status
  Future<Cart> _fetchCart() {
    final user = ref.read(authRepositoryProvider).currentUser;
    if (user != null) {
      return ref.read(remoteCartRepositoryProvider).fetchCart(user.uid);
    } else {
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

  /// Add item to cart
  Future<void> addItem(Item item) async {
    // Validate product exists
    final product = await ref
        .read(productsRepositoryProvider)
        .fetchProduct(item.productId);
    if (product == null) {
      throw ProductNotFoundException();
    }

    // Get current cart
    final cart = await _fetchCart();

    // Apply business logic (immutable update)
    final updated = cart.addItem(item);

    // Persist
    await _setCart(updated);
  }

  Future<void> removeItemById(ProductID productId) async {
    final cart = await _fetchCart();
    final updated = cart.removeItemById(productId);
    await _setCart(updated);
  }

  Future<void> setItem(Item item) async {
    final cart = await _fetchCart();
    final updated = cart.setItem(item);
    await _setCart(updated);
  }
}

/// Service provider
@riverpod
CartService cartService(Ref ref) {
  return CartService(ref);
}
```

**Service Pattern Benefits:**
- ✅ Encapsulates cart switching logic (local vs remote)
- ✅ Validates business rules
- ✅ Coordinates multiple repositories
- ✅ Single source of truth

### 3. Cart Stream Provider

```dart
@riverpod
Stream<Cart> cart(Ref ref) {
  final user = ref.watch(authStateChangesProvider).value;
  if (user != null) {
    return ref.watch(remoteCartRepositoryProvider).watchCart(user.uid);
  } else {
    return ref.watch(localCartRepositoryProvider).watchCart();
  }
}
```

**Reactive Cart Switching:**
- User signs in → automatically switches to remote cart
- User signs out → automatically switches to local cart
- No manual intervention needed!

### 4. Computed Cart Count

```dart
@Riverpod(keepAlive: true)
int cartItemsCount(Ref ref) {
  return ref.watch(cartProvider).maybeMap(
    data: (cart) => cart.value.items.length,
    orElse: () => 0,
  );
}
```

**Pattern: Safe Unwrapping**
- `maybeMap` handles all AsyncValue states
- Returns 0 if loading or error
- Only counts when data is available

### 5. Cart Sync Service

**File:** [lib/src/features/cart/application/cart_sync_service.dart](../lib/src/features/cart/application/cart_sync_service.dart)

```dart
@Riverpod(keepAlive: true)
CartSyncService cartSyncService(Ref ref) {
  return CartSyncService(ref);
}

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

        // User just signed in
        if (previousUser == null && user != null) {
          _moveItemsToRemoteCart(user.uid);
        }
      },
    );
  }

  Future<void> _moveItemsToRemoteCart(String uid) async {
    try {
      final localCartRepository = ref.read(localCartRepositoryProvider);
      final remoteCartRepository = ref.read(remoteCartRepositoryProvider);
      final productsRepository = ref.read(productsRepositoryProvider);

      // Get both carts
      final localCart = await localCartRepository.fetchCart();
      final remoteCart = await remoteCartRepository.fetchCart(uid);

      // Calculate items to add (with inventory checks)
      final localItemsToAdd = <Item>[];
      for (final entry in localCart.items.entries) {
        final productId = entry.key;
        final localQuantity = entry.value;
        final remoteQuantity = remoteCart.items[productId] ?? 0;

        // Get product for inventory check
        final product = await productsRepository.fetchProduct(productId);
        if (product != null) {
          final totalQuantity = localQuantity + remoteQuantity;
          final cappedQuantity = min(totalQuantity, product.availableQuantity);
          final quantityToAdd = cappedQuantity - remoteQuantity;

          if (quantityToAdd > 0) {
            localItemsToAdd.add(Item(
              productId: productId,
              quantity: quantityToAdd,
            ));
          }
        }
      }

      // Merge carts
      final updatedRemoteCart = remoteCart.addItems(localItemsToAdd);
      await remoteCartRepository.setCart(uid, updatedRemoteCart);

      // Clear local cart
      await localCartRepository.setCart(const Cart());
    } catch (e) {
      ref.read(errorLoggerProvider).logError(e, StackTrace.current);
    }
  }
}
```

**Advanced Pattern: Automatic Side Effect**
- ✅ Listens to auth changes
- ✅ Automatically syncs cart on sign-in
- ✅ Handles inventory constraints
- ✅ Error handling with logging
- ✅ User doesn't trigger this manually

**Initialization in main.dart:**

```dart
void main() async {
  // ...
  final container = ProviderContainer(/* ... */);

  // Initialize cart sync service (starts listening)
  container.read(cartSyncServiceProvider);

  runApp(/* ... */);
}
```

### 6. Add to Cart Controller

**File:** [lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart](../lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart)

```dart
@riverpod
class AddToCartController extends _$AddToCartController {
  @override
  FutureOr<void> build() {
    // Initial state
  }

  Future<void> addItem(ProductID productId) async {
    final cartService = ref.read(cartServiceProvider);
    final quantity = ref.read(itemQuantityControllerProvider);
    final item = Item(productId: productId, quantity: quantity);

    state = const AsyncLoading<void>();

    state = await AsyncValue.guard(() => cartService.addItem(item));

    if (!state.hasError) {
      // Reset quantity on success
      ref.read(itemQuantityControllerProvider.notifier).updateQuantity(1);
    }
  }
}
```

### 7. Item Quantity Controller

**File:** [lib/src/features/cart/presentation/add_to_cart/item_quantity_controller.dart](../lib/src/features/cart/presentation/add_to_cart/item_quantity_controller.dart)

```dart
@riverpod
class ItemQuantityController extends _$ItemQuantityController {
  @override
  int build() => 1;

  void increment() => state = min(state + 1, 99);
  void decrement() => state = max(state - 1, 1);
  void updateQuantity(int quantity) => state = quantity.clamp(1, 99);
}
```

**UI Integration:**

```dart
class ItemQuantitySelector extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final quantity = ref.watch(itemQuantityControllerProvider);

    return Row(
      children: [
        IconButton(
          icon: Icon(Icons.remove),
          onPressed: () {
            ref.read(itemQuantityControllerProvider.notifier).decrement();
          },
        ),
        Text('$quantity'),
        IconButton(
          icon: Icon(Icons.add),
          onPressed: () {
            ref.read(itemQuantityControllerProvider.notifier).increment();
          },
        ),
      ],
    );
  }
}
```

### Complete Cart Flow

```
User adds product to cart
        ↓
AddToCartController.addItem()
        ↓
Reads itemQuantityControllerProvider
Creates Item(productId, quantity)
        ↓
Sets state = AsyncLoading (button disabled)
        ↓
Calls cartService.addItem()
        ↓
CartService checks auth state
  - Signed in? → Use remote repository
  - Guest? → Use local repository
        ↓
CartService validates product exists
        ↓
Fetches current cart
        ↓
Applies cart.addItem() (immutable update)
        ↓
Saves to repository (Sembast or remote)
        ↓
Repository emits stream event
        ↓
cartProvider detects change
        ↓
REACTIVE UPDATES:
  1. Cart icon badge updates (cartItemsCountProvider)
  2. Cart screen shows new item
  3. Add to cart button re-enables
        ↓
Controller state = AsyncData (success)
        ↓
Quantity resets to 1
```

---

## Checkout Process

### 1. Checkout Service

**File:** [lib/src/features/checkout/application/fake_checkout_service.dart](../lib/src/features/checkout/application/fake_checkout_service.dart)

```dart
class FakeCheckoutService {
  FakeCheckoutService(this.ref);
  final Ref ref;

  Future<void> placeOrder() async {
    final uid = ref.read(authStateChangesProvider).value?.uid;
    if (uid == null) {
      throw UserNotSignedInException();
    }

    // Get cart
    final cart = await ref.read(cartProvider.future);

    // Validate cart is not empty
    if (cart.items.isEmpty) {
      throw EmptyCartException();
    }

    // Create order
    final total = await _computeTotal(cart);
    final orderDate = DateTime.now();
    final orderId = orderDate.toIso8601String();

    final order = Order(
      id: orderId,
      userId: uid,
      items: cart.items,
      orderStatus: OrderStatus.confirmed,
      orderDate: orderDate,
      total: total,
    );

    // Save order
    await ref.read(ordersRepositoryProvider).addOrder(uid, order);

    // Clear cart
    await ref.read(cartServiceProvider).setCart(const Cart());
  }

  Future<double> _computeTotal(Cart cart) async {
    var total = 0.0;
    final productsList = await ref.read(productsListProvider.future);

    for (final entry in cart.items.entries) {
      final product = productsList.firstWhere((p) => p.id == entry.key);
      total += product.price * entry.value;
    }
    return total;
  }
}

@riverpod
FakeCheckoutService checkoutService(Ref ref) {
  return FakeCheckoutService(ref);
}
```

### 2. Payment Controller

**File:** [lib/src/features/checkout/presentation/payment/payment_button_controller.dart](../lib/src/features/checkout/presentation/payment/payment_button_controller.dart)

```dart
@riverpod
class PaymentButtonController extends _$PaymentButtonController {
  @override
  FutureOr<void> build() {
    // Initial state
  }

  Future<void> pay() async {
    final checkoutService = ref.read(checkoutServiceProvider);

    state = const AsyncLoading();

    state = await AsyncValue.guard(() => checkoutService.placeOrder());
  }
}
```

### 3. Checkout Screen

```dart
class CheckoutScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Listen for successful payment
    ref.listen<AsyncValue<void>>(
      paymentButtonControllerProvider,
      (_, state) {
        if (state.hasValue) {
          // Navigate to orders
          context.goNamed(AppRoute.orders.name);
        } else if (state.hasError) {
          state.showAlertDialogOnError(context);
        }
      },
    );

    final state = ref.watch(paymentButtonControllerProvider);

    return Scaffold(
      body: Column(
        children: [
          const CartItemsList(),
          const CartTotal(),
          PrimaryButton(
            text: 'Place Order',
            isLoading: state.isLoading,
            onPressed: state.isLoading
                ? null
                : () {
                    ref
                        .read(paymentButtonControllerProvider.notifier)
                        .pay();
                  },
          ),
        ],
      ),
    );
  }
}
```

---

## Order Management

### 1. Orders Repository

**File:** [lib/src/features/orders/data/fake_orders_repository.dart](../lib/src/features/orders/data/fake_orders_repository.dart)

```dart
@Riverpod(keepAlive: true)
FakeOrdersRepository ordersRepository(Ref ref) {
  return FakeOrdersRepository();
}

/// Watch all orders for a user
@riverpod
Stream<List<Order>> userOrders(Ref ref, String uid) {
  final ordersRepository = ref.watch(ordersRepositoryProvider);
  return ordersRepository.watchUserOrders(uid);
}

/// Fetch orders (one-time)
@riverpod
Future<List<Order>> userOrdersFuture(Ref ref, String uid) {
  final ordersRepository = ref.watch(ordersRepositoryProvider);
  return ordersRepository.fetchUserOrders(uid);
}
```

### 2. Orders List Screen

**File:** [lib/src/features/orders/presentation/orders_list/orders_list_screen.dart](../lib/src/features/orders/presentation/orders_list/orders_list_screen.dart)

```dart
class OrdersListScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final user = ref.watch(authStateChangesProvider).value;

    if (user == null) {
      return const EmptyPlaceholderWidget(
        message: 'Please sign in to view your orders',
      );
    }

    // Watch user's orders
    final ordersValue = ref.watch(userOrdersProvider(user.uid));

    return AsyncValueWidget<List<Order>>(
      value: ordersValue,
      data: (orders) => orders.isEmpty
          ? const EmptyPlaceholderWidget(
              message: 'No orders yet',
            )
          : ListView.builder(
              itemCount: orders.length,
              itemBuilder: (context, index) {
                final order = orders[index];
                return OrderCard(order: order);
              },
            ),
    );
  }
}
```

**Pattern: Auth-Dependent Data**
- Checks auth state first
- Only fetches orders if user is signed in
- Provider family with uid parameter

---

## Reviews System

### 1. Reviews Service

**File:** [lib/src/features/reviews/application/reviews_service.dart](../lib/src/features/reviews/application/reviews_service.dart)

```dart
class ReviewsService {
  ReviewsService(this.ref);
  final Ref ref;

  Future<void> submitReview({
    required ProductID productId,
    required Review review,
  }) async {
    final user = ref.read(authStateChangesProvider).value;
    if (user == null) {
      throw UserNotSignedInException();
    }

    // Check if user purchased this product
    final orders = await ref.read(userOrdersFutureProvider(user.uid).future);
    final purchasedProductIds = orders
        .expand((order) => order.items.keys)
        .toSet();

    if (!purchasedProductIds.contains(productId)) {
      throw ProductNotPurchasedException();
    }

    // Submit review
    await ref
        .read(reviewsRepositoryProvider)
        .setReview(productId, user.uid, review);
  }
}

@riverpod
ReviewsService reviewsService(Ref ref) {
  return ReviewsService(ref);
}
```

**Business Logic:**
- ✅ Validates user is signed in
- ✅ Checks user purchased the product
- ✅ Prevents fake reviews

### 2. Product Reviews Provider

```dart
@riverpod
Stream<List<Review>> productReviews(Ref ref, ProductID id) {
  return ref.watch(reviewsRepositoryProvider).watchProductReviews(id);
}

@riverpod
Future<Review?> userProductReview(Ref ref, ProductID id) async {
  final user = ref.watch(authStateChangesProvider).value;
  if (user != null) {
    return ref.read(reviewsRepositoryProvider).fetchUserReview(id, user.uid);
  } else {
    return null;
  }
}
```

### 3. Leave Review Controller

**File:** [lib/src/features/reviews/presentation/leave_review_screen/leave_review_controller.dart](../lib/src/features/reviews/presentation/leave_review_screen/leave_review_controller.dart)

```dart
@riverpod
class LeaveReviewController extends _$LeaveReviewController {
  @override
  FutureOr<void> build() {}

  Future<void> submitReview({
    required ProductID productId,
    required double rating,
    required String comment,
  }) async {
    final review = Review(
      rating: rating,
      comment: comment,
      date: DateTime.now(),
    );

    final reviewsService = ref.read(reviewsServiceProvider);

    state = const AsyncLoading();

    state = await AsyncValue.guard(() async {
      await reviewsService.submitReview(
        productId: productId,
        review: review,
      );
    });
  }
}
```

---

## Advanced Patterns

### Pattern 1: Provider Observers for Logging

**File:** [lib/main.dart](../lib/main.dart)

```dart
class AsyncErrorLogger extends ProviderObserver {
  @override
  void didUpdateProvider(
    ProviderBase provider,
    Object? previousValue,
    Object? newValue,
    ProviderContainer container,
  ) {
    final errorLogger = container.read(errorLoggerProvider);
    final error = _findError(newValue);

    if (error != null) {
      if (error.error is AppException) {
        errorLogger.logAppException(error.error as AppException);
      } else {
        errorLogger.logError(error.error, error.stackTrace);
      }
    }
  }

  AsyncError<Object>? _findError(Object? value) {
    if (value is AsyncError) {
      return value;
    }
    return null;
  }
}

// In main:
final container = ProviderContainer(
  observers: [AsyncErrorLogger()],
);
```

**Benefits:**
- ✅ Centralized error logging
- ✅ All AsyncValue errors are logged automatically
- ✅ No need to handle logging in each controller

### Pattern 2: Conditional Provider Watching

```dart
@riverpod
Stream<Cart> cart(Ref ref) {
  // Watch auth state
  final user = ref.watch(authStateChangesProvider).value;

  // Conditionally watch different providers
  if (user != null) {
    return ref.watch(remoteCartRepositoryProvider).watchCart(user.uid);
  } else {
    return ref.watch(localCartRepositoryProvider).watchCart();
  }
}
```

**Pattern: Dynamic Dependencies**
- Provider dependencies change based on state
- Automatically switches streams
- Clean and declarative

### Pattern 3: Computed Derived State

```dart
@Riverpod(keepAlive: true)
int cartItemsCount(Ref ref) {
  return ref.watch(cartProvider).maybeMap(
    data: (cart) => cart.value.items.length,
    orElse: () => 0,
  );
}

@riverpod
Future<double> cartTotal(Ref ref) async {
  final cart = await ref.watch(cartProvider.future);
  final productsList = await ref.watch(productsListProvider.future);

  var total = 0.0;
  for (final entry in cart.items.entries) {
    final product = productsList.firstWhere((p) => p.id == entry.key);
    total += product.price * entry.value;
  }
  return total;
}
```

**Benefits:**
- ✅ No manual state management
- ✅ Auto-recalculates when dependencies change
- ✅ Cached (only recalculates when needed)

### Pattern 4: Side Effect Listeners

```dart
// In widget
ref.listen<AsyncValue<void>>(
  addToCartControllerProvider,
  (previous, next) {
    if (!next.isLoading && next.hasValue) {
      // Success: Show snackbar
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Added to cart!')),
      );
    }
  },
);
```

**Use Cases:**
- Show dialogs
- Show snackbars
- Navigate
- Log analytics events

---

## Performance Optimization

### 1. Keep Alive for Expensive Operations

```dart
// Heavy computation or singleton
@Riverpod(keepAlive: true)
ExpensiveService expensiveService(Ref ref) {
  return ExpensiveService();
}
```

### 2. Auto-Dispose for Temporary State

```dart
// Screen-specific state (auto-disposes when screen closes)
@riverpod
class ScreenState extends _$ScreenState {
  // Will auto-dispose when no widgets watch it
}
```

### 3. Selective Rebuilds with select

```dart
// Only rebuild when specific property changes
final username = ref.watch(
  userProvider.select((user) => user.name),
);
```

### 4. Provider Families for Independent Caching

```dart
@riverpod
Future<Product> product(Ref ref, String id) {
  // Each product cached separately
  return fetchProduct(id);
}
```

### 5. Debouncing Search

```dart
@riverpod
class SearchQuery extends _$SearchQuery {
  Timer? _debounceTimer;

  @override
  String build() => '';

  void updateQuery(String query) {
    _debounceTimer?.cancel();
    _debounceTimer = Timer(Duration(milliseconds: 500), () {
      state = query;
    });
  }
}
```

---

## Error Handling

### 1. Global Error Handling

```dart
// Provider observer logs all errors
class AsyncErrorLogger extends ProviderObserver {
  @override
  void didUpdateProvider(/* ... */) {
    final error = _findError(newValue);
    if (error != null) {
      logError(error);
    }
  }
}
```

### 2. Per-Screen Error Handling

```dart
ref.listen<AsyncValue<void>>(
  controllerProvider,
  (_, state) => state.showAlertDialogOnError(context),
);
```

### 3. Graceful Degradation

```dart
final productsAsync = ref.watch(productsListProvider);

return productsAsync.when(
  loading: () => LoadingShimmer(),
  error: (err, _) => ErrorWidget(
    error: err,
    onRetry: () => ref.invalidate(productsListProvider),
  ),
  data: (products) => ProductList(products),
);
```

### 4. Custom Exceptions

```dart
// Custom exception types
class ProductNotFoundException extends AppException {
  ProductNotFoundException()
      : super('Product not found');
}

// In controller
Future<void> addItem(ProductID id) async {
  state = await AsyncValue.guard(() async {
    final product = await fetchProduct(id);
    if (product == null) {
      throw ProductNotFoundException();
    }
    // ...
  });
}
```

---

## Best Practices Summary

### ✅ DO

1. **Use code generation** - Less boilerplate, type-safe
2. **Keep repositories as providers** - Singleton pattern
3. **Use services for complex logic** - Coordinate multiple repos
4. **Use controllers for UI actions** - AsyncNotifier pattern
5. **Watch in build, read in callbacks** - Correct reactive behavior
6. **Use ref.listen for side effects** - Navigation, dialogs
7. **Use keepAlive for singletons** - Repositories, services
8. **Use families for parameterized data** - Products, orders by ID
9. **Use AsyncValue.guard** - Automatic error handling
10. **Override providers in tests** - Easy mocking

### ❌ DON'T

1. **Don't use ref.read in build** - Won't rebuild!
2. **Don't create providers in widgets** - Define globally
3. **Don't store UI state in services** - Controllers only
4. **Don't forget to dispose** - Use auto-dispose or keepAlive
5. **Don't mix business logic in widgets** - Use controllers/services
6. **Don't ignore AsyncValue states** - Handle loading/error/data
7. **Don't create circular dependencies** - Plan provider hierarchy
8. **Don't overuse keepAlive** - Memory leaks possible
9. **Don't access context in providers** - Use Ref instead
10. **Don't forget provider observers** - Great for logging

---

## Summary

This project demonstrates **production-grade Riverpod patterns**:

✅ **Repository providers** - Singleton data sources
✅ **Service providers** - Business logic orchestration
✅ **Controller providers** - UI action handlers
✅ **Stream providers** - Reactive data
✅ **Computed providers** - Derived state
✅ **Provider families** - Parameterized providers
✅ **Provider observers** - Global logging
✅ **Override pattern** - Async initialization & testing
✅ **AsyncValue** - Error handling
✅ **ref.listen** - Side effects

**Key Insights:**
- Providers create a **dependency graph**
- Changes propagate **reactively**
- UI rebuilds **automatically**
- Business logic is **testable**
- Code is **declarative and clean**

---

## What's Next?

Now that you understand how Riverpod works in practice, let's dive deeper into the **Domain and Data layers**.

Head to **[05-DOMAIN-AND-DATA-LAYERS.md](05-DOMAIN-AND-DATA-LAYERS.md)** to learn:
- How to design domain entities
- Immutability patterns
- Repository implementations
- Local storage with Sembast
- Extension methods for business logic
- Data serialization
- And much more!

Ready to master the core layers? Let's go! 🚀
