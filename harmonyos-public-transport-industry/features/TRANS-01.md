---
id: TRANS-01
title: Transit and navigation app - layered architecture with location, routing and ride codes
industry: 06_public_transport
doc: huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
sample: huawei_industry_tree/06_public_transport/downloads/TravelSolutionDemo.zip
kits: ["@kit.MapKit", "@kit.LocationKit", "@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit"]
apis: [geoLocationManager.on locationChange, geoLocationManager.off, geoLocationManager.getCurrentLocation, geoLocationManager.isGeocoderAvailable, geoLocationManager.getAddressesFromLocation, MapComponent, map.MapComponentController, addMarker, addPolyline, site.searchByText, navi.getDrivingRoutes, navi.getCyclingRoutes, navi.getWalkingRoutes, abilityAccessCtrl.AtManager, preferences.getPreferencesSync, Tabs]
permissions: [ohos.permission.INTERNET, ohos.permission.LOCATION, ohos.permission.APPROXIMATELY_LOCATION]
min_api: 24
modules: [product/phone, common, features/account, features/home, features/pay, features/personal, features/travel, rideCode/cityCode, rideCode/otherCityCode]
findings: [HW-06-0001, HW-06-0002, HW-06-0003, HW-06-0004, HW-06-0005, HW-06-0006, HW-06-0007, HW-06-0008, HW-06-0009, HW-06-0010, HW-06-0011, HW-06-0012, HW-06-0013, HW-06-0014, HW-06-0015, HW-06-0016, HW-06-0050, HW-06-0052]
status: verified-with-fixes
---

## When to use

Load this card when building a **public-transport or travel app**: metro and bus
ride codes, route planning, station lookup, travel history, fare top-up. It
defines the module split and the three capabilities that make this industry
distinct - **reverse geocoding to a city name**, **multi-modal route planning**,
and **on-demand loading of per-city modules**.

It is also the reference for any app whose first action on launch is "work out
which city the user is in".

## Feature checklist

- Home: line listing, route planning, city picker, accessible-travel booking,
  scan-to-ride.
- Offers: top-up promotions, commuter passes.
- Ride: login, QR ride-code generation - the core screen of a transit app.
- Mine: login, settings, card wallet, ride history, profile.
- On launch, resolve the device position into a city name and remember it.
- Route planning across driving, cycling and walking, drawn as a polyline.
- Per-city ride-code modules, loadable on demand.

## Architecture

Three layers, nine modules, one IDE project. Note the **product layer is a
directory of its own** (`product/phone`), not an `entry/` folder - this is the
layered-modularisation shape the document links to.

| Layer | Modules | Type |
|---|---|---|
| Product customisation | `product/phone` | entry HAP |
| Basic feature | `features/{home,account,pay,personal,travel}` | HAR |
| Common capability | `common` | HAR |
| Per-city | `rideCode/{cityCode,otherCityCode}` | HAR |

`common` carries what the whole app shares: `BreakpointSystem` (responsive
breakpoints), `CommonDataSource` (a lazy `IDataSource`), `HttpUtil`, `Logger`,
`CommonWeb`, and the shared view models.

**The one architectural idea worth taking from this practice**: for a transit
app that federates other cities' ride codes, package each city's module as an
**HSP** rather than a HAR and load it on demand. Cities are numerous, each
brings its own SDK, and a traveller uses at most one or two - bundling them all
into the HAP inflates the download for everyone. The document points at the
modular-design best practice for the mechanics.

Dependency rule the document states explicitly: upper layers may depend on lower
ones; cross-layer, reverse and sibling dependencies are discouraged.

> **Permissions belong in `product/phone` only.** The sample declares them in
> all nine manifests - see `HW-06-0009`.

## Implementation steps

1. **Configure AGC before anything else.** Map Kit will not render without the
   map service enabled and a Client ID in the entry manifest's `metadata` block
   (`HW-06-0010`).
2. **Declare `INTERNET`, `LOCATION` and `APPROXIMATELY_LOCATION`** in the entry
   HAP, and nowhere else.
3. **Check every permission, not just the first**, then request once
   (`HW-06-0005`, `HW-06-0004`).
4. **Pick one positioning mechanism** - a one-shot fix, or a subscription that
   unsubscribes after the first result. Not both (`HW-06-0002`).
5. **Unsubscribe on every exit path**, including the geocoder-unavailable branch
   (`HW-06-0003`).
