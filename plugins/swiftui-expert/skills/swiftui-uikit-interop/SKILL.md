---
name: swiftui-uikit-interop
description: Use this skill whenever the user asks about bridging SwiftUI with UIKit or AppKit — `UIViewRepresentable`, `UIViewControllerRepresentable`, `NSViewRepresentable`, `NSViewControllerRepresentable`, the Coordinator pattern, `makeUIView`/`updateUIView`, hosting controllers (`UIHostingController`, `NSHostingController`, `UIHostingConfiguration`), embedding SwiftUI in UIKit table/collection views, communicating across the boundary, lifecycle issues, or migrating code between frameworks. Targets iOS 17+. Triggers on questions like "how do I use a UIKit view in SwiftUI", "how do I host SwiftUI in a `UICollectionView` cell", "Coordinator pattern", "why is my representable not updating", or "how do I reach back into UIKit".
---

# SwiftUI ↔ UIKit / AppKit Interop

How to wrap UIKit/AppKit views and controllers for use in SwiftUI, embed SwiftUI in UIKit/AppKit, and pass data both ways without leaks or update bugs.

---

## The Four Bridges

| Direction | Bridge type |
|---|---|
| Use a `UIView` / `NSView` in SwiftUI | `UIViewRepresentable` / `NSViewRepresentable` |
| Use a `UIViewController` / `NSViewController` in SwiftUI | `UIViewControllerRepresentable` / `NSViewControllerRepresentable` |
| Host SwiftUI in UIKit | `UIHostingController` (or `UIHostingConfiguration` for cells, iOS 16+) |
| Host SwiftUI in AppKit | `NSHostingController` / `NSHostingView` |

Use the controller variant whenever the UIKit/AppKit view manages its own lifecycle (text input, scroll views with delegates, anything that posts to a delegate). Use the view variant only for genuinely leaf views.

---

## `UIViewRepresentable`

```swift
struct ActivityIndicator: UIViewRepresentable {
    @Binding var isAnimating: Bool
    var style: UIActivityIndicatorView.Style = .medium

    func makeUIView(context: Context) -> UIActivityIndicatorView {
        let v = UIActivityIndicatorView(style: style)
        return v
    }

    func updateUIView(_ uiView: UIActivityIndicatorView, context: Context) {
        if isAnimating { uiView.startAnimating() } else { uiView.stopAnimating() }
    }
}
```

Lifecycle:

- `makeUIView(context:)` — called once per identity. Create the view, do one-time setup.
- `updateUIView(_:context:)` — called whenever the representable's stored properties change or the parent re-renders. **Idempotent.** Set state to match the current values; do not append or accumulate.
- `dismantleUIView(_:coordinator:)` — optional cleanup.
- `sizeThatFits(_:uiView:context:)` (iOS 16+) — participate in layout properly.

Critical rule: **`updateUIView` runs often.** Treat it like rendering — set everything from current state, do not assume "first call vs subsequent call." Don't reset `text` if it equals the existing value (that can drop user input mid-edit).

---

## The Coordinator

The `Coordinator` is your bridge for delegates, targets, gesture handlers, and Combine subscriptions. Without it, you cannot route delegate callbacks back to SwiftUI state.

```swift
struct SearchBar: UIViewRepresentable {
    @Binding var text: String

    func makeCoordinator() -> Coordinator { Coordinator(self) }

    final class Coordinator: NSObject, UISearchBarDelegate {
        var parent: SearchBar
        init(_ parent: SearchBar) { self.parent = parent }
        func searchBar(_ bar: UISearchBar, textDidChange t: String) {
            parent.text = t
        }
    }

    func makeUIView(context: Context) -> UISearchBar {
        let bar = UISearchBar()
        bar.delegate = context.coordinator
        return bar
    }

    func updateUIView(_ uiView: UISearchBar, context: Context) {
        context.coordinator.parent = self  // refresh captured parent
        if uiView.text != text { uiView.text = text }
    }
}
```

Two patterns to remember:

1. **Refresh the coordinator's `parent` reference in `updateUIView`.** The representable struct is recreated on every render; the coordinator persists. Without the refresh, delegate callbacks write to a stale binding.
2. **Guard updates that could fight user input** — `if uiView.text != text` avoids cursor-jumping when the user is typing.

---

## `UIViewControllerRepresentable`

Same shape, but `make`/`update` work with a controller:

```swift
struct PhotoPicker: UIViewControllerRepresentable {
    @Binding var selectedImage: UIImage?

    func makeUIViewController(context: Context) -> PHPickerViewController {
        var config = PHPickerConfiguration()
        config.filter = .images
        let picker = PHPickerViewController(configuration: config)
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ vc: PHPickerViewController, context: Context) { /* usually empty */ }

    func makeCoordinator() -> Coordinator { Coordinator(self) }
    final class Coordinator: NSObject, PHPickerViewControllerDelegate {
        let parent: PhotoPicker
        init(_ parent: PhotoPicker) { self.parent = parent }
        func picker(_ p: PHPickerViewController, didFinishPicking r: [PHPickerResult]) { ... }
    }
}
```

Use `.sheet { PhotoPicker(...) }` to present the wrapped controller modally.

---

## Sizing Representables

By default, a representable accepts the proposed size. To express an intrinsic content size, implement `sizeThatFits` (iOS 16+):

