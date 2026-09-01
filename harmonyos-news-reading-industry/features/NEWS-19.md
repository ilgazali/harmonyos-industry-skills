---
id: NEWS-19
title: Smear to recognise - erase a Canvas mask to select an image region, OCR the crop, copy the text
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/19_erase_recognize.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/erase_recognize-0000002313854866
sample: huawei_industry_tree/11_news_reading/downloads/EraseRecognize.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.CoreVisionKit", "@kit.ImageKit", "@kit.PerformanceAnalysisKit"]
apis: [Canvas, CanvasRenderingContext2D, globalCompositeOperation, fillRect, clearRect, stroke, onTouch, onReady, onAreaChange, "Image.onComplete", "UIContext.vp2px", "UIContext.px2vp", "image.createImageSource", createPixelMap, "PixelMap.crop", "textRecognition.recognizeText", VisionInfo, TextRecognitionConfiguration, "pasteboard.createData", MIMETYPE_TEXT_PLAIN, getSystemPasteboard, setData, CustomContentDialog, CustomDialogController, LengthMetrics]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-11-0021, HW-11-0031]
status: verified-with-fixes
---

## When to use

Load this card when the user must **point at part of an image with a finger, not
a rectangle**, and the app then has to act on that region. Here the action is
OCR and copy-to-clipboard; the same construction serves "translate this bit",
"search this logo", "redact this area", or any crop-then-analyse flow.

The pattern is a **scratch-card**: a `Canvas` sized to the displayed image sits
directly on top of it, filled with a translucent mask. The finger does not draw
- it erases, via `globalCompositeOperation = 'destination-out'`, so the smear
reveals the photo underneath. Meanwhile every touch point is recorded, and on
lift their bounding box becomes the crop rectangle. The user sees a highlighter;
the app gets a rectangle.

The transferable part is the **coordinate chain**. Touch points arrive in vp,
the `PixelMap` is indexed in source-image pixels, and the two are separated by
both the display density and the image's display scale. Getting that chain right
is most of the work in any crop-from-a-gesture feature, and this sample gets it
right - see the Architecture note.

## Feature checklist

- A full-screen photo with a translucent dark mask over it.
- Dragging a finger erases a 30 vp-wide round-capped stroke through the mask,
  revealing the photo along the path.
- On lift, the bounding box of the stroke (expanded by half the stroke width) is
  cropped out of the source image.
- The crop is passed to on-device text recognition; the recognised string is
  appended to a results list.
- A 识别 (recognise) button opens a bottom sheet listing the recognised strings,
  one row each.
- Each row has a 复制 (copy) control that puts that string on the system
  clipboard and confirms with a toast.
- The mask is repainted for the next smear.

## Architecture

One `entry` module, one page. There is no model layer; the only data type is a
two-field touch point.

```
entry/src/main/ets
├── entryability/EntryAbility.ets   full-screen window, avoid areas -> AppStorage, exports fullScreen()
├── entrybackupability/EntryBackupAbility.ets
└── pages/Index.ets                 @Entry: the whole feature - mask, crop, OCR, dialog, clipboard
```

The documented tree matches the zip.

**The design decision worth copying** is the coordinate chain, and it is worth
walking through because it is the part that has to be exactly right:

```
Image.onComplete  ->  imageWidth  = px2vp(event.contentWidth)   // Canvas is sized in vp to the drawn image
                      ratio       = event.width / event.contentWidth   // source px per drawn px
touch (vp)        ->  vp2px(minX)                               // vp -> device px
                      * this.ratio                              // device px -> source-image px
                  ->  pixelMap.crop({ x, y, size })             // indexed in source pixels
```

Three different length units are in play - vp for layout and touch, device
pixels for the screen, and source-image pixels for the `PixelMap` - and each
conversion happens exactly once, in the right order. The `Canvas` is *sized* to
the drawn image in vp so that its own coordinate space and the touch coordinate
space are the same thing, which is what removes a fourth conversion. If you copy
one idea from this sample, copy this.

**The decision worth avoiding** is the `flag` field. It is initialised to `0`,
the touch handler resets the results and repaints the mask when it is `0`, and
then sets it to `1` - permanently. The reset therefore runs on the first touch of
the app's lifetime and never again, so the second smear neither clears
`dataValues` nor repaints the mask. Whatever it was meant to guard, a
one-shot integer is the wrong shape; the reset belongs on `TouchType.Down`
unconditionally, or in the code that finishes a recognition.

