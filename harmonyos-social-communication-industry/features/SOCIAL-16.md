---
id: SOCIAL-16
title: Drag-and-drop send - accept image and text records from the system transfer station with onDrop
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/16_drop_to_send_image_and_text.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/drop_to_send_image_and_text-0000002283583388
sample: huawei_industry_tree/14_social_communication/downloads/DropToSendImageAndText.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.PerformanceAnalysisKit"]
apis: [display, fileIo, hilog, image, unifiedDataChannel, window]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0034, HW-14-0035, HW-14-0036, HW-14-0027, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card when a surface in your app should **accept content dragged in
from somewhere else** - the system transfer station (中转站), a gallery, a
browser, or another window on a 2in1. The scenario the sample dramatises is
stashing an image or a line of text from one conversation and dropping it into
another to send it, but the mechanism is generic: a drop target that inspects
the incoming records, branches on their type, and turns each one into a domain
object.

Three pieces make it work and each is reusable alone. `.draggable(true)` marks
a component as a drag source. `.onDrop((event: DragEvent) => …)` marks a drop
target and hands you a `UnifiedData`. `getRecords()` returns the payload as
typed records - `unifiedDataChannel.PlainText`, `unifiedDataChannel.Image` -
which you discriminate with `record.getType()` against the standard uniform
type identifiers `general.plain-text` and `general.image`.

The part most implementations get wrong, including this one, is that
`UnifiedData` is a **list**. A drag can carry many records, the transfer station
makes multi-item drags the normal case, and reading only `record[0]` silently
drops the rest (`HW-14-0036`).

## Feature checklist

- A four-tab shell (消息 / 工作台 / 邮件 / 通讯录), only the first populated.
- A conversation list of eight friends with avatar, name, last message and
  time; tapping one pushes the chat page by route name.
- A chat page with left/right bubbles, an image message seeded on first open,
  and a text input row.
- Long-pressing a text bubble selects it and opens a custom four-icon menu.
- Images in the chat are draggable out (`.draggable(true)`); avatars are not
  (`.draggable(false)`).
- Dropping an image anywhere on the message list decodes it and appends it as a
  sent image bubble.
- Dropping plain text fills the composer with it instead of sending directly.
- The draft text of each conversation survives leaving and re-entering the page,
  keyed by friend index in `AppStorage`.
- The list scrolls to the bottom whenever a message is appended or the keyboard
  height changes.

## Architecture

One `entry` module, six ArkTS files.

```
entry/src/main/ets
├── components/
│   ├── ChatDetails.ets       the message list, the drop target, decode() (223 lines)
│   └── Send.ets              the input row: TextInput + onSubmit
├── constants/StyleConstants.ets  numbers, UTIs, FRIENDS_LIST, ALL_MESSAGE_LIST, ICON_ARR
├── entryability/EntryAbility.ets  full screen, avoid areas, avoidAreaChange -> AppStorage
└── pages/
    ├── ChatPage.ets          NavDestination + TitleBar, draft save/restore
    ├── Home.ets              the conversation list
    └── TabsPage.ets          @Entry: Navigation + Tabs
```

The documented tree matches the zip, including the absence of an
`entrybackupability` folder - this sample genuinely has none. Contrast
`HW-14-0001`, the systematic where four other social samples document files
their zips do not contain.

**The design decision worth copying** is that the drop target is the `List`,
not the individual bubbles:

```typescript
List({ space: StyleConstants.MESSAGE_LIST_SPACE, scroller: this.listScroller }) { … }
  .onDrop((event: DragEvent) => { … })
```

One handler covers the whole conversation area, so the user can release
anywhere over the transcript rather than hunting for a target, and there is one
place that knows how to turn a record into a `MessageItem`. Registering
`onDrop` per bubble would multiply the handler by the message count and leave
the gaps between bubbles dead.

