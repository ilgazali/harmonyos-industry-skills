---
id: MEDIA-24
title: Auto-pause and resume around an audio interruption - listen for INTERRUPT_HINT_RESUME and replay only if still in the foreground
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/24_video_player_resume.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_player_resume-0000002299260088
sample: huawei_industry_tree/13_media_entertainment/downloads/VideoPlayerResumeDemo.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.AudioKit", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.LocalizationKit", "@kit.MediaKit", "@kit.PerformanceAnalysisKit"]
apis: ["media.createAVPlayer", "AVPlayer.on('audioInterrupt')", "AVPlayer.on('stateChange')", "AVPlayer.on('timeUpdate')", "audio.InterruptEvent", "audio.InterruptForceType", "audio.InterruptHint", fdSrc, surfaceId, prepare, play, pause, seek, XComponent, XComponentController, setXComponentSurfaceRect, getXComponentSurfaceId, "resourceManager.getRawFd", Slider, SliderChangeMode, "@Observed", "@State", hilog]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-13-0059, HW-13-0005, HW-13-0012, HW-13-0003]
status: verified-with-fixes
---

## When to use

Load this card when **playback has to survive an interruption it did not
cause** - an incoming call, an alarm, a navigation prompt, another app taking
the audio focus. This is the behaviour users notice only when it is missing:
the alarm ends and the video stays frozen, or worse, resumes shouting from a
pocket after the app has been backgrounded.

The mechanism is one `AVPlayer` callback. The system already pauses you when it
takes the focus; the only thing the app must implement is what happens when it
gives the focus back. That arrives as an `audioInterrupt` event with
`forceType === INTERRUPT_SHARE` and `hintType === INTERRUPT_HINT_RESUME`,
meaning "you may resume, your call". The sample's answer - resume immediately
if the app is in the foreground, otherwise record the intent and act on it at
the next `onForeground` - is the correct one and generalises to every media app
regardless of what it plays.

The surrounding scaffolding is also worth reading as a *contrast* to `MEDIA-21`:
an explicit `AVPlayer` bound to an `XComponent` surface, a module-level
singleton manager, and a hand-built transport bar. That gives full control and
costs you a lifecycle you must manage - which this sample, ironically for a
lifecycle demo, does not: `HW-13-0005` and `HW-13-0012` both land here.

## Feature checklist

- The app launches locked to landscape, full screen, with no system bars.
- A rawfile clip renders into an `XComponent` surface and starts paused.
- A bottom control bar carries play/pause, elapsed time, a slider, total time
  and a (decorative) fullscreen button.
- The play button icon reflects the real player state, not the last tap.
- Elapsed time and the slider follow `timeUpdate`.
- Dragging or tapping the slider seeks proportionally.
- An incoming call or alarm pauses playback and the button flips to "play".
- When the interruption ends and the app is in the foreground, playback resumes
  by itself.
- When the interruption ends while the app is in the background, playback does
  **not** resume - it resumes when the user returns.

## Architecture

One `entry` module, split by responsibility rather than by screen.

```
entry/src/main/ets
├── common
│   ├── Constants.ets           layout numbers + SURFACE_WIDTH/HEIGHT
│   ├── VideoPlayState.ets      @Observed: playState, duration, formatted times, sliderValue
│   └── StaticPlayState.ets     isForeground + playStateOnBackGround (NOT @Observed)
├── controller/AVPlayerController.ets   owns the AVPlayer and all three callbacks
├── entryability/EntryAbility.ets       full screen, landscape, foreground pump
├── entrybackupability/EntryBackupAbility.ets
├── manager/VideoPlayerManager.ets      module-level singleton wiring everything together
├── pages/MainPage.ets          @Entry, a Stack of VideoPlayer + PlayControlView
├── util/CommonUtil.ets         secondToTimeStr + logInfo
└── view
    ├── PlayControlView.ets     the transport bar, reads VideoPlayState
    └── VideoPlayer.ets         29 lines: the XComponent and its onLoad
```

