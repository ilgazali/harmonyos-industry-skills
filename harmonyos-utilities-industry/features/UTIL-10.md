---
id: UTIL-10
title: Banner-driven background - pull a colour out of the current image with effectKit and animate the gradient
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/10_change_color_by_picture.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/change_color_by_picture-0000002320644189
sample: huawei_industry_tree/15_utilities/downloads/ColorChangeByPicture.zip
kits: ["@kit.AbilityKit", "@kit.ArkGraphics2D", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.LocalizationKit", "@kit.PerformanceAnalysisKit"]
apis: [effectKit, hilog, image, resourceManager, window]
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0028, HW-15-0029, HW-15-0030, HW-15-0101]
status: verified-with-fixes
---

## When to use

**Load this card when a page's chrome should take its colour from the
content**, not from a fixed theme: a banner carousel whose backdrop bleeds the
current picture's colour, an album header, a product page that tints itself
per item, a music player that follows the cover art.

The mechanism is four calls deep and always the same:
`resourceManager.getMediaContentSync` for the bytes,
`image.createImageSource` + `createPixelMap` for a `PixelMap`,
`effectKit.createColorPicker` for a picker, and `getMainColorSync()` (or
`getAverageColor()`) for the colour. Then the value goes into a `@State`
string that a `linearGradient` reads, and the assignment is wrapped in
`animateTo` so the change is a cross-fade rather than a jump.

It generalises to any image source - a `PixelMap` decoded from a network
buffer works identically - and to any colour-carrying attribute, not just
gradients: status-bar tint, a button's accent, a shadow colour.
**Read `HW-15-0028` before shipping the hex conversion**; the sample's
string-building is wrong for roughly half of all real colours.

## Feature checklist

- A four-image `Swiper` banner at the top of the page, looping, no autoplay.
- The area behind the banner is a vertical gradient from the current image's
  colour to a fixed base colour.
- Swiping to another banner animates the gradient to the new image's colour
  over 100 ms.
- The colour is also computed on `onPageShow`, so returning to the page
  restores the right backdrop.
- A dot indicator under the banner tracks the current page.
- Below that, a bottom `Tabs` bar with a home tab carrying a 2x3 grid of
  shortcut cards; the other two tabs are empty placeholders.

## Architecture

One `entry` module, five files.

```
entry/src/main/ets
├── constants/StyleConstants.ets       numeric constants + GRID_DATA / TABINFO models
├── entryability/EntryAbility.ets      full-screen window, avoid areas -> AppStorage (in vp)
├── entrybackupability/EntryBackupAbility.ets
└── pages
   ├── Index.ets                       @Entry: Swiper + indicator + Tabs, and getAverageColor
   └── HomePage.ets                    the home tab's shortcut grid
```

The documented 工程目录 lists exactly these files and matches the zip.

**The design decision worth copying** is that the gradient lives on the `Row`
that wraps the `Swiper`, not on the images. The `Swiper` itself is
transparent, its `Image` children are padded away from the edges, and the
extracted colour paints the container behind them:

```typescript
Row() { Swiper(...) { ... } }
  .linearGradient({
    direction: GradientDirection.Bottom,
    colors: [[this.basicColor, 0.0], [$r('app.color.linearGradient_color'), 1.0]]
  })
```

One `@State string` drives it. Nothing about the `Swiper`, its pages or its
transition needs to know a colour exists, and the same `basicColor` could
feed a status-bar call or a card border with no restructuring. The stop list
puts the extracted colour at offset `0.0` and a fixed colour at `1.0`, so the
image colour only ever occupies the top of the band - which is what keeps
text below it readable no matter what the image turns out to be.

**The part worth avoiding** is the second half of `Index.ets`: the indicator.
It is a detached `IndicatorComponent` bound to the `Swiper` by controller,
which is the modern and correct wiring, but every number fed into it is wrong
(`HW-15-0029`) - eight dots for four pages, `TWELVE` defined as `8`, and
`vertical(true)` under a horizontal swiper. Take the controller pattern, not
the constants.

## Implementation steps

1. **Hold the images as `Array<Resource>`** and derive the resource id with
   `this.imgData[index].id` - `getMediaContentSync` takes an id, not a
   `$r(...)`.
2. **Decode to a `PixelMap`**: `getMediaContentSync` -> `.buffer` ->
   `image.createImageSource` -> `await createPixelMap()`.
3. **Create the picker with the callback form**
   `effectKit.createColorPicker(pixelMap, (err, colorPicker) => ...)`, and
   handle `err` - the sample names it `_err` and ignores it.
4. **Pick main vs average deliberately.** `getMainColorSync()` returns the
   dominant colour; `getAverageColor()` returns the mean. The sample's method
   is called `getAverageColor` and the doc promises the average, but the code
   calls `getMainColorSync` (`HW-15-0030`).