**The design decision worth avoiding** is the message store.
`ALL_MESSAGE_LIST` is a module-level `MessageItem[][]` indexed by friend, and
`ChatPage` binds `@State arr: MessageItem[] = ALL_MESSAGE_LIST[AppStorage.get('chat')]`.
Because the arrays are module state, every message ever sent survives page
destruction - which is how the drafts appear to work - but nothing is
persisted, nothing is scoped to a session, and the index into `AppStorage` is
the friend's position in `FRIENDS_LIST`. `ChatDetails` even guards its seed
with `if (this.friendId === 0 && this.arr.length <= 4)` to avoid re-seeding the
image on a second visit, which is exactly the sort of check module-level
mutable state forces on you.

`MessageItem.message` is typed `ResourceStr | image.PixelMap` and disambiguated
by a separate numeric `type` field (`0` text, `1` image). A discriminated union
would be cleaner, but the pairing does keep one array able to hold both kinds
of bubble - which is what the drop handler needs.

## Implementation steps

1. **Mark the drag sources.** `.draggable(true)` on message images;
   `.draggable(false)` on avatars and menu icons, so a long press on them does
   not start a drag that competes with the selection menu.
2. **Put `onDrop` on the container**, not on each item, and take the
   `ListScroller` from the same component so you can scroll after appending.
3. **Bail out early on `event.getData() === undefined`** - a drag can arrive
   with no payload.
4. **Read the UTI with `record.getType()`** and compare against the standard
   identifiers, kept as constants: `general.plain-text`, `general.image`.
5. **Iterate every record** (`HW-14-0036`); `getRecords()` is an array and the
   transfer station routinely produces multi-item drags.
6. **Decode an image URI to a `PixelMap` yourself.** `unifiedDataChannel.Image`
   carries an `imageUri`, not pixels: open with `fileIo`, read into an
   `ArrayBuffer`, `image.createImageSource`, `createPixelMap`.
7. **Attach a `.catch` to the decode promise** so a bad URI logs instead of
   raising an unhandled rejection.
8. **Route text to the composer, images to the transcript.** Dropping text into
   an input the user can still edit is the better default; only images go
   straight through as sent bubbles.
9. **Guard `onSubmit` on non-empty content** (`HW-14-0034`) before pushing a
   message.
10. **Track the keyboard with a `keyboardHeightChange` listener** if the layout
    must resize, and **remove it in `aboutToDisappear`** (`HW-14-0027`).
11. **Convert avoid-area px to vp in the binding expression**, never by
    overwriting the `@StorageProp` (`HW-14-0035`).

## Verified snippets

All snippets are from `DropToSendImageAndText.zip`. Corrected forms are marked.

**The drop handler — `entry/src/main/ets/components/ChatDetails.ets`**
(corrected, see `HW-14-0036`)

```typescript
import { unifiedDataChannel } from '@kit.ArkData';

List({ space: StyleConstants.MESSAGE_LIST_SPACE, scroller: this.listScroller }) {
  // 聊天区域
}
.height(this.keyboardHeight === 0 ? $r('app.float.list_height') : $r('app.float.list_height_keyboard'))
.edgeEffect(EdgeEffect.None)
.expandSafeArea([SafeAreaType.KEYBOARD])
.onDrop((event: DragEvent) => {
  let dragData = event.getData();
  if (dragData === undefined) {
    return;
  }
  let record = dragData.getRecords();
  for (let i = 0; i < record.length; i++) {            // FIX: sample reads record[0] only
    let key: string = record[i].getType();
    if (key === StyleConstants.TEXT_KEY) {             // 'general.plain-text'
      let text = (record[i]) as unifiedDataChannel.PlainText;
      this.mess = text.textContent;
    } else if (key === StyleConstants.IMAGE_KEY) {     // 'general.image'
      decode((record[i] as unifiedDataChannel.Image).imageUri)
        .then((pixlmap) => {
          this.arr.push(new MessageItem(pixlmap!, StyleConstants.IMAGE_TYPE, true));
        })
        .catch((err: Error) => {                       // FIX: sample has no catch
          hilog.error(0x0000, 'DROP', 'decode failed: ' + JSON.stringify(err));
        });
    }
  }
});
```

