---
name: swiftui-navigation
description: Use this skill whenever the user asks about SwiftUI navigation — `NavigationStack`, `NavigationSplitView`, `NavigationPath`, value-based navigation with `navigationDestination`, programmatic navigation, deep linking, sheets, popovers, full-screen covers, alerts, confirmation dialogs, the `dismiss` environment, navigation bar customization, or migrating from the legacy `NavigationView`. Targets iOS 17+. Triggers on "how do I push a view", "how do I pop programmatically", "how do I deep link", "should I use NavigationStack or NavigationSplitView", "how do I show a sheet", or any navigation correctness/architecture question.
---

# SwiftUI Navigation

Navigation in SwiftUI is **state-driven**. You declare a path or selection; the framework renders the corresponding stack. Manual push/pop calls do not exist.

Targets iOS 16+ (`NavigationStack`, `NavigationSplitView`). The legacy `NavigationView` is deprecated — do not write new code with it.

---

## Two Containers

| Container | Use for |
|---|---|
| `NavigationStack` | Drill-down stack. iPhone-style hierarchy. |
| `NavigationSplitView` | Sidebar / content / detail. iPad, macOS, Apple TV. Adapts to compact-width by collapsing into a stack. |

Pick `NavigationSplitView` when the app is split-view-shaped on big screens. It collapses correctly on iPhone — no need to branch on size class.

---

## `NavigationStack` Basics

### Static destinations

```swift
NavigationStack {
    List(items) { item in
        NavigationLink(item.title, value: item)
    }
    .navigationDestination(for: Item.self) { item in
        ItemDetailView(item: item)
    }
}
```

`NavigationLink(_:value:)` pushes the value onto the stack's path. `navigationDestination(for:)` declares how to render values of a given type. Multiple `navigationDestination` modifiers handle multiple value types — but each type must appear at most once per stack.

### Programmatic / path-driven

```swift
@State private var path = NavigationPath()

NavigationStack(path: $path) {
    Root()
        .navigationDestination(for: Item.self) { ItemDetailView(item: $0) }
        .navigationDestination(for: User.self) { UserView(user: $0) }
}
```

Operations on `NavigationPath`:

- `path.append(value)` — push.
- `path.removeLast(k)` — pop.
- `path = NavigationPath()` — pop to root.
- `path.append(contentsOf:)` — push a sequence.

### Typed paths

`NavigationPath` stores `Hashable & Codable` values heterogeneously. For homogeneous stacks, prefer `[Item]` directly:

```swift
@State private var path: [Item] = []
NavigationStack(path: $path) { ... }
```

Typed arrays are easier to introspect and serialize.

### Codable paths (state restoration / deep links)

```swift
let data = try JSONEncoder().encode(path.codable)
// later:
if let codable = try? JSONDecoder().decode(NavigationPath.CodableRepresentation.self, from: data) {
    path = NavigationPath(codable)
}
```

Every value pushed must be `Codable` for `path.codable` to be non-nil.

---

## `NavigationSplitView`

### Two-column

```swift
NavigationSplitView {
    SidebarView(selection: $selection)
} detail: {
    if let selection { DetailView(item: selection) } else { Text("Pick one") }
}
```

### Three-column

```swift
NavigationSplitView {
    Sidebar(selection: $folder)
} content: {
    ContentList(folder: folder, selection: $item)
} detail: {
    if let item { Detail(item: item) }
}
```

Selection is the source of truth; the framework renders accordingly.

### Column visibility

```swift
@State private var visibility: NavigationSplitViewVisibility = .all
NavigationSplitView(columnVisibility: $visibility) { ... }
```

Useful for hiding the sidebar on launch, or for manual show/hide.

### Detail stack

The detail column can itself contain a `NavigationStack`:

```swift
NavigationSplitView {
    Sidebar(...)
} detail: {
    NavigationStack(path: $detailPath) {
        DetailRoot().navigationDestination(for: ...) { ... }
    }
}
```

This is the standard pattern for "sidebar selects a section, section pushes detail."

---

## Deep Linking

Convert a URL into a path mutation. Centralize in a router:

```swift
@Observable final class Router {
    var path = NavigationPath()
    func handle(_ url: URL) {
        // parse and append values
    }
}

@main struct App: App {
    @State private var router = Router()
    var body: some Scene {
        WindowGroup {
            NavigationStack(path: $router.path) { Root().navigationDestination(...) { ... } }
                .environment(router)
                .onOpenURL { router.handle($0) }
        }
    }
}
```

Tips:

- Build the path incrementally with `.append(...)` calls; that's how you get a stack of N levels deep.
- Push synchronously in response to the URL; navigation animates correctly.
- For "open this in a fresh stack," reset the path before appending.

---

## Sheets, Popovers, Covers, Alerts

