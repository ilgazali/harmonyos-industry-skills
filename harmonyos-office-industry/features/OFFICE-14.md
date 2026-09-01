---
id: OFFICE-14
title: Marquee banner notice - scrolling Text with a mirrored gradient fade at both ends
industry: 05_office
doc: huawei_industry_tree/05_office/docs/14_text_marquee.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/text_marquee-0000002331949121
sample: huawei_industry_tree/05_office/downloads/跑马灯横幅通知示例代码.zip
kits: ["@kit.ArkUI", "@kit.PerformanceAnalysisKit"]
apis: [Text, "Text.textOverflow", "TextOverflow.MARQUEE", linearGradient, GradientDirection, Stack, Flex, FlexAlign, List, ListItem, ForEach, "ListItem.swipeAction", Tabs, TabContent, "@Provide", "@Consume", "hilog.info"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-05-0081, HW-05-0082, HW-05-0083, HW-05-0084, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when an office app needs a **pinned banner at the top of a page
that scrolls a message too long to fit** - an alert strip, a system notice, a
holiday announcement above a chat list.

The whole effect is two ArkUI features and no animation code:

- `Text(...).textOverflow({ overflow: TextOverflow.MARQUEE })` makes the text
  scroll horizontally when it overflows its box;
- two small `linearGradient` columns, mirrored left and right and stacked **over**
  the text, fade it out at both edges so it appears to slide in and out of the
  banner rather than being clipped.

No permission, no timer, no `Marquee` component - the fade is pure layout.

## Feature checklist

The application must be able to:

- Render a fixed-height banner strip above the page content.
- Scroll a message that is wider than the strip, with no user interaction.
- Fade the message out at **both** the left and right edges, matching the strip's
  background.
- Keep the fade overlays non-interactive and sized independently of the message
  length.
- Show the surrounding page - here a chat list with swipe-to-delete - underneath
  the banner.

## Architecture

Single `entry` HAP:

| File | Responsibility |
| --- | --- |
| `pages/HomePage.ets` | `@Entry`; the tab shell and `@Provide sessionList` |
| `components/Message.ets` | `MyChat` - the header, **the marquee banner**, and the chat list with `swipeAction` |
| `model/MessageData.ets` | `MessageItem` (with an `id`) and the exported `MESSAGE_DATA` seed |
| `model/TabContent.ets` | tab descriptors |
| `constants/Constants.ets` | names, messages, font weights, tab indices |
| `entryability/`, `entrybackupability/` | ability entry and backup stub |

The banner is a three-layer construction:

```
Flex(column, Center)                     outer strip: height + background colour
  Stack()                                 layers the fades over the text
    Text(notification)                    layer 1 - MARQUEE overflow
    Flex(SpaceBetween)                    layer 2 - the two fades
      Column().linearGradient(Right)        left edge:  opaque -> transparent
      Column().linearGradient(Left)         right edge: transparent -> opaque
```

Both gradient columns use the **same** colour pair
(`linear_gradient_left` -> `linear_gradient_right`) and differ only in
`direction`, which is what makes them mirror images. Their width equals their
height (`app.float.notification_height`), so each fade is a square anchored to
its end of the strip while the `SpaceBetween` flex keeps the middle clear.

## Implementation steps

1. **Declare no permission.** Nothing here touches a system service; the sample's
   `module.json5` has no `requestPermissions` block and the document has no
   权限说明 section - consistent.
2. **Give the banner a fixed height and background.** An outer `Flex` with
   `direction: FlexDirection.Column`, `justifyContent: FlexAlign.Center`, the
   strip height and the banner background colour.
3. **Put the text in a `Stack`.** The `Stack` is what lets the gradients sit on
   top of the text rather than beside it.
4. **Turn on the marquee.** `.textOverflow({ overflow: TextOverflow.MARQUEE })` on
   the `Text`. Give the text `height('100%')` so it centres in the strip.
5. **Add the two fades.** Inside the same `Stack`, a
   `Flex({ justifyContent: FlexAlign.SpaceBetween })` holding two empty `Column`s.
   The left one uses `GradientDirection.Right`, the right one
   `GradientDirection.Left`, both with the same `colors` array. A single column,
   as the document's snippet shows, fades one side only (HW-05-0081).
6. **Size the fades from a resource**, not from a constant the project does not
   have (HW-05-0081).
7. **Back the list with stable keys.** Where the banner sits above a list that
   supports deletion, give `ForEach` a `keyGenerator` over the model's `id`
   (HW-05-0082) and copy the seed array into state rather than aliasing the
   exported constant (HW-05-0083).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### The marquee banner

`跑马灯横幅通知示例代码.zip#entry/src/main/ets/components/Message.ets`

```ts
//通知栏
Flex({ direction: FlexDirection.Column, justifyContent: FlexAlign.Center }) {
  Stack() {
    // 实现方式二
    Text($r('app.string.notification'))
      .height($r('app.string.full_height'))
      .fontColor($r('app.color.notification_text_color'))
      .fontSize($r('app.float.fontsize_fifteen'))
      .fontWeight(Constants.NOTIFICATION_TEXT_FONTWEIGHT)
      .textOverflow({ overflow: TextOverflow.MARQUEE });

    Flex({ justifyContent: FlexAlign.SpaceBetween }) {
      Column() {
      }
      .width($r('app.float.notification_height'))
      .height($r('app.float.notification_height'))
      .linearGradient({
        direction: GradientDirection.Right, // 渐变方向
        colors: [[$r('app.color.linear_gradient_left'), 0], [$r('app.color.linear_gradient_right'), 1]]
      });

      Column() {
      }
      .width($r('app.float.notification_height'))
      .height($r('app.float.notification_height'))
      .linearGradient({
        direction: GradientDirection.Left, // 渐变方向
        colors: [[$r('app.color.linear_gradient_left'), 0], [$r('app.color.linear_gradient_right'), 1]]
      });
    };
  }
  .backgroundColor(Color.Transparent)
  .height($r('app.string.full_height'));
}
.margin({ top: $r('app.float.notification_margin_top'), bottom: $r('app.float.notification_margin_bot') })
.width($r('app.string.full_width'))
.height($r('app.float.notification_height'))
.backgroundColor($r('app.color.notification_background_color'));
```

This block is the entire feature. Two things make it work and are easy to get
wrong: the `Stack` (so the fades overlay rather than displace the text) and the
**pair** of mirrored gradients (so both ends fade).

### The list underneath - as shipped

`跑马灯横幅通知示例代码.zip#entry/src/main/ets/components/Message.ets`

```ts
@Component
export struct MyChat {
  @Consume sessionList: MessageItem[];

  aboutToAppear(): void {
    // 根据会话列表对消息分类
    this.sessionList = MESSAGE_DATA;          // aliases the module constant - HW-05-0083
  }

  // ...
  List() {
    ForEach(this.sessionList, (item: MessageItem, index: number) => {
      ListItem() {
        Column() {
          this.SessionItem(item, index);
        };
      }
      .swipeAction({
        end: {
          // index为该ListItem在List中的索引值。
          builder: () => {
            this.itemEnd(item, index);
          },
        }
      }); // 设置侧滑属性.
    });                                       // no keyGenerator - HW-05-0082
  }

  // 尾端滑出组件
  @Builder
  itemEnd(item: MessageItem, index: number) {
    Row() {
      Column() {
        Image($r('app.media.delete'));
      }
      .onClick(() => {
        hilog.info(0X0000, 'testTag', '', item.sendingTime);   // empty format - HW-05-0084
        this.sessionList.splice(index, 1);
      });
    };
  }
}
```

Corrected list plumbing:

```ts
aboutToAppear(): void {
  this.sessionList = [...MESSAGE_DATA];
}

ForEach(this.sessionList, (item: MessageItem, index: number) => {
  ListItem() { /* ... */ }
    .swipeAction({ end: { builder: () => { this.itemEnd(item, index); } } });
}, (item: MessageItem) => item.id.toString());
```

### The model that already carries a key

`跑马灯横幅通知示例代码.zip#entry/src/main/ets/model/MessageData.ets`

```ts
export class MessageItem {
  /** ID. */
  id: number = 0;
  /** 接收人ID */
  receiver: string = '';
  /** 接收人头像 */
  avatar: Resource = $r('app.media.startIcon');
  /** 消息内容 */
  content: string = '';
  /** 发送时间文本 */
  sendingTime: string = '';
}

/** 聊天列表数据 */
export const MESSAGE_DATA: MessageItem[] = [ /* ... */ ];
```

`id` is exactly the stable key the `ForEach` should use.

## Permissions & config

`跑马灯横幅通知示例代码.zip#entry/src/main/module.json5` declares **no
`requestPermissions` block**, and none is needed - the feature is layout only.
The document likewise has no 权限说明 section, which is consistent.

Resources the banner depends on, all referenced through `$r`:

| Resource | Role |
| --- | --- |
| `app.string.notification` | the scrolling message |
| `app.float.notification_height` | strip height, and the width/height of each fade square |
| `app.color.notification_background_color` | strip background |
| `app.color.notification_text_color` | message colour |
| `app.color.linear_gradient_left` / `app.color.linear_gradient_right` | the two gradient stops, shared by both fades |
| `app.float.notification_margin_top` / `_bot` | spacing above and below the strip |

The gradient stop colours must match the strip background, otherwise the fade
reads as a coloured block rather than a dissolve.

`build-profile.json5` pins the SDK to `6.0.0(20)`.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **The marquee only scrolls on overflow.** `TextOverflow.MARQUEE` animates when
  the text is wider than its container; a short message simply sits still, so the
  banner width and the message length are coupled.
- **The fades are fixed squares.** Each gradient column is
  `notification_height` wide and tall, so the faded region does not scale with the
  message or with the screen; on a very narrow screen the two squares can meet in
  the middle.
- **The gradient colours are tied to the background.** Both columns run
  `linear_gradient_left` -> `linear_gradient_right`; if the strip background
  changes, both stops must change with it.
- **`GradientDirection` is mirrored, not repeated.** The left overlay uses
  `Right` and the right overlay uses `Left`; using the same direction for both
  produces a fade on one side and a solid block on the other.
- **The banner is decorative.** It is not a system notification - nothing here
  uses Notification Kit; the text comes from a string resource and never changes
  at runtime in this sample.

## Pitfalls

- **The document's gradient snippet is incorrect twice over.** It names a
  `CommonConstants` class with a `NOTIFICATION_HEIGHT` member that the project
  does not contain - the sample's class is `Constants` and it sizes the overlay
  from `$r('app.float.notification_height')` - and it shows a single
  `GradientDirection.Left` column, which cannot produce the 两端渐变
  ("gradient at both ends") effect the 场景介绍 promises. Use the shipped pair of
  mirrored columns inside a `SpaceBetween` Flex. (HW-05-0081)
- **`ForEach` without a `keyGenerator` over a list that deletes by index is
  incorrect.** The default key embeds the index, so removing one row changes the
  key of every row after it and forces them all to be rebuilt; the guide warns
  against index-based keys explicitly. The model already has an `id` - key on it.
  (HW-05-0082)
- **`this.sessionList = MESSAGE_DATA` is incorrect.** It aliases the exported
  module constant, and the later `splice` mutates it, so the seed data shrinks
  permanently and cannot be restored by re-entering the page. Spread it:
  `[...MESSAGE_DATA]`. (HW-05-0083)
- **`hilog.info(0X0000, 'testTag', '', value)` is incorrect.** An empty format
  string has no placeholder for the variadic argument, so the value is dropped and
  an empty line is logged. Supply a format such as `'sent at %{public}s'`.
  (HW-05-0084)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-text.md` -
  the `Text` component and its `textOverflow` attribute with
  `TextOverflow.MARQUEE`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-text
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-gradient-color.md` -
  `linearGradient` with `direction`, `colors` and `GradientDirection`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-gradient-color
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` -
  the default key generator `(item: Object, index: number) => index + '__' + JSON.stringify(item)`
  and the warning against index-based keys.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-references/03_system/js-apis-hilog.md` -
  `info(domain, tag, format, ...args)` and the requirement that the variadic
  arguments map to identifiers in the format string.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
