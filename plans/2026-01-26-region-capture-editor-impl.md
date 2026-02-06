# 区域截图编辑一体化实现计划

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 在编辑器中同时支持裁剪框调整与标注编辑，底部浮动工具条美化且在窄窗口不遮挡文字。

**Architecture:** 在编辑画布叠加裁剪交互层，裁剪状态存放于 EditorState；渲染/保存时按裁剪矩形裁剪输出；底部工具条使用自适应布局。

**Tech Stack:** SwiftUI, AppKit, ScreenCaptureKit

## 前置说明
- 当前目录不是 git 仓库，计划内的提交步骤需要先初始化 git 或由你自行跳过。
- 当前无测试目标，计划包含新增 XCTest 目标与基础单元测试步骤。

---

### Task 1: 裁剪状态与几何辅助

**Files:**
- Create: `Sources/GeometryExtensions.swift`
- Modify: `Sources/EditorState.swift`
- Test: `BianLanCaptureTests/CropGeometryTests.swift`

**Step 1: 写一个失败的单元测试（点位/矩形裁剪）**

```swift
import XCTest
@testable import BianLanCapture

final class CropGeometryTests: XCTestCase {
  func testClampPointToRect() {
    let rect = CGRect(x: 10, y: 10, width: 100, height: 100)
    let point = CGPoint(x: -5, y: 200)
    let clamped = rect.clamp(point: point)
    XCTAssertEqual(clamped.x, 10)
    XCTAssertEqual(clamped.y, 110)
  }
}
```

**Step 2: 运行测试，确认失败**

Run: `xcodebuild -project "BianLanCapture.xcodeproj" -scheme "BianLanCapture" test`
Expected: FAIL（找不到 clamp 方法或测试目标）

**Step 3: 写最小实现**

```swift
import AppKit

extension CGRect {
  func clamp(point: CGPoint) -> CGPoint {
    CGPoint(
      x: min(max(point.x, minX), maxX),
      y: min(max(point.y, minY), maxY)
    )
  }
}
```

在 `EditorState` 增加裁剪矩形：

```swift
@Published var cropRect: CGRect

init(image: NSImage) {
  self.image = image
  self.cropRect = CGRect(origin: .zero, size: image.size)
}
```

**Step 4: 运行测试，确认通过**

Run: `xcodebuild -project "BianLanCapture.xcodeproj" -scheme "BianLanCapture" test`
Expected: PASS

**Step 5: 提交**

```bash
git add "Sources/GeometryExtensions.swift" "Sources/EditorState.swift" "BianLanCaptureTests/CropGeometryTests.swift"
git commit -m "feat: add crop geometry helpers"
```

---

### Task 2: 裁剪交互层（控制点+遮罩）

**Files:**
- Create: `Sources/CropOverlayView.swift`
- Modify: `Sources/EditorView.swift`
- Test: `BianLanCaptureTests/CropHandleTests.swift`

**Step 1: 写一个失败的单元测试（裁剪矩形更新）**

```swift
import XCTest
@testable import BianLanCapture

final class CropHandleTests: XCTestCase {
  func testUpdateCropRectWithRightHandle() {
    let rect = CGRect(x: 0, y: 0, width: 100, height: 100)
    let updated = CropRectUpdater.update(rect: rect, handle: .right, delta: CGSize(width: 20, height: 0), imageSize: CGSize(width: 200, height: 200))
    XCTAssertEqual(updated.width, 120)
  }
}
```

**Step 2: 运行测试，确认失败**

Run: `xcodebuild -project "BianLanCapture.xcodeproj" -scheme "BianLanCapture" test`
Expected: FAIL（找不到 CropRectUpdater 或测试目标）

**Step 3: 写最小实现**

在 `Sources/CropOverlayView.swift` 中新增：

```swift
import SwiftUI

enum CropHandle: CaseIterable { case topLeft, top, topRight, right, bottomRight, bottom, bottomLeft, left }

enum CropRectUpdater {
  static func update(rect: CGRect, handle: CropHandle, delta: CGSize, imageSize: CGSize, minSize: CGSize = CGSize(width: 20, height: 20)) -> CGRect {
    var next = rect
    let dx = delta.width
    let dy = delta.height

    switch handle {
    case .topLeft:
      next.origin.x += dx
      next.origin.y += dy
      next.size.width -= dx
      next.size.height -= dy
    case .top:
      next.origin.y += dy
      next.size.height -= dy
    case .topRight:
      next.origin.y += dy
      next.size.width += dx
      next.size.height -= dy
    case .right:
      next.size.width += dx
    case .bottomRight:
      next.size.width += dx
      next.size.height += dy
    case .bottom:
      next.size.height += dy
    case .bottomLeft:
      next.origin.x += dx
      next.size.width -= dx
      next.size.height += dy
    case .left:
      next.origin.x += dx
      next.size.width -= dx
    }

    // 最小尺寸约束
    if next.width < minSize.width {
      let maxX = rect.maxX
      next.size.width = minSize.width
      if [.left, .topLeft, .bottomLeft].contains(handle) { next.origin.x = maxX - minSize.width }
    }
    if next.height < minSize.height {
      let maxY = rect.maxY
      next.size.height = minSize.height
      if [.top, .topLeft, .topRight].contains(handle) { next.origin.y = maxY - minSize.height }
    }

    // 限制在图像范围内
    next.origin.x = max(0, min(next.origin.x, imageSize.width - next.width))
    next.origin.y = max(0, min(next.origin.y, imageSize.height - next.height))
    return next
  }
}
```

