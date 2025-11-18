# Feature Deep-Dive - Complete Walkthroughs

## Table of Contents
1. [Introduction](#introduction)
2. [Authentication Feature](#authentication-feature)
3. [Blog Feature](#blog-feature)
4. [Offline Strategy](#offline-strategy)
5. [Global State Management](#global-state-management)
6. [Summary](#summary)

---

## Introduction

This document provides **complete end-to-end walkthroughs** of the two main features in this application:

1. **Authentication Feature**: User registration, login, session management
2. **Blog Feature**: Creating, uploading, and viewing blog posts

For each feature, we'll trace the complete flow from:
- **UI Interaction** → User taps a button
- **Presentation Layer** → BLoC handles the event
- **Domain Layer** → Use case executes business logic
- **Data Layer** → Repository fetches/stores data
- **Back to UI** → State update triggers UI refresh

Think of this as **following a package** through the postal system from sender to recipient!

---

## Authentication Feature

**Location**: `lib/features/auth/`

**Purpose**: Handle user registration, login, and session management

### Feature Structure

```
features/auth/
├── data/
│   ├── datasources/
│   │   └── auth_remote_data_source.dart    # Supabase API calls
│   ├── models/
│   │   └── user_model.dart                 # User + JSON serialization
│   └── repositories/
│       └── auth_repository_impl.dart       # Repository implementation
├── domain/
│   ├── repository/
│   │   └── auth_repository.dart            # Repository interface
│   └── usecases/
│       ├── current_user.dart               # Get logged-in user
│       ├── user_login.dart                 # Login use case
│       └── user_sign_up.dart               # Sign up use case
└── presentation/
    ├── bloc/
    │   ├── auth_bloc.dart                  # State management
    │   ├── auth_event.dart                 # User actions
    │   └── auth_state.dart                 # UI states
    ├── pages/
    │   ├── login_page.dart                 # Login screen
    │   └── signup_page.dart                # Registration screen
    └── widgets/
        ├── auth_field.dart                 # Text input field
        └── auth_gradient_button.dart       # Submit button
```

---

### Use Case 1: User Sign Up

**Scenario**: A new user wants to create an account

#### Step-by-Step Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 1: User fills form and taps "Sign Up" button                   │
│ Location: features/auth/presentation/pages/signup_page.dart         │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 2: SignupPage dispatches AuthSignUp event to AuthBloc          │
│                                                                      │
│ Code:                                                                │
│   context.read<AuthBloc>().add(                                     │
│     AuthSignUp(                                                      │
│       name: nameController.text.trim(),                             │
│       email: emailController.text.trim(),                           │
│       password: passwordController.text.trim(),                     │
│     ),                                                               │
│   );                                                                 │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 3: AuthBloc receives event, emits AuthLoading state            │
│ Location: features/auth/presentation/bloc/auth_bloc.dart:28         │
│                                                                      │
│ Code:                                                                │
│   on<AuthEvent>((_, emit) => emit(AuthLoading()));                  │
│                                                                      │
│ UI shows loading spinner                                            │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 4: AuthBloc calls UserSignUp use case                          │
│ Location: features/auth/presentation/bloc/auth_bloc.dart:46-62      │
│                                                                      │
│ Code:                                                                │
│   void _onAuthSignUp(AuthSignUp event, Emitter<AuthState> emit) {   │
│     final res = await _userSignUp(                                  │
│       UserSignUpParams(                                              │
│         email: event.email,                                          │
│         password: event.password,                                    │
│         name: event.name,                                            │
│       ),                                                             │
│     );                                                               │
│   }                                                                  │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 5: UserSignUp use case calls AuthRepository                    │
│ Location: features/auth/domain/usecases/user_sign_up.dart:7-18      │
│                                                                      │
│ Code:                                                                │
│   class UserSignUp implements UseCase<User, UserSignUpParams> {     │
│     final AuthRepository authRepository;                            │
│                                                                      │
│     Future<Either<Failure, User>> call(UserSignUpParams params) {   │
│       return await authRepository.signUpWithEmailPassword(          │
│         name: params.name,                                           │
│         email: params.email,                                         │
│         password: params.password,                                   │
│       );                                                             │
│     }                                                                │
│   }                                                                  │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 6: AuthRepositoryImpl checks network connectivity              │
│ Location: features/auth/data/repositories/auth_repository_impl.dart │
│                                                                      │
│ Code:                                                                │
│   if (!await connectionChecker.isConnected) {                       │
│     return left(Failure(Constants.noConnectionErrorMessage));       │
│   }                                                                  │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 7: Repository calls AuthRemoteDataSource                       │
│                                                                      │
│ Code:                                                                │
│   final user = await remoteDataSource.signUpWithEmailPassword(      │
│     name: name,                                                      │
│     email: email,                                                    │
│     password: password,                                              │
│   );                                                                 │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 8: RemoteDataSource makes Supabase API call                    │
│ Location: features/auth/data/datasources/auth_remote_data_source    │
│                                                                      │
│ Operations:                                                          │
│   1. Call Supabase Auth: signUp(email, password)                   │
│   2. Insert user profile into 'profiles' table (id, name, email)    │
│   3. Fetch complete user data from 'profiles' table                 │
│   4. Convert JSON response → UserModel                              │
│                                                                      │
│ Returns: UserModel                                                   │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 9: Data flows back up through layers                           │
│                                                                      │
│   DataSource → Repository → UseCase → BLoC                          │
│                                                                      │
│ Repository wraps in Either:                                          │
│   Success: Right(UserModel) // extends User                         │
│   Error: Left(Failure('error message'))                             │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 10: AuthBloc handles result with fold()                        │
│ Location: features/auth/presentation/bloc/auth_bloc.dart:58-61      │
│                                                                      │
│ Code:                                                                │
│   res.fold(                                                          │
│     (failure) => emit(AuthFailure(failure.message)),                │
│     (user) => _emitAuthSuccess(user, emit),                         │
│   );                                                                 │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 11: On success, update global user state                       │
│ Location: features/auth/presentation/bloc/auth_bloc.dart:81-87      │
│                                                                      │
│ Code:                                                                │
│   void _emitAuthSuccess(User user, Emitter<AuthState> emit) {       │
│     _appUserCubit.updateUser(user);  // Update global state         │
│     emit(AuthSuccess(user));          // Emit local state           │
│   }                                                                  │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 12: UI listens to state change                                 │
│ Location: features/auth/presentation/pages/signup_page.dart         │
│                                                                      │
│ Code:                                                                │
│   BlocConsumer<AuthBloc, AuthState>(                                │
│     listener: (context, state) {                                    │
│       if (state is AuthSuccess) {                                   │
│         // Navigate to BlogPage                                     │
│         Navigator.pushAndRemoveUntil(                               │
│           context,                                                   │
│           BlogPage.route(),                                          │
│           (_) => false,                                              │
│         );                                                           │
│       } else if (state is AuthFailure) {                            │
│         showSnackBar(context, state.message);                       │
│       }                                                              │
│     },                                                               │
│   )                                                                  │
└──────────────────────────────────────────────────────────────────────┘
```

#### Database Operations

**Supabase Tables Used**:

1. **auth.users** (Managed by Supabase Auth)
   - Stores email, encrypted password, session tokens
   - Created automatically by `signUp()`

2. **profiles** (Custom table)
   ```sql
   CREATE TABLE profiles (
     id UUID PRIMARY KEY REFERENCES auth.users(id),
     name TEXT NOT NULL,
     email TEXT NOT NULL,
     created_at TIMESTAMP DEFAULT NOW()
   );
   ```

**What gets stored**:
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "john@example.com",
  "name": "John Doe",
  "created_at": "2024-01-15T10:30:00Z"
}
```

---

### Use Case 2: User Login

**Scenario**: Existing user wants to log in

#### Flow Comparison

The login flow is **very similar** to sign up, with these differences:

| Step | Sign Up | Login |
|------|---------|-------|
| Event | `AuthSignUp` | `AuthLogin` |
| Use Case | `UserSignUp` | `UserLogin` |
| Params | name, email, password | email, password |
| Data Source Method | `signUpWithEmailPassword()` | `loginWithEmailPassword()` |
| Supabase API | `auth.signUp()` | `auth.signInWithPassword()` |
| Database Operation | INSERT into profiles | SELECT from profiles |

#### Code: UserLogin Use Case

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

#### Supabase Authentication

```dart
// Simplified from auth_remote_data_source.dart
Future<UserModel> loginWithEmailPassword({
  required String email,
  required String password,
}) async {
  try {
    // Step 1: Authenticate with Supabase
    final response = await supabaseClient.auth.signInWithPassword(
      email: email,
      password: password,
    );

    // Step 2: Check if user exists
    if (response.user == null) {
      throw ServerException('User is null!');
    }

    // Step 3: Fetch user profile data
    final userProfile = await supabaseClient
        .from('profiles')
        .select()
        .eq('id', response.user!.id)
        .single();

    // Step 4: Convert to UserModel
    return UserModel.fromJson(userProfile);
  } catch (e) {
    throw ServerException(e.toString());
  }
}
```

---

### Use Case 3: Auto-Login (Current User)

**Scenario**: App opens, check if user is already logged in

#### Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 1: App starts                                                   │
│ Location: lib/main.dart:39-42                                        │
│                                                                      │
│ Code:                                                                │
│   void initState() {                                                 │
│     super.initState();                                               │
│     context.read<AuthBloc>().add(AuthIsUserLoggedIn());             │
│   }                                                                  │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 2: AuthBloc calls CurrentUser use case                         │
│ Location: features/auth/presentation/bloc/auth_bloc.dart:34-44      │
│                                                                      │
│ Code:                                                                │
│   void _isUserLoggedIn(AuthIsUserLoggedIn event, emit) async {      │
│     final res = await _currentUser(NoParams());                     │
│                                                                      │
│     res.fold(                                                        │
│       (failure) => emit(AuthFailure(failure.message)),              │
│       (user) => _emitAuthSuccess(user, emit),                       │
│     );                                                               │
│   }                                                                  │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 3: Repository checks for existing session                      │
│ Location: features/auth/data/repositories/auth_repository_impl.dart │
│                                                                      │
│ Logic:                                                               │
│   if (offline) {                                                     │
│     // Get user from local session                                  │
│     final session = remoteDataSource.currentUserSession;            │
│     return UserModel from session;                                  │
│   } else {                                                           │
│     // Fetch fresh user data from Supabase                          │
│     final user = await remoteDataSource.getCurrentUserData();       │
│     return user;                                                     │
│   }                                                                  │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 4: UI decides which page to show                               │
│ Location: lib/main.dart:50-60                                        │
│                                                                      │
│ Code:                                                                │
│   BlocSelector<AppUserCubit, AppUserState, bool>(                   │
│     selector: (state) => state is AppUserLoggedIn,                  │
│     builder: (context, isLoggedIn) {                                │
│       if (isLoggedIn) {                                             │
│         return const BlogPage();  // User is logged in              │
│       }                                                              │
│       return const LoginPage();   // User not logged in             │
│     },                                                               │
│   )                                                                  │
└──────────────────────────────────────────────────────────────────────┘
```

**Key Point**: Supabase stores session tokens locally. When app restarts, tokens are still there, so user remains logged in!

---

## Blog Feature

**Location**: `lib/features/blog/`

**Purpose**: Create, upload, and view blog posts with offline support

### Feature Structure

```
features/blog/
├── data/
│   ├── datasources/
│   │   ├── blog_remote_data_source.dart    # Supabase API
│   │   └── blog_local_data_source.dart     # Hive cache
│   ├── models/
│   │   └── blog_model.dart                 # Blog + JSON
│   └── repositories/
│       └── blog_repository_impl.dart       # Online/offline logic
├── domain/
│   ├── entities/
│   │   └── blog.dart                       # Blog entity
│   ├── repositories/
│   │   └── blog_repository.dart            # Repository interface
│   └── usecases/
│       ├── upload_blog.dart                # Create blog use case
│       └── get_all_blogs.dart              # Fetch blogs use case
└── presentation/
    ├── bloc/
    │   ├── blog_bloc.dart                  # State management
    │   ├── blog_event.dart                 # User actions
    │   └── blog_state.dart                 # UI states
    ├── pages/
    │   ├── blog_page.dart                  # Blog list screen
    │   ├── add_new_blog_page.dart          # Create blog screen
    │   └── blog_viewer_page.dart           # Read blog screen
    └── widgets/
        ├── blog_card.dart                  # Blog preview card
        └── blog_editor.dart                # Content editor
```

---

### Use Case 1: Upload Blog

**Scenario**: User wants to publish a new blog post with an image

#### Step-by-Step Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 1: User fills form and taps "Publish" button                   │
│ Location: features/blog/presentation/pages/add_new_blog_page.dart   │
│                                                                      │
│ User provides:                                                       │
│   - Title (text)                                                     │
│   - Content (text)                                                   │
│   - Image (picked from gallery/camera)                              │
│   - Topics (selected from list)                                     │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 2: AddNewBlogPage dispatches BlogUpload event                  │
│                                                                      │
│ Code:                                                                │
│   context.read<BlogBloc>().add(                                     │
│     BlogUpload(                                                      │
│       posterId: currentUser.id,                                      │
│       title: titleController.text.trim(),                           │
│       content: contentController.text.trim(),                       │
│       image: selectedImage!,  // File                               │
│       topics: selectedTopics,                                        │
│     ),                                                               │
│   );                                                                 │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 3: BlogBloc calls UploadBlog use case                          │
│ Location: features/blog/presentation/bloc/blog_bloc.dart:25-43      │
│                                                                      │
│ Code:                                                                │
│   void _onBlogUpload(BlogUpload event, Emitter<BlogState> emit) {   │
│     emit(BlogLoading());                                             │
│                                                                      │
│     final res = await _uploadBlog(                                  │
│       UploadBlogParams(                                              │
│         posterId: event.posterId,                                    │
│         title: event.title,                                          │
│         content: event.content,                                      │
│         image: event.image,                                          │
│         topics: event.topics,                                        │
│       ),                                                             │
│     );                                                               │
│                                                                      │
│     res.fold(                                                        │
│       (failure) => emit(BlogFailure(failure.message)),              │
│       (blog) => emit(BlogUploadSuccess()),                          │
│     );                                                               │
│   }                                                                  │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 4: UploadBlog use case calls BlogRepository                    │
│ Location: features/blog/domain/usecases/upload_blog.dart:8-21       │
│                                                                      │
│ Code:                                                                │
│   Future<Either<Failure, Blog>> call(UploadBlogParams params) {     │
│     return await blogRepository.uploadBlog(                         │
│       image: params.image,                                           │
│       title: params.title,                                           │
│       content: params.content,                                       │
│       posterId: params.posterId,                                     │
│       topics: params.topics,                                         │
│     );                                                               │
│   }                                                                  │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 5: BlogRepositoryImpl - Multi-step process                     │
│ Location: features/blog/data/repositories/blog_repository_impl.dart │
│                                                                      │
│ STEP 5.1: Check network connectivity                                │
│   if (!await connectionChecker.isConnected) {                       │
│     return left(Failure('No internet connection'));                 │
│   }                                                                  │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 5.2: Generate UUID and create temporary BlogModel              │
│ Location: blog_repository_impl.dart:36-44                           │
│                                                                      │
│ Code:                                                                │
│   BlogModel blogModel = BlogModel(                                  │
│     id: const Uuid().v1(),        // Generate unique ID             │
│     posterId: posterId,                                              │
│     title: title,                                                    │
│     content: content,                                                │
│     imageUrl: '',                 // Placeholder                    │
│     topics: topics,                                                  │
│     updatedAt: DateTime.now(),                                       │
│   );                                                                 │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 5.3: Upload image to Supabase Storage                          │
│ Location: blog_repository_impl.dart:46-49                           │
│                                                                      │
│ Code:                                                                │
│   final imageUrl = await blogRemoteDataSource.uploadBlogImage(      │
│     image: image,                                                    │
│     blog: blogModel,                                                 │
│   );                                                                 │
│                                                                      │
│ What happens:                                                        │
│   1. Read image file as bytes                                       │
│   2. Upload to Supabase storage bucket 'blog_images'                │
│   3. Get public URL of uploaded image                               │
│   4. Return URL: "https://[project].supabase.co/storage/v1/..."     │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 5.4: Update BlogModel with actual image URL                    │
│ Location: blog_repository_impl.dart:51-53                           │
│                                                                      │
│ Code:                                                                │
│   blogModel = blogModel.copyWith(                                   │
│     imageUrl: imageUrl,  // Now has real URL                        │
│   );                                                                 │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 5.5: Upload blog data to Supabase database                     │
│ Location: blog_repository_impl.dart:55-56                           │
│                                                                      │
│ Code:                                                                │
│   final uploadedBlog = await blogRemoteDataSource.uploadBlog(       │
│     blogModel                                                        │
│   );                                                                 │
│                                                                      │
│ SQL equivalent:                                                      │
│   INSERT INTO blogs (                                               │
│     id, poster_id, title, content, image_url, topics, updated_at    │
│   ) VALUES (                                                         │
│     '550e8400...', 'user-id', 'My Blog', 'Content...',              │
│     'https://...', ['Technology', 'Programming'], '2024-01-15'      │
│   )                                                                  │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 6: Return success, data flows back to UI                       │
│                                                                      │
│   Repository → UseCase → BLoC → UI                                  │
│                                                                      │
│ BLoC emits: BlogUploadSuccess()                                      │
│ UI shows: Success message, navigates back to blog list              │
└──────────────────────────────────────────────────────────────────────┘
```

#### Database Structure

**blogs table**:
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "poster_id": "user-uuid",
  "title": "Introduction to Clean Architecture",
  "content": "Clean Architecture is a software design philosophy...",
  "image_url": "https://project.supabase.co/storage/v1/object/public/blog_images/550e8400...",
  "topics": ["Technology", "Programming"],
  "updated_at": "2024-01-15T10:30:00Z"
}
```

**Storage bucket**: `blog_images/`
- Stores image files
- Each image named with blog ID
- Publicly accessible URLs

---

### Use Case 2: Fetch All Blogs (with Offline Support)

**Scenario**: User opens blog list page

#### Flow Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 1: BlogPage loads, dispatches BlogFetchAllBlogs event          │
│ Location: features/blog/presentation/pages/blog_page.dart           │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 2: BlogBloc calls GetAllBlogs use case                         │
│ Location: features/blog/presentation/bloc/blog_bloc.dart:45-55      │
│                                                                      │
│ Code:                                                                │
│   void _onFetchAllBlogs(BlogFetchAllBlogs event, emit) async {      │
│     emit(BlogLoading());                                             │
│                                                                      │
│     final res = await _getAllBlogs(NoParams());                     │
│                                                                      │
│     res.fold(                                                        │
│       (failure) => emit(BlogFailure(failure.message)),              │
│       (blogs) => emit(BlogsDisplaySuccess(blogs)),                  │
│     );                                                               │
│   }                                                                  │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│ STEP 3: Repository checks network connectivity                      │
│ Location: features/blog/data/repositories/blog_repository_impl.dart │
│                                                                      │
│                    ┌─ OFFLINE? ──┐                                  │
│                    ↓              ↓                                  │
│                  YES             NO                                  │
└──────────────────────────────────────────────────────────────────────┘
           ↓                                    ↓
┌─────────────────────────────┐    ┌──────────────────────────────────┐
│ OFFLINE PATH                │    │ ONLINE PATH                      │
│                             │    │                                  │
│ Load from Hive cache        │    │ Fetch from Supabase              │
└─────────────────────────────┘    └──────────────────────────────────┘
```

#### Offline Path (No Internet)

```dart
// lib/features/blog/data/repositories/blog_repository_impl.dart:63-75
@override
Future<Either<Failure, List<Blog>>> getAllBlogs() async {
  try {
    if (!await connectionChecker.isConnected) {
      // ═══════════ OFFLINE PATH ═══════════
      final blogs = blogLocalDataSource.loadBlogs();
      return right(blogs);
    }

    // ... online path ...
  } catch (e) {
    return left(Failure(e.toString()));
  }
}
```

**BlogLocalDataSource** (Hive):

```dart
// lib/features/blog/data/datasources/blog_local_data_source.dart:14-23
@override
List<BlogModel> loadBlogs() {
  List<BlogModel> blogs = [];

  box.read(() {
    // Read all blogs from Hive box
    for (int i = 0; i < box.length; i++) {
      blogs.add(BlogModel.fromJson(box.get(i.toString())));
    }
  });

  return blogs;
}
```

**What happens**:
1. Check Hive box for cached blogs
2. Convert JSON → BlogModel
3. Return cached list
4. **No network call!**

---

#### Online Path (Internet Available)

```dart
// lib/features/blog/data/repositories/blog_repository_impl.dart:63-75
@override
Future<Either<Failure, List<Blog>>> getAllBlogs() async {
  try {
    if (!await connectionChecker.isConnected) {
      // ... offline path ...
    }

    // ═══════════ ONLINE PATH ═══════════
    // Step 1: Fetch from Supabase
    final blogs = await blogRemoteDataSource.getAllBlogs();

    // Step 2: Cache locally for offline use
    blogLocalDataSource.uploadLocalBlogs(blogs: blogs);

    return right(blogs);
  } catch (e) {
    return left(Failure(e.toString()));
  }
}
```

**BlogRemoteDataSource** (Supabase):

```dart
// Simplified from blog_remote_data_source.dart
Future<List<BlogModel>> getAllBlogs() async {
  // SQL: SELECT blogs.*, profiles.name as poster_name
  //      FROM blogs
  //      LEFT JOIN profiles ON blogs.poster_id = profiles.id
  //      ORDER BY updated_at DESC

  final blogsData = await supabaseClient
      .from('blogs')
      .select('*, profiles(name)')
      .order('updated_at', ascending: false);

  return blogsData
      .map((blog) => BlogModel.fromJson(blog))
      .toList();
}
```

**BlogLocalDataSource** (Save to cache):

```dart
// lib/features/blog/data/datasources/blog_local_data_source.dart:25-34
@override
void uploadLocalBlogs({required List<BlogModel> blogs}) {
  box.clear(); // Clear old cache

  box.write(() {
    // Save each blog as JSON
    for (int i = 0; i < blogs.length; i++) {
      box.put(i.toString(), blogs[i].toJson());
    }
  });
}
```

**What happens**:
1. Fetch fresh blogs from Supabase
2. Clear old cache
3. Save new blogs to Hive (as JSON)
4. Return fresh list
5. **Next time offline, cached data available!**

---

## Offline Strategy

### How It Works

This app implements an **offline-first strategy**:

1. **Online**: Fetch from API, cache locally
2. **Offline**: Load from cache
3. **User always gets data** (if previously loaded)

### Network Connectivity Check

```dart
// lib/core/network/connection_checker.dart
abstract interface class ConnectionChecker {
  Future<bool> get isConnected;
}

class ConnectionCheckerImpl implements ConnectionChecker {
  final InternetConnection internetConnection;

  @override
  Future<bool> get isConnected async {
    return await internetConnection.hasInternetAccess;
  }
}
```

Uses `internet_connection_checker_plus` package to actually ping servers.

### Cache Implementation (Hive)

**Why Hive?**
- Fast, lightweight NoSQL database
- Stores data as JSON on device
- No SQL queries needed
- Works offline by default

**Setup**:

```dart
// lib/init_dependencies.main.dart:14-20
// Set Hive storage directory
Hive.defaultDirectory = (await getApplicationDocumentsDirectory()).path;

// Register Hive box
serviceLocator.registerLazySingleton(
  () => Hive.box(name: 'blogs'),
);
```

**Storage location**:
- iOS: `App Documents/blogs.hive`
- Android: `App Documents/blogs.hive`

---

## Global State Management

### AppUserCubit

**Purpose**: Manage globally accessible user state

**Location**: [lib/core/common/cubits/app_user/app_user_cubit.dart](../lib/core/common/cubits/app_user/app_user_cubit.dart:7-17)

```dart
class AppUserCubit extends Cubit<AppUserState> {
  AppUserCubit() : super(AppUserInitial());

  void updateUser(User? user) {
    if (user == null) {
      emit(AppUserInitial());  // User logged out
    } else {
      emit(AppUserLoggedIn(user));  // User logged in
    }
  }
}
```

**Why Separate from AuthBloc?**

- `AuthBloc`: Handles auth operations (login, signup)
- `AppUserCubit`: Holds current user state (used everywhere)

**Usage**:

```dart
// Update user (in AuthBloc)
_appUserCubit.updateUser(user);

// Check if logged in (in main.dart)
BlocSelector<AppUserCubit, AppUserState, bool>(
  selector: (state) => state is AppUserLoggedIn,
  builder: (context, isLoggedIn) {
    if (isLoggedIn) return BlogPage();
    return LoginPage();
  },
)

// Get current user (in AddNewBlogPage)
final currentUser = (context.read<AppUserCubit>().state as AppUserLoggedIn).user;
final posterId = currentUser.id;
```

---

## Summary

### Authentication Feature Summary

| Operation | Use Case | Repository Method | API Call |
|-----------|----------|-------------------|----------|
| Sign Up | `UserSignUp` | `signUpWithEmailPassword()` | `auth.signUp()` + INSERT profiles |
| Login | `UserLogin` | `loginWithEmailPassword()` | `auth.signInWithPassword()` |
| Auto-Login | `CurrentUser` | `currentUser()` | Check session / SELECT profiles |

**Key Points**:
- ✅ Session persisted by Supabase (auto-login)
- ✅ User profile stored in custom `profiles` table
- ✅ Global state managed by `AppUserCubit`

---

### Blog Feature Summary

| Operation | Use Case | Repository Method | Data Sources |
|-----------|----------|-------------------|--------------|
| Upload Blog | `UploadBlog` | `uploadBlog()` | Remote only (Supabase) |
| Fetch Blogs | `GetAllBlogs` | `getAllBlogs()` | Remote (online) + Local (offline) |

**Upload Process**:
1. Generate UUID
2. Upload image → Storage
3. Create BlogModel with image URL
4. Insert into `blogs` table

**Fetch Process**:
1. Check network connectivity
2. **If online**: Fetch from Supabase, cache in Hive
3. **If offline**: Load from Hive cache
4. Return blogs to UI

**Key Points**:
- ✅ Offline-first architecture
- ✅ Local caching with Hive
- ✅ Image storage in Supabase Storage
- ✅ Reading time calculation

---

## What's Next?

You now understand complete feature flows! Continue to:

**[06-BEGINNER-GUIDE.md](06-BEGINNER-GUIDE.md)** - Practical tips for navigating and extending the codebase

---

**Almost done with the documentation! 🎉**
