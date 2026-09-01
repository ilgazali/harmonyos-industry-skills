---
id: KIDS-14
title: Distance-from-home alarm - continuous positioning against a geofence radius
industry: 08_children_education
doc: huawei_industry_tree/08_children_education/docs/14_distance_alarm.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/distance_alarm-0000002355769804
sample: huawei_industry_tree/08_children_education/downloads/DistanceAlarm.zip
kits: ["@kit.LocationKit", "@kit.MapKit", "@kit.ArkUI", "@kit.AbilityKit", "@kit.BackgroundTasksKit", "@kit.PerformanceAnalysisKit"]
apis: ["geoLocationManager.on('locationChange')", "geoLocationManager.off('locationChange')", ContinuousLocationRequest, SingleLocationRequest, getCurrentLocation, "map.calculateDistance", "map.convertCoordinate", MapComponent, MapCircleOptions, setMyLocationStyle, setInfoWindowVisible, "backgroundTaskManager.startBackgroundRunning", "abilityAccessCtrl.requestPermissionsFromUser", showToast]
permissions: ["ohos.permission.LOCATION", "ohos.permission.APPROXIMATELY_LOCATION", "ohos.permission.LOCATION_IN_BACKGROUND", "ohos.permission.KEEP_BACKGROUND_RUNNING", "ohos.permission.GET_NETWORK_INFO"]
min_api: 20
modules: [entry]
findings: [HW-08-0104, HW-08-0105, HW-08-0106, HW-08-0107, HW-08-0108, HW-08-0109, HW-08-0120]
status: verified-with-fixes
---

## When to use

Load this card for a **geofence alarm**: a radius around a fixed point, a live
position, and an alert when the two diverge. A child's safe area here, and
equally a delivery boundary, a site perimeter, a curfew zone.

This is the only sample in the industry that does the **whole** location
lifecycle, and it is the one to copy for that:

- **Runtime permission request, then subscribe** - and a settings dialog when
  the user has refused.
- **`on('locationChange')` paired with `off('locationChange')`**, with the
  unsubscribe actually called from `aboutToDisappear`. Almost nothing else in
  this corpus closes its location subscription.
- **A background task** so the alarm survives the app being backgrounded, which
  is the whole point of a child-safety alarm.

**Six findings.** The serious one is a unit bug: the distance field is written
in metres on two paths and in hundreds of metres on a third, and the alarm is a
comparison against that field.

## Feature checklist

- A map centred on the child's live position with a custom marker.
- A circle showing the safe radius around home.
- Home set by picking a point; the radius set by the parent.
- Continuous positioning updating the distance and the address.
- A toast warning when the child leaves the radius.
- Positioning continues in the background.

## Architecture

One `entry` module, one page, three support files. 1307 lines - the largest
sample in the industry.

```
entry/src/main/ets
├── common/CommonConstants.ets
├── entryability/EntryAbility.ets
├── entrybackupability/
├── model
│   ├── LocationInter.ets        the { latitude, longitude } interface
│   └── TabBar.ets               builders
├── pages/Index.ets              894 lines: map, permissions, positioning, alarm
└── util/PermissionsRequest.ets  the runtime permission flow
```

The documented tree matches the zip.

**The subscription lifecycle is the part worth taking**, because it is the one
this corpus almost always gets wrong:

```typescript
onLocationChange(): void {
  let request: geoLocationManager.ContinuousLocationRequest = {
    interval: 1,                       // see HW-08-0108
    locationScenario: 0x401
  };
  try {
    geoLocationManager.on('locationChange', request, this.locationChange);
    if (!this.judgeHasNet()) { /* toast, hide the info window */ }
  } catch (err) {
    hilog.error(0x0000, TAG, `onLocationChange failed, code: ${err.code}, message: ${err.message}`);
    this.locationFailedAlert(err.code);
  }
}

offLocationChange(): void {
  try {
    geoLocationManager.off('locationChange', this.locationChange);
  } catch (err) {
    hilog.error(0x0000, TAG, `offLocationChange failed, code: ${err.code}, message: ${err.message}`);
  }
}

aboutToDisappear(): void {
  if (this.isBackgroundRunning) {
    this.stopContinuousTask();          // background task owns the subscription
  } else {
    if (this.isOnLocationChange) {
      this.offLocationChange();
    }
  }
}
```

**The callback is a field, not an inline arrow.** `off` matches on the callback
reference, so `on(..., this.locationChange)` and `off(..., this.locationChange)`
only pair up because `locationChange` is a stable property:

```typescript
locationChange = (location: geoLocationManager.Location): void => { ... };
```

Passing an inline closure to `on` makes the subscription impossible to cancel -
which is exactly the defect in `SPORT-11` and `KIDS-12`.

