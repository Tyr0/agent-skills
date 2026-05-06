---
name: swiftui-state-management
description: Use this skill whenever the user asks about SwiftUI state, data flow, or property wrappers — including `@State`, `@Binding`, `@StateObject`, `@ObservedObject`, `@EnvironmentObject`, `@Environment`, `@Bindable`, the `@Observable` macro, the Observation framework, observation tracking, view re-rendering, when SwiftUI invalidates a view, sharing state between views, dependency injection in SwiftUI, or migrating from `ObservableObject` to `@Observable`. Targets iOS 17+ / macOS 14+ / Swift 5.9+. Triggers on questions like "why isn't my view updating", "should I use @State or @StateObject", "when do I use @Bindable", "how does Observation work", or any SwiftUI data-flow correctness question.
---

# SwiftUI State Management

The single most important SwiftUI skill: knowing where state lives, who owns it, and what triggers a view to re-render. Targets iOS 17+ where the Observation framework (`@Observable`) is the default.

---

## Mental Model

A SwiftUI view is a **value-type description** of UI, not the UI itself. SwiftUI re-evaluates `body` whenever a tracked dependency changes; the framework diffs the resulting tree against the previous tree and applies minimal updates to the underlying render tree.

There are three things to keep straight:

1. **Storage** — where does the value actually live across re-renders?
2. **Read/write surface** — how does this view see and mutate it?
3. **Invalidation** — does mutating it cause a re-render here?

Every property wrapper is a different combination of those three.

---

## The Property Wrappers (iOS 17+)

| Wrapper | Storage | Mutates view? | Use for |
|---|---|---|---|
| `@State` | View-owned, persists across re-renders | Yes | Local, view-private value-type state |
| `@Binding` | Reference to state owned elsewhere | Yes (in owner) | Two-way passage of a single value |
| `@Bindable` | Wraps an `@Observable` reference to derive bindings | Yes | Get `$model.field` bindings from any `@Observable` you don't own |
| `@Environment(\.x)` | Environment dictionary, ancestor-provided | Yes | Reading values + observable objects from environment |
| `@Environment(SomeType.self)` | Environment, by type (iOS 17) | Yes | Inject `@Observable` reference types via env |
| `@FocusState` | Focus system | Yes | Programmatic keyboard focus / focus-driven UI |
| `@SceneStorage` / `@AppStorage` | UserDefaults / scene state | Yes | Persistence-backed simple values |
| `@StateObject` *(legacy)* | View-owned reference to `ObservableObject` | Yes | Pre-Observation. Replaced by `@State` + `@Observable`. |
| `@ObservedObject` *(legacy)* | Externally-owned `ObservableObject` | Yes | Pre-Observation. Replaced by direct `@Observable` property. |
| `@EnvironmentObject` *(legacy)* | Environment, by `ObservableObject` type | Yes | Pre-Observation. Replaced by `@Environment(SomeType.self)`. |

---

## `@State` — Local Value State

`@State` gives a view storage that survives re-renders.

```swift
struct Counter: View {
    @State private var count = 0

    var body: some View {
        Button("Tapped \(count)") { count += 1 }
    }
}
```

Rules:

- **Always `private`.** `@State` is per-instance to the view's identity. External access is meaningless.
- The wrapped type should be a **value type** (struct, enum, primitive). For reference types use `@State` with an `@Observable` class — see below.
- `@State` storage is keyed by view *identity*. If SwiftUI gives the view a new identity (e.g. via `.id()` or a different position in a `ForEach`), the state is reset.
- Initializing `@State` in `init` works but the initializer only runs the first time the view appears at that identity. Subsequent inits of the struct discard the new initial value.

### `@State` with `@Observable` reference types (iOS 17+)

You no longer need `@StateObject`. Storing an `@Observable` class in `@State` makes the view its owner and gives correct lifecycle:

```swift
@Observable final class CartModel { var items: [Item] = [] }

struct CartView: View {
    @State private var cart = CartModel()
    var body: some View { Text("\(cart.items.count) items") }
}
```

---

## `@Binding` — Two-Way Pass-Through

A `Binding<T>` is a getter/setter pair pointing at storage somewhere else. A binding does **not** own state; it forwards reads and writes.

```swift
struct Toggleable: View {
    @Binding var isOn: Bool
    var body: some View { Toggle("On", isOn: $isOn) }
}

// Caller:
Toggleable(isOn: $someState)
```

Where bindings come from:

- `$state` for any `@State`, `@SceneStorage`, `@AppStorage`, `@FocusState`.
- `$model.field` for any `@Bindable` reference.
- `Binding(get:set:)` for derived/computed bindings.
- `Binding.constant(value)` for previews and literal cases.

