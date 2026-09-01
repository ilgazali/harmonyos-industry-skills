---
id: UTIL-04
title: Azimuth and included-angle dial - a draggable Canvas sector with three grab targets
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/04_control_sector.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/control_sector-0000002238010112
sample: huawei_industry_tree/15_utilities/downloads/ControlSector.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [hilog, window, Canvas, CanvasRenderingContext2D, RenderingContextSettings, PanGesture, onActionStart, onActionUpdate, createRadialGradient, arc, clearRect, "Math.atan", "Math.acos", "@Builder", Stack, Divider]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0018, HW-15-0019, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card when the user has to **aim something on a map or a plan** - a
camera's field of view, a floodlight cone, a radar bearing, a wind sector, a
"search within this direction" filter. The control is a filled sector on a
`Canvas` with two draggable edges: drag one edge to change the included angle,
drag the body to rotate the whole sector without changing its width.

The transferable part is the interaction model, not the drawing. One
`PanGesture` on the container handles all three gestures, because the *start*
of the pan decides which of them it is: `onActionStart` classifies the touch
point into "on edge one", "on edge two", "inside the sector" or "outside",
stores that verdict in a flag, and `onActionUpdate` applies the same rotation
delta to whichever angles the flag selected. That one-gesture-many-targets
shape is how you avoid stacking overlapping gesture recognisers on small hit
areas.

The maths is the other half: every angle in this component lives in
[−π, π], and **the wraparound at ±π is where the sample breaks**
(`HW-15-0018`). Read that finding before adopting the hit test.

## Feature checklist

- A sector is drawn over a background image, filled translucent green, with
  both radii stroked using a radial gradient that fades outward.
- Dragging near either radius moves that edge only; the sector opens or closes.
- Dragging inside the sector body rotates both edges together, preserving the
  included angle.
- Dragging outside the circle does nothing.
- A card at the top shows 方位角 (azimuth) and 夹角 (included angle) in degrees
  to two decimals, updating on every frame of the drag.
- The included angle readout always reports the sector's smaller side (≤ 180°);
  the fill follows the same rule.

## Architecture

One `entry` module, one component, one constants file.

```
entry/src/main/ets
├── common/Constants.ets                  geometry, colours, and the JUDGE_* enum values
├── components/ControlSector.ets          the whole feature: canvas, gesture, maths, readout
├── entryability/EntryAbility.ets         full-screen layout, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
└── pages/MainPage.ets                    @Entry, a Stack of background image + ControlSector
```

The documented tree matches the zip. `MainPage` is eighteen lines - an
`Image` and `ControlSector()` inside a `Stack`. Everything else is in one
265-line component split into two `@Builder`s (`FanShaped`, `Display`) and
five methods (`Angle2PI`, `JudgeLocation`, `AngleOfRotation`, `Draw`,
`setArcAngle`/`setArcDirection`).

**The design decision worth copying** is the separation between *classify on
start* and *apply on update*. `onActionStart` captures three things: the two
current edge angles (`currentOneAngle`, `currentTwoAngle`), the touch origin
(`startX`, `startY`), and the classification flag. `onActionUpdate` then
computes one signed rotation - the angle between the origin vector and the
current vector, both measured from the sector's apex - and adds it to the
*captured* angles rather than to the live ones. Accumulating from a snapshot
instead of integrating per-frame deltas is what keeps the drag free of drift:
if the finger returns to where it started, the sector returns exactly to where
it was.

The decision **worth avoiding** is that state and geometry are stored twice.
Each edge has both an angle (`lineOneAngle`) and a cartesian endpoint
(`lineOneX/Y`), and every handler must remember to recompute the second from
the first. The endpoints are pure derivations of the angle and the radius;
holding them as state is four extra `@State` fields that can go stale.

## Implementation steps

1. **Put the `Canvas` and the readout card in a `Stack`** and attach the
   `PanGesture` to the `Stack`, not to the canvas - the readout must not
   swallow drags that start under it.
2. **Draw from `onReady`**, and call `Draw()` again on every gesture update.
   `RenderingContextSettings(true)` enables anti-aliasing; without it the
   radii look ragged.
3. **Keep both edge angles in [−π, π]** through one helper (`Angle2PI`) so
   there is a single place where wrapping is defined.
4. **Classify the touch in `onActionStart`** into edge one / edge two / inside
   / outside, and store the verdict for the duration of the drag.
5. **Normalise every angle difference before comparing it** - both in the
   edge-proximity test and in the inside-the-sector test (`HW-15-0018`). A raw
   subtraction is off by 2π for any touch on the far side of the ±π seam.
6. **Compute the drag as a signed angle between two vectors** from the apex:
   `acos` of the normalised dot product for the magnitude, the sign of the
   cross product for the direction.
