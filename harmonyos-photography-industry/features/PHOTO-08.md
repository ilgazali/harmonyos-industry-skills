---
id: PHOTO-08
title: Subject segmentation cut-out - doSegmentation to a foreground PixelMap, then save the rendered result
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/08_image_segmentation.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_segmentation-0000002282720108
sample: huawei_industry_tree/18_photography/downloads/SubjectSegmentation.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.CoreVisionKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [common, fileIo, fileUri, fs, hilog, image, photoAccessHelper, subjectSegmentation, window, doSegmentation, SegmentationConfig, SegmentationResult, PhotoViewPicker, PhotoSelectOptions, createImageSource, createPixelMap, createImagePacker, packToData, showAssetsCreationDialog, PhotoCreationConfig, getComponentSnapshot, setWindowSystemBarProperties]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0002, HW-18-0034, HW-18-0035, HW-18-0036, HW-18-0005, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when the app needs to **lift the subject out of a photo** -
remove the background from a product shot, make a sticker from a pet, cut a
person out for a collage, prepare an avatar. The whole capability is one on-device
call: `subjectSegmentation.doSegmentation({ pixelMap }, config)` returns the
detected subjects, their bounding rectangles, and - if you ask for it - a
`foregroundImage` `PixelMap` with the background made transparent.

There is no model to ship, no network call and no permission. The interesting
part of the card is therefore not the AI call, which is four lines, but the
**three plumbing problems around it** that this sample gets wrong and that every
image-editing feature has to solve: what happens when the user cancels the
picker, how the result gets from a `PixelMap` into the gallery, and which
native handles have to be released.

It generalises to the rest of Core Vision - text recognition, face detection,
document correction - which all take a `PixelMap` and hand back a structure.
Swap the call, keep the surrounding shape.

**Note what "save" means here.** The sample does not save
`result.foregroundImage`; it takes a **component snapshot** of the `Image` that
is displaying it. That is a deliberate and useful trick (it captures whatever
the user actually sees, including scaling and any overlays) but it also means
the saved file is display-resolution, not source-resolution. Decide which you
want before copying.

## Feature checklist

- A dark full-screen page with a large centred image area and a bottom action
  row.
- Empty state: a single plus button that opens the system photo picker,
  images only, one selection.
- After a pick, the chosen photo fills the image area at `ImageFit.Contain`.
- A 抠图 (cut out) button sits at the bottom left of the image area, enabled
  once an image is loaded.
- Tapping it runs subject segmentation and swaps the display for the
  transparent-background foreground image.
- In the segmented state the action row becomes cancel / add-another / finish.
- Cancel returns to the original photo; add-another reopens the picker.
- Finish snapshots the result view and saves it to the gallery through the
  system's create-asset dialog, toasting 保存成功 on success.
- Cancelling the picker must leave the page exactly as it was.

## Architecture

One `entry` module, four files of substance. No model layer - the state is
three booleans and two `PixelMap`s.

```
entry/src/main/ets
├── common/Constants.ets            layout numbers, MAX_SELECT, the 85% stack height
├── entryability/EntryAbility.ets   full screen, avoid areas -> AppStorage, windowStage -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── pages/MainPage.ets              @Entry: the whole UI and the pick/segment/save orchestration
└── utils/
    ├── ImagePickerUtils.ets        openPhotoPicker + loadImage (uri -> PixelMap)
    ├── ImageSegmentation.ets       the Core Vision call and a human-readable analysis string
    └── SaveImage.ets               PixelMap -> PNG -> sandbox -> showAssetsCreationDialog
```

The documented 工程目录 does **not** match the zip: it lists
`entrybackupability/EntryBackAbility.ets`, a truncation of the real
`EntryBackupAbility.ets`. The identical typo appears in `04_ratio_camera`
(whose zip has no backup ability at all), `16_image_effect`, and two docs
outside this industry - the trees are hand-edited from a shared, wrong template
rather than generated from the zips (`HW-18-0002`).

**The design decision worth copying** is the three-state UI driven by two
booleans, `isSelected` and `isSegmented`, with a single `Stack` swapping
between `Image(this.originalImage)` and `Image(this.result?.foregroundImage)`.
The segmented image carries `.id('resultImage')`, and that id is the entire
save path: `getComponentSnapshot().get('resultImage', cb)` rasterises the
rendered component. No second render pass, no manual compositing - whatever the
user sees is what gets saved. It is the cheapest correct answer whenever the
saved artefact is meant to be *the view*, and the reason `PictureSticker`
(`PHOTO-10`) needs the same trick for its overlaid stickers.

