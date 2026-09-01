---
id: EDU-06
title: Stacked word cards - a three-deep card deck with swipe-to-browse and flick-to-dismiss
industry: 04_education
doc: huawei_industry_tree/04_education/docs/06_stackable_word_cards.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/stackable_word_cards-0000002349040357
sample: huawei_industry_tree/04_education/downloads/StackableWordCards.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: [Stack, PanGesture, PanGestureOptions, GestureGroup, GestureMode, animateTo, FinishCallbackType, translate, opacity, zIndex, "@ComponentV2", "@Local", "@ObservedV2", "@Trace", AppStorageV2, "AppStorageV2.connect", PromptAction, showToast, Navigation, NavPathStack, NavDestination, onReady]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-04-0035, HW-04-0036, HW-04-0037, HW-04-0038, HW-04-0039, HW-04-0040, HW-04-0041, HW-04-0042, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card for a **card deck** - a stack where one card is front and centre,
its neighbours peek out behind it, swiping left or right rotates the deck, and
flicking a card up dismisses it. Flashcards are the education case; the same
shape covers a review queue, a swipe-to-triage inbox, or an onboarding carousel.

The construction is worth knowing because it is **not** a `Swiper`. `Swiper`
gives you one item at a time in a viewport; it cannot show three cards at
staggered offsets, scales and opacities, and it has no second axis for dismissal.
Here, every card is a sibling in a `Stack` and its appearance is entirely
data-driven: four numbers per card (`positionX`, `positionY`, `opacityX`,
`sizeRatio`) that the gesture handlers write inside `animateTo`.

This sample is also the industry's only **state management V2** page
(`@ComponentV2`, `@Local`, `@ObservedV2`/`@Trace`, `AppStorageV2`), and it is
instructive both for what V2 buys here and for the three V2 mistakes it makes.

## Feature checklist

- Three cards visible: the middle one full size and opaque, the two neighbours
  offset horizontally, scaled to 83 % and faded to 70 %.
- Swiping horizontally past 100 vp rotates the deck by one, animated over 500 ms.
- The two arrow buttons do the same thing as the swipes.
- Flicking the middle card up past 100 vp animates it away - up, left, shrinking
  to 20 %, fading to 0 - then removes it and shows a toast.
- The deck wraps: swiping past either end continues from the other.
- The remaining cards re-stagger after a deletion.

## Architecture

One `entry` module, two pages. There is no view model; the deck is a
`WordCard[]` and each card carries its own animatable state.

```
entry/src/main/ets
├── entryability/EntryAbility.ets        RectHeight class + full screen + avoid areas -> AppStorageV2
├── entrybackupability/EntryBackupAbility.ets
└── pages
    ├── EntryPage.ets                    @Entry - Navigation + Tabs; pushes the word-card page
    └── MemorizeWordsPage.ets            the deck, and the WordCard model class
```

The document's tree names the `entrybackupability` directory `entryability`
(`HW-04-0042`).

**The model is where the animation lives.** `WordCard` is `@ObservedV2` and its
five layout properties are `@Trace`:

```typescript
@ObservedV2
class WordCard {
  img: Resource;
  public id: number;
  @Trace public offsetX: number;
  @Trace public opacityX: number;
  @Trace public positionX: number;
  @Trace public positionY: number = 0;
  @Trace public sizeRatio: number;
}
```

That is the design decision worth copying: the gesture handlers never touch the
view tree, they assign numbers to the card objects, and `@Trace` re-renders the
bound attributes. Wrapping the assignments in `animateTo` turns every one of
those re-renders into a transition, which is why one 500 ms `animateTo` can move
three cards, fade a fourth and scale all of them with no per-card animation code.

**Three cursors track the deck.** `left`, `middle` and `right` are indices into
`arr`, all advanced modulo `arr.length`, which is what makes the deck circular.
`refreshCards()` walks from `left` for up to three cards and recomputes the four
numbers from `distance`, the card's position in the visible window:

| distance | offset | opacity | scale |
| --- | --- | --- | --- |
| 0 (left peek) | 0 | 0.7 | 0.83 |
| 1 (front) | 50 | 1.0 | 1.00 |
| 2 (right peek) | 100 | 0.7 | 0.83 |

`1 - 0.3 * |distance - 1|` and `1 - 0.17 * |distance - 1|` are the two curves;
they peak at `distance === 1`, which is why the middle card is the front one.

## Implementation steps

1. **Model each card as an `@ObservedV2` class with `@Trace` layout
   properties** - offset, opacity, scale, and a vertical offset for the
   dismissal.
2. **Declare the deck as `@Local arr: WordCard[]`.** The sample leaves it
   undecorated, so the `splice` that deletes a card cannot re-render the list
   (`HW-04-0035`). In a `@ComponentV2` component, `Repeat` is the V2 rendering
   API and is the better choice for a mutable list.
