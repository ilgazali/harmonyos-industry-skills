---
id: SOCIAL-08
title: Send original or compressed - an embedded PhotoPickerComponent with an 原图 toggle in front of packToData
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/08_send_original_image.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/send_original_image-0000002249291536
sample: huawei_industry_tree/14_social_communication/downloads/SendOriginalImage.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [PhotoPickerComponent, PickerController, PickerOptions, ItemInfo, ClickType, PhotoBrowserInfo, onItemClicked, onEnterPhotoBrowser, onExitPhotoBrowser, exitPhotoBrowser, setPhotoBrowserUIElementVisibility, "image.createImageSource", "image.createImagePacker", packToData, PackingOption, "buffer.from", "fs.openSync", Navigation, NavDestination, NavPathStack, "@Provide", "@Consume"]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0016, HW-14-0017, HW-14-0018, HW-14-0019, HW-14-0004, HW-14-0087]
status: verified-with-fixes
---

## When to use

**Load this card when photos leave your app over a network and the user gets
to decide how much quality to spend** - the 原图 (original image) checkbox
every Chinese messenger has, an "upload full resolution" switch, a
data-saver mode in a feed composer.

Two things make it worth reading past the checkbox. The first is
`PhotoPickerComponent`: unlike `PhotoViewPicker` (see `SOCIAL-05`), which
launches the system picker as a separate ability, this one embeds the gallery
grid *inside* your own page, so you can lay your own toolbar over it - which
is the only way to put an 原图 toggle and a Send button in the same sheet as
the thumbnails. The second is the compression pipeline:
`createImageSource` -> `ImagePacker.packToData` with a `quality` is the
platform's re-encode, and it hands you an `ArrayBuffer` you can base64 without
ever touching the disk again.

**Do not ship this sample's selection handling.** `HW-14-0016` is a
high-severity data-loss defect on the core path - deselecting any photo
empties the whole selection - and `HW-14-0017` shows a success toast for work
that has not finished and may never finish. The architecture is sound; three
handlers are not.

## Feature checklist

- A chat page with two seeded bubbles and a bottom input bar.
- Tapping the image icon pushes a picker sheet (`Navigation` -> `NavDestination`).
- The sheet embeds the system gallery grid with the app's own header and
  footer drawn over it; up to 10 images can be selected.
- Selecting a photo increments the counter; the Send button replaces the send
  icon once anything is selected.
- Deselecting decrements the counter and removes only that photo.
- An 原图 row toggles between original and compressed sending; its label and
  tick change colour with the state.
- Tapping a thumbnail opens the picker's full-screen browser; the app's
  chrome hides and the header switches to `已选择(N)`.
- Double-tapping in the browser hides the status bar and navigation
  indicator; a single tap brings the chrome back; the back arrow exits the
  browser.
- Send returns to the chat and appends the images as bubbles - the original
  URIs when 原图 is on, base64 data URLs of the compressed bytes when it is
  off.
- Tapping a sent image opens an in-app viewer with pinch-zoom, pan, double-tap
  zoom and swipe between images.

## Architecture

One `entry` module. A chat page, a picker sheet, a preview component, a
compression utility and a constants file.

```
entry/src/main/ets
├── common/Constants.ets                 every literal in the project - 51 static readonly fields
├── component/ImageViewerComponent.ets   the sent-image viewer: Swiper + pinch/pan/double-tap gestures
├── entryability/EntryAbility.ets        full-screen layout, avoid areas -> AppStorage, exports fullScreen()
├── entrybackupability/EntryBackupAbility.ets
├── pages
│  ├── MainPage.ets                      @Entry, the Navigation host, the chat list, the bottom bar
│  └── SendPage.ets                      the NavDestination sheet: PhotoPickerComponent + toolbar + send
└── utils
   ├── ImageUtils.ets                    compressImage - fs read, createImageSource, packToData
   └── Interface.ets                     MessageType, MsgContent, ImageSize, ImageLocationInfo
```

The documented tree matches the zip.

**The design decision worth copying** is the `@Provide` / `@Consume` pair
across the navigation boundary. `MainPage` owns `pictures` (the sent-message
list) and `isSend`; `SendPage` consumes both and provides its own `imageUri`
(the current selection). So the sheet can append to the conversation without a
callback, a router parameter or a shared singleton, and popping the sheet
needs no result plumbing:

```typescript
// MainPage
@Provide isSend: boolean = false;
@Provide pictures: Array<string> = [];

// SendPage
@Consume isSend: boolean;
@Consume pictures: Array<string>;
@Provide imageUri: Array<string> = [];
```

