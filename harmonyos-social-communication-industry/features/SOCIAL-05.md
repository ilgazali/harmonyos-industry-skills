---
id: SOCIAL-05
title: Recommended photo picking - let RecommendationType pre-filter the gallery to QR codes, and save back through a dialog instead of a permission
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/05_image_recognition.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_recognition-0000002236898070
sample: huawei_industry_tree/14_social_communication/downloads/IntelligentImageRecognition.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [common, fileIo, fileUri, hilog, photoAccessHelper, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0001]
status: verified
---

## When to use

**Load this card when the user has to find one specific kind of photo in a
gallery of thousands** - a QR code someone sent them, a screenshot of a
ticket, a picture of a pet, a photo with text in it. The pattern is to hand
the search to the system: pass a `RecommendationType` into
`PhotoSelectOptions.recommendationOptions`, and the picker opens with the
matching images already surfaced instead of the raw camera roll.

The second half of the sample is the mirror image - writing a picture *back*
into the gallery. It uses `showAssetsCreationDialog`, which grants a one-shot
write for exactly the files the user confirms, so the app never declares
`ohos.permission.WRITE_IMAGEVIDEO`. This sample declares **no permissions at
all**, and that is the point worth taking away.

Both halves generalise past the QR-code demo. Any social app that says "pick
the photo of your ID", "attach a screenshot", or "save this sticker to your
album" is this card. The one caveat is that recommendation is a *hint*, not a
filter: the system only recommends once its own gallery analysis has run, so
the picker must still work when the recommendation list comes back empty.

## Feature checklist

- A home card with two round icon buttons under it: save, and filter.
- Tapping the filter icon opens the system picker already narrowed to QR-code
  images, with `maxSelectNumber` of 1.
- Selection is logged; the picker closes on pick or cancel without the app
  holding any media permission.
- Tapping the save icon copies the bundled image into the app cache, then
  raises the system save-confirmation dialog.
- Confirming writes the file into the gallery and shows a success toast.
- Refusing the dialog shows a failure toast and writes nothing.
- The page renders under the status bar (full-screen layout with avoid areas
  published into `AppStorage`).

## Architecture

One `entry` module, one page, two free functions in `tools/`. There is no
model layer and no state beyond the `SaveButtonOptions` literal.

```
entry/src/main/ets
├── entryability/EntryAbility.ets        full-screen layout, avoid areas -> AppStorage, forces light mode
├── entrybackupability/EntryBackupAbility.ets
├── pages/MainPage.ets                   @Entry, the card and the two icon buttons
└── tools/
    ├── SavePicture.ets                  mediaToCache (resource -> sandbox) + saveDialogToGallery (sandbox -> album)
    └── ShowQRCodePictures.ets           the recommendation-filtered picker call
```

**The documented tree does not match the zip** (`HW-14-0001`): the 工程目录
block lists `entryability/Entryability.ets`, while the archive contains
`EntryAbility.ets`. The case difference is not cosmetic on a case-sensitive
build host, and it is one of four social-industry trees with the same class of
drift.

**The design decision worth copying** is that neither tool function is a
component method. `showQRCodePictures()` and the `mediaToCache` /
`saveDialogToGallery` pair are plain exported async functions that take a
`Context` and a `UIContext` as arguments. The page passes
`this.getUIContext().getHostContext()` and `this.getUIContext()` in at the
call site and owns nothing else. That is what makes the media logic reusable
from a second page, a share target, or a unit test - and it is why the whole
picker feature is 42 lines. Compare the chat samples in this industry, where
the same picker work is welded into a 300-line page struct.

## Implementation steps

1. **Do not declare a media permission.** Check `module.json5` has no
   `requestPermissions` block at all - both halves of this feature are built
   on temporary, user-confirmed authorisation.
2. **Build the options in two layers**: a `RecommendationOptions` carrying
   only `recommendationType`, then a `PhotoSelectOptions` that references it
   alongside `MIMEType` and `maxSelectNumber`.
