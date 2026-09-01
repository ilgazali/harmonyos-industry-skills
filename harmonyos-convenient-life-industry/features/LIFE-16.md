---
id: LIFE-16
title: Route planning between two searched addresses - geocode with Site Kit, plan with navi, draw as a MapPolyline
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/16_direction_demo.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/direction_demo-0000002317152262
sample: huawei_industry_tree/02_convenient_life/downloads/MapDirectionDemo.zip
kits: ["@kit.MapKit", "@kit.AbilityKit", "@kit.BasicServicesKit"]
apis: [MapComponent, "map.MapComponentController", "controller.addMarker", "controller.addPolyline", "controller.on('mapLoad')", "map.Marker", "marker.getPosition", "map.MapPolyline", "mapCommon.MarkerOptions", "mapCommon.MapPolylineOptions", "mapCommon.CapStyle", "mapCommon.JointType", "mapCommon.LatLng", "mapCommon.MapOptions", "site.searchByText", "navi.getDrivingRoutes", "navi.getCyclingRoutes", "navi.getWalkingRoutes", "navi.DrivingRouteParams", "navi.RouteParams", "navi.RouteResult", "@Provide", "@State", "@StorageProp", List, ListItem, constraintSize, Scroller]
permissions: ["ohos.permission.INTERNET", "ohos.permission.APPROXIMATELY_LOCATION", "ohos.permission.LOCATION"]
min_api: 20
modules: [entry]
findings: [HW-02-0113, HW-02-0114, HW-02-0115, HW-02-0116, HW-02-0117, HW-02-0118, HW-02-0119, HW-02-0120]
status: verified-with-fixes
---

## When to use

Load this card when you need the **address-in, route-out** pipeline: the user
types two place names, and the map shows the route between them with per-leg
distance and duration, switchable between driving, cycling and walking.

Three Map Kit surfaces are chained, and the joins are the interesting part:

```
site.searchByText(text)     -> Site[]      geocoding: a place name becomes a LatLng
controller.addMarker(opts)  -> Marker      the pin, which also stores the position
marker.getPosition()        -> LatLng      read the endpoints back off the pins
navi.getXxxRoutes(params)   -> RouteResult the polyline and the step list
controller.addPolyline(...) -> MapPolyline the drawn route
```

Using the markers as the store for the two endpoints is the neat idea here - the
pin and the coordinate cannot drift apart because there is only one of them.
Everything else in the sample needs correcting before reuse: it never requests
its permissions (`HW-02-0113`), swallows every error (`HW-02-0114`), and leaks an
overlay per request (`HW-02-0115`).

Take this for "directions between two places", delivery-route previews, or any
screen that has to turn free text into a drawn route. For turn-by-turn
navigation, Map Kit has a dedicated navigation surface instead.

## Feature checklist

- Two text fields, start and end, seeded with example addresses.
- Three travel-mode buttons - car, walking, bicycle - each planning immediately
  when tapped.
- Each address is geocoded by text search and marked with a pin.
- The route between the two pins is drawn as a coloured polyline.
- A list under the map shows one row per leg: index, distance description and
  duration-in-traffic description; tapping a row selects it.
- The list is capped in height and pinned to the bottom of the map.

## Architecture

One `entry` module, one page, one constants file.

```
entry/src/main/ets
├── constants/StyleConstants.ets   sizes, the two seed addresses
├── entryability/EntryAbility.ets  full screen, avoid areas -> AppStorage
└── pages/DirectionDemo.ets        THE CARD: everything else, 402 lines
```

The documented tree matches the zip exactly.

**Options objects are declared once as fields and mutated per call.** That is
the sample's structural decision, and it is what causes two of its defects:

| field | purpose | mutated by |
| --- | --- | --- |
| `markerOptions` | shared pin config | `getStartPosition`, `getEndPosition` set `.position` |
| `polylineOption` | shared line config | `getRoutes` sets `.points` |
| `drivingRouteParams` | driving request | `getRoutes` sets `.origins`/`.destination` |
| `routeParams` | cycling and walking | `getRoutes` sets `.origins`/`.destination` |

