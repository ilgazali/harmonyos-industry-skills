---
id: EDU-19
title: Spherical word cloud - a 3D tag globe that auto-rotates, follows a drag, and coasts
industry: 04_education
doc: huawei_industry_tree/04_education/docs/19_globe_label_animation.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/globe_label_animation-0000002529698917
sample: huawei_industry_tree/04_education/downloads/GlobeLabelAnimation.zip
kits: ["@kit.ArkUI"]
apis: [setInterval, clearInterval, PanGesture, onActionStart, onActionUpdate, onActionEnd, onTouch, TouchType, GestureEvent, position, opacity, ForEach, "@ObservedV2", "@Trace", "@ComponentV2", "@Local", "@Provider", "@Consumer", Navigation, NavPathStack, NavDestination]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-04-0139, HW-04-0140, HW-04-0141, HW-04-0142, HW-04-0143, HW-04-0144, HW-04-0145, HW-04-0146, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card for a **3D tag cloud** - words arranged on a sphere that spins on
its own, follows a drag, coasts to a stop, and lets each label be tapped. In an
education app it is the vocabulary review screen; the same construction is a
topic cloud, a skill wheel, or any "explore a set" screen that wants depth.

There is no 3D engine involved. The whole effect is **three numbers per label and
two 2-D rotation matrices**, recomputed on a timer:

- `x`, `y` place the `Text` with `.position()`;
- `z` is never rendered - it only feeds `.opacity()`, and that alone produces the
  near-bright / far-faint depth illusion;
- a **Fibonacci-style spherical sampling** spreads the labels evenly over the
  sphere without clustering at the poles, which naive `random()` placement does
  not.

Those two ideas - depth as opacity, and the even sampling - are what to take.
The timer and gesture plumbing around them has eight findings, one of them a
blocker, so read this card before copying `MainPage.ets`.

## Feature checklist

- Words laid out evenly over a sphere, not clustered.
- Continuous slow auto-rotation about both axes.
- Words nearer the viewer are opaque; words at the back fade out.
- Dragging spins the sphere in the drag direction, at a speed set by the drag.
- Auto-rotation pauses while a finger is down and resumes on release.
- Releasing a drag coasts: the rotation decays back to the idle speed.
- Tapping a word shows its meaning, part of speech, phonetic and example
  sentences below the sphere.

## Architecture

One `entry` module, one page, one model file. This is a state-management **V2**
sample (`@ComponentV2`, `@Local`, `@ObservedV2`/`@Trace`,
`@Provider`/`@Consumer`).

```
entry/src/main/ets
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets   (documented as entrybackupablility - HW-04-0146)
├── model/wordData.ets      WordInfo (@ObservedV2) + wordInfoArray
└── pages/MainPage.ets      @Entry shell + MemorizeWordPage: geometry, timers, gestures
```

**`WordInfo` is the animated model.** It is `@ObservedV2` with `@Trace` on `x`,
`y` and `z`, so writing a coordinate re-renders only the bound attribute of only
that label - no array reassignment, no component rebuild. That is the correct V2
shape for a per-frame animation, and it is what the sample's `ForEach` key then
throws away (`HW-04-0139`).

`alpha`, the one property that produces the depth illusion, is **not** `@Trace`
(`HW-04-0143`) - and works only because of that same broken key.

**Three timers with overlapping ownership**, which is where most of the defects
are:

| Timer | Period | Created by | Purpose |
| --- | --- | --- | --- |
| `timerAn` | 17 ms | `aboutToAppear`, `onTouch(Up)`, `onActionEnd` | the rotation loop |
| `speedTimer` | 500 ms | `onActionEnd` | the inertia decay |

Two of those three creators fire on the same finger lift (`HW-04-0142`), the
toggle that stops `timerAn` cannot restart it (`HW-04-0141`), and nothing clears
either on teardown (`HW-04-0140`).

## Implementation steps

1. **Model a label as `@ObservedV2` with `@Trace` on every animated
   property** - `x`, `y`, `z` **and** `alpha`.
2. **Place the labels by spherical sampling**, not randomly - see the snippet
   below. `k` walks the cosine of the polar angle in equal steps, which is what
   makes the distribution even in *area* rather than in angle.
