---
id: EDU-11
title: Read-aloud practice - record the student, play back against the model sentence, with a live mic waveform
industry: 04_education
doc: huawei_industry_tree/04_education/docs/11_oral_english.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/oral_english-0000002349625326
sample: huawei_industry_tree/04_education/downloads/OralEnglish.zip
kits: ["@kit.MediaKit", "@kit.AbilityKit", "@kit.CoreFileKit", "@kit.LocalizationKit", "@kit.CryptoArchitectureKit", "@kit.ArkUI", "@kit.ArkTS"]
apis: ["media.createAVRecorder", "AVRecorderConfig", "AVRecorderProfile", "CodecMimeType.AUDIO_AAC", "ContainerFormatType.CFT_MPEG_4A", "AudioSourceType.AUDIO_SOURCE_TYPE_MIC", prepare, start, stop, release, getAudioCapturerMaxAmplitude, "media.createAVPlayer", fdSrc, AVFileDescriptor, "resourceManager.getRawFdSync", "abilityAccessCtrl.createAtManager", requestPermissionsFromUser, checkAccessToken, requestPermissionOnSetting, animateTo, setInterval, clearInterval, "fileIo.open", "fileIo.close", "@StorageProp", PromptAction]
permissions: ["ohos.permission.MICROPHONE"]
min_api: 20
modules: [entry]
findings: [HW-04-0076, HW-04-0077, HW-04-0078, HW-04-0079, HW-04-0080, HW-04-0081, HW-04-0082, HW-04-0083, HW-04-0084, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card for the **record-and-compare loop**: play a model sentence, let
the student record themselves reading it, then play their recording back. It is
the core interaction of every pronunciation, language-lab and speech-practice
feature.

Three things here are not obvious and are worth taking:

- **`getAudioCapturerMaxAmplitude()`** gives you a live microphone level while
  `AVRecorder` is running, which is what makes a real waveform possible without
  touching `AudioCapturer` directly.
- **One `AVPlayer`, two source kinds.** The model sentence is a rawfile
  (`getRawFdSync` → `fdSrc` with `offset` and `length`); the student's recording
  is a sandbox file (`fileIo.open` → `fdSrc` with just `fd`). Same player, two
  descriptor disciplines - and the sample only cleans up one of them.
- **A single state enum in `AppStorage`** (`IDLE`/`RECORDING`/`PLAYING_EXAMPLE`/
  `PLAYING_RECORD`) drives the whole UI, written by the media callbacks and read
  by the page with `@StorageProp`.

The lifecycle handling in this sample is where it goes wrong - four of the nine
findings are double-release or leak defects - so read the teardown sections
before copying the managers.

## Feature checklist

- A topic card with sentence count, vocabulary size, estimated time and a
  live progress counter of how many sentences have been recorded.
- A list of sentences; tapping one selects it and stops whatever was playing or
  recording.
- On the selected sentence: play the model audio, record, play back your own
  recording.
- While recording, the sentence row is replaced by a 30-bar waveform that reacts
  to the microphone level, plus a stop button.
- Recording is capped (20 s) and self-stops with a toast.
- The microphone permission is requested on first record, and a refusal is
  reported.
- Leaving the page stops and releases both the recorder and the player.

## Architecture

One `entry` module, one page, two media managers.

```
entry/src/main/ets
├── components
│   ├── AudioWaveComponent.ets     30-bar waveform + stop button (shown while recording)
│   ├── RecordButtons.ets          play-example / record / play-back (shown otherwise)
│   └── ImageButton.ets
├── constants/Constants.ets        AMPLITUDE_INTERVAL 200, AMPLITUDE_MAX 32767, TICK_COUNT_MAX 100 ...
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── media
│   ├── AVRecorderManager.ets      create -> prepare -> start -> stop -> release, + amplitude polling
│   └── AVPlayerManager.ets        one player, two fdSrc flavours
├── model
│   ├── OralContent.ets            the exercise: title, vocabulary, estimated, sentences[]
│   └── RecordStatus.ets           IDLE | RECORDING | PLAYING_EXAMPLE | PLAYING_RECORD
├── pages/OralTrainingPage.ets     @Entry - owns both managers
└── utils
    ├── PermissionUtil.ets
    ├── RawfileUtil.ets            reads the exercise JSON
    └── LogUtil.ets
```

The documented tree matches the zip.

**`RecordStatus` in `AppStorage` is the single source of truth,** and the media
callbacks are the only writers:

| Written by | To |
| --- | --- |
| `AVRecorder` `stateChange` 'prepared' | `RECORDING` |
| `AVRecorder` `stateChange` 'stopped', `error` | `IDLE` |
| `AVPlayerManager.prepareRecordAudio` | `PLAYING_RECORD` |
| `AVPlayerManager.prepareExampleAudio` | `PLAYING_EXAMPLE` |
| `AVPlayer` `stateChange` 'stopped'/'completed'/'released', `error` | `IDLE` |

The page reads it with `@StorageProp('recordStatus')` and swaps
`AudioWaveComponent` for `RecordButtons` on it. Nothing in the UI layer decides
what state the media is in - it observes. That is the part of this sample worth
copying wholesale.

**Both managers are plain classes owned by the page,** created in
`aboutToAppear` and released in `aboutToDisappear`, with an extra
`onPageHide` that stops (but does not release) whatever is active. Three
lifecycle hooks, three different jobs - and the release paths overlap with the
state-change callbacks, which is the source of `HW-04-0077` and `HW-04-0078`.

## Implementation steps

1. **Declare `ohos.permission.MICROPHONE`** with a real `reason` and
   `usedScene.abilities` naming **your** ability - the sample names a
   `FormAbility` that does not exist (`HW-04-0076`).
2. **Check before requesting.** `checkAccessToken` first; `requestPermissionsFromUser`
   only when the grant is missing; and read `dialogShownResults` so a permanent
   refusal falls back to `requestPermissionOnSetting` (`HW-04-0082`).
3. **Open the output file and build the config.** `fileIo.open` with
   `READ_WRITE | CREATE`, then `url: 'fd://' + fd`. Keep the fd - you must close
   it after `release()`.
4. **Register the callbacks before `prepare`.** `prepare` drives the state
   machine to `prepared`, which is where `start()` is called, so the handler must
   already exist.
5. **Poll the amplitude on an interval** while recording:
   `getAudioCapturerMaxAmplitude()` every 200 ms, normalised by
   `AMPLITUDE_MAX = 32767` (the 16-bit PCM ceiling).
6. **Cap the recording** with a tick counter in the same interval, and stop with
   a toast when it trips.
7. **Give teardown exactly one owner.** Either the `stateChange` handler cleans
   up (release, `clearInterval`, `fileIo.close`) or the explicit stop method
   does - never both (`HW-04-0077`).
8. **Null the manager's field after releasing** so a second release cannot run
   on a dead instance (`HW-04-0078`).
9. **Track which kind of descriptor is open.** `fileIo.close` for the sandbox
   recording, `resourceManager.closeRawFdSync` for the rawfile - the sample
   never releases the rawfile one (`HW-04-0079`).
10. **Compute the waveform outside `build`** and store it in state; `build` must
    be a pure function of state (`HW-04-0081`).

## Verified snippets

All snippets are from `OralEnglish.zip`. Corrected forms are marked.

**Recorder configuration — `entry/src/main/ets/media/AVRecorderManager.ets`** (as shipped)

```typescript
private avProfile: media.AVRecorderProfile = {
  audioBitrate: 112000,
  audioChannels: 2,
  audioCodec: media.CodecMimeType.AUDIO_AAC,        // AAC, MP3 or G711MU
  audioSampleRate: 48000,
  fileFormat: media.ContainerFormatType.CFT_MPEG_4A // MP4, M4A, MP3, WAV, AMR, AAC
};
private avConfig: media.AVRecorderConfig = {
  audioSourceType: media.AudioSourceType.AUDIO_SOURCE_TYPE_MIC,
  profile: this.avProfile,
  url: ''                                           // filled in per recording
};

async createAndSetFd(context: Context, fileIndex: number): Promise<string> {
  const fileName = util.format('/record_%d.m4a', fileIndex);
  this.filePath = context.filesDir + fileName;
  const audioFile: fileIo.File =
    await fileIo.open(this.filePath, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
  this.avConfig.url = 'fd://' + audioFile.fd;       // AVRecorder writes through the descriptor
  this.fileFd = audioFile.fd;                       // keep it - you must close it after release
  return this.filePath;
}
```

**`url` is `fd://<n>`, not a path.** The recorder never opens the file itself, so
the fd's lifetime is yours: open before `prepare`, close **after** `release`.
Closing it earlier truncates the recording. The codec and container must agree -
`AUDIO_AAC` with `CFT_MPEG_4A` produces the `.m4a` the file name promises.

**The recorder state machine — same file** (corrected, see `HW-04-0077`)

```typescript
setAVRecorderCallback() {
  if (!this.avRecorder) { return; }

  this.avRecorder.on('error', (err) => {
    LOGGER.error(`avRecorder failed ${JSON.stringify(err)}`);
    AppStorage.setOrCreate('recordStatus', RecordStatus.IDLE);
    this.avRecorder?.reset();
  });

  this.avRecorder.on('stateChange', async (state) => {
    switch (state) {
      case 'prepared':
        AppStorage.setOrCreate('recordStatus', RecordStatus.RECORDING);
        this.avRecorder?.start();          // prepare() lands here; start() belongs here
        break;
      case 'stopped':
        AppStorage.setOrCreate('recordStatus', RecordStatus.IDLE);
        this.onRecordingFinish(this.filePath);
        this.onRecordingFinish = () => {};
        // FIX: the sample ALSO releases here and nulls the field, while
        // stopRecordingProcess releases too - and this path clears neither the
        // amplitude interval nor the file descriptor. Report state here; let
        // stopRecordingProcess own the teardown.
        break;
    }
  });
}
```

**`prepare` is what starts the recording, indirectly.** You never call `start()`
from the caller - `prepare(config)` drives the machine to `prepared` and the
handler calls `start()`. That is why the callbacks must be registered before
`prepare`, and why `RECORDING` is published from inside the handler rather than
from the caller.

**Two teardown paths is the defect.** `stop()` fires `stateChange('stopped')`
*during* the `await`, so by the time control returns to `stopRecordingProcess`
the field it checked has been nulled and its own `release()` runs on `undefined`.
And on the paths where `stopRecordingProcess` is *not* called - the tick-count
cutoff, the error handler - the 200 ms amplitude interval is never cleared and
keeps re-raising the "recording too long" toast for the rest of the session.

**Amplitude polling and the recording cap — same file** (as shipped)

```typescript
async startRecordingProcess(context: UIContext, fileIndex: number,
                            amplitudeCallback: Callback<number>) {
  const abilityContext = context.getHostContext() as common.UIAbilityContext;
  if (!await PermissionUtil.requestPermissionsFromUser('ohos.permission.MICROPHONE', abilityContext)) {
    context.getPromptAction().showToast({ message: $r('app.string.toast_not_authorized') });
    return;
  }
  await this.stopRecordingProcess();
  try {
    this.avRecorder = await media.createAVRecorder();
    this.setAVRecorderCallback();                    // before prepare - see above
    await this.createAndSetFd(abilityContext, fileIndex);
    await this.avRecorder.prepare(this.avConfig);
    this.tickCount = 0;
    this.intervalId = setInterval(() => {
      this.avRecorder?.getAudioCapturerMaxAmplitude()
        .then((amplitude) => { amplitudeCallback(amplitude); })
        .catch((err: BusinessError) => {
          LOGGER.error(`getAudioCapturerMaxAmplitude failed ${err.code} ${err.message}`);
        });
      if (++this.tickCount > TICK_COUNT_MAX) {       // 100 ticks x 200 ms = 20 s
        this.stopRecordingProcess();
        context.getPromptAction().showToast({ message: $r('app.string.toast_longtime') });
      }
    }, AMPLITUDE_INTERVAL);
  } catch (error) {
    LOGGER.error(`Failed to create AVRecorder. ${error.code} ${error.message}`);
  }
}
```

**`getAudioCapturerMaxAmplitude` returns the peak since the previous call,** so
the poll interval *is* the sampling window - 200 ms here. It is a promise, and
it is correctly caught; but note the interval keeps firing whether or not the
recorder is still alive, which is why clearing it is part of teardown and not an
afterthought.

**The tick counter doubles as the recording cap.** One timer, two jobs: sample
the level and count elapsed time. `TICK_COUNT_MAX * AMPLITUDE_INTERVAL` is the
limit - change either constant and the limit moves.

**Driving the waveform — `entry/src/main/ets/pages/OralTrainingPage.ets`** (as shipped)

```typescript
await this.avRecorderManager?.startRecordingProcess(this.getUIContext(), index, (amplitude) => {
  this.getUIContext().animateTo({
    curve: Curve.EaseInOut,
    duration: AMPLITUDE_INTERVAL        // the animation spans exactly one sample interval
  }, () => {
    this.amplitude = amplitude / AMPLITUDE_MAX * AMPLITUDE_MAX_HEIGHT;
  });
});
```

**Matching the animation duration to the sample interval is the whole trick.**
The level arrives as a step every 200 ms; wrapping the assignment in an
`animateTo` of exactly 200 ms turns the steps into a continuous line, with each
animation ending as the next sample arrives. Shorter and the bars stutter;
longer and they lag behind the voice.

`AMPLITUDE_MAX = 32767` is the 16-bit PCM ceiling, so the division normalises to
0-1 before scaling to the bar height.

**One player, two source kinds — `entry/src/main/ets/media/AVPlayerManager.ets`** (corrected, see `HW-04-0078`, `HW-04-0079`)

```typescript
async prepareRecordAudio(filePath: string): Promise<void> {
  await this.stopAudio();
  this.avPlayer = await media.createAVPlayer();
  this.setAVPlayerCallback();
  const file = await fileIo.open(filePath);
  this.fileFd = file.fd;
  this.avPlayer.fdSrc = { fd: file.fd };                 // sandbox file: fd alone
  AppStorage.setOrCreate('recordStatus', RecordStatus.PLAYING_RECORD);
}

async prepareExampleAudio(rm: resourceManager.ResourceManager, filePath: string): Promise<void> {
  await this.stopAudio();
  this.avPlayer = await media.createAVPlayer();
  this.setAVPlayerCallback();
  const file = rm.getRawFdSync(filePath);
  this.rawFdPath = filePath;                             // FIX: sample tracks nothing here
  this.avPlayer.fdSrc = { fd: file.fd, offset: file.offset, length: file.length };
  AppStorage.setOrCreate('recordStatus', RecordStatus.PLAYING_EXAMPLE);
}

async stopAudio() {
  if (!this.avPlayer) { return; }
  if (this.avPlayer.state === 'playing' || this.avPlayer.state === 'paused') {
    await this.avPlayer.stop();
  }
  await this.avPlayer.release();
  this.avPlayer = undefined;                             // FIX: absent in the sample
  if (this.fileFd) { await fileIo.close(this.fileFd); this.fileFd = 0; }
  if (this.rawFdPath) {                                  // FIX: absent in the sample
    this.resourceManager?.closeRawFdSync(this.rawFdPath);
    this.rawFdPath = undefined;
  }
}
```

**`offset` and `length` are what make a rawfile playable.** A rawfile is not a
standalone file - it is a region inside the HAP, so its descriptor points at the
package and the triple `{fd, offset, length}` is the only way to say which part.
Pass only `fd` and the player reads the whole HAP. Correspondingly, that
descriptor is released with `closeRawFdSync(path)`, not `fileIo.close(fd)` - the
sample never releases it at all.

**`stopAudio` not nulling `this.avPlayer` is the bug that compounds.** The
`stateChange` 'stopped' handler also releases and nulls, so on the paths where
`stop()` runs both fire; on the paths where it does not (a player that was never
playing), the field survives holding a released instance and `releasePlayer`
releases it again from `aboutToDisappear`.

**The waveform — `entry/src/main/ets/components/AudioWaveComponent.ets`** (corrected, see `HW-04-0081`)

```typescript
@Component
export struct AudioWaveComponent {
  @Link amplitude: number;
  @State barHeights: number[] = new Array<number>(WAVE_BAR_COUNT).fill(AMPLITUDE_MIN_HEIGHT);
  private random: cryptoFramework.Random = cryptoFramework.createRandom();

  // FIX: the sample calls getWaveHeight() from inside build(), so every re-render
  // re-randomises every bar. Recompute on amplitude change instead.
  @Watch('onAmplitudeChange') ...
  onAmplitudeChange() {
    this.barHeights = this.barHeights.map(() => this.getWaveHeight(this.amplitude));
  }

  getWaveHeight(amplitude: number): number {
    const randData = this.random.generateRandomSync(1);
    return Math.min(AMPLITUDE_MAX_HEIGHT,
      Math.max(AMPLITUDE_MIN_HEIGHT,
        amplitude * (1 + randData.data[0] / MAX_BYTE_VALUE + AMPLITUDE_HEIGHT_OFFSET)));
  }

  build() {
    Row({ space: 4 }) {
      ForEach(this.barHeights, (h: number, i: number) => {
        Column().width(3).height(h).borderRadius(1.5)
      }, (h: number, i: number) => i.toString())
    }
  }
}
```

**The per-bar jitter formula is worth reading.** `AMPLITUDE_HEIGHT_OFFSET` is
`-0.9`, so the multiplier is `1 + r/255 - 0.9`, i.e. between 0.1 and 1.1 - each
bar gets 10 %-110 % of the measured level, clamped to `[2, 46]`. That is what
makes 30 bars from one number look like a waveform rather than a bar chart.

But it must not run inside `build`. ArkUI re-runs `build` on any bound change,
and a random draw there means the bars jitter on renders that have nothing to do
with the microphone - and `animateTo` cannot interpolate toward a target that is
re-rolled underneath it.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.MICROPHONE",
    "reason": "$string:microphone_reason",
    "usedScene": {
      "abilities": ["EntryAbility"],     // FIX: sample says ["FormAbility"], which does not exist
      "when": "inuse"
    }
  }
]
```

- `MICROPHONE` is `user_grant`, so `reason` and `usedScene` are mandatory and the
  reason resource must exist.
- `when: "inuse"` is right - this app records only in the foreground and declares
  no continuous task.
- The runtime request must handle a permanent refusal via `dialogShownResults` +
  `requestPermissionOnSetting`; the sample only toasts (`HW-04-0082`).
- `module.json5` also declares `"routerMap": "$profile:router_map"` pointing at an
  **empty** route map, with a filename that differs from the rest of the industry
  (`HW-04-0084`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Recordings are capped at 20 s** (`TICK_COUNT_MAX 100` x `AMPLITUDE_INTERVAL
  200 ms`) and self-stop with a toast.
- Output is fixed: AAC in an M4A container, 48 kHz stereo at 112 kbps, written to
  `filesDir/record_<index>.m4a`. Index is the sentence position, so re-recording
  a sentence overwrites the previous take and recordings from different exercises
  collide on the same names.
- The exercise is a bundled rawfile JSON read once in `aboutToAppear`; there is
  no scoring, no speech recognition and no comparison - "compare" here means the
  student listens to both.
- `AMPLITUDE_MAX` is hard-coded to the 16-bit PCM ceiling; a different
  `audioSampleFormat` would need a different divisor.
- Only the selected sentence shows controls; the rest are text rows.
- Nothing persists across launches - `recordFiles` is page state, though the
  `.m4a` files remain in the sandbox.

## Pitfalls

- **`HW-04-0076` — `usedScene.abilities` names `FormAbility`,** copied from the
  guide's example. The project has no such ability; it should name
  `EntryAbility`.
- **`HW-04-0077` — the recorder is torn down on two paths.** `stop()` triggers
  the `stateChange` handler *during* the await, so `stopRecordingProcess` then
  releases an already-released, already-nulled recorder - and the handler's own
  path clears neither the amplitude interval nor the file descriptor.
- **`HW-04-0078` — `stopAudio` releases the player but leaves the field set,**
  so the next `releasePlayer` releases it again.
- **`HW-04-0079` — the rawfile descriptor from `getRawFdSync` is never
  released.** Only the sandbox path assigns `this.fileFd`, and rawfile
  descriptors need `closeRawFdSync`, not `fileIo.close`.
- **`HW-04-0080` — `onPageHide`'s player guard is `A && B || C`,** so the null
  check covers only one of the two playing states. Optional chaining is what
  hides it.
- **`HW-04-0081` — the waveform is randomised inside `build`,** making the render
  non-deterministic. The trailing `+ i * 0` is dead code from an indexed variant.
- **`HW-04-0082` — the permission is re-requested on every record tap** with no
  `checkAccessToken` fast path and no `requestPermissionOnSetting` fallback, so a
  permanently refused microphone leaves the feature dead with only a toast.
- **`HW-04-0083` — the documented `startRecordingProcess` signature is missing
  its `UIContext` parameter,** and the documented call site passes two arguments
  to a three-parameter method. The snippet also drops the `.catch` and the
  recording cap.
- **`HW-04-0084` — an empty `router_map.json` is declared and packaged,** under a
  filename that differs from every other sample in this industry.
- **Do not close the recording fd before `release()`.** The recorder writes
  through it; closing early truncates the file.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-media-avrecorder.md` - the recorder state machine, `AVRecorderConfig`, `getAudioCapturerMaxAmplitude`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avrecorder
- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - `fdSrc`, `AVFileDescriptor`, the player state machine
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getRawFdSync` / `closeRawFdSync` and the `{fd, offset, length}` triple
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` - `checkAccessToken`, `requestPermissionsFromUser`, `requestPermissionOnSetting`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-references/02_application-framework/js-apis-permissionrequestresult.md` - `authResults` and `dialogShownResults`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-permissionrequestresult
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `reason` and `usedScene.abilities`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `ohos.permission.MICROPHONE`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `documentation/harmonyos-guides/04_system/crypto-generate-random-number.md` - `cryptoFramework.Random`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/crypto-generate-random-number
- `EDU-01` - the `AVPlayer` course-video player, for the same state machine on the video side
- `EDU-10` - the other timer-driven animation in this industry, with the same `build`-purity issue