**The trap in that same decision** is that the two arrays are both
`Array<string>` and mean completely different things - `imageUri` holds
gallery URIs, `pictures` holds either URIs or `data:` base64 blobs. Three of
the four findings on this sample are the same mistake: code that reaches for
`pictures` where it meant `imageUri` (`HW-14-0016`, `HW-14-0019`). Give the
two lists distinct types, or at minimum distinct names that cannot be
transposed by autocomplete.

Also note `EntryAbility` exporting a free function:

```typescript
export function fullScreen(isFull: boolean)   // toggles 'status' and 'navigationIndicator' bars
```

Both the picker sheet and the image viewer call it. Keeping the window handle
in the ability and exposing one verb is cleaner than passing a `window.Window`
down through three components.

## Implementation steps

1. **Host the sheet in a `Navigation`.** `MainPage` builds
   `Navigation(this.pageInfos)`, the image icon does
   `pushPath({ name: 'sendPage' })`, and `SendPage` is registered through an
   exported `@Builder` (`sendPageBuilder`).
2. **Embed the grid with `PhotoPickerComponent`,** sized to 100% and padded so
   your own header and footer sit in the gaps. Drive the padding from state so
   it can collapse to zero when the browser opens.
3. **Build the options object once and pass that object.** The sample keeps a
   `pickerOptions` field and separately passes an inline literal, so the field
   is dead (`HW-14-0019`).
4. **Clear the selection in `aboutToAppear`** - the sheet can be pushed more
   than once in a session.
5. **In `onItemClicked`, branch on `ClickType.SELECTED`.** On select, push the
   URI; on deselect, **filter that URI out of the selection** - never `pop()`,
   and never rebuild the selection from the sent-message list
   (`HW-14-0016`). Return `true` to let the tick through.
6. **Collapse the app chrome in `onEnterPhotoBrowser`** and restore it in
   `onExitPhotoBrowser`; use `pickerController.setPhotoBrowserUIElementVisibility`
   to hide the picker's own browser widgets, and
   `pickerController.exitPhotoBrowser()` to leave from your own back arrow.
7. **Compress with `packToData`, not by hand.** `fs.readSync` the URI,
   `image.createImageSource(buf)`, then `packToData(source, { format,
   quality })`.
8. **Await the compression before you claim success,** and attach a `.catch`
   to every per-image promise (`HW-14-0017`).
9. **Label the base64 with the format you actually packed.** The sample packs
   `image/jpeg` and prefixes `data:image/png;base64,` (`HW-14-0017`).
10. **Key the message `ForEach` on something unique.** A bare URI or base64
    string collides the moment the same image is sent twice
    (`HW-14-0018`).
11. **Guard the viewer's `onAnimationEnd` on the size entry existing** - it is
    only populated by each `Image`'s `onComplete` (`HW-14-0004`).

## Verified snippets

All snippets are from `SendOriginalImage.zip`. Corrected forms are marked.

**The embedded picker — `entry/src/main/ets/pages/SendPage.ets`** (as shipped)

```typescript
Stack() {
  PhotoPickerComponent({
    //定义Picker相关的配置
    pickerOptions: {
      MIMEType: photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE,
      maxSelectNumber: Constants.MAX_SELECT_NUM,
      isSearchSupported: false,
      isPhotoTakingSupported: false,
      backgroundColor: Constants.BACKGROUND_COLOR,
    },
    //设置Picker中点击图片后的回调事件
    onItemClicked: (itemInfo: ItemInfo, clickType: ClickType): boolean =>
    this.onItemClicked(itemInfo, clickType),
    //点击进入大图时的回调事件
    onEnterPhotoBrowser: (photoBrowserInfo: PhotoBrowserInfo): boolean =>
    this.onEnterPhotoBrowser(photoBrowserInfo),
    //退出大图时的回调事件
    onExitPhotoBrowser: (photoBrowserInfo: PhotoBrowserInfo): boolean =>
    this.onExitPhotoBrowser(photoBrowserInfo),
    //通过PickerController向picker组件发送数据
    pickerController: this.pickerController,
  })
    .height(Constants.FULL_PERCENT)
    .width(Constants.FULL_PERCENT)
    .padding({
      top: this.pickerTop,
      left: this.pickerLeft,
      right: this.pickerRight,
      bottom: this.pickerBottom
    });
```

