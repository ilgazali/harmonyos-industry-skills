---
id: NEWS-26
title: Sleep timer for text-to-speech - stop and release TextReader on a countdown
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/26_text_reader.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/text_reader-0000002525367690
sample: huawei_industry_tree/11_news_reading/downloads/TextReader.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit", "@kit.SpeechKit"]
apis: [common, emitter, hilog, window]
permissions: [ohos.permission.KEEP_BACKGROUND_RUNNING]
min_api: 20
modules: [entry (entry)]
findings: [HW-11-0028, HW-11-0031]
status: verified-with-fixes
---

## When to use

Load this card for a **sleep timer over speech playback** - "read the news to
me for twenty minutes and then stop". It is the standard companion to any
listen-to-articles feature: users start playback in bed and want a guaranteed
end, not a decision later.

The pattern has two halves that must not be confused. The **visible**
countdown is UI - a number ticking down once a second so the user can see how
long is left. The **authoritative** shutdown is a single `setTimeout` armed
for the full remaining duration, whose callback calls `TextReader.stop()` and
then `TextReader.release()`. Never derive the shutdown from the display timer;
one dropped tick would silently extend playback.

It generalises to any bounded background activity with a visible remaining
time: a music sleep timer, a guided-meditation session, a timed workout
prompt. The transferable rules are the pause/resume arithmetic (cancel the
timeout, keep the remainder, re-arm with it) and the release discipline on the
speech service. **Read `HW-11-0028` before shipping**: this sample declares
background audio it never actually requests, and the countdown it depends on
is suspended the moment the app backgrounds.

## Feature checklist

- A first page with an `HH:mm:ss` `TimePicker` and a 开始 (start) button.
- Pressing start navigates to a playback page carrying the chosen duration.
- The playback page starts TextReader on the article immediately and shows the
  system 小艺朗读 speech panel with the app's own brand name and icon.
- A three-box countdown at the top ticks down every second.
- 暂停 / 播放 (pause / play) toggles both the speech and the countdown, and
  the remaining time survives the pause.
- When the countdown reaches zero the speech stops, the reader is released and
  the page pops back to the picker.
- 取消 (cancel) stops playback and returns immediately.

## Architecture

One `entry` module, two pages and one utility class. The navigation is
`Navigation` + `NavDestination` with a route map, not `router`.

```
entry/src/main/ets
├── constants/CommonConstants.ets   millisecond constants, the < 10 zero-pad boundary
├── entryability/EntryAbility.ets   full-screen layout, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── pages
│   ├── Index.ets                   @Entry: TimePicker + 开始, pushes PlayPage with the duration
│   └── PlayPage.ets                NavDestination: countdown display, buttons, lifecycle
└── utils/TextReaderUtil.ets        every TextReader call, plus the shutdown setTimeout
entry/src/main/resources/base/profile/route_map.json   PlayPage -> PlayPageBuilder
```

The documented 工程目录 is close but not exact: it omits `constants/` and
spells the backup directory `entrybackupablility`. It also does not mention
`route_map.json`, which is the piece that makes `pushPathByName('PlayPage', ...)`
resolve at all.

**The design decision worth copying** is that `TextReaderUtil` owns the timer,
not the page. Every path that changes playback - start, pause, resume, stop -
goes through one class that arms or cancels the shutdown in the same method
that starts or stops the speech, so the two can never drift apart. The page
keeps only what it draws: three `@State` numbers for the clock and a display
`setInterval`. Communication back to the page is an `emitter` event
(`'TextReader complete'`), which is the right choice here because the shutdown
fires from a timer callback with no component in scope.

**The decision worth questioning** is the `AppStorage` middle layer.
`isPlaying`, `state` and `timeoutId` are written into `AppStorage` by the util
and read back by the page as `@StorageProp`. For a single page this is a
global variable with extra steps - `@Link` or a plain callback would be
narrower - and it is what lets `PlayPage.aboutToDisappear` call
`clearTimeout(this.timeoutId)` on an id it never owned.

## Implementation steps

1. **Collect the duration as h/m/s and convert once to milliseconds** at the
   navigation boundary: `(((h * 60) + m) * 60 + s) * 1000`. Everything
   downstream is milliseconds.
2. **Initialise TextReader with a `ReaderParam` before any other call**,
   supplying `businessBrandInfo` so the system panel carries your app's name
   and icon rather than a blank brand.
3. **Register the listeners immediately after `init`**, in particular
   `requestMore` - and make it actually call `TextReader.loadMore(...)`. The
   sibling AI-recitation sample registers a listener whose body evaluates a
   method reference and throws it away (`HW-11-0010`); this sample is the
   correct version of the same code.
4. **Arm the shutdown in the same method that starts playback**, and store the
   returned id where pause can reach it.
