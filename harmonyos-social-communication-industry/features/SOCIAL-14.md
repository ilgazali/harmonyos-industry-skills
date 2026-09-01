---
id: SOCIAL-14
title: Quoting a chat message - a custom selection menu on Text, plus scrollToIndex to jump back to the original
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/14_chat_reference.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_reference-0000002317503353
sample: huawei_industry_tree/14_social_communication/downloads/ChatReference.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [hilog, window]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0031, HW-14-0032, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card when a message in your app needs to **answer another message**:
long-press a bubble, pick 引用 (quote) from the selection menu, see the quoted
text sitting above the composer, send, and then tap the quote inside the sent
bubble to scroll back to what was quoted. WeChat, Telegram, Slack and Teams all
ship a version of this.

Two mechanisms carry it, and both are reusable on their own. The first is
`bindSelectionMenu` on a `Text` with `copyOption(CopyOptions.InApp)` - that is
how you add an app-specific action to the system's text-selection popup without
building your own long-press gesture. The second is
`ListScroller.scrollToIndex`, which turns "jump to the referenced item" into
one call, provided your references are list indices.

The transferable design idea is smaller than the feature: **a message carries an
integer pointing at another message, and one sentinel value means "none".**
The same shape covers replies in a comment thread, a linked ticket, a cited
paragraph. Everything else here - the chip above the composer, the quote strip
inside the bubble, the scroll - is rendering of that one number.

## Feature checklist

- A conversation view with left/right bubbles, avatars, and a date header.
- Long-pressing a bubble selects text and opens a custom five-icon menu
  (引用 / 复制 / 全选 / 翻译 / 收藏).
- Tapping 引用 stores that message's index as the pending reference; the other
  four icons raise a "demo only" toast.
- The pending reference appears as a grey chip above the input, showing
  `sender: content` truncated to one line, with a cross to clear it.
- Sending attaches the reference to the new message; the sent bubble renders
  the quoted strip underneath its own content.
- Tapping the quoted strip scrolls the list to the original message.
- Quotes preserve inline emoji: an image span inside the quoted message renders
  as a 20vp `ImageSpan` inside the quote.
- The composer is a `RichEditor` with an emoji grid, a send button that appears
  only when there is content, and a delete key with long-press repeat.
- The list scrolls to the bottom when a message is added and when the keyboard
  opens.

## Architecture

One `entry` module, seven ArkTS files.

```
entry/src/main/ets
├── common/
│   ├── CommonConstants.ets   NO_REFERENCE = -1, the two display names
│   └── ListData.ets          one message bubble: menu, quote strip, jump (183 lines)
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── model/Interface.ets       MessageType, MsgTextImage, MsgContent, MenuItem, EditMenuAction
├── pages/ChatPage.ets        @Entry: the list, the shared @Provide state, the header
└── view/
    ├── CustomRichEditor.ets  the composer, the quote chip, sendMessage()
    └── EmojiView.ets         the emoji grid and the repeat-delete key
```

The documented tree matches the zip - unlike the four samples covered by
`HW-14-0001`, whose 工程目录 blocks list files their zips do not contain.

**The design decision worth copying** is `referenceIndex` plus the
`NO_REFERENCE` sentinel. A message is:

```typescript
export interface MsgContent {
  isSelf: boolean;
  referenceIndex: number;      // -1 when the message quotes nothing
  userName: ResourceStr;
  content: MsgTextImage[];
}
```

There is no nested quoted-message object, no duplication of the quoted text,
and therefore no way for a quote to drift out of date. `ChatPage` owns
`data: MsgContent[]` and one `referenceEditor: number` holding the pending
reference; both are `@Provide`d. `ListData` receives the array by value
parameter so it can dereference `data[item.referenceIndex]` when it renders a
quote, and `referenceEditor` by `@Link` so tapping 引用 writes straight back up
to the composer.

The price of an index is that it is positional: delete a message and every
reference after it points at the wrong bubble. For a demo with an append-only
array that is fine. For a real client, store a message id and resolve it to an
index at scroll time.

`content` being `MsgTextImage[]` rather than a string is the second good call -
it is what lets one `Text` hold interleaved `Span`s and `ImageSpan`s, so an
emoji is part of the message rather than a separate message type, and the quote
renderer replays the same array with the same code.

## Implementation steps

1. **Model a message as an array of typed fragments** (`TEXT` / `IMAGE`) so
   inline emoji and text share one bubble and one renderer.
