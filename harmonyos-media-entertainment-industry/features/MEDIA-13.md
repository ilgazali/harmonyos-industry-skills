---
id: MEDIA-13
title: Import local music files - aggregated DocumentViewPicker, AVMetadataExtractor tags, and persisted URI grants
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/13_music_player.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/music_player-0000002296726005
sample: huawei_industry_tree/13_media_entertainment/downloads/MusicPlayer.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.AudioKit", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaKit", "@kit.PerformanceAnalysisKit"]
apis: [audio, fileIo, fileShare, hilog, image, media, picker, preferences, window]
permissions: [ohos.permission.FILE_ACCESS_PERSIST]
min_api: 20
modules: [common (har), entry (entry)]
findings: [HW-13-0003, HW-13-0036, HW-13-0037, HW-13-0038, HW-13-0039, HW-13-0098, HW-13-0099]
status: verified-with-fixes
---

## When to use

Load this card when the app must play **files the user owns but the app does
not**: a local music library, a podcast folder, an audio note pile. The
pattern is a three-stage pipeline - pick with `DocumentViewPicker` in
aggregated audio mode, read the ID3-style tags with `AVMetadataExtractor`,
hand the file descriptor to an `AVPlayer` - plus a fourth stage nobody
remembers until the second launch: **persist the URI grant** so the list
survives a restart.

That fourth stage is the reason to read this card rather than the AVPlayer
reference. A picker grant is a one-shot, per-URI capability. Without
`fileShare.persistPermission` at import time and `fileShare.activatePermission`
before the first read of the next launch, every song in the restored list
opens with a permission error and the library looks empty. The same shape
applies to any app that keeps a user-curated list of external files: an
e-reader's shelf, a video app's local folder, an editor's recent-documents
list.

`ohos.permission.FILE_ACCESS_PERSIST` is a **restricted** permission. Debug
builds get it from DevEco's automatic signing; a release build needs an ACL
application. Decide that before committing to the pattern.

## Feature checklist

- A 导入 (import) action in the header opens the system file picker filtered
  to audio, with multi-select and batch authorisation.
- Picked files join the existing playlist without duplicating anything already
  in it.
- Each row shows the title, artist and album cover taken from the file's own
  metadata, and falls back to the filename plus a default cover when a tag is
  missing.
- Each row shows the duration as `mm:ss`, derived from the metadata.
- Tapping a row starts playback; the bottom bar appears with the current song
  and a play/pause toggle.
- The play/pause icon follows the player, not the tap - it is driven by the
  `stateChange` callback.
- Closing and relaunching the app restores the whole imported list, covers
  included, without asking for the files again.
- An empty library shows an illustration that is itself an import button.

## Architecture

Two modules. The playback and metadata code lives in a `common` HAR so a
second UI module (a widget, a car head unit) could reuse it; the entry module
owns the list, the picker call and the persistence.

```
common/
├── Index.ets                                   the HAR surface: MusicService, MusicMetadata, METADATA_SERVICE
└── src/main/ets/util
    ├── MusicMetadataService.ets                AVMetadataExtractor wrapper -> { metadata, pixelMap }
    └── MusicService.ets                        one AVPlayer, its state machine, playNewSong / switchSongPlayStatus
entry/src/main/ets
├── common/Logger.ets                           hilog wrapper
├── common/PersistPermissionUtils.ets           persist / activate grants + the preferences store of URIs
├── entryability/EntryAbility.ets               full screen, avoid areas -> AppStorage
├── pages/MainPage.ets                          @Entry: header, list, bottom player bar
└── util/ImportSongs.ets                        picker -> dedup -> persist -> metadata -> Map<uri, SongInfo>
```

The documented tree names the ability file `Entryability.ets`; the zip has
`EntryAbility.ets`, and `build-profile.json5` sets
`strictMode.caseSensitiveCheck: true`, so the documented spelling would not
resolve (`HW-13-0003`).

**The design decision worth copying** is that the playlist is a
`Map<string, SongInfo>` keyed by URI rather than an array. Import is
inherently a merge - the user re-picks a folder, half of it is already there -
and a URI-keyed map makes "already imported" a `has()` instead of a scan, so
the dedup, the metadata skip and the render all agree on one identity. The
cost is that the map must be rebuilt (`this.playList = songMap`) to trigger a
re-render, and that assignment is exactly where `HW-13-0038` bites: a failed
import returns an empty map and the assignment wipes the list.

