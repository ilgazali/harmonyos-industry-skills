---
id: PHOTO-22
title: Video format conversion - one ffmpeg command through mp4parser, with an AVPlayer preview
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/22_video_encoding_convert.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_encoding_convert-0000002409157670
sample: huawei_industry_tree/18_photography/downloads/VideoEncodingConvert.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.MediaKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [display, fs, hilog, media, photoAccessHelper, window]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-18-0066, HW-18-0067, HW-18-0068, HW-18-0024, HW-18-0091, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when the app must **re-container or re-encode a video on the
device**, with no server in the loop: the user picks a clip from the gallery,
chooses a target format, and gets a new file back. The pattern is a copy into
the sandbox, a preview on an `XComponent` surface driven by `AVPlayer`, and a
single `ffmpeg` command line executed by `@ohos/mp4parser` whose output is
written straight back into `filesDir`.

The transferable part is the shape, not the codec table. `MP4Parser.ffmpegCmd`
takes a command string and a callback carrying an integer result, so anything
the bundled ffmpeg can do - trimming, muxing, extracting an audio track,
burning in an overlay - fits the same three moves: build the destination path
in the sandbox, build the command, branch on the result code. `PHOTO-21` uses
the identical library and callback for watermark composition, and gets the
result-code branch wrong; this sample gets it right and is the better model of
the two.

The other reusable idea is treating the sandbox directory itself as the data
model, described under Architecture. **Read `HW-18-0066` before copying the
pagination** - one empty page permanently kills the grid's infinite scroll.

## Feature checklist

- A card on the home page opens the system photo picker filtered to video, one
  file at a time; the picked video is copied into the app sandbox and nothing
  outside the sandbox is ever read again.
- The home page shows a three-column grid of sandbox videos, each with a
  first-frame thumbnail, its formatted size and its duration.
- The grid loads twelve files at a time and fetches the next page when the user
  reaches the end.
- Picking a file - or completing the picker - pushes the transcode page.
- The transcode page plays the video on an `XComponent` surface sized to the
  clip's own aspect ratio, with play/pause, elapsed and total time labels, and
  a seek slider.
- Nine target containers are offered (mp4, mkv, mov, avi, flv, ts, 3gp, 3g2,
  f4v); selecting one highlights it and arms the convert button.
- While a conversion runs, the button shows a spinner, the back arrow is inert
  and the system back gesture is swallowed.
- On success the page pops the produced file back to the home page, which
  toasts and inserts it at the head of the grid; on failure the half-written
  output is deleted from the sandbox and a failure toast is shown.

## Architecture

One `entry` module, two pages, one utility class, two model files. No database
and no repository layer.

```
entry/src/main/ets
├── constants/CommonConstants.ets   time bases, size units, separators, the blue
├── entryability/EntryAbility.ets   full-screen layout, avoid areas -> AppStorage
├── model
│   ├── DataModel.ets               VideoFile (@Observed), CUSTOM_TABS, SAMPLE_CODECS, CODEC_TABLE
│   └── FileDataSource.ets          IDataSource over VideoFile[] for LazyForEach
├── pages
│   ├── MainPage.ets                @Entry: Navigation host, picker card, tabs, the file grid
│   └── ParsePage.ets               NavDestination: AVPlayer preview, format chips, the ffmpeg call
└── utils
    ├── Logger.ets                  hilog wrapper
    └── VideoUtils.ets              the whole file layer: pick, copy, list, probe, delete
```

The documented 工程目录 matches the zip exactly - eight files, same names, same
directories. (This industry has a systematic tree-mismatch finding,
`HW-18-0001`, but this sample is not one of its instances.)

`ParsePage` is reached through the router map rather than by import:
`module.json5` declares `"routerMap": "$profile:route_map"`, and
`route_map.json` binds the name `ParsePage` to `parsePageBuilder`. The result
travels back through `pageInfo.pop(this.newVideoFile, true)` into the `onPop`
callback supplied at `pushPath` time, which is how the home grid learns about
a file it never asked for.

**The design decision worth copying is that the sandbox directory is the
model.** There is no catalogue to keep in sync with the disk. `filesDir` is
listed with `fs.listFileSync`, `fs.statSync` supplies size and creation time,
`AVMetadataExtractor` supplies duration and pixel dimensions, and
`AVImageGenerator` supplies the thumbnail; `VideoFile` is a view model
assembled from those four sources and thrown away. Because conversion output
is written into the same `filesDir`, a converted file is automatically a
candidate source for the next conversion, with no registration step. The
`_vec_` filename convention (see `codecChange` below) is what keeps that loop
from degenerating: convert `clip.mp4` to mkv and you get
`clip_vec_<timestamp>.mkv`; convert that to avi and you get
`clip_vec_<timestamp2>.avi`, not `clip_vec_<t1>_vec_<t2>.avi`. The timestamp
keeps names unique, the split that strips a previous suffix keeps them short.

