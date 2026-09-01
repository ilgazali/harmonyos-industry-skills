---
id: PHOTO-21
title: Static video watermark - drag ArkUI marks over a Video and burn them in with one ffmpeg overlay chain
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/21_video_water_mark.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_water_mark-0000002408013854
sample: huawei_industry_tree/18_photography/downloads/VideoWatermark.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [dataSharePredicates, fileIo, fs, hilog, image, photoAccessHelper, promptAction, util, window]
permissions: [ohos.permission.READ_IMAGEVIDEO, ohos.permission.WRITE_IMAGEVIDEO]
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0060, HW-18-0061, HW-18-0062, HW-18-0063, HW-18-0064, HW-18-0065, HW-18-0001, HW-18-0003, HW-18-0004, HW-18-0091, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when the user has to **place something on a video by eye and
then burn it in** - a logo, a sticker, a caption, a "shot on" badge. The
pattern is: author the overlay as ordinary ArkUI components stacked over a
`Video`, let a `PanGesture` move them, and at export time flatten every mark
to a PNG in the cache directory and hand the whole set to one ffmpeg
`-filter_complex` chain built by string concatenation.

The transferable idea is the **two-representation split**. While the user is
editing, a mark is a live component - an `Image` or a `TextInput` - so it
reflows, accepts typing and follows the finger for free. At export it becomes
a file: image marks are read out of `resourceManager`, text marks rasterised
with `ComponentSnapshot.get()`. ffmpeg never has to know that two kinds of
mark existed. The same split works for image editors, PDF stamping and
anything where the edit surface is declarative UI and the output comes from a
native encoder.

**This sample composites with the third-party `@ohos/mp4parser` package, not
with a system kit.** No HarmonyOS API burns a bitmap into a video; the feature
rests on the bundled ffmpeg binary. Budget for that dependency. The rest - the
drag surface, the coordinate mapping, the snapshot, the SaveButton export - is
independent of it.

## Feature checklist

- A plus button opens the system photo picker restricted to videos; one video
  can be chosen.
- The chosen video plays in a `Video` with a custom bar: play/pause, elapsed
  and total time, a seek `Slider`, and a fullscreen icon that toasts.
- The component is sized from the real video aspect ratio, so a portrait clip
  fills the height and a landscape clip is letterboxed.
- Two watermark types: 图案 (pattern) opens a 10-sticker grid, 文字 (text)
  drops an editable `TextInput` onto the video.
- Any mark can be dragged; on release it is clamped so its centre stays inside
  the video frame.
- Cancel during editing removes only the marks added in this round; cancel on
  the main bar clears all of them. Confirm rasterises a text mark to
  `cacheDir/textMark{index}.png`.
- A `SaveButton` runs the composition behind a non-cancelable mask, then the
  watermarked mp4 lands in the gallery and a toast confirms it, and the cache
  copies of the input, output and mark PNGs are deleted.

## Architecture

One `entry` module, twelve source files, no view model layer - the page holds
the state and two utility classes hold the work.

```
entry/src/main/ets
├── components
│   ├── MarkComponent.ets      one mark: Image or TextInput, PanGesture drag, clamp on release
│   └── VideoComponent.ets     Video + the mark overlay + the play/seek control bar
├── constants/CommonConstants.ets   sizes, MARK_SIZE, the 1280x720 reference video
├── entryability/EntryAbility.ets   full-screen window, avoid areas -> AppStorage
├── entrybackupability/
├── model/DataModel.ets        MarkModel (@Observed), Cmd, FileProcessResult, MARK_LIST
├── pages/MainPage.ets         @Entry: picker, type chooser, sticker grid, operation bar, SaveButton
└── utils
    ├── FileUtil.ets           picker + getAssets, snapshot packing, write/copy helpers
    ├── Loading.ets            ComponentContent mask, toast, alert
    ├── Logger.ets
    └── WaterMarkUtil.ets      builds the ffmpeg command, runs it, copies the result to the gallery
```

**The documented tree does not match the zip** (`HW-18-0001`). The document
lists `pages/EntrancePage.ets` as the entry page; the zip has only
`pages/MainPage.ets`, which is what `main_pages.json` and `EntryAbility` load.
The document also spells the utility folder `util` where the zip has `utils`.