**The decision worth avoiding** sits next to it: `MusicService` is
constructed in `aboutToAppear` and never released. There is no
`aboutToDisappear`, no `avPlayer.release()` anywhere in the project. Add one
before you ship - the AVPlayer holds a native decoder and an audio focus
token.

## Implementation steps

1. **Open the picker in aggregated audio mode.** `mergeMode:
   picker.MergeTypeMode.AUDIO` collapses the file tree into a flat audio view;
   `multiAuthMode: true` makes one confirmation cover the whole selection
   instead of one dialog per file.
2. **Concatenate the picked URIs with the already-persisted ones and
   deduplicate** through a `Set`, then **iterate the deduplicated array** -
   the sample bounds the loop by the deduplicated length but indexes the raw
   one (`HW-13-0037`).
3. **Persist before reading.** Call `fileShare.persistPermission` on the whole
   URI list and store the list in `preferences` in the same pass, so a crash
   between the two cannot leave a persisted grant with no record of it.
4. **Guard on `canIUse('SystemCapability.FileManagement.AppFileService.FolderAuthorization')`**
   before every `fileShare` call; the capability is absent on some device
   types and the API throws rather than no-ops.
5. **Read metadata per file with a fresh `AVMetadataExtractor`.** Set
   `fdSrc` from `fs.openSync(uri)`, `await fetchMetadata()`, then
   `fetchAlbumCover()` for the `PixelMap`.
6. **Await `release()` before closing the fd, and guard the close** - the
   shipped `finally` calls `fs.closeSync(avMetadataExtractor?.fdSrc?.fd)` on a
   handle that is `undefined` whenever `createAVMetadataExtractor` or
   `openSync` threw, so the cleanup itself throws and masks the real error
   (`HW-13-0039`).
7. **Seed the merge map from the existing playlist and return that map on
   failure**, never an empty one (`HW-13-0038`).
8. **Keep the fd open for the whole playback.** Open the file, assign
   `avPlayer.url = 'fd://' + file.fd`, and close only after the player is
   released or moves to the next track (`HW-13-0036`).
9. **Drive playback from `stateChange` alone.** `initialized` sets
   `audioRendererInfo` and calls `prepare()`; `prepared` calls `play()`. Do
   not call `play()` from the loader as well.
10. **On launch, restore before you read**: pull the URI list out of
    `preferences`, `await fileShare.activatePermission(policies)`, and only
    then run the metadata pass.

## Verified snippets

All snippets are from `MusicPlayer.zip`. Corrected forms are marked.

**Picking and merging — `entry/src/main/ets/util/ImportSongs.ets`** (corrected, see `HW-13-0037`, `HW-13-0038`)

```typescript
export async function importSongs(playList: Map<string, SongInfo>, context: Context): Promise<Map<string, SongInfo>> {
  let newMap: Map<string, SongInfo> = new Map();
  playList.forEach((song: SongInfo, key: string) => {
    newMap.set(key, song);                       // FIX: seed from the current list, not empty
  });
  try {
    let documentPicker = new picker.DocumentViewPicker(context);
    let selectResult: Array<string> = await documentPicker.select({
      mergeMode: picker.MergeTypeMode.AUDIO,     // aggregated audio view, not a folder tree
      multiAuthMode: true                        // one grant covers the whole multi-selection
    });
    let existPath: Array<string> = await getPersistPermissionUtils().getFilePathExist(context);
    let allPath = selectResult.concat(existPath);
    let filePath: string[] = Array.from(new Set(allPath));
    getPersistPermissionUtils().persistPermissionExample(filePath);
    getPersistPermissionUtils().putFilePath(context, filePath);

    for (let i = 0; i < filePath.length; i++) {
      let uri = filePath[i];                     // FIX: sample reads allPath[i] under filePath.length
      let index = uri.lastIndexOf('/') + 1;
      let fileName = decodeURIComponent(uri.slice(index));
      let info: SongInfo = { uri, duration: 0, songName: fileName };
      if (!newMap.has(info.uri)) {
        await addToList(uri, info, newMap);
      }
    }
  } catch (error) {
    Logger.error(TAG, 'DocumentViewPicker failed with err: ' + JSON.stringify(error));
  }
  return newMap;                                 // FIX: on failure this is now the unchanged list
}
```

