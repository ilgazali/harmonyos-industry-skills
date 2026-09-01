---
id: UTIL-38
title: Geotagged custom camera - attach a location fix to PhotoCaptureSetting so the saved asset carries GPS EXIF
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/38_convenient-life-v1_2-ts_133.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/convenient-life-v1_2-ts_133-0000002429890325
sample: huawei_industry_tree/15_utilities/downloads/LocationData.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkGraphics2D", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CameraKit", "@kit.CoreFileKit", "@kit.LocationKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit", "@kit.PreviewKit"]
apis: [abilityAccessCtrl, camera, colorSpaceManager, common, curves, dataSharePredicates, display, filePreview, geoLocationManager, hilog, photoAccessHelper, window]
permissions: [ohos.permission.CAMERA, ohos.permission.WRITE_IMAGEVIDEO, ohos.permission.READ_IMAGEVIDEO, ohos.permission.APPROXIMATELY_LOCATION, ohos.permission.LOCATION]
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0081, HW-15-0078, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card when you are building a **custom camera that must record where
the shot was taken** - a field-survey app, an insurance claim capture, a
delivery proof-of-arrival, a travel diary. The pattern is small: hold a
location fix in module state, and pass it as `PhotoCaptureSetting.location` on
every `photoOutput.capture()` call. The camera pipeline writes the GPS EXIF into
the file it hands to the media library; you never touch the file yourself.

It is worth reading even if you only want the camera half. This sample is one
of the shortest complete `NORMAL_PHOTO` sessions in the industry tree - input,
preview output on an `XComponent` surface, photo output, session, commit, start
- and the save path uses the modern `photoAssetAvailable` +
`saveCameraPhoto()` route rather than decoding buffers by hand.

**Two warnings before you copy it.** The location fix is asynchronous and
nothing gates capture on it (`HW-15-0081`), so the sample's own headline
feature - the geotag - is `undefined` for the first few seconds of every launch
and forever if the user refuses location. And the permission request ignores the
user's answer entirely (`HW-15-0078`), so a refused camera permission still
starts the pipeline.

The document is also **published under the wrong slug**: the page lives at
`convenient-life-v1_2-ts_133` and is mirrored here as
`38_convenient-life-v1_2-ts_133.md`, but its content, its sample and its
placement in the tree are all utilities/camera. Do not search for it by the
convenient-life name; the topic slug is simply wrong.

## Feature checklist

- On page appear, request camera, both location permissions and read/write
  image-video, then start the preview.
- Live preview fills the top of the screen on an `XComponent` surface locked to
  a 632:360 aspect ratio.
- A round shutter button captures a photo at `QUALITY_LEVEL_HIGH` in the
  DISPLAY_P3 colour space.
- The captured photo is written to the system gallery via `saveCameraPhoto()`.
- Its thumbnail appears in the round preview chip at the bottom left, animated
  in with `curves.springMotion()`.
- Tapping the chip opens the photo in the system Photos app.
- A switch button flips between the two cameras and rebuilds the session.
- The saved photo carries the capture location in its EXIF.

## Architecture

One `entry` module, one page, one util and a constants class.

```
entry/src/main/ets
├── constants/CameraConstants.ets   sizes, the 632/360 surface ratio, '100%'
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── pages/Index.ets                 @Entry: XComponent preview, shutter, switch, thumbnail chip
└── utils/CameraShooter.ets         module-level camera state + cameraShooting/capture/
                                    releaseCamera/getLocation/setPhotoOutputCb/previewPhoto
```

The documented tree matches the zip.

**The design decision worth avoiding** is `CameraShooter.ets` holding its whole
state in module-level `let` bindings - `previewOutput`, `cameraInput`,
`photoSession`, `photoOutPut`, `currentContext`, and the three coordinate
numbers `lat`, `lon`, `alt`. Module scope is a singleton, which is defensible
for a device there is only one of, but it costs the sample three separate
defects. `lat`/`lon`/`alt` are declared without initialisers, so before the
first fix they are `undefined` and `camera.Location` is built out of three
holes. Nothing can observe the state, so the UI cannot disable the shutter until
a fix arrives. And `releaseCamera` and `cameraShooting` both mutate the same
four handles with no lock, so the camera-switch path tears down the old session
while the new one is already opening.

