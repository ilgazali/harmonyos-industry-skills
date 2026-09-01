---
id: PHOTO-09
title: Quality-tiered image compression - packToFile into the sandbox, SaveButton back to the album
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/09_compress_images.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/compress_images-0000002322173825
sample: huawei_industry_tree/18_photography/downloads/CompressImages.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [common, dataSharePredicates, fileIo, fileUri, fs, hilog, image, photoAccessHelper, systemDateTime, window]
permissions: [ohos.permission.READ_IMAGEVIDEO, ohos.permission.WRITE_IMAGEVIDEO]
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0003, HW-18-0004, HW-18-0036, HW-18-0037, HW-18-0038, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when the app has to **shrink images the user picked, on device,
before they are uploaded or shared** - a chat attachment, an insurance claim
photo, a marketplace listing. The pattern is a three-stop pipeline: pick from
the gallery with `PhotoViewPicker`, re-encode each picture with
`ImagePacker.packToFile` at a chosen quality into the app sandbox, then hand
the sandbox file back to the album with a `SaveButton`.

The transferable idea is that **the sandbox is the workbench**. Compression
output never goes straight to the album; it lands in `filesDir` under a
timestamped name, where the app can list it, measure it, show it, delete it and
re-do it. Only when the user explicitly presses a security control does one
file cross back into the media library. That keeps the whole editing loop
permission-free and reversible, and it is the right shape for any "process then
maybe keep" media flow - watermarking, format conversion, cropping.

**Read `HW-18-0003` and `HW-18-0004` before adopting it.** The sample takes a
detour through `photoAccessHelper.getAssets` just to build thumbnails, and
declares two restricted album permissions it never requests and does not need.
Both are copy-paste hazards, not design choices.

## Feature checklist

- A quality selector with three capsule buttons - 高 / 中 / 低 - only one
  highlighted at a time.
- A gallery icon in the navigation menu opens the system picker, up to nine
  images, images only.
- Selected images appear as a three-column thumbnail grid.
- The start button is disabled until images are selected and the previous batch
  has finished.
- Starting compression re-encodes every selected picture into the sandbox as
  JPEG at the chosen quality, and toasts on completion.
- Before writing, the sandbox is pruned so it never holds more than 50 files;
  the oldest are unlinked first.
- A second button opens a list page showing every compressed file with its
  name and its human-readable size (B / KB / MB).
- Each row carries a `SaveButton` that copies that file into the album.

## Architecture

One `entry` module, three page files. There is no view-model layer: the page
holds the state, a singleton utility holds the codec call.

```
entry/src/main/ets
├── entryability/EntryAbility.ets   boilerplate
├── entrybackupability/
└── pages
    ├── Index.ets            @Entry: picker, quality selector, thumbnail grid, sandbox pruning
    ├── PictureProc.ets      PictureUtils singleton: the one compressImg() call
    └── localZipFile.ets     NavDestination: sandbox file list + per-row SaveButton
```

The documented tree matches the zip (the document spells the backup directory
`entrybackupablility`; the zip has `entrybackupability` - a typo in the
document, not a structural difference).

**The design decision worth copying** is that compression is routed through a
`Navigation` / `NavDestination` pair rather than two peer pages. `Index` owns a
`NavPathStack` and pushes `'localZipFile'` by name through the router map; the
results page is a `@Component` with a `NavDestination` root and an `@Builder`
export, so it never needs its own entry point. That matters because the results
page is a *view onto the sandbox*, not a separate feature: it reads
`filesDir` fresh in `aboutToAppear`, so it is always in step with whatever the
previous compression run produced, with no state passed between them.

The second decision worth copying is `PictureUtils` being a singleton with one
method that takes the `Context`. It is stateless, so nothing about the current
batch lives inside it, and the same call serves nine parallel invocations
without interference.

