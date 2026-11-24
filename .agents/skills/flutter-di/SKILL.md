---
name: flutter-di
description: Configures dependency injection using get_it and injectable. Use when registering services, repositories, BLoCs, or setting up the DI container for a new feature.
---

# Flutter Dependency Injection Pattern

> **What:** Strategies for wiring dependencies in Flutter — constructor injection, factory functions, service locator (`get_it`), and widget-tree scoping.
> **When to use:** Creating a new feature's dependency wiring, setting up app-wide singletons, or choosing between DI approaches.
> **Depends on:** Works alongside `flutter-clean-architecture` and `flutter-bloc-pattern`.

## DI Strategies in Flutter

| Approach | When to use | Examples |
|---|---|---|
| **Constructor injection** | Default for BLoCs, use cases, repositories | Pass dependencies via constructor params |
| **Factory functions** | Per-feature wiring without a global container | `createFeatureBloc()` in `_dependencies.dart` |
| **Service locator (`get_it`)** | App-wide singletons (preferences, analytics, API client) | `GetIt.instance<T>()` or custom `AppDI` |
| **InheritedWidget / Provider** | Widget-tree scoped dependencies | `BlocProvider`, `RepositoryProvider` |

## Constructor Injection (Preferred)

The primary DI mechanism. Every class declares its dependencies as constructor parameters.

```dart
class LoginBloc extends Bloc<LoginEvent, LoginState> {
  final LoginUserUseCase _loginUserUseCase;
  final GoogleLoginUseCase _googleLoginUseCase;

  LoginBloc({
    required LoginUserUseCase loginUserUseCase,
    required GoogleLoginUseCase googleLoginUseCase,
  }) : _loginUserUseCase = loginUserUseCase,
       _googleLoginUseCase = googleLoginUseCase,
       super(const LoginState()) {
    on<LoginSubmitted>(_onLoginSubmitted);
    on<GoogleLoginRequested>(_onGoogleLoginRequested);
  }
}
```

Rules:
- Use `required` named parameters for mandatory dependencies
- Store as `final` private fields (prefix with `_`)
- Never use `late` for injected dependencies — they must be available at construction time
- Prefer interfaces (abstract classes) over concrete types for testability
- Declare parameter types as the abstract interface, not the implementation:
  ```dart
  // Good — depends on abstraction
  FeatureRepositoryImpl({required FeatureRemoteDataSource remoteDataSource})

  // Bad — depends on concrete class
  FeatureRepositoryImpl({required FeatureRemoteDataSourceImpl remoteDataSource})
  ```

## Factory Functions (Feature-Level Wiring)

Each feature module has a `<feature>_dependencies.dart` file that wires the dependency graph:

```dart
// feature_dependencies.dart

FeatureRepositoryImpl _createRepository() {
  return FeatureRepositoryImpl(
    remoteDataSource: FeatureRemoteDataSourceImpl(),
    mapper: FeatureMapper(),
  );
}

FeatureBloc createFeatureBloc() {
  final repository = _createRepository();
  return FeatureBloc(
    loadItemsUseCase: LoadItemsUseCase(repository: repository),
    deleteItemUseCase: DeleteItemUseCase(repository: repository),
  );
}
```

Rules:
- One public factory per BLoC/Cubit the feature exposes
- Internal helpers (repository, data source) can be private functions
- The screen calls the factory in `getBlocInstance()`:
  ```dart
  @override
  FeatureBloc getBlocInstance() => createFeatureBloc();
  ```
- Factory functions create fresh instances each time — no hidden singletons
- If a dependency is shared across features, register it in the service locator instead

## Service Locator (App-Wide Singletons)

For dependencies that live for the entire app lifetime (preferences, API client, analytics). Use `get_it` (the standard Flutter community choice) or a lightweight custom container:

### Using `get_it` (Recommended)

```dart
// di/injection_container.dart
import 'package:get_it/get_it.dart';

final sl = GetIt.instance; // sl = service locator

Future<void> initDependencies() async {
  // External
  final prefs = await SharedPreferences.getInstance();
  sl.registerLazySingleton<SharedPreferences>(() => prefs);

  // Core
  sl.registerLazySingleton<ApiClient>(() => ApiClient());
  sl.registerLazySingleton<AnalyticsService>(() => AnalyticsService());

  // Feature-specific (if needed globally)
  sl.registerFactory<AuthRepository>(
    () => AuthRepositoryImpl(remoteDataSource: sl(), mapper: sl()),
  );
}
```

### Using a Custom Container

```dart
class AppDI {
  static T register<T>(T obj, {String? instanceName, bool override = false});
  static T get<T>({String? instanceName});
  static void unregister<T>({String? instanceName});
  static bool isRegistered<T>({String? instanceName});
  static Future<void> clear();
}
```

### Registration (during app initialization)

```dart
// In main() — before runApp
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await initDependencies();
  runApp(const MyApp());
}
```

### Registration Types