5. **Build the hex string with two digits per channel** using
   `padStart(2, '0')` (`HW-15-0028`).
6. **Assign inside `animateTo`** so the gradient interpolates instead of
   snapping.
7. **Recompute on both entry points**: `onPageShow` for the initial and
   returning case, `Swiper.onChange` for every swipe.
8. **Drive the indicator from the data**: `count(this.imgData.length)` and
   `vertical(false)`, and give `selectedItemWidth` a value larger than
   `itemWidth` (`HW-15-0029`).
9. **Release what you decoded** - neither the `ImageSource` nor the `PixelMap`
   is released in the sample; call `pixelMap.release()` and
   `imageSource.release()` once the colour is read.

## Verified snippets

All snippets are from `ColorChangeByPicture.zip`. Corrected forms are marked.

**Colour extraction — `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-15-0028`, `HW-15-0030`)

```typescript
import { image } from '@kit.ImageKit';
import { effectKit } from '@kit.ArkGraphics2D';
import { resourceManager } from '@kit.LocalizationKit';

private async getAverageColor(targetIndex: number) {
  const context = this.getUIContext().getHostContext() as Context;
  // 获取resourceManager资源管理器
  const resourceMgr: resourceManager.ResourceManager = context.resourceManager;
  const fileData: Uint8Array = resourceMgr.getMediaContentSync(targetIndex);

  // 获取图片的ArrayBuffer
  const buffer = fileData.buffer;
  // 创建imageSource
  const imageSource: image.ImageSource = image.createImageSource(buffer);
  // 创建pixelMap
  const pixelMap: image.PixelMap = await imageSource.createPixelMap();

  effectKit.createColorPicker(pixelMap, (err, colorPicker) => {
    if (err) {                                        // FIX: the sample names this `_err` and drops it
      hilog.error(0, StyleConstants.GET_COLOR, `createColorPicker failed: ${err.message}`);
      return;
    }
    // 读取图像主色的颜色值，结果写入Color
    let color = colorPicker.getMainColorSync();
    this.getUIContext().animateTo({ duration: 100, curve: Curve.Linear, iterations: 1 }, () => {
      // 将取色器选取的color示例转换为十六进制颜色代码
      this.basicColor = '#' +
        color.alpha.toString(16).padStart(2, '0') +    // FIX: sample concatenates unpadded digits
        color.red.toString(16).padStart(2, '0') +
        color.green.toString(16).padStart(2, '0') +
        color.blue.toString(16).padStart(2, '0');
    });
    pixelMap.release();                               // FIX: absent in the sample
    imageSource.release();
  });
}
```

**The padding is not cosmetic.** `Color` channels are 0-255; any channel
below 16 stringifies to a single hex digit, so a colour like
`(255, 10, 200, 30)` becomes `#ffac81e` - seven characters, parsed as some
entirely different colour, or rejected. Roughly one image in two hits it.
`padStart(2, '0')` per channel is the whole fix.

**The `animateTo` wraps the assignment, not the extraction.** The decode and
the picker call are async and slow; only the state write belongs inside the
closure. 100 ms with `Curve.Linear` is deliberately shorter than the swiper's
own transition, so the colour has already settled by the time the new image
stops moving.

`getMediaContentSync(targetIndex)` takes the numeric resource id from
`$r('app.media.bannerN').id`, which is why `imgData` is typed
`Array<Resource>` rather than as strings - the `.id` field is the only reason
the resources are held as objects.

**The gradient host and the swiper — same file** (as shipped)

```typescript
Row() {
  Swiper(this.swiperController) {
    ForEach(this.imgData, (item: string) => {
      Image(item)
        .padding({ top: this.topRectHeight, left: 16, right: 16 })
        .objectFit(ImageFit.Cover)
        .borderRadius(StyleConstants.PICTURE_BORDER_RADIUS);
    });
  }
  .autoPlay(false)                      // 关闭自动播放（根据点击事件切换）
  .loop(true)
  .indicator(this.indicatorController)  // detached indicator, bound by controller
  .duration(StyleConstants.SWIPER_DURATION)
  .vertical(false)
  .onChange((index: number) => {
    this.currentIndex = index;
    this.currentId = this.imgData[this.currentIndex].id;
    this.getAverageColor(this.currentId);
  });
}
.height(StyleConstants.IMG_HEIGHT)
.linearGradient({
  direction: GradientDirection.Bottom,
  colors: [[this.basicColor, 0.0], [$r('app.color.linearGradient_color'), 1.0]],
});
```

**`.indicator(this.indicatorController)` is the piece to copy.** Passing an
`IndicatorComponentController` instead of a `DotIndicator` detaches the dots
from the swiper's own bounds, so they can be placed anywhere in the layout -
here below the gradient band rather than on top of the image. The swiper and
the `IndicatorComponent` stay in sync through the shared controller with no
index plumbing.

