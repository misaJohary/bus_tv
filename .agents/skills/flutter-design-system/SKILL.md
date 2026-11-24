---
name: flutter-design-system
description: Design system rules and tokens for spacing, padding, margins, border radii, colors, and typography using AppSpacing, AppColors, and AppTextStyles. Use when designing or refactoring UI widgets, screens, and layouts.
---

# Flutter Design System & Layout Tokens Guide

> **What:** Design system architecture and strict layout rules for GuimelMessages.
> **When to use:** Whenever creating, styling, or refactoring widgets, screens, lists, cards, dialogs, or inputs.
> **Depends on:** Works alongside `flutter-best-practices` and `flutter-localization`.

---

## 1. Zero Magic Numbers Rule

> [!CRITICAL]
> **Zero magic numbers permitted.**
> Never write arbitrary numbers for padding, margins, sized boxes, or border radii in widgets. Always use tokens from `AppSpacing`.

```dart
// ❌ FORBIDDEN:
padding: EdgeInsets.all(14)
padding: EdgeInsets.symmetric(horizontal: 18, vertical: 10)
SizedBox(height: 14)
SizedBox(width: 8)
borderRadius: BorderRadius.circular(18)
Radius.circular(4)

// ✅ REQUIRED:
padding: AppSpacing.cardPaddingCompact
padding: AppSpacing.screenPadding
AppSpacing.gapH14
AppSpacing.gapW8
borderRadius: AppSpacing.borderRadiusLg
AppSpacing.rXxs
```

---

## 2. Spacing Grid Tokens (`AppSpacing`)

Based on a strict 4pt/8pt grid:

| Token | Value | Recommended Usage |
|---|---|---|
| `AppSpacing.xxxs` | `2.0` | Micro adjustments, badge padding |
| `AppSpacing.xxs` | `4.0` | Chip horizontal padding, small gaps |
| `AppSpacing.xs` | `6.0` | Compact gaps between title and subtitle |
| `AppSpacing.sm` | `8.0` | Standard small spacing, icon-to-text gap |
| `AppSpacing.md` | `12.0` | Card compact padding, field-to-field gap |
| `AppSpacing.base` | `16.0` | Standard screen horizontal padding, card padding |
| `AppSpacing.lg` | `20.0` | Section spacing, large card padding |
| `AppSpacing.xl` | `24.0` | Screen vertical margin, inter-section gaps |
| `AppSpacing.xxl` | `32.0` | Large header spacing, hero element gaps |
| `AppSpacing.xxxl` | `40.0` | Empty states, bottom padding |
| `AppSpacing.huge` | `48.0` | Major layout separations |

---

## 3. Reusable Spacers (`const SizedBox`)

Always use `const` spacers from `AppSpacing` instead of instantiating new `SizedBox` widgets:

- **Vertical**: `AppSpacing.gapH2`, `AppSpacing.gapH4`, `AppSpacing.gapH6`, `AppSpacing.gapH8`, `AppSpacing.gapH10`, `AppSpacing.gapH12`, `AppSpacing.gapH14`, `AppSpacing.gapH16`, `AppSpacing.gapH20`, `AppSpacing.gapH24`, `AppSpacing.gapH28`, `AppSpacing.gapH32`, `AppSpacing.gapH40`
- **Horizontal**: `AppSpacing.gapW2`, `AppSpacing.gapW4`, `AppSpacing.gapW6`, `AppSpacing.gapW8`, `AppSpacing.gapW10`, `AppSpacing.gapW12`, `AppSpacing.gapW14`, `AppSpacing.gapW16`, `AppSpacing.gapW20`, `AppSpacing.gapW24`

---

## 4. Insets & Padding Presets

| Preset | Value | Usage |
|---|---|---|
| `AppSpacing.screenPadding` | `horizontal: 16, vertical: 12` | Default screen view padding |
| `AppSpacing.screenPaddingWide` | `horizontal: 24, vertical: 20` | Auth screens / form screens |
| `AppSpacing.cardPadding` | `all: 16` | Standard cards and containers |
| `AppSpacing.cardPaddingCompact` | `all: 12` | Compact list items, bubble interiors |
| `AppSpacing.allSm` | `all: 8` | Small icon containers |
| `AppSpacing.allBase` | `all: 16` | Standard full padding |

---

## 5. Border Radius & Shape Tokens

| Token | Radius Value | Component |
|---|---|---|
| `AppSpacing.borderRadiusXs` | `6.0` | Micro badges, status pills |
| `AppSpacing.borderRadiusSm` | `8.0` | Tag labels, indicators |
| `AppSpacing.borderRadiusMd` | `12.0` | Small cards, badges |
| `AppSpacing.borderRadiusIntermediate` | `14.0` | Pictogram cards, buttons, text fields |
| `AppSpacing.borderRadiusBase` | `16.0` | Standard cards, dialogs |
| `AppSpacing.borderRadiusLg` | `18.0` | Large containers, filter chips |
| `AppSpacing.borderRadiusXl` | `20.0` | Message bubbles, pill containers |
| `AppSpacing.borderRadiusFull` | `999.0` | Circular avatar rings, round buttons |

### Specialized Shapes:
- **Outgoing Chat Bubble:** `AppSpacing.bubbleBorderRadiusOutgoing` (18 top, 18 bottom-left, 4 bottom-right)
- **Incoming Chat Bubble:** `AppSpacing.bubbleBorderRadiusIncoming` (18 top, 4 bottom-left, 18 bottom-right)
- **Bottom Sheet Top:** `AppSpacing.sheetTopBorderRadius` (20 top-left, 20 top-right)

---

## 6. Internationalization Enforcement

Never use inline hardcoded strings for any user-facing UI text:
- Always add strings to `lib/localization/app_en.arb` (source of truth with `@key` description) and `lib/localization/app_fr.arb`.
- Run `flutter gen-l10n`.
- Access strictly through `context.l10n.<key>`.
- For categories, use `category.localizedPlural(context)` or `category.localizedSingular(context)`.

---

## 7. Performance & Quality Checklist

Before submitting code:
- [ ] No hardcoded numbers for paddings, margins, gaps, or radii (all use `AppSpacing`).
- [ ] No hardcoded text strings (all use `context.l10n`).
- [ ] Static widgets and spacers use `const`.
- [ ] `MediaQuery.sizeOf(context)` used for responsive dimensions.
- [ ] `flutter analyze` passes with 0 issues.
- [ ] `flutter test` passes with 100% success.
