# 🔁 Retry with Backoff

![Retry pattern diagram](/docs/images/retry.svg)

**Exponential backoff for transient failures, with cancellation that respects provider lifetime.**

A **Retry** primitive wraps a `Future<T> Function()` in a bounded loop with exponential backoff. The consumer sees one loading state across all attempts; transient errors are silently re-tried; only the final outcome — success or the last error — reaches the UI.

The non-obvious half of the pattern is **cancellation**. In Riverpod, a notifier can be disposed mid-backoff (the user navigates away, the family argument changes, the parent provider rebuilds). A naive retry loop would happily keep firing requests against a dead notifier and crash on the first attempt to write state. This pattern hooks `Ref.onDispose` so the loop bails out cleanly, throwing a sentinel that integration code is expected to swallow.

## The Problem

Reads against a network occasionally fail for reasons that have nothing to do with the user's intent:

- **Transient 5xx and 502s** from a load balancer that flapped for half a second.
- **TCP resets and timeouts** while the device hops between Wi-Fi and cellular.
- **DNS hiccups** on a flaky hotel network.
- **Cold-start latency** on a serverless backend whose first request after idle gets killed by a client timeout.

The naive shape — one `try/catch` and surface the error — turns every flake into a red banner:

```dart
// ❌ Naive read
Future<List<Product>> fetchRecommendations() async {
  try {
    return await _api.recommendations();
  } on Exception catch (_) {
    rethrow; // user sees an error for what was a 200ms blip
  }
}
```

A first attempt at a fix — a manual retry loop — almost always ships with at least one of these bugs:

- **No backoff.** Three retries in 50ms is just a more expensive way to fail. Worse, every client in the same minute retries together and amplifies the incident.
- **No jitter.** Synchronized backoff = thundering herd when the server recovers.
- **No cap.** A `while (true)` loop that retries until the heat death of the universe.
- **No selective retry.** A 401 doesn't get better with retries; an `ArgumentError` is a bug, not a flake. Retrying them is a slow way to mask a real problem.
- **No cancellation.** The user pushed the back button two seconds ago, but the loop is mid-backoff and will try to write state to a disposed notifier when it wakes up.
- **State flicker.** Each attempt flips the UI from `loading` to `error` and back, producing a janky pulse instead of a steady spinner.

You want a single, small primitive that gets all of these right by construction.

## The Pattern

A `Ref` extension method that takes the operation and a policy, and returns a `Future<T>` that resolves once the operation either succeeds or runs out of attempts.

| Abstract concept    | Flutter / Riverpod equivalent                               |
| ------------------- | ----------------------------------------------------------- |
| Operation           | `Future<T> Function() run`                                  |
| Schedule            | `RetryPolicy` (max attempts, base delay, max delay, jitter) |
| Cancellation token  | `Ref.onDispose` + a captured `cancelled` flag               |
| Retry predicate     | `bool Function(Object error) shouldRetry`                   |
| Cancellation signal | `RetryCancelled` — a sentinel exception swallowed upstream  |

The extension lives on `Ref` (not on `Future`) for one reason: it needs `onDispose` to plug into the provider lifecycle. That coupling is the whole point — a generic `retry()` helper would not know when to give up.

```
┌─────────────────┐  run()   ┌──────────────┐
│  Provider body  ├─────────▶│  ref.retry   │
└─────────────────┘          │   (loop)     │
        ▲                    └──────┬───────┘
        │                           │ delayFor(attempt)
        │ onDispose: cancelled=true ▼
        │                    ┌──────────────┐
        └────────────────────┤ RetryPolicy  │
                             └──────────────┘
```

## Implementation Guide

### 1. The Policy

A value object describing the backoff schedule. Defaults are tuned for a typical mobile read: a few attempts, sub-second base, capped tail.