Keep the singleton if you like, but wrap it: a small class with `isReady()`, an
awaited `release()`, and the fix stored as one nullable `camera.Location` object
rather than three loose numbers, fixes all three at once.

## Implementation steps

1. **Declare the five permissions** in `module.json5` with `reason` and
   `usedScene`. Note the sample sets `when: "always"` on both location
   permissions although the camera only needs them in the foreground - `"inuse"`
   is the correct value for this scenario.
2. **Request them and check the answer.** `requestPermissionsFromUser` resolves
   even on refusal; branch on `data.authResults` before starting the pipeline
   (`HW-15-0078`).
3. **Start the camera when the surface exists, not on a timer.** The sample
   waits 200 ms and hopes `surfaceId` has been assigned by `XComponent.onLoad`
   (`HW-15-0081`); drive it from `onLoad` instead.
4. **Await the teardown before the re-init** on a camera switch - `releaseCamera`
   is `async` and its result is dropped (`HW-15-0081`).
5. **Pick the preview profile by exact size, then match the photo profile to
   it.** The sample searches `previewProfiles` for 1920x1080 and then finds the
   photo profile with the same width and height, which keeps preview and
   capture on one aspect ratio.
6. **Configure inside `beginConfig()`/`commitConfig()`**: add the input, both
   outputs and the colour space, then `await commitConfig()` and
   `await start()`.
7. **Register `photoAssetAvailable` before the session starts** so no capture is
   missed, and save with `MediaAssetChangeRequest.saveCameraPhoto()` +
   `applyChanges`.
8. **Gate `capture()` on a real fix** and pass the location through
   `PhotoCaptureSetting` (`HW-15-0081`). Also set `isFront` when switching, or
   `mirror` is permanently `false`.

## Verified snippets

All snippets are from `LocationData.zip`. Corrected forms are marked.

**Permissions and startup — `entry/src/main/ets/pages/Index.ets`**
(corrected, see `HW-15-0078`, `HW-15-0081`)

```typescript
permissions: Array<Permissions> = [
  'ohos.permission.CAMERA',
  'ohos.permission.APPROXIMATELY_LOCATION',
  'ohos.permission.LOCATION',
  'ohos.permission.READ_IMAGEVIDEO',
  'ohos.permission.WRITE_IMAGEVIDEO'
];

async aboutToAppear() {
  try {
    const data = await abilityAccessCtrl.createAtManager()
      .requestPermissionsFromUser(this.context, this.permissions);
    // FIX: the sample ignores `data` entirely and starts the camera regardless
    if (data.authResults[0] !== 0) {
      hilog.error(DOMAIN, TAG, 'Camera permission denied');
      return;
    }
    this.hasPermission = true;      // FIX: new flag, consumed by XComponent.onLoad
  } catch (error) {
    hilog.error(DOMAIN, TAG, `The updatePreview call failed. error: ${JSON.stringify(error)}`);
  }
}

// ...
XComponent({ type: XComponentType.SURFACE, controller: this.mXComponentController })
  .onLoad(async () => {
    this.mXComponentController.setXComponentSurfaceRotation({ lock: true });
    this.mXComponentController.setXComponentSurfaceRect({
      surfaceWidth: displayWidth,
      surfaceHeight: displayWidth * CameraConstants.SURFACE_RATIO
    });
    surfaceId = this.mXComponentController.getXComponentSurfaceId();
    if (this.hasPermission) {
      await cameraShooting(cameraPosition, surfaceId, this.context);   // FIX: was a 200 ms setTimeout
    }
  })
```

**The 200 ms timer in the shipped code is a race, not a delay.** `surfaceId` is
a module-level `let` assigned inside `XComponent.onLoad`; `aboutToAppear` runs
before the first layout pass, so the sample schedules the camera start 200 ms
later and hopes `onLoad` has fired by then. On a cold start with a slow first
frame it has not, and `createPreviewOutput` is handed an empty string. The
surface's own `onLoad` callback is the event you actually want - it fires
exactly once the surface id is valid.