3. **Render all cards as siblings in a `Stack`**, each binding
   `.translate({ x: positionX, y: positionY })`, `.opacity(opacityX)`,
   `.height(576 * sizeRatio)` and `.zIndex(opacityX)`.
4. **Build two `PanGestureOptions`, one per axis,** and combine them with
   `GestureGroup(GestureMode.Exclusive, vertical, horizontal)` so a drag commits
   to one axis and the other gesture stands down.
5. **Guard each handler with its own `isScrolling` flag**, set inside the
   handler and cleared in `onActionEnd`. `onActionUpdate` fires once per frame;
   without the flag one drag triggers dozens of deck rotations.
6. **Test the threshold before calling `animateTo`, not inside it.** The
   vertical handler does; the horizontal one does not (`HW-04-0040`).
7. **Do the deletion in `onFinish`, the visual change in the animation
   closure.** The closure fades and flings the card; `onFinish` splices it out
   of the array once the animation has actually finished. Use
   `finishCallbackType: FinishCallbackType.LOGICALLY` so `onFinish` fires when
   the animation is logically complete rather than when the last frame lands.
8. **Handle the empty deck.** After the splice, reset the cursors and return -
   the sample leaves `left` at `-1` and computes `% 0` (`HW-04-0038`).
9. **Use one constant for the card offset.** The constructor uses 52 and the
   refresh loops use 50 (`HW-04-0039`).

## Verified snippets

All snippets are from `StackableWordCards.zip`. Corrected forms are marked.

**The deck and the cursor refresh — `entry/src/main/ets/pages/MemorizeWordsPage.ets`** (corrected, see `HW-04-0035`, `HW-04-0039`)

```typescript
const CARD_OFFSET = 50;                  // FIX: constructor uses 52, the loops use 50

@Entry
@ComponentV2
export struct MemorizeWordsPage {
  @Local arr: WordCard[] = [ /* eight cards */ ];    // FIX: sample leaves this undecorated
  private panOptionH: PanGestureOptions = new PanGestureOptions({ direction: PanDirection.Horizontal });
  private panOptionV: PanGestureOptions = new PanGestureOptions({ direction: PanDirection.Vertical });
  isScrollingH: boolean = false;
  isScrollingV: boolean = false;
  left: number = 0;      // index of the left peek card
  middle: number = 1;    // index of the front card
  right: number = 2;     // index of the right peek card

  refreshCards() {
    let count = this.arr.length >= 3 ? 3 : this.arr.length;
    for (let i = this.left; count > 0; i++) {
      i = (i + this.arr.length) % this.arr.length;                 // wrap
      let distance = ((i - this.left + this.arr.length) % this.arr.length);
      this.arr[i].offsetX = CARD_OFFSET * distance;
      this.arr[i].positionX = this.arr[i].offsetX;
      this.arr[i].opacityX = 1 - 0.3 * Math.abs(distance - 1);     // peaks at distance 1
      this.arr[i].sizeRatio = 1 - 0.17 * Math.abs(distance - 1);
      count--;
    }
  }
}
```

**`count` drives the loop, not `i`.** `i` is advanced and immediately wrapped
modulo the length, so it can never be used as the termination test - the loop
counts down the number of cards to touch instead. That is the idiom to copy for
any circular window over an array.

**Only three cards are ever refreshed.** Everything outside the window keeps
whatever state it had, which is `opacityX: 0` from the constructor's `else`
branch. Cards are not hidden by a conditional; they are hidden by being
transparent, which is what lets them animate in.

**The card, its two gestures, and the deletion — same file** (corrected, see `HW-04-0038`)

