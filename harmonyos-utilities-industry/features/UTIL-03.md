---
id: UTIL-03
title: Radar scan animation - an infinite animateTo sweep over ripples made from exit transitions
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/03_radar_scan_effect.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/radar_scan_effect-0000002236447458
sample: huawei_industry_tree/15_utilities/downloads/RadarScanEffect.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [hilog, window, "UIContext.animateTo", TransitionEffect, "TransitionEffect.asymmetric", "TransitionEffect.OPACITY", "TransitionEffect.scale", rotate, expectedFrameRateRange, setInterval, clearInterval, "UIContext.px2vp", "@StorageProp"]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0017, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card when a screen has to **look busy while something is being
discovered** - a Bluetooth or Wi-Fi scan, a nearby-user search, a device
pairing wait. The pattern is a rotating sweep image over a stack of expanding
ripples, with result markers fading in one at a time as the count rises.

Two techniques carry the whole effect and neither of them is a canvas. The
sweep is a single `Image` with `rotate({ z: 1, angle })` driven by one
`animateTo` with `iterations: -1`. The ripples are not animated at all: a
1 vp `Row` is mounted and unmounted every 300 ms by a toggling boolean, and
its **exit** transition scales it 400x while fading it out. Every unmount
launches one ripple. That inverts the usual instinct - you do not animate a
circle, you destroy one and let the transition do the work.

It generalises to any pulse: a "listening" microphone halo, a live-location
dot, a "device nearby" beacon. **Read `HW-15-0017` before shipping it**: in
this sample the stop button stops only the result counter, and the animation
runs forever.

## Feature checklist

- A 320 vp circular scan area, clipped, with a rotating sweep gradient over it.
- Ripples expand outward from the centre of that area continuously, one every
  300 ms, each taking 3 s to expand and fade.
- A capsule button toggles between 开始扫描 (start scan) and 停止扫描 (stop
  scan).
- While scanning, a 扫描中 (scanning) caption appears under the circle.
- Result icons fade in one per second at five fixed offsets, up to five.
- The back chevron and the bottom tab bar raise a "sample only" toast.
- Top and bottom padding follow the device avoid areas.

## Architecture

One `entry` module, one page, one constants file. There is no model layer and
no scan API - the "found devices" are a counter.

```
entry/src/main/ets
├── common/Constants.ets              every literal: durations, offsets, sizes
├── entryability/EntryAbility.ets     full-screen layout, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
└── pages/MainPage.ets                @Entry, the whole feature, 260 lines
```

The documented tree matches the zip exactly.

`MainPage` is split into six `@Builder` methods - `scanCircle`, `scanResult`,
`scanResults`, `scanEffect`, `scanPage`, `tabBuilder` - so `build()` reads as
a layout outline. Five `@State` fields hold everything: `isScanning`,
`circleFlag`, `rotateAngle`, `scannedNum`, `scanInterval`.

**The design decision worth copying** is that the ripple is a *conditional
child*, not an animated one. `scanCircle()` is rendered inside
`if (this.circleFlag)`, and `circleFlag` is flipped by a 300 ms interval. Each
flip mounts or unmounts the `Row`; the unmount fires the disappearance half of
its `TransitionEffect.asymmetric`, which scales it by 400 and fades it from
0.5 opacity to 0 over 3 s. Because the exit animation outlives the element,
several ripples overlap on screen at once with no list, no index arithmetic
and no per-ripple state - a single boolean produces the whole pulse train.

The decision **worth avoiding** is in the same place: `startScan()` runs from
`aboutToAppear`, so the ripples begin before the user has pressed anything,
and nothing in the file ever stops them (`HW-15-0017`).

## Implementation steps

1. **Give the scan area a fixed square size and `clip(true)`** so an expanding
   ripple is cut off at the circle instead of covering the page.
2. **Make the ripple a conditional 1 vp `Row`** positioned at 50%/50% of that
   `Stack`, with `borderRadius('50%')` and half opacity.
3. **Put the animation on the disappearance half** of
   `TransitionEffect.asymmetric(IDENTITY, OPACITY.animation({...}).combine(scale))` -
   appearance must stay `IDENTITY` or the mount pops a full-size circle.
4. **Toggle the flag on an interval** shorter than the transition duration
   (300 ms against 3000 ms) so ripples overlap.
5. **Drive the sweep with one `animateTo`** carrying `iterations: -1`,
   `Curve.Linear` and a target of 360°; start it from the sweep's `onAppear`,
   not from `aboutToAppear`, so it binds to a mounted component.
6. **Declare an `expectedFrameRateRange`** on that `animateTo` - a continuous
   background animation is exactly the case the frame-rate hint exists for.
7. **Gate both the sweep and the ripple interval on `isScanning`**, and reset
   `scannedNum` when scanning stops (`HW-15-0017`). As shipped, only the
   result counter is cleared.
