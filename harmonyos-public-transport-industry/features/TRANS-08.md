---
id: TRANS-08
title: Map rotation lock, zoom stepping and locate-me
industry: 06_public_transport
doc: huawei_industry_tree/06_public_transport/docs/08_map_rotation_lock.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_rotation_lock-0000002396727869
sample: huawei_industry_tree/06_public_transport/downloads/MapRotationLock.zip
kits: ["@kit.MapKit", "@kit.LocationKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit"]
apis: [MapComponent, map.MapComponentController, setRotateGesturesEnabled, getCameraPosition, moveCamera, map.newCameraPosition, mapCommon.CameraPosition, map.convertCoordinate, geoLocationManager.getCurrentLocation, abilityAccessCtrl.requestPermissionsFromUser, abilityAccessCtrl.requestPermissionOnSetting, Toggle]
permissions: [ohos.permission.LOCATION, ohos.permission.APPROXIMATELY_LOCATION]
min_api: 20
modules: [entry]
findings: [HW-06-0044, HW-06-0045, HW-06-0046, HW-06-0047, HW-06-0048, HW-06-0051]
status: verified-with-fixes
---

## When to use

Load this card for the **map control surface**: a locate-me button, zoom in/out
buttons, and toggles that enable or disable individual map gestures. In transit
apps the rotation lock matters because riders orient the map against station
signage and street names, and an accidental two-finger twist makes that
impossible.

It also carries the cleanest **permission helper** in this industry - a three
step ladder that is worth lifting whole.

## Feature checklist

- A map filling the screen.
- A locate-me control that centres on the user.
- Zoom in and out controls.
- A toggle that locks and unlocks rotation gestures, with a confirmation
  message when locking.
- Permission handled as a ladder: check, ask, then offer settings.

## Architecture

Single `entry` module.

```
entry/src/main/ets
├── common/Constants.ets              centre, default zoom, dimensions
├── entryability/EntryAbility.ets
├── models/TabBarItem.ets
├── pages/MainPage.ets                the map, the toolbar, the toggle
└── utils/
    ├── MapUtil.ets                   camera, zoom, rotation lock, locate
    ├── PermissionsRequest.ets        the permission ladder
    └── Logger.ets
```

`MapUtil` is a view-model-shaped class holding `mapController`, `center`,
`zoomSize` and `bearing`. It refreshes `zoomSize` and `bearing` from
`getCameraPosition()` when the map settles, so the object tracks what the user
has actually done to the map rather than only what the app set.

## Implementation steps

1. **Declare both location permissions** and run the ladder before touching the
   map.
2. **Await the permission** before requesting a position (`HW-06-0046`).
3. **Take the controller from `MapComponent`'s callback** and null-check it in
   every method that uses it - the sample does this consistently.
4. **Convert WGS84 to GCJ02** before handing coordinates to the camera.
5. **Move the camera straight from the location result**; do not gate it behind
   a reverse-geocode (`HW-06-0044`).
6. **Clamp zoom to the documented 2–20 range** in both directions
   (`HW-06-0045`).
7. **Preserve the user's zoom and bearing** when recentring (`HW-06-0048`).
8. **Toggle gestures with `setRotateGesturesEnabled`** - note the inversion:
   locking means passing `false`.

## Verified snippets

All snippets are from `MapRotationLock.zip`. Corrected forms are marked.

**The permission ladder — `utils/PermissionsRequest.ets`** (as shipped)

```typescript
async requestPermissions(context: UIContext) {
  // 1. already granted?
  let isPermission: boolean = await this.checkPermissions();
  if (!isPermission) {
    // 2. ask
    let isGranted = await this.requestPermissionsFromUser(context);
    if (!isGranted) {
      // 3. refused (or refused before): offer the settings sheet
      this.requestPermissionsOnSetting(context);
    }
  }
}

async requestPermissionsFromUser(context: UIContext): Promise<boolean | undefined> {
  const atManager = abilityAccessCtrl.createAtManager();
  try {
    const data = await atManager.requestPermissionsFromUser(
      context.getHostContext() as common.UIAbilityContext, PERMISSIONS);
    Logger.info('requestPermissionsFromUser success', JSON.stringify(data));
    return data.authResults.every((r) => r === 0);
  } catch (err) {
    Logger.error('requestPermissionsFromUser err', JSON.stringify(err));
    return undefined;
  }
}
```

**This is the pattern to copy in this industry.** Three distinct steps, each
with a clear verdict, and the settings sheet only reached after a real refusal -
which matters because `requestPermissionsFromUser` stops showing its dialog once
the user has declined.

Note the helper takes a **`UIContext`** and converts internally with
`context.getHostContext() as common.UIAbilityContext`. That is deliberate: the
caller passes what a component naturally has, and the conversion lives in one
place.

**Locate me — `utils/MapUtil.ets`** (corrected, see `HW-06-0044`, `HW-06-0048`)

```typescript
private getLocation() {
  const request: geoLocationManager.SingleLocationRequest = {
    locatingPriority: geoLocationManager.LocatingPriority.PRIORITY_LOCATING_SPEED,
    locatingTimeoutMs: 10000
  };
  try {
    geoLocationManager.getCurrentLocation(request)
      .then((result) => {
        // FIX: the shipped version wraps everything below in a
        // getAddressesFromLocation callback whose result it never uses, so an
        // unavailable or failing geocoder silently cancels the camera move.
        this.center.latitude = result.latitude;
        this.center.longitude = result.longitude;
        this.moveCamera();      // FIX: shipped code inlines a camera update
                                //      that resets zoom to the constant
      })
      .catch((error: BusinessError) => {
        Logger.error('promise, getCurrentLocation: error=', JSON.stringify(error));
      });
  } catch (err) {
    Logger.error('errCode:', JSON.stringify(err));
  }
}
```

