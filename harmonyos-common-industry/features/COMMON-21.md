---
id: COMMON-21
title: File download management - agent download tasks with pause/resume, progress and a local file manager
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/21_agent_download_control.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/agent_download_control-0000002262990970
sample: huawei_industry_tree/19_common_technical_solutions/downloads/AgentDownloadControl.zip
kits: ["@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: ["request.agent.create", "request.agent.Config", "request.agent.Action", "request.agent.Mode", "request.agent.Task", "Task.start", "Task.pause", "Task.resume", "Task.on('progress')", "Task.on('completed')", "Task.on('failed')", "Task.off('progress')", "Task.off('completed')", "Task.off('failed')", "request.agent.remove", "request.agent.show", "request.agent.Progress", "fileIo.listFileSync", "fileIo.statSync", "fileIo.unlinkSync", "util.format", "@Watch", "@Link", "@StorageLink", Progress]
permissions: ["ohos.permission.INTERNET"]
min_api: 20
modules: [entry]
findings: [HW-19-0063, HW-19-0064, HW-19-0065, HW-19-0066, HW-19-0067, HW-19-0182, HW-19-0183]
status: verified-with-fixes
---

## When to use

Load this card when the product needs a **download manager**: a list of remote
files, one tap to start, tap again to pause, tap again to resume, live progress,
and a second screen listing what has already been downloaded with the option to
delete it.

Two halves, and they are independent: `request.agent` runs the transfer,
`@ohos.file.fs` inspects and cleans up the sandbox afterwards.

## Feature checklist

The application must:

- Model each row as a small state machine: prepare -> downloading -> paused ->
  finished / already-present.
- Drive the row's icon from that state with `@Watch`.
- Create one `request.agent` task per download, choosing foreground or background
  mode.
- Subscribe to `progress`, `completed` and `failed`, and **release those
  subscriptions** when the row goes away (HW-19-0065).
- Show a percentage and a `Progress` bar while running.
- Advance to the downloading state **only if the task was actually created**
  (HW-19-0066).
- Remove the finished task from the agent's task list.
- Recognise on appear that a file is already present, and offer delete instead of
  download.
- Format sizes with the right units (HW-19-0064).
- Delete exactly once per tap (HW-19-0063).

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup; publishes the `UIContext` into `AppStorage` under `uiContext` |
| `constant/Constants.ets` | the status codes (`STATUS_PREPARE`, `STATUS_DOWNLOADING`, `STATUS_PAUSE`, `STATUS_FINISH`, `STATUS_COMPLETED`), size and progress constants |
| `model/FileModel.ets` | one downloadable entry: name and resource URL |
| `utils/FileUtil.ets` | `fileUtils` singleton over `context.cacheDir`: `findFileByName`, `getFileSizeByName`, `cleanFileByName`, `listFiles`, `unitConversion` |
| `utils/AppRouter.ets` | navigation helper |
| `components/FileItem.ets` | one download row: the state machine, the agent task, the three event handlers |
| `components/DownloadedFile.ets` | one row on the downloaded-files screen |
| `pages/DownloadFilePage.ets` | the download list plus the background-mode toggle |
| `pages/DownloadedFilePage.ets` | the already-downloaded list, built from `fileUtils.listFiles()` |

**The state machine is the design.** `@State @Watch('statusChange') status`
drives `imageResource`, and the same `status` selects what a tap does:

| status | icon | tap action |
| --- | --- | --- |
| `STATUS_PREPARE` | download | create the task, start it, go to downloading |
| `STATUS_DOWNLOADING` | downloading | pause, go to paused |
| `STATUS_PAUSE` | pause | resume, go to downloading |
| `STATUS_FINISH` | delete | delete the file and drop the list row |
| `STATUS_COMPLETED` | delete | same |

`STATUS_FINISH` is reached from the task's `completed` event; `STATUS_COMPLETED`
is set in `aboutToAppear` when `findFileByName` already sees the file in the
cache directory. They behave identically - which is why the missing `break`
between them is easy to miss and still wrong (HW-19-0063).

**Task lifecycle.** `create()` -> `downloadTaskInit()` builds a
`request.agent.Config` (`Action.DOWNLOAD`, mode from the page's toggle, `url`,
`saveas: fileName`, `overwrite: true`, `gauge: true`) and calls
`request.agent.create(context, config)`. The three handlers are attached to the
returned `Task`; `completed` sets the progress to 100%, moves to `STATUS_FINISH`
and calls `request.agent.remove(tid)`; `failed` logs and removes as well.

**Files.** Everything lands in `context.cacheDir`; `FileUtil` is a thin
synchronous wrapper - `listFileSync`, `statSync`, `unlinkSync` - and a size
formatter.

## Implementation steps

1. **Define the status constants** and hold the current one in
   `@State @Watch('statusChange') status`.
2. **Map status to icon** in the watch handler, and map status to behaviour in
   the tap handler - one `switch` each, and give every case a `break`
   (HW-19-0063).
3. **Detect the already-downloaded case in `aboutToAppear`**:
   `fileUtils.findFileByName(name)`, and if present set `STATUS_COMPLETED` and
   read the size from `statSync`.
4. **Build the task config**: `action: Action.DOWNLOAD`, `mode` from the
   foreground/background toggle, `url`, `saveas`, `overwrite: true`,
   `gauge: true` (progress notifications - the reference notes this "applies only
   to background tasks").
5. **Create the task and check the result** before advancing the UI state
   (HW-19-0066).
6. **Subscribe to the three events.** `progress` updates the byte count and
   percentage - guard against an unknown total size, which the sample tracks with
   `hasGotFileSize`; `completed` finishes and removes the task; `failed` reports
   and removes.
7. **Release the subscriptions** in `aboutToDisappear` and after
   completion/failure (HW-19-0065).
8. **Control with `start` / `pause` / `resume`**, each rejecting with a
   `BusinessError` that must be caught.
9. **Manage the files** through `listFileSync` / `statSync` / `unlinkSync` on
   `cacheDir`, and format sizes with correct units (HW-19-0064).

## Verified snippets

All snippets below come from the sample project, not from the document.

### The status machine

`AgentDownloadControl.zip#AgentDownloadControl/entry/src/main/ets/components/FileItem.ets`

```ts
@Component
export struct FileItem {
  @State fileName: string = '';
  @State fileSize: string = '';
  @State @Watch('statusChange') status: number = Constants.STATUS_PREPARE;
  @State imageResource: Resource = $r('app.media.download');
  @State downloadProgress: number = 0;
  @State downloadProgressNum: number = 0;
  @StorageLink('isBackgroundOn') isBackgroundOn: boolean = false;
  @Link fileList: FileModel[];
  hasGotFileSize: boolean = true;
  downloadUrl: string = '';
  downloadTask: request.agent.Task | undefined;

  aboutToAppear(): void {
    this.fileSize = fileUtils.unitConversion(this.downloadProgressNum);
    // 已下载文件处理
    if (fileUtils.findFileByName(this.fileName) === Constants.BOOL_TRUE) {
      this.status = Constants.STATUS_COMPLETED;
      this.fileSize = fileUtils.getFileSizeByName(this.fileName);
    }
  }

  // 文件状态变化
  statusChange() {
    switch (this.status) {
      case Constants.STATUS_PREPARE:
        this.imageResource = $r('app.media.download');
        break;
      case Constants.STATUS_DOWNLOADING:
        this.imageResource = $r('app.media.downloading');
        break;
      case Constants.STATUS_PAUSE:
        this.imageResource = $r('app.media.pause');
        break;
      case Constants.STATUS_FINISH:
        this.imageResource = $r('app.media.delete');
        break;
      case Constants.STATUS_COMPLETED:
        this.imageResource = $r('app.media.delete');
        break;
      default:
        this.imageResource = $r('app.media.download');
    }
  }
}
```

### The tap dispatcher (as shipped - see HW-19-0063 and HW-19-0066)

`AgentDownloadControl.zip#AgentDownloadControl/entry/src/main/ets/components/FileItem.ets`

```ts
// 不同状态下不同点击事件
async fileOperate(fileName: string) {
  switch (this.status) {
    case Constants.STATUS_PREPARE:
      await this.create(fileName);
      this.start();
      this.status = Constants.STATUS_DOWNLOADING;   // FIX (HW-19-0066): only on success
      break;
    case Constants.STATUS_DOWNLOADING:
      this.pause();
      this.status = Constants.STATUS_PAUSE;
      break;
    case Constants.STATUS_PAUSE:
      this.resume();
      this.status = Constants.STATUS_DOWNLOADING;
      break;
    case Constants.STATUS_FINISH:
      fileUtils.cleanFileByName(this.fileName);
      this.removeByName(this.fileName);
      // FIX (HW-19-0063): missing break - falls through and deletes twice
    case Constants.STATUS_COMPLETED:
      fileUtils.cleanFileByName(this.fileName);
      this.removeByName(this.fileName);
  }
}
```

### Task creation and the three handlers

`AgentDownloadControl.zip#AgentDownloadControl/entry/src/main/ets/components/FileItem.ets`

```ts
// 下载任务配置
async downloadTaskInit() {
  let taskMode = request.agent.Mode.FOREGROUND;
  if (this.isBackgroundOn) {
    taskMode = request.agent.Mode.BACKGROUND;
    hilog.info(DOMAIN_ID, TAG, '/Request ---Background Task');
  }
  let config: request.agent.Config = {
    action: request.agent.Action.DOWNLOAD,
    mode: taskMode,
    url: this.downloadUrl,
    saveas: this.fileName,
    overwrite: true,
    gauge: true,
  };
  try {
    this.downloadTask = await request.agent.create(this.getUIContext().getHostContext()!, config);

    this.downloadTask.on('progress', (progress) => {
      this.downloadProgressNum = progress.processed;
      let all = progress.sizes[progress.index];
      if (all === Constants.NO_SIZE) {
        this.hasGotFileSize = false;
      } else {
        this.hasGotFileSize = true;
        this.downloadProgress = Number.parseFloat((this.downloadProgressNum / all *
        Constants.TOTAL_PROGRESS).toFixed(Constants.FRACTION_DIGITS));
      }
    });

    this.downloadTask.on('failed', (progress) => {
      request.agent.show(this.downloadTask?.tid)   // FIX (HW-19-0067): consume the TaskInfo
        .catch((err: BusinessError) => {
          hilog.error(DOMAIN_ID, TAG,
            `/Request ---Failed to removing a download task, Code: ${err.code}, message: ${err.message}`);
        });
      hilog.error(DOMAIN_ID, TAG, '/Request ---failed:  ' + progress);
      request.agent.remove(this.downloadTask?.tid)
        .then(() => { hilog.error(DOMAIN_ID, TAG, '/Request ---check remove'); })
        .catch((err: BusinessError) => {
          hilog.error(DOMAIN_ID, TAG,
            `/Request ---Failed to removing a download task, Code: ${err.code}, message: ${err.message}`);
        });
    });

    this.downloadTask.on('completed', () => {
      this.downloadProgress = Constants.TOTAL_PROGRESS;
      this.status = Constants.STATUS_FINISH;
      request.agent.remove(this.downloadTask?.tid)
        .then(() => { hilog.info(DOMAIN_ID, TAG, '/Request ---check remove'); })
        .catch((err: BusinessError) => {
          hilog.error(DOMAIN_ID, TAG,
            `/Request ---Failed to removing a download task, Code: ${err.code}, message: ${err.message}`);
        });
    });
  } catch (error) {
    hilog.info(DOMAIN_ID, TAG, `code: ${error.code}, message: ${error.message}`);
  }
}
```

### Start / pause / resume

`AgentDownloadControl.zip#AgentDownloadControl/entry/src/main/ets/components/FileItem.ets`

```ts
// 开始下载
async start() {
  this.downloadTask?.start().then(() => {
    hilog.info(DOMAIN_ID, TAG, 'downloading...');
  }).catch((err: BusinessError) => {
    hilog.error(DOMAIN_ID, TAG, `code: ${err.code}, message: ${err.message}`);
  });
}

// 暂停下载
async pause() {
  this.downloadTask?.pause().then(() => {
    hilog.info(DOMAIN_ID, TAG, 'success to pause');
  }).catch((err: BusinessError) => {
    hilog.error(DOMAIN_ID, TAG, `failed to pause, code: ${err.code}, message: ${err.message}`);
  });
}

// 重新下载
async resume() {
  this.downloadTask?.resume().then(() => {
    hilog.info(DOMAIN_ID, TAG, 'success to resume');
  }).catch((err: BusinessError) => {
    hilog.error(DOMAIN_ID, TAG, `failed to resume, code: ${err.code}, message: ${err.message}`);
  });
}
```

### The file utility

`AgentDownloadControl.zip#AgentDownloadControl/entry/src/main/ets/utils/FileUtil.ets`

```ts
let ui = AppStorage.get('uiContext') as UIContext;

export class FileUtil {
  context: common.UIAbilityContext = ui.getHostContext() as common.UIAbilityContext;

  findFileByName(fileName: string): number {
    let result: number = 0;
    try {
      let files: string[] = fileIo.listFileSync(this.context.cacheDir);
      if (files.includes(fileName)) {
        result = Constants.BOOL_TRUE;
      } else {
        result = Constants.BOOL_FALSE;
      }
    } catch (err) {
      hilog.error(DOMAIN_ID, TAG, 'listFiles err = %{public}s', JSON.stringify(err));
    }
    return result;
  }

  cleanFileByName(fileName: string): void {
    try {
      let files: string[] = fileIo.listFileSync(this.context.cacheDir);
      if (files.includes(fileName)) {
        fileIo.unlinkSync(util.format('%s/%s', this.context.cacheDir, fileName));
      }
    } catch (err) {
      hilog.error(DOMAIN_ID, TAG, 'listFiles err = %{public}s', JSON.stringify(err));
    }
  }

  listFiles(): Array<string> {
    let files: string[] = [];
    try {
      files = fileIo.listFileSync(this.context.cacheDir);
      for (let index = 0; index < files.length; index++) {
        let fileName = files[index];
        let stat: fileIo.Stat = fileIo.statSync(util.format('%s/%s', this.context.cacheDir, fileName));
        files[index] = util.format('%s,%d', fileName, stat.size);
      }
    } catch (err) {
      hilog.error(DOMAIN_ID, TAG, 'listFiles err = %{public}s', JSON.stringify(err));
    }
    return files;
  }

  // FIX (HW-19-0064): the third branch divides by MB but formats as KB
  unitConversion(size: number): string {
    if (size < Constants.SIZE_TRANSITION) {
      return util.format('%dB', size);
    } else if (size < Math.pow(Constants.SIZE_TRANSITION, Constants.SIZE_MB)) {
      return util.format('%sKB', (size / Constants.SIZE_TRANSITION).toFixed(Constants.FIXED_POINT));
    } else if (size < Math.pow(Constants.SIZE_TRANSITION, Constants.SIZE_GB)) {
      return util.format('%sKB',
        (size / Math.pow(Constants.SIZE_TRANSITION, Constants.SIZE_MB)).toFixed(Constants.FIXED_POINT));
    } else {
      return util.format('%sGB',
        (size / Math.pow(Constants.SIZE_TRANSITION, Constants.SIZE_GB)).toFixed(Constants.FIXED_POINT));
    }
  }
}

export const fileUtils = new FileUtil();
```

### The foreground/background toggle

`AgentDownloadControl.zip#AgentDownloadControl/entry/src/main/ets/pages/DownloadFilePage.ets`

```ts
@StorageLink('isBackgroundOn') isBackgroundOn: boolean = false;

Toggle({ type: ToggleType.Switch, isOn: $$this.isBackgroundOn })
  .onChange((value: boolean) => {
    this.isBackgroundOn = value;
  });

// each row
FileItem({ fileName: item.getName(), downloadUrl: item.getResource(), fileList: this.fileList });
```

## Permissions & config

`ohos.permission.INTERNET` is genuinely required and is the only permission the
document lists and the sample declares.

`AgentDownloadControl.zip#AgentDownloadControl/entry/src/main/module.json5`:

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" }
]
```

Note what is **not** needed: files land in `context.cacheDir`, inside the
application sandbox, so no media or file-access permission is involved, and
`request.agent`'s background mode needs no extra permission of its own.

The module also declares the usual single `EntryAbility` with the home skill and
an `EntryBackupAbility` backup extension.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later. The `request.agent` module dates
  from API 10.
- **`gauge` applies to background tasks only**: "Whether to send progress
  notifications. This parameter applies only to background tasks." Setting it on
  a foreground task has no effect.
- **Foreground tasks outrank background ones**: "The priority of a foreground
  task is higher than that of a background task."
- **`progress.sizes[progress.index]` can be unknown.** The sample compares it
  against `Constants.NO_SIZE` and skips the percentage when the server did not
  report a length - keep that guard, otherwise the percentage becomes `Infinity`
  or `NaN`.
- **`request.agent.remove(tid)` removes the task from the agent's list**, not the
  downloaded file; the file removal is `fileIo.unlinkSync`.
- **`FileUtil` resolves its context at module load** from
  `AppStorage.get('uiContext')`, so the ability must have published it before the
  module is first imported.
- **Everything lives in `cacheDir`**, which the system may reclaim - and which the
  cache-clearing feature of COMMON-07 deletes wholesale. For downloads the user
  expects to keep, use `filesDir` instead.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **`case Constants.STATUS_FINISH` has no `break`, which is incorrect.** It falls
  through into the identical `STATUS_COMPLETED` case, so the delete path runs
  twice - and `removeByName` splices by first match, so with duplicate names the
  second pass can remove the wrong row. The document's snippet has the `break`.
  (HW-19-0063)
- **`unitConversion` labels megabytes as KB, which is incorrect.** The third
  branch divides by 1024² and formats `'%sKB'`; a 5 MB file reads as `5.0KB`.
  (HW-19-0064)
- **The three `Task.on` subscriptions are never released, which is incorrect.**
  There is no `off` and no `aboutToDisappear`; a row scrolled away keeps writing
  state, and re-downloading registers three more handlers. (HW-19-0065)
- **The status advances to downloading even when `request.agent.create` threw,
  which is incorrect.** `create` swallows the error into an info log, `start()`
  no-ops through `?.`, and the row shows a 0% progress bar forever.
  (HW-19-0066)
- **The `failed` handler discards the `TaskInfo` from `request.agent.show` and
  logs a remove-shaped message for it, which is incorrect** - the failure cause
  is exactly what that call would have provided. (HW-19-0067)
- **`STATUS_FINISH` and `STATUS_COMPLETED` are two names for the same UI state.**
  The only difference is provenance: finished during this session versus already
  present at appear. Do not add behaviour to one without the other.
- **`FileUtil.listFiles()` returns `"name,size"` strings**, not objects - the
  downloaded-files page parses that string back apart. Keep the separator in mind
  if file names can contain a comma.

## References

- `documentation/harmonyos-references/03_system/js-apis-request.md` - the
  `request.agent` module: `create`, `Config`, `Action`, `Mode` (FOREGROUND /
  BACKGROUND and their priority), `Task.start` / `pause` / `resume`,
  `on`/`off('progress'|'completed'|'failed')`, `remove`, `show`, `TaskInfo`,
  `Progress`, and the `gauge` background-only note.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-request
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `listFileSync`, `statSync`, `unlinkSync`, `Stat.size`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-guides/03_application-framework/application-context-stage.md` -
  `cacheDir` semantics, and why a download the user expects to keep belongs in
  `filesDir`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/application-context-stage
- `documentation/harmonyos-guides/03_application-framework/arkts-appstorage.md` -
  `@StorageLink` for the shared background-mode toggle.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-appstorage
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/agent_download_control-0000002262990970
