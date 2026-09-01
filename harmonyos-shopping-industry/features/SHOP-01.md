---
id: SHOP-01
title: Shopping mall app skeleton - one entry HAP over eight feature HARs, breakpoint columns from AppStorage, and a custom-UI barcode scanner
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/01_practice-purchase-app-architecture-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-purchase-app-architecture-v1-0000002077367333
sample: huawei_industry_tree/16_shopping/downloads/MultiShopping.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CameraKit", "@kit.CoreSpeechKit", "@kit.CryptoArchitectureKit", "@kit.IMEKit", "@kit.ImageKit", "@kit.LocalizationKit", "@kit.LocationKit", "@kit.MediaKit", "@kit.MediaLibraryKit", "@kit.PaymentKit", "@kit.PerformanceAnalysisKit", "@kit.ScanKit", "@kit.ShareKit", "@kit.VisionKit"]
apis: [abilityAccessCtrl, autoFillManager, base, camera, cameraPicker, common, cryptoFramework, customScan, detectBarcode, deviceInfo, display, fs, "customScan.init", "customScan.start", "customScan.stop", "customScan.release", "detectBarcode.decode", XComponent, XComponentController, "mediaquery.matchMediaSync", "List.lanes", LazyForEach, IDataSource, NavPathStack, routerMap, "media.createAVRecorder", getAudioCapturerMaxAmplitude, "photoAccessHelper.PhotoViewPicker", MediaAssetChangeRequest, "@StorageProp", "@Provide", "@Consume"]
permissions: [ohos.permission.CAMERA, ohos.permission.MICROPHONE, ohos.permission.APPROXIMATELY_LOCATION, ohos.permission.READ_IMAGEVIDEO, ohos.permission.WRITE_IMAGEVIDEO]
min_api: 20
modules: [phone (entry), common (har), camera (har), commoditydetail (har), conversation (har), home (har), orderdetail (har), personal (har), search (har), shopcart (har)]
findings: [HW-16-0001, HW-16-0002, HW-16-0003, HW-16-0013, HW-16-0026, HW-16-0027]
status: verified-with-fixes
---

## When to use

Load this card when you are **laying out a multi-module HarmonyOS app** and
need a worked reference for where the seams go: one thin entry HAP, a stack of
feature HARs that never import each other, and a single `common` HAR that
everybody depends on. It is the shopping-industry instance of the layered
practice, and the layering is the part that transfers - the same three tiers
serve a news reader, a banking app or a delivery client.

The sample is large (10,500 lines of ArkTS across ten modules) and most of it
is ordinary layout. Three things in it are load-bearing and worth copying:
**cross-HAR navigation through `routerMap` and one shared `NavPathStack`**,
which is what lets `home` push a page that lives in `camera` without importing
it; **one breakpoint listener writing into `AppStorage`**, which turns
responsive column counts into a one-line `@StorageProp` read anywhere in the
tree; and **the custom-UI `customScan` lifecycle** on an `XComponent` surface,
which is the only scanning API that lets you draw your own viewfinder.

Read `HW-16-0001` and `HW-16-0003` before lifting the media code. The voice
recorder writes into a file descriptor it has already closed, and the
photo-save path calls an API whose restricted permission is commented out of
the manifest.

## Feature checklist

- A splash page, then a four-tab shell: 首页 (home), 消息 (messages), 购物车
  (cart), 我的 (mine).
- The home tab carries a search bar with a vertically auto-playing suggestion
  swiper, a banner carousel, a category strip and a product feed.
- The product feed renders 2 columns at the sm breakpoint, 3 at md and 4 at lg;
  the tab bar itself moves from the bottom edge to the side at lg.
- A camera entry in the search bar opens a two-tab page: 扫码 (scan) and 识图
  (recognise).
- The scan tab draws a full-screen camera preview on an `XComponent` and
  reports barcodes and QR codes; an album button decodes a code out of a
  picked image instead. A result stops and releases the stream and opens a
  "similar products" sheet.
- Switching away from the scan tab releases the stream; switching back
  restarts it. The recognise tab runs a Camera Kit session instead.
- Product cards push a detail page, which lays out as one column at sm and as
  a 40/60 split at lg.
- The cart supports quantity edits and deletion; orders move through unpaid,
  delivered, received and evaluated.
