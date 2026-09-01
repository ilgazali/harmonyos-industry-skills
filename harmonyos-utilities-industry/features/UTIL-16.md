---
id: UTIL-16
title: TaskPool file scan - batch results back from a worker with sendData and cancel on page exit
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/16_taskpool_query.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/taskpool_query-0000002292913090
sample: huawei_industry_tree/15_utilities/downloads/TaskPoolQuery.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [common, fileIo, hilog, taskpool, window, "taskpool.Task", "taskpool.execute", "taskpool.cancel", "Task.sendData", "Task.onReceiveData", "Task.isCanceled", "@Concurrent", LazyForEach, IDataSource, Navigation, NavDestination, NavPathStack, CustomDialogController, "fs.listFile", "fs.statSync"]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0038, HW-15-0039, HW-15-0040, HW-15-0041, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card when a page has to **enumerate something large and slow, and
must stay scrollable while it does** - a file manager listing a directory of
100,000 entries, a media scan, a log parse, a local search index. The scan runs
on a TaskPool worker, results come back in batches while it is still running,
and leaving the page cancels it.

The transferable shape is a **three-message protocol over `sendData`**: the
worker announces `START` (clear the list), pushes `SEND_DATA` every N items
(append and reload), and finishes with `END` or `ERROR`. The host thread never
waits for the promise to carry the result - the promise only signals that the
task is over. That inversion is what makes the first rows appear in a fraction
of a second on a directory the worker will take many seconds to finish.

The second transferable idea is the small `BaseTask` / `TaskExecutor` pair: a
task object owns its `taskpool.Task`, its priority, its completion callback and
its cancellation flag, and a singleton executor runs it. Two concrete tasks
(create files, query files) reuse it unchanged. **Read `HW-15-0038` first** -
that executor drops the `.catch`, which turns the sample's own documented
cancel-on-exit flow into an unhandled rejection.

## Feature checklist

- A home page with two rows: 创建文件 (create files) with a button, and
  查询文件 (query files) with a chevron that navigates.
- Tapping create opens a confirmation dialog; confirming spawns a worker that
  writes 100,000 files into the app's `filesDir` and toasts when finished,
  and while it runs the button toasts "already running" instead of starting a
  second one.
- Tapping query pushes a `NavDestination` that immediately starts a worker
  scanning `filesDir`.
- Rows appear in batches of 1000 while the scan continues; the list stays
  scrollable throughout, each row showing name, creation date and size.
- Leaving the query page cancels the running scan; a scan that errors clears
  the list.

## Architecture

One `entry` module, split by role rather than by screen. This is the most
structured sample in the industry pack.

```
entry/src/main/ets
├── consts/Consts.ets              QueryMsg (the 4 protocol messages) + Constants (dialog metrics)
├── entryability/EntryAbility.ets  full screen, avoid areas -> AppStorage
├── entrybackupability/
├── model
│  ├── FileDataSource.ets          IDataSource over FileModel[] + the 7 notify* methods
│  └── FileModel.ets               name / isDir / size / ctime / mtime / showTime / showSize / path
├── pages
│  ├── CustomDialog.ets            @CustomDialog CreateDialog (cancel / create)
│  ├── HomePage.ets                @Entry, Navigation, the two rows, owns the create task
│  └── QueryFilePage.ets           @Builder route target, the List + the receive callback
├── taskpool
│  ├── base
│  │  ├── BaseTask.ets             task + priority + callback + isCancel; cancelTask/onReceiveData
│  │  └── TaskExecutor.ets         singleton: execute(task) / cancelTask(task)
│  ├── CreateFileTask.ets          @Concurrent startTask -> FileUtil.createFile
│  └── QueryFileTask.ets           @Concurrent startTask -> listFile + statSync + batched sendData
└── utils
   ├── FileUtil.ets                createFile, transferTimeToStr, transferSizeToStr
   └── Logger.ets
```

The documented tree matches the zip exactly, including the per-file comments.
Routing is by `routerMap` (`$profile:router_map`): `QueryFilePage` is
registered by name against `queryFilePageBuilder`, so `HomePage` pushes it with
`this.pageInfos.pushPathByName('QueryFilePage', undefined)` and never imports
the page.

**The design decision worth copying** is the split between `BaseTask` and
`TaskExecutor`. `BaseTask` is the *state* of one job - the `taskpool.Task`
handle, the priority, the completion callback, and an `isCancel` boolean that
the worker can consult. `TaskExecutor` is a stateless singleton that knows how
to run and cancel any `BaseTask`. Concrete tasks are then eight lines each:

