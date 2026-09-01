---
id: UTIL-34
title: Background photo upload on 2in1 - a wait queue drained into request.agent BACKGROUND tasks
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/34_pc_upload_image.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/pc_upload_image-0000002459568386
sample: huawei_industry_tree/15_utilities/downloads/PcUploadImage.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.MediaLibraryKit", "@kit.NetworkKit", "@kit.PerformanceAnalysisKit"]
apis: ["photoAccessHelper.PhotoViewPicker", PhotoSelectOptions, "request.agent.create", "request.agent.search", "request.agent.Task", "request.agent.Mode.BACKGROUND", FormItem, "http.createHttp", "preferences.getPreferences", "fileIo.copyFileSync", LazyForEach, IDataSource, "@StorageLink", "@Watch", setInterval]
permissions: [ohos.permission.INTERNET]
min_api: 20
modules: [entry]
findings: [HW-15-0005, HW-15-0072, HW-15-0073, HW-15-0101, HW-15-0102]
status: verified-with-fixes
---

## When to use

Load this card when the app must **hand a batch of user-picked files to the
system and stop caring about them** - a photo backup, a bulk document sync, a
"send these to my desktop" action on a 2in1. The pattern is a decoupled two-
stage pipeline: the picker only appends URIs to an in-memory wait list, and a
separate drain step turns whatever is waiting into `request.agent` tasks in
`BACKGROUND` mode.

The transferable idea is that **the UI never awaits a transfer**. Selecting
images returns instantly; the upload is owned by the system's request agent,
which survives the app going to background and enforces its own concurrency.
The app's only job is to keep the wait list, respect a maximum number of live
tasks, and reflect state. That shape fits any "queue work the OS should finish"
feature - log shipping, offline outbox, media backup.

The counterpart to the upload is a plain `http` GET that lists the server's
files and renders them in a `LazyForEach` table, which is where this sample is
weakest: the names it uploads and the names it parses back do not agree
(`HW-15-0073`). **Read all three findings before copying anything from
`Upload.ets` or `FetchFiles.ets`.**

## Feature checklist

- A desktop-style two-pane layout: a left rail with a single 全部文件 (all
  files) tab, a toolbar and a file table on the right.
- The toolbar's first button opens a confirm dialog, then the system photo
  picker, limited to five images per selection.
- Selected images are queued and a toast announces that a background task took
  over.
- A drain step creates a `request.agent` upload task whenever fewer than ten
  tasks are running and the queue is non-empty.
- The task keeps running when the window is backgrounded; it can be paused and
  resumed through the shared `backTaskState`.
- The refresh button re-reads the server's file list over HTTP and re-renders
  the table with name, size and modification time.
- Folders sort before files; files sort by their recorded time.

## Architecture

One `entry` module, `deviceTypes: ["2in1"]` only. One page, one data source, one
model, five utility singletons.

```
entry/src/main/ets
├── components/CustomDataSource.ets  BasicDataSource + CustomDataSource for LazyForEach
├── entryability/EntryAbility.ets    saves the server URL into preferences, publishes uiContext
├── entrybackupability/
├── model/FileModel.ets              name/size/time/type, parsed out of the server's file name
├── pages/Index.ets                  @Entry, 443 lines: layout, picker, queueing, refresh
└── utils
    ├── Constants.ets                BackgroundTaskState enum, TASK_MAX = 10, font/size literals
    ├── FetchFiles.ets               GET ?tpl=list, parse the listing into FileModel[]
    ├── ImageUtils.ets               copy a picked URI into cacheDir under a size-tagged name
    ├── Logger.ets
    ├── Upload.ets                   the wait list, the drain timer, request.agent tasks
    └── UrlUtil.ets                  server URL in preferences ('server_url' / 'url')
```

The documented tree does not match the zip: the document places
`CustomDataSource.ets` under `common/` while the zip has it under `components/`,
and it names the backup extension `EntryBackAbility.ets` where the zip has
`EntryBackupAbility.ets`. No finding is filed for this - it misdirects a reader
but breaks nothing.

