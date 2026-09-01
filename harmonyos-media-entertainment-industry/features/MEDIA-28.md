---
id: MEDIA-28
title: Buffered progress bar - two stacked Sliders showing played position over cached duration
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/28_buffered_progress_bar.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/buffered_progress_bar-0000002351008293
sample: huawei_industry_tree/13_media_entertainment/downloads/BufferedProgressBar.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.MediaKit", "@kit.PerformanceAnalysisKit"]
apis: [display, hilog, media, window]
permissions: [ohos.permission.INTERNET]
min_api: 20
modules: [entry (entry)]
findings: [HW-13-0063, HW-13-0064, HW-13-0065, HW-13-0046, HW-13-0098, HW-13-0099]
status: verified-with-fixes
---

## When to use

Load this card when a player streams over the network and the scrub bar must
show **three things at once**: where playback is, how far the buffer reaches,
and how far the video goes. The pattern is a `Stack` of two `Slider`s sharing
one `max`: an interactive `SliderStyle.OutSet` on top for position, an inert
`SliderStyle.NONE` behind it for the cached range.

It generalises past video. Anything with a "downloaded so far" band under a
"where you are" marker - an audiobook, a large file transfer with a resumable
read head, a paginated log viewer with a fetched window - is the same two-track
control. The state feeding it is the reusable part: one number from
`on('timeUpdate')` and one from `on('bufferingUpdate')` with
`BufferingInfoType.CACHED_DURATION`.

**This sample gets that second number wrong, and so does the document**
(`HW-13-0063`). `CACHED_DURATION` is an absolute report, and both treat it as a
delta. Take the layout from here; take the buffer arithmetic from the pitfall
list.

## Feature checklist

- A landscape-locked page fills the window with an `XComponent` surface and
  plays a network video whose URL comes from a string resource.
- A play glyph overlays the surface while paused; tapping the surface toggles
  play/pause.
- The progress area shows a played track and, behind it, a buffered track, with
  elapsed and total time under them.
- Dragging the top slider seeks; the position label follows the drag and the
  `timeUpdate` handler stops fighting it while `isSwiping` is true.
- When the seek target is beyond the buffered point, playback pauses and a
  "loading" toast appears; when the buffer catches up, playback resumes and a
  "loaded" toast appears.
- If the buffer never catches up within the timeout, the player stops with a
  failure toast.
- Leaving the page pauses; returning resumes only if the app was not paused by
  the user.
- On destroy all seven listeners are unsubscribed and the player is released.

## Architecture

One `entry` module, one page, two helpers. Everything - UI, player, buffer
logic - is in `MainPage.ets`.

```
entry/src/main/ets
├── common/Constants.ets            OPERATE_STATE, BUFFER_INFO_TYPE[], DELAY_TIME=100, CUT_OFF_VALUE=600
├── entryability/EntryAbility.ets   full screen, avoid areas
├── entrybackupability/
├── pages/MainPage.ets              506 lines: XComponent, both Sliders, the AVPlayer, the stall logic
└── utils
   ├── Logger.ets                   hilog wrapper
   └── TimeUtil.ets                 secondToTime
```

The documented tree matches the zip exactly.

**The design decision worth copying** is the choice of *units* for the two
sliders. Both run `min: 0, max: this.durationTime`, where `durationTime` is
**seconds** (`Math.floor(avPlayer.duration / 1000)`), while the player works in
milliseconds throughout. Every value crossing the boundary is converted at the
crossing: `timeUpdate` scales `time` into slider units, `sliderOnChange` scales
back into milliseconds before `seek`. Keeping the widget in whole seconds is
what makes `step: 1` meaningful and stops the buffered track from jittering by
sub-frame amounts.

The decision **worth avoiding** is putting the stall detector inside
`onChange`. A `Slider`'s `onChange` fires on `Begin`, on every `Moving` tick,
and on `End`; the handler creates a fresh 100 ms interval each time, unnamed and
untracked, all of them mutating one shared counter. That is `HW-13-0064`, and
the same page repeats the untracked-interval pattern in `iconOnClick`
(`HW-13-0046`). Stall detection belongs in one interval owned by the component,
started when playback starts and cleared in `aboutToDisappear`.

## Implementation steps

1. **Read the URL from a string resource** in `initFiles`, so the demo URL is
   swappable without a rebuild; toast if it is empty.
2. **Create the player, then set the source only once the surface exists.**
   Assign `avPlayer.url` from `XComponent.onLoad`, not from `aboutToAppear`
   (`HW-13-0065`).
