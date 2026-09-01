---
id: SPORT-08
title: Fit a whole route on screen - newLatLngBounds plus an animated TraceOverlay
industry: 03_sports_health
doc: huawei_industry_tree/03_sports_health/docs/08_max_display_of_routes.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/max_display_of_routes-0000002335684832
sample: huawei_industry_tree/03_sports_health/downloads/MaxDisplayOfRoutes.zip
kits: ["@kit.MapKit", "@kit.BasicServicesKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit"]
apis: [MapComponent, "map.MapComponentController", "map.newLatLngBounds", moveCamera, "mapCommon.LatLngBounds", addTraceOverlay, "mapCommon.TraceOverlayParams", animationCallback, addMarker, "mapCommon.MapType", scaleControlsEnabled, "map.MapEventManager", NavPathStack, getParamByName, "@Consume", "@StorageProp"]
permissions: ["ohos.permission.INTERNET", "ohos.permission.GET_NETWORK_INFO"]
min_api: 20
modules: [entry]
findings: [HW-03-0032, HW-03-0033, HW-03-0034, HW-03-0035, HW-03-0055]
status: verified-with-fixes
---

## When to use

Load this card when a **recorded track must be shown whole, framed and
replayed** - a finished run or ride, a delivery route, a walking tour, a
flight path. The two questions it answers are the ones every track view has:
*how do I fit an arbitrary polyline into the viewport*, and *how do I animate
it being drawn*.

Both have dedicated Map Kit answers that are easy to miss:

- **`map.newLatLngBounds(bounds, padding)`** computes the camera position and
  zoom that fit a rectangle, so no zoom level is ever guessed.
- **`addTraceOverlay(TraceOverlayParams)`** draws the polyline as a timed
  animation, with a per-point callback - so start and finish markers can be
  dropped exactly when the animation reaches them.

Pair this with `SPORT-11`, which records the track live; this card frames it
afterwards.

## Feature checklist

- A sports feed listing other users' activities.
- Tapping one opens its route on a full map.
- The camera opens framed so the whole track is visible, with padding.
- The track draws itself over a second as an animated trace.
- A start marker appears when the animation begins and a finish marker when it
  reaches the last point.
- Distance, pace and duration are shown over the map.

## Architecture

One `entry` module, two pages, three components, three model files.

```
entry/src/main/ets
├── components
│   ├── SportDataComp.ets        the metrics strip over the map
│   ├── SportMoment.ets          one feed entry
│   └── SportTab.ets             the feed's tab bar
├── entryability/EntryAbility.ets
├── entrybackupability/
├── model
│   ├── DataModel.ets            routes and sport data, keyed by nickname
│   ├── SportDataModel.ets
│   └── SportMomentModel.ets
├── pages
│   ├── HomePage.ets             the feed
│   └── RouteShow.ets            the map page
└── utils/MapUtils.ets           getExtremum over a point array
```

The documented tree matches the zip.

**The framing calculation is two steps, deliberately split:**

```
points ──> MapUtils.getExtremum ──> [maxLat, maxLon, minLat, minLon]
                                            │
                    ┌───────────────────────┴───────────────────────┐
                    │                                               │
       mapOptions.position.target                        LatLngBounds { northeast, southwest }
       = the midpoint, for the initial frame                        │
                                                    map.newLatLngBounds(bounds, 200)
                                                                    │
                                                          moveCamera(cameraUpdate)
```

The midpoint is used for the map's **initial** position so the first frame is
already near the track, and the bounds are used immediately afterwards to set
the exact zoom. Doing only the second would show a jump from the default
position; doing only the first would need a guessed zoom.

**`getExtremum` returns a bare four-element array** rather than a named type,
so the caller indexes `[0]`..`[3]` by convention. That is the design decision
behind two of this card's findings.

## Implementation steps

1. **Declare `INTERNET`, `GET_NETWORK_INFO` and the `client_id` metadata**
   (`HW-03-0032`) - nothing below works otherwise.
2. **Look the track up defensively**; an absent route must yield an empty
   array, not an unchecked cast (`HW-03-0033`).
3. **Compute the extremes in one pass** and handle the empty and single-point
   cases explicitly (`HW-03-0034`).
4. **Seed `mapOptions` with the midpoint** so the first frame is close.
5. **In the init callback, build a `LatLngBounds` and call
   `map.newLatLngBounds(bounds, padding)`**, then `moveCamera`.
6. **Add the trace overlay** with a duration and a per-point
   `animationCallback`, and drop the start and finish markers from it.
