---
id: MEDIA-40
title: Image + text compositing - component snapshots blended by OpenGL ES in a native Pbuffer
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/40_audio-v1_2-ts_64.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio-v1_2-ts_64-0000002444273313
sample: huawei_industry_tree/13_media_entertainment/downloads/ImageMixText.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [fileIo, fileUri, hilog, image, photoAccessHelper, window]
min_api: 17
modules: [entry (entry)]
findings: [HW-13-0015, HW-13-0050, HW-13-0098, HW-13-0104, HW-13-0105, HW-13-0106, HW-13-0107]
status: verified-with-fixes
---

## When to use

Load this card for **"burn the overlay into the picture"** - a caption, a
watermark, a sticker, a signature drawn over a photo and then saved as one
flat image. The insight the sample is built on is that you do not have to
re-implement your overlay for the export path: whatever ArkUI already drew on
screen can be captured as a `PixelMap` with `getComponentSnapshot().getSync(id)`,
and two such snapshots can be blended on the GPU.

So the pipeline is: two overlapping components with the same geometry and the
same `aspectRatio`, one holding the image and one holding the text layer with a
transparent background; snapshot both; hand their raw RGBA buffers to a NAPI
function; composite them in an offscreen EGL **Pbuffer** surface with a
two-sampler fragment shader; read the result back with `glReadPixels` into an
`ArrayBuffer`; rebuild a `PixelMap`; encode to JPEG; save through the
security-control save dialog.

It generalises to any pixel work you would rather not write in ArkTS: colour
grading a photo, alpha-compositing several layers, applying a LUT. The
snapshot-in / ArrayBuffer-out boundary is the reusable part - the shader in
the middle is one `if` statement you will replace.

For the other half of this pack's native OpenGL story - offscreen work on a
*video* stream rather than a still - see `MEDIA-37`.

**A note on where this page lives.** The document is published under
`architecture-guides/audio-v1_2-ts_64-...`, an audio slug, though nothing in it
concerns audio; the repo path `docs/40_audio-v1_2-ts_64.md` preserves that
mismatch. Judge the page by its title (Native侧使用OpenGL ES实现图文合成), not
its URL.

## Feature checklist

- A black editor screen with a `+` button that opens the system photo picker
  (single selection, images only).
- The chosen photo fills the top 80% of the screen at its own aspect ratio.
- A 文字 (text) tool adds an editable caption over the centre of the photo.
- Tapping the caption focuses a `TextInput` at the bottom and shows a dashed
  selection border with delete and rotate handles.
- 取消 (cancel) removes the caption; 保存 (save) composites and saves.
- Saving blends the image layer and the text layer natively, writes a JPEG into
  the cache directory, and hands it to the system save dialog for the gallery.
- All intermediate `PixelMap`s, the `ImagePacker` and every file descriptor are
  released in a `finally`.

## Architecture

One `entry` module: one page, one C++ file.

```
entry/src/main/cpp
├── image_mixed.cpp                  DrawRect: EGL Pbuffer + shaders + glReadPixels (273 lines)
├── CMakeLists.txt                   add_library(entry SHARED); links libEGL / libGLESv3 / napi
└── types/libentry/Index.d.ts        drawRect(w, h, imgBuffer, txtBuffer) => ArrayBuffer | undefined
entry/src/main/ets
├── entryability/EntryAbility.ets    full-screen layout + both avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
└── pages/Index.ets                  @Entry - the editor, the snapshots, the save flow (349 lines)
```

The documented tree matches the zip, except that the document writes the
directory as `entrybackupablility` (two typos); the zip spells it correctly.

**The design decision worth copying** is that the two layers are two sibling
components in one `Stack`, given the **same width and the same
`aspectRatio(this.radio)`**, and identified only by `.id('img')` and
`.id('txt')`:

```typescript
Image(this.pixelmap).id('img').width('100%').aspectRatio(this.radio)
Column() { /* the caption and its handles */ }
  .id('txt').width('100%').aspectRatio(this.radio).backgroundColor(Color.Transparent)
```

Because both boxes are laid out identically, their snapshots come out the same
size, and the shader can sample both with one set of texture coordinates - no
offset maths, no scaling, no knowledge in native code of where the caption sits.
The transparent background on the text layer is what makes it work: the shader's
entire compositing rule is "if the text pixel is fully transparent, take the
image pixel". Move the caption, rotate it, add ten more - the native side does
not change.