**The design decision worth copying** is that a mark is never converted until
it has to be. `MarkModel` carries only `id`, `positionX`, `positionY`, `img`
and an optional `message` flag - it does not know what a mark looks like.
`MarkComponent` renders an `Image` when `img` is set and a `TextInput` when
`message` is true, and both branches sit under the same `translate` and the
same `PanGesture`, so drag, clamping and layout are written once. The type
reappears only at the very end, in `Cmd.type`, where it decides whether the
ffmpeg `scale` filter gets a square or a text-shaped box.

The part **not** worth copying is how the geometry travels: component sizes,
video dimensions, aspect ratio and the measured text width are pushed into
`AppStorage` as loose global keys and read back with unchecked `as number`
casts inside `WaterMarkUtil`. A save before the first `onAreaChange` yields
`NaN` coordinates and an ffmpeg command that silently misplaces every mark.
Pass the geometry in as a parameter object instead.

## Implementation steps

1. **Pick the video with `PhotoViewPicker`** and `PhotoViewMIMETypes.VIDEO_TYPE`,
   `maxSelectNumber: 1`. The picker needs no permission at all - the sample's
   own comment says so - so do not add the restricted album permissions the
   sample declares (`HW-18-0004`).
2. **Get the source dimensions without `photoAccessHelper.getAssets`**
   (`HW-18-0003`): it requires `READ_IMAGEVIDEO`, which this sample declares
   but never requests, so the query fails and `selectVideo` returns
   `undefined`. Read the picker URI directly instead.
3. **Size the `Video` from the ratio**: `aspectRatio(width/height)` plus a
   `constraintSize` cap, switching the height between `'100%'` and a scaled
   default so portrait and landscape both fit the 580 vp band.
4. **Stack the marks over the video** with `ForEach` in the same `Stack`, and
   publish the stack's measured size from `onAreaChange` - that size is the
   coordinate space every later conversion is relative to.
5. **Drag with `PanGesture`,** writing `offsetX/offsetY` on update and
   committing the clamped value into `MarkModel` only in `onActionEnd`.
6. **Give every mark a unique component id** - `'textMark' + item.id`, not the
   shared constant the sample uses (`HW-18-0065`) - and snapshot by that id
   with `{ scale: 1, waitUntilRenderFinished: true }`, which is what stops the
   snapshot catching a half-laid-out input.
7. **Convert positions from component space to video space**:
   `(W/2 + positionX - markWidth/2) * videoWidth / W`. The mark origin is the
   frame centre, ffmpeg's is the top left corner.
8. **Build the filter chain by index**: one `-i` per mark, one `scale` per
   mark producing `[scaled_logoN]`, then `overlay` links chained through
   `[tmpN]`. Handle the single-mark case separately - it has no `[tmp]` at all.
9. **Cover the ratio === 1 case** in the scale computation (`HW-18-0063`).
10. **`await` the composition call** so its rejections reach the `catch` that
    closes the loading mask (`HW-18-0061`), and make the handler async.
11. **Check the ffmpeg result code** in `callBackResult` before copying
    anything to the gallery, and close the mask in a `finally` (`HW-18-0060`).
12. **Delete cache files with `unlinkSync`, not `rmdirSync`** (`HW-18-0062`),
    closing the fds first so a throw cannot leak them. **Format durations with
    a modulo on minutes** (`HW-18-0064`).

## Verified snippets

All snippets are from `VideoWatermark.zip`. Corrected forms are marked.

**Picking the video — `entry/src/main/ets/utils/FileUtil.ets`** (as shipped)

```typescript
static async selectVideo(context: Context) {
  let result: FileProcessResult | undefined = undefined;
  try {
    let photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
    photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.VIDEO_TYPE;
    photoSelectOptions.maxSelectNumber = 1;
    let photoPicker = new photoAccessHelper.PhotoViewPicker();
    // PhotoViewPicker的select方法不需要特殊权限即可读取到图片或者视频文件
    let selectResult: photoAccessHelper.PhotoSelectResult = await photoPicker.select(photoSelectOptions);
    if (!selectResult || selectResult.photoUris.length <= 0) {
      return undefined;                                  // the cancel guard other samples omit
    }
    let originFilePath = selectResult.photoUris[0];
    let asset = await FileUtil.getAssets(context, originFilePath);   // needs READ_IMAGEVIDEO
    let width = asset.get(photoAccessHelper.PhotoKeys.WIDTH);
    let height = asset.get(photoAccessHelper.PhotoKeys.HEIGHT);
    result = { filePath: originFilePath, fileWidth: width, fileHeight: height } as FileProcessResult;
  } catch (e) {
    Logger.error(`Read video from albums error: ${e.code} ${e.message}`);
  }
  return result;
}
```

