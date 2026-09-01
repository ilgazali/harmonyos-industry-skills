---
id: SOCIAL-21
title: Voice call screen - one emitter-driven page for calling, ringing and in-call, over a TCP audio socket
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/21_voice_call.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_call-0000002328140805
sample: huawei_industry_tree/14_social_communication/downloads/VoiceCall.zip
kits: ["@kit.AbilityKit", "@kit.AudioKit", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [UIAbility, Want, emitter, commonEventManager, "audio.getAudioManager", getRoutingManager, setCommunicationDevice, isCommunicationDeviceActive, getVolumeGroupManager, isMicrophoneMute, "media.createAVPlayer", fdSrc, getRawFd, "worker.ThreadWorker", postMessage, "socket.constructTCPSocketInstance", wifiManager, "window.createWindow", setUIContent, abilityAccessCtrl, bundleManager, display, hilog, resourceManager]
permissions: [ohos.permission.MICROPHONE, ohos.permission.INTERNET, ohos.permission.GET_NETWORK_INFO]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0048, HW-14-0049, HW-14-0050, HW-14-0051, HW-14-0085, HW-14-0087, HW-14-0088]
status: verified-with-fixes
---

## When to use

Load this card for the **call screen itself**: the full-bleed page that shows a
caller, a prompt or a running timer, and a row of round buttons whose contents
change with the call's phase - outgoing, incoming, connected. It is the UI
shell around a voice call, plus a worked example of an app owning its own audio
routing while a call is up.

The transferable structure is *one page, three states, driven by events*. The
sample renders one of `UICall`, `UIAnswer` or `UIVoice` from a single number in
`AppStorage`, and that number is written by `emitter` events raised from the
controller and by `commonEvent` messages from the peer app. The same shape fits
a video call, a screen share, or any two-party session with a handshake before
the media flows.

**Read `HW-14-0048` before you trust this sample as a media reference.** The
document opens by claiming AudioCapturer/AudioRenderer capture and render over
a socket. The socket and the worker plumbing are there; the workers themselves
were never shipped, so everything below the transport demonstrates the *shape*
of a call, not working audio. The sample's own 说明 admits as much:
"通话功能需开发者自行实现" (the call functionality is for the developer to
implement).

## Feature checklist

- Launched with a `Want` parameter saying whether this is an outgoing call, an
  incoming call, or (by default) the incoming screen.
- The incoming screen shows the caller's avatar and name, a prompt, and two
  round buttons: 拒绝 (reject, red) and 接听 (answer, green).
- The outgoing screen shows the same avatar with the 等待接听 (waiting for an
  answer) prompt, a microphone toggle and a red hang-up button.
- Answering switches the page to the in-call screen: a running mm:ss timer, a
  microphone toggle labelled 麦克风已开 / 麦克风已关, and hang-up.
- The microphone and speaker toggles go through the audio manager
  (`setCommunicationDevice`, `isMicrophoneMute`).
- A looping ringtone plays from a raw file until the call is answered or ends.
- An unanswered call times out after 30 seconds and terminates the ability.
- Answer and hang-up are published as common events, so the companion chat app
  stays in step, and the peer's answer or hang-up drives this screen too.
- A dropped Wi-Fi connection or a closed socket tears the call down.

## Architecture

One `entry` module, layered honestly: UI components, one controller, three
models, a socket wrapper and five utilities.

```
entry/src/main/ets
├── bean/Caller.ets                     { index, name, head }
├── components
│   ├── ComponentOption.ets             the round button; icon/background flip on `use`
│   ├── ComponentPerson.ets             avatar + name
│   ├── ComponentSingletonTimer.ets     mm:ss text bound to the shared TimerUtil
│   ├── ComponentVoiceBg.ets            shared background + @BuilderParam button row
│   ├── UIAnswer.ets                    incoming: reject / answer
│   ├── UICall.ets                      outgoing: mic / hang up
│   └── UIVoice.ets                     in-call: mic / hang up, with the timer
├── constant/Constants.ets              event ids, worker message codes, want keys
├── controller
│   ├── FloatWindowController.ets       minimise-to-float - never invoked (HW-14-0049)
│   └── IndexController.ets             singleton: init, answer, hangUp, exit, audio
├── entryability/EntryAbility.ets       reads the Want, full screen, avoid areas
├── entrybackupability/
├── model
│   ├── AudioManagerModel.ets           routing + volume group manager wrapper
│   ├── BufferModel.ets                 socket lifecycle + the two audio workers
│   └── OptionModel.ets                 Want parsing, commonEvent publish/subscribe
├── net/SocketImpl.ets                  TCPSocket wrapper (bind, connect, send, listeners)
├── pages/Index.ets                     @Entry: three-way switch on the UI event
└── utils
    ├── IpUtils.ets                     int -> dotted-quad
    ├── Logger.ets
    ├── PermissionUtils.ets             microphone request
    ├── SoundUtil.ets                   AVPlayer ringtone, loops on 'completed'
    ├── TimeUtil.ets                    ms -> mm:ss / hh:mm:ss
    └── TimerUtil.ets                   one setInterval, many observers
```