One thing **not** to copy: the thumbnail grid's `ForEach(this.thnPictures,
(uri: string) => {...})` types its item as `string` while `thnPictures` is
`Array<PixelMap>`, and the arrow has no key generator. It renders because
`Image()` accepts a `PixelMap`, but the annotation is a lie and the missing key
makes the grid re-render wholesale.

## Implementation steps

1. **Pick with `PhotoViewPicker`**, `MIMEType = IMAGE_TYPE` and
   `maxSelectNumber = 9`. Clear both arrays before the picker opens so a second
   selection replaces the first rather than appending to it.
2. **Build thumbnails from the picker URI directly** with `fileIo` +
   `image.createImageSource`. Do not go through `phAccessHelper.getAssets` /
   `getThumbnail`: that path needs `READ_IMAGEVIDEO`, a restricted permission
   this sample declares but never requests, so the query fails at runtime
   (`HW-18-0003`).
3. **Do not declare `READ_IMAGEVIDEO` / `WRITE_IMAGEVIDEO` at all.** The picker
   and the `SaveButton` are both permission-free; the two declarations in
   `module.json5` are a shared-template artefact and will fail app review
   (`HW-18-0004`).
4. **Seed the quality state from the selected mode.** `zipMod` starts at
   `'high'` but `zipQuality` starts at `0`, so a user who never touches the
   selector compresses at the worst possible quality (`HW-18-0037`).
5. **Prune the sandbox before writing,** not after: `fs.listFileSync(...).sort()`
   and unlink as many of the oldest as the incoming batch size.
6. **Compress through one stateless helper.** Open the source URI read-only,
   read it into an `ArrayBuffer`, `createImageSource(buf)`, open a timestamped
   target in `filesDir`, `packToFile(source, fd, { format, quality })`.
7. **Release the packer in `finally` and close both files in `finally`.**
   `CompressImages` is the one photography sample that does this correctly and
   is cited as the reference pattern in `HW-18-0036`.
8. **Track batch completion with `Promise.all`, not a timer.** The shipped
   500 ms `setTimeout` re-enables the start button mid-batch (`HW-18-0038`).
9. **Save with `SaveButton` + `createImageAssetRequest` + `applyChanges`,**
   inside the click handler and only when
   `result === SaveButtonOnClickResult.SUCCESS`.

## Verified snippets

All snippets are from `CompressImages.zip`. Corrected forms are marked.

**Selecting and thumbnailing — `entry/src/main/ets/pages/Index.ets`**
(corrected, see `HW-18-0003`)

```typescript
async selectPicture() {
  this.thnPictures = [];
  this.pictures = [];
  try {
    let photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
    photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
    photoSelectOptions.maxSelectNumber = 9;
    let photoPicker = new photoAccessHelper.PhotoViewPicker();
    photoPicker.select(photoSelectOptions).then(async (photoSelectResult: photoAccessHelper.PhotoSelectResult) => {
      const uris: Array<string> = photoSelectResult.photoUris;
      for (const uri of uris) {
        this.pictures.push(uri);
        // FIX: the sample builds a DataSharePredicates query here and calls
        // phAccessHelper.getAssets(fetchOptions, ...) -> getThumbnail(...),
        // which requires READ_IMAGEVIDEO. Read the picker URI directly instead.
        const file = fs.openSync(uri, fs.OpenMode.READ_ONLY);
        try {
          const size = fs.statSync(file.fd).size;
          const buf = new ArrayBuffer(size);
          fs.readSync(file.fd, buf);
          const source = image.createImageSource(buf);
          this.thnPictures.push(await source.createPixelMap({ editable: false }));
          this.canZip = true;
        } finally {
          fs.closeSync(file);
        }
      }
    }).catch((err: BusinessError) => {
      hilog.error(MY_TAG, 'testTag', 'photoPicker.select failed with err: ' + JSON.stringify(err));
    });
  } catch (error) {
    let err: BusinessError = error as BusinessError;
    hilog.error(MY_TAG, 'testTag', 'photoPicker failed with err: ' + JSON.stringify(err));
  }
}
```

**The URI the picker returns is already readable — that is the whole point of
the picker.** The temporary read grant it hands back is what makes the flow
permission-free, and the corrected form above is not new code: it is exactly
the open/stat/read/`createImageSource` idiom this same project already ships in
`PictureProc.compressImg`. The shipped version instead re-queries the media
library by URI (`predicates.equalTo('uri', uri)`) purely to reach
`PhotoAsset.getThumbnail`, which the reference documents as requiring
`ohos.permission.READ_IMAGEVIDEO` - a restricted permission that is declared in
`module.json5` but never passed to `requestPermissionsFromUser`, so the callback
lands in its error branch and no thumbnail ever arrives.

Note also that the shipped code nests an async callback inside `uris.forEach`,
so `thnPictures` fills in completion order while `pictures` fills in selection
order. The two arrays are indexed independently by the UI, so they only *look*
aligned. The `for...of` above keeps them in step.

**The codec call — `entry/src/main/ets/pages/PictureProc.ets`** (as shipped)

```typescript
public async compressImg(context: Context, pictureUri: string, format: string, quality: number) {
  let file: fs.File | null = null;
  let fileLoc: fs.File | null = null;
  try {
    file = fs.openSync(pictureUri, fs.OpenMode.READ_ONLY);
    let size = fs.statSync(file.fd).size;
    let buf = new ArrayBuffer(size);
    fs.readSync(file.fd, buf);
    let imageSource = image.createImageSource(buf);
    let path: string = context.getApplicationContext().filesDir + '/' + systemDateTime.getTime() + '_picture.jpg';
    fileLoc = fs.openSync(path, fs.OpenMode.CREATE | fs.OpenMode.READ_WRITE);
    let packOpts: image.PackingOption = { format: format, quality: quality };
    let imagePackerApi: image.ImagePacker = image.createImagePacker();
    await imagePackerApi.packToFile(imageSource, fileLoc.fd, packOpts).then(() => {
      hilog.info(MY_TAG, 'testTag', 'Succeeded in packing the image to file.');
    }).catch((error: BusinessError) => {
      hilog.error(MY_TAG, 'testTag', 'Failed to pack the image. And the error code:' + error.code +
        ', msg is: ' + error.message + ', file size:' + size);
    }).finally(() => {
      imagePackerApi.release();
    });
  } catch (error) {
    let err: BusinessError = error as BusinessError;
    hilog.error(MY_TAG, 'testTag', 'compressImg failed with err: ' + JSON.stringify(err));
  } finally {
    if (file != null) {
      fs.closeSync(file);
    }
    if (fileLoc !== null) {
      fs.closeSync(fileLoc.fd);
    }
  }
}
```

**Three details carry the design.** `createImageSource(buf)` takes a buffer,
not the fd - so the source stays valid regardless of when the file handle is
closed, which is precisely the defect `HW-18-0025` describes in the sibling
samples that pass an fd and close it immediately. The filename is
`systemDateTime.getTime() + '_picture.jpg'`, which both guarantees uniqueness
and makes a plain lexical `sort()` of `filesDir` an oldest-first ordering - the
pruning logic in `checkSandBox` depends on exactly that. And
`imagePackerApi.release()` sits in `.finally()`, so the native packer is freed
on the error path too; `HW-18-0036` records that seven other photography
samples leak this instance and names this file as the pattern they should have
followed.

`quality` is the JPEG quality factor, 0-100, higher meaning larger. The sample
maps 高/中/低 to 40 / 60 / 80, so the "low quality" button produces the
*largest* file. That is intentional in the UI's own terms - the label describes
compression strength, not output fidelity - but it is worth naming, because it
is the reason the uninitialised default in `HW-18-0037` is dangerous rather
than merely wrong: `0` sits below all three tiers.

**Starting a batch — `entry/src/main/ets/pages/Index.ets`**
(corrected, see `HW-18-0037`, `HW-18-0038`)

```typescript
@State zipMod: string = 'high';
@State zipQuality: number = 40;        // FIX: shipped value is 0, contradicting zipMod

