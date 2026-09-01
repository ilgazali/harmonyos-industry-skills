---
id: MEDIA-38
title: Metronome - a Worker thread's interval drives low-latency SoundPool beats
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/38_news-v1_2-ts_126.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/news-v1_2-ts_126-0000002444273305
sample: huawei_industry_tree/13_media_entertainment/downloads/SoundPoolBeatDemo.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.AudioKit", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.MediaKit", "@kit.PerformanceAnalysisKit"]
apis: [audio, common, hilog, media, window, worker]
min_api: 20
modules: [entry (entry)]
findings: [HW-13-0006, HW-13-0086, HW-13-0087, HW-13-0005, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when short sounds must fire **on a schedule that must not drift
when the UI is busy** - a metronome, a countdown tick, a drum machine step
sequencer, a rep timer, a game turn clock. Two decisions carry the pattern: the
clock lives on a `Worker` thread so layout and animation cannot delay it, and
the sound is played through `SoundPool` rather than `AVPlayer` because
`SoundPool` keeps decoded PCM resident and starts in milliseconds.

Use `SoundPool` whenever the same short asset is played over and over:
load once, get a `soundId`, then `play(soundId)` costs nothing like a player
setup. Use `AVPlayer` for anything long, streamed or seekable. The split is the
same one Android developers know, and the reason is the same - a pool holds
decoded audio, a player holds a pipeline.

The Worker half generalises past audio. Any periodic work whose timing matters
more than its payload - polling, sampling, a stopwatch - can sit behind the
same one-line protocol this sample uses: post `true` to start, post `false` to
stop, receive a message per tick.

**A note on where this page lives.** The document is published under
`architecture-guides/news-v1_2-ts_126-...`, a news-reading slug, although its
subject is audio playback; the repo path
`docs/38_news-v1_2-ts_126.md` preserves that. The slug tells you nothing about
the content - do not look for a news feature here, and do not trust sibling
`news-v1_2-*` slugs to be about news either.

## Feature checklist

- A dark 专业节拍器 (professional metronome) screen with four bars in a row.
- The bars light in rotation, one per beat; beat 1 of every 4 is red and full
  height, beats 2-4 are white and a third of the height.
- A single large play/pause image toggles the metronome.
- While running, one beat sounds per second.
- Pressing again stops the sound and resets the beat counter to `-1`.
- All `rawfile/soundpool/` audio files are enumerated and loaded into the pool
  at page entry; the first one is what the metronome plays.

## Architecture

One `entry` module, two ArkTS files that matter.

```
entry/src/main/ets
├── entryability/EntryAbility.ets     boilerplate; sets COLOR_MODE_NOT_SET
├── entrybackupability/EntryBackupAbility.ets
├── pages/Index.ets                   @Entry @ComponentV2 - UI, SoundPool, load/play/stop (221 lines)
└── workers/Worker.ets                the clock: setInterval -> postMessage (19 lines)
```

The documented tree matches the zip.

The state model is V2: `@ObservedV2 class SoundFile` with `@Trace` fields, and
the page is `@ComponentV2` with three `@Local`s (`isPlaying`, `interval`,
`soundFilesArr`). Worth noting because the beat animation is driven purely by
`this.interval % 4`, evaluated inside four sibling `Row`s - no animation API,
no timer in the UI, just a counter that a message increments.

**The design decision worth copying** is that both the `SoundPool` and the
`ThreadWorker` are **module-scope singletons, created once per module load**:

```typescript
let soundPool: media.SoundPool;                                        // assigned in create()
const workerInstance = new worker.ThreadWorker('entry/ets/workers/Worker.ets');
```

A worker is an OS thread with its own VM instance; creating one per interaction
is the mistake that makes this pattern fail, and it is exactly what the
document's step-2 snippet does (`HW-13-0086`). Module scope also gives the stop
path something to talk to - the same worker that owns the running interval.

The cost of that choice is that neither is ever torn down: see `HW-13-0005`.

## Implementation steps

1. **Generate the worker from the IDE** (right-click the module > New > Worker)
   so the path in `new worker.ThreadWorker('entry/ets/workers/Worker.ets')`
   and the build config are generated consistently. The path is module-relative
   and does not exist at runtime as a file - getting it wrong fails silently at
   construction.
2. **Create the worker once, at module scope** (`HW-13-0086`), never inside
   `onClick`.
3. **Define a one-value protocol**: host posts `true` to start and `false` to
   stop; the worker posts a message per tick. Nothing else crosses the boundary.
4. **Guard the worker against a double start** (`HW-13-0087`): if a timer id is
   already set, return instead of overwriting it - an overwritten id can never
   be cleared.
5. **Cancel a `setInterval` with `clearInterval`** (`HW-13-0006`), and null the
   stored id afterwards.
6. **Flip the UI state on the click, not on the round-trip** (`HW-13-0087`).
   The sample sets `isPlaying = true` inside `play()`, which only runs when the
   first worker message comes back a second later.
7. **Create the pool with the right stream usage**:
   `audio.StreamUsage.STREAM_USAGE_MUSIC` for a metronome; the pool's first
   argument (`32`) is the maximum number of simultaneous streams.
8. **Subscribe to `loadComplete` before loading**, and only call `play` for
   sounds that have reported in - `load()` resolving with a `soundId` means the
   file was accepted, not that it is ready.
9. **Release the pool and terminate the worker** in `aboutToDisappear`
   (`HW-13-0005`).

## Verified snippets

All snippets are from `SoundPoolBeatDemo.zip`. Corrected forms are marked.

**The clock - `entry/src/main/ets/workers/Worker.ets`** (corrected, see `HW-13-0006`, `HW-13-0087`)

```typescript
import { ThreadWorkerGlobalScope, worker } from '@kit.ArkTS';

const workerPort: ThreadWorkerGlobalScope = worker.workerPort;
let timer: number = 0; // 定时器

workerPort.onmessage = (e) => {
  if (e.data === true) {
    if (timer) {                       // FIX: sample overwrites a live interval id
      return;
    }
    timer = setInterval(() => {
      workerPort.postMessage('timer');
    }, 1000);
  } else {
    clearInterval(timer);              // FIX: sample calls clearTimeout(timer)
    timer = 0;                         // FIX: sample leaves the stale id in place
  }
};
```

**Nineteen lines, and two of them are wrong in the shipped version.**
`clearTimeout` on an interval id is engine-dependent - it happens to work on
the current runtime, but the Timer API pairs `setInterval` with `clearInterval`
and nothing promises the ids share a namespace. The guard matters more: without
it, two `true` messages produce two intervals, the second assignment loses the
first id, and the metronome keeps beating after stop with no handle left to
cancel it. That is `HW-13-0087` seen from the worker's side.

The worker deliberately sends a bare `'timer'` string rather than a beat
number. Keeping the count on the host is what lets `stop()` reset it to `-1`
without a second message type.

**Start and stop - `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-13-0086`, `HW-13-0087`, `HW-13-0005`)