Note also that `requestPermissionsFromUser`'s `.then` in the sample has no
`.catch`, so a thrown request rejects unhandled: the `try/catch` around it only
catches a synchronous throw, not the promise.

**Reading the fix and spending it — `entry/src/main/ets/utils/CameraShooter.ets`**
(corrected, see `HW-15-0081`)

```typescript
let currentLocation: camera.Location | undefined = undefined;   // FIX: was three loose `let lat/lon/alt: number`

export function getLocation(): void {
  let requestInfo: geoLocationManager.CurrentLocationRequest = {
    'priority': 0x203,
    'scenario': geoLocationManager.LocationRequestScenario.DAILY_LIFE_SERVICE,
    'maxAccuracy': 1000,
    'timeoutMs': 5000
  };
  if (!geoLocationManager.isLocationEnabled()) {
    hilog.info(DOMAIN, TAG, 'Location not enabled!');
    return;                                        // FIX: the sample logs and continues anyway
  }
  try {
    geoLocationManager.getCurrentLocation(requestInfo, (err, location) => {
      if (err) {
        hilog.error(DOMAIN, TAG, `getCurrentLocation failed: ${JSON.stringify(err)}`);
        return;
      }
      currentLocation = {
        latitude: location.latitude,
        longitude: location.longitude,
        altitude: location.altitude
      };
    });
  } catch (err) {
    hilog.error(DOMAIN, TAG, `getCurrentLocation error: ${JSON.stringify(err)}}`);
  }
}

export async function capture(isFront: boolean) {
  try {
    let settings: camera.PhotoCaptureSetting = {
      location: currentLocation,          // FIX: sample builds {latitude: lat, ...} from undefined vars
      quality: camera.QualityLevel.QUALITY_LEVEL_HIGH,
      rotation: camera.ImageRotation.ROTATION_0,
      mirror: isFront
    };
    photoOutPut.capture(settings);
  } catch (error) {
    hilog.error(DOMAIN, TAG, `The capture call failed. error: ${error.code}`);
  }
}
```

**`location` is optional on `PhotoCaptureSetting`, and that is the point.**
Omitting the field produces a photo with no GPS EXIF, which is the honest
outcome when there is no fix. The shipped code instead constructs
`{latitude: undefined, longitude: undefined, altitude: undefined}` and hands it
to the camera service - a populated-looking object full of holes, on the very
API call the sample exists to demonstrate. Keeping the fix as one nullable
`camera.Location` makes "no fix yet" representable and lets the UI grey out the
shutter until it arrives.

The three request options are worth naming. `priority` is written as the raw
literal `0x203` rather than a member of `geoLocationManager.LocationRequestPriority`
- check the enum in the reference before copying it, and use the named constant.
The intent is the fast-first-fix priority, which is the right choice for a
camera: a rough fix now beats a precise fix after the moment has passed. The
`scenario` is `DAILY_LIFE_SERVICE`, the low-power everyday profile.
`maxAccuracy: 1000` accepts a
kilometre of error, which is fine for "which city was this". `timeoutMs: 5000`
bounds the wait; note that `getCurrentLocation` is a **one-shot** request, so
the fix is taken once per `cameraShooting` call and never refreshed while the
camera stays open - a user who walks across town keeps geotagging the old spot.

**Session teardown and rebuild — same file** (corrected, see `HW-15-0081`)

