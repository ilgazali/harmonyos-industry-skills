---
id: SHOP-22
title: Pull-down second floor - a touch-driven promo storey above the home page
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/22_second_floor.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/second_floor-0000002386145197
sample: huawei_industry_tree/16_shopping/downloads/SecondFloor.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit"]
apis: [onTouch, TouchEvent, TouchType, "UIContext.animateTo", "UIContext.createAnimator", AnimatorResult, "AnimatorResult.onFrame", px2vp, position, offset, scale, rotate, opacity, "@Link", "@StorageProp", "@StorageLink", componentUtils, getRectangleById, Swiper, Tabs, TabsController, hilog, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-16-0024, HW-16-0027]
status: verified-with-fixes
---

## When to use

Load this card when a home page needs a **second, full-screen surface reached
by dragging down from the top** - the "二楼" (second floor) that Chinese
e-commerce apps put a campaign, a brand zone or a live stream on. It is the
gesture that turns pull-to-refresh into a doorway: a short pull refreshes, a
long pull opens another page.

The technique is worth reading even if you never ship a second floor. It is a
complete worked example of driving layout from raw touch deltas: one `offsetY`
number positioning the first floor, a damping factor, a drag threshold, three
outcome branches on touch-up, and two different animation mechanisms chosen
per branch (`animateTo` where the end state matters, `createAnimator` +
`onFrame` where a controllable frame-by-frame rebound matters). The same
skeleton covers custom bottom sheets, drawer panels and pull-to-collapse
headers.

**Read `HW-16-0024` before adopting it.** Every threshold in the sample is a
magic number tuned to one 760vp screen, and the field that was meant to fix
that (`screenHeight`) is read but never written.

## Feature checklist

- The home page starts with the second floor already laid out above it,
  scrolled out of view by a negative `offsetY`.
- Dragging down from anywhere on the home page pulls the first floor down and
  reveals the second floor behind it.
- The revealed second floor is scaled down (0.8) at rest and grows to 1.0 as
  the pull progresses.
- The first-floor search bar fades out as the pull deepens.
- A hint strip tracks the pull: 下拉刷新 (pull to refresh), then 释放刷新
  (release to refresh), then a "keep pulling for the second floor" message
  past the deeper threshold.
- Releasing under the refresh threshold springs the page back to the top.
- Releasing in the middle band hovers at a fixed height, spins a loading icon
  for two seconds, then springs back.
- Releasing past the expand threshold animates the second floor to full screen.
- Once on the second floor, dragging up past 150vp - or tapping the back arrow
  - retracts it and returns the home page.
- The second floor carries its own product page: banner, spec rows, price
  block and detail images.

## Architecture

One `entry` module. Two pages that are really one page: `MainPage` is the
`@Entry` and the second floor is a component it renders above itself.

```
entry/src/main/ets
├── common
│   ├── Constants.ets            all thresholds, durations and the damping factor
│   └── ListDataConstants.ets    static grid/menu data for the home page
├── components
│   ├── GridItemComponent.ets    one app icon
│   └── MenuItemComponent.ets    one swiper page of icons
├── entryability/EntryAbility.ets  full screen, avoid areas -> AppStorage
├── entrybackupability/
├── model/ProductDataModel.ets   the tab list feed
├── pages
│   ├── MainPage.ets             @Entry: first floor, all touch handling, all animation (478 lines)
│   └── SecondFloor.ets          the second-floor host: its own touch handling + retract animation
├── utils/Logger.ets
└── views
    ├── BottomTabs.ets           bottom navigation bar
    ├── CustomTabBarView.ets     the tab strip, measured with componentUtils
    ├── ProductView.ets          one tab's product list
    └── SecondFloorView.ets      the second floor's actual content (496 lines, pure layout)
```

The documented 工程目录 matches the zip exactly.

