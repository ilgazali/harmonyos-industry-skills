---
id: UTIL-41
title: Decode an image into a chosen pixel format - three sources (camera frame, rawfile, album) into an NV21 PixelMap and a .yuv file
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/41_tools-v1_2-ts_59.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/tools-v1_2-ts_59-0000002329284128
sample: huawei_industry_tree/15_utilities/downloads/ImageDecoder.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CameraKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.LocalizationKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [abilityAccessCtrl, camera, common, fileIo, fs, hilog, image, photoAccessHelper, resourceManager, window]
permissions: [ohos.permission.CAMERA]
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0084, HW-15-0044, HW-15-0078, HW-15-0101]
status: verified-with-fixes
---

## When to use

**Load this card when you have to get pixels out of an image in a specific
format** - NV21, RGBA_8888, a raw YUV buffer for an encoder or an inference
runtime - rather than just showing the picture. The whole point of the sample
is `DecodingOptions.desiredPixelFormat` plus `readPixelsToBuffer`: decode once
into the format the consumer wants, then hand over an `ArrayBuffer` or write
it to a file.

The doc for this page is published under the slug `tools-v1_2-ts_59`, which
says nothing about its subject. **It is not a "tools" page:** the doc, the zip
(`ImageDecoder.zip`) and every screenshot are about image decoding. Do not
look for a tool framework here, and do not let the slug send you to a
different card.

Three source paths appear, and they are the reason to keep the sample around:
a **camera preview frame** arriving through `ImageReceiver` (a byte buffer with
a stride that may not equal the width), a **rawfile resource** reached through
a `RawFileDescriptor`, and an **album pick** reached through a URI that must be
copied into the app sandbox first. Each is a different way to reach
`image.createImageSource`, and the stride handling in the first one is the
piece most people get wrong.

## Feature checklist

- A home page with three buttons pushing three `NavDestination`s through one
  `NavPathStack`.
- **Camera frame page**: requests `ohos.permission.CAMERA`, builds a
  dual-channel preview whose second stream feeds an `ImageReceiver`, converts
  each arriving frame to an NV21 `PixelMap` and shows it rotated 90 degrees.
- 暂停预览 / 继续预览 (pause / resume preview) buttons freeze and unfreeze the
  displayed frame without tearing the session down.
- **Rawfile page**: one 点击开始转换 (start converting) button decodes
  `rawfile/example.jpg` to an NV21 `PixelMap`, shows it, and writes the raw
  pixel bytes to `filesDir/image.yuv`.
- **Album page**: a picker selects one image, the file is copied into the app
  sandbox, decoded with the same options and written out as `.yuv`.
- Both file-writing paths log the destination path and close the fd in a
  `finally`.

## Architecture

One `entry` module, four page files, no model or service layer. Each page is a
self-contained decode pipeline; nothing is shared but the `NavPathStack`.

```
entry/src/main/ets
├── entryability/EntryAbility.ets   colour mode + loadContent('pages/Index'), nothing else
├── entrybackupability/
└── pages
    ├── Index.ets                   @Entry: Navigation + navDestination(pageMap), three buttons
    ├── PreviewPage.ets             camera dual-channel preview -> ImageReceiver -> NV21 PixelMap
    ├── RawfileConverter.ets        getRawFd -> ImageSource -> PixelMap -> filesDir/image.yuv
    └── AlbumPicker.ets             PhotoViewPicker -> sandbox copy -> ImageSource -> .yuv
```

The documented 工程目录 matches the zip exactly (four files under `pages`,
`EntryAbility`, `EntryBackupAbility`).

Routing is by **Chinese display string**: `pushPath({ name: '相机预览帧图片转换' })`
and a `pageMap` builder that compares `name === '相机预览帧图片转换'`. The same
literal is the `NavDestination` title. It works, and it is the structural
choice **worth avoiding** - the route key, the title and the button label are
one untranslatable string in three places, so localising the UI silently
breaks navigation. Route on a stable id and resolve the title from a resource.

**The design decision worth copying** is the split into three independent
pages that each own their whole pipeline. `image.createImageSource` accepts a
path string, a `RawFileDescriptor` or an `ArrayBuffer`, and the differences
between the three sources are all *before* the decode - permissions, fds,
sandbox copies. Keeping them apart means the shared part (`DecodingOptions`,
`createPixelMap`, `readPixelsToBuffer`, the file write) appears three times in
near-identical form, which reads as duplication but is actually the honest
shape: you copy the one page whose source matches yours.

## Implementation steps

1. **Pick the pixel format twice**: once on `createImageSource` via
   `sourcePixelFormat`, once on the decode via
   `DecodingOptions.desiredPixelFormat`. The second is the one that governs
   the resulting `PixelMap`; `editable: true` is required if you intend to
   read pixels back out.
