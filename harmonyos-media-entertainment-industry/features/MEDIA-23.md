---
id: MEDIA-23
title: Extract the audio track from a picked video - PhotoViewPicker in, ffmpeg stream-copy through the sandbox, DocumentViewPicker out
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/23_audio_extractor.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio_extractor-0000002298797484
sample: huawei_industry_tree/13_media_entertainment/downloads/视频中提取音频示例代码.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: ["photoAccessHelper.PhotoViewPicker", PhotoSelectOptions, PhotoViewMIMETypes, "photoAccessHelper.getPhotoAccessHelper", getAssets, getFirstObject, getThumbnail, PhotoKeys, dataSharePredicates, "picker.DocumentViewPicker", DocumentSaveOptions, "MP4Parser.ffmpegCmd", ICallBack, "fs.openSync", "fs.copyFileSync", "fs.closeSync", Navigation, NavPathStack, NavDestination, routerMap, "util.format", showToast]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-13-0057, HW-13-0058, HW-13-0097, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when the app must **run a media transform on a file the user
picked from the gallery and hand the result back to the user's storage** -
extract the soundtrack from a clip, transcode, trim, mux. Audio extraction is
the example; the pipeline is the point.

The pipeline has four hops and each one exists for a reason. `PhotoViewPicker`
returns a URI the app may read but which is **not** a sandbox path and cannot
be handed to a native library. So the video is copied into `context.filesDir`
first, ffmpeg reads and writes sandbox paths only, and the output is copied out
through `DocumentViewPicker.save`, which is what puts the file somewhere the
user can find it. Nothing in that chain requires a permission - the two pickers
are the permission, which is the whole reason to prefer them.

Read both findings before adopting this. `HW-13-0057` means the sample reports
success on every ffmpeg failure, and `HW-13-0058` is the more instructive one:
the code reaches for `getAssets` to fetch a thumbnail, which *does* need
`ohos.permission.READ_IMAGEVIDEO`, and the sample declares no permissions at
all. That is the trap of a picker-based design - one convenience call and you
have silently left the permission-free path.

## Feature checklist

- A four-tab home page (主页 / 模板 / 消息 / 我的) with a 开始创作 (start
  creating) card at the top of the first tab.
- Tapping the card opens the system photo picker filtered to video, single
  selection.
- Selecting a video pushes a `VideoEditPage` through a `NavPathStack`, carrying
  the URI as a typed route parameter.
- The edit page previews the video with a custom play/pause control and a
  `mm:ss / mm:ss` time readout that switches to `hh:mm:ss` past an hour.
- The timeline strip shows the video's first frame as three thumbnails.
- 导出 (export) runs the extraction and, on success, opens the document save
  picker pre-filled with `<name>_SplitAudio_<yyyyMMdd_HHmmss>.aac`.
- Confirming the save copies the .aac to the chosen location and toasts
  保存成功.
- Backgrounding the app clears the playing flag so the control shows the play
  icon on return.

## Architecture

One `entry` module, two pages joined by a `Navigation` route map.

```
entry/src/main/ets
├── component/CreativeComponent.ets     the 开始创作 card, pure layout
├── entryability/EntryAbility.ets       full screen, avoid areas, isForeGround flag
├── entrybackupability/EntryBackupAbility.ets
├── model/VideoParams.ets               { sourceVideoUri?: string } - the route payload
└── pages
    ├── MainPage.ets                    @Entry, tabs + the picker call (268 lines)
    └── VideoEditPage.ets               preview, extract, save, thumbnail (482 lines)
```

Plus `resources/base/profile/route_map.json`, which is what makes
`pushPath({ name: 'VideoEditPage' })` resolve, and `@ohos/mp4parser ^2.0.7-rc.1`
declared in the **root** `oh-package.json5` (the `entry` module's own
`oh-package.json5` has an empty `dependencies` block - the import works through
hoisting, but the declaration belongs at the module that imports it).

The documented tree matches the zip.

**The design decision worth copying** is the routing contract. `VideoEditPage`
is not an `@Entry` page and is not listed in `main_pages.json`; it is a
`NavDestination` exported through a named `@Builder`:

