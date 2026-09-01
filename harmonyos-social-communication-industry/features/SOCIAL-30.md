---
id: SOCIAL-30
title: Inertial long-image scrolling - turn PanGesture release velocity into a UIContext animator
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/30_inertial_sliding.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/inertial_sliding-0000002308946264
sample: huawei_industry_tree/14_social_communication/downloads/LongImageSlideDemo.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.LocalizationKit", "@kit.PerformanceAnalysisKit"]
apis: [PanGesture, onActionStart, onActionUpdate, onActionEnd, "GestureEvent.velocityY", "GestureEvent.offsetY", "UIContext.createAnimator", AnimatorResult, onFrame, onSizeChange, onTouch, translate, "image.createImageSource", createPixelMap, getImageInfo, "resourceManager.getRawFileContent", "display.getDefaultDisplaySync", "@Observed"]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0066]
status: verified-with-fixes
---

## When to use

Load this card when a view must **pan under the finger and keep moving after
release** - a long screenshot in a chat, a tall poster in a feed, a map-like
canvas - and `Scroll` or `List` is not usable because the content is a single
image you are transforming rather than a stack of items.

The pattern is three pieces: a `PanGesture` that writes a translation offset, a
release handler that converts `velocityY` into a target offset, and a
`UIContext.createAnimator` that walks from the current offset to that target on
an `ease-out` curve, writing every frame back into state. Nothing here is
image-specific. The same three pieces give inertia to a custom carousel, a
drag-to-dismiss sheet, or a pan-and-fling chart.

The physics is one constant: `endTranslateY = current + velocityY / 4`. That is
the whole "friction" model - a divisor, not a decay simulation. It is enough
because the animator's `ease-out` curve supplies the deceleration; the divisor
only decides how far a fling travels. Tune that number, not the curve, when the
scroll feels too slippery or too sticky.

## Feature checklist

- A long image loads from `rawfile` and is scaled to the screen width.
- The image is anchored to the top of the view; only the vertical offset moves.
- Dragging moves the image one-to-one with the finger.
- Releasing continues the motion in the same direction, decelerating to a stop
  within 500 ms.
- The travel distance scales with the release velocity.
- Touching the image during the coast stops it immediately at that point.
- Starting a new drag during the coast picks up from the current position, not
  from where the coast would have ended.
- The image never scrolls past its top edge or above its bottom edge.

## Architecture

One `entry` module, seven files, and a clean three-layer split that is unusual
for a sample this size.

```
entry/src/main/ets
├── common/Constants.ets              FULL_HEIGHT/FULL_WIDTH, DURATION = 500, TAG
├── common/ImageScrollParam.ets       gesture-time scratch: start offset, min/max bounds, VELOCITY_SCROLL_FRICTION
├── common/ImageViewState.ets         @Observed render state: pixelMap, width, height, translateY
├── controller/ImageViewController.ets  all the logic; exported as a module-level singleton
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── pages/MainPage.ets                @Entry, a Stack holding one ImageView
└── view/ImageView.ets                the Image, the gesture, and three event bindings - 56 lines
```

The documented tree is close but wrong in two places: it writes the page
directory as `page` (the zip has `pages`) and misspells
`entrybackupablility`. Nothing else differs.

**The design decision worth copying** is the two-class state split.
`ImageViewState` is `@Observed` and holds only what the `build()` reads -
`pixelMap`, `width`, `height`, `translateY`. `ImageScrollParam` is a plain
class holding what only the gesture needs - the offset captured at
`onActionStart`, and the clamp bounds. The view subscribes to the first and
never sees the second, so a pan updates exactly one observable field per frame
and re-renders exactly one `translate`.

The controller is a **module-level singleton**:

```typescript
export const imageViewController: ImageViewController = new ImageViewController();
```

That is the part to copy carefully rather than blindly. It keeps the view down
to 56 lines of pure layout, and it is fine here because the app shows one
image. Two `ImageView`s on one page would share one `translateY` and fight.
Make it an instance owned by the component (`@State controller = new ...`) the
moment a second instance is plausible.

## Implementation steps

1. **Decode from `rawfile` into a `PixelMap`**: `getRawFileContent` →
   `createImageSource(buffer)` → `createPixelMap()` → `getImageInfo()`.
   `imageData.buffer.slice(0)` is needed to get a standalone `ArrayBuffer`.
