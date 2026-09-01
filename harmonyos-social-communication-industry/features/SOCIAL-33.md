---
id: SOCIAL-33
title: Chat multi-select and forward - long-press popup, checkbox mode, and a merged-record message
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/33_chat_multi_selection.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_multi_selection-0000002332574910
sample: huawei_industry_tree/14_social_communication/downloads/聊天消息多选及转发示例代码.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit", "@kit.SensorServiceKit"]
apis: [List, ListScroller, ForEach, bindPopup, LongPressGesture, Checkbox, "vibrator.startVibration", NavPathStack, NavDestination, RichEditor, RichEditorController, "@Provide", "@Consume", "@Watch", "@StorageProp"]
permissions: [ohos.permission.INTERNET, ohos.permission.VIBRATE]
min_api: 17
modules: [entry]
findings: [HW-14-0003, HW-14-0070, HW-14-0071, HW-14-0072, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card when a message list needs a **selection mode**: long-press one
item, pick "multi-select", tick several, act on the set. The sample builds the
chat variant - select messages, forward them to a contact as a single merged
record - but the shape is the same for deleting mail, sharing photos, or
batch-tagging anything rendered by a `List`.

The pattern is three states layered over one list: a normal mode with an input
box at the bottom, a popup anchored on the long-pressed bubble, and a selection
mode that swaps the input box for a toolbar and injects a `Checkbox` in front
of every row. No new page, no dialog - the same `ForEach` renders both modes and
a single boolean decides which.

The part worth studying beyond the demo is the **merged message**: the forwarded
payload is not N messages appended to the target chat, it is one message whose
body is a nested `ChatInfo`. That makes forwarding recursive for free and gives
the receiving chat a compact card instead of a wall of text. Read `HW-14-0070`
and `HW-14-0071` before shipping: the selection is stored in a `Set<number>`
that is sorted with the wrong comparator and never cleared.

## Feature checklist

- A chat list of ten conversations; tapping one pushes a chat detail page.
- Long-pressing any bubble for 500 ms vibrates the device and shows a black
  popup above the bubble with 多选 (multi-select) and 删除 (delete).
- Choosing 多选 enters selection mode: a circular checkbox appears beside every
  message, on the left for the peer's messages and on the right for your own.
- The bottom input box is replaced by a toolbar; the share and trash icons are
  greyed while nothing is selected and enabled once at least one box is ticked.
- Tapping the enabled share icon leaves selection mode and pushes a contact
  picker built from the same chat list.
- Picking a contact clears the navigation stack and opens that conversation with
  the forwarded block appended as one 聊天记录 (chat record) card.
- Tapping the card opens a history page listing the forwarded messages.
- Pressing back while in selection mode leaves the mode instead of leaving the
  page.

## Architecture

One `entry` module, four pages and three components. The state model is two
interfaces in `common/ChatInfo.ets` plus a static array of ten conversations.

```
entry/src/main/ets
├── common
│   ├── ChatInfo.ets            MsgContent / ChatInfo interfaces + the 10 seeded chats
│   └── Constants.ets           three NavDestination names
├── component
│   ├── ChatListView.ets        the conversation list, parameterised by ChatListViewMode
│   ├── CustomRichEditor.ets    the bottom input box and the send button
│   └── MergedChat.ets          the forwarded-record card (first 4 lines, then a page)
├── entryability/EntryAbility.ets   avoid areas -> AppStorage
├── entrybackupability/
└── pages
    ├── ChatDetail.ets          @Component, 315 lines: the list, the popup, selection mode
    ├── ChatHistory.ets         the expanded view of a merged record
    ├── ChatSelection.ets       the forward target picker (ChatListView in SELECT_MODE)
    └── Index.ets               @Entry, the Navigation host and the navDestination builder
```

The documented tree lists the same eight files, but the annotations on two of
them are swapped: `CustomRichEditor.ets` is labelled 合并聊天组件 (merged-chat
component) and `MergedChat.ets` 消息发送组件 (message-sending component). The
files themselves are exactly the other way round.

**The design decision worth copying** is the recursive message type:

```typescript
export interface MsgContent {
  isSelf: boolean
  content: string[]
  merged?: ChatInfo      // a message can BE a chat
}
```

A forwarded block is one `MsgContent` whose `merged` field holds a whole
`ChatInfo` - avatars, names, and the selected `msgContents`. The chat list
renders such a message as a `MergedChat` card instead of a bubble, and because
the nested `ChatInfo` has the same shape, a record can contain another record;
`MergedChat` handles that case by printing `[合并消息]` for any child that
itself has a `merged` field. Forwarding a forward costs no extra code.

The second reusable choice is `ChatListView` taking a mode enum:

```typescript
export enum ChatListViewMode { CHAT_MODE, SELECT_MODE }
```

`Index` mounts it bare (chat mode, tap opens the conversation) and
`ChatSelection` mounts it as `ChatListview({ mode: ChatListViewMode.SELECT_MODE,
mergedChatInfo: this.mergedChatInfo })`. One list component, two jobs, and the
forward picker automatically looks exactly like the chat list because it is the
chat list.

## Implementation steps

1. **Host everything in one `Navigation`.** `Index` provides
   `@Provide('navPathStack') navPathStack` and a `navDestinations` builder that
   maps the three constant names to `ChatDetail`, `ChatSelection` and
   `ChatHistory`. Every page reads the stack with `@Consume`.
2. **Read the route parameter in `onReady`,** not in `aboutToAppear` -
   `NavDestination.onReady` is the point at which
   `navPathStack.getParamByName(CHAT_PAGE)[0]` is populated.
3. **Bind the popup to the bubble, not to the page.** `bindPopup(index ===
   this.popupIndex, ...)` on the message `Text` makes a single `popupIndex`
   state drive one popup per list item, and `onStateChange` resets it to `-1`
   when the popup is dismissed by tapping outside.
4. **Trigger the popup from `LongPressGesture({ duration: 500 })`** and vibrate
   first. Set `popupIndex` outside the vibration callback if the popup must
   appear on devices without a motor - as shipped it is assigned inside the
   callback's success path.
5. **Swap the bottom bar on the mode flag.** `if (this.multiSelectMode)
   this.bottomToolbar() else CustomRichEditor(...)`; the checkboxes appear
   inside the same `ForEach` branches under the same condition.
6. **Keep the selection in a `Set<number>` held as `@State`.** Add and delete
   from `Checkbox.onChange`; the toolbar's enabled/disabled rendering reads
   `size > 0`, which only re-renders because ArkUI observes `Set` mutations for
   `@State` from API 12.
7. **Sort the selected indices with a numeric comparator** before building the
   forwarded array - the default `Array.prototype.sort` is lexicographic
   (`HW-14-0070`).
8. **Clear the `Set` on every exit from selection mode** - the share handler,
   the back handler, both (`HW-14-0071`).
9. **Give the delete popup item and the trash icon handlers, or remove them**
   (`HW-14-0072`).
10. **Declare only the permissions you use.** The sample declares `INTERNET`
    and `VIBRATE`; nothing in the code opens a socket (`HW-14-0003`).

## Verified snippets

All snippets are from `聊天消息多选及转发示例代码.zip` (`ChatMultiSelection`).
Corrected forms are marked.

**Long-press to popup — `entry/src/main/ets/pages/ChatDetail.ets`** (as shipped)

```typescript
@Builder
normalMsgBuilder(item: MsgContent, index: number) {
  Text() {
    ForEach(item.content, (content: string) => {
      Span(content).fontSize(18);
    });
  }
  .constraintSize({ maxWidth: '75%', minHeight: 44 })
  .bindPopup(index === this.popupIndex, {
    builder: this.popupBuilder,
    placement: Placement.Top,
    popupColor: Color.Black,
    backgroundBlurStyle: BlurStyle.NONE,
    onStateChange: (e) => {
      if (!e.isVisible) {
        this.popupIndex = -1;
      }
    }
  })
  .gesture(
    LongPressGesture({ duration: 500 })
      .onAction(() => {
        vibrator.startVibration({ type: 'time', duration: 80 },
          { id: 0, usage: 'alarm' },
          (error: BusinessError) => {
            if (error) {
              hilog.error(0x0000, 'testTag', 'Failed to start vibration. Cause: %{public}s',
                JSON.stringify(error));
              return;
            }
            this.popupIndex = index;
          });
      })
  )
}
```

**One `popupIndex` for the whole list is the trick.** `bindPopup` takes a
boolean, so a naive implementation needs one boolean per row; comparing the
row's `index` against a single number gives the same result and guarantees only
one popup can be open. `onStateChange` is mandatory rather than decorative -
`bindPopup` reads the condition but never writes it back, so without the reset
to `-1` a dismissal by tapping outside would leave the state believing the popup
is still up, and long-pressing the same bubble again would do nothing.

`backgroundBlurStyle: BlurStyle.NONE` is what makes `popupColor: Color.Black`
render as solid black; the default blur would wash the black out over the grey
chat background. Note the ordering hazard in the gesture: `popupIndex` is
assigned inside the vibrator's completion callback, so on a device where
`startVibration` fails - no motor, or `ohos.permission.VIBRATE` refused - the
early `return` swallows the long press and no popup appears at all.

**Building the forwarded payload — same file** (corrected, see `HW-14-0070`, `HW-14-0071`, `HW-14-0072`)

```typescript
@Builder
bottomToolbar() {
  Row({ space: 4 }) {
    if (this.multiSelectIndex.size > 0) {
      Image($r('app.media.share_enable'))
        .size({ height: 24, width: 24 })
        .onClick(() => {
          this.multiSelectMode = false;
          if (this.multiSelectIndex.size > 0) {
            let multiSelChatInfo: ChatInfo = {
              thisAvatar: this.chatInfo!.thisAvatar,
              toAvatar: this.chatInfo!.toAvatar,
              thisName: this.chatInfo!.thisName,
              toName: this.chatInfo!.toName,
              msgContents: [],
              lastTimestamp: this.chatInfo!.lastTimestamp
            }
            let tmpArr: number[] = Array.from(this.multiSelectIndex);
            tmpArr = tmpArr.sort((a, b) => a - b);   // FIX: shipped code calls sort() bare
            tmpArr.forEach(idx => {
              multiSelChatInfo.msgContents.push(this.chatInfo!.msgContents[idx]);
            })
            this.navPathStack.pushPath({ name: CHAT_SELECTION_PAGE, param: multiSelChatInfo })
          }
          this.multiSelectIndex.clear();          // FIX: absent - the Set survives the exit
        })
      Image($r('app.media.trash_enable'))
        .size({ height: 24, width: 24 })
        .onClick(() => { this.deleteSelected(); })   // FIX: shipped icon has no handler
    } else {
      Image($r('app.media.share')).size({ height: 24, width: 24 })
      Image($r('app.media.trash')).size({ height: 24, width: 24 })
    }
  }
  .height(52)
  .justifyContent(FlexAlign.SpaceAround)
  .width('100%')
}
```

**Three defects live in these twenty lines, and they share one root cause:** the
selection is a `Set` that nobody owns. `Array.from(set).sort()` coerces to
strings, so a chat grown past ten messages through the input box forwards
`[10, 2]` in that order and the merged record reads backwards. The `Set` is
never emptied - neither here nor in `onBackPressed`, which only flips
`multiSelectMode` - so re-entering selection mode shows unticked checkboxes over
a toolbar that is already enabled, and the next share forwards the previous
selection. And the trash icon in the enabled branch carries no `onClick` at all,
matching the 删除 entry in the popup builder, which is a `Column` with an image
and a label and no handler.

The enabled/disabled split by `if` rather than by `.enabled(...)` is a
deliberate choice worth noting: it swaps the *asset* (`share_enable` vs
`share`), so the disabled state is a different icon rather than a dimmed one.

**Selection mode inside the list — same file** (as shipped)

```typescript
List({ scroller: this.listScroller }) {
  ForEach(this.msgContents, (item: MsgContent, index: number) => {
    ListItem() {
      if (!item.isSelf) {                                  // 对方消息 - the peer's message
        Flex({ direction: FlexDirection.Row, justifyContent: FlexAlign.Start }) {
          if (this.multiSelectMode) {
            Checkbox({ name: 'checkbox' + index, group: 'checkboxGroup' })
              .shape(CheckBoxShape.CIRCLE)
              .onChange((selected) => this.selectCheckBox(selected, index))
              .margin({ right: 10 })
          }
          Image(this.chatInfo?.toAvatar).width(40).borderRadius(20).aspectRatio(1)
          this.normalMsgBuilder(item, index)
        }.width('100%')
      } else {                                             // 我的消息 - my own message
        Flex({ direction: FlexDirection.RowReverse, justifyContent: FlexAlign.SpaceBetween }) {
          Row() {
            if (item.merged) {
              MergedChat({ mergedChatInfo: item.merged })
            } else {
              this.normalMsgBuilder(item, index)
            }
            Image(this.chatInfo?.thisAvatar).width(40).borderRadius(20).aspectRatio(1)
          }
          if (this.multiSelectMode) {
            Checkbox({ name: 'checkbox' + index, group: 'checkboxGroup' })
              .shape(CheckBoxShape.CIRCLE)
              .onChange((selected) => this.selectCheckBox(selected, index))
          }
        }.width('100%')
      }
    }
  }, (item: MsgContent, index: number) => `${JSON.stringify(item)}_${JSON.stringify(index)}`)
}

selectCheckBox(selected: boolean, index: number) {
  if (selected) {
    this.multiSelectIndex.add(index);
  } else {
    this.multiSelectIndex.delete(index);
  }
}
```

**`FlexDirection.RowReverse` is what mirrors the row**, and it is why the
checkbox declaration order looks inverted for your own messages: written after
the bubble, laid out before it. That keeps the checkbox column aligned on the
outer edge of the screen in both directions, which is what users expect from a
selection list.

Two things here are not worth copying. The `ForEach` key stringifies the whole
message on every diff pass - correct, since it makes the key change when the
content changes, but O(content) per row per render. And `merged` is only checked
in the `isSelf` branch: a merged record received from the peer would fall
through to `normalMsgBuilder` and render as an empty bubble, because a merged
`MsgContent` carries `content: []`.

**Receiving the forward — `entry/src/main/ets/component/ChatListView.ets`** (as shipped)

```typescript
.onClick(() => {
  switch (this.mode) {
    case ChatListViewMode.CHAT_MODE:
      this.navPathStack.pushPath({ name: CHAT_PAGE, param: info })
      break;
    case ChatListViewMode.SELECT_MODE:
      this.navPathStack.clear()
      if (this.mergedChatInfo) {
        info.msgContents.splice(7) // 构造数据，每次转发前清理之前的内容
        info.msgContents.push({ isSelf: true, content: [], merged: this.mergedChatInfo })
      }
      this.navPathStack.replacePath({ name: CHAT_PAGE, param: info })
      break;
  }
})
```

`navPathStack.clear()` followed by `replacePath` is the right end to a forward:
the user should land in the target conversation with no way to "back" into the
picker or the source chat's selection mode. Compare with `CHAT_MODE`, which
simply pushes.

The `splice(7)` is demo scaffolding, and the comment says so - it truncates the
target chat back to its seven seeded messages so a second forward does not
accumulate. In a real client that line is the bug: it discards everything the
user has actually sent. Note also that `info` here is an element of the static
`chatMsgs` array, so the push mutates module-level state that survives the page.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" },
  { "name": "ohos.permission.VIBRATE" }
]
```

- `VIBRATE` is the one that is actually used, by `vibrator.startVibration` in
  the long-press gesture. It is `system_grant`, so no runtime request is needed.
- `INTERNET` is never used - the sample has no network code at all. It is part
  of the copy-pasted template config called out by `HW-14-0003`, which also
  covers the dead `REQUEST_PERMISSIONS` location constants in sibling samples
  (`SOCIAL-34` among them).
- Neither entry carries `reason` or `usedScene`, which is correct for
  `system_grant` permissions and would be mandatory if either were `user_grant`.

## Constraints

- API Version 17 Release or later; HarmonyOS 5.0.5 Release SDK or later;
  DevEco Studio 5.0.5 Release or later.
- `@State multiSelectIndex: Set<number>` relies on ArkUI observing `Set`
  mutations, available from API 12. On an older baseline the toolbar would not
  re-render when the first box is ticked.
- The chat data is a module-level constant in `ChatInfo.ets`; all ten
  conversations start from copies of the same seven messages. There is no
  persistence - everything sent or forwarded is lost on restart.
- Selection state is per-page: `multiSelectIndex` indexes `chatInfo.msgContents`
  directly, so any insertion into the array while selection mode is open (a new
  message arriving) silently shifts the meaning of the stored indices. Key the
  selection by message identity if messages can arrive during selection.
- The delete half of the feature does not exist; only forward is implemented.

## Pitfalls

- **`HW-14-0070`** (B/medium, confirmed): `Array.from(this.multiSelectIndex).sort()`
  sorts numbers lexicographically, so a selection including index 10 or above
  forwards out of chronological order. Fix: `sort((a, b) => a - b)`.
- **`HW-14-0071`** (B/medium, confirmed): `multiSelectIndex` is never cleared -
  neither the share handler nor `onBackPressed` empties it - so re-entering
  selection mode shows an enabled toolbar over unticked checkboxes and forwards
  the stale selection. Fix: call `multiSelectIndex.clear()` on every exit from
  the mode.
- **`HW-14-0072`** (B/low, confirmed): the 删除 popup entry and the enabled
  trash icon have no `onClick`; tapping delete does nothing and does not even
  close the popup. Fix: wire the handlers or drop both controls.
- **`HW-14-0003`** (D/low, confirmed): systematic copy-pasted permission config -
  this sample declares `ohos.permission.INTERNET` with no network code, and the
  same template leaves dead `LOCATION` constants in three sibling social
  samples. Fix: delete the unused `requestPermissions` entries.

## References

- `huawei_industry_tree/14_social_communication/docs/33_chat_multi_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_multi_selection-0000002332574910
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `ListScroller`, `scrollEdge`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-popup.md` - `bindPopup`, `PopupOptions`, `onStateChange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-popup
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-longpressgesture.md` - `LongPressGesture` and its `duration`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-longpressgesture
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-checkbox.md` - `Checkbox`, `CheckBoxShape`, `group`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-checkbox
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navdestination.md` - `onReady`, `onBackPressed`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navdestination
- `documentation/harmonyos-references/03_system/js-apis-vibrator.md` - `vibrator.startVibration` and its callback contract
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-vibrator
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - `NavPathStack`, `replacePath`, `clear`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
