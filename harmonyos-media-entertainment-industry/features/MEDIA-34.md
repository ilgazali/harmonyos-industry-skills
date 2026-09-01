---
id: MEDIA-34
title: Background m3u8 download - a dataTransfer continuous task, a Worker, and a live-view progress notification
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/34_m3u8_download.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/m3u8_download-0000002492523889
sample: huawei_industry_tree/13_media_entertainment/downloads/M3U8Download.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BackgroundTasksKit", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.MediaKit", "@kit.NetworkKit", "@kit.NotificationKit", "@kit.PerformanceAnalysisKit"]
apis: [backgroundTaskManager, common, fs, hilog, http, media, notificationManager, request, url, util, wantAgent, window]
permissions: [ohos.permission.INTERNET, ohos.permission.KEEP_BACKGROUND_RUNNING]
min_api: 20
modules: [entry]
findings: [HW-13-0012, HW-13-0075, HW-13-0076, HW-13-0077, HW-13-0098, HW-13-0099]
status: verified-with-fixes
---

## When to use

Load this card when the app must **fetch an HLS asset for offline playback
while something else keeps playing** - a podcast episode downloading behind the
now-playing screen, a lesson cached before a flight, an episode saved on Wi-Fi.
An m3u8 is not one file: it is a text playlist that points at another playlist
that points at hundreds of `.ts` segments, so the download is a loop, not a
request, and it has to survive the app going to background.

Three mechanisms combine, and each is reusable on its own. A **`dataTransfer`
continuous task** buys background execution time and hands back a
`notificationId`. A **Worker** keeps the segment loop off the UI thread. A
**live-view notification** with the `downloadTemplate` renders the progress bar
the system already knows how to draw. Swap "m3u8 segments" for "chapter images"
or "map tiles" and the shape is unchanged.

The download loop itself is the part to rewrite rather than copy. The sample
drives it from a 500 ms polling timer over a single `isDownLoading` boolean,
which has no failure path at all: **read `HW-13-0075` and `HW-13-0076` before
adopting it**, and be aware that with the shipped placeholder URL the worker
crashes on its first message.

## Feature checklist

- A now-playing screen with a local `art.mp3` playing through `AVPlayer`,
  unaffected by the download.
- Tapping the second control icon starts an m3u8 download and raises a toast.
- A `dataTransfer` continuous task is requested first, so the download keeps
  running when the app is backgrounded.
- The playlist and every `.ts` segment are fetched into a per-download
  directory under `tempDir`.
- A system live-view notification shows "文件已下载N%" and the name of the
  segment currently in flight, refreshed on every progress callback.
- On completion the continuous task stops, a success toast appears, and the
  worker is terminated.
- A rewritten `playlist.m3u8` referencing the local `0.ts`, `1.ts`, ... is
  written next to the segments so the folder is playable offline.
- Once the worker has exited, further taps show a "worker gone" toast instead
  of doing nothing.

## Architecture

One `entry` module. The interesting split is thread-wise, not folder-wise:
`components/` and `model/` run on the UI thread, `workers/` and `utils/` run in
the worker.

