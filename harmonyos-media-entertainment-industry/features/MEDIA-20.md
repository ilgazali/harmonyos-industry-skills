---
id: MEDIA-20
title: Video concatenation - drag-sortable clip strip feeding an FFmpeg concat pipeline
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/20_merge_video.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/merge_video-0000002297453758
sample: huawei_industry_tree/13_media_entertainment/downloads/MergeVideo.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [common, curves, dataSharePredicates, fileIo, fileUri, hilog, image, photoAccessHelper, window]
permissions: [ohos.permission.READ_IMAGEVIDEO]
min_api: 20
modules: [entry (entry)]
findings: [HW-13-0049, HW-13-0050, HW-13-0051, HW-13-0052, HW-13-0053, HW-13-0002, HW-13-0039, HW-13-0015, HW-13-0097, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card for a **lightweight video editor timeline**: pick several clips
from the gallery, see them as a horizontal thumbnail strip, drag to reorder,
then export one merged file to the album. Story creators, vlog apps, "combine
your day" features - all the same two halves.

The first half is a **drag-to-reorder strip**, worth copying whatever the
payload is: a `GestureGroup(GestureMode.Sequence)` of a long press followed by
a pan, a floating clone rendered by a `@Builder` at an absolute position, and a
swap that fires when the drag passes two thirds of an item's width. It reorders
the underlying arrays live, so the model is always what the user sees.

The second half is an **FFmpeg concat pipeline** via the `@ohos/mp4parser`
third-party library: every clip is copied into the cache, transcoded to
MPEG-TS, rotated to match its EXIF orientation, listed in a `video.txt` concat
file and merged. That shape - normalise, list, concat - is how ffmpeg's
stream-copy concat demuxer is meant to be driven, and it is why the sample
refuses to merge clips of differing resolution.

**This sample carries eight findings, five of them its own.** The merge path in
particular has a wrong rotation filter (`HW-13-0049`), a work file that is
never truncated (`HW-13-0050`) and a resolution guard that passes when it has
no data (`HW-13-0052`). Treat the structure as the lesson and the code as a
draft.

## Feature checklist

- A 视频拼接 (video merge) header with a 保存 (save) button, over a preview
  `Video` player with play/pause, ±10 s seek and a current/total time badge.
- A "+" button opens the photo picker filtered to videos, up to nine.
- Each picked clip becomes a thumbnail in a horizontal strip, captioned with
  its duration; tapping one loads it into the preview player.
- Long-pressing a thumbnail lifts a shadowed clone; dragging it past its
  neighbours reorders the strip, and dragging to either edge auto-scrolls.
- The strip order *is* the merge order.
- Save checks that every clip has the same resolution, then runs the merge
  behind a full-screen spinner.
- On success the result is written to the gallery through the system's
  save-confirmation dialog, and the working directory is cleared.

## Architecture

One `entry` module. The page/utility split is clean; the utilities are the
problem.

```
entry/src/main/ets
├── common/Constants.ets           strip geometry, video-area percentages, drag velocity
├── entryability/EntryAbility.ets  full screen, avoid areas -> AppStorage, uiContext -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── pages
│   ├── MainPage.ets               @Entry: picker, thumbnails, preview player, save
│   └── VideoList.ets              the drag-sortable thumbnail strip
└── utils
    ├── CacheUtils.ets             work dir lifecycle + saving to the album (module singleton)
    ├── Logger.ets                 hilog wrapper
    ├── PhotoUtils.ets             PhotoAsset lookups: size, orientation, duration, thumbnail
    └── VideoUtils.ets             the ffmpeg pipeline (module singleton)
```

The documented 工程目录 matches the zip.

**The design decision worth copying** is that `VideoList` never merges
anything and `VideoUtils` never renders anything. The strip receives four
`@Link` arrays plus a `resetVideo` callback and mutates them in place; the
merge reads `videoList` in array order and nothing else. The order the user
arranges is therefore the same object the pipeline consumes - there is no
"apply" step that could go out of sync.

**The design decision worth avoiding** is the module singleton with an
eager context:

```typescript
class VideoUtils {
  private uiContext = AppStorage.get('uiContext') as UIContext;
  context = this.uiContext.getHostContext() as common.UIAbilityContext;
  getLocalDirPath = this.context.cacheDir;
  // ...
}
export default new VideoUtils();
```

Both `VideoUtils` and `CacheUtils` are constructed at *import* time, and their
field initialisers dereference `AppStorage.get('uiContext')` - a key
`EntryAbility` only sets **inside the `loadContent` callback**. Whether the app
launches or crashes depends on module-evaluation order against that callback
(`HW-13-0053`). `PhotoUtils` is the same idea one notch safer: it reads the key
into an optional and calls `getHostContext()` with `?.`, degrading to a helper
with no context rather than throwing. Fetch the context lazily inside the
methods, or set the key before `loadContent`.

The other weakness is that the strip is backed by **three parallel arrays** -
`picList`, `videoList`, `timeList` - kept in step only by convention. Every
reorder splices all three by the same index, which works; the build path fills
them from three different asynchronous places, which does not (`HW-13-0051`).

## Implementation steps

1. **Create a per-run work directory** and delete it first if it exists;
   `CacheUtils.initializeFolder()` does this on page entry, and again after a
   successful save.
2. **Pick with `PhotoViewMIMETypes.VIDEO_TYPE`,** `maxSelectNumber: 9`.
3. **Build one record per clip** - uri, thumbnail path, duration - after
   `packToFile` completes, rather than pushing into three arrays from three
   different callbacks (`HW-13-0051`).
4. **Close the thumbnail fd in the pack callback, not in a `finally`**
   (`HW-13-0015`), and guard every close against `undefined` (`HW-13-0039`).
5. **Compose the drag gesture as a sequence:** long press to select, pan to
   move, both reset on end and on cancel.
6. **Swap when the drag passes two thirds of an item width,** and account for
   any auto-scroll that happened during the drag.
7. **Fail the resolution check closed** - require a size for every input
   (`HW-13-0052`).
8. **Normalise each clip to MPEG-TS, then rotate,** using `hflip,vflip` for the
   180° case rather than `transpose=3` (`HW-13-0049`).
9. **Truncate `video.txt` on open** so a shorter second run cannot inherit the
   first run's segment list (`HW-13-0050`).
10. **Save through `showAssetsCreationDialog`,** which grants the write without
    any permission of your own.

## Verified snippets

All snippets are from `MergeVideo.zip`. Corrected forms are marked.

**The drag-sortable strip — `entry/src/main/ets/pages/VideoList.ets`** (as shipped)

```typescript
.gesture(
  GestureGroup(GestureMode.Sequence,
    LongPressGesture({ repeat: false, duration: 250 })
      .onAction((event?: GestureEvent) => {
        if (this.selectedItem === '') {
          this.selectedItem = item;                               // lift this thumbnail
          this.dragStartX = Number(event?.target.area.globalPosition.x);
        }
      }),
    PanGesture({ fingers: 1, direction: null, distance: 0 })
      .onActionStart(() => { this.scrollLen = 0; })
      .onActionUpdate((event: GestureEvent) => {
        this.offsetX = event.offsetX;
        this.getUIContext().animateTo({ curve: curves.interpolatingSpring(0, 1, 400, 38) }, () => {
          this.handleDrag();
        });
      })
      .onActionEnd(() => { this.reset(); })
  ).onCancel(() => { this.reset(); })
);

handleDrag() {
  let x = this.scrollLen + this.offsetX - this.dragOffsetX;       // travel since the last swap
  let index = this.picList.indexOf(this.selectedItem);
  if (x > 0 && index + 2 <= this.picList.length) {
    let w = Constants.LIST_ITEM_WIDTH;
    if (x >= w / 3 * 2) {                                          // two thirds of an item
      let tmp = this.picList.splice(index, 1);
      this.picList.splice(index + 1, 0, tmp[0]);
      let tmpUri = this.videoList.splice(index, 1);
      this.videoList.splice(index + 1, 0, tmpUri[0]);
      let tmpTime = this.timeList.splice(index, 1);
      this.timeList.splice(index + 1, 0, tmpTime[0]);
      this.dragOffsetX += w;                                       // consume the travel
    }
  }
  // symmetric branch for x < 0
}
```

**Three details make this feel right.** `GestureMode.Sequence` means the pan is
only recognised after the 250 ms press succeeds, so an ordinary horizontal
flick still scrolls the strip instead of dragging an item. `dragOffsetX`
accumulates one item width per swap, so `x` is always "distance since the last
swap" rather than "distance since the press" - that is what stops a long drag
from swapping the same pair repeatedly. And `scrollLen`, fed from the
`Scroll`'s `onDidScroll`, folds auto-scrolling into the same number, so
dragging to the edge and letting the strip scroll under your finger still lands
on the right index.

The lifted item is drawn twice: the original stays in the list at `.opacity(0)`
to hold the gap, and `DragBuilder` renders a shadowed clone at
`x: this.dragStartX + this.offsetX - this.globalPositionX` - positioned against
the container's own global x, so it stays under the finger wherever the strip
sits on screen. `hitTestBehavior(HitTestMode.None)` keeps the clone from
stealing the pan it follows, and its `onAreaChange` is where the edge
auto-scroll lives. The spring wrapping `handleDrag()` animates the
*neighbours* into place; the array swap itself is instantaneous.

**The ffmpeg normalise-and-rotate pass — `entry/src/main/ets/utils/VideoUtils.ets`** (corrected, see `HW-13-0049`)

```typescript
// Execute the FFmpeg conversion command
await new Promise<void>((resolve, reject) => {
  MP4Parser.ffmpegCmd(
    `ffmpeg -i ${this.videoDir}/${videoMp4Name} -c copy -f mpegts ${this.videoDir}/${videoTmpTsName}`,
    { callBackResult: (code: number) => code === 0 ? resolve() : reject(new Error(`code: ${code}`)) }
  );
});
let orientation = await this.photoUtils.acquireOrientationByUrl(uri);
let transposeCmd: string = '';
if (orientation) {
  if (orientation === 90) {
    transposeCmd = '-vf "transpose=1" ';                 // 90° clockwise
  }
  if (orientation === 180) {
    transposeCmd = '-vf "hflip,vflip" ';                 // FIX: shipped code uses transpose=3
  }
  if (orientation === 270) {
    transposeCmd = '-vf "transpose=2" ';                 // 90° counter-clockwise
  }
}
// Rotate the video.
await new Promise<void>((resolve, reject) => {
  MP4Parser.ffmpegCmd(
    `ffmpeg -i ${this.videoDir}/${videoTmpTsName} ${transposeCmd}-c:a copy ${this.videoDir}/${videoTsName}`,
    { callBackResult: (code: number) => code === 0 ? resolve() : reject(new Error(`code: ${code}`)) }
  );
});
```

**`transpose` has no 180° value.** The filter's four modes are all quarter
turns: `0` = 90° counter-clockwise with vertical flip, `1` = 90° clockwise,
`2` = 90° counter-clockwise, `3` = 90° clockwise with vertical flip. The
sample's `transpose=3` for orientation 180 therefore produces a clip rotated a
quarter turn *and* mirrored - visibly wrong - and, because a quarter turn swaps
width and height, it also changes the clip's resolution. That second effect is
worse than the first: the merge assumes every segment shares a resolution, and
`checkVideoSize` only compensates for the 90/270 swap, so a 180-oriented clip
can break the concat with an opaque ffmpeg error. A 180° turn is two flips:
`hflip,vflip` (or `rotate=PI`), and it preserves the dimensions.

The two-pass structure is right, though. The first command is a pure stream
copy into a temporary `.ts` - no re-encode, and it gives the concat demuxer the
container it wants. Only the second pass carries a video filter, and only when
a rotation is needed; with `transposeCmd` empty it is effectively another copy.
`-c:a copy` throughout avoids re-encoding sound that never changes.

**The concat list — same file** (corrected, see `HW-13-0050`, `HW-13-0039`)

```typescript
async mergeVideo(videoList: string[]): Promise<boolean> {
  await this.formatVideo(videoList);
  let isSuccess = false;
  let file: fs.File | undefined = undefined;
  try {
    file = await fs.open(this.videoTxt,
      fs.OpenMode.CREATE | fs.OpenMode.READ_WRITE | fs.OpenMode.TRUNC);   // FIX: no TRUNC shipped
    let videoNum = 0;
    for (let uri of videoList) {
      videoNum++;
      let videoName = 'video' + videoNum.toString() + '.ts';
      let fileContent = 'file \'' + videoName + '\'\n';
      await fs.write(file.fd, fileContent);
    }
    // Execute the FFmpeg batch merging command.
    await new Promise<void>((resolve, reject) => {
      MP4Parser.videoMultMerge(this.videoTxt, this.videoDir + '/' + this.outVideo, {
        callBackResult: (code: number) => {
          if (code === 0) {
            isSuccess = true;
            resolve();
          } else {
            isSuccess = false;
            reject(new Error(`视频合并命令执行失败，返回码: ${code}`));
          }
        },
      });
    });
    return isSuccess;
  } catch (err) {
    Logger.error('操作失败:', err.message);
    return false;
  } finally {
    if (file) {                                     // FIX: shipped code calls fs.close(undefined)
      fs.close(file);
    }
  }
}
```

**`video.txt` is a reused file, and reuse without `TRUNC` is a trap.** The
concat list is one `file 'videoN.ts'` line per clip. Merge five clips, cancel
the save (which is what skips `initializeFolder()`), then merge three: the
write covers the first three lines, the fourth and fifth survive from the
previous run, and the `.ts` segments they name are still on disk. The output
silently contains two clips the user removed. `OpenMode.TRUNC` is one flag;
clearing the work directory per run rather than only after a successful save is
the belt to that braces.

The `finally` is the second defect, and a systematic one. If `fs.open` throws,
`file` is `undefined` and `fs.close(undefined)` throws from the cleanup path,
so a routine failure becomes a `TypeError` masking the real error. The same
double-fault appears in `CacheUtils.saveVideo` - refusing the save dialog
leaves both `srcFile` and `desFile` undefined for two unguarded `closeSync`
calls - and across four other media samples (`HW-13-0039`).

**The resolution guard — same file** (corrected, see `HW-13-0052`, `HW-13-0002`)

```typescript
// Determine if the video resolutions are the same.
async checkVideoSize(videoList: string[]): Promise<boolean> {
  let videoSizes: Size[] = [];
  for (let uri of videoList) {
    let size = await this.photoUtils.acquireSizeByUrl(uri);
    if (size) {
      videoSizes.push(size);
    }
  }
  if (videoSizes.length !== videoList.length) {     // FIX: absent - failures were simply dropped
    return false;                                   // fail closed: we could not verify
  }
  for (let element of videoSizes) {
    if (element.width !== videoSizes[0].width) {
      return false;
    }
    if (element.height !== videoSizes[0].height) {
      return false;
    }
  }
  return true;
}
```

**A guard that passes on missing data is not a guard.** `acquireSizeByUrl`
swallows its errors and returns `undefined`, the loop drops those, and an empty
`videoSizes` sails through the comparison loop and returns `true`. That is the
actual behaviour on a device: `acquireSizeByUrl` goes through
`PhotoUtils.getPhotoAsset`, which calls `getAssets`, which requires
`ohos.permission.READ_IMAGEVIDEO` - declared in `module.json5`, never requested
at runtime (`HW-13-0002`). So every lookup fails, the check passes vacuously,
and mismatched clips reach the merge, which fails with an ffmpeg return code
and a Chinese log line.

The two findings compound: fix the permission and the guard starts working; fix
the guard alone and the feature stops merging anything, which at least fails
honestly. The better fix for both is the one `HW-13-0002` recommends - read the
metadata from the picker URI, which is already granted to your app.

**Building the strip from the picker — `entry/src/main/ets/pages/MainPage.ets`** (corrected, see `HW-13-0051`, `HW-13-0015`)

```typescript
interface ClipRecord {                                 // FIX: three parallel arrays shipped
  uri: string;
  thumb: string;
  time: string;
}

photoViewPicker.select(photoSelectOptions)
  .then(async (photoSelectResult: photoAccessHelper.PhotoSelectResult) => {
    let i = 0;
    for (let result of photoSelectResult.photoUris) {
      i++;
      let thumbnailView = await this.photoUtils.acquireThumbnailByUrl(result);
      if (thumbnailView === undefined) {
        Logger.error(`the pixelMap acquires is undefined!`);
        continue;                                      // FIX: skip the whole clip, not just its thumb
      }
      let uri = fileUri.getUriFromPath(this.picDir + '/pic' + i.toString() + '.jpg');
      let file: fileIo.File | undefined = undefined;
      let imagePackerApi = image.createImagePacker();
      let packOpts: image.PackingOption = { format: 'image/jpeg', quality: 98 };
      try {
        file = await fileIo.open(uri, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
        await imagePackerApi.packToFile(thumbnailView, file.fd, packOpts);   // FIX: awaited, not callback
      } finally {
        if (file) {                                    // FIX: guarded, and after the pack completes
          fileIo.close(file);
        }
        imagePackerApi.release();
      }
      let duration = await this.photoUtils.acquireDurationByUrl(result);
      // FIX: one record, pushed once, after everything for this clip succeeded
      this.clips.push({
        uri: result,
        thumb: uri,
        time: duration ? VideoUtils.formatDuration(duration) : '00:00:00'
      } as ClipRecord);
    }
    if (this.clips.length > 0) {
      this.videoShow = this.clips[0].uri;
    }
  });
```

**The shipped loop pushes into three arrays from three different places**:
`videoList` synchronously, `picList` from inside the `packToFile` *callback*,
`timeList` only when `acquireDurationByUrl` returns something truthy - while
the UI and the drag reorder index all three by the `picList` position. One
failed thumbnail, one zero-length duration, or two callbacks completing out of
order, and the strip shows the wrong duration under the wrong frame, taps play
the wrong clip, and the merged order is not the arranged one. One record pushed
once, after everything for that clip succeeded, makes that impossible by
construction.

The fd is the second correction. In the shipped code `fileIo.close(file)` sits
in a `finally` that runs the moment `packToFile` is *called* - the callback
form returns immediately - so the packer keeps writing thumbnail bytes into a
descriptor the app has already closed, which is why thumbnails come out blank
intermittently. Awaiting the promise form moves the close into the completion
path. Five media samples share that race (`HW-13-0015`).

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": 'ohos.permission.READ_IMAGEVIDEO',
    "reason": '$string:reason',
    "usedScene": {
      "abilities": ["EntryAbility"],
      "when": 'always'
    },
  }
]
```

- Declared and never requested (`HW-13-0002`). It is a restricted user_grant
  permission, so it needs both a runtime `requestPermissionsFromUser` and an
  ACL entry - and everything in `PhotoUtils` depends on it.
- **Writing needs no permission of its own.** `CacheUtils.saveVideo` uses
  `showAssetsCreationDialog`, which asks the user and returns a target URI
  already open for writing. That is the right pattern and the sample gets it
  right; only the reading half is broken.
- The `@ohos/mp4parser` dependency is declared as `^2.0.6` in
  `oh-package.json5` while the document instructs
  `ohpm install @ohos/mp4parser@2.0.7`. The caret range covers 2.0.7, so both
  resolve, but the two do not literally agree.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **All clips must share a resolution.** The concat demuxer is a stream-copy
  operation; differing dimensions cannot be concatenated without re-encoding.
  The sample refuses rather than re-encodes, which is the right trade for
  speed but means a mixed selection simply cannot be merged.
- Up to nine clips per session (`maxSelectNumber: 9`). Everything happens in
  `context.cacheDir/videoDir` - a `.mp4` copy, a temporary `.ts` and a rotated
  `.ts` per clip plus the output - so a long selection multiplies the source
  size several times over, with no free-space check.
- The output is always named `test.mp4`, so repeated saves produce
  indistinguishable album entries. The merge runs on the UI thread's promise
  chain behind a `LoadingProgress`, with no progress reporting and no cancel.
- The strip has no delete: the "+" button clears the entire selection and
  reopens the picker. `Video` re-prepares on every thumbnail tap, which is why
  `resetVideo` and the `canChangeVideo` gate exist - taps are ignored until
  `onPrepared` fires.

## Pitfalls

- **`HW-13-0049`** (B/medium, confirmed): `transpose=3` is used for the 180°
  case, but `transpose` only has quarter-turn modes - `3` is 90° clockwise plus
  a vertical flip. 180-oriented clips come out transposed and mirrored, with
  swapped dimensions that then violate the merge's equal-resolution assumption.
  Fix: `-vf "hflip,vflip"` (or `rotate=PI`) for 180.
- **`HW-13-0050`** (B/medium, probable): `video.txt` is opened
  `CREATE | READ_WRITE` with no `TRUNC` and the work directory is only cleared
  after a *successful* save, so a shorter later run inherits stale
  `file 'videoN.ts'` lines and the segments they point at. Two sibling samples
  (`SplashPage`, `ImageMixText`) reuse work files the same way. Fix: add
  `OpenMode.TRUNC` and clear the work directory per run.
- **`HW-13-0051`** (B/medium, probable): `picList`, `videoList` and `timeList`
  are filled from three different places - one synchronous, one inside an async
  `packToFile` callback, one conditional on a truthy duration - while the UI
  and the drag reorder index all three by the `picList` position. Same shape in
  `SplashPage`. Fix: push one `{uri, thumb, time}` record per clip after
  awaiting the pack.
- **`HW-13-0052`** (B/low, confirmed): `checkVideoSize` drops failed size
  lookups and returns `true` for an empty list, so the resolution guard passes
  precisely when it knows nothing - and with `READ_IMAGEVIDEO` never granted,
  every lookup fails. Fix: return `false` unless
  `videoSizes.length === videoList.length`.
- **`HW-13-0053`** (B/medium, probable): `VideoUtils` and `CacheUtils` are
  module-level singletons whose field initialisers call
  `AppStorage.get('uiContext').getHostContext()` at import time, while
  `EntryAbility` sets that key only inside the `loadContent` callback. Whether
  the app survives launch depends on module-evaluation order.
  `XComponentTransition` has the same shape with its shared `AVPlayer`. Fix:
  fetch the context lazily inside the methods, or set the key before
  `loadContent`.
- **`HW-13-0002`** (B/medium, confirmed): every `PhotoUtils` helper goes
  through `getAssets`, which requires `ohos.permission.READ_IMAGEVIDEO` -
  declared but never requested - so orientation, duration, size and thumbnails
  all fail on a device. Shared with `MEDIA-03` and `MEDIA-19`. Fix: read the
  metadata from the picker URI instead of querying the media library.
- **`HW-13-0039`** (B/medium, confirmed): unguarded `close`/`closeSync` in
  `finally` blocks - `VideoUtils.mergeVideo` closes a possibly-undefined file,
  and `CacheUtils.saveVideo` closes two of them, so refusing the save dialog
  turns a routine cancel into a `TypeError` that masks the original error and
  leaks whichever handle did open. Five media samples share it. Fix: guard each
  handle before closing.
- **`HW-13-0015`** (B/medium, probable): `MainPage` closes the thumbnail fd in
  a `finally` that runs while `packToFile`'s callback is still writing to it.
  Thumbnails fail intermittently by timing. Five media samples share the race.
  Fix: close in the completion path (await the promise form).

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - horizontal `List`, `ListItem`, `edgeEffect`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-guides/03_application-framework/arkts-gesture-events-single-gesture.md` - `LongPressGesture`, `PanGesture`, and `GestureGroup(GestureMode.Sequence)`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-gesture-events-single-gesture
- `documentation/harmonyos-references/02_application-framework/ts-container-scroll.md` - `Scroller.scrollEdge`, `currentOffset`, `onDidScroll`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scroll
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` - `PhotoSelectOptions`, `PhotoViewMIMETypes.VIDEO_TYPE`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoasset.md` - `PhotoKeys.ORIENTATION` / `WIDTH` / `HEIGHT` / `DURATION`, `getThumbnail`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoasset
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `getAssets`, `showAssetsCreationDialog`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagepacker.md` - `createImagePacker`, `packToFile`, `PackingOption`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagepacker
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `open`, `OpenMode.TRUNC`, `write`, `close`, `copy`, `rmdirSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - `ohos.permission.READ_IMAGEVIDEO`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
- `MEDIA-03` and `MEDIA-19` - the two other instances of the same `getAssets` permission defect
- `MEDIA-14` - the same unguarded-`finally` cleanup defect around a creation dialog
