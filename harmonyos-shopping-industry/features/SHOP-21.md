---
id: SHOP-21
title: Product comparison - pick a rival from a full-height bindSheet, then a two-column spec table
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/21_goods_pk.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/goods_pk-0000002349269256
sample: huawei_industry_tree/16_shopping/downloads/GoodsPK.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [bindSheet, DismissSheetAction, Radio, RadioStyle, Navigation, NavPathStack, pushPathByName, getParamByName, NavDestination, "NavDestination.onReady", routerMap, "@Provide", "@Consume", "@StorageProp", getPromptAction, showToast, hilog, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-16-0023, HW-16-0027]
status: verified-with-fixes
---

## When to use

Load this card when a detail page has to **pull a second item in beside the
one already on screen** - compare this chair with that chair, this plan with
that plan - without losing the page the user came from. The pattern is a
full-height `bindSheet` over the detail page carrying a pickable list, and a
`NavDestination` that receives two ids and renders them as parallel columns.

The shape generalises past shopping. Anything with "pick one more of the same
kind and put them side by side" fits: two insurance quotes, two flights, two
phone tariffs, two candidates in a hiring tool. The load-bearing idea is that
the picker is a sheet (modal, cancellable, keeps the detail page mounted
underneath) while the comparison is a route (has a title bar, a back gesture,
and can be linked to directly through the router map).

**Read `HW-16-0023` before copying the selection code.** The sheet looks like
a working single-choice list, but the radios cannot actually tell the
framework which product is selected - the selection only survives because
`onChange` writes the id into state by hand.

## Feature checklist

- The product detail page carries a floating PK entry pill positioned over the
  hero image; tapping it raises a full-height sheet.
- The sheet has a header with a title and a close cross, a search box (a demo
  toast), and a scrollable list of candidate products.
- The candidate list excludes the product already being viewed.
- Exactly one candidate can be selected; the confirm button is grey until then.
- Confirming with nothing selected raises a 请先选择 (please choose first)
  toast instead of navigating.
- Confirming with a selection closes the sheet and pushes the comparison page
  with both product ids.
- Dismissing the sheet by the cross, by swipe-down, or by back resets the
  selection so the next open starts clean.
- The comparison page shows both products as cards, then three comparison
  blocks: key parameters, delivery, and service assurances (tick/cross icons).

## Architecture

One `entry` module, two pages, a constants file and a static product list. No
network, no persistence.

```
entry/src/main/ets
├── common/Constants.ets            Constants + Position + MarginPadding + PKParams row labels
├── entryability/EntryAbility.ets   full-screen window, avoid areas -> AppStorage
├── entrybackupability/
├── model/ListInfo.ets              Goods interface, PageParam interface, 5 static products
└── pages
    ├── MainPage.ets                @Entry: detail page + the whole PK sheet (398 lines)
    └── PKDetails.ets               the comparison NavDestination, reached by routerMap
```

The documented 工程目录 matches the zip exactly.

**The design decision worth copying** is the split between the sheet and the
route. `MainPage` owns a `@Provide('pageInfos') pageInfos: NavPathStack` and
wraps its content in `Navigation(this.pageInfos)`; `PKDetails` takes the same
stack with `@Consume` and is registered in `resources/base/profile/route_map.json`
under the name `PKDetails`. So the picker never leaves the detail page - it is
a sheet bound to the `Navigation` itself - while the result is a real
destination with its own title bar and back behaviour. The only thing that
crosses the boundary is a two-number `PageParam { goodsOriginal, goodsPK }`,
which keeps `PKDetails` re-entrant: it looks both products up from the shared
`goods` array by index rather than receiving copies.

The one structural weakness: `MainPage` is a single 398-line struct holding
eight `@Builder` methods, several of which (`comment`, `shopRow`,
`operationRow`) are static page furniture that has nothing to do with the PK
feature. Lifting the sheet into its own component would make the pattern
reusable; as shipped, adopting it means copying builders out of a page.

## Implementation steps

1. **Wrap the detail page in `Navigation(this.pageInfos)`** with the stack
   declared `@Provide`, and register the comparison page in the module's
   `routerMap` profile so it can be pushed by name.
2. **Bind the sheet to the `Navigation`, not to a child**, with
   `bindSheet($$this.isSheetShow, this.sheetContent(), {...})`. `$$` makes the
   binding two-way so a system-driven dismissal writes `false` back.
