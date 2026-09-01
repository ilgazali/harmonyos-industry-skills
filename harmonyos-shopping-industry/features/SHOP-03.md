---
id: SHOP-03
title: Coupon wallet - ticket-shaped list rows built from a one-sided border, with a usable/expired segmented switch
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/03_coupons_page.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/coupons_page-0000002235638646
sample: huawei_industry_tree/16_shopping/downloads/CouponsPage.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [List, ListItem, ForEach, border, BorderStyle, LengthMetrics, "resourceManager.getRawFileContent", "buffer.from", expandSafeArea, "UIContext.px2vp", "UIContext.getPromptAction", showToast, "@StorageLink", "window.getWindowAvoidArea", setWindowLayoutFullScreen, hilog]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-16-0004, HW-16-0027]
status: verified-with-fixes
---

## When to use

Load this card when you need a **card wallet**: coupons, vouchers, tickets,
membership passes - a list of rows that each look like a torn ticket, split by
a perforation between a value block on the left and a description block on the
right, with a switch at the top between the ones you can still use and the ones
you cannot.

The pattern is small and has exactly one trick worth learning: the perforation
is not an image and not a `Divider`. It is a **one-sided border on the left
column**, drawn by giving `border({ width: ... })` a width of `1` on a single
edge and `0` on the other three. Everything else - the segmented switch, the
two datasets, the greyed-out styling of expired cards - is ordinary state.

It generalises to any "two-part card" row: an order card with a price rail, a
boarding pass with a stub, a receipt line with a totals column. And because the
divider is a border rather than a child component, it costs no extra node in
the layout tree and follows the column's height automatically.

## Feature checklist

- A back chevron and a 优惠劵 (coupon) title in a fixed header row.
- A two-segment switch, 可使用 (usable) / 已过期 (expired), where the active
  segment is a white pill inside a light grey track.
- Tapping a segment swaps the whole list between two datasets.
- Each coupon row shows a discount value and its threshold on the left, and a
  name, a validity window and a scope label on the right.
- A vertical divider separates the value block from the description block.
- A trailing button reads 使用 (use) on usable coupons and 失效 (invalid) on
  expired ones; the expired button has `stateEffect(false)` and no handler.
- Tapping a usable coupon's button raises a 仅供演示 (demo only) toast.
- Both datasets are loaded from rawfile JSON at page entry, not compiled in.
- The page paints under the status bar and the navigation indicator, with the
  content inset by the avoid-area heights.

## Architecture

One `entry` module, one page, one model, one constants file. There is no
view-model layer: the page holds the two arrays and a boolean.

```
entry/src/main/ets
├── constants/StyleConstants.ets     every numeric literal in the page, named
├── entryability/EntryAbility.ets    full screen + avoid areas -> AppStorage
├── model/CouponInfo.ets             the five-field coupon interface
└── pages/CouponsPage.ets            @Entry, the whole feature (224 lines)
entry/src/main/resources/rawfile
├── Coupons.json                     six usable coupons
└── ExpiredCoupons.json              the expired set
```

**The documented tree does not match the zip**: the doc's 工程目录 lists only
`entryability`, `model` and `pages` and omits the `constants` directory, even
though `CouponsPage.ets` imports `StyleConstants` on its fourth line
(`HW-16-0004`). A reader following the tree cannot reconstruct a compiling
project.

**The design decision worth copying** is that the data lives in `rawfile`
JSON and is read through `resourceManager.getRawFileContent` in
`aboutToAppear`, not as a `const` array in a `.ets` file. That single choice is
what makes the sample a usable skeleton: swapping the local read for a network
fetch changes one method and nothing else, because the rest of the page already
treats `coupons` as an asynchronously-arriving `@State` array that starts empty.
The `CouponInfo` interface is the contract between the two.

The decision **worth avoiding** is the segmented switch: it is two `Text`
elements inside a `Flex` with mirrored ternaries on `fontColor` and
`backgroundColor`, driven by one boolean. That is fine for two segments and
collapses immediately at three. Reach for `Tabs` with a `SubTabBarStyle`, or
drive a `ForEach` over a segment array, as soon as a third category appears.

## Implementation steps

1. **Define the row model as an interface**, not a class - `CouponInfo` is
   pure data crossing a `JSON.parse` boundary, so it needs no constructor.
2. **Put both datasets in `resources/rawfile`** as JSON arrays with exactly the
   interface's field names.
3. **Read them in `aboutToAppear`** with `getRawFileContent`, decode the
   `Uint8Array` through `buffer.from(value.buffer).toString()`, and
   `JSON.parse` into the typed array. Keep the `try/catch`: a missing or
   malformed rawfile must leave the page empty, not crash it.
