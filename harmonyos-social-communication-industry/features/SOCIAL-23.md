---
id: SOCIAL-23
title: Voice message to text - record PCM to the ASR contract, then replay the file into speechRecognizer
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/23_voice_to_text.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_to_text-0000002296107966
sample: huawei_industry_tree/14_social_communication/downloads/SpeechToText.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.AudioKit", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.CoreSpeechKit", "@kit.PerformanceAnalysisKit"]
apis: ["speechRecognizer.createEngine", setListener, startListening, writeAudio, finish, shutdown, "audio.createAudioCapturer", "audio.createAudioRenderer", readData, writeData, AudioStreamInfo, AudioSamplingRate, AudioSampleFormat, StreamUsage, SourceType, abilityAccessCtrl, checkAccessTokenSync, requestPermissionsFromUser, bundleManager, fs, "@ObservedV2", "@Trace", LongPressGesture, bindPopup, ImageAnimator, hilog, window]
permissions: [ohos.permission.MICROPHONE]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0053, HW-14-0054, HW-14-0055, HW-14-0056, HW-14-0018, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card when a chat carries **voice messages that a user cannot or does
not want to listen to** - in a meeting, in a noisy street, or for
accessibility. The pattern: record to exactly the format the recognizer
demands, keep the PCM file, and on demand replay that file into
`speechRecognizer` frame by frame, showing the transcript under the bubble.

The load-bearing idea is that recognition here is **file-based, not live**. The
capture and recognition paths never touch: the capturer writes a `.pcm` into
`cacheDir`, and much later - possibly on a message that arrived from someone
else - the recognizer reads that file in 1280-byte frames and pushes them
through `writeAudio`. That is the right shape whenever the audio outlives the
moment it was spoken: voice notes, voicemail, recorded meeting clips. For a
live subtitle you would instead feed `writeAudio` from the capturer's
`readData` callback and never touch the filesystem.

**Read `HW-14-0056` before adopting the UI wiring.** The transcript is routed
back to a bubble through a mutable "currently selected" index, so it can land
on the wrong message.

## Feature checklist

- A chat page alternating own and other bubbles, with a text input row and a
  voice input row toggled by a keyboard icon.
- Holding 按住说话 (hold to talk) starts recording and shows a 语音录制中
  (recording) popup; releasing stops it and appends a voice bubble with its
  duration in seconds.
- Releasing without the microphone permission shows a "go to settings" toast
  and appends nothing.
- Tapping a voice bubble plays it back and runs a three-frame waveform
  animation on that bubble only, for the bubble's own duration.
- Long-pressing a voice bubble opens a popup with 转文本 (to text), 听筒播放,
  引用, 多选, 删除.
- 转文本 creates the ASR engine if needed, streams the saved file through it,
  and renders the recognized text in a card under that bubble.
- Text messages typed in the input row appear as ordinary bubbles; empty sends
  are rejected with a toast.

## Architecture

One `entry` module: one page and three single-responsibility wrappers, each
around one system API.

```
entry/src/main/ets
├── entryability/EntryAbility.ets   full-screen layout, avoid areas; shuts down an ASR engine
├── entrybackupability/
├── models/Interface.ets            EditMenuAction enum (NONE / SPEECH / EMOJI)
├── pages/Speech.ets                @Entry, 626 lines: the whole chat and all wiring
└── utils
    ├── AudioCapturerU.ets          create -> listen readData -> write pcm -> start/stop
    ├── AudioRendererU.ets          create -> listen writeData -> read pcm -> start/stop
    ├── Logger.ets
    └── SpeechRecognizerU.ets       createEngine, setListener, writeAudio loop, message
```

The documented tree matches the zip exactly.

**The structural choice worth avoiding** is that each wrapper holds exactly
**one** live handle (`audioCapturer`, `audioRenderer`, `asrEngine`) while the
page treats the wrapper as a per-operation service: `Speech` owns one instance
of each as `@State` and calls `createPlayOn` / `createrOn` / `writeAudio` again
for every bubble the user touches. Every re-entrant use overwrites the single
handle, and every completion callback resolves `this.audioRenderer` at the time
it fires rather than the instance it was created for. Three of this card's
findings are that one mismatch (`HW-14-0053`, `HW-14-0054`, `HW-14-0055`).

The part that *is* worth copying is the message model. `OneVoice` is
`@ObservedV2` with `@Trace` on the three mutable fields:

```typescript
@ObservedV2
class OneVoice {
  filename: string;                 // the pcm basename == the send timestamp
  during: number;
  @Trace isShow: boolean;
  @Trace context: string;           // the transcript
  @Trace textMsg: string;           // set for plain text messages
}
```

