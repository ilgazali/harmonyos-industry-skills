---
id: SHOP-10
title: Sticky mall home page - Refresh over Scroll over Tabs, with a custom tab bar and a measured indicator
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/10_scroll_celling_demo.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/scroll_celling_demo-0000002331639969
sample: huawei_industry_tree/16_shopping/downloads/ScrollCellingDemo.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit"]
apis: [Refresh, pullToRefresh, onRefreshing, Scroll, Scroller, Tabs, TabsController, TabContent, onAnimationStart, barHeight, nestedScroll, NestedScrollMode, onReachEnd, LazyForEach, IDataSource, DataChangeListener, Grid, rowsTemplate, "componentUtils.getRectangleById", "UIContext.animateTo", onAreaChange, onWillScroll, "@Watch", "@ComponentV2", "@Param", "@Local", expandSafeArea]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-16-0011, HW-16-0013]
status: verified
---

## When to use

Load this card when you need **a shopping home page whose category tabs stick to
the top while the page scrolls** - the layout every marketplace app opens on: a
search bar, a grid of entry points, then a horizontally scrollable filter bar
that pins under the search bar once you have scrolled past the grid, with an
infinite product list beneath it.

The mechanism is a nesting order plus one scroll-coordination rule. `Refresh`
wraps everything so a pull anywhere refreshes. Inside it a `Scroll` owns the
vertical travel of the header block. Inside that, each tab's product `List`
declares `nestedScroll` with `PARENT_FIRST` forward and `SELF_FIRST` backward -
which is the whole ceiling effect. Scrolling down, the outer `Scroll` consumes
first until the header is gone and the tab bar is at the top; only then does the
list scroll. Scrolling back up, the list returns to its own top before the
header comes back.

That inversion - parent first one way, self first the other - is the
generalisable part. It is the same configuration for a profile page with a
pinned segmented control, a news feed under a category strip, or any
"header + sticky bar + inner list" screen. The custom tab bar with a measured
sliding indicator is the second reusable piece, and it is the one the sample
gets more interesting: the indicator position is not computed from a formula but
read back from the laid-out component.

## Feature checklist

- Pull down anywhere on the page to refresh; the built-in `Refresh` spinner
  shows and clears after one second.
- A search row with a scan icon, a `Search` field carrying a camera icon
  overlaid at its right edge, and a 搜索 button.
- A horizontally scrolling category grid (排行榜 / 补贴 / 优惠券 / 签到 / 换肤)
  in one row.
- A horizontally scrolling filter bar of seven tabs with a blue underline that
  slides to the selected tab over 300 ms.
- Tapping a tab switches the product pane; swiping the product pane moves the
  underline and scrolls the tab bar so the active tab is centred.
- Dragging the tab bar sideways drags the underline with it, one-to-one.
- Each tab shows a lazily rendered product list; reaching the bottom shows a
  spinner and appends another page after one second.
- The header scrolls away and the tab bar stops at the top; scrolling back up
  returns the list to its top before the header reappears.

## Architecture

One `entry` module. Four view components, a two-file data layer, one page.

```
entry/src/main/ets
├── component
│  ├── CatalogView.ets        @ComponentV2, a single-row Grid of 5 entry points
│  ├── CustomTabBarView.ets   the filter bar + the measured sliding indicator
│  ├── ProductView.ets        @ComponentV2, one tab's LazyForEach list + load-more
│  └── SearchView.ets         search row; camera icon via overlay()
├── entryability/EntryAbility.ets
├── model
│  ├── DataSource.ets         BasicDataSource<T> implements IDataSource
│  └── ProductDataSource.ets  ProductDataSource + ProductDataModel + PRODUCT_DATA (4 rows)
└── pages/MallMainPage.ets    @Entry, the Refresh/Scroll/Tabs nesting - 94 lines
```

