# SwiftUI Navigation & Sheets Guide

> Use NavigationStack with NavigationPath for type-safe routing. Use sheet(item:) for data-driven presentation. Use presentationDetents for partial sheets.

## Type-Safe Destinations

```swift
enum Route: Hashable {
    case profile(User)
    case settings
    case detail(Item)
}

struct ContentView: View {
    @State private var path = NavigationPath()

    var body: some View {
        NavigationStack(path: $path) {
            List {
                NavigationLink("Profile", value: Route.profile(currentUser))
                NavigationLink("Settings", value: Route.settings)

                ForEach(items) { item in
                    NavigationLink(item.title, value: Route.detail(item))
                }
            }
            .navigationTitle("Home")
            .navigationDestination(for: Route.self) { route in
                switch route {
                case .profile(let user): ProfileView(user: user)
                case .settings: SettingsView()
                case .detail(let item): DetailView(item: item)
                }
            }
        }
    }
}
```

## Programmatic Navigation

```swift
// Push
path.append(Route.detail(item))

// Pop one level
path.removeLast()

// Pop to root
path.removeLast(path.count)
```

## Presentation Detents (Partial Sheets)

```swift
.sheet(isPresented: $showFilter) {
    FilterView()
        .presentationDetents([.medium, .large])
        .presentationDragIndicator(.visible)
        .presentationCornerRadius(20)
        .presentationBackgroundInteraction(.enabled)  // interact with content behind sheet
        .presentationContentInteraction(.scrolls)  // scroll content first, resize sheet only at scroll bounds
}

// Background interaction only up to a specific detent
.presentationBackgroundInteraction(.enabled(upThrough: .medium))

// Custom detent sizes
.presentationDetents([
    .fraction(0.3),      // 30% of screen
    .height(200),        // fixed 200pt
    .medium,             // ~half screen
    .large               // full sheet
])

// Programmatic detent control — track and change current detent
@State private var selectedDetent: PresentationDetent = .medium

.sheet(isPresented: $showSheet) {
    SheetContent()
        .presentationDetents([.medium, .large], selection: $selectedDetent)
}

```

## Item-Based Sheet (Preferred for Data-Driven)

```swift
@State private var selectedItem: Item?

List(items) { item in
    Button(item.title) { selectedItem = item }
}
.sheet(item: $selectedItem) { item in
    DetailSheet(item: item)
}
```

## Sheet with Toolbar (Create/Edit Pattern)

```swift
struct CreateItemSheet: View {
    @Environment(\.dismiss) private var dismiss
    @State private var title = ""

    var body: some View {
        NavigationStack {
            Form {
                TextField("Title", text: $title)
            }
            .navigationTitle("New Item")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("Cancel") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("Save") {
                        saveItem()
                        dismiss()
                    }
                    .disabled(title.isEmpty)
                }
            }
        }
    }
}
```

## Sheets Own Their Actions

Sheets should handle dismiss and data operations internally. Don't pass callbacks from parent.

```swift
// ✅ Sheet handles its own save and dismiss
struct EditItemSheet: View {
    @Environment(\.dismiss) private var dismiss
    @Environment(DataStore.self) private var store
    let item: Item
    @State private var name: String

    init(item: Item) {
        self.item = item
        _name = State(initialValue: item.name)
    }

    var body: some View {
        NavigationStack {
            Form {
                TextField("Name", text: $name)
            }
            .navigationTitle("Edit")
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("Cancel") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("Save") {
                        Task {
                            await store.updateItem(item, name: name)
                            dismiss()
                        }
                    }
                    .disabled(name.isEmpty)
                }
            }
        }
    }
}

// ❌ Avoid — callback prop-drilling from parent
EditItemSheet(item: item, onSave: { name in ... }, onCancel: { ... })
```

## Sheet Content Must Be Edge-to-Edge

The system sheet already provides background, corner radius, and safe-area insets. Adding manual `.padding()`, `.background()`, or `.clipShape()` to the root of sheet content creates a "card inside a card" — visible gaps at the edges and mismatched bottom corners on the device.

```swift
// ❌ BROKEN — gaps at edges, corners don't match device
.sheet(isPresented: $show) {
    VStack {
        Text("Content")
    }
    .padding()
    .background(Color(.systemBackground))
    .clipShape(.rect(cornerRadius: 20))
}

// ❌ ALSO BROKEN — padding on root pushes background inward
.sheet(isPresented: $show) {
    ScrollView {
        content
    }
    .padding()
    .background(.ultraThinMaterial)
}

// ✅ CORRECT — content fills sheet edge-to-edge, padding on inner content
.sheet(isPresented: $show) {
    NavigationStack {
        ScrollView {
            VStack(spacing: 16) {
                // content here
            }
            .padding()  // padding on INNER content, not root
        }
        .navigationTitle("Title")
        .navigationBarTitleDisplayMode(.inline)
    }
    .presentationDetents([.medium, .large])
    .presentationDragIndicator(.visible)
    .presentationContentInteraction(.scrolls)  // scroll content first, resize sheet only at scroll bounds
}

// ✅ CORRECT — Form/List fill edge-to-edge naturally
.sheet(isPresented: $show) {
    NavigationStack {
        Form {
            Section("Options") { /* fields */ }
        }
        .navigationTitle("Settings")
    }
    .presentationDetents([.medium])
}
```

## Never Replace the System Drag Indicator

The system `.presentationDragIndicator(.visible)` is rendered outside your content — it stays fixed at the top of the sheet regardless of scrolling. Do NOT hide it and build a custom one inside a ScrollView.

