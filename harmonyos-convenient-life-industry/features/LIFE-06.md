---
id: LIFE-06
title: Drag-to-reorder to-do list - LongPress plus Pan in a sequential GestureGroup driving an AttributeModifier per row
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/06_sticky_note.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/sticky_note-0000002275644713
sample: huawei_industry_tree/02_convenient_life/downloads/StickyNote.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.ArkTS", "@kit.PerformanceAnalysisKit", "@ohos.curves"]
apis: [GestureGroup, GestureMode, LongPressGesture, PanGesture, onActionStart, onActionUpdate, onActionEnd, onCancel, GestureEvent, fingerList, AttributeModifier, ListItemAttribute, attributeModifier, "UIContext.animateTo", "curves.initCurve", "curves.interpolatingSpring", ICurve, List, ListItem, ListScroller, "scroller.currentOffset", "scroller.scrollTo", "scroller.isAtEnd", swipeAction, backToTop, onAreaChange, TransitionEffect, zIndex, Checkbox, "$$", "@Observed", "@ObjectLink", "@Provide", "@StorageLink", "@StorageProp", "@CustomDialog", CustomDialogController, "util.format", "resourceManager.getStringByNameSync"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-02-0034, HW-02-0035, HW-02-0036, HW-02-0037, HW-02-0038, HW-02-0039, HW-02-0040, HW-02-0041, HW-02-0042]
status: verified-with-fixes
---

## When to use

Load this card when a list has to be **reordered by dragging a row**, and you
want the drag, the shrink-away of the displaced neighbour, and the spring-back
on drop all animated - without `onMove`'s built-in drag sorting, because you
also need a long press to arm it and a per-row style you control.

The construction is: a **sequential** `GestureGroup` (long press, then pan) on
each `ListItem`, one `AttributeModifier` object per row holding that row's
translate/scale/shadow/opacity, and a controller class that mutates those
modifiers inside `animateTo` closures. The list data and the modifier array are
spliced together so they stay index-aligned.

The scenario is a to-do sticky-note list, but this is the general recipe for
hand-rolled drag sorting: playlists, task boards, form-field ordering, anything
where ArkUI's `onMove` is too coarse.

If you only need reordering with the default look, `ForEach` + `onMove` is far
less code. Take this card when you need the animation and the gesture arming.

## Feature checklist

- A list of notes, each with a checkbox, the note text, and a rounded card
  background.
- Long-pressing a note lifts it: shadow on, scale to 1.04, opacity 0.5,
  `zIndex` raised above its siblings.
- Once lifted, dragging moves the note with the finger; the neighbour it is
  about to displace shrinks along an interpolated `Curve.Sharp` ramp.
- Crossing half a row swaps the note with that neighbour, animated over 300 ms.
- Dragging near the top or bottom fifth of the list auto-scrolls, faster in the
  outer tenth.
- Releasing springs the note back to rest and restores its neighbours.
- Swiping a note left reveals a delete button; deleting slides the row out and
  fades it before removing it.
- A header shows the note count and a + button opens a text dialog to add one.

## Architecture

One `entry` module. The interesting split is that **all the animation state
lives outside the components**, in a plain class and an array of modifiers.

```
entry/src/main/ets
├── Constant/Constants.ets          sizes, spacings, opacity  (doc says "constants" - HW-02-0041)
├── entryability/EntryAbility.ets   full screen, avoid areas -> AppStorage
├── model
│   ├── ListInfo.ets                @Observed StickyNoteInfo { text, visible, isCompleted, id }
│   ├── MockData.ets                three seed notes
│   ├── AttributeModifier.ets       ListItemModifier implements AttributeModifier<ListItemAttribute>
│   └── ListExchangeCtrl.ets        THE CARD: onLongPress / onMove / onDrop / changeItem / add / delete
├── pages/StickyNotePage.ets        @Entry - the List, the GestureGroup, the add dialog
└── view/StickyNoteView.ets         one row + the swipeAction delete builder
```

**`ListItemModifier` is the bridge between a plain class and the render tree.**
It implements `AttributeModifier<ListItemAttribute>`; `applyNormalAttribute` is
called by the framework and translates six plain fields into `shadow`,
`zIndex`, `opacity`, `translate` and `scale`. The `ListItem` binds it with
`.attributeModifier(...)`. Mutating a field inside an `animateTo` closure is
therefore enough to animate that row - the component never re-renders.

