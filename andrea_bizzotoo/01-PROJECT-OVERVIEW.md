# Project Overview - Flutter E-Commerce Application

## Table of Contents
- [Introduction](#introduction)
- [What This Project Is](#what-this-project-is)
- [Technology Stack](#technology-stack)
- [Getting Started](#getting-started)
- [Project Structure](#project-structure)
- [Key Concepts Glossary](#key-concepts-glossary)
- [How the App Works (High-Level)](#how-the-app-works-high-level)
- [What You'll Learn](#what-youll-learn)

---

## Introduction

Welcome to the **Flutter E-Commerce Application** documentation! This project is a complete, production-ready example of a modern mobile shopping application built with Flutter. Whether you're a beginner just starting your Flutter journey or an experienced developer looking to understand advanced architectural patterns, this documentation will guide you through every aspect of the codebase.

### Why This Project Exists

This application was created to demonstrate:
- **Professional Flutter architecture** that scales
- **Best practices** in state management, testing, and code organization
- **Real-world patterns** used in production applications
- **Clean code principles** that make maintenance easier

---

## What This Project Is

This is a **fully functional e-commerce mobile application** that allows users to:

### Core Features
1. **Browse Products**
   - View a catalog of products with images, prices, and ratings
   - Search for specific products
   - See detailed product information

2. **Manage Shopping Cart**
   - Add products to cart with custom quantities
   - Update or remove items
   - Cart persists locally (saved even if you close the app)
   - Automatic sync when you sign in

3. **User Authentication**
   - Create an account with email and password
   - Sign in and sign out
   - Account information management

4. **Checkout & Orders**
   - Complete purchase flow
   - View order history
   - Track order status (confirmed, shipped, delivered)

5. **Product Reviews**
   - Rate products with stars (1-5)
   - Write text reviews
   - View average ratings from all users

### What Makes This Special

- **No Backend Required**: Uses fake repositories for demo purposes
- **Works Offline**: Local cart storage works without internet
- **Fully Tested**: Includes unit, widget, and integration tests
- **Responsive Design**: Adapts to different screen sizes
- **Type-Safe**: Leverages Dart's strong typing for reliability
- **Modern Architecture**: Following industry best practices

---

## Technology Stack

Here's what powers this application (don't worry if you don't know these yet - we'll explain everything!):

### Core Framework
- **Flutter 3.5+**: Google's UI toolkit for building mobile, web, and desktop apps
- **Dart**: The programming language Flutter uses

### State Management
- **Riverpod 2.6**: A powerful state management solution
  - Think of it as the "brain" that manages data flow in the app
  - Replaces older patterns like Provider or InheritedWidget
  - Provides dependency injection (more on this later!)

### Navigation
- **GoRouter 15.0**: Handles navigation between screens
  - Declarative routing (you define where routes go)
  - Deep linking support (open specific pages from URLs)
  - Type-safe navigation with route guards

### Local Storage
- **Sembast 3.8**: NoSQL database for Flutter
  - Stores shopping cart data locally
  - Works on iOS, Android, and web
  - Lightweight and easy to use

### Reactive Programming
- **RxDart 0.28**: Reactive extensions for Dart
  - Helps manage streams of data
  - Makes the UI automatically update when data changes

### UI Components
- **flutter_layout_grid**: Creates responsive grid layouts
- **flutter_rating_bar**: Star rating widgets
- **intl**: Internationalization and date/currency formatting

### Development Tools
- **build_runner**: Code generation tool
- **riverpod_generator**: Generates Riverpod provider code automatically
- **mocktail**: Testing framework for creating mock objects
- **flutter_lints**: Code quality and style checker

---

## Getting Started

### Prerequisites
Before running this project, ensure you have:
1. **Flutter SDK** installed (version 3.5 or higher)
2. **Dart SDK** (comes with Flutter)
3. An **IDE** (VS Code or Android Studio recommended)
4. **Device/Emulator** to run the app

### Installation Steps

1. **Clone or navigate to the project directory**
   ```bash
   cd /Users/maxwellfarias/Documents/complete-flutter-course/ecommerce_app
   ```

2. **Install dependencies**
   ```bash
   flutter pub get
   ```

3. **Generate code** (for Riverpod providers)
   ```bash
   dart run build_runner build --delete-conflicting-outputs
   ```

4. **Run the app**
   ```bash
   flutter run
   ```

### Testing

```bash
# Run all unit and widget tests
flutter test

# Run integration tests
flutter test integration_test/
```

---

## Project Structure

Understanding the folder structure is crucial. Here's how the project is organized:

```
ecommerce_app/
│
├── lib/                              # Main application code
│   ├── main.dart                     # App entry point (starts here!)
│   │
│   └── src/                          # Source code
│       ├── app.dart                  # Root widget configuration
│       │
│       ├── constants/                # App-wide constants
│       │   ├── app_sizes.dart        # Spacing, padding values
│       │   ├── breakpoints.dart      # Responsive breakpoints
│       │   └── test_products.dart    # Sample product data
│       │
│       ├── exceptions/               # Custom error types
│       │   ├── app_exception.dart    # Base exception class
│       │   └── error_logger.dart     # Error logging service
│       │
│       ├── utils/                    # Helper utilities
│       │   ├── async_value_ui.dart   # Error UI helpers
│       │   ├── currency_formatter.dart
│       │   ├── date_formatter.dart
│       │   └── in_memory_store.dart  # Reactive data store
│       │
│       ├── common_widgets/           # Reusable UI components
│       │   ├── async_value_widget.dart
│       │   ├── primary_button.dart
│       │   ├── responsive_center.dart
│       │   ├── empty_placeholder_widget.dart
│       │   └── ...more widgets
│       │
│       ├── routing/                  # Navigation setup
│       │   └── app_router.dart       # GoRouter configuration
│       │
│       ├── localization/             # Internationalization
│       │   └── string_hardcoded.dart # i18n helper
│       │
│       └── features/                 # Feature modules (⭐ IMPORTANT)
│           │
│           ├── authentication/       # User sign-in/sign-up
│           │   ├── data/            # Auth data sources
│           │   ├── domain/          # User model
│           │   └── presentation/    # Sign-in UI
│           │
│           ├── products/             # Product catalog
│           │   ├── data/            # Product repository
│           │   ├── domain/          # Product model
│           │   └── presentation/    # Product list/detail UI
│           │
│           ├── cart/                 # Shopping cart
│           │   ├── application/     # Cart business logic
│           │   ├── data/            # Cart storage
│           │   │   ├── local/       # Local cart (Sembast)
│           │   │   └── remote/      # Remote cart (fake)
│           │   ├── domain/          # Cart model
│           │   └── presentation/    # Cart UI
│           │
│           ├── checkout/             # Purchase flow
│           │   ├── application/     # Checkout service
│           │   └── presentation/    # Checkout screen
│           │
│           ├── orders/               # Order history
│           │   ├── application/     # Order service
│           │   ├── data/            # Order repository
│           │   ├── domain/          # Order model
│           │   └── presentation/    # Orders list UI
│           │
│           └── reviews/              # Product reviews
│               ├── application/     # Review service
│               ├── data/            # Review repository
│               ├── domain/          # Review model
│               └── presentation/    # Leave review UI
│
├── test/                             # Unit & widget tests
│   └── src/
│       ├── robot.dart                # Test helper pattern
│       └── features/                 # Feature-specific tests
│
├── integration_test/                 # Full flow tests
│   └── purchase_flow_test.dart
│
├── assets/                           # Images, fonts, etc.
│   └── products/                     # Product images
│
├── pubspec.yaml                      # Dependencies & configuration
│
└── docs/                             # Documentation (you are here!)
    ├── 01-PROJECT-OVERVIEW.md
    ├── 02-ARCHITECTURE-EXPLAINED.md
    └── ...more docs
```

### Key Organizational Principles

#### 1. **Feature-First Structure**
Instead of organizing by type (all models together, all widgets together), this project groups by **feature**:
- ✅ All authentication code is in `features/authentication/`
- ✅ All cart code is in `features/cart/`
- ✅ Easy to find related code
- ✅ Features can be independently developed/tested

#### 2. **Layered Architecture Within Features**
Each feature has up to 4 layers (more details in next doc):
- **domain/**: Business entities (models)
- **data/**: Data access and repositories
- **application/**: Business logic and services
- **presentation/**: UI screens and widgets

#### 3. **Shared vs Feature-Specific**
- **Shared code** → `src/common_widgets/`, `src/utils/`, `src/constants/`
- **Feature code** → `src/features/[feature_name]/`

---

## Key Concepts Glossary

Before diving deeper, let's define important terms you'll encounter:

### Architecture Terms

**Clean Architecture**
> A way of organizing code into layers with clear responsibilities. Inner layers (domain) don't know about outer layers (UI).

**Domain Layer**
> The core business entities and rules. Pure Dart classes with no Flutter dependencies.

**Data Layer**
> Responsible for fetching and storing data. Contains repositories and data sources.

**Application Layer**
> Business logic that coordinates between data and presentation layers.

**Presentation Layer**
> Everything the user sees and interacts with (UI screens, widgets, controllers).

**Repository**
> An abstraction that provides data to the app, hiding the source (API, database, etc.).

### State Management Terms

**State**
> Data that can change over time (like items in cart, logged-in user, etc.).

**Provider**
> In Riverpod, a container that holds and provides state to widgets.

**Consumer**
> A widget that listens to a provider and rebuilds when the state changes.

**Dependency Injection (DI)**
> A pattern where objects receive their dependencies from outside rather than creating them internally.

**Immutable**
> Once created, cannot be changed. To "modify" an immutable object, you create a new one.

### Flutter Terms

**Widget**
> Everything in Flutter is a widget - buttons, text, layouts, screens, etc.

**StatelessWidget**
> A widget that doesn't have mutable state (doesn't change over time).

**StatefulWidget**
> A widget with mutable state (can change and rebuild).

**BuildContext**
> Information about where a widget is in the widget tree. Needed for navigation, themes, etc.

**Async/Await**
> Dart keywords for handling asynchronous operations (like API calls).

**Stream**
> A sequence of asynchronous events. Like a pipe that data flows through over time.

**Future**
> Represents a value that will be available in the future (like a pending API response).

---

## How the App Works (High-Level)

Let's trace what happens when you perform a common action: **Adding a product to cart**.

### The Journey of "Add to Cart"

```
1. USER TAPS BUTTON
   ↓
2. UI Widget calls Controller
   (presentation/add_to_cart_controller.dart)
   ↓
3. Controller calls Cart Service
   (application/cart_service.dart)
   ↓
4. Service checks if user is logged in
   - Logged in? → Use Remote Cart Repository
   - Guest? → Use Local Cart Repository
   ↓
5. Service gets current cart from Repository
   (data/local/sembast_cart_repository.dart)
   ↓
6. Service updates cart (adds item)
   - Cart is immutable, so creates new cart with added item
   ↓
7. Service saves updated cart to Repository
   ↓
8. Repository saves to Sembast database
   ↓
9. Repository emits stream event (cart changed!)
   ↓
10. Riverpod notifies all listeners
   ↓
11. UI automatically rebuilds
    - Cart badge shows new count
    - Cart screen shows new item
```

### Key Observations

1. **Separation of Concerns**: Each layer has a specific job
2. **Unidirectional Data Flow**: Data flows down, events flow up
3. **Reactive Updates**: UI automatically updates via streams
4. **Testable**: Each layer can be tested independently
5. **Type Safety**: Compile-time errors prevent many bugs

---

## What You'll Learn

By studying this project and following these docs, you'll master:

### Beginner Level
- ✅ Flutter project structure and organization
- ✅ How to read and understand Dart code
- ✅ Basic widgets and UI composition
- ✅ Navigation between screens
- ✅ Working with lists and forms

### Intermediate Level
- ✅ State management with Riverpod
- ✅ Repository pattern for data access
- ✅ Async programming (Futures, Streams)
- ✅ Local storage with Sembast
- ✅ Code generation with build_runner
- ✅ Immutable data patterns

### Advanced Level
- ✅ Clean Architecture implementation
- ✅ Dependency injection patterns
- ✅ Testing strategies (unit, widget, integration)
- ✅ Error handling and logging
- ✅ Reactive programming with RxDart
- ✅ Advanced Riverpod patterns (AsyncNotifier, code generation)
- ✅ Performance optimization techniques

---

## Navigation Guide

Now that you understand the overview, here's the recommended reading order:

1. **📘 You are here**: 01-PROJECT-OVERVIEW.md
2. **📗 Next**: 02-ARCHITECTURE-EXPLAINED.md - Dive into the architecture layers
3. **📙 Then**: 03-RIVERPOD-COMPLETE-GUIDE.md - Master state management
4. **📕 Continue**: Follow the numbered docs in order

Each document builds on the previous one, so following the sequence will give you the best learning experience.

---

## Quick Reference

### Important Files to Study First

1. **[lib/main.dart](../lib/main.dart)** - App initialization
2. **[lib/src/app.dart](../lib/src/app.dart)** - Root widget setup
3. **[lib/src/features/products/domain/product.dart](../lib/src/features/products/domain/product.dart)** - Simple entity example
4. **[lib/src/features/cart/domain/cart.dart](../lib/src/features/cart/domain/cart.dart)** - Immutable entity with logic
5. **[lib/src/routing/app_router.dart](../lib/src/routing/app_router.dart)** - Navigation configuration

### Running Common Commands

```bash
# Install dependencies
flutter pub get

# Generate code
dart run build_runner build --delete-conflicting-outputs

# Watch mode (auto-regenerate on changes)
dart run build_runner watch --delete-conflicting-outputs

# Run app
flutter run

# Run tests
flutter test

# Clean build
flutter clean && flutter pub get

# Analyze code
flutter analyze
```

---

## Getting Help

As you go through this documentation:

1. **Don't rush** - Take time to understand each concept
2. **Run the code** - See it in action in your IDE
3. **Experiment** - Try modifying things and see what happens
4. **Read the comments** - The codebase has helpful inline comments
5. **Test as you learn** - Write small tests to verify your understanding

---

## What's Next?

Ready to understand the architecture? Head to **[02-ARCHITECTURE-EXPLAINED.md](02-ARCHITECTURE-EXPLAINED.md)** where we'll dive deep into:
- Clean Architecture principles
- The four layers (Domain, Data, Application, Presentation)
- How data flows through the app
- Why this architecture makes the code better

Let's continue the journey! 🚀
