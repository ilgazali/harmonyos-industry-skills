---
id: KIDS-12
title: Child location and trail - an animated trace overlay with a docked address panel
industry: 08_children_education
doc: huawei_industry_tree/08_children_education/docs/12_map_location.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_location-0000002385607421
sample: huawei_industry_tree/08_children_education/downloads/MapLocation.zip
kits: ["@kit.MapKit", "@kit.LocationKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [MapComponent, "map.MapComponentController", "mapCommon.MapOptions", "mapCommon.MarkerOptions", addMarker, addTraceOverlay, "mapCommon.TraceOverlayParams", "map.PlayImageAnimation", setAnimation, startAnimation, "geoLocationManager.getAddressesFromLocation", ReverseGeoCodeRequest, bindSheet, detents, onWillDismiss, setInterval, "@StorageProp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-08-0091, HW-08-0092, HW-08-0093, HW-08-0094, HW-08-0095, HW-08-0096, HW-08-0120]
status: verified-with-fixes
---

## When to use

Load this card when a map has to **play back a journey** - a child's route
home, a delivery, a run, a field trip - with a marker that travels the path and
a caption naming where it is.

The one API that does the heavy lifting is **`addTraceOverlay`**, and it is
worth knowing because it replaces a hand-rolled animation loop entirely:

```typescript
let traceOptions: mapCommon.TraceOverlayParams = {
  points: MAP_LOCATION,        // the whole path, up front
  animationDuration: 5000,     // it plays itself
  isMapMoving: true,           // and the camera follows
  color: 0xFFA5D61D,
  width: 20
};
await this.mapController.addTraceOverlay(traceOptions, markers);
```

One call takes the full point list, draws the trail, animates it, moves the
camera with it, and carries the markers you hand it along the path. Compare
`SPORT-11`, which adds a two-point overlay per location fix and leaks one every
time - `addTraceOverlay` is what that sample should have used.

The second idea is the **permanently docked sheet**: `bindSheet(true, …)` with
`onWillDismiss: () => false` and a `detents` height, giving a panel that cannot
be swiped away.

**Six findings**, and the first two are the ones that stop it working: the
module declares no permissions at all, and a one-second timer is never cleared.

## Feature checklist

- A full-screen map centred on the start of the route.
- An animated marker travelling a green trail.
- A docked panel naming the district and street, updated as the marker moves.
- The camera follows the animation.

## Architecture

One `entry` module, one page, one mock data file. 407 lines.

```
entry/src/main/ets
├── constants/StyleConstants.ets   sizes and weights
├── entryability/EntryAbility.ets
├── entrybackupability/
├── mock/LngLat.ets                MAP_LOCATION: mapCommon.LatLng[]
└── pages/MapPage.ets              the map, the overlay, the timer, the panel
```

The documented tree matches the zip.

**The map is built from two fields set before the component renders:**

```typescript
private mapOptions?: mapCommon.MapOptions;              // where to start
private callback?: AsyncCallback<map.MapComponentController>;  // what to do once ready
private mapController?: map.MapComponentController;     // captured in the callback
```

`MapComponent` takes both; the controller only exists inside the callback, so
everything that touches the map - markers, overlays - has to happen there or
after it. That ordering is the thing to copy.

**Two independent timelines drive one journey**, which is the design flaw
worth understanding before reusing this:

```
addTraceOverlay(animationDuration: 5000)   ──> marker travels the trail in 5 s
setInterval(TIME_INTERVAL = 1000)          ──> address steps through the points
                                               in ceil(6) + 1 = 7 ticks = 7 s
```

Nothing derives one from the other (`HW-08-0094`).

## Implementation steps

1. **Declare `INTERNET` and `GET_NETWORK_INFO`** in `module.json5`
   (`HW-08-0091`).
2. **Put a placeholder `client_id`** at *module* level, not inside an ability
   (`HW-08-0093`).
3. **Set `mapOptions` and the callback in `aboutToAppear`**, before the
   component builds.
4. **Capture the controller in the callback** and do all map work there.
5. **Add the marker, then the trace overlay**, passing the marker so it travels
   the path.
6. **Drive the caption from the same clock as the animation** (`HW-08-0094`).
7. **Clear the interval and release the overlay in `aboutToDisappear`**
   (`HW-08-0092`, `HW-08-0096`).

## Verified snippets

All snippets are from `MapLocation.zip`. Corrected forms are marked.

**Setting up the map — `entry/src/main/ets/pages/MapPage.ets`** (as shipped)

