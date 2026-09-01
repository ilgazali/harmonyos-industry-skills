---
id: UTIL-19
title: Unzip a file two ways - zlib.decompressFile inline and the same call inside a ThreadWorker
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/19_unzip_compressed_file_demo.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/unzip_compressed_file_demo-0000002331316561
sample: huawei_industry_tree/15_utilities/downloads/UnzipCompressedFileDemo.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [base, fs, hilog, window, worker, zlib]
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0048, HW-15-0049, HW-15-0100, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card when the app must **expand a zip archive inside its own
sandbox** - an offline map pack, a downloaded theme, a signed-document bundle,
an update payload - and you are deciding whether to do it on the UI thread or
hand it to a worker.

The sample is deliberately a side-by-side comparison of exactly that choice.
The list rows call `decompressWorker()`, which spawns a `ThreadWorker` that
runs `zlib.decompressFile` off the UI thread and posts a message back. The
bottom button calls `decompressZlib()`, which runs the same API inline. Both
end in the same toast. Reading the two next to each other is the point of the
sample.

The decision generalises: `zlib.decompressFile` is already asynchronous
(callback-based), so the worker buys you nothing for a small archive - it
costs a thread spin-up and a serialisation hop. It starts paying when the
archive is large enough that the *callback* work (walking `fs`, re-reading
entries, hashing) would block a frame, or when several archives are expanded
concurrently. **For CPU-bound one-shot work, `taskpool` is usually the
better answer than `worker`**; reach for `worker` when the thread needs to
stay alive and hold state across many messages.

## Feature checklist

- On entry the app creates two source folders and two `(解压)` output folders
  in `cacheDir`, writes a `test.txt` into each source folder, and zips each
  source folder into `cacheDir/<name>.zip`.
- A document-style list shows the two archives with a file icon, a name and a
  "N 个文件" subtitle.
- Tapping a row unzips that archive on a worker thread and swaps the subtitle
  for a green tick.
- The bottom button unzips both archives inline on the UI thread.
- Either path ends with a 解压成功 (unzip succeeded) toast.
- Output lands in `.../haps/<module>/cache/<name>(解压)/`.

## Architecture

One `entry` module, three source files plus the worker. There is no service
layer: the file operations are free functions at the bottom of the page file.

```
entry/src/main/ets
├── entryability/EntryAbility.ets   loads pages/UnzipPage
├── entrybackupability/
├── model/ListModel.ets             @ObservedV2 ListInfo + LISTCOMPONENT (2 static rows)
├── pages/UnzipPage.ets             @Entry @ComponentV2: the list, the button, and
│                                   createCompress / createDecompress / createFile /
│                                   compressFile / decompressZlib / decompressWorker
└── work/Worker.ets                 ThreadWorkerGlobalScope: onmessage -> decompressFile -> postMessage
```

The documented 工程目录 matches the zip.

**The design decision worth copying** is not in the ArkTS at all - it is the
worker registration in `entry/build-profile.json5`:

```json5
"buildOption": {
  "sourceOption": {
    "workers": [
      "./src/main/ets/work/Worker.ets"
    ]
  }
}
```

A worker entry file must be listed here or it is not built as a separate
bundle, and `new worker.ThreadWorker('entry/ets/work/Worker.ets')` fails at
runtime with a path that looks perfectly correct. Note the two different
spellings: the build profile uses a **source** path
(`./src/main/ets/work/Worker.ets`) while the constructor uses the **module
+ built** path (`entry/ets/work/Worker.ets`, no `src/main`). Getting this
pair right is most of what "using a worker" costs.

The state model is the modern one and worth noting: `ListInfo` is
`@ObservedV2` with `@Trace isFinish`, the page is `@ComponentV2` with
`@Local`, and the avoid-area values come through `AppStorageV2.connect`
rather than `@StorageProp`. `@Trace` on a single field is what makes
`item.isFinish = true` inside a `ForEach` row redraw only that row.

## Implementation steps

