# Modern SwiftUI API Guide

> Target: iOS 18+. Always use the modern replacement. Deprecated APIs generate warnings and may be removed.

## Deprecated → Modern Replacement Table

| Deprecated | Modern Replacement | Since |
|---|---|---|
| `.foregroundColor(_:)` | `.foregroundStyle(_:)` | iOS 15 |
| `NavigationView` | `NavigationStack` / `NavigationSplitView` | iOS 16 |
| `.cornerRadius(_:)` | `.clipShape(.rect(cornerRadius:))` | Deprecated |
| `.background(Color.red)` | `.background(.red)` (ShapeStyle) | iOS 15 |
| `@StateObject` | `@State` + `@Observable` class | iOS 17 |
| `@ObservedObject` | plain `let` or `@Bindable` | iOS 17 |
| `@Published` | `@Observable` (automatic tracking) | iOS 17 |
| `@EnvironmentObject` | `@Environment(MyType.self)` | iOS 17 |
| `.onChange(of:) { newValue in }` | `.onChange(of:) { oldValue, newValue in }` | iOS 17 |
| `GeometryReader` (for sizing) | `containerRelativeFrame` | iOS 17 |
| `UIScreen.main.bounds` | `GeometryReader` or `containerRelativeFrame` | iOS 16 |
| `.onAppear { Task { } }` | `.task { }` | iOS 15 |
| `.accentColor(_:)` | `.tint(_:)` | iOS 15 |
| `.tabItem { Label(...) }` | `Tab("Title", systemImage:) { }` | iOS 18 |
| `toolbar { ToolbarItem(placement: .navigationBarTrailing)` | `toolbar { ToolbarItem(placement: .topBarTrailing)` | iOS 16 |
| `AnyView(SomeView())` | `@ViewBuilder` functions / `some View` | Best practice |
| `.font(.system(size: 24))` | `.font(.title)` (Dynamic Type) | Prefer semantic fonts |
| `EnvironmentKey` protocol + extension | `@Entry var` in EnvironmentValues extension | iOS 18 |
| `UIImpactFeedbackGenerator(style:).impactOccurred()` | `.sensoryFeedback(.impact, trigger:)` | iOS 17 |
| Custom VStack empty state | `ContentUnavailableView` (as `.overlay`, never inside List) | iOS 17 |
| `Task.sleep(nanoseconds:)` | `Task.sleep(for: .seconds(N))` | iOS 16 |
| `FileManager.default.urls(for:in:)` | `URL.documentsDirectory` / `.cachesDirectory` / `.temporaryDirectory` | iOS 16 |
| `UIGraphicsImageRenderer` | `ImageRenderer` | iOS 16 |
| `NavigationLink { DetailView() } label:` | `NavigationLink(value:)` + `.navigationDestination(for:)` | iOS 16 |
| `GeometryReader` (for effects) | `.visualEffect { content, proxy in }` | iOS 17 |

## Detailed Replacements

### foregroundColor → foregroundStyle

```swift
// ❌ Deprecated
Text("Hello").foregroundColor(.blue)
Text("Hello").foregroundColor(Color.red)

// ✅ Modern — supports gradients, hierarchical styles
Text("Hello").foregroundStyle(.blue)
Text("Hello").foregroundStyle(.red)
Text("Hello").foregroundStyle(.linearGradient(stops: [.init(color: .blue, location: 0), .init(color: .purple, location: 1)], startPoint: .leading, endPoint: .trailing))
Text("Hello").foregroundStyle(.secondary)
```

### NavigationView → NavigationStack

```swift
// ❌ Deprecated
NavigationView {
    List { ... }
        .navigationTitle("Items")
}

// ✅ Modern — see navigation-sheets.md for full patterns
NavigationStack {
    List { ... }
        .navigationTitle("Items")
}
```

### cornerRadius → clipShape

```swift
// ❌ Deprecated behavior (clips without proper shape)
Image("photo")
    .cornerRadius(12)

// ✅ Modern
Image("photo")
    .clipShape(.rect(cornerRadius: 12))

// ✅ With specific corners
content.clipShape(.rect(topLeadingRadius: 16, topTrailingRadius: 16))

// ✅ Other shapes
content.clipShape(.capsule)
content.clipShape(.circle)
```

