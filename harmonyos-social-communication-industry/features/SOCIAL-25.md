---
id: SOCIAL-25
title: Emoji pack association - match typed text against a Record and float a sticker strip over the input bar
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/25_emoji_pack_recommended.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/emoji_pack_recommended-0000002330999945
sample: huawei_industry_tree/14_social_communication/downloads/EmojiAssociation.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [hilog, window]
permissions: [ohos.permission.INTERNET]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0059, HW-14-0018, HW-14-0087, HW-14-0088]
status: verified-with-fixes
---

## When to use

Load this card when the input bar should **suggest content while the user is
still typing** - the sticker strip that pops above the keyboard the moment a
message contains a trigger word. The mechanism is deliberately small: a
`Record<string, EmojiItem[]>` of keyword to stickers, a `String.includes` scan
on every `onChange`, and a floating `Row` positioned over the input bar.

The sticker case is the demo, but the shape generalises to any
suggest-as-you-type affordance that must not steal focus: @-mention
completion, slash commands, quick-reply chips, a "send as voice" hint. What
makes it different from a dropdown is that the strip is **positioned**, not
laid out - it draws over the chat list at a negative `y` offset, so the input
row never reflows and the keyboard never closes.

The sticker images themselves are **network images**, loaded by string URL
from a `BASE_URL` string resource. That is the second reusable idea here:
sticker packs are content, not app resources, so they ship from a server and
the app only needs `ohos.permission.INTERNET` and an `onError` handler. The
doc's 说明 section explains how to point `BASE_URL` at a local HFS file
server for the demo to work at all - **without that step the sample renders
empty boxes**, which is worth knowing before you conclude the code is broken.

## Feature checklist

- Typing into the input runs a keyword match on every keystroke.
- When a keyword matches, a white rounded strip of stickers appears above the
  input bar, with a small triangle pointing down at it.
- The strip's width grows with the number of matched stickers.
- Duplicate stickers matched by two different keywords appear once.
- Clearing the input hides the strip.
- Tapping an image sticker sends it immediately as an own message and clears
  the input; tapping a text emoji appends it to the input instead.
- The message list scrolls to the bottom after every send.
- A time header appears above a message when more than five minutes have
  passed since the previous one.

## Architecture

One `entry` module, four files under `pages`. No data layer, no repository -
the chat is an in-memory array on the page.

```
entry/src/main/ets
├── entryability/EntryAbility.ets    full-screen window, avoid areas -> AppStorage
├── entrybackupability/
└── pages
    ├── Chat.ets                     @Entry: the list, the strip, the input bar, the matcher
    ├── ChatRecord.ets               one bubble: left/right x text/image-sticker
    ├── Models.ets                   ChatMessage and EmojiItem interfaces
    └── TopNameBar.ets               the contact-name header
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that `ChatRecord` is a pure
presentational component driven by three `@Prop` booleans and strings -
`isImageEmoji`, `isLeftSide`, `chatText` - and contains no notion of a
"sticker message" beyond "render an `Image` from a URL instead of a `Text`".
That keeps the four bubble variants (own text, own sticker, peer text, peer
sticker) as one `if/else` tree in one file, and lets `Chat.ets` decide
alignment by wrapping the component in a `Column` with the right
`alignItems`. Adding a fifth message type is a branch in `ChatRecord`, not a
new component.

**The decision worth avoiding** is that `Chat.ets` stores time as a free-form
`string` and formats it in two different places with two different formats.
That is the direct cause of `HW-14-0059`: `sendTextMessage` writes
`toISOString()` while `addImageEmoji` writes `'H:MM'`, and both flow into the
same `new Date(...)` parse.

## Implementation steps

1. **Model the map as `Record<string, Array<EmojiItem>>`,** keyed by the
   trigger word. A single keyword maps to several stickers, so the value is
   an array and matches are concatenated with a spread.
2. **Run the match from `TextInput.onChange`,** not from a submit handler -
   the strip is a live suggestion. Return early on empty text and hide the
   strip.
3. **Dedupe by `(type, content)`** after the scan: two keywords can name the
   same sticker.
4. **Store every timestamp in one format.** Use `toISOString()` on both send
   paths and format at render (`HW-14-0059`); `new Date('14:05')` is
   `Invalid Date` and poisons every comparison it touches.
5. **Give the `ForEach` a unique key.** The shipped generator is
   `item.text + item.time`, which collides when the same sticker is sent
   twice inside one minute (`HW-14-0018`). Append the index or carry a
   message id.
6. **Float the strip with `.position()`,** not by inserting it into the
   column flow, so showing it does not resize the list or dismiss the
   keyboard.
7. **Set `KeyboardAvoidMode.RESIZE` in `aboutToAppear`** and restore
   `OFFSET` in `aboutToDisappear`, so the list shrinks under the keyboard
   instead of the whole page sliding up.
8. **Load stickers by URL with an `onError` handler** on every `Image`, and
   declare `ohos.permission.INTERNET`.

## Verified snippets

All snippets are from `EmojiAssociation.zip`. Corrected forms are marked.

**The matcher — `entry/src/main/ets/pages/Chat.ets`** (as shipped)

```typescript
emojiMap: Record<string, Array<EmojiItem>> = {
  '开心': [
    { type: 'image', content: 'happy_1.png' },
    { type: 'image', content: 'happy_2.png' }
  ]
};

