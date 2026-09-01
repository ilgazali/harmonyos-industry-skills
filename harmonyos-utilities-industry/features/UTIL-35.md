---
id: UTIL-35
title: Pausable download with rcp - a destination callback for back-pressure and transferRange for resume
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/35_rcp_download_pause.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/rcp_download_pause-0000002473755986
sample: huawei_industry_tree/15_utilities/downloads/RcpDownloadPause.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit", "@kit.RemoteCommunicationKit"]
apis: ["rcp.createSession", "rcp.Request", "Session.fetch", "Session.cancel", "Session.close", "Request.destination", transferRange, "rcp.PausePolicy", "rcp.ReceivingPauseByCache", httpEventsHandler, onDownloadProgress, onDataEnd, "rcp.DebugInfo", "fs.openSync", "fs.writeSync", "fs.accessSync", OpenMode, Queue, "util.TextDecoder", "@ObjectLink", "@Observed", Progress]
permissions: [ohos.permission.INTERNET]
min_api: 20
modules: [entry]
findings: [HW-15-0074, HW-15-0075, HW-15-0101, HW-15-0102]
status: verified-with-fixes
---

## When to use

Load this card when a download is **large enough that the user will want to stop
it** - a video, a map pack, an OTA-style asset bundle - and the app must resume
where it left off rather than start again. The mechanism is Remote
Communication Kit: an `rcp.Session` with a `Request` whose `destination` is a
callback, so bytes arrive in the app instead of being written to a file by the
framework.

That callback is the reason to choose `rcp` over `request.agent` here. Because
the app owns every buffer, it can decide **when** to accept the next chunk: the
callback returns a `Promise`, and the transfer does not continue until that
promise resolves. That is real back-pressure, expressed in three lines, and it
generalises far past downloading - any streamed consumer (a decoder, a parser,
a database writer) that must not be flooded can use the same shape.

Resume is `transferRange: { from: lastTransferPosition }` on the same request
object plus an HTTP range-capable server. **Read `HW-15-0074` before shipping
any of this**: the sample's file writing is append-only, so the resume
arithmetic and the bytes on disk disagree the moment the app is relaunched
mid-download.

## Feature checklist

- A list of five download entries, each with an icon that doubles as the
  action button.
- Tapping a fresh entry starts the download; the icon becomes a pause glyph and
  a percentage plus a progress bar appear.
- The total size is known before the first byte arrives, so the percentage is
  meaningful from the start.
- Tapping during a download pauses it: the transfer stops and the icon becomes
  a start glyph.
- Tapping again resumes from the paused byte offset instead of restarting.
- On completion the bar disappears and the icon becomes a remove glyph; tapping
  it drops the entry from the list.
- A failed response leaves the entry in a retryable state that also resumes from
  the last offset.

## Architecture

One `entry` module. The whole feature lives in a single component that is
instantiated once per list row, so each row owns its own session, request,
buffer queue and file handle.

```
entry/src/main/ets
├── common
│   ├── CommonConstants.ets      cache limits, timeouts, size units, separators
│   ├── CommonEnum.ets           DownloadState: NONE START PAUSED FAILED COMPLETED
│   └── StyleConstants.ets       layout literals
├── component/FileListItem.ets   377 lines: session, back-pressure, file writing, the row UI
├── entryability/EntryAbility.ets
├── entrybackupability/
├── model/DataModel.ets          @Observed DownloadFile + five seed entries
├── pages
│   ├── DownloadPage.ets         NavDestination, the List, URLs read from string resources
│   └── OriginPage.ets           @Entry, a Navigation that immediately pushes DownloadPage
└── utils/Logger.ets
```

The documented tree matches the zip exactly.

**The design decision worth copying** is per-row ownership. `FileListItem`
creates its own `rcp.Session` in `aboutToAppear` and closes it in
`aboutToDisappear`, and it keeps its own `bufferQueue`, `cacheSize` and
`lastTransferPosition`. Five concurrent downloads therefore need no scheduler
and no shared mutable state - the row *is* the transfer. Progress reaches the UI
through `@ObjectLink downloadFile`, an `@Observed` model object owned by the
page, so a row can update its own percentage without the page re-rendering the
list.

The cost of that choice is that state is per-component and dies with it. There
is no persistence of `lastTransferPosition`, which is precisely what makes
`HW-15-0074` bite: on relaunch, state resets to `NONE` and the offset to `0`,
while the partially written file is still on disk.

## Implementation steps

1. **Create one session per transfer** with `rcp.createSession()` inside a
   `try/catch`, and `close()` it in `aboutToDisappear`.
