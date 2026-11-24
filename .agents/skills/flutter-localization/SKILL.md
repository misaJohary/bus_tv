---
name: flutter-localization
description: Sets up and manages internationalization with flutter_localizations and intl. Use when adding translatable strings, creating new locales, or configuring l10n for the project.
---

# Flutter Localization

> **What:** Patterns for internationalization (i18n) and localization (l10n) in Flutter using the official `gen-l10n` tooling — ARB files, placeholders, plurals, and runtime locale switching.
> **When to use:** Adding translatable strings, supporting new languages, or setting up localization infrastructure.
> **Depends on:** Optionally pairs with `flutter-bloc-pattern` or `flutter-cubit-pattern` for runtime locale switching.

The recommended approach for Flutter localization is ARB files with `flutter gen-l10n`, which generates strongly typed localization classes. This is the official Flutter-team-endorsed method.

## Architecture

```
lib/localization/
├── app_en.arb              # Template ARB (English — source of truth)
├── app_hi.arb              # Hindi translations
├── app_<locale>.arb        # One file per supported locale
└── gen/                    # Auto-generated (do not edit manually)
    ├── app_localizations.dart
    ├── app_localizations_en.dart
    ├── app_localizations_hi.dart
    └── ...
```

Configuration lives in `l10n.yaml` at the project root:

```yaml
arb-dir: lib/localization
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
output-class: AppLocalizations
output-dir: lib/localization/gen
synthetic-package: false    # Avoids import path issues, recommended
nullable-getter: false      # Makes AppLocalizations.of(context) non-nullable
```

## ARB File Format

ARB (Application Resource Bundle) is a JSON-based format. The template file (English) includes metadata; translation files only need the values.

### Template file (app_en.arb)

```json
{
  "@@locale": "en",

  "loginTitle": "Log in",
  "@loginTitle": {
    "description": "Login screen title"
  },

  "otpCodeSentTo": "Code sent to {phoneNumber}",
  "@otpCodeSentTo": {
    "description": "OTP screen subtitle showing the phone number",
    "placeholders": {
      "phoneNumber": {
        "type": "String"
      }
    }
  },

  "itemCount": "{count, plural, =0{No items} =1{1 item} other{{count} items}}",
  "@itemCount": {
    "description": "Item count with pluralization",
    "placeholders": {
      "count": {
        "type": "int"
      }
    }
  }
}
```

### Translation file (app_hi.arb)

```json
{
  "@@locale": "hi",
  "loginTitle": "लॉग इन",
  "otpCodeSentTo": "कोड {phoneNumber} पर भेजा गया"
}
```

## Key Naming Conventions

Use a `screenPrefix_elementDescription` pattern:

| Pattern | Example | Description |
|---|---|---|
| `<screen><Element>` | `loginTitle` | Screen-specific string |
| `<screen><Action>` | `loginSendOtp` | Button/CTA text |
| `<screen><Element><Qualifier>` | `onboardingTitleStep1` | Variant of an element |
| `common<Element>` | `commonCancel` | Shared across screens |

Rules:
- Use `camelCase` for all keys
- Prefix with screen/feature name for scoped strings
- Use `common` prefix for strings shared across multiple screens
- Every key in the template file must have a `@key` metadata entry with at least a `description`
- Keep descriptions concise but useful — they help translators understand context

## Placeholders

For dynamic values in strings:

```json
{
  "greeting": "Hello, {name}!",
  "@greeting": {
    "placeholders": {
      "name": { "type": "String" }
    }
  }
}
```

Supported types: `String`, `int`, `double`, `num`, `DateTime`.

For date/number formatting:

```json
{
  "lastSeen": "Last seen {date}",
  "@lastSeen": {
    "placeholders": {
      "date": {
        "type": "DateTime",
        "format": "yMMMd"
      }
    }
  }
}
```

## Pluralization

Use ICU message syntax:

```json
{
  "notificationCount": "{count, plural, =0{No notifications} =1{1 notification} other{{count} notifications}}",
  "@notificationCount": {
    "placeholders": {
      "count": { "type": "int" }
    }
  }
}
```