2. **For a camera frame, create the receiver from the matched preview
   profile,** not from a hardcoded size - pick the profile whose aspect ratio
   equals the receiver size's, and bail out if none matches.
3. **Handle stride ≠ width.** `Component.rowStride` is the row pitch in the
   buffer; when it differs from the image width you must repack row by row
   before `createPixelMap`, or every row after the first is offset.
4. **Release `nextImage` on every path,** including the error returns, and
   tear the receiver and session down when the page goes away
   (`HW-15-0044`) - this sample has no teardown at all.
5. **Check `authResults` before starting the camera pipeline** (`HW-15-0078`).
   The shipped `.then()` runs the pipeline whether the user granted or refused.
6. **For the album pick, copy into the sandbox with a path derived from
   `context.filesDir`,** never a hand-written `data/storage/...` literal
   (`HW-15-0084`) - the shipped literal has no leading slash and every copy
   fails.
7. **Close what you open**: `fs.closeSync` the picker fd and the sandbox fd,
   `fileIo.closeSync` the output fd in a `finally`, and `release()` the
   `ImageSource` and the `PixelMap` when the page is done with them.

## Verified snippets

All snippets are from `ImageDecoder.zip`. Corrected forms are marked.

**The stride fix - `entry/src/main/ets/pages/PreviewPage.ets`** (as shipped)

```typescript
async onImageArrival(receiver: image.ImageReceiver): Promise<void> {
  receiver.on('imageArrival', () => {
    receiver.readNextImage((err, nextImage: image.Image) => {
      if (err || nextImage === undefined) {
        hilog.error(DOMAIN, TAG, 'readNextImage failed');
        return;                                   // FIX: nextImage.release() missing here (HW-15-0044)
      }
      nextImage.getComponent(image.ComponentType.JPEG, async (err, imgComponent: image.Component) => {
        if (err || imgComponent === undefined) {
          hilog.error(DOMAIN, TAG, 'getComponent failed');
          return;                                 // FIX: same - the frame is never returned to the pool
        }
        if (this.flag) {
          let width = nextImage.size.width;
          let height = nextImage.size.height;
          let stride = imgComponent.rowStride;    // row pitch, not always == width
          if (stride === width) {
            this.pixmap = await image.createPixelMap(imgComponent.byteBuffer, {
              size: { height: height, width: width },
              srcPixelFormat: image.PixelMapFormat.NV21
            });
          } else {
            const dstBufferSize = width * height * 1.5;   // NV21: 12 bits per pixel
            const dstArr = new Uint8Array(dstBufferSize);
            for (let j = 0; j < height * 1.5; j++) {
              const srcBuf = new Uint8Array(imgComponent.byteBuffer, j * stride, width);
              dstArr.set(srcBuf, j * width);      // drop the padding at the end of each row
            }
            this.pixmap = await image.createPixelMap(dstArr.buffer, {
              size: { height: height, width: width },
              srcPixelFormat: image.PixelMapFormat.NV21
            });
          }
          AppStorage.setOrCreate('stridePixel', this.pixmap);
        }
        nextImage.release();
      });
    });
  });
}
```

**Stride is the whole lesson of this page.** The surface hands back rows
aligned to a hardware boundary, so a 1920-wide NV21 frame can arrive with a
1984-byte pitch. `createPixelMap` is told only `width` and `height`, so it
assumes rows are packed; feeding it the padded buffer produces the classic
diagonal-shear image. The repack loop runs `height * 1.5` times, not `height`,
because NV21 stores the interleaved chroma plane as another half-height of
rows below the luma plane - and those rows carry the same padding.

`this.flag` gates only the conversion, not the subscription: 暂停预览 leaves
the camera session and the receiver running and simply stops producing new
`PixelMap`s. That is the cheap way to freeze a preview, and it is also why the
missing teardown matters - nothing in this page ever stops.

**Camera permission and pipeline start - same file** (corrected, see `HW-15-0078`, `HW-15-0044`)

```typescript
aboutToAppear(): void {
  abilityAccessCtrl.createAtManager()
    .requestPermissionsFromUser(this.getUIContext().getHostContext(), ['ohos.permission.CAMERA'])
    .then((data) => {
      // FIX: the sample ignores the result and starts the pipeline on refusal too
      if (data.authResults.some((r: number) => r !== 0)) {
        hilog.error(DOMAIN, TAG, 'camera permission denied');
        return;
      }
      this.createDualChannelPreview();
    })
    .catch((err: BusinessError) => {              // FIX: absent in the sample
      hilog.error(DOMAIN, TAG, `requestPermissionsFromUser failed: ${err.message}`);
    });
}

// FIX: the sample has no teardown at all - re-entering the page builds a second pipeline
async aboutToDisappear(): Promise<void> {
  this.receiver?.off('imageArrival');
  await this.receiver?.release();
  this.receiver = undefined;
  await this.captureSession?.stop();
  await this.captureSession?.release();
  this.captureSession = undefined;
}
```

