---
name: swift-synchronization
description: Use this skill whenever the user asks about Swift threading, synchronization, thread safety, race conditions, locks, mutexes, atomics, dispatch queues, semaphores, actors, priority inversion, deadlock, concurrency primitives, DispatchQueue, DispatchSemaphore, DispatchPrecondition, OSAllocatedUnfairLock, os_unfair_lock, Swift Mutex, Swift Atomic, or the Synchronization module. Also use it for code reviews where threading or synchronization correctness is a concern, or when a user asks 'is this thread-safe', 'how do I protect this state', or 'why is this deadlocking'. For XPC, shared memory, and inter-process communication use the swift-ipc skill instead.
---

# Swift Synchronization Reference

A dense reference for writing correct, performant concurrent Swift. Covers primitives, patterns, anti-patterns, and cross-process communication.

## Core Philosophy: Know Where You Came From, and Where You're Going

The most powerful synchronization tool is eliminating the need for it. Before reaching for a lock, ask: can I structure this so that a single thread owns this state entirely?

**The DispatchQueue ownership pattern** achieves this by giving each class a queue it owns, and requiring all callers to be on that queue before calling in. This replaces defensive locking with a design contract enforced at the API boundary:

```swift
final class NetworkMonitor {
    private let queue: DispatchQueue

    init(queue: DispatchQueue) {
        self.queue = queue
    }

    func start() {
        // Callers must be on `queue` — enforced at runtime
        dispatchPrecondition(condition: .onQueue(queue))
        // ... safe to touch all mutable state here, no locks needed
    }

    private func handleEvent(_ event: Event) {
        // Internal methods can also assert — useful during development
        dispatchPrecondition(condition: .onQueue(queue))
        // ...
    }
}
```

