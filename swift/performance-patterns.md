# SwiftUI Performance Patterns

> Keep body cheap. Use @Observable for granular re-renders. Prefer transforms over layout changes for animations. Use lazy containers for long lists.

## @Observable — Granular Re-Renders

@Observable only re-renders views that read the specific property that changed. Don't pass the whole object when a child only needs one value.

```swift
@Observable
@MainActor
class ProfileViewModel {
    var name: String = ""
    var bio: String = ""
    var followerCount: Int = 0
}

// ✅ Only re-renders when `name` changes — NOT when bio/followerCount change
struct NameLabel: View {
    let viewModel: ProfileViewModel

    var body: some View {
        Text(viewModel.name)
    }
}
```

### Narrow State Scope

Pass only the specific values each view needs, not entire models:

```swift
// ❌ Unnecessarily broad coupling — view depends on the whole model
struct ItemRow: View {
    @Environment(AppModel.self) private var model
    let item: Item

    var body: some View {
        Text(item.name)
            .foregroundStyle(model.theme.primaryColor)
    }
}

// ✅ Narrow dependency — only updates when themeColor changes
struct ItemRow: View {
    let item: Item
    let themeColor: Color

    var body: some View {
        Text(item.name)
            .foregroundStyle(themeColor)
    }
}
```

## Guard Before State Updates

SwiftUI doesn't always deduplicate state changes. Guard to avoid redundant re-renders:

```swift
// ❌ Triggers update even if value unchanged
.onReceive(publisher) { value in
    self.currentValue = value
}

// ✅ Only update when different
.onReceive(publisher) { value in
    if self.currentValue != value {
        self.currentValue = value
    }
}
```

## Hot Path Optimization

Hot paths (scroll handlers, preference changes) fire continuously. Gate state updates by threshold:

```swift
// ❌ Updates state on every scroll pixel
.onPreferenceChange(ScrollOffsetKey.self) { offset in
    shouldShowTitle = offset.y <= -32
}

// ✅ Only update when crossing threshold
.onPreferenceChange(ScrollOffsetKey.self) { offset in
    let shouldShow = offset.y <= -32
    if shouldShow != shouldShowTitle {
        shouldShowTitle = shouldShow
    }
}
```

## Prefer Transforms Over Layout for Animations

GPU-accelerated transforms (`scaleEffect`, `offset`, `rotationEffect`, `opacity`) are much cheaper than layout-triggering changes (`frame`, `padding`).

```swift
// ✅ Fast — GPU transforms, no layout recalculation
Rectangle()
    .frame(width: 100, height: 100)
    .scaleEffect(isActive ? 1.5 : 1.0)
    .offset(x: isActive ? 50 : 0)
    .rotationEffect(.degrees(isActive ? 45 : 0))
    .animation(.spring, value: isActive)

// ❌ Slow — triggers layout pass on every animation frame
Rectangle()
    .frame(width: isActive ? 150 : 100, height: isActive ? 150 : 100)
    .padding(isActive ? 50 : 0)
    .animation(.spring, value: isActive)
```

## Lazy Containers

Use `LazyVStack` / `LazyHStack` for large collections. Regular stacks create all views immediately.

```swift
// ❌ Creates all 1000 views at once
ScrollView {
    VStack {
        ForEach(items) { item in
            ExpensiveRow(item: item)
        }
    }
}

// ✅ Creates views on demand as they scroll into view
ScrollView {
    LazyVStack {
        ForEach(items) { item in
            ExpensiveRow(item: item)
        }
    }
}
```

## Equatable Views for Expensive Bodies

For views with expensive rendering (Canvas, charts), conform to `Equatable` and apply `.equatable()` to skip `body` when inputs haven't changed:

```swift
struct ExpensiveChartView: View, Equatable {
    let data: [DataPoint]
    let color: Color

    static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.data == rhs.data && lhs.color == rhs.color
    }

    var body: some View {
        Canvas { context, size in
            // expensive drawing
        }
    }
}

// Apply .equatable() to use your custom == for diffing instead of SwiftUI's default
ExpensiveChartView(data: data, color: .blue)
    .equatable()
```

## Debugging Re-Renders

Use `Self._printChanges()` in body to see why a view re-renders. This is a private API (mentioned in WWDC 2021) — it MUST be wrapped in `#if DEBUG` and never ship in production builds.

```swift
var body: some View {
    #if DEBUG
    let _ = Self._printChanges()  // prints: "DebugView: _count changed."
    #endif
    Text("Count: \(count)")
}
```
