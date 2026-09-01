---
id: UTIL-31
title: Batch media download - taskpool LongTask groups of five, then one gallery save dialog
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/31_batch_download_images_and_videos.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/batch_download_images_and_videos-0000002464972285
sample: huawei_industry_tree/15_utilities/downloads/BatchMediaDownload.zip
kits: ["@kit.ArkTS", "@kit.NetworkKit", "@kit.MediaLibraryKit", "@kit.CoreFileKit", "@kit.ArkData", "@kit.AbilityKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit"]
apis: ["taskpool.LongTask", "taskpool.execute", "taskpool.cancel", "taskpool.terminateTask", "@Concurrent", "http.createHttp", "HttpRequest.request", HttpDataType, "photoAccessHelper.getPhotoAccessHelper", showAssetsCreationDialog, PhotoCreationConfig, "fileUri.getUriFromPath", "fs.openSync", "fs.writeSync", "uniformTypeDescriptor.getUniformDataTypeByFilenameExtension", belongsTo, "@Watch", "@StorageProp"]
permissions: ["ohos.permission.INTERNET"]
min_api: 20
modules: [entry]
findings: [HW-15-0069, HW-15-0070, HW-15-0101, HW-15-0102]
status: verified-with-fixes
---

## When to use

**Load this card when the user hands you a list of remote media and expects
all of it in the gallery.** A "save all" in a chat thread, a shared album
import, a product-photo pack, a batch export - anything where the count is
tens rather than one and the destination is the system album rather than app
storage.

Two mechanisms do the work and both generalise well beyond media. The first is
**taskpool with grouped `LongTask`s**: URLs are chunked five at a time, each
chunk becomes one concurrent task, and the group returns its results to the
main thread in one message. That is the right granularity for any batch of
network work - per-item tasks flood the pool with scheduling overhead, one big
task loses all parallelism. The second is **`showAssetsCreationDialog`**: the
app writes files into its own sandbox with no permission at all, then asks the
user once, and gets back write URIs for exactly those files. No
`WRITE_IMAGEVIDEO`, no permission dialog on launch, one confirmation for the
whole batch.

**Read `HW-15-0069` before adopting the task protocol.** Completion is tracked
by a boolean array with a `@Watch`, and no code path sets an entry to `true`
when a group fails - so one rejected task leaves the spinner turning until the
app is restarted.

## Feature checklist

- A full-height `TextArea` where the user pastes URLs separated by semicolons;
  newlines and spaces are stripped.
- Empty entries are dropped, and the list is capped at 100 because the save
  dialog will not accept more in one call.
- One capsule button, 下载并保存 (download and save), which turns into
  下载中 (downloading) with a `LoadingProgress` spinner while work is in
  flight and ignores further taps.
- URLs are downloaded in groups of five, concurrently, off the UI thread.
- Each response is classified as image or video from its `content-type`, or
  from the URL extension when the header is missing; anything else is skipped.
- Each file is written into the app sandbox first.
- When every group has reported, one system dialog asks the user to confirm
  saving the whole batch into the gallery.
- Confirming writes the bytes into the granted URIs and clears the batch;
  pressing the button again with unchanged text re-opens the save dialog
  instead of re-downloading.

## Architecture

One `entry` module. The split that matters is the thread boundary: everything
under `taskpool/` runs in a worker, everything under `pages/` on the UI thread.

```
entry/src/main/ets
├── common
│   ├── CommonConstant.ets    group size 5, 10 MB cap, 60 s timeouts, 100-file limit
│   └── CommonEnum.ets        MediaType.IMAGE / VIDEO
├── entryability/EntryAbility.ets
├── model/DataModel.ets       FileModel: name, extension, dir, type, ArrayBuffer
├── pages/MainPage.ets        @Entry: input, grouping, completion watch, album save
├── taskpool
│   ├── DownloadTask.ets      the LongTask wrapper + the @Concurrent worker body
│   ├── ProcessFunction.ets   content-type -> MediaType, and the sandbox write
│   └── TaskExecutor.ets      singleton: taskpool.execute + callback dispatch
└── utils/Logger.ets
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that the worker returns *data*, not
side effects. `downloadFiles` writes each payload into the sandbox and then
returns an array of `FileModel` - filename, extension, path, media type and the
raw `ArrayBuffer` - and the main thread does the gallery save for all groups at
once. The alternative, saving to the album inside the worker, is not available
anyway: `showAssetsCreationDialog` is UI and needs a `Context`. Splitting it
this way also means the confirmation is one dialog for the whole batch instead
of one per group, which is the difference between a usable feature and an
unusable one.

**The design decision worth avoiding** is the completion protocol:

```typescript
@State @Watch('taskStateWatch') taskState: boolean[] = [];

