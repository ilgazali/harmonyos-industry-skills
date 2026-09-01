---
id: OFFICE-21
title: Voice note record and playback - AudioCapturer to a PCM file, AudioRenderer back out, responsive across breakpoints
industry: 05_office
doc: huawei_industry_tree/05_office/docs/21_voice_insert_note.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_insert_note-0000002348459564
sample: huawei_industry_tree/05_office/downloads/VoiceInsertNote.zip
kits: ["@kit.AudioKit", "@kit.CoreFileKit", "@kit.ArkUI", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["audio.createAudioCapturer", "AudioCapturer.start", "AudioCapturer.stop", "AudioCapturer.release", "AudioCapturer.on('readData')", "AudioCapturer.off('readData')", "AudioCapturer.on('stateChange')", "AudioCapturer.off('stateChange')", "audio.AudioCapturerOptions", "audio.AudioStreamInfo", "audio.AudioCapturerInfo", "audio.SourceType", "audio.createAudioRenderer", "AudioRenderer.start", "AudioRenderer.pause", "AudioRenderer.stop", "AudioRenderer.release", "AudioRenderer.flush", "AudioRenderer.state", "AudioRenderer.on('writeData')", "AudioRenderer.off('writeData')", "AudioRenderer.on('stateChange')", "audio.AudioRendererOptions", "audio.AudioRendererInfo", "audio.StreamUsage", "audio.AudioState", "fileIo.open", "fileIo.write", "fileIo.readSync", "fileIo.statSync", "fileIo.close", "fileIo.closeSync", "UIContext.getMediaQuery", "MediaQuery.matchMediaSync", "MediaQueryListener.on('change')", "MediaQueryListener.off('change')", SideBarContainer, showSideBar, TextTimer, TextTimerController, "abilityAccessCtrl.createAtManager", "AtManager.checkAccessToken", "AtManager.requestPermissionsFromUser", "AtManager.requestPermissionOnSetting", "bundleManager.getBundleInfoForSelf", "AppStorage.setOrCreate", "@StorageLink"]
permissions: ["ohos.permission.MICROPHONE"]
min_api: 20
modules: [entry]
findings: [HW-05-0116, HW-05-0117, HW-05-0118, HW-05-0119, HW-05-0120, HW-05-0121, HW-05-0122, HW-05-0123, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when a note app needs an **in-app voice memo** - record from the
microphone, show a live level meter, then play the recording back with
play/pause and a timer - and the same screen has to work on a phone and a
tablet.

The audio path is deliberately low-level. Rather than `AVRecorder`/`AVPlayer`,
this scenario uses **`AudioCapturer` and `AudioRenderer`**, which hand raw PCM
buffers to and from the application:

- `AudioCapturer.on('readData', cb)` delivers each captured buffer; the app
  appends it to a `.pcm` file in `cacheDir` and computes an RMS value from the
  same samples to drive the waveform.
- `AudioRenderer.on('writeData', cb)` asks the app to fill a buffer; the app
  reads the next slice out of the `.pcm` file at its own offset.

That is what makes seeking, the level meter and the waveform possible - and it
is also why the file-descriptor and state handling in this sample carries most
of its defects.

## Feature checklist

The application must be able to:

- Track the breakpoint with media queries across width **and** height, and pick a
  phone, portrait-tablet or landscape-tablet layout accordingly.
- Request the microphone permission before recording, starting with the ordinary
  authorization dialog.
- Record from the microphone into a PCM file in the app cache.
- Compute a decibel value per buffer and keep the series for the waveform.
- Play the recording back, pause it, and resume from the paused offset.
- Restart from the beginning once the end is reached.
- Seek by setting a read offset and flushing the renderer.
- Show elapsed time with a `TextTimer` controller kept in step with playback.
- Expand and collapse a side bar with `showSideBar`.
- Release the capturer, the renderer, both file descriptors and every listener.

## Architecture

Single `entry` HAP, with the layout split into one page per breakpoint:

| File | Responsibility |
| --- | --- |
| `pages/MainPage.ets` | entry; chooses `MainSM` / `MainMD` / `MainLG` from the current breakpoint |
| `pages/MainSM.ets`, `MainMD.ets`, `MainLG.ets` | phone, portrait-tablet and landscape-tablet layouts |
| `pages/Sidebar.ets`, `MainTitle.ets`, `ShowPage.ets`, `MenuBuilder.ets`, `SheetTransition.ets` | side bar, title, list, menu popup, half-modal sheet |
| `pages/NotePage.ets`, `NoteCavas.ets` | the note body and its drawing canvas |
| `utils/AudioCapturer.ets` | recording: options, `createAudioCapturer`, the `readData` handler, the level maths |
| `utils/AudioRendererManager.ets` | playback: `createAudioRenderer`, the `writeData` handler, start/pause/stop/release, seek |
| `utils/BreakpointSystem.ets` | six media-query listeners plus a `BreakpointType<T>` helper |
| `utils/PermissionsCheck.ets` | microphone grant check and request |
| `constants/`, `model/VoiceModel.ets` | breakpoint ranges, audio constants, the note model |

The two audio classes are symmetric and both keep their own file handle:

```
AudioCapturer                              AudioRendererManager
  audioCapturerOptions {                     audioRendererOptions {
    streamInfo  16 kHz, mono, S16LE, RAW       streamInfo  Constants.SAMPLING_RATE etc.
    capturerInfo SOURCE_TYPE_MIC               rendererInfo STREAM_USAGE_MUSIC
  }                                          }
  open(cacheDir/<name>.pcm, READ_WRITE|CREATE)  open(cacheDir/<name>.pcm, READ_ONLY)
  on('readData')  -> write(fd, buffer, {offset})  on('writeData') -> readSync(fd, buffer, {offset})
                  -> RMS accumulate                              -> RMS accumulate
```

Playback position is an explicit `readOffset` the app advances itself, published
to `AppStorage` as `RWOffset`; `setReadOffset(n)` plus `renderer.flush()` is the
seek. `AudioAtEnd` is set when the offset reaches the file size, and the offset
resets to 0 so the next start replays from the beginning.

The breakpoint system listens on six queries - `sm` / `md` / `lg` by width and
`h_sm` / `h_md` / `h_lg` by combined width and height - and
`BreakpointType.getValue` strips the `h_` prefix before matching, so a height
breakpoint falls back to the matching width value.

## Implementation steps

1. **Declare `ohos.permission.MICROPHONE`** with a `usedScene`. It is a
   `user_grant` permission, so a runtime request is mandatory.
2. **Request it in two stages.** `requestPermissionsFromUser` first, and only
   escalate to `requestPermissionOnSetting` afterwards - the reference forbids
   calling the settings dialog before the ordinary one (HW-05-0123).
3. **Configure the capturer.** `AudioStreamInfo` with
   `SAMPLE_RATE_16000`, `CHANNEL_1`, `SAMPLE_FORMAT_S16LE`, `ENCODING_TYPE_RAW`,
   and `AudioCapturerInfo` with `SOURCE_TYPE_MIC`.
4. **Open the PCM file once and stream into it.** `fileIo.open(cacheDir +
   '/<name>.pcm', READ_WRITE | CREATE)`, then in `on('readData')` write each
   buffer at a running offset and accumulate the squared samples for the level.
5. **Close the recording file exactly once.** Either in the `STATE_RELEASED`
   branch of `on('stateChange')` or in `stopAndRelease` - not both - and leave the
   fd field `undefined` until a file is actually opened (HW-05-0116).
6. **Configure the renderer** with the matching stream info and
   `STREAM_USAGE_MUSIC`, and register `on('writeData')` to fill each buffer from
   the file.
7. **Guard every renderer transition.** Restore the state checks whose bodies are
   empty in the sample, so `start`, `pause`, `stop` and `release` are only issued
   from the states that permit them (HW-05-0117).
8. **Reopen after a stop.** Clear the `playFile` field whenever its descriptor is
   closed, otherwise the "same path, no reopen" shortcut hands the renderer a
   closed descriptor (HW-05-0118).
9. **Never pass an optional-chained fd** to `readSync` or `closeSync`
   (HW-05-0119).
10. **Release every listener.** `off('readData')`, `off('stateChange')` on the
    capturer and `off('writeData')`, `off('stateChange')` on the renderer
    (HW-05-0120), plus the six media-query listeners - using the handles stored at
    registration time (HW-05-0122).
11. **Drive the timer from the same state as the audio.** `TextTimerController`
    `start` / `pause` / `reset` alongside the renderer calls, replaying from zero
    when the elapsed time has reached the total.

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### Capturer configuration and the readData path

`VoiceInsertNote.zip#entry/src/main/ets/utils/AudioCapturer.ets`

```ts
// Audio stream information
private audioStreamInfo: GeneratedObjectLiteralInterface_1 = {
  samplingRate: audio.AudioSamplingRate.SAMPLE_RATE_16000,
  channels: audio.AudioChannel.CHANNEL_1,
  sampleFormat: audio.AudioSampleFormat.SAMPLE_FORMAT_S16LE,
  encodingType: audio.AudioEncodingType.ENCODING_TYPE_RAW
};
// Audio collector information
private audioCapturerInfo: GeneratedObjectLiteralInterface_2 = {
  source: audio.SourceType.SOURCE_TYPE_MIC,
  capturerFlags: 0
};
private audioCapturerOptions: GeneratedObjectLiteralInterface_3 = {
  streamInfo: this.audioStreamInfo,
  capturerInfo: this.audioCapturerInfo
};

// Create an instance and start listening
createOn(filename: string, context: common.UIAbilityContext) {
  audio.createAudioCapturer(this.audioCapturerOptions, async (err, data) => {
    if (err) {
      hilog.error(DOMAIN, TAG, `Invoke createAudioCapturer failed, code is ${err.code}, message is ${err.message}`);
    } else {
      this.audioCapturer = data;
      this.savedDbs = [];
      let bufferSize: number = 0;

      let path = context.cacheDir;
      let filePath = path + `/${filename}.pcm`;
      let file: fileIo.File = await fileIo.open(filePath, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
      this.fd = file.fd;
      this.url = 'fd://' + file.fd;
      let readDataCallback = async (buffer: ArrayBuffer) => {
        let options: Options = {
          offset: bufferSize,
          length: buffer.byteLength
        };
        await fileIo.write(file.fd, buffer, options);
        bufferSize += buffer.byteLength;
        AppStorage.setOrCreate('RecordOffset', bufferSize);

        let samples = new Int16Array(buffer);
        for (let i = 0; i < samples.length; i++) {
          let val = samples[i] / Constants.VOLUME_MAX;
          this.sampleValSum += val * val;
          this.sampleValCnt += 1;
        }
      };
      this.dataCallBack = readDataCallback;          // written, never read - HW-05-0121

      this.audioCapturer.on('readData', readDataCallback);        // no off - HW-05-0120
      this.audioCapturer.on('stateChange', (state: audio.AudioState) => {
        if (state === audio.AudioState.STATE_RELEASED) {
          fileIo.close(file);                        // second close - HW-05-0116
        }
      });
      this.startRecording();
    }
  });
}
```

The level maths is worth keeping - it accumulates the sum of squares in the audio
callback and converts to a bar height on demand, so the UI can sample it at its
own rate:

```ts
calculateDecibelHeight(): number {
  if (this.sampleValCnt === 0) {
    return 0;
  }
  let rms = this.sampleValSum / this.sampleValCnt;
  let db = Math.max(Constants.MIN_DB, Math.min(0, 20 * Math.log10(rms)));
  this.sampleValCnt = 0;
  this.sampleValSum = 0;
  let res = Math.max(2, (db + Math.abs(Constants.MIN_DB)) / Math.abs(Constants.MIN_DB) * Constants.WAVE_HEIGHT_RADIO);
  this.savedDbs.push(res);
  return res;
}
```

Teardown as shipped - the second close of the same descriptor:

```ts
// stop recording
async stopAndRelease() {
  if (this.audioCapturer) {
    try {
      await this.audioCapturer.stop();
      await this.audioCapturer.release();
      if (this.fd !== undefined) {                   // fd is initialised to 0 - HW-05-0116
        await fileIo.close(this.fd);
        this.fd = undefined;
      }
    } catch (err) {
      hilog.error(DOMAIN, TAG, '录音停止失败' + err.code + err.message);
    }
  }
}
```

### Renderer configuration and the writeData path

`VoiceInsertNote.zip#entry/src/main/ets/utils/AudioRendererManager.ets`

```ts
async initRenderer() {
  let audioStreamInfo: audio.AudioStreamInfo = {
    channels: Constants.CHANNEL_NUM,
    samplingRate: Constants.SAMPLING_RATE,
    sampleFormat: Constants.SAMPLE_FORMAT,
    encodingType: audio.AudioEncodingType.ENCODING_TYPE_RAW,
  };
  let audioRendererInfo: audio.AudioRendererInfo = {
    rendererFlags: 0,
    usage: audio.StreamUsage.STREAM_USAGE_MUSIC,
  };
  let audioRendererOptions: audio.AudioRendererOptions = {
    streamInfo: audioStreamInfo,
    rendererInfo: audioRendererInfo,
  };

  this.renderer = await audio.createAudioRenderer(audioRendererOptions);

  this.renderer.on('stateChange', (state: audio.AudioState) => { /* ... */ });   // no off - HW-05-0120

  this.renderer.on('writeData', (buffer: ArrayBuffer) => {
    let lastLen = this.fileSize - this.readOffset;
    let readLen = lastLen >= buffer.byteLength ? buffer.byteLength : lastLen;
    try {
      let actualReadLen = fileIo.readSync(this.playFile?.fd, buffer,           // optional fd - HW-05-0119
        { offset: this.readOffset, length: readLen });
      if (actualReadLen === 0) {
        this.pauseRenderer();
      }
    } catch (e) {
      throw new Error('Failed to read file');
    }

    this.readOffset += readLen;
    AppStorage.setOrCreate('RWOffset', this.readOffset);

    if (this.readOffset >= this.fileSize) {
      AppStorage.setOrCreate('AudioAtEnd', true);
      this.readOffset = 0;
    }

    let samples = new Int16Array(buffer);
    for (let i = 0; i < samples.length; i++) {
      let val = samples[i] / Constants.VOLUME_MAX;
      this.sampleValSum += val * val;
      this.sampleValCnt += 1;
    }
  });
}

setReadOffset(offset: number) {
  this.readOffset = offset;
  this.renderer?.flush();
}
```

`setReadOffset` + `flush()` is the seek idiom: move the app-side read cursor and
drop whatever the renderer has already buffered so playback jumps immediately.

The truncated `readLen` at the end of the file, the `AudioAtEnd` flag and the
offset reset to `0` are what give "play again from the start" for free.

### Renderer transitions - as shipped, with empty guards

`VoiceInsertNote.zip#entry/src/main/ets/utils/AudioRendererManager.ets`

```ts
async startRenderer(fileName?: string, context?: common.UIAbilityContext) {
  if (this.renderer === undefined) {
    throw new Error(`Realse AudioRederer at undefined state`);
  }
  let state = this.renderer.state;
  if (state === audio.AudioState.STATE_INVALID) {
    this.renderer = undefined;
    throw new Error(`Start AudioRenderer at invalid state.`);
  }
  if (state !== audio.AudioState.STATE_PREPARED && state !== audio.AudioState.STATE_STOPPED &&
    state !== audio.AudioState.STATE_PAUSED) {
  }                                                    // empty guard - HW-05-0117

  if (fileName) {
    let pathDir = context?.cacheDir;
    let filePath = pathDir + `/${fileName}.pcm`;
    if (this.playFile?.path !== filePath) {            // stale after stopRenderer - HW-05-0118
      if (this.playFile) {
        await fileIo.close(this.playFile);
        await this.renderer.flush();
      }
      this.playFile = await fileIo.open(filePath, fileIo.OpenMode.READ_ONLY);
    }
    this.fileSize = fileIo.statSync(filePath).size;
  }
  await this.renderer.start();
}

async stopRenderer() {
  // ... same shape, empty guard ...
  await this.renderer.stop();

  fileIo.closeSync(this.playFile?.fd);                // field not cleared - HW-05-0118
}
```

Corrected guard and teardown:

```ts
if (state !== audio.AudioState.STATE_PREPARED && state !== audio.AudioState.STATE_STOPPED &&
  state !== audio.AudioState.STATE_PAUSED) {
  hilog.warn(0x0000, TAG, 'start ignored, renderer state is %{public}d', state);
  return;
}

// stopRenderer
await this.renderer.stop();
if (this.playFile) {
  fileIo.closeSync(this.playFile);
  this.playFile = undefined;
}
```

### Play/pause wiring on the UI side

From the document's step 3, matching the sample's toggle:

```ts
Image(this.imagePlayUrl)
  .onClick(async () => {
    this.isPause = !this.isPause
    if (this.isPause) {
      await this.audioRendererMgr.startRenderer(this.fileName, this.context);
      this.imagePlayUrl = $r('app.media.ic_public_play');
      if (this.elapsedTime >= this.counter) {
        // Replay
        this.textTimerController.reset();
        this.textTimerController.start();
      } else {
        this.textTimerController.start();
        this.durationId = setInterval(() => {
        }, 1000);
      }
    } else {
      this.imagePlayUrl = $r('app.media.ic_public_pause');
      this.textTimerController.pause();
      await this.audioRendererMgr.pauseRenderer();
      clearInterval(this.durationId);
    }
  })
```

The `elapsedTime >= counter` test is the replay case: reset the timer before
starting it, because the renderer has already wrapped its `readOffset` back to
zero.

### Breakpoint listeners

`VoiceInsertNote.zip#entry/src/main/ets/utils/BreakpointSystem.ets`

```ts
public register(context: UIContext): void {
  this.smListener = context.getMediaQuery().matchMediaSync(BreakpointConstants.RANGE_SM);
  this.smListener.on('change', this.isBreakpointSM);
  this.mdListener = context.getMediaQuery().matchMediaSync(BreakpointConstants.RANGE_MD);
  this.mdListener.on('change', this.isBreakpointMD);
  this.lgListener = context.getMediaQuery().matchMediaSync(BreakpointConstants.RANGE_LG);
  this.lgListener.on('change', this.isBreakpointLG);
  this.hsmListener = context.getMediaQuery().matchMediaSync('(height<470vp)');
  this.hsmListener.on('change', this.isBreakpointHSM);
  this.hmdListener = context.getMediaQuery().matchMediaSync('(640vp<=width<1000vp) and (470vp<=height<630vp)');
  this.hmdListener.on('change', this.isBreakpointHMD);
  this.hlgListener = context.getMediaQuery().matchMediaSync('(1000vp<=width) and (630vp<=height<670vp)');
  this.hlgListener.on('change', this.isBreakpointHLG);
}

public unregister(context: UIContext): void {
  this.smListener = context.getMediaQuery().matchMediaSync(BreakpointConstants.RANGE_SM);  // new handle - HW-05-0122
  this.smListener.off('change', this.isBreakpointSM);
  // ... five more, each creating a fresh handle before calling off on it
}
```

Corrected unregister:

```ts
public unregister(): void {
  this.smListener?.off('change', this.isBreakpointSM);
  this.mdListener?.off('change', this.isBreakpointMD);
  this.lgListener?.off('change', this.isBreakpointLG);
  this.hsmListener?.off('change', this.isBreakpointHSM);
  this.hmdListener?.off('change', this.isBreakpointHMD);
  this.hlgListener?.off('change', this.isBreakpointHLG);
  this.smListener = this.mdListener = this.lgListener = null;
  this.hsmListener = this.hmdListener = this.hlgListener = null;
}
```

The handler shape is right, though: each callback is a **bound arrow-function
field**, so the same reference can be passed to both `on` and `off` - which is
exactly what makes a precise unsubscribe possible.

The height-aware fallback in the value lookup is also worth keeping:

```ts
getValue(currentPoint: string): T {
  if (currentPoint.startsWith('h')) {
    let split = currentPoint.split('_');
    currentPoint = split[1];          // 'h_md' -> 'md'
  }
  if (currentPoint === 'sm') {
    return this.options.sm as T;
  } else if (currentPoint === 'md') {
    return this.options.md as T;
  } else {
    return this.options.lg as T;
  }
}
```

### Permission check

`VoiceInsertNote.zip#entry/src/main/ets/utils/PermissionsCheck.ets`

```ts
async checkAccessToken(permission: Permissions): Promise<abilityAccessCtrl.GrantStatus> {
  let atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
  let grantStatus: abilityAccessCtrl.GrantStatus = abilityAccessCtrl.GrantStatus.PERMISSION_DENIED;
  let tokenId: number = 0;
  try {
    let bundleInfo: bundleManager.BundleInfo =
      await bundleManager.getBundleInfoForSelf(bundleManager.BundleFlag.GET_BUNDLE_INFO_WITH_APPLICATION);
    let appInfo: bundleManager.ApplicationInfo = bundleInfo.appInfo;
    tokenId = appInfo.accessTokenId;
    grantStatus = await atManager.checkAccessToken(tokenId, permission);
  } catch (error) {
    let err: BusinessError = error as BusinessError;
    hilog.error(0x000, 'testTag', `Failed to check access token  ${err.code}, message is ${err.message}`);
  }
  return grantStatus;
}

// Request permission when click - missing the first stage, see HW-05-0123
async requestPermission(context: common.UIAbilityContext) {
  let atManager = abilityAccessCtrl.createAtManager();
  try {
    await atManager.requestPermissionOnSetting(context, ['ohos.permission.MICROPHONE']);
  } catch (error) {
    hilog.error(0x0000, '', `get Permission error, error. Code: ${error.code}, message: ${error.message}`);
  }
}
```

The grant check is correct; use the two-stage ladder from OFFICE-18
(`ConferenceWindowChange.zip#entry/src/main/ets/utils/PermissionsUtils.ets`) for
the request half.

## Permissions & config

`VoiceInsertNote.zip#entry/src/main/module.json5`

```json5
"requestPermissions": [
  {
    "name": 'ohos.permission.MICROPHONE',
    "reason": '$string:EntryAbility_desc',
    "usedScene": {
      "abilities": ["EntryAbility"],
      "when": 'always'
    },
  }
]
```

Notes on the config:

- The declared set matches the document's 权限说明 section exactly - one
  permission, the microphone.
- `ohos.permission.MICROPHONE` is `user_grant`, so declaring it is not enough:
  the runtime request must run, and it must start with
  `requestPermissionsFromUser` (HW-05-0123).
- **No storage permission** is needed - the PCM file lives in
  `context.cacheDir`.
- **No media permission** either: the recording is never exported to the media
  library.
- `"when": 'always'` is broader than this scenario needs; recording only happens
  while the note page is in the foreground, which `"inuse"` describes.
- The document's project tree matches the shipped ZIP exactly, including the
  `NoteCavas.ets` spelling.
- `build-profile.json5` pins the SDK to `6.0.0(20)`.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **Raw PCM, not a container.** The capture writes headerless 16 kHz mono S16LE
  samples, and playback must be configured with the *same* stream info - there is
  no format negotiation and no header to read it from.
- **The app owns the read cursor.** `AudioRenderer` does not seek; the
  application tracks `readOffset` itself, and `flush()` is what discards the
  already-buffered audio after a jump.
- **`writeData` runs on the audio thread.** Everything inside it - the
  `readSync`, the offset arithmetic, the RMS loop - happens per buffer, so it must
  stay cheap and must not throw.
- **Renderer transitions are state-dependent.** `start`, `pause`, `stop` and
  `release` are each legal only from particular `AudioState` values, which is what
  the four (empty) guards in this sample were meant to enforce.
- **`requestPermissionOnSetting` has a precondition.** Per the reference, the
  application must have called `requestPermissionsFromUser` first.
- **The breakpoint set is six-way.** Three width ranges plus three combined
  width/height ranges, with `h_*` values folded back onto their width equivalent
  by `getValue`.
- **`matchMediaSync` returns a new handle per call.** Unsubscribing requires the
  handle that was subscribed on, not a freshly created one.

## Pitfalls

- **Closing the recording file in both the `STATE_RELEASED` handler and
  `stopAndRelease` is incorrect**, and `private fd?: number = 0` makes the
  `!== undefined` guard pass before any file exists, so a stop before a record
  closes descriptor 0. Close in one place, and initialise the field to
  `undefined`. (HW-05-0116)
- **The four empty renderer state guards are incorrect.** Each condition
  enumerates exactly the legal source states and each body is empty, so `start`,
  `pause`, `stop` and `release` are issued unconditionally - reachable from the
  play/pause toggle. Restore the early returns. (HW-05-0117)
- **`stopRenderer` closing `playFile` without clearing the field is incorrect.**
  The next `startRenderer` for the same recording sees a matching `path`, skips
  the reopen, and the `writeData` callback reads from a closed descriptor.
  (HW-05-0118)
- **`fileIo.readSync(this.playFile?.fd, ...)` and
  `fileIo.closeSync(this.playFile?.fd)` are incorrect** - both parameters are
  mandatory and the optional chain yields `undefined` before any file is opened,
  which the `writeData` callback can observe. (HW-05-0119)
- **None of the four audio subscriptions is released, which is incorrect** -
  Audio Kit documents `off('readData')`, `off('stateChange')` and
  `off('writeData')`, and `createOn` runs once per recording. (HW-05-0120)
- **The document naming `setDataCallback` as the audio data path is incorrect** -
  the field it writes is never read and the function has no call site; the data
  flows through `audioCapturer.on('readData', ...)`. (HW-05-0121)
- **`unregister` creating fresh media-query handles before calling `off` on them
  is incorrect** - the six subscriptions made by `register` survive, and the
  fields that referenced them have been overwritten, so they can never be removed.
  (HW-05-0122)
- **Calling `requestPermissionOnSetting` without ever calling
  `requestPermissionsFromUser` is incorrect** - the reference states the ordinary
  dialog must have run first, so on a first launch the user is sent to Settings
  for a permission they were never asked for. (HW-05-0123)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/04_media/arkts-apis-audio-audiocapturer.md` -
  `on`/`off('readData')`, `on`/`off('stateChange')`, `start`, `stop`, `release`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio-audiocapturer
- `documentation/harmonyos-references/04_media/arkts-apis-audio-audiorenderer.md` -
  `on`/`off('writeData')`, `start`, `pause`, `stop`, `release`, `flush`, `state`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio-audiorenderer
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` -
  `requestPermissionOnSetting12+` and its precondition "Before calling this API,
  the application must have called requestPermissionsFromUser."
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `readSync(fd: number, ...)`, `closeSync(file: number | File)`, `open`, `write`,
  `statSync`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-mediaquery.md` -
  `matchMediaSync(condition: string)` returning "the corresponding listening
  handle".
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-mediaquery
- `documentation/harmonyos-references/02_application-framework/ts-container-sidebarcontainer.md` -
  `SideBarContainer` and `showSideBar`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-sidebarcontainer
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `usedScene`
  and the `inuse` / `always` values.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
