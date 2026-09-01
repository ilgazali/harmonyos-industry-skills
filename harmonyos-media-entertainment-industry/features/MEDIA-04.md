---
id: MEDIA-04
title: Video compression - pick, compress at three quality presets, compare, and save back to the gallery
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/04_video_compress_demo.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_compress_demo-0000002248722292
sample: huawei_industry_tree/13_media_entertainment/downloads/VideoCompressDemo.zip
kits: ["@kit.MediaKit", "@kit.MediaLibraryKit", "@kit.ArkUI", "@kit.AbilityKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit", "@ohos/videocompressor"]
apis: ["photoAccessHelper.PhotoViewPicker", PhotoSelectOptions, "photoAccessHelper.getPhotoAccessHelper", showAssetsCreationDialog, PhotoCreationConfig, "fs.openSync", "fs.statSync", "fs.readSync", "fs.writeSync", "fs.closeSync", "fileUri.getUriFromPath", "media.createAVImageGenerator", fetchFrameByTime, AVImageQueryOptions, "VideoCompressor.compressVideo", CompressQuality, Video, VideoController, LoadingProgress, "UIContext.showAlertDialog", "UIContext.getPromptAction", "@StorageProp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-13-0014, HW-13-0015, HW-13-0016, HW-13-0097]
status: verified-with-fixes
---

## When to use

Load this card when an app has to **shrink a user's video before it leaves the
device** - an upload with a size cap, a chat attachment, a post, a backup.
The flow it demonstrates is the complete one: pick from the gallery, compress
into the sandbox at a chosen quality, show old and new side by side, then hand
the result back to the gallery through the system save dialog.

Two pieces generalise beyond video. The first is that **compression output
belongs in the sandbox, not in the gallery** - the app writes freely to its
own `cacheDir`, previews from there, and only asks for a gallery slot when the
user presses save. The second is `showAssetsCreationDialog`, which is the
permission-free way to create a gallery item: the user approves the specific
file in a system sheet, and the app gets back a writable URI. No
`WRITE_IMAGEVIDEO`, no `requestPermissionsFromUser` - this sample declares no
permissions at all, which is the model to copy. Contrast `MEDIA-03`, which
declares one and still cannot read anything.

The compression itself is delegated to the OpenHarmony `@ohos/videocompressor`
package rather than hand-rolled over `AVTranscoder`, which is the right call
for a demo and a decision to revisit for production.

## Feature checklist

- A dark full-screen page with an add button that opens the video picker.
- The picked video appears in a `Video` component with a generated thumbnail
  as `previewUri` and a custom play overlay; the default controls are off.
- Three circular quality buttons - 高 (high), 中 (medium), 低 (low) - each
  swapping to a selected icon.
- Pressing one shows a full-screen loading mask with 压缩中 (compressing) and
  runs the compression.
- When it finishes, a second `Video` appears below the first, playing the
  compressed file from the sandbox, with its byte size formatted underneath.
- An alert confirms completion; the mask fades out under an `animateTo`.
- 保存 (save) opens the system asset-creation dialog and writes the compressed
  buffer into the approved gallery item, then reports success or failure.
- 取消 (cancel) raises a demo-only toast.
- Both videos are tap-to-play/pause through their own `VideoController`.

## Architecture

One `entry` module, six ArkTS files, one external dependency.

```
entry/src/main/ets
├── common/CommonConstant.ets      byte dividers, labels, quality tags, durations
├── entryability/EntryAbility.ets  full-screen window, avoid areas -> AppStorage
├── model/DataModel.ets            CompressSettingItem, COMPRESS_SETTINGS, FileProcessResult
├── pages/CompressPage.ets         @Entry, four @Builders, all the state
└── utils/
    ├── CommonUtil.ets             fileSizeConversion + showCommonTip
    ├── CompressUtil.ets           wraps @ohos/videocompressor; reads the output back as a buffer
    └── PictureUtils.ets           picker, gallery save, thumbnail - the three fs-touching operations
```

The documented tree matches the zip. `oh-package.json5` pins
`"@ohos/videocompressor": "^1.0.5"` at both project and module level.

**The design decision worth copying** is that every file descriptor in the
project lives in `PictureUtils` or `CompressUtil`, never in the page.
`CompressPage` holds `videoPath`, `fileDir`, two sizes, a `buffer` and some
booleans - all plain values - and each utility method opens what it needs,
returns a value, and closes in a `finally`. That is what keeps a 268-line page
readable and makes the fd bugs below local to two functions instead of spread
through the UI.

The quality presets being **data** (`COMPRESS_SETTINGS`, an array of
`CompressSettingItem` carrying title, icon, selected icon, `CompressQuality`,
tag and colour) is the second half of it: the three buttons are one `ForEach`
over that array, and adding a fourth preset is one array entry.

`FileProcessResult { buffer?, filePath, fileSize }` is deliberately shared
between "the video you picked" (no buffer) and "the video we produced"
(buffer), which is why the optional field is there.

## Implementation steps

1. **Model the presets as data**, not as three copy-pasted buttons, and key
   the `ForEach` on tag plus index.
2. **Pick with `PhotoViewPicker`** filtered to `VIDEO_TYPE`. This needs no
   permission - the sample's own comment says so.
3. **Open the picked URI to measure it.** `fs.openSync(uri, READ_ONLY)` then
   `fs.statSync(file.fd).size`; assign the handle to the outer variable so the
   `finally` can close it (`HW-13-0014`).
4. **Generate the thumbnail with `AVImageGenerator`** on the same URI, and
   `await` its `release()` before closing the fd it was reading
   (`HW-13-0015`).
5. **Compress into the sandbox** via `videoCompressor.compressVideo(context, path, quality)`
   and check `data.code === CompressorResponseCode.SUCCESS` before touching
   `data.outputPath`.
6. **Read the output back into an `ArrayBuffer`** only if you intend to write
   it somewhere else - and be aware this materialises the whole video in
   memory (see Constraints).
7. **Preview from the sandbox** with `src: file://${path}` on a `Video`, using
   the earlier thumbnail as `previewUri`.
8. **Save with `showAssetsCreationDialog`**, passing the source URI and a
   `PhotoCreationConfig` whose `fileNameExtension` is split off the real file
   name; write the buffer into the returned URI.
9. **Guard each handle separately in `finally`** - `if (originFile)` and
   `if (destFile)` - so a refused dialog cannot double-fault (`HW-13-0039`).
10. **Report the sizes with one formatter** whose thresholds and divisors are
    both binary.
11. **Keep the doc and the interface in step**: the published snippet reads
    fields that do not exist (`HW-13-0016`).

## Verified snippets

All snippets are from `VideoCompressDemo.zip`. Corrected forms are marked.

**Picking and measuring - `entry/src/main/ets/utils/PictureUtils.ets`** (corrected, see `HW-13-0014`)

```typescript
static async selectVideo(selectNumber: number) {
  let originFile: fs.File | undefined = undefined;
  let result: FileProcessResult | undefined = undefined;
  try {
    let photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
    // 设置要选择的媒体文件类型
    photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.VIDEO_TYPE;
    // 设置选择文件最大数量
    photoSelectOptions.maxSelectNumber = selectNumber;
    let photoPicker = new photoAccessHelper.PhotoViewPicker();
    // PhotoViewPicker的select方法不需要特殊权限即可读取到图片或者视频文件
    let selectResult: photoAccessHelper.PhotoSelectResult = await photoPicker.select(photoSelectOptions);
    if (!selectResult || selectResult.photoUris.length <= 0) {
      return undefined;
    }
    // 这里拿到可访问uri后，需要打开该视频文件，才能够获取到源文件大小
    let originFilePath = selectResult.photoUris[0];
    originFile = fs.openSync(originFilePath, fs.OpenMode.READ_ONLY);   // FIX: shipped code re-declares with `let`
    let size = fs.statSync(originFile.fd).size;
    result = { filePath: originFilePath, fileSize: size } as FileProcessResult;
  } catch (e) {
    hilog.error(0x0000, 'testTag', 'Read video from albums error: %{public}d %{public}s', e.code, e.message);
  } finally {
    if (originFile) {
      fs.closeSync(originFile);
    }
  }
  return result;
}
```

**One `let` makes the cleanup dead code.** The shipped line inside the `try` is
`let originFile = fs.openSync(...)`, which declares a *second*, block-scoped
`originFile` shadowing the one at the top. The `finally` tests the outer
variable, which is still `undefined`, so it closes nothing - one descriptor
leaked on every video the user selects, in the demo's very first step. Removing
the inner `let` is the whole fix, and the shape it restores - declare outside,
assign inside, guard in `finally` - is the pattern the two sibling functions in
this file already use correctly.

The comment on the picker line is worth keeping in your own code: `select`
needs no special permission, and the URI it returns is readable by `fs`
immediately. That is what makes this sample's empty `requestPermissions` array
correct rather than an oversight.

**The thumbnail, and the fd race - same file** (corrected, see `HW-13-0015`)

```typescript
static async getVideoThumbnails(videoPath: string) {
  let pixelMap: PixelMap | undefined = undefined;
  let videoFile: fs.File | undefined = undefined;
  try {
    // 创建AVImageGenerator对象。
    let avImageGenerator: media.AVImageGenerator = await media.createAVImageGenerator();
    videoFile = fs.openSync(videoPath, fs.OpenMode.READ_ONLY);
    // 设置fdSrc。
    avImageGenerator.fdSrc = { fd: videoFile.fd };

    // 初始化入参。
    let timeUs = 0;
    let queryOption = media.AVImageQueryOptions.AV_IMAGE_QUERY_NEXT_SYNC;
    let param: media.PixelMapParams = {};

    // 获取缩略图（promise模式）。
    pixelMap = await avImageGenerator.fetchFrameByTime(timeUs, queryOption, param);

    // 释放资源（promise模式）。
    await avImageGenerator.release();     // FIX: shipped code does not await the release
  } catch (e) {
    hilog.error(0x0000, 'testTag', `Get video thumbnails to albums error: %{public}d %{public}s`, e.code, e.message);
  } finally {
    if (videoFile) {
      fs.closeSync(videoFile);            // now provably after the generator let the fd go
    }
  }
  return pixelMap;
}
```

**`release()` returns a promise, and `finally` does not wait for it.** In the
shipped code the `finally` closes the descriptor the generator may still be
tearing down, so the thumbnail fails intermittently depending on timing - the
kind of defect that passes every manual test and shows up in the field. One
`await` fixes it. This is `HW-13-0015`, and it is systematic: the same
close-during-async-use pattern appears in five media samples, over
`AVImageGenerator`, `packToFile`, `startVibration`, `SoundPool.load` and
`createPixelMap`.

`AV_IMAGE_QUERY_NEXT_SYNC` at `timeUs = 0` means "the first decodable frame at
or after position zero" - the right query for a poster frame, because the
exact frame at 0 is often not a keyframe. An empty `PixelMapParams` returns
the frame at its native size; pass `width`/`height` if the target is a small
preview.

Note also that the generator is created *before* the file is opened, so a
`createAVImageGenerator` failure leaves nothing to clean up - and that the
whole function returns `undefined` rather than throwing, which is why
`CompressPage` can assign it straight into `thumbnails: PixelMap | undefined`.

**Saving back to the gallery - same file** (as shipped)

```typescript
static async saveVideoToAlbums(ctx: Context | undefined, quality: string, compressPath: string, buffer: ArrayBuffer) {
  let originFile: fs.File | undefined = undefined;
  let destFile: fs.File | undefined = undefined;
  let result: boolean = false;
  if (!ctx) {
    return result;
  }
  try {
    // 获取文件uri
    let uri = fileUri.getUriFromPath(compressPath);
    originFile = fs.openSync(compressPath);
    let wholeName: string = originFile.name;
    let split = wholeName.split('.');

    let accessHelper = photoAccessHelper.getPhotoAccessHelper(ctx);
    // 设置保存到相册中的文件的基本信息
    let creationConfigs: photoAccessHelper.PhotoCreationConfig[] = [
      {
        title: `${split[0]}-${quality}-QUALITY`,
        fileNameExtension: split[split.length - 1],
        photoType: photoAccessHelper.PhotoType.VIDEO,
        subtype: photoAccessHelper.PhotoSubtype.DEFAULT
      }
    ];
    // 向用户申请保存视频至相册的权限
    let desFileUris = await accessHelper.showAssetsCreationDialog([uri], creationConfigs);
    if (desFileUris && desFileUris.length > 0) {
      destFile = fs.openSync(desFileUris[0], fs.OpenMode.READ_WRITE);
      fs.writeSync(destFile.fd, buffer);
      result = true;
    }
  } catch (e) {
    hilog.error(0x0000, 'testTag', 'Save compressed video to albums error: %{public}d %{public}s', e.code, e.message);
  } finally {
    if (originFile) {
      fs.closeSync(originFile);
    }
    if (destFile) {
      fs.closeSync(destFile);
    }
  }
  return result;
}
```

**This is the function to copy verbatim.** `showAssetsCreationDialog` is the
permission-free gallery write: the app proposes a file (`uri`) and a
`PhotoCreationConfig`, the system shows the user a sheet, and on approval
returns a URI the app may write to - once. Refusal returns an empty array,
which the `if` handles by leaving `result` false and letting the caller show
the failure alert.

**Both handles are guarded separately in `finally`,** and that is not a
formality: `HW-13-0039` records five sibling samples where refusing this exact
dialog throws a `TypeError` out of the cleanup block, because they call
`closeSync` on an undefined destination file and mask the real outcome. Two
independent `if`s also mean a failure to open the destination still closes the
source.

`title` and `fileNameExtension` are split off the *real* file name rather than
guessed, so a `.mp4` output lands as `.mp4`; `PhotoType.VIDEO` with
`PhotoSubtype.DEFAULT` is what files it under videos rather than as a moving
photo or burst.

**Driving it from the page - `entry/src/main/ets/pages/CompressPage.ets`** (as shipped)

```typescript
.onClick(() => {
  if (this.videoPath === '') {
    return;
  }
  this.quantity = item.qualityTag;
  this.selectedQuality = item.qualityTag;
  this.isLoading = true;
  CompressUtil.compressVideo(this.uiContext.getHostContext() as Context, item.quality, item.qualityTag,
    this.videoPath)
    .then((res) => {
      if (res) {
        this.fileDir = res.filePath;            // doc says res.compressPath - HW-13-0016
        this.buffer = res.buffer;
        this.compressedSize = res.fileSize;     // doc says res.compressSize
        this.uiContext.showAlertDialog({ message: $r('app.string.compression_completion_tip') });
      }
      this.uiContext.animateTo({ duration: CommonConstant.LOADING_ANIMATION_DURATION, curve: Curve.EaseOut },
        () => {
          this.isLoading = false;
        });
    });
})
```

**The loading mask is a state flag, not a dialog,** which is why it can be
faded out inside `animateTo` while the alert is already up. Note the ordering:
`isLoading` is set true synchronously before the promise starts, and cleared in
`then` **outside** the `if (res)` - so a failed compression still dismisses the
mask. That is the detail most implementations get wrong.

`this.uiContext.getHostContext() as Context` is the sanctioned way to reach the
ability context from a page (`uiContext` itself is initialised from
`this.getUIContext()` at the top of the struct). `HW-13-0032` records four
sibling samples that write `new UIContext()` instead and end up with an
instance that has no host at all.

The two field names commented above are `HW-13-0016`: the document's 实现思路
snippet reads `res.compressPath` and `res.compressSize`, which do not exist on
`FileProcessResult` - a reader copying the published snippet gets compile
errors on both lines.

## Permissions & config

**None.** `module.json5` has no `requestPermissions` array at all, and that is
correct rather than an omission:

- `PhotoViewPicker.select` grants read access to the selected item by the act
  of selection.
- Compression writes into the app's own sandbox, which needs no permission.
- `showAssetsCreationDialog` obtains write access to one gallery item per
  approval, replacing `ohos.permission.WRITE_IMAGEVIDEO`.

`extensionAbilities` is present but empty. `deviceTypes` is `phone`, `tablet`,
`2in1`. The only external dependency is `@ohos/videocompressor ^1.0.5`,
declared in both `oh-package.json5` files; release builds enable obfuscation
via `obfuscation-rules.txt`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`,
  matching the document.
- **The compressed video is read entirely into an `ArrayBuffer`.**
  `CompressUtil.getVideo` allocates `new ArrayBuffer(videoSize)` and
  `fs.readSync`s the whole file, and the page holds it until save. A
  several-hundred-megabyte clip will exhaust the ArkTS heap. Streaming the
  sandbox file into the destination URI in chunks - or passing the path to
  `showAssetsCreationDialog` and letting the system copy - avoids the
  allocation entirely.
- Nothing deletes the sandbox output. Each compression leaves a file behind;
  a real app clears `cacheDir` or removes the previous output on the next run.
- The quality mapping is whatever `@ohos/videocompressor` implements behind
  `CompressQuality.COMPRESS_QUALITY_HIGH|MEDIUM|LOW`; there is no bitrate,
  resolution or codec control here. For explicit control use `AVTranscoder`.
- There is no progress: `compressVideo` resolves once, so the mask is
  indeterminate however long the clip is.
- 取消 (cancel) is a toast. There is no way to abort a running compression.
- `fileSizeConversion` handles B/KB/MB/GB and then falls back to raw bytes
  above 1 TB - harmless here, but note it is the *correct* binary-consistent
  formatter, unlike `MEDIA-03`'s (`HW-13-0013`).

## Pitfalls

- **`HW-13-0014` (B/medium, confirmed) - `selectVideo` shadows `originFile`
  with an inner `let`,** so the `finally` close is dead code and one file
  descriptor leaks per selection. Fix: drop the inner `let` and assign to the
  outer variable.
- **`HW-13-0015` (B/medium, probable) - the fd is closed while an async
  consumer is still using it.** `getVideoThumbnails` calls
  `avImageGenerator.release()` without awaiting, then closes the descriptor in
  `finally`, so thumbnails fail intermittently. Fix: `await` the release (or
  close in the completion path). Systematic across five media samples.
- **`HW-13-0016` (D/low, confirmed) - the document's snippet reads
  `res.compressPath` and `res.compressSize`,** fields that do not exist on
  `FileProcessResult` (`filePath` / `fileSize`). Copying the published code
  does not compile. Fix: update the document.
- **Memory: the whole compressed video is buffered in the ArkTS heap** before
  it is written to the gallery. Fine for the demo's short clips, not for a
  real camera roll.
- Related systematics this sample avoids, and which its siblings do not:
  `HW-13-0039` (`closeSync` on an undefined handle in `finally` - this sample
  guards both handles), `HW-13-0032` (`new UIContext()` - this sample uses
  `this.getUIContext()`), `HW-13-0002` (`getAssets` behind an unrequested
  permission - this sample never queries the media library at all), and
  `HW-13-0003` (documented tree disagreeing with the zip - this one matches).

## References

- `huawei_industry_tree/13_media_entertainment/docs/04_video_compress_demo.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_compress_demo-0000002248722292
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` - `PhotoViewPicker.select`, `PhotoSelectOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `showAssetsCreationDialog`, `PhotoCreationConfig`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-guides/05_media/photoaccesshelper-savebutton.md` - the permission-free save routes into the gallery
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-savebutton
- `documentation/harmonyos-guides/05_media/component-guidelines-photoviewpicker.md` - why the picker needs no read permission
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/component-guidelines-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-media-avimagegenerator.md` - `fdSrc`, `fetchFrameByTime`, `AVImageQueryOptions`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avimagegenerator
- `documentation/harmonyos-references/04_media/arkts-apis-media-avtranscoder.md` - the platform alternative to `@ohos/videocompressor` when you need explicit bitrate control
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avtranscoder
- `MEDIA-03` - the same picker, used for metadata instead of compression
- `MEDIA-20` - video merging, which shares the fd-lifetime defects listed above
