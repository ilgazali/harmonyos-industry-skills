---
id: SOCIAL-27
title: Chat bubble backgrounds - a stretchable nine-patch image via backgroundImageResizable, in a RichEditor sticker chat
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/27_chat_bubbles.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_bubbles-0000002332591285
sample: huawei_industry_tree/14_social_communication/downloads/ChatBubbles.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.IMEKit", "@kit.PerformanceAnalysisKit"]
apis: [common, display, hilog, inputMethod, window]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0061, HW-14-0062, HW-14-0027, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card when a container must wear an **image background that stretches
without distorting its corners** - a chat bubble, a ribbon, a badge, a
themed button. The mechanism is `backgroundImageResizable({ slice })`: four
inset distances that mark the frame ArkUI must not scale. It is the CSS
`border-image-slice` idea applied to a background, and it is what makes a
user-selectable bubble skin possible at all, because a bubble has to fit both
"好" and a three-line paragraph out of one PNG.

The card is also the reference implementation of the **sticker keyboard on a
`RichEditor`** in this industry. Text and inline images live in one editable
document; on send, `getSpans()` is walked and each result is classified into
a `SpanItem` of type `TEXT` or `IMAGE`, and the receiving bubble replays that
list as `Span` and `ImageSpan` inside a single `Text`. If you need emoji
mixed into text rather than sent as standalone stickers, this is the shape;
`SOCIAL-25` is the simpler alternative where a sticker is its own message.

**Two defects here are about the keyboard, not the bubble** (`HW-14-0061`,
`HW-14-0027`). The bubble technique is sound and can be lifted as-is; the
keyboard-height plumbing around it should not be.

## Feature checklist

- The chat opens with three preloaded messages, own and peer, already mixing
  text and inline emoji.
- Own messages wear a stretchable image bubble that grows to at most 95% of
  the space between the two avatars and keeps its corners at any size; peer
  messages use a plain white rounded rectangle.
- The input is a `RichEditor`; tapping the emoji button opens a 6-column grid
  of 32 stickers.
- If the soft keyboard is up, pressing the emoji button first closes it, then
  opens the sticker panel after a short delay - no flicker.
- Tapping a sticker inserts it inline at the caret, sized relative to the
  message font.
- Long-pressing a sticker pops up a magnified preview with its Chinese
  meaning.
- Recently used stickers appear in a separate row above the full grid, most
  recent first, capped at six.
- Send lifts the editor's spans into a message, clears the editor and scrolls
  to the bottom.
- Tapping anywhere in the page background dismisses both keyboards.

## Architecture

One `entry` module. The feature is one large page component plus a small
model layer; the emoji keyboard is split into a grid and a single-cell
component so the long-press popup has somewhere to live.

```
entry/src/main/ets
├── constants/ChatConstants.ets      SpanType enum, FaceGridConstants, CommonConstants
├── entryability/EntryAbility.ets
├── entrybackupability/
├── model
│   ├── BasicDataSource.ets          IDataSource + listener bookkeeping, generic
│   ├── BubbleInfo.ets               the bubble class and the MY_BUBBLE instance
│   ├── Emoji.ets                    EmojiModel, the 32-sticker EMOJI_DATA, LastEmojiData
│   └── Message.ets                  SpanItem, MessageBase, TextDetailData
├── pages
│   ├── ChatWithExpression.ets       the page + both bubble components (552 lines)
│   ├── EmojiDetail.ets              one sticker: tap to insert, long-press to preview
│   ├── EmojiKeyboard.ets            the 6-column Grid over an EmojiModel[]
│   └── Index.ets                    @Entry, hosts the page component
└── utils/Logger.ets
```

The documented tree matches the zip exactly.

**The design decision worth copying** is `BasicDataSource<T>` as an abstract
generic base. It implements `registerDataChangeListener`,
`unregisterDataChangeListener` and the five `notifyDataXxx` fan-outs once,
and leaves `totalCount`/`getData` abstract. Both `TextDetailData` (messages)
and `LastEmojiData` (recent stickers) extend it and add only their own
storage and push semantics - `LastEmojiData.pushData` moves an existing entry
to the front and caps the list at six, `TextDetailData.pushData` appends and
notifies. This is the pattern to reach for whenever a screen has two lazy
lists; compare `SOCIAL-28`, which hand-rolls the same boilerplate once per
adapter.

