---
id: PHOTO-18
title: Salt-and-pepper denoising - a 3x3 median filter split across a taskpool TaskGroup
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/18_image_denoising.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_denoising-0000002419864905
sample: huawei_industry_tree/18_photography/downloads/ImageDenoising.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [fileIo, hilog, image, photoAccessHelper, taskpool, window]
permissions: [ohos.permission.WRITE_IMAGEVIDEO]
min_api: 20
modules: [entry]
findings: [HW-18-0050, HW-18-0051, HW-18-0004, HW-18-0024, HW-18-0025, HW-18-0036, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when you have to **run a per-pixel algorithm over a full-resolution
PixelMap without freezing the UI**. The pattern is: read the pixels out into a
plain `ArrayBuffer`, cut the buffer into horizontal bands, hand each band to a
`@Concurrent` function through a `taskpool.TaskGroup`, then stitch the returned
buffers back into a new PixelMap.

The algorithm here is a 3x3 median filter, which is the textbook answer to
salt-and-pepper noise: isolated black and white pixels are extreme values, and a
median throws extremes away while a mean would smear them across the neighbourhood.
But the algorithm is the least transferable part. What generalises is the
buffer-split-merge skeleton — it is the same for a box blur, a sharpen kernel, a
threshold pass, a histogram, or a colour-space conversion. Swap the body of
`medianFilterTask` and everything around it still holds.

**Read `HW-18-0050` before adopting the filter itself.** The shipped kernel
writes the blue median into the red byte and the red median into the blue byte,
so every image it produces comes out with red and blue exchanged. It is one line
each way, and it is on the sample's only working path.

## Feature checklist

- The page opens with a bundled noisy PNG (`img_8.png`) already decoded and shown.
- A round 降噪 (denoise) button under the preview starts the filter; the icon swaps
  to a pressed variant while a touch is down.
- A `LoadingDialog` covers the run and closes when the filtered PixelMap is assigned.
- The filtered image replaces the original in place — one `Image(this.pixelMap)`,
  no before/after split.
- An add button opens `PhotoViewPicker` and swaps in a picture from the gallery,
  resetting the "has been denoised" flag.
- `SaveButton` writes a PNG to the gallery, and refuses with a toast if nothing
  has been denoised yet.
- Colours in the saved file match the colours the user saw.

## Architecture

One `entry` module, one page, two utility files. No model layer — the whole
state is one PixelMap and two booleans.

```
entry/src/main/ets
├── constants/CommonConstants.ets    layout numbers, all as static readonly
├── entryability/EntryAbility.ets    dark colour mode + avoid areas -> AppStorage
├── entrybackupability/
├── pages/MainPage.ets               @Entry: preview, denoise button, footer, save
└── utils
    ├── DenoisingUtil.ets            medianFilter + @Concurrent medianFilterTask + mergeResults
    └── ImageUtil.ets                decode from rawfile / from gallery, and median()
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that `medianFilter` is an ordinary
`async` function taking a PixelMap and returning a PixelMap, with the whole
taskpool apparatus hidden inside. The page does not know that six worker threads
exist:

```typescript
this.pixelMap = await medianFilter(this.pixelMap);
```

That boundary is what makes the pattern reusable. `@Concurrent` functions carry
hard restrictions — they cannot close over anything, all arguments must be
serialisable or transferable — and keeping them behind a plain async facade means
those restrictions stop at the file boundary instead of leaking into the page.

`median()` living in `ImageUtil` rather than next to the filter is the one
questionable placement: it is imported *into* the `@Concurrent` function, which
works because module-level imports are re-resolved in the worker, but it puts a
pure numeric helper in a file otherwise full of file and picker I/O.

## Implementation steps

1. **Decode to an editable PixelMap.** `createPixelMap({ editable: true })` —
   without `editable` the buffer round-trip still works but nothing else you add
   later (rotate, crop, write-back) will.
2. **Read the pixels once** with `readPixelsToBuffer` into an `ArrayBuffer` sized
   by `getPixelBytesNumber()`. Cache `width`/`height` in locals; `getImageInfoSync()`
   is not free and the sample calls it twice.
3. **Slice by whole rows.** `buffer.slice(startY * width * 4, endY * width * 4)` —
   the row stride is what makes a slice a valid sub-image. Give each block one
   overlap row on each side and discard it after filtering (`HW-18-0051`).
4. **Call `task.setTransferList()`** on every task so the block buffer is moved
   into the worker instead of copied.
5. **Group with `taskpool.TaskGroup`,** not six separate `execute` calls: the group
   resolves once, in order, as an `ArrayBuffer[]` — the merge step depends on that
   ordering.
6. **Write each channel back to its own byte** (`HW-18-0050`). R at `targetIdx`,
   G at `+1`, B at `+2`, alpha copied through untouched at `+3`.
7. **Rebuild with `createPixelMapSync(mergedBuffer, opts)`** where `opts.pixelFormat`
   is `RGBA_8888` and `size` is `{ height, width }` — the same format the buffer
   was read in.
8. **Guard the picker's cancel path** before indexing `photoUris[0]` (`HW-18-0024`),
   and keep the source fd open until `createPixelMap` has resolved (`HW-18-0025`).
9. **Release the `ImagePacker`** after `packToData` (`HW-18-0036`), and drop the
   `WRITE_IMAGEVIDEO` declaration — `SaveButton` + `createAsset` needs no
   permission at all (`HW-18-0004`).

## Verified snippets

All snippets are from `ImageDenoising.zip`. Corrected forms are marked.

**The block split — `entry/src/main/ets/utils/DenoisingUtil.ets`** (as shipped)

```typescript
export async function medianFilter(pixelMap: image.PixelMap) {
  let pixelBytesNumber = pixelMap.getPixelBytesNumber();
  let buffer = new ArrayBuffer(pixelBytesNumber);
  await pixelMap.readPixelsToBuffer(buffer);
  let width = pixelMap.getImageInfoSync().size.width;
  let height = pixelMap.getImageInfoSync().size.height;
  let channels = 4;

  // 分块处理参数
  let taskCount = 6;
  let blockHeight = Math.ceil(height / taskCount);
  let taskGroup: taskpool.TaskGroup = new taskpool.TaskGroup();
  for (let i = 0; i < taskCount; i++) {
    let startY = i * blockHeight;
    let endY = Math.min((i + 1) * blockHeight, height);
    let blockBuffer = buffer.slice(startY * width * channels, endY * width * channels);
    let task = new taskpool.Task(medianFilterTask, blockBuffer, width, endY - startY);
    task.setTransferList();
    taskGroup.addTask(task);
  }
  let mergedBuffer = new ArrayBuffer(pixelBytesNumber);
  let opts: image.InitializationOptions = {
    editable: true,
    pixelFormat: image.PixelMapFormat.RGBA_8888,
    size: { height, width }
  };
  try {
    let results = await taskpool.execute(taskGroup) as ArrayBuffer[];
    mergedBuffer = mergeResults(results, width, height, channels);
    return image.createPixelMapSync(mergedBuffer, opts);
  } catch (error) {
    hilog.error(0x0000, 'medianFilter', 'Failed to execute task: %s', `${(error as BusinessError).message}`);
    return image.createPixelMapSync(mergedBuffer, opts);
  }
}
```

**Three things carry the design.** `setTransferList()` with no arguments transfers
every ArrayBuffer argument of the task, so the block is *moved* into the worker —
without it each of the six blocks is structured-cloned and a 12 MP image copies
48 MB twice. `TaskGroup` is what guarantees the results array comes back in the
order the tasks were added, which is the entire correctness argument for
`mergeResults` doing a naive sequential `merged.set(data, offset)`. And
`Math.min((i + 1) * blockHeight, height)` on the last block is what keeps a height
that does not divide by six from over-reading — the last block is simply shorter.

The `catch` returning `createPixelMapSync(mergedBuffer, opts)` on a still-zeroed
buffer is a debatable choice: a failed run produces a fully transparent black
image rather than an error. It is at least not a crash, but a real app should
surface the failure instead of showing the user a blank frame.

**The kernel — same file** (corrected, see `HW-18-0050`)

```typescript
@Concurrent
function medianFilterTask(buffer: ArrayBuffer, width: number, height: number): ArrayBuffer {
  let channels = 4;
  let filterSize = 3;
  let pixelData = new Uint8Array(buffer);
  let filteredData = new Uint8Array(pixelData.length);
  let radius = Math.floor(filterSize / 2);
  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      let neighborsB: number[] = [];
      let neighborsG: number[] = [];
      let neighborsR: number[] = [];
      for (let dy = -radius; dy <= radius; dy++) {
        for (let dx = -radius; dx <= radius; dx++) {
          let ny = y + dy, nx = x + dx;
          if (ny >= 0 && ny < height && nx >= 0 && nx < width) {
            let idx = (ny * width + nx) * channels;
            neighborsR.push(pixelData[idx]);
            neighborsG.push(pixelData[idx + 1]);
            neighborsB.push(pixelData[idx + 2]);
          }
        }
      }
      // 计算中值并替换原值
      let targetIdx = (y * width + x) * channels;
      filteredData[targetIdx] = median(neighborsR);       // FIX: shipped code writes median(neighborsB)
      filteredData[targetIdx + 1] = median(neighborsG);
      filteredData[targetIdx + 2] = median(neighborsB);   // FIX: shipped code writes median(neighborsR)
      filteredData[targetIdx + 3] = pixelData[targetIdx + 3]; // Alpha保持不变
    }
  }
  return filteredData.buffer;
}
```

**The channel swap is the headline defect of this sample.** The neighbour arrays
are collected correctly — `idx` is red, `idx + 1` green, `idx + 2` blue, matching
the `RGBA_8888` the PixelMap was created with. The write-back then crosses them.
It is invisible in a code review because the three lines look symmetric, and it is
invisible in a unit test on a greyscale image, where R, G and B carry the same
value. On a real photo it is unmistakable: skies turn orange and skin turns blue.
Take the two names in the write-back from the two names in the collection loop and
the bug cannot recur.

Two structural details are worth keeping. The `ny >= 0 && ny < height` clamp means
edge pixels are filtered over a truncated 6- or 4-element neighbourhood rather than
a padded one, which is the right default — no invented pixels. And alpha is copied
straight through, never medianed: running a median over alpha would erode
antialiased edges and soft masks.

**The overlap the split is missing — same file** (corrected, see `HW-18-0051`)

```typescript
let radius = 1;                                  // 3x3 kernel
for (let i = 0; i < taskCount; i++) {
  let startY = i * blockHeight;
  let endY = Math.min((i + 1) * blockHeight, height);
  // FIX: grow the slice by one row on each side so interior rows keep their neighbours
  let padTop = Math.max(startY - radius, 0);
  let padBottom = Math.min(endY + radius, height);
  let blockBuffer = buffer.slice(padTop * width * channels, padBottom * width * channels);
  let task = new taskpool.Task(medianFilterTask, blockBuffer, width, padBottom - padTop);
  task.setTransferList();
  taskGroup.addTask(task);
  // and in mergeResults: drop (startY - padTop) rows from the head and
  // (padBottom - endY) rows from the tail of each returned buffer before set()
}
```

**Why the shipped split leaves seams.** Every block is filtered as a standalone
image, so `ny < height` clamps at the *block* edge, not the image edge. Row 0 of
block 2 has no rows above it as far as the kernel is concerned, so it is filtered
over a 6-pixel neighbourhood taken entirely from below. With `taskCount = 6` that
is ten mistreated rows (two per interior boundary, five boundaries) running the
full width of the image — visible as faint horizontal lines exactly where nothing
in the picture changes. Padding by the kernel radius and trimming on merge costs
one extra row of work per block and removes the artefact entirely.

**Save without a permission — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
SaveButton({ text: SaveDescription.SAVE })
  .onClick(async (event: ClickEvent, result: SaveButtonOnClickResult) => {
    if (this.isDenoising) {
      if (result === SaveButtonOnClickResult.SUCCESS) {
        const imagePackerApi: image.ImagePacker = image.createImagePacker();
        let packOpts: image.PackingOption = { format: 'image/png', quality: 98 };
        this.arrayBuffer = await imagePackerApi.packToData(this.pixelMap, packOpts);
        let helper = photoAccessHelper.getPhotoAccessHelper(this.context);
        // onClick触发后10秒内通过createAsset接口创建图片文件，10秒后createAsset权限收回。
        let uri = await helper.createAsset(photoAccessHelper.PhotoType.IMAGE, 'png');
        // 使用uri打开文件，可以持续写入内容，写入过程不受时间限制
        let file = await fileIo.open(uri, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
        await fileIo.write(file.fd, this.arrayBuffer);
        await fileIo.close(file.fd);
        // imagePackerApi.release();  <- FIX: missing, see HW-18-0036
      }
    } else {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.image_not_processed') });
    }
  });
```