### onChange — Two-Parameter Closure

```swift
// ❌ Deprecated single-parameter
.onChange(of: searchText) { newValue in
    performSearch(newValue)
}

// ✅ Modern two-parameter (even if you don't need oldValue)
.onChange(of: searchText) { oldValue, newValue in
    performSearch(newValue)
}

// ✅ If you don't need either value
.onChange(of: searchText) {
    performSearch()
}
```

### GeometryReader → containerRelativeFrame

```swift
// ❌ Using GeometryReader just for sizing
GeometryReader { geo in
    Image("banner")
        .frame(width: geo.size.width, height: geo.size.width * 0.5)
}

// ✅ Modern — no layout side effects
Image("banner")
    .containerRelativeFrame(.horizontal) { width, _ in
        width
    }
    .aspectRatio(2, contentMode: .fit)

// ✅ Half-width card
CardView()
    .containerRelativeFrame(.horizontal) { width, _ in
        width * 0.5
    }
```

### Button vs onTapGesture

```swift
// ❌ Using onTapGesture for buttons (no accessibility, no highlight)
Text("Delete")
    .foregroundStyle(.red)
    .onTapGesture { deleteItem() }

// ✅ Use Button — gets accessibility, hover, highlight for free
Button("Delete", role: .destructive) { deleteItem() }

// ✅ Custom styled button
Button { deleteItem() } label: {
    Label("Delete", systemImage: "trash")
        .foregroundStyle(.red)
}

// ✅ onTapGesture is appropriate when you need tap count
content.onTapGesture(count: 2) { doubleTapAction() }
```

### Task Modifier vs onAppear

```swift
// ❌ Awkward async in onAppear
.onAppear {
    Task { await loadData() }
}

// ✅ .task — auto-cancels when view disappears
.task { await loadData() }

// ✅ .task with ID — re-runs when value changes
.task(id: selectedCategory) {
    await loadItems(for: selectedCategory)
}
```

### Background and Overlay Styles

```swift
// ❌ Explicit Color type
.background(Color.blue)
.overlay(Color.black.opacity(0.3))

// ✅ ShapeStyle inference
.background(.blue)
.background(.ultraThinMaterial)
.background(.thinMaterial)
.overlay(.black.opacity(0.3))

// ✅ Background with shape
.background(.blue, in: .rect(cornerRadius: 12))
.background(.regularMaterial, in: .capsule)
```

### Toolbar Placement

```swift
// ❌ Old placement names
.toolbar {
    ToolbarItem(placement: .navigationBarTrailing) { ... }
    ToolbarItem(placement: .navigationBarLeading) { ... }
    ToolbarItem(placement: .bottomBar) { ... }
}

// ✅ Modern placement names
.toolbar {
    ToolbarItem(placement: .topBarTrailing) { ... }
    ToolbarItem(placement: .topBarLeading) { ... }
    ToolbarItem(placement: .bottomBar) { ... }
}
```

### tabItem → Tab (iOS 18+)

```swift
// ❌ Legacy tabItem
TabView {
    HomeView()
        .tabItem {
            Label("Home", systemImage: "house")
        }
    SearchView()
        .tabItem {
            Label("Search", systemImage: "magnifyingglass")
        }
}

// ✅ Modern Tab API — type-safe, supports search role
TabView {
    Tab("Home", systemImage: "house") {
        HomeView()
    }
    Tab(role: .search) {
        SearchView()
    }
}

// ✅ With programmatic selection (value: parameter)
@State private var selectedTab = 0

TabView(selection: $selectedTab) {
    Tab("Home", systemImage: "house", value: 0) {
        HomeView()
    }
    Tab("Search", systemImage: "magnifyingglass", value: 1) {
        SearchView()
    }
}

// ⚠️ NEVER mix Tab with .tabItem — causes silent failures
```

### String(format:) → .formatted()

```swift
// ❌ Old C-style formatting
Text(String(format: "%.2f%%", progress * 100))
Text(String(format: "$%.2f", price))

// ✅ Modern formatted()
Text(progress.formatted(.percent.precision(.fractionLength(2))))
Text(price.formatted(.currency(code: "USD")))
```