**`getType()` returns a uniform type identifier, and that is the whole
dispatch.** `general.plain-text` and `general.image` are the standard UTIs for
the two record classes; keeping them in `StyleConstants` rather than inline is
what lets you add `general.file-uri` or `general.video` later without hunting
string literals. The cast that follows (`as unifiedDataChannel.PlainText`) is
safe precisely because the UTI was checked first.

The loop is the fix. `UnifiedData.getRecords()` is defined to return many
records; the transfer station's whole purpose is stashing several items and
dragging them together; the shipped code sends the first and discards the rest
with no error anywhere. Note the text branch overwrites `this.mess` on each
pass, which is acceptable behaviour for a multi-text drag but worth a decision
- concatenating is often more useful.

Text goes into `this.mess`, which is `@Link`ed to the composer's `TextInput`,
while images are appended straight to `this.arr`. That asymmetry is deliberate
and right: dropped text is a draft the user may want to edit; a dropped image
is already the message.

**Decoding a dropped image URI — same file** (as shipped)

```typescript
import { image } from '@kit.ImageKit';
import { fileIo } from '@kit.CoreFileKit';

export async function decode(images: string): Promise<image.PixelMap | undefined> {
  try {
    let buffer: ArrayBuffer;
    let file = fileIo.openSync(images, fileIo.OpenMode.READ_ONLY);
    buffer = new ArrayBuffer(fileIo.statSync(file.fd).size);
    // 读取图片到buffer
    fileIo.readSync(file.fd, buffer);
    fileIo.closeSync(file);
    // 创建imageSource
    let imageSource = image.createImageSource(buffer);
    let imageInfo = await imageSource.getImageInfo();
    let singleWidth = imageInfo.size.width;
    let singleHeight = imageInfo.size.height;
    let singleOpts: image.DecodingOptions = {
      editable: true,
      desiredPixelFormat: image.PixelMapFormat.BGRA_8888,
      desiredSize: { width: singleWidth, height: singleHeight }
    };
    let singlePixelMap = await imageSource.createPixelMap(singleOpts);
    return singlePixelMap;
  } catch (err) {
    hilog.error(0x0000, 'DECODE_IMAGE', 'error: ' + JSON.stringify(err));
  }
  return undefined;
}
```

**A dropped image is a URI, not pixels**, and the URI's grant is scoped to the
drop - so read it now, in the handler, not later. The `openSync` /
`statSync` / `readSync` / `closeSync` sequence is the short path from a file
descriptor to an `ArrayBuffer`; `closeSync` before decoding matters because the
buffer already holds the bytes and leaking fds across a promise boundary is how
these handlers run out of descriptors.

`desiredSize` set from the source's own `imageInfo.size` is a no-op resize that
exists to force the decoder down the explicit-options path;
`desiredPixelFormat: BGRA_8888` with `editable: true` gives a `PixelMap` the UI
can render and later transform. If the images can be large, this is the place
to downscale - decoding a 12MP photo at full size for a 100vp bubble is the
memory cost this sample does not pay attention to.

Returning `undefined` on failure is honest, but every caller must then handle
it; the shipped drop handler asserts with `pixlmap!` and would push
`undefined` as a message on a decode failure. The `.catch` in the corrected
handler above is the minimum; a `if (!pixlmap) return;` is better.

**Sending typed text — `entry/src/main/ets/components/Send.ets`**
(corrected, see `HW-14-0034`)