The documented tree matches the zip. The **doc's own snippet does not**: it
labels the page `MainMallPage.ets` while both the tree in the same document and
the zip have `MallMainPage.ets` (`HW-16-0011`). The same snippet is also cut
mid-construct - the `ForEach` key generator ends at a comment with no closing
parenthesis and no closing brace for `TabContent` (`HW-16-0013`). Take the code
from the zip.

**The design decision worth copying** is that the tab bar and the `Tabs` are
separate components sharing two handles: a `@Link`ed index and a
`TabsController`. `Tabs` runs with `.barHeight(0)` - its own bar is switched off
entirely - and `CustomTabBarView` is placed above it as an ordinary sibling.
Selection flows both ways over those two handles: a tap on a bar item calls
`tabsController.changeIndex(i)` and writes the `@Link`; a swipe on the content
fires `onAnimationStart(index, targetIndex)` which writes the same `@Link`, and
the bar's `@Watch` reacts by animating the underline. Neither component knows
how the other is built.

This is worth copying because `Tabs`' built-in `tabBar` builder cannot do what
this bar does - horizontal scrolling of the bar itself, an underline that tracks
finger drags, centring the active tab. Turning the built-in bar off and driving
`Tabs` purely as a controllable content pager is the clean way out, and it costs
exactly one `@Link` and one controller.

## Implementation steps

1. **Nest in this order:** `Refresh` → `Column` → (`SearchView`, `Scroll`) →
   `Column` → (`CatalogView`, `Column`(`CustomTabBarView`, `Tabs`)). The search
   row is outside the `Scroll` so it never scrolls away; the category grid is
   inside it so it does.
2. **Bind refresh two-way** with `Refresh({ refreshing: $$this.isRefreshing })`
   and clear the flag from `onRefreshing`. `$$` is required - without it the
   spinner never retracts.
3. **Set `.barHeight(0)` on `Tabs`** so the built-in bar is gone and the custom
   one is the only bar.
4. **Share one index as `@Link @Watch('change')`** in the bar, and one
   `TabsController` passed in as a plain property.
5. **Give each inner `List` the inverted `nestedScroll` pair** -
   `scrollForward: NestedScrollMode.PARENT_FIRST`,
   `scrollBackward: NestedScrollMode.SELF_FIRST`. This is the ceiling effect;
   nothing else implements it.
6. **Measure the indicator, do not compute it.** Tag each bar label with
   `.id('bar' + index)`, then read its window offset and width back with
   `componentUtils.getRectangleById(id)` and convert with `px2vp`.
7. **Seed the indicator from `onAreaChange`** on the initially selected label,
   guarded by `indicatorLeftMargin === 0 || indicatorWidth === 0` so it fires
   once - `getRectangleById` returns nothing useful before first layout.
8. **Move the indicator inside an `animateTo` closure** (300 ms, `Curve.Linear`)
   so both the margin and the width interpolate together.
9. **Track bar drags with `onWillScroll`,** subtracting the reported `xOffset`
   from the indicator margin so the underline stays glued to its label while the
   bar is dragged.
10. **Serve products from an `IDataSource`,** and append on `onReachEnd` with a
    `LoadingProgress` footer whose `visibility` is driven by the loading flag.

## Verified snippets

All snippets are from `ScrollCellingDemo.zip`. The doc's version of the first one
is mislabelled and unbalanced (`HW-16-0011`, `HW-16-0013`); this is the shipped
code.

**The nesting — `entry/src/main/ets/pages/MallMainPage.ets`** (as shipped, trimmed)