6. **Length-check every geocoding and place-search result** before indexing
   (`HW-06-0008`, `HW-06-0007`).
7. **Search places at a realistic radius** - the default is 50000 m
   (`HW-06-0006`).
8. **Clear loading flags in `finally`** (`HW-06-0001`).

## Verified snippets

All snippets are from `TravelSolutionDemo.zip`. Corrected forms are marked.

**Reverse-geocoding a position into a city name — `features/home/.../Home.ets`** (corrected, see `HW-06-0002`, `HW-06-0003`, `HW-06-0008`)

```typescript
import { geoLocationManager } from '@kit.LocationKit';

// FIX: the shipped version also fires getCurrentLocation() alongside this
// subscription, so both race to write this.city.
getLocationInfo() {
  const requestInfo: geoLocationManager.LocationRequest = {
    'priority': geoLocationManager.LocationRequestPriority.FIRST_FIX,
    'timeInterval': 0,
    'distanceInterval': 0,
    'maxAccuracy': 0
  };

  const locationChange = (location: geoLocationManager.Location): void => {
    // FIX: unsubscribe once, on every path - the shipped code only does it
    // inside the isGeocoderAvailable() branch.
    geoLocationManager.off('locationChange', locationChange);
    if (!geoLocationManager.isGeocoderAvailable()) {
      return;
    }
    this.latitude = location.latitude;
    this.longitude = location.longitude;
    const req: geoLocationManager.ReverseGeoCodeRequest = {
      'latitude': location.latitude, 'longitude': location.longitude, 'maxItems': 1
    };
    geoLocationManager.getAddressesFromLocation(req, (err, data) => {
      if (err || !data || data.length === 0) {     // FIX: shipped code indexes data[0] blind
        Logger.error('Home', `reverse geocode failed: ${JSON.stringify(err)}`);
        return;
      }
      this.city = `${data[0].locality}`;
      this.localArea = `${data[0].placeName}`;
      AppStorage.setOrCreate('firstSelectCity', `${data[0].locality}`);
    });
  };

  geoLocationManager.on('locationChange', requestInfo, locationChange);
}
```

`LocationRequestPriority.FIRST_FIX` is the right choice here: the screen wants a
position quickly, not an accurate one, because the answer is used only to name a
city. `isGeocoderAvailable()` is worth keeping as a guard - reverse geocoding is
not available everywhere.

Publishing the resolved city to `AppStorage` under a well-known key is the
pattern the rest of the app reads from.

**Permission gate — `features/home/.../Home.ets`** (corrected, see `HW-06-0004`, `HW-06-0005`)

```typescript
permissions: Array<Permissions> = ['ohos.permission.LOCATION', 'ohos.permission.APPROXIMATELY_LOCATION'];

async checkPermissions(): Promise<void> {
  // FIX: shipped code checks only this.permissions[0]
  for (const p of this.permissions) {
    if (await this.checkAccessToken(p) !== abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED) {
      this.reqPermissionsFromUser(this.permissions,
        this.getUIContext().getHostContext() as common.UIAbilityContext);
      return;
    }
  }
  this.getLocationInfo();
}

reqPermissionsFromUser(permissions: Array<Permissions>, context: common.UIAbilityContext): void {
  const atManager = abilityAccessCtrl.createAtManager();
  atManager.requestPermissionsFromUser(context, permissions).then((data) => {
    // FIX: shipped code calls getLocationInfo() inside the loop, once per grant
    if (data.authResults.some((r) => r !== 0)) {
      return;
    }
    this.getLocationInfo();
  }).catch((err: BusinessError) => {
    Logger.error('Home', `request permission error: ${err.code} ${err.message}`);
  });
}
```

**Place search — `features/travel/.../RouterPlan.ets`** (corrected, see `HW-06-0006`, `HW-06-0007`)

```typescript
import { site, mapCommon } from '@kit.MapKit';

.onBlur(async () => {
  try {
    const rsp = await site.searchByText({
      query: this.start,
      // FIX: shipped code sets radius: 50 (metres). Documented default is 50000.
      pageIndex: 1,
      pageSize: 5
    });
    if (!rsp || !rsp.sites || rsp.sites.length === 0) {   // FIX: shipped guard omits length
      // tell the user nothing matched; do not fall back to 0,0
      return;
    }
    const position: mapCommon.LatLng = {
      latitude: rsp.sites[0].location.latitude,
      longitude: rsp.sites[0].location.longitude
    };
    if (this.startMarker) {
      this.startMarker.setPosition(position);
    } else {
      this.markerOptions.position = position;
      this.startMarker = await this.mapController?.addMarker(this.markerOptions);
    }
  } catch (error) {
    Logger.error('RouterPlan', `searchByText failed: ${JSON.stringify(error)}`);
  }
})

```