**The design decision worth copying** is the split between *queueing* and
*dispatching*. `uploadFilesBackground` does nothing but push URIs; a separate
`flushBackgroundTask` decides when a task may be created, gated on
`request.agent.search({ state: RUNNING })` against `TASK_MAX`. Because the gate
asks the system how many tasks are actually running - rather than counting
locally - the app cannot over-subscribe the agent after a restart or a crash.
`backTaskState` lives in `AppStorage`, so the page's `@StorageLink @Watch` and
the upload singleton coordinate without holding references to each other.

**The design decision worth avoiding** is in the same file: that dispatcher is
driven by an interval started in the class constructor and never stopped
(`HW-15-0005`). The gating logic is good; its clock is not.

## Implementation steps

1. **Store the server URL through `preferences`,** written once in
   `onWindowStageCreate` and read by both the uploader and the lister. The
   sample ships `const URL: string = ''` - the document says to fill it in
   before running.
2. **Publish the `UIContext` into `AppStorage`** in the `loadContent` callback,
   so singletons that are not components can still reach a `UIAbilityContext`.
3. **Pick with `PhotoViewPicker`,** `MIMEType = IMAGE_TYPE`,
   `maxSelectNumber = 5`. Deduplicate on arrival - `addImages` already skips
   URIs already in the list.
4. **Queue, then return.** `uploadFilesBackground` must not await anything; the
   toast is the user-visible acknowledgement.
5. **Queue only the delta**, not the whole accumulated `imageList`, or a second
   selection re-uploads the first batch (`HW-15-0072`).
6. **Drain on a timer you own**: start it when the first item is queued, clear
   it when the queue empties and on teardown (`HW-15-0005`).
7. **Gate on the live task count** with `request.agent.search({ state: RUNNING })`
   against `TASK_MAX`, then `request.agent.create(context, config)` with
   `mode: BACKGROUND` and `await task.start()`.
8. **Copy each picked file into `cacheDir` first.** `request.agent` uploads by
   relative path inside the app's cache, not by media URI, so the copy is
   mandatory - and the copied name is the name the server will show.
9. **Use one canonical name in both directions** (`HW-15-0073`): whatever suffix
   the uploader appends must be the suffix the listing parser strips, otherwise
   the "keep the existing entry" lookup never matches and every refresh
   re-stamps all timestamps.
10. **Re-check `backTaskState` after `start()`**: if the user hit pause while
    the task was being created, pause it immediately - the sample does this and
    it is easy to forget.

## Verified snippets

All snippets are from `PcUploadImage.zip`. Corrected forms are marked.

**The queue and its drain — `entry/src/main/ets/utils/Upload.ets`** (corrected, see `HW-15-0005`)

```typescript
class Upload {
  private waitList: Array<string> = [];
  private backgroundTask: request.agent.Task | undefined = undefined;
  private flushTimer: number = -1;                     // FIX: the sample keeps no id

  private startFlushing() {                            // FIX: sample starts this in the constructor
    if (this.flushTimer === -1) {
      this.flushTimer = setInterval(() => {
        this.flushBackgroundTask();
      }, 2000);
    }
  }

  private stopFlushing() {                             // FIX: absent in the sample
    if (this.flushTimer !== -1) {
      clearInterval(this.flushTimer);
      this.flushTimer = -1;
    }
  }

  async uploadFilesBackground(fileUris: Array<string>): Promise<void> {
    if (fileUris.length === 0) {
      return;
    }
    fileUris.forEach((item: string) => {
      this.waitList.push(item);
    });
    this.startFlushing();                              // FIX: timer follows the work
  }

  async flushBackgroundTask() {
    let tasks = await request.agent.search({
      state: request.agent.State.RUNNING
    });
    let state = AppStorage.get<number>('backTaskState');
    if (state === BackgroundTaskState.RUNNING) {
      if (tasks.length < TASK_MAX && this.waitList.length > 0) {
        this.createBackgroundTask(this.waitList);
        this.waitList = [];
      } else {
        if (this.backgroundTask === undefined || tasks.indexOf(this.backgroundTask.tid) === -1) {
          AppStorage.setOrCreate('backTaskState', BackgroundTaskState.NONE);
          this.backgroundTask = undefined;
          this.stopFlushing();                         // FIX: nothing left to drain
        }
      }
    }
  }
}

export const requestUpload = new Upload();
```