The documented tree **omits `common/StaticPlayState.ets`**, which the zip has
and which carries half of the resume logic. `HW-13-0003` records this industry's
habit of hand-writing project trees instead of regenerating them from the zips;
in this doc the error is an omission rather than a case mismatch, but the cause
is the same.

**The design decision worth copying** is the split between the two state
objects. `VideoPlayState` is `@Observed` and holds everything the UI renders -
play state, formatted times, slider value. `StaticPlayState` is a plain class
and holds the two facts the *ability* knows and the UI never draws: whether the
app is in front, and what the playback state was when it left. Keeping the
second out of the observed object means the ability can write it from
`onForeground`/`onBackground` without dirtying a single component.

The seam between them is the whole feature. `AVPlayerController` reads
`staticPlayState.isForeground` to decide whether an INTERRUPT_HINT_RESUME
should play now or be deferred; `VideoPlayerManager.setForeground` reads
`playStateOnBackGround` to decide whether returning to the foreground should
resume. Two flags, one decision each.

**Worth avoiding:** `videoPlayerManager` is a module-level singleton
(`export const videoPlayerManager = new VideoPlayerManager()`) and it never
tears anything down. There is no `release()` anywhere in the project
(`HW-13-0005`) and no `closeRawFd` for the descriptor `initPlayerSource` opens
(`HW-13-0012`). A sample whose entire subject is player lifecycle management
demonstrates the half of the lifecycle that is easy.

## Implementation steps

1. **Create the surface before the player needs it.** `XComponent.onLoad` fires
   when the surface exists; only then call `getXComponentSurfaceId()` and only
   then create the `AVPlayer`. Creating the player first is the race
   `HW-13-0065` records in a sibling sample.
2. **Set `surfaceId` in the `'initialized'` state, then `prepare()`.** The
   player enters `initialized` when `fdSrc` is assigned; that is the only state
   in which the surface may be bound.
3. **Read `duration` in `'prepared'`** and store it - it is `-1` before that
   (`HW-13-0059`).
4. **Register `audioInterrupt` at the same time as `stateChange`,** inside
   `createAVPlayer`'s callback, so no event can arrive unhandled.
5. **Handle only `INTERRUPT_SHARE`.** A `forceType` of `INTERRUPT_FORCE` means
   the system has already acted; there is nothing for the app to do but let
   `stateChange` update the UI.
6. **On `INTERRUPT_HINT_RESUME`, branch on foreground.** Play if in front;
   otherwise record `STATUS_PLAY` into `playStateOnBackGround` and let
   `onForeground` replay it.
7. **Pump foreground state from the ability**, not from the page - a page's
   `aboutToDisappear` does not fire on backgrounding.
8. **Gate the slider on a prepared player** (`HW-13-0059`): `getDuration()` is
   `-1` until `'prepared'`, so an early drag calls `seek()` with a negative
   millisecond value on a player that cannot seek.
9. **Release the player and close the raw fd** in the ability's `onDestroy`
   (`HW-13-0005`, `HW-13-0012`). Neither exists in the sample.

## Verified snippets

All snippets are from `VideoPlayerResumeDemo.zip`. Corrected forms are marked.

**The interruption handler — `entry/src/main/ets/controller/AVPlayerController.ets`** (as shipped)

```typescript
import { audio } from '@kit.AudioKit';

// 设置音频焦点抢占的回调函数  (register the audio-focus preemption callback)
private setInterruptCallback(avPlayer: media.AVPlayer) {
  avPlayer.on('audioInterrupt', async (interruptEvent: audio.InterruptEvent) => {
    if (interruptEvent.forceType === audio.InterruptForceType.INTERRUPT_SHARE) {
      switch (interruptEvent.hintType) {
        case audio.InterruptHint.INTERRUPT_HINT_RESUME:
          CommonUtil.logInfo('INTERRUPT_HINT_RESUME');
          if (this.staticPlayState?.isForeground) {
            avPlayer.play();                                  // in front: resume now
          } else {
            this.staticPlayState?.setPlayStateOnBackGround(PlayStatus.STATUS_PLAY);
          }                                                   // behind: remember, resume on return
          break;
        default:
          break;
      }
    }
  });
}
```