**The decision worth avoiding** is the keyboard-height plumbing. The page
writes `keyboardHeight` into `AppStorage` from window listeners registered in
`aboutToAppear`, and reads it back through `@StorageLink`. That indirection
would be fine; registering a second listener from inside another listener's
callback, with a different formula, is not (`HW-14-0061`).

## Implementation steps

1. **Model the bubble as data, not attributes.** A `BubbleInfo` holds a
   `Resource` and a `ResizableOptions`, so a skin is one object a settings
   screen can swap.
2. **Measure the slices off the actual PNG.** The four values are the
   pixel distances from each edge to where the stretchable middle begins;
   they are asymmetric here (top 14, left 12, bottom 17, right 20) because
   the bubble has a tail.
3. **Apply all three attributes together**: `backgroundImage`,
   `backgroundImageSize({ width: '100%', height: '100%' })` and
   `backgroundImageResizable(...)`. Without the explicit size the image tiles
   instead of filling.
4. **Constrain the bubble, do not fix it**: `constraintSize` with a
   `minWidth`, a `minHeight` and a computed `maxWidth`, so the same bubble
   image serves one character and three lines.
5. **Compute `maxWidth` once from the screen width** in `aboutToAppear` and
   carry it on the message object, so `@Reusable` list items do not
   re-measure.
6. **Register exactly one `keyboardHeightChange` listener, at the top level
   of `aboutToAppear`,** with one formula, and `off` it in
   `aboutToDisappear` (`HW-14-0061`, `HW-14-0027`).
7. **Set `List.initialIndex` from the data source's real length,** not from a
   counter that is still zero (`HW-14-0062`).
8. **Close the system keyboard before opening the sticker panel:** set a
   flag, call `inputMethod.getController().stopInputSession()`, and open the
   panel from the avoid-area change once the keyboard is actually gone.
9. **Insert stickers via `RichEditorController.addImageSpan`,** sizing them
   from the message font size so they sit on the text baseline.
10. **Replay a received message** by mapping `SpanItem[]` to `ImageSpan` /
    `Span` inside one `Text(undefined) { ... }` container.

## Verified snippets

All snippets are from `ChatBubbles.zip`. Corrected forms are marked.

**The bubble model — `entry/src/main/ets/model/BubbleInfo.ets`** (as shipped)

```typescript
/**
 * 气泡模型类
 */
export class BubbleInfo {
  imgSrc: Resource;              // 图片资源
  resizable: ResizableOptions;   // 气泡四边非拉伸距离设置对象

  constructor(imgSrc: Resource, resizable: ResizableOptions) {
    this.imgSrc = imgSrc;
    this.resizable = resizable;
  }
}

export const MY_BUBBLE = new BubbleInfo($r('app.media.bubble'), {
  slice: {
    top: 14,
    left: 12,
    bottom: 17,
    right: 20
  }
});
```

**The four numbers are the whole feature.** `slice` divides the source image
into nine regions: the four corners are drawn at their natural size, the four
edges stretch along one axis only, and the centre stretches in both. So a
bubble whose corner radius is 12 px and whose tail occupies 20 px on the
right survives being stretched to any width and height without the radius
turning into an ellipse or the tail smearing.

The asymmetry is the tell. `right: 20` is larger than `left: 12` because the
tail is on the right of an own-message bubble, and `bottom: 17` exceeds
`top: 14` because the tail sits low. Get these wrong in either direction and
the failure is visible immediately: too small, and the corner gets scaled;
too large, and the fixed regions overlap on a short message and the middle
disappears.

Wrapping the pair in a class rather than applying literals at the call site
is what makes a bubble *theme-able*: one `MY_BUBBLE` here, but the same
component takes any `BubbleInfo`, so a settings screen only has to hand it a
different instance.

**Applying it to an own message — `entry/src/main/ets/pages/ChatWithExpression.ets`** (as shipped)