- The message tab has a conversation list and a detail view with a
  press-to-talk voice button that records audio and shows a live amplitude
  ripple.

## Architecture

Ten modules: one entry HAP under `product/`, eight feature HARs under
`features/`, and `common/` at the root.

```
common/                              the shared HAR - no dependencies of its own
├── index.ets                        20 named exports: the module's entire public API
└── src/main/ets
    ├── components/                  CommodityList, CounterProduct, ShippingAddress,
    │                                AddressFormPage, EmptyComponent
    ├── constants/                   Breakpoint, Camera, Grid, Style
    ├── service/AlbumService.ets     PhotoViewPicker wrapper (no permission needed)
    ├── utils
    │   ├── BreakpointSystem.ets     3 media-query listeners -> AppStorage
    │   ├── CommonDataSource.ets     IDataSource for LazyForEach
    │   ├── LocalDataManager.ets     in-memory singleton standing in for a backend
    │   ├── LocationManager.ets
    │   └── Logger.ets
    └── viewmodel/                   Commodity, Order, Product, ShopData
features/
├── camera/       CameraPage (2 tabs) + CustomScan + CustomCamera + CameraService
├── commoditydetail/  the product detail page and its specification sheet
├── conversation/     the message list, the chat view, AudioRecorder + AVPlayer
├── home/             Home.ets - the home tab; depends on camera and search
├── orderdetail/      confirm, pay, order list, address block
├── personal/         the mine tab and its settings page
├── search/           SearchPage
└── shopcart/         ShopCart
product/phone/                       the only HAP
└── src/main/ets
    ├── entryability/EntryAbility.ets  portrait lock, full screen, avoid areas
    ├── pages/SplashPage.ets           the launch page (loadContent target)
    ├── pages/Index.ets                @Entry: Navigation + @Provide'd NavPathStack
    ├── pages/MainPage.ets             the 4 tabs; registers BreakpointSystem
    └── viewmodel/MainPageData.ets
```

The dependency graph is strictly acyclic and one level deep:

```
phone ──> orderdetail, commoditydetail, conversation, home, personal, shopcart
home  ──> camera, search
every module ──> common
common ──> nothing
```

**The documented tree broadly matches the zip** but drifts in three places:
`common/service` is shown as a directory named `AlbumService` when the zip has
`AlbumService.ets`; three feature modules are shown with a `constant`
directory where the zip has `constants`; and the doc's tree omits `index.ets` /
`Index.ets`, the export barrels that make the HARs importable at all. The
document's scan snippet is also brace-unbalanced (`HW-16-0013`).

**The design decision worth copying** is that feature HARs are wired together
by *route names*, not by imports. Each HAR declares a `routerMap` profile in
its own `module.json5`:

```json5
{ "module": { "name": "camera", "type": "har", "routerMap": "$profile:route_map" } }
```

and its `route_map.json` names a page and the exported `@Builder` that builds
it. `Index.ets` in the HAP creates one `NavPathStack` and `@Provide`s it as
`'pageInfo'`; every component down the tree `@Consume`s it and pushes by name:

```typescript
this.pageInfos.pushPath({ name: 'CameraPage' });        // Home.ets, no import of camera
this.pageInfos.pushPathByName('CommodityDetailPage', item.id);  // CommodityList.ets, in common
```

That is what lets `CommodityList` - which lives in `common`, the *bottom* of
the stack - push a page that lives in `commoditydetail`, a module above it,
without inverting the dependency. The page is resolved from the route table at
runtime and loaded lazily. It is the single most reusable idea in the sample.

The decision **worth avoiding** is `LocalDataManager`: a private-constructor
singleton holding arrays in memory, seeded from static `ShopData` literals.
Every module reaches for `LocalDataManager.instance()` directly, so the data
layer is a global with no interface. It is fine as a stand-in for a backend and
must be replaced by an injected repository before it grows.

## Implementation steps

1. **Put the HAP in `product/<device>` and the HARs in `features/`,** with the
   shared one at the root as `common`. Register all of them in the root
   `build-profile.json5`; only the HAP gets a `targets`/`applyToProducts`
   block.
2. **Give every HAR an export barrel** (`index.ets` / `Index.ets`) named in its
   `oh-package.json5` `main` field, and export from it only what other modules
   may use. `common/index.ets` exports exactly twenty symbols; that list is the
   module's API surface.