**The decision worth avoiding** is setting `isSelected = true` in the *button
handler*, before `pickImage()` has resolved. That single line is what turns a
picker cancel into a broken UI (`HW-18-0035`).

## Implementation steps

1. **Declare nothing.** `module.json5` has no `requestPermissions` block at
   all, and that is correct: `PhotoViewPicker` grants read access to what the
   user picked, `showAssetsCreationDialog` grants write access to what the user
   confirmed, and Core Vision runs on-device. This sample is the counter-example
   to the nine that declare restricted `READ`/`WRITE_IMAGEVIDEO` for the same
   permission-free flows (`HW-18-0004`).
2. **Pick, then load, then flip the state** - in that order (`HW-18-0035`).
   Check `photoUris.length`, never `=== undefined`; a cancel resolves to `[]`.
3. **Decode the URI to a `PixelMap`**: `fileIo.open` read-only,
   `image.createImageSource(fd)`, `createPixelMap()`. Release the `ImageSource`
   and close the fd afterwards (`HW-18-0005`).
4. **Configure the segmentation**: `maxCount` for how many subjects,
   `enableSubjectDetails` for per-subject rectangles, and
   `enableSubjectForegroundImage: true` - without the last flag there is no
   `foregroundImage` and you get only geometry.
5. **`await subjectSegmentation.doSegmentation({ pixelMap }, CONFIG)`** inside a
   `try`, and map failures onto a result object rather than throwing at the UI.
6. **Give the result `Image` a stable `.id()`** and read it back with
   `getComponentSnapshot().get(id, callback, { scale })`.
7. **Pack the snapshot to PNG** (not JPEG - the cut-out has an alpha channel
   that JPEG discards) and write it into `filesDir`.
8. **Hand it to the gallery with `showAssetsCreationDialog`**, and put a
   `.catch` on the chain (`HW-18-0034`) - the user can refuse that dialog, and
   a refusal is a rejection, not an empty array.
9. **Close each fd exactly once, in a `finally`, guarded for null**
   (`HW-18-0034`), and **release the `ImagePacker`** (`HW-18-0036`).

## Verified snippets

All snippets are from `SubjectSegmentation.zip`. Corrected forms are marked.

**The Core Vision call - `entry/src/main/ets/utils/ImageSegmentation.ets`** (as shipped)

```typescript
import { subjectSegmentation } from '@kit.CoreVisionKit';

export class ImageSegmentation {
  static async process(pixelMap: image.PixelMap, maxSubjects: number): Promise<SegmentationResult> {
    try {
      // 配置通用主体分割的配置项SegmentationConfig
      const CONFIG: subjectSegmentation.SegmentationConfig = {
        maxCount: maxSubjects,                  // how many subjects to detect
        enableSubjectDetails: true,             // per-subject rectangles
        enableSubjectForegroundImage: true      // without this there is no cut-out
      };
      const RESULT = await subjectSegmentation.doSegmentation({ pixelMap }, CONFIG);
      return {
        success: true,
        analysis: ImageSegmentation.formatAnalysis(RESULT, maxSubjects),
        foregroundImage: RESULT.fullSubject?.foregroundImage
      };
    } catch (err) {
      const ERR = err as BusinessError;
      hilog.error(0x0000, TAG, `Failed with code ${ERR.code}: ${ERR.message}`);
      return { success: false, error: ERR.message };
    }
  }
}

export interface SegmentationResult {
  success: boolean;
  analysis?: string;
  foregroundImage?: image.PixelMap;
  error?: string;
}
```

**`fullSubject` versus `subjectDetails` is the distinction to internalise.**
`fullSubject` is the union of everything detected, and its `foregroundImage` is
the one cut-out covering all subjects at once - that is what the UI displays.
`subjectDetails` is the per-subject array, each entry carrying its own
`subjectRectangle`, which is what you would iterate to let the user choose
*which* person to keep. The sample formats them into a Chinese debug string
(发现主体数量 - number of subjects found, 主物体区域 - main object region)
that it then never displays; `analysis` is computed and dropped.