**Four things carry this design.** `isSearchSupported: false` and
`isPhotoTakingSupported: false` strip the picker's own search field and
camera tile, because the app is supplying its own chrome and two competing
toolbars would be confusing. The state-driven `padding` is the mechanism that
lets both coexist: at rest it is `{top: 59, left: 10, right: 10, bottom: 49}`
so the app's header and footer sit in the margins, and `onEnterPhotoBrowser`
sets all four to zero so the full-screen browser really is full screen.

`pickerController` is the *outbound* channel - the only way for the app to
command the component. `SendPage` uses it for exactly two things:
`setPhotoBrowserUIElementVisibility([0, 1], false)` to hide the browser's
built-in widgets on entry, and `exitPhotoBrowser()` from its own back arrow.
The three `onXxx` callbacks are the inbound channel, and each returns `true`
to mean "proceed" - returning `false` from `onItemClicked` is how you veto a
selection, which is where a per-image size or type rule would live.

The whole thing sits in a `Stack` so the header `Row` (positioned `top: 0`)
and the footer `Row` (positioned `bottom: 0`) draw over the grid, and both are
wrapped in `if (!this.isClickImg)` so a double-tap in the browser can remove
them entirely.

**Selection and deselection — same file** (corrected, see `HW-14-0016`, `HW-14-0019`)

```typescript
// 图片选择
private onItemClicked(itemInfo: ItemInfo, clickType: ClickType): boolean {
  if (!itemInfo) {
    return false;
  }
  let uri: string | undefined = itemInfo.uri;
  if (clickType === ClickType.SELECTED) {
    this.selectNum++;
    // 应用做自己的业务处理。
    if (uri) {
      this.imageUri.push(uri);
      this.pickerOptions.preselectedUris = [...this.imageUri];  // FIX: sample copies this.pictures
    }
    return true; // 返回true则勾选，否则则不响应勾选。
  } else {
    this.selectNum--;
    if (uri) {
      // FIX: sample does `this.imageUri.pop()` then
      //      `this.imageUri = this.pictures.filter(item => item !== uri)`
      this.imageUri = this.imageUri.filter((item: string) => item !== uri);
      this.pickerOptions.preselectedUris = [...this.imageUri];  // FIX: sample copies this.pictures
    }
  }
  return true;
}
```

**The deselect branch is the worst defect in this industry's samples.** As
shipped it does two destructive things in a row: `pop()` removes whatever URI
happens to be *last*, regardless of which thumbnail the user just unticked;
then the reassignment rebuilds `imageUri` from `this.pictures` - the
sent-message list, which is empty before the first send. So unticking any
photo leaves `imageUri` as `[]` while the picker still shows the others
ticked and `selectNum` still reads 2. Press Send and nothing is sent, or on a
later use, stale base64 strings from earlier messages are re-sent
(`HW-14-0016`). One `filter` over the selection itself is the entire fix.

The `preselectedUris` lines are dead code in the sample for a second reason:
`this.pickerOptions` is a `PickerOptions` field that is never passed to the
component - the component receives the separate inline literal shown above.
Wiring the field in (and copying `imageUri`, not `pictures`) is what makes
the sheet remember a selection across pushes (`HW-14-0019`).

**Send and compress — same file** (corrected, see `HW-14-0017`)

```typescript
Button($r('app.string.send'))
  .onClick(async () => {
    this.isSend = true;
    if (!this.isChecked) {
      await this.selectAndCompressPicture();      // FIX: sample calls it un-awaited
    } else {
      for (let index = 0; index < this.imageUri.length; index++) {
        this.pictures.push(this.imageUri[index].toString());
      }
    }
    this.pageInfos.pop();                         // FIX: sample pops before the work starts
    this.uiContext.getPromptAction().showToast({ message: $r('app.string.send_success') });
  });

//选择和压缩图片，在此调用compressImage进行图片压缩，并将图片转成base64
async selectAndCompressPicture(): Promise<void> {
  let pictureUriArr: Array<string> = this.imageUri;
  for (const uri of pictureUriArr) {              // FIX: sample uses forEach + un-awaited .then
    try {
      const data: ArrayBuffer = await ImageUtils.compressImage(uri, 'image/jpeg', this.quality);
      // 将图片转成base64
      let base64Str = buffer.from(data).toString('base64');
      let resultBase64Str = 'data:image/jpeg;base64,' + base64Str;   // FIX: sample says image/png
      this.pictures.push(resultBase64Str);
    } catch (err) {
      hilog.error(0X0000, 'testTag', `compress failed: ${err?.message}`);
    }
  }
}
```