3. **Register all seven listeners in one `setAVPlayerCallback`,** and unregister
   the same seven in `aboutToDisappear` before `release()`.
4. **In `initialized`, set `surfaceId` and `prepare()`; in `prepared`, read
   `duration`,** convert to seconds for both sliders, and `seek(1)` so the first
   frame renders while paused.
5. **Assign, do not accumulate, the buffered position**
   (`HW-13-0063`): `CACHED_DURATION` reports how much playable data is cached
   *now*, so the buffered mark is the current position plus that value. Reset it
   with the rest of the state.
6. **Stack the two sliders** with the interactive one at the higher `zIndex`
   and the buffer one behind at `SliderStyle.NONE`; give them the same `max`.
7. **Guard `timeUpdate` with an `isSwiping` flag** raised on
   `SliderChangeMode.Begin` and lowered on `End`, so the player cannot yank the
   thumb out from under the finger.
8. **Run one stall-detection interval, created on `End` only**, tracked in a
   field and cleared in `aboutToDisappear` (`HW-13-0064`, `HW-13-0046`).

## Verified snippets

All snippets are from `BufferedProgressBar.zip`. Corrected forms are marked.

**The two-track bar — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
Stack() {
  // 播放进度条 — the interactive one, on top
  Slider({
    value: this.currentTime,
    step: 1,
    min: 0,
    max: this.durationTime,
    style: SliderStyle.OutSet
  })
    .trackColor($r('app.color.track_color_show'))
    .selectedColor($r('app.color.select_color_show'))
    .showSteps(false)
    .showTips(false)
    .trackThickness($r('app.float.track_thickness_4'))
    .zIndex(Constants.Z_INDEX[1])
    .onChange((value: number, mode: SliderChangeMode) => {
      this.sliderOnChange(value, mode);
    });

  // 缓冲进度条 — the buffered range, behind, non-interactive
  Slider({
    value: this.currentBufferTime,
    step: 1,
    min: 0,
    max: this.durationTime,
    style: SliderStyle.NONE
  })
    .trackColor($r('app.color.track_color_buffer'))
    .selectedColor($r('app.color.select_color_buffer'))
    .trackThickness($r('app.float.track_thickness_4'))
    .zIndex(Constants.Z_INDEX[0]);
}
.height($r('app.float.height_22'))
```

**`SliderStyle.NONE` is the whole reason this works with two real Sliders**
rather than a Slider over a Progress: it draws the selected track but no block,
so the back layer contributes a filled band and nothing to grab. Identical
`min`/`max`/`step` on both is what keeps the two tracks in register; the only
asymmetry that remains is the padding, which the sample tunes per slider
(`margin_3`/`margin_9` against `margin_13`/`margin_12`) to line the two tracks
up around the top slider's block - a fudge that a shared padding constant would
do better.

`zIndex` decides which one receives the drag. Put the buffer slider on top by
accident and the control becomes inert.

**The buffered position — same file** (corrected, see `HW-13-0063`)

```typescript
// 监听流媒体缓冲状态、缓冲百分比、已缓冲数据预估可播放时长
avPlayer.on('bufferingUpdate', (infoType: media.BufferingInfoType, value: number) => {
  if (infoType === Constants.BUFFER_INFO_TYPE[3]) {          // CACHED_DURATION, in ms
    this.currentBufferTime = this.currentTime + value / 1000; // FIX: absolute, not `+=`
  }
});

reset(sourceFlag?: boolean): void {
  this.isPlay = false;
  this.currentTime = 0;
  this.currentBufferTime = 0;          // FIX: absent from the shipped reset()
  this.durationTime = 0;
  this.durationStringTime = '00:00';
  this.currentStringTime = '00:00';
  this.isFishBuffered = true;
  this.bufferPausedCount = 0;
  this.flag = false;
  if (sourceFlag) {
    this.currentIndex = 0;
    this.initFiles();
    if (this.avPlayer) {
      this.avPlayer.reset();
    }
  }
}
```

**`CACHED_DURATION` answers "how many more milliseconds could I play from
here", and it is reported periodically.** The shipped line is
`this.currentBufferTime += value / 1000`, which sums a running series of
absolute readings: after a handful of reports the value exceeds the duration and
the buffered track is pinned at 100% for the rest of the video. The document
reproduces the same line verbatim in step 6, so a reader who follows the guide
inherits the bug.

The consequence is not only cosmetic. `sliderOnChange` compares
`this.currentTime > this.currentBufferTime` to decide whether the seek target is
buffered; against an inflated number that comparison is always false, so the
stall detection - the sample's actual subject - never fires, and the loading
toasts never appear. `currentBufferTime` is also the one state field
`reset()` forgets, so it survives a source reset into the next video.

**Stall detection — same file** (corrected, see `HW-13-0064`, `HW-13-0046`)

```typescript
private stallIntervalId: number = -1;         // FIX: was a local, recreated per event