2. **Give every message a `referenceIndex`,** defaulted to a module-level
   `NO_REFERENCE = -1`. Never use `undefined` here - `-1` keeps the field a
   plain `number` and keeps the ArkTS strictness checks happy.
3. **Enable in-app copy on the bubble** with `.copyOption(CopyOptions.InApp)`;
   without it the text is not selectable and no selection menu can appear.
4. **Attach the custom menu**:
   `.bindSelectionMenu(TextSpanType.DEFAULT, this.CustomMenu, TextResponseType.DEFAULT)`.
   `TextSpanType.DEFAULT` covers plain text spans, `TextResponseType.DEFAULT`
   means long-press.
5. **Build the menu as a `@Builder` row of icon+label cells** driven by a
   `MenuItem[]`, and branch on the cell index inside one shared `onClick`.
6. **On 引用, write the message's own index into the shared
   `referenceEditor`** - the bubble knows its index because the parent passed
   it in.
7. **Dismiss the menu through the controller of the component that owns it**
   (`HW-14-0032`) - a freshly constructed `RichEditorController` that was never
   passed to any component controls nothing.
8. **Render the pending reference above the input** when
   `referenceEditor !== NO_REFERENCE`, and grow the composer's height to match
   so the chip does not overlap the text field.
9. **Copy the pending reference into the message on send, then reset it** to
   `NO_REFERENCE`.
10. **Toggle the speaker after sending, not before** (`HW-14-0031`) - otherwise
    the very first message is attributed to the peer.
11. **Render the quoted strip inside the bubble** by dereferencing
    `data[item.referenceIndex]` and replaying its fragments through the same
    `ForEach`.
12. **Jump with `this.listScroller.scrollToIndex(this.item.referenceIndex)`,**
    using the *same* `ListScroller` instance the `List` was constructed with -
    pass it down, do not create a second one.
13. **Autoscroll on change** with `@Watch` on the message array, and again on
    `onEditingChange` after a ~100ms delay so the scroll happens once the
    keyboard has finished rising.

## Verified snippets

All snippets are from `ChatReference.zip`. Corrected forms are marked.

**The custom selection menu — `entry/src/main/ets/common/ListData.ets`**
(corrected, see `HW-14-0032`)

```typescript
@Component
export struct ListData {
  @State item: MsgContent = { isSelf: false, referenceIndex: NO_REFERENCE, userName: '', content: [] };
  @Link referenceEditor: number;
  index: number = 0;
  @State listScroller: ListScroller = new ListScroller();
  @State data: MsgContent[] = [];
  controller: TextController = new TextController();      // FIX: sample uses a stray RichEditorController
  private iconArr: MenuItem[] =
    [{ icon: $r('app.media.ic_reference_name'), name: $r('app.string.Reference_Icon_Name') },
      { icon: $r('app.media.ic_copy'), name: $r('app.string.copy_icon_name') },
      { icon: $r('app.media.ic_select_all'), name: $r('app.string.select_all_name') },
      { icon: $r('app.media.ic_translate'), name: $r('app.string.translate_icon_name') },
      { icon: $r('app.media.ic_collection'), name: $r('app.string.collection_icon_name') },];

  @Builder
  CustomMenu() {
    Column() {
      Row({ space: 2 }) {
        ForEach(this.iconArr, (item: MenuItem, index ?: number) => {
          Flex({ direction: FlexDirection.Column, justifyContent: FlexAlign.SpaceAround,
                 alignItems: ItemAlign.Center }) {
            Image(item.icon).width(20).height(20).draggable(false);
            Text(item.name).fontSize(10);
          }
          .width(40)
          .height(40)
          .onClick(() => {
            if (index as number === 0) {
              this.referenceEditor = this.index;          // 引用: publish this bubble's index
            } else {
              this.getUIContext().getPromptAction().showToast({
                message: $r('app.string.video_linkage_list_function_toast')
              });
            }
            this.controller.closeSelectionMenu();
          });
        });
      };
    }
    .height(64)
    .backgroundColor(Color.White)
    .borderRadius(16);
  }
```

**One handler, five icons, branch on the index.** The menu is data-driven from
`iconArr`, so adding a sixth action is one array entry plus one branch. Only
index 0 does anything real; the rest raise a toast, which is honest about the
sample's scope.

`closeSelectionMenu()` is the line to get right. In the shipped code
`controller` is `new RichEditorController()` that is never handed to any
`RichEditor` in this file - it is a controller attached to nothing, so the call
is a no-op and the menu stays on screen after 引用 is tapped. The menu here
belongs to a `Text`, so the controller must be a `TextController` bound to that
same `Text` via `Text(undefined, { controller: this.controller })`. The general
rule: a controller only controls the component instance it was passed to.

