---
id: UTIL-05
title: Canvas speed gauge - a reusable needle dial driven by an Animator and netQuality
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/05_network_speed_guage.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/network_speed_guage-0000002284206189
sample: huawei_industry_tree/15_utilities/downloads/NetworkSpeedTest.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.NetworkBoostKit", "@kit.PerformanceAnalysisKit"]
apis: [Canvas, CanvasRenderingContext2D, RenderingContextSettings, createConicGradient, arc, fillText, measureText, "UIContext.createAnimator", AnimatorResult, "AnimatorResult.onFrame", "netQuality.on", "netQuality.off", NetworkQos, "request.downloadFile", DownloadTask, "fileIo.unlinkSync", "fs.accessSync", "@Prop", "@Watch", getRectangleById, px2vp, AlertDialog, CustomDialogController]
permissions: [ohos.permission.GET_NETWORK_INFO, ohos.permission.INTERNET]
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0020, HW-15-0021, HW-15-0101, HW-15-0102]
status: verified-with-fixes
---

## When to use

Load this card when you need **a circular gauge that a number drives** - a
needle sweeping an arc between a min and a max, with a coloured progress ring,
numeric ticks and a big value in the middle. The sample uses it for a network
speed test, but the component (`SmartGauge`) takes range, angles, colours and
animation duration as props and knows nothing about networking.

The second half of the card is the measurement itself: subscribe to
`netQuality`'s `netQosChange`, start a large download to create traffic worth
measuring, and read `linkDownRate` off each QoS report. That is the sanctioned
way to observe throughput on HarmonyOS - you do not time the download
yourself, the system tells you the link rate.

The two halves generalise separately. The gauge fits any bounded scalar:
temperature, RPM, battery, a credit score, storage used. The netQuality half
fits any "how good is the connection right now" feature - a video player
picking a bitrate, an uploader deciding whether to defer. **Read `HW-15-0020`
before shipping either**: the sample's failure path leaves the button
permanently disabled and the QoS subscription alive.

## Feature checklist

- A full-page gauge: 0-200 scale, arc from 135 deg to 45 deg, six numbered
  ticks, blue progress ring and a triangular needle.
- The value and the unit `Mbps` are drawn in the centre of the dial.
- A test button below; while a test runs the label switches to 测速中 (testing)
  and the button dims to 0.4 opacity.
- Pressing the button while a test is running raises a toast instead of
  starting a second test.
- The needle animates between readings rather than jumping - each QoS report
  starts a 300 ms tween and the dial is repainted every frame.
- When the download finishes, the needle returns to zero and a report dialog
  shows the peak rate observed.
- Confirming the dialog resets the peak and re-enables the button.
- The temporary download file is deleted when the test completes.

## Architecture

One `entry` module, seven files. The gauge is a real component; everything
else is a page and two small utilities.

```
entry/src/main/ets
├── common/Constants.ets              sizes, colours, canvas font strings, MBPS divisor
├── components/Dashboard.ets          SmartGauge - the whole Canvas dial, 282 lines
├── entryability/EntryAbility.ets     full-screen layout, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── pages/SpeedTest.ets               @Entry - the gauge, the button, the QoS subscription
└── utils
    ├── DownloadUtil.ets              downLoadFileByRequest + the DownLoadConfig interface
    └── FileUtil.ets                  FILE_UTILS.removeFile, an unlinkSync wrapper
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that `SmartGauge` is parameterised by
props and has no idea what it is measuring. `minValue`, `maxValue`,
`startAngle`, `endAngle`, `progressColors`, `pointColor`, `enableAnimation`
and `animationDuration` are all `@Prop`; the only input that changes at
runtime is `value`, and it carries `@Watch('onValueChange')`. The page hands
it a number and the component owns the animation:

```typescript
SmartGauge({
  minValue: 0, maxValue: 200, startAngle: 135, endAngle: 45,
  value: this.currentValue,
  progressColors: [Constants.THEME_COLOR],
  pointColor: Constants.THEME_COLOR,
  animationDuration: Constants.ANIMATION_DURATION
})
  .width('100%')
  .aspectRatio(1);
