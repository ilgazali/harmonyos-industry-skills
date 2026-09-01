---
id: SHOP-18
title: Clearing unread badges - a traced singleton model so one write repaints one row
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/18_clear_unread_messages.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/clear_unread_messages-0000002341399506
sample: huawei_industry_tree/16_shopping/downloads/ClearUnreadMessages.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: ["@ObservedV2", "@Trace", List, ListItem, ForEach, LongPressGesture, TapGesture, GestureGroup, GestureMode, CustomDialogController, "@CustomDialog", "@Link", "@StorageLink", "display.getDefaultDisplaySync", "UIContext.px2vp", "UIContext.getPromptAction", showToast, "window.getWindowAvoidArea", setWindowLayoutFullScreen]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-16-0027, HW-16-0032, HW-16-0033]
status: verified
---

## When to use

Load this card when **an unread count is shown in more than one place and has
to be cleared from more than one place** - a chat list with per-conversation
badges plus a "mark all read" button in the header, a notification centre, an
order list with "new" markers. The hard part is never the rendering; it is that
the header button, the row itself and a long-press menu all mutate the same
counter and every view must agree instantly.

The pattern: put the list in an `@ObservedV2` singleton, mark the mutable
fields `@Trace`, and let every component that needs it read
`MessageInfoModel.instance` directly. No `@Provide`/`@Consume` chain, no
`@Link` threading through intermediate components, no manual refresh call. A
component two levels down a `@CustomDialog` writes `numbers = 0` and the badge
in the list repaints - because the badge is bound to that exact field, not to
the array.

This is the V2 answer to a problem V1 handles badly. With `@State` on an array
of plain objects, mutating a field of an element does not invalidate anything;
with `@Observed`/`@ObjectLink` you need a wrapper component per row. `@Trace`
gives per-field observation with neither.

## Feature checklist

- A message list of four conversations: avatar, nickname, preview line,
  timestamp, and a red unread badge.
- The badge is hidden when the count is zero, shows the bare number when the
  count is 1, and shows `n+` otherwise.
- Long-pressing a row highlights it and opens a four-item context menu
  positioned at the finger.
- The menu's first item, 标记已读 (mark as read), clears that one
  conversation's count and closes the menu; the other three items are inert.
- The header's broom icon clears every unread count at once.
- Pressing the broom when nothing is unread raises a toast instead.
- Dismissing the menu by tapping outside restores the row's background.

## Architecture

One `entry` module, four source files, one page.

```
entry/src/main/ets
├── components/MessageInfoCom.ets   the List, the long-press gesture, and the @CustomDialog menu
├── constants/Constants.ets         CommonConstant + MarginPadding - every literal in the UI
├── entryability/EntryAbility.ets   full screen, both avoid areas -> AppStorage
├── entrybackupability/
├── model/MessageInfoModel.ets      @ObservedV2 MessageInfo + the singleton model
└── pages/Pages.ets                 @Entry: header row, MessageInfoCom, bottom tab image
```

The documented tree matches the zip exactly - unusual in this industry, and
worth stating because `SHOP-17` and `SHOP-19` both fail this check.

**The design decision worth copying** is the singleton with a lazy static
getter, reached by direct reference rather than by injection:

```typescript
vm: MessageInfoModel = MessageInfoModel.instance
```

That same line appears in three places - `Pages`, `MessageInfoCom`, and the
`@CustomDialog` struct - and each gets the identical instance. Because the
instance is `@ObservedV2`, the assignment is not a snapshot: the component
subscribes to whichever `@Trace` fields its `build()` actually reads. So
`Pages` re-renders nothing when a single badge clears (it reads no traced
field), `MessageInfoCom` re-renders one `Text` (it reads `item.numbers`), and
the dialog re-renders nothing. Compare the `@Provide`/`@Consume` alternative,
which would force the whole subtree through the provider component and give you
no finer granularity than "the model changed".

The cost is honesty about lifetime: a static singleton lives as long as the
process, so this shape is right for app-global state (unread counts, cart,
session) and wrong for anything scoped to a page.

## Implementation steps

1. **Decorate the item class `@ObservedV2` and every mutable field `@Trace`.**
   `@Trace` on the array in the model handles insert/remove; `@Trace` on
   `numbers` inside `MessageInfo` handles the per-row count. You need both -
   tracing only the array would not observe a field write.
