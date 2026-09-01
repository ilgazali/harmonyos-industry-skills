---
id: UTIL-07
title: Bottom sheet into side drawer - bindSheet for the peek, a RelativeContainer overlay for the full panel
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/07_drawer_sliding_effect.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/drawer_sliding_effect-0000002296764169
sample: huawei_industry_tree/15_utilities/downloads/BottomDrawerSlidingEffect.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [Stack, RelativeContainer, alignRules, AlignRuleOption, List, ListItemGroup, ListScroller, scrollToIndex, ScrollToIndexOptions, ScrollAlign, sticky, StickyStyle, edgeEffect, bindSheet, SheetType, "$$", TransitionEffect, animation, onTouch, TouchEvent, "window.WindowStage.getMainWindow", setImmersiveModeEnabledState, getWindowAvoidArea, "display.getAllDisplays", "@StorageProp"]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0024, HW-15-0025]
status: verified-with-fixes
---

## When to use

Load this card when a detail panel must **cover the map or canvas underneath
without relaying it out** - the pattern in ship tracking, ride hailing,
navigation and store locators: tap a marker, a card peeks up from the bottom;
tap the card, it becomes a full-height panel pinned to one side while the map
stays visible and interactive beside it.

The sample builds it out of two different mechanisms deliberately. The peek is
a system `bindSheet` - the framework owns its height, its drag handle and its
dismissal. The expanded panel is a plain `List` inside a `RelativeContainer`
inside a `Stack`, positioned with `alignRules` and dismissed by a horizontal
swipe the page interprets itself. Knowing when to stop using `bindSheet` is
the transferable lesson: a sheet is right while the panel is a modal
interruption, and wrong once the panel is a persistent second pane the user
works alongside.

**This sample's structure is a trap in places.** Two of its three showcase
mechanisms - the anchor tab bar and the slide-in offset - never execute. Read
`HW-15-0024` and `HW-15-0025` before copying the file wholesale; the parts
worth taking are marked below.

## Feature checklist

- An immersive full-screen page over a background map image, with a back
  arrow and a title in the top bar.
- A bottom sheet peeks up on entry showing a summary card plus an action row
  (favourite, alert, route, two capsule buttons).
- Tapping the summary card promotes it to the full side drawer.
- The drawer is a 290 vp-wide scrolling list pinned to the left, with a white
  status-bar spacer above it and a translucent black scrim over the rest of
  the screen.
- The scrim is tappable to dismiss.
- Section headers stick to the top of the drawer as it scrolls
  (`StickyStyle.Header`).
- Swiping the drawer left by more than 25% of the screen width closes it and
  restores the sheet.
- Sections (route, AIS message, mini message, weather) expand and collapse,
  rotating their chevron.
- Leaving the page turns immersive mode back off.

## Architecture

One `entry` module. One 880-line page plus four utilities.

```
entry/src/main/ets
├── constants/CommonConstants.ets   sizes, durations and three AlignRuleOption constants
├── entryability/EntryAbility.ets   full screen, avoid areas -> AppStorage, windowStage -> GlobalContext
├── pages/SideBarSlideCase.ets      the entire feature: 14 @Builders, 880 lines
└── utils
    ├── ArrayUtil.ets               isNotNullEmpty
    ├── GlobalContext.ets           a Map singleton, used to pass the WindowStage to the page
    ├── Logger.ets                  hilog wrapper
    └── WindowModel.ets             singleton over WindowStage: immersive mode, avoid areas, display size
```

The documented tree matches the zip.

**The design decision worth copying is `WindowModel`, and the one worth
avoiding is the 880-line page.** `WindowModel` is a singleton that takes the
`WindowStage` once and exposes four intentions - `setMainWindowImmersive`,
`getStatusBarHeight`, `getBottomAvoidHeight`, `getWindowHeight/Width` - each
with a sane fallback when the window is not available (`STATUS_BAR_HEIGHT =
38.8`, `BOTTOM_AVOID_HEIGHT = 10`). It also caches the `display.getAllDisplays()`
result in its own key/value map, so the second caller does not pay for the
async lookup. That is a good boundary: no page should be calling
`getMainWindow` directly.