matchEmojis(text: string) {
  if (text.length === 0) {
    this.showEmojis = false;
    return;
  }

  let matched: Array<EmojiItem> = [];
  let keywords = Object.keys(this.emojiMap);
  // 遍历所有表情关键词进行匹配
  for (let keyword of keywords) {
    if (text.includes(keyword)) {
      matched.push(...this.emojiMap[keyword]);
    }
  }

  if (matched.length > 0) {
    // 对匹配结果进行去重处理
    let uniqueEmojis = matched.filter((emoji, index, self) =>
    index === self.findIndex((t) => (
      t.type === emoji.type && t.content === emoji.content
    ))
    );
    this.matchedEmojis = uniqueEmojis;
    this.showEmojis = true;
  } else {
    this.showEmojis = false;
  }
}
```

**Three properties of this matcher carry the design.** It scans *keys*, not
the text - `Object.keys` then `text.includes(keyword)` - so the cost is
`O(keywords)` per keystroke and the map can hold multi-character Chinese
triggers like 开心 (happy) without any tokenizer. It concatenates across
keywords, so "开心" and a hypothetical "笑" both contributing `happy_1.png`
is the normal case, which is why the dedupe on `(type, content)` exists at
all. And `showEmojis` is written on **every** path, including the empty-text
early return, so the strip can never survive the input being cleared.

The `EmojiItem.type` discriminator (`'image'` versus anything else) is what
lets the same map hold server-hosted sticker files and plain unicode emoji;
the strip branches on it and gives them different tap behaviour - a sticker
is *sent*, a unicode emoji is *appended to the input*.

**The floating strip — same file** (as shipped)

```typescript
Column() {
  if (this.showEmojis && this.matchedEmojis.length > 0) {
    Row() {
      ForEach(this.matchedEmojis, (emoji: EmojiItem) => {
        if (emoji.type === 'image') {
          Image(this.BASE_URL + emoji.content)
            .width(44)
            .height(44)
            .margin(5)
            .onClick(() => {
              this.addImageEmoji(emoji.content);
            })
            .onError((error: ImageError) => {
              hilog.error(0x0000, 'testTag', `ImageLoadError： ${error.message}`);
            })
        } else {
          Text(emoji.content)
            .fontSize(30)
            .margin(5)
            .onClick(() => {
              this.inputText += emoji.content;
              this.showEmojis = false;
            })
        }
      });
    }
    .padding(10)
    .backgroundColor('#ffffff')
    .borderRadius(16)
    .width(this.matchedEmojis.length * 50 + 20)      // 44 + 2*5 margin, per sticker
    .position({ x: 250 - this.matchedEmojis.length * 25, y: -60 })

    Polygon()
      .width(16)
      .height(12)
      .fill('#ffffff')
      .points([[100, 4], [108, 13], [116, 4]])
  }
}
.width('100%')
.expandSafeArea([SafeAreaType.KEYBOARD])
```

**The strip is sized and placed arithmetically, not by layout.** Width is
`n * 50 + 20` because each sticker occupies 44 vp plus two 5 vp margins, and
`x` is `250 - n * 25` so the strip stays centred on a fixed 500 vp-wide
notion of the screen as it grows. That is the honest description of the code
and also its weakness: on a tablet or a resized 2in1 window the strip drifts
off centre, because none of these numbers come from the measured container.
If you copy this, replace the two literals with a measured width and
`FlexAlign.Center`.

`expandSafeArea([SafeAreaType.KEYBOARD])` on the wrapper is what keeps the
strip glued to the input bar while the soft keyboard is up; combined with
`KeyboardAvoidMode.RESIZE` set in `aboutToAppear`, the list above shrinks
rather than the page sliding, so the strip's `y: -60` stays meaningful.

The `Polygon` is the little downward tail under the strip. It is drawn in the
column flow *after* the strip, and the strip is lifted out of flow by
`position`, so the tail lands where the strip's bottom edge is.

**Timestamps and the message key — same file** (corrected, see `HW-14-0059`, `HW-14-0018`)

```typescript
addImageEmoji(imageName: string) {
  let now = new Date();
  let timeStr = now.toISOString();     // FIX: shipped code writes `${now.getHours()}:${mm}`

  this.chatMessages.push({
    text: imageName,
    isEmoji: false,
    isImageEmoji: true,
    isMe: true,
    time: timeStr
  });
  this.showEmojis = false;
  this.inputText = '';
  this.scrollToBottom();
}

