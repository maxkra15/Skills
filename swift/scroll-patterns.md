# SwiftUI Scroll Patterns

> Use scrollPosition for programmatic scrolling (iOS 17+). Use scrollTargetBehavior for paging/snapping.

**IMPORTANT: `.scrollTargetLayout()` makes every child a snap target.** Only use it with `.scrollTargetBehavior(.paging)` or `.scrollTargetBehavior(.viewAligned)` for carousels/paging. NEVER add `.scrollTargetLayout()` to a regular feed/list — it makes scrolling jerky and unusable because the system snaps to each item.

**IMPORTANT: Horizontal scroll padding.** NEVER use `.padding(.horizontal)` on content inside a horizontal `ScrollView` — it creates permanent padding strips at scroll edges. Use `.contentMargins(.horizontal, N)` on the ScrollView instead for edge-to-edge scrolling with initial insets.

## scrollPosition(id:) — Track Scroll by Item ID (iOS 17+)

Requires `.scrollTargetLayout()` for tracking. Use for item pickers/carousels, NOT for free-scrolling feeds. For feeds, use `ScrollViewReader` instead.

```swift
struct ItemPickerView: View {
    @State private var items: [Item] = []
    @State private var scrolledID: Item.ID?

    var body: some View {
        ScrollView {
            LazyVStack {
                ForEach(items) { item in
                    ItemCard(item: item)
                }
            }
            .scrollTargetLayout()
        }
        .scrollPosition(id: $scrolledID)
        .scrollTargetBehavior(.viewAligned)
    }
}
```

## ScrollViewReader — Programmatic Scrolling

For scroll-to-item without `scrollPosition` (works on all iOS versions), or when you need anchor control:

```swift
struct ChatView: View {
    let messages: [Message]

    var body: some View {
        ScrollViewReader { proxy in
            ScrollView {
                LazyVStack {
                    ForEach(messages) { message in
                        MessageBubble(message: message)
                    }
                }
            }
            .onChange(of: messages.count) {
                withAnimation {
                    proxy.scrollTo(messages.last?.id, anchor: .bottom)
                }
            }
        }
    }
}
```

**For free-scrolling feeds:** always use `ScrollViewReader` for programmatic scroll-to-top. `scrollPosition(id:)` and `.scrollPosition($position)` require `.scrollTargetLayout()` and are designed for snap/picker views, not feeds.

## ScrollPosition — Unified Scroll Control (iOS 18+)

`ScrollPosition` struct provides full scroll control: scroll to edges, IDs, or exact offsets.

**WARNING:** `.scrollPosition($position)` can interfere with free-scrolling in feeds/lists. For scroll-to-top on a feed, use `ScrollViewReader` instead (see "Direction-based collapsing header" section below). Reserve `.scrollPosition()` for views that also use `.scrollTargetLayout()` + `.scrollTargetBehavior()`.

```swift
// Good use case: item picker with snap behavior
struct ItemPickerView: View {
    @State private var position = ScrollPosition(edge: .top)

    var body: some View {
        ScrollView {
            LazyVStack {
                ForEach(items) { item in
                    ItemCard(item: item)
                }
            }
            .scrollTargetLayout()
        }
        .scrollPosition($position)
        .scrollTargetBehavior(.viewAligned)
    }
}
```

### Scroll Phase & Visibility Tracking (iOS 18+)

```swift
// Basic phase tracking
.onScrollPhaseChange { oldPhase, newPhase in
    // .idle, .tracking, .interacting, .decelerating, .animating
    isScrolling = newPhase != .idle
}

// With geometry context — use for scroll-direction detection (preferred over onScrollGeometryChange)
.onScrollPhaseChange { oldPhase, newPhase, context in
    if newPhase == .interacting {
        lastOffset = context.geometry.contentOffset.y
    }
    if oldPhase == .interacting, newPhase != .animating {
        let scrolledUp = context.geometry.contentOffset.y - lastOffset < 0
        showHeader = scrolledUp
    }
}

// Visibility tracking — fires when visible scroll targets change
.onScrollTargetVisibilityChange(idType: Item.ID.self) { visibleIDs in
    // Called when the set of visible target items changes
}
```

## Full-Page Paging

`.scrollTargetLayout()` is required here — it tells the system which children to snap to.

```swift
ScrollView(.horizontal) {
    LazyHStack(spacing: 0) {
        ForEach(pages) { page in
            PageView(page: page)
                .containerRelativeFrame(.horizontal)
        }
    }
    .scrollTargetLayout()
}
.scrollTargetBehavior(.paging)
.scrollIndicators(.hidden)
```

## Snap to Item (Card Carousel)

`.scrollTargetLayout()` + `.viewAligned` — items snap to alignment. Use for carousels, NOT for feeds.

```swift
ScrollView(.horizontal) {
    LazyHStack(spacing: 16) {
        ForEach(cards) { card in
            CardView(card: card)
                .containerRelativeFrame(.horizontal) { width, _ in
                    width * 0.85
                }
        }
    }
    .scrollTargetLayout()
}
.scrollTargetBehavior(.viewAligned)
.contentMargins(.horizontal, 20)
.scrollIndicators(.hidden)
```

## scrollTransition — Scroll-Driven Effects (iOS 17+)

```swift
ScrollView {
    LazyVStack(spacing: 16) {
        ForEach(items) { item in
            ItemCard(item: item)
                .scrollTransition { content, phase in
                    content
                        .opacity(phase.isIdentity ? 1 : 0.3)
                        .scaleEffect(phase.isIdentity ? 1 : 0.9)
                        .blur(radius: phase.isIdentity ? 0 : 2)
                }
        }
    }
}

// phase.value: -1.0 (topLeading, entering) → 0.0 (identity, visible) → +1.0 (bottomTrailing, exiting)
.scrollTransition(.interactive) { content, phase in
    content.offset(x: phase.value * 20)
}
```

