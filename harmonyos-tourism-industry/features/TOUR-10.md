---
id: TOUR-10
title: Hotel order list - a custom sliding-underline tab bar over status-filtered orders
industry: 09_tourism
doc: huawei_industry_tree/09_tourism/docs/10_travel_checkin_order.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/travel_checkin_order-0000002386506221
sample: huawei_industry_tree/09_tourism/downloads/TravelCheckinOrder.zip
kits: ["@kit.ArkTS", "@kit.BasicServicesKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit"]
apis: [Tabs, TabsController, changeIndex, onAnimationStart, onAnimationEnd, TabsAnimationEvent, List, Scroller, scrollToIndex, onScrollFrameBegin, onAreaChange, "UIContext.animateTo", "@Observed", "@ObjectLink", "@Link", "@Watch", "@State", "@StorageProp", ForEach, "ArrayList", "Intl.DateTimeFormat", "@Extend"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-09-0060, HW-09-0061, HW-09-0062, HW-09-0063, HW-09-0064, HW-09-0065, HW-09-0080]
status: verified-with-fixes
---

## When to use

Load this card for an **order or booking list filtered by status**, where each
row's available actions depend on which status it is in and acting on a row
moves it between filters. Hotel bookings here; the shape is identical for
flight orders, tickets, deliveries, service requests or returns.

Two things are worth taking:

- **A hand-built tab bar with a sliding underline.** `Tabs` with
  `barHeight(0)` plus a horizontal `List` of labels plus one `Column` used as
  the indicator, animated from `onAnimationStart` / `onAnimationEnd` so the
  underline tracks the page transition rather than jumping after it. This is
  the most complete custom-tab implementation in the industry corpus.
- **State-driven action buttons.** Each row renders its button set from its
  own status, so "pay now", "close", "refund", "review", "rebook" are `if`
  branches over one enum instead of per-row configuration.

For the reverse direction - how the review flow returns a status change to
this list - see `TOUR-08`.

## Feature checklist

- Four filter tabs: all, awaiting payment, awaiting check-in, awaiting review.
- Tapping a tab or swiping the content switches the filter; the underline
  slides between labels in step with the page animation.
- After a switch the bar scrolls so the previous tab stays visible.
- Each order card: hotel logo and name, status, room photo, room type, stay
  dates, and the amount labelled "to pay" or "paid" by status.
- Action buttons per status - delete a closed order, close or pay an unpaid
  one, refund an upcoming stay, review a completed one, rebook anything not
  awaiting payment.
- Acting on an order updates its status in place and re-filters the tab.
- Deleting or rebooking re-sorts the list by creation time, newest first.
- An empty state with an icon and message when a filter matches nothing.

## Architecture

One `entry` module, one page, one view file holding two components.

```
entry/src/main/ets
├── constants/CommonConstants.ets   animation duration, underline width, paddings
├── entryability/EntryAbility.ets   full screen, avoid areas -> AppStorage as vp
├── model
│   ├── CommonEnum.ets              OrderType (filter) + OrderStatus (state)
│   └── DataModel.ets               @Observed Order, CUSTOM_TABS, ORDER_SAMPLES
├── pages/OrderPage.ets             @Entry - the tab bar, the indicator, the Tabs
└── views/OrderView.ets             OrderView (one filter) + OrderContent (one card)
```

The documented tree matches the zip.

**Two enums, two jobs.** `OrderType` is the *filter* (ALL, PAYING, STAYING,
COMMENTING) and drives which tab is which. `OrderStatus` is the *order state*
(CLOSED, PAYING, STAYING, COMMENTING, FINISHED) and drives the buttons. They
deliberately do not match: there are five states and four filters, because
CLOSED and FINISHED orders only appear under ALL.

**The change signal travels up by `@Link`, the data down by `@ObjectLink`:**

```
OrderPage  ──@Link selectedIndex──>  OrderView  ──@ObjectLink order──>  OrderContent
                                         ^                                   │
                                         └────── @Link change (toggled) ──────┘
```

`OrderContent` cannot re-filter the list - it does not own it - so it flips a
boolean the parent `@Watch`es. `isAddOrDelete` rides alongside to tell the
parent whether the change needs a re-sort or only a re-filter. It is a
serviceable pattern for V1 state management, and the reason it is needed is
that `@ObjectLink` propagates *property* changes but not membership changes to
the array behind the list.

`@Observed class Order` with `@ObjectLink order` in the card is what makes
`this.order.status = OrderStatus.CLOSED` repaint the status text, the amount
label and the whole button row of that one card, with no notification code.

