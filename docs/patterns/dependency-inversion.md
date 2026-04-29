# 🔌 Dependency Inversion

![Dependency inversion pattern diagram](/docs/images/dependency-inversion.svg)

Dependency Inversion is the practice of **programming to an interface rather than a concrete implementation**. In a standard approach, your business logic depends directly on specific tools or SDKs, making the code rigid and difficult to test. With Dependency Inversion, you define a "contract" (an interface) that describes what a service should do, and your application only interacts with that contract.

In this example, we will walk through a common real-world scenario: **Analytics**. You’ll see how to decouple your Flutter app from specific vendors like Firebase or Facebook, allowing you to swap services, run mock versions in tests, or use a simple console logger during development—all without changing a single line of your feature logic.

## The Problem

- **Rigid Coupling:** Your code is smeared with third-party SDKs. Swapping Firebase for Amplitude or Mixpanel requires editing every file that touches analytics.
- **Untestable Logic:** Unit tests fail because they require a real Firebase instance to be initialized. You end up skipping tests or wasting hours on complex mocks.
- **No Substitutability:** You can’t easily provide "Console Logging" for development and "Real SDKs" for production without `if (kDebugMode)` checks everywhere.

## The Pattern — SOLID's "D"

High-level modules (your business logic) should not depend on low-level modules (the SDKs). Both should depend on **abstractions** (interfaces).

1.  **Domain Layer:** Owns the `interface` (the "contract").
2.  **Infrastructure Layer:** Implements the contract (the "details").
3.  **Application Layer:** Wires it all together (via **Riverpod** in this example).

## Implementation Guide

### 1\. Define the Contract (Domain)

We use `abstract interface class` so consumers can only `implements` the contract — they can't `extends` it and inherit partial behavior. This keeps the contract pure: every method must be explicitly fulfilled.

> [!TIP]
> **Use Domain Primitives:** Never leak third-party types (like a RevenueCat `Package`) through your interface. Use `double`, `String`, or your own domain entities.

```dart
abstract interface class IAnalytics {
  Future<void> logEvent({
    required String name,
    Map<String, Object>? properties,
  });

  Future<void> logPurchase({
    required String productId,
    required double amount,
    required String currencyCode,
  });
}
```

### 2\. Create the Strategies (Infrastructure)

Create a **Production** version that talks to real SDKs and a **Development** version that simply logs to the console.

```dart
// Production: Wires up Firebase + Facebook
class ProductionAnalytics implements IAnalytics {
  final FirebaseAnalytics _firebase;
  ProductionAnalytics({FirebaseAnalytics? firebase})
    : _firebase = firebase ?? FirebaseAnalytics.instance;

  @override
  Future<void> logEvent({required String name, Map<String, Object>? properties}) =>
      _firebase.logEvent(name: name, parameters: properties);

  // ... other methods
}

// Development: Zero SDK dependencies, prints to console
class DevelopmentAnalytics implements IAnalytics {
  @override
  Future<void> logEvent({required String name, Map<String, Object>? properties}) async =>
      print('[Analytics] $name: $properties');
}
```

### 3\. The Composition Root (Riverpod)

The provider acts as the single decision point for which implementation to use.

```dart
final analyticsProvider = Provider<IAnalytics>((ref) {
  // kDebugMode is a compile-time constant.
  // The unused branch is physically tree-shaken from your release binary!
  if (kDebugMode) {
    return DevelopmentAnalytics();
  }
  return ProductionAnalytics();
});
```

## Why This Matters

- **Tree-Shaking:** Because `kDebugMode` is a constant, `DevelopmentAnalytics` doesn't even exist in your production APK/IPA.
- **Instant Mocking:** In your tests, simply override the provider with a `TestAnalytics` spy to verify logic without network calls.
- **Clean Boundaries:** Your feature developers can track events without ever knowing (or caring) that Firebase is under the hood.
