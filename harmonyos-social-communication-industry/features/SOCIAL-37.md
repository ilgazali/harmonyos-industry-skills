---
id: SOCIAL-37
title: Emoji reactions on a message - long-press selection menu, ComponentContent dialogs and an emitter bus
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/37_emoji_comment.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/emoji_comment-0000002351840732
sample: huawei_industry_tree/14_social_communication/downloads/EmojiComment.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [emitter, hilog, promptAction, window, bindSelectionMenu, selection, ComponentContent, wrapBuilder, openCustomDialog, closeCustomDialog, openBindSheet, "@ObjectLink", "@StorageLink", onAreaChange]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-14-0077, HW-14-0087]
status: verified-with-fixes
---

## When to use

**Load this card when a message bubble needs reactions** - the pattern where a
long press on a chat message raises an emoji palette, the chosen emoji lands
as a counted chip under the bubble, and tapping the chip shows who reacted.
It is one of the highest-traffic interactions in a messaging app and it is
harder than it looks, because three separate popups have to appear anchored to
a message that lives inside a scrolling list.

The transferable part is not the emoji list; it is **how the popups are built
and how they talk back**. The sample builds all three popups as global
`@Builder` functions wrapped in `ComponentContent` nodes, opens them through
`UIContext.getPromptAction().openCustomDialog()` / `UIContext.openBindSheet()`,
and routes every click from inside a popup back to the page over `emitter`.
That shape applies to any content that must float above the page and be
positioned per item: a message action bar, a like-and-share strip, a
per-row context menu.

**Read `HW-14-0077` before adopting it.** The emitter bus is process-global,
and the sample subscribes in `aboutToAppear` without ever unsubscribing, so a
rebuilt page double-counts every reaction. The bus is a good idea; the
lifecycle around it is missing.

## Feature checklist

- Long-pressing a message selects its whole text and raises a custom action
  bar over the bubble, with an arrow pointing at the sender's side.
- Tapping the leftmost 表情 (emoji) action opens the emoji palette dialog,
  which shows a 最近使用 (recent) row and a 6-column grid of all emoji.
- Picking an emoji closes the dialog and appends a reaction chip under the
  message bubble.
- Repeating an emoji already used on that message increments its count instead
  of adding a second chip.
- The same user reacting with the same emoji twice raises a toast and changes
  nothing.
- Reaction chips align to the outer edge of the bubble - right for your own
  messages, left for the other party's.
- Once a message has at least one reaction, a small "+" chip appears next to
  the reactions and re-opens the palette.
- Tapping a reaction chip opens a half-modal sheet listing the reactions and,
  for the selected one, the users who sent it.
- Tapping the contact name in the title bar switches the acting user, so a
  single device can produce a two-person reaction count.
- Tapping the page background dismisses the action bar and clears the text
  selection.

## Architecture

One `entry` module, eight source files. Every popup is a global builder; no
popup is a page-local `@CustomDialog`.

```
entry/src/main/ets
├── common/Constants.ets              colors, user names, emitter event ids, polygon points
├── components/
│   ├── MessageItem.ets               one bubble: avatar, tail, text, reaction grid
│   └── PromptBuilder.ets             the three global @Builder popups
├── entryability/EntryAbility.ets     loads ChatPage, publishes the UIContext to AppStorage
├── entrybackupability/
├── model/
│   ├── MenuModel.ets                 ICON_LIST - the five action-bar entries
│   ├── MessageModel.ets              @Observed MessageModel + TextState + 3 seed messages
│   └── PromptModel.ets               popup param classes, EmojiCommented, the 24 emoji
├── pages/ChatPage.ets                @Entry: owns emojiMap, the ComponentContent nodes, the emitter subscriptions
└── utils/PromptActionUtil.ets        static open/close/update wrappers + the three emit helpers
```

The documented 工程目录 matches the zip file for file.

**The design decision worth copying** is `ComponentContent` plus a global
`@Builder`, instead of `@CustomDialog`. A `@CustomDialog` is a child of the
component that declares it, which means the emoji palette would have to be
declared inside `MessageItem` and would exist once per message. Here it is
built once in `ChatPage`:

```typescript
dialogContentNode: ComponentContent<Object> =
  new ComponentContent(this.ctx, wrapBuilder(buildEmojiDialog), this.emojiParams);
```