## Implementation steps

1. **Model the filter and the state separately** - one enum each - and keep
   state tokens free of display text (`HW-09-0060`).
2. **Hide the built-in tab bar** with `barHeight(0)` and `scrollable(false)`,
   and build your own from a horizontal `List` in a `Stack` above it.
3. **Measure each label with `onAreaChange`** and cache its left edge and
   width in a `Map`, so the indicator can be positioned without a second
   layout pass.
4. **Position the indicator as a `Column`** with a left margin bound to state.
5. **Animate it from `onAnimationStart`** with the target index and stop it in
   `onAnimationEnd`, so the underline moves with the transition.
6. **Keep the bar scrolled** so the neighbouring tab stays visible - with a
   floor at 0 (`HW-09-0064`).
7. **Give each tab its own `OrderView`** with a fixed `orderTabType`, and
   filter from the shared list into a private array. Copy; do not alias
   (`HW-09-0061`).
8. **Render the action buttons as `if` branches on status**, and have each one
   write the new status and toggle the change flag.

## Verified snippets

All snippets are from `TravelCheckinOrder.zip`. Corrected forms are marked.

**Measuring the labels — `entry/src/main/ets/pages/OrderPage.ets`** (as shipped)

```typescript
private tabPosWidthMap: Map<string, Record<string, number>> = new Map();
private underlineDistance: number = 0;
@State indicatorLeftMargin: number = 0;

// inside the horizontal List, per label
.id(tab.index.toString())
.onAreaChange((oldValue: Area, newValue: Area) => {
  if (newValue.globalPosition.x !== undefined) {
    const width = Number.parseFloat(newValue.width.toString());
    const globalX = Number.parseFloat(newValue.globalPosition.x.toString());
    // first layout only: seed the indicator under the initially selected tab
    if (newValue.position.x !== undefined && this.selectedIndex === tab.index &&
      this.indicatorLeftMargin === 0) {
      const positionX = Number.parseFloat(newValue.position.x.toString());
      this.underlineDistance = (width - CommonConstants.UNDERLINE_WIDTH) / 2;   // centre it
      this.indicatorLeftMargin = Number.isNaN(positionX) ? 0 : positionX + this.underlineDistance;
    }
    this.tabPosWidthMap.set(tab.index.toString(), { 'left': globalX, 'width': width });
  }
})
```

**`onAreaChange` is how you get a label's geometry in ArkUI** - there is no
synchronous measure API for a declarative child. Caching `left` and `width`
per tab id turns every later indicator move into arithmetic instead of a
layout query. The `this.indicatorLeftMargin === 0` test is the "first layout"
guard: it seeds the indicator once and then never fights the animation.

`underlineDistance` centres a fixed-width underline inside a variable-width
label, computed once.

**The indicator and the animation — same file** (corrected, see `HW-09-0064`)

```typescript
Stack({ alignContent: Alignment.TopStart }) {
  List({ initialIndex: 0, scroller: this.scroller }) { /* the labels */ }
    .listDirection(Axis.Horizontal)
    .onScrollFrameBegin((offset: number, state: ScrollState) => {
      this.indicatorLeftMargin -= offset;      // drag the bar, the underline comes along
      return { offsetRemain: offset };
    })

  Column()                                     // the underline: an empty Column
    .width($r('app.float.order_tab_underline_width'))
    .height($r('app.float.order_tab_underline_height'))
    .backgroundColor($r('app.color.normal_blue_color'))
    .margin({ left: this.indicatorLeftMargin, top: $r('app.float.order_tab_bar_height') })

  Tabs({ barPosition: BarPosition.Start, controller: this.tabsController }) {
    ForEach(CUSTOM_TABS, (tab: OrderTabModel) => {
      TabContent() { OrderView({ orderTabType: tab, selectedIndex: this.selectedIndex }) }
    }, (tab: OrderTabModel) => tab.index.toString())
  }
  .barHeight(0)                                // hide the built-in bar
  .scrollable(false)
  .onChange((index: number) => {
    this.selectedIndex = index;
    this.scroller.scrollToIndex(Math.max(0, index - 1), true);   // FIX: sample passes index - 1
  })
  .onAnimationStart((index: number, targetIndex: number, event: TabsAnimationEvent) => {
    this.selectedIndex = targetIndex;
    const targetIndexInfo = this.getTextInfo(targetIndex.toString());
    this.startAnimateTo(CommonConstants.NORMAL_ANIMATE_DURATION, targetIndexInfo.left - this.stackPadding);
  })
  .onAnimationEnd((index: number, event: TabsAnimationEvent) => {
    const targetIndexInfo = this.getTextInfo(index.toString());
    this.startAnimateTo(0, targetIndexInfo.left - this.stackPadding);   // duration 0: snap, no bounce
  })
}

private startAnimateTo(duration: number, leftMargin: number) {
  this.uiContext.animateTo({ duration, curve: Curve.Linear, iterations: 1, playMode: PlayMode.Normal }, () => {
    this.indicatorLeftMargin = leftMargin + this.underlineDistance;
  });
}
```

