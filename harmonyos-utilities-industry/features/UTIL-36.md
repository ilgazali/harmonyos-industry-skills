---
id: UTIL-36
title: Inner audio recording in native code - AVScreenCapture with the video stream discarded, written as WAV
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/36_audio_inner_record.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio_inner_record-0000002523743763
sample: huawei_industry_tree/15_utilities/downloads/AudioInnerRecord.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.MediaKit", "@kit.PerformanceAnalysisKit"]
apis: [OH_AVScreenCapture_Create, OH_AVScreenCapture_Init, OH_AVScreenCapture_StartScreenCapture, OH_AVScreenCapture_StopScreenCapture, OH_AVScreenCapture_Release, OH_AVScreenCapture_SetMicrophoneEnabled, OH_AVScreenCapture_SetDataCallback, OH_AVScreenCapture_SetStateCallback, OH_AVBuffer_GetAddr, OH_AVBuffer_GetCapacity, napi_threadsafe_function, napi_call_threadsafe_function, "media.createAVPlayer", AVPlayer, backgroundTaskManager.startBackgroundRunning, wantAgent.getWantAgent, Canvas, CanvasRenderingContext2D, "@ComponentV2", "@Local", "@Monitor"]
permissions: [ohos.permission.KEEP_BACKGROUND_RUNNING]
min_api: 20
modules: [libavinnerrecord (har), entry]
findings: [HW-15-0006, HW-15-0076, HW-15-0077, HW-15-0101, HW-15-0102]
status: verified-with-fixes
---

## When to use

Load this card when the app must **capture the audio the device is playing**
rather than the microphone - recording a call-free voice memo of in-app
playback, capturing a music-app loop for editing, taking the audio track of a
screen session. On HarmonyOS there is no "record the system output" audio API:
the sanctioned route is `AVScreenCapture` in native code, configured to hand
back the *inner* audio buffers, with the video side left unused.

The transferable pieces are three. First, the **capture configuration**: pick
`OH_ORIGINAL_STREAM` so buffers come back raw through a data callback instead of
being muxed to a file, and disable the microphone so only playback audio is
captured. Second, the **NAPI callback bridge**: capture runs on its own native
thread, so state and duration reach ArkTS through
`napi_call_threadsafe_function`. Third, the **WAV framing**: PCM has no
container, so a 44-byte header is written first and rewritten with the final
sizes when recording ends.

The ArkTS side is deliberately thin - a `@ComponentV2` page whose `@Monitor`
turns a boolean into start/stop, a `Canvas` drawing amplitude bars, an
`AVPlayer` for playback. **Read all three findings before reusing the page or
the background-task module**: the teardown path is incomplete and
`BackgroundTask.ets` does not compile as shipped.

## Feature checklist

- A waveform canvas fills the top of the page and scrolls right to left while
  recording.
- A running duration in `HH:MM:SS` derived from the bytes written so far.
- One button starts and stops inner recording; a second plays the result back.
  Each is disabled while the other is active, and playback is disabled until a
  recording exists.
- The recording is a real `.wav` in the app sandbox, playable by an ordinary
  player.
- The recording folder is emptied when the page is created, so each session
  starts clean.
- Backgrounding the app requests a continuous task so capture is not suspended.

## Architecture

Two modules: an `entry` app and a `libavinnerrecord` HAR that wraps the native
recorder and exposes it as a singleton.

```
entry/src/main/ets
├── entryability/EntryAbility.ets    starts/stops the continuous task on background/foreground
├── model
│   ├── AVPlayer.ets                 AVPlayer wrapper: init, play(path), reset, state machine
│   ├── CanvasController.ets         the waveform: a rolling array of amplitudes drawn per tick
│   └── ListenerBase.ets             tiny key -> callbacks registry
├── pages/Home.ets                   @Entry @ComponentV2, the whole UI, 148 lines
└── utils/BackgroundTask.ets         continuous-task singleton  (broken, HW-15-0076)

libavinnerrecord
├── Index.ets                        re-exports AVInnerRecord
├── src/main/ets/AVInnerRecord.ets   ArkTS facade over the .so: init/start/stop/getAmplitudeScale/register*
└── src/main/cpp
    ├── IAVInnerRecord.h                        error codes, WavHeader/FmtChunk/DataChunk structs
    ├── innerRecord/AVInnerRecorder.cpp         AVScreenCapture lifecycle, PCM writing, WAV header
    ├── innerRecord/AVInnerRecordCallback.cpp   static C callbacks -> the recorder instance
    └── native/{InitNapi,InnerRecorder,InnerRecordNative}.cpp   NAPI surface + threadsafe callbacks
```