## Select (Gender / Enum)

```json
{
  "pronoun": "{gender, select, male{He} female{She} other{They}}",
  "@pronoun": {
    "placeholders": {
      "gender": { "type": "String" }
    }
  }
}
```

## Code Generation

Run after adding/modifying ARB files:

```bash
flutter gen-l10n
```

This generates the `AppLocalizations` class and per-locale implementations in the `gen/` directory.

The generated files should be committed to version control (not gitignored) so the project builds without running the generator.

## App Setup

In `MaterialApp` or `MaterialApp.router`:

```dart
MaterialApp.router(
  locale: currentLocale,
  localizationsDelegates: const [
    AppLocalizations.delegate,
    GlobalMaterialLocalizations.delegate,
    GlobalWidgetsLocalizations.delegate,
    GlobalCupertinoLocalizations.delegate,
  ],
  supportedLocales: AppLocalizations.supportedLocales,
  // ... other config
)
```

## Usage in Widgets

```dart
// Access localized strings — prefer the non-null assertion or extension approach
final l10n = AppLocalizations.of(context)!;

Text(l10n.loginTitle);
Text(l10n.otpCodeSentTo('+91 9876543210'));
Text(l10n.itemCount(3));
```

Shorthand with an extension (recommended):

```dart
extension BuildContextL10n on BuildContext {
  AppLocalizations get l10n => AppLocalizations.of(this)!;
}

// Usage — cleaner and consistent across the codebase
Text(context.l10n.loginTitle);
```

## Accessibility Considerations

- Ensure all user-visible text is localized, including:
  - Semantic labels for images and icons (`Semantics` widget)
  - Tooltip text
  - Error messages and validation text
  - Snackbar and dialog content
- Use `Directionality` or `TextDirection` for RTL language support
- Test with screen readers in each supported locale

## Locale Switching at Runtime

Manage locale state in a BLoC or Cubit:

```dart
// Event
class ChangeLocale extends AppEvent {
  final Locale locale;
  const ChangeLocale(this.locale);
}

// In BLoC handler
Future<void> _onChangeLocale(ChangeLocale event, Emitter<AppState> emit) async {
  await prefs.setLanguageCode(event.locale.languageCode);
  emit(AppLoaded(locale: event.locale, themeMode: state.themeMode));
}
```

The `MaterialApp` rebuilds with the new locale when the BLoC state changes.

## Adding a New Language — Checklist

1. Create `lib/localization/app_<code>.arb` with `"@@locale": "<code>"`
2. Translate all keys from the template file
3. Run `flutter gen-l10n`
4. Add the `Locale('<code>')` to the supported locales list (if not auto-detected)
5. Verify the new locale appears in `AppLocalizations.supportedLocales`

## Adding a New String — Checklist

1. Add the key + value to `app_en.arb` (template)
2. Add `@key` metadata with description (and placeholders if needed)
3. Add translations to all other `app_<locale>.arb` files
4. Run `flutter gen-l10n`
5. Use `AppLocalizations.of(context)!.newKey` in the widget

## Common Pitfalls

- **Missing translations**: if a key exists in the template but not in a translation file, `gen-l10n` uses the template value as fallback. This is fine for development but should be caught before release
- **Editing generated files**: never edit files in `gen/` — they're overwritten on each `flutter gen-l10n` run
- **Hardcoded strings in widgets**: all user-visible text should come from ARB files, not inline strings. This includes error messages, tooltips, and semantic labels
- **Forgetting `flutter gen-l10n`**: after modifying ARB files, always regenerate before building. Consider adding `synthetic-package: false` to `l10n.yaml` to avoid import issues
- **Special characters**: use `'` for apostrophes in ICU messages, escape with `''` if needed
- **Long strings**: for multi-line or HTML-rich content, keep the ARB value on a single line and handle formatting in the widget
- **Missing `@@locale`**: every ARB file must have the `@@locale` key — omitting it can cause locale detection issues
- **Not testing with real devices**: always test localized text on actual devices to catch overflow, truncation, and RTL layout issues