```typescript
@Builder
export function videoEditPageBuilder() {
  VideoEditPage();
}
```

registered in `route_map.json` against `buildFunction: "videoEditPageBuilder"`.
The payload is a declared interface, `VideoParams`, not a bag of strings, and
the destination reads it in `onReady` from `context.pathInfo.param`. That gives
a typed, lazily-loaded, back-stack-aware page transition with no global state -
the right default for any "pick something, then work on it" flow.

**Worth avoiding in the same file:** `VideoEditPage.onReady` copies the source
video into the sandbox, and then the 导出 button's `onClick` copies it *again*
before every extraction. The second copy is redundant, synchronous, and on the
UI thread. `copyFileSync` on a phone-camera clip is not free.

## Implementation steps

1. **Pick with `PhotoViewPicker`,** setting `MIMEType = PhotoViewMIMETypes.VIDEO_TYPE`
   and `maxSelectNumber = 1`. This needs no permission and no
   `photoAccessHelper` instance.
2. **Guard the empty result explicitly** - a cancelled picker resolves with an
   empty `photoUris`, it does not reject.
3. **Push the URI as a typed route param** through `NavPathStack.pushPath`, and
   register the destination in `route_map.json`.
4. **Copy the picked URI into `context.filesDir` before touching ffmpeg.**
   The picker URI is readable but is not a sandbox path; a native library
   cannot open it. Do this once, in `onReady`, not per export.
5. **Name the output deterministically** - `<stem>_SplitAudio_<timestamp>.aac` -
   so repeated exports of the same clip never collide.
6. **Run `ffmpeg -i <in> -c:a copy -vn <out> -y`.** `-c:a copy` is a stream
   copy: no re-encode, so the extraction is I/O-bound and the audio is
   bit-identical to the track inside the container. `-vn` drops video, `-y`
   overwrites.
7. **Branch on the callback's `code`** (`HW-13-0057`). The shipped
   `callBackResult` ignores its parameter and gates on a path string that is
   always truthy after the first click.
8. **Do not call `getAssets` for the thumbnail** unless
   `ohos.permission.READ_IMAGEVIDEO` is declared *and* requested
   (`HW-13-0058`). A picker URI grants read access to that file; it does not
   grant the right to query the media library.
9. **Save through `DocumentViewPicker`,** pre-filling `newFileNames`, and copy
   the sandbox file to each returned URI.
10. **Close both descriptors in a `finally`,** guarded on non-null - the
    sample's `copyFile` is the correct shape for this.

## Verified snippets

All snippets are from `视频中提取音频示例代码.zip` (`AudioExtractor`).
Corrected forms are marked.

**Picking the video — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
import { photoAccessHelper } from '@kit.MediaLibraryKit';

private async selectVideoFile(): Promise<string> {
  let selectVideoUri: string = '';
  try {
    let photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
    photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.VIDEO_TYPE;
    photoSelectOptions.maxSelectNumber = 1;
    let photoPicker = new photoAccessHelper.PhotoViewPicker();

    const RESULT = await photoPicker.select(photoSelectOptions);
    if (RESULT && RESULT.photoUris && RESULT.photoUris.length > 0) {
      selectVideoUri = RESULT.photoUris[0];
      return selectVideoUri;
    } else {
      throw new Error(`${this.context.resourceManager.getStringSync($r('app.string.Selected_null_file').id)}`);
    }
  } catch (error) {
    let err: BusinessError = error as BusinessError;
    hilog.error(0x0000, TAG, `PhotoViewPicker failed with err: ${err.code}, ${err.message}`);
    throw new Error(`Failed to select the file: ${err.message}`);
  }
}
```

**`new photoAccessHelper.PhotoViewPicker()` with no context argument is the
permission-free form.** The picker runs in the system's own process and hands
back URIs the caller is authorised to read; the app never obtains a
`PhotoAccessHelper` and never sees anything the user did not choose. That is
the entire reason this sample's `module.json5` can be empty of
`requestPermissions` - and the reason `HW-13-0058` matters, because
`getAssets` later in `VideoEditPage` steps outside that contract.

Note the guard order: `RESULT && RESULT.photoUris && length > 0`. A cancelled
picker **resolves** with an empty array rather than rejecting, so the empty
case must be checked, not caught.

**Route hand-off — same file** (as shipped)

```typescript
CreativeComponent()
  .onClick(async () => {
    this.sourceVideoUri = await this.selectVideoFile();

    if (this.sourceVideoUri !== '') {
      let videoParams: VideoParams = {
        sourceVideoUri: this.sourceVideoUri,
      };

      this.pageInfo.pushPath({
        name: 'VideoEditPage',                 // resolved through route_map.json
        param: videoParams
      });
    }
  });