**Two systematic defects meet in these ten lines.** `requestPermissionsFromUser`
resolves successfully on a refusal - the refusal shows up as `-1` in
`authResults`, not as a rejected promise - so a bare `.then(() => start())`
treats "no" as "yes" and the camera calls fail later with raw errors
(`HW-15-0078`, shared with `ImagePosition` and `LocationData`). And the page
never releases the receiver or the session, so leaving and re-entering through
the `NavPathStack` stacks a second complete camera pipeline on the first
(`HW-15-0044`, shared with `ScanFrameNo` - see `UTIL-17`). The receiver is
created with an eight-slot buffer (`image.createImageReceiver(size, 2000, 8)`),
so the un-released frames on the two error returns above starve it after eight
bad frames even without the re-entry case.

**The album pick - `entry/src/main/ets/pages/AlbumPicker.ets`** (corrected, see `HW-15-0084`)

```typescript
photoPicker.select(photoSelectOptions).then(async (photoSelectResult: photoAccessHelper.PhotoSelectResult) => {
  const filesDir = this.getUIContext().getHostContext()!.filesDir;   // FIX: was a 'data/storage/...' literal
  let file1 = fs.openSync(photoSelectResult.photoUris[0]);
  const dst = `${filesDir}/${file1.name}`;
  try {
    fs.copyFileSync(file1.fd, dst);
    let file2 = fs.openSync(dst, fs.OpenMode.READ_WRITE);
    try {
      this.filePath = file2.path;
      await this.decodeImage();                                      // FIX: was un-awaited
    } finally {
      fs.closeSync(file2);
    }
  } catch (error) {
    hilog.error(DOMAIN, TAG, `open file error: ${JSON.stringify(error)}`);
  } finally {
    fs.closeSync(file1);
  }
}).catch((err: BusinessError) => {
  hilog.error(DOMAIN, TAG, `PhotoViewPicker.select failed with err: ${err.message}`);
});
```

**The shipped line is `data/storage/el2/base/haps/entry/files/${file1.name}`
with no leading slash**, which is not the sandbox path but a relative one, so
`copyFileSync` throws, the outer `catch` logs, and the album half of the sample
- one of its three advertised flows - silently does nothing. The correct value
is never a literal: `context.filesDir` already resolves to
`/data/storage/el2/base/haps/entry/files` for this module, and it stays correct
if the module name or the encryption level changes.

The copy is not optional. `photoUris[0]` is a media-library URI, and the
picker grants only a transient read on it; `image.createImageSource` wants a
path it can open for the lifetime of the decode. Copy in, decode from the
sandbox, delete when done.

**The decode and the buffer write - same file** (as shipped)

```typescript
const imageSource: image.ImageSource = image.createImageSource(filePath, {
  sourceDensity: 120, sourcePixelFormat: image.PixelMapFormat.NV21
});
let opts: image.DecodingOptions = {
  editable: true, desiredPixelFormat: image.PixelMapFormat.NV21
};
imageSource.createPixelMap(opts).then((pixelMap: image.PixelMap) => {
  this.pixelMap = pixelMap;
  const readBuffer: ArrayBuffer = new ArrayBuffer(this.pixelMap.getPixelBytesNumber());
  if (this.pixelMap !== undefined) {
    this.pixelMap.readPixelsToBuffer(readBuffer).then(() => {
      let path1: string = this.getUIContext().getHostContext()?.filesDir + '/image.yuv';
      let file1 = fileIo.openSync(path1, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
      let opt: WriteOptions = { length: readBuffer.byteLength };
      try {
        fileIo.writeSync(file1.fd, readBuffer, opt);
      } catch (error) {
        hilog.error(DOMAIN, TAG, `write to buffer failed: ${JSON.stringify(error)}`);
      } finally {
        fileIo.closeSync(file1);
      }
    }).catch((error: BusinessError) => {
      hilog.error(DOMAIN, TAG, `Failed to read image pixel data. code is ${error.code}, message is ${error.message}`);
    });
  }
});
```

