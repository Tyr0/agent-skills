---
name: swift-ipc
description: Use this skill whenever the user asks about inter-process communication (IPC) in Swift or on Apple platforms, including XPC, NSXPCConnection, shared memory, mmap, POSIX shm, lock-free ring buffers, Mach messages, or cross-process synchronization. Also use it for questions about communicating between a main app and an extension, a helper process, or a system service, or when designing a high-throughput IPC channel. Triggers on 'how do I use XPC', 'how do I share memory between processes', 'how do I communicate with my app extension', 'what is the fastest IPC on Apple platforms', or any question about cross-process communication in Swift/ObjC.
---

# Swift IPC Reference

A dense reference for inter-process communication on Apple platforms — XPC, shared memory, and lock-free ring buffers.

---

## Scheduler and Memory Model for IPC

The Darwin scheduler treats threads across processes the same as threads within one process — same priority bands, same QoS classes, same boosting rules. The only difference is **memory isolation**: each process has its own virtual address space.

Synchronization primitives that work across thread boundaries within a process have the same semantics across process boundaries, with one practical exception: kernel-crossing overhead is higher for cross-process calls.

---

## XPC

XPC is Apple's recommended IPC mechanism. It is built on Mach messages (Darwin's lowest-level IPC primitive), providing:
- Automatic serialization of messages over a connection
- Process-level isolation and sandboxing
- Launchd integration for on-demand service activation
- Crash isolation — a crashed XPC service is restarted transparently

### `NSXPCConnection` — high-level ObjC/Swift API

```swift
// Client: connect to the service
let connection = NSXPCConnection(serviceName: "com.example.MyService")
connection.remoteObjectInterface = NSXPCInterface(with: MyServiceProtocol.self)
connection.resume()

// Async call (fire and forget)
let proxy = connection.remoteObjectProxy as! MyServiceProtocol
proxy.doWork(with: data) { result in
    // completion runs on an XPC-internal GCD queue
    DispatchQueue.main.async { self.handle(result) }
}

// Fault-tolerant proxy — returns nil proxy on connection failure instead of crashing
let safeProxy = connection.remoteObjectProxyWithErrorHandler { error in
    print("XPC connection error: \(error)")
} as! MyServiceProtocol
```

```swift
// Service side: export an object implementing the protocol
class MyServiceListener: NSObject, NSXPCListenerDelegate {
    func listener(_ listener: NSXPCListener,
                  shouldAcceptNewConnection connection: NSXPCConnection) -> Bool {
        connection.exportedInterface = NSXPCInterface(with: MyServiceProtocol.self)
        connection.exportedObject = MyServiceImpl()
        connection.resume()
        return true
    }
}
```

### Completion queue

Completions from XPC run on an internal GCD queue. Dispatch to your own queue before touching your state:

```swift
proxy.fetchData { [weak self] result in
    guard let self else { return }
    self.appQueue.async {    // hop to your queue before mutating state
        self.handleResult(result)
    }
}
```

### Synchronous XPC

```swift
// AVOID — blocks the calling thread waiting for the reply
let syncProxy = connection.synchronousRemoteObjectProxyWithErrorHandler { _ in } as! MyServiceProtocol
syncProxy.blockingCall()
```

**Never call synchronous XPC from the main thread or a high-priority thread.** It causes priority inversion and will produce UI jank or watchdog kills. Use async calls and completion handlers.

---

## `XPC` framework — lower-level C API

The lower-level `xpc_*` C API offers finer control over message types and lifecycle. Prefer `NSXPCConnection` for Swift — it is type-safe and integrates with Swift protocols.

---

## Shared Memory

For high-throughput IPC where XPC serialization overhead is too high, shared memory with `mmap` eliminates the kernel from the hot path entirely — reads and writes go directly to RAM.

### One-way read-only (POSIX `shm_open`)

```c
// Writer (creates and maps read-write)
int fd = shm_open("/myapp.data", O_CREAT | O_RDWR, 0600);
ftruncate(fd, size);
void *buf = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
// Write data into buf; notify the reader via XPC or Mach port

// Reader (maps read-only)
int fd = shm_open("/myapp.data", O_RDONLY, 0);
void *buf = mmap(NULL, size, PROT_READ, MAP_SHARED, fd, 0);
```

### Bidirectional: two regions, each process owns one

```
Process A                        Process B
writable  →  regionA  →  read-only (mapped by B)
read-only ←  regionB  ←  writable  (mapped by A)
```