private shouldShowTime(index: number): boolean {
  if (index === 0) {
    return true;
  }
  let currentMsg = this.chatMessages[index];
  let prevMsg = this.chatMessages[index - 1];

  let currentTime = currentMsg.time ? new Date(currentMsg.time) : new Date();
  let prevTime = prevMsg.time ? new Date(prevMsg.time) : new Date();

  // 如果两条消息间隔超过5分钟，显示时间
  return currentTime.getTime() - prevTime.getTime() > 5 * 60 * 1000;
}

// in build(), the list:
ForEach(this.chatMessages, (item: ChatMessage, index: number) => {
  // ...
}, (item: ChatMessage, index: number) => item.text + (item.time ?? '') + index);
//                                                                     ^^^^^^^ FIX
```

**One format, parsed in one place.** `shouldShowTime` and `getFormattedTime`
both go through `new Date(item.time)`. The four seeded messages and
`sendTextMessage` supply ISO strings, which parse; `addImageEmoji` shipped
`'14:5'`-style strings, which do not. An `Invalid Date` returns `NaN` from
`getTime()`, and `NaN > 300000` is `false`, so the header is silently
suppressed around every sticker - and if a header *is* taken (index 0), the
weekday lookup yields `星期undefined NaN:NaN`. Note that `padStart` is applied
to the minutes but not the hours in the shipped string, so it is not even a
consistent `HH:MM`.

The key generator is the sample's instance of the industry-wide broken-key
systematic (`HW-14-0018`): `item.text + item.time` for a sticker is the file
name plus a minute-precision string, so sending `happy_1.png` twice inside
the same minute produces two identical keys and ArkUI reuses one node instead
of rendering two bubbles. Appending the index makes the key unique;
carrying a real message id is better still.

**The sticker bubble — `entry/src/main/ets/pages/ChatRecord.ets`** (as shipped)

```typescript
@Component
export struct ChatRecord {
  @Prop isImageEmoji: boolean;
  @Prop isLeftSide: boolean;
  @Prop chatText: string;
  BASE_URL = this.getUIContext().getHostContext()!.resourceManager
    .getStringSync($r('app.string.BASE_URL').id);