3. **Declare a `routerMap` in each HAR that owns pages**, with one entry per
   page naming its `pageSourceFile` and its exported `buildFunction`. Export
   the builder as a global `@Builder function XxxBuilder()`.
4. **Create one `NavPathStack` in the entry page and `@Provide` it**; navigate
   only by route name (`arkts-navigation-cross-package`). Never import a page
   component across modules.
5. **Register one `BreakpointSystem` in the shell page's `aboutToAppear` and
   unregister it in `aboutToDisappear`.** Three `matchMediaSync` listeners
   write one string into `AppStorage`; everything else reads it with
   `@StorageProp` and never touches `mediaquery` again.
6. **Drive the feed with `List(...).lanes(column)` over a `LazyForEach`**, not a
   `Grid` - the column count is a plain number derived from the breakpoint at
   the call site, and the `IDataSource` keeps only visible cards built.
7. **Declare each `user_grant` permission in the HAR that uses it**, not in the
   HAP - HAR manifests contribute their `requestPermissions` to the bundle,
   which keeps the permission next to the feature that justifies it - and
   request it in the same function that uses it.
8. **Drive scanner initialisation off the grant boolean with `@Watch`,** so the
   preview only starts once the permission exists.
9. **Follow the four-step customScan lifecycle**: get the `surfaceId` from the
   `XComponent`'s `onLoad`, size the preview, `init(options)`, then
   `start(viewControl)`. Release on tab change and on `aboutToDisappear` - a
   stream left running fails to reopen next time.
10. **Do not use the restricted album permissions to save a photo.** Use
    `SaveButton` or the authorization-dialog save flow instead (`HW-16-0003`).
11. **Keep the recording file descriptor open for the whole session** and close
    it after `release()` (`HW-16-0001`); clear the amplitude interval
    unconditionally before releasing the recorder (`HW-16-0002`).

## Verified snippets

All snippets are from `MultiShopping.zip`. Corrected forms are marked.

**One breakpoint listener for the whole app — `common/src/main/ets/utils/BreakpointSystem.ets`** (as shipped)

```typescript
import { mediaquery } from '@kit.ArkUI';

export class BreakpointSystem {
  private currentBreakpoint: string = '';
  private smListener?: mediaquery.MediaQueryListener;
  private mdListener?: mediaquery.MediaQueryListener;
  private lgListener?: mediaquery.MediaQueryListener;

  private updateCurrentBreakpoint(breakpoint: string) {
    if (this.currentBreakpoint !== breakpoint) {
      this.currentBreakpoint = breakpoint;
      AppStorage.set<string>(BreakpointConstants.CURRENT_BREAKPOINT, this.currentBreakpoint);
    }
  }

  private isBreakpointSM = (mediaQueryResult: mediaquery.MediaQueryResult) => {
    if (mediaQueryResult.matches) {
      this.updateCurrentBreakpoint(BreakpointConstants.BREAKPOINT_SM);
    }
  };
  // md and lg are identical with their own constants

  public register(uiContext: UIContext) {
    this.smListener = uiContext.getMediaQuery().matchMediaSync(BreakpointConstants.RANGE_SM);
    this.smListener.on('change', this.isBreakpointSM);
    this.mdListener = uiContext.getMediaQuery().matchMediaSync(BreakpointConstants.RANGE_MD);
    this.mdListener.on('change', this.isBreakpointMD);
    this.lgListener = uiContext.getMediaQuery().matchMediaSync(BreakpointConstants.RANGE_LG);
    this.lgListener.on('change', this.isBreakpointLG);
  }

  public unregister() {
    this.smListener?.off('change', this.isBreakpointSM);
    this.mdListener?.off('change', this.isBreakpointMD);
    this.lgListener?.off('change', this.isBreakpointLG);
  }
}
```

with the ranges as string constants -
`'(320vp<=width<520vp)'`, `'(520vp<=width<840vp)'`, `'(840vp<=width)'` - and
consumed anywhere in the tree as:

```typescript
@StorageProp('currentBreakpoint') currentBreakpoint: string = 'sm';

CommodityList({
  commodityList: $commodityList,
  column: this.currentBreakpoint === BreakpointConstants.BREAKPOINT_LG ? StyleConstants.DISPLAY_FOUR :
    (this.currentBreakpoint === BreakpointConstants.BREAKPOINT_MD ?
      StyleConstants.DISPLAY_THREE : StyleConstants.DISPLAY_TWO)
});
```