```typescript
export class QueryFileTask extends BaseTask {
  constructor(path: string, callback: Function) {
    super('QueryFileTask');
    this.task = new taskpool.Task(startTask, path);
    this.onReceiveData(callback);      // streaming: messages during the run
  }
}
// CreateFileTask is identical except for the last line:
//   this.callback = callback;         // one-shot: fires when the promise resolves
```

The two subclasses differ in exactly one line, and that line is the whole
distinction between a streaming task and a fire-and-forget one:
`onReceiveData(callback)` subscribes to `sendData` messages, while
`this.callback = callback` is invoked once by the executor when the promise
resolves. Same base class, two delivery models, no flags.

**Worth avoiding:** `BaseTask.isCancel` is set by `cancelTask()` and never
read by anything. The worker cannot see it - it lives on the host side of a
thread boundary - so the real cancellation check inside the concurrent function
is `taskpool.Task.isCanceled()`. The field is a decoy; delete it rather than
copy it.

## Implementation steps

1. **Define the message vocabulary first** (`QueryMsg.START` / `SEND_DATA` /
   `ERROR` / `END`) in a module both threads import. `sendData` carries
   arbitrary arguments; a named enum is what keeps the switch honest.
2. **Mark the worker function `@Concurrent`** and keep it top-level. It cannot
   close over anything - everything it needs arrives as a serialisable
   argument.
3. **Batch the sends.** The sample flushes every 1000 files and clears the
   local array; the comment says 防止子线程OOM (prevent worker OOM), and it also
   prevents one enormous serialisation at the end.
4. **Check `taskpool.Task.isCanceled()` inside the loop** and bail out with an
   `ERROR` message. `taskpool.cancel` does not stop a task that has already
   been dispatched to a worker - only this check does.
5. **Attach `.catch` to `taskpool.execute`** and run the same cleanup as the
   success path (`HW-15-0038`). A cancelled task rejects; without a catch it is
   an unhandled rejection, and the state that gates the button is never reset.
6. **Reset the task handle in both outcomes.** `HomePage.task` is the "already
   running" gate; leave it set after a failure and the create button toasts
   forever (`HW-15-0038`).
7. **Feed a `LazyForEach` from an `IDataSource`,** not a `@State` array. The
   sample is designed for 100,000 rows; an `@State` array would re-diff the
   whole list on every batch.
8. **Give the key generator a real unique key** - `item.path`, not the item
   itself (`HW-15-0039`).
9. **Convert sizes from bytes**, and start the unit table at `'B'`
   (`HW-15-0040`).
10. **Do not chain the same attribute twice** on a `Text`; a mistyped
    `.fontWeight` silently overrides the real one (`HW-15-0041`).
11. **Cancel in `aboutToDisappear`** so a scan does not outlive its page.

## Verified snippets

All snippets are from `TaskPoolQuery.zip`. Corrected forms are marked.

**The executor — `entry/src/main/ets/taskpool/base/TaskExecutor.ets`** (corrected, see `HW-15-0038`)

```typescript
public execute<T>(task?: BaseTask): void {
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
    Logger.error(TAG, 'executeTask failed: ' + error.message);
    try {
      if (task.callback) {
        task.callback(undefined);               // FIX: same cleanup as the success path
      }
    } catch (e) {
      Logger.error(TAG, 'executeTask callback error: ' + e.toString());
    }
  });
}
```

**The missing `.catch` is not a hypothetical.** The reference is explicit about
what cancellation does: if the task is still queued, "the task will not be
executed after being canceled, and an exception indicating task cancellation is
returned"; if it has already been dispatched, "the execution result is returned
in the catch branch". The sample's own documented flow -
`aboutToDisappear` → `cancelTask` → `taskpool.cancel` - therefore rejects this
promise every single time a user leaves the query page mid-scan, and with no
catch attached that is an unhandled rejection.

The consequence on the create side is worse than a log line. `HomePage` gates
the button on `this.task` (`onClick` toasts 创建中 and returns when it is set),
and the only assignment of `this.task = undefined` sits inside the completion
callback that the resolve path invokes (`HomePage.ets:86-89`).
One failed create task leaves `this.task` set for the lifetime of the page, and
the button answers every later tap with 创建中 ("creating") without ever
creating anything. Invoking the callback from the catch branch too is the
minimum fix; a real implementation would distinguish the two outcomes in the
toast.

