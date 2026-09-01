---
id: NEWS-11
title: Hot-search board - a custom Tabs bar kept in step with the content by two indices
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/11_hot_search.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/hot_search-0000002258742170
sample: huawei_industry_tree/11_news_reading/downloads/HotSearch.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [hilog, window, Tabs, TabContent, tabBar, TabsController, onChange, onAnimationStart, BarMode, BarPosition, Search, CancelButtonStyle, Divider, Stack, Flex, ForEach, "window.getWindowAvoidArea", "window.on('avoidAreaChange')", setWindowLayoutFullScreen]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-11-0031, HW-11-0040, HW-11-0041, HW-11-0042]
status: verified
---

## When to use

**Load this card when a page needs a tab bar the framework's default bar
cannot draw** - a coloured underline that is narrower than the tab, a badge, an
icon that changes with selection, a pill. The moment `tabBar` takes a
`@Builder` instead of a string, `Tabs` stops managing the bar's appearance and
the app owns it, which introduces a synchronisation problem this sample solves
correctly.

The concrete feature is a hot-search board: a search field, a row of recently
used apps, and a `Tabs` of daily rankings where each row is rank badge, headline,
heat count and a hot/new tag. The same shape covers league tables, trending
topics, top-sellers, "most read this week" - anything that is a ranked list
sliced by a small set of categories.

The transferable core is the **two-index pattern**: one index drives the
`TabContent`, a second drives the custom bar, and they are updated from two
different events. It is the piece of this card worth carrying to any custom
`tabBar`, and it is documented behaviour rather than a trick.

## Feature checklist

- A search field with a persistent cancel button, plus a round scan button
  beside it.
- A white rounded card of five recently used apps, each an icon over a label.
- Four ranking tabs (热搜榜单 / XX新闻 / XX热搜 / 同城) rendered by a custom
  builder, laid out fixed-width across the bar.
- The selected tab's label turns blue and a 90%-wide 2vp underline appears
  beneath it; the others keep a hairline divider.
- Each tab shows a title and seven ranked rows.
- Ranks 1-3 get a taller badge image than ranks 4-7.
- A headline longer than the row ellipsises on one line; the heat count and the
  hot/new tag stay put.
- Swiping the content moves the bar highlight as the swipe animation *starts*,
  not when it ends.

## Architecture

One `entry` module, four ArkUI files. Everything visible is in one page; the
data is two constant files.

```
entry/src/main/ets
├── constants/AppsConstant.ets     APP_DATA: five {name, src} app shortcuts
├── constants/NewsConstant.ets     NEWS_DATA: four tabs; MESSAGES: seven ranked rows
├── entryability/EntryAbility.ets  full screen + avoid areas -> AppStorage
├── entrybackupability/
└── pages/Index.ets                @Entry: search row, app card, Tabs, both @Builders
```

The documented tree matches the zip (the doc writes `resource` where the
directory is `resources`, which is the doc's own shorthand for the resource
root and appears in several documents in this industry).

**The design decision worth copying** is the split between `currentIndex` and
`selectedIndex`. `currentIndex` is bound to the `Tabs`' `index` property and
updated in `onChange`; `selectedIndex` is what `tabBuilder` compares against
and is updated in **`onAnimationStart`**. The reference states the reason
plainly: with a custom bar, "relying solely on the `onChange` event for
synchronization between tabs and swipe gestures may result in delayed visual
updates, since it is triggered after the swipe-triggered tab switching
animation is completed. For smooth animations, listen for the active tab index
in `onAnimationStart`."

In practice: swipe the content and the underline slides to the new tab as the
animation begins, rather than snapping into place 400 ms later when it ends.
`onChange` still runs and sets both indices, which covers the tap path and
keeps the two in agreement afterwards. The `if (index === targetIndex) return;`
guard in `onAnimationStart` skips the no-op case where a gesture settles back
on the tab it started from.

## Implementation steps

1. **Keep two indices.** Bind `index: this.currentIndex` on the `Tabs`; compare
   `this.selectedIndex` inside the bar builder.
2. **Update `selectedIndex` in `onAnimationStart`** (guarded against
   `index === targetIndex`) and both indices in `onChange`.
3. **Build the bar item as a `Column` of label + underline**, and give the
   `Column` `width('100%')` so the item fills the slot the bar allocates it.
4. **Draw the underline as two stacked `Divider`s** and toggle the coloured one
   with `opacity`, not with `if`. An `if` changes the node tree and the layout
   height; `opacity` does not, so the label never shifts by a pixel when
   selection moves.
5. **Use `BarMode.Fixed`** so the four tabs divide the bar width equally - the
   right mode for a small, known set of categories.
6. **Give the ranked row one `Flex` with `SpaceBetween`,** headline
   `maxLines(1)` + `TextOverflow.Ellipsis`, so a long headline truncates
   instead of pushing the heat count off the row.
7. **Size the rank badge from the row index,** not from the data.
8. **Consume the avoid areas the ability publishes** - this sample computes
   them and then pads with a literal instead (see Pitfalls).

## Verified snippets

