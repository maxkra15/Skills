# SwiftUI Gestures Reference

> Use GestureState for temporary state during gestures. Use gesture composition for multi-touch. Always provide accessibility alternatives.

## Gesture Types

### TapGesture

```swift
// Single tap — prefer Button for standard actions
Text("Tap me")
    .onTapGesture { handleTap() }

// Double tap
Image("photo")
    .onTapGesture(count: 2) { toggleZoom() }

// Tap with location
content.onTapGesture { location in
    addAnnotation(at: location)
}
```

### SpatialTapGesture (iOS 16+)

Returns tap location as part of the gesture value:

```swift
content.gesture(
    SpatialTapGesture()
        .onEnded { value in
            let point = value.location    // CGPoint in local coordinates
            addPin(at: point)
        }
)

// Double spatial tap
SpatialTapGesture(count: 2)
    .onEnded { value in
        zoomIn(at: value.location)
    }
```

### DragGesture

```swift
@GestureState private var dragOffset = CGSize.zero
@State private var position = CGSize.zero

Circle()
    .frame(width: 60, height: 60)
    .offset(x: position.width + dragOffset.width,
            y: position.height + dragOffset.height)
    .gesture(
        DragGesture()
            .updating($dragOffset) { value, state, _ in
                state = value.translation
            }
            .onEnded { value in
                position.width += value.translation.width
                position.height += value.translation.height
            }
    )
```

### LongPressGesture

```swift
@GestureState private var isPressed = false

Circle()
    .fill(isPressed ? .red : .blue)
    .scaleEffect(isPressed ? 1.2 : 1.0)
    .gesture(
        LongPressGesture(minimumDuration: 0.5)
            .updating($isPressed) { value, state, _ in
                state = value
            }
            .onEnded { _ in
                showContextMenu()
            }
    )
    .animation(.spring, value: isPressed)
```

### MagnifyGesture (Pinch to Zoom)

iOS 17+ replaces `MagnificationGesture` with `MagnifyGesture`:

```swift
@State private var scale: CGFloat = 1.0
@GestureState private var magnification: CGFloat = 1.0

Image("photo")
    .scaleEffect(scale * magnification)
    .gesture(
        MagnifyGesture()
            .updating($magnification) { value, state, _ in
                state = value.magnification
            }
            .onEnded { value in
                scale *= value.magnification
                scale = min(max(scale, 0.5), 4.0)    // Clamp
            }
    )
```

`MagnifyGesture.Value` properties: `.magnification` (scale factor), `.velocity` (rate of change), `.startAnchor` (initial pinch center as UnitPoint), `.startLocation` (initial pinch center as CGPoint).

### RotateGesture

iOS 17+ replaces `RotationGesture` with `RotateGesture`:

```swift
@State private var angle: Angle = .zero
@GestureState private var rotation: Angle = .zero

Image("dial")
    .rotationEffect(angle + rotation)
    .gesture(
        RotateGesture()
            .updating($rotation) { value, state, _ in
                state = value.rotation
            }
            .onEnded { value in
                angle += value.rotation
            }
    )
```

`RotateGesture.Value` properties: `.rotation` (angle), `.velocity` (angular velocity).

## GestureState vs State

| | `@GestureState` | `@State` |
|---|---|---|
| Resets when gesture ends | Yes (automatic) | No |
| Use for | Temporary visual feedback during gesture | Committed final values |
| Updated via | `.updating()` | `.onChanged()` / `.onEnded()` |

```swift
// ✅ GestureState for live feedback, State for final position
@GestureState private var dragOffset = CGSize.zero    // Resets on release
@State private var position = CGSize.zero              // Persists after release
```

## Gesture Composition

### Simultaneously — Multiple gestures at once

```swift
// Drag + Pinch (photo viewer)
Image("photo")
    .offset(x: dragOffset.width, y: dragOffset.height)
    .scaleEffect(scale * magnification)
    .gesture(
        DragGesture()
            .updating($dragOffset) { value, state, _ in
                state = value.translation
            }
            .simultaneously(with:
                MagnifyGesture()
                    .updating($magnification) { value, state, _ in
                        state = value.magnification
                    }
            )
    )
```

### Sequenced — One after another

```swift
// Long press then drag (drag-to-reorder)
@GestureState private var dragState = DragState.inactive

enum DragState {
    case inactive, pressing, dragging(translation: CGSize)
}

let longPressDrag = LongPressGesture(minimumDuration: 0.3)
    .sequenced(before: DragGesture())
    .updating($dragState) { value, state, _ in
        switch value {
        case .first(true):
            state = .pressing
        case .second(true, let drag):
            state = .dragging(translation: drag?.translation ?? .zero)
        default:
            state = .inactive
        }
    }
```

### Exclusively — Only one gesture wins

```swift
// Double-tap to zoom OR single-tap to select
let doubleTap = TapGesture(count: 2).onEnded { toggleZoom() }
let singleTap = TapGesture(count: 1).onEnded { select() }

content.gesture(
    doubleTap.exclusively(before: singleTap)
)
```

## Gesture Priority

```swift
// Default: child gesture takes priority
// Use .highPriorityGesture to override
ScrollView {
    content
}
.highPriorityGesture(
    DragGesture().onChanged { _ in customDrag() }
)

// Use .simultaneousGesture to not block parent/child
content.simultaneousGesture(
    TapGesture().onEnded { track() }
)
```

## ScrollView Conflicts

