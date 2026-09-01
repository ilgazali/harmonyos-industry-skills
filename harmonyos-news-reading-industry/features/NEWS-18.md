---
id: NEWS-18
title: Reading magnifier - a clipped component snapshot panned under a long-press-then-drag gesture
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/18_magnifier.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/magnifier-0000002296851262
sample: huawei_industry_tree/11_news_reading/downloads/Magnifier.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: ["UIContext.getComponentSnapshot", "componentSnapshot.get", Stack, GestureGroup, GestureMode, LongPressGesture, PanGesture, "UIContext.px2vp", translate, clip, borderRadius, ImageFit, copyOption, CopyOptions, "image.PixelMap", "PixelMap.getImageInfoSync"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-11-0020, HW-11-0031]
status: verified-with-fixes
---

## When to use

Load this card when you need a **loupe over your own UI** - a floating circle
that shows an enlarged copy of whatever the finger is over. Accessibility
reading aids are the obvious case; the same construction serves a colour picker
over a canvas, a precision crop handle, or a map detail lens.

The technique is worth understanding even if you never ship a magnifier,
because it is the general answer to "show a zoomed copy of a live component".
You do not re-render the component at a larger scale. You take one **snapshot**
of it as a `PixelMap` when the gesture begins, draw that snapshot at its natural
enlarged size inside a small clipped container, and then *pan the image* under
the clip as the finger moves. One rasterisation, then pure translation - so the
drag stays at frame rate no matter how expensive the underlying text layout is.

The gesture half is equally transferable: `GestureGroup(GestureMode.Sequence,
LongPressGesture, PanGesture)` is the canonical "press to arm, then drag to
operate" pattern. Reach for it whenever a drag must not be confused with a
scroll.

## Feature checklist

- A two-chapter reader page: title, subtitle, chapter heading, body text, and a
  hidden bottom navigation bar.
- A magnifier button in the header toggles the feature on and off by switching
  the body text's `copyOption` between `CopyOptions.InApp` and `CopyOptions.None`.
- With the magnifier armed, a long press on the body raises a round lens showing
  the text under the finger at 1.2x.
- Dragging without lifting moves the lens and the magnified content together.
- Lifting the finger, or the gesture being cancelled, hides the lens.
- Tapping the body toggles the bottom chapter bar, which slides in and out over
  700 ms; while that bar is visible the lens is suppressed.
- Previous/next chapter buttons grey out at the ends of the two-chapter list.

## Architecture

One `entry` module, one page and one view. There is no model layer - the two
chapters are a literal array of `Resource` pairs.

```
entry/src/main/ets
├── entryability/EntryAbility.ets           program entry
├── entrybackupability/EntryBackupAbility.ets
├── pages/DetailPage.ets                    @Entry: the reader, the lens, the gesture group
└── views/BottomView.ets                    the chapter bar that slides in from below
```

The documented tree does **not** match the zip: the doc's 工程目录 names the
directory `view`, the zip ships `views` (`HW-11-0020`). The import in
`DetailPage.ets` is `from '../views/BottomView'`, so the zip is authoritative.

**The design decision worth copying** is the choice of *what* to snapshot. The
body `Text` carries `.id('targetComponent')`, and the lens calls
`getComponentSnapshot().get('targetComponent', callback, { scale: 1.2,
waitUntilRenderFinished: true })`. Snapshotting the *content* component rather
than the whole page means the lens never contains the header, the toolbar or its
own previous frame - which is exactly the recursion trap you fall into if you
snapshot the page root while the lens is drawn on top of it. Giving the content
an `id` and magnifying only that id is the load-bearing structural choice.

The `scale: 1.2` option is the second half of the same decision: the
magnification is baked into the raster, not applied as a UI transform
afterwards. `waitUntilRenderFinished: true` guarantees the pixels are the
committed frame, not a half-laid-out one.

