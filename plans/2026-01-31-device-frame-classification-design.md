# Two-Level Device Frame Classification Design Document

## 1. Goal
Improve device frame selection by grouping models into Brand/Type categories. This will make it easier for users to find specific device frames in the picker.

## 2. Data Model
Add a `group` property to the `DeviceFrameTemplate` struct.

```swift
struct DeviceFrameTemplate: Identifiable, Codable, Hashable {
    // Existing properties...
    let group: String // New property
}
```

## 3. Logic
Implement a `detectGroup(from name: String)` function in `DeviceFrameStore` to automatically categorize frames based on their filenames or display names.

### Categorization Rules
The function will check for keywords in the following priority order:

1.  **Apple Phone**
    -   Keywords: "iphone", "ios"
2.  **Apple Computer**
    -   Keywords: "mac", "macbook"
3.  **Apple Watch**
    -   Keywords: "watch"
4.  **Apple Tablet**
    -   Keywords: "ipad"
5.  **Android Phone**
    -   Keywords: "pixel", "galaxy", "android"

### Fallback Strategy
If no keywords match, use the capitalized first word of the filename as the group name.

## 4. UI Implementation
Update the frame picker view to use SwiftUI's `Section` for grouping items.

```swift
List {
    ForEach(groupedFrames.keys.sorted(), id: \.self) { group in
        Section(header: Text(group)) {
            ForEach(groupedFrames[group] ?? []) { frame in
                DeviceFrameRow(frame: frame)
            }
        }
    }
}
```
