---
id: TOUR-12
title: Long-press to drop a marker, reverse-geocode it and save the address
industry: 09_tourism
doc: huawei_industry_tree/09_tourism/docs/12_add_and_collect_map_marker.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_and_collect_map_marker-0000002402495269
sample: huawei_industry_tree/09_tourism/downloads/AddAndCollectMapMarker.zip
kits: ["@kit.MapKit", "@kit.LocationKit", "@kit.BasicServicesKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit"]
apis: [MapComponent, "map.MapComponentController", "map.MapEventManager", "on('mapLongClick')", "on('markerClick')", "on('mapLoad')", addMarker, "Marker.getId", "map.newCameraPosition", animateCamera, "geoLocationManager.getAddressesFromLocation", "geoLocationManager.ReverseGeoCodeRequest", "navi.getWalkingRoutes", "navi.DrivingRouteParams", "@Provide", "@Consume", "@Prop", "UIContext.getPromptAction", openCustomDialog]
permissions: ["ohos.permission.INTERNET", "ohos.permission.GET_NETWORK_INFO"]
min_api: 20
modules: [entry]
findings: [HW-09-0072, HW-09-0073, HW-09-0074, HW-09-0075, HW-09-0076, HW-09-0077, HW-09-0078, HW-09-0080]
status: verified-with-fixes
---

## When to use

Load this card when the user must be able to **mark a place the app does not
already know about** - long-press the map, get a pin, see what address that
pin is, and keep it. It is the "save this spot" interaction: a meeting point,
a parking space, a viewpoint, a pickup location.

Three pieces are worth taking:

- **`on('mapLongClick')`** delivers a `mapCommon.LatLng` for the pressed
  point, which is all a pin needs.
- **`geoLocationManager.getAddressesFromLocation`** turns those coordinates
  into a postal address - the reverse of geocoding, and the API that makes a
  dropped pin meaningful.
- **`navi.getWalkingRoutes`** gives distance and time on foot from a
  reference point, which is what makes a saved address useful rather than just
  recorded.

`Marker.getId()` is the identity that ties the pin on the map to the row in
the saved list. Getting that id at the right moment is the one hard part
(`HW-09-0077`).

## Feature checklist

- A full-screen map centred on a reference coordinate.
- Long-pressing anywhere drops a pin at that point.
- A bottom sheet opens with the address of the pressed point, in a short form
  (country, province, city, district) and a detailed form (place name).
- The sheet also shows walking distance and time from the reference point.
- A star on the sheet saves or unsaves the address; saving raises a
  confirmation toast dialog.
- Tapping an existing pin re-opens the sheet for it, flies the camera to it,
  and shows the star already filled if it is saved.
- A second sheet lists everything saved, with a count.

## Architecture

One `entry` module, one page and two sheet components, with the shared state
provided from the page.

```
entry/src/main/ets
├── components
│   ├── AddressDialog.ets       the address sheet: address, walk time, save star
│   └── CollectListDialog.ets   the saved list + collectItem row
├── constants/StyleConstants.ets  MarkerType, the reference coordinate, literals
├── entryability/EntryAbility.ets
├── entrybackupability/
└── pages/MapPage.ets           @Entry - the map, the callbacks, the geocoding
```

The documented tree matches the zip.

**All shared state is `@Provide` on the page:**

```
MapPage
├── @Provide collectArr: MarkerType[]        the saved addresses
├── @Provide currmarker: MarkerType          the one being looked at
├── @Provide showAddressDialog: boolean
└── @Provide showCollectListDialog: boolean
        │
        ├── AddressDialog      @Consume all four - reads currmarker, pushes into collectArr
        └── CollectListDialog  @Consume collectArr - renders the list
```

`@Provide` on a class instance observes first-level property changes, so
`this.currmarker.detailAddress = ...` written from an asynchronous geocode
callback in the page repaints the sheet with no notification code. That is why
the sheet can open immediately and fill in a moment later.

**The sheets are `if`-gated components, not dialogs.** `if (this.showAddressDialog) { AddressDialog() }`
inside a bottom-aligned `Stack`. That keeps them in the page's own layout and
under the page's own state, at the cost of the dismissal and modality
behaviour a real dialog would bring.

**`MarkerType` is the join.** One class carries the marker id, both address
forms, the coordinate, the walk time and distance, and the saved flag - and
the same class is used for the live selection and for each saved row.

## Implementation steps

1. **Declare `INTERNET` and `GET_NETWORK_INFO`** and add the `client_id`
   metadata (`HW-09-0072`).
