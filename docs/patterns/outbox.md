# 📦 Outbox

![Outbox pattern diagram](/docs/images/outbox.svg)

**Persistent, idempotent retries for writes that absolutely must reach the server.**

An **Outbox** is a durable queue, stored on the device, that holds the intent of a server-bound write. The user's action is recorded to the outbox _first_ — synchronously, before the network call. A dispatcher then drains the queue: every time the app starts, every time connectivity is restored, every time the user resumes the app, and once eagerly right after the action is enqueued.

If the request succeeds, the entry is deleted. If the request fails transiently (no internet, server 5xx, timeout), the entry stays. If the request fails permanently (the resource doesn't exist, the input is invalid), the handler drops it. The user, meanwhile, gets immediate feedback as though the operation already worked — because, from the app's point of view, the _commitment_ has been made.

This pattern is borrowed straight from server-side distributed systems, where it's traditionally used to bridge a database transaction and a message broker (write to the DB and the outbox in one transaction, then a separate worker drains the outbox to Kafka/RabbitMQ). On the client, the "transaction" is `SharedPreferences` / SQLite, and the "broker" is your HTTP API — but the guarantees are the same: **at-least-once delivery with idempotent retries**.

## The Problem

Some writes are not allowed to fail silently:

- The user just paid for something and you need to tell the server.
- The user just sent a message in a chat that needs to actually reach the recipient.
- The user just completed an exercise / lesson / quest, and the streak counter on the home screen would be lying if the server didn't get the update.
- The user just submitted a form that triggers a downstream process you cannot recover from manually.

The naive shape of these flows looks like this:

```dart
// ❌ Naive write
Future<void> completeOrder(Cart cart) async {
  state = const ActionLoading();
  try {
    await _api.submitOrder(cart);
    state = const ActionSuccess();
  } on Exception catch (e, s) {
    state = ActionError(e, s);
  }
}
```

Every one of these scenarios will eventually break it:

- **Spotty connectivity.** The user is on a train, in a tunnel, in an elevator. Their tap lands in the dead seconds between two cell towers. The button shows a red toast. They give up, or worse, they tap again — five seconds later — and now you have a duplicate write.
- **App killed mid-request.** The OS evicted the process while the request was in flight. You have no idea whether the server received it. There is no retry; there isn't even a record that an attempt happened.
- **Transient 5xx / 502 from the load balancer.** Common, expected, recoverable — and yet your app surfaces it to the user as if the order failed permanently.
- **Auth token expired mid-flight.** Refresh-on-401 is a separate concern, but a write that hits a 401 should not be lost while the refresh happens; it should be retried after.
- **The user is offline by design.** Modern apps are expected to record state locally and reconcile when the network returns. Without an outbox, your "offline support" is just "the screen renders without crashing."

A more elaborate fix — wrap the call in `retry()` with exponential backoff — addresses one of these (transient 5xx) and none of the others. A retry loop dies with the process. A retry loop doesn't survive being killed for memory pressure. A retry loop doesn't help when the user closes the app and reopens it three hours later on a different network.

## The Pattern

The outbox separates **the intent of a write** from **the act of performing it**.

1. **Action.** A small, serializable value object describing _what should happen_ (e.g. `SendMessage`, `CompleteOrder`). It carries an idempotency key (`id`), a discriminator (`type`), a creation timestamp, and a JSON payload.
2. **Record.** The persisted form of an action — what's actually written to disk. Identical fields plus an `attemptCount`. The repository deals only in records, never in feature-specific action subclasses.
3. **Repository.** A persistence interface (`add` / `getAll` / `remove` / `incrementAttempts` / `clear`) backed by anything durable: `SharedPreferences`, `Hive`, `sqflite`, the filesystem. Survives process death.
4. **Handler.** Per-feature code that knows how to replay one action `type`. Owns deserialization of the payload and decides whether to retry, give up, or remove the record.
5. **Dispatcher.** A dumb iterator. Reads pending records, looks up the handler for each `type`, calls it, catches whatever it throws. Coalesces concurrent flushes behind a single in-flight future.
6. **Triggers.** UI-layer hooks that fire `dispatcher.flush()` on cold start, on connectivity restored, on app resumed, and right after enqueue.

The producer and the consumer are decoupled. The feature that enqueues the action does not import the dispatcher; the dispatcher does not import any feature. The only shared types live in `core/outbox/domain/`.

```
┌──────────────┐   add()    ┌──────────────┐    flush()   ┌──────────────┐
│   Feature    ├───────────▶│  Repository  │◀─────────────┤  Dispatcher  │
│  (enqueues)  │            │  (durable)   │              │  (iterates)  │
└──────────────┘            └──────────────┘              └──────┬───────┘
                                                                 │ handle(record)
                                                                 ▼
                                                         ┌──────────────┐
                                                         │   Handler    │
                                                         │ (per-feature)│
                                                         └──────────────┘
```

## Implementation Guide

### 1. The Action (Domain)

The base class is generic. It has no idea what any concrete action does — it just defines the contract every action must fulfill so the repository can persist it.

```dart
abstract class OutboxAction {
  const OutboxAction();

  /// Unique identifier for this action. Reused as the `Idempotency-Key`
  /// HTTP header so the server can dedupe retries of the same logical write.
  String get id;

  /// Discriminator used by the dispatcher to route this action to its
  /// handler. Must match the corresponding `OutboxHandler.type`.
  String get type;

  /// When the action was first enqueued. Used by handlers as the basis for
  /// "older than N days → drop" safety valves.
  DateTime get createdAt;

  /// Feature-defined JSON payload. The handler for [type] is the only code
  /// that ever interprets this map.
  Map<String, dynamic> toPayload();
}
```

A concrete action lives next to the feature that owns it. For a chat app:

```dart
class SendMessageAction extends OutboxAction {
  const SendMessageAction({
    required this.id,
    required this.conversationId,
    required this.body,
    required this.createdAt,
  });

  @override
  final String id;
  @override
  final DateTime createdAt;

  final String conversationId;
  final String body;

  @override
  String get type => 'send_message';

  @override
  Map<String, dynamic> toPayload() => {
        'conversation_id': conversationId,
        'body': body,
      };
}
```

> [!TIP]
> **Generate `id` client-side.** Use `Uuid().v4()` or a ULID. The same `id` becomes the `Idempotency-Key` HTTP header when the handler eventually fires the request. If two retries hit the server, the second one is a no-op because the server recognizes the key. You don't need a dedupe table on the client — the server does it for you.

### 2. The Record (Wire/Storage Form)

The repository never sees `SendMessageAction`. It sees a record — strings and a JSON map — so the core layer can store and replay actions from any feature without importing feature code.

```dart
class OutboxRecord {
  const OutboxRecord({
    required this.id,
    required this.type,
    required this.payload,
    required this.createdAt,
    required this.attemptCount,
  });

  final String id;
  final String type;
  final Map<String, dynamic> payload;
  final DateTime createdAt;
  final int attemptCount;

  OutboxRecord copyWith({int? attemptCount}) => OutboxRecord(
        id: id,
        type: type,
        payload: payload,
        createdAt: createdAt,
        attemptCount: attemptCount ?? this.attemptCount,
      );

  Map<String, dynamic> toJson() => {
        'id': id,
        'type': type,
        'payload': payload,
        'created_at': createdAt.toUtc().toIso8601String(),
        'attempt_count': attemptCount,
      };

  factory OutboxRecord.fromJson(Map<String, dynamic> json) => OutboxRecord(
        id: json['id'] as String,
        type: json['type'] as String,
        payload: Map<String, dynamic>.from(json['payload'] as Map),
        createdAt: DateTime.parse(json['created_at'] as String).toUtc(),
        attemptCount: json['attempt_count'] as int? ?? 0,
      );
}
```

### 3. The Repository (Domain Interface)

```dart
abstract interface class IOutboxRepository {
  /// Enqueue an action. If an entry with the same id already exists it is
  /// replaced (idempotent enqueue).
  Future<void> add(OutboxAction action);

  /// All pending records, oldest first (FIFO).
  Future<List<OutboxRecord>> getAll();

  /// Remove the record with the given id. No-op if it does not exist.
  Future<void> remove(String id);

  /// Increment the attempt counter on the record with the given id.
  Future<void> incrementAttempts(String id);

  /// Wipe all records. Called on logout.
  Future<void> clear();
}
```

A `SharedPreferences` implementation is fine for the typical case — most apps queue 0 or 1 entries at a time, and reading/rewriting a tiny JSON array on every mutation is faster than spinning up a database. **Scope the storage key by user id**: a queue belongs to the account that produced it.

```dart
class SharedPreferencesOutboxRepository implements IOutboxRepository {
  SharedPreferencesOutboxRepository(this._prefs, this._currentUserId);

  final SharedPreferences _prefs;
  final String? Function() _currentUserId;

  String? get _key {
    final userId = _currentUserId();
    return userId == null ? null : 'outbox:$userId';
  }

  Future<List<OutboxRecord>> _readAll() async {
    final key = _key;
    if (key == null) return [];

    final raw = _prefs.getStringList(key) ?? const [];
    final records = <OutboxRecord>[];
    for (final entry in raw) {
      try {
        records.add(OutboxRecord.fromJson(jsonDecode(entry) as Map<String, dynamic>));
      } on Exception {
        // A corrupt entry must not poison the whole queue. Drop it.
      }
    }
    return records;
  }

  Future<void> _writeAll(String key, List<OutboxRecord> records) =>
      _prefs.setStringList(key, records.map((r) => jsonEncode(r.toJson())).toList());

  @override
  Future<void> add(OutboxAction action) async {
    final key = _key;
    if (key == null) throw StateError('Cannot enqueue without a logged-in user');

    final records = await _readAll()
      ..removeWhere((r) => r.id == action.id)
      ..add(OutboxRecord(
        id: action.id,
        type: action.type,
        payload: action.toPayload(),
        createdAt: action.createdAt,
        attemptCount: 0,
      ));
    await _writeAll(key, records);
  }

  // remove / incrementAttempts / clear / getAll — see full source.
}
```

> [!NOTE]
> **Per-user scoping is not optional.** If user A enqueues a write, then logs out and user B logs in on the same device, user B must not see (let alone replay) user A's queue. Either key by `userId` as shown above, or call `clear()` from your logout flow.

### 4. The Handler (Per-Feature)

This is the only place that knows what an action _means_. It owns deserialization, the HTTP call, and the give-up logic.

```dart
abstract class OutboxHandler {
  const OutboxHandler();

  /// Discriminator that matches `OutboxAction.type` / `OutboxRecord.type`.
  String get type;

  Future<void> handle(OutboxRecord record);
}
```

The error contract is the most important part of the whole pattern, and it's worth being explicit:

| Outcome               | Examples                       | What the handler does                       |
| --------------------- | ------------------------------ | ------------------------------------------- |
| **Success**           | `200`, `201`, `204`            | Remove record, return normally              |
| **Permanent failure** | `400`, `404`, `409`, `422`     | Log, remove record, return normally         |
| **Transient failure** | Offline, `5xx`, timeout, `401` | Throw / rethrow — the dispatcher will retry |

A concrete handler:

```dart
class SendMessageOutboxHandler extends OutboxHandler {
  SendMessageOutboxHandler({
    required this.api,
    required this.repository,
  });

  final ChatApi api;
  final IOutboxRepository repository;

  @override
  String get type => 'send_message';

  @override
  Future<void> handle(OutboxRecord record) async {
    // Safety valve: drop records that have been stuck for too long.
    if (DateTime.now().toUtc().difference(record.createdAt) > const Duration(days: 7)) {
      await repository.remove(record.id);
      return;
    }

    final conversationId = record.payload['conversation_id'] as String;
    final body = record.payload['body'] as String;

    try {
      await api.sendMessage(
        conversationId: conversationId,
        body: body,
        idempotencyKey: record.id, // 🔑 server dedupes on this
      );
      await repository.remove(record.id);
    } on ApiException catch (e) {
      if (e.isPermanent) {
        // 4xx (other than 401/408/429) — never going to succeed. Drop it.
        await repository.remove(record.id);
        return;
      }
      rethrow; // transient — let the dispatcher count the attempt and retry.
    }
  }
}
```

The handler — not the dispatcher — decides what counts as permanent. Different features have different rules: a `409 Conflict` on "create resource" might mean "already exists, success"; a `409` on "complete checkout" might mean "drop, the user already paid." The dispatcher cannot possibly know which is which, so it doesn't try.

### 5. The Dispatcher

The dispatcher is intentionally dumb. It reads, routes, catches, increments. That's it.

```dart
class OutboxDispatcher {
  OutboxDispatcher({
    required IOutboxRepository repository,
    required List<OutboxHandler> handlers,
  })  : _repository = repository,
        _handlersByType = {for (final h in handlers) h.type: h};

  final IOutboxRepository _repository;
  final Map<String, OutboxHandler> _handlersByType;

  Future<void>? _inFlight;

  /// Triggers a flush. Concurrent callers receive the same future — at most
  /// one flush runs at a time.
  Future<void> flush() {
    return _inFlight ??= _doFlush().whenComplete(() => _inFlight = null);
  }

  Future<void> _doFlush() async {
    final records = await _repository.getAll();
    if (records.isEmpty) return;

    for (final record in records) {
      final handler = _handlersByType[record.type];
      if (handler == null) {
        // No handler registered (feature was removed?). Skip — don't crash.
        continue;
      }

      try {
        await handler.handle(record);
      } on Exception {
        // Transient failure: leave the record, count the attempt, move on.
        await _repository.incrementAttempts(record.id);
      }
    }
  }
}
```

> [!IMPORTANT]
> **Coalesce concurrent flushes.** Without the in-flight mutex, a single connectivity-restored event arriving while the dispatcher is already working would cause every record to be replayed twice in parallel. Even with idempotency keys, that's wasteful at best and racy at worst (two concurrent `remove` calls). The `_inFlight` future is the simplest fix that works.

### 6. The Triggers

A flush should happen on every plausible "now would be a good time to retry" signal. The naive list is short:

- **App start** (cold or warm) — drain anything stuck from the last session.
- **Connectivity restored** — go from offline → online and the queue should drain.
- **App resumed** — the user came back to the app; their attention is here, so should the work be.
- **Right after enqueue** — the happy path. The user just tapped the button, the network is probably up, fire it now.

A small Flutter widget mounted at the router shell handles the first three:

```dart
class OutboxFlushTrigger extends ConsumerStatefulWidget {
  const OutboxFlushTrigger({required this.child, super.key});
  final Widget child;

  @override
  ConsumerState<OutboxFlushTrigger> createState() => _OutboxFlushTriggerState();
}

class _OutboxFlushTriggerState extends ConsumerState<OutboxFlushTrigger>
    with WidgetsBindingObserver {
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    WidgetsBinding.instance.addPostFrameCallback((_) => _flush());
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    if (state == AppLifecycleState.resumed) _flush();
  }

  void _flush() => ref.read(outboxDispatcherProvider).flush();

  @override
  Widget build(BuildContext context) {
    ref.listen<AsyncValue<bool>>(connectivityStatusProvider, (prev, next) {
      final wasOnline = prev?.value ?? false;
      final isOnline = next.value ?? false;
      if (!wasOnline && isOnline) _flush();
    });
    return widget.child;
  }
}
```

The fourth trigger — fire-on-enqueue — lives in the feature itself, typically inside the [Action](/docs/patterns/actions.md) that produced the record:

```dart
class SendMessageAction extends AutoDisposeNotifier<ActionState<void>>
    with ActionHandler<void> {
  @override
  ActionState<void> build() => const ActionIdle();

  Future<void> run({required String conversationId, required String body}) =>
      execute(() async {
        final action = SendMessageOutboxAction(
          id: const Uuid().v4(),
          conversationId: conversationId,
          body: body,
          createdAt: DateTime.now().toUtc(),
        );

        // 1. Persist intent. This is the commit point.
        await ref.read(outboxRepositoryProvider).add(action);

        // 2. Update local state immediately so the UI feels instant.
        ref.read(eventBusProvider).publish(MessageQueued(action));

        // 3. Best-effort: try the network now. If it fails, the dispatcher
        //    will pick the record up on the next trigger.
        await ref.read(outboxDispatcherProvider).flush();
      });
}
```

The user sees their message in the conversation immediately (step 2), regardless of whether the network call succeeds. From their perspective, sending is instantaneous — a property you cannot get from a naive write.

### 7. Composition Root (Riverpod)

```dart
final outboxRepositoryProvider = Provider<IOutboxRepository>((ref) {
  return SharedPreferencesOutboxRepository(
    ref.read(sharedPreferencesProvider),
    () => ref.read(currentUserIdProvider),
  );
}, name: 'outboxRepository');

final outboxHandlersProvider = Provider<List<OutboxHandler>>((ref) => [
      ref.watch(sendMessageOutboxHandlerProvider),
      ref.watch(completeOrderOutboxHandlerProvider),
      // Add new handlers here.
    ], name: 'outboxHandlers');

final outboxDispatcherProvider = Provider<OutboxDispatcher>((ref) {
  return OutboxDispatcher(
    repository: ref.read(outboxRepositoryProvider),
    handlers: ref.read(outboxHandlersProvider),
  );
}, name: 'outboxDispatcher');
```

The handler list is intentionally explicit. Adding a new outbox-backed action means touching this file — that's the trade-off for a statically-known, easily-tested set of participants.

## Why This Matters

### For offline-first architectures

An outbox _is_ the offline write story. Without it, "offline support" usually means "the screen renders cached data" — which is the read side. Writes require durability, and durability requires a queue.

- **The user's intent survives any failure mode.** Process kill, OS upgrade, dead battery, plane mode — the record is on disk before the request fires. Whatever happens to the network or the process, the next launch will re-attempt.
- **Optimistic UI is safe.** You can update local state the instant the action is enqueued because you _know_ the server will hear about it eventually. Without an outbox, optimistic updates are a lie that gets caught the moment the user looks at another device.
- **Reconciliation is centralized.** All your "what's stuck and should we retry it?" logic lives in one place. Every feature gets it for free instead of reinventing per-screen retry loops with their own subtly-different semantics.

### For high-stakes writes

Even an online-only app benefits from the outbox for the writes you cannot afford to lose:

- **Payments and purchases.** "We charged your card but didn't record it" is a class of bug that bankrupts trust. The outbox guarantees the record outlives the network call.
- **Anti-cheat / progress writes.** A user who lost their streak because the server didn't get the "completed" event will not stay a user for long. Idempotent retries fix this without server changes.
- **Sequenced writes.** Some flows have a strict order — "submit form, then upload attachments." Encoding that as two outbox actions (with the second waiting on the first to complete) gives you durable ordering for free.
- **Crash safety.** Even on a perfect network, the OS can kill your app between "send request" and "see response." Without an outbox, that window is a hole. With one, you simply re-attempt the next time the app is opened.

### Architectural benefits

- **The dispatcher is dumb.** Per-feature give-up logic stays in the handler, where it belongs. The core layer never grows feature-specific branches.
- **Features stay decoupled.** A new feature wanting offline-safe writes implements one handler and registers it. Nothing in the core has to change.
- **Testable in isolation.** Each handler is a unit test (`given a record with payload X, the API was called with Y, and the record was removed`). The dispatcher is also a unit test (`given two pending records, both handlers were invoked`). The repository swaps for an in-memory fake.

## Trade-offs to Consider

- **Eventual consistency.** The user's local view (optimistic) and the server's view diverge for a window of time. If a permanent failure later discards the record, you owe the user a "we couldn't send this" message and a way to undo or retry. Don't pretend everything is fine forever.
- **Idempotency is a server contract.** This pattern requires your backend to honor `Idempotency-Key` (or an equivalent `client_request_id` field). If duplicate retries cause duplicate side effects on the server, the entire "at-least-once" model collapses into "at-least-once means at-least-twice." Coordinate with your backend team before adopting.
- **Not for reads.** The outbox is write-only. For caching server data, use a normal repository with a stale-while-revalidate strategy.
- **Not for huge payloads.** A multi-megabyte file upload doesn't belong in `SharedPreferences`. Either store the file path and stream from disk in the handler, or use a real database with blob support.
- **Ordering is FIFO per queue, not per resource.** If you need strict ordering across two different action types (e.g. "create resource" then "update it"), encode the ordering inside one of the handlers — don't rely on the order the dispatcher iterates.
- **Storage growth requires a safety valve.** A stuck record (a `5xx` that never resolves, a server bug that returns transient errors forever) will accumulate `attemptCount` indefinitely. Drop records older than _N_ days, or after _M_ attempts, in the handler. Otherwise, you get a queue that never drains.

> [!TIP]
> **Pair with [Actions](/docs/patterns/actions.md) and the [Domain Event Bus](/docs/patterns/domain-event-bus.md).** The Action handles the user-facing operation. The outbox makes that operation durable. The event bus tells the rest of the app it happened. The three patterns compose: `Action.run()` → `outbox.add()` → `eventBus.publish(...)` → `dispatcher.flush()`. Each layer does one thing, and "the message I just sent" is reliably reflected in every screen that cares.