1. **Register the worker file** in `entry/build-profile.json5` under
   `buildOption.sourceOption.workers`.
2. **Create the output directory before decompressing.**
   `zlib.decompressFile` writes into an existing directory; `fs.mkdirSync`
   throws if it is already there, which is why the sample wraps it in a
   `try`.
3. **Keep every path inside the sandbox.** `inFile` and `outFile` must be
   application sandbox paths obtained from the `Context` - here
   `context.cacheDir`.
4. **Do the work off the UI thread when it is worth it**: construct a
   `ThreadWorker`, set `onmessage`, then `postMessage` the two paths. Worker
   messages are structured-cloned, so send plain data - paths, not `Context`
   objects or `Resource`s.
5. **Branch on `errData` in the `decompressFile` callback.** The callback
   fires on both success and failure; reporting success unconditionally is
   the sample's headline defect (`HW-15-0048`).
6. **Set the per-row completion flag inside the callback**, not at click time
   (`HW-15-0048`).
7. **Register `onerror` and `onexit` on the host-side worker and terminate
   from them**, otherwise a failing worker is stranded for the process
   lifetime (`HW-15-0049`).
8. **Do not race the setup.** `compressFile` in `aboutToAppear` is
   asynchronous; a fast first tap can ask to decompress an archive that does
   not exist yet (`HW-15-0048`).

## Verified snippets

All snippets are from `UnzipCompressedFileDemo.zip`. Corrected forms are marked.

**Inline decompression - `entry/src/main/ets/pages/UnzipPage.ets`** (corrected, see `HW-15-0048`)

```typescript
function decompressZlib(context: Context, name: string, promptActionDecompress: PromptAction) {
  let outFileDir = `${context.cacheDir}/${name}(解压)`
  let inFile = `${context.cacheDir}/${name}.zip`
  let options: zlib.Options = {
    level: zlib.CompressLevel.COMPRESS_LEVEL_DEFAULT_COMPRESSION
  }
  try {
    zlib.decompressFile(inFile, outFileDir, options, (errData: BusinessError) => {
      if (errData !== null) {
        // FIX: shipped code logs at info level and falls through to the success toast
        hilog.error(0x0000, 'testTag', `errData is errCode:${errData.code}  message:${errData.message}`)
        promptActionDecompress.showToast({ message: '解压失败' })   // no failure string resource exists
        return
      }
      promptActionDecompress.showToast({
        message: $r('app.string.success')
      })
    })
  } catch (errData) {
    let code = (errData as BusinessError).code
    let message = (errData as BusinessError).message
    hilog.error(0x0000, 'testTag', `errData is errCode:${code}  message:${message}`)
  }
}
```

**The shape of `decompressFile` is what to read carefully.** It has both a
synchronous throw path and an asynchronous error path, and they carry
different failures: bad arguments and a missing sandbox permission throw out
of the call, while a corrupt archive, a missing input file or a full disk
arrive as a non-null `errData` in the callback. Handling only one of the two -
which is what the shipped code effectively does, since its callback branch has
no `return` - means the primary failure mode of an unzip feature is
invisible. The document's own snippet is worse still: it shows the callback
with no `errData` check whatsoever and a bare success toast.

`options.level` is a *compression* level and has no effect on decompression;
it is passed because `Options` is shared between `compressFile` and
`decompressFile`. The value that would matter here, and that neither the
sample nor the doc sets, is a size or entry-count limit - `decompressFile`
will happily expand a zip bomb into `cacheDir`.

**Setting the row flag - same file** (corrected, see `HW-15-0048`)

```typescript
.onClick(() => {
  // FIX: shipped code also sets item.isFinish = true here, before any work runs.
  // The flag belongs in the worker's onmessage, next to the toast.
  decompressWorker(this.context, item.name, this.promptAction)
})
```