## Implementation steps

1. **Filter the picker to video and take one file**: `PhotoSelectOptions` with
   `MIMEType = PhotoViewMIMETypes.VIDEO_TYPE` and `maxSelectNumber = 1`. No
   permission is required for `PhotoViewPicker.select`; the sample declares
   none, which is correct and worth preserving.
2. **Return quietly when the user cancels.** Cancellation resolves with an
   empty `photoUris`, and the shipped code throws into a `.catch`-less
   `onClick` (`HW-18-0024`). Return `undefined` and let the caller's existing
   `if (!this.sourceVideoFile) return;` handle it.
3. **Copy the picked URI into `filesDir` immediately** with
   `fs.copyFileSync(originFile.fd, destFile.fd)`, closing both handles in a
   `finally`. Everything downstream - probing, playback, ffmpeg - then works on
   a plain sandbox path with no URI lifetime to worry about.
4. **Build `VideoFile` from the filesystem**, not from the picker result:
   `statSync` for size and `ctime`, `AVMetadataExtractor.fetchMetadata` for
   duration and pixel dimensions, `AVImageGenerator.fetchFrameByTime(0, ...)`
   for the thumbnail. Release each probe exactly once (`HW-18-0068`) and key
   the dedup set the same way on both sides (`HW-18-0067`).
5. **Reset the paging flag on every exit path.** `listVideos` returns
   `undefined` when a page contains no new files, and the shipped reset lives
   after that early return, so the flag stays `true` and `onReachEnd` is dead
   for the rest of the session (`HW-18-0066`).
6. **Render the grid with `LazyForEach` over an `IDataSource`,** and call
   `notifyDataReload()` after any list replacement. `VideoFile` is `@Observed`
   and the grid item takes it with `@ObjectLink`, so a thumbnail arriving
   asynchronously repaints that one cell.
7. **Register the AVPlayer listeners before setting `fdSrc`.** The playback
   guide is explicit that `stateChange` and `error` must be attached while the
   player is in the `idle` state, before the resource-setting call; the sample
   calls `setAVPlayerCallback` first and assigns `fdSrc` second.
8. **Drive preparation from the state machine**, not from a promise chain:
   `initialized` sets `surfaceId` and calls `prepare()`, `prepared` reads
   `duration`, seeks one millisecond forward and plays. On teardown, close the
   player's fd only after `release()` has resolved (`HW-18-0068`).
9. **Guard the conversion with `isConverting`,** and swallow both the back
   arrow and `onBackPressed` while it is set - the ffmpeg callback captures
   `this.pageInfo` and would pop a page that has already gone.
10. **Branch on the ffmpeg result code.** `res === 0` means success; anything
    else must delete the partial output with `removeUselessFile(destDir)` and
    toast the failure.

## Verified snippets

All snippets are from `VideoEncodingConvert.zip`. Corrected forms are marked.

**Picking and copying into the sandbox — `entry/src/main/ets/utils/VideoUtils.ets`**
(corrected, see `HW-18-0024`)

```typescript
async selectVideoFile(): Promise<VideoFile | undefined> {   // FIX: was Promise<VideoFile>
  let selectVideoUri: string = '';
  let photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
  photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.VIDEO_TYPE;
  photoSelectOptions.maxSelectNumber = 1;
  let photoPicker = new photoAccessHelper.PhotoViewPicker();

  const RESULT = await photoPicker.select(photoSelectOptions);
  if (RESULT && RESULT.photoUris && RESULT.photoUris.length > 0) {
    let originFile: fs.File | undefined = undefined;
    let destFile: fs.File | undefined = undefined;
    try {
      selectVideoUri = RESULT.photoUris[0];
      originFile = fs.openSync(selectVideoUri, fs.OpenMode.READ_ONLY);
      let destFileDir = this.filesDir + originFile.name;
      destFile = fs.openSync(destFileDir, fs.OpenMode.CREATE | fs.OpenMode.READ_WRITE);
      fs.copyFileSync(originFile.fd, destFile.fd);
      return new VideoFile(originFile.name, destFileDir, CommonConstants.INITIAL_TIME,
        CommonConstants.INITIAL_FILE_SIZE, $r('app.media.ic_default_item_image'), 0);
    } catch (err) {
      throw new Error(`file operate error: ${err.code} ${err.message}`);
    } finally {
      if (originFile) {
        fs.closeSync(originFile);
      }
      if (destFile) {
        fs.closeSync(destFile);
      }
    }
  }
  return undefined;            // FIX: shipped code throws `can not find file` on cancel
}
```

