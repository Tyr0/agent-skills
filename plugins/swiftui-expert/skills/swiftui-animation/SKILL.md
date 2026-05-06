---
name: swiftui-animation
description: Use this skill whenever the user asks about SwiftUI animation — `withAnimation`, `.animation` modifier, `Transaction`, implicit vs explicit animations, `matchedGeometryEffect`, phase animators (`PhaseAnimator`), keyframe animators (`KeyframeAnimator`), custom `Animatable` and `AnimatableModifier`, transitions, `AnyTransition`, spring animations, custom timing curves, animating shape/path, animating gradient, or any "why isn't my view animating" / "why is the wrong thing animating" / "how do I do shared element transitions" question. Targets iOS 17+. Triggers on questions about animation correctness, animation timing, or animation performance.
---

# SwiftUI Animation

Animation in SwiftUI is **state-driven**: you change a value in an animated transaction; the framework interpolates animatable properties on every affected view. Targets iOS 17+.

---

## Two Worlds: Implicit and Explicit

### Explicit — `withAnimation`

Wrap the *state mutation* that should be animated:

```swift
withAnimation(.easeInOut(duration: 0.3)) {
    isExpanded.toggle()
}
```

Anything that re-evaluates because of that mutation animates. This is the default tool — reach for it first.

### Implicit — `.animation(_:value:)`

Attach an animation to a value; the framework animates whenever that value changes.

```swift
Circle()
    .scaleEffect(scale)
    .animation(.spring, value: scale)
```

Always use the **value-bound** form `.animation(_:value:)`. The legacy `.animation(_:)` (no value) is deprecated and dangerous — it animates *every* state change in scope, often things you didn't intend.

### When to use which

- **`withAnimation`** when the change happens at one place (button tap, gesture end). You control the animation per call site.
- **`.animation(_:value:)`** when a property changes from many places and the rule "this view always animates *this* value with *this* curve" is what you want.

---

## Animatable Properties

A view animates *animatable* properties on its modifier chain:

- `.scaleEffect`, `.rotationEffect`, `.offset`, `.position`, `.frame`, `.padding`
- `.opacity`, `.foregroundStyle` (when between two color values)
- Shape paths via `Animatable`
- Gradient stops, etc.

Not animatable on the render thread:

- Text content (the string itself)
- Conditional view *presence* — that's a transition, not an animation
- Stored state values that aren't bound to an animatable view modifier

If you need to animate a non-animatable thing (e.g., a counter going from 0→100 with text changes), use `Text(value, format:)` and animate the source `Double` — SwiftUI animates the binding (iOS 17 has `contentTransition(.numericText())`).

---

## Animation Curves

| Curve | Use |
|---|---|
| `.linear(duration:)` | Linear interpolation; rare in UI |
| `.easeIn` / `.easeOut` / `.easeInOut(duration:)` | Generic UI |
| `.spring(duration:, bounce:)` (iOS 17) | Default-feeling iOS motion |
| `.smooth` / `.snappy` / `.bouncy` (iOS 17) | Pre-tuned springs |
| `.interpolatingSpring(stiffness:, damping:)` | Physics-based, momentum-preserving |
| `.timingCurve(c0x, c0y, c1x, c1y, duration:)` | Custom Bézier |

`.spring` is the most natural for UI gestures; `.smooth` for ambient changes; `.snappy` for confirmation-style feedback.

### Modifying an animation

```swift
.animation(.spring.delay(0.2).speed(1.5), value: x)
.animation(.linear.repeatForever(autoreverses: true), value: pulse)
```

---

## Transitions

A *transition* describes how a view appears/disappears when added/removed from the hierarchy:

```swift
if showing {
    Banner()
        .transition(.move(edge: .top).combined(with: .opacity))
}
```

For the transition to actually run, the change in `showing` must happen inside an animated context (`withAnimation`).

Built-ins: `.opacity`, `.scale`, `.move(edge:)`, `.slide`, `.identity`, `.asymmetric(insertion:removal:)`, `.combined(with:)`.

Custom: implement `Transition` (iOS 17+) or `AnyTransition.modifier(active:identity:)`.

---

## `matchedGeometryEffect` — Shared Element Transitions

Two views in different parts of the tree can be linked so that when one disappears and the other appears, they appear to morph:

```swift
@Namespace private var ns

if expanded {
    LargeCard().matchedGeometryEffect(id: "card", in: ns)
} else {
    SmallCard().matchedGeometryEffect(id: "card", in: ns)
}
```

Wrap the toggle in `withAnimation`. Use the same `id:` on both sides; only one should be present at a time.

iOS 18 introduces zoom/`zoom` transitions for nav-stack-level shared element moves.

---

## `PhaseAnimator` (iOS 17+)

Steps through a sequence of phases automatically:

```swift
PhaseAnimator([0, 1, 2]) { phase in
    Star()
        .scaleEffect(phase == 1 ? 1.5 : 1)
        .rotationEffect(.degrees(Double(phase) * 90))
} animation: { phase in
    .easeInOut(duration: 0.4)
}
```

Use for canned multi-step animations driven by the framework rather than explicit timers.

`.phaseAnimator(_:trigger:content:)` modifier form re-runs when a `trigger` value changes — perfect for "celebrate on success" effects.

---

## `KeyframeAnimator` (iOS 17+)

