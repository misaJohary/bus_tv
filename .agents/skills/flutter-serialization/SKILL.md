---
name: flutter-serialization
description: Generates JSON serialization and deserialization code with json_serializable. Use when creating DTOs, custom JsonConverters, or serializing complex models and enums.
---

# Flutter JSON Serialization Pattern

> **What:** Complete guide for generating JSON serialization code using `json_serializable` and `json_annotation` in Dart & Flutter.
> **When to use:** Creating Data Transfer Objects (DTOs), mapping REST API responses, handling custom data types, enums, and nested payloads.
> **Depends on:** Pairs with `flutter-freezed`, `flutter-retrofit`, and `flutter-repository`.

`json_serializable` automates the generation of `fromJson` and `toJson` methods, preventing manual mapping typos, null-safety bugs, and boilerplate.

---

## 1. Setup & Dependencies

Add to `pubspec.yaml`:

```yaml
dependencies:
  json_annotation: ^4.9.0

dev_dependencies:
  build_runner: ^2.4.14
  json_serializable: ^6.9.4
```

---

## 2. Standard DTO Implementation

A standard DTO with `json_serializable`:

```dart
import 'package:json_annotation/json_annotation.dart';

part 'user_dto.g.dart';

@JsonSerializable(explicitToJson: true)
class UserDto {
  final String id;
  
  @JsonKey(name: 'full_name')
  final String fullName;
  
  final String email;

  @JsonKey(name: 'is_active', defaultValue: true)
  final bool isActive;

  @JsonKey(name: 'created_at')
  final DateTime createdAt;

  const UserDto({
    required this.id,
    required this.fullName,
    required this.email,
    required this.isActive,
    required this.createdAt,
  });

  factory UserDto.fromJson(Map<String, dynamic> json) =>
      _$UserDtoFromJson(json);

  Map<String, dynamic> toJson() => _$UserDtoToJson(this);
}
```

---

## 3. Global Configuration via `build.yaml`

Avoid repeating `@JsonSerializable(explicitToJson: true, fieldRename: FieldRename.snake)` on every file by adding a root `build.yaml`:

```yaml
targets:
  $default:
    builders:
      json_serializable:
        options:
          # Automatically map camelCaseDart to snake_case_json
          field_rename: snake
          # Ensure nested models also serialize to json
          explicit_to_json: true
          # Exclude null values from generated toJson() output
          include_if_null: false
```

With `build.yaml` configured, you simply write `@JsonSerializable()` without repetitive parameters.

---

## 4. Useful `@JsonKey` Attributes

| Parameter | Purpose | Example |
|---|---|---|
| `name` | Custom JSON key name | `@JsonKey(name: 'user_phone_number')` |
| `defaultValue` | Fallback value if null or missing | `@JsonKey(defaultValue: false)` |
| `includeIfNull` | Don't write key to JSON if value is null | `@JsonKey(includeIfNull: false)` |
| `readValue` | Read from custom or nested key path | `@JsonKey(readValue: _readNestedTitle)` |
| `fromJson` / `toJson` | Inline custom transform functions | `@JsonKey(fromJson: _stringToInt)` |

### Reading Nested JSON Values with `readValue`

If an API returns a nested value like `{"author": {"profile": {"name": "Alice"}}}`:

```dart
Object? _readAuthorName(Map json, String key) {
  return json['author']?['profile']?['name'];
}

@JsonSerializable()
class PostDto {
  @JsonKey(readValue: _readAuthorName)
  final String authorName;

  PostDto({required this.authorName});

  factory PostDto.fromJson(Map<String, dynamic> json) => _$PostDtoFromJson(json);
  Map<String, dynamic> toJson() => _$PostDtoToJson(this);
}
```

---

## 5. Custom `JsonConverter` Implementations

Use `JsonConverter<T, S>` for reusable transformations across multiple models.

### Epoch Milliseconds to DateTime Converter

```dart
import 'package:json_annotation/json_annotation.dart';

class EpochDateTimeConverter implements JsonConverter<DateTime, int> {
  const EpochDateTimeConverter();

  @override
  DateTime fromJson(int json) =>
      DateTime.fromMillisecondsSinceEpoch(json, isUtc: true);

  @override
  int toJson(DateTime object) => object.millisecondsSinceEpoch;
}
```