同时新增裁剪视图 `CropOverlayView`，绘制遮罩、边框与控制点，并在命中控制点时更新 `state.cropRect`。

**Step 4: 运行测试，确认通过**

Run: `xcodebuild -project "BianLanCapture.xcodeproj" -scheme "BianLanCapture" test`
Expected: PASS

**Step 5: 提交**

```bash
git add "Sources/CropOverlayView.swift" "Sources/EditorView.swift" "BianLanCaptureTests/CropHandleTests.swift"
git commit -m "feat: add crop overlay and handle updates"
```

---

### Task 3: 画布绘制限制与裁剪输出

**Files:**
- Modify: `Sources/EditorView.swift`
- Modify: `Sources/EditorSession.swift`
- Modify: `Sources/ImageProcessor.swift`
- Test: `BianLanCaptureTests/CropImageTests.swift`

**Step 1: 写一个失败的单元测试（裁剪输出尺寸）**

```swift
import XCTest
@testable import BianLanCapture

final class CropImageTests: XCTestCase {
  func testCropImageSize() {
    let image = NSImage(size: NSSize(width: 200, height: 200))
    let rect = CGRect(x: 50, y: 50, width: 80, height: 60)
    let cropped = ImageProcessor.crop(image: image, rect: rect)
    XCTAssertEqual(cropped?.size.width, 80)
    XCTAssertEqual(cropped?.size.height, 60)
  }
}
```

**Step 2: 运行测试，确认失败**

Run: `xcodebuild -project "BianLanCapture.xcodeproj" -scheme "BianLanCapture" test`
Expected: FAIL（找不到 ImageProcessor.crop）

**Step 3: 写最小实现**

在 `Sources/ImageProcessor.swift` 增加裁剪函数：

```swift
static func crop(image: NSImage, rect: CGRect) -> NSImage? {
  guard let cgImage = image.cgImage() else { return nil }
  let scaleX = CGFloat(cgImage.width) / image.size.width
  let scaleY = CGFloat(cgImage.height) / image.size.height
  let scaled = CGRect(
    x: rect.origin.x * scaleX,
    y: rect.origin.y * scaleY,
    width: rect.size.width * scaleX,
    height: rect.size.height * scaleY
  )
  let flipped = CGRect(x: scaled.minX, y: CGFloat(cgImage.height) - scaled.maxY, width: scaled.width, height: scaled.height).integral
  guard let cropped = cgImage.cropping(to: flipped) else { return nil }
  return NSImage(cgImage: cropped, size: rect.size)
}
```

在 `EditorSession` 添加 `renderCroppedImage()` 并在保存/钉图时使用。
在 `EditorView` 的绘制手势中，限制绘制点在 `cropRect` 内（超出则 clamp）。

**Step 4: 运行测试，确认通过**

Run: `xcodebuild -project "BianLanCapture.xcodeproj" -scheme "BianLanCapture" test`
Expected: PASS

**Step 5: 提交**

```bash
git add "Sources/ImageProcessor.swift" "Sources/EditorSession.swift" "Sources/EditorView.swift" "BianLanCaptureTests/CropImageTests.swift"
git commit -m "feat: crop output uses crop rect"
```

---

### Task 4: 底部工具条与自适应布局

**Files:**
- Modify: `Sources/EditorView.swift`

**Step 1: 写一个失败的单元测试（工具条布局）**

说明：SwiftUI 布局难以单测，本任务以手动验收为主。

**Step 2: 实现底部浮动工具条**

- 移除顶部工具栏
- 新增底部 `EditorBottomToolbar`，使用 `ViewThatFits` 提供全量与紧凑两种布局
- 紧凑布局使用图标优先策略，避免文字被遮挡

**Step 3: 手动验收**

- 窗口缩小到较窄时，工具条自动切换紧凑模式
- 按钮文字不再被遮挡

**Step 4: 提交**

```bash
git add "Sources/EditorView.swift"
git commit -m "feat: bottom toolbar with adaptive layout"
```

---

### Task 5: 集成与验证

**Files:**
- Modify: `Sources/EditorView.swift`

**Step 1: 端到端手动验证**

- 裁剪框拖拽与标注同时可用
- 保存/钉图结果与裁剪预览一致
- 工具条在窄窗口仍可用

**Step 2: 构建验证**

Run: `xcodebuild -project "BianLanCapture.xcodeproj" -scheme "BianLanCapture" -configuration Debug build`
Expected: BUILD SUCCEEDED

**Step 3: 提交**

```bash
git add "Sources/EditorView.swift"
git commit -m "chore: verify crop+edit integration"
```
