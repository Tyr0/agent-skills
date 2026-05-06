---
name: swiftui-performance
description: Use this skill whenever the user asks about SwiftUI performance, slow scrolling, dropped frames, view re-rendering cost, body invalidation, view identity, lazy containers, large lists, image performance, drawing performance, `EquatableView`, `@ViewBuilder` cost, `.task` lifecycle, `MainActor` interaction with SwiftUI, or any "why is my SwiftUI view slow" question. Targets iOS 17+. Triggers on questions like "why does my list lag", "how do I avoid recomputing body", "should I use `LazyVStack`", "is `GeometryReader` expensive", "how do I profile SwiftUI", or "view is rendering too often".
---

# SwiftUI Performance

How SwiftUI evaluates and renders views, where the costs hide, and how to keep frame time under 16.6 ms.

---

## The Cost Model

A SwiftUI frame breaks into:

1. **Body evaluation** — re-running `var body` on dirty views. Cheap individually (microseconds) but multiplicative.
2. **Diffing** — comparing the new view tree to the previous one to decide what changed.
3. **Layout** — proposed-size / required-size negotiation across the changed subtree.
4. **Render** — handing to Core Animation / Metal.

Most performance problems live in **(1)** and **(3)**. (2) is rarely the bottleneck on its own. (4) is the floor — once you bottom out there, you've optimized correctly.

**Profile in Release with Instruments → SwiftUI template** (Xcode 15+). Debug builds are wildly unrepresentative — the optimizer eliminates a lot of allocation and dispatch.

---

## Optimization Priority

Work top to bottom; stop when Instruments says you're done.

1. **Reduce the set of views invalidated per change** — push `@State` down, narrow `@Observable` reads, split big views.
2. **Make invalidation cheaper** — `Equatable` conformance on data, `EquatableView`, stable identity.
3. **Use lazy containers correctly** — `LazyVStack`/`LazyHGrid`/`List` for unbounded content.
4. **Avoid `GeometryReader`** in hot paths; prefer `containerRelativeFrame`, `onGeometryChange` (iOS 18), or `Layout`.
5. **Drop into `Canvas` or `drawingGroup()`** for heavy drawing.
6. **Move work off `body`** — pre-compute, cache in the model, use `.task(id:)`.

---

## Body Invalidation Rules

A view's `body` is recomputed when **any tracked dependency it accessed last time** changes. The Observation framework tracks per-property; legacy `ObservableObject` invalidates every observer on any `@Published` change.

Practical implications:

- A view that reads `model.title` re-renders only on title changes — not on `model.body` changes. (`@Observable` only.)
- A view that reads nothing observable re-renders only when its parent passes new stored values.
- A view's stored properties are compared field-by-field. If the field is `Equatable`, SwiftUI skips re-render when equal. If not, it conservatively re-renders.

### Make data `Equatable`

```swift
struct Row: View, Equatable {
    let item: Item  // Item: Equatable
    var body: some View { ... }
}
```

Or wrap a problematic subview:

```swift
EquatableView(content: Row(item: item))
```

`EquatableView` short-circuits diffing when `==` returns true. Use sparingly — `==` runs every diff.

### Inspect re-renders

```swift
let _ = Self._printChanges()  // logs which dependency invalidated this body
```

Place at the top of `body`. Available since iOS 15. Invaluable for diagnosing churn.

---

## View Identity and Stability

SwiftUI keys state and animation by view identity. Unstable identity causes:

- Lost `@State`
- Animations that don't animate
- Lazy containers re-creating cells unnecessarily

Rules:

- `ForEach(items, id: \.stableID)` — use a real stable id. Don't use `\.self` for value types that can collide. Don't use array index for mutable arrays.
- Prefer `Identifiable` conformance: `ForEach(items)`.
- Avoid `.id(UUID())` — it forces full re-creation every render.
- `if`/`else` branches have distinct identity; switching branches resets state in either branch.

---

## Lazy Containers

| Container | When to use |
|---|---|
| `VStack`/`HStack` | Small, bounded content. Renders all children eagerly. |
| `LazyVStack`/`LazyHStack` | Long scrollable content where you want every-row layout |
| `LazyVGrid`/`LazyHGrid` | Grids of unknown length |
| `List` | Plain or styled lists. Internally lazy + reuse. Prefer for table-like UI. |
| `ScrollView` + `LazyVStack` | When `List` styling fights you |

`List` reuses cells and integrates with system gestures (swipe actions, separators, selection). Reach for it first.

`LazyVStack` does not reuse cells the way UIKit `UITableView` does — it instantiates on demand and keeps them while in view + a small over-scan. For tens of thousands of rows that's still fine; for millions you need a windowing strategy.

### Pinned headers (sections)

```swift
LazyVStack(pinnedViews: [.sectionHeaders]) {
    ForEach(sections) { section in
        Section(header: SectionHeader(section)) { ... }
    }
}
```

---

## `GeometryReader` Pitfalls

`GeometryReader` consumes all available space in its parent and ignores its child's preferences for that frame. Common bugs:

- A `GeometryReader` inside a `VStack` expands to fill, breaking layout.
- Reading geometry to drive a `@State` write causes layout-during-update warnings.

Prefer:

- **iOS 17+**: `containerRelativeFrame` for sizing relative to a scroll/window container.
- **iOS 18+**: `.onGeometryChange(for:of:action:)` reads geometry without distorting layout.
- **iOS 16+**: Custom `Layout` protocol for measurement-driven layouts.