| API | Use |
|---|---|
| `.sheet(isPresented:)` / `.sheet(item:)` | Modal sheet (form, picker) |
| `.fullScreenCover(...)` | iOS only; sheet that fully covers |
| `.popover(isPresented:)` | iPadOS/macOS popover |
| `.alert(_:isPresented:)` | System alert with buttons |
| `.confirmationDialog(_:isPresented:)` | Action sheet replacement |
| `.inspector(isPresented:)` (iOS 17) | Inspector pane |

Two binding shapes:

```swift
.sheet(isPresented: $showing) { Editor() }
.sheet(item: $selectedItem) { item in Editor(item: item) }
```

Prefer the `item:` form when the sheet's content depends on a value — avoids the "show sheet with stale selection" race.

### Sheet sizing (iOS 16+)

```swift
.sheet(isPresented: $show) {
    Editor()
        .presentationDetents([.medium, .large])
        .presentationDragIndicator(.visible)
        .presentationContentInteraction(.scrolls)
}
```

### Dismissing

```swift
@Environment(\.dismiss) private var dismiss
Button("Close") { dismiss() }
```

`dismiss` works for sheets, covers, popovers, and pushed views. Inside an environment-injected view it's the closest "go back" the system can produce.

---

## Toolbars and Navigation Bar

```swift
.toolbar {
    ToolbarItem(placement: .topBarLeading) { Button("Cancel") { dismiss() } }
    ToolbarItem(placement: .topBarTrailing) { Button("Save") { save() } }
    ToolbarItem(placement: .principal) { Title() }
}
.navigationTitle("Edit")
.navigationBarTitleDisplayMode(.inline)
.toolbarBackground(.ultraThinMaterial, for: .navigationBar)
.toolbarBackground(.visible, for: .navigationBar)
```

Place `.navigationTitle` inside the destination view, not on the `NavigationStack`. The title belongs to the top of the stack.

---

## Common Patterns

### Router model

Hoist navigation state into an `@Observable` router so multiple parts of the app can mutate it:

```swift
@Observable final class AppRouter {
    var path = NavigationPath()
    var selectedTab: Tab = .home
    var presentedSheet: SheetID?
}
```

Inject as environment, drive deep links, tab switches, and "tap notification → navigate" flows from one place.

### Conditional destinations

`navigationDestination(for:)` registers a closure for a type. If you need branching, branch *inside* the closure based on the value's properties. Don't use `if`/`else` to register different destinations conditionally — registration must be stable.

### Tab + Stack

Each tab gets its own `NavigationStack` so each tab preserves its own stack:

```swift
TabView {
    NavigationStack(path: $homePath) { Home().navigationDestination(...) { ... } }
        .tabItem { Label("Home", systemImage: "house") }
    NavigationStack(path: $searchPath) { Search().navigationDestination(...) { ... } }
        .tabItem { Label("Search", systemImage: "magnifyingglass") }
}
```

### Pop to root

```swift
path.removeLast(path.count)   // typed array
path = NavigationPath()        // NavigationPath
```

---

## Migration from `NavigationView`

| Legacy | Replacement |
|---|---|
| `NavigationView { ... }` (stack style) | `NavigationStack { ... }` |
| `NavigationView { ... }` (split style) | `NavigationSplitView { ... }` |
| `NavigationLink(destination: View) { ... }` (iOS 14 form) | `NavigationLink(value:)` + `navigationDestination(for:)` |
| `isActive: $bool` push trigger | path-based push: `path.append(...)` |

Replace by tree, not file-by-file. Mixing old and new in the same hierarchy produces broken back behavior.

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| `NavigationLink(destination:)` form on iOS 16+ | Eagerly creates the destination view (allocates models) | Value form + `navigationDestination(for:)` |
| Multiple `navigationDestination(for: SameType.self)` in one stack | Undefined behavior; only one wins | Centralize per type |
| Branching `navigationDestination` registration with `if` | Registration churns; navigation breaks | Branch inside the closure |
| Mixing `NavigationView` and `NavigationStack` in one tree | Broken back stack | Migrate the whole tree |
| `@State` for path inside a child view that gets re-created | Stack resets on parent re-render | Hoist path to a stable owner (router, scene) |
| `dismiss()` called on a view that owns the sheet | No-op | `dismiss` from the *presented* view's env, not the presenter |
| Using `.fullScreenCover` for a normal modal | Heavier; iOS-only; harder to dismiss | Use `.sheet` with detents |
| `@State var sheet: Bool` AND `@State var selected: Item?` | Two sources of truth race | One `.sheet(item:)` |
| Animating the path with `withAnimation` | Navigation already animates; doubles up | Just mutate the path |
| Putting `navigationTitle` on `NavigationStack` | Ignored; the title belongs to the top of stack | Apply on the destination view |
| Storing non-`Hashable` values in `NavigationPath` | Won't compile | Use `Hashable & Codable` value types (IDs > full models) |
| Pushing entire model objects | Bloats path; restoration fails on schema change | Push IDs; resolve in the destination |