The document's tree is close but not exact: it lists `InnerRecorder.cpp` twice
where the second entry should be `InnerRecorder.h`, and it omits
`entryability/`, `entrybackupability/` and `libavinnerrecord/Index.ets`. No
finding is filed for this.

**The design decision worth copying** is the three-layer split inside the HAR.
`AVInnerRecorder` knows only `AVScreenCapture` and files; `InnerRecorder` knows
only NAPI and holds the threadsafe functions; `InnerRecordNative` is the
argument marshalling. The layer boundary is a `std::function`
(`SetStateCallback`, `SetRecordDurationCallback`), so the capture class has no
NAPI dependency at all and can be unit-tested or reused in a non-ArkTS host.
That separation is what makes the "capture on a native thread, deliver on the
JS thread" problem tractable: only the middle layer touches
`napi_call_threadsafe_function`.

On the ArkTS side, the equivalent decision is that **state drives the device,
not the other way round**. The button assigns `isInnerRecording`, and a
`@Monitor` on that field calls `start()` or `stop()`. The UI never calls the
recorder directly, so the enabled/disabled logic and the device calls can never
disagree.

## Implementation steps

1. **Declare `ohos.permission.KEEP_BACKGROUND_RUNNING`** and add
   `"backgroundModes": ["audioRecording"]` to the ability - both are required
   for the continuous task to be granted.
2. **Configure the capture** with `OH_CAPTURE_HOME_SCREEN`, `OH_ORIGINAL_STREAM`
   and an inner-audio source of `OH_ALL_PLAYBACK`. Original-stream mode is what
   routes buffers to your data callback instead of a muxer.
3. **Register error, state and data callbacks before `Init`**, then disable the
   microphone with `OH_AVScreenCapture_SetMicrophoneEnabled(capturer, false)`
   before `Start`, or the mic mixes into the recording.
4. **Filter buffers by type** in the data callback: only
   `OH_SCREEN_CAPTURE_BUFFERTYPE_AUDIO_INNER` is the inner audio; mic and video
   buffers arrive on the same callback.
5. **Write a placeholder WAV header first, and rewrite it on stop** with the
   final `riffSize`/`dataSize` - a PCM file with a zeroed size field is
   unplayable in most players.
6. **Recreate the capturer for every recording.** `Start` calls `Stop` first if
   one exists; `AVScreenCapture` instances are not restartable.
7. **Bridge to ArkTS with a threadsafe function**, and release it in the native
   destructor - `napi_call_threadsafe_function` from the capture thread is the
   only safe way to reach JS.
8. **Fix the continuous-task module before using it** (`HW-15-0076`): import
   `BusinessError`, use the exact ability name `EntryAbility`, and `return`
   after the invalid-context log.
9. **Tear everything down in `aboutToDisappear`**: stop the capture if it is
   still running, clear the 40 ms waveform interval, and release the
   `AVPlayer` (`HW-15-0077`, `HW-15-0006`).

## Verified snippets

All snippets are from `AudioInnerRecord.zip`. Corrected forms are marked.

**The capture configuration — `libavinnerrecord/src/main/cpp/innerRecord/AVInnerRecorder.cpp`** (as shipped)