```typescript
aboutToAppear(): void {
  this.mapOptions = {
    position: {
      target: { latitude: this.latitude, longitude: this.longitude },
      zoom: 17
    }
  };

  this.callback = async (err, mapController) => {
    if (!err) {
      this.mapController = mapController;          // the ONLY place it arrives

      let markerOptions: mapCommon.MarkerOptions = {
        position: { latitude: this.latitude, longitude: this.longitude },
        visible: true
      };
      let markerChild = await this.mapController.addMarker(markerOptions);

      // ... animation set-up ...

      let traceOptions: mapCommon.TraceOverlayParams = {
        points: MAP_LOCATION,
        animationDuration: 5000,
        isMapMoving: true,
        color: 0xFFA5D61D,
        width: 20
      };
      let markers: Array<map.Marker> = [];
      markers.push(markerChild);
      await this.mapController.addTraceOverlay(traceOptions, markers);   // marker rides the trail
    }
  };
  this.convertLatToPosition(this.longitude, this.latitude);
  this.changePosition();
}
```

**`if (!err)` guards the whole body.** The callback is an `AsyncCallback`, so
failure arrives as a populated `err` with an unusable controller - checking it
first is what stops every subsequent `await` from running against `undefined`.

**The marker is passed to `addTraceOverlay`, not moved by hand.** That second
parameter is why no per-frame position update exists anywhere in the file: the
overlay owns the marker's travel for the duration of the animation.

**`isMapMoving: true` is the camera.** Without it the trail animates off-screen
as soon as it leaves the initial viewport at zoom 17.

**`color: 0xFFA5D61D` is ARGB, not RGB** - the leading `FF` is opacity. Dropping
it yields a fully transparent trail.

**The docked panel — same file** (as shipped)

```typescript
MapComponent({ mapOptions: this.mapOptions, mapCallback: this.callback })
  .width(StyleConstants.FULL)
  .height(StyleConstants.FULL)
  .bindSheet(true, this.PositionText(), {
    showClose: false,
    backgroundColor: $r('app.color.gray_back'),
    detents: [137],                        // one fixed height, no expansion
    enableOutsideInteractive: true,        // the map stays usable behind it
    onWillDismiss: () => {
      return false;                        // refuse every dismissal
    }
  });
```

**This is the one place a literal `true` on `bindSheet` is correct.** Everywhere
else the flag needs `$$` so the component can write back when the sheet is
swiped away - here it never can be, because `onWillDismiss` returns `false`.
The three options work as a set: no close button, one detent so it cannot be
dragged taller, and a refused dismissal.

**`enableOutsideInteractive: true` is what makes it a dock rather than a
modal** - without it the sheet would swallow taps on the map underneath.

**The stepping timer — same file** (corrected, see `HW-08-0092` and `HW-08-0094`)

```typescript
changePosition() {
  // FIX: the sample derives nothing from animationDuration and clamps at the
  // end instead of stopping, so it geocodes the final point once a second
  // forever.
  const step = Math.floor(MAP_LOCATION.length / INTERVAL) + 1;
  this.timer = setInterval(() => {
    if (this.index >= MAP_LOCATION.length) {
      clearInterval(this.timer);           // FIX: the sample clamps and continues
      return;
    }
    this.longitude = MAP_LOCATION[this.index].longitude;
    this.latitude = MAP_LOCATION[this.index].latitude;
    this.convertLatToPosition(this.longitude, this.latitude);
    this.index += step;
  }, TIME_INTERVAL);
}

// FIX: the sample declares no aboutToDisappear at all.
aboutToDisappear(): void {
  clearInterval(this.timer);
}
```

**Reverse geocoding — same file** (corrected, see `HW-08-0095`)

```typescript
convertLatToPosition(longitude: number, latitude: number) {
  let reverseGeocodeRequest: geoLocationManager.ReverseGeoCodeRequest = {
    latitude: latitude, longitude: longitude, maxItems: 1
  };
  try {
    geoLocationManager.getAddressesFromLocation(reverseGeocodeRequest).then((data) => {
      // FIX: the sample does `data[0].placeName!` with no length check.
      // An empty array is a RESOLVED promise, so the catch below does not cover it.
      if (!data || data.length === 0) {
        return;
      }
      this.address = data[0].placeName ?? '';
      this.area = `${data[0].locality ?? ''}${data[0].subLocality ?? ''}`;
    }).catch((error: BusinessError) => {
      hilog.error(DOMAIN, TAG, 'promise, getAddressesFromLocation: error=' + JSON.stringify(error));
    });
  } catch (err) {
    hilog.error(DOMAIN, TAG, 'errCode:' + JSON.stringify(err));
  }
}
```