The documented tree matches the zip file-for-file. What neither the tree nor
the document mentions is `ets/workers/` - **which does not exist**, although
`BufferModel` constructs two `ThreadWorker`s from it (`HW-14-0048`).

**The design decision worth copying** is `TimerUtil`: one process-wide
`setInterval` at 1 Hz with a list of observers, instead of a timer per feature.

```typescript
private observers: Observer[] = [];
startTimer(): void {
  this.clearObservers();
  this.clearCount();
  this.taskId = setInterval(() => { this.task(); }, ONE_SECOND);
  this.setCurrentTimestamp();
}
```

Three unrelated concerns ride that one tick: the ring timeout in
`IndexController`, the 3-second socket health check in `BufferModel`, and the
on-screen call duration in `ComponentSingletonTimer`. Each registers a callback
and removes it on teardown. The file's own comment gives the reason - several
`setInterval`s on one thread drift against each other and multiply wake-ups;
one tick keeps them coherent and makes "stop everything" a single call. The
trap in the same file is that `startTimer()` begins with `clearObservers()`, so
anything registered before the timer starts is silently dropped;
`IndexController.init` gets the order right only by accident.

## Implementation steps

1. **Decide the screen from the `Want`, before the page loads.** `onCreate`
   calls `IndexController.beforeInit(want, this.context)`, which reads the
   caller id and the event key, writes the UI event into `AppStorage` and
   emits it; a `Want` with no usable parameters terminates the ability.
2. **Render one of three components from one number.** `Index` holds
   `@StorageLink('UI_EVENT')` and switches on it. Everything shared - the
   background, avatar, prompt or timer - lives in `ComponentVoiceBg`, whose
   `@BuilderParam options` slot is where each screen puts its own button row.
3. **Make the buttons one component.** `ComponentOption` takes `@Link use`, two
   icon resources, a background colour and a `canChange` predicate: toggles
   (mic) return `true`, one-shot actions (answer, hang up) return `false` and
   only fire the callback.
4. **Start the shared timer first, register observers after** - see above.
5. **Request the microphone on entry** through `PermissionUtils`, and toast
   `tips_no_permission` if it is refused; the call screen still renders.
6. **Route audio through the routing manager**, not through volume APIs:
   `setCommunicationDevice(CommunicationDeviceType.SPEAKER, on)` to toggle the
   speaker, `isCommunicationDeviceActive` to read it back.
7. **Loop the ringtone from the AVPlayer state machine**, replaying on
   `'completed'`; release it on `paused` / `stopped` / `error`.
8. **Poll the socket from the timer tick** every 3 seconds, and treat "was
   connected, now Wi-Fi or socket is down" as a disconnect. **Reset the
   connected flag when you report the disconnect** (`HW-14-0051`).
9. **Run the same teardown on every exit path**, including a back gesture -
   not only on the hang-up button (`HW-14-0050`).
10. **Ship the workers, or drop the claim.** Capture and render live in two
    `ThreadWorker`s that this sample does not contain (`HW-14-0048`); if you
    want the minimise-to-float behaviour, wire the controller up and add the
    page it loads (`HW-14-0049`).

## Verified snippets

All snippets are from `VoiceCall.zip`. Corrected forms are marked.

**The three-state page - `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-14-0050`)