| Type | Method | Lifecycle |
|---|---|---|
| **Singleton** | `registerSingleton<T>(instance)` | Created immediately, lives forever |
| **Lazy singleton** | `registerLazySingleton<T>(() => instance)` | Created on first access, lives forever |
| **Factory** | `registerFactory<T>(() => instance)` | New instance every time |
| **Async singleton** | `registerSingletonAsync<T>(() async => instance)` | Async initialization |

### Rules
- Register singletons early in the app lifecycle (before `runApp`)
- Use `registerLazySingleton` over `registerSingleton` when initialization is expensive
- Use `registerFactory` for objects that should not be shared (e.g., BLoCs if registered globally)
- Implement `Disposable` interface for objects that need cleanup:
  ```dart
  abstract class Disposable {
    void dispose();
  }
  ```
- Use named instances when you need multiple registrations of the same type:
  ```dart
  sl.registerSingleton<ApiClient>(productionClient, instanceName: 'production');
  sl.registerSingleton<ApiClient>(stagingClient, instanceName: 'staging');
  ```
- For larger projects, consider `injectable` package for annotation-based auto-registration — it eliminates boilerplate registration code and reduces configuration errors
- Prefer explicit constructor injection over service locator access in business logic classes. Reserve service locator for app bootstrap and cross-cutting concerns

## Widget-Tree Scoped DI

For dependencies scoped to a subtree (a screen, a tab, a dialog):

```dart
// BlocProvider scopes a BLoC to the widget subtree
BlocProvider<FeatureBloc>(
  create: (_) => createFeatureBloc(),
  child: const FeatureScreen(),
)

// RepositoryProvider for sharing a repository across child widgets
RepositoryProvider<FeatureRepository>(
  create: (_) => createFeatureRepository(),
  child: const FeatureScreen(),
)

// MultiBlocProvider for multiple BLoCs
MultiBlocProvider(
  providers: [
    BlocProvider(create: (_) => AppBloc()..add(const LoadSavedConfig())),
    BlocProvider(create: (_) => ThemeBloc()),
  ],
  child: const MyApp(),
)

// MultiRepositoryProvider for multiple repositories
MultiRepositoryProvider(
  providers: [
    RepositoryProvider(create: (_) => AuthRepository()),
    RepositoryProvider(create: (_) => UserRepository()),
  ],
  child: const MyApp(),
)
```

Rules:
- `BlocProvider` automatically closes the BLoC when the widget is disposed
- Never close a BLoC manually if it's managed by `BlocProvider`
- Use `BlocProvider.value` only when passing an existing BLoC to a new route (e.g., dialog, bottom sheet) — it does not manage the BLoC lifecycle
- Access via `context.read<T>()` (one-time access, use in callbacks) or `context.watch<T>()` (reactive, use in `build()`)
- Never call `context.read<T>()` or `context.watch<T>()` inside `create` callbacks — the BLoC is not yet in the tree

## Choosing the Right Approach

```
Is it app-wide and lives forever?
  → Service locator (get_it or AppDI.register)

Is it feature-specific and created per screen?
  → Factory function + BlocProvider

Is it shared between parent and child widgets?
  → RepositoryProvider or BlocProvider.value

Is it a direct dependency of a class?
  → Constructor injection (always)
```

## Testing with DI

Constructor injection makes testing straightforward:

```dart
// Create mock
class MockLoginUseCase extends Mock implements LoginUserUseCase {}

// Inject mock
final bloc = LoginBloc(
  loginUserUseCase: MockLoginUseCase(),
  googleLoginUseCase: MockGoogleLoginUseCase(),
);
```

For service locator dependencies in tests:

```dart
setUp(() {
  // Reset get_it before each test
  sl.reset();
  sl.registerSingleton<PreferenceManager>(MockPreferenceManager());
});

tearDown(() async {
  await sl.reset();
});
```

For widget tests with BlocProvider:

```dart
testWidgets('renders feature screen', (tester) async {
  final mockBloc = MockFeatureBloc();
  when(() => mockBloc.state).thenReturn(const FeatureState());

  await tester.pumpWidget(
    MaterialApp(
      home: BlocProvider<FeatureBloc>.value(
        value: mockBloc,
        child: const FeatureScreen(),
      ),
    ),
  );

  expect(find.byType(FeatureScreen), findsOneWidget);
});
```

## Anti-Patterns

- **Static access everywhere**: `SomeService.instance.doThing()` scattered across code makes testing hard. Prefer injecting via constructor
- **God DI file**: one massive file registering everything. Keep wiring per-feature or split into modules
- **Circular dependencies**: A depends on B, B depends on A. Break the cycle with an interface or event-based communication
- **Registering BLoCs in service locator as singletons**: BLoCs are screen-scoped, not app-scoped. Use `BlocProvider`, not `registerSingleton`
- **Late initialization**: using `late final` for dependencies that should be constructor-injected
- **Accessing context-based DI outside widget tree**: `context.read<T>()` only works inside the widget tree. For non-widget code, use constructor injection or service locator
- **Over-injecting**: if a class needs 5+ dependencies, it may be doing too much. Consider splitting responsibilities