```typescript
Button($r('app.string.button'))
  .onClick(() => {
    // FIX: shipped code sets LISTCOMPONENT[n].isFinish = true synchronously after each call
    decompressZlib(this.context, LISTCOMPONENT[0].name, this.promptAction)
    decompressZlib(this.context, LISTCOMPONENT[1].name, this.promptAction)
  })
```

`@Trace isFinish` makes the tick appear the instant it is assigned, so
assigning at click time means the tick is a record of *tapping*, not of
succeeding. Combined with the unconditional toast, the sample reports a
successful extraction for an archive that does not exist - which is easy to
observe, because `compressFile` is started in `aboutToAppear` and is
asynchronous: tap a row fast enough after launch and you decompress a file
that has not been written yet.

**The worker body - `entry/src/main/ets/work/Worker.ets`** (corrected, see `HW-15-0048`)

```typescript
const WORKERPORT: ThreadWorkerGlobalScope = worker.workerPort;

WORKERPORT.onmessage = (e: MessageEvents) => {
  let pathDir: string = e.data.pathDir;               // 沙箱目录
  let rawfileZipName: string = e.data.rawfileZipName; // 带.zip后缀的压缩文件名称

  let options: zlib.Options = {
    level: zlib.CompressLevel.COMPRESS_LEVEL_DEFAULT_COMPRESSION
  };

  try {
    zlib.decompressFile(rawfileZipName, pathDir, options, (errData: BusinessError) => {
      if (errData !== null) {
        hilog.error(0x0000, 'testTag', `errData is errCode:${errData.code}  message:${errData.message}`);
        WORKERPORT.postMessage('解压失败');   // FIX: the worker had no failure message at all
        return;
      }
      // Worker线程向主线程发送信息。
      WORKERPORT.postMessage('解压成功');
    })
  } catch (errData) {
    let code = (errData as BusinessError).code;
    let message = (errData as BusinessError).message;
    hilog.error(0x0000, 'testTag', `errData is errCode:${code}  message:${message}`);
    WORKERPORT.postMessage('解压失败');       // FIX: the throw path posted nothing, stranding the worker
  }
}
```

**A worker that can finish without posting is a worker that never gets
terminated.** In the shipped code the host's only `terminate()` lives inside
`onmessage`, so the synchronous `catch` - which posts nothing - leaves the
thread alive forever, and every subsequent list tap spawns another one. Making
*both* exits post a message is half the fix; the other half is on the host.

Note that `hilog.info` was the shipped level for both error branches here and
in `UnzipPage`. Logging a failure at info level is how a defect this visible
survives review - nothing in a normal log filter shows it.

**Host-side worker lifecycle - `entry/src/main/ets/pages/UnzipPage.ets`** (corrected, see `HW-15-0049`)

```typescript
function decompressWorker(context: Context, name: string, promptActionDecompress: PromptAction) {
  const WORKERINSTANCE: worker.ThreadWorker = new worker.ThreadWorker('entry/ets/work/Worker.ets')
  WORKERINSTANCE.onmessage = (e: MessageEvents): void => {
    // 主线程使用terminate()销毁Worker线程。
    promptActionDecompress.showToast({
      message: e.data
    })
    WORKERINSTANCE.terminate()
  }
  // FIX: neither handler exists in the sample
  WORKERINSTANCE.onerror = (e: ErrorEvent): void => {
    hilog.error(0x0000, 'testTag', `worker error: ${e.message}`)
    promptActionDecompress.showToast({ message: '解压失败' })
    WORKERINSTANCE.terminate()
  }
  WORKERINSTANCE.onexit = (code: number): void => {
    hilog.info(0x0000, 'testTag', `worker exited with ${code}`)
  }
  WORKERINSTANCE.postMessage({
    pathDir: `${context.cacheDir}/${name}(解压)`,
    rawfileZipName: `${context.cacheDir}/${name}.zip`
  })
}
```