## Implementation steps

1. **Stack the `Canvas` on the `Image`,** and size the `Canvas` from the image's
   own `onComplete` event rather than from the container, so mask coordinates
   and image coordinates coincide.
2. **Capture `ratio` in the same callback:** `event.width / event.contentWidth`
   is the source-pixels-per-drawn-pixel factor, and it is the only thing that
   makes the crop land on the right part of the photo when the image is scaled
   to fit.
3. **Configure the brush in `onReady`,** which runs once when the `Canvas` is
   ready: `lineWidth`, `lineCap = 'round'`, `lineJoin = 'round'`, and
   `globalCompositeOperation = 'destination-out'`.
4. **Paint the mask when the size first becomes valid,** in `onAreaChange`,
   switching to `'source-over'` for the `fillRect` and back to
   `'destination-out'` afterwards. Guard with a `hasDrawnBackground` flag so a
   later layout change does not repaint over the user's smear.
5. **Record every touch point** into `erasePathList` on `Down` and `Move` while
   stroking the path, so the reveal and the selection are built from the same
   coordinates.
6. **On `Up`, compute the bounding box** and expand it by `PATH_LINE_WIDTH / 2`
   on each side - the stroke is drawn centred on the path, so its visible extent
   is half a line width beyond the recorded points.
7. **Convert and crop**: `vp2px` each bound, multiply by `ratio`, then
   `await pixelMap.crop({ x, y, size })`. `crop` mutates the `PixelMap` in
   place.
8. **Recognise** with `textRecognition.recognizeText(visionInfo,
   textConfiguration)` and push `data.value` onto an `@State string[]` that the
   dialog builder reads.
9. **Copy with the ArkTS pasteboard**, `pasteboard.createData(
   pasteboard.MIMETYPE_TEXT_PLAIN, item)` then
   `getSystemPasteboard().setData(...)`. The document links the NDK C API here;
   the sample uses the ArkTS module and so should you (`HW-11-0021`).

## Verified snippets

All snippets are from `EraseRecognize.zip`,
`entry/src/main/ets/pages/Index.ets`. Corrected forms are marked.

**The mask and its brush** (as shipped)

```typescript
const PATH_LINE_WIDTH: number = 30;

Image($r('app.media.back'))
  .height('100%')
  .width('100%')
  .onComplete((event) => {
    if (event) {
      this.imageWidth = this.getUIContext().px2vp(event.contentWidth);
      this.imageHeight = this.getUIContext().px2vp(event.contentHeight);
      this.ratio = event.width / event.contentWidth;
    }
  });
Canvas(this.canvasContext)
  .width(this.imageWidth)
  .height(this.imageHeight)
  .onReady(() => {
    // 这个回调函数在 Canvas 组件准备好时，执行一次
    this.canvasContext.lineWidth = PATH_LINE_WIDTH;
    this.canvasContext.lineCap = 'round';
    this.canvasContext.lineJoin = 'round';
    this.canvasContext.globalCompositeOperation = 'destination-out';
  })
```

**`globalCompositeOperation = 'destination-out'` is the entire trick.** In that
mode the source shape does not paint colour, it removes alpha from what is
already on the canvas. So a stroke drawn over the mask punches a hole in it, and
because the `Canvas` is transparent everywhere else, the photo behind shows
through. No pixel manipulation, no second layer, no shader - one attribute.

`lineCap: 'round'` and `lineJoin: 'round'` are not cosmetic either. With butt
caps the revealed strip ends in a hard rectangle and, more importantly, the
bounding box computed from the path points would understate the revealed area
at the ends of the stroke by half a line width - which is precisely the
correction step 6 applies.

The `Image` and `Canvas` are siblings in a `Stack`, and the `Canvas` takes its
size from the image's *content* box, not from the container. That is what keeps
touch coordinates, mask coordinates and the crop in agreement.

**From a smear to a crop rectangle** (as shipped)

