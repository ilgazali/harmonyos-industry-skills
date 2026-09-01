---
id: SOCIAL-32
title: Keyword search in chat history - highlight the match, then pop an index back and scrollToIndex
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/32_seek_chat_history.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/seek_chat_history-0000002351789373
sample: huawei_industry_tree/14_social_communication/downloads/SeekChatHistory.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [Navigation, NavPathStack, NavDestination, pushPathByName, "NavPathStack.pop", PopInfo, ListScroller, scrollToIndex, Search, SearchController, onChange, Span, ForEach, RegExp, "@Provide", "@Consume", "@StorageProp"]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0069, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card for the **"search inside this conversation, then jump to the
message" flow** - the one behind the ⋯ menu of every chat app. Two problems are
solved together here, and they are independent: rendering a keyword highlight
inside a text run, and getting an index back from a page two levels deep in the
navigation stack so a `List` can scroll to it.

The highlight half is a pure function - string in, `{char, isAim}[]` out -
rendered as a `Text` of `Span`s. It has nothing to do with chat and covers any
"show me where the match is" list: search results, log viewers, contact
filters.

The navigation half is the more transferable idea. `NavPathStack.pop(result)`
plus the `onPop` callback of `pushPathByName` is a **typed return value from a
page**, the ArkUI equivalent of `startActivityForResult`. This sample chains it
across two levels - the search page pops an index to the details page, which
pops it again to the chat page - which is worth studying precisely because the
chaining is what makes it fragile.

## Feature checklist

- The chat opens on a conversation of 17 seeded messages, own and other,
  bubbles mirrored by sender.
- A ⋯ button in the title bar opens a chat-details page.
- 查找聊天内容 (find chat content) on that page opens the search page.
- Typing shows the matching messages, newest first, each with avatar, sender
  name and date.
- The keyword is drawn in the highlight colour inside the surrounding message
  text.
- Tapping a result returns to the chat and scrolls that message into view.
- Clearing the field should return the page to its pre-search state.
- Only matching messages should be rendered and clickable.

## Architecture

One `entry` module, four pages, one data file. No services, no persistence -
the message list is a hardcoded `MSG` array.

```
entry/src/main/ets
├── entryability/EntryAbility.ets            full-screen layout, avoid areas -> AppStorage, light mode forced
├── model/MessageContent.ets                 MsgContent {isSelf, content, data} + MSG: 17 entries
└── pages
   ├── HomePage.ets                          @Entry: an empty Navigation that pushes chatPage on appear
   ├── ChatPage.ets                          the conversation, the ListScroller, the ⋯ button
   ├── ChatDetailsPage.ets                   the profile card and the 查找聊天内容 row
   └── SearchChatHistoryPage.ets             the Search box, findKeyword, the results list
```

The documented tree matches the zip. All three destinations are registered in
`route_map.json` (`chatPage`, `chatDetailsPage`, `searchChatHistoryPage`) with
exported `@Builder` functions; `main_pages.json` contains only
`pages/HomePage`.

**The design decision worth copying** is `HomePage`: an `@Entry` component
holding `@Provide pathInfos: NavPathStack` and an *empty* `Navigation` with
`hideNavBar(true)`, which pushes `chatPage` in `aboutToAppear`. The visible
chat is therefore a `NavDestination` like every other page, not a special root.
Every page reaches the stack through `@Consume pathInfos`, so nothing has to
thread a controller down through props. When a stack has more than two levels,
this is the shape that stays manageable.

**The part worth avoiding** is the two-level pop chain. `ChatPage` pushes
`chatDetailsPage` with an `onPop` that consumes the result as a message index;
`ChatDetailsPage` pushes `searchChatHistoryPage` with an `onPop` whose only job
is to `pop(info.result)` again, relaying the value upward. It works, but the
index is untyped (`info.result as number`) at every hop, and an intermediate
page that pops for any other reason - a back press, a cancel - delivers
`undefined` into `scrollToIndex`. A result object with a discriminator, or a
shared state holder, survives refactoring better than a relay chain.

