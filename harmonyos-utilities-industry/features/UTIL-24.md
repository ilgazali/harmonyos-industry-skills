---
id: UTIL-24
title: Custom-start timer - two stacked TextTimers swapped by zIndex, with the display rewritten by a ContentModifier
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/24_custom_start_time_timer.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_start_time_timer-0000002330585486
sample: huawei_industry_tree/15_utilities/downloads/CustomStartTimeTimer.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [hilog, window, TextTimer, TextTimerController, TextTimerConfiguration, ContentModifier, wrapBuilder, WrappedBuilder, CustomDialogController, "@CustomDialog", "UIContext.animateTo", "UIContext.getPromptAction", AppStorage, "@StorageProp", "@Extend"]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0057, HW-15-0058, HW-15-0101]
status: verified-with-fixes
---

## When to use

**Load this card when the built-in `TextTimer` is the right engine but its
rendering is wrong for you** - you need a different format, a different layout,
an offset start value, or lap records alongside it.

The key API is `contentModifier`. A `ContentModifier<TextTimerConfiguration>`
lets you keep `TextTimer`'s ticking, pausing and resetting while replacing
everything it draws with your own `@Builder`. You receive `elapsedTime` and
`isCountDown` in the config and render whatever you like. That is what makes
"start from 12:34 instead of zero" possible without writing a timer at all:
the component still counts from zero, and the modifier adds the offset at
display time.

The second idea is the mode switch. Count-up and count-down are two separate
`TextTimer` instances stacked in the same `Stack`, each with its own
`TextTimerController`, swapped by `zIndex`. Neither is ever destroyed, so
switching modes and switching back does not lose a running count. That
generalises to any pair of exclusive stateful widgets you want to toggle
between without tearing down - two media players, two forms, two maps.

The scenario the document names is marathon leg timing: continue counting from
the previous leg's total. Interval training, cooking stages and billable-time
tracking are the same shape.

## Feature checklist

- A page with a 正计时 (count up) / 倒计时 (count down) segmented switch that
  slides its white pill with a 300 ms animation.
- A large timer readout, `H:MM:SS.cc`, filling a white card.
- 设置开始时间 opens a modal dialog with a numeric input; confirming resets the
  active timer and stores the value as its base.
- The input caps at 86400 (24 h), raising a toast, and strips leading zeros.
- 开始 / 暂停 toggles the active timer; 重置 clears it; while a count-up is
  running the left button becomes 记录 and appends a lap.
- The lap list shows each record's absolute end time (base + elapsed) and the
  split against the previous lap.
- Reaching 24 h on a count-up, or zero on a count-down, raises a dialog and
  stops the timer.
- Leaving the page pauses both timers.

## Architecture

One `entry` module, three source files. The split is by *role*, not by
feature: the page owns state and controllers, the viewmodel owns rendering and
formatting.

```
entry/src/main/ets
├── common/Constants.ets            spacing, font sizes, radii, MAXIMUM_TIME = 86400
├── entryability/EntryAbility.ets   window setup, avoid areas -> AppStorage
├── entrybackupability/
├── pages/Timer.ets                 @Entry Timer + @CustomDialog initTimeDialog + two @Extend styles
└── viewmodel/TextTimerModifier.ets the ContentModifier, buildTextTimer, and every format helper
```

The documented tree matches the zip.

**The design decision worth copying** is that the base time lives in
`AppStorage`, not in a prop:

```typescript
AppStorage.setOrCreate('initCountUpTime', this.countUp);
// ...
const INITTIME = AppStorage.get('initCountUpTime') as number;
```

`buildTextTimer` is a **global** `@Builder` wrapped by `wrapBuilder` - it is
not a method on the component and has no `this`. It receives exactly one
argument, the `TextTimerConfiguration` the framework hands it, and that
carries `elapsedTime` and `isCountDown` and nothing else. So there is no way to
pass the base time down the normal path. `AppStorage` is the side channel that
makes the offset reachable from a context-free builder, and it is the right
answer here rather than a shortcut.

The cost is that the same builder's helpers must each defend against the key
being unset. Two of the three do (`?? 0`, `?? 60`); the third does not, which
is `HW-15-0058`.

## Implementation steps

1. **Implement `ContentModifier<TextTimerConfiguration>`** with a single
   `applyContent()` returning `wrapBuilder(buildTextTimer)`, and hold one
   instance in `@State` on the page.
2. **Keep the builder global and pure.** It gets `config.elapsedTime` (in
   centiseconds) and `config.isCountDown`; anything else has to come from
   `AppStorage`.
3. **Stack the two timers** in one `Stack`, both carrying the same modifier,
   and swap them with `.zIndex(this.isCountUp ? 1 : 0)` and its inverse.