### Derived bindings

```swift
let upper: Binding<String> = Binding(
    get: { name.uppercased() },
    set: { name = $0.lowercased() }
)
```

Beware: if the `set` mutates state that the producing view depends on, you'll re-render the parent on every keystroke. Push the binding's source as far down the tree as it can live.

---

## The Observation Framework (`@Observable`)

`@Observable` is a Swift macro that replaces `ObservableObject` + `@Published`. It enables **automatic, granular tracking**: a view that reads only `model.name` re-renders only when `name` changes, even if the same model has 50 other fields.

```swift
import Observation

@Observable
final class UserModel {
    var name: String = ""
    var email: String = ""
    @ObservationIgnored var cache: NSCache<NSString, AnyObject> = .init()
}
```

Key facts:

- Apply to a `class`. Value types use `@State` directly.
- Stored properties are tracked by default. Use `@ObservationIgnored` on properties that should never trigger updates (caches, debounce timers, etc).
- Computed properties are tracked through their dependencies — read `name` inside a computed, and views observe `name`.
- Tracking is **per-property, per-view**. The framework records which properties a view's `body` accesses and only invalidates that view when one of those properties is set.
- Tracking works through chains: `model.user.profile.name` invalidates only views that read that exact path.
- Tracking is also non-magical inside collections — reading `array.count` tracks count, reading `array[i].field` tracks the element.

### How to consume an `@Observable`

| Source of truth | View declaration |
|---|---|
| This view owns the model | `@State private var model = MyModel()` |
| Parent owns and passes it | `let model: MyModel` — plain property |
| Need bindings from a model you don't own | `@Bindable var model: MyModel` |
| Comes from environment by type | `@Environment(MyModel.self) private var model` |

The parent-passes-it case is what trips people up: **you do not need any property wrapper** to observe a passed-in `@Observable`. Just use a plain stored property. SwiftUI tracks accesses inside `body` automatically.

```swift
struct ProfileView: View {
    let user: UserModel  // No wrapper needed.
    var body: some View { Text(user.name) }
}
```

### `@Bindable`

Use when you need `$model.field` syntax for a model you didn't create with `@State`:

```swift
struct EditView: View {
    @Bindable var user: UserModel
    var body: some View {
        TextField("Name", text: $user.name)  // $ requires @Bindable
    }
}
```

You can also wrap inline: `@Bindable var user = user` inside `body` is occasionally useful.

---

## `@Environment` — Implicit Dependency Injection

Two forms:

**Keyed values** (system-provided or custom `EnvironmentKey`):

```swift
@Environment(\.colorScheme) private var colorScheme
@Environment(\.dismiss) private var dismiss
@Environment(\.locale) private var locale
```

**Observable injection by type** (iOS 17+):

```swift
@main struct App: App {
    @State private var session = SessionModel()
    var body: some Scene {
        WindowGroup {
            ContentView().environment(session)
        }
    }
}

struct DeepView: View {
    @Environment(SessionModel.self) private var session
    var body: some View { Text(session.user.name) }
}
```

Notes:

- `.environment(value)` (no key) replaces the legacy `.environmentObject(_:)`.
- If you forget to inject and read `@Environment(SomeType.self)`, the app crashes at runtime when the view appears. Treat env-injected models like required dependencies.
- If you need bindings from an environment-injected `@Observable`, redeclare with `@Bindable` locally:
  ```swift
  @Environment(SessionModel.self) private var session
  var body: some View {
      @Bindable var session = session
      TextField("Name", text: $session.user.name)
  }
  ```

---

## When Does a View Re-Render?

A view's `body` is recomputed when **any tracked dependency it read last time** changes:

- Any `@State`, `@Binding`, `@FocusState`, `@SceneStorage`, `@AppStorage`, `@Environment` value the view read.
- Any property of an `@Observable` model the view read (transitively through computed properties).
- The parent recomputes `body` and produces a new instance whose stored `let`/`var` values differ (SwiftUI compares stored properties via `Equatable` if available, else by re-evaluation).

A view does **not** re-render just because a parent recomputed — SwiftUI compares the new view value to the old and skips invalidation if equivalent. This is how the system stays cheap.

### Common re-render bugs

