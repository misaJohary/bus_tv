---
name: flutter-routing
description: Configures declarative routing with go_router. Use when adding new screens, setting up navigation, deep linking, or managing route guards.
---

# Flutter Routing

> **What:** Patterns for declarative routing in Flutter using `go_router` — route definitions, guards, transitions, deep linking, and shell routes.
> **When to use:** Adding or modifying routes, implementing auth guards, setting up tab navigation, or handling deep links.
> **Depends on:** Optionally pairs with `flutter-bloc-pattern` for route guards based on auth state.

## Architecture

Routing is centralized in a dedicated `routes/` directory:

```
routes/
├── app_router.dart          # GoRouter configuration and route definitions
├── app_route_names.dart     # Route name constants (used with goNamed)
└── app_route_paths.dart     # Route path constants (URL patterns)
```

## Route Names vs Route Paths

Separate constants for names and paths keeps navigation type-safe and refactor-friendly.

```dart
// app_route_names.dart — logical identifiers for goNamed()
class AppRouteNames {
  AppRouteNames._();

  static const String login = 'auth';
  static const String dashboard = 'dashboard';
  static const String productDetail = 'productDetail';
}

// app_route_paths.dart — URL path patterns
class AppRoutePaths {
  AppRoutePaths._();

  static const String login = '/auth';
  static const String dashboard = '/dashboard';
  static const String productDetail = '/product/:id';
}
```

Rules:
- Private constructor (`._()`) to prevent instantiation
- Names are short identifiers; paths are URL-style strings
- Use path parameters (`:id`) for resource identifiers
- Use query parameters for optional/filter data

## Router Configuration

```dart
class AppRouter {
  AppRouter._();

  static GoRouter create({
    required GlobalKey<NavigatorState> navigatorKey,
    List<NavigatorObserver> observers = const [],
  }) {
    return GoRouter(
      navigatorKey: navigatorKey,
      initialLocation: AppRoutePaths.splash,
      observers: observers,
      routes: _routes,
      errorBuilder: (context, state) => ErrorScreen(error: state.error),
    );
  }

  static final List<RouteBase> _routes = [
    GoRoute(
      name: AppRouteNames.splash,
      path: AppRoutePaths.splash,
      builder: (context, state) => const SplashScreen(),
    ),
    // ... more routes
  ];
}
```

Rules:
- Router is a static utility class — no instance state
- Pass `navigatorKey` from a global shared file for programmatic navigation
- Add `NavigatorObserver`s for analytics, debugging (e.g., Chucker in staging)
- Always provide an `errorBuilder` for unmatched routes

## Route Registration Patterns

### Simple screen

```dart
GoRoute(
  name: AppRouteNames.login,
  path: AppRoutePaths.login,
  builder: (context, state) => const LoginScreen(),
),
```

### Screen with BLoC provider

```dart
GoRoute(
  name: AppRouteNames.splash,
  path: AppRoutePaths.splash,
  builder: (context, state) => BlocProvider(
    create: (_) => SplashBloc(),
    child: const SplashScreen(),
  ),
),
```

### Screen with path parameters

```dart
GoRoute(
  name: AppRouteNames.productDetail,
  path: AppRoutePaths.productDetail, // '/product/:id'
  builder: (context, state) {
    final id = state.pathParameters['id'] ?? '';
    return ProductDetailScreen(productId: id);
  },
),
```

### Screen with query parameters

```dart
GoRoute(
  name: AppRouteNames.otp,
  path: AppRoutePaths.otp,
  builder: (context, state) {
    final q = state.uri.queryParameters;
    return OtpScreen(
      uuid: q['uuid'] ?? '',
      mobile: q['mobile'] ?? '',
    );
  },
),
```

### Custom page transitions

```dart
GoRoute(
  name: AppRouteNames.onboarding,
  path: AppRoutePaths.onboarding,
  pageBuilder: (context, state) => CustomTransitionPage(
    key: state.pageKey,
    child: const OnboardingScreen(),
    transitionDuration: const Duration(milliseconds: 600),
    reverseTransitionDuration: const Duration(milliseconds: 300),
    transitionsBuilder: (context, animation, secondaryAnimation, child) {
      return FadeTransition(opacity: animation, child: child);
    },
  ),
),
```

### Nested routes (shell route)

For persistent navigation UI (bottom bars, sidebars), use `ShellRoute` or `StatefulShellRoute`:

```dart
// ShellRoute — for stateless tabs (content rebuilds on switch)
ShellRoute(
  builder: (context, state, child) => DashboardShell(child: child),
  routes: [
    GoRoute(
      name: AppRouteNames.home,
      path: AppRoutePaths.home,
      builder: (context, state) => const HomeTab(),
    ),
    GoRoute(
      name: AppRouteNames.profile,
      path: AppRoutePaths.profile,
      builder: (context, state) => const ProfileTab(),
    ),
  ],
),

// StatefulShellRoute — preserves state across tab switches (scroll positions, forms, etc.)
StatefulShellRoute.indexedStack(
  builder: (context, state, navigationShell) {
    return ScaffoldWithNav(navigationShell: navigationShell);
  },
  branches: [
    StatefulShellBranch(
      navigatorKey: GlobalKey<NavigatorState>(debugLabel: 'feed'),
      routes: [
        GoRoute(
          path: '/feed',
          builder: (context, state) => const FeedScreen(),
        ),
      ],
    ),
    StatefulShellBranch(
      navigatorKey: GlobalKey<NavigatorState>(debugLabel: 'profile'),
      routes: [
        GoRoute(
          path: '/profile',
          builder: (context, state) => const ProfileScreen(),
        ),
      ],
    ),
  ],
),
```

Use `StatefulShellRoute` when tabs contain scroll positions, form input, or in-progress state that must survive tab switches. Use `ShellRoute` for simple, read-only content.

Rules for tab switching:
- Use `context.go` for tab switches, not `context.push` — `go` resets the stack, `push` stacks screens
- Use `navigationShell.goBranch(index, initialLocation: index == navigationShell.currentIndex)` to reset a tab when re-tapped

## Navigation from Code

### Named navigation (preferred)

```dart
// Push a named route
context.goNamed(AppRouteNames.dashboard);

// With query parameters
context.goNamed(
  AppRouteNames.otp,
  queryParameters: {'uuid': uuid, 'mobile': mobile},
);

// With path parameters
context.goNamed(
  AppRouteNames.productDetail,
  pathParameters: {'id': productId},
);
```

### Push vs Go

| Method | Behavior |
|---|---|
| `context.go('/path')` | Replaces the entire navigation stack |
| `context.push('/path')` | Pushes on top of current stack |
| `context.goNamed('name')` | Same as `go` but uses route name |
| `context.pushNamed('name')` | Same as `push` but uses route name |
| `context.pop()` | Pops the top route |
| `context.pop(result)` | Pops with a return value |

Rules:
- Use `go` for top-level navigation (login → dashboard, splash → onboarding)
- Use `push` for drill-down navigation (list → detail)
- Use `pop` to go back
- Always use named variants (`goNamed`, `pushNamed`) over raw paths for type safety

## Build Variant Routing

When routes differ between build variants (staging vs production):

```dart
static GoRouter routerFor(BuildVariant variant) {
  final observers = <NavigatorObserver>[
    routeObserver,
    if (variant == BuildVariant.staging) debugObserver,
  ];

  return GoRouter(
    observers: observers,
    routes: [
      GoRoute(
        path: '/',
        builder: (context, state) {
          if (variant == BuildVariant.staging) {
            return const EnvironmentSelectionScreen();
          }
          return const SplashScreen();
        },
      ),
      // ... shared routes
    ],
  );
}
```

## Route Guards / Redirects

```dart
GoRouter(
  redirect: (context, state) {
    final isLoggedIn = authService.isLoggedIn;
    final isOnLoginPage = state.matchedLocation == AppRoutePaths.login;

    if (!isLoggedIn && !isOnLoginPage) return AppRoutePaths.login;
    if (isLoggedIn && isOnLoginPage) return AppRoutePaths.dashboard;
    return null; // no redirect
  },
  routes: [...],
);
```

For more complex auth flows, use `GoRouterRedirect` with async checks:

```dart
redirect: (context, state) async {
  final authState = context.read<AuthBloc>().state;
  // ... check auth state
},
```

## Deep Linking

`go_router` supports deep linking out of the box. Ensure:
- All routes have meaningful path patterns (e.g., `/product/:id` not `/screen2`)
- Path parameters use descriptive names
- The `errorBuilder` handles unknown deep links gracefully

```dart
// Handle deep link parameters
GoRoute(
  path: '/invite/:code',
  builder: (context, state) {
    final inviteCode = state.pathParameters['code'] ?? '';
    return InviteScreen(code: inviteCode);
  },
),
```