Reusing one `markerOptions` for both pins is fine because `addMarker` takes a
snapshot; reusing the returned handles without releasing the previous ones is
not (`HW-02-0115`).

**The request flow, per travel-mode tap:**

```
mode button onClick
  -> activeRouterClassIndex = index
  -> getRoutes()
       await getStartPosition()   searchByText -> addMarker -> startMarker
       await getEndPosition()     searchByText -> addMarker -> endMarker
       guard: both markers exist
       startMarker.getPosition() / endMarker.getPosition()
       index 0 -> navi.getDrivingRoutes(drivingRouteParams)
       index 1 -> navi.getCyclingRoutes(routeParams)
       else    -> navi.getWalkingRoutes(routeParams)      <- -1 lands here (HW-02-0120)
       collect steps -> this.routes;  collect road polylines -> this.points
       prefer overviewPolyline, else the collected points
       decimate to every other point
       addPolyline
```

**Two parameter types, not one.** `navi.DrivingRouteParams` and
`navi.RouteParams` are distinct - driving carries `waypoints`, the shared
`routeParams` carries `avoids` and `extension` - which is why the branch splits
before the call rather than picking a function at the end.

## Implementation steps

1. **Request the location permissions at runtime** before anything touches Map
   Kit (`HW-02-0113`). Declaring them in `module.json5` grants nothing.
2. **Geocode each endpoint with `site.searchByText`,** and treat an empty
   `sites` array as a failure rather than defaulting to `0, 0` (`HW-02-0117`).
3. **Store the endpoint on the marker** and read it back with `getPosition()`.
   One object, no drift.
4. **Release the previous marker and polyline before adding new ones**
   (`HW-02-0115`), and do not create a placeholder pin in `mapLoad`.
5. **Point `MarkerOptions.icon` at `resources/rawfile`** or use `$r()`
   (`HW-02-0116`).
6. **Branch on the travel mode before the call,** because driving takes a
   different params type. Guard the unselected state (`HW-02-0120`).
7. **Check `result.routes.length` before indexing** (`HW-02-0117`).
8. **Prefer `overviewPolyline` and fall back to the per-road points,** then
   decimate before drawing - a full route can be thousands of points.
9. **Log every failure.** Three empty catches is what makes all of the above
   invisible (`HW-02-0114`).

## Verified snippets

All snippets are from `MapDirectionDemo.zip`. Corrected forms are marked.

**Geocoding an address - `MapDirectionDemo.zip#entry/src/main/ets/pages/DirectionDemo.ets:172`** (corrected, see `HW-02-0114`, `HW-02-0115` and `HW-02-0117`)

```typescript
async getStartPosition() {
  try {
    const RESULT = await site.searchByText({
      query: this.start,
      radius: 50,           // km around the implicit location bias
      pageIndex: 1,
      pageSize: 5
    });
    // FIX: the sample tests only `RESULT && RESULT.sites` and then reads
    //      RESULT.sites[0]?.location?.latitude || 0 - an empty result becomes 0,0
    if (!RESULT || !RESULT.sites || RESULT.sites.length === 0 || !RESULT.sites[0].location) {
      return;
    }
    const position: mapCommon.LatLng = {
      latitude: RESULT.sites[0].location.latitude,
      longitude: RESULT.sites[0].location.longitude
    };
    this.markerOptions.position = position;
    this.startMarker?.remove();                    // FIX: sample leaks the previous pin
    this.startMarker = await this.mapController?.addMarker(this.markerOptions);
  } catch (error) {
    // FIX: the sample's catch is empty, here and in getEndPosition and getRoutes
    hilog.error(0x0000, TAG, 'searchByText failed: code %{public}d, message %{public}s',
      (error as BusinessError).code, (error as BusinessError).message);
  }
}
```

**`site.searchByText` is geocoding, not a place picker.** It returns ranked
candidates; taking `sites[0]` is "best match", which is right for a demo but
means an ambiguous name silently resolves to whichever candidate ranked first.
`pageSize: 5` fetches five and uses one.

