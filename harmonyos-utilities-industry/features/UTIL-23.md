---
id: UTIL-23
title: Spirit level - orientation beta/gamma folded into a bounded Canvas bubble
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/23_spirit_level.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/spirit_level-0000002362176089
sample: huawei_industry_tree/15_utilities/downloads/SpiritLevel.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit", "@kit.SensorServiceKit"]
apis: [hilog, sensor, window, "sensor.on", "sensor.SensorId.ORIENTATION", OrientationResponse, Canvas, CanvasRenderingContext2D, RenderingContextSettings, "@StorageProp"]
permissions: []
min_api: 17
modules: [entry (entry)]
findings: [HW-15-0003, HW-15-0004, HW-15-0101]
status: verified-with-fixes
---

## When to use

**Load this card when a continuous sensor value has to become a position on a
drawn surface** - a bubble in a circle here, but equally a crosshair on a
target, a dot on a radar, a horizon line in an artificial horizon.

Three problems recur in that shape, and this sample answers all three in about
forty lines: how to stop a jittery sensor from repainting on noise (a
dead-band threshold), how to keep a two-axis value inside a *circular* bound
rather than a square one (clamp then scale by the radius ratio), and how to
fold an angle that wraps at ±180° into a usable ±90° tilt.

The one thing not to copy is the lifecycle. This sample is the original anchor
of the sensor-leak family in this industry (`HW-15-0004`, later recurring as
`HW-15-0055` in `UTIL-22`): it subscribes in `aboutToAppear` and never
unsubscribes. Wire the teardown before you copy anything else.

## Feature checklist

- A full-screen page titled 水平仪 with a 280×280 canvas below it.
- Four concentric circles drawn on the canvas, the outermost heavier and blue,
  the inner three thin and light blue.
- A green bubble with two white highlight arcs, drawn at an offset from the
  centre proportional to device tilt.
- Tilting the device forward/back moves the bubble vertically; tilting
  left/right moves it horizontally.
- Laying the device flat brings the bubble to the exact centre.
- The bubble never leaves the outer circle, including on a diagonal tilt.
- A large button below shows the two live angles, `X:` from roll and `Y:` from
  pitch, one decimal each.
- Movements below 0.1° are ignored, so a still device does not shimmer.

## Architecture

One `entry` module, one page. There is no controller, no model, no viewmodel -
the whole feature is a sensor callback and a draw function on the same struct.

```
entry/src/main/ets
├── entryability/EntryAbility.ets   window setup, avoid areas -> AppStorage
├── entrybackupability/
└── pages/MainPage.ets              @Entry: constants, sensor.on, clamping, onDraw - 166 lines
```

The documented tree matches the zip.

The geometry lives in eight file-level constants above the struct, and the
relationship between them is the interesting part:

```typescript
const CENTER_X = 140;
const CENTER_Y = 140;
const MAX_RADIUS = 135;
const BUBBLE_RADIUS = 20;
const CIRCLES_NUM = 4;
const OUTER_LINE_WIDTH = 4;
const INNER_LINE_WIDTH = 2;
const MAX_OFFSET = MAX_RADIUS - BUBBLE_RADIUS - OUTER_LINE_WIDTH / 2;
```

**The design decision worth copying** is `MAX_OFFSET` being *derived* rather
than tuned. It is the outer radius minus the bubble's own radius minus half
the stroke width - which is exactly the distance at which the bubble's edge
kisses the inside of the outer ring. Change `BUBBLE_RADIUS` or the ring weight
and the containment still holds. Every other value in the file is a raw
literal; this one is the only piece of geometry that had to be right, and it
is the only one expressed as a formula.

The decision **worth avoiding** is drawing straight from the sensor callback.
`updateBubblePosition()` ends with `context.reset()` and a full `onDraw`, so at
`interval: 1000000` (1 ms requested) the canvas is cleared and four arcs plus
three strokes are re-issued on every accepted sample. The 0.1° threshold is
what keeps that survivable; without it the page would repaint continuously.

## Implementation steps

1. **Create the rendering context once as a field**:
   `new CanvasRenderingContext2D(new RenderingContextSettings(true))`. The
   `true` enables anti-aliasing, which the bubble's highlight arcs need.
