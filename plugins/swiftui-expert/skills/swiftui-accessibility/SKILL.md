---
name: swiftui-accessibility
description: Use this skill whenever the user asks about SwiftUI accessibility — VoiceOver, accessibility modifiers, accessibility labels/values/hints/traits, custom actions, accessibility rotors, Dynamic Type, Reduce Motion, increase contrast, differentiate without color, accessibility identifiers for UI testing, custom accessibility elements (`accessibilityElement(children:)`), or auditing a SwiftUI view for accessibility. Targets iOS 17+. Triggers on questions like "is this accessible", "how do I label this for VoiceOver", "Dynamic Type support", "how do I group children for accessibility", or any a11y review.
---

# SwiftUI Accessibility

SwiftUI gives you a lot of accessibility for free — and a lot of ways to silently break it. This is the audit checklist plus the modifier vocabulary.

---

## What You Get for Free

- Standard controls (`Button`, `Toggle`, `Slider`, `Picker`, `TextField`, `Link`) come with correct labels, traits, and gestures.
- `Image(systemName:)` and labeled `Image("name", label: Text(...))` produce VoiceOver descriptions.
- `Text` is read verbatim; concatenated text is read in order.
- Standard navigation (`NavigationStack`, sheets, alerts) has correct focus management and hierarchy announcements.

What you have to do yourself:

- Label *non-decorative* images you create with `Image("…")` or `Image(uiImage:)`.
- Label custom controls (anything you build with `onTapGesture` or shapes).
- Group related visuals so VoiceOver speaks them as a unit.
- Verify Dynamic Type, color, and motion behaviors.
- Provide `.accessibilityIdentifier` for UI tests when you need to find an element by id.

---

## The Core Modifiers

| Modifier | Sets |
|---|---|
| `.accessibilityLabel(_:)` | The "what is this" string VoiceOver reads |
| `.accessibilityValue(_:)` | The current value (e.g., "75%") |
| `.accessibilityHint(_:)` | Optional "double tap to ..." instruction |
| `.accessibilityAddTraits(_:)` / `.accessibilityRemoveTraits(_:)` | `.isButton`, `.isHeader`, `.isImage`, `.isSelected`, `.updatesFrequently`, `.isModal`, `.isSummaryElement` |
| `.accessibilityHidden(_:)` | Hide a decorative element from VoiceOver |
| `.accessibilityElement(children: .ignore | .combine | .contain)` | Group/treat as one element |
| `.accessibilityAction(named:_:)` | Add a custom rotor action |
| `.accessibilityRotor(_:entries:)` | Make a custom rotor for jumping between items |
| `.accessibilitySortPriority(_:)` | Force read order within a container |
| `.accessibilityIdentifier(_:)` | UI-test identifier (not user-visible) |
| `.accessibilityRespondsToUserInteraction(_:)` | Make a non-interactive element interactive for AT |
| `.accessibilityFocused($flag)` | Programmatically move VoiceOver focus |
| `.accessibilityChildren { ... }` | Provide synthetic children for a custom element |

---

## Custom Controls

Anything that's tappable but not a `Button` needs explicit accessibility:

```swift
HStack { Image(systemName: "heart"); Text("Like") }
    .onTapGesture { liked.toggle() }
    .accessibilityElement(children: .combine)
    .accessibilityAddTraits(.isButton)
    .accessibilityLabel(liked ? "Liked" : "Like")
    .accessibilityHint("Double tap to toggle")
```

Better: use a real `Button` with a custom label:

```swift
Button { liked.toggle() } label: {
    HStack { Image(systemName: "heart"); Text("Like") }
}
```

Real `Button` gets `.isButton`, hit testing, and focus for free.

---

## Grouping with `accessibilityElement`

By default, every leaf view in a `VStack`/`HStack` is its own VoiceOver element. For things that read better as a unit (a stat block, a row of related labels), combine them:

```swift
VStack {
    Text("Distance")
    Text("3.2 mi")
}
.accessibilityElement(children: .combine)
.accessibilityLabel("Distance, 3.2 miles")
```

Modes:

- `.ignore` — treat children as decoration; the parent gets the label.
- `.combine` — children's text is concatenated for the parent's label.
- `.contain` — parent is a container; children remain individually navigable.

---

## Images

```swift
Image("hero")  // decorative? supply nothing OR hide:
    .accessibilityHidden(true)

Image("hero", label: Text("Sunset over the bay"))   // labeled

Image(decorative: "divider")                         // explicit decorative
```

For `Image(systemName:)`, SF Symbols have built-in localized names — usually fine, but override when context demands ("envelope" → "Compose message").

---

## Dynamic Type

`Text`, `Label`, `TextField`, and most controls scale automatically when the user adjusts text size. Things to watch for:

- Hard-coded `frame` heights that clip text at large sizes.
- Custom views drawing fixed-point text — use scaled metrics:
  ```swift
  @ScaledMetric var iconSize: CGFloat = 24
  Image(systemName: "star").font(.system(size: iconSize))
  ```
- Tests at the largest accessibility size: `.environment(\.dynamicTypeSize, .accessibility5)` in previews.
- `.lineLimit(1)` + `Text` can cut off the user's setting; consider wrapping.