7. **Release the listener and the overlay** in `aboutToDisappear`
   (`HW-03-0035`).

## Verified snippets

All snippets are from `MaxDisplayOfRoutes.zip`. Corrected forms are marked.

**Finding the bounding box — `entry/src/main/ets/utils/MapUtils.ets`** (corrected, see `HW-03-0034`)

```typescript
/**
 * The coverage of a route, from the smaller to the larger latitude and longitude.
 * Near longitude 180 or -180, compute the eastern and western spans separately and add them.
 */
getExtremum(points: Array<mapCommon.LatLng>): Array<number> | undefined {
  if (!points.length) {
    return undefined;                     // FIX: the sample returns the sentinels below
  }
  let maxLatitude: number = -91;
  let maxLongitude: number = -181;
  let minLatitude: number = 91;
  let minLongitude: number = 181;
  for (const point of points) {
    maxLatitude = point.latitude > maxLatitude ? point.latitude : maxLatitude;
    maxLongitude = point.longitude > maxLongitude ? point.longitude : maxLongitude;
    minLatitude = point.latitude < minLatitude ? point.latitude : minLatitude;
    minLongitude = point.longitude < minLongitude ? point.longitude : minLongitude;
  }
  return [maxLatitude, maxLongitude, minLatitude, minLongitude];
}
```

**The comment is worth reading twice.** It records the one case this simple
min/max does not handle: a track crossing the antimeridian, where the
longitudes jump between +180 and -180 and a naive bounding box spans the whole
globe the wrong way. The stated workaround - compute the eastern and western
spans separately and combine them - is the right approach, and knowing the
limitation is documented is what makes the simple version safe to use inside
one country.

Seeding the accumulators outside the legal range (-91, 181) is the standard
one-pass trick, and it is exactly what makes the empty case dangerous.

**Framing the camera — `entry/src/main/ets/pages/RouteShow.ets`** (as shipped)

```typescript
const extremumValues = mapUtils.getExtremum(this.points);

// first frame: centre on the midpoint so the map does not open somewhere else
this.mapOptions = {
  position: {
    target: {
      latitude: (extremumValues[0] + extremumValues[2]) / 2,
      longitude: (extremumValues[1] + extremumValues[3]) / 2
    },
    zoom: 16
  },
  mapType: mapCommon.MapType.STANDARD,
  scaleControlsEnabled: true          // a scale bar: useful when the zoom is computed, not chosen
};

this.callback = async (err, mapController) => {
  if (err) {
    hilog.error(0x0001, this.TAG, `map init failed, code ${err.code}, message ${err.message}`);
    return;
  }
  this.mapController = mapController;

  const latLngBounds: mapCommon.LatLngBounds = {
    northeast: { latitude: extremumValues[0], longitude: extremumValues[1] },
    southwest: { latitude: extremumValues[2], longitude: extremumValues[3] }
  };
  const cameraUpdate = map.newLatLngBounds(latLngBounds, 200);   // 200 = padding in px
  this.mapController?.moveCamera(cameraUpdate);
  // ...
};
```

**`map.newLatLngBounds` is the API to remember.** It takes a rectangle and a
padding and produces a `CameraUpdate` at the position and zoom that fit it -
so the code never contains a zoom level for the track itself, and a 200 m run
and a 40 km ride both frame correctly. The `zoom: 16` in `mapOptions` is only
for the instant before the bounds are applied.

`scaleControlsEnabled: true` earns its place here: when the zoom is computed
rather than chosen, the user needs the scale bar to judge distance.

Note this sample **does** handle the error branch of the init callback -
unlike four of the five map samples in the tourism industry.

**Animating the trace — same file** (as shipped)

```typescript
const traceOptions: mapCommon.TraceOverlayParams = {
  points: this.points,
  animationDuration: 1000,        // draw the whole track over one second
  isMapMoving: false,             // the camera stays framed; it does not follow the pen
  color: 0xAA36C18D,              // ARGB: AA = ~67% opaque, so the road shows through
  width: 20,
  animationCallback: async (pointIndex) => {
    if (pointIndex === 0) {
      this.startMarkerBuilder();                    // drop the start pin as drawing begins
    }
    if (pointIndex === this.points.length - 1) {
      this.destMarkerBuilder();                     // and the finish pin as it ends
    }
  }
};
await this.mapController.addTraceOverlay(traceOptions);
```