## Type-Safe Route Parameters

For complex route data, use `extra` with typed objects instead of encoding everything in query params:

```dart
// Passing typed data
context.goNamed(
  AppRouteNames.productDetail,
  pathParameters: {'id': product.id.toString()},
  extra: product, // Pass the full object
);

// Receiving typed data
GoRoute(
  name: AppRouteNames.productDetail,
  path: AppRoutePaths.productDetail,
  builder: (context, state) {
    final product = state.extra as Product?;
    final id = state.pathParameters['id'] ?? '';
    // Use product if available, otherwise fetch by id
    return ProductDetailScreen(product: product, productId: id);
  },
),
```

Note: `extra` does not survive app restarts, browser refreshes, or deep links. Always have a fallback to fetch data from the path parameter. Treat `extra` as a last resort.

### Type-Safe Routes with `go_router_builder`

For compile-time route safety, use `go_router_builder` to generate type-checked route classes:

```yaml
dev_dependencies:
  go_router_builder: ^2.7.0
  build_runner: ^2.4.0
```

```dart
@TypedGoRoute<ProductDetailRoute>(path: '/product/:id')
class ProductDetailRoute extends GoRouteData {
  const ProductDetailRoute({required this.id});
  final String id;

  @override
  Widget build(BuildContext context, GoRouterState state) {
    return ProductDetailScreen(productId: id);
  }
}

// Navigate with type safety — compiler catches errors
const ProductDetailRoute(id: '42').go(context);
```

This eliminates typo-related bugs entirely. Path parameters become constructor arguments, and changing a parameter name flags every call site at compile time.

## Reactive Route Refreshing

When auth state or locale changes at runtime, use `refreshListenable` to re-evaluate redirects automatically:

```dart
final authNotifier = AuthNotifier(); // extends ChangeNotifier

final router = GoRouter(
  refreshListenable: Listenable.merge([authNotifier, localeNotifier]),
  redirect: (context, state) {
    if (!authNotifier.isLoggedIn) return AppRoutePaths.login;
    return null;
  },
  routes: [...],
);
```

## Navigator Key Management

Declare all navigator keys at the top of your `router.dart` file. Each navigator needs a unique `GlobalKey<NavigatorState>`:

```dart
final rootNavigatorKey = GlobalKey<NavigatorState>(debugLabel: 'root');
final shellNavigatorKey = GlobalKey<NavigatorState>(debugLabel: 'shell');
```

Rules:
- Use `debugLabel` on every key — it appears in error messages and DevTools
- Never create keys inside `build` methods — they must persist across rebuilds
- Use `parentNavigatorKey: rootNavigatorKey` on routes that should render outside a shell (e.g., checkout, onboarding)

## Adding a New Route — Checklist

1. Add route name constant to `app_route_names.dart`
2. Add route path constant to `app_route_paths.dart`
3. Add `GoRoute` entry in `app_router.dart`
4. If the screen needs a BLoC, wrap with `BlocProvider` in the builder
5. If using query/path params, extract them from `state` in the builder
6. If custom transition is needed, use `pageBuilder` with `CustomTransitionPage` and `state.pageKey`
7. Test the route with deep linking if applicable
8. Add redirect logic if the route requires authentication
9. Write tests for redirects, parameter extraction, and shell navigation

## Common Pitfalls

- **Forgetting `errorBuilder`**: always handle unmatched routes with a user-friendly error screen
- **Using `go` when `push` is needed**: `go` replaces the stack, `push` adds to it. Use `go` for top-level navigation, `push` for drill-down
- **Not extracting route constants**: hardcoded path strings scattered across the codebase are fragile. Always use `AppRouteNames` and `AppRoutePaths`
- **Passing large objects via query params**: use `extra` for complex data, but always have a fallback for deep links
- **Missing `BlocProvider` in route builder**: if a screen needs a BLoC, forgetting to provide it in the route builder causes runtime errors
- **Not handling null path parameters**: always provide defaults when extracting `pathParameters` or `queryParameters`
- **Creating navigator keys inside `build`**: keys recreated every frame cause assertion failures. Declare as top-level variables
- **Using `push` for tab switches**: tab switching should use `go` to reset the stack, not `push` which stacks screens
- **Nesting shells too deeply**: limit to two levels. Three or more nested shells become difficult to debug
- **Missing `state.pageKey` in `pageBuilder`**: omitting it causes stale pages to linger in the navigator