```typescript
build() {
  Refresh({ refreshing: $$this.isRefreshing }) {
    Column() {
      SearchView()
        .margin({ top: 8 });

      Scroll(this.scrollController) {
        Column() {
          CatalogView()
            .height(100)
            .margin({ left: 6, right: 6, top: 4 });

          Column() {
            CustomTabBarView({ tabBarIndex: this.curTabBarIndex, tabsController: this.tabsController })
              .width('100%')
              .margin({ left: 16, right: 16, bottom: 4 });

            Tabs({ barPosition: BarPosition.Start, controller: this.tabsController }) {
              ForEach(this.tabArray, (item: Resource) => {
                TabContent() {
                  ProductView({ filterParam: item });
                }
                .backgroundColor('#F1F3F5');
              }, (item: Resource, index: number) =>
              JSON.stringify(this.context.resourceManager.getStringSync(item.id) + index));
            }
            .width('100%')
            .barHeight(0)
            .animationDuration(300)
            .onAnimationStart((index: number, targetIndex: number) => {
              this.curTabBarIndex = targetIndex;
              this.listScroller.scrollToIndex(targetIndex, true, ScrollAlign.CENTER);
            });
          }
          .height('100%');
        };
      }
      .width('100%')
      .height('100%')
      .scrollBar(BarState.Off);
    };
  }
  .expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.BOTTOM, SafeAreaEdge.TOP])
  .pullToRefresh(true)
  .onRefreshing(() => {
    setTimeout(() => {
      this.isRefreshing = false;
    }, 1000);
  })
  .padding({ left: 10, right: 10 })
  .backgroundColor('#F1F3F5');
}
```

**Three details carry the sticky behaviour.** The inner `Column` holding the bar
and the `Tabs` is `.height('100%')` - a full viewport tall - so the `Scroll`'s
content is exactly "grid height + one screen", meaning the scroll ends precisely
when the tab bar reaches the top. That is the ceiling: there is nothing left to
scroll, so the bar cannot go further. `onAnimationStart` (not `onChange`) writes
the index, so the bar reacts as the swipe animation begins rather than when it
lands. And `barHeight(0)` is what makes the whole arrangement legal - without it
you would see two bars.

`.height('100%')` also appears twice on the `Refresh` and twice on nothing else;
the duplicate on `Refresh` is harmless.

Note that `this.listScroller` in this page is a `Scroller` that is never attached
to any scrollable component - `CustomTabBarView` builds its own private
`listScroller` for its `List`. So the `scrollToIndex` call on swipe is a no-op:
swiping the content moves the underline but does not centre the tab in the bar.
Passing the bar's scroller down (or moving the centring call into the bar's
`@Watch`) is the fix.

**The ceiling contract — `entry/src/main/ets/component/ProductView.ets`** (as shipped, trimmed)

```typescript
@ComponentV2
export struct ProductView {
  @Param filterParam: ResourceStr = '';
  @Local isLoading: boolean = false;
  private productData: ProductDataSource = new ProductDataSource();

  build() {
    List() {
      LazyForEach(this.productData, (item: ProductDataModel, index: number) => {
        ListItem() {
          this.ProductItem(item, index);
        };
      }, (item: ProductDataModel, index: number) => item.id + '' + index);

      ListItem() {
        this.footer();
      };
    }
    .nestedScroll({
      scrollForward: NestedScrollMode.PARENT_FIRST,  // 父组件先滚动，滚动到边缘后自身滚动
      scrollBackward: NestedScrollMode.SELF_FIRST    // 自身先滚动，滚动到边缘后父组件滚动
    })
    .scrollBar(BarState.Off)
    .height('100%')
    .edgeEffect(EdgeEffect.Fade)
    .onReachEnd(() => {
      this.isLoading = true;
      setTimeout(() => {
        this.productData.pushData(PRODUCT_DATA);
        this.isLoading = false;
      }, 1000);
    });
  }
}
```

**`nestedScroll` is the whole ceiling feature.** `PARENT_FIRST` on the forward
(downward) direction hands the gesture to the outer `Scroll` until it is at its
end - which is exactly when the tab bar has reached the top - and only then does
the list start moving. `SELF_FIRST` backward means an upward flick returns the
list to its own top before the outer scroll starts pulling the header back into
view. Swap the two and the header snaps back the instant you flick up, which is
the wrong feel and the most common mistake in this layout.