```dart
// lib/core/retry/retry_policy.dart
import 'dart:math';

class RetryPolicy {
  const RetryPolicy({
    this.maxAttempts = 4,
    this.baseDelay = const Duration(milliseconds: 200),
    this.maxDelay = const Duration(seconds: 10),
    this.multiplier = 2.0,
    this.jitter = 0.25,
  });

  final int maxAttempts;
  final Duration baseDelay;
  final Duration maxDelay;
  final double multiplier;
  final double jitter; // 0.0 = none, 1.0 = ±100%

  /// Delay to wait *before* the (attempt+1)-th try. `attempt` is 0-indexed.
  Duration delayFor(int attempt) {
    final raw = baseDelay.inMilliseconds * pow(multiplier, attempt);
    final capped = min(raw, maxDelay.inMilliseconds.toDouble());
    final spread = capped * jitter;
    final jittered = capped + (Random().nextDouble() * 2 - 1) * spread;
    return Duration(milliseconds: jittered.round().clamp(0, 1 << 31));
  }
}
```

Jitter is symmetric (`±jitter * delay`). A 25% jitter on a 1-second base means the actual wait is somewhere in `[750ms, 1250ms]` — enough to break up synchronized clients without making the schedule feel random to a debugger.

### 2. The Cancellation Sentinel

A dedicated exception type so upstream integration code can pattern-match on it and stay silent. It is _not_ a domain error — it's a control-flow signal.

```dart
// lib/core/retry/retry_cancelled.dart
class RetryCancelled implements Exception {
  const RetryCancelled();
  @override
  String toString() => 'RetryCancelled';
}
```

### 3. The Extension

The whole pattern fits in ~30 lines. Read it top to bottom — the structure _is_ the contract.

```dart
// lib/core/retry/ref_retry_extension.dart
import 'dart:async';

import 'package:flutter_riverpod/flutter_riverpod.dart';

import 'retry_cancelled.dart';
import 'retry_policy.dart';

extension RefRetry on Ref {
  /// Runs [run] with exponential backoff retry.
  ///
  /// The consumer observes only the final outcome — state stays in `loading`
  /// across retries; transient errors are not surfaced. On exhaustion the
  /// last underlying error is rethrown unchanged.
  ///
  /// Cancellation: registers an `onDispose` callback. If the notifier is
  /// disposed mid-backoff, no further attempts are made and [RetryCancelled]
  /// is thrown. Upstream integration code is expected to swallow this
  /// sentinel so no state writes happen on a disposed notifier.
  ///
  /// [shouldRetry] is consulted for every throw; returning false rethrows
  /// immediately. Defaults to retrying every throw — pass a predicate at
  /// call sites where some errors are known to be deterministic (auth
  /// failures, 4xx, programming errors).
  Future<T> retry<T>({
    required Future<T> Function() run,
    RetryPolicy policy = const RetryPolicy(),
    bool Function(Object error)? shouldRetry,
  }) async {
    var cancelled = false;
    onDispose(() => cancelled = true);

    for (var attempt = 0; attempt < policy.maxAttempts; attempt++) {
      if (cancelled) throw const RetryCancelled();
      try {
        return await run();
      } catch (error) {
        if (cancelled) throw const RetryCancelled();
        final isLastAttempt = attempt == policy.maxAttempts - 1;
        final canRetry = shouldRetry?.call(error) ?? true;
        if (isLastAttempt || !canRetry) rethrow;
        await Future<void>.delayed(policy.delayFor(attempt));
      }
    }
    throw StateError('unreachable: retry loop exited without return or throw');
  }
}
```

Two `cancelled` checks bracket the `await run()`: one before, one in the `catch`. The second matters more than the first — a request can take seconds, the user can leave the screen during that window, and you want to throw `RetryCancelled` immediately when execution resumes, before scheduling another `Future.delayed`.

### 4. Consuming it from a Provider

The pattern's natural home is a `FutureProvider` (or `FutureProvider.family`) wrapping a repository call. The provider body stays a one-liner.

```dart
// lib/features/catalog/application/recommendations_provider.dart
final recommendationsProvider = FutureProvider.autoDispose
    .family<List<Product>, CategoryId>(
  (ref, categoryId) => ref.retry(
    run: () => ref
        .read(catalogRepositoryProvider)
        .fetchRecommendations(categoryId: categoryId),
    policy: const RetryPolicy(maxAttempts: 6),
  ),
  name: 'recommendations',
);
```

The widget watches the provider as it would any other `FutureProvider`:

```dart
final recs = ref.watch(recommendationsProvider(categoryId));
return recs.when(
  loading: () => const ShimmerList(),
  error:   (e, _) => ErrorRetryView(onRetry: () => ref.invalidate(recommendationsProvider(categoryId))),
  data:    (items) => RecommendationsList(items),
);
```

The `loading` state covers _all_ attempts — the spinner doesn't flicker through error states between retries.

### 5. Selective Retry

The default — retry every throw — is right for most call sites because most call sites are network reads where any throw is potentially transient. Where it isn't, the predicate is the entire opt-out:

```dart
final currentUserProvider = FutureProvider.autoDispose<User>(
  (ref) => ref.retry(
    run: () => ref.read(authRepositoryProvider).fetchCurrentUser(),
    shouldRetry: (error) => error is! AuthExpiredException,
  ),
  name: 'currentUser',
);
```

A blanket "retry only network errors" rule would be tempting and wrong: a `TimeoutException` from a half-open socket _is_ a network error, but so is a `FormatException` from a malformed response — and the second one will not get better on retry. Decide per-call site.

### 6. Swallowing `RetryCancelled` Upstream

`RetryCancelled` reaching the UI as an error toast would be a bug — the user _caused_ the cancellation by navigating away. If the retry is invoked from inside an action notifier (see [[actions]]), the action handler is the right place to filter:

```dart
mixin ActionHandler<T> on AutoDisposeNotifier<ActionState<T>> {
  Future<void> execute(Future<T> Function() fn) async {
    if (state.isLoading) return;
    state = const ActionLoading();
    try {
      state = ActionState.success(await fn());
    } on RetryCancelled {
      // Notifier is disposed — do not touch state.
      return;
    } on Exception catch (e, s) {
      state = ActionState.error(e, s);
    }
  }
}
```

For plain `FutureProvider` bodies, no special handling is needed: a disposed provider has no listeners, and the thrown `RetryCancelled` is discarded by the framework.

## Why This Matters

- **One spinner, not a strobe.** The UI sees `loading → data` or `loading → error`. Intermediate retries don't leak.
- **Bounded.** `maxAttempts` × `maxDelay` is a hard ceiling on wasted time, set per call site.
- **Polite under load.** Jittered exponential backoff is the standard defense against thundering herds when a server recovers.
- **Lifecycle-aware.** A disposed notifier doesn't accidentally write state from a request that resolved after the user left.
- **Explicit about what's deterministic.** `shouldRetry` forces the call site to articulate which errors are flakes vs. bugs.
- **Tiny.** The whole primitive is one file, ~30 lines of logic, no dependencies beyond Riverpod.

## Trade-offs to Consider

- **Not durable.** If the OS kills the process mid-backoff, the operation is lost. For writes that absolutely must reach the server, pair with the [Outbox](/docs/patterns/outbox.md) pattern — retry handles the in-process flakes, outbox handles the cross-process ones.
- **Reads, not writes — unless idempotent.** Retrying a non-idempotent POST is how you end up with duplicate orders. Either restrict retry to GETs, send an idempotency key with the request, or both.
- **The user can't tell.** A 12-second `loading` (three retries of a 2-second request plus backoff) looks identical to a 200ms `loading` followed by a stuck spinner. If your max budget is long, expose progress out-of-band (a subtle "retrying…" banner after the second attempt) or shrink the budget.
- **No retry budget across providers.** Each provider retries independently. Ten providers all hitting the same flaky endpoint multiply the load by ten. For a fleet-wide solution, layer a circuit breaker or token-bucket on the HTTP client instead.
- **No circuit breaker.** This primitive will keep retrying even when the server is clearly down for the day. Combine with a higher-level "the API has been failing for 30 seconds, stop trying" gate if your backend has incident patterns long enough to outlast the per-call budget.
- **Default policy is opinionated.** `maxAttempts: 4`, `baseDelay: 200ms`, `maxDelay: 10s` is a reasonable mobile default — long enough to cover a tower handoff, short enough not to torture the user. Override per call site for cold-start-prone endpoints (more attempts, longer cap) or interactive paths (fewer attempts, tighter cap).