2. **Expose the model as a lazy static singleton** and read it as a plain
   member (`vm = MessageInfoModel.instance`) wherever it is needed.
3. **Give the model the mutations, not the components**: `updateMessage()`
   (clear all), `cleanSingleMessage(index)` (clear one), `statisticMessage()`
   (sum, used to decide whether "clear all" has anything to do).
4. **Render the badge conditionally on the count** and choose the label form
   from the same value - no separate "hasUnread" flag to drift.
5. **Attach `LongPressGesture` to the `ListItem`,** capture
   `event.fingerList[0].globalX/globalY`, and construct the
   `CustomDialogController` inside the handler so the offset can be computed
   from the touch point.
6. **Convert the touch point to a dialog offset relative to screen centre** -
   `CustomDialogController.offset` is measured from the alignment anchor, so
   with `DialogAlignment.Center` the conversion is `-screenWidth/2 + ... + x`.
7. **Zero the open and close animations** (`duration: 0, tempo: 0`) so a
   context menu appears at the finger without a fade.
8. **Restore the row highlight in the controller's `cancel` callback**, which
   is the only hook that fires on a tap-outside dismissal.

## Verified snippets

All snippets are from `ClearUnreadMessages.zip`.

**The traced model — `entry/src/main/ets/model/MessageInfoModel.ets`** (as shipped)

```typescript
@ObservedV2
export class MessageInfo {
  @Trace picture?: Resource;
  @Trace text: ResourceStr;
  @Trace nickname: ResourceStr;
  @Trace time: ResourceStr;
  @Trace numbers?: number;

  constructor(picture: Resource, text: ResourceStr, nickname: ResourceStr, time: ResourceStr, numbers: number) {
    this.picture = picture
    this.text = text
    this.nickname = nickname
    this.time = time
    this.numbers = numbers
  }
}

@ObservedV2
export class MessageInfoModel {
  @Trace dialogList: string[] = ['标记已读', '置顶聊天', '不显示该聊天', '删除该聊天']
  @Trace messageList: MessageInfo[] = [ /* four MessageInfo instances */ ]

  public updateMessage() {                     // 一键清除 - clear every badge
    for (let i = 0; i < this.messageList.length; i++) {
      this.messageList[i].numbers = 0
    }
  }

  public statisticMessage() {                  // used to decide whether "clear all" is a no-op
    let count: number = 0
    for (let i = 0; i < this.messageList.length; i++) {
      count += Number(this.messageList[i].numbers)
    }
    return count
  }

  public cleanSingleMessage(index: number) {
    this.messageList[index].numbers = 0
  }

  private static _instance: MessageInfoModel;

  public static get instance() {
    if (!MessageInfoModel._instance) {
      MessageInfoModel._instance = new MessageInfoModel();
    }
    return MessageInfoModel._instance;
  }
}
```

**Two levels of tracing, and both are needed.** `@Trace messageList` observes
the array identity and its structure; `@Trace numbers` inside `MessageInfo`
observes the field. `updateMessage` writes only fields - it never reassigns the
array - so without the inner `@Trace` the loop would repaint nothing at all.
Conversely, if you later `push` a conversation, the outer `@Trace` is what
picks it up.

`updateMessage` deliberately writes to all four entries even when some are
already zero. That is free: `@Trace` compares before notifying, so writing `0`
over `0` does not invalidate.

The menu labels are `@Trace`d plain strings while every other user-visible
string in the sample is a `$r('app.string...')` resource. The four menu items
are the one untranslatable piece of the UI - a translation gap rather than a
defect, but if you copy this file, move them to `string.json`.

**The badge — `entry/src/main/ets/components/MessageInfoCom.ets`** (as shipped)

```typescript
if (item.numbers !== CommonConstant.JUDGE_MESSAGE_ZERO) {
  Column() {
    Text(item.numbers === CommonConstant.JUDGE_MESSAGE_IF_ONE ? item.numbers + '' : item.numbers + '+')
      .fontSize(CommonConstant.FONT_SIZE_TEN)
      .fontColor(CommonConstant.WHITE)
  }
  .padding({ left: MarginPadding.FIVE, right: MarginPadding.FIVE })
  .justifyContent(FlexAlign.Center)
  .borderRadius(CommonConstant.IMAGE_BORDER_RADIUS)   // 100 - larger than the height, so a pill
  .height(CommonConstant.SIXTEEN)
  .backgroundColor(CommonConstant.MESSAGE_RED_DOT_BACKGROUND_COLOR)
}
```

