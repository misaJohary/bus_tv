---
name: flutter-retrofit
description: Implements type-safe REST API clients using Retrofit and Dio. Use when building remote data sources, configuring HTTP interceptors, or handling network errors.
---

# Flutter Retrofit & Dio Pattern

> **What:** Guide for creating robust, type-safe REST API clients using `retrofit` and `dio` in Flutter.
> **When to use:** Integrating remote backend APIs, handling authentication headers, token refresh flows, error normalization, and file transfers.
> **Depends on:** Works alongside `flutter-serialization`, `flutter-repository`, and `flutter-clean-architecture`.

`retrofit` turns your REST API interface into a type-safe client powered by `dio`. It eliminates repetitive HTTP request handling and integrates directly with `json_serializable` DTOs.

---

## 1. Setup & Dependencies

Add the dependencies to `pubspec.yaml`:

```yaml
dependencies:
  dio: ^5.7.0
  retrofit: ^4.4.1

dev_dependencies:
  build_runner: ^2.4.14
  retrofit_generator: ^8.2.0
```

---

## 2. Defining the Retrofit API Interface

Create an abstract class annotated with `@RestApi()`:

```dart
import 'package:dio/dio.dart';
import 'package:retrofit/retrofit.dart';
import '../model/user_dto.dart';
import '../model/login_request_dto.dart';
import '../model/login_response_dto.dart';
import '../model/paginated_response_dto.dart';

part 'api_client.g.dart';

@RestApi()
abstract class ApiClient {
  factory ApiClient(Dio dio, {String baseUrl}) = _ApiClient;

  // GET with query parameters
  @GET('/users')
  Future<PaginatedResponseDto<UserDto>> getUsers(
    @Query('page') int page,
    @Query('limit') int limit,
    @Query('search') String? search,
  );

  // GET with path parameter
  @GET('/users/{id}')
  Future<UserDto> getUserById(@Path('id') String id);

  // POST with JSON body
  @POST('/auth/login')
  Future<LoginResponseDto> login(@Body() LoginRequestDto request);

  // PUT with dynamic header
  @PUT('/users/{id}')
  Future<UserDto> updateUser(
    @Path('id') String id,
    @Body() Map<String, dynamic> updateData,
    @Header('X-Custom-Header') String customHeader,
  );

  // DELETE
  @DELETE('/users/{id}')
  Future<void> deleteUser(@Path('id') String id);

  // Multipart file upload with progress
  @POST('/users/{id}/avatar')
  @MultiPart()
  Future<UserDto> uploadAvatar(
    @Path('id') String id,
    @Part(name: 'file') File file,
    @SendProgress() ProgressCallback? onSendProgress,
  );
}
```

---

## 3. Dio Configuration & Factory

Centralize Dio instance creation with sane defaults (timeouts, validation):

```dart
class DioFactory {
  static Dio createDio({required String baseUrl, required AuthInterceptor authInterceptor}) {
    final dio = Dio(
      BaseOptions(
        baseUrl: baseUrl,
        connectTimeout: const Duration(seconds: 15),
        receiveTimeout: const Duration(seconds: 15),
        sendTimeout: const Duration(seconds: 15),
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json',
        },
        responseType: ResponseType.json,
      ),
    );

    dio.interceptors.addAll([
      authInterceptor,
      LogInterceptor(
        requestBody: true,
        responseBody: true,
        logPrint: (obj) => debugPrint('[DIO] $obj'),
      ),
    ]);

    return dio;
  }
}
```

---

## 4. Auth & Automated 401 Token Refresh Interceptor

Use Dio's `QueuedInterceptor` to safely pause concurrent requests while refreshing expired access tokens:

```dart
import 'package:dio/dio.dart';

class AuthInterceptor extends QueuedInterceptor {
  final TokenStorage _tokenStorage;
  final Dio _refreshDio; // Separate Dio instance to prevent interceptor loops

  AuthInterceptor({
    required TokenStorage tokenStorage,
    required Dio refreshDio,
  })  : _tokenStorage = tokenStorage,
        _refreshDio = refreshDio;

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    // Inject Bearer token if available
    final token = await _tokenStorage.getAccessToken();
    if (token != null && !options.headers.containsKey('Authorization')) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    return handler.next(options);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    // Check if error is 401 Unauthorized
    if (err.response?.statusCode == 401) {
      final refreshToken = await _tokenStorage.getRefreshToken();
      if (refreshToken != null) {
        try {
          // Attempt to refresh the access token
          final response = await _refreshDio.post(
            '/auth/refresh',
            data: {'refresh_token': refreshToken},
          );

          final newAccessToken = response.data['access_token'];
          final newRefreshToken = response.data['refresh_token'];
          await _tokenStorage.saveTokens(newAccessToken, newRefreshToken);

          // Retry the original request with the new token
          final requestOptions = err.requestOptions;
          requestOptions.headers['Authorization'] = 'Bearer $newAccessToken';
          
          final clonedResponse = await _refreshDio.fetch(requestOptions);
          return handler.resolve(clonedResponse);
        } catch (_) {
          // Refresh failed - purge credentials and force logout
          await _tokenStorage.clear();
        }
      }
    }
    return handler.next(err);
  }
}
```

