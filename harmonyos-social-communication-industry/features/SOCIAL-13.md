---
id: SOCIAL-13
title: Unread badges on a chat list - Badge counts derived from message status, swipeAction to mark read or unread
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/13_chat_unread_reminder.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_unread_reminder-0000002282675266
sample: huawei_industry_tree/14_social_communication/downloads/ChatUnreadReminder.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [hilog, util, window]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0028, HW-14-0029, HW-14-0030, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card when you are building **the conversation list of a messenger**:
one row per peer, an unread count on the avatar, the last message and a
relative timestamp under the name, and a swipe that offers "mark read" /
"mark unread". It is the screen every chat app opens on.

The pattern is: keep messages in one store keyed by peer, and **derive**
everything the row shows - the badge number, the preview line, the time
string - from that store rather than storing them as independent fields.
The badge is then never wrong by construction, because it is a `filter().length`
over the messages, not a counter someone forgot to decrement.

It generalises past chat: a mail inbox, a notification centre, a support-ticket
queue. Anywhere a list row summarises a collection, the same rule holds - one
source of truth, recompute on change. This sample is a good illustration of
that rule and simultaneously the best available illustration of what happens
when you break it: it keeps **two** message stores and only writes to one of
them (`HW-14-0028`). Read that finding before copying the structure.

## Feature checklist

- A chat list of eight conversations, each with avatar, peer name, last
  message preview and a relative time ("刚刚", "3分钟前", "2小时前", a date).
- Simulated incoming messages arrive on four independent timers (5s, 15s, 30s,
  70s), one per peer.
- Each arrival bumps that row's unread badge and rewrites its preview line and
  timestamp.
- The badge sits on the top-right of the avatar and disappears at zero.
- Relative times refresh once per second while the list is on screen.
- Swiping a row left reveals two buttons: a mark button whose label flips
  between 标为已读 (mark read) and 标为未读 (mark unread) with the current
  count, and a delete button that removes the conversation.
- Opening a conversation clears its badge; going back keeps it cleared.
- All timers are cleared when the list is destroyed.

## Architecture

One `entry` module, ten ArkTS files. State lives entirely in `HomePage` and is
handed down by `@Provide` / `@Consume`.

```
entry/src/main/ets
├── components/
│   ├── MessageSend.ets      the RichEditor composer + the send button
│   ├── MyChat.ets           the chat list: timers, badges, swipe actions (299 lines)
│   └── TabContent.ets       BottomNavigationModel + the four BUTTON_INFO entries
├── constant/Constants.ets   names, canned messages, intervals, magic numbers
├── entryability/EntryAbility.ets    full screen + avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── model/
│   ├── Message.ets          Message class + Status enum
│   ├── SessionData.ets      SESSION_DATA, eight static conversations
│   └── SessionItem.ets      id / receiver / avatar
├── pages/
│   ├── ChatPage.ets         the NavDestination conversation view
│   └── HomePage.ets         @Entry: Navigation + Tabs + all the @Provide state
└── utils/AppRouter.ets      a singleton wrapper over NavPathStack
```

The documented tree matches the zip exactly - worth stating, because four other
samples in this industry document files their zips do not contain
(`HW-14-0001`).

**The design decision worth avoiding** is the message storage. `HomePage`
declares *two* stores over the same data:

```typescript
@StorageLink('message') data: Message[] = [];              // flat, global
@Provide chatMessage: Map<string, Message[]> = new Map();  // grouped by peer
```

plus three derived Maps (`lastMessage`, `reminders`, `timeStr`), all keyed by
the peer's name. The Map-of-arrays keyed by peer is the good half: it makes
"the messages of this conversation" an O(1) lookup and makes the derived Maps
trivially parallel. The `@StorageLink('message')` array is the bad half. It
exists so `ChatPage` can `@Watch` it and autoscroll, but the simulated arrivals
in `MyChat.action()` only push into `chatMessage` - so the array that drives
autoscroll never sees a received message, and the read-marking loops that walk
`this.data` iterate over nothing (`HW-14-0028`).

If you copy this structure, copy it with one store. Keep the `Map<string,
Message[]>`, drop the flat array, and let the conversation view watch
`chatMessage` (which `ChatPage` already does, via
`@Watch('onAllMessageChanged')`).

Note also that the peers are keyed **by display name** (`Constants.NAME_ONE` =
'大侠'). That is fine for a demo with eight hardcoded contacts and wrong the
moment two contacts share a name. Key by `SessionItem.id`.

## Implementation steps

1. **Model the message with a status enum**, not a boolean. `Status.SENT` means
   delivered-but-unread here; `Status.READ` is set when the recipient opens the
   conversation. The unread count is a filter over that enum, never a counter.