```typescript
@Component
@Reusable
export struct MessageItemSelfView {
  @State msg: MessageBase = new MessageBase(true, '', '', 0);

  aboutToReuse(params: Record<string, MessageBase>) {
    this.msg = params.msg;
  }

  build() {
    Row() {
      Row() {
        Text(undefined) {
          ForEach(this.msg.spanItems, (item: SpanItem) => {
            if (item.spanType === SpanType.IMAGE) {
              ImageSpan($rawfile(item.imgSrc as string))
                .width($r('app.integer.chat_with_expression_chat_font_size'))
                .height($r('app.integer.chat_with_expression_chat_font_size'))
                .margin(FaceGridConstants.EMOJI_MARGIN)
                .verticalAlign(ImageSpanAlignment.BOTTOM)
                .objectFit(ImageFit.Cover);
            } else if (item.spanType === SpanType.TEXT) {
              Span(item.text);
            }
          });
        }.constraintSize({
          minHeight: $r('app.integer.chat_with_expression_chat_inline_height'),
          maxWidth: this.msg.maxWidth
        })
      }
      .constraintSize({
        minHeight: $r('app.integer.chat_with_expression_chat_inline_height'),
        minWidth: $r('app.string.chat_with_expression_layout_10'),
        maxWidth: this.msg.maxWidth
      })
      .backgroundImage(MY_BUBBLE.imgSrc)
      .backgroundImageSize({ width: '100%', height: '100%' })
      .backgroundImageResizable(MY_BUBBLE.resizable)
      .padding({ top: 10, bottom: 10, left: 14, right: 14 });

      Image($rawfile(this.msg.profilePicture))
        .width(50).height(50);
    }
    .width('100%')
    .justifyContent(FlexAlign.End);
  }
}
```

**Three things make the bubble behave.** The background goes on the *outer*
`Row`, not on the `Text`, so the padding is inside the image and the tail
never overlaps a glyph. `constraintSize` gives a `minWidth` of 10% and a
`maxWidth` computed per message, which is exactly the range
`backgroundImageResizable` exists to survive - a fixed `width` would defeat
the point. And `backgroundImageSize` at 100%/100% is mandatory: the default
repeats the image, and a nine-patch that tiles looks like a bug rather than a
skin.

The message body is `Text(undefined) { ... }`. Passing `undefined` as the
content and supplying children is how ArkUI builds a mixed inline run;
`ImageSpan` with `verticalAlign: ImageSpanAlignment.BOTTOM` then puts the
sticker on the text baseline rather than centring it on the line box.

`@Reusable` with `aboutToReuse` is the right pairing for a chat list - the
node is recycled and only `msg` is re-bound. Note that `maxWidth` travels on
the message object, computed once from the screen width in the page's
`aboutToAppear`, precisely so a recycled item never re-measures.

The peer variant is the same component with the background swapped for
`.backgroundColor(Color.White)` and a three-corner `borderRadius`, which is
the honest comparison: the resizable image buys you a tail and a texture that
a border radius cannot.

**The keyboard listener — same file** (corrected, see `HW-14-0061`, `HW-14-0027`)

```typescript
private keyboardListener = (data: number) => {
  const bottomAvoidArea = this.currentWindow!
    .getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR).bottomRect.height;
  AppStorage.setOrCreate('keyboardHeight', data > 0 ? data - bottomAvoidArea : 0);
};
private currentWindow?: window.Window;

async aboutToAppear() {
  // ... preload the three messages, compute msgFontSize / msgMaxWidth ...
  this.context.windowStage.getMainWindowSync()
    .getUIContext()
    .setKeyboardAvoidMode(KeyboardAvoidMode.RESIZE);

  this.currentWindow = await window.getLastWindow(this.context);
  // FIX: register ONE listener, with ONE formula, at the top level
  this.currentWindow.on('keyboardHeightChange', this.keyboardListener);

  this.currentWindow.on('avoidAreaChange', async data => {
    if (data.type !== window.AvoidAreaType.TYPE_KEYBOARD) {
      return;
    }
    // FIX: the shipped code registers another keyboardHeightChange listener HERE
    // 点击表情按钮之后，等待系统软键盘关闭后再延迟刷新表情键盘，避免屏幕闪烁
    if (data.area.bottomRect.height === 0 && this.isFaceClick === true) {
      setTimeout(async () => {
        this.isFaceDlgOpen = true;
      }, DELAY_TIME);
    }
  });
}

aboutToDisappear(): void {
  this.isFaceDlgOpen = false;
  this.isFaceClick = false;
  // FIX: absent in the sample - listeners outlive the component otherwise
  this.currentWindow?.off('keyboardHeightChange', this.keyboardListener);
  this.currentWindow?.off('avoidAreaChange');
}
```