8. **Clear both intervals in `aboutToDisappear`** - the sample does this
   correctly and it is the one lifecycle detail most copies of this pattern
   miss.

## Verified snippets

All snippets are from `RadarScanEffect.zip`. Corrected forms are marked.

**The ripple — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
startScan(): void {
  this.circleFlag = !this.circleFlag;
  this.circleInterval = setInterval(() => {
    this.circleFlag = !this.circleFlag;
  }, Constants.CIRCLE_DURATION);        // 300 ms
}

// Round ripple effect
@Builder
scanCircle() {
  Row()
    .width(Constants.CIRCLE_INITAL_SCALE)      // 1 vp
    .height(Constants.CIRCLE_INITAL_SCALE)
    .position({ x: Constants.HALF, y: Constants.HALF })
    .backgroundColor($r('app.color.circle_color'))
    .opacity(Constants.HALF_OPACITY)
    .transition(TransitionEffect.asymmetric(
      TransitionEffect.IDENTITY,
      TransitionEffect.OPACITY.animation({
        duration: Constants.CIRCLE_ANIMATION_DURATION,   // 3000 ms
        curve: Curve.EaseInOut
      })
        .combine(
          TransitionEffect.scale({ x: Constants.CIRCLE_SCALE, y: Constants.CIRCLE_SCALE })  // 400x
        )
    ))
    .borderRadius(Constants.HALF);
}
```

**Three numbers carry the design.** The element is 1 vp wide, the exit scale
is 400, and the exit runs for 3 s - so one ripple grows from a point to 400 vp
and dies just past the 320 vp clip boundary. The toggle period is 300 ms, one
tenth of the exit duration, which is what makes ten ripples visible at once.

`TransitionEffect.IDENTITY` on the appearance half is not filler: without it
the newly mounted `Row` would animate in with the default transition and the
pulse would flash. `.combine()` is what lets opacity and scale share one
timing configuration - the `.animation()` call attaches to the whole combined
effect, not just the opacity part.

Note the ripple is `position`ed at `'50%'/'50%'` of the parent `Stack`, not
centred by alignment, because the element it is positioned as is 1 vp: its
top-left corner *is* the centre point it must expand from.

**The sweep — same file** (as shipped)

```typescript
// Sweeping effect
@Builder
scanEffect() {
  Column() {
    Image($r('app.media.scan_effect'))
      .width('120%')
      .height('100%');
  }
  .width(Constants.SCAN_SIZE)            // 320
  .height(Constants.SCAN_SIZE)
  .rotate({ x: 0, y: 0, z: 1, angle: this.rotateAngle })
  .onAppear(() => {
    this.getUIContext().animateTo({
      duration: Constants.SCAN_ANIMATION_DURATION,   // 2000 ms
      curve: Curve.Linear,
      iterations: -1,      // Setting -1 indicates that the animation loops infinitely
      playMode: PlayMode.Normal,
      expectedFrameRateRange: {
        min: Constants.MIN_RATE,          // 10
        max: Constants.MAX_RATE,          // 120
        expected: Constants.EXPECTED_RATE // 60
      }
    }, () => {
      this.rotateAngle = Constants.FULL_CIRCLE;      // 360
    });
  });
}
```

**`iterations: -1` plus `Curve.Linear` is the whole sweep.** The closure
assigns 360 once; the infinite iteration count restarts the 0→360 interpolation
each cycle, and a linear curve is what keeps the restart invisible - any eased
curve would make the sweep visibly stall at the seam.

`expectedFrameRateRange` is the part worth copying into your own code. A radar
sweep is a long-lived decorative animation, so telling the system it is happy
between 10 and 120 fps and would like 60 lets the frame-rate governor drop it
when the device is under load instead of competing with foreground work.

The rotation is applied to the wrapping `Column`, not the `Image`, and the
image is `width('120%')` inside it - the sweep gradient overhangs its own
rotation box so no corner of the sweep leaves a gap as it turns. The parent
`Stack` has `.clip(true)`, which is what trims the overhang back to the circle.

**Start and stop — same file** (corrected, see `HW-15-0017`)

```typescript
@State isScanning: boolean = false;
@State circleFlag: boolean = false;
@State scannedNum: number = 0;
@State scanInterval: number = 0;
circleInterval: number = 0;

aboutToAppear(): void {
  // FIX: the sample calls startScan() here, so ripples run before the user starts
}

aboutToDisappear(): void {
  clearInterval(this.scanInterval);
  clearInterval(this.circleInterval);
}