**`aboutToDisappear` branches on whether the background task is running**, so
the subscription is torn down through whichever path owns it. That distinction
is what makes the alarm survive backgrounding without leaking when it does not.

## Implementation steps

1. **Declare all five permissions the document lists** - including `INTERNET`
   (`HW-08-0105`).
2. **Put `client_id` at module level** with a placeholder (`HW-08-0106`).
3. **Request the runtime permissions**, and offer the settings dialog on
   refusal.
4. **Subscribe with `on('locationChange')`** at an interval proportional to the
   radius (`HW-08-0108`).
5. **Compute the distance one way** - `map.calculateDistance` (`HW-08-0107`).
6. **Keep the field in one unit** (`HW-08-0104`).
7. **Unsubscribe in `aboutToDisappear`**, or hand ownership to the background
   task.

## Verified snippets

All snippets are from `DistanceAlarm.zip`. Corrected forms are marked.

**The alarm — `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-08-0104` and `HW-08-0109`)

```typescript
locationChange = (location: geoLocationManager.Location): void => {
  this.nowLocation.latitude = location.latitude;
  this.nowLocation.longitude = location.longitude;

  // FIX: the sample uses its own haversine helper here and
  // map.calculateDistance elsewhere - see HW-08-0107.
  this.distanceToHome = map.calculateDistance(this.nowLocation, this.homeLocation);
  this.isSafe = this.distanceToHome < this.distanceAlarm;

  this.getAddress({ latitude: location.latitude, longitude: location.longitude }).then(() => {
    let style: mapCommon.MyLocationStyle = {
      anchorU: 0.5, anchorV: 1, displayType: 2, icon: $r('app.media.girl')
    };
    this.mapController?.setMyLocationStyle(style);
    if (!this.isSafe) {
      // FIX: the sample builds `警告：当前离家：${...}m` as a literal
      this.getUIContext().getPromptAction().showToast({
        message: $r('app.string.distance_warning', Math.floor(this.distanceToHome)),
        duration: 2000
      });
    }
  });
};
```

**`MyLocationStyle` with a custom `icon` is how the "my location" dot becomes a
child avatar** - `displayType: 2`, and `anchorU: 0.5, anchorV: 1` puts the
icon's bottom-centre on the coordinate, which is what you want for a pin-shaped
image rather than a dot.

**The distance and the safe flag are computed before the address lookup**, so
the alarm does not wait on a network round trip. Only the toast is inside the
`.then`, which is the right split - though it means the warning is delayed by
the geocode.

**Requesting the permissions — `entry/src/main/ets/util/PermissionsRequest.ets` and the page**

```typescript
// on refusal, offer Settings rather than re-asking
this.getUIContext().showAlertDialog({
  title: $r('app.string.unauthorized'),
  message: $r('app.string.enable_permissions'),
  buttons: [
    { text: $r('app.string.cancel'), color: $r('app.color.dialog_text_color') },
    { text: $r('app.string.sure'), color: $r('app.color.dialog_text_color'), primary: true }
  ],
}, (err, data) => {
  hilog.info(0x000, 'testTag', 'showDialog success callback, click button: ' + data.index);
});
```

**Every string in this dialog is a resource.** That is the norm in this file -
which is what makes the one hardcoded alarm message stand out (`HW-08-0109`).

**Two positioning modes — same file** (as shipped)

```typescript
// continuous, for the live alarm
let request: geoLocationManager.ContinuousLocationRequest = {
  interval: 1,
  locationScenario: 0x401
};

// one-shot, for a single distance reading
async getDistance(homeLocation: LocationInter): Promise<void> {
  let request: geoLocationManager.SingleLocationRequest = {
    locatingPriority: 0x502,
    locatingTimeoutMs: 10000
  };
  geoLocationManager.getCurrentLocation(request).then((location) => {
    // FIX: the sample divides by 100 here and by nothing on the other paths
    this.distanceToHome = Math.floor(map.calculateDistance(location, homeLocation));
  });
}
```

**`SingleLocationRequest` with an explicit `locatingTimeoutMs` is the right
shape for a one-shot** - a single fix that never returns would otherwise hang
the caller silently. The continuous request has no equivalent, which is why the
interval matters so much.

**Distance and geometry — same file** (as shipped)

```typescript
let mapCircleOptions: mapCommon.MapCircleOptions = {
  center: {
    latitude: this.homeLocation.latitude,
    longitude: this.homeLocation.longitude
  },
  radius: this.distanceAlarm,          // the safe radius, drawn on the map
  // ...
};

await map.convertCoordinate(mapCommon.CoordinateType.WGS84, mapCommon.CoordinateType.GCJ02, {
  // Location Kit returns WGS84; Map Kit draws in GCJ02
});
```