2. **Subscribe the orientation sensor in `aboutToAppear`** and take `beta` as
   pitch and `gamma` as roll. `alpha` (heading) is unused here - that is the
   compass value, see `UTIL-22`.
3. **Reject samples inside a dead band** before touching any state: if both
   axes moved less than 0.1°, return.
4. **Fold the raw angle into ±90°** with the `getOrigin` helper - past 90° the
   device is tilting past vertical and the useful measure starts coming back
   down.
5. **Scale degrees to pixels** by `angle / 90 * MAX_OFFSET`, so a 90° tilt puts
   the bubble exactly on the rim.
6. **Clamp per axis, then rescale radially.** Clamping alone leaves a square
   bound; the radial pass is what makes the constraint circular.
7. **Reset and redraw the whole canvas** - `context.reset()` then `onDraw` -
   rather than trying to erase the previous bubble.
8. **Draw the rings in a loop**, giving the last iteration the heavier stroke
   and the blue colour, so `CIRCLES_NUM` remains a single knob.
9. **Unsubscribe in `aboutToDisappear`** (`HW-15-0004`). The sample has no
   teardown hook at all.
10. **State the real API floor.** `build-profile.json5` says `5.0.5(17)`; the
    document claims API 20 (`HW-15-0003`).

## Verified snippets

All snippets are from `SpiritLevel.zip`. Corrected forms are marked.

**The subscription — `entry/src/main/ets/pages/MainPage.ets`** (corrected, see `HW-15-0004`)

```typescript
import { sensor } from '@kit.SensorServiceKit';

private pitch: number = 0;
private roll: number = 0;
private threshold: number = 0.1;
private settings: RenderingContextSettings = new RenderingContextSettings(true);
private context: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings);

aboutToAppear(): void {
  // 注册方向传感器监听
  sensor.on(sensor.SensorId.ORIENTATION, (data) => {
    let pitch = data.beta;   // 俯仰角
    let roll = data.gamma;   // 侧倾角
    if (Math.abs(pitch - this.pitch) < this.threshold && Math.abs(roll - this.roll) < this.threshold) {
      return;
    }
    this.pitch = pitch;
    this.roll = roll;
    this.originBubbleX = this.getOrigin(roll);
    this.originBubbleY = this.getOrigin(pitch);
    // 计算气泡位置（根据屏幕尺寸调整比例）
    this.bubbleX = this.originBubbleX / 90 * MAX_OFFSET;
    this.bubbleY = this.originBubbleY / 90 * MAX_OFFSET;
    this.updateBubblePosition();
  }, { interval: 1000000 });
}

aboutToDisappear(): void {
  sensor.off(sensor.SensorId.ORIENTATION);   // FIX: absent in the sample
}

private getOrigin(data: number) {
  let absData = Math.abs(data);
  if (absData <= 90) {
    return data;
  }
  return (180 - absData) * Math.sign(data);
}
```

**Three lines carry the design.** The threshold test comes first and returns
before any assignment, so a still device produces no state write and no
repaint - this is the difference between a level that sits calmly and one that
vibrates. Note that it is an AND: the sample only skips when *both* axes are
quiet, so a deliberate single-axis movement is never swallowed.

`getOrigin` is the fold. `OrientationResponse.beta` and `gamma` run to ±180°,
but a spirit level only has meaning up to vertical: at 120° of pitch the
surface is 60° past flat coming back the other way, so the value must mirror
around 90°. `(180 - absData) * Math.sign(data)` does that while preserving
which way the device leans. Without it, a device tilted past vertical would
send the bubble racing back toward the centre.

The interval is `1000000` ns - 1 ms - which is far faster than the display and
far faster than a human can tilt a phone. It is the threshold, not the
interval, that governs the real repaint rate. A slower interval (10-20 ms, as
`UTIL-22` uses) would be the better first knob.

Two `@State` pairs exist for a reason: `originBubbleX/Y` hold **degrees** and
feed the button's readout, `bubbleX/Y` hold **pixels** and feed the canvas.
Keeping them apart is why the clamping below can mutate the pixel values
without corrupting the displayed angles.

**Bounding the bubble to a circle — same file** (as shipped)