`searchByText` params worth knowing: `radius` in metres (1–50000, default
50000), `cityId` to scope to one city (3–4 digit city code or 6-digit division
code), `pageIndex` / `pageSize` for paging, `poiTypes` to filter by place type,
`language` for result language.

**Multi-modal route planning — `features/travel/.../RouterPlan.ets`** (corrected, see `HW-06-0001`, `HW-06-0016`)

```typescript
import { navi } from '@kit.MapKit';

async getRoutes() {
  if (!this.startMarker || !this.endMarker) {
    return;
  }
  try {
    this.routes = [];
    this.points = [];
    const startLatLng = this.startMarker.getPosition();
    const endLatLng = this.endMarker.getPosition();
    if (!startLatLng || !endLatLng) {      // FIX: shipped code substitutes a Beijing coordinate
      return;
    }

    let result: navi.RouteResult;
    if (this.activeRouterClassIndex === 0) {
      this.drivingRouteParams.origins = [startLatLng];
      this.drivingRouteParams.destination = endLatLng;
      result = await navi.getDrivingRoutes(this.drivingRouteParams);
    } else {
      this.routeParams.origins = [startLatLng];
      this.routeParams.destination = endLatLng;
      result = this.activeRouterClassIndex === 1
        ? await navi.getCyclingRoutes(this.routeParams)
        : await navi.getWalkingRoutes(this.routeParams);
    }
    if (!result.routes || result.routes.length === 0) {   // FIX: shipped code indexes [0] blind
      return;
    }

    result.routes[0].steps.forEach((step) => {
      this.routes.push({
        distance: step.distance,
        distanceDescription: step.distanceDescription,
        duration: step.duration,
        durationDescription: step.durationDescription,
        durationInTraffic: step.durationInTraffic,
        durationInTrafficDescription: step.durationInTrafficDescription,
      });
      step.roads.forEach((road) => {
        road.polyline.forEach((polyline) => { this.points.push(polyline); });
      });
    });

    // overviewPolyline when the service supplies one, otherwise the per-road points
    const POINTS = result.routes[0].overviewPolyline ?? this.points;
    // thin the polyline: every other point, always keeping the last
    this.polylineOption.points = POINTS.filter((item, index) =>
      index % 2 === 0 || index === POINTS.length - 1);

    if (this.mapPolyline) {
      this.mapPolyline.setPoints(this.polylineOption.points);
    } else {
      this.mapPolyline = await this.mapController?.addPolyline(this.polylineOption);
    }
  } catch (err) {
    Logger.error('RouterPlan', `getRoutes failed: ${JSON.stringify(err)}`);   // FIX: shipped catch is empty
  } finally {
    this.getRoutesLoading = false;    // FIX: shipped code clears this only on success
  }
}
```

Three things worth lifting: one `RouteResult` shape covers driving, cycling and
walking so the rendering code is shared; `overviewPolyline` is the cheap
route-shape when the service returns it, with the per-road points as fallback;
and thinning the polyline before handing it to `addPolyline` keeps the draw cheap
on a long route.

`RouteParams.avoids` takes a numeric mask (the sample uses `[0, 8]`), and
`language` should follow the device locale rather than being pinned.

**Permissions in the entry HAP — `product/phone/src/main/module.json5`** (corrected, see `HW-06-0009`, `HW-06-0010`)

```json5
"metadata": [
  { "name": "client_id", "value": "YOUR_CLIENT_ID" },   // FIX: shipped file has a real ID
  { "name": "app_id", "value": "YOUR_APP_ID" }
],
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" },
  { "name": "ohos.permission.APPROXIMATELY_LOCATION",
    "reason": "$string:permission_location_approximately_reason",
    "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.LOCATION",
    "reason": "$string:permission_location_reason",
    "usedScene": { "abilities": ["EntryAbility"] } }
]
```

The `client_id` metadata is how Map Kit authorises the app against the AGC
project. Miss it and the map is simply blank, with nothing in the code to
suggest why.

## Permissions & config

| Permission | Why | Grant type |
|---|---|---|
| `ohos.permission.INTERNET` | Map tiles, place search, routing | system_grant |
| `ohos.permission.APPROXIMATELY_LOCATION` | Coarse position, ~5 km | user_grant |
| `ohos.permission.LOCATION` | Precise position, metre-level | user_grant |