**The design decision worth copying** is that there is **one offset number and
it lives in the parent**. `MainPage` owns `@State offsetY`, seeded to
`-floorHeight`, and passes it to `SecondFloor` as `mainPageOffsetY: $offsetY`
- an `@Link`, so both components write the same value. The first floor is
positioned at `y: this.offsetY + this.floorHeight`; the second floor is simply
laid out above it and revealed as that offset grows. No component ever asks
"where is the other one" - both derive their geometry from the single number.

That is what makes the two touch handlers composable. `MainPage.onTouch`
handles drags while the home page is on top; `SecondFloor.onTouch` handles
drags once the second floor is up, calls `event.stopPropagation()` so the
gestures do not fight, and writes the same `offsetY` back through the `@Link`.
The pair behaves like one continuous surface.

The part **worth avoiding** is how the thresholds are expressed. `floorHeight`
is `@State ... = 760` in the parent and `@State ... = 0` in the child, then
passed in on construction; `-560`, `690`, `70`, `1560 / 1000` and `800` appear
as bare literals inside the handlers. They are all derived from that 760 and
none of them says so. `Constants.ets` holds the honest half of the tuning
(`FLING_FACTOR`, `TRIGGER_HEIGHT`, `BACK_HEIGHT`, the durations); the geometry
never made it there.

## Implementation steps

1. **Seed the offset to minus the floor height** (`offsetY = -floorHeight`) and
   position the first floor at `y: offsetY + floorHeight`, so `0` means
   "second floor fully open" and `-floorHeight` means "closed".
2. **Derive `floorHeight` from the window height, not from a literal**
   (`HW-16-0024`). The sample declares `@StorageLink('screenHeight')` for
   exactly this and then never uses it - and nothing ever writes that key.
3. **Handle `TouchType.Down / Move / Up / Cancel` in one `onTouch` switch** and
   call `event.stopPropagation()` so the drag does not reach the lists below.
4. **Track a `dragging` flag, not just deltas.** Set it only once the finger
   has moved in the opening direction; `TouchType.Down` clears it. Without
   this every tap becomes a one-pixel drag.
5. **Convert deltas with `px2vp` and multiply by a damping factor.**
   `event.touches[0].windowY` is in px; the offset is in vp. `FLING_FACTOR`
   (1.5) is the "feel" knob and is the one constant the sample documents as
   device-tunable.
6. **Clamp both ends inside the move handler** - never past `-floorHeight`,
   never past the open position - rather than letting the animation correct an
   out-of-range value afterwards.
7. **Branch three ways on touch-up**: past the expand threshold, in the refresh
   band, or below it. Each branch gets its own animation.
8. **Use `animateTo` when you only care about the end state** (the expand), and
   `createAnimator` + `onFrame` when you need a controlled rebound to an
   arbitrary height (the refresh hover and the spring-back).
9. **Give the second floor its own touch handler with a slop threshold**
   (`TOUCH_SLOP`) so scrolling its content does not retract it.
10. **Fade the search bar from the same offset**, deriving the band from
    `floorHeight` rather than hardcoding the 690/70 pair (`HW-16-0024`).

## Verified snippets

All snippets are from `SecondFloor.zip`. Corrected forms are marked.

**The single offset and the touch switch - `entry/src/main/ets/pages/MainPage.ets`** (corrected, see `HW-16-0024`)

```typescript
@State floorHeight: number = this.getUIContext().px2vp(
  AppStorage.get<number>('screenHeight') ?? 0);   // FIX: shipped code is `= 760`
@State expandFloorTriggerDistance: number = 200;
@State packUpFloorTriggerDistance: number = 150;
@State dragging: boolean = false;
@State offsetY: number = -this.floorHeight;
@State miniAppScale: Scale = { x: 0.8, y: 0.8 };

build() {
  Column() {
    SecondFloor({
      floorHeight: this.floorHeight,
      mainPageOffsetY: $offsetY,           // @Link: the child writes the same number
      packUpFloorTriggerDistance: this.packUpFloorTriggerDistance,
      onShow: $onShow,
      miniAppScale: $miniAppScale
    });

    Column() {
      if (this.onShow) {
        this.refresh();                    // the pull hint strip
      }
      this.mainPageView();
    }
    .layoutWeight(1)
    .position({ x: 0, y: this.offsetY + this.floorHeight });
  }
  .width('100%')
  .height('100%')
  .onTouch((event) => {
    switch (event.type) {
      case TouchType.Down:
        this.onTouchDown(event);
        break;
      case TouchType.Move:
        this.onTouchMove(event);
        break;
      case TouchType.Up:
      case TouchType.Cancel:
        this.onTouchUp();
        break;
    }
    event.stopPropagation();
  });
}
```