---

## Color, Contrast, and "Differentiate Without Color"

```swift
@Environment(\.accessibilityDifferentiateWithoutColor) var diffWithoutColor
@Environment(\.colorSchemeContrast) var contrast
@Environment(\.colorScheme) var scheme
```

Patterns:

- Use shape, position, or text in addition to color for status (✓ vs ✗ vs •).
- Test in **Increase Contrast** mode; ensure text remains legible.
- Don't rely on subtle hue differences for state — Color Filters and color blindness flatten them.
- Use semantic colors (`.primary`, `.secondary`, `.accentColor`) so the system can adjust appearance.

---

## Reduce Motion

```swift
@Environment(\.accessibilityReduceMotion) var reduceMotion

withAnimation(reduceMotion ? .none : .spring) { ... }
```

Adapt:

- Replace bouncy springs with crossfades.
- Skip parallax/zoom transitions.
- Avoid `repeatForever` ambient motion.

`Image(systemName:).symbolEffect(...)` respects reduce-motion automatically.

---

## VoiceOver Focus

Move focus programmatically when the UI changes context:

```swift
@AccessibilityFocusState var focusedField: Field?

TextField("Email", text: $email).accessibilityFocused($focusedField, equals: .email)
Button("Continue") {
    if email.isEmpty { focusedField = .email }
}
```

Also use `.accessibilityFocused` after presenting a sheet to direct VoiceOver to the new content's heading.

---

## Custom Actions and Rotors

For complex elements, add named actions instead of asking the user to find a button:

```swift
MessageRow(message: m)
    .accessibilityAction(named: "Reply") { reply(m) }
    .accessibilityAction(named: "Mark as unread") { markUnread(m) }
    .accessibilityAction(.delete) { delete(m) }
```

Rotors let users jump through a category of items:

```swift
.accessibilityRotor("Headings") {
    ForEach(headings) { h in
        AccessibilityRotorEntry(h.title, id: h.id)
    }
}
```

---

## Identifiers for UI Testing

`.accessibilityIdentifier("submit-button")` is **not** read by VoiceOver — it's purely for `XCUIElement` lookup. Add identifiers to elements your UI tests need to interact with; don't repurpose `accessibilityLabel` for this.

---

## Audit Checklist

Run this before declaring a view shippable:

- [ ] Every interactive element has a meaningful label or is a real `Button`/`Toggle`.
- [ ] Decorative images use `.accessibilityHidden(true)` or `Image(decorative:)`.
- [ ] Related label/value pairs are combined via `accessibilityElement(children: .combine)`.
- [ ] Headings use `.accessibilityAddTraits(.isHeader)`.
- [ ] Selected state is communicated via `.isSelected` trait, not just color.
- [ ] Dynamic Type tested at `.accessibility3` or higher; nothing clips.
- [ ] Reduce Motion tested; no required-motion content.
- [ ] Differentiate Without Color tested; status conveyed by more than color.
- [ ] VoiceOver swipe order is logical (use `accessibilitySortPriority` if not).
- [ ] After modal presentation, VoiceOver focus lands on the right element.
- [ ] All UI-test entry points have `accessibilityIdentifier`.
- [ ] Run **Accessibility Inspector** (Xcode → Open Developer Tool) and audit.

---

## Previewing Accessibility

```swift
#Preview("A11y - Big Text") {
    ContentView()
        .environment(\.dynamicTypeSize, .accessibility3)
}

#Preview("A11y - VoiceOver-style") {
    ContentView()
        .environment(\.accessibilityEnabled, true)
        .environment(\.accessibilityReduceMotion, true)
}
```

For audits, run on device with VoiceOver on (Settings → Accessibility → VoiceOver) — simulator support exists but device behavior is canonical.

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Custom tappable view with no label | VoiceOver reads "button" with no context | `.accessibilityLabel(...)` or use a real `Button` |
| `Image("icon")` with no label | "Image, icon" — useless | Provide label or hide if decorative |
| Status conveyed only by color | Invisible to color-blind users; flat under filters | Add shape/text/symbol |
| `.frame(height: 20)` around `Text` | Truncates at large Dynamic Type | Let text size; use `@ScaledMetric` for icons |
| Three labels in a row left as separate elements | VoiceOver reads each separately, awkward | `.accessibilityElement(children: .combine)` |
| Always-on `repeatForever` animation | Burns battery; ignores Reduce Motion | Gate on `accessibilityReduceMotion` |
| Hijacking `accessibilityLabel` for UI test ids | Pollutes spoken UI | Use `accessibilityIdentifier` |
| Heading styled with `.font(.title)` only | Not a heading semantically | Add `.accessibilityAddTraits(.isHeader)` |
| Sheet presented, focus stays on the presenter | User confused after sheet appears | `@AccessibilityFocusState` to set focus on sheet content |
| Hidden via `.opacity(0)` | Still in accessibility tree | `.accessibilityHidden(true)` or remove from tree |
| `.accessibilityHidden(true)` on a container | Hides all descendants from VoiceOver entirely | Apply selectively, or use `.ignore` mode in `accessibilityElement` |
| Custom controls without `.isButton` trait | VoiceOver doesn't announce as button | Add the trait or use a real `Button` |