**The empty-array guard is the half of this that is right.** Cancelling the
picker resolves with `photoUris: []`, and this method returns `undefined`
instead of indexing `[0]` - the failure six other photography samples share
(`HW-18-0024`). `MainPage.selectVideoFromAlbums` checks the result before
touching it, so a cancel is a no-op.

**The `getAssets` call underneath it is the half that is wrong.** The comment
two lines above says the picker needs no special permission, and the very next
call reaches for `photoAccessHelper.getAssets`, documented as requiring
`ohos.permission.READ_IMAGEVIDEO` - declared in `module.json5` but never
requested at runtime, and a declared user-grant permission is not a granted
one. The query throws, the surrounding `try` swallows it into a log line, and
the plus button appears to do nothing (`HW-18-0003`). Only the dimensions are
wanted here, and they are reachable from the picker URI without a permission.

**The mark, both kinds — `entry/src/main/ets/components/MarkComponent.ets`**
(corrected, see `HW-18-0065`)

```typescript
@ObjectLink item: MarkModel;
@State offsetX: number = 0;
@State offsetY: number = 0;

build() {
  Row() {
    if (this.item.img) {                            // image mark
      Image(this.item.img)
        .width(CommonConstants.MARK_SIZE)
        .height(CommonConstants.MARK_SIZE)
        .objectFit(ImageFit.Contain);
    }
    if (this.item.message) {                        // text mark
      TextInput({ text: this.drawText, controller: this.controller })
        .onChange((value: string) => { this.drawText = value; })
        .id(`textMark${this.item.id}`)              // FIX: sample hardcodes 'textMark'
        .onAreaChange((_, newV) => {
          AppStorage.setOrCreate('textWidth', newV.width);
        });
    }
  }
  .translate({ x: this.offsetX, y: this.offsetY })  // one transform for both kinds
  .gesture(
    PanGesture()
      .onActionUpdate((event) => {
        if (event) {
          this.offsetX = this.item.positionX + event.offsetX;
          this.offsetY = this.item.positionY + event.offsetY;
        }
      })
      .onActionEnd(() => {
        const BOTTOM = AppStorage.get('height') as number / 2 - CommonConstants.MARK_SIZE / 2;
        const TOP = -BOTTOM;
        const RIGHT = AppStorage.get('width') as number / 2 - CommonConstants.MARK_SIZE / 2;
        const LEFT = -RIGHT;
        if (this.offsetY < TOP) { this.offsetY = TOP; }
        if (this.offsetY > BOTTOM) { this.offsetY = BOTTOM; }
        if (this.offsetX > RIGHT) { this.offsetX = RIGHT; }
        if (this.offsetX < LEFT) { this.offsetX = LEFT; }
        this.item.positionX = this.offsetX;         // commit only once, at the end
        this.item.positionY = this.offsetY;
      })
  );
}
```

**The drag is deliberately two-tier.** `onActionUpdate` writes only the local
`@State` offsets, so a drag in progress repaints this component and nothing
else; `onActionEnd` clamps and then writes back into the `@ObjectLink` model,
which is the value the exporter later reads. Adding `event.offsetX` to
`item.positionX` rather than to the current offset is what makes the gesture
resumable - each pan is measured from where the mark was committed, so
releasing and grabbing again does not jump. `@ObjectLink` against `@Observed
MarkModel` is required, not decorative: `MainPage` owns the array and mutates
fields of individual elements, and only that pair propagates a field write.

**The corrected id line is what makes more than one text mark possible.** As
shipped every `TextInput` carries `.id('textMark')`, and `ComponentSnapshot`
resolves an id to the first matching node - so the second caption a user types
is exported carrying the text of the first. Keying on `item.id`, which
`MainPage.INDEX` already makes unique per mark, is the whole fix; the cache
file name is derived from the same counter.

**Export trigger — `entry/src/main/ets/pages/MainPage.ets`** (corrected, see
`HW-18-0061`, `HW-18-0065`)

