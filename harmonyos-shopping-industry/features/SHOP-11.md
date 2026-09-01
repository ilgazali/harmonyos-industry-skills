---
id: SHOP-11
title: Skeleton screen - one wrapper component that swaps a shimmering placeholder for the real content
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/11_skeleton_screen.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/skeleton_screen-0000002294856764
sample: huawei_industry_tree/16_shopping/downloads/SkeletonScreen.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: ["@ComponentV2", "@Param", "@Local", "@BuilderParam", "UIContext.animateTo", AnimateParam, iterations, translate, linearGradient, clip, borderRadius, List, ListItem, ForEach, setTimeout, "window.getWindowAvoidArea", "AppStorage.setOrCreate", hilog]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-16-0012, HW-16-0027]
status: verified-with-fixes
---

## When to use

Load this card when a screen has **a known layout but unknown data** and you
want to show its shape while the data loads, instead of a spinner or a blank
page. The classic cases are a coupon wallet, an order list, a profile header, a
feed - anywhere the number and size of the boxes is fixed by the design even
before the content arrives.

The idea this sample gets right is that a skeleton is not a second screen. There
is no `SkeletonCouponPage` mirroring `CouponPage`; there is one wrapper
component, `ToggleCom`, that takes the real content as a trailing builder
closure and renders **either** that closure **or** a grey shimmering block. The
placeholder has no dimensions of its own - it fills its parent at `100%` - so
the *host* container that already sizes and rounds the real content also sizes
and rounds the placeholder. Add a skeleton to a new element by wrapping it; no
placeholder layout is ever written twice.

That generalises directly. Any framework with a slot mechanism can host this
pattern, and the `@BuilderParam` + `@Param` pair is ArkUI's version of it. The
part you must not copy verbatim is the shimmer animation itself - as shipped it
does not loop (`HW-16-0012`).

## Feature checklist

- On entry, the coupon page renders its full layout as grey rounded blocks: back
  button, title, search field, button, section label, and two coupon cards each
  broken into five separate blocks.
- Every block is the exact size and corner radius of the content it stands in
  for, because it inherits both from its host container.
- A soft white band sweeps left to right across each block.
- After the (simulated) load completes, every block is replaced by its real
  content in place, with no layout shift.

## Architecture

One `entry` module, one page, one component, one data file.

```
entry/src/main/ets
├── components/ToggleCom.ets        the wrapper: placeholder or slot, 71 lines
├── constant/couponData.ets         CouponListItem + COUPON_DATA (2 coupons)
├── entryability/EntryAbility.ets   full screen, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
└── pages/CouponPage.ets            @Entry, the layout - 13 ToggleCom sites
```

The documented tree matches the zip.

**The design decision worth copying** is `ToggleCom`'s interface: two things in,
nothing out.

```typescript
@Param loading: boolean = false;
@BuilderParam ui: () => void = this.customBuilder;
```

`@BuilderParam` accepts a trailing closure at the call site, so wrapping an
existing element means adding two lines around it and changing nothing inside:

```typescript
Row() {
  ToggleCom({ loading: this.loading }) {
    Text(item.title).fontWeight(500).fontSize(16);
  };
}
.width(116).height(18).borderRadius(4);
```

The `Row` is doing double duty - it is the frame for the real `Text` *and* the
frame for the placeholder. `ToggleCom` renders at `width('100%')`/`height('100%')`
either way, so both branches occupy exactly the same rectangle and the swap
causes no reflow. Note the `.clip(true)` on the host wherever the placeholder is
rounded: the sweeping band is a full-height rectangle and would otherwise paint
over the rounded corners.

The one thing worth fixing before copying: the parameter is called `loading` but
`true` means **loaded**. `CouponPage` starts at `false`, shows placeholders, and
sets it to `true` when data arrives. Rename it `loaded`, or invert it - as it
stands every call site reads backwards.

## Implementation steps

1. **Write one wrapper, not a second screen.** `@Param loading` plus
   `@BuilderParam ui`, with `build()` reduced to a single `if`.
2. **Give `@BuilderParam` a default** (`this.customBuilder`, an empty `@Builder`)
   so the component is still constructible without a trailing closure.
3. **Let the placeholder fill its parent.** `width('100%')`, `height('100%')`,
   a `backgroundColor` of `rgba(0, 0, 0, 0.1)` and `clip(true)`; no fixed sizes
   anywhere inside the component.