**Three properties make this worth copying.** The listeners are **arrow-function
fields**, not methods - that is what makes the same reference available to
`off()` in `unregister`, so the subscription actually detaches; a `this.isBreakpointSM.bind(this)`
at registration time would leak. The `updateCurrentBreakpoint` guard means
`AppStorage` is written only on an actual transition, so `@StorageProp`
consumers do not re-render on every media-query tick. And the value crosses
module boundaries as a **string in `AppStorage`**, which is why `common`,
`home` and `commoditydetail` can all be responsive without any of them owning
the listener or depending on each other.

The register/unregister pair sits in `MainPage.aboutToAppear` /
`aboutToDisappear` - once, in the shell. `BreakPointType<T>` in the same file
is the value-picking companion: `new BreakPointType({ sm: '100%', md: '50%',
lg: '40%' }).getValue(this.currentBreakpoint)` for widths that are not integer
column counts.

**The custom scanner lifecycle — `features/camera/src/main/ets/components/CustomScan.ets`** (as shipped)

```typescript
import { customScan, detectBarcode, scanBarcode, scanCore } from '@kit.ScanKit';

@Component
export struct CustomScan {
  @Watch('init') @Link reinit: boolean;
  @Watch('onPermit') @Prop userGrant: boolean;        // 是否已申请相机权限
  @Watch('onChangeTab') @Prop tabNumber: number;
  @State surfaceId: string = '';
  private mXComponentController: XComponentController = new XComponentController();

  init() {
    if (!this.userGrant) {
      return;                                          // no grant, no stream
    }
    let options: scanBarcode.ScanOptions = {
      scanTypes: [scanCore.ScanType.ALL],
      enableMultiMode: false,
      enableAlbum: true
    };
    this.setDisplay(this.getUIContext());              // step 2: size the preview
    try {
      customScan.init(options);                        // step 3: initialise
    } catch (error) {
      hilog.error(0x0001, TAG, `Failed to init customScan. Code: ${error.code}, message: ${error.message}`);
    }
    this.initCamera();
  }

  initCamera() {
    this.scanResult = [];
    let viewControl: customScan.ViewControl = {
      width: this.cameraWidth,
      height: this.cameraHeight,
      surfaceId: this.surfaceId
    };
    try {
      customScan.start(viewControl)                    // step 4: start, resolves on a hit
        .then(async (result: Array<scanBarcode.ScanResult>) => {
          if (result.length) {
            this.scanResult = result;
            await this.customScanStop();               // stop the stream before showing UI
            await this.customScanRelease();
            this.dialogController.open();
          }
        })
        .catch((err: BusinessError) => {
          hilog.error(0x0001, TAG, `Failed to start customScan. error: ${err}`);
        });
    } catch (error) {
      hilog.error(0x0001, TAG, `Failed to start customScan. Code: ${error.code}, message: ${error.message}`);
    }
  }

  // onChangeTab(): tabNumber === 0 ? initCamera() : (customScanStop(), customScanRelease())
  // 切换到其他tab时，需要停止并释放当前相机流，否则下次tab切换回来时会开启失败

  build() {
    Stack() {
      Column() {
        XComponent({ type: XComponentType.SURFACE, controller: this.mXComponentController })
          .onLoad(async () => {
            this.surfaceId = this.mXComponentController.getXComponentSurfaceId();
            this.init();                               // step 1: the surface exists
          })
          .width(this.cameraWidth)
          .height(this.cameraHeight)
          .position({ x: this.offsetX, y: this.offsetY });
      }
      .height('100%')
      .width('100%');
      // the viewfinder chrome and the album button draw over the surface
    }
    .width('100%')
    .height('100%');
  }
}
```

**The ordering is not stylistic, it is the API contract.** `customScan.start`
needs a `surfaceId`, and a surface only exists once the `XComponent` has been
laid out - which is why every step hangs off `onLoad` rather than
`aboutToAppear`. `start` returns a promise that resolves *when a code is
found*, not when the camera opens, so the `.then` is the result handler and
the `.catch` is a camera failure; both a `try/catch` and a `.catch` are needed
because the API throws synchronously on bad arguments and rejects
asynchronously on device errors.

