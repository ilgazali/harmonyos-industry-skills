---
id: SHOP-08
title: Long-press to mark a product - an exclusive tap/long-press pair driving an in-card overlay menu
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/08_product_card_demo.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/product_card_demo-0000002324599805
sample: huawei_industry_tree/16_shopping/downloads/商品长按标记示例代码.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit", "@kit.CoreFileKit"]
apis: [parallelGesture, GestureGroup, GestureMode, TapGesture, LongPressGesture, Grid, GridItem, columnsTemplate, Stack, "@Link", "@State", "@StorageProp", "AppStorage.setOrCreate", "AppStorage.get", NavPathStack, pushPathByName, NavDestination, "UIContext.getPromptAction", showToast, draggable]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-16-0009, HW-16-0027]
status: verified-with-fixes
---

## When to use

Load this card when a **list item needs a secondary action set that must not
cost a tap** — the feed-tuning menu behind a long press. In shopping feeds it is
不感兴趣 / 不想看 / 买过了 / 看相似 (not interested / don't show / already
bought / show similar); the same construct covers "hide this post", "mute this
sender", "remove from history".

The pattern has two halves that people usually get wrong separately. The
**gesture** half: a tap and a long press on the same target, combined
exclusively so exactly one of them wins, attached with `parallelGesture` so the
ancestor's tap-to-dismiss still works. The **overlay** half: the menu is not a
popup or a dialog — it is a conditionally rendered `Stack` layer inside the card
itself, so it is positioned by the layout rather than by coordinates, and it
scrolls with the grid.

Take the overlay approach when the menu belongs to one card and must line up
with it. Take `bindMenu`/`bindContextMenu` instead when the menu should escape
the card's bounds or survive a scroll. This sample's version costs no
positioning maths at the price of being clipped by the card.

## Feature checklist

- A two-column product grid; each cell is a card with image, title, price,
  subsidy tag, sold count, repeat-customer count and store name.
- Long-pressing a card's image dims that card and draws a menu over it.
- The menu offers five marking actions plus a 看相似 (see similar) row.
- Tapping the image again, tapping the dimmed backdrop, or tapping the grid's
  empty area closes the menu.
- Choosing a marking action removes the product from the list and raises a
  toast (已隐藏 / 已标记).
- Choosing 看相似 pushes a similar-products page showing the pressed product at
  the top and a grid of alternatives underneath.
- The similar-products page supports the same long-press marking on its own
  list.
- The back arrow on the similar page returns to the feed.

## Architecture

One `entry` module. Note the layout: everything visual lives under `model/`,
including components — the two `pages/` files are only the routed screens.

```
entry/src/main/ets
├── entryability/EntryAbility.ets           window setup, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── model
│   ├── BottomNavigation.ets                the bottom bar (decorative)
│   ├── Product.ets                         the Product class + PRODUCT_LIST + similarItems
│   ├── ProductCard.ets                     THE FEATURE: card + gestures + overlay + handleAction
│   ├── SearchBar.ets                       search bar component
│   └── TopMenuBar.ets                      top category bar
└── pages
    ├── ItemListPage.ets                    @Entry: Navigation > Grid of ProductCard
    └── SimilarItemsPage.ets                the push target (NavDestination, route name 'PageOne')
entry/src/main/resources/base/profile/route_map.json   'PageOne' -> PageOneBuilder
```

The documented tree matches the zip.

**The design decision worth copying** is that the overlay is *inside*
`ProductCard`'s own `Stack`, gated by a shared selection id:

```typescript
Stack() {
  Column() { /* the card */ }
  if (this.selectedProductId === this.product.id) {
    Stack() { /* backdrop + close icon + menu */ }
  }
}
```

One `@Link selectedProductId: number` is shared by every card in the grid. Each
card independently asks "am I the selected one?", so opening one menu closes
every other by construction — no bookkeeping, no list of open states, no
"close the previous one" call. `0` is the sentinel for "none selected", which
works because product ids start at 1.

The trade-off worth naming: because the overlay is a sibling inside the card's
`Stack`, it inherits the card's bounds. The menu cannot be taller than the card
(253 vp here) and cannot overhang its neighbours. That is why the five action
buttons are 25 vp tall with 8 vp gaps — the design is squeezed to fit the
container it chose. A `bindContextMenu` would not have that constraint.

**Worth avoiding:** the `NavPathStack` is handed from the entry page to the
child through `AppStorage`:

```typescript
// ItemListPage
aboutToAppear(): void { AppStorage.setOrCreate('PathStack', this.pathStack); }
// ProductCard and SimilarItemsPage
pathStack: NavPathStack = AppStorage.get('PathStack') as NavPathStack;
```