```

`aspectRatio(1)` is load-bearing: every dimension inside the component is
derived from a single `canvasSize`, so the canvas must be square.

## Implementation steps

1. **Make the canvas square and measure it once.** In `onReady`, read the
   canvas width with `getComponentUtils().getRectangleById('canvas')`, convert
   px to vp, and store it in a `@State` with `@Watch`.
2. **Derive every radius from `canvasSize`** in that watcher - ring width,
   radius, needle length, needle hub. No absolute vp anywhere in the drawing
   code.
3. **Support wrap-around angle ranges.** `endAngle` (45) is smaller than
   `startAngle` (135), so add 360 before subtracting; that one helper
   (`getEffectiveAngleRange`) is what lets the dial be a 270 deg arc.
4. **Draw in a fixed order** into one `drawGauge()`: clear, border arc,
   gradient progress arc, ticks, needle, value text. Anything else and the
   needle ends up under the ring.
5. **Animate by repainting, not by transforming.** Create an `AnimatorResult`
   in `aboutToAppear`, reset it with `begin`/`end` on each value change, and
   redraw from `onFrame`.
6. **Set `fillStyle` for colours and `font` for fonts** - the sample assigns a
   colour string to `ctx.font` for the unit label (`HW-15-0021`).
7. **Subscribe to `netQosChange` when the test starts**, divide `linkDownRate`
   by 1e6 for Mbps, and track the max separately from the displayed value.
8. **Delete a stale temp file before downloading**, otherwise `accessSync`
   sees a leftover from a killed run and completes the test instantly with a
   0 Mbps report (`HW-15-0020`).
9. **Reset state and unsubscribe on every exit path** - success, `fail`,
   the rejected `downloadFile` promise, and `aboutToDisappear`
   (`HW-15-0020`).

## Verified snippets

All snippets are from `NetworkSpeedTest.zip`. Corrected forms are marked.

**Deriving the geometry — `entry/src/main/ets/components/Dashboard.ets`** (as shipped)

```typescript
@State @Watch('onCanvasSizeChange') canvasSize: number = 0;
private progressRingWidth: number = this.canvasSize * 0.06;              // ring thickness
private radius: number = (this.canvasSize - this.progressRingWidth) * 0.45;
private pointerLength: number = (this.radius - this.progressRingWidth) * 0.8;
private pointerRadius: number = this.pointerLength * 0.1;                // the hub

onCanvasSizeChange() {
  this.progressRingWidth = this.canvasSize * 0.06;
  this.radius = (this.canvasSize - this.progressRingWidth) * 0.5 * 0.9;
  this.pointerLength = (this.radius - this.progressRingWidth) * 0.8;
  this.pointerRadius = this.pointerLength * 0.1;
}

build() {
  Column() {
    Canvas(this.ctx)
      .width('100%')
      .height('100%')
      .onReady(() => {
        let canvasWidth: number = this.getUIContext()
          .getComponentUtils()
          .getRectangleById('canvas')
          .size
          .width;
        this.canvasSize = this.getUIContext().px2vp(canvasWidth);
        this.drawGauge();
      })
      .id('canvas');
  };
}
```

**The measurement happens exactly once, in `onReady`.** A `Canvas` does not
report its own size to script, so the component gives itself an `id` and asks
`getComponentUtils().getRectangleById()` for the laid-out rectangle. That
returns **px**, and every drawing call below works in vp, hence the `px2vp`.
Writing the result into a `@Watch`ed `@State` means the four derived radii are
recomputed by the framework's own change notification rather than by a manual
call - so a window resize that relays the canvas fixes the geometry for free.

Note the initialisers on lines 2-5 all run against `canvasSize === 0` and
produce zeros; they exist only to satisfy the type checker. The watcher is
what actually sets them, and it uses a slightly different radius formula
(`* 0.5 * 0.9` rather than `* 0.45` - numerically the same, which is a hint
that only one of the two is maintained).

**The dial paint order and the conic gradient — same file** (as shipped)

```typescript
private drawGauge() {
  let centerX = this.canvasSize / 2;
  let centerY = this.canvasSize / 2;
  this.ctx.clearRect(0, 0, this.canvasSize, this.canvasSize);
  this.drawProgressBorder(centerX, centerY);   // the grey track
  this.drawProgressRing(centerX, centerY);     // the coloured arc up to currentValue
  this.drawScale(centerX, centerY);            // tick marks + numbers
  this.drawPointer(centerX, centerY);          // the needle
  this.drawValueText(centerX, centerY);        // the big number + unit
}

