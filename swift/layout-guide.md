# SwiftUI Layout Guide

> .fill images MUST use Color+overlay pattern. Use `.contentMargins` for horizontal ScrollView spacing. Use ViewThatFits for adaptive layouts.

## ViewThatFits — Adaptive Layout (iOS 16+)

Automatically picks the first child that fits the available space.

```swift
ViewThatFits {
    HStack {
        Image(systemName: icon)
        Text(title)
        Spacer()
        Text(value)
    }

    VStack(alignment: .leading) {
        Label(title, systemImage: icon)
        Text(value)
            .frame(maxWidth: .infinity, alignment: .trailing)
    }
}
```

### Adaptive Button Layout

```swift
ViewThatFits {
    HStack(spacing: 12) {
        Button("Cancel") { cancel() }
            .buttonStyle(.bordered)
        Button("Save") { save() }
            .buttonStyle(.borderedProminent)
    }

    VStack(spacing: 8) {
        Button("Save") { save() }
            .buttonStyle(.borderedProminent)
        Button("Cancel") { cancel() }
            .buttonStyle(.bordered)
    }
}
```

## safeAreaInset — Floating Elements

```swift
List(items) { item in
    ItemRow(item: item)
}
.safeAreaInset(edge: .bottom) {
    Button("Add Item") { addItem() }
        .buttonStyle(.borderedProminent)
        .frame(maxWidth: .infinity)
        .padding()
        .background(.bar)
}
```

## @ScaledMetric — Accessibility-Aware Sizing

```swift
@ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24

Image(systemName: "star")
    .frame(width: iconSize, height: iconSize)
```

### Limit Dynamic Type Scaling

```swift
Text("Tab Label")
    .font(.caption2)
    .dynamicTypeSize(...DynamicTypeSize.xxxLarge)  // cap scaling
```

## LazyVGrid — Grid Layouts

**CRITICAL — .fill images break layout. `.frame(maxWidth: .infinity)` does NOT fix it.**

A `.fill` image reports its unconstrained render size back to the parent. If the image aspect ratio requires a wider render to fill the given height, the layout frame is wider than the column/screen. `.clipped()` clips visually but the layout is still wider. `.frame(maxWidth: .infinity)` does NOT constrain — it means "up to infinity", so it never clamps the image's wider size.

**The ONLY correct pattern:** use `Color` as a size anchor with the image in `.overlay`:

```swift
// ❌ BROKEN — image layout overflows, shifts all content sideways
AsyncImage(url: url) { phase in
    if let image = phase.image {
        image.resizable().aspectRatio(contentMode: .fill)
    } else { Color(.secondarySystemBackground) }
}
.frame(height: 200)
.clipped()

// ❌ STILL BROKEN — .frame(maxWidth: .infinity) does NOT constrain .fill image width
AsyncImage(url: url) { phase in
    if let image = phase.image {
        image.resizable().aspectRatio(contentMode: .fill)
    } else { Color(.secondarySystemBackground) }
}
.frame(maxWidth: .infinity)
.frame(height: 200)
.clipped()

// ✅ CORRECT — Color anchors layout, image in overlay, allowsHitTesting(false) prevents tap interception
Color(.secondarySystemBackground)
    .frame(height: 200)
    .overlay {
        AsyncImage(url: url) { phase in
            if let image = phase.image {
                image.resizable().aspectRatio(contentMode: .fill)
            }
        }
        .allowsHitTesting(false)
    }
    .clipShape(.rect(cornerRadius: 12))
```

**Why this works:** `Color` is a "greedy" view — it accepts the proposed width exactly and reports it back. The overlay constrains the image's layout participation to the Color's size. `.allowsHitTesting(false)` prevents the overflowing image from intercepting taps on nearby elements (filters, buttons above/below the card).

**For buttons/text on images**, chain `.overlay(alignment:)` AFTER `.clipShape`:

```swift
Color(.secondarySystemBackground)
    .frame(height: 150)
    .overlay {
        AsyncImage(url: url) { phase in
            if let image = phase.image {
                image.resizable().aspectRatio(contentMode: .fill)
            }
        }
        .allowsHitTesting(false)
    }
    .clipShape(.rect(cornerRadius: 12))
    .overlay(alignment: .topTrailing) {
        Button { } label: {
            Image(systemName: "heart.fill")
                .foregroundStyle(.white)
                .padding(6)
                .background(.black.opacity(0.4), in: Circle())
        }
        .padding(8)
    }
```

