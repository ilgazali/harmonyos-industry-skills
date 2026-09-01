---
id: SPORT-12
title: The 3-2-1 countdown - chained animateTo with onFinish
industry: 03_sports_health
doc: huawei_industry_tree/03_sports_health/docs/12_countdown_to_run.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/countdown_to_run-0000002394156713
sample: huawei_industry_tree/03_sports_health/downloads/CountdownToRun.zip
kits: ["@kit.ArkUI"]
apis: ["UIContext.animateTo", AnimateParam, onFinish, Curve, PlayMode, onAppear, "@Link", "@State", "@StorageProp", aspectRatio, ResourceColor, Length]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-03-0046, HW-03-0047, HW-03-0048, HW-03-0055]
status: verified-with-fixes
---

## When to use

Load this card for a **countdown before something starts** - a run, a timed
exercise, a photo timer, a game round - where each number should animate in
its own right rather than simply being replaced.

The technique is **chaining `animateTo` through `onFinish`**: each animation's
completion callback starts the next one, so the sequence is driven by the
animation system rather than by a timer. That is worth knowing because it is
the general answer to "run these animations one after another" in ArkUI, and
it has a property a `setInterval` does not - each step begins exactly when the
previous one has finished rendering, so the steps cannot drift or overlap.

It also has the weakness this sample demonstrates: an `onFinish` chain has no
handle, so it cannot be stopped (`HW-03-0047`).

## Feature checklist

- Tapping start covers the screen with a full-bleed countdown.
- Each number appears large, shrinks over a second, and is replaced by the
  next.
- After the last number, the overlay collapses into the run screen's stop
  button, which changes shape and colour.
- When the collapse finishes, the run is marked started.

## Architecture

One `entry` module, one page, three components.

```
entry/src/main/ets
├── common/Constants.ets            durations, the countdown start, the button ratio
├── components
│   ├── CountdownPage.ets           the overlay: the chained animation
│   ├── ProcessButton.ets           the start/stop control
│   └── SportsData.ets              the metrics behind the overlay
├── entryability/EntryAbility.ets
├── entrybackupability/
├── model/RunDataModel.ets
├── pages/MainPage.ets              @Entry - owns every piece of shared state
└── utils/Logger.ets
```

The documented tree matches the zip.

**All the state lives in the page and is passed down by `@Link`** - seven of
them into `CountdownPage` alone: the number, the font size, the two flags, the
button state and the two durations. The overlay owns only its own `isFinish`,
`buttonHeight` and `buttonColor`.

That is a defensible choice for an overlay that has to hand control back to
its host, and it is also why the uncancellable chain matters: the last
callback writes four of the parent's fields.

**The animation shape:**

```
onAppear
   │
animateTo(1000ms) ──closure: fontSize=240, number--     shows 3
   └─ onFinish: fontSize=150
        animateTo(1000ms) ──closure: fontSize=240, number--     shows 2
           └─ onFinish: fontSize=150
                animateTo(1000ms) ──closure: fontSize=240, number--     shows 1
                   └─ onFinish: fontSize=150, isFinish=true
                        animateTo(500ms) ──closure: height=62, colour=red
                           └─ onFinish: reset the number, clickStart=false,
                                        countdownFinished=true, buttonState++
```

**Each step sets the font size twice**: to 240 inside the closure, which the
animation interpolates *from* the 150 set by the previous `onFinish`. So the
number grows into place over the second, and the jump back to 150 happens
outside any animation, instantly, at the moment the next number appears.

## Implementation steps

1. **Set the value and the target size inside the closure** - `animateTo`
   animates whatever the closure changes.
2. **Reset the starting size in `onFinish`**, outside any animation, so the
   next step has somewhere to grow from.
3. **Start the next step from `onFinish`** rather than from a timer.
4. **Drive the step count from the constant**, not from the nesting depth
   (`HW-03-0046`).
5. **Guard the callbacks against a destroyed component** and against a second
   chain starting while one runs (`HW-03-0047`).
6. **Collapse into the next screen's control** in a final, shorter animation,
   and hand control back from its `onFinish`.

## Verified snippets

All snippets are from `CountdownToRun.zip`. Corrected forms are marked.

**One step of the chain — `entry/src/main/ets/components/CountdownPage.ets`** (as shipped)

```typescript
this.getUIContext().animateTo({
  duration: this.animationDuration,      // 1000 ms
  playMode: PlayMode.Normal,
  curve: Curve.EaseInOut,
  onFinish: () => {
    this.countdownFontSize = $r('app.float.font_size_150');   // instant: the next number's start size
    // ... the next animateTo goes here
  }
}, () => {
  this.countdownFontSize = $r('app.float.font_size_240');     // animated: grows to this over 1 s
  this.countdownNumber--;
});
```

**The two font sizes are doing different jobs.** The one in the closure is the
animation's target and is interpolated; the one in `onFinish` is a plain
assignment that resets the baseline for the next step. Swap them and the
number shrinks instead of growing; omit the `onFinish` reset and every number
after the first starts already at its final size, so nothing appears to
animate.

Decrementing inside the closure rather than in `onFinish` is what makes the
new number appear at the *start* of its own animation.

**The whole chain, driven by the constant — same file** (corrected, see `HW-03-0046`, `HW-03-0047`)

