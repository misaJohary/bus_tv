---
name: flutter-architecture
description: Scaffolds Flutter feature modules following Clean Architecture with BLoC. Use when creating new features, setting up layer boundaries, or structuring a module end-to-end.
---

# Flutter Clean Architecture

> **What:** Patterns and rules for building features in a Flutter project following Clean Architecture with BLoC.
> **When to use:** Scaffolding a new feature module, reviewing layer boundaries, or onboarding new team members.
> **Depends on:** Optionally pairs with `flutter-bloc-pattern`, `flutter-repository-pattern`, `flutter-di-pattern`.

## Architecture Layers

Every feature is organized into three layers with strict dependency rules:

```
Presentation → Domain ← Data
```

- **Presentation** depends on **Domain** (uses entities, calls use cases)
- **Data** depends on **Domain** (implements repository interfaces)
- **Domain** depends on nothing (pure Dart, no Flutter imports in entities/use cases)

This aligns with the [official Flutter architecture recommendations](https://docs.flutter.dev/app-architecture/recommendations) which strongly recommend clear data and UI layer separation, the repository pattern in the data layer, and ViewModels/Views in the UI layer (MVVM). The BLoC pattern serves as the ViewModel in this architecture.

## Feature Module Structure

When creating a new feature, scaffold this folder structure:

```
modules/<feature_name>/
├── data/
│   ├── api/
│   │   ├── data_source/
│   │   │   ├── <feature>_remote_data_source.dart       # Abstract interface
│   │   │   └── <feature>_remote_data_source_impl.dart   # Implementation
│   │   ├── mapper/
│   │   │   └── <feature>_mapper.dart                    # DTO → Entity mappers
│   │   └── model/
│   │       └── <feature>_response.dart                  # API response DTOs (use freezed or json_serializable)
│   └── repository/
│       └── <feature>_repository_impl.dart               # Implements domain interface
├── domain/
│   ├── entities/
│   │   └── <feature>_entity.dart                        # Domain models (immutable)
│   ├── repository/
│   │   └── <feature>_repository.dart                    # Abstract repository interface
│   └── use_case/
│       └── <action>_use_case.dart                       # One class per business action
├── presentation/
│   ├── view/
│   │   └── <feature>_screen.dart                        # Screen widgets
│   ├── view_model/
│   │   ├── events/
│   │   │   └── <feature>_event.dart                     # BLoC events (sealed classes)
│   │   ├── states/
│   │   │   └── <feature>_state.dart                     # BLoC states (sealed or Equatable)
│   │   └── <feature>_bloc.dart                          # BLoC implementation
│   └── widgets/
│       └── <feature>_widget.dart                        # Feature-specific widgets
└── <feature>_dependencies.dart                          # Factory functions for DI wiring
```

## Layer Rules

### Domain Layer
- Contains **entities**, **repository interfaces** (abstract classes), and **use cases**
- Entities are immutable plain Dart classes — no framework annotations, no JSON serialization
- Use `const` constructors and `final` fields for all entities
- Repository interfaces define the contract; they return a typed response wrapper (e.g., `Either<Failure, T>` from dartz/fpdart, or a sealed class)
- Each use case has a single `call()` method (using Dart's callable class pattern) or `execute()` method that calls one repository method and converts the response
- Use cases should implement a common base class (e.g., `UseCase<Type, Params>`) for consistency

### Data Layer
- **Data source** (abstract + impl): makes raw API calls, returns network response wrappers
- **Model / DTO**: JSON-serializable classes matching API response shape. Use `fromJson` factory constructors. Consider `freezed` or `json_serializable` for code generation
- **Mapper**: converts DTOs to domain entities. Keep mapping logic here, not in repository. Consider using extension methods for cleaner mapping syntax
- **Repository impl**: implements the domain interface. Calls data source, uses mapper, wraps result in repository response type. Should handle exceptions and convert them to typed failures

### Presentation Layer
- **View**: screen widgets. Prefer `StatelessWidget` with `BlocProvider`/`BlocConsumer`. Extend the project's base widget class if one exists
- **View model (BLoC)**: extends `Bloc<Event, State>` or project's base BLoC. Register event handlers in constructor via `on<Event>`. Inject use cases through constructor. Emit states, never call repositories directly
- **Events**: use Dart 3 sealed classes for exhaustive pattern matching, or extend base event class with Equatable. One class per user action
- **States**: use Dart 3 sealed classes for distinct states, or extend base state class with Equatable. Use a status enum pattern (`initial`, `loading`, `success`, `failure`) plus data fields for simpler cases

## Response Flow Pattern

The architecture uses a layered response conversion pattern:

```
Network Response → Repository Response → UseCase Response → BLoC State
```

Each layer has its own sealed class hierarchy:
- **Network**: `NetworkSuccessResponse | NetworkServerError | NetworkConnectionError | NetworkUnknownError`
- **Repository**: `RepositorySuccessResponse | RepositoryServerError | RepositoryConnectionError | RepositoryUnknownError`
- **UseCase**: `UseCaseSuccessResponse | UseCaseServerError | UseCaseConnectionError | UseCaseUnknownError`

Alternatively, use `Either<Failure, T>` (from dartz/fpdart) for a more functional approach:
```dart
// Repository returns Either
Future<Either<Failure, List<Product>>> getProducts();

// Use case forwards or transforms
Future<Either<Failure, List<Product>>> call(NoParams params);

// BLoC folds the result
result.fold(
  (failure) => emit(state.copyWith(status: FeatureStatus.failure, error: failure.message)),
  (data) => emit(state.copyWith(status: FeatureStatus.success, data: data)),
);
```

Use converter functions to transform between layers:
- `networkResponseToRepositoryResponseConverter()` in repository impl
- `repositoryResponseToUseCaseResponseConverter()` in use case

## BLoC Conventions

- BLoC class extends `Bloc<EventType, StateType>` or project's `BaseBloc<EventType, StateType>`
- Register all event handlers in the constructor using `on<SpecificEvent>(handler)`
- Use `EventTransformer` for debouncing, throttling, or sequential processing where needed:
  ```dart
  on<SearchQueryChanged>(_onSearchChanged, transformer: debounce(const Duration(milliseconds: 300)));
  ```
- Use cases are injected via constructor, stored as private final fields
- Handle all response branches in event handlers: success, server error, connection error, unknown
- Dispose controllers and subscriptions in `close()`
- UI controllers (TextEditingController, FocusNode) live in the BLoC, not the widget
- Prefer `copyWith` on state to avoid losing existing state data when emitting

## Dependency Wiring

Each feature has a `<feature>_dependencies.dart` file at the module root with factory functions:

```dart
// Example pattern
FeatureRepositoryImpl createFeatureRepository() {
  return FeatureRepositoryImpl(
    remoteDataSource: FeatureRemoteDataSourceImpl(),
    mapper: FeatureMapper(),
  );
}

FeatureBloc createFeatureBloc() {
  final repository = createFeatureRepository();
  return FeatureBloc(
    someUseCase: SomeUseCase(repository: repository),
    anotherUseCase: AnotherUseCase(repository: repository),
  );
}
```

The screen's `getBlocInstance()` calls the factory function from this file.

For larger projects, consider using `get_it` + `injectable` for automated DI registration.

## Naming Conventions

| Item | Pattern | Example |
|---|---|---|
| Feature folder | `snake_case` | `login_register/` |
| BLoC class | `PascalCaseBloc` | `LoginBloc` |
| Event class | `PascalCaseEvent` | `LoginSendOtpRequested` |
| State class | `PascalCaseState` | `LoginState` |
| Use case class | `VerbNounUseCase` | `LoginUserUseCase` |
| Repository interface | `PascalCaseRepository` | `LoginRegisterRepository` |
| Repository impl | `PascalCaseRepositoryImpl` | `LoginRegisterRepositoryImpl` |
| Data source interface | `PascalCaseRemoteDataSource` | `LoginRegisterRemoteDataSource` |
| Data source impl | `PascalCaseRemoteDataSourceImpl` | `LoginRegisterRemoteDataSourceImpl` |
| Mapper | `PascalCaseMapper` | `LoginMapper` |
| Screen widget | `PascalCaseScreen` | `LoginScreen` |
| Dependencies file | `<feature>_dependencies.dart` | `login_register_dependencies.dart` |
| Failure class | `PascalCaseFailure` | `ServerFailure` |

## Checklist for New Features

1. Define domain entities first (immutable, with `const` constructors)
2. Define repository interface (what operations are needed?)
3. Create API models/DTOs matching the API response (use code generation where appropriate)
4. Implement data source with API calls
5. Create mapper (DTO → entity, handle nulls with defaults)
6. Implement repository (calls data source, maps, wraps response, handles exceptions)
7. Create use cases (one per action, calls repository, converts response)
8. Define BLoC events and states (prefer sealed classes for exhaustive matching)
9. Implement BLoC (inject use cases, handle events, emit states with `copyWith`)
10. Build screen widget using `BlocProvider`/`BlocConsumer` or base widget pattern
11. Create dependencies file with factory functions
12. Register route if this is a new screen
13. Write unit tests for use cases, mappers, repository, and BLoC
