---
name: flutter-bloc
description: Implements BLoC pattern for complex event-driven state management. Use when a feature requires handling multiple distinct events, side effects, or complex state transitions.
---

# Flutter BLoC Pattern

> **What:** Patterns for implementing BLoC (Business Logic Component) state management using `flutter_bloc` — events, states, event transformers, and UI integration.
> **When to use:** Complex event-driven flows, features needing debouncing/throttling, or when event traceability matters.
> **Depends on:** Works alongside `flutter-clean-architecture` and `flutter-di-pattern`.
> **See also:** For simpler state without event traceability, see `flutter-cubit-pattern`.

BLoC uses an event-driven architecture where the UI dispatches events and the BLoC emits states. Every state change traces back to a specific event, making debugging and logging predictable. It remains the enterprise standard for Flutter state management, particularly in regulated industries requiring strict audit trails.

## When to Use BLoC (not Cubit)

| BLoC is the right choice when |
|---|
| Complex event-driven flows with multiple triggers |
| Need event transformers (debouncing, throttling, droppable) |
| Multiple events trigger different logic paths |
| Traceability of every state change matters (audit trails) |
| Regulated industries (banking, healthcare, logistics) |
| Large teams (10+ developers) requiring strict conventions |

## Core Flow

```
User Action → Event → BLoC → State → UI Rebuild
```

- **Events**: immutable objects describing what happened (user tapped, data loaded, etc.)
- **States**: immutable objects describing the current UI condition
- **BLoC**: receives events, runs logic, emits new states

## Event Design

Events should be past-tense or request-style names describing what the user did. Prefer Dart 3 sealed classes for exhaustive pattern matching:

```dart
// Dart 3 sealed classes (recommended)
sealed class LoginEvent {
  const LoginEvent();
}

class LoginSendOtpRequested extends LoginEvent {
  final String phoneNumber;
  const LoginSendOtpRequested(this.phoneNumber);
}

class LoginOtpVerified extends LoginEvent {
  final String otp;
  const LoginOtpVerified(this.otp);
}

// Alternative: Equatable-based (for projects not yet on Dart 3)
class LoginSendOtpRequested extends BaseEvent {
  final String phoneNumber;
  const LoginSendOtpRequested(this.phoneNumber);

  @override
  List<Object?> get props => [phoneNumber];
}
```

```dart
// Good — descriptive, one per user action
class LoginSendOtpRequested extends BaseEvent { ... }
class ProfilePhotoChanged extends BaseEvent { ... }
class CartItemRemoved extends BaseEvent { ... }

// Bad — vague, imperative
class LoadData extends BaseEvent { ... }
class ButtonClicked extends BaseEvent { ... }
```

Rules:
- Extend the project's base event class (which extends Equatable) or use sealed classes
- Override `props` for equality — include all fields that affect identity
- Keep events flat. Carry only the data needed to process the action
- One event class per distinct user action or system trigger
- Use `const` constructors where possible

## State Design

Use a **status enum + data fields** pattern with `copyWith` for state updates:

```dart
enum FeatureStatus { initial, loading, success, failure }

class FeatureState extends Equatable {
  final FeatureStatus status;
  final String? errorMessage;
  final SomeEntity? data;

  const FeatureState({
    this.status = FeatureStatus.initial,
    this.errorMessage,
    this.data,
  });

  FeatureState copyWith({
    FeatureStatus? status,
    String? errorMessage,
    SomeEntity? data,
  }) {
    return FeatureState(
      status: status ?? this.status,
      errorMessage: errorMessage ?? this.errorMessage,
      data: data ?? this.data,
    );
  }

  @override
  List<Object?> get props => [status, errorMessage, data];
}
```

For complex features with distinct UI states, consider sealed classes:

```dart
sealed class FeatureState extends Equatable {
  const FeatureState();
}

class FeatureInitial extends FeatureState {
  const FeatureInitial();
  @override
  List<Object?> get props => [];
}

class FeatureLoading extends FeatureState {
  const FeatureLoading();
  @override
  List<Object?> get props => [];
}

class FeatureLoaded extends FeatureState {
  final List<SomeEntity> items;
  const FeatureLoaded(this.items);
  @override
  List<Object?> get props => [items];
}

class FeatureError extends FeatureState {
  final String message;
  const FeatureError(this.message);
  @override
  List<Object?> get props => [message];
}
```

Rules:
- Extend the project's base state class (which extends Equatable) or use sealed classes
- Choose single state class + status enum for simple features, sealed classes for complex features with distinct UI states
- All fields should be nullable or have defaults so the initial state is trivial to construct
- Override `props` with every field that should trigger a rebuild when changed
- Always provide `copyWith` if using a single state class — avoids losing existing state data when emitting
- Use `const` constructors for all state classes