**`TouchType.Cancel` falling through to the same handler as `Up` is not
cosmetic.** A cancel arrives when the system takes the gesture away - an
incoming call, a parent container claiming the drag - and if it is not handled
the page is left frozen mid-pull with no finger on screen. Grouping the two
cases is the minimum correct handling.

The `if (this.onShow)` around the hint strip is what keeps the refresh row out
of the layout entirely between gestures, rather than sizing it to zero: the
strip's height is `floorHeight - |offsetY|`, which is only meaningful while a
pull is in progress.

**Reading the drag - same file** (as shipped)

```typescript
private onTouchDown(event: TouchEvent): void {
  this.lastY = event.touches[0].windowY;
  this.onShow = true;
  this.dragging = false;             // every new touch starts as "not yet a drag"
}

private onTouchMove(event: TouchEvent): void {
  this.onShow = true;
  let currentY = event.touches[0].windowY;
  // negative deltaY is an upward swipe, positive is downward
  let deltaY = currentY - this.lastY;
  if (this.dragging) {
    if (deltaY < 0) {
      // at offsetY -760 the scale is 0.8; at -560 it is 1.0
      this.miniAppScale = {
        x: (1560 - Math.abs(this.offsetY)) / 1000,
        y: (1560 - Math.abs(this.offsetY)) / 1000
      };
      if (this.offsetY > -this.floorHeight) {
        this.offsetY = this.offsetY + this.getUIContext().px2vp(deltaY) * Constants.FLING_FACTOR;
      } else {
        this.offsetY = -this.floorHeight;          // clamp: cannot close further
      }
    } else if (deltaY > 0 && this.offsetY <= -560) {
      this.offsetY = this.offsetY + this.getUIContext().px2vp(deltaY) * Constants.FLING_FACTOR;
      this.miniAppScale = {
        x: (1560 - Math.abs(this.offsetY)) / 1000,
        y: (1560 - Math.abs(this.offsetY)) / 1000
      };
    } else if (this.offsetY > -560) {
      this.offsetY = -559;                         // clamp: hand over to the release branch
    }
    this.lastY = currentY;
  } else {
    if (deltaY > 0) {
      this.dragging = true;                        // only a downward move starts a drag
      this.lastY = currentY;
    }
  }
}
```

**`this.lastY = currentY` at the end of every handled move is what makes this
incremental** rather than absolute: each frame moves the page by the delta
since the previous frame, so the page tracks the finger even after clamping
has thrown some travel away. Forget it and the offset accelerates away from
the finger.

The scale formula is the sample's magic-number problem in miniature.
`(1560 - |offsetY|) / 1000` maps `-760 -> 0.8` and `-560 -> 1.0` - correct, and
completely opaque. Written as `1 - (|offsetY| - OPEN_AT) / SCALE_BAND` with
two named constants it would survive a change of floor height; as written,
changing `floorHeight` silently breaks the zoom, the `-560` gate and the
search-bar fade at once. That coupling is what `HW-16-0024` is about.

**The three release branches - same file** (as shipped)

