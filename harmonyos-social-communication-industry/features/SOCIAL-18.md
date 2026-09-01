---
id: SOCIAL-18
title: Post draft autosave - write the draft on every keystroke, not on the way out
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/18_save_draft_on_exit.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/save_draft_on_exit-0000002284059456
sample: huawei_industry_tree/14_social_communication/downloads/动态发布退出时保存草稿示例代码.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.LocationKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [abilityAccessCtrl, common, dataSharePredicates, fileIo, geoLocationManager, hilog, image, photoAccessHelper, preferences, util, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0040, HW-14-0041, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card when a **composition screen must survive being left** - the
moments/feed post editor, a comment box, a long review, a support ticket.
The user taps back, or the system reclaims the app, and everything they typed
must still be there when they come back.

The name of the feature ("save draft on exit") describes the user-visible
promise, not the technique. The technique is the opposite of exit-time saving:
the draft is written to a `preferences` store on every `onChange`, so it is
already durable long before the user presses back. The back-press dialog is
then only a confirmation of intent - discard or keep - and never the thing
that persists data. That inversion is the transferable idea: **do not attach
persistence to a lifecycle callback you might not get.**

It generalises to any editor with cheap, small state. Preferences is a
key-value store meant for exactly this size of payload; the moment your draft
contains binary (images, audio, attachments) the shape has to change, and
`HW-14-0040` is what happens when it does not.

## Feature checklist

- A moments-style feed page with a camera icon in the title bar that pushes
  the composer.
- The composer has a `TextArea` for body text and a 3-column grid for images,
  capped at nine.
- The + tile opens the system photo picker (multi-select, up to five at a
  time); each pick produces a thumbnail tile.
- Tapping a thumbnail removes it.
- Every text change is written to the preferences store immediately.
- Re-entering the composer restores the text draft.
- Re-entering the composer restores the image draft. (**Not implemented** -
  see `HW-14-0040`.)
- Pressing back opens a confirm dialog; 取消 (cancel) clears the whole store
  and pops, 确定 (confirm) pops keeping the draft.

## Architecture

One `entry` module. A read-only feed page, a composer page, and a preferences
wrapper shared by both halves of the draft.

```
entry/src/main/ets
├── components
│  ├── AddPic.ets            the image grid, the picker call, the image draft write
│  ├── ImageInfo.ets         feed page: 3x3 image grid (display only)
│  ├── InteractiveInfo.ets   feed page: like / comment strip (display only)
│  ├── MainTextArea.ets      the TextArea + the text draft read and write
│  └── UserInfo.ets          feed page: avatar, nickname, caption
├── constants
│  ├── CommonConstants.ets   sizes, caps, and a dead location-permission array
│  └── StyleConstants.ets    numeric literals for the feed page
├── entryability/EntryAbility.ets   full-screen window, avoid areas -> AppStorage
├── entrybackupability/
├── model/ContentInfo.ets    ImageInfo { imagePixelMap, imageArrayBuffer?, imageName }
├── pages
│  ├── MainPage.ets          @Entry, Navigation host, custom title bar
│  └── PostingPage.ets       NavDestination composer + the discard dialog
├── utils
│  ├── FileUtil.ets          photo picker wrapper
│  ├── ListOptionUtils.ets   the preference key names
│  ├── LocationUtil.ets      geoLocationManager on/off - never called
│  └── PreferenceUtils.ets   getStore / save / get / delete
└── viewmodel/ImageData.ets  the static feed content
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that the persistence key is a
constant in one place and the store is opened synchronously on demand:

```typescript
export class LstOptionClass {
  static mainTitle: string[] = ['mainTitle1', 'mainTitle2', 'mainTitle3', 'mainTitle4'];
  static textContent: string[] = ['textContent1', 'textContent2', 'textContent3', 'textContent4'];
  static localImage: string[] = ['LocalImage1'];
}
```

The arrays are indexed (`LstOptionClass.textContent[0]`) because the template
anticipates several parallel drafts - four editors, four slots - without
needing four constants. `PreferencesClass.getStore` uses
`preferences.getPreferencesSync`, so there is no promise to await at the call
site and the write can sit directly inside an `onChange` handler. For a
key-value store this small, sync is the right call.

**The design decision worth avoiding** is `PreferencesClass` having three
near-identical getters (`getPreferenceInfo`, `getPreferenceInfo1`,
`getPreferenceInfo2`) that differ only in their default value and their `as`
cast. Two of the three are wrong (`HW-14-0041`) and one of the three is never
called from anywhere (`HW-14-0040`). A single generic getter taking the
default as a parameter would have made both defects visible.

## Implementation steps

1. **Wrap `preferences` once,** with a store-name constant and a `getStore`
   that calls `getPreferencesSync`. Every other method goes through it.
2. **Write on `onChange`, not on exit.** `putSync` followed by `flushSync`;
   for a string of a few hundred characters the cost is negligible and the
   guarantee is total.
3. **Read in `aboutToAppear`** of the component that owns the field, so the
   restore happens where the field lives rather than in the page.
4. **Pass the correct default to `getSync`** - `''` for a string draft, not
   `0` (`HW-14-0041`).
5. **Store images in a round-trippable encoding.** Base64 the packed JPEG;
   never run image bytes through `TextDecoder` (`HW-14-0040`).
6. **Give the image draft a reader too.** `getPreferenceInfo2` exists and is
   never called; `AddPic` needs an `aboutToAppear` that rebuilds the grid
   (`HW-14-0040`).
7. **Intercept back with `NavDestination.onBackPressed` returning `true`** and
   raise the dialog; the same dialog is opened by the back arrow image, so
   both routes agree.
8. **Discard means `clearSync()` + `flushSync()`** on the store, plus emptying
   the in-memory array - the store and the state must be cleared together.

## Verified snippets

All snippets are from `动态发布退出时保存草稿示例代码.zip` (`SaveDraft`).
Corrected forms are marked.

**The text draft round trip — `entry/src/main/ets/components/MainTextArea.ets`** (as shipped)

```typescript
@Component
export struct MainTextArea {
  @StorageLink('textContent') textContent: string = '';
  private context = this.getUIContext().getHostContext() as common.UIAbilityContext;

  aboutToAppear(): void {
    this.textContent =
      PreferencesClass.getPreferenceInfo(this.context, PreferencesClass.DEFAULT_STORE,
        LstOptionClass.textContent[0]);
  }

  build() {
    Flex({ direction: FlexDirection.Column }) {
      TextArea({ text: this.textContent, placeholder: $r('app.string.richEditor_placeholder') })
        .width(CommonConstants.FULL_PERCENT)
        .height(48)
        .id(CommonConstants.TITLE_ID)
        .constraintSize({ maxHeight: $r('app.integer.text_input_height') })
        .onChange((textContent: string) => {
          this.textContent = textContent;
          PreferencesClass.savePreferenceInfo(this.context, LstOptionClass.textContent[0],
            PreferencesClass.DEFAULT_STORE, textContent);
        });
    }
    .width(CommonConstants.FULL_PERCENT)
    .layoutWeight(CommonConstants.DEFAULT_LAYOUT_WEIGHT)
    .expandSafeArea([SafeAreaType.KEYBOARD]);
  }
}
```

**This is the whole feature for the text half, in twenty lines.** Three
choices carry it. `@StorageLink` rather than `@State` means the draft is also
readable from anywhere else in the app through `AppStorage` - the dialog in
`PostingPage` needs that. `aboutToAppear` is where the read goes, not the
page's `onReady`, so the component is self-restoring wherever it is mounted.
And `expandSafeArea([SafeAreaType.KEYBOARD])` is what stops the soft keyboard
from covering the field it is typing into.

Writing on every keystroke sounds expensive and is not:
`preferences.getPreferencesSync` returns a cached in-memory instance and
`flushSync` writes a small file. If the draft ever grows past a few kilobytes,
debounce the write - but do not move it to `aboutToDisappear`, which is
exactly the callback a process kill does not give you.

**The default value bug — `entry/src/main/ets/utils/PreferenceUtils.ets`** (corrected, see `HW-14-0041`)

```typescript
export class PreferencesClass {
  static DEFAULT_STORE: string = 'DEFAULT_STORE';

  static getStore(content: Context, storeName: string) {
    return preferences.getPreferencesSync(content, { name: storeName });
  }

  static savePreferenceInfo(content: Context, storeKey: string, storeName: string,
                            value: ResourceStr | boolean) {
    const STORE = PreferencesClass.getStore(content, storeName);
    STORE.putSync(storeKey, value);
    STORE.flushSync();
  }

  static getPreferenceInfo(content: Context, storeName: string, storeKey: string) {
    const STORE = PreferencesClass.getStore(content, storeName);
    const VAL = STORE.getSync(storeKey, '');    // FIX: sample passes 0 and casts `as string`
    return VAL as string;
  }

  static getPreferenceInfo1(content: Context, storeName: string, storeKey: string) {
    const STORE = PreferencesClass.getStore(content, storeName);
    const VAL = STORE.getSync(storeKey, false); // FIX: sample passes 0 and casts `as boolean`
    return VAL as boolean;
  }

  static getPreferenceInfo2(content: Context, storeName: string, storeKey: string) {
    const STORE = PreferencesClass.getStore(content, storeName);
    let imageInfoNull: ImageInfo[] = [];
    const VAL = STORE.getSync(storeKey, imageInfoNull);
    return VAL as ImageInfo[];
  }
}
```

**`getSync`'s second argument is the value returned when the key is absent,
and its type is the contract.** The sample passes the number `0` to all three
string/boolean getters and then asserts the result into the wrong type with
`as`. On a first run - or immediately after the discard dialog clears the
store - `getPreferenceInfo` hands back the number `0`, `MainTextArea` assigns
it to `textContent`, and the composer opens with a literal `0` in the field
instead of the placeholder. The `as string` cast is what hides it from the
compiler.

`getPreferenceInfo2` is the only getter with a correctly typed default. It is
also the only getter nothing ever calls.

**The image draft — `entry/src/main/ets/components/AddPic.ets`** (corrected, see `HW-14-0040`)

```typescript
@Component
export struct AddPic {
  @LocalStorageLink('imageUriArray') imageUriArray: ImageInfo[] = [];
  private context = this.getUIContext().getHostContext() as common.UIAbilityContext;

  // FIX: the sample has no aboutToAppear at all - the image draft is write-only
  aboutToAppear(): void {
    const saved = PreferencesClass.getPreferenceInfo2(this.context,
      PreferencesClass.DEFAULT_STORE, LstOptionClass.localImage[0]);
    saved.forEach(async (item: ImageInfo) => {
      const bytes = new util.Base64Helper().decodeSync(item.imageArrayBuffer as string);
      const source = image.createImageSource(bytes.buffer as ArrayBuffer);
      item.imagePixelMap = await source.createPixelMap();
      this.imageUriArray.push(item);
    });
  }

  async getThumbnail(uri: string) {
    // ... photoAccessHelper fetch elided ...
    asset.getThumbnail((err, pixelMap) => {
      if (err === undefined) {
        let imageName = asset.displayName.substring(0, (asset.displayName).indexOf('.'));
        this.pixelMapToBuffer(pixelMap, imageName).then((data) => {
          // FIX: sample decodes JPEG bytes with util.TextDecoder.create('utf-8')
          let stringData = new util.Base64Helper().encodeToStringSync(new Uint8Array(data));
          this.imageUriArray.push({
            imagePixelMap: pixelMap, imageArrayBuffer: stringData, imageName: imageName
          });
          PreferencesClass.savePreferenceInfo1(this.context, LstOptionClass.localImage[0],
            PreferencesClass.DEFAULT_STORE, this.imageUriArray);
        });
      }
    });
  }

  async pixelMapToBuffer(pixelMap: image.PixelMap, displayName: string): Promise<ArrayBuffer> {
    const imagePackerApi: image.ImagePacker = image.createImagePacker();
    let packOpts: image.PackingOption = { format: 'image/jpeg', quality: 100 };
    let buffer = await imagePackerApi.packToData(pixelMap, packOpts);
    this.writeDistributedFile(buffer, displayName);
    return buffer;
  }
}
```

**Two defects, and the second one makes the first unfixable.** The write path
exists and fires on every pick and every delete, so the store really does hold
an image draft - but nothing reads it back, because `getPreferenceInfo2` is
dead code and `AddPic` has no `aboutToAppear`. That alone would be a missing
half of a feature.

What makes it worse is *what* gets written.
`util.TextDecoder.create('utf-8').decodeToString(...)` over a packed JPEG is
not a conversion, it is a filter: every byte sequence that is not valid UTF-8
- which is most of a JPEG - is replaced by U+FFFD, and the original bytes are
gone. So even a correct restore path could not reconstruct the image from what
the sample stored. Base64 is the minimum fix; it is lossless, it is a string
so `putSync` accepts it, and `Base64Helper.decodeSync` gives the bytes back
verbatim for `image.createImageSource`.

Note the third write in `pixelMapToBuffer`: `writeDistributedFile` also dumps
the raw buffer into `context.distributedFilesDir` under the asset's display
name. That path *is* byte-exact and would be the better draft store for
images, with preferences holding only the file names. The sample writes it and
then never reads it either.

**The discard dialog — `entry/src/main/ets/pages/PostingPage.ets`** (as shipped)

```typescript
@CustomDialog
struct CustomDialogExample {
  private context = this.getUIContext().getHostContext() as common.UIAbilityContext;
  @LocalStorageLink('imageUriArray') imageUriArray: ImageInfo[] = [];
  controller?: CustomDialogController;

  build() {
    Column() {
      Text($r('app.string.CustomDialog_text'))
      Row() {
        Button($r('app.string.CustomDialog_cancel'))
          .onClick(() => {
            if (this.controller !== undefined) {
              this.controller.close();
            }
            this.cancel();
            this.imageUriArray = [];                                    // clear the state
            PreferencesClass.getStore(this.context, PreferencesClass.DEFAULT_STORE).clearSync();
            PreferencesClass.getStore(this.context, PreferencesClass.DEFAULT_STORE).flushSync();
          });
        Button($r('app.string.CustomDialog_accept'))
          .onClick(() => {
            if (this.controller !== undefined) {
              this.controller.close();
            }
            this.confirm();                                            // keep the draft, just pop
          });
      }
    };
  }
}

// in PostingPage
.onBackPressed(() => {
  if (this.dialogController !== null) {
    this.dialogController.open();
    return true;                    // swallow the back gesture; the dialog decides
  } else {
    return false;
  }
})
```

**Returning `true` from `onBackPressed` is what makes the dialog meaningful.**
It tells the navigation stack the event was consumed, so the page stays until
one of the two buttons pops it explicitly. Returning `false` - or omitting the
handler - would pop the page underneath the dialog.

The discard branch clears the store *and* the in-memory array, in that order,
because `@LocalStorageLink('imageUriArray')` is shared with `AddPic`: emptying
only the store would leave the grid populated from state until the next
mount. Note that it does not clear `textContent`, which lives in `AppStorage`
rather than the page's `LocalStorage` - `clearSync()` on the store handles the
persisted copy, but the `@StorageLink` in `MainTextArea` still holds the old
string for the lifetime of the app.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions` at all - correct,
because the photo picker (`photoAccessHelper.PhotoViewPicker`) is a system
picker that grants access to exactly the files the user chose, with no
`READ_IMAGEVIDEO` permission needed. That is the pattern to copy: prefer the
picker over the permission.

Two leftovers contradict this:

- `CommonConstants.REQUEST_PERMISSIONS` declares
  `ohos.permission.APPROXIMATELY_LOCATION` and `ohos.permission.LOCATION`.
  Nothing reads the constant.
- `utils/LocationUtil.ets` wraps `geoLocationManager.on('locationChange')`.
  Nothing calls it.

Both are inherited from a larger "publish a moment with a location tag"
template. If you copy this sample as a starting point, delete them - a
declared-but-unused permission array is the kind of thing that gets pasted
into `module.json5` by the next person.

Navigation is wired through `"routerMap": "$profile:route_map"` rather than
an in-page `@Builder` map, so `PostingPage` is reached by name
(`pushPath({ name: 'PostingPage' })`) with no import in `MainPage`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Preferences is a key-value store for small data. The image draft in this
  sample stores whole encoded images as values, which is outside what the
  store is designed for even after the base64 fix - for anything past a
  thumbnail, keep the bytes in the file system and the paths in preferences.
- Nine images maximum (`CommonConstants.MAX_ADD_PIC`), five per picker
  invocation (`FileUtil.fileSelect`). Exceeding nine silently does nothing -
  the `else` branch of the + tile handler is empty.
- The feed page is entirely static: `ImageData.ets` holds nine bundled images,
  one nickname and a three-comment list. Nothing published in the composer
  ever reaches it.
- The doc's 参考文档 points at `@ohos.data.sendablePreferences`, but the sample
  uses plain `@ohos.data.preferences`. The sendable variant matters only if
  the store is shared across workers; this sample is single-threaded.

## Pitfalls

- **`HW-14-0040` (B/medium, confirmed) — the image draft is written but never
  read, and the payload is corrupted on the way in.** `AddPic` saves
  `imageUriArray` on every change, `getPreferenceInfo2` is never called from
  anywhere, and the JPEG buffer is stored as `TextDecoder.decodeToString`
  output so invalid sequences become U+FFFD and the bytes are unrecoverable.
  The doc promises both text *and* image drafts. Fix: base64 the packed
  buffer, and restore in `AddPic.aboutToAppear`.
- **`HW-14-0041` (B/medium, probable) — the string draft's `getSync` default
  is the number `0`.** On a first run, or right after the discard dialog
  clears the store, the composer shows a literal `0` instead of the
  placeholder. The `as string` cast hides the type error. Fix: default to
  `''` (and to `false` in `getPreferenceInfo1`).
- **Not filed:** the discard branch clears the persisted store but not the
  `@StorageLink('textContent')` copy in `AppStorage`, so within one app run
  the text can survive a discard. Assign `''` alongside the `clearSync()`.
- **Not filed:** `writeDistributedFile` is called from inside
  `pixelMapToBuffer` as a side effect, with no `await` on the caller side and
  its result never used. It writes into `distributedFilesDir` under the
  gallery asset's display name, so two picks with the same file name
  overwrite each other.

## References

- `huawei_industry_tree/14_social_communication/docs/18_save_draft_on_exit.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/save_draft_on_exit-0000002284059456
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` - `getPreferencesSync`, `putSync`, `getSync` and its default-value contract, `clearSync`, `flushSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `documentation/harmonyos-guides/03_application-framework/data-persistence-by-preferences.md` - when preferences is the right store and when it is not
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/data-persistence-by-preferences
- `documentation/harmonyos-references/04_media/js-apis-photoaccesshelper.md` - `PhotoViewPicker`, `PhotoSelectOptions`, `getAssets`, `getThumbnail`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-photoaccesshelper
- `documentation/harmonyos-references/04_media/js-apis-image.md` - `createImagePacker`, `packToData`, `createImageSource`, `createPixelMap`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-image
- `documentation/harmonyos-references/02_application-framework/js-apis-util.md` - `Base64Helper` and `TextDecoder`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-util
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textarea.md` - `TextArea` and `onChange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textarea
- `documentation/harmonyos-guides/03_application-framework/arkts-localstorage.md` - `@LocalStorageLink` versus `@StorageLink`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-localstorage
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navdestination.md` - `onBackPressed` and its return value
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navdestination