4. **Give each timer its own `TextTimerController`** and pause the other one
   whenever the switch is tapped, or a background count keeps ticking.
5. **Pass the count-down seed in milliseconds**: `count: this.countDown * 1000`.
   `onTimer`'s `elapsedTime`, by contrast, is in **centiseconds** - the two
   units differ and every comparison in the file has to respect that.
6. **Store the base in `AppStorage` from the dialog's confirm handler**, then
   `reset()` the active controller so the display restarts from the new base.
7. **Add the base back at display time**: `timeFormat(elapsed / 100 + base)`.
   The component itself always counts from zero.
8. **Default the base whenever you read it** (`HW-15-0058`). `undefined +
   number` is `NaN`, and every comparison against `NaN` is false, so a missing
   default silently disables the guard that uses it.
9. **Derive the countdown's fractional digits from the remaining time itself**,
   not from `100 - elapsed` (`HW-15-0057`).
10. **Pause both controllers in `onPageHide`** so a backgrounded page does not
    keep counting.

## Verified snippets

All snippets are from `CustomStartTimeTimer.zip`. Corrected forms are marked.

**The modifier and its builder — `entry/src/main/ets/viewmodel/TextTimerModifier.ets`** (as shipped)

```typescript
export class TextTimerModifier implements ContentModifier<TextTimerConfiguration> {
  constructor() {
  }

  applyContent(): WrappedBuilder<[TextTimerConfiguration]> {
    return wrapBuilder(buildTextTimer);
  }
}

@Builder
function buildTextTimer(config: TextTimerConfiguration) {
  Column() {
    Stack({ alignContent: Alignment.Center }) {
      Column() {
        Text(
          // 判断是否为倒计时模式，分别获取计时器实时时间
          config.isCountDown ?
            getCountDownTime(config.elapsedTime) : getCountUpTime(config.elapsedTime)
        )
          .fontSize(Constants.FONT_SIZE_TIMER);
      }
      .backgroundColor($r('sys.color.white'))
      .width(Constants.FULL_PERCENT)
      .height(Constants.HEIGHT_TEXT_TIMER)
      .borderRadius(Constants.BORDER_RADIUS_NORMAL)
      .justifyContent(FlexAlign.Center)
      .margin({ top: Constants.SPACE_8 });
    };
  };
}
```

**`applyContent` is called once, the builder on every tick.** The modifier
object is therefore cheap to hold in `@State` and is deliberately stateless -
its constructor is empty. Everything variable arrives through `config`.

`wrapBuilder` is what turns a global `@Builder` function into the
`WrappedBuilder<[TextTimerConfiguration]>` the interface demands, and the
builder has to be global precisely because a `WrappedBuilder` cannot capture a
component instance. That constraint is the reason the whole formatting layer
sits in the viewmodel file rather than on the page: it *has* to be reachable
without `this`.

One modifier instance serves both `TextTimer`s. `config.isCountDown` tells the
builder which one it is rendering for, so the two timers can share it.

**The formatters — same file** (corrected, see `HW-15-0057`)

```typescript
const TIME_CONVERSION = 60;

export function getCountUpTime(time: number) {
  const INITTIME = AppStorage.get('initCountUpTime') as number;
  let countUpTime = timeFormat(time / 100 + (INITTIME ?? 0)) +
    `.${getStrFragment((time / 100).toFixed(2))}`;
  if (time / 100 + (INITTIME ?? 0) >= Constants.MAXIMUM_TIME) {   // FIX: guard was NaN-blind
    countUpTime = timeFormat(Constants.MAXIMUM_TIME);
  }
  return countUpTime;
}

function getCountDownTime(time: number) {
  // FIX: the sample formats the fraction from (100 - time / 100), which goes
  // negative once the elapsed time passes 100 s and inverts the digits.
  const remaining = (AppStorage.get('initCountDownTime') as number ?? 60) - time / 100;
  return timeFormat(remaining) + `.${getStrFragment(remaining.toFixed(2))}`;
}

function getStrFragment(str: string) {
  let length: number = str.length;
  return str.slice(length - 2);
}
```

**`getStrFragment` is a string slice, not arithmetic** - it takes the last two
characters of a `toFixed(2)` result, which is the two digits after the decimal
point. That is a neat trick and it is also the source of the bug: it has no
idea what number produced the string, so a negative value like `-12.34`
yields `34` with no complaint.

