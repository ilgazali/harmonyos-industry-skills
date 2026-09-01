---
id: AUTO-04
title: Nearby petrol stations - markers, selection and distance on a Map Kit map
industry: 01_auto
doc: huawei_industry_tree/01_auto/docs/04_nearby_gas_station.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/nearby_gas_station-0000002342199206
sample: huawei_industry_tree/01_auto/downloads/NearbyGasStation.zip
kits: ["@kit.MapKit", "@kit.LocationKit", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.ArkUI"]
apis: [MapComponent, map.MapComponentController, mapCommon.MapOptions, mapCommon.MarkerOptions, addMarker, map.Marker, setInfoWindowVisible, setAnimation, startAnimation, map.ScaleAnimation, map.newCameraPosition, animateCamera, map.calculateDistance, map.convertCoordinate, setMyLocation, setMyLocationEnabled, on markerClick, on myLocationButtonClick, geoLocationManager.getCurrentLocation, bindSheet]
permissions: [ohos.permission.LOCATION, ohos.permission.APPROXIMATELY_LOCATION, ohos.permission.INTERNET]
min_api: 20
modules: [entry]
findings: [HW-01-0020, HW-01-0021, HW-01-0022, HW-01-0023, HW-01-0024, HW-01-0025, HW-01-0026, HW-01-0027, HW-01-0028, HW-01-0029, HW-01-0030, HW-01-0050]
status: verified-with-fixes
---

## When to use

Load this card for any **"points of interest near me" screen**: petrol stations,
charging piles, dealerships, service centres, parking. The pattern is a
full-screen map with custom markers, a selectable marker that animates and opens
an info window, a bottom sheet listing the same places, and a distance shown per
place.

The document says outright that it applies equally to charging-pile lookup, so
treat it as the industry's generic POI-map card.

## Feature checklist

- Full-screen map centred on the user's current position.
- A custom marker per place, tappable.
- Tapping a marker: enlarges it, opens its info window, and animates the camera
  to it.
- A bottom sheet listing the places, with distance from the user.
- A my-location button that recentres the map.
- Distance computed with the Map Kit distance helper, shown in km to one
  decimal.

## Architecture

Single `entry` module.

```
entry/src/main/ets
├── common/Constants.ets
├── entryability/EntryAbility.ets   full-screen layout, avoid areas to AppStorage
├── model/StationData.ets           mock station list
├── pages/MainPage.ets              entry, pushes the station page
├── pages/GasStationPage.ets        the map screen
└── utils/
    ├── CalculateUtil.ets           distance between two coordinates
    ├── Logger.ets
    ├── MapUtil.ets                 camera, markers, animation, coordinate conversion
    └── PermissionsUtil.ets         check and request location permissions
```

Flow: permission → one-shot location fix → `MapComponent` initialised with
`mapOptions` → in the map callback, enable the my-location layer, register
`markerClick` and `myLocationButtonClick`, then add one marker per station →
`bindSheet` shows the list, driven by `$$this.isShow`.

**Coordinate systems matter here and are easy to get wrong.** The rule the
sample follows correctly:

| API | Coordinate system |
|---|---|
| `geoLocationManager.getCurrentLocation()` | WGS84 |
| `mapController.setMyLocation(location)` | **WGS84** - pass the raw fix, do not convert |
| `map.newCameraPosition` / `animateCamera` / `newLatLng` | **GCJ02** - convert first |

So the same fix is used twice, unconverted for the blue dot and converted for
the camera. That asymmetry looks like a bug and is not.

## Implementation steps

1. **Declare `LOCATION`, `APPROXIMATELY_LOCATION` and `INTERNET`** in
   `module.json5`, with `usedScene.abilities` naming an ability that actually
   exists (`HW-01-0026`).
2. **Check permissions, and only proceed when every one is granted**
   (`HW-01-0021`, `HW-01-0022`).
3. **Take exactly one location fix** and reuse it (`HW-01-0023`).
4. **Build `MapOptions`** with the target position, zoom and
   `myLocationControlsEnabled: true`, and hand it to `MapComponent` together
   with the callback.
5. **In the callback: check `err` first**, then enable the my-location layer,
   register listeners, and add markers.
6. **Add markers with `Promise.all`**, not `forEach` with an async callback
   (`HW-01-0024`).
7. **On `markerClick`, reset the previous selection before selecting the new
   one** (`HW-01-0020`).
8. **Release both listeners in `aboutToDisappear`** (`HW-01-0025`).

## Verified snippets

All snippets are from `NearbyGasStation.zip`. Corrected forms are marked.

**Distance between two coordinates — `utils/CalculateUtil.ets`** (as shipped)

```typescript
import { map, mapCommon } from '@kit.MapKit';

export class CalculateUtil {
  public static getDistance(lat1: number, long1: number, lat2: number, long2: number): string {
    let lan1: mapCommon.LatLng = { latitude: lat1, longitude: long1 };
    let lan2: mapCommon.LatLng = { latitude: lat2, longitude: long2 };
    // calculateDistance returns metres
    let distance: number = map.calculateDistance(lan1, lan2) / 1000;
    return distance.toFixed(1);
  }
}
```

Use `map.calculateDistance` rather than hand-rolling haversine - it is the one
piece of this sample that is clean and it is the piece most often written badly
by hand.

**Adding a marker — `utils/MapUtil.ets`** (as shipped)

```typescript
async addMapMaker(latitude: number, longitude: number, mapController: map.MapComponentController): Promise<void> {
  let markerOptions: mapCommon.MarkerOptions = {
    position: { latitude, longitude },
    rotation: 0,
    visible: true,
    zIndex: 0,
    alpha: 1,
    anchorU: 0.5,
    anchorV: 1,          // anchor at the pin's tip, not its centre
    clickable: true,
    flat: true,
    icon: 'station.svg', // resolved from entry/src/main/resources/rawfile/
    collisionRule: mapCommon.CollisionRule.NAME,
  };
  await mapController.addMarker(markerOptions);
}
```

Two details worth keeping: `anchorV: 1` puts the anchor at the bottom of the
icon so the pin tip sits on the coordinate, and `collisionRule` controls what
happens when markers overlap at low zoom. The `icon` string is a **rawfile**
path, so the asset belongs in `resources/rawfile/`, not `resources/base/media/`.

**Camera move — `utils/MapUtil.ets`** (as shipped)

```typescript
public moveToCurrentPosition(latitude: number, longitude: number,
  mapController: map.MapComponentController): void {
  let cameraPosition: mapCommon.CameraPosition = {
    target: { latitude, longitude },
    zoom: 15.9
  };
  let cameraUpdate: map.CameraUpdate = map.newCameraPosition(cameraPosition);
  mapController?.animateCamera(cameraUpdate, 500);   // 500 ms animation
}
```

`animateCamera(update, duration)` for a smooth move; `moveCamera(update)` for an
instant jump.

**Locating the user — `utils/MapUtil.ets`** (as shipped)

```typescript
async moveToMyLocation(mapController: map.MapComponentController): Promise<void> {
  let location: geoLocationManager.Location = await this.getMyLocation();
  mapController?.setMyLocation(location);                              // WGS84, unconverted
  let gcj02Position = await this.convertToGCJ02(location.latitude, location.longitude);
  this.moveToCurrentPosition(gcj02Position.latitude, gcj02Position.longitude, mapController);
}

public async convertToGCJ02(latitude: number, longitude: number): Promise<mapCommon.LatLng> {
  let theWGS84Position: mapCommon.LatLng = { latitude, longitude };
  return await map.convertCoordinate(
    mapCommon.CoordinateType.WGS84, mapCommon.CoordinateType.GCJ02, theWGS84Position);
}
```

**Marker selection — `pages/GasStationPage.ets`** (corrected, see `HW-01-0020`)

```typescript
this.mapController.on('markerClick', (marker) => {
  // FIX: shipped code never resets the previous marker, so selections accumulate
  if (this.curMarker && this.curMarker !== marker) {
    this.curMarker.setInfoWindowVisible(false);
    mapUtil.imageAnimation(this.curMarker, 1);
  }
  this.isShow = true;
  marker.setInfoWindowVisible(true);
  this.curMarker = marker;
  this.imageScale = 1.5;
  mapUtil.imageAnimation(marker, this.imageScale);
  mapUtil.moveToCurrentPosition(
    marker.getPosition().latitude, marker.getPosition().longitude, mapController);
});
```

**Marker scale animation — `utils/MapUtil.ets`** (as shipped, minus the pointless async)

```typescript
imageAnimation(marker: map.Marker, imageScale: number): void {
  let animation = new map.ScaleAnimation(Constants.ONE, imageScale, Constants.ONE, imageScale);
  animation.setDuration(100);
  animation.setFillMode(map.AnimationFillMode.FORWARDS);   // hold the end state
  marker.setAnimation(animation);
  marker.startAnimation();
}
```

`ScaleAnimation(fromX, toX, fromY, toY)` plus `AnimationFillMode.FORWARDS` is
what makes the enlarged marker stay enlarged after the animation ends - without
it the marker snaps back.

**Initialising the map — `pages/GasStationPage.ets`** (corrected, see `HW-01-0023` and `HW-01-0024`)

```typescript
this.callback = async (err, mapController): Promise<void> => {
  if (err) {
    Logger.error('testTag', `init fail, code: ${err.code}, message: ${err.message}`);
    return;
  }
  this.mapController = mapController;
  this.mapController.setMyLocationEnabled(true);

  this.mapController.on('myLocationButtonClick', () => {
    mapUtil.moveToMyLocation(mapController).then(() => { this.isShow = true; });
  });
  this.mapController.on('markerClick', (marker) => { /* see above */ });

  // FIX: shipped code calls getMyLocation() three times and mixes two fixes
  const location = await mapUtil.getMyLocation();
  mapController.setMyLocation(location);
  const gcj02 = await mapUtil.convertToGCJ02(location.latitude, location.longitude);
  mapUtil.moveToCurrentPosition(gcj02.latitude, gcj02.longitude, mapController);
  this.currentLatitude = location.latitude;
  this.currentLongitude = location.longitude;

  // FIX: shipped code uses forEach with an async callback, so nothing is awaited
  try {
    await Promise.all(this.stationInfoList.map((item: StationData) =>
      mapUtil.addMapMaker(item.latitude, item.longitude, mapController)));
  } catch (e) {
    Logger.error('testTag', `addMarker failed: ${JSON.stringify(e)}`);
  }
};
```

**Bottom sheet over the map — `pages/GasStationPage.ets`** (as shipped)

```typescript
.bindSheet($$this.isShow, this.bindBuilder(), {
  detents: [Constants.BIND_SHEET_HEIGHT, SheetSize.MEDIUM],
  dragBar: false,
  title: { title: $r('app.string.gas_station') },
  blurStyle: BlurStyle.COMPONENT_THICK,
  backgroundColor: $r('app.color.bind_sheet_background'),
  onWillDismiss: ((dismissSheetAction: DismissSheetAction) => {
    if (this.curMarker) {
      this.imageScale = 1;
      mapUtil.imageAnimation(this.curMarker, this.imageScale);
    }
    dismissSheetAction.dismiss();
  })
});
```

`detents` with two stops gives the sheet a peek height and an expanded height.
`onWillDismiss` must call `dismissSheetAction.dismiss()` or the sheet will not
close - intercepting the event makes you responsible for completing it.

**Releasing the listeners** (corrected, see `HW-01-0025`)

```typescript
// FIX: the shipped page has no aboutToDisappear at all.
aboutToDisappear(): void {
  this.mapController?.off('markerClick');
  this.mapController?.off('myLocationButtonClick');
}
```

## Permissions & config

`entry/src/main/module.json5`:

```json5
"requestPermissions": [
  { "name": "ohos.permission.LOCATION",
    "reason": "$string:permission_reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" } },
  { "name": "ohos.permission.APPROXIMATELY_LOCATION",
    "reason": "$string:permission_reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" } },
  { "name": "ohos.permission.INTERNET",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" } }
]
```

The shipped manifest names `ShopAbility` here, which does not exist in the
module (`HW-01-0026`). `INTERNET` is a system-grant permission so it needs no
`reason`; the two location permissions are user-grant and do.

**Map Kit needs AppGallery Connect setup before anything renders**: enable the
map service under API management for the app, then complete manual signing.
Without both steps the map is blank and the failure is not obvious from the
code.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- AppGallery Connect: map service enabled **and** a manual signing profile.
- Station data is mock (`model/StationData.ets`). Real POI lookup needs the Map
  Kit site search, which is why `INTERNET` is declared.
- The document states no device scope, and the manifest declares
  `phone, tablet, 2in1, wearable` with no responsive handling anywhere in the
  source (`HW-01-0030`).
- The marker icon must live under `resources/rawfile/`.

## Pitfalls

- **`HW-01-0020` — marker selection never resets.** Tapping a second marker
  while the sheet is open leaves the first enlarged with its info window open
  and orphans it, because `curMarker` is overwritten and the only reset lives in
  `onWillDismiss`, which does not fire.
- **`HW-01-0021` — the permission result is never inspected.** The `then` branch
  runs on refusal too, logs success, and calls `getCurrentLocation()` anyway -
  producing an unhandled rejection instead of a handled denial.
- **`HW-01-0022` — `checkPermissions` only reads the last permission.** Same
  `applyResult` overwrite as the industry's architecture practice; a
  granted-approximate / denied-precise user is treated as fully granted.
- **`HW-01-0023` — three GNSS fixes per screen, and latitude and longitude come
  from two different ones.** In a moving vehicle the two halves describe
  different places and every station distance is wrong.
- **`HW-01-0024` — `forEach` with an async callback.** The promises are dropped,
  the awaits do not sequence anything, and marker failures are unhandled.
- **`HW-01-0025` — three listeners, no `off()`, no `aboutToDisappear`.** Stale
  `markerClick` handlers keep writing into destroyed component state.
- **`HW-01-0026` — `usedScene.abilities: ["ShopAbility"]`** on all three
  permissions, in a module whose only ability is `EntryAbility`.
- **`HW-01-0027` — the document's permission snippet does not compile.** It
  passes a `context` that is neither a parameter nor a member, and deletes the
  `Logger.error` call, leaving an empty catch.
- **`HW-01-0028` — the tree says `EntryBackAbility.ets`,** the archive and the
  manifest say `EntryBackupAbility.ets`.
- **`HW-01-0029` — `MapUtil` has a comment describing an older synchronous
  implementation, an `@Observed` on a class with no properties, and an `async`
  method with no `await`.**
- **`HW-01-0030` — `wearable` is declared with no watch adaptation** and
  contradicts the device set of the industry's architecture sample.

## References

- `documentation/harmonyos-guides/04_application-services/map-location.md` - my-location layer; states explicitly that `setMyLocation` uses **WGS84**
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-location
- `documentation/harmonyos-guides/04_application-services/map-marker.md` - markers, anchors, info windows
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-marker
- `documentation/harmonyos-guides/04_application-services/map-listening.md` - `on`/`off` for map events
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-listening
- `documentation/harmonyos-guides/04_application-services/map-interaction.md` - camera moves
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-interaction
- `documentation/harmonyos-guides/04_application-services/map-config-agc.md` - AppGallery Connect prerequisites
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-config-agc
- `documentation/harmonyos-references/06_application-services/map-module-desc.md` - `convertCoordinate`, `calculateDistance`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-module-desc
- `documentation/harmonyos-guides/04_system/request-app-permissions.md` - `authResults`, the field this sample ignores
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-app-permissions
- `AUTO-01` - the industry architecture; its `ServicePage` embeds a smaller preview version of this same map
