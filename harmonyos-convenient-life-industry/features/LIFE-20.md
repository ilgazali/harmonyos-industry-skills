---
id: LIFE-20
title: Rotate the map to the heading, then travel along it - animateCamera for the turn, animateCameraWithMarker for the move
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/20_confirm_direction_in_map_to_rotate_and_move.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/confirm_direction_in_map_to_rotate_and_move-0000002335234358
sample: huawei_industry_tree/02_convenient_life/downloads/ConfirmDirectionInMap.zip
kits: ["@kit.MapKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.ImageKit"]
apis: [MapComponent, "map.MapComponentController", "map.newCameraPosition", "map.CameraUpdate", "controller.animateCamera", "controller.animateCameraWithMarker", "controller.addMarker", "controller.getEventManager", "MapEventManager.on", "MapEventManager.off", "marker.setPosition", "marker.setAlpha", "mapCommon.CameraPosition", "mapCommon.MyLocationStyle", MyLocationDisplayType, "controller.setMyLocationEnabled", "controller.setMyLocationControlsEnabled", "controller.setZoomControlsEnabled", "controller.setScalePosition", "controller.setCompassControlsEnabled", "controller.setLogoAlignment", "controller.setPadding", "controller.setAllGesturesEnabled", "abilityAccessCtrl.createAtManager", "atManager.checkAccessToken", "atManager.requestPermissionsFromUser", "bundleManager.getBundleInfoForSelf", setInterval, clearInterval, onPageShow, onPageHide]
permissions: ["ohos.permission.INTERNET", "ohos.permission.APPROXIMATELY_LOCATION", "ohos.permission.LOCATION"]
min_api: 20
modules: [entry]
findings: [HW-02-0142, HW-02-0143, HW-02-0144, HW-02-0145, HW-02-0146, HW-02-0147, HW-02-0148, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card when the map should **turn to face the direction of travel and
then move along it** - the "heading-up" behaviour of a navigation view, where
north is not up and the route always points forward.

Two `MapComponentController` calls do it, and the split between them is the
idea:

- **`animateCamera(cameraUpdate, duration)`** changes the camera - here only the
  `bearing`, which rotates the whole map under a fixed centre.
- **`animateCameraWithMarker(cameraUpdate, marker, duration)`** does the same
  *and* carries a marker along with the camera, so the "you are here" pin travels
  with the view instead of being repositioned afterwards.

The bearing itself is one line of trigonometry: `atan2(Δlongitude, Δlatitude)`
in degrees, negated, because a map bearing is measured clockwise from north
while `atan2` is counter-clockwise.

This sample is the weakest in the industry on everything around that core: its
permissions are requested but never declared (`HW-02-0142`), and the sequencing
is three nested `setInterval`s where one `await` would do (`HW-02-0144`,
`HW-02-0147`).

Take this for heading-up navigation, delivery tracking, any "advance to the next
waypoint" animation. `MapUtil.ets` is also the most complete map-configuration
reference in this industry - zoom, scale, compass, logo, padding and gestures all
in one place.

## Feature checklist

- A full-screen map with zoom, scale and compass controls, the logo repositioned
  and all gestures enabled.
- A start marker and a target marker, the target initially transparent.
- One tap rotates the map so the start-to-target line points up the screen.
- The camera then travels to the target, carrying the start marker with it.
- On arrival the target marker fades in and the start marker is re-anchored
  there, ready for the next leg.
- The my-location layer and button are enabled once location permission is held.
- The map is suspended in `onPageHide` and resumed in `onPageShow`; all map
  listeners are released in `aboutToDisappear`.

## Architecture

One `entry` module. Everything reusable is in `utils/`, which is the reason to
read this sample.

```
entry/src/main/ets
├── common/Constants.ets       POSITIONS, PERMISSION_LIST, durations, zoom, marker alphas
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── pages/MainPage.ets         THE CARD: the map, the two-phase animation, the lifecycle
└── utils
    ├── CalculateUtil.ets      calculateAngle - the whole bearing computation, 8 lines
    ├── Logger.ets             hilog wrapper
    ├── MapUtil.ets            camera, markers, listeners, controls, padding, my-location
    └── PermissionsUtil.ets    check / request  (both defective - HW-02-0145, HW-02-0146)
```

The documented tree matches the zip exactly.

**All four utils are module-scope singletons** (`MAP_UTIL`, `CALCULATE_UTIL`,
`LOGGER`, and the permissions helper), which is fine for the stateless ones and
is a problem for `MapUtil`, because it carries the mutable
`cameraMoveFinished` flag (`HW-02-0147`).

**The two-phase animation, as shipped:**

```
onPageShow -> setInterval(3000)  -> rotateAndMove(), then clear itself

rotateAndMove()
  rotateAngle = -calculateAngle(current, target)
  setInterval(1000):
    tick 1  -> rotateToAngle(current, rotateAngle)          animateCamera, 500 ms
    tick 2  -> moveCameraToPositionWithMarker(target, ...)  animateCameraWithMarker, 1000 ms
               setInterval(100): poll MAP_UTIL.cameraMoveFinished
                 -> targetMarker.setAlpha(...); current = target; startMarker.setPosition(current)
               clear the outer interval
```

**Three timers, none of whose handles the component keeps** - which is the
defect in `HW-02-0144`. The whole construction exists only because
`moveCameraToPositionWithMarker` drops the promise it should return
(`HW-02-0147`); with that fixed, the sequence is two `await`s.

**The completion signal comes from the map, not from the promise.**
`MapUtil.setMapEventListener` subscribes to `cameraMove` and `cameraIdle` and
toggles `cameraMoveFinished` between them - a reasonable mechanism in itself,
and the reason `releaseMapEventListener` exists. It is also why leaving the page
mid-move strands the inner poll forever.

## Implementation steps

1. **Declare the permissions before requesting them** (`HW-02-0142`). A
   `requestPermissionsFromUser` for an undeclared permission resolves with
   `authResults` 2 and shows no dialog.
2. **Compute the bearing with `atan2(Δlng, Δlat) * 180 / Math.PI`, and negate
   it.** Longitude first, latitude second - the argument order is what makes the
   result a compass bearing rather than a mathematical angle.
3. **Configure the map once in the init callback** - controls, padding, logo,
   gestures, my-location style - then subscribe to the camera events.
4. **Rotate with `animateCamera`,** passing a `CameraPosition` whose `target` is
   the current position and whose `bearing` is the new heading. Keeping the
   target fixed is what makes it a rotation rather than a pan.
5. **Move with `animateCameraWithMarker`,** so the marker travels with the
   camera instead of jumping when the animation ends.
6. **Await the move.** Return the promise from the wrapper (`HW-02-0147`) and the
   polling interval disappears entirely.
7. **If you must poll, keep every timer handle on the component** and clear them
   all in `aboutToDisappear` (`HW-02-0144`).
8. **Release the map listeners in `aboutToDisappear`** - this sample is the only
   one in the industry that does, and it is worth copying.
9. **Check every permission, not any** (`HW-02-0145`), and inspect `authResults`
   (`HW-02-0146`).

## Verified snippets

All snippets are from `ConfirmDirectionInMap.zip`. Corrected forms are marked.

**The bearing - `ConfirmDirectionInMap.zip#entry/src/main/ets/utils/CalculateUtil.ets`** (as shipped)

```typescript
class CalculateUtil {
  calculateAngle(startPosition: mapCommon.LatLng, targetPosition: mapCommon.LatLng): number {
    // 一段路线起点和终点两端的经纬度坐标相减
    let offsetLng: number = targetPosition.longitude - startPosition.longitude;
    let offsetLat: number = targetPosition.latitude - startPosition.latitude;
    // 通过反正切函数得到的弧度值再转角度
    let angle: number = Math.atan2(offsetLng, offsetLat) * 180 / Math.PI;
    return angle;
  }
}
```

**`atan2(Δlng, Δlat)`, not the usual `atan2(y, x)`.** Passing longitude as the
first argument and latitude as the second measures the angle **from north,
clockwise** - which is exactly what a map bearing is. The conventional
`atan2(Δlat, Δlng)` would measure from east, counter-clockwise, and every heading
would be 90 degrees out and mirrored.

**The caller negates it:** `MainPage.ets:177` does
`this.rotateAngle = -CALCULATE_UTIL.calculateAngle(...)`. Rotating the *camera*
to face a heading means turning the map the opposite way, so the sign flip is at
the call site rather than in the helper - worth knowing if you reuse
`calculateAngle` for a marker's own `rotation`, where you want it unnegated.

This is a planar approximation over degrees, which is accurate enough for the
short legs a navigation view animates and wrong for long distances, where the
great-circle initial bearing needs the spherical formula.

**Rotating in place - `ConfirmDirectionInMap.zip#entry/src/main/ets/utils/MapUtil.ets:29`** (as shipped)

```typescript
public rotateToAngle(position: mapCommon.LatLng, angle: number,
  mapController: map.MapComponentController): boolean {
  let cameraPostion: mapCommon.CameraPosition = {
    target: { latitude: position.latitude, longitude: position.longitude },   // unchanged - so it turns
    zoom: Constants.ZOOM_16,
    bearing: angle
  };
  let cameraUpdate: map.CameraUpdate = map.newCameraPosition(cameraPostion);
  mapController.animateCamera(cameraUpdate, Constants.DURATION_500);
  return true;
}
```

**A rotation is a camera update whose `target` does not change.** There is no
separate rotate API - you build a `CameraPosition` with the same centre and a
new `bearing`, and `animateCamera` interpolates the turn. Change the target as
well and the same call becomes a combined pan-and-turn.

`map.newCameraPosition(...)` is the factory that wraps a `CameraPosition` into
the `CameraUpdate` the animate calls take; there are siblings for
`newLatLng`, `newLatLngBounds` and so on when you only want to move.

**Moving with the marker - same file, line 95** (corrected, see `HW-02-0147`)

```typescript
public async rotateCameraToAngleWithMarker(
  position: mapCommon.LatLng, angle: number, marker: map.Marker,
  mapController: map.MapComponentController): Promise<void> {
  let cameraPostion: mapCommon.CameraPosition = {
    target: { latitude: position.latitude, longitude: position.longitude },
    zoom: Constants.ZOOM_16,
    bearing: angle
  };
  let cameraUpdate: map.CameraUpdate = map.newCameraPosition(cameraPostion);
  // 在1000ms内以动画的形式移动相机, 并更新指定的marker
  await mapController.animateCameraWithMarker(cameraUpdate, marker, Constants.DURATION_1000);
}

public async moveCameraToPositionWithMarker(position: mapCommon.LatLng, marker: map.Marker,
  angle: number, mapController: map.MapComponentController): Promise<void> {
  // FIX: the sample calls this without await and without returning it, so the
  //      Promise<void> it advertises resolves before the animation has begun.
  return this.rotateCameraToAngleWithMarker(position, angle, marker, mapController);
}
```

**`animateCameraWithMarker` is the reason the pin does not lag.** Passing the
marker makes the map move it in step with the camera for the whole 1000 ms;
calling `animateCamera` and then `marker.setPosition` would snap the pin at the
end. The marker still has to be re-anchored afterwards - the call moves it
visually, and `setPosition` commits the new coordinate.

**The wrapper's missing `return` is what forces the polling.** With it, the
caller can `await` and the entire nested-interval construction below collapses.

**The two-phase sequence - `ConfirmDirectionInMap.zip#entry/src/main/ets/pages/MainPage.ets:176`** (corrected, see `HW-02-0144` and `HW-02-0147`)

```typescript
// FIX: the sample nests two setIntervals - a 1000 ms one that runs the rotation on
//      its first tick and the move on its second, and a 100 ms one polling
//      MAP_UTIL.cameraMoveFinished. All three handles (including the 3000 ms one
//      started from onPageShow) are locals, so aboutToDisappear cannot clear them.
async rotateAndMove(): Promise<void> {
  if (!this.startMarker || !this.mapController) {
    return;
  }
  this.rotateAngle = -CALCULATE_UTIL.calculateAngle(this.currentPosition, this.targetPosition);

  MAP_UTIL.rotateToAngle(this.currentPosition, this.rotateAngle, this.mapController);
  await new Promise<void>((resolve) => setTimeout(resolve, Constants.DURATION_500));   // the turn

  await MAP_UTIL.moveCameraToPositionWithMarker(                                       // the move
    this.targetPosition, this.startMarker, this.rotateAngle, this.mapController);

  this.targetMarker?.setAlpha(Constants.MARKER_ALPHA[0]);
  this.currentPosition = this.targetPosition;
  this.startMarker?.setPosition(this.currentPosition);
}
```

**Why the shipped version leaks permanently.** The inner 100 ms poll exits only
when `MAP_UTIL.cameraMoveFinished` turns true, and that flag is set by the
`cameraIdle` listener. `aboutToDisappear` calls `releaseMapEventListener`, which
removes that listener - so leaving the page mid-move means the flag can never
change, the poll never reaches its `clearInterval`, and its handle is a local of
a callback that has already returned. It fires every 100 ms for the life of the
process, touching markers on a destroyed component.

The 3000 ms timer in `onPageShow` clears itself on its first tick, which is what
`setTimeout` is for.

**Camera events as a completion signal - `ConfirmDirectionInMap.zip#entry/src/main/ets/utils/MapUtil.ets:126`** (as shipped)

```typescript
public setMapEventListener(mapEventManager: map.MapEventManager, prompt: PromptAction): void {
  mapEventManager.on('cameraChange', (position: mapCommon.LatLng) => { LOGGER.info(...); });
  mapEventManager.on('cameraIdle', () => {
    this.cameraMoveFinished = true;            // the move has settled
  });
  mapEventManager.on('cameraMoveCancel', () => { LOGGER.info(...); });
  mapEventManager.on('cameraMove', () => {
    this.cameraMoveFinished = false;           // a move has started
  });
  mapEventManager.on('myLocationButtonClick', () => {
    prompt.showToast({ message: $r('app.string.only_show') });
  });
}

public releaseMapEventListener(mapEventManager: map.MapEventManager): void {
  mapEventManager.off('cameraChange');
  mapEventManager.off('cameraIdle');
  mapEventManager.off('cameraMoveCancel');
  mapEventManager.off('cameraMove');
  mapEventManager.off('myLocationButtonClick');
}
```

**This pair is the best thing in the sample and the only one of its kind in this
industry.** `MapEventManager.off` exists and every registration here has a
matching release, called from `aboutToDisappear` at `MainPage.ets:81` - the
sibling map sample in `LIFE-14` registers two listeners and releases neither
(`HW-02-0149`).

**`cameraMove` / `cameraIdle` bracket every camera animation,** which is a
genuinely useful signal - it covers user gestures as well as programmatic moves,
which an awaited promise does not. The mistake is storing the flag on a
module-scope singleton (`HW-02-0148`), so two pages sharing `MAP_UTIL` would
overwrite each other's state.

**Configuring the map once - same file, line 162** (as shipped)

```typescript
public setControllersEnabled(mapController: map.MapComponentController): void {
  mapController?.setZoomControlsEnabled(true);
  mapController.setScalePosition({ positionX: Constants.SCALE_POSITION[0],
                                   positionY: Constants.SCALE_POSITION[1] });   // px from top-left
  mapController.setScaleControlsEnabled(true);
  mapController.setCompassControlsEnabled(true);
  mapController.setCompassPosition({ positionX: Constants.COMPASS_POSITION[0],
                                     positionY: Constants.COMPASS_POSITION[1] });
  mapController.setLogoAlignment(mapCommon.LogoAlignment.BOTTOM_START);
  mapController.setLogoPadding({ left: Constants.MAP_PADDING_50,
                                 bottom: Constants.MAP_PADDING_50 });           // px
  mapController.setAllGesturesEnabled(true);
}

public setMapPadding(top: number, bottom: number, mapController: map.MapComponentController): void {
  mapController.setPadding({ top, left: Constants.MAP_PADDING_50,
                             right: Constants.MAP_PADDING_50, bottom });
}
```

**`setPadding` defines the *visible* region, not a margin.** It insets the area
the camera centres on, so a panel overlaying the bottom of the map no longer
hides the point the camera is looking at - the right tool whenever a sheet or a
card sits over the map, as in `LIFE-14`.

**All the control positions are in px, not vp** - `setScalePosition`,
`setCompassPosition` and `setLogoPadding` alike. The comments in the source say
so explicitly (`向右移动100px`), which is why these constants are large numbers.

`LogoAlignment.BOTTOM_START` moves the mandatory map logo out of the way rather
than hiding it; it cannot be removed.

**Permissions - `ConfirmDirectionInMap.zip#entry/src/main/ets/utils/PermissionsUtil.ets:24`** (corrected, see `HW-02-0145` and `HW-02-0146`)

```typescript
// FIX: the sample returns true on the FIRST granted permission and enables the
//      my-location layer from inside the check.
async checkPermissions(): Promise<boolean> {
  for (const permission of Constants.PERMISSION_LIST) {
    if (await this.checkAccessToken(permission) !== abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED) {
      return false;
    }
  }
  return true;
}

requestPermissions(uiContext: common.UIAbilityContext, mapController: map.MapComponentController): void {
  const atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
  atManager.requestPermissionsFromUser(uiContext, Constants.PERMISSION_LIST)
    .then((data: PermissionRequestResult) => {
      LOGGER.info(`authResults: ${JSON.stringify(data.authResults)}`);   // FIX: sample logs the object
      if (data.authResults.every((r: number) => r === 0)) {              // FIX: sample ignores results
        mapController.setMyLocationEnabled(true);
        mapController.setMyLocationControlsEnabled(true);
      }
    })
    .catch((err: BusinessError) => {
      LOGGER.error(`Failed to request permissions. Code ${err.code}, message ${err.message}`);
    });
}
```

**None of this can work from the zip.** `module.json5` declares no
`requestPermissions` block at all, so every entry in `authResults` comes back as
2 - "The permission is not declared in the configuration file" - with no dialog
shown, and the shipped `.then` enables the my-location layer anyway
(`HW-02-0142`).

`checkAccessToken` going through
`bundleManager.getBundleInfoForSelf(... GET_BUNDLE_INFO_WITH_APPLICATION)` for
`appInfo.accessTokenId` is the documented way to self-check and is worth copying
verbatim.

## Permissions & config

`ConfirmDirectionInMap.zip#entry/src/main/module.json5` — **as shipped there is
no permission block at all**, which is `HW-02-0142`:

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "pages": "$profile:main_pages",
    "abilities": [{ /* EntryAbility, entity.system.home */ }],
    "extensionAbilities": [{ /* EntryBackupAbility */ }]
    // FIX: add the block below - the sample requests these two at runtime
    // "requestPermissions": [
    //   { "name": "ohos.permission.INTERNET" },
    //   { "name": "ohos.permission.APPROXIMATELY_LOCATION",
    //     "reason": "$string:...", "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" } },
    //   { "name": "ohos.permission.LOCATION",
    //     "reason": "$string:...", "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" } }
    // ]
  }
}
```

The document has no 权限说明 section either (`HW-02-0143`), unlike `LIFE-14` and
`LIFE-16`, which both list all three.

Notably, this sample carries **no `client_id` metadata** - unlike `LIFE-14` and
`LIFE-16`, which both ship the sample author's AGC project id. That is the
correct configuration for a project targeting `6.0.0(20)`: the Map Kit guide
says the client ID has not been needed since HarmonyOS 5.0.2(14).

Map Kit still requires enabling the service in AppGallery Connect and a manual
signing configuration - the document's 说明 block at lines 93-98.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 102-104).
- **Map Kit must be enabled in AppGallery Connect and the build manually
  signed**; Map Kit is a Chinese-mainland service.
- The route is two hard-coded points, `Constants.POSITIONS` =
  `{118.78, 31.965}` and `{118.79, 31.975}` - there is no route source and no
  third leg.
- `calculateAngle` is a planar approximation over degrees. Accurate for short
  legs, wrong over long distances and near the poles, where the great-circle
  initial bearing is needed.
- The control positions (`setScalePosition`, `setCompassPosition`,
  `setLogoPadding`, `setPadding`) are all in **px**, so they do not scale with
  display density.
- `MapUtil` is a module-scope singleton holding `cameraMoveFinished`, so only
  one map animation can be tracked per process.
- The my-location button is wired to a toast, not to a real recentre.

## Pitfalls

- **`HW-02-0142` - the two location permissions are requested but never
  declared.** `module.json5` has no `requestPermissions` block, so
  `authResults` is 2 for both, no dialog appears, and the `.then` enables the
  my-location layer anyway. The sample cannot work as shipped.
- **`HW-02-0143` - the document has no 权限说明 section,** although the sample
  requests location permissions and both sibling Map Kit documents in this
  industry carry one.
- **`HW-02-0144` - three `setInterval` handles are locals,** and the innermost
  polls a flag that `aboutToDisappear`'s own listener release stops updating -
  so leaving the page mid-move leaves a 100 ms timer running forever with no way
  to stop it.
- **`HW-02-0145` - `checkPermissions` returns true on the first granted
  permission** (an OR where the reference requires both), and enables the map's
  location layer from inside a function named `check`.
- **`HW-02-0146` - the request ignores `authResults`** and logs the result
  object as `[object Object]`.
- **`HW-02-0147` - `moveCameraToPositionWithMarker` does not await or return the
  call it delegates to,** so its `Promise<void>` resolves before the animation
  starts. Fixing this removes the need for the polling interval entirely.
- **`HW-02-0148` - `MyLocationStyle.icon` is set to `''`,** which is neither a
  rawfile-relative path nor a data URL. Omit the property to get the default
  marker.
- **Do not swap the arguments to `atan2`.** `atan2(Δlng, Δlat)` gives a compass
  bearing; the conventional order gives a mathematical angle, 90 degrees out and
  mirrored.
- **Do not forget the sign.** Turning the camera to face a heading is the
  negation of the heading.
- **Do not use `animateCamera` plus `setPosition` to move a marker with the
  view.** `animateCameraWithMarker` exists so the pin travels rather than snaps.
- **Do copy `releaseMapEventListener`.** It is the only correct map-listener
  teardown in this industry.

## References

- `documentation/harmonyos-references/06_application-services/map-map-mapcomponentcontroller.md` - `animateCamera`, `animateCameraWithMarker`, `addMarker`, `getEventManager`, `setPadding`, `setZoomControlsEnabled`, `setScalePosition`, `setCompassPosition`, `setLogoAlignment`, `setAllGesturesEnabled`, `setMyLocationEnabled`, `setMyLocationControlsEnabled`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-map-mapcomponentcontroller
- `documentation/harmonyos-references/03_application-services/map-common.md` - `CameraPosition` (`target`, `zoom`, `bearing`), `MarkerOptions`, `MyLocationStyle`, `MyLocationDisplayType.FOLLOW_ROTATE`, `MapPoint`, `Padding`, `LogoAlignment`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-common
- `documentation/harmonyos-references/03_application-services/map-map-marker.md` - `Marker.setPosition`, `setAlpha`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-map-marker
- `documentation/harmonyos-guides/04_application-services/map-listening.md` - `MapEventManager` events: `cameraMove`, `cameraIdle`, `cameraChange`, `cameraMoveCancel`, `myLocationButtonClick`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-listening
- `documentation/harmonyos-guides/04_application-services/map-config-agc.md` - enabling Map Kit; the client ID has not been required since HarmonyOS 5.0.2(14)
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-config-agc
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `ohos.permission.LOCATION` and its `APPROXIMATELY_LOCATION` prerequisite
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `documentation/harmonyos-references/02_application-framework/js-apis-permissionrequestresult.md` - `authResults` values 0 / -1 / 2, including "the permission is not declared in the configuration file"
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-permissionrequestresult
- `LIFE-14` - the docked half-modal over a map; its listeners are the ones this sample's `releaseMapEventListener` shows how to release
- `LIFE-16` - route planning between two searched addresses, the other half of a navigation screen
- `LIFE-26`, `LIFE-29` - the industry's remaining Map Kit scenarios
