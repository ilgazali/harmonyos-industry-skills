---
id: MEDIA-11
title: Audio output switching without AVSession - AVCastPicker over a bare AVPlayer
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/11_avplayer_audio.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/avplayer_audio-0000002293630521
sample: huawei_industry_tree/13_media_entertainment/downloads/AVPlayerAudio.zip
kits: ["@kit.AVSessionKit", "@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.AudioKit", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.MediaKit", "@kit.PerformanceAnalysisKit"]
apis: [audio, hilog, media, util, window, "media.createAVPlayer", "AVPlayer.fdSrc", AVFileDescriptor, audioRendererInfo, "audio.StreamUsage", "on('stateChange')", "on('timeUpdate')", "on('durationUpdate')", "on('seekDone')", "on('error')", AVCastPicker, customPicker, "resourceManager.getRawFd", Slider, SliderStyle, "@Watch", "@StorageLink", "@StorageProp"]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-13-0032, HW-13-0033, HW-13-0012, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when an app plays audio and the user must be able to **send it
to another speaker without the app adopting AVSession**. `AVCastPicker` is a
drop-in ArkUI component: place it, and the system handles device enumeration
and routing for the local device and non-cast peripherals.

That is the narrow, correct reading of this sample - a player screen with a
device button, a scrub bar and a play/pause toggle, built on `AVPlayer` with
an `fdSrc` from `rawfile`. The generalisation is the split it demonstrates:
routing is a *system* concern that costs one component, while transport and
progress remain the app's. Reach for the full AVSession integration only when
you also need lock-screen controls, media notifications, or true cast to a
remote renderer.

**Do not copy this file's plumbing.** Two defects sit in load-bearing places:
the model constructs its own `UIContext` with `new UIContext()`, so the audio
never loads (`HW-13-0032`), and the background handler flips a UI flag while
its comment claims it stops playback, after which one more tap destroys the
player (`HW-13-0033`). Both fixes are below and both are small.

## Feature checklist

- A player screen: artwork, title, artist, a four-icon action row, a scrub
  bar with elapsed and total time, and previous / play-pause / next.
- The audio is `art.mp3` in `rawfile`, played through `AVPlayer.fdSrc`.
- A device-picker button in the top right that opens the system output-device
  sheet and switches the sink, with a custom icon rather than the default
  glyph.
- The first tap on play starts playback and starts a 1s ticker that pulls the
  current position into the slider.
- Later taps toggle pause and resume.
- Dragging the slider seeks and resumes.
- Elapsed and total times render as `mm:ss` (or `hh:mm:ss`).
- Backgrounding the app stops audio and the button reflects it.

## Architecture

One `entry` module. A model class wraps the player, a view owns the screen,
and the page supplies avoid-area padding.

```
entry/src/main/ets
├── constants/StyleConstants.ets      every numeric literal in the layout
├── entryability/EntryAbility.ets     full-screen, avoid areas + isForeGround -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── model/AVPlayerDemo.ets            AVPlayer lifecycle: create, fdSrc, callbacks, transport
├── pages/MainPage.ets                @Entry - background gradient and safe-area padding
└── views/AudioPlay.ets               the player UI and all @State
```

The documented tree matches the zip.

**The design decision worth copying** is that `AVPlayerDemo` is the only
place that knows `media.AVPlayer` exists. It exposes `play`, `pause`,
`seek`, `switchPlayOrPause` and two plain numbers, `duration` and
`currentDuration`, and the view never touches the player object. That
boundary is what makes the whole state machine below replaceable.

**The design decision worth avoiding** is how state crosses that boundary.
`duration` and `currentDuration` are plain fields on a non-observed class, so
the view cannot react to them - it polls with `setInterval(..., 1000)` and
copies `avPlayer.currentDuration` into its own `@State`. That is why the
scrub bar moves in one-second steps, why `duration` is read exactly once (in
the first click handler, before `durationUpdate` may have fired), and why the
`completed` and `stopped` states never reach the UI at all (`HW-13-0033`).
Make the model an `@Observed` class, or hand it callbacks; do not poll.

## Implementation steps

1. **Take the `UIContext` from the component**, `this.getUIContext()`, and
   pass it into the model - never `new UIContext()` (`HW-13-0032`).
2. **Create the player and set `audioRendererInfo` before the source.** The
   renderer's `StreamUsage` decides focus behaviour and which volume stream
   the audio lands on.
3. **Register every callback before assigning `fdSrc`.** Setting the source is
   what moves the player to `initialized` and starts the state machine.
4. **Resolve the raw file with `resourceManager.getRawFd(name)`** and build
   an `AVFileDescriptor` from its `fd`, `offset` and `length`. The offset and
   length matter: the descriptor points into the HAP, not at a standalone
   file.
5. **Close the descriptor with `closeRawFd(name)`** once the player is
   released (`HW-13-0012`).
6. **Drive preparation from `stateChange`**: `initialized` -> `prepare()`,
   `prepared` -> `pause()` so the screen opens ready but silent.
7. **Propagate `playing`, `paused`, `completed` and `stopped` to the UI**, not
   only to a private `status` string (`HW-13-0033`).
8. **Actually pause in the background hook.** `AppStorage`'s `isForeGround`
   is watched by the view; the watcher must call `pause()`, not only set a
   boolean (`HW-13-0033`).
9. **Place `AVCastPicker` with a `customPicker` builder** to match the app's
   iconography, and give it a width and height - the component has no
   intrinsic size.
10. **Gate the slider's seek on `SliderChangeMode.End`** so a drag issues one
    seek instead of one per frame.

## Verified snippets

All snippets are from `AVPlayerAudio.zip`. Corrected forms are marked.

**Loading the audio - `entry/src/main/ets/model/AVPlayerDemo.ets`** (corrected, see `HW-13-0032`, `HW-13-0012`)

```typescript
import { media } from '@kit.MediaKit';
import { UIContext } from '@kit.ArkUI';
import { audio } from '@kit.AudioKit';
import { common } from '@kit.AbilityKit';

export class AVPlayerDemo {
  private avPlayer: media.AVPlayer | null = null;
  private status: string = '';
  private uiContext: UIContext;                    // FIX: shipped code has `= new UIContext()`
  private rawName: string = '';
  duration: number = 0;
  currentDuration: number = 0;

  constructor(uiContext: UIContext) {              // FIX: injected by the component
    this.uiContext = uiContext;
    this.duration = 0;
    this.currentDuration = 0;
  }

  async avPlayerFdSrcDemo(audio_name: string) {
    let avPlayer: media.AVPlayer = await media.createAVPlayer();
    avPlayer.audioRendererInfo = { usage: audio.StreamUsage.STREAM_USAGE_VOICE_MESSAGE, rendererFlags: 0 };
    this.setAVPlayerCallback(avPlayer);
    let context = this.uiContext.getHostContext() as common.UIAbilityContext;
    let fileDescriptor = await context.resourceManager.getRawFd(audio_name);
    let avFileDescriptor: media.AVFileDescriptor =
      { fd: fileDescriptor.fd, offset: fileDescriptor.offset, length: fileDescriptor.length };
    avPlayer.fdSrc = avFileDescriptor;
    this.rawName = audio_name;                     // FIX: remember it so it can be closed
    this.avPlayer = avPlayer;
  }

  async release() {                                // FIX: absent in the sample
    await this.avPlayer?.release();
    this.avPlayer = null;
    const context = this.uiContext.getHostContext() as common.UIAbilityContext;
    context.resourceManager.closeRawFd(this.rawName);   // FIX: pairs with getRawFd (HW-13-0012)
  }
}
```

**`new UIContext()` is the defect that breaks the sample outright.** A
`UIContext` is a handle to a *window's* UI instance; the only supported way to
obtain one is from a component (`this.getUIContext()`) or from a window
(`window.getUIContext()`). Bare construction yields an instance attached to
nothing, so `getHostContext()` has no host, and the shipped line
`this.uiContext.getHostContext()!` asserts non-null over that. The
`resourceManager` lookup that follows is the only path to the audio - no
file, no playback. The same construction appears in three other media samples
(`DynamicPhoto`, `SplashPage`, `M3U8Download`), and the document copies it
verbatim into step 1.

**`audioRendererInfo` is set before the source and never after.** It is only
settable in the `idle` state, which is why the order in this method is not
stylistic. Note the value the sample chose: `STREAM_USAGE_VOICE_MESSAGE` for
a music player is wrong - it routes onto the voice stream and takes
voice-assistant focus behaviour. `STREAM_USAGE_MUSIC` is what a music player
wants, and it changes what the output-device sheet offers.

`getRawFd` hands back a descriptor into the HAP that stays open for the
process lifetime unless `closeRawFd` is called; seven media samples in this
industry never call it (`HW-13-0012`).

**The state machine - same file** (as shipped)

```typescript
avPlayer.on('durationUpdate', (duration: number) => { this.duration = duration; });
avPlayer.on('timeUpdate', (timeStamp: number) => { this.currentDuration = timeStamp; });

avPlayer.on('stateChange', async (state: string) => {
  switch (state) {
    case 'idle':
      avPlayer.release();
      break;
    case 'initialized':
      avPlayer.prepare();
      break;
    case 'prepared':
      avPlayer.pause();
      break;
    case 'playing':
      this.status = state;
      break;
    case 'paused':
      this.status = state;
      break;
    case 'completed':
      avPlayer.stop();
      break;
    case 'stopped':
      avPlayer.prepare();
      break;
    case 'released':
      break;
  }
});
```

**Read this as the graph it is.** `initialized -> prepare -> prepared ->
pause` is the standard opening: the source is ready and the transport is
armed but nothing is audible. `completed -> stop -> stopped -> prepare` is
the loop-back, and it is why a finished track leaves the player prepared at
position zero rather than dead - a reasonable choice for a single-track
screen, but note that `isPlaying` in the view is never told, so the button
still shows "pause" after the song ends (`HW-13-0033`).

`idle -> release()` is the dangerous edge. `reset()` moves a player to
`idle`, and the `error` handler above calls exactly that - so *any* player
error silently releases the player, permanently. Combined with the background
bug below, one background-and-tap sequence is enough to lose playback for the
rest of the session.

**The background hook - `entry/src/main/ets/views/AudioPlay.ets`** (corrected, see `HW-13-0033`)

```typescript
@Watch('network') @StorageLink('isForeGround') isForeGround: boolean = false;
private avPlayer: AVPlayerDemo = new AVPlayerDemo(this.getUIContext());   // FIX: inject the context

network() {
  // 切换到后台了，音频停止播放  (switched to the background - audio stops)
  if (!this.isForeGround) {
    this.avPlayer.pause();        // FIX: absent in the sample - only the flag was set
    this.isPlaying = false;
  }
}

aboutToDisappear(): void {
  clearInterval(this.timer);
  this.avPlayer.release();        // FIX: absent in the sample
}
```

**The shipped `network()` sets `this.isPlaying = false` and nothing else** -
the comment above it says the audio stops, and the audio does not stop.
`EntryAbility` writes `isForeGround` into `AppStorage` in `onForeground` and
`onBackground`, the `@StorageLink` delivers it, and the `@Watch` fires; the
plumbing is all correct and the handler simply does not use it. The
consequence is worse than an out-of-date icon: with `isPlaying` false and the
player still playing, the next tap takes the resume branch and calls `play()`
on a playing player, which raises an error, which the `error` handler answers
with `reset()`, which lands in `idle`, which the `stateChange` handler answers
with `release()`. Playback is gone until the page is rebuilt.

Compare `MEDIA-12`'s `VideoView.network()`, which does the same thing
correctly - it calls `controller.pause()` and `controller.start()` on the real
controller.

**The picker and the scrub bar - same file** (as shipped, with the seek gate corrected)

```typescript
@Builder
customPickerBuilder(): void {
  Image($r('app.media.play_ic_device'));
}

// in build():
AVCastPicker({
  normalColor: Color.Red,
  customPicker: () => this.customPickerBuilder()
}).width(StyleConstants.WIDTH_24)
  .height(StyleConstants.HEIGHT_24);

Slider({ style: SliderStyle.NONE, value: this.currentDuration, max: this.duration })
  .blockStyle({ type: SliderBlockType.DEFAULT })
  .selectedColor($r('app.color.gray_selected_color'))
  .trackColor($r('app.color.light_gray'))
  .onChange((value: number, mode: SliderChangeMode) => {
    this.currentDuration = value;
    if (mode === SliderChangeMode.End) {     // FIX: sample seeks on every drag frame
      this.avPlayer.seek(value);
      this.avPlayer.play();
      this.isPlaying = true;
    }
  });
```

**`AVCastPicker` is three lines and no session.** It renders a button, opens
the system output-device sheet, and switches the sink; the app writes no
routing code. `customPicker` replaces the default glyph with a `@Builder`,
which is how the button matches the rest of the transport row - and because
the component has no intrinsic size, the explicit `width`/`height` are
required, not decorative. `normalColor` only applies to the default glyph, so
in this sample the `Color.Red` is inert.

The slider is styled `SliderStyle.NONE` - no track ends, just the thumb over
the bar - and it is the only place the sample can seek from, since previous
and next are decorative. Its shipped `onChange` seeks and calls `play()` on
*every* drag callback; gating on `SliderChangeMode.End` issues one seek when
the finger lifts and avoids hammering the player during the drag.

## Permissions & config

**None.** The sample declares no `requestPermissions`, and none are needed:
`AVCastPicker` performs local output switching through the system UI, and the
audio is a `rawfile` inside the HAP.

Two configuration points are load-bearing:

- `EntryAbility.onCreate` forces light mode with
  `setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT)`, because
  the screen is painted over a fixed dark gradient with hardcoded white text.
- `onForeground` / `onBackground` write `isForeGround` into `AppStorage`; that
  key is the only lifecycle signal the view gets.

Real background playback would additionally need a `ohos.permission.KEEP_BACKGROUND_RUNNING`
declaration and a long-running task - out of scope here, since this sample is
supposed to *stop* in the background.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`
  with `strictMode.caseSensitiveCheck` on, so the constraint section is
  accurate here - unlike the three docs in `HW-13-0004`.
- **Switching range is the local device and non-cast peripherals**, and the
  local device cannot be split into speaker versus earpiece. The document
  states this; it is a property of the picker without AVSession, not a bug.
- Without AVSession there are no lock-screen or notification controls and no
  remote cast target. If either is needed, this card is the wrong pattern.
- Previous and next are static images with no handlers; there is one track.
- Progress resolution is one second, from a `setInterval` poll rather than
  from `timeUpdate`.
- `duration` is captured once, inside the first play tap. If
  `durationUpdate` has not fired by then, the slider's `max` stays `0` and the
  thumb cannot move.

## Pitfalls

- **`HW-13-0032`** (B/medium, probable): `AVPlayerDemo` builds its context
  with `new UIContext()` - a detached instance whose `getHostContext()` has no
  host, so `resourceManager.getRawFd` never resolves and no audio loads. The
  same construction appears in `DynamicPhoto`, `SplashPage` and
  `M3U8Download`, and in step 1 of the document. Fix: inject
  `this.getUIContext()` from the component (or use the ability context).
- **`HW-13-0033`** (B/medium, confirmed): the `isForeGround` watcher only
  flips `isPlaying` although its comment says audio stops, and `completed` /
  `stopped` are never propagated to the UI. After backgrounding, the next tap
  calls `play()` on a playing player, and the error path
  (`error -> reset -> idle -> release`) destroys it. Fix: pause in the
  background hook and push player states to the view.
- **`HW-13-0012`** (B/low, confirmed): `getRawFd('art.mp3')` is never paired
  with `closeRawFd`; the descriptor leaks for the process lifetime. Seven
  media samples share this. Fix: close it after releasing the player.
- **`STREAM_USAGE_VOICE_MESSAGE` for music**: the renderer is declared as a
  voice stream, which puts it on the wrong volume control and gives it
  voice-call focus semantics. Use `STREAM_USAGE_MUSIC`.
- **The player is never released on page teardown**: `aboutToDisappear` clears
  the ticker only. The same class of omission is tracked across five other
  media samples in `HW-13-0005`.

## References

- `documentation/harmonyos-guides/05_media/using-avplayer-for-playback.md` - the AVPlayer audio lifecycle and the `fdSrc` flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/using-avplayer-for-playback
- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - `AVPlayer` states, `on('stateChange')`, `on('timeUpdate')`, `AVFileDescriptor`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/04_media/ohos-multimedia-avcastpicker.md` - `AVCastPicker`, `customPicker`, `normalColor`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ohos-multimedia-avcastpicker
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-slider.md` - `SliderStyle`, `blockStyle`, `SliderChangeMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-slider
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - where a `UIContext` legitimately comes from
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-references/02_application-framework/js-apis-rawfiledescriptor.md` - `RawFileDescriptor`'s `fd`/`offset`/`length`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-rawfiledescriptor
- `MEDIA-10` - the lyric view that belongs on top of this player
- `MEDIA-12` - the same `isForeGround` pattern, implemented correctly