```
entry/src/main/ets
├── components/AudioPlay.ets        the player UI + task lifecycle + worker owner + notification
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── model/
│   ├── AVPlayerModel.ets           AVPlayer wrapper for the local art.mp3
│   └── M3u8Model.ets               VideoInfo / tsInfo / M3U8Info interfaces
├── pages/MainPage.ets              @Entry, gradient background + avoid-area padding
├── utils/
│   ├── Loading.ets                 toast helper bound to the UIContext
│   ├── Logger.ets                  hilog wrapper
│   └── M3u8Utils.ets               download + parse + playlist rewrite (runs in the worker)
└── workers/M3u8Worker.ets          the worker entry point and the segment scheduler
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that the worker is created at *module
scope*, not inside the component:

```typescript
const workerInstance = new worker.ThreadWorker('entry/ets/workers/M3u8Worker.ets');
```

Spawning a worker is expensive, and a download that must outlive a navigation
cannot be owned by a component that may be destroyed. Hoisting it to the module
means the worker survives page churn and there is exactly one of it. The price
is that `terminate()` is final for the process lifetime - which the sample
handles honestly with a `hasWorker` flag set from `onexit` and a toast telling
the user so. That is the right trade for a demo; a real app would terminate
per-download and re-create, or keep the worker and stop sending it work.

The second structural choice worth noting is that `M3u8Utils` is imported *by
the worker*, so `http`, `fs` and all the parsing run off the UI thread while the
main thread only ever handles two message codes: `100` (progress) and `200`
(done).

## Implementation steps

1. **Declare the background mode and permissions.** `INTERNET`,
   `KEEP_BACKGROUND_RUNNING`, and `"backgroundModes": ["dataTransfer"]` on the
   ability - the mode string must match the one passed to
   `startBackgroundRunning`.
2. **Request the continuous task before posting to the worker.** Build a
   `WantAgent` with `OperationType.START_ABILITY` pointing at your own
   `EntryAbility`, then `startBackgroundRunning(context, ['dataTransfer'],
   wantAgentObj)`.
3. **Keep the returned `notificationId`.** The live-view notification will only
   update if `NotificationRequest.id` is exactly the id the task handed back.
4. **Post the job to the worker** with the URL, the target directory and the
   folder name. Build the folder name with a clean template literal - the
   sample's has a stray closing brace (`HW-13-0077`).
5. **Fetch and parse the master playlist,** resolving relative and
   root-relative segment URLs against the playlist URL. **Check the parse
   result before indexing it** (`HW-13-0076`).
6. **Fetch the media playlist** the same way and collect `#EXTINF` durations
   alongside the `.ts` URLs - you need the durations to rewrite the local
   playlist.
7. **Download segments one at a time,** and register `'fail'` as well as
   `'complete'`, resetting the in-flight flag on both (`HW-13-0075`).
8. **Await the playlist write** before posting the completion message
   (`HW-13-0077`).
9. **On the main thread,** refresh the notification on code `100`; on code
   `200` stop the continuous task, clear the loading flag and terminate.

## Verified snippets

All snippets are from `M3U8Download.zip`. Corrected forms are marked.

**Starting the task and the job - `entry/src/main/ets/components/AudioPlay.ets`** (corrected, see `HW-13-0077`)

```typescript
private notificationId: number = 0;

// on the second control icon
if (index === 1 && this.isLoading === false) {
  this.isLoading = true;
  let m3u8Str = 'xxxxx.xxx.xxx/xxxxxxx.m3u8';        // replace with a real HTTP(S) address
  let filePath = this.context.tempDir;
  let fileName = `download${new Date().getTime()}`;  // FIX: sample has a stray '}' at the end
  workerInstance.postMessage({
    code: 200, context: this.context, m3u8Url: m3u8Str,
    filePath: filePath, fileName: fileName
  });
  this.startContinuousTask();
  Loading.showToast($r('app.string.toast_message_1'));
}

startContinuousTask() {
  let wantAgentInfo: wantAgent.WantAgentInfo = {
    wants: [{ bundleName: 'com.example.m3u8download', abilityName: 'EntryAbility' }],
    actionType: wantAgent.OperationType.START_ABILITY,
    requestCode: 0,
    actionFlags: [wantAgent.WantAgentFlags.UPDATE_PRESENT_FLAG],
  };
  wantAgent.getWantAgent(wantAgentInfo).then((wantAgentObj: WantAgent) => {
    let list: string[] = ['dataTransfer'];
    backgroundTaskManager.startBackgroundRunning(this.context, list, wantAgentObj)
      .then((res: backgroundTaskManager.ContinuousTaskNotification) => {
        this.notificationId = res.notificationId;    // the ONLY id the live view will accept
      });
  });
}
```

**The `WantAgent` is not optional and its target is your own ability.** A
continuous task must be able to bring the user back to the app when they tap
the system notification, so the task API refuses to start without one.
`UPDATE_PRESENT_FLAG` means a second request replaces the pending intent rather
than stacking one.

`ContinuousTaskNotification.notificationId` is the load-bearing return value.
The system has already published a notification for the task; publishing your
own `NOTIFICATION_CONTENT_SYSTEM_LIVE_VIEW` request under a *different* id
silently fails to update it. Note also that `postMessage` sends
`this.context` - a `UIAbilityContext` - across the thread boundary, which is
what lets `request.downloadFile` run inside the worker at all.

**The progress notification - same file** (as shipped)