The document also names `ohos.permission.LOCATION_IN_BACKGROUND` for apps that
must keep positioning while backgrounded - a real need for a transit app tracking
a journey - but this sample neither declares nor uses it.

`build-profile.json5`: `compatibleSdkVersion` and `targetSdkVersion` both
`6.1.1(24)`.

## Constraints

- **API Version 24 Release and DevEco Studio 6.1.1 Release** - stated under
  `环境准备` rather than the usual `约束与限制`.
- AGC: map service enabled, plus the Client ID in the entry manifest.
- The login screen performs no authentication: any password is accepted once the
  phone number is long enough. The document says so; implement real auth.
- Route and place data come from Map Kit; the home feed, notices and local-life
  list are mock data in `features/home/.../MockData.ets`.
- The document says phones only; the manifests say otherwise (`HW-06-0011`).

## Pitfalls

- **`HW-06-0001` — one failed route lookup kills the search button.** The
  loading flag is cleared only on the success path and the catch is empty, so
  after any failure every tap is silently ignored for the rest of the session.
- **`HW-06-0002` — two positioning mechanisms race.** A `locationChange`
  subscription and a `getCurrentLocation()` call both reverse-geocode and both
  write `this.city`; only one of them also writes `localArea` and the
  `AppStorage` key.
- **`HW-06-0003` — the subscription leaks when the geocoder is unavailable.**
  Every `off()` sits inside the `isGeocoderAvailable()` branch.
- **`HW-06-0004` — `getLocationInfo()` is called inside the grant loop,** so
  accepting the two-permission dialog starts two full positioning cycles.
- **`HW-06-0005` — `checkPermissions` inspects only `permissions[0]`.** Third
  variant of this defect family in the corpus; the automotive samples get it
  wrong differently.
- **`HW-06-0006` — `radius: 50` metres** on both place searches, against a
  documented default of 50000.
- **`HW-06-0007` — an empty search result becomes coordinate 0,0.** The guard
  checks that `sites` exists, not that it has anything in it, and `|| 0` does
  the rest.
- **`HW-06-0008` — geocoding results are indexed blind** and the one-shot path
  has no `.catch()`; the surrounding `try` catches only synchronous throws.
- **`HW-06-0009` — all eight HARs redeclare the permissions** and name an
  `EntryAbility` none of them contains.
- **`HW-06-0010` — real AGC `client_id` and `app_id` ship in the manifest,** and
  the document never tells you to replace them.
- **`HW-06-0011` — three different answers to "which devices".** Document says
  phone; entry says phone+tablet; HARs say default+tablet.
- **`HW-06-0012` — the `calendarManager` link goes to gitcode.com/openharmony,**
  a different product's docs, while the HarmonyOS reference page exists.
- **`HW-06-0013` — five empty catch blocks** across route planning, place search
  and preferences, in a project that imports its own `Logger` next door.
- **`HW-06-0014` — the destination search is hardcoded to city `025`** while the
  origin search has no city filter, so you can start anywhere and only arrive in
  Nanjing.
- **`HW-06-0015` — the location request is commented as a microphone request.**
- **`HW-06-0016` — four unrelated hardcoded coordinates,** including a default
  destination in the Irish Sea and reachable Beijing fallbacks.

## References

- `documentation/harmonyos-references/03_application-services/map-site.md` - `searchByText`, `SearchByTextParams.radius` and `cityId`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-site
- `documentation/harmonyos-guides/07_application-services/map-site-search.md` - place search guide
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-site-search
- `documentation/harmonyos-references/06_application-services/js-apis-geolocationmanager.md` - `LocationRequest`, `on`/`off('locationChange')`, `getAddressesFromLocation`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-geolocationmanager
- `documentation/harmonyos-guides/04_application-services/map-config-agc.md` - AGC prerequisites for Map Kit
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-config-agc
- `documentation/harmonyos-references/06_application-services/js-apis-calendarmanager.md` - the page `HW-06-0012`'s link should point at
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-calendarmanager
- `documentation/harmonyos-guides/04_system/request-app-permissions.md` - `authResults`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-app-permissions
- Document's own pointers: `practice-common-app-layered-v1` (layered modularisation), `bpta-modular-design` (on-demand HSP loading), `liveview-design-formula` (Live View Kit), `form-kit` (service cards)
