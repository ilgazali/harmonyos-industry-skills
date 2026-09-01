---
id: SPORT-11
title: Live GPS track - locationChange subscription driving an incremental TraceOverlay
industry: 03_sports_health
doc: huawei_industry_tree/03_sports_health/docs/11_motion_trajectory.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/motion_trajectory-0000002351150394
sample: huawei_industry_tree/03_sports_health/downloads/MotionTrajectory.zip
kits: ["@kit.LocationKit", "@kit.MapKit", "@kit.AbilityKit", "@kit.ArkUI"]
apis: ["geoLocationManager.on('locationChange')", "geoLocationManager.off('locationChange')", "geoLocationManager.getCurrentLocation", "geoLocationManager.isLocationEnabled", "geoLocationManager.LocationRequest", LocationRequestPriority, LocationRequestScenario, SingleLocationRequest, "map.newLatLng", animateCamera, addTraceOverlay, "mapCommon.TraceOverlayParams", "map.convertCoordinateSync", "abilityAccessCtrl.requestPermissionsFromUser", MapComponent]
permissions: ["ohos.permission.LOCATION", "ohos.permission.APPROXIMATELY_LOCATION", "ohos.permission.INTERNET", "ohos.permission.GET_NETWORK_INFO"]
min_api: 20
modules: [entry]
findings: [HW-03-0042, HW-03-0043, HW-03-0044, HW-03-0045, HW-03-0055]
status: verified-with-fixes
---

## When to use

Load this card when the app must **draw where the user is going, as they go** -
a run, a ride, a hike, a delivery leg. It is the recording half of the pair
whose other half is `SPORT-08`, which frames a finished track.

Three things are specific to live tracking and worth taking:

- **`LocationRequestScenario.TRAJECTORY_TRACKING`** - a scenario constant that
  tells the platform this is route recording, so it tunes the fix strategy for
  it rather than being handed raw priority and interval numbers.
- **A single-shot fix first, a subscription second.** `getCurrentLocation`
  positions the map immediately; `on('locationChange')` then keeps it moving.
- **WGS84 to GCJ02 on every fix**, not just the first.

The subscription is also the most expensive thing an app can hold, which is
why two of this card's four findings are about releasing and bounding it.

## Feature checklist

- Prompt to enable location services and request both location permissions.
- Take one fix, convert it, and centre the map on it.
- Subscribe to location changes for the duration of the run.
- On each fix: recentre the camera and extend the drawn track.
- The map is stripped of zoom, compass and rotation controls so the track is
  the only thing on screen.

## Architecture

One `entry` module, two pages, three utilities.

```
entry/src/main/ets
├── entryability/EntryAbility.ets
├── entrybackupability/
├── pages
│   ├── MainPage.ets            the entry screen
│   └── MapPage.ets             the run screen: permissions, fixes, camera, track
└── utils
    ├── Logger.ets
    ├── MapUtil.ets             convertToGcj02
    └── PermissionsRequest.ets  commonRequestPermissions
```

**The flow is three stages, and the sample keeps them in that order:**

```
1. isLocationEnabled()  ──> prompt the user to switch location services on
   commonRequestPermissions(['LOCATION', 'APPROXIMATELY_LOCATION'])
                │
2. getCurrentLocation(SingleLocationRequest)  ──> one fix, fast
   convertToGcj02 ──> map.newLatLng ──> animateCamera        (the map is now on the user)
                │
3. on('locationChange', requestInfo, callback)               (the map now follows the user)
   per fix: convert ──> animateCamera ──> extend the track
```

**Checking `isLocationEnabled()` before requesting the permission** is the
detail most samples skip: a granted permission is useless while the device's
location service is off, and the two failures need different messages.

## Implementation steps

1. **Declare both location permissions *and* the network permissions**
   (`HW-03-0044`) - the map will not draw without the latter.
2. **Check the location service, then request the permissions** as a pair.
3. **Take one fix with `SingleLocationRequest`** and a short timeout to
   position the map.
4. **Convert every fix from WGS84 to GCJ02** before it touches the map.
5. **Subscribe with `TRAJECTORY_TRACKING`** and keep the callback in a field.
6. **Accumulate the points and keep one overlay**, replacing it as the track
   grows (`HW-03-0043`).
