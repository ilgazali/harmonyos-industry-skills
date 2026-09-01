---
id: MEDIA-41
title: Automatic video subtitles - extract PCM with ffmpeg, run speech recognition, replay the text against the playback clock
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/41_video_subtitle.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_subtitle-0000002568904821
sample: huawei_industry_tree/13_media_entertainment/downloads/VideoSubtitle.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.CoreSpeechKit", "@kit.MediaKit", "@kit.PerformanceAnalysisKit"]
apis: ["speechRecognizer.createEngine", "SpeechRecognitionEngine.setListener", "SpeechRecognitionEngine.startListening", "SpeechRecognitionEngine.writeAudio", "SpeechRecognitionEngine.isBusy", "SpeechRecognitionEngine.finish", "SpeechRecognitionEngine.shutdown", "MP4Parser.ffmpegCmd", "media.createAVPlayer", "AVPlayer.fdSrc", "AVPlayer.on('timeUpdate')", "AVPlayer.on('durationUpdate')", "AVPlayer.on('stateChange')", "resourceManager.getRawFd", "resourceManager.getRawFileContentSync", "fileIo.openSync", "fileIo.readSync", "emitter.emit", "emitter.on", "XComponent", "XComponentController.getXComponentSurfaceId", "display.getDefaultDisplaySync", "window.setPreferredOrientation", "util.generateRandomUUID"]
permissions: []
min_api: 17
modules: [entry]
findings: [HW-13-0090, HW-13-0091, HW-13-0092, HW-13-0005, HW-13-0097, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when you need **captions for a video that has none** and there
is no subtitle track to load - user-generated clips, lecture recordings,
in-app short video, voice notes played back with a transcript. The pattern is:
pull a mono 16 kHz PCM track out of the container with ffmpeg, feed it to the
Core Speech Kit recogniser in small blocks, turn "how much audio have I fed
in" into a timestamp, and then look the text up again from the player's
`timeUpdate` callback.

The transferable idea is **the clock**. There is no timing information
anywhere in the recogniser's output - `SpeechRecognitionResult` carries text
and an `isFinal` flag, nothing else. The sample recovers time arithmetically
from the byte offset of the PCM stream, because raw PCM at a known sample
rate, sample width and channel count is a linear function from bytes to
milliseconds. That works for any "align recognised text with media" problem:
lyric alignment, transcript scrubbing, keyword search inside audio.

**Read `HW-13-0090` before adopting it.** The shipped sample starts
recognition without waiting for the extraction to finish, and the reader
treats a short read as end-of-audio. On a real device the subtitles come out
truncated or empty most launches. The fix is small but it is not optional.

## Feature checklist

- A full-screen landscape player opens on a bundled `input.mp4` and starts
  playing as soon as the surface is ready.
- The audio track is extracted to a sandbox PCM file (16 kHz, mono, s16le).
- Speech recognition runs over that PCM file and yields one subtitle record per
  completed sentence, each with a start and end time in milliseconds.
- While the video plays, the caption matching the current playback position is
  drawn near the bottom of the frame; when no caption covers the position,
  nothing is drawn.
- The last caption of the video displays like every other one (this is what
  `HW-13-0092` breaks).
- A tap toggles the control overlay; an eye button toggles the caption layer
  independently of the overlay.
- The slider seeks; the elapsed/total readout is formatted `mm:ss` or
  `hh:mm:ss`.
- Nothing keeps spinning after the recogniser errors out (this is what
  `HW-13-0091` breaks).

## Architecture

One `entry` module, three layers: a page, two view-model classes, and a
`common` folder holding constants, the record type and the ffmpeg wrapper.

```
entry/src/main/ets
├── common
│  ├── Constants.ets              BUF_SIZE 1280, INTERVAL_MS 5, 16 kHz/16 bit/mono, file names
│  ├── Objects.ets                interface Subtitle { startTime, endTime, content }
│  └── utils.ets                  time2str + extractAudio (rawfile -> sandbox -> ffmpeg)
├── entryability/EntryAbility.ets forces light colour mode, setWindowLayoutFullScreen
├── entrybackupability/
├── pages/Index.ets               @Entry: XComponent, caption layer, control overlay
└── viewmodel
   ├── SubtitleRecognizer.ets     ASR engine, the 5 ms feed loop, offset -> time
   └── VideoPlayer.ets            AVPlayer, subtitle store, timeUpdate lookup
```

The documented tree matches the zip file for file. The document's constraints
section claims API Version 20, but `build-profile.json5` pins
`compatibleSdkVersion: "5.0.5(17)"` - treat 17 as the real floor. The one
third-party dependency, `@ohos/mp4parser` `^2.0.6`, is declared in the
project-level `oh-package.json5`, not in `entry/oh-package.json5`.

**The design decision worth copying** is that the subtitle store lives in
`VideoPlayer`, not in the page, and is filled through an `emitter` event
rather than a direct call. `SubtitleRecognizer` knows nothing about the
player: it emits `'Subtitle'` with a `Subtitle` payload, and `VideoPlayer`
subscribes in `init()` and pushes into a plain array. That keeps the
recogniser reusable for audio-only transcription and means recognition results
can arrive long after playback has started - which they will, since ASR of a
ten-minute video takes far longer than opening the file. The page never sees
either object; it reads three `AppStorage` keys (`Subtitle`, `CurrentTime`,
`Duration`, plus `IsPlaying`) through `@StorageProp`.

**The decision worth avoiding** is in the same place: `extractAudio` is
started as a fire-and-forget async call and `startRecognizer` runs on the very
next line. The producer and the consumer share a sandbox file with no
handshake. See `HW-13-0090`.

## Implementation steps

1. **Copy the video into the sandbox first.** `ffmpegCmd` takes file paths, not
   raw file descriptors, so the rawfile has to be written to `context.filesDir`
   with `getRawFileContentSync` + `fs.writeSync` before ffmpeg can see it.
   Unlink both the video and the PCM output first so a second launch does not
   append to a stale file.
2. **Match the ffmpeg output format to the recogniser's `audioInfo` exactly**:
   `-ar 16000 -ac 1 -f s16le -acodec pcm_s16le` against `sampleRate: 16000`,
   `soundChannel: 1`, `sampleBit: 16`, `audioType: 'pcm'`. Any mismatch turns
   the timestamp arithmetic into nonsense as well as the audio.
3. **Wait for ffmpeg's `callBackResult` before calling `startListening`**
   (`HW-13-0090`). Wrap `MP4Parser.ffmpegCmd` in a promise resolved from the
   callback and `await` it.
4. **Create the engine with `recognizerMode: 'long'`** and a
   `maxAudioDuration` in `startExtraParams` - the default short mode ends the
   session after a single utterance.
5. **Feed the PCM in `BUF_SIZE` blocks on a timer, and skip the tick when
   `isBusy()` is true.** Do not feed from a tight loop: `writeAudio` is
   synchronous into a bounded queue and the engine drops what it cannot take.
6. **Zero the tail of the last partial block** before writing it, otherwise the
   final fragment carries bytes from the previous block.
7. **Distinguish end-of-file from catch-up.** Once step 3 is in place, `len === 0`
   really is EOF; without it, it is usually just the reader overtaking the
   writer (`HW-13-0090`).
8. **Clear the interval and close the fd on every exit path**, not only the EOF
   path - `onError` and `onComplete` included (`HW-13-0091`).
9. **Reset `writeOffset` after `finish()` has drained its callbacks**, or
   snapshot it, so the last sentence gets a real end time (`HW-13-0092`).
10. **Look the caption up from `timeUpdate`, guarded by the current record's
    range**, so the linear scan runs only when the playhead leaves the caption
    currently on screen.
11. **Release the AVPlayer when the page goes away** (`HW-13-0005`) - this
    sample creates one and never calls `release()`, and it also never closes
    the raw fd it opened with `getRawFd`.

## Verified snippets

All snippets are from `VideoSubtitle.zip`. Corrected forms are marked.

**Extracting the PCM track — `entry/src/main/ets/common/utils.ets`** (as shipped)

```typescript
export async function extractAudio(context: Context, rawFileName: string) {
  // Write video file to sandbox
  let fileDir = context.filesDir;
  let videoFilePath = fileDir + '/' + Constants.videoFileName;
  if (fs.accessSync(videoFilePath)) {
    fs.unlinkSync(videoFilePath);
  }
  let audioFilePath = fileDir + '/' + Constants.audioFileName;
  if (fs.accessSync(audioFilePath)) {
    fs.unlinkSync(audioFilePath);
  }
  let rawFileContent = context.resourceManager.getRawFileContentSync(rawFileName);

  let videoFD: number = -1;
  try {
    let videoFile = fs.openSync(videoFilePath, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
    videoFD = videoFile.fd;
    fs.writeSync(videoFile.fd, rawFileContent.buffer, { offset: 0, length: rawFileContent.length });
  } finally {
    if (videoFD !== -1) {
      fs.closeSync(videoFD);
    }
  }
  // Extract audio from video (sampling rate: 16000, channel count: 1, sampling format: s16le)
  let cmd = `ffmpeg -i ${videoFilePath} -vn -acodec pcm_s16le -ar 16000 -ac 1 -f s16le ${audioFilePath}`;
  MP4Parser.ffmpegCmd(cmd, callback);
}
```

**Four flags in that command line carry the whole design.** `-vn` drops the
video stream so ffmpeg does not spend time on frames nobody will look at.
`-ar 16000 -ac 1` produce exactly what the recogniser is configured for, and
also make the byte-to-millisecond conversion in `getTime` a one-liner.
`-f s16le` forces headerless raw PCM rather than a WAV container - important,
because the reader later opens the file with `fileIo.readSync` from offset 0
and would otherwise feed a 44-byte RIFF header to the recogniser as if it were
audio.

The `try/finally` around the write closes the descriptor on the throwing path.
Note the copy is unconditional: every launch rewrites the whole rawfile into
the sandbox, acceptable for a demo asset and not for a user-picked 4K clip.

**Starting extraction and recognition in the right order — `entry/src/main/ets/viewmodel/VideoPlayer.ets`** (corrected, see `HW-13-0090`)

```typescript
async init() {
  emitter.on('Subtitle', (data: emitter.EventData) => {
    let subtitle: Subtitle = data.data!.subtitle;
    this.subtitles.push(subtitle);
  });
  // Extract audio
  await extractAudio(this.context, 'input.mp4');   // FIX: the sample does not await this
  // Recognize subtitle
  await this.recognizer.initEngine();
  let fileDir = this.context.filesDir;
  this.recognizer.startRecognizer(fileDir);
  // Create avplayer
  this.videoPlayer = await media.createAVPlayer();
  this.setCallbacks();
}
```

For the `await` to mean anything, `extractAudio` has to resolve from ffmpeg's
own callback rather than from the call that queues it:

```typescript
// FIX: minimal corrected form of the callback block in utils.ets
return new Promise<void>((resolve, reject) => {
  let callback: ICallBack = {
    callBackResult: (code: number) => {
      if (code === 0) {
        hilog.info(DOMAIN, TAG, 'Extract audio succeed');
        resolve();
      } else {
        hilog.info(DOMAIN, TAG, 'Extract audio failed');
        reject(new Error(`ffmpegCmd failed, code: ${code}`));
      }
    }
  };
  MP4Parser.ffmpegCmd(cmd, callback);
});
```

**`MP4Parser.ffmpegCmd` is asynchronous and its return conveys nothing.**
The shipped `extractAudio` logs `'Extract audio succeed'` on the line after the
call, before ffmpeg has decoded a single frame, and the `ICallBack` that would
have told the truth only writes to hilog. Two failure modes follow. If
`startListening` reaches `onStart` before ffmpeg has created `output.pcm`,
`fileIo.openSync` throws inside the listener and there are no captions at all.
If the file exists but is still being written, the reader catches up with the
writer, `readSync` returns 0, and the shipped loop treats that as
end-of-audio: it closes the file, clears the timer and calls `finish()`. The
video then plays with captions for its first few seconds and silence after
that - and it is timing-dependent, so it looks intermittent.

**The feed loop — `entry/src/main/ets/viewmodel/SubtitleRecognizer.ets`** (corrected, see `HW-13-0090`, `HW-13-0091`, `HW-13-0092`)

```typescript
onStart: (sessionId: string, eventMessage: string): void => {
  let buffer = new Uint8Array(Constants.BUF_SIZE);
  let audioFile = fileIo.openSync(this.audioFilePath, fileIo.OpenMode.READ_ONLY);
  this.audioFile = audioFile;                      // FIX: keep it, so onError can close it
  this.intervalId = setInterval(() => {
    if (this.asrEngine?.isBusy()) {
      return;                                      // engine still chewing: skip this tick
    }
    let len =
      fileIo.readSync(audioFile.fd, buffer.buffer, { offset: this.writeOffset, length: Constants.BUF_SIZE });
    if (len === 0) {
      // FIX: only a true EOF now that extraction completed before startListening
      this.stopFeeding();                          // clearInterval + closeSync
      this.asrEngine?.finish(this.sessionId);      // FIX: writeOffset is NOT reset here
      return;
    } else if (len < Constants.BUF_SIZE) {
      for (let i = len; i < Constants.BUF_SIZE; i++) {
        buffer[i] = 0;                             // zero the tail of the last block
      }
    }
    this.writeOffset += len;
    this.asrEngine?.writeAudio(this.sessionId, buffer);
  }, Constants.INTERVAL_MS);
},
onComplete: (sessionId: string, eventMessage: string): void => {
  this.stopFeeding();                              // FIX: absent in the sample
  this.asrEngine?.shutdown();
},
onError: (sessionId: string, errorCode: number, errorMessage: string): void => {
  hilog.error(DOMAIN, TAG,
    `onError, sessionId: ${sessionId}, erroeCode: ${errorCode}, errorMessage: ${errorMessage}`);
  this.stopFeeding();                              // FIX: absent in the sample
  this.asrEngine?.shutdown();
}
```

**`isBusy()` is the back-pressure and the 5 ms tick is the throttle.** 1280
bytes of 16 kHz mono 16-bit PCM is exactly 40 ms of audio, so a 5 ms interval
feeds the engine roughly eight times faster than real time whenever it is
willing to accept data, and stalls automatically when it is not. Both numbers
are in `Constants.ets` and they are the pair to tune - not the loop shape.

The three exit paths are the part the sample gets wrong. `clearInterval` and
`closeSync` appear only on the `len === 0` branch, so an ASR error leaves a
5 ms timer hammering a shut-down engine for the lifetime of the page, with the
PCM file descriptor still open (`HW-13-0091`). Factor the teardown into one
private method and call it from all three.

And the reset of `writeOffset` must not happen before `finish()`
(`HW-13-0092`): `finish` flushes the last utterance, and the resulting
`onResult` computes its end time from `writeOffset`. Reset it in
`startRecognizer` instead, where a new session begins.

**Turning bytes into timestamps — same file** (as shipped)

```typescript
onResult: (sessionId: string, result: speechRecognizer.SpeechRecognitionResult): void => {
  let time = this.getTime(this.writeOffset);
  time = Math.max(0, time - Constants.RECOGNIZE_DELAY)
  if (this.startTime === -1) {
    this.startTime = time;                          // first partial result of a sentence
  }
  if (result.isFinal) {
    if (result.result.length > 0) {
      let subtitle: Subtitle = { startTime: this.startTime, endTime: time, content: result.result };
      let eventData: emitter.EventData = { data: { 'subtitle': subtitle } };
      emitter.emit('Subtitle', eventData);
    }
    this.startTime = -1;                            // arm for the next sentence
  }
},

private getTime(offset: number) {
  return (offset * 8 * 1000) / (Constants.SAMPLE_RATE * Constants.SAMPLE_BIT * Constants.SOUND_CHANNEL);
}
```

**`getTime` is the whole timing model in one expression**: bytes times eight
gives bits, divided by bits per second (16000 x 16 x 1) gives seconds, times
1000 gives milliseconds. It only holds because the ffmpeg command produced
headerless PCM at exactly those parameters, and because `writeOffset` counts
bytes actually fed to the engine.

`RECOGNIZE_DELAY` (1300 ms) is a fudge factor for the recogniser's latency:
results arrive after the audio that produced them was written, so the raw
offset runs ahead of the speech. Subtracting a constant is crude - the real lag
varies with utterance length - and `Math.max(0, ...)` keeps the first sentence
off a negative time. `startTime === -1` as the "no sentence in progress"
sentinel is what collapses a run of partial results into one record at
`isFinal`.

**Displaying the caption — `entry/src/main/ets/viewmodel/VideoPlayer.ets`** (as shipped)

```typescript
this.videoPlayer?.on('timeUpdate', (time: number) => {
  AppStorage.setOrCreate('CurrentTime', time);

  if (this.currSubtitle === undefined || time < this.currSubtitle.startTime || time > this.currSubtitle.endTime) {
    let content = '';
    let length = this.subtitles.length;
    for (let idx = 0; idx < length; idx++) {
      let start = this.subtitles[idx].startTime;
      let end = this.subtitles[idx].endTime;
      if (time >= start && time <= end) {
        content = this.subtitles[idx].content;
        this.currSubtitle = { content: content, startTime: start, endTime: end };
        break;
      }
    }
    AppStorage.setOrCreate('Subtitle', content);
  }
});
```

**The `currSubtitle` guard is what keeps a linear scan affordable.**
`timeUpdate` fires several times a second; re-scanning the array each time
would be wasteful and would also thrash `AppStorage`. The guard means the scan
runs only on the transition out of the current caption's range - which is also
exactly when the displayed text has to change. Because the array is appended
to while playback is already running, an index or a binary search would need
maintaining; a scan over a few dozen records does not.

This is also where `HW-13-0092` shows: a record whose `endTime` is 0 while its
`startTime` is positive can never satisfy `time >= start && time <= end`, so
the last sentence of every video is silently unreachable.

The page renders it with a plain conditional over
`@StorageProp('Subtitle') subtitle: string = ''`, guarded by
`this.subtitle.length > 0 && this.subtitleShow`, so an empty string removes the
caption layer entirely rather than drawing an empty box.

## Permissions & config

**None.** The sample declares no `requestPermissions`. Everything it touches -
the bundled `input.mp4`, its own sandbox `filesDir`, the on-device recogniser -
is inside the application's own boundary.

Two configuration points do matter:

- `speechRecognizer.createEngine({ online: 1, language: 'zh-CN', extraParams: { 'locate': 'CN', 'recognizerMode': 'long' } })`.
  `recognizerMode: 'long'` is what allows a whole video rather than one
  utterance; `maxAudioDuration` (8 hours here) is passed separately in
  `StartParams.extraParams`.
- `EntryAbility` pins `COLOR_MODE_LIGHT` and calls
  `setWindowLayoutFullScreen(true)`; the page forces
  `window.Orientation.LANDSCAPE` in `aboutToAppear` and never restores it, so a
  portrait page opened after this one has to reset the orientation itself.

## Constraints

- The document says API Version 20 Release and HarmonyOS 6.0.0 Release SDK;
  the project builds at `5.0.5(17)`. Believe the project.
- Requires the third-party `@ohos/mp4parser` `^2.0.6` from OHPM. The whole
  extraction step is that library's bundled ffmpeg; there is no system API in
  this sample doing the demux.
- Recognition is Chinese only as configured (`language: 'zh-CN'`,
  `locate: 'CN'`). Other locales need both fields changed and a check against
  what the device's engine supports.
- The video source is a hardcoded rawfile `input.mp4`. There is no picker, no
  network source, and the "next" button in the control bar has no handler.
- Extraction and recognition happen on every launch, uncached, so a long video
  is re-transcribed each time the page opens.
- Recognition is far slower than playback for anything but a short clip, so
  early captions appear late and the beginning of a long video plays uncaptioned
  even after `HW-13-0090` is fixed. For production, transcribe ahead of
  playback or persist the result.

## Pitfalls

- **`HW-13-0090` (D/high, confirmed) — recognition starts before the async
  ffmpeg extraction finishes, and `len === 0` is treated as end-of-audio.**
  `extractAudio` is called without `await` (and would not be awaitable as
  written), then `startRecognizer` runs immediately; `onStart` opens
  `output.pcm` - which may not exist yet - and its 5 ms loop finishes the
  session the first time the reader overtakes the writer. Subtitles come out
  truncated or missing, differently on each launch. Fix: resolve a promise from
  ffmpeg's `callBackResult` and await it before `startListening`, so a zero-length
  read is a genuine EOF.
- **`HW-13-0091` (D/medium, confirmed) — `onError` shuts the engine down but
  never clears the 5 ms interval or closes the audio fd.** The timer keeps
  firing against a dead engine for the life of the page and the descriptor
  leaks. Fix: one teardown method (`clearInterval` + `closeSync`) called from
  the EOF branch, `onComplete` and `onError` alike.
- **`HW-13-0092` (B/medium, probable) — `writeOffset` is reset to 0 before
  `finish()`.** The final `onResult` callbacks that `finish` triggers compute
  their time from the already-zeroed offset, so the last sentence is stored
  with `endTime` 0 while `startTime` is positive and can never match the
  `timeUpdate` range test. Fix: reset after the callbacks have run, or snapshot
  the offset at the start of the EOF branch.
- **`HW-13-0005` (B/medium, confirmed) — systematic: five media samples create
  an AVPlayer or SoundPool and never release it,** and `VideoSubtitle` is one of
  them. `VideoPlayer.ets` has no `release()` anywhere, and the page has no
  `aboutToDisappear`. The raw file descriptor from `getRawFd` in
  `setMediaAsset` is never closed either. Fix: add `aboutToDisappear` on the
  page calling into a `VideoPlayer.release()` that releases the player, then
  closes the raw fd, then tears down the recogniser and the `emitter`
  subscription.
- **The `emitter.on('Subtitle')` subscription is never removed** and neither is
  `emitter.on('VideoWH')` in the page. On a page that can be re-entered, each
  visit adds a listener and old `VideoPlayer` instances keep receiving results.
- **The document's constraints contradict the project's own build profile** on
  the minimum API version (20 in prose, 17 in `build-profile.json5`).

## References

- `documentation/harmonyos-references/07_ai/hms-ai-speechrecognizer.md` - `createEngine`, `StartParams`, `AudioInfo`, `writeAudio`, `isBusy`, `finish`, `RecognitionListener`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/hms-ai-speechrecognizer
- `documentation/harmonyos-guides/08_ai/speechrecognizer-guide.md` - the ASR session lifecycle and the audio-feed loop
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/speechrecognizer-guide
- `documentation/harmonyos-guides/08_ai/core-speech-kit-guide.md` - Core Speech Kit overview and engine modes
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/core-speech-kit-guide
- `documentation/harmonyos-references/07_ai/errorcode-corespeech.md` - the codes arriving in `onError`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/errorcode-corespeech
- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - `fdSrc`, `timeUpdate`, `durationUpdate`, `stateChange`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-guides/02_media/video-playback.md` - the AVPlayer + XComponent surface flow used by the page
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/video-playback
- `huawei_industry_tree/13_media_entertainment/docs/41_video_subtitle.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_subtitle-0000002568904821
- `MEDIA-18` - the player-release systematic (`HW-13-0005`) this sample is an instance of
- `MEDIA-23` - audio extraction from a container, the system-API alternative to mp4parser
