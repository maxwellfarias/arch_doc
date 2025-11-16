# Routing, Testing, and Best Practices - Production-Ready Flutter

## Table of Contents
- [Introduction](#introduction)
- [GoRouter Deep Dive](#gorouter-deep-dive)
- [Route Configuration](#route-configuration)
- [Navigation Patterns](#navigation-patterns)
- [Route Guards and Redirects](#route-guards-and-redirects)
- [Testing Strategy](#testing-strategy)
- [Unit Testing](#unit-testing)
- [Widget Testing](#widget-testing)
- [Integration Testing](#integration-testing)
- [Robot Pattern](#robot-pattern)
- [Performance Optimization](#performance-optimization)
- [Production Best Practices](#production-best-practices)
- [Common Pitfalls](#common-pitfalls)
- [Deployment Checklist](#deployment-checklist)
- [Conclusion](#conclusion)

---

## Introduction

This final guide covers the **essential production topics**:
- **Routing**: Navigation with GoRouter
- **Testing**: Comprehensive testing strategies
- **Performance**: Optimization techniques
- **Best Practices**: Production-ready patterns
- **Deployment**: Checklist for release

---

## GoRouter Deep Dive

**GoRouter** is a declarative routing solution that provides type-safe navigation with deep linking support.

### Why GoRouter?

✅ **Declarative**: Define routes in one place
✅ **Type-safe**: Compile-time route validation
✅ **Deep linking**: URL-based navigation
✅ **Route guards**: Authentication-based redirects
✅ **Nested routes**: Child/parent relationships
✅ **Web support**: Clean URLs for web apps

### Basic Concepts

**Routes**: Define where users can navigate
**Path Parameters**: Dynamic route segments (`/product/:id`)
**Query Parameters**: Optional parameters (`?search=shoes`)
**Redirects**: Conditional navigation based on state
**Navigation**: Push/pop/replace routes

---

## Route Configuration

### Complete Router Setup

**File:** [lib/src/routing/app_router.dart](../lib/src/routing/app_router.dart)

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

@Riverpod(keepAlive: true)
GoRouter goRouter(Ref ref) {
  final authRepository = ref.watch(authRepositoryProvider);

  return GoRouter(
    initialLocation: '/',
    debugLogDiagnostics: false,

    // Reactive navigation: rebuilds when auth state changes
    refreshListenable: GoRouterRefreshStream(
      authRepository.authStateChanges(),
    ),

    // Route guards
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
      return null;  // No redirect needed
    },

    routes: [
      // Home route
      GoRoute(
        path: '/',
        name: AppRoute.home.name,
        builder: (context, state) => const ProductsListScreen(),
        routes: [
          // Product detail
          GoRoute(
            path: 'product/:id',
            name: AppRoute.product.name,
            builder: (context, state) {
              final productId = state.pathParameters['id']!;
              return ProductScreen(productId: productId);
            },
            routes: [
              // Leave review (nested under product)
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
        ],
      ),

      // Shopping cart
      GoRoute(
        path: '/cart',
        name: AppRoute.cart.name,
        pageBuilder: (context, state) => const MaterialPage(
          fullscreenDialog: true,
          child: ShoppingCartScreen(),
        ),
        routes: [
          // Checkout
          GoRoute(
            path: 'checkout',
            name: AppRoute.checkout.name,
            pageBuilder: (context, state) => const MaterialPage(
              fullscreenDialog: true,
              child: CheckoutScreen(),
            ),
          ),
        ],
      ),

      // Orders
      GoRoute(
        path: '/orders',
        name: AppRoute.orders.name,
        pageBuilder: (context, state) => const MaterialPage(
          fullscreenDialog: true,
          child: OrdersListScreen(),
        ),
      ),

      // Account
      GoRoute(
        path: '/account',
        name: AppRoute.account.name,
        pageBuilder: (context, state) => const MaterialPage(
          fullscreenDialog: true,
          child: AccountScreen(),
        ),
      ),

      // Sign in
      GoRoute(
        path: '/signIn',
        name: AppRoute.signIn.name,
        pageBuilder: (context, state) => const MaterialPage(
          fullscreenDialog: true,
          child: EmailPasswordSignInScreen(),
        ),
      ),
    ],

    // Error handling
    errorBuilder: (context, state) => Scaffold(
      appBar: AppBar(title: const Text('Page Not Found')),
      body: Center(
        child: Text('Could not find route: ${state.uri.path}'),
      ),
    ),
  );
}
```

### Route Structure Visualization

```
/ (home - ProductsListScreen)
│
├─ /product/:id (ProductScreen)
│  └─ /product/:id/review (LeaveReviewScreen)
│
├─ /cart (ShoppingCartScreen)
│  └─ /cart/checkout (CheckoutScreen)
│
├─ /orders (OrdersListScreen)
│
├─ /account (AccountScreen)
│
└─ /signIn (EmailPasswordSignInScreen)
```

### GoRouterRefreshStream

**File:** [lib/src/routing/go_router_refresh_stream.dart](../lib/src/routing/go_router_refresh_stream.dart)

```dart
/// Wrapper to convert Stream to ChangeNotifier for GoRouter
class GoRouterRefreshStream extends ChangeNotifier {
  GoRouterRefreshStream(Stream<dynamic> stream) {
    notifyListeners();
    _subscription = stream.asBroadcastStream().listen((_) {
      notifyListeners();
    });
  }

  late final StreamSubscription<dynamic> _subscription;

  @override
  void dispose() {
    _subscription.cancel();
    super.dispose();
  }
}
```

**Purpose**: Converts auth state stream to ChangeNotifier so GoRouter can react to auth changes.

---

## Navigation Patterns

### 1. Named Route Navigation

```dart
// Navigate to product detail
context.goNamed(
  AppRoute.product.name,
  pathParameters: {'id': product.id},
);

// Navigate to leave review (nested route)
context.goNamed(
  AppRoute.leaveReview.name,
  pathParameters: {'id': product.id},
);
```

### 2. Imperative Navigation

```dart
// Push route
context.push('/product/123');

// Replace current route
context.replace('/home');

// Go back
context.pop();

// Go back with result
context.pop(result);
```

### 3. Full-Screen Dialogs

```dart
GoRoute(
  path: '/cart',
  name: AppRoute.cart.name,
  pageBuilder: (context, state) => const MaterialPage(
    fullscreenDialog: true,  // Slide up animation
    child: ShoppingCartScreen(),
  ),
)
```

### 4. Navigation with State

```dart
// Pass extra data
context.goNamed(
  AppRoute.product.name,
  pathParameters: {'id': '123'},
  extra: {'source': 'search'},
);

// Retrieve in destination
final extra = GoRouterState.of(context).extra as Map?;
final source = extra?['source'];
```

### 5. Conditional Navigation

```dart
// In controller after async operation
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

## Route Guards and Redirects

### Authentication-Based Redirects

```dart
redirect: (context, state) {
  final isLoggedIn = authRepository.currentUser != null;
  final path = state.uri.path;

  // Logged in trying to access sign-in
  if (isLoggedIn && path == '/signIn') {
    return '/';
  }

  // Not logged in trying to access protected route
  if (!isLoggedIn && (path == '/account' || path == '/orders')) {
    return '/';
  }

  return null;  // Allow navigation
}
```

### Reactive Redirects

```dart
refreshListenable: GoRouterRefreshStream(
  authRepository.authStateChanges(),
)
```

When auth state changes:
1. GoRouter rebuilds
2. `redirect` function runs
3. User redirected if needed
4. Automatically handles sign-in/out

### Deep Link Handling

```dart
// User opens link: myapp://product/123
// GoRouter automatically navigates to ProductScreen(productId: '123')
```

---

## Testing Strategy

This project demonstrates **three levels of testing**:

### Testing Pyramid

```
         ⬆ Integration Tests (slow, comprehensive)
        ███
       █████ Widget Tests (medium speed, UI focused)
      ███████
     █████████ Unit Tests (fast, focused)
    ███████████
```

**Distribution**:
- **70%** Unit tests (domain, data, application)
- **20%** Widget tests (presentation)
- **10%** Integration tests (full flows)

---

## Unit Testing

Testing **business logic** without UI.

### Testing Domain Entities

**File:** [test/src/features/cart/domain/cart_test.dart](../test/src/features/cart/domain/cart_test.dart)

```dart
import 'package:test/test.dart';

void main() {
  group('Cart', () {
    test('empty cart has zero items', () {
      const cart = Cart();

      expect(cart.items, isEmpty);
      expect(cart.itemsCount, 0);
      expect(cart.length, 0);
    });

    test('addItem adds new item to cart', () {
      const cart = Cart();
      const item = Item(productId: 'p1', quantity: 2);

      final updated = cart.addItem(item);

      expect(updated.items['p1'], 2);
      expect(updated.itemsCount, 2);
      expect(cart.items, isEmpty);  // Original unchanged
    });

    test('addItem increases quantity of existing item', () {
      final cart = Cart({'p1': 2});
      const item = Item(productId: 'p1', quantity: 3);

      final updated = cart.addItem(item);

      expect(updated.items['p1'], 5);
    });

    test('setItem replaces quantity', () {
      final cart = Cart({'p1': 5});
      const item = Item(productId: 'p1', quantity: 2);

      final updated = cart.setItem(item);

      expect(updated.items['p1'], 2);
    });

    test('removeItemById removes item from cart', () {
      final cart = Cart({'p1': 2, 'p2': 1});

      final updated = cart.removeItemById('p1');

      expect(updated.items, {'p2': 1});
      expect(updated.itemsCount, 1);
    });

    test('toJson and fromJson round-trip', () {
      final cart = Cart({'p1': 2, 'p2': 3});

      final json = cart.toJson();
      final deserialized = Cart.fromJson(json);

      expect(deserialized, cart);
    });
  });
}
```

### Testing Services

**File:** [test/src/features/cart/application/cart_service_test.dart](../test/src/features/cart/application/cart_service_test.dart)

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

class MockAuthRepository extends Mock implements FakeAuthRepository {}
class MockRemoteCartRepository extends Mock implements RemoteCartRepository {}
class MockLocalCartRepository extends Mock implements LocalCartRepository {}

void main() {
  late MockAuthRepository authRepository;
  late MockRemoteCartRepository remoteCartRepository;
  late MockLocalCartRepository localCartRepository;

  setUp(() {
    authRepository = MockAuthRepository();
    remoteCartRepository = MockRemoteCartRepository();
    localCartRepository = MockLocalCartRepository();
  });

  ProviderContainer makeProviderContainer() {
    return ProviderContainer(
      overrides: [
        authRepositoryProvider.overrideWithValue(authRepository),
        remoteCartRepositoryProvider.overrideWithValue(remoteCartRepository),
        localCartRepositoryProvider.overrideWithValue(localCartRepository),
      ],
    );
  }

  group('CartService', () {
    test('uses local cart when not authenticated', () async {
      const uid = 'test-uid';
      const cart = Cart({'p1': 1});

      when(() => authRepository.currentUser).thenReturn(null);
      when(() => localCartRepository.fetchCart())
          .thenAnswer((_) async => const Cart());
      when(() => localCartRepository.setCart(any()))
          .thenAnswer((_) async {});

      final container = makeProviderContainer();
      final service = container.read(cartServiceProvider);
      const item = Item(productId: 'p1', quantity: 1);

      await service.setItem(item);

      verify(() => localCartRepository.setCart(any())).called(1);
      verifyNever(() => remoteCartRepository.setCart(any(), any()));
    });

    test('uses remote cart when authenticated', () async {
      const uid = 'test-uid';
      final user = AppUser(uid: uid);

      when(() => authRepository.currentUser).thenReturn(user);
      when(() => remoteCartRepository.fetchCart(uid))
          .thenAnswer((_) async => const Cart());
      when(() => remoteCartRepository.setCart(uid, any()))
          .thenAnswer((_) async {});

      final container = makeProviderContainer();
      final service = container.read(cartServiceProvider);
      const item = Item(productId: 'p1', quantity: 1);

      await service.setItem(item);

      verify(() => remoteCartRepository.setCart(uid, any())).called(1);
      verifyNever(() => localCartRepository.setCart(any()));
    });
  });
}
```

### Key Testing Patterns

✅ **Arrange-Act-Assert**: Standard testing structure
✅ **Mocking with Mocktail**: Mock dependencies
✅ **Provider overrides**: Replace real implementations
✅ **Verify interactions**: Check methods called
✅ **Test isolation**: Each test independent

---

## Widget Testing

Testing **UI components** in isolation.

### Testing Simple Widgets

```dart
import 'package:flutter_test/flutter_test.dart';

void main() {
  testWidgets('ProductCard displays product info', (tester) async {
    const product = Product(
      id: '1',
      imageUrl: 'assets/test.jpg',
      title: 'Test Product',
      price: 29.99,
      description: 'Test description',
      availableQuantity: 10,
    );

    await tester.pumpWidget(
      MaterialApp(
        home: Scaffold(
          body: ProductCard(product: product),
        ),
      ),
    );

    // Verify text appears
    expect(find.text('Test Product'), findsOneWidget);
    expect(find.text('\$29.99'), findsOneWidget);
  });
}
```

### Testing with Riverpod

```dart
testWidgets('AddToCartButton shows loading state', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      child: MaterialApp(
        home: Scaffold(
          body: AddToCartButton(productId: 'p1'),
        ),
      ),
    ),
  );

  // Initial state
  expect(find.text('Add to Cart'), findsOneWidget);
  expect(find.byType(CircularProgressIndicator), findsNothing);

  // Tap button
  await tester.tap(find.text('Add to Cart'));
  await tester.pump();

  // Loading state
  expect(find.byType(CircularProgressIndicator), findsOneWidget);
});
```

---

## Integration Testing

Testing **complete user flows** end-to-end.

### Purchase Flow Test

**File:** [integration_test/purchase_flow_test.dart](../integration_test/purchase_flow_test.dart)

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('Complete purchase flow', (tester) async {
    // Start app
    await tester.pumpWidget(const MyApp());
    await tester.pumpAndSettle();

    // 1. Browse products
    expect(find.text('Products'), findsOneWidget);

    // 2. Tap first product
    await tester.tap(find.byType(ProductCard).first);
    await tester.pumpAndSettle();

    // 3. Add to cart
    await tester.tap(find.text('Add to Cart'));
    await tester.pumpAndSettle();

    // 4. Verify cart badge shows 1
    expect(find.text('1'), findsOneWidget);

    // 5. Go to cart
    await tester.tap(find.byIcon(Icons.shopping_cart));
    await tester.pumpAndSettle();

    // 6. Verify item in cart
    expect(find.byType(CartItemWidget), findsOneWidget);

    // 7. Go to checkout (redirects to sign-in)
    await tester.tap(find.text('Checkout'));
    await tester.pumpAndSettle();

    // 8. Sign in
    await tester.enterText(
      find.byType(TextField).first,
      'test@test.com',
    );
    await tester.enterText(
      find.byType(TextField).last,
      'password123',
    );
    await tester.tap(find.text('Sign In'));
    await tester.pumpAndSettle();

    // 9. Place order
    await tester.tap(find.text('Place Order'));
    await tester.pumpAndSettle();

    // 10. Verify on orders screen
    expect(find.text('Your Orders'), findsOneWidget);
    expect(find.byType(OrderCard), findsOneWidget);
  });
}
```

### Running Integration Tests

```bash
flutter test integration_test/purchase_flow_test.dart
```

---

## Robot Pattern

The **Robot Pattern** makes tests more readable and maintainable.

### Example Robot

**File:** [test/src/robot.dart](../test/src/robot.dart)

```dart
class Robot {
  Robot(this.tester);

  final WidgetTester tester;

  Future<void> pumpMyApp() async {
    await tester.pumpWidget(const MyApp());
    await tester.pumpAndSettle();
  }

  Future<void> tapButton(String text) async {
    await tester.tap(find.text(text));
    await tester.pumpAndSettle();
  }

  void expectTextFound(String text) {
    expect(find.text(text), findsOneWidget);
  }

  void expectNoText(String text) {
    expect(find.text(text), findsNothing);
  }
}
```

### Auth Robot

**File:** [test/src/features/authentication/auth_robot.dart](../test/src/features/authentication/auth_robot.dart)

```dart
class AuthRobot {
  AuthRobot(this.tester);

  final WidgetTester tester;

  Future<void> signIn(String email, String password) async {
    await tester.enterText(find.byKey(const Key('email')), email);
    await tester.enterText(find.byKey(const Key('password')), password);
    await tester.tap(find.text('Sign In'));
    await tester.pumpAndSettle();
  }

  void expectSignedIn() {
    expect(find.text('Account'), findsOneWidget);
    expect(find.text('Sign In'), findsNothing);
  }

  void expectSignedOut() {
    expect(find.text('Sign In'), findsOneWidget);
  }
}
```

### Using Robots in Tests

```dart
testWidgets('Sign in flow', (tester) async {
  final r = Robot(tester);
  final auth = AuthRobot(tester);

  await r.pumpMyApp();

  // Navigate to sign-in
  await r.tapButton('Account');

  // Sign in
  await auth.signIn('test@test.com', 'password123');

  // Verify signed in
  auth.expectSignedIn();
});
```

### Benefits of Robot Pattern

✅ **Reusable**: Same actions across tests
✅ **Readable**: Tests read like user stories
✅ **Maintainable**: Change UI, update robot once
✅ **Encapsulation**: Hide implementation details

---

## Performance Optimization

### 1. Const Constructors

```dart
// ✅ Good: const constructor
class ProductCard extends StatelessWidget {
  const ProductCard({super.key, required this.product});
  final Product product;
  // ...
}

// Usage
const ProductCard(product: product)  // Reuses same instance!
```

### 2. Provider Auto-Dispose

```dart
// Auto-disposes when no longer used
@riverpod
Future<Product> product(Ref ref, String id) {
  return fetchProduct(id);
}

// Keep alive for singletons
@Riverpod(keepAlive: true)
AuthRepository authRepository(Ref ref) {
  return FakeAuthRepository();
}
```

### 3. Selective Rebuilds

```dart
// ❌ Bad: Rebuilds on any cart change
final cart = ref.watch(cartProvider);
return Text('Items: ${cart.items.length}');

// ✅ Good: Only rebuilds when count changes
final count = ref.watch(cartItemsCountProvider);
return Text('Items: $count');
```

### 4. ListView.builder

```dart
// ✅ Good: Lazy loading
ListView.builder(
  itemCount: products.length,
  itemBuilder: (context, index) {
    return ProductCard(product: products[index]);
  },
)

// ❌ Bad: All at once
Column(
  children: products.map((p) => ProductCard(product: p)).toList(),
)
```

### 5. Image Caching

```dart
// Cached network images
CachedNetworkImage(
  imageUrl: product.imageUrl,
  placeholder: (context, url) => CircularProgressIndicator(),
  errorWidget: (context, url, error) => Icon(Icons.error),
)

// Asset images are cached by default
Image.asset(product.imageUrl)
```

### 6. Avoid Expensive Operations in build()

```dart
// ❌ Bad: Filtering in build
@override
Widget build(BuildContext context) {
  final filtered = products.where((p) => p.price < 100).toList();
  return ProductList(products: filtered);
}

// ✅ Good: Use computed provider
@riverpod
List<Product> affordableProducts(Ref ref) {
  final products = ref.watch(productsListProvider);
  return products.where((p) => p.price < 100).toList();
}
```

---

## Production Best Practices

### 1. Error Logging

```dart
// Setup error logging
void registerErrorHandlers(ErrorLogger errorLogger) {
  // Flutter errors
  FlutterError.onError = (FlutterErrorDetails details) {
    FlutterError.presentError(details);
    errorLogger.logError(details.exception, details.stack);
  };

  // Platform errors
  PlatformDispatcher.instance.onError = (Object error, StackTrace stack) {
    errorLogger.logError(error, stack);
    return true;
  };
}
```

### 2. Analytics Integration

```dart
// Track screen views
class AnalyticsObserver extends NavigatorObserver {
  @override
  void didPush(Route route, Route? previousRoute) {
    analytics.logScreenView(
      screenName: route.settings.name,
    );
  }
}

// Usage
MaterialApp.router(
  routerConfig: goRouter,
  navigatorObservers: [AnalyticsObserver()],
)
```

### 3. Environment Configuration

```dart
// config.dart
class Config {
  static const apiUrl = String.fromEnvironment(
    'API_URL',
    defaultValue: 'https://api.example.com',
  );

  static const isDev = bool.fromEnvironment('DEV');
}

// Run with:
// flutter run --dart-define=API_URL=https://staging.api.com --dart-define=DEV=true
```

### 4. Secure Storage

```dart
// Use flutter_secure_storage for sensitive data
final storage = FlutterSecureStorage();

// Store
await storage.write(key: 'auth_token', value: token);

// Read
final token = await storage.read(key: 'auth_token');
```

### 5. Network Resilience

```dart
// Retry logic
Future<T> retryOperation<T>(Future<T> Function() operation) async {
  int retries = 3;

  while (retries > 0) {
    try {
      return await operation();
    } catch (e) {
      retries--;
      if (retries == 0) rethrow;
      await Future.delayed(Duration(seconds: 2));
    }
  }

  throw Exception('Max retries exceeded');
}
```

### 6. Offline Support

```dart
// Check connectivity
final connectivityResult = await Connectivity().checkConnectivity();

if (connectivityResult == ConnectivityResult.none) {
  // Show offline message
  // Use cached data
}
```

---

## Common Pitfalls

### ❌ Pitfall 1: Not Disposing Resources

```dart
// ❌ Bad
class MyScreen extends StatefulWidget {
  final controller = TextEditingController();
  // Memory leak!
}

// ✅ Good
class MyScreen extends StatefulWidget {
  final controller = TextEditingController();

  @override
  void dispose() {
    controller.dispose();
    super.dispose();
  }
}
```

### ❌ Pitfall 2: Using ref.read in build()

```dart
// ❌ Bad: Won't rebuild
@override
Widget build(BuildContext context, WidgetRef ref) {
  final count = ref.read(counterProvider);
  return Text('$count');
}

// ✅ Good: Rebuilds on change
@override
Widget build(BuildContext context, WidgetRef ref) {
  final count = ref.watch(counterProvider);
  return Text('$count');
}
```

### ❌ Pitfall 3: Deep Widget Trees

```dart
// ❌ Bad: All in one widget
Widget build(BuildContext context) {
  return Column(
    children: [
      Container(
        child: Row(
          children: [
            Container(
              child: Text('Title'),
            ),
            Container(
              child: Icon(Icons.star),
            ),
          ],
        ),
      ),
      // ... 100 more lines
    ],
  );
}

// ✅ Good: Extract widgets
Widget build(BuildContext context) {
  return Column(
    children: [
      ProductHeader(),
      ProductRating(),
      ProductDescription(),
    ],
  );
}
```

### ❌ Pitfall 4: Ignoring AsyncValue States

```dart
// ❌ Bad: Only handles data
final products = ref.watch(productsProvider).value;
return ProductList(products: products!);  // Crash if loading/error!

// ✅ Good: Handle all states
final productsValue = ref.watch(productsProvider);
return productsValue.when(
  loading: () => LoadingWidget(),
  error: (e, s) => ErrorWidget(e),
  data: (products) => ProductList(products: products),
);
```

### ❌ Pitfall 5: Business Logic in Widgets

```dart
// ❌ Bad: Logic in widget
class CheckoutButton extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return ElevatedButton(
      onPressed: () async {
        final cart = await ref.read(cartProvider.future);
        final total = cart.items.entries
            .map((e) => e.value * getPrice(e.key))
            .reduce((a, b) => a + b);

        final order = Order(/* ... */);
        await saveOrder(order);
        await clearCart();
      },
      child: Text('Checkout'),
    );
  }
}

// ✅ Good: Logic in controller/service
class CheckoutButton extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return ElevatedButton(
      onPressed: () {
        ref.read(checkoutControllerProvider.notifier).checkout();
      },
      child: Text('Checkout'),
    );
  }
}
```

---

## Deployment Checklist

### Pre-Release

- [ ] **All tests pass** (`flutter test`)
- [ ] **No linter warnings** (`flutter analyze`)
- [ ] **Remove debug code** (print statements, test data)
- [ ] **Update version** (pubspec.yaml)
- [ ] **Generate release notes**
- [ ] **Test on real devices** (iOS and Android)
- [ ] **Check app permissions** (AndroidManifest.xml, Info.plist)
- [ ] **Optimize images** (compress assets)
- [ ] **Enable ProGuard** (Android obfuscation)
- [ ] **Configure crash reporting** (Sentry, Firebase Crashlytics)
- [ ] **Setup analytics** (Firebase Analytics, Mixpanel)
- [ ] **Review privacy policy**
- [ ] **Update app store listings**

### Build Commands

```bash
# Android
flutter build appbundle --release

# iOS
flutter build ipa --release

# Web
flutter build web --release
```

### Code Obfuscation

```bash
flutter build appbundle --obfuscate --split-debug-info=./debug-info
```

### Performance Profiling

```bash
# Run with performance overlay
flutter run --profile

# Analyze performance
flutter run --trace-startup
```

---

## Conclusion

Congratulations! You've completed the comprehensive documentation for this Flutter e-commerce project.

### What You've Learned

✅ **Project Overview** - Structure, features, technology stack
✅ **Clean Architecture** - Layers, dependencies, organization
✅ **Riverpod** - State management from basics to advanced
✅ **Riverpod in Practice** - Real implementations
✅ **Domain & Data Layers** - Entities, repositories, persistence
✅ **Application & Presentation** - Services, controllers, UI
✅ **Feature Walkthroughs** - Complete user journeys
✅ **Routing & Testing** - Navigation, testing strategies, best practices

### Key Takeaways

**Architecture:**
- Feature-first organization scales well
- Clean architecture separates concerns
- Dependency injection via Riverpod

**State Management:**
- Riverpod provides type-safe, reactive state
- Providers create dependency graph
- Controllers handle UI logic

**Testing:**
- Unit tests for business logic
- Widget tests for UI components
- Integration tests for complete flows
- Robot pattern for maintainability

**Production:**
- Error logging and analytics
- Performance optimization
- Security best practices
- Deployment checklist

### Next Steps

1. **Experiment**: Modify the code, add features
2. **Practice**: Build your own app using these patterns
3. **Extend**: Add real backend, authentication, payments
4. **Deploy**: Publish to app stores
5. **Learn More**: Explore advanced topics (animations, platform channels)

### Resources

**Official Documentation:**
- [Flutter Docs](https://flutter.dev/docs)
- [Riverpod Docs](https://riverpod.dev)
- [GoRouter Docs](https://pub.dev/packages/go_router)

**Community:**
- [Flutter Discord](https://discord.gg/flutter)
- [r/FlutterDev](https://reddit.com/r/FlutterDev)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/flutter)

**Advanced Topics:**
- Animations and transitions
- Platform-specific code
- Background processing
- Push notifications
- CI/CD pipelines

### Final Thoughts

This project demonstrates **production-quality Flutter development**. The patterns and practices shown here are used in real-world applications serving millions of users.

Key principles to remember:
- **Separation of concerns** makes code maintainable
- **Testing** gives confidence in changes
- **Type safety** catches bugs early
- **Reactive programming** simplifies state management
- **Clean code** is a team effort

Thank you for following this comprehensive guide. You now have the knowledge to build professional Flutter applications!

**Happy coding! 🚀**

---

## Quick Reference

### Common Commands

```bash
# Development
flutter run
flutter pub get
dart run build_runner watch

# Testing
flutter test
flutter test integration_test/

# Analysis
flutter analyze
dart fix --apply

# Build
flutter build appbundle --release
flutter build ipa --release
flutter build web --release
```

### Project Structure

```
lib/
├── main.dart
└── src/
    ├── app.dart
    ├── constants/
    ├── common_widgets/
    ├── exceptions/
    ├── features/
    │   └── [feature]/
    │       ├── domain/
    │       ├── data/
    │       ├── application/
    │       └── presentation/
    ├── routing/
    └── utils/
```

### Provider Patterns

```dart
// Repository (singleton)
@Riverpod(keepAlive: true)
Repository repository(Ref ref) => Repository();

// Stream provider
@riverpod
Stream<Data> dataStream(Ref ref) => repository.watchData();

// Async provider with parameter
@riverpod
Future<Item> item(Ref ref, String id) => repository.fetchItem(id);

// Controller
@riverpod
class MyController extends _$MyController {
  @override
  FutureOr<void> build() {}

  Future<void> action() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() => doSomething());
  }
}
```

### Navigation

```dart
// Navigate to named route
context.goNamed(AppRoute.product.name, pathParameters: {'id': '123'});

// Go back
context.pop();

// Listen for navigation success
ref.listen<AsyncValue<void>>(controllerProvider, (_, state) {
  if (state.hasValue) context.goNamed(AppRoute.success.name);
});
```

### Testing

```dart
// Unit test
test('description', () {
  final result = function();
  expect(result, expected);
});

// Widget test
testWidgets('description', (tester) async {
  await tester.pumpWidget(widget);
  expect(find.text('Hello'), findsOneWidget);
});

// With Riverpod
testWidgets('description', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [provider.overrideWithValue(mock)],
      child: widget,
    ),
  );
});
```

---

**End of Documentation**

You've reached the end of this comprehensive guide. All 8 documents together provide a complete understanding of professional Flutter development with Clean Architecture and Riverpod.

Good luck with your Flutter journey! 🎯