## BLoC Implementation

```dart
class FeatureBloc extends Bloc<FeatureEvent, FeatureState> {
  final SomeUseCase _someUseCase;

  FeatureBloc({required SomeUseCase someUseCase})
      : _someUseCase = someUseCase,
        super(const FeatureState()) {
    on<FeatureLoadRequested>(_onLoadRequested);
    on<FeatureItemSelected>(_onItemSelected);
    // Use event transformers for search/debounce scenarios
    on<FeatureSearchChanged>(
      _onSearchChanged,
      transformer: restartable(), // or debounce(Duration(milliseconds: 300))
    );
  }

  Future<void> _onLoadRequested(
    FeatureLoadRequested event,
    Emitter<FeatureState> emit,
  ) async {
    emit(state.copyWith(status: FeatureStatus.loading));

    final result = await _someUseCase.execute(event.param);

    // Option A: Pattern matching with sealed response classes
    switch (result) {
      case UseCaseSuccessResponse(:final data):
        emit(state.copyWith(status: FeatureStatus.success, data: data));
      case UseCaseServerError(:final message):
        emit(state.copyWith(status: FeatureStatus.failure, errorMessage: message));
      case UseCaseConnectionError():
        emit(state.copyWith(status: FeatureStatus.failure, errorMessage: 'No internet connection'));
      case UseCaseUnknownError():
        emit(state.copyWith(status: FeatureStatus.failure, errorMessage: 'Something went wrong'));
    }

    // Option B: Using Either (fold pattern)
    // result.fold(
    //   (failure) => emit(state.copyWith(status: FeatureStatus.failure, errorMessage: failure.message)),
    //   (data) => emit(state.copyWith(status: FeatureStatus.success, data: data)),
    // );
  }
}
```

## Event Transformers

Event transformers control how the BLoC processes concurrent events. This is a key advantage BLoC has over Cubit.

```dart
import 'package:bloc_concurrency/bloc_concurrency.dart';

class SearchBloc extends Bloc<SearchEvent, SearchState> {
  SearchBloc({required SearchUseCase searchUseCase})
      : _searchUseCase = searchUseCase,
        super(const SearchState()) {

    // Debounce: wait for user to stop typing before processing
    on<SearchQueryChanged>(
      _onQueryChanged,
      transformer: restartable(),
    );

    // Droppable: ignore new events while one is processing (e.g., pagination)
    on<SearchNextPageRequested>(
      _onNextPage,
      transformer: droppable(),
    );

    // Sequential: process events one at a time in order
    on<SearchFilterApplied>(
      _onFilterApplied,
      transformer: sequential(),
    );

    // Concurrent: process all events simultaneously (default behavior)
    on<SearchResultTapped>(
      _onResultTapped,
      transformer: concurrent(),
    );
  }
}
```

| Transformer | Behavior | Use case |
|---|---|---|
| `concurrent()` | Process all events simultaneously (default) | Independent actions |
| `sequential()` | Process one at a time, queue the rest | Ordered operations |
| `droppable()` | Ignore new events while processing | Pagination, form submit |
| `restartable()` | Cancel current, start new | Search-as-you-type |

Rules:
- Register all handlers in the constructor with `on<Event>`
- Use `EventTransformer` for debouncing, throttling, or sequential processing
- Inject dependencies (use cases) through the constructor as private final fields
- Never import or call repositories directly — always go through use cases
- Handle every response branch (success, server error, connection error, unknown)
- Use `state.copyWith()` instead of creating new state instances to preserve existing data
- Use Dart 3 `switch` expressions for exhaustive pattern matching on sealed response types
- UI controllers (TextEditingController, FocusNode, ScrollController) live in the BLoC
- Dispose controllers and cancel subscriptions in `close()`
- Avoid emitting state after the BLoC is closed — check `isClosed` in async callbacks outside `on<>` handlers

## Connecting BLoC to UI

### With Base Widget (project-specific)

If the project has a `BaseWidget<BlocType, StateType>` pattern:

```dart
class FeatureScreen extends BaseWidget<FeatureBloc, FeatureState> {
  @override
  FeatureBloc getBlocInstance() => createFeatureBloc();

  @override
  void onScreenReady(BuildContext context, FeatureBloc bloc) {
    bloc.add(const FeatureLoadRequested());
  }

  @override
  void listenState(BuildContext context, FeatureBloc bloc, FeatureState state) {
    if (state.status == FeatureStatus.failure && state.errorMessage != null) {
      // Show snackbar or dialog
    }
  }

  @override
  Widget getBuildWidget(BuildContext context, FeatureBloc bloc, FeatureState state, bool isLandscape) {
    return switch (state.status) {
      FeatureStatus.loading => const Center(child: CircularProgressIndicator()),
      FeatureStatus.success => _buildContent(state.data!),
      FeatureStatus.failure => _buildError(state.errorMessage),
      _ => const SizedBox.shrink(),
    };
  }
}
```

