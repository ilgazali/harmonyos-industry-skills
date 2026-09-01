---
id: SPORT-03
title: Hold-to-finish a workout - a Path-drawn fill animation over a long press
industry: 03_sports_health
doc: huawei_industry_tree/03_sports_health/docs/03_indoor_run.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/indoor_run-0000002266807001
sample: huawei_industry_tree/03_sports_health/downloads/IndoorRun.zip
kits: ["@kit.ArkUI", "@kit.ArkTS"]
apis: [Path, commands, hitTestBehavior, HitTestMode, LongPressGesture, onAction, onActionEnd, GestureEvent, "UIContext.animateTo", "animation attribute", TextTimer, TextTimerController, "util.format", "UIContext.vp2px", "UIContext.px2vp", setInterval, clearInterval, "@State", "@StorageProp", position, zIndex]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-03-0014, HW-03-0015, HW-03-0016, HW-03-0017, HW-03-0055]
status: verified-with-fixes
---

## When to use

Load this card when **a destructive action must be held rather than tapped** -
ending a workout, deleting a recording, unlocking a payment, cancelling an
order in progress. The reason the document gives is the right one:
为了避免运动过程中因误触等操作意外终止 (to avoid a workout being ended by an
accidental touch).

The technique is worth taking whole: a `Path` component whose `commands`
string is recomputed on a timer, so the confirm button **fills from left to
right** as the finger stays down and empties the moment it lifts. That is
built from three parts:

- **`LongPressGesture`** to start and stop a fast interval.
- **A geometry function** that turns a 0-100 progress into an SVG path
  covering that fraction of a capsule.
- **`hitTestBehavior(HitTestMode.None)`** so the drawn overlay never steals
  the press it is visualising.

There is no progress component involved and no image sequence - the fill is
one string, regenerated every 4 ms.

## Feature checklist

- A running screen with a timer and workout metrics.
- A start button that begins the timer.
- Once running, the button splits into 继续 (continue) and a finish button
  that slide apart.
- Holding the finish button fills it progressively from the left.
- Releasing before the fill completes resets it instantly to empty.
- Completing the fill ends the workout, resets the timer and restores the
  start button.

## Architecture

One `entry` module, one page, two components, one constants file.

```
entry/src/main/ets
├── components
│   ├── ProgressBar.ets    the three buttons, the gesture, the geometry, the Path
│   └── SportsData.ets     the metrics and the TextTimer
├── constants/Constants.ets  colours, geometry, durations - and the vp2px conversion
├── entryability/EntryAbility.ets
├── entrybackupability/
└── pages/IndoorRun.ets    @Entry - composes the two components, owns the TextTimerController
```

The documented tree matches the zip.

**The `TextTimerController` is owned by the page and passed to both children**,
so `SportsData` renders the elapsed time and `ProgressBar` drives it - `start()`
on continue, `pause()` on pause, `reset()` on finish. That is the right split:
one controller, one owner, two consumers.

**The geometry is the interesting part.** The button is a capsule of width
`4R` and height `2R`, and the fill boundary is a vertical chord whose
half-length comes from a circle equation, so the leading edge is round while
it crosses either end cap and flat in the middle:

```
progress    0%              25%             50%              100%
           ( )             (|)             (  |)            (    )
            ^ empty         ^ chord in       ^ flat edge      ^ full
                              the left cap     in the middle
```

Three cases, decided by where the leading edge `abscissa` falls:

| `abscissa` | region | path built |
| --- | --- | --- |
| `< R` | inside the left cap | a single arc closed back on itself |
| `R … 3R` | the straight middle | left cap + a rectangle to the edge |
| `3R … 4R` | inside the right cap | the above, plus a second closed arc |

## Implementation steps

1. **Express the button in radii**, not pixels: width `4R`, height `2R`, and
   the two boundaries at `R` and `3R`.
2. **Convert vp to px once**, from a `UIContext` a component supplies - not at
   module load (`HW-03-0014`).
3. **Map progress to an abscissa** across the full width, then get the chord
   half-length from the circle equation for whichever cap it is in.
4. **Build the path with a floating-point format specifier** (`HW-03-0015`).
5. **Recompute `commands` on every tick** and let ArkUI redraw - the `Path` is
   bound to `@State`, so no animation API is involved in the fill itself.
6. **Start the interval in `onAction`, clear it in `onActionEnd`** - and in
   `aboutToDisappear` (`HW-03-0016`).
7. **Take the overlay out of hit testing** with `HitTestMode.None`.

## Verified snippets

All snippets are from `IndoorRun.zip`. Corrected forms are marked.

**The chord geometry — `entry/src/main/ets/components/ProgressBar.ets`** (as shipped)

