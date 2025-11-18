# Features Deep-Dive

## Table of Contents
- [Introduction](#introduction)
- [Authentication Feature](#authentication-feature)
- [Products Feature](#products-feature)
- [Cart Feature](#cart-feature)
- [Checkout Feature](#checkout-feature)
- [Orders Feature](#orders-feature)
- [Reviews Feature](#reviews-feature)
- [Feature Interactions](#feature-interactions)

---

## Introduction

This document provides a **detailed walkthrough** of each feature in the e-commerce application. We'll explore:
- Feature architecture
- Key files and their roles
- Data flow
- User journeys
- Code examples

### The Seven Features

1. **Authentication** - User sign-in and account management
2. **Products** - Browse and search product catalog
3. **Cart** - Add items and manage shopping cart
4. **Checkout** - Complete purchases
5. **Orders** - View order history
6. **Reviews** - Rate and review products
7. **Address** - Manage shipping addresses (basic)

---

## Authentication Feature

### Overview

The authentication feature handles user sign-in, sign-out, and account management. It uses a **fake authentication system** (no real backend) for demonstration purposes.

**Location:** [lib/src/features/authentication/](lib/src/features/authentication/)

### Architecture

```
authentication/
├── domain/
│   └── app_user.dart              # User entity
├── data/
│   └── fake_auth_repository.dart  # Auth data source
└── presentation/
    ├── account/
    │   └── account_screen.dart    # Account screen
    └── sign_in/
        ├── email_password_sign_in_screen.dart
        ├── email_password_sign_in_controller.dart
        └── email_password_sign_in_form_type.dart
```

### Domain Layer

#### AppUser Entity

**File:** [lib/src/features/authentication/domain/app_user.dart](lib/src/features/authentication/domain/app_user.dart:5-20)

```dart
/// User entity
@immutable
class AppUser {
  const AppUser({
    required this.uid,
    required this.email,
  });

  final String uid;      // Unique identifier
  final String email;    // Email address

  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is AppUser && other.uid == uid && other.email == email;
  }

  @override
  int get hashCode => uid.hashCode ^ email.hashCode;
}
```

**Characteristics:**
- Immutable
- Value-based equality
- No password (never stored in domain)
- Simple structure for demonstration

### Data Layer

#### Fake Auth Repository

**File:** [lib/src/features/authentication/data/fake_auth_repository.dart](lib/src/features/authentication/data/fake_auth_repository.dart:15-120)

```dart
class FakeAuthRepository {
  FakeAuthRepository({this.addDelay = true});

  final bool addDelay;

  /// In-memory store for current user
  final _authState = InMemoryStore<AppUser?>(null);

  /// Stream of auth state changes (Observer pattern)
  Stream<AppUser?> authStateChanges() => _authState.stream;

  /// Current user (null if signed out)
  AppUser? get currentUser => _authState.value;

  /// Fake users database
  final List<FakeAppUser> _users = [
    FakeAppUser(
      uid: '123',
      email: 'test@test.com',
      password: 'test1234',
    ),
  ];

  /// Sign in with email and password
  Future<void> signInWithEmailAndPassword(
    String email,
    String password,
  ) async {
    if (addDelay) {
      await Future.delayed(const Duration(seconds: 2));
    }

    // Find user
    final user = _users.firstWhereOrNull(
      (u) => u.email == email && u.password == password,
    );

    if (user == null) {
      throw Exception('Invalid credentials');
    }

    // Update auth state (triggers stream emission)
    _authState.value = AppUser(uid: user.uid, email: user.email);
  }

  /// Create account
  Future<void> createUserWithEmailAndPassword(
    String email,
    String password,
  ) async {
    if (addDelay) {
      await Future.delayed(const Duration(seconds: 2));
    }

    // Check if user exists
    final existingUser = _users.firstWhereOrNull((u) => u.email == email);
    if (existingUser != null) {
      throw Exception('Email already in use');
    }

    // Create new user
    final newUser = FakeAppUser(
      uid: DateTime.now().millisecondsSinceEpoch.toString(),
      email: email,
      password: password,
    );

    _users.add(newUser);

    // Sign in
    _authState.value = AppUser(uid: newUser.uid, email: newUser.email);
  }

  /// Sign out
  Future<void> signOut() async {
    if (addDelay) {
      await Future.delayed(const Duration(seconds: 1));
    }

    // Clear auth state
    _authState.value = null;
  }

  void dispose() {
    _authState.dispose();
  }
}

/// Internal class for storing credentials (never exposed)
class FakeAppUser {
  const FakeAppUser({
    required this.uid,
    required this.email,
    required this.password,
  });

  final String uid;
  final String email;
  final String password;
}
```

**Key Points:**
- Uses `InMemoryStore` for reactive state
- Fake delay to simulate network calls
- Password validation
- Stream emits on sign-in/sign-out

#### Riverpod Providers

**File:** [lib/src/features/authentication/data/fake_auth_repository.dart](lib/src/features/authentication/data/fake_auth_repository.dart:130-145)

```dart
@Riverpod(keepAlive: true)
FakeAuthRepository authRepository(Ref ref) {
  final auth = FakeAuthRepository();

  // Cleanup on dispose
  ref.onDispose(() => auth.dispose());

  return auth;
}

@Riverpod(keepAlive: true)
Stream<AppUser?> authStateChanges(Ref ref) {
  final authRepository = ref.watch(authRepositoryProvider);
  return authRepository.authStateChanges();
}
```

### Presentation Layer

#### Sign-In Controller

**File:** [lib/src/features/authentication/presentation/sign_in/email_password_sign_in_controller.dart](lib/src/features/authentication/presentation/sign_in/email_password_sign_in_controller.dart:15-65)

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
    // Set loading state
    state = const AsyncLoading<void>();

    // Get auth repository
    final authRepository = ref.read(authRepositoryProvider);

    // Call appropriate method based on form type
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

**What it does:**
1. Takes email, password, and form type
2. Sets loading state (UI shows spinner)
3. Calls auth repository
4. Sets success or error state
5. UI reacts automatically

#### Sign-In Screen

**File:** [lib/src/features/authentication/presentation/sign_in/email_password_sign_in_screen.dart](lib/src/features/authentication/presentation/sign_in/email_password_sign_in_screen.dart:20-150)

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
  Widget build(BuildContext context) {
    // Watch controller state
    final state = ref.watch(emailPasswordSignInControllerProvider);

    return Scaffold(
      appBar: AppBar(
        title: Text(_formType == EmailPasswordSignInFormType.signIn
            ? 'Sign In'
            : 'Create Account'),
      ),
      body: Form(
        key: _formKey,
        child: Column(
          children: [
            // Email field
            TextFormField(
              controller: _emailController,
              decoration: InputDecoration(labelText: 'Email'),
              validator: (value) {
                if (value == null || value.isEmpty) {
                  return 'Email is required';
                }
                return null;
              },
            ),

            // Password field
            TextFormField(
              controller: _passwordController,
              decoration: InputDecoration(labelText: 'Password'),
              obscureText: true,
              validator: (value) {
                if (value == null || value.length < 6) {
                  return 'Password must be at least 6 characters';
                }
                return null;
              },
            ),

            // Submit button
            state.when(
              data: (_) => ElevatedButton(
                onPressed: _submit,
                child: Text(_formType == EmailPasswordSignInFormType.signIn
                    ? 'Sign In'
                    : 'Create Account'),
              ),
              loading: () => CircularProgressIndicator(),
              error: (error, _) => Column(
                children: [
                  Text('Error: $error'),
                  ElevatedButton(
                    onPressed: _submit,
                    child: Text('Retry'),
                  ),
                ],
              ),
            ),

            // Toggle form type
            TextButton(
              onPressed: () {
                setState(() {
                  _formType = _formType == EmailPasswordSignInFormType.signIn
                      ? EmailPasswordSignInFormType.register
                      : EmailPasswordSignInFormType.signIn;
                });
              },
              child: Text(_formType == EmailPasswordSignInFormType.signIn
                  ? 'Create Account'
                  : 'Have an account? Sign In'),
            ),
          ],
        ),
      ),
    );
  }

  Future<void> _submit() async {
    if (!_formKey.currentState!.validate()) {
      return;
    }

    // Call controller
    final controller = ref.read(emailPasswordSignInControllerProvider.notifier);

    await controller.submit(
      email: _emailController.text,
      password: _passwordController.text,
      formType: _formType,
    );

    // Navigate back on success
    if (mounted && ref.read(emailPasswordSignInControllerProvider).hasValue) {
      context.pop();
    }
  }

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }
}
```

### User Journey

```
1. User taps "Account" → Goes to sign-in screen
         ↓
2. User enters email & password
         ↓
3. User taps "Sign In"
         ↓
4. Controller sets loading state → UI shows spinner
         ↓
5. Repository validates credentials
         ↓
6. Success: Auth state updated → Stream emits
         ↓
7. GoRouter detects auth change → Redirects to home
         ↓
8. Cart sync service triggers → Merges local cart to remote
         ↓
9. UI updates everywhere (account button shows "Sign Out")
```

---

## Products Feature

### Overview

The products feature displays a catalog of products with search functionality. Products are stored in-memory (hardcoded) for demonstration.

**Location:** [lib/src/features/products/](lib/src/features/products/)

### Architecture

```
products/
├── domain/
│   └── product.dart               # Product entity
├── data/
│   └── fake_products_repository.dart  # Product data source
└── presentation/
    ├── home_app_bar/
    │   └── home_app_bar.dart      # App bar with search
    ├── products_list/
    │   ├── products_list_screen.dart
    │   └── product_card.dart
    └── product_screen/
        ├── product_screen.dart
        └── product_average_rating.dart
```

### Domain Layer

#### Product Entity

**File:** [lib/src/features/products/domain/product.dart](lib/src/features/products/domain/product.dart:7-50)

```dart
typedef ProductID = String;

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
  final String imageUrl;         // Asset path
  final String title;
  final String description;
  final double price;
  final int availableQuantity;   // Stock
  final double avgRating;        // 0.0 to 5.0
  final int numRatings;          // Number of reviews

  // Serialization
  factory Product.fromJson(Map<String, dynamic> json) { /* ... */ }
  Map<String, dynamic> toJson() { /* ... */ }
}
```

### Data Layer

#### Fake Products Repository

**File:** [lib/src/features/products/data/fake_products_repository.dart](lib/src/features/products/data/fake_products_repository.dart:50-150)

```dart
class FakeProductsRepository {
  FakeProductsRepository({this.addDelay = true});

  final bool addDelay;

  /// Hardcoded product catalog
  static final List<Product> _allProducts = [
    Product(
      id: '1',
      imageUrl: 'assets/products/bruschetta-plate.jpg',
      title: 'Bruschetta',
      description: 'Italian appetizer with fresh tomatoes',
      price: 15.00,
      availableQuantity: 10,
      avgRating: 4.5,
      numRatings: 12,
    ),
    Product(
      id: '2',
      imageUrl: 'assets/products/burger.jpg',
      title: 'Classic Burger',
      description: 'Juicy beef burger with all toppings',
      price: 12.99,
      availableQuantity: 20,
      avgRating: 4.8,
      numRatings: 28,
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

  /// Watch products as stream
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
      await Future.delayed(const Duration(seconds: 1));
    }

    final lowerQuery = query.toLowerCase();

    return _allProducts.where((product) {
      return product.title.toLowerCase().contains(lowerQuery) ||
          product.description.toLowerCase().contains(lowerQuery);
    }).toList();
  }
}
```

#### Riverpod Providers

**File:** [lib/src/features/products/data/fake_products_repository.dart](lib/src/features/products/data/fake_products_repository.dart:155-175)

```dart
@Riverpod(keepAlive: true)
FakeProductsRepository productsRepository(Ref ref) {
  return FakeProductsRepository();
}

@Riverpod(keepAlive: true)
Stream<List<Product>> productsListStream(Ref ref) {
  final repository = ref.watch(productsRepositoryProvider);
  return repository.watchProductsList();
}

@riverpod
Product? product(Ref ref, ProductID id) {
  final repository = ref.watch(productsRepositoryProvider);
  return repository.getProduct(id);
}

@riverpod
Future<List<Product>> productsListSearch(Ref ref, String query) async {
  final repository = ref.watch(productsRepositoryProvider);

  if (query.isEmpty) {
    return repository.getProductsList();
  }

  return repository.searchProducts(query);
}
```

### Presentation Layer

#### Products List Screen

**File:** [lib/src/features/products/presentation/products_list/products_list_screen.dart](lib/src/features/products/presentation/products_list/products_list_screen.dart:15-80)

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
          // Watch products with search
          final productsValue = ref.watch(
            productsListSearchProvider(_searchQuery),
          );

          // Handle different states
          return AsyncValueWidget<List<Product>>(
            value: productsValue,
            data: (products) => products.isEmpty
                ? const EmptyPlaceholderWidget(
                    message: 'No products found',
                  )
                : ProductsGrid(products: products),
          );
        },
      ),
    );
  }
}
```

#### Product Card

**File:** [lib/src/features/products/presentation/products_list/product_card.dart](lib/src/features/products/presentation/products_list/product_card.dart:10-80)

```dart
class ProductCard extends ConsumerWidget {
  const ProductCard({required this.product, super.key});

  final Product product;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Card(
      child: InkWell(
        onTap: () {
          // Navigate to product details
          context.goNamed(
            AppRoute.product.name,
            pathParameters: {'id': product.id},
          );
        },
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
                  // Title
                  Text(
                    product.title,
                    style: Theme.of(context).textTheme.titleMedium,
                    maxLines: 1,
                    overflow: TextOverflow.ellipsis,
                  ),

                  // Price
                  Text(
                    '\$${product.price.toStringAsFixed(2)}',
                    style: Theme.of(context).textTheme.titleSmall,
                  ),

                  // Rating
                  if (product.numRatings > 0)
                    Row(
                      children: [
                        Icon(Icons.star, size: 16, color: Colors.amber),
                        SizedBox(width: 4),
                        Text(
                          '${product.avgRating.toStringAsFixed(1)} (${product.numRatings})',
                          style: Theme.of(context).textTheme.bodySmall,
                        ),
                      ],
                    ),
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

#### Product Detail Screen

**File:** [lib/src/features/products/presentation/product_screen/product_screen.dart](lib/src/features/products/presentation/product_screen/product_screen.dart:15-120)

```dart
class ProductScreen extends ConsumerWidget {
  const ProductScreen({required this.productId, super.key});

  final ProductID productId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Get product
    final product = ref.watch(productProvider(productId));

    if (product == null) {
      return Scaffold(
        appBar: AppBar(),
        body: EmptyPlaceholderWidget(message: 'Product not found'),
      );
    }

    return Scaffold(
      appBar: AppBar(
        title: Text(product.title),
      ),
      body: SingleChildScrollView(
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            // Product image
            AspectRatio(
              aspectRatio: 1,
              child: Image.asset(
                product.imageUrl,
                fit: BoxFit.cover,
              ),
            ),

            // Product details
            Padding(
              padding: const EdgeInsets.all(16.0),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  // Title
                  Text(
                    product.title,
                    style: Theme.of(context).textTheme.headlineSmall,
                  ),

                  SizedBox(height: 8),

                  // Price
                  Text(
                    '\$${product.price.toStringAsFixed(2)}',
                    style: Theme.of(context).textTheme.headlineMedium,
                  ),

                  SizedBox(height: 16),

                  // Description
                  Text(
                    product.description,
                    style: Theme.of(context).textTheme.bodyLarge,
                  ),

                  SizedBox(height: 16),

                  // Average rating
                  ProductAverageRating(product: product),

                  SizedBox(height: 16),

                  // Add to cart button
                  AddToCartWidget(productId: productId),

                  SizedBox(height: 16),

                  // Reviews section
                  ProductReviewsList(productId: productId),
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

### User Journey

```
1. User opens app → Products list loads
         ↓
2. User types in search → Products filter in real-time
         ↓
3. User taps product → Navigate to product details
         ↓
4. User views product info, reviews, ratings
         ↓
5. User taps "Add to Cart" → (See Cart feature)
```

---

## Cart Feature

### Overview

The cart feature manages shopping cart items. It supports:
- **Local cart** for guest users (stored in Sembast)
- **Remote cart** for authenticated users (fake remote storage)
- **Cart synchronization** when users sign in

**Location:** [lib/src/features/cart/](lib/src/features/cart/)

### Architecture

```
cart/
├── domain/
│   ├── cart.dart              # Cart entity
│   ├── item.dart              # Item entity
│   └── mutable_cart.dart      # Business logic (extensions)
├── data/
│   ├── local/
│   │   ├── local_cart_repository.dart      # Interface
│   │   └── sembast_cart_repository.dart    # Implementation
│   └── remote/
│       ├── remote_cart_repository.dart     # Interface
│       └── fake_remote_cart_repository.dart # Implementation
├── application/
│   ├── cart_service.dart      # Cart business logic
│   └── cart_sync_service.dart # Sync local→remote on sign-in
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

### Domain Layer

We covered Cart and Item entities in previous docs. The key innovation is **extension methods for business logic**:

**File:** [lib/src/features/cart/domain/mutable_cart.dart](lib/src/features/cart/domain/mutable_cart.dart:5-90)

```dart
extension MutableCart on Cart {
  /// Add item (increases quantity if exists)
  Cart addItem(Item item) {
    final copy = Map<String, Item>.from(items);
    final currentItem = copy[item.productId];

    if (currentItem != null) {
      // Increase quantity
      copy[item.productId] = Item(
        productId: item.productId,
        quantity: currentItem.quantity + item.quantity,
      );
    } else {
      // Add new item
      copy[item.productId] = item;
    }

    return Cart(copy);
  }

  /// Set item quantity
  Cart setItem(Item item) {
    final copy = Map<String, Item>.from(items);
    copy[item.productId] = item;
    return Cart(copy);
  }

  /// Remove item
  Cart removeItemById(ProductID productId) {
    final copy = Map<String, Item>.from(items);
    copy.remove(productId);
    return Cart(copy);
  }
}

extension CartItems on Cart {
  List<Item> toItemsList() => items.values.toList();

  int get totalQuantity {
    return items.values.fold<int>(
      0,
      (sum, item) => sum + item.quantity,
    );
  }

  double totalPrice(List<Product> productsList) {
    double total = 0.0;
    for (final item in items.values) {
      final product = productsList.firstWhere((p) => p.id == item.productId);
      total += product.price * item.quantity;
    }
    return total;
  }
}
```

### Application Layer

#### Cart Service

**File:** [lib/src/features/cart/application/cart_service.dart](lib/src/features/cart/application/cart_service.dart:15-110)

```dart
class CartService {
  CartService(this.ref);

  final Ref ref;

  /// Fetch cart (handles auth state automatically)
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

  /// Save cart (handles auth state automatically)
  Future<void> _setCart(Cart cart) async {
    final user = ref.read(authStateChangesProvider).value;

    if (user != null) {
      await ref.read(remoteCartRepositoryProvider).setCart(user.uid, cart);
    } else {
      await ref.read(localCartRepositoryProvider).setCart(cart);
    }
  }

  /// Use case: Set item
  Future<void> setItem(Item item) async {
    final cart = await _fetchCart();
    final updatedCart = cart.setItem(item);
    await _setCart(updatedCart);
  }

  /// Use case: Add item
  Future<void> addItem(Item item) async {
    final cart = await _fetchCart();
    final updatedCart = cart.addItem(item);
    await _setCart(updatedCart);
  }

  /// Use case: Remove item
  Future<void> removeItemById(ProductID productId) async {
    final cart = await _fetchCart();
    final updatedCart = cart.removeItemById(productId);
    await _setCart(updatedCart);
  }
}

// Providers
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

**Key insight:** The service abstracts away authentication complexity. Controllers don't need to know if the user is signed in or not!

#### Cart Sync Service

**File:** [lib/src/features/cart/application/cart_sync_service.dart](lib/src/features/cart/application/cart_sync_service.dart:10-90)

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

        // Detect sign-in (was null, now has value)
        if (previousUser == null && user != null) {
          _moveItemsToRemoteCart(user.uid);
        }
      },
    );
  }

  /// Merge local cart into remote cart
  Future<void> _moveItemsToRemoteCart(String uid) async {
    try {
      // Get both repositories
      final localRepo = ref.read(localCartRepositoryProvider);
      final remoteRepo = ref.read(remoteCartRepositoryProvider);

      // Fetch both carts
      final localCart = await localRepo.fetchCart();
      final remoteCart = await remoteRepo.fetchCart(uid);

      // Merge (local items take precedence on conflicts)
      final mergedCart = _mergeCarts(localCart, remoteCart);

      // Save merged cart to remote
      await remoteRepo.setCart(uid, mergedCart);

      // Clear local cart
      await localRepo.setCart(const Cart());
    } catch (e) {
      debugPrint('Error syncing cart: $e');
    }
  }

  /// Business logic: Merge two carts
  Cart _mergeCarts(Cart local, Cart remote) {
    final merged = <String, Item>{...remote.items};

    for (final item in local.items.values) {
      final existingItem = merged[item.productId];

      if (existingItem != null) {
        // Sum quantities
        merged[item.productId] = Item(
          productId: item.productId,
          quantity: existingItem.quantity + item.quantity,
        );
      } else {
        // Add new item
        merged[item.productId] = item;
      }
    }

    return Cart(merged);
  }

  void dispose() {
    _subscription?.cancel();
  }
}

@Riverpod(keepAlive: true)
CartSyncService cartSyncService(Ref ref) {
  final service = CartSyncService(ref);
  ref.onDispose(() => service.dispose());
  return service;
}
```

### Presentation Layer

#### Shopping Cart Screen

**File:** [lib/src/features/cart/presentation/shopping_cart/shopping_cart_screen.dart](lib/src/features/cart/presentation/shopping_cart/shopping_cart_screen.dart:15-80)

```dart
class ShoppingCartScreen extends ConsumerWidget {
  const ShoppingCartScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final cartValue = ref.watch(cartProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Shopping Cart'),
      ),
      body: AsyncValueWidget<Cart>(
        value: cartValue,
        data: (cart) => cart.items.isEmpty
            ? const EmptyPlaceholderWidget(
                message: 'Your cart is empty',
              )
            : Column(
                children: [
                  // Cart items list
                  Expanded(
                    child: ShoppingCartItemsList(cart: cart),
                  ),

                  // Cart total
                  CartTotalText(),

                  // Checkout button
                  Padding(
                    padding: const EdgeInsets.all(16.0),
                    child: ElevatedButton(
                      onPressed: () {
                        context.goNamed(AppRoute.checkout.name);
                      },
                      child: const Text('Checkout'),
                    ),
                  ),
                ],
              ),
      ),
    );
  }
}
```

### User Journey

```
Guest user:
1. Add items to cart → Saved to local database (Sembast)
2. Sign in → Cart sync service merges local → remote
3. Continue shopping → All changes go to remote cart

Authenticated user:
1. Add items to cart → Saved to remote (fake cloud storage)
2. Cart synced across devices (in real app)
3. Sign out → Cart saved remotely, can resume later
```

---

## Checkout Feature

### Overview

The checkout feature handles the purchase flow: reviewing cart, entering payment info, and creating orders.

**Location:** [lib/src/features/checkout/](lib/src/features/checkout/)

### Architecture

```
checkout/
├── application/
│   └── fake_checkout_service.dart  # Checkout business logic
└── presentation/
    ├── checkout_screen/
    │   └── checkout_screen.dart
    └── payment/
        ├── payment_button.dart
        └── payment_button_controller.dart
```

### Application Layer

#### Checkout Service

**File:** [lib/src/features/checkout/application/fake_checkout_service.dart](lib/src/features/checkout/application/fake_checkout_service.dart:15-75)

```dart
class FakeCheckoutService {
  FakeCheckoutService(this.ref);

  final Ref ref;

  /// Use case: Place order
  Future<void> placeOrder() async {
    // 1. Validate user is authenticated
    final user = ref.read(authStateChangesProvider).value;
    if (user == null) {
      throw Exception('User must be signed in');
    }

    // 2. Get cart
    final cart = await ref.read(cartProvider.future);
    if (cart.items.isEmpty) {
      throw Exception('Cart is empty');
    }

    // 3. Get products for price calculation
    final products = await ref.read(productsListStreamProvider.future);
    final total = cart.totalPrice(products);

    // 4. Create order
    final orderId = DateTime.now().toIso8601String();
    final order = Order(
      id: orderId,
      userId: user.uid,
      items: cart.items.map((id, item) => MapEntry(id, item.quantity)),
      orderStatus: OrderStatus.confirmed,
      orderDate: DateTime.now(),
      total: total,
    );

    // 5. Save order
    final ordersRepository = ref.read(ordersRepositoryProvider);
    await ordersRepository.addOrder(user.uid, order);

    // 6. Clear cart
    final remoteCartRepository = ref.read(remoteCartRepositoryProvider);
    await remoteCartRepository.setCart(user.uid, const Cart());
  }
}

@Riverpod(keepAlive: true)
FakeCheckoutService checkoutService(Ref ref) {
  return FakeCheckoutService(ref);
}
```

### Presentation Layer

#### Payment Button Controller

**File:** [lib/src/features/checkout/presentation/payment/payment_button_controller.dart](lib/src/features/checkout/presentation/payment/payment_button_controller.dart:15-40)

```dart
@riverpod
class PaymentButtonController extends _$PaymentButtonController {
  @override
  FutureOr<void> build() {}

  Future<void> processPayment() async {
    // Set loading state
    state = const AsyncLoading();

    // Call checkout service
    final checkoutService = ref.read(checkoutServiceProvider);

    state = await AsyncValue.guard(() => checkoutService.placeOrder());

    // On success, navigate to orders
    if (state.hasValue) {
      // Will be handled by UI
    }
  }
}
```

### User Journey

```
1. User taps "Checkout" from cart → Checkout screen
         ↓
2. User reviews cart items and total
         ↓
3. User taps "Pay" button → Payment controller triggered
         ↓
4. Controller sets loading → UI shows spinner
         ↓
5. Checkout service orchestrates:
   - Validates user authenticated
   - Validates cart not empty
   - Calculates total
   - Creates order
   - Saves to orders repository
   - Clears cart
         ↓
6. On success → Navigate to orders screen
         ↓
7. User sees order confirmation
```

---

## Orders Feature

### Overview

Displays user's order history.

**Location:** [lib/src/features/orders/](lib/src/features/orders/)

### Architecture

```
orders/
├── domain/
│   └── order.dart                 # Order entity
├── data/
│   └── fake_orders_repository.dart # Orders data source
├── application/
│   └── user_orders_provider.dart  # User-specific orders
└── presentation/
    └── orders_list/
        ├── orders_list_screen.dart
        └── order_card.dart
```

### Domain Layer

**File:** [lib/src/features/orders/domain/order.dart](lib/src/features/orders/domain/order.dart:7-50)

```dart
typedef OrderID = String;

enum OrderStatus {
  confirmed,
  shipped,
  delivered,
}

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
  final Map<ProductID, int> items;  // productId → quantity
  final OrderStatus orderStatus;
  final DateTime orderDate;
  final double total;

  // Serialization methods
}
```

### Providers

**File:** [lib/src/features/orders/application/user_orders_provider.dart](lib/src/features/orders/application/user_orders_provider.dart:10-25)

```dart
@riverpod
Future<List<Order>> userOrders(Ref ref) async {
  final user = ref.watch(authStateChangesProvider).value;

  if (user == null) {
    return [];
  }

  final ordersRepository = ref.watch(ordersRepositoryProvider);
  return ordersRepository.fetchOrders(user.uid);
}
```

---

## Reviews Feature

### Overview

Allows users to leave reviews and ratings for products.

**Location:** [lib/src/features/reviews/](lib/src/features/reviews/)

### Architecture

```
reviews/
├── domain/
│   └── review.dart                # Review entity
├── data/
│   └── fake_reviews_repository.dart # Reviews data source
├── application/
│   └── reviews_service.dart       # Review submission logic
└── presentation/
    ├── product_reviews/
    │   ├── product_reviews_list.dart
    │   └── product_review_card.dart
    └── leave_review_screen/
        ├── leave_review_screen.dart
        └── leave_review_controller.dart
```

### Reviews Service

**File:** [lib/src/features/reviews/application/reviews_service.dart](lib/src/features/reviews/application/reviews_service.dart:15-50)

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
      throw Exception('Must be signed in to leave review');
    }

    if (review.rating < 1.0 || review.rating > 5.0) {
      throw Exception('Rating must be between 1 and 5');
    }

    // Save review
    final reviewsRepository = ref.read(reviewsRepositoryProvider);
    await reviewsRepository.setReview(productId, user.uid, review);

    // Invalidate cache to refresh UI
    ref.invalidate(productReviewsProvider(productId));
  }
}
```

---

## Feature Interactions

### Complete Purchase Flow

```
Products Feature → Cart Feature → Checkout Feature → Orders Feature

1. User browses products
2. User adds product to cart
3. Cart service saves to database
4. Cart icon updates (reactive)
5. User goes to cart
6. User proceeds to checkout
7. Checkout service creates order
8. Order saved to repository
9. Cart cleared
10. User redirected to orders
11. Order appears in history
```

### Authentication Impact

```
Authentication Feature affects ALL features:

Sign In:
  → Cart: Local cart merges to remote
  → Orders: Can now view order history
  → Reviews: Can leave reviews
  → Navigation: Redirected from sign-in page

Sign Out:
  → Cart: Switches to local cart
  → Orders: No longer accessible
  → Reviews: Can't leave reviews
  → Navigation: Redirected from protected pages
```

### Data Flow Example: Adding to Cart

```
[Products Screen]
       ↓ User taps "Add to Cart"
[AddToCartController]
       ↓ addItem(productId)
[CartService]
       ↓ addItem(item)
       ├─ fetchCart() ──→ [LocalCartRepository or RemoteCartRepository]
       ├─ cart.addItem() ──→ [Domain logic]
       └─ setCart() ──→ [LocalCartRepository or RemoteCartRepository]
              ↓ Saves to database
       [Database emits stream]
              ↓
       [cartProvider notifies]
              ↓
       ┌──────┴──────────────────────────┐
       ↓                                  ↓
[Shopping Cart Screen]          [Cart Icon Badge]
Updates item count              Updates badge number
```

---

## Summary

This e-commerce app demonstrates how features work together in a clean architecture:

1. **Clear boundaries** between features
2. **Shared dependencies** (auth, products) used across features
3. **Reactive updates** propagate through provider system
4. **Business logic** centralized in services
5. **UI reacts** automatically to state changes

---

## Next Steps

- **[Getting Started Guide](06-getting-started-guide.md)**: Learn how to work with the codebase

---
