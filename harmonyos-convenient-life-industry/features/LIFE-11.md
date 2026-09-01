---
id: LIFE-11
title: Pinch to switch card view - a two-finger gesture that collapses one card into a snapping deck and back
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/11_card_pinch_scale.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_pinch_scale-0000002282825670
sample: huawei_industry_tree/02_convenient_life/downloads/CardPinchScale.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: [PinchGesture, GestureEvent, "GestureEvent.scale", onActionUpdate, onActionEnd, List, ListItem, listDirection, scrollSnapAlign, ScrollSnapAlign, chainAnimation, edgeEffect, EdgeEffect, initialIndex, Tabs, TabContent, tabBar, onContentWillChange, BarPosition, "@State", "@Prop", "@Builder", ForEach, "window.getMainWindow", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "AppStorage.setOrCreate"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-02-0076, HW-02-0077, HW-02-0078, HW-02-0079, HW-02-0080, HW-02-0081, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card when a pinch should **change what is on screen rather than
magnify it** - the iOS-style "pinch out of a document to see all documents"
idiom. Pinch in on the open card and you get the deck; pinch out on a card in
the deck (or tap it) and you are back in that card.

The whole mechanism is two `PinchGesture`s at different levels of the tree and
one boolean:

```
page Column      PinchGesture -> onActionEnd: scale < 1  -> isSingleTodo = false   (collapse)
each ListItem    PinchGesture -> onActionEnd: scale > 1  -> isSingleTodo = true    (expand)
```

There is no animation of the card itself - the sample does not scale anything.
The pinch is used purely as a **direction-carrying trigger**, and the visual
effect comes from the `List` that replaces the card: horizontal,
centre-snapping, with chain animation.

Take this for memo/note apps, card decks, photo albums, dashboards - anywhere a
single item and its siblings are two views of the same data. If you want the
card to actually scale under the fingers, the `PinchGesture` reference's own
example (with `scaleValue`/`pinchValue` and a `.scale()` binding) is the pattern
to follow instead.

## Feature checklist

- One card fills the screen, showing a title and five icon-plus-text rows.
- A two-finger pinch in, anywhere on the page, collapses to a horizontal deck.
- The deck snaps each card to the centre, with a chain animation between
  neighbours and a spring at the ends, opening on the fourth card.
- Tapping a card in the deck, or pinching out on it, returns to the single-card
  view showing that card.
- The card renders differently in the two modes - font sizes, icon sizes, row
  height, corner radius and background all switch on one flag.
- A bottom tab bar with three tabs; the second and third refuse to open and show
  a toast instead.

## Architecture

One `entry` module, one page, one reusable card component.

```
entry/src/main/ets
├── common
│   ├── Model.ets        ContentRowInterface { icon, content }, TodoItemType { title, items, isListView }
│   ├── CardData.ets     TODO_LIST - five entries (all identical)
│   └── Constants.ets    sizes, colours, TWO_FINGERS = 2, INITIAL_INDEX = 3
├── entryability/EntryAbility.ets        full screen, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
└── pages
    ├── CardView.ets     the card, in either presentation
    └── MainPage.ets     @Entry - Tabs, the two gestures, the deck
```

The documented tree matches the zip exactly.

**One component renders both presentations.** `CardView` takes
`@Prop isListView` and switches six attributes on it - title font, row height,
icon size, divider inset, corner radius and background. That is the reason there
is no separate list-cell component, and it is the part of this sample most worth
copying: the deck and the detail view can never drift apart, because they are
the same `build()`.

**Two `PinchGesture`s, one flag.** The outer gesture is on the page-level
`Column` inside `mainContent()`; the inner one is on each `ListItem`. ArkUI
resolves the innermost gesture first, so in deck mode a pinch on a card runs the
card's handler and a pinch on the background runs the page's. Only one boolean
changes:

```
isSingleTodo = true   -> CardView(selectedItem, isListView: false)
isSingleTodo = false  -> List of CardView(item, isListView: true)
```

**The deck effect is four `List` attributes, not animation code:**
`listDirection(Axis.Horizontal)`, `scrollSnapAlign(ScrollSnapAlign.CENTER)`,
`chainAnimation(true)` and `edgeEffect(EdgeEffect.Spring)`. None of them appears
in the document (`HW-02-0079`).

The pinch ratio itself is routed through `@State scaleValue`, which is where the
sample's main defect lives (`HW-02-0076`).

## Implementation steps

1. **Model the card content and nothing else.** `title` plus a list of
   `{ icon, content }`. The presentation flag belongs to the view, not the data
   (`HW-02-0080`).
2. **Write one card component with an `isListView` `@Prop`** and switch every
   size, colour and radius on it. Both modes then stay in step by construction.
3. **Give each entry a stable id and key the `ForEach` on it** (`HW-02-0077`).
4. **Attach the collapse gesture to the page-level container**, so a pinch
   anywhere - not only on the card - triggers it.
5. **Attach the expand gesture to each `ListItem`,** alongside an `onClick` that
   does the same thing. Two ways into the same state change is the whole point:
   the pinch is discoverable, the tap is reliable.
6. **Read the ratio from the event, not from state.** `onActionEnd` receives a
   `GestureEvent`; use `event.scale` and delete the `@State` (`HW-02-0076`).
7. **Configure the deck with the four `List` attributes** above.
   `initialIndex` opens the deck on a chosen card.
8. **Await `setWindowLayoutFullScreen` before reading the avoid areas,** and
   subscribe to `avoidAreaChange` (`HW-02-0078`).

## Verified snippets

All snippets are from `CardPinchScale.zip`. Corrected forms are marked.

**One component, two presentations - `CardPinchScale.zip#entry/src/main/ets/pages/CardView.ets`** (as shipped)

```typescript
@Component
export struct CardView {
  @Prop title: string;
  @Prop items: Array<ContentRowInterface>;
  @Prop isListView: boolean;

  @Builder
  contentRow(item: ContentRowInterface, isList: boolean) {
    Row() {
      Image(item.icon)
        .width(isList ? Constants.IMAGE_LIST_SIZE : Constants.CONTENT_IMAGE_SIZE)   // 32 : 48
        .aspectRatio(1);
      Text(item.content)
        .fontSize(isList ? Constants.FONT_LIST_CONTENT : Constants.FONT_SIZE_NORMAL) // 14 : 16
        .margin({ left: Constants.MARGIN_SIXTEEN });
    }
    .width(Constants.FULL_WIDTH)
    .height(isList ? Constants.CONTENT_ROW_LIST_HEIGHT : Constants.CONTENT_ROW_HEIGHT); // 60 : '17.5%'
  }

  build() {
    Column() {
      Text(this.title)
        .fontSize(this.isListView ? Constants.FONT_LIST_TITLE : Constants.FONT_SIZE_ORIGINAL);

      ForEach(this.items, (item: ContentRowInterface, index: number) => {
        this.contentRow(item, this.isListView);
        if (index < this.items.length - 1) {          // no divider after the last row
          this.div(this.isListView);
        }
      }, (item: ContentRowInterface, index: number) => `${index}-${item.icon}`);
    }
    .height(this.isListView ? Constants.FULL_HEIGHT : Constants.CARD_HEIGHT)   // '100%' : '62%'
    .borderRadius(this.isListView ? Constants.RADIUS : 0)
    .backgroundColor(this.isListView ? Constants.WHITE : Constants.BACKGROUND_COLOR)
    .padding({ left: Constants.MARGIN_TWENTYFOUR, right: Constants.MARGIN_TWENTYFOUR });
  }
}
```

**Note the row height switches unit as well as value:** a fixed `60` vp in the
deck, a proportional `'17.5%'` in the detail view. The deck cell has a bounded
height from its `ListItem`, so a percentage there would collapse; the detail
card fills the page, so a percentage is what keeps five rows evenly spread
whatever the screen height.

**`if (index < this.items.length - 1)` inside a `ForEach`** is the idiomatic way
to omit the trailing separator - ArkUI's conditional rendering works at any
depth of the builder, so no `Divider` node is created for the last row at all.

The key `${index}-${item.icon}` interpolates a `Resource` object, which
stringifies to `[object Object]` - so the index is doing all the work. Harmless
here (the rows never change), but it is not the unique key it looks like.

**The two gestures - `CardPinchScale.zip#entry/src/main/ets/pages/MainPage.ets:59`** (corrected, see `HW-02-0076` and `HW-02-0077`)

```typescript
Row() {
  if (this.isSingleTodo) {
    CardView({ title: this.selectedItem.title, items: this.selectedItem.items, isListView: false });
  } else {
    List({ space: Constants.SPACE, initialIndex: Constants.INITIAL_INDEX, scroller: this.scrollerForList }) {
      ForEach(TODO_LIST, (item: TodoItemType, index: number) => {
        ListItem() {
          CardView({ title: item.title, items: item.items, isListView: true })
            .onClick(() => {                       // tap: the reliable way in
              this.selectedItem = item;
              this.isSingleTodo = true;
            });
        }
        .width(Constants.LIST_CARD_WIDTH)          // '60%' - neighbours peek in
        .height(Constants.LIST_ITEM_HEIGHT)
        .gesture(
          PinchGesture({ fingers: Constants.TWO_FINGERS })
            .onActionEnd((event: GestureEvent) => {          // FIX: sample takes no parameter
              if (event.scale > 1) {                         // FIX: sample reads this.scaleValue
                this.isSingleTodo = true;
                this.selectedItem = item;
              }
            })
        );
      }, (item: TodoItemType) => item.id);         // FIX: sample supplies no key generator
    }
    .chainAnimation(true)
    .edgeEffect(EdgeEffect.Spring)
    .listDirection(Axis.Horizontal)
    .scrollSnapAlign(ScrollSnapAlign.CENTER)
    .scrollBarWidth(0);
  }
}
```

**`LIST_CARD_WIDTH` is `'60%'` and the snap is `CENTER`** - together those two
give the deck its shape: the focused card sits in the middle with its neighbours
visibly cut off at both edges, and letting go always centres exactly one card.
Change either and it stops reading as a deck.

**`chainAnimation(true)`** makes the neighbouring items trail the dragged one
with a spring rather than moving rigidly - the effect the 效果预览 animation is
showing.

**`onClick` and the pinch do the same two assignments.** Keeping both is
deliberate: the pinch is the feature, the tap is the fallback for users who
never discover it.

**The page-level collapse gesture - same file, line 112** (corrected, see `HW-02-0076`)

```typescript
}
.width(Constants.FULL_WIDTH)
.height(Constants.FULL_HEIGHT)
.gesture(
  PinchGesture({ fingers: Constants.TWO_FINGERS })
    .onActionEnd((event: GestureEvent) => {      // FIX: sample takes no parameter
      if (event.scale < 1) {                     // FIX: sample reads this.scaleValue
        this.isSingleTodo = false;
      }
    })
);
```

**Attaching it to the outermost `Column` of `mainContent`, not to the card,** is
what makes the gesture work from anywhere on the page - including the margins
around the card. ArkUI dispatches to the innermost matching gesture first, so
this handler is reached only when no `ListItem` claimed the pinch.

**Why the shipped `@State` is wrong on two counts.** As written, both handlers
take no argument and branch on `this.scaleValue`, which only `onActionUpdate`
writes. `onActionUpdate` is documented as firing "when the user moves the finger
in the pinch gesture on the screen", so a pinch that is recognised at the 5 vp
threshold and released without further movement reaches `onActionEnd` with the
**previous** gesture's ratio. And because `scaleValue` is `@State` that nothing
renders, every frame of every pinch rebuilds the whole page - including the
five-card `List` with its chain animation - to update a number that is never
displayed. Reading `event.scale` fixes both, and `onActionUpdate` can then be
deleted entirely.

**The pinch threshold is not zero.** The reference: "PinchGesture is used to
trigger a pinch gesture, which requires two to five fingers with a minimum 5 vp
distance between the fingers", with `distance` defaulting to 5. So `event.scale`
is already meaningfully above or below 1 by the time the gesture is recognised -
a bare `> 1` / `< 1` test is enough and no dead-zone is needed.

**The tab shell - same file, line 142** (as shipped)

```typescript
Tabs({ barPosition: BarPosition.End }) {
  TabContent() { this.mainContent(); }
    .tabBar(this.tabBuilder($r('app.string.home'), $r('app.media.home'), true));
  TabContent() { }
    .tabBar(this.tabBuilder($r('app.string.message'), $r('app.media.message'), false));
  TabContent() { }
    .tabBar(this.tabBuilder($r('app.string.mine'), $r('app.media.mine'), false));
}
.onContentWillChange(() => {
  this.getUIContext().getPromptAction().showToast({ message: '仅展示', duration: Constants.TOAST_INTERVAL_TIME });
  return false;
});
```

**`onContentWillChange` returning `false` vetoes the tab switch** before it
happens - which is the clean way to stub out unimplemented tabs in a demo:
the bar still renders and responds, but the content never changes and the user
gets a 仅展示 ("display only") toast. Returning `true` (or omitting the hook)
would let the empty `TabContent`s show.

## Permissions & config

None. `CardPinchScale.zip#entry/src/main/module.json5` declares no
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
    }],
    "extensionAbilities": [{
      "name": "EntryBackupAbility",
      "srcEntry": "./ets/entrybackupability/EntryBackupAbility.ets",
      "type": "backup",
      "exported": false,
      "metadata": [{ "name": "ohos.extension.backup", "resource": "$profile:backup_config" }]
    }]
  }
}
```

Root `build-profile.json5` targets `6.0.0(20)`.

`EntryAbility.onWindowStageCreate` is `async`, takes the main window with
`await windowStage.getMainWindow()`, and writes `topHeight` /
`bottomHeight` into `AppStorage` for the page to read with
`AppStorage.get<number>(...)` - but it drops the `setWindowLayoutFullScreen`
promise and never subscribes to `avoidAreaChange` (`HW-02-0078`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 63-65).
- `PinchGesture` needs two to five fingers and 5 vp of movement before it is
  recognised. There is no single-finger path into the deck - only the tap out of
  it.
- The reference notes: "To trigger the pinch gesture again after successful
  recognition, all fingers must be lifted and then make contact again." A
  continuous pinch cannot toggle twice.
- Nothing is animated between the two views. The transition is an immediate
  `if`/`else` swap; `chainAnimation` only animates within the deck.
- All five entries in `TODO_LIST` are byte-identical, so selecting any card
  shows the same content - which also hides any key-collision symptom
  (`HW-02-0077`).
- The card content is a fixed five rows. `CONTENT_ROW_HEIGHT` is `'17.5%'`, so
  a longer list overflows the `'62%'`-high card rather than scrolling.
- Tabs 2 and 3 are permanently vetoed by `onContentWillChange`.
- Two-finger gestures are not reachable with an accessibility service driving
  the screen; the `onClick` fallback matters.

## Pitfalls

- **`HW-02-0076` - both `onActionEnd` handlers ignore their `GestureEvent` and
  branch on a `@State`.** The reference signature is
  `onActionEnd(event: (event: GestureEvent) => void)`. Reading `this.scaleValue`
  instead means a recognised-but-not-moved pinch decides on the previous
  gesture's ratio, and the `@State` write re-renders the whole page on every
  pinch frame for a value nothing displays. Use `event.scale` and delete the
  field.
- **`HW-02-0077` - the deck's `ForEach` has no key generator** over five
  byte-identical entries, so the default key differs only by index - which the
  guidance rules out. Add an `id` to `TodoItemType`.
- **`HW-02-0078` - `setWindowLayoutFullScreen` is not awaited** in a method that
  awaits the line above, and the avoid areas are read immediately after and
  never refreshed.
- **`HW-02-0079` - the document's snippets do not compile.** Both reference an
  undeclared `this.scaleValue`, both have empty bodies, the empty `Column`
  hides that the collapse gesture is page-wide, and none of the four `List`
  attributes that produce the deck effect appears anywhere in the document.
- **`HW-02-0080` - `TodoItemType.isListView` is stored and never read.** The
  mode comes from literals at the two call sites; the data value is `false` on
  every entry and would be wrong in the deck branch if it were used.
- **`HW-02-0081` - a `TextInputController` is constructed on a page with no
  text input.**
- **Do not attach the collapse gesture to the card.** On the page container it
  works from the margins too; on the card it stops working the moment the card
  does not fill the screen.
- **Do not animate the card scale unless you mean it.** This pattern uses the
  pinch as a direction signal only - binding `.scale()` to the live ratio and
  then swapping views is a different, harder interaction.
- **Do not drop the `onClick`.** A two-finger gesture is undiscoverable on its
  own and unavailable under an accessibility service.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pinchgesture.md` - `PinchGesture({ fingers, distance })`, the 5 vp threshold, `onActionUpdate`/`onActionEnd` signatures, `GestureEvent.scale`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pinchgesture
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `listDirection`, `scrollSnapAlign`, `chainAnimation`, `edgeEffect`, `initialIndex`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - default key generation and the "do not use the index as the key" rule
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` - `Tabs`, `tabBar`, `onContentWillChange` and its veto return
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `getMainWindow`, `setWindowLayoutFullScreen`, `getWindowAvoidArea`, `on`/`off('avoidAreaChange')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `LIFE-06` - the same industry's other gesture card, where `LongPressGesture` + `PanGesture` are combined in a sequential `GestureGroup`
- `LIFE-01` - the industry shell this page would sit in