private getEffectiveAngleRange(): number {
  let effectiveEnd = this.endAngle;
  if (this.endAngle < this.startAngle) {       // 45 < 135: wrap the arc past 360
    effectiveEnd += 360;
  }
  return effectiveEnd - this.startAngle;
}

private createProgressGradient(cx: number, cy: number): CanvasGradient {
  let gradient = this.ctx.createConicGradient(this.degToRad(this.startAngle), cx, cy);
  if (this.progressColors) {
    this.progressColors.forEach((color, index) => {
      gradient.addColorStop(index / this.progressColors.length, color);
    });
  }
  return gradient;
}

private drawPointer(cx: number, cy: number) {
  let angleRange = this.getEffectiveAngleRange();
  let normalizedValue = (this.currentValue - this.minValue) / (this.maxValue - this.minValue);
  let currentAngle = this.startAngle + angleRange * normalizedValue;
  let rad = this.degToRad(currentAngle - 90);  // canvas 0 deg points right, the needle points up
  this.ctx.save();
  this.ctx.translate(cx, cy);
  this.ctx.rotate(rad);
  this.ctx.beginPath();
  this.ctx.arc(0, 0, this.pointerRadius, 0, 2 * Math.PI);
  this.ctx.fillStyle = this.pointColor;
  this.ctx.fill();
  this.ctx.beginPath();
  this.ctx.moveTo(0, this.pointerLength);      // tip
  this.ctx.lineTo(-this.pointerRadius, 0);     // base left
  this.ctx.lineTo(this.pointerRadius, 0);      // base right
  this.ctx.closePath();
  this.ctx.fill();
  this.ctx.restore();
}
```

**Three details carry this drawing code.** `getEffectiveAngleRange` is what
allows a gauge whose end angle is numerically less than its start - without
the `+= 360` the arc range would be negative and the needle would run
backwards. `createConicGradient` anchored at `startAngle` is the right
gradient for a dial: a linear gradient across the bounding box would put the
colour transition in the wrong place on a curve, whereas a conic gradient
sweeps with the arc, so a `['#4CAF50','#FFC107','#F44336']` array becomes
green-through-red *around* the ring. And `translate`/`rotate`/`restore` means
the needle is drawn in its own upright coordinate system - the three
`moveTo`/`lineTo` calls describe a triangle pointing straight up and the
rotation places it, which is far easier to reason about than computing three
rotated vertices. The `- 90` compensates for the canvas convention that 0 rad
points along +x.

**Animating by repaint — same file** (as shipped)

```typescript
@Prop @Watch('onValueChange') value: number = 0;
private currentValue: number = 0;
private gaugeAnimator: AnimatorResult | undefined = undefined;
private animatorOptions: AnimatorOptions = {
  duration: this.animationDuration, easing: 'ease-in-out', delay: 0,
  fill: 'forwards', direction: 'normal', iterations: 1, begin: 0, end: 0
};

aboutToAppear(): void {
  this.gaugeAnimator = this.getUIContext().createAnimator(this.animatorOptions);
}

onValueChange() {
  if (this.currentValue === this.value) {
    return;
  }
  if (this.gaugeAnimator) {
    this.gaugeAnimator.finish();            // land the previous tween before retargeting
  }
  if (this.enableAnimation) {
    this.startAnimation(this.currentValue, this.value);
  } else {
    this.currentValue = this.value;
    this.drawGauge();
  }
}

