# Application and Presentation Layers - Business Logic Meets UI

## Table of Contents
- [Introduction](#introduction)
- [Application Layer Deep Dive](#application-layer-deep-dive)
- [Service Patterns](#service-patterns)
- [Use Case Examples](#use-case-examples)
- [Presentation Layer Deep Dive](#presentation-layer-deep-dive)
- [Controller Pattern](#controller-pattern)
- [Widget Organization](#widget-organization)
- [Common Widgets Library](#common-widgets-library)
- [Responsive Design](#responsive-design)
- [Form Handling](#form-handling)
- [Navigation Integration](#navigation-integration)
- [Error Handling in UI](#error-handling-in-ui)
- [Loading States](#loading-states)
- [Best Practices](#best-practices)

---

## Introduction

The **Application** and **Presentation** layers are where:
- **Business logic is orchestrated** (Application)
- **Users interact with the app** (Presentation)

### Layer Responsibilities

```
┌────────────────────────────────────────┐
│    PRESENTATION LAYER                  │
│  - Screens and Widgets                 │
│  - Controllers (UI state)              │
│  - User input handling                 │
│  - Visual presentation                 │
└────────────────────────────────────────┘
              ↕ (calls)
┌────────────────────────────────────────┐
│    APPLICATION LAYER                   │
│  - Services (business logic)           │
│  - Use cases                           │
│  - Orchestration                       │
│  - Validation                          │
└────────────────────────────────────────┘
              ↕ (calls)
┌────────────────────────────────────────┐
│    DATA LAYER                          │
│  - Repositories                        │
│  - Data sources                        │
└────────────────────────────────────────┘
```

---

## Application Layer Deep Dive

The application layer contains **business logic services** that coordinate between repositories and handle complex operations.

### When to Create a Service?

Create a service when you need to:
✅ Coordinate **multiple repositories**
✅ Implement **complex business logic**
✅ Perform **multi-step operations**
✅ **Validate business rules**
✅ Handle **side effects** (like syncing data)

### Service Structure

```dart
class MyService {
  MyService(this.ref);
  final Ref ref;

  // Private helper methods
  Future<Data> _fetchData() { }

  // Public API methods
  Future<void> performAction() { }
}

// Provider
@riverpod
MyService myService(Ref ref) {
  return MyService(ref);
}
```

---

## Service Patterns

### Pattern 1: Coordination Service

**Example: Cart Service**

**File:** [lib/src/features/cart/application/cart_service.dart](../lib/src/features/cart/application/cart_service.dart)

```dart
/// Service that coordinates cart operations between local and remote storage
class CartService {
  CartService(this.ref);
  final Ref ref;

  /// Determines which repository to use based on auth state
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

  /// Saves to appropriate repository
  Future<void> _setCart(Cart cart) async {
    final user = ref.read(authRepositoryProvider).currentUser;
    if (user != null) {
      await ref.read(remoteCartRepositoryProvider).setCart(user.uid, cart);
    } else {
      await ref.read(localCartRepositoryProvider).setCart(cart);
    }
  }

  /// Add item with validation
  Future<void> addItem(Item item) async {
    // 1. Validate product exists
    final product = await ref
        .read(productsRepositoryProvider)
        .fetchProduct(item.productId);

    if (product == null) {
      throw ProductNotFoundException();
    }

    // 2. Get current cart
    final cart = await _fetchCart();

    // 3. Apply business logic
    final updated = cart.addItem(item);

    // 4. Persist
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

@riverpod
CartService cartService(Ref ref) {
  return CartService(ref);
}
```

**Service Benefits:**
- ✅ Hides complexity of local vs remote switching
- ✅ Validates business rules (product exists)
- ✅ Single source of truth for cart operations
- ✅ Easy to test (mock repositories)

### Pattern 2: Side Effect Service

**Example: Cart Sync Service**

**File:** [lib/src/features/cart/application/cart_sync_service.dart](../lib/src/features/cart/application/cart_sync_service.dart)

```dart
/// Service that automatically syncs local cart to remote on sign-in
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

  /// Complex orchestration: merge local and remote carts
  Future<void> _moveItemsToRemoteCart(String uid) async {
    try {
      final localCartRepository = ref.read(localCartRepositoryProvider);
      final remoteCartRepository = ref.read(remoteCartRepositoryProvider);
      final productsRepository = ref.read(productsRepositoryProvider);

      // 1. Fetch both carts
      final localCart = await localCartRepository.fetchCart();
      final remoteCart = await remoteCartRepository.fetchCart(uid);

      // 2. Calculate items to add (with inventory checks)
      final localItemsToAdd = <Item>[];

      for (final entry in localCart.items.entries) {
        final productId = entry.key;
        final localQuantity = entry.value;
        final remoteQuantity = remoteCart.items[productId] ?? 0;

        // Fetch product for inventory check
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
      // Log error but don't throw (background operation)
      ref.read(errorLoggerProvider).logError(e, StackTrace.current);
    }
  }
}

@Riverpod(keepAlive: true)
CartSyncService cartSyncService(Ref ref) {
  return CartSyncService(ref);
}
```

**Initialization:**

```dart
// In main.dart
void main() async {
  final container = ProviderContainer(/* ... */);

  // Initialize service (starts listening for auth changes)
  container.read(cartSyncServiceProvider);

  runApp(/* ... */);
}
```

**Service Characteristics:**
- ✅ Runs in background
- ✅ Triggered by events (auth changes)
- ✅ Complex orchestration (3 repositories)
- ✅ Error handling (logs but doesn't throw)
- ✅ Respects inventory constraints

### Pattern 3: Validation Service

**Example: Reviews Service**

**File:** [lib/src/features/reviews/application/reviews_service.dart](../lib/src/features/reviews/application/reviews_service.dart)

```dart
/// Service with business validation logic
class ReviewsService {
  ReviewsService(this.ref);
  final Ref ref;

  Future<void> submitReview({
    required ProductID productId,
    required Review review,
  }) async {
    // 1. Validate user is authenticated
    final user = ref.read(authStateChangesProvider).value;
    if (user == null) {
      throw UserNotSignedInException();
    }

    // 2. Validate user purchased the product
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

@riverpod
ReviewsService reviewsService(Ref ref) {
  return ReviewsService(ref);
}
```

**Business Rules Enforced:**
- ✅ Must be signed in
- ✅ Must have purchased the product
- ✅ Prevents fake reviews

### Pattern 4: Transaction Service

**Example: Checkout Service**

**File:** [lib/src/features/checkout/application/fake_checkout_service.dart](../lib/src/features/checkout/application/fake_checkout_service.dart)

```dart
/// Service that orchestrates the checkout process
class FakeCheckoutService {
  FakeCheckoutService(this.ref);
  final Ref ref;

  /// Multi-step transaction
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

    // 3. Validate inventory (all products available)
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

@riverpod
FakeCheckoutService checkoutService(Ref ref) {
  return FakeCheckoutService(ref);
}
```

**Transaction Pattern:**
- ✅ Multi-step operation
- ✅ Validation at each step
- ✅ Atomicity (all or nothing)
- ✅ Multiple repository coordination

---

## Use Case Examples

Services can be organized around **use cases** - specific user actions.

### Use Case Structure

```dart
/// Use case: Add product to cart
class AddProductToCartUseCase {
  AddProductToCartUseCase({
    required this.productsRepository,
    required this.cartService,
  });

  final ProductsRepository productsRepository;
  final CartService cartService;

  Future<void> execute({
    required ProductID productId,
    required int quantity,
  }) async {
    // Validation
    final product = await productsRepository.fetchProduct(productId);
    if (product == null) {
      throw ProductNotFoundException();
    }

    if (quantity > product.availableQuantity) {
      throw InsufficientInventoryException();
    }

    // Execute
    await cartService.addItem(Item(
      productId: productId,
      quantity: quantity,
    ));
  }
}
```

**Benefits:**
- ✅ Single Responsibility Principle
- ✅ Clear intent (use case name)
- ✅ Easy to test
- ✅ Reusable across UI

**This project uses services instead of separate use cases**, but the pattern is similar.

---

## Presentation Layer Deep Dive

The presentation layer is everything the **user sees and interacts with**.

### Organization

```
features/[feature]/presentation/
├── [screen_name]/
│   ├── screen.dart              # Main screen widget
│   ├── controller.dart          # Screen state controller
│   └── widgets/                 # Screen-specific widgets
│       ├── widget1.dart
│       └── widget2.dart
```

### Example: Products Feature

```
features/products/presentation/
├── products_list/
│   ├── products_list_screen.dart
│   ├── products_search_bar.dart
│   └── products_layout_grid.dart
├── product_screen/
│   ├── product_screen.dart
│   ├── product_average_rating.dart
│   └── leave_review_action.dart
└── home_app_bar/
    ├── home_app_bar.dart
    └── shopping_cart_icon.dart
```

---

## Controller Pattern

Controllers manage **UI state** and handle **user actions**.

### Controller Responsibilities

✅ Handle user input
✅ Call services
✅ Manage async states (loading, error, success)
✅ Update UI state
✅ Navigate on success

### AsyncNotifier Pattern

**File:** [lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart](../lib/src/features/cart/presentation/add_to_cart/add_to_cart_controller.dart)

```dart
@riverpod
class AddToCartController extends _$AddToCartController {
  @override
  FutureOr<void> build() {
    // Initial state: ready (not loading)
  }

  Future<void> addItem(ProductID productId) async {
    // Get dependencies
    final cartService = ref.read(cartServiceProvider);
    final quantity = ref.read(itemQuantityControllerProvider);

    final item = Item(productId: productId, quantity: quantity);

    // Set loading state
    state = const AsyncLoading<void>();

    // Execute with error handling
    state = await AsyncValue.guard(() => cartService.addItem(item));

    // Success: reset quantity
    if (!state.hasError) {
      ref.read(itemQuantityControllerProvider.notifier).updateQuantity(1);
    }
  }
}
```

**UI Integration:**

```dart
class AddToCartButton extends ConsumerWidget {
  const AddToCartButton({super.key, required this.productId});

  final ProductID productId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Watch state for loading indicator
    final state = ref.watch(addToCartControllerProvider);

    // Listen for errors
    ref.listen<AsyncValue<void>>(
      addToCartControllerProvider,
      (_, state) => state.showAlertDialogOnError(context),
    );

    return PrimaryButton(
      text: 'Add to Cart',
      isLoading: state.isLoading,
      onPressed: state.isLoading
          ? null
          : () => ref
              .read(addToCartControllerProvider.notifier)
              .addItem(productId),
    );
  }
}
```

### Notifier Pattern (Synchronous State)

**File:** [lib/src/features/cart/presentation/add_to_cart/item_quantity_controller.dart](../lib/src/features/cart/presentation/add_to_cart/item_quantity_controller.dart)

```dart
@riverpod
class ItemQuantityController extends _$ItemQuantityController {
  @override
  int build() => 1;  // Initial quantity

  void increment() {
    state = min(state + 1, 99);  // Cap at 99
  }

  void decrement() {
    state = max(state - 1, 1);  // Minimum 1
  }

  void updateQuantity(int quantity) {
    state = quantity.clamp(1, 99);
  }
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
          icon: const Icon(Icons.remove),
          onPressed: quantity > 1
              ? () => ref
                  .read(itemQuantityControllerProvider.notifier)
                  .decrement()
              : null,
        ),
        Text(
          '$quantity',
          style: Theme.of(context).textTheme.headlineSmall,
        ),
        IconButton(
          icon: const Icon(Icons.add),
          onPressed: quantity < 99
              ? () => ref
                  .read(itemQuantityControllerProvider.notifier)
                  .increment()
              : null,
        ),
      ],
    );
  }
}
```

### Form Controller Pattern

**File:** [lib/src/features/authentication/presentation/sign_in/email_password_sign_in_controller.dart](../lib/src/features/authentication/presentation/sign_in/email_password_sign_in_controller.dart)

```dart
@riverpod
class EmailPasswordSignInController extends _$EmailPasswordSignInController {
  @override
  FutureOr<void> build() {
    // Initial state
  }

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

---

## Widget Organization

### Screen Widget Pattern

**File:** [lib/src/features/products/presentation/products_list/products_list_screen.dart](../lib/src/features/products/presentation/products_list/products_list_screen.dart)

```dart
/// Main screen widget
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

**Pattern:**
- ✅ Scaffold provides structure
- ✅ CustomScrollView for complex scrolling
- ✅ Consumer for reactive data
- ✅ Composition of smaller widgets

### Product Card Widget

**File:** [lib/src/features/products/presentation/product_card.dart](../lib/src/features/products/presentation/product_card.dart)

```dart
class ProductCard extends StatelessWidget {
  const ProductCard({super.key, required this.product});

  final Product product;

  @override
  Widget build(BuildContext context) {
    return Card(
      elevation: 2,
      child: InkWell(
        onTap: () => context.goNamed(
          AppRoute.product.name,
          pathParameters: {'id': product.id},
        ),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            // Product image
            Expanded(
              child: Image.asset(
                product.imageUrl,
                fit: BoxFit.cover,
              ),
            ),
            // Product info
            Padding(
              padding: const EdgeInsets.all(8.0),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    product.title,
                    style: Theme.of(context).textTheme.titleMedium,
                  ),
                  const SizedBox(height: 4),
                  Text(
                    formatCurrency(product.price),
                    style: Theme.of(context).textTheme.titleSmall,
                  ),
                  const SizedBox(height: 4),
                  ProductAverageRating(product: product),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

**Widget Best Practices:**
- ✅ Single responsibility (displays one product)
- ✅ Stateless (no local state)
- ✅ Composition (uses smaller widgets)
- ✅ Const constructor (performance)

### Product Detail Screen

**File:** [lib/src/features/products/presentation/product_screen/product_screen.dart](../lib/src/features/products/presentation/product_screen/product_screen.dart)

```dart
class ProductScreen extends StatelessWidget {
  const ProductScreen({super.key, required this.productId});

  final ProductID productId;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        actions: const [ShoppingCartIcon()],
      ),
      body: Consumer(
        builder: (context, ref, child) {
          final productValue = ref.watch(productProvider(productId));

          return AsyncValueWidget<Product?>(
            value: productValue,
            data: (product) => product == null
                ? EmptyPlaceholderWidget(
                    message: 'Product not found',
                  )
                : CustomScrollView(
                    slivers: [
                      ResponsiveSliverCenter(
                        padding: const EdgeInsets.all(16),
                        child: ProductDetails(product: product),
                      ),
                    ],
                  ),
          );
        },
      ),
    );
  }
}
```

---

## Common Widgets Library

Reusable UI components shared across features.

### AsyncValueWidget

**File:** [lib/src/common_widgets/async_value_widget.dart](../lib/src/common_widgets/async_value_widget.dart)

```dart
/// Reusable widget for rendering AsyncValue states
class AsyncValueWidget<T> extends StatelessWidget {
  const AsyncValueWidget({
    super.key,
    required this.value,
    required this.data,
  });

  final AsyncValue<T> value;
  final Widget Function(T) data;

  @override
  Widget build(BuildContext context) {
    return value.when(
      data: data,
      loading: () => const Center(
        child: CircularProgressIndicator(),
      ),
      error: (e, st) => Center(
        child: ErrorMessageWidget(e.toString()),
      ),
    );
  }
}
```

**Usage:**
```dart
AsyncValueWidget<List<Product>>(
  value: ref.watch(productsListProvider),
  data: (products) => ProductList(products: products),
)
```

### PrimaryButton

**File:** [lib/src/common_widgets/primary_button.dart](../lib/src/common_widgets/primary_button.dart)

```dart
class PrimaryButton extends StatelessWidget {
  const PrimaryButton({
    super.key,
    required this.text,
    this.isLoading = false,
    this.onPressed,
  });

  final String text;
  final bool isLoading;
  final VoidCallback? onPressed;

  @override
  Widget build(BuildContext context) {
    return SizedBox(
      height: 48,
      child: ElevatedButton(
        onPressed: isLoading ? null : onPressed,
        child: isLoading
            ? const CircularProgressIndicator()
            : Text(
                text,
                style: const TextStyle(fontSize: 16),
              ),
      ),
    );
  }
}
```

### ResponsiveCenter

**File:** [lib/src/common_widgets/responsive_center.dart](../lib/src/common_widgets/responsive_center.dart)

```dart
/// Centers content on large screens, full width on small screens
class ResponsiveCenter extends StatelessWidget {
  const ResponsiveCenter({
    super.key,
    this.maxContentWidth = Breakpoint.tablet,
    this.padding = EdgeInsets.zero,
    required this.child,
  });

  final double maxContentWidth;
  final EdgeInsetsGeometry padding;
  final Widget child;

  @override
  Widget build(BuildContext context) {
    return Center(
      child: SizedBox(
        width: min(MediaQuery.of(context).size.width, maxContentWidth),
        child: Padding(
          padding: padding,
          child: child,
        ),
      ),
    );
  }
}
```

### EmptyPlaceholderWidget

**File:** [lib/src/common_widgets/empty_placeholder_widget.dart](../lib/src/common_widgets/empty_placeholder_widget.dart)

```dart
class EmptyPlaceholderWidget extends StatelessWidget {
  const EmptyPlaceholderWidget({
    super.key,
    required this.message,
  });

  final String message;

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Text(
          message,
          style: Theme.of(context).textTheme.headlineSmall,
          textAlign: TextAlign.center,
        ),
      ),
    );
  }
}
```

---

## Responsive Design

### Breakpoints

**File:** [lib/src/constants/breakpoints.dart](../lib/src/constants/breakpoints.dart)

```dart
class Breakpoint {
  static const double desktop = 900;
  static const double tablet = 600;
}
```

### Responsive Grid

**File:** [lib/src/features/products/presentation/products_list/products_layout_grid.dart](../lib/src/features/products/presentation/products_list/products_layout_grid.dart)

```dart
class ProductsLayoutGrid extends StatelessWidget {
  const ProductsLayoutGrid({
    super.key,
    required this.itemCount,
    required this.itemBuilder,
  });

  final int itemCount;
  final Widget Function(BuildContext, int) itemBuilder;

  @override
  Widget build(BuildContext context) {
    return SliverPadding(
      padding: const EdgeInsets.all(16),
      sliver: SliverLayoutGrid(
        gridDelegate: SliverGridDelegateWithMaxCrossAxisExtent(
          maxCrossAxisExtent: 200,  // Max width per item
          mainAxisSpacing: 16,
          crossAxisSpacing: 16,
          childAspectRatio: 0.75,
        ),
        delegate: SliverChildBuilderDelegate(
          itemBuilder,
          childCount: itemCount,
        ),
      ),
    );
  }
}
```

**Responsive Behavior:**
- Phone: 2 columns
- Tablet: 3-4 columns
- Desktop: 4-5 columns

### Two-Column Layout

**File:** [lib/src/common_widgets/responsive_two_column_layout.dart](../lib/src/common_widgets/responsive_two_column_layout.dart)

```dart
class ResponsiveTwoColumnLayout extends StatelessWidget {
  const ResponsiveTwoColumnLayout({
    super.key,
    required this.startContent,
    required this.endContent,
    this.breakpoint = Breakpoint.tablet,
  });

  final Widget startContent;
  final Widget endContent;
  final double breakpoint;

  @override
  Widget build(BuildContext context) {
    final screenWidth = MediaQuery.of(context).size.width;

    if (screenWidth >= breakpoint) {
      // Desktop: side by side
      return Row(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Flexible(flex: 1, child: startContent),
          const SizedBox(width: 16),
          Flexible(flex: 1, child: endContent),
        ],
      );
    } else {
      // Mobile: stacked
      return Column(
        children: [
          startContent,
          const SizedBox(height: 16),
          endContent,
        ],
      );
    }
  }
}
```

---

## Form Handling

### Email/Password Sign-In Form

**File:** [lib/src/features/authentication/presentation/sign_in/email_password_sign_in_screen.dart](../lib/src/features/authentication/presentation/sign_in/email_password_sign_in_screen.dart)

```dart
class EmailPasswordSignInScreen extends ConsumerStatefulWidget {
  const EmailPasswordSignInScreen({super.key});

  @override
  ConsumerState<EmailPasswordSignInScreen> createState() =>
      _EmailPasswordSignInScreenState();
}

class _EmailPasswordSignInScreenState
    extends ConsumerState<EmailPasswordSignInScreen> {
  final _formKey = GlobalKey<FormState>();
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();

  var _formType = EmailPasswordSignInFormType.signIn;

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

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

  @override
  Widget build(BuildContext context) {
    ref.listen<AsyncValue<void>>(
      emailPasswordSignInControllerProvider,
      (_, state) => state.showAlertDialogOnError(context),
    );

    final state = ref.watch(emailPasswordSignInControllerProvider);

    return Scaffold(
      appBar: AppBar(
        title: Text(_formType == EmailPasswordSignInFormType.signIn
            ? 'Sign In'
            : 'Register'),
      ),
      body: SingleChildScrollView(
        child: ResponsiveCenter(
          maxContentWidth: Breakpoint.tablet,
          padding: const EdgeInsets.all(16),
          child: Form(
            key: _formKey,
            child: Column(
              children: [
                // Email field
                TextFormField(
                  controller: _emailController,
                  decoration: const InputDecoration(labelText: 'Email'),
                  keyboardType: TextInputType.emailAddress,
                  validator: (value) {
                    if (value == null || value.isEmpty) {
                      return 'Email is required';
                    }
                    return null;
                  },
                ),
                const SizedBox(height: 16),

                // Password field
                TextFormField(
                  controller: _passwordController,
                  decoration: const InputDecoration(labelText: 'Password'),
                  obscureText: true,
                  validator: (value) {
                    if (value == null || value.isEmpty) {
                      return 'Password is required';
                    }
                    if (value.length < 6) {
                      return 'Password must be at least 6 characters';
                    }
                    return null;
                  },
                ),
                const SizedBox(height: 24),

                // Submit button
                PrimaryButton(
                  text: _formType == EmailPasswordSignInFormType.signIn
                      ? 'Sign In'
                      : 'Register',
                  isLoading: state.isLoading,
                  onPressed: state.isLoading ? null : _submit,
                ),
                const SizedBox(height: 16),

                // Toggle form type
                TextButton(
                  onPressed: () {
                    setState(() {
                      _formType = _formType == EmailPasswordSignInFormType.signIn
                          ? EmailPasswordSignInFormType.register
                          : EmailPasswordSignInFormType.signIn;
                    });
                  },
                  child: Text(
                    _formType == EmailPasswordSignInFormType.signIn
                        ? 'Need an account? Register'
                        : 'Have an account? Sign in',
                  ),
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

**Form Patterns:**
- ✅ GlobalKey for form state
- ✅ TextEditingController for input
- ✅ Validators for each field
- ✅ Dispose controllers
- ✅ Listen for errors
- ✅ Disable button while loading

---

## Navigation Integration

### Declarative Navigation

```dart
// Navigate to product detail
context.goNamed(
  AppRoute.product.name,
  pathParameters: {'id': productId},
);

// Navigate to leave review
context.goNamed(
  AppRoute.leaveReview.name,
  pathParameters: {'id': productId},
);

// Go back
context.pop();
```

### Navigation After Success

```dart
ref.listen<AsyncValue<void>>(
  paymentButtonControllerProvider,
  (_, state) {
    if (state.hasValue) {
      // Success: navigate to orders
      context.goNamed(AppRoute.orders.name);
    }
  },
);
```

---

## Error Handling in UI

### Global Error Listener

**File:** [lib/src/utils/async_value_ui.dart](../lib/src/utils/async_value_ui.dart)

```dart
extension AsyncValueUI on AsyncValue {
  void showAlertDialogOnError(BuildContext context) {
    if (!isLoading && hasError) {
      showExceptionAlertDialog(
        context: context,
        title: 'Error',
        exception: error,
      );
    }
  }
}
```

**Usage:**
```dart
ref.listen<AsyncValue<void>>(
  controllerProvider,
  (_, state) => state.showAlertDialogOnError(context),
);
```

### Custom Error Messages

```dart
void showExceptionAlertDialog({
  required BuildContext context,
  required String title,
  required Object? exception,
}) {
  final message = _getErrorMessage(exception);

  showDialog(
    context: context,
    builder: (_) => AlertDialog(
      title: Text(title),
      content: Text(message),
      actions: [
        TextButton(
          onPressed: () => Navigator.of(context).pop(),
          child: const Text('OK'),
        ),
      ],
    ),
  );
}

String _getErrorMessage(Object? exception) {
  if (exception is AppException) {
    return exception.message;
  }
  return exception.toString();
}
```

---

## Loading States

### Button Loading

```dart
PrimaryButton(
  text: 'Add to Cart',
  isLoading: state.isLoading,
  onPressed: state.isLoading ? null : _onPressed,
)
```

### Screen Loading

```dart
AsyncValueWidget<List<Product>>(
  value: productsAsync,
  data: (products) => ProductList(products),
)
// Shows CircularProgressIndicator during loading
```

### Skeleton Loading (Custom)

```dart
class ProductCardSkeleton extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(
        children: [
          Container(
            height: 150,
            color: Colors.grey[300],
          ),
          Padding(
            padding: const EdgeInsets.all(8),
            child: Column(
              children: [
                Container(
                  height: 20,
                  color: Colors.grey[300],
                ),
                const SizedBox(height: 8),
                Container(
                  height: 16,
                  color: Colors.grey[300],
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## Best Practices

### Application Layer

✅ **Services coordinate, don't implement** - Call repositories, don't access data directly
✅ **Validate business rules** - Check preconditions before operations
✅ **Handle errors gracefully** - Throw custom exceptions
✅ **Keep services focused** - Single responsibility
✅ **Use Ref for dependencies** - Dependency injection
✅ **Return Future/Stream** - Async by nature
✅ **Log errors** - Don't swallow exceptions

### Presentation Layer

✅ **Separate concerns** - Controllers handle logic, widgets display
✅ **Use ConsumerWidget** - For reactive UI
✅ **Const constructors** - Performance optimization
✅ **Composition over inheritance** - Build from smaller widgets
✅ **Extract widgets** - Reusable components
✅ **Handle all AsyncValue states** - loading, error, data
✅ **Listen for side effects** - Use ref.listen for dialogs/navigation
✅ **Dispose resources** - Controllers, animations, streams

### ❌ Common Mistakes

❌ **Business logic in widgets** - Move to controllers/services
❌ **Tight coupling** - Widgets shouldn't know about repositories
❌ **Forgetting loading states** - Always handle AsyncValue properly
❌ **Not disposing controllers** - Memory leaks
❌ **Deep widget trees** - Extract into smaller widgets
❌ **Magic numbers** - Use constants
❌ **Ignoring errors** - Always handle error cases

---

## Summary

**Application Layer:**
- ✅ Services orchestrate business logic
- ✅ Coordinate multiple repositories
- ✅ Validate business rules
- ✅ Handle side effects

**Presentation Layer:**
- ✅ Controllers manage UI state
- ✅ Widgets compose the UI
- ✅ Reactive with Riverpod
- ✅ Responsive design patterns
- ✅ Comprehensive error handling

Together, these layers create a **clean, maintainable, testable UI** built on solid business logic.

---

## What's Next?

Now let's see complete **feature walkthroughs** that tie everything together!

Head to **[07-FEATURES-WALKTHROUGH.md](07-FEATURES-WALKTHROUGH.md)** for:
- Complete authentication flow
- Product browsing and search
- Shopping cart from guest to signed-in user
- Checkout and order placement
- Reviews workflow
- End-to-end examples

Ready to see it all come together? Let's go! 🚀