**The gate is the interesting half.** `request.agent.search({ state: RUNNING })`
asks the *system* how many transfers are live, so the ten-task ceiling holds
across app restarts and across other entry points that create tasks. The `else`
branch is the completion detector: when the tracked task's `tid` is no longer
in the running set, the batch is finished and `backTaskState` returns to `NONE`,
which is the signal the page watches.

As shipped, the class body is `constructor() { setInterval(() => this.flushBackgroundTask(), 2000); }`
with no stored id and no `clearInterval` anywhere in the project. The module
exports a singleton, so importing the file starts a 2-second poll that runs for
the process lifetime, calling `request.agent.search` forever even when the user
never uploads anything (`HW-15-0005`). The corrected form above changes nothing
about the gating - only who owns the clock.

**Creating the task — same file** (as shipped)

```typescript
private config: request.agent.Config = {
  action: request.agent.Action.UPLOAD,
  headers: HEADER,                                  // { 'Content-Type': 'multipart/form-data' }
  url: '',
  mode: request.agent.Mode.FOREGROUND,
  method: 'POST',
  title: 'upload',
  network: request.agent.Network.ANY,
  data: [],
  token: UPLOAD_TOKEN
};

async createBackgroundTask(fileUris: Array<string>) {
  const uiContext: UIContext | undefined = AppStorage.get('uiContext');
  if (!uiContext) {
    return;
  }
  const context = uiContext.getHostContext() as common.UIAbilityContext;
  this.config.url = await urlUtils.getUrl(context);
  this.config.data = await this.getFilesAndData(context.cacheDir, fileUris);
  this.config.mode = request.agent.Mode.BACKGROUND;
  try {
    this.backgroundTask = await request.agent.create(context, this.config);
    await this.backgroundTask.start();
    let state = AppStorage.get<number>('backTaskState');
    if (state === BackgroundTaskState.PAUSE) {
      await this.backgroundTask.pause();              // the user paused mid-creation
    }
  } catch (err) {
    logger.error(TAG, `task  err, err  = ${JSON.stringify(err)}`);
  }
}

private async getFilesAndData(cacheDir: string, fileUris: Array<string>): Promise<Array<request.agent.FormItem>> {
  let files: Array<request.agent.FormItem> = [];
  for (let i = 0; i < fileUris.length; i++) {
    let imagePath = await this.imageUtils.copyFileToCache(cacheDir, fileUris[i]);
    let file: request.agent.FormItem = {
      name: imagePath.split('cache/')[1],
      value: {
        path: './' + imagePath.split('cache/')[1]      // paths are relative to the app cache
      }
    };
    files.push(file);
  }
  return files;
}
```

**`mode: BACKGROUND` and the relative path are the two lines that matter.** A
`FOREGROUND` task dies with the foreground; `BACKGROUND` hands the transfer to
the system agent, which continues while the window is hidden and reports through
its own task events. And `value.path` must be `./name` *relative to the app's
cache directory* - handing it a `file://media/...` picker URI or an absolute
sandbox path fails, which is exactly why every picked image is copied into
`cacheDir` first.

The re-check of `backTaskState` after `start()` closes a real race: the config
is built asynchronously (URL from preferences, N file copies), so the user can
press pause while the task is still being created, and without this the task
would start anyway.

**Only queue what is new — `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-15-0072`)

