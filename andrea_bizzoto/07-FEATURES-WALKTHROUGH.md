# Features Walkthrough - Complete User Journeys

## Table of Contents
- [Introduction](#introduction)
- [Feature 1: Product Browsing](#feature-1-product-browsing)
- [Feature 2: Product Search](#feature-2-product-search)
- [Feature 3: Shopping Cart (Guest User)](#feature-3-shopping-cart-guest-user)
- [Feature 4: User Authentication](#feature-4-user-authentication)
- [Feature 5: Cart Sync on Sign-In](#feature-5-cart-sync-on-sign-in)
- [Feature 6: Checkout Process](#feature-6-checkout-process)
- [Feature 7: Order Management](#feature-7-order-management)
- [Feature 8: Product Reviews](#feature-8-product-reviews)
- [Complete Purchase Flow](#complete-purchase-flow)
- [Edge Cases and Error Handling](#edge-cases-and-error-handling)
- [Summary](#summary)

---

## Introduction

This document provides **complete walkthroughs** of each feature in the e-commerce app. We'll trace the entire user journey from UI interaction to data persistence and back, showing how all the layers work together.

### How to Read This Guide

Each feature walkthrough includes:
- **User Story**: What the user wants to do
- **UI Flow**: Step-by-step user interaction
- **Code Flow**: What happens behind the scenes
- **Data Flow**: How data moves through layers
- **Files Involved**: Exact file paths
- **Key Concepts**: Important patterns demonstrated

---

## Feature 1: Product Browsing

### User Story
> As a user, I want to browse all available products so I can see what's for sale.

### UI Flow

1. User opens the app
2. Home screen displays with products grid
3. Products load and display with images, titles, prices, ratings
4. User can scroll through products
5. User taps a product to see details

### Code Flow Diagram

```
App Starts
    ↓
main.dart initializes ProviderContainer
    ↓
MyApp widget renders
    ↓
GoRouter determines initial route (/)
    ↓
ProductsListScreen builds
    ↓
Watches productsListStreamProvider
    ↓
FakeProductsRepository.watchProductsList()
    ↓
Returns stream from InMemoryStore
    ↓
Stream emits List<Product>
    ↓
AsyncValueWidget receives data
    ↓
ProductsLayoutGrid displays products
    ↓
ProductCard for each product
```

### Detailed Code Trace

#### 1. App Initialization

**File:** [lib/main.dart](../lib/main.dart)

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Initialize local cart repository
  final localCartRepository = await SembastCartRepository.makeDefault();

  // Create provider container
  final container = ProviderContainer(
    overrides: [
      localCartRepositoryProvider.overrideWithValue(localCartRepository),
    ],
    observers: [AsyncErrorLogger()],
  );

  // Initialize cart sync service
  container.read(cartSyncServiceProvider);

  runApp(
    UncontrolledProviderScope(
      container: container,
      child: const MyApp(),
    ),
  );
}
```

#### 2. Root Widget

**File:** [lib/src/app.dart](../lib/src/app.dart)

```dart
class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final goRouter = ref.watch(goRouterProvider);

    return MaterialApp.router(
      routerConfig: goRouter,
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        primarySwatch: Colors.indigo,
        appBarTheme: const AppBarTheme(
          backgroundColor: Colors.indigo,
          foregroundColor: Colors.white,
          elevation: 2,
        ),
      ),
    );
  }
}
```

#### 3. Products List Screen

**File:** [lib/src/features/products/presentation/products_list/products_list_screen.dart](../lib/src/features/products/presentation/products_list/products_list_screen.dart)

```dart
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
              // Watch the products stream
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

#### 4. Products Provider

**File:** [lib/src/features/products/data/fake_products_repository.dart](../lib/src/features/products/data/fake_products_repository.dart)

```dart
@Riverpod(keepAlive: true)
Stream<List<Product>> productsListStream(Ref ref) {
  final productsRepository = ref.watch(productsRepositoryProvider);
  return productsRepository.watchProductsList();
}
```

#### 5. Repository Implementation

```dart
class FakeProductsRepository {
  final _products = InMemoryStore<List<Product>>(
    List.from(kTestProducts),
  );

  Stream<List<Product>> watchProductsList() => _products.stream;
}
```

### Key Concepts Demonstrated

✅ **Provider watching**: `ref.watch` makes UI reactive
✅ **Stream providers**: Continuous data updates
✅ **AsyncValue handling**: Loading, error, data states
✅ **Composition**: Screen → Grid → Card widgets
✅ **Singleton pattern**: Repository kept alive
✅ **In-memory data**: Fake repository for demo

---

## Feature 2: Product Search

### User Story
> As a user, I want to search for products by name so I can find what I'm looking for quickly.

### UI Flow

1. User types in search bar
2. Search query updates
3. Products filter in real-time
4. Results display matching products

### Code Flow Diagram

```
User types "salmon"
    ↓
TextField onChange callback
    ↓
Updates productsSearchQueryStateProvider
    ↓
productsSearchResultsProvider watches query
    ↓
Detects query change
    ↓
Calls productsListSearchProvider(query)
    ↓
Repository.searchProducts("salmon")
    ↓
Filters products by title
    ↓
Returns filtered list
    ↓
UI rebuilds with filtered products
```

### Detailed Code Trace

#### 1. Search Bar Widget

**File:** [lib/src/features/products/presentation/products_list/products_search_bar.dart](../lib/src/features/products/presentation/products_list/products_search_bar.dart)

```dart
class ProductsSearchBar extends ConsumerStatefulWidget {
  const ProductsSearchBar({super.key});

  @override
  ConsumerState<ProductsSearchBar> createState() => _ProductsSearchBarState();
}

class _ProductsSearchBarState extends ConsumerState<ProductsSearchBar> {
  final _controller = TextEditingController();

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return SliverToBoxAdapter(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: TextField(
          controller: _controller,
          decoration: InputDecoration(
            hintText: 'Search products',
            prefixIcon: const Icon(Icons.search),
            suffixIcon: IconButton(
              icon: const Icon(Icons.clear),
              onPressed: () {
                _controller.clear();
                ref.read(productsSearchQueryStateProvider.notifier).state = '';
              },
            ),
          ),
          onChanged: (value) {
            // Update search query state
            ref.read(productsSearchQueryStateProvider.notifier).state = value;
          },
        ),
      ),
    );
  }
}
```

#### 2. Search Query Provider

```dart
final productsSearchQueryStateProvider = StateProvider<String>((ref) {
  return '';
});
```

#### 3. Search Provider

```dart
@riverpod
Future<List<Product>> productsListSearch(Ref ref, String query) {
  final productsRepository = ref.watch(productsRepositoryProvider);
  return productsRepository.searchProducts(query);
}
```

#### 4. Search Results Provider

```dart
@riverpod
Future<List<Product>> productsSearchResults(Ref ref) {
  final searchQuery = ref.watch(productsSearchQueryStateProvider);
  return ref.watch(productsListSearchProvider(searchQuery).future);
}
```

#### 5. Repository Search Method

```dart
Future<List<Product>> searchProducts(String query) async {
  final lowercaseQuery = query.toLowerCase();
  final productsList = await fetchProductsList();

  return productsList
      .where((product) =>
          product.title.toLowerCase().contains(lowercaseQuery))
      .toList();
}
```

### Data Flow

```
User Input → StateProvider
     ↓
Computed Provider (watches StateProvider)
     ↓
Family Provider (with query parameter)
     ↓
Repository (filter logic)
     ↓
Filtered Results
     ↓
UI Rebuilds
```

### Key Concepts Demonstrated

✅ **StateProvider**: Simple mutable state
✅ **Computed providers**: Derive from other providers
✅ **Provider families**: Parameters for providers
✅ **Reactive search**: Auto-updates on query change
✅ **TextField integration**: Two-way binding

---

## Feature 3: Shopping Cart (Guest User)

### User Story
> As a guest user, I want to add products to my cart and have them persist locally so I don't lose my selections.

### UI Flow

1. User views product details
2. User selects quantity
3. User taps "Add to Cart"
4. Loading indicator shows
5. Success! Cart badge updates
6. User can view cart
7. Cart persists even after closing app

### Complete Code Flow

```
User taps "Add to Cart"
    ↓
AddToCartController.addItem(productId)
    ↓
state = AsyncLoading (button disables)
    ↓
Reads itemQuantityControllerProvider (quantity)
    ↓
Creates Item(productId, quantity)
    ↓
Calls CartService.addItem(item)
    ↓
CartService checks auth state (guest)
    ↓
Validates product exists
    ↓
Fetches current cart from LocalCartRepository
    ↓
SembastCartRepository.fetchCart()
    ↓
Reads from Sembast database
    ↓
Deserializes JSON → Cart entity
    ↓
Returns Cart to service
    ↓
Cart.addItem(item) - immutable update
    ↓
Creates new Cart with added item
    ↓
CartService.setCart(updatedCart)
    ↓
SembastCartRepository.setCart(cart)
    ↓
Serializes Cart → JSON
    ↓
Saves to Sembast database
    ↓
Sembast emits stream event
    ↓
cartProvider detects change
    ↓
REACTIVE UPDATES:
  - cartItemsCountProvider recalculates
  - Cart badge shows new count
  - Cart screen updates
    ↓
Controller state = AsyncData (success)
    ↓
Quantity resets to 1
    ↓
Button re-enables
```

### Files Involved

| Layer | File | Role |
|-------|------|------|
| Presentation | [add_to_cart_controller.dart](../lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart) | Handles user action |
| Presentation | [item_quantity_controller.dart](../lib/src/features/cart/presentation/add_to_cart/item_quantity_controller.dart) | Manages quantity state |
| Application | [cart_service.dart](../lib/src/features/cart/application/cart_service.dart) | Orchestrates cart operations |
| Data | [sembast_cart_repository.dart](../lib/src/features/cart/data/local/sembast_cart_repository.dart) | Local persistence |
| Domain | [cart.dart](../lib/src/features/cart/domain/cart.dart) | Business entity |
| Domain | [item.dart](../lib/src/features/cart/domain/item.dart) | Cart item entity |

### Detailed Code: Add to Cart

#### 1. UI Widget

```dart
class AddToCartWidget extends ConsumerWidget {
  const AddToCartWidget({super.key, required this.product});

  final Product product;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(addToCartControllerProvider);

    ref.listen<AsyncValue<void>>(
      addToCartControllerProvider,
      (_, state) => state.showAlertDialogOnError(context),
    );

    return Column(
      children: [
        const ItemQuantitySelector(),
        const SizedBox(height: 16),
        PrimaryButton(
          text: 'Add to Cart',
          isLoading: state.isLoading,
          onPressed: state.isLoading
              ? null
              : () {
                  ref
                      .read(addToCartControllerProvider.notifier)
                      .addItem(product.id);
                },
        ),
      ],
    );
  }
}
```

#### 2. Controller

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
      ref.read(itemQuantityControllerProvider.notifier).updateQuantity(1);
    }
  }
}
```

#### 3. Service

```dart
Future<void> addItem(Item item) async {
  // Validate
  final product = await ref
      .read(productsRepositoryProvider)
      .fetchProduct(item.productId);

  if (product == null) {
    throw ProductNotFoundException();
  }

  // Get current cart (local for guest)
  final cart = await _fetchCart();

  // Apply business logic
  final updated = cart.addItem(item);

  // Persist
  await _setCart(updated);
}
```

#### 4. Repository

```dart
Future<void> setCart(Cart cart) async {
  final cartJson = json.encode(cart.toJson());
  await store.record(cartItemsKey).put(db, cartJson);
}

Stream<Cart> watchCart() {
  return store.record(cartItemsKey).onSnapshot(db).map((snapshot) {
    if (snapshot != null && snapshot.value != null) {
      return Cart.fromJson(json.decode(snapshot.value as String));
    }
    return const Cart();
  });
}
```

### Viewing the Cart

#### Cart Screen

**File:** [lib/src/features/cart/presentation/shopping_cart/shopping_cart_screen.dart](../lib/src/features/cart/presentation/shopping_cart/shopping_cart_screen.dart)

```dart
class ShoppingCartScreen extends ConsumerWidget {
  const ShoppingCartScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final cartValue = ref.watch(cartProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Shopping Cart')),
      body: AsyncValueWidget<Cart>(
        value: cartValue,
        data: (cart) => cart.items.isEmpty
            ? const EmptyPlaceholderWidget(
                message: 'Your cart is empty',
              )
            : Column(
                children: [
                  Expanded(child: CartItemsList(cart: cart)),
                  CartTotal(cart: cart),
                  CheckoutButton(),
                ],
              ),
      ),
    );
  }
}
```

### Key Concepts Demonstrated

✅ **Local persistence**: Sembast for guest cart
✅ **Immutable updates**: Cart.addItem returns new cart
✅ **Reactive UI**: Badge updates automatically
✅ **AsyncValue**: Loading/error/success states
✅ **Validation**: Product exists check
✅ **Multi-layer flow**: Presentation → Application → Data → Domain

---

## Feature 4: User Authentication

### User Story
> As a user, I want to sign in with email and password so I can access my account and order history.

### UI Flow

1. User taps "Account" (redirects to sign-in if not logged in)
2. User enters email and password
3. User taps "Sign In"
4. Loading indicator shows
5. On success: navigates to account screen
6. On error: shows error dialog

### Code Flow

```
User taps "Sign In"
    ↓
Form validates input
    ↓
EmailPasswordSignInController.submit()
    ↓
state = AsyncLoading
    ↓
Calls AuthRepository.signInWithEmailAndPassword()
    ↓
FakeAuthRepository simulates network delay
    ↓
Validates password (demo: "wrong" throws error)
    ↓
Creates AppUser(uid, email)
    ↓
Updates _authState (InMemoryStore)
    ↓
Stream emits new user
    ↓
authStateChangesProvider notifies listeners
    ↓
REACTIVE EFFECTS:
  1. GoRouter detects auth change
  2. Redirects to home page
  3. CartSyncService triggers
  4. Navigation menu updates
    ↓
Controller state = AsyncData
    ↓
Form re-enables
```

### Detailed Code

#### 1. Sign-In Screen

**File:** [lib/src/features/authentication/presentation/sign_in/email_password_sign_in_screen.dart](../lib/src/features/authentication/presentation/sign_in/email_password_sign_in_screen.dart)

```dart
Future<void> _submit() async {
  if (_formKey.currentState!.validate()) {
    await ref
        .read(emailPasswordSignInControllerProvider.notifier)
        .submit(
          email: _emailController.text,
          password: _passwordController.text,
          formType: _formType,
        );
  }
}
```

#### 2. Controller

```dart
@riverpod
class EmailPasswordSignInController extends _$EmailPasswordSignInController {
  @override
  FutureOr<void> build() {}

  Future<void> submit({
    required String email,
    required String password,
    required EmailPasswordSignInFormType formType,
  }) async {
    state = const AsyncLoading();

    final authRepository = ref.read(authRepositoryProvider);

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

#### 3. Fake Auth Repository

```dart
Future<void> signInWithEmailAndPassword(
  String email,
  String password,
) async {
  await Future.delayed(const Duration(seconds: 1));

  if (password == 'wrong') {
    throw WrongPasswordException();
  }

  _authState.value = AppUser(
    uid: email.split('@')[0],
    email: email,
  );
}
```

#### 4. Auth State Provider

```dart
@Riverpod(keepAlive: true)
Stream<AppUser?> authStateChanges(Ref ref) {
  final authRepository = ref.watch(authRepositoryProvider);
  return authRepository.authStateChanges();
}
```

### Router Integration

**File:** [lib/src/routing/app_router.dart](../lib/src/routing/app_router.dart)

```dart
@Riverpod(keepAlive: true)
GoRouter goRouter(Ref ref) {
  final authRepository = ref.watch(authRepositoryProvider);

  return GoRouter(
    refreshListenable: GoRouterRefreshStream(
      authRepository.authStateChanges(),
    ),
    redirect: (context, state) {
      final isLoggedIn = authRepository.currentUser != null;
      final path = state.uri.path;

      if (isLoggedIn) {
        if (path == '/signIn') {
          return '/';  // Already logged in, go home
        }
      } else {
        if (path == '/account' || path == '/orders') {
          return '/';  // Not logged in, can't access
        }
      }
      return null;
    },
    routes: [...],
  );
}
```

### Key Concepts Demonstrated

✅ **Form validation**: Client-side checks
✅ **AsyncNotifier**: Async controller pattern
✅ **Error handling**: Custom exceptions
✅ **Reactive routing**: Auto-redirect on auth change
✅ **Stream providers**: Auth state stream
✅ **Route guards**: Protect authenticated routes

---

## Feature 5: Cart Sync on Sign-In

### User Story
> As a user, when I sign in, I want my guest cart items to merge with my remote cart so I don't lose anything.

### UI Flow

1. Guest user adds items to cart (saved locally)
2. User signs in
3. **Automatically** (no user action):
   - Local cart items merge with remote cart
   - Inventory constraints respected
   - Local cart cleared
   - Cart icon updates

### Code Flow

```
User signs in
    ↓
AuthRepository updates _authState
    ↓
authStateChanges stream emits new user
    ↓
CartSyncService is listening
    ↓
Detects: previousUser = null, newUser = User
    ↓
Triggers _moveItemsToRemoteCart(uid)
    ↓
Fetches local cart
    ↓
Fetches remote cart
    ↓
For each local item:
  - Get remote quantity
  - Fetch product for inventory check
  - Calculate: min(local + remote, available)
  - Add to merge list
    ↓
Update remote cart with merged items
    ↓
Clear local cart
    ↓
cartProvider switches to remote
    ↓
UI updates with merged cart
```

### Detailed Code

**File:** [lib/src/features/cart/application/cart_sync_service.dart](../lib/src/features/cart/application/cart_sync_service.dart)

```dart
class CartSyncService {
  CartSyncService(this.ref) {
    _init();
  }

  final Ref ref;

  void _init() {
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

      // 1. Get both carts
      final localCart = await localCartRepository.fetchCart();
      final remoteCart = await remoteCartRepository.fetchCart(uid);

      // 2. Calculate items to add
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

      // 3. Merge carts
      final updatedRemoteCart = remoteCart.addItems(localItemsToAdd);
      await remoteCartRepository.setCart(uid, updatedRemoteCart);

      // 4. Clear local cart
      await localCartRepository.setCart(const Cart());
    } catch (e) {
      ref.read(errorLoggerProvider).logError(e, StackTrace.current);
    }
  }
}
```

### Cart Provider Switching

```dart
@riverpod
Stream<Cart> cart(Ref ref) {
  final user = ref.watch(authStateChangesProvider).value;

  if (user != null) {
    // Signed in: watch remote cart
    return ref.watch(remoteCartRepositoryProvider).watchCart(user.uid);
  } else {
    // Guest: watch local cart
    return ref.watch(localCartRepositoryProvider).watchCart();
  }
}
```

### Example Scenario

**Before Sign-In:**
- Local cart: `{product-1: 2, product-2: 1}`
- Remote cart: `{product-1: 1, product-3: 3}`

**Merge Logic:**
- product-1: local=2, remote=1, available=10 → add 2 to remote (total: 3)
- product-2: local=1, remote=0, available=5 → add 1 to remote (total: 1)
- product-3: already in remote (total: 3)

**After Sign-In:**
- Local cart: `{}` (cleared)
- Remote cart: `{product-1: 3, product-2: 1, product-3: 3}`

### Key Concepts Demonstrated

✅ **Side effect service**: Automatic background operation
✅ **Provider listening**: `ref.listen` for events
✅ **Complex orchestration**: 3 repositories coordinated
✅ **Business rules**: Inventory constraints
✅ **Error handling**: Logs but doesn't throw
✅ **Reactive switching**: Cart provider changes data source

---

## Feature 6: Checkout Process

### User Story
> As a signed-in user, I want to complete my purchase and create an order.

### UI Flow

1. User views cart
2. User taps "Checkout"
3. Checkout screen shows cart summary and total
4. User taps "Place Order"
5. Loading indicator shows
6. On success: navigates to orders screen
7. Cart is cleared

### Code Flow

```
User taps "Place Order"
    ↓
PaymentButtonController.pay()
    ↓
state = AsyncLoading
    ↓
Calls CheckoutService.placeOrder()
    ↓
Service validates user is signed in
    ↓
Fetches cart (must not be empty)
    ↓
Validates inventory for all items
    ↓
Calculates total price
    ↓
Creates Order entity
    ↓
Saves order to OrdersRepository
    ↓
Clears cart via CartService
    ↓
Controller state = AsyncData
    ↓
UI listens to success
    ↓
Navigates to orders screen
```

### Detailed Code

#### 1. Checkout Screen

**File:** [lib/src/features/checkout/presentation/checkout_screen/checkout_screen.dart](../lib/src/features/checkout/presentation/checkout_screen/checkout_screen.dart)

```dart
class CheckoutScreen extends ConsumerWidget {
  const CheckoutScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    ref.listen<AsyncValue<void>>(
      paymentButtonControllerProvider,
      (_, state) {
        if (state.hasValue) {
          // Success: navigate to orders
          context.goNamed(AppRoute.orders.name);
        } else if (state.hasError) {
          state.showAlertDialogOnError(context);
        }
      },
    );

    final state = ref.watch(paymentButtonControllerProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Checkout')),
      body: Column(
        children: [
          Expanded(child: const CartItemsList()),
          const CartTotal(),
          Padding(
            padding: const EdgeInsets.all(16),
            child: PrimaryButton(
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
          ),
        ],
      ),
    );
  }
}
```

#### 2. Payment Controller

```dart
@riverpod
class PaymentButtonController extends _$PaymentButtonController {
  @override
  FutureOr<void> build() {}

  Future<void> pay() async {
    final checkoutService = ref.read(checkoutServiceProvider);

    state = const AsyncLoading();

    state = await AsyncValue.guard(() => checkoutService.placeOrder());
  }
}
```

#### 3. Checkout Service

```dart
class FakeCheckoutService {
  FakeCheckoutService(this.ref);
  final Ref ref;

  Future<void> placeOrder() async {
    // 1. Validate user is signed in
    final uid = ref.read(authStateChangesProvider).value?.uid;
    if (uid == null) {
      throw UserNotSignedInException();
    }

    // 2. Get and validate cart
    final cart = await ref.read(cartProvider.future);
    if (cart.items.isEmpty) {
      throw EmptyCartException();
    }

    // 3. Validate inventory
    final productsList = await ref.read(productsListProvider.future);
    for (final entry in cart.items.entries) {
      final product = productsList.firstWhere((p) => p.id == entry.key);
      if (entry.value > product.availableQuantity) {
        throw InsufficientInventoryException();
      }
    }

    // 4. Calculate total
    final total = await _computeTotal(cart, productsList);

    // 5. Create order
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

    // 6. Save order
    await ref.read(ordersRepositoryProvider).addOrder(uid, order);

    // 7. Clear cart
    await ref.read(cartServiceProvider).setCart(const Cart());
  }

  Future<double> _computeTotal(Cart cart, List<Product> products) async {
    var total = 0.0;
    for (final entry in cart.items.entries) {
      final product = products.firstWhere((p) => p.id == entry.key);
      total += product.price * entry.value;
    }
    return total;
  }
}
```

### Key Concepts Demonstrated

✅ **Multi-step transaction**: Validate → Calculate → Save → Clear
✅ **Navigation on success**: `ref.listen` for side effects
✅ **Validation chain**: User → Cart → Inventory
✅ **Entity creation**: Building complex domain objects
✅ **Service orchestration**: Multiple repositories

---

## Feature 7: Order Management

### User Story
> As a signed-in user, I want to view my order history.

### UI Flow

1. User taps "Orders" in navigation
2. Orders screen loads
3. Shows list of past orders
4. Each order shows items, date, status, total
5. Real-time updates if order status changes

### Code Flow

```
User navigates to /orders
    ↓
OrdersListScreen builds
    ↓
Checks auth state
    ↓
If not signed in: shows "Sign in" message
    ↓
If signed in: watches userOrdersProvider(uid)
    ↓
OrdersRepository.watchUserOrders(uid)
    ↓
Returns stream of orders
    ↓
AsyncValueWidget handles loading/error/data
    ↓
If empty: shows "No orders" message
    ↓
If has orders: ListView of OrderCard widgets
```

### Detailed Code

#### Orders List Screen

**File:** [lib/src/features/orders/presentation/orders_list/orders_list_screen.dart](../lib/src/features/orders/presentation/orders_list/orders_list_screen.dart)

```dart
class OrdersListScreen extends ConsumerWidget {
  const OrdersListScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final user = ref.watch(authStateChangesProvider).value;

    return Scaffold(
      appBar: AppBar(title: const Text('Your Orders')),
      body: user == null
          ? const EmptyPlaceholderWidget(
              message: 'Please sign in to view your orders',
            )
          : Consumer(
              builder: (context, ref, child) {
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
              },
            ),
    );
  }
}
```

#### Orders Provider

```dart
@riverpod
Stream<List<Order>> userOrders(Ref ref, String uid) {
  final ordersRepository = ref.watch(ordersRepositoryProvider);
  return ordersRepository.watchUserOrders(uid);
}
```

### Key Concepts Demonstrated

✅ **Auth-dependent data**: Check user before loading
✅ **Provider family**: Parameter for user ID
✅ **Empty states**: Different messages for different cases
✅ **Stream providers**: Real-time order updates

---

## Feature 8: Product Reviews

### User Story
> As a user who purchased a product, I want to leave a review so I can share my experience.

### UI Flow

1. User views product detail
2. If user purchased product: shows "Leave Review" button
3. User taps button
4. Review screen opens
5. User selects star rating (1-5)
6. User writes comment
7. User taps "Submit"
8. Loading indicator shows
9. On success: navigates back to product
10. Product rating updates

### Code Flow

```
User taps "Submit Review"
    ↓
Form validates (rating required, comment not empty)
    ↓
LeaveReviewController.submitReview()
    ↓
state = AsyncLoading
    ↓
Calls ReviewsService.submitReview()
    ↓
Service validates user is signed in
    ↓
Fetches user's orders
    ↓
Checks if product is in any order
    ↓
If not purchased: throws exception
    ↓
If purchased: creates Review entity
    ↓
Saves to ReviewsRepository
    ↓
Controller state = AsyncData
    ↓
UI navigates back
    ↓
Product screen shows new review
```

### Detailed Code

#### 1. Leave Review Screen

**File:** [lib/src/features/reviews/presentation/leave_review_screen/leave_review_screen.dart](../lib/src/features/reviews/presentation/leave_review_screen/leave_review_screen.dart)

```dart
class LeaveReviewScreen extends ConsumerStatefulWidget {
  const LeaveReviewScreen({super.key, required this.productId});

  final ProductID productId;

  @override
  ConsumerState<LeaveReviewScreen> createState() => _LeaveReviewScreenState();
}

class _LeaveReviewScreenState extends ConsumerState<LeaveReviewScreen> {
  final _formKey = GlobalKey<FormState>();
  final _commentController = TextEditingController();
  double _rating = 0;

  @override
  void dispose() {
    _commentController.dispose();
    super.dispose();
  }

  Future<void> _submit() async {
    if (_formKey.currentState!.validate() && _rating > 0) {
      await ref
          .read(leaveReviewControllerProvider.notifier)
          .submitReview(
            productId: widget.productId,
            rating: _rating,
            comment: _commentController.text,
          );
    }
  }

  @override
  Widget build(BuildContext context) {
    ref.listen<AsyncValue<void>>(
      leaveReviewControllerProvider,
      (_, state) {
        if (state.hasValue) {
          context.pop();
        } else if (state.hasError) {
          state.showAlertDialogOnError(context);
        }
      },
    );

    final state = ref.watch(leaveReviewControllerProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Leave a Review')),
      body: SingleChildScrollView(
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: Form(
            key: _formKey,
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.stretch,
              children: [
                Text('Rating', style: Theme.of(context).textTheme.titleLarge),
                const SizedBox(height: 8),
                RatingBar.builder(
                  initialRating: _rating,
                  minRating: 1,
                  itemCount: 5,
                  itemBuilder: (context, _) => const Icon(
                    Icons.star,
                    color: Colors.amber,
                  ),
                  onRatingUpdate: (rating) {
                    setState(() {
                      _rating = rating;
                    });
                  },
                ),
                const SizedBox(height: 24),
                TextFormField(
                  controller: _commentController,
                  decoration: const InputDecoration(
                    labelText: 'Your Review',
                    border: OutlineInputBorder(),
                  ),
                  maxLines: 5,
                  validator: (value) {
                    if (value == null || value.isEmpty) {
                      return 'Please enter your review';
                    }
                    return null;
                  },
                ),
                const SizedBox(height: 24),
                PrimaryButton(
                  text: 'Submit Review',
                  isLoading: state.isLoading,
                  onPressed: state.isLoading ? null : _submit,
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}
```

#### 2. Leave Review Controller

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

#### 3. Reviews Service (with validation)

```dart
class ReviewsService {
  ReviewsService(this.ref);
  final Ref ref;

  Future<void> submitReview({
    required ProductID productId,
    required Review review,
  }) async {
    // 1. Validate user is signed in
    final user = ref.read(authStateChangesProvider).value;
    if (user == null) {
      throw UserNotSignedInException();
    }

    // 2. Validate user purchased this product
    final orders = await ref.read(userOrdersFutureProvider(user.uid).future);

    final purchasedProductIds = orders
        .expand((order) => order.items.keys)
        .toSet();

    if (!purchasedProductIds.contains(productId)) {
      throw ProductNotPurchasedException();
    }

    // 3. Submit review
    await ref
        .read(reviewsRepositoryProvider)
        .setReview(productId, user.uid, review);
  }
}
```

### Key Concepts Demonstrated

✅ **Business validation**: Must purchase before reviewing
✅ **Form validation**: Rating and comment required
✅ **Navigation on success**: Pop back to previous screen
✅ **Third-party widgets**: RatingBar integration
✅ **Service layer validation**: Enforce business rules

---

## Complete Purchase Flow

Let's trace the **entire journey** from browsing to completed order.

### Step-by-Step Flow

```
1. Browse Products
   User opens app → Products list displays

2. View Product Detail
   User taps product → Product screen shows

3. Add to Cart (Guest)
   User adds item → Saves to local cart

4. Continue Shopping
   User browses more → Adds more items

5. Sign In
   User taps Account → Redirects to sign-in
   User signs in → Auto-redirects to home
   Cart sync runs → Merges local + remote

6. View Cart
   User taps cart icon → Shows merged cart

7. Checkout
   User taps Checkout → Checkout screen shows
   User taps Place Order → Creates order

8. View Orders
   Auto-navigates to orders → Shows new order

9. Leave Review
   User views product → Taps Leave Review
   User submits review → Review saved

10. See Updated Rating
    Product page → Shows new average rating
```

### Data Flow Summary

```
UI (Presentation)
    ↓ user actions
Controllers (Presentation)
    ↓ calls
Services (Application)
    ↓ coordinates
Repositories (Data)
    ↓ persistence
Sembast/InMemory (Data Source)
    ↑ streams
Repositories
    ↑ emits
Providers (Riverpod)
    ↑ notifies
UI rebuilds
```

---

## Edge Cases and Error Handling

### Case 1: Product Not Found

```dart
// User navigates to /product/invalid-id
Stream<Product?> watchProduct(String id) {
  return watchProductsList().map((products) {
    try {
      return products.firstWhere((p) => p.id == id);
    } catch (e) {
      return null;  // Not found
    }
  });
}

// UI handles null
AsyncValueWidget<Product?>(
  value: productValue,
  data: (product) => product == null
      ? EmptyPlaceholderWidget(message: 'Product not found')
      : ProductDetails(product: product),
)
```

### Case 2: Empty Cart Checkout

```dart
// Service validation
Future<void> placeOrder() async {
  final cart = await ref.read(cartProvider.future);

  if (cart.items.isEmpty) {
    throw EmptyCartException();
  }
  // Continue...
}

// UI shows error dialog
ref.listen<AsyncValue<void>>(
  paymentButtonControllerProvider,
  (_, state) => state.showAlertDialogOnError(context),
);
```

### Case 3: Insufficient Inventory

```dart
// Validation in service
for (final entry in cart.items.entries) {
  final product = productsList.firstWhere((p) => p.id == entry.key);

  if (entry.value > product.availableQuantity) {
    throw InsufficientInventoryException();
  }
}
```

### Case 4: Review Without Purchase

```dart
// Service validates
final purchasedProductIds = orders
    .expand((order) => order.items.keys)
    .toSet();

if (!purchasedProductIds.contains(productId)) {
  throw ProductNotPurchasedException();
}

// Custom error message shown
class ProductNotPurchasedException extends AppException {
  ProductNotPurchasedException()
      : super('You can only review products you have purchased');
}
```

### Case 5: Network/Database Errors

```dart
// AsyncValue.guard catches all errors
state = await AsyncValue.guard(() async {
  await riskyOperation();
});

// UI shows appropriate error
AsyncValueWidget handles error state
```

---

## Summary

This walkthrough demonstrated:

✅ **Complete user journeys** through all major features
✅ **Data flow** from UI to persistence and back
✅ **Layer interaction** - how all layers work together
✅ **Riverpod patterns** - providers, controllers, services
✅ **Error handling** - validation and error states
✅ **Side effects** - automatic cart sync
✅ **Reactive updates** - UI auto-updates
✅ **Business rules** - enforced in services

### Key Takeaways

1. **Separation of Concerns**: Each layer has clear responsibilities
2. **Reactive Architecture**: Changes propagate automatically
3. **Type Safety**: Compile-time error checking
4. **Testability**: Each layer can be tested independently
5. **User Experience**: Loading states, error handling, validation
6. **Business Logic**: Enforced in services, not UI
7. **Data Persistence**: Local (Sembast) and fake remote

---

## What's Next?

Ready for the final document? Let's dive into **routing, testing, and best practices**!

Head to **[08-ROUTING-TESTING-BEST-PRACTICES.md](08-ROUTING-TESTING-BEST-PRACTICES.md)** for:
- GoRouter deep dive
- Testing strategies (unit, widget, integration)
- Robot pattern for tests
- Performance optimization
- Production deployment tips
- And more!

This is the last document - let's finish strong! 🎯