Button($r('app.string.startZip'), { stateEffect: true, type: ButtonType.Capsule })
  .width(150).enabled(this.isLastZipFin === true && this.canZip === true)
  .onClick(() => {
    // 检查沙箱是否文件过多  (check whether the sandbox holds too many files)
    this.checkSandBox(this.pictures.length);
    this.isLastZipFin = false;
    // FIX: the sample fires these without collecting them and re-enables the
    // button from a fixed 500 ms setTimeout.
    Promise.all(this.pictures.map((uri: string) =>
      PictureUtils.getInstance().compressImg(this.context, uri, 'image/jpeg', this.zipQuality)
    )).then(() => {
      this.isLastZipFin = true;
      this.promptAction.showToast({ message: $r('app.string.finish'), duration: 1000 });
    });
  });
```

**`enabled(this.isLastZipFin && this.canZip)` is the only guard against
overlapping batches,** which is why the flag must track real completion. Nine
full-resolution photos take well over half a second to re-encode; the shipped
timer releases the button in the middle of the run, a second tap calls
`checkSandBox` again (unlinking files the first batch is still writing) and
starts a duplicate batch, and the user gets two toasts per image.

The `checkSandBox(this.pictures.length)` call before, not after, the batch is
the right order: it makes room for exactly the number of files about to be
written, so the 50-file ceiling holds without ever deleting an output that has
just been produced.

**Handing a sandbox file to the album — `entry/src/main/ets/pages/localZipFile.ets`**
(as shipped)

```typescript
SaveButton({ text: SaveDescription.DOWNLOAD, buttonType: ButtonType.Capsule })
  .onClick(async (event, result: SaveButtonOnClickResult) => {
    if (result === SaveButtonOnClickResult.SUCCESS) {
      try {
        let context = this.getUIContext().getHostContext();
        let phAccessHelper = photoAccessHelper.getPhotoAccessHelper(context);
        // 需要确保fileUri对应的资源存在。 (the resource behind fileUri must exist)
        let assetChangeRequest: photoAccessHelper.MediaAssetChangeRequest =
          photoAccessHelper.MediaAssetChangeRequest.createImageAssetRequest(context, item.uri);
        await phAccessHelper.applyChanges(assetChangeRequest);
        hilog.info(MY_TAG, 'testTag',
          'createAsset successfully, uri: ' + assetChangeRequest.getAsset().uri);
        this.promptAction.showToast({ message: 'download success', duration: 1000 });
      } catch (err) {
        hilog.error(MY_TAG, 'testTag', `create asset failed with error: ${err.code}, ${err.message}`);
        this.promptAction.showToast({ message: 'download failed', duration: 1000 });
      }
    } else {
      hilog.error(MY_TAG, 'testTag', 'SaveButtonOnClickResult create asset failed');
    }
  })