```typescript
const workerInstance = new worker.ThreadWorker('entry/ets/workers/Worker.ets');   // module scope

@Entry
@ComponentV2
struct Index {
  @Local isPlaying: boolean = false;
  @Local interval: number = -1;
  @Local soundFilesArr: Array<SoundFile> = [];

  async aboutToAppear(): Promise<void> {
    await this.create();
    workerInstance.onmessage = (): void => {        // FIX: sample re-binds this on every start
      this.interval++;
      this.play(this.soundFilesArr[0].soundId);
    };
  }

  aboutToDisappear(): void {                        // FIX: absent in the sample
    workerInstance.postMessage(false);
    workerInstance.terminate();
    soundPool?.release();
  }

  // ... inside build(), on the play/pause image:
  .onClick(() => {
    if (this.isPlaying) {
      this.isPlaying = false;                       // FIX: state on the click, not on the round-trip
      workerInstance.postMessage(false);
      this.stop(this.soundFilesArr[0].soundId);
    } else {
      this.isPlaying = true;
      workerInstance.postMessage(true);
    }
  })
}
```

**The race the correction removes.** In the shipped code `isPlaying` is only
set to `true` inside `play()`, which runs when the worker's first message
arrives - a full second after the tap. Anything the user does in that second
sees `isPlaying === false` and takes the start branch again: a second `true`
message, a second interval, and a metronome that survives stop. Setting the
flag on the click closes the window; the guard in the worker closes it a second
time, which is the right amount of paranoia for a cross-thread toggle.

Binding `onmessage` once in `aboutToAppear` rather than inside the start branch
is the other half. Re-binding on every start is harmless today only because the
handler body is identical each time.

**Loading the pool - same file** (as shipped)

```typescript
let soundPool: media.SoundPool;

async create() {
  let audioRendererInfo: audio.AudioRendererInfo = {
    usage: audio.StreamUsage.STREAM_USAGE_MUSIC,   // 音频流使用类型：音乐
    rendererFlags: 1
  };
  soundPool = await media.createSoundPool(32, audioRendererInfo);
  this.loadCallback(soundPool);                    // 监听 loadComplete - subscribed before loading
  await this.scanRawFile();
}

async scanRawFile() {
  let context = this.getUIContext().getHostContext() as common.UIAbilityContext;
  let files = await context.resourceManager.getRawFileList('soundpool/');
  for (let i = 0; i < files.length; i++) {
    let soundFile: SoundFile = new SoundFile(files[i], -1);
    this.soundFilesArr.push(soundFile);
    await this.load(soundFile.fileName);
  }
}

async load(fileName: string) {
  let context = this.getUIContext().getHostContext() as common.UIAbilityContext;
  let rowFd = await context.resourceManager.getRawFd('soundpool/' + fileName);
  await soundPool.load(rowFd.fd, rowFd.offset, rowFd.length).then((soundId) => {
    this.updListArrSoundId(fileName, soundId);
  }).catch((e: BusinessError) => {
    console.error('load sound err: ', e.code, e.message);
  });
}
```