`filename` doubles as the message identity and as the path to its audio - the
send timestamp names the `.pcm` in `cacheDir` - so playback and recognition
both need nothing but the bubble's own data, and `@Trace` on `context` makes a
transcript arriving from an asynchronous callback repaint one bubble rather
than the list.

## Implementation steps

1. **Configure the stream to the recognizer's contract, not to taste**:
   16 kHz, mono, `SAMPLE_FORMAT_S16LE`, `ENCODING_TYPE_RAW`. Core Speech Kit
   accepts nothing else, and the same four values must be repeated in
   `StartParams.audioInfo`.
2. **Record with `AudioCapturer`**: create it, open the file, register
   `readData` to `fs.writeSync` each buffer at a running offset, then `start()`.
   Track the offset yourself - `readData` does not.
3. **Guard the create/stop race with a cancelled flag** (`HW-14-0055`): the
   create callback is asynchronous, so a quick press-release can stop before
   there is anything to stop.
4. **Close the file from the `stateChange` callback** when the state reaches
   `STATE_RELEASED` - the sample's `state === 4`; prefer the named enum.
5. **Check the permission on release, not on press**: read
   `checkAccessTokenSync(tokenID, 'ohos.permission.MICROPHONE')` and only append
   the bubble if it is granted; the token id comes from
   `bundleManager.getBundleInfoForSelf`.
6. **Play back with `AudioRenderer` and the mirror-image loop**: `writeData`
   reads the file into the supplied buffer and stops when the offset passes the
   file size. **Stop the previous renderer before creating a new one, and
   capture the instance inside the callback** (`HW-14-0054`).
7. **Create the ASR engine lazily**, on the first long press, and reuse it -
   `createEngine` fails with `1002200006` while another engine is busy.
8. **Stream the file through `writeAudio` in 1280-byte frames** with a ~40 ms
   pause between frames, then call `finish(sessionId)` in the `finally`.
9. **Bind the result to the submitting bubble's index** (`HW-14-0056`), and
   **shut down the page's engine, not a fresh one** (`HW-14-0053`).

## Verified snippets

All snippets are from `SpeechToText.zip`. Corrected forms are marked.

**The recording contract - `entry/src/main/ets/utils/AudioCapturerU.ets`** (corrected, see `HW-14-0055`)

```typescript
export class AudioCapturerU {
  audioCapturer: audio.AudioCapturer | undefined = undefined;
  cancelled: boolean = false;                       // FIX: absent in the sample
  url: string = '';
  time1: number = 0;
  time2: number = 0;

  createrOn(filename: string, context: Context) {
    let audioStreamInfo: audio.AudioStreamInfo = {
      samplingRate: audio.AudioSamplingRate.SAMPLE_RATE_16000, // 采样率
      channels: audio.AudioChannel.CHANNEL_1,                  // 通道
      sampleFormat: audio.AudioSampleFormat.SAMPLE_FORMAT_S16LE, // 采样格式
      encodingType: audio.AudioEncodingType.ENCODING_TYPE_RAW  // 编码格式
    };
    let audioCapturerInfo: audio.AudioCapturerInfo = {
      source: audio.SourceType.SOURCE_TYPE_MIC,
      capturerFlags: 0
    };
    this.cancelled = false;                         // FIX
    audio.createAudioCapturer({ streamInfo: audioStreamInfo, capturerInfo: audioCapturerInfo },
      (err, data) => {
        if (err) {
          logger.error(`Invoke createAudioCapturer failed, code is ${err.code}, message is ${err.message}`);
          return;
        }
        this.audioCapturer = data;
        if (this.cancelled) {                       // FIX: the press was already released
          this.stopAndRelease();
          return;
        }
        let bufferSize: number = 0;
        let filePath = context.cacheDir + `/${filename}.pcm`;
        let file: fs.File = fs.openSync(filePath, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
        this.url = 'fd://' + file.fd;
        this.audioCapturer.on('readData', (buffer: ArrayBuffer) => {
          fs.writeSync(file.fd, buffer, { offset: bufferSize, length: buffer.byteLength });
          bufferSize += buffer.byteLength;
        });
        this.audioCapturer.on('stateChange', (state: audio.AudioState) => {
          if (state === 4) {                        // STATE_RELEASED
            fs.close(file);
          }
        });
        this.startRecording();
      });
  }

  stopAndRelease() {
    this.time2 = new Date().getTime();
    this.cancelled = true;                          // FIX: covers "stop arrived before create finished"
    if (this.audioCapturer) {
      this.audioCapturer.stop().then(() => {
        this.audioCapturer?.release();
      }).catch((err: BusinessError) => {
        logger.error('录音停止失败' + err.code + err.message);
      });
    }
  }
}
```

