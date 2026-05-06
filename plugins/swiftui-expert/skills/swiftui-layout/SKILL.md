---
name: swiftui-layout
description: Use this skill whenever the user asks about SwiftUI layout — how views are sized and positioned, the proposed-size / required-size negotiation, `frame` vs `fixedSize`, `Spacer`, `padding`, alignment guides, safe areas, `GeometryReader`, the custom `Layout` protocol, `containerRelativeFrame`, `onGeometryChange`, scroll-driven layout, lazy grids, or any "why is my view the wrong size" question. Targets iOS 17+. Triggers on "how does SwiftUI layout work", "why is my view full-width", "what does fixedSize do", "how do I align two views", "should I use GeometryReader", or "how do I write a custom Layout".
---

# SwiftUI Layout

The SwiftUI layout system is a two-pass negotiation. Once the model clicks, the rest is just modifiers applied in the right order.

---

## The Layout Algorithm

Every layout is a conversation between a **parent** and a **child**:

1. **Parent proposes a size** — "you may use up to W×H" (either dimension can be `nil` = "as much as you want", or a concrete value).
2. **Child returns its required size** — it picks. The parent does not get to override.
3. **Parent positions the child** within its own bounds.

The child is sovereign about its size. The parent is sovereign about position. **There is no "force this child to be 100 points wide"** — there's only "I propose 100, you decide."

This is the mental model. Memorize it.

### Examples

