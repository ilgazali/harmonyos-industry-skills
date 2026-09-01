---
id: PHOTO-01
title: Photography app architecture - seven feature HARs behind one Navigation, wired by system routing tables
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/01_practice-photo-app-architecture-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1-0000002077325929
sample: huawei_industry_tree/18_photography/downloads/Picture.zip
kits: ["@kit.CameraKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.ArkGraphics2D", "@kit.ArkUI", "@kit.AbilityKit", "@kit.CoreFileKit", "@kit.ArkData", "@kit.PaymentKit", "@kit.PreviewKit", "@kit.SensorServiceKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [NavPathStack, Navigation, NavDestination, routerMap, pushPathByName, pushPath, getParamByName, "camera.getCameraManager", createPreviewOutput, createPhotoOutput, createSession, PhotoCaptureSetting, photoAssetAvailable, CanvasRenderingContext2D, drawImage, transform, fillText, PinchGesture, PanGesture, onDrop, draggable, "image.createImageSource", createPixelMapSync, readPixelsSync, writePixels, clonePixelMap, "effectKit.createEffect", setColorMatrix, SaveButton, setWindowPrivacyMode, "abilityAccessCtrl.checkAccessToken", requestPermissionOnSetting]
permissions: ["ohos.permission.CAMERA", "ohos.permission.MICROPHONE", "ohos.permission.PRIVACY_WINDOW"]
min_api: 20
modules: [entry, base (har), home (har), Shoot (har), crop (har), stitch (har), Mine (har), vip (har)]
findings: [HW-18-0005, HW-18-0009, HW-18-0010, HW-18-0011, HW-18-0012, HW-18-0013, HW-18-0014, HW-18-0015, HW-18-0016, HW-18-0017, HW-18-0018, HW-18-0019, HW-18-0020, HW-18-0021, HW-18-0022, HW-18-0023, HW-18-0030, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when you are **laying out a multi-feature media app** and have
to decide where the module boundaries go, how pages in different modules reach
each other, and who owns the camera. It is the architecture reference for the
whole 18_photography pack: every scenario card from `PHOTO-03` on is one
feature of this app extracted into a standalone project.

The shape it teaches is three layers - one `entry` HAP that holds only the tab
shell and navigation stack, six feature HARs that each own one page tree and
its own routing table, and one `base` HAR of shared constants and image
utilities. Features never import each other. They are reached by **name**
through the system routing table, so the HAP does not import them at build time
either; only `home` and `Mine` are imported directly, because they render
inside the tabs.

That generalises well beyond photography. Any app with a home grid of
independent features - a bank, a utilities app, a super-app - wants exactly
this: features that can be added, removed or lazily loaded without touching the
shell. The two things to take from this sample specifically are the **routing
table wiring** and the **module-scoped camera singleton**. The two things to be
careful with are the Canvas editor (its transform handling is wrong in three
distinct ways) and `JoinImages` (its declared signature is a lie).

Because it is a framework demo, the sample ships an unusual density of defects
- fifteen findings anchored here plus two industry systematics. Read the
Pitfalls before copying any one file.

## Feature checklist

- A two-tab shell (首页 / 我的) inside a `Navigation`, with the tab bar drawn by
  a custom builder and swipe-to-change disabled.
- A home page with a hero banner and a 4-up grid of feature entries: 图片精修
  (retouch), 拼图 (stitch), 照片裁剪 (crop), 会员尊享 (VIP), plus a full-width
  拍摄照片 (take photo) button.
- Grid entries that need images open the system picker first and push their
  destination only once enough images were chosen.
- A camera page with pinch zoom, discrete 0.5x/1x/5x/10x zoom buttons, a
  four-mode flash menu, live-photo toggle, front/back flip, a device-rotating
  thumbnail, and a tap-through to the system Photos app.
- A crop page: canvas preview with a diagonal watermark, pinch to scale, pan to
  move, rotate 90 degrees, a one-inch crop preset, and reset. In "edit" mode the
  same page instead shows a filter strip (原图 / 粉色 / 灰度 / 高亮) rendered as
  thumbnails of the real image, and sets the window to privacy mode so
  screenshots are blocked.
- A stitch page holding two images side by side; dragging one onto the other
  swaps them, and save crops them to a common height and joins them.
- An order-confirmation page previewing the result with a `SaveButton`, a VIP
  page backed by Payment Kit, and a profile page with a mock order list.

## Architecture

Eight modules: one entry HAP, seven HARs. `common/base` is the bottom layer,
the six `feature/*` HARs sit on it, and `entry` sits on all of them.

```
Picture/
├── entry/                                entry HAP - shell only
│   └── src/main/ets
│       ├── entryability/EntryAbility.ets full-screen; ImageUtils.context = this.context
│       └── pages/Index.ets               @Entry: Navigation + 2 Tabs, owns @Provide pageRouter
├── common/base/                          har - the bottom layer
│   ├── Index.ets                         barrel: what the rest of the app may import
│   └── src/main/ets
│       ├── constants/{CommonConstants,ImageConstants,BreakpointConstants,RouteMap}.ets
│       └── utils
│           ├── ImageUtils.ets            uri -> buffer, clonePixelMap, static context
│           ├── JoinImages.ets            two-image pixel-buffer stitch  (see HW-18-0016)
│           ├── BreakpointType.ets
│           └── Logger.ets
└── feature/
    ├── home/                             har - HomePage, Banner, ModuleEntry, Shoot button
    ├── Shoot/                            har - ShootPage + CameraShooter + PermissionsRequest
    │   └── src/main/module.json5          declares CAMERA + MICROPHONE
    ├── crop/                             har - CropPage (crop AND filter) + OrderConfirmPage
    │   └── src/main/module.json5          declares PRIVACY_WINDOW  (see HW-18-0019)
    ├── stitch/                           har - StitchPage + DragListItem
    ├── Mine/                             har - MinePage + OrderPage
    └── vip/                              har - VipPurchase (Payment Kit)

entry/libs/image_operation.har            prebuilt binary HAR: cropImage, rotateImage,
                                          getPixelMapByUri, selectImages,
                                          savePixelMapToGalleryBySaveButton
```

Each feature HAR carries `"routerMap": "$profile:route_map"` in its
`module.json5` and a `resources/base/profile/route_map.json` naming its pages.
The documented tree in the guide covers the `.ets` sources faithfully but omits
all of that - the barrel `Index.ets` files, the six `route_map.json` files, the
per-module `module.json5` files and the prebuilt `image_operation.har` - which
is exactly the wiring a reader needs. The guide also says 6 HARs; there are 7.

**The design decision worth copying** is that navigation is by **string name
resolved at runtime**, not by import. `base` exports a `RouteMap` enum of
names; each feature registers those names in its own `route_map.json` against a
`@Builder` function; the shell pushes a name onto the shared `NavPathStack`:

```typescript
// common/base — the shared vocabulary, no code dependency in either direction
export enum RouteMap {
  VIP_PURCHASE = 'VipPurchase',
  PIC_CROP = 'CropPage',
  PIC_STITCH = 'StitchPage',
  SHOOT_PAGE = 'ShootPage',
  ORDER_PAGE = 'OrderConfirmPage',
  HOME_PAGE = 'HomePage',
  MINE_PAGE = 'MinePage'
}
```

`home` therefore pushes `CropPage` without importing `crop`, and `crop` pushes
`OrderConfirmPage` without importing itself by path. The `NavPathStack` is
published once by the shell as `@Provide pageRouter` and every page picks it up
with `@Consume('pageRouter')` - one line per page, no prop drilling through
module boundaries.

The cost is that a typo in a route name is a runtime failure, not a build
failure, and that parameters cross the boundary as untyped `Object` (each page
re-asserts its own `RouterParams` interface on arrival). For a feature-modular
app that is the standard trade, and it is what makes the HARs genuinely
independent.

## Implementation steps

1. **Put the shell in `entry` and nothing else.** `Index.ets` is 47 lines: a
   `Navigation(this.pageRouter)` wrapping `Tabs`, two `TabContent`s, and a
   `tabBuilder`. All business pages live in HARs.
2. **Publish the stack once**: `@Provide pageRouter: NavPathStack = new NavPathStack()`
   in the shell, `@Consume('pageRouter')` in every page.
3. **Give each feature HAR a routing table.** `"routerMap": "$profile:route_map"`
   in its `module.json5`, and one entry per page in `route_map.json` naming
   `pageSourceFile` and a `buildFunction`.
4. **Export a `@Builder` per page** with the exact name the table gives -
   `export function CropPageBuilder(name: string, param: Object) { CropPage(); }`.
5. **Keep the route names in `base`** as an enum so pusher and pushee agree
   without a code dependency.
6. **Gather the images before you navigate.** `ModuleEntry.handleJump` awaits
   the picker and pushes only when `uri.length >= item.selectImgNum`, so no
   page ever opens with nothing to edit.
7. **Own the camera at module scope**, not on a component - see `PHOTO-04` for
   the same file with ratio switching. **Await the teardown before rebuilding**
   (`HW-18-0010`).
8. **Request CAMERA and wait for the answer** before opening the camera;
   `NavDestination.onShown` does not re-fire after the dialog is answered
   (`HW-18-0009`, `HW-18-0030`).
9. **Guard the profile `find()` results** before passing them into
   `createPreviewOutput` / `createPhotoOutput` (`HW-18-0021`).
10. **Assign the returned `zoomRatioRange` on every `cameraShooting` call**,
    including the camera flip, or the clamp uses the previous camera's range
    (`HW-18-0023`).
11. **On the Canvas, use `setTransform` per frame, not `transform`** - the
    latter concatenates onto a matrix nothing resets (`HW-18-0012`) - and treat
    `0` as a valid pan offset (`HW-18-0017`).
12. **Swap the dimension metadata whenever you swap or rotate the bitmaps it
    describes** (`HW-18-0011`, `HW-18-0013`), and **release every `ImageSource`
    and close every fd** (`HW-18-0005`, `HW-18-0022`).

## Verified snippets

All snippets are from `Picture.zip`. Corrected forms are marked.

**The shell and the routing tables** (as shipped)

```typescript
// entry/src/main/ets/pages/Index.ets
@Entry
@Component
struct Index {
  @Provide pageRouter: NavPathStack = new NavPathStack();
  @State currentIndex: number = 0;

  build() {
    Navigation(this.pageRouter) {
      Tabs({ barPosition: BarPosition.End }) {
        TabContent() { HomePage(); }
          .tabBar(this.tabBuilder('首页', 0, $r('app.media.home_selected'), $r('app.media.home')));
        TabContent() { MinePage(); }
          .tabBar(this.tabBuilder('我的', 1, $r('app.media.mine_selected'), $r('app.media.mine')));
      }
      .onChange((index: number) => { this.currentIndex = index; })
      .scrollable(false)
      .padding({ bottom: 24 });
    }
    .mode(NavigationMode.Stack)
    .hideTitleBar(true)
    .hideBackButton(true);
  }
}
```

```json5
// feature/crop/src/main/module.json5
{ "module": { "routerMap": "$profile:route_map", "name": "crop", "type": "har", ... } }
```

```json
// feature/crop/src/main/resources/base/profile/route_map.json
{ "routerMap": [
  { "name": "CropPage",         "pageSourceFile": "src/main/ets/view/CropPage.ets",
    "buildFunction": "CropPageBuilder" },
  { "name": "OrderConfirmPage", "pageSourceFile": "src/main/ets/view/OrderConfirmPage.ets",
    "buildFunction": "OrderConfirmPageBuilder" }
] }
```

```typescript
// feature/crop/src/main/ets/view/CropPage.ets — what the table points at
@Builder
export function CropPageBuilder(name: string, param: Object) {
  CropPage();
}
```

**`Tabs` inside `Navigation`, not the other way round.** The tab bar is the
root; pushing a destination onto the stack covers the tabs entirely
(`NavigationMode.Stack`), which is the behaviour a full-screen camera or editor
needs. Inverting the nesting would leave the tab bar visible over the
viewfinder.

**The three files above are the whole cross-module wiring.** Nothing in `entry`
or `home` imports `crop`; the loader resolves `'CropPage'` through the merged
routing tables at push time and calls `CropPageBuilder`. `.scrollable(false)`
is deliberate - the home page hosts horizontal drag and pan gestures that would
otherwise fight the tab swipe.

**Gathering inputs before navigating — `feature/home/src/main/view/common/ModuleEntry.ets`** (as shipped)

```typescript
async handleJump(item: ModuleEntryItem) {
  let uri: string[] = []
  if (item.needSelectImg) {
    uri = await selectImages(item.selectImgNum)
    if (uri && uri.length >= item.selectImgNum) {
      this.pageRouter.pushPath({
        name: item.routePath,
        param: {
          imageUri: item.selectImgNum > 1 ? uri : uri[0],   // string | string[] by count
          from: item.id                                     // 'crop' | 'edit' | 'stitch'
        } as RouterParams
      })
    }
  } else {
    this.pageRouter.pushPathByName(item.routePath, '')
  }
}
```

**This is the correct place for the picker** - before the push, not in the
destination's `aboutToAppear`. Cancelling leaves the user on the home grid with
nothing to undo, and the destination can assume its parameters are valid. It is
also the one photo-picker call site in this industry that handles cancellation
properly (compare `HW-18-0024` in `PHOTO-03`).

`from` is what lets one page serve two grid entries: 照片裁剪 and 图片精修 both
route to `CropPage`, which reads `from` and renders either `CropOperationBar`
or `EditOperationBar`. Two features, one page, one flag - reasonable here
because the two modes share the canvas, the loading and the watermark, and
differ only in the bottom bar.

The declared type is a lie in the other direction, though: `RouterParams.imageUri`
is `string`, and the stitch path assigns a `string[]` to it under an `as` cast
that the compiler cannot check. `StitchPage` then re-declares its own
`RouterParams` with `imageUri: string[]`. Two incompatible interfaces of the
same name across a module boundary is the price of untyped route parameters;
declare a union and narrow on `from` instead.

**The camera pipeline — `feature/Shoot/src/main/ets/utils/CameraShooter.ets`** (corrected, see `HW-18-0010`, `HW-18-0021`, `HW-18-0022`)

```typescript
let previewOutput: camera.PreviewOutput;
let cameraInput: camera.CameraInput;
let photoSession: camera.PhotoSession;
let photoOutPut: camera.PhotoOutput;
let currentContext: Context;

export async function cameraShooting(cameraPosition: number, surfaceId: string,
  context: Context): Promise<number[]> {
  currentContext = context;
  await releaseCamera();                       // FIX: sample calls releaseCamera() bare
  let cameraManager: camera.CameraManager = camera.getCameraManager(context);
  let cameraArray: camera.CameraDevice[] = cameraManager.getSupportedCameras();
  if (cameraArray.length <= 0) {
    return [];
  }
  cameraInput = cameraManager.createCameraInput(cameraArray[cameraPosition]);
  await cameraInput.open();
  let cameraOutputCap: camera.CameraOutputCapability =
    cameraManager.getSupportedOutputCapability(cameraArray[cameraPosition], camera.SceneMode.NORMAL_PHOTO);

  let previewProfile: undefined | camera.Profile = cameraOutputCap.previewProfiles.find((profile) => {
    let screen = display.getDefaultDisplaySync();
    if (screen.width <= 1080) {
      return profile.size.height === 1080 && profile.size.width === 1440;
    } else if (screen.width <= 1440 && screen.width > 1080) {
      return profile.size.height === 1440 && profile.size.width === 1920;
    }
    return profile.size.height <= screen.width && profile.size.height >= 1080 &&
      (profile.size.width / profile.size.height) < (screen.height / screen.width);
  });
  if (previewProfile === undefined) {
    previewProfile = cameraOutputCap.previewProfiles[0];      // FIX: sample guards AFTER the create
  }
  let photoProfile: undefined | camera.Profile = cameraOutputCap.photoProfiles.find((profile) =>
    profile.size.width <= 4096 && profile.size.width >= 2448);
  if (photoProfile === undefined) {
    photoProfile = cameraOutputCap.photoProfiles[0];          // FIX: same
  }

  previewOutput = cameraManager.createPreviewOutput(previewProfile, surfaceId);
  photoOutPut = cameraManager.createPhotoOutput(photoProfile);
  setPhotoOutputCb(photoOutPut);                              // register before commitConfig

  photoSession = cameraManager.createSession(camera.SceneMode.NORMAL_PHOTO) as camera.PhotoSession;
  photoSession.beginConfig();
  photoSession.addInput(cameraInput);
  photoSession.addOutput(previewOutput);
  photoSession.addOutput(photoOutPut);
  await photoSession.commitConfig();
  await photoSession.start();
  return photoSession.getZoomRatioRange();
}

export async function releaseCamera(): Promise<void> {
  if (photoSession) { await photoSession.stop(); }            // FIX: sample awaits none of these
  if (cameraInput) { await cameraInput.close(); }
  if (previewOutput) { await previewOutput.release(); }
  if (photoSession) { await photoSession.release(); }
  if (photoOutPut) { await photoOutPut.release(); }
  photoSession = undefined;                                   // FIX: sample leaves dangling handles
  cameraInput = undefined;
  previewOutput = undefined;
  photoOutPut = undefined;
}
```

**Module-level handles are the right model for a single hardware resource.**
There is one camera; a module-scoped `let` says so, while a component field
would suggest each page instance gets its own. It also lets `ShootPage` be
purely declarative: every control calls a free function, and the page never
holds a session it might leak on an unexpected teardown.

**The two corrections are what make it safe.** `releaseCamera()` called bare
returns immediately and `cameraInput.open()` on the next line races a sensor
still being handed back - intermittently "camera occupied" and a black preview
on flip or re-entry. And because the handles are never nulled, the truthiness
guards stay true on released objects, so a second `releaseCamera` double-frees.
The same shape recurs in nine samples across this industry (`HW-18-0010`); fix
it once, here, and copy the fixed version.

The profile `find`s are the sample's other structural weakness. The first two
branches match exact pixel sizes for two known phone classes; only the fallback
branch is a real predicate. On a device outside those classes `find` returns
`undefined`, and the shipped code passes it into `createPreviewOutput` and only
*then* checks `previewOutput === undefined` - by which point `7400101` has
already been thrown inside an async function with no catch (`HW-18-0021`).

**Canvas editing: watermark, pinch and pan — `feature/crop/src/main/ets/view/CropPage.ets`** (corrected, see `HW-18-0012`, `HW-18-0015`, `HW-18-0017`)

```typescript
setWaterMarkStyle() {
  this.waterMarkContext.beginPath();
  this.waterMarkContext.font = `${100 / this.imageScale}px 宋体`;   // FIX: shipped is `宋体 ${...}px}`
  this.waterMarkContext.textBaseline = 'top';
  this.waterMarkContext.fillStyle = '#80b2bec3';
  this.waterMarkContext.rotate(Math.PI / 180 * 30);
  this.waterMarkContext.fillText('xx相机专用水印\n保存图片后无水印',
    100 / this.imageScale, 100 / this.imageScale);
  this.waterMarkContext.rotate(-Math.PI / 180 * 30);               // undo the rotation, not the state
  this.waterMarkContext.closePath();
}

transformCanvasScale(scale: number, x: number, y: number) {
  this.waterMarkContext.clearRect(0, 0, this.canvasSize.width, this.canvasSize.height);
  this.waterMarkContext.setTransform(scale * this.imageScale, 0, 0,   // FIX: shipped uses transform()
    scale * this.imageScale, 0, 0);                                   //      which concatenates
  this.waterMarkContext.drawImage(this.pixelMap,
    this.getUIContext().px2vp(x), this.getUIContext().px2vp(y));
}

drawImage(offsetX?: number, offsetY?: number) {
  let point = this.getCenterPoint(this.canvasSize.width, this.canvasSize.height,
    this.imageWidthTrans, this.imageHeightTrans);
  if (offsetX !== undefined && offsetY !== undefined) {             // FIX: shipped is `if (offsetX && offsetY)`
    point.x += offsetX;
    point.y += offsetY;
  }
  this.waterMarkContext.clearRect(0, 0, this.canvasSize.width, this.canvasSize.height);
  this.waterMarkContext.drawImage(this.pixelMap,
    this.getUIContext().px2vp(point.x), this.getUIContext().px2vp(point.y));
  this.addWaterMark();
}
```

**Every line here is about the canvas matrix, and the sample loses track of it
three times.** `transform()` multiplies onto the current matrix; `setTransform()`
replaces it. `PinchGesture.onActionUpdate` fires many times per gesture with
`event.scale` cumulative *since the gesture started*, so multiplying by it each
frame turns a steady 1.1x pinch into 1.1^n, and the matrix is never reset
afterwards, so every subsequent `drawImage` inherits the pollution
(`HW-18-0012`). Note the base scale: `loadImage` already applied
`transform(this.imageScale, ...)` once to fit the image to the canvas, which is
why the corrected `setTransform` multiplies the two together rather than
replacing one with the other.

**`if (offsetX && offsetY)` is a falsy-zero bug in a gesture handler**, which is
the worst place for one: a pan straight down has `offsetX === 0` for whole
frames, both offsets are then dropped, and the image snaps back to centre
mid-drag (`HW-18-0017`).

**The font string is malformed** - `宋体 413px}` is neither valid CSS shorthand
(size must precede family) nor free of the stray brace, so the context keeps its
default font and the computed, scale-aware size never applies. The document
reproduces the same line verbatim (`HW-18-0015`).

The `rotate` / `-rotate` pair around `fillText` is the one thing this method
gets right: the diagonal watermark is drawn under a temporary rotation that is
immediately undone, so the image drawn afterwards is unaffected. `save()` /
`restore()` would express the same intent more robustly.

## Permissions & config

Permissions are declared **per feature HAR**, not centrally - each module states
what it needs and the packer merges them into the HAP:

```json5
// feature/Shoot/src/main/module.json5
"requestPermissions": [
  { "name": "ohos.permission.CAMERA",     "reason": "$string:reason_camera",
    "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.MICROPHONE", "reason": "$string:reason_microphone",
    "usedScene": { "abilities": ["EntryAbility"] } }
]

// feature/crop/src/main/module.json5
"requestPermissions": [ { "name": "ohos.permission.PRIVACY_WINDOW" } ]
```

- **Declaring permissions in the HAR that uses them is the right call** - the
  camera permission travels with the camera feature, and a build that drops
  `Shoot` drops the permission with it. `usedScene.abilities` still has to name
  the HAP's `EntryAbility`, since HARs have no abilities of their own. No
  `when` field is set anywhere; `"when": "inuse"` belongs on all three.
  `MICROPHONE` exists only for live photo, which records a short audio clip
  with the still.
- **`PRIVACY_WINDOW` is a `system_basic` permission** an ordinary app cannot
  hold without an ACL signing profile. `setWindowPrivacyMode(true)` is called in
  a `.then` with no `.catch`, so in a normally signed build it rejects with 201,
  the rejection is unhandled, and the advertised anti-screenshot feature
  silently does nothing (`HW-18-0019`).
- **The document's claim that the camera works 在无需申请相机权限下 (without
  requesting camera permission) is wrong for this sample** (`HW-18-0014`). That
  is true of `cameraPicker`, which hands the capture to a system UI. This sample
  drives Camera Kit directly and must hold `ohos.permission.CAMERA`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` and
  `targetSdkVersion` are both `6.0.0(20)`; `useNormalizedOHMUrl: true`.
- Feature HARs declare `deviceTypes: ["default", "tablet", "2in1"]` and the
  entry HAP `["phone", "tablet", "2in1"]`, but the camera geometry and the crop
  canvas are laid out for one portrait phone - the camera surface height is a
  hardcoded `1600` px against a component `SURFACE_HEIGHT` of 500 vp.
- **`image_operation.har` is a prebuilt binary** in `entry/libs`. `cropImage`,
  `rotateImage`, `getPixelMapByUri`, `selectImages` and
  `savePixelMapToGalleryBySaveButton` cannot be inspected or modified; the
  sample is not self-contained.
- Payment, login, order history and the banner content are mock data. The doc
  says so for login; it does not say so for the banner (`HW-18-0020`).
- The four-mode flash menu re-checks CAMERA on every single action, five
  near-identical 12-line closures inline in `build()`. Factor that into one
  helper before copying.

## Pitfalls

- **`HW-18-0009`** (B/high, probable): `requestPermissionsFromUser` is called
  fire-and-forget in `aboutToAppear` while `NavDestination.onShown` opens the
  camera immediately. On first run the grant dialog is still open when
  `cameraInput.open()` runs; it fails as an unhandled rejection, the preview
  stays black, and `onShown` does not re-fire after the answer. Fix: await the
  request, check `authResults`, then start, and re-run on grant.
- **`HW-18-0030`** (B/medium, confirmed): the same defect in the live-photo
  toggle - `requestPermissionsFromUser(..., microphonePermission).then(() => { this.isMovingPhoto = !this.isMovingPhoto; ... })`
  flips the feature on whether or not MICROPHONE was granted. Systematic across
  six samples. Fix: read `data.authResults` before acting.
- **`HW-18-0010`** (B/medium, confirmed): fire-and-forget camera teardown races
  the rebuild; handles are never nulled. Systematic across nine samples. Fix:
  await each call in order and reset the handles.
- **`HW-18-0021`** (B/medium, probable): hardcoded camera profiles -
  `find()` can return `undefined` and is passed straight into
  `createPreviewOutput`/`createPhotoOutput`, whose guards run too late.
  Systematic across four samples. Fix: guard before the create calls, fall back
  to a supported profile.
- **`HW-18-0023`** (B/low, confirmed): the camera-flip handler discards
  `cameraShooting`'s returned `zoomRatioRange`, so pinch zoom keeps clamping to
  the previous camera's range - a front camera clamped to the rear camera's 10x.
  Fix: assign the returned range at every call site.
- **`HW-18-0011`** (B/medium, confirmed): the stitch drag-swap exchanges the
  `PixelMap`s but not `leftInfo`/`rightInfo`, and `confirmOrder` crops using
  those stale dimensions. Two images of different heights, swapped, then saved,
  crop the wrong bitmap. Fix: swap the info objects with the pixel maps, or
  re-query `getImageInfo` before cropping.
- **`HW-18-0012`** (B/medium, probable): the crop pinch calls
  `transform(scale,...)` per frame on a matrix that is never reset, so a steady
  pinch compounds and the pollution persists into later draws. Fix:
  `setTransform`, or `resetTransform()` plus a single `transform`.
- **`HW-18-0013`** (B/medium, probable): rotating mutates the `PixelMap` in
  place but `imageWidth`/`imageHeight` (and the `*Trans` pair) are never
  exchanged, so `handleCropImage` computes the one-inch crop from transposed
  dimensions and can exceed the rotated bitmap. Fix: swap the dimension fields
  after each 90/270-degree rotation.
- **`HW-18-0015`** (B/low, confirmed): the watermark font string
  `` `宋体 ${100 / this.imageScale}px}` `` is malformed - stray brace, family
  before size - so the context keeps its default font and ignores `imageScale`.
  The doc ships the same line. Fix: `` `${100 / this.imageScale}px 宋体` ``.
- **`HW-18-0016`** (B/low, confirmed): `joinImages` declares
  `Array<string> | Array<image.PixelMap>` but casts every element to `PixelMap`,
  so the URI overload always throws into its own catch and returns `null`; a
  third image is silently dropped because the canvas is sized for exactly two;
  and two `ImageSource`s built from raw RGBA buffers are never used and never
  released. Fix: narrow the signature, size the canvas from all inputs, delete
  the dead sources.
- **`HW-18-0017`** (B/low, confirmed): `if (offsetX && offsetY)` drops a pan
  frame whose offset is exactly zero on either axis, snapping the image to
  centre mid-drag. Fix: compare against `undefined`.
- **`HW-18-0018`** (B/low, confirmed): 重置 (reset) does
  `clonePixelMap(this.rawPixelMap as PixelMap)` while `rawPixelMap` is still
  `undefined` during the async load - `getImageInfoSync()` on `undefined`
  crashes on an early tap. Fix: guard, or disable the button until loaded.
- **`HW-18-0019`** (B/low, probable): `PRIVACY_WINDOW` is `system_basic`;
  `setWindowPrivacyMode` rejects with 201 in a normally signed build and there
  is no `.catch`. Fix: catch it and document the ACL signing requirement.
- **`HW-18-0022`** (B/low, confirmed): `ImageUtils.getBufferByUri` opens, stats,
  reads and returns without closing the fd - one leaked descriptor per image
  load, for the process lifetime. Systematic across five samples. Fix: close in
  a `finally`.
- **`HW-18-0005`** (B/low, confirmed): `ImageSource`s created in `JoinImages`
  and `CropPage.loadImage` are never released, accumulating native decode
  resources. Systematic across eight files. Fix: `release()` after the pixel map
  is produced.
- **`HW-18-0014`** (E/medium, confirmed): the doc claims the camera needs no
  permission while the sample declares and requests `ohos.permission.CAMERA`.
  Fix: reword - the permission-free claim applies to `cameraPicker` only.
- **`HW-18-0020`** (D/low, confirmed): the doc promises a Banner 轮播图 carousel
  on the home page; `Banner.ets` loads `bannerData` into `imageArr` in
  `aboutToAppear` and then renders a single static `Image`. No `Swiper` exists.
  Fix: implement it or correct the doc.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - `Navigation`, `NavPathStack`, `NavDestination`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `documentation/harmonyos-guides/03_application-framework/arkts-set-navigation-routing.md` - the system routing table, `routerMap`, `buildFunction`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-set-navigation-routing
- `documentation/harmonyos-guides/01_getting-started/har-package.md` - HAR structure, the barrel `Index.ets`, cross-module dependencies
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/har-package
- `documentation/harmonyos-guides/02_media/camera-kit.md` - the Camera Kit model and its permission requirement
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-kit
- `documentation/harmonyos-references/04_media/js-apis-camera.md` - `Profile`, `CameraOutputCapability`, session lifecycle, `getZoomRatioRange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-camera
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - `Canvas`, `CanvasRenderingContext2D`, `transform` vs `setTransform`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pinchgesture.md` - `event.scale` is cumulative per gesture
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pinchgesture
- `documentation/harmonyos-references/02_application-framework/ts-universal-events-drag-drop.md` - `draggable`, `onDragStart`, `onDrop`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-events-drag-drop
- `documentation/harmonyos-guides/05_media/image-transformation.md` - `readPixelsSync` / `writePixels` and the stitch technique
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-transformation
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `setWindowPrivacyMode` and its permission
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` - `checkAccessToken`, `requestPermissionsFromUser`, `requestPermissionOnSetting`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `PHOTO-02` - the index of the 29 scenario samples that expand each feature of this app
- `PHOTO-03` - the filter strip as a standalone project
- `PHOTO-04` - the same `CameraShooter` module with aspect-ratio switching