7. **Skip the first segment** until two real fixes exist (`HW-03-0045`).
8. **Unsubscribe in `aboutToDisappear`** (`HW-03-0042`).

## Verified snippets

All snippets are from `MotionTrajectory.zip`. Corrected forms are marked.

**Getting started — `entry/src/main/ets/pages/MapPage.ets`** (as shipped)

```typescript
import { geoLocationManager } from '@kit.LocationKit';

const isLocationEnabled = geoLocationManager.isLocationEnabled();
const permissions: Array<Permissions> = ['ohos.permission.LOCATION', 'ohos.permission.APPROXIMATELY_LOCATION'];
await PermissionsRequest.commonRequestPermissions(this.getUIContext(), permissions);
```

**Requesting the two location permissions as one array is mandatory**, not a
convenience: `LOCATION` alone is refused. `isLocationEnabled()` answers a
different question from the permission - whether the device's location service
is switched on at all - and a run screen needs both.

**The first fix — same file** (as shipped)

```typescript
const request: geoLocationManager.SingleLocationRequest = {
  locatingPriority: 0x501,        // priority constant for a fast, accurate single fix
  locatingTimeoutMs: 1000         // do not make the user wait: fall back to the map default
};
geoLocationManager.getCurrentLocation(request).then((location: geoLocationManager.Location) => {
  this.point = MapUtil.convertToGcj02(location.latitude, location.longitude);
  const cameraUpdate = map.newLatLng(this.point);
  this.mapController?.animateCamera(cameraUpdate);
});
```

**`SingleLocationRequest` with a one-second timeout** is the right tool for
"where am I, now" - it is a different type from the continuous
`LocationRequest` used for the subscription, and using the subscription for
the initial position would leave the map on its default until the first fix
arrived.

**The subscription — same file** (corrected, see `HW-03-0042`, `HW-03-0043`, `HW-03-0045`)

```typescript
private locationChange = async (location: geoLocationManager.Location): Promise<void> => {
  const next = MapUtil.convertToGcj02(location.latitude, location.longitude);
  this.points.push(next);

  const cameraUpdate = map.newLatLng(next);
  this.mapController?.animateCamera(cameraUpdate);

  if (this.points.length < 2) {
    return;                                    // FIX: the sample draws from a seeded literal
  }
  this.traceOverlay?.remove();                 // FIX: the sample adds a new overlay per fix
  this.traceOverlay = await this.mapController?.addTraceOverlay({
    points: this.points,                       // FIX: the sample passes only the last pair
    animationDuration: 100,
    color: 0xAA36C18D,                         // ARGB: translucent green over the road
    width: 20
  });
};

Location() {
  const requestInfo: geoLocationManager.LocationRequest = {
    'priority': geoLocationManager.LocationRequestPriority.ACCURACY,
    'scenario': geoLocationManager.LocationRequestScenario.TRAJECTORY_TRACKING,
    'timeInterval': 20,        // seconds; the minimum for network-based positioning
    'distanceInterval': 0,     // report regardless of how far the user has moved
    'maxAccuracy': 0
  };
  try {
    geoLocationManager.on('locationChange', requestInfo, this.locationChange);
  } catch (err) {
    LOGGER.error(`locationChange: errCode: ${err.code} , message:${err.message}`);
  }
}

aboutToDisappear(): void {                     // FIX: absent in the sample
  geoLocationManager.off('locationChange', this.locationChange);
  this.traceOverlay?.remove();
  this.mapController?.clear();
}
```

**`LocationRequestScenario.TRAJECTORY_TRACKING` is the parameter to reach
for.** The scenario tells the platform what the app is doing, and it tunes the
positioning strategy accordingly - which is more portable than hand-picking a
priority and an interval for every device. `ACCURACY` alongside it is the
highest-cost priority, so this configuration is correct for an active run and
must not outlive it.

`distanceInterval: 0` means every fix is reported even when the user is
standing still, which is what makes the missing teardown expensive.

**Coordinate conversion — `entry/src/main/ets/utils/MapUtil.ets`** (as shipped)

```typescript
public static convertToGcj02(latitude: number, longitude: number) {
  const wgs84Position: mapCommon.LatLng = { latitude: latitude, longitude: longitude };
  return map.convertCoordinateSync(
    mapCommon.CoordinateType.WGS84, mapCommon.CoordinateType.GCJ02, wgs84Position);
}
```

