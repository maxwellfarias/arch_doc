# 06 - Getting Started Guide

## Table of Contents

1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Project Setup](#project-setup)
4. [Understanding the Structure](#understanding-the-structure)
5. [Your First Feature: Forgot Password](#your-first-feature-forgot-password)
6. [Common Tasks](#common-tasks)
7. [Development Workflow](#development-workflow)
8. [Troubleshooting](#troubleshooting)
9. [Best Practices](#best-practices)
10. [Further Learning](#further-learning)

---

## Introduction

Welcome! This guide will help you get started with the Clean Flutter App, whether you're:
- A junior developer learning Clean Architecture
- An experienced developer new to this project
- Someone wanting to contribute to the codebase

**By the end of this guide**, you'll be able to add new features following the established architecture.

---

## Prerequisites

### Required Knowledge

**Flutter & Dart:**
- Basic Flutter widgets (StatelessWidget, StatefulWidget)
- Dart basics (classes, async/await, futures)
- State management concepts

**If you're new to Flutter:**
- Complete the [official Flutter tutorial](https://flutter.dev/docs/get-started/codelab)
- Understand [Dart language tour](https://dart.dev/guides/language/language-tour)

**Clean Architecture (Optional but helpful):**
- Not required to start, but read [Architecture Overview](01-architecture-overview.md) first

### Required Tools

```bash
# Flutter SDK
flutter --version
# Should be >= 2.12.0 (null safety)

# Dart SDK
dart --version
# Should be >= 2.12.0

# IDE (choose one)
- Android Studio with Flutter plugin
- VS Code with Flutter extension
- IntelliJ IDEA with Flutter plugin

# Git
git --version
```

---

## Project Setup

### Step 1: Clone the Repository

```bash
git clone https://github.com/your-repo/clean-flutter-app.git
cd clean-flutter-app
```

### Step 2: Install Dependencies

```bash
flutter pub get
```

**Expected output:**
```
Running "flutter pub get" in clean-flutter-app...
Resolving dependencies... (5.2s)
+ get 4.3.8
+ provider 6.0.1
+ equatable 2.0.3
...
Got dependencies!
```

### Step 3: Run the App

```bash
flutter run
```

**Or using IDE:**
- Open project in your IDE
- Select device (simulator/emulator/physical device)
- Press Run/Debug button

**First run may take a few minutes** to build.

### Step 4: Run Tests

```bash
# Run all tests
flutter test

# Run specific test file
flutter test test/presentation/presenters/getx_login_presenter_test.dart

# Run with coverage
flutter test --coverage
```

**Expected output:**
```
00:04 +32: All tests passed!
```

---

## Understanding the Structure

### Quick Project Tour

```
clean-flutter-app/
│
├── lib/                          ← All application code
│   ├── domain/                   ← Business logic (start here to understand app)
│   │   ├── entities/             ← Business objects
│   │   ├── usecases/             ← What the app can do
│   │   └── helpers/              ← Domain helpers
│   │
│   ├── data/                     ← Data operations
│   │   ├── usecases/             ← Use case implementations
│   │   ├── models/               ← Data models (JSON adapters)
│   │   ├── http/                 ← HTTP abstractions
│   │   └── cache/                ← Cache abstractions
│   │
│   ├── infra/                    ← External library adapters
│   │   ├── http/                 ← HTTP client wrapper
│   │   └── cache/                ← Storage wrappers
│   │
│   ├── presentation/             ← Screen logic
│   │   ├── presenters/           ← Presenters (GetX)
│   │   ├── mixins/               ← Reusable behaviors
│   │   └── protocols/            ← Presentation contracts
│   │
│   ├── ui/                       ← Visual layer
│   │   ├── pages/                ← Screens
│   │   ├── components/           ← Reusable widgets
│   │   ├── helpers/              ← UI utilities (i18n, errors)
│   │   └── mixins/               ← UI mixins
│   │
│   ├── validation/               ← Form validation
│   │   ├── validators/           ← Validators
│   │   └── protocols/            ← Validation contracts
│   │
│   ├── main/                     ← Dependency injection
│   │   ├── factories/            ← Object factories
│   │   ├── builders/             ← Builders
│   │   ├── composites/           ← Composites
│   │   ├── decorators/           ← Decorators
│   │   └── main.dart             ← App entry point
│   │
│   └── [feature folders follow same structure]
│
├── test/                         ← Mirror of lib/ with tests
│   └── [same structure as lib/]
│
├── requirements/                 ← Requirements documentation
│   ├── bdd_specs/                ← BDD feature files
│   └── use_cases/                ← Use case docs
│
└── docs/                         ← This documentation!

```

### Where to Find Things

**"Where is...?"**

| What | Where |
|------|-------|
| Business rules | `lib/domain/usecases/` |
| API calls | `lib/data/usecases/` (Remote* classes) |
| Screen logic | `lib/presentation/presenters/` |
| UI widgets | `lib/ui/pages/` and `lib/ui/components/` |
| Form validation | `lib/validation/validators/` |
| Dependency wiring | `lib/main/factories/` |
| Tests | `test/` (mirrors `lib/`) |

---

## Your First Feature: Forgot Password

Let's build a **Forgot Password** feature from scratch following Clean Architecture!

**Feature Requirements:**
- User enters email on forgot password screen
- App sends password reset link to email
- Show success message
- Navigate back to login

### Step 1: Define Domain Use Case

**File:** `lib/domain/usecases/forgot_password.dart`

```dart
import 'package:equatable/equatable.dart';

abstract class ForgotPassword {
  Future<void> sendResetLink(ForgotPasswordParams params);
}

class ForgotPasswordParams extends Equatable {
  final String email;

  List get props => [email];

  ForgotPasswordParams({required this.email});
}
```

**Why start here?**
- Domain defines WHAT the app does
- No implementation details yet
- Framework-independent

### Step 2: Create Data Implementation

**File:** `lib/data/usecases/forgot_password/remote_forgot_password.dart`

```dart
import '../../../domain/usecases/usecases.dart';
import '../../../domain/helpers/helpers.dart';
import '../../http/http.dart';

class RemoteForgotPassword implements ForgotPassword {
  final HttpClient httpClient;
  final String url;

  RemoteForgotPassword({
    required this.httpClient,
    required this.url,
  });

  Future<void> sendResetLink(ForgotPasswordParams params) async {
    final body = {'email': params.email};
    try {
      await httpClient.request(
        url: url,
        method: 'post',
        body: body,
      );
    } on HttpError catch (error) {
      throw error == HttpError.notFound
          ? DomainError.invalidCredentials
          : DomainError.unexpected;
    }
  }
}
```

**What we did:**
- Implemented the interface from Domain
- Depends on `HttpClient` abstraction (not concrete implementation)
- Maps HTTP errors to Domain errors
- Sends POST request with email

### Step 3: Create Presenter

**File:** `lib/presentation/presenters/getx_forgot_password_presenter.dart`

```dart
import '../../domain/helpers/helpers.dart';
import '../../domain/usecases/usecases.dart';
import '../../ui/helpers/helpers.dart';
import '../../ui/pages/pages.dart';
import '../mixins/mixins.dart';

import 'package:get/get.dart';

class GetxForgotPasswordPresenter extends GetxController
    with LoadingManager, NavigationManager, FormManager, UIErrorManager
    implements ForgotPasswordPresenter {
  final Validation validation;
  final ForgotPassword forgotPassword;

  final _emailError = Rx<UIError?>(null);
  final _successMessage = Rx<String?>(null);

  String? _email;

  Stream<UIError?> get emailErrorStream => _emailError.stream;
  Stream<String?> get successMessageStream => _successMessage.stream;

  GetxForgotPasswordPresenter({
    required this.validation,
    required this.forgotPassword,
  });

  void validateEmail(String email) {
    _email = email;
    _emailError.value = _validateField('email');
    _validateForm();
  }

  UIError? _validateField(String field) {
    final formData = {'email': _email};
    final error = validation.validate(field: field, input: formData);
    switch (error) {
      case ValidationError.invalidField:
        return UIError.invalidField;
      case ValidationError.requiredField:
        return UIError.requiredField;
      default:
        return null;
    }
  }

  void _validateForm() {
    isFormValid = _emailError.value == null && _email != null;
  }

  Future<void> sendResetLink() async {
    try {
      mainError = null;
      isLoading = true;
      await forgotPassword.sendResetLink(
          ForgotPasswordParams(email: _email!));
      _successMessage.value =
          'Link de recuperação enviado para ${_email}';
      await Future.delayed(Duration(seconds: 2));
      navigateTo = '/login';
    } on DomainError catch (error) {
      switch (error) {
        case DomainError.invalidCredentials:
          mainError = UIError.invalidField;
          break;
        default:
          mainError = UIError.unexpected;
          break;
      }
      isLoading = false;
    }
  }

  void backToLogin() {
    navigateTo = '/login';
  }
}
```

**Key points:**
- Extends `GetxController` for lifecycle management
- Uses mixins for common behaviors (loading, navigation, etc.)
- Exposes streams for UI to listen
- Validates email in real-time
- Handles success and error cases

### Step 4: Create Presenter Protocol

**File:** `lib/ui/pages/forgot_password/forgot_password_presenter.dart`

```dart
import '../../helpers/helpers.dart';

abstract class ForgotPasswordPresenter {
  Stream<UIError?> get emailErrorStream;
  Stream<bool> get isLoadingStream;
  Stream<bool> get isFormValidStream;
  Stream<UIError?> get mainErrorStream;
  Stream<String> get navigateToStream;
  Stream<String?> get successMessageStream;

  void validateEmail(String email);
  Future<void> sendResetLink();
  void backToLogin();
}
```

**Why separate interface?**
- UI depends on abstraction, not concrete presenter
- Easy to swap presenters
- Mockable for testing

### Step 5: Create UI Page

**File:** `lib/ui/pages/forgot_password/forgot_password_page.dart`

```dart
import '../../components/components.dart';
import '../../helpers/helpers.dart';
import '../../mixins/mixins.dart';
import './components/components.dart';
import './forgot_password_presenter.dart';

import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

class ForgotPasswordPage extends StatelessWidget
    with KeyboardManager, LoadingManager, UIErrorManager, NavigationManager {
  final ForgotPasswordPresenter presenter;

  ForgotPasswordPage(this.presenter);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(R.string.forgotPassword),
        leading: IconButton(
          icon: Icon(Icons.arrow_back),
          onPressed: presenter.backToLogin,
        ),
      ),
      body: Builder(
        builder: (context) {
          handleLoading(context, presenter.isLoadingStream);
          handleMainError(context, presenter.mainErrorStream);
          handleNavigation(presenter.navigateToStream, clear: true);
          handleSuccessMessage(context, presenter.successMessageStream);

          return GestureDetector(
            onTap: () => hideKeyboard(context),
            child: SingleChildScrollView(
              child: Padding(
                padding: EdgeInsets.all(32),
                child: ListenableProvider(
                  create: (_) => presenter,
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.stretch,
                    children: <Widget>[
                      Headline1(text: R.string.forgotPasswordTitle),
                      Padding(
                        padding: EdgeInsets.only(top: 16, bottom: 32),
                        child: Text(
                          R.string.forgotPasswordDescription,
                          textAlign: TextAlign.center,
                        ),
                      ),
                      EmailInput(),
                      SizedBox(height: 32),
                      SendResetLinkButton(),
                    ],
                  ),
                ),
              ),
            ),
          );
        },
      ),
    );
  }

  void handleSuccessMessage(
      BuildContext context, Stream<String?> stream) {
    stream.listen((message) {
      if (message != null) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            backgroundColor: Colors.green[700],
            content: Text(message),
          ),
        );
      }
    });
  }
}
```

### Step 6: Create UI Components

**File:** `lib/ui/pages/forgot_password/components/email_input.dart`

```dart
import '../../../helpers/helpers.dart';
import '../forgot_password_presenter.dart';

import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

class EmailInput extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final presenter = Provider.of<ForgotPasswordPresenter>(context);

    return StreamBuilder<UIError?>(
      stream: presenter.emailErrorStream,
      builder: (context, snapshot) {
        return TextFormField(
          decoration: InputDecoration(
            labelText: R.string.email,
            errorText: snapshot.hasData ? snapshot.data?.description : null,
            icon: Icon(Icons.email),
          ),
          keyboardType: TextInputType.emailAddress,
          onChanged: presenter.validateEmail,
        );
      },
    );
  }
}
```

**File:** `lib/ui/pages/forgot_password/components/send_reset_link_button.dart`

```dart
import '../../../helpers/helpers.dart';
import '../forgot_password_presenter.dart';

import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

class SendResetLinkButton extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final presenter = Provider.of<ForgotPasswordPresenter>(context);

    return StreamBuilder<bool>(
      stream: presenter.isFormValidStream,
      builder: (context, snapshot) {
        return ElevatedButton(
          onPressed: snapshot.data == true ? presenter.sendResetLink : null,
          child: Text(R.string.send.toUpperCase()),
        );
      },
    );
  }
}
```

**File:** `lib/ui/pages/forgot_password/components/components.dart` (barrel file)

```dart
export './email_input.dart';
export './send_reset_link_button.dart';
```

### Step 7: Add i18n Strings

**File:** `lib/ui/helpers/i18n/strings/pt_br.dart`

Add to the `PtBr` class:

```dart
String get forgotPassword => 'Esqueci a Senha';
String get forgotPasswordTitle => 'Recuperar Senha';
String get forgotPasswordDescription =>
    'Digite seu e-mail para receber um link de recuperação';
String get send => 'Enviar';
```

**File:** `lib/ui/helpers/i18n/strings/translations.dart`

Add to the `Translations` abstract class:

```dart
String get forgotPassword;
String get forgotPasswordTitle;
String get forgotPasswordDescription;
String get send;
```

### Step 8: Create Validation Factory

**File:** `lib/main/factories/pages/forgot_password/forgot_password_validation_factory.dart`

```dart
import '../../../../presentation/protocols/protocols.dart';
import '../../../../validation/protocols/protocols.dart';
import '../../../builders/builders.dart';
import '../../../composites/composites.dart';

Validation makeForgotPasswordValidation() {
  return ValidationComposite(makeForgotPasswordValidations());
}

List<FieldValidation> makeForgotPasswordValidations() {
  return [
    ...ValidationBuilder.field('email').required().email().build(),
  ];
}
```

### Step 9: Create Use Case Factory

**File:** `lib/main/factories/usecases/forgot_password_factory.dart`

```dart
import '../../../domain/usecases/usecases.dart';
import '../../../data/usecases/usecases.dart';
import '../http/http.dart';

ForgotPassword makeRemoteForgotPassword() {
  return RemoteForgotPassword(
    httpClient: makeHttpAdapter(),
    url: makeApiUrl('forgot-password'),
  );
}
```

### Step 10: Create Presenter Factory

**File:** `lib/main/factories/pages/forgot_password/forgot_password_presenter_factory.dart`

```dart
import '../../../../presentation/presenters/presenters.dart';
import '../../../../ui/pages/pages.dart';
import '../../factories.dart';

ForgotPasswordPresenter makeGetxForgotPasswordPresenter() =>
    GetxForgotPasswordPresenter(
      validation: makeForgotPasswordValidation(),
      forgotPassword: makeRemoteForgotPassword(),
    );
```

### Step 11: Create Page Factory

**File:** `lib/main/factories/pages/forgot_password/forgot_password_page_factory.dart`

```dart
import '../../../../ui/pages/pages.dart';
import '../../factories.dart';

import 'package:flutter/material.dart';

Widget makeForgotPasswordPage() {
  return ForgotPasswordPage(makeGetxForgotPasswordPresenter());
}
```

### Step 12: Add Route

**File:** `lib/main/main.dart`

Add to the `getPages` list:

```dart
GetPage(
  name: '/forgot_password',
  page: makeForgotPasswordPage,
),
```

### Step 13: Link from Login Page

**File:** `lib/ui/pages/login/login_page.dart`

Add a "Forgot Password?" button to the login form:

```dart
TextButton(
  onPressed: () => Get.toNamed('/forgot_password'),
  child: Text(R.string.forgotPassword),
)
```

### Step 14: Test It!

```bash
# Run the app
flutter run

# Navigate to login page
# Click "Esqueci a Senha"
# Enter email and send
```

---

## Common Tasks

### Task 1: Add a New Page

**Checklist:**

1. [ ] Create presenter protocol in `ui/pages/[feature]/`
2. [ ] Create presenter in `presentation/presenters/`
3. [ ] Create page in `ui/pages/[feature]/`
4. [ ] Create components in `ui/pages/[feature]/components/`
5. [ ] Create validation factory in `main/factories/pages/[feature]/`
6. [ ] Create presenter factory in `main/factories/pages/[feature]/`
7. [ ] Create page factory in `main/factories/pages/[feature]/`
8. [ ] Add route in `main/main.dart`
9. [ ] Add i18n strings

### Task 2: Add a New Use Case

**Checklist:**

1. [ ] Define interface in `domain/usecases/`
2. [ ] Create params class (if needed) extending `Equatable`
3. [ ] Implement in `data/usecases/`
4. [ ] Create model (if needed) in `data/models/`
5. [ ] Create factory in `main/factories/usecases/`
6. [ ] Write tests in `test/`

### Task 3: Add a New Validator

**Checklist:**

1. [ ] Create validator class in `validation/validators/`
2. [ ] Implement `FieldValidation` interface
3. [ ] Extend `Equatable`
4. [ ] Add to `ValidationBuilder` in `main/builders/validation_builder.dart`
5. [ ] Write tests

### Task 4: Add Error Handling

**Checklist:**

1. [ ] Add to `DomainError` enum (if business error)
2. [ ] Add to `UIError` enum
3. [ ] Add i18n string for error message
4. [ ] Map in presenter (DomainError → UIError)
5. [ ] Display in UI

---

## Development Workflow

### TDD Approach (Recommended)

This project follows Test-Driven Development:

**Red → Green → Refactor**

1. **Write Test First (Red)**
   ```dart
   test('Should call ForgotPassword with correct email', () async {
     // Arrange
     final forgotPassword = MockForgotPassword();
     final sut = GetxForgotPasswordPresenter(
       validation: MockValidation(),
       forgotPassword: forgotPassword,
     );

     // Act
     sut.validateEmail('test@test.com');
     await sut.sendResetLink();

     // Assert
     verify(() => forgotPassword.sendResetLink(
       ForgotPasswordParams(email: 'test@test.com')
     )).called(1);
   });
   ```

2. **Make Test Pass (Green)**
   - Implement minimum code to pass

3. **Refactor**
   - Improve code quality
   - Tests still pass

### Typical Development Flow

```
1. Understand requirement
   ↓
2. Define domain use case (interface)
   ↓
3. Write test for use case implementation
   ↓
4. Implement use case (make test pass)
   ↓
5. Write test for presenter
   ↓
6. Implement presenter (make test pass)
   ↓
7. Create UI (no test needed for simple widgets)
   ↓
8. Wire dependencies in Main layer
   ↓
9. Manual testing
   ↓
10. Refactor and optimize
```

---

## Troubleshooting

### Common Issues

**1. "Can't find package"**

```bash
# Solution:
flutter pub get
```

**2. "Version solving failed"**

```bash
# Solution:
flutter pub upgrade
# Or delete pubspec.lock and run:
flutter pub get
```

**3. "Build failed"**

```bash
# Solution:
flutter clean
flutter pub get
flutter run
```

**4. "Stream has already been listened to"**

**Problem:** Trying to listen to a stream multiple times.

**Solution:** Use `.asBroadcastStream()` or create separate streams.

**5. "RenderFlex overflowed"**

**Problem:** Widget doesn't fit in available space.

**Solution:** Wrap in `SingleChildScrollView` or `Expanded`.

### Debugging Tips

**Use Flutter DevTools:**
```bash
flutter run
# Then open DevTools link in browser
```

**Add breakpoints in VS Code:**
- Click left of line number
- Run in debug mode (F5)

**Print statements:**
```dart
print('DEBUG: email = $_email');
```

**Check logs:**
```bash
flutter logs
```

---

## Best Practices

### Code Style

**1. Use meaningful names:**
```dart
// ❌ Bad
final a = getData();

// ✅ Good
final surveys = await loadSurveys.load();
```

**2. Keep functions small:**
```dart
// ✅ Single responsibility
void validateEmail(String email) {
  _email = email;
  _emailError.value = _validateField('email');
  _validateForm();
}
```

**3. Use const when possible:**
```dart
// ✅ Performance optimization
const SizedBox(height: 16)
```

### Architecture Guidelines

**1. Respect layer boundaries:**
```dart
// ❌ Don't import concrete classes in presentation
import '../data/usecases/remote_authentication.dart';

// ✅ Import abstractions
import '../domain/usecases/authentication.dart';
```

**2. Use dependency injection:**
```dart
// ✅ Inject dependencies
class MyPresenter {
  final UseCase useCase;
  MyPresenter({required this.useCase});
}
```

**3. Map errors between layers:**
```dart
// ✅ Each layer has its own errors
try {
  await httpClient.request(...);
} on HttpError catch (error) {
  throw error == HttpError.unauthorized
    ? DomainError.invalidCredentials
    : DomainError.unexpected;
}
```

---

## Further Learning

### Recommended Reading

**Clean Architecture:**
- 📘 "Clean Architecture" by Robert C. Martin
- 📘 "Clean Code" by Robert C. Martin

**Flutter:**
- 📘 [Flutter Documentation](https://flutter.dev/docs)
- 📘 [Dart Language Tour](https://dart.dev/guides/language/language-tour)

**Design Patterns:**
- 📘 "Design Patterns" (Gang of Four)
- 📘 "Head First Design Patterns"

### Practice Projects

**Ideas to practice:**
1. Add "Edit Profile" feature
2. Add "Change Password" feature
3. Add "Delete Account" feature
4. Add offline mode indicator
5. Add dark theme support

---

## Key Takeaways

**Architecture Rules:**
1. Domain has no dependencies
2. Data implements Domain
3. Infra wraps external libraries
4. Presentation coordinates
5. UI displays
6. Main wires everything

**Development Flow:**
1. Start with Domain (what)
2. Implement in Data (how)
3. Coordinate in Presentation
4. Display in UI
5. Wire in Main
6. Test each layer

**Best Practices:**
1. Write tests first (TDD)
2. Keep functions small
3. Use meaningful names
4. Respect layer boundaries
5. Map errors properly

---

## Next Steps

**You're ready to:**
1. ✅ Add new features to the project
2. ✅ Fix bugs systematically
3. ✅ Write tests for your code
4. ✅ Follow Clean Architecture principles

**Continue learning:**
1. **[Testing Strategy](07-testing-strategy.md)** - Master testing
2. **[Design Patterns](03-design-patterns.md)** - Deep dive into patterns
3. **[Data Flow Guide](05-data-flow-guide.md)** - Understand complete flows

---

**Documentation Version:** 1.0
**Last Updated:** 2025

---

**Happy coding! You're now a Clean Architecture developer!** 🚀
