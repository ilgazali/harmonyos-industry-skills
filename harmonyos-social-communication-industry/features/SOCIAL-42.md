---
id: SOCIAL-42
title: Send a contact card - a Navigation round trip that returns the picked friend through pop(result)
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/42_chat_send_conntact.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_send_conntact-0000002412138797
sample: huawei_industry_tree/14_social_communication/downloads/ChatSendContact.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.IMEKit", "@kit.PerformanceAnalysisKit"]
apis: [hilog, inputMethod, window, NavPathStack, pushPath, pop, onPop, AlphabetIndexer, ListItemGroup, scrollToIndex, onScrollIndex, "@Provide", "@Consume", KeyboardAvoidMode]
permissions: [ohos.permission.INTERNET]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0031, HW-14-0003, HW-14-0087, HW-14-0091, HW-14-0092, HW-14-0093, HW-14-0094]
status: verified-with-fixes
---

## When to use

**Load this card when a chat needs to send something the user must go and pick
first** - a contact card here, but equally a location, a file, a poll or a
coupon. The shape is the same every time: the composer pushes a picker
destination, the picker returns one object through `pop(result)`, and the
`onPop` callback on the *pushing* side turns that object into a new message.

The second half of the card is the picker itself: an A-Z contact book built
from `ListItemGroup` section headers plus an `AlphabetIndexer` rail, wired both
ways - tapping a letter scrolls the list, scrolling the list moves the
highlight.

Adopt the round-trip pattern, but **read `HW-14-0031` before copying the
composer**: `BottomActionBar` toggles the `isSelf` flag before it sends, so the
first message a user types renders as if the peer had sent it. This sample is
the second instance of that defect in the pack, which is a good sign it came
from a shared template rather than from a considered design.

## Feature checklist

- The chat page opens on a conversation with an avatar-flanked bubble list.
- Typing text reveals a 发送 (send) button; sending appends a text bubble and
  clears the input.
- The `+` button opens a four-icon panel (picture, camera, contact card, file);
  only the third is implemented, the rest toast a "demo only" tip.
- Tapping the contact-card icon pushes a friends page titled 选择好友.
- The friends page groups contacts under A/B/C/D headers with an alphabet rail
  on the right edge.
- Tapping a rail letter scrolls the list to that group; scrolling the list
  moves the rail highlight.
- Tapping a contact returns to the chat immediately and appends a card bubble
  showing the avatar, the name and a 个人名片 footer.
- The more-panel closes on return.
- Focusing the input scrolls the list to the bottom and, with the panel open,
  dismisses the IME first.

## Architecture

One `entry` module. A single `@Entry` page hosting a `Navigation`, with both
screens as `NavDestination` components resolved by name.

```
entry/src/main/ets
├── components/
│   ├── BottomActionBar.ets   input row + the more-panel + the pushPath/onPop round trip
│   ├── ChatContent.ets       NavDestination: title, bubble list, text and card builders
│   └── ContactBook.ets       NavDestination: ListItemGroup sections + AlphabetIndexer
├── constants/PageConstant.ets  the two destination names
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── model/
│   ├── ContactBookModel.ets  LetterInfo, ListItemInfo, the A-D seed lists, alphabets
│   └── MoreData.ets          MessageType, MsgContent, the four more-panel entries
└── pages/ChatPage.ets        @Entry: 46 lines, a Navigation and a pageMap builder
```

The documented tree matches the zip.

**The design decision worth copying** is that `ChatPage` is nothing but a
router. It owns the `NavPathStack`, publishes it as `@Provide('pageInfo')`,
pushes the chat destination in `aboutToAppear` so even the first screen is a
`NavDestination`, and maps names to components in a `pageMap` builder. Every
child then reaches the stack with `@Consume('pageInfo')` instead of receiving it
as a parameter through two levels of composition:

```typescript
@Provide('pageInfo') pageInfos: NavPathStack = new NavPathStack();
```

That matters here because the component that *pushes* (`BottomActionBar`) is a
grandchild of the component that renders the result list (`ChatContent`), and
the `onPop` closure it registers writes into `@Consume data`. Provide/Consume
lets the deep child both navigate and mutate the conversation without any
prop-drilling. The cost is the usual one: `@Consume` fails at runtime, not at
compile time, if a host forgets the matching `@Provide`.

## Implementation steps

1. **Make the first screen a destination too** - push `PAGE_CHAT` in
   `aboutToAppear` rather than putting the chat inside the `Navigation` body.
   The picker then pops back to a real stack entry.
2. **Publish the stack once with `@Provide`,** consume it where you navigate.
3. **Push with an `onPop` closure,** not with a shared state flag: the callback
   is scoped to the one navigation that raised it.