**Three `@Watch`es replace three lifecycle hooks.** `userGrant` re-runs `init`
the moment the permission lands, `tabNumber` releases the stream on tab-out and
restarts it on tab-in, and `reinit` is a toggled boolean the parent flips from
`onShown` to force a restart after the page returns to the foreground. The
comment on `onChangeTab` states the reason plainly: a stream that is not
released fails to open next time.

`setDisplay` sizes the preview to a 16:9 rectangle centred on the screen -
`cameraHeight = max(w,h)`, `cameraWidth = that / (16/9)` - because the scan
preview must keep the sensor's aspect ratio or the decoder sees a distorted
image. The album path takes a different route entirely:
`AlbumService.selectPicture` uses `PhotoViewPicker`, which needs **no**
permission, and `detectBarcode.decode` reads the code out of the returned URI.

**The product feed — `common/src/main/ets/components/CommodityList.ets`** (as shipped)

```typescript
@Watch('onCommodityListChange') @Link commodityList: Commodity[];
@Prop column: number = 0;
@Consume('pageInfo') pageInfos: NavPathStack;
private data: CommonDataSource<Commodity> | undefined;

List({ space: StyleConstants.TWELVE_SPACE }) {
  LazyForEach(this.data, (item: Commodity) => {
    ListItem() {
      this.CommodityItem(item);
    }
    .onClick(() => {
      this.pageInfos.pushPathByName('CommodityDetailPage', item.id);
    });
  }, (item: Commodity) => JSON.stringify(item));
}
.lanes(this.column);

onCommodityListChange() {
  // !!批量修改lazyForEach数据源：数据源重新赋值 -> reload
  this.data?.batchUpdateData(this.commodityList);
}
```

**`lanes(n)` on a `List` is the cheap multi-column feed** - it keeps `List`'s
lazy recycling while laying items out in `n` columns, so the whole responsive
story is one integer arriving as a `@Prop`; a `Grid` would need column
templates and would give up the `IDataSource` recycling. The `@Watch` +
`batchUpdateData` pair is the piece people miss: `LazyForEach` observes the
data source's notifications, not the array, so re-assigning `commodityList`
alone changes nothing on screen. Note also that this component lives in
`common`, at the bottom of the module stack, yet pushes a page owned by
`commoditydetail` above it - the route-name indirection in action.

**The voice recorder — `features/conversation/src/main/ets/viewmodel/AudioRecorder.ets`** (corrected, see `HW-16-0001`, `HW-16-0002`)

```typescript
private avRecorder: media.AVRecorder | undefined = undefined;
private file: fs.File | undefined = undefined;          // FIX: keep the handle alive
public time: number = 0;
// avProfile packs media.ContainerFormatType.CFT_MPEG_4A - an m4a container

async startRecordingProcess(uiContext: UIContext): Promise<void> {
  this.avRecorder = await media.createAVRecorder();
  this.setAudioRecorderCallback();

  const context = uiContext.getHostContext();
  const filepath = context?.filesDir + '/01.m4a';       // FIX: was '01.mp3' under an m4a profile
  this.file = fs.openSync(filepath, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
  // FIX: the sample closes the file in a `finally` here, before prepare()
  this.avConfig.url = 'fd://' + this.file.fd;

  await this.avRecorder.prepare(this.avConfig);
  await this.avRecorder.start();

  this.time = setInterval(() => {
    if (!this.avRecorder) {
      return;                                            // FIX: guard a released recorder
    }
    this.avRecorder.getAudioCapturerMaxAmplitude((_: BusinessError, amplitude: number) => {
      this.maxAmplitude = amplitude;
    });
  }, Const.COLUMN_HEIGHT);
}

async stopRecordingProcess(): Promise<void> {
  clearInterval(this.time);                              // FIX: unconditional, was inside the state check
  if (this.avRecorder !== undefined) {
    if (this.avRecorder.state === 'started' || this.avRecorder.state === 'paused') {
      await this.avRecorder.stop();
    }
    await this.avRecorder.reset();
    await this.avRecorder.release();
    this.avRecorder = undefined;
  }
  if (this.file !== undefined) {
    fs.closeSync(this.file);                             // FIX: close after release, not before prepare
    this.file = undefined;
  }
}
```