**Two arrays, spliced in lockstep.**

```
deductionData: Array<T>              === the page's @Provide appInfoList (same object)
modifier:      Array<ListItemModifier>
```

`getModifier(item)` looks the row up by `indexOf`, so the two arrays must stay
index-aligned; every mutation (`changeItem`, `addItem`, `deleteItem`) splices
both. That is the invariant the whole class is built around.

The gesture flow for one drag:

```
LongPressGesture.onAction   -> currentData = item (raises zIndex)
                               isLongPress = true  (@StorageLink)
                               ctrl.onLongPress(item)      shadow + scale 1.04
                               originListOffsetY = scroller.currentOffset().yOffset
PanGesture.onActionUpdate   -> ctrl.onMove(item, event.offsetY, scroller.currentOffset().yOffset)
                               offsetY = pan - dragRefOffset + scroll - originScroll
                               neighbour scale interpolated from |offsetY| / pitch
                               |offsetY| > pitch/2  ->  animateTo { swap, correct offsetY }
                               finger in the outer 20% -> scroller.scrollTo(...)
PanGesture.onActionEnd      -> ctrl.onDrop(item)   two springs: neighbours, then self
GestureGroup.onCancel       -> ctrl.onDrop(item)   same cleanup
```

`GestureMode.Sequence` is what makes the long press mandatory: the pan is not
recognised until the long press has fired, so an ordinary scroll drag still
scrolls the list.

**Why `dragRefOffset` exists.** `PanGesture.offsetY` is cumulative from the
start of the gesture and does not reset when a swap happens. After each swap the
item has moved one row in the data, so the code adds one pitch to
`dragRefOffset` and subtracts one pitch from `offsetY` - keeping the visual
position under the finger while the data index advances. That correction uses
the wrong pitch in the sample (`HW-02-0034`).

## Implementation steps

1. **Derive one pitch constant** from the row height and the list `space`:
   `ITEM_PITCH = STICKY_NOTE_VIEW_HEIGHT + STICKY_NOTE_LIST_SPACE`. Everything
   in the drag maths is in units of this. The sample hard-codes 50 against a
   real pitch of 60, in two files (`HW-02-0034`).
2. **Write the modifier class.** Six plain fields, one
   `applyNormalAttribute(instance: ListItemAttribute)` that maps them onto
   `shadow`/`zIndex`/`opacity`/`translate`/`scale`. No decorators - the
   framework re-applies it when the object changes.
3. **Build the controller with both arrays** in its constructor: keep the data
   array by reference and allocate one modifier per row.
4. **Give the controller the `UIContext` explicitly.** The sample parks it in
   `AppStorage` under a global key (`HW-02-0040`); a constructor parameter is
   both safer and shorter.
5. **`onLongPress`:** reset `dragRefOffset`, then `animateTo` shadow on and
   scale 1.04. Record the list's current scroll offset in the page so `onMove`
   can subtract it.
6. **`onMove`:** compute `offsetY = pan - dragRefOffset + scroll - originScroll`.
   Interpolate the neighbour's shrink from `|offsetY| / ITEM_PITCH` through
   `curves.initCurve(Curve.Sharp)`.
7. **Check the swap bounds *before* touching the offsets** (`HW-02-0035`). The
   sample adjusts `offsetY` and `dragRefOffset` unconditionally and then guards
   only the splice, so both ends of the list jump one row without reordering.
8. **`changeItem` splices both arrays** with the same indices, in the same
   order. Never splice one without the other.
9. **`onDrop`:** two separate `animateTo` calls with different springs - a stiff
   one (`interpolatingSpring(0, 1, 400, 38)`) to restore the neighbours, a
   bouncier one (`interpolatingSpring(14, 1, 170, 17)`) to drop the item itself.
10. **Key the `ForEach` on the item's `id`,** not on `JSON.stringify(item)`
    (`HW-02-0036`) - the note's `isCompleted` is mutable and would change the
    key on every checkbox tick.