**The trap to know about**: the geometry contract is implicit. The native
function is told only the *image* layer's size, and blindly assumes the text
buffer has the same dimensions. Nothing checks it -
`napi_get_arraybuffer_info` is called twice into the same `byteLength`, and the
second value is never compared against `width * height * 4`. If the two
components ever diverge in size, the shader samples past the end of a buffer.

## Implementation steps

1. **Pick with `PhotoViewPicker`**, `MIMEType = IMAGE_TYPE` and
   `maxSelectNumber = 1`. The picker returns a URI you may open once; no
   gallery read permission is required for this flow.
2. **Decode with `desiredPixelFormat: RGBA_8888`** - the shader uploads with
   `GL_RGBA / GL_UNSIGNED_BYTE`, so any other decode format produces swapped
   channels.
3. **Close the picked descriptor only after the decode has finished**
   (`HW-13-0015`). `createPixelMap` is asynchronous; the sample's `finally`
   closes the fd while it is still reading.
4. **Give both layers the same box** - same width, same `aspectRatio`, text
   layer transparent - and give each an `id`.
5. **Snapshot with `getComponentSnapshot().getSync(id)`** from the `UIContext`,
   then `readPixelsToBuffer` into `ArrayBuffer`s sized by
   `getPixelBytesNumber()`.
6. **Do the compositing in one draw call** with two texture units and a
   fragment shader that picks per pixel; render into an EGL Pbuffer surface, so
   no window is involved.
7. **Read back and return an `ArrayBuffer`** allocated by
   `napi_create_arraybuffer`, then rebuild with `image.createPixelMapSync`.
8. **Flip vertically after the read-back.** `glReadPixels` returns rows
   bottom-up; `mixPixelmap.flip(false, true)` is what puts the image the right
   way up.
9. **Open the work file with `TRUNC`** (`HW-13-0050`), otherwise a smaller JPEG
   leaves the previous export's trailing bytes in place.
10. **Save through `showAssetsCreationDialog`** and copy into the descriptor it
    returns - the sanctioned way to write to the gallery without holding a
    write permission.
11. **Release everything in a `finally`**: both snapshots, the mixed
    `PixelMap`, the `ImagePacker` and the temp file.

## Verified snippets

All snippets are from `ImageMixText.zip`. Corrected forms are marked.

**Picking and decoding - `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-13-0015`)

```typescript
let file: fileIo.File | undefined;
let imageSource: image.ImageSource | undefined;
try {
  file = fileIo.openSync(uris[0], fileIo.OpenMode.READ_ONLY);
  imageSource = image.createImageSource(file.fd);
  let decodingOpts: image.DecodingOptions = {
    editable: true,
    desiredPixelFormat: image.PixelMapFormat.RGBA_8888,
  };

  // FIX: the sample fires this promise and closes the fd in its own finally,
  // while createPixelMap is still decoding from it.
  const pixelMap: image.PixelMap = await imageSource.createPixelMap(decodingOpts);
  if (this.pixelmap) {
    await this.pixelmap.release();
  }
  this.pixelmap = pixelMap;
  let imageInfo: image.ImageInfo = this.pixelmap.getImageInfoSync();
  this.radio = imageInfo.size.width / imageInfo.size.height;
} catch (error) {
  console.error(`failed to read image data, ${JSON.stringify(error)}`);
} finally {
  imageSource?.release();
  if (file) {
    fileIo.closeSync(file);          // now genuinely after the decode
  }
}
```

**The bug is in the shape, not in any single line.** The shipped code writes
`imageSource.createPixelMap(...).then(...).catch(...).finally(...)` without
awaiting it, inside a `try/finally` whose `finally` runs `fileIo.closeSync(file)`.
The synchronous body finishes first, so the descriptor the decoder is reading
from is closed underneath it. It usually works - small images decode before the
microtask queue drains - and fails on large photos, on slow storage, under
memory pressure. `HW-13-0015` finds the same anti-pattern in five samples
across this industry; awaiting the consumer is the whole fix.

`editable: true` is required because the composited result is later flipped in
place; `desiredPixelFormat: RGBA_8888` is required because the native side
uploads the bytes as `GL_RGBA`.