2. **Scale to the screen width yourself.** Take
   `display.getDefaultDisplaySync().width` in px, derive the height from the
   image's aspect ratio, and convert both with `px2vp` before writing them into
   state. Do not let `Image` fit the box - the layout height is what the clamp
   arithmetic is built on.
3. **Anchor the `Stack` with `.align(Alignment.Top)`** so translation 0 means
   "top of the image at the top of the view", which makes `maxTranslateY = 0`
   the natural upper bound.
4. **On `onActionStart`, snapshot the current offset** into `ImageScrollParam`
   and `pause()` any running animator - this is what makes a grab during the
   coast pick up from where the image actually is.
5. **On `onActionUpdate`, write `snapshot + event.offsetY`**, clamped. Pan
   offsets are cumulative from the gesture start, so they must be added to the
   snapshot, never to the live value.
6. **On `onActionEnd`, convert velocity to a destination** and animate to it
   with `createAnimator`, feeding `onFrame` back through the same clamped
   setter.
7. **Clamp the minimum against zero** when the view is taller than the image,
   or the bounds invert and the first pan shoves the image down
   (`HW-14-0066`).
8. **Bind `onTouch` as well as the gesture** and pause the animator on
   `TouchType.Down`: a tap that never becomes a pan must still stop the coast,
   and `PanGesture` will not fire for it.

## Verified snippets

All snippets are from `LongImageSlideDemo.zip`. Corrected forms are marked.

**The two state classes — `entry/src/main/ets/common/`** (as shipped)

```typescript
// ImageViewState.ets - what the view renders
@Observed
export class ImageViewState {
  pixelMap: image.PixelMap | ImageContent.EMPTY = ImageContent.EMPTY; // 像素图
  width: number = 0;
  height: number = 0;
  translateY: number = 0;                       // the only field that changes per frame
}

// ImageScrollParam.ets - what only the gesture needs
export class ImageScrollParam {
  static readonly VELOCITY_SCROLL_FRICTION = 4; // 速度转换因子
  translateY: number = 0;                       // offset captured when the pan starts
  maxTranslateY: number = 0;                    // top edge
  minTranslateY: number = 0;                    // bottom edge, negative
}
```

`ImageContent.EMPTY` as the initial `pixelMap` is worth noting: `Image` accepts
it as a valid "nothing yet" source, so the view needs no `if (loaded)` branch
and no placeholder resource. `maxTranslateY` stays 0 for the lifetime of the
app - the image is anchored to the top and can only ever be dragged upward -
so only the minimum is ever recomputed.

**Decoding and sizing — `entry/src/main/ets/controller/ImageViewController.ets`** (as shipped)

```typescript
public loadPixelMapFromRawFile(context: UIContext, fileName: string) {
  let hostContext = context.getHostContext();
  if (!hostContext) {
    return;
  }
  let resourceManager: resourceManager.ResourceManager = hostContext.resourceManager;
  resourceManager.getRawFileContent(fileName).then((imageData: Uint8Array) => {
    let buffer = imageData.buffer.slice(0);
    let imageSource: image.ImageSource = image.createImageSource(buffer);
    imageSource.createPixelMap().then((pixelMap: image.PixelMap) => {
      pixelMap.getImageInfo().then((info: image.ImageInfo) => {
        this.imageViewState.pixelMap = pixelMap;
        let screenWidth = display.getDefaultDisplaySync().width;
        let imageHeight = screenWidth / info.size.width * info.size.height;
        // 将像素图大小按屏幕宽度调整
        this.imageViewState.width = context.px2vp(screenWidth);
        this.imageViewState.height = context.px2vp(imageHeight);
      });
    });
  }).catch((err: BusinessError) => {
    hilog.error(0x0000, Constants.TAG,
      `Failed to getRawFileContent. error code: ${err.code}, error message:${err.message}`);
  });
}
```

**The sizing is the point, not the decoding.** `display.getDefaultDisplaySync()`
returns px; `info.size` is in px; the ratio gives the displayed height in px;
`px2vp` converts both to the vp the layout works in. Writing an explicit
`width`/`height` onto the `Image` - rather than letting `ImageFit` do it -
is what makes `viewHeight - imageHeight` a meaningful scroll range later.

