---
id: MEDIA-42
title: Pause on headset disconnect - AVSession commands for wear detection plus deviceChange for the physical unplug
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/42_demo_audio_session.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/demo_audio_session-0000002668471732
sample: huawei_industry_tree/13_media_entertainment/downloads/demo_AudioSession.zip
kits: ["@kit.AVSessionKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.LocalizationKit", "@kit.MediaKit", "@kit.PerformanceAnalysisKit"]
apis: ["avSession.createAVSession", "AVSession.activate", "AVSession.on('play')", "AVSession.on('pause')", "AVSession.on('stop')", "AVSession.setAVPlaybackState", "AVSession.setAVMetadata", "AVSession.deactivate", "AVSession.destroy", "audio.getAudioManager", "AudioManager.getRoutingManager", "AudioRoutingManager.on('deviceChange')", "AudioRoutingManager.off('deviceChange')", "AudioRoutingManager.getDevicesSync", "audio.createAudioRenderer", "AudioRenderer.on('writeData')", "media.createAVPlayer", "AVPlayer.fdSrc", "resourceManager.getRawFdSync", "resourceManager.getRawFd", "fs.readSync", "Video", "VideoController", "XComponent", "promptAction.openToast", "IjkMediaPlayer"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-13-0093, HW-13-0094, HW-13-0095, HW-13-0062, HW-13-0012, HW-13-0097, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when **audio must not keep playing out of the loudspeaker after
the headphones go away**. Pulling a wired jack, disconnecting a Bluetooth
headset, or taking an earbud out of the ear all route the stream back to the
device speaker, and the user gets their podcast, their call recording or their
private playlist broadcast to the room. Every media app needs this and most
get it half right.

The pattern has two halves because the platform gives you two different
signals. Wear detection - taking the earbud out - is handled by the system: it
issues a `pause` command through the media control centre, so the app only
needs an `AVSession` with a `pause` listener wired to its player. A physical
disconnect is not a command; it is a routing change, so the app subscribes to
`AudioRoutingManager`'s `deviceChange` event and pauses itself. The sample
implements both in two small utility classes and then repeats the same wiring
across five different players to show that the mechanism is
player-independent.

That last point is what generalises. `HeadsetMonitor` and `AVSessionManager`
never touch a player type: they emit enum events and take callbacks. Anything
that plays - `AudioRenderer`, `AVPlayer`, the `Video` component, an
`XComponent` surface, a third-party `IjkMediaPlayer` - plugs in with the same
six lines.

**Read `HW-13-0094` before adopting it.** This is a demo about keeping the
control centre in sync with playback, and it never handles audio interruption -
the single most common way those two get out of sync.

## Feature checklist

- A home page lists five players; each opens a page that plays a bundled asset.
- Each player page registers an `AVSession` on entry, activates it, and shows
  the track title and artist in the system media control centre.
- Play, pause and stop issued from the control centre drive the actual player.
- Every local play/pause/stop pushes an `AVPlaybackState` back to the session,
  so the control centre never shows a stale transport state.
- On entry the app queries the currently connected output devices and shows a
  Bluetooth and a wired indicator, ignoring the built-in earpiece and speaker.
- Connecting a headset shows a toast and lights the corresponding indicator.
- Disconnecting a Bluetooth or a wired headset shows a toast, clears the
  indicator, **and pauses playback**.
- Taking off a headset that supports wear detection pauses playback through the
  control centre's `pause` command; putting it back on resumes through `play`.
- Leaving a player page removes the listeners, deactivates and destroys the
  session, and releases the player.

## Architecture

One `entry` module. Two utility classes carry the whole feature; the five
pages are variations on one another.

```
entry/src/main/ets
├── entryability/EntryAbility.ets
├── entrybackupability/
├── pages
│  ├── Index.ets                      home: five @Builder buttons, router.pushUrl
│  ├── SimpleAudioPlayer.ets          AudioRenderer + raw PCM (world.pcm), writeData callback
│  ├── SimpleAVPlayer.ets             AVPlayer audio (world.wav) via fdSrc
│  ├── SimpleVideoPlayer.ets          AVPlayer + XComponent surface (mp4)
│  ├── SimpleIjkPlayer.ets            third-party @ohos/ijkplayer (mp4), own 300 ms progress timer
│  └── SimpleNativeVideoPlayer.ets    the built-in Video component + VideoController
└── utils
   ├── HeadsetMonitor.ets             AudioRoutingManager deviceChange -> 4 enum events
   └── AVSessionManager.ets           session lifecycle + play/pause/stop commands + state push
```

The documented tree matches the zip. `compatibleSdkVersion` is `6.0.0(20)`,
matching the document's stated floor. The one third-party dependency is
`@ohos/ijkplayer` `^2.0.9`, used by a single page.

**The design decision worth copying** is the shape of the two utilities: both
are plain classes with `setCallback` / `init` / `release`, both emit an enum
rather than raw platform types, and neither imports anything from `pages/`.
`HeadsetMonitor` collapses `DeviceChangeAction` into four events
(`BLUETOOTH_CONNECTED`, `BLUETOOTH_DISCONNECTED`, `NON_BLUETOOTH_CONNECTED`,
`NON_BLUETOOTH_DISCONNECTED`); `AVSessionManager` collapses the control centre
into three (`PLAY`, `PAUSE`, `STOP`). A page then contains exactly one switch
per utility, and the five pages prove the abstraction holds across five
unrelated playback APIs.

`HeadsetMonitor` also does something easy to forget: `initAudioRouting` calls
`queryInitialDeviceState()` **before** registering the listener, so a headset
that was already connected when the page opened produces a connect event too.
Event subscriptions only tell you about changes; the initial state has to be
polled.

**The decision worth avoiding** is the duplication itself. `initHeadsetMonitor`
and `initAVSession` are copied verbatim into all five pages, along with the
`DeviceStatus` builder - so a fix to the routing logic (or the addition of the
interrupt handling `HW-13-0094` calls for) has to be made five times. A base
component or a single `PlaybackController` holding both utilities and exposing
`play`/`pause`/`stop` would have collapsed roughly 400 duplicated lines.

## Implementation steps

1. **Create the session with the right type**: `avSession.createAVSession(context, tag, 'audio' | 'video')`.
   The sample uses `'audio'` for the two audio players and `'video'` for the
   three video ones; the type drives what the control centre renders.
2. **Register the command listeners before `activate()`.** The session becomes
   visible to the control centre on activation, and a command can arrive
   immediately after.
3. **Handle `play`, `pause` and `stop` by driving the real player**, then let
   the player's own path push the state back. Do not update the UI directly
   from the command handler and the player separately.
4. **Push `AVPlaybackState` on every local transport change** -
   `PLAYBACK_STATE_PLAY`, `_PAUSE`, `_STOP` - so the control centre and the app
   never disagree. Compare state *values* when deduplicating, not object
   references (`HW-13-0095`).
5. **Set `AVMetadata`** (`assetId`, `title`, `artist`) or the control centre
   shows an unlabelled card.
6. **Query connected output devices once on entry** with
   `getDevicesSync(audio.DeviceFlag.OUTPUT_DEVICES_FLAG)`, skipping `EARPIECE`
   and `SPEAKER`, then register the `deviceChange` listener.
7. **Read `deviceChanged.type`**: `0` is connect, `1` is disconnect. Classify
   the device with `deviceDescriptors[0].deviceType` against
   `BLUETOOTH_A2DP` / `BLUETOOTH_SCO`.
8. **Pause on both disconnect branches** - wired and Bluetooth. This is the
   feature; the connect branches only update indicators.
9. **Register `audioInterrupt` on every player** and drive the same pause path
   from it (`HW-13-0094`). Without it a phone call or another app's stream
   leaves `isPlaying` and the session state saying PLAY while nothing is
   audible.
10. **Clamp raw PCM reads to what is left of the asset** and stop rendering at
    end of file, resetting `isPlaying` and the session state
    (`HW-13-0093`, `HW-13-0062`).
11. **Tear down in `aboutToDisappear`**: stop and release the player, close the
    raw file descriptor with `closeRawFd` - not `fs.closeSync` (`HW-13-0012`) -
    `off('deviceChange')`, then `off('play'/'pause'/'stop')`, `deactivate()`
    and `destroy()`.

## Verified snippets

All snippets are from `demo_AudioSession.zip`. Corrected forms are marked.

**The control-centre commands — `entry/src/main/ets/utils/AVSessionManager.ets`** (as shipped)

```typescript
async init(context: Context, tag: string, type: avSession.AVSessionType): Promise<void> {
  try {
    this.session = await avSession.createAVSession(context, tag, type);
    this.registerControlCommandListener();
    await this.session.activate();
  } catch (err) {
    hilog.error(DOMAIN, TAG, `AVSession init failed: ${JSON.stringify(err)}`);
  }
}

private registerControlCommandListener(): void {
  if (!this.session || this.isListenerRegistered) {
    return;
  }
  this.session.on('play', () => {
    this.callback?.(MediaControlEvent.PLAY);
  });
  this.session.on('pause', () => {
    this.callback?.(MediaControlEvent.PAUSE);
  });
  this.session.on('stop', () => {
    this.callback?.(MediaControlEvent.STOP);
  });
  this.isListenerRegistered = true;
}
```

**This is the entire wear-detection implementation.** There is no
"headset removed" API to call: on earbuds that support wear detection, the
system detects the removal and sends a `pause` command down the same channel
the control centre's pause button uses; putting the bud back in sends `play`.
An app that registers these two listeners gets the behaviour for free, and an
app that does not gets audio playing into an empty room. The document's
compatibility table is really a list of which of these two mechanisms fires in
which situation.

The order in `init` matters: listeners first, `activate()` second. Activation
is what publishes the session, and the sample's `isListenerRegistered` flag
guards against a double registration if `init` is called twice.

**Classifying the routing change — `entry/src/main/ets/utils/HeadsetMonitor.ets`** (as shipped)

```typescript
private registerAudioRoutingListener(): void {
  if (!this.audioRoutingManager || this.isRoutingListenerRegistered) {
    return;
  }
  this.audioRoutingManager.on('deviceChange', audio.DeviceFlag.OUTPUT_DEVICES_FLAG,
    (deviceChanged: audio.DeviceChangeAction) => {
      const deviceType = deviceChanged.deviceDescriptors?.[0]?.deviceType;
      if (this.isBluetoothDeviceType(deviceType)) {
        if (deviceChanged.type === 0) {
          this.callback?.(HeadsetEventType.BLUETOOTH_CONNECTED, deviceType);
        } else if (deviceChanged.type === 1) {
          this.callback?.(HeadsetEventType.BLUETOOTH_DISCONNECTED, deviceType);
        }
      } else {
        if (deviceChanged.type === 0) {
          this.callback?.(HeadsetEventType.NON_BLUETOOTH_CONNECTED, deviceType);
        } else if (deviceChanged.type === 1) {
          this.callback?.(HeadsetEventType.NON_BLUETOOTH_DISCONNECTED, deviceType);
        }
      }
    });
  this.isRoutingListenerRegistered = true;
}

private isBluetoothDeviceType(deviceType: audio.DeviceType | undefined): boolean {
  if (deviceType === undefined) {
    return false;
  }
  return deviceType === audio.DeviceType.BLUETOOTH_A2DP
    || deviceType === audio.DeviceType.BLUETOOTH_SCO;
}
```

**Three details carry this.** The `DeviceFlag.OUTPUT_DEVICES_FLAG` argument
restricts the subscription to output routing - without it the callback also
fires for microphones and the pause would trigger on unrelated changes.
`deviceChanged.type` is the connect/disconnect discriminator (`0` connect,
`1` disconnect), while `deviceDescriptors` says *what* changed. And the
optional chain `deviceDescriptors?.[0]?.deviceType` matters because a change
action can arrive with an empty descriptor array; `isBluetoothDeviceType`
treats `undefined` as non-Bluetooth, which routes it to the wired branch - a
disconnect with no descriptor still pauses, which is the safe direction.

`queryInitialDeviceState` uses the same classifier over `getDevicesSync` and
skips `EARPIECE` and `SPEAKER`, because the built-in outputs are always
"connected" and would otherwise light the wired indicator on every launch.

**The PCM render callback — `entry/src/main/ets/pages/SimpleAudioPlayer.ets`** (corrected, see `HW-13-0093`, `HW-13-0062`)

```typescript
this.audioRenderer.on('writeData', (buffer: ArrayBuffer) => {
  if (!this.rawFileInfo) {
    return audio.AudioDataCallbackResult.INVALID;
  }
  const remaining = this.rawFileInfo.length - this.bytesRead;
  if (remaining <= 0) {
    this.onPlaybackFinished();                       // FIX: reset isPlaying + AVSession, allow rewind
    return audio.AudioDataCallbackResult.INVALID;
  }
  try {
    let readOpts: ReadOptions = {
      offset: this.bytesRead + this.rawFileInfo.offset,
      length: Math.min(buffer.byteLength, remaining)  // FIX: was buffer.byteLength, unclamped
    };
    const len = fs.readSync(this.rawFileInfo.fd, buffer, readOpts);
    if (len < buffer.byteLength) {
      new Uint8Array(buffer).fill(0, len);            // FIX: zero the tail of the last block
    }
    this.bytesRead += len;                            // FIX: was += buffer.byteLength
  } catch (err) {
    return audio.AudioDataCallbackResult.INVALID;
  }
  return audio.AudioDataCallbackResult.VALID;
});
```

**`getRawFdSync` returns a window into the HAP, not a file.** `offset` is
where the asset starts inside the package and `length` is how long it is, so a
read of `buffer.byteLength` bytes near the end of `world.pcm` runs straight
into whatever resource is packed after it. The shipped code compounds that by
ignoring `readSync`'s return value and advancing `bytesRead` by the full buffer
size, so the final partial block is rendered with stale bytes from the previous
callback still in its tail - the systematic `HW-13-0062` catalogues across four
samples in this industry.

The second half is the state bug. Once `bytesRead >= length` the callback
returns `INVALID` forever, the renderer goes silent, and nothing resets
`isPlaying` or the `AVPlaybackState`. In a sample whose entire subject is
"the control centre must always agree with the player", the end of a track
leaves the control centre showing PLAY over a player that can never make sound
again, and the play button does nothing because `audioRenderer.start()` still
succeeds. `onPlaybackFinished` has to set `isPlaying = false`, push
`PLAYBACK_STATE_STOP` (or `_PAUSE`), and zero `bytesRead` so the next play
starts from the top.

**Deduplicating the pushed state — `entry/src/main/ets/utils/AVSessionManager.ets`** (corrected, see `HW-13-0095`)

```typescript
setAVPlaybackState(state: avSession.AVPlaybackState): void {
  if (!this.session) {
    hilog.warn(DOMAIN, TAG, 'setAVPlaybackState: session not initialized');
    return;
  }
  if (state.state === this.currentPlayState.state) {   // FIX: was `state === this.currentPlayState`
    return;
  }
  this.currentPlayState = state;
  try {
    this.session.setAVPlaybackState(this.currentPlayState);
  } catch (err) {
    hilog.error(DOMAIN, TAG, `setAVPlaybackState failed: ${JSON.stringify(err)}`);
  }
}
```

**A one-character class of bug worth recognising.** Every caller passes a fresh
object literal - `{ state: avSession.PlaybackState.PLAYBACK_STATE_PLAY }` - so
the identity comparison `state === this.currentPlayState` is false on every
call and the guard never fires. It is dead code that looks like a working
optimisation, which is worse than no guard at all: a reader assumes duplicate
pushes are being suppressed. Compare the discriminating field.

Note also `hilog.info(... , \`setAVPlaybackState: ${state}\`)` in the shipped
version interpolates an object and logs `[object Object]`; log `state.state`.

## Permissions & config

**None.** The sample declares no `requestPermissions`. `AudioRoutingManager`
output-device queries, `AVSession` creation and playback of bundled rawfiles
are all permission-free.

Two things a real app would add:

- A **background continuous task** if playback should survive the app going to
  the background. `avsession-background-scene` in the guides covers it; the
  sample only ever plays in the foreground.
- `ohos.permission.KEEP_BACKGROUND_RUNNING` accompanies that task.

The three rawfile assets are `world.pcm` (raw 48 kHz stereo s16le, matching the
`AudioStreamInfo` in `SimpleAudioPlayer`), `world.wav` and
`video_sample_product.mp4`. The `AudioRendererInfo` uses
`StreamUsage.STREAM_USAGE_MUSIC` with `volumeMode: SYSTEM_GLOBAL`, which is what
makes the stream a music stream to the routing and focus policy - get this
wrong and the interrupt and routing behaviour changes underneath the feature.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **Wear detection depends on the headset**, not on the app. On earbuds without
  it, removing a bud produces no command and playback continues; only a full
  disconnect fires `deviceChange`. The document's table is explicit about which
  rows need 支持佩戴检测的耳机 (headsets supporting wear detection).
- `SimpleIjkPlayer` needs the third-party `@ohos/ijkplayer` and does not build
  without it; the other four pages use system APIs only.
- The five players duplicate their monitoring code; there is no shared base
  component.
- Only `play`, `pause` and `stop` are registered. A production session usually
  also wants `playNext`, `playPrevious` and `seek`, and should publish position
  in `AVPlaybackState` so the control centre's progress bar moves.
- `SimpleNativeVideoPlayer` pushes PLAY from the `Video` component's `onStart`
  but has no `onPause`/`onFinish` handler, so using the component's own
  built-in controls desynchronises the session.

## Pitfalls

- **`HW-13-0093` (B/medium, confirmed) — the PCM renderer ignores `readSync`'s
  return value and can read past the rawfile window; after EOF the player is
  permanently silent while the UI and the control centre still show playing.**
  `length` is the full buffer size regardless of how much of the asset is left,
  `bytesRead` advances by the buffer size rather than the bytes actually read,
  and the EOF branch returns `INVALID` forever without touching `isPlaying` or
  the session state. Fix: clamp the read to the remaining length, advance by the
  real return, and on EOF set `isPlaying = false`, push a terminal
  `AVPlaybackState` and reset `bytesRead` so playback can restart.
- **`HW-13-0094` (D/medium, confirmed) — a state-sync demo with no
  `audioInterrupt` handling in any of its five players.** There is no
  `audioRenderer.on('audioInterrupt')` or `avPlayer.on('audioInterrupt')`
  anywhere in the project. An incoming call or another app taking focus pauses
  the stream while `isPlaying` and the `AVPlaybackState` both still say PLAY -
  desynchronising precisely what the document promises to keep in sync
  (播控中心的状态要始终和音视频的播放状态保持同步). Fix: register the interrupt
  event on each player and route `PAUSE`/`RESUME` hints through the same code
  path the headset disconnect uses.
- **`HW-13-0095` (B/low, confirmed) — the `AVPlaybackState` dedupe compares
  object identity.** Callers always pass fresh literals, so
  `state === this.currentPlayState` is never true and the guard is dead code
  that reads as a working optimisation. Fix: compare `state.state`.
- **`HW-13-0062` (B/low, confirmed) — systematic: the final partial audio chunk
  leaves stale bytes in the render buffer, in four samples.**
  `demo_AudioSession` is one of them, at `SimpleAudioPlayer.ets` `writeData`;
  the others are `AudioVisualization`, `PCMTranscode` and the native
  `demo_HttpAudioRender`. The last frame of every playback renders garbage or an
  echo of the previous buffer. Fix: zero the remainder of the buffer past the
  bytes actually read.
- **`HW-13-0012` (B/low, confirmed) — systematic: `getRawFd`/`getRawFdSync`
  descriptors are never closed with `closeRawFd`, across seven media samples.**
  In this project `SimpleAudioPlayer.release()` closes the raw descriptor with
  `fs.closeSync(this.rawFileInfo.fd)` - the wrong call for a resource-manager
  descriptor - while `SimpleAVPlayer` and `SimpleVideoPlayer` never close theirs
  at all. Fix: `resourceManager.closeRawFd(name)` after the player has been
  released.
- **`AVSessionManager.release()` is not awaited** in any page's
  `aboutToDisappear` (`this.avSessionManager.release();` with no `await`), so
  `deactivate()`/`destroy()` may still be in flight as the page is destroyed.
- **Errors are swallowed.** `initAudioRenderer`, `loadPcmFile`, `playAudio`,
  `pauseAudio` and `loadSong` all end in `catch (err) { // ignore }`. A failed
  renderer creation leaves a page whose buttons silently do nothing.

## References

- `documentation/harmonyos-guides/05_media/avsession-overview.md` - session roles, activation, the command set
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/avsession-overview
- `documentation/harmonyos-guides/02_media/avsession-kit.md` - AVSession Kit introduction
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/avsession-kit
- `documentation/harmonyos-guides/05_media/avsession-access-scene.md` - what the control centre renders and what metadata it needs
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/avsession-access-scene
- `documentation/harmonyos-references/04_media/arkts-apis-avsession-avsession.md` - `on('play')`, `on('pause')`, `on('stop')`, `setAVPlaybackState`, `setAVMetadata`, `deactivate`, `destroy`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-avsession-avsession
- `documentation/harmonyos-guides/05_media/audio-output-device-management.md` - `AudioRoutingManager`, `getDevicesSync`, `deviceChange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/audio-output-device-management
- `documentation/harmonyos-guides/05_media/audio-output-device-change.md` - what happens to a stream when the route changes
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/audio-output-device-change
- `documentation/harmonyos-references/04_media/arkts-apis-audio-audioroutingmanager.md` - `DeviceChangeAction`, `DeviceFlag`, `DeviceType`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio-audioroutingmanager
- `documentation/harmonyos-guides/05_media/audio-playback-concurrency.md` - audio focus and the interrupt events `HW-13-0094` is about
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/audio-playback-concurrency
- `documentation/harmonyos-references/04_media/arkts-apis-audio-audiorenderer.md` - `writeData` and `AudioDataCallbackResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio-audiorenderer
- `huawei_industry_tree/13_media_entertainment/docs/42_demo_audio_session.md` - the source document, including the device/action compatibility table
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/demo_audio_session-0000002668471732
- `MEDIA-27` - the stale-tail render-buffer systematic (`HW-13-0062`)
- `MEDIA-01` - the unclosed raw-descriptor systematic (`HW-13-0012`)