For the count-up the fraction of `elapsed` is the right thing, and it is
always positive. For the count-down the sample writes `(100 - time / 100)`.
While the elapsed time is under 100 s, the fractional part of `100 - elapsed`
happens to equal the fractional part of the remaining time (the base is a whole
number), so it looks correct in a short demo. Past 100 s the expression turns
negative and the displayed centiseconds run backwards - `.01` where `.99`
belongs - for the entire rest of a countdown that may legitimately run to
86400 s. Computing the fraction from `remaining` itself is both correct and
simpler, and it reuses the value already needed for `timeFormat`.

`timeFormat` takes **seconds** and builds `H:MM:SS` by hand, padding minutes
and seconds to two digits and letting hours run unpadded. Note that
`elapsedTime` from `onTimer` is in centiseconds throughout, hence the `/ 100`
at every entry point.

**Two timers, one Stack — `entry/src/main/ets/pages/Timer.ets`** (corrected, see `HW-15-0058`)

```typescript
@Builder
textTimerBuilder() {
  Stack() {
    // 正计时计时器
    TextTimer({ isCountDown: false, controller: this.countUpTextTimerController })
      .contentModifier(this.myTimerModifier)
      .zIndex(this.isCountUp ? 1 : 0)
      .onTimer((utc: number, elapsedTime: number) => {
        this.elapsedTime = elapsedTime;
        // FIX: `as number` on an unset AppStorage key gives undefined; undefined + n = NaN,
        // and NaN >= 86400 is always false, so the 24-hour cap never fired on a first run.
        if (elapsedTime / 100 + (AppStorage.get('initCountUpTime') as number ?? 0) >=
          Constants.MAXIMUM_TIME) {
          this.context.getPromptAction().showDialog({
            message: $r('app.string.dialog_message_2')
          });
          this.countUpTextTimerController.pause();
          this.isStart = true;
        }
      });

    // 倒计时计时器
    TextTimer({ isCountDown: true, count: this.countDown * 1000, controller: this.countDownTextTimerController })
      .contentModifier(this.myTimerModifier)
      .zIndex(this.isCountUp ? 0 : 1)
      .onTimer((utc: number, elapsedTime: number) => {
        if (elapsedTime >= this.countDown * 100) {
          this.context.getPromptAction()
            .showDialog({
              message: $r('app.string.dialog_message_1')
            });
          this.countDownTextTimerController.reset();
          this.isStart = true;
        }
      });
  };
}
```

**`zIndex` swaps which timer the user sees; both keep existing.** That is the
point of the stack - `TextTimer` has no "hide" that preserves its count, and
destroying the component would lose it. The switch handlers pause the *other*
controller before animating `isCountUp`, so the hidden one is stopped rather
than merely covered.

**Watch the three different units in this one builder.** `count` is
milliseconds (`this.countDown * 1000`). `elapsedTime` in `onTimer` is
centiseconds. The countdown's completion test therefore compares against
`this.countDown * 100`, and the count-up's cap divides by 100 before comparing
against `MAXIMUM_TIME` in seconds. Getting any of those factors wrong produces
a timer that is off by 10× and still looks plausible.

The `NaN` defect is the subtler one. `AppStorage.get('initCountUpTime') as
number` is a cast, not a conversion: before the dialog has ever been confirmed
the key does not exist and the cast yields `undefined`. `number + undefined` is
`NaN`, and `NaN >= 86400` is `false` - so the 24-hour protection is silently
absent until the user sets a custom start time at least once. The display
helpers in the viewmodel already use `?? 0`; this check simply did not.

**The dialog and the base time — same file** (as shipped)

```typescript
dialogController: CustomDialogController = new CustomDialogController({
  builder: initTimeDialog({
    time: this.initTime,
    cancel: () => { this.onCancel(); },
    confirm: (time) => {
      this.onAccept(time);
      this.initTime = time;
    },
  }),
  isModal: true
});

onAccept(time: string) {
  // 判断计时方式，获取用户输入时间，重置计时器并设置计时开始时间
  if (this.isCountUp) {
    this.countUp = Number(time);
    AppStorage.setOrCreate('initCountUpTime', this.countUp);
    this.isStart = true;
    this.countUpTextTimerController.reset();
    this.countUpTime = [];
  } else {
    this.countDown = Number(time);
    AppStorage.setOrCreate('initCountDownTime', this.countDown);
    this.isStart = true;
    this.countDownTextTimerController.reset();
  }
}
```

```typescript
// inside @CustomDialog initTimeDialog
TextInput({ placeholder: $r('app.string.textInput_placeholder'), text: $$this.time })
  .type(InputType.Number)
  .onChange(() => {
    if (Number(this.time) > Constants.MAXIMUM_TIME) {
      this.time = '86400';
      this.getUIContext().getPromptAction()
        .showToast({ message: $r('app.string.toast_message_2'), duration: 1000 });
    }
    if (this.time.startsWith('0') && this.time.length > 1) {
      this.time = this.time.slice(1);
    }
  });
```

