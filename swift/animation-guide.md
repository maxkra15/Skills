# SwiftUI Animation Guide

> Springs are the default for iOS. Use PhaseAnimator for multi-step sequences. Use KeyframeAnimator for per-property keyframes.

## Spring Presets

```swift
.animation(.spring, value: x)                                     // system default
.animation(.snappy, value: x)                                      // quick, slight bounce
.animation(.bouncy, value: x)                                      // more bounce
.animation(.smooth, value: x)                                      // no bounce, smooth stop
.animation(.spring(response: 0.3, dampingFraction: 0.6), value: x) // custom spring
.animation(.spring(duration: 0.5, bounce: 0.3), value: x)          // duration-based
```

## Animation Placement

Place `.animation` **after** the properties it should animate. Animation modifiers only affect properties declared above them.

```swift
// ✅ Animation after properties — animates both
Rectangle()
    .frame(width: isExpanded ? 200 : 100, height: 50)
    .foregroundStyle(isExpanded ? .blue : .red)
    .animation(.default, value: isExpanded)

// ❌ Animation before properties — nothing animated
Rectangle()
    .animation(.default, value: isExpanded)
    .frame(width: isExpanded ? 200 : 100, height: 50)
```

### Selective Animation

Use `animation(nil, value:)` to block specific properties from animating:

```swift
Rectangle()
    .frame(width: isExpanded ? 200 : 100, height: 50)
    .animation(.spring, value: isExpanded)       // animate size
    .foregroundStyle(isExpanded ? .blue : .red)
    .animation(nil, value: isExpanded)            // don't animate color
```

## Timing Curves — When to Use

| Curve | Use Case |
|-------|----------|
| `.spring` | Interactive elements, most UI (default) |
| `.snappy` | Quick actions, slight bounce |
| `.bouncy` | Playful feedback |
| `.smooth` | Smooth stop, no bounce |
| `.easeInOut` | Appearance transitions |
| `.linear` | Progress indicators only — never for UI (feels robotic) |

## Animation Modifiers

```swift
.animation(.spring.delay(0.1), value: x)
.animation(.spring.speed(2), value: x)
.animation(.easeInOut.repeatForever(autoreverses: true), value: x)
.animation(.easeInOut.repeatCount(3), value: x)
```

## Animation Scope — Keep It Narrow

Apply `.animation` to specific subviews, not at root level:

```swift
// ✅ Only animates the expandable content
VStack {
    HeaderView()
    ExpandableContent(isExpanded: isExpanded)
        .animation(.spring, value: isExpanded)
    FooterView()
}

// ❌ Animates everything including header and footer
VStack {
    HeaderView()
    ExpandableContent(isExpanded: isExpanded)
    FooterView()
}
.animation(.spring, value: isExpanded)
```

## Animation Performance

Prefer GPU-accelerated transforms over layout changes:

```swift
// ✅ Fast — scaleEffect, offset, rotationEffect, opacity
.scaleEffect(isActive ? 1.5 : 1.0)
.offset(x: isActive ? 50 : 0)

// ❌ Slow — triggers layout recalculation every frame
.frame(width: isActive ? 150 : 100)
.padding(isActive ? 50 : 0)
```

## PhaseAnimator — Multi-Step Sequences (iOS 17+)

Cycles through phases automatically. Each phase gets its own animation curve.

```swift
enum AnimationPhase: CaseIterable {
    case initial, scaled, rotated, done

    var scale: Double {
        switch self {
        case .initial: 1.0
        case .scaled: 1.5
        case .rotated: 1.2
        case .done: 1.0
        }
    }

    var rotation: Double {
        switch self {
        case .initial, .scaled: 0
        case .rotated, .done: 360
        }
    }
}

PhaseAnimator(AnimationPhase.allCases) { phase in
    Image(systemName: "star.fill")
        .font(.largeTitle)
        .foregroundStyle(.yellow)
        .scaleEffect(phase.scale)
        .rotationEffect(.degrees(phase.rotation))
} animation: { phase in
    switch phase {
    case .initial: .snappy
    case .scaled: .spring(response: 0.3, dampingFraction: 0.5)
    case .rotated: .easeInOut(duration: 0.5)
    case .done: .smooth
    }
}
```

### Trigger-Based PhaseAnimator

```swift
Image(systemName: "checkmark.circle.fill")
    .font(.system(size: 60))
    .foregroundStyle(.green)
    .phaseAnimator([false, true], trigger: triggerCount) { content, phase in
        content
            .scaleEffect(phase ? 1.3 : 1.0)
            .rotationEffect(.degrees(phase ? 10 : 0))
    } animation: { _ in
        .spring(response: 0.2, dampingFraction: 0.3)
    }
```

## KeyframeAnimator — Per-Property Keyframes (iOS 17+)

Animate different properties independently with precise timing.

```swift
struct AnimationValues {
    var verticalOffset: Double = 0
    var verticalStretch: Double = 1.0
}

Circle()
    .fill(.orange)
    .frame(width: 50, height: 50)
    .keyframeAnimator(initialValue: AnimationValues(), trigger: trigger) { content, value in
        content
            .offset(y: value.verticalOffset)
            .scaleEffect(y: value.verticalStretch)
    } keyframes: { _ in
        KeyframeTrack(\.verticalOffset) {
            SpringKeyframe(-100, duration: 0.3)
            SpringKeyframe(0, duration: 0.3, spring: .bouncy)
        }
        KeyframeTrack(\.verticalStretch) {
            LinearKeyframe(1.2, duration: 0.1)
            SpringKeyframe(0.8, duration: 0.15)
            SpringKeyframe(1.0, duration: 0.2)
        }
    }
```