2. **Ask for the size first.** Issue a tiny ranged request
   (`transferRange = { from: 0, to: 1 }`) and read the total out of the
   `content-range` response header. This also proves the server supports ranges
   - without that, resume is impossible.
3. **Set a receiving pause policy**: `{ kind: 'cacheSize', size: SYS_CACHE_LIMIT }`
   caps how much the framework buffers before it stops reading the socket.
4. **Make `destination` a callback,** not a file path. Accumulate the incoming
   `ArrayBuffer` into a queue, add its length to `lastTransferPosition`, and
   update the model's percentage.
5. **Return a pending promise when your own cache is full** and resolve it after
   flushing to disk. That is the back-pressure handshake; returning an
   immediately-resolved promise means "keep sending".
6. **Truncate on a fresh start, append only on resume** (`HW-15-0074`). Opening
   with `APPEND` unconditionally corrupts any file left over from a previous run.
7. **Flush the tail in `onDataEnd`** and set the state to `COMPLETED` there -
   the last partial buffer is still in the queue when the response completes.
8. **Pause with `session.cancel(request)`, inside a `try/catch` around the
   awaited `fetch`** (`HW-15-0075`): cancelling makes the in-flight `fetch`
   reject, and that rejection surfaces in whichever handler is awaiting it.
9. **Guard the click handler on the current state** so a fast double tap cannot
   start two `fetch` calls that interleave into one file (`HW-15-0075`).
10. **Resume by setting `transferRange = { from: lastTransferPosition }`** on the
    same request and calling `fetch` again; accept `206` as well as `200`.

## Verified snippets

All snippets are from `RcpDownloadPause.zip`. Corrected forms are marked.

**The request configuration and the back-pressure callback — `entry/src/main/ets/component/FileListItem.ets`** (as shipped)

```typescript
initRcpConfig() {
  try {
    this.session = rcp.createSession();
  } catch (e) {
    Logger.error(this.downloadFile.fileName, `create rcp session error: ${e.code} ${e.message}`);
    return;
  }
  // 设置接收暂停策略 — the largest amount rcp itself will buffer
  let receivePolicy: rcp.ReceivingPauseByCache = {
    kind: 'cacheSize',
    size: CommonConstants.SYS_CACHE_LIMIT              // 16 KB
  };
  let pausePolicy: rcp.PausePolicy = { receiving: receivePolicy };

  this.request = new rcp.Request(this.downloadFile.url, 'GET');
  this.request.configuration = {
    transfer: {
      pausePolicy: pausePolicy,
      timeout: { connectMs: CommonConstants.CONNECT_TIMEOUT, transferMs: CommonConstants.TRANSFER_TIMEOUT }
    },
    security: { remoteValidation: 'skip' },
    tracing: {
      infoToCollect: { textual: true },
      httpEventsHandler: {
        onDownloadProgress: (totalSize, transferSize, request) => { /* logging only */ },
        // 数据接收完毕后的回调 — the tail of the queue is still unwritten here
        onDataEnd: (request) => {
          this.downloadFile.state = DownloadState.COMPLETED;
          this.itemIcon = this.getItemIcon();
          this.writeFile();
        }
      }
    }
  };

  this.request.destination = (incomingData: ArrayBuffer) => {
    this.lastTransferPosition += incomingData.byteLength;
    this.downloadFile.progress =
      Math.round(this.lastTransferPosition / this.totalSize * CommonConstants.HUNDRED_PERCENT);
    this.cacheSize += incomingData.byteLength;
    this.bufferQueue.add(incomingData);
    return this.cacheSize >= CommonConstants.APP_CACHE_LIMIT ? new Promise((resolve, reject) => {
      // 未调用resolve之前，不会进行下一批次的数据处理
      setTimeout(() => {
        this.writeFile();
        resolve();
      }, CommonConstants.DELAY_TIME);
    }) : new Promise((resolve) => {
      resolve();
    });
  };
}
```

**Two cache limits, two different jobs.** `SYS_CACHE_LIMIT` (16 KB) is the
framework's own receive buffer, declared through `PausePolicy` - once that much
data is unread, `rcp` stops pulling from the socket. `APP_CACHE_LIMIT` (100 KB)
is the app's threshold: below it the destination callback returns an
already-resolved promise, meaning "send more"; at or above it the callback
returns a promise that stays pending until the queued buffers have been written
to disk. **The returned promise is the throttle** - the download literally does
not advance while it is unsettled, which is what keeps a fast link from growing
an unbounded in-memory queue on a slow disk.

`onDataEnd` is not optional. The last chunks usually sit below the 100 KB
threshold, so nothing has flushed them; without the `writeFile()` here the tail
of every file would be lost. Note also `security: { remoteValidation: 'skip' }`,
which disables certificate validation - acceptable for a demo against a test
host, never in a shipping build.