11. **Auto-scroll from the finger position,** not from the item offset:
    `event.fingerList[0].globalY` against the list's `Area` from
    `onAreaChange`.

## Verified snippets

All snippets are from `StickyNote.zip`. Corrected forms are marked.

**The per-row modifier - `StickyNote.zip#entry/src/main/ets/model/AttributeModifier.ets`** (as shipped)

```typescript
export class ListItemModifier implements AttributeModifier<ListItemAttribute> {
  public hasShadow: boolean = false;
  public isDeleted: boolean = false;
  public scale: number = 1;
  public offsetY: number = 0;
  public offsetX: number = 0;
  public opacity: number = 1;

  applyNormalAttribute(instance: ListItemAttribute): void {
    if (this.hasShadow) {
      instance.shadow({
        radius: $r('app.integer.list_exchange_shadow_radius'),
        color: $r('app.color.list_exchange_box_shadow')
      });
      instance.zIndex(1);
      instance.opacity(0.5);
    } else {
      instance.opacity(this.opacity);
    }
    instance.translate({ x: this.offsetX, y: this.offsetY });
    instance.scale({ x: this.scale, y: this.scale });
  }
}
```

**This is the whole reason the drag animates without re-rendering.** The
component tree is built once; the controller mutates plain fields on these
objects inside `animateTo`, and the framework re-runs
`applyNormalAttribute` for the affected row. No `@State`, no `@ObjectLink`, no
diffing of the list.

`translate` rather than `offset` matters: `translate` is a render-time transform
that does not affect layout, so moving a row does not reflow its siblings.

Note the branch: `opacity` is 0.5 while lifted and `this.opacity` otherwise -
which is what lets the delete animation fade the row to 0 through the same
field.

The class also carries a `getInstance()` singleton (not shown) that nothing
uses; ignore it - a shared modifier would style every row identically.

**The controller's constructor - `StickyNote.zip#entry/src/main/ets/model/ListExchangeCtrl.ets:51`** (as shipped)

```typescript
@Observed
export class ListExchangeCtrl<T> {
  private deductionData: Array<T>;
  private modifier: Array<ListItemModifier>;
  private dragRefOffset: number = 0;
  offsetY: number = 0;
  state: OperationStatus = OperationStatus.IDLE;
  originListOffsetY: number = 0;

  constructor(deductionData: Array<T>) {
    this.deductionData = deductionData;             // by reference - same array as @Provide appInfoList
    this.modifier = new Array<ListItemModifier>();
    deductionData.forEach(() => {
      this.modifier.push(new ListItemModifier());
    });
  }

  getModifier(item: T): ListItemModifier {
    let index: number = this.deductionData.indexOf(item);
    return this.modifier[index];
  }
}
```

**Holding the data array by reference is deliberate.** The page passes its
`@Provide appInfoList`, so when `changeItem` splices `deductionData` it is
splicing the very array the `ForEach` renders - the reorder shows up with no
extra plumbing.

**The move handler - same file, line 91** (corrected, see `HW-02-0034` and `HW-02-0035`)

```typescript
onMove(item: T, offsetY: number, scrollerOffset: number): void {
  const index: number = this.deductionData.indexOf(item);
  this.offsetY = offsetY - this.dragRefOffset + scrollerOffset - this.originListOffsetY;
  this.modifier[index].offsetY = this.offsetY;
  const direction: number = this.offsetY > 0 ? 1 : -1;

  // the neighbour shrinks along a Sharp ramp as the item approaches it
  const curveValue: ICurve = curves.initCurve(Curve.Sharp);
  const value: number = curveValue.interpolate(Math.abs(this.offsetY) / ITEM_PITCH);
  const shrinkScale: number = 1 - value / 10;
  if (index < this.modifier.length - 1) {
    this.modifier[index + 1].scale = direction > 0 ? shrinkScale : 1;
  }
  if (index > 0) {
    this.modifier[index - 1].scale = direction > 0 ? 1 : shrinkScale;
  }

  if (Math.abs(this.offsetY) > ITEM_PITCH / 2) {          // FIX: pitch is 60, sample uses 50
    const target = index + direction;
    if (target < 0 || target >= this.modifier.length) {   // FIX: bounds first, and both ends
      return;
    }
    this.uiContext.animateTo({ curve: Curve.Friction, duration: ANIMATE_DURATION }, () => {
      this.offsetY -= direction * ITEM_PITCH;             // keep the row under the finger
      this.dragRefOffset += direction * ITEM_PITCH;       // ...while the data index advances
      this.modifier[index].offsetY = this.offsetY;
      this.changeItem(index, target);
    });
  }
}
```

