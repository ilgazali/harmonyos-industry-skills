---
id: MEDIA-37
title: Offscreen video render - decode into a NativeImage, composite through an FBO, present on the XComponent
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/37_opengl_offscreen_render.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/opengl_offscreen_render-0000002553789479
sample: huawei_industry_tree/13_media_entertainment/downloads/opengl-offscreen-render.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.LocalizationKit", "@kit.PerformanceAnalysisKit"]
apis: [hilog, resourceManager, window]
min_api: 20
modules: [entry (entry)]
findings: [HW-13-0083, HW-13-0084, HW-13-0085, HW-13-0012, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when the app must **touch every decoded video frame on the GPU
before it reaches the screen** - a filter, a blur, a watermark, a text or
subtitle overlay, two layers blended together. The straight path (decoder
surface handed directly to the `XComponent`) gives you no place to insert that
work; this pattern inserts one.

The shape is: the decoder renders into an **`OH_NativeImage`**, not into the
window. `OH_NativeImage` hands out an `OHNativeWindow` that acts as the
consumer, so decoded frames land in a queue the app owns. A dedicated render
thread binds the newest frame as a `GL_TEXTURE_EXTERNAL_OES` texture, draws it
(plus whatever else you want) into a **framebuffer object**, and only then
blits the FBO's colour texture to the EGL surface created from the
`XComponent`'s window, ending with `eglSwapBuffers`.

It generalises past video: the same NativeImage-to-external-texture bridge is
how you post-process a camera preview, how you feed a watermarked frame into an
encoder's input surface for transcoding, and how you get any producer that only
speaks `OHNativeWindow` into a shader. This sample is also the only place in
this industry pack that shows the AVCodec demuxer/decoder loop in C++ rather
than through `AVPlayer`.

**Read `HW-13-0083` before shipping any of it.** The teardown path in the
sample detaches its decoder threads and then frees the objects they are still
using.

## Feature checklist

- A home page with a single button that pushes an `OpenGLPlayer`
  `NavDestination` through a `routerMap` entry.
- The player page holds one `XComponent` (`type: SURFACE`, `id: 'OpenGL'`,
  `libraryname: 'player'`) so the native side receives the surface callbacks.
- Pressing 播放 (play) opens `sample.mp4` from `rawfile`, and native code
  demuxes it, creates a decoder for the track's MIME, and starts.
- Decoded frames arrive on an `OH_NativeImage` consumer, not on the window.
- The render thread wakes on NativeVSync **and** on frame-available, and draws
  only when a frame is actually pending.
- Every frame is composited with a text bitmap ("离屏渲染", drawn with the
  Drawing text API) into an FBO, then blitted to the XComponent surface and
  presented with `eglSwapBuffers`.
- Leaving the page calls `stopNative`; destroying the surface stops the player
  and releases the render stack.

## Architecture

One `entry` module. The ArkTS side is 130 lines of navigation; everything real
is in `cpp/`.

```
entry/src/main/cpp
├── main.cpp                             napi module 'player': playNative / stopNative
├── CMakeLists.txt                       one shared lib, links vdec/avdemuxer/native_image/native_vsync
├── capbilities/
│   ├── include/SampleInfo.h             SampleInfo + CodecUserData (the two buffer queues)
│   ├── src/Demuxer.cpp                  OH_AVSource_CreateWithFD, track scan, ReadSample
│   ├── src/VideoDecoder.cpp             OH_VideoDecoder_* create/configure/SetSurface/render
│   └── src/SampleCallback.cpp           codec callbacks -> push into the queues, notify
├── drawing/sample_bitmap.cpp            the text layer, drawn with OH_Drawing_Typography*
├── player/src/Player.cpp                singleton: input thread + output thread + teardown
└── render/
    ├── src/PluginManager.cpp            napi_unwrap of the XComponent, RegisterCallback
    ├── src/PluginRender.cpp             surface created/destroyed callbacks, id -> instance map
    ├── src/OpenGLRender.cpp             the EGL context (display, config, context, surfaces)
    ├── src/ShaderProgram.cpp            compile/link + SetInt / SetMatrix4v helpers
    └── src/OpenGLRenderThread.cpp       the render thread: NativeImage, FBO, textures, draw
entry/src/main/ets
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets   present, but NOT declared in module.json5
└── pages/{Index.ets, OpenGLPlayer.ets}
```

The documented 工程目录 matches the zip file for file. Two leftovers worth
knowing about: `CMakeLists.txt` still calls the project `vulkanSample` and
defines `VK_USE_PLATFORM_OHOS`, and `module.json5` has no `extensionAbilities`
block, so the shipped `EntryBackupAbility.ets` is dead code.

**The design decision worth copying** is that `OpenGLRenderThread` owns *all*
GPU state and every other thread talks to it by posting closures:

```cpp
using RenderTask = std::function<void(OpenGLRender &renderContext)>;
void PostTask(const RenderTask &task);
```

The EGL context, the `OH_NativeImage`, the external texture, the FBO, the two
shader programs and the VAOs are created, used and destroyed on one thread, and
the surface callbacks (which arrive on the UI thread) only enqueue work. That
is what makes `UpdateNativeWindow` safe to call from `OnSurfaceCreatedCB`, and
it is why `Release()` posts its cleanup task *before* clearing `running_` and
joining. Copy that structure; a GL context torn down from the wrong thread is
a class of bug you cannot debug from logs.

**The design decision worth avoiding** is the ownership model around it:
`Player` is a function-local-static singleton, `PluginRender` keeps a static
`unordered_map<std::string, PluginRender*>` of raw pointers, and
`PluginRender::~PluginRender` calls the same static `Release(id_)` that deletes
instances - so the destructor and the map can re-enter each other. Both of the
high/medium native defects on this card grow out of that model.

## Implementation steps

1. **Declare the XComponent with `libraryname`**, not a controller:
   `XComponent({ id: 'OpenGL', type: XComponentType.SURFACE, libraryname: 'player' })`.
   That is what makes the runtime hand the `OH_NativeXComponent` to your napi
   module, which `PluginManager::RenderConfig` unwraps and registers callbacks on.
2. **Create the `OH_NativeImage` on the render thread**, with
   `OH_NativeImage_Create(-1, GL_TEXTURE_EXTERNAL_OES)`, acquire its
   `OHNativeWindow`, and set the frame-available listener before any decoding
   starts.
3. **Give that window to the decoder**, not the XComponent's:
   `OH_VideoDecoder_SetSurface(decoder_, sampleInfo.window)` where
   `sampleInfo.window = render->openGLRenderThread_->GetNativeImageWindow()`.
   `GetNativeImageWindow()` blocks until the render thread has created it.
4. **Create the EGL surface from the XComponent's window** in
   `OnSurfaceCreatedCB` -> `UpdateNativeWindow` -> `PostTask`, after
   `SET_BUFFER_GEOMETRY` with the reported size.
5. **Drive the loop from NativeVSync plus the frame counter.** Re-request the
   frame each pass (`OH_NativeVSync_RequestFrame`) and skip drawing when
   `availableFrameCnt_ <= 0`, so a vsync without a new frame costs nothing.
6. **Use one sentinel for texture ids and guard creation with it**
   (`HW-13-0085`). The sample creates on `== 9999` and resets to `0`, so nothing
   can be recreated after a surface change.
7. **Use the transform you queried**, do not overwrite it with a constant
   (`HW-13-0085`): `GET_TRANSFORM` and `OH_NativeImage_GetTransformMatrix` are
   both called and both discarded by `transformTmp = 5`.
8. **Composite into the FBO first, then blit.** The shipped `DrawImage` calls
   `DrawPreviewImage()` before `DrawFboImage()`, which shows the previous
   frame's FBO content; the document's own snippet has the two the right way
   round.
9. **Tear down by joining, never by detaching** (`HW-13-0083`): clear the
   running flag, notify both condition variables, join both threads, and only
   then destroy the demuxer, the decoder and the `CodecUserData` they wait on.
10. **Delete before nulling** (`HW-13-0084`): `PluginRender::Release` does
    `render = nullptr; delete render;`, which deletes a null pointer and leaks
    the whole render stack. Swap the two statements.
11. **Close the raw file descriptor** (`HW-13-0012`): `getRawFd('sample.mp4')`
    is called on every press of the play button and neither the ArkTS side nor
    the native side ever calls `closeRawFd`.

## Verified snippets

All snippets are from `opengl-offscreen-render.zip`. Corrected forms are marked.

**Teardown - `entry/src/main/cpp/player/src/Player.cpp`** (corrected, see `HW-13-0083`)

```cpp
void Player::ReleaseThread()
{
    // FIX: the sample detaches both threads and resets the pointers.
    if (videoDecInputThread_ && videoDecInputThread_->joinable()) {
        videoDecInputThread_->join();
        videoDecInputThread_.reset();
    }
    if (videoDecOutputThread_ && videoDecOutputThread_->joinable()) {
        videoDecOutputThread_->join();
        videoDecOutputThread_.reset();
    }
}

void Player::Release()
{
    std::lock_guard<std::mutex> lock(mutex_);
    isStarted_ = false;

    if (videoDecContext_ != nullptr) {                 // FIX: wake the waiters before joining
        videoDecContext_->inputCond.notify_all();
        videoDecContext_->outputCond.notify_all();
    }
    ReleaseThread();

    if (demuxer_ != nullptr) {
        demuxer_->Release();
        demuxer_.reset();
    }
    if (videoDecoder_ != nullptr) {
        videoDecoder_->Release();
        videoDecoder_.reset();
    }
    if (videoDecContext_ != nullptr) {
        delete videoDecContext_;
        videoDecContext_ = nullptr;
    }

    doneCond_.notify_all();
}
```

**Detach is not a teardown.** Both worker threads block on
`videoDecContext_->inputCond.wait_for(lock, 5s, ...)` and dereference
`demuxer_` and `videoDecoder_` right after waking. `detach()` only gives up the
handle; the thread keeps running, and the next four statements delete exactly
what it is about to touch. The window is up to five seconds wide - the
condition-variable timeout - and it opens on the ordinary paths: pressing back
out of the page (`aboutToDisappear` -> `stopNative`) and surface destroy. The
notify before the join is not optional either: `isStarted_` alone does not wake
a thread already parked in `wait_for`, so a bare join would stall for the full
timeout.

There is a second, subtler danger in the same file: `VideoDecOutputThread` ends
by calling `StartRelease()` **from inside itself**, so on end-of-stream the
release path runs on the very thread it must join. Replace that self-call with
a completion notification to the owner - the `playDoneCallback` hook already
sits unused in `SampleInfo`.

**The NativeImage bridge - `entry/src/main/cpp/render/src/OpenGLRenderThread.cpp`** (as shipped)

```cpp
bool OpenGLRenderThread::CreateNativeImage()
{
    nativeImage_ = OH_NativeImage_Create(-1, GL_TEXTURE_EXTERNAL_OES);
    if (nativeImage_ == nullptr) {
        return false;
    }
    int ret = 0;
    {
        std::lock_guard<std::mutex> lock(nativeImageSurfaceIdMutex_);
        nativeImageWindow_ = OH_NativeImage_AcquireNativeWindow(nativeImage_);
        ret = OH_NativeImage_GetSurfaceId(nativeImage_, &nativeImageSurfaceId_);
    }
    if (ret != 0) {
        return false;
    }

    nativeImageFrameAvailableListener_.context = this;
    nativeImageFrameAvailableListener_.onFrameAvailable = &OpenGLRenderThread::OnNativeImageFrameAvailable;
    ret = OH_NativeImage_SetOnFrameAvailableListener(nativeImage_, nativeImageFrameAvailableListener_);
    return ret == 0;
}

void OpenGLRenderThread::OnNativeImageFrameAvailable(void *data)
{
    auto renderThread = reinterpret_cast<OpenGLRenderThread *>(data);
    if (renderThread == nullptr) {
        return;
    }
    renderThread->availableFrameCnt_++;
    renderThread->wakeUpCond_.notify_one();
}
```

**Three arguments carry the design.** `-1` as the texture id means "no texture
yet" - the real id is attached later with `OH_NativeImage_AttachContext`, which
is what lets the image be created before any GL object exists.
`GL_TEXTURE_EXTERNAL_OES` is the only correct target for a producer-supplied
buffer (typically NV12); it is why the fragment shader declares
`uniform samplerExternalOES texture` under
`#extension GL_OES_EGL_image_external : require`. And the frame-available
listener turns a push producer into a counter the render loop can poll:
`availableFrameCnt_` is an `std::atomic<int>`, incremented in the codec's
thread, decremented after each draw.

`GetNativeImageWindow()` is the handshake the decoder needs - it waits on the
same condition variable until `nativeImageWindow_` exists, so `StartPlayer` can
be called from the UI thread at any time after the page opens.

**The per-frame update - same file** (corrected, see `HW-13-0085`)

```cpp
void OpenGLRenderThread::DrawImage()
{
    if (nativeImageTexId_ == 0U) {                 // FIX: sample tests == 9999 but resets to 0
        glGenTextures(1, &nativeImageTexId_);
        glBindTexture(GL_TEXTURE_EXTERNAL_OES, nativeImageTexId_);
        glTexParameteri(GL_TEXTURE_EXTERNAL_OES, GL_TEXTURE_WRAP_S, GL_REPEAT);
        glTexParameteri(GL_TEXTURE_EXTERNAL_OES, GL_TEXTURE_WRAP_T, GL_REPEAT);
        glTexParameteri(GL_TEXTURE_EXTERNAL_OES, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
        glTexParameteri(GL_TEXTURE_EXTERNAL_OES, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
        int viewWidth = 0;
        int viewHeight = 0;
        OH_NativeWindow_NativeWindowHandleOpt(nativeWindow_, GET_BUFFER_GEOMETRY, &viewHeight, &viewWidth);
        CreateFrameBufferObj(viewWidth, viewHeight);
    }
    if (eglSurface_ == EGL_NO_SURFACE) {
        return;
    }

    OH_NativeImage_AttachContext(nativeImage_, nativeImageTexId_);
    int32_t ret = OH_NativeImage_UpdateSurfaceImage(nativeImage_);   // newest frame -> the OES texture
    if (ret != 0) {
        return;
    }

    OHNativeWindow *nativeWindow = OH_NativeImage_AcquireNativeWindow(nativeImage_);
    int32_t transformTmp = 0;
    int code = NativeWindowOperation::GET_TRANSFORM;
    OH_NativeWindow_NativeWindowHandleOpt(nativeWindow, code, &transformTmp);

    ret = OH_NativeImage_GetTransformMatrix(nativeImage_, matrix_);
    if (ret != 0) {
        return;
    }

    ComputeTransformByMatrix(transformTmp, matrix_);   // FIX: sample forces transformTmp = 5 (FlipV)
    DrawFboImage();                                    // FIX: sample draws the preview first
    DrawPreviewImage();
}
```

**`AttachContext` then `UpdateSurfaceImage` is the whole bridge.** The first
call binds the (now existing) GL texture id to the image; the second dequeues
the newest buffer the decoder produced and makes it the content of that
texture. Neither copies pixels - the frame stays in the graphics buffer the
decoder wrote.

The transform handling is where the sample gives up. It queries
`GET_TRANSFORM`, then queries the matrix, then throws both away by assigning
`transformTmp = 5`, and `ComputeTransformByMatrix` case `5` overwrites
`matrix_` with a fixed vertical flip. That happens to look right for
`sample.mp4`, whose decoder output is bottom-up; a video carrying a rotation
tag renders sideways. Feed the queried value in, and `case 0` (identity) leaves
the real matrix untouched - which is why the corrected form must *not*
reinitialise `matrix_` afterwards.

Note also that the FBO is sized from the **window**, not from the video, and
the width/height arguments to `GET_BUFFER_GEOMETRY` are passed in the order
`(&viewHeight, &viewWidth)` throughout the file - consistently, so it cancels
out, but do not copy that pairing into new code.

**The composite pass - same file** (as shipped)

```cpp
void OpenGLRenderThread::DrawFboImage(void)
{
    int viewWidth = 0;
    int viewHeight = 0;
    OH_NativeWindow_NativeWindowHandleOpt(nativeWindow_, GET_BUFFER_GEOMETRY, &viewHeight, &viewWidth);

    glViewport(0, 0, viewWidth, viewHeight);
    glClearColor(1.0f, 1.0f, 1.0f, 1.0f);
    glClear(GL_COLOR_BUFFER_BIT);

    glBindFramebuffer(GL_FRAMEBUFFER, fboId_);            // draw into the FBO, not the screen
    fboProgram_->Use();
    fboProgram_->SetInt("texture", 0);
    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_EXTERNAL_OES, nativeImageTexId_);
    fboProgram_->SetMatrix4v("matTransform", matrix_, 16, false);

    CreateTextTexture(viewWidth, viewHeight);             // unit 1: the Drawing text bitmap
    fboProgram_->SetInt("textTexture", 1);
    glActiveTexture(GL_TEXTURE1);
    glBindTexture(GL_TEXTURE_2D, textTexId_);

    glBindVertexArray(fboVertexArrayObject_);
    glEnable(GL_DEPTH_TEST);
    glEnable(GL_BLEND);
    glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, Detail::indices);

    glBindVertexArray(0);
    glBindTexture(GL_TEXTURE_EXTERNAL_OES, 0);
    glBindTexture(GL_TEXTURE_2D, 0);
    glBindFramebuffer(GL_FRAMEBUFFER, 0);
}
```

**Two texture units, one draw call, and the blend done in the shader.** Unit 0
is the external video texture, unit 1 the RGBA bitmap produced by
`SampleBitMap::DrawText`. The fragment shader does the over-operator by hand -
`r = text.r + (1.0 - text.a) * image.r`, and the same for g and b - which is
premultiplied-alpha compositing without relying on `glBlendFunc` state. That is
the slot where a real effect goes: swap `fragmentShader` for
`fragmentShaderBW` (already in the file, a luminance conversion) and the same
pipeline gives you a black-and-white filter. `DrawPreviewImage` is then the
mirror of this: unbind the FBO, sample `fboTexId_` as an ordinary `sampler2D`
with the pass-through shader pair, and `SwapBuffers(eglSurface_)`.

## Permissions & config

**None.** No `requestPermissions` in `module.json5` - the video is a rawfile,
not a gallery asset.

Two config items are load-bearing:

```json5
"pages": "$profile:main_pages",
"routerMap": "$profile:router_map"
```

`router_map.json` maps `OpenGLPlayer` to `src/main/ets/pages/OpenGLPlayer.ets`
with `buildFunction: "OpenGLBuilder"`, which is what makes
`pushPathByName('OpenGLPlayer', null)` work without importing the page. On the
native side, `CMakeLists.txt` must link
`libnative_image.so`, `libnative_vsync.so`, `libnative_window.so` and the
`libnative_media_*` codec libraries; the `add_library(player SHARED ...)` name
is the same string as `libraryname: 'player'` in the XComponent and
`.nm_modname = "player"` in the napi module. All three must agree.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- `deviceTypes` is `["phone"]` only. The decoder is created by MIME from the
  demuxed track, so format support is whatever the device advertises - there is
  no software fallback in the sample.
- Playback is paced by `sleep_until(lastPushTime + frameInterval)` in the
  output thread with `frameInterval = 1000000 / frameRate`. There is no audio
  track and therefore no A/V sync; do not lift this loop into a real player.
- There is no pause, seek or restart: `Player::Init` refuses a second start
  while `demuxer_` exists, so a second play press is a no-op until the page is
  left.
- `VideoDecoder::Configure` returns early on a failed configure without
  destroying the `OH_AVFormat` it created - a leak on the error path only.

## Pitfalls

- **`HW-13-0083`** (D/high, confirmed): `Player::Release` detaches the input and
  output decoder threads and then destroys the demuxer, the decoder and the
  `CodecUserData` those threads are still reading - a native use-after-free on
  every stop or page exit. Fix: signal, notify both condition variables, join,
  then free.
- **`HW-13-0084`** (D/medium, confirmed): `PluginRender::Release` runs
  `render = nullptr; delete render;`, so it deletes a null pointer and the
  destructor never runs - the EGL context, the `OH_NativeImage` and the render
  thread all leak on every surface destroy. Fix: swap the two statements.
- **`HW-13-0085`** (B/medium, confirmed): texture-id sentinels are inconsistent
  (creation tests `== 9999`, destruction resets to `0`), so after any recycle
  the video texture is never recreated and the text overlay disappears, while
  `DestroyFrameBufferObj` happily deletes a never-generated id `9999`. The same
  function then hardcodes `transformTmp = 5` and discards the queried
  transform. Fix: standardise on `0` plus a `glGenTextures` guard, and use the
  transform you asked for.
- **`HW-13-0012`** (B/low, confirmed, systematic): `getRawFd('sample.mp4')` is
  called on each play and `closeRawFd` is never called - here and in six other
  media samples. The descriptor is handed to `OH_AVSource_CreateWithFD` and
  leaks for the process lifetime. Fix: pair every `getRawFd` with `closeRawFd`
  after the consumer is released.

## References

- `documentation/harmonyos-references/05_graphics/capi-native-image-h.md` - `OH_NativeImage_Create`, `AcquireNativeWindow`, `AttachContext`, `UpdateSurfaceImage`, `GetTransformMatrix`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/capi-native-image-h
- `documentation/harmonyos-references/05_graphics/capi-external-window-h.md` - `OH_NativeWindow_NativeWindowHandleOpt`, `SET_BUFFER_GEOMETRY`, `GET_TRANSFORM`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/capi-external-window-h
- `documentation/harmonyos-references/05_graphics/capi-native-vsync-h.md` - `OH_NativeVSync_Create` / `RequestFrame` / `Destroy`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/capi-native-vsync-h
- `documentation/harmonyos-references/06_standard-libraries/opengles.md` - EGL setup, `eglCreateWindowSurface`, `eglSwapBuffers`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/opengles
- `documentation/harmonyos-references/09_standard-libraries/opengl.md` - the GL entry points the render thread calls
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/opengl
- `documentation/harmonyos-references/05_graphics/capi-drawing-text-typography-h.md` - `OH_Drawing_TypographyPaint` and the text style calls in `sample_bitmap.cpp`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/capi-drawing-text-typography-h
- `documentation/harmonyos-guides/05_media/avcodec-kit-intro.md` - the demuxer/decoder model
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/avcodec-kit-intro
- `documentation/harmonyos-guides/05_media/parallel-decoding-nativewindow.md` - decoding onto a NativeWindow surface
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/parallel-decoding-nativewindow
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - `libraryname` and the surface lifecycle callbacks
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `MEDIA-40` - the other native OpenGL ES sample in this pack, offscreen into a Pbuffer instead of a window