**Snapshot, composite, encode - same file** (corrected, see `HW-13-0050`)

```typescript
async mixAndSaveImage(): Promise<void> {
  let imgPixelmap: image.PixelMap | undefined;
  let txtPixelmap: image.PixelMap | undefined;
  let mixPixelmap: image.PixelMap | undefined;
  let imagePacker: image.ImagePacker | undefined;
  let tempFile: fileIo.File | undefined;
  try {
    imgPixelmap = this.getUIContext().getComponentSnapshot().getSync('img');
    txtPixelmap = this.getUIContext().getComponentSnapshot().getSync('txt');
    let imgSize = (await imgPixelmap.getImageInfo()).size;

    let imgBuffer: ArrayBuffer = new ArrayBuffer(imgPixelmap.getPixelBytesNumber());
    await imgPixelmap.readPixelsToBuffer(imgBuffer);
    let txtBuffer: ArrayBuffer = new ArrayBuffer(txtPixelmap.getPixelBytesNumber());
    await txtPixelmap.readPixelsToBuffer(txtBuffer);

    let resultBuf = testNapi.drawRect(imgSize.width, imgSize.height, imgBuffer, txtBuffer);
    if (resultBuf === undefined) {
      return;
    }

    let options: image.InitializationOptions = {
      editable: true,
      srcPixelFormat: image.PixelMapFormat.RGBA_8888,
      size: { width: imgSize.width, height: imgSize.height },
    };
    mixPixelmap = image.createPixelMapSync(resultBuf, options);
    await mixPixelmap.flip(false, true);           // glReadPixels returns rows bottom-up

    let context: Context = this.getUIContext().getHostContext() as Context;
    let filePath: string = context.cacheDir + '/image_temp.jpg';
    tempFile = fileIo.openSync(filePath,
      fileIo.OpenMode.CREATE | fileIo.OpenMode.TRUNC | fileIo.OpenMode.WRITE_ONLY);  // FIX: no TRUNC
    imagePacker = image.createImagePacker();
    await imagePacker.packToFile(mixPixelmap, tempFile.fd, { format: 'image/jpeg', quality: 100 });

    await this.saveImage();                        // FIX: the sample does not await it
  } catch (error) {
    console.error(`Failed to get image pixelmap, ${JSON.stringify(error)}`);
  } finally {
    imgPixelmap?.release();
    txtPixelmap?.release();
    mixPixelmap?.release();
    imagePacker?.release();
    if (tempFile) {
      fileIo.closeSync(tempFile);
    }
  }
}
```

**`getSync` is the reason this whole approach is practical.** The asynchronous
`get(id)` waits for the next frame; `getSync` captures the component as it
stands, so both layers are captured in the same state with no risk of the
caption moving between the two calls. Note that snapshotting captures what is
*rendered* - the dashed selection border and the delete/rotate handles would be
burned in too if they were visible, which is why the handles carry
`.visibility(this.inputText ? Visible : Hidden)`.

**`TRUNC` is the finding.** `image_temp.jpg` is a fixed path in `cacheDir`,
reused for every export. Opened with `CREATE | WRITE_ONLY` and no truncation,
the writer starts at offset 0 and stops when the new JPEG ends - so if the
second export is smaller than the first, the bytes of the first survive past
the new EOI marker. A JPEG decoder normally stops at EOI, so the saved photo
looks right while carrying the tail of a previous image; that is a privacy
problem as much as a correctness one. `HW-13-0050` finds the same missing
`TRUNC` in two more samples in this pack.

The un-awaited `saveImage()` is the third rough edge: the `finally` closes
`tempFile` while the save is in flight. It survives only because `saveImage`
re-opens the file by URI rather than reusing the descriptor - but any failure
in it is unobserved.

**The native composite - `entry/src/main/cpp/image_mixed.cpp`** (as shipped)

