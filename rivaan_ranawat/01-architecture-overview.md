# Project Overview - Blog App Clean Architecture

## Table of Contents
1. [Introduction](#introduction)
2. [What is This App?](#what-is-this-app)
3. [Key Features](#key-features)
4. [Technology Stack](#technology-stack)
5. [Project Statistics](#project-statistics)
6. [Project Structure](#project-structure)
7. [Prerequisites](#prerequisites)
8. [Quick Start Guide](#quick-start-guide)
9. [Architecture at a Glance](#architecture-at-a-glance)

---

## Introduction

Welcome to the **Blog App Clean Architecture** project! This is a production-ready Flutter application that demonstrates how to build scalable, maintainable mobile applications using **Clean Architecture** principles and modern Flutter best practices.

Think of this project as a **blueprint** for building professional Flutter apps. Whether you're a beginner learning Flutter or an experienced developer looking for architecture patterns, this project serves as an excellent reference.

---

## What is This App?

This is a **full-featured blogging platform** that allows users to:
- Create an account and log in securely
- Write and publish blog posts with images
- Browse all published blogs
- Read blog posts with estimated reading time
- Work offline with local data caching

**The Real Purpose**: Beyond being a functional blog app, this project is designed to teach you how to structure Flutter applications following industry-standard patterns. Every file, every folder, and every line of code follows a specific architectural principle.

---

## Key Features

### 1. User Authentication
- **Sign Up**: Register with name, email, and password
- **Login**: Secure authentication with session persistence
- **Auto-Login**: Remember user sessions across app restarts
- **Profile Management**: User data stored securely in the cloud

### 2. Blog Management
- **Create Posts**: Write blogs with title, content, cover image, and topic tags
- **Browse Blogs**: View all published blogs in a scrollable feed
- **Read Posts**: Full-screen reading experience with formatted content
- **Image Upload**: Upload and store images in cloud storage
- **Topic Filtering**: Categorize blogs by topics (Technology, Business, Programming, Entertainment)

### 3. Offline Capability
- **Smart Caching**: Automatically saves fetched blogs locally
- **Offline Reading**: View previously loaded blogs without internet
- **Network Detection**: Intelligently switches between online and offline modes
- **Seamless Sync**: Fetches fresh data when connection is restored

### 4. User Experience
- **Reading Time**: Calculates estimated reading time (based on 225 words/minute)
- **Dark Theme**: Beautiful dark mode UI
- **Loading States**: Smooth loading indicators
- **Error Handling**: User-friendly error messages

---

## Technology Stack

This project uses cutting-edge Flutter technologies and packages:

### Core Framework
- **Flutter SDK**: `>=3.3.0 <4.0.0`
- **Dart Language**: Modern, type-safe programming
- **Multi-Platform**: Runs on iOS, Android, Web, Windows, macOS, Linux

### State Management
- **flutter_bloc** `^8.1.4` - BLoC (Business Logic Component) pattern
  - Separates business logic from UI
  - Predictable state management
  - Easy to test

### Backend & Database
- **supabase_flutter** `^2.3.4` - Backend as a Service
  - PostgreSQL database
  - Authentication
  - File storage
  - Real-time capabilities

- **hive** `^4.0.0-dev.2` - Local NoSQL Database
  - Fast, lightweight local storage
  - Offline data caching
  - Type-safe data persistence

### Functional Programming
- **fpdart** `^1.1.0` - Functional programming utilities
  - `Either` monad for error handling
  - Type-safe operations
  - No more try-catch blocks!

### Dependency Injection
- **get_it** `^7.6.7` - Service locator pattern
  - Manages object creation and lifetime
  - Loose coupling between components
  - Makes testing easier

### Utilities
- **uuid** `^4.3.3` - Generate unique IDs for blogs
- **intl** `^0.19.0` - Date formatting and internationalization
- **image_picker** `^1.0.7` - Pick images from gallery/camera
- **path_provider** `^2.1.0` - Access device storage paths
- **internet_connection_checker_plus** `^2.2.0` - Network connectivity detection
- **dotted_border** `^2.1.0` - UI component for image upload area

---

## Project Statistics

Here's a quick overview of the project size:

```
📊 Project Metrics
├── Total Dart Files: 48 files
├── Features: 2 (Authentication, Blog Management)
├── Layers per Feature: 3 (Domain, Data, Presentation)
├── Use Cases: 5
│   ├── UserSignUp
│   ├── UserLogin
│   ├── CurrentUser
│   ├── UploadBlog
│   └── GetAllBlogs
├── BLoCs: 3 (AuthBloc, BlogBloc, AppUserCubit)
├── Pages: 5 (Login, Signup, BlogList, AddBlog, ViewBlog)
├── Entities: 2 (User, Blog)
├── Design Patterns: 8+
└── Dependencies: 14 packages
```

---

## Project Structure

Here's the high-level folder structure:

```
blog-app-clean-architecture/
│
├── lib/                           # All Dart source code
│   │
│   ├── main.dart                 # App entry point (starts here!)
│   ├── init_dependencies.dart    # Dependency injection setup
│   │
│   ├── core/                     # Shared/common code
│   │   ├── common/              # Shared entities, widgets, cubits
│   │   ├── constants/           # App-wide constants
│   │   ├── error/               # Error handling (Failures, Exceptions)
│   │   ├── network/             # Network connectivity checker
│   │   ├── secrets/             # API keys and secrets
│   │   ├── theme/               # App theme and colors
│   │   ├── usecase/             # Base UseCase interface
│   │   └── utils/               # Helper functions
│   │
│   └── features/                 # Feature modules (modular architecture)
│       │
│       ├── auth/                 # Authentication feature
│       │   ├── data/            # Data layer (API calls, models)
│       │   │   ├── datasources/  # Remote data source (Supabase)
│       │   │   ├── models/       # Data models (UserModel)
│       │   │   └── repositories/ # Repository implementation
│       │   │
│       │   ├── domain/          # Domain layer (business logic)
│       │   │   ├── repository/   # Repository interface
│       │   │   └── usecases/     # Business operations
│       │   │
│       │   └── presentation/    # Presentation layer (UI)
│       │       ├── bloc/         # State management (AuthBloc)
│       │       ├── pages/        # UI screens
│       │       └── widgets/      # Reusable UI components
│       │
│       └── blog/                 # Blog feature
│           ├── data/            # Data layer
│           │   ├── datasources/  # Remote & Local data sources
│           │   ├── models/       # Data models (BlogModel)
│           │   └── repositories/ # Repository implementation
│           │
│           ├── domain/          # Domain layer
│           │   ├── entities/     # Core business objects (Blog)
│           │   ├── repositories/ # Repository interface
│           │   └── usecases/     # Business operations
│           │
│           └── presentation/    # Presentation layer
│               ├── bloc/         # State management (BlogBloc)
│               ├── pages/        # UI screens
│               └── widgets/      # Reusable UI components
│
├── android/                      # Android platform files
├── ios/                          # iOS platform files
├── web/                          # Web platform files
├── test/                         # Test files
├── pubspec.yaml                  # Project dependencies
└── README.md                     # Project readme
```

### Understanding the Structure

**Think of this structure like a company organization:**

- **`lib/main.dart`** = The CEO (starts everything)
- **`core/`** = Shared Services (HR, IT, Finance) - used by all departments
- **`features/`** = Departments (Sales, Marketing) - independent teams
- **`features/auth/`** = Authentication Department (handles login/signup)
- **`features/blog/`** = Blog Department (handles blog operations)

Each feature has **3 floors** (layers):
1. **Domain** (Top Floor): The brains - business rules and logic
2. **Data** (Middle Floor): The workers - fetching and storing data
3. **Presentation** (Ground Floor): The face - what users see and interact with

---

## Prerequisites

Before running this project, make sure you have:

### 1. Development Environment
- **Flutter SDK** `>=3.3.0` installed ([Install Flutter](https://flutter.dev/docs/get-started/install))
- **Dart SDK** (comes with Flutter)
- **IDE**: VS Code or Android Studio with Flutter plugins
- **Git** for version control

### 2. Supabase Account (Backend)
- Create a free account at [supabase.com](https://supabase.com)
- You'll need:
  - Supabase project URL
  - Supabase anonymous key

### 3. Device/Emulator
- iOS Simulator (macOS only)
- Android Emulator
- Physical device connected via USB
- Chrome browser (for web)

---

## Quick Start Guide

### Step 1: Clone the Repository
```bash
git clone <repository-url>
cd blog-app-clean-architecture
```

### Step 2: Install Dependencies
```bash
flutter pub get
```

This downloads all packages listed in [pubspec.yaml](../pubspec.yaml:30-48).

### Step 3: Set Up Supabase Backend

#### 3.1 Create Supabase Project
1. Go to [supabase.com](https://supabase.com) and create a new project
2. Wait for project initialization (~2 minutes)

#### 3.2 Create Database Tables

Run these SQL commands in Supabase SQL Editor:

**Profiles Table** (stores user information):
```sql
CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id),
  name TEXT NOT NULL,
  email TEXT NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Enable Row Level Security
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

-- Allow users to read all profiles
CREATE POLICY "Profiles are viewable by everyone"
  ON profiles FOR SELECT
  USING (true);

-- Allow users to insert their own profile
CREATE POLICY "Users can insert their own profile"
  ON profiles FOR INSERT
  WITH CHECK (auth.uid() = id);
```

**Blogs Table** (stores blog posts):
```sql
CREATE TABLE blogs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  poster_id UUID NOT NULL REFERENCES profiles(id),
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  image_url TEXT NOT NULL,
  topics TEXT[] NOT NULL,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Enable Row Level Security
ALTER TABLE blogs ENABLE ROW LEVEL SECURITY;

-- Allow everyone to read blogs
CREATE POLICY "Blogs are viewable by everyone"
  ON blogs FOR SELECT
  USING (true);

-- Allow authenticated users to insert blogs
CREATE POLICY "Authenticated users can create blogs"
  ON blogs FOR INSERT
  WITH CHECK (auth.uid() = poster_id);
```

#### 3.3 Create Storage Bucket

1. Go to **Storage** in Supabase dashboard
2. Create a new bucket named `blog_images`
3. Set it to **Public**
4. Add this storage policy:

```sql
-- Allow authenticated users to upload images
CREATE POLICY "Authenticated users can upload images"
  ON storage.objects FOR INSERT
  WITH CHECK (bucket_id = 'blog_images' AND auth.role() = 'authenticated');

-- Allow everyone to view images
CREATE POLICY "Images are publicly accessible"
  ON storage.objects FOR SELECT
  USING (bucket_id = 'blog_images');
```

#### 3.4 Configure App Secrets

1. Get your Supabase URL and Anon Key from **Settings > API**
2. Open [lib/core/secrets/app_secrets.dart](../lib/core/secrets/app_secrets.dart)
3. Replace the placeholder values:

```dart
class AppSecrets {
  static const supabaseUrl = 'YOUR_SUPABASE_URL_HERE';
  static const supabaseAnonKey = 'YOUR_SUPABASE_ANON_KEY_HERE';
}
```

### Step 4: Run the App

```bash
# Run on connected device/emulator
flutter run

# Or specify platform
flutter run -d chrome      # Web
flutter run -d macos       # macOS
flutter run -d windows     # Windows
```

### Step 5: Test the App

1. **Sign Up**: Create a new account (name, email, password)
2. **Login**: Use your credentials to log in
3. **Create Blog**: Click the '+' button, add title, content, image, and topics
4. **View Blogs**: See all blogs in the feed
5. **Read Blog**: Tap any blog card to read the full content
6. **Test Offline**: Turn off internet, reopen app, see cached blogs

---

## Architecture at a Glance

This project implements **Clean Architecture** - a way of organizing code that makes it:
- ✅ **Testable**: Each part can be tested independently
- ✅ **Maintainable**: Easy to find and fix bugs
- ✅ **Scalable**: Easy to add new features
- ✅ **Independent**: UI, database, and business logic don't depend on each other

### The Three Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                        │
│  (What users see - UI, widgets, pages, BLoC)                │
│  Location: features/{feature}/presentation/                  │
├─────────────────────────────────────────────────────────────┤
│                       DATA LAYER                             │
│  (How we get/store data - API calls, database, models)      │
│  Location: features/{feature}/data/                          │
├─────────────────────────────────────────────────────────────┤
│                      DOMAIN LAYER                            │
│  (Business rules - entities, use cases, interfaces)         │
│  Location: features/{feature}/domain/                        │
│  This layer is PURE - no Flutter, no database code!         │
└─────────────────────────────────────────────────────────────┘
```

**Dependency Rule**: Arrows point inward (outer layers depend on inner layers, never vice versa)

```
Presentation → Data → Domain
     ↓          ↓        ↓
    UI      Database  Rules
```

**Example: User Login Flow**
1. User taps "Login" button **(Presentation)**
2. LoginPage calls AuthBloc **(Presentation)**
3. AuthBloc calls UserLogin use case **(Domain)**
4. Use case calls AuthRepository interface **(Domain)**
5. Repository implementation calls API **(Data)**
6. Data flows back up the chain
7. UI updates with success/error **(Presentation)**

---

## What's Next?

Now that you understand what this app does and how it's structured, continue to the next documentation files:

1. ✅ **01-PROJECT-OVERVIEW.md** ← You are here
2. ➡️ **[02-CLEAN-ARCHITECTURE-EXPLAINED.md](02-CLEAN-ARCHITECTURE-EXPLAINED.md)** - Deep dive into Clean Architecture
3. **[03-LAYER-BREAKDOWN.md](03-LAYER-BREAKDOWN.md)** - Detailed explanation of each layer
4. **[04-DESIGN-PATTERNS-GUIDE.md](04-DESIGN-PATTERNS-GUIDE.md)** - All design patterns used
5. **[05-FEATURE-DEEP-DIVE.md](05-FEATURE-DEEP-DIVE.md)** - Complete feature walkthroughs
6. **[06-BEGINNER-GUIDE.md](06-BEGINNER-GUIDE.md)** - Practical tips for navigating the codebase

---

## Quick Reference

### Important Files to Know
- [main.dart](../lib/main.dart) - App entry point (line 11-28)
- [init_dependencies.dart](../lib/init_dependencies.dart) - Dependency injection setup
- [usecase.dart](../lib/core/usecase/usecase.dart:4-6) - Base use case interface
- [user.dart](../lib/core/common/entities/user.dart:1-11) - User entity
- [blog.dart](../lib/features/blog/domain/entities/blog.dart:1-21) - Blog entity

### Common Commands
```bash
flutter pub get              # Install dependencies
flutter run                  # Run app
flutter test                 # Run tests
flutter clean                # Clean build files
flutter pub upgrade          # Upgrade dependencies
```

### Getting Help
- Read the other documentation files in this folder
- Check Flutter documentation: https://flutter.dev/docs
- Supabase documentation: https://supabase.com/docs
- BLoC documentation: https://bloclibrary.dev

---

**Happy coding! 🚀**