sliderOnChange(value: number, mode: SliderChangeMode) {
  if (!this.avPlayer) {
    return;
  }
  if (mode === SliderChangeMode.Begin) {
    this.isSwiping = true;
  }
  this.currentTime = Math.floor(value);
  if (mode !== SliderChangeMode.End && mode !== SliderChangeMode.Click) {
    return;                                   // FIX: Moving ticks change the label and nothing else
  }
  const seekTime: number = value * this.avPlayer.duration / this.durationTime;
  this.currentStringTime = TIME_UTIL.secondToTime(Math.floor(seekTime / 1000));
  this.avPlayer.seek(seekTime, media.SeekMode.SEEK_CLOSEST);
  this.isSwiping = false;

  if (this.stallIntervalId !== -1) {          // FIX: one interval, never stacked
    clearInterval(this.stallIntervalId);
  }
  this.stallIntervalId = setInterval(() => {
    if (this.currentTime > this.currentBufferTime) {
      this.pause();
      if (this.isFishBuffered) {
        this.isFishBuffered = false;
        this.prompt.showToast({ message: $r('app.string.loading') });
      }
      this.bufferPausedCount++;
      if (this.bufferPausedCount >= Constants.CUT_OFF_VALUE) {
        this.prompt.showToast({ message: $r('app.string.loading_failed') });
        this.avPlayer?.stop();
        this.bufferPausedCount = 0;
        clearInterval(this.stallIntervalId);
        this.stallIntervalId = -1;
      }
    } else {
      this.play();
      if (!this.isFishBuffered) {
        this.isFishBuffered = true;
        this.prompt.showToast({ message: $r('app.string.loading_fished') });
      }
      this.bufferPausedCount = 0;
      clearInterval(this.stallIntervalId);
      this.stallIntervalId = -1;
    }
  }, Constants.DELAY_TIME);
}

aboutToDisappear(): void {
  if (this.stallIntervalId !== -1) {          // FIX: nothing clears the timers in the sample
    clearInterval(this.stallIntervalId);
  }
  if (this.avPlayer) {
    this.avPlayer.off('startRenderFrame');
    this.avPlayer.off('timeUpdate');
    this.avPlayer.off('seekDone');
    this.avPlayer.off('speedDone');
    this.avPlayer.off('error');
    this.avPlayer.off('stateChange');
    this.avPlayer.off('bufferingUpdate');
    this.avPlayer.release();
  }
}
```

**The gate on `SliderChangeMode` is the correction that matters.** The shipped
handler runs the interval-creating block on *every* `onChange` call, `Moving`
included, so one ordinary drag spawns dozens of concurrent 100 ms timers. They
all increment the same `bufferPausedCount`, so `CUT_OFF_VALUE` (600 ticks, i.e.
60 s of stall) is reached N times faster and the player stops on what should
have been a routine seek; they all call `play()` while the finger is still
down; and none of them is reachable for `clearInterval` because `intervalID` is
a local in each invocation. `iconOnClick` has the same defect in a smaller form
(`HW-13-0046`): a failed `prepare` leaves a 10 Hz timer running forever, and
each tap adds another.

`isSwiping` is the other half of the drag contract - `timeUpdate` refuses to
write `currentTime` while it is true - and the corrected form still clears it on
`End`/`Click` only, which is correct.

**Source after surface — same file** (corrected, see `HW-13-0065`)

```typescript
XComponent({ id: '', type: XComponentType.SURFACE, libraryname: '', controller: this.xComponentController })
  .onLoad(() => {
    this.xComponentController.setXComponentSurfaceRect({
      surfaceWidth: display.getDefaultDisplaySync().width,
      surfaceHeight: display.getDefaultDisplaySync().height
    });
    this.surfaceID = this.xComponentController.getXComponentSurfaceId();
    if (this.avPlayer && this.sourceUrl !== '') {
      this.avPlayer.url = this.sourceUrl;      // FIX: moved here from createAVPlayer()
    }
  })

// stateChange, unchanged:
case 'initialized':
  this.reset();
  avPlayer.surfaceId = this.surfaceID;
  avPlayer.prepare();
  break;