```cpp
// EGL config asks for a Pbuffer, not a window surface
EGLint attribs[] = {
    EGL_SURFACE_TYPE, EGL_PBUFFER_BIT,
    EGL_RENDERABLE_TYPE, EGL_OPENGL_ES3_BIT_KHR,
    EGL_RED_SIZE, 8, EGL_GREEN_SIZE, 8, EGL_BLUE_SIZE, 8, EGL_ALPHA_SIZE, 8,
    EGL_NONE,
};
eglChooseConfig(display, attribs, &config, 1, &numConfigs);
EGLint attribList[] = {EGL_WIDTH, width, EGL_HEIGHT, height, EGL_TEXTURE_FORMAT, EGL_NO_TEXTURE, EGL_NONE};
surface = eglCreatePbufferSurface(display, config, attribList);
eglMakeCurrent(display, surface, surface, context);

const char *fragmentShaderSource = R"(#version 300 es
    precision mediump float;
    out vec4 fragColor;
    in vec2 texCoord;
    vec4 color1;
    vec4 color2;
    uniform sampler2D texture1;      // the photo
    uniform sampler2D texture2;      // the caption layer
    void main() {
        color1 = texture(texture1, texCoord);
        color2 = texture(texture2, texCoord);
        if (color2.a == 0.0)  {
            fragColor = color1;
        } else {
            fragColor = color2;
        }
    }
)";

glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, width, height, 0, GL_RGBA, GL_UNSIGNED_BYTE, imgData);
// ... same for txtData into textureId[1] ...
glDrawArrays(GL_TRIANGLES, 0, 6);

std::unique_ptr<uint8_t[]> pixels = std::make_unique<uint8_t[]>(width * height * 4);
glReadPixels(0, 0, width, height, GL_RGBA, GL_UNSIGNED_BYTE, (void *)pixels.get());

uint8_t *pixelmapData = nullptr;
napi_value result = nullptr;
napi_create_arraybuffer(env, width * height * 4, (void **)&pixelmapData, &result);
for (int i = 0; i < width * height * 4; ++i) {
    pixelmapData[i] = pixels[i];
}
return result;
```

**`EGL_PBUFFER_BIT` is what makes this offscreen.** There is no window, no
`OHNativeWindow`, no XComponent - the Pbuffer is a plain off-screen colour
buffer sized to the image, and `glViewport(0, 0, width, height)` makes one
full-screen quad cover it exactly. Two triangles, five floats per vertex (three
position, two texture coordinate), and the fragment shader's binary choice on
`color2.a` is the whole "compositing engine". Replace those four lines with a
real over-operator (`fragColor = color2 + (1.0 - color2.a) * color1`) if the
caption ever has soft edges or partial transparency - the current version
hard-cuts anti-aliased glyph edges.

Two things in this function are worth *not* copying. `std::this_thread::sleep_for(50ms)`
before the read-back is a guess standing in for a real sync point:
`glFinish()` (or a fence) is the correct way to know the draw has landed, and
it is both faster and reliable. And the copy loop moves `width * height * 4`
bytes one at a time when `glReadPixels` could have been pointed straight at the
NAPI-allocated buffer, or the loop replaced by a `memcpy` - on a 12 MP photo
that is 48 million iterations of avoidable work on the UI thread, since the
napi function is synchronous and has no worker.

**Saving to the gallery - `entry/src/main/ets/pages/Index.ets`** (as shipped)

```typescript
let phAccessHelper = photoAccessHelper.getPhotoAccessHelper(context);
let photoCreationConfigs: Array<photoAccessHelper.PhotoCreationConfig> = [{
  title: 'mixed',
  fileNameExtension: 'jpg',
  photoType: photoAccessHelper.PhotoType.IMAGE,
  subtype: photoAccessHelper.PhotoSubtype.DEFAULT,
}];
let srcFileUris: Array<string> = [fileUri.getUriFromPath(filePath)];
let desFileUris: Array<string> = await phAccessHelper.showAssetsCreationDialog(srcFileUris, photoCreationConfigs);
srcFile = fileIo.openSync(srcFileUris[0], fileIo.OpenMode.READ_ONLY);
desFile = fileIo.openSync(desFileUris[0], fileIo.OpenMode.WRITE_ONLY);
fileIo.copyFileSync(srcFile.fd, desFile.fd);
```

**No write permission anywhere.** `showAssetsCreationDialog` is a
security-control component flow: the user confirms the save, and the system
hands back URIs the app may write to for that one operation. That is the
current sanctioned way to put a file into the gallery, and it is why
`module.json5` has an empty permission list. The app supplies a source URI and
a `PhotoCreationConfig` (title and extension), and does the copy itself.