It works, but it makes a navigation stack a process-global under an untyped
string key, and the `as` cast will not save you if the key is missing — the
field is then `undefined` and the first `pushPathByName` throws. Prefer
`NavDestinationContext.pathStack` in the destination (see `SHOP-06`) and a
`@Consume` or an explicit parameter on the way down.

## Implementation steps

1. **Model the selection as one shared id, not a per-card boolean.** Declare
   `@Link selectedProductId: number` in the card and `@State selectedProductId:
   number = 0` in the page.
2. **Attach the gesture pair to the image, not the card.** Only the image is a
   sensible long-press target; the title and price rows stay tappable for a
   normal product open.
3. **Turn off dragging first**: `.draggable(false)`. `Image` is draggable by
   default, and a long press is exactly the gesture the drag recogniser also
   claims — leave it on and the card starts a drag preview instead of opening
   the menu.
4. **Combine with `GestureGroup(GestureMode.Exclusive, tap, longPress)`.** In
   exclusive mode all members are evaluated together and the first success
   cancels the rest, so a press either opens the menu or closes it, never both.
5. **Order matters inside an exclusive group.** The guide is explicit: an
   earlier gesture that consumes the touch prevents a later one from ever being
   recognised. Tap-then-longpress is safe because a tap requires a *release*;
   putting a double-tap before a tap is the case that breaks.
6. **Bind with `parallelGesture`, not `gesture`.** `parallelGesture` lets the
   component and its ancestors recognise the same touch, which is what keeps
   the grid container's tap-to-dismiss handler alive.
7. **Render the overlay conditionally inside the card's `Stack`** and put the
   dismiss handler on the overlay's own root, so a tap anywhere on the dimmed
   area closes it.
8. **Filter the list, do not mutate it.** `this.itemList = this.itemList.filter(...)`
   assigns a new array to an `@Link`, which is what propagates the change back
   to the page's `@State`. An in-place `splice` on the same array reference
   would not re-render reliably.
9. **Close the overlay after the action**, in the button's own `onClick`, after
   `handleAction` has run — the handler reads `selectedProductId` and clearing
   it first would filter nothing.
10. **Do not follow the document's `Stack` link** (`HW-16-0009`): it points at
    `@ohos.util.Stack`, the linear-container data structure, not the ArkUI
    `Stack` layout component the sample uses.

## Verified snippets

All snippets are from `商品长按标记示例代码.zip`. The corrections marked below
are hardening, not filed defects.

**The gesture pair — `entry/src/main/ets/model/ProductCard.ets`** (as shipped)

```typescript
Image(this.product.imageUrl)
  .width('100%')
  .aspectRatio(1)
  .objectFit(ImageFit.Cover)                  // 关键属性，保持比例填满容器
  .borderRadius({
    topLeft: this.cardRadius,
    topRight: this.cardRadius,
    bottomLeft: 0,
    bottomRight: 0
  })
  .draggable(false)                           // 关闭拖拽
  .parallelGesture(GestureGroup(GestureMode.Exclusive,
    TapGesture({ count: 1, fingers: 1 })
      .onAction(() => {
        this.selectedProductId = 0;           // tap: close whatever is open
      }),
    LongPressGesture({ repeat: true })
      .onAction(() => {
        this.selectedProductId = this.product.id;   // long press: this card owns the menu
      })
  ));
```

**Three attributes carry the design.** `draggable(false)` comes first for a
reason: an `Image` participates in drag-and-drop by default, and the drag
recogniser competes for the same long press. Leave it enabled and the user gets
a floating image preview instead of a menu — the single most common failure when
this pattern is copied onto an image.

`GestureMode.Exclusive` is the guarantee that exactly one branch runs. The
reference defines it as "all registered gestures are processed simultaneously;
once any gesture is recognized successfully, the recognition process ends, and
all other gestures are deemed unrecognized". So a quick press closes the menu
and a held press opens it, with no window where both fire.

`parallelGesture` rather than `gesture` is the subtle one. Its reference says
the gesture "can be recognized at once by the component and its child
component… both it and its child component can respond to the same gesture
events, thereby implementing a quasi-bubbling effect". Here that means the
image's tap does not swallow the touch from the ancestors, so the grid's
`onClick(() => this.selectedProductId = 0)` still runs — the tap-outside
dismissal keeps working even when the tap lands on a card.

One thing to change when copying: `LongPressGesture({ repeat: true })` re-fires
`onAction` every 500 ms for as long as the finger is held. The handler is
idempotent (it assigns the same id), so nothing breaks, but the repeat buys
nothing here and is the wrong default for a handler with side effects. `repeat`
is for progress-style long presses that accumulate; a menu-opening press should
use the default `false`.

**The overlay — same file** (as shipped)