```swift
func sizeThatFits(_ proposal: ProposedViewSize, uiView: UIView, context: Context) -> CGSize? {
    let target = CGSize(width: proposal.width ?? UIView.layoutFittingExpandedSize.width,
                        height: proposal.height ?? UIView.layoutFittingCompressedSize.height)
    return uiView.systemLayoutSizeFitting(target,
        withHorizontalFittingPriority: .required,
        verticalFittingPriority: .fittingSizeLevel)
}
```

Returning `nil` falls back to default behavior. Implementing this avoids the common "embedded UIKit view collapses to zero" bug.

---

## Hosting SwiftUI in UIKit

### As a child view controller

```swift
let host = UIHostingController(rootView: ProfileView(user: user))
addChild(host)
view.addSubview(host.view)
host.view.translatesAutoresizingMaskIntoConstraints = false
NSLayoutConstraint.activate([ ... ])
host.didMove(toParent: self)
```

### As a cell content (iOS 16+) — `UIHostingConfiguration`

```swift
cell.contentConfiguration = UIHostingConfiguration {
    ProfileRow(user: user)
}
```

This is the *correct* way to put SwiftUI inside `UICollectionView` / `UITableView` cells in iOS 16+. It handles cell reuse, sizing, and theming. Don't roll your own `UIHostingController`-in-cell hack on iOS 16+.

### Sizing a hosting controller

`UIHostingController.sizeThatFits(in:)` reports the intrinsic SwiftUI size. Use this for "auto-size to content" embedding. You can also adjust `safeAreaRegions`, `disableSafeArea`, etc.

---

## Communicating Across the Boundary

| SwiftUI → UIKit | UIKit → SwiftUI |
|---|---|
| Stored properties on the representable struct | `Coordinator` writes to `@Binding` |
| `@Binding`/`@State` reflected in `updateUIView` | NotificationCenter / Combine into an `@Observable` model |
| Environment values via `context.environment` | Closures stored on the representable (`onSubmit: () -> Void`) |

The mental model: representable struct is a **snapshot** for the current frame. The coordinator is the **persistent identity** that owns delegate routing. Bindings flow both ways through the coordinator.

### Closures as callbacks

```swift
struct WebView: UIViewRepresentable {
    let url: URL
    var onLoaded: () -> Void = {}
    // coordinator forwards delegate calls to onLoaded via parent
}
```

Closure-based events are usually cleaner than wiring a binding for one-shot signals.

---

## Lifecycle and Identity

- A representable is recreated on every parent re-render. The `UIView`/`UIViewController` and `Coordinator` are not — they're keyed to view identity, like `@State`.
- `dismantleUIView(_:coordinator:)` and `dismantleUIViewController(_:coordinator:)` run when SwiftUI removes the representable from the tree. Cancel timers, observers, KVO here.
- Avoid recreating expensive UIKit objects in `makeUIView` if they could be reused; SwiftUI calls it only on identity change.

---

## Common Pitfalls

| Pitfall | Cause | Fix |
|---|---|---|
| Stale binding writes after user typed | Coordinator captures old `parent` from first `make` call | Refresh `coordinator.parent = self` in `update*` |
| Cursor jumps mid-typing | `updateUIView` resets `text` unconditionally | Guard with `if uiView.text != text` |
| Wrapped view has zero size | Default proposal handling | Implement `sizeThatFits(_:uiView:context:)` |
| Memory leak | Coordinator strongly captures self that captures coordinator | `weak` references in delegate methods or coordinator owns no strong cycle |
| SwiftUI updates not reflected | State written outside a view-tracked path | Use `@Binding` / `@Observable` + read in body |
| Cell reuse glitches | Manual `UIHostingController` in cell | Use `UIHostingConfiguration` (iOS 16+) |
| Animation discontinuity at boundary | UIKit animates with its own context | Drive UIKit animation from `updateUIView` and align timing manually |

---

## When *Not* to Bridge

- If a SwiftUI-native equivalent exists (`Map`, `WebView` in iOS 17.4+ via `WKWebView` wrapper, `Picker`), prefer it.
- For one-off effects, `Canvas` + native shapes often beats wrapping a Core Animation view.
- For full apps, mixing too many representable wrappers fragments the architecture; pick a primary framework per screen.

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Capturing `self` from the representable in `makeCoordinator` | Stale parent struct in callbacks | Pass and refresh `parent` in `update*` |
| Not implementing `sizeThatFits` (iOS 16+) | Wrong intrinsic size; collapses or overflows | Implement it |
| Writing to bindings during `makeUIView` | Mutates state during view construction | Defer to user-driven events |
| `UIHostingController` in `UICollectionViewCell` on iOS 16+ | Fights cell reuse; sizing bugs | `UIHostingConfiguration` |
| Storing UIKit references in SwiftUI `@State` | Lifecycle mismatch | Keep them in the coordinator |
| Forgetting `dismantle*` cleanup | Leaks observers, timers | Implement and unregister |
| Setting `delegate` on a fresh view in `updateUIView` | Sets it every render; harmless but smelly | Set it in `make*` |
| Wrapping a SwiftUI-native control "for performance" | Usually slower with the bridge overhead | Profile first; bridges aren't free |
| Force-unwrapping `context.coordinator` casts | Brittle | Use a typed `Coordinator` and rely on the system's typing |