**The `finally` is the part to copy; the `else` is the part to drop.** Both
descriptors are closed on every path including the throw, which is unusual
discipline in this industry's samples - the systematic finding `HW-18-0022`
records five photography projects that open files and never close them, and
this one is not among them. The document's own version of this snippet is
weaker than the shipped code: it has no `finally` at all, so a reader who
copies the page instead of the zip inherits two leaked fds per import.

The cancel path is the opposite story. `photoPicker.select` resolves normally
with `photoUris: []` when the user backs out, which is an ordinary thing to
do, and the shipped `else` turns it into
`throw new Error('can not find file')` inside an `async` arrow passed to
`onClick` - there is no `.catch` anywhere up the chain, so every cancel raises
an unhandled rejection. Returning `undefined` costs one type widening and the
caller already tests for it.

**Listing and probing the sandbox — same file** (corrected, see `HW-18-0067`,
`HW-18-0068`)

```typescript
async listVideos(pageNum: number, pageSize: number, originList: VideoFile[]) {
  let fileList = fs.listFileSync(this.filesDir, { listNum: pageNum * pageSize });
  if (!fileList || fileList.length <= 0) {
    return [];
  }
  let videoFileList: VideoFile[] = [];
  fileList.forEach((value: string) => {
    if (!this.videoSet.has(value)) {                 // keyed by fileName - correct
      videoFileList.push(new VideoFile(value, this.filesDir + value, CommonConstants.INITIAL_TIME,
        CommonConstants.INITIAL_FILE_SIZE, $r('app.media.ic_default_item_image'), 0));
    }
  });
  if (videoFileList.length <= 0) {
    return undefined;                                // see HW-18-0066 at the call site
  }
  videoFileList.forEach((videoFile: VideoFile) => {
    if (this.videoSet.has(videoFile.fileName)) {     // FIX: shipped code tests filePath
      return;
    }
    let stat = fs.statSync(videoFile.filePath);
    videoFile.fileSize = this.getFormattedFileSize(stat.size);
    videoFile.createTime = stat.ctime;
    this.initVideoFile(videoFile);
  });
  // ... sorted by createTime descending, then appended to originList
  return originList;
}

async initVideoFile(videoFile: VideoFile) {
  let avMetadataExtractor: media.AVMetadataExtractor = await media.createAVMetadataExtractor();
  let avImageGenerator: media.AVImageGenerator = await media.createAVImageGenerator();
  let file: fs.File | undefined = undefined;
  try {
    file = fs.openSync(videoFile.filePath, fs.OpenMode.READ_ONLY);
    await this.getVideoMetaData(videoFile, file, avMetadataExtractor);
    await this.getVideoThumbnails(videoFile, file, avImageGenerator);
    this.videoSet.add(videoFile.fileName);
  } catch (e) {
    Logger.error(`Init video file error: %{public}d %{public}s`, e.code, e.message);
  } finally {
    if (file) {
      fs.closeSync(file);
    }
  }
  // FIX: the two release() calls that stood here are duplicates - both helpers
  // already release in their own finally blocks.
}
```

**One file descriptor feeds both probes.** `initVideoFile` opens the video
once and hands the same `fs.File` to the metadata extractor and to the image
generator as `fdSrc`, closing it in a `finally` after both have finished. That
is the right ordering and it is why the two `await`s cannot be run in
parallel here.

The dedup key is where it comes apart. The set is filled with
`videoFile.fileName` at the end of `initVideoFile`, and the first filter pass
queries it with the bare file name, so those two agree. The second pass then
queries `videoSet.has(videoFile.filePath)` - an absolute path that was never
inserted - so the guard never fires and every already-known file is
re-`stat`ed, re-probed and re-thumbnailed on every page load. Either key works
as long as both sides use it. The trailing
`avMetadataExtractor.release()` / `avImageGenerator.release()` are likewise a
second release on instances the two helpers already released in their own
`finally` blocks, returning promises nobody awaits.

**The conversion — `entry/src/main/ets/pages/ParsePage.ets`** (as shipped)

