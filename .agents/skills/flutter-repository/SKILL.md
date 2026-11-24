---
name: flutter-repository
description: Implements the repository pattern as the single source of truth for data access. Use when adding data sources, caching strategies, or coordinating between remote and local storage.
---

# Flutter Repository Pattern

> **What:** Step-by-step guide for implementing the Repository pattern in Flutter Clean Architecture — data sources, DTOs, mappers, and repository implementations.
> **When to use:** Building or modifying the data layer for a feature (API integration, caching, DTOs, mappers).
> **Depends on:** Works alongside `flutter-clean-architecture`.

## Purpose

The repository sits between the domain layer and the data layer. It:
- Abstracts data sources (remote API, local DB, cache) behind a clean interface
- Converts raw data (DTOs) into domain entities
- Wraps results in typed response objects (e.g., `Either<Failure, T>` or sealed classes) so callers never deal with raw exceptions
- Coordinates between multiple data sources (remote, local cache) when needed

## Structure

```
data/
├── api/
│   ├── data_source/
│   │   ├── feature_remote_data_source.dart        # Abstract interface
│   │   └── feature_remote_data_source_impl.dart    # API calls
│   ├── mapper/
│   │   └── feature_mapper.dart                     # DTO → Entity
│   └── model/
│       └── feature_response_dto.dart               # JSON models
└── repository/
    └── feature_repository_impl.dart                # Implements domain interface

domain/
├── entities/
│   └── feature_entity.dart                         # Clean domain model
└── repository/
    └── feature_repository.dart                     # Abstract interface
```

## Step-by-Step Implementation

### 1. Define the Domain Entity

Immutable plain Dart class. No JSON annotations, no framework dependencies. Use `const` constructors.

```dart
class Product {
  final int id;
  final String name;
  final double price;
  final bool isAvailable;

  const Product({
    required this.id,
    required this.name,
    required this.price,
    required this.isAvailable,
  });

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is Product &&
          runtimeType == other.runtimeType &&
          id == other.id;

  @override
  int get hashCode => id.hashCode;
}
```

Consider using `freezed` or `equatable` for entities with many fields to auto-generate equality, `hashCode`, `copyWith`, and `toString`.

### 2. Define the Repository Interface (Domain Layer)

Abstract class in `domain/repository/`. Returns the project's response wrapper type.

```dart
// Option A: Using sealed class response wrapper
abstract class ProductRepository {
  Future<RepositoryResponse<List<Product>>> getProducts(int page, int limit);
  Future<RepositoryResponse<Product>> getProductById(int id);
  Future<RepositoryResponse<Product>> createProduct(String name, double price);
}

// Option B: Using Either (from dartz/fpdart) — more functional approach
abstract class ProductRepository {
  Future<Either<Failure, List<Product>>> getProducts(int page, int limit);
  Future<Either<Failure, Product>> getProductById(int id);
  Future<Either<Failure, Product>> createProduct(String name, double price);
}
```

Rules:
- Method signatures use primitive types or domain entities as parameters
- Return type is always the response wrapper (e.g., `RepositoryResponse<T>`, `Either<Failure, T>`, `Result<T>`)
- Use generic type parameters for type-safe responses
- No data-layer imports (no Dio, no JSON, no DTOs)

### 3. Create the API Response DTO (Data Layer)

Matches the JSON shape from the API. Lives in `data/api/model/`.

```dart
// Option A: Manual serialization
class ProductResponseDto {
  final int? id;
  final String? name;
  final double? price;
  final int? status;

  const ProductResponseDto({this.id, this.name, this.price, this.status});

  factory ProductResponseDto.fromJson(Map<String, dynamic> json) {
    return ProductResponseDto(
      id: json['id'] as int?,
      name: json['name'] as String?,
      price: (json['price'] as num?)?.toDouble(),
      status: json['status'] as int?,
    );
  }

  Map<String, dynamic> toJson() => {
    'id': id,
    'name': name,
    'price': price,
    'status': status,
  };
}

// Option B: Using json_serializable (recommended for larger projects)
@JsonSerializable()
class ProductResponseDto {
  final int? id;
  final String? name;
  final double? price;
  final int? status;

  const ProductResponseDto({this.id, this.name, this.price, this.status});

  factory ProductResponseDto.fromJson(Map<String, dynamic> json) =>
      _$ProductResponseDtoFromJson(json);

  Map<String, dynamic> toJson() => _$ProductResponseDtoToJson(this);
}

// Option C: Using freezed (for immutability + code generation)
@freezed
class ProductResponseDto with _$ProductResponseDto {
  const factory ProductResponseDto({
    int? id,
    String? name,
    double? price,
    int? status,
  }) = _ProductResponseDto;

  factory ProductResponseDto.fromJson(Map<String, dynamic> json) =>
      _$ProductResponseDtoFromJson(json);
}
```