**The four terms of `offsetY` are each necessary.** `offsetY` is cumulative pan
distance; `dragRefOffset` is how much of it has already been consumed by
completed swaps; `scrollerOffset - originListOffsetY` is how far the list has
auto-scrolled since the drag began. Drop the scroll terms and the row lags
behind the finger during auto-scroll.

**`curveValue.interpolate(t)` is a plain function evaluation, not an animation.**
`curves.initCurve(Curve.Sharp)` returns an `ICurve` whose `interpolate` maps
0..1 to 0..1 along that easing; here it converts "how far through the row have I
dragged" into "how much should the neighbour shrink". `/ 10` caps the shrink at
10 %.

The shipped guard is `if (target !== -1 && target <= this.modifier.length)`,
placed **after** the three offset assignments. Both halves are wrong: `-1` is
tested but `-2` and below are not (unreachable here, but the intent is a range
check), `length` is admitted when the last valid index is `length - 1`, and the
offsets have already been shifted by the time the guard rejects anything.

**Splicing both arrays - same file, line 159** (as shipped)

```typescript
changeItem(index: number, newIndex: number): void {
  let tmp: Array<T> = this.deductionData.splice(index, 1);
  this.deductionData.splice(newIndex, 0, tmp[0]);
  let tmp2: Array<ListItemModifier> = this.modifier.splice(index, 1);
  this.modifier.splice(newIndex, 0, tmp2[0]);
}
```

**The modifier travels with its row.** Because the dragged row's modifier
carries the live `offsetY` and `hasShadow`, moving it to the new index in the
same operation is what keeps the lifted appearance on the item the user is
holding rather than on whatever now sits at the old index.

**The drop - same file, line 124** (as shipped)

```typescript
onDrop(item: T): void {
  const index: number = this.deductionData.indexOf(item);
  this.dragRefOffset = 0;
  this.offsetY = 0;
  AppStorage.setOrCreate('isLongPress', false);

  // neighbours: stiff, no overshoot   (velocity 0, mass 1, stiffness 400, damping 38)
  getUIContext()?.animateTo({ curve: curves.interpolatingSpring(0, 1, 400, 38) }, () => {
    this.state = OperationStatus.DROPPING;
    if (index < this.modifier.length - 1) { this.modifier[index + 1].scale = 1; }
    if (index > 0) { this.modifier[index - 1].scale = 1; }
  });

  // the item itself: launched, bouncy (velocity 14, mass 1, stiffness 170, damping 17)
  getUIContext()?.animateTo({ curve: curves.interpolatingSpring(14, 1, 170, 17) }, () => {
    this.state = OperationStatus.IDLE;
    this.modifier[index].hasShadow = false;
    this.modifier[index].scale = 1;
    this.modifier[index].offsetY = 0;
  });
}
```

**Two springs, deliberately different.** `interpolatingSpring(velocity, mass,
stiffness, damping)` ignores any `duration`; the physics decides. The
neighbours get high stiffness and high damping so they settle flat and quietly;
the dropped item gets an initial velocity of 14 and much lower damping so it
lands with a visible bounce. Running both in one `animateTo` would force one
feel on both.

**The gesture wiring - `StickyNote.zip#entry/src/main/ets/pages/StickyNotePage.ets:102`** (as shipped)