```typescript
SaveButton({ text: SaveDescription.SAVE })
  .buttStyle()
  .onClick(async (event: ClickEvent, result: SaveButtonOnClickResult) => {   // FIX: async
    if (result === SaveButtonOnClickResult.SUCCESS && this.marks.length > 0) {
      Loading.open();
      try {
        await WaterMarkUtil.addWaterMark(                                    // FIX: was un-awaited
          this.context.getHostContext()!, this.marks, this.videoPath);
      } catch (error) {
        Loading.close();                                                     // FIX: mask was stuck
        Loading.showAlert(error);
      }
    }
    this.hasMark = false;
    this.marks = [];
  });

// the confirm button rasterises the text mark, same file
this.context.getComponentSnapshot().get(`textMark${MainPage.INDEX}`,         // FIX: was 'textMark'
  (error: Error, pixmap: image.PixelMap) => {
    if (error) { return; }
    FileUtil.savePixelMap(this.context.getHostContext()!, pixmap, MainPage.INDEX);
  }, { scale: 1, waitUntilRenderFinished: true });
```

**`SaveButton` is a security component, and the `result` check is not
optional.** The system grants a one-shot write to the gallery only for the
click it authorised; `SaveButtonOnClickResult.SUCCESS` says the grant is live,
and `createAsset` inside `addWaterMark` depends on it. That is also why the
sample needs no `WRITE_IMAGEVIDEO` at runtime (`HW-18-0004`).

**Un-awaiting an async function inside a `try` is the bug worth internalising.**
As shipped, `addWaterMark` is called without `await`, so the `try` exits
immediately and the `catch` is unreachable: every rejection - `createAsset`,
the source copy, a missing mark resource - becomes an unhandled promise
rejection, and because `Loading.options` sets `autoCancel: false` the user is
left under a mask that cannot be dismissed (`HW-18-0061`). Note also that
`marks` is reset while the composition still runs against the same array: safe
only because `marks = []` rebinds rather than mutating.

**The ffmpeg command and its callback — `entry/src/main/ets/utils/WaterMarkUtil.ets`**
(corrected, see `HW-18-0060`, `HW-18-0062`)

```typescript
static async addWaterMark(context: Context, watermarks: MarkModel[], sourceUri: string) {
  let helper = photoAccessHelper.getPhotoAccessHelper(context);
  let targetUri = await helper.createAsset(photoAccessHelper.PhotoType.VIDEO, 'mp4');

  let getLocalDirPath = context.cacheDir + '/';
  let inputVideoPath = getLocalDirPath + 'inputVideo.mp4';
  FileUtil.copyFile(sourceUri, inputVideoPath);        // picker URI -> sandbox, ffmpeg needs a real path
  let outVideoPath = getLocalDirPath + 'outVideo.mp4';
  FileUtil.writeFile(outVideoPath, '');

  let ffmpegCmd = `ffmpeg -y -i ${inputVideoPath}`;
  let index = 0;
  for (let watermark of watermarks) {
    let cacheWaterMarkPath = '';
    if (watermark.message) {
      cacheWaterMarkPath = getLocalDirPath + `textMark${watermark.id}.png`;   // written by the snapshot
    } else {
      let waterMarkBuffer: ArrayBuffer = WaterMarkUtil.uint8ArrayToBuffer(
        context.resourceManager.getMediaContentSync((watermark.img as Resource).id));
      cacheWaterMarkPath = getLocalDirPath + `waterMark${watermark.id}.png`;
      FileUtil.writeFile(cacheWaterMarkPath, waterMarkBuffer);
    }
    WaterMarkUtil.cmds.push({
      index: ++index,
      path: cacheWaterMarkPath,
      overlay: `overlay=${WaterMarkUtil.getPosition(watermark)}`,
      type: watermark.message ? 1 : 0
    } as Cmd);
  }
  ffmpegCmd = ffmpegCmd.concat(WaterMarkUtil.connectCmd(WaterMarkUtil.cmds), ` -c:a copy ${outVideoPath}`);

  let callBack: ICallBack = {
    callBackResult: (res: number) => {                 // FIX: sample takes no parameter
      let file: fileIo.File | undefined = undefined;
      let targetFile: fileIo.File | undefined = undefined;
      try {
        if (res !== 0) {                               // FIX: failure used to be saved as a success
          Loading.showToast($r('app.string.toast_message_1'));
          return;
        }
        file = fileIo.openSync(outVideoPath, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
        targetFile = fileIo.openSync(targetUri, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
        fileIo.copyFileSync(file.fd, targetFile.fd);
        Loading.showToast($r('app.string.toast_message_2'));
      } catch (err) {
        Logger.error(`createAsset failed, error: ${err.code}, ${err.message}`);
      } finally {
        if (file) { fileIo.closeSync(file.fd); }       // FIX: closes moved above the deletions
        if (targetFile) { fileIo.closeSync(targetFile.fd); }
        WaterMarkUtil.cmds.forEach((cmd) => fileIo.unlinkSync(cmd.path));   // FIX: was rmdirSync
        WaterMarkUtil.cmds = [];
        fileIo.unlinkSync(outVideoPath);               // FIX: was rmdirSync
        fileIo.unlinkSync(inputVideoPath);             // FIX: was rmdirSync
        Loading.close();                               // FIX: was on the success path only
      }
    }
  };
  MP4Parser.ffmpegCmd(ffmpegCmd, callBack);          // shipped code wraps this in try/catch
}
```