```typescript
@Entry
@Component
struct Index {
  private controller: IndexController = IndexController.getInstance();
  @State mRemote: Caller = { index: -1, name: $r('app.string.name_default'), head: $r('app.media.icon') };
  @StorageLink('UI_EVENT') mUIEvent: number = -1;

  aboutToAppear() {
    emitter.on({ eventId: Constants.EVENT_UI_CALL }, () => { this.onUIEvent(Constants.EVENT_UI_CALL); });
    emitter.on({ eventId: Constants.EVENT_UI_ANSWER }, () => { this.onUIEvent(Constants.EVENT_UI_ANSWER); });
    emitter.on({ eventId: Constants.EVENT_UI_VOICE }, () => { this.onUIEvent(Constants.EVENT_UI_VOICE); });

    grantPermission(this.getUIContext().getHostContext() as common.UIAbilityContext).then(result => {
      if (!result) {
        this.getUIContext().getPromptAction()
          .showToast({ message: $r('app.string.tips_no_permission'), duration: 2000 });
      }
    });

    this.controller.init();
    this.mRemote = this.controller.getCaller();
  }

  aboutToDisappear() {
    try {
      emitter.off(Constants.EVENT_UI_CALL);
      emitter.off(Constants.EVENT_UI_ANSWER);
      emitter.off(Constants.EVENT_UI_VOICE);
      this.controller.exit();      // FIX: the sample calls destroy(), which only stops the audio scene
    } catch (e) {
    }
  }

  build() {
    Column() {
      if (this.mUIEvent === Constants.EVENT_UI_CALL) {          // 呼叫页
        UICall({ controller: this.controller, mRemote: $mRemote });
      } else if (this.mUIEvent === Constants.EVENT_UI_ANSWER) { // 来电页
        UIAnswer({ controller: this.controller, mRemote: $mRemote });
      } else if (this.mUIEvent === Constants.EVENT_UI_VOICE) {  // 通话页
        UIVoice({ controller: this.controller, mRemote: $mRemote });
      }
    }
    .width('100%')
    .height('100%');
  }
}
```

**`@StorageLink` rather than `@State` is what lets a non-UI object drive the
screen.** `beforeInit` runs in `UIAbility.onCreate`, before the page exists;
it writes `UI_EVENT` into `AppStorage`, so by the time `Index` is built the
correct screen is already selected. The `emitter` subscriptions then handle
every later transition, and `onUIEvent` writes the value back into `AppStorage`
so the two stay in sync.

**The teardown asymmetry is the defect.** `controller.destroy()` calls only
`exitVoiceCall()` - the audio scene and the (absent) worker stop. Meanwhile
`exit()` clears the ringtone, stops the shared timer, unsubscribes the common
events and closes the socket. Press hang-up and you get `exit()`; swipe back
during ringing and you get `destroy()`, leaving the looping `AVPlayer`, the 1 Hz
interval, the `commonEvent` subscriber and the TCP socket alive in a process
the user believes they have left. Calling `exit()` from `aboutToDisappear`
makes every exit path pay the same cost.

**Audio routing - `entry/src/main/ets/model/AudioManagerModel.ets`** (as shipped)

```typescript
async initManager(): Promise<void> {
  this.mAudioManager = audio.getAudioManager();
  this.mAudioRoutingManager = this.mAudioManager.getRoutingManager();
  this.mAudioVolumeGroupManager = await this.mAudioManager.getVolumeManager()
    .getVolumeGroupManager(audio.DEFAULT_VOLUME_GROUP_ID);
}

async setSpeaker(speaker: boolean): Promise<void> {
  try {
    if (!this.mAudioRoutingManager) {
      return;
    }
    await this.mAudioRoutingManager.setCommunicationDevice(audio.CommunicationDeviceType.SPEAKER, speaker);
  } catch (err) {
    Logger.error(TAG, `setSpeaker failed, code is ${err.code}, message is ${err.message}`);
  }
}

async isSpeakerActive(): Promise<boolean> {
  if (!this.mAudioRoutingManager) {
    return false;
  }
  return await this.mAudioRoutingManager.isCommunicationDeviceActive(audio.CommunicationDeviceType.SPEAKER);
}
```

**Two managers, two jobs.** The *routing* manager decides where the sound goes
(earpiece vs. speaker) and is the correct home for the speaker toggle; the
*volume group* manager owns the microphone mute state. Reaching for volume APIs
to switch the earpiece is the common mistake this file avoids.

The file also carries the sample's most useful comment, on `setVoiceScene`:
`setAudioScene` is a system API, and since 4.0.7.1 a third-party app does not
need it - a `VOICE_CALL`-usage stream auto-switches to `AUDIO_SCENE_VOICE_CHAT`
on `start()` and back on stop. So that method is deliberately a logging stub,
and so is `setMicrophoneMute`: the mute here is a UI state only.

**Socket health from the shared tick - `entry/src/main/ets/model/BufferModel.ets`** (corrected, see `HW-14-0051`)