```typescript
Stack({ alignContent: Alignment.Center }) {
  ForEach(this.arr, (item: WordCard) => {
    Column() {
      Image(item.img)
        .height(576 * item.sizeRatio)
        .objectFit(ImageFit.Contain)
        .borderRadius(16)
    }
    .shadow({ radius: 36, color: '#28000000', offsetY: 8 })
    .zIndex(item.opacityX)                                  // opacity doubles as depth
    .translate({ x: item.positionX, y: item.positionY })
    .opacity(item.opacityX)
    .gesture(GestureGroup(GestureMode.Exclusive,
      PanGesture(this.panOptionV)                           // dismiss
        .onActionUpdate((event: GestureEvent) => {
          if (event.offsetY < -100 && !this.isScrollingV && this.arr.length !== 0) {
            this.middle = this.arr.length >= 2 ? (this.left + 1 + this.arr.length) % this.arr.length : 0;
            this.getUIContext().animateTo({
              duration: 500,
              curve: Curve.EaseInOut,
              finishCallbackType: FinishCallbackType.LOGICALLY,
              onFinish: () => {
                this.arr.splice(this.middle, 1);
                if (this.arr.length === 0) {                // FIX: sample leaves left at -1
                  this.left = this.middle = this.right = 0;
                  return;
                }
                this.left = this.left === this.arr.length ? this.arr.length - 1 : this.left;
                this.right = (this.right + 1 + this.arr.length) % this.arr.length;
                this.promptAction.showToast({ message: $r('app.string.good_memory'), offset: { dx: 0, dy: 28 } });
              }
            }, () => {
              this.isScrollingV = true;
              // the card being dismissed: up, left, transparent, small
              this.arr[this.middle].positionX = -50;
              this.arr[this.middle].positionY = -300;
              this.arr[this.middle].opacityX = 0;
              this.arr[this.middle].sizeRatio = 0.2;
              // ... and the survivors re-stagger in the same animation
            });
          }
        })
        .onActionEnd(() => { this.isScrollingV = false; }),
      PanGesture(this.panOptionH)                           // browse
        .onActionUpdate((event: GestureEvent) => {
          if (Math.abs(event.offsetX) <= 100 || this.isScrollingH) {
            return;                                        // FIX: sample tests inside animateTo
          }
          this.getUIContext().animateTo({ duration: 500, curve: Curve.EaseInOut }, () => {
            this.isScrollingH = true;
            if (event.offsetX > 100) {
              this.arr[this.right].opacityX = 0;            // the card leaving the window
              this.left = (this.left - 1 + this.arr.length) % this.arr.length;
              this.right = (this.right - 1 + this.arr.length) % this.arr.length;
            } else {
              this.arr[this.left].opacityX = 0;
              this.left = (this.left + 1 + this.arr.length) % this.arr.length;
              this.right = (this.right + 1 + this.arr.length) % this.arr.length;
            }
            this.refreshCards();
          });
        })
        .onActionEnd(() => { this.isScrollingH = false; })
    ))
  })
}
```

**`GestureMode.Exclusive` is what makes two `PanGesture`s on one node work.**
The two options objects restrict each gesture to one axis, and Exclusive mode
means the first one to be recognised wins for the duration of the drag - a
diagonal flick becomes either a browse or a dismiss, never both.

**The `onFinish` / closure split is the pattern to copy.** Everything inside the
`animateTo` closure is *interpolated*: the card flies up and fades because those
four assignments are wrapped. Everything in `onFinish` is *committed*: the
splice, the cursor update, the toast. Put the splice in the closure and the card
vanishes instantly with no animation; put the fade in `onFinish` and it snaps.

**Order matters in the browse handler**: fade out the card leaving the window
*first*, then move the cursors, then `refreshCards()`. Reverse it and the card
that just entered the window is the one you blank.

**Avoid areas through AppStorageV2 — `entry/src/main/ets/entryability/EntryAbility.ets`** (corrected, see `HW-04-0036`)

```typescript
@ObservedV2
export class RectHeight {
  @Trace height: number = 0;
  constructor(height: number) { this.height = height; }
}

onWindowStageCreate(windowStage: window.WindowStage): void {
  const windowClass: window.Window = windowStage.getMainWindowSync();
  windowClass.setWindowLayoutFullScreen(true).catch(/* ... */);

  const bottom = AppStorageV2.connect(RectHeight, 'bottomRectHeight', () => new RectHeight(0))!;
  const top = AppStorageV2.connect(RectHeight, 'topRectHeight', () => new RectHeight(0))!;
  bottom.height = windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR).bottomRect.height;
  top.height = windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM).topRect.height;

  windowClass.on('avoidAreaChange', (data) => {
    // FIX: the sample calls connect(..., () => new RectHeight(h)) here, which returns
    // the STORED object and throws the new one away - the heights never change.
    if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
      top.height = data.area.topRect.height;
    } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
      bottom.height = data.area.bottomRect.height;
    }
  });
}
```

**`connect` is get-or-create, and the third argument is only the *create*
half.** Once `onWindowStageCreate` has stored a `RectHeight` under a key, every
later `connect` on that key returns the stored instance and ignores the factory.
Updating means assigning to the `@Trace` property of the object you get back -
which is also the only thing that propagates, since `@Trace` tracks the property
on that object.

**And on the page side, hold the object, not the number** (`HW-04-0037`):

```typescript
// FIX: the sample writes
//   @Local topRectHeight: number | undefined = AppStorageV2.connect(RectHeight, 'topRectHeight')?.height;
// which copies a primitive out and severs the @Trace link.
@Local topRect: RectHeight = AppStorageV2.connect(RectHeight, 'topRectHeight', () => new RectHeight(0))!;
// ...
.padding({ top: this.uiContext.px2vp(this.topRect.height) })
```

