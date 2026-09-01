---
id: SOCIAL-40
title: Pinned chats and a fold row - a Set drives ordering, two Lists split the session list
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/40_chat_fold_top.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_fold_top-0000002405205461
sample: huawei_industry_tree/14_social_communication/downloads/ChatFoldTop.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [List, ListItem, swipeAction, Badge, Tabs, TabContent, Navigation, NavPathStack, RichEditor, RichEditorController, "@Provide", "@Consume", "@StorageLink", setInterval, clearInterval, hilog, util, window]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-14-0079, HW-14-0080, HW-14-0028, HW-14-0087]
status: verified-with-fixes
---

## When to use

**Load this card when a list has a promoted section that can itself be
collapsed** - pinned chats above ordinary chats, with a one-line "N 个置顶聊天"
header that folds them away when there are too many. It is the standard top of
a messaging app's session list, and the same shape covers pinned notes, starred
mailboxes, favourite boards.

The pattern is deliberately not one list with a sort comparator. It is **two
`List`s stacked in a `Column`** with a header row between them: the first
renders the pinned sessions and can be hidden wholesale, the second renders
everything else. Collapsing then costs one `visibility` toggle instead of a
re-filter, and the header row can live between them as an ordinary component
rather than a sticky list item.

The membership store is a `Set<string>` of receiver ids, not a boolean on the
model. That is the choice worth internalising: the Set is the single source of
truth for "is this pinned", the model's `isPinned` flag exists only so the
swipe button can show the right icon, and ordering falls out of iterating the
Set. It generalises to any tag-like membership that drives both filtering and
ordering.

**Read `HW-14-0079` before copying the document's snippet** - its visibility
condition is inverted relative to the shipped code - and `HW-14-0080` before
shipping the delete action.

## Feature checklist

- A four-tab bottom navigation; only the first tab (消息) has content.
- The chat tab lists eight seeded sessions with avatar, name, last message,
  relative time and an unread `Badge`.
- Relative time refreshes once a second: 刚刚, N 分钟前, N 小时前, then a date.
- Four of the eight contacts receive an automatic incoming message on their own
  interval (15s, 15s, 30s, 70s), bumping their unread count and last message.
- Swiping a session left reveals a pin/unpin button and a delete button.
- Pinning moves the session into a separate list above the others and re-sorts
  the underlying array.
- The pin icon toggles between 置顶 and 取消置顶 art depending on the session's
  current state.
- Once at least one session is pinned, a 32vp header row appears showing
  折叠置顶聊天 when expanded and "N 个置顶聊天" when folded.
- Tapping the header folds or unfolds the pinned list; the header disappears
  entirely when nothing is pinned.
- Deleting a session removes it from the list and from the pinned set.
- Tapping a session opens the chat page, marks its incoming messages read and
  clears the badge.

## Architecture

One `entry` module, eleven source files. A single `@Entry` page hosts a
`Navigation` whose `navDestination` renders the chat page, so there is one
router stack for the whole app held in a singleton.

```
entry/src/main/ets
├── components/
│   ├── MessageSend.ets       RichEditor + send button; writes both message stores
│   ├── MyChat.ets            the whole session list: pinning, folding, timers (412 lines)
│   └── TabContent.ets        BottomNavigationModel + the four tab descriptors
├── constant/Constants.ets    names, canned messages, intervals, indices
├── entryability/EntryAbility.ets   full screen, avoid areas -> AppStorage
├── entrybackupability/
├── model/
│   ├── Message.ets           Message + Status enum (SENT / SENT_READ / DELIVERED / READ)
│   ├── SessionData.ets       SESSION_DATA - eight seeded sessions
│   └── SessionItem.ets       id, receiver, avatar, isPinned
├── pages/
│   ├── ChatPage.ets          NavDestination: the conversation, autoscroll, read-marking
│   └── HomePage.ets          @Entry: Tabs + Navigation + every @Provide in the app
└── utils/AppRouter.ets       singleton NavPathStack wrapper
```

The documented 工程目录 matches the zip.

**The design decision worth copying** is the split into two `List`s with the
fold header between them:

