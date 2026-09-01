---
id: PHOTO-11
title: Face detection with a focus matrix - faceDetector rects driving ImageFit.MATRIX pan and zoom
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/11_face_detection.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/face_detection-0000002328775849
sample: huawei_industry_tree/18_photography/downloads/FaceDetection.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.CoreVisionKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [faceDetector, fileIo, hilog, image, matrix4, photoAccessHelper, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0022, HW-18-0092, HW-18-0093, HW-18-0094, HW-18-0095]
status: verified-with-fixes
---

## When to use

Load this card when you need to **find faces in a still image and then show
them** - a per-face thumbnail strip, a "tap a face to zoom to it" viewer, an
auto-crop that centres on a person, a retouching tool that has to know where a
face is before it can work on one.

Two things make this sample worth studying, and they are separable. The first
is the `faceDetector` lifecycle: `init` → `detect` → `release`, on-device, no
permission, no network, no model to bundle. The second, and the more broadly
useful one, is **`ImageFit.MATRIX` as a viewport**. Instead of cropping the
image or nesting scroll containers, the whole image stays in one `Image`
component and a `matrix4` transform decides which part of it is visible. That
gives face-focus, pan, pinch-zoom and thumbnail rendering from a single
mechanism - and the thumbnails in this sample are literally the same component
with a different matrix and gestures disabled.

That last idea generalises past faces. Any "region of interest in a large
image" UI - a map hotspot, an OCR match, a detected object, a QR code - can use
the same rect-to-matrix conversion.

## Feature checklist

- A plus button in the header opens the picker (single image) and loads it into
  the viewer.
- The loaded image is shown scaled to *cover* the 360 vp viewer and centred.
- The image can be dragged and pinch-zoomed, clamped so it never shows empty
  space beside the picture and never exceeds 10× magnification.
- A start button runs face detection; it is enabled only after a fresh image is
  picked, and disables itself once detection has run.
- Detected faces appear as circular thumbnails in a four-column grid.
- Tapping a thumbnail animates nothing but re-frames the main viewer onto that
  face, at 80% of the tightest fit.
- The selected thumbnail grows, gains a blue border and a shadow.
- A toast confirms detection finished.

## Architecture

One `entry` module, seven files, no permissions declared at all.

```
entry/src/main/ets
├── common/Constants.ets           MAX_SCALE = 10.0, SCALE_0_8 = 0.8
├── components/ViewImage.ets       the matrix viewport: main view AND every thumbnail
├── entryability/EntryAbility.ets  full screen, avoid areas -> AppStorage
├── model/TransitInfo.ets          { scale, translationX, translationY }
├── pages/PicPage.ets              @Entry: header, viewer, face grid, start button
└── utils
    ├── FaceDetectionUtil.ets      picker + decode + faceDetector init/detect/release
    └── Matrix4Util.ets            rect -> matrix, offset clamping
```

The documented tree matches the zip exactly. Note there is no
`entrybackupability` here and the document does not claim one - unusual for
this industry, and correct.

**The design decision worth copying** is that `ViewImage` is used at two very
different sizes and roles with one flag. `PicPage` instantiates it once at
360 vp for the main viewer, and again inside a `ForEach` at 64-68 vp for each
detected face:

```typescript
ViewImage({ chooseImage: this.chooseImage, faceData: this.currentFace })
  .width('100%').height(360).borderRadius(16).clip(true);

ViewImage({ chooseImage: this.chooseImage, faceData: face, movable: false })
  .width(this.currentFaceIndex === index ? 68 : 64)
  .borderRadius(34).clip(true);
```

Same component, same pixel map, same matrix maths. The only differences are
the component's own size - which `onComplete` reports back, so the matrix
recomputes itself for a 64 vp box exactly as it did for a 360 vp one - and
`movable: false`, which feeds `.enabled()` and switches the gestures off.
**There is no separate thumbnail pipeline: no cropping, no extra decodes, no
extra pixel maps.** Ten detected faces cost ten matrices, not ten images.

The complement is `@Prop @Watch('onFaceDataReceived') faceData`. The parent
sets `currentFace`, the watch fires, the matrix recomputes. Nothing else in the
main viewer needs to know a selection happened.

## Implementation steps

1. **Pick and decode.** `PhotoViewPicker` with `maxSelectNumber: 1`, then open
   the URI and `createPixelMapSync`. Guard the empty `photoUris` case - the
   sample already returns `''` on the picker's own throw, but a cancel resolves
   normally with an empty array.
2. **Close the file descriptor after decoding** (`HW-18-0022`). `loadImage`
   opens the picked file and returns without ever closing it.
