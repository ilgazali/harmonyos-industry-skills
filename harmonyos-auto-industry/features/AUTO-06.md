---
id: AUTO-06
title: Custom dashboard gauge drawn on Canvas
industry: 01_auto
doc: huawei_industry_tree/01_auto/docs/06_customize_dashboard.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/customize_dashboard-0000002369250730
sample: huawei_industry_tree/01_auto/downloads/CustomizeDashboard.zip
kits: ["@kit.ArkUI", "@kit.ArkTS"]
apis: [Canvas, CanvasRenderingContext2D, RenderingContextSettings, beginPath, arc, stroke, fill, fillText, createConicGradient, createRadialGradient, ImageBitmap, drawImage, clearRect, getComponentUtils, getRectangleById, Decimal]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-01-0036, HW-01-0037, HW-01-0038, HW-01-0039, HW-01-0040, HW-01-0041, HW-01-0042, HW-01-0050]
status: verified-with-fixes
---

## When to use

Load this card for any **circular gauge or radial progress readout drawn by
hand**: speedometer, tachometer, battery or charge level, tyre pressure, range
remaining, a fitness ring. The technique is `Canvas` plus
`CanvasRenderingContext2D` arcs with conic and radial gradients - reach for it
when the built-in `Progress` and `Gauge` components cannot express the design.

If a plain arc with one colour would do, use `Progress({ type: ProgressType.Ring })`
instead. Hand-drawn canvas costs you the repaint discipline described below.

## Feature checklist

- A large dial: a static background track, a value arc that fills
  proportionally, an inner ring, and a soft radial glow.
- A conic gradient sweeping the value arc through several colours.
- The numeric value and its unit centred in the dial.
- A second, smaller dial for a secondary measure.
- Values clamped at a configured maximum.
- Redraw when the value changes, and only then.

## Architecture

Single `entry` module.

```
entry/src/main/ets
├── entryability/EntryAbility.ets
├── model/GridInfo.ets       grid content below the gauge
├── pages/MainPage.ets       owns the values, @Provide's them
└── utils/Dashboard.ets      the gauge component
```

The page owns four `@Provide` values - `speed`, `acceleration`, `maxSpeed`,
`maxAcceleration`. `Dashboard` takes them with `@Consume` and puts
`@Watch('draw')` on the two that change, so any write to a value repaints the
canvas.

```typescript
@Consume maxSpeed: number;
@Consume @Watch('draw') speed: number;
@Consume maxAcceleration: number;
@Consume @Watch('draw') acceleration: number;
```

> Note the cost: **two watched values means two `draw()` calls when both are
> written in the same tick.** The sample's demo timer writes both every 100 ms,
> so the canvas repaints twenty times a second. Everything inside `draw()` is
> therefore on a hot path - see `HW-01-0036` and `HW-01-0037`.

`draw()` is the single entry point and paints the whole gauge in order:
background track → value arc → inner ring → glow and centre image → speed text →
small dial → small glow → acceleration text. `clearRect` first.

## Implementation steps

1. **Create the context once as a member**:
   `new CanvasRenderingContext2D(new RenderingContextSettings(true))`. The
   `true` enables anti-aliasing, which a gauge needs.
2. **Measure the canvas once**, from `onReady`, and cache it - do not re-query
   the rectangle on every frame.
3. **Load any bitmap once**, in `aboutToAppear`, and close it in
   `aboutToDisappear` (`HW-01-0037`).
4. **Write a `degToRad` helper and use it for every angle** - `arc` takes
   radians (`HW-01-0039`).
5. **Keep per-shape geometry local.** Do not let one draw method write instance
   fields another one reads (`HW-01-0038`).
6. **Express every coordinate as a fraction of the measured canvas size**,
   including text (`HW-01-0040`).
7. **Clamp the value with a plain `Math.min`**, not a copy-pasted branch
   (`HW-01-0041`).
8. **Own the redraw driver's lifetime.** If a timer feeds the gauge, store its
   id and clear it in `aboutToDisappear` (`HW-01-0036`).

## Verified snippets

All snippets are from `CustomizeDashboard.zip`. Corrected forms are marked.

