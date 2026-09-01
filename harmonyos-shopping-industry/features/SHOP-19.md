---
id: SHOP-19
title: Rolling sales counters - a 0-9 column per digit, clipped, moved by animateTo
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/19_sales_update_roll.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/sales_update_roll-0000002380510193
sample: huawei_industry_tree/16_shopping/downloads/SalesUpdateRoll.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.CryptoArchitectureKit", "@kit.PerformanceAnalysisKit"]
apis: ["UIContext.animateTo", Curve, translate, clip, ForEach, "@Link", "@State", "@StorageProp", setInterval, setTimeout, clearInterval, "cryptoFramework.createRandom", generateRandomSync, "window.getWindowAvoidArea", setWindowLayoutFullScreen, "UIContext.px2vp", Grid, GridItem, List, hilog]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-16-0022, HW-16-0027]
status: verified-with-fixes
---

## When to use

Load this card for **a number on screen that changes and should visibly count,
not just replace itself** - a live sales total on a merchant dashboard, an
order counter, a visitor count, a leaderboard score, an odometer.

The technique is older than any framework and does not need a bespoke
animation API: render one vertical strip of the digits 0-9 per decimal place,
clip each strip to one digit's height, and translate the strip so the wanted
digit lands in the window. Animating that translation is what produces the
mechanical roll. Nothing tweens a number; the animation is pure layout.

It generalises to any small fixed alphabet you want to spin between - digits
here, but the same component shape does a slot-machine reel, an hour/minute
column in a custom time picker, or a unit switcher. What it does *not* do is
scale: one strip per digit means ten `Text` nodes per digit, so a 12-digit
counter is 120 text nodes. Above roughly six digits, render the number once and
cross-fade instead.

## Feature checklist

- A merchant home page with a header card, a three-metric sales card, an order
  service grid and a recommended-product list.
- The sales card shows three independently rolling numbers: revenue (with a
  currency prefix and a unit suffix), order count, and visitors.
- Each metric refreshes every 3 seconds with a fresh secure random increment,
  and each digit rolls to its new value on a decelerating curve.
- The three metrics grow at different rates - 100, 5 and 1000 per tick
  respectively.
- Only the wanted digit is visible; the other nine in each strip are clipped
  away.
- The last digit is preceded by a separator, rendering the revenue figure as a
  one-decimal value.
- Refreshing stops after 10 minutes, and the interval is cleared when the
  component is destroyed.

## Architecture

One `entry` module. The rolling counter is one small component; the rest of the
tree is the merchant page it lives on.

```
entry/src/main/ets
├── components
│   ├── DigitalScrollDetail.ets   the rolling counter - the whole feature, 133 lines
│   ├── SalesDetail.ets           the three-metric card; instantiates the counter 3x
│   ├── MerchantTop.ets           avatar + shop name header card
│   ├── ServiceList.ets           2x5 Grid of order-status icons
│   └── ProductView.ets           recommended-product List
├── entryability/EntryAbility.ets full screen, both avoid areas -> AppStorage
├── entrybackupability/
├── model/ConstData.ets           DATA_CONFIG, STYLE_CONFIG, the product and service lists
├── pages/Merchant.ets            @Entry, a Column of the four cards
└── utils/Logger.ets              hilog wrapper
```

**The documented tree names the directory `component`; the zip has
`components`** (`HW-16-0022`). Every import in the sample is
`'../components/...'`, so a reader following the document creates a directory
whose imports do not resolve.

**The design decision worth copying** is that `DigitalScrollDetail` owns its
own timer and its own data. It takes exactly one input:

```typescript
@Link rangeValue: number;   // how much this metric may grow per tick
```

and `SalesDetail` instantiates it three times with 100, 5 and 1000. The parent
does not tick, does not hold the numbers, and does not coordinate the three
animations. That is why the card reads as nine lines of layout. The trade is
three independent `setInterval`s that drift apart rather than one that refreshes
all three in step - acceptable for decorative counters, wrong if the three
numbers must be mutually consistent (revenue must not update before the order
count that produced it). If they must, hoist the timer to the parent and pass
values down.

`@Link` is a slightly odd choice for a value the child never writes -
`@Prop` would express the one-way flow - but it is harmless because the source
values in `SalesDetail` are constants.

## Implementation steps

1. **Model the number as `number[]`, one entry per decimal place.** Split with
   `total.toString()` and `parseInt(char)`; do not carry the number itself into
   the render path.