**The decision worth avoiding** is the lens geometry. Five numeric literals -
`158`, `79`, `165`, `332` and `1.2` - encode the lens size, its offset from the
finger and the magnification, and only two of them are related to each other in
the source (`79` is both half of `158` and the `borderRadius`). Change the lens
diameter and you must retune three unrelated constants by eye. `HW-11-0020`
records this; the corrected snippet below shows the derivation.

## Implementation steps

1. **Give the magnifiable content an `id`.** `.id('targetComponent')` on the body
   `Text` is what `componentSnapshot.get` resolves.
2. **Arm the feature by disabling text selection.** The lens is drawn only when
   `copyStyle === CopyOptions.None`; the header button flips it. With
   `CopyOptions.InApp` active a long press belongs to the system text-selection
   handles, and the two would fight over the same gesture.
3. **Compose a sequence gesture**, `GestureGroup(GestureMode.Sequence,
   LongPressGesture({ repeat: false }), PanGesture())`, on the `Stack` that
   contains the text. Sequence means the pan is only recognised after the long
   press has succeeded, so an ordinary swipe still scrolls.
4. **Take the snapshot once, in `onAction` of the long press,** with `scale`
   set to the magnification and `waitUntilRenderFinished: true`. Do not
   re-snapshot on every pan update.
5. **Record `getImageInfoSync().size`** and display the `Image` at
   `px2vp(width) x px2vp(height)` - the snapshot's own natural size, already
   enlarged by `scale`.
6. **Set `objectFit(ImageFit.None)`.** This is not decorative. Without it the
   default fit rescales the snapshot to the 158 vp lens box and the translation
   pans a shrunk image over itself. The document's snippet omits this line
   (`HW-11-0020`).
7. **Clip the lens**: a `Row` of fixed size with `borderRadius(radius)` and
   `clip(true)`. The circle is the clip; the image inside is square.
8. **Derive the offsets from the lens size and the scale,** not from tuned
   literals (`HW-11-0020`). The corrected form is
   `translate = { x: radius - scale * posX, y: radius - scale * posY }` with the
   lens positioned at `margin: { left: posX - radius, top: posY - lift }`.
9. **Clear the flag on both exits.** `PanGesture.onActionEnd` covers the normal
   lift; `GestureGroup.onCancel` covers an interrupted gesture. Missing either
   leaves the lens stuck on screen.

## Verified snippets

All snippets are from `Magnifier.zip`,
`entry/src/main/ets/pages/DetailPage.ets`. Corrected forms are marked.

**The lens** (corrected - see `HW-11-0020`)

```typescript
private static readonly LENS_SIZE: number = 158;
private static readonly LENS_RADIUS: number = DetailPage.LENS_SIZE / 2;   // 79
private static readonly LENS_SCALE: number = 1.2;
private static readonly LENS_LIFT: number = 165;   // how far above the finger the lens sits

if (this.isTouch && this.copyStyle === CopyOptions.None && !this.isMenuViewVisible) {
  Row() {
    Image(this.pixmap)
      .height(this.uiContext.px2vp(this.pixmapH))
      .width(this.uiContext.px2vp(this.pixmapW))
      .objectFit(ImageFit.None)                      // draw at natural size so translate can pan it
      .translate({
        // FIX: shipped code is (79 - posX) * 1.2, which scales the radius too
        x: DetailPage.LENS_RADIUS - DetailPage.LENS_SCALE * this.posX,
        y: DetailPage.LENS_RADIUS - DetailPage.LENS_SCALE * this.posY
      });
  }
  .backgroundColor($r('app.color.pageflip_swiper_backgroundcolor'))
  .margin({
    left: this.posX - DetailPage.LENS_RADIUS,
    top: this.posY - DetailPage.LENS_LIFT
  })
  .shadow({ radius: 12, color: '#26000000' })
  .borderRadius(DetailPage.LENS_RADIUS)
  .width(DetailPage.LENS_SIZE)
  .height(DetailPage.LENS_SIZE)
  .clip(true);
}
```