**NEVER use these patterns with .fill images:**
- `AsyncImage(...).frame(height: N)` — layout overflows
- `AsyncImage(...).frame(maxWidth: .infinity).frame(height: N)` — still overflows
- `ZStack { AsyncImage(...) Button(...) }` — ZStack adopts overflow width
- `AsyncImage(...).frame(maxWidth: .infinity, maxHeight: N).clipped()` — still overflows

**Exceptions where Color+overlay is NOT needed:**
- Horizontal scroll cards with FIXED `width` and `height` (e.g. `.frame(width: 300, height: 400).clipped()`) — both dimensions are explicit.
- Full-screen interactive images (zoomable viewers, photo editors) — use `Image` directly with gestures. Color+overlay+`.allowsHitTesting(false)` disables pinch/zoom/drag.

**Gestures on Color+overlay cards** (e.g. double-tap to like):

```swift
Color(.secondarySystemBackground)
    .frame(height: 200)
    .overlay {
        AsyncImage(url: url) { phase in
            if let image = phase.image {
                image.resizable().aspectRatio(contentMode: .fill)
            }
        }
        .allowsHitTesting(false)
    }
    .clipShape(.rect(cornerRadius: 12))
    .onTapGesture(count: 2) { like() }  // tap goes to Color, works correctly
```

Always put `.onTapGesture` AFTER `.clipShape`, not on the image overlay.

### Adaptive Columns (scales across device sizes)

```swift
let columns = [GridItem(.adaptive(minimum: 100, maximum: 200))]

LazyVGrid(columns: columns, spacing: 8) {
    ForEach(items) { item in
        ItemCard(item: item)
    }
}
.padding(.horizontal)
```

### Fixed Column Count

```swift
let columns = Array(repeating: GridItem(.flexible(), spacing: 4), count: 3)

LazyVGrid(columns: columns, spacing: 4) {
    ForEach(photos) { photo in
        PhotoCell(url: photo.url)
    }
}
.padding(.horizontal)
```

### Hero Card with Photo Overlay

```swift
Color(white: 0.15)
    .frame(height: 250)
    .overlay {
        AsyncImage(url: destination.imageURL) { phase in
            if let image = phase.image {
                image.resizable().aspectRatio(contentMode: .fill)
            }
        }
        .allowsHitTesting(false)
    }
    .clipShape(.rect(cornerRadius: 16))
    .overlay(alignment: .bottomLeading) {
        LinearGradient(
            stops: [.init(color: .clear, location: 0.4), .init(color: .black.opacity(0.7), location: 1.0)],
            startPoint: .top, endPoint: .bottom
        )
        .clipShape(.rect(cornerRadius: 16))
    }
    .overlay(alignment: .bottomLeading) {
        VStack(alignment: .leading) {
            Text(destination.name).font(.title2.bold())
            Text(destination.country).font(.subheadline)
        }
        .foregroundStyle(.white)
        .padding()
    }
```

### Card Grid with Images (2-column)

The most common layout. Uses Color as a size anchor so the .fill image never overflows the grid column.

```swift
let columns = [GridItem(.flexible(), spacing: 12), GridItem(.flexible(), spacing: 12)]

ScrollView {
    LazyVGrid(columns: columns, spacing: 12) {
        ForEach(destinations) { destination in
            VStack(alignment: .leading, spacing: 8) {
                Color(.secondarySystemBackground)
                    .frame(height: 120)
                    .overlay {
                        AsyncImage(url: destination.imageURL) { phase in
                            if let image = phase.image {
                                image.resizable().aspectRatio(contentMode: .fill)
                            }
                        }
                        .allowsHitTesting(false)
                    }
                    .clipShape(.rect(cornerRadius: 12))
                    .overlay(alignment: .topTrailing) {
                        Button { toggleFavorite(destination) } label: {
                            Image(systemName: destination.isFavorite ? "heart.fill" : "heart")
                                .font(.subheadline)
                                .foregroundStyle(.white)
                                .padding(6)
                                .background(.black.opacity(0.4), in: Circle())
                        }
                        .padding(6)
                    }

                Text(destination.name).font(.headline)
                Text(destination.subtitle)
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }
        }
    }
    .padding(.horizontal)
}
```