2. **Write real reason strings** for any location permission you keep, and
   drop the ones you do not use (`HW-09-0073`, `HW-09-0074`).
3. **Register the three listeners through `getEventManager()`** and release
   them in `aboutToDisappear` (`HW-09-0075`).
4. **Turn the map's own chrome off** if this is a picking map rather than a
   navigation map.
5. **On long press: add the marker, await it, take its id**, and only then
   open the sheet (`HW-09-0077`).
6. **Reverse-geocode the pressed coordinate** and write the result into the
   provided `MarkerType`; guard the result array (`HW-09-0076`).
7. **Compute the walking route** from your reference point to the pin and
   format distance and duration.
8. **Save by copying** the current `MarkerType` into the array, and unsave by
   searching that array for the marker id.
9. **Re-derive the saved flag on every marker tap** from the array, rather
   than trusting a stored boolean.

## Verified snippets

All snippets are from `AddAndCollectMapMarker.zip`. Corrected forms are marked.

**Registering the three events — `entry/src/main/ets/pages/MapPage.ets`** (corrected, see `HW-09-0075`)

```typescript
private mapController?: map.MapComponentController;
private mapEventManager?: map.MapEventManager;
private mapLoadCallback = async () => { this.setMarkers(); };

mapCallback?: AsyncCallback<map.MapComponentController> = async (err, mapController) => {
  if (err) {
    hilog.error(0x0000, 'MapPage', 'map init failed: %{public}s', JSON.stringify(err));
    return;
  }
  this.mapController = mapController;
  this.mapController.setZoomControlsEnabled(false);        // a picking map, not a navigation map
  this.mapController.setMyLocationEnabled(false);
  this.mapController.setMyLocationControlsEnabled(false);
  this.mapEventManager = this.mapController.getEventManager();
  this.mapEventManager.on('mapLoad', this.mapLoadCallback);
  this.mapEventManager.on('markerClick', this.markerClickCallback);
  this.mapEventManager.on('mapLongClick', this.mapLongClickcallback);
};

aboutToDisappear(): void {                                 // FIX: absent in the sample
  this.mapEventManager?.off('mapLongClick');
  this.mapEventManager?.off('markerClick');
  this.mapEventManager?.off('mapLoad');
  this.mapController?.clear();
}
```

The three events are the whole interaction vocabulary of a pin-dropping map:
`mapLoad` to place the initial marker, `mapLongClick` to add one,
`markerClick` to reopen one.

**Long press to marker — same file** (corrected, see `HW-09-0077`)

```typescript
mapLongClickcallback = async (position: mapCommon.LatLng) => {
  const marker = new MarkerType('', '', '', { latitude: 0, longitude: 0 }, false, '', '', '');
  marker.location = position;

  const markerOptions: mapCommon.MarkerOptions = {
    position: { latitude: position.latitude, longitude: position.longitude },
    icon: $r('app.media.ic_location_blue'),
    clickable: true,
    draggable: false,
    flat: true,        // stays flat against the map as the camera tilts
    zIndex: 1
  };
  // FIX: the sample fires addMarker().then(...) and opens the sheet without waiting,
  //      so the id can land on a marker the user has already replaced
  const added = await this.mapController?.addMarker(markerOptions);
  marker.markerId = added ? added.getId() : '';

  this.currmarker = marker;
  this.showAddressDialog = true;
  await this.reverseGeocode(position);
};
```

**`Marker.getId()` is the only stable handle you get.** It is assigned by the
map when the marker is added, so it exists only after the promise resolves -
and everything downstream (matching a tap to a saved row, deleting a
favourite) keys on it.

**Reverse geocoding — same file** (corrected, see `HW-09-0076`)

```typescript
import { geoLocationManager } from '@kit.LocationKit';

private reverseGeocode(position: mapCommon.LatLng): void {
  const reverseGeocodeRequest: geoLocationManager.ReverseGeoCodeRequest = {
    'latitude': position.latitude,
    'longitude': position.longitude,
    'maxItems': 1
  };
  try {
    geoLocationManager.getAddressesFromLocation(reverseGeocodeRequest, async (err, data) => {
      if (err) {
        hilog.error(0x0000, 'MapPage', 'getAddressesFromLocation: %{public}s', JSON.stringify(err));
        return;                                  // FIX: the sample logs and falls through
      }
      if (!data || data.length === 0) {          // FIX: the sample tests only `if (data)`
        return;
      }
      const a = data[0];
      this.currmarker.detailAddress = a.placeName ?? '';
      this.currmarker.simpleAddress =
        [a.countryName, a.administrativeArea, a.locality, a.subLocality]
          .filter((s) => !!s).join('');          // FIX: sample concatenates all four unguarded
      await this.commutingCalculate(position.latitude, position.longitude);
    });
  } catch (err) {
    hilog.error(0x0000, 'MapPage', 'errCode:%{public}d, message:%{public}s', err.code, err.message);
  }
}
```