3. **Detect:** `await faceDetector.init()` as the gate, build a
   `VisionInfo { pixelMap }`, `await faceDetector.detect(visionInfo)`, then
   `await faceDetector.release()`. Wrap only `detect` in the try/catch, so a
   detection failure still reaches the release.
4. **Render the image with `objectFit(ImageFit.MATRIX)` and
   `imageMatrix(this.matrixTransit)`.** Set `draggable(false)` so the system's
   own image drag does not fight the pan gesture.
5. **Read the geometry from `onComplete`,** which is the only place that
   reports both the decoded image size and the laid-out component size. Compute
   the initial cover matrix there.
6. **Convert a face rect to a matrix**: scale to the tighter of the two axis
   fits, times 0.8 for breathing room, then translate so the rect's centre
   lands on the component's centre.
7. **Clamp every offset through one function** so the image cannot be dragged
   off its own edges, and reuse it for both pan and pinch.
8. **Copy the matrix at gesture start** and derive from the copy during the
   gesture; commit the scalar state only on `onActionEnd`.
9. **Reset `currentFace` and `currentFaceIndex` before a new detection run**,
   otherwise a stale selection index highlights the wrong thumbnail.

## Verified snippets

All snippets are from `FaceDetection.zip`. Corrected forms are marked.

**The detector lifecycle — `entry/src/main/ets/utils/FaceDetectionUtil.ets`**
(as shipped)

```typescript
import { faceDetector } from '@kit.CoreVisionKit';

static async detectFace(pixelMap: PixelMap): Promise<faceDetector.Face[]> {
  let res: faceDetector.Face[] = [];
  if (await faceDetector.init()) {
    let visionInfo: faceDetector.VisionInfo = {
      pixelMap: pixelMap
    };
    try {
      res = await faceDetector.detect(visionInfo);
    } catch (error) {
      hilog.error(0x0000, 'faceDetector',
        `Face detection failed. Code: ${error.code}, message: ${error.message}`);
    }
    await faceDetector.release();
  }
  return res;
}
```

**The structure is three deliberate decisions.** `init()` returns a boolean and
is used as the gate, not fire-and-forget: on a device or form factor where the
detector is unavailable the function returns an empty array rather than
throwing, and the caller's `this.faceData = data` simply produces no
thumbnails. The try/catch wraps `detect` alone, so `release()` runs on the
failure path too - `faceDetector` is a **process-wide singleton**, not an
instance, so a leaked init would stay leaked for the process lifetime and there
is no second handle to clean it up with. And `res` is initialised to `[]` at the
top, so every exit path returns a real array.

`VisionInfo` takes the `PixelMap` directly. There is no file path variant, so
whatever decodes the image has to hand over a live pixel map - which is why
`loadImage` uses `createPixelMapSync` and why the pixel map is held in page
state rather than re-decoded per detection.

**Loading the picked image — same file** (corrected, see `HW-18-0022`)

```typescript
private static async openPhotoViewPicker(): Promise<string> {
  let photoPicker: photoAccessHelper.PhotoViewPicker = new photoAccessHelper.PhotoViewPicker();
  try {
    return (await photoPicker.select({
      MIMEType: photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE,
      maxSelectNumber: 1
    })).photoUris[0];
  } catch (err) {
    hilog.error(0x0000, 'faceDetection', `Failed to get photo image uri.code: ${err.code}, message: ${err.message}`);
    return '';
  }
}

private static loadImage(name: string): image.PixelMap {
  let imageSource: image.ImageSource | undefined = undefined;
  let fileSource = fileIo.openSync(name, fileIo.OpenMode.READ_ONLY);
  try {
    imageSource = image.createImageSource(fileSource.fd);
    return imageSource.createPixelMapSync();
  } finally {
    imageSource?.release();          // FIX: absent in the sample
    fileIo.closeSync(fileSource.fd); // FIX: absent in the sample - one fd leaked per image
  }
}
```

**`createPixelMapSync` is the reason the corrected form is this simple.**
Because the decode is synchronous, the descriptor is guaranteed live for the
whole decode and the `finally` cleanup cannot run too early - the fd-lifetime
inversion that `HW-18-0025` describes in `PictureSticker` and `ImageFilter`
is structurally impossible here. The only defect is that nothing closes
anything: `HW-18-0022` counts one leaked descriptor per picked image, across
five samples that share the pattern.

Note the two-step `selectImage` → `openPhotoViewPicker` → `loadImage` split
also handles the cancel case that trips six other photography samples: the
picker's own rejection is caught and turned into `''`, and `selectImage` checks
`if (uri)` before decoding. A cancel that *resolves* with an empty array still
yields `photoUris[0] === undefined`, which the same `if (uri)` catches.