  build() {
    if (this.isLeftSide) {
      if (this.isImageEmoji) {
        Row() {
          Image($r('app.media.avatar2')).width(40).height(40)
          Image(this.BASE_URL + this.chatText)      // network sticker
            .width(80)
            .height(80)
            .onError((error: ImageError) => {
              hilog.error(0x0000, 'testTag', `ImageLoadError： ${error.message}`);
            })
        }
      } else {
        Row() {
          Image($r('app.media.avatar2')).width(40).height(40)
          bubbleTail(false, '#ffffff');             // @Builder polygon, points left
          Text(this.chatText).fontSize(16).padding(12).backgroundColor('#ffffff')
        }
      }
    }
    // ... the mirrored isSelf branch
  }
}
```

**Two details are worth lifting.** The bubble tail is a free `@Builder`
function taking `(isRight, bubbleColor)` and drawing a three-point `Polygon`,
so the same primitive serves both sides by flipping its point list - cheaper
and sharper than a nine-patch image for a solid-colour bubble. And
`BASE_URL` is read once per component instance from a string resource with
`getStringSync`, which is what makes the sticker host configurable without a
rebuild; the trade-off is that it is read in a field initialiser, so a
missing `BASE_URL` resource throws during component construction rather than
producing a broken image.

Sticker bubbles have **no** tail and no background - the image floats
directly on the chat background, which is the conventional treatment and the
reason `isImageEmoji` branches before `isLeftSide` styling is applied.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.INTERNET",
    "reason": "$string:reason_internet",
    "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
  }
]
```

`INTERNET` is a `system_grant` permission - no runtime dialog, no
`requestPermissionsFromUser` call anywhere in the sample. It is needed purely
because the stickers are fetched by URL. `when: "always"` is generous for a
foreground chat; `"inuse"` would be the tighter declaration.

The sticker host lives in `resources/base/element/string.json` as `BASE_URL`.
The doc's 说明 prescribes running the HFS file server from the
`RcpFileTransfer` sample, binding its disk source to
`src/main/resources/base/media`, and pasting the resulting URL into
`BASE_URL`. Until that is done every sticker `Image` fails and only the
`onError` log fires.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `deviceTypes` is `["phone"]` only, and the strip's `position({ x: 250 - ... })`
  geometry assumes a phone-width screen.
- The keyword map holds exactly one entry (开心 / happy) with two stickers.
  There is no server-side pack list and no persistence of recent stickers.
- The peer never replies: `chatMessages` is seeded with four static entries
  and only own messages are ever appended.
- The microphone, emoji-panel and "+" buttons are decorative - they carry no
  `onClick`.
- `scrollToBottom` is a bare `setTimeout(..., 100)` around
  `scroller.scrollTo`, so on a slow first layout the list can settle above
  the newest message.

## Pitfalls

- **`HW-14-0059`** (B/medium, confirmed): `addImageEmoji` stores time as
  `'H:MM'` while `sendTextMessage` stores `toISOString()`, and both are read
  back through `new Date(...)`. Sticker messages therefore carry an
  `Invalid Date`, which suppresses the five-minute time header via `NaN`
  comparisons or renders `星期undefined NaN:NaN`. Fix: store `toISOString()`
  on both paths and format only at render.
- **`HW-14-0018`** (B/medium, confirmed - systematic across six chat samples):
  the `ForEach` key generator is `item.text + (item.time || '')`. For
  stickers that is the file name plus a minute-precision timestamp, so the
  same sticker sent twice within a minute yields duplicate keys and ArkUI's
  diffing reuses the wrong node. Fix: append the index or key on a message id.
- **The strip's placement is arithmetic, not layout.** `width(n * 50 + 20)`
  and `position({ x: 250 - n * 25 })` are tuned for one screen width; on a
  foldable or a resized window the strip drifts. Measure the container
  instead.
- **`BASE_URL` is resolved in a field initialiser** in both `Chat` and
  `ChatRecord` via `getStringSync($r(...).id)`. It is read twice, and a
  failure surfaces during component construction rather than as a load error.
- **The stickers are unauthenticated plain HTTP by default** in the doc's HFS
  setup. Do not ship that shape; a real sticker CDN needs HTTPS and cache
  headers.

## References

- `huawei_industry_tree/14_social_communication/docs/25_emoji_pack_recommended.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/emoji_pack_recommended-0000002330999945
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-image.md` - `Image` with a string URL, `onError`, `ImageError`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-image
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-foreach.md` - the key generator and why duplicates break diffing
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-foreach
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md` - `onChange`, `onFocus`, `onBlur`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
- `documentation/harmonyos-guides/03_application-framework/arkts-manage-keyboard.md` - `KeyboardAvoidMode.RESIZE` versus `OFFSET`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-manage-keyboard
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.INTERNET`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `SOCIAL-27` - the other sticker keyboard in this industry, with `RichEditor` and `ImageSpan` instead of a plain input