```

and on the receiving side, in `VideoEditPage`:

```typescript
.onReady(async (context: NavDestinationContext) => {
  let videoParams: VideoParams = context.pathInfo.param as VideoParams;
  this.sourceVideoUri = videoParams.sourceVideoUri as string;
  // ...
  this.sourceVideoName = this.getFileNameFromUri(this.sourceVideoUri);
  this.sourceVideoSandboxPath = this.context.filesDir + '/' + this.sourceVideoName;
  this.copyFile(this.sourceVideoUri, this.sourceVideoSandboxPath);
})
```

**The sandbox copy is not optional.** `sourceVideoUri` is a
`file://media/Photo/...` URI: `fs.openSync` can open it for reading because the
picker granted that, but ffmpeg is handed a *path string* on the native side
and needs something under the app's own directory. `filesDir` is the right
choice over `cacheDir` here only because the file is short-lived and deleted by
nothing - a production version should use `cacheDir` and clean up.

**Copying between a URI and a path — `entry/src/main/ets/pages/VideoEditPage.ets`** (as shipped)

```typescript
private copyFile(sourceUri: string, destinationPath: string) {
  let sourceFile: fs.File | null = null;
  let destFile: fs.File | null = null;

  try {
    sourceFile = fs.openSync(sourceUri, fs.OpenMode.READ_ONLY);
    destFile = fs.openSync(destinationPath, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
    fs.copyFileSync(sourceFile.fd, destFile.fd);
  } catch (err) {
    hilog.error(0x0000, 'CopyFile', `Failed to copy file：${err.message}`);
  } finally {
    if (sourceFile) {
      fs.closeSync(sourceFile);
    }
    if (destFile) {
      fs.closeSync(destFile);
    }
  }
}
```

**This is the shape to copy.** Both handles start as `null`, the close happens
in `finally`, and each close is guarded - so a failure to open the *destination*
still closes the source. The industry's `HW-13-0039` catalogues five samples
that get exactly this wrong by closing possibly-undefined handles
unconditionally, turning a cancelled picker into a second crash that masks the
original error.
The same function serves both directions of the pipeline: URI→sandbox on entry,
sandbox→URI on save, because `fs.openSync` accepts either.

`READ_WRITE | fs.OpenMode.CREATE` without `TRUNC` is the one weak spot: on a
pre-existing destination longer than the new content, stale bytes survive past
the end. Harmless here because every output name carries a timestamp.

**Extraction and its result code — same file** (corrected, see `HW-13-0057`)

```typescript
import { ICallBack, MP4Parser } from '@ohos/mp4parser';

callBack: ICallBack = {
  callBackResult: (code: number) => {
    if (code !== 0) {                            // FIX: the sample ignores `code` entirely
      hilog.error(0x0000, TAG, `ffmpeg failed with code ${code}`);
      // the sample ships no failure string - add one alongside Extract_successfully
      this.uiContext.getPromptAction().showToast({ message: `Extract failed (${code})`, duration: 2000 });
      return;
    }
    if (this.splitAudioOutputPath) {
      try {
        this.uiContext.getPromptAction().showToast({
          message: `${this.context.resourceManager.getStringSync($r('app.string.Extract_successfully').id)}`,
          duration: 2000
        });
        this.saveToFile(this.splitAudioName, this.splitAudioOutputPath);
      } catch (e) {
        hilog.error(0x0000, TAG, `Failed to extract audio. Error: `, e);
      }
    }
  }
};

private extractAudio(sourceVideoSandboxPath: string, splitAudioOutputPath: string) {
  try {
    MP4Parser.ffmpegCmd(util.format('ffmpeg -i %s -c:a copy -vn %s -y', sourceVideoSandboxPath,
      splitAudioOutputPath), this.callBack);
  } catch (e) {
    hilog.error(0x0000, TAG, `Failed to extract audio. Error: ${e}`);
  }
}
```