```

**The `initialized` handler reads `this.surfaceID`, and only `onLoad` writes
it.** The sample sets `avPlayer.url` inside `createAVPlayer`, called from
`aboutToAppear` - so the state callback races the surface. When the callback
wins, `surfaceId` is bound to an empty string and nothing ever re-applies it:
the audio plays over a black rectangle for the rest of the session. Setting the
source from `onLoad` makes the ordering explicit rather than probable. `MEDIA-26`
ships the correct shape and is worth reading alongside this.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" }
]
```

`INTERNET` is `system_grant`: declaring it is enough, there is no runtime
request and no `reason` or `usedScene` needed. It is the only permission the
sample uses - the video is a URL, nothing is written to the gallery or read
from it.

The ability declares `"orientation": "landscape"`, so the page is landscape for
its whole life rather than rotating on a full-screen toggle. The full-screen
button in the control bar is a toast (`app.string.only_show`), as are previous
and next.

`app.string.url` ships empty in the sample - the document says as much
(播放的网络视频地址需用户自行设置, "the network video address must be set by
the user"). `createAVPlayer` toasts `check_url` when it is; supply your own URL
before expecting anything to play.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `CUT_OFF_VALUE` is 600 at a 100 ms tick, i.e. a 60-second stall before the
  player gives up - reasonable only if exactly one interval is running
  (`HW-13-0064`).
- The surface is sized to `display.getDefaultDisplaySync()` in `onLoad` and
  never re-sized, so a window resize or fold leaves the surface at the old
  dimensions.
- `play()` guards on `Constants.OPERATE_STATE.indexOf(state) === 1`, which
  excludes `'playing'` by its index in the array. It works, but it is a
  positional dependency on the order of a constant list; compare the state
  string instead.
- The stall detector only ever runs after a seek. A buffer underrun during
  ordinary playback is not detected at all - `BUFFERING_START` and
  `BUFFERING_END` arrive on the same listener and are ignored.
- No `Constants.BUFFER_INFO_TYPE` entry other than index 3 is handled, so the
  percentage report is available and unused.

## Pitfalls

- **`HW-13-0063`** (B/high, confirmed): `bufferingUpdate` accumulates
  `CACHED_DURATION` with `+=` although each report is absolute, and `reset()`
  clears every field except `currentBufferTime`. The buffered track pegs at
  100% after a few reports and the stall comparison runs on the inflated
  number - the sample's core feature. The document reproduces the same code in
  step 6. Fix: `currentBufferTime = currentTime + value / 1000`, and reset it
  with the rest.
- **`HW-13-0064`** (B/medium, confirmed): `sliderOnChange` creates a new 100 ms
  interval on every event including each `Moving` tick, with the id in a local.
  Dozens of concurrent timers share one `bufferPausedCount`, so the 60 s
  cut-off trips N times faster and calls `avPlayer.stop()` on a routine seek;
  they also `play()` mid-drag and are never cleared. Fix: gate on
  `SliderChangeMode.End`, keep the id in a field, clear it in
  `aboutToDisappear`.
- **`HW-13-0065`** (B/medium, probable): `createAVPlayer` in `aboutToAppear`
  sets `url` immediately, racing `XComponent.onLoad`; when `'initialized'` wins,
  `avPlayer.surfaceId` is bound to an empty string and nothing re-applies it -
  audio with no picture. Fix: set the source from `onLoad` (see `MEDIA-26`).
- **`HW-13-0046`** (B/low, confirmed): `iconOnClick`'s "wait for prepared"
  interval is stored in a local and cleared only when `this.flag` flips, so a
  failed prepare leaves a 10 Hz timer running forever and every tap adds
  another - each of which later calls `play()`. Two other samples repeat it.
  Fix: track the id in a field, clear it on teardown and on error.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - `on('bufferingUpdate')`, `on('timeUpdate')`, `seek`, the state machine
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/04_media/arkts-apis-media-t.md` - `BufferingInfoType.CACHED_DURATION` and what the reported value means
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-t
- `documentation/harmonyos-references/04_media/arkts-apis-media-f.md` - `media.createAVPlayer`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-f
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-slider.md` - `SliderStyle.NONE`, `SliderChangeMode`, `onChange` firing order
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-slider
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - `onLoad`, `getXComponentSurfaceId`, `setXComponentSurfaceRect`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-guides/05_media/using-avplayer-for-playback.md` - the full playback lifecycle including teardown
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/using-avplayer-for-playback
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.INTERNET`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `MEDIA-26` - the correct surface-then-source ordering