3. **Rotate by composing two 2-D rotations** - one in the y/z plane, one in the
   x/z plane - applying both on every tick.
4. **Derive opacity from `z`** on the same pass: `(z + r) / 2r` maps `[-r, r]` to
   `[0, 1]`.
5. **Run one 17 ms interval and give it exactly one owner.** Store the handle,
   reset it to 0 when you clear it, and never create it from two callbacks
   (`HW-04-0141`, `HW-04-0142`).
6. **Clear every interval in `aboutToDisappear`** (`HW-04-0140`).
7. **Key the `ForEach` on the word**, never on the animated item
   (`HW-04-0139`).
8. **Update the two rotation speeds independently** from the drag's two axes
   (`HW-04-0144`).
9. **Decay the inertia on the animation tick**, not on a separate slow timer
   (`HW-04-0145`).

## Verified snippets

All snippets are from `GlobeLabelAnimation.zip`. Corrected forms are marked.

**The model — `entry/src/main/ets/model/wordData.ets`** (corrected, see `HW-04-0143`)

```typescript
@ObservedV2
export class WordInfo {
  @Trace x: number = 0;
  @Trace y: number = 0;
  @Trace z: number = 0;          // never drawn - it only drives alpha
  @Trace alpha: number = 0.5;    // FIX: sample leaves this undecorated
  word: string = '';
  meaning: string = '';
  partOfSpeech: string = '';
  pronunciation: string = '';
  color: string = '#333';
  fontSize: number = 12;
  examples: string[] = [];
  translations: string[] = [];
}
```

**Only the animated properties need `@Trace`.** `word`, `meaning` and the example
arrays never change, so decorating them would cost observation for nothing - the
split in this class is exactly right, except that `alpha` is on the wrong side of
it. It is written on every frame and bound to `.opacity()`, so it belongs with
`x`, `y` and `z`.

**Even placement on the sphere — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
aboutToAppear(): void {
  this.wordInfoArray.forEach((item, i) => {
    const k = -1 + (2 * (i + 1) - 1) / this.wordInfoArray.length;   // walk cos(polar) linearly
    const a = Math.acos(k);                                         // polar angle
    const b = a * Math.sqrt(this.wordInfoArray.length * Math.PI);   // azimuth, golden-angle-like

    item.x = this.radiusValue * Math.sin(a) * Math.cos(b) + this.centerX;
    item.y = this.radiusValue * Math.sin(a) * Math.sin(b) + this.centerY;
    item.z = this.radiusValue * Math.cos(a);                        // no centre - z is depth only
    item.alpha = (item.z + this.radiusValue) / (2 * this.radiusValue);
  });
  this.timerAnAction();
}
```

**Stepping `k` linearly through `[-1, 1]` and taking `acos` is the whole
trick.** Sampling the polar angle uniformly would crowd the poles, because a band
of constant angle near a pole covers far less surface than one at the equator;
sampling its **cosine** uniformly gives equal area per step. The azimuth is
advanced by an irrational multiple so successive points do not line up into
visible spirals.

**`x` and `y` carry the centre offset, `z` does not.** They are screen
coordinates fed to `.position()`; `z` is a pure depth in `[-r, r]`, which is why
`(z + r) / 2r` normalises straight to an opacity.

**Rotation — same file** (as shipped)

```typescript
// about the X axis: y and z change, x is untouched
rotateX() {
  const cos = Math.cos(this.angleX);
  const sin = Math.sin(this.angleX);
  this.wordInfoArray.forEach((item) => {
    const y = (item.y - this.centerY) * cos - item.z * sin + this.centerY;
    const z = item.z * cos + (item.y - this.centerY) * sin;
    item.y = y;
    item.z = z;
    item.alpha = (item.z + this.radiusValue) / (2 * this.radiusValue);
  });
}