```swift
// ❌ DragGesture blocks ScrollView scrolling
ScrollView {
    content.gesture(DragGesture())
}

// ✅ Use minimumDistance to let scroll happen first
ScrollView {
    content.gesture(
        DragGesture(minimumDistance: 30)    // Requires 30pt before activating
    )
}

// ✅ Or use simultaneousGesture
ScrollView {
    content.simultaneousGesture(
        LongPressGesture().onEnded { _ in haptic() }
    )
}

// ✅ Empty onTapGesture unblocks scroll when using LongPress + Drag
ScrollView {
    content
        .onTapGesture { }    // Unblocks scrolling
        .gesture(
            LongPressGesture(minimumDuration: 0.3)
                .sequenced(before: DragGesture())
        )
}
```

## Velocity

DragGesture provides `velocity` directly on the value (available since iOS 13):

```swift
DragGesture()
    .onEnded { value in
        // value.velocity is CGSize (points per second)
        let velocity = value.velocity

        // Swipe detection
        if velocity.width > 500 {
            swipeRight()
        } else if velocity.width < -500 {
            swipeLeft()
        }

        // Use predictedEndLocation for momentum-based animations
        let predictedEnd = value.predictedEndLocation
    }
```

## Coordinate Spaces

```swift
// Local to the view (default)
DragGesture(coordinateSpace: .local)

// Global screen coordinates
DragGesture(coordinateSpace: .global)

// Named coordinate space (iOS 17+: use .named())
ZStack {
    content.gesture(
        DragGesture(coordinateSpace: .named("canvas"))
    )
}
.coordinateSpace(.named("canvas"))

// ScrollView coordinate space (iOS 17+)
ScrollView {
    content.gesture(
        DragGesture(coordinateSpace: .scrollView)
    )
}
```

## Accessibility for Custom Gestures

Always provide VoiceOver alternatives for gesture-driven interactions:

```swift
Image("slider-track")
    .gesture(
        DragGesture()
            .onChanged { value in
                updateValue(from: value.location.x)
            }
    )
    .accessibilityElement()
    .accessibilityLabel("Volume")
    .accessibilityValue("\(Int(volume))%")
    .accessibilityAdjustableAction { direction in
        switch direction {
        case .increment: volume = min(100, volume + 5)
        case .decrement: volume = max(0, volume - 5)
        @unknown default: break
        }
    }
```

## Common Patterns

### Swipe-to-Dismiss

```swift
@Environment(\.dismiss) private var dismiss
@GestureState private var dragOffset: CGFloat = 0

content
    .offset(y: max(dragOffset, 0))
    .gesture(
        DragGesture()
            .updating($dragOffset) { value, state, _ in
                state = value.translation.height
            }
            .onEnded { value in
                if value.translation.height > 150 {
                    dismiss()
                }
            }
    )
    .animation(.spring, value: dragOffset)
```

### Zoomable Image (Pinch + Drag + Double-Tap)

```swift
@State private var scale: CGFloat = 1.0
@State private var offset = CGSize.zero
@GestureState private var gestureScale: CGFloat = 1.0

Image("photo")
    .scaleEffect(scale * gestureScale)
    .offset(offset)
    .gesture(
        MagnifyGesture()
            .updating($gestureScale) { value, state, _ in
                state = value.magnification
            }
            .onEnded { value in
                scale = min(max(scale * value.magnification, 1.0), 5.0)
                if scale <= 1.0 { offset = .zero }
            }
            .simultaneously(with:
                DragGesture()
                    .onChanged { value in
                        guard scale > 1 else { return }
                        offset = value.translation
                    }
            )
    )
    .onTapGesture(count: 2) {
        withAnimation(.spring) {
            if scale > 1 {
                scale = 1.0
                offset = .zero
            } else {
                scale = 2.5
            }
        }
    }
```

## Hit Testing & Content Shape

### .contentShape — Define Tap Area

Use `.contentShape(Rectangle())` to make the entire frame tappable, even transparent areas:

```swift
HStack {
    Image(systemName: "star")
    Text("Tap anywhere in this row")
    Spacer()
}
.contentShape(Rectangle())
.onTapGesture { handleTap() }
```

### .allowsHitTesting — Disable Touch on Decorative Views

`.fill` images in overlays render larger than their frame. Even with `.clipShape()`, the overflowing image can intercept taps on nearby elements. Always disable hit testing on decorative image overlays:

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
```

### Button Inside NavigationLink

A Button inside a NavigationLink can cause both to trigger. SwiftUI gives the inner Button priority, but `.buttonStyle(.plain)` on the NavigationLink can interfere. Keep `.buttonStyle(.plain)` on the NavigationLink and place the Button in a separate `.overlay` AFTER `.clipShape`:

```swift
NavigationLink(value: item) {
    CardContent(item: item)
}
.buttonStyle(.plain)

// Inside CardContent — heart button as overlay AFTER clipShape:
Color(...)
    .frame(height: 150)
    .overlay { image.allowsHitTesting(false) }
    .clipShape(.rect(cornerRadius: 12))
    .overlay(alignment: .topTrailing) {
        Button { toggleFavorite() } label: { ... }
            .padding(8)
    }
```

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| `.onTapGesture` for buttons | Use `Button` — gets accessibility for free |
| Heavy computation in `.onChanged` | Throttle or defer to `.onEnded` |
| `@State` for drag offset during gesture | Use `@GestureState` — auto-resets on release |
| DragGesture with minimumDistance: 0 in ScrollView | Use minimumDistance ≥ 10 to not block scrolling |
| Computing velocity from predictedEndTranslation | Use `value.velocity` directly (available since iOS 13) |
| Missing accessibility for custom gestures | Add `.accessibilityAdjustableAction` or `.accessibilityAction` |
| `.fill` image overlay without `.allowsHitTesting(false)` | Overflowing image intercepts taps on nearby elements |
