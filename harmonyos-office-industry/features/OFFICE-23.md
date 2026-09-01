---
id: OFFICE-23
title: Live speech-to-text notes - AudioCapturer PCM fed into a Core Speech Kit recognition engine
industry: 05_office
doc: huawei_industry_tree/05_office/docs/23_voice_input_notes.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_input_notes-0000002351441712
sample: huawei_industry_tree/05_office/downloads/VoiceInputNotes.zip
kits: ["@kit.CoreSpeechKit", "@kit.AudioKit", "@kit.CoreFileKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["speechRecognizer.createEngine", "speechRecognizer.CreateEngineParams", "SpeechRecognitionEngine.setListener", "SpeechRecognitionEngine.startListening", "SpeechRecognitionEngine.writeAudio", "SpeechRecognitionEngine.finish", "SpeechRecognitionEngine.shutdown", "speechRecognizer.RecognitionListener", "speechRecognizer.SpeechRecognitionResult", "speechRecognizer.StartParams", "speechRecognizer.AudioInfo", "audio.createAudioCapturer", "AudioCapturer.start", "AudioCapturer.stop", "AudioCapturer.release", "AudioCapturer.on('readData')", "AudioCapturer.off('readData')", "AudioCapturer.on('stateChange')", "AudioCapturer.off('stateChange')", "audio.AudioCapturerOptions", "audio.SourceType", "audio.AudioState", "audio.createAudioRenderer", "AudioRenderer.on('writeData')", "fileIo.open", "fileIo.write", "fileIo.close", "abilityAccessCtrl.createAtManager", "AtManager.requestPermissionsFromUser", "AtManager.checkAccessToken", "bundleManager.getBundleInfoForSelf", Slider, "AppStorage.setOrCreate", "@StorageProp", "@Watch"]
permissions: ["ohos.permission.MICROPHONE"]
min_api: 20
modules: [entry]
findings: [HW-05-0129, HW-05-0130, HW-05-0131, HW-05-0132, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when a note app must **transcribe speech while it is being
spoken** - a meeting recorder that fills the note as people talk, with the
partial text updating live and the confirmed text appended sentence by
sentence.

The pattern is a two-consumer audio fan-out:

```
microphone -> AudioCapturer.on('readData') -> buffer
                                              |-> write to a .pcm file   (so it can be replayed)
                                              '-> writeAudio(sessionId)  (so it can be transcribed)
```

Core Speech Kit does **not** open the microphone itself. The application
captures the PCM and pushes it into the engine with `writeAudio`, which is what
makes the file and the transcript come from the same stream and stay in sync.

Recognition results arrive on a listener with an `isFinal` flag: non-final
results are the live preview, final ones are appended to the durable transcript.

## Feature checklist

The application must be able to:

- Request the microphone permission and re-check the grant before recording.
- Create a recognition engine for the target language and mode, and set a result
  listener on it.
- Start a recognition session with the PCM format the capturer produces.
- Capture microphone audio and deliver **each buffer to both** the sandbox file
  and the recognition engine.
- Show the live partial result and accumulate the final results into the note.
- Stop the session, flush the final result, shut the engine down, and be able to
  start a new session afterwards.
- Release the capturer, its file descriptor and its listeners.
- Play the recorded PCM back.

## Architecture

Single `entry` HAP:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | insets, `requestPermissionsFromUser` for the microphone at start-up |
| `pages/NotesPage.ets` | the note screen |
| `components/AudioToText.ets` | the recording sheet: owns **the** `AudioCapturer` and a `SpeechRecognizer`, drives start/stop, renders the transcript |
| `media/AudioCapturer.ets` | capture: options, `createOn`, the `readData` handler, level maths, `setDataCallback` |
| `media/AudioRendererManager.ets` | playback of the recorded `.pcm` |
| `utils/SpeechRecognizer.ets` | engine creation, listener, `startListening`, the `writeAudio` feed, `stop` |
| `utils/PermissionsCheck.ets` | grant check only (`checkPermissions` / `checkAccessToken`) |
| `utils/Utils.ets`, `common/Constants.ets`, `common/FunctionButton.ets` | helpers, constants, toolbar model |

The intended wiring is a callback handshake between the two media classes:

```ts
// AudioCapturer exposes a sink
setDataCallback(dataCallBack: (data: ArrayBuffer) => void) { this.dataCallBack = dataCallBack; }

// SpeechRecognizer registers itself as that sink
this.audioCapturer.setDataCallback((dataBuffer: ArrayBuffer) => {
  const uint8Array: Uint8Array = new Uint8Array(dataBuffer);
  this.asrEngine?.writeAudio(this.sessionId, uint8Array);
});
```

**As shipped that handshake is broken at both ends** - the recogniser registers
on a second capturer instance that is never started, and the capturer's
`readData` handler never invokes the stored sink (HW-05-0129). Everything below
describes the intended design with the correction applied.

Session flow:

```
record button
  PermissionsCheck.checkPermissions(['ohos.permission.MICROPHONE'])
  speechRecognizer.createByCallback()      -> createEngine -> setListener
  audioCapturer.createOn(filename, uiContext)
       open cacheDir/<name>.pcm (READ_WRITE|CREATE)
       on('readData')  -> write to file, accumulate level, forward to the sink
       on('stateChange')
       start()
  speechRecognizer.startRecording()
       startListening({ sessionId, audioInfo: pcm 16 kHz mono 16-bit, extraParams })
       setDataCallback(buffer => writeAudio(sessionId, new Uint8Array(buffer)))

onResult(sessionId, result)
  result.isFinal ? recognitionResult += result.result : generatedText = result.result
  AppStorage['IsFinal'] = result.isFinal

stop button
  speechRecognizer.stop()      -> finish(sessionId) + shutdown() + clear the reference
  audioCapturer.stopAndRelease()
```

## Implementation steps

1. **Declare `ohos.permission.MICROPHONE`** and request it at start-up with
   `requestPermissionsFromUser`; re-check the grant with `checkAccessToken`
   before each recording, which the sample does correctly.
2. **Create the engine.** `speechRecognizer.createEngine({ language: 'zh-CN',
   online: 1, extraParams: { locate: 'CN', recognizerMode: 'long' } }, cb)` and
   keep the returned `SpeechRecognitionEngine`.
3. **Set the listener before starting.** `onStart` clears both text buffers,
   `onResult` splits partial from final, `onError` reports the code, `onComplete`
   and `onEvent` log.
4. **Match the audio formats.** `StartParams.audioInfo` must describe exactly
   what the capturer produces - `audioType: 'pcm'`, `sampleRate: 16000`,
   `soundChannel: 1`, `sampleBit: 16` against the capturer's
   `SAMPLE_RATE_16000` / `CHANNEL_1` / `SAMPLE_FORMAT_S16LE`.
5. **Share one capturer.** Inject the instance the UI starts into the recogniser
   instead of constructing a second one (HW-05-0129).
6. **Fan the buffers out.** In the capturer's `readData` handler write to the
   file **and** call the registered sink, so the recogniser sees every buffer
   (HW-05-0129).
7. **Convert before writing.** `writeAudio` takes a `Uint8Array`, so wrap the
   `ArrayBuffer` at the call site.
8. **Close the recording file once.** Either in the `STATE_RELEASED` branch or in
   `stopAndRelease`, not both, and leave the fd field `undefined` until a file is
   opened (HW-05-0131).
9. **Release both capturer listeners** before releasing the capturer
   (HW-05-0132).
10. **End the session properly.** `finish(sessionId)` to flush the final result,
    then `shutdown()`, then clear the engine reference so the next session
    re-creates it (HW-05-0130).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### Engine creation and the result listener

`VoiceInputNotes.zip#entry/src/main/ets/utils/SpeechRecognizer.ets`

```ts
import { speechRecognizer } from '@kit.CoreSpeechKit';

private sessionId: string = '123456';
asrEngine: speechRecognizer.SpeechRecognitionEngine | undefined = undefined;
generatedText: string = '';        // live partial result
recognitionResult: string = '';    // accumulated final result

// Create an engine and return it via a callback.
createByCallback() {
  let extraParam: Record<string, Object> = { 'locate': 'CN', 'recognizerMode': 'long' };
  let initParamsInfo: speechRecognizer.CreateEngineParams = {
    language: 'zh-CN',
    online: 1,
    extraParams: extraParam
  };

  speechRecognizer.createEngine(initParamsInfo, (err: BusinessError, speechRecognitionEngine:
    speechRecognizer.SpeechRecognitionEngine) => {
    if (!err) {
      this.asrEngine = speechRecognitionEngine;
      this.setListener();
    } else {
      // 1002200001 unsupported language/mode, init timeout, missing resources
      // 1002200006 engine busy - another application is using it
      // 1002200008 the engine has been destroyed
      hilog.error(DOMAIN, TAG, `Failed to create engine. Message: ${err.message}.`);
    }
  });
}

// Set Callback
setListener() {
  let setListener: speechRecognizer.RecognitionListener = {
    onStart: (sessionId: string, eventMessage: string) => {
      this.generatedText = '';
      this.recognitionResult = '';
    },
    onEvent(sessionId: string, eventCode: number, eventMessage: string) { /* ... */ },
    // Callback for recognition results, including intermediate and final outcomes
    onResult: (sessionId: string, result: speechRecognizer.SpeechRecognitionResult) => {
      if (result.isFinal) {
        this.recognitionResult += result.result;
      }
      this.generatedText = result.result;
      AppStorage.setOrCreate('IsFinal', result.isFinal);
    },
    onComplete(sessionId: string, eventMessage: string) { /* ... */ },
    // 1002200002 recognition failed at startup, triggered when restarting startListening
    onError(sessionId: string, errorCode: number, errorMessage: string) {
      hilog.error(DOMAIN, TAG,
        `onError, sessionId: ${sessionId} errorCode: ${errorCode} errorMessage: ${errorMessage}`);
    },
  };
  try {
    this.asrEngine?.setListener(setListener);
  } catch (e) {
    hilog.error(DOMAIN, TAG, `设置监听回调失败`);
  }
}
```

The `isFinal` split is the reusable core: partial results overwrite
`generatedText` so the UI shows a live caption, and only final segments are
appended to `recognitionResult`, which is what the note keeps. Note that
`onStart` and `onResult` are **arrow properties** so `this` binds to the class,
while `onEvent` / `onComplete` / `onError` are method shorthands that do not need
it.

The error codes documented in the comments are worth keeping: `1002200001`
unsupported configuration, `1002200006` engine busy (another app holds it),
`1002200008` engine already destroyed, `1002200002` restarting `startListening`.

### Session parameters

`VoiceInputNotes.zip#entry/src/main/ets/utils/SpeechRecognizer.ets`

```ts
startListeningForRecording() {
  let audioParam: speechRecognizer.AudioInfo = {
    audioType: 'pcm',
    sampleRate: 16000,
    soundChannel: 1,
    sampleBit: 16
  };
  let extraParam: Record<string, Object> = {
    'recognitionMode': 0,
    'vadBegin': 500,
    'vadEnd': 10000,
    // Maximum audio duration supported for recognition
    'maxAudioDuration': 8 * 60 * 60 * 1000
  };
  let recognizerParams: speechRecognizer.StartParams = {
    sessionId: this.sessionId,
    audioInfo: audioParam,
    extraParams: extraParam
  };
  this.asrEngine?.startListening(recognizerParams);
}
```

`vadBegin` / `vadEnd` are the voice-activity-detection windows in milliseconds -
`vadEnd: 10000` keeps a session alive through a ten-second pause, which is what
makes long-form meeting capture work rather than cutting off between sentences.
`maxAudioDuration` is set to eight hours for the same reason.

### The audio feed - as shipped

`VoiceInputNotes.zip#entry/src/main/ets/utils/SpeechRecognizer.ets`

```ts
private audioCapturer = new AudioCapturer();      // second instance - HW-05-0129

// Microphone Voice to Text
async startRecording() {
  try {
    this.startListeningForRecording();
    let data: ArrayBuffer;
    this.audioCapturer.setDataCallback((dataBuffer: ArrayBuffer) => {
      data = dataBuffer;
      let uint8Array: Uint8Array = new Uint8Array(data);
      // Write audio stream
      this.asrEngine?.writeAudio(this.sessionId, uint8Array);
    });
  } catch (err) {
    this.generatedText = `Message: ${err.message}.`;
  }
}
```

`VoiceInputNotes.zip#entry/src/main/ets/media/AudioCapturer.ets`

```ts
private dataCallBack: ((data: ArrayBuffer) => void) | null = null;

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
  // this.dataCallBack is never invoked - HW-05-0129
};
this.dataCallBack = readDataCallback;

this.audioCapturer.on('readData', readDataCallback);          // no off - HW-05-0132
this.audioCapturer.on('stateChange', (state: audio.AudioState) => {
  if (state === audio.AudioState.STATE_RELEASED) {
    fileIo.close(file);                                        // second close - HW-05-0131
  }
});

setDataCallback(dataCallBack: (data: ArrayBuffer) => void) {
  this.dataCallBack = dataCallBack;
  hilog.info(DOMAIN, 'testTag', '%{public}s', this.dataCallBack.length);
}
```

Corrected fan-out - one capturer, and the sink actually called:

```ts
// AudioCapturer
let readDataCallback = async (buffer: ArrayBuffer) => {
  const options: Options = { offset: bufferSize, length: buffer.byteLength };
  await fileIo.write(file.fd, buffer, options);
  bufferSize += buffer.byteLength;
  AppStorage.setOrCreate('RecordOffset', bufferSize);
  // ... level accumulation ...
  this.dataCallBack?.(buffer);        // forward to the registered consumer
};

// SpeechRecognizer takes the capturer the UI actually started
constructor(private audioCapturer: AudioCapturer) {}

// AudioToText
private audioCapturer: AudioCapturer = new AudioCapturer();
private speechRecognizer: SpeechRecognizer = new SpeechRecognizer(this.audioCapturer);
```

Also note the stray `hilog.info(..., '%{public}s', this.dataCallBack.length)` in
`setDataCallback`: it formats a number through a `%s` placeholder and logs a
function's arity, which is of no diagnostic value.

### Teardown - as shipped

`VoiceInputNotes.zip#entry/src/main/ets/utils/SpeechRecognizer.ets`

```ts
// Stop recognition
stop() {
  this.asrEngine?.shutdown();          // reference kept, session not finished - HW-05-0130
}
```

`VoiceInputNotes.zip#entry/src/main/ets/components/AudioToText.ets`

```ts
this.speechRecognizer.stop();
await this.audioCapturer.stopAndRelease();
```

Corrected:

```ts
stop() {
  this.asrEngine?.finish(this.sessionId);   // flush the final result
  this.asrEngine?.shutdown();
  this.asrEngine = undefined;               // force a fresh engine next time
}

async startRecording() {
  if (!this.asrEngine) {
    this.createByCallback();
  }
  // ...
}
```

### Permission gate

`VoiceInputNotes.zip#entry/src/main/ets/entryability/EntryAbility.ets`

```ts
let atManager = abilityAccessCtrl.createAtManager();
atManager.requestPermissionsFromUser(this.context, ['ohos.permission.MICROPHONE']).then((data) => {
  hilog.info(DOMAIN, 'testTag', 'data authResults:' + data.authResults);
}).catch((err: BusinessError) => {
  hilog.error(DOMAIN, 'testTag', 'errCode: ' + err.code + 'errMessage: ' + err.message);
});
```

`VoiceInputNotes.zip#entry/src/main/ets/components/AudioToText.ets`

```ts
let permissionAllowed = await PermissionsCheck.checkPermissions(this.permission);
```

The request result is only logged, but the UI **re-checks the grant with
`checkAccessToken` before recording**, so the gate is real - a better shape than
OFFICE-21, which skipped the first-stage request entirely (HW-05-0123), and than
OFFICE-01/OFFICE-11, which acted on a resolved promise without reading
`authResults`.

## Permissions & config

`VoiceInputNotes.zip#entry/src/main/module.json5`

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
- It is `user_grant`: the start-up `requestPermissionsFromUser` plus the
  per-recording `checkAccessToken` re-check are both required.
- **No network permission is declared**, although the engine is created with
  `online: 1`. Verify whether your recognition mode needs
  `ohos.permission.INTERNET` before shipping.
- **No storage permission** - the PCM file lives in `context.cacheDir`.
- `"when": 'always'` is broader than the scenario needs; recording only happens
  while the note sheet is in the foreground, which `"inuse"` describes.
- The document's project tree matches the shipped ZIP exactly.
- `build-profile.json5` pins the SDK to `6.0.0(20)`.

## Constraints

- **Core Speech Kit does not capture audio.** The application must feed it PCM
  through `writeAudio`; the engine has no microphone of its own, which is the
  reason for the whole fan-out design.
- **The formats must agree.** `StartParams.audioInfo` describes what the
  application will push, so it has to match the `AudioCapturer` stream info
  exactly - 16 kHz, mono, 16-bit PCM here.
- **`writeAudio` takes a `Uint8Array`**, not the `ArrayBuffer` the capturer
  delivers; the wrap is required at every call.
- **One engine at a time, system-wide.** Error `1002200006` is documented in the
  sample as "the engine is currently busy, typically triggered when multiple
  applications simultaneously invoke the speech recognition engine".
- **`shutdown()` destroys the engine.** Error `1002200008` is "the engine has been
  destroyed" - after shutting down, a new engine must be created before the next
  session.
- **Restarting `startListening` on a live session fails** with `1002200002`, per
  the comment in the sample's `onError`.
- **`vadEnd` bounds the silence tolerance** and `maxAudioDuration` the total
  session length; the sample sets 10 s and 8 h respectively.
- **The language is fixed at construction.** `language: 'zh-CN'` with
  `locate: 'CN'` and `recognizerMode: 'long'` are engine-creation parameters, so
  changing language means a new engine.

## Pitfalls

- **The audio never reaches the recognition engine, which is a blocker.** Two
  breaks compound: `SpeechRecognizer` registers its sink on a second
  `AudioCapturer` that is never started, and the capturer's `readData` handler
  never invokes `this.dataCallBack` at all - so `writeAudio` is unreachable and
  no transcript can ever be produced. Share one capturer and call the sink from
  the handler. (HW-05-0129)
- **`stop()` calling only `shutdown()` is incorrect.** The engine reference is
  kept, so the next recording calls `startListening` on a destroyed engine -
  exactly the `1002200008` / `1002200002` errors the file's own comments describe
  - and the session is never finished, so the last buffered segment is discarded.
  (HW-05-0130)
- **Closing the recording file in both the `STATE_RELEASED` handler and
  `stopAndRelease` is incorrect**, and `private fd?: number = 0` makes the
  `!== undefined` guard pass before any file exists, so a stop before a record
  closes descriptor 0. (HW-05-0131)
- **Neither capturer subscription is released, which is incorrect** - Audio Kit
  documents `off('readData')` and `off('stateChange')`, and `createOn` runs once
  per recording, so `readData` handlers accumulate. (HW-05-0132)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/04_media/arkts-apis-audio-audiocapturer.md` -
  `on`/`off('readData')`, `on`/`off('stateChange')`, `start`, `stop`, `release`
  and the capturer options.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio-audiocapturer
- `documentation/harmonyos-references/04_media/arkts-apis-audio-audiorenderer.md` -
  the renderer used for playing the recorded PCM back.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio-audiorenderer
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `open`, `write`, `close` and the "once closed, the File object or FD cannot be
  used" rule behind the double-close finding.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` -
  `requestPermissionsFromUser` and `checkAccessToken`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-references/07_ai/hms-ai-speechrecognizer.md` - the
  Core Speech Kit reference the document names for `setListener`, `writeAudio`,
  `startListening`, `finish` and `shutdown`; it is a stub in this repository, so
  the API behaviour above was taken from the sample's own error-code comments and
  could not be cross-checked against a signature list.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/hms-ai-speechrecognizer
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `usedScene`
  and the `inuse` / `always` values.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