**Three lines carry the design.** `desiredPixelFormat: NV21` is what makes the
decoded `PixelMap` NV21 rather than the default RGBA_8888 - `sourcePixelFormat`
on `createImageSource` only describes the input and is meaningless for a JPEG.
`editable: true` is what allows `readPixelsToBuffer` afterwards; without it the
decode produces a read-only map. And `getPixelBytesNumber()` is the only
correct way to size the buffer - `width * height * 4` is wrong for every format
except RGBA_8888, and wrong here by a factor of 2.67.

Note what is missing: `imageSource.release()` and `pixelMap.release()` are
never called on any of the three pages, so every tap leaks a decoder and a
pixel buffer. The rawfile page additionally leaks the raw fd from `getRawFd`,
which has no `closeRawFd` companion call anywhere in the file. The write path
is otherwise correct - the `finally` closes the output fd even when
`writeSync` throws, and this is the one place in the sample where cleanup is
done properly.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.CAMERA",
    "reason": "$string:reason_camera",
    "usedScene": { "abilities": ["EntryAbility"] }
  }
]
```

- `CAMERA` is `user_grant`, so `reason` and `usedScene` are mandatory; the
  `usedScene` here omits `when`, which defaults to `inuse` - correct for this
  sample, which has no background capture.
- The album picker needs **no permission at all**: `PhotoViewPicker` is a
  system picker and returns a URI the app is granted access to for the
  selection. Do not add `READ_IMAGEVIDEO` for this flow.
- `deviceTypes` is `["phone"]` only.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The camera page needs real hardware - the dual-channel preview, the
  `ImageReceiver` surface and the stride behaviour do not reproduce on the
  emulator.
- The rawfile page decodes a fixed `example.jpg`; the source comment says
  开发者需手动替换 (the developer must replace it manually).
- `imageSize` is hardcoded to 1920x1080 and used only as an aspect-ratio
  probe when selecting the preview profile; on a device with no 16:9 preview
  profile the page logs `previewProfile not found!` and shows nothing.
- Only the second preview stream is wired up. There is no `XComponent`
  surface, so there is no live viewfinder - the `Image` showing the converted
  `PixelMap` *is* the preview, at whatever rate conversion keeps up.

## Pitfalls

- **`HW-15-0084`** (B/high, confirmed): the album flow's sandbox path is the
  literal `data/storage/el2/base/haps/entry/files/...` without a leading
  slash, so `copyFileSync` and the following `openSync` always fail; the outer
  `catch` only logs, so the page looks alive and converts nothing. Fix: derive
  the path from `context.filesDir`, and release the `ImageSource`/`PixelMap`
  and the raw fd on every path.
- **`HW-15-0044`** (B/medium, confirmed, systematic): the ImageReceiver
  pipeline leaks. `PreviewPage` has no teardown at all - session, input and
  receiver outlive the page, and re-entry builds a second pipeline - and the
  two error returns inside `readNextImage`/`getComponent` skip
  `nextImage.release()`, starving the eight-slot buffer after eight bad
  frames. `ImageDecoder` is the second instance; `ScanFrameNo` (`UTIL-17`) is
  the first. Fix: `off` and `release` the receiver in `aboutToDisappear`, and
  release the image in a `finally`.
- **`HW-15-0078`** (D/high, confirmed, systematic): `requestPermissionsFromUser`
  is called with a `.then()` that ignores `authResults`, so refusing the camera
  permission still starts the whole capture pipeline, which then fails with raw
  errors and no `.catch`. `ImageDecoder` is one of three utilities samples with
  this exact shape (`ImagePosition`, `LocationData`). Fix: branch on
  `data.authResults` before proceeding.
- **Doc slug** — the page is published as `tools-v1_2-ts_59`, a generic
  "tools" slug that matches neither the title (如何将图片解码为不同格式类型)
  nor the sample (`ImageDecoder.zip`). Search by title, not by slug.
- **Route keys are display strings** — `pushPath({ name: '相机预览帧图片转换' })`
  and the `pageMap` comparison share one Chinese literal with the button label
  and the `NavDestination` title. Translating the UI breaks navigation.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-image-pixelmap.md` - `readPixelsToBuffer`, `getPixelBytesNumber`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-pixelmap
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagesource.md` - `createImageSource` overloads, `DecodingOptions`, `desiredPixelFormat`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagesource
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagereceiver.md` - `createImageReceiver`, `imageArrival`, `readNextImage`, `Component.rowStride`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagereceiver
- `documentation/harmonyos-guides/05_media/camera-dual-channel-preview.md` - the dual-channel preview the camera page is built on
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-dual-channel-preview
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `ohos.permission.CAMERA` and the user_grant flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `UTIL-17` - `ScanFrameNo`, the first instance of the same ImageReceiver leak
- `UTIL-37` - `ImagePosition`, the anchor of the refusal-as-granted systematic