```typescript
private cancelled: boolean = false;
private running: boolean = false;

changeText(): void {
  if (this.running) {
    return;                              // FIX: the sample can start a second chain over the first
  }
  this.running = true;
  this.countdownNumber = Constants.COUNTDOWN_NUMBER;
  this.animationDuration = Constants.DURATION_1000;
  this.shrinkDuration = Constants.DURATION_500;
  this.isFinish = false;
  this.step();                           // FIX: the sample nests four animateTo calls by hand
}

private step(): void {
  this.getUIContext().animateTo({
    duration: this.animationDuration,
    playMode: PlayMode.Normal,
    curve: Curve.EaseInOut,
    onFinish: () => {
      if (this.cancelled) {
        return;                          // FIX: absent in the sample
      }
      this.countdownFontSize = $r('app.float.font_size_150');
      if (this.countdownNumber > 1) {
        this.step();                     // one recursion per remaining number
      } else {
        this.isFinish = true;
        this.collapse();
      }
    }
  }, () => {
    this.countdownFontSize = $r('app.float.font_size_240');
    this.countdownNumber--;
  });
}

private collapse(): void {
  this.getUIContext().animateTo({
    duration: this.shrinkDuration,
    playMode: PlayMode.Normal,
    curve: Curve.EaseInOut,
    onFinish: () => {
      this.running = false;
      if (this.cancelled) {
        return;
      }
      this.countdownNumber = Constants.START_NUMBER;
      this.clickStart = false;
      this.countdownFinished = true;     // hand control back to the page
      this.buttonState++;
    }
  }, () => {
    this.buttonHeight = $r('app.float.height_62');
    this.buttonColor = $r('app.color.button_color_red');
  });
}

aboutToDisappear(): void {               // FIX: absent in the sample
  this.cancelled = true;
}
```

**The collapse is a different animation from the counting steps** - shorter
(500 ms against 1000) and changing geometry rather than text - which is what
makes the transition into the run screen read as one motion rather than a
fourth count.

**Swapping the view, not the component — same file** (as shipped)

```typescript
build() {
  if (!this.isFinish) {
    Column() {
      Text(this.countdownNumber.toString())
        .fontSize(this.countdownFontSize)
    }
    .width(Constants.FULL_PERCENT)
    .height(Constants.FULL_PERCENT)
    .backgroundColor($r('app.color.countdown_background_color'))
    .onAppear(() => {
      this.changeText();
    })
  } else {
    Column() {
      Button()
        .backgroundColor(this.buttonColor)
        .height(this.buttonHeight)
        .aspectRatio(Constants.ASPECT_RATIO)      // 172 / 62: the stop button's shape
    }
    .justifyContent(FlexAlign.End)
    .padding({
      top: this.getUIContext().px2vp(this.topRectHeight),
      bottom: this.getUIContext().px2vp(this.bottomRectHeight)
    })
  }
}
```

**`aspectRatio` with a ratio expressed as a division** is a small habit worth
copying: `172 / 62` records the design's dimensions rather than the computed
2.774, so a change to either is obvious.

Flipping `isFinish` inside the third `onFinish` and letting `build` swap the
branch is what makes the overlay become the button - the shrink animation then
runs against the new branch's `Button`.

## Permissions & config

**None.** The sample declares no `requestPermissions` - it is an animation.

No routing configuration; a single `@Entry` page composing three components.

`Constants.ets` holds `COUNTDOWN_NUMBER` (4), `START_NUMBER` (3),
`DURATION_1000`, `DURATION_500` and `ASPECT_RATIO`. Note the two starting
numbers: `COUNTDOWN_NUMBER` seeds the countdown and `START_NUMBER` is what it
is reset to afterwards, so the names do not distinguish their roles.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The countdown takes a fixed 3.5 seconds** and cannot be skipped - there is
  no tap-to-dismiss and, as shipped, no way to cancel it at all.
- **`onFinish` does not fire if the animation is interrupted** by the
  component being destroyed, which is what makes the missing cancellation
  guard a correctness issue rather than only a cleanup one.
- The overlay assumes it covers the whole screen; the button branch pads for
  the avoid areas but the counting branch does not, so a number is centred in
  the full display including the status bar.
- Seven `@Link` fields is a wide interface for one overlay - the component
  cannot be reused anywhere that does not have all seven.

## Pitfalls

- **`HW-03-0046` — the chain is four `animateTo` calls nested by hand,** so
  the number of steps is encoded in the code's shape while the starting value
  is a constant. Raising `COUNTDOWN_NUMBER` does not add a step.
- **`HW-03-0047` — the chain cannot be cancelled,** and its last `onFinish`
  writes four `@Link` fields, so backing out mid-countdown still flips the
  parent into the started state 3.5 seconds later. Starting from `onAppear`
  also lets a second chain interleave with a running one.
- **`HW-03-0048` — `buttonColor` is typed `Length`** and initialised with a
  colour resource, copied from the `buttonHeight` line above it.

## References

- `documentation/harmonyos-references/02_application-framework/ts-explicit-animation.md` - `animateTo`, `AnimateParam`, `onFinish`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-explicit-animation
- `documentation/harmonyos-references/02_application-framework/ts-appendix-enums.md` - `Curve` and `PlayMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-appendix-enums
- `documentation/harmonyos-references/02_application-framework/ts-types.md` - `Length` and `ResourceColor`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-types
- `documentation/harmonyos-references/02_application-framework/ts-universal-events-show-hide.md` - `onAppear`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-events-show-hide
- `documentation/harmonyos-guides/03_application-framework/arkts-link.md` - `@Link`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-link
- `SPORT-03` - the run screen this countdown hands over to
- `TOUR-05` - the industry-neighbour's `animateTo` sample, animating position instead of a sequence
