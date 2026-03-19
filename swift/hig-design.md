# Apple HIG & iOS Design Reference

> Design native iOS apps following Apple Human Interface Guidelines. Use semantic colors, system fonts, SF Symbols, and native capabilities.

## Core HIG Principles

- **Clarity** — Text legible at every size, icons precise, adornments subtle
- **Deference** — Content is king, UI doesn't compete with it
- **Depth** — Layers and motion provide hierarchy and context
- **Consistency** — Standard UI elements and patterns familiar to iOS users

## Layout & Spacing

- Touch targets: minimum **44×44pt**
- Standard horizontal margins: **16pt** on compact-width iPhones, **20pt** on larger iPhones and iPad
- Use an **8pt grid** for spacing: 4, 8, 12, 16, 24, 32, 40
- Use `safeAreaInset` for floating elements over scrollable content
- Use `.ignoresSafeArea()` for full-bleed backgrounds only
- Group related elements with 8-12pt spacing, separate sections with 24-32pt

### Common Spacing Patterns

```swift
// Section spacing
VStack(spacing: 24) { ... }

// Element spacing within a section
VStack(spacing: 8) { ... }

// Standard card padding
.padding(16)

// Compact element padding
.padding(.horizontal, 12)
.padding(.vertical, 8)
```

## Typography

### Dynamic Type Sizes (Default "Large" category)

| SwiftUI Style | Weight | Size (pt) | Leading (pt) | Use For |
|---------------|--------|-----------|--------------|---------|
| `.largeTitle` | Regular | 34 | 41 | Screen titles |
| `.title` | Regular | 28 | 34 | Section headers |
| `.title2` | Regular | 22 | 28 | Subsection headers |
| `.title3` | Regular | 20 | 24 | Card titles |
| `.headline` | **Semibold** | 17 | 22 | Emphasized body text |
| `.body` | Regular | 17 | 22 | Primary content |
| `.callout` | Regular | 16 | 21 | Secondary content |
| `.subheadline` | Regular | 15 | 20 | Supporting text |
| `.footnote` | Regular | 13 | 18 | Timestamps, metadata |
| `.caption` | Regular | 12 | 16 | Labels, annotations |
| `.caption2` | Regular | 11 | 13 | Fine print |

### Typography Rules

- The system font is **SF Pro** — a single variable font with built-in optical sizing. It is built into iOS — never import or bundle it manually. `.font(.title)`, `.font(.body)`, etc. already use SF Pro.
- ALWAYS use system text styles — never hardcode font sizes
- Support Dynamic Type — 7 default sizes + 5 accessibility sizes
- Use `@ScaledMetric` for custom values that scale with text size — see `layout-guide.md` for examples
- Limit scaling for compact UI: `.dynamicTypeSize(...DynamicTypeSize.xxxLarge)`
- One prominent title per screen, supporting text progressively smaller
- Use `.fontWeight()` or `.bold()` for emphasis, not larger font sizes

```swift
// Correct: system styles
Text("Title").font(.title.bold())
Text("Body").font(.body)
Text("Note").font(.caption).foregroundStyle(.secondary)
```

### System Font Design Variants

Use font design variants to give apps a distinct personality without bundling custom fonts. These all support Dynamic Type and accessibility automatically.

```swift
// Design variants — pick one that fits the app's character
.font(.system(.title, design: .default))    // SF Pro — clean, neutral
.font(.system(.title, design: .rounded))    // SF Rounded — friendly, approachable
.font(.system(.title, design: .serif))      // New York — editorial, elegant
.font(.system(.title, design: .monospaced)) // SF Mono — technical, code-like

// Width variants (iOS 16+) — for visual impact
.font(.system(.largeTitle, weight: .bold).width(.compressed))  // tight, impactful headlines
.font(.system(.largeTitle, weight: .bold).width(.expanded))    // wide, spacious headlines

// Combine for character
Text("Recipe Book").font(.system(.largeTitle, design: .serif, weight: .bold))
Text("Step 1").font(.system(.headline, design: .serif))
Text("Fitness Pro").font(.system(.largeTitle, design: .rounded, weight: .black))
Text("Terminal").font(.system(.body, design: .monospaced))
```