```typescript
private async socketConnTask(): Promise<void> {
  if (this.mSocketImpl == null) {
    this.mSocketImpl = new SocketImpl();
  }
  let wifiState: boolean = await this.checkWifiState();
  let socketConnected: boolean = await this.isConnected();
  if (!wifiState) {
    if (this.lastConnected) {          // was up, wifi went away
      this.lastConnected = false;      // FIX: absent in the sample
      this.onDisconnected();
    }
    return;
  }
  if (this.lastConnected && !socketConnected) {   // was up, socket went away
    this.lastConnected = false;                   // FIX: absent in the sample
    this.onDisconnected();
    return;
  }
  if (!socketConnected) {
    let ret: boolean = await this.connectToAp();
    if (ret) {
      this.setSocketListener();
    }
    return;
  }
  this.lastConnected = socketConnected;
}
```

**This is a state machine over one boolean, and the boolean is only ever set
one way.** `lastConnected` is assigned `true` on the healthy path at the bottom
and never assigned `false` anywhere in the file. So after the first real
disconnect it stays `true` forever: the check runs again three seconds later,
sees "was connected, is not connected", and fires `onDisconnected()` → `hangUp()`
→ `exit()` again - a teardown storm every tick until the ability finally goes
away. The two assignments above make the transition one-shot, which is what the
comment block above the method describes.

Note the reason this polling exists at all, stated in the source: after the
network drops, `onError`/`onClose` may not fire and `getState` may report
stale values. So the sample deliberately supplements the callbacks with a
3-second poll on the shared timer, and treats the callbacks as an optimisation.

**The workers that are not there - same file** (as shipped)

```typescript
public init(callback: () => void): void {
  this.connect();
  try {
    this.mCapturerWorker = new worker.ThreadWorker('../../ets/workers/CapturerWorker');
    this.mRendererWorker = new worker.ThreadWorker('../../ets/workers/RendererWorker');
    this.mCapturerWorker.onmessage = (message: MessageEvents) => {
      this.onMessage(message);
    };
    this.onDisconnectedCallback = callback;
  } catch (err) {
    Logger.error(TAG, `init failed ${JSON.stringify(err)}`);   // swallows the missing-worker error
  }
}

public startWorkTask(): void {
  try {
    if (this.mCapturerWorker && this.mRendererWorker) {        // both null: the body never runs
      this.mCapturerWorker.postMessage({ 'code': Constants.WORK_MESSAGE_CAPTURER_START });
      this.mRendererWorker.postMessage({ 'code': Constants.WORK_MESSAGE_RENDERER_START });
    }
  } catch (err) {
    Logger.error(TAG, `startWorkTask failed ${JSON.stringify(err)}`);
  }
}
```

**The design is right and the implementation is absent.** Capture and render on
a worker thread, the UI thread doing only socket I/O, and a four-code message
protocol (`CAPTURER_START/STOP/SEND`, `RENDERER_START/STOP/RECV`) with the
audio buffer transferred rather than copied -
`postMessage({...}, [buffer])` - is exactly how a real-time audio path should
be built. But `ets/workers/` is not in the project, there is no `workers` entry
in `build-profile.json5`, and the document's tree lists none. Both constructors
throw into the catch above, both handles stay `null`, and every guarded call
site quietly does nothing. `onDisconnectedCallback` is assigned *after* the
constructors, so it is never assigned either - which means the socket
disconnect detection above cannot actually reach `hangUp()`.

If you adopt this card's architecture, write the two workers first; treat the
rest of the sample as the call shell around them.