```
List()   ForEach over the pinned sessions        <- .visibility(!isCollapsed)
Row()    the fold header                          <- .visibility(pinnedSessions.size >= 1)
List()   ForEach over the non-pinned sessions
```

Folding is then a property of a container, not of the data. Nothing is
filtered, removed or re-sorted when the user folds; the pinned `List` simply
stops occupying space (`Visibility.None`, not `Hidden`, so the row below moves
up). The header sits between the two lists as a plain `Row`, which is why it
can carry a different layout in each state - an up-arrow and 折叠置顶聊天 when
expanded, a down-arrow and the count when folded - without fighting a list
item's constraints.

**The structure worth avoiding** is in the same component: `HomePage` declares
nine `@Provide` fields (`chatObject`, `chatMessage`, `lastMessage`,
`reminders`, `timeStr`, `sessionList`, `user`, `receiver`, plus a
`@StorageLink('message') data`) and `MyChat` `@Consume`s eight of them. Every
piece of app state is global to the page tree, and the two stores that result -
`chatMessage` and `data` - drift apart (`HW-14-0028`). A `SessionItem`-shaped
view model owned by `MyChat`, with the conversation fetched by receiver id,
would remove both the fan-out and the desync.

## Implementation steps

1. **Store pinning as a `Set<string>` of receiver ids** on the list component,
   and mirror it into `SessionItem.isPinned` only so the swipe button can pick
   its icon.
2. **Render two `List`s** - pinned first, then the header row, then the rest -
   and filter the second list on `!pinnedSessions.has(session.receiver)`.
3. **Iterate the Set in reverse** for the pinned list, so the most recently
   pinned session appears at the top, then map ids back to sessions and
   `filter(Boolean)` to drop any id whose session was deleted.
4. **Hide the pinned list with `!this.isCollapsed ? Visibility.Visible :
   Visibility.None`.** The document's snippet has this condition the wrong way
   round (`HW-14-0079`); copying it folds the list open and unfolds it closed.
5. **Gate the header on `pinnedSessions.size >= 1`** so it never appears over
   an empty pinned section.
6. **Attach `swipeAction({ end: { builder } })` to both lists' `ListItem`s**,
   with the same `itemEnd` builder, so pinned and unpinned rows offer the same
   actions.
7. **Re-sort after every membership change** with a comparator that returns
   `-1`/`1`/`0` on Set membership - a stable sort keeps the relative order of
   unpinned sessions.
8. **Clear the contact's message timer when its session is deleted**
   (`HW-14-0080`), and drop its `chatMessage` / `lastMessage` / `reminders`
   entries with it.
9. **Write incoming messages into the same store the chat page watches**
   (`HW-14-0028`), or the conversation will not autoscroll when a message
   arrives while it is open.
10. **Clear all five intervals in `aboutToDisappear`** - the sample does do
    this, and it is the reason the timers do not survive a tab switch.

## Verified snippets

All snippets are from `ChatFoldTop.zip`. Corrected forms are marked.

**The two lists and the fold header — `entry/src/main/ets/components/MyChat.ets`**
(as shipped)

```typescript
// Session list
List() {
  // Display pinned sessions first
  ForEach(Array.from(this.pinnedSessions).reverse().map(receiver => {
    return this.sessionList.find(session => session.receiver === receiver);
  }).filter(Boolean), (item: SessionItem) => {
    ListItem() {
      Column() {
        this.sessionItem(item);
        Divider().strokeWidth(0.5)
          .color('rgba(0, 0, 0, 0.2)')
          .margin({ left: 64, right: 16 });
      }
      .padding({ left: $r('app.float.margin_sixteen'), right: $r('app.float.margin_sixteen') });
    }
    .backgroundColor('#0D000000')
    .swipeAction({
      end: {
        builder: () => {
          this.itemEnd(item);
        },
      }
    });
  });
}
.visibility(!this.isCollapsed ? Visibility.Visible : Visibility.None);

Row() {
  if (this.isCollapsed) {
    Image($r('app.media.arrow_down_to_line')).width(18).height(18);
    Text(this.pinnedSessions.size + '个置顶聊天')     // "N pinned chats"
      .fontSize(12)
      .fontColor('#99000000');
    Image($r('app.media.downline')).width(18).height(18).position({ x: '90%' });
  } else {
    Image($r('app.media.arrow_up_to_line')).width(18).height(18);
    Text('折叠置顶聊天')                              // "fold pinned chats"
      .fontSize(12)
      .fontColor('#99000000');
    Image($r('app.media.upline')).width(18).height(18).position({ x: '90%' });
  }
}
.visibility(this.pinnedSessions.size >= 1 ? Visibility.Visible : Visibility.None)
.onClick(() => {
  this.isCollapsed = !this.isCollapsed;
})
.height(32)
.backgroundColor('#0d000000');
```