7. **Add the drag to the angles captured at start**, never to the live angles.
8. **Choose the arc direction from the sign and size of the difference** so
   the fill is always the ≤ 180° side.
9. **Derive both readouts from the two angles** on every update - do not store
   the azimuth or the included angle as independent values.
10. **Use the sample's constant names, not the document's**: the documented
    snippet references `Constants.LINE_ONE_CHANGE`, which does not exist
    (`HW-15-0019`); the code uses `JUDGE_ONE`/`JUDGE_TWO`/`JUDGE_THREE`.

## Verified snippets

All snippets are from `ControlSector.zip`. Corrected forms are marked.

**One gesture, three targets — `entry/src/main/ets/components/ControlSector.ets`** (as shipped)

```typescript
build() {
  Stack() {
    this.FanShaped(); // 扇形绘制  (the sector canvas)
    this.Display();   // 上方显示行 (the readout row)
  }
  .width(Constants.FULL_WIDTH)
  .height(Constants.FULL_HEIGHT)
  .gesture(
    PanGesture()
      .onActionStart((event) => {
        this.currentOneAngle = this.lineOneAngle;      // snapshot both edges
        this.currentTwoAngle = this.lineTwoAngle;
        this.startX = event.fingerList[0].localX;
        this.startY = event.fingerList[0].localY;
        this.judgeLocationFlag = this.JudgeLocation(this.startX - this.pointX, this.startY - this.pointY);
      })
      .onActionUpdate((event) => {
        let currentAngle =
          this.AngleOfRotation(this.startX, this.startY, this.startX + event.offsetX, this.startY + event.offsetY);
        if (this.judgeLocationFlag === Constants.JUDGE_ONE) {
          this.lineOneAngle = this.Angle2PI(this.currentOneAngle, currentAngle);
          this.lineOneX = this.radius * Math.cos(this.lineOneAngle) + this.pointX;
          this.lineOneY = this.radius * Math.sin(this.lineOneAngle) + this.pointY;
        } else if (this.judgeLocationFlag === Constants.JUDGE_TWO) {
          this.lineTwoAngle = this.Angle2PI(this.currentTwoAngle, currentAngle);
          this.lineTwoX = this.radius * Math.cos(this.lineTwoAngle) + this.pointX;
          this.lineTwoY = this.radius * Math.sin(this.lineTwoAngle) + this.pointY;
        } else if (this.judgeLocationFlag === Constants.JUDGE_THREE) {
          this.lineOneAngle = this.Angle2PI(this.currentOneAngle, currentAngle);
          this.lineTwoAngle = this.Angle2PI(this.currentTwoAngle, currentAngle);
          this.lineOneX = this.radius * Math.cos(this.lineOneAngle) + this.pointX;
          this.lineOneY = this.radius * Math.sin(this.lineOneAngle) + this.pointY;
          this.lineTwoX = this.radius * Math.cos(this.lineTwoAngle) + this.pointX;
          this.lineTwoY = this.radius * Math.sin(this.lineTwoAngle) + this.pointY;
        }
        this.Draw();
        this.setArcAngle();
        this.setArcDirection();
      })
  );
}
```

**The `JUDGE_THREE` branch is the whole reason for the classify-then-apply
split.** Rotating the body means adding the *same* delta to both edges, and
that only stays coherent if both are measured from the same instant - the
snapshot taken in `onActionStart`. Add the delta to the live values instead
and the second edge drifts relative to the first as the frames accumulate.

`event.fingerList[0].localX` is the touch position in the container's own
coordinate space, which is what makes the fixed apex (`pointX`, `pointY`
= 185, 300) meaningful. `event.offsetX/Y` is the cumulative displacement since
the pan started, so `startX + event.offsetX` reconstructs the current finger
position without tracking it frame by frame.

Note that this component's angles run in canvas coordinates: y grows
downward, so a positive angle turns clockwise on screen.

**The hit test — same file** (corrected, see `HW-15-0018`)

