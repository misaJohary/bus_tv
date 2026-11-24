---
name: flutter-freezed
description: Implements immutable models, unions, and sealed classes using Freezed. Use when creating data classes, DTOs, domain entities, or union states.
---

# Flutter Freezed Pattern

> **What:** Guide for using `freezed` to generate immutable data classes, union types, pattern matching, and deep copy capabilities in Dart & Flutter.
> **When to use:** Creating domain entities, API request/response DTOs, complex BLoC states/events, or any immutable data structure.
> **Depends on:** Works alongside `flutter-serialization`, `flutter-bloc`, and `flutter-clean-architecture`.

`freezed` is a code generator for data-classes, unions, and pattern-matching. It eliminates boilerplate by generating `operator ==`, `hashCode`, `toString()`, `copyWith`, and serialization logic.

---

## 1. Setup & Dependencies

Add the necessary dependencies to `pubspec.yaml`:

```yaml
dependencies:
  freezed_annotation: ^2.4.4
  json_annotation: ^4.9.0

dev_dependencies:
  build_runner: ^2.4.14
  freezed: ^2.5.7
  json_serializable: ^6.9.4
```

> **Note:** For Dart 3 projects, Freezed integrates seamlessly with Dart 3 class modifiers (`sealed`, `final`) and pattern matching.

---

## 2. Basic Data Class

To create an immutable model:
1. Define a `part '<filename>.freezed.dart';`
2. Add `@freezed` annotation
3. Use `with _$<ClassName>`
4. Provide a `const factory` constructor

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user_profile.freezed.dart';

@freezed
class UserProfile with _$UserProfile {
  const factory UserProfile({
    required String id,
    required String username,
    required String email,
    @Default(false) bool isVerified,
    @Default([]) List<String> tags,
    DateTime? lastActiveAt,
  }) = _UserProfile;
}
```

### Adding Custom Methods or Getters

By default, Freezed does not allow adding custom methods or getters to the class unless you define an empty private `const` constructor:

```dart
@freezed
class UserProfile with _$UserProfile {
  // Required private constructor for custom getters/methods
  const UserProfile._();

  const factory UserProfile({
    required String id,
    required String firstName,
    required String lastName,
  }) = _UserProfile;

  // Custom getter
  String get fullName => '$firstName $lastName';

  // Custom method
  bool get hasInitials => firstName.isNotEmpty && lastName.isNotEmpty;
}
```

---

## 3. Cloning & Deep `copyWith`

Freezed provides type-safe `copyWith` out of the box, including deep copying of nested Freezed classes:

```dart
@freezed
class Address with _$Address {
  const factory Address({
    required String street,
    required String city,
  }) = _Address;
}

@freezed
class Company with _$Company {
  const factory Company({
    required String name,
    required Address address,
  }) = _Company;
}

void updateCompanyAddress(Company company) {
  // Deep copy syntax:
  final updated = company.copyWith.address(
    city: 'San Francisco',
  );
}
```

---

## 4. Freezed with JSON Serialization

Combine Freezed with `json_serializable` by adding `part '<filename>.g.dart';` and the `fromJson` factory:

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'product.freezed.dart';
part 'product.g.dart';

@freezed
class Product with _$Product {
  const factory Product({
    required int id,
    required String title,
    @JsonKey(name: 'unit_price') required double price,
    @Default(0) int stock,
    @JsonKey(includeIfNull: false) String? description,
  }) = _Product;

  factory Product.fromJson(Map<String, dynamic> json) =>
      _$ProductFromJson(json);
}
```

---

## 5. Union Types & Sealed States

Freezed is ideal for modeling states, events, and API results with multiple variations:

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'auth_state.freezed.dart';

@freezed
sealed class AuthState with _$AuthState {
  const factory AuthState.initial() = AuthInitial;
  const factory AuthState.loading() = AuthLoading;
  const factory AuthState.authenticated({
    required String token,
    required String userId,
  }) = AuthAuthenticated;
  const factory AuthState.failure({
    required String message,
    int? statusCode,
  }) = AuthFailure;
}
```

### Pattern Matching: Dart 3 `switch` vs Freezed `.when()`

With Dart 3 sealed classes, you can use native pattern matching (recommended):

```dart
Widget buildUI(AuthState state) {
  return switch (state) {
    AuthInitial() => const InitialView(),
    AuthLoading() => const LoadingSpinner(),
    AuthAuthenticated(:final token, :final userId) => HomeDashboard(token: token, userId: userId),
    AuthFailure(:final message) => ErrorBanner(message: message),
  };
}
```

Or use Freezed's built-in `.when()` / `.maybeWhen()` / `.map()`:

```dart
Widget buildUI(AuthState state) {
  return state.when(
    initial: () => const InitialView(),
    loading: () => const LoadingSpinner(),
    authenticated: (token, userId) => HomeDashboard(token: token, userId: userId),
    failure: (message, statusCode) => ErrorBanner(message: message),
  );
}
```

---

## 6. Collections & Immutability

By default, Freezed converts `List`, `Map`, and `Set` parameters into unmodifiable collections to guarantee immutability:

```dart
final profile = UserProfile(id: '1', username: 'alex', email: 'a@b.com', tags: ['flutter']);
profile.tags.add('dart'); // Throws UnsupportedError!
```

To modify a collection, create a new list and use `copyWith`:

```dart
final updated = profile.copyWith(
  tags: [...profile.tags, 'dart'],
);
```

If you explicitly want mutable collections inside your model (rarely recommended):
```dart
@Freezed(makeCollectionsUnmodifiable: false)
class MutableContainer with _$MutableContainer { ... }
```

---

## 7. Assertions & Validation

Add `@Assert` annotations to constructor declarations to enforce validation at runtime:

```dart
@freezed
class Discount with _$Discount {
  @Assert('percentage >= 0 && percentage <= 100', 'Percentage must be between 0 and 100')
  const factory Discount({
    required double percentage,
    required String code,
  }) = _Discount;
}
```

---

## 8. Code Generation Commands

Run the build runner to generate `.freezed.dart` and `.g.dart` files:

```bash
# Run one-time build (cleans conflicting outputs)
dart run build_runner build --delete-conflicting-outputs

# Run watch mode for development
dart run build_runner watch --delete-conflicting-outputs
```

---

## 9. Best Practices & Rules

1. **Always use `const` constructors** when instantiating Freezed classes: `const UserProfile(...)`.
2. **Add `const ClassName._();`** whenever you need custom getters, methods, or property overrides.
3. **Never mutate nested objects**; always utilize `copyWith` or deep copy syntax (`state.copyWith.user(name: '...')`).
4. **Use Dart 3 sealed unions** for BLoC states and responses to leverage exhaustive compile-time checking.
5. **Separate DTOs from Domain Entities**: Keep `@JsonKey` annotations in the data layer DTOs, and keep domain entities clean unless your team explicitly unifies them.
6. **Include both `part` declarations** when using JSON serialization:
   ```dart
   part '<file>.freezed.dart';
   part '<file>.g.dart';
   ```