// ...
Button(this.isScanning === true ? $r('app.string.stop_scan') : $r('app.string.start_scan'))
  .type(ButtonType.Capsule)
  .onClick(() => {
    this.isScanning = !this.isScanning;
    if (this.isScanning === true) {
      this.scannedNum = 0;
      this.startScan();                       // FIX: start the ripple train here
      this.scanInterval = setInterval(() => {
        this.scannedNum++;
      }, Constants.RESULT_DURATION);          // 1000 ms
    } else {
      clearInterval(this.scanInterval);
      clearInterval(this.circleInterval);     // FIX: absent - ripples keep pulsing
      this.circleFlag = false;
      this.scannedNum = 0;                    // FIX: absent - results stay on screen
    }
  });
```

**Stopping a scan means stopping three things, and the sample stops one.** The
result counter is cleared; the ripple interval started in `aboutToAppear` is
not, and the sweep's `animateTo` has no handle at all - an infinite explicit
animation can only be ended by re-running `animateTo` on the same property or
by unmounting the component. The practical fix is to render `scanEffect()`
inside `if (this.isScanning)` so stopping unmounts it, which also frees the
frame-rate reservation.

Resetting `scannedNum` on **both** edges matters: the sample resets on start,
so between a stop and the next start the stale result icons stay visible under
a button that says 开始扫描.

**The result markers — same file** (as shipped)

```typescript
@Builder
scanResult(offsetX: number, offsetY: number) {
  Image($r('app.media.startIcon')).width(Constants.RESULT_WIDTH)
    .transition(
      TransitionEffect.asymmetric(
        TransitionEffect.OPACITY.animation({
          duration: Constants.RESULT_ANIMATION_DURATION, curve: Curve.Ease
        }),
        TransitionEffect.IDENTITY
      )
    )
    .offset({ x: offsetX, y: offsetY });
}

// Result show effect
@Builder
scanResults() {
  if (this.scannedNum > 0) {
    this.scanResult(Constants.RESULT_FIRST_X, Constants.RESULT_FIRST_Y);
  }
  if (this.scannedNum > 1) {
    this.scanResult(Constants.RESULT_SECOND_X, Constants.RESULT_SECOND_Y);
  }
  // ... three more, up to five
}
```

This is the mirror image of the ripple: the asymmetric transition puts the
animation on **appearance** and `IDENTITY` on disappearance, so each marker
fades in over 1 s as the counter crosses its threshold and vanishes instantly
when the count resets.

The five positions are hardcoded constant pairs and the cascade is written as
five `if`s rather than a `ForEach` over an array of offsets. For a real scan,
replace `scannedNum` with the discovered-device list and the `if` ladder with
a `ForEach` keyed on the device id - the transition attribute works the same
way on `ForEach` children, which is the reason this pattern is worth learning
in the toy form first.

## Permissions & config

**None.** The sample declares no `requestPermissions` - it never scans
anything. A real Bluetooth scan would add `ohos.permission.ACCESS_BLUETOOTH`,
and a Wi-Fi or nearby-device scan the corresponding location permissions,
which changes the shape of the page: the button would have to raise a
permission request before it can start.

Avoid areas come through `AppStorage` as `topRectHeight` / `bottomRectHeight`,
read with `@StorageProp` and converted at the point of use:

```typescript
.padding({
  top: this.getUIContext().px2vp(this.topRectHeight),
  bottom: this.getUIContext().px2vp(this.bottomRectHeight)
})
```

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- The scan area is a fixed 320 vp square (`SCAN_SIZE`) and the five result
  offsets are absolute vp values tuned to it. On a tablet or a resized 2in1
  window the circle does not grow and the markers stay where they are.
- The ripple scale factor of 400 is relative to a 1 vp element, so it is tied
  to the same 320 vp geometry; changing `SCAN_SIZE` means retuning
  `CIRCLE_SCALE`.
- The bottom tab bar is decorative: `tabBuilder` colours index 0 as selected
  unconditionally and the whole `Row` raises one toast. There is no second
  page.
- Nothing here is a real scan. `scannedNum` is a 1 Hz counter capped visually
  at five markers - it keeps incrementing past five with no further effect.

## Pitfalls

- **`HW-15-0017`** (B/low, probable): **stop-scan only stops the counter.**
  `clearInterval` is called on `scanInterval` alone; the `-1`-iteration sweep
  and the `circleInterval` ripple train - both started outside the button -
  keep running, and the already-found markers stay on screen under a button
  that now reads 开始扫描. Fix: gate the animations on `isScanning` (render
  `scanEffect()` conditionally, clear `circleInterval` on stop) and reset
  `scannedNum` on the stopping edge as well as the starting one.

## References

- `documentation/harmonyos-references/02_application-framework/ts-transition-animation-component.md` - `TransitionEffect`, `asymmetric`, `combine`, and the mount/unmount trigger rules
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-transition-animation-component
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `animateTo`, `iterations: -1`, `expectedFrameRateRange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-transformation.md` - `rotate` and its axis vector
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-transformation
- `UTIL-04` - the other geometry sample in this industry, drawn on a `Canvas` instead of composed from transitions