`edgeEffect(EdgeEffect.Fade)` rather than `Spring` matters here too: a springy
inner list fights the parent hand-off at the boundary.

`onReachEnd` fires whenever the end is visible, and the sample sets `isLoading`
before the timeout without checking whether a load is already in flight - so a
user who sits at the bottom triggers overlapping appends. Guard on `isLoading`
in production. `pushData` also calls `notifyDataAdd` once with the last index
after pushing four items, which under-reports the insertion; `LazyForEach` gets
away with it here because the list is re-measured anyway, but the correct form
notifies once per inserted index (or calls `notifyDataReload`).

**The measured indicator — `entry/src/main/ets/component/CustomTabBarView.ets`** (as shipped, trimmed)

```typescript
import { componentUtils } from '@kit.ArkUI';

@Component
export struct CustomTabBarView {
  @Link @Watch('change') tabBarIndex: number;
  @State indicatorLeftMargin: number = 0;
  @State indicatorWidth: number = 0;
  tabsController: TabsController = new TabsController();
  private listScroller: Scroller = new Scroller();

  change(changedPropertyName: string) {
    let targetIndexInfo = this.getTextInfo('bar' + this.tabBarIndex);
    this.startAnimateTo(300, targetIndexInfo.left - 10, targetIndexInfo.width);
  }

  getTextInfo(id: string): Record<string, number> {
    let modePosition: componentUtils.ComponentInfo = this.getUIContext().getComponentUtils().getRectangleById(id);
    return {
      'left': this.getUIContext().px2vp(modePosition.windowOffset.x),
      'width': this.getUIContext().px2vp(modePosition.size.width)
    };
  }

  startAnimateTo(duration: number, leftMargin: number, width: number) {
    this.getUIContext()?.animateTo({
      duration: duration,
      curve: Curve.Linear,
      iterations: 1,
      playMode: PlayMode.Normal,
    }, () => {
      this.indicatorLeftMargin = leftMargin;
      this.indicatorWidth = width;
    });
  }
}
```

**Measure, do not compute.** The tabs have different label lengths, so there is
no arithmetic that gives the underline its position; `getRectangleById` reads the
laid-out rectangle of the node tagged `.id('bar' + index)` and returns it in
**px**, which is why both values go through `px2vp`. The `- 10` on the left is
the parent padding the window offset includes and the margin does not.

Both state writes happen **inside** the `animateTo` closure - that is the
contract (`animateTo` animates state changed within the closure), and it is the
exact rule the sibling sample `SHOP-11` violates. Because both writes are in one
closure, the underline slides and resizes as a single motion.

The seeding path is the subtle part. `getRectangleById` returns zeros before the
first layout, so the initial position cannot come from `change()`; it comes from
`onAreaChange` on the label itself, guarded so it only takes effect once:

```typescript
.id('bar' + tabIndex.toString())
.onAreaChange((oldValue: Area, newValue: Area) => {
  if (this.tabBarIndex === tabIndex &&
    (this.indicatorLeftMargin === 0 || this.indicatorWidth === 0)) {
    if (newValue.position.x !== undefined) {
      let positionX = Number.parseFloat(newValue.position.x.toString());
      this.indicatorLeftMargin = Number.isNaN(positionX) ? 0 : positionX;
    }
    let width = Number.parseFloat(newValue.width.toString());
    this.tabWidth = Number.isNaN(width) ? 0 : width;
    this.indicatorWidth = this.tabWidth;
  }
});
```

And the drag path is one line on the `List`:

```typescript
.onWillScroll((xOffset: number) => {
  this.indicatorLeftMargin -= xOffset;
});
```