taskStateWatch() {
  let taskResult = this.taskState.every((value) => value) && this.isInit;
  if (taskResult) {
    this.isInit = false;
    this.isDownloading = false;
    this.saveMediaFile();
  }
}
```

An array of booleans, one per group, and a watcher that fires the save when
they are all true. The idea is sound and it is cheap; the flaw is that only the
success callback writes `true`. There is no failure leg, and `taskpool.execute`
is called without a `.catch`, so a rejected group's slot stays `false` for the
lifetime of the page (`HW-15-0069`). `Promise.allSettled` over the group
promises would express the same thing without a state array and without the
hole.

Note also the chunking loop:

```typescript
for (let i = 0; i < urls.length; ) {
  result.push(urls.splice(i, i + CommonConstant.ONE_GROUP_COUNT));
}
```

It works, but only because `i` never increments: `splice(0, 5)` removes the
first five each pass and the array shrinks until the condition fails. The
second argument is a *delete count*, so `i + 5` is arithmetic that happens to
equal 5 forever. Add an `i++` anywhere and the grouping breaks.

## Implementation steps

1. **Normalise the input before splitting.** Strip newlines and spaces, split
   on `;`, filter out empties - a pasted list always carries stray whitespace.
2. **Cap the list at 100.** `showAssetsCreationDialog` accepts a bounded batch;
   the sample's `MAX_SAVE_LIMIT` is the number it was written against.
3. **Chunk into groups of five and create one `taskpool.LongTask` per group.**
   `LongTask` rather than `Task` because a group of five downloads can exceed
   the ordinary task timeout.
4. **Attach a `.catch` to `taskpool.execute` that marks the group done**
   (`HW-15-0069`). A failed group must complete the protocol, not drop out of
   it.
5. **Call `terminateTask` on every leg, success and failure** (`HW-15-0069`).
   The reference is explicit: a continuous task's thread lives until
   `terminateTask` is called.
6. **Cancel outstanding tasks in `aboutToDisappear`** (`HW-15-0069`).
   `cancelTask` exists in `DownloadTask` and is never called from anywhere.
7. **Classify by `content-type` first, URL extension second.** Only accept
   `image/*` and `video/*`; return `undefined` for anything else so it is
   skipped rather than saved as a broken asset.
8. **Build the sandbox filename with a dot and a unique component**
   (`HW-15-0070`). Timestamp alone collides across concurrent groups.
9. **Handle the dialog's rejection and release the helper** (`HW-15-0069`).
   `showAssetsCreationDialog` returns a promise that rejects if the user
   dismisses it; without a `.catch` the `PhotoAccessHelper` is never released.
10. **Write the bytes into the returned URIs, in order.** The array of granted
    URIs is index-aligned with the configs you passed in.

## Verified snippets

All snippets are from `BatchMediaDownload.zip`. Corrected forms are marked.

**Grouping and dispatch — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
mediaResourceDownload() {
  if (this.preContent === this.content) {
    this.saveToAlbum(this.getUIContext().getHostContext() as Context);
    return;                                    // same text: re-open the dialog, do not refetch
  }
  this.fileModels = [];
  let urls = this.preprocessUri();
  if (!urls || urls.length <= 0) {
    return;
  }
  this.preContent = this.content;
  this.isDownloading = true;
  let result: string[][] = [];
  for (let i = 0; i < urls.length; ) {
    result.push(urls.splice(i, i + CommonConstant.ONE_GROUP_COUNT));
  }
  this.taskState = new Array(result.length);
  this.isInit = true;
  this.taskState.fill(false);
  result.forEach((groupValue: string[], index: number) => {
    let downloadTask = new DownloadTask('DownloadTask', (resultModels: FileModel[]) => {
      this.fileModels.push(...resultModels);
      this.taskState[index] = true;
      downloadTask.terminateTask();
    }, groupValue, this.filesDir);
    TaskExecutor.getInstance().execute(downloadTask);
  });
}
```

**`preContent` is the whole retry story.** The first tap downloads; a second
tap with identical text skips straight to `saveToAlbum`, because the files are
already in the sandbox and only the user's confirmation is missing. It is reset
to `'none'` after a successful save, so editing the box or completing a save
both re-arm the download path. Cheap, and it means a dismissed dialog costs
nothing to retry.

`this.taskState[index] = true` inside the per-group callback is the only writer
of that array, and `@Watch('taskStateWatch')` on it is what eventually triggers
the album save. `isInit` guards against the watcher firing on the `fill(false)`
of a zero-length array.

`filesDir` is read once in a field initialiser
(`(this.getUIContext().getHostContext() as Context).filesDir`) and passed into
the worker as a plain string - a `Context` cannot cross the taskpool boundary,
a path can.

**Execution and cleanup — `entry/src/main/ets/taskpool/TaskExecutor.ets`** (corrected, see `HW-15-0069`)

```typescript
public execute(task?: DownloadTask) {
  if (!task || !task.task) {
    Logger.error(TAG, 'executeTask task is null.');
    return;
  }
  taskpool.execute(task.task, task.priority).then((result: Object) => {
    try {
      if (task.callback) {
        task.callback(result);
      }
    } catch (error) {
      Logger.error(TAG, 'executeTask callback error: ' + error.toString());
    }
  }).catch((error: BusinessError) => {          // FIX: absent in the sample
    Logger.error(TAG, `executeTask rejected: ${error.code} ${error.message}`);
    if (task.callback) {
      task.callback([]);                        // complete the protocol with an empty group
    }
    task.terminateTask();                       // FIX: otherwise the LongTask thread leaks
  });
}
```

```typescript
// entry/src/main/ets/pages/MainPage.ets — FIX: no exit cleanup in the sample
aboutToDisappear(): void {
  this.downloadTasks.forEach((task: DownloadTask) => task.cancelTask());
  this.downloadTasks = [];
}
```

**The failure leg has to satisfy the same contract as the success leg.** The
callback is not "here are your files", it is "this group is finished" - it
happens to carry files. Calling it with `[]` marks `taskState[index]` true, the
watcher sees every slot true, and the batch proceeds with whatever downloaded.
Without it the spinner never stops and the button is permanently inert, because
`onClick` returns early while `isDownloading` is set.

`terminateTask` is not optional housekeeping. The taskpool reference states
that the thread executing a continuous task "exists until `terminateTask` is
called after the execution is complete" - so a rejected group holds a worker
thread for the life of the process. `DownloadTask.cancelTask` already exists in
the sample, fully written, and is called from nowhere; wiring it into
`aboutToDisappear` costs two lines and stops downloads continuing after the
user has left the page.

**The worker and the sandbox write — `entry/src/main/ets/taskpool/DownloadTask.ets` and `ProcessFunction.ets`** (corrected, see `HW-15-0070`)

```typescript
@Concurrent
async function downloadFiles(urls: string[], storeDir: string) {
  let results: FileModel[] = [];
  for (let url of urls) {
    try {
      const RESPONSE = await http.createHttp()
        .request(url, {
          method: http.RequestMethod.GET,
          maxLimit: CommonConstant.MAX_DOWNLOAD_LIMIT,       // 10 MB
          expectDataType: http.HttpDataType.ARRAY_BUFFER,
          connectTimeout: CommonConstant.CONNECT_TIMEOUT,
          readTimeout: CommonConstant.READ_TIMEOUT
        });
      if (RESPONSE.responseCode === http.ResponseCode.OK) {
        let respData = getRespData(url, RESPONSE);
        if (respData) {
          saveDownloadFile(respData, storeDir);
          results.push(respData);
        }
      }
    } catch (e) {
      Logger.error('DownloadTask', `request download file failed, url: ${url}, ${e.code} ${e.message}`);
    }
  }
  // 此处将数据返回至主线程，由主线程统一保存至相册
  return results;
}
```

```typescript
export function saveDownloadFile(respData: FileModel, storeDir: string) {
  let filePrefix = respData.type === MediaType.IMAGE ? CommonConstant.IMAGE_NAME : CommonConstant.VIDEO_NAME;
  // FIX: shipped name is `filePrefix + '-' + new Date().getTime()` and the path
  // concatenates the extension with no separator: 'image-1750000000000jpeg'.
  // A per-worker counter is not enough - groups run on different threads.
  let unique = `${new Date().getTime()}-${Math.floor(Math.random() * 1000000)}`;
  let fileName = `${filePrefix}-${unique}`;
  let fileDir = `${storeDir}/${fileName}.${respData.fileExtension}`;
  let file: fo.File | undefined = undefined;
  try {
    file = fo.openSync(fileDir, fo.OpenMode.CREATE | fo.OpenMode.WRITE_ONLY);
    fo.writeSync(file.fd, respData.data);
    respData.fileName = fileName;
    respData.fileDir = fileDir;
  } catch (e) {
    throw new Error(`save file to sandbox error: ${e.code} ${e.message}`);
  } finally {
    if (file) {
      fo.closeSync(file);
    }
  }
}
```

**Three options on the request carry the design.** `expectDataType:
ARRAY_BUFFER` is what makes the response usable as file bytes rather than a
string - get this wrong and binary data is mangled by text decoding.
`maxLimit` raises the response ceiling to 10 MB; the default is 5 MB and a
video will exceed it. `connectTimeout` / `readTimeout` at 60 s are generous
because this runs on a worker where a stall costs no frames.

The `try` is *inside* the loop, deliberately: one bad URL in a group of five
must not abort the other four. Note the flip side - a group where all five fail
still resolves successfully with `[]`, which is why the rejection path in
`TaskExecutor` is about task-level failure, not HTTP failure.

The filename correction is one character and a disambiguator. Without the dot the
sandbox file has no extension at all, which feeds an extension-less path into
`fileUri.getUriFromPath` and then into the save dialog's validation. Without
the suffix, up to twenty concurrent groups can call `new Date().getTime()` in
the same millisecond and produce the same path - `OpenMode.CREATE |
WRITE_ONLY` then truncates and one download silently overwrites another.

**The album save — `entry/src/main/ets/pages/MainPage.ets`** (corrected, see `HW-15-0069`)

```typescript
saveToAlbum(context: Context) {
  if (!this.saveUris || !this.configs || this.saveUris.length <= 0 || this.configs.length <= 0) {
    Logger.error('MainPage', 'can not save to album, please check parameter');
    return;
  }
  try {
    let accessHelper = photoAccessHelper.getPhotoAccessHelper(context);
    accessHelper.showAssetsCreationDialog(this.saveUris, this.configs)
      .then((permUris) => {
        if (permUris && permUris.length > 0) {
          permUris.forEach((permUri, index) => {
            let file: fo.File | undefined = undefined;
            try {
              file = fo.openSync(permUri, fo.OpenMode.WRITE_ONLY);
              fo.writeSync(file.fd, this.fileModels[index].data);
            } catch (e) {
              Logger.error('MainPage', `save file to media failed`);
            } finally {
              if (file) { fo.closeSync(file); }
            }
          });
          this.preContent = 'none';
          this.saveUris = undefined;
          this.configs = undefined;
          this.fileModels = [];
        }
        accessHelper.release();
      })
      .catch((err: BusinessError) => {          // FIX: absent in the sample
        Logger.error('MainPage', `showAssetsCreationDialog failed: ${err.code} ${err.message}`);
        accessHelper.release();                 // FIX: the helper leaks on rejection
      });
  } catch (e) {
    Logger.error('MainPage', `save media resource to album failed: ${e.code} ${e.message}`);
  }
}
```

**This is the permission-free path into the gallery.** The app never asks for
`ohos.permission.WRITE_IMAGEVIDEO`; it presents the files it already holds in
its sandbox, and the system returns write URIs for the ones the user approved.
`PhotoCreationConfig` is the metadata that dialog shows - `title` (the base
name, no extension), `fileNameExtension` and `photoType` - built in
`saveMediaFile` from the same `FileModel` list, so `permUris[index]` lines up
with `fileModels[index]`.

The surrounding `try` catches only the synchronous throw from
`getPhotoAccessHelper`. The dialog itself is a promise and its rejection - the
ordinary outcome when the user dismisses the sheet - escapes as an unhandled
rejection, leaving `accessHelper` unreleased and `preContent` still equal to
the input, so the next tap re-opens a dialog from a helper that was never
cleaned up.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" }
]
```

- `INTERNET` is `system_grant`, so no `reason` or `usedScene` is needed and no
  dialog appears.
- **No storage or media permission is declared, and none is needed.** That is
  the point of `showAssetsCreationDialog`: the user's confirmation in the
  system sheet *is* the grant, scoped to those files.
- `deviceTypes` is `["phone", "tablet", "2in1"]`.
- The home skill uses `"ohos.want.action.home"` where the other three samples
  in this industry use `"action.system.home"`. Both resolve; the sample is just
  inconsistent with its siblings.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **`maxLimit` is 10 MB per response.** `http.request` buffers the whole body
  in memory; for genuinely large video use `requestInStream` instead of raising
  this number.
- Every downloaded `ArrayBuffer` is held in `fileModels` on the main thread
  until the save completes - 100 files at 10 MB is a gigabyte. The sandbox copy
  already exists, so a production version would re-read from disk instead of
  keeping the bytes.
- There is no per-item progress, no pause and no per-item retry. The UI has one
  spinner for the whole batch; failed URLs are logged in the worker and
  disappear from the results.
- `content-type` decides the extension; a server sending
  `application/octet-stream` falls back to the URL-extension branch, and one
  sending nothing useful gets skipped entirely.

## Pitfalls

- **`HW-15-0069`** (B/medium, confirmed): one failed group deadlocks the batch
  forever. `taskpool.execute` has no `.catch` (`TaskExecutor.ets:36-45`), so a
  rejection never runs the callback, `taskState[index]` stays `false`,
  `isDownloading` stays `true`, the spinner never stops and `terminateTask` is
  never called; `showAssetsCreationDialog` is equally unguarded
  (`MainPage.ets:125-147`) and leaks the `PhotoAccessHelper` when the user
  dismisses the sheet; and `cancelTask` is defined but invoked from nowhere, so
  leaving the page leaves LongTasks downloading. Fix: `.catch` on both promises,
  completing the group protocol and releasing the helper, plus `cancelTask` in
  `aboutToDisappear`.
- **`HW-15-0070`** (B/medium, confirmed): sandbox filenames are built as
  `fileName + respData.fileExtension` with no dot (`ProcessFunction.ets:80-81`),
  producing `image-1750000000000jpeg`, and the only unique component is a
  millisecond timestamp - up to twenty concurrent groups can finish in the same
  millisecond and overwrite each other. Fix: insert the `.` and add an index or
  a URL hash to the name.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-taskpool.md` -
  `LongTask`, `execute`, `cancel`, and the `terminateTask` lifetime rule
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-taskpool
- `documentation/harmonyos-guides/03_application-framework/taskpool-vs-worker.md` -
  when a taskpool task is the right unit of concurrency
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/taskpool-vs-worker
- `documentation/harmonyos-references/03_system/js-apis-http.md` -
  `HttpRequestOptions`, `maxLimit`, `expectDataType`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-http
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` -
  `getPhotoAccessHelper`, `showAssetsCreationDialog`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-guides/03_application-framework/save-user-file.md` -
  the sandbox-then-confirm save flow and its permission model
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/save-user-file
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `openSync`, `OpenMode`, `writeSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/02_application-framework/js-apis-data-uniformtypedescriptor.md` -
  `getUniformDataTypeByFilenameExtension` and `belongsTo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-uniformtypedescriptor
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` -
  `ohos.permission.INTERNET`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