### Horizontal ScrollView Padding — NEVER use `.padding(.horizontal)` inside

**`.padding(.horizontal)` on inner content creates permanent visible padding strips at scroll edges.** Use `.contentMargins(.horizontal, N)` or `.safeAreaPadding(.horizontal, N)` on the ScrollView instead — content starts inset but scrolls edge-to-edge.

```swift
// ❌ WRONG — permanent padding strips visible at edges
ScrollView(.horizontal) {
    HStack(spacing: 12) {
        ForEach(items) { item in ItemCard(item: item) }
    }
    .padding(.horizontal)
}

// ✅ CORRECT — content scrolls edge-to-edge, starts with inset
ScrollView(.horizontal) {
    HStack(spacing: 12) {
        ForEach(items) { item in ItemCard(item: item) }
    }
}
.contentMargins(.horizontal, 16)
```

### LazyHGrid — Horizontal Scrolling Grid

```swift
let rows = [GridItem(.fixed(120))]

ScrollView(.horizontal) {
    LazyHGrid(rows: rows, spacing: 12) {
        ForEach(categories) { category in
            CategoryCard(category: category)
        }
    }
}
.contentMargins(.horizontal, 16)
.scrollIndicators(.hidden)
```

For horizontal image galleries, use fixed-size cells with `.fill` + `.clipped()`:

```swift
ScrollView(.horizontal) {
    HStack(spacing: 8) {
        ForEach(photos) { photo in
            AsyncImage(url: photo.url) { phase in
                if let image = phase.image {
                    image.resizable().aspectRatio(contentMode: .fill)
                } else {
                    Color(.secondarySystemBackground)
                }
            }
            .frame(width: 200, height: 150)
            .clipped()
            .clipShape(.rect(cornerRadius: 8))
        }
    }
}
.contentMargins(.horizontal, 16)
.scrollIndicators(.hidden)
```

### Tag / Chip Layout

For tags, suggestions, or filter chips — use a horizontal `ScrollView` (most reliable):

```swift
ScrollView(.horizontal) {
    HStack(spacing: 8) {
        ForEach(tags, id: \.self) { tag in
            Button(tag) { select(tag) }
                .font(.subheadline)
                .padding(.horizontal, 12)
                .padding(.vertical, 6)
                .background(.thinMaterial)
                .clipShape(Capsule())
        }
    }
}
.contentMargins(.horizontal, 16)
.scrollIndicators(.hidden)
```

For wrapping tags (flow layout), use `LazyVGrid` with `.adaptive`. Set `minimum` to the expected chip width:

```swift
LazyVGrid(columns: [GridItem(.adaptive(minimum: 80), spacing: 8)], alignment: .leading, spacing: 8) {
    ForEach(tags, id: \.self) { tag in
        Text(tag)
            .font(.subheadline)
            .padding(.horizontal, 12)
            .padding(.vertical, 6)
            .background(.thinMaterial)
            .clipShape(Capsule())
    }
}
```

### GridItem Types

| Type | Behavior |
|------|----------|
| `.adaptive(minimum:maximum:)` | Auto-fits as many columns as possible within range |
| `.flexible(minimum:maximum:)` | Flexible width, one item per GridItem |
| `.fixed(CGFloat)` | Exact pixel width |

Use `.adaptive` for tag wrapping, icon pickers, settings grids. Use `.flexible` with fixed count for photo grids. Use `.fixed` for uniform-width columns.

## Detail Page with Hero Image

The most common layout that breaks. Use Color + overlay for the hero image.

```swift
ScrollView {
    VStack(alignment: .leading, spacing: 0) {
        Color(.secondarySystemBackground)
            .frame(height: 300)
            .overlay {
                AsyncImage(url: item.imageURL) { phase in
                    if let image = phase.image {
                        image.resizable().aspectRatio(contentMode: .fill)
                    }
                }
                .allowsHitTesting(false)
            }
            .clipped()

        VStack(alignment: .leading, spacing: 16) {
            Text(item.title)
                .font(.title.bold())

            Text(item.subtitle)
                .font(.subheadline)
                .foregroundStyle(.secondary)

            Text(item.description)
                .font(.body)
        }
        .padding()
    }
}
```

