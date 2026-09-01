---
id: MEDIA-30
title: Download a track over HTTP and play it through a native OHAudio renderer
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/30_music_demo_httpaudiorender.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/music_demo_httpaudiorender-0000002330168664
sample: huawei_industry_tree/13_media_entertainment/downloads/demo_HttpAudioRender.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.AudioKit", "@kit.BasicServicesKit", "@kit.NetworkKit", "@kit.PerformanceAnalysisKit"]
apis: ["http.createHttp", "http.HttpDataType", "fs.openSync", "fs.OpenMode", "fs.writeSync", "MP4Parser.ffmpegCmd", OH_AudioStreamBuilder_Create, OH_AudioStreamBuilder_GenerateRenderer, OH_AudioRenderer_Start, OH_AudioRenderer_GetCurrentState, napi_get_value_string_utf8, abilityAccessCtrl, audio, common, fs, hilog, http, window]
permissions: [ohos.permission.INTERNET]
min_api: 20
modules: [entry]
findings: [HW-13-0067, HW-13-0068, HW-13-0069, HW-13-0039, HW-13-0062, HW-13-0003, HW-13-0096, HW-13-0097, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when audio arrives **from the network in a format the platform
player will not take**, and the app has to land it on disk, transcode it, and
push raw PCM at an output stream itself. Here the server answers an SSE-style
stream whose `data:` lines carry a hex-encoded MP3; the app decodes the hex into
an `ArrayBuffer`, writes an `.mp3` into `cacheDir`, shells it through
`MP4Parser.ffmpegCmd` to 48 kHz stereo `s16le` PCM, and hands the PCM path to a
NAPI module that drives `OH_AudioRenderer`.

That is a long pipeline, and most apps should not build it: `AVPlayer` plays a
downloaded MP3 directly. Reach for this shape only when you actually need the
native renderer - low-latency mode, a custom mixer, generated or streamed PCM,
or an existing C/C++ audio engine you are porting. The transferable pieces are
the **hex-to-ArrayBuffer-to-file** hop, and the **NAPI boundary**: ArkTS owns
the file path and the UI, C++ owns one global renderer and answers state
queries.

**Every stage of the shipped pipeline has a defect.** The parser drops the
payload unless it is on line 0 (`HW-13-0067`), the file helpers close
descriptors they never opened (`HW-13-0068`, `HW-13-0039`), the state query
leaks a native builder twenty times a second (`HW-13-0069`), and the write
callback renders stale bytes on the final partial block (`HW-13-0062`). Treat
the sample as a map of the pipeline, not as code to lift.

## Feature checklist

- A music home page with recommendation cards and a bottom tab bar; a 我的
  (mine) tab listing what has been downloaded locally.
- Opening a track pushes a play page with a download button.
- The download button fetches the track over HTTP, converts the hex payload to
  bytes, writes an MP3 into `cacheDir`, transcodes it to PCM, appends the result
  to the local list, toasts 下载成功，本地音乐查看 (downloaded - see local
  music) and pops back.
- Tapping an entry in the local list initialises the native renderer against
  that PCM file.
- The play bar's button starts, pauses and resumes the native renderer, and its
  icon follows the renderer's real state.
- When the file reaches EOF, playback stops, the renderer is released and
  re-initialised so the same track can be played again.

## Architecture

One `entry` module, with a `cpp` half that is the actual audio engine.

```
entry/src/main/cpp
├── CMakeLists.txt                 links libohaudio.so, libace_napi.z.so, libhilog_ndk.z.so
├── napi_init.cpp                  the whole renderer: one global OH_AudioRenderer + FILE* (241 lines)
└── types/libentry/Index.d.ts      the nine exported natives, typed for ArkTS

entry/src/main/ets
├── common/Constants.ets           one field: the hilog domain
├── components/
│   ├── CardBigComponent.ets       recommendation hero cards
│   ├── CardSmallComponent.ets     recommendation strip
│   ├── HomeComponent.ets          the 推荐 tab
│   ├── MineComponent.ets          the 我的 tab: the local list, taps call renderCreate
│   └── PlayLineComponent.ets      the mini play bar, taps call render()
├── entryability/EntryAbility.ets  immersive setup, avoid areas -> AppStorage
├── models/
│   ├── MusicData.ets              DataEntry / ResultData - the shape of one SSE data: line
│   └── MusicModel.ets             MusicInfo + the two static card lists
├── pages/
│   ├── Index.ets                  @Entry, Navigation + four Tabs, requests permissions
│   └── PlayPage.ets               NavDestination, the download button runs the whole pipeline
└── utils/
    ├── AudioRendererUtil.ets      the ArkTS side of the renderer + the 50 ms EOF poll
    ├── FileUtil.ets               hex -> ArrayBuffer -> .mp3 in cacheDir
    ├── HttpUtil.ets               postHttp (hex string) and getHttp (ArrayBuffer)
    ├── Mp4ParserUtil.ets          the ffmpeg command that produces the PCM
    └── PermissionUtil.ets         requestPermissionsFromUser for INTERNET
```

The documented tree is wrong in one entry: it lists
`cpp/types/libentry/index.d.ets`, and the zip ships `Index.d.ts` - both the case
and the extension differ, and the projects enable `caseSensitiveCheck`
(`HW-13-0003`). The extension is the part that matters: a NAPI typing file is
`.d.ts`, not `.d.ets`.

**The design decision worth copying** is the contract between the two halves.
ArkTS never touches audio data. It produces a *file path* and calls
`audioRendererInit(path)`; C++ owns exactly one `OH_AudioRenderer`, one
`OH_AudioStreamBuilder` and one `FILE*` as statics, and exposes three booleans
back (`getRendererState`, `getFileState`, `getFastState`). That keeps the NAPI
surface to nine void-ish functions with no buffers crossing the boundary, and it
is why `AudioRendererUtil` can be a plain state machine.

The load-bearing coupling to notice is between the ffmpeg command and the
builder:

```
ffmpeg -i <mp3> -f s16le -ar 48000 -ac 2 <pcm>
OH_AudioStreamBuilder_SetSamplingRate(rendererBuilder, 48000);
OH_AudioStreamBuilder_SetChannelCount(rendererBuilder, 2);
```

Raw PCM carries no header, so nothing at runtime can detect a mismatch - change
one and the audio plays at the wrong pitch or in the wrong channel order. Both
numbers should come from one place; in this sample they are a literal in a
string and two globals in `napi_init.cpp`.

## Implementation steps

1. **Declare `ohos.permission.INTERNET`** in `module.json5`. It is
   `system_grant` - it does not need `requestPermissionsFromUser` (the sample
   asks anyway; see Permissions & config).
2. **Fetch with `http.createHttp()`** and `expectDataType: http.HttpDataType.STRING`
   for the hex stream, or `ARRAY_BUFFER` when the endpoint returns the file
   directly. Register `headersReceive`, and `off` it plus `destroy()` on every
   exit path.
3. **Parse every `data:` line, not just the first** (`HW-13-0067`). Split on
   `\n`, take the lines that start with `data:`, `JSON.parse` the remainder and
   keep the first non-empty `audio` field; reject if none was found rather than
   resolving `undefined`.
4. **Convert hex to bytes** two characters at a time into a `Uint8Array` over a
   pre-sized `ArrayBuffer` - `new ArrayBuffer(hex.length / 2)`.
5. **Write the buffer to `cacheDir`** with `fs.OpenMode.CREATE | fs.OpenMode.READ_WRITE`.
   The bitwise `|` is required; `||` evaluates to `CREATE` alone and the write
   then fails on a read-only handle (`HW-13-0068`).
6. **Guard the cleanup**: only close a descriptor that was actually opened, and
   return `undefined` on the error path instead of a path that was never
   written (`HW-13-0068`, `HW-13-0039`).
7. **Transcode to PCM** with `MP4Parser.ffmpegCmd('ffmpeg -i ... -f s16le -ar 48000 -ac 2 ...')`
   and resolve only when the callback reports code `0`.
8. **Build the renderer natively**: create the builder, set sampling rate,
   channel count and latency mode, install the `OH_AudioRenderer_OnWriteData`
   callback, `GenerateRenderer`, then `OH_AudioRenderer_Start`.
9. **Read the file correctly in the write callback**: `fread` with
   `size = 1, nmemb = bufferLen` so a short final block still returns its bytes,
   and zero the remainder of the buffer before returning (`HW-13-0062`).
10. **Do not allocate in a state query** (`HW-13-0069`). `getRendererState` is
    polled every 50 ms while playing; it must do nothing but read the state.
11. **Poll for EOF from ArkTS** with a `setInterval`, and `clearInterval` on
    every exit including pause - the sample does this correctly.

## Verified snippets

All snippets are from `demo_HttpAudioRender.zip`. Corrected forms are marked.

**The SSE payload parser — `entry/src/main/ets/utils/HttpUtil.ets`** (corrected, see `HW-13-0067`)

```typescript
static async postHttp(url: string): Promise<string> {
  return new Promise((resolve, reject) => {
    let music: string = '';                       // FIX: was declared unassigned
    let httpRequest = http.createHttp();
    httpRequest.on('headersReceive', (header) => {
      hilog.info(0x0000, 'testTag', 'postHttp---headersReceive:%{public}s', `${header}`);
    });
    httpRequest.request(url, {
      method: http.RequestMethod.POST,
      header: { 'Content-Type': 'application/json' },
      extraData: 'data to send',
      expectDataType: http.HttpDataType.STRING,
      usingCache: true,
      priority: 1,
      connectTimeout: 60000,
      readTimeout: 60000,
      usingProtocol: http.HttpProtocol.HTTP1_1,
      usingProxy: false
    }, async (error: BusinessError, data: http.HttpResponse) => {
      if (!error) {
        let strr = data.result as string;
        let lines = strr.split('\n');
        for (let i = 0; i < lines.length; i++) {
          if (lines[i] && lines[i] !== '' && lines[i].startsWith('data:')) {
            let linei = lines[i].slice(5);
            let dataEntry: DataEntry = JSON.parse(linei) as DataEntry;
            let resultData: ResultData = dataEntry.data;
            let song = resultData.audio;
            if (song && song !== '') {            // FIX: shipped code also requires i === 0
              music = song;
              break;
            }
          }
        }
        httpRequest.off('headersReceive');
        httpRequest.destroy();
        if (music === '') {
          reject(new Error('no audio payload in response'));   // FIX: shipped code resolves undefined
        } else {
          resolve(music);
        }
      } else {
        httpRequest.off('headersReceive');
        httpRequest.destroy();
        reject(error);
      }
    });
  });
}
```

**The `i === 0` in the shipped condition defeats the parser.** An SSE response
is a sequence of `data:` lines; a stream that opens with a status or metadata
line - the normal case - puts the audio anywhere but index 0. When it does,
`music` is never assigned, `resolve(music)` hands `undefined` to
`hexStringToArrayBuffer`, and `hex.length` throws inside the download handler
with no error text about the response at all. The `break` in the corrected form
also makes the intent explicit: take the first payload, ignore the rest.

Note `expectDataType: http.HttpDataType.STRING`. If it were left at the default
the framework would guess from the content type, and a hex body served as
`application/octet-stream` would arrive as an `ArrayBuffer` that
`split('\n')` cannot handle.

**The file hops — `HttpUtil.getHttp` and `FileUtil.arrayBufferToMP3`** (corrected, see `HW-13-0068`, `HW-13-0039`)

```typescript
// HttpUtil.getHttp - the shipped prologue opens a scratch file it never uses:
//   let file = fs.openSync(context.cacheDir + '/new.text',
//                          fs.OpenMode.CREATE || fs.OpenMode.READ_WRITE);   // ||  -> CREATE only
//   ...
//   } catch (e) {
//   } finally { fs.closeSync(fd!); }                                        // fd is undefined here
// FIX: delete the whole block - getHttp resolves an ArrayBuffer and touches no file.

// FileUtil.arrayBufferToMP3 - corrected
static async arrayBufferToMP3(arraybuffer: ArrayBuffer,
  context: common.UIAbilityContext): Promise<string | undefined> {
  let filePath = `${context.cacheDir}/temp_audio_${Date.now()}.mp3`;
  let file: fs.File | undefined = undefined;                 // FIX: was `let fd = 0`
  try {
    file = fs.openSync(filePath, fs.OpenMode.CREATE | fs.OpenMode.READ_WRITE);
    fs.writeSync(file.fd, arraybuffer);
  } catch (error) {
    hilog.error(0x0000, 'testTag', 'arrayBufferToMP3---error:%{public}s',
      `code is ${error.code}, message is ${error.message}`);
    return undefined;                                        // FIX: was `return filePath` regardless
  } finally {
    if (file) {                                              // FIX: was fs.closeSync(0) - stdin
      fs.closeSync(file.fd);
    }
  }
  return filePath;
}
```

**Three defects live in six lines, and they are the industry's signature ones.**
`fs.OpenMode.CREATE || fs.OpenMode.READ_WRITE` is a logical or over two flag
constants: `CREATE` is non-zero, so the expression is `CREATE` and the
`READ_WRITE` bit is silently dropped - `|` is the operator these flags want.
The `fd = 0` initialiser turns the `finally` into `fs.closeSync(0)`, which
closes **stdin** when the open failed. And returning `filePath` from a function
whose `try` threw feeds a nonexistent file into the ffmpeg step, which then
fails with a code that points at the transcoder rather than at the write.
`HW-13-0039` records the same `finally`-block shape across five samples in this
industry; `HW-13-0068` is the `||` half.

**The native renderer — `entry/src/main/cpp/napi_init.cpp`** (corrected, see `HW-13-0069`, `HW-13-0062`)

```cpp
static OH_AudioRenderer *audioRenderer;
static OH_AudioStreamBuilder *rendererBuilder;

static napi_value GetRendererState(napi_env env, napi_callback_info info)
{
    // FIX: the shipped body starts with
    //   OH_AudioStreamBuilder *builder;
    //   OH_AudioStreamBuilder_Create(&builder, AUDIOSTREAM_TYPE_RENDERER);
    // which is never used and never destroyed.
    OH_AudioStream_State state;
    OH_AudioRenderer_GetCurrentState(audioRenderer, &state);
    napi_value sum;
    napi_create_int32(env, state, &sum);
    return sum;
}

static int32_t AudioRendererOnWriteData(OH_AudioRenderer *renderer, void *userData,
                                        void *buffer, int32_t bufferLen)
{
    // FIX: shipped call is fread(buffer, bufferLen, 1, g_file) - one element of bufferLen
    // bytes, so a short final block reads 0 elements and the previous contents are rendered.
    size_t readCount = fread(buffer, 1, static_cast<size_t>(bufferLen), g_file);
    if (readCount < static_cast<size_t>(bufferLen)) {
        std::memset(static_cast<char *>(buffer) + readCount, 0, bufferLen - readCount);
        if (feof(g_file)) {
            g_readEnd = true;
        }
    }
    return 0;
}
```

**A state getter must not allocate.** `GetRendererState` is called from ArkTS on
every tick of a 50 ms interval for the whole duration of playback - twenty
native `OH_AudioStreamBuilder` objects per second, none of them destroyed, none
of them used for anything. The renderer the function actually queries is the
static `audioRenderer` built in `AudioRendererInit`; the local builder is
leftover boilerplate from the creation path.

**`fread`'s argument order decides whether the last frame is audible.**
`fread(buffer, bufferLen, 1, g_file)` asks for *one* element of `bufferLen`
bytes and returns 0 when fewer bytes remain, without telling the caller how many
it copied - the tail of the file is dropped and OHAudio renders whatever the
buffer held from the previous callback, which is the echo of an earlier block.
Swapping the two middle arguments makes the return value a byte count, and
zeroing the remainder guarantees silence rather than repeat. `HW-13-0062` tracks
this pattern across four samples in this industry.

**The EOF poll — `entry/src/main/ets/utils/AudioRendererUtil.ets`** (as shipped)

```typescript
renderStart(filepath: string) {
  try {
    testNapi.audioRendererStart();
    this.renderState = testNapi.getRendererState();
    this.interval = setInterval(async () => {
      if (testNapi.getFileState()) {                 // g_readEnd flipped by the write callback
        testNapi.audioRendererStop();
        testNapi.audioRendererRelease();
        if (testNapi.getFastState()) {
          testNapi.audioRendererLowLatencyInit();
        } else {
          testNapi.audioRendererInit(filepath);      // re-arm so the track can be replayed
        }
        clearInterval(this.interval);
        this.renderState = testNapi.getRendererState();
        return;
      }
    }, 50);
  } catch (err) {
    let error = err as BusinessError;
    hilog.error(0x0000, 'testTag', 'audioRendererStart---error:%{public}s',
      `code is ${error.code}, message is ${error.message}`);
  }
}
```

**This is the ArkTS side of "the file ended".** OHAudio's write callback runs on
an audio thread in C++ and cannot touch the ArkTS VM, so the sample flips a
global `g_readEnd` there and lets ArkTS discover it by polling. 50 ms is a
reasonable period for an end-of-track transition, and the interval is cleared on
the EOF branch and in `renderPause` - unlike the polling in `MEDIA-17` /
`MEDIA-32` (`HW-13-0046`), this one has no leak. It is, however, the multiplier
that turns `HW-13-0069` from a curiosity into twenty leaked native objects per
second.

The re-init in the EOF branch is the part worth copying: an `OH_AudioRenderer`
that has hit the end of its stream is released, and a fresh one is built against
the same path, so a second tap on the play button starts from the beginning
instead of finding a dead renderer.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.INTERNET",
    "reason": "$string:app_name",
    "usedScene": {
      "abilities": ["EntryAbility"],
      "when": "always"
    }
  }
]
```

- `INTERNET` is a **`system_grant`** permission: declaring it is enough, and it
  is granted at install time. `Index.aboutToAppear` nonetheless calls
  `PermissionUtils.requestPermissions`, which runs `requestPermissionsFromUser`
  on it - no dialog can appear for a system-granted permission, so the branch
  that logs `permissions get fail` is unreachable noise. Delete the helper, or
  keep it for a `user_grant` permission you actually add.
- `reason` points at `$string:app_name`, which is the app's name rather than an
  explanation. `reason` and `usedScene` are only mandatory for `user_grant`
  permissions, so this is cosmetic here - but it is the wrong habit to copy.
- No storage permission is needed: everything is written under
  `context.cacheDir`, the app's own sandbox.

`CMakeLists.txt` links `libohaudio.so`, `libace_napi.z.so` and
`libhilog_ndk.z.so` (plus three `libnative_media_*` libraries the code does not
use), and the module name in `napi_module` is `entry`, which is what makes
`import testNapi from 'libentry.so'` resolve.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The sample ships no endpoint.** `PlayPage` calls `HttpUtils.postHttp('')`
  with an empty URL, so the download path cannot be run as-is; supply a URL that
  returns the SSE hex stream the `DataEntry` / `ResultData` model describes.
- The transcode depends on the third-party `@ohos/mp4parser` package from ohpm,
  not on a system API. Its `ffmpegCmd` runs asynchronously and reports through
  `ICallBack.callBackResult`; the sample resolves on code `0` and rejects with
  no reason on anything else.
- PCM is written into `cacheDir`, which the system may clear. `FileUtils.FileList`
  enumerates `*.pcm` there, but nothing rebuilds the in-memory `audioList` from
  it, so downloads do not survive a restart.
- `AudioRendererInit` allocates with `new char[length + 1]` and frees with
  `delete buf` - the array form `delete[] buf` is what this needs. Not filed as a
  finding, but fix it when adopting the file. `g_filePath`'s hardcoded
  `/data/storage/el2/base/haps/entry/files/oh_test_audio.pcm`, used by the
  low-latency path, is a development leftover in the same file.
- The 喜欢 and 动态 tabs are empty white columns.

## Pitfalls

- **`HW-13-0067`** (B/medium, confirmed): `postHttp` accepts the audio payload
  only when it appears on line index 0 of the response, and resolves an
  unassigned `music` otherwise - the download then fails inside
  `hexStringToArrayBuffer` with an unrelated error. Fix: drop the `i === 0`
  condition and reject when no payload was found.
- **`HW-13-0068`** (B/low, confirmed): `fs.OpenMode.CREATE || fs.OpenMode.READ_WRITE`
  evaluates to `CREATE` alone, and `FileUtil` initialises `fd = 0` so its
  `finally` closes stdin on the error path and still returns the unwritten path.
  Fix: bitwise `|`, guard the close on a real handle, return `undefined` on
  error.
- **`HW-13-0069`** (B/medium, confirmed): `GetRendererState` creates an
  `OH_AudioStreamBuilder` it never uses and never destroys, and ArkTS polls it
  every 50 ms during playback - up to twenty leaked native objects per second.
  Fix: remove the builder creation.
- **`HW-13-0039`** (B/medium, confirmed, systematic across five samples):
  `finally`-block `closeSync` on possibly-undefined handles turns a refusal or a
  failed open into a secondary `TypeError` that masks the original error.
  `HttpUtil.ets:74-82` (`closeSync(fd!)` after an empty catch) and
  `FileUtil.ets:32-43` are this sample's instances. Fix: guard each handle
  before closing.
- **`HW-13-0062`** (B/low, confirmed, systematic across four samples): the final
  partial block leaves stale bytes in the render buffer.
  `napi_init.cpp:62-71` calls `fread(buffer, bufferLen, 1, g_file)`, so a short
  read returns 0 elements and the previous buffer contents are played. Fix:
  `fread(buffer, 1, bufferLen, g_file)` and `memset` the remainder.
- **`HW-13-0003`** (E/low, confirmed, systematic): the documented tree lists
  `cpp/types/libentry/index.d.ets`; the zip ships `Index.d.ts`. Both the case
  and the extension are wrong, and the second misstates the format of a NAPI
  typing file. Fix: regenerate the tree from the zip.

## References

- `huawei_industry_tree/13_media_entertainment/docs/30_music_demo_httpaudiorender.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/music_demo_httpaudiorender-0000002330168664
- `documentation/harmonyos-references/03_system/js-apis-http.md` - `createHttp`, `HttpDataType`, `HttpResponse`, `headersReceive`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-http
- `documentation/harmonyos-guides/05_media/using-ohaudio-for-playback.md` - the builder, the write callback and the renderer lifecycle
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/using-ohaudio-for-playback
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `OpenMode` flag values and `closeSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.INTERNET` and its `system_grant` level
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `documentation/harmonyos-references/06_standard-libraries/napi.md` - `napi_get_value_string_utf8`, module registration
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/napi
- `MEDIA-27` - the audio-visualisation sample that shares the partial-buffer defect (`HW-13-0062`)
- `MEDIA-01` - the audio app architecture page this player's structure follows