## SF Symbol Effects (iOS 17+)

Animate SF Symbols with built-in effects. These are purpose-built for icons and always look polished.

### Continuous Effects (auto-repeat)

```swift
// Pulse — fades opacity up and down
Image(systemName: "heart.fill")
    .symbolEffect(.pulse)

// Variable color — fills layers sequentially (great for signal bars, wifi)
Image(systemName: "wifi")
    .symbolEffect(.variableColor.iterative)

// Breathing — scales gently (iOS 18+)
Image(systemName: "circle.fill")
    .symbolEffect(.breathe)
```

### Triggered Effects (fire once on value change)

```swift
// Bounce on tap
Image(systemName: "star.fill")
    .symbolEffect(.bounce, value: tapCount)

// Wiggle for attention (iOS 18+)
Image(systemName: "bell.fill")
    .symbolEffect(.wiggle, value: notificationCount)

// Scale for emphasis
Image(systemName: "heart.fill")
    .symbolEffect(.scale.up, isActive: isLiked)
```

### Symbol Replacement (animated icon swap)

```swift
// Smooth animated transition between two symbols
Image(systemName: isPlaying ? "pause.fill" : "play.fill")
    .contentTransition(.symbolEffect(.replace))

// With custom direction
Image(systemName: isMuted ? "speaker.slash.fill" : "speaker.wave.2.fill")
    .contentTransition(.symbolEffect(.replace.downUp))
```

### Conditional Effect (on/off toggle)

```swift
// Active while isActive is true, stops when false
Image(systemName: "mic.fill")
    .symbolEffect(.variableColor.iterative.reversing, isActive: isRecording)
```

## Recipes

### Staggered List Animation

```swift
ForEach(Array(items.enumerated()), id: \.element.id) { index, item in
    ItemRow(item: item)
        .opacity(appeared ? 1 : 0)
        .offset(y: appeared ? 0 : 20)
        .animation(.spring(response: 0.4).delay(Double(index) * 0.05), value: appeared)
}
.onAppear { appeared = true }
```

### Shimmer / Loading Skeleton

```swift
struct ShimmerModifier: ViewModifier {
    @State private var phase: CGFloat = 0

    func body(content: Content) -> some View {
        content
            .overlay(
                LinearGradient(
                    colors: [.clear, .white.opacity(0.3), .clear],
                    startPoint: .leading,
                    endPoint: .trailing
                )
                .offset(x: phase)
                .mask { content }
            )
            .onAppear {
                withAnimation(.linear(duration: 1.5).repeatForever(autoreverses: false)) {
                    phase = 300
                }
            }
    }
}
```

### Matched Geometry (Shared Element Transition)

```swift
@Namespace private var animation
@State private var isExpanded = false

if isExpanded {
    ExpandedCard()
        .matchedGeometryEffect(id: "card", in: animation)
        .onTapGesture {
            withAnimation(.spring(response: 0.4)) { isExpanded = false }
        }
} else {
    CompactCard()
        .matchedGeometryEffect(id: "card", in: animation)
        .onTapGesture {
            withAnimation(.spring(response: 0.4)) { isExpanded = true }
        }
}
```

## Transitions — Animate View Insertion/Removal

Transitions animate views being added or removed from the tree. **Animation context must be outside the conditional.**

```swift
// ✅ Animation outside conditional
VStack {
    if showDetail {
        DetailView()
            .transition(.scale.combined(with: .opacity))
    }
}
.animation(.spring, value: showDetail)

// ✅ Explicit animation on state change
Button("Toggle") {
    withAnimation(.spring) { showDetail.toggle() }
}

// ❌ Animation inside conditional — won't work on removal!
if showDetail {
    DetailView()
        .transition(.slide)
        .animation(.spring, value: showDetail)
}
```

### Built-in Transitions

| Transition | Effect |
|------------|--------|
| `.opacity` | Fade in/out |
| `.scale` | Scale up/down |
| `.slide` | Slide from leading edge |
| `.move(edge:)` | Move from specific edge |

### Asymmetric Transitions

```swift
.transition(
    .asymmetric(
        insertion: .scale.combined(with: .opacity),
        removal: .move(edge: .bottom).combined(with: .opacity)
    )
)
```

### Custom Transition (iOS 17+)

```swift
struct BlurTransition: Transition {
    var radius: CGFloat

    func body(content: Content, phase: TransitionPhase) -> some View {
        content
            .blur(radius: phase.isIdentity ? 0 : radius)
            .opacity(phase.isIdentity ? 1 : 0)
    }
}

.transition(BlurTransition(radius: 10))
```

## Disabling Animations

```swift
// Remove animation from a specific view
Text("Count: \(count)")
    .transaction { $0.animation = nil }

// Prevent parent animations from propagating
DataView()
    .transaction { $0.disablesAnimations = true }
```

## Animation Completion (iOS 17+)

```swift
Button("Animate") {
    withAnimation(.spring) {
        isExpanded.toggle()
    } completion: {
        showNextStep = true
    }
}
```

## Animatable Protocol — Custom Property Interpolation

Implement `animatableData` for custom animated properties. **Without it, values jump to final state with no interpolation.**

```swift
struct ShakeModifier: ViewModifier, Animatable {
    var shakeCount: Double

    var animatableData: Double {
        get { shakeCount }
        set { shakeCount = newValue }
    }

    func body(content: Content) -> some View {
        content.offset(x: sin(shakeCount * .pi * 2) * 10)
    }
}
```

For multiple animated properties, use `AnimatablePair<CGFloat, Double>`.
