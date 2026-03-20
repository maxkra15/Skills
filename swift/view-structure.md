# SwiftUI View Structure & Composition Guide

> Keep view bodies declarative. Move logic to computed properties. Modifier order matters.

## Modifier Ordering

Modifier order matters. Apply in this general sequence:

```swift
Text("Hello")
    // 1. Content modifiers (font, color, text-specific)
    .font(.headline)
    .foregroundStyle(.primary)
    .lineLimit(2)
    // 2. Layout modifiers (frame, padding, alignment)
    .frame(maxWidth: .infinity, alignment: .leading)
    .padding()
    // 3. Background / overlay / border
    .background(.regularMaterial, in: .rect(cornerRadius: 12))
    .overlay {
        RoundedRectangle(cornerRadius: 12).strokeBorder(.separator)
    }
    // 4. Clipping
    .clipShape(.rect(cornerRadius: 12))
    // 5. Shadow
    .shadow(color: .black.opacity(0.1), radius: 8, y: 4)
    // 6. Interaction modifiers
    .onTapGesture { }
    // 7. Lifecycle / data flow
    .task { }
    .onChange(of: value) { _, _ in }
```

### Padding vs Background Order

```swift
// padding THEN background → background includes padding
Text("Tag")
    .padding(.horizontal, 12)
    .padding(.vertical, 6)
    .background(.blue, in: .capsule)

// background THEN padding → padding is outside background
Text("Tag")
    .background(.blue, in: .capsule)
    .padding()  // adds space around the background
```

## View Identity & Stability

SwiftUI uses **structural identity** (position in the view tree) and **explicit identity** (id, ForEach) to track views. Getting this wrong causes lost state and broken animations.

### Don't Swap Views Conditionally at the Same Position

```swift
// ❌ SwiftUI sees these as different views — state is lost, no smooth animation
if isGrid {
    GridView(items: items)  // identity = branch 0
} else {
    ListView(items: items)  // identity = branch 1
}

// ✅ Use a single view with different layout
ScrollView {
    let columns = isGrid
        ? [GridItem(.adaptive(minimum: 150))]
        : [GridItem(.flexible())]
    LazyVGrid(columns: columns) {
        ForEach(items) { item in
            ItemCell(item: item, isGrid: isGrid)
        }
    }
}
```

### Don't Create Objects or Do Computation in Body

```swift
// ❌ Logic, object creation in body — re-runs every render
var body: some View {
    let formatter = DateFormatter()
    formatter.dateStyle = .medium
    let sorted = items.sorted { $0.date > $1.date }
    List(sorted) { item in
        Text(formatter.string(from: item.date))
    }
}

// ✅ Move to computed properties
private var sortedItems: [Item] {
    items.sorted { $0.date > $1.date }
}

var body: some View {
    List(sortedItems) { item in
        Text(item.date, style: .date)
    }
}
```

## Extract Subviews vs Computed Properties

For large views, prefer extracting to **separate struct views** over `@ViewBuilder` computed properties. SwiftUI can skip a struct's `body` when its inputs haven't changed, but computed properties re-execute on every parent state change.

```swift
// ✅ Best — struct can be skipped when inputs unchanged
struct ParentView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Button("Tap: \(count)") { count += 1 }
            ExpensiveSection()  // body SKIPPED during re-evaluation
        }
    }
}

struct ExpensiveSection: View {
    var body: some View {
        ForEach(0..<100) { i in
            HStack { Image(systemName: "star"); Text("Item \(i)"); Spacer() }
        }
    }
}

// ⚠️ Acceptable for small sections — re-executes on every parent update
private var headerSection: some View {
    HStack {
        Text("Title")
        Spacer()
        Button("Action") { }
    }
}
```

Use **struct extraction** when:
- The section has many views or expensive rendering
- You need to isolate state changes for performance
- The component is reusable

Use **computed properties** when:
- The section is small (2-5 simple views)
- It needs direct access to parent's state without passing many parameters

## Prefer Modifiers Over Conditional Views

When toggling visibility of the **same view** (not swapping different views), prefer modifiers that preserve view identity:

```swift
// ✅ Same view, different states — preserves identity, animates smoothly
SomeView()
    .opacity(isVisible ? 1 : 0)

// ❌ Creates/destroys view — state lost, no smooth transition
if isVisible {
    SomeView()
}
```

Use conditionals when the views are **fundamentally different**:
```swift
// ✅ Different views — conditional is correct
if isLoggedIn {
    DashboardView()
} else {
    LoginView()
}
```

## Conditional Modifiers — Use Ternary, Not .if() Extensions

When you need to conditionally apply a modifier, use ternary expressions directly. Never create a `.if()` View extension — it changes structural identity on every toggle, breaking animations and losing `@State`.

```swift
// ✅ Ternary — same view identity, animates smoothly
Text("Hello")
    .foregroundStyle(isActive ? .blue : .primary)
    .fontWeight(isActive ? .bold : .regular)
    .background(isActive ? AnyShapeStyle(.blue.opacity(0.1)) : AnyShapeStyle(.clear))

// ❌ Anti-pattern — switches structural identity, breaks state & animations
extension View {
    func `if`<T: View>(_ condition: Bool, transform: (Self) -> T) -> some View {
        if condition { transform(self) } else { self }
    }
}
Text("Hello").if(isActive) { $0.bold().foregroundStyle(.blue) }
```

For modifiers that can't be ternary'd (e.g. adding a `.sheet` or `.overlay` conditionally), it's fine to use `if` in the body — those represent different view structures, not two states of the same view.

## Theming with @Observable

Use a single `Theme` object as the source of truth. Inject at app root, read via `@Environment`.

```swift
@MainActor
@Observable
final class Theme {
    var tintColor: Color = .blue
    var primaryBackground: Color = Color(.systemBackground)
    var secondaryBackground: Color = Color(.secondarySystemBackground)
    var labelColor: Color = .primary
}
```

### Inject at App Root

```swift
@main
struct MyApp: App {
    @State private var theme = Theme()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(theme)
        }
    }
}
```

### Read in Views

```swift
@Environment(Theme.self) private var theme

Text("Title")
    .foregroundStyle(theme.labelColor)
    .background(theme.primaryBackground)
```

Use semantic names (`primaryBackground`, `labelColor`, `tint`) — never sprinkle raw `Color` values in views. Store user-selected values in persistent storage if needed.