```cpp
constexpr uint32_t SAMPLE_RATE = 44100;
constexpr uint16_t CHANNELS = 2;

AVInnerRecorder::AVInnerRecorder(const std::string &audioInnerFileName)
    : mIsInnerRecording(false), mCapturer(nullptr), mUserData(nullptr)
    , mCaptureConfig({
        OH_CAPTURE_HOME_SCREEN, OH_ORIGINAL_STREAM,
        {
            {SAMPLE_RATE, CHANNELS, OH_AudioCaptureSourceType::OH_MIC},          // micInfo (disabled below)
            {SAMPLE_RATE, CHANNELS, OH_AudioCaptureSourceType::OH_ALL_PLAYBACK}, // innerInfo: the playback mix
            {SAMPLE_RATE, OH_AudioCodecFormat::OH_AAC_LC}                        // encoder info, unused here
        },
        {{0}, {OH_VideoCodecFormat::OH_H264, 0, 0}}                              // video side left at zero
      })
    , mAudioInnerFileName(audioInnerFileName)
{
}

int32_t AVInnerRecorder::Start()
{
    if (mCapturer) {
        Stop();                                   // AVScreenCapture instances are single-use
    }
    int32_t ret = AV_INNER_RECORD_ERR_START_FAIL;
    do {
        ret = CreateInnerRecorder();              // Create + callbacks + Init + WAV header
        if (ret != AV_SCREEN_CAPTURE_ERR_OK) { break; }

        ret = OpenInnerRecordFile("ab");
        if (ret != AV_SCREEN_CAPTURE_ERR_OK) { break; }

        OH_AVScreenCapture_SetMicrophoneEnabled(mCapturer, false);   // inner audio only

        ret = OH_AVScreenCapture_StartScreenCapture(mCapturer);
        if (ret != AV_SCREEN_CAPTURE_ERR_OK) {
            OH_LOG_ERROR(LOG_APP, "Start inner record failed.");
            break;
        }
        mIsInnerRecording = true;
        return ret;
    } while (0);

    Stop();                                       // any failure unwinds through one path
    return ret;
}
```

**Three fields make this an audio recorder instead of a screen recorder.**
`OH_ORIGINAL_STREAM` selects raw buffers through the data callback (the
alternative, capture-to-file, would produce an MP4 and give you nothing to
process). The `innerInfo` audio source `OH_ALL_PLAYBACK` is the system playback
mix - that is the whole feature. And `SetMicrophoneEnabled(false)` after `Init`
suppresses the mic that the config still describes, so the file contains only
what the device was playing.

The video config is present but zeroed; `AVScreenCapture` requires the struct,
and the data callback simply never handles video buffers. The `do { } while (0)`
with a trailing `Stop()` is the C idiom for "one unwind path" and is worth
copying in NDK code where there is no RAII wrapper around the capturer.

**Receiving buffers and framing the WAV — same file** (as shipped)

```cpp
void AVInnerRecorder::OnBufferAvailable(OH_AVBuffer *buffer, const OH_AVScreenCaptureBufferType &bufferType)
{
    if (!mAudioInnerFile || !mIsInnerRecording) {
        return;
    }
    if (bufferType == OH_AVScreenCaptureBufferType::OH_SCREEN_CAPTURE_BUFFERTYPE_AUDIO_INNER) {
        int bufferLen = OH_AVBuffer_GetCapacity(buffer);
        uint8_t *buf = OH_AVBuffer_GetAddr(buffer);
        CalcAmplitude(buf, bufferLen);
        if (buf != nullptr) {
            fwrite(buf, 1, bufferLen, mAudioInnerFile);
        }
        NotifyRecordDuration();
    }
}

void AVInnerRecorder::UpdateWavFormatHeader()
{
    std::string openMode = "rb+";
    std::ifstream stream(mAudioInnerFileName.c_str());
    if (!stream.good()) {
        openMode = "wb";                          // first call: create the file
    }
    CloseInnerRecordFile();
    OpenInnerRecordFile(openMode);

    const uint16_t bitsPerSample = 16;
    WavHeader wavHeader = {
        .riff = {{'R', 'I', 'F', 'F'}, 0, {'W', 'A', 'V', 'E'}},
        .fmt = {{'f', 'm', 't', ' '}, 16, {1, CHANNELS, SAMPLE_RATE, 0, 0, bitsPerSample}},
        .data = {{'d', 'a', 't', 'a'}, 0}
    };
    wavHeader.fmt.wavFormat.byteRate = SAMPLE_RATE * CHANNELS * bitsPerSample / 8;
    wavHeader.fmt.wavFormat.blockAlign = CHANNELS * bitsPerSample / 8;
    if (mAudioInnerFile) {
        fseek(mAudioInnerFile, 0, SEEK_END);
        const int64_t size = ftell(mAudioInnerFile) - sizeof(WavHeader);
        wavHeader.riff.riffSize = size + sizeof(FmtChunk) + sizeof(DataChunk) + sizeof(wavHeader.riff.riffSize);
        wavHeader.data.dataSize = size;
        fseek(mAudioInnerFile, 0, SEEK_SET);       // patch the placeholder in place
        fwrite(&wavHeader, 1, sizeof(wavHeader), mAudioInnerFile);
    }
    CloseInnerRecordFile();
}
```


