---
name: flutter-best-practices
description: Core Flutter architectural, performance, and UI best practices from docs.flutter.dev. Use for optimizing builds, memory management, responsiveness, and widget lifecycle.
---

# Flutter Official Best Practices & Performance Guide

> **What:** Comprehensive best practices, performance guidelines, and architectural rules directly derived from [docs.flutter.dev](https://docs.flutter.dev/).
> **When to use:** Designing widget trees, optimizing frame rates (60/120 FPS), eliminating jank, handling memory lifecycle, and structuring scalable Flutter apps.
> **Depends on:** Works alongside `flutter-architecture`, `flutter-bloc`, and `flutter-clean-architecture`.

---

## 1. Architectural Recommendations (docs.flutter.dev)

The Flutter team recommends a layered architecture following the **MVVM (Model-View-ViewModel)** pattern:

```
[UI Layer (Views)] ⇆ [ViewModel (BLoC / Cubit)] ⇆ [Domain (UseCases / Entities)] ⇆ [Data Layer (Repositories & DataSources)]
```

### Key Principles:
1. **Separation of Concerns:** UI widgets should ONLY render layout and receive user input. No business logic, networking, or direct database access inside widgets.
2. **Unidirectional Data Flow:** Data flows down from repositories to states to widgets; user actions dispatch events/methods upward.
3. **Single Source of Truth:** Repositories serve as the authoritative single source of truth for application data.
4. **Feature-First Structure:** Organize codebase by feature (e.g. `modules/auth`, `modules/chat`) rather than by technical layer type alone.

---

## 2. Widget Rebuild & Rendering Performance

### Prefer `const` Constructors Everywhere
`const` widgets are canonicalized at compile time and skipped during widget tree rebuilds:

```dart
// Good: skipped during parent rebuilds
const Center(child: Text('Hello'));

// Bad: recreated on every parent rebuild
Center(child: Text('Hello'));
```

Enable the linter rule in `analysis_options.yaml`:
```yaml
linter:
  rules:
    - prefer_const_constructors
    - prefer_const_literals_to_create_immutables
```

---

### Extract Separate Widgets Instead of Helper Methods

> [!IMPORTANT]
> Never use private helper methods like `Widget _buildItem()` to split large widget trees. Always extract them into a separate `StatelessWidget`.

**Why?**
- Helper methods execute every time the parent widget rebuilds.
- `StatelessWidget` has its own `BuildContext` and `Element`, enabling Flutter to short-circuit rebuilds when properties haven't changed.
- Enables `const` constructor usage.

```dart
// Bad: rebuilds whenever the parent builds, regardless of changes
Widget _buildUserInfo(String name) {
  return Text(name);
}

// Good: Flutter diffs properties and caches instances
class UserInfoView extends StatelessWidget {
  final String name;
  const UserInfoView({super.key, required this.name});

  @override
  Widget build(BuildContext context) {
    return Text(name);
  }
}
```

---

### Isolate Repaints with `RepaintBoundary`

When a widget repaints (e.g., an animated progress indicator, a custom canvas, or a video player), the entire render subtree can repaint by default. Wrap rapidly changing widgets in `RepaintBoundary` to create a separate display list:

```dart
RepaintBoundary(
  child: CircularProgressIndicator(),
)
```

---

### Use Flutter Keys Correctly

Keys preserve state when widgets move within the widget tree (e.g., reordering lists or removing items):

| Key Type | When to use |
|---|---|
| `ValueKey(item.id)` | Distinct item in a list with a stable identifier (e.g., DB ID) |
| `ObjectKey(item)` | Distinct object reference when no single unique ID exists |
| `UniqueKey()` | Force recreation of a widget and its state on next build |
| `GlobalKey()` | Access widget state across the tree (use sparingly; high overhead) |

```dart
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) {
    final item = items[index];
    return Dismissible(
      key: ValueKey(item.id), // Preserves identity on deletion
      child: ItemTile(item: item),
    );
  },
);
```

---

## 3. Scrolling & List Optimization

### Always Use `ListView.builder` for Dynamic Lists
Never use `SingleChildScrollView(child: Column(children: ...))` for long or dynamic lists. `Column` renders ALL items in memory immediately, causing memory spikes and dropped frames.

```dart
// Good: Lazy loads items as they enter the viewport
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) => ItemTile(item: items[index]),
);
```

### Provide `itemExtent` or `prototypeItem` for Fixed-Height Items
If list items have a fixed or uniform height, specify `itemExtent` or `prototypeItem`. This allows Flutter to calculate scroll offsets instantly without measuring each child:

```dart
// Huge scroll performance boost!
ListView.builder(
  itemCount: 1000,
  itemExtent: 72.0, // Fixed item height
  itemBuilder: (context, index) => ListTile(title: Text('Item $index')),
);

// Or if height is based on a prototype widget:
ListView.builder(
  itemCount: 1000,
  prototypeItem: const ListTile(title: Text('Prototype')),
  itemBuilder: (context, index) => ListTile(title: Text('Item $index')),
);
```

---

## 4. Memory Lifecycle & Resource Disposal

### Always Dispose State Controllers & Subscriptions

Any controller or subscription created in a `StatefulWidget` or BLoC MUST be disposed in `dispose()` or `close()`:

```dart
class _FormScreenState extends State<FormScreen> {
  late final TextEditingController _textController;
  late final FocusNode _focusNode;
  late final AnimationController _animController;
  StreamSubscription? _sub;

  @override
  void initState() {
    super.initState();
    _textController = TextEditingController();
    _focusNode = FocusNode();
  }

  @override
  void dispose() {
    _textController.dispose();
    _focusNode.dispose();
    _animController.dispose();
    _sub?.cancel();
    super.dispose(); // Always call super.dispose() last
  }
}
```

---

### Image Memory & Downsampling

High-resolution network images loaded into small widgets waste massive amounts of RAM and cause Out-Of-Memory (OOM) crashes:

```dart
// Bad: Decodes full 4K (3840x2160) image into memory for an 80x80 thumbnail
Image.network(avatarUrl, width: 80, height: 80);

// Good: Downsamples image during decoding to fit the display size
Image.network(
  avatarUrl,
  width: 80,
  height: 80,
  cacheWidth: (80 * MediaQuery.devicePixelRatioOf(context)).round(),
  cacheHeight: (80 * MediaQuery.devicePixelRatioOf(context)).round(),
  fit: BoxFit.cover,
);
```

### Precache Critical Assets
Pre-decode large hero images or background banners before navigation to eliminate flash of blank content:

```dart
@override
void didChangeDependencies() {
  super.didChangeDependencies();
  precacheImage(const AssetImage('assets/images/banner.png'), context);
}
```

---

## 5. Heavy Compute & Isolates

Dart is single-threaded. CPU-heavy tasks running on the main UI thread cause frame drops (jank):
- Parsing large JSON payloads (e.g., > 100 KB)
- Image manipulation or resizing
- Cryptography / token hashing
- Complex sorting or filtering on thousands of items

### Offload with `Isolate.run()` or `compute()`:

```dart
// Using Dart 3+ Isolate.run:
Future<List<UserDto>> parseLargeJson(String jsonString) async {
  return await Isolate.run(() {
    final decoded = jsonDecode(jsonString) as List;
    return decoded.map((e) => UserDto.fromJson(e as Map<String, dynamic>)).toList();
  });
}
```

---

## 6. Responsive & Adaptive UI Best Practices

### Avoid `MediaQuery.of(context)` for Size
Calling `MediaQuery.of(context)` causes the widget to rebuild whenever ANY `MediaQueryData` property changes (e.g. keyboard opens, screen brightness changes, accessibility setting toggled):

```dart
// Bad: Rebuilds on keyboard toggle or system theme change
final width = MediaQuery.of(context).size.width;

// Good: Only rebuilds when screen size actually changes
final size = MediaQuery.sizeOf(context);

// Or for padding (e.g. keyboard insets):
final insets = MediaQuery.viewInsetsOf(context);
```

### Use `LayoutBuilder` for Relative Container Sizing
`LayoutBuilder` gives you the constraints passed from the parent widget, allowing dynamic column or grid layout switching:

```dart
LayoutBuilder(
  builder: (context, constraints) {
    if (constraints.maxWidth > 600) {
      return TwoColumnView();
    } else {
      return SingleColumnView();
    }
  },
);
```

---

## 7. Spacing, Padding, Dimensions & Radius Tokens

> [!IMPORTANT]
> Never use arbitrary magic numbers for padding, margins, SizedBox gaps, or border radii (e.g. `EdgeInsets.all(14)`, `SizedBox(height: 10)`, `BorderRadius.circular(18)`, `Radius.circular(4)`). Always use design system tokens (e.g. `AppSpacing`).

### Key Rules:
1. **Grid Consistency (4pt/8pt):**
   - Use standard scalars (`AppSpacing.sm = 8.0`, `AppSpacing.md = 12.0`, `AppSpacing.base = 16.0`, `AppSpacing.lg = 20.0`, `AppSpacing.xl = 24.0`, etc.).
2. **Canonical Spacers:**
   - Always use `const` spacers: `AppSpacing.gapH8`, `AppSpacing.gapH12`, `AppSpacing.gapH16`, `AppSpacing.gapW8`, `AppSpacing.gapW16`, etc. instead of creating ad-hoc `SizedBox` widgets.
3. **Standard Insets:**
   - Use predefined insets: `AppSpacing.screenPadding`, `AppSpacing.screenPaddingWide`, `AppSpacing.cardPadding`, `AppSpacing.cardPaddingCompact`, `AppSpacing.allBase`, etc.
4. **Border Radii & Radius:**
   - Use centralized tokens: `AppSpacing.borderRadiusXs`, `AppSpacing.borderRadiusSm`, `AppSpacing.borderRadiusMd`, `AppSpacing.borderRadiusIntermediate`, `AppSpacing.borderRadiusBase`, `AppSpacing.borderRadiusLg`, `AppSpacing.borderRadiusXl`, `AppSpacing.borderRadiusFull`.
   - For specialized shapes (e.g. chat bubbles, bottom sheets), use `AppSpacing.bubbleBorderRadiusOutgoing`, `AppSpacing.bubbleBorderRadiusIncoming`, `AppSpacing.sheetTopBorderRadius`.

```dart
// Bad: Magic numbers recreate objects and break design rhythm
Padding(
  padding: const EdgeInsets.symmetric(horizontal: 14, vertical: 10),
  child: Column(
    children: [
      Container(
        decoration: BoxDecoration(
          borderRadius: BorderRadius.circular(18),
        ),
      ),
      const SizedBox(height: 10),
    ],
  ),
);

// Good: Uses AppSpacing design system tokens
Padding(
  padding: AppSpacing.screenPadding,
  child: Column(
    children: [
      Container(
        decoration: const BoxDecoration(
          borderRadius: AppSpacing.borderRadiusLg,
        ),
      ),
      AppSpacing.gapH12,
    ],
  ),
);
```

---

## 8. Internationalization (l10n) UI Strictness

> [!CRITICAL]
> Never write raw user-facing strings in UI code (widgets, dialogs, snackbars, tooltips, semantics labels, hints, empty states). All strings must be extracted to `app_<locale>.arb` and accessed via `context.l10n.<key>`.

---

## 9. Performance & Quality Checklist Summary

- [ ] All stateless static widgets use `const`
- [ ] No magic numbers for spacing, padding, margins, or radii (uses `AppSpacing`)
- [ ] 100% of user-visible strings localized with `context.l10n` (no hardcoded French/English text)
- [ ] Large widget trees broken into separate `StatelessWidget` classes (no `_buildX()` methods)
- [ ] Long lists use `ListView.builder` with `itemExtent` or `prototypeItem` where applicable
- [ ] Heavy animations or repaints isolated with `RepaintBoundary`
- [ ] Controllers (`TextEditingController`, `AnimationController`, etc.) and `StreamSubscription`s disposed properly
- [ ] Network images downsampled with `cacheWidth` / `cacheHeight`
- [ ] CPU-heavy computations (JSON parsing, crypto) offloaded to `Isolate.run()`
- [ ] `MediaQuery.sizeOf(context)` used instead of `MediaQuery.of(context)`
- [ ] Keys used for list item reordering or deletion