**`getRawFileList` plus `getRawFd` is the whole asset story.** The directory is
enumerated at runtime, so dropping another `.ogg` into
`resources/rawfile/soundpool/` adds it to the pool with no code change. The
`(fd, offset, length)` triple is mandatory for HAP resources: a rawfile is not
a standalone file, it is a range inside the package, and passing only the fd
loads the wrong bytes.

`soundId` is the currency for everything afterwards - `play`, `stop`, `unload`
all take it - which is why the sample keeps a `SoundFile` record per file and
patches the id in when `load` resolves. The `-1` initialiser is the "not loaded
yet" marker; a production version should refuse to `play` a `-1`, and should
wait for the `loadComplete` callback rather than for `load`'s promise, exactly
as the comment inside `play()` says.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions` - local rawfile audio
through `SoundPool` needs none. `deviceTypes` is `["phone"]`.

The audio configuration is the one place a wrong value is audible:
`STREAM_USAGE_MUSIC` puts the metronome on the media volume stream and makes it
duck and pause like music. For a metronome accompanying live playing,
`STREAM_USAGE_ALARM` or a game usage may be the better choice - it changes
which volume slider the user reaches for, and whether the sound survives a
focus loss.

The page also uses `.expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.TOP, SafeAreaEdge.BOTTOM])`
so the black background reaches under the status bar, rather than doing avoid
area arithmetic in the ability.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- The tempo is hardcoded: `setInterval(..., 1000)` in the worker, i.e. exactly
  60 BPM. Despite the document's 精确调整每分钟的节拍数 ("precisely adjust the
  beats per minute"), there is no BPM control anywhere in the sample - making
  the interval a message payload is the first change a real app needs.
- `setInterval` is a best-effort timer, not a high-resolution clock. It is on a
  Worker so UI work cannot delay it, but for musically tight timing you would
  schedule ahead against an audio clock rather than trigger per tick.
- `soundFilesArr[0]` is the only sound ever played; the other rawfiles are
  loaded and unused. `interval % 4` fixes the bar to 4/4.
- The pool is created with 32 streams, far more than a one-sound metronome
  needs; the number caps simultaneous playback, and oversizing it costs memory.
- There is no unload path in the UI; `unload()` exists and is never called.

## Pitfalls

- **`HW-13-0006`** (B/low, probable): the worker cancels its `setInterval` with
  `clearTimeout(timer)`. Relying on the two id namespaces being shared is
  engine-dependent, and it misleads every later reader about which timer is
  running. Fix: use `clearInterval`.
- **`HW-13-0086`** (E/medium, confirmed): the document's step-2 snippet creates
  a **new** `ThreadWorker` inside `onClick`. Following the document, "stop"
  posts `false` to a freshly created worker that has no timer, while the old
  worker keeps beating - an unstoppable metronome that leaks a thread per tap.
  The shipped code is correct (module scope); only the document is wrong. Fix:
  correct the snippet.
- **`HW-13-0087`** (B/medium, confirmed): double-start race. `isPlaying` is set
  from the worker's first message roughly a second after the tap, so a second
  tap in that window posts `true` again; the worker overwrites its timer id and
  the orphaned interval beats forever. Fix: set the state on the click, and
  return early in the worker when a timer is already running.
- **`HW-13-0005`** (B/medium, confirmed, systematic): the project never calls
  `release()` on the `SoundPool` - one of five media samples in this pack with
  zero `.release()` calls anywhere. The pool holds decoded audio and an audio
  renderer for the process lifetime, and the worker thread is never terminated
  either. Fix: `soundPool.release()` and `workerInstance.terminate()` in
  `aboutToDisappear`.

## References

- `documentation/harmonyos-references/04_media/js-apis-inner-multimedia-soundpool.md` - `createSoundPool`, `load`, `play`, `stop`, `unload`, `loadComplete`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inner-multimedia-soundpool
- `documentation/harmonyos-guides/05_media/using-soundpool-for-playback.md` - the full low-latency short-audio flow, including where `release` belongs
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/using-soundpool-for-playback
- `documentation/harmonyos-references/02_application-framework/js-apis-worker.md` - `ThreadWorker`, `workerPort`, `postMessage`, `terminate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-worker
- `documentation/harmonyos-guides/03_application-framework/worker-introduction.md` - creating a Worker and the path rules
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/worker-introduction
- `documentation/harmonyos-guides/03_application-framework/worker-communicates-with-mainthread.md` - the host/worker message protocol
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/worker-communicates-with-mainthread
- `MEDIA-01` - the audio app architecture page these playback choices sit under