3. **Set `height: '100%'` and `showClose: false`** and draw your own header
   cross - a full-height sheet with the system close button leaves two close
   affordances in the same corner.
4. **Reset the selection in `onWillDismiss`** after calling
   `dismissSheetAction.dismiss()`, so a swipe-down leaves no stale
   `isChosen`/`isClickable`.
5. **Exclude the current product from the candidate list** with
   `goods.slice(this.goodsOriginal + 1)`, and key the `ForEach` on
   `goodsItem.id.toString()` - the sample does key it correctly.
6. **Give every radio a unique `value` and a group name of its own
   cluster** (`HW-16-0023`). The shipped code uses `value: 'Radio1'`,
   `group: 'radioGroup'` for every row, which is what makes the radios purely
   decorative.
7. **Track the choice in two pieces of state**: `isChosen` (the id, `0` when
   nothing is picked) and `isClickable` (drives the button colour). Guard the
   confirm handler on `isChosen !== 0` and toast otherwise.
8. **Push with a typed param object** and read it back in
   `NavDestination.onReady` via `getParamByName`, which returns an *array* of
   params for that name - index `[0]` and check the length.
9. **Build the comparison as three parameterised builders** over `string[]`
   columns rather than a row-per-spec table; see the snippet below for why.

## Verified snippets

All snippets are from `GoodsPK.zip`. Corrected forms are marked.

**The sheet binding - `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
@Provide('pageInfos') pageInfos: NavPathStack = new NavPathStack();
@State isSheetShow: boolean = false;
@State isChosen: number = 0;
@State isClickable: boolean = false;
private goodsOriginal: number = 0;

build() {
  Navigation(this.pageInfos) {
    Column() {
      this.goodsInformation();   // hero image + the floating PK pill
      this.comment();
      this.shopRow();
      this.operationRow();
    }
    .padding({ bottom: this.getUIContext().px2vp(this.bottomRectHeight) });
  }
  .hideTitleBar(true)
  .bindSheet($$this.isSheetShow, this.sheetContent(), {
    height: Constants.FULL_HEIGHT,
    blurStyle: BlurStyle.Thick,
    showClose: false,
    backgroundColor: Constants.BACKGROUND_COLOR,
    onWillDismiss: ((dismissSheetAction: DismissSheetAction) => {
      dismissSheetAction.dismiss();      // let the dismissal proceed...
      this.isClickable = false;          // ...then clear the selection
      this.isChosen = 0;
    }),
  });
}
```

**Three options carry the design.** `$$this.isSheetShow` is a two-way binding:
`bindSheet` writes `false` back when the sheet is dismissed by gesture, so the
page state cannot drift out of step with what is on screen. `showClose: false`
hands the close affordance to the sheet's own header row, which is what lets
the header carry a title as well. And `onWillDismiss` is the only hook that
sees *every* dismissal route - cross, swipe, back - which is why the selection
reset lives there rather than in the cross's `onClick` (the cross handler
resets it too, which is redundant but harmless).

Note the sheet is bound to the `Navigation`, not to the inner `Column`. That
matters once `PKDetails` is on the stack: the sheet belongs to the page host,
so it cannot be left dangling over a destination that did not open it.

**The candidate row and its radio - same file** (corrected, see `HW-16-0023`)

```typescript
@Builder
singleGoods(goodsItem: Goods) {
  Row() {
    Column() {
      Image(goodsItem.goodsImage)
        .width(Constants.GOODS_IMAGE_WIDTH)
        .height(Constants.GOODS_IMAGE_HEIGHT);
    }
    .height(Constants.FULL_HEIGHT);

    Column() {
      Text(goodsItem.goodsName)
        .maxLines(Constants.MAX_LINES);
      Row() {
        Text(`${goodsItem.price} 补贴后 `)          // price after subsidy
          .fontColor(Constants.RED);
        Text(`${goodsItem.customers}人付款`)        // N people bought this
          .fontColor(Constants.FONT_COLOR_GREY);
      }
    }
    .flexGrow(1)
    .alignItems(HorizontalAlign.Start);

    Radio({ value: goodsItem.id.toString(), group: 'pkGoodsGroup' })   // FIX: shipped code is
      .checked(this.isChosen === goodsItem.id)                         // value:'Radio1', group:'radioGroup'
      .radioStyle({ checkedBackgroundColor: Constants.BLUE })          // FIX: shipped code is .checked(false)
      .width(Constants.RADIO_WIDTH)
      .onChange(() => {
        this.isChosen = goodsItem.id;
        this.isClickable = true;
      });
  }
  .height(Constants.SINGLE_GOODS_HEIGHT)
  .backgroundColor(Constants.WHITE)
  .borderRadius(Constants.RADIUS_NORMAL);
}
```