**`|| 0` on a coordinate is not a default, it is a bug.** Longitude 0, latitude 0
is a real point in the Gulf of Guinea, so a failed lookup produces a pin and a
route request rather than an error - which is how the empty catch two lines
below ends up hiding a user-visible failure.

**Planning the route - same file, line 125** (corrected, see `HW-02-0117`, `HW-02-0118` and `HW-02-0120`)

```typescript
async getRoutes() {
  if (this.activeRouterClassIndex < 0) {           // FIX: -1 otherwise falls through to walking
    return;
  }
  await this.getStartPosition();
  await this.getEndPosition();
  if (!this.startMarker || !this.endMarker) {
    return;
  }
  try {
    this.routes = [];
    this.points = [];
    let result: navi.RouteResult;
    const START_LAT_LNG = this.startMarker.getPosition();
    const END_LAT_LNG = this.endMarker.getPosition();

    if (this.activeRouterClassIndex === 0) {
      this.drivingRouteParams.origins = START_LAT_LNG ? [START_LAT_LNG] : [];
      this.drivingRouteParams.destination = END_LAT_LNG ? END_LAT_LNG : { latitude: 39.9, longitude: 116.4 };
      result = await navi.getDrivingRoutes(this.drivingRouteParams);
    } else {
      this.routeParams.origins = START_LAT_LNG ? [START_LAT_LNG] : [];
      this.routeParams.destination = END_LAT_LNG ? END_LAT_LNG : { latitude: 39.9, longitude: 116.4 };
      result = this.activeRouterClassIndex === 1 ? await navi.getCyclingRoutes(this.routeParams) :
        await navi.getWalkingRoutes(this.routeParams);
    }

    if (!result.routes || result.routes.length === 0) {   // FIX: sample indexes routes[0] blindly
      return;
    }
    const ROUTE = result.routes[0];

    ROUTE.steps.forEach((step) => {                       // FIX: sample declares this callback async
      this.routes.push({
        distance: step.distance,
        distanceDescription: step.distanceDescription,
        duration: step.duration,
        durationDescription: step.durationDescription,
        durationInTraffic: step.durationInTraffic,
        durationInTrafficDescription: step.durationInTrafficDescription,
      });
      step.roads.forEach((road) => {
        road.polyline.forEach((polyline) => {
          this.points.push(polyline);
        });
      });
    });

    const POINTS = ROUTE.overviewPolyline == null ? this.points : ROUTE.overviewPolyline;
    this.polylineOption.points = POINTS.filter((item, index) =>
      index % 2 === 0 || index === POINTS.length - 1);   // decimate: every other point, keep the last

    this.mapPolyline?.remove();                          // FIX: sample leaks the previous line
    this.mapPolyline = await this.mapController?.addPolyline(this.polylineOption);
  } catch (err) {
    hilog.error(0x0000, TAG, 'route planning failed: %{public}s', JSON.stringify(err));   // FIX
  } finally {
    this.getRoutesLoading = false;
  }
}
```

**`overviewPolyline` first, per-road points as the fallback.** The overview is
the service's own simplified geometry for the whole route; walking that route's
`steps -> roads -> polyline` gives the full-resolution path. Preferring the
overview is right, and the collected points are worth building anyway because
the fallback path is reachable.

**The decimation is not optional at scale.** A cross-city driving route can
carry thousands of `LatLng` points; keeping every second one halves the vertex
count with no visible change at map zoom, and `index === POINTS.length - 1`
keeps the endpoint so the line does not stop short.

**The three-way branch splits before the call because the parameter types
differ.** `navi.DrivingRouteParams` carries `waypoints`; `navi.RouteParams`
carries `avoids: [0, 8]` and `extension: 0`. A single `params` object cannot
serve both.

**The overlay options - same file, line 45 and line 58** (as shipped)