**The four stream fields are not stylistic.** Core Speech Kit's short-mode
recognizer takes 16 kHz mono 16-bit little-endian raw PCM and nothing else, and
the same numbers appear again in `StartParams.audioInfo` further down. Record
at 44.1 kHz and the transcript is silence; record stereo and it is noise.
Because the file is raw PCM with no header, every one of those parameters has
to be re-stated at recognition time - there is nothing in the file to infer
them from.

**The cancelled flag is the fix for a real user gesture, not a theoretical
race.** `createAudioCapturer` is callback-asynchronous. A quick tap on the
"hold to talk" button runs `onAction` → `createrOn` and then `onActionEnd` →
`stopAndRelease` well before the create callback fires. The shipped
`stopAndRelease` sees `this.audioCapturer === undefined`, does nothing, and
returns; moments later the callback assigns the capturer and calls
`startRecording()`. The microphone is now recording with no one holding a
reference to stop it, the `.pcm` grows without limit, and the fd is never
closed - the `stateChange` handler that closes it only fires on release.

Note also that `time2` is stamped in `stopAndRelease` and `time1` in
`startRecording`, so this same race produces `time2 < time1` and a negative
duration on the bubble.

**Playback that stops the wrong renderer - `entry/src/main/ets/utils/AudioRendererU.ets`** (corrected, see `HW-14-0054`)

```typescript
createPlayOn(filename: string, context: Context) {
  this.stopAndRelease();                      // FIX: stop whatever is playing before replacing it
  let audioStreamInfo: audio.AudioStreamInfo = {
    samplingRate: audio.AudioSamplingRate.SAMPLE_RATE_16000,
    channels: audio.AudioChannel.CHANNEL_1,
    sampleFormat: audio.AudioSampleFormat.SAMPLE_FORMAT_S16LE,
    encodingType: audio.AudioEncodingType.ENCODING_TYPE_RAW
  };
  let audioRendererInfo: audio.AudioRendererInfo = {
    usage: audio.StreamUsage.STREAM_USAGE_VOICE_MESSAGE,
    rendererFlags: 0
  };
  audio.createAudioRenderer({ streamInfo: audioStreamInfo, rendererInfo: audioRendererInfo },
    (err, data) => {
      if (err) {
        logger.error(`Invoke createAudioRenderer failed, code is ${err.code}, message is ${err.message}`);
        return;
      }
      const renderer = data;                  // FIX: capture this playback's own handle
      this.audioRenderer = renderer;
      let filePath = context.cacheDir + `/${filename}.pcm`;
      let file: fs.File = fs.openSync(filePath, fs.OpenMode.READ_ONLY);
      let fileSize: number = fs.statSync(filePath).size;
      let bufferSize: number = 0;

      renderer.on('writeData', (buffer: ArrayBuffer) => {
        if (bufferSize >= fileSize) {
          return;
        }
        let bytesRead = fs.readSync(file.fd, buffer, { offset: bufferSize, length: buffer.byteLength });
        bufferSize += bytesRead;
        if (bufferSize >= fileSize) {
          fs.close(file);
          renderer.stop().then(() => renderer.release());   // FIX: not this.stopAndRelease()
        }
      });
      renderer.start();
    });
}
```

**`STREAM_USAGE_VOICE_MESSAGE` is the correct usage and it changes behaviour.**
It routes the clip through the voice-message audio path, which is what makes a
messenger's voice note duck other audio and follow the earpiece/speaker policy
rather than playing as media.

**The shipped bug is entirely about `this`.** `createPlayOn` overwrites
`this.audioRenderer` without stopping the old one, and the per-file completion
inside `writeDataCallback` calls `this.stopAndRelease()`, which resolves
`this.audioRenderer` *at completion time*. Tap bubble A, then B: A's renderer is
orphaned and still playing, B's renderer is now the field, and when A's file
runs out its callback stops and releases **B** in the middle of playback.
Capturing `renderer` in a local closes the hole, and stopping the previous
renderer at the top of `createPlayOn` prevents the overlap in the first place.

**The recognition loop - `entry/src/main/ets/utils/SpeechRecognizerU.ets`** (as shipped)

