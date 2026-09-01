---
id: TOUR-06
title: Destination map with nearby POIs - MapComponent plus site.searchByText
industry: 09_tourism
doc: huawei_industry_tree/09_tourism/docs/06_obtain_destination_map.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/obtain_destination_map-0000002318736725
sample: huawei_industry_tree/09_tourism/downloads/DestinationMap.zip
kits: ["@kit.MapKit", "@kit.BasicServicesKit", "@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: [MapComponent, "map.MapComponentController", "map.MapEventManager", "site.searchByText", "site.SearchByTextParams", addMarker, "map.newCameraPosition", animateCamera, "mapCommon.CameraPosition", setZoomControlsEnabled, setMyLocationEnabled, "BaseOverlay.remove", NavPathStack, pushPathByName, NavDestinationContext, ListScroller, scrollToIndex, PanGesture, "@Observed", "@Track", "@Prop"]
permissions: ["ohos.permission.INTERNET", "ohos.permission.GET_NETWORK_INFO"]
min_api: 20
modules: [entry]
findings: [HW-09-0034, HW-09-0035, HW-09-0036, HW-09-0037, HW-09-0038, HW-09-0039, HW-09-0040, HW-09-0080]
status: verified-with-fixes
---

## When to use

Load this card when a **listing has an address and the user wants to know
what is around it** - a hotel, a rental, a venue, a restaurant. The pattern is
a preview card on the detail page ("2 min from Xinjiekou station") that opens
a full map with the destination pinned, the nearby facilities pinned around
it, and a draggable sheet listing them by distance, where tapping a pin
scrolls the list and tapping a row flies the camera.

The transferable pieces are **`site.searchByText` for a radius POI query** -
the part most people do not know Map Kit provides - and the **two-way
selection binding** between a marker layer and a list.

**Check `HW-09-0034` first.** The sample ships with no permissions and no
Client ID, so out of the box neither the map nor the search works.

## Feature checklist

- A rental detail page: photo, price, room facts, tag chips, description.
- An address preview card with a blurred static map image and the closest
  subway station with its distance.
- Tapping the card pushes a full map page.
- The map centres on the property, with a distinct pin for it.
- Subway stations within 2 km are queried, filtered, sorted by distance and
  pinned.
- Stations still under construction (name ending 规划中) are excluded.
- A bottom sheet lists the stations; drag up and down to expand and fold it.
- Tapping a pin selects and scrolls to its row and flies the camera to it.
- Tapping a row flies the camera to its pin.
- Tapping the page title recentres the map on the property.

## Architecture

One `entry` module. The interesting choice is that the map state lives in an
`@Observed` singleton view model rather than in the page.

```
entry/src/main/ets
├── components/SearchNearby.ets     the draggable bottom sheet + station list
├── constants/CommonConstants.ets   every layout number and the 2000 m radius
├── entryability/EntryAbility.ets   full screen, avoid areas -> AppStorage as vp
├── model
│   ├── HouseInfoModel.ets          static listing content
│   └── MetroStationModel.ets       MetroInfo { siteId, name, address, location, distance }
├── pages
│   ├── RentingPage.ets             @Entry, the detail page, runs the POI search
│   └── MapPage.ets                 the NavDestination: map + title bar + sheet
└── viewmodel/MarkerViewModel.ets   map controller, markers, camera - a singleton
```

The documented tree matches the zip.

**The data flows one way, the selection flows both ways.**

```
RentingPage.aboutToAppear
   site.searchByText(radius 2000)  ->  filter, sort by distance
        |                                    |
        v                                    v
   markerVM.metros                    pathStack.pushPathByName('mapPage', metros)
        |                                    |
        v                                    v
   MarkerViewModel.setMarkers()        MapPage.onReady -> @State metros -> SearchNearby
   (on 'mapLoad')                            |
        |                                    |
   searchMarkers[]  <-- index --------> list rows
```

The two sides are joined by **array index**: `searchMarkers[i]` is the pin for
`metros[i]`, because both are built from the same sorted array. A row tap
looks up `searchMarkers[index]`; a pin tap does `searchMarkers.indexOf(marker)`.
That is why the sort has to happen before either array is built, and why
`setMarkers` must not be allowed to leave stale pins in the array
(`HW-09-0038`).

`MarkerViewModel` is `@Observed` with `@Track` on `center`, `metros` and
`searchMarkers`, and is reached through a lazily created static `instance`.
The singleton is what lets `RentingPage` fill the data and `MapPage` and
`SearchNearby` all reach the same controller - and it is also why the never
released listeners in `HW-09-0037` outlive the page.

## Implementation steps

1. **Declare `INTERNET` and `GET_NETWORK_INFO`** and add the `client_id`
   metadata from AGC (`HW-09-0034`, `HW-09-0035`). Nothing below works
   otherwise.
2. **Query the POIs before you need the map** - in the detail page's
   `aboutToAppear`, so the preview card can show the closest one immediately.
   Guard the call (`HW-09-0036`).
3. **Filter twice**: by `distance` against your radius, and by name for
   entries that are not usable yet.
4. **Sort by distance ascending** - this ordering is the contract the marker
   and list arrays share.
5. **Pass the result as the route parameter** and read it in
   `NavDestination.onReady` from `ctx.pathInfo.param`.
6. **Initialise the map through `MapComponent`** with the view model's
   `mapOptions` and `mapCallback`; handle the error branch (`HW-09-0039`).
7. **Register the listeners through `getEventManager()`** and release them on
   teardown (`HW-09-0037`).
8. **Add the destination marker at a higher `zIndex`** than the POI markers so
   it is never hidden.
9. **Move the camera with `animateCamera`** on a `CameraUpdate` built by
   `map.newCameraPosition`, with an explicit duration.
10. **Bind list and markers by index** in both directions.

## Verified snippets

All snippets are from `DestinationMap.zip`. Corrected forms are marked.

**The radius POI query — `entry/src/main/ets/pages/RentingPage.ets`** (corrected, see `HW-09-0036`)

```typescript
import { mapCommon, site } from '@kit.MapKit';

center: mapCommon.LatLng = { latitude: 32.02065982629459, longitude: 118.788899213002 };

async aboutToAppear() {
  this.markerVM.center = this.center;
  this.detail = HOUSE;

  const params: site.SearchByTextParams = {
    query: '地铁站',            // free-text term
    location: this.center,      // centre of the search
    poiTypes: ['SUBWAY'],       // narrow to one POI category
    isChildren: true,           // include child sites (individual entrances)
    language: 'zh_CN',
    radius: 2000                // metres
  };

  try {                                             // FIX: the sample has no try/catch
    const result = await site.searchByText(params);
    if (result.sites) {
      this.metros.length = 0;
      for (const s of result.sites) {
        // radius is a hint, not a guarantee - filter again, and drop stations not open yet
        if (s.distance! <= CommonConstants.DISTANCE && !s.name!.endsWith('(规划中)')) {
          this.metros.push({
            siteId: s.siteId, name: s.name, address: s.formatAddress,
            location: s.location, distance: s.distance
          } as MetroInfo);
        }
      }
      this.metros.sort((a, b) => a.distance - b.distance);
      this.markerVM.metros = this.metros;
    }
  } catch (err) {
    hilog.error(DOMAIN, 'RentingPage', 'searchByText failed: %{public}s', JSON.stringify(err));
  }
  this.closestOne = this.metros[0];
}
```

**`site.searchByText` is the API worth knowing.** `poiTypes` plus `radius`
turns a free-text search into a structured "what is within N metres" query,
and the results carry `distance` already computed from `location`, so no
haversine is needed. Two details in the filtering are worth copying: the
`distance` re-check, because `radius` is a search hint rather than a hard
bound, and the `规划中` (under construction) name filter, which is the kind of
domain cleanup a raw POI feed always needs.

`this.metros[0]` after the sort is the closest station - that is what the
detail page's preview line shows.

**Map init and listeners — `entry/src/main/ets/viewmodel/MarkerViewModel.ets`** (corrected, see `HW-09-0037`, `HW-09-0039`)

```typescript
@Observed
export class MarkerViewModel {
  @Track center: mapCommon.LatLng = { latitude: 32.02065982629459, longitude: 118.788899213002 };
  @Track metros: MetroInfo[] = [];
  @Track searchMarkers: map.Marker[] = [];
  private mapController?: map.MapComponentController;
  private mapEventManager?: map.MapEventManager;

  mapOptions?: mapCommon.MapOptions = { position: { target: this.center, zoom: 16 } };

  mapCallback?: AsyncCallback<map.MapComponentController> = async (err, mapController) => {
    if (err) {
      hilog.error(DOMAIN, 'MarkerVM', 'map init failed: %{public}s', JSON.stringify(err));
      return;                                   // FIX: the sample has no error branch
    }
    this.mapController = mapController;
    this.mapController.setZoomControlsEnabled(false);      // a small embedded map: no chrome
    this.mapController.setMyLocationEnabled(false);        // this map is about the listing,
    this.mapController.setMyLocationControlsEnabled(false);// not about the user
    this.mapEventManager = this.mapController.getEventManager();
    this.mapEventManager.on('mapLoad', this.mapLoadCallback);
    this.mapEventManager.on('markerClick', this.markerClickCallback);
  };

  mapLoadCallback = () => { this.setMarkers(); };
  markerClickCallback = (marker: map.Marker) => { this.moveCamera(marker); };

  release() {                                   // FIX: absent in the sample
    this.mapEventManager?.off('markerClick');
    this.mapEventManager?.off('mapLoad');
    this.mapController?.clear();
    this.mapEventManager = undefined;
    this.mapController = undefined;
  }
}
```

**Turn the map's own chrome off deliberately.** Three calls -
`setZoomControlsEnabled`, `setMyLocationEnabled`, `setMyLocationControlsEnabled` -
are what make this read as a property map rather than a navigation map, and
switching my-location off is also what lets the sample skip the location
permissions entirely.

**Adding markers on `mapLoad`, not on init** — same file (corrected, see `HW-09-0038`)

```typescript
async setMarkers() {
  if (!this.mapController) {
    return;
  }
  await this.init();                                  // the destination pin

  this.searchMarkers.forEach((m) => m.remove());      // FIX: the sample only truncates
  this.searchMarkers.length = 0;

  for (let i = 0; i < this.metros.length; i++) {
    const markerOptions: mapCommon.MarkerOptions = {
      position: this.metros[i].location,
      icon: $r('app.media.ic_location_blue'),
      clickable: true, draggable: false, flat: true,
      zIndex: 1                                       // below the destination pin
    };
    this.searchMarkers.push(await this.mapController.addMarker(markerOptions));
  }
}

private centerMarker?: map.Marker;

async init() {
  if (!this.mapController) {
    return;
  }
  this.centerMarker?.remove();                        // FIX: the sample re-adds every time
  this.centerMarker = await this.mapController.addMarker({
    position: this.center,
    alpha: 1, anchorU: 0.5, anchorV: 1,               // anchor at the bottom centre: the pin tip
    clickable: true, draggable: true, flat: false,
    zIndex: 2                                         // above the POI pins
  });
}
```

**Markers must be added on `mapLoad`, not in the init callback** - the
controller exists before the map surface is ready, and pins added too early
are dropped. `zIndex` 2 against 1 keeps the destination visible when a station
sits on top of it, and `anchorV: 1` puts the pin's point, not its centre, on
the coordinate.

`remove()` comes from `BaseOverlay`, which `Marker` extends; without it,
truncating `searchMarkers` loses the handles and the old pins stay drawn.

**Flying the camera — same file** (as shipped)

```typescript
moveCamera(marker: map.Marker) {
  if (!this.mapController) {
    return;
  }
  const cameraPosition: mapCommon.CameraPosition = {
    target: marker.getPosition(),
    zoom: 18                                   // closer than the overview 16
  };
  const cameraUpdate: map.CameraUpdate = map.newCameraPosition(cameraPosition);
  this.mapController.animateCamera(cameraUpdate, CommonConstants.MOVECAMERA_DURATION);  // 500 ms
}
```

`map.newCameraPosition` builds the `CameraUpdate`; `animateCamera` with a
duration is what makes the transition readable rather than a jump. `moveToCenter`
is the same three lines against `this.center` at zoom 16 - overview zoom for
the property, detail zoom for a station.

**The draggable sheet and the two-way selection — `entry/src/main/ets/components/SearchNearby.ets`** (as shipped)

```typescript
@State curHeight: string = CommonConstants.LIST_HEIGHT_FOLD;   // '20%'
@State curIndex: number = -1;
@Prop metros: MetroInfo[];
private scroller: ListScroller = new ListScroller();

expanding(offset: number) {
  if (offset > 0 && this.curHeight !== CommonConstants.LIST_HEIGHT_FOLD) {
    this.curHeight = CommonConstants.LIST_HEIGHT_FOLD;
  } else if (offset < 0 && this.curHeight !== CommonConstants.LIST_HEIGHT_EXPAND) {
    this.curHeight = CommonConstants.LIST_HEIGHT_EXPAND;   // '40%'
  }
}

// in build():
.height(this.curHeight)
.clip(true)
.animation({ duration: CommonConstants.EXPANDING_DURATION })
.gesture(PanGesture({ direction: PanDirection.Vertical })
  .onActionStart((event: GestureEvent) => { this.expanding(event.offsetY); }))

// row tap -> camera
.onClick(() => {
  this.curIndex = index;
  this.markerVM.moveCamera(this.markerVM.searchMarkers[index]);
})
```

**A two-detent sheet in six lines**: two height constants, a `PanGesture` that
picks one by drag direction, `.animation()` on the container so the height
change tweens, and `clip(true)` so the list does not spill while it shrinks.
`onActionStart` rather than `onActionUpdate` means one decision per drag
instead of a follow-the-finger resize - cruder, but it never fights the list's
own scrolling.

The pin-to-row direction is the half to redesign: the sample swaps the view
model's `markerClickCallback` field from `SearchNearby.aboutToAppear`, which
works only because of mount ordering (`HW-09-0040`). Prefer a `selectedIndex`
on the observed view model that both sides read.

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

Both permissions are `system_grant` - no runtime request, no user prompt.

No location permission is needed, and that is deliberate: the map is centred
on a fixed listing coordinate and my-location is switched off. Add
`ohos.permission.LOCATION` and `ohos.permission.APPROXIMATELY_LOCATION` only
if you turn `setMyLocationEnabled(true)` on - see `TOUR-03` for the request
flow.

Routing is by `routerMap` (`"routerMap": "$profile:route_map"`), with
`mapPage` mapped to `MapPageBuilder`.

Resource directories: `base`, `dark`, `en_US`, `zh_CN`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **AGC setup is mandatory**: enable Map Kit under Manage open capabilities,
  complete manual signing, and put the Client ID in `module.json5`. Without
  it, app identity verification fails with error `1002600003` and the map
  stays blank.
- Map Kit and `site.searchByText` are **mainland-China services**; coordinates
  are GCJ02.
- The search radius is a fixed `CommonConstants.DISTANCE = 2000`, used both as
  the query `radius` and as the post-filter bound. Change both together.
- `poiTypes: ['SUBWAY']` limits this to metro stations. Other categories -
  schools, hospitals, supermarkets - need separate queries or a wider
  `poiTypes` array, and the single `searchMarkers` array would then need a
  type per marker to keep the index binding meaningful.
- The listing content in `HouseInfoModel.ets` and the property coordinate are
  static; the preview card's background is a flat image, not a live map.

## Pitfalls

- **`HW-09-0034` — the module declares no permissions at all.** No `INTERNET`,
  no `GET_NETWORK_INFO`, so the tiles never load and `searchByText` fails.
  This is the first cause listed in Map Kit's own "Map Not Displayed" page.
- **`HW-09-0035` — the setup note stops at AGC and signing** and never
  mentions the `client_id` metadata in `module.json5`, which the sample also
  omits. The sibling sample in `TOUR-01` ships that block; this one does not.
- **`HW-09-0036` — `site.searchByText` is awaited with no `try`/`catch`.**
  Given the two gaps above, the rejection is the likely path, and it is
  unhandled: empty list, blank address line, nothing logged.
- **`HW-09-0037` — the listeners are never released,** and they live on a
  process-lifetime singleton, so each visit to the map page leaks another
  `mapLoad` and `markerClick` subscription bound to a destroyed component.
- **`HW-09-0038` — `setMarkers` truncates `searchMarkers` without calling
  `remove()`,** and `init()` adds another destination pin each time. A second
  `mapLoad` stacks a duplicate set of pins that can never be cleaned up.
- **`HW-09-0039` — the map init callback has no error branch,** in the
  document's snippet as well as in the code.
- **`HW-09-0040` — `SearchNearby` reassigns the view model's
  `markerClickCallback` field after registration.** `on()` captured the old
  reference; this works only because `mapCallback` happens to run after the
  list mounts.

## References

- `documentation/harmonyos-references/06_application-services/map-mapcomponent.md` - `MapComponent` and `mapCallback`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-mapcomponent
- `documentation/harmonyos-references/03_application-services/map-site.md` - `site.searchByText` and `SearchByTextParams`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-site
- `documentation/harmonyos-references/03_application-services/map-map-marker.md` - `Marker` and its setters
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-map-marker
- `documentation/harmonyos-references/06_application-services/map-map-baseoverlay.md` - `remove()`, `setVisible`, `setZIndex`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-map-baseoverlay
- `documentation/harmonyos-guides/04_application-services/map-presenting.md` - map initialization and the error branch
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-presenting
- `documentation/harmonyos-guides/04_application-services/map-listening.md` - `getEventManager()` and the event names
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-listening
- `documentation/harmonyos-guides/07_application-services/map-faq-1.md` - why a map does not display, and the two network permissions
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-faq-1
- `documentation/harmonyos-guides/07_application-services/map-faq-4.md` - the `off()` plus `clear()` teardown
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-faq-4
- `documentation/harmonyos-guides/04_application-services/map-config-agc.md` - enabling Map Kit in AGC
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-config-agc
- `documentation/harmonyos-guides/04_system/configuration_client_id.md` - the `client_id` metadata
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/configuration_client_id
- `TOUR-07` - naming a single marker; `TOUR-12` - adding and saving markers; `TOUR-01` - the same Map Kit setup at app scale