The page is the counter-example. Fourteen `@Builder` methods and eleven state
fields in one struct, with the drawer's list rows, the sheet's contents, the
tab bar and the layout all in the same scope - which is exactly the condition
under which `tabOpacity` can be declared, styled, guarded on, and never
assigned by anybody (`HW-15-0024`), and under which three `'__container'`
typos survive review (`HW-15-0025`). Each section builder is an obvious
component boundary; extracting them would have made both defects visible.

## Implementation steps

1. **Publish the `WindowStage` where the page can reach it.** `EntryAbility`
   puts it in `GlobalContext` before `loadContent`; the page pulls it in
   `aboutToAppear` and hands it to `WindowModel`.
2. **Turn immersive mode on in `aboutToAppear` and off in `aboutToDisappear`.**
   It is window-global state - leaving it on leaks into whatever page follows.
3. **Read the display size once**, in vp, via
   `display.getAllDisplays()[0].width / densityPixels`. The swipe threshold
   depends on it.
4. **Peek with `bindSheet($$this.isShowDown, builder, { preferType: SheetType.BOTTOM })`.**
   Use the `$$` two-way binding so a system dismissal writes the flag back.
5. **Make the expanded drawer a sibling in a `RelativeContainer`,** not a
   sheet: scrim `Column`, status spacer, `List`, tab bar, each with an `id`
   and `alignRules` anchored to `'__container__'` or to a named sibling.
   **Spell it `'__container__'` with both trailing underscores**
   (`HW-15-0025`).
6. **Give the `List` a `ListScroller`** so the tab bar can `scrollToIndex`
   into it, and `sticky(StickyStyle.Header)` so `ListItemGroup` headers pin.
7. **Wrap every direct `List` child in `ListItem` or `ListItemGroup`** - the
   sample's `shipImg()` emits a bare `Stack` (`HW-15-0025`).
8. **Drive the tab bar's opacity from scroll state** rather than leaving it at
   its initialiser, or the guard `this.isShow && this.tabOpacity !== 0` can
   never pass and the whole navigation is unreachable (`HW-15-0024`). Anchor
   it to an id that exists.
9. **Interpret the dismiss swipe in `onTouch`**, comparing the `Down` and `Up`
   x-coordinates against a fraction of the window width, and make sure any
   offset you animate is `@State` (`HW-15-0025`).

## Verified snippets

All snippets are from `BottomDrawerSlidingEffect.zip`. Corrected forms are
marked.

**The overlay skeleton — `entry/src/main/ets/pages/SideBarSlideCase.ets`** (corrected, see `HW-15-0024`)

```typescript
build() {
  Stack({ alignContent: Alignment.TopStart }) {
    RelativeContainer() {
      if (!this.isShow) {
        Row() {
          this.Top();                                   // back arrow + title
        }
        .width('100%')
        .bindSheet($$this.isShowDown, this.SheetTransition(), {
          showClose: false,
          height: 312,
          width: '100%',
          preferType: SheetType.BOTTOM,
        });
      }

      if (this.isShow) {
        Column()                                        // the scrim
          .id('_padding')
          .width(this.windowWidth)
          .height('100%')
          .backgroundColor('rgba(0,0,0,0.4)')
          .alignRules({
            'right': { 'anchor': '__container__', 'align': HorizontalAlign.End }
          })
          .onClick(() => {
            this.isShow = false;
            this.isShowDown = true;                     // hand control back to the sheet
          });

        this.StatusHead();                              // id('statusHead'), a white status-bar spacer
        this.drawerList();                              // id('scrollPart'), anchored under statusHead
      }

      if (this.isShow && this.tabOpacity !== 0) {       // FIX: tabOpacity is never assigned
        this.listIndex();
      }
    }
    .backgroundImage($r('app.media.background_map'))
    .transition(TransitionEffect.opacity(0.99));
  }
  .width('100%')
  .height('100%');
}
```