**`convertCoordinate` between WGS84 and GCJ02 is mandatory when mixing the two
kits**, and this sample does it - a position plotted without the conversion sits
tens to hundreds of metres from where it belongs. `SPORT-11` and `TOUR-06` need
the same call.

## Permissions & config

Five declared, and the document lists five - but they are not the same five.
`INTERNET`, which the document names, is missing (`HW-08-0105`):

| Permission | Declared | In the document's 权限说明 |
| --- | --- | --- |
| `ohos.permission.LOCATION` | yes | yes |
| `ohos.permission.APPROXIMATELY_LOCATION` | yes | yes |
| `ohos.permission.LOCATION_IN_BACKGROUND` | yes | yes |
| `ohos.permission.KEEP_BACKGROUND_RUNNING` | yes | yes |
| `ohos.permission.GET_NETWORK_INFO` | yes | no |
| `ohos.permission.INTERNET` | **no** | **yes** |

**Each permission has its own `reason` string**, not one shared placeholder -
`$string:location_permission`, `$string:fuzzy_location_permission`,
`$string:location_background`. That is what the review gate expects and what
`SPORT-01` fails to do.

**`LOCATION` and `APPROXIMATELY_LOCATION` are requested together.** Precise
location cannot be granted without the approximate one; asking for `LOCATION`
alone yields the fuzzy grant silently.

`client_id` is present but nested inside `EntryAbility` rather than at module
level, and carries a live value (`HW-08-0106`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Requires Map Service enabled in AppGallery Connect and a manual signing
  configuration**, per the document's 说明 section.
- **The alarm is a toast**, not a notification - so it is invisible while the
  app is backgrounded, which is precisely when the background positioning
  permissions keep it running.
- **Home is a single point** set in-app; there is no persistence, so it is lost
  with the page, along with the radius.
- **One radius, one child.** No schedule, no multiple zones, no history of
  crossings.
- **Positioning runs at a one-second interval in the background**
  (`HW-08-0108`).
- The sample carries its own `haversine` helper alongside `map.calculateDistance`
  (`HW-08-0107`).

## Pitfalls

- **`HW-08-0104` — `distanceToHome` is written in metres on two paths and in
  hundreds of metres on a third** (`map.calculateDistance(...) / 100`), and the
  alarm is a comparison against that field. After the one-shot path runs, a
  child two kilometres away is recorded as 20 and the alarm is silently
  disabled. The message labels every value `m`.
- **`HW-08-0105` — `ohos.permission.INTERNET` is missing** although the
  document's own 权限说明 lists it and the sample is built on Map Kit. Because
  `GET_NETWORK_INFO` *is* declared, the app can see a network it cannot use.
- **`HW-08-0106` — `client_id` is nested inside `EntryAbility`** instead of at
  module level, where Map Kit reads it, and ships a live nineteen-digit value
  where the sibling samples ship a mask.
- **`HW-08-0107` — two distance implementations coexist.** The hand-written
  `haversine` is used for the alarm; `map.calculateDistance`, which the document
  prescribes, is used only in the one-shot path. Its doc comment says kilometres
  and its body returns metres.
- **`HW-08-0108` — `interval: 1` continuous positioning, in the background,**
  with a reverse-geocoding request on every fix - for a threshold measured in
  hundreds of metres that cannot be crossed meaningfully faster than every ten
  seconds on foot.
- **`HW-08-0109` — the warning message is a hardcoded Chinese template
  literal,** in a page whose every other string, including the toast six lines
  above it, resolves through `$r`.

## References

- `documentation/harmonyos-guides/07_application-services/location-guidelines.md` - `on`/`off('locationChange')`, `ContinuousLocationRequest`, `SingleLocationRequest`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/location-guidelines
- `documentation/harmonyos-guides/07_application-services/location-permission-guidelines.md` - requesting `LOCATION` with `APPROXIMATELY_LOCATION`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/location-permission-guidelines
- `documentation/harmonyos-guides/07_application-services/map-calculate-distance.md` - `map.calculateDistance`, the prescribed mechanism
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-calculate-distance
- `documentation/harmonyos-guides/07_application-services/map-convert-coordinate.md` - WGS84 to GCJ02
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-convert-coordinate
- `documentation/harmonyos-guides/07_application-services/map-faq-1.md` - the network permissions Map Kit needs
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-faq-1
- `documentation/harmonyos-guides/04_application-services/map-config-agc.md` - `client_id` placement and AppGallery Connect setup
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-config-agc
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` - `requestPermissionsFromUser`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `KIDS-12` - the industry's other map sample, which mocks its positions instead of taking them
- `SPORT-11` - the sports trail sample, which never unsubscribes from `locationChange`
- `KIDS-04` - the other background-behaviour sample, whose lock also fails while backgrounded