Note the nesting: three chained `then`s with a single `catch` on the outermost
promise. A failure inside `createPixelMap` or `getImageInfo` is unhandled, and
the app shows an empty view with no log line. Flattening this into one `async`
function with a single `try/catch` is the obvious improvement.

**Pan, release, animate — same file** (as shipped)

```typescript
// 滑动手势识别成功监听函数
public panGestureOnActionStart() {
  this.imageScrollParam.translateY = this.imageViewState.translateY;  // snapshot
  this.translateYAnimator?.pause();                                   // stop any coast
}

// 滑动手势移动过程监听函数
public panGestureOnActionUpdate(event: GestureEvent) {
  let translateY = this.imageScrollParam.translateY + event.offsetY;
  this.setImageTranslateY(translateY);
}

// 滑动手势识别成功后,手指抬起监听函数
public panGestureOnActionEnd(context: UIContext, event: GestureEvent) {
  let endTranslateY = this.getFinalTranslateY(this.imageViewState.translateY +
    event.velocityY / ImageScrollParam.VELOCITY_SCROLL_FRICTION);
  this.translateYAnimator = this.createTranslateYAnimator(context, this.imageViewState.translateY, endTranslateY);
  this.translateYAnimator.onFrame = (value: number) => {
    this.setImageTranslateY(value);
  };
  this.translateYAnimator.play();
}

private createTranslateYAnimator(context: UIContext,
  startTranslateY: number, endTranslateY: number): AnimatorResult {
  return context.createAnimator({
    duration: Constants.DURATION,     // 500 ms, always
    easing: 'ease-out',
    delay: 0,
    fill: 'forwards',
    direction: 'normal',
    iterations: 1,
    begin: startTranslateY,
    end: endTranslateY
  });
}
```

**Three options carry the design.** `easing: 'ease-out'` is the deceleration -
fast at the start of the coast, asymptotically slow at the end, which is what
"inertia" looks like. `fill: 'forwards'` keeps the final value applied after
the animation completes, so the image does not snap back. And the destination
is passed through `getFinalTranslateY` **before** the animator is built, so the
animator interpolates toward a legal position instead of overshooting and
being clamped mid-flight - that is what keeps the coast smooth at the edges
instead of ending in a stutter.

`duration` is a constant 500 ms regardless of velocity, so a hard fling and a
gentle one take the same time and differ only in distance. That is a
simplification, not a bug, but it is the first thing to change if the motion
feels wrong: scale the duration with `Math.abs(velocityY)` and the gesture
starts to feel like a real scroller.

Note also that `pause()` is used rather than `cancel()`; the animator object is
replaced on the next release anyway, and `pause` leaves the applied value in
place.

**The bounds — same file** (corrected, see `HW-14-0066`)

```typescript
// 图片可视区域大小变化监听函数, 用于获取高度值计算y轴平移的最小值
public onImageVisibleAreaSizeChange(oldValue: SizeOptions, newValue: SizeOptions) {
  if (newValue.height) {
    // FIX: sample assigns (height - imageHeight) unclamped, which can be > maxTranslateY
    this.imageScrollParam.minTranslateY = Math.min(0, newValue.height as number - this.imageViewState.height);
  }
}

// 获取最终的y轴平移值,限制平移值范围
private getFinalTranslateY(translateY: number) {
  if (translateY > this.imageScrollParam.maxTranslateY) {
    translateY = this.imageScrollParam.maxTranslateY;
  } else if (translateY < this.imageScrollParam.minTranslateY) {
    translateY = this.imageScrollParam.minTranslateY;
  }
  return translateY;
}
```

**`getFinalTranslateY` assumes `min <= max` and does not check.** The clamp is
an `if/else if`: it first pulls the value down to `maxTranslateY` (0), then, in
the `else` branch only, pushes it up to `minTranslateY`. When
`minTranslateY` is positive - a short image, or, more likely, `onSizeChange`
firing before the async decode has written `imageViewState.height`, leaving it
at 0 - the two bounds invert and the first branch no longer protects anything.
The first pan is clamped *up* to a positive offset and the image jumps
downward off its anchor. `Math.min(0, ...)` restores the invariant.