private startAnimation(startValue: number, targetValue: number) {
  if (this.gaugeAnimator) {
    this.animatorOptions.begin = startValue;
    this.animatorOptions.end = targetValue;
    this.gaugeAnimator.reset(this.animatorOptions);
    this.gaugeAnimator.onFrame = (tmpValue) => {
      this.currentValue = tmpValue;
      this.drawGauge();                     // repaint the whole dial each frame
    };
    this.gaugeAnimator.play();
  }
}
```

**This is the reason an `Animator` is used instead of `animateTo`.** ArkUI's
declarative animation interpolates component *attributes*; nothing about a
`Canvas` is an attribute here - the needle exists only as pixels the component
paints. `AnimatorResult.onFrame` hands back the interpolated scalar on each
vsync, and the component turns that scalar into a repaint. The animator is
created once in `aboutToAppear` and `reset` with new `begin`/`end` for each
value change, which is cheaper than constructing one per reading - QoS
callbacks can arrive several times a second.

`currentValue` is deliberately a plain field, not `@State`: it changes every
frame and nothing in the declarative tree depends on it, so making it stateful
would queue a useless re-render sixty times a second. The one thing to watch
is that `drawValueText` prints `this.value` (the target) while the needle
tracks `this.currentValue` (the tween), so the number snaps while the needle
glides.

**The value text — same file** (corrected, see `HW-15-0021`)

```typescript
private drawValueText(cx: number, cy: number) {
  this.ctx.font = Constants.DASHBOARD_VALUE_FONT;      // '500 38vp sans-serif'
  this.ctx.fillStyle = Constants.DASHBOARD_VALUE_COLOR; // '#161E26'
  let message = `${this.value.toFixed(1)}`;
  let yOffset = this.radius * 0.6;
  this.ctx.fillText(message, cx, cy + yOffset);
  // the unit label
  this.ctx.font = Constants.DASHBOARD_UNIT_FONT;       // '400 14vp sans-serif'
  this.ctx.fillStyle = Constants.DASHBOARD_UNIT_COLOR; // FIX: shipped code assigns this to ctx.font
  this.ctx.fillText(Constants.DASHBOARD_UNIT, cx, cy + yOffset + 30);
}
```

The shipped line is `this.ctx.font = Constants.DASHBOARD_UNIT_COLOR;`, i.e.
`'#4E4E4E'` fed to the font parser. Canvas ignores an unparseable font string,
so the unit keeps the previous 38vp value font and inherits the value's
`fillStyle`; the `DASHBOARD_UNIT_FONT` assigned on the line above is
immediately thrown away. It is a one-word fix and a good reminder that canvas
state is a machine you set, not a call you make.

**Subscribing, downloading, cleaning up — `entry/src/main/ets/pages/SpeedTest.ets`** (corrected, see `HW-15-0020`)

```typescript
private onNetworkQos = async () => {
  try {
    netQuality.on('netQosChange', (list: Array<netQuality.NetworkQos>) => {
      if (list.length > 0) {
        list.forEach((qos) => {
          this.currentValue = qos.linkDownRate / Constants.MBPS;   // bit/s -> Mbps
          this.maxRate = Math.max(this.maxRate, this.currentValue);
        });
      }
    });
  } catch (err) {
    hilog.error(0x0000, 'NetWork', 'error: %{public}d %{public}s',
      (err as BusinessError).code, (err as BusinessError).message);
  }
};

private getNetworkQosOff = async () => {
  try {
    netQuality.off('netQosChange');
  } catch (err) {
    let e: BusinessError = err as BusinessError;
    hilog.error(0x0000, 'testTag', 'off netQosChange error: %{public}d %{public}s', e.code, e.message);
  }
};

private startSpeedTest() {
  if (this.isRunning) {
    this.getUIContext().getPromptAction().showToast({ message: $r('app.string.speed_running_message') });
    return;
  }
  this.isRunning = true;
  this.onNetworkQos();

  let context = this.getUIContext().getHostContext() as common.UIAbilityContext;
  let filePath = `${context.cacheDir}/tmp.zip`;
  FILE_UTILS.removeFile(filePath);          // FIX: absent - a stale tmp.zip completes the test instantly
  downLoadFileByRequest({
    context: context,
    targetUrl: this.downloadUrl,
    filePath: filePath,
    onComplete: (): void => {
      FILE_UTILS.removeFile(filePath);
      this.getNetworkQosOff();
      this.currentValue = 0;
      setTimeout(() => {
        this.dialogControllerConfirm.open();
      }, Constants.ANIMATION_DURATION);
    },
    onError: (): void => {                  // FIX: no failure callback exists in the sample
      this.getNetworkQosOff();
      this.isRunning = false;
      this.currentValue = 0;
    }
  });
}