4. **Size and round on the host container**, and repeat `clip(true)` there when
   the corners are rounded.
5. **Build the shimmer as a gradient on an empty `Text`** rather than an image:
   a 90-degree `linearGradient` from transparent to 70% white and back.
6. **Start the sweep off-screen left** (`translateX = '-100%'`) and animate it to
   `'100%'`, both values relative to the block's own width.
7. **Put the `translateX` assignment inside the `animateTo` closure**
   (`HW-16-0012`). The shipped code assigns it before the call and passes an
   empty closure, so nothing is animated and the band jumps to its end position.
8. **Loop with `iterations: -1`,** and use `Curve.EaseInOut` so the band eases in
   and out at each pass instead of scrolling at constant speed.
9. **Flip the flag once, at the top level.** `CouponPage` owns the boolean; all
   thirteen `ToggleCom`s read it, so the whole page resolves in one frame.

## Verified snippets

All snippets are from `SkeletonScreen.zip`. Corrected forms are marked.

**The wrapper — `entry/src/main/ets/components/ToggleCom.ets`** (corrected, see `HW-16-0012`)

```typescript
@ComponentV2
export struct ToggleCom {
  @Local translateX: string = '-100%';
  @Local duration: number = 2000;
  @Local delay: number = 0;
  @Param loading: boolean = false;
  @BuilderParam ui: () => void = this.customBuilder;

  @Builder
  textBuild() {
    Column() {
      Text()
        .width('100%')
        .height('100%')
        .translate({ x: this.translateX })
        .onAppear(() => {
          this.getUIContext()?.animateTo({
            duration: this.duration,
            delay: this.delay,
            iterations: -1,
            curve: Curve.EaseInOut
          }, () => {
            this.translateX = '100%';   // FIX: the sample assigns this before the call
          });                           //      and passes an empty closure
        })
        .linearGradient({
          angle: 90,
          colors: [
            ['rgba(255, 255, 255, 0)', 0],
            ['rgba(255, 255, 255, 0.7)', 0.5],
            ['rgba(255, 255, 255, 0)', 1]
          ]
        });
    }
    .height('100%')
    .width('100%')
    .backgroundColor('rgba(0, 0, 0, 0.1)')
    .clip(true);
  }

  build() {
    if (this.loading) {
      this.ui();
    } else {
      this.textBuild();
    }
  }

  @Builder
  customBuilder() {
  }
}
```

**`animateTo` only animates what changes inside its closure.** The reference is
explicit: it "adds transition animations for state changes in closure code", and
the system inserts the transition when the state changes *in the closure
function*. The shipped code sets `this.translateX = '100%'` on the line before
and hands `animateTo` an empty body, so there is no state change to animate:
`iterations: -1` loops nothing, and the band simply appears at its end position
and stays there. Moving the single assignment inside the closure is the whole
fix.

**The three parts of the visual.** The grey base is the `Column`'s
`rgba(0, 0, 0, 0.1)` - a translucent black, so the placeholder tints whatever the
page background is instead of fighting it. The band is an empty `Text` with no
content at all, carrying only a `linearGradient` at 90 degrees whose middle stop
is 70% white and whose ends are fully transparent - a soft highlight, not a hard
edge. And `translate` with **percentage** values means the sweep is relative to
each block's own width, so a 40 vp avatar and a 300 vp title bar both take
`duration` ms to complete one pass regardless of size.

`clip(true)` on the `Column` keeps the band inside the block while it is at
`-100%` and `100%`; without it the highlight would be drawn outside the
placeholder for most of the cycle.

**A call site — `entry/src/main/ets/pages/CouponPage.ets`** (as shipped, trimmed)

```typescript
Row() {
  Row() {
    ToggleCom({
      loading: this.loading
    }) {
      Column() {
        Text(item.money)
          .margin({ bottom: 5 })
          .fontColor('#f9f4d7')
          .fontSize(24);
        Text(item.text)
          .textAlign(TextAlign.Center)
          .fontSize(12)
          .fontColor('#f9f4d7');
      }
      .height('100%')
      .width('100%')
      .backgroundColor('rgb(10, 89, 247)')
      .justifyContent(FlexAlign.Center);
    };
  }
  .clip(true)
  .borderRadius({ topLeft: 16, bottomLeft: 16 })
  .width(100)
  .height(110);
  // ... the right-hand half of the coupon card
}
```