**`onAnimationStart` is what makes it feel native.** Animating from `onChange`
would move the underline *after* the page had already swiped; starting it when
the transition starts, with the same duration, makes them one gesture.
`onAnimationEnd` re-runs the same move with duration 0, so an interrupted or
overshooting swipe still lands the underline exactly on the final tab.

`onScrollFrameBegin` handles the third case - the user dragging the bar itself
- by shifting the indicator by the same offset, so it stays glued to its
label.

**Filtering per tab — `entry/src/main/ets/views/OrderView.ets`** (corrected, see `HW-09-0061`)

```typescript
orderTabType: OrderTabModel = { index: -1, title: '', orderType: OrderType.ALL };
orderList: ArrayList<Order> = new ArrayList();
@State orderArray: Order[] = [];
@State @Watch('orderChange') change: boolean = true;
@State isAddOrDelete: boolean = false;
@Link @Watch('tabChange') selectedIndex: number;

aboutToAppear()  { this.filterOrder(); }
orderChange()    { this.filterOrder(); }
tabChange()      { if (this.orderTabType.index === this.selectedIndex) { this.filterOrder(); } }

filterOrder() {
  // sort only when membership changed; ordering depends solely on creation time
  if (this.isAddOrDelete) {
    ORDER_SAMPLES.sort((first: Order, second: Order) => second.createTime - first.createTime);
  }
  this.orderList.clear();                                   // FIX: sample aliases on the ALL branch
  ORDER_SAMPLES.forEach((order) => {
    if (this.orderTabType.orderType === OrderType.ALL ||
      (this.orderTabType.orderType === OrderType.PAYING     && order.status === OrderStatus.PAYING) ||
      (this.orderTabType.orderType === OrderType.STAYING    && order.status === OrderStatus.STAYING) ||
      (this.orderTabType.orderType === OrderType.COMMENTING && order.status === OrderStatus.COMMENTING)) {
      this.orderList.add(order);
    }
  });
  this.orderArray = this.orderList.convertToArray();        // @State array: assignment repaints
}
```

**`tabChange` guarding on `this.orderTabType.index === this.selectedIndex`** is
the detail that keeps this cheap: all four `OrderView` instances share the
`@Link selectedIndex` and all four `@Watch` handlers fire on a switch, but only
the one becoming visible actually re-filters.

`convertToArray()` at the end is deliberate - `ArrayList` is the working
collection, but `ForEach` needs a plain array, and assigning a fresh one to
`@State` is what triggers the render.

**Status-driven actions — same file** (as shipped)

```typescript
@Component
struct OrderContent {
  @ObjectLink order: Order;
  @Link change: boolean;
  @Link isAddOrDelete: boolean;

  @Builder
  orderButtons() {
    Row({ space: 8 }) {
      if (this.order.status === OrderStatus.CLOSED) {
        Button($r('app.string.delete_order_text'), { type: ButtonType.Capsule })
          .orderButtonStyle(false)
          .onClick(() => {
            let rIndex = -1;
            ORDER_SAMPLES.forEach((order: Order, index: number) => {
              if (order.orderId === this.order.orderId) { rIndex = index; }
            });
            if (rIndex >= 0 && rIndex < ORDER_SAMPLES.length) {
              ORDER_SAMPLES.removeByIndex(rIndex);
              this.isAddOrDelete = true;        // membership changed: parent will re-sort
              this.change = !this.change;       // ping the parent's @Watch
            }
          })
      }
      if (this.order.status === OrderStatus.PAYING) {
        Button($r('app.string.immediate_payment_text'), { type: ButtonType.Capsule })
          .orderButtonStyle(true)
          .onClick(() => {
            this.order.status = OrderStatus.STAYING;   // @ObjectLink: repaints this card
            this.isAddOrDelete = false;                // only a status change: no re-sort
            this.change = !this.change;
          })
      }
      // ... refund, review, rebook, modify - one `if` per status
    }
    .justifyContent(FlexAlign.End)
  }
}
```