**The comment is the important line.** `SaveButton` is a security component: the
user's tap grants a ten-second window in which `createAsset` may be called, and
after that the grant is withdrawn. That is why the code creates the asset
immediately but is free to take its time on the write — the returned URI stays
valid once the asset exists. It is also why this whole flow needs **no permission
at all**, which makes the `WRITE_IMAGEVIDEO` entry in `module.json5` both useless
and harmful (`HW-18-0004`).

Guarding on `this.isDenoising` before saving is a small nicety: the app refuses to
export an unmodified picture back to the gallery, which is the right call for an
editor whose only operation is one filter.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": 'ohos.permission.WRITE_IMAGEVIDEO',
    "reason": '$string:reason',
    "usedScene": { "abilities": ["EntryAbility"], "when": 'always' }
  }
]
```

- **Delete this block.** `WRITE_IMAGEVIDEO` is a restricted (ACL) permission that
  ordinary apps cannot ship with, and nothing in the sample requests it at runtime.
  Reading goes through `PhotoViewPicker` and writing through `SaveButton` +
  `createAsset`, both of which are permission-free by construction (`HW-18-0004`).
- `when: 'always'` would be wrong even if the permission were needed — there is no
  background work here.
- `EntryAbility.onCreate` forces `COLOR_MODE_DARK` for the whole application, not
  just this page. That is a deliberate choice for a photo editor (a neutral dark
  surround stops the surround from biasing colour judgement) but it is applied at
  application scope and never restored.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `taskCount` is a hardcoded 6 regardless of image size or core count. On a small
  image the dispatch overhead dominates; on a very tall one six blocks may still
  be too coarse to keep the dialog responsive.
- The filter is O(width x height x 9 log 9) with a `sort()` per pixel per channel —
  27 array allocations and 3 sorts per pixel. It is a correctness demo, not a fast
  one; a real median filter uses a running histogram.
- The `@Concurrent` function may not capture anything from its enclosing scope.
  `channels`, `filterSize` and `radius` are re-declared inside it for exactly this
  reason, and `median` is reached through a module import, not a closure.
- `mergeResults` assumes the results arrive in task order and that the blocks tile
  the image exactly. Both hold for the shipped split; both must be revisited if
  you add the overlap of `HW-18-0051`.
- The bundled `img_8.png` is copied to `cacheDir/test.png` on every launch and
  never cleaned up.

## Pitfalls

- **`HW-18-0050` (B/high, confirmed) — the median filter swaps red and blue.**
  Neighbours are gathered as R/G/B from an `RGBA_8888` buffer but written back as
  `filteredData[targetIdx] = median(neighborsB)` and
  `filteredData[targetIdx + 2] = median(neighborsR)`. Every denoised image on the
  sample's main path comes out with the two channels exchanged. Fix: write each
  channel to its own byte.
- **`HW-18-0051` (B/low, confirmed) — the taskpool split has no overlap rows.**
  Each block is filtered as if its first and last rows were image edges, so the
  five interior boundaries lose their real vertical neighbours and leave faint
  horizontal seams. Fix: pad each slice by the kernel radius and trim on merge.
- **`HW-18-0004` (D/medium, confirmed) — restricted album permission declared but
  never used.** Systematic across nine photography samples; `ImageDenoising`
  declares `WRITE_IMAGEVIDEO` while saving exclusively through `SaveButton`.
  A copying developer inherits an app-review rejection for nothing. Fix: delete
  the `requestPermissions` block.
- **`HW-18-0024` (B/low, confirmed) — the picker's cancel path is unhandled.**
  Systematic across six samples. `ImageUtil.getPixelMapFromGallery` indexes
  `photoUris[0]` inside the `.then` regardless of whether the array is empty, and
  the caller at `MainPage.ets:115` attaches no `.catch`, so cancelling the picker
  raises an unhandled rejection. Fix: return early on
  `photoSelectResult.photoUris.length === 0` and add a `.catch`.
- **`HW-18-0025` (B/probable) — the fd is closed before the decode finishes.**
  Systematic across three samples. `getPixelMapFromRaw` calls
  `image.createImageSource(file.fd)` then `fileIo.closeSync(file.fd)` and only
  then `await imageSourceApi.createPixelMap(...)`. The source needs the fd to stay
  valid until decoding completes; on the startup path this can leave the main image
  blank. Fix: close in a `finally` after the pixel map is produced.
- **`HW-18-0036` (B/low, confirmed) — the `ImagePacker` is never released.**
  Systematic across eight samples. `createImagePacker()` at `MainPage.ets:131`
  allocates a native packer per save with no matching `release()`, so repeated
  saves accumulate native memory for the process lifetime. Fix: `release()` in a
  `finally`.

## References

- `huawei_industry_tree/18_photography/docs/18_image_denoising.md` — the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_denoising-0000002419864905
- `documentation/harmonyos-references/02_application-framework/js-apis-taskpool.md` — `Task`, `TaskGroup`, `setTransferList`, `execute`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-taskpool
- `documentation/harmonyos-guides/03_application-framework/taskpool-introduction.md` — the `@Concurrent` restrictions
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/taskpool-introduction
- `documentation/harmonyos-references/04_media/arkts-apis-image-pixelmap.md` — `readPixelsToBuffer`, `getPixelBytesNumber`, `getImageInfoSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-pixelmap
- `documentation/harmonyos-references/04_media/arkts-apis-image-ImagePacker.md` — `packToData` and `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagepacker
- `documentation/harmonyos-references/02_application-framework/ts-security-components-savebutton.md` — `SaveButtonOnClickResult` and the grant window
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-savebutton
- `documentation/harmonyos-guides/04_system/savebutton.md` — why the security component needs no permission
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/savebutton
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` — `select` and `PhotoSelectResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `PHOTO-20` — the same `SaveButton` + `createAsset` export, with the picker cancel path done right