**The shipped code registers a nested listener.** `aboutToAppear` installs
one `keyboardHeightChange` handler that stores the raw `data`, then installs
an `avoidAreaChange` handler which - on *every* keyboard avoid-area event -
installs *another* `keyboardHeightChange` handler that stores
`data - bottomAvoidArea`. Nothing is ever removed. After a few keyboard
open/close cycles several callbacks are live, half of them subtracting the
navigation-bar height and half not, all writing the same `AppStorage` key.
The input row's bottom padding then depends on callback registration order,
which is not a contract you can rely on.

The `avoidAreaChange` handler's real job is the second half: when the
keyboard's avoid area collapses to zero *and* the emoji button was the cause,
wait 200 ms and open the sticker panel. That sequencing - close the system
keyboard first, swap in the panel second - is the part worth copying, and it
survives the fix intact.

This is the sample's aggravated instance of `HW-14-0027`, the industry-wide
systematic where `keyboardHeightChange` listeners are registered per
component and never removed; the others register one listener and merely leak
it, whereas this one multiplies them.

**The list and its start position — same file** (corrected, see `HW-14-0062`)

```typescript
List({
  scroller: this.scroller,
  initialIndex: this.textDetailData.totalCount() - 1   // FIX: shipped code uses this.msgNums - 1
}) {
  // 使用懒加载组件渲染数据
  LazyForEach(this.textDetailData, (msg: MessageBase) => {
    ListItem() {
      if (msg.isSelf) {
        MessageItemSelfView({ msg: msg });
      } else {
        MessageItemView({ msg: msg });
      }
    };
  });
}
.onAreaChange(() => {
  // 控制列表滚动条到底部
  this.scroller.scrollEdge(Edge.Bottom);
})
.layoutWeight(LAYOUT_WEIGHT);
```

**`msgNums` is only ever written by `sendChatMsg`.** The three preloaded
messages go straight into `textDetailData` without touching it, so at first
build `msgNums` is `0` and `initialIndex` is `-1` - not a valid index. The
list still opens at the newest message, but only because `onAreaChange` fires
during layout and calls `scrollEdge(Edge.Bottom)`; the documented
start-at-the-last-message wiring contributes nothing. Reading
`totalCount()` from the data source makes the declared intent real and
removes the dependence on a layout side effect.

Keep `onAreaChange` regardless - it is what re-pins the list to the bottom
when the keyboard resizes the viewport under `KeyboardAvoidMode.RESIZE`.

**Sticker insert and the long-press preview — `entry/src/main/ets/pages/EmojiDetail.ets`** (as shipped)

```typescript
Image(this.emojiItem!.imgSrc)
  .onClick(() => {
    // 将表情添加到输入框中
    this.controllerRich!.addImageSpan(this.emojiItem!.imgSrc, {
      imageStyle: {
        size: [this.msgFontSize / CommonConstants.INPUT_EMOJI_RESIZE,
          this.msgFontSize / CommonConstants.INPUT_EMOJI_RESIZE],   // 3.2
        verticalAlign: ImageSpanAlignment.CENTER,
        layoutStyle: { margin: FaceGridConstants.EMOJI_MARGIN }
      }
    });
    this.lastEmojiData.pushData(this.emojiItem!);   // 最近使用表情
  })
  .draggable(false)
  .gesture(
    LongPressGesture()
      .onAction(() => { this.isPopup = true; })
      .onActionEnd(() => { this.isPopup = false; })
  )
  .bindPopup(this.isPopup, {
    builder: this.popupBuilder, onStateChange: (e) => {
      if (!e.isVisible) {
        this.isPopup = false;
      }
    }
  });
```

**The sticker size is derived, not fixed.** `msgFontSize` is computed in the
page's `aboutToAppear` as `screenWidth * 100 / 640` - the design width scaled
to the device - and the inserted image is that over 3.2, so the sticker
tracks the font on every screen size instead of being a magic 24 vp.
`onStateChange` resetting `isPopup` is not optional: `bindPopup` reads the
boolean but never writes it back, so a tap-outside dismissal would otherwise
leave the flag true and the preview could never reopen. `draggable(false)`
stops a long press from starting a drag instead of firing the gesture.

