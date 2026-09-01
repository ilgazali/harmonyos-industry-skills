---
id: COMMON-24
title: Card-to-fullscreen shared element transition - one continuous motion open, pull-down to return
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/24_pull_back.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/pull_back-0000002320205289
sample: huawei_industry_tree/19_common_technical_solutions/downloads/PullBack.zip
kits: ["@kit.ArkUI"]
apis: ["component .bindContentCover", ModalTransition, "component .geometryTransition", "UIContext.animateTo", TransitionEffect, "TransitionEffect.asymmetric", "TransitionEffect.opacity", Refresh, "Refresh.onRefreshing", "Refresh.refreshingContent", ComponentContent, wrapBuilder, Scroll, "component .opacity", "component .clip", ForEach, "PromptAction.showToast", "promptAction.ToastShowMode", LengthMetrics]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0077, HW-19-0078, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when tapping a card must **expand it into a full-screen detail
page as one continuous motion** - the element the user touched grows into the
detail header rather than being replaced by a new page - and pulling the detail
page down returns it to its place in the list. The document's example is an app
store: 用户使用应用市场时需要点击卡片放大全屏查看应用信息 ("in an app market the
user taps a card to enlarge it to full screen and view the application
information").

Three ArkUI features combine, and all three are required:

| Feature | Role |
| --- | --- |
| `bindContentCover` | hosts the full-screen page **without** its own transition (`ModalTransition.NONE`) |
| `geometryTransition` | ties the card and the detail content together by identifier, so position and size interpolate |
| `animateTo` | supplies the single animation that both sides follow |

## Feature checklist

The application must:

- Bind the full-screen detail page to the list page with `bindContentCover`,
  driven by one boolean.
- Set `modalTransition: ModalTransition.NONE` so the modal contributes no
  animation of its own.
- Give the card and the detail content the **same** `geometryTransition`
  identifier, with `{ follow: true }` on the card side.
- Toggle the boolean inside `animateTo`, so the geometry interpolation is driven
  by that animation's duration and curve.
- Fade the underlying list out as the detail opens and back in as it closes.
- Offer two ways back: a close button and a pull-down gesture.
- Reset the state in `onWillDisappear` so a dismissal from any source leaves the
  page consistent.

## Architecture

Single-module project (`entry` HAP), five source files:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup; publishes `topRectHeight` / `bottomRectHeight` |
| `model/DataModel.ets` | `CardData`, `allCardData`, `TabData`, `tabs` |
| `view/CardItem.ets` | one card; carries `geometryTransition(index, { follow: true })` and the tap callback |
| `view/RefreshBuilder.ets` | `refreshingContent` - a deliberately empty `@Builder`, so the pull-down shows no spinner |
| `pages/Market.ets` | the list page, the full-screen builder, and the two animation entry points |

**State.** Three variables carry the whole effect:

- `@State isDetailShow: boolean` - the `bindContentCover` trigger.
- `@State selectedIndex: number` - which card is expanded; it selects both the
  data and the `geometryTransition` identifier.
- `@State alphaValue: number` - opacity of the list page, 1 when the list is
  showing and 0 while the detail is open.

**The pairing.** `CardItem` applies
`.geometryTransition(this.index.toString(), { follow: true })` to the card body;
the full-screen builder applies `.geometryTransition(index.toString())` to the
`Scroll` inside the `Refresh`. The identifiers match, so ArkUI interpolates the
two components' geometry. `follow: true` on the source side is what makes the
card track the animation rather than jumping.

**Why `ModalTransition.NONE`.** `bindContentCover`'s built-in transitions would
animate the modal as a whole and fight the geometry interpolation. With `NONE`,
the only motion comes from `animateTo` plus the `TransitionEffect`s declared on
each side - `TransitionEffect.OPACITY` on the card,
`TransitionEffect.asymmetric(TransitionEffect.opacity(1), TransitionEffect.OPACITY)`
on the modal.

**The pull-down back gesture is a `Refresh`.** The detail content is wrapped in
`Refresh({ refreshing: false, refreshingContent: this.contentNode, offset: 4 })`
whose `onRefreshing` calls `onDetailBack()`. `refreshing` is pinned to `false`
and `refreshingContent` is an empty `ComponentContent`, so the component
contributes the pull gesture and nothing else - no spinner, no refresh state.
That reuse of `Refresh` as a pure gesture source is the non-obvious part of this
sample.

## Implementation steps

1. **Model the state**: one boolean for visibility, one index for the selection,
   one number for the list opacity.
2. **Build the empty refresh indicator** in `aboutToAppear`:
   `new ComponentContent(uiContext, wrapBuilder(refreshingContent))`.
3. **Bind the modal to the list page**:
   ```ts
   .bindContentCover(this.isDetailShow, this.cardContentBuilder(this.selectedIndex), {
     modalTransition: ModalTransition.NONE,
     onWillDisappear: () => { this.isDetailShow = false; this.alphaValue = 1; }
   })
   .opacity(this.alphaValue)
   ```
4. **Tag the card**: `.geometryTransition(this.index.toString(), { follow: true })`
   plus a `TransitionEffect.OPACITY` with the same duration and curve as the
   `animateTo`.
5. **Tag the detail content** with the identical identifier -
   `.geometryTransition(index.toString())` - on the component whose geometry
   should match the card.
6. **Animate both directions** with the same parameters
   (`duration: 350, curve: Curve.Friction`), toggling `isDetailShow` and
   `alphaValue` inside the callback.
7. **Add the pull-down return**: wrap the detail in `Refresh` with
   `refreshing: false` and an empty `refreshingContent`, and call the close
   animation from `onRefreshing`.
8. **Key the card list on stable data**, not on the array position
   (HW-19-0077).

## Verified snippets

All snippets below come from the sample project, not from the document.

### Binding the modal and fading the list

`PullBack.zip#PullBack/entry/src/main/ets/pages/Market.ets`

```ts
build() {
  Column() {
    // ... list and bottom tabs ...
  }
  .padding({ top: this.getUIContext().px2vp(this.topRectHeight) })
  .size({ width: '100%', height: '100%' })
  .backgroundColor('#0d000000')
  .bindContentCover(this.isDetailShow,
    this.cardContentBuilder(this.selectedIndex), {
      modalTransition: ModalTransition.NONE,
      onWillDisappear: () => {
        this.isDetailShow = false;
        this.alphaValue = 1;
      }
    })
  .opacity(this.alphaValue);
}
```

### The two animation entry points

`PullBack.zip#PullBack/entry/src/main/ets/pages/Market.ets`

```ts
/**
 * 点击卡片全屏显示
 */
private onCardClicked(index: number): void {
  this.selectedIndex = index;
  this.getUIContext()?.animateTo({
    duration: 350,
    curve: Curve.Friction
  }, () => {
    this.isDetailShow = !this.isDetailShow;
    this.alphaValue = 0;
  });
}

/**
 * 全屏返回卡片
 */
private onDetailBack(): void {
  this.getUIContext()?.animateTo({
    duration: 350,
    curve: Curve.Friction
  }, () => {
    this.isDetailShow = !this.isDetailShow;
    this.alphaValue = 1;
  });
}
```

Note that the sample animates `alphaValue` too - the document's snippet shows
only the `isDetailShow` toggle.

### The card side of the pairing

`PullBack.zip#PullBack/entry/src/main/ets/view/CardItem.ets`

```ts
@Component
export default struct CardItem {
  @Prop data: CardData;
  @Prop index: number;
  private onCardClicked: (index: number) => void = () => {};

  build() {
    Column() {
      Row() {
        Column() {
          // ... card content ...
        }
        .width('100%')
        .height('100%')
        .alignItems(HorizontalAlign.Start)
        .borderRadius(20)
        .clip(true)
        .onClick(() => {
          this.onCardClicked(this.index);
        })
        .geometryTransition(this.index.toString(), { follow: true })
        .transition(TransitionEffect.OPACITY.animation({ duration: 350, curve: Curve.Friction }));
      }
      .justifyContent(FlexAlign.Start);
    }
    .backgroundColor(Color.White)
    .size({ width: '92%', height: this.index === 0 ? 446 : 156 })
    .borderRadius(20);
  }
}
```

### The detail side, with the pull-down gesture

`PullBack.zip#PullBack/entry/src/main/ets/pages/Market.ets`

```ts
@Builder
cardContentBuilder(index: number) {
  Column({ space: 20 }) {
    Refresh({ refreshing: false, refreshingContent: this.contentNode, offset: 4 }) {
      Scroll() {
        Column() {
          Stack({ alignContent: Alignment.Start }) {
            Image(allCardData[index].image)
              .clip(true)
              .transition(TransitionEffect.opacity(1))
              .width('100%')
              .height('47%');
            // close button and topic overlay ...
          };
          // avatar, title, get button, body text ...
        };
      }
      .scrollable(ScrollDirection.Vertical)
      .scrollBar(BarState.Off)
      .edgeEffect(EdgeEffect.None)
      .width('100%')
      .height('100%')
      .geometryTransition(index.toString());
    }
    .onRefreshing(() => {
      this.onDetailBack();
    });
  }
  .padding({ bottom: this.getUIContext().px2vp(this.bottomRectHeight) })
  .width('100%')
  .height('100%')
  .backgroundColor(Color.White)
  .transition(TransitionEffect.asymmetric(
    TransitionEffect.opacity(1),
    TransitionEffect.OPACITY
  ));
}
```

### The empty refresh indicator

`PullBack.zip#PullBack/entry/src/main/ets/view/RefreshBuilder.ets`

```ts
/**
 * 下拉显示内容为空
 */
@Builder
export function refreshingContent() {
  Stack() {
  }
  .width('100%');
}
```

`PullBack.zip#PullBack/entry/src/main/ets/pages/Market.ets`

```ts
private contentNode?: ComponentContent<Object> = undefined;

aboutToAppear(): void {
  // 动态更新
  let uiContext = this.getUIContext();
  this.contentNode = new ComponentContent(uiContext, wrapBuilder(refreshingContent));
  // ... date formatting ...
}
```

### The card list (as shipped - see HW-19-0077)

`PullBack.zip#PullBack/entry/src/main/ets/pages/Market.ets`

```ts
ForEach(allCardData, (cardData: CardData, index: number) => {
  Column() {
    CardContent({
      data: cardData, index: index, onCardClicked: (index: number) => {
        this.onCardClicked(index);
      }
    });
  }
  .margin({ top: 14 })
  .width('100%');
}, (index: number) => index.toString());   // FIX: (cardData: CardData, index: number) => cardData.title
```

## Permissions & config

**No permissions are required** and none are declared - the feature is entirely
animation and layout.

`PullBack.zip#PullBack/entry/src/main/module.json5` declares
`"deviceTypes": ["phone", "tablet", "2in1"]`, the usual `EntryAbility` with the
home skill, and an `EntryBackupAbility` backup extension. There is no
`requestPermissions` block and no `routerMap` - the detail page is a modal, not
a route.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **The two `geometryTransition` identifiers must match exactly**, and only one
  pair may be active at a time - which is why `selectedIndex` selects both the
  data and the identifier.
- **`ModalTransition.NONE` is mandatory for this effect.** Any other modal
  transition animates the cover independently and the shared-element
  interpolation is lost.
- **The state toggle must happen inside `animateTo`.** `geometryTransition`
  interpolates against the ongoing animation; flipping `isDetailShow` outside one
  produces a jump cut.
- **`follow: true` belongs on the source (card) side.** The document's snippet
  shows it only there, and so does the sample.
- **`Refresh` here is a gesture source, not a refresher.** `refreshing` is a
  literal `false` and `refreshingContent` is empty; do not wire real refresh
  state into it without rethinking the back gesture.
- **Card heights are hardcoded per position** - `this.index === 0 ? 446 : 156` in
  `CardItem`, and the first card also switches which sub-block is visible. The
  layout is data-position-dependent, which is a second reason the list should not
  be reordered (see HW-19-0077).
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **The `ForEach` key generator takes the item, not the index, which the sample
  gets wrong.** `(index: number) => index.toString()` is called with a
  `CardData`, so it returns `'[object Object]'`; the framework then appends the
  real index because the key generator does not declare it - the exact fallback
  the official guide warns produces duplicate-key problems, and the
  index-as-key anti-pattern it separately warns against. Key on stable data.
  (HW-19-0077)
- **The date label mixes hardcoded Chinese with resource strings.** The seven
  weekdays are `$r('app.string.*')` but the month/day separators are literals in
  `` `${month}月${day}日` ``. (HW-19-0078)
- **`onWillDisappear` is not optional.** `bindContentCover` can be dismissed by
  the system, and without resetting `isDetailShow` and `alphaValue` there the
  list would stay invisible.
- **The document's `animateTo` snippet omits `alphaValue`.** Copying it verbatim
  gives the geometry motion without the crossfade, and the list page stays fully
  opaque underneath the modal.
- **`allCardData[index]` is read directly inside the modal builder.** The builder
  is invoked with `this.selectedIndex`; if the array can change while the modal
  is open, the index is stale.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` -
  the `keyGenerator` signature `(item: Object, index: number)`, the automatic
  index-appending fallback, the duplicate-key warning and "Do not use the data
  item index as the key".
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-shared-element-transition -
  the shared element (一镜到底) transition guide the document is based on.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-attributes-modal-transition#bindcontentcover -
  `bindContentCover` and `ModalTransition`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-transition-animation-geometrytransition -
  `geometryTransition` and its `follow` option.
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` -
  `UIContext.animateTo`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-uicontext-uicontext#animateto
- `documentation/harmonyos-guides/03_application-framework/arkts-user-defined-arktsnode-buildernode.md` -
  `ComponentContent` and `wrapBuilder`, used for the empty refresh indicator.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-user-defined-arktsnode-buildernode
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/pull_back-0000002320205289
