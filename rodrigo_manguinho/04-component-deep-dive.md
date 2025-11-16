# 04 - Component Deep-Dive

## Table of Contents

1. [Introduction](#introduction)
2. [What You'll Learn](#what-youll-learn)
3. [Authentication System](#authentication-system)
4. [Survey Management System](#survey-management-system)
5. [State Management System](#state-management-system)
6. [Navigation System](#navigation-system)
7. [Error Handling System](#error-handling-system)
8. [Validation System](#validation-system)
9. [Caching & Offline Support](#caching--offline-support)
10. [Session Management](#session-management)
11. [Key Takeaways](#key-takeaways)
12. [Next Steps](#next-steps)

---

## Introduction

This document provides **end-to-end walkthroughs** of the major systems in the Clean Flutter App. We'll trace complete user interactions from button tap to screen update, showing how all the layers and patterns work together.

**Why deep dives matter:**
Understanding individual components is good, but seeing how they interact in real scenarios is essential for truly grasping the architecture.

---

## What You'll Learn

By the end of this document, you will understand:

- ✅ How user login works from start to finish
- ✅ How surveys are loaded with offline fallback
- ✅ How voting on surveys works
- ✅ How state flows through the application
- ✅ How navigation is triggered
- ✅ How errors are handled at each layer
- ✅ How validation works in real-time
- ✅ How session expiration is managed

---

## Authentication System

### Overview

The authentication system handles:
- User login with email/password
- Token storage (secure)
- Account persistence
- Navigation to surveys on success
- Error handling (invalid credentials, network errors)

### Component Diagram

```
┌──────────────────────────────────────────────────────────┐
│                    AUTHENTICATION FLOW                    │
└──────────────────────────────────────────────────────────┘

┌─────────────┐
│  LoginPage  │  UI Layer - Displays form and handles input
└──────┬──────┘
       │ user taps "Login"
       ↓
┌────────────────────────┐
│  GetxLoginPresenter    │  Presentation Layer - Coordinates logic
└──────┬─────────────────┘
       │ calls authentication.auth()
       ↓
┌────────────────────────┐
│  RemoteAuthentication  │  Data Layer - Implements use case
└──────┬─────────────────┘
       │ calls httpClient.request()
       ↓
┌──────────────────────────────────┐
│  AuthorizeHttpClientDecorator    │  Main Layer - Adds token (for future requests)
└──────┬───────────────────────────┘
       │ delegates to decoratee
       ↓
┌────────────────────────┐
│  HttpAdapter           │  Infra Layer - Wraps http package
└──────┬─────────────────┘
       │ makes HTTP POST
       ↓
┌────────────────────────┐
│  API Server            │  External - Backend API
└──────┬─────────────────┘
       │ returns JSON
       ↓
     (flows back up through layers)
       ↓
┌────────────────────────────┐
│  LocalSaveCurrentAccount   │  Data Layer - Saves token
└──────┬─────────────────────┘
       │ calls secureStorage.save()
       ↓
┌──────────────────────────┐
│  SecureStorageAdapter    │  Infra Layer - Wraps flutter_secure_storage
└──────┬───────────────────┘
       │ saves to device
       ↓
┌────────────────────────┐
│  Device Secure Storage │  External - OS keychain
└────────────────────────┘
```

---

### Step-by-Step: User Login

**Step 1: User Fills Form**

File: [lib/ui/pages/login/components/email_input.dart](../lib/ui/pages/login/components/email_input.dart)

```dart
class EmailInput extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final presenter = Provider.of<LoginPresenter>(context);

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
          onChanged: presenter.validateEmail, // ← Triggers validation
        );
      },
    );
  }
}
```

**What happens when user types:**
```
User types "t" → presenter.validateEmail("t")
User types "te" → presenter.validateEmail("te")
User types "test@email.com" → presenter.validateEmail("test@email.com")
```

---

**Step 2: Real-time Validation**

File: [lib/presentation/presenters/getx_login_presenter.dart](../lib/presentation/presenters/getx_login_presenter.dart:30-40)

```dart
void validateEmail(String email) {
  _email = email;
  _emailError.value = _validateField('email');
  _validateForm();
}

UIError? _validateField(String field) {
  final formData = {
    'email': _email,
    'password': _password,
  };
  final error = validation.validate(field: field, input: formData);
  switch (error) {
    case ValidationError.invalidField: return UIError.invalidField;
    case ValidationError.requiredField: return UIError.requiredField;
    default: return null;
  }
}

void _validateForm() {
  isFormValid = _emailError.value == null
    && _passwordError.value == null
    && _email != null
    && _password != null;
}
```

**Flow:**
```
1. Store email value
2. Validate the field
   └─ ValidationComposite runs:
      - RequiredFieldValidation('email') → null (has value)
      - EmailValidation('email') → ValidationError.invalidField (not email yet)
3. Update _emailError stream → UI shows error
4. Validate full form → isFormValid = false → Login button disabled
```

---

**Step 3: Form Becomes Valid**

```
User types: "test@email.com"
            "password123"

Final validation:
  - Email: null (valid)
  - Password: null (valid)
  - Both have values: true

isFormValid = true → Login button enabled!
```

File: [lib/ui/pages/login/components/login_button.dart](../lib/ui/pages/login/components/login_button.dart)

```dart
class LoginButton extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final presenter = Provider.of<LoginPresenter>(context);

    return StreamBuilder<bool>(
      stream: presenter.isFormValidStream,
      builder: (context, snapshot) {
        return ElevatedButton(
          onPressed: snapshot.data == true ? presenter.auth : null,
          child: Text(R.string.enter.toUpperCase()),
        );
      },
    );
  }
}
```

**Button state:**
- `isFormValid = false` → `onPressed: null` (disabled, grey)
- `isFormValid = true` → `onPressed: presenter.auth` (enabled, clickable)

---

**Step 4: User Taps Login**

File: [lib/presentation/presenters/getx_login_presenter.dart](../lib/presentation/presenters/getx_login_presenter.dart:62-76)

```dart
Future<void> auth() async {
  try {
    mainError = null;              // Clear previous error
    isLoading = true;              // Show loading spinner

    final account = await authentication.auth(
      AuthenticationParams(email: _email!, secret: _password!)
    );

    await saveCurrentAccount.save(account);  // Save token
    navigateTo = '/surveys';                 // Navigate to surveys

  } on DomainError catch (error) {
    switch (error) {
      case DomainError.invalidCredentials:
        mainError = UIError.invalidCredentials;
        break;
      default:
        mainError = UIError.unexpected;
        break;
    }
    isLoading = false;  // Hide loading spinner
  }
}
```

**Mixin provides `isLoading`:**

File: [lib/presentation/mixins/loading_manager.dart](../lib/presentation/mixins/loading_manager.dart)

```dart
mixin LoadingManager on GetxController {
  final _isLoading = false.obs;
  Stream<bool> get isLoadingStream => _isLoading.stream;

  set isLoading(bool value) => _isLoading.value = value;
}
```

**UI reacts to loading:**

File: [lib/ui/mixins/loading_manager.dart](../lib/ui/mixins/loading_manager.dart)

```dart
mixin LoadingManager {
  void handleLoading(BuildContext context, Stream<bool> stream) {
    stream.listen((isLoading) {
      if (isLoading) {
        showDialog(
          context: context,
          barrierDismissible: false,
          builder: (context) => SpinnerDialog(),
        );
      } else {
        if (Navigator.canPop(context)) {
          Navigator.of(context).pop();
        }
      }
    });
  }
}
```

**User sees:**
```
isLoading = true → Spinner dialog appears (blocks UI)
```

---

**Step 5: HTTP Request**

File: [lib/data/usecases/authentication/remote_authentication.dart](../lib/data/usecases/authentication/remote_authentication.dart:16-26)

```dart
Future<AccountEntity> auth(AuthenticationParams params) async {
  final body = RemoteAuthenticationParams.fromDomain(params).toJson();
  try {
    final httpResponse = await httpClient.request(
      url: url,
      method: 'post',
      body: body
    );
    return RemoteAccountModel.fromJson(httpResponse).toEntity();
  } on HttpError catch(error) {
    throw error == HttpError.unauthorized
      ? DomainError.invalidCredentials
      : DomainError.unexpected;
  }
}
```

**Data transformation:**
```
AuthenticationParams(email: 'test@email.com', secret: 'password123')
  ↓
RemoteAuthenticationParams(email: 'test@email.com', password: 'password123')
  ↓
JSON: {"email": "test@email.com", "password": "password123"}
  ↓
HTTP POST to /login
```

---

**Step 6: HTTP Adapter Makes Request**

File: [lib/infra/http/http_adapter.dart](../lib/infra/http/http_adapter.dart:11-34)

```dart
Future<dynamic> request({
  required String url,
  required String method,
  Map? body,
  Map? headers
}) async {
  final defaultHeaders = headers?.cast<String, String>() ?? {}..addAll({
    'content-type': 'application/json',
    'accept': 'application/json'
  });
  final jsonBody = body != null ? jsonEncode(body) : null;
  var response = Response('', 500);
  Future<Response>? futureResponse;

  try {
    if (method == 'post') {
      futureResponse = client.post(
        Uri.parse(url),
        headers: defaultHeaders,
        body: jsonBody
      );
    }
    // ... other methods
    if (futureResponse != null) {
      response = await futureResponse.timeout(Duration(seconds: 10));
    }
  } catch(error) {
    throw HttpError.serverError;
  }
  return _handleResponse(response);
}
```

**Request sent:**
```
POST https://fordevs.herokuapp.com/api/login
Headers: {"content-type": "application/json", "accept": "application/json"}
Body: "{\"email\":\"test@email.com\",\"password\":\"password123\"}"
Timeout: 10 seconds
```

---

**Step 7: Response Handling**

File: [lib/infra/http/http_adapter.dart](../lib/infra/http/http_adapter.dart:36-46)

```dart
dynamic _handleResponse(Response response) {
  switch (response.statusCode) {
    case 200: return response.body.isEmpty ? null : jsonDecode(response.body);
    case 204: return null;
    case 400: throw HttpError.badRequest;
    case 401: throw HttpError.unauthorized;
    case 403: throw HttpError.forbidden;
    case 404: throw HttpError.notFound;
    default: throw HttpError.serverError;
  }
}
```

**Success case (200):**
```
Response: {"accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
  ↓
Parse JSON: Map<String, dynamic>
  ↓
Return to RemoteAuthentication
```

**Error case (401):**
```
Response: HTTP 401 Unauthorized
  ↓
throw HttpError.unauthorized
  ↓
Caught in RemoteAuthentication
  ↓
throw DomainError.invalidCredentials
```

---

**Step 8: Model to Entity Transformation**

File: [lib/data/models/remote_account_model.dart](../lib/data/models/remote_account_model.dart:9-16)

```dart
factory RemoteAccountModel.fromJson(Map json) {
  if (!json.containsKey('accessToken')) {
    throw HttpError.invalidData;
  }
  return RemoteAccountModel(accessToken: json['accessToken']);
}

AccountEntity toEntity() => AccountEntity(token: accessToken);
```

**Transformation:**
```
JSON Response:
  {"accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
    ↓
RemoteAccountModel:
  RemoteAccountModel(accessToken: "eyJhbGci...")
    ↓
AccountEntity (Domain):
  AccountEntity(token: "eyJhbGci...")
    ↓
Return to presenter
```

---

**Step 9: Save Account**

File: [lib/data/usecases/account/local_save_current_account.dart](../lib/data/usecases/account/local_save_current_account.dart)

```dart
class LocalSaveCurrentAccount implements SaveCurrentAccount {
  final SaveSecureCacheStorage saveSecureCacheStorage;

  LocalSaveCurrentAccount({ required this.saveSecureCacheStorage });

  Future<void> save(AccountEntity account) async {
    try {
      await saveSecureCacheStorage.save(key: 'token', value: account.token);
    } catch (error) {
      throw DomainError.unexpected;
    }
  }
}
```

**Flow:**
```
AccountEntity(token: "eyJhbGci...")
  ↓
saveSecureCacheStorage.save(key: 'token', value: "eyJhbGci...")
  ↓
SecureStorageAdapter (Infra layer)
  ↓
FlutterSecureStorage (flutter_secure_storage package)
  ↓
OS Keychain (iOS) / Keystore (Android)
```

**Token is now securely stored!**

---

**Step 10: Navigation**

File: [lib/presentation/presenters/getx_login_presenter.dart](../lib/presentation/presenters/getx_login_presenter.dart:68)

```dart
navigateTo = '/surveys';
```

**Mixin triggers navigation:**

File: [lib/presentation/mixins/navigation_manager.dart](../lib/presentation/mixins/navigation_manager.dart)

```dart
mixin NavigationManager on GetxController {
  final _navigateTo = RxString('');
  Stream<String> get navigateToStream =>
    _navigateTo.stream.where((route) => route.isNotEmpty);

  set navigateTo(String value) => _navigateTo.value = value;
}
```

**UI listens and navigates:**

File: [lib/ui/mixins/navigation_manager.dart](../lib/ui/mixins/navigation_manager.dart)

```dart
mixin NavigationManager {
  void handleNavigation(Stream<String> stream, {bool clear = false}) {
    stream.listen((route) {
      if (route.isNotEmpty) {
        Get.offAllNamed(route); // If clear = true
        // or Get.toNamed(route)
      }
    });
  }
}
```

**Result:**
```
navigateTo = '/surveys'
  ↓
Stream emits '/surveys'
  ↓
Get.offAllNamed('/surveys')
  ↓
Clears navigation stack and navigates to SurveysPage
```

---

### Success Flow Summary

```
User taps Login
  ↓
presenter.auth()
  ↓
isLoading = true (spinner appears)
  ↓
authentication.auth(params)
  ↓
httpClient.request(url, method, body)
  ↓
HTTP POST to API
  ↓
API returns {"accessToken": "..."}
  ↓
Parse to RemoteAccountModel
  ↓
Convert to AccountEntity
  ↓
Return to presenter
  ↓
saveCurrentAccount.save(account)
  ↓
Save token to secure storage
  ↓
navigateTo = '/surveys'
  ↓
Navigate to SurveysPage (clear stack)
  ↓
isLoading = false (spinner hidden automatically on navigation)
```

---

### Error Flow: Invalid Credentials

```
User taps Login with wrong password
  ↓
presenter.auth()
  ↓
isLoading = true (spinner appears)
  ↓
authentication.auth(params)
  ↓
HTTP POST to API
  ↓
API returns HTTP 401 Unauthorized
  ↓
HttpAdapter throws HttpError.unauthorized
  ↓
RemoteAuthentication catches and throws DomainError.invalidCredentials
  ↓
Presenter catches DomainError
  ↓
mainError = UIError.invalidCredentials
  ↓
isLoading = false (spinner hidden)
  ↓
UI shows snackbar: "Credenciais inválidas"
```

---

## Survey Management System

### Overview

The survey system handles:
- Loading surveys from API
- Caching surveys locally
- Offline fallback
- Displaying surveys in a list
- Navigating to survey results
- Reloading when returning from survey result

### Component Diagram

```
┌──────────────────────────────────────────────────────────┐
│              SURVEY LOADING WITH FALLBACK                 │
└──────────────────────────────────────────────────────────┘

┌─────────────┐
│SurveysPage  │  UI Layer
└──────┬──────┘
       │ calls loadData()
       ↓
┌──────────────────────────┐
│ GetxSurveysPresenter     │  Presentation Layer
└──────┬───────────────────┘
       │ calls loadSurveys.load()
       ↓
┌────────────────────────────────────────┐
│ RemoteLoadSurveysWithLocalFallback     │  Main Layer (Composite)
└──────┬─────────────────────────────────┘
       │
       ├─ TRY: remote.load()
       │    ↓
       │  ┌─────────────────────────┐
       │  │ RemoteLoadSurveys       │  Data Layer
       │  └──────┬──────────────────┘
       │         │ calls httpClient.request()
       │         ↓
       │  ┌─────────────────────────────────┐
       │  │ AuthorizeHttpClientDecorator    │  Main Layer (adds token)
       │  └──────┬──────────────────────────┘
       │         │
       │         ↓
       │  ┌─────────────────────────┐
       │  │ HttpAdapter             │  Infra Layer
       │  └──────┬──────────────────┘
       │         │
       │         ↓
       │  ┌─────────────────────────┐
       │  │ API Server              │  External
       │  └──────┬──────────────────┘
       │         │
       │         ↓ SUCCESS
       │  Save to local cache
       │    ↓
       │  ┌─────────────────────────┐
       │  │ LocalLoadSurveys.save() │  Data Layer
       │  └─────────────────────────┘
       │
       └─ CATCH: local.load()
            ↓
          ┌─────────────────────────┐
          │ LocalLoadSurveys.load() │  Data Layer
          └──────┬──────────────────┘
                 │
                 ↓
          ┌─────────────────────────┐
          │ LocalStorageAdapter     │  Infra Layer
          └──────┬──────────────────┘
                 │
                 ↓
          ┌─────────────────────────┐
          │ Device Local Storage    │  External
          └─────────────────────────┘
```

---

### Step-by-Step: Load Surveys

**Step 1: Page Loads**

File: [lib/ui/pages/surveys/surveys_page.dart](../lib/ui/pages/surveys/surveys_page.dart:31)

```dart
@override
Widget build(BuildContext context) {
  return Scaffold(
    appBar: AppBar(title: Text(R.string.surveys)),
    body: Builder(
      builder: (context) {
        handleLoading(context, widget.presenter.isLoadingStream);
        handleSessionExpired(widget.presenter.isSessionExpiredStream);
        handleNavigation(widget.presenter.navigateToStream);
        widget.presenter.loadData(); // ← Calls presenter

        return StreamBuilder<List<SurveyViewModel>>(
          stream: widget.presenter.surveysStream,
          builder: (context, snapshot) {
            if (snapshot.hasError) {
              return ReloadScreen(
                error: '${snapshot.error}',
                reload: widget.presenter.loadData
              );
            }
            if (snapshot.hasData) {
              return ListenableProvider(
                create: (_) => widget.presenter,
                child: SurveyItems(snapshot.data!)
              );
            }
            return SizedBox(height: 0);
          }
        );
      },
    ),
  );
}
```

**Note:** `loadData()` is called in `build()` - this is intentional for simplicity. In production, you might call it in `initState()` or use a better lifecycle approach.

---

**Step 2: Presenter Loads Data**

File: [lib/presentation/presenters/getx_surveys_presenter.dart](../lib/presentation/presenters/getx_surveys_presenter.dart:18-39)

```dart
Future<void> loadData() async {
  try {
    isLoading = true;
    final surveys = await loadSurveys.load();
    _surveys.value = surveys
      .map((survey) => SurveyViewModel(
        id: survey.id,
        question: survey.question,
        date: DateFormat('dd MMM yyyy').format(survey.dateTime),
        didAnswer: survey.didAnswer
      ))
      .toList();
  } on DomainError catch(error) {
    if (error == DomainError.accessDenied) {
      isSessionExpired = true;
    } else {
      _surveys.subject.addError(UIError.unexpected.description);
    }
  } finally {
    isLoading = false;
  }
}
```

**Key points:**
1. Shows loading spinner
2. Calls `loadSurveys.load()` (composite with fallback)
3. Transforms entities to ViewModels
4. Updates stream (UI rebuilds)
5. Handles errors (session expired or general error)
6. Always hides loading (finally block)

---

**Step 3: Composite Tries Remote First**

File: [lib/main/composites/remote_load_surveys_with_local_fallback.dart](../lib/main/composites/remote_load_surveys_with_local_fallback.dart:15-27)

```dart
Future<List<SurveyEntity>> load() async {
  try {
    final surveys = await remote.load();
    await local.save(surveys);
    return surveys;
  } catch(error) {
    if (error == DomainError.accessDenied) {
      rethrow;
    }
    await local.validate();
    return await local.load();
  }
}
```

**Try Remote:**
```
remote.load()
  ↓
RemoteLoadSurveys makes HTTP GET request
  ↓
If successful:
  - Save surveys to local cache
  - Return surveys
```

---

**Step 4: Remote Load Surveys**

File: [lib/data/usecases/load_surveys/remote_load_surveys.dart](../lib/data/usecases/load_surveys/remote_load_surveys.dart)

```dart
class RemoteLoadSurveys implements LoadSurveys {
  final HttpClient httpClient;
  final String url;

  Future<List<SurveyEntity>> load() async {
    try {
      final httpResponse = await httpClient.request(url: url, method: 'get');
      return httpResponse
        .map<SurveyEntity>((json) => RemoteSurveyModel.fromJson(json).toEntity())
        .toList();
    } on HttpError catch (error) {
      throw error == HttpError.forbidden
        ? DomainError.accessDenied
        : DomainError.unexpected;
    }
  }
}
```

**HTTP Request:**
```
GET https://fordevs.herokuapp.com/api/surveys
Headers: {
  "content-type": "application/json",
  "accept": "application/json",
  "x-access-token": "eyJhbGci..." ← Added by AuthorizeHttpClientDecorator
}
```

---

**Step 5: Authorize Decorator Adds Token**

File: [lib/main/decorators/authorize_http_client_decorator.dart](../lib/main/decorators/authorize_http_client_decorator.dart:15-33)

```dart
Future<dynamic> request({
  required String url,
  required String method,
  Map? body,
  Map? headers,
}) async {
  try {
    final token = await fetchSecureCacheStorage.fetch('token');
    final authorizedHeaders = headers ?? {}
      ..addAll({'x-access-token': token});
    return await decoratee.request(
      url: url,
      method: method,
      body: body,
      headers: authorizedHeaders
    );
  } catch(error) {
    if (error is HttpError && error != HttpError.forbidden) {
      rethrow;
    } else {
      await deleteSecureCacheStorage.delete('token');
      throw HttpError.forbidden;
    }
  }
}
```

**Flow:**
```
1. Fetch token from secure storage
2. Add token to headers: {"x-access-token": "..."}
3. Delegate to HttpAdapter
4. If 403 error: delete token and throw forbidden
```

---

**Step 6: API Response**

**Success Response (200):**
```json
[
  {
    "id": "1",
    "question": "Qual é seu framework favorito?",
    "date": "2024-01-15T10:00:00Z",
    "didAnswer": false
  },
  {
    "id": "2",
    "question": "Você prefere TypeScript ou JavaScript?",
    "date": "2024-01-14T10:00:00Z",
    "didAnswer": true
  }
]
```

**Transformation:**
```
JSON Array
  ↓
List<Map<String, dynamic>>
  ↓
List<RemoteSurveyModel>
  ↓
List<SurveyEntity>
```

---

**Step 7: Save to Cache (Success Path)**

File: [lib/data/usecases/load_surveys/local_load_surveys.dart](../lib/data/usecases/load_surveys/local_load_surveys.dart)

```dart
Future<void> save(List<SurveyEntity> surveys) async {
  try {
    await cacheStorage.save(
      key: 'surveys',
      value: surveys
        .map((entity) => LocalSurveyModel.fromEntity(entity).toJson())
        .toList()
    );
  } catch (error) {
    throw DomainError.unexpected;
  }
}
```

**Flow:**
```
List<SurveyEntity>
  ↓
List<LocalSurveyModel>
  ↓
List<Map<String, dynamic>> (JSON)
  ↓
LocalStorageAdapter.save(key: 'surveys', value: [...])
  ↓
Saved to device storage
```

**Why cache?**
- Next time user opens app without internet → load from cache
- Faster subsequent loads

---

**Step 8: Transform to ViewModels**

File: [lib/presentation/presenters/getx_surveys_presenter.dart](../lib/presentation/presenters/getx_surveys_presenter.dart:22-29)

```dart
_surveys.value = surveys
  .map((survey) => SurveyViewModel(
    id: survey.id,
    question: survey.question,
    date: DateFormat('dd MMM yyyy').format(survey.dateTime),
    didAnswer: survey.didAnswer
  ))
  .toList();
```

**Entity → ViewModel:**
```
SurveyEntity(
  id: "1",
  question: "Qual é seu framework favorito?",
  dateTime: DateTime(2024, 1, 15, 10, 0, 0),
  didAnswer: false
)
  ↓
SurveyViewModel(
  id: "1",
  question: "Qual é seu framework favorito?",
  date: "15 Jan 2024", ← Formatted for display
  didAnswer: false
)
```

---

**Step 9: UI Updates**

```dart
StreamBuilder<List<SurveyViewModel>>(
  stream: widget.presenter.surveysStream,
  builder: (context, snapshot) {
    if (snapshot.hasData) {
      return SurveyItems(snapshot.data!); // ← Renders list
    }
    return SizedBox(height: 0);
  }
)
```

**Result:**
```
_surveys.value = [...]
  ↓
Stream emits new list
  ↓
StreamBuilder receives list
  ↓
Builds SurveyItems widget
  ↓
User sees surveys!
```

---

### Offline Flow: No Internet

**What happens when user has no internet?**

```
remote.load()
  ↓
HTTP request times out (10 seconds)
  ↓
HttpAdapter throws HttpError.serverError
  ↓
RemoteLoadSurveys throws DomainError.unexpected
  ↓
RemoteLoadSurveysWithLocalFallback catches error
  ↓
error != DomainError.accessDenied (so don't rethrow)
  ↓
local.validate()
  ↓
local.load()
  ↓
Load surveys from cache
  ↓
Return cached surveys
  ↓
User sees surveys (from cache!)
```

---

**Local Load Implementation:**

File: [lib/data/usecases/load_surveys/local_load_surveys.dart](../lib/data/usecases/load_surveys/local_load_surveys.dart)

```dart
Future<void> validate() async {
  try {
    final data = await cacheStorage.fetch('surveys');
    if (data.isEmpty) {
      throw Exception();
    }
  } catch (error) {
    await cacheStorage.delete('surveys');
    throw DomainError.unexpected;
  }
}

Future<List<SurveyEntity>> load() async {
  try {
    final data = await cacheStorage.fetch('surveys');
    if (data.isEmpty) {
      throw Exception();
    }
    return data
      .map<SurveyEntity>((json) => LocalSurveyModel.fromJson(json).toEntity())
      .toList();
  } catch (error) {
    throw DomainError.unexpected;
  }
}
```

**Validate ensures cache is not empty before attempting to load.**

---

### RouteAware Pattern: Reload on Return

**Problem:** User navigates to survey result, votes, then returns to surveys. The list should refresh to show updated "didAnswer" status.

**Solution:** RouteAware pattern.

File: [lib/ui/pages/surveys/surveys_page.dart](../lib/ui/pages/surveys/surveys_page.dart:20-62)

```dart
class _SurveysPageState extends State<SurveysPage>
  with LoadingManager, NavigationManager, SessionManager, RouteAware {

  @override
  Widget build(BuildContext context) {
    Get.find<RouteObserver>().subscribe(this, ModalRoute.of(context) as PageRoute);
    // ...
  }

  @override
  void didPopNext() {
    widget.presenter.loadData(); // ← Reload when returning
  }

  @override
  void dispose() {
    Get.find<RouteObserver>().unsubscribe(this);
    super.dispose();
  }
}
```

**Flow:**
```
User on SurveysPage
  ↓
Taps survey
  ↓
Navigate to SurveyResultPage
  ↓
User votes
  ↓
Presses back button
  ↓
didPopNext() called
  ↓
loadData() called again
  ↓
Fetches fresh data from API
  ↓
List updates with didAnswer = true
```

---

## State Management System

### Overview

State management is handled via:
- **GetX** reactive variables (`Rx<T>`)
- **Streams** for UI updates
- **Mixins** for common state patterns

### State Flow Diagram

```
┌──────────────────────────────────────────────────────────┐
│                    STATE MANAGEMENT                       │
└──────────────────────────────────────────────────────────┘

PRESENTATION LAYER:
┌─────────────────────────────┐
│  GetxPresenter              │
│                             │
│  final _state = Rx<T>(...)  │  ← Reactive variable
│  Stream<T> get stream       │
│    => _state.stream         │  ← Exposed as stream
│                             │
│  set state(T value)         │
│    => _state.value = value  │  ← Update triggers notification
└─────────────────────────────┘
              │
              │ emits on change
              ↓
       ┌──────────────┐
       │   Stream<T>  │
       └──────────────┘
              │
              │ listened by
              ↓
UI LAYER:
┌─────────────────────────────┐
│  StreamBuilder<T>           │
│    stream: presenter.stream │
│    builder: (_, snapshot) { │
│      return Widget(         │
│        data: snapshot.data  │
│      );                     │
│    }                        │
└─────────────────────────────┘
```

---

### Common State Patterns

**1. Loading State**

**Presenter (Mixin):**
```dart
mixin LoadingManager on GetxController {
  final _isLoading = false.obs;
  Stream<bool> get isLoadingStream => _isLoading.stream;

  set isLoading(bool value) => _isLoading.value = value;
}
```

**Usage in Presenter:**
```dart
Future<void> loadData() async {
  isLoading = true;  // Emit true
  await doWork();
  isLoading = false; // Emit false
}
```

**UI Mixin:**
```dart
mixin LoadingManager {
  void handleLoading(BuildContext context, Stream<bool> stream) {
    stream.listen((isLoading) {
      if (isLoading) {
        showDialog(...SpinnerDialog());
      } else {
        Navigator.of(context).pop();
      }
    });
  }
}
```

---

**2. Form State**

**Presenter:**
```dart
final _emailError = Rx<UIError?>(null);
Stream<UIError?> get emailErrorStream => _emailError.stream;

void validateEmail(String email) {
  _email = email;
  _emailError.value = _validateField('email'); // Emit error or null
}
```

**UI:**
```dart
StreamBuilder<UIError?>(
  stream: presenter.emailErrorStream,
  builder: (context, snapshot) {
    return TextFormField(
      errorText: snapshot.data?.description,
      onChanged: presenter.validateEmail,
    );
  },
)
```

**Real-time updates:**
```
User types → validateEmail() → Update Rx → Stream emits → UI rebuilds
```

---

**3. Data State**

**Presenter:**
```dart
final _surveys = Rx<List<SurveyViewModel>>([]);
Stream<List<SurveyViewModel>> get surveysStream => _surveys.stream;

Future<void> loadData() async {
  final data = await loadSurveys.load();
  _surveys.value = data.map(...).toList(); // Emit new list
}
```

**UI:**
```dart
StreamBuilder<List<SurveyViewModel>>(
  stream: presenter.surveysStream,
  builder: (context, snapshot) {
    if (snapshot.hasError) return ErrorWidget();
    if (snapshot.hasData) return ListView(snapshot.data);
    return LoadingWidget();
  },
)
```

---

**4. Navigation State**

**Presenter:**
```dart
mixin NavigationManager on GetxController {
  final _navigateTo = RxString('');
  Stream<String> get navigateToStream =>
    _navigateTo.stream.where((route) => route.isNotEmpty);

  set navigateTo(String value) => _navigateTo.value = value;
}
```

**Usage:**
```dart
Future<void> login() async {
  // ... auth logic
  navigateTo = '/surveys'; // Emit route
}
```

**UI:**
```dart
mixin NavigationManager {
  void handleNavigation(Stream<String> stream, {bool clear = false}) {
    stream.listen((route) {
      clear ? Get.offAllNamed(route) : Get.toNamed(route);
    });
  }
}
```

---

### Benefits of This Approach

**✅ Unidirectional Data Flow**
```
User Action → Presenter → State Update → Stream Emit → UI Rebuild
```

**✅ Separation of Concerns**
- Presenter manages state
- UI displays state
- No business logic in UI

**✅ Testability**
```dart
test('Should emit loading states', () async {
  final states = [];
  presenter.isLoadingStream.listen(states.add);

  await presenter.loadData();

  expect(states, [true, false]);
});
```

**✅ Reactive**
- UI automatically updates when state changes
- No manual `setState()` calls
- No boilerplate

---

## Navigation System

### Overview

Navigation is handled via:
- **GetX navigation** (`Get.toNamed`, `Get.offAllNamed`)
- **Stream-based triggers** from presenters
- **UI mixins** that listen and navigate
- **Parametrized routes** for dynamic data

### Navigation Flow

```
Presenter                    UI Mixin                   GetX Router
   │                            │                          │
   │ navigateTo = '/surveys'    │                          │
   │─────────────────────────→  │                          │
   │                            │                          │
   │                            │ Get.offAllNamed(route)   │
   │                            │─────────────────────────→│
   │                            │                          │
   │                            │                      Navigate!
```

---

### Navigation Examples

**1. Simple Navigation (Login → Surveys)**

```dart
// Presenter
navigateTo = '/surveys';

// UI Mixin listens
handleNavigation(presenter.navigateToStream, clear: true);

// Results in
Get.offAllNamed('/surveys'); // Clear stack and navigate
```

---

**2. Parametrized Navigation (Surveys → Survey Result)**

**Presenter:**
```dart
void goToSurveyResult(String surveyId) {
  navigateTo = '/survey_result/$surveyId';
}
```

**Route definition:**

File: [lib/main/main.dart](../lib/main/main.dart)

```dart
GetPage(
  name: '/survey_result/:survey_id',
  page: makeSurveyResultPage,
),
```

**Factory extracts parameter:**

File: [lib/main/factories/pages/survey_result/survey_result_page_factory.dart](../lib/main/factories/pages/survey_result/survey_result_page_factory.dart)

```dart
Widget makeSurveyResultPage() {
  final args = Get.parameters;
  return SurveyResultPage(
    makeGetxSurveyResultPresenter(args['survey_id']!)
  );
}
```

**Result:**
```
User taps survey with id "123"
  ↓
goToSurveyResult("123")
  ↓
navigateTo = '/survey_result/123'
  ↓
Get.toNamed('/survey_result/123')
  ↓
Route matches pattern '/survey_result/:survey_id'
  ↓
Extract parameter: survey_id = "123"
  ↓
Create presenter with surveyId="123"
  ↓
Load survey result for survey 123
```

---

**3. Clear Stack vs Keep Stack**

**Clear stack (Login → Surveys):**
```dart
handleNavigation(stream, clear: true);
// Uses Get.offAllNamed()
// User can't press back to login
```

**Keep stack (Surveys → Survey Result):**
```dart
handleNavigation(stream);
// Uses Get.toNamed()
// User can press back to surveys
```

---

## Error Handling System

### Overview

Three-layer error system:
1. **HttpError** (Infrastructure layer)
2. **DomainError** (Domain layer)
3. **UIError** (UI layer)

Each layer only knows about its own errors!

### Error Flow

```
Infrastructure Layer    →    Data Layer    →    Presentation    →    UI Layer
   HttpError                 DomainError           UIError            String

┌──────────────┐         ┌──────────────┐     ┌──────────────┐   ┌─────────────┐
│unauthorized  │────────→│invalidCred...│────→│invalidCred...│──→│"Credenciais │
│              │         │              │     │              │   │ inválidas"  │
├──────────────┤         ├──────────────┤     ├──────────────┤   ├─────────────┤
│forbidden     │────────→│accessDenied  │────→│(session exp) │──→│Navigate to  │
│              │         │              │     │              │   │login        │
├──────────────┤         ├──────────────┤     ├──────────────┤   ├─────────────┤
│serverError   │────────→│unexpected    │────→│unexpected    │──→│"Algo errado │
│badRequest    │         │              │     │              │   │aconteceu"   │
│notFound      │         │              │     │              │   │             │
└──────────────┘         └──────────────┘     └──────────────┘   └─────────────┘
```

---

### Error Mapping Examples

**Infrastructure → Data:**

File: [lib/data/usecases/authentication/remote_authentication.dart](../lib/data/usecases/authentication/remote_authentication.dart:21-25)

```dart
} on HttpError catch(error) {
  throw error == HttpError.unauthorized
    ? DomainError.invalidCredentials
    : DomainError.unexpected;
}
```

---

**Data → Presentation:**

File: [lib/presentation/presenters/getx_login_presenter.dart](../lib/presentation/presenters/getx_login_presenter.dart:69-75)

```dart
} on DomainError catch (error) {
  switch (error) {
    case DomainError.invalidCredentials:
      mainError = UIError.invalidCredentials;
      break;
    default:
      mainError = UIError.unexpected;
      break;
  }
  isLoading = false;
}
```

---

**Presentation → UI:**

File: [lib/ui/helpers/errors/ui_error.dart](../lib/ui/helpers/errors/ui_error.dart)

```dart
enum UIError {
  requiredField,
  invalidField,
  invalidCredentials,
  emailInUse,
  unexpected
}

extension UIErrorExtensions on UIError {
  String get description {
    switch (this) {
      case UIError.requiredField: return R.string.msgRequiredField;
      case UIError.invalidField: return R.string.msgInvalidField;
      case UIError.invalidCredentials: return R.string.msgInvalidCredentials;
      case UIError.emailInUse: return R.string.msgEmailInUse;
      default: return R.string.msgUnexpectedError;
    }
  }
}
```

**Portuguese strings:**
```dart
String get msgInvalidCredentials => 'Credenciais inválidas.';
String get msgUnexpectedError => 'Algo errado aconteceu. Tente novamente em breve.';
```

---

### Displaying Errors

**Error Stream:**
```dart
Stream<UIError?> get mainErrorStream => _mainError.stream;

set mainError(UIError? value) => _mainError.value = value;
```

**UI Mixin:**

File: [lib/ui/mixins/ui_error_manager.dart](../lib/ui/mixins/ui_error_manager.dart)

```dart
mixin UIErrorManager {
  void handleMainError(BuildContext context, Stream<UIError?> stream) {
    stream.listen((error) {
      if (error != null) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            backgroundColor: Colors.red[900],
            content: Text(error.description),
          ),
        );
      }
    });
  }
}
```

**Result:**
```
mainError = UIError.invalidCredentials
  ↓
Stream emits UIError.invalidCredentials
  ↓
Mixin listens
  ↓
Shows red snackbar: "Credenciais inválidas"
```

---

## Validation System

### Overview

The validation system:
- Validates form fields in real-time
- Uses Strategy pattern (multiple validators)
- Uses Composite pattern (combines validators)
- Uses Builder pattern (fluent API)

### Validation Flow

```
┌──────────────────────────────────────────────────────────┐
│                   VALIDATION SYSTEM                       │
└──────────────────────────────────────────────────────────┘

User types in field
  ↓
UI: onChanged(value)
  ↓
Presenter: validateField(value)
  ↓
ValidationComposite: validate(field, input)
  ↓
Filter validators for this field
  ↓
Run each validator in sequence
  ├─ RequiredFieldValidation → null (valid)
  ├─ EmailValidation → ValidationError.invalidField
  └─ Return first error found
  ↓
Presenter: Map ValidationError → UIError
  ↓
Update stream
  ↓
UI: StreamBuilder rebuilds
  ↓
Show error message under field
```

---

### Validation Example: Email Field

**Step 1: Build Validators**

File: [lib/main/factories/pages/login/login_validation_factory.dart](../lib/main/factories/pages/login/login_validation_factory.dart)

```dart
List<FieldValidation> makeLoginValidations() {
  return [
    ...ValidationBuilder.field('email').required().email().build(),
    ...ValidationBuilder.field('password').required().min(3).build(),
  ];
}
```

**Expands to:**
```dart
[
  RequiredFieldValidation('email'),
  EmailValidation('email'),
  RequiredFieldValidation('password'),
  MinLengthValidation(field: 'password', size: 3),
]
```

---

**Step 2: User Types**

```
User types: "t"
  ↓
EmailInput widget: onChanged("t")
  ↓
presenter.validateEmail("t")
```

---

**Step 3: Presenter Validates**

```dart
void validateEmail(String email) {
  _email = email;
  _emailError.value = _validateField('email');
  _validateForm();
}

UIError? _validateField(String field) {
  final formData = {'email': _email, 'password': _password};
  final error = validation.validate(field: field, input: formData);
  switch (error) {
    case ValidationError.invalidField: return UIError.invalidField;
    case ValidationError.requiredField: return UIError.requiredField;
    default: return null;
  }
}
```

---

**Step 4: ValidationComposite Runs**

```dart
ValidationError? validate({ required String field, required Map input}) {
  ValidationError? error;
  for (final validation in validations.where((v) => v.field == field)) {
    error = validation.validate(input);
    if (error != null) {
      return error;
    }
  }
  return error;
}
```

**For field 'email' with input "t":**
```
Filter validators: [RequiredFieldValidation('email'), EmailValidation('email')]
  ↓
Run RequiredFieldValidation:
  - input['email'] = "t"
  - isNotEmpty? Yes
  - Return null (valid)
  ↓
Run EmailValidation:
  - input['email'] = "t"
  - Matches email regex? No
  - Return ValidationError.invalidField
  ↓
Return ValidationError.invalidField (first error)
```

---

**Step 5: Map to UI Error**

```dart
switch (error) {
  case ValidationError.invalidField: return UIError.invalidField;
  // ...
}
```

**Result:**
```
_emailError.value = UIError.invalidField
```

---

**Step 6: UI Shows Error**

```dart
StreamBuilder<UIError?>(
  stream: presenter.emailErrorStream,
  builder: (context, snapshot) {
    return TextFormField(
      errorText: snapshot.hasData ? snapshot.data?.description : null,
      // ...
    );
  },
)
```

**User sees:**
```
┌─────────────────────────┐
│ Email                   │
│ t                       │
│ Campo inválido          │ ← Error text
└─────────────────────────┘
```

---

**Step 7: User Completes Email**

```
User types: "test@email.com"
  ↓
validateEmail("test@email.com")
  ↓
RequiredFieldValidation → null (has value)
EmailValidation → null (valid email)
  ↓
_emailError.value = null
  ↓
Error message disappears!
```

---

## Caching & Offline Support

### Overview

The app supports offline usage via:
- **Local cache** for surveys
- **Secure storage** for tokens
- **Fallback strategy** (try remote, fall back to local)
- **Automatic caching** on successful remote loads

### Cache Architecture

```
┌──────────────────────────────────────────────────────────┐
│                      CACHE SYSTEM                         │
└──────────────────────────────────────────────────────────┘

TWO STORAGE TYPES:

1. SECURE STORAGE (Sensitive Data)
   ┌─────────────────────────────┐
   │  SecureStorageAdapter       │
   │  (FlutterSecureStorage)     │
   ├─────────────────────────────┤
   │  Stores:                    │
   │  • Authentication token     │
   │                             │
   │  OS-level encryption        │
   │  • iOS: Keychain            │
   │  • Android: Keystore        │
   └─────────────────────────────┘

2. LOCAL STORAGE (Non-sensitive Data)
   ┌─────────────────────────────┐
   │  LocalStorageAdapter        │
   │  (localstorage package)     │
   ├─────────────────────────────┤
   │  Stores:                    │
   │  • Surveys list             │
   │  • Survey results           │
   │                             │
   │  File-based storage         │
   └─────────────────────────────┘
```

---

### Offline Flow Example

**Scenario:** User opens app with no internet, but has previously loaded surveys.

```
1. SurveysPage loads
   ↓
2. presenter.loadData()
   ↓
3. RemoteLoadSurveysWithLocalFallback.load()
   ↓
4. Try: remote.load()
   ↓
5. HttpAdapter makes GET request
   ↓
6. Timeout after 10 seconds (no internet)
   ↓
7. Throws HttpError.serverError
   ↓
8. RemoteLoadSurveys throws DomainError.unexpected
   ↓
9. Composite catches error
   ↓
10. error != DomainError.accessDenied (don't rethrow)
    ↓
11. local.validate() - Check if cache exists and is valid
    ↓
12. local.load() - Load from LocalStorageAdapter
    ↓
13. Fetch from device storage: key = 'surveys'
    ↓
14. Parse JSON to LocalSurveyModel
    ↓
15. Convert to SurveyEntity
    ↓
16. Return to presenter
    ↓
17. Transform to ViewModels
    ↓
18. Update stream
    ↓
19. UI shows cached surveys!
```

**User sees surveys despite no internet! ✅**

---

### Cache Invalidation

**When is cache cleared?**

1. **On successful remote load** - Cache is overwritten with fresh data
2. **On validation failure** - If cache is corrupted or empty, delete it
3. **On 403 error** - Session expired, delete token

**Example: Token deletion on 403**

File: [lib/main/decorators/authorize_http_client_decorator.dart](../lib/main/decorators/authorize_http_client_decorator.dart:25-32)

```dart
} catch(error) {
  if (error is HttpError && error != HttpError.forbidden) {
    rethrow;
  } else {
    await deleteSecureCacheStorage.delete('token');
    throw HttpError.forbidden;
  }
}
```

---

## Session Management

### Overview

Session management handles:
- Detecting session expiration (403 errors)
- Deleting invalid tokens
- Navigating to login
- Showing user-friendly message

### Session Expiration Flow

```
User on SurveysPage
  ↓
Presenter calls loadSurveys.load()
  ↓
Remote makes GET /api/surveys
  ↓
AuthorizeHttpClientDecorator adds token
  ↓
API responds: 403 Forbidden (token expired)
  ↓
HttpAdapter throws HttpError.forbidden
  ↓
RemoteLoadSurveys throws DomainError.accessDenied
  ↓
Composite rethrows (session expired is special)
  ↓
Presenter catches DomainError.accessDenied
  ↓
isSessionExpired = true
  ↓
UI SessionManager mixin listens
  ↓
Navigate to login
  ↓
Show message: "Sessão expirada"
```

---

### Implementation

**Presentation Mixin:**

File: [lib/presentation/mixins/session_manager.dart](../lib/presentation/mixins/session_manager.dart)

```dart
mixin SessionManager on GetxController {
  final _isSessionExpired = false.obs;
  Stream<bool> get isSessionExpiredStream => _isSessionExpired.stream;

  set isSessionExpired(bool value) => _isSessionExpired.value = value;
}
```

**Presenter usage:**

```dart
} on DomainError catch(error) {
  if (error == DomainError.accessDenied) {
    isSessionExpired = true; // Trigger session expiration
  } else {
    _surveys.subject.addError(UIError.unexpected.description);
  }
}
```

**UI Mixin:**

File: [lib/ui/mixins/session_manager.dart](../lib/ui/mixins/session_manager.dart)

```dart
mixin SessionManager {
  void handleSessionExpired(Stream<bool> stream) {
    stream.listen((isExpired) async {
      if (isExpired) {
        Get.offAllNamed('/login');
        Get.snackbar(
          R.string.expired,
          R.string.msgSessionExpired,
          backgroundColor: Colors.red[900],
          colorText: Colors.white,
        );
      }
    });
  }
}
```

**Result:**
```
isSessionExpired = true
  ↓
Navigate to login (clear stack)
  ↓
Show snackbar: "Sua sessão expirou. Por favor, faça login novamente."
```

---

## Key Takeaways

### System Interactions Summary

All systems work together:

```
1. User logs in (Authentication System)
   ↓
2. Token saved (Caching System)
   ↓
3. Navigate to surveys (Navigation System)
   ↓
4. Load surveys with token (Survey System + Session Management)
   ↓
5. Cache surveys (Caching System)
   ↓
6. Display surveys (State Management System)
   ↓
7. If error, show message (Error Handling System)
   ↓
8. If offline, load cache (Offline Support)
```

---

### Design Patterns in Action

**Authentication uses:**
- Factory (create all dependencies)
- Dependency Injection (inject into presenter)
- Adapter (HttpAdapter, SecureStorageAdapter)
- Decorator (AuthorizeHttpClientDecorator)
- Observer (streams)
- Command (use cases)

**Survey System uses:**
- Composite (RemoteLoadSurveysWithLocalFallback)
- Repository (LoadSurveys abstraction)
- ViewModel (Entity → ViewModel transformation)
- Observer (surveysStream)
- Strategy (different load strategies)

---

### Layer Collaboration

**Complete request touches all layers:**
```
UI → Presentation → Domain ← Data ← Infra → External
```

**Each layer has clear responsibility:**
- UI: Display
- Presentation: Coordinate
- Domain: Define contracts
- Data: Implement contracts
- Infra: Wrap external libs

---

## Next Steps

Now that you understand how major components work:

1. **[Data Flow Guide](05-data-flow-guide.md)** - Detailed flow diagrams
2. **[Getting Started Guide](06-getting-started-guide.md)** - Build your own feature
3. **[Testing Strategy](07-testing-strategy.md)** - Test these components

---

**Documentation Version:** 1.0
**Last Updated:** 2025

---

**You now understand the inner workings of a production-grade Flutter app!** 🎉