The `try`/`catch` returning `{ success: false, error }` rather than throwing is
the right shape for an on-device model call: segmentation fails routinely (no
subject in frame, unsupported device) and that is a UI state, not an exception.
Note the sample never checks `result.success` before rendering
`this.result?.foregroundImage`, so a failure shows an empty image area with no
message.

**Pick and load - `entry/src/main/ets/pages/MainPage.ets` and `utils/ImagePickerUtils.ets`** (corrected, see `HW-18-0035`, `HW-18-0005`)

```typescript
// MainPage - the plus button
Image($r('app.media.add'))
  .onClick(() => {
    this.pickImage();                            // FIX: sample sets isSelected/isSegmented here,
  });                                            //      before the pick has resolved

private async pickImage() {
  try {
    const URIS = await ImagePickerUtil.openPhotoPicker();
    if (URIS.length === 0) {                     // FIX: sample tests `URI === undefined`,
      return;                                    //      which is never true for []
    }
    this.originalImage = await ImagePickerUtil.loadImage(URIS);
    this.result = undefined;
    this.isSegmented = false;
    this.isSelected = true;                      // FIX: only after a successful load
  } catch (err) {
    hilog.error(0x0000, TAG, `${err.message}`);
  }
}

// ImagePickerUtils.loadImage
static async loadImage(names: string[]): Promise<image.PixelMap> {
  let fileSource: fileIo.File | undefined = undefined;
  let imageSource: image.ImageSource | undefined = undefined;
  try {
    fileSource = await fileIo.open(names[0], fileIo.OpenMode.READ_ONLY);
    imageSource = image.createImageSource(fileSource.fd);
    return await imageSource.createPixelMap();
  } catch (err) {
    let error = err as BusinessError;
    throw new Error(`Image loading failed: ${error.message}`);
  } finally {
    imageSource?.release();                      // FIX: never released in the sample (HW-18-0005)
    if (fileSource !== undefined) {
      fileIo.closeSync(fileSource);              // FIX: the fd is leaked in the sample
    }
  }
}
```

**Cancelling a picker is a routine action, not an error path.**
`photoPicker.select()` resolves with `photoUris: []` when the user backs out -
it does not reject and it does not return `undefined` - so the shipped
`if (URI === undefined)` test can never fire, `loadImage([])` indexes `names[0]`
as `undefined`, `fileIo.open(undefined)` throws, and the user gets an
`Image loading failed` log for having changed their mind. Worse, because both
button handlers set `this.isSelected = true` *before* calling `pickImage`, the
cut-out button is left active with no image: tapping it sets `isSegmented = true`
and the page renders an empty result view that looks like a broken feature.
Six samples in this industry share the empty-array bug (`HW-18-0024`);
`ImageStitch` (`PHOTO-05`) and `ImageRotateAndFlip` have the correct guard.

The `finally` in `loadImage` is the fix for two leaks at once: the fd opened for
the decode, and the `ImageSource`, which holds native decoder resources and is
created-and-abandoned in eight files across this industry (`HW-18-0005`).
Release it *after* `createPixelMap` has resolved - releasing earlier, or
closing the fd earlier, breaks the decode, which is its own systematic defect
elsewhere (`HW-18-0025`).

**Snapshot and save - `entry/src/main/ets/pages/MainPage.ets` and `utils/SaveImage.ets`** (corrected, see `HW-18-0034`, `HW-18-0036`)

```typescript
// MainPage - the finish button
Image($r('app.media.finish'))
  .onClick(() => {
    try {
      this.uiContext.getComponentSnapshot().get('resultImage' /*图片组件绑定的id*/,
        (error: Error, pixelMap: image.PixelMap) => {
          if (pixelMap) {
            saveImage(pixelMap, this.context, this.uiContext);
          } else {
            this.uiContext.getPromptAction().showToast({ message: $r('app.string.save_error') });
          }
        }, { scale: Constants.ONE });
    } catch (error) {
      this.uiContext.getPromptAction().showToast({ message: $r('app.string.save_error') });
    }
  });
```