**A recorder owns its descriptor for the whole session.** `AVRecorder` writes
into `avConfig.url = 'fd://<n>'` continuously from `start()` to `stop()`, so
the descriptor must outlive the recording. The shipped code opens the file and
closes it in a `finally` on the very next line, then hands
`file.fd` - now a stale integer - to `prepare()`, so the voice-message feature
records into nothing (`HW-16-0001`). The extension is wrong too: the profile
packs `CFT_MPEG_4A` while the path says `.mp3`.

**Interval cleanup must not be conditional on state.** The shipped
`clearInterval` sits inside the `started || paused` branch while `release()`
and `this.avRecorder = undefined` run unconditionally below it, so a stop
reached from any other state leaves a timer firing forever against
`this.avRecorder!` - a non-null assertion on a field that is now `undefined`
(`HW-16-0002`). Hoisting the `clearInterval` to the first line and adding the
early return inside the callback closes both halves.

## Permissions & config

Permissions are declared **per HAR**, next to the feature that needs them:

```json5
// features/camera/src/main/module.json5
"requestPermissions": [
  { "name": "ohos.permission.CAMERA", "reason": "$string:reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" } }
]

// features/conversation/src/main/module.json5 -> ohos.permission.MICROPHONE
// common/src/main/module.json5                -> ohos.permission.CAMERA (duplicate)
// product/phone/src/main/module.json5         -> ohos.permission.APPROXIMATELY_LOCATION
```

- HAR manifests contribute their `requestPermissions` to the built bundle, so
  this keeps each `user_grant` permission next to its justification. The cost
  is that `CAMERA` is declared twice - in `common` and in `camera` - which is
  harmless but hides the owner.
- Each of the three is requested at its point of use, all through
  `abilityAccessCtrl.requestPermissionsFromUser`: `CAMERA` in
  `CameraPage.init`, `MICROPHONE` in `ConversationDetailBottom` before the
  press-to-talk button arms, and `APPROXIMATELY_LOCATION` inside
  `common/utils/LocationManager.getCurrentLocation`, which only calls
  `geoLocationManager.getCurrentLocation` when `authResults[0] === 0`. That
  last one is the pattern to copy: request and use in the same function, so
  the ungranted path simply does nothing.
- `common/src/main/module.json5` carries `READ_IMAGEVIDEO` and
  `WRITE_IMAGEVIDEO` **commented out** in the `requestPermissions` array. They
  are restricted (ACL) permissions an ordinary app cannot ship, and
  `CameraService.saveCameraPhoto` uses `MediaAssetChangeRequest` +
  `applyChanges`, which requires `WRITE_IMAGEVIDEO` - so the save path cannot
  work as shipped either way (`HW-16-0003`). Album *reading* needs no
  permission at all here, because `AlbumService` goes through
  `PhotoViewPicker`.
