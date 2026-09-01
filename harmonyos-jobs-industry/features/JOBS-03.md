---
id: JOBS-03
title: Stacked job cards - a three-slot deck swiped with PanGesture and animated with animateTo
industry: 12_jobs
doc: huawei_industry_tree/12_jobs/docs/03_position_sliding_window.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/position_sliding_window-0000002391546225
sample: huawei_industry_tree/12_jobs/downloads/PositionSlidingWindow.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [Stack, PanGesture, PanGestureOptions, GestureGroup, "UIContext.animateTo", "@ObservedV2", "@Trace", "display.getDefaultDisplaySync", "UIContext.px2vp", zIndex, offset, opacity, ForEach, "@StorageProp", "UIContext.getPromptAction"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-12-0002, HW-12-0003, HW-12-0005]
status: verified-with-fixes
---

## When to use

Load this card for a **card deck the user swipes through** — recommended jobs
here, but the same shape covers dating profiles, product recommendations,
flashcards, and any "one item at a time, with a peek at the neighbours" list.
Three cards are visible: a large one centred, a smaller one peeking on each
side. A horizontal pan rotates the deck; two buttons consume the centre card.

The transferable technique is that **the deck is not a scroller**. There is no
`Swiper`, no `List`, no scroll offset. Every card carries its own
`offsetX`/`size`/`zIndex`/`opacity`, a `Stack` overlaps them all, and each
gesture is a reassignment of four numbers on three or four objects inside one
`animateTo` closure. That gives you effects a scroller cannot express — scale
and depth ordering changing mid-transition — at the cost of owning the index
arithmetic yourself.

**Read `HW-12-0002` before adopting it.** The sample's array is not observed,
so "deleting" a card only makes it invisible; it stays mounted and keeps its
gesture handlers. That is the load-bearing flaw of the whole design.

## Feature checklist

- Three job cards visible at once: centre card at full size, left and right
  neighbours at 83% and behind it.
- A horizontal pan of at least 100 vp rotates the deck one position, in either
  direction.
- The transition is a 500 ms `EaseInOut` animation of position, scale and
  opacity together.
- The deck wraps: past the last card it continues from the first.
- 立即投递 (apply now) and 不感兴趣 (not interested) each remove the centre
  card with a fly-away animation and raise a toast.
- Once the deck is empty the prompt line and both buttons disappear.
- The back arrow in the title bar is a demo toast, not navigation.

## Architecture

One `entry` module, one page, one constants file. No model layer, no
repository — the card model is a class in the constants file.

```
entry/src/main/ets
├── constants/StyleConstants.ets     StyleConstants (3 toast strings) + the @ObservedV2 Card class
├── entryability/EntryAbility.ets    full-screen window, avoid areas (px) -> AppStorage
├── entrybackupability/
└── pages/PositionSlidingPage.ets    @Entry: the deck, the gestures, the animations, TitleBar
```

The documented 工程目录 matches the zip exactly.

**The design decision worth copying** is `Card` as an `@ObservedV2` class whose
visual properties are `@Trace`:

```typescript
@ObservedV2
export class Card {
  img: Resource;
  id: number;
  @Trace offsetX: number = 0;
  @Trace offsetY: number = 0;
  @Trace zIndex: number = 0;
  @Trace size: number = 0;
  @Trace opacity: number = 0;
}
```

The card owns its own presentation state. `handleSlider` then reads as a
description of the target layout — "mid goes small and right, left becomes the
centre, the one past left fades in on the far side" — with no index-into-array
bookkeeping in the render tree. `@Trace` on the individual number fields is
what makes a mutation of `this.arr[this.mid].size` repaint one card without
touching the array.

**The design decision worth avoiding** is the other half of the same choice.
Because `Card` is `@ObservedV2`, the sample concluded the array holding those
cards needs no decoration at all, and declared it as a plain field. Per-object
observation and collection observation are different mechanisms: `@Trace`
repaints a card, it does not tell `ForEach` that the collection changed
(`HW-12-0002`). The tautological `(this.updateArrLength || !this.updateArrLength)`
guard in `build()` is the scar tissue from that gap.

## Implementation steps

1. **Compute the geometry from the display, once**, in `aboutToAppear`:
   `display.getDefaultDisplaySync()`, `px2vp` the width, then
   `cardWidth = windowWidth * 0.7833` and
   `rightCardOffsetX = windowWidth - 32 - cardWidth * 0.83`. Nothing about the
   layout is a raw literal except the ratios.
2. **Model each card as `@ObservedV2` with `@Trace` visual fields** so a
   mutation repaints exactly that card.
3. **Decorate the array too** — `@State arr: Card[]`, or move the page to
   `@ComponentV2` with `@Local` — so `splice` actually drives the `ForEach`
   (`HW-12-0002`). Then delete `updateArrLength` and write the guards as plain
   `if (this.arr.length > 0)`.