Rules:
- All fields nullable — APIs can return unexpected shapes
- `fromJson` factory constructor for deserialization
- `toJson` method only if needed for request bodies
- No domain logic — this is a pure data transfer object
- Use `const` constructors where possible
- For projects with many DTOs, prefer `json_serializable` or `freezed` to reduce boilerplate and avoid manual serialization bugs

### 4. Create the Mapper (Data Layer)

Converts DTOs to domain entities. Lives in `data/api/mapper/`.

```dart
class ProductMapper {
  Product toEntity(ProductResponseDto dto) {
    return Product(
      id: dto.id ?? 0,
      name: dto.name ?? '',
      price: dto.price ?? 0.0,
      isAvailable: dto.status == 1,
    );
  }

  List<Product> toEntityList(List<dynamic> jsonList) {
    return jsonList
        .map((json) => toEntity(ProductResponseDto.fromJson(json)))
        .toList();
  }
}
```

Rules:
- Handle null/missing fields with sensible defaults
- Transform API-specific values into domain concepts (e.g., `status: 1` → `isAvailable: true`)
- Keep it stateless — mapper is a pure function wrapper

### 5. Create the Data Source (Data Layer)

Abstract interface + implementation. Lives in `data/api/data_source/`.

#### Option A: Type-Safe Retrofit Data Source (Recommended)
See `flutter-retrofit` for the complete guide.

```dart
// Retrofit abstract client
@RestApi()
abstract class ProductApiClient {
  factory ProductApiClient(Dio dio, {String baseUrl}) = _ProductApiClient;

  @GET('/api/products')
  Future<List<ProductResponseDto>> getProducts(
    @Query('page') int page,
    @Query('limit') int limit,
  );
}

// Data Source wraps the Retrofit client & catches DioExceptions
class ProductRemoteDataSourceImpl implements ProductRemoteDataSource {
  final ProductApiClient _client;
  ProductRemoteDataSourceImpl(this._client);

  @override
  Future<List<ProductResponseDto>> getProducts(int page, int limit) async {
    try {
      return await _client.getProducts(page, limit);
    } on DioException catch (e) {
      throw mapDioException(e);
    }
  }
}
```

#### Option B: Standard Communication Manager
```dart
// Abstract
abstract class ProductRemoteDataSource {
  Future<NetworkResponse> getProducts(int page, int limit);
  Future<NetworkResponse> getProductById(int id);
  Future<NetworkResponse> createProduct(Map<String, dynamic> data);
}

// Implementation
class ProductRemoteDataSourceImpl implements ProductRemoteDataSource {
  @override
  Future<NetworkResponse> getProducts(int page, int limit) async {
    return await communicationManager.get(
      baseUrl: environmentManager.getBaseUrl(),
      urlEndPoint: '/api/products',
      headers: _getDefaultHeaders(),
      queryParams: {'page': page, 'limit': limit},
    );
  }

  // ... other methods
}
```

Rules:
- Returns typed DTOs or network response wrapper
- For REST APIs, prefer Retrofit + Dio (`flutter-retrofit`) to guarantee compile-time type safety
- Handles header construction, URL building, request body formatting
- One method per API endpoint
- No business logic — just HTTP plumbing

### 6. Implement the Repository (Data Layer)

Ties everything together. Lives in `data/repository/`.

```dart
class ProductRepositoryImpl implements ProductRepository {
  final ProductRemoteDataSource _remoteDataSource;
  final ProductMapper _mapper;

  ProductRepositoryImpl({
    required ProductRemoteDataSource remoteDataSource,
    required ProductMapper mapper,
  }) : _remoteDataSource = remoteDataSource,
       _mapper = mapper;

  @override
  Future<RepositoryResponse> getProducts(int page, int limit) async {
    final networkResponse = await _remoteDataSource.getProducts(page, limit);
    return networkResponseToRepositoryResponseConverter(
      networkResponse,
      convertResponseData: (success) async {
        final list = success.data['items'] as List? ?? [];
        final products = _mapper.toEntityList(list);
        return RepositorySuccessResponse(data: products);
      },
    );
  }
}
```