```typescript
@Component
export struct Send {
  @Link arr: MessageItem[];
  @Link mess: string;

  build() {
    Column() {
      Row() {
        Image($r('app.media.tell')).width($r('app.float.icon_width'));
        TextInput({ text: $$this.mess })
          .backgroundColor(Color.White)
          .borderRadius(StyleConstants.TEXTINPUT_BORDER_RADIUS)
          .onSubmit((entryKey: EnterKeyType, e: SubmitEvent) => {
            if (e.text.trim().length === 0) {          // FIX: absent in the sample
              return;
            }
            this.mess = e.text;
            this.arr.push(new MessageItem(this.mess, StyleConstants.TEXT_TYPE, true));
            this.mess = '';
          })
          .layoutWeight(1);
        Image($r('app.media.face')).width($r('app.float.icon_width'));
        Image($r('app.media.add')).width($r('app.float.icon_width'));
      }
      .width(StyleConstants.FULL_WIDTH);
    }
    .expandSafeArea([SafeAreaType.KEYBOARD]);
  }
}
```

**`$$this.mess` is what makes the drop land in the input.** The two-way binding
means the drop handler in `ChatDetails` can assign `this.mess` and the
`TextInput` updates, while typing updates the same `@Link` back up to
`ChatPage`, which stores it as the per-conversation draft in
`aboutToDisappear`. One string, three owners, no synchronisation code.

`.expandSafeArea([SafeAreaType.KEYBOARD])` on the composer is what lets the
input row sit against the keyboard instead of above the navigation inset.

The missing guard is a one-line defect with a visible symptom: pressing enter
on an empty field appends an empty white bubble, and because `ALL_MESSAGE_LIST`
is module state those bubbles accumulate for the whole session. The sibling
sample `SOCIAL-19` guards the same submit path with `if (this.showText)`, so
the correct form exists elsewhere in this industry.

**Keyboard listener and avoid areas — `ChatDetails.ets` / `TabsPage.ets`**
(corrected, see `HW-14-0027`, `HW-14-0035`)

```typescript
// ChatDetails.ets
private keyboardCb = (data: number) => {
  this.keyboardHeight = this.getUIContext().px2vp(data);
};
private currentWindow?: window.Window;

aboutToAppear(): void {
  window.getLastWindow(this.getUIContext().getHostContext())
    .then(currentWindow => {
      this.currentWindow = currentWindow;             // FIX: sample keeps no reference
      currentWindow.on('keyboardHeightChange', this.keyboardCb);
    })
    .catch((err: Error) => {                          // FIX: sample drops the rejection
      hilog.error(0x0000, 'KEYBOARD', JSON.stringify(err));
    });
}

aboutToDisappear(): void {                            // FIX: absent in the sample
  this.currentWindow?.off('keyboardHeightChange', this.keyboardCb);
}

// TabsPage.ets
@StorageProp('bottomRectHeight') bottomRectHeight: number = 0;

// FIX: the sample overwrites the prop in aboutToAppear:
//   this.bottomRectHeight = this.getUIContext().px2vp(AppStorage.get('bottomRectHeight'));
.padding({
  bottom: this.getUIContext().px2vp(this.bottomRectHeight)   // convert at the use site
})
```

**Both fixes are the same mistake in two costumes: writing a derived value back
over its source.** The listener case writes a subscription into a component
that outlives it - a new `keyboardHeightChange` callback is registered every
time a chat is opened, none is ever removed, and each one keeps firing into a
destroyed component for the lifetime of the window. `H5Interception` and
`LatestMessage` do the same thing; that is the systematic `HW-14-0027`.

The `@StorageProp` case writes vp back into a prop whose source is px.
`aboutToAppear` converts once, so the first frame is right; then
`EntryAbility`'s `avoidAreaChange` handler posts a fresh **px** value into
`AppStorage`, the prop re-syncs, and the same number is now used as vp padding
- roughly three times too large on a typical density. Converting inside the
binding expression, as `SOCIAL-13` does, is immune to this because it never
stores the converted value at all.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. Dragged content
arrives through the unified data channel with its own transient grants, so no
storage or media permission is needed to read the dropped `imageUri` - one of
the practical arguments for accepting content by drop rather than by picker.
Compare `HW-14-0003`, where four sibling social samples ship copy-pasted
`INTERNET` / `VIBRATE` declarations and dead location constants that nothing
uses.