**Three structural choices carry this layout.** The outer `Stack` is what lets
the drawer *cover* the map rather than push it - the document's whole premise,
"不变动原页面布局" (does not alter the original page layout). The
`RelativeContainer` is what lets the four overlay pieces position themselves
against each other by name: the scrim right-aligns to the container, the list
tops out at `statusHead`'s bottom edge, and nothing needs to know anyone
else's pixel height. And `isShow` / `isShowDown` are two separate flags, not
one tri-state, so the transition from sheet to drawer is two assignments and
either can be driven independently by the scrim, the arrow, the swipe or the
system.

`bindSheet` takes `$$this.isShowDown` rather than `this.isShowDown`: the `$$`
makes the binding two-way, so when the user drags the sheet away, the
framework writes `false` back into the state. Without it the flag and the
sheet drift apart in exactly the way `TOUR-03`'s popup does without
`onStateChange`.

The last `if` is the defect. `tabOpacity` is declared `tabOpacity: number = 0`
- not `@State`, never written anywhere in the file - so the guard is
permanently false and `listIndex()` is dead code.

**The drawer list — same file** (corrected, see `HW-15-0025`)

```typescript
List({ scroller: this.listScroller, space: 11 }) {
  this.shipImg();          // FIX: must be wrapped in ListItem - it emits a bare Stack
  this.shipDirection();    // ListItemGroup
  this.aMessage();         // ListItemGroup
  this.miniMsg();          // ListItemGroup
  this.weatherMsg();       // ListItemGroup
}
.padding({ bottom: 55 })
.alignListItem(ListItemAlign.Start)
.id('scrollPart')
.alignRules({
  'top': { 'anchor': 'statusHead', 'align': VerticalAlign.Bottom },
  'left': { 'anchor': '__container__', 'align': HorizontalAlign.Start }
})
.width(this.listWidth)                 // 290 vp
.height('100%')
.edgeEffect(EdgeEffect.None)
.scrollBar(BarState.Off)
.sticky(StickyStyle.Header)            // ListItemGroup headers pin while scrolling
.backgroundColor(this.isShow ? Color.White : $r('app.color.list_show_background_color'))
.transition(TransitionEffect.OPACITY.animation({ duration: 200, curve: Curve.LinearOutSlowIn })
  .combine(TransitionEffect.translate({ x: -100 })))
.onTouch((event: TouchEvent) => {
  switch (event.type) {
    case TouchType.Down: {
      this.xStart = event.touches[0].x;
      break;
    }
    case TouchType.Up: {
      const xEnd = event.touches[0].x;
      const width = Math.abs(Math.abs(xEnd) - Math.abs(this.xStart));
      if (width >= this.windowWidth * 0.25 && xEnd < this.xStart) {
        this.isLeft = true;
        this.isShow = false;           // swiped left far enough: close
      } else {
        this.isLeft = false;
        this.isShow = true;
      }
      this.xStart = event.touches[0].x;
      return;
    }
  }
})
.align(Alignment.Start);
```

**`transition` is what actually animates this drawer**, and it is the right
tool: the list is conditionally rendered by `if (this.isShow)`, so it appears
and disappears rather than moving, and `TransitionEffect.OPACITY.combine(
TransitionEffect.translate({ x: -100 }))` gives ArkUI both halves of the
appear/disappear animation in one declaration - fade in while sliding 100 vp
from the left. Nothing needs to hold an offset in state.

Which is why the `.offset({ x: this.positionX })` plus 500 ms `.animation({...})`
that the shipped code also carries is inert: `positionX` is a plain field
initialised to `0` and never assigned, so there is no state change for the
`animation` attribute to pick up. Two animation systems declared, one working.
Delete the offset pair or make `positionX` `@State` and drive it - do not
leave both.

The swipe detection is worth understanding before copying. It compares only
the `Down` and `Up` x-coordinates, so it is a flick test, not a drag: the
drawer does not follow the finger. That is a legitimate simplification for a
dismissal gesture, but note the `else` branch sets `isShow = true` on every
short touch - so a plain tap inside the list re-asserts the drawer open. Fine
here; wrong if the list rows had their own navigation.

**The anchor tab bar — same file** (corrected, see `HW-15-0024`)