```typescript
codecChange() {
  if (this.targetCodec && this.originVideo) {
    let ctx = this.uiContext.getHostContext();
    if (!ctx) {
      return;
    }
    let codecArray: string[] = CODEC_TABLE.get(this.targetCodec) as string[];
    if (!codecArray || codecArray.length <= 0) {
      showToast(this.getUIContext(), $r('app.string.convert_failed_no_type_msg'));
      return;
    }
    this.isConverting = true;
    let bitCodec = codecArray[0];
    let fileName = new Date().getTime().toString();
    let lastDotIndex = this.originVideo.fileName.lastIndexOf(CommonConstants.DOT_SPLIT_SYMBOL);
    let originName = this.originVideo.fileName.substring(0, lastDotIndex);
    let split = originName.split('_vec_');
    if (split && split.length > 1) {
      originName = split[0];
    }
    fileName = originName.concat('_vec_', fileName);
    let destDir = ctx.filesDir + CommonConstants.FILE_PATH_SEPARATOR + fileName +
      CommonConstants.DOT_SPLIT_SYMBOL + this.targetCodec;
    let cmd = `ffmpeg -i "${this.originVideo.filePath}" -c:v ${bitCodec} -c:a aac -y "${destDir}"`;
    MP4Parser.openNativeLog();
    MP4Parser.ffmpegCmd(cmd, {
      callBackResult: (res) => {
        this.isConverting = false;
        if (res === 0) {
          this.newVideoFile = new VideoFile(fileName + CommonConstants.DOT_SPLIT_SYMBOL +
            this.targetCodec, destDir, CommonConstants.INITIAL_TIME,
            CommonConstants.INITIAL_FILE_SIZE, $r('app.media.ic_default_item_image'), 0);
          this.pageInfo.pop(this.newVideoFile, true);
        } else {
          VideoUtils.removeUselessFile(destDir);
          showToast(this.getUIContext(), $r('app.string.convert_failed_msg'));
        }
      }
    });
  }
}
```

**Three details carry this method.** `CODEC_TABLE` maps each container to the
video codecs it can legally hold - `f4v` to `['h264']` only, `mkv` to four -
and the code takes `codecArray[0]` as the default, so the container the user
picked and the `-c:v` argument can never contradict each other. Both paths in
the command string are quoted, which matters because the file name originates
from the gallery and can contain spaces. `-y` overwrites without prompting,
which is safe here only because the destination name carries a millisecond
timestamp.

`callBackResult` receives an integer and the code branches on it. This is the
contract that `PHOTO-21` ignores - its callback takes no parameter at all, so
a failed ffmpeg run there still copies the output to the gallery and reports
success (`HW-18-0060`). Copy this shape, not that one. The `isConverting` flag
guards more than the spinner: the back arrow's `onClick` and `onBackPressed`
both check it, because `callBackResult` closes over `this.pageInfo` and firing
`pop` from a destination the user has already left is how this kind of page
crashes.

**Player teardown — same file** (corrected, see `HW-18-0068`)

```typescript
async releaseAVPlayer() {
  this.avPlayer?.off('timeUpdate');
  this.avPlayer?.off('error');
  this.avPlayer?.off('stateChange');
  await this.avPlayer?.stop();       // FIX: shipped code does not await
  await this.avPlayer?.release();    // FIX: shipped code does not await
  if (this.vFile) {
    fo.closeSync(this.vFile);        // FIX: shipped code closes this first, before stop()
    this.vFile = undefined;
  }
}
```

**The order is the whole fix.** `fdSrc` hands the player a descriptor the
playback engine keeps reading on its own thread; the shipped
`releaseAVPlayer` closes that descriptor synchronously and only then calls
`stop()` and `release()`, neither awaited, so the engine spends its teardown
reading a closed fd. Deregistering the listeners first, awaiting the two state
transitions, and closing last inverts that. `aboutToDisappear` cannot await
the call, but the ordering inside the async function is what guarantees the
descriptor outlives the player.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions` array, and that is
the correct outcome rather than an oversight: `PhotoViewPicker.select` returns
a temporarily readable URI without any grant, and the sample copies the bytes
out of that URI with `fs` instead of querying `photoAccessHelper.getAssets`.
Nine other samples in this industry declare restricted `READ_IMAGEVIDEO` /
`WRITE_IMAGEVIDEO` they never request (`HW-18-0004`); this one shows what the
same flow looks like without them.

Other configuration that matters: `"routerMap": "$profile:route_map"` with
`route_map.json` binding `ParsePage` to the exported `parsePageBuilder`
(`main_pages.json` lists only `pages/MainPage`); `@ohos/mp4parser` at `2.0.6`
in both the root and the entry `oh-package.json5`; `deviceTypes` of `phone`,
`tablet`, `2in1`; and an `EntryAbility` that pins the app to light mode with
`setColorMode(ColorMode.COLOR_MODE_LIGHT)` before publishing `topRectHeight`
and `bottomRectHeight` into `AppStorage` for the pages to read as
`@StorageProp`. A `dark` resource directory is carried but never used.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` and
  `targetSdkVersion` are both `6.0.0(20)`.