All snippets are from `HotSearch.zip`. All are as shipped - the sample has no
open findings.

**The custom tab item — `entry/src/main/ets/pages/Index.ets`** (as shipped)

```typescript
@State currentIndex: number = 0;
@State selectedIndex: number = 0;
private fontColor: string = '#777777';
private selectedFontColor: string = '#0067D1';
private controller: TabsController = new TabsController();

@Builder
tabBuilder(index: number, name: string) {
  Column() {
    Text(name)
      .fontColor(this.selectedIndex === index ? this.selectedFontColor : this.fontColor)
      .fontSize(16)
      .fontWeight(400)
      .margin({ top: 12, bottom: 10 });
    Stack() {
      Divider();                                   // the hairline every tab shows
      Divider()
        .width('90%')
        .strokeWidth(2)
        .borderRadius(1)
        .color('#0067D1')
        .opacity(this.selectedIndex === index ? 1 : 0);   // the highlight, always present
    };
  }
  .width('100%');
}
```

**The second `Divider` is always in the tree; only its opacity moves.** That is
the whole trick of a stable custom bar. Written as
`if (this.selectedIndex === index) { Divider()... }` the node would be created
and destroyed on every switch, the `Stack`'s measured height would change with
it, and the label above would jitter. With `opacity` the geometry is fixed at
first layout and selection is a pure paint change - which also means it can be
animated for free by wrapping the assignment in `animateTo`.

`Stack` rather than a single `Divider` because two different underlines are
wanted at once: a full-width hairline that separates the bar from the content,
and a shorter, rounder, coloured one that marks selection. Stacking them lets
the coloured one be 90% wide and 2vp thick without disturbing the hairline.

`.width('100%')` on the `Column` is required: without it the builder's content
sizes to the label, and the tap target shrinks to the text.

**Wiring the two indices — same file** (as shipped)

```typescript
Tabs({ barPosition: BarPosition.Start, index: this.currentIndex, controller: this.controller }) {
  ForEach(newsArray, (item: number) => {
    TabContent() {
      Column() {
        Text(NEWS_DATA[item].title)
          .fontWeight(500)
          .fontSize(18)
          .width('100%');
        ForEach(msgsArray, (item: number) => {
          this.textColumn(MESSAGES[item].num, MESSAGES[item].msg, MESSAGES[item].million, MESSAGES[item].img,
            item);
        });
      }
      .padding({ left: 17, right: 17, top: 17 })
      .height(375)
      .backgroundColor(Color.White)
      .borderRadius(16);
    }.tabBar(this.tabBuilder(item, NEWS_DATA[item].name));
  });
}
.vertical(false)
.barMode(BarMode.Fixed)
.barWidth(360)
.barHeight(56)
.animationDuration(400)
.onChange((index: number) => {
  // currentIndex drives which TabContent is shown
  this.currentIndex = index;
  this.selectedIndex = index;
})
.onAnimationStart((index: number, targetIndex: number) => {
  if (index === targetIndex) {
    return;
  }
  // selectedIndex drives the colour and the underline inside the custom bar
  this.selectedIndex = targetIndex;
})
.height(455);
```

**`onAnimationStart` fires with the destination before the animation runs;
`onChange` fires with it after.** For a default bar that difference is
invisible because the framework animates the bar itself. For a custom bar
driven by app state, using only `onChange` means a 400 ms window (the
`animationDuration` set right above) where the content has moved and the
underline has not. Setting `selectedIndex` from `onAnimationStart` closes it.

The inner `ForEach` shadows `item` from the outer one, so
`NEWS_DATA[item].title` reads the tab while `MESSAGES[item]` reads the row -
correct, but only because the shadowing happens to be intended. Rename one of
them.

**The ranked row — same file** (as shipped)

```typescript
@Builder
textColumn(src: Resource, msg: string, million: string, img: Resource, n: number) {
  Flex({ direction: FlexDirection.Row, justifyContent: FlexAlign.SpaceBetween }) {
    Row() {
      Row() {
        Image(src)
          .size(n <= 2 ? { height: 22, width: 15 } : { height: 12 });   // top 3 get a bigger badge
      }
      .width(16)
      .justifyContent(FlexAlign.Center);

      Text(msg)
        .fontSize(14)
        .width(180)
        .margin({ left: 13 })
        .textOverflow({ overflow: TextOverflow.Ellipsis })
        .maxLines(1);

      Text(million)                                                     // heat, e.g. 661万
        .fontColor('#999999')
        .fontSize(12)
        .margin({ left: 20 });
    };

    Image(img)                                                          // 热 / 新 tag
      .size({ height: 18 });
  }
  .size({ width: '100%', height: 24 })
  .margin({ top: 20 });
}
```

**The fixed 16vp badge box is what keeps seven rows aligned** despite the
badges for ranks 1-3 being a different size from 4-7. The `Image` changes
dimensions inside a `Row` of constant width, centred, so the headline always
starts at the same x. Sizing the outer box instead of the image is the general
form of this: let the variable-size thing float inside a fixed slot.