## Implementation steps

1. **Make the stack a `@Provide`** on an entry component whose `Navigation` is
   empty, and push the real first page in `aboutToAppear`.
2. **Give the message `List` a `ListScroller`** and keep it on the page that
   owns the list - it is the only thing that can scroll it later.
3. **Push with a callback**: `pushPathByName(name, param, (info: PopInfo) => ...)`,
   and return the index with `pathInfos.pop(index)`.
4. **Escape the query before building a `RegExp`.** User input goes straight
   into a pattern otherwise, and a lone `(` throws.
5. **Return segments, not HTML.** `findKeyword` produces
   `{ char, isAim }[]`; the view maps `isAim` to the highlight colour. Keep the
   matching pure and testable.
6. **Return an empty array when nothing matched**, so callers can use
   `.length` as the "does this message match" predicate.
7. **Filter the data before `ForEach`, do not hide content inside the row**
   (`HW-14-0069`) - a row that renders nothing still occupies width and still
   carries its `onClick`.
8. **Derive the "has searched" flag from the query**, do not latch it on the
   first keystroke (`HW-14-0069`).
9. **Call `scrollToIndex` from the `onPop` callback** on the chat page, using
   the index of the message in the unreversed source array.

## Verified snippets

All snippets are from `SeekChatHistory.zip`. Corrected forms are marked.

**The matcher — `entry/src/main/ets/pages/SearchChatHistoryPage.ets`** (as shipped)

```typescript
class FormatString {
  //定义为解析后的标签内容数组，char是要显示的内容，isAim为是否高亮标志
  char: string = '';
  isAim: boolean = false;
}

// 转义特殊字符的函数
escapeRegExp(subStr: string): string {
  return subStr.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}

// 查找搜索关键词并高亮显示
findKeyword(mainStr: string, subStr: string): FormatString[] {
  let result: FormatString[] = [];
  if (mainStr.length === 0 || subStr.length === 0) {
    return result;                                  // empty query matches nothing
  }
  let escapedSubStr = this.escapeRegExp(subStr);
  let globalMatchRegex = RegExp(escapedSubStr, 'g');
  let indexList: number[] = [];
  let matchResult: RegExpExecArray | null = null;
  while ((matchResult = globalMatchRegex.exec(mainStr)) !== null) {
    indexList.push(matchResult.index, globalMatchRegex.lastIndex - 1);
  }
  let cache = '';
  let hasTarget = false;
  for (let index = 0; index < mainStr.length; index++) {
    if (!indexList.includes(index)) {
      cache = cache + mainStr[index];
      if (index === mainStr.length - 1) {
        result.push({ char: cache, isAim: false });
      }
    } else {
      if (cache.length > 0) {
        result.push({ char: cache, isAim: false });
        cache = '';
      }
      result.push({ char: subStr, isAim: true });
      hasTarget = true;
      index = index + subStr.length - 1;            // skip past the match
    }
  }
  return hasTarget ? result : [];
}
```

**Two decisions in here are right and worth keeping.** `escapeRegExp` before
`RegExp(...)` is not optional: the query is raw user input, and a single `(`
would otherwise throw inside a `build()`. And returning `[]` rather than the
un-highlighted segments when nothing matched turns the same function into the
row filter - `findKeyword(...).length` is the predicate the rest of the page
uses.

The shape is less good. `exec` in a loop already yields every match position;
pushing start and end indices into a flat array and then re-scanning the string
character by character with `indexList.includes(index)` makes the function
O(n·m) where a single pass with `substring` between match offsets would be
O(n). It matters because of how often this runs - see the next snippet. Note
also that the highlighted segment is `subStr`, the query, not the matched text,
which is correct only because the regex has no `i` flag.

**The results list — same file** (corrected, see `HW-14-0069`)