**The worker — `entry/src/main/ets/taskpool/QueryFileTask.ets`** (as shipped)

```typescript
@Concurrent
async function startTask(path: string = ''): Promise<void> {
  const TAG = 'QueryFileTask';
  taskpool.Task.sendData([], QueryMsg.START);
  let fileList: FileModel[] = [];
  if (path === '') {
    taskpool.Task.sendData(fileList, QueryMsg.ERROR);
    return;
  }
  try {
    let filenames = await fs.listFile(path);
    const BATCH_SEND_NUM = 1000;                     // 分批发送文件数量
    for (let i = 0; i < filenames.length; i++) {
      let filePath = path + '/' + filenames[i];
      let stat = fs.statSync(filePath);
      let fileModel: FileModel = new FileModel();
      fileModel.fileName = filenames[i];
      fileModel.fileSize = stat.size;
      fileModel.ctime = stat.ctime;
      fileModel.showTime = FileUtil.transferTimeToStr(fileModel.ctime);
      fileModel.showSize = FileUtil.transferSizeToStr(fileModel.fileSize);
      fileModel.path = filePath;
      fileList.push(fileModel);
      // 每多少个发送一批，防止子线程OOM
      if (i !== 0 && (i % BATCH_SEND_NUM === 0)) {
        taskpool.Task.sendData(fileList, QueryMsg.SEND_DATA);
        fileList = [];
      }
      if (taskpool.Task.isCanceled()) {
        taskpool.Task.sendData([], QueryMsg.ERROR);   // 主动退出
        return;
      }
    }
    taskpool.Task.sendData(fileList, QueryMsg.END);   // the tail, under BATCH_SEND_NUM
  } catch (error) {
    Logger.error(TAG, 'query error: ' + error.toString());
    taskpool.Task.sendData([], QueryMsg.ERROR);
  }
}
```

**Three properties make this function safe to run on a worker.** It is
top-level and `@Concurrent`, so it captures nothing from the page - `path`
arrives as a serialised argument through `new taskpool.Task(startTask, path)`.
Every exit is a `sendData`: the empty-path guard, the cancel branch, the
`catch`, and the normal completion. And the batch flush *resets* `fileList`
after sending, so each message carries only the new 1000 - forgetting that line
would resend the entire accumulated list every batch, which on 100,000 files is
quadratic serialisation.

`taskpool.Task.isCanceled()` is checked once per file, which is the right
granularity here because `statSync` dominates the loop. Note it is checked
*after* the batch flush, so a cancel arriving mid-batch still delivers the
partial batch before the `ERROR` - harmless, since `ERROR` clears the list
anyway.

`fs.listFile(path)` returns names only, so each entry costs a separate blocking
`fs.statSync` - which is the reason this work belongs off the UI thread at all.

**The receive callback and the list — `entry/src/main/ets/pages/QueryFilePage.ets`** (corrected, see `HW-15-0039`)

```typescript
List() {
  LazyForEach(this.data, (item: FileModel) => {
    ListItem() {
      Column() {
        Row() {
          Image($r('app.media.doc')).width($r('app.float.48_vp')).height($r('app.float.48_vp'));
          Column() {
            Text(item.fileName).fontSize($r('app.float.font_16_fp'));
            Text(item.showTime + '-' + item.showSize).fontSize($r('app.float.font_14_fp'));
          }
          .alignItems(HorizontalAlign.Start);
        };
        Divider().margin({ left: $r('app.float.66_vp') });
      };
    };
  }, (item: FileModel) => item.path);          // FIX: sample has `(item: string) => item`
}.cachedCount(5);

queryFile(path: string = '') {
  if (this.queryTask) {
    TaskExecutor.getInstance().cancelTask(this.queryTask);
    this.queryTask = undefined;
  }
  this.queryTask = new QueryFileTask(path, (fileList: FileModel[], msg: QueryMsg) => {
    switch (msg) {
      case QueryMsg.START:
        this.data.dataArray = [];
        break;
      case QueryMsg.SEND_DATA:
        this.data.dataArray.push(...fileList);
        this.data.notifyDataReload();
        break;
      case QueryMsg.ERROR:
        this.data.dataArray = [];
        this.data.notifyDataReload();
        break;
      case QueryMsg.END:
        this.data.dataArray.push(...fileList);
        this.data.notifyDataReload();
        break;
    }
  });
  TaskExecutor.getInstance().execute<void>(this.queryTask);
}

aboutToDisappear(): void {
  TaskExecutor.getInstance().cancelTask(this.queryTask);   // 退出页面，取消查询任务
}
```