```typescript
} else if (event.type === TouchType.Up) {
  this.canvasContext.closePath();
  if (this.erasePathList.length > 0) {
    setTimeout(() => {
      let minX = Infinity;
      let maxX = -Infinity;
      let minY = Infinity;
      let maxY = -Infinity;

      // 遍历 erasePathList 找到用户绘制路径的最小和最大 X、Y 坐标
      this.erasePathList.forEach((path: ErasePath) => {
        minX = Math.min(minX, path.x);
        maxX = Math.max(maxX, path.x);
        minY = Math.min(minY, path.y);
        maxY = Math.max(maxY, path.y);
      });

      minX -= PATH_LINE_WIDTH / 2;
      minY -= PATH_LINE_WIDTH / 2;
      maxX += PATH_LINE_WIDTH / 2;
      maxY += PATH_LINE_WIDTH / 2;
      this.cropImage(this.getUIContext().vp2px(minX), this.getUIContext().vp2px(minY),
        this.getUIContext().vp2px(maxX), this.getUIContext().vp2px(maxY));
    }, 0);
  }
}
```

**The half-line-width expansion is a correctness fix, not padding.** The points
in `erasePathList` are the *centre line* of the stroke; the visible reveal
extends `PATH_LINE_WIDTH / 2` beyond it in every direction. Crop to the raw
bounding box and you cut the top and bottom off every character the user thought
they had selected, which for a 30 vp brush over body text is most of the glyph
height. OCR accuracy on a vertically clipped line is close to zero, so this
adjustment is the difference between the feature working and the feature
returning empty strings.

**`setTimeout(..., 0)` defers the crop by one turn of the event loop,** which
lets the final `stroke()` of the gesture commit before the page starts an async
decode-and-recognise. It is a scheduling hint rather than a correctness
requirement, but it keeps the reveal visually instantaneous.

**Crop and recognise** (as shipped)

```typescript
async cropImage(minX: number, minY: number, maxX: number, maxY: number) {
  let pixelMap = await this.getPixelMap();
  await pixelMap.crop({
    x: minX * this.ratio,
    y: minY * this.ratio,
    size: { width: (maxX - minX) * this.ratio, height: (maxY - minY) * this.ratio }
  });
  this.imagePixelMap = pixelMap;
  this.textRecognitionTest();
  return pixelMap;
}

textRecognitionTest() {
  if (!this.imagePixelMap) {
    return;
  }
  let visionInfo: textRecognition.VisionInfo = {
    pixelMap: this.imagePixelMap
  };
  let textConfiguration: textRecognition.TextRecognitionConfiguration = {
    isDirectionDetectionSupported: false
  };
  textRecognition.recognizeText(visionInfo, textConfiguration)
    .then((data: textRecognition.TextRecognitionResult) => {
      this.dataValues.push(data.value);
      hilog.info(0x0000, 'OCRDemo', data.value);
    })
    .catch((error: BusinessError) => {
      hilog.error(0x0000, 'OCRDemo', `Failed to recognize text. Code: ${error.code}, message: ${error.message}`);
    });
}
```

**`crop` mutates the `PixelMap` in place** - it does not return a new one, which
is why the sample decodes a fresh copy from resources for every smear rather
than cropping the one it is displaying. That is the right instinct (cropping the
displayed map would destroy the photo on screen) implemented the expensive way:
`getPixelMap()` re-reads and re-decodes the whole JPEG on each lift, and neither
the source nor the crop is ever `release()`d. Cache one decoded master and crop
a copy of it instead.

`isDirectionDetectionSupported: false` is worth understanding before you flip
it. Direction detection lets the engine handle rotated text at the cost of extra
passes; for a crop taken from an upright photo it is wasted work, and turning it
on is only justified when the source can be sideways.

The results array is `@State dataValues: string[]` and is appended with `push`,
which ArkUI observes - the dialog's `ForEach` re-renders without any explicit
notification.

**Copying to the clipboard** (as shipped - and the API the doc should be pointing at, see `HW-11-0021`)

```typescript
import { BusinessError, pasteboard } from '@kit.BasicServicesKit';

Row() {
  Text($r('app.string.Copy'));
  Image($r('app.media.copy')).height(24).width(24);
}
.onClick(async () => {
  // 创建一条纯文本类型的剪贴板内容对象
  let pasteData = pasteboard.createData(pasteboard.MIMETYPE_TEXT_PLAIN, item);
  // 将数据写入系统剪贴板
  let systemPasteboard = pasteboard.getSystemPasteboard();
  await systemPasteboard.setData(pasteData);
  this.getUIContext().getPromptAction().showToast({
    message: $r('app.string.Copy_Succeed'),
    alignment: Alignment.Bottom,
    offset: { dx: 0, dy: -340 }
  });
});
```