2. **Group messages into a `Map<string, Message[]>`** in `aboutToAppear`, one
   empty array per entry of `SESSION_DATA`, before any timer can fire.
3. **Simulate arrivals with `setInterval`,** one timer per peer, and store every
   timer id. Push each new message into **every** store that any view reads -
   or keep exactly one store (`HW-14-0028`).
4. **Recompute the badge after each arrival** with `filter(status === SENT &&
   receiverId === user).length`. Never `count++`.
5. **Recompute the preview line** from the last element of the peer's array,
   choosing the key by direction (`senderId === user ? receiverId : senderId`).
6. **Run one extra 1s timer** that re-renders the relative timestamps; the
   message data does not change, only the string derived from
   `now - timestamp`.
7. **Clear every timer in `aboutToDisappear`.** The sample does this correctly
   and it is the part most demos get wrong.
8. **Bind `Badge` around the avatar** with `count` read from the reminders Map
   and `position: BadgePosition.RightTop`.
9. **Attach `swipeAction({ end: { builder } })` to the `ListItem`** and build
   the two buttons in a `@Builder` that receives the item and its index.
10. **Make "mark unread" change message state, not just the badge number**
    (`HW-14-0030`) - otherwise the next recompute erases it.
11. **Wire the tab bar's `onClick` to a `TabsController`** if you use a custom
    `tabBar` builder that fills the whole bar cell; an empty `onClick` swallows
    the tap that would otherwise reach the `Tabs` (`HW-14-0029`).
12. **Clear the badge on entry to the conversation** and again in
    `onBackPressed`, because messages can arrive while the conversation is open.

## Verified snippets

All snippets are from `ChatUnreadReminder.zip`. Corrected forms are marked.

**Timers, arrival, and the derived counters — `entry/src/main/ets/components/MyChat.ets`**
(corrected, see `HW-14-0028`)

```typescript
@Consume chatMessage: Map<string, Message[]>;   // 聊天信息集合
@Consume lastMessage: Map<string, Message>;     // 最后一条信息集合
@Consume reminders: Map<string, number>;        // 未读信息集合
@StorageLink('message') data: Message[] = [];

aboutToAppear(): void {
  // 根据会话列表对消息分类
  this.sessionList = SESSION_DATA;
  this.sessionList.forEach((session: SessionItem) => {
    this.chatMessage.set(session.receiver, []);
  });
  this.startTimers();
  this.timerId = setInterval(() => {
    this.lastMessage.forEach((message: Message, object: string) => {
      this.calculateTimeDiff(message, object);
    })
  }, Constants.REFRESH_INTERVAL)
}

aboutToDisappear() {
  this.clearTimers();
}

private action(name: string, message: string): void {
  let newArray: Message[] = this.chatMessage.get(name) as Message[];
  const incoming = new Message(name, this.user, message, new Date().getTime(), Status.SENT);
  newArray.push(incoming);
  this.chatMessage.set(name, newArray);
  this.data.push(incoming);        // FIX: absent in the sample - the flat store never sees arrivals
  this.getUnreadMessageNum(name);
  this.getLastMessage(name);
}

// 根据会话对象获取未读消息数量
getUnreadMessageNum(object: string) {
  this.sessionList.forEach((session: SessionItem) => {
    if (object === session.receiver) {
      let currentArray: Message[] = this.chatMessage.get(session.receiver) as Message[];
      let count =
        currentArray.filter((item: Message) => item.status === Status.SENT && item.receiverId === this.user).length;
      this.reminders.set(session.receiver, count);
    }
  })
}
```

**The badge is a filter, and that is the whole point.** `getUnreadMessageNum`
never increments anything; it counts the messages that are still `SENT` and
addressed to the local user. Marking a conversation read is therefore a single
pass that flips those messages to `READ`, and the number follows. Any code path
that forgets to bump a counter is impossible here.

`this.chatMessage.set(name, newArray)` after the `push` looks redundant -
`newArray` is the same object that is already in the Map - but it is what
notifies ArkUI that a `@Provide`d `Map` changed. Mutating the array in place
does not; re-`set`ting the key does. Same reason `getLastMessage` and
`calculateTimeDiff` write through `.set` rather than mutating a held reference.

The one-second `timerId` interval exists only to refresh strings like 3分钟前.
It rewrites `timeStr` for every conversation once per second, which is cheap
for eight rows and would not be for eight hundred - derive the label in the row
builder instead if the list is long.

**The badge and the row — same file** (as shipped)