**`INTERRUPT_SHARE` versus `INTERRUPT_FORCE` is the distinction that makes this
short.** `INTERRUPT_FORCE` means the framework has already paused or ducked the
stream - the app is being *told*, and acting on it would double-handle. Only
`INTERRUPT_SHARE` hands the decision back, and `INTERRUPT_HINT_RESUME` is the
system saying "the alarm is over, you may continue". That is why there is no
pause branch here at all: the system's forced pause arrives through
`stateChange` as `'paused'`, which the other callback already routes into
`videoPlayState.stateChangeToPause()`, and the transport button flips on its
own.

**The foreground check is what stops the pocket-shouting bug.** Without it, an
alarm that fires and clears while the user is in a different app resumes the
video with audio into a locked screen. Deferring through
`playStateOnBackGround` costs one enum field and is completely reliable,
because the ability's `onForeground` is guaranteed.

**The other half of the deferral — `entry/src/main/ets/manager/VideoPlayerManager.ets`** (as shipped)

```typescript
// 设置应用是否处于前景  (record whether the app is in the foreground)
public setForeground(isForeground: boolean) {
  if (this.staticPlayState.isForeground !== isForeground) {   // edge-triggered, not level
    this.staticPlayState.isForeground = isForeground;
    if (isForeground && (this.staticPlayState.playStateOnBackGround === PlayStatus.STATUS_PLAY)) {
      this.avPlayerController.play();                          // the deferred resume
    } else {
      this.staticPlayState.setPlayStateOnBackGround(this.videoPlayState.playState);
    }
  }
}
```

driven from the ability:

```typescript
onForeground(): void {
  videoPlayerManager.setForeground(true);
}

onBackground(): void {
  videoPlayerManager.setForeground(false);
}
```

**The `if` on the transition, not on the value, matters.** `onForeground` can
fire more than once for what the user experiences as one return; guarding on
the change means the deferred resume runs at most once. The `else` branch
snapshots the current play state on the way *out*, which is how a video the
user had paused before backgrounding stays paused on return.

Note the ordering constraint this creates: `setContext` is called from
`onCreate` and `setForeground` from `onForeground`, so the manager must
tolerate being asked about foreground state before the player exists - which it
does, because `avPlayerController.play()` is `this.avPlayer?.play()`.

**Binding the surface — `entry/src/main/ets/view/VideoPlayer.ets` and the manager** (as shipped)

```typescript
// VideoPlayer.ets — the whole file's build()
XComponent({
  id: 'VideoPlayer_XComponent',
  type: XComponentType.SURFACE,
  controller: videoPlayerManager.getXComponentController()
})
  .onLoad(() => videoPlayerManager.onXComponentLoad())
```

```typescript
// VideoPlayerManager.ets
public onXComponentLoad() {
  this.xComponentController.setXComponentSurfaceRect({
    surfaceWidth: Constants.SURFACE_WIDTH,      // 2180
    surfaceHeight: Constants.SURFACE_HEIGHT     // 1024
  });
  this.avPlayerController.setSurfaceId(this.xComponentController.getXComponentSurfaceId());
  this.avPlayerController.createAVPlayer(() => {
    this.avPlayerController.initPlayerSource(this.context, this.PLAY_SOURCE);
  });
}
```