---

## Image Performance

`Image("foo")` and `Image(uiImage:)` are cheap descriptors, but **decoding** can be expensive at scroll time. Best practices:

- Pre-decode large images on a background queue and cache `CGImage`/`UIImage`.
- `AsyncImage` is convenient but has no caching; for production lists prefer Nuke / Kingfisher / your own.
- Use `.resizable().interpolation(.medium)` only when needed; `.none` is fastest for pixel art / icons.
- Match the image's pixel size to its display size — a 4K image rendered at 100×100 burns memory and bandwidth.
- For SVG/PDF assets in asset catalogs, prefer the "Preserve Vector Data" + "Single Scale" combination so the asset rasterizes once at the right size.

---

## `drawingGroup()` and `Canvas`

`drawingGroup()` flattens a subtree into a single Metal layer. Useful when a complex composition (many shapes, blurs, blends) is animated as a unit. Cost: an offscreen render pass, so don't slap it on small static views.

`Canvas` (iOS 15+) is a direct draw API — use it for charts, particle effects, custom drawings of N elements. Cheaper than N `Shape` views once N grows past a few hundred.

```swift
Canvas { ctx, size in
    for p in points { ctx.fill(Path(...), with: .color(.blue)) }
}
```

---

## Animation Cost

Animating layout (frame, padding) is cheaper than animating things that force a re-render every frame (e.g., a `@State` Double driving text content). To animate visual transforms cheaply, prefer:

- `.scaleEffect`, `.rotationEffect`, `.offset`, `.opacity` — render-thread transforms.
- `.matchedGeometryEffect` for shared element transitions.
- `withAnimation` around state mutation, not `.animation(_:)` which is implicit and easy to over-apply.

`.animation(_:value:)` (the new value-bound form) avoids the broad implicit form and is preferred.

---

## `.task` and Concurrency

```swift
.task(id: itemID) { await load(itemID) }
```

- Runs on `MainActor` by default since the enclosing view is `@MainActor`.
- `id:` re-runs when the id changes; cancels the previous task automatically.
- `.task` is cancelled on view disappearance (structured concurrency). `.onAppear` is not.

For background work, hop to a non-main actor explicitly: `await Task.detached { ... }.value` or call into an actor-isolated method on your model.

In Swift 6 / strict concurrency:

- `View` is `@MainActor` by default.
- Don't pass non-`Sendable` references across isolation boundaries.
- Models touched by SwiftUI views should usually be `@MainActor` (read-write on main) or expose only `Sendable` snapshots.

---

## Profiling Workflow

1. Build & profile **Release** scheme on device.
2. Instruments → **SwiftUI** template.
3. Look at:
   - **View Body** track — which views ran `body`, how often, how long.
   - **View Properties** — what changed and triggered the update.
   - **Core Animation Commits** — how often the render server commits.
4. Cross-reference with **Time Profiler** for actual CPU.
5. For `.task`-driven work, also enable the **Swift Concurrency** instrument.

In code: `Self._printChanges()` for quick triage. `signpost` (`os_signpost`) inside expensive computed properties to attribute time.

---

## Common Speed Wins

| Win | When |
|---|---|
| Push `@State` down to leaf views | Top-level state forces tree-wide re-render |
| Split big `body` into subviews | Lets SwiftUI skip un-changed subviews via Equatable check |
| Make data models `Equatable` | Skips re-render when stored values match |
| `LazyVStack` / `List` for long content | Eager stacks instantiate everything |
| Replace `GeometryReader` with `containerRelativeFrame` | Avoids parent-distortion + layout-pass cost |
| Cache pre-decoded images | Removes per-row decode |
| `.drawingGroup()` on complex animated composition | Flattens to one Metal layer |
| `Canvas` over many `Shape` siblings | Direct draw beats N views |
| Equatable-conform leaf views | `EquatableView` short-circuits diffing |
| Avoid `AnyView` in hot paths | Erases types and disables structural diffing optimizations |

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| `AnyView` everywhere | Disables type-driven diffing; forces re-creation | Return `some View`; use `Group`, `@ViewBuilder`, or `if`/`else` |
| `GeometryReader` wrapping every screen | Distorts layout, reads on every frame | `containerRelativeFrame`, `onGeometryChange`, or `Layout` |
| Top-level `@State` for deep input field | Whole screen re-renders per keystroke | Push `@State` into the field's view |
| `ObservableObject` for a 50-field model | Any change invalidates every observer | Migrate to `@Observable` (per-property tracking) |
| `.id(UUID())` to force refresh | Pathological re-creation | Use stable IDs; reset state explicitly |
| `AsyncImage` in fast scroll list | No cache; re-fetches on scroll | Use a real image pipeline with cache |
| Heavy work inside `body` (sort, filter, format) | Re-runs every invalidation | Pre-compute in model; cache; `.task(id:)` |
| `.animation(.default)` blanket modifier | Animates unrelated changes | `.animation(_:value:)` scoped to one value |
| `ForEach(array, id: \.self)` on mutable structs | Identity churns on edit | Use a real id (`UUID`, db pk, etc.) |
| Mutating `@State` inside `body` | Layout-during-update warning, infinite invalidation | Move into action closures, `.task`, or `.onChange` |
| Treating `Self._printChanges()` as production logging | It's debug-only | Use only while profiling |
| Reaching for `@StateObject` on iOS 17+ | Legacy; no per-property tracking | `@State` + `@Observable` |