**Rect to matrix — `entry/src/main/ets/utils/Matrix4Util.ets`** (as shipped)

```typescript
import { matrix4 } from '@kit.ArkUI';
import { SCALE_0_8 } from '../common/Constants';

static getMatrix4(scale?: number, transX?: number, transY?: number): matrix4.Matrix4Transit {
  return matrix4.identity().scale({ x: scale, y: scale }).translate({ x: transX, y: transY });
}

static getFaceCenterTransit(
  face: faceDetector.Face,
  componentWidth: number,
  componentHeight: number
): TransitInfo {
  let scale = Math.max(componentWidth / face.rect.width, componentHeight / face.rect.height) * SCALE_0_8;
  return {
    scale: scale,
    translationX: (componentWidth - face.rect.width * scale) / 2 - face.rect.left * scale,
    translationY: (componentHeight - face.rect.height * scale) / 2 - face.rect.top * scale
  };
}

static getLimitOffset(imageSize: number, componentSize: number, scale: number, targetOffset: number): number {
  let minOffset = componentSize - imageSize * scale;
  let maxOffset = 0;
  if (minOffset > 0) {
    minOffset /= 2;
    return minOffset;      // image smaller than the box: pin it centred
  }
  if (minOffset <= targetOffset && targetOffset <= maxOffset) {
    return targetOffset;
  } else if (targetOffset < minOffset) {
    return minOffset;
  } else {
    return 0;
  }
}
```

**Three lines carry the whole viewer.** `Math.max` of the two axis ratios is a
*cover* fit, not a contain fit - the face fills the shorter dimension and
overflows the longer one, which is what you want for a face crop and the
opposite of what you want for a document. `SCALE_0_8` then backs it off to 80%,
leaving hair and chin visible instead of a rect-tight crop; it is the one
tunable that decides how the result feels.

The translation is the composition of two moves written as one expression:
`(componentSize - rectSize * scale) / 2` centres the scaled rect in the
component, and `- rect.left * scale` shifts the image so that the rect's own
origin is where the maths assumed it was. Order matters - `getMatrix4` applies
`scale` *then* `translate`, so the translation is in already-scaled units,
which is why every term is multiplied by `scale`.

`getLimitOffset` encodes the clamp as a closed interval `[componentSize -
imageSize*scale, 0]`: offset 0 means the image's left/top edge is flush with
the component's, and the negative bound is where its right/bottom edge is
flush. The `minOffset > 0` branch is the degenerate case where the scaled image
is *smaller* than the box, and it returns half the slack - centring rather than
clamping. Both pan and pinch route through this one function, so there is
exactly one definition of "in bounds" in the project.

**Pinch with a limit — `entry/src/main/ets/components/ViewImage.ets`**
(as shipped)

```typescript
PinchGesture({ fingers: 2 })
  .onActionStart(() => {
    this.matrixPinch = this.matrixTransit.copy();
  })
  .onActionUpdate((event: GestureEvent) => {
    if (this.scaleValue * event.scale > MAX_SCALE) {
      this.matrixTransit = this.matrixPinch.copy().scale({
        x: MAX_SCALE / this.scaleValue,
        y: MAX_SCALE / this.scaleValue,
        centerX: this.getUIContext().vp2px(event.pinchCenterX),
        centerY: this.getUIContext().vp2px(event.pinchCenterY)
      });
      this.scaleTemp = MAX_SCALE;
    } else if (this.scaleValue * event.scale < this.minScale) {
      // ...symmetric clamp at minScale...
    } else {
      this.matrixTransit = this.matrixPinch.copy().scale({
        x: event.scale, y: event.scale,
        centerX: this.getUIContext().vp2px(event.pinchCenterX),
        centerY: this.getUIContext().vp2px(event.pinchCenterY)
      });
      this.scaleTemp = this.scaleValue * event.scale;
    }
  })
  .onActionEnd((event: GestureEvent) => {
    let scale = this.scaleTemp / this.scaleValue;
    this.scaleValue = this.scaleTemp;
    this.offsetX = Matrix4Util.getLimitOffset(this.imageWidth, this.componentWidth, this.scaleValue,
      this.pinchCenterX - (this.pinchCenterX - this.offsetX) * scale);
    // ...same for Y...
    this.matrixTransit = Matrix4Util.getMatrix4(this.scaleValue, this.offsetX, this.offsetY);
  })
```

**`matrixPinch = this.matrixTransit.copy()` on gesture start is the load-bearing
line.** `event.scale` is cumulative from the start of the pinch, so every
update must be applied to the matrix as it was when the fingers went down -
applying it to the live matrix would compound exponentially. `copy()` is
required because `Matrix4Transit` methods mutate and return `this`; without it
`matrixPinch` would be the same object being scaled repeatedly.

