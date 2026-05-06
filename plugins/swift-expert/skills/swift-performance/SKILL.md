---
name: swift-performance
description: Use this skill whenever the user asks about Swift performance, optimization, low-level Swift patterns, runtime cost, memory usage, dispatch overhead, ARC, copy-on-write, existentials, value types, reference types, move-only types, noncopyable, ~Copyable, or how to write faster Swift code. Also triggers on questions about specific Swift types like Array, Dictionary, Data, String, Optional, or actor/concurrency performance. Use it for code reviews where performance is a concern, or when a user asks 'why is this slow' or 'how do I make this faster' in a Swift context.
---

# Swift Performance Reference

A dense reference for writing high-performance Swift. Covers the full cost model from dispatch through memory layout.

## Optimization Priority Order

Work top-to-bottom; stop when profiling shows the bottleneck is gone.

1. Eliminate dynamic dispatch (`final`, generics over `any`, WMO)
2. Eliminate heap allocation (structs, avoid boxing, stack closures)
3. Reduce ARC traffic (value types, `unowned`, `inout`, `~Copyable` for single-owner types)
4. Eliminate unnecessary copying (`inout`, CoW correctness)
5. Enable cross-module optimization (`@inlinable`, WMO)
6. Type-level wins (`ContiguousArray`, `StaticString`, `@frozen`, struct field ordering)

**Always profile in release mode (`-O`) with Instruments before optimizing. Many issues the optimizer silently eliminates in release builds look expensive in debug.**

---

## 1. Method Dispatch

| Mechanism | Cost | When |
|---|---|---|
| Direct call | ~0, inlineable | `struct` methods; `final` class methods; `private`/`fileprivate` class members |
| vtable | 1 pointer + no inlining | Non-`final` class methods (Swift default) |
| Witness table | 2 pointers + no inlining | `any Protocol` method calls via existential |
| ObjC message send | Hash lookup | `dynamic` keyword |

The real cost of dynamic dispatch is not the indirection itself — it's that it **blocks inlining, specialization, and ARC optimization** at the call site.

**`final`** — single most impactful keyword. Prevents subclassing/overriding; compiler emits a direct call. Apply to entire classes or individual methods/properties.

**`private` / `fileprivate`** — same devirtualization effect as `final` within the file; no promise needed about the class hierarchy.