and handed to a static holder (`PromptActionUtil`) that any message can open,
after first pushing that message's screen position into the dialog options.
One node, N anchors. The price is that the popup content is no longer inside
the component tree, so it cannot use `@Link` to talk back - hence the emitter
bus, and hence `PromptActionUtil` keeping `ctx`, the three nodes and their
options as statics.

The counterpart decision is `emojiMap: Map<number, EmojiCommented[]>` keyed by
message id, held by the page and passed down as `@Prop @Watch('onChange')`.
Reactions are page state, not bubble state, because the popup that produces
them is page-owned.

## Implementation steps

1. **Seed one `TextState` per message** in `aboutToAppear` and one empty
   reaction array per message id. `TextState` is `@Observed` so that writing
   `start`/`end` from a popup callback re-renders only that bubble.
2. **Create the three `ComponentContent` nodes once**, from `wrapBuilder(...)`
   over the global builders, and register them plus the current `UIContext`
   into `PromptActionUtil`.
3. **Subscribe to the three emitter events** for dialog pick, chip tap and
   sheet-item tap - and **unsubscribe in `aboutToDisappear`** (`HW-14-0077`).
4. **Enable long-press selection on the bubble text** with
   `copyOption(CopyOptions.InApp)`, `selection(start, end)` and
   `bindSelectionMenu(TextSpanType.DEFAULT, builder, TextResponseType.LONG_PRESS, ...)`.
   The bound builder is deliberately **empty** - the visible menu is the custom
   dialog opened from `onAppear`.