**Three expressions carry the design.** `!this.isCollapsed ? Visible : None`
is the fold - and it is `None`, not `Hidden`, so the folded list surrenders its
height and the header slides up against the search bar. This is exactly the
line the document gets backwards (`HW-14-0079`).

`Array.from(this.pinnedSessions).reverse()` is the ordering rule: a `Set`
preserves insertion order, so reversing it puts the most recently pinned
session on top - the behaviour users expect from pinning. The `.map(find)`
that follows converts ids back to session objects, and `.filter(Boolean)` is
the safety net for an id whose session no longer exists; without it a deleted
pinned session would render `undefined`.

The header's two branches are near-duplicates that differ in arrow direction
and label, which is a fair trade for keeping each state's layout independent.
The grey `#0D000000` background on the pinned `ListItem`s and `#0d000000` on
the header tie the pinned block together visually against the white list below.

**Pin, unpin, delete — same file** (corrected, see `HW-14-0080`)

```typescript
@Builder
itemEnd(item: SessionItem) {
  Row({ space: 0 }) {
    // Pin button
    Column() {
      Image(item.isPinned ? $r('app.media.unpin') : $r('app.media.pin'))
        .width(40)
        .height(40);
    }
    .onClick(() => {
      if (item.isPinned) {
        this.pinnedSessions.delete(item.receiver);
        item.isPinned = false;
      } else {
        this.pinnedSessions.add(item.receiver);
        item.isPinned = true;
      }
      // Re-sort the session list
      this.sortSessionList();
    });

    // Delete session
    Column() {
      Image($r('app.media.delete'))
        .width(40)
        .height(40);
    }
    .margin({ left: 8 })
    .onClick(() => {
      if (item.isPinned) {
        this.pinnedSessions.delete(item.receiver);
      }
      this.clearTimerFor(item.receiver);          // FIX: absent in the sample
      this.chatMessage.delete(item.receiver);     // FIX: map entries were left behind
      this.lastMessage.delete(item.receiver);
      this.reminders.delete(item.receiver);
      const index = this.sessionList.findIndex(session => session.receiver === item.receiver);
      if (index !== -1) {
        this.sessionList.splice(index, 1);
      }
      this.sortSessionList();
    });
  }
  .margin({ right: 28 })
  .height(64)
  .width(130)
  .justifyContent(FlexAlign.End);
}

private sortSessionList(): void {
  this.sessionList.sort((a, b) => {
    if (this.pinnedSessions.has(a.receiver) && !this.pinnedSessions.has(b.receiver)) {
      return -1; // a is pinned, appears first
    } else if (!this.pinnedSessions.has(a.receiver) && this.pinnedSessions.has(b.receiver)) {
      return 1; // b is pinned, appears first
    } else {
      return 0; // Unpinned sessions maintain original order
    }
  });
}
```

**The Set is the truth and `isPinned` is the icon.** Both are written on every
toggle, and the delete path shows why the duplication is a liability: it
removes the id from the Set but never touches `isPinned`, which is harmless
only because the object is being discarded. A comparator over the Set means
`sortSessionList` never has to trust the flag.

`sortSessionList` returning `0` for same-class pairs is what preserves the
seeded order among unpinned sessions - `Array.prototype.sort` is stable in
ArkTS, so equal-ranked items keep their positions. Note that the sort is
strictly cosmetic for the pinned block: the pinned `List` iterates the Set, not
`sessionList`, so re-sorting only matters for the second list and for what
happens if the two lists are ever merged.