Two details bite. The `ForEach` item is typed `string` while `imgData` is
`Array<Resource>`; it compiles because `Image` accepts both, but
`this.imgData[i].id` in `onChange` is the real contract. And `topRectHeight`
is applied as raw `padding.top` - correct here only because this sample's
`EntryAbility` already converts the avoid area with `px2vp` before storing it
(sibling samples in this industry store px and convert at the point of use;
mixing the two conventions is how status-bar overlaps appear).

**The indicator — same file** (corrected, see `HW-15-0029`)

```typescript
IndicatorComponent(this.indicatorController)
  .initialIndex(StyleConstants.ZERO)
  .style(
    new DotIndicator()
      .itemWidth(StyleConstants.EIGHT)            // 8
      .itemHeight(StyleConstants.EIGHT)
      .selectedItemWidth(StyleConstants.TWELVE)   // FIX: TWELVE is defined as 8 in StyleConstants
      .selectedItemHeight(StyleConstants.EIGHT)
      .color($r('app.color.circle_unselect_color'))
      .selectedColor($r('app.color.circle_select_color')))
  .loop(true)
  .count(this.imgData.length)                     // FIX: sample passes StyleConstants.EIGHT
  .vertical(false);                               // FIX: sample passes true under a horizontal Swiper
```

Three independent mistakes in ten lines, all of the same kind: a constant used
for its name rather than its value. `StyleConstants.TWELVE: number = 8` makes
the selected dot exactly as wide as the others, so the only remaining cue is
colour. `count(EIGHT)` renders eight dots for four pages, so half the strip is
permanently dead. `vertical(true)` stacks them in a column beside a
horizontally-scrolling swiper. The controller wiring above is right; only the
arguments are wrong.

## Permissions & config

**None.** The sample declares no `requestPermissions` - `resourceManager`
reads the app's own resources and `effectKit` works on an in-process
`PixelMap`.

`deviceTypes` is `phone`, `tablet`, `2in1`. `colorMode` is
`COLOR_MODE_NOT_SET`, so the page follows the system theme - worth noting
given that the extracted colour is applied on top of a fixed
`app.color.linearGradient_color` stop that is not theme-aware.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `getMediaContentSync` is synchronous and decodes the whole media file on the
  UI thread; for large banners or a long list, prefer the async
  `getMediaContent` and cache the extracted colour per image id rather than
  recomputing on every swipe.
- `effectKit.createColorPicker` needs a `PixelMap` in a supported pixel
  format; a `PixelMap` obtained from a video frame or an unusual format may
  fail through the `err` argument the sample ignores.
- The colour has no contrast guarantee. A near-white banner produces a
  near-white gradient behind whatever sits under it; if text overlaps the
  band, clamp the luminance before use.
- The message and mine tabs are empty `TabContent()` blocks - this is a colour
  sample, not a shell.

## Pitfalls

- **`HW-15-0028`** (B/medium, confirmed): the hex string is assembled from raw
  `toString(16)` per channel, so any channel below 16 contributes one digit
  and the result is a 5-8 character string that parses as a different colour -
  the sample's core effect is wrong for a large share of images.
  Fix: `padStart(2, '0')` on each component.
- **`HW-15-0029`** (B/low, confirmed): the indicator is misconfigured three
  ways - `count(StyleConstants.EIGHT)` against four pages,
  `TWELVE: number = 8` feeding `selectedItemWidth` so the selected dot has no
  emphasis, and `.vertical(true)` over a horizontal `Swiper`.
  Fix: `count(this.imgData.length)`, correct `TWELVE`, `vertical(false)`.
- **`HW-15-0030`** (E/low, confirmed): the doc says the sample takes the
  picture's average colour (通过@ohos.effectKit获取图片平均颜色) and the
  method is even named `getAverageColor`, but the code calls
  `getMainColorSync()`, which returns the dominant colour. `effectKit` has a
  real `getAverageColor()` that is never used.
  Fix: call `getAverageColor()`, or correct the doc and rename the method.

## References

- `documentation/harmonyos-references/05_graphics/js-apis-effectkit.md` - `createColorPicker`, `ColorPicker.getMainColorSync`, `getAverageColor`, the `Color` struct
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-effectkit
- `documentation/harmonyos-references/02_application-framework/ts-container-swiper.md` - `Swiper`, `indicator`, `onChange`, `loop`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-swiper
- `documentation/harmonyos-references/02_application-framework/ts-swiper-components-indicator.md` - `IndicatorComponent`, `IndicatorComponentController`, `DotIndicator`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-swiper-components-indicator
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-gradient-color.md` - `linearGradient`, `GradientDirection`, colour stops
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-gradient-color
- `documentation/harmonyos-references/02_application-framework/ts-explicit-animation.md` - `animateTo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-explicit-animation
- `huawei_industry_tree/15_utilities/docs/10_change_color_by_picture.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/change_color_by_picture-0000002320644189