```typescript
updateBubblePosition() {
  // 限制气泡在圆盘范围内
  this.bubbleX = Math.min(Math.max(this.bubbleX, -MAX_OFFSET), MAX_OFFSET);
  this.bubbleY = Math.min(Math.max(this.bubbleY, -MAX_OFFSET), MAX_OFFSET);
  let currentDistance = Math.sqrt(this.bubbleX**2 + this.bubbleY**2);
  // 如果超出圆形范围，按比例缩放坐标
  if (currentDistance > MAX_OFFSET) {
    let scale = MAX_OFFSET / currentDistance;
    this.bubbleX *= scale;
    this.bubbleY *= scale;
  }
  this.context.reset();
  this.onDraw(this.context);
}
```

**The two clamps are not enough on their own, and the sample knows it.** Per-
axis clamping bounds the bubble to a square of side `2 * MAX_OFFSET`; at 45°
of tilt on both axes the corner of that square is `MAX_OFFSET * √2` from the
centre, roughly 1.41 radii out - the bubble would sit well outside the ring.
The radial pass rescales the vector to length `MAX_OFFSET` while preserving
its direction, which is the correct projection onto a disc.

The axis clamps are still worth keeping ahead of it: they bound the magnitude
before the square, which keeps `currentDistance` finite if a sensor returns
something absurd.

`context.reset()` before `onDraw` is the honest choice for a canvas this
small. There is no dirty-region tracking, no double buffer - the whole 280×280
surface is repainted, which at four `arc` strokes plus two highlight arcs is
cheap and completely predictable.

**Drawing the dial and the bubble — same file** (as shipped)

```typescript
onDraw(context: CanvasRenderingContext2D) {
  // 绘制背景
  for (let i = 0; i < CIRCLES_NUM; i++) {
    let radius = (i + 1) * (MAX_RADIUS / CIRCLES_NUM);
    context.beginPath();
    context.arc(CENTER_X, CENTER_Y, radius, 0, Math.PI * 2);
    context.strokeStyle = i === CIRCLES_NUM - 1 ? Color.Blue : 'rgba(135, 206, 250, 1.00)';
    context.lineWidth = i === CIRCLES_NUM - 1 ? OUTER_LINE_WIDTH : INNER_LINE_WIDTH;
    context.stroke();
  }
  // 绘制气泡
  context.beginPath();
  context.arc(CENTER_X + this.bubbleX, CENTER_Y + this.bubbleY, BUBBLE_RADIUS, 0, Math.PI * 2);
  context.fillStyle = 'rgb(100, 187, 92)';
  context.fill();
  // 添加左上白色弧形
  context.beginPath();
  context.arc(
    CENTER_X + this.bubbleX,
    CENTER_Y + this.bubbleY,
    BUBBLE_RADIUS * 0.9,     // 稍小于原圆半径
    Math.PI * 1,             // 起始角度
    Math.PI * 1.05,          // 结束角度
    false                    // 顺时针绘制
  );
  context.strokeStyle = 'white';
  context.lineWidth = 2;
  context.lineCap = 'round';
  context.stroke();
  // ... a second arc from 1.12π to 1.5π, same style
}
```

**`beginPath()` before every shape is mandatory, not stylistic.** Without it
each `arc` appends to the current path and the following `stroke()` or `fill()`
redraws every previous shape with the *current* style - the rings would all
end up blue and heavy, and the highlight arcs would repaint the bubble outline
in white.

The rings are drawn from the inside out so the heaviest, darkest stroke lands
last and is not overdrawn by a thinner neighbour. The ternaries on
`strokeStyle` and `lineWidth` keep the whole dial parameterised by
`CIRCLES_NUM` - raise it to 6 and the spacing, colours and weights all follow.

The two white arcs are the entire "glass bubble" illusion: a short arc near π
(the left edge) and a longer one from 1.12π to 1.5π (upper left), both at 0.9
of the bubble radius with `lineCap = 'round'`. They are drawn at the bubble's
current position, not the centre, so the highlight travels with it.

**Where the readout comes from — same file** (as shipped)

```typescript
Canvas(this.context)
  .onReady(() => {
    this.onDraw(this.context);
  })
  .width(280)
  .height(280)
  .backgroundColor($r('app.color.start_window_background'))
  .margin({ top: 90 });

Button(`X:${this.originBubbleX.toFixed(1)}°       Y:${(-this.originBubbleY).toFixed(1)}°`)
  .width(270)
  .height(68)
  .type(ButtonType.Normal)
  .borderRadius(16)
  .fontSize(26)
  .margin({ top: 44 });
```