`million` is a pre-formatted string (`'661万'` - 6.61 million), not a number.
That is a shortcut worth noticing rather than copying: 万 (ten thousand) and
亿 (hundred million) grouping is locale-specific, so a real board formats a
number at render time instead of shipping formatted strings in the data.

**Publishing the avoid areas — `entry/src/main/ets/entryability/EntryAbility.ets`** (as shipped)

```typescript
windowStage.loadContent('pages/Index', (err) => {
  if (err.code) { /* ... */ return; }
  let windowClass: window.Window = windowStage.getMainWindowSync();
  windowClass.setWindowLayoutFullScreen(true).then(() => { /* ... */ });

  let type = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR;
  let avoidArea = windowClass.getWindowAvoidArea(type);
  AppStorage.setOrCreate('bottomRectHeight', avoidArea.bottomRect.height);

  type = window.AvoidAreaType.TYPE_SYSTEM;
  avoidArea = windowClass.getWindowAvoidArea(type);
  AppStorage.setOrCreate('topRectHeight', avoidArea.topRect.height);

  windowClass.on('avoidAreaChange', (data) => {
    if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
      AppStorage.setOrCreate('topRectHeight', data.area.topRect.height);
    } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
      AppStorage.setOrCreate('bottomRectHeight', data.area.bottomRect.height);
    }
  });
});
```

This is the industry-standard boilerplate and it is correct as far as it goes:
read both avoid areas once, then keep them current from `avoidAreaChange` -
necessary because the status-bar height changes on a call banner and the
navigation indicator disappears in gesture-free modes.

Two caveats. `Index.ets` never reads either key: it clears the status bar with
`.margin({ top: 50 })`, a literal that is right on one device class and wrong
on the rest. The correct consumer is
`@StorageProp('topRectHeight')` plus `px2vp` at the point of use, as `NEWS-10`'s
`HomePage` does. And nothing calls `windowClass.off('avoidAreaChange')` in
`onWindowStageDestroy` - the same boilerplate omission filed as `HW-09-0021`
in the tourism industry.

## Permissions & config

**None.** The sample declares no `requestPermissions` - it has no network, no
storage and no device access. Everything on screen is a bundled resource or a
constant.

`deviceTypes` is `phone`, `tablet`, `2in1`. `EntryAbility.onCreate` forces
`COLOR_MODE_LIGHT`, which is consistent with the page's hardcoded
`#F1F3F5` / `Color.White` palette but means the board ignores the system dark
theme.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The layout is fixed-size on several axes: `.barWidth(360)`, `.height(455)` on
  the `Tabs`, `.height(375)` on each `TabContent`, `.width(180)` on the
  headline, `.margin({ top: 50 })` on the search row. On a tablet or a resized
  2in1 window the board neither grows nor reflows.
- `BarMode.Fixed` divides the bar equally among the tabs; adding a fifth
  category shrinks all of them. Switch to `BarMode.Scrollable` past four or
  five.
- All four tabs render the same seven `MESSAGES` rows - only the title above
  them differs. `News.news` (the per-tab headline arrays in `NewsConstant.ets`)
  is declared and never read, so wiring the tabs to real per-tab data means
  replacing the inner `ForEach` source.
- The `Search` field has no `onSubmit`/`onChange`; it accepts input and does
  nothing with it. For a working search history see `NEWS-22`.
- The app shortcuts row is a static `Row({ space: 28 })` of five items - no
  wrap, no overflow handling.

## Pitfalls

**No HW findings were raised against this document or sample.** It is the
cleanest of the four cards in this batch. The observations below are unfiled
and are quality notes, not defects:

- **The avoid-area values are computed and unused.** `EntryAbility` publishes
  `topRectHeight`/`bottomRectHeight` into `AppStorage`; `Index.ets` pads with a
  literal `top: 50` instead. Read them with `@StorageProp` and `px2vp`.
- **`avoidAreaChange` is registered and never released.** Add
  `windowClass.off('avoidAreaChange')` in `onWindowStageDestroy`. Same shape as
  `HW-09-0021`.
- **`item` is shadowed between the outer and inner `ForEach`.** It works, but
  the reader has to prove it does.
- **Neither `ForEach` supplies a key generator,** so ArkUI falls back to the
  default. With static arrays of indices that is harmless; with real data it is
  not.
- **The tab arrays are module-level `let`s** (`appArray`, `newsArray`,
  `msgsArray`) holding index ranges that duplicate the lengths of the constant
  arrays. `ForEach(NEWS_DATA, ...)` removes the possibility of them disagreeing.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` -
  `tabBar` builders, `BarMode`, and the `onChange` vs `onAnimationStart`
  synchronisation note
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-search.md` -
  `Search`, `cancelButton`, `CancelButtonStyle`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-search
- `documentation/harmonyos-guides/03_application-framework/window-overview.md` -
  `setWindowLayoutFullScreen` and avoid areas
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/window-overview
- `NEWS-04` - the channel picker, the other custom-`Tabs` sample in this
  industry
- `NEWS-22` - a search field that actually keeps history