**Whole Module Optimization (WMO)** — compiles the entire module as one unit. Devirtualizes all `internal` declarations (Swift's default), enables cross-file inlining, generic specialization, and dead function elimination. Enable for all release builds. Since `internal` is the default access level, WMO effectively treats everything internal as `final` at no source change cost.

**`dynamic`** — forces ObjC message send. Only use when ObjC runtime features (KVO, swizzling) explicitly require it.

---

## 2. Existentials (`any Protocol`)

An existential container has three components:
- **3-word inline value buffer** — stores the value if it fits (≤24 bytes on 64-bit); otherwise heap-allocates it and stores a pointer
- **Value Witness Table (VWT)** — pointer to lifecycle functions (copy, move, destroy)
- **Protocol Witness Table (PWT)** — pointer to protocol method implementations

Every `any Protocol` method call goes through the PWT; this indirection cannot be inlined.

```swift
// Slow — VWT + PWT dispatch; heap alloc if Line > 24 bytes
func draw(_ shape: any Drawable) { shape.draw() }

// Fast — compiler specializes per concrete type; static dispatch
func draw<T: Drawable>(_ shape: T) { shape.draw() }

// Also fast — one concrete type per call site; statically dispatched
func draw(_ shape: some Drawable) { shape.draw() }
```

**`any` is fine for:** heterogeneous collections, non-hot API boundaries, type-erased wrappers, configuration objects.  
**Avoid `any` in:** tight loops, high-frequency dispatch, protocol methods on large value types.

**Enum instead of existential for closed sets** — zero VWT/PWT overhead, no heap alloc, just an integer tag:

```swift
enum Shape { case circle(Double), rect(Double, Double) }  // vs any Drawable
```

**Quantified overhead:** 1M large-struct existential calls ≈ +3ms vs generics. Matters in tight loops, not in UI code.

**Typed throws (`throws(E)`) — Swift 6** — `throws` uses an existential error box at the call site. `throws(MyError)` avoids that box: the error type is statically known, so no heap allocation, no VWT overhead:

```swift
// Untyped: error is existential (any Error) — heap-boxed at throw site
func parse(_ s: String) throws -> Int { ... }

// Typed: error is inline value — zero existential overhead
func parse(_ s: String) throws(ParseError) -> Int { ... }
```
Use typed throws in hot paths that propagate errors frequently (e.g., parsers, decoders).

---

## 3. Heap Allocation

Stack allocation = decrement stack pointer. Heap allocation = lock shared allocator, find free block, zero memory + matching `free()` + ARC overhead for lifetime.

**What triggers a heap allocation:**

| Trigger | Notes |
|---|---|
| `class` instance | Always |
| Value type > 24 bytes in `any Protocol` | Existential buffer spills to heap |
| `@escaping` closure captures | Context object heap-allocated + ARC-managed |
| `var` captured by `@escaping` closure | Boxing into a heap ref cell so both owner and closure share it |
| `indirect enum` case | Associated value behind a heap pointer |
| `String` > 15 UTF-8 bytes | Buffer heap-allocated (CoW-managed) |
| Array / Dictionary / Set | Backing buffer always heap (CoW-managed) |
| Value type stored in `class` property | Lives in the class's heap allocation |

**`inout` prevents variable boxing** — if a `var` is only mutated via non-escaping closures, using `inout` keeps it on the stack:

```swift
// BAD — 'count' gets heap-boxed if captured by escaping closure
var count = 0

// GOOD — stays on stack; no box, no ARC
func tally(_ items: [Int], into result: inout Int) {
    items.forEach { result += $0 }  // forEach is non-escaping
}
```

---

## 4. ARC (Reference Counting)

Retain/release are **atomic operations** — they require memory-bus synchronization. Expensive on contended multicore code.

**Minimize class-typed fields in hot-path structs.** A struct with N class references generates N retain/release calls per copy:

```swift
// 2 retains + 2 releases per copy (String = heap ref)
struct Address { var street: String; var city: String }

// 0 ARC per copy — replace fixed-domain strings with typed alternatives
enum AddressType { case home, work }
struct Address { var id: UUID; var type: AddressType }  // UUID = 16 bytes, no heap
```

**`weak` vs `unowned` vs `unowned(unsafe)`:**

| | Access cost | Safety |
|---|---|---|
| `weak` | Side table hop on every load + optional unwrap | Nil on dealloc; always safe |
| `unowned` | Direct pointer; no side table | Trap on dealloc access |
| `unowned(unsafe)` | Zero overhead | UB on dealloc; programmer responsibility |

Use `unowned` over `weak` when the referenced object is guaranteed to outlive the reference. `unowned(unsafe)` removes all overhead but produces undefined behavior on dealloc access — only reach for it in unsafe Swift contexts where you are already managing lifetime manually and the cost of `unowned` is measurable.

**Escaping closures cause heap allocation** for their capture context + retain/release for every captured reference. Non-escaping closures (most stdlib HOFs: `map`, `filter`, `forEach`) can have their context stack-allocated.

---

## 4a. `~Copyable` — Move-Only Types (Swift 5.9+)

The ultimate ARC escape hatch: suppress copying at the type level. A `~Copyable` type has exactly one owner at any given time; ownership is *moved*, not *copied*, so the compiler never inserts retain/release for it.

```swift
// ~Copyable structs (and enums) may define deinit — unlike regular structs.
// Single ownership guarantees deinit runs exactly once, just like a class.
struct FileDescriptor: ~Copyable {
    let fd: Int32
    consuming func close() { Darwin.close(fd) }
    deinit { Darwin.close(fd) }  // guaranteed exactly once; invalid on Copyable structs
}

func process(_ f: consuming FileDescriptor) {
    // f is moved into this scope; caller can no longer use it
    f.close()
}
```

**Key terms:**
- `consuming` parameter/method — takes ownership; caller's binding is invalidated after the call
- `borrowing` parameter — read-only access without ownership transfer; zero copy, zero ARC
- `inout` — mutable borrow; exclusive access, no copy

**Performance impact:**
- Zero retain/release for the `~Copyable` value itself
- Ideal for wrappers around OS resources, cryptographic keys, buffers, or any single-owner handle
- Cannot be stored in generic collections that require `Copyable` (the default constraint); use with care in data structures

**When to reach for it:** when profiling shows ARC traffic on a specific type that is logically single-owned (file handles, locks, network connections, arena-allocated nodes).

---

## 5. Copy-on-Write (CoW)

`Array`, `Dictionary`, `Set`, `String` share their buffer until mutation. The mutation check uses `isKnownUniquelyReferenced()` — inspects whether the buffer's strong refcount == 1. If 1: mutate in place. If > 1: copy first.

**Anti-patterns that silently defeat CoW:**

```swift
// BAD — the +1 retain at the call site makes refcount = 2; copy triggered
func appendOne(_ a: [Int]) -> [Int] { var a = a; a.append(1); return a }
var arr = [1, 2, 3]
arr = appendOne(arr)  // copies even though result is immediately reassigned

// GOOD — inout passes a direct reference; refcount stays 1
func appendOne(_ a: inout [Int]) { a.append(1) }
appendOne(&arr)
```

Modifying a single element of a shared array copies the **entire** buffer — there is no partial CoW.

**Custom CoW pattern:**

```swift
final class Storage<T> { var value: T; init(_ v: T) { value = v } }

struct Box<T> {
    private var _s: Storage<T>
    init(_ x: T) { _s = Storage(x) }
    var value: T {
        get { _s.value }
        set {
            if !isKnownUniquelyReferenced(&_s) { _s = Storage(newValue); return }
            _s.value = newValue
        }
    }
}
```

**`reserveCapacity` before batch appends** — eliminates O(log n) geometric reallocations:

```swift
var result = [Int]()
result.reserveCapacity(items.count)
items.forEach { result.append(transform($0)) }
```

---

## 6. `inout`

Passes a direct reference (pointer) to caller's storage. Two distinct uses:
1. **Avoid copying large value types** across a call boundary
2. **Prevent heap boxing** of captured `var`s (see §3)

The Law of Exclusivity: the caller's variable is frozen for the duration of the call. Swift enforces this statically — no locking needed.

---

## 7. Collections

**`ContiguousArray<T>`** — for value-type elements, behaves the same as `Array` (both are already contiguous; no bridge). For class or `@objc` elements, `Array` may store an `NSArray` under the hood; `ContiguousArray` explicitly opts out of that bridge, giving faster indexing and no ObjC overhead. Prefer in performance-critical code when elements are class types or when bridging must be prevented.

**Avoiding `Data`** — `Foundation.Data` carries `NSData` bridge overhead. Alternatives:

| Type | Use case |
|---|---|
| `[UInt8]` / `ContiguousArray<UInt8>` | Mutable byte buffer; CoW; no ObjC bridge |
| `UnsafeRawBufferPointer` | Zero-overhead read-only view over any contiguous storage |
| `UnsafeMutableRawBufferPointer` | Zero-overhead read-write view; use via `withUnsafeMutableBytes` |

```swift
// Zero-copy read from Data or Array
data.withUnsafeBytes { (buf: UnsafeRawBufferPointer) in
    let header = buf.load(fromByteOffset: 0, as: MyHeader.self)
}
```

**Bounds-check-free iteration** — `withUnsafeBufferPointer` provides a raw pointer view; indexed access has no bounds checks inside the closure:

```swift
array.withUnsafeBufferPointer { buf in
    for i in 0..<buf.count { process(buf[i]) }
}
```

**Dictionary key cost** (ascending): `Int`/`enum` ≈ 1ns < `UUID` ≈ 5ns < `String` ≈ 50ns. Use typed enums or integer keys in hot-path lookups.

**Lazy sequences** — `.lazy` before `map`/`filter` eliminates intermediate array allocations, computing each element on demand:

```swift
// Eager: 3 intermediate arrays allocated
array.filter { $0 > 0 }.map { $0 * 2 }.prefix(10)

// Lazy: zero intermediate allocations, one pass
array.lazy.filter { $0 > 0 }.map { $0 * 2 }.prefix(10)
```

Lazy pitfalls:
- **No caching** — iterating a lazy result twice recomputes all closures twice
- `LazyFilterCollection.endIndex` requires scanning ahead; `prefix(n)` on a lazy filter can be slower than eager for small collections
- Performance degrades significantly in debug builds (optimizer disabled)
- Prefer `first(where:)` over `.lazy.filter { }.first` — stdlib method has internal optimizations

---

## 8. Inlining and Cross-Module Optimization

The Swift optimizer inlines aggressively within a module. Inlining enables constant folding, dead code elimination, and ARC elision at call sites. Across module boundaries, this visibility is lost by default.

**`@inlinable`** — exports the function body as part of the module's public interface. Callers in other modules can inline it:

```swift
@inlinable public func clamp<T: Comparable>(_ v: T, lo: T, hi: T) -> T {
    v < lo ? lo : (v > hi ? hi : v)
}
```

- Use for small, hot public functions (<10 lines)
- Locks implementation as public ABI — changing the body is a breaking change for optimized callers
- Internal dependencies must be `@usableFromInline`

**`@inline(__always)` / `@inline(never)`** — optimizer hints (not guaranteed). Not part of stable Swift language; use `@inlinable` instead for supported cross-module inlining. `@inline(never)` is useful for error/slow paths to keep hot code compact in the instruction cache.

**`@usableFromInline`** — marks `internal` declarations as accessible from `@inlinable` code without promoting them to public API.

---

## 9. `@frozen`

Promises the compiler that a public `enum`'s cases or `struct`'s stored properties will never change. Primary benefits:

- **Enum**: compiler uses a jump table for `switch`; layout exposed to client modules for optimization; exhaustive `switch` without `default`
- **Struct**: memory layout fixed; direct field access from client code; no runtime indirection layer

Without `@frozen` (library evolution mode), the compiler must assume future cases/properties can be added — switch statements need a `default`, and layout cannot be exposed to clients.

In **application code** (not binary frameworks), all types are effectively frozen from the compiler's perspective. `@frozen` is mainly relevant when shipping a binary framework or Swift package with `-enable-library-evolution`.

---

## 10. Struct Memory Layout

Swift lays out struct fields **in declaration order**. Each field is aligned to its natural alignment. Misaligned fields require padding bytes, increasing `stride` and array memory footprint.

```swift
// Wastes 7 bytes — Bool(1) then Int(8) requires 7 bytes padding before Int
struct Bad  { var flag: Bool; var value: Int }  // size=16, stride=16

// No internal waste — Int(8) then Bool(1), 7 bytes trailing padding only
struct Good { var value: Int; var flag: Bool }  // size=9,  stride=16
```

**Rule: order fields from largest alignment to smallest.**

Verify with `MemoryLayout<T>.stride` — that's what an `Array<T>` uses per element.

**Replace `String` fields with typed alternatives:**

```swift
// BAD — 2 heap-allocated strings per copy, ARC overhead
struct Packet { var type: String; var id: String }

// GOOD — zero heap alloc, zero ARC, more type-safe
enum PacketType: UInt8 { case data, ack, nak }
struct Packet { var type: PacketType; var id: UUID }  // UUID = 16 bytes inline
```

---

## 11. Strings

**Small string optimization** — strings ≤15 UTF-8 bytes stored inline in the `String` struct; no heap allocation.

**`StaticString`** — stores only a pointer into the binary. Zero heap alloc, zero ARC, zero CoW. Use for compile-time constants: log keys, format strings, C-interop labels.

```swift
let tag: StaticString = "network.request"  // pointer to binary data, no alloc
```

Do **not** use `StaticString` in string interpolation — it implicitly bridges to `String`, allocating.

**Character iteration is expensive** — `String.characters` performs Unicode grapheme cluster segmentation (heap allocs per `Character`). For parsing:

```swift
for byte in str.utf8 { ... }          // fastest: raw integers
for scalar in str.unicodeScalars { }  // fast: simple Unicode scalars
for ch in str { }                     // slow: full grapheme cluster segmentation
```

**`Substring`** — zero-copy slice sharing the parent's buffer. Convert to `String` when the parent can be released (holding a `Substring` retains the entire original buffer).

**Concatenation in loops:**

```swift
// O(n²) — new allocation per +
var r = ""; for s in parts { r += s }

// O(n) — one allocation
let r = parts.joined(separator: "")

// O(n) — in-place append
var r = ""; r.reserveCapacity(estimate)
for s in parts { r.append(contentsOf: s) }
```

---

## 12. Swift Concurrency Performance

**Actor hopping** — each `await` crossing an actor boundary is a context switch. In a loop, this is O(n) context switches:

```swift
// BAD — 100 hops between DB actor and MainActor
for id in ids { let u = await db.load(id); users.append(u) }

// GOOD — 1 hop: batch the work, cross the boundary once
let all = await db.loadAll(ids: ids)
users.append(contentsOf: all)
```

**`nonisolated`** — methods that don't touch actor state should be `nonisolated`. Callers invoke them synchronously without `await`, eliminating the context switch entirely:

```swift
actor Processor {
    var cache = [String: Data]()
    func fetch(key: String) -> Data? { cache[key] }  // isolated: needs actor
    nonisolated func buildURL(path: String) -> URL { ... }  // no actor needed
}
```

**`@MainActor` over-annotation** — annotating an entire class `@MainActor` serializes all its async work on the main thread. Mark only the properties/methods that genuinely touch UI. Mark pure-compute methods `nonisolated`.

**Structured vs unstructured tasks** — `withTaskGroup` child tasks are lightweight and runtime-optimized. Unstructured `Task { }` heap-allocates a new task object per call:

```swift
// GOOD — structured parallelism
await withTaskGroup(of: Int.self) { group in
    for item in items { group.addTask { process(item) } }
}

// MORE OVERHEAD — independent heap-allocated tasks per iteration
for item in items { Task { await process(item) } }
```

**`withDiscardingTaskGroup` (Swift 5.9+)** — like `withTaskGroup` but discards child results immediately instead of buffering them. Use when child tasks produce side effects only and you don't need to collect return values. Avoids accumulating results in memory:

```swift
// GOOD for fire-and-forget parallel work — no result buffer allocated
await withDiscardingTaskGroup { group in
    for item in items { group.addTask { await store(item) } }
}
```

**`Sendable` and value types** — structs/enums with `Sendable` fields conform automatically at zero runtime cost. Prefer passing `Sendable` value types across actor boundaries instead of class instances to avoid ref-counting synchronization.

---

## 13. Build Flags

| Flag | Effect |
|---|---|
| `-Onone` | Debug: no optimization, full debug info |
| `-O` | Release: inlining, specialization, dead code elimination, ARC elision |
| `-Osize` | Release: optimize for binary size over speed; disables large function inlining |
| `-whole-module-optimization` | Compile entire module as one unit; highest-leverage single flag for release |
| `-enable-library-evolution` | Binary framework ABI stability; non-`@frozen` public types become conservative |

---

## 14. Common Anti-Patterns

| Anti-pattern | Cost | Fix |
|---|---|---|
| `any Protocol` in hot-path params | VWT+PWT dispatch + possible heap alloc | `<T: Protocol>` generic |
| Non-`final` class in hot code | vtable dispatch blocks inlining | `final`, `struct`, or WMO |
| Large struct (>24 bytes) as `any` | Heap alloc per existential | Generics or small struct |
| Pass `[T]` by value to mutating func | CoW copy from +1 retain | `inout` |
| `String` for bounded domain fields | Heap alloc + ARC per copy | `enum`; `UUID` for identifiers |
| `Character` iteration for parsing | Heap alloc per `Character` | `.utf8` or `.unicodeScalars` |
| Array without `reserveCapacity` | O(log n) reallocations | `reserveCapacity(n)` before loop |
| `Data` for byte manipulation | `NSData` bridge overhead | `[UInt8]` or `UnsafeRawBufferPointer` |
| `weak` where `unowned` suffices | Side table hop on every load | `unowned` (or `unowned(unsafe)` in tight loops) |
| Actor hop inside loop | O(n) context switches | Batch across boundary once |
| `@MainActor` on entire compute-heavy class | Pure computation serialized on main thread | `nonisolated` on compute methods |
| Eager `map/filter/map` chains on large collections | Multiple intermediate array allocs | `.lazy` or manual loop |
| `Bool`/small field before `Int`/`Double` in struct | Padding inflates stride | Largest-alignment-first ordering |
| `indirect enum` outside recursive types | Heap alloc per enum instance | Inline associated values when size is bounded |
| Missing `@inlinable` on hot public API | No inlining or specialization for callers | `@inlinable` for small hot functions |
| String `+` in loop | O(n²) allocations | `joined()` or `reserveCapacity` + `append(contentsOf:)` |
| Unstructured `Task { }` for fine-grained parallelism | Heap-allocated task per call | `withTaskGroup` |

---

## 15. Profiling Workflow

1. **Time Profiler** (Instruments) — find CPU hotspots; sort by *Self* time
2. **Allocations** (Instruments) — find unexpected heap allocations in hot paths; look for transient object churn
3. **SIL inspection** (`swiftc -emit-sil`) — search for `witness_method` (PWT dispatch) and `class_method` (vtable dispatch) in hot functions
4. **`MemoryLayout<T>.stride`** — verify struct strides at development time; print once, not in a hot path

Profile in **release mode only**. Many allocations and dispatch patterns visible in `-Onone` are completely eliminated by the optimizer in `-O`.