**`ffmpegCmd` is asynchronous and reports through the callback, not through a
throw.** The surrounding `try/catch` only catches a synchronous dispatch
failure; a codec error, an unreadable input or a container with no audio track
all arrive as a non-zero `code`. The shipped guard - `if (this.splitAudioOutputPath)` -
tests a field that the button handler assigned two lines before the call, so it
is true on every invocation after the first click. The result is that *every*
failure toasts 提取成功 and then opens a save dialog for an empty or missing
`.aac`.

`-c:a copy` is what makes this cheap enough to run on the UI thread's callback:
the audio elementary stream is remuxed, not decoded and re-encoded. Change it
to `-c:a aac -b:a 128k` and you are back to needing a worker.

**The thumbnail, and why it needs a permission — same file** (as shipped; see `HW-13-0058`)

```typescript
public async getFirstFrameAsThumbnail(context: common.UIAbilityContext, videoUrl: string): Promise<image.PixelMap> {
  let phAccessHelper = photoAccessHelper.getPhotoAccessHelper(context);
  let predicates: dataSharePredicates.DataSharePredicates = new dataSharePredicates.DataSharePredicates();
  predicates.equalTo('uri', videoUrl);
  let videoFetchResult: photoAccessHelper.FetchResult<photoAccessHelper.PhotoAsset> =
    await phAccessHelper.getAssets({                 // <-- requires ohos.permission.READ_IMAGEVIDEO
      fetchColumns: ['width', 'height', 'orientation'],
      predicates: predicates
    });
  let photoAsset: photoAccessHelper.PhotoAsset = await videoFetchResult.getFirstObject();

  let thumbnailSize: Size = { width: 0, height: 0 };
  if (photoAsset.get(photoAccessHelper.PhotoKeys.ORIENTATION) === 90 ||
    photoAsset.get(photoAccessHelper.PhotoKeys.ORIENTATION) === 270) {
    thumbnailSize.width = photoAsset.get(photoAccessHelper.PhotoKeys.HEIGHT) as number;
    thumbnailSize.height = photoAsset.get(photoAccessHelper.PhotoKeys.WIDTH) as number;
  } else {
    thumbnailSize.width = photoAsset.get(photoAccessHelper.PhotoKeys.WIDTH) as number;
    thumbnailSize.height = photoAsset.get(photoAccessHelper.PhotoKeys.HEIGHT) as number;
  }
  return photoAsset.getThumbnail(thumbnailSize);
}
```

**The orientation swap is the correct part and is worth keeping.** A phone
clip shot in portrait is usually stored landscape with `ORIENTATION` 90 or 270;
requesting a thumbnail at the stored width and height would give a sideways
box. Swapping the two for the quarter-turn cases produces a thumbnail whose
aspect matches what the user will actually see.

**The `getAssets` call is the incorrect part.** `getPhotoAccessHelper` +
`getAssets` is a *query against the media library*, gated on
`ohos.permission.READ_IMAGEVIDEO`, and this project's `module.json5` has no
`requestPermissions` array at all. The URI the picker handed over authorises
reading that one file; it does not authorise enumerating the library to find
it. The call throws, the caller's `try/catch` in `onReady` logs it, and
`videoFirstFrame` stays `null` - so the three timeline thumbnails and the
`Video`'s `previewUri` are silently empty. Nothing in the UI says so.
`HW-13-0058` records the same pattern in a second sample (`SplashPage`).

If you need frame-level metadata from a picked file, open its fd and use
`AVMetadataExtractor` - no permission, no library query, and it works for files
that are not in the gallery at all.

## Permissions & config