**The header is written twice, and that is the point.** `CreateInnerRecorder`
calls `UpdateWavFormatHeader` before capture starts, which creates the file and
lays down 44 bytes of placeholder; PCM then appends after it (the file is
reopened `"ab"`). `Release` calls the same function again, which now finds a
non-empty file, measures it, subtracts the header size and patches `riffSize`
and `dataSize` in place. One function, two roles, decided by whether the file
already exists.

Note the constraints baked into `IAVInnerRecord.h`: the structs are declared
inside `#pragma pack(1)`, without which the compiler's padding would corrupt the
header layout. `bitsPerSample` 16 and `formatType` 1 (PCM) must match what
`AVScreenCapture` actually delivers - the amplitude calculation reads the same
buffers as little-endian `int16` pairs, which only holds for this format.

**Page lifecycle — `entry/src/main/ets/pages/Home.ets`** (corrected, see `HW-15-0077`, `HW-15-0006`)

```typescript
@Entry
@ComponentV2
struct Index {
  @Local isInnerRecording: boolean = false;
  @Local isPlaying: boolean = false;
  @Local recordDuration: number = 0;

  private intervalId: number = -1;
  private avPlayer: AVPlayer = new AVPlayer();
  private canvas: CanvasController = new CanvasController();

  aboutToAppear(): void {
    const context = this.getUIContext().getHostContext();
    if (context) {
      this.defaultFileDir = context.filesDir + '/audio_files';
      this.emptyFolder();
      this.avPlayer.init();
      this.avPlayer.register('completed', this.onPlayFinishChange);
      this.audioInnerRecordFilePath = `${this.defaultFileDir}/${util.generateRandomUUID()}.wav`;
      audioInnerRecord.init(this.audioInnerRecordFilePath);
      audioInnerRecord.registerStateChangeCallback(this.onStateChange);
      audioInnerRecord.registerDurationCallback(this.onRecordDurationChange);
    }
  }

  aboutToDisappear(): void {
    this.avPlayer.unregister('completed', this.onPlayFinishChange);
    if (this.intervalId !== -1) {          // FIX: shipped teardown does none of the following
      clearInterval(this.intervalId);
      this.intervalId = -1;
    }
    if (this.isInnerRecording) {
      audioInnerRecord.stop();             // FIX: native capture otherwise keeps running
    }
    this.avPlayer.release();               // FIX: HW-15-0006 — never released in the sample
  }

  // 状态返回值参考OH_AVScreenCaptureStateCode，主动调用Stop不会触发上报状态变化
  private onStateChange = (state: number) => {
    if (state === 0) {                     // STARTED: begin sampling the waveform
      this.intervalId = setInterval(() => {
        const waveScale = audioInnerRecord.getAmplitudeScale();
        this.canvas.updateWaveData(waveScale);
      }, 40);
    } else if (state === 2) {              // CANCELED / stopped by the system
      this.isInnerRecording = false;
    }
  }

  @Monitor('isInnerRecording')
  onIsInnerRecordingChange() {
    if (this.isInnerRecording) {
      audioInnerRecord.start();
    } else {
      clearInterval(this.intervalId);
      this.intervalId = -1;
      audioInnerRecord.stop();
    }
  }
}
```

**`@Monitor` is the whole control flow.** The buttons only flip
`isInnerRecording` / `isPlaying`; the monitors translate a state change into a
device call. That is why the two buttons can be mutually exclusive with plain
`enabled(!this.isPlaying)` / `enabled(!this.isInnerRecording && this.recordDuration > 0)`
guards and no extra bookkeeping. The waveform interval is deliberately started
from the *native* state callback rather than from the button, so it only ticks
once capture is genuinely running.