```typescript
build() {
  Stack() {
    Column() { /* image + title + price + store row: the card itself */ }
      .width('100%')
      .height(this.cardHeight)                       // 253
      .backgroundColor(Color.White)
      .borderRadius(this.cardRadius);

    // 长按覆层，只有选中时显示
    if (this.selectedProductId === this.product.id) {
      Stack() {
        Column()
          .width('100%')
          .height(this.cardHeight)
          .backgroundColor('#80000000')              // 半透明背景
          .borderRadius(this.cardRadius);

        Image($r('app.media.ic_close'))
          .width(24).height(24)
          .position({ x: '82%', y: '1%' });

        Column() {
          this.buildActionButton($r('app.string.card_text_dislike1'), 'hide', $r('app.media.ic_dislike'));
          this.buildActionButton($r('app.string.card_text_dislike2'), 'dislike', $r('app.media.ic_cry'));
          this.buildActionButton($r('app.string.card_text_dislike3'), 'hide', $r('app.media.ic_shopping_car'));
          this.buildActionButton($r('app.string.card_text_dislike4'), 'hide', $r('app.media.ic_shopping'));
          this.buildActionButton($r('app.string.card_text_dislike5'), 'hide', $r('app.media.ic_ignore'));

          Row() {
            Text($r('app.string.card_text_similar'))
              .onClick(() => { this.handleAction('similar'); });
            Image($r('app.media.ic_right')).width(8).height(8);
          }
          .margin({ top: 5 });
        }
        .borderRadius(this.cardRadius)
        .margin({ top: 20 });
      }
      .onClick(() => {
        this.selectedProductId = 0;                  // 点击覆盖层外部关闭
      });
    }
  };
}
```

**The `if` inside the `Stack` is the whole overlay mechanism.** ArkUI's
conditional rendering removes the subtree entirely when the test fails, so a
non-selected card carries no hidden layer, no `visibility` juggling and no
opacity animation to reset. Because the overlay is the *second* child of the
`Stack` it paints above the card without any `zIndex`, and because `Stack`
defaults to centre alignment the menu lands centred on the card for free — the
only positioning in the whole block is the close icon's `position({ x: '82%',
y: '1%' })`.

`backgroundColor('#80000000')` is a 50%-alpha black scrim that both dims the
product and gives the white action buttons contrast. Note the dismissal handler
is on the **outer overlay `Stack`**, so it covers the scrim, the close icon and
the gaps between buttons; the buttons themselves have their own `onClick`,
which runs first and stops there.

The five action buttons are built by one `@Builder` taking
`(text, action, imgUrl)`, and four of the five pass the *same* `'hide'` action.
That is the tell that this is a UI sample: five distinct labels, two distinct
behaviours.

**The action handler — same file** (corrected — filter on the card's own id)

```typescript
@Builder
buildActionButton(text: string | Resource, action: string, imgUrl: Resource) {
  Button() {
    Row() {
      Image(imgUrl).width(16).height(16).margin({ right: 8 });
      Text(text).fontSize(12).fontWeight(500).fontColor('#ff000000').height(20);
    }
    .justifyContent(FlexAlign.Start)
    .width('100%')
    .padding({ left: 12, right: 12 });
  }
  .height(25)
  .margin({ left: 18, right: 18, bottom: 8 })
  .backgroundColor('#ffffffff')
  .onClick(() => {
    this.handleAction(action);
    this.selectedProductId = 0;              // 关闭覆盖层 — after the filter, never before
  });
}

// 处理菜单选项点击
private handleAction(action: string) {
  switch (action) {
    case 'hide':
    case 'dislike':
      // 从列表中移除该商品
      this.itemList = this.itemList.filter(item => item.id !== this.product.id);  // FIX: shipped code
      this.getUIContext().getPromptAction().showToast({                           // reads selectedProductId
        message: action === 'hide' ? $r('app.string.already_hides') : $r('app.string.already_marking')
      });
      break;

    case 'similar':
      // 跳转到相似商品页面
      this.pathStack.pushPathByName('PageOne', null);
      break;
  }
}
```

**Reassignment, not mutation, is what updates the grid.** `itemList` is an
`@Link Array<Product>`, so writing a *new* array to it propagates to the page's
`@State itemList` and re-runs the `ForEach`. `filter` returns a new array, which
is exactly right; `splice` on the existing reference would mutate the same
object the parent holds and may not trigger the observation.

The correction is a coupling fix. The shipped handler filters on
`this.selectedProductId`, a value that is only equal to `this.product.id`
because the overlay is rendered under that exact condition — so the handler
silently depends on the caller's ordering, and clearing the selection one line
earlier would delete nothing. Filtering on `this.product.id` states the intent:
this card removes this product.

Note also that `handleAction` runs *before* `selectedProductId = 0` in the
button's `onClick`. With the correction that ordering stops mattering, which is
the point of making it.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

```json5
"pages": "$profile:main_pages",
"routerMap": "$profile:route_map"
```

```json5
// route_map.json
{ "name": "PageOne",
  "pageSourceFile": "src/main/ets/pages/SimilarItemsPage.ets",
  "buildFunction": "PageOneBuilder" }