**An empty array is truthy.** The geocoder returns `[]` rather than an error
for a point it has no entry for - out at sea, in a desert - so `if (data)`
passes and `data[0].placeName` throws. And every field of a reverse-geocode
result is optional, so a plain template concatenation of the four
administrative levels renders `undefined` into the middle of the address
whenever one level is missing.

`maxItems: 1` is right for a pin: you want the single best match, not a list.

**Walking distance and time — same file** (as shipped)

```typescript
import { navi } from '@kit.MapKit';

async commutingCalculate(markerLatitude: number, markerLongitude: number) {
  const params: navi.DrivingRouteParams = {
    origins: [{ 'latitude': this.latitude, 'longitude': this.longitude }],
    destination: { 'latitude': markerLatitude, 'longitude': markerLongitude },
    language: 'zh_CN'
  };
  try {
    const result = await navi.getWalkingRoutes(params);
    const distance = result.routes[0].steps[0].distance as number;
    this.currmarker.currDisance = distance >= 1000 ? `${(distance / 1000).toFixed(1)}公里`
                                                  : `${distance}米`;
    const time = result.routes[0].steps[0].duration as number;
    this.currmarker.walkingTime = this.secondToTime(time);
  } catch (error) {
    const err: BusinessError = error as BusinessError;
    hilog.error(0x0000, 'MapPage',
      `error in getting driving routes. Code is ${err.code}, message is ${err.message}`);
  }
}
```

`getWalkingRoutes` takes the same `DrivingRouteParams` shape as the driving
call - `origins` is an array, `destination` a single point. Note the `1000`
threshold switching between kilometres and metres; that and the unit words are
the strings that should be resources (`HW-09-0078`).

`result.routes[0].steps[0]` is unguarded, but here the `try` does catch it,
because the call is awaited rather than callback-based.

**Save and unsave — `entry/src/main/ets/components/AddressDialog.ets`** (as shipped)

```typescript
@Consume currmarker: MarkerType;
@Consume collectArr: MarkerType[];

add() {
  const temp = new MarkerType(
    this.currmarker.markerId, this.currmarker.simpleAddress, this.currmarker.detailAddress,
    this.currmarker.location, this.currmarker.isCollect,
    this.currmarker.drivingTime, this.currmarker.walkingTime, this.currmarker.currDisance);
  this.collectArr.push(temp);
}

delete(id: string) {
  for (let i = 0; i < this.collectArr.length; i++) {
    if (this.collectArr[i].markerId === id) {
      this.collectArr.splice(i, 1);
      break;
    }
  }
}

// the star
Image(this.currmarker.isCollect ? $r('app.media.collect') : $r('app.media.uncollect'))
  .onClick(() => {
    if (this.currmarker.isCollect) {
      this.delete(this.currmarker.markerId);
      this.currmarker.isCollect = false;
    } else {
      this.add();
      this.currmarker.isCollect = true;
      this.getUIContext().getPromptAction().openCustomDialog({
        builder: () => { this.customDialogComponent(); },
        isModal: false, showInSubWindow: false,
        offset: { dx: 0, dy: '40%' }, height: 50, width: '90%'
      });
    }
  })
```

**Copying rather than pushing the live object** is the right call: `currmarker`
is replaced wholesale on the next long press, so pushing it by reference would
give the saved list an object the page then reuses. Note the ordering wart -
`add()` runs before `isCollect` is set, so every stored copy carries
`isCollect: false`; harmless only because the flag is re-derived on each
marker tap and never read back from the array.

**Re-deriving the saved flag on tap — `MapPage.ets`** (as shipped)

```typescript
markerClickCallback = (marker: map.Marker) => {
  this.moveCamera(marker);
  this.currmarker.markerId = marker.getId();
  this.currmarker.isCollect = false;
  for (let i = 0; i < this.collectArr.length; i++) {
    if (this.currmarker.markerId === this.collectArr[i].markerId) {
      this.currmarker.isCollect = true;
      break;
    }
  }
  // ... reverse geocode, then open the sheet
};
```

