---
id: COMMON-16
title: TabBar background blur - a frosted bottom tab bar over scrolling content with barOverlap and barBackgroundBlurStyle
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/16_tab_bar_blur.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/tab_bar_blur-0000002257193008
sample: huawei_industry_tree/19_common_technical_solutions/downloads/TabBarBlur.zip
kits: ["@kit.ArkUI"]
apis: [Tabs, TabContent, "Tabs.barOverlap", "Tabs.barBackgroundBlurStyle", "Tabs.barHeight", "Tabs.barPosition", "Tabs.barWidth", "Tabs.scrollable", "Tabs.onContentWillChange", "Tabs.onAnimationStart", BlurStyle, WaterFlow, FlowItem, "WaterFlow.columnsTemplate", "WaterFlow.nestedScroll", NestedScrollMode, Scroll, "Scroll.onDidScroll", "component .safeAreaPadding", "component .opacity", "UIContext.getMeasureUtils", "MeasureUtils.measureText", "UIContext.px2vp", "@StorageLink", "PromptAction.showToast"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0037, HW-19-0038, HW-19-0182]
target_note: "The sample's constants file is named Contants.ets (sic) - see HW-19-0037"
status: verified-with-fixes
---

## When to use

Load this card when the bottom tab bar must sit **on top of** scrolling content
rather than beside it, showing the content blurred through it - the frosted-glass
tab bar. The document's rationale: 给Tab栏（页签栏、导航栏）加上背景模糊的毛玻璃效果后，
可以带来更好的视觉体验 ("adding a frosted-glass background blur to the tab bar
(tab strip, navigation bar) gives a better visual experience").

Two attributes do the whole job. Everything else in the sample - the waterfall
feed, the nested top tab strip, the fading page title - is the content that makes
the effect visible.

## Feature checklist

The application must:

- Overlay the tab bar on the content with `Tabs.barOverlap(true)`.
- Apply a blur style to the bar background with
  `Tabs.barBackgroundBlurStyle(BlurStyle.Thin)`.
- Give the scrollable content enough bottom padding that its last row is not
  permanently hidden behind the overlaid bar.
- Keep the bar clear of the navigation indicator with `safeAreaPadding`.
- Provide custom tab bar builders for both the bottom bar and the nested top
  strip.
- Optionally fade the page title as the content scrolls (clamped - see
  HW-19-0038).

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup, publishes `topRectHeight` / `bottomRectHeight` |
| `common/Contants.ets` (sic - HW-19-0037) | `EXPLORE_TAB_INDEX = 0`, `BOTTOM_TAB_BAR_HEIGHT = 56`, `TITLE_HEIGHT = 20`, `INDICATOR_PADDING_WIDTH = 28` |
| `model/TabIcon.ets` | `TabIcon` and `BOTTOM_TABS` - the bottom bar items |
| `model/CardInfo.ets` | `CardInfo`, `CategoryInfo`, `CARD_INFOS`, `TOP_TAB_BAR_INFO` - the feed data and the top tab categories |
| `components/WaterFlowContentList.ets` | the `WaterFlow` feed for one category |
| `components/WaterFlowCard.ets` | one feed card |
| `utils/MeasureUtils.ets` | `measureText(context, text, fontSize)` - text width in vp, used to size the top tab underline |
| `pages/HomePage.ets` | the whole screen: outer `Tabs` (bottom bar, blurred) containing a `Scroll` with the title and an inner `Tabs` (top strip) of `WaterFlow` feeds |

**Nesting.** Outer `Tabs` (bottom bar, `BarPosition.End`) -> `TabContent` ->
`Scroll` -> `Column` { title, inner `Tabs` (top strip) -> `TabContent` ->
`WaterFlowContentList` }. The inner `WaterFlow` sets
`nestedScroll({ scrollForward: PARENT_FIRST, scrollBackward: SELF_FIRST })` so
that scrolling down first moves the outer `Scroll` (which fades the title away)
and only then the feed, while scrolling up moves the feed first.

**Why `barOverlap` needs compensating padding.** With `barOverlap(true)` the bar
no longer occupies layout space, so the content extends underneath it. The sample
compensates in two places: `safeAreaPadding({ bottom: px2vp(bottomRectHeight) })`
on the `Tabs` keeps the bar off the navigation indicator, and the last
`FlowItem` gets
`BOTTOM_TAB_BAR_HEIGHT + px2vp(bottomRectHeight)` of bottom padding so the final
card can be scrolled clear of the bar.

**The tab-switch guard.** `onContentWillChange` returns `false` for every index
above 0, showing a "demo only" toast, so only the first tab is functional. This
is how the sample keeps itself to one implemented screen - it is not part of the
blur technique.

## Implementation steps

1. **Build the `Tabs` with a custom bar.** `barPosition(BarPosition.End)`,
   `barHeight(BOTTOM_TAB_BAR_HEIGHT)`, and a `@Builder` per tab bound with
   `.tabBar(...)`.
2. **Turn on the overlay:** `.barOverlap(true)`. The document is explicit that
   this is step 1 - the blur has nothing to blur until the bar overlaps the
   content.
3. **Choose the blur:** `.barBackgroundBlurStyle(BlurStyle.Thin)`. Heavier
   `BlurStyle` values give a stronger frost.
4. **Reserve room at the end of the content** equal to the bar height plus the
   navigation-indicator inset, otherwise the last item is unreachable.
5. **Keep the bar out of the system inset** with
   `.safeAreaPadding({ bottom: px2vp(bottomRectHeight) })`.
6. **Disable content swiping if the bar is the only navigation**
   (`.scrollable(false)`), and gate switching in `onContentWillChange` when some
   tabs are placeholders.
7. **For the nested top strip**, size the selected-state underline from the
   measured text width:
   `measureText(uiContext, name, fontSize) + INDICATOR_PADDING_WIDTH`, and track
   the selected index from `onAnimationStart`'s `targetIndex`.
8. **If you fade the title on scroll, derive the opacity from the absolute scroll
   offset and clamp it to [0, 1]** rather than accumulating deltas
   (HW-19-0038).

## Verified snippets

All snippets below come from the sample project, not from the document.

### The blurred, overlapping tab bar

`TabBarBlur.zip#TabBarBlur/entry/src/main/ets/pages/HomePage.ets`

```ts
Tabs() {
  ForEach(BOTTOM_TABS, (tab: TabIcon, index: number) => {
    TabContent() {
      if (index === EXPLORE_TAB_INDEX) {
        Scroll() {
          Column() {
            this.headerBuilder();
            Tabs() {
              ForEach(TOP_TAB_BAR_INFO, (tabInfo: CategoryInfo, index: number) => {
                TabContent() {
                  WaterFlowContentList({ cards: tabInfo.cards });
                }
                .tabBar(this.topTabBuilder(index, tabInfo.tabName));
              });
            }
            .barWidth('90%')
            .onAnimationStart((index: number, targetIndex: number) => {
              this.topTabIndex = targetIndex;
            });
          }
          .safeAreaPadding({ top: this.getUIContext().px2vp(this.topRectHeight) })
          .width('100%');
        }
        .scrollBar(BarState.Off)
        .onDidScroll((xOffset: number, yOffset: number) => {
          this.titleOpacity -= yOffset / TITLE_HEIGHT;   // FIX (HW-19-0038): clamp, derive from offset
        });
      }
    }
    .tabBar(this.bottomTabBuilder(tab, index));
  });
}
.onContentWillChange((currentIndex: number, comingIndex: number) => {
  if (comingIndex > 0) {
    this.getUIContext().getPromptAction().showToast({ message: $r('app.string.toast_demo') });
    return false;
  } else {
    this.bottomTabIndex = comingIndex;
    return true;
  }
})
.scrollable(false)
.safeAreaPadding({ bottom: this.getUIContext().px2vp(this.bottomRectHeight) })
.barHeight(BOTTOM_TAB_BAR_HEIGHT)
.barPosition(BarPosition.End)
.barOverlap(true)
.barBackgroundBlurStyle(BlurStyle.Thin);
```

### Reserving room under the overlaid bar

`TabBarBlur.zip#TabBarBlur/entry/src/main/ets/components/WaterFlowContentList.ets`

```ts
import { BOTTOM_TAB_BAR_HEIGHT } from '../common/Contants';

@Component
export struct WaterFlowContentList {
  @State cards: CardInfo[] = [];
  @StorageLink('bottomRectHeight') bottomRectHeight: number = 0;

  build() {
    WaterFlow() {
      ForEach(this.cards, (card: CardInfo, index: number) => {
        FlowItem() {
          WaterFlowCard({ info: this.cards[index] });
        }
        .padding({
          bottom: index === this.cards.length - 1 ?
            BOTTOM_TAB_BAR_HEIGHT + this.getUIContext().px2vp(this.bottomRectHeight) : 0
        });
      }, (card: CardInfo) => JSON.stringify(card));
    }
    .columnsTemplate('1fr 1fr')
    .columnsGap($r('app.integer.flow_list_gap'))
    .rowsGap($r('app.integer.flow_list_gap'))
    .padding({ left: $r('app.integer.spaces_16'), right: $r('app.integer.spaces_16') })
    .nestedScroll({
      scrollForward: NestedScrollMode.PARENT_FIRST,
      scrollBackward: NestedScrollMode.SELF_FIRST
    });
  }
}
```

### Underline sized to the measured tab text

`TabBarBlur.zip#TabBarBlur/entry/src/main/ets/utils/MeasureUtils.ets`

```ts
export function measureText(context: UIContext, text: ResourceStr, fontSize: number | string | Resource): number {
  return context.px2vp(context.getMeasureUtils().measureText({
    textContent: text,
    fontSize: fontSize
  }));
}
```

`TabBarBlur.zip#TabBarBlur/entry/src/main/ets/pages/HomePage.ets`

```ts
@Builder
topTabBuilder(index: number, name: ResourceStr) {
  Column() {
    Text(name)
      .fontColor(this.topTabIndex === index ? $r('app.color.tab_selected_color') : $r('app.color.top_tab_unselected'))
      .fontSize($r('app.integer.top_tab_font_size'));
    Row()
      .height($r('app.integer.top_tab_indicator_height'))
      .width(measureText(this.getUIContext(), name, $r('app.integer.top_tab_font_size')) + INDICATOR_PADDING_WIDTH)
      .borderRadius($r('app.integer.top_tab_indicator_border_radius'))
      .backgroundColor($r('app.color.tab_selected_color'))
      .margin({ top: $r('app.integer.spaces_8') })
      .opacity(this.topTabIndex === index ? 1 : 0);
  }
  .width('100%')
  .height('100%')
  .padding({ bottom: $r('app.integer.spaces_8') })
  .justifyContent(FlexAlign.End);
}
```

### Layout constants

`TabBarBlur.zip#TabBarBlur/entry/src/main/ets/common/Contants.ets`

```ts
export const EXPLORE_TAB_INDEX = 0;
export const BOTTOM_TAB_BAR_HEIGHT = 56;
export const TITLE_HEIGHT = 20;
export const INDICATOR_PADDING_WIDTH = 28;
```

## Permissions & config

**No permissions are required** and none are declared - the feature is entirely a
rendering attribute.

`TabBarBlur.zip#TabBarBlur/entry/src/main/module.json5` declares
`"deviceTypes": ["phone", "tablet", "2in1"]` and the usual single
`EntryAbility` with the home skill; there is no `requestPermissions` block, no
`routerMap`, and no extension ability.

The project's `build-profile.json5` enables
`"strictMode": { "caseSensitiveCheck": true }`, which is worth noting alongside
the misspelt constants filename (HW-19-0037).

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later. `barBackgroundBlurStyle` is an
  API 11 attribute and `barOverlap` an API 10 attribute.
- **`barBackgroundBlurStyle` needs `barOverlap(true)`.** Without the overlap the
  bar has nothing behind it to blur; the document lists the two as ordered steps
  for that reason.
- **The overlay costs layout space.** Content under the bar is reachable only if
  the scrollable area adds bar-height padding at the end, as the sample does on
  the last `FlowItem`.
- **`opacity` clamps the rendered value only**: "Values less than 0 are treated as
  0. Values greater than 1 are treated as 1." A state variable driving it can
  still drift out of range (HW-19-0038).
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **The documented tree says `common/Constants.ets`, but the sample ships
  `common/Contants.ets`.** Both imports in the project use the misspelt name, so
  recreating the tree exactly as documented breaks the build. (HW-19-0037)
- **`this.titleOpacity -= yOffset / TITLE_HEIGHT` is incorrect.** It integrates
  scroll deltas into an unbounded value; the rendered opacity is clamped but the
  stored one is not, and with `PARENT_FIRST`/`SELF_FIRST` nested scrolling the
  up and down deltas do not cancel, so the title can stay invisible at the top of
  the page. Derive the opacity from the current scroll offset and clamp it.
  (HW-19-0038)
- **`onContentWillChange` returning `false` is not an error path.** The sample
  uses it to keep every tab except the first as a demo placeholder; remove that
  guard before reusing the shell.
- **`ForEach` with `JSON.stringify(card)` as key generator** is workable for this
  fixed-size demo feed, but a real waterfall feed should use `LazyForEach` with a
  stable id - see the performance practice in COMMON-04.
- **`@State cards: CardInfo[] = []` is initialised from the parent.** For a value
  the child never mutates, `@Prop` states the intent better.

## References

- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-container-tabs -
  `Tabs`, `barOverlap`, `barBackgroundBlurStyle`, `barHeight`, `barPosition`,
  `onContentWillChange`, `onAnimationStart`.
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-opacity.md` -
  `opacity` value range and the clamping rule.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-opacity
- `documentation/harmonyos-guides/03_application-framework/arkts-layout-development-create-waterflow.md` -
  `WaterFlow` / `FlowItem` and `nestedScroll`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-layout-development-create-waterflow
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-tabs.md` -
  tab-based navigation layout guidance.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-tabs
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/tab_bar_blur-0000002257193008