**Writing to disk — same file** (corrected, see `HW-15-0074`)

```typescript
writeFile(truncate: boolean = false) {                 // FIX: shipped signature takes no argument
  let file: fo.File | undefined = undefined;
  try {
    if (fo.accessSync(this.fileDir)) {
      // FIX: a fresh (NONE-state) start must not append onto a stale partial file
      const mode = truncate
        ? fo.OpenMode.WRITE_ONLY | fo.OpenMode.TRUNC
        : fo.OpenMode.WRITE_ONLY | fo.OpenMode.APPEND;
      file = fo.openSync(this.fileDir, mode);
    } else {
      file = fo.openSync(this.fileDir, fo.OpenMode.WRITE_ONLY | fo.OpenMode.APPEND | fo.OpenMode.CREATE);
    }
    while (this.bufferQueue.length > 0) {
      let buffer = this.bufferQueue.pop();
      if (!buffer || !buffer.byteLength || buffer.byteLength <= 0) {
        continue;
      }
      let arrayInt = new Uint8Array(buffer);
      fo.writeSync(file.fd, arrayInt.buffer);
      this.cacheSize -= buffer.byteLength;
    }
  } catch (e) {
    Logger.error('DownloadPage', `write download file error: ${e.code} ${e.message}`);
  } finally {
    if (file) {
      fo.closeSync(file);
    }
  }
}
```

**Open, drain, close - every flush.** The handle is deliberately not held open
across the whole download: each flush opens the file, writes whatever the queue
holds, and closes in a `finally`, so a crash mid-transfer never leaves a dangling
fd and the bytes already flushed are durable. `APPEND` is what makes that safe
for a *resume*, because the write offset is always the current end of file, so
no seek arithmetic is needed.

The shipped code, though, uses `APPEND` for **every** start, including a fresh
one. The component's state (`DownloadState`) and `lastTransferPosition` live only
in memory, so after a kill and relaunch the row is `NONE` with offset `0` while
the partial file is still on disk - and the new download appends a complete copy
onto the stale prefix (`HW-15-0074`). The one-line fix is above: truncate (or
delete the file) when starting from `NONE`.

**The click handlers — same file** (corrected, see `HW-15-0075`)

```typescript
getItemClickFunc() {
  switch (this.downloadFile.state) {
    case DownloadState.NONE:
      return async () => {
        if (this.downloadFile.state !== DownloadState.NONE) {   // FIX: guard the stale binding
          return;
        }
        this.downloadFile.state = DownloadState.START;
        this.itemIcon = this.getItemIcon();
        this.lastTransferPosition = 0;
        this.writeFile(true);                                   // FIX: truncate any stale file
        try {                                                   // FIX: pause rejects this fetch
          let response = await this.session?.fetch(this.request);
          this.complete = true;
          this.processResponse(response);
        } catch (err) {
          Logger.info(this.downloadFile.fileName, `fetch ended: ${err?.code}`);
        }
      };
    case DownloadState.START:
      return () => {
        this.downloadFile.state = DownloadState.PAUSED;
        this.hasPaused = true;
        this.itemIcon = this.getItemIcon();
        this.session?.cancel(this.request);      // makes the awaited fetch above reject
        this.complete = true;
      };
    case DownloadState.PAUSED:
      return async () => {
        this.downloadFile.state = DownloadState.START;
        this.itemIcon = this.getItemIcon();
        this.request.transferRange = { from: this.lastTransferPosition };
        this.complete = false;
        try {                                                   // FIX: same rejection path
          let response = await this.session?.fetch(this.request);
          this.complete = true;
          this.processResponse(response);
        } catch (err) {
          Logger.info(this.downloadFile.fileName, `fetch ended: ${err?.code}`);
        }
      };
  }
}
```

**Pause is a cancellation, and cancellation is a rejection.** `session.cancel`
aborts the transfer the `NONE`/`PAUSED` handler is still awaiting, so
`await this.session?.fetch(...)` throws. In the shipped code neither handler has
a `try/catch`, so every single pause raises an unhandled promise rejection and
the statements after the `await` - including `processResponse` - never run
(`HW-15-0075`).

The second half of the same finding is the binding: `onClick(this.getItemClickFunc())`
captures the closure for the state **at render time**. Two fast taps on a fresh
row therefore both run the `NONE` branch, starting two `fetch` calls whose
`destination` callbacks push into one `bufferQueue` and one file. The state
guard at the top of the handler is the cheap defence; re-reading
`this.downloadFile.state` inside a single handler instead of returning a
per-state closure is the structural one.