```typescript
// SaveImage.ets
export async function saveImage(pixelMap: PixelMap, context: common.UIAbilityContext, uiContext: UIContext) {
  const IMAGE_PACKER_API: image.ImagePacker = image.createImagePacker();
  let packOptions: image.PackingOption = { format: 'image/png', quality: 100 };   // PNG keeps the alpha
  try {
    const buffer: ArrayBuffer = await IMAGE_PACKER_API.packToData(pixelMap, packOptions);

    const path = context.filesDir + '/test.png';
    let file: fileIo.File | undefined = undefined;                 // FIX: sample uses `= null!`
    try {
      file = fs.openSync(path, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
      fs.writeSync(file.fd, buffer);
    } finally {
      if (file !== undefined) {                                    // FIX: sample closes null.fd
        fs.closeSync(file.fd);                                     //      when openSync threw
      }
    }

    const srcFileUris: Array<string> = [fileUri.getUriFromPath(path)];
    const photoCreationConfigs: Array<photoAccessHelper.PhotoCreationConfig> = [{
      title: 'test',
      fileNameExtension: 'png',
      photoType: photoAccessHelper.PhotoType.IMAGE,
      subtype: photoAccessHelper.PhotoSubtype.DEFAULT
    }];

    const desFileUris: Array<string> =
      await photoAccessHelper.getPhotoAccessHelper(context)
        .showAssetsCreationDialog(srcFileUris, photoCreationConfigs);  // FIX: .then with no .catch
    if (desFileUris.length === 0) {
      return;                                                      // user refused the dialog
    }

    let imageFile: fileIo.File | undefined = undefined;
    try {
      imageFile = fileIo.openSync(desFileUris[0], fileIo.OpenMode.CREATE | fileIo.OpenMode.READ_WRITE);
      fileIo.copyFileSync(path, imageFile.fd);
      uiContext.getPromptAction().showToast({ message: '保存成功' });
    } finally {
      if (imageFile !== undefined) {
        fileIo.closeSync(imageFile);                               // FIX: sample closes this fd twice
      }
    }
  } catch (error) {
    hilog.error(0x0000, 'testTag', `Failed to saveFile. Code: ${error.code}, message: ${error.message}`);
  } finally {
    IMAGE_PACKER_API.release();                                    // FIX: never released (HW-18-0036)
  }
}
```

**`showAssetsCreationDialog` is the permission-free write path**, and it is the
right one whenever the app already has the bytes on disk: the app names the
files it wants to add, the system shows a confirmation sheet, and the returned
URIs are pre-authorised destinations. There is no `WRITE_IMAGEVIDEO` anywhere
in this project, which is exactly right.

Three things about the shipped version are wrong in ways that only show up
after the happy path. The dialog promise has a `.then` and **no `.catch`**, so
refusing the sheet - a normal thing to do - produces an unhandled rejection.
The inner block closes `imageFile1.fd` inside the `try` *and* `imageFile1`
again in the `finally`, so the **success** path throws `EBADF` from the
`finally` after the toast has already been shown. And the outer `file` is
declared `let file: fileIo.File = null!` and closed unconditionally in its
`finally`, so if `openSync` throws, `closeSync(null.fd)` raises a `TypeError`
that replaces the real error. The rule that fixes all three: **one close per
descriptor, in a `finally`, guarded for undefined** (`HW-18-0034`).

`format: 'image/png'` is load-bearing and easy to get wrong by copying a save
helper from a sibling sample: the whole point of a cut-out is the alpha
channel, and `image/jpeg` silently flattens it onto black.

## Permissions & config

**None.** `module.json5` has no `requestPermissions` block, and the sample needs
none:

- `PhotoViewPicker` returns URIs with read access to exactly what the user
  selected.
- `showAssetsCreationDialog` returns URIs with write access to exactly what the
  user confirmed.
- `subjectSegmentation` is on-device; no network, no account.

`deviceTypes` is `phone`, `tablet`, `2in1`. `EntryAbility` publishes the
`windowStage` into `AppStorage`, which `MainPage` pulls back out to call
`setWindowSystemBarProperties` - a heavier route than `@StorageProp` on the
avoid areas, which the same page also uses.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **Subject segmentation is device-dependent.** `doSegmentation` rejects on
  devices without the capability; the sample maps that to
  `{ success: false }` and then never reads `success`, so the failure renders
  as a blank image area.
- `maxSubjects` is fixed at `1`, so `subjectDetails` never has more than one
  entry and the `MAX_SELECT: 20` constant is unused.