**Two of the picker options carry the whole import UX.** `MergeTypeMode.AUDIO`
is what turns a document browser into a music browser - the user sees audio
files aggregated across directories rather than having to navigate to them.
`multiAuthMode: true` is what makes a multi-select worth offering: without it
each selected URI needs its own authorisation round trip, so a fifty-song
import becomes fifty prompts.

The two index bugs in this function are the same bug seen twice. `allPath` is
the concatenation (with duplicates), `filePath` the deduplicated set, and the
loop mixes them: with any overlap between the new selection and the existing
library, `filePath.length` is shorter than `allPath`, so the tail of `allPath`
- the genuinely new URIs, which sort last after `concat` - is never visited
(`HW-13-0037`). And because `newMap` is only populated inside the `try`, any
throw before `newMap = tempMap` returns an empty map that `MainPage` assigns
straight over `this.playList` (`HW-13-0038`). The user cancels the picker;
the library disappears.

**Metadata and cover — `common/src/main/ets/util/MusicMetadataService.ets`** (corrected, see `HW-13-0039`)

```typescript
async fetchMetadata(uri: string): Promise<MusicMetadata> {
  let avMetadataExtractor: media.AVMetadataExtractor | null = null;
  let pixelMap: PixelMap | undefined = undefined;
  let metadata: media.AVMetadata | undefined = undefined;
  if (!canIUse('SystemCapability.Multimedia.Media.AVMetadataExtractor')) {
    throw new Error('System does not support AVMetadataExtractor');
  }
  try {
    avMetadataExtractor = await media.createAVMetadataExtractor();
    avMetadataExtractor.fdSrc = fs.openSync(uri);
    metadata = await avMetadataExtractor.fetchMetadata();
    hilog.info(DOMAIN, 'this avPlayer', '%{public}s', `Get metadata successfully, mimeType: ${metadata.mimeType}`);
    await avMetadataExtractor
      .fetchAlbumCover()
      .then((cover: PixelMap) => {
        pixelMap = cover;
      })
      .catch(() => {
        hilog.error(DOMAIN, 'this avPlayer', '%{public}s', `Get Album Cover Failed`);   // no cover is normal
      });
  } catch (e) {
    hilog.error(DOMAIN, 'this avPlayer', '%{public}s', `createAVMetadataExtractor Failed, error:${e}`);
  } finally {
    const file = avMetadataExtractor?.fdSrc;
    await avMetadataExtractor?.release();        // FIX: sample drops this promise
    if (file) {                                  // FIX: sample calls closeSync(undefined) when open threw
      fs.closeSync(file);
    }
  }
  return { pixelMap: pixelMap, metadata: metadata };
}
```

**A missing album cover is not an error, a missing file is.** That asymmetry
is why `fetchAlbumCover` has its own `.catch` inside the `try`: most audio
files have no embedded artwork, and letting that reject would throw away the
title and artist that were read successfully one line earlier. The outer
`catch` is for the real failures - unsupported container, revoked URI.

The `finally` in the shipped code is the local instance of a defect that
recurs across five media samples (`HW-13-0039`): it calls
`fs.closeSync(avMetadataExtractor?.fdSrc?.fd)` unconditionally, and if
`createAVMetadataExtractor` or `fs.openSync` threw, `fdSrc` is `undefined`,
`?.fd` is `undefined`, and `closeSync(undefined)` raises a `TypeError` **from
inside the finally** - replacing the diagnosed error with a cleanup error. The
second half of the fix is the `await` on `release()`: closing a descriptor the
extractor may still be reading from is a race even when both objects exist.
Note also that `closeSync` accepts the `File` object; passing the raw `.fd`
number works but discards the handle the API expects.

**Handing the fd to the player — `common/src/main/ets/util/MusicService.ets`** (corrected, see `HW-13-0036`)

```typescript
async playNewSong(uri: string): Promise<void> {
  if (this.avPlayer) {
    switch (this.avPlayer.state) {
      case 'playing':
      case 'paused':
      case 'prepared':
      case 'completed':
        await this.avPlayer.stop();              // fallthrough is deliberate: stopped -> reset -> idle
      case 'stopped':
        await this.avPlayer.reset();
      case 'idle':
        if (this.currentFile) {
          fs.closeSync(this.currentFile);        // FIX: close the PREVIOUS track's fd here
          this.currentFile = undefined;
        }
        this.currentFile = await fs.open(uri);
        this.avPlayer.url = 'fd://' + this.currentFile.fd;
        // FIX: no play() here - the stateChange handler drives initialized -> prepare -> prepared -> play
        return;
      default:
        hilog.error(DOMAIN, 'this avPlayer', '%{public}s', 'AVPlayer is not in correct status.');
    }
  }
}
```

