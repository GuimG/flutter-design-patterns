# ⚡ Actions

![Actions pattern diagram](/docs/images/actions.svg)

**One-shot operations as first-class state, with declarative listeners.**

An **Action** is a tiny piece of reactive state that represents a single imperative operation — logging in, completing a purchase, deleting a row. It exposes four observable states (`idle`, `loading`, `success`, `error`) and lets the UI subscribe to lifecycle callbacks (`onSuccess`, `onError`, `onLoading`) instead of awaiting futures and wrapping them in `try/catch`.

This pattern is directly inspired by React Query's `useMutation`. If you've ever written:

```tsx
const { mutate, isLoading } = useMutation(login, {
  onSuccess: () => navigate("/home"),
  onError: (e) => toast.error(e.message),
});
```

…this is the Riverpod equivalent.

## The Problem

Async operations triggered by the user (button taps, form submits) don't fit `AsyncValue` cleanly:

- **No idle state.** `AsyncValue` starts in `loading`. But a login screen isn't loading until the user taps the button — it's _idle_. Forcing it into `AsyncValue.data(null)` is a lie that leaks into your UI logic.
- **Imperative side effects belong outside `build`.** You want to navigate on success and show a toast on error. Doing that with `ref.listen` + a manual `switch` on `AsyncValue` is verbose and easy to get wrong (you'll forget the `previous == next` guard at least once).
- **Try/catch in widgets.** Without a primitive, every submit handler ends up with `try { await ... } catch (e) { showToast(e) }` inline — which mixes domain errors with UI concerns and makes the widget untestable.
- **Double-submission.** A button tapped twice fires the operation twice. Every action needs the same guard, and re-implementing it per-screen is how bugs ship.

## The Pattern

An Action has three parts:

1. **`ActionState<T>`** — a sealed union: `idle | loading | success(T) | error(Exception, StackTrace)`. The current state of the operation.
2. **`ActionHandler<T>`** — a Riverpod notifier mixin that runs the operation, manages the state transitions, and guards against double-submission.
3. **`listenAction`** — a `WidgetRef` extension that subscribes to lifecycle callbacks declaratively from inside `build`.

The widget _watches_ the state for rendering (spinner, disabled button) and _listens_ to the state for side effects (navigation, toasts). Same provider, two consumption styles.

## Implementation Guide

### 1\. The State (Sealed Union)

```dart
sealed class ActionState<T> {
  const ActionState();

  const factory ActionState.idle() = ActionIdle<T>;
  const factory ActionState.loading() = ActionLoading<T>;
  const factory ActionState.success(T value) = ActionSuccess<T>;
  const factory ActionState.error(Exception error, StackTrace stack) = ActionError<T>;

  bool get isIdle    => this is ActionIdle;
  bool get isLoading => this is ActionLoading;
  bool get isSuccess => this is ActionSuccess;
  bool get isError   => this is ActionError;
}

class ActionIdle<T>    extends ActionState<T> { const ActionIdle(); }
class ActionLoading<T> extends ActionState<T> { const ActionLoading(); }
class ActionSuccess<T> extends ActionState<T> { const ActionSuccess(this.data); final T data; }
class ActionError<T>   extends ActionState<T> {
  const ActionError(this.error, this.stackTrace);
  final Exception error;
  final StackTrace stackTrace;
}
```

The `is*` getters are the entire reason this exists. From a widget you can ask `state.isLoading` without pattern-matching every state.

### 2\. The Handler (The Notifier Mixin)

`ActionHandler` wraps the operation in three guarantees: **don't run twice, transition through `loading`, never throw.**

```dart
mixin ActionHandler<T> on AutoDisposeNotifier<ActionState<T>> {
  Future<void> execute(Future<T> Function() fn) async {
    if (state.isLoading) return;       // double-submission guard

    state = const ActionLoading();
    state = await ActionState.guard(fn);
  }
}
```

`ActionState.guard` is a `try/catch` that converts a `Future<T>` into an `ActionState<T>` — success on completion, error on throw. The handler never rethrows: errors land in the state, where listeners can react.

### 3\. The Action Provider

A real action — login — looks like this. It's a regular notifier that mixes in `ActionHandler` and exposes `run(...)` as the public entry point.

```dart
final loginActionProvider =
    NotifierProvider.autoDispose<LoginAction, ActionState<void>>(
  LoginAction.new,
  name: 'loginAction',
);

class LoginAction extends AutoDisposeNotifier<ActionState<void>>
    with ActionHandler<void> {
  @override
  ActionState<void> build() => const ActionIdle();

  Future<void> run({required String email, required String password}) =>
      execute(() async {
        final jwt = await ref.read(authRepository).login(
          email: Email(email),
          password: password,
        );
        ref.read(eventBusProvider).publish(UserLoggedIn(jwt));
      });
}
```

> [!TIP]
> Pair Actions with the [In-Process Domain Event Bus](/docs/patterns/domain-event-bus.md). The action handles the _operation_; the bus handles the _consequences_ (analytics, profile sync, notifications). The action stays tiny and the side effects stay testable.

### 4\. Consuming from the UI

The widget has two jobs: render the current state and react to transitions.

```dart
@override
Widget build(BuildContext context) {
  final isLoading = ref.watch(loginActionProvider).isLoading;

  ref.listenAction(
    loginActionProvider,
    onSuccess: (_) => context.go(HomeRoute.routePath),
    onError: (error, _) => context.showErrorToast(error),
  );

  return CustomButton(
    isLoading: isLoading,
    onPressed: () => ref.read(loginActionProvider.notifier).run(
      email: _email,
      password: _password,
    ),
    child: Text(context.l10n.loginSubmit),
  );
}
```

That's the whole API. **`watch` for rendering, `listenAction` for side effects, `read(...).run(...)` to fire it.**

### 5\. The Listener Extension

```dart
extension ActionNotifierX on WidgetRef {
  void listenAction<T>(
    ProviderListenable<ActionState<T>> provider, {
    void Function(T data)? onSuccess,
    void Function(Exception error, StackTrace stack)? onError,
    void Function()? onLoading,
  }) {
    listen<ActionState<T>>(provider, (_, next) {
      switch (next) {
        case ActionLoading():            onLoading?.call();
        case ActionSuccess(:final data): onSuccess?.call(data);
        case ActionError(:final error, :final stackTrace):
          onError?.call(error, stackTrace);
        case ActionIdle(): break;
      }
    });
  }
}
```

Each callback is optional — most screens only need `onError`. The exhaustive `switch` over the sealed union means adding a new state at the type level forces every call site to handle it.

## Why This Matters

- **Idle is real.** Forms render correctly before the user has done anything. No fake `loading: false, data: null` checks.
- **No try/catch in widgets.** Errors flow through the state and surface in `onError`. Domain errors stay in the domain layer.
- **Free double-tap protection.** `if (state.isLoading) return` in the mixin means every action is debounced by construction.
- **Tree-shaped lifecycle.** Side effects are declared at the top of `build` next to the state they depend on, not buried inside callbacks.
- **Testable.** Override the provider with `loginActionProvider.overrideWith(...)` and pump states (`idle → loading → error`) to assert UI behavior.

## Trade-offs to Consider

- **Not for queries.** Actions are one-shot, user-initiated, and write-flavored. For server data that should auto-fetch and cache, use `FutureProvider` / `AsyncNotifier` — that's what `AsyncValue` is for.
- **One operation per provider.** A `UserAction` with `login()`, `logout()`, and `deleteAccount()` methods sharing one state is a smell — they'd stomp on each other. Make them separate providers.
- **Result lifetime is short.** With `autoDispose`, the success value lives only as long as the screen. If you need to persist the result (e.g. the JWT from login), publish it to a domain event or write it to a repository inside the action — don't read it from the action's state later.