**This is a hand-rolled badge, not the ArkUI `Badge` component** - worth
knowing before you assume otherwise, because `Badge` would give you `count`,
`maxCount` and automatic positioning for less code. The sample builds it by
hand because it wants the badge in the layout flow (under the timestamp in the
right-hand `Column`), not overlaid on a child. If your badge sits on an avatar,
use `Badge`; if it sits in a row like this, this shape is simpler.

`borderRadius(100)` on a 16 vp-high box is the standard pill trick - any radius
at or above half the height gives fully rounded ends without measuring.

The label logic is `n` for exactly 1 and `n+` otherwise, so the seeded value 99
renders `99+` and the value 1 renders `1`. Note what that means for a count of
5: it shows `5+`, claiming more than it knows. Real implementations cap
instead - show `n` up to 99 and `99+` above.

**Positioning the context menu at the finger — same file** (as shipped)

```typescript
.gesture(LongPressGesture({ repeat: false })
  .onAction((event: GestureEvent) => {
    this.actionColor = CommonConstant.ACTION_COLOR
    this.listIndex = index
    let localX = event.fingerList[CommonConstant.ZERO].globalX
    let localY = event.fingerList[CommonConstant.ZERO].globalY
    // flip the menu to the left of the finger when the touch is past mid-screen
    let offsetX: number = (localX > 1 / 2 * this.screenWidth) ? localX - 136 : localX
    this.dialogController = new CustomDialogController({
      builder: LoadingDialogExample({ actionColor: this.actionColor, listIndex: this.listIndex }),
      cancel: () => {
        this.actionColor = CommonConstant.WHITE      // tap-outside: un-highlight the row
      },
      offset: {
        dx: -1 / 2 * this.screenWidth + 68 + offsetX,
        dy: -1 / 2 * this.screenHeight + 100 + localY
      },
      alignment: DialogAlignment.Center,
      autoCancel: true,
      maskColor: Color.Transparent,
      openAnimation: { duration: CommonConstant.ZERO, tempo: CommonConstant.ZERO },
      closeAnimation: { duration: CommonConstant.ZERO, tempo: CommonConstant.ZERO },
    })
    this.dialogController.open()
  })
)
```

**Three things make this behave like a context menu rather than a dialog.**
`alignment: DialogAlignment.Center` plus an offset computed as
`-half the screen + half the dialog + touch` is the arithmetic that converts a
screen coordinate into a centre-relative one; 68 is half the dialog's 136 vp
width and 100 is a hand-tuned vertical nudge. The zeroed open/close animations
remove the dialog fade, which is correct for a menu that should feel attached
to the finger. `maskColor: Color.Transparent` with `autoCancel: true` keeps the
tap-outside dismissal while removing the scrim that would say "modal".

The left/right flip is the only responsive behaviour: past mid-screen the menu
is drawn 136 vp to the left of the touch so it does not run off the edge. There
is no equivalent flip vertically, so a long-press near the bottom of a short
window puts part of the menu off-screen.

`screenWidth`/`screenHeight` come from `display.getDefaultDisplaySync()`
converted with `px2vp` in `aboutToAppear`. That is the *display*, not the
window - correct here because the app is full-screen, wrong the moment it runs
in a floating or split window on a 2in1, which `deviceTypes` does list.

**Clearing, from both entry points** (as shipped)

```typescript
// pages/Pages.ets - the header broom
Image($r('app.media.clean_icon'))
  .onClick(() => {
    if (this.vm.statisticMessage() === CommonConstant.JUDGE_MESSAGE_ZERO) {
      this.promptAction.showToast({ message: $r('app.string.Pages_promptAction') })
    } else {
      this.vm.updateMessage()
    }
  })

// components/MessageInfoCom.ets - the menu's first item, 标记已读
TapGesture({ count: CommonConstant.ONE })
  .onAction(() => {
    if (index !== CommonConstant.JUDGE_MESSAGE_ZERO) {
      return                                   // items 1-3 are decorative
    }
    this.actionColor = CommonConstant.WHITE
    this.listColor = CommonConstant.ACTION_COLOR
    this.vm.cleanSingleMessage(this.listIndex)
    this.controller?.close()
  })
```