**Two flags, two meanings.** `change` says "something happened, re-filter";
`isAddOrDelete` says "the set of orders changed, so re-sort first". Splitting
them is what lets a plain status change skip the sort. Writing
`this.order.status` alone would repaint the card but leave it in the wrong tab
- the toggle is what moves it.

**One `@Extend` for the whole button vocabulary — same file** (as shipped)

```typescript
@Extend(Button)
function orderButtonStyle(main: boolean) {
  .height($r('app.float.order_oper_buttons_width'))
  .fontColor(main ? Color.White : $r('app.color.normal_other_button_font_color'))
  .backgroundColor(main ? $r('app.color.normal_blue_color') : $r('app.color.normal_other_button_background_color'))
  .fontSize($r('app.float.small_font_size'))
}
```

A single boolean parameter separating primary from secondary, applied at nine
call sites. This is the right size for `@Extend`: enough to remove repetition,
small enough to read.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

No routing configuration - the app is a single `@Entry` page with `Tabs`;
there is no `Navigation` and no `routerMap`.

Resource directories: `base`, `dark`, `en_US`, `zh_CN`. Tab titles, button
labels, the empty-state message and the payment prefixes are all `$r` string
resources; the order statuses and the sample hotel names are not
(`HW-09-0060`).

Layout constants live in two places, deliberately: numeric design tokens in
`resources/base/element/float.json` (`$r('app.float....')`) and behavioural
constants in `CommonConstants.ets` (animation duration 300 ms, underline width
56, stack padding 16).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` and
  `targetSdkVersion` are both `6.0.0(20)`.
- **State management V1** (`@Observed` / `@ObjectLink` / `@Link` / `@Watch`).
  Compare `TOUR-08`, which does the same job in V2 with `@ObservedV2` and
  `@Trace` and needs no change-flag plumbing.
- The tab bar assumes each label is `width('25%')`, so exactly four tabs fit
  the screen; more tabs scroll, but the `index - 1` scroll rule and the
  underline maths are tuned for this width.
- The data is a module-level `ArrayList` in `DataModel.ets`. There is no
  persistence: closing, paying, deleting and rebooking all survive only until
  the process ends.
- All five sample orders are created with `new Date()` for both check-in and
  check-out and the same `systemDateTime.getTime()` creation time, so the
  re-sort has nothing to distinguish them until a rebook adds a newer one.
- The "modify order", "more actions" and "additional evaluation" controls are
  rendered without handlers.

## Pitfalls

- **`HW-09-0060` — `OrderStatus`'s enum values are the Chinese labels,** and
  the card prints the enum directly. The status is the one string on screen
  that cannot be translated, and twelve comparisons match against display
  text.
- **`HW-09-0061` — `filterOrder` aliases `ORDER_SAMPLES` into the view's own
  `orderList` on the ALL branch** while the other branch calls `clear()` on
  that same field. Latent today because `orderTabType` never changes; fatal to
  the app's only data store the moment it does.
- **`HW-09-0062` — the stay dates are formatted with a hardcoded `'zh-CN'`
  `Intl.DateTimeFormat`,** ignoring the device locale, and a new formatter is
  built per card.
- **`HW-09-0063` — the `ForEach` key is `order.orderId + index`,** so a delete
  or a rebook re-keys every following row and rebuilds it instead of updating
  it. The ids are already unique.
- **`HW-09-0064` — `scrollToIndex(index - 1)` is a documented no-op at index
  0,** so selecting the first tab is the one case that does not scroll the bar
  back.
- **`HW-09-0065` — `avoidAreaChange` is never released.** The same boilerplate
  leak as `TOUR-01`, `TOUR-03` and the rest of this industry's samples.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` - `barHeight`, `onAnimationStart`, `onAnimationEnd`, `TabsAnimationEvent`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-references/02_application-framework/ts-container-scroll.md` - `Scroller.scrollToIndex` and its negative-index contract
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scroll
- `documentation/harmonyos-references/02_application-framework/ts-universal-component-area-change-event.md` - `onAreaChange` and `Area`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-component-area-change-event
- `documentation/harmonyos-guides/03_application-framework/arkts-observed-and-objectlink.md` - `@Observed` and `@ObjectLink`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-observed-and-objectlink
- `documentation/harmonyos-references/02_application-framework/ts-explicit-animation.md` - `animateTo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-explicit-animation
- `documentation/harmonyos-references/02_application-framework/js-apis-arraylist.md` - `ArrayList`, `removeByIndex`, `convertToArray`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-arraylist
- `TOUR-08` - the review flow that consumes an order from this list, in state management V2
- `TOUR-11` - the confirmation step that creates one