```typescript
getAbscissa(progressPercent: number): number {
  return progressPercent * Constants.WHOLE_BUTTON_WITH;      // 0 .. 4R
}

getDistanceSquare(abscissa: number): number {
  if (abscissa < Constants.RADIUS) {                          // inside the left cap
    return Constants.RADIUS * Constants.RADIUS -
      (Constants.RADIUS - abscissa) * (Constants.RADIUS - abscissa);
  } else if (abscissa > Constants.ARRIVE_RIGHT_SEMICIRCLE) {  // inside the right cap
    return Constants.RADIUS * Constants.RADIUS -
      (abscissa - Constants.ARRIVE_RIGHT_SEMICIRCLE) * (abscissa - Constants.ARRIVE_RIGHT_SEMICIRCLE);
  } else {
    return Constants.ZERO;                                    // the straight middle
  }
}

getPathCommands(progress: number): string {
  const abscissa: number = this.getAbscissa(progress / Constants.PERCENT_RATE);
  const distanceSquare: number = this.getDistanceSquare(abscissa);
  if (distanceSquare >= Constants.ZERO) {
    const distance: number = distanceSquare === Constants.ZERO ? Constants.ZERO : Math.sqrt(distanceSquare);
    const firstOrdinate: number = Constants.RADIUS - distance;    // top of the chord
    const secondOrdinate: number = Constants.RADIUS + distance;   // bottom of the chord
    return this.formatPathCommands(firstOrdinate, secondOrdinate, abscissa, Constants.RADIUS);
  }
  return '';
}
```

**`r² - (r - x)²` is the whole trick.** It is the circle equation solved for
half the chord at horizontal offset `x` into a cap, so the leading edge of the
fill follows the button's rounded end instead of cutting across it squarely.
Returning zero in the middle region collapses the chord to the full height,
which is exactly what a straight edge is.

**Building the path — same file** (corrected, see `HW-03-0015`)

```typescript
formatPathCommands(firstOrdinate: number, secondOrdinate: number, abscissa: number, radius: number): string {
  if (abscissa < radius) {
    // still inside the left cap: one arc from the bottom of the chord to the top, closed by a line
    const startPosition = util.format('M%f %f', abscissa, secondOrdinate);      // FIX: sample uses %d
    const ellipticalArc = util.format('A%f %f 0 0 1 %f %f', radius, radius, abscissa, firstOrdinate);
    const lineTo = util.format('L%f %f', abscissa, secondOrdinate);
    return util.format('%s %s %s', startPosition, ellipticalArc, lineTo);
  } else if (abscissa < Constants.ARRIVE_RIGHT_SEMICIRCLE) {
    // the whole left cap plus a rectangle out to the leading edge
    const startPosition = util.format('M%f %f', radius, Constants.DIAMETER_PX);
    const ellipticalArc = util.format('A%f %f 0 %d 1 %f 0',
      radius, radius, abscissa > Constants.RADIUS ? 1 : 0, radius);            // the large-arc flag stays %d
    const lineTo = util.format('L%f 0 L %f %f L %f %f',
      abscissa, abscissa, Constants.DIAMETER_PX, radius, Constants.DIAMETER_PX);
    return util.format('%s %s %s Z', startPosition, ellipticalArc, lineTo);
  }
  // ... the right-cap case appends a second closed arc
}
```

**Only the large-arc flag should stay `%d`** - it is genuinely 0 or 1. Every
coordinate is fractional, and the util reference defines `%d` as "Converts a
parameter into a decimal integer", so the shipped version quantises the sweep
onto whole pixels.

**Driving it from a long press — same file** (corrected, see `HW-03-0016`)

```typescript
private interval: number | undefined = undefined;      // FIX: sample seeds it to 0, a legal timer id

Button()
  .type(ButtonType.Capsule)
  .gesture(
    LongPressGesture({ repeat: false })                // one action per press, not repeated
      .onAction((event: GestureEvent) => {
        if (event) {
          this.stopInterval();                          // FIX: make it idempotent
          this.interval = setInterval(() => {
            this.process = true;
            this.changeProgressNum(true);               // += 0.5, recolour, rebuild the path
            if (this.progress >= Constants.FULL_PROGRESS) {
              this.textTimerController?.reset();
              this.showButton = Constants.SHOW_START;
              this.isExpand = !this.isExpand;
              this.stopInterval();
            }
          }, Constants.PROGRESS_CHANGE_INTERVAL);       // 4 ms -> 100% in about 800 ms
        }
      })
      .onActionEnd(() => {
        this.stopInterval();
        this.progress = Constants.ZERO;                 // release resets instantly, no animation
        this.process = false;
        this.pathCommands = '';
      })
  )

private stopInterval(): void {
  if (this.interval !== undefined) {
    clearInterval(this.interval);
    this.interval = undefined;
  }
}

aboutToDisappear(): void {                              // FIX: absent in the sample
  this.stopInterval();
}
```