```typescript
@State tabOpacity: number = 0;         // FIX: was a plain field, never @State, never assigned

@Builder
listIndex() {
  Row() {
    this.tabBar($r('app.string.bottomTab_text_main7'), 1);
    this.tabBar($r('app.string.list_text_main7'), 2);
    this.tabBar($r('app.string.list_text_main8'), 3);
    this.tabBar($r('app.string.list_text_main9'), 4);
  }
  .padding({ left: 16, right: 16 })
  .opacity(this.tabOpacity)
  .width(this.listWidth)
  .height(60)
  .id('listIndex')
  .backgroundColor($r('app.color.list_show_background_color'))
  .alignRules({
    'top': { 'anchor': 'statusHead', 'align': VerticalAlign.Bottom },  // FIX: was 'sideBarHead', no such id
    'left': { 'anchor': '__container__', 'align': HorizontalAlign.Start }
  });
}

@Builder
tabBar(text: ResourceStr, index: number) {
  Column() {
    Text(text)
      .fontSize(16)
      .fontWeight(index === this.tabChoseIndex ? FontWeight.Bold : FontWeight.Normal)
      .fontColor(Color.Black);
    if (index === this.tabChoseIndex) {
      Divider()
        .strokeWidth(2)
        .width(25)
        .lineCap(LineCapStyle.Round)
        .color(Color.Blue)
        .margin({ top: 5 });
    }
  }
  .onClick(() => {
    this.tabChoseIndex = index;
    let scrollToIndexOptions: ScrollToIndexOptions = { extraOffset: { value: -102, unit: 1 } };
    this.listScroller.scrollToIndex(index, false, ScrollAlign.START, scrollToIndexOptions);
  })
  .width(60)
  .height(30)
  .margin({ left: 5, right: 5 });
}
```

**This is the most reusable code in the sample and none of it runs.** The
mechanism is exactly right for a sectioned panel: a shared `ListScroller`,
`scrollToIndex(index, smooth, ScrollAlign.START, { extraOffset })` to land a
section under the sticky header, and an underline `Divider` rendered only for
the selected tab so the indicator needs no separate animation or measurement.
`extraOffset: { value: -102, unit: 1 }` (unit 1 is vp) is what compensates for
the pinned header height - without it `ScrollAlign.START` puts the section
title underneath the sticky one.

To bring it to life you need three changes: make `tabOpacity` `@State`, assign
it (the obvious driver is the list's `onScroll`/`onDidScroll` - fade the bar in
past a scroll threshold), and anchor the row to an id that exists.
`'sideBarHead'` appears nowhere else in the project; `'statusHead'` is the
spacer directly above the list.

**Window management — `entry/src/main/ets/utils/WindowModel.ets`** (as shipped)

```typescript
export default class WindowModel {
  public static readonly STATUS_BAR_HEIGHT = 38.8;
  public static readonly BOTTOM_AVOID_HEIGHT = 10;
  private static instance?: WindowModel;
  private objects = new Map<string, Object>();
  private windowStage?: window.WindowStage;

  static getInstance(): WindowModel {
    if (!WindowModel.instance) {
      WindowModel.instance = new WindowModel();
    }
    return WindowModel.instance;
  }

  setMainWindowImmersive(enable: boolean): void {
    if (this.windowStage === undefined) {
      Logger.error('windowStage is undefined.');
      return;
    }
    this.windowStage.getMainWindow((err, windowClass: window.Window) => {
      if (err.code) {
        Logger.error(`Failed to obtain the main window. Code:${err.code}, message:${err.message}`);
        return;
      }
      try {
        windowClass.setImmersiveModeEnabledState(enable);
      } catch (error) {
        Logger.error(`Failed to set full-screen mode. Code:${err.code}, message:${err.message}`);
      }
    });
  }

  async getWindowWidth(): Promise<number> {
    let globalDisplays = WindowModel.getInstance().getObject('display') as Array<display.Display>;
    if (ArrayUtil.isNotNullEmpty(globalDisplays)) {
      return globalDisplays[0].width / globalDisplays[0].densityPixels;   // px -> vp
    }
    globalDisplays = await display.getAllDisplays();
    WindowModel.getInstance().setObject('display', globalDisplays);       // cache for the next caller
    return globalDisplays[0].width / globalDisplays[0].densityPixels;
  }
}
```

