---
id: SHOP-17
title: Red envelope rain - frame animation (Animator) so falling items stay clickable in flight
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/17_red_envelope_rain.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/red_envelope_rain-0000002370873497
sample: huawei_industry_tree/16_shopping/downloads/RedEnvelopeRain.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: ["UIContext.createAnimator", AnimatorOptions, AnimatorResult, onFrame, onFinish, "@ObservedV2", "@Trace", ForEach, onAppear, onAreaChange, position, scale, "window.getLastWindow", setWindowLayoutFullScreen, setInterval, hilog]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-16-0020, HW-16-0021, HW-16-0027]
status: verified-with-fixes
---

## When to use

Load this card when you need **many objects moving on screen that the user must
be able to hit while they are still moving** - a red-envelope rain, a coin
drop, falling coupons, a whack-a-mole grid. The defining constraint is not the
motion, it is the hit test.

The whole reason this sample exists is one sentence in the document: property
animation (`animateTo` on a `translate` or `offset`) animates the *rendered*
position but the component's hit region stays at the final layout position, so
a tap during the flight either misses or lands on the wrong envelope. Frame
animation (`UIContext.createAnimator`) hands you an interpolated number on
every frame; you write it into observed state, the layout property
(`position`) actually changes, and the hit region follows the pixels.

It generalises to anything where "where it looks" and "where it is" must agree
during an animation. If your falling item is decorative and never tapped, use
`animateTo` instead - it is cheaper and does not run a JS callback per frame
per item.

## Feature checklist

- The page opens full-screen over a background image with a dark scrim.
- A 3-2-1 countdown occupies the screen before the rain starts.
- Twenty envelopes fall from above the top edge to below the bottom edge.
- Each envelope has a random horizontal position and a random start delay of up
  to 10 s, so they do not fall as a block.
- Tapping an envelope mid-flight makes it vanish immediately.
- When the last envelope finishes falling, the rain is replaced by a reward
  panel with a close button.
- Closing the panel raises a toast (the sample has no reward backend).

## Architecture

One `entry` module, three source files in `pages/`. There is no model layer
beyond one small observed class.

```
entry/src/main/ets
├── entryability/EntryAbility.ets   loads pages/Index, forces light color mode
├── entrybackupability/
└── pages
    ├── Index.ets                   @Entry: countdown, the 20 envelopes, the reward panel
    ├── RedEnvelope.ets             @ObservedV2 class: id, x/y, scale, its AnimatorResult
    └── constant.ets                one exported hilog TAG
```

**The documented tree does not match the zip** (`HW-16-0020`): the document
lists `Constant.ets` with a capital C where the file is `constant.ets`, and
annotates `Index.ets` as `// 优惠券页面` (coupon page) - a comment
copy-pasted from a different scenario. The sample projects enable
`caseSensitiveCheck`, so the capitalised name is not a cosmetic slip; an import
written from the document fails to resolve on a case-sensitive build.

**The design decision worth copying** is that the per-envelope animation state
lives on the envelope object, not on the page. `RedEnvelope` is `@ObservedV2`
with `@Trace offsetX / offsetY / scaleX / scaleY`, and it also carries its own
`fallAnimation?: AnimatorResult` - deliberately *not* traced, because the
animator is machinery, not display state. The page holds a plain
`Array<RedEnvelope>` and the `ForEach` binds each `Image` to one object's
traced fields. That is what makes twenty independent animations cheap: an
`onFrame` callback writes `item.offsetY` and V2 observation re-renders exactly
that one `Image`, not the array and not the page.

The counterpart of that choice is that the page owns nothing it can clean up
generically - which is precisely the shape of `HW-16-0021`.

## Implementation steps

1. **Model the envelope as an `@ObservedV2` class** with `@Trace` on the four
   geometry fields only, and an untraced `fallAnimation?: AnimatorResult`.
   Seed `offsetY` to `'-10%'` so envelopes start above the viewport.
2. **Measure the play area from the container**, not from `display`. The outer
   `Stack` carries `onAreaChange` and writes `screenWidth` / `screenHeight`;
   those two numbers are the animator's `end` value and the random-x range,
   and they stay correct in a resized window.
3. **Create the animator in `onAppear` of each item**, not in `aboutToAppear`
   of the page - `onAppear` fires once the component exists and the area has
   been measured.
4. **Give every envelope a random `delay`** (`Math.random() * 10000`) and a
   random x. A shared duration with staggered delays is what turns twenty
   identical animations into rain.
5. **Write the interpolated value into traced state in `onFrame`** - this is
   the step that keeps the hit region under the pixels.
6. **Count completions in `onFinish`** and switch to the reward panel when the
   count equals the array length, releasing every `AnimatorResult`.
7. **Cancel on the way out.** Add `aboutToDisappear` that cancels every
   animator and clears the countdown interval (`HW-16-0021`); the sample only
   releases animators after they all finish naturally.
8. **Name the constants file `constant.ets`** and import it with that exact
   case (`HW-16-0020`).

## Verified snippets