**Three attributes make this a lens rather than a thumbnail.** `objectFit(
ImageFit.None)` renders the snapshot at its declared `width`/`height` - the full
magnified page - instead of shrinking it into the 158 vp box; `clip(true)`
together with `borderRadius(79)` turns the square `Row` into the round aperture;
and `translate` slides the oversized image behind that aperture. The `Row` never
moves relative to its own content coordinate system, so no layout pass happens
during the drag - only a transform.

**The correction.** To put content point `posX` under the centre of the lens you
need `scale * posX + translateX = radius`, so `translateX = radius - scale *
posX`. The shipped code writes `(79 - this.posX) * 1.2`, which expands to
`94.8 - 1.2 * posX` - the radius has been multiplied by the scale factor as
well, leaving a fixed 15.8 vp offset between where the finger is and what the
lens shows. The vertical term has the same shape with `332` in place of `79`,
adding another 66.4 vp of constant error on top of whatever `332` was tuned to
mean. Both errors are invisible on the demo's screen because `165` and `332`
were eyeballed against them; both reappear the moment you change the lens size,
the magnification, or the page padding. That is exactly `HW-11-0020`'s point:
the offsets belong in terms of `LENS_RADIUS` and `LENS_SCALE`.

Note the third condition on the `if`: `!this.isMenuViewVisible`. The document's
snippet drops it, along with `objectFit`. Copying the document rather than the
zip yields a lens that stays visible over the chapter bar and renders a shrunk,
un-pannable snapshot.

**Snapshot on press, pan on drag** (as shipped)

```typescript
.gesture(
  // 声明该组合手势的类型为 Sequence 类型
  GestureGroup(GestureMode.Sequence,
    // 该组合手势第一个触发的手势为长按手势
    LongPressGesture({ repeat: false })
      .onAction((event: GestureEvent | undefined) => {
        if (event) {
          this.isTouch = true;
          this.posX = event.fingerList[0].localX;
          this.posY = event.fingerList[0].localY;
          this.uiContext.getComponentSnapshot().get('targetComponent',
            (error: Error, pixmap: image.PixelMap) => {
              if (error) {
                return;
              }
              this.pixmap = pixmap;
              this.pixmapH = this.pixmap.getImageInfoSync().size.height;
              this.pixmapW = this.pixmap.getImageInfoSync().size.width;
            }, { scale: 1.2, waitUntilRenderFinished: true });
        }
      })
    ,
    // 当长按之后进行拖动，PanGesture 手势被触发
    PanGesture()
      .onActionUpdate((event: GestureEvent | undefined) => {
        if (event) {
          this.posX = event.fingerList[0].localX;
          this.posY = event.fingerList[0].localY;
        }
      })
      .onActionEnd(() => {
        this.isTouch = false;
      })
  )
    .onCancel(() => {
      this.isTouch = false;
    })
)
```

**`GestureMode.Sequence` is the whole reason this coexists with scrolling.** In
a sequence group the second gesture is only offered the touch stream after the
first has been recognised, so a plain drag on the article is never captured by
the pan handler - it falls through to whatever container wants to scroll. The
long press is the arming step, and `repeat: false` means `onAction` fires
exactly once per press, which matters because that callback is where the
snapshot is taken.

**The snapshot is taken once and reused for the entire drag.** `onActionUpdate`
writes only two numbers. That is the performance property that makes the
magnifier viable: rasterising a full page of laid-out text every frame would not
hold 60 fps, but translating an existing `PixelMap` is a compositor operation.
The trade-off is that the lens shows a frozen copy - correct here, since the
article cannot change mid-press, and wrong if you point this at animating
content.

`fingerList[0].localX` is coordinates local to the component the gesture is bound
to, which is the `Stack` wrapping the text - the same coordinate space the lens's
`margin` is resolved in. Mixing `localX` here with a `margin` on a differently
positioned parent is the usual way this pattern ends up off by the header height.

