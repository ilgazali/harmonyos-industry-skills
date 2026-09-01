---
id: OFFICE-20
title: Insert and edit an image in a note - responsive breakpoints, picker import, PanGesture resize, drawImage commit
industry: 05_office
doc: huawei_industry_tree/05_office/docs/20_note_with_camera.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/note_with_camera-0000002378064505
sample: huawei_industry_tree/05_office/downloads/笔记中图片插入及编辑示例代码.zip
kits: ["@kit.ArkUI", "@kit.MediaLibraryKit", "@kit.CameraKit", "@kit.ImageKit", "@kit.CoreFileKit", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: [Canvas, CanvasRenderingContext2D, "CanvasRenderingContext2D.drawImage", "CanvasRenderingContext2D.getImageData", "CanvasRenderingContext2D.putImageData", "CanvasRenderingContext2D.scale", "CanvasRenderingContext2D.beginPath", "CanvasRenderingContext2D.closePath", PanGesture, "PanGesture.onActionUpdate", "PanGesture.onActionEnd", "PanGesture.onActionCancel", "photoAccessHelper.PhotoViewPicker", "PhotoViewPicker.select", "photoAccessHelper.PhotoSelectOptions", "cameraPicker.pick", "cameraPicker.PickerProfile", "cameraPicker.PickerMediaType", "image.createImageSource", "ImageSource.createPixelMapSync", "ImageSource.release", "PixelMap.release", "fileIo.openSync", "fileIo.closeSync", "window.getLastWindow", "window.on('windowSizeChange')", "window.off('windowSizeChange')", "window.on('avoidAreaChange')", "window.setPreferredOrientation", "window.setWindowSystemBarEnable", "window.setWindowLayoutFullScreen", "display.getDefaultDisplaySync", "AppStorage.setOrCreate", "@StorageLink", "@Link", "@State"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-05-0111, HW-05-0112, HW-05-0113, HW-05-0114, HW-05-0115, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when a note or document editor must let the user **drop a photo
onto a canvas, move and resize it by hand, and then commit it into the drawing**
- and must do so on both a phone and a tablet.

Three things come together here:

- **Import** from either the gallery (`PhotoViewPicker`) or a fresh shot
  (`cameraPicker.pick`), decoded to a `PixelMap`. Both are system pickers, so the
  feature needs **no permission at all**.
- **Direct manipulation** while the image is still floating: a `PanGesture` on
  the image body moves it, and eight `PanGesture`-bearing handle points around it
  resize it, with boundary clamping on both.
- **Commit** with `CanvasRenderingContext2D.drawImage`, which rasterises the
  floating image into the note at its current position and size, after which the
  floating copy is discarded.

The responsive half is a breakpoint string in `AppStorage` plus a small
`BreakpointType<T>` helper that picks one of three values per breakpoint.

## Feature checklist

The application must be able to:

- Derive a breakpoint from the window width and publish it into `AppStorage`,
  updating it as the window resizes.
- Choose per-breakpoint sizes for text, icons, bars and paddings through one
  helper.
- Hide the system bars and free the orientation on large windows; keep them on
  small ones.
- Import an image from the gallery or the camera and decode it to a `PixelMap`.
- Show the imported image floating over the canvas with eight resize handles.
- Drag the image with one finger and clamp it to the canvas bounds.
- Resize from any of the eight handles, clamping the same way.
- Rescale the already-drawn canvas content and the floating image when the window
  size changes, preserving the drawing via `getImageData` / `putImageData`.
- Commit the floating image into the canvas with `drawImage`, or discard it.

## Architecture

Single `entry` HAP:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | computes the breakpoint from the window width, sets system bars and orientation per breakpoint, publishes `currentBreakpoint` / `topRectHeight` / `bottomRectHeight` |
| `pages/MainPage.ets` | the note screen: title bar, tool bar, and the per-breakpoint sizing through `BreakpointType`; owns `tmpPix`, `pixWidth`, `pixHeight`, `isPointShow` |
| `components/NoteCanvas.ets` | **the whole editing surface**: the `Canvas`, the floating `Image`, the eight handles, both `PanGesture` families, and `drawImage` |
| `utils/PhotoUtils.ets` | gallery import: `PhotoViewPicker.select` -> URI -> `PixelMap` |
| `utils/CameraUtils.ets` | camera import: `cameraPicker.pick` -> URI -> `PixelMap` (byte-identical otherwise) |
| `utils/BreakpointType.ets` | `BreakpointType<T>.getValue(currentBreakpoint)` -> one of `sm` / `md` / `lg` |
| `model/PointModel.ets` | one resize handle: id plus x/y |
| `constants/Constants.ets` | breakpoint names, handle size, bar heights, paddings |

State is split between the page and the canvas by `@Link`, so the toolbar can
import an image and the canvas can consume it:

```ts
// MainPage owns them, NoteCanvas links them
@Link tmpPix: image.PixelMap | undefined   // the floating image, undefined once committed
@Link pixWidth: number                     // its current size, mutated by the handles
@Link pixHeight: number
@Link isPointShow: boolean                 // whether the handles and the action bar are visible
```

Editing flow:

```
toolbar -> PhotoUtils.insertImage() | CameraUtils.insertImage(uiContext)
             picker -> uri -> openSync(READ_ONLY) -> createImageSource(fd)
                    -> createPixelMapSync()  -> tmpPix

NoteCanvas
  floating Image(tmpPix)
    .onClick        -> isPointShow toggles; refreshPoint(); handlePop()
    PanGesture      -> handleDrag(offsetX, offsetY) + handlePop()
                       onActionEnd / onActionCancel -> reset()
  8 x PointModel handles
    PanGesture      -> handleChange(index, offsetX, offsetY)

  action bar (visible while isPointShow)
    插入            -> drawImage(); tmpPix = undefined; isPointShow = false
    删除            -> tmpPix = undefined; isPointShow = false
```

The `reset()` / `Tmp` pairing is the drag idiom: `pixPositionX/Y` hold the
committed origin, `pixPositionXTmp/YTmp` hold the live position during a gesture,
and `reset()` copies Tmp back into the base at gesture end - because `PanGesture`
offsets are cumulative within one gesture, not incremental.

## Implementation steps

1. **Declare no permission.** Both importers are system pickers that return a URI
   for what the user chose; the sample's `module.json5` has no
   `requestPermissions` block and the document has no 权限说明 section -
   consistent, and correct.
2. **Compute the breakpoint from the window width in vp.**
   `windowWidth / display.getDefaultDisplaySync().densityPixels`, then map to
   `sm` / `md` / `lg` and store it in `AppStorage`. Emit all three, not just two
   (HW-05-0112).
3. **React to resizes.** `window.getLastWindow` -> `on('windowSizeChange')` ->
   recompute. Check `err` before using `data` (HW-05-0114).
4. **Adapt the chrome per breakpoint.** `setWindowSystemBarEnable(['status',
   'navigation'])` and a portrait `setPreferredOrientation` on small windows;
   `setWindowSystemBarEnable([])` on large ones.
5. **Pick per-breakpoint values with one helper.**
   `new BreakpointType(smValue, mdValue, lgValue).getValue(this.currentBreakpoint)`
   at each call site.
6. **Import to a `PixelMap`.** Picker -> URI -> `fileIo.openSync(uri,
   READ_ONLY)` -> `image.createImageSource(fd)` -> `createPixelMapSync()`.
   Guard the `closeSync` and release the `ImageSource` in the `finally`
   (HW-05-0113).
7. **Float the image over the canvas** and build eight `PointModel` handles from
   the current position and size; recompute them on every change.
8. **Drag with `PanGesture({ fingers: 1 })`**: `onActionUpdate` applies the
   offset to the Tmp position with clamping, `onActionEnd` / `onActionCancel`
   call `reset()`.
9. **Resize from the handles** with a second `PanGesture` per handle, dispatching
   on the handle index so each corner and edge moves the right dimension.
10. **Rescale on window change.** Snapshot with `getImageData`, apply
    `ctx.scale(s, s)`, scale the stored position and size, then restore with
    `putImageData`.
11. **Commit with `drawImage`**, dividing every coordinate by `currentScale` so
    the image lands where the user sees it under the current canvas transform.
    Guard the `PixelMap` and release it afterwards (HW-05-0115).
12. **Unsubscribe precisely.** Pass the callback to
    `off('windowSizeChange', callback)` - the no-argument form cancels every
    subscription on that window (HW-05-0111).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### Breakpoint production

`笔记中图片插入及编辑示例代码.zip#entry/src/main/ets/entryability/EntryAbility.ets`

```ts
private updateBreakpoint(windowWidth: number, windowClass: window.Window, windowStage: window.WindowStage): void {
  let windowWidthVp = windowWidth / display.getDefaultDisplaySync().densityPixels;
  let curBp: string = '';
  if (windowWidthVp < 600) {
    curBp = Constants.BREAKPOINT_SM;
    windowStage.getMainWindowSync().setWindowSystemBarEnable(['status', 'navigation'])
    let orientation = window.Orientation.PORTRAIT;
    windowClass.setPreferredOrientation(orientation);
  } else {
    curBp = Constants.BREAKPOINT_LG;                    // md never produced - HW-05-0112
    windowStage.getMainWindowSync().setWindowSystemBarEnable([])
  }
  AppStorage.setOrCreate('currentBreakpoint', curBp);
}

onWindowStageCreate(windowStage: window.WindowStage): void {
  let windowClass: window.Window = windowStage.getMainWindowSync();
  window.getLastWindow(this.context, (err, data) => {
    windowClass = data;                                 // err unchecked - HW-05-0114
    this.updateBreakpoint(windowClass.getWindowProperties().windowRect.width, windowClass, windowStage);
    windowClass.on('windowSizeChange', (windowSize: window.Size) => {
      this.updateBreakpoint(windowSize.width, windowClass, windowStage);
    })
  });
  // ...
}
```

Dividing the raw window width by `densityPixels` is the conversion to vp that the
600/840 breakpoint thresholds are defined in. Corrected mapping:

```ts
if (windowWidthVp < 600) {
  curBp = Constants.BREAKPOINT_SM;
} else if (windowWidthVp < 840) {
  curBp = Constants.BREAKPOINT_MD;
} else {
  curBp = Constants.BREAKPOINT_LG;
}
```

### Breakpoint consumption

`笔记中图片插入及编辑示例代码.zip#entry/src/main/ets/utils/BreakpointType.ets`

```ts
export class BreakpointType<T> {
  sm: T;
  md: T;
  lg: T;

  constructor(sm: T, md: T, lg: T) {
    this.sm = sm;
    this.md = md;
    this.lg = lg;
  }

  getValue(currentBreakpoint: string): T {
    if (currentBreakpoint === Constants.BREAKPOINT_MD) {
      return this.md;
    }
    if (currentBreakpoint === Constants.BREAKPOINT_LG) {
      return this.lg;
    } else {
      return this.sm;
    }
  }
}
```

`笔记中图片插入及编辑示例代码.zip#entry/src/main/ets/pages/MainPage.ets`

```ts
@StorageLink('currentBreakpoint') currentBreakpoint: string = Constants.BREAKPOINT_SM;

Text(/* ... */).fontSize(new BreakpointType(22, 28, 28).getValue(this.currentBreakpoint))
Image(/* ... */)
  .width(new BreakpointType(25, 30, 30).getValue(this.currentBreakpoint))
  .height(new BreakpointType(25, 30, 30).getValue(this.currentBreakpoint))
// ...
.padding({
  top: new BreakpointType(this.uiContext.px2vp(this.topRectHeight), Constants.TOP_PADDING,
    Constants.TOP_PADDING).getValue(this.currentBreakpoint)
})
```

This generic-over-`T` helper is the reusable piece: it works for numbers,
strings and resources alike, and keeps the breakpoint conditional out of the
build method.

### Import to a PixelMap

`笔记中图片插入及编辑示例代码.zip#entry/src/main/ets/utils/PhotoUtils.ets`

```ts
export class PhotoUtils {
  static async selectImage(): Promise<image.PixelMap | undefined> {
    let uri = await PhotoUtils.openPhotoViewPicker();
    if (uri) {
      return PhotoUtils.loadImage(uri);
    } else {
      Logger.error('Failed to get uri.');
      return undefined;
    }
  }

  private static async openPhotoViewPicker(): Promise<string> {
    let photoPicker: photoAccessHelper.PhotoViewPicker = new photoAccessHelper.PhotoViewPicker();
    let photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
    photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
    photoSelectOptions.maxSelectNumber = 1;
    try {
      return (await photoPicker.select(photoSelectOptions)).photoUris[0];
    } catch (err) {
      Logger.error(`Failed to get photo image uri.code: ${err.code}, message: ${err.message}`);
      return '';
    }
  }

  private static loadImage(name: string): image.PixelMap {
    let fileSource: fileIo.File | undefined = undefined
    let imageSource: image.ImageSource | undefined = undefined;
    try {
      fileSource = fileIo.openSync(name, fileIo.OpenMode.READ_ONLY);
      imageSource = image.createImageSource(fileSource.fd);
      return imageSource.createPixelMapSync();
    } finally {
      fileIo.closeSync(fileSource)               // may be undefined; source never released - HW-05-0113
    }
  }
}
```

The cancellation handling here is the right shape - `select` rejects on dismissal,
the `catch` returns `''`, and `selectImage` turns that into `undefined` rather
than a broken import. `CameraUtils` is identical apart from
`cameraPicker.pick(context.getHostContext(), [cameraPicker.PickerMediaType.PHOTO],
pickerProfile)` and `pickerResult.resultUri.toString()`.

Corrected `loadImage`:

```ts
private static loadImage(name: string): image.PixelMap {
  let fileSource: fileIo.File | undefined = undefined;
  let imageSource: image.ImageSource | undefined = undefined;
  try {
    fileSource = fileIo.openSync(name, fileIo.OpenMode.READ_ONLY);
    imageSource = image.createImageSource(fileSource.fd);
    return imageSource.createPixelMapSync();
  } finally {
    imageSource?.release();
    if (fileSource !== undefined) {
      fileIo.closeSync(fileSource);
    }
  }
}
```

### Drag and commit

`笔记中图片插入及编辑示例代码.zip#entry/src/main/ets/components/NoteCanvas.ets`

```ts
Image(this.tmpPix)
  // ...
  .onClick(() => {
    this.reset()
    this.isPointShow = !this.isPointShow
    this.refreshPoint()
    this.handlePop()
  })
  .gesture(
    PanGesture({ fingers: 1 })
      .onActionUpdate(event => {
        this.handleDrag(event.offsetX, event.offsetY)
        this.handlePop()
      })
      .onActionEnd(() => {
        this.reset()
      })
      .onActionCancel(() => {
        this.reset()
      })
  )

// action bar, visible while isPointShow
Text($r('app.string.insert_to_canvas'))
  .onClick(() => {
    this.drawImage()
    this.tmpPix = undefined          // PixelMap dropped, not released - HW-05-0115
    this.isPointShow = false
  })
Text($r('app.string.delete'))
  .onClick(() => {
    this.tmpPix = undefined
    this.isPointShow = false
  })

// Draw the pix to the canvas
drawImage() {
  this.myPen.beginPath()
  let imgBitmap = this.tmpPix        // typed PixelMap | undefined - HW-05-0115
  this.myPen.drawImage(imgBitmap, this.pixPositionXTmp / this.currentScale, this.pixPositionYTmp / this.currentScale,
    this.pixWidth / this.currentScale, this.pixHeight / this.currentScale)
  this.myPen.closePath()
}
```

The `/ this.currentScale` on every coordinate is the important detail: the canvas
context has been scaled by `currentScale` after a window resize, so the on-screen
position and size have to be divided back into context units before drawing.

Corrected commit:

```ts
drawImage() {
  const bitmap = this.tmpPix;
  if (!bitmap) {
    return;
  }
  this.myPen.beginPath();
  this.myPen.drawImage(bitmap, this.pixPositionXTmp / this.currentScale, this.pixPositionYTmp / this.currentScale,
    this.pixWidth / this.currentScale, this.pixHeight / this.currentScale);
  this.myPen.closePath();
  bitmap.release();
  this.tmpPix = undefined;
}
```

### Rescaling the drawing when the window changes

`笔记中图片插入及编辑示例代码.zip#entry/src/main/ets/components/NoteCanvas.ets`

```ts
aboutToAppear(): void {
  if (this.currentBreakpoint === Constants.BREAKPOINT_LG) {
    this.canvasWidth = this.uiContext.px2vp(display.getDefaultDisplaySync().width)
    this.canvasHeight = this.canvasWidth / 2
  } else {
    this.canvasWidth = this.uiContext.px2vp(display.getDefaultDisplaySync().width)
    this.canvasHeight = this.uiContext.px2vp(display.getDefaultDisplaySync().height) - Constants.CANVAS_START_Y
  }
  this.originCanvasWidth = this.canvasWidth
  this.originCanvasHeight = this.canvasHeight
  let context = this.uiContext.getHostContext() as common.UIAbilityContext;
  window.getLastWindow(context, (err, data) => {
    let windowClass: window.Window = data;
    windowClass.on('windowSizeChange', (windowSize: window.Size) => {
      this.imageDataTmp = this.myPen.getImageData(0, 0, this.canvasWidth, this.canvasHeight)
      let tmpCanvasWidth = this.uiContext.px2vp(display.getDefaultDisplaySync().width)
      this.currentScale = tmpCanvasWidth / this.originCanvasWidth
      this.myPen.scale(this.currentScale, this.currentScale)
      this.canvasWidth = this.uiContext.px2vp(display.getDefaultDisplaySync().width)
      this.canvasHeight = this.canvasWidth / 2
      this.pixPositionXTmp = this.pixPositionXTmp * this.currentScale
      this.pixPositionYTmp = this.pixPositionYTmp * this.currentScale
      this.pixWidth = this.pixWidth * this.currentScale
      this.pixHeight = this.pixHeight * this.currentScale
      this.myPen.putImageData(this.imageDataTmp, 0, 0)
      this.reset()
      this.refreshPoint()
      this.handlePop()
    })
  });
}

aboutToDisappear(): void {
  let context = this.uiContext.getHostContext() as common.UIAbilityContext;
  window.getLastWindow(context, (err, data) => {
    let windowClass: window.Window = data;
    windowClass.off('windowSizeChange')          // cancels ALL subscriptions - HW-05-0111
  });
}
```

The `getImageData` -> `scale` -> `putImageData` sandwich is the reusable idea:
the already-drawn strokes are snapshotted, the context transform is changed, and
the snapshot is restored so previous content survives the resize.

Corrected unsubscribe:

```ts
private sizeChangeCallback: Callback<window.Size> = (windowSize: window.Size) => { /* ... */ };

aboutToDisappear(): void {
  const context = this.uiContext.getHostContext() as common.UIAbilityContext;
  window.getLastWindow(context, (err, data) => {
    if (err.code) {
      return;
    }
    data.off('windowSizeChange', this.sizeChangeCallback);
  });
}
```

### The eight resize handles

`笔记中图片插入及编辑示例代码.zip#entry/src/main/ets/components/NoteCanvas.ets`

```ts
// Refresh the position of the point according to id.
refreshPoint() {
  this.points = []
  this.points.push(new PointModel(0,
    this.pixPositionXTmp - Constants.POINT_WIDTH_HEIGHT / 2,
    this.pixPositionYTmp - Constants.POINT_WIDTH_HEIGHT / 2))
  this.points.push(new PointModel(1,
    this.pixPositionXTmp + this.pixWidth / 2 - Constants.POINT_WIDTH_HEIGHT / 2,
    this.pixPositionYTmp - Constants.POINT_WIDTH_HEIGHT / 2))
  // ... ids 2..7 for the remaining corners and edge midpoints
}

// each handle
PanGesture({ fingers: 1 })
  .onActionUpdate(event => {
    this.handleChange(index, event.offsetX, event.offsetY)
  })
```

Handles are rebuilt from the current position and size rather than moved
individually, so `refreshPoint()` is the single place that defines the handle
geometry and is called after every drag, resize and rescale.

## Permissions & config

`笔记中图片插入及编辑示例代码.zip#entry/src/main/module.json5` declares **no
`requestPermissions` block**, and none is needed:

- `PhotoViewPicker.select` is a system picker returning a read-only URI for the
  selected item - no `ohos.permission.READ_IMAGEVIDEO`.
- `cameraPicker.pick` opens the system camera UI - no `ohos.permission.CAMERA`.
  Compare OFFICE-05 (HW-05-0022) and OFFICE-07 (HW-05-0040), where a camera
  permission was declared for a system picker that does not need one.
- Nothing is saved, so no media-write permission either; the note lives only in
  the canvas.

The document has no 权限说明 section, which matches - verified consistent, and
the project tree matches the shipped ZIP exactly.

`build-profile.json5` pins the SDK to `6.0.0(20)`.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **Breakpoint thresholds are in vp, not px.** The width from
  `windowSizeChange` and `getWindowProperties` is in px and must be divided by
  `display.getDefaultDisplaySync().densityPixels` first.
- **`off(type)` without a callback is global.** The reference states that with no
  callback "all subscriptions to the specified event are canceled" - so any
  component sharing a window with another subscriber must pass its own callback.
- **`getLastWindow` returns the app's last-shown window**, which the ability and
  the canvas both use - that is why their `windowSizeChange` subscriptions
  collide.
- **`PanGesture` offsets are cumulative within a gesture.** `onActionUpdate`
  delivers the total offset since the gesture started, which is why the sample
  keeps a `Tmp` position and copies it back in `reset()` at gesture end.
- **`drawImage` is destructive.** Once committed, the image is part of the canvas
  bitmap and cannot be moved again; the floating `PixelMap` is discarded.
- **The canvas transform persists.** `ctx.scale()` applied on resize stays in
  effect, so every subsequent coordinate must be divided by `currentScale`.
- **Large-window layout drops the system bars.**
  `setWindowSystemBarEnable([])` on `lg`, and the portrait
  `setPreferredOrientation` only on `sm`.
- **Nothing is persisted.** The note is not saved anywhere; closing the app loses
  the drawing.

## Pitfalls

- **`windowClass.off('windowSizeChange')` with no callback is incorrect.** The
  reference says the no-argument form cancels every subscription to that event,
  and `EntryAbility` subscribes to the same event on the same window - so leaving
  the note canvas silently stops the app's breakpoint updates. Pass the
  component's own callback. (HW-05-0111)
- **Producing only `sm` and `lg` is incorrect** for a three-value
  `BreakpointType`: the `md` branch and every `md` argument at every call site are
  unreachable, and a 600-840 vp window gets the large layout. Emit all three, or
  drop `md` from the helper. (HW-05-0112)
- **`fileIo.closeSync(fileSource)` in the `finally` is incorrect** because
  `fileSource` is still `undefined` on exactly the path the `finally` exists for -
  an `openSync` failure - and the `ImageSource` declared beside it is never
  released. (HW-05-0113)
- **Ignoring `err` in the three `getLastWindow` callbacks is incorrect**, and
  neither of the ability's two window subscriptions is released in
  `onWindowStageDestroy`. (HW-05-0114)
- **Passing `this.tmpPix` to `drawImage` unguarded is incorrect** - it is typed
  `PixelMap | undefined` and the surrounding code sets it to `undefined` on both
  the insert and the delete path - and the discarded `PixelMap` is never
  released. (HW-05-0115)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `off('windowSizeChange')` ("If no value is passed in, all subscriptions to the
  specified event are canceled"), `on('windowSizeChange')`,
  `on`/`off('avoidAreaChange')`, `getLastWindow`, `setWindowSystemBarEnable`,
  `setPreferredOrientation`, `setWindowLayoutFullScreen`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/02_application-framework/ts-canvasrenderingcontext2d.md` -
  the three `drawImage(image: ImageBitmap | PixelMap, ...)` overloads,
  `getImageData`, `putImageData` and `scale`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-canvasrenderingcontext2d
- `documentation/harmonyos-guides/02_media/image-decoding.md` - step 5, releasing
  the `PixelMap` and `ImageSource` instances.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-decoding
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `openSync` with `OpenMode.READ_ONLY` and `closeSync(file: number | File)`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-guides/05_media/photoaccesshelper-photoviewpicker.md` -
  `PhotoViewPicker.select` and the read-only URI it returns.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` -
  `PanGesture` with `onActionUpdate` / `onActionEnd` / `onActionCancel`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/js-apis-display.md` -
  `getDefaultDisplaySync` and `densityPixels`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-display