**Component skeleton — `utils/Dashboard.ets`** (as shipped)

```typescript
@Component
export struct Dashboard {
  @State canvasSize: number = 0;
  private settings: RenderingContextSettings = new RenderingContextSettings(true);
  private context: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings);
  @Consume maxSpeed: number;
  @Consume @Watch('draw') speed: number;

  degToRad(degree: number) {
    return degree * Math.PI / 180;
  }

  build() {
    Column() {
      Canvas(this.context)
        .zIndex(1)
        .id('canvas')
        .height('50%')
        .onReady(() => {
          this.draw();
        });
    };
  }
}
```

`.id('canvas')` is what makes the size measurable, and `onReady` is the earliest
point at which the context can be drawn to.

**Measuring the canvas** (corrected, see `HW-01-0037`)

```typescript
// FIX: the shipped draw() re-runs this measurement on every repaint.
onReady(() => {
  this.canvasSize = this.getUIContext()
    .px2vp(this.getUIContext().getComponentUtils().getRectangleById('canvas').size.width);
  this.bgImage = new ImageBitmap('resources/base/media/img.png');
  this.draw();
})

aboutToDisappear(): void {
  this.bgImage?.close();
}
```

`getRectangleById` returns **px**, so `px2vp` is required before using the value
as a canvas coordinate - the canvas coordinate space is vp.

**The value arc with a conic gradient — `utils/Dashboard.ets`** (corrected, see `HW-01-0038` and `HW-01-0041`)

```typescript
private static readonly SWEEP_DEG: number = 310;   // total travel of the dial

drawCircleMiddle(centerx: number, centery: number) {
  const grad = this.context.createConicGradient(this.degToRad(70), centerx, centery);
  grad.addColorStop(0.0, '#FE9AE3');
  grad.addColorStop(0.25, '#A19FFF');
  grad.addColorStop(0.5, '#58B0F5');
  grad.addColorStop(0.9, '#ce6e6dd7');

  const radius = this.canvasSize * 0.293;     // FIX: shipped code writes this.radius
  this.context.lineWidth = 18;
  this.context.strokeStyle = grad;
  this.context.beginPath();

  // FIX: shipped code duplicates the arc in an else branch whose expression
  // (maxSpeed * 310 / maxSpeed) cancels to 310.
  const ratio = Math.min(this.speed, this.maxSpeed) / this.maxSpeed;
  this.context.arc(centerx, centery, radius,
    this.degToRad(Dashboard.START_DEG),
    this.degToRad(Dashboard.START_DEG + ratio * Dashboard.SWEEP_DEG + 0.1));
  this.context.stroke();
}
```

`createConicGradient(startAngle, x, y)` is the piece worth remembering: it
sweeps colour around the centre so the arc changes hue along its length, which
is what makes a gauge look like a gauge rather than a coloured ring.

The `+ 0.1` is deliberate - it keeps the arc from collapsing to nothing when
`speed` is 0, where a zero-length arc would render as a dot artefact.

**Full circles** (corrected, see `HW-01-0039`)

```typescript
// FIX: the shipped drawBigFill and drawSmallFill pass 0, 360 - degrees into an
// API that takes radians. drawCircleSmall in the same file does it correctly:
this.context.arc(centerx, centery, radius, this.degToRad(0), this.degToRad(360));
```

**Radial glow — `utils/Dashboard.ets`** (as shipped)

```typescript
const grad = this.context.createRadialGradient(
  centerx, centery, this.canvasSize * 0.01,     // inner circle
  centerx, centery, this.canvasSize * 0.22);    // outer circle
grad.addColorStop(0.0, 'rgb(201, 218, 254)');
grad.addColorStop(0.75, 'rgba(52, 104, 254, 0.3)');
grad.addColorStop(1.0, 'rgba(164, 182, 219, 0)');
this.context.fillStyle = grad;
```

Fading the last stop to alpha 0 is what makes the glow blend out instead of
ending in a hard edge.

**Text with a decimal-aware offset — `utils/Dashboard.ets`** (as shipped)