4. **Hold the category as one boolean** and switch the `ForEach` source on it -
   `this.couponCategory ? this.coupons : this.expiredCoupons`. Both arrays stay
   loaded, so switching costs no I/O.
5. **Draw the perforation with `border`**, giving the left column a width of
   `1` on the `end` edge and `0` on the other three, using `LengthMetrics.vp`
   so the edge follows the layout direction in RTL locales.
6. **Set `style: { end: BorderStyle.Dashed }`** if you want the perforated look
   the document promises; the sample ships `{ right: BorderStyle.Solid }`,
   which is both a solid line and a non-localized key that does not pair with
   the localized widths (`HW-16-0004`).
7. **Publish the avoid-area heights from the ability** into `AppStorage` and
   read them back with `@StorageLink`, converting with `px2vp` at the point of
   use - the window API reports px, the layout wants vp.
8. **Give the list `expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.BOTTOM])`**
   so rows scroll under the navigation indicator instead of stopping above it.

## Verified snippets

All snippets are from `CouponsPage.zip`. Corrected forms are marked.

**Loading the two datasets — `entry/src/main/ets/pages/CouponsPage.ets`** (as shipped)

```typescript
import { CouponInfo } from '../model/CouponInfo';
import { buffer } from '@kit.ArkTS';
import { hilog } from '@kit.PerformanceAnalysisKit';

@Entry
@Component
struct Coupons {
  @State coupons: CouponInfo[] = [];
  @State expiredCoupons: CouponInfo[] = [];
  @State couponCategory: boolean = true;
  private context = this.getUIContext().getHostContext() as Context;

  async aboutToAppear() {
    // 加载初始数据
    this.coupons = await this.getDataFromJSON('Coupons.json');
    this.expiredCoupons = await this.getDataFromJSON('ExpiredCoupons.json');
  }

  async getDataFromJSON(fileName: string) {
    let result: CouponInfo[] = [];
    try {
      let value: Uint8Array = await this.context.resourceManager.getRawFileContent(fileName);
      let str = buffer.from(value.buffer).toString();
      result = JSON.parse(str) as CouponInfo[];
    } catch (err) {
      hilog.error(0x0000, 'tag', err);
    }
    return result;
  }
}
```

**Three details carry this loader.** `getRawFileContent` returns a
`Uint8Array`, not text, so the `buffer.from(value.buffer).toString()` hop is
mandatory - passing the array straight to `JSON.parse` yields the string
`"[object Uint8Array]"` and throws. The `try` wraps both the read and the
parse, so a corrupt file degrades to an empty list rather than an unhandled
rejection inside `aboutToAppear`. And `result` is declared and returned
whatever happens, which is why the two `@State` assignments can be
unconditional.

Note the context is taken once as a field via
`this.getUIContext().getHostContext()`, not fetched per call - the modern
replacement for the deprecated `getContext(this)`.

**The ticket divider — same file** (corrected, see `HW-16-0004`)

```typescript
Column({ space: StyleConstants.SPACE_4 }) {
  Text(info.discount)                      // e.g. 88折 or ￥50
    .fontColor(this.couponCategory ? $r('app.color.usable') : $r('app.color.unusable'))
    .fontSize(StyleConstants.FONT_SIZE_24);
  Text(info.condition)                     // e.g. 满100可用 - usable over 100
    .fontColor(this.couponCategory ? $r('app.color.usable') : $r('app.color.unusable'))
    .fontSize(StyleConstants.FONT_SIZE_12);
}
.margin({ start: LengthMetrics.vp(-16) })
.layoutWeight(StyleConstants.ONE)
.height(StyleConstants.FULL)
.border({
  width: {
    start: LengthMetrics.vp(0),
    end: LengthMetrics.vp(1),
    top: LengthMetrics.vp(0),
    bottom: LengthMetrics.vp(0)
  },
  color: { end: $r('app.color.border') },
  style: {
    end: BorderStyle.Dashed          // FIX: sample ships `{ right: BorderStyle.Solid }`
  }
});
```

**A border with three zero widths is a divider that costs nothing.** The value
block already spans the row's full height because of `.height('100%')`, so the
single `end` edge draws a rule from the top padding to the bottom padding with
no extra component, no measured height and no `Divider` to keep in sync. Using
`layoutWeight(1)` on this column and letting the description column size itself
is what keeps the rule in the same place on every row regardless of text
length.

`LengthMetrics.vp` with the `start`/`end` keys is the localized form: in an
RTL locale the rule moves to the other side of the block automatically. The
shipped `style: { right: ... }` mixes the non-localized vocabulary into the
same object, so in RTL the width and the style would describe different edges.
And `BorderStyle.Solid` contradicts the document's own claim of a
单侧虚线 - a one-sided **dashed** line (`HW-16-0004`).

**The segmented switch — same file** (as shipped)