```typescript
export async function cameraShooting(cameraPosition: number, surfaceId: string, context: Context): Promise<void> {
  currentContext = context;
  await releaseCamera();          // FIX: the sample calls it without await
  getLocation();
  try {
    let cameraManager: camera.CameraManager = camera.getCameraManager(context);
    let cameraArray: camera.CameraDevice[] = cameraManager.getSupportedCameras();
    if (cameraArray.length <= 0) {
      hilog.error(DOMAIN, TAG, 'No camera devices found!');
      return;
    }
    cameraInput = cameraManager.createCameraInput(cameraArray[cameraPosition]);
    await cameraInput.open();
    // ... profile selection, outputs ...
    photoSession = cameraManager.createSession(camera.SceneMode.NORMAL_PHOTO) as camera.PhotoSession;
    photoSession.beginConfig();
    photoSession.addInput(cameraInput);
    photoSession.addOutput(previewOutput);
    photoSession.addOutput(photoOutPut);
    photoSession.setColorSpace(colorSpaceManager.ColorSpace.DISPLAY_P3);
    await photoSession.commitConfig();
    await photoSession.start();
  } catch (error) {
    hilog.error(DOMAIN, TAG, `The cameraShooting call failed. error: ${JSON.stringify(error)}`);
  }
}

export async function releaseCamera(): Promise<void> {
  try {
    // FIX: every one of these returns a Promise the sample drops
    if (photoSession) { await photoSession.stop(); }
    if (cameraInput) { await cameraInput.close(); }
    if (previewOutput) { await previewOutput.release(); }
    if (photoSession) { await photoSession.release(); }
    if (photoOutPut) {
      photoOutPut.off('photoAssetAvailable');
      photoOutPut.off('captureReady');
      await photoOutPut.release();
    }
  } catch (error) {
    hilog.error(DOMAIN, TAG, `The releaseCamera call failed. error: ${error.code}`);
  }
}
```

**The ordering inside `releaseCamera` is correct and worth keeping**: stop the
session, close the input, release the preview output, release the session,
unregister the photo-output listeners and release it last. What is missing is
the `await` on each step and on the call itself. `cameraInput.close()` returns
before the device is actually free; the shipped code then immediately runs
`cameraManager.createCameraInput` for the other camera and calls `open()` on a
device the previous session has not let go of. On a fast device this usually
works and on a slow one it throws `7400102`. Awaiting costs nothing here -
`cameraShooting` is already `async`.

Two smaller notes. `cameraArray[cameraPosition]` indexes the device list with a
`camera.CameraPosition` **enum value**, which happens to be 0 and 1 on a
two-camera phone and is wrong on anything with three; look the device up by its
`cameraPosition` field instead. And `Index.ets` never assigns `isFront`, so the
front camera's `mirror` flag is always `false`.

**Saving to the gallery — same file** (as shipped)

```typescript
function setPhotoOutputCb(photoOutput: camera.PhotoOutput): void {
  photoOutput.on('photoAssetAvailable',
    async (_err: BusinessError, photoAsset: photoAccessHelper.PhotoAsset): Promise<void> => {
      let accessHelper: photoAccessHelper.PhotoAccessHelper =
        photoAccessHelper.getPhotoAccessHelper(currentContext);
      let assetChangeRequest: photoAccessHelper.MediaAssetChangeRequest =
        new photoAccessHelper.MediaAssetChangeRequest(photoAsset);
      try {
        assetChangeRequest.saveCameraPhoto();
        await accessHelper.applyChanges(assetChangeRequest);
        uri = photoAsset.uri;
        AppStorage.setOrCreate('photoUri', await photoAsset.getThumbnail());
      } catch (error) {
        hilog.error(DOMAIN, TAG, `The setPhotoOutputCb call failed. error: ${error.code}`);
      }
    });
}
```

**`photoAssetAvailable` hands back an asset, not a buffer.** The camera service
has already written the file - including the EXIF built from
`PhotoCaptureSetting` - and `saveCameraPhoto()` plus `applyChanges` is what
commits it to the user's gallery. Nothing here re-encodes the image, which is
why the geotag survives: the app never rewrites the file.