```typescript
@Builder
SessionItem(item: SessionItem, index: number) {
  Row() {
    Badge({
      count: this.reminders.get(item.receiver),
      style: {},
      position: BadgePosition.RightTop,
    }) {
      Image(item.avatar)
        .width($r('app.float.avatar_image_size'))
        .height($r('app.float.avatar_image_size'))
    }
    .width($r('app.float.avatar_image_size'))

    Column() {
      Text(item.receiver)
      Text(this.lastMessage.get(item.receiver)?.content)
        .fontColor($r('app.color.last_message'))
    }
    .justifyContent(FlexAlign.SpaceBetween)
    .alignItems(HorizontalAlign.Start)

    Blank()

    Text(this.timeStr.get(item.receiver))
  }
  .width($r('app.string.whole_width'))
  .height($r('app.float.session_height'))
  .alignItems(VerticalAlign.Top)
  .onClick(() => {
    AppRouter.push(Constants.CHAT_PAGE);
    this.receiver = item.receiver;
    this.chatObject = item;
  })
}
```

**`Badge` wraps the avatar rather than overlaying it.** Passing the `Image` as
`Badge`'s child content is what anchors the dot to the avatar's box, so
`BadgePosition.RightTop` needs no offsets and no `Stack`. `count` of `0` hides
the badge automatically - there is no `visibility` ternary anywhere in this
file. `style: {}` accepts the system defaults for colour and size; fill it in
only when the design demands it.

Three of the row's four texts read out of the derived Maps with the peer name
as key, which is why the row itself carries no state at all. `Blank()` between
the name column and the timestamp is what pushes the time to the right edge
without a fixed width.

**Swipe actions — same file** (corrected, see `HW-14-0030`)

```typescript
ListItem() {
  Column() {
    this.SessionItem(item, index)
  }
}
.swipeAction({
  end: {
    // index为该ListItem在List中的索引值。
    builder: () => {
      this.itemEnd(item, index)
    },
  }
})

@Builder
itemEnd(item: SessionItem, index: number) {
  Row({ space: 10 }) {
    Column() {
      Text(this.reminders.get(item.receiver)! > 0 ? Constants.MARK_READ : Constants.MARK_UNREAD)
    }
    .onClick(() => {
      if (this.reminders.get(item.receiver)! > 0) {
        this.changeMessageStatus(item);          // flips SENT -> READ for this peer
        this.getUnreadMessageNum(item.receiver);
      } else {
        const arr = this.chatMessage.get(item.receiver) as Message[];
        const last = arr[arr.length - 1];
        if (last && last.receiverId === this.user) {
          last.status = Status.SENT;             // FIX: back the manual mark with message state
          this.chatMessage.set(item.receiver, arr);
        }
        this.getUnreadMessageNum(item.receiver); // FIX: sample sets reminders = 1 directly
      }
    })

    Column() {
      Image($r('app.media.delete'))
    }
    .onClick(() => {
      // this.messages为列表数据源，可根据实际场景构造。点击后从数据源删除指定数据项。
      this.sessionList.splice(index, 1);
    })
  }
  .width($r('app.float.item_end_width'))
  .justifyContent(FlexAlign.End)
}
```

**The mark button is a toggle whose label is derived, not stored.** It reads
`Constants.MARK_READ` (标为\n已读) when the count is positive and
`Constants.MARK_UNREAD` (标为\n未读) when it is zero, so a single button covers
both directions and can never show a label that contradicts the badge.

The shipped "mark unread" branch writes `this.reminders.set(item.receiver, 1)`
and stops there - it moves the badge without moving the data behind it. The
very next timer arrival calls `getUnreadMessageNum`, which recounts from
`Status.SENT` messages, and the manual mark evaporates (or, worse, one new
message shows a badge of 1 when the user expects 2). The corrected form flips
the last received message back to `SENT` so the recompute agrees. That is the
general rule for this whole card: if a UI number is derived, the only way to
change it is to change what it derives from.

`swipeAction` takes a `builder` closure rather than a component, which is what
lets `itemEnd` close over both `item` and `index` - the delete button needs the
index, the mark button needs the item.

**The tab bar — `entry/src/main/ets/pages/HomePage.ets`** (corrected, see `HW-14-0029`)

```typescript
@State currentPageIndex: number = Constants.CHAT_INDEX;
private tabsController: TabsController = new TabsController();   // FIX: not in the sample

Tabs({ barPosition: BarPosition.End, controller: this.tabsController }) {
  TabContent() {
    MyChat()
  }.tabBar(this.BottomNavigation(BUTTON_INFO[Constants.CHAT_INDEX]))
  // ... three more TabContent blocks, all empty
}
.scrollable(false)                        // 不可以通过页面滑动切换页面
.onChange((index: number) => {
  this.currentPageIndex = index;
})

@Builder
BottomNavigation(button: BottomNavigationModel) {
  Column({ space: Constants.TAB_SPACE }) {
    // 根据index判断导航按钮是否聚焦
    Image(this.currentPageIndex === button.index ? button.selectedImage : button.primaryImage)
    Text(button.title)
      .fontColor(this.currentPageIndex === button.index ? $r('app.color.tab_selected') : Color.Black)
  }
  .width($r('app.string.whole_width'))
  .height($r('app.string.whole_height'))
  .onClick(() => {
    this.tabsController.changeIndex(button.index);   // FIX: the sample's onClick body is empty
  })
}
```