All snippets are from `RedEnvelopeRain.zip`. Corrected forms are marked.

**The envelope model — `entry/src/main/ets/pages/RedEnvelope.ets`** (as shipped)

```typescript
import { AnimatorResult } from '@kit.ArkUI';

let redEnvelopeId = 0;

@ObservedV2
export default class RedEnvelope {
  id: number = 0;
  @Trace offsetX: number;
  @Trace offsetY: string | number;   // starts as '-10%', becomes a number from onFrame
  @Trace scaleX: number;
  @Trace scaleY: number;
  fallAnimation?: AnimatorResult;    // machinery, not display state - deliberately untraced

  constructor() {
    this.id = redEnvelopeId++;
    this.offsetX = 0;
    this.offsetY = '-10%';
    this.scaleX = 1;
    this.scaleY = 1;
  }
}
```

**Three decisions carry this class.** `@ObservedV2` / `@Trace` instead of V1
`@Observed` / `@ObjectLink` means the page can hold a plain array and still get
per-field invalidation - no wrapper component per item, no `@State` array
churn. `offsetY` is typed `string | number` so the initial value can be the
percentage `'-10%'` (above the viewport, independent of screen height) and then
switch to the absolute vp numbers the animator produces. And `fallAnimation` is
outside the observation graph: tracing it would invalidate the item every time
the animator object is assigned or released, for no visual reason.

`redEnvelopeId` is a module-level counter, so ids are unique across the whole
run rather than per array.

**The animator, created per item in `onAppear` — `entry/src/main/ets/pages/Index.ets`** (as shipped)

```typescript
ForEach(this.redEnvs, (item: RedEnvelope) => {
  Image($r('app.media.redEnvelope'))
    .size({ width: this.redEnvWidth })
    .position({ x: item.offsetX, y: item.offsetY })
    .scale({ x: item.scaleX, y: item.scaleY })
    .onAppear(() => {
      item.offsetX = this.getRandomXAxis();
      const options: AnimatorOptions = {
        duration: 2000,
        easing: 'linear',
        delay: this.getAnimationDelay(),   // 0-10 s: what staggers the rain
        fill: 'forwards',
        direction: 'normal',
        iterations: 1,
        begin: 0,                          // onFrame first-frame value
        end: this.screenHeight             // onFrame last-frame value
      };
      item.fallAnimation = this.getUIContext().createAnimator(options);
      item.fallAnimation.onFrame = (value: number) => {
        item.offsetY = value;              // layout position, so the hit region follows
      };
      item.fallAnimation.onFinish = () => {
        this.finishAnimationCount++;
        if (this.finishAnimationCount === this.redEnvs.length) {
          this.allAnimationFinish();
        }
      };
      item.fallAnimation.play();
    })
    .onClick(() => {
      item.scaleX = 0;                     // "collected": scaled to nothing, not removed
      item.scaleY = 0;
    })
})
```

**`onFrame` writing to `position`, not to `translate`, is the entire point.**
`begin`/`end` are not pixels to the animator - they are just the interpolation
range, and it is your callback that decides what they mean. Here they mean "top
of screen" to "bottom of screen", written into a `@Trace`d field bound to
`position({ y })`, which is a layout property. That is why `.onClick` fires on
a moving envelope: the component really is where it is drawn. Had the same
value gone into `.translate({ y })` inside an `animateTo`, the draw would move
and the touch target would not.

`fill: 'forwards'` keeps the last frame's value after the animation ends, so
envelopes rest off the bottom edge instead of snapping back to `-10%`.

The tap handler zeroes the scale rather than splicing the array. With a
`ForEach` over an array whose keys are positional that is the safer move -
removing an element mid-flight would re-key the survivors and hand their
`onAppear` a second animator.

**Cleanup on leaving the page — `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-16-0021`)

```typescript
private countdownTimerId: number = -1;    // FIX: shipped code keeps this local to startCountdown

startCountdown() {
  this.countdownTimerId = setInterval(() => {
    this.countdown--;
    if (this.countdown < 0) {
      clearInterval(this.countdownTimerId);
    }
  }, 1000);
}

allAnimationFinish() {
  this.isAllAnimationFinish = true;
  this.redEnvs.forEach(envelope => {
    envelope.fallAnimation = undefined;   // 释放动画对象 (release the animator)
  });
}

aboutToDisappear(): void {               // FIX: absent in the sample
  clearInterval(this.countdownTimerId);
  this.redEnvs.forEach(envelope => {
    envelope.fallAnimation?.cancel();
    envelope.fallAnimation = undefined;
  });
}
```

**Frame animation is the one animation type you must cancel yourself.** An
`animateTo` dies with the component tree; an `AnimatorResult` is a JS object
holding a closure over `item`, and its `onFrame` keeps being invoked. Leave the
page during the rain and up to twenty callbacks per frame keep writing into
state nobody renders, plus the countdown interval survives until its own
`countdown < 0` test happens to fire. `cancel()` stops the animator without
running `onFinish`, which is what you want here - the reward panel must not
appear because the user navigated away.