### Without Base Widget (standard flutter_bloc)

```dart
class FeatureScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => createFeatureBloc()..add(const FeatureLoadRequested()),
      child: BlocConsumer<FeatureBloc, FeatureState>(
        listener: (context, state) {
          // Side effects: navigation, snackbars, dialogs
          // BLoC 9.0+ automatically verifies context.mounted before callbacks
        },
        builder: (context, state) {
          // Return widget tree based on state
        },
      ),
    );
  }
## The Golden Rules of Context Lookups

From the official [bloclibrary.dev](https://bloclibrary.dev/) documentation:

| Method | Where to call | When to use | Rule / Warning |
|---|---|---|---|
| `context.read<T>()` | Callbacks, event handlers | Access BLoC to dispatch events: `context.read<CartBloc>().add(...)` | **NEVER** call inside `build()`. It does not listen to state changes and can lead to bugs. |
| `context.watch<T>()` | Widget `build()` method | Rebuild widget when BLoC state changes | **NEVER** call inside event handlers/callbacks (e.g. `onPressed`). |
| `context.select<T, R>()` | Widget `build()` method | Rebuild widget ONLY when a specific property `R` changes | Great for high-frequency state objects to minimize rebuilds. |

### Choosing the Right Flutter BLoC Widget

- **`BlocBuilder<B, S>`**: Use when you only need to render widgets based on state. Pure UI rendering.
- **`BlocListener<B, S>`**: Use for 1-time side effects (navigation, snackbar, dialog, analytics). Guaranteed to execute once per state change.
- **`BlocConsumer<B, S>`**: Use when a single widget needs BOTH rendering (`builder`) and side effects (`listener`).
- **`BlocSelector<B, S, T>`**: Use when a widget only cares about a single field of state `T`. Prevents unnecessary rebuilds when other fields change.

## Global Monitoring with BlocObserver

Create a custom `BlocObserver` to centralize logging, analytics, and crash reporting (e.g. Sentry or Firebase Crashlytics) across all BLoCs and Cubits:

```dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:flutter/foundation.dart';

class AppBlocObserver extends BlocObserver {
  @override
  void onCreate(BlocBase bloc) {
    super.onCreate(bloc);
    debugPrint('🟢 onCreate -- ${bloc.runtimeType}');
  }

  @override
  void onEvent(Bloc bloc, Object? event) {
    super.onEvent(bloc, event);
    debugPrint('📥 onEvent -- ${bloc.runtimeType}, Event: $event');
  }

  @override
  void onChange(BlocBase bloc, Change change) {
    super.onChange(bloc, change);
    debugPrint('🔄 onChange -- ${bloc.runtimeType}, Current: ${change.currentState} -> Next: ${change.nextState}');
  }

  @override
  void onTransition(Bloc bloc, Transition transition) {
    super.onTransition(bloc, transition);
    debugPrint('⚡ onTransition -- ${bloc.runtimeType}, Event: ${transition.event}');
  }

  @override
  void onError(BlocBase bloc, Object error, StackTrace stackTrace) {
    debugPrint('❌ onError -- ${bloc.runtimeType}, Error: $error');
    // Forward to crash reporting service:
    // FirebaseCrashlytics.instance.recordError(error, stackTrace);
    super.onError(bloc, error, stackTrace);
  }

  @override
  void onClose(BlocBase bloc) {
    super.onClose(bloc);
    debugPrint('🔴 onClose -- ${bloc.runtimeType}');
  }
}
```

Initialize in `main.dart` before `runApp()`:

```dart
void main() {
  WidgetsFlutterBinding.ensureInitialized();
  Bloc.observer = AppBlocObserver();
  runApp(const MyApp());
}
```

## State Persistence with HydratedBloc

For state that must survive app restarts (e.g. theme preference, user auth token, cached cart):

```yaml
dependencies:
  hydrated_bloc: ^9.1.5
  path_provider: ^2.1.5
```

Initialize storage in `main.dart`:
```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  HydratedBloc.storage = await HydratedStorage.build(
    storageDirectory: HydratedStorageDirectory((await getApplicationDocumentsDirectory()).path),
  );
  runApp(const MyApp());
}
```

Extend `HydratedBloc` instead of `Bloc`:

```dart
class ThemeBloc extends HydratedBloc<ThemeEvent, ThemeMode> {
  ThemeBloc() : super(ThemeMode.system) {
    on<ThemeToggled>((event, emit) {
      emit(state == ThemeMode.dark ? ThemeMode.light : ThemeMode.dark);
    });
  }