Multi-track keyframe animations:

```swift
KeyframeAnimator(initialValue: AnimationValues(), trigger: bump) { values in
    Logo()
        .scaleEffect(values.scale)
        .rotationEffect(values.rotation)
} keyframes: { _ in
    KeyframeTrack(\.scale) {
        CubicKeyframe(1.2, duration: 0.2)
        SpringKeyframe(1.0, duration: 0.5)
    }
    KeyframeTrack(\.rotation) {
        LinearKeyframe(.degrees(20), duration: 0.2)
        SpringKeyframe(.zero, duration: 0.5)
    }
}
```

Each track interpolates an independent property over time. Choose `LinearKeyframe`, `CubicKeyframe`, `SpringKeyframe`, `MoveKeyframe`.

---

## Custom `Animatable`

For animating a value SwiftUI doesn't already know how to interpolate, conform `Animatable`:

```swift
struct CircleProgress: Shape, Animatable {
    var progress: Double
    var animatableData: Double {
        get { progress }
        set { progress = newValue }
    }
    func path(in rect: CGRect) -> Path { ... uses progress ... }
}
```

For two-dimensional animation: `animatableData = AnimatablePair<Double, Double>`.

---

## Transactions

A `Transaction` is the context-bag carrying animation info through `body` evaluation. You can inspect or override it:

```swift
withTransaction(\.disablesAnimations, true) {
    state = newValue   // mutate without animating, even inside animation scope
}
```

`Transaction(animation: .spring)` constructs a transaction explicitly. Useful in custom controls that compose with caller animations.

---

## `contentTransition` (iOS 16+)

Animate textual content changes:

```swift
Text(count, format: .number)
    .contentTransition(.numericText())
    .animation(.snappy, value: count)
```

Variants: `.numericText()`, `.opacity`, `.identity`, `.symbolEffect` (for SF Symbols).

`Image(systemName:).symbolEffect(.bounce, value: bounceTrigger)` — system-symbol animations.

---

## Animation Performance

- Animating `transform`-like modifiers (`scaleEffect`, `rotationEffect`, `offset`, `opacity`) is render-thread cheap.
- Animating `frame`/layout reflows the layout pass each frame — cheaper than UIKit's equivalent but more expensive than transforms.
- Animating something that re-runs `body` on each frame (e.g., a `ForEach` whose count changes) is the most expensive — usually reflects a design problem.
- `drawingGroup()` flattens an animated subtree into a single Metal layer — useful for complex composites.
- Avoid `.animation(.default)` blanket modifiers; they animate state changes you didn't think about.

---

## Gestures and Animation

Spring physics with `interactiveSpring` give tactile feel to drag gestures:

```swift
@State private var drag: CGSize = .zero

DragGesture()
    .onChanged { drag = $0.translation }
    .onEnded { _ in
        withAnimation(.interactiveSpring) { drag = .zero }
    }
```

`.interactiveSpring(response:dampingFraction:blendDuration:)` is tuned for follow-finger interactions.

---

## Recipes

| Goal | Approach |
|---|---|
| Toggle visibility with fade | `if x { V() }` + `.transition(.opacity)` + `withAnimation { x.toggle() }` |
| Animate number counter | `Text(n, format: .number).contentTransition(.numericText()).animation(.snappy, value: n)` |
| Animate SF Symbol | `Image(systemName: ...).symbolEffect(.bounce, value: trigger)` |
| Shared element morph | `@Namespace` + `matchedGeometryEffect(id:in:)` on both endpoints + `withAnimation` |
| Pulse loop | `.scaleEffect(pulse).animation(.easeInOut.repeatForever(autoreverses: true), value: pulse)` |
| Confetti / one-shot | `PhaseAnimator` with trigger |
| Multi-track choreographed | `KeyframeAnimator` |
| Path morph | Conform shape to `Animatable` |

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| `.animation(.default)` (no value) at top of view | Animates every unrelated state change | Use `.animation(_:value:)` or `withAnimation` |
| `withAnimation` around the *consequence* of state | Nothing animates; you wrapped the wrong thing | Wrap the *mutation* |
| `matchedGeometryEffect` with both copies present | No transition; just two views | Ensure exactly one is in the tree per state |
| Animating text by recomputing whole subtree per frame | High body cost | `contentTransition(.numericText())` |
| Reaching for `Timer` to animate | Re-renders every tick; fights the framework | `PhaseAnimator` / `KeyframeAnimator` / `.repeatForever` |
| `.transition` without `withAnimation` | Transition never runs | Wrap the toggling state change |
| Missing `@Namespace` scope (declared in different views) | Matched geometry breaks | One namespace, shared via parent or environment |
| Animating frame when transform suffices | Triggers layout each frame | Prefer `scaleEffect` / `offset` |
| Long `repeatForever` on hidden views | Wastes CPU offscreen | Stop animation when offscreen (`.onDisappear`) |
| Using the legacy `AnimatableModifier` | Deprecated; clunkier than `Animatable` on the type itself | Conform shape/view to `Animatable` |
| Interactive drag without `interactiveSpring` | "Follow then snap" feels mushy | `.interactiveSpring` for in-gesture, `.spring` for release |
| Animating a `@State` change without a UI consequence | Nothing happens; "why doesn't it animate?" | Bind the state to an animatable modifier |
