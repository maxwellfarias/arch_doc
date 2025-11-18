# Layer Breakdown - Deep Dive

## Table of Contents
1. [Introduction](#introduction)
2. [Domain Layer](#domain-layer)
3. [Data Layer](#data-layer)
4. [Presentation Layer](#presentation-layer)
5. [Core Layer](#core-layer)
6. [Layer Communication](#layer-communication)
7. [Complete Example](#complete-example)

---

## Introduction

This document provides an **in-depth exploration** of each layer in this Clean Architecture project. We'll examine:
- What each layer contains
- The responsibility of each component
- Real code examples with line references
- How components interact

Think of this as a **detailed map** of the codebase - after reading this, you'll know exactly where to find any piece of code and why it's there.

---

## Domain Layer

**Location**: `lib/features/{feature}/domain/`

**Purpose**: Pure business logic, independent of any framework or external service

**Think of it as**: The constitution of your app - fundamental rules that never change

### Components

#### 1. Entities

**What are they?**: Pure data classes representing core business objects

**Characteristics**:
- ✅ Immutable (all fields are `final`)
- ✅ No methods except constructors
- ✅ No JSON parsing, no API calls, no database code
- ✅ Pure Dart - no Flutter dependencies

**Example 1: User Entity**

```dart
// lib/core/common/entities/user.dart:1-11
class User {
  final String id;
  final String email;
  final String name;

  User({
    required this.id,
    required this.email,
    required this.name,
  });
}
```

**Why it's in Domain**: A "User" is a core concept in any application with authentication. This definition doesn't care:
- Where the user data comes from (API, database, local storage)
- How it's displayed (text, card, avatar)
- What framework you're using (Flutter, React, etc.)

**Example 2: Blog Entity**

```dart
// lib/features/blog/domain/entities/blog.dart:1-21
class Blog {
  final String id;
  final String posterId;
  final String title;
  final String content;
  final String imageUrl;
  final List<String> topics;
  final DateTime updatedAt;
  final String? posterName;

  Blog({
    required this.id,
    required this.posterId,
    required this.title,
    required this.content,
    required this.imageUrl,
    required this.topics,
    required this.updatedAt,
    this.posterName,
  });
}
```

**Key Points**:
- `posterName` is nullable (`String?`) because it might not always be loaded
- All other fields are required
- `topics` is a list - a blog can have multiple topics
- This is just a data container - no logic!

---

#### 2. Repository Interfaces

**What are they?**: Contracts that define what data operations are needed, not how to do them

**Analogy**: Like a restaurant menu - it lists what you can order, not how the chef makes it

**Characteristics**:
- ✅ Abstract interface classes (no implementation)
- ✅ Return `Either<Failure, Data>` for type-safe error handling
- ✅ Async operations (`Future<...>`)
- ✅ Define method signatures only

**Example 1: Auth Repository Interface**

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

**Reading this code**:
- `abstract interface class`: Can't be instantiated, must be implemented
- `Future<Either<Failure, User>>`: Returns either an error (Failure) or success (User)
- Named parameters (`required String email`): Clear, self-documenting
- Three methods: sign up, login, get current user

**Why `Either<Failure, User>`?**

Traditional error handling:
```dart
// ❌ Traditional way - can throw unexpected exceptions
Future<User> login(String email, String password) async {
  // What if network fails? What if wrong password?
  // You need try-catch everywhere!
}
```

Clean Architecture way:
```dart
// ✅ Clean Architecture - errors are explicit
Future<Either<Failure, User>> login(String email, String password) async {
  // Returns Left(Failure) on error
  // Returns Right(User) on success
  // Compiler forces you to handle both cases!
}
```

**Example 2: Blog Repository Interface**

```dart
// Simplified from lib/features/blog/domain/repositories/blog_repository.dart
abstract interface class BlogRepository {
  Future<Either<Failure, Blog>> uploadBlog({
    required File image,
    required String title,
    required String content,
    required String posterId,
    required List<String> topics,
  });

  Future<Either<Failure, List<Blog>>> getAllBlogs();
}
```

Notice how `getAllBlogs()` returns a **list** of blogs: `Either<Failure, List<Blog>>`

---

#### 3. Use Cases

**What are they?**: Single-purpose business operations

**Analogy**: Each use case is like a single button on a vending machine - it does ONE thing

**Characteristics**:
- ✅ One class = one business operation
- ✅ Implements `UseCase<SuccessType, Params>` interface
- ✅ Has a `call()` method (makes the class callable like a function)
- ✅ Depends only on repository interfaces (via constructor injection)

**Base UseCase Interface**:

```dart
// lib/core/usecase/usecase.dart:4-8
abstract interface class UseCase<SuccessType, Params> {
  Future<Either<Failure, SuccessType>> call(Params params);
}

class NoParams {}
```

**Generic Types**:
- `SuccessType`: What you get back on success (User, Blog, List<Blog>, etc.)
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

**Breaking it down**:
1. **Implements UseCase<User, UserLoginParams>**: Returns User, takes UserLoginParams
2. **Constructor injection**: `AuthRepository` is injected (provided by dependency injection)
3. **call() method**: Does the actual work - delegates to repository
4. **Params class**: Encapsulates all needed parameters

**Why a separate Params class?**
- Clean separation of concerns
- Easy to add new parameters later
- Type-safe parameter passing

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

**Notice**: This use case takes more parameters (title, content, image, topics) but still follows the same pattern!

**All Use Cases in This Project**:

1. **UserSignUp** - Register a new user
2. **UserLogin** - Authenticate a user
3. **CurrentUser** - Get currently logged-in user
4. **UploadBlog** - Create a new blog post
5. **GetAllBlogs** - Fetch all blog posts

Each one has **one job**, making the code:
- Easy to understand
- Easy to test
- Easy to reuse

---

## Data Layer

**Location**: `lib/features/{feature}/data/`

**Purpose**: Handle all data operations - API calls, database, caching

**Think of it as**: The workers - they know HOW to get things done

### Components

#### 1. Models

**What are they?**: Data transfer objects that extend entities and add serialization capabilities

**Key Difference from Entities**:
- **Entities**: Pure business objects (in Domain)
- **Models**: Entities + JSON conversion (in Data)

**Characteristics**:
- ✅ Extend domain entities
- ✅ Add `fromJson()` factory constructor
- ✅ Add `toJson()` method
- ✅ Add `copyWith()` for immutable updates

**Example 1: UserModel**

```dart
// lib/features/auth/data/models/user_model.dart:1-29
class UserModel extends User {
  UserModel({
    required super.id,
    required super.email,
    required super.name,
  });

  factory UserModel.fromJson(Map<String, dynamic> map) {
    return UserModel(
      id: map['id'] ?? '',
      email: map['email'] ?? '',
      name: map['name'] ?? '',
    );
  }

  UserModel copyWith({
    String? id,
    String? email,
    String? name,
  }) {
    return UserModel(
      id: id ?? this.id,
      email: email ?? this.email,
      name: name ?? this.name,
    );
  }
}
```

**Breaking it down**:
- `extends User`: Inherits all fields from User entity
- `super.id, super.email, super.name`: Pass to parent constructor
- `fromJson()`: Convert JSON map → UserModel
- `copyWith()`: Create a new instance with some fields changed

**Why copyWith()?**

Since all fields are `final` (immutable), we can't do this:
```dart
user.name = "New Name"; // ❌ Error: can't modify final field
```

Instead, we create a new instance:
```dart
final updatedUser = user.copyWith(name: "New Name"); // ✅ Works!
```

**Example 2: BlogModel**

```dart
// lib/features/blog/data/models/blog_model.dart:3-62
class BlogModel extends Blog {
  BlogModel({
    required super.id,
    required super.posterId,
    required super.title,
    required super.content,
    required super.imageUrl,
    required super.topics,
    required super.updatedAt,
    super.posterName,
  });

  Map<String, dynamic> toJson() {
    return <String, dynamic>{
      'id': id,
      'poster_id': posterId,
      'title': title,
      'content': content,
      'image_url': imageUrl,
      'topics': topics,
      'updated_at': updatedAt.toIso8601String(),
    };
  }

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

  BlogModel copyWith({
    String? id,
    String? posterId,
    String? title,
    String? content,
    String? imageUrl,
    List<String>? topics,
    DateTime? updatedAt,
    String? posterName,
  }) {
    return BlogModel(
      id: id ?? this.id,
      posterId: posterId ?? this.posterId,
      title: title ?? this.title,
      content: content ?? this.content,
      imageUrl: imageUrl ?? this.imageUrl,
      topics: topics ?? this.topics,
      updatedAt: updatedAt ?? this.updatedAt,
      posterName: posterName ?? this.posterName,
    );
  }
}
```

**Key Points**:
- `toJson()`: Converts BlogModel → JSON (for sending to API)
- `fromJson()`: Converts JSON → BlogModel (for receiving from API)
- Field name mapping: `posterId` ↔ `poster_id` (Dart camelCase ↔ SQL snake_case)
- `topics` is an array: `List<String>.from(map['topics'] ?? [])`
- DateTime handling: `DateTime.parse(map['updated_at'])`

---

#### 2. Data Sources

**What are they?**: Classes that fetch data from specific sources (API, database, cache)

**Types in this project**:
- **Remote Data Sources**: Fetch from Supabase API
- **Local Data Sources**: Fetch from Hive database

**Characteristics**:
- ✅ Direct interaction with external services
- ✅ Throw exceptions on errors (caught by repository)
- ✅ Return models (not entities)
- ✅ Handle low-level details (HTTP requests, SQL queries, etc.)

**Example: Remote vs Local**

```
BlogRemoteDataSource       BlogLocalDataSource
        ↓                          ↓
   Supabase API              Hive Database
  (Cloud storage)          (Local storage)
        ↓                          ↓
   Fresh data              Cached data
  (requires network)      (works offline)
```

**Typical Remote Data Source Methods**:
- `signUpWithEmailPassword()` - Call Supabase auth API
- `loginWithEmailPassword()` - Call Supabase auth API
- `uploadBlog()` - Insert blog into Supabase database
- `getAllBlogs()` - Query blogs from Supabase
- `uploadBlogImage()` - Upload image to Supabase storage

**Typical Local Data Source Methods**:
- `loadBlogs()` - Read blogs from Hive box
- `uploadLocalBlogs()` - Save blogs to Hive box for offline access

---

#### 3. Repository Implementations

**What are they?**: Concrete implementations of domain repository interfaces

**Responsibility**: Coordinate between data sources, handle errors, check network connectivity

**Characteristics**:
- ✅ Implements domain repository interface
- ✅ Depends on data sources (injected via constructor)
- ✅ Converts exceptions → Failures
- ✅ Returns `Either<Failure, Data>`
- ✅ Implements offline-first strategy (if applicable)

**Example 1: AuthRepositoryImpl**

```dart
// lib/features/auth/data/repositories/auth_repository_impl.dart:11-46
class AuthRepositoryImpl implements AuthRepository {
  final AuthRemoteDataSource remoteDataSource;
  final ConnectionChecker connectionChecker;

  const AuthRepositoryImpl(
    this.remoteDataSource,
    this.connectionChecker,
  );

  @override
  Future<Either<Failure, User>> currentUser() async {
    try {
      if (!await (connectionChecker.isConnected)) {
        final session = remoteDataSource.currentUserSession;

        if (session == null) {
          return left(Failure('User not logged in!'));
        }

        return right(
          UserModel(
            id: session.user.id,
            email: session.user.email ?? '',
            name: '',
          ),
        );
      }

      final user = await remoteDataSource.getCurrentUserData();
      if (user == null) {
        return left(Failure('User not logged in!'));
      }

      return right(user);
    } on ServerException catch (e) {
      return left(Failure(e.message));
    }
  }
}
```

**Breaking it down**:
1. **Implements AuthRepository**: Fulfills the contract defined in domain
2. **Constructor injection**: Receives dependencies
3. **Network check**: `if (!await connectionChecker.isConnected)`
4. **Offline fallback**: Get user from session if offline
5. **Error handling**: `try-catch` converts `ServerException` → `Failure`
6. **Returns Either**: `left(Failure)` or `right(User)`

**Example 2: BlogRepositoryImpl with Offline Strategy**

```dart
// lib/features/blog/data/repositories/blog_repository_impl.dart:14-76
class BlogRepositoryImpl implements BlogRepository {
  final BlogRemoteDataSource blogRemoteDataSource;
  final BlogLocalDataSource blogLocalDataSource;
  final ConnectionChecker connectionChecker;

  BlogRepositoryImpl(
    this.blogRemoteDataSource,
    this.blogLocalDataSource,
    this.connectionChecker,
  );

  @override
  Future<Either<Failure, List<Blog>>> getAllBlogs() async {
    try {
      if (!await (connectionChecker.isConnected)) {
        // OFFLINE: Load from local cache
        final blogs = blogLocalDataSource.loadBlogs();
        return right(blogs);
      }

      // ONLINE: Fetch from remote
      final blogs = await blogRemoteDataSource.getAllBlogs();

      // Save to local cache for offline use
      blogLocalDataSource.uploadLocalBlogs(blogs: blogs);

      return right(blogs);
    } on ServerException catch (e) {
      return left(Failure(e.message));
    }
  }
}
```

**Offline-First Strategy**:
1. Check network connectivity
2. **If offline**: Load from local Hive database
3. **If online**: Fetch from Supabase, then cache locally
4. User gets data either way!

**Example 3: UploadBlog with Image Upload**

```dart
// lib/features/blog/data/repositories/blog_repository_impl.dart:24-60
@override
Future<Either<Failure, Blog>> uploadBlog({
  required File image,
  required String title,
  required String content,
  required String posterId,
  required List<String> topics,
}) async {
  try {
    if (!await (connectionChecker.isConnected)) {
      return left(Failure(Constants.noConnectionErrorMessage));
    }

    // Step 1: Create blog model with temporary empty imageUrl
    BlogModel blogModel = BlogModel(
      id: const Uuid().v1(),
      posterId: posterId,
      title: title,
      content: content,
      imageUrl: '',
      topics: topics,
      updatedAt: DateTime.now(),
    );

    // Step 2: Upload image to Supabase Storage
    final imageUrl = await blogRemoteDataSource.uploadBlogImage(
      image: image,
      blog: blogModel,
    );

    // Step 3: Update blog model with actual image URL
    blogModel = blogModel.copyWith(
      imageUrl: imageUrl,
    );

    // Step 4: Upload blog to Supabase database
    final uploadedBlog = await blogRemoteDataSource.uploadBlog(blogModel);

    return right(uploadedBlog);
  } on ServerException catch (e) {
    return left(Failure(e.message));
  }
}
```

**Multi-step process**:
1. Generate UUID for blog ID
2. Create temporary blog model (empty imageUrl)
3. Upload image → get URL
4. Update blog model with real image URL
5. Upload blog to database
6. Return result

---

## Presentation Layer

**Location**: `lib/features/{feature}/presentation/`

**Purpose**: Handle UI and user interactions

**Think of it as**: The face of the app - what users see and touch

### Components

#### 1. BLoC (Business Logic Component)

**What is BLoC?**: State management pattern that separates business logic from UI

**Flow**: `Events → BLoC → States → UI`

```
User taps button → Dispatch Event → BLoC processes → Emit State → UI updates
```

**Structure**:
- **Events**: User actions (AuthLogin, AuthSignUp, BlogFetchAllBlogs)
- **States**: UI states (Loading, Success, Failure)
- **BLoC**: Coordinates between events and states, calls use cases

**Example: AuthBloc**

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
    // Update global user state
    _appUserCubit.updateUser(user);
    // Emit success state
    emit(AuthSuccess(user));
  }
}
```

**Key Concepts**:
1. **Constructor injection**: All use cases are injected
2. **Initial state**: `super(AuthInitial())`
3. **Event handlers**: `on<AuthLogin>(_onAuthLogin)`
4. **Emit states**: `emit(AuthLoading())`, `emit(AuthSuccess(user))`
5. **fold()**: Handle Either - left = failure, right = success

**Events** (user actions):
```dart
abstract class AuthEvent {}

class AuthLogin extends AuthEvent {
  final String email;
  final String password;
  AuthLogin({required this.email, required this.password});
}

class AuthSignUp extends AuthEvent {
  final String email;
  final String password;
  final String name;
  // ...
}
```

**States** (UI states):
```dart
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

---

#### 2. Pages

**What are they?**: Full-screen views that represent app screens

**Characteristics**:
- ✅ StatefulWidget or StatelessWidget
- ✅ Use BlocProvider/BlocConsumer to interact with BLoC
- ✅ Display data from states
- ✅ Dispatch events on user interaction

**Example: Simplified LoginPage**

```dart
class LoginPage extends StatefulWidget {
  static route() => MaterialPageRoute(
    builder: (context) => const LoginPage(),
  );

  @override
  State<LoginPage> createState() => _LoginPageState();
}

class _LoginPageState extends State<LoginPage> {
  final emailController = TextEditingController();
  final passwordController = TextEditingController();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: BlocConsumer<AuthBloc, AuthState>(
        listener: (context, state) {
          // React to state changes
          if (state is AuthFailure) {
            showSnackBar(context, state.message);
          } else if (state is AuthSuccess) {
            Navigator.pushAndRemoveUntil(
              context,
              BlogPage.route(),
              (_) => false,
            );
          }
        },
        builder: (context, state) {
          // Build UI based on state
          if (state is AuthLoading) {
            return const Loader();
          }

          return Column(
            children: [
              AuthField(
                hintText: 'Email',
                controller: emailController,
              ),
              AuthField(
                hintText: 'Password',
                controller: passwordController,
                isObscureText: true,
              ),
              AuthGradientButton(
                buttonText: 'Sign in',
                onPressed: () {
                  // Dispatch login event
                  context.read<AuthBloc>().add(
                    AuthLogin(
                      email: emailController.text.trim(),
                      password: passwordController.text.trim(),
                    ),
                  );
                },
              ),
            ],
          );
        },
      ),
    );
  }
}
```

**BlocConsumer**:
- **listener**: Side effects (navigation, show errors)
- **builder**: Build UI based on state

---

#### 3. Widgets

**What are they?**: Reusable UI components

**Examples in this project**:
- `AuthField` - Text input field for auth forms
- `AuthGradientButton` - Gradient button for auth actions
- `BlogCard` - Card displaying blog preview
- `BlogEditor` - Text editor for blog content
- `Loader` - Loading spinner

**Benefits**:
- Reusability (use AuthField in login AND signup)
- Consistency (all buttons look the same)
- Maintainability (change style in one place)

---

## Core Layer

**Location**: `lib/core/`

**Purpose**: Shared code used across multiple features

**Think of it as**: The utility closet - tools used by everyone

### Components

#### 1. Common (Shared Entities, Widgets, Cubits)

**AppUserCubit** - Global user state:

```dart
// lib/core/common/cubits/app_user/app_user_cubit.dart:7-17
class AppUserCubit extends Cubit<AppUserState> {
  AppUserCubit() : super(AppUserInitial());

  void updateUser(User? user) {
    if (user == null) {
      emit(AppUserInitial());
    } else {
      emit(AppUserLoggedIn(user));
    }
  }
}
```

**Purpose**: Manage logged-in user state globally (used by both auth and blog features)

---

#### 2. Error Handling

**Failure Class**:

```dart
// lib/core/error/failures.dart:1-4
class Failure {
  final String message;
  Failure([this.message = 'An unexpected error occurred,']);
}
```

**Simple but powerful**: Wraps error messages for type-safe error handling

---

#### 3. Base UseCase Interface

```dart
// lib/core/usecase/usecase.dart:4-8
abstract interface class UseCase<SuccessType, Params> {
  Future<Either<Failure, SuccessType>> call(Params params);
}

class NoParams {}
```

All use cases implement this interface for consistency.

---

#### 4. Utils

**Example: Calculate Reading Time**

```dart
// lib/core/utils/calculate_reading_time.dart:1-7
int calculateReadingTime(String content) {
  final wordCount = content.split(RegExp(r'\s+')).length;
  final readingTime = wordCount / 225;
  return readingTime.ceil();
}
```

**Logic**: Assumes average reading speed of 225 words per minute

---

## Layer Communication

### Rule 1: Dependencies Point Inward

```
Presentation → Data → Domain
```

- Presentation imports from Data and Domain
- Data imports from Domain
- Domain imports from NOTHING (except core)

### Rule 2: Never Skip Layers

❌ **Wrong**: Presentation calls Data Source directly

```dart
// ❌ BAD: UI calling data source directly
final user = await authRemoteDataSource.login(email, password);
```

✅ **Right**: Presentation → BLoC → UseCase → Repository → Data Source

```dart
// ✅ GOOD: Follow the layers
context.read<AuthBloc>().add(AuthLogin(email, password));
// → BLoC calls UserLogin use case
// → Use case calls AuthRepository
// → Repository calls AuthRemoteDataSource
```

---

## Complete Example

Let's trace **"User logs in"** through all layers:

### 1. Presentation Layer

**User Action** (LoginPage):
```dart
// User taps button
context.read<AuthBloc>().add(
  AuthLogin(email: email, password: password),
);
```

**BLoC Handles Event** (AuthBloc):
```dart
void _onAuthLogin(AuthLogin event, Emitter<AuthState> emit) async {
  emit(AuthLoading()); // Show loading spinner

  final res = await _userLogin(
    UserLoginParams(email: event.email, password: event.password),
  );

  res.fold(
    (failure) => emit(AuthFailure(failure.message)),
    (user) => emit(AuthSuccess(user)),
  );
}
```

### 2. Domain Layer

**Use Case Executes** (UserLogin):
```dart
Future<Either<Failure, User>> call(UserLoginParams params) async {
  return await authRepository.loginWithEmailPassword(
    email: params.email,
    password: params.password,
  );
}
```

### 3. Data Layer

**Repository Implements** (AuthRepositoryImpl):
```dart
Future<Either<Failure, User>> loginWithEmailPassword(...) async {
  try {
    if (!await connectionChecker.isConnected) {
      return left(Failure('No internet connection'));
    }

    final user = await remoteDataSource.loginWithEmailPassword(...);
    return right(user);
  } on ServerException catch (e) {
    return left(Failure(e.message));
  }
}
```

**Data Source Makes API Call** (AuthRemoteDataSource):
```dart
final response = await supabaseClient.auth.signInWithPassword(
  email: email,
  password: password,
);
return UserModel.fromJson(response.user);
```

### 4. Back Up the Layers

**Data** → Returns `Right(UserModel)` to Repository
**Repository** → Returns `Right(User)` to Use Case
**Use Case** → Returns `Right(User)` to BLoC
**BLoC** → Emits `AuthSuccess(user)` state
**UI** → Listens to state, navigates to BlogPage

---

## Summary

**Domain Layer**:
- Entities (User, Blog)
- Repository Interfaces (AuthRepository, BlogRepository)
- Use Cases (UserLogin, UploadBlog)
- Pure logic, no dependencies

**Data Layer**:
- Models (UserModel, BlogModel) - Entities + JSON
- Data Sources (Remote: Supabase, Local: Hive)
- Repository Implementations (AuthRepositoryImpl, BlogRepositoryImpl)
- Handles API calls, caching, errors

**Presentation Layer**:
- BLoC (AuthBloc, BlogBloc) - State management
- Pages (LoginPage, BlogPage) - Full screens
- Widgets (AuthField, BlogCard) - Reusable components
- Displays UI, captures user input

**Core Layer**:
- Shared entities, widgets, cubits
- Error handling (Failure class)
- Base interfaces (UseCase)
- Utility functions

---

## What's Next?

Now that you understand each layer in detail, continue to:

**[04-DESIGN-PATTERNS-GUIDE.md](04-DESIGN-PATTERNS-GUIDE.md)** - Explore all design patterns used in this project

---

**You're making great progress! 🎉**
