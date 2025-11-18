# Design Patterns Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Repository Pattern](#repository-pattern)
3. [Dependency Injection (Service Locator)](#dependency-injection-service-locator)
4. [Use Case Pattern](#use-case-pattern)
5. [BLoC Pattern (Observer)](#bloc-pattern-observer)
6. [Either Monad (Functional Programming)](#either-monad-functional-programming)
7. [Adapter Pattern](#adapter-pattern)
8. [Strategy Pattern](#strategy-pattern)
9. [Factory Pattern](#factory-pattern)
10. [Singleton Pattern](#singleton-pattern)
11. [Summary Table](#summary-table)

---

## Introduction

**Design patterns** are reusable solutions to common programming problems. They're like recipes for solving specific challenges in code organization.

This project uses **8+ design patterns** to achieve:
- ✅ Loose coupling between components
- ✅ Easy testing and mocking
- ✅ Clear separation of concerns
- ✅ Type-safe error handling
- ✅ Maintainable and scalable code

Each pattern is explained with:
- 🎯 **What it is**: Simple definition + real-world analogy
- 🤔 **Why we use it**: Problem it solves
- 📍 **Where it's implemented**: File references with line numbers
- 💻 **Code examples**: Real code from the project

---

## Repository Pattern

### 🎯 What It Is

The **Repository Pattern** acts as a middleman between your business logic and data sources. It provides a clean API for data access without exposing where the data comes from.

**Analogy**: Think of a **library**:
- You (business logic) want a book (data)
- The librarian (repository) gets it for you
- You don't care if the book comes from the main shelf, storage room, or another library
- You just ask the librarian, and they handle the details

### 🤔 Why We Use It

**Problem**: What if you need to change your database from Supabase to Firebase? Without the repository pattern, you'd have to update code everywhere!

**Solution**: The repository pattern creates an abstraction layer:
```
Business Logic → Repository Interface → Repository Implementation → Data Source
```

Change the data source? Just update the repository implementation. Business logic stays untouched!

### 📍 Where It's Implemented

**Domain Layer** (Interface):
- [lib/features/auth/domain/repository/auth_repository.dart](../lib/features/auth/domain/repository/auth_repository.dart:5-16)
- [lib/features/blog/domain/repositories/blog_repository.dart](../lib/features/blog/domain/repositories/blog_repository.dart)

**Data Layer** (Implementation):
- [lib/features/auth/data/repositories/auth_repository_impl.dart](../lib/features/auth/data/repositories/auth_repository_impl.dart:11-90)
- [lib/features/blog/data/repositories/blog_repository_impl.dart](../lib/features/blog/data/repositories/blog_repository_impl.dart:14-76)

### 💻 Code Example

**Step 1: Define Interface (Domain)**

```dart
// lib/features/auth/domain/repository/auth_repository.dart:5-16
abstract interface class AuthRepository {
  Future<Either<Failure, User>> signUpWithEmailPassword({
    required String name,
    required String email,
    required String password,
  });

  Future<Either<Failure, User>> loginWithEmailPassword({
    required String email,
    required String password,
  });

  Future<Either<Failure, User>> currentUser();
}
```

**Step 2: Implement Interface (Data)**

```dart
// lib/features/auth/data/repositories/auth_repository_impl.dart:11-90
class AuthRepositoryImpl implements AuthRepository {
  final AuthRemoteDataSource remoteDataSource;
  final ConnectionChecker connectionChecker;

  const AuthRepositoryImpl(
    this.remoteDataSource,
    this.connectionChecker,
  );

  @override
  Future<Either<Failure, User>> loginWithEmailPassword({
    required String email,
    required String password,
  }) async {
    try {
      // Check network connectivity
      if (!await connectionChecker.isConnected) {
        return left(Failure(Constants.noConnectionErrorMessage));
      }

      // Delegate to data source
      final user = await remoteDataSource.loginWithEmailPassword(
        email: email,
        password: password,
      );

      return right(user);
    } on ServerException catch (e) {
      return left(Failure(e.message));
    }
  }
}
```

**Step 3: Use in Business Logic (Domain Use Case)**

```dart
// Use case depends on interface, not implementation!
class UserLogin implements UseCase<User, UserLoginParams> {
  final AuthRepository authRepository; // Interface, not implementation!

  @override
  Future<Either<Failure, User>> call(UserLoginParams params) async {
    return await authRepository.loginWithEmailPassword(
      email: params.email,
      password: params.password,
    );
  }
}
```

**Benefits**:
- ✅ Use case doesn't know about Supabase, Firebase, or any specific implementation
- ✅ Easy to mock for testing: `MockAuthRepository implements AuthRepository`
- ✅ Easy to swap implementations: Just change dependency injection configuration

---

## Dependency Injection (Service Locator)

### 🎯 What It Is

**Dependency Injection (DI)** is a technique where objects receive their dependencies from external sources rather than creating them internally.

**Service Locator** is a specific DI pattern that provides a central registry for accessing dependencies.

**Analogy**: Think of a **phone directory**:
- Instead of remembering everyone's number (creating dependencies yourself)
- You look them up in the directory (service locator)
- The directory knows how to find everyone (dependency resolution)

### 🤔 Why We Use It

**Problem Without DI**:
```dart
// ❌ BAD: Creating dependencies inside the class
class AuthBloc {
  final UserLogin _userLogin = UserLogin(
    AuthRepositoryImpl(
      AuthRemoteDataSourceImpl(
        SupabaseClient()
      ),
      ConnectionCheckerImpl()
    )
  );
}
```

Problems:
- Hard to test (can't mock dependencies)
- Tightly coupled (changing one thing breaks everything)
- Can't reuse instances (creates new objects every time)

**Solution With DI**:
```dart
// ✅ GOOD: Inject dependencies
class AuthBloc {
  final UserLogin _userLogin;

  AuthBloc({required UserLogin userLogin})
      : _userLogin = userLogin;
}
```

Benefits:
- Easy to test (inject mocks)
- Loosely coupled (AuthBloc doesn't know about implementation details)
- Control object lifetime (singleton, factory, etc.)

### 📍 Where It's Implemented

**Service Locator Setup**:
- [lib/init_dependencies.main.dart](../lib/init_dependencies.main.dart:3-116)

**Usage in main.dart**:
- [lib/main.dart](../lib/main.dart:11-27)

### 💻 Code Example

**Library Used**: `get_it` (Service Locator pattern)

**Step 1: Setup Service Locator**

```dart
// lib/init_dependencies.main.dart:3
final serviceLocator = GetIt.instance;
```

**Step 2: Register Dependencies**

```dart
// lib/init_dependencies.main.dart:5-116
Future<void> initDependencies() async {
  // Initialize Supabase
  final supabase = await Supabase.initialize(
    url: AppSecrets.supabaseUrl,
    anonKey: AppSecrets.supabaseAnonKey,
  );

  // Register as Lazy Singleton (created once, on first use)
  serviceLocator.registerLazySingleton(() => supabase.client);

  // Register core dependencies
  serviceLocator.registerLazySingleton(() => AppUserCubit());

  // Register as Factory (new instance each time)
  serviceLocator.registerFactory<ConnectionChecker>(
    () => ConnectionCheckerImpl(serviceLocator()),
  );

  _initAuth();
  _initBlog();
}
```

**Step 3: Register Feature Dependencies**

```dart
// lib/init_dependencies.main.dart:35-75
void _initAuth() {
  serviceLocator
    // Data Source (Factory - new instance each time)
    ..registerFactory<AuthRemoteDataSource>(
      () => AuthRemoteDataSourceImpl(
        serviceLocator(), // Automatically resolves SupabaseClient
      ),
    )
    // Repository (Factory)
    ..registerFactory<AuthRepository>(
      () => AuthRepositoryImpl(
        serviceLocator(), // Resolves AuthRemoteDataSource
        serviceLocator(), // Resolves ConnectionChecker
      ),
    )
    // Use Cases (Factory)
    ..registerFactory(
      () => UserSignUp(serviceLocator()), // Resolves AuthRepository
    )
    ..registerFactory(
      () => UserLogin(serviceLocator()),
    )
    ..registerFactory(
      () => CurrentUser(serviceLocator()),
    )
    // BLoC (Lazy Singleton - one instance for the app)
    ..registerLazySingleton(
      () => AuthBloc(
        userSignUp: serviceLocator(),
        userLogin: serviceLocator(),
        currentUser: serviceLocator(),
        appUserCubit: serviceLocator(),
      ),
    );
}
```

**Step 4: Use in Application**

```dart
// lib/main.dart:11-27
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Initialize all dependencies
  await initDependencies();

  runApp(MultiBlocProvider(
    providers: [
      BlocProvider(
        // Retrieve from service locator
        create: (_) => serviceLocator<AppUserCubit>(),
      ),
      BlocProvider(
        create: (_) => serviceLocator<AuthBloc>(),
      ),
      BlocProvider(
        create: (_) => serviceLocator<BlogBloc>(),
      ),
    ],
    child: const MyApp(),
  ));
}
```

**Dependency Resolution**:

When you call `serviceLocator<AuthBloc>()`, GetIt automatically:
1. Resolves `UserSignUp` use case
2. Resolves `UserLogin` use case
3. Resolves `CurrentUser` use case
4. Resolves `AppUserCubit`
5. Each use case needs `AuthRepository`
6. Repository needs `AuthRemoteDataSource` and `ConnectionChecker`
7. Data source needs `SupabaseClient`
8. All dependencies are injected automatically!

**Types of Registration**:

| Type | When Created | How Many Instances | Use Case |
|------|--------------|-------------------|----------|
| **Lazy Singleton** | On first use | 1 (same instance always) | BLoCs, Cubits, Clients |
| **Factory** | Every time requested | New instance each time | Use cases, Repositories, Data sources |
| **Singleton** | Immediately on registration | 1 (same instance always) | Rarely used |

---

## Use Case Pattern

### 🎯 What It Is

The **Use Case Pattern** (also called **Interactor**) represents a single business operation. Each use case does ONE thing.

**Analogy**: Think of **buttons on a vending machine**:
- Each button has ONE job (e.g., "Dispense Coke", "Dispense Water")
- Press a button → machine executes that specific action
- You don't press multiple buttons at once

### 🤔 Why We Use It

**Problem**: Where do you put business logic?
- In the UI? ❌ (mixes concerns, hard to test)
- In the repository? ❌ (repository should only handle data)
- In a god class that does everything? ❌ (violates Single Responsibility Principle)

**Solution**: Create a separate class for each business operation!

**Benefits**:
- ✅ Single Responsibility Principle (one class = one job)
- ✅ Easy to test (mock repository, test logic)
- ✅ Easy to understand (class name tells you what it does)
- ✅ Reusable (use same use case in multiple places)

### 📍 Where It's Implemented

**Base Interface**:
- [lib/core/usecase/usecase.dart](../lib/core/usecase/usecase.dart:4-8)

**Auth Use Cases**:
- [lib/features/auth/domain/usecases/user_sign_up.dart](../lib/features/auth/domain/usecases/user_sign_up.dart)
- [lib/features/auth/domain/usecases/user_login.dart](../lib/features/auth/domain/usecases/user_login.dart:7-28)
- [lib/features/auth/domain/usecases/current_user.dart](../lib/features/auth/domain/usecases/current_user.dart)

**Blog Use Cases**:
- [lib/features/blog/domain/usecases/upload_blog.dart](../lib/features/blog/domain/usecases/upload_blog.dart:8-38)
- [lib/features/blog/domain/usecases/get_all_blogs.dart](../lib/features/blog/domain/usecases/get_all_blogs.dart)

### 💻 Code Example

**Base Interface**:

```dart
// lib/core/usecase/usecase.dart:4-8
abstract interface class UseCase<SuccessType, Params> {
  Future<Either<Failure, SuccessType>> call(Params params);
}

class NoParams {}
```

**Generic Types**:
- `SuccessType`: What you get back (User, Blog, List<Blog>)
- `Params`: What you need to provide (email/password, blog data, or NoParams)

**Example 1: UserLogin Use Case**

```dart
// lib/features/auth/domain/usecases/user_login.dart:7-28
class UserLogin implements UseCase<User, UserLoginParams> {
  final AuthRepository authRepository;
  const UserLogin(this.authRepository);

  @override
  Future<Either<Failure, User>> call(UserLoginParams params) async {
    return await authRepository.loginWithEmailPassword(
      email: params.email,
      password: params.password,
    );
  }
}

class UserLoginParams {
  final String email;
  final String password;

  UserLoginParams({
    required this.email,
    required this.password,
  });
}
```

**Usage**:
```dart
// In AuthBloc
final result = await _userLogin(
  UserLoginParams(email: 'user@example.com', password: 'password123'),
);
```

**Example 2: UploadBlog Use Case**

```dart
// lib/features/blog/domain/usecases/upload_blog.dart:8-38
class UploadBlog implements UseCase<Blog, UploadBlogParams> {
  final BlogRepository blogRepository;
  UploadBlog(this.blogRepository);

  @override
  Future<Either<Failure, Blog>> call(UploadBlogParams params) async {
    return await blogRepository.uploadBlog(
      image: params.image,
      title: params.title,
      content: params.content,
      posterId: params.posterId,
      topics: params.topics,
    );
  }
}

class UploadBlogParams {
  final String posterId;
  final String title;
  final String content;
  final File image;
  final List<String> topics;

  UploadBlogParams({
    required this.posterId,
    required this.title,
    required this.content,
    required this.image,
    required this.topics,
  });
}
```

**Why Params Classes?**

Instead of:
```dart
// ❌ Too many parameters - hard to read
Future<Either<Failure, Blog>> call(
  String posterId,
  String title,
  String content,
  File image,
  List<String> topics,
) { }
```

We use:
```dart
// ✅ Single parameter object - clean and organized
Future<Either<Failure, Blog>> call(UploadBlogParams params) { }
```

---

## BLoC Pattern (Observer)

### 🎯 What It Is

**BLoC** (Business Logic Component) is a state management pattern that:
- Separates business logic from UI
- Uses streams to communicate
- Follows the **Observer Pattern** (UI observes state changes)

**Flow**: `Events → BLoC → States → UI`

**Analogy**: Think of a **thermostat**:
- You (UI) set temperature (dispatch event)
- Thermostat (BLoC) processes request
- Heater/AC turns on (new state)
- Room temperature changes (UI updates)

### 🤔 Why We Use It

**Problem**: Where should you put UI logic?
- In the widget? ❌ (hard to test, hard to reuse)
- Scattered everywhere? ❌ (unmaintainable)

**Solution**: Centralize logic in BLoC!

**Benefits**:
- ✅ Testable (test BLoC without UI)
- ✅ Reusable (same BLoC for multiple UIs)
- ✅ Predictable (events in, states out)
- ✅ Debuggable (track all state changes)

### 📍 Where It's Implemented

**Auth BLoC**:
- [lib/features/auth/presentation/bloc/auth_bloc.dart](../lib/features/auth/presentation/bloc/auth_bloc.dart:13-88)
- [lib/features/auth/presentation/bloc/auth_event.dart](../lib/features/auth/presentation/bloc/auth_event.dart)
- [lib/features/auth/presentation/bloc/auth_state.dart](../lib/features/auth/presentation/bloc/auth_state.dart)

**Blog BLoC**:
- [lib/features/blog/presentation/bloc/blog_bloc.dart](../lib/features/blog/presentation/bloc/blog_bloc.dart)
- [lib/features/blog/presentation/bloc/blog_event.dart](../lib/features/blog/presentation/bloc/blog_event.dart)
- [lib/features/blog/presentation/bloc/blog_state.dart](../lib/features/blog/presentation/bloc/blog_state.dart)

**App User Cubit**:
- [lib/core/common/cubits/app_user/app_user_cubit.dart](../lib/core/common/cubits/app_user/app_user_cubit.dart:7-17)

### 💻 Code Example

**Events** (User Actions):

```dart
// What the user can do
abstract class AuthEvent {}

class AuthLogin extends AuthEvent {
  final String email;
  final String password;
  AuthLogin({required this.email, required this.password});
}

class AuthSignUp extends AuthEvent {
  final String name;
  final String email;
  final String password;
  // ...
}

class AuthIsUserLoggedIn extends AuthEvent {}
```

**States** (UI States):

```dart
// What the UI can display
abstract class AuthState {}

class AuthInitial extends AuthState {}

class AuthLoading extends AuthState {}

class AuthSuccess extends AuthState {
  final User user;
  AuthSuccess(this.user);
}

class AuthFailure extends AuthState {
  final String message;
  AuthFailure(this.message);
}
```

**BLoC** (Business Logic):

```dart
// lib/features/auth/presentation/bloc/auth_bloc.dart:13-88
class AuthBloc extends Bloc<AuthEvent, AuthState> {
  final UserSignUp _userSignUp;
  final UserLogin _userLogin;
  final CurrentUser _currentUser;
  final AppUserCubit _appUserCubit;

  AuthBloc({
    required UserSignUp userSignUp,
    required UserLogin userLogin,
    required CurrentUser currentUser,
    required AppUserCubit appUserCubit,
  })  : _userSignUp = userSignUp,
        _userLogin = userLogin,
        _currentUser = currentUser,
        _appUserCubit = appUserCubit,
        super(AuthInitial()) {
    // Register event handlers
    on<AuthEvent>((_, emit) => emit(AuthLoading()));
    on<AuthSignUp>(_onAuthSignUp);
    on<AuthLogin>(_onAuthLogin);
    on<AuthIsUserLoggedIn>(_isUserLoggedIn);
  }

  void _onAuthLogin(
    AuthLogin event,
    Emitter<AuthState> emit,
  ) async {
    // Call use case
    final res = await _userLogin(
      UserLoginParams(
        email: event.email,
        password: event.password,
      ),
    );

    // Handle result
    res.fold(
      (failure) => emit(AuthFailure(failure.message)),
      (user) => _emitAuthSuccess(user, emit),
    );
  }

  void _emitAuthSuccess(User user, Emitter<AuthState> emit) {
    _appUserCubit.updateUser(user);
    emit(AuthSuccess(user));
  }
}
```

**UI Usage**:

```dart
// Dispatch event
ElevatedButton(
  onPressed: () {
    context.read<AuthBloc>().add(
      AuthLogin(email: email, password: password),
    );
  },
  child: Text('Login'),
)

// Listen to states
BlocConsumer<AuthBloc, AuthState>(
  listener: (context, state) {
    if (state is AuthFailure) {
      showSnackBar(context, state.message);
    } else if (state is AuthSuccess) {
      Navigator.push(context, BlogPage.route());
    }
  },
  builder: (context, state) {
    if (state is AuthLoading) {
      return CircularProgressIndicator();
    }
    return LoginForm();
  },
)
```

**Observer Pattern**: UI observes BLoC state changes and reacts automatically!

---

## Either Monad (Functional Programming)

### 🎯 What It Is

**Either** is a type that represents a value of one of two possible types:
- **Left**: Typically an error/failure
- **Right**: Typically a success value

**Analogy**: Think of a **fork in the road**:
- You can only go left (failure) OR right (success)
- Never both, never neither

### 🤔 Why We Use It

**Problem With Traditional Error Handling**:

```dart
// ❌ Traditional way - can throw unexpected exceptions
Future<User> login(String email, String password) async {
  // What if network fails?
  // What if wrong password?
  // Caller might forget try-catch!
  throw Exception('Something went wrong');
}

// Usage - easy to forget error handling
try {
  final user = await login(email, password);
} catch (e) {
  // What type is e? String? Exception? Custom error?
}
```

**Solution With Either**:

```dart
// ✅ Errors are explicit in the type system
Future<Either<Failure, User>> login(
  String email,
  String password,
) async {
  // Returns Left(Failure) OR Right(User)
  // Compiler FORCES you to handle both!
}

// Usage - can't forget error handling
final result = await login(email, password);
result.fold(
  (failure) => print('Error: ${failure.message}'),
  (user) => print('Success: ${user.name}'),
);
```

**Benefits**:
- ✅ Type-safe error handling
- ✅ Compiler enforces handling both cases
- ✅ No uncaught exceptions
- ✅ Clear, explicit API

### 📍 Where It's Implemented

**Library Used**: `fpdart` package

**Used In**:
- All repository interfaces (return `Either<Failure, Data>`)
- All repository implementations
- All use cases
- BLoC fold() calls

### 💻 Code Example

**Failure Class**:

```dart
// lib/core/error/failures.dart:1-4
class Failure {
  final String message;
  Failure([this.message = 'An unexpected error occurred,']);
}
```

**Repository Method**:

```dart
// Returns Either - forces handling both cases
Future<Either<Failure, User>> loginWithEmailPassword({
  required String email,
  required String password,
}) async {
  try {
    final user = await remoteDataSource.login(email, password);
    return right(user); // Success case
  } on ServerException catch (e) {
    return left(Failure(e.message)); // Failure case
  }
}
```

**Usage in BLoC**:

```dart
final result = await _userLogin(params);

result.fold(
  // Left (failure) case
  (failure) => emit(AuthFailure(failure.message)),

  // Right (success) case
  (user) => emit(AuthSuccess(user)),
);
```

**Helper Functions**:
- `left(value)`: Create Left (failure)
- `right(value)`: Create Right (success)
- `fold(onLeft, onRight)`: Handle both cases

---

## Adapter Pattern

### 🎯 What It Is

The **Adapter Pattern** converts one interface into another. It "adapts" data from one format to another.

**Analogy**: Think of a **power adapter**:
- US plug (JSON data from API)
- European outlet (Dart objects in your app)
- Adapter converts between them

### 🤔 Why We Use It

**Problem**: APIs return JSON, but your app needs Dart objects.

**Solution**: Models act as adapters - they extend entities and add JSON conversion.

### 📍 Where It's Implemented

**Models** (adapt JSON ↔ Dart objects):
- [lib/features/auth/data/models/user_model.dart](../lib/features/auth/data/models/user_model.dart:1-29)
- [lib/features/blog/data/models/blog_model.dart](../lib/features/blog/data/models/blog_model.dart:3-62)

### 💻 Code Example

**Entity** (Pure Dart):

```dart
class User {
  final String id;
  final String email;
  final String name;

  User({required this.id, required this.email, required this.name});
}
```

**Model** (Adapter):

```dart
// lib/features/auth/data/models/user_model.dart:3-29
class UserModel extends User {
  UserModel({
    required super.id,
    required super.email,
    required super.name,
  });

  // Adapt JSON → Dart
  factory UserModel.fromJson(Map<String, dynamic> map) {
    return UserModel(
      id: map['id'] ?? '',
      email: map['email'] ?? '',
      name: map['name'] ?? '',
    );
  }

  // Adapt Dart → JSON
  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'email': email,
      'name': name,
    };
  }
}
```

**Usage**:

```dart
// Receiving from API
final jsonData = {'id': '123', 'email': 'user@example.com', 'name': 'John'};
final user = UserModel.fromJson(jsonData); // JSON → Dart

// Sending to API
final json = userModel.toJson(); // Dart → JSON
```

---

## Strategy Pattern

### 🎯 What It Is

The **Strategy Pattern** defines a family of algorithms and makes them interchangeable. You can switch strategies at runtime.

**Analogy**: Think of **payment methods**:
- You can pay with cash, credit card, or mobile payment
- The store doesn't care which you choose
- All methods accomplish the same goal (payment) differently

### 🤔 Why We Use It

**Problem**: How do you handle online vs offline data fetching?

**Solution**: Different strategies:
- **Online Strategy**: Fetch from remote API
- **Offline Strategy**: Load from local cache

### 📍 Where It's Implemented

**Data Sources** (different strategies):
- [lib/features/blog/data/datasources/blog_remote_data_source.dart](../lib/features/blog/data/datasources/blog_remote_data_source.dart) (Remote strategy)
- [lib/features/blog/data/datasources/blog_local_data_source.dart](../lib/features/blog/data/datasources/blog_local_data_source.dart) (Local strategy)

**Repository** (chooses strategy):
- [lib/features/blog/data/repositories/blog_repository_impl.dart](../lib/features/blog/data/repositories/blog_repository_impl.dart:63-75)

### 💻 Code Example

**Two Strategies**:

```dart
// Strategy 1: Remote (Supabase)
class BlogRemoteDataSource {
  Future<List<BlogModel>> getAllBlogs() async {
    // Fetch from Supabase API
  }
}

// Strategy 2: Local (Hive)
class BlogLocalDataSource {
  List<BlogModel> loadBlogs() {
    // Load from Hive database
  }
}
```

**Repository Chooses Strategy**:

```dart
// lib/features/blog/data/repositories/blog_repository_impl.dart:63-75
@override
Future<Either<Failure, List<Blog>>> getAllBlogs() async {
  try {
    if (!await connectionChecker.isConnected) {
      // Strategy 1: Offline - use local data source
      final blogs = blogLocalDataSource.loadBlogs();
      return right(blogs);
    }

    // Strategy 2: Online - use remote data source
    final blogs = await blogRemoteDataSource.getAllBlogs();

    // Cache for offline use
    blogLocalDataSource.uploadLocalBlogs(blogs: blogs);

    return right(blogs);
  } on ServerException catch (e) {
    return left(Failure(e.message));
  }
}
```

**At Runtime**: Repository decides which strategy to use based on network connectivity!

---

## Factory Pattern

### 🎯 What It Is

The **Factory Pattern** creates objects without specifying their exact class. It provides a method for creating instances.

**Analogy**: Think of a **car factory**:
- You order a "car" (generic)
- Factory decides whether to build a sedan, SUV, or truck based on specifications
- You get a car without worrying about construction details

### 🤔 Why We Use It

**Problem**: Creating objects can be complex (parsing JSON, validation, defaults).

**Solution**: Factory methods encapsulate object creation logic.

### 📍 Where It's Implemented

**Model Factory Constructors**:
- `UserModel.fromJson()` - Creates UserModel from JSON
- `BlogModel.fromJson()` - Creates BlogModel from JSON

**Route Factories**:
- `LoginPage.route()` - Creates MaterialPageRoute
- `BlogPage.route()` - Creates MaterialPageRoute

### 💻 Code Example

**Factory Constructor**:

```dart
class BlogModel extends Blog {
  // Regular constructor
  BlogModel({required super.id, ...});

  // Factory constructor - creates instance from JSON
  factory BlogModel.fromJson(Map<String, dynamic> map) {
    return BlogModel(
      id: map['id'] as String,
      posterId: map['poster_id'] as String,
      title: map['title'] as String,
      content: map['content'] as String,
      imageUrl: map['image_url'] as String,
      topics: List<String>.from(map['topics'] ?? []),
      updatedAt: map['updated_at'] == null
          ? DateTime.now()
          : DateTime.parse(map['updated_at']),
    );
  }
}
```

**Route Factory**:

```dart
class LoginPage extends StatefulWidget {
  // Factory method for creating routes
  static route() => MaterialPageRoute(
    builder: (context) => const LoginPage(),
  );
}

// Usage
Navigator.push(context, LoginPage.route());
```

---

## Singleton Pattern

### 🎯 What It Is

The **Singleton Pattern** ensures a class has only ONE instance and provides a global access point.

**Analogy**: Think of the **sun**:
- There's only one sun in our solar system
- Everyone sees the same sun
- You can't create another sun

### 🤔 Why We Use It

**Problem**: Some objects should only exist once (database connection, app configuration, user session).

**Solution**: Use singleton to ensure single instance.

### 📍 Where It's Implemented

**Service Locator**:
- [lib/init_dependencies.main.dart:3](../lib/init_dependencies.main.dart:3) - `GetIt.instance` is a singleton

**Registered as Singletons**:
- `SupabaseClient` - One instance for the entire app
- `AppUserCubit` - Global user state
- `AuthBloc` - One instance
- `BlogBloc` - One instance

### 💻 Code Example

**GetIt Singleton**:

```dart
// lib/init_dependencies.main.dart:3
final serviceLocator = GetIt.instance; // Singleton
```

**Lazy Singleton Registration**:

```dart
// Created once, on first use
serviceLocator.registerLazySingleton(() => supabase.client);
serviceLocator.registerLazySingleton(() => AppUserCubit());
serviceLocator.registerLazySingleton(() => AuthBloc(...));
```

**Usage**:

```dart
// Always returns the SAME instance
final bloc1 = serviceLocator<AuthBloc>();
final bloc2 = serviceLocator<AuthBloc>();

print(bloc1 == bloc2); // true - same object!
```

---

## Summary Table

| Pattern | Purpose | Where Used | Key Benefit |
|---------|---------|------------|-------------|
| **Repository** | Abstract data access | Domain & Data layers | Easy to swap data sources |
| **Dependency Injection** | Provide dependencies externally | `init_dependencies.dart` | Loose coupling, testability |
| **Use Case** | Single business operation | Domain layer | Single responsibility |
| **BLoC (Observer)** | Separate UI from logic | Presentation layer | Testable, predictable state |
| **Either Monad** | Type-safe error handling | Repositories, Use Cases | No uncaught exceptions |
| **Adapter** | Convert data formats | Models | JSON ↔ Dart objects |
| **Strategy** | Interchangeable algorithms | Remote vs Local data | Runtime flexibility |
| **Factory** | Object creation | `fromJson()`, `route()` | Encapsulate creation logic |
| **Singleton** | Single instance | Service locator, BLoCs | Shared state, efficiency |

---

## What's Next?

Now that you understand all design patterns, continue to:

**[05-FEATURE-DEEP-DIVE.md](05-FEATURE-DEEP-DIVE.md)** - Complete walkthroughs of Auth and Blog features

---

**You're becoming an architecture expert! 🎨**