**Camera move — same file** (as shipped)

```typescript
private moveCamera() {
  if (!this.mapController) {
    return;
  }
  const cameraPosition: mapCommon.CameraPosition = {
    target: this.center,
    zoom: this.zoomSize,
    bearing: this.bearing
  };
  const cameraUpdate: map.CameraUpdate = map.newCameraPosition(cameraPosition);
  this.mapController.moveCamera(cameraUpdate);
}
```

Carrying `zoom` and `bearing` alongside `target` is what makes a recentre feel
like a recentre - the view keeps the scale and orientation the user chose.

**Zoom stepping** (corrected, see `HW-06-0045`)

```typescript
// documented range: 2 to 20
private static readonly MAP_ZOOM_MIN: number = 2;
private static readonly MAP_ZOOM_MAX: number = 20;

setZoom(enlarge: boolean) {
  if (!this.mapController) {
    return;
  }
  // FIX: shipped code has no upper guard, and its lower guard (>= 1) permits 0
  if (enlarge && this.zoomSize < MapUtil.MAP_ZOOM_MAX) {
    this.zoomSize++;
    this.moveCamera();
  } else if (!enlarge && this.zoomSize > MapUtil.MAP_ZOOM_MIN) {
    this.zoomSize--;
    this.moveCamera();
  }
}
```

`CameraPosition` field ranges worth knowing: **`zoom` 2–20** (default 2),
**`tilt` 0–75**, **`bearing` [0, 360)**. Only `bearing` is documented as
self-correcting when out of range - values like 361 become 1. `zoom` and `tilt`
carry no such promise, so clamp them yourself.

**The rotation lock — same file** (as shipped)

```typescript
rotateGesturesLock(context: UIContext, isLock: boolean) {
  if (!this.mapController) {
    return;
  }
  this.mapController.setRotateGesturesEnabled(!isLock);   // note the inversion
  if (isLock) {
    context.getPromptAction().showToast({ message: $r('app.string.toast_message_rotation_lock') });
  }
}
```

Driven from a `Toggle`:

```typescript
Toggle({ type: ToggleType.Switch, isOn: this.isLock })
  .onChange((isOn: boolean) => {
    this.mapVM.rotateGesturesLock(this.getUIContext(), isOn);
  })
```

`setRotateGesturesEnabled` has siblings for the other gestures -
`setScrollGesturesEnabled`, `setZoomGesturesEnabled`, `setTiltGesturesEnabled` -
and each can be locked independently. Confirming only the *lock* direction with
a toast, and staying silent on unlock, is a good default: the restrictive action
is the one that needs explaining.

**Keeping the model in step with the map — same file** (as shipped)

```typescript
// in the map-ready callback, once the camera settles:
this.zoomSize = this.mapController!.getCameraPosition().zoom;
this.bearing = this.mapController!.getCameraPosition().bearing as number;
this.center = { /* from getCameraPosition().target */ };
```

Without this, the zoom buttons would step from whatever value the app last set
rather than from what the user pinched to.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.LOCATION", ... },
  { "name": "ohos.permission.APPROXIMATELY_LOCATION", ... }
]
```

**Map Kit needs AppGallery Connect setup**: enable the map service under API
management, then complete manual signing. The document spells out the five
console steps - worth following before assuming the code is at fault when the
map renders blank.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. The industry's architecture practice
  requires API 24.
- AGC: map service enabled **and** a manual signing profile.
- `zoom` 2–20, `tilt` 0–75, `bearing` [0, 360).
- Camera coordinates are GCJ02; `geoLocationManager` returns WGS84. Convert.
- `mapController` is null until the map callback fires; every method here
  null-checks it, and so should yours.

## Pitfalls

- **`HW-06-0044` — locate-me is gated on a reverse-geocode whose result is
  discarded.** If the geocoder is unavailable or the lookup fails, the map never
  moves - even though the coordinates were already in hand.
- **`HW-06-0045` — zoom is unclamped upward and under-clamped downward.** The
  documented range is 2–20; the guard permits 0 and 1 and nothing stops the
  value climbing past 20. Five taps from the default leaves the range.
- **`HW-06-0046` — the permission request is not awaited** before the location
  lookup it authorises, so the first tap always fails silently. The same file
  awaits it correctly in `aboutToAppear`.
- **`HW-06-0047` — the document's step 2 calls the permission helper with no
  argument** while its step 1 declares it takes a context.
- **`HW-06-0048` — locating resets the zoom to the default constant** and drops
  the bearing, discarding the view the user had set.

## References

- `documentation/harmonyos-references/03_application-services/map-common.md` - `CameraPosition`; `zoom` 2–20, `tilt` 0–75, `bearing` [0, 360)
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-common
- `documentation/harmonyos-guides/07_application-services/map-controls-and-gestures.md` - gesture enable/disable APIs
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-controls-and-gestures
- `documentation/harmonyos-guides/04_application-services/map-config-agc.md` - AGC prerequisites
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-config-agc
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-toggle.md` - `Toggle`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-toggle
- `documentation/harmonyos-guides/04_system/request-app-permissions.md` - `requestPermissionOnSetting`, and why it is needed after a refusal
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-app-permissions
- `TRANS-04` - the other permission ladder in this industry; this one is the better of the two