```typescript
// 转换角度为-PI~PI  (normalise to [-PI, PI])
Angle2PI(originAngle: number, addAngle: number) {
  let angle: number = (addAngle + originAngle) % (2 * Math.PI);
  if (angle > Math.PI) {
    angle -= Constants.TRANS_ANGLE;           // TRANS_ANGLE = 2 * Math.PI
  } else if (angle < -Math.PI) {
    angle += Constants.TRANS_ANGLE;
  }
  return angle;
}

// 1/2 = on an edge, 3 = inside the sector, 0 = outside; RANGE = 0.15 rad
JudgeLocation(x: number, y: number) {
  let angle = Math.atan(y / x);
  if (x > Constants.ZERO && y > Constants.ZERO) {
  } else if (x > Constants.ZERO) {
  } else if (y > Constants.ZERO) {
    angle = Math.PI + angle;
  } else {
    angle = -Math.PI + angle;
  }
  if (x * x + y * y <= this.radius * this.radius) {
    let lineAngle: number = this.Angle2PI(this.lineOneAngle, -this.lineTwoAngle);   // FIX: was a raw subtraction
    if (Math.abs(this.Angle2PI(this.lineOneAngle, -angle)) < Constants.RANGE) {     // FIX: wrap before comparing
      return Constants.JUDGE_ONE;
    } else if (Math.abs(this.Angle2PI(this.lineTwoAngle, -angle)) < Constants.RANGE) {
      return Constants.JUDGE_TWO;
    } else if (lineAngle < Constants.ZERO) {
      if (angle > this.lineOneAngle && angle < this.lineTwoAngle) {
        return Constants.JUDGE_THREE;
      }
    } else {
      if (angle < this.lineOneAngle && angle > this.lineTwoAngle) {
        return Constants.JUDGE_THREE;
      }
    }
  }
  return Constants.JUDGE_ZERO;
}
```

**Every angle comparison in a wrapped space must be a wrapped comparison.**
The shipped code writes `Math.abs(this.lineOneAngle - angle) < 0.15`. With the
edge sitting at +3.10 rad and the finger 0.08 rad past the seam at −3.12 rad,
the two are 0.08 rad apart on the dial and 6.22 apart in the arithmetic, so
the edge simply cannot be grabbed anywhere near 180° - the region a bearing
control is used in most. Routing the difference through the existing
`Angle2PI` helper (with the second argument negated, so it computes
`lineOneAngle − angle` normalised) fixes both proximity tests with no new
code.

The same reasoning removed two of the shipped branches. `lineAngle > Math.PI`
and `lineAngle < -Math.PI` were reachable only because the raw difference of
two normalised angles can exceed π; once the difference is itself normalised
into [−π, π] those cases collapse into the two that remain, and the
containment test is just "is the touch angle between the two edges, in the
direction the sign of the difference gives".

One rough edge stays even after the fix: `Math.atan(y / x)` divides by zero
when the touch is exactly on the vertical through the apex, and the
`x > 0`-first ladder then classifies the resulting ±π/2 into the wrong
quadrant branch. `Math.atan2(y, x)` returns the quadrant-correct angle in
[−π, π] directly and makes the entire four-branch ladder unnecessary.

**Signed rotation between two vectors — same file** (as shipped)

```typescript
// 判断拖动过的角度  (how far the drag has turned)
AngleOfRotation(startX: number, startY: number, endX: number, endY: number) {
  let x1 = startX - this.pointX;
  let y1 = startY - this.pointY;
  let x2 = endX - this.pointX;
  let y2 = endY - this.pointY;
  let res = Constants.ZERO;
  if ((x1 === Constants.ZERO && y1 === Constants.ZERO) || (x2 === Constants.ZERO && y2 === Constants.ZERO)) {
    return res;
  } else {
    res = Math.acos((x1 * x2 + y1 * y2) /
      ((Math.sqrt(Math.pow(x1, 2) + Math.pow(y1, 2))) * (Math.sqrt(Math.pow(x2, 2) + Math.pow(y2, 2)))));
  }
  if (x1 * y2 - x2 * y1 > 0) {
    return res;
  } else {
    return -res;
  }
}
```

**`acos` for the size, the cross product for the sign.** `acos` of the
normalised dot product returns an unsigned angle in [0, π] - correct in
magnitude, useless for a drag, because it cannot tell clockwise from
counter-clockwise. The 2-D cross product `x1*y2 − x2*y1` is positive when the
turn from the first vector to the second is counter-clockwise in maths
coordinates, and that single sign is what turns a distance into a rotation.

The zero-vector guard matters: a finger landing exactly on the apex makes both
magnitudes zero and the division `NaN`, which would poison both angles and
blank the canvas for the rest of the session. Returning 0 makes that drag a
no-op instead.

**Filling the smaller side — same file** (as shipped)

