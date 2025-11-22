# Anti-Corruption Layer em Dart - Guia Completo

## Índice

1. [Introdução](#introdução)
2. [Conceitos Fundamentais](#conceitos-fundamentais)
3. [Estrutura do Projeto](#estrutura-do-projeto)
4. [Implementação Completa](#implementação-completa)
5. [Princípios Arquiteturais](#princípios-arquiteturais)
6. [Testes](#testes)
7. [Benefícios e Trade-offs](#benefícios-e-trade-offs)
8. [Boas Práticas](#boas-práticas)

---

## Introdução

O **Anti-Corruption Layer (ACL)** é um padrão arquitetural do Domain-Driven Design (DDD) que protege o modelo de domínio de influências externas indesejadas. Ele atua como uma barreira de tradução entre seu domínio puro e sistemas externos (APIs, bancos de dados, serviços third-party).

### Quando Usar?

- ✅ Aplicações que você espera manter e evoluir
- ✅ Integração com APIs de terceiros
- ✅ Sistemas que precisam isolar mudanças externas
- ✅ Projetos que seguem Clean Architecture ou DDD
- ❌ Protótipos de vida curta
- ❌ MVPs com prazo muito apertado (mas considere para v2)

### Por que Mapear Mesmo Quando os Objetos São Idênticos?

Mesmo que o DTO da API seja idêntico à entidade de domínio **hoje**, há razões importantes para manter o mapeamento:

#### Desvantagens de NÃO fazer o mapeamento:

1. **Acoplamento Forte**: Seu domínio fica "refém" de decisões externas
2. **Quebra de Contrato Inesperada**: Mudanças na API quebram múltiplos pontos do sistema
3. **Contaminação do Domínio**: Anotações JSON, validações de API no modelo de negócio
4. **Dificuldade de Evolução**: Refatoração massiva quando precisar divergir
5. **Testes Mais Difíceis**: Lógica de negócio acoplada a estruturas externas

---

## Conceitos Fundamentais

### 1. Separação de Responsabilidades

```
┌─────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                    │
│                   (UI/Controllers)                       │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                     DOMAIN LAYER                         │
│            (Entities, Value Objects, UseCases)          │
│           → Linguagem Ubíqua do Negócio                 │
│           → Sem Dependências Externas                   │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                      DATA LAYER                          │
│     ┌──────────────────────────────────────────┐       │
│     │   ANTI-CORRUPTION LAYER (Mappers)       │       │
│     │   → Traduz DTO ↔ Domain                 │       │
│     │   → Isola Mudanças Externas             │       │
│     └──────────────────────────────────────────┘       │
│                                                          │
│   DTOs (Estrutura da API) ← → Entities (Negócio)      │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                   EXTERNAL SYSTEMS                       │
│              (REST APIs, Databases, etc)                │
└─────────────────────────────────────────────────────────┘
```

### 2. DTO vs Entity

| Aspecto | DTO (Data Transfer Object) | Entity (Domain) |
|---------|---------------------------|-----------------|
| **Propósito** | Transporte de dados | Lógica de negócio |
| **Nomenclatura** | Segue convenção da API (snake_case) | Segue convenção Dart (camelCase) |
| **Validação** | Estrutural (JSON válido) | Regras de negócio |
| **Mutabilidade** | Geralmente imutável | Pode ter comportamentos |
| **Dependências** | Anotações de serialização | Zero dependências externas |
| **Estabilidade** | Muda com a API | Muda com o negócio |

---

## Estrutura do Projeto

```
lib/
├── core/
│   ├── errors/
│   │   ├── failures.dart
│   │   └── exceptions.dart
│   └── types/
│       └── either.dart
│
├── features/
│   └── user/
│       ├── data/
│       │   ├── datasources/
│       │   │   ├── user_remote_datasource.dart
│       │   │   └── user_remote_datasource_impl.dart
│       │   ├── dtos/
│       │   │   └── user_dto.dart
│       │   ├── mappers/
│       │   │   └── user_mapper.dart          ← ACL
│       │   └── repositories/
│       │       └── user_repository_impl.dart
│       │
│       ├── domain/
│       │   ├── entities/
│       │   │   └── user.dart
│       │   ├── value_objects/
│       │   │   ├── email.dart
│       │   │   └── user_id.dart
│       │   ├── repositories/
│       │   │   └── user_repository.dart
│       │   └── usecases/
│       │       ├── get_user.dart
│       │       └── create_user.dart
│       │
│       └── presentation/
│           └── ...
```

---

## Implementação Completa

### 1. Core - Tratamento de Erros

#### failures.dart

```dart
// lib/core/errors/failures.dart
abstract class Failure {
  final String message;
  final StackTrace? stackTrace;

  const Failure(this.message, [this.stackTrace]);
}

class ServerFailure extends Failure {
  const ServerFailure(super.message, [super.stackTrace]);
}

class NetworkFailure extends Failure {
  const NetworkFailure(super.message, [super.stackTrace]);
}

class ValidationFailure extends Failure {
  const ValidationFailure(super.message, [super.stackTrace]);
}

class MappingFailure extends Failure {
  const MappingFailure(super.message, [super.stackTrace]);
}
```

#### exceptions.dart

```dart
// lib/core/errors/exceptions.dart
class ServerException implements Exception {
  final String message;
  final int? statusCode;

  ServerException(this.message, [this.statusCode]);
}

class NetworkException implements Exception {
  final String message;

  NetworkException(this.message);
}

class MappingException implements Exception {
  final String message;
  final dynamic originalData;

  MappingException(this.message, [this.originalData]);
}
```

#### either.dart

```dart
// lib/core/types/either.dart
abstract class Either<L, R> {
  const Either();

  bool get isLeft;
  bool get isRight;

  L get left;
  R get right;

  T fold<T>(T Function(L left) ifLeft, T Function(R right) ifRight);
}

class Left<L, R> extends Either<L, R> {
  final L _value;

  const Left(this._value);

  @override
  bool get isLeft => true;

  @override
  bool get isRight => false;

  @override
  L get left => _value;

  @override
  R get right => throw Exception('Cannot get right value from Left');

  @override
  T fold<T>(T Function(L left) ifLeft, T Function(R right) ifRight) {
    return ifLeft(_value);
  }
}

class Right<L, R> extends Either<L, R> {
  final R _value;

  const Right(this._value);

  @override
  bool get isLeft => false;

  @override
  bool get isRight => true;

  @override
  L get left => throw Exception('Cannot get left value from Right');

  @override
  R get right => _value;

  @override
  T fold<T>(T Function(L left) ifLeft, T Function(R right) ifRight) {
    return ifRight(_value);
  }
}
```

---

### 2. Domain Layer - Modelos Puros de Negócio

#### Value Objects

```dart
// lib/features/user/domain/value_objects/user_id.dart
class UserId {
  final String value;

  UserId._(this.value);

  factory UserId(String value) {
    if (value.isEmpty) {
      throw ArgumentError('UserId cannot be empty');
    }
    return UserId._(value);
  }

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is UserId && 
      runtimeType == other.runtimeType && 
      value == other.value;

  @override
  int get hashCode => value.hashCode;

  @override
  String toString() => value;
}
```

```dart
// lib/features/user/domain/value_objects/email.dart
class Email {
  final String value;

  Email._(this.value);

  factory Email(String value) {
    if (!_isValid(value)) {
      throw ArgumentError('Invalid email format: $value');
    }
    return Email._(value.toLowerCase());
  }

  static bool _isValid(String email) {
    final emailRegex = RegExp(
      r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$',
    );
    return emailRegex.hasMatch(email);
  }

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is Email && 
      runtimeType == other.runtimeType && 
      value == other.value;

  @override
  int get hashCode => value.hashCode;

  @override
  String toString() => value;
}
```

#### Entity

```dart
// lib/features/user/domain/entities/user.dart
enum UserStatus {
  active,
  inactive,
  suspended,
  pending;

  bool get isActive => this == UserStatus.active;
  bool get canLogin => this == UserStatus.active || this == UserStatus.pending;
}

class User {
  final UserId id;
  final String name;
  final Email email;
  final UserStatus status;
  final DateTime createdAt;
  final DateTime? lastLoginAt;

  User({
    required this.id,
    required this.name,
    required this.email,
    required this.status,
    required this.createdAt,
    this.lastLoginAt,
  });

  // Business logic methods
  bool canPerformAction() => status.canLogin;

  User copyWith({
    UserId? id,
    String? name,
    Email? email,
    UserStatus? status,
    DateTime? createdAt,
    DateTime? lastLoginAt,
  }) {
    return User(
      id: id ?? this.id,
      name: name ?? this.name,
      email: email ?? this.email,
      status: status ?? this.status,
      createdAt: createdAt ?? this.createdAt,
      lastLoginAt: lastLoginAt ?? this.lastLoginAt,
    );
  }

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is User &&
          runtimeType == other.runtimeType &&
          id == other.id;

  @override
  int get hashCode => id.hashCode;

  @override
  String toString() => 
      'User(id: $id, name: $name, email: $email, status: $status)';
}
```

#### Repository Interface

```dart
// lib/features/user/domain/repositories/user_repository.dart
abstract class UserRepository {
  Future<Either<Failure, User>> getUserById(UserId id);
  Future<Either<Failure, List<User>>> getAllUsers();
  Future<Either<Failure, User>> createUser({
    required String name,
    required String email,
  });
  Future<Either<Failure, User>> updateUser(User user);
  Future<Either<Failure, void>> deleteUser(UserId id);
}
```

---

### 3. Data Layer - DTOs e Anti-Corruption Layer

#### DTOs

```dart
// lib/features/user/data/dtos/user_dto.dart
import 'package:json_annotation/json_annotation.dart';

part 'user_dto.g.dart';

/// DTO que representa EXATAMENTE o contrato da API externa
/// Usa snake_case, pode ter campos técnicos, etc.
@JsonSerializable()
class UserDto {
  @JsonKey(name: 'user_id')
  final String userId;

  @JsonKey(name: 'full_name')
  final String fullName;

  @JsonKey(name: 'email_address')
  final String emailAddress;

  @JsonKey(name: 'account_status')
  final int accountStatus; // 0=pending, 1=active, 2=inactive, 3=suspended

  @JsonKey(name: 'created_timestamp')
  final String createdTimestamp; // ISO 8601 string

  @JsonKey(name: 'last_login_timestamp')
  final String? lastLoginTimestamp;

  // Campos técnicos da API que não interessam ao domínio
  @JsonKey(name: '_internal_version')
  final int? internalVersion;

  @JsonKey(name: '_sync_token')
  final String? syncToken;

  const UserDto({
    required this.userId,
    required this.fullName,
    required this.emailAddress,
    required this.accountStatus,
    required this.createdTimestamp,
    this.lastLoginTimestamp,
    this.internalVersion,
    this.syncToken,
  });

  factory UserDto.fromJson(Map<String, dynamic> json) =>
      _$UserDtoFromJson(json);

  Map<String, dynamic> toJson() => _$UserDtoToJson(this);
}

/// DTO para criação de usuário (pode ter estrutura diferente)
@JsonSerializable()
class CreateUserDto {
  @JsonKey(name: 'full_name')
  final String fullName;

  @JsonKey(name: 'email_address')
  final String emailAddress;

  const CreateUserDto({
    required this.fullName,
    required this.emailAddress,
  });

  factory CreateUserDto.fromJson(Map<String, dynamic> json) =>
      _$CreateUserDtoFromJson(json);

  Map<String, dynamic> toJson() => _$CreateUserDtoToJson(this);
}
```

#### Mapper - O Coração do ACL

```dart
// lib/features/user/data/mappers/user_mapper.dart
import '../../domain/entities/user.dart';
import '../../domain/value_objects/email.dart';
import '../../domain/value_objects/user_id.dart';
import '../../../core/errors/exceptions.dart';
import '../dtos/user_dto.dart';

/// Mapper que implementa o Anti-Corruption Layer
/// Converte entre o mundo externo (API) e o mundo interno (Domínio)
abstract class UserMapper {
  /// Converte DTO da API para Entidade de Domínio
  static User toDomain(UserDto dto) {
    try {
      return User(
        id: UserId(dto.userId),
        name: dto.fullName,
        email: Email(dto.emailAddress),
        status: _mapStatus(dto.accountStatus),
        createdAt: DateTime.parse(dto.createdTimestamp),
        lastLoginAt: dto.lastLoginTimestamp != null
            ? DateTime.parse(dto.lastLoginTimestamp!)
            : null,
      );
    } catch (e, stackTrace) {
      throw MappingException(
        'Failed to map UserDto to User: ${e.toString()}',
        dto,
      );
    }
  }

  /// Converte Entidade de Domínio para DTO da API
  static UserDto toDto(User user) {
    try {
      return UserDto(
        userId: user.id.value,
        fullName: user.name,
        emailAddress: user.email.value,
        accountStatus: _mapStatusToInt(user.status),
        createdTimestamp: user.createdAt.toIso8601String(),
        lastLoginTimestamp: user.lastLoginAt?.toIso8601String(),
      );
    } catch (e, stackTrace) {
      throw MappingException(
        'Failed to map User to UserDto: ${e.toString()}',
        user,
      );
    }
  }

  /// Converte dados de criação para DTO
  static CreateUserDto toCreateDto({
    required String name,
    required String email,
  }) {
    // Valida antes de enviar para API
    Email(email); // Valida formato do email
    
    return CreateUserDto(
      fullName: name,
      emailAddress: email,
    );
  }

  /// Converte lista de DTOs para lista de Entidades
  static List<User> toDomainList(List<UserDto> dtos) {
    return dtos.map((dto) => toDomain(dto)).toList();
  }

  // Métodos privados para mapeamento de enums e valores específicos
  static UserStatus _mapStatus(int status) {
    switch (status) {
      case 0:
        return UserStatus.pending;
      case 1:
        return UserStatus.active;
      case 2:
        return UserStatus.inactive;
      case 3:
        return UserStatus.suspended;
      default:
        throw MappingException(
          'Unknown account status: $status',
          status,
        );
    }
  }

  static int _mapStatusToInt(UserStatus status) {
    switch (status) {
      case UserStatus.pending:
        return 0;
      case UserStatus.active:
        return 1;
      case UserStatus.inactive:
        return 2;
      case UserStatus.suspended:
        return 3;
    }
  }
}
```

---

### 4. Data Source - Comunicação com API

#### Interface

```dart
// lib/features/user/data/datasources/user_remote_datasource.dart
import '../dtos/user_dto.dart';

abstract class UserRemoteDataSource {
  Future<UserDto> getUserById(String id);
  Future<List<UserDto>> getAllUsers();
  Future<UserDto> createUser(CreateUserDto dto);
  Future<UserDto> updateUser(String id, UserDto dto);
  Future<void> deleteUser(String id);
}
```

#### Implementação

```dart
// lib/features/user/data/datasources/user_remote_datasource_impl.dart
import 'dart:convert';
import 'package:http/http.dart' as http;
import '../../../core/errors/exceptions.dart';
import '../dtos/user_dto.dart';

class UserRemoteDataSourceImpl implements UserRemoteDataSource {
  final http.Client client;
  final String baseUrl;

  UserRemoteDataSourceImpl({
    required this.client,
    required this.baseUrl,
  });

  @override
  Future<UserDto> getUserById(String id) async {
    try {
      final response = await client.get(
        Uri.parse('$baseUrl/users/$id'),
        headers: {'Content-Type': 'application/json'},
      );

      if (response.statusCode == 200) {
        final json = jsonDecode(response.body) as Map<String, dynamic>;
        return UserDto.fromJson(json);
      } else if (response.statusCode == 404) {
        throw ServerException('User not found', response.statusCode);
      } else {
        throw ServerException(
          'Failed to get user: ${response.body}',
          response.statusCode,
        );
      }
    } catch (e) {
      if (e is ServerException) rethrow;
      throw NetworkException('Network error: ${e.toString()}');
    }
  }

  @override
  Future<List<UserDto>> getAllUsers() async {
    try {
      final response = await client.get(
        Uri.parse('$baseUrl/users'),
        headers: {'Content-Type': 'application/json'},
      );

      if (response.statusCode == 200) {
        final jsonList = jsonDecode(response.body) as List;
        return jsonList
            .map((json) => UserDto.fromJson(json as Map<String, dynamic>))
            .toList();
      } else {
        throw ServerException(
          'Failed to get users: ${response.body}',
          response.statusCode,
        );
      }
    } catch (e) {
      if (e is ServerException) rethrow;
      throw NetworkException('Network error: ${e.toString()}');
    }
  }

  @override
  Future<UserDto> createUser(CreateUserDto dto) async {
    try {
      final response = await client.post(
        Uri.parse('$baseUrl/users'),
        headers: {'Content-Type': 'application/json'},
        body: jsonEncode(dto.toJson()),
      );

      if (response.statusCode == 201) {
        final json = jsonDecode(response.body) as Map<String, dynamic>;
        return UserDto.fromJson(json);
      } else {
        throw ServerException(
          'Failed to create user: ${response.body}',
          response.statusCode,
        );
      }
    } catch (e) {
      if (e is ServerException) rethrow;
      throw NetworkException('Network error: ${e.toString()}');
    }
  }

  @override
  Future<UserDto> updateUser(String id, UserDto dto) async {
    try {
      final response = await client.put(
        Uri.parse('$baseUrl/users/$id'),
        headers: {'Content-Type': 'application/json'},
        body: jsonEncode(dto.toJson()),
      );

      if (response.statusCode == 200) {
        final json = jsonDecode(response.body) as Map<String, dynamic>;
        return UserDto.fromJson(json);
      } else {
        throw ServerException(
          'Failed to update user: ${response.body}',
          response.statusCode,
        );
      }
    } catch (e) {
      if (e is ServerException) rethrow;
      throw NetworkException('Network error: ${e.toString()}');
    }
  }

  @override
  Future<void> deleteUser(String id) async {
    try {
      final response = await client.delete(
        Uri.parse('$baseUrl/users/$id'),
        headers: {'Content-Type': 'application/json'},
      );

      if (response.statusCode != 204 && response.statusCode != 200) {
        throw ServerException(
          'Failed to delete user: ${response.body}',
          response.statusCode,
        );
      }
    } catch (e) {
      if (e is ServerException) rethrow;
      throw NetworkException('Network error: ${e.toString()}');
    }
  }
}
```

---

### 5. Repository Implementation - Orquestra o ACL

```dart
// lib/features/user/data/repositories/user_repository_impl.dart
import '../../domain/entities/user.dart';
import '../../domain/repositories/user_repository.dart';
import '../../domain/value_objects/user_id.dart';
import '../../../core/errors/exceptions.dart';
import '../../../core/errors/failures.dart';
import '../../../core/types/either.dart';
import '../datasources/user_remote_datasource.dart';
import '../mappers/user_mapper.dart';

class UserRepositoryImpl implements UserRepository {
  final UserRemoteDataSource remoteDataSource;

  UserRepositoryImpl({required this.remoteDataSource});

  @override
  Future<Either<Failure, User>> getUserById(UserId id) async {
    try {
      // 1. Busca dados externos (DTO)
      final userDto = await remoteDataSource.getUserById(id.value);
      
      // 2. Anti-Corruption Layer: converte DTO -> Domain
      final user = UserMapper.toDomain(userDto);
      
      return Right(user);
    } on ServerException catch (e) {
      return Left(ServerFailure(e.message));
    } on NetworkException catch (e) {
      return Left(NetworkFailure(e.message));
    } on MappingException catch (e) {
      return Left(MappingFailure(e.message));
    } catch (e, stackTrace) {
      return Left(ServerFailure(
        'Unexpected error: ${e.toString()}', 
        stackTrace,
      ));
    }
  }

  @override
  Future<Either<Failure, List<User>>> getAllUsers() async {
    try {
      final userDtos = await remoteDataSource.getAllUsers();
      final users = UserMapper.toDomainList(userDtos);
      return Right(users);
    } on ServerException catch (e) {
      return Left(ServerFailure(e.message));
    } on NetworkException catch (e) {
      return Left(NetworkFailure(e.message));
    } on MappingException catch (e) {
      return Left(MappingFailure(e.message));
    } catch (e, stackTrace) {
      return Left(ServerFailure(
        'Unexpected error: ${e.toString()}', 
        stackTrace,
      ));
    }
  }

  @override
  Future<Either<Failure, User>> createUser({
    required String name,
    required String email,
  }) async {
    try {
      // 1. Valida e converte dados de entrada para DTO
      final createDto = UserMapper.toCreateDto(name: name, email: email);
      
      // 2. Envia para API
      final userDto = await remoteDataSource.createUser(createDto);
      
      // 3. Anti-Corruption Layer: converte resposta para Domain
      final user = UserMapper.toDomain(userDto);
      
      return Right(user);
    } on ArgumentError catch (e) {
      return Left(ValidationFailure(e.message));
    } on ServerException catch (e) {
      return Left(ServerFailure(e.message));
    } on NetworkException catch (e) {
      return Left(NetworkFailure(e.message));
    } on MappingException catch (e) {
      return Left(MappingFailure(e.message));
    } catch (e, stackTrace) {
      return Left(ServerFailure(
        'Unexpected error: ${e.toString()}', 
        stackTrace,
      ));
    }
  }

  @override
  Future<Either<Failure, User>> updateUser(User user) async {
    try {
      // 1. Anti-Corruption Layer: converte Domain -> DTO
      final userDto = UserMapper.toDto(user);
      
      // 2. Envia para API
      final updatedDto = await remoteDataSource.updateUser(
        user.id.value,
        userDto,
      );
      
      // 3. Converte resposta de volta para Domain
      final updatedUser = UserMapper.toDomain(updatedDto);
      
      return Right(updatedUser);
    } on ServerException catch (e) {
      return Left(ServerFailure(e.message));
    } on NetworkException catch (e) {
      return Left(NetworkFailure(e.message));
    } on MappingException catch (e) {
      return Left(MappingFailure(e.message));
    } catch (e, stackTrace) {
      return Left(ServerFailure(
        'Unexpected error: ${e.toString()}', 
        stackTrace,
      ));
    }
  }

  @override
  Future<Either<Failure, void>> deleteUser(UserId id) async {
    try {
      await remoteDataSource.deleteUser(id.value);
      return const Right(null);
    } on ServerException catch (e) {
      return Left(ServerFailure(e.message));
    } on NetworkException catch (e) {
      return Left(NetworkFailure(e.message));
    } catch (e, stackTrace) {
      return Left(ServerFailure(
        'Unexpected error: ${e.toString()}', 
        stackTrace,
      ));
    }
  }
}
```

---

### 6. Use Cases

```dart
// lib/features/user/domain/usecases/get_user.dart
import '../entities/user.dart';
import '../repositories/user_repository.dart';
import '../value_objects/user_id.dart';
import '../../../core/errors/failures.dart';
import '../../../core/types/either.dart';

class GetUser {
  final UserRepository repository;

  GetUser(this.repository);

  Future<Either<Failure, User>> call(String userId) async {
    try {
      final id = UserId(userId);
      return await repository.getUserById(id);
    } on ArgumentError catch (e) {
      return Left(ValidationFailure(e.message));
    }
  }
}
```

```dart
// lib/features/user/domain/usecases/create_user.dart
import '../entities/user.dart';
import '../repositories/user_repository.dart';
import '../../../core/errors/failures.dart';
import '../../../core/types/either.dart';

class CreateUser {
  final UserRepository repository;

  CreateUser(this.repository);

  Future<Either<Failure, User>> call({
    required String name,
    required String email,
  }) async {
    if (name.isEmpty) {
      return const Left(ValidationFailure('Name cannot be empty'));
    }

    return await repository.createUser(name: name, email: email);
  }
}
```

---

## Princípios Arquiteturais

### 1. Separation of Concerns (SoC)

Cada camada tem responsabilidades distintas:
- **Domain**: Lógica de negócio pura
- **Data**: Acesso e persistência de dados
- **Mapper**: Tradução entre camadas

### 2. Dependency Inversion Principle (DIP)

```
Domain (alto nível) → não depende de → Data (baixo nível)
           ↓
    Ambos dependem de abstrações (interfaces)
```

### 3. Open/Closed Principle

O domínio está:
- ✅ Aberto para extensão (novas funcionalidades)
- ❌ Fechado para modificação (mudanças externas)

### 4. Single Responsibility Principle

- **Entity**: Representa conceitos de negócio
- **DTO**: Representa estrutura de transporte
- **Mapper**: Faz apenas a tradução

### 5. Domain-Driven Design (DDD)

O modelo de domínio deve:
- Refletir a linguagem ubíqua do negócio
- Ser independente de detalhes técnicos
- Conter apenas lógica de negócio

---

## Testes

### Testes do Mapper

```dart
// test/features/user/data/mappers/user_mapper_test.dart
import 'package:flutter_test/flutter_test.dart';

void main() {
  group('UserMapper', () {
    group('toDomain', () {
      test('should convert UserDto to User correctly', () {
        // Arrange
        final dto = UserDto(
          userId: '123',
          fullName: 'John Doe',
          emailAddress: 'john@example.com',
          accountStatus: 1,
          createdTimestamp: '2024-01-01T00:00:00Z',
          lastLoginTimestamp: '2024-01-15T10:30:00Z',
        );

        // Act
        final user = UserMapper.toDomain(dto);

        // Assert
        expect(user.id.value, '123');
        expect(user.name, 'John Doe');
        expect(user.email.value, 'john@example.com');
        expect(user.status, UserStatus.active);
        expect(user.createdAt, DateTime.parse('2024-01-01T00:00:00Z'));
        expect(user.lastLoginAt, DateTime.parse('2024-01-15T10:30:00Z'));
      });

      test('should throw MappingException when email is invalid', () {
        // Arrange
        final dto = UserDto(
          userId: '123',
          fullName: 'John Doe',
          emailAddress: 'invalid-email',
          accountStatus: 1,
          createdTimestamp: '2024-01-01T00:00:00Z',
        );

        // Act & Assert
        expect(
          () => UserMapper.toDomain(dto),
          throwsA(isA<MappingException>()),
        );
      });

      test('should throw MappingException for unknown status', () {
        // Arrange
        final dto = UserDto(
          userId: '123',
          fullName: 'John Doe',
          emailAddress: 'john@example.com',
          accountStatus: 999, // Status desconhecido
          createdTimestamp: '2024-01-01T00:00:00Z',
        );

        // Act & Assert
        expect(
          () => UserMapper.toDomain(dto),
          throwsA(isA<MappingException>()),
        );
      });

      test('should handle null lastLoginTimestamp', () {
        // Arrange
        final dto = UserDto(
          userId: '123',
          fullName: 'John Doe',
          emailAddress: 'john@example.com',
          accountStatus: 0,
          createdTimestamp: '2024-01-01T00:00:00Z',
          lastLoginTimestamp: null,
        );

        // Act
        final user = UserMapper.toDomain(dto);

        // Assert
        expect(user.lastLoginAt, isNull);
        expect(user.status, UserStatus.pending);
      });
    });

    group('toDto', () {
      test('should convert User to UserDto correctly', () {
        // Arrange
        final user = User(
          id: UserId('123'),
          name: 'John Doe',
          email: Email('john@example.com'),
          status: UserStatus.active,
          createdAt: DateTime.parse('2024-01-01T00:00:00Z'),
          lastLoginAt: DateTime.parse('2024-01-15T10:30:00Z'),
        );

        // Act
        final dto = UserMapper.toDto(user);

        // Assert
        expect(dto.userId, '123');
        expect(dto.fullName, 'John Doe');
        expect(dto.emailAddress, 'john@example.com');
        expect(dto.accountStatus, 1);
        expect(dto.createdTimestamp, '2024-01-01T00:00:00.000Z');
      });

      test('should handle all UserStatus values', () {
        final statuses = [
          (UserStatus.pending, 0),
          (UserStatus.active, 1),
          (UserStatus.inactive, 2),
          (UserStatus.suspended, 3),
        ];

        for (final (status, expectedInt) in statuses) {
          final user = User(
            id: UserId('123'),
            name: 'John Doe',
            email: Email('john@example.com'),
            status: status,
            createdAt: DateTime.parse('2024-01-01T00:00:00Z'),
          );

          final dto = UserMapper.toDto(user);

          expect(dto.accountStatus, expectedInt);
        }
      });
    });

    group('toDomainList', () {
      test('should convert list of DTOs to list of Users', () {
        // Arrange
        final dtos = [
          UserDto(
            userId: '1',
            fullName: 'User 1',
            emailAddress: 'user1@example.com',
            accountStatus: 1,
            createdTimestamp: '2024-01-01T00:00:00Z',
          ),
          UserDto(
            userId: '2',
            fullName: 'User 2',
            emailAddress: 'user2@example.com',
            accountStatus: 1,
            createdTimestamp: '2024-01-02T00:00:00Z',
          ),
        ];

        // Act
        final users = UserMapper.toDomainList(dtos);

        // Assert
        expect(users.length, 2);
        expect(users[0].id.value, '1');
        expect(users[1].id.value, '2');
      });

      test('should return empty list for empty input', () {
        // Arrange
        final dtos = <UserDto>[];

        // Act
        final users = UserMapper.toDomainList(dtos);

        // Assert
        expect(users, isEmpty);
      });
    });

    group('toCreateDto', () {
      test('should create CreateUserDto with valid data', () {
        // Act
        final dto = UserMapper.toCreateDto(
          name: 'John Doe',
          email: 'john@example.com',
        );

        // Assert
        expect(dto.fullName, 'John Doe');
        expect(dto.emailAddress, 'john@example.com');
      });

      test('should throw ArgumentError for invalid email', () {
        // Act & Assert
        expect(
          () => UserMapper.toCreateDto(
            name: 'John Doe',
            email: 'invalid-email',
          ),
          throwsArgumentError,
        );
      });
    });
  });
}
```

### Testes do Repository

```dart
// test/features/user/data/repositories/user_repository_impl_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

class MockUserRemoteDataSource extends Mock 
    implements UserRemoteDataSource {}

void main() {
  late UserRepositoryImpl repository;
  late MockUserRemoteDataSource mockRemoteDataSource;

  setUp(() {
    mockRemoteDataSource = MockUserRemoteDataSource();
    repository = UserRepositoryImpl(
      remoteDataSource: mockRemoteDataSource,
    );
  });

  group('getUserById', () {
    final tUserId = UserId('123');
    final tUserDto = UserDto(
      userId: '123',
      fullName: 'John Doe',
      emailAddress: 'john@example.com',
      accountStatus: 1,
      createdTimestamp: '2024-01-01T00:00:00Z',
    );

    test('should return User when the call is successful', () async {
      // Arrange
      when(() => mockRemoteDataSource.getUserById(any()))
          .thenAnswer((_) async => tUserDto);

      // Act
      final result = await repository.getUserById(tUserId);

      // Assert
      expect(result.isRight, true);
      result.fold(
        (failure) => fail('Should not return failure'),
        (user) {
          expect(user.id.value, '123');
          expect(user.name, 'John Doe');
        },
      );
      verify(() => mockRemoteDataSource.getUserById('123')).called(1);
    });

    test('should return ServerFailure when ServerException is thrown', 
        () async {
      // Arrange
      when(() => mockRemoteDataSource.getUserById(any()))
          .thenThrow(ServerException('Server error'));

      // Act
      final result = await repository.getUserById(tUserId);

      // Assert
      expect(result.isLeft, true);
      result.fold(
        (failure) {
          expect(failure, isA<ServerFailure>());
          expect(failure.message, 'Server error');
        },
        (user) => fail('Should not return user'),
      );
    });

    test('should return NetworkFailure when NetworkException is thrown', 
        () async {
      // Arrange
      when(() => mockRemoteDataSource.getUserById(any()))
          .thenThrow(NetworkException('Network error'));

      // Act
      final result = await repository.getUserById(tUserId);

      // Assert
      expect(result.isLeft, true);
      result.fold(
        (failure) {
          expect(failure, isA<NetworkFailure>());
          expect(failure.message, 'Network error');
        },
        (user) => fail('Should not return user'),
      );
    });

    test('should return MappingFailure when mapping fails', () async {
      // Arrange
      final invalidDto = UserDto(
        userId: '123',
        fullName: 'John Doe',
        emailAddress: 'invalid-email', // Email inválido
        accountStatus: 1,
        createdTimestamp: '2024-01-01T00:00:00Z',
      );
      
      when(() => mockRemoteDataSource.getUserById(any()))
          .thenAnswer((_) async => invalidDto);

      // Act
      final result = await repository.getUserById(tUserId);

      // Assert
      expect(result.isLeft, true);
      result.fold(
        (failure) => expect(failure, isA<MappingFailure>()),
        (user) => fail('Should not return user'),
      );
    });
  });

  group('createUser', () {
    const tName = 'John Doe';
    const tEmail = 'john@example.com';
    final tUserDto = UserDto(
      userId: '123',
      fullName: tName,
      emailAddress: tEmail,
      accountStatus: 0,
      createdTimestamp: '2024-01-01T00:00:00Z',
    );

    test('should create user successfully', () async {
      // Arrange
      when(() => mockRemoteDataSource.createUser(any()))
          .thenAnswer((_) async => tUserDto);

      // Act
      final result = await repository.createUser(
        name: tName,
        email: tEmail,
      );

      // Assert
      expect(result.isRight, true);
      result.fold(
        (failure) => fail('Should not return failure'),
        (user) {
          expect(user.name, tName);
          expect(user.email.value, tEmail);
          expect(user.status, UserStatus.pending);
        },
      );
    });

    test('should return ValidationFailure for invalid email', () async {
      // Act
      final result = await repository.createUser(
        name: tName,
        email: 'invalid-email',
      );

      // Assert
      expect(result.isLeft, true);
      result.fold(
        (failure) => expect(failure, isA<ValidationFailure>()),
        (user) => fail('Should not return user'),
      );
      verifyNever(() => mockRemoteDataSource.createUser(any()));
    });
  });
}
```

---

## Benefícios e Trade-offs

### ✅ Benefícios

| Benefício | Descrição |
|-----------|-----------|
| **Isolamento** | Mudanças na API não afetam o domínio |
| **Flexibilidade** | Fácil trocar APIs ou adicionar novas fontes |
| **Testabilidade** | Cada camada testável independentemente |
| **Manutenibilidade** | Mudanças localizadas, sem efeito cascata |
| **Type Safety** | Value Objects e Enums garantem tipos seguros |
| **Expressividade** | Código reflete linguagem de negócio |
| **Resiliência** | Sistema continua funcionando mesmo com mudanças na API |

### ⚠️ Trade-offs

| Desvantagem | Mitigação |
|-------------|-----------|
| **Código Extra** | Use code generation para DTOs |
| **Complexidade Inicial** | Documentação e padrões claros |
| **Performance** | Impacto mínimo; considere cache se necessário |
| **Curva de Aprendizado** | Treinamento da equipe |

---

## Boas Práticas

### 1. Organize os Mappers

```dart
// ✅ BOM: Mapper como classe abstrata com métodos estáticos
abstract class UserMapper {
  static User toDomain(UserDto dto) { ... }
  static UserDto toDto(User user) { ... }
}

// ❌ EVITE: Mapper como instância (overhead desnecessário)
class UserMapper {
  User toDomain(UserDto dto) { ... }
}
```

### 2. Valide Cedo

```dart
// ✅ BOM: Valida no Value Object
class Email {
  factory Email(String value) {
    if (!_isValid(value)) {
      throw ArgumentError('Invalid email');
    }
    return Email._(value);
  }
}

// ❌ EVITE: Validação dispersa pelo código
if (!isValidEmail(email)) { ... }
```

### 3. Use Enums Expressivos

```dart
// ✅ BOM: Enum com comportamento
enum UserStatus {
  active,
  inactive;
  
  bool get canLogin => this == active;
}

// ❌ EVITE: Strings ou números mágicos
const STATUS_ACTIVE = 1;
```

### 4. Trate Erros Específicos

```dart
// ✅ BOM: Hierarquia de erros clara
try {
  return UserMapper.toDomain(dto);
} on MappingException catch (e) {
  return Left(MappingFailure(e.message));
} on ArgumentError catch (e) {
  return Left(ValidationFailure(e.message));
}

// ❌ EVITE: Catch genérico
try {
  return UserMapper.toDomain(dto);
} catch (e) {
  return Left(Failure('Error'));
}
```

### 5. Documente o Contrato

```dart
/// DTO que representa EXATAMENTE o contrato da API
/// 
/// Estrutura esperada:
/// ```json
/// {
///   "user_id": "string",
///   "full_name": "string",
///   "account_status": 0 | 1 | 2 | 3
/// }
/// ```
class UserDto { ... }
```

### 6. Mantenha DTOs Imutáveis

```dart
// ✅ BOM: DTO imutável
class UserDto {
  final String userId;
  final String fullName;
  
  const UserDto({required this.userId, required this.fullName});
}

// ❌ EVITE: DTO mutável
class UserDto {
  String userId;
  String fullName;
}
```

### 7. Use Code Generation

```yaml
# pubspec.yaml
dependencies:
  json_annotation: ^4.8.0

dev_dependencies:
  build_runner: ^2.4.0
  json_serializable: ^6.7.0
```

```bash
# Gere código automaticamente
dart run build_runner build --delete-conflicting-outputs
```

---

## Exemplo Prático: Mudança na API

### Cenário: API mudou o campo de email

#### Antes (API v1):
```json
{
  "email_address": "john@example.com"
}
```

#### Depois (API v2):
```json
{
  "contact": {
    "primary_email": "john@example.com"
  }
}
```

### Solução com ACL:

```dart
// 1. Atualize APENAS o DTO
class UserDto {
  @JsonKey(name: 'contact')
  final ContactDto? contact;
  
  // Campo antigo mantido para compatibilidade
  @JsonKey(name: 'email_address')
  final String? emailAddress;
}

class ContactDto {
  @JsonKey(name: 'primary_email')
  final String primaryEmail;
}

// 2. Atualize APENAS o Mapper
static User toDomain(UserDto dto) {
  // Prioriza novo formato, fallback para antigo
  final email = dto.contact?.primaryEmail ?? dto.emailAddress ?? '';
  
  return User(
    // ... outros campos
    email: Email(email),
  );
}
```

**Resultado**: Domínio permanece intacto! ✅

---

## Conclusão

O Anti-Corruption Layer é um investimento em:
- 🛡️ **Proteção**: Seu domínio está isolado de mudanças externas
- 🔧 **Manutenibilidade**: Mudanças localizadas e previsíveis
- 📈 **Escalabilidade**: Fácil adicionar novas fontes de dados
- 🧪 **Testabilidade**: Cada camada testável independentemente

### Quando Implementar?

- ✅ Sempre que integrar com APIs externas
- ✅ Projetos de médio a longo prazo
- ✅ Equipes que valorizam código limpo
- ✅ Sistemas que precisam de manutenibilidade

### Lembre-se:

> "O custo de implementar um ACL é pequeno comparado ao custo de refatorar um sistema inteiro quando a API mudar."

---

## Recursos Adicionais

### Livros
- **Domain-Driven Design** - Eric Evans
- **Clean Architecture** - Robert C. Martin
- **Implementing Domain-Driven Design** - Vaughn Vernon

### Artigos
- [Martin Fowler - Anti-Corruption Layer](https://martinfowler.com/bliki/AnticorruptionLayer.html)
- [Microsoft - Anti-Corruption Layer Pattern](https://docs.microsoft.com/azure/architecture/patterns/anti-corruption-layer)

### Packages Dart Úteis
```yaml
dependencies:
  # Serialização JSON
  json_annotation: ^4.8.0
  
  # Either/Result type
  dartz: ^0.10.1
  # ou
  fpdart: ^1.1.0
  
  # Dependency Injection
  get_it: ^7.6.0
  injectable: ^2.3.2

dev_dependencies:
  # Code generation
  build_runner: ^2.4.0
  json_serializable: ^6.7.0
  
  # Testing
  mocktail: ^1.0.1
```

---

**Criado por:** Documentação de Arquitetura de Software  
**Data:** 2024  
**Versão:** 1.0