```

**This is the permission-free save, and the shape is not optional.**
`createImageAssetRequest` takes a *file URI in the app sandbox* and copies it
into the album; the temporary write grant is issued by the security component
itself, and only inside this click handler. Checking
`result === SaveButtonOnClickResult.SUCCESS` before doing anything is what
distinguishes a real user press from a programmatic click - a
`SaveButton` that skips that check has no security value.

The list page reaches the sandbox files through `fileUri.getUriFromPath`, and
`loadFiles` skips zero-byte entries (`if (stat.size !== 0)`), which quietly
hides the placeholder files a failed `packToFile` leaves behind.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.READ_IMAGEVIDEO",  "reason": "$string:readImgReason",  "usedScene": { "abilities": [] } },
  { "name": "ohos.permission.WRITE_IMAGEVIDEO", "reason": "$string:writeImgReason", "usedScene": { "abilities": [] } }
]
```

Both entries should be **deleted** (`HW-18-0004`). They are restricted (ACL)
permissions an ordinary app cannot ship; nothing in the sample requests them at
runtime; and both jobs they would do - reading picked images and writing to the
album - are already done permission-free by `PhotoViewPicker` and `SaveButton`.
The empty `"abilities": []` in `usedScene` is itself a tell that these were
pasted in rather than reasoned about.