// about the Y axis: x and z change, y is untouched
rotateY() {
  const cos = Math.cos(this.angleY);
  const sin = Math.sin(this.angleY);
  this.wordInfoArray.forEach((item) => {
    const x = (item.x - this.centerX) * cos - item.z * sin + this.centerX;
    const z = item.z * cos + (item.x - this.centerX) * sin;
    item.x = x;
    item.z = z;
    item.alpha = (item.z + this.radiusValue) / (2 * this.radiusValue);
  });
}
```

**Both new values are computed before either is assigned.** `y` and `z` are read
on both lines, so writing `item.y` first would feed the updated value into the
`z` calculation and shear the sphere. This is the one piece of the animation the
sample gets exactly right, and it is the easiest thing to get wrong when
transcribing a rotation by hand.

**Subtracting the centre, rotating, adding it back** rotates about the sphere's
centre rather than about the screen origin.

**Rendering — same file** (corrected, see `HW-04-0139`)

```typescript
ForEach(this.wordInfoArray, (item: WordInfo) => {
  Text(item.word)
    .fontSize(item.fontSize)
    .fontColor(item.color)
    .position({ x: item.x, y: item.y })
    .opacity(item.alpha)                     // the entire depth cue
    .onClick(() => { /* fill the detail panel from item */ });
}, (item: WordInfo) => item.word)            // FIX: sample keys on JSON.stringify(item)
```

**This one line is the blocker.** `JSON.stringify(item)` includes `x`, `y`, `z`
and `alpha`, all rewritten every 17 ms, so the key differs on every frame and
ArkUI destroys and rebuilds every label sixty times a second - serialising each
word's meaning, part of speech and eight sentences into a string each time. The
`@Trace` decorations exist so that assigning `item.x` updates one attribute of one
node; keying on the item defeats them completely.

It is also what makes the missing `@Trace` on `alpha` invisible: the node is
rebuilt anyway, so the new opacity is picked up. Fix the key without fixing
`alpha` and the depth fade freezes.

**The timers — same file** (corrected, see `HW-04-0140`, `HW-04-0141`, `HW-04-0142`, `HW-04-0145`)

```typescript
timerAn: number = 0;

startRotation() {
  if (this.timerAn) {
    return;                                  // already running
  }
  this.timerAn = setInterval(() => {
    this.decayTowardsIdle();                 // FIX: sample decays on a separate 500 ms timer
    this.rotateX();
    this.rotateY();
  }, 17);                                    // ~60 fps
}

stopRotation() {
  if (this.timerAn) {
    clearInterval(this.timerAn);
    this.timerAn = 0;                        // FIX: sample never resets the handle,
  }                                          //      so its toggle can only ever stop
}

private decayTowardsIdle() {
  if (Math.abs(this.angleX) > this.viewStartX) { this.angleX *= 0.95; }
  if (Math.abs(this.angleY) > this.viewStartY) { this.angleY *= 0.95; }
}

aboutToDisappear(): void {                   // FIX: absent in the sample
  this.stopRotation();
}
```

**`clearInterval` does not clear the variable.** The sample's
`timerAnAction()` - `if (this.timerAn) { clearInterval(this.timerAn) } else {
setInterval(...) }` - leaves a stale non-zero id behind, so after the first stop
the `if` branch is taken forever and the animation can never restart.

**One owner for the timer.** The sample creates it in three places -
`aboutToAppear`, `onTouch(Up)` and `PanGesture.onActionEnd` - and the last two
both fire on the same finger lift, so each drag leaves an orphaned 60 Hz interval
running with no handle. The sphere speeds up the more the user plays with it.

**Decay belongs on the animation tick.** A 0.7 factor every 500 ms holds the
speed flat for thirty frames and then drops it 30 % in one - visible as a stutter.
A per-frame factor on the existing tick is both smoother and one timer fewer.

**Gestures — same file** (corrected, see `HW-04-0142`, `HW-04-0144`)

```typescript
listener(event: GestureEvent) {
  const x = -event.offsetX;                  // negate: drag right spins the near face right
  const y = -event.offsetY;
  const cap = this.radiusValue * 0.002;      // clamp so a fast flick cannot spin it to a blur
  const changeX = x > 0 ? Math.min(cap, x * 0.001) : Math.max(-cap, x * 0.001);
  const changeY = y > 0 ? Math.min(cap, y * 0.001) : Math.max(-cap, y * 0.001);

  // FIX: the sample guards both assignments with `changeX !== 0 && changeY !== 0`,
  //      so a purely horizontal or vertical drag updates neither angle.
  if (changeX !== 0) { this.angleY = changeX; }   // horizontal drag -> spin about Y
  if (changeY !== 0) { this.angleX = changeY; }   // vertical drag   -> spin about X
  this.rotateX();
  this.rotateY();
}