**Three things had to move for this callback to be correct.** The result code
has to be read - `ICallBack.callBackResult` is handed the ffmpeg exit status
and the shipped signature drops it, so a failed encode copies whatever
`outVideoPath` contains (an empty file, since `writeFile(outVideoPath, '')`
created it) into the gallery and toasts success (`HW-18-0060`). The cleanup
has to use `unlinkSync`: `rmdirSync` on a regular file raises `ENOTDIR` on the
first mark PNG and aborts the rest of the `finally`, so the cache survives,
both fds leak, `cmds` is never emptied, and the next save composites against
the previous save's stale command list (`HW-18-0062`). And `Loading.close()`
belongs in the `finally`, or any throw strands the mask.

`-c:a copy` is the flag worth noticing: audio is remuxed, not re-encoded, so a
watermark pass costs one video encode and nothing else.

**The scale computation — same file, `connectCmd`** (corrected, see `HW-18-0063`)

```typescript
let ratio = AppStorage.get('aspectRatio') as number;
let scale = 1;
if (ratio >= 1) {                                      // FIX: sample tests ratio > 1, so 1:1 falls through
  scale = AppStorage.get('videoWidth') as number / CommonConstants.VIDEO_WIDTH * CommonConstants.MARK_SIZE_PX;
} else {
  scale =
    CommonConstants.MARK_SIZE / CommonConstants.VIDEO_COMP_HEIGHT_MAX * (AppStorage.get('videoHeight') as number);
}
```