4. **Return the whole picked object** with `this.pageInfos.pop(project)` and
   cast it back in `onPop` - the framework carries it untyped.
5. **Close the more-panel inside `onPop`,** so returning from the picker leaves
   the composer in its resting state.
6. **Toggle `isSelf` after `sendMessage()`, not before** (`HW-14-0031`).
7. **Build the picker from `ListItemGroup` with a `header` builder,** one group
   per letter, so `scrollToIndex(groupIndex)` addresses whole sections.
8. **Bind the rail with `selected: $$this.selectedIndex`** and update that same
   field from `onScrollIndex` for the reverse direction.
9. **Delete the unused `INTERNET` permission and the `maps` querySchemes entry**
   (`HW-14-0003`) - neither has any code behind it in this sample.

## Verified snippets

All snippets are from `ChatSendContact.zip`. Corrected forms are marked.

**The round trip, push side — `entry/src/main/ets/components/BottomActionBar.ets`** (as shipped)

```typescript
@Consume data: MsgContent[];
@Consume('pageInfo') pageInfos: NavPathStack;

// inside the more-panel grid, index 2 is the contact-card entry
.onClick(() => {
  if (index === 2) {
    this.pageInfos.pushPath({
      name: PageConstant.PAGE_CONTACT, param: item, onPop: (popInfo: PopInfo) => {
        this.isMoreOpen = false;
        let letterInfo = popInfo.result as LetterInfo;
        this.data.push({
          isSelf: true,
          type: MessageType.CARD,
          text: '',
          name: letterInfo.name,
          image: letterInfo.image,
        });
      }
    });
  } else {
    this.getUIContext().getPromptAction().showToast({ message: $r('app.string.tips') });
  }
})
```

**and the pop side — `entry/src/main/ets/components/ContactBook.ets`** (as shipped)

```typescript
.onClick(() => {
  this.innerIndex = index;        // 更新选中索引
  this.outerIndex = outerIndex;   // 更新选中索引
  this.pageInfos.pop(project);
})
```

**`onPop` is the whole contract.** `pop(project)` hands the entire `LetterInfo`
back, not an id, so the composer needs no lookup table; the cast
`popInfo.result as LetterInfo` is the one place type safety is lost, and it is
worth keeping the pop and the cast in the same commit for that reason. Because
the callback is attached to *this* push, two different picker entries in the
same panel would each get their own handler with no dispatch logic.

Note that the pushed `param: item` is the more-panel entry (icon + label), not
anything the contact book reads - `ContactBook` never touches
`pathInfo.param`. It is a leftover; the meaningful direction of travel here is
the return value.

**The message that renders as the peer — same file** (corrected, see `HW-14-0031`)

```typescript
@State isSelf: boolean = true;

Text($r('app.string.send'))
  .visibility(this.hasText ? Visibility.Visible : Visibility.None)
  .onClick(() => {
    this.sendMessage();
    this.isSelf = !this.isSelf;   // FIX: shipped code toggles BEFORE sendMessage()
  })

private sendMessage() {
  this.data.push({
    isSelf: this.isSelf,
    type: MessageType.TEXT,
    text: this.text,
    name: '',
    image: null,
  });
  this.text = '';
}
```

**The alternation is a demo device, and the ordering makes it wrong.** The
sample flips `isSelf` on every send so a single user can produce a two-sided
conversation. But `sendMessage` reads the field *after* the toggle, so with the
initial value `true` the first message is pushed with `isSelf === false` - it
appears on the left, with the peer's avatar and the peer's bubble tail. Moving
the toggle below the call restores the intended "my first message is mine".
The contradiction is visible inside the sample itself: contact cards are always
pushed with `isSelf: true` and never alternate.

`ChatContent` consumes that flag three times - the flex direction, the avatar
resource and which of the two triangle images is visible - so one boolean
mirrors the entire bubble:

```typescript
Flex({
  direction: item.isSelf ? FlexDirection.RowReverse : FlexDirection.Row,
  space: { main: LengthMetrics.vp(8) }
}) {
  Image(item.isSelf ? $r('app.media.avatar_right') : $r('app.media.avatar_left'))
    .width($r('app.float.vp_40'))
    .aspectRatio(1)
  ...
}
```

**The indexed contact book — `entry/src/main/ets/components/ContactBook.ets`** (as shipped)