```typescript
// AVPlayerController.ets — state machine
avPlayer.on('stateChange', async (state: string) => {
  switch (state) {
    case 'initialized':
      avPlayer.surfaceId = this.surfaceId;      // only legal here
      avPlayer.prepare();
      break;
    case 'prepared':
      this.videoPlayState?.setDuration(avPlayer.duration);
      break;
    case 'playing':
      this.videoPlayState?.stateChangeToPlay();
      break;
    case 'paused':
      this.videoPlayState?.stateChangeToPause();
      break;
    case 'completed':
      this.videoPlayState?.stateChangeToStop();
      break;
  }
});
```

**The ordering here is the part that is easy to get wrong.** The surface must
exist before `getXComponentSurfaceId()` returns anything usable, which is why
everything hangs off `onLoad` rather than off `aboutToAppear`. The player is
then created, `fdSrc` assignment drives it to `'initialized'`, and only in that
state may `surfaceId` be assigned - assign it earlier and it is silently
dropped; assign it later and `prepare()` has already committed to no surface.
`HW-13-0065` records the sibling sample that builds the player in
`aboutToAppear` and can bind an empty surface id it never re-applies.

`SURFACE_WIDTH`/`SURFACE_HEIGHT` are hardcoded 2180×1024 and the ability is
locked to `"orientation": "landscape"` in `module.json5` - the two facts are
related, and both need replacing with measured values for anything shipping to
tablets or 2in1.

**The transport bar, and the seek guard — `entry/src/main/ets/manager/VideoPlayerManager.ets`** (corrected, see `HW-13-0059`)

```typescript
public onSliderChange(value: number, mode: SliderChangeMode) {
  if (this.videoPlayState.getDuration() <= 0) {
    return;                                     // FIX: duration is -1 until 'prepared'
  }
  switch (mode) {
    case SliderChangeMode.Click:
    case SliderChangeMode.Moving:
      this.avPlayerController.seek(Math.floor(this.videoPlayState.getDuration() * value /
      Constants.HUNDRED));
      break;
    case SliderChangeMode.Begin:
    case SliderChangeMode.End:
      break;
  }
}
```

**`VideoPlayState.duration` is initialised to `-1`** and only assigned in the
`'prepared'` branch of `stateChange`. The `Slider` in `PlayControlView` is
always enabled, so a tap during the second or so before `prepare()` completes
computes `Math.floor(-1 * value / 100)` and calls `seek()` with a negative
value on a player still in `initialized`. The guard above is one line and
covers both the negative value and the invalid state.

Note that `setCurrentTime` in `VideoPlayState` already carries the same guard
in the other direction (`if (this.duration > 0)` before computing
`sliderValue`) - the invariant was understood on the read path and forgotten on
the write path.

**Formatting, worth noting for how little it needs — `entry/src/main/ets/util/CommonUtil.ets`** (as shipped)

```typescript
static secondToTimeStr(seconds: number): string {
  let hour = Math.floor(seconds / Constants.SECONDS);
  let minute = Math.floor((seconds - hour * Constants.SECONDS) / Constants.SIXTY);
  let second = seconds - hour * Constants.SECONDS - minute * Constants.SIXTY;
  if (hour > 0) {
    return `${hour.toString().padStart(Constants.TWO, '0')}:${minute.toString().padStart(Constants.TWO, '0')}` +
      `:${second.toString().padStart(Constants.TWO, '0')}`;
  }
  return `${minute.toString().padStart(Constants.TWO, '0')}:${second.toString().padStart(Constants.TWO, '0')}`;
}
```

The hour segment appears only when there is one, which is the right default for
a player: `03:12` for a clip, `01:03:12` for a film. `setCurrentTime` calls
this on every `timeUpdate` but compares the *result* before assigning
(`if (this.currentTime !== time)`), so a component re-render happens once a
second rather than at the callback rate.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. The clip is a
rawfile inside the HAP, read through `resourceManager.getRawFd`, and audio
focus is acquired implicitly by `AVPlayer` - there is no permission for it.

The one configuration line that carries behaviour:

```json5
"abilities": [
  {
    "name": "EntryAbility",
    "srcEntry": "./ets/entryability/EntryAbility.ets",
    "orientation": "landscape"
  }
]
```