```typescript
@StorageLink('backTaskState') @Watch('stateChange') backTaskState: BackgroundTaskState = BackgroundTaskState.NONE;
imageList: Array<string> = [];
private queuedUris: Set<string> = new Set();          // FIX: nothing tracks what was already sent

stateChange() {
  if (this.backTaskState === BackgroundTaskState.NONE) {
    this.imageList = [];
  }
}

addImages = (images: Array<string>) => {
  images.forEach((item: string) => {
    if (!this.imageList.includes(item)) {
      this.imageList.push(item);
    }
  });
};

uploadFiles() {
  const pending = this.imageList.filter((uri) => !this.queuedUris.has(uri));   // FIX
  if (pending.length === 0) {
    return;
  }
  pending.forEach((uri) => this.queuedUris.add(uri));                          // FIX
  AppStorage.setOrCreate('backTaskState', BackgroundTaskState.RUNNING);
  requestUpload.uploadFilesBackground(pending);        // FIX: sample passes this.imageList
  this.getUIContext()
    .getPromptAction()
    .showToast({ message: $r('app.string.background_task_notification'), bottom: TOAST_BOTTOM });
}
```

**`imageList` is a session accumulator, not a batch.** It is only emptied by
`stateChange` when the state returns to `NONE`, i.e. when *all* transfers have
finished. Select five images, then select five more while the first batch is
still uploading, and the shipped `uploadFiles` hands the uploader all ten - the
first five a second time (`HW-15-0072`). The fix is one filter: remember what
has been queued and pass only the delta.

Note the coordination mechanism here, which is worth keeping: the page and the
upload singleton share exactly one value, `backTaskState` in `AppStorage`, read
by the page through `@StorageLink` with a `@Watch` and by the uploader through
`AppStorage.get`. Neither imports the other's state.

**The name round trip — `utils/ImageUtils.ets` and `model/FileModel.ets`** (corrected, see `HW-15-0073`)

```typescript
// ImageUtils.copyFileToCache — shipped form appends '.jpg' to a name that already has an extension
async copyFileToCache(cacheDir: string, uri: string): Promise<string> {
  let id = uri.split('/').pop();
  let file = await fileIo.open(uri);
  let size = fileIo.statSync(file.fd).size;
  if (id !== undefined) {
    let lastDotIndex = id.lastIndexOf('.');
    let baseName = id.substring(0, lastDotIndex);
    let extension = id.substring(lastDotIndex);        // '.png' stays '.png'
    id = baseName + '_' + size + extension;            // 'photo_284133.png'
  }
  let imagePath = `${cacheDir}/${id}`;                 // FIX: sample writes `${cacheDir}/${id}.jpg`
  try {
    fileIo.copyFileSync(file.fd, imagePath);
  } catch (err) {
    logger.info(TAG, `copyFileToCache copyFileSync err = ${err}`);
  }
  await fileIo.close(file.fd);
  return imagePath;
}
```

```typescript
// FetchFiles.fetchFiles — the lookup that preserves an existing row's timestamp
let name = splitFiles.pop();                           // 'photo_284133.png' as listed by the server
if (name) {
  let existingFile = oldFiles.find(file => file.name === name);   // FIX: sample compares against a
  if (existingFile) {                                             // '.jpg'-stripped variant instead
    files.push(existingFile);
  } else {
    files.push(new FileModel(name, false, [tempFiles[i]]));
  }
}
```

**One name, two spellings, and the join never matches.** The uploader appends
`.jpg` to a name that already ends in its real extension, so a PNG reaches the
server as `photo_284133.png.jpg`. `FileModel`'s constructor then splits the
size back out of the name and rewrites `this.name`, while `FetchFiles` compares
the *raw* listing name against a differently-trimmed variant. The `find` never
hits, so every refresh constructs new `FileModel`s with `time = new Date()` and
the whole table's modification times jump to "now" (`HW-15-0073`). Two edits fix
it: stop appending the extra extension, and compare the same string on both
sides.