3. **Wrap the whole call in try/catch *and* attach `.catch` to the promise.**
   The synchronous throw and the asynchronous rejection are different failure
   paths and the sample handles both; the document's snippet handles neither.
4. **Treat an empty result as normal.** Recommendation depends on the gallery
   having finished its own image analysis, which the system only runs on a
   charging, screen-off, cool device. Never gate the UI on a non-empty
   recommendation.
5. **To save, stage the bytes in the app sandbox first.** `mediaToCache`
   writes the resource into `applicationContext.cacheDir` and returns a real
   URI via `fileUri.getUriFromPath` - the picker dialog needs a path, not a
   `Resource`.
6. **Await the staging write before handing the URI on.** The shipped
   `mediaToCache` does not (see Constraints).
7. **Call `showAssetsCreationDialog` with matched arrays**: one
   `PhotoCreationConfig` per source URI, its `fileNameExtension` agreeing with
   the actual bytes.
8. **Copy by file descriptor, then close both.** `fileIo.copyFile(srcFd,
   desFd)` followed by two `closeSync` calls; a leaked fd here survives the
   toast.
9. **Regenerate the documented project tree** so `EntryAbility.ets` is spelled
   as it is in the archive (`HW-14-0001`).

## Verified snippets

All snippets are from `IntelligentImageRecognition.zip`.

**The recommendation-filtered picker — `entry/src/main/ets/tools/ShowQRCodePictures.ets`** (as shipped)

```typescript
import { photoAccessHelper } from '@kit.MediaLibraryKit';
import { BusinessError } from '@kit.BasicServicesKit';
import { hilog } from '@kit.PerformanceAnalysisKit';

export async function showQRCodePictures() {
  try {
    // 指定筛选类型为二维码
    let recommendOptions: photoAccessHelper.RecommendationOptions = {
      recommendationType: photoAccessHelper.RecommendationType.QR_CODE
    };
    let options: photoAccessHelper.PhotoSelectOptions = {
      MIMEType: photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE,
      maxSelectNumber: 1,
      recommendationOptions: recommendOptions
    };
    let photoPicker = new photoAccessHelper.PhotoViewPicker();
    photoPicker.select(options).then((photoSelectResult: photoAccessHelper.PhotoSelectResult) => {
      hilog.info(0X0000, 'testTag', '筛选成功: ' + JSON.stringify(photoSelectResult));
    }).catch((err: BusinessError) => {
      hilog.error(0X0000, 'testTag', `筛选失败: ${err.code}, ${err.message}`);
    });
  } catch (error) {
    let err: BusinessError = error as BusinessError;
    hilog.error(0X0000, 'testTag', `推荐失败: ${err.code}, ${err.message}`);
  }
}
```

**Three options carry the design.** `recommendationOptions` is the only one
that does anything intelligent - it tells the gallery which of its own
classification buckets to lead with, and `RecommendationType.QR_CODE` is one
member of an enum that also covers ID cards, profile pictures, art and text
images. `MIMEType: IMAGE_TYPE` narrows the picker to stills so a video never
appears in a QR-code result. `maxSelectNumber: 1` is what makes this a
scan-one-code flow rather than a multi-attach flow; raise it and the same
options object serves an attachment picker.

Note what is *not* here: no context, no permission check, no component. A
`PhotoViewPicker` starts the system picker as a separate ability, which is
exactly why the caller needs no media permission - the app only ever receives
the URIs the user picked. The two error paths are both real:
`new PhotoViewPicker()` and the options validation can throw synchronously,
while a user cancel or a picker failure rejects the promise.

The Chinese log strings are the sample's own: 筛选成功 (filter succeeded),
筛选失败 (filter failed), 推荐失败 (recommendation failed).

**Saving without a write permission — `entry/src/main/ets/tools/SavePicture.ets`** (as shipped)

