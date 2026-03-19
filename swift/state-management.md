# SwiftUI State Management Guide

> Target: iOS 18+. Use @Observable as the default. Never use ObservableObject for new code.

## Property Wrapper Decision Guide

```
Is this value owned by THIS view?
├── YES → Is it a simple value type (String, Int, Bool, enum, struct)?
│   ├── YES → @State private var
│   └── NO → Is it a reference type (class)?
│       └── YES → @State private var + make the class @Observable
├── NO → Is it passed from a parent?
│   ├── Does the child need to WRITE to it?
│   │   ├── YES → @Binding var
│   │   └── NO → Does it need to REACT to changes with .onChange()?
│   │       ├── YES → var (not let) + .onChange(of:)
│   │       └── NO → let (plain property)
│   └── Is it an @Observable class from a parent?
│       ├── Child reads only → let (plain property, tracking is automatic)
│       ├── Child needs to mutate a property → @Bindable var
│       └── Child needs the whole object as binding → @Binding var
└── Is it from the environment?
    ├── System value → @Environment(\.colorScheme) var
    └── Custom @Observable → @Environment(MyService.self) var
```

## @State — Local View State

@State is for values **owned and created** by the view. Always `private`.

```swift
// ✅ Correct — view owns this value
struct CounterView: View {
    @State private var count = 0

    var body: some View {
        Button("Count: \(count)") { count += 1 }
    }
}

// ❌ WRONG — @State for a value passed from parent
struct ProfileView: View {
    @State var username: String  // BUG: creates a copy, parent changes ignored

    var body: some View {
        Text(username)
    }
}

// ✅ Fix — use let for read-only passed values
struct ProfileView: View {
    let username: String

    var body: some View {
        Text(username)
    }
}
```

## @Observable — Shared Mutable State

@Observable replaces ObservableObject. SwiftUI automatically tracks which properties each view reads and only re-renders when those specific properties change.

**Always mark `@Observable` classes with `@MainActor`** to guarantee UI-safe property access:

```swift
@Observable
@MainActor
class ShoppingCart {
    var items: [Item] = []
    var couponCode: String = ""

    var total: Double {
        items.reduce(0) { $0 + $1.price }
    }

    func addItem(_ item: Item) {
        items.append(item)
    }
}
```

### Creating in a View (view owns it)

```swift
struct ShopView: View {
    @State private var cart = ShoppingCart()

    var body: some View {
        VStack {
            ItemListView(cart: cart)
            CartSummary(cart: cart)
        }
    }
}
```

### Passing to Child Views

```swift
// Child that only READS — plain let
struct CartSummary: View {
    let cart: ShoppingCart

    var body: some View {
        Text("Total: \(cart.total, format: .currency(code: "USD"))")
    }
}

// Child that WRITES to a property — @Bindable
struct CouponField: View {
    @Bindable var cart: ShoppingCart

    var body: some View {
        TextField("Coupon", text: $cart.couponCode)
    }
}
```

### @Bindable — Create Bindings to @Observable Properties

@Bindable lets you use `$` syntax to get bindings to properties of an @Observable object.

```swift
struct EditProfileView: View {
    @Bindable var profile: UserProfile

    var body: some View {
        Form {
            TextField("Name", text: $profile.name)
            Toggle("Notifications", isOn: $profile.notificationsEnabled)
            Picker("Theme", selection: $profile.theme) {
                ForEach(Theme.allCases, id: \.self) { Text($0.rawValue) }
            }
        }
    }
}
```

## @Environment with @Observable

Inject @Observable objects into the environment for deep view hierarchies.

```swift
@Observable
@MainActor
class AppSettings {
    var accentColor: Color = .blue
    var fontSize: Double = 16
}

// Inject at a high level
struct MyApp: App {
    @State private var settings = AppSettings()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(settings)
        }
    }
}

// Read anywhere in the hierarchy
struct SettingsConsumer: View {
    @Environment(AppSettings.self) private var settings

    var body: some View {
        Text("Hello")
            .font(.system(size: settings.fontSize))
    }
}

// Write with @Bindable
struct SettingsEditor: View {
    @Environment(AppSettings.self) private var settings

    var body: some View {
        @Bindable var settings = settings
        Slider(value: $settings.fontSize, in: 12...24)
    }
}
```

## ViewModel Pattern with @Observable

```swift
@Observable
@MainActor
class RecipeListViewModel {
    var recipes: [Recipe] = []
    var isLoading = false
    var errorMessage: String?
    var searchText = ""

    var filteredRecipes: [Recipe] {
        guard !searchText.isEmpty else { return recipes }
        return recipes.filter { $0.name.localizedCaseInsensitiveContains(searchText) }
    }

    func fetchRecipes() async {
        isLoading = true
        defer { isLoading = false }
        do {
            recipes = try await RecipeService.fetchAll()
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}

struct RecipeListView: View {
    @State private var viewModel = RecipeListViewModel()

    var body: some View {
        @Bindable var viewModel = viewModel

        List(viewModel.filteredRecipes) { recipe in
            RecipeRow(recipe: recipe)
        }
        .searchable(text: $viewModel.searchText)
        .overlay {
            if viewModel.isLoading { ProgressView() }
        }
        .task { await viewModel.fetchRecipes() }
    }
}
```

## @AppStorage — Persistent User Preferences

For simple values persisted in UserDefaults. Always `private`.

```swift
@AppStorage("hasOnboarded") private var hasOnboarded = false
@AppStorage("theme") private var theme = "system"
```

## Anti-Patterns

### Don't use @State for passed values
```swift
// ❌ Creates an independent copy — parent updates are lost
struct ChildView: View {
    @State var name: String  // BUG
}

// ✅ Use let for read-only, @Binding for read-write
struct ChildView: View {
    let name: String
}
```

### Don't use ObservableObject for new code
```swift
// ❌ Legacy pattern
class MyVM: ObservableObject {
    @Published var items: [Item] = []
}
struct MyView: View {
    @StateObject var vm = MyVM()
}

// ✅ Modern pattern
@Observable
@MainActor
class MyVM {
    var items: [Item] = []
}
struct MyView: View {
    @State private var vm = MyVM()
}
```

### Don't create @Observable objects inside body
```swift
// ❌ Recreated every render
var body: some View {
    let vm = MyViewModel()  // BUG: new instance each render
    ChildView(viewModel: vm)
}

// ✅ Use @State
@State private var vm = MyViewModel()
```

### Don't use @State for @Observable objects without private
```swift
// ❌ Missing private — can be set from outside
@State var settings = AppSettings()

// ✅ Always private when using @State
@State private var settings = AppSettings()
```