**Every geometric property lives on the host `Row`, not on the content and not
on the placeholder.** The 100x110 stub with two rounded left corners is the
coupon's blue value block; while loading it is a grey 100x110 rectangle with the
same two rounded corners, because `ToggleCom` fills it. `clip(true)` appears here
for the same reason it appears inside the component - the sweep must respect the
asymmetric radius.

The coupon card is wrapped **five times**, not once: value block, title, date,
limit text, and the redeem button each get their own `ToggleCom`. That is
deliberate and it is what makes the skeleton read as a card rather than as one
grey slab. It also means five independent shimmer animations start on the same
frame, all with `delay: 0` - staggering them by index would look better and is a
one-line change since `delay` is already a field.

**The load flip — same file** (as shipped)

```typescript
@Local listDate: CouponListItem[] = COUPON_DATA;
@Local loading: boolean = false;

aboutToAppear(): void {
  this.startAnimation();
}

startAnimation() {
  setTimeout(() => {
    this.loading = true;
  }, 3000);
}
```

One boolean at the top of the page, read by every wrapper below it. In a real
app this is the `.then()` of the request, and the important property is that it
resolves the whole page at once: partial reveal, where some blocks are content
and others are still shimmering, looks broken. Since the coupon data is a static
constant, `listDate` is already populated before the timer fires and the three
seconds are pure theatre - the boolean is the only thing that changes.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`.

`EntryAbility` forces light mode, sets
`setWindowLayoutFullScreen(true)`, and writes the `TYPE_SYSTEM` and
`TYPE_NAVIGATION_INDICATOR` avoid-area heights into `AppStorage` as
`topRectHeight` / `bottomRectHeight`, refreshing them from an `avoidAreaChange`
subscription. **`CouponPage` never reads either value** - it uses a hardcoded
`.margin({ top: 44 })` on its header row instead. So the ability-side plumbing is
dead code here, and the header is only correctly placed on a device whose status
bar happens to be 44 vp. Either consume the two keys with `@StorageProp` (as
`SHOP-09` does) or drop the plumbing and use `expandSafeArea` (as `SHOP-10`
does).

The `avoidAreaChange` listener is also never released in `onWindowStageDestroy`
- the same omission as the sibling shopping samples.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The load is a three-second `setTimeout` over a static two-entry array; there is
  no request, no error path and no timeout. A real integration needs a third
  state (failed) that the two-branch `if` in `ToggleCom` cannot express.
- `@Param loading` means "loaded". Every call site reads inverted.
- All thirteen shimmer animations start together with `delay: 0`; the `delay`
  field exists but is never set by any caller because `ToggleCom` declares it
  `@Local` rather than `@Param`, so it is not reachable from outside.
- The placeholder colour, gradient stops and 2000 ms duration are hardcoded in
  the component. Only `duration` and `delay` are fields, and neither is settable.
- Coupon rows are rendered with `ForEach` and **no key generator**, so keys are
  index-based; fine for a static list, wrong once coupons can be redeemed and
  removed.
- `CouponPage` is `@ComponentV2` and so is `ToggleCom`; the `@Param`/`@Local`
  pairing is required for the flag to propagate. Wrapping V1 `@State` around a
  V2 `@Param` in this position will not update.

## Pitfalls

- **`HW-16-0012`** (C/medium, confirmed): `ToggleCom.textBuild` assigns
  `this.translateX = '100%'` immediately before `animateTo` and passes an empty
  closure, so per the documented contract nothing is animated - the infinite
  shimmer sweep the page promises never runs and the highlight band simply sits
  at its end position. Fix: move the assignment inside the closure -
  `this.getUIContext().animateTo({ duration: this.duration, delay: this.delay, iterations: -1, curve: Curve.EaseInOut }, () => { this.translateX = '100%'; });`

## References

- `huawei_industry_tree/16_shopping/docs/11_skeleton_screen.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/skeleton_screen-0000002294856764
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `UIContext.animateTo` and the closure contract
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-references/02_application-framework/ts-explicit-animation.md` - `AnimateParam`, `iterations`, `curve`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-explicit-animation
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-gradient-color.md` - `linearGradient` angle and colour stops
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-gradient-color
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-transformation.md` - `translate` with percentage values
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-transformation
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sharp-clipping.md` - `clip` and why the band needs it
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sharp-clipping
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - the coupon `List`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-guides/03_application-framework/arkts-new-param.md` - `@Param` and V2 propagation
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-param
- `SHOP-10` - the same `animateTo` API used correctly, with both writes inside the closure