**An empty `onClick` is not a no-op.** The builder's `Column` fills the whole
bar cell, so registering a click handler on it makes that cell the hit target;
the tap is consumed there and never reaches `Tabs`. Combined with
`.scrollable(false)`, which is deliberate (the comment says so: switching by
page swipe is disabled), nothing can change the tab. The selected-state styling
computed from `currentPageIndex` in the same builder is consequently dead code.

Either delete the `onClick` and let `Tabs` handle the tap itself, or - as here
- keep it and drive a `TabsController`. Do not leave it empty. The sibling
sample `ChatFoldTop` has the same builder guarded by `if (button.index === 0)`,
which is the same bug with three quarters of the tabs dead.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions` at all - correct for
a sample whose only data source is a timer. Compare `HW-14-0003`, where four
other social samples carry copy-pasted `INTERNET` / `VIBRATE` declarations and
dead location-permission constants that nothing uses; this one is clean.

`EntryAbility` sets `COLOR_MODE_LIGHT`, calls `setWindowLayoutFullScreen(true)`
and publishes `topRectHeight` / `bottomRectHeight` into `AppStorage`, which
pages read with `@StorageProp` and convert at the point of use:

```typescript
.padding({
  top: this.getUIContext().px2vp(this.topRectHeight),
  bottom: this.getUIContext().px2vp(this.bottomRectHeight)
})
```

Converting in the binding expression - rather than overwriting the prop once in
`aboutToAppear` - is the right form; `SOCIAL-16` gets this wrong and the
padding silently triples (`HW-14-0035`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Three of the four tabs are empty `TabContent()` blocks, and unreachable
  anyway until `HW-14-0029` is fixed.
- `Message.id` is `util.format('%s-%s', senderId, receiverId)` - a conversation
  id, not a message id. Every message between the same two people shares it.
  Give messages a real unique id before persisting anything.
- Conversations are keyed by display name throughout. Two peers with the same
  name collapse into one row.
- The list's `ForEach` has no key generator, so deleting a conversation
  re-keys by index and can leave stale rows; supply
  `(item: SessionItem) => item.id.toString()`.
- `changeMessageStatus` mutates `Message` objects inside the array. That
  updates the count on the next explicit recompute but does not by itself
  notify any view bound to the message - the `.set()` on the Map does.
- No persistence: everything lives in memory and resets on relaunch.
- Only the first conversation's timers ever fire messages; peers five to eight
  in `SESSION_DATA` have no timer and stay empty forever.

## Pitfalls

- **`HW-14-0028`** (B/medium, confirmed): systematic - the sample keeps two
  message stores and timer-generated arrivals enter only `chatMessage`, never
  the `@StorageLink('message')` array that `ChatPage` watches for autoscroll;
  the read-marking passes over `this.data` are therefore unreachable. Two
  samples share this `MyChat` template (`ChatFoldTop` is the other). Fix: push
  incoming messages into both stores, or collapse to one store.
- **`HW-14-0029`** (B/low, probable): the tab bar is neutralised - an empty
  `onClick` on the full-size tab `Column` consumes the tap while
  `.scrollable(false)` blocks the swipe, so no tab can be selected. Fix: call
  `changeIndex(button.index)` on a `TabsController`, unconditionally, for every
  tab.
- **`HW-14-0030`** (B/low, confirmed): "mark unread" sets `reminders` to 1
  without changing any `Message.status`, so the next recompute (triggered by
  the next timer arrival) overwrites the manual mark and the badge silently
  disappears or miscounts. Fix: flip the last received message back to
  `Status.SENT` and let the count derive itself.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-badge.md` - `Badge`, `BadgePosition`, `BadgeStyle`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-badge
- `documentation/harmonyos-references/02_application-framework/ts-container-listitem.md` - `swipeAction`, `SwipeActionOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-listitem
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `divider`, `ListScroller`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` - `Tabs`, `TabsController.changeIndex`, `scrollable`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-references/05_common-capabilities/js-apis-timer.md` - `setInterval` / `clearInterval`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-timer
- `documentation/harmonyos-guides/03_application-framework/arkts-appstorage.md` - `@StorageLink` vs `@StorageProp`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-appstorage
- `huawei_industry_tree/14_social_communication/docs/13_chat_unread_reminder.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_unread_reminder-0000002282675266
- `SOCIAL-16` - the same avoid-area boilerplate, done wrong