```typescript
export async function saveDialogToGallery(context: common.UIAbilityContext, uiContext: UIContext,
  srcFileUris: Array<string>,
  photoCreationConfigs: Array<photoAccessHelper.PhotoCreationConfig>) {
  let helper = photoAccessHelper.getPhotoAccessHelper(context);
  try {
    // 基于弹窗授权的方式获取媒体库的目标uri
    let desFileUris: Array<string> = await helper.showAssetsCreationDialog(srcFileUris, photoCreationConfigs);
    // 将来源于应用沙箱的照片内容写入媒体库的目标uri
    let desFile: fileIo.File = await fileIo.open(desFileUris[0], fileIo.OpenMode.WRITE_ONLY);
    let srcFile: fileIo.File = await fileIo.open(srcFileUris[0], fileIo.OpenMode.READ_ONLY);
    await fileIo.copyFile(srcFile.fd, desFile.fd);
    fileIo.closeSync(srcFile);
    fileIo.closeSync(desFile);

    uiContext.getPromptAction().showToast({ message: $r('app.string.save_success_tip') });
    hilog.info(0x0000, TAG, 'create asset by dialog successfully');
  } catch (err) {
    uiContext.getPromptAction().showToast({ message: $r('app.string.get_permission_failed') });
    hilog.error(0x0000, TAG, `failed to create asset by dialog successfully errCode is: ${err.code}, ${err.message}`);
  }
}
```

**`showAssetsCreationDialog` returns URIs the app is allowed to write, and
nothing else.** The user sees one confirmation sheet listing the files; on
confirm the system creates the assets in the album and hands back their
target URIs, already writable by this app for this operation. On refusal the
call rejects, which is why the whole body sits in one `try` and the `catch`
shows the 获取权限失败 (failed to get permission) toast. There is no state to
roll back because nothing was created.

The two arrays are positional: `photoCreationConfigs[i]` describes
`srcFileUris[i]`. The caller in `MainPage` supplies
`{ title: 'startIcon', fileNameExtension: 'png', photoType: PhotoType.IMAGE,
subtype: PhotoSubtype.DEFAULT }` - the extension must match the real bytes,
because the gallery indexes on it.

Copying by descriptor rather than by reading the file into an `ArrayBuffer`
keeps a large photo out of the JS heap entirely; `fileIo.copyFile` moves it
inside the kernel.

**Staging a bundled resource into the sandbox — same file** (as shipped)

```typescript
export async function mediaToCache(context: common.UIAbilityContext, photo: Resource): Promise<string> {
  const applicationContext = context.getApplicationContext();
  // 获取应用文件路径
  const cacheDir = applicationContext.cacheDir;
  // onClick触发后5秒内通过createAsset接口创建图片文件，5秒后createAsset权限收回。
  let uri = cacheDir + '/startIcon.png';
  try {
    // 使用uri打开文件，可以持续写入内容，写入过程不受时间限制
    let file = await fileIo.open(uri, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
    // photo需要替换为开发者所需的图像资源文件
    context.resourceManager.getMediaContent(photo.id, 0)
      .then(async value => {
        let media = value.buffer;
        // 写到媒体库文件中
        await fileIo.write(file.fd, media);
        await fileIo.close(file.fd);
      });
  } catch (error) {
    const ERR: BusinessError = error as BusinessError;
    hilog.error(0x0000, TAG, `Failed to save media to cache. Code is ${ERR.code}, message is ${ERR.message}`);
  }
  // 展示缩略图的关键，需要该方法获得完整路径
  return fileUri.getUriFromPath(uri);
}
```

**The `Resource` -> sandbox step exists because the dialog cannot take a
`Resource`.** `showAssetsCreationDialog` wants file URIs it can open, so a
bundled `$r('app.media.image_code')` has to be materialised first:
`resourceManager.getMediaContent(photo.id, 0)` yields the raw bytes, and
`fileUri.getUriFromPath` turns the sandbox path into the `file://` form the
media stack expects. The sample's own comment names the reason - the path
must be complete for the thumbnail to render.