4. **Seed the slots in the constructor** by `id`: 0 is the left peek, 1 is the
   centre, 2 is the right peek, everything else is parked at the centre offset
   with `opacity: 0`.
5. **Stack them with `alignContent: Alignment.Start`** and drive
   `zIndex`/`opacity`/`offset` from the card object, never from the loop index.
6. **Attach the pan to each card**, not to the `Stack`, with
   `PanGestureOptions({ direction: PanDirection.Horizontal, distance: 100 })`.
7. **Derive the direction at the end of the gesture** from
   `event.offsetX > 0`, not from a threshold test at `onActionStart`
   (`HW-12-0003`) — otherwise a pan that activates inside the dead zone reuses
   the previous gesture's direction.
8. **Reassign the slots inside one `animateTo`** (500 ms, `Curve.EaseInOut`),
   then rotate the `left`/`mid`/`right` indices *after* the closure via
   `refresh()`.
9. **Handle the short-deck cases explicitly**: the code branches on
   `arr.length === 2` and `> 2` because with two cards there is no distinct
   left and right slot to rotate through.

## Verified snippets

All snippets are from `PositionSlidingWindow.zip`. Corrected forms are marked.

**The card model and its slot seeding — `entry/src/main/ets/constants/StyleConstants.ets`** (as shipped)

```typescript
import { display } from '@kit.ArkUI';

@ObservedV2
export class Card {
  img: Resource;
  id: number;
  @Trace offsetX: number = 0;
  @Trace offsetY: number = 0;
  @Trace zIndex: number = 0;
  @Trace size: number = 0;
  @Trace opacity: number = 0;

  constructor(img: Resource, id: number, uiContext: UIContext) {
    this.img = img;
    this.id = id;

    let displayClass: display.Display = display.getDefaultDisplaySync();
    let windowWidth = uiContext.px2vp(displayClass.width);
    let midCardOffsetX = 25;
    let rightCardOffsetX = windowWidth - 32 - (windowWidth * 0.7833 * 0.83);

    if (id === 0) {                    // left peek
      this.offsetX = 0;    this.size = 0.83; this.zIndex = 1; this.opacity = 1;
    } else if (id === 1) {             // centre
      this.offsetX = midCardOffsetX; this.size = 1; this.zIndex = 2; this.opacity = 1;
    } else if (id === 2) {             // right peek
      this.offsetX = rightCardOffsetX; this.size = 0.83; this.zIndex = 1; this.opacity = 1;
    } else {                           // parked off-stage
      this.offsetX = midCardOffsetX; this.size = 0.83; this.zIndex = 1; this.opacity = 0;
    }
  }
}
```

**Four numbers describe a slot, and only three slots exist.** `size` and
`zIndex` together are the depth cue — the centre card is both bigger and on
top; the peeks are 0.83 and behind. The parked cards are not unmounted, they
sit at the centre offset with `opacity: 0`, so promoting one into a visible
slot is a pure animation with nothing to mount.

`rightCardOffsetX` is `windowWidth - 32 - cardWidth * 0.83`: the right peek is
pinned to the right edge minus the page's 16 vp padding on each side, computed
from the actual scaled card width. That is the one piece of geometry that
survives a device change — the rest of the numbers (25 vp centre nudge, 569 vp
card height) are literals tied to the artwork.

Note that each `Card` calls `display.getDefaultDisplaySync()` in its own
constructor, so the deck of seven measures the display seven times. Hoisting
the measurement to the page and passing the width in would be strictly better.

**The deck and its gesture — `entry/src/main/ets/pages/PositionSlidingPage.ets`** (corrected, see `HW-12-0002`, `HW-12-0003`)

```typescript
private panOptionH: PanGestureOptions =
  new PanGestureOptions({ direction: PanDirection.Horizontal, distance: 100 });
@State arr: Card[] = [ /* 7 cards, ids 0..6 */ ];   // FIX: shipped as a plain, unobserved field

build() {
  Column() {
    TitleBar();

    Stack({ alignContent: Alignment.Start }) {
      ForEach(this.arr, (item: Card) => {
        Column() {
          Image(item.img)
            .objectFit(ImageFit.Contain)
            .width(this.cardWidth * item.size)
            .height(569 * item.size);
        }
        .zIndex(item.zIndex)
        .opacity(item.opacity)
        .offset({ x: item.offsetX, y: item.offsetY })
        .gesture(GestureGroup(GestureMode.Exclusive,
          PanGesture(this.panOptionH)
            .onActionStart(() => {
              this.needRefresh = false;
            })
            .onActionEnd((event: GestureEvent) => {
              this.leftToRight = event.offsetX > 0;   // FIX: was a ±100 threshold in onActionStart
              this.handleSlider();
            })
        ));
      });
    }
    .width('100%')
    .margin({ top: 25 });

    if (this.arr.length > 0) {          // FIX: shipped as `&& (this.updateArrLength || !this.updateArrLength)`
      // prompt line and the 立即投递 / 不感兴趣 buttons
    }
  }
}
```

