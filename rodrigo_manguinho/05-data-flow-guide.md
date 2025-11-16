# 05 - Data Flow Guide

## Table of Contents

1. [Introduction](#introduction)
2. [What You'll Learn](#what-youll-learn)
3. [Flow Notation Guide](#flow-notation-guide)
4. [Complete Flow: User Login](#complete-flow-user-login)
5. [Complete Flow: Load Surveys](#complete-flow-load-surveys)
6. [Complete Flow: Vote on Survey](#complete-flow-vote-on-survey)
7. [Complete Flow: Session Expiration](#complete-flow-session-expiration)
8. [Data Transformation Journey](#data-transformation-journey)
9. [State Update Propagation](#state-update-propagation)
10. [Error Flow Patterns](#error-flow-patterns)
11. [Key Takeaways](#key-takeaways)
12. [Next Steps](#next-steps)

---

## Introduction

This document provides **visual representations** of how data flows through the Clean Flutter App. We'll trace complete user journeys with detailed sequence diagrams, showing every method call, data transformation, and state update.

**Why flow diagrams matter:**
Seeing the complete picture of data flow helps you understand how all the pieces fit together and makes debugging much easier.

---

## What You'll Learn

By the end of this document, you will understand:

- ✅ Complete request/response cycles
- ✅ Data transformations at each layer
- ✅ State propagation patterns
- ✅ Error handling flows
- ✅ How to trace bugs through layers
- ✅ How to add new features following existing patterns

---

## Flow Notation Guide

### Symbols Used

```
│  Vertical flow (sequential steps)
├─ Branch/fork
└─ End of branch
→  Horizontal flow (calls/returns)
↓  Downward flow
↑  Upward return
✓  Success case
✗  Error case
```

### Layer Abbreviations

```
[UI]      - UI Layer (pages, widgets)
[P]       - Presentation Layer (presenters)
[D]       - Domain Layer (entities, use cases)
[DATA]    - Data Layer (implementations, models)
[INFRA]   - Infrastructure Layer (adapters)
[MAIN]    - Main Layer (factories, composites, decorators)
[EXT]     - External (API, storage, packages)
```

---

## Complete Flow: User Login

### Success Path

```
══════════════════════════════════════════════════════════════════════
FLOW 1: USER LOGIN - SUCCESS
══════════════════════════════════════════════════════════════════════

[UI] LoginPage
  │
  │ User fills form: email="test@email.com", password="123456"
  │
  ├─ User types in EmailInput
  │   │
  │   └→ [UI] EmailInput.onChanged("test@email.com")
  │       │
  │       └→ [P] GetxLoginPresenter.validateEmail("test@email.com")
  │           │
  │           ├─ Store: _email = "test@email.com"
  │           │
  │           ├─ Call: _validateField('email')
  │           │   │
  │           │   └→ [P] ValidationComposite.validate(field: 'email', input: {...})
  │           │       │
  │           │       ├─ Filter validations for 'email'
  │           │       │   • RequiredFieldValidation('email')
  │           │       │   • EmailValidation('email')
  │           │       │
  │           │       ├─ Run RequiredFieldValidation
  │           │       │   └→ Result: null (valid)
  │           │       │
  │           │       ├─ Run EmailValidation
  │           │       │   └→ Result: null (valid email format)
  │           │       │
  │           │       └─ Return: null (no errors)
  │           │
  │           ├─ Update: _emailError.value = null
  │           │   └→ Stream emits: null
  │           │       └→ [UI] StreamBuilder rebuilds
  │           │           └─ No error text shown ✓
  │           │
  │           └─ Call: _validateForm()
  │               └─ Check: emailError==null && passwordError==null
  │                   └→ Update: isFormValid = true
  │                       └→ Stream emits: true
  │                           └→ [UI] Login button enabled ✓
  │
  └─ User taps "Login" button
      │
      └→ [UI] LoginButton.onPressed()
          │
          └→ [P] GetxLoginPresenter.auth()
              │
              ├─ Set: mainError = null
              │   └→ Stream emits: null (clear previous errors)
              │
              ├─ Set: isLoading = true
              │   └→ Stream emits: true
              │       └→ [UI] LoadingManager shows SpinnerDialog ⏳
              │
              ├─ Call: authentication.auth(params)
              │   │
              │   └→ [DATA] RemoteAuthentication.auth(params)
              │       │
              │       ├─ Transform: AuthenticationParams → RemoteAuthenticationParams
              │       │   │ Domain: AuthenticationParams(email, secret)
              │       │   └→ Data: RemoteAuthenticationParams(email, password)
              │       │
              │       ├─ Serialize: toJson()
              │       │   └→ {"email": "test@email.com", "password": "123456"}
              │       │
              │       ├─ Call: httpClient.request(url, method: 'post', body)
              │       │   │
              │       │   └→ [MAIN] AuthorizeHttpClientDecorator.request(...)
              │       │       │
              │       │       ├─ Fetch token from secure storage
              │       │       │   └→ [INFRA] SecureStorageAdapter.fetch('token')
              │       │       │       └→ [EXT] FlutterSecureStorage.read(key: 'token')
              │       │       │           └→ Return: null (first login, no token yet)
              │       │       │
              │       │       ├─ Add headers: {'x-access-token': null}
              │       │       │
              │       │       └─ Call: decoratee.request(...)
              │       │           │
              │       │           └→ [INFRA] HttpAdapter.request(...)
              │       │               │
              │       │               ├─ Set headers:
              │       │               │   • content-type: application/json
              │       │               │   • accept: application/json
              │       │               │
              │       │               ├─ Encode body: jsonEncode(body)
              │       │               │   └→ '{"email":"test@email.com","password":"123456"}'
              │       │               │
              │       │               ├─ Make HTTP request
              │       │               │   └→ [EXT] http.Client.post(...)
              │       │               │       │
              │       │               │       POST https://fordevs.herokuapp.com/api/login
              │       │               │       Headers: {...}
              │       │               │       Body: '{"email":...}'
              │       │               │       │
              │       │               │       └→ Response: 200 OK
              │       │               │           Body: '{"accessToken":"eyJhbGci..."}'
              │       │               │
              │       │               └─ Handle response:
              │       │                   │
              │       │                   ├─ Status: 200
              │       │                   ├─ Decode: jsonDecode(response.body)
              │       │                   └→ Return: {"accessToken": "eyJhbGci..."}
              │       │
              │       ├─ Parse JSON to Model
              │       │   └→ [DATA] RemoteAccountModel.fromJson(response)
              │       │       │
              │       │       ├─ Validate: json.containsKey('accessToken')? ✓
              │       │       └→ Return: RemoteAccountModel(accessToken: "eyJhbGci...")
              │       │
              │       └─ Convert to Entity
              │           └→ model.toEntity()
              │               └→ [D] AccountEntity(token: "eyJhbGci...")
              │
              ├─ Save account
              │   │
              │   └→ [P] saveCurrentAccount.save(account)
              │       │
              │       └→ [DATA] LocalSaveCurrentAccount.save(account)
              │           │
              │           └→ saveSecureCacheStorage.save(key: 'token', value: account.token)
              │               │
              │               └→ [INFRA] SecureStorageAdapter.save(...)
              │                   │
              │                   └→ [EXT] FlutterSecureStorage.write(key: 'token', value: "eyJhbGci...")
              │                       └→ Saved to OS Keychain/Keystore ✓
              │
              ├─ Set: navigateTo = '/surveys'
              │   └→ Stream emits: '/surveys'
              │       └→ [UI] NavigationManager.handleNavigation()
              │           └→ Get.offAllNamed('/surveys')
              │               └─ Clear navigation stack
              │               └─ Navigate to SurveysPage ✓
              │
              └─ (isLoading automatically becomes false when navigation occurs)

══════════════════════════════════════════════════════════════════════
END: User is logged in, token saved, navigated to surveys ✓
══════════════════════════════════════════════════════════════════════
```

---

### Error Path: Invalid Credentials

```
══════════════════════════════════════════════════════════════════════
FLOW 1B: USER LOGIN - INVALID CREDENTIALS
══════════════════════════════════════════════════════════════════════

[UI] LoginPage
  │
  │ User enters: email="test@email.com", password="wrong"
  │
  └─ User taps "Login"
      │
      └→ [P] GetxLoginPresenter.auth()
          │
          ├─ Set: mainError = null
          ├─ Set: isLoading = true (show spinner)
          │
          ├─ Call: authentication.auth(params)
          │   │
          │   └→ [DATA] RemoteAuthentication.auth(params)
          │       │
          │       ├─ Prepare request: {"email":"test@email.com","password":"wrong"}
          │       │
          │       ├─ Call: httpClient.request(...)
          │       │   │
          │       │   └→ [INFRA] HttpAdapter.request(...)
          │       │       │
          │       │       ├─ HTTP POST to API
          │       │       │   └→ [EXT] API validates credentials
          │       │       │       └→ Wrong password!
          │       │       │           └→ Response: 401 Unauthorized
          │       │       │
          │       │       └─ Handle response:
          │       │           │
          │       │           ├─ Status: 401
          │       │           └→ throw HttpError.unauthorized ✗
          │       │
          │       └─ Catch HttpError:
          │           │
          │           ├─ error == HttpError.unauthorized? ✓
          │           └→ throw DomainError.invalidCredentials ✗
          │
          └─ Catch DomainError:
              │
              ├─ error == DomainError.invalidCredentials? ✓
              ├─ Set: mainError = UIError.invalidCredentials
              │   └→ Stream emits: UIError.invalidCredentials
              │       └→ [UI] UIErrorManager.handleMainError()
              │           └→ Show SnackBar:
              │               "Credenciais inválidas" (red) ✗
              │
              └─ Set: isLoading = false
                  └→ Stream emits: false
                      └→ [UI] Spinner hidden ✓

══════════════════════════════════════════════════════════════════════
END: User sees error, remains on login page
══════════════════════════════════════════════════════════════════════
```

---

## Complete Flow: Load Surveys

### Success with Caching

```
══════════════════════════════════════════════════════════════════════
FLOW 2: LOAD SURVEYS - SUCCESS WITH CACHE
══════════════════════════════════════════════════════════════════════

[UI] SurveysPage (user just logged in and navigated here)
  │
  │ Page builds
  │
  └→ Builder calls: widget.presenter.loadData()
      │
      └→ [P] GetxSurveysPresenter.loadData()
          │
          ├─ Set: isLoading = true
          │   └→ [UI] Shows loading spinner ⏳
          │
          ├─ Call: loadSurveys.load()
          │   │
          │   └→ [MAIN] RemoteLoadSurveysWithLocalFallback.load()
          │       │
          │       ├─ TRY: remote.load()
          │       │   │
          │       │   └→ [DATA] RemoteLoadSurveys.load()
          │       │       │
          │       │       ├─ Call: httpClient.request(url: '/api/surveys', method: 'get')
          │       │       │   │
          │       │       │   └→ [MAIN] AuthorizeHttpClientDecorator.request(...)
          │       │       │       │
          │       │       │       ├─ Fetch token
          │       │       │       │   └→ [INFRA] SecureStorageAdapter.fetch('token')
          │       │       │       │       └→ [EXT] FlutterSecureStorage.read(key: 'token')
          │       │       │       │           └→ Return: "eyJhbGci..." ✓
          │       │       │       │
          │       │       │       ├─ Add headers: {'x-access-token': "eyJhbGci..."}
          │       │       │       │
          │       │       │       └─ Call: decoratee.request(...)
          │       │       │           │
          │       │       │           └→ [INFRA] HttpAdapter.request(...)
          │       │       │               │
          │       │       │               ├─ HTTP GET to API
          │       │       │               │   └→ [EXT] API Server
          │       │       │               │       │
          │       │       │               │       GET /api/surveys
          │       │       │               │       Headers: {'x-access-token': "..."}
          │       │       │               │       │
          │       │       │               │       └→ Response: 200 OK
          │       │       │               │           Body: '[
          │       │       │               │             {"id":"1","question":"...","date":"...","didAnswer":false},
          │       │       │               │             {"id":"2","question":"...","date":"...","didAnswer":true}
          │       │       │               │           ]'
          │       │       │               │
          │       │       │               └─ Decode JSON
          │       │       │                   └→ Return: List<Map<String, dynamic>>
          │       │       │
          │       │       └─ Transform to entities:
          │       │           │
          │       │           ├─ For each JSON object:
          │       │           │   │
          │       │           │   ├─ [DATA] RemoteSurveyModel.fromJson(json)
          │       │           │   │   └→ Validate fields, create model
          │       │           │   │
          │       │           │   └─ model.toEntity()
          │       │           │       └→ [D] SurveyEntity(
          │       │           │           id: "1",
          │       │           │           question: "...",
          │       │           │           dateTime: DateTime(...),
          │       │           │           didAnswer: false
          │       │           │         )
          │       │           │
          │       │           └→ Return: List<SurveyEntity> (2 surveys) ✓
          │       │
          │       ├─ SUCCESS! Save to cache:
          │       │   │
          │       │   └→ local.save(surveys)
          │       │       │
          │       │       └→ [DATA] LocalLoadSurveys.save(surveys)
          │       │           │
          │       │           ├─ Transform: entities → models → JSON
          │       │           │   │
          │       │           │   └─ surveys.map((entity) =>
          │       │           │       LocalSurveyModel.fromEntity(entity).toJson()
          │       │           │     )
          │       │           │     └→ List<Map<String, dynamic>>
          │       │           │
          │       │           └─ Save to storage
          │       │               │
          │       │               └→ cacheStorage.save(key: 'surveys', value: [...])
          │       │                   │
          │       │                   └→ [INFRA] LocalStorageAdapter.save(...)
          │       │                       │
          │       │                       └→ [EXT] LocalStorage.setItem('surveys', [...])
          │       │                           └─ Saved to device file ✓
          │       │
          │       └→ Return: List<SurveyEntity> (from remote)
          │
          ├─ Transform to ViewModels:
          │   │
          │   └─ surveys.map((survey) =>
          │       SurveyViewModel(
          │         id: survey.id,
          │         question: survey.question,
          │         date: DateFormat('dd MMM yyyy').format(survey.dateTime),
          │         didAnswer: survey.didAnswer
          │       )
          │     )
          │     │
          │     └→ List<SurveyViewModel>:
          │         [
          │           SurveyViewModel(id: "1", question: "...", date: "15 Jan 2024", didAnswer: false),
          │           SurveyViewModel(id: "2", question: "...", date: "14 Jan 2024", didAnswer: true)
          │         ]
          │
          ├─ Set: _surveys.value = viewModels
          │   └→ Stream emits: List<SurveyViewModel>
          │       └→ [UI] StreamBuilder rebuilds
          │           └─ Shows SurveyItems (list of survey cards) ✓
          │
          └─ Set: isLoading = false (finally block)
              └→ [UI] Hides spinner ✓

══════════════════════════════════════════════════════════════════════
END: Surveys displayed, cached for offline use ✓
══════════════════════════════════════════════════════════════════════
```

---

### Offline Fallback

```
══════════════════════════════════════════════════════════════════════
FLOW 2B: LOAD SURVEYS - OFFLINE (CACHE FALLBACK)
══════════════════════════════════════════════════════════════════════

[UI] SurveysPage (user has no internet but previously loaded surveys)
  │
  └→ [P] GetxSurveysPresenter.loadData()
      │
      ├─ Set: isLoading = true
      │
      ├─ Call: loadSurveys.load()
      │   │
      │   └→ [MAIN] RemoteLoadSurveysWithLocalFallback.load()
      │       │
      │       ├─ TRY: remote.load()
      │       │   │
      │       │   └→ [DATA] RemoteLoadSurveys.load()
      │       │       │
      │       │       └─ Call: httpClient.request(...)
      │       │           │
      │       │           └→ [INFRA] HttpAdapter.request(...)
      │       │               │
      │       │               ├─ HTTP GET attempt
      │       │               │   └→ Timeout after 10 seconds ✗
      │       │               │       (No internet connection)
      │       │               │
      │       │               └→ throw HttpError.serverError ✗
      │       │
      │       ├─ CATCH error:
      │       │   │
      │       │   ├─ error == DomainError.accessDenied? No
      │       │   │
      │       │   ├─ Don't rethrow, try local cache
      │       │   │
      │       │   ├─ Call: local.validate()
      │       │   │   │
      │       │   │   └→ [DATA] LocalLoadSurveys.validate()
      │       │   │       │
      │       │   │       ├─ Fetch cached data
      │       │   │       │   └→ cacheStorage.fetch('surveys')
      │       │   │       │       └→ [EXT] Returns: List<Map> (cached surveys) ✓
      │       │   │       │
      │       │   │       ├─ Check: data.isEmpty? No
      │       │   │       └→ Validation passed ✓
      │       │   │
      │       │   └─ Call: local.load()
      │       │       │
      │       │       └→ [DATA] LocalLoadSurveys.load()
      │       │           │
      │       │           ├─ Fetch: cacheStorage.fetch('surveys')
      │       │           │   └→ [EXT] Returns cached JSON
      │       │           │
      │       │           ├─ Parse: LocalSurveyModel.fromJson(...)
      │       │           │
      │       │           ├─ Convert: model.toEntity()
      │       │           │   └→ [D] SurveyEntity
      │       │           │
      │       │           └→ Return: List<SurveyEntity> (from cache) ✓
      │       │
      │       └→ Return cached surveys
      │
      ├─ Transform to ViewModels (same as online)
      │
      ├─ Set: _surveys.value = viewModels
      │   └→ [UI] Shows surveys from cache ✓
      │
      └─ Set: isLoading = false

══════════════════════════════════════════════════════════════════════
END: User sees surveys from cache (offline mode) ✓
══════════════════════════════════════════════════════════════════════
```

---

## Complete Flow: Vote on Survey

```
══════════════════════════════════════════════════════════════════════
FLOW 3: VOTE ON SURVEY
══════════════════════════════════════════════════════════════════════

[UI] SurveyResultPage (user viewing survey with id "123")
  │
  │ User taps answer option "Flutter"
  │
  └→ [UI] SurveyAnswer.onTap()
      │
      └→ [P] GetxSurveyResultPresenter.save(answer: "Flutter")
          │
          └─ Call: showResultOnAction(() => saveSurveyResult.save(answer: "Flutter"))
              │
              ├─ Set: isLoading = true
              │   └→ [UI] Shows spinner ⏳
              │
              ├─ Execute action:
              │   │
              │   └→ [P] saveSurveyResult.save(answer: "Flutter")
              │       │
              │       └→ [DATA] RemoteSaveSurveyResult.save(answer: "Flutter")
              │           │
              │           ├─ Build request body:
              │           │   └→ {"answer": "Flutter"}
              │           │
              │           ├─ Call: httpClient.request(
              │           │     url: '/api/surveys/123/results',
              │           │     method: 'put',
              │           │     body: {"answer": "Flutter"}
              │           │   )
              │           │   │
              │           │   └→ [MAIN] AuthorizeHttpClientDecorator
              │           │       │
              │           │       ├─ Fetch token, add to headers
              │           │       │
              │           │       └─ [INFRA] HttpAdapter
              │           │           │
              │           │           ├─ HTTP PUT to API
              │           │           │   │
              │           │           │   PUT /api/surveys/123/results
              │           │           │   Headers: {'x-access-token': "..."}
              │           │           │   Body: '{"answer":"Flutter"}'
              │           │           │   │
              │           │           │   └→ [EXT] API processes vote
              │           │           │       └→ Response: 200 OK
              │           │           │           Body: '{
              │           │           │             "surveyId": "123",
              │           │           │             "question": "Qual framework?",
              │           │           │             "answers": [
              │           │           │               {"answer":"Flutter","count":42,"percent":60,"isCurrentAccountAnswer":true},
              │           │           │               {"answer":"React Native","count":28,"percent":40,"isCurrentAccountAnswer":false}
              │           │           │             ],
              │           │           │             "date": "..."
              │           │           │           }'
              │           │           │
              │           │           └─ Parse JSON
              │           │               └→ Return: Map<String, dynamic>
              │           │
              │           ├─ Transform: JSON → Model → Entity
              │           │   │
              │           │   ├─ RemoteSurveyResultModel.fromJson(response)
              │           │   │
              │           │   └─ model.toEntity()
              │           │       └→ [D] SurveyResultEntity(
              │           │           surveyId: "123",
              │           │           question: "Qual framework?",
              │           │           answers: [
              │           │             SurveyAnswerEntity(answer:"Flutter", count:42, percent:60, isCurrentAccountAnswer:true),
              │           │             SurveyAnswerEntity(answer:"React Native", count:28, percent:40, isCurrentAccountAnswer:false)
              │           │           ],
              │           │           date: DateTime(...)
              │           │         )
              │           │
              │           └→ Return: SurveyResultEntity ✓
              │
              ├─ Transform to ViewModel:
              │   │
              │   └─ surveyResult.toViewModel()
              │       │
              │       └→ [P] Extension method (SurveyResultEntityExtensions)
              │           │
              │           └→ SurveyResultViewModel(
              │               surveyId: "123",
              │               question: "Qual framework?",
              │               answers: [
              │                 SurveyAnswerViewModel(
              │                   image: "flutter_logo.png",
              │                   answer: "Flutter",
              │                   percent: "60%",
              │                   isCurrentAnswer: true
              │                 ),
              │                 SurveyAnswerViewModel(
              │                   image: "react_logo.png",
              │                   answer: "React Native",
              │                   percent: "40%",
              │                   isCurrentAnswer: false
              │                 )
              │               ],
              │               date: "15 Jan 2024"
              │             )
              │
              ├─ Set: _surveyResult.subject.add(viewModel)
              │   └→ Stream emits: SurveyResultViewModel
              │       └→ [UI] StreamBuilder rebuilds
              │           └─ Shows updated percentages:
              │               ✓ Flutter: 60% (highlighted as user's answer)
              │               • React Native: 40%
              │
              └─ Set: isLoading = false (finally)
                  └→ [UI] Hides spinner ✓

══════════════════════════════════════════════════════════════════════
END: Vote registered, results updated in real-time ✓
══════════════════════════════════════════════════════════════════════
```

---

## Complete Flow: Session Expiration

```
══════════════════════════════════════════════════════════════════════
FLOW 4: SESSION EXPIRATION (TOKEN EXPIRED)
══════════════════════════════════════════════════════════════════════

[UI] SurveysPage (user's token expired on server)
  │
  └→ [P] GetxSurveysPresenter.loadData()
      │
      ├─ Set: isLoading = true
      │
      ├─ Call: loadSurveys.load()
      │   │
      │   └→ [MAIN] RemoteLoadSurveysWithLocalFallback.load()
      │       │
      │       ├─ TRY: remote.load()
      │       │   │
      │       │   └→ [DATA] RemoteLoadSurveys.load()
      │       │       │
      │       │       └─ Call: httpClient.request(...)
      │       │           │
      │       │           └→ [MAIN] AuthorizeHttpClientDecorator.request(...)
      │       │               │
      │       │               ├─ Fetch token
      │       │               │   └→ Return: "eyJhbGci..." (expired token)
      │       │               │
              │               ├─ Add to headers: {'x-access-token': "eyJhbGci..."}
              │               │
              │               ├─ Call: decoratee.request(...)
              │               │   │
              │               │   └→ [INFRA] HttpAdapter.request(...)
              │               │       │
              │               │       ├─ HTTP GET to API
              │               │       │   └→ [EXT] API validates token
              │               │       │       └→ Token expired! ✗
              │               │       │           └→ Response: 403 Forbidden
              │               │       │
              │               │       └─ Status: 403
              │               │           └→ throw HttpError.forbidden ✗
              │               │
              │               ├─ CATCH error:
              │               │   │
              │               │   ├─ error == HttpError.forbidden? ✓
              │               │   │
              │               │   ├─ Delete invalid token:
              │               │   │   └→ deleteSecureCacheStorage.delete('token')
              │               │   │       └→ [INFRA] SecureStorageAdapter.delete('token')
              │               │   │           └→ [EXT] FlutterSecureStorage.delete(key: 'token')
              │               │   │               └─ Token deleted ✓
              │               │   │
              │               │   └→ throw HttpError.forbidden ✗
              │               │
              │               └→ (propagates up)
              │
              ├─ RemoteLoadSurveys catches HttpError.forbidden
              │   └→ throw DomainError.accessDenied ✗
              │
              └─ RemoteLoadSurveysWithLocalFallback catches DomainError
                  │
                  ├─ error == DomainError.accessDenied? ✓
                  │
                  └→ rethrow DomainError.accessDenied ✗
                      (Don't use cache for session expiration!)
      │
      └─ CATCH DomainError in presenter:
          │
          ├─ error == DomainError.accessDenied? ✓
          │
          ├─ Set: isSessionExpired = true
          │   └→ Stream emits: true
          │       └→ [UI] SessionManager.handleSessionExpired()
          │           │
          │           ├─ Navigate: Get.offAllNamed('/login')
          │           │   └─ Clear stack, go to LoginPage ✓
          │           │
          │           └─ Show SnackBar:
          │               "Sua sessão expirou. Por favor, faça login novamente." ✗
          │
          └─ Set: isLoading = false (finally)

══════════════════════════════════════════════════════════════════════
END: User redirected to login, token cleared, message shown ✓
══════════════════════════════════════════════════════════════════════
```

---

## Data Transformation Journey

### JSON → Entity → ViewModel

```
══════════════════════════════════════════════════════════════════════
TRANSFORMATION: API RESPONSE TO UI DISPLAY
══════════════════════════════════════════════════════════════════════

STEP 1: API Response (External)
────────────────────────────────
{
  "id": "1",
  "question": "Qual é seu framework favorito?",
  "date": "2024-01-15T10:00:00.000Z",
  "didAnswer": false
}

                    ↓ JSON parsing

STEP 2: Data Model (Data Layer)
────────────────────────────────
RemoteSurveyModel(
  id: "1",
  question: "Qual é seu framework favorito?",
  date: "2024-01-15T10:00:00.000Z",  ← Still a string
  didAnswer: false
)

                    ↓ model.toEntity()

STEP 3: Domain Entity (Domain Layer)
─────────────────────────────────────
SurveyEntity(
  id: "1",
  question: "Qual é seu framework favorito?",
  dateTime: DateTime(2024, 1, 15, 10, 0, 0),  ← Now a DateTime object
  didAnswer: false
)

                    ↓ Presenter transformation

STEP 4: ViewModel (Presentation/UI Layer)
──────────────────────────────────────────
SurveyViewModel(
  id: "1",
  question: "Qual é seu framework favorito?",
  date: "15 Jan 2024",  ← Formatted for display
  didAnswer: false
)

                    ↓ UI rendering

STEP 5: Displayed to User
─────────────────────────
┌─────────────────────────────────────┐
│ Qual é seu framework favorito?      │
│ 15 Jan 2024                         │
│ [ ] Not answered                    │
└─────────────────────────────────────┘

══════════════════════════════════════════════════════════════════════
WHY SO MANY TRANSFORMATIONS?
══════════════════════════════════════════════════════════════════════

1. JSON (API format) → Model
   • Validates data structure
   • Handles missing/invalid fields
   • Adapts to API format

2. Model → Entity (Domain format)
   • Pure business object
   • Type-safe (DateTime, not String)
   • Framework-independent

3. Entity → ViewModel (Display format)
   • Formatted strings ("15 Jan 2024")
   • Display logic (icons, colors)
   • UI-optimized

BENEFITS:
✓ Each layer works with appropriate types
✓ Easy to change API format without touching domain
✓ Easy to change UI format without touching business logic
✓ Testable at each stage

══════════════════════════════════════════════════════════════════════
```

---

## State Update Propagation

### How State Changes Flow to UI

```
══════════════════════════════════════════════════════════════════════
STATE PROPAGATION: VALUE CHANGE → UI UPDATE
══════════════════════════════════════════════════════════════════════

PRESENTER (Presentation Layer)
──────────────────────────────
final _emailError = Rx<UIError?>(null);  ← Reactive variable

Stream<UIError?> get emailErrorStream => _emailError.stream;  ← Exposed stream

void validateEmail(String email) {
  _email = email;
  _emailError.value = _validateField('email');  ← UPDATE HAPPENS HERE
}

                    ↓ Set value triggers

REACTIVE SYSTEM (GetX)
──────────────────────
_emailError.value = UIError.invalidField
  │
  ├─ Notify all stream listeners
  │
  └→ Stream emits: UIError.invalidField

                    ↓ Stream emission

UI LAYER (StreamBuilder)
────────────────────────
StreamBuilder<UIError?>(
  stream: presenter.emailErrorStream,  ← Listening to stream
  builder: (context, snapshot) {
    return TextFormField(
      errorText: snapshot.data?.description,  ← snapshot.data = UIError.invalidField
      // ...
    );
  },
)

                    ↓ Builder called with new data

FLUTTER REBUILD
───────────────
StreamBuilder receives new snapshot:
  • snapshot.hasData = true
  • snapshot.data = UIError.invalidField

Calls builder function with new snapshot

Builder returns new TextFormField with errorText

Flutter rebuilds widget ✓

                    ↓ Visual update

USER SEES
─────────
┌─────────────────────────┐
│ Email                   │
│ test                    │
│ Campo inválido          │ ← Error appears!
└─────────────────────────┘

══════════════════════════════════════════════════════════════════════
TIMING
══════════════════════════════════════════════════════════════════════

User types "t"
  ↓ ~0ms
validateEmail("t") called
  ↓ ~1ms
_emailError.value = UIError.invalidField
  ↓ ~1ms
Stream emits
  ↓ ~1ms
StreamBuilder receives
  ↓ ~1-2ms
Flutter rebuilds
  ↓
User sees error (~5-10ms total)

REAL-TIME! ✓

══════════════════════════════════════════════════════════════════════
```

---

## Error Flow Patterns

### Three-Layer Error Mapping

```
══════════════════════════════════════════════════════════════════════
ERROR TRANSFORMATION: INFRASTRUCTURE → UI
══════════════════════════════════════════════════════════════════════

LAYER 1: INFRASTRUCTURE ERRORS
───────────────────────────────
[INFRA] HttpAdapter encounters:

HTTP Status 400 → throw HttpError.badRequest
HTTP Status 401 → throw HttpError.unauthorized
HTTP Status 403 → throw HttpError.forbidden
HTTP Status 404 → throw HttpError.notFound
HTTP Status 500+ → throw HttpError.serverError
Network timeout → throw HttpError.serverError
Exception → throw HttpError.serverError

                    ↓ Caught by Data layer

LAYER 2: DOMAIN ERRORS
──────────────────────
[DATA] RemoteAuthentication catches and maps:

HttpError.unauthorized → throw DomainError.invalidCredentials
HttpError.forbidden → throw DomainError.accessDenied
HttpError.* → throw DomainError.unexpected

                    ↓ Caught by Presentation layer

LAYER 3: UI ERRORS
──────────────────
[P] GetxLoginPresenter catches and maps:

DomainError.invalidCredentials → mainError = UIError.invalidCredentials
DomainError.accessDenied → isSessionExpired = true
DomainError.emailInUse → mainError = UIError.emailInUse
DomainError.* → mainError = UIError.unexpected

                    ↓ Displayed to user

LAYER 4: USER MESSAGES
──────────────────────
[UI] UIError.description (i18n):

UIError.invalidCredentials → "Credenciais inválidas"
UIError.emailInUse → "Esse e-mail já está em uso"
UIError.unexpected → "Algo errado aconteceu. Tente novamente em breve."

══════════════════════════════════════════════════════════════════════
EXAMPLE: 401 ERROR FLOW
══════════════════════════════════════════════════════════════════════

API returns 401
  ↓
[INFRA] HttpAdapter
  throw HttpError.unauthorized
    ↓
[DATA] RemoteAuthentication
  catch HttpError.unauthorized
  throw DomainError.invalidCredentials
    ↓
[P] GetxLoginPresenter
  catch DomainError.invalidCredentials
  mainError = UIError.invalidCredentials
    ↓
[UI] UIErrorManager
  Shows SnackBar: "Credenciais inválidas"

══════════════════════════════════════════════════════════════════════
WHY THREE LAYERS?
══════════════════════════════════════════════════════════════════════

✓ Each layer only knows its own error types
✓ Domain doesn't depend on HTTP details
✓ UI doesn't depend on backend details
✓ Easy to change error messages without touching logic
✓ Easy to swap implementations (REST → GraphQL)

══════════════════════════════════════════════════════════════════════
```

---

## Key Takeaways

### Flow Patterns Summary

**1. Standard Request Flow:**
```
UI → Presenter → Use Case → HTTP Client → API
    ← ← ← ← ← ← ← ←
   Response flows back through layers
```

**2. State Update Flow:**
```
Presenter updates Rx variable
  → Stream emits
  → StreamBuilder receives
  → Flutter rebuilds
  → User sees update
```

**3. Error Flow:**
```
External error
  → Infra maps to HttpError
  → Data maps to DomainError
  → Presentation maps to UIError
  → UI displays message
```

**4. Cache Flow:**
```
Try remote
  → Success: Save to cache, return
  → Failure: Load from cache, return
```

---

### Debugging Tips

**Tracing a Bug:**

1. **Identify the symptom** (what's wrong?)
2. **Find the layer** (UI error? Data not loading? Validation issue?)
3. **Trace backwards:**
   - UI issue → Check presenter stream
   - Stream issue → Check presenter logic
   - Logic issue → Check use case
   - Use case issue → Check implementation
   - Implementation issue → Check adapter/external
4. **Add breakpoints** at layer boundaries
5. **Check transformations** (JSON → Model → Entity → ViewModel)

---

### Adding New Features

**Follow the Flow:**

1. **Define domain use case** (interface)
2. **Implement in data layer** (with model transformations)
3. **Create presenter** (coordinate use case)
4. **Build UI** (listen to streams)
5. **Wire in main** (factories)
6. **Test each layer** independently

---

## Next Steps

You now understand complete data flows! Next:

1. **[Getting Started Guide](06-getting-started-guide.md)** - Build your own feature following these flows
2. **[Testing Strategy](07-testing-strategy.md)** - Test these flows at each layer

---

**Documentation Version:** 1.0
**Last Updated:** 2025

---

**You can now trace any request through the entire application!** 🎯