```typescript
List({ scroller: this.listScroller, space: Constants.STICKY_NOTE_LIST_SPACE }) {
  ForEach(this.appInfoList, (item: StickyNoteInfo) => {
    ListItem() {
      StickyNoteView({ listItemInfo: item });
    }
    .zIndex(this.currentData === item ? 2 : 1)
    .swipeAction({ end: deleteBuilder(item, this.listExchangeCtrl) })
    .transition(TransitionEffect.OPACITY)
    .attributeModifier(this.listExchangeCtrl.getModifier(item))
    .gesture(
      // 顺序识别: the pan is not recognised until the long press has fired
      GestureGroup(GestureMode.Sequence,
        LongPressGesture()
          .onAction((event: GestureEvent) => {
            this.currentData = item;
            this.isLongPress = true;
            this.listExchangeCtrl.onLongPress(item);
            this.listExchangeCtrl.originListOffsetY = this.listScroller.currentOffset().yOffset;
          }),
        PanGesture()
          .onActionStart(() => {
            this.listExchangeCtrl.originListOffsetY = this.listScroller.currentOffset().yOffset;
          })
          .onActionUpdate((event: GestureEvent) => {
            this.listExchangeCtrl.onMove(item, event.offsetY, this.listScroller.currentOffset().yOffset);
            // ... auto-scroll, below ...
          })
          .onActionEnd((event: GestureEvent) => {
            this.listExchangeCtrl.onDrop(item);
            this.listExchangeCtrl.originListOffsetY = this.listScroller.currentOffset().yOffset;
          })
      ).onCancel(() => {
        if (!this.isLongPress) { return; }
        this.listExchangeCtrl.onDrop(item);
      })
    );
  }, (item: StickyNoteInfo) => item.id.toString());       // FIX: sample uses JSON.stringify(item)
}
```

**`GestureMode.Sequence` is the whole arming mechanism.** In sequential mode the
group recognises its children in declaration order, so a plain drag never
reaches the `PanGesture` and the list scrolls normally. Swap to
`GestureMode.Parallel` and every scroll starts a reorder.

**`onCancel` on the group, not on the pan.** A sequential group can be cancelled
between its children - for example when the finger leaves the item - and without
this handler the row would stay lifted forever. The `isLongPress` guard stops it
firing for gestures that never armed.

`.zIndex(this.currentData === item ? 2 : 1)` is set on the `ListItem` in the
page while the modifier sets `zIndex(1)` when shadowed - the page's value wins
during the drag, which is what lifts the dragged row above its neighbours.

**Auto-scroll from the finger - same file, line 133** (as shipped)

```typescript
.onActionUpdate((event: GestureEvent) => {
  this.listExchangeCtrl.onMove(item, event.offsetY, this.listScroller.currentOffset().yOffset);
  let curListOffset = this.listScroller.currentOffset();
  let fingerInfo = event.fingerList[0];
  let clickPercentY =
    (fingerInfo.globalY - Number(this.listArea.globalPosition.y)) / Number(this.listArea.height);
  if (clickPercentY > 0.8 && !this.listScroller.isAtEnd()) {
    let scrollVelocity = clickPercentY > 0.9 ? 4 : 2;
    if (this.listMaxScrollOffsetY - curListOffset.yOffset > scrollVelocity + 5) {
      this.listScroller.scrollTo({ xOffset: 0, yOffset: curListOffset.yOffset += scrollVelocity });
    }
  } else if (clickPercentY < 0.2 && curListOffset.yOffset >= 0) {
    let scrollVelocity = clickPercentY < 0.1 ? 4 : 2;
    if (curListOffset.yOffset > scrollVelocity + 5) {
      this.listScroller.scrollTo({ xOffset: 0, yOffset: curListOffset.yOffset -= scrollVelocity });
    }
  }
})
```

**The trigger is where the finger is, not where the row is.**
`event.fingerList[0].globalY` is a screen coordinate; subtracting the list's
`globalPosition.y` (captured by `onAreaChange`) and dividing by its height gives
a 0..1 position inside the list. Two thresholds give two speeds - 2 vp per frame
in the outer fifth, 4 vp in the outer tenth.

`isAtEnd()` and the `> scrollVelocity + 5` tests are the clamps that stop the
list scrolling past its ends; `listMaxScrollOffsetY` is a rough content-height
estimate computed in `onAreaChange`, and it inherits the wrong pitch
(`HW-02-0034`).

**The row - `StickyNote.zip#entry/src/main/ets/view/StickyNoteView.ets`** (as shipped)