**One handler, two very different effects.** For count-up the stored value is
purely cosmetic - it is added at *display* time by `getCountUpTime`, and the
component still counts from zero. For count-down it is structural: `countDown`
feeds `count: this.countDown * 1000`, which re-seeds the actual timer. The
same dialog therefore does presentation in one mode and configuration in the
other, and `reset()` is needed in both to make the change visible immediately.

Clearing `countUpTime` on a count-up confirm is correct: the lap list stores
elapsed values relative to the old base, so a new base invalidates them.

The input validation lives in `onChange` with `$$this.time` two-way binding,
which means a paste that exceeds 24 h is corrected in place as the user types
rather than rejected on confirm. `InputType.Number` plus the leading-zero
strip keeps the string safe for `Number()`.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

Values worth knowing about the configuration rather than the manifest:

- `Constants.MAXIMUM_TIME = 86400` is the 24-hour cap, used in three places -
  the dialog's input validation, the count-up `onTimer` guard, and the ceiling
  clamp inside `getCountUpTime`.
- `Constants.ANIMATION_DURATION = 300` drives the segmented switch's sliding
  pill through `this.context.animateTo({ duration }, () => { this.isCountUp = ... })`.
- Two `@Extend(Text)` functions, `switchTextStyle` and `listTextStyle`, keep
  the repeated typography out of the layout - a good habit at this file size.
- `safeAreaPadding` is fed from `@StorageProp('topRectHeight')` /
  `bottomRectHeight` converted with `px2vp`, the same avoid-area boilerplate
  the other samples in this industry use.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`,
  so the constraint and the sample agree.
- `contentModifier` on `TextTimer` is available from API 12.
- **The base time is display-only for count-up.** Nothing persists across a
  cold start: `AppStorage` is process memory, so a relaunch loses the base and
  the lap list.
- `onPageHide` pauses both timers, so a backgrounded app does not continue
  counting. A real stopwatch would record a wall-clock timestamp and
  reconstruct the elapsed time on return - `TextTimer` alone cannot do that.
- The lap list is only rendered in count-up mode (the `ForEach` body is wrapped
  in `if (this.isCountUp)`), and the records are held in a plain array with no
  persistence.
- The back arrow in the header has an empty `onClick` - the page is a
  standalone demo with no navigation.
- Counting down from 0 is blocked with a toast, but the guard reads
  `AppStorage.get('initCountDownTime')`, which is unset until the dialog is
  used once; the default seed of 60 s comes from the `@State` field instead.

## Pitfalls

- **`HW-15-0057`** (B/medium, confirmed): `TextTimerModifier.ets:93-96`
  derives the countdown's centiseconds from `(100 - elapsed).toFixed(2)`. For
  countdowns longer than 100 s - and 86400 is supported - the value goes
  negative and `getStrFragment` slices inverted digits out of it, showing
  `.01` where `.99` belongs, for the whole tail of the countdown. Fix: compute
  the fraction from the remaining time itself.
- **`HW-15-0058`** (B/low, confirmed): `Timer.ets:278` reads
  `AppStorage.get('initCountUpTime') as number` with no fallback. Before the
  dialog is ever used the key is unset, so the cast yields `undefined`, the sum
  is `NaN`, and `NaN >= 86400` is always false - the 24-hour cap is silently
  disabled until a custom start time has been set once. The display helpers in
  the same feature already use `?? 0`. Fix: add the same fallback here.
- Unit confusion is the standing hazard in this file: `count` is milliseconds,
  `onTimer`'s `elapsedTime` is centiseconds, `MAXIMUM_TIME` and the stored
  bases are seconds. Every arithmetic site has a `* 1000`, a `* 100` or a
  `/ 100` and none of them are interchangeable.
- The global `@Builder` reached through `wrapBuilder` cannot see component
  state; anything it needs must travel through `TextTimerConfiguration` or
  `AppStorage`. Do not try to close over `this`.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-texttimer.md` -
  `TextTimer`, `count`, `isCountDown`, `onTimer`, `TextTimerController`
  (`start` / `pause` / `reset`), `contentModifier` and `TextTimerConfiguration`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-texttimer
- `documentation/harmonyos-references/02_application-framework/ts-methods-custom-dialog-box.md` -
  `@CustomDialog`, `CustomDialogController`, `isModal`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-methods-custom-dialog-box
- `huawei_industry_tree/15_utilities/docs/24_custom_start_time_timer.md` - the
  source architecture guide
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_start_time_timer-0000002330585486