**Three details make this a deck rather than a list.** The gesture is attached
to each card's `Column`, so the card under the finger is the one that reacts and
`GestureMode.Exclusive` stops the neighbours competing for the same pan.
`distance: 100` means the gesture does not even activate until the finger has
travelled 100 vp — a deliberate stiffness, so a vertical scroll or a stray
touch never rotates the deck. And `offset` (not `position`, not `translate`)
is used because the cards are all laid out at the same `Stack` origin and each
one is displaced from it.

The two corrections are small and both structural. Deriving `leftToRight` in
`onActionEnd` from the sign of `offsetX` removes the dead zone: with the pan
threshold and the direction threshold both set to 100, a pan that activates at
exactly the threshold sets neither branch and the deck rotates in whatever
direction the *previous* gesture left behind (`HW-12-0003`). Decorating `arr`
lets `splice` re-render the `ForEach`, which retires
`(this.updateArrLength || !this.updateArrLength)` — an always-true expression
whose only job was to smuggle a `@State` boolean into the condition so
`arr.length` would be re-read (`HW-12-0002`).

**Rotating the slots — same file** (as shipped)

```typescript
handleSlider() {
  this.getUIContext().animateTo({
    duration: 500,
    curve: Curve.EaseInOut,
    iterations: 1,
    playMode: PlayMode.Normal,
    onFinish: () => {
    }
  }, () => {
    if (this.leftToRight && this.arr.length > 1) {
      // ...
    } else if (!this.leftToRight && this.arr.length > 1) {
      if (this.arr.length === 2 && this.arr[this.right].offsetX !== 0) {
        // two-card case: just exchange centre and right
      } else if (this.arr.length > 2) {
        this.arr[this.mid].size = 0.83;
        this.arr[this.mid].offsetX = 0;                  // demoted to the left peek
        this.arr[this.mid].zIndex = 1;
        this.arr[this.right].size = 1;
        this.arr[this.right].offsetX = this.midCardOffsetX;   // promoted to centre
        this.arr[this.right].zIndex = 2;
        this.arr[this.left].size = 0.83;
        this.arr[this.left].offsetX = this.midCardOffsetX;
        this.arr[this.left].zIndex = 1;
        this.arr[this.left].opacity = 0;                 // old left peek fades out under the centre
        let rightNext = this.right === this.arr.length - 1 ? 0 : this.right + 1;
        this.arr[rightNext].size = 0.83;
        this.arr[rightNext].offsetX = this.rightCardOffsetX;
        this.arr[rightNext].zIndex = 1;
        this.arr[rightNext].opacity = 1;                 // the next one fades in on the right
        this.needRefresh = true;
      }
    }
  });
  if (this.needRefresh) {
    this.refresh();
  }
}

refresh() {
  if (this.leftToRight) {
    this.right = this.mid;
    this.mid = this.left;
    this.left = this.left === 0 ? this.arr.length - 1 : this.left - 1;
  } else {
    this.mid = (this.mid + 1) % this.arr.length;
    this.left = (this.left + 1) % this.arr.length;
    this.right = (this.right + 1) % this.arr.length;
  }
}
```

**Four cards move per swipe, not three.** The outgoing peek fades out *at the
centre offset* rather than sliding off the edge, and a fourth card — the one
past the incoming edge — fades in at the far peek position. Without that fourth
assignment the deck would look empty on one side after every rotation. All four
happen inside one `animateTo` closure, so position, scale, depth and opacity
share a single 500 ms `EaseInOut` curve and read as one motion.

**`refresh()` runs after the closure, not inside it.** The closure mutates the
cards; `refresh()` mutates the page's idea of which array index is in which
slot. Rotating the indices inside the animation would make the second half of
the closure address the new slots. `needRefresh` exists because the closure has
branches that do nothing (a one-card deck), and the index rotation must be
skipped in exactly those cases.

**Consuming the centre card — same file** (as shipped)

