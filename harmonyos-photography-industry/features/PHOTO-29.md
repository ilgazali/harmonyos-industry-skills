---
id: PHOTO-29
title: Gravity-sensor orientation - rotate the camera icons and the saved photo while the screen stays rotation-locked
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/29_camera_twist.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/camera_twist-0000002552826219
sample: huawei_industry_tree/18_photography/downloads/CameraTwist.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CameraKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit", "@kit.SensorServiceKit"]
apis: [abilityAccessCtrl, base, bundleManager, camera, common, curves, display, hilog, image, photoAccessHelper, sensor, window]
permissions: [ohos.permission.CAMERA, ohos.permission.READ_IMAGEVIDEO, ohos.permission.WRITE_IMAGEVIDEO]
min_api: 20
modules: [entry]
findings: [HW-18-0084, HW-18-0085, HW-18-0086, HW-18-0087, HW-18-0004, HW-18-0010, HW-18-0041, HW-18-0090, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when a **camera page is locked to portrait but must still react
to how the device is being held**. A camera UI cannot rotate its layout - the
preview surface is bound to a fixed surface rect - yet the toolbar glyphs
should stay upright in the user's hand, and the photo written to the gallery
must carry the orientation the user was actually shooting at.

The pattern is one gravity-sensor reading turned into a quadrant (0, 90, 180,
270), then spent twice: as the `rotate({ angle })` of every icon, and as
`PhotoCaptureSetting.rotation` on the shutter. Nothing else in the page moves.

It generalises past cameras: any rotation-locked surface with chrome on top -
a scanner viewfinder, a document capture frame, an AR overlay - wants exactly
this. **Note before adopting: the icon angle and the capture angle are not the
same number.** The icons must counter-rotate against the device, the image
must rotate with it, so the two mappings are mirror images. The sample does
this correctly and never says so; the document gets it wrong twice
(`HW-18-0087`).

## Feature checklist

- The camera preview fills a fixed-size `XComponent`; the page never re-lays
  out when the device turns.
- Turning the device 90 degrees turns the five toolbar glyphs, the thumbnail
  and the switch button so they stay upright, animated on a spring curve.
- The tilt is ignored when the device is close to flat (the sensor's z axis
  dominates), so a phone lying on a table does not flip icons at random.
- Pressing the shutter writes a photo whose gallery orientation matches how
  the device was held.
- A black flash overlay plays over the preview on each shutter press.
- Front/back switch flips `mirror` on the capture settings.
- Denying the camera permission shows a single button that opens the system
  permission settings sheet rather than re-asking.

## Architecture

One `entry` module: five UI components, one camera utility, one constants
file.

```
entry/src/main/ets
├── component
│   ├── FlashBlackComponent.ets              black overlay, driven by @Watch on a click counter
│   ├── PictureComponent.ets                 thumbnail + shutter + camera switch; owns a sensor listener
│   ├── StackXComponent.ets                  XComponent preview surface, stacked under the flash overlay
│   ├── ToolsComponents.ets                  the top glyph row; owns a second sensor listener
│   └── TwiceReqPermissionButtonComponent.ets  requestPermissionOnSetting fallback
├── constants/CameraConstants.ets            sizes and the hardcoded 1440x1080 profiles
├── entryability/EntryAbility.ets            full-screen layout, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── pages/Index.ets                          @Entry, the @Provide state hub, permission bootstrap
└── utils
    ├── CameraUtils.ets                      session build/teardown, capture, save; a third sensor listener
    └── Logger.ets
```

The documented tree matches the zip with one naming slip: the document lists
`ToolsComponent.ets`, the zip ships `ToolsComponents.ets` (the struct inside
is `ToolsComponent`).

**The design decision worth avoiding** is how the orientation state is owned.
`onDegree()` and `CalDegree()` are copy-pasted verbatim into three files -
`ToolsComponents.ets`, `PictureComponent.ets` and `CameraUtils.ets` - and each
copy calls `sensor.on(sensor.SensorId.GRAVITY, ...)` itself. Two of them
subscribe in `aboutToAppear`; the third subscribes inside `capture()`, so every
shutter press adds another permanent listener. The project contains no
`sensor.off` at all (`HW-18-0084`). Worse, the three copies do not agree: the
component copies map `[60,120]` to **270** and `[240,300]` to **90**, while
`CameraUtils` maps `[60,120]` to **90**. That divergence is the actual feature
- icons counter-rotate, images rotate with the device - but it is expressed as
duplicated code that looks like a bug, so the first developer who "fixes the
inconsistency" breaks either the icons or the photos.

The shape to copy instead: **one gravity subscription, started once, writing
into `AppStorage`**, plus two pure functions `iconAngle(degree)` and
`captureRotation(degree)` that read from it. Then the inversion is visible,
the listener count is one, and `sensor.off` has one obvious home.

## Implementation steps

1. **Fix the surface first.** `StackXComponent` calls
   `setXComponentSurfaceRect({ surfaceWidth: 1440, surfaceHeight: 1920 })` in
   `onLoad` and takes the surface id there. Nothing about the camera may run
   before that callback (`HW-18-0041` - the sample's `aboutToAppear` fires
   `cameraShooting` with `surfaceId` still `''`).
2. **Request `ohos.permission.CAMERA` and read `authResults[0]`.** The sample
   does check the result here, which is the exception in this industry - six
   sibling samples do not (`HW-18-0030`).
3. **Do not declare `READ_IMAGEVIDEO` / `WRITE_IMAGEVIDEO`.** The save path is
   `saveCameraPhoto()` on a `MediaAssetChangeRequest`, which needs no album
   permission; those two are restricted (ACL) and will fail app review
   (`HW-18-0004`).
4. **Subscribe to `sensor.SensorId.GRAVITY` exactly once** and unsubscribe in
   `aboutToDisappear`. Never subscribe inside a click handler
   (`HW-18-0084`).
5. **Guard the flat-device case** in `CalDegree`: if `(x*x + y*y) * 3 < z*z`
   the device is lying down and the degree is meaningless - return the sentinel
   and keep the previous quadrant.
6. **Snap the degree to a quadrant with dead zones,** not to the nearest 90.
   The four accepted bands are `[330,30]`, `[60,120]`, `[150,210]`, `[240,300]`;
   the 30-degree gaps between them are what stops the icons twitching at the
   diagonals.
7. **Bind the icon angle to `rotate({ angle })`** and add
   `.animation({ curve: curves.springMotion() })` on the thumbnail and switch
   button so the turn is a movement, not a jump.
8. **Read the cached rotation synchronously in `capture()`** - do not start a
   listener there and hope it fires first (`HW-18-0085`).
9. **Await the teardown before rebuilding** on a camera switch: `stop` →
   `close` → `release`, then null the fields (`HW-18-0010`).
10. **Wire the timer UI to `timerShooting`** or delete it; as shipped the
    countdown branch is unreachable (`HW-18-0086`).

## Verified snippets

All snippets are from `CameraTwist.zip`. Corrected forms are marked.

**The icon mapping — `entry/src/main/ets/component/ToolsComponents.ets`** (as shipped)

```typescript
import sensor from '@ohos.sensor';
import base from '@ohos.base';

@Component
struct ToolsComponent {
  @State rotation: number = -1;

  onDegree(callback: base.Callback<number>): void {
    sensor.on(sensor.SensorId.GRAVITY, (data: sensor.GravityResponse) => {
      let degree: number = -1;
      degree = this.CalDegree(data.x, data.y, data.z);
      if (degree >= 0 && (degree <= 30 || degree >= 330)) {
        this.rotation = 0;
      } else if (degree >= 60 && degree <= 120) {   // device turned one way
        this.rotation = 270;                        // icon turns the other
      } else if (degree >= 150 && degree <= 210) {
        this.rotation = 180;
      } else if (degree >= 240 && degree <= 300) {
        this.rotation = 90;
      }
      callback(this.rotation);
    });
  }

  CalDegree(x: number, y: number, z: number): number {
    let degree: number = -1;
    // 3为有效_增量_角度_阈值_系数  (threshold coefficient for a valid tilt)
    if ((x * x + y * y) * 3 < z * z) {
      return degree;
    }
    degree = 90 - (Number)(Math.round(Math.atan2(y, -x) / Math.PI * 180));
    return degree >= 0 ? degree % 360 : degree % 360 + 360;
  }

  build() {
    Row() {
      Image($r('app.media.scope')).height(20).width(20).rotate({ angle: this.rotation });
      Image($r('app.media.camera_filters_fill')).height(20).width(20).rotate({ angle: this.rotation });
      Image($r('app.media.clock_close')).height(20).width(20).rotate({ angle: this.rotation });
      Image($r('app.media.livephoto')).height(20).width(20).rotate({ angle: this.rotation });
      Image($r('app.media.gearshape')).height(20).width(20).rotate({ angle: this.rotation });
    }
    .expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.TOP]);
  }
}
```

**Three lines carry the whole feature.** `(x*x + y*y) * 3 < z*z` is the
flat-device test: gravity pointing mostly down the z axis means the phone is
face-up on a table and any planar angle is noise, so the function returns `-1`
and every band rejects it - the last good quadrant simply stays. The
`atan2(y, -x)` term converts the two planar gravity components into a compass
degree, and `90 -` re-bases it so that upright reads as 0. The four bands with
30-degree gaps between them are the hysteresis: without those gaps a phone
held at 45 degrees would flicker between two icon orientations every sensor
tick.

The initial `@State rotation: number = -1` is deliberate too - `rotate({ angle:
-1 })` is a visually harmless 1-degree tilt that persists until the first
sensor callback lands, which is the honest way to render "orientation unknown".

**Capture rotation, from a single listener — `entry/src/main/ets/utils/CameraUtils.ets`** (corrected, see `HW-18-0084`, `HW-18-0085`)

```typescript
import { sensor } from '@kit.SensorServiceKit';
import { camera } from '@kit.CameraKit';

export class CameraUtils {
  private rotation: number = 0;

  // FIX: called once from the page, not from capture()
  startOrientationTracking(): void {
    sensor.on(sensor.SensorId.GRAVITY, (data: sensor.GravityResponse) => {
      let degree: number = this.CalDegree(data.x, data.y, data.z);
      if (degree >= 0 && (degree <= 30 || degree >= 330)) {
        this.rotation = 0;
      } else if (degree >= 60 && degree <= 120) {   // Use ROTATION_90 when degree range is [60, 120]
        this.rotation = 90;                         // note: the mirror of the icon mapping
      } else if (degree >= 150 && degree <= 210) {
        this.rotation = 180;
      } else if (degree >= 240 && degree <= 300) {
        this.rotation = 270;
      }
    });
  }

  stopOrientationTracking(): void {
    sensor.off(sensor.SensorId.GRAVITY);            // FIX: the project has no sensor.off at all
  }

  capture(isFront: boolean): void {
    let settings: camera.PhotoCaptureSetting = {
      quality: camera.QualityLevel.QUALITY_LEVEL_HIGH,
      rotation: this.rotation,                      // already resolved, not requested here
      mirror: isFront
    };
    if (this.photoOutPut) {
      this.photoOutPut.capture(settings);
    }
  }
}
```

**The shipped `capture()` calls `this.onDegree(callback)` and then builds the
settings object on the very next line.** `sensor.on` is asynchronous: the first
callback has not fired when `this.rotation` is read, so the first photo of a
session is always saved at rotation 0 - the exact failure the feature exists to
prevent (`HW-18-0085`). And because the subscription is inside the shutter
handler, ten photos means ten live listeners for the process lifetime
(`HW-18-0084`). Lifting the listener out of `capture()` fixes both at once:
`this.rotation` becomes a value that is always current, and `capture()` becomes
synchronous and cheap.

Keep the comments naming `ROTATION_90` / `ROTATION_270`: `PhotoCaptureSetting.rotation`
is typed `camera.ImageRotation`, whose members are the numbers 0/90/180/270, so
passing the raw number is correct. The document instead assigns the *strings*
`'ROTATION_90'` and `'ROTATION_270'` and feeds them into both `rotate()` and
`PhotoCaptureSetting` - which does not compile, and inverts the mapping on top
(`HW-18-0087`).

**Surface, then camera — `entry/src/main/ets/component/StackXComponent.ets`** (as shipped)

```typescript
@Component
struct StackXComponent {
  mXComponentController: XComponentController = new XComponentController;
  @State imageSize: image.Size = { width: 1440, height: 1920 };
  @Consume surfaceId: string;
  @Consume notHasPermission: boolean;
  @Consume cameraPosition: number;

  build() {
    Stack() {
      XComponent({
        type: XComponentType.SURFACE,
        controller: this.mXComponentController,
        imageAIOptions: { types: [ImageAnalyzerType.SUBJECT], aiController: this.aiController }
      })
        .onLoad(async () => {
          this.mXComponentController.setXComponentSurfaceRect({
            surfaceWidth: this.imageSize.width,
            surfaceHeight: this.imageSize.height
          });
          this.surfaceId = this.mXComponentController.getXComponentSurfaceId();
          if (!this.notHasPermission) {
            this.cameraUtils.cameraShooting(this.cameraPosition, this.surfaceId, this.getUIContext().getHostContext()!);
          }
        })
        .height(CameraConstants.XCOMPONENT_HEIGHT)
        .hitTestBehavior(HitTestMode.Block);
      FlashBlackComponent();
    };
  }
}
```

**This `onLoad` is the only place that may start the camera.** The surface id
does not exist until the surface is created, so `onLoad` is both the earliest
and the only correct trigger. `Index.aboutToAppear`, however, also calls
`cameraShooting(this.cameraPosition, this.surfaceId, context)` from the
permission `.then` - with `surfaceId` still `''`, because the `XComponent` is
inside the `else` branch that only builds after `notHasPermission` flips. That
doomed init opens the camera input, fails at `createPreviewOutput`, and its
un-awaited `releaseCamera()` then races the real init from `onLoad`
(`HW-18-0041`, `HW-18-0010`). Drive the camera from `onLoad` alone and let the
permission callback do nothing but flip `notHasPermission`.

`hitTestBehavior(HitTestMode.Block)` keeps the surface from swallowing the
gestures of the controls stacked above it, and `imageAIOptions` is what makes
the preview subject-aware without any code of its own.

**Awaited teardown — `entry/src/main/ets/utils/CameraUtils.ets`** (corrected, see `HW-18-0010`)

```typescript
async releaseCamera(): Promise<void> {
  try {
    if (this.photoSession) {
      await this.photoSession.stop();
    }
    if (this.cameraInput) {
      await this.cameraInput.close();
    }
    if (this.previewOutput) {
      await this.previewOutput.release();
      this.previewOutput = undefined;               // FIX: fields left populated in the sample
    }
    if (this.photoSession) {
      await this.photoSession.release();
      this.photoSession = undefined;
    }
    if (this.photoOutPut) {
      this.photoOutPut.off('photoAssetAvailable');  // FIX: the shipped code never detaches it
      await this.photoOutPut.release();
      this.photoOutPut = undefined;
    }
  } catch (error) {
    Logger.error(`releaseCamera failed: ${JSON.stringify(error)}`);
  }
}
```

Every call in the shipped version is fire-and-forget, and `cameraShooting`
opens with a bare `this.releaseCamera();` - no `await` - immediately before
building the next pipeline. Camera switch therefore configures a new session
while the old input is still closing, which is the source of the intermittent
"camera occupied" errors and black previews across this whole industry
(`HW-18-0010`, nine samples). Nulling the fields matters as much as awaiting:
the guards are `if (this.photoSession)`, and a released object is still truthy,
so a second teardown re-releases dead handles.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.CAMERA", "reason": "$string:reason_camera",
    "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.READ_IMAGEVIDEO",  ... },   // remove - see HW-18-0004
  { "name": "ohos.permission.WRITE_IMAGEVIDEO", ... }    // remove - see HW-18-0004
]
```

- Only `CAMERA` is requested at runtime, and only `CAMERA` is needed: photos
  reach the gallery through `MediaAssetChangeRequest.saveCameraPhoto()`, which
  is a permission-free security-control flow.
- `READ_IMAGEVIDEO` / `WRITE_IMAGEVIDEO` are restricted (ACL) permissions that
  an ordinary app cannot ship. They are declared here by inheritance from a
  shared template - the same nine-sample defect as `HW-18-0004`.
- `usedScene` carries no `when`, so the default applies. The camera is
  foreground-only in this sample; add `"when": "inuse"` if you copy the block.
- `SENSOR` needs no declaration: the gravity sensor is not a permissioned
  sensor.
- `deviceTypes` is `phone`, `tablet`, `2in1` - but the surface rect is a
  hardcoded 1440x1920 and the profiles a hardcoded 1440x1080.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- The preview and photo profiles are hardcoded to 1440x1080 and looked up with
  `find()`; on a device that does not expose exactly that profile the result
  is `undefined` and `createPreviewOutput` throws inside an async function no
  caller catches. This is the same shape as `HW-18-0021`, whose evidence names
  four sibling samples.
- The screen must be rotation-locked for the pattern to make sense; on an
  unlocked screen the platform rotates the layout and the icons would
  double-rotate.
- The timer UI (`timerShooting`, `isVisibleTimerSet`) is inert (`HW-18-0086`).
  For a working countdown see `PHOTO-13` (`CaptureTimer`).
- `previewPhoto()` hard-codes the system gallery's bundle and ability names, so
  it only works on a device that ships `com.huawei.hmos.photos`.

## Pitfalls

- **`HW-18-0084`** (B/medium, confirmed): every shutter press registers another
  permanent `GRAVITY` listener, and three components subscribe independently;
  the project contains no `sensor.off`. Fix: one subscription, released in
  `aboutToDisappear`.
- **`HW-18-0085`** (B/medium, confirmed): `capture()` starts the async listener
  and then reads `this.rotation` on the next line, so the first photo of a
  session is saved at rotation 0. Fix: keep rotation current from one
  long-lived listener and read it directly.
- **`HW-18-0086`** (B/low, confirmed): `timerShooting` has an `@Provide` and a
  reader but no writer, so `captureTimer` is always 0 and the countdown branch
  is dead. Fix: wire the timer selection UI, or remove it.
- **`HW-18-0087`** (D/low, confirmed): the document's snippet assigns the
  strings `'ROTATION_90'` / `'ROTATION_270'` and passes them to `rotate()` and
  `PhotoCaptureSetting` - non-compiling, and inverted relative to the shipped
  numeric mapping, which the document never explains. Fix: publish the sample's
  numbers and state that the icon rotation is deliberately the mirror of the
  capture rotation.
- **`HW-18-0004`** (D/medium, confirmed, systematic): nine photography samples
  declare restricted `READ_IMAGEVIDEO` / `WRITE_IMAGEVIDEO` although their code
  deliberately uses the permission-free save path; `29_camera_twist` is one of
  them. Fix: delete both entries from `module.json5`.
- **`HW-18-0010`** (B/medium, confirmed, systematic): `releaseCamera()` awaits
  nothing and nulls nothing, and `cameraShooting` calls it without `await`
  right before rebuilding - `CameraTwist CameraUtils.ets:55+157-173` is one of
  nine instances. Fix: await `stop` → `close` → `release` and reset the fields.
- **`HW-18-0041`** (B/medium, probable, systematic): camera init races the
  `XComponent` surface - `Index.ets:54-62` fires `cameraShooting` from the
  permission callback with `surfaceId === ''`, and `onLoad` fires a second
  init. Fix: gate on `surfaceId !== ''` and drive init from `onLoad` only.

## References

- `documentation/harmonyos-references/03_system/js-apis-sensor.md` - `sensor.on`, `SensorId.GRAVITY`, `GravityResponse`, `sensor.off`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-sensor
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-transformation.md` - the `rotate` attribute
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-transformation
- `documentation/harmonyos-references/04_media/arkts-apis-camera-i.md` - `PhotoCaptureSetting`, `Profile`, `ImageRotation`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-camera-i
- `documentation/harmonyos-references/04_media/arkts-apis-camera.md` - the camera module overview
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-camera
- `documentation/harmonyos-guides/02_media/camera-shooting.md` - the session build order this sample follows
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-shooting
- `documentation/harmonyos-guides/05_media/camera-rotation-angle-adaptation.md` - why the capture angle is not the display angle
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-rotation-angle-adaptation
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `ohos.permission.CAMERA`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - why `READ/WRITE_IMAGEVIDEO` must go
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
- `PHOTO-13` - the timed capture this sample only stubs
- `PHOTO-30` - the same gravity reading used for video, via `videoOutput.getVideoRotation`