```typescript
Flex({ justifyContent: FlexAlign.SpaceAround, alignItems: ItemAlign.Center }) {
  Text($r('app.string.usable'))
    .height(StyleConstants.HEIGHT_40)
    .borderRadius(StyleConstants.BORDER_RADIUS_20)
    .textAlign(TextAlign.Center)
    .fontColor(this.couponCategory ? $r('app.color.font_color_dark') : $r('app.color.font_color_light'))
    .backgroundColor(this.couponCategory ? Color.White : '')
    .flexGrow(StyleConstants.ONE)
    .onClick(() => {
      this.couponCategory = true;
    });

  Text($r('app.string.unusable'))
    .height(StyleConstants.HEIGHT_40)
    .borderRadius(StyleConstants.BORDER_RADIUS_20)
    .textAlign(TextAlign.Center)
    .fontColor(!this.couponCategory ? $r('app.color.font_color_dark') : $r('app.color.font_color_light'))
    .backgroundColor(!this.couponCategory ? Color.White : '')
    .flexGrow(StyleConstants.ONE)
    .onClick(() => {
      this.couponCategory = false;
    });
}
.width(StyleConstants.FULL)
.borderRadius(StyleConstants.BORDER_RADIUS_20)
.backgroundColor($r('app.color.light_background_color'));
```

**The pill effect comes from three cooperating radii,** not from a moving
indicator. The track is a full-width `Flex` with `borderRadius(20)` over a grey
background; each half is `flexGrow(1)` so the two segments split it exactly;
and the active half paints itself white with the *same* radius so it reads as a
pill sitting inside the track. The inactive half sets `backgroundColor('')` -
an empty string, which resolves to transparent and lets the track show through.

This is cheap and readable at two segments. It is also why the whole page's
category state is a boolean rather than an index, which is what makes the
`ForEach` source, four `fontColor` ternaries, the row background and the button
label all switch off one assignment.

## Permissions & config

**None.** `entry/src/main/module.json5` declares no `requestPermissions`.

`deviceTypes` is `["phone", "tablet", "2in1"]`, and the layout is a single
column with no breakpoint handling, so on a 2in1 window the coupon rows stretch
to the full window width. `EntryAbility.onCreate` pins the app to light mode
with `setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT)`, which is
why the hardcoded `Color.White` on the active segment is safe here and would
not be in a theme-following app.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- The back chevron in the header has **no `onClick`** - it is a picture of a
  back button. The page is the ability's loaded content, so there is nowhere to
  go back to in this sample; wire it to a `NavPathStack.pop()` when the wallet
  becomes a pushed destination.
- The row background ternary reads
  `!this.couponCategory ? light_background_color : lapsedBackgroundColor`, i.e.
  the *usable* list gets the colour named "lapsed". The names and the branches
  disagree; check the palette before reusing the expression.
- `ForEach`'s key generator is `JSON.stringify(info)`, so two coupons with
  identical field values would collide into one row. The shipped `Coupons.json`
  contains two entries both named 商品优惠券 that differ only in `discount`,
  so it works here by luck; key on a coupon id in real data.
- Both datasets are read on every page entry with no cache and no loading
  state; the list is briefly empty on first frame.
- Validity windows in the sample data run to 2030, so nothing "expires" during
  a demo - the expired list is a separate file, not a computed filter.

## Pitfalls

- **`HW-16-0004`** (E/low, confirmed): **the document's central claim is not
  what the code draws.** Doc line 37 promises 使用border属性来绘制单侧虚线 - a
  one-sided **dashed** line - while `CouponsPage.ets:142-144` sets
  `style: { right: BorderStyle.Solid }`, so the shipped divider is solid, and
  the non-localized `right` key is paired with localized `start`/`end` widths.
  The same finding covers the incomplete 工程目录: the tree omits
  `constants/StyleConstants.ets`, which the page imports. Fix: set
  `style: { end: BorderStyle.Dashed }` (or correct the doc's wording) and
  regenerate the tree from the zip.

## References

- `huawei_industry_tree/16_shopping/docs/03_coupons_page.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/coupons_page-0000002235638646
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `lanes`, `scrollBar`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-foreach.md` - `ForEach` and its key generator
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-foreach
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-border.md` - per-edge `width`, `color`, `style` and the localized `start`/`end` keys
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-border
- `documentation/harmonyos-references/02_application-framework/ts-appendix-enums.md` - `BorderStyle`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-appendix-enums
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getRawFileContent`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-expand-safe-area.md` - `expandSafeArea`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-expand-safe-area
- `documentation/harmonyos-references/02_application-framework/js-apis-window.md` - `getWindowAvoidArea`, `setWindowLayoutFullScreen`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-window
- `SHOP-11` - the skeleton screen that belongs in front of this page's empty first frame
- `SHOP-05` - coupon collection, the flow that fills this wallet