**This is the ArkTS pasteboard (`@ohos.pasteboard`, re-exported through
`@kit.BasicServicesKit`), not the NDK one.** `createData` builds a
`PasteData` with one record of the given MIME type, and `setData` is the
asynchronous write to the system board. The document's 参考文档 links
`capi-pasteboard` - the C/NDK surface - for both mentions, so a reader following
the reference lands on `OH_Pasteboard_SetData` and native handles that share no
identifier with a single line of this sample (`HW-11-0021`).

The sample additionally reads the board straight back with `getData()` and logs
each record. That is verification scaffolding for a demo, not something to ship:
on recent releases reading the system clipboard raises a user-visible paste
notification, so a copy button that immediately reads back will look to the user
like the app pasted something.

## Permissions & config

**None.** The sample declares no `requestPermissions` - on-device text
recognition through Core Vision Kit needs no permission because the image never
leaves the device and the app supplies the `PixelMap` itself.

`deviceTypes` is `phone`, `tablet`, `2in1`. Resources carry `base` plus a `dark`
colour override. `EntryAbility` exports a `fullScreen()` helper and publishes
`topRectHeight` / `bottomRectHeight` into `AppStorage`; the page calls
`fullScreen(false)` in `aboutToAppear`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The mask is only reset once.** `this.flag` goes `0 -> 1` on the first touch
  and is never restored, so from the second smear on, the previous reveal stays
  open and `dataValues` keeps growing. See the Architecture note.
- **The blue selection rectangle is never drawn.** After the crop, the touch
  handler sets `strokeStyle = '#00bfff'`, calls `beginPath()`, then resets
  `maxX`/`minY`/`maxY` back to infinities without ever calling `rect()`, and
  strokes an empty path. The comment 设置边界框样式并绘制 promises a box the code
  does not produce.
- **The two mask colours disagree.** `onAreaChange` fills with `'#00ffffff'` -
  alpha `00`, fully transparent - while the touch handler repaints with
  `'#33000000'`. On first launch there is therefore no visible mask until the
  first touch.
- **Every smear re-decodes the source image** and no `PixelMap` is ever
  released. Cache the decoded master and release crops.
- The 识别 button is positioned with `margin({ top: 725 })` inside the `Stack`,
  and the confirmation toast with `offset: { dy: -340 }`. Both are absolute
  values for one screen height.
- Text recognition is on-device and language-limited; check the Core Vision Kit
  reference for the supported scripts before promising it on arbitrary content.

## Pitfalls

- **`HW-11-0021`** (E/low, confirmed): the document's 场景介绍 and 参考文档 link
  "Pasteboard" to `capi-pasteboard`, the NDK C API, while the zip imports the
  ArkTS module (`Index.ets:3`, `import { BusinessError, pasteboard } from
  '@kit.BasicServicesKit'`). The same wrong link appears in the shopping
  `address_manager` document, so it is a copy-pasted reference block rather than
  a one-off. Fix: link `@ohos.pasteboard` (`js-apis-pasteboard`) in both places.

## References

- `huawei_industry_tree/11_news_reading/docs/19_erase_recognize.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/erase_recognize-0000002313854866
- `documentation/harmonyos-references/07_ai/core-vision-text-recognition-api.md` - `recognizeText`, `VisionInfo`, `TextRecognitionConfiguration`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/core-vision-text-recognition-api
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - `Canvas`, `onReady`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/04_media/arkts-apis-image-pixelmap.md` - `crop`, and that it mutates in place
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-pixelmap
- `documentation/harmonyos-references/03_system/js-apis-pasteboard.md` - the ArkTS pasteboard the sample actually uses (the reference `HW-11-0021` asks for)
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-pasteboard
- `documentation/harmonyos-guides/04_system/use-pasteboard-to-copy-and-paste.md` - the copy/paste flow and the read-back notification
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/use-pasteboard-to-copy-and-paste
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-image.md` - `onComplete` and the meaning of `width` vs `contentWidth`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-image