**`terminate()` must be reachable from every way the worker can end.** There
are three: a normal `postMessage` back (`onmessage`), an uncaught exception
inside the worker (`onerror`), and the worker calling `close()` on itself.
The sample covers only the first, so any failure strands a thread - and
because a fresh `ThreadWorker` is constructed per click rather than reused,
the leak is proportional to how often the user taps. `onexit` does not need
to terminate anything; it is there to confirm the thread actually went away.

The one-worker-per-click construction is itself worth questioning. If the
feature really needs a background thread, keep a single worker for the page
and post several messages to it; if each job is independent and short-lived,
`taskpool` handles the pooling for you.

## Permissions & config

**None.** The sample declares no `requestPermissions`, and it needs none:
everything happens under `context.cacheDir`, which is inside the application
sandbox. Reading a user-chosen archive from the device would change that - it
would go through the file picker and a URI, not through a raw path.

Build configuration that is load-bearing:

- `entry/build-profile.json5` → `buildOption.sourceOption.workers` must list
  the worker source file.
- `compatibleSdkVersion` is `6.0.0(20)`; `strictMode.useNormalizedOHMUrl` is
  `true`, which is what makes the `entry/ets/work/Worker.ets` form of the
  constructor path correct.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The archives are manufactured by the sample itself: `aboutToAppear` creates
  the folders, writes a two-line `test.txt` and zips it. There is no file
  picker and no user-supplied archive anywhere in the demo.
- Output path is `/data/app/el2/100/base/<bundle>/haps/<module>/cache/`, as
  the document notes. `cacheDir` can be cleared by the system at any time.
- The list is two hardcoded `ListInfo` rows; the "3 个文件" / "4 个文件"
  counts are string literals, not counted from the archive.
- Extracted files are never listed or opened - the feature ends at the toast.
- The bottom button is positioned with `margin({ top: 456 })`, so the layout
  is tied to one screen height.
- `createCompress` and `createDecompress` are byte-identical functions.

## Pitfalls

- **`HW-15-0048`** (B/medium, confirmed): both unzip paths report success
  unconditionally. In `UnzipPage.decompressZlib` the `errData` branch only
  logs (at `info` level) and falls through to the 解压成功 toast; the worker
  has no failure message at all. Meanwhile `item.isFinish = true` is set
  synchronously on click and `LISTCOMPONENT[n].isFinish = true` immediately
  after each `decompressZlib` call, so the green tick appears before any work
  runs - and since `compressFile` in `aboutToAppear` is async, a fast first
  tap can "successfully" decompress an archive that does not exist yet.
  Corrupt or missing zips, the primary failure mode of an unzip feature, are
  therefore invisible. Fix: `return` from the error branch with a failure
  toast, and move the flag assignment into the completion callback.
- **`HW-15-0049`** (B/medium, confirmed): the worker leaks whenever it fails.
  `terminate()` is called only from `onmessage`, and no `onerror`/`onexit` is
  registered on the host side, so a throw inside `Worker.ets`'s `catch` - which
  posts nothing - strands the `ThreadWorker` for the app's lifetime. A fresh
  worker is constructed on every list click, so the leak accumulates. Fix:
  register `onerror` (terminate + report) and `onexit`, and post a message
  from the worker's failure paths too.

## References

- `documentation/harmonyos-references/03_system/js-apis-zlib.md` -
  `decompressFile`, `compressFile`, `Options`, sandbox-path requirement,
  error codes
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-zlib
- `documentation/harmonyos-references/02_application-framework/js-apis-worker.md` -
  `ThreadWorker`, `postMessage`, `onmessage`, `onerror`, `onexit`,
  `terminate`, `ThreadWorkerGlobalScope`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-worker
- `documentation/harmonyos-guides/03_application-framework/worker-introduction.md` -
  worker registration in `build-profile.json5` and the two path spellings
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/worker-introduction
- `documentation/harmonyos-guides/03_application-framework/taskpool-vs-worker.md` -
  when a one-shot job belongs in `taskpool` instead
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/taskpool-vs-worker
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `mkdirSync`, `openSync`, `writeSync`, `closeSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