```typescript
updateDownloadProcess(process: number, fileName: string) {
  let downLoadTemplate: notificationManager.NotificationTemplate = {
    name: 'downloadTemplate',                    // the only template name currently supported
    data: {
      title: `文件已下载${process}%`,             // "file downloaded N%"
      fileName: `正在下载：${fileName}`,          // "downloading: <name>"
      progressValue: process,
    }
  };
  let request: notificationManager.NotificationRequest = {
    id: this.notificationId,                     // must be the continuous task's id
    content: {
      notificationContentType: notificationManager.ContentType.NOTIFICATION_CONTENT_SYSTEM_LIVE_VIEW,
      systemLiveView: {
        typeCode: 8,                             // 8 = upload/download; the only supported value
        title: `下载到应用沙箱`,                   // "downloading into the app sandbox"
        text: `正在下载：${fileName}`,
      }
    },
    notificationSlotType: notificationManager.SlotType.LIVE_VIEW,
    template: downLoadTemplate
  };
  notificationManager.publish(request);
}
```

**Four fields are fixed by the platform and three are yours.** `name:
'downloadTemplate'`, `typeCode: 8`, `ContentType.NOTIFICATION_CONTENT_SYSTEM_LIVE_VIEW`
and `SlotType.LIVE_VIEW` are the contract that makes the system render a
download-shaped live view with a progress bar; `title`, `fileName` and
`progressValue` are the payload. Getting the slot type wrong downgrades this to
an ordinary notification with no progress bar and no ability to update in place.

Note that `process` here is `Math.ceil((completedCount / tsCount) * 100)` -
segment-granular, not byte-granular. The `progress` callback fires with
`receivedSize`/`totalSize` for the current segment and both are ignored. That
is a defensible simplification for equal-length HLS segments and a poor one for
anything else.

**The segment scheduler - `entry/src/main/ets/workers/M3u8Worker.ets`** (corrected, see `HW-13-0075`, `HW-13-0076`, `HW-13-0077`)

```typescript
let isDownLoading: boolean = false;
let completedCount: number = 0;
let tsCount: number = 0;
let intervalId: number = 0;

function downLoadFile(context: common.UIAbilityContext, fileUrl: string,
                      filePath: string, fileName: string) {
  try {
    isDownLoading = true;
    request.downloadFile(context, { url: fileUrl, filePath: filePath + `/${fileName}` },
      (err: BusinessError, data: request.DownloadTask) => {
        if (err) {
          Logger.error(`Failed to request the download. Code: ${err.code}`);
          isDownLoading = false;                  // FIX: sample just returns, flag stays true forever
          return;
        }
        let downloadTask: request.DownloadTask = data;
        downloadTask.on('progress', (receivedSize: number, totalSize: number) => {
          workerPort.postMessage({
            code: 100,
            percent: Math.ceil((completedCount / tsCount) * 100),
            currentFile: fileName,
            completedCount: completedCount
          });
        });
        downloadTask.on('complete', () => {
          completedCount++;
          isDownLoading = false;
        });
        downloadTask.on('fail', (err: number) => {   // FIX: never registered in the sample
          Logger.error(`Download failed for ${fileName}, err ${err}`);
          isDownLoading = false;
          workerPort.postMessage({ code: 500, currentFile: fileName });
        });
      });
  } catch (err) {
    isDownLoading = false;                        // FIX: the throw path also has to clear it
    Logger.error(`Failed to request the download. Code: ${err.code}`);
  }
}

workerPort.onmessage = async (event: MessageEvents) => {
  completedCount = 0;
  let context: common.UIAbilityContext = event.data.context;
  let filePath: string = event.data.filePath + '/' + event.data.fileName;
  fs.mkdirSync(filePath);

  const videoContent = await downloadM3u8(event.data.m3u8Url);
  const videoInfo = obtainM3u8Url(event.data.m3u8Url, videoContent);
  if (videoInfo.length === 0) {                   // FIX: sample indexes videoInfo[0] blind
    workerPort.postMessage({ code: 500, message: 'no playable variant in the master playlist' });
    return;
  }
  const m3u8Content = await downloadM3u8(videoInfo[0].m3u8Url);
  const m3u8Info = obtainTsUrls(videoInfo[0].m3u8Url, m3u8Content);
  tsCount = m3u8Info.tsUrls.length;

  intervalId = setInterval(async () => {
    if (completedCount < tsCount) {
      if (!isDownLoading) {
        downLoadFile(context, m3u8Info.tsUrls[completedCount].tsUrl, filePath, `${completedCount}.ts`);
      }
    } else {
      clearInterval(intervalId);
      const fileStr = await saveM3U8Playlist(m3u8Info, filePath);   // FIX: sample does not await
      workerPort.postMessage({ code: 200, data: fileStr });         // FIX: was JSON.stringify(Promise) -> '{}'
    }
  }, 500);
};
```