The size trick itself is worth keeping in mind - the server listing carries no
size, so the sample encodes the byte count into the file name at upload time and
parses it back at display time. It is a hack, but it is the reason the table can
show sizes at all with a dumb file server.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.INTERNET",
    "reason": "$string:media_internet_permission",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
]
```

- `INTERNET` is `system_grant`: no runtime request, but `reason` and
  `usedScene` are still supplied here and are good practice.
- No media permission is needed. `PhotoViewPicker` is a system picker - it
  returns URIs the app is temporarily allowed to read, which is why the sample
  can copy them into cache without `READ_IMAGEVIDEO`.
- `deviceTypes` is `["2in1"]` only. This is the PC-shaped member of the family;
  the layout assumes a wide window (a 12% rail plus an 88% content pane).
- The server URL is empty in `EntryAbility.ets` (`const URL: string = ''`) and
  must be filled in before the sample does anything - the document says so in a
  说明 note.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The server is assumed to be a plain HTTP file server that answers
  `GET /?tpl=list&folders-filter=&recursive` with `\r\n`-separated paths and
  accepts `multipart/form-data` POSTs. The listing parser is written to that
  format and does `tempFiles[i].split('//')[1]` with no guard, so a blank
  trailing line or a different listing format throws into the `catch` and the
  table renders empty.
- `TASK_MAX` is 10 and each task carries at most five files, since the picker
  caps a selection at five.
- Sizes shown for non-image types are constants (`MP4_SIZE`, `DOC_SIZE`, …) -
  the file table's numbers are only real for the images the app itself uploaded.
- There is no delete, no create-folder and no download: three of the five
  toolbar buttons only toggle their own highlight.

## Pitfalls

- **`HW-15-0005`** (B/low, confirmed): `Upload`'s constructor starts a 2-second
  `setInterval` that is never cleared, and the module exports a singleton - so
  importing the file starts a permanent poll of `request.agent.search`, even
  with an empty queue. Fix: keep the timer id, start it on the first queued
  item, and `clearInterval` when the queue drains or the owner is destroyed.
- **`HW-15-0072`** (B/medium, confirmed): a second selection re-uploads the
  entire previous batch. `imageList` is only cleared when `backTaskState`
  returns to `NONE`, and `uploadFiles` always passes the whole list with no
  dedupe. Fix: track flushed URIs and queue only the delta.
- **`HW-15-0073`** (B/medium, confirmed): the uploaded name and the parsed name
  never agree - `ImageUtils` appends `.jpg` to a name that already has an
  extension (mislabelling PNGs as `x_123.png.jpg`), while `FetchFiles` looks up
  a `.jpg`-stripped variant against the raw listing name. The "keep the existing
  entry" branch is dead, so every refresh resets all displayed modification
  times. Fix: one canonical name format on both sides.
- Not a filed finding, but note it: the document's project tree lists
  `common/CustomDataSource.ets` and `EntryBackAbility.ets`, while the zip has
  `components/CustomDataSource.ets` and `EntryBackupAbility.ets`.
- Also unfiled: `FetchFiles` only calls `httpRequest.destroy()` on the error
  path, so successful refreshes leak an `HttpRequest` object each time.

## References

- `documentation/harmonyos-references/03_system/js-apis-request.md` - `request.agent.create`, `search`, `Task.start/pause/resume`, `Config`, `FormItem`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-request
- `documentation/harmonyos-guides/04_system/upload-download.md` - the upload task flow and the relative-path rule for `FormItem.value.path`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/upload-download
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` - `PhotoViewPicker.select`, `PhotoSelectOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` - `getPreferences`, `put`, `flush`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-lazyforeach.md` - `IDataSource` and `notifyDataReload`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-lazyforeach
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.INTERNET`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `huawei_industry_tree/15_utilities/docs/34_pc_upload_image.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pc_upload_image-0000002459568386
