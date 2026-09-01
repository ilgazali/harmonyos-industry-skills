---
id: LIFE-17
title: Two-way linked category lists - a thin index list driving a section list, and a guard flag to stop the feedback loop
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/17_dual_list_linkage.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/dual_list_linkage-0000002319372516
sample: huawei_industry_tree/02_convenient_life/downloads/DualListLinkage.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit"]
apis: [List, ListItem, ListItemGroup, Scroller, "scroller.scrollToIndex", onScrollIndex, onScrollStop, Grid, GridItem, columnsTemplate, Tabs, TabContent, TabsController, "controller.changeIndex", onAnimationStart, barHeight, "@Observed", "@ObjectLink", "@State", "@Consume", "@Provide", NavPathStack, NavDestination, onBackPressed, "MeasureUtils.measureText", "context.terminateSelf", "PromptAction.showToast", "AppStorage.get"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-02-0121, HW-02-0122, HW-02-0123, HW-02-0124, HW-02-0125, HW-02-0126, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card for the **category index beside a sectioned list** - the layout
every food-delivery menu, service directory and product catalogue uses. Tapping
a category on the left jumps the right-hand list to that section; scrolling the
right-hand list moves the highlight on the left.

The interesting problem is not either direction on its own - both are three
lines - it is that **each direction triggers the other**. Scrolling
programmatically fires `onScrollIndex`, which would move the left selection,
which would scroll again. The sample's answer is a boolean that suppresses the
reverse path while a programmatic scroll is in flight. That idea is right; the
way it is released is not (`HW-02-0121`), and it is the single thing to get
right when copying this.

Take this for menus, catalogues, settings with a section index, contact lists
with an A-Z rail. If the left list is just a filter and the right list should
*reload* rather than scroll, you want two independent lists and no linkage at
all.

## Feature checklist

- A left rail (26 % width) of eight category names, the current one highlighted.
- A right pane (74 %) listing every category in turn: a heading and a 3-column
  grid of services.
- Tapping a category scrolls the right pane to that section, smoothly.
- Scrolling the right pane moves the left highlight to whatever section is at
  the top.
- The last section carries 320 vp of bottom padding so it can reach the top of
  the viewport.
- Above: a back title bar, a search row, and a three-tab bar; only the first tab
  has content, the others toast.

## Architecture

One `entry` module, one page. The two lists share one data array.

```
entry/src/main/ets
├── constants/CommonConstant.ets   widths (26 % / 74 %), numeric constants, AppStorage keys
├── datas/ListData.ets             @Observed Level1Data { categoryName, isSelect, level2List }
│                                  @Observed Level2Data { serviceIcon, serviceName }
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── pages
│   ├── NavigationPage.ets         @Entry - Navigation + @Provide NavPathStack
│   └── DoubleListLinkagePage.ets  THE CARD: both lists, the linkage, the tab shell
└── utils/DisplayUtil.ets          status-bar height from AppStorage
```

The documented tree matches the zip exactly.

**One array feeds both lists.** `mList: Level1Data[]` is iterated by the left
rail (rendering `categoryName` and `isSelect`) and by the right pane (rendering
`categoryName` as a heading plus `level2List` as a grid). Because the index into
`mList` is the same on both sides, `scrollToIndex(i)` and `onScrollIndex(start)`
speak the same language and no mapping table is needed. That is the reason the
right pane is a list of *categories* rather than a flat list of services.

**The selection lives on the data, not in the view.** `Level1Data.isSelect` is
the highlight, `Level1Data` is `@Observed`, and `LeftListChild` binds it with
`@ObjectLink` - so flipping the flag repaints one row without the parent
re-rendering either list. `currentIndex` is a plain field, deliberately: it is
only ever read to find which flag to clear.

**The loop and its guard:**

```
tap left row i          isScrollUpdate = false          <- suppress the reverse path
                        mList[currentIndex].isSelect = false
                        currentIndex = i
                        level2Scroller.scrollToIndex(i, true)     smooth
                        mList[i].isSelect = true
                        setTimeout(() => isScrollUpdate = true, 500)   <- HW-02-0121

scroll right pane       onScrollIndex(start)
                        if (isScrollUpdate) { move the highlight to `start` }
```

## Implementation steps

1. **Model each category as one observed object** carrying its own selection
   flag and its children. Give it an id so the lists can be keyed
   (`HW-02-0124`).
2. **Build the right pane as sections, not as a flat list,** so its indices line
   up 1:1 with the left rail. Use `ListItemGroup` rather than a `ListItem`
   containing a `Grid` (`HW-02-0123`).
3. **Wrap the row component in a `ListItem` at the call site** and put the
   `onClick` on the `ListItem`, not on the custom component (`HW-02-0122`).
4. **Give the last section enough bottom padding** to reach the top of the
   viewport, or the final category can never become the top-most one and the
   linkage cannot select it.
5. **Suppress the reverse path with a flag while scrolling programmatically,**
   and clear it from the list's `onScrollStop`, never from a timer
   (`HW-02-0121`).
6. **Repaint the highlight through `@ObjectLink`,** not by re-rendering the
   list - flipping `isSelect` on two objects is cheaper than a diff.
7. **Bind the underline width to measured text** with
   `getMeasureUtils().measureText(...)` and `px2vp`, so the tab indicator
   matches the label rather than a guessed constant.

## Verified snippets

All snippets are from `DualListLinkage.zip`. Corrected forms are marked.

**The model - `DualListLinkage.zip#entry/src/main/ets/datas/ListData.ets`** (as shipped)

```typescript
// 左侧列表数据实体
@Observed
export class Level1Data {
  categoryName: Resource | string = '';
  isSelect: boolean = false;          // the highlight, observed through @ObjectLink
  level2List: Level2Data[] = [];
}

// 右侧列表数据实体
@Observed
export class Level2Data {
  serviceIcon: Resource | string = '';
  serviceName: Resource | string = '';
}
```

**Putting `isSelect` on the model is what keeps the two lists cheap.** The
alternative - a `@State selectedIndex` on the page compared against each row's
index - re-evaluates every row on every change. Here two objects change and two
rows repaint.

Neither class carries an id, which is what forces the `ForEach` calls onto the
default index-based key (`HW-02-0124`).

**Both lists, side by side - `DualListLinkage.zip#entry/src/main/ets/pages/DoubleListLinkagePage.ets:97`** (corrected, see `HW-02-0121`, `HW-02-0122`, `HW-02-0123` and `HW-02-0124`)

```typescript
@Builder
TabContentListBuilder() {
  Row() {
    // 左侧一级列表
    List({ scroller: this.level1Scroller }) {
      ForEach(this.mList, (level1data: Level1Data, index: number) => {
        ListItem() {                                   // FIX: sample has the ListItem inside
          LeftListChild({ level1Data: level1data });   //      LeftListChild and onClick on it
        }
        .onClick(() => {
          this.isScrollUpdate = false;                 // suppress the reverse path
          this.mList[this.currentIndex].isSelect = false;
          this.currentIndex = index;
          this.level2Scroller.scrollToIndex(this.currentIndex, true);   // true = smooth
          this.mList[this.currentIndex].isSelect = true;
          // FIX: the sample re-arms with setTimeout(..., 500) - see below
        });
      }, (level1data: Level1Data) => level1data.id);   // FIX: sample supplies no key
    }
    .width(Constant.PERCENT_26)
    .height(Constant.FULL_PERCENT);

    // 右侧二级列表
    List({ scroller: this.level2Scroller }) {
      ForEach(this.mList, (level1data: Level1Data, index: number) => {
        // FIX: the sample uses ListItem { Column { Text; Grid } } - a nested scrollable.
        ListItemGroup({ header: this.categoryHeader(level1data) }) {
          ForEach(level1data.level2List, (level2data: Level2Data) => {
            ListItem() { this.serviceCell(level2data); }
          }, (level2data: Level2Data) => level2data.id);
        }
        .margin({ bottom: index === this.mList.length - 1 ? Constant.NUM_320 : Constant.NUM_0 });
      }, (level1data: Level1Data) => level1data.id);
    }
    .width(Constant.PERCENT_74)
    .height(Constant.FULL_PERCENT)
    .scrollBar(BarState.Off)
    .onScrollStop(() => {
      this.isScrollUpdate = true;                      // FIX: re-arm when the scroll really ends
    })
    .onScrollIndex((start: number) => {
      if (this.isScrollUpdate) {
        this.mList[this.currentIndex].isSelect = false;
        this.currentIndex = start;
        this.mList[this.currentIndex].isSelect = true;
        // the sample also assigns isScrollUpdate = true here, inside the branch
        // that already required it to be true - dead code
      }
    });
  }
}
```

**`onScrollIndex(start)` gives the index of the first item in the viewport,**
which is exactly the section the user is looking at - so the reverse direction
needs no measurement, no offset arithmetic and no mapping. It is the payoff for
making the right pane a list of categories.

**`.margin({ bottom: 320 })` on the last section is load-bearing.** Without it
the final category can never scroll to the top of the viewport, so
`onScrollIndex` never reports its index and tapping it on the left leaves the
highlight where it was.

**The guard flag is right; the timer is not.** `scrollToIndex(i, true)` animates
for a distance-dependent time and reports no completion, so a fixed 500 ms
either re-arms while the list is still moving - and the left selection jumps to
a section the scroll is merely passing - or keeps the reverse path suppressed
after the scroll has settled, ignoring a genuine user scroll. `onScrollStop`
fires exactly when the movement ends.

**The row component - same file, line 320** (corrected, see `HW-02-0122`)

```typescript
@Component
struct LeftListChild {
  @ObjectLink level1Data: Level1Data;

  build() {
    // FIX: the sample opens with ListItem() here. Move it to the call site so the
    //      List's direct child is a ListItem and the handler can sit on it.
    Text(this.level1Data.categoryName)
      .width(Constant.FULL_PERCENT)
      .height($r('app.float.vp_24'))
      .fontSize($r('app.float.fontsize_14'))
      .textAlign(TextAlign.Center)
      .fontColor(this.level1Data.isSelect ?
        $r('app.color.btn_bg_search') : $r('app.color.list_text'))
      .margin({ bottom: $r('app.float.vp_36') });
  }
}
```

**`@ObjectLink` is what makes the highlight a one-row repaint.** The parent
never re-renders either list when a selection changes; the two components whose
`level1Data` was mutated rebuild themselves. That requires `Level1Data` to be
`@Observed`, which it is.

The `List` reference is explicit about the wrapping: "When using custom
components inside `List`, you are advised to wrap the custom component with a
`ListItem` or `ListItemGroup` as the top-level container. Setting attributes or
event methods directly on custom components is not recommended." The right-hand
list in the same file already does it that way.

**Seeding - same file, line 54** (as shipped)

```typescript
aboutToAppear(): void {
  for (let i = 0; i < this.firstTitles.length; i++) {
    let data = new Level1Data();
    let l2List: Level2Data[] = [];
    for (let j = 0; j < this.secondTitles.length; j++) {
      let l2 = new Level2Data();
      l2.serviceIcon = this.secondImages[j];
      l2.serviceName = this.secondTitles[j];
      l2List.push(l2);
    }
    data.categoryName = this.firstTitles[i];
    data.isSelect = (i === 0);             // the first category starts selected
    data.level2List = l2List;
    this.mList.push(data);
  }
}
```

**A fresh `Level2Data` per category, not a shared array.** Every category gets
its own nine objects even though the content is identical - which is what an
`@Observed` model requires, since two categories sharing one object would repaint
together.

Seeding `isSelect` on index 0 matches `currentIndex = 0`; the two must agree or
the first tap clears the wrong row's flag.

**The measured tab underline - same file, line 28 and line 231** (as shipped)

```typescript
private dividerWidth: number = this.getUIContext().px2vp(
  this.getUIContext().getMeasureUtils().measureText({
    textContent: $r('app.string.tab1'),
    fontSize: $r('app.float.fontsize_16')
  }));

// ... in TabBuilder, under each tab label:
Text()
  .width(this.dividerWidth)
  .height(Constant.NUM_2)
  .backgroundColor($r('app.color.btn_bg_search'))
  .borderRadius(Constant.NUM_1)
  .opacity(this.selectedTabIndex === index ? Constant.NUM_1 : Constant.NUM_0);
```

**`measureText` returns px, so `px2vp` is required** - the same unit trap as
`onDetentsDidChange` in `LIFE-14` and the avoid-area heights everywhere in this
industry. Measuring the label rather than guessing a width is the right instinct;
note it measures `tab1` only and applies that width under all three tabs, so
labels of different lengths would all get the first one's underline.

**Toggling opacity rather than mounting and unmounting** keeps the three
underlines in the layout at all times, so the tab row does not reflow when the
selection moves.

**Tab animation state - same file, line 201** (as shipped)

```typescript
.onChange((index: number) => {
  this.currentTabIndex = index;
  this.selectedTabIndex = index;
  if (index !== 0) {
    this.promptAction.showToast({ message: $r('app.string.tip_tab') });
  }
})
.onAnimationStart((index: number, targetIndex: number) => {
  if (index === targetIndex) {
    return;
  }
  this.selectedTabIndex = targetIndex;
});
```

**Two indices, and the difference matters.** `onChange` fires when the swipe
*completes*; `onAnimationStart` fires when it *begins*, with the destination.
Updating `selectedTabIndex` from the latter moves the underline with the swipe
instead of snapping it at the end, while `currentTabIndex` - the one bound to
`Tabs({ index })` - only changes on commit.

## Permissions & config

None. `DualListLinkage.zip#entry/src/main/module.json5` declares no
`requestPermissions` block.

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
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

`EntryAbility` writes `topRectHeight` into `AppStorage`; the page reads it
through `DisplayUtil.getTopRectHeight(uiContext)`, which is a plain
`AppStorage.get` plus `px2vp`. As in `LIFE-15`, that is a snapshot rather than a
binding - `@StorageProp` would be the reactive form.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 44-46).
- The two lists are coupled by **index**, so the right pane must contain exactly
  one section per left-rail row, in the same order. A filtered or reordered right
  pane breaks the linkage silently.