2. **Keep a parallel `scrollYList: number[]` of Y offsets,** indexed the same
   way. This is the only state the animation touches.
3. **Render a `Row` of `Column`s.** Outer `ForEach` over the digits, inner
   `ForEach` over the fixed `[0..9]`, one `Text` per numeral.
4. **Give the `Column` a fixed height of one digit and `.clip(true)`** - this
   is what turns a ten-digit strip into a one-digit window. Without the clip
   you see the whole strip and the effect is gone.
5. **Translate every `Text` in a strip by the same `scrollYList[index]`,** not
   the `Column`. Moving the children keeps the container's layout box fixed at
   one digit's height so the clip stays anchored.
6. **Compute the offset as `-digit * ITEM_HEIGHT`,** and make `ITEM_HEIGHT`
   the same constant the `Column` height uses. If those two drift, every digit
   sits a fraction off.
7. **Wrap the offset writes in `animateTo` with `Curve.LinearOutSlowIn`** so
   the strip decelerates into position like a mechanical counter.
8. **Drive the ticks with `setInterval`, and clear it in `aboutToDisappear`**
   - the sample does this correctly, unlike `SHOP-17` (`HW-16-0021`).
9. **Import the component from `components/`, not `component/`**
   (`HW-16-0022`).

## Verified snippets

All snippets are from `SalesUpdateRoll.zip`. Corrected forms are marked.

**The strip — `entry/src/main/ets/components/DigitalScrollDetail.ets`** (as shipped)

```typescript
@State scrollYList: number[] = [];                    // one Y offset per decimal place
@State currentData: number[] = new Array(2).fill(1);  // the digits, most significant first
private dataItem: number[] = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9];

build() {
  Row() {
    // 性能知识点：数据量小并且可确定的场景，使用ForEach进行循环渲染。
    ForEach(this.currentData, (item: number, index: number) => {
      if (index === this.currentData.length - 1 && index !== 0) {
        Text($r('app.string.digital_scroll_animation_comma'))   // resource value is "."
          .fontColor(Color.Black)
      }
      Column() {
        ForEach(this.dataItem, (subItem: number) => {
          Text(subItem.toString())
            .fontSize(16)
            .fontColor('#0A59F7')
            .height($r('app.string.digital_scroll_animation_max_size'))   // "100%"
            .textAlign(TextAlign.Center)
            .translate({ x: 0, y: this.scrollYList[index] })
        })
      }
      .alignItems(HorizontalAlign.Start)
      .height(STYLE_CONFIG.ITEM_HEIGHT)   // 26 - one digit tall
      .clip(true)                         // 裁剪超出容器的视图 - the window
    })
  }
}
```

**Three lines carry the whole illusion.** `.height(ITEM_HEIGHT)` on the
`Column` fixes the viewport at exactly one digit. `.clip(true)` hides the other
nine, and it must be on the `Column` - putting it on the `Row` would clip the
row's edges instead and leave every strip fully visible. `.translate` on the
`Text` children rather than on the `Column` is what keeps the container's own
layout box still: a translated container would move its clip region with it and
nothing would ever scroll past the window.

Each `Text` is `height('100%')` - 100% of the 26 vp `Column` - so ten stacked
children make a 260 vp strip, and an offset of `-4 * 26 = -104` brings the `4`
into view. The digit height therefore exists in two places (the `Column`'s
height and `STYLE_CONFIG.ITEM_HEIGHT` used to compute the offset); they are the
same constant here, and they must stay that way.

The separator is named `digital_scroll_animation_comma` but its resource value
is `"."` and it is emitted before the *last* digit, so the counter reads as a
one-decimal figure - `1234` renders `123.4`. That is deliberate for a revenue
figure in 万元, not a mis-placed thousands separator. `DATA_CONFIG.MILLENNIAL_LEN`
(3), which would be the thousands grouping, is declared and never used.

**The tick — same file** (as shipped)