**Neither handler touches the UI it affects.** The broom calls a model method
and the list repaints; the menu item calls another model method and one badge
disappears. That is the payoff of the traced singleton - the dialog does not
need a callback back to the list, and the header does not need a reference to
it. `statisticMessage()` in the guard is a read of traced fields, so the header
would also re-render on a count change if it displayed the total.

The menu items use a `GestureGroup(GestureMode.Exclusive, LongPressGesture,
TapGesture)` so both a tap and a long-press activate 标记已读, with the
long-press variant giving a pressed-state colour on `onAction` and committing
on `onActionEnd`. `GestureMode.Exclusive` is what stops both firing.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

`deviceTypes` is the widest in this industry - `phone`, `tablet`, `2in1`,
`wearable` - which the layout does not really earn: the list height is a fixed
288 vp, the row height a fixed 72 vp, and the dialog geometry is computed from
the display rather than the window.

`EntryAbility` sets full-screen layout and publishes both avoid areas:

```typescript
AppStorage.setOrCreate('getTopHeight', avoidArea.topRect.height);
AppStorage.setOrCreate('getBottomHeight', avoidAreaNavigation.bottomRect.height);
```

`Pages` reads them with `@StorageLink` and applies only the top one as padding.
Two notes: `@StorageProp` is the right decorator here since the page only reads
(`@StorageLink` opens a write path back into `AppStorage` that nothing uses),
and the keys `getTopHeight`/`getBottomHeight` differ from the
`topRectHeight`/`bottomRectHeight` used by `SHOP-19` and `SHOP-20` - if you
merge these samples into one app, pick one naming scheme. There is also no
`avoidAreaChange` subscription, so the padding is frozen at whatever it was at
window-stage creation.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The conversation list is four hardcoded entries in `MessageInfoModel`, with
  timestamps as literal strings (`'昨天'`, `'星期五'`). There is no
  persistence, so every relaunch restores the unread counts.
- Row borders are drawn by comparing `index === JUDGE_MESSAGE_IF_THREE` (3) to
  suppress the separator on the last row. That constant hardcodes the list
  length; a fifth conversation gets a stray separator and the fourth loses one.
  Use `index === list.length - 1`.
- `MESSAGE_LIST_HEIGHT` is a fixed 288 vp (4 x 72), so the `List` does not grow
  with the data and does not scroll to reach a fifth item.
- Three of the four menu items (置顶聊天 pin, 不显示该聊天 hide,
  删除该聊天 delete) return early and do nothing. Only 标记已读 is wired.
- `MessageInfo.numbers` is optional (`?: number`), which is why
  `statisticMessage` wraps each read in `Number(...)`. Since the constructor
  always assigns it, the optionality buys nothing and costs the coercion.

## Pitfalls

- **No findings were filed against this document or sample.** The review found
  the doc's single snippet syntactically valid and correctly abridged - it is
  not among the instances of `HW-16-0013`, the corpus-wide finding on truncated
  excerpts - and found no behavioural defect in the shipped code.
- The items above under Constraints are design limits of a demo, not defects:
  the hardcoded last-row index, the fixed list height, the frozen avoid area,
  and `@StorageLink` where `@StorageProp` would do. Fix them when lifting the
  pattern into a real list.
- **`n+` for any count above 1** overstates every count between 2 and 98. Cap
  at 99 instead if the number is meant to be read literally.
- **The singleton outlives the page.** `MessageInfoModel.instance` is process-
  scoped; do not use this shape for state that should reset when the user
  leaves the screen.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-new-observedv2-and-trace.md` - `@ObservedV2`, `@Trace`, and which writes are observed
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-observedv2-and-trace
- `documentation/harmonyos-guides/03_application-framework/arkts-state-management-v2.md` - the V2 model and when to prefer it over `@Observed`/`@ObjectLink`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-state-management-v2
- `documentation/harmonyos-references/02_application-framework/ts-methods-custom-dialog-box.md` - `CustomDialogController`, `offset`, `alignment`, `cancel`, `openAnimation`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-methods-custom-dialog-box
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-longpressgesture.md` - `LongPressGesture` and `fingerList`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-longpressgesture
- `documentation/harmonyos-references/02_application-framework/gesture-binding.md` - `GestureGroup` and `GestureMode.Exclusive`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/gesture-binding
- `huawei_industry_tree/16_shopping/docs/18_clear_unread_messages.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/clear_unread_messages-0000002341399506
- `SHOP-17` - the other `@ObservedV2` / `@Trace` sample in this industry, tracing animation geometry instead of counts