## Permissions & config

**None.** The sample declares no `requestPermissions` at all - no network, no
storage. Every sticker is a `$rawfile` shipped in the app, and the bubble is
`$r('app.media.bubble')`.

That is worth noting as a positive: a bubble-theme feature needs no
permissions until the skins come from a server, at which point you inherit
`ohos.permission.INTERNET` and a cache. The `BubbleInfo` indirection is
already the seam where that swap happens.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `backgroundImageResizable` needs
  API 12 or later.
- `msgFontSize` and `msgMaxWidth` are computed once in `aboutToAppear` from
  `display.getDefaultDisplaySync().width`. They do **not** recompute on a
  fold, rotation or window resize, so bubbles keep the width of the launch
  geometry.
- The 32 stickers are `$rawfile('01.png')` … `'32.png'`, and the parser
  assumes exactly that shape: two characters before `.png`
  (`EMOJI_FILENAME_LEN = 2`) and a fixed 19-character `resource://RAWFILE/`
  prefix (`EMOJI_SRC_POS = 19`). A three-character sticker name breaks
  message parsing silently.
- Only own messages get the image bubble; the peer style is a plain white
  rounded rectangle. There is one skin and no picker.
- `LastEmojiData` is in memory only, so recent stickers empty on relaunch,
  and the three seeded messages are fixed string resources.

## Pitfalls

- **`HW-14-0061`** (B/medium, confirmed): a new `keyboardHeightChange`
  listener is registered *inside* the `avoidAreaChange` callback, so one is
  added per avoid-area event; the outer listener stores the raw height while
  every inner one stores `height - bottomAvoidArea`, and none is ever
  removed. The input row's padding then depends on nondeterministic callback
  order, and the listeners keep firing into a destroyed component for the
  window's lifetime. Fix: register one listener with one formula at the top
  level of `aboutToAppear` and `off` it in `aboutToDisappear`.
- **`HW-14-0027`** (B/low, confirmed - systematic across three further
  samples): the same "register a window keyboard listener per component and
  never remove it" defect recurs in `H5Interception`,
  `DropToSendImageAndText` and `LatestMessage`. Treat `window.on` as
  something that always needs a matching `off`, and give
  `window.getLastWindow` a `.catch`.
- **`HW-14-0062`** (B/low, confirmed): `List.initialIndex` is
  `this.msgNums - 1`, and `msgNums` is still `0` at first build because the
  three preloaded messages bypass it - so `initialIndex` is `-1`. The list
  only lands at the bottom by way of a coincidental `scrollEdge` in
  `onAreaChange`. Fix: `this.textDetailData.totalCount() - 1`.
- **`backgroundImageSize` is not optional.** Omit it and the nine-patch tiles
  instead of filling the bubble.
- **Slices are in source-image pixels, not vp.** Re-measure them whenever the
  bubble asset is redrawn or re-exported at another density.
- **The `LazyForEach` has no key generator.** With reusable items and a
  growing list, supply one before this list carries real data.

## References

- `huawei_industry_tree/14_social_communication/docs/27_chat_bubbles.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_bubbles-0000002332591285
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-background.md` - `backgroundImage`, `backgroundImageSize`, `backgroundImageResizable`, `ResizableOptions.slice`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-background
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-richeditor.md` - `RichEditorController`, `addImageSpan`, `getSpans`, `deleteSpans`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-richeditor
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-imagespan.md` - `ImageSpanAlignment` and inline image layout
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-imagespan
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-lazyforeach.md` - `IDataSource` and the notify contract `BasicDataSource` implements
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-lazyforeach
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `on('keyboardHeightChange')`, `on('avoidAreaChange')` and their `off` counterparts
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-guides/03_application-framework/arkts-manage-keyboard.md` - `KeyboardAvoidMode.RESIZE` and `inputMethod.stopInputSession`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-manage-keyboard
- `documentation/harmonyos-guides/03_application-framework/arkts-reusable.md` - `@Reusable` and `aboutToReuse`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-reusable
- `SOCIAL-25` - the simpler sticker model, where a sticker is a whole message rather than an inline span