```typescript
aboutToAppear() {
  this.uiContext = this.getUIContext();
  if (!this.uiContext) {
    return;
  }
  this.timerID = setInterval(() => {
    this.refreshData();          // every 3 s
  }, 3000);

  setTimeout(() => {             // stop after 10 minutes
    clearInterval(this.timerID);
  }, 600000);
}

aboutToDisappear() {
  clearInterval(this.timerID);
}

refreshData() {
  const tempArr: number[] = [];
  for (let i = 0; i < this.currentData.length; i++) {
    this.total = this.total + (this.currentData.length - i - 1) * 10 * this.currentData[i];
  }
  this.total = this.total + Math.floor(getSecureRandomFloat() * this.rangeValue);
  let str: string = this.total.toString();
  for (let char of str) {
    tempArr.push(parseInt(char));         // 将字符转换为数字存入数组
  }
  this.currentData = tempArr;             // 更新当前数据
  this.currentData.forEach((item: number, index: number) => {
    this.uiContext?.animateTo({
      duration: DATA_CONFIG.DURATION_TIME,   // 500 ms
      curve: Curve.LinearOutSlowIn,          // 减速曲线
    }, () => {
      this.scrollYList[index] = -item * STYLE_CONFIG.ITEM_HEIGHT;
    });
  });
}
```

**One `animateTo` per digit, not one for the whole number.** They are issued in
the same turn with the same duration and curve, so they run together - but each
carries its own closure writing its own array slot, which is what lets a digit
that did not change stay still while its neighbour rolls. A single `animateTo`
around the whole `forEach` would work identically here; the per-digit form
matters once you stagger the durations to make higher places settle later, which
is the usual polish on this effect.

`Curve.LinearOutSlowIn` is the right family: it leaves fast and arrives slowly,
which reads as a mechanism settling. An ease-in-out would read as a fade.