```typescript
@Component
export struct StickyNoteView {
  @ObjectLink listItemInfo: StickyNoteInfo;

  build() {
    Row() {
      Checkbox()
        .select($$this.listItemInfo.isCompleted)
        .selectedColor($r('app.color.select_color'))
        .unselectedColor($r('app.color.unselect_color'));
      Text(this.listItemInfo.text)
        .margin({ left: Constants.STICKY_NOTE_VIEW_TEXT_MARGIN_LEFT });
      Blank();
    }
    .width(Constants.NINETY_PERCENT)
    .height(Constants.STICKY_NOTE_VIEW_HEIGHT)          // 48 - with space 12 the pitch is 60
    .borderRadius(Constants.STICKY_NOTE_VIEW_BORDER_RADIUS);
  }
}

@Builder
export function deleteBuilder(item: StickyNoteInfo, listExchangeCtrl: ListExchangeCtrl<StickyNoteInfo>) {
  Image($r('app.media.list_exchange_icon_delete'))
    .interpolation(ImageInterpolation.High)
    .onClick(() => {
      listExchangeCtrl.deleteItem(item);
    });
}
```

**`@ObjectLink` on an `@Observed` note is what makes the checkbox tick stick**
without the page rebuilding the list - and `$$` gives the two-way write straight
into `isCompleted`. That mutation is exactly what breaks a `JSON.stringify` key
(`HW-02-0036`).

The delete affordance is a **standalone exported `@Builder` function**, not a
method, so it can be handed to `.swipeAction({ end: ... })` with the item and
the controller captured as parameters. That is the idiom for a per-row swipe
action whose handler lives outside the row component.

**Chained delete animation - `StickyNote.zip#entry/src/main/ets/model/ListExchangeCtrl.ets:180`** (as shipped)

```typescript
deleteItem(item: T): void {
  const index: number = this.deductionData.indexOf(item);
  this.dragRefOffset = 0;
  getUIContext()?.animateTo({
    curve: Curve.Friction, duration: 300, onFinish: () => {          // 1. slide out + fade
      getUIContext()?.animateTo({
        curve: Curve.Friction, duration: 500, onFinish: () => {      // 2. then remove
          this.state = OperationStatus.IDLE;
        }
      }, () => {
        this.modifier.splice(index, 1);
        this.deductionData.splice(index, 1);
      });
    }
  }, () => {
    this.state = OperationStatus.DELETE;
    this.modifier[index].offsetX = 150;
    this.modifier[index].opacity = 0;
  });
}
```

**`onFinish` is how two phases are sequenced.** The row must finish sliding out
before it is spliced away, otherwise the removal snaps. The splice inside the
second `animateTo` is what drives the `TransitionEffect.OPACITY` on the
`ListItem` - the transition animates the node's actual removal from the tree.

## Permissions & config

