# SwiftUI List & ForEach Patterns

> Use stable identity. Never use .indices for dynamic content. Prefer List for built-in features.

## ForEach Identity Rules

ForEach needs **stable, unique identity** for each element to correctly animate, reorder, and maintain view state.

### Use Identifiable (Preferred)

```swift
struct Task: Identifiable {
    let id = UUID()
    var title: String
    var isCompleted: Bool
}

ForEach(tasks) { task in
    TaskRow(task: task)
}
```

### Use id: Parameter for Non-Identifiable Types

```swift
ForEach(categories, id: \.name) { category in
    Text(category.name)
}
```

### Never Use .indices for Dynamic Content

```swift
// ❌ .indices breaks identity — items lose state on insert/delete, animations break
ForEach(items.indices, id: \.self) { index in
    ItemRow(item: items[index])
}

// ❌ enumerated() with offset is the same problem
ForEach(Array(items.enumerated()), id: \.offset) { index, item in
    ItemRow(item: item)
}

// ✅ Use Identifiable items
ForEach(items) { item in
    ItemRow(item: item)
}

// ✅ If you need the index, use zip
ForEach(Array(zip(items.indices, items)), id: \.1.id) { index, item in
    Text("\(index + 1). \(item.name)")
}
```

### Why .indices Breaks

When you insert or delete from an array:
- Index 0 stays index 0, but points to a different item
- SwiftUI thinks the view at index 0 is the "same" view (same identity)
- Animations show the wrong item morphing instead of inserting/removing
- Any @State inside the row gets attached to the wrong item

### Wrap ForEach Bodies in a Container

Always wrap ForEach content in a single container (VStack, HStack, etc). Conditionals inside the container are fine — identity comes from the `Identifiable` ID, not the view structure.

```swift
// ❌ Multiple top-level views without a container
ForEach(items) { item in
    Text(item.name)
    if item.hasSubtitle {
        Text(item.subtitle)
    }
}

// ✅ Wrapped in a container — conditionals inside are fine
ForEach(items) { item in
    VStack(alignment: .leading) {
        Text(item.name)
        if item.hasSubtitle {
            Text(item.subtitle)
        }
    }
}
```

## List — Built-In Features

List provides swipe actions, selection, separators, and section headers automatically.

### Basic List

```swift
struct ItemListView: View {
    @State private var items: [Item] = []

    var body: some View {
        List(items) { item in
            ItemRow(item: item)
        }
    }
}
```

### Swipe Actions

```swift
List(items) { item in
    ItemRow(item: item)
        .swipeActions(edge: .trailing, allowsFullSwipe: true) {
            Button(role: .destructive) {
                deleteItem(item)
            } label: {
                Label("Delete", systemImage: "trash")
            }
        }
        .swipeActions(edge: .leading) {
            Button {
                togglePin(item)
            } label: {
                Label(item.isPinned ? "Unpin" : "Pin", systemImage: "pin")
            }
            .tint(.orange)
        }
}
```

### Selection

```swift
struct SelectableList: View {
    @State private var items: [Item] = []
    @State private var selection: Set<Item.ID> = []

    var body: some View {
        List(items, selection: $selection) { item in
            ItemRow(item: item)
        }
        .toolbar {
            EditButton()
        }
    }
}
```

### Sections

```swift
List {
    Section("Favorites") {
        ForEach(favorites) { item in
            ItemRow(item: item)
        }
    }

    Section("All Items") {
        ForEach(allItems) { item in
            ItemRow(item: item)
        }
    }
}
```

### List Styles

```swift
List { ... }
    .listStyle(.plain)          // no background, minimal
    .listStyle(.insetGrouped)   // iOS Settings-style (default)
    .listStyle(.grouped)        // grouped with gray background
    .listStyle(.sidebar)        // sidebar navigation style
```

### Custom List Background

```swift
List(items) { item in
    ItemRow(item: item)
        .listRowSeparator(.hidden)
        .listRowInsets(EdgeInsets(top: 8, leading: 16, bottom: 8, trailing: 16))
}
.listStyle(.plain)
.scrollContentBackground(.hidden)       // remove default List background
.background(Color.customBackground)
.environment(\.defaultMinListRowHeight, 1)  // allow custom row heights
```

### Pull to Refresh

```swift
List(items) { item in
    ItemRow(item: item)
}
.refreshable {
    await fetchItems()
}
```

### Search