// FIX: the sample also creates a rotation interval in onTouch(TouchType.Up),
//      duplicating what onActionEnd does. Keep one.
.onTouch((event?: TouchEvent) => {
  if (event?.type === TouchType.Down) {
    this.stopRotation();
  }
})
.gesture(
  PanGesture()
    .onActionUpdate((ev) => { this.listener(ev); })
    .onActionEnd(() => { this.startRotation(); })
)
```

**`angleY` comes from the horizontal drag and `angleX` from the vertical one** -
the cross-assignment is correct, not a typo: dragging sideways spins the globe
about its vertical axis.

**The clamp is what keeps it usable.** `event.offsetX` is cumulative over the
gesture, so a long drag would otherwise set an angular step large enough to make
the sphere unreadable; capping at `radius * 0.002` bounds the per-frame rotation
regardless of how far the finger travels.

## Permissions & config

None. `entry/src/main/module.json5` declares no `requestPermissions` block, and
there is no route map - the shell pushes its single destination by name from
`aboutToAppear`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The sphere's geometry is hard-coded**: `radiusValue = 100`,
  `centerX = 144`, `centerY = 120`, inside a container fixed at 250 vp tall. The
  globe does not scale to the screen, and on a wider device it sits off centre.
- The rotation is a fixed 17 ms `setInterval`, not a frame callback, so it does
  not track the display's actual refresh rate.
- **Coordinates are integrated in place**: each frame rotates the previous
  frame's values rather than re-deriving them from a base position and an
  accumulated angle. Floating-point error therefore accumulates over long
  sessions.
- The detail panel is eleven separate `@Local` string fields
  (`selectedExample`, `selectedExample2`, …) rather than the selected `WordInfo`,
  so a word with fewer than four examples writes `undefined` into the tail
  fields.
- The counts under the sphere - 已背 424, 错误 2 - are literal `Text`.

## Pitfalls

- **`HW-04-0139` — the `ForEach` key is `JSON.stringify(item)`** on an item whose
  coordinates change every frame, so every label is destroyed and rebuilt at
  60 Hz. This is the defect to fix first; everything the `@Trace` decorations buy
  is lost to it.
- **`HW-04-0140` — neither interval is cleared on teardown.** There is no
  `aboutToDisappear`, so the 17 ms loop keeps running over the word array after
  the page is gone.
- **`HW-04-0141` — the toggle clears the interval but not its handle,** so
  auto-rotation stops permanently on the first press.
- **`HW-04-0142` — `onTouch(Up)` and `onActionEnd` both start a rotation
  interval** on the same finger lift; one leaks, and which one is not
  deterministic.
- **`HW-04-0143` — `alpha` is the only animated property without `@Trace`.** It
  works today only because of `HW-04-0139`.
- **`HW-04-0144` — `changeX !== 0 && changeY !== 0`** means a straight horizontal
  or vertical drag - the two most natural ones - updates neither angle.
- **`HW-04-0145` — the inertia decays every 500 ms** against a 17 ms animation,
  and assigns `null` to a field typed `number`.
- **`HW-04-0146` — the project tree spells the directory
  `entrybackupablility`.**
- **Compute both rotated values before assigning either.** Writing `item.y`
  before computing `z` shears the sphere.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - the "unique and persistent key" rule and what a changing key costs
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-guides/03_application-framework/arkts-new-observedv2-and-trace.md` - `@ObservedV2` and `@Trace` on individual properties
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-observedv2-and-trace
- `documentation/harmonyos-guides/03_application-framework/arkts-new-local.md` - `@Local` in a `@ComponentV2`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-local
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - `PanGesture`, the cumulative `offsetX`/`offsetY`, and the action callbacks
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/ts-universal-events-touch.md` - `onTouch` and `TouchType`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-events-touch
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-opacity.md` - `opacity`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-opacity
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-location.md` - `position`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-location
- `documentation/harmonyos-references/05_common-capabilities/js-apis-timer.md` - `setInterval` / `clearInterval`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-timer
- `EDU-06` - the other `@ObservedV2`/`@Trace` animation in this industry, with the same array-observation gap
- `EDU-10` - the same timer-driven-animation-with-a-broken-key pattern, at 16 ms