**Every fix needs converting, not just the first.** `geoLocationManager`
reports WGS84; the Huawei map renders GCJ02. Convert once at the boundary -
as this utility does - so nothing downstream has to remember which system a
coordinate is in.

**Stripping the map — same file** (as shipped)

```typescript
this.mapOptions = {
  position: { target: this.point, zoom: 17 },
  zoomControlsEnabled: false,
  compassControlsEnabled: false,
  rotateGesturesEnabled: false
};
```

Three controls switched off, deliberately: during a run the screen is glanced
at, not operated, and a stray rotation would break the sense of direction the
track provides.

## Permissions & config

The module gets the map credential right and the permissions wrong
(`HW-03-0044`). Corrected:

```json5
// entry/src/main/module.json5
"metadata": [
  // It needs to be modified according to your application before it can run.
  { "name": "client_id", "value": "***********" }      // correct: module level, placeholder value
],
"requestPermissions": [
  {
    "name": "ohos.permission.LOCATION",
    "reason": "$string:location_reason_route",          // FIX: sample uses EntryAbility_desc
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  },
  {
    "name": "ohos.permission.APPROXIMATELY_LOCATION",
    "reason": "$string:location_reason_approximate",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  },
  { "name": "ohos.permission.INTERNET",         "usedScene": { "when": "always" } },   // FIX: absent
  { "name": "ohos.permission.GET_NETWORK_INFO", "usedScene": { "when": "always" } }    // FIX: absent
]
```

The `client_id` placement is worth noting as the good example: module level,
an obvious placeholder, and a comment saying it must be changed - compare
`SPORT-01`, which nests it inside the ability with a real-looking value.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **`deviceTypes` is `["phone"]` only** - narrower than the rest of the
  industry, and correct for a screen carried on a run.
- **AGC setup plus manual signing** is required for the map, as for every Map
  Kit sample.
- **The run does not survive the screen locking or the app backgrounding.**
  There is no continuous task and no background mode, so a real running app
  needs both - and the linter rule cited in `HW-03-0042` forbids background
  location without one.
- Nothing is persisted: the track exists only as overlays on a live map, and
  `this.points` is discarded with the page. Feeding `SPORT-08` means storing
  it.
- Map Kit, the coordinate conversion and GCJ02 are **mainland-China**.

## Pitfalls

- **`HW-03-0042` — the `locationChange` subscription is never cancelled** and
  the page has no `aboutToDisappear`, so an `ACCURACY` /
  `TRAJECTORY_TRACKING` request with a zero distance filter keeps running for
  the life of the process.
- **`HW-03-0043` — every fix adds another two-point trace overlay** and none
  is removed, so an hour's run leaves thousands of live overlays. The full
  path is collected in `this.points` and never used.
- **`HW-03-0044` — no `INTERNET` or `GET_NETWORK_INFO`**, so the map the track
  is drawn on cannot load, and both location permissions are justified with
  the ability description.
- **`HW-03-0045` — the first segment is drawn from the seeded literal
  `{ latitude: 22, longitude: 113 }`,** a point in the sea, to the user's real
  position.

## References

- `documentation/harmonyos-references/06_application-services/js-apis-geolocationmanager.md` - `on('locationChange')`, `LocationRequest`, `LocationRequestScenario`, `SingleLocationRequest`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-geolocationmanager
- `documentation/harmonyos-guides/12_coding-and-debugging/ide-reasonable-gps-use-check.md` - the rule on releasing location subscriptions
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ide-reasonable-gps-use-check
- `documentation/harmonyos-guides/07_application-services/map-dyntrajectories.md` - `addTraceOverlay` and `TraceOverlayParams`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-dyntrajectories
- `documentation/harmonyos-references/06_application-services/map-map-traceoverlay.md` - the overlay handle and `remove()`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-map-traceoverlay
- `documentation/harmonyos-guides/07_application-services/map-camera.md` - `newLatLng` and `animateCamera`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-camera
- `documentation/harmonyos-guides/04_application-services/map-location.md` - the two location permissions
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-location
- `documentation/harmonyos-guides/07_application-services/map-faq-1.md` - the network permissions a map needs
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-faq-1
- `SPORT-08` - framing the finished track; `TOUR-01` - the same WGS84-to-GCJ02 conversion at app scale