Note also that the sample's timer id is a `const` inside `startCountdown`, so
nothing outside that closure can clear it; hoisting it to a field is part of
the fix.

**Full screen — `entry/src/main/ets/pages/Index.ets`** (as shipped)

```typescript
async setFullScreen(): Promise<void> {
  try {
    const context = this.getUIContext().getHostContext();
    window.getLastWindow(context).then(topWindow => {
      topWindow.setWindowLayoutFullScreen(true);
    });
  } catch (e) {
    hilog.error(0x0000, TAG, '设置全屏失败');   // "failed to set full screen"
  }
}
```

Worth reading as a counter-example. The `try/catch` cannot see a rejection from
either promise - `getLastWindow` rejecting produces an unhandled rejection, and
`setWindowLayoutFullScreen` has no handler at all - so the log line is
unreachable for the failures it names. Either `await` both calls inside the
`try`, or attach `.catch()` to each. The pattern the sibling samples use
(`SHOP-19`, `SHOP-20`) is to do this in `onWindowStageCreate` with an explicit
`.catch((err: BusinessError) => ...)`, which is the better shape: full-screen
is a window concern, not a page concern, and doing it in the ability avoids a
visible relayout on the first frame.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

`deviceTypes` is `["phone"]` only - narrower than the other shopping samples,
which is honest: the geometry is a full-screen portrait rain and the layout has
no tablet or 2in1 story.

The countdown, the envelope count (20), the envelope width (51 vp), the fall
duration (2000 ms) and the delay ceiling (10000 ms) are all literals in
`Index.ets`. `constant.ets` holds exactly one export, the hilog `TAG`. If you
lift this pattern, move the five tuning numbers there too - the delay ceiling
in particular has to be re-picked whenever the envelope count changes, because
rain density is `count / delayCeiling`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Twenty envelopes means twenty JS `onFrame` callbacks per frame. That is
  comfortable at this count; frame animation does not scale to hundreds of
  items the way a property animation does, because every frame crosses into the
  ArkTS runtime. Budget accordingly before raising `redEnvCount`.
- With `delay` up to 10 s and `duration` 2 s, the rain lasts up to 12 s and the
  reward panel only appears after the *slowest* envelope lands - even if the
  user collected all twenty in the first second. A real implementation should
  end the round on collection count or on a wall clock, not on `onFinish`.
- Collected envelopes are scaled to zero, not removed, so their animators keep
  running to completion. That is intentional for the completion count, but it
  means "collected" costs the same as "falling".
- There is no reward: `onClick` shrinks the image and the close button raises a
  toast. Nothing is credited anywhere.

## Pitfalls

- **`HW-16-0020`** (E/low, confirmed): the project tree lists `Constant.ets`
  where the zip has `constant.ets`, and annotates `Index.ets` as the coupon
  page - a comment carried over from another scenario doc. Fix: rename the tree
  entry to `constant.ets` and label `Index.ets` as 红包雨页面.
- **`HW-16-0021`** (B/low, probable): the animators are never cancelled and the
  countdown interval is never cleared when the page is left mid-rain; animators
  are only released after all twenty finish naturally
  (`finishAnimationCount === redEnvs.length`). Fix: add `aboutToDisappear`
  that calls `fallAnimation?.cancel()` on every envelope, drops the references,
  and clears the interval - which first requires hoisting the timer id out of
  `startCountdown`.
- **The document's step-2 excerpt is not runnable.** It elides `onFinish` and,
  more importantly, the `item.fallAnimation.play()` call, so a reader who
  copies it gets an animator that is created and never started - nothing falls.
  This doc is not among the instances enumerated in `HW-16-0013`, the corpus-wide
  finding on abridged snippets, but it is the same editorial failure: the
  abridgement removes load-bearing statements rather than only bodies.
- **`setFullScreen` swallows nothing it thinks it swallows** - both window
  promises escape the `try/catch`. Not filed as a finding; noted because the
  same code is easy to copy verbatim.
- **Step 1 of the document declares `@State redEnvCount`** where the zip has a
  plain field. `redEnvCount` is read once in `createRedEnvelope`; decorating it
  buys nothing and costs an observation slot.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-animator.md` - `createAnimator`, `AnimatorOptions`, `onFrame`, `onFinish`, `cancel`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-animator
- `documentation/harmonyos-references/02_application-framework/js-apis-animator.md` - the `AnimatorResult` API surface
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-animator
- `documentation/harmonyos-guides/03_application-framework/arkts-new-observedv2-and-trace.md` - `@ObservedV2` / `@Trace` and per-field invalidation
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-observedv2-and-trace
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-transformation.md` - `scale`, and why `translate` is not used here
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-transformation
- `documentation/harmonyos-references/02_application-framework/ts-explicit-animation.md` - `animateTo`, the property-animation alternative this sample rejects
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-explicit-animation
- `huawei_industry_tree/16_shopping/docs/17_red_envelope_rain.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/red_envelope_rain-0000002370873497
- `SHOP-18` - the other `@ObservedV2` / `@Trace` sample in this industry
