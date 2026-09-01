---
id: MEDIA-35
title: PCM audio editing - cut a raw PCM file by byte offset, with a canvas waveform and a file-path undo stack
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/35_pcm_audio_edit.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/pcm_audio_edit-0000002476817872
sample: huawei_industry_tree/13_media_entertainment/downloads/PCMAudioEdit.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.AudioKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [abilityAccessCtrl, audio, common, fileIo, hilog, util, window]
permissions: [ohos.permission.MICROPHONE]
min_api: 20
modules: [entry]
findings: [HW-13-0061, HW-13-0062, HW-13-0078, HW-13-0079, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when the user has to **edit audio they just recorded** - a voice
memo with a cough in the middle, a podcast take with dead air at the front, a
sung phrase to be replaced. The whole feature rests on one property of raw PCM:
it has no container, no frame boundaries and no index, so *time maps to a byte
offset by arithmetic alone* and a cut is a file copy that skips a range.

That is the transferable idea. `offset = ms * sampleRate * channels *
bytesPerSample / 1000`. Recording an overwrite is the same expression used as a
write offset instead of a cut boundary. Any format with framing - AAC, MP3,
Opus - loses this and forces you through a demuxer, which is why the samples in
this industry record PCM first and only wrap it afterwards (see `MEDIA-36`).

The second half of the card is the waveform: an `AudioCapturer` `periodReach`
callback turning a window of samples into one bar height, a `Canvas` drawing a
scrolling strip of those bars, and a divide/delete/undo/redo model layered on
top. **Read `HW-13-0078` before you ship any of it** - the shipped cut writes
one byte too many, and on 16-bit audio every delete leaves the file misaligned.

## Feature checklist

- Record PCM from the microphone into a sandbox file, with a 30-minute cap.
- A live waveform scrolls under a fixed centre indicator line while recording.
- Drag the waveform to move the indicator; the elapsed and indicator times both
  update.
- Enter clip mode; the indicator becomes an edit cursor.
- **Divide** splits the region under the indicator into two, drawn with a gap.
- **Delete** removes the region under the indicator from both the waveform and
  the PCM file, and shortens the total duration.
- **Undo** and **Redo** step through the operation history, restoring both the
  waveform and the audio.
- Cancel-clip restores the original recording; save-clip commits the edits and
  discards the temp files.
- Playback starts from the indicator position, and resumes from the start when
  the indicator is already at the end.
- Recording with the indicator moved back overwrites from that point.

## Architecture

One `entry` module, and an unusually clean separation for a sample of this size:
the audio layer, the waveform data model and the drawing controller do not know
about each other's UI.

```
entry/src/main/ets
├── constants/Constants.ets         Wave.* drawing geometry + AudioOptions.* stream config
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── model/
│   ├── audio/AudioRecorder.ets     AudioCapturer -> file, dB window, the edit/undo file stack
│   ├── audio/AudioPlayer.ets       AudioRenderer <- file, playback from an arbitrary offset
│   ├── CanvasController.ets        all Canvas drawing; owns preciseOffset
│   ├── IAudioEdit.ets              the state enums and the WaveArea / WaveOperation shapes
│   ├── ListenerBase.ets            a tiny key -> callback[] bus, the only coupling mechanism
│   └── WaveSampler.ets             the waveform data model: divide areas, undo/redo, remapping
├── pages/AudioEdit.ets             @Entry (@ComponentV2), owns the four objects and the buttons
├── utils/
│   ├── ArrayUtil.ets               remapArray: regroup samples when the zoom interval changes
│   ├── FileManager.ets             cutFile - the byte-level edit
│   ├── Logger.ets
│   ├── PermissionManager.ets       MICROPHONE, with the requestPermissionOnSetting fallback
│   └── TimeUtil.ets                the milliseconds <-> byte-offset conversions
└── view/
    ├── AudioClipView.ets           the toolbar (clip/divide/delete/undo/redo) + time readouts
    └── AudioWaveView.ets           the Canvas host and its gestures
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that **undo is a stack of file paths,
not a stack of diffs**. `cutFile` never mutates its input: it writes a brand
new file into a `temp/` directory and returns the path. `AudioRecorder` keeps
`historyFilePaths: string[]` plus an index, so undo is `historyFileIndex -= 1`
and redo is `+= 1`. Nothing has to be inverted, nothing can be half-applied,
and a crash mid-edit leaves the original intact. `saveAudio` promotes the
current file out of `temp/` and deletes the rest; `restoreAudio` drops the
whole temp directory.

That decision is only affordable because the recordings are small and bounded
(30 minutes of 44.1 kHz mono S16LE is about 158 MB, and each edit copies the
whole file). It is exactly the right trade for voice memos and exactly the
wrong one for a multi-track editor.

The parallel structure on the waveform side is `WaveSampler`, which keeps
`oriWaveDataList` (never edited until save) beside `waveDataList` (what is
drawn) and an `undoRecord`/`redoRecord` pair of `WaveOperation`s. The two
histories are stepped together from `AudioClipView.onClickUndo`, which is the
one place that knows both.

## Implementation steps

1. **Fix the stream format up front and reuse it everywhere.**
   `SAMPLE_RATE_44100`, `CHANNEL_1`, `SAMPLE_FORMAT_S16LE`, `ENCODING_TYPE_RAW`,
   `CH_LAYOUT_MONO`. Every offset calculation depends on all five.
2. **Request `MICROPHONE` before creating the capturer,** and fall back to
   `requestPermissionOnSetting` when `dialogShownResults[0] === false`.
3. **Write in `on('readData')`,** advancing your own `offset` rather than
   relying on the file position - that is what makes overwrite-from-a-point
   work with the same code path as append.
4. **Report the waveform on `on('periodReach', frame, cb)`,** where `frame =
   samplingRate / 1000 * intervalMs`. One callback becomes one bar.
5. **Convert time to bytes in one helper.** `Math.floor(ms * sampleRate *
   channels * bytes / 1000)` - `TimeUtil.millisecondsToFileSize`.
6. **Cut by copying the head and the tail into a new file.** Length of the tail
   is `srcFileSize - lastPos`, with **no `+ 1`** (`HW-13-0078`).
7. **Push the new path onto the history stack** and truncate any redo tail
   before pushing.
8. **Return early when the dB window is empty,** do not fall through into a
   `0 / 0` (`HW-13-0079`), and take the square root before the log
   (`HW-13-0061`).
9. **Zero the tail of the render buffer** on the last partial read
   (`HW-13-0062`).

## Verified snippets

All snippets are from `PCMAudioEdit.zip`. Corrected forms are marked.

**Time to bytes - `entry/src/main/ets/utils/TimeUtil.ets`** (as shipped)

```typescript
const sampleFormatBytes: [audio.AudioSampleFormat, number][] = [
  [audio.AudioSampleFormat.SAMPLE_FORMAT_U8, 1],
  [audio.AudioSampleFormat.SAMPLE_FORMAT_S16LE, 2],
  [audio.AudioSampleFormat.SAMPLE_FORMAT_S24LE, 4],
  [audio.AudioSampleFormat.SAMPLE_FORMAT_S32LE, 4],
  [audio.AudioSampleFormat.SAMPLE_FORMAT_F32LE, 4],
];

export function millisecondsToFileSize(milliseconds: number, streamInfo?: audio.AudioStreamInfo) {
  const audioStreamInfo = streamInfo ?? defaultAudioStreamInfo;
  const value = sampleFormatBytes.find(ele => ele[0] === audioStreamInfo.sampleFormat);
  const bytes: number = value ? value[1] : 2;
  return Math.floor(
    milliseconds * audioStreamInfo.samplingRate * audioStreamInfo.channels * bytes / secondToMillisecond
  );
}

export function fileSizeToMilliseconds(fileSize: number, streamInfo?: audio.AudioStreamInfo) {
  const audioStreamInfo = streamInfo ?? defaultAudioStreamInfo;
  const value = sampleFormatBytes.find(ele => ele[0] === audioStreamInfo.sampleFormat);
  const bytes: number = value ? value[1] : 2;
  return fileSize / audioStreamInfo.samplingRate / (audioStreamInfo.channels * bytes) * secondToMillisecond;
}
```

**This pair is the whole reason the feature is simple.** Duration is derived
from `lstatSync(path).size` on demand - there is no separate duration field to
keep in sync, so a cut automatically shortens the displayed time. Note
`SAMPLE_FORMAT_S24LE` mapping to 4 bytes rather than 3: that is correct for the
padded layout the audio framework hands back, and getting it wrong would skew
every offset in the app.

The `Math.floor` on the way in is deliberate but incomplete: it rounds to a
whole *byte*, not to a whole *frame*. For 16-bit mono a cut boundary can still
land on an odd byte, which is precisely what makes `HW-13-0078` audible.

**The cut - `entry/src/main/ets/utils/FileManager.ets`** (corrected, see `HW-13-0078`)

```typescript
public cutFile(srcFilePath: string, startPos: number, endPos: number,
               cutFileSaveDir?: string): string | undefined {
  try {
    const index = srcFilePath.lastIndexOf('/');
    const fileDir = cutFileSaveDir ? cutFileSaveDir : srcFilePath.slice(0, index);
    if (!fileIo.accessSync(fileDir)) {
      fileIo.mkdirSync(fileDir);
    }

    const dstFilePath = fileDir + '/' + util.generateRandomUUID();
    const dstFile = fileIo.openSync(dstFilePath, fileIo.OpenMode.WRITE_ONLY | fileIo.OpenMode.CREATE);
    const srcFile = fileIo.openSync(srcFilePath, fileIo.OpenMode.READ_ONLY);
    const srcFileSize = fileIo.statSync(srcFilePath).size;

    const lastPos = endPos > srcFileSize ? srcFileSize : endPos;
    if (startPos > 0) {                                  // the head, [0, startPos)
      const buffer = new ArrayBuffer(startPos);
      fileIo.readSync(srcFile.fd, buffer, { offset: 0, length: startPos });
      fileIo.writeSync(dstFile.fd, buffer, { offset: 0, length: startPos });
    }
    if (lastPos !== srcFileSize) {                       // the tail, [lastPos, srcFileSize)
      const tailLength = srcFileSize - lastPos;          // FIX: sample uses  ... - lastPos + 1
      const buffer = new ArrayBuffer(tailLength);
      fileIo.readSync(srcFile.fd, buffer, { offset: lastPos, length: tailLength });
      fileIo.writeSync(dstFile.fd, buffer, { offset: startPos, length: tailLength });
    }

    fileIo.closeSync(srcFile.fd);
    fileIo.closeSync(dstFile.fd);
    return dstFilePath;
  } catch (error) {
    logger.error(`cut file fail, code: ${error.code}, message: ${error.message}`);
  }
  return undefined;
}
```

**Two offsets, one file, no seeking.** `readSync`/`writeSync` both take an
explicit `offset`, so the head is read from 0 and written to 0, and the tail is
read from `lastPos` and written to `startPos` - closing the gap. The cut range
is never touched. Because the destination is a new UUID-named file in `temp/`,
the source stays valid and becomes the undo target for free.

The `+ 1` in the shipped code is not harmless rounding. `readSync` can only fill
`srcFileSize - lastPos` bytes, so the final byte of the buffer is whatever the
freshly allocated `ArrayBuffer` held - zero - and `writeSync` writes all
`length` bytes anyway. **The file therefore grows by exactly one byte per
delete.** On mono S16LE that makes the size odd: every subsequent frame boundary
is off by one byte, `fileSizeToMilliseconds` reports a fractional sample, and
any recording appended at `offset = fileSize` is byte-misaligned - the high and
low halves of each 16-bit sample swap, and the result is noise rather than
audio. Errors accumulate: two deletes, two bytes.

**One window of samples to one bar - `entry/src/main/ets/model/audio/AudioRecorder.ets`** (corrected, see `HW-13-0079`, `HW-13-0061`)

```typescript
private decibelRMSSum: number = 0;
private decibelRMSCount: number = 0;

private calcDecibelRMSSum(data: ArrayBuffer) {
  const amplitudes = new Int16Array(data);
  for (let i = 0; i < amplitudes.length; ++i) {
    const val = amplitudes[i] / AudioOptions.MAX_AMPLITUDE;   // MAX_AMPLITUDE = 32767
    this.decibelRMSSum += val * val;                          // sum of squares
    this.decibelRMSCount += 1;
  }
}

private periodReachCallback = (position: number) => {
  position;

  if (this.decibelRMSCount === 0) {
    this.notify('amplitudeScale', 0);
    return;                                                   // FIX: sample falls through into 0 / 0
  }

  const meanSquare = this.decibelRMSSum / this.decibelRMSCount;
  const rms = Math.sqrt(meanSquare);                           // FIX: sample logs the MEAN SQUARE
  const decibel = Math.min(AudioOptions.MAX_DECIBEL, Math.abs(20 * Math.log10(rms)));
  const amplitudeScale = Math.abs(AudioOptions.MAX_DECIBEL - decibel) / AudioOptions.MAX_DECIBEL;

  this.decibelRMSSum = 0;
  this.decibelRMSCount = 0;
  this.notify('amplitudeScale', amplitudeScale);
}
```

**`periodReach` is what makes the waveform frame-rate independent.** `start()`
registers it as `on('periodReach', frame, cb)` with `frame = samplingRate /
1000 * interval` - at 44.1 kHz and a 100 ms interval that is 4410 frames - so
the callback fires once per bar regardless of how large the `readData` buffers
happen to be. `readData` accumulates the sum of squares; `periodReach` drains
it and resets. The two callbacks share `decibelRMSSum` and `decibelRMSCount`
and nothing else.

Both corrections are in the drain. The `decibelRMSCount === 0` guard assigns 0
and then keeps going, so a `periodReach` that arrives before any samples
computes `0 / 0` = `NaN`, `log10(NaN)` = `NaN`, and a `NaN` bar height goes into
the waveform. And `rms` is misnamed: it holds the *mean square*, so
`20 * log10(meanSquare)` doubles every level - a true -30 dB signal reads as
-60 dB, bars are systematically far too short, and quiet audio clamps to the
90 dB floor early. `AudioVisualization` has the same formula in two files
(`HW-13-0061`).

**The undo stack - same file** (as shipped)

```typescript
private historyFilePaths: string[] = [];
private historyFileIndex: number = 0;

public deleteAudio(startTime: number, endTime: number) {
  this.unableRedoAudio();                                    // drop any redo tail first

  const startPos = TimeUtil.millisecondsToFileSize(startTime, this.audioSteamInfo);
  const endPos = TimeUtil.millisecondsToFileSize(endTime, this.audioSteamInfo);
  const newFilePath = fileMgr.cutFile(this.audioFileInfo.filePath, startPos, endPos,
                                      this.defaultTempFileDir);
  if (newFilePath) {
    this.historyFilePaths.push(newFilePath);
    this.historyFileIndex += 1;
    this.audioFileInfo.filePath = newFilePath;
    this.notify('recordTime', this.recordTime());            // duration re-read from the file
  }
}

public undoAudio() {
  if ((this.historyFileIndex - 1) >= 0 && this.historyFileIndex < this.historyFilePaths.length) {
    this.historyFileIndex -= 1;
    this.audioFileInfo.filePath = this.historyFilePaths[this.historyFileIndex];
    this.notify('recordTime', this.recordTime());
  }
}

public unableRedoAudio() {
  if (this.historyFileIndex >= 0 && this.historyFileIndex + 1 < this.historyFilePaths.length) {
    this.historyFilePaths.splice(this.historyFileIndex + 1);
  }
}

public saveAudio() {
  const fileName = this.audioFileInfo.filePath.slice(this.audioFileInfo.filePath.lastIndexOf('/') + 1);
  const newFilePath = this.defaultFileDir + '/' + fileName;
  if (this.historyFileIndex > 0 && this.historyFilePaths.length > 0) {
    fileIo.unlinkSync(this.historyFilePaths[0]);             // the original recording
    fileIo.moveFileSync(this.audioFileInfo.filePath, newFilePath);
    this.audioFileInfo.filePath = newFilePath;
  }
  if (fileIo.accessSync(this.defaultTempFileDir)) {
    fileIo.rmdirSync(this.defaultTempFileDir);               // every intermediate at once
  }
  this.historyFilePaths = [this.audioFileInfo.filePath];
  this.historyFileIndex = 0;
}
```

**Three lines are the whole undo model**: push on edit, index on undo/redo,
`splice` to invalidate the redo tail when a new edit lands mid-history. That
last call is the one implementations forget - without it, an undo followed by a
new delete leaves an orphaned branch that redo would walk into.

`saveAudio` is where the temp-file strategy pays off: one `rmdirSync` on
`temp/` disposes of every intermediate, and the surviving file is moved up into
`audio_files/`. `restoreAudio` is the mirror image - same `rmdirSync`, then
reset to `historyFilePaths[0]`. Note the guard `historyFileIndex > 0`: with no
edits the current file *is* the original and must not be unlinked before being
moved onto itself.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.MICROPHONE",
    "reason": "$string:microphone_reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
]
```

- `MICROPHONE` is `user_grant`, so `reason` and `usedScene` are mandatory and
  `$string:microphone_reason` must exist in `resources/base/element/string.json`.
- `when: "inuse"` is correct: there is no continuous task, so recording must
  stop when the app leaves the foreground.
- `PermissionManager` handles the permanent-refusal case properly - when
  `authResults[0]` is not 0 and `dialogShownResults[0] === false` it calls
  `requestPermissionOnSetting`, which opens the settings sheet.
- `AudioEdit.init()` is re-invoked from the record button when
  `isMicrophone` is false, so a user who denies and then changes their mind
  gets a second chance without restarting.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `deviceTypes` is `phone` only.
- The stream is fixed at 44.1 kHz mono S16LE; the UI exposes no format choice
  and the drawing geometry assumes one bar per 100 ms.
- Recording is capped at 30 minutes (`MAX_RECORD_DURATION`), and
  `AudioEdit.onRecordTimeChange` flips `isRecording` off when the cap is hit.
- **Every edit copies the whole file.** At the 30-minute cap that is a ~158 MB
  copy per delete, and every undo step keeps another copy in `temp/` until save
  or cancel.
- `init()` deletes the entire `audio_files` directory on every launch, so
  recordings do not survive a restart. That is the sample being a sample, but
  it also means a crash during an edit loses everything.
- Only `delete` touches the audio file; `divide` is a waveform-only operation
  that marks a boundary for a later delete.
- The waveform is redrawn imperatively from `CanvasController.drawCanvas()`;
  it is not driven by state, so any new mutation path has to remember to call
  it.

## Pitfalls

- **`HW-13-0078`** (B/high, confirmed): `cutFile` allocates, reads and writes
  `srcFileSize - lastPos + 1` bytes for the tail. `readSync` can only fill
  `srcFileSize - lastPos`, so a stale zero byte is appended and **the PCM file
  grows by one byte per delete**. On mono S16LE the file becomes odd-sized:
  duration arithmetic is off, and any recording appended at
  `offset = fileSize` is byte-misaligned, so all further recorded audio is
  noise. Errors accumulate per delete. Fix: drop the `+ 1`.
- **`HW-13-0079`** (B/low, confirmed): the `decibelRMSCount === 0` guard in
  `periodReachCallback` sets `amplitudeScale = 0` but does not return, so the
  code falls through into `0 / 0` and pushes a `NaN` bar height into the
  waveform. A `periodReach` that lands before any samples corrupts the drawing.
  Fix: return after the guard.
- **`HW-13-0061`** (B/medium, confirmed): the dB formula omits the square root.
  `decibelRMSSum / decibelRMSCount` is the *mean square*, and
  `20 * log10(meanSquare)` doubles every level, so bars are systematically far
  too short and quiet audio clamps to the floor early - the sample's core
  visual is wrong. Same defect in `AudioVisualization`'s capturer and renderer
  managers. Fix: `Math.sqrt` first, or use `10 * log10(meanSquare)`.
- **`HW-13-0062`** (B/low, confirmed): the systematic stale-tail defect.
  `AudioPlayer.writeDataCallback` clamps `readLen` to the remaining file bytes
  but reads into the renderer's buffer without zeroing the rest, so the final
  partial frame of every playback renders whatever the previous frame left
  behind. The finding's evidence names `AudioVisualization`, `PCMTranscode` and
  two others; PCMAudioEdit's `AudioPlayer.ets` carries the identical callback.
  Fix: `memset` the remainder of the buffer beyond `readLen`.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-audio-audiocapturer.md` - `createAudioCapturer`, `on('readData')`, `on('periodReach')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio-audiocapturer
- `documentation/harmonyos-references/04_media/arkts-apis-audio-audiorenderer.md` - `createAudioRenderer`, `on('writeData')`, the partial-write contract
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-audio-audiorenderer
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `openSync`, `readSync`/`writeSync` with explicit offsets, `moveFileSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - the `Canvas` the waveform is drawn on
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-guides/03_application-framework/arkts-new-observedv2-and-trace.md` - `@ObservedV2` / `@Trace` / `@Monitor`, used by `WaveSampler` and the views
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-observedv2-and-trace
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `MICROPHONE` as a `user_grant` permission
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `MEDIA-27` - the same dB formula defect, and the waveform animation card
- `MEDIA-36` - the next step: wrapping the edited PCM into m4a or amr