```typescript
import { Decimal } from '@kit.ArkTS';

drawAcceleration() {
  this.context.fillStyle = Color.Black;
  this.context.font = 'normal normal 17vp sans-serif';
  this.context.fillText('m/s²', this.canvasSize * 0.71, 310);
  this.context.font = 'normal 500 30vp sans-serif';
  if (new Decimal(this.acceleration).decimalPlaces() > 0 && this.acceleration <= this.maxAcceleration) {
    this.context.fillText(String(this.acceleration), this.canvasSize * 0.695, 290);
  } else if (new Decimal(this.acceleration).decimalPlaces() === 0 && ...) {
    this.context.fillText(String(this.acceleration), this.canvasSize * 0.725, 290);
  }
}
```

Two things worth taking: the `font` string follows the CSS shorthand
(`style weight size family`) and accepts **vp** units directly; and
`Decimal.decimalPlaces()` from `@kit.ArkTS` is used to nudge the x offset so a
value with a decimal point stays optically centred. The hardcoded y values are a
defect - see `HW-01-0040`.

**Driving the gauge — `pages/MainPage.ets`** (corrected, see `HW-01-0036`)

```typescript
@Provide speed: number = 0;
@Provide acceleration: number = 0;
@Provide maxSpeed: number = 120;
@Provide maxAcceleration: number = 5;

private timeoutId: number = -1;
private intervalId: number = -1;

aboutToAppear(): void {
  this.timeoutId = setTimeout(() => {
    this.intervalId = setInterval(() => {
      this.speed += 5;
      this.acceleration += 0.5;
    }, 100);
  }, 1500);
}

aboutToDisappear(): void {     // FIX: absent from the shipped sample
  clearTimeout(this.timeoutId);
  clearInterval(this.intervalId);
}
```

In a real app, replace the timer with the vehicle data source. Whatever the
source, the point stands: **one write per frame, not two**, or the canvas
repaints twice for every update.

## Permissions & config

**None.** `entry/src/main/module.json5` declares no `requestPermissions`.

The centre image lives at `entry/src/main/resources/base/media/img.png` and is
loaded by path string through `ImageBitmap`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `CanvasRenderingContext2D.arc` takes **radians**.
- `getRectangleById` returns **px**; canvas coordinates are **vp**.
- The canvas is not data-bound: nothing repaints unless a `@Watch` fires, so
  every input that affects the drawing must be watched.
- The gauge is a fixed 310° sweep starting at 70°; changing the dial's span
  means changing both constants together.

## Pitfalls

- **`HW-01-0036` — the demo timer can never be stopped.** Neither
  `setTimeout` nor `setInterval` returns into a variable, and there is no
  `aboutToDisappear`, so it keeps mutating `@Provide` state after the page dies.
  Writing two watched values per tick also doubles the repaint rate.
- **`HW-01-0037` — an `ImageBitmap` is constructed inside a draw routine,** so
  a constant image is decoded, drawn and closed twenty times a second, and the
  canvas rectangle is re-measured just as often.
- **`HW-01-0038` — `drawCircleInside` writes `startAngle` and `endAngle`,**
  which the two arcs drawn before it read. The first frame uses 70/20, every
  frame after uses 69/21, so the dial shifts a degree and stays there.
- **`HW-01-0039` — `arc(..., 0, 360)` passes degrees into a radian API,** twice,
  in a file that has a `degToRad` helper and uses it correctly elsewhere. It
  looks right only because a 360-radian sweep still closes the circle.
- **`HW-01-0040` — text is positioned with relative x and absolute y.** The dial
  scales with the canvas, the readouts do not.
- **`HW-01-0041` — clamp branches contain `max * 310 / max`,** which is just
  310. Both the speed and the acceleration dial do it, and the document prints
  one of them as its only code sample.
- **`HW-01-0042` — the `stroke` link in 实现思路 leaves the HarmonyOS
  references** for the atomic-service documentation, while `beginPath` and `arc`
  in the same sentence do not.

## References

- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - the `Canvas` component
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/02_application-framework/ts-canvasrenderingcontext2d.md` - `arc`, `fillText`, `font`, gradients; angles are in radians
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-canvasrenderingcontext2d
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `getComponentUtils`, `px2vp`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `AUTO-01` - the industry architecture this component drops into