5. **Capture the bubble geometry** with two `onAreaChange` handlers: the inner
   `Text` yields `textWidth` (used to place the action bar's arrow), the outer
   `Column` yields `globalPosition.y` (used as the dialog offset).
6. **Set the dialog offset before opening**, not inside the builder: the popup
   is a single shared node, so its position is part of the options, updated
   per message.
7. **Publish the tapped emoji over emitter**, reading the focused message id
   from `AppStorage.get('focus')` - the palette itself does not know which
   message it was opened for.
8. **Merge, do not append**, when adding a reaction: same emoji + same sender
   is a toast; same emoji + new sender increments `total`; new emoji pushes a
   chip.
9. **Mirror the reaction grid** with `layoutDirection(GridDirection.RowReverse)`
   for your own messages so chips hug the same edge as the bubble.

## Verified snippets

All snippets are from `EmojiComment.zip`. Corrected forms are marked.

**Wiring the popups and the bus — `entry/src/main/ets/pages/ChatPage.ets`**
(corrected, see `HW-14-0077`)

```typescript
@Entry
@Component
struct ChatPage {
  @State messageModels: MessageModel[] = INITIAL_MESSAGE_LIST;
  @State textStates: TextState[] = [];
  @State emojiMap: Map<number, EmojiCommented[]> = new Map;
  @StorageLink('focus') focusMessageId: number = 0;
  @StorageLink('userName') userName: string = Constants.CURRENT_USER;
  ctx: UIContext = this.getUIContext();
  emojiParams: EmojiParams = new EmojiParams(EMOJI_LIST_DATA, INITIAL_RECENT_EMOJI);
  messageParams: MessageParams = new MessageParams([]);
  dialogContentNode: ComponentContent<Object> =
    new ComponentContent(this.ctx, wrapBuilder(buildEmojiDialog), this.emojiParams);
  bindSheetContentNode: ComponentContent<Object> =
    new ComponentContent(this.ctx, wrapBuilder(buildMessageBindSheet), this.messageParams);
  textMenuContentNode: ComponentContent<Object> =
    new ComponentContent(this.ctx, wrapBuilder(buildCustomTextMenu), 0);

  aboutToAppear(): void {
    INITIAL_MESSAGE_LIST.forEach((item: MessageModel) => {
      this.textStates.push(new TextState(item.messageId));
      this.emojiMap.set(item.messageId, []);
    });
    PromptActionUtil.setContext(this.ctx);
    PromptActionUtil.setDialogContentNode(this.dialogContentNode);
    PromptActionUtil.setTextMenuContentNode(this.textMenuContentNode);
    PromptActionUtil.setBindSheetContentNode(this.bindSheetContentNode);
    let eventDialogCloseBtnOnclick: emitter.InnerEvent = {
      eventId: Constants.DIALOG_ON_CLICK_EVENT_ID
    };
    emitter.on(eventDialogCloseBtnOnclick, (eventData: emitter.EventData) => {
      if (eventData.data && eventData.data.value !== null && typeof eventData.data.value !== 'undefined') {
        this.dialogOnClick(eventData.data.value);
      }
    });
    emitter.on(Constants.BIND_SHEET_EVENT_ID, (eventData: emitter.GenericEventData<string>) => {
      if (eventData.data !== null && typeof eventData.data !== 'undefined') {
        let emojis = JSON.parse(eventData.data) as EmojiOnClickEventParam;
        this.emojiOnClick(emojis);
      }
    });
    let eventEmojiOnclick: emitter.InnerEvent = {
      eventId: Constants.EMOJI_ON_CLICK_EVENT_ID
    };
    emitter.on(eventEmojiOnclick, (eventData: emitter.EventData) => {
      if (eventData.data && eventData.data.value !== null && typeof eventData.data.value !== 'undefined') {
        this.bindSheetItemOnClick(eventData.data.value);
      }
    });
  }

  aboutToDisappear(): void {
    // FIX: absent in the sample - emitter subscriptions are process-global
    emitter.off(Constants.DIALOG_ON_CLICK_EVENT_ID);
    emitter.off(Constants.BIND_SHEET_EVENT_ID);
    emitter.off(Constants.EMOJI_ON_CLICK_EVENT_ID);
  }
}
```

**Three things carry the design here.** `ctx: UIContext = this.getUIContext()`
is captured as a field so the same instance reaches `PromptActionUtil`, which
is static and has no component of its own; a popup opened on the wrong
`UIContext` lands in the wrong window. `wrapBuilder` is what lets a *global*
builder function become a node - the builders in `PromptBuilder.ets` are
module-level functions precisely so they are not tied to a component instance.
And the two event id styles are both legal: `DIALOG_ON_CLICK_EVENT_ID` is a
number wrapped in `emitter.InnerEvent`, `BIND_SHEET_EVENT_ID` is the string
`'some'` passed directly, carrying a JSON payload through
`emitter.GenericEventData<string>`.

The `aboutToDisappear` block is the fix. `emitter.on` registers against the
process, not the component: navigate away and back, or relaunch the page, and
a second closure over a dead `this` is added next to the live one, so one tap
runs `addEmojiComment` twice and the reaction count jumps by two.

**Long-press selection with an empty menu builder — `entry/src/main/ets/components/MessageItem.ets`**
(as shipped)

```typescript
Text(this.messageModel?.content)
  .fontSize(16)
  .backgroundColor(Color.White)
  .borderRadius(8)
  .copyOption(CopyOptions.InApp)
  .selection(this.textState.start, this.textState.end)
  .onAreaChange((oldVal, newVal) => {
    this.textWidth = newVal.width as number; // The width of the text.
  })
  .bindSelectionMenu(TextSpanType.DEFAULT, this.CustomMenuBuilder(), TextResponseType.LONG_PRESS, {
    onAppear: () => {
      this.textState.start = 0;
      this.textState.end = this.messageModel.content.length;
      PromptActionUtil.setTextMenuOptions({
        alignment: DialogAlignment.Top,
        offset: { dx: 0, dy: this.globalY - Constants.DISTANCE_ADJUST_MENU_Y },
        maskColor: Color.Transparent,
        onWillDisappear: () => {
          this.textState.start = -1;
          this.textState.end = -1;
        }
      });
      this.textMenuContentNode.update(this.getPositionX());
      PromptActionUtil.setDialogOptions({
        alignment: DialogAlignment.Top,
        offset: { dx: 0, dy: this.globalY + Constants.DISTANCE_ADJUST_DIALOG_Y },
        maskColor: Color.Transparent,
      });
      PromptActionUtil.openTextMenu();
      this.focusMessageId = this.messageModel.messageId;
    },
    onDisappear: () => {
      // Reset the state of text when the dialog disappears.
      this.textState.start = -1;
      this.textState.end = -1;
    }
  });

@Builder
CustomMenuBuilder() {
  Column() {
  };
}
```

**The empty `CustomMenuBuilder` is deliberate, and it is the trick.**
`bindSelectionMenu` is used only as a long-press *detector* that also selects
text; its own menu renders nothing, and the visible action bar is a custom
dialog opened from `onAppear`. That buys full control over placement (the
system selection menu cannot be offset to sit exactly over the bubble) and
lets the same node be re-pointed per message via
`textMenuContentNode.update(this.getPositionX())`.

`selection(0, content.length)` inside `onAppear` is what produces the
"whole message highlighted" look, and both `onWillDisappear` and `onDisappear`
reset it to `-1, -1`; `TextState` is `@Observed` and reached through
`@ObjectLink`, so only the touched bubble repaints. `getPositionX()` returns
`textWidth / 2 - DISTANCE_FROM_AVATAR_TO_CENTER` (negated for your own
messages) - that number is the horizontal offset of the little triangle
`Polygon` under the action bar, which is how the bar appears to point at the
bubble it belongs to.

**Merging a reaction instead of appending — `entry/src/main/ets/pages/ChatPage.ets`**
(as shipped)

```typescript
// Add emojis under the message based on the message ID and the emoji.
addEmojiComment(messageId: number, emojiCommented: EmojiCommented) {
  let emojiList = this.emojiMap.get(messageId) as EmojiCommented[];
  let isExisting = emojiList.some((value: EmojiCommented) =>
  isResourceEqual(value.emojiResource, emojiCommented.emojiResource) &&
  value.senders.includes(emojiCommented.senders[0])
  );
  if (isExisting) {
    this.getUIContext().getPromptAction().showToast({ message: $r('app.string.same_emoji_reminder') });
  } else {
    let sameEmoji =
      emojiList.find((value: EmojiCommented) => isResourceEqual(value.emojiResource, emojiCommented.emojiResource));
    if (sameEmoji !== undefined) {
      sameEmoji.senders.push(emojiCommented.senders[0]);
      sameEmoji.total += 1;
    } else {
      emojiList.push(emojiCommented);
    }
    this.emojiMap.set(messageId, emojiList);
  }
}

function isResourceEqual(a: ResourceStr, b: ResourceStr): boolean {
  if (typeof a === 'object' && typeof b === 'object') {
    return a.id === b.id;
  }
  return a === b;
}
```

**`isResourceEqual` is the load-bearing helper.** An emoji is a `ResourceStr`,
and `$r('app.media.emoji3') === $r('app.media.emoji3')` is false - each `$r`
call produces a fresh object. Comparing `.id` is the only way to tell two
references to the same media resource apart, and the same helper is reused in
`dialogOnClick` to move a picked emoji to the front of the 最近使用 row. If
you copy nothing else from this sample, copy this: resource identity is not
object identity.

Note the three-way branch. Same emoji from the same sender is refused with a
toast rather than silently ignored, which is the right feedback for a
long-press that visibly did something. Same emoji from a new sender mutates
`senders` and `total` in place; new emoji pushes a chip. `this.emojiMap.set(...)`
at the end is the write that `MessageItem`'s `@Prop @Watch('onChange')` sees -
mutating the array alone would not re-render, because ArkUI observes the Map
assignment, not the nested array.

**The reaction strip — `entry/src/main/ets/components/MessageItem.ets`** (as shipped)

```typescript
@Builder
emojiCommentBuilder() {
  Grid() {
    ForEach(this.emojis, (item: EmojiCommented) => {
      GridItem() {
        Row() {
          Image(item.emojiResource).size({ width: 20, height: 20 });
          Text(item.total.toString())
            .fontColor('#0A59F7')
            .margin({ left: 8 });
        }
        .backgroundColor('#1a000000')
        .width(50)
        .height(36)
        .borderRadius(8)
        .onClick(() => {
          emojiItemOnClick(this.emojis);
        });
      };
    });

    if (this.emojis.length > 0) {
      GridItem() {
        Row() {
          Image($r('app.media.emoji')).size({ width: 20, height: 20 });
        };
      }
      .onClick(() => {
        this.focusMessageId = this.messageModel.messageId;
        PromptActionUtil.setDialogOptions({
          alignment: DialogAlignment.Top,
          offset: { dx: 0, dy: this.globalY + Constants.DISTANCE_ADJUST_DIALOG_Y },
          maskColor: Color.Transparent,
        });
        PromptActionUtil.openDialog();
      });
    }
  }
  .width('100%')
  .padding({ left: 72, right: 72 })
  .columnsGap(9)
  .rowsGap(4)
  .layoutDirection(this.messageModel.sender === Constants.CURRENT_USER ? GridDirection.RowReverse :
    GridDirection.Row);
}
```

`layoutDirection` doing the mirroring is why there is only one builder for
both sides of the conversation: `GridDirection.RowReverse` fills right-to-left,
so your own reactions stack outward from the right edge under your own bubble
and the "+" chip ends up on the inner side. The symmetric 72vp horizontal
padding keeps the strip inside the bubble column on both sides.

The `if (this.emojis.length > 0)` guard is what makes the "+" chip a
*secondary* affordance: with no reactions there is only one route in (long
press), once a reaction exists there are two.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. Device types are
`phone`, `tablet` and `2in1`; there is a single `EntryAbility` and the standard
`EntryBackupAbility`.

`EntryAbility` is minimal by this industry's standards: it forces
`COLOR_MODE_LIGHT`, loads `pages/ChatPage`, and publishes exactly one
`AppStorage` key inside the `loadContent` callback -

```typescript
AppStorage.setOrCreate('uiContext', windowStage.getMainWindowSync().getUIContext());
```

There is no `setWindowLayoutFullScreen` and no avoid-area plumbing here; the
page handles system insets itself with `.expandSafeArea([SafeAreaType.SYSTEM])`
on its root `Column`. That `'uiContext'` key exists for exactly one consumer:
`buildCustomTextMenu` is a global builder with no `this`, so the only way it
can raise a toast from the four unimplemented actions is
`AppStorage.get('uiContext') as UIContext`.

Two further `AppStorage` keys are written by the page, not the ability:
`'focus'` (the message id a popup was opened for, via `@StorageLink`) and
`'userName'` (the acting user). `PromptBuilder.ets` reads both with
`AppStorage.get(...)`, for the same reason - a global builder cannot reach the
page through the component tree, so `AppStorage` is the downward channel and
`emitter` is the upward one.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The conversation is three hardcoded `MessageModel`s in `MessageModel.ets`.
  There is no send path: the bottom input row is a `Text` placeholder and three
  static icons.
- `ForEach(this.messageModels, ...)` has no key generator, and `textStates` is
  indexed by `messageModel.messageId` - both are safe only because the message
  ids are 0..2 and the list never changes. A real chat needs a stable id key.
- Only the first action-bar entry (表情) works; the other four raise a
  "not implemented" toast through the `AppStorage`-held `UIContext`.
- `MessageItem` constructs its own `textMenuContentNode` default *and* receives
  the page's - the field initializer runs before the parent's value is applied,
  so one extra `ComponentContent` is built per message and discarded.
- The reaction sheet lists senders with a single hardcoded avatar
  (`$r('app.media.mine')`) regardless of who reacted.

## Pitfalls

- **`HW-14-0077`** (B/low, confirmed): three `emitter.on` subscriptions are
  registered in `aboutToAppear` and never removed. Emitter subscriptions are
  process-global, so a page rebuild (back-navigation, relaunch) leaves the
  stale closures alive next to the new ones and every emoji pick is processed
  twice - the reaction count jumps by two, or the "same sender" toast fires on
  a first tap. Fix: `emitter.off` for all three ids in `aboutToDisappear`.

## References

- `huawei_industry_tree/14_social_communication/docs/37_emoji_comment.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/emoji_comment-0000002351840732
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-text.md` - `bindSelectionMenu`, `selection`, `copyOption`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-text
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-promptaction.md` - `openCustomDialog`, `closeCustomDialog`, `updateCustomDialog`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-promptaction
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `openBindSheet`, `updateBindSheet`, `closeBindSheet`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-references/02_application-framework/js-apis-arkui-componentcontent.md` - `ComponentContent` and `update`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-arkui-componentcontent
- `documentation/harmonyos-guides/03_application-framework/arkts-wrapbuilder.md` - turning a global `@Builder` into a node
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-wrapbuilder
- `documentation/harmonyos-references/03_system/js-apis-emitter.md` - `emitter.on`, `emitter.off`, `emit`, `GenericEventData`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-emitter
- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` - `layoutDirection` and `GridDirection.RowReverse`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `SOCIAL-40` - the session list this chat page would sit behind