**`value` is the radio's identity inside `group`, and it must be unique.** The
reference is explicit that only one radio in a given group can be selected at
a time, and that `value` is the value of that button - so five rows carrying
`'Radio1'` in one group give the framework nothing to distinguish them by. In
the shipped sample the visual single-choice behaviour comes entirely from the
group exclusion (any one of five identical values still deselects the others),
and the *actual* selection is recovered by `onChange` writing `goodsItem.id`
into state. Change the list to be paged or filtered and that coincidence stops
holding.

The second half of the fix is `checked`. The sample hardcodes `.checked(false)`,
so the component has no declarative link to `isChosen` at all: after
`onWillDismiss` clears the state, the radio's own visual state is whatever the
framework kept. Driving `checked` from `this.isChosen === goodsItem.id` makes
the list a function of state, which is what lets the reset in step 4 actually
show.

**Confirm and push - same file** (as shipped)

```typescript
Button($r('app.string.PK'))
  .width(Constants.FULL_WIDTH)
  .height(Constants.PK_BUTTON_HEIGHT)
  .backgroundColor(this.isClickable ? Color.Blue : Color.Gray)
  .onClick(() => {
    if (this.isChosen !== 0) {
      this.isSheetShow = false;
      const goodsChooseParam: PageParam = { goodsOriginal: this.goodsOriginal, goodsPK: this.isChosen };
      this.pageInfos.pushPathByName('PKDetails', goodsChooseParam);
      this.isClickable = false;
      this.isChosen = 0;
    } else {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.please_choose_at_first') });
    }
  });
```

The button is never actually disabled - it is grey and it still fires,
answering with a toast. That is the friendlier of the two options: a disabled
button that does nothing gives the user no explanation. Note the ordering:
`isSheetShow = false` first, then the push, so the sheet is already retracting
as the destination animates in.

The guard is `isChosen !== 0`, which quietly makes **product id 0 unselectable
as a comparison target** - it is indistinguishable from "nothing chosen". The
sample gets away with it because `goodsOriginal` is 0 and the candidate list
starts at index 1. Use `-1` (or `null`, which `PageParam` already permits) as
the sentinel in real code.

**Reading the two ids back - `entry/src/main/ets/pages/PKDetails.ets`** (as shipped)

```typescript
@Builder
export function PKDetailsBuilder() {
  PKDetails();
}

@Component
struct PKDetails {
  @Consume('pageInfos') pageInfos: NavPathStack;
  @State original: number = 0;
  @State pk: number = 0;

  build() {
    NavDestination() {
      Column() {
        Row() {
          this.goods(false, goods[this.original].goodsImage,
            goods[this.original].goodsName, goods[this.original].price);
          this.goods(true, goods[this.pk].goodsImage,
            goods[this.pk].goodsName, goods[this.pk].price);
        }
        .justifyContent(FlexAlign.SpaceBetween);

        this.textComparison(
          [goods[this.original].handrail, goods[this.original].fabric, goods[this.original].lifting],
          [goods[this.pk].handrail, goods[this.pk].fabric, goods[this.pk].lifting],
          PKParams.PARAMETER);            // ['扶手类型', '面料材质', '升降方式']
        this.assuranceView(
          goods[this.original].assurance, goods[this.pk].assurance, PKParams.ASSURANCE);
      }
    }
    .title($r('app.string.goods_pk'))
    .onReady(() => {
      let indexStr: string = JSON.stringify(this.pageInfos.getParamByName('PKDetails'));
      let indexObj: Array<PageParam> = JSON.parse(indexStr);
      if (indexObj.length > 0) {
        this.original = indexObj[0].goodsOriginal;
        this.pk = indexObj[0].goodsPK as number;
      }
    });
  }
}
```

