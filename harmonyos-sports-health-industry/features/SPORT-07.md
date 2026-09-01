---
id: SPORT-07
title: The three activity rings - concentric ProgressType.Ring in a Stack
industry: 03_sports_health
doc: huawei_industry_tree/03_sports_health/docs/07_sport_three_ring.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/sport_three_ring-0000002345359673
sample: huawei_industry_tree/03_sports_health/downloads/SportThreeRing.zip
kits: ["@kit.ArkUI"]
apis: [Progress, ProgressType, ProgressStyleOptions, strokeWidth, Stack, Alignment, offset, "@State", "@StorageProp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-03-0029, HW-03-0030, HW-03-0031, HW-03-0055]
status: verified-with-fixes
---

## When to use

Load this card for the **activity-rings readout** - three or more concentric
progress arcs showing separate daily goals at a glance. Calories, exercise
minutes and stand hours here; the same shape covers water intake, sleep
stages, reading targets, any small set of independent daily goals.

The point of the card is how little it takes: **three `Progress` components of
type `Ring`, centred in a `Stack`, with different widths.** There is no
`Canvas`, no path maths and no animation code - each ring is one component
whose `value` and `total` do the work. Compare `SPORT-03`, which needs a
hand-built `Path` because its arc has to follow a capsule, and `SPORT-10`,
which draws its own connectors.

## Feature checklist

- Three concentric rings, each with its own goal and colour.
- A step count above them with its target.
- An icon on each ring marking its track.
- Below the rings, one detail row per metric: coloured badge, label, current
  value and target.
- Rings share a centre and are sized in equal radius steps.

## Architecture

One `entry` module, one page. This is the smallest sample in the industry.

```
entry/src/main/ets
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
└── pages/ProgressExample.ets     @Entry - the whole screen, 215 lines
```

The documented tree matches the zip.

**Four metrics, eight state fields, arranged as value/target pairs:**

```
stepNumber       / targetStepNumber      1820 / 10000   (shown as text only)
exerciseCalories / targetCalories          72 / 270     outer ring,  200 wide, red
exerciseTime     / targetTime              16 / 60      middle ring, 150 wide, amber
totalTime        / targetTotalTime          5 / 12      inner ring,  100 wide, blue
```

Pairing each value with its own target is what lets `Progress` do the
arithmetic - no percentage is ever computed in the page.

**The concentric geometry is three widths in equal steps:**

```
        ┌─────────────────┐
        │   ┌─────────┐   │   200 wide → radius 100, calories   (outer)
        │   │  ┌───┐  │   │   150 wide → radius  75, minutes    (middle)
        │   │  │   │  │   │   100 wide → radius  50, hours      (inner)
        │   │  └───┘  │   │
        │   └─────────┘   │   strokeWidth 18 on all three
        └─────────────────┘   → 7 vp of clear space between rings
```

`Stack({ alignContent: Alignment.Center })` centres them on one point, so the
only thing separating the rings is their differing `width` - the radius step
of 25 against a stroke of 18 is what leaves a visible gap rather than three
touching bands.

## Implementation steps

1. **Hold each metric as a value and a target**, and let `Progress` compute
   the fraction.
2. **Set `type: ProgressType.Ring`** on each and give it a `width` - for a
   ring, `width` is the diameter.
3. **Centre them in a `Stack`** with `alignContent: Alignment.Center`.
4. **Choose the radius step larger than the stroke width** so the rings do not
   touch: here 25 vp of step against 18 vp of stroke.
5. **Set `color` for the filled arc and `backgroundColor` for the track** -
   through colour resources, with dark variants (`HW-03-0029`).
6. **Derive the icon offsets from the radii** rather than hardcoding them
   (`HW-03-0031`).

## Verified snippets

All snippets are from `SportThreeRing.zip`. Corrected forms are marked.

**The three rings — `entry/src/main/ets/pages/ProgressExample.ets`** (corrected, see `HW-03-0029`, `HW-03-0031`)

```typescript
Stack({ alignContent: Alignment.Center }) {
  // inner: stand hours
  Progress({ value: this.totalTime, total: this.targetTotalTime, type: ProgressType.Ring })
    .width(100)                                  // diameter, not stroke
    .color($r('app.color.ring_hours'))           // FIX: sample uses '#2e80ff'
    .style({ strokeWidth: 18, shadow: true })
    .backgroundColor($r('app.color.ring_hours_track'))   // FIX: sample uses '#d5e5ff'
  Image($r('app.media.hours'))
    .height(12)
    .offset({ x: 3, y: -30 })                    // FIX: derive from the radius - see HW-03-0031

  // middle: exercise minutes
  Progress({ value: this.exerciseTime, total: this.targetTime, type: ProgressType.Ring })
    .width(150)
    .color($r('app.color.ring_minutes'))
    .style({ strokeWidth: 18, shadow: true })
    .backgroundColor($r('app.color.ring_minutes_track'))
  Image($r('app.media.times')).height(12).offset({ x: 3, y: -55 })

  // outer: calories
  Progress({ value: this.exerciseCalories, total: this.targetCalories, type: ProgressType.Ring })
    .width(200)
    .color($r('app.color.ring_calories'))
    .style({ strokeWidth: 18, shadow: true })
    .backgroundColor($r('app.color.ring_calories_track'))
  Image($r('app.media.calories')).height(12).offset({ x: 3, y: -80 })
}
.padding({ top: 20 })
```

**Three things make this work and are easy to get wrong:**

- **`width` is the ring's diameter, `strokeWidth` its thickness.** They are
  set in different places - `width` as an attribute, `strokeWidth` inside
  `style` - so it is natural to assume `width` means the band. It does not.
- **The radius step must exceed the stroke.** Widths of 100, 150 and 200 give
  radii 50, 75 and 100, a step of 25 against a stroke of 18, leaving 7 vp
  clear. Set the widths 30 apart with the same stroke and the rings overlap.
- **`backgroundColor` is the unfilled track, not the component's background.**
  On `Progress` it paints the remainder of the ring, which is why each ring
  pairs a saturated `color` with a pale tint of the same hue.

`shadow: true` lifts each arc off its track, which is what keeps three
concentric rings readable when several are near completion.

**Ordering inside the Stack is the z-order.** Each icon is declared
immediately after its ring, so it paints above that ring and below the next
one out - which is why the icons sit on their own tracks rather than all on
top.

**The detail rows — same file** (as shipped)

```typescript
Row() {
  Row() {
    Image($r('app.media.calories')).height(20)
  }
  .height(34).width(34)
  .backgroundColor('#f23322')                  // the same hue as its ring
  .justifyContent(FlexAlign.Center)
  .borderRadius(8)

  Column() {
    Text($r('app.string.activity_heat'))
      .fontSize(18)
    Row() {
      Text(this.exerciseCalories.toString()).fontSize(34)
      Text(' /' + this.targetCalories.toString() + '千卡')   // FIX: unit literal - HW-03-0030
        .fontColor(Color.Gray)
    }
    .alignItems(VerticalAlign.Bottom)
  }
  .alignItems(HorizontalAlign.Start)
}
```

**Repeating the ring's colour on the row's badge** is what ties the two halves
of the screen together - the user reads the ring, then finds the matching
swatch below it. That is also the reason the colours belong in resources: they
are used twice each, and a change has to reach both.

`alignItems(VerticalAlign.Bottom)` on the value row is a small but deliberate
touch: the large current value and the small target sit on a shared baseline
instead of being centred against each other.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

No routing configuration; a single `@Entry` page.

Resource directories: `base` and `dark`, each holding one colour -
`start_window_background`. Everything the screen actually paints is inline
(`HW-03-0029`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The data is static.** All eight values are `@State` literals - 1820 steps,
  72 of 270 kilocalories, and so on. The document describes the screen as
  实时显示运动数据 (displaying exercise data in real time), and nothing here
  reads a sensor or a store; for the pedometer that would feed it see
  `SPORT-01`.
- **`Progress` clamps but does not overflow.** A value above `total` fills the
  ring and stops, so an activity ring cannot show the "exceeded the goal"
  second lap that some fitness apps draw. That needs a second ring or a
  `Canvas`.
- The ring sizes are fixed in vp, so the group does not scale with the screen;
  on a small display the 200 vp outer ring plus its padding dominates the
  page.
- Only colour distinguishes the three rings. The icons help, but a
  colour-blind user has no other cue in the ring group itself - the labelled
  rows below carry the meaning.

## Pitfalls

- **`HW-03-0029` — all twelve ring and badge colours are inline hex,** so the
  `dark` resource directory the module ships is bypassed and three near-white
  ring tracks stay near-white in dark mode.
- **`HW-03-0030` — the four units (`步`, `千卡`, `分钟`, `小时`) are
  concatenated into the readouts as literals** while the labels beside them
  are `$r` resources, fixing both the wording and the number-unit order.
- **`HW-03-0031` — each icon's offset is a literal tuned to its ring's
  radius,** with nothing linking the two, so changing a ring size leaves its
  icon behind.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-progress.md` - `Progress`, `ProgressType.Ring`, `ProgressStyleOptions`, `strokeWidth`, `shadow`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-progress
- `documentation/harmonyos-references/02_application-framework/ts-container-stack.md` - `Stack` and `alignContent`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-stack
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-location.md` - `offset`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-location
- `documentation/harmonyos-guides/01_getting-started/resource-categories-and-access.md` - colour resources and the `dark` qualifier
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/resource-categories-and-access
- `SPORT-01` - the pedometer that would supply these numbers
- `SPORT-05` - the same industry's charts drawn with a library; `SPORT-03` and `SPORT-10` - arcs and connectors drawn by hand