**Assigning `url` is not a read, it is a subscription.** The AVPlayer keeps
the descriptor and reads from it for the entire life of the playback, so the
fd must outlive the assignment - which is exactly what the shipped code gets
wrong: it opens the file, builds `'fd://' + file.fd` inside a `try`, and then
calls `fs.close(file)` in the `finally`, **before** `this.avPlayer.url =
fdPath` on the next line (`HW-13-0036`). The player receives a string naming a
descriptor the process has already released. `fs.close` without `await` makes
it a race rather than a certainty, which is worse - it will work on a fast
device and fail on a slow one.

The premature `play()` is the second half of the same finding. At the moment
`url` is assigned the player is still `idle`; it becomes `initialized`
asynchronously, and only the `stateChange` handler knows when. That handler
already calls `prepare()` on `initialized` and `play()` on `prepared`, so the
extra `play()` is guaranteed to be an invalid-state call - and the `error`
callback responds to it with `avPlayer.reset()`, tearing down the load that
was in flight.

The switch fallthrough (no `break` after `stop()` or `reset()`) is
intentional and worth keeping: whatever state the player is in, it walks down
to `idle` and loads from there.

**Persisting and reactivating the grants — `entry/src/main/ets/common/PersistPermissionUtils.ets`** (as shipped)

```typescript
public getFilePolicyInfo(pathArray: Array<string>) {
  let policies: Array<fileShare.PolicyInfo> = [];
  for (let index = 0; index < pathArray.length; index++) {
    policies.push({ uri: pathArray[index], operationMode: fileShare.OperationMode.READ_MODE });
  }
  return policies;
}

public async persistPermissionExample(pathArray: Array<string>) {
  let policies: Array<fileShare.PolicyInfo> = this.getFilePolicyInfo(pathArray);
  if (!this.checkSystemCapability()) {
    Logger.error(TAG, 'persistPermission can not use');
    return;
  }
  try {
    await fileShare.persistPermission(policies);
  } catch (err) {
    Logger.error(TAG, 'persistPermission failed with error message: ' + err.message + ', error code: ' + err.code);
    this.dataPreferences?.clearSync();           // the stored URI list is worthless without the grants
  }
}

public async activatePermissionExample(pathArray: Array<string>) {
  try {
    if (pathArray.length !== 0 && this.checkSystemCapability()) {
      let policies: Array<fileShare.PolicyInfo> = this.getFilePolicyInfo(pathArray);
      await fileShare.activatePermission(policies);
    }
  } catch (error) {
    let err: BusinessError = error as BusinessError;
    Logger.error(TAG, `activatePermissionExample failed with err, Error code: ${err.code}, message: ${err.message}`);
  }
}
```

**Persist and activate are two different operations and both are required.**
`persistPermission` turns the picker's transient grant into a durable one,
once, at import time. `activatePermission` re-arms it in the current process,
every launch, before the first read - a persisted grant that has not been
activated still fails to open. `MainPage.initPlayListFromPf` gets this order
right: read the URI list from `preferences`, `await activatePermission`, then
extract metadata.

`operationMode: READ_MODE` is the correct narrowing here - the app plays
files, it never writes them. Requesting `READ_WRITE_MODE` would widen the
persisted capability for no gain.