The four `FIX` lines are `HW-14-0080` and its tail. Removing a session from
`sessionList` does not stop `timerOne`..`timerFour`, which are keyed to contact
*names* in `Constants` and keep calling `action(name, message)` forever. That
call does `this.chatMessage.get(name) as Message[]` on a map entry no view
reads any more, pushes into it, and recomputes an unread count and a last
message for a contact the user deleted - unbounded array growth and stale
bookkeeping for the lifetime of the page. A per-receiver timer map
(`Map<string, number>`) replacing the four numbered fields makes
`clearTimerFor` a one-liner.

**The message generator, and the store it writes to — same file**
(corrected, see `HW-14-0028`)

```typescript
// Start all timers
private startTimers() {
  this.timerOne = setInterval(
    () => this.action(Constants.NAME_ONE, Constants.MESSAGE_ONE), Constants.FIFTEEN_SECOND);
  // ... timerTwo (15s), timerThree (30s), timerFour (70s), each bound to one contact name
}

// Timer actions
private action(name: string, message: string): void {
  let newArray: Message[] = this.chatMessage.get(name) as Message[];
  let incoming = new Message(name, this.user, message, new Date().getTime(), Status.SENT);
  newArray.push(incoming);
  this.chatMessage.set(name, newArray);
  this.data.push(incoming);          // FIX: absent in the sample - ChatPage watches `data`
  this.getUnreadMessageNum(name);
  this.getLastMessage(name);
}

// Clear all timers - called from aboutToDisappear
private clearTimers() {
  clearInterval(this.timerOne);
  clearInterval(this.timerTwo);
  clearInterval(this.timerThree);
  clearInterval(this.timerFour);
  clearInterval(this.timerId);   // the 1s relative-time refresh
}
```

**There are two message stores and only one of them sees incoming messages.**
`chatMessage: Map<string, Message[]>` is the per-conversation store the list
reads for its last-message line; `data`, reached everywhere through
`@StorageLink('message')`, is the flat store `ChatPage` watches:

```typescript
@StorageLink('message') @Watch('onChangedData') data: Message[] = [];

onChangedData() {
  this.listScroller.scrollEdge(Edge.End); // Scroll to the bottom
}
```

`MessageSend.sendMessage` pushes an outgoing message into **both** (`this.data.push`
and `this.chatData.push`), which is why sending scrolls correctly. `action`
pushes into `chatMessage` only, so a message arriving while the conversation is
open never fires `@Watch` and never scrolls into view - and
`changeMessageStatus`'s first loop, which walks `data` looking for unread
incoming messages to mark read, has nothing to find. Its second loop over
`chatMessage` is what actually does the work; the first is dead code created by
the same split. Pushing into both stores is the one-line fix; collapsing to one
store is the real one.

`clearTimers` in `aboutToDisappear` is correct and worth noting as the
counter-example to the delete path: the component does know how to release its
intervals, it just never does it per-contact.

**A session row — same file** (as shipped)

```typescript
@Builder
sessionItem(item: SessionItem) {
  Row() {
    Badge({
      count: this.reminders.get(item.receiver),
      style: {},
      position: BadgePosition.RightTop,
    }) {
      Image(item.avatar)
        .width($r('app.float.avatar_image_size'));
    }
    .width($r('app.float.avatar_image_size'));

    Column() {
      Text(item.receiver);
      Text(this.lastMessage.get(item.receiver)?.content)
        .fontColor($r('app.color.last_message'));
    }
    .justifyContent(FlexAlign.SpaceBetween)
    .alignItems(HorizontalAlign.Start);

    Blank();

    Text(this.timeStr.get(item.receiver))
      .fontColor($r('app.color.text_time'));
  }
  .height($r('app.float.session_height'))
  .alignItems(VerticalAlign.Top)
  .onClick(() => {
    AppRouter.push(Constants.CHAT_PAGE);
    this.receiver = item.receiver;
    this.chatObject = item;
  });
}
```

**One builder serves both lists**, which is the payoff of splitting on
containers rather than on item types: a pinned row and an ordinary row are the
same component, differing only in the `backgroundColor` their `ListItem`
carries. `Badge` wrapping the avatar means the count follows the avatar's box
automatically, and `count: undefined` - a receiver with no entry in
`reminders` - renders no badge, so the zero case needs no guard. `Blank()`
right-aligns the timestamp without a fixed width.