Each process writes to its own region and reads the other's. **No lock is needed on the mapping itself.** Only the indices (head/tail) that indicate where valid data is require atomic synchronization.

---

## Lock-Free Ring Buffer over Shared Memory (SPSC)

Single-producer single-consumer (SPSC) ring buffer over shared memory: **zero locks in the hot path, zero syscalls per message**.

### Layout

```c
// Pad head and tail to separate cache lines — CRITICAL on Apple Silicon (128-byte lines)
typedef struct {
    _Atomic(uint64_t) head;     // written only by consumer; read by both
    uint8_t _pad0[120];         // isolate head from tail on Apple Silicon
    _Atomic(uint64_t) tail;     // written only by producer; read by both
    uint8_t _pad1[120];
    uint8_t data[CAPACITY];     // CAPACITY must be a power of 2
} RingBuffer;
```

### Producer

```c
uint64_t t = atomic_load_explicit(&rb->tail, memory_order_relaxed);   // own index
uint64_t h = atomic_load_explicit(&rb->head, memory_order_acquire);   // other's index
if (t - h < CAPACITY) {
    rb->data[t % CAPACITY] = byte;
    atomic_store_explicit(&rb->tail, t + 1, memory_order_release);    // publish
}
```

### Consumer

```c
uint64_t h = atomic_load_explicit(&rb->head, memory_order_relaxed);   // own index
uint64_t t = atomic_load_explicit(&rb->tail, memory_order_acquire);   // other's index
if (t > h) {
    uint8_t byte = rb->data[h % CAPACITY];
    atomic_store_explicit(&rb->head, h + 1, memory_order_release);    // consume
}
```

### Memory ordering rules

- Each side reads its own index with `.relaxed` (no other thread writes it).
- Each side reads the other's index with `.acquiring` — ensures the data written before the release is visible.
- Each side updates its index with `.releasing` — ensures data is fully written before the index is published.
- Write data **before** the releasing store on `tail`; read data **after** the acquiring load on `tail`.

### Cache line padding — mandatory

Apple Silicon has 128-byte cache lines. If `head` and `tail` share a cache line, every write by either side invalidates the line in the other core's L1 cache — **false sharing can degrade throughput by 10×**. Pad to 128 bytes between them.

### Waking the consumer without polling

Use a Mach port or `dispatch_source_t` of type `DISPATCH_SOURCE_TYPE_MACH_RECV` to notify the consumer when new data is available. This avoids busy-waiting and keeps CPU usage zero when the buffer is empty.

**Avoid cross-process semaphores** (`sem_open`) for the same priority-inversion reasons as in-process semaphores — they do not support priority boosting.

---

## Choosing the Right IPC Mechanism

| Mechanism | Throughput | Latency | Use when |
|---|---|---|---|
| `NSXPCConnection` (async) | Low-medium | ~50–200µs | Structured calls; sandboxed services; crash isolation needed |
| `NSXPCConnection` (sync) | Low | ~50–200µs | **Avoid** — blocks thread, causes priority inversion |
| Shared memory + ring buffer | Very high | <1µs hot path | Streaming telemetry, audio, video, sensor data |
| Mach messages (raw) | Medium-high | ~10–50µs | Custom protocol; tight latency budget; no need for ObjC ABI |
| UNIX domain sockets | Medium | ~10–100µs | Non-Apple platform compat; stream-oriented data |

---

## Anti-Patterns

| Anti-pattern | Risk | Fix |
|---|---|---|
| Synchronous XPC from main thread | UI hang; watchdog kill | Use async proxy with completion handler |
| Synchronous XPC from high-priority thread | Priority inversion; system may not service reply | Async XPC; never block on cross-process reply |
| Cross-process semaphore (`sem_open`) for wakeup | Priority inversion; no OS boosting | Use Mach port or dispatch source for notification |
| `head` and `tail` on same cache line in SPSC ring buffer | 10× throughput degradation (false sharing) | Pad to 128 bytes on Apple Silicon |
| `.relaxed` on the flag index that guards ring buffer data | Stale reads; consumer sees partial writes | `.releasing` store on writer's index; `.acquiring` load on reader's index |
| Writing data after the release store | Reader may observe index update before data is written | Write data first; then issue the release store |
| Shared memory without coordination on indices | Race condition | Only the indices need synchronization — use atomics |
| Using `mmap` file-backed shared memory for IPC | Disk I/O involvement; less predictable | Use `shm_open` for in-memory IPC; file-backed is for persistence |