**The bubble and the quote strip — same file** (as shipped)

```typescript
Column() {
  Text() {
    ForEach(this.item.content, (content: MsgTextImage) => {
      if (content.type === MessageType.TEXT) {
        Span(content.content);
      }
      if (content.type === MessageType.IMAGE) {
        ImageSpan(content.content)
          .clip(true)
          .objectFit(ImageFit.Contain)
          .size({ width: '20vp', height: '20vp' })
          .verticalAlign(ImageSpanAlignment.CENTER);
      }
    }, (item: MsgTextImage, index: number) => `${JSON.stringify(item)}_${JSON.stringify(index)}`);
  }
  .backgroundColor(Color.White)
  .borderRadius({
    bottomLeft: 16,
    bottomRight: 16,
    topLeft: this.item.isSelf ? 16 : 0,
    topRight: this.item.isSelf ? 0 : 16
  })
  .constraintSize({ maxWidth: '70%' })
  .copyOption(CopyOptions.InApp)
  .bindSelectionMenu(TextSpanType.DEFAULT, this.CustomMenu, TextResponseType.DEFAULT);

  if (this.item.referenceIndex !== NO_REFERENCE) {
    Row() {
      Text() {
        Span(this.data[this.item.referenceIndex].userName + ':')
          .fontSize(16);
        ForEach(this.data[this.item.referenceIndex].content, (content: MsgTextImage) => {
          if (content.type === MessageType.TEXT) {
            Span(content.content);
          }
          if (content.type === MessageType.IMAGE) {
            ImageSpan(content.content).size({ width: '20vp', height: '20vp' });
          }
        }, (item: MsgTextImage, index: number) => `${JSON.stringify(item)}_${JSON.stringify(index)}`);
      }
      .textOverflow({ overflow: TextOverflow.Ellipsis })
      .maxLines(1);
    }
    .constraintSize({ maxWidth: '50%' })
    .backgroundColor('rgba(0, 0, 0, 0.05)')
    .onClick(() => {
      this.listScroller.scrollToIndex(this.item.referenceIndex);
    })
  }
}
.alignItems(this.item.isSelf ? HorizontalAlign.End : HorizontalAlign.Start);
```

**Three details carry this layout.** `copyOption(CopyOptions.InApp)` is a
prerequisite, not decoration: `bindSelectionMenu` never fires on a `Text` that
cannot be selected. The asymmetric `borderRadius` - one corner square on the
side facing the avatar - is what makes a rounded rectangle read as a speech
bubble, and it flips with `isSelf` so the same builder draws both sides.
`constraintSize({ maxWidth: '70%' })` on the bubble and `'50%'` on the quote,
with `maxLines(1)` and an ellipsis on the quote, keep a long quoted message
from dwarfing the reply that quotes it.

The quote block is the *same* `ForEach` as the bubble body, run over
`data[referenceIndex].content`. That is the payoff of storing fragments rather
than a rendered string: emoji survive being quoted, and there is one renderer
to maintain.

`scrollToIndex` needs the `ListScroller` that the `List` in `ChatPage` was
constructed with, which is why `ChatPage` passes `listScroller: this.listScroller`
down into every `ListData`. A locally constructed scroller would compile and
silently do nothing.

**Send — `entry/src/main/ets/view/CustomRichEditor.ets`** (corrected, see `HW-14-0031`)

```typescript
private sendMessage() {
  let message: MsgTextImage[] = [];
  this.richController?.getSpans().forEach(span => {
    if ((span as RichEditorTextSpanResult).textStyle !== undefined) {
      message.push({ type: MessageType.TEXT, content: (span as RichEditorTextSpanResult).value });
    } else {
      message.push({ type: MessageType.IMAGE, content: (span as RichEditorImageSpanResult).valueResourceStr });
    }
  });
  if (message.length > 0) {
    this.data.push({
      isSelf: this.isSelf,
      referenceIndex: this.referenceEditor,
      userName: this.isSelf ? AVATAR_NAME02 : AVATAR_NAME01,
      content: message
    });
  }
  this.richController?.deleteSpans();
}

// the send button
Button($r('app.string.Send_Message'), { type: ButtonType.Normal })
  .onClick(() => {
    this.sendMessage();
    this.isSelf = !this.isSelf;      // FIX: the sample toggles BEFORE sendMessage()
    this.referenceEditor = NO_REFERENCE;
  });
```