- The HAP declares `metadata: { "name": "client_id" }`, needed for the HMS
  services the sample references.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` and
  `targetSdkVersion` are both `6.0.0(20)`; `useNormalizedOHMUrl` is on, which
  is required for this many HARs.
- **This is framework code, not a whole app.** The document says so: the data
  layer is `LocalDataManager`, an in-memory singleton over static literals.
  There is no network, no persistence, no login and no real payment.
- `EntryAbility.onCreate` locks the orientation to portrait for every
  non-tablet device, so the lg breakpoint is only reachable on a tablet or a
  2in1 window - the four-column feed cannot be seen by rotating a phone.
- `conversation` depends on a prebuilt binary, `../../har/image_operation.har`
  (shipped in the zip, absent from the documented tree). It is a black box:
  there is no source for it in the sample, so anything the conversation module
  does through it cannot be reviewed or modified.
- `features/search` and `features/camera` are absent from the HAP's
  `oh-package.json5`; they are pulled in transitively through `home`. Adding a
  second entry point to either one means adding the dependency explicitly.
- `EntryAbility.onWindowStageCreate` writes the avoid areas into `AppStorage`
  from inside the `getMainWindow` callback, which resolves after
  `loadContent`; the first frame therefore reads `topAvoid`/`bottomAvoid` as
  `undefined` and falls back to `0` via `AppStorage.get(...) || 0`.
- The `customScan` path needs a physical device with a camera. The doc's own
  comparison table is worth heeding: `scanBarcode` (the default UI) is
  pre-authorised and gives a consistent system experience but does not support
  floating or split-screen windows, while `customScan` requires the camera
  permission and makes you build the viewfinder yourself.

## Pitfalls

- **`HW-16-0001`** (B/high, confirmed): **`AudioRecorder` closes the recording
  file immediately after opening it, then hands the closed fd to `AVRecorder`.**
  `AudioRecorder.ets:66-73` opens the file and closes it in a `finally` before
  reading `file.fd`, so `prepare()`/`start()` get a dead descriptor and the
  voice message is empty or fails. The target is also named `01.mp3` while the
  profile packs `CFT_MPEG_4A`. Fix: keep the `fs.File` on the class, drop the
  `finally`, close it in `stopRecordingProcess` after `release()`, and rename
  to `01.m4a`.
- **`HW-16-0002`** (B/medium, confirmed): **the amplitude polling interval
  leaks and dereferences a released `AVRecorder`.** `clearInterval` runs only
  inside the `started || paused` branch of `stopRecordingProcess`, while
  `release()` and `this.avRecorder = undefined` run unconditionally, so a stop
  from any other state leaves a timer evaluating `this.avRecorder!` on
  `undefined` forever. Fix: hoist `clearInterval(this.time)` to the first line
  of `stopRecordingProcess` and guard the callback with
  `if (!this.avRecorder) return;`.
- **`HW-16-0003`** (D/medium, confirmed): **the photo-save flow needs a
  restricted permission it never obtains.** `CameraService.saveCameraPhoto`
  writes through `MediaAssetChangeRequest` + `applyChanges`, which requires
  `WRITE_IMAGEVIDEO`; the only runtime request in the app is `CAMERA`, and the
  album permissions are commented out of `common/src/main/module.json5`. Album
  reading meanwhile goes through `PhotoViewPicker`, which needs no permission,
  so `READ_IMAGEVIDEO` would be dead weight if restored. Fix: keep both
  declarations out and switch `saveCameraPhoto` to the security-component /
  authorization-dialog save flow (`SaveButton`,
  `showAssetsCreationDialog`).
- **`HW-16-0013`** (E/medium, confirmed): **systematic — abridged doc snippets
  are cut mid-construct and no longer parse.** This document's `CustomScan`
  excerpt (doc line ~310) carries a stray closing brace where the `Column`
  wrapper was elided, so it does not compile as printed; the zip source is
  valid. The same editorial defect recurs across roughly thirty shopping docs
  and five other industries. Fix: regenerate excerpts with brace-balanced
  elision.

## References

- `huawei_industry_tree/16_shopping/docs/01_practice-purchase-app-architecture-v1.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-purchase-app-architecture-v1-0000002077367333
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-cross-package.md` - `routerMap`, route names, cross-HAR navigation
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-cross-package
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - `Navigation`, `NavPathStack`, `NavDestination`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `documentation/harmonyos-guides/03_application-framework/arkts-layout-development-media-query.md` - the breakpoint ranges and `matchMediaSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-layout-development-media-query
- `documentation/harmonyos-references/02_application-framework/js-apis-mediaquery.md` - `MediaQueryListener`, `on`/`off`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-mediaquery
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List` and `lanes`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-lazyforeach.md` - `IDataSource` and the notification contract
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-lazyforeach
- `documentation/harmonyos-guides/02_media/scan-customscan.md` - the custom-UI scanning flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/scan-customscan
- `documentation/harmonyos-references/04_media/scan-customscan-api.md` - `customScan.init`, `start`, `stop`, `release`, `ViewControl`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/scan-customscan-api
- `documentation/harmonyos-guides/02_media/scan-scanbarcode.md` - the default-UI alternative the doc compares against
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/scan-scanbarcode
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - `XComponentType.SURFACE` and `getXComponentSurfaceId`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-guides/02_media/camera-shooting.md` - the Camera Kit session used by the recognise tab
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-shooting
- `documentation/harmonyos-references/04_media/arkts-apis-media-avrecorder.md` - `AVRecorder` states, `prepare`, `getAudioCapturerMaxAmplitude`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avrecorder
- `documentation/harmonyos-guides/05_media/photoaccesshelper-savebutton.md` - the permission-free way to save a captured photo
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-savebutton
- `documentation/harmonyos-guides/04_system/app-permissions.md` - which permissions are restricted
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/app-permissions
- `documentation/harmonyos-guides/11_environment-setup/ide-har.md` - HAR packaging and export barrels
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ide-har
- `SHOP-02` - the index of the 21 key-scenario samples that flesh out this skeleton