The `centerX/centerY` on `scale` are what make the zoom happen under the
fingers rather than at the origin, and they must be converted with `vp2px`:
gesture coordinates arrive in vp while the matrix operates in the image's pixel
space. The whole file is consistent about this - every `event.offsetX`,
`pinchCenterX` and so on is wrapped in `vp2px` before it touches a matrix.

`onActionEnd` recomputes the scalar offsets analytically -
`center - (center - offset) * scale` is the fixed-point formula for a scale
about an arbitrary centre - rather than reading them back out of the matrix,
because `Matrix4Transit` exposes no accessors. Keeping `scaleValue`, `offsetX`
and `offsetY` as the source of truth and the matrix as a derived value is what
lets a face selection and a gesture write to the same viewer without conflict.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions` at all - and the
feature is complete without any. Face detection runs on-device through
`CoreVisionKit`; image access is the picker's temporary grant; nothing is
written back to the album.

This is the correct baseline for this industry, and worth holding up against
`HW-18-0004`: eight sibling samples declare restricted `READ_IMAGEVIDEO` /
`WRITE_IMAGEVIDEO` for flows no more privileged than this one. `FaceDetection`
demonstrates that the empty manifest is sufficient.

`EntryAbility` sets `COLOR_MODE_LIGHT` at `onCreate`, calls
`setWindowLayoutFullScreen(true)` and publishes `topRectHeight` /
`bottomRectHeight` into `AppStorage`, which `PicPage` consumes with
`@StorageProp` and converts through `px2vp` at the point of use. The
`avoidAreaChange` subscription registered in `onWindowStageCreate` is never
released in `onWindowStageDestroy` - the same boilerplate omission seen across
this industry's samples.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `faceDetector` is a process-wide singleton, not an instance. Concurrent
  detections are not isolated, and a `release()` from one caller ends the
  session for all of them. Serialise detection if more than one screen can
  trigger it.
- `init()` returning `false` is a normal outcome on devices without the
  capability, and the sample treats it as "no faces". If the distinction
  matters to your UI, surface it explicitly - here it is indistinguishable from
  a photo with nobody in it.
- Detection runs on the full-resolution pixel map with no downscale, so the
  cost scales with the picked image.
- The geometry all depends on `onComplete` having fired. `onFaceDataReceived`
  called before the image has completed would divide by a zero
  `componentWidth`; the component guards against this by also invoking the
  focus from inside `onComplete` itself.
- The face grid's `ForEach` keys on `JSON.stringify(face)`, so two faces with
  identical rects would collide - unlikely, but it is an object-stringify key.
- The main viewer and the thumbnails share one `@Link chooseImage`, so a new
  pick re-frames everything at once; `faceData` is cleared at the same time to
  keep the grid from showing rects belonging to the previous photo.

## Pitfalls

- **`HW-18-0022`** (B/low, confirmed): `FaceDetectionUtil.loadImage` opens the
  picked file with `fileIo.openSync` and never closes it, leaking one file
  descriptor per image loaded for the process lifetime; heavy use exhausts the
  descriptor table and later opens fail. The same create-without-close appears
  in `Picture`'s `ImageUtils.getBufferByUri`, `VideoClip`'s `VideoClipPage`,
  `CropRect`'s `CropUtil.getResourceFd` and `VideoCropping`'s `AVPlayerDemo`
  (which keeps `// fs.closeSync(file);` commented out). Fix: close the file in a
  `finally` after the pixel map is produced, and release the `ImageSource` with
  it.
- No further defects were found in this sample. The detector lifecycle, the
  matrix maths, the gesture handling and the empty permission manifest are all
  correct as shipped.

## References

- `documentation/harmonyos-references/07_ai/core-vision-face-detector-api.md` - `faceDetector.init`, `detect`, `release`, `VisionInfo`, `Face.rect`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/core-vision-face-detector-api
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-image.md` - `ImageFit.MATRIX`, `imageMatrix`, `onComplete`, `draggable`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-image
- `documentation/harmonyos-references/02_application-framework/js-apis-matrix4.md` - `matrix4.identity`, `scale`, `translate`, `copy`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-matrix4
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-transformation.md` - transform ordering
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-transformation
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pinchgesture.md` - `event.scale`, `pinchCenterX/Y` semantics
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pinchgesture
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagesource.md` - `createPixelMapSync`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagesource
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` - `select`, `PhotoSelectResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `openSync`, `closeSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `PHOTO-10` - the same picker-to-`PixelMap` load, with the fd-lifetime defect this sample avoids
