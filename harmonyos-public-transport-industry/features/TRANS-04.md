---
id: TRANS-04
title: Real-time bus arrivals - coarse location, reverse geocoding, nearby lines
industry: 06_public_transport
doc: huawei_industry_tree/06_public_transport/docs/04_real_time_bus.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/real_time_bus-0000002300272510
sample: huawei_industry_tree/06_public_transport/downloads/实时公交服务示例代码.zip
kits: ["@kit.LocationKit", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.ArkUI"]
apis: [geoLocationManager.getCurrentLocation, geoLocationManager.SingleLocationRequest, geoLocationManager.LocatingPriority, geoLocationManager.isGeocoderAvailable, geoLocationManager.getAddressesFromLocation, abilityAccessCtrl.requestPermissionsFromUser, abilityAccessCtrl.requestPermissionOnSetting]
permissions: [ohos.permission.APPROXIMATELY_LOCATION]
min_api: 20
modules: [entry]
findings: [HW-06-0026, HW-06-0027, HW-06-0028, HW-06-0029, HW-06-0030, HW-06-0051]
status: verified-with-fixes
---

## When to use

Load this card when a screen must answer **"what is near me right now"** with a
one-shot position and a list - nearby bus stops and arrival times, nearby
stations, nearby branches or pickup points.

Two things make it worth reading even outside transit:

- It uses **coarse location only**. A nearby-stops list does not need
  metre-level precision, and asking for less is both cheaper and easier for the
  user to accept.
- It shows the **full permission ladder**: request, and if refused, offer the
  system settings sheet as a second chance.

## Feature checklist

- On open, request coarse location and resolve the device position once.
- Reverse-geocode that position into a human-readable place name shown in the
  header.
- List nearby lines with arrival information.
- A search bar over the list.
- If the permission is refused, offer to open the settings sheet rather than
  dead-ending.

## Architecture

Single `entry` module, flat and small.

```
entry/src/main/ets
├── common/CommonConstants.ets
├── component/Card.ets        one nearby-line row
├── component/SearchBar.ets   header with the resolved place name
├── entryability/EntryAbility.ets
├── mock/MockData.ets         the arrival data - this is mock, not a service
├── model/StationModel.ets
├── pages/MainPage.ets        owns the location and permission flow
├── pages/Nearby.ets          the nearby list
└── utils/Logger.ets
```

Flow: `aboutToAppear` → `requestPermission(context)` → request from user → if
refused, `requestPermissionOnSetting` → on grant, `getLocation()` →
`getCurrentLocation` → `isGeocoderAvailable` → `getAddressesFromLocation` →
place name into `@State province` → rendered by `SearchBar`.

> **The arrival data is mock.** The document says so. This practice covers the
> location and permission half; the real-time feed is yours to supply.

## Implementation steps

1. **Declare only `ohos.permission.APPROXIMATELY_LOCATION`.** Coarse location is
   enough for a nearby list, and requesting it alone is valid - unlike
   `LOCATION`, which must be requested together with the approximate one.
2. **Point `usedScene.abilities` at the ability that actually uses it**
   (`HW-06-0030`).
3. **Request, then check `authResults` before doing anything**
   (`HW-06-0026`).
4. **On refusal, offer `requestPermissionOnSetting`** - and check *its* result
   too, since the sheet resolves whether or not the user changed anything.
5. **Use `SingleLocationRequest` with a timeout** for a one-shot fix.
6. **Guard `isGeocoderAvailable()` and the result array** before reading an
   address (`HW-06-0028`).
7. **Pick the right `GeoAddress` field** for the label you are showing
   (`HW-06-0029`).

## Verified snippets

All snippets are from `实时公交服务示例代码.zip`. Corrected forms are marked.

**One-shot position with a timeout — `pages/MainPage.ets`** (as shipped)

```typescript
import { geoLocationManager } from '@kit.LocationKit';

getLocation() {
  let request: geoLocationManager.SingleLocationRequest = {
    'locatingPriority': geoLocationManager.LocatingPriority.PRIORITY_LOCATING_SPEED,
    'locatingTimeoutMs': 10000
  };
  try {
    geoLocationManager.getCurrentLocation(request).then((result) => {
      Logger.info('current location: ' + JSON.stringify(result));
      // ... reverse geocode ...
    })
      .catch((error: BusinessError) => {
        Logger.error('promise, getCurrentLocation: error=' + JSON.stringify(error));
      });
  } catch (err) {
    Logger.error('errCode:' + JSON.stringify(err));
  }
}
```

**This is the positioning pattern to copy in this industry.**
`SingleLocationRequest` with `PRIORITY_LOCATING_SPEED` and an explicit
`locatingTimeoutMs` is the right shape for "I need a rough fix, quickly, once" -
and unlike the architecture practice's version it has both a `catch` on the
promise and a `try` around the call, which are the two failure paths
`getCurrentLocation` actually has.

`LocatingPriority` alternatives: `PRIORITY_LOCATING_SPEED` for a fast, rougher
fix; `PRIORITY_ACCURACY` when precision matters more than latency.

**Reverse geocoding to a place name — same file** (corrected, see `HW-06-0028`, `HW-06-0029`)

```typescript
let isAvailable = geoLocationManager.isGeocoderAvailable();
if (isAvailable) {
  geoLocationManager.getAddressesFromLocation(
    { latitude: result.latitude, longitude: result.longitude },
    (err, data) => {
      if (err) {
        Logger.error('getAddressesFromLocation err: ' + JSON.stringify(err));
        return;
      }
      if (!data || data.length === 0) {     // FIX: shipped code reads data[0] straight away
        return;
      }
      // FIX: shipped code assigns subAdministrativeArea to a field named `province`
      this.district = data[0].subAdministrativeArea ?? '';
    });
}
```

`isGeocoderAvailable()` is worth keeping - reverse geocoding is not available on
every device or in every region, and calling through without it fails opaquely.

**`GeoAddress` fields, and which to use**: `administrativeArea` is the
province-level division, `subAdministrativeArea` the level below it, `locality`
the city, `placeName` the specific place. Pick the one whose level matches your
label - and pick the same one across the app. This industry currently uses
`subAdministrativeArea` here and `locality` in `TRANS-01`.

**The permission ladder — `pages/MainPage.ets`** (corrected, see `HW-06-0026`, `HW-06-0027`)

```typescript
const PERMISSION_ARRAY: Permissions[] = ['ohos.permission.APPROXIMATELY_LOCATION'];

// FIX: shipped version calls getLocation() here, before returning the verdict
async requestPermissionFromUser(context: Context): Promise<number[] | undefined> {
  const atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
  const requestStatus = await atManager.requestPermissionsFromUser(context, PERMISSION_ARRAY);
  return requestStatus?.authResults;
}

// FIX: shipped version calls getLocation() from the then branch without reading `data`
requestPermissionOnSetting(context: Context) {
  const atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
  atManager.requestPermissionOnSetting(context, PERMISSION_ARRAY)
    .then((data: Array<abilityAccessCtrl.GrantStatus>) => {
      if (data?.[0] === abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED) {
        this.getLocation();
      }
    })
    .catch((err: BusinessError) => {
      Logger.error('requestPermissionOnSetting: ' + JSON.stringify(err));
    });
}

async requestPermission(context: Context) {
  const permissionStatus = await this.requestPermissionFromUser(context);
  if (permissionStatus?.[0] === 0) {      // FIX: shipped code then indexes [0] unguarded below
    this.getLocation();
    return;
  }
  // refused: offer the settings sheet as a second chance
  this.requestPermissionOnSetting(context);
}
```

**`requestPermissionOnSetting` is the piece most samples miss.** Once a user has
refused, `requestPermissionsFromUser` will not show the dialog again - the
system suppresses it. `requestPermissionOnSetting` opens a settings half-sheet
so the user can change their mind without leaving the app. Its promise resolves
when the sheet closes, **whether or not anything changed**, so the returned
`GrantStatus` array is the only way to know the outcome.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.APPROXIMATELY_LOCATION",
    "reason": "$string:LOCATION_REASON",
    "usedScene": {
      "abilities": ["EntryAbility"],   // FIX: shipped file names a FormAbility that does not exist
      "when": "inuse"
    }
  }
]
```

Coarse location alone is a deliberate and correct choice here. `LOCATION`
(precise) may not be requested on its own - it must be paired with
`APPROXIMATELY_LOCATION` - but the reverse is fine, and a nearby-stops list does
not need metre-level accuracy. `when: "inuse"` matches a foreground-only feature.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. The industry's architecture practice
  requires API 24, so a combined project sits at 24.
- Arrival data is mock (`mock/MockData.ets`). There is no service integration in
  this practice.
- Coarse location is accurate to roughly 5 km; do not build distance-sorted
  results on it without saying so.
- Reverse geocoding requires the geocoder to be available, which is
  device- and region-dependent.

## Pitfalls

- **`HW-06-0026` — location is fetched whatever the user chose.**
  `requestPermissionFromUser` calls `getLocation()` before returning the verdict,
  and `requestPermissionOnSetting`'s `then` calls it again without reading the
  grant statuses. A double refusal still triggers three ten-second positioning
  attempts.
- **`HW-06-0027` — `permissionStatus[0]` is read on exactly the path where the
  guard admits the array may be missing.** The return type says `number[]` while
  the body returns `requestStatus?.authResults`.
- **`HW-06-0028` — `data[0]` is read without a length check.** `?? ''` guards the
  field, not the index, so an empty result throws before it applies.
- **`HW-06-0029` (probable) — a sub-administrative area is stored and shown as
  the province.** Recorded as probable: the local corpus copy of the
  geolocationmanager reference is an eleven-line stub with no `GeoAddress` field
  table, so this rests on the field names rather than a quoted definition.
  Confirm against the live reference before reporting.
- **`HW-06-0030` — `usedScene.abilities` names `FormAbility`,** which this
  module does not declare; the sample ships no widget at all.

## References

- `documentation/harmonyos-references/06_application-services/js-apis-geolocationmanager.md` - **stub in this corpus (11 lines)**; `GeoAddress`, `SingleLocationRequest` and `LocatingPriority` must be checked against the live reference
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-geolocationmanager
- `documentation/harmonyos-guides/04_system/request-app-permissions.md` - `authResults`, and the rule that `LOCATION` must be paired with `APPROXIMATELY_LOCATION`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-app-permissions
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `usedScene` semantics
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- `TRANS-01` - the industry architecture; its location code uses `locality` and a `locationChange` subscription instead, and is buggier