**The 原图 checkbox is one boolean and two entirely different payloads.**
With `isChecked` true the gallery URIs go into `pictures` untouched - no
re-encode, no copy, full resolution. With it false every URI is round-tripped
through `packToData` at `quality: 5` and the result becomes a `data:` URL. The
list rendering does not care which it got, because ArkUI's `Image` accepts
both a `file://` URI and a base64 data URL from the same `string` field. That
is what makes the toggle a one-line branch instead of two send paths.

Three corrections above matter. `forEach` over an `async` body does not wait -
the shipped code returns immediately, the sheet pops, and the success toast
fires while zero images have been encoded; whichever compressions finish do so
in nondeterministic order, so a multi-select arrives shuffled. There is no
`.catch` on the per-image `.then`, and `ImageUtils.compressImage` explicitly
`throw`s on failure, so a failed compression is an unhandled rejection
*behind* a success toast - the message silently never appears. And the prefix
lies: the bytes are packed as `image/jpeg` and labelled `data:image/png`,
which breaks any consumer that trusts the MIME (`HW-14-0017`).

`quality: 5` on a 0-100 scale is very aggressive - close to the smallest,
blockiest JPEG the encoder will emit. It reads as "5% quality", not "level 5
of compression"; pick your own number deliberately.

**The compression utility — `entry/src/main/ets/utils/ImageUtils.ets`** (as shipped)

```typescript
static async compressImage(pictureUri: string, format: string, quality: number): Promise<ArrayBuffer> {
  let sourceFile: fs.File | null = null;
  try {
    sourceFile = fs.openSync(pictureUri, fs.OpenMode.READ_ONLY);
    let size = fs.statSync(sourceFile.fd).size;
    let buf = new ArrayBuffer(size);
    fs.readSync(sourceFile.fd, buf);
    let imageSource = image.createImageSource(buf);
    let imagePackerApi: image.ImagePacker = image.createImagePacker();
    let packOpts: image.PackingOption = { format: format, quality: quality };
    return await imagePackerApi.packToData(imageSource, packOpts);
  } catch (err) {
    hilog.error(0x0000, 'CompressFile', `${$r('app.string.file_compression_failed')}：${err.message}`);
    throw new Error(`${$r('app.string.file_compression_failed')}`);
  } finally {
    if (sourceFile) {
      fs.closeSync(sourceFile);
    }
  }
}
```

**`packToData` is the compression - everything before it is just getting the
bytes into memory.** Decode happens implicitly inside the packer:
`createImageSource(ArrayBuffer)` wraps the encoded file, and `packToData`
decodes and re-encodes it to `format` at `quality`, returning the encoded
result as an `ArrayBuffer` that never hits the disk. `format` accepts
`image/jpeg`, `image/webp` and `image/png`; note that `quality` is meaningless
for PNG, which is lossless - if you pass `image/png` with `quality: 5` you get
a full-size file back and the feature silently does nothing.