The `clearSync()` in the catch is the honest response to a persistence
failure: the preferences list is only a record of URIs the app believes it can
still open, so if the grants did not persist, keeping the list would produce a
library of unopenable rows on the next launch.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.FILE_ACCESS_PERSIST",
    "reason": "$string:permission_reason_file_access_persist",
    "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
  }
]
```

- `FILE_ACCESS_PERSIST` is a **restricted** permission, not an ordinary
  `user_grant` one. In debug, DevEco's automatic signing applies for it on the
  developer's behalf; for release the app needs a manual ACL application
  against the release profile. Plan for the lead time.
- `when: "always"` is correct here even though the app has no background task:
  the persisted grant is activated during startup, before the ability is
  necessarily in the foreground.
- The `common` HAR declares no permissions of its own - `type: "har"` modules
  inherit the host's.
- `build-profile.json5` sets `caseSensitiveCheck: true` and
  `useNormalizedOHMUrl: true`, which is what makes the documented
  `Entryability.ets` spelling a real build failure rather than a typo
  (`HW-13-0003`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- `AVMetadataExtractor` is behind
  `SystemCapability.Multimedia.Media.AVMetadataExtractor` and
  `fileShare.persistPermission` behind
  `SystemCapability.FileManagement.AppFileService.FolderAuthorization`. The
  sample guards both with `canIUse`; keep the guards.
- There is no seek bar, no progress display and no next/previous track. The
  bottom bar is a play/pause toggle over the current song only.
- The player is never released: `MainPage` has no `aboutToDisappear` and the
  project contains no `avPlayer.release()` call. Add one.
- `ForEach` over the playlist keys on `JSON.stringify(info)`, which includes
  the `PixelMap` - stringifying a decoded cover on every diff pass. Key on
  `info.uri` instead.
- Metadata extraction is serial (`await addToList` inside the loop), so a
  hundred-song import is a hundred sequential decodes with no progress
  indicator.

## Pitfalls

- **`HW-13-0036`** (B/high, confirmed) — `playNewSong` closes the fd in its
  `finally` **before** assigning `avPlayer.url = 'fd://N'`, so the player is
  handed a released descriptor; the unawaited `fs.close` makes it a race. The
  same block also calls `play()` while the player is still idle, and the
  `error` handler answers that invalid-state call with `reset()`, killing the
  load. Fix: close the previous track's fd on the next load or at release, and
  let `stateChange` drive `prepare()`/`play()`.
- **`HW-13-0037`** (B/medium, confirmed) — the import loop is bounded by
  `filePath.length` (deduplicated) but indexes `allPath[i]` (raw). Any overlap
  between the new selection and the existing library truncates the walk, so
  genuinely new files at the tail are silently never imported. Fix: index
  `filePath[i]`.
- **`HW-13-0038`** (B/low, confirmed) — `newMap` is only assigned at the end of
  the `try`, so any throw (including a cancelled picker) returns an empty map,
  and `MainPage` assigns it over `this.playList`. One failed import clears the
  visible library until restart. Fix: seed `newMap` from the existing entries
  before the `try`.
- **`HW-13-0039`** (B/medium, confirmed) — systematic across five media
  samples: `finally` blocks call `closeSync` on possibly-undefined handles.
  Here `fs.closeSync(avMetadataExtractor?.fdSrc?.fd)` throws a `TypeError`
  whenever the open or the extractor creation failed, masking the original
  error, and `release()` is not awaited before the close. Fix: capture the
  handle, `await release()`, and guard the close.
- **`HW-13-0003`** (E/low, confirmed) — systematic tree defect: the document's
  工程目录 names `Entryability.ets` where the zip has `EntryAbility.ets`. With
  `caseSensitiveCheck: true` the documented name does not resolve. The same
  class of error appears in `32_video_screenshot` (`Logger.ets` vs
  `logger.ets`) and `30_music_demo_httpaudiorender` (`index.d.ets` vs
  `Index.d.ts`). Fix: regenerate the trees from the zips.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-file-picker.md` - `DocumentViewPicker.select`, `MergeTypeMode`, `multiAuthMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-picker
- `documentation/harmonyos-references/04_media/arkts-apis-media-avmetadataextractor.md` - `fdSrc`, `fetchMetadata`, `fetchAlbumCover`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avmetadataextractor
- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - the state machine, `url`, `audioRendererInfo`, `stateChange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-guides/03_application-framework/file-persistpermission.md` - `persistPermission` / `activatePermission` and the launch-time reactivation
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/file-persistpermission
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - `ohos.permission.FILE_ACCESS_PERSIST`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
- `documentation/harmonyos-guides/04_system/declare-permissions-in-acl.md` - the release-build ACL application
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions-in-acl
- `documentation/harmonyos-guides/08_coding-and-debugging/ide-signing.md` - automatic signing for the debug phase
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ide-signing
- `MEDIA-11` - the AVPlayer audio player this service is a cut-down version of
- `MEDIA-14` - the same `finally`-closeSync defect (`HW-13-0039`) in the GIF sample
