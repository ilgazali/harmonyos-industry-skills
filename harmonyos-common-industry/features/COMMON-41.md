---
id: COMMON-41
title: Automatic page grayscale - turning the application gray for mourning or emergency observances
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/41_page_grayscale.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/page_grayscale-0000002368995430
sample: huawei_industry_tree/19_common_technical_solutions/downloads/PageGrayscale.zip
kits: ["@kit.ArkUI", "@kit.ArkTS"]
apis: [".grayscale", "window.Window.setWindowGrayScale", setTimeout, clearTimeout, "@State", "@Prop", "@Link", "@StorageProp", WaterFlow, FlowItem, ForEach, "UIContext.getPromptAction", "UIContext.px2vp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0123, HW-19-0124, HW-19-0125, HW-19-0126, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when the application must be rendered in grayscale on demand -
national days of mourning, memorial observances, or any policy that requires the
product to drop its colour. The requirement usually arrives with a date and a
deadline, so the implementation has to be something that can be switched on
centrally and switched off again.

The document demonstrates one mechanism: the `.grayscale()` universal attribute
on a page's root container, driven by a three-second `setTimeout`. That is the
right tool for *part of a page*. For the stated scenario - the whole application
- the platform provides `window.setWindowGrayScale`, which the document does not
mention (HW-19-0123). Read the Architecture section before copying the sample.

## Feature checklist

- Decide the **scope**: whole window (`setWindowGrayScale`) or one component
  subtree (`.grayscale()`).
- Drive the value from a **single source of truth** - a server flag or a
  configured date range - not from a per-page timer.
- Neutralise **accent-coloured selected states**: grayscale turns a red highlight
  into a muddy tone, so swap highlighted assets for neutral ones (HW-19-0126).
- Store and clear any timer used to trigger the switch (HW-19-0124).

## Architecture

Single-module project (`entry` HAP), one page:

| File | Responsibility |
| --- | --- |
| `pages/MainPage.ets` | holds `value` (the grayscale ratio) and `isGrayedOut`; applies `.grayscale(this.value)` to the root Column |
| `components/AllBusiness.ets` | the 4-lane service grid |
| `components/ProductList.ets` | the two-column `WaterFlow` product list |
| `components/TabBar.ets` | the bottom bar; reads `isGrayedOut` to restyle the selected tab |
| `viewmodel/*.ets` | static mock data for the three lists |

**Two variables, two different jobs.** `value: number` feeds `.grayscale()` and
does the visual work. `isGrayedOut: boolean` exists only because grayscale is
indiscriminate: the selected tab is drawn in `#F34D4F` with a coloured icon, and
a grayscale filter turns that into a washed-out tone that still reads as
"different" without reading as "selected". So the sample swaps in a neutral icon
and a neutral label colour before the filter is applied. This is the part of the
feature that is easy to miss, and it is exactly the part the document's snippet
leaves out.

**`.grayscale()` composes downward.** The reference: "The grayscale rendering of
the upper layer will overlay that of lower-layer child components." One call on
the root container covers the whole subtree - but only that subtree. Anything
mounted on the window rather than inside the page, and every other page in the
application, is unaffected.

**`setWindowGrayScale` is the whole-application answer.** API 12+, promise-based,
callable only after `loadContent()` / `setUIContent()`, gated by
`SystemCapability.Window.SessionManager`. One call in `EntryAbility` grays
everything the window draws, with no per-page state at all.

**The three-second timer is a demo device, not a design.** Real deployments read
a flag; the timer exists so the GIF in the document can show the transition.

## Implementation steps

1. **Choose the scope.**
   - Whole application: call `setWindowGrayScale` once in
     `EntryAbility.onWindowStageCreate`, after `loadContent`, inside
     `canIUse('SystemCapability.Window.SessionManager')` and with a `.catch`.
     This is what the scenario in the document actually describes (HW-19-0123).
   - One region of one page: `.grayscale(ratio)` on that region's container.
2. **Bind the ratio to state** so it can be turned on and off: `@State value:
   number = 0`, `.grayscale(this.value)`. `0.0` is unchanged, `1.0` is fully
   gray, and values in between are linear.
3. **Decide what turns it on.** A server-provided flag or a date comparison. If
   you use a timer for a demo, store its ID and clear it in `aboutToDisappear`
   (HW-19-0124).
4. **Neutralise selected states.** Add a boolean that accompanies the grayscale
   switch and use it to pick neutral icons and label colours for anything drawn
   in an accent colour.
5. **Pass that boolean down as `@Prop`**, not `@Link`, when the child only reads
   it (HW-19-0125).

## Verified snippets

All snippets below come from the sample project, not from the document.

### The page: state, timer, and the grayscale attribute

`PageGrayscale.zip#PageGrayscale/entry/src/main/ets/pages/MainPage.ets`

```ts
import { AllBusiness } from '../components/AllBusiness';
import { ProductList } from '../components/ProductList';
import { TabBar } from '../components/TabBar';

@Entry
@Component
struct MainPage {
  // Indicates whether the page has been grayed out
  @State isGrayedOut: boolean = false;
  // The ratio of grayscale conversion
  @State value: number = 0;
  @StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
  @StorageProp('topRectHeight') topRectHeight: number = 0;

  // The page will automatically turn gray after 3 seconds
  aboutToAppear(): void {
    setTimeout(() => {              // FIX (HW-19-0124): keep the returned ID
      this.value = 1;
      this.isGrayedOut = true;
    }, 3000);
  }

  build() {
    Column({ space: 19 }) {
      Text($r('app.string.shopping'))
        .fontSize(26)
        .fontWeight(700)
        .fontColor('rgba(0, 0, 0, 0.9)')
        .margin({ top: 11, left: 16 })
        .alignSelf(ItemAlign.Start);
      // Business
      AllBusiness();
      // Product List
      Column() {
        ProductList();
      }
      .width('100%')
      .layoutWeight(1)
      .padding({ left: 16, right: 16 });

      // Bottom tab bar
      TabBar({
        isGrayedOut: this.isGrayedOut
      });
    }
    .width('100%')
    .height('100%')
    .grayscale(this.value)
    .backgroundColor('#F1F3F5')
    .padding({
      top: this.getUIContext().px2vp(this.topRectHeight),
      bottom: this.getUIContext().px2vp(this.bottomRectHeight)
    });
  }
}
```

The corrected timer handling:

```ts
private timerId: number = -1;

aboutToAppear(): void {
  this.timerId = setTimeout(() => {
    this.value = 1;
    this.isGrayedOut = true;
  }, 3000);
}

aboutToDisappear(): void {
  clearTimeout(this.timerId);
}
```

### Neutralising the selected tab before the filter runs

`PageGrayscale.zip#PageGrayscale/entry/src/main/ets/components/TabBar.ets`

```ts
import { TabBarDetail, TAB_BAR_DETAILS } from '../viewmodel/TabBarModel';

@Component
export struct TabBar {
  @State tabBar: TabBarDetail[] = TAB_BAR_DETAILS;
  @Link isGrayedOut: boolean;        // FIX (HW-19-0125): @Prop - never written here

  build() {
    Row() {
      ForEach(this.tabBar, (item: TabBarDetail, index: number) => {
        Column({ space: 4 }) {
          Image(index === 0 && this.isGrayedOut ? $r('app.media.featured_products_normal') : item.icon)
            .width(24)
            .height(24);
          Text(item.name)
            .fontSize(10)
            .fontWeight(500)
            .fontColor(index === 0 && !this.isGrayedOut ? '#F34D4F' : 'rgba(0, 0, 0, 0.9)');
        }
        .height(48)
        .onClick(() => {
          this.getUIContext().getPromptAction().showToast({
            message: $r('app.string.toast')
          });
        });
      });
    }
    .width('100%')
    .justifyContent(FlexAlign.SpaceAround);
  }
}
```

Note the two conditions: the icon switches to `featured_products_normal` when
grayed, and the label drops `#F34D4F` at the same moment. `index === 0` is the
selected tab in this static sample.

### The product list - unaffected by grayscale, inherits it from the parent

`PageGrayscale.zip#PageGrayscale/entry/src/main/ets/components/ProductList.ets`

```ts
@Component
export struct ProductList {
  @State productList: ProductDetail[] = PRODUCT_DETAILS;
  private scroller: Scroller = new Scroller();

  build() {
    WaterFlow({ scroller: this.scroller }) {
      ForEach(this.productList, (item: ProductDetail) => {
        FlowItem() {
          this.listItem(item);
        }
        .width('100%')
        .height(274);
      });
    }
    .columnsTemplate('1fr 1fr')
    .columnsGap(8)
    .rowsGap(8);
  }
  // ...
}
```

Nothing here knows about grayscale - that is the point of applying the attribute
at the container. The price labels drawn in `#F34D4F` go gray with everything
else.

### The whole-window alternative (from the window reference, for HW-19-0123)

Not present in the sample. The reference form, adapted for `EntryAbility`:

```ts
if (canIUse('SystemCapability.Window.SessionManager')) {
  try {
    windowClass.setWindowGrayScale(1.0)
      .then(() => { hilog.info(0x0000, 'Grayscale', 'window grayscale set'); })
      .catch((err: BusinessError) => {
        hilog.error(0x0000, 'Grayscale', `failed: ${err.code} ${err.message}`);
      });
  } catch (exception) {
    hilog.error(0x0000, 'Grayscale', `failed: ${(exception as BusinessError).code}`);
  }
}
```

## Permissions & config

**No permissions are required** and none are declared. Neither `.grayscale()` nor
`setWindowGrayScale` is permission-gated.

`PageGrayscale.zip#PageGrayscale/build-profile.json5`: `targetSdkVersion` and
`compatibleSdkVersion` are `6.0.0(20)`, with
`strictMode: { caseSensitiveCheck: true, useNormalizedOHMUrl: true }`.

`module.json5` declares `deviceTypes: ["phone", "tablet", "2in1"]`, a single
`EntryAbility` with the home skill, and an `EntryBackupAbility`.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later. `.grayscale(value: number)` has
  been available since the early API versions; the `Optional<number>` overload is
  API 18+; `setWindowGrayScale` is API 12+.
- **Value range.** `[0.0, 1.0]`. "A value less than 0.0 evaluates to the value
  0.0. A value greater than 1.0 evaluates to the value 1.0." The API 18+
  `Optional` overload treats `undefined` as `0.0`.
- **`.grayscale()` is component-scoped.** It covers the component and its
  children, nothing else.
- **`setWindowGrayScale` requires `SystemCapability.Window.SessionManager`** and
  "can be called only after loadContent() or setUIContent() is called". Error
  codes 401, 801, 1300002, 1300003.
- **Grayscale destroys colour-encoded meaning.** Selected states, error reds,
  status badges, and brand colours all collapse to similar grays; anything that
  communicates through colour alone needs a non-colour cue while gray.
- **Devices.** phone, tablet, 2in1 per `module.json5`; `setWindowGrayScale` also
  lists TV and wearable.

## Pitfalls

- **The document presents a per-component attribute as the solution to an
  application-wide requirement, which is incorrect;** for "all pages" use
  `window.setWindowGrayScale`, and keep `.grayscale()` for a single subtree.
  (HW-19-0123)
- **`setTimeout` is called without keeping its ID, which is incorrect** - the
  reference says the timer is deleted "after callback execution, or you may
  manually delete it via the clearTimeout() API", and the callback writes to
  `@State` on a component that may already be gone. Store the ID and clear it in
  `aboutToDisappear`. (HW-19-0124)
- **`TabBar` declares `@Link` for a value it only reads, which is incorrect;**
  `@Prop` is the documented one-way decorator and prevents an accidental write in
  the child from silently re-rendering the parent. (HW-19-0125)
- **The document's snippet omits `this.isGrayedOut = true`, which is incorrect** -
  without it the selected tab keeps its red highlight through the filter. The
  printed braces are also misaligned. (HW-19-0126)
- **Do not ship the three-second timer.** It is a demonstration trigger. A real
  observance is driven by a server flag or a date range, and must be switchable
  off without an app update.
- **Grayscale is not a theme.** It is a render-time filter on top of whatever
  colours the page already has - it does not replace dark-mode or high-contrast
  handling, and it stacks with them.

## References

- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-image-effect.md` -
  `grayscale` (the component-scope statement, the value range, and the API 18+
  `Optional<number>` overload).
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-attributes-image-effect
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `setWindowGrayScale12+`, the window-level mechanism the document omits.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/05_common-capabilities/js-apis-timer.md` -
  `setTimeout` / `clearTimeout` and the shared ID pool note.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-timer
- `documentation/harmonyos-guides/03_application-framework/arkts-prop.md` and
  `arkts-link.md` - one-way versus two-way synchronization.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-prop
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/page_grayscale-0000002368995430
