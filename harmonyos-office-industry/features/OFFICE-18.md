---
id: OFFICE-18
title: Online meeting main/sub window swap - two sub-windows sharing one camera session across two XComponent surfaces
industry: 05_office
doc: huawei_industry_tree/05_office/docs/18_conference_window_change.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/conference_window_change-0000002325708930
sample: huawei_industry_tree/05_office/downloads/ConferenceWindowChange.zip
kits: ["@kit.ArkUI", "@kit.CameraKit", "@kit.AbilityKit", "@kit.ArkData", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["windowStage.createSubWindow", "window.setUIContent", "window.resize", "window.moveWindowTo", "window.showWindow", "window.setRaiseByClickEnabled", "window.setWindowBackgroundColor", "window.destroyWindow", "window.findWindow", "window.getWindowProperties", "window.getWindowAvoidArea", "window.on('avoidAreaChange')", "display.getAllDisplays", "display.getDefaultDisplaySync", XComponent, XComponentController, "XComponentController.getXComponentSurfaceId", "camera.getCameraManager", "CameraManager.getSupportedCameras", "CameraManager.createCameraInput", "CameraManager.createPreviewOutput", "CameraManager.createSession", "camera.SceneMode.NORMAL_VIDEO", "camera.VideoSession", "VideoSession.beginConfig", "VideoSession.addInput", "VideoSession.addOutput", "VideoSession.commitConfig", "VideoSession.start", "VideoSession.stop", "VideoSession.release", "CameraInput.open", "CameraInput.close", "CameraOutput.release", "camera.Profile", "@StorageLink", "@Watch", "@StorageProp", "abilityAccessCtrl.requestPermissionsFromUser", "AtManager.requestPermissionOnSetting", "preferences.getPreferencesSync", "Preferences.hasSync"]
permissions: ["ohos.permission.CAMERA"]
min_api: 20
modules: [entry]
findings: [HW-05-0097, HW-05-0098, HW-05-0099, HW-05-0100, HW-05-0101, HW-05-0102, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when a conferencing UI needs **two independently positioned video
surfaces on screen at once** - a full-screen main stage and a small floating
tile - with the ability to swap who is on which by tapping the tile.

The architectural choice that makes this scenario distinctive: the two stages are
not two components inside one page, they are **two `windowStage` sub-windows**,
each with its own page and its own `XComponent` surface. That gives the floating
tile real window semantics (position, size, z-order, click-raise behaviour) that
an in-page overlay cannot have.

One camera session then feeds **both** surfaces: `createPreviewOutput` is called
once per surfaceId and every output is added to the same `VideoSession`, so
swapping the stages is just a visibility flip - no camera restart.

## Feature checklist

The application must be able to:

- Create a full-screen moderator sub-window and a small guest sub-window, in that
  order, and stop either from raising itself on click.
- Give each sub-window its own page and its own `XComponent` surface, and register
  both surfaceIds in shared state.
- Request the camera permission with a two-stage flow that remembers whether the
  first dialog was already answered.
- Open one camera video session and attach a preview output for **every**
  registered surface.
- Show the camera picture only in the window that currently hosts the local
  participant.
- Swap the participants between the two windows when the guest tile is tapped,
  and move the camera picture with its owner.
- Run a meeting timer and stop it when the page goes away.
- Release the camera when a window's content is destroyed, and when the camera is
  switched off.

## Architecture

Single `entry` HAP, but three pages because of the multi-window design:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | publishes the `windowStage` and the avoid-area insets into `AppStorage`, loads `pages/HomePage` |
| `pages/HomePage.ets` | `@Entry`, renders **nothing** - its only job is to create the two sub-windows in `aboutToAppear` |
| `pages/ModeratorPage.ets` | the main window's content: timer, toolbar, `WindowContentComponent` |
| `pages/GuestPage.ets` | the sub window's content: `WindowContentComponent` |
| `component/WindowContentComponent.ets` | the shared body - avatar/name, the `XComponent`, and **the whole camera pipeline** |
| `utils/PermissionsUtils.ets` | the permission ladder, with a `Preferences`-backed "already asked" flag |
| `model/AttenderData.ets`, `constants/CommonConstants.ets`, `utils/Logger.ets` | data, constants, logging |

Three pieces of application-global state tie the two windows together:

```ts
@StorageLink('surfaceIds') surfaceIds: string[] = [];                                   // both windows' surfaces
@StorageLink('isCameraOpen') @Watch('cameraStatusChange') isCameraOpen: boolean = false; // camera on/off
@StorageLink('hostAttender') @Watch('hostAttenderChange') hostAttender: AttenderData = MYSELF; // who is on the main stage
```

`@Watch` is the mechanism that makes the swap work: both window instances observe
the same `AppStorage` keys, so a tap in the guest window changes `hostAttender`,
and **both** components' `hostAttenderChange` handlers run and re-derive which
participant they show and whether their `XComponent` is visible.

Swap flow:

```
guest window body onClick
  -> hostAttender = this.attender            (AppStorage, @StorageLink)
  -> @Watch fires in BOTH components
       moderator window: attender = hostAttender
       guest window:     attender = preAttender      (the one that just left the main stage)
       both: preAttender = hostAttender
       both: videoVisible = (attender is me) && isCameraOpen
```

Camera flow:

```
toolbar toggles isCameraOpen -> @Watch cameraStatusChange in both windows
  camera off -> await releaseCamera()   (both windows release, deliberately)
  not the camera owner -> return
  camera on  -> createRecorder()
                  getCameraManager -> getSupportedCameras
                  createCameraInput(devices[0]) -> await open()
                  createSession(NORMAL_VIDEO) as VideoSession
                  beginConfig -> addInput
                  surfaceIds.forEach: createPreviewOutput(profile, surfaceId) -> addOutput
                  await commitConfig -> await start
```

## Implementation steps

1. **Declare `ohos.permission.CAMERA`** with a complete `usedScene` including
   `"when": "inuse"` - the sample does this correctly, and it matches the
   document's 权限说明 section.
2. **Publish the `windowStage`** from `EntryAbility` into `AppStorage` so the page
   can create sub-windows, and publish the avoid-area insets. Release the
   `avoidAreaChange` subscription in `onWindowStageDestroy` (HW-05-0102).
3. **Read the display size** with `display.getAllDisplays`, checking `err` and the
   array length before indexing (HW-05-0100).
4. **Create the moderator window first and show it, then create the guest
   window.** The order is load-bearing and the sample says why in its comments:
   `showWindow` raises the window level, so the guest window must be created after
   the moderator window has been shown, and `setRaiseByClickEnabled(false)` only
   takes effect once `showWindow` has completed. The document's snippet omits both
   rules (HW-05-0101).
5. **Size and place each window.** `resize` + `moveWindowTo` in raw pixels -
   convert from vp with `uiContext.vp2px` for the guest tile. Handle the promises
   these calls return (HW-05-0099).
6. **Destroy both sub-windows** when the creating page goes away (HW-05-0099).
7. **Capture each surfaceId in `XComponent.onLoad`** and register it in the shared
   array - and **remove it** when the component is destroyed (HW-05-0098). Set the
   `XComponent` invisible immediately after; the comment in the sample records the
   key fact that a later visibility change does not invalidate the surfaceId.
8. **Build one session across all surfaces.** Loop `surfaceIds`, create a preview
   output per surface, add them all to the same `VideoSession`, then
   `commitConfig` and `start`.
9. **Gate the camera on ownership.** Only the window whose `attender` is the local
   user creates the session; the other returns early from `cameraStatusChange`.
10. **Release properly.** `releaseCamera` must **await** every stop/close/release
    before `createRecorder` opens a new input (HW-05-0097).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### Creating the two sub-windows in the right order

`ConferenceWindowChange.zip#entry/src/main/ets/pages/HomePage.ets`

```ts
aboutToAppear() {
  display.getAllDisplays((err: BusinessError, data: Array<display.Display>) => {
    const DISPLAY_CONFIG = data;                    // err unchecked - HW-05-0100
    this.screenWidth = DISPLAY_CONFIG[0].width;
    this.screenHeight = DISPLAY_CONFIG[0].height;

    this.windowStage.createSubWindow(CommonConstants.MODERATOR_WINDOW, async (err, subWindow) => {
      if (err.code > 0) {
        Logger.error(`create subwindow ${CommonConstants.MODERATOR_WINDOW} error: ${JSON.stringify(err)}`);
        return;
      }
      subWindow.setUIContent('pages/ModeratorPage', () => {
        subWindow.setWindowBackgroundColor(CommonConstants.NORMAL_WINDOW_BACKGROUND);
      });
      subWindow.resize(this.screenWidth, this.screenHeight);
      subWindow.moveWindowTo(0, 0);
      // 通过subWindow.setRaiseByClickEnabled(false)来禁止窗口点击抬升，避免窗口层级混乱
      await subWindow.showWindow();
      // 该接口一定要在showWindow接口调用完毕之后调用才能确保生效
      await subWindow.setRaiseByClickEnabled(false);
      // showWindow会提升窗口层级，因此副窗口创建需要在主窗口showWindow之后
      this.windowStage.createSubWindow(CommonConstants.GUEST_WINDOW, (err, guestWindow) => {
        if (err.code > 0) {
          Logger.error(`create subwindow ${CommonConstants.MODERATOR_WINDOW} error: ${JSON.stringify(err)}`);
          return;
        }
        guestWindow.setUIContent('pages/GuestPage', () => {
          guestWindow.setWindowBackgroundColor(CommonConstants.NORMAL_WINDOW_BACKGROUND);
        });
        guestWindow.resize(this.getUIContext().vp2px(CommonConstants.GUEST_WINDOW_WIDTH),
          this.getUIContext().vp2px(CommonConstants.GUEST_WINDOW_HEIGHT));
        guestWindow.moveWindowTo(this.screenWidth - this.getUIContext().vp2px(CommonConstants.SUB_X_POSITION_DIS),
          this.getUIContext().vp2px(this.topRectHeight + CommonConstants.SUB_Y_POSITION_DIS));
        guestWindow.showWindow()
          .then(() => {
            guestWindow.setRaiseByClickEnabled(false);
          });
      });
    });
  });
}

build() {
  Column() {
  }
  .height('100%')
  .width('100%');
}
```

The three comments in this block are the most valuable thing in the sample - they
encode ordering rules that are not obvious and that the document drops:
`setRaiseByClickEnabled` after `showWindow`, and the guest window created after
the moderator window is shown. The empty `build()` is deliberate: `HomePage`
exists only to spawn the two windows.

### Shared state and the swap

`ConferenceWindowChange.zip#entry/src/main/ets/component/WindowContentComponent.ets`

```ts
@State attender: AttenderData = MYSELF;
@State videoVisible: boolean = true;
@StorageLink('surfaceIds') surfaceIds: string[] = [];
@StorageLink('isCameraOpen') @Watch('cameraStatusChange') isCameraOpen: boolean = false;
@StorageLink('hostAttender') @Watch('hostAttenderChange') hostAttender: AttenderData = MYSELF;
windowName: string = CommonConstants.MODERATOR_WINDOW;
private preAttender: AttenderData = MYSELF;

aboutToAppear() {
  let windowProp = window.findWindow(this.windowName).getWindowProperties();
  this.xWidth = this.uiContext.px2vp(windowProp.windowRect.width);
  this.xHeight = this.uiContext.px2vp(windowProp.windowRect.height);
}

/**
 * 点击副窗口时，交换两个窗口的与会者数据，并调整两个窗口内XComponent组件的可见性来控制相机流的切换
 */
hostAttenderChange() {
  if (this.windowName === CommonConstants.MODERATOR_WINDOW) {
    this.attender = this.hostAttender;
  } else {
    this.attender = this.preAttender;
  }
  this.preAttender = this.hostAttender;
  if (this.attender.attenderId === MYSELF.attenderId && this.isCameraOpen) {
    this.videoVisible = true;
  } else {
    this.videoVisible = false;
  }
}
```

`window.findWindow(this.windowName).getWindowProperties()` is how each component
sizes its `XComponent` to its own window - the same component renders in both
windows and adapts by name.

### Surface registration

`ConferenceWindowChange.zip#entry/src/main/ets/component/WindowContentComponent.ets`

```ts
XComponent({
  id: 'RenderXComponent' + this.attender.attenderId,
  type: XComponentType.SURFACE,
  controller: this.xComponentController
})
  .onLoad(() => {
    // 初始化XComponent组件时获取到对应的surfaceId，后续的可见性变更不影响surfaceId
    this.xComponentSurfaceId = this.xComponentController.getXComponentSurfaceId();
    // 将主副窗口的surfaceId放入数组中管理，方便添加预览流
    this.surfaceIds.push(this.xComponentSurfaceId);      // never removed - HW-05-0098
    // 获取surfaceId后将可见性设置为false
    this.videoVisible = false;
  })
  .width(this.xWidth)
  .height(this.xHeight)
  .visibility(this.videoVisible ? Visibility.Visible : Visibility.None);
```

The first comment is the reason the surface is captured once in `onLoad` rather
than on demand: toggling `visibility` later does not invalidate the surfaceId, so
the preview outputs stay valid across every swap.

Corrected teardown:

```ts
aboutToDisappear(): void {
  const i = this.surfaceIds.indexOf(this.xComponentSurfaceId);
  if (i >= 0) {
    this.surfaceIds.splice(i, 1);
    this.surfaceIds = [...this.surfaceIds];   // reassign so @StorageLink observes it
  }
  this.releaseCamera();
}
```

### One session, one preview output per surface

`ConferenceWindowChange.zip#entry/src/main/ets/component/WindowContentComponent.ets`

```ts
async cameraStatusChange() {
  // 当相机关闭，主副窗口组件均需要释放一次相机，避免窗口切换后导致相机资源无法释放
  if (!this.isCameraOpen) {
    await this.releaseCamera();
  }
  if (this.attender.attenderId !== MYSELF.attenderId) {
    return;
  }
  // 相机开启时，由相机持有者所在窗口创建、开启相机流
  if (this.isCameraOpen) {
    this.createRecorder();
    this.videoVisible = true;
  } else {
    this.videoVisible = false;
  }
}

async createRecorder(): Promise<void> {
  await this.releaseCamera();
  let context = this.uiContext.getHostContext();
  if (!context) { Logger.error('camera.getCameraManager error'); return; }
  let cameraManager = camera.getCameraManager(context);
  if (!cameraManager) { Logger.error('camera.getCameraManager error'); return; }

  let cameraDevices: Array<camera.CameraDevice> = cameraManager.getSupportedCameras();
  if (cameraDevices !== undefined && cameraDevices.length <= 0) {
    Logger.error('cameraManager.getSupportedCameras error!');
    return;
  }

  // 使用XComponent来展示相机的预览流
  let xComponentPreviewProfile: camera.Profile = {
    format: camera.CameraFormat.CAMERA_FORMAT_YUV_420_SP,
    size: {
      width: CommonConstants.CAMERA_RESOLUTION_WIDTH,
      height: CommonConstants.CAMERA_RESOLUTION_HEIGHT
    }
  };

  this.cameraInput = cameraManager.createCameraInput(cameraDevices[0]);
  if (this.cameraInput === undefined) { return; }
  await this.cameraInput.open();

  this.videoSession = cameraManager.createSession(camera.SceneMode.NORMAL_VIDEO) as camera.VideoSession;
  if (this.videoSession === undefined) { return; }

  this.videoSession.beginConfig();
  this.videoSession.addInput(this.cameraInput);

  // 添加XComponent预览流到会话
  this.surfaceIds.forEach((surfaceId) => {
    let preview = cameraManager.createPreviewOutput(xComponentPreviewProfile, surfaceId);
    this.previewOutputs.push(preview);
    this.videoSession?.addOutput(preview);
  });

  await this.videoSession.commitConfig();
  await this.videoSession.start();
}
```

Every step in the shipped version is individually wrapped in `try/catch` with a
`Logger.error` (elided above for length) - unlike OFFICE-07, whose equivalent
logs go nowhere because of an invalid hilog domain. Note the profile is a
hand-built `camera.Profile` rather than one selected from
`getSupportedOutputCapability`, which is simpler but does not adapt to the device.

### Release - as shipped, and corrected

`ConferenceWindowChange.zip#entry/src/main/ets/component/WindowContentComponent.ets`

```ts
async releaseCamera(): Promise<void> {
  // 停止视频会话
  this.videoSession?.stop();
  // 关闭相机输入流
  this.cameraInput?.close();
  // 释放预览流资源
  this.previewOutputs.forEach((preview) => {
    preview.release();
  });
  let len = this.previewOutputs.length;
  for (let i = 0; i < len; i++) {
    this.previewOutputs.pop();
  }
  // 释放视频会话资源
  this.videoSession?.release();
}
```

```ts
// corrected - the callers already write `await this.releaseCamera()`
async releaseCamera(): Promise<void> {
  try {
    await this.videoSession?.stop();
    await this.cameraInput?.close();
    for (const preview of this.previewOutputs) {
      await preview.release();
    }
    this.previewOutputs = [];
    await this.videoSession?.release();
  } catch (error) {
    Logger.error(`releaseCamera failed: ${JSON.stringify(error)}`);
  } finally {
    this.videoSession = undefined;
    this.cameraInput = undefined;
  }
}
```

### The permission ladder - the best version in this industry

`ConferenceWindowChange.zip#entry/src/main/ets/utils/PermissionsUtils.ets`

```ts
initMethod(uiContext: UIContext) {
  this.uiContext = uiContext;
  this.permsStore = preferences.getPreferencesSync(uiContext.getHostContext(), { name: 'permsHasOnSetting' });
}

async commonRequestPermissions(permissions: Array<Permissions>) {
  let checkOk: boolean = await this.checkPermissions(permissions);
  if (!checkOk) {
    let keyPerms = JSON.stringify(permissions);
    if (!this.permsStore?.hasSync(keyPerms)) {
      this.requestPermissions(permissions); // 一次授权
    } else if (this.permsStore?.hasSync(keyPerms)) {
      this.requestPermissionsOnSetting(permissions); // 二次授权
    }
  }
}

async requestPermissions(permissions: Array<Permissions>): Promise<void> {
  let atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
  try {
    if (this.uiContext?.getHostContext() !== undefined) {
      let data: PermissionRequestResult =
        await atManager.requestPermissionsFromUser(this.uiContext?.getHostContext() as common.UIAbilityContext,
          permissions);
      let keyPerms = JSON.stringify(permissions);
      if (!data.authResults.every((res) => res === 0)) {
        this.permsStore?.putSync(keyPerms, 1);
        this.permsStore?.flush();
      }
    }
  } catch (err) {
    Logger.error(`requestPermissions1 err Code is ${err.code}, message is ${err.message}`);
  }
}

async requestPermissionsOnSetting(permissions: Array<Permissions>): Promise<void> {
  try {
    let keyPerms = JSON.stringify(permissions);
    let atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
    if (this.uiContext?.getHostContext() !== undefined) {
      let res: Array<abilityAccessCtrl.GrantStatus> =
        await atManager.requestPermissionOnSetting(this.uiContext?.getHostContext() as common.UIAbilityContext,
          permissions);
      if (!res.every((grad) => grad === abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED)) {
        this.permsStore?.putSync(keyPerms, 2);
        this.permsStore?.flush();
      }
    }
  } catch (e) {
    Logger.info('requestPermissions2 err', JSON.stringify(e));
  }
}
```

Copy this one rather than the ladders in OFFICE-03 or OFFICE-07: it **casts the
context on both calls** (unlike HW-05-0013 and HW-05-0042), it **checks
`authResults`** (unlike HW-05-0002 and HW-05-0065), and it persists a
"first dialog already answered" flag in `Preferences` so the second-stage
Settings dialog is chosen on a durable fact rather than on `dialogShownResults`.

## Permissions & config

`ConferenceWindowChange.zip#entry/src/main/module.json5`

```json5
"requestPermissions": [
  {
    "name" : "ohos.permission.CAMERA",
    "reason": "$string:permission_reason",
    "usedScene": {
      "abilities": ["EntryAbility"],
      "when": "inuse"
    }
  }
]
```

Notes on the config:

- `ohos.permission.CAMERA` is genuinely required here: this scenario drives the
  camera directly through `CameraManager` and `XComponent`, not through a system
  picker. Contrast OFFICE-05 (HW-05-0022) and OFFICE-16, where the same
  declaration would be an over-declaration because the capture goes through a
  system picker.
- The declaration matches the document's 权限说明 section exactly, and the
  `usedScene` carries `when` - better than the OFFICE-07 sample (HW-05-0041).
- It is a `user_grant` permission, so the runtime ladder in `PermissionsUtils` is
  mandatory; declaring it is not enough.
- **No microphone permission** is declared, and none is used - this sample renders
  video only.
- `build-profile.json5` pins the SDK to `6.0.0(20)`.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **Window ordering is mandatory.** `setRaiseByClickEnabled(false)` only takes
  effect after `showWindow` has completed, and the guest window must be created
  after the moderator window has been shown because `showWindow` raises the window
  level. Both rules are recorded in the sample's own comments.
- **Sub-windows are application child windows.** `destroyWindow` is documented as
  applying to "a system window, an application child window, a global floating
  window, or a modal window" - it is the correct release for both of these.
- **`resize` and `moveWindowTo` take pixels.** The guest window's vp constants are
  converted with `uiContext.vp2px` at the call site.
- **The surfaceId survives visibility changes.** Captured once in
  `XComponent.onLoad`; toggling `visibility` afterwards does not invalidate it,
  which is what lets one session keep two preview outputs across every swap.
- **One camera owner at a time.** `cameraStatusChange` returns early unless the
  component's `attender` is the local user, so exactly one of the two windows owns
  the session - but *both* call `releaseCamera` on camera-off, deliberately, to
  force the device free after a swap.
- **The preview profile is hard-coded.** `camera.Profile` is built from
  `CAMERA_FORMAT_YUV_420_SP` and fixed resolution constants rather than selected
  from `getSupportedOutputCapability`, so it does not adapt to device capability.
- **`HomePage` renders nothing.** It exists solely to create the sub-windows; all
  visible content lives in `ModeratorPage` and `GuestPage`.

## Pitfalls

- **`releaseCamera` awaiting none of its four release calls is incorrect.** The
  method is `async` and its callers `await` it, but it resolves as soon as the
  releases are scheduled - so `createRecorder` opens a new camera input while the
  previous session is still tearing down. Await each one. (HW-05-0097)
- **Pushing every surfaceId into shared state with no removal is incorrect.**
  `aboutToDisappear` releases the camera but leaves the id registered, so a
  reloaded window leaves a dead surface in the array and the next session builds a
  preview output against it. Splice it out on teardown. (HW-05-0098)
- **Creating two sub-windows without ever destroying them is incorrect** -
  `destroyWindow` exists precisely for application child windows - and the
  `resize` / `moveWindowTo` / `showWindow` promises around them are discarded, so
  a window that fails to size or show does so silently. (HW-05-0099)
- **Ignoring `err` in the `getAllDisplays` callback and indexing `data[0]` is
  incorrect.** On failure the TypeError inside the callback means neither
  sub-window is created and the meeting screen stays blank; the two window
  callbacks in the same method already show the right guard. (HW-05-0100)
- **The document's step-1 snippet does not parse and drops the ordering rules.**
  Both callbacks are closed with `};`, both bind the same `subWindow` name, and
  the `await showWindow()` / `setRaiseByClickEnabled(false)` sequence - which the
  sample comments as mandatory - is absent. (HW-05-0101)
- **Subscribing to `avoidAreaChange` without a matching `off` is incorrect** -
  release it in `onWindowStageDestroy`. (HW-05-0102)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `destroyWindow9+` ("Destroys the current window ... takes effect for a system
  window, an application child window, ..."), `showWindow`, `resize`,
  `moveWindowTo`, `setRaiseByClickEnabled`, `getWindowAvoidArea` and
  `on`/`off('avoidAreaChange')`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-windowstage.md` -
  `createSubWindow`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-windowstage
- `documentation/harmonyos-references/02_application-framework/js-apis-display.md` -
  `getAllDisplays` and `getDefaultDisplaySync`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-display
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` -
  `requestPermissionsFromUser` and `requestPermissionOnSetting12+` with its
  mandatory `Context`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-references/02_application-framework/js-apis-permissionrequestresult.md` -
  `authResults`, checked correctly by this sample's permission ladder.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-permissionrequestresult
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` -
  `getPreferencesSync`, `hasSync`, `putSync`, `flush`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `usedScene`
  and `when` for user_grant permissions.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