## Infinite Scroll / Pagination

```swift
struct InfiniteListView: View {
    @State private var items: [Item] = []
    @State private var page = 1
    @State private var hasMore = true
    @State private var isLoadingMore = false

    var body: some View {
        List(items) { item in
            ItemRow(item: item)
                .onAppear {
                    if item.id == items.last?.id { loadMore() }
                }
        }
        .overlay(alignment: .bottom) {
            if isLoadingMore { ProgressView().padding() }
        }
        .task { await loadInitial() }
    }

    private func loadMore() {
        guard hasMore, !isLoadingMore else { return }
        isLoadingMore = true
        Task {
            page += 1
            let newItems = (try? await ItemService.fetch(page: page)) ?? []
            hasMore = !newItems.isEmpty
            items.append(contentsOf: newItems)
            isLoadingMore = false
        }
    }
}
```

## visualEffect — Position-Based Effects (iOS 17+)

Use `.visualEffect` when you need access to the view's geometry for custom scroll-based effects:

```swift
ScrollView {
    LazyVStack(spacing: 20) {
        ForEach(items) { item in
            ItemCard(item: item)
                .visualEffect { content, geometry in
                    let frame = geometry.frame(in: .scrollView)
                    let distance = min(0, frame.minY)
                    return content
                        .opacity(1 + distance / 200)
                        .scaleEffect(1 + distance / 500)
                }
        }
    }
}
```

Use `.scrollTransition` for simple appear/disappear effects, `.visualEffect` for custom geometry-based effects.

## Scroll-Based Header & Scroll-to-Top (iOS 18+)

**NEVER store raw scroll offset in `@State`** — it updates every frame and causes constant re-renders, making scrolling janky.

### Direction-based collapsing header (preferred — onScrollPhaseChange)

`onScrollPhaseChange` with context is the **Apple-recommended pattern**. It fires only on phase transitions (not every frame), so it never causes scroll jank:

```swift
struct FeedView: View {
    @State private var showHeader = true
    @State private var showScrollToTop = false
    @State private var gestureStartOffset: CGFloat = 0

    var body: some View {
        VStack(spacing: 0) {
            if showHeader {
                HeaderView()
                    .transition(.move(edge: .top).combined(with: .opacity))
            }

            ScrollViewReader { proxy in
                ScrollView {
                    LazyVStack(spacing: 0) {
                        Color.clear.frame(height: 0).id("top")

                        ForEach(items) { item in
                            ItemView(item: item)
                        }
                    }
                }
                .onScrollPhaseChange { oldPhase, newPhase, context in
                    let offset = context.geometry.contentOffset.y

                    // Capture offset when user starts scrolling
                    if newPhase == .interacting {
                        gestureStartOffset = offset
                    }

                    // When gesture ends (not animating to snap target), check direction
                    if oldPhase == .interacting, newPhase != .animating {
                        let scrolledDown = offset - gestureStartOffset > 20
                        let scrolledUp = gestureStartOffset - offset > 20
                        if scrolledDown && showHeader && offset > 60 {
                            withAnimation(.easeOut(duration: 0.2)) { showHeader = false }
                        } else if scrolledUp && !showHeader {
                            withAnimation(.easeOut(duration: 0.2)) { showHeader = true }
                        }
                    }
                }
                .onScrollGeometryChange(for: Bool.self) { geo in
                    geo.contentOffset.y > 400
                } action: { _, pastThreshold in
                    if pastThreshold != showScrollToTop {
                        withAnimation(.spring(duration: 0.3)) { showScrollToTop = pastThreshold }
                    }
                }
                .overlay(alignment: .bottomTrailing) {
                    if showScrollToTop {
                        Button {
                            withAnimation(.spring(duration: 0.4, bounce: 0.2)) {
                                proxy.scrollTo("top", anchor: .top)
                                showHeader = true
                            }
                        } label: {
                            Image(systemName: "arrow.up")
                                .fontWeight(.semibold)
                                .foregroundStyle(.white)
                                .frame(width: 48, height: 48)
                                .background(.blue, in: Circle())
                                .shadow(radius: 8, y: 4)
                        }
                        .padding(20)
                        .transition(.scale.combined(with: .opacity))
                    }
                }
            }
        }
    }
}
```

### Simple threshold-based header (alternative)

For cases where you only need to hide when past a threshold (no direction detection):

```swift
ScrollView {
    content
}
.onScrollGeometryChange(for: Bool.self) { geometry in
    geometry.contentOffset.y < 50
} action: { _, isNearTop in
    if isNearTop != showHeader {
        withAnimation(.easeOut(duration: 0.2)) { showHeader = isNearTop }
    }
}
```

### Anti-patterns

| Anti-pattern | Fix |
|---|---|
| `.scrollPosition($position)` on a free-scrolling feed | Use `ScrollViewReader` for scroll-to-top — `.scrollPosition()` can interfere with normal scrolling |
| `@State private var lastOffset: CGFloat = 0` updated every frame | Use a plain class: `@State private var tracker = ScrollTracker()` |
| `withAnimation` inside every scroll callback | Only animate when crossing a threshold (`if new != old`) |
| `.scrollTargetLayout()` on a feed/list | Only use with `.scrollTargetBehavior(.paging/.viewAligned)` |

## Scroll Modifiers Reference

```swift
.scrollIndicators(.hidden)
.scrollDismissesKeyboard(.interactively)     // dismiss on drag
.scrollDismissesKeyboard(.immediately)
.contentMargins(.horizontal, 16)
.scrollContentBackground(.hidden)            // remove List background
```