`onWillScroll` reports the delta about to be applied, so subtracting it from the
margin keeps the underline pinned to its label through a finger drag without any
re-measurement. This is cheaper and smoother than re-calling `getRectangleById`
per frame - but note it is unguarded, so the margin drifts if the bar is dragged
past its bounds and rubber-banded back.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`.

The page reaches under the system bars with
`.expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.BOTTOM, SafeAreaEdge.TOP])`
on the `Refresh`, which is the declarative alternative to the
`setWindowLayoutFullScreen` + `AppStorage` avoid-area dance used by `SHOP-09`
and `SHOP-11`. It is the better option when the page has no absolutely
positioned overlays: no ability code, no listener to leak.

Both `MallMainPage` and `CustomTabBarView` hold
`this.getUIContext().getHostContext() as common.UIAbilityContext` purely to call
`resourceManager.getStringSync(item.id)` inside `ForEach` key generators, because
`Resource` objects have no stable string form. Note that `ProductView` does the
same inside `build()` for display text - synchronous resource reads on every
render pass; prefer binding the `Resource` straight into `Text` where the label
is not being concatenated.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `Refresh` only refreshes: `onRefreshing` clears the flag after a second and
  reloads nothing. The product data is four static `ProductDataModel` entries
  appended over and over, so scrolling forever shows the same four items cycling.
- `MallMainPage.listScroller` is never attached to a component, so the
  swipe-to-centre-the-tab behaviour the code intends does not happen.
- The tab bar's `@State tabArray` duplicates the page's `@State tabArray`
  verbatim - seven `$r()` entries declared twice. If the tab set becomes dynamic
  these will diverge; hoist it to a shared constant.
- `V1` and `V2` state are mixed across the module: the page and the tab bar are
  `@Component`/`@State`/`@Link`, while `CatalogView` and `ProductView` are
  `@ComponentV2`/`@Param`/`@Local`. That works, but `@Param filterParam` in
  `ProductView` is never actually read - the filter is decorative.
- Only `phone`, `tablet` and `2in1` device types are declared, and the tab bar
  geometry (`height(52)`, `top: 44` on the indicator) is hardcoded in vp.

## Pitfalls

- **`HW-16-0011`** (E/low, confirmed): The doc's step-2 snippet is headed
  `// MainMallPage.ets` while the project tree in the same document and the zip
  both use `MallMainPage.ets`, so a reader searching the sample for the named
  file finds nothing. Fix: correct the snippet comment to `MallMainPage.ets`.
- **`HW-16-0013`** (E/medium, confirmed): This doc is one of the ~30 shopping
  pages whose abridged snippets are cut mid-construct. The step-2 snippet ends
  the `ForEach` key generator with a bare `// ...` and never closes
  `TabContent`, the `ForEach` call or the arrow function, so it does not parse.
  Fix: take the code from `ScrollCellingDemo.zip`, not from the document; the
  zip source is valid. See `SHOP-12` for the full account of this systematic
  defect.

## References

- `huawei_industry_tree/16_shopping/docs/10_scroll_celling_demo.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/scroll_celling_demo-0000002331639969
- `documentation/harmonyos-references/02_application-framework/ts-container-refresh.md` - `Refresh`, `pullToRefresh`, `onRefreshing`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-refresh
- `documentation/harmonyos-references/02_application-framework/ts-container-scroll.md` - `Scroll` and `Scroller`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scroll
- `documentation/harmonyos-references/02_application-framework/ts-container-scrollable-common.md` - `nestedScroll` and `NestedScrollMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scrollable-common
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` - `barHeight`, `onAnimationStart`, `TabsController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-lazyforeach.md` - `IDataSource` and the notify methods
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-lazyforeach
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-componentutils.md` - `getRectangleById` and `ComponentInfo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-componentutils
- `documentation/harmonyos-references/02_application-framework/ts-explicit-animation.md` - `animateTo` and the closure rule
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-explicit-animation
- `documentation/harmonyos-references/02_application-framework/ts-universal-component-area-change-event.md` - `onAreaChange` and `Area`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-component-area-change-event
- `SHOP-11` - the same `animateTo` API, used incorrectly
- `SHOP-12` - the systematic doc-snippet defect `HW-16-0013`
