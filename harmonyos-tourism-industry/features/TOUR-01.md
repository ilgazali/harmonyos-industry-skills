---
id: TOUR-01
title: Tourist park app - multi-HAR architecture with park map, routing and navigation
industry: 09_tourism
doc: huawei_industry_tree/09_tourism/docs/01_practice-tourist-park-app-architecture-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1-0000001965211653
sample: huawei_industry_tree/09_tourism/downloads/TouristParkDemo.zip
kits: ["@kit.MapKit", "@kit.LocationKit", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.ArkData", "@kit.ScanKit", "@kit.CryptoArchitectureKit", "@kit.PerformanceAnalysisKit", "@kit.ArkUI"]
apis: [MapComponent, "map.MapComponentController", "map.MapEventManager", addPolygon, addMarker, addPolyline, "navi.getDrivingRoutes", "map.convertCoordinate", "geoLocationManager.on('locationChange')", "geoLocationManager.getAddressesFromLocation", "geoLocationManager.isGeocoderAvailable", "abilityAccessCtrl.createAtManager", checkAccessToken, requestPermissionsFromUser, "bundleManager.getBundleInfoForSelf", "UIAbilityContext.startAbility", "scanBarcode.startScanForResult", "relationalStore.getRdbStore", "window.getWindowAvoidArea", "windowClass.on('avoidAreaChange')", NavPathStack, "@Provide", "@Consume", LazyForEach, WaterFlow]
permissions: ["ohos.permission.INTERNET", "ohos.permission.APPROXIMATELY_LOCATION", "ohos.permission.LOCATION"]
min_api: 24
modules: [phone, common, home, ParkService, MapService, PersonalCenter, AccountCenter]
findings: [HW-09-0001, HW-09-0002, HW-09-0003, HW-09-0004, HW-09-0005, HW-09-0006, HW-09-0007, HW-09-0008, HW-09-0009, HW-09-0010, HW-09-0011, HW-09-0012, HW-09-0013, HW-09-0014, HW-09-0015, HW-09-0079]
status: verified-with-fixes
---

## When to use

Load this card when building a **venue app for a bounded physical site** - a
theme park, a resort, a zoo, a campus, a scenic area - where the product is
"one place, many services": tickets, hotel, parking, dining, a shop, a map of
the site, and an account area. It is the architecture card for the whole app;
the individual capabilities have their own cards (`TOUR-03` location
permission, `TOUR-06` / `TOUR-07` / `TOUR-12` map and markers).

The part worth copying is the **module split** and the **park map**: draw the
site boundary as a polygon, drop a marker on every attraction, plan a driving
route inside the site, and hand off to Petal Maps for turn-by-turn navigation.

**Read the Pitfalls section before copying any of it.** Fifteen findings, four
of them in the map and location lifecycle. This sample is a framework skeleton
- the document says so, 本篇代码非应用的全量代码 (this is not the full
application code) - and it shows: the login is UI only, several utility
classes are stubs, and the entry manifest still carries NFC declarations from
a different demo.

## Feature checklist

- Four bottom-tab sections: 首页 (home), 游园 (park visiting), 地图 (map),
  我的 (mine).
- Splash screen that auto-replaces into the main page.
- Home: park poster swiper, service entries (ticket / annual card / hotel /
  dining / parking / shop), an announcement list, a city and park picker
  dialog, and a barcode scanner entry.
- Location on entering home: request permission, take a fix, reverse-geocode
  it, convert WGS84 to GCJ02 and publish it to AppStorage for the map tab.
- Map tab: site boundary polygon, one marker per attraction, marker tap opens
  a navigation dialog, a driving route drawn as a polyline, hand-off to Petal
  Maps by Want.
- Mine: account, personal info, membership, messages, orders, favourites,
  settings, about.
- Responsive down to sm/md/lg breakpoints; the entry HAP installs on phone,
  tablet and 2in1.

## Architecture

Three layers, one HAP and six HARs, all in one IDE project. This is the
分层模块化实践 (layered modularization practice) shape.

```
phone/                         product customization layer - the only HAP (type: entry)
├── ets/entryability/EntryAbility.ets     window setup, avoid-area -> AppStorage
├── ets/entrybackupability/               backup extension
├── ets/pages/SplashPage.ets              @Entry, 3 s timer -> MainPage
├── ets/pages/MainPage.ets                @Entry, Navigation + Tabs, owns @Provide state
├── ets/utils/{GlobalContext,PreferencesUtil}.ets
└── ets/viewmodel/MainPageData.ets

features/                      basic feature layer - one HAR per section
├── home/            Home.ets, Notice.ets, CustomDialogExample.ets  (location, scan, city picker)
├── ParkService/     ParkVisiting.ets, CardPage.ets, TravelTips.ets
├── MapService/      MapPage.ets (the map), MainPage.ets (navigation dialog)
├── PersonalCenter/  Personal.ets, PersonalCenter.ets, AppSettingPage.ets, MessagePage.ets, AboutApp.ets
└── AccountCenter/   LoginPage.ets

common/                        common capability layer - one HAR
├── components/LocalList.ets
├── constants/{Breakpoint,Common,Page,Style}Constants.ets
├── database/Rdb.ets, database/tables/AccountTable.ets
├── utils/{BreakpointSystem,CommonDataSource,Logger}.ets
└── viewmodel/{AccountInfo,ConstantsInterface,LocalDataModel,MemberShipData}.ets
```

Note the directory names: `features` (plural), `home` lower case,
`PersonalCenter` upper camel case. The document's tree spells all three
differently (`HW-09-0010`), and `build-profile.json5` `srcPath` entries are
case sensitive.

**How state moves.** `MainPage` is the single provider: it declares
`@Provide('pageStack') NavPathStack`, `@Provide('topRectHeight')` and
`@Provide('bottomRectHeight')`, and every feature HAR consumes them by alias.
That is what lets a HAR push a page without importing the router owner. The
cross-tab data - the location fix, the selected city and park - travels
through `AppStorage` instead, because `home` computes it and `MapService`
reads it.

**Routing between HARs** is by `routerMap`: each feature HAR declares
`"routerMap": '$profile:route_map'` in its own `module.json5` and lists its
`@Builder` entry functions in `resources/base/profile/route_map.json`. A
caller then does `this.pageStack.pushPathByName('Notice', null)` with no
import of the target module.

## Implementation steps

1. **Create the HAP and the HARs** and wire them in the root
   `build-profile.json5` `modules` array, each with a `srcPath`. Only `phone`
   gets a `targets` block - it is the only module that builds into a product.
2. **Depend by alias.** Each HAR's `oh-package.json5` lists what it needs, for
   example `"@ohos/common": "file:../../common"`. Keep one alias convention
   for the whole project; this sample uses `@ohos/common` in five modules and
   `@oh/common` in AccountCenter, which builds but reads as a mistake.
3. **Export a public surface per HAR** from its root `Index.ets`. Only export
   what actually works - see `HW-09-0015`.
4. **Declare the permissions once.** Put `INTERNET`,
   `APPROXIMATELY_LOCATION` and `LOCATION` in the module that uses them; a
   HAP referencing a HAR merges their permission configurations at build time,
   so there is no need to repeat them. The document's snippet lists only
   `LOCATION` and is wrong (`HW-09-0009`).
5. **Set the map credentials** in the entry `module.json5` `metadata` -
   `client_id` and `app_id` from AGC. The document calls this file
   `model.json5` (`HW-09-0011`); it is `module.json5`.
6. **Take the avoid areas in the ability**, publish them to `AppStorage`, and
   subscribe to `avoidAreaChange` - then release that subscription in
   `onWindowStageDestroy` (`HW-09-0013`).
7. **Provide them from the page as numbers** and consume them as numbers.
   The sample provides `number` and consumes `string` in ten files
   (`HW-09-0005`).
8. **Gate location behind the permission check**, then take a fix, convert the
   coordinate and publish it. Release the subscription on teardown
   (`HW-09-0004`).
9. **Build the map** in the `MapComponent` callback: polygon first, then the
   markers, then the listeners - through the event manager, and `off` them in
   `aboutToDisappear` (`HW-09-0002`).
10. **Hand off navigation** to Petal Maps with a `Want`, using
    `getUIContext().getHostContext()` (`HW-09-0001`).

## Verified snippets

All snippets are from `TouristParkDemo.zip`. Corrected forms are marked.

**Avoid areas into AppStorage — `phone/src/main/ets/entryability/EntryAbility.ets`** (corrected, see `HW-09-0013`)

```typescript
import { UIAbility } from '@kit.AbilityKit';
import { window } from '@kit.ArkUI';

export default class EntryAbility extends UIAbility {
  private windowClass?: window.Window;

  onWindowStageCreate(windowStage: window.WindowStage): void {
    windowStage.loadContent('pages/SplashPage', (err) => {
      if (err.code) {
        hilog.error(0x0000, 'testTag', 'Failed to load the content: %{public}s', JSON.stringify(err) ?? '');
        return;
      }
    });
    this.windowClass = windowStage.getMainWindowSync();
    this.windowClass.setWindowLayoutFullScreen(true)
      .catch((err: BusinessError) => {                    // FIX: shipped call has no catch
        hilog.error(0x0000, 'testTag', 'setWindowLayoutFullScreen failed: %{public}s', JSON.stringify(err));
      });

    let type = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR;
    AppStorage.setOrCreate('bottomRectHeight', this.windowClass.getWindowAvoidArea(type).bottomRect.height);
    type = window.AvoidAreaType.TYPE_SYSTEM;
    AppStorage.setOrCreate('topRectHeight', this.windowClass.getWindowAvoidArea(type).topRect.height);

    this.windowClass.on('avoidAreaChange', (data) => {
      if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
        AppStorage.setOrCreate('topRectHeight', data.area.topRect.height);
      } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
        AppStorage.setOrCreate('bottomRectHeight', data.area.bottomRect.height);
      }
    });
  }

  onWindowStageDestroy(): void {
    this.windowClass?.off('avoidAreaChange');            // FIX: shipped body only logs
  }
}
```

The pattern itself is right and worth copying: read the avoid areas once,
publish them to `AppStorage` in px, and let the page convert with `px2vp`.
Full-screen layout means the app draws under the status bar and the navigation
indicator, so every page needs these two numbers as padding.

**The single provider — `phone/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
@Entry
@Component
struct MainPage {
  @Provide('bottomRectHeight') bottomRectHeight: number =
    this.getUIContext().px2vp(AppStorage.get<number>('bottomRectHeight'));
  @Provide('topRectHeight') topRectHeight: number =
    this.getUIContext().px2vp(AppStorage.get<number>('topRectHeight'));
  @Provide('pageStack') pageStack: NavPathStack = new NavPathStack();

  build() {
    Navigation(this.pageStack) {
      Column() {
        Stack({ alignContent: Alignment.BottomStart }) {
          Tabs({ index: this.currentPageIndex }) {
            TabContent() { Home({}); }
            TabContent() { ParkVisiting({}); }
            TabContent() { MapPage({}); }
            TabContent() { Personal(); }
          }
          .barHeight(0)          // the built-in bar is hidden; the Row below is the real one
          .scrollable(false)
          .animationDuration(0)
        }
      }
    }
    .hideTitleBar(true)
    .hideToolBar(true)
  }
}
```

**`barHeight(0)` plus `scrollable(false)` plus a hand-built `Row` of tabs** is
the idiom here: `Tabs` keeps the four sections alive and switchable by index
while the visible bar is an ordinary row you can style freely and pad by
`bottomRectHeight`. Consumers must declare these as `number`
(`@Consume('topRectHeight') topRectHeight: number`), not `string`.

**Permission check then location — `features/home/src/main/ets/components/Home.ets`** (as shipped)

```typescript
import { abilityAccessCtrl, bundleManager, common, Permissions } from '@kit.AbilityKit';

permissions: Array<Permissions> = ['ohos.permission.LOCATION', 'ohos.permission.APPROXIMATELY_LOCATION'];

async checkAccessToken(permission: Permissions): Promise<abilityAccessCtrl.GrantStatus> {
  let atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
  let grantStatus: abilityAccessCtrl.GrantStatus = abilityAccessCtrl.GrantStatus.PERMISSION_DENIED;
  let tokenId: number = 0;
  try {
    let bundleInfo: bundleManager.BundleInfo =
      await bundleManager.getBundleInfoForSelf(bundleManager.BundleFlag.GET_BUNDLE_INFO_WITH_APPLICATION);
    tokenId = bundleInfo.appInfo.accessTokenId;
  } catch (error) {
    const err: BusinessError = error as BusinessError;
    Logger.error(`Failed to get bundle info for self. Code is ${err.code}, message is ${err.message}`);
  }
  try {
    grantStatus = await atManager.checkAccessToken(tokenId, permission);
  } catch (error) {
    const err: BusinessError = error as BusinessError;
    Logger.error(`Failed to check access token. Code is ${err.code}, message is ${err.message}`);
  }
  return grantStatus;
}

async checkPermissions(context: common.UIAbilityContext): Promise<void> {
  let grantStatus: abilityAccessCtrl.GrantStatus = await this.checkAccessToken(this.permissions[0]);
  if (grantStatus === abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED) {
    this.getLocationInfo();
  } else {
    this.reqPermissionsFromUser(this.permissions, context);
  }
}

aboutToAppear() {
  this.checkPermissions(this.getUIContext().getHostContext() as common.UIAbilityContext);
}
```

Check first, request only if not granted - `requestPermissionsFromUser` itself
decides whether to raise the dialog, so this is belt and braces, but it keeps
the granted path off the async request entirely. Note that the two location
permissions must be requested **as one array**; `LOCATION` alone is refused.

**Location, reverse geocode, coordinate conversion — same file** (corrected, see `HW-09-0004`)

```typescript
import { geoLocationManager } from '@kit.LocationKit';
import { map, mapCommon } from '@kit.MapKit';

private locationChange = async (location: geoLocationManager.Location): Promise<void> => {
  try {
    if (geoLocationManager.isGeocoderAvailable()) {
      const req: geoLocationManager.ReverseGeoCodeRequest =
        { 'latitude': location.latitude, 'longitude': location.longitude, 'maxItems': 1 };

      // the fix is WGS84; Huawei map layers are GCJ02 - convert before storing
      const gcj02: mapCommon.LatLng = await map.convertCoordinate(
        mapCommon.CoordinateType.WGS84, mapCommon.CoordinateType.GCJ02,
        { latitude: location.latitude, longitude: location.longitude });
      AppStorage.setOrCreate('latitude', gcj02.latitude);
      AppStorage.setOrCreate('longitude', gcj02.longitude);

      geoLocationManager.getAddressesFromLocation(req, (err, data) => {
        Logger.info('getAddressesFromLocation: ' + JSON.stringify(err ?? data));
        geoLocationManager.off('locationChange', this.locationChange);
      });
    }
  } catch (err) {
    Logger.error(`errCode:${err.code}, errMessage:${err.message}`);
    geoLocationManager.off('locationChange', this.locationChange);
  }
};

getLocationInfo() {
  const requestInfo: geoLocationManager.LocationRequest = {
    'priority': geoLocationManager.LocationRequestPriority.FIRST_FIX,
    'timeInterval': 0, 'distanceInterval': 0, 'maxAccuracy': 0
  };
  geoLocationManager.on('locationChange', requestInfo, this.locationChange);
}

aboutToDisappear(): void {                                    // FIX: absent in the sample
  geoLocationManager.off('locationChange', this.locationChange);
}
```

**`map.convertCoordinate` is the line people forget.** A `geoLocationManager`
fix is WGS84; the Huawei map renders GCJ02. Skipping the conversion puts the
user's blue dot a few hundred metres off the road inside mainland China.
`FIRST_FIX` priority plus an `off()` from inside the callback is the sample's
one-shot idiom - but it only unsubscribes on a path that runs, hence the
unconditional `aboutToDisappear` above.

**Site boundary and attraction markers — `features/MapService/src/main/ets/components/MapPage.ets`** (corrected, see `HW-09-0002`, `HW-09-0003`)

```typescript
import { MapComponent, mapCommon, map, navi } from '@kit.MapKit';

private mapController?: map.MapComponentController;
private mapEventManager?: map.MapEventManager;

aboutToAppear() {
  this.callback = async (err, mapController) => {
    if (err) {
      Logger.error(this.TAG, `Failed to initialize the map. Code: ${err.code}; message: ${err.message}`);
      return;                                   // FIX: shipped code has no error branch
    }
    this.mapController = mapController;

    // the park boundary: a closed polygon over the site's corner coordinates
    const polygonOptions: mapCommon.MapPolygonOptions = {
      points: [
        { latitude: 31.92076, longitude: 118.884807 },
        { latitude: 31.920765, longitude: 118.901785 },
        { latitude: 31.875771, longitude: 118.885653 },
        { latitude: 31.895816, longitude: 118.851926 },
        { latitude: 31.92315, longitude: 118.849931 },
        { latitude: 31.925617, longitude: 118.882312 }
      ],
      holes: [], clickable: true,
      fillColor: 0x00000000,                    // ARGB, alpha 0x00 - outline only
      geodesic: false,
      strokeColor: 0xff000000, strokeWidth: 10,
      jointType: mapCommon.JointType.DEFAULT, patterns: [],
      visible: true, zIndex: 0
    };
    await this.mapController.addPolygon(polygonOptions);

    this.marker = await this.mapController.addMarker(markerOptions);
    this.marker.setTitle('初始位置');
    this.marker.setSnippet('这是子标题');
    this.marker.setClickable(true);
    this.addBasicPoints();

    this.mapEventManager = this.mapController.getEventManager();   // FIX: official on/off surface
    this.mapEventManager.on('mapLoad', this.onMapLoad);
    this.mapEventManager.on('markerClick', this.onMarkerClick);
  };
}

private onMarkerClick = (marker: map.Marker) => {
  this.distinction = marker.getTitle();
  this.distinclat = marker.getPosition().latitude;
  this.distinclon = marker.getPosition().longitude;
  this.dialogController?.open();
};

aboutToDisappear(): void {
  this.mapEventManager?.off('markerClick');     // FIX: shipped teardown only nulls the dialog
  this.mapEventManager?.off('mapLoad');
  this.mapController?.clear();
  this.dialogController = null;
}
```

`fillColor: 0x00000000` is worth reading twice: the colours are **ARGB**, so a
zero alpha byte gives a boundary outline with no tint over the site. Marker
titles double as the payload - `onMarkerClick` reads `getTitle()` and
`getPosition()` straight back out to feed the navigation dialog, so no side
table of attractions is needed.

**Driving route inside the site — same file** (corrected, see `HW-09-0014`)

```typescript
async getRoutes() {
  try {
    this.routes = [];                            // FIX: shipped code never resets
    this.points = [];
    this.drivingRouteParams.origins = this.marker?.getPosition() ? [this.marker.getPosition()] : [];
    this.drivingRouteParams.destination = { latitude: 31.904083, longitude: 118.88457 };

    const result: navi.RouteResult = await navi.getDrivingRoutes(this.drivingRouteParams);
    if (!result.routes || result.routes.length === 0) {   // FIX: shipped code indexes [0] blind
      return;
    }
    result.routes[0].steps.forEach((step) => {           // FIX: the shipped callback is async for nothing
      this.routes.push({
        distance: step.distance, distanceDescription: step.distanceDescription,
        duration: step.duration, durationDescription: step.durationDescription,
        durationInTraffic: step.durationInTraffic,
        durationInTrafficDescription: step.durationInTrafficDescription
      });
      step.roads.forEach((road) => road.polyline.forEach((p) => this.points.push(p)));
    });

    const points = result.routes[0].overviewPolyline == null ? this.points : result.routes[0].overviewPolyline;
    // thin the path: a polyline of thousands of points costs more than it shows
    this.polylineOption.points = points.length >= 1000
      ? points.filter((_, i) => i % 50 === 0 || i === points.length - 1)
      : points.filter((_, i) => i % 2 === 0 || i === points.length - 1);

    if (this.mapPolyline) {
      this.mapPolyline.setPoints(this.polylineOption.points);   // reuse, do not re-add
    } else {
      this.mapPolyline = await this.mapController?.addPolyline(this.polylineOption);
    }
  } catch (err) {
    Logger.error('naviDemo', 'getDrivingRoutes fail err =' + JSON.stringify(err));
  }
}
```

Two details are right and worth keeping: **thinning the polyline** by index
before drawing, and **reusing the existing `MapPolyline` with `setPoints`**
instead of adding a second overlay on every request. The `overviewPolyline` is
preferred when present; `this.points`, stitched from every road of every step,
is the fallback.

**Hand-off to Petal Maps — `features/MapService/src/main/ets/components/MainPage.ets`** (as shipped)

```typescript
import { common, Want } from '@kit.AbilityKit';

let petalMapWant: Want = {
  bundleName: 'com.huawei.hmos.maps.app',
  uri: 'maps://navigation',
  parameters: {
    linkSource: 'com.example.touristparkdemo',   // your own bundle name
    destinationLatitude: this.distinclat,
    destinationLongitude: this.distinclon,
    destinationName: this.distinction,
    vehicleType: 0
  }
};
let context = this.getUIContext().getHostContext() as common.UIAbilityContext;
context.startAbility(petalMapWant);
```

**Use this form, not the document's** - the document's copy of this snippet
calls `getContext(this)`, deprecated since API 18 (`HW-09-0001`). Turn-by-turn
navigation is not something you implement; you pass the destination to Petal
Maps and it takes over.

**Cross-HAR routing — `features/home/src/main/resources/base/profile/route_map.json`** (as shipped)

```json
{
  "routerMap": [
    {
      "name": "Notice",
      "pageSourceFile": "src/main/ets/components/Notice.ets",
      "buildFunction": "NoticeBuilder",
      "data": { "description": "this is NoticeBuilder" }
    }
  ]
}
```

with, in the same HAR's `module.json5`, `"routerMap": '$profile:route_map'`,
and in the component file:

```typescript
@Builder
export function NoticeBuilder() {
  Notice();
}
```

The caller then pushes by name only - `this.pageStack.pushPathByName('Notice', null)` -
so a feature HAR never imports another feature HAR to navigate into it. This
is the mechanism that keeps the module graph acyclic.

## Permissions & config

Entry `module.json5` (`phone/src/main/module.json5`):

```json5
"metadata": [
  { "name": "client_id", "value": "<your AGC Client ID>" },
  { "name": "app_id",    "value": "<your AGC App ID>" }
],
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" },
  {
    "name": "ohos.permission.APPROXIMATELY_LOCATION",
    "reason": "$string:permission_location_approximately_reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  },
  {
    "name": "ohos.permission.LOCATION",
    "reason": "$string:permission_location_reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
]
```

- `LOCATION` must be requested together with `APPROXIMATELY_LOCATION`. Both
  are `user_grant`, so `reason` and `usedScene` are mandatory and the reason
  string must exist in `resources/base/element/string.json`.
- `INTERNET` is what makes the map tiles load. The document's permission list
  omits it and omits `APPROXIMATELY_LOCATION` (`HW-09-0009`).
- Delete the NFC `actions` and `tag-tech` `uris` the sample carries in its
  `skills` block (`HW-09-0008`).
- Each feature HAR: `"type": "har"` and `"routerMap": '$profile:route_map'`.
  A HAP merges a referenced HAR's permission configuration at build time, so
  declare each permission once.

## Constraints

- API Version 24 Release; DevEco Studio 6.1.1 Release;
  `compatibleSdkVersion` and `targetSdkVersion` both `6.1.1(24)`.
- **The map does not render without AGC setup.** App registered in AGC,
  fingerprint configured, Client ID filled into `module.json5`, Map service
  switched on. The document lists the failure causes: no network, unfinished
  preparation, missing Client ID, map permission not enabled, app identity
  verification failure.
- Map Kit and the Petal Maps hand-off are **mainland-China services**; the
  `com.huawei.hmos.maps.app` bundle is not present on every device, so
  `startAbility` can fail and needs a catch in production.
- The sample is a **skeleton**: the login accepts any password with an
  11-digit phone number, the verification code is printed to the log, and the
  data is static. The document says so - 框架代码中登录验证，只是UI能力 (the
  login check in the framework code is UI only) - and completing the
  authentication is left to the developer.
- Coordinates in the sample are a specific park near Nanjing; replace both the
  polygon vertices and the marker positions, in GCJ02.

## Pitfalls

- **`HW-09-0001` — the document's navigation snippet uses `getContext(this)`,**
  deprecated since API 18 while the document targets API 24. The shipped code
  is already correct; copy from the zip, not from the page.
- **`HW-09-0002` — `mapLoad` and `markerClick` are registered and never
  released.** The whole teardown is `this.dialogController = null`. Register
  through `mapController.getEventManager()` and `off()` both in
  `aboutToDisappear`, then `mapController.clear()`.
- **`HW-09-0003` — the map init callback has no error branch,** so every cause
  of the blank map the document warns about arrives as `err` and is discarded.
  Log it.
- **`HW-09-0004` — the location subscription is only cancelled from inside its
  own callback.** No fix, no callback, no `off()` - GNSS runs on in the
  background. Add `aboutToDisappear`.
- **`HW-09-0005` — `@Provide` says `number`, ten `@Consume` declarations say
  `string`.** The guide is explicit that the types must match or implicit
  conversion causes application exceptions; since API 20 a BuilderNode-crossing
  mismatch is a runtime error.
- **`HW-09-0006` — `PreferencesUtil.getPreferencesValue` ends in a bare
  `return;`** after reading the value, so it always resolves to `undefined`.
- **`HW-09-0007` — `TravelTips` loads `$rawfile('userIndex.html')` from
  `ParkService`, which ships no `rawfile` directory.** `$rawfile` resolves
  against the calling module, so the travel-guide page is blank.
- **`HW-09-0008` — the exported entry ability is registered as an NFC tag and
  HCE handler for nine tag technologies,** with no NFC code anywhere in the
  project. Copy-paste residue; the `'[ReadTagTest-Demo]'` log tag in
  `EntryAbility.ets` is the fingerprint. Delete it.
- **`HW-09-0009` — the document's permission block lists only `LOCATION`.**
  That alone yields no fix, and `INTERNET` is missing too.
- **`HW-09-0010` — the documented tree does not match the zip:** `feature` vs
  `features`, `Home` vs `home`, `personalcenter` vs `PersonalCenter`,
  `ParkingVistingData.ets` vs `ParkingVisitingData.ets`.
- **`HW-09-0011` — the map setup note says `model.json5`.** No such file; it
  is `module.json5`.
- **`HW-09-0012` — the document says phone only,** the manifest says
  `["phone", "tablet", "2in1"]`.
- **`HW-09-0013` — `avoidAreaChange` is subscribed in `onWindowStageCreate`
  and never released,** although `onWindowStageDestroy`'s own comment says UI
  resources should be released there.
- **`HW-09-0014` — `getRoutes` indexes `result.routes[0]` blind and never
  resets its accumulators,** so an empty route result throws into a silent
  catch and a second tap on 路线管理 draws both routes concatenated.
- **`HW-09-0015` — `BreakPointType.getValue()` returns `'' as T`** with the
  real implementation commented out, and it is exported from
  `common/Index.ets` as public API of the common capability layer.

## References

- `documentation/harmonyos-guides/04_application-services/map-presenting.md` - map initialization, the `mapCallback` error branch
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-presenting
- `documentation/harmonyos-guides/04_application-services/map-listening.md` - `MapEventManager`, `getEventManager()`, the event names
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-listening
- `documentation/harmonyos-guides/07_application-services/map-faq-4.md` - the `off()` plus `clear()` teardown
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-faq-4
- `documentation/harmonyos-guides/04_application-services/map-location.md` - the two location permissions Map Kit requires
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-location
- `documentation/harmonyos-guides/04_application-services/map-marker.md` - `MarkerOptions`, anchors, icons
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-marker
- `documentation/harmonyos-guides/12_coding-and-debugging/ide-reasonable-gps-use-check.md` - the linter rule on unreleased location subscriptions
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ide-reasonable-gps-use-check
- `documentation/harmonyos-guides/03_application-framework/arkts-provide-and-consume.md` - the same-type rule for `@Provide` and `@Consume`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-provide-and-consume
- `documentation/harmonyos-references/02_application-framework/js-apis-getcontext.md` - `getContext` deprecation
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-getcontext
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `user_grant` permission declaration
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- `documentation/harmonyos-guides/01_getting-started/har-package.md` - HAR permission merging at build time
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/har-package
- `documentation/harmonyos-guides/04_system/nfc-tag-access-guide.md` - what the NFC skills in the manifest actually declare
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/nfc-tag-access-guide