```typescript
private onTouchUp(): void {
  if (this.dragging) {
    // (floorHeight - |offsetY|) is how far the user has pulled
    if ((this.floorHeight - Math.abs(this.offsetY)) > this.expandFloorTriggerDistance) {
      this.expandSecondFloor();          // past 200vp: open the second floor
    } else if ((this.floorHeight - Math.abs(this.offsetY)) <= this.expandFloorTriggerDistance &&
      (this.floorHeight - Math.abs(this.offsetY)) > Constants.BACK_HEIGHT) {
      this.scrollByUpdate();             // 100..200vp: hover, then
      this.updateData();                 // spin for 2s and spring back
    } else {
      this.scrollByTop();                // under 100vp: straight back
    }
  }
}

private expandSecondFloor(): void {
  if (this.offsetY < 0) {
    this.getUIContext().animateTo({
      duration: Constants.EXPAND_SECOND_FLOOR_TIME,
      curve: Curve.EaseOut,
      iterations: 1,
      playMode: PlayMode.Normal,
      finishCallbackType: FinishCallbackType.REMOVED,
      onFinish: () => {
        this.onShow = false;             // drop the hint strip once we are there
      }
    }, () => {
      this.offsetY = 0;
      this.miniAppScale = { x: 1, y: 1 };
    });
  }
}

private scrollByUpdate(): void {
  this.backAnimator = this.getUIContext().createAnimator({
    duration: Constants.SCROLL_BY_UPDATE,
    easing: 'linear',
    delay: 0,
    fill: 'forwards',                    // hold the end state: this is a hover, not a bounce
    direction: 'normal',
    iterations: 1,
    begin: this.offsetY,
    end: -this.floorHeight + Constants.UPDATE_HEIGHT
  });
  this.backAnimator.onFrame = (value: number) => {
    this.offsetY = value;
  };
  this.backAnimator.play();
}
```

**Two animation mechanisms, chosen deliberately.** `animateTo` is a
state-transition animation: you assign the end state inside the closure and
the framework interpolates every property that changed - here both `offsetY`
and the scale, on one curve, which is exactly what "open the second floor"
should look like. `createAnimator` is a value generator: `begin`, `end` and an
`onFrame` callback that writes each intermediate number. The refresh hover
needs the second form because it must stop at an arbitrary height
(`-floorHeight + 150`) and stay there - `fill: 'forwards'` is what keeps it
from snapping back when the animation ends.

`finishCallbackType: FinishCallbackType.REMOVED` on the expand means `onFinish`
fires when the animation is genuinely removed from the pipeline, not merely
when its own duration elapses - so the hint strip is not torn down while a
follow-up animation is still driving the same property.

Note that `backAnimator` is a single field reused by both `scrollByUpdate` and
`scrollByTop`, and `updateData`'s `setTimeout` can fire after a new gesture has
replaced it. Nothing calls `finish()` or `cancel()` on the old animator.

**The search-bar fade - same file** (corrected, see `HW-16-0024`)

```typescript
@Builder
search() {
  Row() {
    Image($r('app.media.scan')).height(40);
    Row() {
      Image($r('app.media.search')).height(16);
      TextInput({ placeholder: $r('app.string.chair') })
        .backgroundColor(Color.Transparent);
      Image($r('app.media.camera')).height(23);
    }
    .backgroundColor('#0C000000')
    .borderRadius(24);
    Button($r('app.string.search')).height(40);
  }
  .padding({ top: this.getUIContext().px2vp(this.topRectHeight) })
  // FIX: shipped code is `.opacity((Math.abs(this.offsetY) - 690) / 70)`
  .opacity((Math.abs(this.offsetY) - (this.floorHeight - Constants.FADE_BAND)) / Constants.FADE_BAND)
  .backgroundColor('#F1F3F5')
  .width('100%');
}
```

`|offsetY|` runs from `floorHeight` (closed) down to `0` (open), so the
expression maps the last `FADE_BAND` vp of closure to opacity `1..0` and
everything beyond it to a negative value, which clamps to transparent. Written
against `floorHeight` it holds on any screen; written as `690 / 70` it holds
only when `floorHeight` is 760.

