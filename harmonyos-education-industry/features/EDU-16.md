---
id: EDU-16
title: Camera and microphone self-check - preview, three-second test recording, and playback before an exam
industry: 04_education
doc: huawei_industry_tree/04_education/docs/16_equipment_detection.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/equipment_detection-0000002398440861
sample: huawei_industry_tree/04_education/downloads/EquipmentDetection.zip
kits: ["@kit.CameraKit", "@kit.MediaKit", "@kit.AbilityKit", "@kit.CoreFileKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit"]
apis: ["camera.getCameraManager", getSupportedCameras, getSupportedSceneModes, getSupportedOutputCapability, createCameraInput, createPreviewOutput, createSession, beginConfig, addInput, addOutput, commitConfig, "Session.start", "camera.SceneMode", "camera.Profile", XComponent, XComponentController, setXComponentSurfaceRect, getXComponentSurfaceId, "media.createAVRecorder", "media.createAVPlayer", fdSrc, AVRecorderConfig, "abilityAccessCtrl.createAtManager", checkAccessToken, requestPermissionsFromUser, "bundleManager.getBundleInfoForSelf", bindSheet]
permissions: ["ohos.permission.CAMERA", "ohos.permission.MICROPHONE"]
min_api: 20
modules: [entry]
findings: [HW-04-0114, HW-04-0115, HW-04-0116, HW-04-0117, HW-04-0118, HW-04-0119, HW-04-0120, HW-04-0121, HW-04-0122, HW-04-0123, HW-04-0124, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card for a **device self-check before something that cannot be
retaken** - an online oral exam, a proctored test, a live class. The user is
walked through a sheet: see yourself in the camera, record three seconds, play it
back, then continue.

This is the industry's most API-dense sample - Camera Kit, `AVRecorder`,
`AVPlayer`, `XComponent` and the full permission flow in one feature - and it is
also the one with the most defects: **eleven findings, three of them blockers.**
Read the permission utility, which is exemplary, and treat the camera and media
utilities as a worked example of what to check rather than what to copy.

The one idea worth taking whole: **`PermissionsRequest.commonRequestPermissions`**
- check the token, request only if missing, and open the settings sheet when the
dialog no longer appears. Every other sample in this industry gets some part of
that wrong.

## Feature checklist

- A bottom sheet with three stages: camera check, microphone check, done.
- Stage 1 shows a live camera preview in an `XComponent`; a toggle confirms the
  picture is visible; 下一步 advances.
- Stage 2 records three seconds from the microphone and offers playback.
- Camera and microphone permissions are requested with a settings fallback.
- The camera is released when the sheet closes.

## Architecture

One `entry` module, four pages (three of which are sheet stages), four utilities.

```
entry/src/main/ets
├── constants/CommonConstants.ets   sheet heights, RESOLUTION_*, TIME_RECORDING (3)
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── pages
│   ├── MainPage.ets                @Entry
│   ├── ExamPage.ets                hosts the bindSheet and the Tabs inside it
│   ├── DetectionBindSheet.ets      stage 1 - XComponent + camera preview
│   └── MicrophoneBindSheet.ets     stage 2 - record and play back
└── utils
    ├── CameraUtil.ets              getPreviewOutput / startPreviewOutput / stopPreviewOutput
    ├── AVRecorderUtil.ets          the three-second test recording
    ├── AVPlayerUtil.ets            playback of that recording
    └── PermissionsRequest.ets      (documented as PermissionsView.ets - HW-04-0121)
```

**The sheet stages are `Tabs` inside one `bindSheet`,** with the sheet height
driven by a `@Link` that each stage sets for itself
(`SHEET_HEIGHT_CAMERA` 480 → `SHEET_HEIGHT_MICROPHONE` 330 →
`SHEET_HEIGHT_END` 280). Advancing is `tabsController.changeIndex(n)` plus a
height change - the sheet resizes to the stage rather than each stage padding
itself into a fixed height.

**Both media utilities have the same fatal shape.** Each opens a file
descriptor, hands it to a media object, and closes it in a `finally` - before the
recording or the playback that reads through it has finished (`HW-04-0114`,
`HW-04-0122`). The correct discipline, which `EDU-11` gets right, is to close the
descriptor only after `release()`.

**`CameraUtil` keeps `session` and `cameraInput` as module-level `let`s.** That
makes them process-global: two previews cannot coexist, the state outlives the
sheet, and `stopPreviewOutput` dereferences an uninitialised `session` if no
preview ever started (`HW-04-0118`).

## Implementation steps

1. **Request both permissions through one check-request-settings helper** before
   opening the sheet - see the snippet below, which is the part of this sample to
   reuse verbatim.
2. **Create the `XComponent` with `XComponentType.SURFACE`** and take the surface
   id in `onLoad`. The id does not exist before that callback.
3. **Choose the preview profile from the device**, not from a literal: read
   `getSupportedOutputCapability(device, sceneMode).previewProfiles` and pick the
   entry whose aspect ratio matches your surface (`HW-04-0117`).
4. **Set the surface rectangle, the component size and the profile from one
   ratio** - or use `.renderFit(RenderFit.RESIZE_CONTAIN)` and set none of them
   (`HW-04-0120`).
5. **Select the camera by position, not by index** (`HW-04-0115`), and **check
   the scene mode you are actually going to create** (`HW-04-0116`).
6. **Configure the session in order**: `beginConfig` → `addInput` → `addOutput` →
   `commitConfig` → `start`. Wrap the whole thing in `try`/`catch`; every step
   throws a `CameraErrorCode`.
7. **Release in `aboutToDisappear`**, not on a button: stop the session, release
   the preview output, release the session, close the input (`HW-04-0118`).
8. **For the test recording**, open the file, `prepare(config)`, `start()`, and
   schedule the stop - but **close the descriptor in the stop path, after
   `release()`** (`HW-04-0114`).
9. **For playback, assign `fdSrc` and let the state machine drive.** Do not also
   call `prepare()` and `play()` from the caller (`HW-04-0123`).
10. **Hold one utility instance per page** rather than constructing a new one per
    call (`HW-04-0124`).

## Verified snippets

All snippets are from `EquipmentDetection.zip`. Corrected forms are marked.

**The permission flow — `entry/src/main/ets/utils/PermissionsRequest.ets`** (as shipped — copy this one)

```typescript
import { abilityAccessCtrl, bundleManager, Permissions } from '@kit.AbilityKit';

async commonRequestPermissions(context: UIContext, permissions: Array<Permissions>) {
  const isPermission: boolean = await this.checkPermissions(permissions);
  if (!isPermission) {
    const isDialogShown = await this.requestPermissions(context, permissions);
    if (isDialogShown !== true) {
      // the dialog did not appear -> refused permanently -> open the settings page
      await this.checkAndOpenPermissionsInSystemSettings(context, permissions);
    }
  }
}

async checkPermissions(permissions: Array<Permissions>) {
  for (const permission of permissions) {
    const grantStatus = await this.checkAccessToken(permission);
    if (grantStatus !== abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED) {
      return false;                 // accumulate correctly: any missing one fails the set
    }
  }
  return true;
}

async checkAccessToken(permission: Permissions): Promise<abilityAccessCtrl.GrantStatus> {
  const atManager = abilityAccessCtrl.createAtManager();
  let grantStatus: abilityAccessCtrl.GrantStatus = abilityAccessCtrl.GrantStatus.PERMISSION_DENIED;
  let tokenId: number = 0;
  try {
    const bundleInfo: bundleManager.BundleInfo =
      await bundleManager.getBundleInfoForSelf(bundleManager.BundleFlag.GET_BUNDLE_INFO_WITH_APPLICATION);
    tokenId = bundleInfo.appInfo.accessTokenId;
    grantStatus = await atManager.checkAccessToken(tokenId, permission);
  } catch (error) {
    hilog.error(0x000, 'testTag', `Failed to check access token ${(error as BusinessError).code}`);
  }
  return grantStatus;
}
```

**Three steps, in this order, and the third is the one everyone forgets.**
`checkAccessToken` avoids raising a dialog for a permission already granted;
`requestPermissionsFromUser` handles the first refusal; and when the dialog is
reported as *not shown*, the user has refused permanently and the only remaining
route is the settings page. `EDU-11` and `EDU-13` each omit part of this;
`EDU-01`'s framework code has none of it.

**`GET_BUNDLE_INFO_WITH_APPLICATION` is required** for `bundleInfo.appInfo` to be
populated - without that flag `accessTokenId` is undefined and `checkAccessToken`
rejects. Initialising `grantStatus` to `PERMISSION_DENIED` before the `try` means
a failure denies rather than returning `undefined`.

**Camera preview — `entry/src/main/ets/utils/CameraUtil.ets`** (corrected, see `HW-04-0115`, `HW-04-0116`, `HW-04-0117`, `HW-04-0118`)

```typescript
// FIX: the sample keeps `session` and `cameraInput` as module-level lets.
// Make them fields of the component that owns the preview.

async function startPreview(cameraManager: camera.CameraManager, surfaceId: string) {
  try {
    const cameras = cameraManager.getSupportedCameras();
    // FIX: sample indexes cameraArray[1] behind a `length === 0` guard only
    const device = cameras.find(c => c.cameraPosition === camera.CameraPosition.CAMERA_POSITION_FRONT)
                ?? cameras[0];
    if (!device) {
      return;
    }
    const mode = camera.SceneMode.NORMAL_VIDEO;
    // FIX: sample checks NORMAL_PHOTO support and then creates a NORMAL_VIDEO session
    if (cameraManager.getSupportedSceneModes(device).indexOf(mode) < 0) {
      return;
    }
    // FIX: sample hard-codes { CAMERA_FORMAT_YUV_420_SP, 1920 x 1080 }
    const capability = cameraManager.getSupportedOutputCapability(device, mode);
    const profile = capability.previewProfiles.find(p => p.size.width / p.size.height === 16 / 9);
    if (!profile) {
      return;
    }
    const previewOutput = cameraManager.createPreviewOutput(profile, surfaceId);

    const input = cameraManager.createCameraInput(device);
    await input.open();
    const session = cameraManager.createSession(mode) as camera.VideoSession;
    session.beginConfig();
    session.addInput(input);
    session.addOutput(previewOutput);
    await session.commitConfig();
    await session.start();
    return { session, input, previewOutput };
  } catch (error) {
    hilog.error(0x0000, 'CameraUtil', `startPreview failed: ${(error as BusinessError).code}`);
    return undefined;
  }
}

async function stopPreview(ctx?: { session: camera.VideoSession, input: camera.CameraInput,
                                   previewOutput: camera.PreviewOutput }) {
  if (!ctx) {                                  // FIX: sample calls session.stop() unguarded
    return;
  }
  await ctx.session.stop();
  await ctx.previewOutput.release();           // FIX: absent in the sample
  await ctx.session.release();                 // FIX: absent in the sample
  await ctx.input.close();
}
```

**The profile must come from the device.** `createPreviewOutput` throws when the
profile is not in `previewProfiles`, and the guide is explicit that you read the
capability and pick from it. The sample's literal 1920x1080 YUV_420_SP works on
the device it was written on and throws on one that does not offer exactly that.

**`beginConfig` … `commitConfig` is a transaction.** Inputs and outputs added
outside that bracket are not part of the session; `start()` before `commitConfig`
fails. This is the one part of the sample's camera code that is correct.

**The scene mode appears three times** - the capability check, the
`getSupportedOutputCapability` call and `createSession` - and all three must
agree. The sample checks `NORMAL_PHOTO` and creates `NORMAL_VIDEO`.

**Wiring the preview to the surface — `entry/src/main/ets/pages/DetectionBindSheet.ets`** (corrected, see `HW-04-0117`, `HW-04-0118`, `HW-04-0120`)

```typescript
XComponent({
  type: XComponentType.SURFACE,
  controller: this.mXComponentController
})
  .onLoad(async () => {
    // FIX: the sample sets the surface to 1170 x 2080, sizes the component
    //      1200 x 1920 and requests a 1920 x 1080 profile - three ratios.
    this.previewCtx = await startPreview(
      this.cameraManager,
      this.mXComponentController.getXComponentSurfaceId());
  })
  .renderFit(RenderFit.RESIZE_CONTAIN)      // FIX: lets the framework fit the surface
  .width('100%')
  .height(200)

// FIX: the sample releases the camera only inside the 下一步 button's handler,
//      and only when this.isCamera is true. There is no aboutToDisappear.
aboutToDisappear(): void {
  stopPreview(this.previewCtx);
  this.previewCtx = undefined;
}
```

**`getXComponentSurfaceId()` is only valid inside `onLoad`** - the surface does
not exist before it. That is why the whole camera bring-up hangs off that
callback rather than off `aboutToAppear`.

**The camera must be released on every exit path.** A `bindSheet` can be
dismissed by swiping down or pressing back, neither of which runs the next
button's handler, and the handler itself does nothing when the confirmation
toggle is off. On all of those paths the sample leaves the camera open for every
other application on the device.

**The three-second test recording — `entry/src/main/ets/utils/AVRecorderUtil.ets`** (corrected, see `HW-04-0114`)

```typescript
private avProfile: media.AVRecorderProfile = {
  audioBitrate: 100000,
  audioChannels: 2,
  audioCodec: media.CodecMimeType.AUDIO_AAC,
  audioSampleRate: 48000,
  fileFormat: media.ContainerFormatType.CFT_MPEG_4A
};

async startRecordingProcess(context: UIContext) {
  if (this.avRecorder !== undefined) {
    await this.avRecorder.release();
    this.avRecorder = undefined;
  }
  this.avRecorder = await media.createAVRecorder();
  this.setAudioRecorderCallback();
  const path = context.getHostContext()?.filesDir + `/ken.m4a`;   // the container is M4A
  // FIX: the sample wraps everything below in try { ... } finally { fs.close(this.file); },
  //      which closes the descriptor the instant start() resolves - three seconds early.
  this.file = fs.openSync(path, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
  this.avConfig.url = 'fd://' + this.file.fd;
  await this.avRecorder.prepare(this.avConfig);
  await this.avRecorder.start();
  this.stopTimer = setTimeout(() => { this.stopRecordingProcess(); },
                              CommonConstants.TIME_RECORDING * 1000);
}

async stopRecordingProcess(): Promise<void> {
  if (this.stopTimer !== undefined) {
    clearTimeout(this.stopTimer);              // FIX: sample never clears it
    this.stopTimer = undefined;
  }
  if (this.avRecorder === undefined) {
    return;
  }
  if (this.avRecorder.state === 'started' || this.avRecorder.state === 'paused') {
    await this.avRecorder.stop();
  }
  await this.avRecorder.reset();
  await this.avRecorder.release();
  this.avRecorder = undefined;
  if (this.file !== null) {                    // FIX: the descriptor belongs here
    fs.close(this.file);
    this.file = null!;
  }
}
```

**A `finally` around an asynchronous recording is always wrong.** The write
outlives the function, so there is no lexical scope whose exit corresponds to the
end of it. The descriptor's owner is the *stop* path, not the *start* path -
which is exactly how `EDU-11`'s recorder is written.

The file is named `.wav` in the sample while the container is `CFT_MPEG_4A`;
harmless because the player opens by descriptor, misleading to anyone who looks
in the sandbox.

**Playback — `entry/src/main/ets/utils/AVPlayerUtil.ets`** (corrected, see `HW-04-0122`, `HW-04-0123`, `HW-04-0124`)

```typescript
export class AVPlayUtil {
  private file: fs.File | null = null;
  private avPlayer: media.AVPlayer | undefined = undefined;

  async init(): Promise<void> {
    this.avPlayer = await media.createAVPlayer();
    this.avPlayer.on('stateChange', async (state) => {
      switch (state) {
        case 'initialized': this.avPlayer?.prepare(); break;   // fdSrc assignment lands here
        case 'prepared':    this.avPlayer?.play();    break;
        case 'completed':   await this.endAudio();    break;   // FIX: sample did `new AVPlayUtil()`
      }
    });
  }

  async playAudio(context: UIContext) {
    const path = context.getHostContext()?.filesDir + `/ken.m4a`;
    this.file = fs.openSync(path, fs.OpenMode.READ_ONLY);      // FIX: sample uses READ_WRITE|CREATE
    this.avPlayer!.fdSrc = { fd: this.file.fd };
    // FIX: the sample then calls prepare() and play() here as well, while the
    //      handler above is already doing it - play() runs from `initialized`.
  }

  async endAudio() {
    await this.avPlayer?.stop();
    if (this.file !== null) {
      fs.close(this.file);                                     // FIX: sample closes in a finally
      this.file = null;
    }
  }
}
```

**Assigning `fdSrc` is the trigger, not `prepare()`.** It drives the player to
`initialized`, and the handler takes it from there - `prepare` on `initialized`,
`play` on `prepared`. Calling `prepare` and `play` from the caller as well means
two `prepare`s in flight and a `play` issued from a state that does not accept
it.

**Opening the playback file `READ_WRITE | CREATE`** means a missing recording is
silently created empty and the player fails on empty content, instead of the
check reporting that nothing was recorded. `READ_ONLY` turns that into a clear
failure.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.CAMERA",
    "reason": "$string:camera_reason",        // FIX: sample uses $string:module_desc
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  },
  {
    "name": "ohos.permission.MICROPHONE",
    "reason": "$string:microphone_reason",    // FIX: sample uses $string:app_name
    "usedScene": {
      "abilities": ["EntryAbility"],          // FIX: sample says ["FormAbility"]
      "when": "inuse"                         // FIX: sample omits `when` on both entries
    }
  }
]
```

- Both are `user_grant`, so `reason` and `usedScene` are mandatory, `when` cannot
  be empty, and the reason must name the scenario (`HW-04-0119`).
- Request them together before opening the sheet, through
  `commonRequestPermissions`.
- The shipped zip also contains `entry/patch.json`, a quick-fix artefact that has
  no place in a sample (`HW-04-0121`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The test recording is fixed at three seconds** (`TIME_RECORDING`), AAC in an
  M4A container at 48 kHz stereo, written to `filesDir/ken.wav` - one file, so
  re-testing overwrites the previous take.
- The check is **presence, not quality**: the user confirms by toggle that they
  can see and hear. Nothing measures the amplitude, the frame rate or the noise
  floor.
- The camera preview holds the device camera for as long as stage 1 is open, and
  in the shipped code beyond that (`HW-04-0118`).
- Sheet heights are three fixed constants; the stages do not measure themselves.
- `CameraUtil`'s module-level `session`/`cameraInput` make the preview a process
  singleton - a second `DetectionBindSheet` would overwrite the first's state.

## Pitfalls

- **`HW-04-0114` — the recording descriptor is closed in a `finally` right after
  `start()`,** three seconds before the recording ends.
- **`HW-04-0122` — the playback descriptor is closed the same way,** before the
  player has read anything; the class's own `endAudio` is the correct place and
  is dead code.
- **`HW-04-0123` — `prepare` is called from both the state handler and the
  caller,** and `play` is issued while the player is still `initialized`.
- **`HW-04-0115` — `cameraArray[1]` behind a `length === 0` guard.** A
  single-camera device passes the guard and then throws.
- **`HW-04-0116` — `NORMAL_PHOTO` is checked, `NORMAL_VIDEO` is created.**
- **`HW-04-0117` — the preview profile is a literal,** the `catch` is empty and
  the caller casts the resulting `undefined` to `PreviewOutput`, so a failure
  surfaces as `addOutput(undefined)`.
- **`HW-04-0118` — the camera is released only by the next button,** never in
  `aboutToDisappear`, and neither the preview output nor the session is released
  at all.
- **`HW-04-0119` — both permission declarations omit `when`,** one names a
  non-existent `FormAbility`, and both reasons are unrelated placeholder strings.
- **`HW-04-0120` — three different aspect ratios** for the component, the surface
  and the profile, against a guide that requires them to match.
- **`HW-04-0121` — the documented `PermissionsView.ets` is really
  `PermissionsRequest.ets`,** and an undocumented `patch.json` ships in the zip.
- **`HW-04-0124` — utility instances are constructed per call** and discarded,
  orphaning the file handle the class exists to own.
- **Never wrap an asynchronous media operation in `try`/`finally` to close its
  descriptor.** The operation outlives the scope; close it from the stop path.

## References

- `documentation/harmonyos-guides/02_media/camera-preview.md` - the canonical preview flow: surface, `previewProfiles`, matching scene modes, session transaction, aspect-ratio requirement
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-preview
- `documentation/harmonyos-guides/05_media/camera-preparation.md` - requesting the camera permission before any Camera Kit call
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-preparation
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - `XComponentType.SURFACE`, `getXComponentSurfaceId`, `setXComponentSurfaceRect`, `renderFit`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-references/04_media/arkts-apis-media-avrecorder.md` - the recorder state machine and `AVRecorderConfig.url` as `fd://`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avrecorder
- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - `fdSrc` and the player state machine
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` - `checkAccessToken`, `requestPermissionsFromUser`, `requestPermissionOnSetting`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `reason`, `usedScene.abilities` and the mandatory `when`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- `EDU-11` - the same recorder and player used correctly: the descriptor is closed after `release()`
- `EDU-01` - the `AVPlayer` + `XComponent` video pattern, and the same deprecated-XComponent question