```swift
// ❌ BROKEN — custom handle inside ScrollView scrolls away with content
.sheet(isPresented: $show) {
    ScrollView {
        VStack {
            Capsule().fill(.white.opacity(0.3)).frame(width: 36, height: 5) // scrolls away!
            // content...
        }
    }
    .presentationDragIndicator(.hidden)
}

// ✅ CORRECT — use system drag indicator, it stays pinned automatically
.sheet(isPresented: $show) {
    ScrollView {
        VStack(spacing: 16) {
            // content only, no custom drag handle
        }
        .padding()
    }
    .presentationDragIndicator(.visible)
    .presentationDetents([.medium, .large])
}
```

Use `.presentationBackground()` to change the sheet background — not `.background()` on content:

```swift
// ✅ Change sheet background properly
.sheet(isPresented: $show) {
    content
        .presentationBackground(.ultraThinMaterial)
}
```

## Zoom Transitions (iOS 18+)

Morph from a source view into a pushed view or sheet. Creates a hero-like zoom effect.

```swift
@Namespace private var namespace

// Push with zoom from a list item
NavigationLink {
    DetailView(item: item)
        .navigationTransition(.zoom(sourceID: item.id, in: namespace))
} label: {
    ItemRow(item: item)
        .matchedTransitionSource(id: item.id, in: namespace)
}

// Sheet with zoom from a toolbar button
Button("Info", systemImage: "info.circle") { showSheet = true }
    .matchedTransitionSource(id: "info", in: namespace)

.sheet(isPresented: $showSheet) {
    InfoView()
        .navigationTransition(.zoom(sourceID: "info", in: namespace))
}
```

Both source and destination must use the same `@Namespace`. The source view uses `.matchedTransitionSource(id:in:)`, the destination uses `.navigationTransition(.zoom(sourceID:in:))`.

## Popover

```swift
Button("Info") { showPopover = true }
.popover(isPresented: $showPopover) {
    PopoverContent()
        .presentationCompactAdaptation(horizontal: .popover, vertical: .sheet)
}
```

## Full Screen Cover

```swift
.fullScreenCover(isPresented: $showOnboarding) {
    OnboardingView()
}

.fullScreenCover(item: $selectedPhoto) { photo in
    PhotoViewer(photo: photo)
}
```

## Alerts

```swift
@State private var showDeleteAlert = false
@State private var itemToDelete: Item?

Button("Delete") {
    itemToDelete = item
    showDeleteAlert = true
}
.alert("Delete Item?", isPresented: $showDeleteAlert, presenting: itemToDelete) { item in
    Button("Delete", role: .destructive) { delete(item) }
    Button("Cancel", role: .cancel) { }
} message: { item in
    Text("This will permanently delete "\(item.name)".")
}
```

## Confirmation Dialog

```swift
@State private var showOptions = false

Button("Options") { showOptions = true }
.confirmationDialog("Choose Action", isPresented: $showOptions, titleVisibility: .visible) {
    Button("Share") { share() }
    Button("Duplicate") { duplicate() }
    Button("Delete", role: .destructive) { delete() }
    Button("Cancel", role: .cancel) { }
}
```

## Focus Handling & Field Chaining

Use `@FocusState` to control keyboard focus, chain fields with `.onSubmit`, and set initial focus.

### Single Field Focus

```swift
@FocusState private var isNameFocused: Bool

TextField("Name", text: $name)
    .focused($isNameFocused)
    .onAppear { isNameFocused = true }
```

### Chained Focus with Enum

```swift
enum FocusField { case title, email, password }
@FocusState private var focusedField: FocusField?

Form {
    TextField("Title", text: $title)
        .focused($focusedField, equals: .title)
        .onSubmit { focusedField = .email }

    TextField("Email", text: $email)
        .focused($focusedField, equals: .email)
        .onSubmit { focusedField = .password }

    SecureField("Password", text: $password)
        .focused($focusedField, equals: .password)
}
.onAppear { focusedField = .title }
```

### Dynamic Focus for Variable Fields

```swift
enum FocusField: Hashable { case option(Int) }
@FocusState private var focused: FocusField?
@State private var options: [String] = ["", ""]

ForEach(options.indices, id: \.self) { index in
    TextField("Option \(index + 1)", text: $options[index])
        .focused($focused, equals: .option(index))
        .onSubmit {
            options.append("")
            Task { @MainActor in
                focused = .option(options.count - 1)
            }
        }
}
.onAppear { focused = .option(0) }
```

Keep focus state local to the view that owns the fields. Pair with `.scrollDismissesKeyboard(.interactively)` in ScrollView/Form.

## NavigationSplitView — Two/Three Column

```swift
struct SplitView: View {
    @State private var selectedCategory: Category.ID?
    @State private var selectedItem: Item.ID?

    var body: some View {
        NavigationSplitView {
            List(categories, selection: $selectedCategory) { category in
                Text(category.name)
            }
            .navigationTitle("Categories")
        } content: {
            if let category = categories.first(where: { $0.id == selectedCategory }) {
                List(category.items, selection: $selectedItem) { item in
                    Text(item.name)
                }
            }
        } detail: {
            if let category = categories.first(where: { $0.id == selectedCategory }),
               let item = category.items.first(where: { $0.id == selectedItem }) {
                DetailView(item: item)
            } else {
                ContentUnavailableView("Select an Item", systemImage: "sidebar.left")
            }
        }
    }
}
```