5. **On pause: cancel the timeout and record the remainder.** On resume,
   re-arm with the remainder - never with the original duration.
6. **In the shutdown callback, `stop()` first and `release()` inside its
   `then`,** then emit the completion event. `stop` is a promise; releasing
   before it resolves races the service teardown.
7. **Make completion idempotent.** The emitter handler in this sample calls
   `stop()` again on an already-released reader - guard it, or let the timeout
   emit and do the releasing only in the handler.
8. **Decide honestly about background playback** (`HW-11-0028`): either call
   `backgroundTaskManager.startBackgroundRunning` with `AUDIO_PLAYBACK` while
   reading and stop it on completion, or delete the
   `KEEP_BACKGROUND_RUNNING` permission and the `backgroundModes` entry. Do
   not ship the declaration without the call.

## Verified snippets

All snippets are from `TextReader.zip`. Corrected forms are marked.

**Start and arm the shutdown — `entry/src/main/ets/utils/TextReaderUtil.ets`** (as shipped)

```typescript
start(): void {
  TextReader.start(this.readInfoList!, this.selectedReadInfo?.id).catch((err: BusinessError) => {
    console.error(`Failed to start TextReader. code is ${err.code} message is ${err.message}`);
  });
  this.textTimerController?.start();
  this.isPlaying = true;
  AppStorage.set('isPlaying', this.isPlaying);
  this.timeoutID = setTimeout(() => {
    TextReader.stop().then(async () => {
      TextReader.release();
      emitter.emit('TextReader complete');
    }).catch((e: BusinessError) => {
      console.error(`TextReader failed to stop. Code: ${e.code}, message: ${e.message}`);
    });
  }, this.timeCount);
  AppStorage.setOrCreate('timeoutId', this.timeoutID);
}
```

**One timeout, not a countdown, is the whole idea.** `this.timeCount` is the
full duration in milliseconds, so the shutdown is a single scheduled event that
cannot accumulate drift from a per-second loop. The visible clock on the page
is a separate `setInterval` and is allowed to be approximate; this timer is
not.

The order inside the callback matters and the **document's snippet has it
backwards** - it emits the completion event before `TextReader.release()`,
which lets the page's handler run against a reader that is still holding its
listeners. The shipped code releases first. Either way, `release()` belongs
inside the `then` of `stop()`, never beside it.

`AppStorage.setOrCreate('timeoutId', ...)` publishes the id so the page can
cancel it from `aboutToDisappear` and from the pause button. That is the
`AppStorage` shortcut noted above: it works, but it means the id has two
owners.

**Pause and resume — same file** (as shipped)

```typescript
pause(): void {
  try {
    TextReader.pause();
    this.isPlaying = false;
    AppStorage.set('isPlaying', this.isPlaying);
  } catch (error) {
    console.error(`pause fail, ${error}`);
  }
}

resume(timeout: number): void {
  try {
    TextReader.resume();
    this.isPlaying = true;
    AppStorage.set('isPlaying', this.isPlaying);
    this.timeoutID = setTimeout(() => {
      TextReader.stop().then(async () => {
        TextReader.release();
        emitter.emit('TextReader complete');
      }).catch((e: BusinessError) => {
        console.error(`TextReader failed to stop. Code: ${e.code}, message: ${e.message}`);
      });
    }, timeout);
    AppStorage.set('timeoutId', this.timeoutID);
  } catch (error) {
    console.error(`pause fail, ${error}`);
  }
}
```

**`resume` takes the remainder as a parameter** - that is the load-bearing
signature. `pause()` deliberately does not touch the timeout; the page cancels
it and captures `remainingTime = this.timeCount` (the countdown's current
value) in the same click handler. So the arithmetic lives at the call site:

```typescript
Button(this.isPlaying ? '暂停' : '播放')
  .onClick(() => {
    if (this.isPlaying) {
      this.textReaderUtil.pause();
      this.textTimerController.pause();
      this.remainingTime = this.timeCount;      // ms still on the clock
      this.pauseCountDown();
      clearTimeout(this.timeoutId);             // cancel the shutdown
    } else {
      this.textReaderUtil.resume(this.remainingTime);
      this.textTimerController.start();
      this.startCountDown();
    }
  })
```

Splitting it this way is defensible - the page owns the display clock, so the
page knows the remainder - but it is also why the shutdown accuracy now depends
on the display interval. A cleaner form records a wall-clock deadline
(`Date.now() + duration`) at start, and computes the remainder as
`deadline - Date.now()` at pause; that survives a suspended interval, which
this version does not.

**Stop and release — same file** (as shipped)