- `Text("Hi")` — proposed full available space, returns just enough for the text.
- `Color.blue` — proposed N, returns N (greedy; takes whatever's offered).
- `Image("foo")` — returns the image's intrinsic size unless `.resizable()`.
- `Image("foo").resizable()` — returns the proposed size.
- `Spacer()` — returns the proposed size in its stack's axis, 0 perpendicular.

### Why your view is full-width

`Color`, `Rectangle`, `Spacer`, and `.background(Color)` are greedy. Stack them with a `Text` and the stack proposes the full width to the greedy view. The fix is `.frame(maxWidth: ...)` or `.fixedSize()` on the appropriate child.

---

## `frame` Modifiers

`.frame` doesn't force a size on its child. It inserts a wrapper view between parent and child, and the wrapper does its own proposal.

```swift
Text("Hi").frame(width: 100, height: 50)
```

Wrapper view: proposes (100, 50) to `Text`. Text returns its required (small) size. Wrapper reports (100, 50) to its parent and centers the Text inside.

### Forms

| Form | Behavior |
|---|---|
| `.frame(width:height:)` | Proposes exact size; reports exact size |
| `.frame(maxWidth: .infinity)` | Proposes whatever the outer parent gave; reports same |
| `.frame(minWidth:, maxWidth:)` | Clamps |
| `.frame(maxWidth: .infinity, alignment: .leading)` | Greedy + child aligned within |

### `fixedSize`

Tells the view to ignore its proposal and use its intrinsic (ideal) size.

```swift
Text(longString)
    .fixedSize(horizontal: false, vertical: true)  // wrap is fine, but never truncate vertically
```

Use case: text that should wrap to its full content rather than being truncated by a constraining parent.

---

## Stacks

`HStack` / `VStack` / `ZStack` divide proposed space among children using a "least-flexible-first" allocation:

1. Sort children by flexibility (smallest range first).
2. Propose `total / remainingChildren` to each in order.
3. Remaining children share leftover space.

This is why a `Text` in an `HStack` with a `Spacer` lays out predictably: `Text` is rigid, `Spacer` is flexible.

`spacing:` controls gaps. `.layoutPriority(1)` raises a child's priority so it gets space first.

`ZStack` overlays children with alignment; the ZStack's size is the union of its children's sizes.

### `LazyVStack` / `LazyHStack` / `LazyVGrid` / `LazyHGrid`

Same model, but children are realized only when scrolled near. Use inside a `ScrollView`. Required for tens of thousands of rows.

---

## Alignment

Alignment is **per-axis** and uses **alignment guides**:

```swift
HStack(alignment: .firstTextBaseline) { ... }
VStack(alignment: .leading) { ... }
ZStack(alignment: .topTrailing) { ... }
```

Custom alignment guides let you align non-adjacent views or align across nested stacks:

```swift
extension HorizontalAlignment {
    enum FormLabel: AlignmentID {
        static func defaultValue(in d: ViewDimensions) -> CGFloat { d[.leading] }
    }
    static let formLabel = HorizontalAlignment(FormLabel.self)
}

VStack(alignment: .formLabel) {
    HStack { Text("Name").alignmentGuide(.formLabel) { $0[.trailing] }
             TextField("", text: $name) }
    HStack { Text("Email").alignmentGuide(.formLabel) { $0[.trailing] }
             TextField("", text: $email) }
}
```

Result: labels right-aligned to each other, fields aligned to a shared column.

---

## Safe Area & Insets

By default, content respects the device's safe area (notches, home indicator, keyboard). To extend visuals into the safe area:

```swift
SomeBackground().ignoresSafeArea()
SomeBackground().ignoresSafeArea(.keyboard, edges: .bottom)
```

Use `.safeAreaInset(edge:)` to stick a view to a safe area edge while pushing content out of its way:

```swift
ScrollView { Content() }
    .safeAreaInset(edge: .bottom) { BottomBar() }
```

`.safeAreaPadding` (iOS 17+) adds padding inside the safe area without altering geometry.

---

## `padding`

`padding` adds insets *outside* the modified view (from the wrapper's perspective: wrapper is bigger by N, child gets the original proposal minus 2N).

```swift
Text("Hi").padding()                   // system-default all sides
Text("Hi").padding(.horizontal, 16)
Text("Hi").padding(.bottom, 8)
```

Order matters: `.padding().background(.red)` paints around the padding; `.background(.red).padding()` paints only behind the text and adds outer padding.

---

## `GeometryReader` (and What to Use Instead)

`GeometryReader` is greedy in both axes and gives you the proposed size + safe-area insets. It distorts layout because it always takes whatever the parent offers.

```swift
GeometryReader { proxy in
    Text("\(proxy.size.width)")
}
```

Problems:

- Inside a `VStack`, fills the whole stack vertically.
- Reading geometry to write `@State` fires update warnings.
- Re-runs on every layout pass; expensive in lists.

**Modern alternatives** (iOS 17+):

| Need | Use |
|---|---|
| Size relative to scroll/window container | `.containerRelativeFrame(.horizontal) { length, _ in length * 0.8 }` |
| Read geometry without distorting layout (iOS 18) | `.onGeometryChange(for: CGSize.self) { $0.size } action: { newSize in ... }` |
| Coordinate space transforms | `GeometryReader` is still fine, but isolate it |
| Build a measurement-driven layout | Implement `Layout` protocol |

---

## Custom `Layout` Protocol (iOS 16+)

When you need fully custom positioning (a flow layout, radial layout, custom split):

```swift
struct FlowLayout: Layout {
    var spacing: CGFloat = 8

    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGSize {
        // Return required size for the subviews given the proposal.
    }

    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) {
        for s in subviews { s.place(at: ..., proposal: ...) }
    }
}

FlowLayout(spacing: 8) { ForEach(tags) { TagView($0) } }
```

`Layout` is faster than `GeometryReader`-based emulations because it participates in the real layout pass and doesn't force re-renders.

---

## Grids

Two grid systems:

- **`Grid`** (iOS 16+) — eager grid with row/column alignment guarantees. Like CSS grid.
  ```swift
  Grid(alignment: .leading, horizontalSpacing: 12, verticalSpacing: 8) {
      GridRow { Text("Name");  TextField("", text: $name) }
      GridRow { Text("Email"); TextField("", text: $email) }
  }
  ```
- **`LazyVGrid` / `LazyHGrid`** — lazy, configured with `[GridItem]`.
  ```swift
  LazyVGrid(columns: [GridItem(.adaptive(minimum: 100))]) { ForEach(items) { ... } }
  ```

`GridItem.Size` cases: `.fixed`, `.flexible(minimum:maximum:)`, `.adaptive(minimum:maximum:)`.

---

## ScrollView

`ScrollView` proposes its content axis as `nil` (unbounded) and the cross axis as the proposed value.

iOS 17+ scroll APIs:

- `.scrollPosition(id:)` — bind a leading-cell ID.
- `.scrollTargetBehavior(.viewAligned)` — snap-to-page-like behavior.
- `.scrollTargetLayout()` — mark the layout that contains scroll targets.
- `.scrollTransition { content, phase in ... }` — phase-driven transitions per cell.
- `.containerRelativeFrame` — size cells by the visible container.
- `.scrollIndicators(.hidden)` — hide indicators.
- `.scrollBounceBehavior(.basedOnSize)` — only bounce when content overflows.

---

## Common Layout Recipes

| Goal | Approach |
|---|---|
| Center a view | `Color.clear.overlay(view)` or stack with `Spacer()`s |
| Fill available width | `.frame(maxWidth: .infinity)` |
| Fill width but cap | `.frame(maxWidth: 600)` |
| Two columns 50/50 | `HStack { A; B }` with each `.frame(maxWidth: .infinity)` |
| Pin to bottom | `.safeAreaInset(edge: .bottom) { BottomBar() }` |
| Sticky header in ScrollView | `LazyVStack(pinnedViews: [.sectionHeaders])` + `Section` |
| Aspect-fit image | `Image.resizable().aspectRatio(contentMode: .fit)` |
| Square cell | `.aspectRatio(1, contentMode: .fit)` |
| Wrap text and stop truncation | `.fixedSize(horizontal: false, vertical: true)` |
| Match width of sibling | Custom `PreferenceKey` or `Grid` |

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| `GeometryReader` wrapping a whole screen | Greedy in both axes; distorts; reads on every frame | `containerRelativeFrame`, `onGeometryChange`, or `Layout` |
| `.frame(width:)` to force child size | The child still chooses; you only constrain the proposal | Use `fixedSize` for intrinsic, or accept child sovereignty |
| `Spacer()` to pad inside an HStack instead of `.padding` | Spacers grow unpredictably with siblings | `.padding(.horizontal, ...)` |
| `.padding` then `.background` (or vice-versa, mixed up) | Wrong color extent | Order modifiers deliberately and remember they are nested wrappers |
| Animating layout that depends on `GeometryReader` size | Causes layout-during-update warnings | Move the geometry reading outside the animation, or use `Layout` |
| `LazyVStack` for ten items | Lazy machinery costs more than the items | Use plain `VStack` |
| `Grid` for unbounded scroll content | Eager; lays out everything | `LazyVGrid` |
| `.fixedSize()` blanket-applied | Disables wrapping; clips off-screen | Use the per-axis form |
| Multi-line text truncated unexpectedly | Parent proposed too little vertical space | `.fixedSize(horizontal: false, vertical: true)` |
| `frame(width: UIScreen.main.bounds.width)` | Ignores multitasking, iPad split view, macOS windows | Use proposal/`containerRelativeFrame` |
| Reading geometry to drive `@State` writes inside `body` | Layout/update cycle warning | Use `onGeometryChange` (iOS 18) or `PreferenceKey` |