```typescript
Draw() {
  this.context.clearRect(Constants.START, Constants.START, Constants.END, Constants.END);
  this.context.beginPath();
  this.context.lineWidth = Constants.LINE_WIDTH;
  let grad = this.context.createRadialGradient(this.pointX, this.pointY, 0, this.pointX, this.pointY, 200);
  grad.addColorStop(0.0, Constants.COLOR_ONE);      // '#8000ff00' - 50% alpha at the apex
  grad.addColorStop(1.0, Constants.COLOR_TWO);      // '#2000ff00' - 12% alpha at the rim
  this.context.strokeStyle = grad;
  this.context.moveTo(this.pointX, this.pointY);
  this.context.lineTo(this.lineOneX, this.lineOneY);
  this.context.moveTo(this.pointX, this.pointY);
  this.context.lineTo(this.lineTwoX, this.lineTwoY);
  this.context.stroke();
  this.context.beginPath();
  this.context.fillStyle = Constants.COLOR_THREE;
  this.context.moveTo(this.pointX, this.pointY);
  // 小于180度的填色，构成扇形  (fill the side under 180 degrees)
  if (this.lineOneAngle - this.lineTwoAngle < -Math.PI ||
    this.lineOneAngle - this.lineTwoAngle > 0 && this.lineOneAngle - this.lineTwoAngle < Math.PI) {
    this.context.arc(this.pointX, this.pointY, this.radius, this.lineTwoAngle, this.lineOneAngle, false);
  } else {
    this.context.arc(this.pointX, this.pointY, this.radius, this.lineOneAngle, this.lineTwoAngle, false);
  }
  this.context.closePath();
  this.context.fill();
}
```

**Two `beginPath()` calls, deliberately.** The radii are stroked with a
gradient and the body is filled flat, so they cannot share a path - a single
path would fill the stroke's outline too. `moveTo` before each `lineTo` is
what stops the two radii being joined by a chord.

`arc(..., startAngle, endAngle, false)` draws counter-clockwise = false, i.e.
in increasing-angle direction, so **which angle is passed first decides which
side of the sector gets filled**. The condition picks the ordering whose sweep
is under 180°; that is the same rule `setArcAngle` applies to the readout
(`absAngle > Math.PI ? 2π − absAngle : absAngle`), and keeping the two in
agreement is why both live one after the other in the file.

`setArcDirection` adds π/2 to the bisector before converting to degrees: the
bisector is measured from the +x axis, and an azimuth is measured from north,
which in canvas coordinates (y downward) is a quarter turn away.

## Permissions & config

**None.** The sample declares no `requestPermissions`. A real bearing tool
would add location and possibly sensor access, but the control itself needs
nothing.

Every literal lives in `common/Constants.ets` - apex at (185, 300), radius
150, edge-proximity `RANGE` 0.15 rad (≈ 8.6°), the three fill/stroke colours,
and the `JUDGE_*` values. That is the file to change when adapting the control
to a different layout.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **The geometry is absolute.** The apex is a hardcoded (185, 300) vp and the
  radius a hardcoded 150 vp, while the canvas is `'100%'` of the page. On any
  screen that is not the phone the sample was tuned on, the sector is off
  centre; on a tablet or a resized 2in1 window it can sit against an edge.
  Derive the apex from the measured canvas size if the control must be
  responsive.
- `clearRect(0, 0, 1000, 1000)` assumes the canvas is never larger than
  1000x1000 vp. On a large window, stale pixels survive outside that box.
- `RANGE` is an angular tolerance, not a distance, so the grab target is
  ~2.6 vp wide at the rim and effectively a point near the apex. A distance-
  based test in vp would give a uniform touch target.
- The readout card is drawn at `offset({ top: -335 })` inside the same `Stack`
  as the canvas - another literal tied to the same phone-sized layout.
- There is no snap, no keyboard input and no way to type an exact bearing; the
  sector can only be set by dragging.

## Pitfalls

- **`HW-15-0018`** (B/medium, confirmed): **the edge grab fails across the ±π
  wraparound.** `Math.abs(lineAngle − angle) < RANGE` compares two normalised
  angles without wrapping the difference, so an edge just below +π and a touch
  just past the seam differ by ~2π and never match; the interior branches can
  misclassify the same touch. Dragging an edge near 180° - the sample's core
  interaction - half-fails. Fix: normalise every difference into [−π, π]
  before comparing, e.g. through the existing `Angle2PI` helper.
- **`HW-15-0019`** (D/low, confirmed): **the document's snippet references
  `Constants.LINE_ONE_CHANGE`,** which does not exist anywhere in the sample -
  the code branches on `Constants.JUDGE_ONE` / `JUDGE_TWO` / `JUDGE_THREE`.
  Copying the documented block does not compile. Fix: use the shipped
  constant names.

## References

- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - `Canvas`, `onReady`, `RenderingContextSettings`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - `PanGesture`, `onActionStart`/`onActionUpdate`, `fingerList` and `offsetX/offsetY`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-guides/03_application-framework/arkts-drawing-customization-on-canvas.md` - drawing with `CanvasRenderingContext2D`, paths, gradients
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-drawing-customization-on-canvas
- `UTIL-03` - the sibling geometry sample, which builds its animation out of transitions instead of a canvas