## Permissions & config

**None.** The sample declares no `requestPermissions`. `module.json5` targets
`phone`, `tablet` and `2in1`.

`EntryAbility` loads `pages/MainPage`, sets the window layout full screen and
publishes `topRectHeight` and `bottomRectHeight` into `AppStorage`, which both
floors consume with `@StorageProp`/`@StorageLink` and convert with `px2vp`. It
does **not** publish `screenHeight`, although `MainPage` declares
`@StorageLink('screenHeight')` for it - see `HW-16-0024`. Adding
`AppStorage.setOrCreate('screenHeight', windowClass.getWindowProperties().windowRect.height)`
next to the two avoid-area writes is the missing half of the fix.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The gesture is implemented with raw `onTouch`, not a `PanGesture` or
  `Refresh`. That buys per-pixel control and costs the framework's gesture
  arbitration: `event.stopPropagation()` in both handlers is what keeps the
  inner `Swiper`, `Tabs` and lists from receiving the drag, and it is why the
  home page cannot itself be vertically scrollable.
- The first-floor stack applies `.offset({ top: this.offsetY === 0 ? 800 : 0 })`
  to keep the search bar from reappearing at the second floor's bottom edge -
  another literal tied to the 760 floor height.
- Only single-finger drags are handled; `SecondFloor` explicitly drops events
  with `event.touches.length !== 1`.
- No pull-to-refresh data actually loads: `updateData` spins for
  `UPDATE_TIME` (2000ms) and returns. The comment marks the spot.
- Second-floor content (`SecondFloorView`, 496 lines) is static layout with
  string and media resources - it is the destination, not a feature.

## Pitfalls

- **`HW-16-0024`** (B/low, probable): `floorHeight` is hardcoded to `760` and
  the search-bar fade to `(|offsetY| - 690) / 70`; the `-560` scale gate, the
  `1560 / 1000` scale formula and the `800` offset guard are all silently
  derived from that same 760. `MainPage` declares
  `@StorageLink('screenHeight')` to size the floor properly and never reads
  it - and no code ever writes that AppStorage key, so it would be `undefined`
  if it did. On a foldable, a tablet or a split-screen window the second floor
  overflows or leaves a gap and the fade band no longer lines up with the pull
  range. Fix: publish the window height from `EntryAbility`, set
  `floorHeight` from it via `px2vp`, and express every threshold as a named
  constant relative to `floorHeight`.
- `backAnimator` is one field shared by both rebound paths and is never
  cancelled; `updateData`'s `setTimeout` can also outlive the gesture that
  started it and write `offsetY` under a newer drag. Keep a handle and
  `cancel()` on `TouchType.Down`.
- The hint strip's height is `floorHeight - |offsetY|`, so it renders with a
  negative height for one frame if the offset ever exceeds the floor height.
  The clamps in `onTouchMove` are the only thing preventing it.

## References

- `documentation/harmonyos-references/02_application-framework/ts-universal-events-touch.md` - `onTouch`, `TouchEvent`, `touches[].windowY`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-events-touch
- `documentation/harmonyos-references/02_application-framework/ts-appendix-enums.md` - `TouchType` and `FinishCallbackType`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-appendix-enums
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `animateTo`, `createAnimator`, `px2vp` off the UIContext
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-references/02_application-framework/ts-explicit-animation.md` - `AnimateParam`, `onFinish` and the finish callback types
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-explicit-animation
- `documentation/harmonyos-references/02_application-framework/js-apis-animator.md` - `AnimatorResult`, `onFrame`, `fill: 'forwards'`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-animator
- `documentation/harmonyos-guides/03_application-framework/arkts-link.md` - `@Link` and the `$state` binding used for `mainPageOffsetY`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-link
- `huawei_industry_tree/16_shopping/docs/22_second_floor.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/second_floor-0000002386145197
- `SHOP-06` - the `Refresh`-based variant of "pull down to go somewhere", when the framework component is enough
