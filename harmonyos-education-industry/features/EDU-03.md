---
id: EDU-03
title: Score-range dual slider - two drag handles over a three-segment DataPanel
industry: 04_education
doc: huawei_industry_tree/04_education/docs/03_adjusting_score_interval_screening_schools.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/adjusting_score_interval_screening_schools-0000002294530662
sample: huawei_industry_tree/04_education/downloads/ScoreIntervalScreeningSchools.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: [DataPanel, DataPanelOptions, "DataPanelType.Line", valueColors, PanGesture, PanDirection, onActionStart, onActionUpdate, position, "@Link", "@StorageProp", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", setWindowSystemBarProperties]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-04-0015, HW-04-0016, HW-04-0017, HW-04-0018, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card when you need a **two-handle range control that ArkUI does not
ship**. `Slider` has one thumb; there is no built-in range slider. The trick
here is to stop thinking of it as a slider at all: a three-segment `DataPanel`
draws the track, and two absolutely positioned `Column`s with `PanGesture` are
the handles.

The education case is filtering schools by an exam-score band, with the
student's own score marked on the same track. The same construction fits any
min/max band over a fixed numeric range - price, age, difficulty, duration.

Choose it when you need **the segment between the handles to be styled
differently** from the ends, which is exactly what a three-value `DataPanel`
with three `valueColors` gives you for free.

## Feature checklist

- A 300 vp horizontal track split into three segments: grey, blue, grey.
- The blue middle segment is the selected score band.
- Two handles, each a rounded blue card showing its own score and the unit 分.
- Dragging either handle resizes the middle segment and moves the other segments
  in step; the total always stays 258.
- A handle stops at the edge of the range and at the other handle - it never
  produces a negative segment and never crosses.
- The student's own score is marked on the track with a two-circle pin and the
  label 我.
- Scores read 200 at the far left and 458 at the far right.

## Architecture

One `entry` module, four pages, no model layer - the whole state is three
numbers.

```
entry/src/main/ets
├── entryability/EntryAbility.ets       full screen, system bar colours, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
└── pages
    ├── FilterPage.ets                  @Entry - owns leftValue / rightValue / myValue
    ├── ScreenPage.ets                  the filter chips above the slider
    ├── DualSlider.ets                  THE CARD: track + two handles + gestures
    └── SchoolPage.ets                  the result list
```

The documented tree matches the zip exactly.

**The state split is the design decision worth copying.** `DualSlider` keeps
two kinds of state and they are deliberately different:

- `@State sliderLength: number[]` — the three **segment lengths** in score
  units. This is the geometry, private to the component, and it is what the
  `DataPanel` renders.
- `@Link leftValue / rightValue: string` — the two **displayed scores**, owned
  by `FilterPage`.

The handles are positioned from `sliderLength`, not from the values, so the
component has one source of truth for layout and publishes only the two numbers
the host cares about. `myValue` is a plain member, passed once at construction,
because the student's own score never changes while the page is open.

The three segments always sum to 258 (`[33, 225, 0]` initially): every branch of
both gesture handlers moves length **between** neighbouring segments rather than
setting them independently. That invariant is what keeps the `DataPanel` and the
handle positions consistent, and it is why the clamping is written as three
branches rather than a `Math.min`/`Math.max` pair.

## Implementation steps

1. **Fix the scale first.** Name the two magic numbers:
   `TRACK_WIDTH = 300` (vp) and `SCORE_RANGE = 258` (score units), with
   `SCORE_MIN = 200`. Score-to-vp is `v / SCORE_RANGE * TRACK_WIDTH`; vp-to-score
   is `px / TRACK_WIDTH * SCORE_RANGE`. The sample uses the first expression for
   both directions (`HW-04-0015`).
2. **Draw the track** with a three-value `DataPanel`, `max` equal to the score
   range, `type: DataPanelType.Line`, and three `valueColors` - grey, blue,
   grey. Give it `height(3)` and `borderRadius('50%')` so it reads as a rail.
3. **Stack everything on the track.** The student marker, the two circles and
   both handles are children of one `Stack`, each placed with
   `.position({ left: ... })` computed from `sliderLength`.
4. **Attach a `PanGesture` per handle** with `direction: PanDirection.Horizontal`
   and `distance: 1` so the drag starts immediately.
5. **Reset `lastOffsetX` to 0 in `onActionStart`.** `offsetX` is cumulative from
   the start of the gesture, so each update must work on the delta since the
   previous update, not on the absolute offset.
6. **In `onActionUpdate`, compute both neighbouring segments, then clamp with
   three branches:** under-run (the moving edge hits 0), the normal case, and
   over-run (the neighbouring segment hits 0). Assign into `sliderLength` only
   inside a branch, never before the check.
7. **Derive the displayed score from the geometry**, not from an accumulator:
   left is `sliderLength[0] + SCORE_MIN`, right is
   `SCORE_MIN + SCORE_RANGE - sliderLength[2]`.
8. **Wire the confirm button.** `FilterPage` already holds both values; pass
   them to `SchoolPage` as `@Prop` and filter the list. The sample leaves the
   button inert (`HW-04-0017`).

## Verified snippets

All snippets are from `ScoreIntervalScreeningSchools.zip`. Corrected forms are marked.

**The track — `entry/src/main/ets/pages/DualSlider.ets`** (as shipped)

```typescript
@State sliderLength: number[] = [33, 225, 0];   // left, middle, right - in score units

DataPanel({ values: this.sliderLength, max: 258, type: DataPanelType.Line })
  .width(300)
  .height(3)
  .borderRadius('50%')
  .valueColors([$r('app.color.gray4'), $r('app.color.blue'), $r('app.color.gray4')])
```

**One component draws all three segments.** `DataPanelType.Line` lays the values
out end to end along the width, and `valueColors` colours them independently, so
the "selected band is blue, the rest is grey" appearance costs no layout code at
all. `max: 258` fixes the scale: because the three values always sum to 258, the
rail is always full and the segment boundaries land exactly where the handles
are. Set `max` to 0 or omit it and the panel would normalise to the sum instead,
which looks identical here but breaks the moment a rounding error makes the
values sum to 257.

**A handle and its gesture — same file** (corrected, see `HW-04-0015`)

```typescript
const TRACK_WIDTH = 300;      // vp        FIX: the sample inlines both numbers
const SCORE_RANGE = 258;      // score units
const SCORE_MIN = 200;

// 滑块1 - the left handle
Column() {
  Row() {
    Text(this.leftValue).fancy(14, 400, $r('app.color.white'))
    Text($r('app.string.DualSlider_text2')).fancy(14, 400, $r('app.color.white'))   // 分
  }
  Image($r('app.media.hand')).width(17).height(19)
}
.borderRadius(8)
.backgroundColor($r('app.color.blue'))
.width(this.rectWidth)
.height(this.rectHeight)
.position({ left: this.sliderLength[0] / SCORE_RANGE * TRACK_WIDTH - 27 })   // score -> vp
.gesture(
  PanGesture({ direction: PanDirection.Horizontal, distance: 1 })
    .onActionStart(() => {
      this.lastOffsetX = 0;                    // offsetX is cumulative per gesture
    })
    .onActionUpdate((even) => {
      const delta = (even.offsetX - this.lastOffsetX) / TRACK_WIDTH * SCORE_RANGE;  // FIX: vp -> score
      this.leftNum = this.sliderLength[0] + delta;
      this.midNum = this.sliderLength[1] - delta;
      if (this.leftNum < 0) {                                    // hit the left edge
        this.midNum = this.sliderLength[0] + this.sliderLength[1];
        this.sliderLength[0] = 0;
        this.sliderLength[1] = this.midNum;
        this.leftValue = String(SCORE_MIN);
      } else if (this.leftNum >= 0 && this.midNum >= 0) {        // normal case
        this.sliderLength[0] = this.leftNum;
        this.sliderLength[1] = this.midNum;
        this.leftValue = (this.leftNum + SCORE_MIN).toFixed();
      } else {                                                   // hit the other handle
        this.leftNum = this.sliderLength[0] + this.sliderLength[1];
        this.leftValue = (this.leftNum + SCORE_MIN).toFixed();
        this.sliderLength[0] = this.leftNum;
        this.sliderLength[1] = 0;
      }
      this.lastOffsetX = even.offsetX;
    })
)
```

**Two conversions, two directions, and the sample uses one expression for
both.** `.position()` goes score-to-vp (`v / 258 * 300`); the gesture goes
vp-to-score, which is the reciprocal (`px / 300 * 258`). The shipped code writes
`/ 258 * 300` in both places, so a drag is scaled by (300/258)² = 1.35 and the
handle visibly outruns the finger. Naming the two constants is what makes the
mistake impossible to repeat.

**`lastOffsetX` is not optional.** `PanGestureEvent.offsetX` is measured from
where the gesture began, not from the previous callback, so subtracting the
previous value is what turns it into a per-frame delta. Forget it and the handle
accelerates away on the first drag.

**The three branches are a clamp, written out.** The middle branch is the normal
move; the other two transfer *all* remaining length to the neighbour instead of
letting either segment go negative. Because every branch moves length between
two adjacent segments, the sum stays 258 and the `DataPanel` never renders a
gap.

**The right handle — same file** (as shipped)

```typescript
.position({ left: (this.sliderLength[0] + this.sliderLength[1]) / 258 * 300 - 27 })
// ...
.onActionUpdate((even) => {
  this.midNum = this.sliderLength[1] + (even.offsetX - this.lastOffsetX) / 258 * 300;
  this.rightNum = this.sliderLength[2] - (even.offsetX - this.lastOffsetX) / 258 * 300;
  // ... same three branches, on segments 1 and 2 ...
  this.rightValue = 258 - this.rightNum + 200 < 458 ? (258 - this.rightNum + 200).toFixed() : '458';
  this.lastOffsetX = even.offsetX;
})
```

The right handle is positioned by the **sum** of the two left segments, which is
the only place the invariant is used directly. Its value is derived the same way
from the far end: `458 - rightNum`. The ternary is a guard for the over-run
branch, where `rightNum` is left negative - deriving the value from
`sliderLength[2]` after the branch removes the need for it.

**Host wiring — `entry/src/main/ets/pages/FilterPage.ets`** (as shipped)

```typescript
@Entry
@Component
struct FilterPage {
  @State leftValue: string = '233';     // 200 + 33, matching sliderLength[0]
  @State rightValue: string = '458';    // 200 + 258
  myValue: number = 222;                // the student's own score

  build() {
    Column() {
      ScreenPage()
      DualSlider({ leftValue: $leftValue, rightValue: $rightValue, myValue: this.myValue })
      SchoolPage()
    }
    .width('100%')
    .height('100%')
    .backgroundColor($r('app.color.gray2'))
  }
}
```

The initial `@State` values must agree with `DualSlider`'s initial
`sliderLength` - `'233'` is `200 + 33`. They are declared in two files with no
link between them, which is worth a comment in real code.

## Permissions & config

None. `entry/src/main/module.json5` declares no `requestPermissions` block at
all - the feature is pure UI.

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "pages": "$profile:main_pages"
  }
}
```

`build-profile.json5` enables strict mode, which is worth keeping:

```json5
"buildOption": {
  "strictMode": {
    "caseSensitiveCheck": true,
    "useNormalizedOHMUrl": true
  }
}
```

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The track width (300 vp), the score range (258) and the base score (200) are
  **hard-coded in four places** across `DualSlider.ets`. The control is not
  reusable at another width or range without editing all of them.
- The handle offset `-27` is half of `rectWidth` (54), also inlined.
- `DataPanel` accepts at most nine values; three are used here.
- `EntryAbility` reads the avoid areas once and does not subscribe to
  `avoidAreaChange`, so the insets are stale after a rotation or fold
  (`HW-04-0018`).
- The sample demonstrates the control only. There is no filtering: `SchoolPage`
  renders the same hard-coded school nine times (`HW-04-0017`).

## Pitfalls

- **`HW-04-0015` — the vp-to-score conversion is the score-to-vp one.** Both
  `onActionUpdate` handlers, and the document's own snippet, compute
  `offset / 258 * 300` where they need `offset / 300 * 258`. The handle moves
  35 % further than the finger and the score under it is wrong. This is the one
  defect that makes the sample visibly misbehave - fix it before copying
  anything else here.
- **`HW-04-0016` — the confirm button carries `alignRules` inside a `Column`,
  anchored to `'left_rect'`,** an id no component in the project declares.
  `alignRules` only takes effect inside a `RelativeContainer`; this is dead code.
- **`HW-04-0017` — nothing is filtered.** The 确定 button has no `onClick` and
  `SchoolPage` never reads the range, although 场景介绍 presents school filtering
  as the scenario.
- **`HW-04-0018` — the avoid areas are read once and never refreshed,** unlike
  the `EntryAbility` in `EDU-01`, which subscribes to `avoidAreaChange` (but
  then never releases it). Neither sample shows the complete pattern.
- **Do not forget `onActionStart`.** Without resetting `lastOffsetX` to 0, the
  first delta of every gesture after the first is the whole previous drag.
- **Do not swap `@State sliderLength` for three separate `@State` numbers.**
  The array is what `DataPanel` consumes directly; splitting it means rebuilding
  the array on every frame.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-datapanel.md` - `DataPanelOptions`, `max` semantics, `DataPanelType.Line`, `valueColors`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-datapanel
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - `PanGesture`, `PanDirection`, `distance`, and the cumulative `offsetX`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-location.md` - `position` and the `RelativeContainer`-only rule for `alignRules`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-location
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-size.md` - `width`/`height` and `Length`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-size
- `EDU-01` - the industry shell this page would sit in, and the `EntryAbility` boilerplate it diverges from