Choose design variant based on the app's personality — **`.default` is correct for most apps**:
- **`.default` (SF Pro)**: The safe, professional choice for most apps. Use varied weights (.bold, .heavy, .black) and width variants (.compressed, .expanded) for visual hierarchy instead of switching font designs.
- **`.rounded` (SF Rounded)**: Only for genuinely playful apps — kids, casual games, fitness trackers
- **`.serif` (New York)**: Only for reading-heavy editorial content — news readers, book apps, long-form articles. Do NOT use for travel, food, social, or general-purpose apps.
- **`.monospaced` (SF Mono)**: Only for code editors, terminal apps, developer tools

When in doubt, use `.default`. Most Apple apps use SF Pro with weight variation for hierarchy. Do NOT mix font designs unless you have a clear editorial reason.

## Colors

### Semantic Colors (adapt to Light/Dark Mode automatically)

```swift
// Text
.foregroundStyle(.primary)       // Main text
.foregroundStyle(.secondary)     // Supporting text
.foregroundStyle(.tertiary)      // Tertiary content
Color(.placeholderText)         // Text field placeholders

// Backgrounds
Color(.systemBackground)        // Primary background
Color(.secondarySystemBackground)    // Grouped/card background
Color(.tertiarySystemBackground)     // Nested content

// Grouped backgrounds (for Form/List with .insetGrouped style)
Color(.systemGroupedBackground)
Color(.secondarySystemGroupedBackground)

// Separators
Color(.separator)               // Standard separator
Color(.opaqueSeparator)         // Opaque separator

// Interactive
.tint(.blue)                    // Default tint for buttons/links
```

### Color Rules

- NEVER hardcode colors — use semantic colors or Asset Catalog named colors
- Always support Dark Mode — semantic colors adapt automatically
- Keep the default **blue tint** for interactive elements (tabs, buttons, links). Custom accent colors (orange, purple, etc.) often look amateurish and clash with the iOS system UI. Only use a custom tint if the app has a strong, established brand color.
- Use `.tint()` for interactive elements, not `.foregroundStyle(.blue)`
- Minimum contrast ratio: **4.5:1** for body text, **3:1** for large text (18pt+)
- Use materials (`.ultraThinMaterial`, `.regularMaterial`) for translucent depth
- Use `MeshGradient` (iOS 18+) for organic, creative backgrounds — see `modern-apis.md`
- Test both light and dark appearances

## Components

### Tab Bar

- Maximum **5 tabs** — use "More" tab if needed (rare, avoid when possible)
- Use SF Symbols for tab icons — filled variant for selected state
- Keep tab labels short (1-2 words)
- Tab bar is always visible — never hide it during navigation
- iOS 18+: use `Tab("Title", systemImage:)` API, not `.tabItem` — see `modern-apis.md` for correct Tab API usage

### Navigation Bar

- Use `.navigationTitle()` — common pattern: `.large` for root views, `.inline` for child/detail views. But choose based on content: media-heavy screens may prefer `.inline` everywhere for more space.
- Leading: back button (automatic). Trailing: action buttons (1-2 max)
- Use `ToolbarItem(placement:)` for toolbar buttons

### Buttons

- Use native `Button` — never `.onTapGesture` for interactive elements
- `.borderedProminent` for primary actions (one per screen)
- `.bordered` for secondary actions
- `.plain` or `.borderless` for tertiary/text-link actions
- `.glass` / `.glassProminent` for Liquid Glass buttons (iOS 26+ only — see `liquid-glass-reference.md`)
- Destructive actions: `Button("Delete", role: .destructive)`
- Full-width buttons: `.frame(maxWidth: .infinity)` with `.buttonStyle(.borderedProminent)`

### Forms & Text Fields

- Use `Form` for settings-style layouts
- Use `TextField` with clear placeholder text describing expected input
- Use `SecureField` for passwords
- Show appropriate keyboard: `.keyboardType(.emailAddress)`, `.keyboardType(.phonePad)`
- Use `.textContentType()` for autofill: `.emailAddress`, `.password`, `.name`
- Chain fields with `@FocusState` and `.onSubmit` (see `navigation-sheets.md`)
- Use `.submitLabel(.next)` for field-to-field, `.submitLabel(.done)` for last field

### Lists

- Use `List` for scrollable content with built-in features (swipe actions, selection)
- Use `.listStyle(.insetGrouped)` for settings-style, `.plain` for content feeds
- Swipe actions: `.swipeActions(edge:)` — destructive on trailing, secondary on leading
- Empty state: **always** use `ContentUnavailableView("No Items", systemImage: "tray")` — never build custom empty state VStack. Place as `.overlay` on List, never inside it.

### Alerts & Sheets