  @override
  ThemeMode? fromJson(Map<String, dynamic> json) {
    return ThemeMode.values[json['theme_mode'] as int];
  }

  @override
  Map<String, dynamic>? toJson(ThemeMode state) {
    return {'theme_mode': state.index};
  }
}
```

## BLoC-to-BLoC Communication

When one BLoC needs to react to another:
1. **Preferred: Shared Reactive Repository**
   - Inject a repository that exposes a `Stream<T>` into both BLoCs.
   - BLoC A updates the repository; BLoC B listens to the repository stream and adds events to itself.
2. **Alternative: Widget Tree `BlocListener`**
   - Use `BlocListener<BlocA, StateA>` to dispatch an event to `BlocB`:
     ```dart
     BlocListener<AuthBloc, AuthState>(
       listener: (context, state) {
         if (state is AuthUnauthenticated) {
           context.read<CartBloc>().add(const CartCleared());
         }
       },
       child: child,
     )
     ```
3. **NEVER inject one BLoC directly into another BLoC's constructor**. This couples them tightly and makes testing difficult.

## Feature Module Structure (BLoC)

```
modules/<feature_name>/
├── presentation/
│   ├── view/
│   │   └── <feature>_screen.dart
│   ├── bloc/
│   │   ├── <feature>_bloc.dart
│   │   ├── <feature>_event.dart
│   │   └── <feature>_state.dart
│   └── widgets/
│       └── <feature>_widget.dart
└── <feature>_dependencies.dart
```

## Dependency Wiring

```dart
FeatureBloc createFeatureBloc() {
  final repository = FeatureRepositoryImpl(
    remoteDataSource: FeatureRemoteDataSourceImpl(),
    mapper: FeatureMapper(),
  );
  return FeatureBloc(
    loadUseCase: LoadItemsUseCase(repository: repository),
    deleteUseCase: DeleteItemUseCase(repository: repository),
  );
}
```

## Testing BLoCs

```dart
blocTest<FeatureBloc, FeatureState>(
  'emits [loading, success] when load succeeds',
  build: () {
    when(() => mockUseCase.execute(any())).thenAnswer(
      (_) async => UseCaseSuccessResponse(data: testEntity),
    );
    return FeatureBloc(someUseCase: mockUseCase);
  },
  act: (bloc) => bloc.add(const FeatureLoadRequested()),
  expect: () => [
    const FeatureState(status: FeatureStatus.loading),
    FeatureState(status: FeatureStatus.success, data: testEntity),
  ],
);
```

## Naming Conventions

| Item | Pattern | Example |
|---|---|---|
| BLoC class | `PascalCaseBloc` | `LoginBloc` |
| Event base | `PascalCaseEvent` | `LoginEvent` |
| Event subclass | `PascalCase` (past-tense/request) | `LoginSendOtpRequested` |
| State class | `PascalCaseState` | `LoginState` |
| BLoC file | `<feature>_bloc.dart` | `login_bloc.dart` |
| Event file | `<feature>_event.dart` | `login_event.dart` |
| State file | `<feature>_state.dart` | `login_state.dart` |

## Common Pitfalls

- **Emitting after close**: check `isClosed` before emitting in async callbacks outside `on<>` handlers. Inside `on<>` handlers, `bloc_test` handles this automatically
- **Missing Equatable props**: if a field is not in `props`, state changes for that field won't trigger rebuilds
- **Fat BLoCs**: if a BLoC has 10+ event handlers, consider splitting into multiple BLoCs
- **Business logic in UI**: if you're writing `if/else` chains in `builder`, that logic likely belongs in the BLoC
- **Forgetting to handle all response types**: always handle success, server error, connection error, and unknown
- **Creating new state instead of copyWith**: `emit(FeatureState(status: loading))` loses all existing data. Use `emit(state.copyWith(status: loading))` instead
- **Not using event transformers**: search fields without debouncing fire excessive API calls. Use `restartable()` or custom debounce transformers
- **Nested BlocBuilder/BlocListener**: use `BlocConsumer` when you need both, or `MultiBlocListener` for multiple listeners
- **Accessing BLoC after disposal**: storing a BLoC reference and using it after the widget is disposed causes errors. Always access via `context.read<T>()` in callbacks
- **Using Cubit when BLoC is needed**: if you need event traceability, debouncing, or complex event flows, use BLoC — not Cubit