**None declared.** `module.json5` has no `requestPermissions` block:

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "pages": "$profile:main_pages",
    "routerMap": "$profile:route_map",
    "abilities": [ /* EntryAbility */ ],
    "extensionAbilities": [ /* EntryBackupAbility */ ]
  }
}
```

For the picker-only path that is correct and deliberate. It becomes a defect
the moment `getAssets` is called (`HW-13-0058`). If the thumbnail is kept,
`ohos.permission.READ_IMAGEVIDEO` must be **declared** with `reason` and
`usedScene` *and* **requested** at runtime with `requestPermissionsFromUser` -
declaring it alone is the sibling defect `HW-13-0002` on `MEDIA-03`.

`"routerMap": "$profile:route_map"` is required for the `VideoEditPage`
push to resolve, and `@ohos/mp4parser` must be in the module's
`oh-package.json5` dependencies (the sample has it only in the project root).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **`@ohos/mp4parser` is a third-party OpenHarmony package**, not an SDK kit.
  It bundles ffmpeg natively, so it adds meaningful size to the HAP and its
  licensing follows from ffmpeg's.
- `-c:a copy` writes the source's audio codec into an `.aac` container name
  regardless of what that codec is. A clip whose audio is not AAC produces a
  mislabelled file. Probe first, or transcode.
- Both `Tabs` in `MainPage` and the one in `VideoEditPage` return `false` from
  `onContentWillChange` for every index but one, so most tabs are decorative
  and cannot be entered - the bar highlight moves and the content does not.
- `copyFile` runs synchronously on the UI thread and is invoked twice per
  export for the source video. On a multi-hundred-megabyte clip that is a
  visible stall.
- Extraction results are written to `filesDir` and never deleted; every export
  leaves both a full copy of the source video and the `.aac` behind.
- `CreativeComponent` sets `.fontSize(400)` and then `.fontSize(14)` on the
  same `Text`; the second wins, but the first is a typo left in the shipped
  source.

## Pitfalls

- **`HW-13-0057`** (B/medium, confirmed): `callBackResult` ignores its `code`
  parameter and gates on `this.splitAudioOutputPath`, which is always truthy
  after the first export click. Every ffmpeg failure toasts 提取成功 and opens
  the save picker on a nonexistent or empty `.aac`. `MEDIA-25`'s `VideoUtils`
  checks `code === 0` - the correct pattern exists elsewhere in the same
  corpus. Fix: branch on `code === 0`.
- **`HW-13-0058`** (D/high, probable): `getAssets` is called with
  `ohos.permission.READ_IMAGEVIDEO` neither declared in `module.json5` nor
  requested at runtime, so the query throws, is only logged, and the preview
  thumbnail never appears. The same defect is present in `SplashPage`
  (`PhotoUtils.ets` + `MainPage.ets`), where it also triggers that sample's
  parallel-array desync. Distinct from `HW-13-0002`, which is
  declared-but-never-requested. Fix: declare and request the permission, or
  read the frame with `AVMetadataExtractor` on the picked fd.
- **Double sandbox copy** (observation, no HW id): `onReady` copies the source
  video into `filesDir`, and the export handler copies it again on every click.
- **`CREATE` without `TRUNC`** (observation, no HW id): `copyFile` opens the
  destination `READ_WRITE | CREATE`. Safe here because output names carry a
  timestamp; not safe if you reuse a fixed working filename - see
  `HW-13-0050` for the three samples where that bites.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` - `PhotoViewPicker`, `PhotoSelectOptions`, `PhotoViewMIMETypes`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `getPhotoAccessHelper`, `getAssets` and its permission
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoasset.md` - `PhotoKeys`, `get`, `getThumbnail`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoasset
- `documentation/harmonyos-references/02_application-framework/js-apis-file-picker.md` - `DocumentViewPicker.save`, `DocumentSaveOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-picker
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` - how a user_grant permission such as `READ_IMAGEVIDEO` must be requested
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `huawei_industry_tree/13_media_entertainment/docs/23_audio_extractor.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio_extractor-0000002298797484
- `MEDIA-25` - the same ffmpeg wrapper with the result code checked correctly