| Symptom | Cause | Fix |
|---|---|---|
| View doesn't update when model changes | Reading model property outside `body` (e.g., stored in `let` at init time) | Read inside `body` or any view-builder closure |
| Whole screen re-renders on every keystroke | Owner of `@State` is too high in the tree | Push the state down to the smallest view that needs it |
| List item re-renders all rows on one change | Binding sourced from parent collection by index | Use `ForEach($items)` with element bindings (iOS 15+) |
| `@Observable` updates not seen | Property is `@ObservationIgnored` or set on a different instance | Remove ignore, verify identity |
| State resets unexpectedly | View identity changed (`.id()`, position in `ForEach` without stable IDs) | Use stable `id:` in `ForEach`; avoid spurious `.id()` |

---

## Migrating from `ObservableObject` → `@Observable`

| Before | After |
|---|---|
| `class M: ObservableObject { @Published var x = 0 }` | `@Observable class M { var x = 0 }` |
| `@StateObject var m = M()` | `@State private var m = M()` |
| `@ObservedObject var m: M` | `var m: M` (plain property) |
| `@EnvironmentObject var m: M` | `@Environment(M.self) var m` |
| `.environmentObject(m)` | `.environment(m)` |
| `$m.x` (from `@ObservedObject`) | `@Bindable var m: M` then `$m.x` |

You can mix old and new in the same app while migrating, but a single model class cannot be both `ObservableObject` and `@Observable`.

---

## Choosing the Right Tool — Decision Table

| Need | Use |
|---|---|
| Local primitive/struct that this view mutates | `@State private var x = ...` |
| Local reference model this view owns | `@State private var model = MyModel()` (`@Observable`) |
| Receive a model from parent, read-only | plain `let model: MyModel` |
| Receive a model from parent, need bindings | `@Bindable var model: MyModel` |
| Pass a single value down for two-way edit | `@Binding var x: T` |
| Inject app-wide observable | `.environment(model)` + `@Environment(MyModel.self)` |
| Read system value (color scheme, dismiss, etc.) | `@Environment(\.keyPath)` |
| Persist across launches | `@AppStorage` |
| Persist across scene restoration | `@SceneStorage` |
| Programmatic keyboard focus | `@FocusState` |

---

## Identity and State Lifetime

State is keyed to view *identity*. SwiftUI determines identity from:

1. **Structural identity** — position in the view hierarchy.
2. **Explicit identity** — `.id(value)` or `id:` in `ForEach`.

Consequences:

- `if cond { A() } else { B() }` produces two distinct identities. A `@State` inside `A` is destroyed when `cond` becomes false, recreated when true.
- Adding `.id(x)` re-creates the view (and its state) every time `x` changes. Use deliberately for "reset on input change" effects.
- `ForEach(items)` requires identity. Prefer `ForEach(items, id: \.stableID)` or `Identifiable`. Using array index is correct only for static lists.

---

## Async Work and State

`.task` is the right entry point for async work tied to view lifetime:

```swift
.task(id: userID) {
    await load(for: userID)  // re-runs when userID changes; cancelled on disappear
}
```

- `.task` runs on `MainActor` by default; the parent view body is also `MainActor`-isolated.
- Don't write to `@State` from a non-main actor. The compiler enforces this in Swift 6.
- `.onAppear` is fine for synchronous setup but lacks structured cancellation — prefer `.task`.

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| `@ObservedObject var m = M()` | Recreates `M` on every parent re-render; state lost | `@State private var m = M()` (or `@StateObject` pre-iOS 17) |
| Reading model props in `init` and storing in `let` | View won't track them; no re-render on change | Read in `body` |
| Single huge `@Observable` for whole app | Excess invalidations; debugging nightmare | Split by concern; inject narrowly |
| Bindings derived in parent and passed deep | Parent re-renders on every change | Inject the model itself; let the leaf derive its binding |
| `@State` for data owned elsewhere | Source-of-truth duplication; drift | Use `@Binding` or pass the model |
| Forgetting `private` on `@State` | Misleading API; external sets are no-ops at the wrong level | Always `private` |
| `@ObservationIgnored` on a property the UI reads | View never updates | Remove the ignore |
| Mutating model from background task without `@MainActor` | Data race / Swift 6 error | Hop to main, or mark the model `@MainActor` |
| `.id(UUID())` to force refresh | Resets state on every render; pathological | Use a stable id, or `.task(id:)`, or invalidate state explicitly |
| `if let x = optional { View(x) }` swapping identity | State inside resets when optional flips | Hoist non-optional state to parent or use `.animation`-friendly patterns |
| `EnvironmentObject` crash at runtime | Forgot to inject ancestor | Inject at the scene/app root; use previews with the same injection |
| Binding into an array by index inside `ForEach` | Wrong row updates when array reorders | `ForEach($items) { $item in ... }` (binding-based ForEach) |