**`onReady` is the only safe place for the first draw.** The canvas has no
backing surface until it fires, so calling `onDraw` from `aboutToAppear` would
paint into nothing; the sensor callback then keeps it current. The `Y` value is
negated for display because screen Y grows downward while a nose-up pitch
should read positive - the same sign flip that would otherwise have to be
buried in `getOrigin`.

The readout is a `Button` purely for its shape and elevation; it has no
`onClick`. That is worth flagging as a copy hazard: a non-interactive element
styled as a button is an accessibility problem in anything more serious than a
demo. A `Text` inside a rounded `Column` gives the same look without claiming
to be actionable.

## Permissions & config

**None.** The sample declares no `requestPermissions` at all, and it does not
need to: `sensor.SensorId.ORIENTATION` is in the unrestricted set. Only the
motion-sensor family (`ACCELEROMETER`, `GYROSCOPE`, and derivatives) requires
`ohos.permission.ACCELEROMETER` / `ohos.permission.GYROSCOPE`, and this sample
touches neither.

`deviceTypes` is `["phone"]`, which is right for a tool the user physically
lays on a surface.

## Constraints

- **The document and the sample disagree on the API floor** (`HW-15-0003`).
  The 约束与限制 section claims API Version 20 Release and HarmonyOS 6.0.0
  Release SDK; `build-profile.json5` sets `compatibleSdkVersion: "5.0.5(17)"`.
  Trust the build profile - nothing in this code needs API 20.
- Requires real hardware. The orientation sensor is not simulated on the
  emulator, so the bubble stays at the centre and the readout at 0.0°.
- The canvas geometry is fixed in vp: a 280×280 component with a 135 vp outer
  radius drawn at `CENTER_X/Y = 140`. Change the component size and the
  drawing no longer fits; there is no measurement pass.
- No calibration and no reference-surface offset - the level reads absolute
  device tilt, so a phone with a camera bump lying on its back reads a degree
  or two off flat.
- Accuracy is whatever the fused orientation sensor gives, typically ±1°;
  this is a demo of the pattern, not a substitute for an instrument.
- `interval: 1000000` requests 1 ms sampling. The framework may not honour it,
  and it is faster than anything the UI can use - the 0.1° dead band is doing
  the real rate limiting.

## Pitfalls

- **`HW-15-0004`** (B/medium, confirmed): the orientation subscription at
  `MainPage.ets:44` is never released - there is no `sensor.off` anywhere in
  the sample and no `aboutToDisappear`. A continuously sampling hardware sensor
  left subscribed after page destruction drains battery and fires callbacks
  into detached state. This is the anchor instance of the class; `UTIL-22`
  repeats it with four streams at once (`HW-15-0055`). Fix: add
  `aboutToDisappear() { sensor.off(sensor.SensorId.ORIENTATION); }`.
- **`HW-15-0003`** (E/low, confirmed): 约束与限制 states "本示例支持API
  Version 20 Release及以上版本" while `build-profile.json5` targets
  `5.0.5(17)` - the constraint overstates the requirement by three API
  versions against the sample's own configuration. Fifth instance of this
  class corpus-wide. Fix: align the constraint with `5.0.5(17)`, or bump the
  sample.
- Not filed, but adjacent: the dead-band test is an AND across both axes, so a
  device rotating in a perfect diagonal at just under 0.1° per sample on each
  axis is ignored even though its total movement exceeds the threshold. An OR,
  or a test on the combined magnitude, is the stricter form.
- The readout `Button` has no `onClick` handler; it is a label wearing a
  button's clothes.

## References

- `documentation/harmonyos-references/03_system/js-apis-sensor.md` -
  `sensor.on(SensorId.ORIENTATION)`, `OrientationResponse` (`alpha`, `beta`,
  `gamma`), the `interval` option, and `sensor.off`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-sensor
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` -
  the `Canvas` component and `onReady`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/02_application-framework/ts-canvasrenderingcontext2d.md` -
  `arc`, `beginPath`, `stroke`, `fill`, `lineCap`, `reset`, and
  `RenderingContextSettings(antialias)`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-canvasrenderingcontext2d
- `UTIL-22` - the compass: the same sensor, the `alpha` axis instead, and the
  same missing teardown at four times the cost
