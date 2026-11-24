---
name: flutter-cubit
description: Implements Cubit pattern for simpler state management without events. Use when a feature has straightforward state changes driven by method calls rather than discrete events.
---

# Flutter Cubit Pattern

> **What:** Patterns for implementing Cubit state management using `flutter_bloc` — direct method calls, states, and UI integration.
> **When to use:** Simple state changes, toggles, UI preferences, or rapid prototyping where event traceability is not needed.
> **Depends on:** Works alongside `flutter-clean-architecture` and `flutter-di-pattern`.
> **See also:** For complex event-driven flows with transformers and audit trails, see `flutter-bloc-pattern`.

Cubit is a lightweight subset of BLoC — it uses direct method calls instead of events, reducing boilerplate while keeping the same state management foundation. A common pattern is to use Cubit for simple synchronous state and reserve full BLoC for complex async flows requiring event traceability.

## When to Use Cubit (not BLoC)

| Cubit is the right choice when |
|---|
| Simple state changes with direct method calls |
| Few, straightforward actions (toggle, increment, select) |
| No need for event transformers or debouncing |
| Lightweight screens or components |
| Rapid prototyping where speed trumps ceremony |
| UI state management (theme, locale, navigation index) |

## Core Flow

```
User Action → Method Call → Cubit → State → UI Rebuild
```

- **Methods**: public functions on the Cubit that the UI calls directly
- **States**: immutable objects describing the current UI condition
- **Cubit**: receives method calls, runs logic, emits new states

The key difference from BLoC: no Event classes. The UI calls methods directly on the Cubit instead of dispatching event objects.

## State Design

States work identically to BLoC. Use a **status enum + data fields** pattern with `copyWith`:

```dart
enum FeatureStatus { initial, loading, success, failure }

class FeatureState extends Equatable {
  final FeatureStatus status;
  final String? errorMessage;
  final List<SomeEntity> items;

  const FeatureState({
    this.status = FeatureStatus.initial,
    this.errorMessage,
    this.items = const [],
  });

  FeatureState copyWith({
    FeatureStatus? status,
    String? errorMessage,
    List<SomeEntity>? items,
  }) {
    return FeatureState(
      status: status ?? this.status,
      errorMessage: errorMessage ?? this.errorMessage,
      items: items ?? this.items,
    );
  }

  @override
  List<Object?> get props => [status, errorMessage, items];
}
```

For simple Cubits with few distinct states, sealed classes work well:

```dart
sealed class ThemeState extends Equatable {
  const ThemeState();
}

class ThemeLight extends ThemeState {
  const ThemeLight();
  @override
  List<Object?> get props => [];
}

class ThemeDark extends ThemeState {
  const ThemeDark();
  @override
  List<Object?> get props => [];
}
```

Rules:
- Same state design principles as BLoC — Equatable, `copyWith`, `const` constructors
- All fields should be nullable or have defaults
- Override `props` with every field that should trigger a rebuild

## Cubit Implementation

### Simple Cubit (synchronous)

```dart
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);

  void increment() => emit(state + 1);
  void decrement() => emit(state - 1);
  void reset() => emit(0);
}
```

### Cubit with Use Case (async)

```dart
class FeatureCubit extends Cubit<FeatureState> {
  final LoadItemsUseCase _loadItemsUseCase;
  final DeleteItemUseCase _deleteItemUseCase;

  FeatureCubit({
    required LoadItemsUseCase loadItemsUseCase,
    required DeleteItemUseCase deleteItemUseCase,
  }) : _loadItemsUseCase = loadItemsUseCase,
       _deleteItemUseCase = deleteItemUseCase,
       super(const FeatureState());

  Future<void> loadItems({int page = 1, int limit = 20}) async {
    emit(state.copyWith(status: FeatureStatus.loading));

    final result = await _loadItemsUseCase.execute(page: page, limit: limit);

    // Option A: Pattern matching with sealed response classes
    switch (result) {
      case UseCaseSuccessResponse(:final data):
        emit(state.copyWith(status: FeatureStatus.success, items: data));
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
    //   (data) => emit(state.copyWith(status: FeatureStatus.success, items: data)),
    // );
  }

  Future<void> deleteItem(int id) async {
    final result = await _deleteItemUseCase.execute(id: id);
    switch (result) {
      case UseCaseSuccessResponse():
        final updated = state.items.where((item) => item.id != id).toList();
        emit(state.copyWith(items: updated));
      case UseCaseServerError(:final message):
        emit(state.copyWith(errorMessage: message));
      default:
        emit(state.copyWith(errorMessage: 'Failed to delete item'));
    }
  }
}
```

### App-Level Cubit (theme, locale, connectivity)

