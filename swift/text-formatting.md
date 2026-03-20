# SwiftUI Text Formatting Guide

> Never use String(format:). Use Text(value, format:) for type-safe, locale-aware formatting. Use localizedStandardContains for search.

## Number Formatting

```swift
let value = 42.12345

// Fixed decimal places
Text(value, format: .number.precision(.fractionLength(2)))
// → "42.12"

// With grouping separator
Text(1234567, format: .number)
// → "1,234,567" (locale-dependent)

// Without grouping
Text(1234567, format: .number.grouping(.never))
// → "1234567"
```

## Currency

```swift
let price = 19.99

Text(price, format: .currency(code: "USD"))
// → "$19.99"

// With locale
Text(price, format: .currency(code: "EUR").locale(Locale(identifier: "de_DE")))
// → "19,99 €"
```

## Percentage

```swift
let progress = 0.856

Text(progress, format: .percent.precision(.fractionLength(1)))
// → "85.6%"

Text(progress, format: .percent.precision(.fractionLength(0)))
// → "86%"
```

## Date & Time

```swift
let date = Date()

// Date components
Text(date, format: .dateTime.day().month().year())
// → "Jan 23, 2026"

Text(date, format: .dateTime.day().month(.wide).year())
// → "January 23, 2026"

// Time
Text(date, format: .dateTime.hour().minute())
// → "2:30 PM"

// Built-in styles
Text(date, style: .date)      // "1/23/26"
Text(date, style: .time)      // "2:30 PM"
Text(date, style: .relative)  // "in 1 hour"
Text(date, style: .timer)     // "0:12" (live elapsed time from date)
```

## Search — Localized String Matching

Use `localizedStandardContains` for user-input filtering. It handles case, diacritics, and locale-aware matching.

```swift
let searchText = "cafe"
let items = ["Café Latte", "Coffee", "Tea"]

// ✅ Handles diacritics + case
let filtered = items.filter { $0.localizedStandardContains(searchText) }
// Matches "Café Latte"

// ❌ Exact match only — misses diacritics
let filtered = items.filter { $0.contains(searchText) }
```

For simpler case-insensitive search:
```swift
items.filter { $0.localizedCaseInsensitiveContains(searchText) }
```

## Sorting — Locale-Aware

```swift
let names = ["Zoë", "Zara", "Åsa"]

// ✅ Locale-aware sorting
let sorted = names.sorted { $0.localizedStandardCompare($1) == .orderedAscending }
// → ["Åsa", "Zara", "Zoë"]

// ❌ Byte-wise — wrong order for non-ASCII
let sorted = names.sorted()
```

## Styled Text — Concatenation & Markdown

```swift
// Concatenation for mixed styles
Text("Hello ")
    .foregroundStyle(.primary)
+ Text("World")
    .foregroundStyle(.blue)
    .bold()

// Markdown in Text — only works with string LITERALS, not String variables
Text("This is **bold** and *italic*")
Text("Visit [Apple](https://apple.com)")

// For dynamic markdown strings, use AttributedString:
let dynamicText = "This is **bold**"
Text(try! AttributedString(markdown: dynamicText))
```

## AttributedString

```swift
// AttributedString attributes OVERRIDE view modifiers (.foregroundStyle on Text is ignored for styled ranges)
var text = AttributedString("Hello World")
if let range = text.range(of: "World") {
    text[range].foregroundColor = .blue
    text[range].font = .body.bold()
}
Text(text)
```