```typescript
handleDelete() {
  this.getUIContext().animateTo({
    duration: 500, curve: Curve.EaseInOut, iterations: 1, playMode: PlayMode.Normal,
    onFinish: () => {
    }
  }, () => {
    this.deleteCard();
  });

  this.arr.splice(this.mid, 1);
  this.updateArrLength = !this.updateArrLength;   // the re-render hack (HW-12-0002)
  if (this.mid === this.arr.length) {
    this.mid = 0;
    this.right = 1;
  } else if (this.mid === 0) {
    this.left--;
  } else if (this.right === this.arr.length) {
    this.right = 0;
  }
}

deleteCard() {
  this.arr[this.mid].size = 0.1;
  this.arr[this.mid].opacity = 0;
  this.arr[this.mid].offsetX = 137;
  this.arr[this.mid].offsetY = -300;        // shrink and fly up-and-away
  // ... promote right into the centre, fade in the one past it
}
```

This is where the missing observation becomes visible. `deleteCard()` animates
the card to `opacity: 0` at 10% scale, and then `splice` removes it from the
array — but the `ForEach` never re-runs, so the `Column` stays in the tree with
its `PanGesture` still attached. After all seven cards are "deleted" the
`Stack` holds seven invisible, still-interactive cards, and the index
arithmetic below the `splice` is operating on an array that no longer matches
what is on screen. Note also that the mutation and the `splice` race: the
`splice` runs immediately while the fly-away animation is still playing, so
`this.mid` has already moved on.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`.

`deviceTypes` is `phone`, `tablet`, `2in1`. `compatibleSdkVersion` and
`targetSdkVersion` are both `6.0.0(20)`.

The four user-visible strings that live in
`resources/base/element/string.json` are 推荐岗位 (the title), 向右滑动查看更多岗位
(the swipe hint), 立即投递 and 不感兴趣. The three toast strings, by contrast,
are hardcoded literals on `StyleConstants` — a split with no rationale; the
toasts are just as user-visible and just as translatable.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The card height is a hardcoded `569 * item.size` while the width is derived
  from the display, so the cards distort on any device whose aspect ratio
  differs from the one the artwork was cut for. `ImageFit.Contain` hides the
  distortion by letterboxing rather than stretching.
- Geometry is computed once in `aboutToAppear` from
  `display.getDefaultDisplaySync()`, not from the window. On a resized 2in1
  window or a foldable that changes state, the deck keeps the old layout — the
  right peek will sit off-screen or short of the edge.
- `PositionSlidingPage` writes `px2vp(...)` results into its own `@StorageProp`
  fields in `aboutToAppear`. `@StorageProp` is a one-way binding *from*
  `AppStorage`: the local write does not propagate back, and the next
  `avoidAreaChange` overwrites the field with a raw px value, so the page
  padding jumps by roughly a factor of three. Store vp in `AppStorage` instead
  (as `JOBS-04`'s ability does), or convert at the point of use (as `JOBS-02`
  does).
- The deck is seven cards backed by three images repeated; there is no data
  source and no job model at all — the card is a picture.
- `EntryAbility` registers `avoidAreaChange` and never releases it in
  `onWindowStageDestroy`.

## Pitfalls

- **`HW-12-0002`** (B/medium, confirmed): the card array is not state-observed,
  so `this.arr.splice(this.mid, 1)` never re-runs the `ForEach` — the removed
  card's `Column` stays mounted, invisible only because `deleteCard()` set its
  opacity to 0, and it keeps its `PanGesture`. The bottom rows are gated with
  `(this.updateArrLength || !this.updateArrLength)`, a tautology that exists
  only to pull a `@State` boolean into the condition so `arr.length` gets
  re-evaluated. Fix: decorate the array (`@State arr: Card[]`, or `@Local`
  under `@ComponentV2`), delete `updateArrLength`, and write the guards as
  `if (this.arr.length > 0)`.
- **`HW-12-0003`** (B/low, probable): `leftToRight` is set in `onActionStart`
  only when `event.offsetX` exceeds ±100, but `PanGestureOptions.distance` is
  also 100, so a pan that activates at or below the threshold leaves the flag
  at its stale value from the previous gesture — and `onActionEnd` always calls
  `handleSlider()`, so the deck rotates the wrong way. Fix: set `leftToRight`
  from `event.offsetX > 0` in `onActionEnd` (or track it in `onActionUpdate`).

## References

- `huawei_industry_tree/12_jobs/docs/03_position_sliding_window.md` — the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/position_sliding_window-0000002391546225
- `documentation/harmonyos-references/02_application-framework/ts-container-stack.md` — `Stack` and `alignContent`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-stack
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` — `PanGesture`, `PanGestureOptions`, `onActionStart`/`onActionEnd`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` — `animateTo`, `px2vp`, `getPromptAction`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-guides/03_application-framework/arkts-new-observedv2-and-trace.md` — what `@ObservedV2`/`@Trace` do and do not observe
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-observedv2-and-trace
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` — when `ForEach` re-renders
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `JOBS-02` — the same avoid-area boilerplate, consumed correctly
- `JOBS-04` — the list-based counterpart to this deck