**The recomposition loop is vestigial and inflates the number.** `this.total`
already holds the running value; the first loop then adds a second,
digit-weighted quantity on top of it every tick - and the weight is
`(len - i - 1) * 10`, a linear factor, not the `10^(len-i-1)` that
digit-to-number reconstruction would need. The document describes the intent
plainly - 将随机数与原数据相加生成新数据 ("add a random number to the existing
value") - which is the second statement alone. Dropping the loop leaves
`this.total += Math.floor(getSecureRandomFloat() * this.rangeValue)` and the
counter grows at the advertised rate instead of accelerating. No finding was
filed against this; the visible behaviour (a number going up) is the same, only
faster than intended.

Note also that `scrollYList` is only ever written by index. As `total` gains a
digit the array grows, but it never shrinks, so a counter that could lose a
digit would keep a stale offset. Here the value is monotonic, so it does not
bite.

**Secure randomness — same file** (as shipped)

```typescript
import { cryptoFramework } from '@kit.CryptoArchitectureKit';

function getSecureRandomFloat(): number {
  let rand = cryptoFramework.createRandom();
  let len = 4;
  try {
    let randData = rand.generateRandomSync(len);
    let bytes = new Uint8Array(randData.data);
    let buffer = bytes.buffer;
    let intValue = new DataView(buffer).getUint32(0, true);   // little-endian
    return intValue / 0xFFFFFFFF;                             // 0-1 float
  } catch (error) {
    Logger.error('Generate random failed: ' + error);
    return 0;
  }
}
```

**This is `Math.random()` with four extra lines and a kit dependency**, and it
is here because HarmonyOS sample code is expected to use `cryptoFramework` for
randomness rather than the JS engine's PRNG - the platform guidance treats
`Math.random()` as non-secure. For a decorative counter that is theatre, but
the helper is worth keeping as a reference implementation: four bytes,
`getUint32` with `littleEndian: true`, divided by `0xFFFFFFFF` to normalise.

The `catch` returning `0` is the right failure mode for this call site - the
counter stops growing rather than throwing inside a timer callback, where an
exception would be unhandled.

**Wiring three counters — `entry/src/main/ets/components/SalesDetail.ets`** (as shipped)

```typescript
@State rangeValue: number = 5;      // orders
@State rangeValue1: number = 100;   // revenue
@State rangeValue2: number = 1000;  // visitors

Flex({ justifyContent: FlexAlign.SpaceBetween, alignItems: ItemAlign.Center }) {
  Column() {
    Row() {
      Text($r('app.string.money'))                      // ￥
      DigitalScrollDetail({ rangeValue: this.rangeValue1 })
      Text($r('app.string.unit'))                       // 万
    }
    Text($r('app.string.amount'))                       // 销售额
  }
  // ... two more Columns for orders and visitors
}
.width('84%')
```

The counter composes inline with ordinary `Text` because it renders as a `Row`
with no width of its own - it is as wide as its digits, so a prefix and a suffix
sit flush against it and the group grows rightwards as the number does. That is
the reason `DigitalScrollDetail` sets no width and only a bottom margin; give it
a fixed width and the currency symbol detaches.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

`deviceTypes` is `phone`, `tablet`, `2in1`. The layout is percentage-based
(`92%` cards inside a full-width `Column`), so it survives the wider forms; the
counter itself is width-agnostic.

`EntryAbility` sets `setWindowLayoutFullScreen(true)` with a proper
`.catch((err: BusinessError) => ...)`, then publishes both avoid areas:

```typescript
AppStorage.setOrCreate('bottomRectHeight', avoidArea.bottomRect.height);
AppStorage.setOrCreate('topRectHeight', avoidArea.topRect.height);
```

`Merchant` reads only the top one, with `@StorageProp('topRectHeight')` and a
typed default of `0`, converting at the point of use with `px2vp`. That is the
correct read-only shape (compare `SHOP-18`, which uses `@StorageLink` for the
same read-only purpose). Note that all of this happens inside the
`loadContent` callback and there is no `avoidAreaChange` subscription, so the
padding is fixed at the value present when the page loaded.

Numeric layout values live in `model/ConstData.ets` as two `Record<string,
number>` maps (`DATA_CONFIG`, `STYLE_CONFIG`), which is the right place for
them - but `NUMBER_LEN` (7) and `MILLENNIAL_LEN` (3) are declared and never
read.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Ten `Text` nodes per digit. Three counters at three to five digits each is
  roughly 100-150 text nodes updating every 3 s. Fine; a longer number is not.
- The counter only goes up, and only ever gains digits. Nothing in the
  component handles a decreasing value or a shrinking digit count -
  `scrollYList` would keep a stale trailing offset.
- Each instance owns its own `setInterval`, so the three metrics drift out of
  step within a minute. Hoist the timer if the numbers must be consistent.
- The 10-minute stop is a `setTimeout` that is never cleared in
  `aboutToDisappear`. Leaving the page leaves three timers pending for up to
  ten minutes; each only calls `clearInterval` on an already-cleared id, so it
  is harmless, but it is not cleanup.
- Everything below the counter is static: the service grid, the product list
  and the header card are hardcoded in `ConstData.ets` and have no handlers.
  `Merchant.ets` carries a `// 下拉刷新组件` (pull-to-refresh component)
  comment above `MerchantTop()`, which is not a refresh component.
- `Logger` in `utils/` announces itself with the prefix `'Base64ImageToAlbum'`,
  carried over from an unrelated sample. Every log line from this app is
  labelled with another app's name.

## Pitfalls

- **`HW-16-0022`** (E/low, confirmed): the project tree names the directory
  `component` while the zip and every import use `components`. Fix: regenerate
  the tree entry as `components`.
- **The digit-recomposition loop in `refreshData` double-counts.**
  `this.total` already carries the value, and the loop adds
  `(len - i - 1) * 10 * digit` on top of it each tick - a linear weight where
  positional reconstruction would need `10^(len-i-1)`. The counter accelerates
  instead of growing by `rangeValue`. No finding filed (the visible effect is
  still "a number going up"); delete the loop when reusing the component.
- **`ITEM_HEIGHT` is duplicated implicitly.** The `Column` height and the
  offset multiplier must be the same value; they are here, but nothing enforces
  it, and a mismatch shows as every digit sitting a few vp off centre with no
  obvious cause.
- **`.clip(true)` must be on the digit `Column`.** Moving it up to the `Row`
  or omitting it renders all ten numerals and there is no error to tell you
  why.
- **`@Link rangeValue` should be `@Prop`.** The child never writes it; `@Link`
  advertises a two-way channel that does not exist.
- **The 10-minute `setTimeout` is not cleared on destroy** - see Constraints.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `UIContext.animateTo` and `AnimateParam`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-references/02_application-framework/ts-explicit-animation.md` - explicit animation and the curve families
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-explicit-animation
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-transformation.md` - `translate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-transformation
- `documentation/harmonyos-guides/04_system/crypto-generate-random-number.md` - `cryptoFramework.createRandom` and `generateRandomSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/crypto-generate-random-number
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - `ForEach` and its keying rules
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `huawei_industry_tree/16_shopping/docs/19_sales_update_roll.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sales_update_roll-0000002380510193
- `SHOP-17` - the counter-example on timer and animation cleanup (`HW-16-0021`)
