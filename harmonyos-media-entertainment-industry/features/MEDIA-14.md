---
id: MEDIA-14
title: Video segment to GIF - a frame-strip range picker over AVPlayer, then an ffmpeg command through mp4parser
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/14_video_demo_videocreategif.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_demo_videocreategif-0000002312993113
sample: huawei_industry_tree/13_media_entertainment/downloads/demo_VideoCreateGIF.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [fileUri, fs, hilog, image, media, photoAccessHelper, window]
min_api: 20
modules: [entry (entry)]
findings: [HW-13-0039, HW-13-0040, HW-13-0041, HW-13-0033, HW-13-0097, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card for the **"clip this bit and export it" flow**: a player, a
frame strip under it with two draggable handles, and an export step that
produces a new file. The demo makes a GIF, but the machinery is
format-agnostic - the same three screens give you a trimmed MP4, a ringtone
cut out of a song, or an animated sticker.

Three techniques are worth taking away independently. **Frame-strip
scrubbing**: `MP4Parser.getFrameAtTimeRang` decodes one thumbnail per second
in a worker and streams them back through a callback, so the strip fills in
progressively instead of blocking the page. **Range clamping**: the parent
page computes the selectable window (`startTime`/`endTime`) and the initially
selected sub-window (`currentStartTime`/`currentEndTime`) from the current
playhead before it hands control to the picker, so the child never has to
know about the video's duration. **Save-to-gallery without a permission**:
`showAssetsCreationDialog` gives the app a write URI into the media library on
the strength of a user confirmation, so no `WRITE_IMAGEVIDEO` declaration is
needed - and indeed this sample declares no permissions at all.

**Read `HW-13-0040` before adopting the progress gating.** The sample makes a
single `canNext` boolean govern both the forward button and the back button,
so a decode failure locks the user inside the screen.

## Feature checklist

- The bundled video is copied out of `resources` into the sandbox on first
  load and played through an `XComponent` surface.
- A slider and a `mm:ss` / `mm:ss` time pair track playback.
- A 裁GIF button pauses the video and opens the range picker at the current
  playhead.
- The picker shows a horizontal strip of one thumbnail per second, filling in
  as frames decode, with a placeholder in each slot until then.
- Two handles select a sub-range between 5 s and 10 s; the label reads
  已选择Ns,最多裁剪10s.
- The preview loops the selected range only - it seeks back to the start
  whenever playback passes the end handle.
- While frames are still decoding, both buttons answer with a
  视频处理中，请等候 toast.
- The next step runs an ffmpeg command and shows the resulting GIF.
- Save copies the GIF into the system gallery via a confirmation dialog.

## Architecture

One `entry` module, three stacked full-screen views inside a single `@Entry`
page. There is no router: the two later screens are `if`-gated components
drawn over the first inside one `Stack`.

```
entry/src/main/ets
├── common
│   ├── Constants.ets                  every numeric literal, including the 5/10/20 s trim window
│   ├── Object.ets                     ThumbContent, VideoThumbOption, RangeSeekBarListener, TouchType
│   └── Utils.ets                      getTimeString(ms) -> "mm:ss" / "hh:mm:ss"
├── entryability/EntryAbility.ets      immersive full-screen layout
└── pages
    ├── VideoCreateGIF.ets             @Entry: the player, the transport bar, the two child views
    ├── SelectGifTimeFrameView.ets     the range picker: second player + frame strip + seek bar
    ├── RangeSeekBarView.ets           the two handles and the progress overlay, drawn on a Canvas
    └── GifCreateView.ets              runs ffmpegCmd, previews the GIF, saves it to the gallery
```

The documented tree matches the zip.

**The design decision worth copying** is that `SelectGifTimeFrameView` takes
its whole world as `@Prop`s - `srcFilePath`, `maxTime`, `minTime`,
`startTime`, `endTime`, `currentStartTime`, `currentEndTime`, `videoWidth`,
`videoHeight` - and returns its result through two callbacks (`back`,
`startCreateGif`). It never touches the parent's player and never queries the
video. That is what makes it reusable: the same component clips a ringtone if
you feed it different bounds. The parent's `updateTruncateTime()` is the only
code that knows the trim window is "10 s back, 20 s forward, 10 s selected".

**The decision worth avoiding** is the second `AVPlayer`. The picker creates
its own player on its own `XComponent` surface while the parent's player is
merely paused, so two decoders are alive at once over the same file. On a
device with a single hardware decoder path that is a real risk; pass the
surface down or stop the parent player instead of pausing it.

## Implementation steps

1. **Materialise the video in the sandbox.** `resourceManager.getMediaContentSync($r('app.media.video').id).buffer`
   into a `filesDir` path opened `READ_WRITE | CREATE`. mp4parser's ffmpeg
   commands need a real path, not a `$rawfile` reference.
2. **Bind the player to the surface in `XComponent.onLoad`**, where
   `getXComponentSurfaceId()` is first valid, then set `url = 'fd://' + fd'`
   and let `stateChange` do `initialized -> surfaceId + prepare -> prepared ->
   play`.
3. **Compute the trim window from the playhead before pushing the picker**,
   clamping `prevTime` at 0 and `nextTime` at `duration`.
4. **Decode the frame strip in two stages**: `MP4Parser.setDataSource(path,
   cb)` first, and only from inside its success callback
   `getFrameAtTimeRang(startUs, endUs, OPTION_CLOSEST, frameCb)`.
5. **Count failures as well as successes** when deciding the strip is
   complete, and never gate the back button on it (`HW-13-0040`).
6. **Release each `ImageSource` after `createPixelMap`** - the strip decodes
   one source per second of video and they are not garbage-collected for you.
7. **Stop the frame worker in `aboutToDisappear`** with
   `MP4Parser.stopGetFrame()` before releasing the player, otherwise the
   callback fires into a destroyed component.
8. **Loop the selection in `timeUpdate`**: when `currentTime >=
   endTruncationTime`, `play()` then `seek(start)`.
9. **Branch on the ffmpeg result code before enabling Save** - the sample
   enables it either way and only changes the toast (`HW-13-0041`).
10. **Guard every handle in the save `finally`** (`HW-13-0039`), and **pause
    the player in `onPageHide`** rather than only flipping the UI flag
    (`HW-13-0033`).

## Verified snippets

All snippets are from `demo_VideoCreateGIF.zip`. Corrected forms are marked.

**Deriving the trim window — `entry/src/main/ets/pages/VideoCreateGIF.ets`** (as shipped)

```typescript
private updateTruncateTime(): void {
  let playTime = Math.floor(this.currentTime / Constants.TIMESTAMP_1000) * Constants.TIMESTAMP_1000;

  // 当前播放时间向前10秒作为最小可截取到的时间，
  // 当前播放时间向后20秒作为最大可截取到的时间，
  // 当前播放时间到向后10秒作为当前选中的截取时间
  let prevTime = playTime - Constants.TIME_GIF_BEFORE_10 * Constants.TIMESTAMP_1000;
  let nextTime = playTime + Constants.TIME_GIF_AFTER_20 * Constants.TIMESTAMP_1000;
  if (prevTime < 0) {
    prevTime = 0;
  }
  if (nextTime > this.avPlayer!.duration) {
    nextTime = this.avPlayer!.duration;
  }

  let currentPrevTime = playTime;
  if (this.currentTime === this.avPlayer!.duration) {          // at the very end, back up 5 s
    currentPrevTime = this.avPlayer!.duration - Constants.TIME_GIF_MIN_5 * Constants.TIMESTAMP_1000;
    if (currentPrevTime < 0) {
      currentPrevTime = 0;
    }
  }
  let currentNextTime = currentPrevTime + Constants.TIME_GIF_MAX_10 * Constants.TIMESTAMP_1000;
  if (currentNextTime > this.avPlayer!.duration) {
    currentNextTime = this.avPlayer!.duration;
  }

  this.startTime = prevTime;                 // strip bounds
  this.endTime = nextTime;
  this.currentStartTime = currentPrevTime;   // initial handle positions
  this.currentEndTime = currentNextTime;
}
```

**Two ranges, not one.** `startTime`/`endTime` is the 30-second window the
frame strip covers; `currentStartTime`/`currentEndTime` is the 10-second
selection inside it. Keeping them separate is what lets the user drag the
handles without re-decoding the strip, and it is why the picker needs no
knowledge of the video at all - both ranges arrive as props.

The `Math.floor(... / 1000) * 1000` at the top snaps the playhead to a whole
second, which matters because the strip has exactly one thumbnail per second
and the handle positions are indexed against it.

**The frame strip — `entry/src/main/ets/pages/SelectGifTimeFrameView.ets`** (corrected, see `HW-13-0040`)

```typescript
private initThumbList(): void {
  let videoThumbs: ThumbContent[] = [];
  for (let i = this.startTime; i < this.endTime; i = i + Constants.TIMESTAMP_1000) {
    videoThumbs.push(new ThumbContent());          // one placeholder slot per second
  }
  this.videoThumbCount = videoThumbs.length;
  let videoThumbListTopPadding = Constants.THUMB_TOP_PADDING_14;
  let count = 0;

  let callBack: ICallBack = {
    callBackResult: (code: number) => {
      if (code !== 0) {
        this.canNext = true;                       // FIX: never strand the user on a source failure
        return;
      }
      let frameCallBack: IFrameCallBack = {
        callBackResult: async (data: ArrayBuffer, timeUs: number) => {
          let imageSource: image.ImageSource = image.createImageSource(data);
          let videoThumbWidth = this.uiContext.vp2px((this.seekBarWidth - 2 * this.thumbWidth) /
          this.videoThumbCount);
          let videoThumbHeight = this.uiContext.vp2px(this.seekBarHeight - videoThumbListTopPadding + 1);
          if (videoThumbWidth / this.videoWidth > videoThumbHeight / this.videoHeight) {
            videoThumbHeight = videoThumbWidth * this.videoHeight / this.videoWidth;
          } else {
            videoThumbWidth = videoThumbHeight * this.videoWidth / this.videoHeight;
          }
          let decodingOptions: image.DecodingOptions = {
            sampleSize: Constants.SAMPLE_SIZE_1,
            editable: true,
            desiredSize: { width: videoThumbWidth, height: videoThumbHeight },
            desiredPixelFormat: image.PixelMapFormat.RGBA_8888
          };
          imageSource.createPixelMap(decodingOptions).then((px: image.PixelMap) => {
            let framePos = timeUs / Constants.TIMESTAMP_1000 / Constants.TIMESTAMP_1000;
            videoThumbs[framePos].pixelMap = px;
            this.updateList(videoThumbs, this.videoThumbCount);
          }).catch(() => {
            hilog.error(Constants.DOMAIN, 'testTag', '%{public}s', 'thumb decode failed');
          }).finally(() => {
            imageSource.release();                 // FIX: sample releases only on the success path
            count++;                               // FIX: sample counts successes only
            if (count === this.videoThumbCount) {
              this.canNext = true;
            }
          });
        }
      };
      MP4Parser.getFrameAtTimeRang(
        this.startTime * Constants.TIMESTAMP_1000 + '',   // mp4parser takes microseconds, as strings
        this.endTime * Constants.TIMESTAMP_1000 + '',
        MP4Parser.OPTION_CLOSEST,
        frameCallBack
      );
    }
  };
  MP4Parser.setDataSource(this.srcFilePath, callBack);
  this.updateList(videoThumbs, this.videoThumbCount);      // draw the placeholders immediately
}
```

**The placeholder array is allocated before a single frame is decoded**, and
`updateList` is called once at the end of the function - that is what makes
the strip appear at full width with grey slots and fill in progressively.
`updateList` also recomputes `videoThumbWidth` as `(seekBarWidth - 2 *
thumbWidth) / videoThumbCount`, so the strip always exactly spans the seek bar
minus the two handles.

The two nested callbacks are not decoration: `setDataSource` is what opens and
parses the container, and asking for frames before its callback reports 0
fails. The API is microsecond-based and takes its bounds **as strings**, which
is why both arguments end in `+ ''`.

`canNext` is where the sample traps the user (`HW-13-0040`).
`createPixelMap` has no `.catch`, `count` only increments on success, and
`canNext` requires `count === videoThumbCount`. One failed decode - and a
startup race makes that realistic, because `seekBarWidth` is still 0 until
`onAreaChange` fires, which can produce a negative `desiredSize` - leaves the
page permanently in the 视频处理中，请等候 state, with **both** the next
button and the back button refusing. A back button should never be gated on
work in progress.

**Generating the GIF — `entry/src/main/ets/pages/GifCreateView.ets`** (corrected, see `HW-13-0041`)

```typescript
createGif(srcFilePath: string, sTime: number, eTime: number): void {
  let dst = this.uiContext?.getHostContext()?.cacheDir + '/output' + Date.now() + '.gif';
  let startTime = Utils.getTimeString(sTime);                            // "mm:ss" for -ss
  let duration = Math.floor((eTime - sTime) / Constants.TIMESTAMP_1000); // seconds for -t

  let that = this;
  let callBack: ICallBack = {
    callBackResult(code: number) {
      if (code === 0) {                                 // FIX: sample sets these unconditionally
        that.gifSandBoxPath = dst;
        that.gifFilePath = fileUri.getUriFromPath(dst);
        that.canNext = true;
      }
      that.uiContext.getPromptAction().showToast({ message: code === 0 ? '生成GIF成功' : '生成GIF失败' });
    }
  };

  MP4Parser.ffmpegCmd('ffmpeg -i ' + srcFilePath + ' -ss ' + startTime + ' -t ' + duration + ' ' + dst, callBack);
}
```

**The command is a plain ffmpeg invocation and the output extension decides
the format** - `.gif` is why this produces a GIF; change `dst` to `.mp4` and
the same three screens become a video trimmer. `-ss` before `-i` would be a
fast seek; here it comes after, which is the accurate-seek form and the right
choice for a user-selected range.

`Date.now()` in the filename is doing real work: `cacheDir` is not cleared
between attempts, so a fixed name would let the `Image` component show a
stale, cached GIF from the previous run.

The shipped `callBackResult` sets `gifSandBoxPath`, `gifFilePath` and
`canNext` before it looks at `code` (`HW-13-0041`), so a failed generation
still enables Save and still points it at a path that was never written. Only
the toast differs - and pressing Save then walks straight into the
`closeSync(undefined)` fault below.

**Saving to the gallery — same file** (corrected, see `HW-13-0039`)

```typescript
.onClick(async () => {
  try {
    let srcFileUris: Array<string> = [this.gifFilePath];
    let photoCreationConfigs: Array<photoAccessHelper.PhotoCreationConfig> = [{
      title: 'test',
      fileNameExtension: 'gif',
      photoType: photoAccessHelper.PhotoType.IMAGE,
      subtype: photoAccessHelper.PhotoSubtype.DEFAULT
    }];
    let phAccessHelper = photoAccessHelper.getPhotoAccessHelper(this.uiContext?.getHostContext());
    // the dialog IS the permission: the user's confirmation returns a writable media-library uri
    let desFileUris: Array<string> = await phAccessHelper.showAssetsCreationDialog(srcFileUris, photoCreationConfigs);
    this.desFile = await fsUtils.open(desFileUris[0], fsUtils.OpenMode.WRITE_ONLY);
    this.srcFile = await fsUtils.open(this.gifFilePath, fsUtils.OpenMode.READ_ONLY);
    await fsUtils.copyFile(this.srcFile.fd, this.desFile.fd);
    this.uiContext.getPromptAction().showToast({ message: '保存成功' });
  } catch (err) {
    hilog.error(Constants.DOMAIN, 'testTag', '%{public}s',
      `failed to create asset by dialog successfully errCode is: ${err.code}, ${err.message}`);
  } finally {
    if (this.srcFile) {                       // FIX: sample calls closeSync(undefined) on refusal
      fsUtils.closeSync(this.srcFile);
    }
    if (this.desFile) {                       // FIX: same, and this one leaks when only src opened
      fsUtils.closeSync(this.desFile);
    }
    this.srcFile = undefined;
    this.desFile = undefined;
  }
})
```

**`showAssetsCreationDialog` is the reason this sample needs no permissions.**
It hands the user's confirmation a write URI into the media library, which the
app then opens `WRITE_ONLY` and copies into. There is no
`ohos.permission.WRITE_IMAGEVIDEO` in `module.json5` and none is needed - the
dialog is the grant. The cost is that refusal is a normal, frequent outcome.

Which is exactly what the shipped `finally` cannot survive (`HW-13-0039`).
When the user dismisses the dialog, `showAssetsCreationDialog` rejects,
neither `open` has run, both `srcFile` and `desFile` are `undefined`, and
`fsUtils.closeSync(undefined)` throws a `TypeError` out of the `finally` -
replacing the handled refusal with an unhandled error. The same shape recurs
in `MergeVideo`, `MusicPlayer`, `demo_HttpAudioRender` and
`AudioVisualization`; guard each handle individually, because the middle case
(source opened, destination not) leaks the one that did open.

## Permissions & config

**None.** The sample's `module.json5` declares no `requestPermissions` block
at all.

That is a deliberate consequence of two choices. The source video is bundled
in `resources/base/media` and copied into the app's own sandbox, so no file
picker and no read permission are involved. The output goes to the gallery
through `showAssetsCreationDialog`, whose per-file confirmation replaces
`ohos.permission.WRITE_IMAGEVIDEO`. Any app that follows this pattern inherits
the same clean manifest - worth preserving when you adapt it.

The one external dependency is `@ohos/mp4parser` from ohpm, which carries a
native ffmpeg build. Check its licence and its ABI coverage before shipping;
it is not part of the SDK.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The clip is bounded to 5-10 s by `TIME_GIF_MIN_5` / `TIME_GIF_MAX_10`. GIF
  has no inter-frame compression to speak of, so a longer clip at full
  resolution produces a file measured in tens of megabytes.
- The ffmpeg command passes no `-vf scale`, `-r` or palette filter, so the GIF
  comes out at the source resolution and frame rate. Production use wants at
  minimum a `fps` and `scale` filter.
- Two `AVPlayer` instances exist while the range picker is open; the parent's
  is only paused.
- `onPageHide` sets `isPlaying = false` without pausing the player, in both
  `VideoCreateGIF` and the picker (`HW-13-0033`), so backgrounding leaves audio
  running and the transport icon lying about it.
- The share and comment buttons on the result screen are icons with no
  handlers.
- The frame strip decodes one full-size `ImageSource` per second of the
  30-second window on the UI thread's microtask queue; on a long window this
  is visible jank.

## Pitfalls

- **`HW-13-0039`** (B/medium, confirmed) — systematic across five media
  samples: the save `finally` calls `fsUtils.closeSync(this.srcFile)` and
  `closeSync(this.desFile)` unguarded, so a refused or cancelled
  `showAssetsCreationDialog` throws a `TypeError` from the cleanup path and
  masks the real outcome; the partial case leaks the handle that did open.
  Fix: `if (f) closeSync(f)` per handle.
- **`HW-13-0040`** (B/medium, probable) — `canNext` gates both the next button
  and the **back** button, and only increments on a successful
  `createPixelMap`, which has no `.catch`. A single failed thumbnail decode -
  realistic, since `seekBarWidth` is 0 until `onAreaChange` fires and can yield
  a negative `desiredSize` - leaves the page stuck in 视频处理中，请等候 with
  no way out. Fix: count failures too, and never gate back.
- **`HW-13-0041`** (B/low, confirmed) — `callBackResult` assigns
  `gifSandBoxPath`, `gifFilePath` and `canNext` before checking `code`, so a
  failed generation still enables Save on a file that does not exist. Fix:
  branch on `code` first.
- **`HW-13-0033`** (B/medium, confirmed) — systematic across three samples:
  `onPageHide` only flips `isPlaying`, it never pauses the player. Audio keeps
  playing in the background and the transport icon shows the wrong state on
  return. Fix: pause in the background hook and propagate the real player
  state to the UI.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - the state machine, `surfaceId`, `seek`, `timeUpdate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - `XComponentType.SURFACE`, `getXComponentSurfaceId`, `onLoad`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagesource.md` - `createImageSource`, `DecodingOptions`, `createPixelMap`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagesource
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `showAssetsCreationDialog` and `PhotoCreationConfig`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-guides/03_application-framework/save-user-file.md` - the dialog-authorised save-to-gallery flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/save-user-file
- `documentation/harmonyos-guides/02_media/video-playback.md` - the AVPlayer video playback walkthrough
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/video-playback
- `MEDIA-13` - the same `finally`-closeSync defect (`HW-13-0039`) in the music importer
- `MEDIA-11` - the same `onPageHide` flag-only defect (`HW-13-0033`)