The click handler writes `receiver` and `chatObject` **after** pushing the
route; that works only because `ChatPage`'s `aboutToAppear` runs after the
transition commits, and writing the state first would be safer.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`; the sample is
entirely local, with eight seeded contacts and canned messages in
`Constants.ets`. Device types are `phone`, `tablet` and `2in1`, plus the
standard `EntryAbility` / `EntryBackupAbility` pair.

`EntryAbility` publishes four `AppStorage` keys inside the `loadContent`
callback: `windowStage`, `mainWindowId`, and the two avoid-area heights
`topRectHeight` / `bottomRectHeight` **in px**. `HomePage` consumes the two
heights with `@StorageProp` and converts at the point of use:

```typescript
.padding({
  top: this.getUIContext().px2vp(this.topRectHeight),
  bottom: this.getUIContext().px2vp(this.bottomRectHeight)
});
```

Unlike most samples in this industry, this one registers **no**
`avoidAreaChange` listener at all - the heights are read once at startup, so a
rotation or a fold/unfold leaves the padding stale. Fewer leaks, less
correctness.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Only the first bottom tab has content; `bottomNavigation` explicitly ignores
  taps on the other three (`if (button.index === 0)`), and `Tabs` is
  `.scrollable(false)`, so the workbench, mail and contacts tabs are
  unreachable by design.
- Nothing persists. `SESSION_DATA` is re-seeded on every `aboutToAppear`,
  which also means pinning, folding and deletions are lost on a tab rebuild.
- `Message.id` is `util.format('%s-%s', senderId, receiverId)` - the same id
  for every message between the same two people. It is never used as a key
  here, but it is not an identity.
- Neither `ForEach` in `MyChat` supplies a key generator, so ArkUI falls back
  to the default. With eight static sessions this is survivable; a real session
  list needs `(item: SessionItem) => item.receiver`.
- The fold header is shown from **one** pinned session (`size >= 1`); a real
  app usually applies a threshold so a single pin does not gain a fold control.
- `Status.SENT_READ` and `Status.DELIVERED` are declared and never used.

## Pitfalls

- **`HW-14-0079`** (D/low, confirmed): the document's snippet writes
  `.visibility(this.isCollapsed ? Visibility.Visible : Visibility.None)` on the
  pinned list, while the shipped code has `!this.isCollapsed`. Copying the
  document inverts the whole feature - the pinned chats vanish when the header
  says 折叠置顶聊天 and reappear when it offers to unfold them. Fix: correct
  the document's snippet to match `MyChat.ets`.
- **`HW-14-0080`** (B/low, confirmed): deleting a session splices
  `sessionList` but never clears the contact's message timer, so
  `timerOne`..`timerFour` keep pushing `Message` objects into `chatMessage` for
  a contact that no longer exists, and the map, `lastMessage` and `reminders`
  entries are never removed. Unbounded growth plus stale bookkeeping. Fix:
  `clearInterval` the contact's timer on delete (a `Map<string, number>` of
  timers makes this trivial) and drop its map entries.
- **`HW-14-0028`** (B/medium, confirmed, systematic - `ChatFoldTop` is the
  second instance after `ChatUnreadReminder`, which shares this `MyChat`
  template): timer-generated incoming messages are pushed into `chatMessage`
  only and never into the `@StorageLink('message')` array, which is the store
  `ChatPage` watches for autoscroll and walks when marking messages read. A
  message arriving while the conversation is open does not scroll into view,
  and the read-marking pass over `data` is unreachable. Fix: push incoming
  messages into both stores, or collapse to one.

## References

- `huawei_industry_tree/14_social_communication/docs/40_chat_fold_top.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_fold_top-0000002405205461
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `divider`, `scrollBar`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-container-listitem.md` - `swipeAction` and its `end` builder
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-listitem
- `documentation/harmonyos-guides/03_application-framework/arkts-state.md` - `@State`, and why a `Set` mutation needs the assignment to be observed
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-state
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - key generators for list items
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `SOCIAL-13` - `ChatUnreadReminder`, the sibling sample sharing this `MyChat` template and the same dual-store defect
- `SOCIAL-37` - the message-level interactions that belong inside the chat page this list opens