The comment about a five-second window applies to the *other* save API
(`createAsset` on a granted permission); with the dialog flow there is no
timer, which is the second reason to prefer it.

**Read the ordering carefully before copying this function.** The
`getMediaContent` promise is started but never awaited, and the `return` runs
immediately - so `MainPage` receives the URI and calls `saveDialogToGallery`
on a file that may still be empty. See Constraints.

## Permissions & config

**None.** `module.json5` has no `requestPermissions` array, and that is the
correct outcome for this design:

- The picker half runs in a separate system ability; the app only receives
  URIs the user chose.
- The save half is authorised per-file by `showAssetsCreationDialog`.
- So neither `ohos.permission.READ_IMAGEVIDEO` nor
  `ohos.permission.WRITE_IMAGEVIDEO` is needed. Both are restricted anyway and
  would need a justification to ship.

`deviceTypes` is `["phone", "tablet", "2in1"]`. `EntryAbility` pins the app to
light mode with
`setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT)` before
loading content, so the hardcoded `#FFFFFF` card and `#F1F3F5` page
backgrounds never meet a dark theme.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Recommendation depends on gallery analysis that the app cannot trigger.**
  The system classifies images only while the device is charging with the
  screen off, on adequate battery and normal temperature. Until the QR-code
  category exists in the gallery, the picker opens with nothing recommended -
  on a fresh emulator that is the normal case.
- Only image files can be recommended; there is no video equivalent.
- Location-bearing photos need network access for the system to resolve the
  place name.
- **Unfiled observation - `mediaToCache` returns before the write finishes.**
  `getMediaContent(...).then(...)` is not awaited and its rejection is not
  caught, so the returned URI can point at a zero-length file and the
  subsequent `copyFile` can save an empty image. Awaiting the buffer, then
  writing and closing, is the minimal fix:
  `const value = await context.resourceManager.getMediaContent(photo.id, 0);`
  before the write.
- **Unfiled observation - the file name is a constant.** Every save writes
  `cacheDir + '/startIcon.png'`, so two saves in flight collide. Derive the
  cache name per call if the feature saves more than one asset.
- The picker result is only logged. Wiring the returned `photoUris` to a
  preview is left to the reader; the sample's card image is a static
  `$r('app.media.image_code')`.
- `MainPage` sizes with string literals (`.width('166')`, `.borderRadius('20')`)
  rather than numbers or resources - it works, but it is not the house style
  the rest of this industry's samples use.

## Pitfalls

- **`HW-14-0001` (E/low, confirmed) — four social project trees list files
  their zips do not contain.** Here the 工程目录 block spells
  `entryability/Entryability.ets` while the archive holds `EntryAbility.ets`;
  the same defect appears in `29_custom_album_style` (`PhotoPickerView.ets`
  absent), `36_quick_reply_in_chat` (`DataUtil.ets` absent) and
  `44_web_socket_client2` (`pages/Index.ets` against a zip whose only page is
  `WebSocketClient.ets`). Fix: regenerate the four trees from the archives.
- **The document's own snippet does not compile.** Its `showQRCodePictures`
  has a `try` block closed by a bare `}` with no `catch` or `finally`, an
  empty `.then` body and no `.catch`. Take the shipped version above instead -
  it is the same function with both error paths intact.

## References

- `documentation/harmonyos-references/04_media/ohos-file-photopickercomponent.md` - the `PhotoPicker` component and `PickerOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ohos-file-photopickercomponent
- `documentation/harmonyos-references/04_media/js-apis-photoaccesshelper.md` - `PhotoViewPicker.select`, `PhotoSelectOptions`, `PhotoSelectResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-photoaccesshelper
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `showAssetsCreationDialog` and `PhotoCreationConfig`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-e.md` - the `RecommendationType` enum members
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-e
- `SOCIAL-08` - the in-app `PhotoPickerComponent` alternative, when the picker must be embedded in the page rather than launched as a separate ability