Rules:
- Constructor injection for data source and mapper
- Use the project's response converter functions
- Parse JSON, map to entities, wrap in success response
- Let the converter handle error cases automatically
- If the API returns non-standard error shapes, provide a custom `convertServerErrorData` callback

## Response Conversion Chain

```
DataSource returns NetworkResponse
    ↓ networkResponseToRepositoryResponseConverter()
Repository returns RepositoryResponse
    ↓ repositoryResponseToUseCaseResponseConverter()
UseCase returns UseCaseResponse
    ↓ pattern match in BLoC
BLoC emits State
```

Each converter handles the error mapping automatically. You only write the success path.

## Adding a Local Data Source (sqflite / cache)

When a feature needs caching, offline-first support, or persistent database tables, use `sqflite` (see `flutter-sqflite` for schema management, migrations, and transactions):

```
data/
├── api/
│   └── data_source/
│       ├── feature_remote_data_source.dart
│       └── feature_remote_data_source_impl.dart
├── local/
│   └── data_source/
│       ├── feature_local_data_source.dart          # sqflite or key-value cache
│       └── feature_local_data_source_impl.dart
└── repository/
    └── feature_repository_impl.dart                # Coordinates remote + local
```

The repository impl decides the caching strategy (Network-First, Cache-First, or Stale-While-Revalidate):

```dart
class ProductRepositoryImpl implements ProductRepository {
  final ProductRemoteDataSource _remoteDataSource;
  final ProductLocalDataSource _localDataSource;
  final ProductMapper _mapper;
  final NetworkInfo _networkInfo;

  // Network-first with local fallback
  @override
  Future<Either<Failure, List<Product>>> getProducts(int page, int limit) async {
    if (await _networkInfo.isConnected) {
      try {
        final response = await _remoteDataSource.getProducts(page, limit);
        final products = _mapper.toEntityList(response.data);
        await _localDataSource.cacheProducts(products);
        return Right(products);
      } on ServerException catch (e) {
        return Left(ServerFailure(e.message));
      }
    } else {
      final cachedProducts = await _localDataSource.getCachedProducts();
      if (cachedProducts != null) return Right(cachedProducts);
      return const Left(CacheFailure('No cached data available'));
    }
  }
}
```

Common caching strategies:
- **Network-first**: try remote, fall back to cache on failure
- **Cache-first**: return cache if available, fetch remote in background
- **Network-only**: always fetch from remote (default for most features)
- **Cache-only**: offline-first features (e.g., saved drafts)

## Error Handling Best Practices

Define a `Failure` hierarchy in the domain layer so callers get typed errors:

```dart
// domain/failures/failure.dart
sealed class Failure {
  final String message;
  const Failure(this.message);
}

class ServerFailure extends Failure {
  final int? statusCode;
  const ServerFailure(super.message, {this.statusCode});
}

class ConnectionFailure extends Failure {
  const ConnectionFailure([super.message = 'No internet connection']);
}

class CacheFailure extends Failure {
  const CacheFailure([super.message = 'Cache error']);
}

class UnknownFailure extends Failure {
  const UnknownFailure([super.message = 'Something went wrong']);
}
```

The repository catches exceptions from data sources and converts them to typed failures — the domain and presentation layers never see raw exceptions.

## Checklist

- [ ] Domain entity is an immutable Dart class with `const` constructor and no external dependencies
- [ ] Repository interface is in `domain/repository/` with no data-layer imports
- [ ] Return types use generics for type safety (e.g., `Either<Failure, Product>`)
- [ ] DTO fields are all nullable with `fromJson` factory
- [ ] Mapper handles null fields with sensible defaults
- [ ] Data source returns raw network response, no parsing
- [ ] Repository impl catches exceptions and converts to typed failures
- [ ] Repository impl uses converter functions for response wrapping
- [ ] Dependencies file has factory function to wire everything together
- [ ] Unit tests cover success path, error paths, and null/edge cases