```typescript
stop(): void {
  try {
    TextReader.stop().catch((err: BusinessError) => {
      console.error(`Failed to stop TextReader. code is ${err.code} message is ${err.message}`);
    });
    TextReader.release();
    this.isPlaying = false;
    AppStorage.set('isPlaying', this.isPlaying);
    TextReader.off('stateChange');
    TextReader.off('requestMore');
  } catch (error) {
    console.error(`pause fail, ${error}`);
  }
}
```

This is the 取消 path and it is the release discipline the sibling
AI-recitation sample is missing entirely (`HW-11-0010`): `off` every event you
registered, then let the service go. Two caveats before copying it. First,
`release()` here is *not* inside the `then` of `stop()`, unlike the timeout
path - the two shutdown routes in this one file disagree with each other.
Second, the page's completion handler calls this same `stop()` after the
timeout callback has already stopped and released, so the natural end of a
session runs the teardown twice. Guard with a flag (`isEnd`, which the class
already declares and never uses) or let one route own the teardown.

Note also that `readProgress` is registered in `setActionListener` but never
unregistered here.

**Initialisation and the branded panel — same file** (as shipped)

```typescript
async init(readInfoList: TextReader.ReadInfo[]) {
  const readerParam: TextReader.ReaderParam = {
    keepBackgroundRunning: true,
    isVoiceBrandVisible: true,
    businessBrandInfo: {
      panelName: '小艺朗读',                       // the panel title the user sees
      panelIcon: $r('app.media.startIcon')
    },
    person: {
      tone: 0, style: 'interaction-broadcast'
    }
  };
  try {
    if (this.context) {
      this.setReadInfoList(readInfoList);
      await TextReader.init(this.context, readerParam);
      this.setActionListener();
    }
  } catch (err) {
    console.error(`TextReader failed to init. Code: ${err.code}, message: ${err.message}`);
  }
}

setActionListener() {
  TextReader.on('stateChange', (state: TextReader.ReadState) => {
    this.onStateChanged(state);
  });
  TextReader.on('requestMore', () => {
    TextReader.loadMore([], true);
  });
}
```

**`keepBackgroundRunning: true` is a request to the speech service, not a
continuous task.** It is the flag most readers of this sample mistake for
background support, and it is why the `KEEP_BACKGROUND_RUNNING` permission is
declared - but the app itself never calls `startBackgroundRunning`, so the
sleep timer's `setTimeout` is still subject to the ordinary rule that timers
are suspended when the UI goes to the background (`HW-11-0028`).

`await` before `setActionListener()` is required: listeners registered before
`init` resolves are attached to a service that does not exist yet.
`style: 'interaction-broadcast'` selects the news-broadcast voice persona,
which is the right one for articles; `tone: 0` is the default speaker.

**Where the duration arrives — `entry/src/main/ets/pages/PlayPage.ets`** (as shipped)

```typescript
async aboutToAppear() {
  emitter.on('TextReader complete', () => {
    this.textReaderUtil.stop();
    this.pagePath.pop();
  });
  this.readInfoList = [{ id: '001', title: { text: '水调歌头.明月几时有', isClickable: true }, /* ... */ }];
  await this.textReaderUtil.init(this.readInfoList);   // load-bearing await, see below

  this.calculateTime();
  this.startCountDown();
  this.textReaderUtil.start();
}

.onReady((context: NavDestinationContext) => {
  this.pagePath = context.pathStack;
  this.navigationParam = this.pagePath.getParamByIndex(this.pagePath.size() - 1) as Record<string, Object>;
  this.timeCountHour = this.navigationParam.hour as number;
  this.timeCountMinute = this.navigationParam.minute as number;
  this.timeCountSecond = this.navigationParam.second as number;
  this.timeCount = (((this.timeCountHour * 60) + this.timeCountMinute) * 60 + this.timeCountSecond) * 1000;
  this.textReaderUtil.setTimeCount(this.timeCount);
})
```

**`aboutToAppear` runs before `onReady`, and `onReady` is where `timeCount` is
computed.** The only reason `start()` does not arm a `setTimeout` with the
initial `-1` - firing immediately and killing playback the instant it begins -
is that `await this.textReaderUtil.init(...)` yields, letting `onReady` run
first. The correctness of this feature rests on an `await` that looks like
ordinary async hygiene. If you refactor `init` to be synchronous, or move the
`await` down, the sleep timer fires at once.

