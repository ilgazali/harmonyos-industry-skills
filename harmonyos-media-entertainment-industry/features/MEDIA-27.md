---
id: MEDIA-27
title: Scrolling audio waveform - two canvases leapfrogging under an animator, one bar per PCM window
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/27_audio_waveform_animation.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio_waveform_animation-0000002349412093
sample: huawei_industry_tree/13_media_entertainment/downloads/AudioVisualization.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.AudioKit", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [abilityAccessCtrl, audio, bundleManager, common, display, fileIo, fs, hilog, util, window]
permissions: [ohos.permission.MICROPHONE]
min_api: 20
modules: [entry (entry)]
findings: [HW-13-0061, HW-13-0062, HW-13-0039, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when a screen must show **live audio level as a waveform that
scrolls, endlessly, at a fixed speed** - a voice recorder, a walkie-talkie
push-to-talk view, a call-recording indicator, a music app's input meter. The
pattern has two halves that are worth separating in your head: how the level is
measured (RMS over a PCM window, converted to a normalised bar height), and how
an unbounded scroll is drawn without an unbounded canvas.

The second half is the reusable trick. Two `Canvas` components of the same
width sit side by side in a `Row`; an `AnimatorResult` drives both leftwards
each frame; when the leading one has fully left the screen the two swap roles
and it is repositioned to the right of the other. The drawing code only ever
targets "the forward canvas" at a local x. The same idea covers a scrolling ECG
trace, a live network-throughput strip, any infinite chart.

**Read `HW-13-0061` before you copy the level maths.** The measurement half of
this sample is wrong in a way that is easy to inherit and hard to notice: the
bars are simply too short, uniformly, and the sample still "works".

## Feature checklist

- The main page lists previously recorded files, newest first, with name,
  modification date and a duration derived from file size.
- A record button opens a bottom sheet, starts `AudioCapturer`, and draws a
  live waveform that scrolls right-to-left at a constant rate.
- Bar height tracks input level: loud in, tall bars; silence, a flat line.
- Recorded PCM is written to a temp file and renamed to 录音N (recording N) on
  stop, then the list refreshes.
- Tapping a list row opens a play sheet with the same waveform, driven by the
  renderer instead of the capturer, plus a position slider and elapsed/total
  times.
- Play/pause inside the sheet pauses the animation as well as the audio.
- A swipe-left on a row reveals a delete button that unlinks the file.
- Without the microphone permission, the record button shows a toast instead of
  starting.

## Architecture

One `entry` module. One page, two symmetric manager classes, one shared
constants file.

```
entry/src/main/ets
├── common
│  ├── Constants.ets              MIN_DB=-90, VOLUME_MAX=32768, LINE_SPACE=6, 48 kHz/mono/S16LE
│  ├── Objects.ets                RecordFileInfo { name, mtime, size }
│  └── utils.ets                  checkPermission / requestPermissions, size->duration, state->string
├── entryability/EntryAbility.ets full screen; avoid areas -> AppStorage
├── entrybackupability/
├── pages/Index.ets               746 lines: the list, both sheets, both animators, both draw loops
└── viewmodel
   ├── AudioCapturerManager.ets   AudioCapturer + readData -> file + level accumulator
   └── AudioRendererManager.ets   AudioRenderer + writeData <- file + level accumulator
```

The documented tree matches the zip exactly.

**The design decision worth copying** is the symmetry between the two managers.
`AudioCapturerManager` and `AudioRendererManager` expose the same shape -
`initX`, `startX`, `stopX`, `releaseX`, `xState()`, and crucially the identical
`calculateDecibel()` - so `Index.ets` can run one waveform implementation
against either. `drawOnRecord()` and `drawOnPlay()` differ by exactly one
expression (which manager to ask for a level) and one constant in the x offset.
Both are state-guarded the same way: every method checks the current
`audio.AudioState` and throws with a named state on a wrong transition, which is
the right discipline for an API whose calls are only legal from certain states.

The decision **worth avoiding** is that `Index.ets` is one 746-line struct
holding two dialogs, two animators, two canvases and the file list. The two
`onWillAppear` blocks that build the animators are near-identical copies; the
two dialog builders repeat the canvas pair verbatim. Extract the animator setup
and the canvas pair into a component and the page halves in size.

## Implementation steps

1. **Fix the audio format once** in `Constants`: mono, 48 kHz, `S16LE`. Both
   the capturer and the renderer use the same `AudioStreamInfo`, which is what
   makes the raw PCM file playable back with no header.
2. **Check the permission on entry and request it if denied**; check it again
   at the record tap and toast rather than starting (`clickRecord`).
3. **Accumulate the level inside the data callback**, not on a timer:
   `on('readData')` for capture, `on('writeData')` for playback. Sum the
   squares of the normalised samples and count them.
4. **Convert to a normalised bar height in `calculateDecibel()`,** taking the
   square root of the mean square before the log (`HW-13-0061`), clamping to
   `[MIN_DB, 0]` and mapping onto `[0, 1]`. Reset the accumulator on read - the
   function is a consuming read, called once per drawn bar.
5. **Zero (or bound) the tail of the render buffer** on the final partial chunk
   (`HW-13-0062`).
6. **Build the animator in `onWillAppear`, start it in `onDidAppear`,** and
   cancel it in `onWillDisappear`. Starting it only after the renderer or
   capturer actually started keeps the waveform and the audio in step.
7. **Draw one bar per `LINE_SPACE` of travel,** not per frame: compare the
   animator value against the last drawn x.
8. **Swap the two canvases in `onFinish`** and re-`play()` the animator, giving
   an endless scroll from a 30 s animation.
9. **Guard `closeSync` on a file that may never have been opened**
   (`HW-13-0039`).

## Verified snippets

All snippets are from `AudioVisualization.zip`. Corrected forms are marked.

**Measuring the level — `entry/src/main/ets/viewmodel/AudioCapturerManager.ets`** (corrected, see `HW-13-0061`)

```typescript
private sampleValCnt: number = 0;
private sampleValSum: number = 0;

calculateDecibel(): number {
  if (this.sampleValCnt === 0) {
    return 0;
  }
  let meanSquare = this.sampleValSum / this.sampleValCnt;      // mean of val², not an RMS
  let rms = Math.sqrt(meanSquare);                             // FIX: take the root
  let db = Math.max(Constants.MIN_DB, Math.min(0, 20 * Math.log10(rms)));
  this.sampleValCnt = 0;
  this.sampleValSum = 0;

  return (db + Math.abs(Constants.MIN_DB)) / Math.abs(Constants.MIN_DB);
}

this.capturer.on('readData', (buffer: ArrayBuffer) => {
  let options: WriteOptions = { offset: this.writeOffset, length: buffer.byteLength };
  fsUtils.writeSync(this.recordFile?.fd, buffer, options);
  this.writeOffset += buffer.byteLength;
  AppStorage.setOrCreate('RWOffset', this.writeOffset);

  let samples = new Int16Array(buffer);                        // S16LE maps straight onto Int16Array
  for (let i = 0; i < samples.length; i++) {
    let val = samples[i] / Constants.VOLUME_MAX;               // normalise to [-1, 1]
    this.sampleValSum += val * val;
    this.sampleValCnt += 1;
  }
});
```

**The variable named `rms` in the shipped code is not an RMS.** It is the mean
of the squares; the root is never taken, and `20 * log10` is then applied to
it. Since `20·log₁₀(x²) = 2 · 20·log₁₀(x)`, every level is doubled in dB: a
true −30 dB signal is computed as −60 dB. With `MIN_DB = -90` the normalised
output is `(db + 90) / 90`, so that signal draws a bar of 0.33 instead of 0.67 -
half height - and anything below −45 dB clamps to the floor and disappears
entirely. Either take the square root as above, or equivalently use
`10 * Math.log10(meanSquare)`. `AudioRendererManager.calculateDecibel()` is the
same function, character for character, and `PCMAudioEdit`'s `AudioRecorder`
repeats it a third time.

Two details in the callback are right and worth keeping. The level is
accumulated in the audio callback, so it is measured over *every* sample, not
sampled at draw time; and `calculateDecibel()` zeroes the accumulator as it
reads, which makes each drawn bar the mean over exactly the interval since the
previous bar - no smoothing, no double counting.

**Feeding the renderer — `entry/src/main/ets/viewmodel/AudioRendererManager.ets`** (corrected, see `HW-13-0062`)

```typescript
this.renderer.on('writeData', (buffer: ArrayBuffer) => {
  let lastLen = this.fileSize - this.readOffset;
  let readLen = lastLen >= buffer.byteLength ? buffer.byteLength : lastLen;
  fsUtils.readSync(this.playFile?.fd, buffer, { offset: this.readOffset, length: readLen });

  if (readLen < buffer.byteLength) {                                  // FIX: the final partial chunk
    new Uint8Array(buffer).fill(0, readLen);                          // FIX: zero the stale tail
  }

  this.readOffset += readLen;
  AppStorage.setOrCreate('RWOffset', this.readOffset);
  if (this.readOffset >= this.fileSize) {
    this.readOffset = 0;
  }

  let samples = new Int16Array(buffer);
  for (let i = 0; i < samples.length; i++) {
    let val = samples[i] / Constants.VOLUME_MAX;
    this.sampleValSum += val * val;
    this.sampleValCnt += 1;
  }
});
```

**`writeData` hands you a fixed-size buffer and renders all of it.** At the end
of the file `readLen` is short, the read fills only the front, and the bytes
behind it are whatever the previous callback left there - so the last fragment
of every playback ends in an echo of audio from ~one buffer earlier. Because
`readOffset` then wraps to 0 and the sample loops, that garbage tail recurs on
every pass. Zeroing from `readLen` costs one line. The same shape appears in
`PCMTranscode`, `demo_AudioSession` and `demo_HttpAudioRender`.

Note also that the level loop runs over the *whole* buffer, tail included, so
the stale bytes also corrupt the final bar's height - fixing the tail fixes
both symptoms.

**The scrolling pair — `entry/src/main/ets/pages/Index.ets`** (as shipped)

```typescript
private context0: CanvasRenderingContext2D = new CanvasRenderingContext2D({ antialias: true });
private context1: CanvasRenderingContext2D = new CanvasRenderingContext2D({ antialias: true });

Scroll() {
  Row() {
    Canvas(this.context0)
      .width(2 * this.dWidth)
      .offset({ x: this.offsetX0 })
      .aspectRatio(Constants.CANVAS_ASPECT_RADIO)
      .onReady(() => { this.context0.reset(); })
      .onVisibleAreaChange([0.01], (isVisible: boolean) => {
        if (!isVisible) {
          this.context0.reset();          // wipe a canvas the moment it is fully off-screen
        }
      })
    Canvas(this.context1)
      .width(2 * this.dWidth)
      .offset({ x: this.offsetX1 })
      .aspectRatio(Constants.CANVAS_ASPECT_RADIO)
      .onReady(() => { this.context1.reset(); })
      .onVisibleAreaChange([0.01], (isVisible: boolean) => {
        if (!isVisible) {
          this.context1.reset();
        }
      })
  }
}
.scrollBar(BarState.Off)
.scrollable(ScrollDirection.None)          // scrolled by offset, never by the user
```

**Three attributes carry the design.** Each canvas is `2 * dWidth` wide - two
screens - so one full animation cycle (`begin: 0, end: 2 * this.dWidth`) walks
exactly one canvas past the viewport. `offset({ x })` is what moves them, not
the `Scroll`: the container is `ScrollDirection.None` and exists only to clip.
And `onVisibleAreaChange([0.01], ...)` with a `reset()` is the garbage
collection - a canvas that has just left the screen is cleared before it is
repositioned on the right, so it comes back blank instead of showing the
previous cycle's bars.

**The animator loop — same file, `clickRecord`'s `onWillAppear`** (as shipped)

```typescript
let options: AnimatorOptions = {
  duration: Constants.ANIMATOR_DURATION,   // 30 s
  easing: 'linear', delay: 0, fill: 'forwards', direction: 'normal', iterations: 1,
  begin: 0, end: 2 * this.dWidth
};
this.animator = this.getUIContext().createAnimator(options);

this.animator.onFrame = (value: number) => {
  let diff = value - this.animatorValue;
  this.animatorValue = value;
  this.offsetX0 -= diff;                   // both canvases move together
  this.offsetX1 -= diff;
  if (Math.round(this.animatorValue - this.drawXPos) >= Constants.LINE_SPACE) {
    this.drawOnRecord();                   // one bar per LINE_SPACE of travel
  }
};
this.animator.onFinish = () => {
  if (Math.floor(this.animatorValue) !== Math.floor(2 * this.dWidth)) {
    return;                                // a cancelled run also fires onFinish
  }
  this.drawXPos = 0;
  this.animatorValue = 0;
  this.forwardCanvas = 1 - this.forwardCanvas;
  if (this.forwardCanvas === 1) {
    this.offsetX0 = 2 * this.dWidth;       // canvas 0 has left: re-hang it right of canvas 1
  } else {
    this.offsetX1 = 0;                     // canvas 1 has left: it is now the leading one
  }
  this.animator?.play();                   // restart: the scroll never ends
};
this.animator.setExpectedFrameRateRange({ min: 30, max: 60, expected: 60 });
```

**`easing: 'linear'` is not a stylistic choice** - the waveform's x axis is
time, so any curve would distort the trace. `iterations: 1` plus an explicit
`play()` in `onFinish` is used instead of `iterations: -1` because each cycle
must do work between runs (swap the canvases, reset `drawXPos`); an infinite
iteration count would never give you that seam.

The `Math.floor(this.animatorValue) !== Math.floor(2 * this.dWidth)` guard is
the subtle line: `onFinish` also fires when the dialog closes and the animator
is cancelled mid-cycle, and without the guard that would relaunch the animation
on a dismissed sheet. Drawing is decoupled from the frame rate by comparing
travelled distance against `drawXPos`, so the bar spacing stays constant
whether the device renders at 30 or 120 Hz.

**Closing what was opened — `AudioRendererManager.stopRenderer`** (corrected, see `HW-13-0039`)

```typescript
await this.renderer.stop();

if (this.playFile) {                       // FIX: playFile is only set when startRenderer got a path
  fsUtils.closeSync(this.playFile.fd);
  this.playFile = undefined;
}
```

`startRenderer(path?)` only opens a file when a path is passed - the resume
button calls it with none - so `closeSync(this.playFile?.fd)` can be handed
`undefined` and throw out of the cleanup path, masking whatever caused the stop.
The same double-fault shape appears in five samples across this industry.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.MICROPHONE",
    "reason": "$string:microphone_reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
]
```

`MICROPHONE` is `user_grant`, so `reason` and `usedScene` are mandatory and the
reason string must exist in `resources/base/element/string.json`. `when: "inuse"`
is correct here: recording only happens with the sheet open and the app in the
foreground; background capture would need a continuous task on top of the
permission.

The flow is the right one: `checkPermission` in `aboutToAppear`, request only
if denied, and then a **second** check at the record tap
(`clickRecord`) that toasts `msg_permissionOnSetting` rather than calling
`initCapturer`. Never assume the answer from launch is still true.

Playback needs no permission - the renderer reads the app's own `filesDir`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The recordings are **headerless PCM** (48 kHz mono S16LE). They play only
  through this app's renderer configured identically; nothing else on the
  device will open them, and the "duration" shown is arithmetic on file size
  (`toTimeStr` divides by `48000 * 2 * 1`), not a container field.
- `dWidth` and `dHeight` are captured once in `aboutToAppear` from
  `display.getDefaultDisplaySync()`. The canvas geometry and the sheet size do
  not follow a window resize or a fold, so this layout is fixed-window only.
- The naming loop in `stopCapturer` scans for the first free 录音N and iterates
  `i <= files.length`; delete a middle file and the next recording reuses its
  name.
- The search box on the main page is a bare `TextInput` with no handler - the
  list is never filtered.
- `ForEach` keys rows on `JSON.stringify(info)`, so any change to name, mtime
  or size rebuilds the row rather than updating it.

## Pitfalls

- **`HW-13-0061`** (B/medium, confirmed): `calculateDecibel()` applies
  `20 * log10` to the **mean square** without taking the square root, doubling
  every level in dB - a true −30 dB signal computes as −60 dB, bars are
  uniformly far too short and quiet audio clamps to the floor early. Present in
  `AudioCapturerManager`, `AudioRendererManager` and `PCMAudioEdit`'s
  `AudioRecorder`. Fix: `Math.sqrt` the mean square first, or use
  `10 * Math.log10`.
- **`HW-13-0062`** (B/low, confirmed): the final partial chunk in
  `on('writeData')` leaves the tail of the render buffer holding the previous
  callback's bytes, which are then rendered - a garbage/echo fragment at the end
  of every playback pass, and a corrupted last bar. Four samples share the
  shape. Fix: zero from `readLen`, or render only `readLen` bytes.
- **`HW-13-0039`** (B/medium, confirmed): `stopRenderer` calls
  `fsUtils.closeSync(this.playFile?.fd)` on a file that `startRenderer` may
  never have opened, turning a routine stop into a `TypeError` from the cleanup
  path. Fix: `if (this.playFile) closeSync(this.playFile.fd)`.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-audio-audiocapturer.md` - `on('readData')` and the capturer state machine
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio-audiocapturer
- `documentation/harmonyos-references/04_media/arkts-apis-audio-audiorenderer.md` - `on('writeData')` and the obligation to fill the whole buffer
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio-audiorenderer
- `documentation/harmonyos-guides/05_media/using-audiocapturer-for-recording.md` - the create → start → stop → release sequence
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/using-audiocapturer-for-recording
- `documentation/harmonyos-guides/05_media/using-audiorenderer-for-playback.md` - the playback counterpart
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/using-audiorenderer-for-playback
- `documentation/harmonyos-references/02_application-framework/js-apis-animator.md` - `AnimatorOptions`, `onFrame`, `onFinish`, `setExpectedFrameRateRange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-animator
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - `CanvasRenderingContext2D`, `onReady`, `reset`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-promptaction.md` - `openCustomDialog` and its `onWillAppear` / `onDidAppear` / `onWillDisappear` hooks
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-promptaction
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `ohos.permission.MICROPHONE`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