**Key principles:**
- Know which queue you are on when entering any function (where you came from)
- Know which queue requirement every class you call out to has (where you're going)
- Avoid thread-hopping: if you're already on the right queue, stay there
- Extend this to third-party types: `URLSession` accepts a `delegateQueue` — set it to your queue at init time, treat it as immutable, and assert on it

**Queue minimalism** — keep the number of queues small and purposeful:

| Queue | Purpose |
|---|---|
| `DispatchQueue.main` | UI updates, user-facing state |
| App serial queue | Central app/business logic state (often one shared queue) |
| Database queue | Long-lived read/write operations with strict isolation |
| Networking queue | URLSession delegate, response parsing |
| Workloop (`makeCurrentWorkloopDispatchQueue`) | Urgent fixed-priority I/O or timer work |

Prefer a shared app queue for the bulk of your code. Proliferating queues increases the chance of accidental hops between them.

**Completion handlers with `Result<T, E>`** — mutual exclusivity of success and failure is structurally guaranteed by the type:

```swift
// Callers get exactly one callback, exactly one outcome
func fetch(completion: @escaping (Result<Data, NetworkError>) -> Void) {
    // Implemented on queue; calls completion on queue
    dispatchPrecondition(condition: .onQueue(queue))
    // ...
}

// async/await convenience layer on top — fine for callers who want it
func fetch() async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        fetch { continuation.resume(with: $0) }
    }
}
```

---

## Primitive Selection Guide

| Primitive | Overhead | Use when |
|---|---|---|
| `Atomic<T>` | ~1–5ns, lock-free | Single numeric/bool flag; trivial state; limited side effects |
| `OSAllocatedUnfairLock` | ~5–10ns | Small critical section; Apple platforms; needs priority inheritance |
| `Swift Mutex` | ~5–10ns | Small critical section; cross-platform or Swift 6+ codebases |
| `DispatchQueue` (serial, ownership pattern) | ~50–200ns dispatch | Business logic classes with complex state; natural async work |
| Actor | ~50–200ns await | Swift-native async APIs; when callers expect async interface |
| `DispatchSemaphore` | — | **Avoid** — see §Anti-Patterns |

**Decision flow:**
1. Can this state be owned by a single queue? → DispatchQueue ownership pattern
2. Is the shared state a single integer/bool/pointer with no associated work? → `Atomic<T>`
3. Is this a small model type needing mutual exclusion? → `Mutex` or `OSAllocatedUnfairLock`
4. Is this an async-native API for external callers? → Actor (with `nonisolated` on anything that doesn't touch state)

---

## 1. Memory Ordering

Memory ordering controls how the compiler and CPU reorder memory operations around an atomic. Getting this wrong produces races that are non-deterministic, reproduce rarely, and are invisible in debug builds.

| Ordering | Guarantees | Use when |
|---|---|---|
| `.relaxed` | Atomicity only; no ordering constraints | Counter increments where the count value is not used to make decisions about other memory |
| `.acquiring` | No load can be reordered before this | Reading a flag that, when true, means other data is ready to read |
| `.releasing` | No store can be reordered after this | Writing data first, then setting a flag to signal it's ready |
| `.acquiringAndReleasing` | Both acquire + release | Read-modify-write (e.g., atomic swap, compare-exchange) |
| `.sequentiallyConsistent` | Total global order across all threads | Rarely needed; use when acquiring/releasing is too subtle and correctness matters more than throughput |

**The acquire/release pair** is the fundamental synchronization handshake:

```swift
// Producer (on one thread):
data.value = 42                        // prepare data
readyFlag.store(true, ordering: .releasing)   // publish: nothing after here can precede the store

// Consumer (on another thread):
while !readyFlag.load(ordering: .acquiring) { } // acquire: nothing after here can precede this load
print(data.value)  // safe — sees 42
```

Without the `.releasing` store and `.acquiring` load, the CPU may reorder the data write after the flag write (or the data read before the flag read), producing stale values.

**Default: prefer `.acquiringAndReleasing` for any RMW operation and `.sequentiallyConsistent` when in doubt.** Pay the ordering cost only when profiling shows it matters.

---

## 2. `os_unfair_lock` and `OSAllocatedUnfairLock`

`os_unfair_lock` is the lowest-overhead mutex on Apple platforms (~CPU lock instructions only; spins briefly before parking). The OS tracks ownership, so it resolves priority inversion automatically: if a high-priority thread waits on a lock held by a low-priority thread, the OS temporarily boosts the lower thread's priority.

**`os_unfair_lock` is unsafe to use directly in Swift** — it is a value type and Swift does not guarantee a stable memory address for value types. If the lock moves, the OS's ownership tracking breaks. Use `OSAllocatedUnfairLock` instead:

```swift
import os

// OSAllocatedUnfairLock: heap-allocated, stable address, Swift-safe (macOS 13+/iOS 16+)
// Optionally wraps a protected value directly in the initializer:
let lock = OSAllocatedUnfairLock(initialState: MyModel())

lock.withLock { model in
    model.count += 1
}

let snapshot = lock.withLockUnchecked { $0 }  // when T is not Sendable but usage is safe
```

**Properties:**
- Non-recursive: locking twice on the same thread traps
- Non-FIFO ("unfair"): no queue of waiters; a woken thread may lose to a newly arriving thread
- Same-thread lock/unlock required: unlocking from a different thread traps
- Priority inheritance: automatic via OS ownership tracking

---

## 3. Swift `Mutex` (Swift 6+, SE-0433)

`Mutex` is `OSAllocatedUnfairLock` made cross-platform and idiomatic. Uses `os_unfair_lock` on Apple platforms. Available in the `Synchronization` module (iOS 18+/macOS 15+; back-deployable via swift-synchronization package).

```swift
import Synchronization

// Mutex wraps its protected value — compiler prevents accessing value outside withLock
let mutex = Mutex(MyModel())

mutex.withLock { model in
    model.count += 1
}
```

**Key properties:**
- `~Copyable` — cannot be accidentally copied (important: a copied lock is a broken lock)
- `Sendable` — safe to share across concurrency boundaries
- Closure-based API prevents forgetting to unlock
- Same non-recursive, non-FIFO, ownership-tracked semantics as `OSAllocatedUnfairLock`

**Prefer `Mutex` for new Swift 6+ code.** Use `OSAllocatedUnfairLock` when targeting macOS 12–/iOS 15– or when you need the `withLockUnchecked` escape hatch.

**Lock discipline (applies to all mutex types):**
Never call out to external clients while holding a lock. "External" means code outside the class that owns the lock — completion handlers, delegates, protocol methods on injected objects. This is the primary cause of deadlock in practice:

```swift
// DEADLOCK RISK — calling external delegate while locked
mutex.withLock { state in
    state.process()
    delegate?.didFinish()  // delegate may try to re-enter this lock
}

// SAFE — capture value, release lock, then call out
let result = mutex.withLock { state in state.process() }
delegate?.didFinish(result)  // called after lock is released
```

---

## 4. Swift `Atomic<T>` (Swift 6+, SE-0440)

Lock-free atomic operations on integer types, booleans, and raw representable enums. From the `Synchronization` module.

```swift
import Synchronization

let counter = Atomic<Int>(0)
counter.add(1, ordering: .relaxed)

let flag = Atomic<Bool>(false)
flag.store(true, ordering: .releasing)
let wasSet = flag.load(ordering: .acquiring)

// Compare-exchange: only stores newValue if current == expected
let (exchanged, original) = flag.compareExchange(
    expected: false,
    desired: true,
    ordering: .acquiringAndReleasing
)
```

**`Atomic` cannot protect compound state** — if you need to atomically update two fields together, use a `Mutex`. Use `Atomic` only when one value fully captures the shared state and the side effects of mutation are self-contained.

**Good candidates:** generation counters, cancellation flags, reference counts, connection state enums, once-initialization guards.

---

## 5. DispatchQueue Ownership Pattern (Deep Dive)

A serial `DispatchQueue` is already a mutex: only one block executes at a time. The ownership pattern makes this explicit and auditable:

```swift
final class SessionManager {
    private let queue: DispatchQueue
    private var sessions: [UUID: Session] = [:]

    init(queue: DispatchQueue) {
        self.queue = queue
    }

    // Public API — callers must arrive on `queue`
    func add(_ session: Session) {
        dispatchPrecondition(condition: .onQueue(queue))
        sessions[session.id] = session
    }

    func remove(id: UUID) {
        dispatchPrecondition(condition: .onQueue(queue))
        sessions.removeValue(forKey: id)
    }
}
```

**Setting up a URLSession to call back on your queue:**
```swift
let session = URLSession(
    configuration: .default,
    delegate: self,
    delegateQueue: OperationQueue(underlyingQueue: appQueue)  // delegate called on appQueue
)
// Now URLSessionDelegate methods are guaranteed on appQueue — no hopping needed
```

**Serial vs concurrent queues:**

| Type | Use case | Notes |
|---|---|---|
| Serial | Owning mutable state | Default choice; acts as a mutex |
| Concurrent + barrier | Read-heavy shared state (reader-writer lock) | `sync(flags: .barrier)` for writes; `sync` for reads |
| Target queue | Grouping QoS; routing to workloop | Set via `init(target:)` |

**`dispatchPrecondition` variants:**
```swift
dispatchPrecondition(condition: .onQueue(queue))       // must be on this queue
dispatchPrecondition(condition: .notOnQueue(queue))    // must NOT be on this queue (re-entrancy guard)
dispatchPrecondition(condition: .onQueueAsBarrier(queue)) // must be running as a barrier
```

Use `.notOnQueue` on any function that would deadlock if called re-entrantly from the same queue.

---

## 6. Actors — Honest Assessment

Swift actors provide automatic mutual exclusion over their state and integrate naturally with `async/await`. But the user's instinct is correct: **actors make context-switching invisible.**

**What happens on every actor method call from outside the actor:**

```swift
actor DataStore {
    var cache: [String: Data] = [:]
    func insert(_ key: String, _ value: Data) { cache[key] = value }
}

// From a non-isolated context:
await store.insert("key", data)
// ↑ This always hops to the actor's executor — you cannot see or control which thread
```

Every `await` on an actor from a non-isolated context is a context switch. In a loop this is O(n) hops — the same problem as the actor-hopping section in the performance skill.

**The DispatchQueue pattern gives you explicit control; actors give you implicit safety.**

| | DispatchQueue pattern | Actor |
|---|---|---|
| Visibility of thread hops | Explicit — `dispatchPrecondition` makes it auditable | Implicit — every `await` is a potential hop |
| Integration with legacy sync code | Natural — completion handlers, delegates | Requires care — cannot `await` from sync context |
| Priority / QoS control | Explicit via `DispatchQoS` | Inherited from caller; limited explicit control |
| Priority inversion | Resolved by OS (unfair lock semantics) | Runtime provides some protection; less explicit |
| Debugging | Queue names show in stack traces | Actor type shows; harder to trace hop chains |
| Best for | Large "business logic" classes; legacy codebases | Swift-native async APIs; Swift 6 strict concurrency |

**`nonisolated` is essential:** any actor method that doesn't touch the actor's mutable state should be `nonisolated`. A non-isolated call requires no `await` and causes no hop:

```swift
actor Processor {
    var results: [String] = []

    func store(_ result: String) { results.append(result) }   // isolated: hops in

    nonisolated func buildRequest(path: String) -> URLRequest { ... }  // no hop; synchronous
}
```

**Verdict:** Actors are the right choice when building async-native APIs in Swift 6+ codebases and when the caller expects `async` interfaces. For classes with complex internal state, callbacks, and delegate patterns — especially in mixed ObjC/Swift or pre-Swift 6 codebases — the DispatchQueue ownership pattern gives more control and auditability.

---

## 7. Anti-Patterns

### DispatchSemaphore.wait() in Async Contexts

The most dangerous synchronization anti-pattern on Apple platforms:

```swift
// CATASTROPHIC — do not do this
func fetchSync() -> Data {
    let sema = DispatchSemaphore(value: 0)
    var result: Data!
    Task {
        result = await download()
        sema.signal()          // this task needs a thread to run on
    }
    sema.wait()                // blocks the current thread
    return result
}
```

**Why it can deadlock or stall:**

1. **Priority inversion** — the blocked high-priority thread waits on work that may be scheduled at a lower priority. In iOS low-power mode, the system may not service lower-priority work at all.
2. **Thread pool starvation** — Swift Concurrency's cooperative thread pool is sized to the number of CPU cores. Each `wait()` burns a thread without notifying the runtime. If all threads block, `signal()` never runs — there's no thread to run it on.
3. **Loss of priority boosting** — in the DispatchQueue world, the OS boosts the running thread's priority when a higher-priority thread blocks waiting on it. The async Task has no thread identity to boost.

**The same issue applies to `DispatchGroup.wait()`** called from within an async or actor context.

**Fix:** Either stay in the async world end-to-end, or implement the underlying work synchronously with a completion handler and wrap it:

```swift
// Synchronous completion-based implementation on known queue
func fetch(completion: @escaping (Result<Data, Error>) -> Void) {
    dispatchPrecondition(condition: .onQueue(queue))
    // ... real implementation
}

// Async shim — suspension is cooperative, no thread blocked
func fetch() async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        fetch { continuation.resume(with: $0) }
    }
}
```

### Re-entering a Serial Queue (Sync Deadlock)

```swift
queue.sync {              // blocks calling thread, waiting for queue
    queue.sync { }        // queue is already blocked — deadlock
}
```

Use `dispatchPrecondition(condition: .notOnQueue(queue))` at the top of any function that internally calls `queue.sync(...)` to catch re-entrancy early.

---

## 8. Common Anti-Patterns

| Anti-pattern | Risk | Fix |
|---|---|---|
| `DispatchSemaphore.wait()` in async context | Deadlock; thread pool starvation | Use `withCheckedContinuation` or restructure as async |
| `DispatchGroup.wait()` from async/actor | Same as above | Use `withTaskGroup` or async notification |
| `queue.sync { queue.sync { } }` | Deadlock | `dispatchPrecondition(condition: .notOnQueue(queue))` at entry |
| Calling delegate/callback while holding a lock | Deadlock via re-entrancy | Capture result, release lock, then call out |
| `os_unfair_lock` as a value type in Swift | Memory corruption | Use `OSAllocatedUnfairLock` or `Mutex` |
| Locking recursively with `OSAllocatedUnfairLock` / `Mutex` | Trap / deadlock | Design to avoid re-entrancy; use a flag if needed |
| `Atomic` protecting compound state (two fields) | Partial update / tearing | Use `Mutex` for anything requiring multi-field atomicity |
| `@MainActor` on entire compute-heavy class | Serializes all work on main thread | Mark compute methods `nonisolated` |
| Hopping through multiple actors in a loop | O(n) context switches | Batch work; cross actor boundary once |
| `.relaxed` ordering on a flag that guards other data | Stale reads; race condition | `.releasing` store, `.acquiring` load |
| `weak` where `unowned` suffices in hot-path lock-protected delegate | Side table hop per call | `unowned` (verify lifetime) |