As shipped, `aboutToDisappear` contains only the `unregister` line. Leaving the
page mid-recording therefore leaves the native `AVScreenCapture` running and the
40 ms interval firing against a dead component, and the WAV header is never
patched, so the file stays unplayable (`HW-15-0077`). Separately, the `AVPlayer`
created in `init()` is never released anywhere in the project - a project-wide
grep finds zero `.release()` calls (`HW-15-0006`); the wrapper needs the method
before the page can call it:

```typescript
// entry/src/main/ets/model/AVPlayer.ets — FIX: add the missing teardown
public async release() {
  if (!this.avplayer) {
    return;
  }
  try {
    this.closeAudioFile();
    await this.avplayer.release();
    this.avplayer = undefined;
  } catch (error) {
    logger.error(`AVPlayer release failed, err: ${error.code}, ${error.message}`);
  }
}
```

**The background-task module — `entry/src/main/ets/utils/BackgroundTask.ets`** (corrected, see `HW-15-0076`)

```typescript
import { Context, WantAgent, wantAgent } from '@kit.AbilityKit';
import { BusinessError } from '@kit.BasicServicesKit';        // FIX: missing — the file does not compile
import backgroundTaskManager from '@ohos.resourceschedule.backgroundTaskManager';

public startContinuousTask() {
  if (!this.context) {
    logger.info(`Context is invalid.`);
    return;                                                   // FIX: shipped guard falls through
  }

  const wantAgentInfo: wantAgent.WantAgentInfo = {
    wants: [
      {
        bundleName: 'com.audioinnerrecord.tool.demo',
        abilityName: 'EntryAbility',                          // FIX: shipped value is 'entryAbility'
      }
    ],
    actionType: wantAgent.OperationType.START_ABILITY,
    requestCode: 0,
    wantAgentFlags: [wantAgent.WantAgentFlags.UPDATE_PRESENT_FLAG]
  };

  wantAgent.getWantAgent(wantAgentInfo).then((wantAgentObj: WantAgent) => {
    backgroundTaskManager.startBackgroundRunning(this.context,
      backgroundTaskManager.BackgroundMode.AUDIO_RECORDING, wantAgentObj).catch((err: BusinessError<void>) => {
      logger.error(`Background task running failed, message: ${err.message}.`);
    });
  }).catch((err: BusinessError<void>) => {
    logger.error(`Failed to Operation getWantAgent, message: ${err.message}.`);
  });
}
```

**Three independent defects in fifty lines** (`HW-15-0076`). `BusinessError` is
used in two catch signatures and imported nowhere, so the module does not
compile. The `WantAgent` targets `abilityName: 'entryAbility'` while
`module.json5` declares `EntryAbility` - ability names are case-sensitive, so
tapping the continuous-task notification resolves nothing. And the
invalid-context guard logs and then continues into
`startBackgroundRunning(undefined, ...)`.

The surrounding wiring is right and worth keeping: `EntryAbility` calls
`startContinuousTask()` in `onBackground` and `stopContinuousTask()` in
`onForeground`, with `BackgroundMode.AUDIO_RECORDING` matching the
`"backgroundModes": ["audioRecording"]` declaration - the mode requested at
runtime must be one the module declares, or the request is rejected.

## Permissions & config

```json5
"abilities": [
  {
    "name": "EntryAbility",
    "backgroundModes": ["audioRecording"]
  }
],
"requestPermissions": [
  {
    "name": "ohos.permission.KEEP_BACKGROUND_RUNNING",
    "reason": "$string:background_running",
    "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
  }
]
```

- `KEEP_BACKGROUND_RUNNING` is `system_grant`; `reason` and `usedScene` are
  still declared, with `when: "always"` because the point is running while not
  in the foreground.
- No microphone permission is declared, and none is needed: the mic is
  explicitly disabled and only the playback mix is captured.
- `AVScreenCapture` itself raises the system's screen-capture consent prompt on
  `StartScreenCapture` - capturing playback audio is a privileged action even
  without video.