The `finally` closing the source fd is right and is the reason this utility is
safe to call in a loop. Two things it does not do: it never calls
`imagePackerApi.release()`, and it interpolates a `Resource` object into a
template string - `${$r('app.string.file_compression_failed')}` stringifies to
`[object Object]` in both the log and the thrown `Error`, so the failure
message carries no information. Resolve the resource through
`resourceManager.getStringSync` if the message is meant to be read.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`, and that is
correct: `PhotoPickerComponent` runs the gallery grid under the system's own
authority and hands back only the URIs the user ticked, each carrying a
temporary read grant. No `ohos.permission.READ_IMAGEVIDEO` is required, and
`fs.openSync` on those URIs works because of that grant.

`deviceTypes` is `["phone", "tablet", "2in1"]`. `MainPage` sets
`KeyboardAvoidMode.RESIZE` in `aboutToAppear` so the chat list shrinks rather
than slides when the keyboard opens, and both pages consume `topRectHeight` /
`bottomRectHeight` from `AppStorage` through `@StorageProp` with `px2vp` at
the point of use.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **`maxSelectNumber` is 10** (`Constants.MAX_SELECT_NUM`), and every selected
  image is base64'd into an in-memory `Array<string>` on the send path. Base64
  inflates by ~33% and the strings live for the lifetime of the page; ten
  originals at full resolution would be tens of megabytes of JS heap. Real
  apps stream to a server instead.
- The chat is a demo: two hardcoded bubbles, no persistence, no network. The
  "sent" images exist only in the `pictures` array.
- `quality: 5` is fixed and not exposed in the UI, so "compressed" always
  means "heavily degraded" - there is no medium setting.
- `fullScreen()` reaches for a module-level `windowStage_` captured in
  `onWindowStageCreate`; it works for a single-window app but will not survive
  multi-instance or a 2in1 second window.
- **Unfiled observation - `hideTitle` is resolved once in a field
  initialiser** via `resourceManager.getStringSync`, so the 已选择 label does
  not follow a runtime language change.

## Pitfalls

- **`HW-14-0016` (B/high, confirmed) — deselecting one photo wipes the whole
  selection.** `SendPage.ets:289-298`: the deselect branch calls
  `imageUri.pop()` (removing the *last* URI whatever was unticked) and then
  reassigns `this.imageUri = this.pictures.filter(...)` from the
  sent-message list, which is empty before the first send. The picker still
  shows the other photos ticked while `imageUri` is `[]`, so Send sends
  nothing - or stale base64 from earlier messages. Fix:
  `this.imageUri = this.imageUri.filter(item => item !== uri);`
- **`HW-14-0017` (B/medium, confirmed) — the success toast fires before
  un-awaited, catch-less compression, and JPEG bytes are labelled PNG.**
  `SendPage.ets:188-199` pops the sheet and toasts before
  `selectAndCompressPicture` has done anything, and each per-image
  `compressImage(...).then(...)` has no `.catch` although the utility throws;
  `:266-269` wraps `image/jpeg` output in `'data:image/png;base64,'`. The same
  pair of defects appears verbatim in `ChatRecentImage`
  (`SendPage.ets:116-126` and `:156-159`). Fix: `await` the compression,
  `.catch` the failures, and emit `data:image/jpeg`.
- **`HW-14-0018` (B/medium, confirmed) — broken `ForEach` keys across six chat
  samples.** Here `MainPage.ets:183-187` keys on the bare URI/base64 string
  (`(item: string) => item`), so sending the same image twice produces
  duplicate keys and ArkUI's diffing either drops the second bubble or reuses
  the wrong node. The systematic set: `CustomAlbumStyle ChatPage.ets:39-43`
  and `ChatPageFileEncryption ChatPage.ets:356-360` stringify whole objects to
  `'[object Object]'`; `ChatRecentImage ChatPage.ets:301-305` repeats this
  sample's bare-content key; `SpeechToText Speech.ets:581-599` uses
  `JSON.stringify(item)`; `EmojiAssociation Chat.ets:139` keys on text plus
  minute-precision time. Fix: append the index or carry a message id -
  `(item: string, index: number) => item + index.toString()`.
- **`HW-14-0019` (B/low, confirmed) — `pickerOptions.preselectedUris` is dead
  code.** `SendPage.ets:44,114-134,286,296`: the field is assigned on every
  select and deselect but the component is given a separate inline literal, so
  nothing reads it; and the assignments copy `this.pictures` (sent messages,
  possibly base64) rather than the selection, so even once wired up it would
  inject invalid URIs. Fix: pass `this.pickerOptions` to the component and
  copy `imageUri`.
- **`HW-14-0004` (B/low, probable) — the viewer dereferences the size of a
  not-yet-loaded image.** `ImageViewerComponent.ets:155-173`: `onAnimationEnd`
  reads `this.imageListSize[index].width`, but that array is only filled from
  each `Image`'s `onComplete`, so swiping onto an image whose load has not
  finished throws a `TypeError`. `imagePreview
  ImagePreview.ets:188-206` has the identical defect. Fix:
  `if (!this.imageListSize[index]) { return; }` before the two assignments.

## References

- `documentation/harmonyos-references/04_media/ohos-file-photopickercomponent.md` - `PhotoPickerComponent`, `PickerOptions`, `PickerController`, `ItemInfo`, `ClickType`, and the browser callbacks
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ohos-file-photopickercomponent
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagepacker.md` - `ImagePacker.packToData`, `PackingOption.format` and `quality`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagepacker
- `documentation/harmonyos-references/04_media/js-apis-image.md` - `image.createImageSource` over an `ArrayBuffer`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-image
- `documentation/harmonyos-references/02_application-framework/ts-container-swiper.md` - `Swiper`, `disableSwipe` and `onAnimationEnd` in the image viewer
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-swiper
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-foreach.md` - the `ForEach` key generator contract behind `HW-14-0018`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-foreach
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - why duplicate keys break diffing
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `SOCIAL-05` - the launched-picker alternative (`PhotoViewPicker`), when the gallery need not be embedded