```typescript
async writeAudio(sessionId: string, filename: string, context: Context) {
  this.message = '';
  this.startListeningForWriteAudio(sessionId);
  let filePath = context.cacheDir + `/${filename}.pcm`;
  let file = fileIo.openSync(filePath, fileIo.OpenMode.READ_WRITE);
  try {
    let buf: ArrayBuffer = new ArrayBuffer(1280);
    let offset: number = 0;
    while (1280 === fileIo.readSync(file.fd, buf, { offset: offset })) {
      let uint8Array: Uint8Array = new Uint8Array(buf);
      this.asrEngine?.writeAudio(sessionId, uint8Array);
      await this.countDownLatch(1);           // ~40 ms, one frame of wall-clock pacing
      offset = offset + 1280;
    }
  } catch (err) {
    logger.error(`Failed to read from file. Code: ${err.code}, message: ${err.message}.`);
  } finally {
    if (null != file) {
      this.asrEngine?.finish(sessionId);
      fileIo.closeSync(file);
    }
  }
}
```

**1280 bytes is one 40 ms frame at 16 kHz mono S16LE** - 16000 samples/s x 2
bytes x 0.04 s. That is why the loop sleeps 40 ms per frame: the engine expects
audio at roughly the rate it would arrive live, and feeding a whole file at
once is rejected or truncated. It also explains the strict `1280 === readSync`
condition, which both drives the loop and drops the final partial frame.

`finish(sessionId)` in the `finally` is what flushes the tail and produces the
final `onResult`. Without it the last words never arrive, because `onResult`
before `finish` reports intermediate hypotheses only. Every result - partial and
final - is written into the wrapper's `message` field, which the page watches.

**Routing the result to the right bubble - `entry/src/main/ets/pages/Speech.ets`** (corrected, see `HW-14-0056`)

```typescript
@State cIndex: number = 0;                       // "currently playing/selected bubble"
@State eIndex: number = -1;                      // the bubble that asked for a transcript
@State @Watch('result') speechRecognizerU: SpeechRecognizerU = new SpeechRecognizerU();

result() {
  if (this.eIndex < 0) {
    return;                                      // FIX: shipped code writes messageArr[this.cIndex]
  }
  this.messageArr[this.eIndex].context = this.speechRecognizerU.message;
}

// in voicePopup(index, filename), the 转文本 action:
.onClick(() => {
  this.eIndex = index;                           // captured at submission - now actually read
  this.messageArr[index].isShow = true;
  this.speechRecognizerU.writeAudio(new Date().getTime().toString(), filename, this.context);
  this.handlePopup_1 = false;
})
```

**Two indexes exist and the sample reads the wrong one.** `cIndex` means "the
bubble the user last touched" - it is reassigned by every playback tap and
every long press, and it is also what drives which bubble's `ImageAnimator`
runs. `eIndex` means "the bubble whose transcription is in flight"; the popup
handler sets it and nothing ever reads it. Since `result()` fires from an
asynchronous `onResult` callback that may be seconds behind the tap, converting
bubble A and then tapping bubble B to play it puts A's transcript under B.

The `@Watch('result')` on a `@State` object is itself worth noting: the wrapper's
`message` is set from the engine callback and the watcher copies it into the
`@Trace context` of one `OneVoice` - V1 `@Watch` at the page chained to V2
`@Trace` inside the model. It works, but that handoff is exactly where the
index must be pinned.

**Engine lifetime.** `EntryAbility` declares a module-level
`let speechRecognizerU1: SpeechRecognizerU = new SpeechRecognizerU();` and calls
`speechRecognizerU1.asrEngine?.shutdown()` from `onDestroy` - on an instance
whose `createByCallback` is never called, so `asrEngine` is always `undefined`
(`HW-14-0053`). Hold the engine where both the page and the ability can reach
it, and shut down that one.

**`shutdown()` on a never-created engine is a guaranteed no-op**, and this one
matters more than a usual leak: Core Speech Kit engines are exclusive.
`createEngine` returns `1002200006` ("engine busy") while another application -
or another instance in this process - holds one. Leaving the page's engine
alive past ability destruction can therefore block the *next* launch from
transcribing at all, with an error the user sees as nothing happening.

## Permissions & config

```json5
"requestPermissions": [{
  "name": 'ohos.permission.MICROPHONE',
  "reason": '$string:EntryAbility_desc',
  "usedScene": {
    "abilities": ["EntryAbility"],
    "when": 'always'
  }
}]
```

- `MICROPHONE` is `user_grant`, so `reason` and `usedScene` are mandatory and
  the reason resource must exist.
- `when: 'always'` is **wrong for this scenario**. The app records only while a
  button is held, in the foreground, with no continuous task or background
  audio capability declared. `when: "inuse"` is the honest declaration and the
  one a reviewer will expect.