**Two things make this worth lifting into your own project.** The width is
divided by `densityPixels` at the source, so every consumer works in vp and
nobody downstream has to remember a `px2vp` - contrast the avoid-area values,
which `EntryAbility` pushes into `AppStorage` as raw px and every page has to
convert. And `display.getAllDisplays()` is async but effectively constant, so
caching the array in the singleton's map turns every call after the first into
a synchronous return.

`setMainWindowImmersive` is called with `true` in `aboutToAppear` and `false`
in `aboutToDisappear`, which is the discipline immersive mode needs: it is a
property of the window, not of the page, so a page that turns it on owes the
next page turning it off.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`.

`EntryAbility` does the usual four things before `loadContent`:
`setColorMode(COLOR_MODE_LIGHT)`, `setWindowLayoutFullScreen(true)`,
`getWindowAvoidArea` for the top and bottom rects into `AppStorage`, and an
`avoidAreaChange` listener that keeps them current. It also stashes the
`WindowStage` in `GlobalContext` so the page can construct `WindowModel`.
The `setWindowLayoutFullScreen` promise is fire-and-forget and the
`avoidAreaChange` listener is never removed in `onWindowStageDestroy` - the
same boilerplate defect that recurs across these samples.

Note that the page declares `@StorageProp('bottomRectHeight')` and
`@StorageProp('topRectHeight')` and then never uses either: the layout uses
the hardcoded `statusBarHeight: number = 38` instead. If you adapt this page,
use the published values.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The drawer width is a hardcoded `290` vp assigned in `aboutToAppear`, and
  the status spacer a hardcoded `38` vp. Neither adapts to a tablet, a
  landscape window or a device with a taller status bar.
- The sheet height is a fixed `312` vp with `showClose: false`, so the only
  way out of the peek is the card's own close icon or dragging it away.
- All list content is static string and media resources - there is no model
  layer, no data source and no `LazyForEach`. The list is a layout showcase.
- The page depends on `GlobalContext` having been populated by `EntryAbility`;
  entering it by any other route leaves `windowStage` undefined and
  `aboutToAppear` returns early, so immersive mode and the window size are
  silently never set (`windowWidth` stays `0`, which makes the swipe threshold
  `0` and every touch-up close the drawer).
- `bottomTab()` renders three `Image(true ? A : B)` ternaries - placeholders
  for a selected state that was never wired.

## Pitfalls

- **`HW-15-0024`** (B/medium, confirmed): the drawer's section tab bar is
  dead. `tabOpacity` is declared as a plain field initialised to `0` and never
  assigned, so the render guard `if (this.isShow && this.tabOpacity !== 0)`
  can never pass; even if it did, `listIndex`'s `alignRules` anchor
  `'sideBarHead'`, an id that exists nowhere in the project. The implemented
  anchor navigation (`tabBar` + `scrollToIndex`) is unreachable. Fix: make it
  `@State`, drive it from scroll state, and anchor to `'statusHead'`.
- **`HW-15-0025`** (B/low, confirmed): a dead-code cluster in the same page -
  the `Top()` builder and `CommonConstants.HOME_TOP_ALIGN_RULES` /
  `LIST_TITLE_ALIGN_RULES` anchor `'__container'` instead of `'__container__'`,
  so those rules match nothing; `positionX` is a non-`@State` field that is
  never assigned, leaving the `.offset({ x: positionX })` plus 500 ms
  `.animation()` pair permanently inert; and `shipImg()` emits a bare `Stack`
  directly under `List`, which accepts only `ListItem` and `ListItemGroup`
  children. Fix: correct the keyword, wire or delete `positionX`, wrap the
  `Stack` in a `ListItem`.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-stack.md` - `Stack` and `alignContent`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-stack
- `documentation/harmonyos-references/02_application-framework/ts-container-relativecontainer.md` - `alignRules`, `AlignRuleOption`, the `'__container__'` keyword
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-relativecontainer
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, valid children, `ListScroller.scrollToIndex`, `sticky`, `edgeEffect`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - `bindSheet`, `SheetOptions`, `SheetType`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-windowstage.md` - `getMainWindow`, `setImmersiveModeEnabledState`, `getWindowAvoidArea`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-windowstage