- The HAR builds `liblibavinnerrecord.so`, typed by
  `cpp/types/liblibavinnerrecord/Index.d.ts` and imported by the ArkTS facade as
  `avInnerRecord from 'liblibavinnerrecord.so'`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Recording is fixed at 44.1 kHz stereo 16-bit PCM. The WAV header, the
  duration arithmetic (`size / SAMPLE_RATE / (CHANNELS * 2) * 1000`) and the
  amplitude calculation all assume it.
- **A capturer cannot be restarted.** Every `Start` after a `Stop` must create a
  new `OH_AVScreenCapture` - the sample handles this by calling `Stop()` first.
- A self-initiated `Stop` does **not** fire a state-change callback, so ArkTS
  must not wait for one to update its UI. An incoming call
  (`OH_SCREEN_CAPTURE_STATE_STOPPED_BY_CALL`) releases on a detached thread and
  the recording cannot be resumed.
- The page empties `filesDir/audio_files` in `aboutToAppear`, so a previous
  recording is destroyed on every entry; there is no recording list and no
  export.
- The waveform keeps every amplitude sample in `waveDataList` for the lifetime
  of the page - 25 entries per second, never trimmed - though only the last
  screenful is drawn.

## Pitfalls

- **`HW-15-0006`** (B/low, confirmed): the `AVPlayer` is created in
  `AVPlayer.init()` and `release()` is never called anywhere in the sample - a
  project-wide grep finds zero occurrences. Native player resources are never
  freed, and the demo teaches an incomplete `AVPlayer` lifecycle. Fix: add a
  `release()` to the wrapper and call it from the page's `aboutToDisappear`.
- **`HW-15-0076`** (B/medium, confirmed): `BackgroundTask.ets` is broken three
  ways - `BusinessError` used in two catch signatures without an import (the
  file does not compile), the notification `WantAgent` targets `'entryAbility'`
  where the declared ability is `EntryAbility`, and the invalid-context guard
  logs without returning, so `startBackgroundRunning(undefined, ...)` runs.
  Fix: import `BusinessError`, use the exact ability name, and `return`.
- **`HW-15-0077`** (B/medium, probable): `aboutToDisappear` only unregisters the
  `AVPlayer` callback. Leaving the page while recording never calls
  `audioInnerRecord.stop()` and never clears the 40 ms waveform interval, so
  native capture keeps running against a dead UI and the WAV header is never
  patched with its final sizes. Fix: stop the capture and clear the interval on
  teardown.
- Not filed: the document's project tree lists `InnerRecorder.cpp` twice (the
  second should be `InnerRecorder.h`) and omits the ability directories and the
  HAR's `Index.ets`.
- Not filed, but a trap when adapting the code: `CalcAmplitude` reads the buffer
  as little-endian 16-bit samples with `(buffer[i] + (buffer[i + 1] << 8))` into
  an `int16_t`. Change the sample format and the waveform silently becomes
  noise.

## References

- `documentation/harmonyos-references/04_media/capi-avscreencapture.md` - `OH_AVScreenCapture_Create/Init/Start/Stop/Release`, `SetMicrophoneEnabled`, the data and state callbacks
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/capi-avscreencapture
- `documentation/harmonyos-guides/05_media/avscreencapture-c-basic-process.md` - the capture flow, `OH_ORIGINAL_STREAM` and the buffer types
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/avscreencapture-c-basic-process
- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - the `AVPlayer` state machine, `prepare`, `play`, `reset`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - `Canvas`, `CanvasRenderingContext2D`, `onReady`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-guides/03_application-framework/continuous-task.md` - continuous tasks, `backgroundModes` and the `WantAgent` requirement
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/continuous-task
- `documentation/harmonyos-references/02_application-framework/js-apis-resourceschedule-backgroundTaskManager.md` - `startBackgroundRunning`, `BackgroundMode.AUDIO_RECORDING`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resourceschedule-backgroundtaskmanager
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.KEEP_BACKGROUND_RUNNING`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `huawei_industry_tree/15_utilities/docs/36_audio_inner_record.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio_inner_record-0000002523743763