- `reason` reuses `$string:EntryAbility_desc` - the value is literally
  `"description"` - rather than a sentence explaining why the microphone is
  needed. Users see this string in the permission dialog.
- The page requests the permission unconditionally from `aboutToAppear` and
  ignores the result; the actual gate is the `checkAccessTokenSync` on release,
  which toasts 请前往应用设置授予麦克风权限 (please grant the microphone
  permission in settings).
- No `INTERNET` permission is declared even though the engine is created with
  `online: 1`. Depending on the device's recognizer that is either irrelevant
  (on-device model) or a latent failure.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- The engine is created with `language: 'zh-CN'` and
  `extraParams: { locate: 'CN', recognizerMode: 'short' }`. Short mode is
  bounded - it is for utterances, not for long recordings.
- Recognition is **not** real time: the file is replayed at ~1x through a 40 ms
  sleep per frame, so transcribing a 30-second note takes about 30 seconds.
- Files accumulate in `cacheDir` under timestamp names and are never deleted.
  Bubble sides are decided by `index % 2` - there is no sender model.
- Only one engine, one capturer and one renderer exist at a time; see the
  findings for what happens when the user is faster than that, and
  `HW-14-0018` for the unstable `JSON.stringify(item)` list key.

## Pitfalls

- **`HW-14-0053`** (B/medium, confirmed): `EntryAbility.onDestroy` shuts down a
  module-level `new SpeechRecognizerU()` whose `asrEngine` is always
  `undefined`, while the page's real engine is never shut down. Cleanup is a
  guaranteed no-op and the leaked exclusive engine can make the next
  `createEngine` fail with "engine busy". Fix: route the shutdown to the
  instance the page created.
- **`HW-14-0054`** (B/medium, confirmed): `createPlayOn` overwrites
  `this.audioRenderer` without stopping the old one, and the completion
  callback calls `this.stopAndRelease()` on whatever renderer is current - so
  playing bubble A then B leaves both playing and lets A's end-of-file stop and
  release B. Fix: stop the previous renderer first and capture the instance in
  the callback.
- **`HW-14-0055`** (B/medium, probable): a quick press-release runs
  `stopAndRelease` before `createAudioCapturer`'s callback assigns the
  capturer, so the stop is a no-op and the late-started capture runs forever -
  microphone held, `.pcm` growing, fd never closed, and `time2 < time1` giving
  a negative duration. Fix: set a cancelled flag in `stopAndRelease` and honour
  it in the create callback.
- **`HW-14-0056`** (B/medium, confirmed): `result()` writes the transcript into
  `messageArr[this.cIndex]`, and `cIndex` is reassigned by any playback tap or
  long press; the correct `eIndex`, captured at submission, is dead code. Fix:
  read `eIndex`.
- **`HW-14-0018`** (B/medium, confirmed; systematic across six chat samples):
  the message `ForEach` keys on `JSON.stringify(item)`, which collides for two
  identical texts and changes identity when a `@Trace` field is written - so
  the second identical message may not render and a transcript can reuse the
  wrong node. Fix: append the index or give `OneVoice` an id.

## References

- `documentation/harmonyos-guides/08_ai/speechrecognizer-guide.md` - the create → setListener → startListening → writeAudio → finish flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/speechrecognizer-guide
- `documentation/harmonyos-references/07_ai/hms-ai-speechrecognizer.md` - `CreateEngineParams`, `StartParams.audioInfo`, `writeAudio`, `finish`, `shutdown`, error codes
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/hms-ai-speechrecognizer
- `documentation/harmonyos-guides/08_ai/core-speech-kit-guide.md` - the audio-format constraints and engine exclusivity
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/core-speech-kit-guide
- `documentation/harmonyos-guides/05_media/using-audiocapturer-for-recording.md` - the `readData` recording loop
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/using-audiocapturer-for-recording
- `documentation/harmonyos-references/04_media/arkts-apis-audio-audiocapturer.md` - `AudioCapturer`, `AudioState`, `stateChange`; `arkts-apis-audio-audiorenderer.md` - `writeData`, `StreamUsage`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio-audiocapturer
- `documentation/harmonyos-references/04_media/arkts-apis-audio.md` - `AudioStreamInfo`, sampling rates and sample formats
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-popup.md` - `bindPopup` and `onStateChange`; and `ts-basic-components-imageanimator.md` for the waveform `AnimationStatus`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-popup
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `ohos.permission.MICROPHONE`, `usedScene.when`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `SOCIAL-08` - where the systematic `ForEach` key finding `HW-14-0018` is recorded
