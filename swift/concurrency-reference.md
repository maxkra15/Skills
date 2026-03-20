# Swift Concurrency Reference

> Start with @MainActor for UI code. Add async/await for I/O. Only move CPU work off-main when profiling shows you need it.

## @MainActor — UI Thread Isolation

### On Classes (ViewModel Pattern)

```swift
@MainActor
@Observable
final class ProfileViewModel {
    var name = ""
    var isLoading = false

    func loadProfile() async {
        isLoading = true
        let profile = try? await ProfileService.fetch()
        name = profile?.name ?? ""
        isLoading = false
    }
}
```

### On Functions

```swift
@MainActor
func updateUI(with data: Data) {
    label.text = String(data: data, encoding: .utf8)
}
```

### nonisolated — Opt Out of MainActor

```swift
@MainActor
@Observable
final class ImageLoader {
    var image: UIImage?

    nonisolated func processImage(_ data: Data) -> UIImage? {
        UIImage(data: data)?.preparingThumbnail(of: CGSize(width: 200, height: 200))
    }
}
```

## async/await

### Basic Async Function

```swift
func fetchUser(id: String) async throws -> User {
    let url = URL(string: "https://api.example.com/users/\(id)")!
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data)
}
```

### Calling from SwiftUI

```swift
struct UserView: View {
    @State private var user: User?

    var body: some View {
        VStack {
            if let user {
                Text(user.name)
            }
        }
        .task {
            user = try? await fetchUser(id: "123")
        }
    }
}
```