---

## 5. Normalized Error Handling & Failure Mapping

Map raw `DioException` instances into domain-layer failure sealed classes:

```dart
sealed class NetworkFailure {
  final String message;
  const NetworkFailure(this.message);
}

class ServerFailure extends NetworkFailure {
  final int statusCode;
  final dynamic errorData;
  const ServerFailure({required this.statusCode, required String message, this.errorData})
      : super(message);
}

class ConnectionFailure extends NetworkFailure {
  const ConnectionFailure([super.message = 'No internet connection. Please check your network.']);
}

class TimeoutFailure extends NetworkFailure {
  const TimeoutFailure([super.message = 'Request timed out. Please try again.']);
}

class UnauthorizedFailure extends NetworkFailure {
  const UnauthorizedFailure([super.message = 'Session expired. Please log in again.']);
}

class UnknownNetworkFailure extends NetworkFailure {
  const UnknownNetworkFailure([super.message = 'An unexpected network error occurred.']);
}

// Exception to Failure mapper
NetworkFailure mapDioException(DioException error) {
  switch (error.type) {
    case DioExceptionType.connectionTimeout:
    case DioExceptionType.sendTimeout:
    case DioExceptionType.receiveTimeout:
      return const TimeoutFailure();

    case DioExceptionType.connectionError:
      return const ConnectionFailure();

    case DioExceptionType.badResponse:
      final statusCode = error.response?.statusCode ?? 500;
      if (statusCode == 401) return const UnauthorizedFailure();
      
      final data = error.response?.data;
      final serverMessage = data is Map ? data['message']?.toString() : null;
      return ServerFailure(
        statusCode: statusCode,
        message: serverMessage ?? 'Server error ($statusCode)',
        errorData: data,
      );

    case DioExceptionType.cancel:
      return const UnknownNetworkFailure('Request was cancelled');

    default:
      return UnknownNetworkFailure(error.message ?? 'Unknown error');
  }
}
```

---

## 6. Remote Data Source Implementation Example

```dart
abstract class UserRemoteDataSource {
  Future<List<UserDto>> fetchUsers(int page);
}

class UserRemoteDataSourceImpl implements UserRemoteDataSource {
  final ApiClient _apiClient;

  UserRemoteDataSourceImpl(this._apiClient);

  @override
  Future<List<UserDto>> fetchUsers(int page) async {
    try {
      final response = await _apiClient.getUsers(page, 20, null);
      return response.items;
    } on DioException catch (e) {
      throw mapDioException(e);
    }
  }
}
```

---

## 7. Cancellation with `CancelToken`

Cancel in-flight requests when users leave screens or re-type search queries:

```dart
class SearchRemoteDataSource {
  final ApiClient _apiClient;
  CancelToken? _cancelToken;

  SearchRemoteDataSource(this._apiClient);

  Future<List<UserDto>> searchUsers(String query) async {
    // Cancel previous ongoing request
    _cancelToken?.cancel('New search triggered');
    _cancelToken = CancelToken();

    try {
      return await _apiClient.searchUsers(query, cancelToken: _cancelToken);
    } on DioException catch (e) {
      if (CancelToken.isCancel(e)) {
        return []; // Gracefully handle cancellation
      }
      throw mapDioException(e);
    }
  }
}
```

---

## 8. Best Practices Checklist

1. **Use `QueuedInterceptor` for auth**: Regular interceptors can trigger multiple simultaneous refresh calls on concurrent 401s; `QueuedInterceptor` queues them.
2. **Dedicated Dio instance for refresh**: Always use a clean Dio instance (without the auth interceptor) to call `/auth/refresh` to avoid infinite retry loops.
3. **Always specify timeouts**: Define `connectTimeout`, `receiveTimeout`, and `sendTimeout` explicitly.
4. **Never expose `DioException` to BLoC or UI**: Catch it in the data source / repository and map it to typed `Failure` classes.
5. **Use `CancelToken` for search & pagination**: Prevent stale search results from overwriting newer queries.
6. **Code generation command**:
   ```bash
   dart run build_runner build --delete-conflicting-outputs
   ```