- The last section needs the 320 vp bottom margin to be reachable; that number is
  a constant, not derived from the viewport, so a much taller screen still cannot
  select the final category.
- All eight categories carry the same nine services - the data is generated, not
  loaded - so nothing exercises differing section heights.
- Only the first tab has content; tabs 2 and 3 render a placeholder and toast.
- The rail and pane widths are fixed percentages (26 % / 74 %) with no
  breakpoint handling, on a module that declares phone, tablet and 2in1.
- `scrollToIndex` reports no completion, which is the root of the timing problem
  below.

## Pitfalls

- **`HW-02-0121` - the guard flag is re-armed by a fixed `setTimeout(500)`**
  against a smooth `scrollToIndex` whose duration depends on the distance. Too
  slow and the left selection lands on a section the scroll passed through; too
  fast and a genuine user scroll is ignored. Re-arm from `onScrollStop`. The
  `isScrollUpdate = true` inside the `if (this.isScrollUpdate)` branch is dead.
- **`HW-02-0122` - the left list's direct child is a custom component with
  `.onClick` on it,** and the `ListItem` is hidden inside that component - both
  inverted relative to the `List` reference's advice, and inconsistent with the
  right-hand list in the same file.
- **`HW-02-0123` - each right-pane row wraps an unsized `Grid` inside a
  `ListItem`,** the nested-scrollable case the reference tells you to solve with
  `ListItemGroup` - which is also the natural shape for header-plus-items.
