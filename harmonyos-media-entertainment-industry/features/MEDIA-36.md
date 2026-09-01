---
id: MEDIA-36
title: Native PCM transcode - wrap raw PCM into m4a or amr with OH_AudioCodec plus OH_AVMuxer behind a NAPI async work
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/36_pcm_transcode.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/pcm_transcode-0000002513212618
sample: huawei_industry_tree/13_media_entertainment/downloads/PCMTranscode.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.AudioKit", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.MediaKit", "@kit.PerformanceAnalysisKit"]
apis: [abilityAccessCtrl, audio, common, fileIo, hilog, media, util, window]
permissions: [ohos.permission.MICROPHONE]
min_api: 20
modules: [entry, libavproc]
findings: [HW-13-0062, HW-13-0080, HW-13-0081, HW-13-0082, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when raw PCM has to become **a file other apps can open** - the
last step of any record-then-edit pipeline. PCM is what `AudioCapturer` gives
you and what `MEDIA-35`'s byte-offset editing needs, but nothing plays a headerless
`.pcm`: it has to be encoded and put in a container before it can be shared,
attached or uploaded.

The pattern is a HAR with a C++ core: **`OH_AudioCodec` encodes, `OH_AVMuxer`
containerises, and a NAPI async work keeps both off the UI thread.** The codec
is asynchronous by design - it calls you back when it wants an input buffer and
again when it has an output buffer - so the shape is two threads pumping two
queues, with a condition variable for "the stream is finished". Change the mime
type and the same code produces mp3, wav, flac or AMR-WB.

This is the one card in this set where the load-bearing defect is in C++.
**`HW-13-0080` is a lost-wakeup hang**: `StopEncode` waits on a condition
variable with no predicate, so a short recording finishes encoding before the
wait begins, the notify is dropped, and the transcode never returns. Read it
before you copy `AudioEncode.cpp` at all.

## Feature checklist

- Record PCM from the microphone into the app sandbox, with a 30-minute cap and
  a live elapsed-time readout.
- Play the recorded PCM back through an `AudioRenderer`.
- "Play m4a" transcodes the PCM to AAC-in-M4A on first use, then plays the
  result with `AVPlayer`.
- "Play amr" does the same with AMR-NB.
- While a transcode is in flight the button is disabled and reads 转码中
  (transcoding).
- A second tap on an already-transcoded format skips the encode and plays the
  existing file.
- Each panel shows the file name and, after playback starts, the duration
  reported by `AVPlayer`.
- Only one of the three players can be active at a time; starting one stops the
  others.

## Architecture

Two modules. `entry` is ArkTS only; `libavproc` is a HAR whose entire payload is
a native `.so` exposed through two NAPI functions.

```
entry/src/main/ets
├── constants/Constants.ets       MAX_RECORD_DURATION + AudioOptions (8 kHz mono S16LE)
├── entryability/EntryAbility.ets
├── model/
│   ├── AudioFileInfo.ets         path/fd/offset/size, and create/open/close
│   ├── AudioRecorder.ets         AudioCapturer -> .pcm, and getAVProcConfig for the encoder
│   ├── AudioPlayer.ets           AudioRenderer for the raw PCM
│   ├── AVPlayer.ets              media.AVPlayer for the transcoded m4a / amr
│   └── ListenerBase.ets          the key -> callback[] bus
├── pages/Home.ets                @Entry, a title + TranscodePage + a fake tab bar
├── pages/TranscodePage.ets       the three panels and all of the state machine
└── utils/                        Logger, PermissionManager, TimeUtil

libavproc
├── Index.ets / src/main/ets/AVEncoder.ets   singleton wrapper over liblibavproc.so
└── src/main/cpp
    ├── CMakeLists.txt            links libnative_media_acodec / _avmuxer / _core / _codecbase
    ├── include/IAVProc.h         FileInfo, AudioSampleInfo, AVProcResult, error codes
    ├── types/liblibavproc/Index.d.ts   the TS face of the two exported functions
    ├── native/
    │   ├── InitNapi.cpp          napi_module_register, exports encodePcmToM4a / encodePcmToAmrNb
    │   ├── AudioCodecNative.cpp  napi_create_async_work + the JS result callback
    │   ├── parse/AVProcParse.cpp napi_value -> FileInfo / AudioSampleInfo
    │   └── encode/AudioEncode.cpp the two pump threads and the stop handshake
    └── proc/
        ├── AVProcDef.h           AVProcContext: two {mutex, cond, indexQueue, bufferQueue} pairs
        ├── avcodec/AudioCodec.cpp     OH_AudioCodec lifecycle + buffer queues
        ├── avcodec/AVCodecCallback.cpp the four C callbacks, routed via userData
        ├── avmuxer/AVMuxer.cpp        OH_AVMuxer, track setup, mime -> container map
        └── sample/SampleConfig.cpp    the OH_AVFormat key/value bag and frame sizing
```

The documented tree matches the zip; the only difference is spelling
(`entrybackupablility` in the doc, `entrybackupability` on disk) and the doc's
`type/` for the real `types/`.

**The design decision worth copying** is `AVProcContext`. Encoding with
`OH_AudioCodec` is callback-driven from the codec's own threads, and the
temptation is to do the work inside those callbacks. The sample instead makes
the callbacks pure producers:

```cpp
typedef struct {
    std::mutex mutex;
    std::condition_variable cond;
    std::queue<uint32_t> indexQueue;
    std::queue<OH_AVBuffer *> bufferQueue;
} AVProcContextLock;

typedef struct { AVProcContextLock in; AVProcContextLock out; } AVProcContext;
```

`OnNeedInputBuffer` pushes onto `in`; `OnNewOutputBuffer` pushes onto `out`.
Two owned threads consume them: the input thread reads the file and pushes
frames, the output thread writes samples to the muxer. File I/O and muxing
therefore never run on a codec thread, and the two directions cannot deadlock
against each other because they hold different mutexes. This is the reference
shape for every `OH_AVCodec` integration.

**The design decision worth avoiding** is the stop handshake built on a bare
`condition_variable::wait`. See `HW-13-0080`; the fix is three lines and the bug
is total.

Worth noting on the ArkTS side: `SampleConfig` holds a
`std::map<std::string, std::variant<...>>` and applies it to an `OH_AVFormat`
through a `std::visit` visitor, so the *same* configuration object configures
both the codec and the muxer track. One source of truth for sample rate,
bitrate, channel count, layout, sample format and mime.

## Implementation steps

1. **Record PCM with an explicit stream config** and hand the *same* numbers to
   the encoder - `AudioRecorder.getAVProcConfig` builds the `IAVProcConfig`
   straight from `audioSteamInfo`, so the encoder can never disagree with the
   recorder.
2. **Expose the encoder as `napi_create_async_work`,** not a synchronous call:
   the whole encode blocks until EOS, and it must not block the JS thread. Take
   a `napi_ref` on the JS callback in the execute phase and call it from the
   complete phase.
3. **Create the muxer before the codec.** `OH_AVMuxer_Create(fd, format)` needs
   a writable fd and an `OH_AVOutputFormat` chosen from the mime type; add the
   audio track and only then start the codec.
4. **Register all four codec callbacks,** passing a `userData` struct carrying
   `this` so the static C callbacks can find the object. Do not leave
   `OnCodecError` empty (`HW-13-0080`).
5. **Pump input from a dedicated thread,** sizing each frame from
   `SampleConfig::AudioSampleFrameBytes()`. **Set `attr.size` from
   `gcount()`,** and validate `open()` before the loop (`HW-13-0082`).
6. **Pump output from a second thread,** writing each buffer to the muxer and
   freeing it, and break on `AVCODEC_BUFFER_FLAGS_EOS`.
7. **Signal completion through a flag, not a bare notify,** and wait on a
   predicate (`HW-13-0080`).
8. **Stop codec, stop muxer, release** - in that order - and return the error
   code to the async-work complete callback.
9. **On the ArkTS side, unregister the event you registered** and guard the
   registration (`HW-13-0081`).

## Verified snippets

All snippets are from `PCMTranscode.zip`. Corrected forms are marked.

**The stop handshake - `libavproc/src/main/cpp/native/encode/AudioEncode.cpp`** (corrected, see `HW-13-0080`)

```cpp
// AudioEncode.h
private:
    std::mutex mEncodeMutex;
    std::condition_variable mEncodeCond;
    bool mEncodeDone = false;                    // FIX: no such flag in the sample

int32_t AudioEncode::StopEncode()
{
    std::unique_lock<std::mutex> lock(mEncodeMutex);
    mEncodeCond.wait(lock, [this]() { return mEncodeDone; });   // FIX: sample is wait(lock)

    if (mAudioCodec != nullptr) {
        mAudioCodec->StopCodec();
    }
    if (mAudioMuxer != nullptr) {
        mAudioMuxer->StopMuxer();
    }

    Release();
    return mErrorCode;
}

void AudioEncode::AudioEncodeOutputThread()
{
    // ... drain mAudioContext->out, WriteSampleData, FreeOutputBuffer,
    //     break on attr.flags == AVCODEC_BUFFER_FLAGS_EOS ...

    {
        std::lock_guard<std::mutex> lock(mEncodeMutex);          // FIX: notify under the lock
        mErrorCode = AVPROC_ERR_OK;
        mEncodeDone = true;
    }
    mEncodeCond.notify_all();
}

// FIX: the sample's OnCodecError is an empty no-op, so a codec failure never wakes StopEncode
void AudioEncode::OnCodecError(int32_t errorCode)
{
    {
        std::lock_guard<std::mutex> lock(mEncodeMutex);
        mErrorCode = AVPROC_ERR_CODEC_FAIL;
        mEncodeDone = true;
    }
    mEncodeCond.notify_all();
}
```

**A condition variable is not a semaphore, and this is the canonical way to
learn that.** `StartEncode` detaches both pump threads and returns immediately;
`ExecuteEncodeCB` then calls `StopEncode`, which takes the mutex and waits. For
a recording of a few hundred milliseconds - which is exactly what a tester
produces first - the output thread can reach EOS and call `notify_all()` in the
gap between those two calls. A notify with no waiter is simply lost, and the
subsequent `wait(lock)` blocks forever. The napi worker thread hangs, the JS
result callback never fires, and `TranscodePage` sits on a disabled 转码中
button for the life of the process.

The predicate form fixes both halves: `wait(lock, pred)` checks `mEncodeDone`
*before* sleeping, so an early completion is seen rather than missed, and it
re-checks on every wakeup, so a spurious wakeup does not fall through. Setting
the flag under the same mutex is what makes that check meaningful. The empty
`AVCodecCallback::OnCodecError` is the second half of the same hang: a codec
that fails never reaches EOS, so without an error path that also sets the flag
there is no wakeup at all.

**The input pump - same file** (corrected, see `HW-13-0082`)

```cpp
void AudioEncode::AudioEncodeInputThread()
{
    if (!mAudioContext || !mAudioCodec || !mSampleConfig) {
        return;
    }

    std::ifstream inputFile;
    inputFile.open(mFileInfo.inputFilePath, std::ios::in | std::ios::binary);
    if (!inputFile.is_open()) {                       // FIX: sample never checks
        OH_LOG_ERROR(LOG_APP, "Open input file failed.");
        OnCodecError(AVPROC_ERR_CODEC_FAIL);          // FIX: otherwise StopEncode waits forever
        return;
    }

    const int32_t inputFrameBytes = mSampleConfig->AudioSampleFrameBytes();
    while (true) {
        std::unique_lock<std::mutex> lock(mAudioContext->in.mutex);
        bool hasData = mAudioContext->in.cond.wait_for(lock, std::chrono::seconds(1),
                           [this]() { return !mAudioContext->in.bufferQueue.empty(); });
        if (!hasData) {
            continue;
        }

        uint32_t index = mAudioContext->in.indexQueue.front();
        auto buffer = mAudioContext->in.bufferQueue.front();
        OH_AVCodecBufferAttr attr = {0, 0, 0, AVCODEC_BUFFER_FLAGS_NONE};

        inputFile.read((char *)OH_AVBuffer_GetAddr(buffer), inputFrameBytes);
        std::streamsize got = inputFile.gcount();     // FIX: sample assigns inputFrameBytes blindly
        if (got > 0) {
            attr.size = static_cast<int32_t>(got);    // the tail frame is shorter
            attr.flags = AVCODEC_BUFFER_FLAGS_NONE;
        } else {
            attr.size = 0;
            attr.flags = AVCODEC_BUFFER_FLAGS_EOS;
        }

        mAudioContext->in.indexQueue.pop();
        mAudioContext->in.bufferQueue.pop();
        mAudioCodec->PushInputBuffer(index, attr, buffer);
        if (attr.flags == AVCODEC_BUFFER_FLAGS_EOS) {
            break;
        }
    }

    inputFile.close();
}
```

**`gcount()` is the only thing that knows how much was actually read.** The
shipped loop tests `!inputFile.eof()` before the read and then declares
`attr.size = inputFrameBytes` regardless, so the last frame - which is almost
never a whole `inputFrameBytes` - is submitted at full size with uninitialised
tail bytes. Every m4a and amr the sample produces ends with a frame of garbage.
Worse, `eof()` is only set *after* a read that hits the end, and a failed
`open()` sets `failbit`, not `eofbit`: with a missing input file `eof()` stays
false forever and the thread pushes full-size garbage frames indefinitely,
never reaching EOS - which then hangs `StopEncode` for the second time.

Note the 1-second `wait_for` with a predicate on both pump threads. That is the
correct shape and it is a pointed contrast with `StopEncode`: the same file gets
the predicate right twice and wrong once.

`inputFrameBytes` comes from `SampleConfig::AudioSampleFrameBytes()`, which is
the one place the sample encodes the fact that **every codec has a fixed frame
size**: 1024 samples per channel for AAC-LC, 2048 for HE-AAC (SBR halves the
core rate), and `sampleRate * 0.02` for everything else - the 20 ms AMR frame.
Feeding the encoder a buffer that is not a whole number of frames is a silent
quality bug, not an error: it shows up as clicks at frame boundaries.

**Mime to container - `libavproc/src/main/cpp/proc/avmuxer/AVMuxer.cpp`** (as shipped)

```cpp
int32_t AVMuxer::CreateMuxer(const FileInfo &fileInfo)
{
    const std::string outputFile(fileInfo.outputFilePath);
    mOutputFile = fopen(outputFile.c_str(), "w+");
    if (!mOutputFile) {
        return AVPROC_ERR_INIT_FAIL;
    }
    int32_t fd = fileno(mOutputFile);
    if (fd == -1) {
        Release();
        return AVPROC_ERR_INIT_FAIL;
    }

    OH_AVOutputFormat format = AV_OUTPUT_FORMAT_DEFAULT;
    int32_t ret = GetOutputFormat(&format);
    if (ret != AVPROC_ERR_OK) {
        return ret;
    }
    mAVMuxer = OH_AVMuxer_Create(fd, format);
    return AVPROC_ERR_OK;
}

int32_t AVMuxer::GetOutputFormat(OH_AVOutputFormat *format)
{
    const std::map<std::string, OH_AVOutputFormat> aFormats = {
        {std::string(OH_AVCODEC_MIMETYPE_AUDIO_AAC),     AV_OUTPUT_FORMAT_M4A},
        {std::string(OH_AVCODEC_MIMETYPE_AUDIO_MPEG),    AV_OUTPUT_FORMAT_MP3},
        {std::string(OH_AVCODEC_MIMETYPE_AUDIO_AMR_NB),  AV_OUTPUT_FORMAT_AMR},
        {std::string(OH_AVCODEC_MIMETYPE_AUDIO_AMR_WB),  AV_OUTPUT_FORMAT_AMR},
        {std::string(OH_AVCODEC_MIMETYPE_AUDIO_G711MU),  AV_OUTPUT_FORMAT_WAV},
    };
    auto it = aFormats.find(mSampleConfig->GetSampleInfo().audio.codecMine);
    if (it != aFormats.end()) {
        *format = it->second;
        return AVPROC_ERR_OK;
    }
    return AVPROC_ERR_INIT_FAIL;
}
```

**The codec and the container are two independent choices, and this map is the
join.** `OH_AudioCodec_CreateByMime` picks the encoder; `OH_AVMuxer_Create`
picks the file format; nothing in the API forces them to agree. Adding mp3
output to this sample is one entry in the map plus one more NAPI export - the
threads, the queues and the config bag do not change. Note that AMR-NB and
AMR-WB share a container, and that `OH_AVMuxer_Create` takes an **fd**, so the
file must be opened for writing first; the `FILE*` is kept only so `Release`
can `fclose` it.

**Renderer listener hygiene - `entry/src/main/ets/model/AudioPlayer.ets`** (corrected, see `HW-13-0081`)

```typescript
private isAlreadyRegisterListen: boolean = false;      // FIX: absent here, present in AudioRecorder

public start() {
  if (!this.player) {
    return;
  }
  if (!this.isAlreadyRegisterListen) {                 // FIX: sample re-registers on every start
    try {
      this.player.on('writeData', this.writeDataCallback);
      this.isAlreadyRegisterListen = true;
    } catch (error) {
      logger.error(`register listen fail, code: ${error.code}, message: ${error.message}`);
    }
  }
  this.audioFileInfo.openAudioFile();
  this.player.start();
}

public stop() {
  if (!this.player) {
    return;
  }
  this.player.stop(() => {
    this.player?.off('writeData', this.writeDataCallback);   // FIX: sample calls off('periodReach')
    this.isAlreadyRegisterListen = false;
    this.audioFileInfo.closeAudioFile();
  });
}
```

**The sibling class in the same sample gets this right.** `AudioRecorder`
guards its `on('readData')` with `isAlreadyRegisterListen` and unregisters the
matching event in `release()`; `AudioPlayer` guards nothing and unregisters
`'periodReach'`, an event it never subscribed to, so the `off` is a silent
no-op. Each play/stop cycle therefore stacks another `writeData` callback, and
because every callback advances `audioFileInfo.offset` by its own `readLen`, the
second playback consumes the file at twice the rate and the third at three
times. Passing the callback reference to `off` as well as the event name is what
makes the unregistration exact.

## Permissions & config

```json5
// entry/src/main/module.json5
"requestPermissions": [
  {
    "name": "ohos.permission.MICROPHONE",
    "reason": "$string:microphone_reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
]

// libavproc/src/main/module.json5
{ "module": { "name": "libavproc", "type": "har",
              "deviceTypes": ["default", "tablet", "2in1"] } }
```

- `MICROPHONE` is `user_grant`; `reason` and `usedScene` are mandatory.
  `PermissionManager` handles the permanent refusal with
  `requestPermissionOnSetting`.
- The transcode itself needs no permission - everything happens inside the app
  sandbox.
- `libavproc` declares `deviceTypes` of `default`, `tablet`, `2in1` while
  `entry` declares only `phone`. The HAR is the wider set, so the intersection
  is `phone` and the build is consistent, but the two lists should be kept in
  step if the app grows.
- `CMakeLists.txt` links `libnative_media_acodec.so`, `libnative_media_avmuxer.so`,
  `libnative_media_core.so`, `libnative_media_codecbase.so`, plus
  `libace_napi.z.so` and `libhilog_ndk.z.so`. The demuxer and avsource libraries
  are linked but unused.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Recording is 8 kHz mono S16LE (`SAMPLE_RATE_8000`), chosen so AMR-NB - which
  only accepts 8 kHz - works without resampling. There is no resampler
  anywhere in the sample, so a 44.1 kHz source cannot be transcoded to AMR-NB
  by this code.
- Bitrates are hardcoded per format: 32000 for m4a, 10200 for AMR-NB.
- `AVProcParse::StringConvert` copies into a fixed `char str[1024]`, so a
  sandbox path longer than 1023 bytes is silently truncated; the class is also
  a singleton holding one `napi_env` set at module init, so the encoder is not
  safe to call from more than one JS thread.
- `AudioRecorder.init` deletes the whole `audio_files` directory on every
  launch, so nothing survives a restart.
- Only encoding is implemented. The decode direction (`isEncoder` on
  `AVProcSampleInfo`) is plumbed through `SampleConfig` and `AudioCodec` but is
  never exercised. The encode runs in one synchronous pass with no progress
  reporting.

## Pitfalls

- **`HW-13-0080`** (D/high, confirmed): `StopEncode` calls
  `mEncodeCond.wait(lock)` with no predicate. The detached output thread
  notifies at EOS, which for a millisecond-length PCM file can happen before
  `StopEncode` starts waiting - the notify is lost and the wait never returns.
  `AVCodecCallback::OnCodecError` is an empty no-op, so an error never notifies
  either. The napi worker hangs, the JS callback never fires, and
  `TranscodePage` stays on a disabled 转码中 button forever. Fix: add a
  `mEncodeDone` flag set under the mutex at EOS *and* on error, and wait with
  `wait(lock, [&]{ return mEncodeDone; })`.
- **`HW-13-0082`** (B/medium, confirmed): the encoder input loop sets
  `attr.size = inputFrameBytes` unconditionally, ignoring `gcount()`, so the
  final partial frame of every transcode carries garbage; and `open()` is never
  checked, so a missing input file sets `failbit` rather than `eofbit`,
  `eof()` stays false, and the thread pushes full-size garbage frames forever
  without ever emitting EOS. Fix: size the frame from `gcount()` and validate
  `open()` before the loop.
- **`HW-13-0081`** (B/medium, confirmed): `AudioPlayer.stop()` unregisters
  `'periodReach'` although only `'writeData'` was ever registered - a
  copy-paste no-op - and `start()` re-registers on every call with no guard
  (unlike `AudioRecorder`, which guards with `isAlreadyRegisterListen`).
  Play/stop cycles therefore stack `writeData` callbacks that each advance the
  read offset, corrupting every playback after the first. Fix: `off('writeData',
  callback)` in `stop`, and guard the registration.
- **`HW-13-0062`** (B/low, confirmed): the systematic stale-tail defect.
  `writeDataCallback` clamps `readLen` to the remaining file bytes but leaves
  the rest of the renderer's buffer holding the previous frame, so the last
  frame of every PCM playback renders garbage or an echo. Four samples share
  this. Fix: zero the buffer beyond `readLen`, or use a partial-write API.

## References

- `documentation/harmonyos-guides/05_media/avcodec-kit-intro.md` - what AVCodec Kit covers and the encode pipeline
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/avcodec-kit-intro
- `documentation/harmonyos-references/04_media/capi-audiocodec.md` - `OH_AudioCodec_CreateByMime`, `Configure`, `Prepare`, `Start`, `PushInputBuffer`, `FreeOutputBuffer`, the callback struct
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/capi-audiocodec
- `documentation/harmonyos-guides/02_media/audio-video-muxer.md` - `OH_AVMuxer_Create`, `AddTrack`, `Start`, `WriteSampleBuffer`, and the fd requirement
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/audio-video-muxer
- `documentation/harmonyos-references/04_media/capi-avmuxer.md` - `OH_AVOutputFormat` and the supported containers
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/capi-avmuxer
- `documentation/harmonyos-guides/10_ndk-development/use-napi-asynchronous-task.md` - `napi_create_async_work`, `napi_ref` on the JS callback, the execute/complete split
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/use-napi-asynchronous-task
- `documentation/harmonyos-references/04_media/arkts-apis-audio-audiocapturer.md` - `on('readData')` and the capturer lifecycle
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio-audiocapturer
- `documentation/harmonyos-references/04_media/arkts-apis-audio-audiorenderer.md` - `on('writeData')` / `off`, and the partial-write contract
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio-audiorenderer
- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - playing the transcoded file through `fd://`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `MICROPHONE` as a `user_grant` permission
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `MEDIA-35` - the PCM editing step that produces the input to this encoder
- `MEDIA-27` - the systematic stale-tail render defect