```json5
"pages": "$profile:main_pages",
"routerMap": "$profile:route_map",   // chatPage -> chatPageBuilder
"deviceTypes": ["phone", "tablet", "2in1"]
```

`EntryAbility` sets `COLOR_MODE_LIGHT`, publishes the display size and both
avoid-area heights into `AppStorage`, and - unlike most samples in this
industry - registers `avoidAreaChange` to keep them current. That listener is
never released either, and it is what makes `HW-14-0035` fire.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Drag from the transfer station is a phone/tablet/2in1 interaction; there is
  no keyboard-only or accessibility path to the same feature in this sample.
- `TabsPage` returns `false` from `.onContentWillChange(() => { return false; })`,
  which vetoes every tab switch - three of the four tabs are unreachable and
  only the bar highlight moves. Remove the callback or return `true`.
- `ALL_MESSAGE_LIST` is module-level mutable state. Messages and dropped images
  persist across navigation within a session and vanish on relaunch; there is
  no storage layer.
- The image bubble decodes at full source resolution
  (`desiredSize` = the source size). Large photos are held as full `PixelMap`s
  for the session.
- Only conversation index 0 is seeded with an image
  (`if (this.friendId === 0 && this.arr.length <= 4)`); the other seven start
  with a single text message.
- The chat's `ForEach` has no key generator, so appended messages re-key by
  index.
- The composer's emoji and plus buttons are decorative - no handlers.

## Pitfalls

- **`HW-14-0036`** (B/low, confirmed): `onDrop` reads `record[0]` and ignores
  the rest of `getRecords()`, so dragging several images from the transfer
  station sends only the first, silently; the `decode().then` also has no
  `.catch`. Fix: loop over every record and attach a rejection handler.
- **`HW-14-0034`** (B/low, confirmed): `onSubmit` pushes a `MessageItem`
  regardless of content, so pressing enter on an empty input litters the
  transcript with empty bubbles that persist for the session. Fix: guard on
  trimmed text (`SOCIAL-19` shows the guarded form).
- **`HW-14-0035`** (B/low, confirmed): `@StorageProp` px values are converted to
  vp once in `aboutToAppear` by overwriting the local prop, and the next
  `AppStorage` write from `avoidAreaChange` re-syncs it back to raw px, which
  is then used as vp padding - roughly 3x too large. Fix: convert in the
  binding expression, never overwrite the prop.
- **`HW-14-0027`** (B/low, confirmed): systematic - `keyboardHeightChange`
  listeners are registered per component and never removed (a new one on every
  chat open), and the `getLastWindow` promise has no `.catch`. This sample is
  one of three listed instances, with `H5Interception` and `LatestMessage`.
  Fix: keep the window and callback, call `off` in `aboutToDisappear`.

## References

- `documentation/harmonyos-references/02_application-framework/ts-universal-events-drag-drop.md` - `onDrop`, `DragEvent.getData`, `onDragStart`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-events-drag-drop
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-drag-drop.md` - `draggable`, `dragPreview`, `allowDrop`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-drag-drop
- `documentation/harmonyos-references/02_application-framework/js-apis-data-unifieddatachannel.md` - `UnifiedData.getRecords`, `PlainText`, `Image`, the `general.*` UTIs
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-unifieddatachannel
- `documentation/harmonyos-guides/03_application-framework/unified-data-channels.md` - the unified data model and record types
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/unified-data-channels
- `documentation/harmonyos-guides/03_application-framework/arkts-common-events-drag-event.md` - the drag-and-drop event flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-common-events-drag-event
- `documentation/harmonyos-references/02_application-framework/js-apis-window.md` - `getLastWindow`, `on`/`off('keyboardHeightChange')`, `avoidAreaChange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-window
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` - `onContentWillChange` and its return value
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `huawei_industry_tree/14_social_communication/docs/16_drop_to_send_image_and_text.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/drop_to_send_image_and_text-0000002283583388
- `SOCIAL-14` - the same `bindSelectionMenu` custom menu, used for quoting
- `SOCIAL-13` - the avoid-area conversion done correctly