**`isDownLoading` is the entire concurrency control, and that is why every exit
path must clear it.** The 500 ms timer only starts the next segment when the
flag is false; only `'complete'` cleared it in the shipped code, so a single
404, a timeout, or a re-download into a path that already exists leaves the
scheduler spinning at 2 Hz forever with the continuous task alive and
`isLoading` stuck true on the UI. `request.downloadFile` has a `'fail'` event
precisely for this, and the callback's `err` branch and the outer `catch` need
the same reset.

The `videoInfo.length` guard matters more than it looks: `downloadM3u8` catches
its own errors and returns `''`, so a network failure - or the shipped
placeholder URL `xxxxx.xxx.xxx/xxxxxxx.m3u8`, which is not even a valid
scheme - produces an empty array and `videoInfo[0].m3u8Url` throws *inside the
worker*. The main thread's `onerror` only logs, so the app's first-run
behaviour is a silent crash with a spinner that never stops.

**Rewriting the playlist for local playback - `entry/src/main/ets/utils/M3u8Utils.ets`** (as shipped)

```typescript
export async function saveM3U8Playlist(m3u8info: M3U8Info, filePath: string): Promise<string> {
  const targetDuration = Math.ceil(Math.max(...m3u8info.durations));

  let m3u8Str = '#EXTM3U\n';
  m3u8Str += '#EXT-X-VERSION:3\n';
  m3u8Str += `#EXT-X-TARGETDURATION:${targetDuration}\n`;
  m3u8Str += '#EXT-X-MEDIA-SEQUENCE:0\n';
  m3u8Str += '#EXT-X-PLAYLIST-TYPE:VOD\n\n';

  m3u8info.durations.forEach((segment, index) => {
    m3u8Str += `#EXTINF:${segment.toFixed(3)},\n`;
    m3u8Str += `${index}.ts\n`;                     // matches the `${completedCount}.ts` names on disk
  });
  m3u8Str += '\n#EXT-X-ENDLIST';

  const playlistPath = `${filePath}/playlist.m3u8`;
  let file: fs.File | undefined = undefined;
  try {
    file = await fs.open(playlistPath, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
    await fs.write(file.fd, m3u8Str);
    return playlistPath;
  } catch (error) {
    Logger.error(`Error writing M3U8 file: ${JSON.stringify(error)}`);
    return '';
  } finally {
    if (file) { fs.close(file.fd); }
  }
}
```

**A downloaded HLS folder is not playable without a rewritten playlist**, and
this is the smallest correct one. The remote playlist's URIs are absolute or
relative to the origin; the segments on disk are named `0.ts`, `1.ts`, ... by
the scheduler, so the local playlist must list those names in order.
`#EXT-X-TARGETDURATION` is mandatory and must be at least the longest
`#EXTINF`, hence the `Math.ceil(Math.max(...))`. `#EXT-X-PLAYLIST-TYPE:VOD` plus
`#EXT-X-ENDLIST` tell the player the list is complete and seekable rather than a
live window.

The `finally` block closing the fd is right; the `await` on `fs.write` is right;
only the caller was wrong (`HW-13-0077`).

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" },
  { "name": "ohos.permission.KEEP_BACKGROUND_RUNNING" }
],
"abilities": [
  {
    "name": "EntryAbility",
    "backgroundModes": ["dataTransfer"],
    "supportWindowMode": ["fullscreen"]
  }
]
```

- Both permissions are `system_grant`: no `reason`, no `usedScene`, no dialog.
- `"backgroundModes": ["dataTransfer"]` must be present *and* must match the
  `['dataTransfer']` array passed to `startBackgroundRunning`; a mismatch fails
  the request at runtime, not at build time.
- Publishing the live-view notification needs no notification permission
  because the id belongs to a continuous task the system already owns.
- `deviceTypes` is `phone` only.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The sample does not run out of the box.** `m3u8Str` is the placeholder
  `'xxxxx.xxx.xxx/xxxxxxx.m3u8'`; the document's version at least carries the
  `https://` prefix. Substitute a real playlist URL before testing.
