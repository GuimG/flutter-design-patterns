# 📡 In-Process Domain Event Bus

![Domain event bus pattern diagram](/docs/images/domain-event-bus.svg)

**Decoupling feature modules through asynchronous communication.**

An **In-Process Domain Event Bus** is a mechanism that allows different parts of your app to communicate without knowing about each other. Instead of Feature A calling Feature B directly, Feature A simply broadcasts that "something happened" (a Domain Event). Any other feature can listen for that event and react accordingly.

In this example, we will see how to handle a common modularity nightmare: **Post-Login Orchestration**. You'll learn how to trigger updates across a dozen independent features (Analytics, User Profile, Notifications, etc.) without creating a circular dependency web that makes your code impossible to test or maintain.

## The Problem

As a Flutter app grows into a "Modular Monolith" with dozens of features, you often encounter **Horizontal Coupling**:

- **Circular Imports:** Feature A (Auth -> Login) needs to refresh Feature B (User Profile). Later, Feature B needs to check something in Feature A. Dart will eventually refuse to compile this.
- **The "God" Action:** A single "login" function ends up importing 10 different repositories just to call `.refresh()` on all of them. Every time you add a new feature, you have to modify this unrelated auth code.
- **Fragile Tests:** To unit test the "Login" logic, you are forced to mock the entire app because the login function is hard-wired to every other service.

## Pub-Sub Architecture

The Event Bus acts as a **Mediator**. It provides:

1.  **Spatial Decoupling:** The publisher doesn't know who is listening (or if anyone is listening at all).
2.  **Temporal Decoupling:** The publisher doesn't wait for the listeners to finish; it fires the event and moves on.

## Implementation Guide

### 1\. Define the Domain Event (The Contract)

An event is a **Value Object** describing a fact about the past. Always use **past-tense** naming.

```dart
abstract class DomainEvent extends Equatable {
  final DateTime occurredOn;

  DomainEvent() : occurredOn = DateTime.now();

  @override
  String toString() {
    return '[EVENT] ==> ${super.toString()}';
  }

  @override
  List<Object?> get props => [occurredOn];
}

class UserLoggedIn extends DomainEvent {
  final String userId;

  UserLoggedIn({required this.userId});

  @override
  List<Object?> get props => [...super.props, userId];
}
```

### 2\. The Event Bus (The Channel)

We use a **Broadcast Stream** to allow multiple subscribers for a single event.

```dart
class EventBus {
  final _controller = StreamController<DomainEvent>.broadcast();

  void publish(DomainEvent event) => _controller.add(event);

  Stream<T> on<T extends DomainEvent>() =>
      _controller.stream.where((event) => event is T).cast<T>();

  void dispose() => _controller.close();
}
```

> [!NOTE]
> Errors thrown inside a handler stay inside that handler — they don't bubble up to the publisher. Wrap handler bodies in `try/catch` and log failures, otherwise a single broken subscriber will silently drop events.

### 3\. The Event Handler (The Reaction)

Each feature defines its own handlers. The **Analytics** feature has a handler for `UserLoggedIn`; the **Notifications** feature has another.

```dart
abstract class DomainEventHandler<T extends DomainEvent> {
  Future<void> handle(T event);
}

class OnUserLoggedInAnalyticsHandler extends DomainEventHandler<UserLoggedIn> {
  final IAnalytics _analytics;
  OnUserLoggedInAnalyticsHandler(this._analytics);

  @override
  Future<void> handle(UserLoggedIn event) async {
    await _analytics.setUserId(event.userId);
    await _analytics.logEvent(name: 'login_success');
  }
}
```

### 4\. The Observer (The Wiring)

The `DomainEventsObserver` is the **Composition Root**. It’s the one place where you map events to their handlers.

```dart
void initializeEventSubscribers(Ref ref) {
  final bus = ref.read(eventBusProvider);

  void subscribe<T extends DomainEvent>(DomainEventHandler<T> handler) {
    bus.on<T>().listen(handler.handle);
  }

  subscribe<UserLoggedIn>(OnUserLoggedInAnalyticsHandler(ref.read(analyticsProvider)));
  subscribe<UserLoggedIn>(OnUserLoggedInSyncHandler(ref.read(userRepositoryProvider)));
  // Add 10 more reactions here without touching the Auth feature!
}
```

## Key Benefits

- **Zero-Dependency Features:** Feature A can be completely removed from the project, and Feature B will still compile because they only shared a "contract" (the event), not code.
- **Asynchronous Safety:** A slow analytics call won't block the user from navigating to the home screen after login.
- **Traceability:** By subscribing a global logger to the base `DomainEvent` class, you get a perfect "flight data recorder" of everything that happened in the app.

## Trade-offs to Consider

- **The "God" File:** The `Observer` will eventually import many features. This is the only place where this coupling is allowed.
- **In-Memory Only:** If the app crashes, events that were just published but not yet handled are lost. For events that **must** reach the server (purchases, critical writes), persist them first with a Transactional Outbox-style pattern instead of relying on the bus.
