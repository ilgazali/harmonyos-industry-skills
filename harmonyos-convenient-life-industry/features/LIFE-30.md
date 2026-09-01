---
id: LIFE-30
title: Batch thumbnail generation off the UI thread - a bounded taskpool executor feeding a LazyForEach list
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/30_thumbnail_taskpool.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/thumbnail_taskpool-0000002338966908
sample: huawei_industry_tree/02_convenient_life/downloads/ThumbnailTaskpool.zip
kits: ["@kit.ArkTS", "@kit.ImageKit", "@kit.CoreFileKit", "@kit.MediaLibraryKit", "@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: ["taskpool.Task", "taskpool.execute", "taskpool.cancel", "taskpool.Priority", "taskpool.Task.sendData", "taskpool.Task.isCanceled", "task.onReceiveData", Concurrent, LazyForEach, IDataSource, DataChangeListener, "notifyDataAdd", "notifyDataReload", "notifyDatasetChange", "List.cachedCount", "photoAccessHelper.PhotoViewPicker", "photoAccessHelper.PhotoSelectOptions", "photoAccessHelper.PhotoViewMIMETypes", "image.createImageSource", "imageSource.createPixelMap", "pixelmap.getImageInfo", "pixelmap.scale", "image.createImagePacker", "imagePacker.packToData", "fileIo.openSync", "fileIo.copyFileSync", "fileIo.closeSync", "fileIo.listFile", "fileIo.statSync", "context.filesDir", HashMap, StorageProp]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-02-0254, HW-02-0255, HW-02-0256, HW-02-0257, HW-02-0258, HW-02-0259, HW-02-0260, HW-02-0261, HW-02-0262, HW-02-0263, HW-02-0264, HW-02-0265, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card for **many expensive, independent jobs that must not touch the
UI thread**: thumbnail generation, file imports, bulk decoding, checksums. The
feature is a file list whose rows each need a thumbnail; the transferable part
is the small executor layered on top of `taskpool`.

`taskpool` on its own has no concurrency ceiling and no queue you control -
`taskpool.execute` simply submits. This sample wraps it:

```
TaskManager  (singleton)  -> HashMap<poolName, TaskExecutor>
   execute(task, maxRunningTask, maxWaitLimit)
        TaskExecutor      -> waitList (LIFO)  +  runningList (capped)
             taskpool.execute(task.task, task.priority)
                  @Concurrent function on a worker thread
                       -> result via the promise, or
                       -> progress via taskpool.Task.sendData / onReceiveData
```

Three things make the shape worth copying:

- **A pool per job kind.** `BaseTask` carries a `taskPoolName`, and the manager
  keeps one executor per name - so thumbnails and file queries get independent
  caps and independent cancellation.
- **Cancellation is cooperative.** `BaseTask.cancelTask()` sets `isCancel` and
  calls `taskpool.cancel`; the worker checks `taskpool.Task.isCanceled()` inside
  its loop.
- **Streaming, not one big result.** The query task calls
  `taskpool.Task.sendData(batch, msg)` every thousand files, and the page
  receives them through `task.onReceiveData(callback)`.

The bookkeeping in the shipped executor is broken in two specific ways
(pitfalls 1 and 2) - fix those and the layer is sound.

**No permissions.** `PhotoViewPicker` is a system picker that returns URIs the
application is granted access to, so `module.json5` has no `requestPermissions`
block at all.

## Feature checklist

- [ ] Every `taskpool.execute` given a `.catch`, and the running-list cleanup
      moved somewhere both outcomes reach (HW-02-0254).
- [ ] The wait-list overflow dropping the **oldest** entries
      (HW-02-0255).
- [ ] `LazyForEach` keyed on a unique string from the item (HW-02-0256).
- [ ] Every `ImageSource` and intermediate `PixelMap` released
      (HW-02-0257).
- [ ] One scale factor for both axes (HW-02-0258).
- [ ] One entry point on the manager, or names that say the difference
      (HW-02-0259).
- [ ] Batch appends published with `pushData` / `notifyDataAdd`, not a full
      reload (HW-02-0261).
- [ ] The import reporting what actually happened (HW-02-0260).

## Architecture

| File | Role |
| --- | --- |
| `taskpool/base/BaseTask.ets` | The task envelope: the `taskpool.Task`, its callback, priority, pool name and cancel flag. |
| `taskpool/base/TaskExecutor.ets` | One pool: wait list, running list, concurrency cap, overflow rule. |
| `taskpool/base/TaskManager.ets` | Singleton. Resolves an executor by pool name; also exposes an unmanaged path. |
| `taskpool/QuerySandBoxFile.ets` | Lists the sandbox directory and streams `FileModel` batches back. |
| `taskpool/GenerateThumbnailTask.ets` | Decodes, scales and repacks one image into a thumbnail. |
| `taskpool/ImportFileTask.ets` | Copies picked gallery URIs into `filesDir`. |
| `model/FileDataSource.ets` | A clean `IDataSource` implementation for `LazyForEach`. |
| `pages/ThumbnailPage.ets` | The picker entry point. |
| `pages/QueryFilePage.ets` | The list, and the query task's lifecycle. |
| `components/FileItem.ets` | One row; requests its own thumbnail on appear. |

The division of labour between the three layers is the part to internalise:

```ts
// 1. the envelope - subclass per job, build the taskpool.Task in the constructor
export class GenerateThumbnailTask extends BaseTask {
  constructor(fileModel: FileModel, callback: Function) {
    super('GenerateThumbnailTask');            // the pool name
    this.task = new taskpool.Task(startTask, fileModel);
    this.callback = callback;
  }
}

// 2. the worker - a top-level @Concurrent function, arguments passed by value
@Concurrent
async function startTask(fileModel: FileModel) { /* ... */ return fileModel; }

// 3. the submission - resolved to a named pool with a cap
TaskManager.getInstance().execute(task, 3, 1000);
```

Thumbnails are requested **per row, on demand** - `FileItem.aboutToAppear`
checks whether the model already carries one and only queues a task if not, then
writes the result back onto the shared `FileModel` so scrolling away and back
does not regenerate it. That, plus `List.cachedCount(5)`, is what keeps the
work bounded as the user scrolls.

## Implementation steps

Where the shipped code is wrong, the step gives the corrected version and names
the finding.

1. **Define the task envelope.** `BaseTask` is what lets the executor treat all
   jobs alike:

   ```ts
   export class BaseTask {
     task?: taskpool.Task;
     callback?: Function = () => {};
     priority: taskpool.Priority = taskpool.Priority.MEDIUM;
     taskPoolName: string = '';
     isCancel: boolean = false;

     constructor(taskPoolName: string) { this.taskPoolName = taskPoolName; }

     cancelTask(): void {
       try {
         this.isCancel = true;
         if (this.task) { taskpool.cancel(this.task); }
       } catch (error) {
         Logger.error(TAG, 'cancel task error: ' + error.toString());
       }
     }
   }
   ```

   `taskpool.cancel` can throw, so the try is not optional.

2. **Write the worker as a top-level `@Concurrent` function.** It must be
   top-level, not a method - the reference reports error 10200014, "The function
   is not marked as concurrent", when this is wrong, and 10200006 when an
   argument cannot be serialised:

   ```ts
   @Concurrent
   async function startTask(fileModel: FileModel) { /* ... */ return fileModel; }
   ```

3. **Bound the pool.** The executor's admission test is the running list; the
   overflow rule is the wait list:

   ```ts
   private canExecute(): boolean {
     return this.runningList.length < this.maxTaskRunningNum;
   }

   private getTask(): BaseTask | undefined {
     return this.waitList.pop();          // last in, first out
   }

   private addTask(task: BaseTask): void {
     this.waitList.push(task);
     if (this.maxWaitLimit && this.waitList.length >= this.maxWaitLimit) {
       let startIndex = this.waitList.length - this.maxWaitLimit;
       if (startIndex < 0) { startIndex = 0; }
       this.waitList = this.waitList.slice(startIndex);   // shipped: slice(startIndex, this.maxWaitLimit)
     }
   }
   ```

   LIFO plus drop-the-oldest is the right policy for thumbnails: what the user
   is looking at now was queued most recently. The shipped second argument to
   `slice` inverts it (HW-02-0255).

4. **Submit, and free the slot on both outcomes** (HW-02-0254). This is the
   single most important correction in the card:

   ```ts
   private executeTask(task: BaseTask): void {
     this.runningList.push(task);
     taskpool.execute(task.task, task.priority)
       .then((result: object) => {
         try {
           if (task.callback) { task.callback(result); }
         } catch (error) {
           Logger.info(TAG, 'executeTask callback error: ' + error.toString());
         }
       })
       .catch((err: BusinessError) => {                    // shipped: absent
         Logger.error(TAG, `task failed. Code: ${err.code}, message: ${err.message}`);
       })
       .finally(() => {                                    // shipped: inside then only
         this.removeTaskFromRunningList(task);
         this.startTask();
       });
   }
   ```

   The reference is explicit that cancellation lands here too: "If the task is
   in the internal queue of the task pool, the task will not be executed after
   being canceled, and an exception indicating task cancellation is returned. If
   the task has been distributed to the worker thread of the task pool,
   canceling the task does not affect the task execution, and the execution
   result is returned in the catch branch."

5. **Stream long results instead of returning them.** The query worker sends a
   batch every thousand files and checks for cancellation each iteration:

   ```ts
   if (i !== 0 && (i % BATCH_SNED_NUM === 0)) {
     taskpool.Task.sendData(fileList, QueryMsg.SEND_DATA);
     fileList = [];
   }
   if (taskpool.Task.isCanceled()) {
     taskpool.Task.sendData([], QueryMsg.ERROR);
     return;
   }
   ```

   The receiving side registers through the envelope, not through the promise:

   ```ts
   export class QuerySandBoxFile extends BaseTask {
     constructor(path: string, callback: Function) {
       super('QueryFileTask');
       this.task = new taskpool.Task(startTask, path);
       this.onReceiveData(callback);        // subscribes task.onReceiveData
     }
   }
   ```

6. **Publish batches incrementally** (HW-02-0261). The data source already has
   the right method; use it instead of reloading everything:

   ```ts
   case QueryMsg.SEND_DATA:
     fileList.forEach((f) => this.data.pushData(f));   // pushData -> notifyDataAdd
     break;
   ```

7. **Key the list on something unique and stable** (HW-02-0256):

   ```ts
   LazyForEach(this.data, (item: FileModel, index: number) => {
     ListItem() {
       FileItem({ item: item, isShowDivider: index !== (this.data.totalCount() - 1) });
     }
   }, (item: FileModel) => item.path)      // shipped: (item: string) => item
   ```

   `FileModel.path` is assigned per file by the query worker and is unique.

8. **Request a thumbnail only when the row lacks one.**

   ```ts
   aboutToAppear(): void {
     if (!this.item) { return; }
     this.thumbnail = this.item.thumbnail;
     if (this.thumbnail === undefined) {
       let task = new GenerateThumbnailTask(this.item, (result: FileModel | undefined) => {
         if (result === undefined) { return; }
         if (this.item) { this.item.thumbnail = result.thumbnail; }   // cache on the shared model
         this.thumbnail = result.thumbnail;                            // and on the row's own state
       });
       TaskManager.getInstance().execute(task, 3, 1000);
     }
   }
   ```

   Writing the result back onto the `FileModel` is what stops a re-scroll from
   re-queuing the same work.

9. **Generate the thumbnail, releasing as you go** (HW-02-0257,
   HW-02-0258):

   ```ts
   @Concurrent
   async function startTask(fileModel: FileModel) {
     const imageSource = image.createImageSource(fileModel.path);
     let pixelmap: image.PixelMap | undefined = undefined;
     let packed: image.ImageSource | undefined = undefined;
     try {
       pixelmap = await imageSource.createPixelMap({ editable: true, desiredPixelFormat: 3 });
       let info = await pixelmap.getImageInfo();
       let scale = Math.min(500 / info.size.width, 500 / info.size.height);  // one factor, both axes
       await pixelmap.scale(scale, scale);

       const packer = image.createImagePacker();
       let data: ArrayBuffer = await packer.packToData(pixelmap, { format: 'image/jpeg', quality: 90 });
       packed = image.createImageSource(data.slice(0));
       fileModel.thumbnail = await packed.createPixelMap({ editable: true, desiredPixelFormat: 3 });
     } catch (e) {
       Logger.error(TAG, 'generate thumbnail error: ' + e.toString());
     } finally {
       pixelmap?.release();       // shipped: never released
       packed?.release();
       imageSource.release();
     }
     return fileModel;
   }
   ```

   The kept `fileModel.thumbnail` is displayed through an `Image` component, so
   it does **not** need a manual release - the image decoding guide states the
   `Image` component manages a PixelMap passed to it. Everything else here does.

10. **Import picked files on a worker.**

    ```ts
    let photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
    photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
    photoSelectOptions.maxSelectNumber = 50;
    let photoPicker = new photoAccessHelper.PhotoViewPicker();
    photoPicker.select(photoSelectOptions).then((photoSelectResult) => {
      let task = new ImportFileTask(this.getUIContext().getHostContext()?.filesDir || '',
        photoSelectResult.photoUris, (copied: number, failed: number) => {
          this.getUIContext().getPromptAction().showToast({
            message: failed === 0 ? $r('app.string.import_success') : $r('app.string.import_partial')
          });
        });
      TaskManager.getInstance().execute(task);      // shipped: executeTask, which bypasses the pool
    }).catch((err: BusinessError) => { Logger.error('PhotoViewPicker.select failed: ' + JSON.stringify(err)); });
    ```

    Report what happened - the shipped callback takes no parameter and always
    shows success (HW-02-0260).

11. **Cancel on the way out.** The page cancels its own query task and the whole
    thumbnail pool:

    ```ts
    aboutToDisappear(): void {
      TaskManager.getInstance().cancelTask(this.queryTask);
      TaskManager.getInstance().cancelExecutorTask('GenerateThumbnailTask');
    }
    ```

    Remove the executor from the static map while you are there
    (HW-02-0262).

## Verified snippets

All snippets below are copied from the ZIP, not from the document.

`ThumbnailTaskpool.zip#entry/src/main/ets/taskpool/base/TaskExecutor.ets:78-109` -
the admission loop and the submission, which is the core of the layer:

```ts
  private startTask(): void {
    while (!this.isListEmpty(this.waitList) && this.canExecute()) {
      let task = this.getTask();
      if (task) {
        this.executeTask(task);
      }
    }
  }

  /**
   * 执行具体任务
   * @param task 任务
   */
  private executeTask(task: BaseTask): void {
    if (!task || !task.task) {
      Logger.error(TAG, 'executeTask task is null.');
      return;
    }
    this.runningList.push(task);
    Logger.info(TAG, 'executeTask is really start, task.task name = ' + task.task.name);
    taskpool.execute(task.task, task.priority).then((result: object) => {
      try {
        if (task.callback) {
          task.callback(result);
        }
      } catch (error) {
        Logger.info(TAG, 'executeTask callback error: ' + error.toString());
      }
      this.removeTaskFromRunningList(task);
      this.startTask();
    });
  }
```

The missing `.catch`, and the fact that the removal and the restart live only in
the success path, is HW-02-0254.

`ThumbnailTaskpool.zip#entry/src/main/ets/taskpool/base/TaskExecutor.ets:133-155` -
the overflow rule and the admission test:

```ts
  private addTask(task: BaseTask): void {
    if (!task || !task.task) {
      return;
    }

    // 后插入的先执行,如果达到最大数就将前面的舍弃
    this.waitList.push(task);
    if (this.maxWaitLimit && (this.waitList.length) >= this.maxWaitLimit) {
      let startIndex = this.waitList.length - this.maxWaitLimit;
      if (startIndex < 0) {
        startIndex = 0;
      }
      this.waitList = this.waitList.slice(startIndex, this.maxWaitLimit);
    }
  }

  /**
   * 当前是否能执行
   * @returns
   */
  private canExecute(): boolean {
    return this.runningList.length < this.maxTaskRunningNum;
  }
```

The `slice` on the eleventh line is HW-02-0255 - the second argument is an
end index, so it drops the task pushed five lines above.

`ThumbnailTaskpool.zip#entry/src/main/ets/taskpool/base/TaskManager.ets:80-95` -
resolving one executor per pool name:

```ts
  execute(task: BaseTask, maxRunningTask: number = TaskManager.maxRunningTask, maxWaitLimit?: number): void {
    if (!task || !task.task) {
      Logger.error(TAG, 'execute task is null');
      return;
    }
    let taskName = task.taskPoolName;
    if (taskName === undefined || taskName === '') {
      taskName = TaskManager.DEFAULT_TASKPOOL;
    }
    let executor = TaskManager.executorMap.get(taskName);
    if (!executor) {
      executor = new TaskExecutor(taskName, maxRunningTask, maxWaitLimit);
      TaskManager.executorMap.set(taskName, executor);
    }
    executor.execute(task);
  }
```

Note that the caps are captured when the executor is first created - a later
call with different numbers reuses the original (HW-02-0262).

`ThumbnailTaskpool.zip#entry/src/main/ets/taskpool/QuerySandBoxFile.ets:49-77` -
batched streaming with cooperative cancellation:

```ts
  try {
    let filenames = await fs.listFile(path);
    // 分批发送文件数量
    const BATCH_SNED_NUM = 1000;
    for (let i = 0; i < filenames.length; i++) {
      let filePath = path + '/' + filenames[i];
      let stat = fs.statSync(filePath);
      let fileModel: FileModel = new FileModel();
      fileModel.fileName = filenames[i];
      fileModel.isDir = stat.isDirectory();
      fileModel.fileSize = stat.size;
      fileModel.ctime = stat.ctime;
      fileModel.mtime = stat.mtime;
      fileModel.showTime = FileUtil.transferTimeToStr(fileModel.ctime);
      fileModel.showSize = FileUtil.transferSizeToStr(fileModel.fileSize);
      fileModel.path = filePath;
      fileList.push(fileModel);
      // 每多少个发送一批，防止子线程OOM
      if (i !== 0 && (i % BATCH_SNED_NUM === 0)) {
        taskpool.Task.sendData(fileList, QueryMsg.SEND_DATA);
        fileList = [];
      }
      if (taskpool.Task.isCanceled()) {
        taskpool.Task.sendData([], QueryMsg.ERROR);
        return;
      }
    }
    // 发送结束
    taskpool.Task.sendData(fileList, QueryMsg.END);
```

`ThumbnailTaskpool.zip#entry/src/main/ets/model/FileDataSource.ets:29-36` - the
incremental append the page never calls:

```ts
  public getData(index: number): FileModel {
    return this.dataArray[index];
  }

  public pushData(data: FileModel): void {
    this.dataArray.push(data);
    this.notifyDataAdd(this.dataArray.length - 1);
  }
```

`ThumbnailTaskpool.zip#entry/src/main/ets/components/FileItem.ets:31-49` - the
on-demand thumbnail request with its result cached back onto the model:

```ts
  aboutToAppear(): void {
    if (!this.item) {
      return;
    }
    this.thumbnail = this.item.thumbnail;
    this.getShowInfo(this.item?.showTime, this.item?.showSize);
    if (this.thumbnail === undefined) {
      let task = new GenerateThumbnailTask(this.item, (result: FileModel | undefined) => {
        if (result === undefined) {
          return;
        }
        if (this.item) {
          this.item.thumbnail = result.thumbnail;
        }
        this.thumbnail = result.thumbnail;
      });
      TaskManager.getInstance().execute(task, 3, 1000);
    }
  }
```

`ThumbnailTaskpool.zip#entry/src/main/ets/taskpool/ImportFileTask.ets:43-62` -
the copy loop, with its file descriptors correctly closed in a finally:

```ts
  for (let i = 0; i < uris.length; i++) {
    let nameIndex = uris[i].lastIndexOf('/');
    let name = uris[i].substring(nameIndex, uris[i].length);
    let srcFile: fs.File | undefined = undefined;
    let destFile: fs.File | undefined = undefined;
    try {
      srcFile = fs.openSync(uris[i], fs.OpenMode.READ_ONLY);
      destFile = fs.openSync(path + name, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
      fs.copyFileSync(srcFile.fd, destFile.fd);
    } catch (error) {
      Logger.error(TAG, 'copy fail: ' + error.toString());
    } finally {
      if (srcFile) {
        fs.closeSync(srcFile);
      }
      if (destFile) {
        fs.closeSync(destFile);
      }
    }
  }
```

The descriptor handling is exactly right. What is missing is any record of how
many copies failed (HW-02-0260).

## Permissions & config

**No permissions.**
`ThumbnailTaskpool.zip#entry/src/main/module.json5` has no `requestPermissions`
block. `PhotoViewPicker` is a system picker: the user selects, and the
application receives URIs it is granted access to. Everything else the sample
touches is inside its own sandbox, reached through
`getHostContext()?.filesDir`.

Build target:

```json5
"targetSdkVersion": "6.0.0(20)",
"compatibleSdkVersion": "6.0.0(20)",
```

`oh-package.json5` has no runtime dependencies - `taskpool` comes from
`@kit.ArkTS`.

## Constraints

- **API 20 / HarmonyOS 6.0.0 Release and later**, DevEco Studio 6.0.0 Release
  and later.
- **A task function must be top-level and `@Concurrent`.** The reference lists
  error 10200014, "The function is not marked as concurrent", among
  `execute`'s error codes.
- **Arguments are serialised.** Error 10200006, "An exception occurred during
  serialization", is the failure mode for anything that cannot cross the thread
  boundary - which is why the workers take plain models and strings.
- **Cancellation reports through the catch branch.** Queued tasks return "an
  exception indicating task cancellation"; a task already dispatched to a
  worker keeps running and "the execution result is returned in the catch
  branch". A `taskpool.execute` chain with no `.catch` therefore cannot observe
  either.
- **`taskpool.cancel` can throw** - the reference documents error 10200018 for
  the group form - so it belongs in a try.
- **`taskpool` has no built-in concurrency ceiling**; the cap in this sample is
  entirely `TaskExecutor.canExecute`.
- **`LazyForEach` keys must be unique**: "Duplicate key values will cause
  rendering issues."
- **`notifyDataAdd` creates one child**, `notifyDataReload` "instruct[s]
  LazyForEach to refresh all child components" - choose per operation.
- **`PhotoSelectOptions.maxSelectNumber`** bounds the import; this sample uses
  50.
- **A PixelMap handed to an `Image` component is managed by it**; every other
  `PixelMap` and every `ImageSource` is the application's to release.

## Pitfalls

1. **HW-02-0254 - a failed task permanently consumes a concurrency slot.**
   `TaskExecutor.ets:98-108` runs `taskpool.execute(...).then(...)` with no
   `.catch`, and that `then` is the only place `removeTaskFromRunningList` and
   `startTask` are called. Admission is `runningList.length < maxTaskRunningNum`
   (`:153-155`). The thumbnail pool is created with a cap of 3
   (`FileItem.ets:47`), so three rejected or cancelled thumbnails are enough to
   make `canExecute` false forever and stop the pool for the life of the
   process. The reference confirms cancellation reports through the catch
   branch, so the page's own `cancelExecutorTask` on exit is one of the ways in.
   `TaskManager.ets:54-62` has the same uncaught shape.

2. **HW-02-0255 - the overflow rule discards the newest task.**
   `TaskExecutor.ets:145` is
   `this.waitList = this.waitList.slice(startIndex, this.maxWaitLimit);` -
   `slice`'s second argument is an end index, not a count. With the limit at
   1000 and 1001 entries, it returns indexes 1 to 999 and drops index 1000,
   which is the task pushed seven lines earlier. Since `getTask` pops from the
   end (`:126`), that is precisely the task that would have run next. The
   comment above it says the opposite - "后插入的先执行,如果达到最大数就将前面的舍弃"
   ("the later inserted runs first; when the maximum is reached, discard the
   earlier ones"). Drop the second argument.

3. **HW-02-0265 - the document promises the policy the code inverts.** Step 3
   states '后插入的先执行，如果达到最大数就将前面的舍弃，缓冲任务请求，避免任务丢失'
   ("the later inserted runs first; when the maximum is reached the earlier ones
   are discarded, buffering task requests so that tasks are not lost"). That is
   the correct design, and it is the sentence a reader trusts while copying an
   executor that loses the newest task instead.

4. **HW-02-0256 - the LazyForEach key generator is typed as a string and handed
   a model.** `QueryFilePage.ets:54` iterates `FileModel` items and `:62` closes
   with `}, (item: string) => item);`. The guidance requires unique persistent
   keys and warns that duplicates "will cause rendering issues". `FileModel.path`
   is right there and is unique per row.

5. **HW-02-0257 - the thumbnail worker releases nothing.**
   `GenerateThumbnailTask.ets` creates an `ImageSource` (`:36`), a full-size
   `PixelMap` (`:42`), an `ImagePacker` (`:49`) and a second `ImageSource`
   (`:52`) per image, and the string `release` appears nowhere in the ZIP. The
   image decoding guide says to release the `ImageSource` once `createPixelMap`
   has returned, and to release a `PixelMap` the application handles itself. The
   intermediate full-resolution bitmap is provably dead after `packToData`.

6. **HW-02-0258 - every thumbnail is stretched to a square.**
   `GenerateThumbnailTask.ets:45-47` computes `scaleX = 500 / width` and
   `scaleY = 500 / height` and applies both, which resizes any input to exactly
   500 by 500. The distortion is baked into the packed thumbnail, and
   `FileItem.ets:66-70` renders it in a 48vp square with no `objectFit`, so
   nothing downstream can undo it. Use one factor for both axes.

7. **HW-02-0259 - the manager has two entry points and the document uses the
   unmanaged one.** `TaskManager.executeTask` (`:50-63`) calls
   `taskpool.execute` directly - no executor, no cap, no queue - while
   `TaskManager.execute` (`:80-95`) is the managed path. The import flow uses
   the former (`ThumbnailPage.ets:127`, reproduced at the document's step 1) for
   a batch of up to fifty file copies; the thumbnail flow uses the latter. The
   names differ by a suffix and the signatures give no hint.

8. **HW-02-0260 - the import always says it succeeded.**
   `ImportFileTask.ets:52-53` catches each per-file failure and only logs it,
   the worker returns nothing, and `ThumbnailPage.ets:124-126` passes a
   zero-argument callback that shows `import_success` unconditionally. Fifty
   failed copies produce the same toast as fifty successful ones.

9. **HW-02-0261 - batch appends reload the whole list.**
   `QueryFilePage.ets:92-93` and `:102-103` do
   `dataArray.push(...fileList); notifyDataReload();`, while
   `FileDataSource.pushData` (`:33-36`) already pairs the append with
   `notifyDataAdd` exactly as the guidance describes. The worker sends batches
   of a thousand while the list is on screen, so every batch tells LazyForEach
   to refresh every child rather than to create the new ones.

10. **HW-02-0262 - the executor map is static and never pruned.**
    `TaskManager.ets:40` holds it, `:89-93` inserts, and `cancelExecutorTask`
    (`:98-106`) cancels the tasks but leaves the entry. Because `:90` only
    constructs an executor when the lookup misses, the caps captured on first
    use stick for the process lifetime - a later `execute` with different
    numbers silently reuses the old executor.

11. **HW-02-0263 - the logger format and its call sites disagree.**
    `Logger.ets:23` declares `'%{public}s, %{public}s'` and all four methods pass
    `args` as one array. The call sites are inconsistent too:
    `ImportFileTask.ets:53` passes a tag and a message,
    `GenerateThumbnailTask.ets:59` passes only a message - so the one line the
    thumbnail worker emits on failure carries no tag.

12. **HW-02-0264 - `onReceiveData`'s catch says "cancel task error".**
    `BaseTask.ets:73`, copied from the method directly above it. `onReceiveData`
    is how the query task streams its batches to the page
    (`QuerySandBoxFile.ets:33`), so a failure there means the file list never
    arrives - and the only log line that would say so names cancellation, under
    the same tag.

## References

- Document:
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/thumbnail_taskpool-0000002338966908
- @ohos.taskpool (`Task`, `execute`, `cancel` semantics, `sendData`,
  `isCanceled`, error codes):
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-taskpool
- LazyForEach (key uniqueness, `notifyDataAdd` vs `notifyDataReload`,
  combining with @Reusable):
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-lazyforeach
- Image decoding (when to release ImageSource and PixelMap):
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-decoding
- PhotoViewPicker:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- hilog (format-argument mapping):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