**`getParamByName` returns an array, one entry per instance of that
destination on the stack** - hence the length check and the `[0]`. The
`JSON.stringify`/`JSON.parse` round trip is the sample's way of getting a
typed `Array<PageParam>` out of an `Object[]` in ArkTS, where a plain cast is
rejected; it is idiomatic in this corpus, if wasteful.

The comparison itself is built **column-first, not row-first**:
`textComparison(left[], right[], labels[])` renders three parallel `Column`s
whose items line up because all three use `justifyContent(FlexAlign.SpaceBetween)`
inside the same fixed height. That is what makes the left and right panels
tintable as whole blocks (`WEAK_RED` against `WEAK_BLUE`) and it keeps the
spec labels in one place - `PKParams` in `Constants.ets`. The cost is that the
alignment is geometric rather than structural: a label that wraps to two lines
on a narrow screen shifts one column out of step with the others. A `Grid`
with three columns would survive that; for a fixed spec count on phones, this
does not.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

`module.json5` targets `phone`, `tablet` and `2in1` and points `routerMap` at
`$profile:route_map`, whose single entry maps the name `PKDetails` to
`PKDetailsBuilder`. The exported `@Builder` function next to the struct is
mandatory for router-map navigation - the struct alone is not addressable.

`EntryAbility` sets `COLOR_MODE_LIGHT` for the whole application, then goes
full screen and publishes `topRectHeight` / `bottomRectHeight` into
`AppStorage`; both pages read them with `@StorageProp` and convert with
`px2vp` at the point of use.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The comparison is hardwired to **exactly two** products and to this product
  schema (handrail / fabric / lifting / delivery / assurance). The assurance
  block renders five identical ticks or five identical crosses from one
  boolean per product - `ForEach(Array.from(Array(5)), ...)` - so it is a
  placeholder, not five independent assurances.
- The whole product set is five entries in `ListInfo.ets`; `goodsOriginal` is
  fixed at `0`, so the detail page always shows the same chair.
- Layout is in absolute vp throughout (`Position.PK_X = 280`,
  `GOODS_TOTAL_HEIGHT = 512`), and the PK pill is placed with `position({x, y})`
  over the hero image. On a tablet or a resized 2in1 window the pill lands in
  the wrong place. Declared device types include `tablet` and `2in1`.
- `setColorMode(COLOR_MODE_LIGHT)` is forced, so the hardcoded rgb literals
  are safe here and only here - the sample has no dark palette.
- Search, buy, favourite and the shop row are all display-only toasts.

## Pitfalls

- **`HW-16-0023`** (B/medium, confirmed): every candidate radio in the compare
  sheet is `Radio({ value: 'Radio1', group: 'radioGroup' })`, so the framework
  cannot distinguish the rows, and the group name is generic enough to collide
  with any other radio cluster added to the page later. The selection works
  only because `onChange` assigns `goodsItem.id` by hand, and `.checked(false)`
  is hardcoded so nothing drives the visual state from `isChosen`. Fix: unique
  `value` per row (`goodsItem.id.toString()`), a dedicated group
  (`'pkGoodsGroup'`), and `checked(this.isChosen === goodsItem.id)`. The doc's
  step-1 snippet teaches the same broken pattern.
- The doc's step-1 excerpt is not compilable as printed: it shows an empty
  `Navigation(this.pageInfos) { }` with the `bindSheet` chain, then an
  orphaned `Radio` at the same level referencing an undeclared `goodsItem`.
  Treat it as an illustration of the two calls, not as code to paste - the
  zip's `MainPage.ets` is the source of truth. (This doc is not one of the
  instances enumerated under the corpus-wide excerpt defect `HW-16-0013`, but
  it is the same editorial pattern.)
- `isChosen === 0` doubles as "nothing selected", so product id `0` can never
  be chosen as the comparison target. Use `-1` or `null`.

## References

- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - `bindSheet`, `SheetOptions`, `onWillDismiss` and `DismissSheetAction`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-radio.md` - `value`, `group` and the one-per-group rule
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-radio
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `NavPathStack`, `pushPathByName`, `getParamByName`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navdestination.md` - `NavDestination` and `onReady`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navdestination
- `documentation/harmonyos-guides/03_application-framework/arkts-set-navigation-routing.md` - the `routerMap` profile and the exported `@Builder`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-set-navigation-routing
- `huawei_industry_tree/16_shopping/docs/21_goods_pk.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/goods_pk-0000002349269256
- `SHOP-08` - long-press marking on the same product-card vocabulary