**`isMapMoving: false` is the choice that makes this card's title true.** Set
it to `true` and the camera follows the animation like a replay, which loses
the framing the bounds just established. False keeps the whole route in view
while the line grows across it.

**`animationCallback` is what synchronises the markers to the animation.**
Adding both markers before `addTraceOverlay` would show start and finish
before the line existed; dropping them from the callback at index 0 and
`length - 1` ties them to the drawing. The colour is **ARGB**, so `0xAA36C18D`
is a translucent green - the alpha byte is what lets the road show through a
20-px-wide track.

**Releasing — same file** (corrected, see `HW-03-0035`)

```typescript
private mapLoadCallback = () => { hilog.info(0x0001, this.TAG, `on-mapLoad`); };
private traceOverlay?: map.TraceOverlay;

// in the init callback
this.traceOverlay = await this.mapController.addTraceOverlay(traceOptions);   // FIX: result discarded
this.mapEventManager = this.mapController.getEventManager();
this.mapEventManager.on('mapLoad', this.mapLoadCallback);

aboutToDisappear(): void {                       // FIX: absent in the sample
  this.mapEventManager?.off('mapLoad');
  this.traceOverlay?.remove();
  this.mapController?.clear();
}
```

## Permissions & config

The sample declares **none**. It needs:

```json5
// entry/src/main/module.json5
"metadata": [
  { "name": "client_id", "value": "<your AGC Client ID>" }
],
"requestPermissions": [
  { "name": "ohos.permission.INTERNET",         "usedScene": { "when": "always" } },
  { "name": "ohos.permission.GET_NETWORK_INFO", "usedScene": { "when": "always" } }
]
```

Both are `system_grant`. No location permission is needed - the track is
supplied, not recorded. `SPORT-11` is the sample that needs location.

The document's setup steps are otherwise unusually complete, and one of them
is worth carrying: **automatic signing does not work with Map Kit** -
使用自动签名无法使用地图服务 - the debug certificate must carry the
fingerprint and be signed manually.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **AGC setup plus manual signing is mandatory.** Enable Map Kit, add the
  public-key fingerprint to the debug certificate, and sign manually.
- Map Kit is a **mainland-China service**; the tracks are GCJ02.
- **`getExtremum` does not handle the antimeridian**, as its own comment
  states. A route crossing longitude 180 needs the eastern and western spans
  computed separately.
- The padding passed to `newLatLngBounds` is a fixed 200; on a small screen
  with a long track that leaves little room for the polyline itself.
- Routes are static data keyed by nickname in `DataModel`; there is no
  recording here.

## Pitfalls

- **`HW-03-0032` — no `INTERNET`, no `GET_NETWORK_INFO`, no `client_id`.** The
  map cannot load from a clean checkout, and the document's four setup steps
  never mention `module.json5`.
- **`HW-03-0033` — `getPoints` casts a possibly absent `Map` entry to an
  array,** and the caller casts a possibly empty route parameter to a string
  just above it. An unknown name reaches `for (const point of undefined)` in
  `aboutToAppear`.
- **`HW-03-0034` — an empty track leaves the out-of-range sentinels in
  place,** which average to latitude 0, longitude 0 and produce an inverted
  `LatLngBounds`.
- **`HW-03-0035` — the `mapLoad` listener and the trace overlay are never
  released,** and `addTraceOverlay`'s return is discarded so there is no
  handle to remove the trace with.

## References

- `documentation/harmonyos-guides/07_application-services/map-camera.md` - `newLatLngBounds`, `moveCamera`, `CameraUpdate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-camera
- `documentation/harmonyos-guides/07_application-services/map-dyntrajectories.md` - `addTraceOverlay` and `TraceOverlayParams`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-dyntrajectories
- `documentation/harmonyos-references/06_application-services/map-map-traceoverlay.md` - the overlay handle and `remove()`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-map-traceoverlay
- `documentation/harmonyos-references/03_application-services/map-common.md` - `LatLngBounds`, `MapOptions`, `MarkerOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-common
- `documentation/harmonyos-guides/04_application-services/map-marker.md` - start and finish markers
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-marker
- `documentation/harmonyos-guides/07_application-services/map-faq-1.md` - the two network permissions
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-faq-1
- `documentation/harmonyos-guides/07_application-services/map-faq-4.md` - the `off()` plus `clear()` teardown
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-faq-4
- `documentation/harmonyos-guides/04_system/configuration_client_id.md` - the `client_id` metadata
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/configuration_client_id
- `SPORT-11` - recording the track this card frames