**4 ms with a 0.5 step is 800 ms to fill**, which is the deliberate part: long
enough that an accidental brush cannot complete it, short enough not to feel
like a punishment. Resetting `progress` to zero on release rather than
animating it back is also deliberate - an accidental touch should leave no
trace.

**The overlay must not take the press — same file** (as shipped)

```typescript
Path()
  .width(Constants.TWO * Constants.DIAMETER)
  .height(Constants.DIAMETER)
  .commands(this.pathCommands)
  .fill(this.colorBackgroundFill)
  .antiAlias(true)
  .stroke(this.colorBackgroundFill)
  .strokeWidth(Constants.ONE)
  .zIndex(Constants.THREE)
  .hitTestBehavior(HitTestMode.None)      // the fill is painted over the button being held
```

**`HitTestMode.None` is what makes the whole thing work.** The `Path` sits at
the highest `zIndex`, directly over the button whose long press is driving it,
so without this the first frame of the fill would swallow the gesture that
produced it. The same attribute is used to disable the two capsule buttons
while their colour is transparent - `hitTestBehavior(this.finishColor === Constants.COLOR_TRANSPARENT ? HitTestMode.None : HitTestMode.Default)` -
which is a cleaner way to hide a control than mounting and unmounting it.

**Two animation mechanisms, deliberately different**

```typescript
// the buttons sliding apart: a property animation on the position
.position({ left: this.isExpand ? Constants.POSITION_LEFT_LEFT : Constants.POSITION_LEFT, top: ... })
.animation({ duration: Constants.DURATION })

// the pause transition: an explicit animateTo around the state change
this.getUIContext().animateTo({ duration: Constants.DURATION }, () => {
  this.isExpand = !this.isExpand;
});
```

The declarative `.animation()` covers changes the component makes to its own
attributes; `animateTo` is used where one handler changes state that several
components react to at once.

## Permissions & config

**None.** The sample declares no `requestPermissions` - the screen is a
gesture and a drawing, with no sensor and no location.

No routing configuration; a single `@Entry` page.

`Constants.ets` holds the colours, the geometry and the two timings
(`DURATION` 500 ms for the slide, `PROGRESS_CHANGE_INTERVAL` 4 ms for the
fill).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` and
  `targetSdkVersion` are both `6.0.0(20)`.
- **The screen is pinned to one device size.** Every control is placed with
  `position()` at an absolute offset (`HW-03-0017`), so the module's declared
  tablet and 2in1 support is nominal.
- The geometry is in **px**, converted once from a vp diameter, so the path
  coordinates and the component's vp dimensions are in different units by
  design - `RADIUS` is px, `DIAMETER` is vp.
- The workout data in `SportsData` is static apart from the `TextTimer`; there
  is no pedometer here. For that see `SPORT-01`.
- `LongPressGesture({ repeat: false })` fires `onAction` once per press, so
  the interval is what produces the repetition, not the gesture.

## Pitfalls

- **`HW-03-0014` — `Constants` reads `uiContext` from `AppStorage` at module
  scope**, while the ability writes that key inside the `loadContent`
  callback, which runs after the page that imports `Constants` has been
  loaded. Every geometry constant derives from that read. *(probable — the
  exact module-evaluation order is not documented in this corpus, but the
  ordering in the two files is the wrong way round.)*
- **`HW-03-0015` — the path is formatted with `%d`,** which the util reference
  defines as converting to a decimal integer, so a square-root curve sampled
  in half-percent steps is quantised onto the pixel grid.
- **`HW-03-0016` — the 4 ms interval is cleared only from the gesture's own
  paths** and the component has no `aboutToDisappear`, so a press interrupted
  by leaving the page leaves a 250 Hz callback running against a destroyed
  component. The handle is also seeded to `0`, a legal timer id.
- **`HW-03-0017` — every control is at an absolute `position()`,** with seven
  offsets in `Constants` and two more inline, tuned for one screen.

## References

- `documentation/harmonyos-references/02_application-framework/ts-drawing-components-path.md` - `Path` and its `commands` syntax
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-drawing-components-path
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-longpressgesture.md` - `LongPressGesture`, `repeat`, `onAction`, `onActionEnd`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-longpressgesture
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-hit-test-behavior.md` - `HitTestMode.None`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-hit-test-behavior
- `documentation/harmonyos-references/02_application-framework/ts-explicit-animation.md` - `animateTo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-explicit-animation
- `documentation/harmonyos-references/02_application-framework/ts-animatorproperty.md` - the `animation` attribute
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-animatorproperty
- `documentation/harmonyos-references/02_application-framework/js-apis-util.md` - `util.format` and its specifiers
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-util
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-texttimer.md` - `TextTimer` and `TextTimerController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-texttimer
- `SPORT-12` - the countdown that runs before this screen; `SPORT-01` - the app this screen belongs to