**`getSpans()` is the whole parser.** A `RichEditor`'s content comes back as a
span list where text spans carry a `textStyle` and image spans do not, so the
`textStyle !== undefined` test is the discriminator between
`RichEditorTextSpanResult` and `RichEditorImageSpanResult`. Emoji inserted by
`EmojiView.addImageSpan` come out of the same call as image fragments with a
`valueResourceStr`, so no separate bookkeeping is needed. `deleteSpans()` with
no arguments clears the editor.

The `message.length > 0` guard means an empty editor cannot produce an empty
bubble - compare `SOCIAL-16`, which has no such guard (`HW-14-0034`).

`isSelf` alternates the speaker so one person can demo a two-sided chat. In the
shipped code the toggle runs **before** `sendMessage()`, and `isSelf` starts
`true`, so the first message is written with `isSelf: false` - it lands on the
left, with the peer's avatar and the peer's name (`AVATAR_NAME01`, 大侠).
Moving the toggle after the send fixes both the side and the attribution, and
`referenceEditor` must be reset after the message has consumed it, not before.

## Permissions & config

**None.** `module.json5` carries an empty `"requestPermissions": []`. No
network, no storage, no contacts - the conversation is in-memory only.

`ChatPage` reads `topRectHeight` / `bottomRectHeight` from `AppStorage` with
`@StorageLink` and converts with `px2vp` in the binding expression, and it sets
`KeyboardAvoidMode.RESIZE` in `aboutToAppear` so the page shrinks rather than
slides when the keyboard opens - required for the list-plus-composer layout,
where the composer must stay pinned above the keyboard.

The composer's bottom padding is `curMenuAction === EditMenuAction.TEXT ? 0 :
px2vp(bottomRectHeight)`: while the soft keyboard is up, the navigation-bar
inset must not be added on top of it.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `referenceIndex` is a positional index into `data`. The sample only appends,
  so it holds; add deletion and every quote after the deleted message points at
  the wrong bubble.
- `data` starts empty - there is no seed conversation. The first thing to do in
  the running app is type a message, and because of `HW-14-0031` it arrives as
  the peer.
- Every `ForEach` in this sample keys on `JSON.stringify(item)`. Two identical
  messages produce the same key, which is a diffing hazard the moment the same
  text is sent twice.
- `ChatPage` defines an `onCreateMenu` callback that would add a 引用 item to
  the *system* text menu, but it is never bound to any component - dead code,
  and a hint that the sample was migrated from `editMenuOptions` to
  `bindSelectionMenu`.
- The header's back button has no `onClick`; there is only one page.
- Only two participants exist, `AVATAR_NAME01` (大侠) and `AVATAR_NAME02`
  (求财求财), both hardcoded in `CommonConstants.ets`.
- The emoji grid renders one row of seven empty `GridItem`s
  (`emptyEmojiList`) purely so the floating delete key cannot cover a real
  emoji - a layout hack worth knowing about, not worth copying.

## Pitfalls

- **`HW-14-0031`** (B/low, probable): systematic - `isSelf` is toggled *before*
  `sendMessage()` reads it, so with `isSelf` initialised to `true` the user's
  first typed message renders on the left with the peer's avatar and the peer's
  name. `ChatSendContact`'s `BottomActionBar.ets` has the identical sequence.
  Fix: move `this.isSelf = !this.isSelf` below `this.sendMessage()`, or start
  it `false`.
- **`HW-14-0032`** (B/low, probable): `closeSelectionMenu()` is called on a
  `RichEditorController` constructed in `ListData` and never passed to any
  component, while the menu on screen belongs to a `Text` bound through
  `bindSelectionMenu`. The custom menu therefore stays open after 引用 is
  tapped. Fix: bind a `TextController` to that `Text` and close through it.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-text.md` - `bindSelectionMenu`, `copyOption`, `TextSpanType`, `TextResponseType`, `TextController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-text
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-menu.md` - menu control and custom menu builders
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-menu
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `ListScroller.scrollToIndex`, `scrollEdge`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-container-scroll.md` - `scrollToIndex` semantics and alignment
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scroll
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-richeditor.md` - `RichEditorController.getSpans`, `addImageSpan`, `deleteSpans`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-richeditor
- `huawei_industry_tree/14_social_communication/docs/14_chat_reference.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_reference-0000002317503353
- `SOCIAL-16` - the same `bindSelectionMenu` menu, applied to drag-and-drop