**Derive, do not store.** The star's state comes from a lookup in the saved
array on every tap, so the map and the list cannot drift apart - which is what
makes the stale `isCollect` in the stored copy harmless.

## Permissions & config

The sample declares `ohos.permission.LOCATION` and
`ohos.permission.APPROXIMATELY_LOCATION` and uses neither. What it needs is:

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

Both are `system_grant` - no prompt, no runtime request.

Add the two location permissions **only if** you replace the hardcoded
reference coordinate with a real fix. Then they need purpose-specific `reason`
strings, not `$string:EntryAbility_desc` (`HW-09-0073`), and a runtime request
- see `TOUR-03` for the flow.

Note that this document's 权限说明 describes the two location permissions
correctly (`LOCATION` for location information, `APPROXIMATELY_LOCATION` for
the approximate one) while `TOUR-03`'s document has them swapped. This one is
right.

Resource directories: `base` and `dark` only - no locale set.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` and
  `targetSdkVersion` are both `6.0.0(20)`.
- **AGC setup is mandatory** - Map Kit enabled, manual signing done, Client ID
  in `module.json5`.
- Map Kit, `getAddressesFromLocation` and `navi` are **mainland-China
  services**; coordinates are GCJ02 and the route request is pinned to
  `language: 'zh_CN'`.
- **Nothing is persisted.** `collectArr` is page state; every saved address is
  gone when the page is destroyed. A real feature needs preferences or an RDB
  store behind the same array.
- Markers are never removed. Long-pressing twenty times leaves twenty pins,
  and unsaving an address deletes the row but leaves the pin.
- The walking distance is measured from `StyleConstants.LATITUDE` /
  `LONGITUDE`, a fixed point near Nanjing - not from the user.
- The address sheet opens before the geocode returns, so it is briefly blank;
  there is no loading state.

## Pitfalls

- **`HW-09-0072` — no `INTERNET`, no `GET_NETWORK_INFO`, no `client_id`.** The
  map cannot load and the route call cannot run from a clean checkout, and the
  document's setup steps never mention `module.json5`.
- **`HW-09-0075` — three listeners registered, none released,** and no
  `aboutToDisappear`. Both retained callbacks write `@Provide` state on a
  destroyed page.
- **`HW-09-0077` — the marker id is written from an unawaited `.then()`** into
  a field the next long press replaces. Save too fast and the favourite gets
  an empty id; long-press twice quickly and one pin's id lands on another
  pin's address.
- **`HW-09-0076` — `data[0]` is dereferenced after testing only `if (data)`,**
  and the `err` branch logs without returning. An empty geocode result throws
  inside an async callback that the surrounding `try` cannot catch.
- **`HW-09-0074` — both location permissions are declared, never requested and
  never used;** the "my location" the walk is measured from is a constant.
- **`HW-09-0073` — the permission `reason` points at
  `$string:EntryAbility_desc`,** whose value is the word `description`. That
  is the sentence the user would be shown.
- **`HW-09-0078` — the units are hardcoded Chinese literals** in
  `secondToTime` and the distance formatter, with no locale directory.

## References

- `documentation/harmonyos-guides/04_application-services/map-listening.md` - `mapLongClick`, `markerClick`, `getEventManager()`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-listening
- `documentation/harmonyos-guides/04_application-services/map-marker.md` - `MarkerOptions`, `flat`, `zIndex`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-marker
- `documentation/harmonyos-references/03_application-services/map-common.md` - `LatLng`, `MarkerOptions`, `CameraPosition`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-common
- `documentation/harmonyos-references/03_application-services/map-navi-api.md` - `getWalkingRoutes` and `DrivingRouteParams`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/map-navi-api
- `documentation/harmonyos-references/06_application-services/js-apis-geolocationmanager.md` - `getAddressesFromLocation`, `ReverseGeoCodeRequest`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-geolocationmanager
- `documentation/harmonyos-guides/07_application-services/map-faq-1.md` - the two network permissions
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-faq-1
- `documentation/harmonyos-guides/07_application-services/map-faq-4.md` - the `off()` plus `clear()` teardown
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-faq-4
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `reason` and `usedScene` for user_grant permissions
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- `documentation/harmonyos-guides/03_application-framework/arkts-provide-and-consume.md` - `@Provide` and `@Consume`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-provide-and-consume
- `TOUR-06` - POI-driven markers and a bottom sheet list; `TOUR-07` - labelled markers and a custom info window; `TOUR-03` - the location permission request this sample skips
