---
id: SOCIAL-36
title: Quick-reply panel - LazyForEach phrase categories plus an editable common-phrase list
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/36_quick_reply_in_chat.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/quick_reply_in_chat-0000002385043665
sample: huawei_industry_tree/14_social_communication/downloads/QuickReplyInChat.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [LazyForEach, IDataSource, DataChangeListener, Tabs, TabContent, tabBar, onChange, onAnimationStart, TabsController, CustomDialogController, RichEditorController, addTextSpan, deleteSpans, "resourceManager.getRawFileContentSync", "util.TextDecoder", "@Provide", "@Consume", "@Link"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-14-0076, HW-14-0001, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card when the input box needs a **panel of canned phrases**: a drawer
that slides in under the editor, one side holding the user's own frequently used
lines (add, delete), the other holding categorised presets loaded from a bundled
file. Chat is the obvious host; the same panel serves canned responses in a
support console, snippet insertion in a note app, or template replies in mail.

Two mechanisms carry the feature. The presets are rendered with `LazyForEach`
over a custom `IDataSource`, so a category list of any length costs only the
tabs currently on screen. The user's own phrases are a plain observed array
manipulated with `push` and `splice`, because that list is short and fully
resident by definition. **Pick the right one per list rather than one for both** -
that split is the transferable lesson here.

The third piece worth taking is the insertion contract: tapping a phrase does
not send it, it writes into the `RichEditor` through the controller, and tapping
a second phrase *replaces* the first. Read `HW-14-0076` before adopting the tab
strip: the sample's category tabs mis-handle their first entry.

## Feature checklist

- A chat page with an expandable input box; the ⊕ button opens a four-icon
  expansion row.
- The second icon in that row opens the quick-reply panel in place of the
  expansion row.
- The panel has a two-position pill switch: 常用语 (common phrases) and 快捷回复
  (quick replies).
- 常用语 lists the user's own phrases, each with a delete cross, plus a
  新增常用语 row that opens a text-area dialog and appends the entered line.
- 快捷回复 shows category tabs (毒鸡汤, 文艺, 迟到, 请假, 表白) read from
  `resources/rawfile/reply_phrases.json`; each tab lists its phrases.
- Tapping a phrase inserts it into the editor; tapping another replaces the
  inserted text rather than appending to it.
- Phrases highlight blue while pressed, in both lists.
- Sending clears the editor and appends the message to the chat, alternating the
  sender so the demo can converse with itself.

## Architecture

One `entry` module. The panel is four components deep below the input box.

```
entry/src/main/ets
├── common
│   ├── CommonConstants.ets      two display names
│   ├── CommonPhrasesTabs.ets    the user's own phrase list: push / splice / dialog
│   ├── ListData.ets             one chat bubble
│   └── QuickPhrasesTabs.ets     the preset categories: Tabs + LazyForEach (144 lines)
├── entryability/EntryAbility.ets
├── entrybackupability/
├── model/Interface.ets          EditMenuAction, MsgContent, QuickPhrases
├── pages/ChatPage.ets           @Entry, owns data + curMenuAction + commonReplyData
├── utils
│   ├── CommonDataSource.ets     a generic IDataSource implementation
│   └── DataUtils.ets            getRawFileContentSync -> QuickPhrases[]
└── view
    ├── CustomRichEditor.ets     the input row; mounts ExpandView or QuickReplyView
    ├── ExpandView.ets           the four-icon expansion row
    ├── QuickReplyView.ets       the panel: SlideButton + one of the two tab views
    ├── SlideButton.ets          the animated two-position pill switch
    └── TextAreaDialog.ets       the add-a-phrase dialog
```

The documented tree names `utils/DataUtil.ets`; the zip ships
`utils/DataUtils.ets`. That is one of the four trees covered by the systematic
`HW-14-0001`.

**The design decision worth copying** is that the whole panel is driven by one
enum on the page, not by booleans in the editor:

```typescript
export enum EditMenuAction { NONE, TEXT, EXPAND, REPLY }
```

`ChatPage` declares `@Provide curMenuAction: EditMenuAction`, and
`CustomRichEditor` reads it with `@Consume` to decide what to mount below the
input row - nothing, `ExpandView`, or `QuickReplyView`. `TEXT` means the soft
keyboard is up, which is how the editor knows to drop its bottom safe-area
padding. Because the states are mutually exclusive by construction, no
combination of flags can put two panels on screen at once, and any component in
the subtree can close the panel by writing `EditMenuAction.NONE`.

The second choice worth copying is `CommonDataSource<T>` in `utils/`: one
generic `IDataSource` with `pushData` / `deleteData` / `setData` and the five
`notifyXxx` helpers, reusable by any `LazyForEach` in the app. `QuickReplyView`
instantiates it as `CommonDataSource<QuickPhrases>` and fills it from the
rawfile in `aboutToAppear`.

## Implementation steps

1. **Model the input box as a state machine** with `EditMenuAction`, provided by
   the page and consumed wherever the panel is mounted or dismissed.
2. **Load the presets from a rawfile** with
   `context.resourceManager.getRawFileContentSync(fileName)` and a
   `util.TextDecoder.create('utf-8')` - the explicit UTF-8 decode is what keeps
   the Chinese phrases intact - then `JSON.parse` into `QuickPhrases[]`.
3. **Feed them to `LazyForEach` through an `IDataSource`,** not to `ForEach`
   through an array: `quickReplyData.setData(phrasesData)` fires
   `notifyDataReload` and the tabs materialise as they scroll into view.
4. **Key the `LazyForEach` on something stable** - `(item: QuickPhrases) =>
   item.title` here. A duplicated category title would collide, so titles are a
   contract on the JSON.
5. **Do not filter inside `LazyForEach` with an `if`.** Skipping item 0 shifts
   every rendered child's index away from the data index, and every guard
   downstream then has to know which space it is in - which is exactly the
   defect (`HW-14-0076`). Slice the array before it reaches the data source.
6. **Keep `Tabs({ index: this.currentIndex })` and `onChange` in agreement.**
   `index` is a controlled input; if `onChange` refuses to store an index, the
   next re-render snaps the strip back to the stale one.
7. **Insert phrases through the `RichEditor` controller,** remembering the
   length of the last insertion so a second tap can delete it first.
8. **Use a plain array plus `push` / `splice`** for the user's own phrases, and
   let the `CustomDialogController` hand the new line back through a `confirm`
   callback.

## Verified snippets

All snippets are from `QuickReplyInChat.zip`. Corrected forms are marked.

**The category tabs — `entry/src/main/ets/common/QuickPhrasesTabs.ets`** (corrected, see `HW-14-0076`)

```typescript
Tabs({ barPosition: BarPosition.Start, index: this.currentIndex, controller: this.controller }) {
  LazyForEach(this.quickPhrases, (item: QuickPhrases, index: number) => {
    TabContent() {
      Column() {
        Divider().strokeWidth(0.5).color($r('sys.color.mask_fourth'));
        List({ scroller: this.listScroller }) {
          ForEach(item.phrases, (phrase: string, index1: number) => {
            ListItem() {
              Text(phrase)
                .maxLines(1)
                .fontColor(this.clickIndex === index1 ? 'rgb(10, 89, 247)' : 'rgba(0, 0, 0, 0.7)');
            }
            .onClick(() => { this.insertSpan(phrase); })
            .size({ height: 30 });
          });
        }
        .layoutWeight(1)
        .edgeEffect(EdgeEffect.None)
        .alignListItem(ListItemAlign.Start);
      }
    }
    .width('100%').height('100%')
    .tabBar(this.tabBuilder(index, item.title));   // FIX: shipped code renders index - 1
  }, (item: QuickPhrases) => item.title);
}
.barMode(BarMode.Fixed)
.animationDuration(400)
.onChange((index: number) => {
  this.currentIndex = index;                       // FIX: shipped code wraps this in if (index > 0)
  this.selectedIndex = index;
})
.onAnimationStart((index: number, targetIndex: number) => {
  if (index === targetIndex) {
    return;
  }
  this.selectedIndex = targetIndex;                // FIX: shipped code wraps this in if (index > 0)
})
```

**Two index spaces, one variable name.** The shipped `LazyForEach` body opens
with `if (index > 0)`, because entry 0 of `reply_phrases.json` is the 常用
category already shown by the other pane. That filter means the first *rendered*
`TabContent` is data index 1 but tab-strip child 0 - which is why the shipped
`tabBar(this.tabBuilder(index - 1, item.title))` subtracts one. `onChange` and
`onAnimationStart`, however, report the **child** index, so their `if (index > 0)`
guards swallow every event for the first visible category. Selecting it leaves
the highlight on the previous tab, and because `Tabs({ index: this.currentIndex })`
is a controlled input, the next re-render - tapping a phrase is enough - snaps
the content back to the stale tab.

Removing the guards fixes the events. Removing the `if (index > 0)` filter as
well, by slicing the category out before `setData`, fixes the confusion at its
root and lets `tabBuilder(index, ...)` drop its `- 1`.

`barMode(BarMode.Fixed)` divides the bar evenly among the categories, which is
right for five short labels; a longer list wants `BarMode.Scrollable`. The
`tabBuilder` reads `selectedIndex` rather than `currentIndex` for its colour -
two states for one concept, so that the label can turn blue when the swipe
animation *starts* while the content index only commits when it ends.

**Inserting a phrase — same file** (as shipped)

```typescript
@Consume replyTextLength: number;
@Consume caretOffset: number;

private insertSpan(text: string) {
  if (text.length <= 0) {
    return;
  }
  if (this.replyTextLength === 0) {
    richController.addTextSpan(text, { offset: this.caretOffset });
  } else {
    richController.deleteSpans({ start: this.caretOffset, end: this.caretOffset + this.replyTextLength });
    richController.addTextSpan(text, { offset: this.caretOffset });
  }
  this.replyTextLength = text.length;
}
```

**Tapping a phrase is a replacement, not an append,** and `replyTextLength` is
the entire mechanism: it remembers how many characters the last quick reply
contributed, so the next tap deletes exactly that range before writing the new
text. The user can browse the categories, tapping through candidates, and the
editor always holds one of them rather than a concatenation.

`replyTextLength` and `caretOffset` are `@Provide`d by `QuickReplyView` and
`@Consume`d identically by both panes, so the two lists share one insertion
cursor - switching from 常用语 to 快捷回复 mid-selection still replaces rather
than appends. Note that `caretOffset` is provided as `0` and never written by
any component in the zip, so insertions always land at the start of the editor;
wiring it from `RichEditor`'s `onSelectionChange` is the missing piece if the
user is allowed to type around the inserted phrase.

**The user's own phrases — `entry/src/main/ets/common/CommonPhrasesTabs.ets`** (as shipped)

```typescript
@State commonReplyData: string[] = [];
private newTextInput: string = '';

dialogController: CustomDialogController = new CustomDialogController({
  builder: TextAreaDialog({
    confirm: (content) => {
      this.newTextInput = content;
      if (this.newTextInput) {
        this.commonReplyData.push(this.newTextInput);   // 新增 - add
        this.newTextInput = '';
      }
    }
  })
});

// inside the list row:
Image($r('app.media.ic_cancel'))
  .size({ width: 16, height: 16 })
  .onClick(() => {
    this.commonReplyData.splice(index, 1);              // 删除 - delete
  });

// the add row:
Row({ space: 4 }) {
  Image($r('app.media.plus_circle')).size({ width: 18, height: 18 });
  Text($r('app.string.add_common')).fontColor('rgba(10, 89, 247, 1)');
}
.onClick(() => {
  this.curMenuAction = EditMenuAction.NONE;
  this.dialogController.open();
})
```

**`push` and `splice` on an `@State` array are enough here** because the list is
the user's own handful of lines - fully resident, edited one item at a time, and
observed by ArkUI at the array level. Reaching for a data source and
`notifyDataAdd` would buy nothing. Contrast the presets next door, which are
paged in by category through `CommonDataSource`: same page, two list strategies,
each matched to its data.

`this.curMenuAction = EditMenuAction.NONE` before `dialogController.open()` is
the load-bearing line in the add handler - it retracts the panel so the dialog
does not open on top of a drawer that would still be there behind it. The array
itself lives on `ChatPage` as `@Provide commonReplyData` and is passed down by
name, so additions survive closing and reopening the panel (though not a
restart - nothing is persisted).

**Loading the presets — `entry/src/main/ets/utils/DataUtils.ets` and `view/QuickReplyView.ets`** (as shipped)

```typescript
export function getDataFromJSON(context: Context, fileName: string) {
  let result: QuickPhrases[] = [];
  try {
    let value: Uint8Array = context.resourceManager.getRawFileContentSync(fileName);
    // Compile Chinese UTF-8
    let decoder = util.TextDecoder.create('utf-8');
    let str = decoder.decodeToString(value);
    result = JSON.parse(str) as QuickPhrases[];
  } catch (err) {
    hilog.error(0x0000, 'testTag', `err msg is ${err.message}`);
  }
  return result;
}

// QuickReplyView.aboutToAppear
aboutToAppear(): void {
  let phrasesData: QuickPhrases[] = getDataFromJSON(this.context, 'reply_phrases.json');
  this.quickReplyData.setData(phrasesData);
}
```

**`getRawFileContentSync` returns bytes, not text.** The explicit
`util.TextDecoder.create('utf-8')` step is what keeps 毒鸡汤 and the rest legible;
reading the array as a string any other way mangles multi-byte characters. The
whole read is wrapped in one `try`, and a failure yields an empty array rather
than a throw - the panel then shows no categories, which is a survivable
degradation for an optional feature.

`setData` on `CommonDataSource` assigns the array and fires `notifyDataReload`,
so a `LazyForEach` already mounted rebuilds against the new content. Loading in
`aboutToAppear` of `QuickReplyView` means the file is read each time the panel
opens; hoisting it to the page or to an `AppStorage` entry would read it once.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. The sample reads one
rawfile from its own bundle and touches no system service.

The preset content is `entry/src/main/resources/rawfile/reply_phrases.json`: six
objects of `{ title, phrases[] }`, the first being 常用 - which the panel never
renders as a tab, because the 常用语 side of the switch is the editable list
instead. That unused first entry is the reason for the index shift behind
`HW-14-0076`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `caretOffset` is `@Provide`d as `0` and never updated, so a quick reply is
  always written at the start of the editor regardless of where the user's
  cursor is.
- Nothing is persisted: phrases added through the dialog live in an `@Provide`
  array on `ChatPage` and are gone on restart. The doc's reference to
  `@arkts.collections`' `Array` is aspirational - the shipped code uses ordinary
  arrays.
- The entry point into the panel is the *second* icon of `ExpandView`
  (`if (index === 1)`), which is the calendar-and-pencil glyph; the other three
  icons do nothing. Nothing about that icon suggests quick replies.
- `ChatPage` calls `setKeyboardAvoidMode(KeyboardAvoidMode.NONE)` and
  `CustomRichEditor` immediately sets `RESIZE` in its own `aboutToAppear` - the
  page-level call is dead.
- The send button flips `isSelf` on every send, so the demo alternates between
  the two participants. That is a demo affordance, not a chat behaviour.
- Panel height is fixed at 200 vp with a 134 vp list inside it; long phrase
  lists scroll, they do not expand.

## Pitfalls

- **`HW-14-0076`** (B/medium, confirmed): the `Tabs` guards confuse the child
  index with the JSON index. `LazyForEach` skips item 0, so the first real
  category is child 0, and `if (index > 0)` in both `onChange` and
  `onAnimationStart` swallows its events; since `Tabs({ index: currentIndex })`
  is a controlled input, the strip snaps back to the stale tab on the next
  re-render. Fix: drop the `> 0` guards - the indices are already shifted - or
  slice the unused category out before `setData` and drop the `- 1` in
  `tabBar` too.
- **`HW-14-0001`** (E/low, confirmed): systematic - four social project trees
  list files their zips do not contain; this one documents `utils/DataUtil.ets`
  where the zip ships `utils/DataUtils.ets`. Fix: regenerate the 工程目录 block.

## References

- `huawei_industry_tree/14_social_communication/docs/36_quick_reply_in_chat.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/quick_reply_in_chat-0000002385043665
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-lazyforeach.md` - `LazyForEach`, `IDataSource`, `DataChangeListener`, key generators
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-lazyforeach
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` - `Tabs`, the `index` controlled input, `onChange`, `onAnimationStart`, `BarMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-richeditor.md` - `RichEditorController.addTextSpan` / `deleteSpans` and their offset semantics
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-richeditor
- `documentation/harmonyos-references/02_application-framework/arkts-apis-arkts-collections-array.md` - the `push` / `splice` contract the doc cites
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-arkts-collections-array
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `ListItemAlign`, `edgeEffect`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `SOCIAL-05` - the systematic tree-mismatch finding (`HW-14-0001`) and its other instances