Resume itself is the single line `this.request.transferRange = { from: this.lastTransferPosition }`
on the *same* `Request` object, which is why the configuration and the
destination callback survive across a pause.

**Learning the total size before downloading — same file** (as shipped)

```typescript
async getFileTotalSize() {
  let request = new rcp.Request(this.downloadFile.url, 'GET');
  request.transferRange = { from: 0, to: 1 };            // ask for two bytes
  try {
    let rep = await this.session?.fetch(request);
    if (rep) {
      // content-range: bytes 0-1/52428800  ->  everything after the slash
      let contentRange = rep.headers['content-range'];
      let sizeStr =
        contentRange ? contentRange.substring(contentRange.indexOf('\/') + 1, contentRange.length) : '0';
      this.totalSize = Number(sizeStr);
    }
  } catch (err) {
    Logger.error('DownloadPage', `getTotalSize err: code is ${err.code}, message is ${JSON.stringify(err)}`);
  }
}
```

**A two-byte range request does two jobs at once.** It yields the total length
from `content-range` - more reliable than `content-length`, which on a ranged
response describes the range, not the file - and it proves the server honours
`Range` at all. If this call comes back without a `content-range`, resume will
not work later either, which makes it a good place to fall back to a
non-resumable strategy. Note `totalSize` stays `0` on failure, and the
percentage then evaluates to `Infinity` or `NaN`; a production version should
hide the bar until the size is known.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" }
]
```

- `INTERNET` is `system_grant`, so no runtime request and no `reason` needed.
- `deviceTypes` is `["phone"]`.
- The five download URLs are **string resources**, read in
  `DownloadPage.aboutToAppear` through `resourceManager.getStringSync`, and
  ship empty - fill them in with range-capable URLs before running.
- `routerMap` is declared: `OriginPage` is the `@Entry` and immediately pushes
  `DownloadPage` onto a `NavPathStack` provided through `@Provide('pathStack')`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The server must support HTTP range requests.** Both the size probe and the
  resume depend on it; against a server that ignores `Range`, a resume silently
  re-downloads from byte 0 into an appending file.
- Timeouts are set enormous (`connectMs` one hour, `transferMs` sixty hours), so
  a dead connection is not detected in any practical time.
- `remoteValidation: 'skip'` disables TLS peer validation - demo-only.
- Nothing is persisted. Download state and byte offset are component fields, so
  leaving the page (or killing the app) loses the ability to resume, and
  `aboutToDisappear` closes the session and aborts the transfer.
- `DownloadPage` applies `padding({ top: this.topRectHeight, bottom: this.bottomRectHeight })`
  with the raw pixel values from `AppStorage`, without `px2vp` - the safe-area
  padding is therefore too large by the display's density factor.
- The completed action removes the row from the list but never deletes the file
  from the sandbox.

## Pitfalls

- **`HW-15-0074`** (B/high, confirmed): every write opens the file with
  `OpenMode.APPEND` and a fresh download always starts at byte 0. Exit
  mid-download and relaunch: the state resets to `NONE`, `lastTransferPosition`
  resets to `0`, the partial file survives, and the new transfer appends a full
  copy onto stale bytes. The saved file is corrupt and the resume arithmetic no
  longer matches the disk. Fix: truncate (or delete) when starting from `NONE`,
  and verify the on-disk size before resuming.
- **`HW-15-0075`** (B/medium, confirmed): pause calls `session.cancel` on a
  request whose `fetch` is being awaited with no `try/catch`, so every pause is
  an unhandled rejection and the post-await code is skipped. The same handler is
  bound to the state at last render, so a fast second tap re-runs the `NONE`
  branch and interleaves two transfers into one file. Fix: catch the
  cancellation rejection and guard the handler on the current state.
- Not filed, but worth knowing: `this.complete` is assigned in four places and
  read nowhere, and `hasPaused` is set but never used - both are leftovers, not
  part of the mechanism.
- The `Progress` component is constructed with `value: 30` and then overridden
  by `.value(this.downloadFile.progress)`; the literal is dead, but it will
  briefly mislead anyone reading the row's build method.

## References

- `documentation/harmonyos-references/03_system/remote-communication-rcp.md` - `createSession`, `Request.destination`, `transferRange`, `PausePolicy`, `cancel`, `DebugInfo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/remote-communication-rcp
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `openSync`, `OpenMode.APPEND` / `TRUNC` / `CREATE`, `writeSync`, `accessSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.INTERNET`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-progress.md` - the linear progress bar
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-progress
- `huawei_industry_tree/15_utilities/docs/35_rcp_download_pause.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/rcp_download_pause-0000002473755986