```typescript
.onChange((value: string) => {
  this.text = value;
  this.isSearched = value.length > 0;    // FIX: sample set isSearched = true unconditionally
});

// 包含搜索关键词的聊天记录列表
@Builder
chatHistoryList() {
  Column() {
    // FIX: filter first. The sample iterated every message and merely hid the
    // contents of non-matching rows, leaving a full-width clickable Row behind.
    ForEach(this.data.slice().reverse().filter((item: MsgContent) =>
      this.findKeyword(item.content, this.text).length > 0),
      (item: MsgContent) => {
        Row() {
          Image(item.isSelf ? $r('app.media.ic_user_avatar_1') : $r('app.media.ic_user_avatar_2'))
            .width(48)
            .height(48)
            .borderRadius(20)
            .margin({ top: 16, bottom: 12 });
          Column({ space: 5 }) {
            Text(item.isSelf ? $r('app.string.user_name1') : $r('app.string.user_name2'))
              .margin({ top: 10 });
            Text() {
              ForEach(this.findKeyword(item.content, this.text), (part: FormatString) => {
                Span(part.char)
                  .fontSize(14)
                  .fontColor(part.isAim ? $r('app.color.high_light_color') : 'rgba(0, 0, 0, 0.6)')
                  .fontWeight(400);
              });
            };
          }
          .width(128)
          .margin({ left: 16 })
          .alignItems(HorizontalAlign.Start);
          Text(item.data)
            .fontSize(14)
            .width(112)
            .margin({ left: 16 });
        }
        .width('100%')
        .onClick(() => {
          this.pathInfos.pop(this.data.indexOf(item));   // index into the unreversed array
        });
      }, (item: MsgContent) => `${this.data.indexOf(item)}`);
  }
  .width(328)
  .borderRadius(16)
  .backgroundColor(Color.White);
}
```

**Hiding content is not the same as not rendering a row.** The shipped builder
wraps the avatar, the name and the date each in their own
`if (this.findKeyword(item.content, this.text).length)`, so a non-matching
message draws an empty `Row` - still `width('100%')`, still carrying
`onClick(() => this.pathInfos.pop(this.data.indexOf(item)))`. The list looks
like three results and is in fact seventeen stacked invisible buttons; a tap in
the whitespace under the results pops back to the chat and scrolls to a message
the user never chose. Filtering the array is both the correct rendering and the
correct hit-testing.

`isSearched` has the same character of defect: `onChange` set it to `true` and
nothing ever set it back, so clearing the field left the (empty, all-hiding)
result list mounted. Deriving it from `value.length` costs nothing.

Notice how many times `findKeyword` runs in the original: four calls per
message per keystroke (three visibility guards plus the render), over the whole
history. With the filter it is still called twice per surviving row; computing
the segments once into a local `FormatString[]` per item is the next
improvement. The container is also a plain `Column`, not a `List`, so nothing
is virtualised - fine for 17 messages, wrong for a real conversation.

**Getting the index back to the chat — `entry/src/main/ets/pages/ChatPage.ets`** (as shipped)

```typescript
@State index: number = 0;
@Consume pathInfos: NavPathStack;
private listScroller: ListScroller = new ListScroller();

Image($r('app.media.ic_public_ellipsis'))
  .width(24)
  .height(24)
  .onClick(() => {
    this.pathInfos.pushPathByName('chatDetailsPage', null, (info: PopInfo) => {
      this.index = info.result as number;
      // 跳转到搜索关键词所在位置
      this.listScroller.scrollToIndex(this.index);
    });
  });

// ... the list the scroller drives
List({ scroller: this.listScroller }) {
  ForEach(this.data, (item: MsgContent, index: number) => {
    ListItem() { /* bubble */ }
  }, (item: MsgContent, index: number) => `${JSON.stringify(item)}_${JSON.stringify(index)}`);
}
```

and the relay in `ChatDetailsPage.ets`:

```typescript
.onClick(() => {
  this.pathInfos.pushPathByName('searchChatHistoryPage', null, (info: PopInfo) => {
    this.pathInfos.pop(info.result);        // relay the index one level further up
  });
});
```

**`onPop` is where the scroll belongs, not `onPageShow`.** The callback fires
after the destination has popped, on the page that owns the `ListScroller`, so
`scrollToIndex` runs against a list that is already back on screen. That is the
whole mechanism: `pop(value)` → `PopInfo.result` → `scrollToIndex`.

Two cautions come with it. `info.result as number` is an unchecked cast at both
hops - a back-press pop carries no result, and `scrollToIndex(undefined)` is
what arrives. And the index is the position in the **unreversed** `MSG` array,
which is why the search page calls `this.data.indexOf(item)` rather than using
the loop index of its reversed copy; get that wrong and the chat scrolls to the
mirror-image message.

The chat list's key generator here is `JSON.stringify(item) + index`, which is
unique - unlike the object-valued and content-only keys the systematic finding
`HW-14-0018` records across six other samples in this industry. It is,
however, expensive: the whole message is serialised per item per render. An
`id` field on `MsgContent` would do the same job. The search page's own
`ForEach` has no key generator at all.

## Permissions & config

**None.** The sample declares no `requestPermissions` and no
`extensionAbilities` - there is not even an `EntryBackupAbility`. All data is
the hardcoded `MSG` array.

`EntryAbility` forces light mode
(`setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT)`), goes
full-screen with `setWindowLayoutFullScreen(true)`, and publishes
`topRectHeight` / `bottomRectHeight` from `getWindowAvoidArea` into
`AppStorage`, which every page reads with `@StorageProp` and applies as padding
or margin. The `avoidAreaChange` listener is registered and never released -
the same boilerplate defect that recurs across these samples.

`deviceTypes` is `phone`, `tablet`, `2in1`; `routerMap` is `$profile:route_map`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Search is over 17 in-memory messages with no persistence, no paging and no
  cross-conversation scope.
- Matching is case-sensitive and literal; there is no fuzzy or
  whitespace-insensitive matching, and the highlight renders the query rather
  than the matched text.
- The results container is a `Column`, not a `List` - every result is realised
  eagerly.
- Result rows are fixed-width (`Column` 128 vp, date 112 vp) inside a 328 vp
  card, so a long message is clipped rather than elided.
- The returned index is relayed through an intermediate page as an untyped
  `Object`; any other pop path delivers `undefined`.
- `scrollToIndex` positions the message but nothing highlights it in the chat
  once you arrive.

## Pitfalls

- **`HW-14-0069`** (B/low, probable): the results builder iterates every
  message and only hides the *contents* of non-matching rows, so each one
  leaves a full-width `Row` with an active `onClick` that pops an arbitrary
  index - tapping blank space below the results jumps the chat to a message
  the user never selected. `isSearched` also latches `true` on the first
  keystroke and is never cleared, so emptying the field leaves the result list
  mounted. Fix: filter the data before `ForEach`, and derive `isSearched` from
  the query length.
- Not a numbered finding, but adjacent: `findKeyword` runs four times per
  message per keystroke over the whole history, and is itself O(n·m) because it
  re-scans the string against a flat index list rather than slicing between
  match offsets.
- Also not numbered: `info.result as number` is unchecked at both hops of the
  pop chain, so a plain back press out of the details page calls
  `scrollToIndex(undefined)`.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `NavPathStack`, `pushPathByName`, `pop`, `PopInfo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - route maps and returning a result from a destination
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `ListScroller`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-container-scroll.md` - `scrollToIndex`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scroll
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-search.md` - `Search`, `SearchController`, `onChange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-search
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-span.md` - per-`Span` styling inside one `Text`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-span
- `SOCIAL-08` - the systematic `ForEach` key defect (`HW-14-0018`) this sample's chat list narrowly avoids
