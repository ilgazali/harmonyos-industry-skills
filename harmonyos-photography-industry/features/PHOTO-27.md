---
id: PHOTO-27
title: Camera preview across background switches - the same release/rebuild pair on three page models
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/27_photo-v1_2-ts_12.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/photo-v1_2-ts_12-0000002298869489
sample: huawei_industry_tree/18_photography/downloads/CustomCamera.zip
kits: ["@kit.AbilityKit", "@kit.ArkGraphics2D", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CameraKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [abilityAccessCtrl, bundleManager, camera, colorSpaceManager, common, display, emitter, hilog, window]
permissions: [ohos.permission.CAMERA]
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0076, HW-18-0077, HW-18-0078, HW-18-0079, HW-18-0010, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when a page **holds an exclusive system resource that must be
given up when the app goes to the background and taken again on return** - the
camera being the canonical case, but the same shape covers the microphone, a
sensor subscription, or a hardware codec.

The rule the OS imposes is simple: the camera is exclusive, and an app that
keeps a session while backgrounded either blocks other apps or is torn down
under it. So the page must release on the way out and rebuild on the way back
in. The interesting part is *which callback pair* tells you that, and the
answer depends on how the page is mounted. This sample demonstrates the three
cases side by side, which is its real value:

- an `@Entry` page uses `onPageShow` / `onPageHide`;
- a page inside a `Navigation` stack uses `NavDestination`'s `onShown` /
  `onHidden`;
- a component that is neither can subscribe to `emitter` events sent from the
  ability's `onForeground` / `onBackground`.

**Read `HW-18-0076` first.** The preview in this sample never starts on any of
the three pages - a module-level `context` that is declared and never assigned
is passed to `camera.getCameraManager()`. The lifecycle structure is worth
copying; the code as shipped does not run.

## Feature checklist

- A home list with three rows: Entry scenario, Navigation scenario, Emitter
  scenario.
- Tapping a row asks for `ohos.permission.CAMERA` on first use, and does
  nothing if it is refused.
- Each scenario shows a full-bleed camera preview on a black background with a
  shutter graphic and a way back.
- Sending the app to the background releases the camera session; returning
  rebuilds it and the preview resumes.
- Leaving a scenario page releases the camera and restores the light status
  bar.
- The status bar switches to white-on-black inside a camera page and back to
  black-on-grey outside it.

## Architecture

One `entry` module, three pages that differ only in their lifecycle wiring, and
three utilities.

```
entry/src/main/ets
├── common/Constants.ets            layout numbers, colours and the row titles
├── entryability/EntryAbility.ets   full screen + avoid areas -> AppStorage; emits events 1 and 2
├── entrybackupability/
├── pages
│   ├── MainPage.ets                @Entry, the list + the Entry-scenario camera (onPageShow/onPageHide)
│   ├── NavigationPage.ets          NavDestination, onShown/onHidden
│   └── EmitterPage.ets             NavDestination, emitter.on(1)/emitter.on(2)
└── utils
    ├── CameraManager.ets           cameraCreate / releaseCamera over module-scope session state
    ├── PermissionsManager.ets      check, request, and the settings fallback
    └── StatusBarManager.ets        status-bar colour via a JSON action string
```

The documented tree matches the zip.

**The design decision worth copying** is that all three pages call the *same*
`cameraCreate(surfaceId, context)` and `releaseCamera()`. The page model only
decides *when*; the pipeline is one implementation. That is what makes the
sample readable as a comparison, and it is the right factoring for a real app
too - a camera helper should not know whether it is being driven by a router
page, a `NavDestination`, or an ability callback.

**The structure worth avoiding** is the module-level `let context: Context;`
repeated at the top of all three page files. It is never assigned in any of
them (`HW-18-0076`), while each component separately holds a perfectly good
`this.getUIContext().getHostContext()`. `CameraManager.cameraCreate` even opens
with `context = context;` - a self-assignment of its own parameter, which is
the fossil of the intended `currentContext = context`. Pass the context as an
argument from the component, or store it on the component; a file-scope
mutable context has no owner and no initialisation point.

## Implementation steps

1. **Declare `ohos.permission.CAMERA`** in `module.json5` with `reason` and
   `usedScene`.
2. **Ask before entering a scenario, and await the answer** - a
   fire-and-forget request followed by an immediate check always reads the
   pre-dialog state (`HW-18-0077`).
3. **Take the surface id in `XComponent.onLoad`** and build the session there,
   passing a real context (`HW-18-0076`).
4. **For an `@Entry` page**, rebuild in `onPageShow` and release in
   `onPageHide`, guarded by a flag so the list state does not touch the camera.
5. **For a `Navigation` page**, use `NavDestination`'s `onShown` / `onHidden` -
   `onPageShow` does not fire for a `NavDestination`.
6. **For anything else**, emit from the ability's `onForeground` /
   `onBackground` and subscribe with `emitter.on` - and unsubscribe in
   `aboutToDisappear` (`HW-18-0078`).
7. **Release on leaving the page as well as on backgrounding**; the emitter
   pair alone does not cover a pop (`HW-18-0079`).
8. **Await the teardown before the rebuild** and null the session fields
   (`HW-18-0010`).

## Verified snippets

All snippets are from `CustomCamera.zip`. Corrected forms are marked.

**The context that is never assigned — `entry/src/main/ets/pages/MainPage.ets`** (corrected, see `HW-18-0076`)

```typescript
// let context: Context;                         // FIX: shipped module-level, never assigned

@Entry
@Component
struct Index {
  @State context: Context | undefined = this.getUIContext().getHostContext();
  @State surfaceId: string = '';
  @State isCamera: boolean = false;
  private xcomponentController: XComponentController = new XComponentController;

  async onPageShow() {
    if (this.isCamera === true) {
      cameraCreate(this.surfaceId, this.context!);   // FIX: was the file-scope `context`
    }
  }

  async onPageHide() {
    if (this.isCamera === true) {
      releaseCamera();
    }
  }
}

// in build(), inside the camera Column:
XComponent({ id: 'xcomponent', type: XComponentType.SURFACE, controller: this.xcomponentController })
  .onLoad(() => {
    this.xcomponentController.setXComponentSurfaceRect({
      surfaceWidth: this.screenHeight / Constants.SURFACE_RATIO,
      surfaceHeight: this.screenHeight
    });
    this.surfaceId = this.xcomponentController.getXComponentSurfaceId();
    cameraCreate(this.surfaceId, this.context!);     // FIX: same substitution
  })
```

**`getCameraManager(undefined)` throws, and the throw is swallowed.** All three
pages declare their own `let context: Context;` at file scope, pass it into
`cameraCreate`, and never write to it. `cameraCreate` wraps its whole body in a
`try/catch` that only logs, so the failure surfaces as a black `XComponent` and
one console line - which is why the defect survived into the published sample.
The component's own `this.context` is present in `MainPage` and used for the
permission request, three lines away from the broken call.

Note the `isCamera` guard on both lifecycle callbacks. `MainPage` is both the
list and the Entry-scenario camera, so without it every return to the list
would rebuild a camera session for a hidden surface.

**The three lifecycle pairs — `NavigationPage.ets` and `EmitterPage.ets`** (as shipped)

```typescript
// NavigationPage.ets - a NavDestination gets onShown/onHidden, not onPageShow/onPageHide
NavDestination() { /* TabView + XComponent */ }
  .hideTitleBar(true)
  .padding({
    top: this.getUIContext().px2vp(this.topRectHeight),
    bottom: this.getUIContext().px2vp(this.bottomRectHeight)
  })
  .onShown(() => {
    cameraCreate(this.surfaceId, context);
    StatusBarManager.handleStatusBarAction(this.getUIContext().getHostContext()!,
      JSON.stringify({ 'action': 'backgroundColorByHexString', 'args': ['#000000', '#FFFFFF'] }));
  })
  .onHidden(() => {
    releaseCamera();
    StatusBarManager.handleStatusBarAction(this.getUIContext().getHostContext()!,
      JSON.stringify({ 'action': 'backgroundColorByHexString', 'args': ['#F1F3F5', '#000000'] }));
  });

// EntryAbility.ets - the ability announces its own foreground/background
onForeground(): void {
  let event: emitter.InnerEvent = { eventId: 1, priority: emitter.EventPriority.LOW };
  emitter.emit(event);
}

onBackground(): void {
  let event: emitter.InnerEvent = { eventId: 2, priority: emitter.EventPriority.LOW };
  emitter.emit(event);
}
```

**`onShown`/`onHidden` versus `onPageShow`/`onPageHide` is the point of the
second variant.** A `NavDestination` is not a page in the router sense, so the
`@Entry`-level callbacks never fire for it; `NavDestination` publishes its own
pair, and they fire both on a stack push/pop and on an app foreground/background
transition. That makes the Navigation case strictly simpler than the Entry case
- no `isCamera` guard is needed because the destination only exists while it is
on screen.

The emitter variant exists for the case where neither applies: a component
mounted somewhere with no lifecycle of its own. `emitter.emit` from the
ability is a broadcast, so any number of components can react, and the numeric
`eventId` is the whole contract - which is also its weakness, since nothing
ties event `1` to "foreground" except the two call sites.

**Subscribing and unsubscribing — `entry/src/main/ets/pages/EmitterPage.ets`** (corrected, see `HW-18-0078`, `HW-18-0079`)

```typescript
private event1: emitter.InnerEvent = { eventId: 1 };
private event2: emitter.InnerEvent = { eventId: 2 };
private foregroundCallback = (): void => {
  cameraCreate(this.surfaceId, this.getUIContext().getHostContext()!);
};
private backgroundCallback = (): void => {
  releaseCamera();
};

aboutToAppear() {
  emitter.on(this.event1, this.foregroundCallback);
  emitter.on(this.event2, this.backgroundCallback);
}

aboutToDisappear() {
  // FIX: absent in the sample - dead pages keep answering every foreground event
  emitter.off(this.event1.eventId, this.foregroundCallback);
  emitter.off(this.event2.eventId, this.backgroundCallback);
  releaseCamera();                                // FIX: the pop path released nothing
}
```

**A subscription outlives the component unless you remove it.** The shipped
page declares both callbacks as locals inside `aboutToAppear` and subscribes
them, and there is no `aboutToDisappear` anywhere in the file. Every visit adds
a pair, so after N visits a single app-foreground event runs `cameraCreate` N
times, each with a stale `surfaceId` from a destroyed `XComponent`
(`HW-18-0078`). Hoisting the callbacks to fields is what makes `emitter.off`
possible at all - `off` needs the same function reference.

The second correction covers the other exit. `EmitterPage.onHidden` only
restyles the status bar, so popping back to the list leaves the session running
against a surface that is gone; the only teardown in the file is the
background event (`HW-18-0079`). Releasing in `aboutToDisappear` closes that
path.

**The pre-dialog check — `entry/src/main/ets/pages/MainPage.ets`** (corrected, see `HW-18-0077`)

```typescript
.onClick(async () => {
  let permissions: Array<Permissions> = this.permissions;
  if (this.context) {
    await commonRequestPermissions(permissions, this.context);   // FIX: shipped call is not awaited
  }
  let permissionAllowed = await checkPermissions(permissions);
  if (!permissionAllowed) {
    return;
  }
  if (index === 1) {
    this.isCamera = true;
    cameraCreate(this.surfaceId, this.context!);
    // ...
  }
})
```

**Without the `await`, the check runs while the dialog is still open.**
`commonRequestPermissions` is `async`: it checks, requests, and falls back to
`requestPermissionOnSetting` when no dialog appeared. Dropping its promise
means `checkPermissions` reads the token state from before the user has
answered, returns `false`, and the handler bails - so the first tap on any of
the three rows always does nothing and only the second works. The helper itself
is well built and worth reusing as-is; only the call site is wrong.

`PermissionsManager.checkAccessToken` is also the clean version of a check we
have seen go wrong elsewhere: `tokenId` and `grantStatus` are initialised
before the `try`, so a failing `getBundleInfoForSelf` returns
`PERMISSION_DENIED` rather than `undefined`.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.CAMERA",
    "reason": "$string:reason_camera",
    "usedScene": {
      "abilities": ["EntryAbility"],
      "when": "always"
    }
  }
]
```

- `CAMERA` is `user_grant`, so `reason` and `usedScene` are mandatory and the
  reason string must exist in `resources/base/element/string.json`.
- `when: "always"` is declared, which is the opposite of what this sample
  demonstrates: the whole feature is about *not* holding the camera in the
  background. `inuse` is the correct value here.
- `routerMap` is declared as `$profile:route_map`, which is how
  `pushPathByName('navigationPage')` and `pushPathByName('emitterPage')` resolve
  to the two builder functions.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- The preview profile is chosen by an exact match on 1920x1080; if the device
  offers no such preview profile, `cameraCreate` logs `Preview profile not
  found!` and returns with no preview. There is no fallback to the nearest
  profile - compare `PHOTO-25`'s `PreviewUtil`, which searches by ratio and
  proximity.
- The session sets `colorSpaceManager.ColorSpace.DISPLAY_P3` unconditionally
  rather than querying supported colour spaces.
- The surface rectangle is derived from `display.getDefaultDisplaySync().height`
  divided by a constant ratio, so it is fixed at page construction and does not
  follow a window resize or a fold.
- The shutter graphic is decorative: `EmitterPage`'s `onClick` body is empty and
  the other two pages have no handler at all. This sample captures nothing; for
  capture see `PHOTO-13` or `PHOTO-26`.
- `EntryAbility` registers `avoidAreaChange` and never removes it.

## Pitfalls

- **`HW-18-0076`** (B/high, confirmed): the module-level `let context: Context;`
  in `MainPage`, `NavigationPage` and `EmitterPage` is never assigned, so every
  camera entry point calls `camera.getCameraManager(undefined)`; the throw is
  swallowed by `cameraCreate`'s catch and the preview never starts in any of
  the three scenarios. Fix: pass the component's
  `this.getUIContext().getHostContext()`.
- **`HW-18-0077`** (B/medium, confirmed): `commonRequestPermissions` is not
  awaited, so `checkPermissions` races the dialog and the first tap on a
  scenario row always returns early. Fix: `await` the request, or branch on its
  `authResults`.
- **`HW-18-0078`** (B/medium, confirmed): `EmitterPage` subscribes two callbacks
  in `aboutToAppear` and never removes them; after N visits every app
  foreground runs `cameraCreate` N times with stale surface ids. Fix: hoist the
  callbacks to fields and `emitter.off` them in `aboutToDisappear`.
- **`HW-18-0079`** (B/medium, confirmed): leaving `EmitterPage` releases
  nothing - `onHidden` only restyles the status bar - so the session survives
  the pop. Fix: call `releaseCamera` in `aboutToDisappear`.
- **`HW-18-0010`** (B/medium, confirmed): `releaseCamera` calls `stop`, `close`
  and `release` without awaiting any of them and never nulls the module fields,
  so a rebuild races the teardown and the `if (photoSession)` guards stay
  truthy on released objects. Nine samples in this industry share it. Fix:
  await each call in order, null the fields, and await the helper at its call
  sites.

## References

- `documentation/harmonyos-references/02_application-framework/ts-universal-entry.md` - what `@Entry` provides
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-entry
- `documentation/harmonyos-references/02_application-framework/ts-custom-component-lifecycle.md` - `onPageShow`, `onPageHide`, `aboutToAppear`, `aboutToDisappear`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-custom-component-lifecycle
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `NavPathStack` and `pushPathByName`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-references/02_application-framework/ui-design-hdsnavdestination.md` - `onShown` and `onHidden`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ui-design-hdsnavdestination
- `documentation/harmonyos-references/03_system/js-apis-emitter.md` - `emitter.on`, `emitter.off`, `InnerEvent`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-emitter
- `documentation/harmonyos-guides/03_application-framework/uiability-lifecycle.md` - `onForeground` and `onBackground`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/uiability-lifecycle
- `documentation/harmonyos-guides/05_media/camera-overview.md` - the camera session lifecycle and its exclusivity
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-overview
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` - `requestPermissionsFromUser`, `requestPermissionOnSetting`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `PHOTO-25` - a preview profile chosen by search rather than an exact size match
- `PHOTO-26` - the same `cameraCreate`/`releaseCamera` pair driven by a resolution toggle