```swift
struct SearchableList: View {
    @State private var items: [Item] = []
    @State private var searchText = ""

    private var filteredItems: [Item] {
        guard !searchText.isEmpty else { return items }
        return items.filter { $0.name.localizedCaseInsensitiveContains(searchText) }
    }

    var body: some View {
        List(filteredItems) { item in
            ItemRow(item: item)
        }
        .searchable(text: $searchText, prompt: "Search items")
    }
}
```

## List vs LazyVStack

| Feature | List | LazyVStack in ScrollView |
|---------|------|--------------------------|
| Swipe actions | Built-in | Manual |
| Selection | Built-in | Manual |
| Separators | Automatic | Manual |
| Section headers | Built-in (sticky) | Via pinnedViews |
| Pull to refresh | .refreshable | .refreshable |
| Edit mode / reorder | Built-in | Manual |
| Custom layout | Limited | Full control |
| Performance | Excellent | Excellent |

**Use List** when you need swipe actions, selection, or edit mode.
**Use LazyVStack** when you need full layout control or custom design.

## Avoid Inline Filtering in ForEach

```swift
// ❌ Filtering inside ForEach — inefficient, runs every render
ForEach(items.filter { $0.isActive }) { item in
    ItemRow(item: item)
}

// ✅ Use a computed property
private var activeItems: [Item] {
    items.filter { $0.isActive }
}

var body: some View {
    ForEach(activeItems) { item in
        ItemRow(item: item)
    }
}
```

## Empty State — Always Use ContentUnavailableView

**ALWAYS use `ContentUnavailableView` for empty states.** Never build custom VStack+Image+Text empty states — `ContentUnavailableView` provides correct layout, spacing, Dynamic Type support, and visual consistency out of the box.

**CRITICAL: Place it as an `.overlay` on the container, NEVER inside `List`/`ScrollView`.** Placing it inside a scroll container breaks vertical centering — the view gets pushed to the top instead of centering on screen.

```swift
// ✅ Correct — overlay on List, centers perfectly
List(items) { item in
    ItemRow(item: item)
}
.overlay {
    if items.isEmpty {
        ContentUnavailableView(
            "No Items",
            systemImage: "tray",
            description: Text("Add items to get started")
        )
    }
}

// ❌ WRONG — inside List, breaks centering
List {
    if items.isEmpty {
        ContentUnavailableView("No Items", systemImage: "tray")
    } else {
        ForEach(items) { item in ItemRow(item: item) }
    }
}

// ❌ WRONG — custom empty state when ContentUnavailableView exists
if items.isEmpty {
    VStack(spacing: 12) {
        Image(systemName: "tray")
            .font(.largeTitle)
        Text("No Items")
            .font(.title2)
    }
    .foregroundStyle(.secondary)
}
```

### With Action Button

```swift
ContentUnavailableView {
    Label("No Favorites", systemImage: "heart")
} description: {
    Text("Items you favorite will appear here.")
} actions: {
    Button("Browse Items") { showBrowse = true }
        .buttonStyle(.borderedProminent)
}
```

### Empty Search Results

Use the built-in `.search` convenience — it automatically shows "No Results for [query]":

```swift
List(filteredItems) { item in
    ItemRow(item: item)
}
.searchable(text: $searchText)
.overlay {
    if filteredItems.isEmpty && !searchText.isEmpty {
        ContentUnavailableView.search(text: searchText)
    }
}
```

### Non-List Empty States

`ContentUnavailableView` works for any screen, not just lists:

```swift
// Tab with no content yet
struct FavoritesTab: View {
    let favorites: [Item]

    var body: some View {
        if favorites.isEmpty {
            ContentUnavailableView(
                "No Favorites",
                systemImage: "heart",
                description: Text("Tap the heart icon on items to add them here.")
            )
        } else {
            ScrollView { /* content */ }
        }
    }
}
```

## Delete and Move

```swift
struct EditableList: View {
    @State private var items: [Item] = []

    var body: some View {
        List {
            ForEach(items) { item in
                ItemRow(item: item)
            }
            .onDelete { indexSet in
                items.remove(atOffsets: indexSet)
            }
            .onMove { source, destination in
                items.move(fromOffsets: source, toOffset: destination)
            }
        }
        .toolbar { EditButton() }
    }
}
```

## Constant Range ForEach

For a fixed number of items (like a star rating), plain range is fine:

```swift
// ✅ This is fine — the range never changes
ForEach(1...5, id: \.self) { star in
    Image(systemName: star <= rating ? "star.fill" : "star")
        .foregroundStyle(.yellow)
}
```