The safe form is to read the navigation parameter in `aboutToAppear` itself
(the stack is reachable through the builder's parameters) or to start playback
from `onReady` after `timeCount` is known, rather than relying on microtask
ordering. `NavDestination`'s reference is explicit that stack operations do not
belong in `aboutToAppear`; the same caution applies to reading its parameters.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.KEEP_BACKGROUND_RUNNING",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
],
"abilities": [
  {
    "name": "EntryAbility",
    "backgroundModes": ["audioPlayback"],
    ...
  }
]
```

Both declarations are inert as shipped (`HW-11-0028`). `KEEP_BACKGROUND_RUNNING`
is a `system_basic`-facing declaration that only takes effect when the app
calls `backgroundTaskManager.startBackgroundRunning` with a matching mode, and
`backgroundModes: ["audioPlayback"]` on the ability is the manifest half of the
same contract. The source contains no `backgroundTaskManager` import.

If background playback is actually wanted, the continuous-task guide's
`AUDIO_PLAYBACK` mode is the correct type ("audio and video playback in the
background"), it must be started while the app is still in the foreground, and
it must be stopped when the sleep timer expires - a continuous task left
running is its own review finding. Otherwise delete both entries and document
that playback ends when the app backgrounds.

`routerMap: '$profile:route_map'` registers `PlayPage` -> `PlayPageBuilder`;
without it `pushPathByName('PlayPage', record, true)` silently fails.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **`setTimeout` and `setInterval` are suspended in the background.** The timer
  reference states that a timer on the UI is suspended when the UI moves to the
  background and expired timers fire in sequence on return. Without a
  continuous task, both the shutdown and the visible countdown stall - so a
  sleep timer built exactly like this only holds while the screen is on the
  page.
- The article list is one hardcoded `ReadInfo` (苏轼's 水调歌头) built in
  `aboutToAppear`; there is no content loading and `loadMore` is answered with
  an empty array.
- `deviceTypes` is `["phone"]`.
- Completion runs the teardown twice (timeout callback plus emitter handler),
  and `TextReader.off('readProgress')` is never called.
- `stopCountDown()` zeroes the three display fields and `pauseCountDown()`
  immediately recomputes them from `timeCount`; on real expiry the zeroes
  stand. Do not reuse `stopCountDown` as a generic "stop the clock".

## Pitfalls

- **`HW-11-0028`** (D/low, confirmed): `ohos.permission.KEEP_BACKGROUND_RUNNING`
  and `backgroundModes: ["audioPlayback"]` are declared but no continuous-task
  API is ever called; the declarations promise background playback the code
  does not implement, while the feature's own `setTimeout` is suspended once
  the app backgrounds. Fix: call
  `backgroundTaskManager.startBackgroundRunning(AUDIO_PLAYBACK)` while reading
  and stop it on completion, or drop the permission and the `backgroundModes`
  entry. This finding is systematic: the AI-recitation sample behind `NEWS-09`
  carries the identical declaration-without-implementation and is filed as a
  second instance of the same id.
- **`HW-11-0010`** (the sibling AI-recitation sample, `NEWS-09`): there
  `requestMore` is registered with a body that evaluates a method reference
  without calling it, and `TextReader` is never released. This sample fixes
  both - `loadMore` is invoked and `stop()` calls `off` and `release`. If you
  are porting between the two, port in this direction.
- **The document's snippet emits before releasing.** `emitter.emit('TextReader
  complete')` ahead of `TextReader.release()` lets the page's handler act on a
  live reader and then release it twice. The shipped code has the order right;
  the document does not.
- **Double teardown at natural expiry.** The timeout callback releases, then
  the emitter handler calls `TextReaderUtil.stop()`, which stops, releases and
  unregisters again. Guard with the unused `isEnd` flag.
- **The start path depends on microtask ordering.** `timeCount` is set in
  `onReady`, which runs after `aboutToAppear`; only the `await` on `init`
  keeps `start()` from arming `setTimeout(..., -1)`.

## References

- `huawei_industry_tree/11_news_reading/docs/26_text_reader.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/text_reader-0000002525367690
- `documentation/harmonyos-guides/08_ai/speech-textreader-guide.md` - the TextReader integration flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/speech-textreader-guide
- `documentation/harmonyos-references/07_ai/speech-textreader-api.md` - `init`, `start`, `pause`, `resume`, `stop`, `release`, `ReaderParam`, `ReadInfo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/speech-textreader-api
- `documentation/harmonyos-references/05_common-capabilities/js-apis-timer.md` - `setTimeout` / `clearTimeout` and background suspension
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-timer
- `documentation/harmonyos-guides/03_application-framework/continuous-task.md` - `AUDIO_PLAYBACK`, `backgroundModes`, `startBackgroundRunning`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/continuous-task
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-texttimer.md` - `TextTimerController` start/pause
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-texttimer
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `NavPathStack`, `pushPathByName`, `onReady` ordering
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `NEWS-09` - AI 朗读, the same `TextReaderUtil` shape without the release path
- `NEWS-21` - the reading-time dashboard, the other timer-driven feature in this industry