### Safe Numeric String to Double Converter

Backend APIs sometimes return `"12.50"` or `12.50` inconsistently:

```dart
class SafeDoubleConverter implements JsonConverter<double, dynamic> {
  const SafeDoubleConverter();

  @override
  double fromJson(dynamic json) {
    if (json is num) return json.toDouble();
    if (json is String) return double.tryParse(json) ?? 0.0;
    return 0.0;
  }

  @override
  dynamic toJson(double object) => object;
}
```

### Applying Converters:

```dart
@JsonSerializable()
class TransactionDto {
  final String id;

  @EpochDateTimeConverter()
  final DateTime timestamp;

  @SafeDoubleConverter()
  final double amount;

  const TransactionDto({
    required this.id,
    required this.timestamp,
    required this.amount,
  });

  factory TransactionDto.fromJson(Map<String, dynamic> json) =>
      _$TransactionDtoFromJson(json);
  Map<String, dynamic> toJson() => _$TransactionDtoToJson(this);
}
```

---

## 6. Enum Serialization

Annotate enums using `@JsonEnum`:

```dart
import 'package:json_annotation/json_annotation.dart';

@JsonEnum(alwaysCreate: true)
enum OrderStatus {
  @JsonValue('pending')
  pending,
  
  @JsonValue('in_transit')
  inTransit,
  
  @JsonValue('delivered')
  delivered,
  
  @JsonValue('cancelled')
  cancelled,
  
  // Fallback for unknown future enum values from backend
  @JsonValue('unknown')
  unknown,
}

@JsonSerializable()
class OrderDto {
  @JsonKey(unknownEnumValue: OrderStatus.unknown)
  final OrderStatus status;

  OrderDto({required this.status});

  factory OrderDto.fromJson(Map<String, dynamic> json) =>
      _$OrderDtoFromJson(json);
  Map<String, dynamic> toJson() => _$OrderDtoToJson(this);
}
```

---

## 7. Generic Response Wrappers (`genericArgumentFactories`)

For standardized API response envelopes (e.g. `{"status": 200, "data": ...}`):

```dart
import 'package:json_annotation/json_annotation.dart';

part 'api_response.g.dart';

@JsonSerializable(genericArgumentFactories: true)
class ApiResponse<T> {
  final bool success;
  final String? message;
  final T? data;

  const ApiResponse({
    required this.success,
    this.message,
    this.data,
  });

  factory ApiResponse.fromJson(
    Map<String, dynamic> json,
    T Function(Object? json) fromJsonT,
  ) =>
      _$ApiResponseFromJson(json, fromJsonT);

  Map<String, dynamic> toJson(Object? Function(T value) toJsonT) =>
      _$ApiResponseToJson(this, toJsonT);
}
```

Usage with Retrofit:
```dart
@GET('/profile')
Future<ApiResponse<UserDto>> getProfile();

@GET('/items')
Future<ApiResponse<List<ItemDto>>> getItems();
```

---

## 8. Pairing `json_serializable` with `freezed`

When using Freezed, both generators work together seamlessly:

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'item.freezed.dart';
part 'item.g.dart';

@freezed
class Item with _$Item {
  const factory Item({
    required int id,
    @JsonKey(name: 'display_name') required String name,
    @Default(0.0) double price,
  }) = _Item;

  factory Item.fromJson(Map<String, dynamic> json) => _$ItemFromJson(json);
}
```

---

## 9. Best Practices Checklist

1. **Always set `explicitToJson: true`**: Without this, nested objects will not be converted to JSON maps when calling `toJson()`.
2. **Handle unknown enum values**: Always supply `unknownEnumValue` so unexpected backend values don't crash JSON parsing.
3. **Use `includeIfNull: false`**: Avoid sending unnecessary `null` values over the network.
4. **Don't put business logic in DTOs**: Keep DTOs strictly for data transfer and mapping. Convert DTOs to Domain Entities using separate mappers.
5. **Code generation command**:
   ```bash
   dart run build_runner build --delete-conflicting-outputs
   ```