- **`HW-02-0124` - none of the four `ForEach` calls has a key generator,** and
  the default key serialises the `isSelect` flag the linkage toggles. Latent
  today because the repaint comes through `@ObjectLink`, but any real diff
  rebuilds the sections and loses the scroll position the feature depends on.
- **`HW-02-0125` - the back arrow pops the stack and then calls
  `terminateSelf()`,** so the pop is unobservable and the arrow closes the
  application.
- **`HW-02-0126` - the tab `ForEach` declares `item: string` over a
  `Resource[]`.**
- **Do not let the programmatic scroll re-enter the highlight update.** The flag
  is the whole design; what varies is only how you clear it.
- **Do not make the right pane a flat list of services.** The 1:1 index
  correspondence between the two lists is what removes all the mapping code.
- **Do not forget the trailing padding.** Without it the last category is
  unreachable by the linkage, and the bug looks like a scroll problem rather than
  a layout one.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `Scroller.scrollToIndex`, `onScrollIndex`, `onScrollStop`, the Child Components rule for custom components, and the nested-scrollable warning
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-container-listitemgroup.md` - `ListItemGroup` and `header`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-listitemgroup
- `documentation/harmonyos-guides/03_application-framework/arkts-observed-and-objectlink.md` - `@Observed`/`@ObjectLink` for per-row repaint
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-observed-and-objectlink
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - key generation and the "use a unique id" rule
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` - `TabsController.changeIndex`, `onChange`, `onAnimationStart`, `barHeight(0)`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-measureutils.md` - `measureText` and its px return
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-measureutils
- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` - `columnsTemplate` and the scrolling implied by it
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `LIFE-07` - the same industry's other nested-`List` case, with the same `ListItemGroup` remedy
- `LIFE-05` - three linked lists rather than two, with the selection held on the parent instead
- `LIFE-01` - the industry shell this page would sit in