One more object matters for `HW-14-0050`: `SoundUtil` loops the ringtone by
calling `play()` again from the `'completed'` branch of the AVPlayer's
`stateChange` handler, releasing only on `paused` / `stopped` / `error`. Nothing
in the back-gesture path reaches any of those states, so a player that replays
itself on every completion keeps the process audible after the user has left.
The fd comes from `resourceManager.getRawFd('wechat_voice_ring.mp3')` - a raw
file in the package, so no file permission is involved.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.MICROPHONE",
    "reason": "$string:MICROPHONE",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  },
  { "name": "ohos.permission.INTERNET",         "reason": "$string:INTERNET" },
  { "name": "ohos.permission.GET_NETWORK_INFO", "reason": "$string:GET_NETWORK_INFO" }
]
```

- `MICROPHONE` is `user_grant`: `reason` and `usedScene` are mandatory, and
  `when: "inuse"` is correct here - the app has no continuous task and must not
  hold the microphone in the background.
- `INTERNET` and `GET_NETWORK_INFO` are `system_grant`; the `reason` fields are
  harmless but not required.
- `GET_NETWORK_INFO` is what `wifiManager.getLinkedInfo()` and `getIpInfo()`
  need for the connection health check and for the local bind address.
- The peer address is a hardcoded `192.168.137.1:8978` in `BufferModel`, with a
  comment telling the developer to find it with `ipconfig`. The device must be
  on the PC's hotspot for the socket to connect at all.
- No floating-window permission is declared, consistent with
  `FloatWindowController` never running (`HW-14-0049`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **No audio is captured or rendered.** The workers are missing
  (`HW-14-0048`); this is a call *screen*, not a call.
- The sample is one half of a pair. It expects a companion chat app to launch
  it with `START_ABILITY_EVENT_KEY` / `START_ABILITY_DATA_KEY` and to exchange
  `COMM_EVENT_ANSWER_FROM_HOST` / `COMM_EVENT_HANGUP_FROM_HOST`. Launched
  alone, `readWant` falls back to `EVENT_UI_ANSWER` and caller `callData[0]`.
- The unanswered-call timeout is 30 ticks of the shared timer, hardcoded in
  `IndexController.timeoutCallback`; `exit()` then terminates the whole ability
  400 ms later, so the screen cannot be reused for a second call.
- `TimerUtil.startTimer()` clears all observers first; registering before
  starting silently loses the observer.
- `Caller.head` for the second test user is a **string** resource used where a
  media resource is expected (`$r('app.string.head2')`), so that avatar cannot
  render. The three permission `reason` strings are the permission names
  themselves rather than sentences a user could act on.

## Pitfalls

- **`HW-14-0048`** (B/high, confirmed): `BufferModel` constructs
  `ThreadWorker('../../ets/workers/CapturerWorker')` and `RendererWorker`, but
  the project has no `ets/workers` directory and no `workers` entry in
  `build-profile.json5`. Both constructors throw into a swallowing catch, both
  handles stay `null`, every guarded call is a no-op - and, because
  `onDisconnectedCallback` is assigned after them, the disconnect callback is
  never registered either. The document's claim of AudioCapturer/AudioRenderer
  capture and render over a socket is unsupported by the code. Fix: ship the
  worker sources, or correct the document and the code.
- **`HW-14-0049`** (B/medium, confirmed): the documented 悬浮窗控制类 is dead.
  `initWindowStage` and `hideMain` are never called from anywhere,
  `setUIContent('pages/Float')` names a page that is in neither the project nor
  `main_pages.json`, and `destroyFloatWindow` never nulls the handle so a
  second call destroys an already-destroyed window. Fix: call
  `initWindowStage` from `onWindowStageCreate` and add the page - or delete the
  controller.
- **`HW-14-0050`** (B/medium, probable): `aboutToDisappear` only removes three
  emitter subscriptions and calls `controller.destroy()`, which stops the audio
  scene and nothing else. A back gesture during ringing leaves the looping
  `AVPlayer`, the 1 Hz `TimerUtil` interval, the `commonEvent` subscriber and
  the TCP socket running in the background. Fix: run the full `exit()` teardown
  from `aboutToDisappear`.
- **`HW-14-0051`** (B/low, confirmed): `lastConnected` is only ever set `true`,
  so after one disconnect the 3-second health check re-fires the whole
  hang-up-and-exit teardown on every tick; and `TimeUtil.getTimeStr` combines
  `getUTCHours()` with local `getMinutes()`/`getSeconds()`, so in a half-hour
  offset timezone a 10-second call displays as `30:10`. Fix: set
  `lastConnected = false` when reporting the disconnect, and use the UTC
  accessors consistently.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-audio.md` - `getAudioManager`, `AudioRoutingManager`, `CommunicationDeviceType`, volume group manager
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio
- `documentation/harmonyos-references/04_media/arkts-apis-audio-audiocapturer.md` - the capture side the missing worker was meant to implement
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio-audiocapturer
- `documentation/harmonyos-references/04_media/arkts-apis-audio-audiorenderer.md` - `VOICE_CALL` usage and the automatic audio-scene switch
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio-audiorenderer
- `documentation/harmonyos-references/02_application-framework/js-apis-worker.md` - `ThreadWorker`, `postMessage`, transferable buffers
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-worker
- `documentation/harmonyos-references/03_system/errorcode-net-socket.md` - TCPSocket bind/connect failures; and `js-apis-bluetooth-socket.md` for the alternative transport the document cites
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/errorcode-net-socket
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `INTERNET`, `GET_NETWORK_INFO`; `permissions-for-all-user.md` - `MICROPHONE`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `SOCIAL-22` - the in-page float bubble to compare against this sample's unfinished system float window
- `SOCIAL-23` - working `AudioCapturer`/`AudioRenderer` code in this industry, if you need the media path this sample omits