**The key generator is the correction, and it matters most in exactly this
sample.** The shipped form declares `(item: string) => item` over a
`FileModel[]`; the parameter type is a lie the compiler accepts, and the
returned "key" is the object coerced to a string, so every row in the list
keys as `[object Object]`. The reference requires keys to be unique per data
item and stable across updates; identical keys defeat `LazyForEach`'s diffing
and item reuse, and this list is designed for 100,000 rows arriving in 100
batches. `item.path` is already carried on the model and is unique by
construction (`HW-15-0039`).

**`notifyDataReload` after every batch is the deliberate choice here.** The
alternative, `notifyDataAdd` per item, would be cheaper per row but is wrong
against a source that was just bulk-mutated; reload lets the framework re-key
the whole set once per 1000 rows, and `cachedCount(5)` keeps the off-screen
buffer small. Note that `START` and `ERROR` assign a fresh `[]` to `dataArray`
rather than replacing the `FileDataSource`, so its registered listeners stay
attached - swapping the source instead would silently detach the list.

**The size formatter — `entry/src/main/ets/utils/FileUtil.ets`** (corrected, see `HW-15-0040`)

```typescript
// 转换大小
static transferSizeToStr(size: number): string {
  const UNITS: string[] = ['B', 'KB', 'MB', 'GB', 'TB'];   // FIX: sample starts at 'KB'
  let count = 0;
  let unit: string = UNITS[count];
  let res = size;
  let resTmp = size / 1024;

  while (resTmp >= 1 && count < UNITS.length - 1) {         // FIX: sample hardcodes `count < 3`
    count++;
    res = Math.floor(resTmp * 10) / 10;
    unit = UNITS[count];
    resTmp = resTmp / 1024;
  }
  return res + unit;
}
```

**The input is `stat.size`, which is bytes** (`QueryFileTask.ets:62` passes
`fileModel.fileSize = stat.size` straight in), but the shipped unit table
starts at `'KB'`. Every size on screen is therefore labelled one unit too
large: a 500-byte file reads `500KB`, and a 1 MB file reads `1GB`. Prepending
`'B'` fixes the whole table with one element, and deriving the loop bound from
`UNITS.length` keeps the guard correct when it changes.

The rounding is `Math.floor(resTmp * 10) / 10`, one decimal place, applied to
the *pre-increment* value - which is why `res` and `unit` are assigned together
inside the loop rather than computed after it.

**The dialog — `entry/src/main/ets/pages/CustomDialog.ets`** (corrected, see `HW-15-0041`)

```typescript
Text($r('app.string.create_file'))
  .fontSize(Constants.FONT_SIZE_TITLE)
  .fontWeight(Constants.FONT_WEIGHT_TITLE)      // 800
  .lineHeight(Constants.LINE_HEIGHT_TITLE)      // FIX: sample calls .fontWeight(27) here
  .fontColor(Color.Black);
Text($r('app.string.create_file_dialog_msg'))
  .fontSize(Constants.FONT_SIZE_SUBTITLE)
  .fontWeight(Constants.FONT_WEIGHT_SUBTITLE)   // 700
  .lineHeight(Constants.LINE_HEIGHT_NORMAL)     // FIX: sample calls .fontWeight(21) here
  .margin({ top: 16 });
```

**Both `Text`s chain `.fontWeight` twice, and the second call wins.** The
second argument is a line-height constant (27 and 21), so the intended 800 and
700 weights are replaced by a numeric weight of 27 - effectively the lightest
possible - and the line heights are never applied at all. The constants are
correctly named in `Consts.ets`; only the method name is wrong. This is the
kind of defect no type checker catches, because `fontWeight` legitimately
accepts a number.

## Permissions & config

**None.** The sample declares no `requestPermissions`. Everything happens
inside the application sandbox: `context.filesDir` for both the create and the
query, so no `ohos.permission.READ_MEDIA` or file-picker flow is involved.
Pointing the same code at a user-selected directory would change that.