None. `StickyNote.zip#entry/src/main/module.json5` declares no
`requestPermissions` block.

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "deliveryWithInstall": true,
    "installationFree": false,
    "pages": "$profile:main_pages",
    "abilities": [{
      "name": "EntryAbility",
      "srcEntry": "./ets/entryability/EntryAbility.ets",
      "exported": true,
      "skills": [{ "entities": ["entity.system.home"], "actions": ["action.system.home"] }]
    }]
  }
}
```

Root `build-profile.json5` targets `6.0.0(20)` for both `compatibleSdkVersion`
and `targetSdkVersion`.

`EntryAbility` writes `topRectHeight` and `bottomRectHeight` into `AppStorage`
and subscribes to `avoidAreaChange` without ever releasing it (`HW-02-0039`).
The page consumes both with `@StorageProp` - top as padding, bottom as a
trailing blank `ListItem`, which is the right shape for a scrolling list: the
last row can scroll clear of the navigation bar instead of being permanently
inset.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 117-119).
- The row height is fixed. `getModifier`, the pitch arithmetic and the
  auto-scroll clamp all assume every row is the same height; variable-height
  rows need a different approach entirely.
- `getModifier(item)` is an `indexOf` per row per frame, so the drag is O(n) per
  rendered row. Fine for a to-do list, not for hundreds of items - and note that
  `LazyForEach` cannot be used here at all, because the modifier array is
  indexed against the full data array.
- The seed data is a module-scope array (`MEMO_DATA`) that `@Provide
  appInfoList` takes by reference, so notes added or deleted persist across
  navigations within the process - and are lost on restart. There is no
  persistence.
- `visible` on `StickyNoteInfo` is declared and never read.
- `ListItemModifier.getInstance()` exists but is unused; a shared modifier would
  give every row the same transform.
- The `state` field (`OperationStatus`) is written on every transition and never
  read.

## Pitfalls

- **`HW-02-0034` - the drag step is 50 vp but the rows are 60 vp apart.** The
  row is 48 vp (`STICKY_NOTE_VIEW_HEIGHT`) and the list `space` is 12
  (`STICKY_NOTE_LIST_SPACE`). Every swap leaves the note 10 vp further from the
  finger. Derive one pitch constant instead of typing 50 in two files.
- **`HW-02-0035` - the swap bounds check comes after the offset is adjusted,
  and tests the wrong limits.** `target !== -1 && target <= length` should be
  `target >= 0 && target < length`, and it has to run before `offsetY` and
  `dragRefOffset` are shifted. As shipped, dragging the first note up or the
  last note down jumps it a row with no reorder.
- **`HW-02-0036` - the `ForEach` key is `JSON.stringify(item)`.** `isCompleted`
  is in that serialisation and the checkbox writes it, so every tick changes the
  key and ArkUI rebuilds the row. Use `item.id.toString()`.
- **`HW-02-0037` - the document's snippets call a bare `getUIContext()`.** It is
  a file-local helper in `ListExchangeCtrl.ets` reading `AppStorage`, and
  neither it nor the `aboutToAppear` line that seeds the value appears in the
  document. The snippets do not compile as printed.
- **`HW-02-0038` - the dialog initialises its own `controller` field with a
  self-referential `CustomDialogController`.** The reference declares it as
  `controller?: CustomDialogController;` with no initializer, for the framework
  to inject. Also clear `content` on confirm, or the next open shows the
  previous note.
- **`HW-02-0039` - `on('avoidAreaChange')` has no `off()`.**
- **`HW-02-0040` - the live `UIContext` is stored in `AppStorage`** under
  `'currentUIContext'` and never removed, keeping the window and the component
  tree reachable for the life of the process. Pass it into the controller
  instead.
- **`HW-02-0041` - the documented tree says `constants/` and
  `Entryability.ets`;** the zip has `Constant/` and `EntryAbility.ets`, and both
  spellings are load-bearing (`srcEntry` in `module.json5`, the import in
  `StickyNotePage.ets`).
- **`HW-02-0042` - `deductionDataSize` duplicates `deductionData.length`** and
  is decremented inside an animation callback, so the header count lags the list
  by 800 ms after a delete.
- **Do not use `GestureMode.Parallel`.** Sequential recognition is what lets a
  plain drag still scroll the list.
- **Do not splice one array without the other.** `getModifier` resolves by
  `indexOf` into the data array and indexes the modifier array with the result;
  the two must stay aligned or rows get each other's transforms.
- **Do not put `duration` on an `interpolatingSpring`.** The spring parameters
  determine the timing; `duration` is ignored.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-longpressgesture.md` - `LongPressGesture`, `onAction`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-longpressgesture
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - `PanGesture`, the cumulative `offsetY`, `onActionStart`/`Update`/`End`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/ts-combined-gestures.md` - `GestureGroup`, `GestureMode.Sequence`, `onCancel`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-combined-gestures
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `UIContext.animateTo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-references/02_application-framework/js-apis-curve.md` - `curves.initCurve`, `ICurve.interpolate`, `curves.interpolatingSpring(velocity, mass, stiffness, damping)`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-curve
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-attribute-modifier.md` - `AttributeModifier`, `applyNormalAttribute`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-attribute-modifier
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - key generation rules and the "use a unique id property" guidance
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `space`, `ListScroller.currentOffset`/`scrollTo`/`isAtEnd`, `swipeAction`, `backToTop`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-methods-custom-dialog-box.md` - `@CustomDialog`, the optional `controller` field
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-methods-custom-dialog-box
- `documentation/harmonyos-guides/03_application-framework/arkts-two-way-sync.md` - what `$$` does and does not synchronise
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-two-way-sync
- `LIFE-01` - the industry shell this page would sit in