- Use `.alert()` for critical decisions (destructive actions, confirmations)
- Use `.confirmationDialog()` for action menus (multiple choices)
- Use `.sheet()` for content presentation, `.fullScreenCover()` for immersive flows
- Destructive button always uses `role: .destructive` (renders red)
- Always provide a Cancel option

## Accessibility

- All images need `.accessibilityLabel()` if meaningful, `.accessibilityHidden(true)` if decorative
- Tap targets minimum 44×44pt — check with `.contentShape(Rectangle())`
- Support VoiceOver: meaningful labels, correct traits, logical reading order
- Use `.accessibilityHint()` for non-obvious actions (e.g. "Double-tap to edit")
- Support Dynamic Type at all sizes — test with largest accessibility size
- Use `.accessibilityAddTraits()` and `.accessibilityRemoveTraits()` for custom controls
- Never use color as the sole indicator of state — always pair with text, icons, or shapes
- Respect `@Environment(\.accessibilityReduceMotion)` — replace animations with fades or instant transitions
- Test with "Increase Contrast" and "Reduce Motion" settings

```swift
// Good: accessible toggle
Toggle("Notifications", isOn: $notificationsEnabled)

// Custom control with accessibility
Button { toggle() } label: {
    CustomToggleView(isOn: isOn)
}
.accessibilityLabel("Notifications")
.accessibilityValue(isOn ? "On" : "Off")
.accessibilityAddTraits(.isButton)
```

## SF Symbols

Use SF Symbols for all icons. Make sure symbols actually exist.

### Rendering Modes

| Mode | Behavior | Use |
|------|----------|-----|
| `.monochrome` | Single color, uniform | Default, simple icons |
| `.hierarchical` | Single color, layers vary in opacity | Depth without extra colors |
| `.palette` | Up to 3 custom colors per layer | Brand colors, custom styling |
| `.multicolor` | Intrinsic fixed colors (sun=yellow, etc.) | Realistic, system-standard look |

```swift
Image(systemName: "star.fill")
    .symbolRenderingMode(.multicolor)

// Animated symbol effects (iOS 17+)
Image(systemName: "wifi")
    .symbolEffect(.variableColor.iterative)

Image(systemName: "checkmark.circle")
    .symbolEffect(.bounce, value: isComplete)

// Replace with animation
Image(systemName: isPlaying ? "pause.fill" : "play.fill")
    .contentTransition(.symbolEffect(.replace))
```

Common categories:
- Navigation: `chevron.right`, `arrow.left`, `xmark`
- Actions: `plus`, `minus`, `trash`, `pencil`, `square.and.arrow.up`
- Status: `checkmark.circle.fill`, `exclamationmark.triangle`, `info.circle`
- Media: `play.fill`, `pause.fill`, `speaker.wave.2.fill`
- Communication: `message`, `envelope`, `phone`

## Native iOS Capabilities

Lean into native capabilities that web apps can't match. Use proactively when relevant:

### Haptics & Feedback
- `.sensoryFeedback(.impact, trigger: value)` — preferred SwiftUI declarative haptics (iOS 17+)
- `.sensoryFeedback(.success, trigger:)`, `.warning`, `.error` — notification-style
- `.sensoryFeedback(.selection, trigger:)` — subtle feedback for picker/selection changes
- `Core Haptics` — custom haptic patterns for immersive experiences
- Fallback: `UIImpactFeedbackGenerator` / `UINotificationFeedbackGenerator` for UIKit contexts

### Animations
- **Spring animations**: `.spring()`, `.snappy`, `.bouncy` for natural motion
- **Phase animations**: `PhaseAnimator` for multi-step animated sequences
- **Keyframe animations**: `KeyframeAnimator` for complex property-driven animations
- **Symbol effects**: `.bounce`, `.pulse`, `.variableColor`, `.replace`
- **Scroll-driven**: `.scrollTransition`, `.visualEffect` for parallax effects
- **Metal shaders**: `.colorEffect`, `.distortionEffect`, `.layerEffect`

### Platform Features
- **Live Activities & Dynamic Island** — real-time status (timers, tracking, progress)
- **Widgets** — Home screen and Lock Screen widgets with WidgetKit
- **SharePlay & Group Activities** — shared experiences over FaceTime
- **Core Motion** — accelerometer, gyroscope for motion-based interactions
- **MapKit** — rich maps with Look Around, custom annotations, route planning
- **Context menus** — `.contextMenu` with previews for rich long-press interactions
- **Spotlight** — `CoreSpotlight` for in-app content searchability

Don't force these into every app — use when the user's idea naturally benefits from a native capability.