With overlay text on the hero:
```swift
Color(.secondarySystemBackground)
    .frame(height: 350)
    .overlay {
        AsyncImage(url: item.imageURL) { phase in
            if let image = phase.image {
                image.resizable().aspectRatio(contentMode: .fill)
            }
        }
        .allowsHitTesting(false)
    }
    .clipped()
    .overlay(alignment: .bottomLeading) {
        LinearGradient(
            stops: [.init(color: .clear, location: 0.4), .init(color: .black.opacity(0.7), location: 1.0)],
            startPoint: .top, endPoint: .bottom
        )
    }
    .overlay(alignment: .bottomLeading) {
        VStack(alignment: .leading, spacing: 4) {
            Text(item.title).font(.title.bold())
            Text(item.subtitle).font(.subheadline)
        }
        .foregroundStyle(.white)
        .padding()
    }
```

## Context-Agnostic Views

Views should work in any context — full screen, sheet, popover, or embedded. Never assume screen size.

```swift
// ❌ Assumes full screen
Image("banner")
    .frame(width: UIScreen.main.bounds.width)

// ✅ Adapts to given space — Color anchors layout, image fills in overlay
Color(.secondarySystemBackground)
    .frame(height: 200)
    .overlay {
        Image("banner")
            .resizable()
            .aspectRatio(contentMode: .fill)
            .allowsHitTesting(false)
    }
    .clipped()
```

## Own Your Container

Custom views should own static containers (HStack, VStack) but callers own lazy/repeatable ones:

```swift
// ✅ View owns its layout container
struct HeaderView: View {
    var body: some View {
        HStack {
            Image(systemName: "star")
            Text("Title")
            Spacer()
        }
    }
}

// ❌ Caller has to wrap — fragile
struct HeaderView: View {
    var body: some View {
        Image(systemName: "star")
        Text("Title")
    }
}
```

## Measuring Views — onGeometryChange (iOS 16+)

Modern replacement for GeometryReader when you only need to measure size:

```swift
@State private var textHeight: CGFloat = 0

Text("Measured text")
    .onGeometryChange(for: CGFloat.self) { geometry in
        geometry.size.height
    } action: { newValue in
        textHeight = newValue
    }
```

## containerRelativeFrame — Size Relative to Container (iOS 17+)

Modern replacement for GeometryReader when you need to size a view relative to its nearest scrollable container (ScrollView, List) or the window.

```swift
// Half-width card
CardView()
    .containerRelativeFrame(.horizontal) { width, _ in
        width * 0.5
    }

// Full-width banner with fixed aspect ratio
Image("banner")
    .resizable()
    .aspectRatio(2, contentMode: .fit)
    .containerRelativeFrame(.horizontal)
```

### Paged Carousel — count/span/spacing

Divide container into equal columns. `count` = total divisions, `span` = how many the view occupies.

```swift
ScrollView(.horizontal) {
    LazyHStack(spacing: 10) {
        ForEach(items) { item in
            CardView(item: item)
                .containerRelativeFrame(
                    .horizontal, count: 4, span: 3, spacing: 10)
        }
    }
}
.safeAreaPadding(.horizontal, 20)
```

Size formula: `itemWidth = ((containerWidth - spacing * (count - 1)) / count) * span + (span - 1) * spacing`

## Parallax Header

Use GeometryReader only when containerRelativeFrame can't do what you need (e.g., reading exact position).

```swift
ScrollView {
    GeometryReader { geo in
        let offset = geo.frame(in: .global).minY
        Color.clear
            .frame(height: max(250 + offset, 250))
            .overlay {
                Image("header")
                    .resizable()
                    .aspectRatio(contentMode: .fill)
                    .allowsHitTesting(false)
            }
            .clipped()
            .offset(y: offset > 0 ? -offset : 0)
    }
    .frame(height: 250)

    LazyVStack {
        ForEach(items) { item in
            ItemRow(item: item)
        }
    }
}
```