Note that the dialog's result array is indexed without a length check - a user
who dismisses the dialog takes the `catch` branch on `desFileUris[0]` being
`undefined`, which is survivable but produces a misleading log.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. Both media
interactions - `PhotoViewPicker` for reading and `showAssetsCreationDialog` for
writing - are permission-free by design.

`EntryAbility` does the immersive setup: `setWindowLayoutFullScreen(true)`,
then both avoid areas into `AppStorage`:

```typescript
AppStorage.setOrCreate('TopRectHeight', uiContext.px2vp(avoidArea.topRect.height));
AppStorage.setOrCreate('BottomRectHeight', uiContext.px2vp(avoidArea.bottomRect.height));
```

The page reads only the bottom one (`@StorageProp('BottomRectHeight')`) and
adds it to the button row's padding. Both are read once at load and never
updated on a fold, rotation or window resize - there is no `avoidAreaChange`
subscription.

`CMakeLists.txt` names the library `entry`, which must match the
`.nm_modname = "entry"` in the napi module and the `import testNapi from 'libentry.so'`
in the page; the typed surface lives in `types/libentry/Index.d.ts`.

## Constraints

- The document claims API Version 20 Release or later, but the shipped
  `build-profile.json5` sets `compatibleSdkVersion: "5.0.5(17)"` with
  `targetSdkVersion: "6.0.0(20)"`, so the sample actually builds against
  API 17. `showAssetsCreationDialog` needs API 12 or later.
- `deviceTypes` is `["phone"]`; `nativeCompiler` is BiSheng.
- `drawRect` is a **synchronous** napi call: the EGL setup, the draw, the
  read-back and the byte copy all run on the UI thread. On a large photo that
  is a visible freeze; move it to a worker or an async napi call for anything
  real.
- The composite is done at the **snapshot's** resolution, which is the
  component's on-screen size, not the original photo's. Saving therefore
  downsamples a high-resolution picture to roughly screen width.
- The delete and rotate handles are decorative images with no gesture attached;
  the caption cannot be moved, resized or rotated, and there is only ever one.
- `napi_get_arraybuffer_info` writes both buffers' lengths into the same
  `byteLength` and neither is validated against `width * height * 4`.
- EGL objects are cleaned up only on the success path - every early `return`
  after `eglInitialize` leaks the display, the surface or the context.

## Pitfalls

- **`HW-13-0015`** (B/medium, probable, systematic): the picked file descriptor
  is closed in a `finally` while `imageSource.createPixelMap`'s promise is
  still decoding from it. Intermittent decode failures that depend on image
  size and storage speed. The same pattern appears in five samples in this
  industry. Fix: await the decode (or close in its completion path) before
  closing the fd.
- **`HW-13-0050`** (B/medium, probable, systematic): `image_temp.jpg` is reused
  across exports and opened `CREATE | WRITE_ONLY` without `TRUNC`, so a smaller
  new JPEG leaves the previous export's trailing bytes after its EOI marker -
  the saved image carries old-image data. Two other samples in this pack reuse
  work files the same way. Fix: add `OpenMode.TRUNC` (and clear the work
  directory per run).

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-arkui-componentsnapshot.md` - `getComponentSnapshot`, `get` vs `getSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-arkui-componentsnapshot
- `documentation/harmonyos-references/04_media/arkts-apis-image-pixelmap.md` - `readPixelsToBuffer`, `getPixelBytesNumber`, `createPixelMapSync`, `flip`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-pixelmap
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagesource.md` - `createImageSource`, `DecodingOptions`, `desiredPixelFormat`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagesource
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagepacker.md` - `packToFile` and `PackingOption`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagepacker
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` - `PhotoSelectOptions`, `PhotoViewMIMETypes`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-guides/05_media/photoaccesshelper-savebutton.md` - the permission-free save-to-gallery flow behind `showAssetsCreationDialog`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-savebutton
- `documentation/harmonyos-references/06_standard-libraries/opengles.md` - EGL Pbuffer surfaces, shader compilation, `glReadPixels`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/opengles
- `documentation/harmonyos-guides/03_application-framework/arraybuffer-object.md` - returning an `ArrayBuffer` across the NAPI boundary
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arraybuffer-object
- `MEDIA-37` - the same OpenGL ES offscreen idea applied to decoded video frames