**Both a `try` and a `.catch()` are needed** and the sample has both -
`getAddressesFromLocation` can throw synchronously on a malformed request and
reject asynchronously on a service failure. What neither covers is the third
case: success with nothing in it.

**`maxItems: 1`** bounds the array above, which is why `data[0]` looks safe. It
says nothing about the lower bound.

## Permissions & config

**None declared, and that is the blocker.** `module.json5` has no
`requestPermissions` block at all, while Map Kit needs `ohos.permission.INTERNET`
and `ohos.permission.GET_NETWORK_INFO` to fetch tiles - both `system_grant`, so
declaring them costs no dialog (`HW-08-0091`).

The `client_id` metadata **is placed correctly** - at module level, beside
`abilities`, not nested inside one:

```json
"metadata": [
  { "name": "client_id", "value": "6917577716846984139" }
]
```

That placement is what `SPORT-01` gets wrong. What this sample gets wrong is
shipping a live value where `MotionTrajectory` ships `"***********"`
(`HW-08-0093`).

The document devotes a **说明** section to the AppGallery Connect setup - enable
Map Service under API management, then sign manually. That step is mandatory
and is the second cause listed on the Map Kit "map not displayed" page.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Requires Map Service enabled in AppGallery Connect and a manual signing
  configuration.** Without both, the map does not render regardless of the code.
- **The route is mock data.** `MAP_LOCATION` in `mock/LngLat.ets` is a fixed
  list of coordinates in Shanghai; nothing reads a real device position, and
  there is no Location Kit positioning call anywhere - only reverse geocoding of
  the mock points.
- **The journey plays once, at a fixed speed**, and cannot be paused, restarted
  or scrubbed.
- **The marker "animation" is a single frame** looped forever, so nothing about
  it moves (`HW-08-0096`).
- Nothing is released: no `aboutToDisappear`, no `clearInterval`, no
  `stopAnimation`, no overlay teardown.
- The page reads `bottomRectHeight` from `AppStorage` and applies it with
  `px2vp`, but never reads the top inset, so the map runs under the status bar.

## Pitfalls

- **`HW-08-0091` — the module declares no permissions at all,** so the
  `INTERNET` and `GET_NETWORK_INFO` that Map Kit needs are missing and the map -
  the only content on the page - cannot load. The Map Kit FAQ opens with exactly
  this.
- **`HW-08-0092` — the one-second interval is never cleared.** There is no
  `aboutToDisappear` and no `clearInterval` anywhere; the end-of-track branch
  clamps rather than stopping, so the final coordinate is reverse-geocoded once
  a second for the life of the process - on a child's device, in an app whose
  sibling feature monitors data usage.
- **`HW-08-0093` — a live nineteen-digit `client_id` is committed** where the
  sibling map sample ships a mask. Readers issue Map Kit traffic against
  someone else's project and never perform the setup the document describes.
- **`HW-08-0094` — the trail animation (5000 ms) and the address readout
  (7 x 1000 ms) are two unsynchronised clocks over one journey,** so the caption
  names a place the marker is not at.
- **`HW-08-0095` — `data[0].placeName!` with no length check.** An empty
  geocoding result is a *resolved* promise, so the `.catch()` beside it does not
  apply, and the optional `locality`/`subLocality` interpolate as `undefined`
  into the only caption on screen.
- **`HW-08-0096` — the marker's `PlayImageAnimation` holds one frame** and
  repeats forever, so nothing animates; neither it nor the trace overlay is
  released.

## References

- `documentation/harmonyos-guides/07_application-services/map-faq-1.md` - "Map Not Displayed" and the required permissions
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-faq-1
- `documentation/harmonyos-guides/04_application-services/map-config-agc.md` - enabling Map Service and the `client_id`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-config-agc
- `documentation/harmonyos-guides/07_application-services/map-dyntrajectories.md` - dynamic trajectories
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-dyntrajectories
- `documentation/harmonyos-references/06_application-services/map-mapcomponent.md` - `MapComponent`, `mapOptions`, `mapCallback`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-mapcomponent
- `documentation/harmonyos-references/06_application-services/map-map-mapcomponentcontroller.md` - `addMarker`, `addTraceOverlay`, `TraceOverlayParams`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-map-mapcomponentcontroller
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - `bindSheet`, `detents`, `onWillDismiss`, `enableOutsideInteractive`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
- `documentation/harmonyos-references/05_common-capabilities/js-apis-timer.md` - `setInterval`, `clearInterval`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-timer
- `SPORT-11` - the other trail sample, which adds a two-point overlay per fix instead of using `addTraceOverlay`
- `SPORT-08` - the other Map Kit sample missing its network permissions
- `KIDS-14` - the industry's other Location Kit sample, which does take real positions