**Arming by turning selection off** (as shipped)

```typescript
Image($r('app.media.magnifier'))
  .width(40)
  .onClick(() => {
    if (this.copyStyle === CopyOptions.None) {
      this.copyStyle = CopyOptions.InApp;
    } else {
      this.copyStyle = CopyOptions.None;
    }
  });

// ...
Text(this.bookList[this.bookmark].bookBody)
  .copyOption(this.copyStyle)
  .id('targetComponent')
```

**One piece of state serves two purposes, deliberately.** `copyStyle` is both
the text component's selection mode and the magnifier's armed flag. That is not
a shortcut - the two features genuinely cannot coexist, because
`CopyOptions.InApp` makes the system claim long press for its selection handles.
Encoding the exclusion in a single value means there is no state in which both
are on.

`.id('targetComponent')` is the anchor for the snapshot. It must be on the
component you want magnified, and it must be unique in the page.

## Permissions & config

**None.** The sample declares no `requestPermissions`. `deviceTypes` is
`phone`, `tablet`, `2in1`; `resources` carries `base` plus a `dark` colour
override.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The lens geometry is tuned for one layout. `165` and `332` have no derivation
  in the source and no relationship to `158`/`79`; on a tablet, a resized 2in1
  window, or with a different body font size the lens points at the wrong text
  (`HW-11-0020`).
- The `PixelMap` produced by each long press is assigned to `this.pixmap` and
  never released. A long reading session with many presses accumulates full-page
  rasters at 1.2x. Call `release()` on the previous map before replacing it.
- The lens shows a frozen snapshot. Any content that animates or updates during
  the press will be stale inside the lens.
- `BottomView` is hardcoded to two chapters: `if (this.bookmark < 1)` bounds the
  next-chapter button and the greyed-out artwork is chosen by
  `!this.bookmark` / `this.bookmark`. Compare `NEWS-20`'s `BottomView`, which
  takes a `bookLength` and is the version to copy.
- The chapter bar is hidden by `margin({ bottom: -90 })` rather than by removing
  it from the tree, so it still occupies its slot in the layout and still
  receives no touches only because it is off-screen.

## Pitfalls

- **`HW-11-0020`** (E/low, confirmed): the document's 工程目录 lists a `view`
  directory while the zip uses `views`, and the documented magnifier snippet
  positions the lens with hardcoded literals (`79`, `332`, `165`, factor `1.2`)
  annotated only as 尺寸参数. The offsets are pixel constants valid for one
  screen layout, and the shipped `(79 - posX) * 1.2` form multiplies the lens
  radius by the scale factor as well. Fix: rename the tree entry to `views`, and
  derive the offsets from the lens radius and the magnification -
  `radius - scale * pos` - instead of literals.
- **The document's snippet is not the shipped code.** It omits
  `objectFit(ImageFit.None)` and the `!this.isMenuViewVisible` guard. A reader
  who copies the document gets a lens that scales the snapshot into the aperture
  - so panning it does nothing visible - and that draws over the chapter bar.

## References

- `huawei_industry_tree/11_news_reading/docs/18_magnifier.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/magnifier-0000002296851262
- `documentation/harmonyos-references/02_application-framework/ts-container-stack.md` - `Stack` and `alignContent`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-stack
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-componentsnapshot.md` - `getComponentSnapshot().get`, the `scale` and `waitUntilRenderFinished` options
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-componentsnapshot
- `documentation/harmonyos-references/02_application-framework/ts-gesture-settings.md` - `GestureGroup`, `GestureMode.Sequence`, `onCancel`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-gesture-settings
- `documentation/harmonyos-guides/03_application-framework/arkts-gesture-events-combined-gestures.md` - the sequence-gesture recognition rules
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-gesture-events-combined-gestures
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-image.md` - `ImageFit.None`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-image
- `NEWS-20` - the same reader shell with a chapter-count-driven `BottomView`