paired with `setWindowLayoutFullScreen(true)` and `setWindowSystemBarEnable([])`
in `onWindowStageCreate` - locked landscape, no status bar, no navigation bar.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Locked to landscape** and a hardcoded 2180×1024 surface. On a tablet or a
  resized 2in1 window the surface does not follow the component.
- `videoPlayerManager` is a module-level singleton created at import time. It
  outlives every page, holds the `AVPlayer` and the raw fd for the process
  lifetime, and is not safe if a second ability is added.
- The fullscreen button in `PlayControlView` has no `onClick` - it is
  decoration.
- `AVPlayerController.createAVPlayer` checks `player != null` but ignores the
  `BusinessError` argument; a failed creation is silent.
- Only `INTERRUPT_HINT_RESUME` is handled. `INTERRUPT_HINT_PAUSE` and
  `INTERRUPT_HINT_DUCK` under `INTERRUPT_SHARE` fall through the `default` -
  acceptable for this scenario, wrong for an app that must duck rather than
  pause.
- There is no `AVSession`, so the player does not appear in the system media
  controls and nothing coordinates it with the lock screen.

## Pitfalls

- **`HW-13-0059`** (B/low, confirmed): `onSliderChange` seeks with
  `videoPlayState.getDuration()`, which is `-1` until the `'prepared'` state,
  and the `Slider` is enabled from first paint. Touching it early calls
  `seek()` with a negative millisecond value on an unprepared player. Fix:
  return early unless `getDuration() > 0` (or disable the slider until
  prepared).
- **`HW-13-0005`** (B/medium, confirmed): systematic across five media samples -
  `AVPlayer`/`SoundPool` instances are created and `release()` is never called
  anywhere in the project. `VideoPlayerResumeDemo` is a named instance, and the
  most conspicuous one, since player lifecycle management is the sample's
  entire subject. Native decoder and audio resources are held until the process
  dies. Fix: `avPlayer.release()` in the ability's `onDestroy`.
- **`HW-13-0012`** (B/low, confirmed): systematic across seven media samples -
  `resourceManager.getRawFd` descriptors are never closed with `closeRawFd`.
  `initPlayerSource` here opens `sample.mp4` and drops the descriptor; the
  sample is not among the seven the finding enumerates, but it is the same
  defect. Fix: pair every `getRawFd` with a `closeRawFd` after the player is
  released.
- **`HW-13-0003`** (E/low, confirmed): systematic - this industry's project
  trees are hand-written rather than regenerated from the zips. Here
  `common/StaticPlayState.ets` is present in the zip and missing from the
  document's 工程目录, which hides the class that carries the deferred-resume
  state.
- **Dropped `BusinessError` in `createAVPlayer`** (observation, no HW id): the
  callback's error argument is never inspected, so a creation failure leaves
  `this.avPlayer` undefined and every later `?.` call silently no-ops.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - `createAVPlayer`, the state machine, `fdSrc`, `surfaceId`, `prepare`, `seek`, the `audioInterrupt` event
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/04_media/arkts-apis-audio-e.md` - `InterruptForceType.INTERRUPT_SHARE`, `InterruptHint.INTERRUPT_HINT_RESUME`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio-e
- `documentation/harmonyos-guides/05_media/audio-playback-concurrency.md` - what audio focus is and which events the system sends
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/audio-playback-concurrency
- `documentation/harmonyos-guides/05_media/using-avplayer-for-playback.md` - the full lifecycle, including the `release()` this sample omits
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/using-avplayer-for-playback
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - `XComponentController`, `onLoad`, `getXComponentSurfaceId`, `setXComponentSurfaceRect`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-slider.md` - `SliderChangeMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-slider
- `huawei_industry_tree/13_media_entertainment/docs/24_video_player_resume.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_player_resume-0000002299260088
- `MEDIA-21` - the opposite trade-off: `Video` components with no explicit player at all