`module.json5` lists `deviceTypes: ["phone", "tablet", "2in1"]`, registers
`$profile:router_map` alongside `$profile:main_pages`, and adds the standard
`EntryBackupAbility`. `main_pages.json` contains only `pages/HomePage`;
`QueryFilePage` is reachable solely through its router-map entry, which names
`queryFilePageBuilder` as the build function. That is the right registration
for a `Navigation`/`NavDestination` pair - the route is resolved by name at
push time, so `HomePage` has no import edge to the page it opens.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **`taskpool.cancel` does not stop a running task.** If the task is queued it
  is dropped and the promise rejects; if it has already been dispatched to a
  worker it keeps running, and only `taskpool.Task.isCanceled()` inside the
  function can end it. From API 20 the catch branch can carry a
  `BusinessError<taskpool.TaskResult>` with the final state.
- A `@Concurrent` function may not capture anything from its defining scope;
  all inputs must be serialisable arguments, and all results must come back
  through `sendData` or the resolved value.
- The create task writes **100,000 files** into `filesDir` in a hardcoded loop
  (`FileUtil.createFile`), with no progress reporting and no way to stop it -
  `CreateFileTask` never checks `isCanceled()`. On a real device this takes a
  long time and is not undone by the sample.
- `fs.listFile` is not recursive here and directories are listed as ordinary
  rows; `FileModel.isDir` is populated by the worker and never used in the UI.
- The query page starts its scan from `aboutToAppear` with no pull-to-refresh
  and no re-query entry point.

## Pitfalls

- **`HW-15-0038` — `taskpool.execute` has no `.catch`.** (B/medium, confirmed)
  `TaskExecutor.ets:45-53` attaches only `.then`. The documented page-exit
  cancel (`QueryFilePage.aboutToDisappear` → `cancelTask` → `taskpool.cancel`)
  rejects that promise, so every mid-query exit is an unhandled rejection; and
  because only the resolve path clears `HomePage.task` (`HomePage.ets:86-89`),
  a single failed create task blocks the button behind the 创建中 toast
  permanently. **Fix:** add a `.catch` that runs the same cleanup as the
  callback.
- **`HW-15-0039` — systematic: `(item: string) => item` key generators over
  object arrays.** (B/medium, confirmed) `QueryFilePage.ets:56,83` - a
  `LazyForEach` over `FileModel` designed for 100,000 rows whose every key is
  `[object Object]`; the same defect appears in `LogTemp`
  (`PrintDocxPage.ets:623`, `ForEach` over `PrintFileModel`, where duplicate
  keys break the per-item delete). Two samples in this industry.
  **Fix:** return `item.path`, or another genuinely unique id.
- **`HW-15-0040` — file sizes are displayed one unit too big.** (B/medium,
  confirmed) `FileUtil.ets:54-68` starts `UNITS` at `'KB'` while the input is
  `stat.size` in bytes (`QueryFileTask.ets:62`), so every size is inflated
  1024x - a 1 MB file shows as `1GB`. **Fix:** prepend `'B'` to `UNITS`.
- **`HW-15-0041` — dialog texts chain `.fontWeight` twice.** (B/low,
  confirmed) `CustomDialog.ets:31-38` calls `.fontWeight(FONT_WEIGHT_TITLE)`
  then `.fontWeight(LINE_HEIGHT_TITLE = 27)`; the mistyped second call
  overrides the intended 800/700 weights and no line height is ever set.
  **Fix:** rename the second call to `.lineHeight`.
- **`BaseTask.isCancel` is written and never read.** Cancellation actually
  works through `taskpool.cancel` plus the worker's `isCanceled()` check; the
  host-side boolean cannot cross the thread boundary and is dead weight.
- **`CreateFileTask` ignores cancellation entirely** - its concurrent function
  is a bare 100,000-iteration loop with no `isCanceled()` check, so
  `cancelTask` on it does nothing once it is running.
- **The receive callback can outlive its page.** Nothing clears
  `queryTask`'s callback in `aboutToDisappear`, so a late `ERROR` message
  mutates the data source of a component that is already gone.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-taskpool.md` - `Task`, `execute`, `cancel` and its rejection semantics, `sendData`, `onReceiveData`, `isCanceled`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-taskpool
- `documentation/harmonyos-guides/03_application-framework/taskpool-introduction.md` - when TaskPool is the right concurrency model and the `@Concurrent` capture rules
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/taskpool-introduction
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-lazyforeach.md` - `IDataSource`, the `keyGenerator` uniqueness and consistency requirements, and the `notifyDataReload` contract
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-lazyforeach
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `listFile`, `statSync` and the `Stat.size` / `Stat.ctime` units
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `huawei_industry_tree/15_utilities/docs/16_taskpool_query.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/taskpool_query-0000002292913090
- `UTIL-01` (`LogTemp`) - the other instance of the `HW-15-0039` key-generator systematic