```

`main_pages` lists `pages/ItemListPage` only. `Navigation` is configured
`.mode(NavigationMode.Stack)` with the title bar, tool bar and back button all
hidden, and the top/bottom avoid-area insets applied as padding from
`@StorageProp('topRectHeight')` / `('bottomRectHeight')`.

Every user-visible string is a `$r('app.string...')` resource except three:
`'相似商品'` (the similar page's title) and the two template strings
`` `回头客${...}万` `` and `` `${...}+人付款` ``, which are hardcoded Chinese in
`ProductCard.ets` and `SimilarItemsPage.ets`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- `parallelGesture` cannot be called inside an `attributeModifier`, and none of
  `gesture` / `priorityGesture` / `parallelGesture` support switching the bound
  gesture with a ternary expression.
- **The overlay is clipped to the card.** The menu is a child of the card's
  `Stack`, so it cannot exceed 253 vp of height and cannot overhang into the
  neighbouring column. Use `bindContextMenu` if the menu must escape.
- **`@State product: Product` in `ProductCard` should be `@Prop` (or
  `@ObjectLink`).** `@State` initialised from a parent takes the value once at
  construction and then ignores the parent; the grid happens to rebuild cards
  on every list change, so the bug does not surface here — but it will the
  moment a product is edited in place.
- **`similarItems` gives all six entries `id: 6`.** Long-pressing any of them
  matches `selectedProductId === product.id` on all six, so six overlays open at
  once, and marking one filters all six out of the list. The list is a
  copy-paste placeholder, but it demonstrates precisely the failure mode of
  keying selection on a non-unique id.
- Neither `ForEach` supplies a key generator, so ArkUI falls back to its default
  key. With duplicate ids in `similarItems` that is another reason the similar
  page misbehaves.
- The similar page's back arrow calls `pathStack.clear()`, not `pop()`. With a
  one-level stack the effect is the same; with a deeper stack it would unwind
  everything.
- `SimilarItemsPage` declares `@State similarItems: Array<Product> = similarItems;`
  — the field initialiser resolves to the *imported* constant of the same name.
  It compiles and works, but the shadowing is a trap for the next reader.
- The top menu bar and search bar on `ItemListPage` are flat images
  (`app.media.SearchBar`, `app.media.TopMenuBar`) positioned by percentage, not
  the `SearchBar.ets` / `TopMenuBar.ets` components that also ship in `model/`.
  The bottom navigation is likewise decorative.
- The grid is fixed at `'1fr 1fr'` with a `height('80%')` and percentage-positioned
  overlays, so the page does not adapt to a tablet or a resized 2in1 window.

## Pitfalls

- **`HW-16-0009`** (E/low, confirmed): the document links `Stack` to
  `js-apis-stack` (`@ohos.util.Stack`, the linear-container data structure) in
  both the 实现思路 step 1 sentence and the 参考文档 list, but the overlay it
  describes is built with the ArkUI `Stack` **layout container**. The sample
  never imports `@ohos.util.Stack`. A reader following the link lands on an
  unrelated collections API. Fix: point both links at the `ts-container-stack`
  reference.

The systematic snippet defect `HW-16-0013` does **not** apply to this document:
both of its excerpts elide with `// ...` inside balanced blocks and parse as
written.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-stack.md` — the ArkUI `Stack` layout container (the page `HW-16-0009` says the doc should link)
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-stack
- `documentation/harmonyos-references/02_application-framework/ts-combined-gestures.md` — `GestureGroup` and the `GestureMode` table, including `Exclusive`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-combined-gestures
- `documentation/harmonyos-guides/03_application-framework/arkts-gesture-events-combined-gestures.md` — exclusive recognition and why binding order matters
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-gesture-events-combined-gestures
- `documentation/harmonyos-references/02_application-framework/ts-gesture-settings.md` — `gesture` vs `priorityGesture` vs `parallelGesture`, and the quasi-bubbling behaviour
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-gesture-settings
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-longpressgesture.md` — `fingers`, `repeat` and `duration`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-longpressgesture
- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` — `Grid`, `GridItem`, `columnsTemplate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `documentation/harmonyos-guides/03_application-framework/arkts-layout-development-create-grid.md` — building the two-column feed
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-layout-development-create-grid
- `SHOP-06` — the `routerMap` + `NavDestinationContext.pathStack` form this sample's `AppStorage` handoff should have used
- `huawei_industry_tree/16_shopping/docs/08_product_card_demo.md` — the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/product_card_demo-0000002324599805