- The `analysis` string is computed on every run and never shown.
- **The saved file is the snapshot, not the source.** `getComponentSnapshot`
  captures the rendered `Image` at `scale: 1`, so the output is bounded by the
  on-screen size of an `85%`-height stack, not by the resolution of the picked
  photo. To save at full resolution, pack `result.foregroundImage` directly.
- The sandbox staging file is always `filesDir + '/test.png'`, so two saves in
  flight overwrite each other and the file is never cleaned up.
- `formatAnalysis` reads `data.fullSubject.subjectRectangle` without the
  optional chaining that `process` uses one line earlier for the same object.
- The picker is capped at one image; the add-another button replaces rather
  than adds.

## Pitfalls

- **`HW-18-0034`** (B/low, confirmed): `SaveImage` error handling is broken
  three ways - `showAssetsCreationDialog(...).then(...)` has no `.catch` so
  refusing the dialog is an unhandled rejection; `imageFile1.fd` is closed in
  the `try` and `imageFile1` again in the `finally`, so the success path throws
  `EBADF`; and the outer `let file: fileIo.File = null!` is closed in a
  `finally` even when `openSync` threw, masking the original error with a
  `TypeError`. Fix: one close per fd, in a `finally`, guarded for undefined,
  plus a `.catch` on the dialog chain.
- **`HW-18-0035`** (B/low, confirmed): a picker cancel leaves the page broken -
  the `URI === undefined` check can never fire for an empty array, so
  `loadImage` throws on `names[0]`, and both button handlers set
  `isSelected = true` *before* the pick, so after a cancel the cut-out button is
  active with no image and produces an empty result view. Fix: test
  `photoUris.length`, and set `isSelected` only after a successful load.
- **`HW-18-0036`** (B/low, confirmed): systematic - an `ImagePacker` is created
  per save and never released in eight photography samples, this one at
  `SaveImage.ets:26`. Each save leaks a native packer for the process
  lifetime. `CompressImages` releases correctly and is the reference. Fix:
  `imagePacker.release()` in a `finally`.
- **`HW-18-0005`** (B/low, confirmed): systematic - `ImageSource` instances are
  created and never released across eight photography files, this one in
  `ImagePickerUtils.loadImage`, which also leaks the fd it opened. Native
  decoder resources accumulate with every picked image. Fix:
  `imageSource.release()` and `closeSync` in a `finally`, after
  `createPixelMap` has resolved.
- **`HW-18-0002`** (E/low, confirmed): systematic - the doc's 工程目录 lists
  `EntryBackAbility.ets`, a truncation of the zip's `EntryBackupAbility.ets`.
  The same typo appears in `04_ratio_camera` (whose zip has no backup ability
  at all), `16_image_effect`, `11_news_reading/17_regular_highlight` and
  `15_utilities/34_pc_upload_image` - evidence the trees are hand-edited from a
  shared template rather than generated. Fix: regenerate the trees from the
  zips; drop the entry entirely for `ratio_camera`.
- **`result.success` is never checked** before rendering, so a segmentation
  failure is indistinguishable from an empty image.

## References

- `huawei_industry_tree/18_photography/docs/08_image_segmentation.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_segmentation-0000002282720108
- `documentation/harmonyos-references/07_ai/core-vision-subjectsegmentation-api.md` - `doSegmentation`, `SegmentationConfig`, `SegmentationResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/core-vision-subjectsegmentation-api
- `documentation/harmonyos-guides/08_ai/core-vision-subject-segmentation.md` - the subject-segmentation guide
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/core-vision-subject-segmentation
- `documentation/harmonyos-guides/05_media/photoaccesshelper-photoviewpicker.md` - the permission-free picker and its empty-array cancel
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `showAssetsCreationDialog`, `PhotoCreationConfig`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-guides/05_media/image-decoding-arts.md` - `createImageSource` / `createPixelMap` and releasing the source
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-decoding-arts
- `documentation/harmonyos-guides/05_media/image-encoding-arts.md` - `ImagePacker`, `packToData`, PNG versus JPEG and the alpha channel
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-encoding-arts
- `PHOTO-05` - the other gallery-write path in this industry (`SaveButton` + `createAsset`)
- `PHOTO-10` - the same component-snapshot save, over a sticker overlay