```dart
class ThemeCubit extends Cubit<ThemeMode> {
  final PreferenceManager _prefs;

  ThemeCubit({required PreferenceManager prefs})
      : _prefs = prefs,
        super(ThemeMode.system);

  void setTheme(ThemeMode mode) {
    _prefs.setThemeMode(mode);
    emit(mode);
  }

  void toggleTheme() {
    final next = state == ThemeMode.dark ? ThemeMode.light : ThemeMode.dark;
    setTheme(next);
  }
}
```

```dart
class LocaleCubit extends Cubit<Locale> {
  final PreferenceManager _prefs;

  LocaleCubit({required PreferenceManager prefs})
      : _prefs = prefs,
        super(const Locale('en'));

  void changeLocale(Locale locale) {
    _prefs.setLanguageCode(locale.languageCode);
    emit(locale);
  }
}
```

Rules:
- Methods are public and named descriptively (`loadItems`, `deleteItem`, `toggleTheme`)
- Inject dependencies (use cases) through the constructor as private final fields
- Never import or call repositories directly — always go through use cases
- Handle every response branch (success, server error, connection error, unknown)
- Use `state.copyWith()` to preserve existing data when emitting
- Check `isClosed` before emitting in async callbacks if the Cubit might be disposed
- Dispose controllers and cancel subscriptions in `close()`

## Connecting Cubit to UI

### Standard flutter_bloc

```dart
class FeatureScreen extends StatelessWidget {
  const FeatureScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => createFeatureCubit()..loadItems(),
      child: BlocConsumer<FeatureCubit, FeatureState>(
        listener: (context, state) {
          if (state.status == FeatureStatus.failure && state.errorMessage != null) {
            ScaffoldMessenger.of(context).showSnackBar(
              SnackBar(content: Text(state.errorMessage!)),
            );
          }
          // BLoC 9.0+ automatically verifies context.mounted before callbacks
        },
        builder: (context, state) {
          return switch (state.status) {
            FeatureStatus.loading => const Center(child: CircularProgressIndicator()),
            FeatureStatus.success => _buildList(context, state.items),
            FeatureStatus.failure => _buildError(state.errorMessage),
            _ => const SizedBox.shrink(),
          };
        },
      ),
    );
  }

  Widget _buildList(BuildContext context, List<SomeEntity> items) {
    return ListView.builder(
      itemCount: items.length,
      itemBuilder: (context, index) {
        final item = items[index];
        return ListTile(
          title: Text(item.name),
          trailing: IconButton(
            icon: const Icon(Icons.delete),
            onPressed: () => context.read<FeatureCubit>().deleteItem(item.id),
          ),
        );
      },
    );
  }
}
```

### Calling Cubit Methods from UI

```dart
// One-time read (in callbacks, onPressed, etc.)
context.read<FeatureCubit>().loadItems();
context.read<ThemeCubit>().toggleTheme();

// Reactive rebuild (in build method)
final state = context.watch<FeatureCubit>().state;
```

### Using BlocSelector for Granular Rebuilds

```dart
BlocSelector<FeatureCubit, FeatureState, bool>(
  selector: (state) => state.status == FeatureStatus.loading,
  builder: (context, isLoading) {
    return isLoading ? const LinearProgressIndicator() : const SizedBox.shrink();
  },
)
```

## Feature Module Structure (Cubit)

```
modules/<feature_name>/
├── presentation/
│   ├── view/
│   │   └── <feature>_screen.dart
│   ├── cubit/
│   │   ├── <feature>_cubit.dart
│   │   └── <feature>_state.dart       # No event file needed
│   └── widgets/
│       └── <feature>_widget.dart
└── <feature>_dependencies.dart
```

Note: compared to BLoC, there is no `<feature>_event.dart` file — that's the boilerplate Cubit eliminates.

## Dependency Wiring

```dart
FeatureCubit createFeatureCubit() {
  final repository = FeatureRepositoryImpl(
    remoteDataSource: FeatureRemoteDataSourceImpl(),
    mapper: FeatureMapper(),
  );
  return FeatureCubit(
    loadItemsUseCase: LoadItemsUseCase(repository: repository),
    deleteItemUseCase: DeleteItemUseCase(repository: repository),
  );
}
```

## Testing Cubits

Cubits use the same `blocTest` helper as BLoC, but you call methods directly instead of adding events:

```dart
import 'package:bloc_test/bloc_test.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

class MockLoadItemsUseCase extends Mock implements LoadItemsUseCase {}

void main() {
  late MockLoadItemsUseCase mockUseCase;

  setUp(() {
    mockUseCase = MockLoadItemsUseCase();
  });

  group('FeatureCubit', () {
    test('initial state is correct', () {
      final cubit = FeatureCubit(loadItemsUseCase: mockUseCase);
      expect(cubit.state.status, FeatureStatus.initial);
      cubit.close();
    });

    blocTest<FeatureCubit, FeatureState>(
      'emits [loading, success] when loadItems succeeds',
      build: () {
        when(() => mockUseCase.execute(page: any(named: 'page'), limit: any(named: 'limit')))
            .thenAnswer((_) async => UseCaseSuccessResponse(data: [testItem]));
        return FeatureCubit(loadItemsUseCase: mockUseCase);
      },
      act: (cubit) => cubit.loadItems(),  // Direct method call, not add(event)
      expect: () => [
        isA<FeatureState>().having((s) => s.status, 'status', FeatureStatus.loading),
        isA<FeatureState>()
            .having((s) => s.status, 'status', FeatureStatus.success)
            .having((s) => s.items.length, 'items length', 1),
      ],
    );

    blocTest<FeatureCubit, FeatureState>(
      'emits [loading, failure] when loadItems fails',
      build: () {
        when(() => mockUseCase.execute(page: any(named: 'page'), limit: any(named: 'limit')))
            .thenAnswer((_) async => UseCaseServerError(message: 'Server error'));
        return FeatureCubit(loadItemsUseCase: mockUseCase);
      },
      act: (cubit) => cubit.loadItems(),
      expect: () => [
        isA<FeatureState>().having((s) => s.status, 'status', FeatureStatus.loading),
        isA<FeatureState>()
            .having((s) => s.status, 'status', FeatureStatus.failure)
            .having((s) => s.errorMessage, 'error', 'Server error'),
      ],
    );
  });
}
```

### Testing Simple Cubits

```dart
blocTest<CounterCubit, int>(
  'emits [1] when increment is called',
  build: () => CounterCubit(),
  act: (cubit) => cubit.increment(),
  expect: () => [1],
);

blocTest<CounterCubit, int>(
  'emits [1, 2, 3] when increment is called 3 times',
  build: () => CounterCubit(),
  act: (cubit) {
    cubit.increment();
    cubit.increment();
    cubit.increment();
  },
  expect: () => [1, 2, 3],
);
```

## Naming Conventions

| Item | Pattern | Example |
|---|---|---|
| Cubit class | `PascalCaseCubit` | `LoginCubit` |
| State class | `PascalCaseState` | `LoginState` |
| Cubit file | `<feature>_cubit.dart` | `login_cubit.dart` |
| State file | `<feature>_state.dart` | `login_state.dart` |
| Methods | `camelCase` (verb-first) | `loadItems()`, `deleteItem()`, `toggleTheme()` |

## Cubit-to-Cubit Communication

Same rules as BLoC-to-BLoC:
- Use `BlocListener` in the widget tree to bridge between Cubits
- Or pass a stream/callback from a shared service
- Never inject one Cubit into another directly

## Migrating from Cubit to BLoC

If a Cubit grows complex enough to need event transformers or audit trails, migrate to BLoC:

1. Create an event class for each public method on the Cubit
2. Replace the Cubit class with a Bloc class
3. Move method bodies into `on<Event>` handlers
4. Update UI to dispatch events instead of calling methods
5. Add event transformers where needed

```dart
// Before (Cubit)
class SearchCubit extends Cubit<SearchState> {
  void search(String query) async { ... }
}
// UI: context.read<SearchCubit>().search(query);

// After (BLoC) — now with debouncing
class SearchBloc extends Bloc<SearchEvent, SearchState> {
  SearchBloc() : super(const SearchState()) {
    on<SearchQueryChanged>(_onQueryChanged, transformer: restartable());
  }
}
// UI: context.read<SearchBloc>().add(SearchQueryChanged(query));
```

## Common Pitfalls

- **Emitting after close**: check `isClosed` before emitting in async callbacks
- **Missing Equatable props**: if a field is not in `props`, state changes for that field won't trigger rebuilds
- **Fat Cubits**: if a Cubit has 10+ methods, consider splitting into multiple Cubits
- **Business logic in UI**: if you're writing `if/else` chains in `builder`, that logic belongs in the Cubit
- **Creating new state instead of copyWith**: `emit(FeatureState(status: loading))` loses all existing data. Use `emit(state.copyWith(status: loading))`
- **Using Cubit when BLoC is needed**: if you need debouncing, event traceability, or complex concurrent event handling, switch to BLoC
- **Nested BlocBuilder/BlocListener**: use `BlocConsumer` when you need both
- **Calling methods in `build()`**: never call Cubit methods inside `build()`. Use `create` callback or `onPressed` handlers
- **Not handling all response types**: always handle success, server error, connection error, and unknown — even in simple Cubits