- Downloads go to `context.tempDir`, which the system may reclaim. Use
  `filesDir` for anything the user expects to keep.
- Segments are fetched strictly one at a time on a 500 ms tick, so throughput is
  capped at two segments per second regardless of bandwidth, and a 600-segment
  episode takes at least five minutes of wall clock even on fibre.
- Encrypted HLS is not handled: `#EXT-X-KEY` is ignored by both parsers.
- Byte-range segments (`#EXT-X-BYTERANGE`) and segment URIs whose path does not
  contain `.ts` are dropped by `obtainTsUrls`.
- `workerInstance.terminate()` on completion is final - `hasWorker` goes false
  and no further download is possible until the app restarts.
- Nothing plays the downloaded folder; the sample proves the download, not the
  offline playback.

## Pitfalls

- **`HW-13-0075`** (B/medium, confirmed): no failure path resets
  `isDownLoading`. The `err` branch of the `downloadFile` callback just
  returns, no `'fail'` listener is registered, and only `'complete'` clears the
  flag - so one failed segment (or a re-download into an existing path) hangs
  the entire download forever, with the 500 ms scheduler spinning and the
  continuous task still running. Fix: register `'fail'`, and clear the flag in
  the error branch and the outer `catch`.
- **`HW-13-0076`** (B/medium, confirmed): `videoInfo[0].m3u8Url` is
  dereferenced with no check that the master playlist parsed. `downloadM3u8`
  returns `''` on any failure, `obtainM3u8Url` then returns an empty array, and
  the worker throws. The main thread only logs `onerror`, so `isLoading` stays
  true and the UI is stuck. With the shipped placeholder URL this is the
  first-run behaviour. Fix: check `videoInfo.length` and report an error
  message to the main thread.
- **`HW-13-0077`** (B/low, confirmed): `saveM3U8Playlist` is not awaited, so the
  completion message carries `JSON.stringify(Promise)` - the string `'{}'` -
  and races the actual write; and the download directory name template literal
  `` `download${new Date().getTime()}}` `` has a stray closing brace, so every
  folder is named `download<ts>}`. Fix: `await` the save, drop the brace.
- **`HW-13-0012`** (B/low, confirmed): the systematic raw-descriptor leak.
  `AVPlayerModel.avPlayerFdSrcDemo` calls
  `context.resourceManager.getRawFd('art.mp3')` and never calls `closeRawFd`,
  and the `AVPlayer` it creates is only released from the `'idle'` state
  handler. The finding's evidence names seven other samples; M3U8Download
  carries the identical shape. Fix: pair every `getRawFd` with `closeRawFd`
  after the player is released.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-worker.md` - `ThreadWorker`, `postMessage`, `onexit`, `terminate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-worker
- `documentation/harmonyos-guides/03_application-framework/worker-introduction.md` - what may cross the thread boundary
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/worker-introduction
- `documentation/harmonyos-references/03_system/js-apis-request.md` - `request.downloadFile`, `DownloadTask` and its `progress` / `complete` / `fail` events
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-request
- `documentation/harmonyos-references/03_system/js-apis-http.md` - `createHttp`, `request`, `responseCode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-http
- `documentation/harmonyos-references/02_application-framework/js-apis-resourceschedule-backgroundtaskmanager.md` - `startBackgroundRunning`, `ContinuousTaskNotification.notificationId`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resourceschedule-backgroundtaskmanager
- `documentation/harmonyos-guides/03_application-framework/continuous-task.md` - the `dataTransfer` mode and the `backgroundModes` declaration
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/continuous-task
- `documentation/harmonyos-references/02_application-framework/js-apis-app-ability-wantagent.md` - `WantAgentInfo`, `OperationType.START_ABILITY`, `UPDATE_PRESENT_FLAG`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-app-ability-wantagent
- `documentation/harmonyos-references/06_application-services/js-apis-notificationmanager.md` - `NotificationTemplate`, `systemLiveView`, `SlotType.LIVE_VIEW`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-notificationmanager
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `fs.open`, `fs.write`, `fs.mkdirSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/05_common-capabilities/js-apis-timer.md` - `setInterval` / `clearInterval`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-timer
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `INTERNET`, `KEEP_BACKGROUND_RUNNING`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `MEDIA-01` - the systematic raw-descriptor leak this sample shares