The mark is authored at a fixed 40 vp on screen and must come out proportional
in the encoded frame, so the scale is derived from whichever video dimension
was the constrained one: width for landscape (against the 1280 px reference),
height for portrait (against the 580 vp component band). A square video
matches neither strict comparison, `scale` keeps its initialiser of `1`, and
ffmpeg scales the watermark to one pixel - invisible, on exactly the aspect
ratio social platforms use most.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.READ_IMAGEVIDEO",  "reason": "$string:reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" } },
  { "name": "ohos.permission.WRITE_IMAGEVIDEO", "reason": "$string:reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" } }
]
```

**Delete both entries when you copy this sample** (`HW-18-0004`). They are
restricted (ACL) permissions an ordinary app cannot ship with, and this sample
needs neither: reading goes through `PhotoViewPicker`, whose `select` grants
access to the returned URI on its own, and writing goes through `SaveButton`,
which carries a one-shot grant from the user's click. Nine photography samples
share this boilerplate `module.json5`, and this one contradicts itself out
loud - `FileUtil.ets:39` comments that the picker 不需要特殊权限 (needs no
special permission) six lines before the code depends on a declared one.
`READ_IMAGEVIDEO` is also insufficient rather than merely unnecessary: never
requested at runtime, so `getAssets` fails anyway (`HW-18-0003`).

Dependency: `@ohos/mp4parser` `^2.0.6` in the root `oh-package.json5`. Device
types `phone`, `tablet`, `2in1`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- The composition is done by the bundled ffmpeg in `@ohos/mp4parser`. No
  progress callback is used - the mask is indeterminate for the whole encode.
- `WaterMarkUtil.cmds` is a **static** array cleared only in the callback's
  `finally`: two overlapping saves interleave into one command list.
- Coordinates travel through `AppStorage` globals populated by `onAreaChange`.
  Saving before the video component has been measured produces `NaN` offsets,
  and nothing checks for it.
- The text mark is snapshotted at `scale: 1`, i.e. screen density, then scaled
  up by ffmpeg to video resolution: a caption over a 4K clip will be soft.
- `EntryAbility` registers `avoidAreaChange` without ever releasing it and
  drops the promise from `setWindowLayoutFullScreen`.
- The app forces `COLOR_MODE_LIGHT` in `onCreate`; the fullscreen button is a
  toast. The control bar is a demonstration, not a player.

## Pitfalls

- **`HW-18-0060`** (B/medium, confirmed): `callBackResult` takes no result
  parameter, so a failed ffmpeg run still copies `outVideo.mp4` to the gallery
  and toasts success, and `Loading.close()` sits on the success path only. Fix:
  accept the code, return early when non-zero, close the mask in `finally`.
- **`HW-18-0061`** (B/medium, confirmed): `addWaterMark` is called without
  `await` inside a `try/catch`, so every rejection before ffmpeg starts escapes
  the `catch` and strands the non-cancelable mask. Fix: `await` the call and
  make the handler `async`.
- **`HW-18-0062`** (B/low, probable): the cleanup `finally` calls `rmdirSync`
  on regular files; the first call throws `ENOTDIR` and aborts the rest, so the
  cache survives, both fds leak, and the static `cmds` array is never reset.
  Fix: `unlinkSync` for files, and close the fds before deleting.
- **`HW-18-0063`** (B/low, confirmed): an aspect ratio of exactly 1 matches
  neither `ratio > 1` nor `ratio < 1`, so `scale` keeps its initial `1` and the
  watermark is one pixel square. Fix: use `>=`, or an explicit equality branch.
- **`HW-18-0064`** (B/low, confirmed): `timeFormat` divides out hours but takes
  no modulo on minutes, so 3661 s renders as `01:61:01`. Fix:
  `Math.floor(time / 60) % 60`.
- **`HW-18-0065`** (B/low, probable): every text mark's `TextInput` carries the
  same `.id('textMark')` and `MainPage` snapshots by that id, so the second
  caption's cached PNG holds the first caption's text and the exported video
  shows the wrong content. Fix: append the mark's `id` to the component id.
- **`HW-18-0001`** (E/low, confirmed): systematic - five photography project
  trees do not match their zips. This one documents `pages/EntrancePage.ets`
  where the zip has only `pages/MainPage.ets` (and `util` for `utils`). Fix:
  regenerate the 工程目录 blocks from the current zips.
- **`HW-18-0003`** (B/medium, confirmed): `FileUtil.getAssets` queries
  `phAccessHelper.getAssets` on the picker URI, which requires
  `READ_IMAGEVIDEO`; declared but never requested at runtime, so the call fails
  and the video's dimensions never arrive - same pattern as `PHOTO-09`. Fix:
  read the dimensions from the picker URI with file/media APIs instead.
- **`HW-18-0004`** (D/medium, confirmed): systematic - nine photography samples
  declare restricted `READ`/`WRITE_IMAGEVIDEO` although built around the
  permission-free picker + `SaveButton` flow. Fix: delete both entries.

## References

- `huawei_industry_tree/18_photography/docs/21_video_water_mark.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_water_mark-0000002408013854
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` - `PhotoViewPicker.select`, `PhotoSelectOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `createAsset`, `getAssets` and its permission requirement
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-references/02_application-framework/ts-security-components-savebutton.md` - `SaveButtonOnClickResult` and the one-shot grant
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-savebutton
- `documentation/harmonyos-references/02_application-framework/ts-media-components-video.md` - `Video`, `VideoController`, `onPrepared` / `onUpdate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-media-components-video
- `documentation/harmonyos-references/02_application-framework/ts-container-stack.md` - the overlay container
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-stack
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - `onActionUpdate`, `onActionEnd`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-componentsnapshot.md` - `get()`, `scale`, `waitUntilRenderFinished`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-componentsnapshot
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagepacker.md` - `packToData` for the snapshot PNG
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagepacker
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - why `READ`/`WRITE_IMAGEVIDEO` cannot simply be declared
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
- `PHOTO-09` - the same `getAssets`-without-a-grant defect
- `PHOTO-22` - the sibling mp4parser sample, whose callback does check `res === 0`