`routerMap: "$profile:route_map"` is required, since `localZipFile` is reached
by `pushPathByName` rather than by a page path.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `packToFile` re-encodes; it does not resize. A 12 MP photo at quality 40 is
  still 12 MP. If the goal is upload size, pair the quality option with
  `desiredSize` on `createPixelMap`.
- `format` is passed as a string (`'image/jpeg'`); the supported set is jpg,
  webp and png. PNG is lossless, so `quality` has no effect there.
- The sandbox ceiling is a hardcoded `maxSaveImg = 50` and the pruning is
  purely lexical over the timestamped names.
- The thumbnail `ForEach` has no key generator, so the grid rebuilds entirely
  on every push.

## Pitfalls

- **`HW-18-0003`** (B/medium, confirmed): `getAssets` is called on the
  picker-returned URI with `READ_IMAGEVIDEO` declared but never requested at
  runtime, so the thumbnail query fails and the grid stays empty; the same
  pattern appears in `VideoWatermark`'s `FileUtil.ets`. Fix: read the picker URI
  with `fs` + `image.createImageSource` instead of querying the media library.
- **`HW-18-0004`** (D/medium, confirmed): the module declares restricted
  `READ_IMAGEVIDEO` and `WRITE_IMAGEVIDEO` although the code uses only the
  permission-free picker and `SaveButton` flows - one of nine photography
  samples sharing this `module.json5` template. Fix: delete both
  `requestPermissions` entries.
- **`HW-18-0037`** (B/medium, confirmed): `zipMod` defaults to `'high'` but
  `zipQuality` defaults to `0`, so the pre-highlighted mode compresses at
  maximum loss until the user taps a button. Fix: initialise `zipQuality` to
  the high-mode value, 40.
- **`HW-18-0038`** (B/low, confirmed): `isLastZipFin` is reset by a fixed
  500 ms `setTimeout` rather than by the compression promises, so the start
  button re-enables mid-batch and a second tap starts an overlapping run
  (duplicate outputs, duplicate toasts, extra deletions). Fix:
  `Promise.all(...)` over the `compressImg` promises.
- **`HW-18-0036`** (B/low, confirmed): eight photography samples create an
  `ImagePacker` per save and never release it. This sample is the exception and
  is cited in the finding as the reference implementation - keep the
  `.finally(() => imagePackerApi.release())` if you copy `PictureProc.ets`
  anywhere.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-image-imagepacker.md` - `packToFile`, `PackingOption`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagepacker
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagesource.md` - `createImageSource`, `createPixelMap`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagesource
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` - `PhotoSelectOptions`, `select`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-absalbum.md` - `getAssets` and its `READ_IMAGEVIDEO` requirement
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-absalbum
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoasset.md` - `getThumbnail`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoasset
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-mediaassetchangerequest.md` - `createImageAssetRequest`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-mediaassetchangerequest
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `applyChanges`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-references/02_application-framework/ts-security-components-savebutton.md` - `SaveButtonOnClickResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-savebutton
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `listFileSync`, `statSync`, `unlinkSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/02_application-framework/js-apis-data-datasharepredicates.md` - the predicate API the shipped thumbnail path uses
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-datasharepredicates
- `PHOTO-10` - the same `SaveButton` save, from a composed pixel map rather than a sandbox file
- `PHOTO-12` - the same picker-to-sandbox-to-`SaveButton` pipeline for GIF output