aboutToDisappear(): void {                  // FIX: absent - leaving mid-test leaks the subscription
  this.getNetworkQosOff();
}
```

**`netQuality` is a push subscription, not a poll.** You do not measure the
download yourself; you start traffic and the system reports link quality as it
changes. `linkDownRate` is in bit/s, hence the `MBPS = 1000000` divisor. The
consequence is that the subscription's lifetime, not the download's, is what
you must manage: every path that ends a test has to call
`netQuality.off('netQosChange')`, including page teardown.

The shipped code has exactly one such path. `DownloadUtil`'s `.catch` only
logs, so a rejected `request.downloadFile` (bad URL - and the sample ships
`'https://xxx.huawei.com/xxx.zip'` as a TODO placeholder) leaves `isRunning`
`true` forever: it is cleared only in the report dialog's confirm action, and
that dialog is opened only from `onComplete`. The button stays at 0.4 opacity
showing 测速中 for the rest of the process's life. Adding an `onError` member
to `DownLoadConfig` and calling it from both the `catch` and the `'fail'`
event is the minimum fix; the `removeFile` before starting closes the second
half of the same finding.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.GET_NETWORK_INFO" },
  { "name": "ohos.permission.INTERNET" }
]
```

Both are `system_grant` (normal level), so no `reason`/`usedScene` is needed
and no runtime request appears - which is why this sample has no permission
code at all. `INTERNET` is for `request.downloadFile`; `GET_NETWORK_INFO` is
what `netQuality` needs to report link state.

`EntryAbility` sets `COLOR_MODE_LIGHT` on the application context, then goes
full screen and publishes `topRectHeight` / `bottomRectHeight` into
`AppStorage` from `getWindowAvoidArea` plus an `avoidAreaChange` listener; the
page consumes them with `@StorageProp` and `px2vp` into padding. The listener
is never unregistered in `onWindowStageDestroy` and
`setWindowLayoutFullScreen` is fire-and-forget - the same avoid-area
boilerplate defect that recurs across these industry samples.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `deviceTypes` is `["phone"]` only.
- **The sample does not run out of the box.** `downloadUrl` is
  `'https://xxx.huawei.com/xxx.zip'` behind a `TODO` comment; the document
  says so too. Without a real large file there is no traffic and `netQuality`
  reports nothing.
- The gauge assumes a square canvas. `aspectRatio(1)` on the component is not
  cosmetic - drop it and the dial is measured from the width but drawn into a
  non-square box.
- The value text shows `value` while the needle shows the tween, so the digits
  do not glide with the needle.
- `maxValue` is fixed at 200 Mbps by the page. A faster link pins the needle:
  `drawProgressRing` and `drawPointer` clamp `currentValue` to `maxValue`.
- The two bottom tabs are decorative - the whole `Row` has one `onClick` that
  raises a "sample only" toast.

## Pitfalls

- **`HW-15-0020`** (B/medium, confirmed): the download failure path skips all
  cleanup - `DownloadUtil`'s `.catch` only logs, so `isRunning` is never
  reset (it is cleared only in the report dialog's confirm) and
  `getNetworkQosOff` is never called; a leftover `tmp.zip` from a run killed
  mid-download makes `accessSync` succeed and fires `onComplete` immediately
  with a 0 Mbps report; and `SpeedTest` has no `aboutToDisappear`, so leaving
  the page mid-test leaks both the subscription and the download. Fix: run the
  cleanup from `catch`/`finally` and from an explicit error callback, and
  unlink `tmp.zip` before starting.
- **`HW-15-0021`** (B/low, confirmed): `drawValueText` assigns the colour
  constant `DASHBOARD_UNIT_COLOR` (`'#4E4E4E'`) to `ctx.font`, so the `Mbps`
  label keeps the 38vp value font and the value's fill colour. Fix: assign it
  to `fillStyle`.

## References

- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - the `Canvas` component and `onReady`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/02_application-framework/ts-canvasrenderingcontext2d.md` - `arc`, `createConicGradient`, `fillStyle`/`font`, `translate`/`rotate`/`save`/`restore`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-canvasrenderingcontext2d
- `documentation/harmonyos-references/02_application-framework/js-apis-animator.md` - `createAnimator`, `AnimatorOptions`, `AnimatorResult.onFrame`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-animator
- `documentation/harmonyos-references/03_system/networkboost-netquality.md` - `netQosChange`, `NetworkQos.linkDownRate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/networkboost-netquality
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `GET_NETWORK_INFO` and `INTERNET` are system_grant
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `UTIL-09` - network awareness, the other consumer of the same NetworkBoost subscription