### AnyView → @ViewBuilder

```swift
// ❌ Type erasure has performance cost, hides identity
func content() -> AnyView {
    if condition {
        return AnyView(Text("A"))
    } else {
        return AnyView(Image(systemName: "photo"))
    }
}

// ✅ Use @ViewBuilder
@ViewBuilder
func content() -> some View {
    if condition {
        Text("A")
    } else {
        Image(systemName: "photo")
    }
}
```

### Button Image Accessibility

```swift
// ❌ Image-only button — no accessibility label
Button { addItem() } label: {
    Image(systemName: "plus")
}

// ✅ Always include text for accessibility
Button("Add Item", systemImage: "plus") { addItem() }
```

### @Entry — Custom Environment Values (iOS 18+)

```swift
// ❌ Old way — verbose boilerplate
private struct AccentColorKey: EnvironmentKey {
    static let defaultValue: Color = .blue
}
extension EnvironmentValues {
    var accentThemeColor: Color {
        get { self[AccentColorKey.self] }
        set { self[AccentColorKey.self] = newValue }
    }
}

// ✅ Modern — one line with @Entry
extension EnvironmentValues {
    @Entry var accentThemeColor: Color = .blue
}

// Usage is identical
.environment(\.accentThemeColor, .purple)

@Environment(\.accentThemeColor) private var accentThemeColor
```

Also works with `FocusedValues`, `ContainerValues`, and `Transaction`.

### NavigationLink — Value-Based (iOS 16+)

```swift
// ❌ Inline destination — creates destination eagerly, can't programmatically navigate
NavigationLink {
    DetailView(item: item)
} label: {
    Text(item.title)
}

// ✅ Value-based — lazy, type-safe, supports programmatic navigation
NavigationLink(value: item) {
    Text(item.title)
}
.navigationDestination(for: Item.self) { item in
    DetailView(item: item)
}
```

### visualEffect — Geometry-Based Effects Without GeometryReader (iOS 17+)

```swift
// ❌ GeometryReader just for visual effects — disrupts layout
GeometryReader { proxy in
    content
        .scaleEffect(proxy.size.width / 300)
        .opacity(proxy.frame(in: .global).minY / 500)
}

// ✅ Modern — no layout side effects
content
    .visualEffect { content, proxy in
        content
            .scaleEffect(proxy.size.width / 300)
            .opacity(proxy.frame(in: .global).minY / 500)
    }
```

### sensoryFeedback — SwiftUI Haptics (iOS 17+)

```swift
// ❌ Old UIKit approach
let generator = UIImpactFeedbackGenerator(style: .medium)
generator.impactOccurred()

// ✅ Modern — declarative SwiftUI modifier
Button("Add") { addItem() }
    .sensoryFeedback(.impact(weight: .medium), trigger: itemCount)

// Notification-style feedback
.sensoryFeedback(.success, trigger: isComplete)
.sensoryFeedback(.warning, trigger: hasError)
.sensoryFeedback(.error, trigger: failureCount)

// Conditional — only fire when condition is met
.sensoryFeedback(.selection, trigger: selectedIndex) { oldValue, newValue in
    oldValue != newValue
}
```

Available feedback types: `.impact(weight:intensity:)`, `.selection`, `.success`, `.warning`, `.error`, `.increase`, `.decrease`, `.start`, `.stop`, `.alignment`, `.levelChange`, `.pathComplete`.

### MeshGradient (iOS 18+)

```swift
// Organic mesh gradient background
MeshGradient(
    width: 3, height: 3,
    points: [
        [0.0, 0.0], [0.5, 0.0], [1.0, 0.0],
        [0.0, 0.5], [0.5, 0.5], [1.0, 0.5],
        [0.0, 1.0], [0.5, 1.0], [1.0, 1.0]
    ],
    colors: [
        .red, .purple, .indigo,
        .orange, .cyan, .blue,
        .yellow, .green, .mint
    ]
)
.ignoresSafeArea()
```

Points are `SIMD2<Float>` values in the 0...1 range. Animate by changing point positions or colors with `withAnimation`. Useful for creative app backgrounds, onboarding screens, and hero sections.