The result reaches the UI through `AppStorage.setOrCreate('photoUri', ...)`,
read by `@StorageLink('photoUri')` in `Index`. That is the only coupling between
the util module and the page, and it is a good one - the util has no reference
to the component. Note that the page also carries an unused `getThumbnail()`
method that queries the newest gallery asset; it is dead code superseded by this
callback.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.CAMERA",
    "reason": "$string:reason_camera",
    "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.WRITE_IMAGEVIDEO",
    "reason": "$string:reason_write_imagevideo",
    "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.READ_IMAGEVIDEO",
    "reason": "$string:reason_read_imagevideo",
    "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.APPROXIMATELY_LOCATION",
    "reason": "$string:reason_app_location",
    "usedScene": { "abilities": ["EntryAbility"], "when": "always" } },
  { "name": "ohos.permission.LOCATION",
    "reason": "$string:reason_location",
    "usedScene": { "abilities": ["EntryAbility"], "when": "always" } }
]
```

- All five are `user_grant`; the document's 权限说明 lists exactly these five
  and matches the zip.
- `READ_IMAGEVIDEO` and `WRITE_IMAGEVIDEO` are **restricted** permissions: they
  need an ACL entry in the signing profile, and a store submission needs a
  justification. This sample does not actually need them - `saveCameraPhoto()`
  on an asset the camera produced writes to the gallery without them, and the
  thumbnail comes from that same asset. The only code that would have needed
  them is the dead `getThumbnail()` method.
- `when: "always"` on both location permissions overstates what a foreground
  camera needs; `"inuse"` is the correct value and avoids a background-location
  review.
- `deviceTypes` is `phone`, `tablet` - no 2in1, consistent with a fixed-ratio
  preview surface.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The preview profile is searched for exactly 1920x1080. A device whose camera
  does not publish that profile logs `Preview profile not found!` and shows a
  black surface - there is no fallback.
- `SURFACE_RATIO` is `632 / 360`, a hardcoded phone aspect; the surface rect is
  computed from `display.getDefaultDisplaySync().width` captured once at module
  load, so a fold or a window resize does not re-lay-out.
- `getCurrentLocation` is one-shot per session start. For a camera that stays
  open, subscribe with `on('locationChange')` instead.
- The document is published under the `convenient-life-v1_2-ts_133` slug despite
  being a utilities/camera topic; the sample zip is `LocationData.zip`.
- `filePreview.closePreview` is called in `onPageShow` to dismiss any preview
  left open when returning from the Photos app.

## Pitfalls

- **`HW-15-0081`** (B/medium, confirmed): `lat`, `lon` and `alt` are
  module-level numbers with no initialiser, so `capture()` builds a
  `camera.Location` out of `undefined` fields for every shot taken before the
  first fix arrives or after the user refuses location - the sample's entire
  purpose. In the same file `releaseCamera` is neither awaited by its caller nor
  internally, so a camera switch races teardown against re-open; and
  `Index.ets` never assigns `isFront`, leaving `mirror` permanently `false`
  while using `CameraPosition` enum values as `getSupportedCameras` indices.
  Fix: gate capture on a real fix, await the releases, set `isFront`.
- **`HW-15-0078`** (D/high, confirmed): systematic across three utilities
  samples - here `Index.ets` calls `requestPermissionsFromUser(...).then(...)`
  without reading `authResults` and with no `.catch`, so the camera pipeline
  starts on a refusal and fails later with a raw error instead of a handled
  state. The 200 ms `setTimeout` inside that `.then` is the surface-id race
  described above. Fix: branch on `data.authResults` and drive the start from
  `XComponent.onLoad`.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-camera-cameramanager.md` - `getSupportedCameras`, `getSupportedOutputCapability`, `createSession`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-camera-cameramanager
- `documentation/harmonyos-references/04_media/arkts-apis-camera-i.md` - `PhotoCaptureSetting`, `Location`, `Profile`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-camera-i
- `documentation/harmonyos-references/04_media/arkts-apis-camera-photooutput.md` - `capture`, `photoAssetAvailable`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-camera-photooutput
- `documentation/harmonyos-guides/05_media/camera-shooting-case.md` - the full session lifecycle this sample condenses
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-shooting-case
- `documentation/harmonyos-references/06_application-services/js-apis-geolocationmanager.md` - `CurrentLocationRequest`, priorities, `getCurrentLocation`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-geolocationmanager
- `documentation/harmonyos-guides/07_application-services/location-guidelines.md` - one-shot versus continuous fixes
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/location-guidelines
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `CAMERA`, `LOCATION`, `APPROXIMATELY_LOCATION`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - `READ_IMAGEVIDEO`, `WRITE_IMAGEVIDEO` and the ACL requirement
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` - `authResults` and the refusal path
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `UTIL-37` - reading the geotag back out of a photo