```typescript
polylineOption: mapCommon.MapPolylineOptions = {
  points: [],
  clickable: true,
  startCap: mapCommon.CapStyle.BUTT,
  endCap: mapCommon.CapStyle.BUTT,
  geodesic: false,            // straight segments, not great-circle arcs
  jointType: mapCommon.JointType.BEVEL,
  visible: true,
  color: 0xff00ffff,          // ARGB: opaque cyan
  width: 15,
  zIndex: 10,                 // above the markers, whose zIndex is 0
  gradient: false
};

markerOptions: mapCommon.MarkerOptions = {
  position: { latitude: 2.922865, longitude: 101.58584 },   // overwritten before every use
  rotation: 0,
  visible: true,
  zIndex: 0,
  alpha: 1,
  anchorU: 0.5,
  anchorV: 1,                 // anchor at the bottom centre - the pin's tip
  clickable: true,
  draggable: false,
  flat: false,
  icon: 'icon.png'            // must live under resources/rawfile - HW-02-0116
};
```

**`geodesic: false` is the right default for a road route.** Geodesic segments
bend along the great circle, which is correct for flight paths and wrong for a
polyline that is already following roads.

**`anchorV: 1` with `anchorU: 0.5`** anchors the icon at its bottom centre, so
the pin's tip sits on the coordinate rather than its middle - the difference
between a marker that points at a place and one that hovers over it.

**`zIndex: 10` on the line against `0` on the markers** puts the route under the
pins, so the endpoints stay visible where the line meets them.

**Map setup - same file, line 103** (corrected, see `HW-02-0115`)

```typescript
aboutToAppear(): void {
  this.mapOption = {
    position: { target: { latitude: 39.9, longitude: 116.4 }, zoom: 10 }
  };
  this.callback = async (err, mapController) => {
    if (!err) {
      this.mapController = mapController;
      // FIX: the sample adds a marker here, at markerOptions' default coordinate -
      //      a pin with no relation to anything the user searched for, never removed.
    }
  };
}
```

The shipped `mapLoad` handler creates `this.startMarker` from the unmodified
`markerOptions`, whose default position is `2.922865, 101.58584`. That pin is
overwritten as a *handle* by `getStartPosition` but never removed from the map.

**The route list - same file, line 212** (as shipped)

```typescript
List({ scroller: this.scroller }) {
  ForEach(this.routes, (item: RouteObj, index: number) => {
    ListItem() {
      this.RouterListItem(item, index);
    };
  });
}
.divider({ strokeWidth: StyleConstants.WIDTH_1, color: $r('app.color.light_gray') })
.constraintSize({ maxHeight: StyleConstants.MAX_HEIGHT_200 })
.position({ bottom: StyleConstants.BOTTOM_0 })
```

**`constraintSize({ maxHeight })` plus `position({ bottom })` is how a list
overlays a map without covering it.** The list takes only the height its rows
need, up to 200 vp, and grows upward from the bottom edge - so a two-leg route
shows a short panel and a twenty-leg route shows a scrollable one, without either
being laid out against the map's height.

The `ForEach` has no key generator; `RouteObj` carries no id, so adding one means
adding a key to the model.

## Permissions & config