The ordering hazard is worth internalising: `onSizeChange` fires from layout,
`imageViewState.height` is written from a promise chain, and nothing sequences
them. Any clamp derived from an async-loaded dimension needs a defined value
for "not loaded yet", and here that value is 0.

**Binding it to the view — `entry/src/main/ets/view/ImageView.ets`** (as shipped)

```typescript
@Component
export struct ImageView {
  imageSrc: string = '';
  @State viewState: ImageViewState = imageViewController.getImageViewState();

  aboutToAppear(): void {
    imageViewController.loadPixelMapFromRawFile(this.getUIContext(), this.imageSrc);
  }

  build() {
    Stack() {
      Image(this.viewState.pixelMap)
        .width(this.viewState.width)
        .height(this.viewState.height)
        .translate({ y: this.viewState.translateY });
    }
    .align(Alignment.Top)
    .gesture(
      PanGesture()
        .onActionStart(() => imageViewController.panGestureOnActionStart())
        .onActionUpdate((event: GestureEvent) => imageViewController.panGestureOnActionUpdate(event))
        .onActionEnd((event: GestureEvent) => imageViewController.panGestureOnActionEnd(this.getUIContext(), event))
    )
    .onSizeChange((oldValue: SizeOptions, newValue: SizeOptions) => {
      imageViewController.onImageVisibleAreaSizeChange(oldValue, newValue);
    })
    .onTouch((event: TouchEvent) => imageViewController.onTouch(event));
  }
}
```

`translate` rather than `offset` or `position` is the right choice: it is a
render-time transform, so moving the image does not invalidate layout and the
`Stack`'s measured height - the number the clamp depends on - stays constant.
`onTouch` sits alongside the gesture because a bare tap during the coast never
becomes a `PanGesture`; the controller's `onTouch` pauses the animator on
`TouchType.Down`, which is the "tap to stop scrolling" behaviour every scroller
has.

## Permissions & config

**None.** The image is a `rawfile` inside the package, read through
`resourceManager` - no media-library access, no storage permission.

`deviceTypes` is `phone`, `tablet`, `2in1`. There is no `routerMap`;
`main_pages.json` points straight at `pages/MainPage`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The controller is a module-level singleton, so exactly one `ImageView` may
  exist at a time. Two would share one `translateY`.
- Vertical only. `maxTranslateY` is a fixed 0 and there is no horizontal axis,
  so a wide image cannot be panned sideways.
- No zoom, no double-tap, no edge bounce - `translate` is clamped hard at both
  ends, which stops dead rather than springing.
- `display.getDefaultDisplaySync().width` is the display width, not the window
  width, so in a split-screen or freeform window the image is scaled to the
  wrong width.
- Fling duration is a constant 500 ms; velocity only changes the distance.

## Pitfalls

- **`HW-14-0066`** (B/low, probable): `minTranslateY = viewHeight - imageHeight`
  is not clamped to `<= 0`, so with a short image - or with `onSizeChange`
  firing before the async decode sets `imageViewState.height` - the minimum
  exceeds the maximum and the `if/else if` clamp inverts, shoving the image
  downward off its top anchor on the first pan. Fix:
  `Math.min(0, viewHeight - imageHeight)`.
- Not a numbered finding, but adjacent: the decode chains three `then`s under
  a single outer `catch`, so a `createPixelMap` or `getImageInfo` failure is
  silent - the view stays empty with nothing in the log.
- Also not numbered: the doc's 工程目录 writes `page/MainPage.ets` (the zip has
  `pages/`) and misspells `entrybackupablility`.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - `PanGesture`, `offsetY`, `velocityY`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `createAnimator`, `AnimatorResult`, `onFrame`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-guides/03_application-framework/arkts-animator.md` - the animator lifecycle and `fill: 'forwards'`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-animator
- `documentation/harmonyos-guides/03_application-framework/arkts-gesture-events-single-gesture.md` - binding a single gesture and its action callbacks
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-gesture-events-single-gesture
- `documentation/harmonyos-references/04_media/js-apis-image.md` - `createImageSource`, `createPixelMap`, `ImageInfo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-image
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-transformation.md` - `translate` as a render transform
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-transformation
