---
id: AUTO-01
title: Automotive app - layered architecture with map and scan services
industry: 01_auto
doc: huawei_industry_tree/01_auto/docs/01_practice-auto-app-architecture-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-auto-app-architecture-v1-0000001903742656
sample: huawei_industry_tree/01_auto/downloads/BTYOMI.zip
kits: ["@kit.ArkUI", "@kit.MapKit", "@kit.ScanKit", "@kit.LocationKit", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.ArkData"]
apis: [Navigation, NavPathStack, MapComponent, map.MapComponentController, mapCommon.MapOptions, map.convertCoordinate, map.newLatLng, site.searchByText, scanBarcode.startScanForResult, scanCore.ScanType, geoLocationManager.getCurrentLocation, abilityAccessCtrl.AtManager, bundleManager.getBundleInfoForSelf, preferences.getPreferencesSync, window.getLastWindow]
permissions: [ohos.permission.LOCATION, ohos.permission.APPROXIMATELY_LOCATION]
min_api: 20
modules: [entry, common, features/explore, features/buyingCar, features/shoppingMall, features/service, features/mine]
findings: [HW-01-0001, HW-01-0002, HW-01-0003, HW-01-0004, HW-01-0005, HW-01-0006, HW-01-0007, HW-01-0008, HW-01-0009, HW-01-0010, HW-01-0011, HW-01-0049]
status: verified-with-fixes
---

## When to use

Load this card when building an **automotive brand or dealership app**: news
feed, model catalogue with test-drive booking, accessory store, and a service
area with charging-station maps and QR-code charging. It defines the module
split, the navigation skeleton, and the two service capabilities that make this
industry distinct - **Map Kit** and **Scan Kit**.

It is also the reference for any app whose main screen is a map with a
half-modal list sheet over it.

## Feature checklist

- Five bottom tabs: explore, buy, mall, service, mine.
- Splash countdown, then a privacy-policy gate, then login, then the tab shell.
- Explore: hero banner carousel plus a waterfall list of industry news.
- Buy: model detail pages, marquee imagery, test-drive booking.
- Mall: product search, detail, purchase.
- Service: **QR scan to charge**, **optimal charging station** on a map with a
  half-modal station list, and a personal charging record.
- Mine: my vehicle, orders, points, home charger, settings.
- Location permission requested up front on the service tab.

## Architecture

Three layers, one IDE project, five feature HARs plus a shared `common` HAR.

| Layer | Contents | Module |
|---|---|---|
| Product customisation | Splash, privacy gate, login, `Navigation` root, tab host | `entry` |
| Basic feature | explore, buyingCar, shoppingMall, service, mine - one HAR each | `features/*` |
| Common capability | Shared components, models, lazy data source, logging, permission and preferences utils | `common` |

`build-profile.json5` registers all seven; only `entry` carries a `targets`
block.

Navigation is a **single `NavPathStack` in `NavigationPage`**, provided as
`@Provide('pageInfos')` and consumed by every feature page with
`@Consume('pageInfos')`. Feature HARs never import one another - they push route
names. The startup chain is
`PrivacyPolicyPage → SplashScreenPage → LoginPage → TabPage`, each step done
with `replacePath` so the back gesture cannot return into it.

System inset heights are measured once in `NavigationPage.aboutToAppear` and
published to `AppStorage` under `avoidArea` and `bottom`.

> The document's module table also promises 国际化 (internationalisation) in the
> common layer. The scaffolding exists but is essentially unused - see
> `HW-01-0008`.

## Implementation steps

1. **Register all modules in `build-profile.json5`**, HARs with `name` and
   `srcPath` only.
2. **Create the stack once** in the entry page, `@Provide` it, map route names
   to destinations in an `@Builder`, and `replacePath` through the startup
   chain.
3. **Publish inset heights to `AppStorage`** so feature pages can pad
   themselves.
4. **Declare `LOCATION` and `APPROXIMATELY_LOCATION` together** in
   `entry/src/main/module.json5`. They are requested as a pair; asking for
   precise location alone is not the supported model.
5. **Check and request permission before the map needs it**, and *await the
   result* (`HW-01-0001`, `HW-01-0003`).
6. **Configure the map through `mapCommon.MapOptions`** at
   `aboutToAppear`, then take the controller from the `mapCallback`.
7. **Convert WGS84 to GCJ02 before centring the camera** - this step is
   mandatory for mainland China and easy to miss.
8. **Release every map listener** in `aboutToDisappear` (`HW-01-0002`).
9. **For scanning, prefer `scanBarcode.startScanForResult`** - the default UI is
   pre-authorised for the camera, so no `ohos.permission.CAMERA` is needed. Only
   reach for `customScan` if you must draw your own scanner UI, and then you do
   need the camera permission.

## Verified snippets

All snippets are from `BTYOMI.zip`. Corrected forms are marked.

**Single navigation stack — `entry/src/main/ets/pages/NavigationPage.ets`** (as shipped)

```typescript
@Entry
@Component
struct NavigationPage {
  @Provide('pageInfos') pageInfos: NavPathStack = new NavPathStack();
  uiContext: UIContext = this.getUIContext();

  @Builder
  PagesMap(name: string) {
    if (name === 'SplashScreenPage') {
      SplashScreenPage();
    } else if (name === 'PrivacyPolicyPage') {
      PrivacyPolicyPage();
    } else if (name === 'LoginPage') {
      LoginPage();
    } else if (name === 'TabPage') {
      TabPage();
    } else if (name === 'OptimalStation') {
      OptimalStation();
    } else if (name === 'MyCharge') {
      MyCharge();
    }
  }

  aboutToAppear(): void {
    this.pageInfos.replacePath({ name: 'PrivacyPolicyPage' }, false);
    window.getLastWindow(this.uiContext.getHostContext()).then((windowStage: window.Window) => {
      let area = windowStage.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM);
      let topHeight = this.uiContext.px2vp(area.topRect.height);
      let bottomArea = windowStage.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR);
      let bottom = this.uiContext.px2vp(bottomArea.bottomRect.height);
      if (topHeight > 0) {
        AppStorage.setOrCreate('avoidArea', topHeight);
        AppStorage.setOrCreate('bottom', bottom);
      }
    });
  }

  build() {
    Navigation(this.pageInfos) {
    }
    .hideTitleBar(true)
    .navDestination(this.PagesMap)
  }
}
```

Feature pages are imported from their HARs (`@ohos/service`, `@ohos/mine`, …)
and rendered by name. Note the destination structs are plain `@Component`, not
`@Entry` - the document's snippet gets this wrong (`HW-01-0007`).

**Map setup — `features/service/src/main/ets/pages/ServicePage.ets`** (as shipped)

```typescript
import { map, mapCommon, MapComponent } from '@kit.MapKit';
import { AsyncCallback } from '@kit.BasicServicesKit';

private mapOption?: mapCommon.MapOptions;
private callback?: AsyncCallback<map.MapComponentController>;
private mapController?: map.MapComponentController;
private zoom: number = 14;

aboutToAppear(): void {
  this.mapOption = {
    position: { target: this.latLng, zoom: this.zoom },
    scrollGesturesEnabled: false,      // an embedded preview map, not interactive
    zoomControlsEnabled: false,
    myLocationControlsEnabled: false,
    compassControlsEnabled: false,
    scaleControlsEnabled: true
  };

  this.callback = async (err, mapController) => {
    if (!err) {
      this.mapController = mapController;
      this.mapController?.setMyLocationEnabled(true);
      this.mapController?.setMyLocationControlsEnabled(false);
      this.mapController.on('mapClick', () => {
        this.pageInfos.pushPath({ name: 'OptimalStation' });
      });
      this.getMyLocation();
    }
  };
}

// in build():
MapComponent({ mapOptions: this.mapOption, mapCallback: this.callback })
  .width('100%').height('100%');
```

The pattern worth copying is the **preview map**: gestures and controls all
disabled, one `mapClick` listener that pushes the full-screen map page. The
full page (`OptimalStation`) then enables `myLocationControlsEnabled` and adds
`markerClick` and `myLocationButtonClick`.

**Releasing the listeners** (corrected, see `HW-01-0002`)

```typescript
// FIX: neither shipped page releases anything. Add this to both.
aboutToDisappear(): void {
  this.mapController?.off('mapClick');
  this.mapController?.off('markerClick');
  this.mapController?.off('myLocationButtonClick');
}
```

**Locating the user and centring the camera — `ServicePage.ets`** (corrected, see `HW-01-0003`)

```typescript
// FIX: shipped code uses .then() with no .catch(), and runs before the
// permission request has resolved.
async getMyLocation() {
  try {
    const result = await geoLocationManager.getCurrentLocation();
    this.mapController?.setMyLocation({
      latitude: result.latitude, longitude: result.longitude,
      altitude: 0, accuracy: 0, speed: 0, timeStamp: 0, direction: 0,
      timeSinceBoot: 0, altitudeAccuracy: 0, speedAccuracy: 0,
      directionAccuracy: 0, uncertaintyOfTimeSinceBoot: 0,
      sourceType: geoLocationManager.LocationSourceType.GNSS
    });
    const gcj02 = await this.convertCoordinate(result.latitude, result.longitude);
    this.mapController?.moveCamera(map.newLatLng(gcj02, this.zoom));
  } catch (err) {
    Logger.error(`getCurrentLocation failed: ${JSON.stringify(err)}`);
  }
}
```

**Coordinate conversion — same file** (as shipped)

```typescript
async convertCoordinate(latitude: number, longitude: number): Promise<mapCommon.LatLng> {
  let wgs84Position: mapCommon.LatLng = { latitude, longitude };
  return await map.convertCoordinate(
    mapCommon.CoordinateType.WGS84,
    mapCommon.CoordinateType.GCJ02,
    wgs84Position);
}
```

`geoLocationManager` returns WGS84; Map Kit renders GCJ02 in mainland China.
Skipping this shifts every pin by a few hundred metres. This is the single most
valuable line in the sample.

**Scan to charge — `ServicePage.ets`** (as shipped)

```typescript
import { scanBarcode, scanCore } from '@kit.ScanKit';

let options: scanBarcode.ScanOptions = {
  scanTypes: [scanCore.ScanType.ALL],
  enableMultiMode: true,
  enableAlbum: true
};
try {
  scanBarcode.startScanForResult(this.getUIContext().getHostContext(), options)
    .then((result: scanBarcode.ScanResult) => {
      Logger.info(`Promise scan result:${JSON.stringify(result)}`);
      this.showScanResult(result);
    })
    .catch((error: BusinessError) => {
      Logger.error(`Promise error: ${JSON.stringify(error)}`);
    });
} catch (error) {
  Logger.error(`failReason: ${JSON.stringify(error)}`);
}
```

Both a `catch` on the promise and a `try` around the call are needed: the API
throws synchronously for bad parameters and rejects for runtime failures.
`startScanForResult` uses the system scanner UI, which is **pre-authorised for
the camera** - the sample declares no `ohos.permission.CAMERA`.

**Permission gate — `common/src/main/ets/utils/PermissionsUtil.ets`** (corrected, see `HW-01-0001` and `HW-01-0004`)

```typescript
// FIX: shipped code overwrites applyResult each iteration, so only the last
// permission decides; and every catch is empty.
async checkPermissions(permissions: Array<Permissions>, uiContext: UIContext): Promise<boolean> {
  for (const permission of permissions) {
    const grantStatus = await this.checkAccessToken(permission);
    if (grantStatus !== abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED) {
      return await this.requestPermissions(permissions, uiContext);
    }
  }
  return true;
}

async checkAccessToken(permission: Permissions): Promise<abilityAccessCtrl.GrantStatus> {
  const atManager = abilityAccessCtrl.createAtManager();
  try {
    const bundleInfo = await bundleManager.getBundleInfoForSelf(
      bundleManager.BundleFlag.GET_BUNDLE_INFO_WITH_APPLICATION);
    return await atManager.checkAccessToken(bundleInfo.appInfo.accessTokenId, permission);
  } catch (error) {
    Logger.error(`checkAccessToken failed: ${JSON.stringify(error)}`);
    return abilityAccessCtrl.GrantStatus.PERMISSION_DENIED;
  }
}
```

## Permissions & config

`entry/src/main/module.json5` — declared in the entry HAP only; the feature HARs
declare none:

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.LOCATION",
    "reason": "$string:EntryAbility_desc",
    "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
  },
  {
    "name": "ohos.permission.APPROXIMATELY_LOCATION",
    "reason": "$string:EntryAbility_desc",
    "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
  }
]
```

**No camera permission** - the default scan UI is pre-authorised.

**Map Kit needs AppGallery Connect configuration** before any map renders. The
app does not bundle a map SDK, so there is no package-size cost and no SDK
upgrade to manage, but the AGC setup is a hard prerequisite.

`build-profile.json5`: `compatibleSdkVersion: "6.0.0(20)"`, `runtimeOS:
"HarmonyOS"`.

## Constraints

- DevEco Studio 6.0.0 Release or later; HarmonyOS 6.0.0 Release SDK or later.
- Map Kit requires an AppGallery Connect project with the map APIs enabled.
- `startScanForResult` does not support floating-window or split-screen, and
  album scanning recognises a single code only. `customScan` lifts the UI
  restriction but requires `ohos.permission.CAMERA` and a hand-built scanner
  screen.
- **The login screen performs no authentication.** The document states this
  plainly: any password is accepted once 11 digits are entered. Implement real
  authentication before shipping anything derived from this.
- Station data is local (`features/service/.../model/StationModel.ets`), not
  from a POI service. Swap in Map Kit's location search to make it real.
- The document scopes the practice to phones but the manifest declares tablet
  and 2in1, and the project has no breakpoint handling at all (`HW-01-0009`).

## Innovation worth knowing

For "navigate me there", the document recommends **embedding a navigation
atomic service** rather than firing an implicit `Want`. An implicit Want makes
the user pick from a disambiguation dialog, and shows "please install a map app"
when none is present. An atomic service needs no installation and renders inside
your own app. Worth adopting for any industry with a navigate-to-location step.

## Pitfalls

- **`HW-01-0001` — the permission check only reads the last permission.**
  `applyResult` is assigned, not accumulated, so with
  `['LOCATION', 'APPROXIMATELY_LOCATION']` a user who granted approximate but
  denied precise location is treated as fully granted and never asked again.
- **`HW-01-0002` — five map listeners, zero `off()`.** Neither map page has
  `aboutToDisappear`. Both are pages the user enters and leaves repeatedly, so
  handlers pile up and eventually fire several times per tap.
- **`HW-01-0003` — `getCurrentLocation()` has no `catch`, and runs unawaited.**
  Location off, permission refused, or no fix in time all produce an unhandled
  rejection, and the permission request is fire-and-forget so the map proceeds
  while the dialog is still up.
- **`HW-01-0004` — six empty catch blocks** across `PermissionsUtil` and
  `PreferencesUtil` turn infrastructure failures into ordinary denials and
  silent no-ops. A `LoggerUtil` sits in the same folder, unused by them.
- **`HW-01-0005` — `PreferencesUtil` returns `false` instead of your default**
  when the store failed to open, and never awaits `flush()`, so a write followed
  by an exit is lost.
- **`HW-01-0006` — the project tree in the document is wrong in six places,**
  including a `compontents` misspelling in the sample and three files that do
  not exist in the archive.
- **`HW-01-0007` — the document's snippets are an older revision.** They use
  `getContext(this)`, add an `@Entry` the sample does not have, and call
  `SheetTransition({ StationList: Station.getStationList() })` where the sample
  exports `stationList` and `station` - so that block does not compile verbatim.
- **`HW-01-0008` — 64 hardcoded Chinese strings** despite `en_US` resource
  directories in every module and an explicit i18n promise in the module table.
- **`HW-01-0009` — phone-only in prose, phone/tablet/2in1 in the manifest,**
  with no responsive handling in the code.
- **`HW-01-0010` — `PreferencesUtil` imports `@ohos.app.ability.common` and
  `@ohos.data.preferences`** while the file next to it uses `@kit.AbilityKit`.
- **`HW-01-0011` — `(password.length > 5 ?? false)`** - `??` on a boolean is
  dead code and hides what the check was meant to be.

## References

- `documentation/harmonyos-guides/07_application-services/` - Map Kit and Scan Kit guides
- `documentation/harmonyos-guides/03_application-framework/arkts-global-interface.md` - why `getContext` is being replaced by `getHostContext`; the ambiguous-UI-context problem
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-global-interface
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `NavPathStack`, `replacePath`, `navDestination`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-guides/04_system/request-app-permissions.md` - runtime permission flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-app-permissions
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - declaring permissions in `module.json5`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- Document's own pointers: `practice-common-app-layered-v1` (layered modularisation), `map-config-agc` (AGC setup for Map Kit), `scan-scanbarcode` (default scan UI)