`MapDirectionDemo.zip#entry/src/main/module.json5`

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    // FIX: obsolete since HarmonyOS 5.0.2(14), and this is the sample author's own - HW-02-0119
    // "metadata": [{ "name": "client_id", "value": "6917580808115985768" }],
    "requestPermissions": [
      {
        "name": "ohos.permission.INTERNET",
        "reason": "$string:EntryAbility_desc",
        "usedScene": { /* ... */ }
      },
      {
        "name": "ohos.permission.APPROXIMATELY_LOCATION",
        "reason": "$string:permission_location_approximately_reason",
        "usedScene": { /* ... */ }
      },
      {
        "name": "ohos.permission.LOCATION",
        "reason": "$string:permission_location_reason",
        "usedScene": { /* ... */ }
      }
    ]
  }
}
```

The declaration block is correct - both user-grant location permissions carry a
`reason` and a `usedScene`, and `APPROXIMATELY_LOCATION` is present as
`LOCATION`'s documented prerequisite. What is missing is the runtime half: no
`requestPermissionsFromUser` call exists anywhere in the project
(`HW-02-0113`).

Map Kit additionally requires enabling the service in AppGallery Connect and a
manual signing configuration - the document's 说明 block at lines 59-64.

`build-profile.json5` sets `compatibleSdkVersion: "6.0.0(20)"` and no
`targetSdkVersion`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 68-70).
- **Map Kit must be enabled in AppGallery Connect and the build manually
  signed.** The zip cannot run as-is without an AGC project.
- Map Kit, Site Kit and `navi` are Chinese-mainland services; availability is
  regional.
- Only the first route is drawn and only the first geocoding candidate is used.
- The travel-mode buttons plan immediately on tap; there is no explicit search
  action, so every mode switch is a fresh pair of geocoding calls plus a routing
  call.
- `navi.RouteParams.avoids: [0, 8]` and `extension: 0` are magic numbers with no
  named constants in the sample.
- The default `routeParams.destination` is `54.216608, -4.66529` (the Isle of
  Man) and `drivingRouteParams`' origin and destination are the same point -
  leftovers that are overwritten before every real call.
- `RouteObj` has no identifier, so the route list cannot be keyed.

## Pitfalls

- **`HW-02-0113` - the location permissions are declared but never requested.**
  Both are user-grant; the `permissions` array on the page is declared and never
  read, and there is no `requestPermissionsFromUser` anywhere in the zip.
- **`HW-02-0114` - all three catches are empty,** around every `site` and `navi`
  call. Combined with the missing permission request, every failure this sample
  can have is invisible.
- **`HW-02-0115` - overlays accumulate.** `addMarker` runs three times and
  `addPolyline` once per route request with nothing removed, and the `mapLoad`
  handler drops an extra pin at the hard-coded default coordinate that is never
  cleaned up.
- **`HW-02-0116` - `MarkerOptions.icon: 'icon.png'` cannot resolve.** The string
  form is looked up under `resources/rawfile`; the project has no such directory
  and the file sits in `resources/base/media`.
- **`HW-02-0117` - `result.routes[0]` is indexed with no length check,** and the
  geocoder's `|| 0` fallback makes an empty result reachable from ordinary input.
- **`HW-02-0118` - `forEach(async (step) => ...)` discards its promises.** It
  works only because the body happens to contain no `await`.
- **`HW-02-0119` - `module.json5` ships a `client_id` metadata entry** the Map
  Kit guide says is unnecessary from HarmonyOS 5.0.2(14), holding the sample
  author's AGC project identifier.
- **`HW-02-0120` - `activeRouterClassIndex` starts at -1** and falls through both
  tests to the walking branch, so an unselected mode silently plans a walk.
- **Do not default a failed geocode to `0, 0`.** It is a real coordinate, so the
  failure becomes a route request instead of an error.
- **Do not draw the raw route geometry.** Prefer `overviewPolyline` and decimate;
  a full driving route is thousands of vertices.
- **Do not reuse one `markerOptions` and forget that the handles are separate.**
  Snapshotting the options is fine; leaking the returned overlays is not.

## References

- `documentation/harmonyos-references/03_application-services/map-navi-api.md` - `navi.getDrivingRoutes`, `getCyclingRoutes`, `getWalkingRoutes`, `DrivingRouteParams`, `RouteParams`, `RouteResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-navi-api
- `documentation/harmonyos-references/03_application-services/map-site.md` - `site.searchByText` and its result shape
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-site
- `documentation/harmonyos-references/03_application-services/map-common.md` - `MarkerOptions` (including the `resources/rawfile` rule for a string `icon`), `MapPolylineOptions`, `CapStyle`, `JointType`, `LatLng`, `MapOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-common
- `documentation/harmonyos-references/03_application-services/map-map-marker.md` - `Marker`, `getPosition`, and its inheritance from `BaseOverlay`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-map-marker
- `documentation/harmonyos-guides/04_application-services/map-config-agc.md` - enabling Map Kit; client ID unnecessary from HarmonyOS 5.0.2(14)
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-config-agc
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `ohos.permission.LOCATION` and its `APPROXIMATELY_LOCATION` prerequisite, and the runtime request requirement
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-size.md` - `constraintSize`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-size
- `LIFE-14` - the same industry's other Map Kit sample, which does implement the permission flow (with its own defects) and repeats the `client_id` and icon-path mistakes
- `LIFE-20`, `LIFE-26`, `LIFE-29` - the industry's remaining Map Kit scenarios