```typescript
List({ scroller: this.listScroller }) {
  ForEach(this.listItemArr, (item: ListItemInfo, outerIndex) => {
    ListItemGroup({ header: this.itemHead(item.alphabet) }) {
      ForEach(item.letterInfo, (project: LetterInfo, index: number) => {
        ListItem() { /* avatar + name row */ }
      })
    }
  })
}
.onScrollIndex((firstIndex: number) => {
  this.selectedIndex = ListItemModel.getListItem()[firstIndex].index;
})

AlphabetIndexer({ arrayValue: alphabets, selected: $$this.selectedIndex })
  .itemSize(17)
  .onSelect((index) => {
    let point: number | null = this.findAlphabetIndex(index);   // rail letter -> group index
    if (point !== null) {
      this.listScroller.scrollToIndex(point, true);             // scroll to the group
    }
  })

findAlphabetIndex(index: number): number | null {
  let selectedLetter: string = alphabets[index];
  for (let i = 0; i < ListItemModel.getListItem().length; i++) {
    if (ListItemModel.getListItem()[i].alphabet === selectedLetter) {
      return i;
    }
  }
  return null;   // letters with no contacts are simply ignored
}
```

**Two indices, deliberately not the same number.** `AlphabetIndexer` indexes
the 26-letter rail; the `List` indexes four groups. `findAlphabetIndex`
translates one to the other and returns `null` for a letter nobody's name
starts with, which is what stops a tap on "Q" from scrolling anywhere. The
reverse direction rides on `onScrollIndex`, whose `firstIndex` is the group
index because the top-level `ForEach` produces `ListItemGroup`s - so
`ListItemModel.getListItem()[firstIndex].index` is just the group's own
position, and assigning it to the two-way-bound `$$this.selectedIndex` moves
the rail highlight.

`$$` is the reason the rail stays in sync: the indexer writes the user's tap
back into `selectedIndex`, and `onScrollIndex` writes scroll-driven changes into
the same field, with no third source of truth.

## Permissions & config

```json5
"querySchemes": [
  "maps"
],
"requestPermissions": [
  {
    "name": "ohos.permission.INTERNET"
  }
]
```

**Both entries are dead** (`HW-14-0003`). Nothing in the sample opens a socket
- the contact list is four hardcoded arrays in `ContactBookModel.ets` - and
nothing calls `canOpenLink` or starts a map. The `maps` scheme and the card
type named 位置卡片 in `ChatContent`'s comments are fossils of the
location-card sample this project was cloned from (`SOCIAL-26`). Delete both;
the feature builds and runs without them.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `deviceTypes` is `phone` only.
- The contact book is 16 seeded friends across four letters. There is no
  search (the search row is a static `Row`, not a `Search`), no persistence and
  no real contacts-app integration.
- `ListItemModel.getListItem()` rebuilds the whole list, including 16
  `LetterInfo` objects, on every call - and `findAlphabetIndex` calls it once
  per loop iteration. Harmless at this size, wrong at any real one: hoist it
  into the `listItemArr` field the component already has.
- `ChatContent.onBackPressed` returns `true` unconditionally, so the system
  back gesture is swallowed on the chat screen; the picker is popped only by
  choosing a contact.
- The card bubble is a fixed 221×99 vp `Stack`. Long names ellipsise; they do
  not wrap.
- Only the contact-card entry of the more-panel is implemented. Picture,
  camera and file all raise the same `$r('app.string.tips')` toast.

## Pitfalls

- **`HW-14-0031`** (B/low, probable): `isSelf` starts `true` and is toggled
  *before* `sendMessage()` reads it, so the first message the user types is
  rendered as an incoming message with the peer's avatar and bubble tail. The
  same code appears in `ChatReference` (`SOCIAL-14`), which is where the finding
  was raised - this sample is the second instance. Fix: move the toggle below
  `sendMessage()`, or start the flag at `false`.
- **`HW-14-0003`** (D/low, confirmed): copy-pasted module config leaves
  `ohos.permission.INTERNET` declared with no networking code and a `maps`
  entry in `querySchemes` with nothing that queries it. Same template defect as
  `ChatMultiSelection` (`SOCIAL-33`), where it was raised. Fix: delete both
  declarations.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `NavPathStack`, `pushPath`, `pop(result)`, `PopInfo`, `navDestination`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-references/02_application-framework/ts-container-alphabet-indexer.md` - `AlphabetIndexer`, `arrayValue`, `selected`, `onSelect`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-alphabet-indexer
- `documentation/harmonyos-references/02_application-framework/ts-state-management.md` - `@Provide` / `@Consume`, `@Watch`, `$$` two-way binding
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-state-management
- `SOCIAL-14` - the chat composer this one was cloned from, and the origin of `HW-14-0031`
- `SOCIAL-26` - the location-card flow whose `maps` config survives here as dead config
- `SOCIAL-33` - the origin of `HW-14-0003`, the same copy-pasted permission block