- The conversion is software ffmpeg through `@ohos/mp4parser`, not the
  hardware codec pipeline. It is correspondingly slow on long clips and the
  sample offers no progress percentage, only a spinner - `ffmpegCmd` reports
  once, at the end.
- Only the first codec of each container is ever used. `CODEC_TABLE` lists up
  to four per format, but nothing in the UI selects among them.
- The audio track is always re-encoded to AAC (`-c:a aac`), which is wrong for
  containers that cannot carry it and lossy even when the source was already
  AAC. `-c:a copy` is the better default for a pure re-container.
- The sandbox is never pruned. Every import and every conversion adds a file to
  `filesDir` permanently; there is no delete affordance despite the
  最近删除 (recently deleted) tab in the tab bar. Four of the five tabs are
  inert by design - `tabBuilder`'s `onClick` only acts when
  `tab.id === 'convert'` - and all five `TabContent`s render the same grid.
- `screenWidth` comes from `display.getDefaultDisplaySync().width` in px and is
  compared against a vp2px'd height; the surface sizing therefore assumes a
  full-width window and will be wrong in a resized 2in1 window.

## Pitfalls

- **`HW-18-0066` (B/low, confirmed) — the pagination flag leaks.**
  `onReachEnd` sets `isLoading = true` and resets it inside a `setTimeout`
  that sits *after* `if (!resultList) return;`, while `listVideos` returns
  `undefined` whenever a page contains no new files - the normal case once
  conversions have pre-registered their outputs. One such page and
  `onReachEnd` returns early forever after. Reset the flag before the early
  return, or in a `finally`.
- **`HW-18-0067` (B/low, confirmed) — the dedup guard is dead.** `videoSet` is
  filled with `fileName` and the second pass queries it with `filePath`, so the
  skip never fires and every listed file is re-`stat`ed, re-probed and
  re-thumbnailed on each call. Use one key on both sides.
- **`HW-18-0068` (B/low, confirmed) — double release, and an fd closed too
  early.** `initVideoFile` releases `AVMetadataExtractor` and
  `AVImageGenerator` that `getVideoMetaData` and `getVideoThumbnails` already
  released in their own `finally` blocks, and the second release rejects with
  nobody awaiting it. Separately, `releaseAVPlayer` closes `this.vFile`
  synchronously before the un-awaited `stop()` / `release()`, so teardown
  reads a closed descriptor. Drop the duplicate releases; await stop and
  release, then close.
- **`HW-18-0024` (B/low, confirmed) — the picker cancel path is unhandled**
  across six samples in this industry, this one included:
  `VideoUtils.ets:58-60` throws `can not find file` into a `.catch`-less
  `onClick`. Cancelling the picker is routine; guard on
  `photoUris.length === 0` and return quietly.
  (`ImageRotateAndFlip` in the same industry contains the correct empty-array
  guard and is the reference for the intended pattern.)

Two things that are not findings but will bite: the document's step-1 snippet
is not the shipped code - it has no `finally`, so copying the page rather than
the zip leaks both descriptors on every import - and
`MP4Parser.openNativeLog()` is left enabled on the conversion path.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` - `PhotoViewPicker.select`, `PhotoSelectOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - the AVPlayer state machine, `fdSrc`, `seek`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/04_media/arkts-apis-media-avmetadataextractor.md` - `fetchMetadata`, `duration`, `videoWidth`/`videoHeight`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avmetadataextractor
- `documentation/harmonyos-references/04_media/arkts-apis-media-avimagegenerator.md` - `fetchFrameByTime` and `AVImageQueryOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avimagegenerator
- `documentation/harmonyos-guides/05_media/using-avplayer-for-playback.md` - why `stateChange` and `error` must be attached in the `idle` state, and when to `reset()` versus `release()`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/using-avplayer-for-playback
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `NavPathStack`, `pushPath`, `pop` with a result
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-guides/03_application-framework/arkts-set-navigation-routing.md` - the router map and `buildFunction`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-set-navigation-routing
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - `XComponentController`, `setXComponentSurfaceRect`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-lazyforeach.md` - `IDataSource` and the notify methods
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-lazyforeach
- `PHOTO-21` - the same mp4parser callback, with the result code ignored