Compare with `EDU-01`/`EDU-03`/`EDU-04`, which use V1 `AppStorage` with
`@StorageProp` and get the same effect in one line. V2's `AppStorageV2` only
stores class instances - primitives are explicitly unsupported - which is why
the plain number had to be wrapped in `RectHeight` at all.

## Permissions & config

None. `entry/src/main/module.json5` declares no `requestPermissions` block.

Routing config, corrected (see `HW-04-0041`):

```json5
// entry/src/main/resources/base/profile/main_pages.json
{ "src": ["pages/EntryPage"] }            // FIX: sample also lists pages/MemorizeWordsPage,
                                          // whose root is a NavDestination

// entry/src/main/resources/base/profile/route_map.json
{
  "routerMap": [
    { "name": "EntryPage",          "pageSourceFile": "src/main/ets/pages/EntryPage.ets",
      "buildFunction": "entryPageBuilder" },
    { "name": "MemorizeWordsPage",  "pageSourceFile": "src/main/ets/pages/MemorizeWordsPage.ets",
      "buildFunction": "memorizeWordsPageBuilder" }   // FIX: sample names this route "PageOne"
  ]
}
```

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The visible window is hard-coded to three cards.** `refreshCards` caps at 3,
  the post-delete loop has bespoke `count` arithmetic for lengths 2, 3 and >3,
  and the opacity/scale curves are written around `distance - 1`. A four-card
  deck is not a parameter change.
- The card height 576 and the offsets 50/52 are inlined; the deck is sized for a
  phone in portrait.
- The eight cards are three images repeated, held in a literal. There is no data
  source and no persistence - deletions are lost on relaunch.
- 再记一次 (memorize again) and 忘记了 (forgot) both raise a
  功能待在实际使用场景中实现 toast; only the deck itself is implemented.
- `.zIndex(item.opacityX)` derives stacking order from opacity. It works because
  the front card is the only one at 1.0, but the two peek cards tie at 0.7 and
  their relative order is then whatever the array order gives.

## Pitfalls

- **`HW-04-0035` — `arr` is undecorated in a `@ComponentV2` struct,** so the
  `splice` is invisible to `ForEach`. Per-card animation still works because
  `WordCard` is `@ObservedV2`/`@Trace`; the array mutation is the part that is
  lost. Decorate it `@Local`, or move to `Repeat`.
- **`HW-04-0036` — `AppStorageV2.connect(..., () => newObject)` in the
  `avoidAreaChange` handler discards the new object.** `connect` returns the
  stored instance once the key exists; the factory is the create path only. The
  avoid areas are frozen at launch values.
- **`HW-04-0037` — the pages copy `.height` out into a `@Local number`,**
  severing the `@Trace` link. Hold the `RectHeight` instance instead.
- **`HW-04-0038` — deleting the last card leaves `left` at `-1` and computes
  `% 0`,** which is `NaN`. The two arrow buttons guard on `arr.length > 0`; the
  two pan handlers do not.
- **`HW-04-0039` — the constructor staggers at 52 vp and both refresh loops at
  50 vp,** so the deck jumps on the first interaction.
- **`HW-04-0040` — the horizontal handler calls `animateTo` on every frame,**
  with the threshold test inside the closure. The vertical handler tests first;
  copy that one.
- **`HW-04-0041` — the route map names the page `PageOne`** while the code
  pushes `MemorizeWordsPage`, and `main_pages.json` lists a NavDestination-rooted
  component as a standalone page.
- **`HW-04-0042` — the project tree lists `entryability` twice.** The same
  template error appears in `EDU-04`.
- **Do not reach for `Swiper`.** It cannot show three cards at once, and it has
  no second axis for the dismissal.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-stack.md` - `Stack` and `alignContent`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-stack
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - `PanGesture`, `PanGestureOptions`, the cumulative `offsetX`/`offsetY`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/ts-combined-gestures.md` - `GestureGroup` and `GestureMode.Exclusive`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-combined-gestures
- `documentation/harmonyos-guides/03_application-framework/arkts-new-local.md` - `@Local` and what an undecorated V2 member does not observe
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-local
- `documentation/harmonyos-guides/03_application-framework/arkts-new-observedv2-and-trace.md` - `@ObservedV2`, `@Trace`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-observedv2-and-trace
- `documentation/harmonyos-guides/03_application-framework/arkts-new-appstoragev2.md` - `connect` as get-or-create, and the no-primitives constraint
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-appstoragev2
- `documentation/harmonyos-guides/03_application-framework/arkts-new-rendering-control-repeat.md` - `Repeat`, the V2 replacement for `ForEach`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-rendering-control-repeat
- `EDU-01`, `EDU-03`, `EDU-04`, `EDU-05` - the V1 `AppStorage` avoid-area boilerplate this page reimplements in V2
