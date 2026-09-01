---
id: UTIL-22
title: Compass - a controller that fans four sensor and location streams out to the page through listener setters
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/22_compass_effect.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/compass_effect-0000002317882746
sample: huawei_industry_tree/15_utilities/downloads/Compass.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.LocationKit", "@kit.PerformanceAnalysisKit", "@kit.SensorServiceKit"]
apis: [abilityAccessCtrl, bundleManager, common, geoLocationManager, hilog, sensor, window]
permissions: [ohos.permission.INTERNET, ohos.permission.APPROXIMATELY_LOCATION, ohos.permission.LOCATION]
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0055, HW-15-0056, HW-15-0101]
status: verified-with-fixes
---

## When to use

**Load this card when a screen has to combine several continuous hardware
streams into one readout** - here orientation, magnetic field, barometric
pressure, derived altitude, and GPS position, all feeding a single compass
page.

The pattern is a plain controller class that owns every subscription and
publishes each value through a `setXxxUpdateListener` setter. The page
registers six callbacks in `aboutToAppear` and thereafter is only a renderer.
No decorator, no state-management framework, no event bus - just a class with
fields and a listener per field.

That shape generalises well beyond a compass: a fitness screen mixing heart
rate and step count, a diving app mixing depth and temperature, a drone
controller mixing attitude and battery. The value is that the sensor code
never touches `@State` and so can be unit-tested and reused across pages.
**Read `HW-15-0055` before copying it**, though - the sample defines the
teardown and then never calls it, so the controller keeps all four streams
alive for the lifetime of the process.

## Feature checklist

- A dial that counter-rotates against the device heading, so the painted north
  needle always points at magnetic north while a fixed blue tick marks the top
  of the phone.
- A large heading readout combining the cardinal direction and the degrees -
  e.g. 北 with 45°.
- Latitude and longitude below it in degrees/minutes/seconds, with the N/S and
  E/W prefix picked by sign.
- Three cards along the bottom: atmospheric pressure (hPa), altitude (m),
  magnetic field strength (μT).
- Precise and approximate location are requested on first entry, with a second
  request routed to the settings sheet if the system no longer shows a dialog.
- Altitude comes from GPS when available and is otherwise derived from
  pressure.

## Architecture

One `entry` module, five source files, a clean two-layer split.

```
entry/src/main/ets
├── common/Constants.ets            AppStorage keys + SENSOR_INTERVAL (10 ms in ns)
├── common/Logger.ets               hilog wrapper
├── component/CompassView.ets       the dial: four stacked images under one rotate()
├── controller/CompassController.ets  every subscription, every conversion, the listener setters
├── entryability/EntryAbility.ets   window setup, avoid areas -> AppStorage
└── pages/CompassPage.ets           @Entry: permissions, listener wiring, layout
```

The documented tree matches the zip (the document's tree header reads
`├─ets/src/main/ets`, a typo for `entry/src/main/ets`; the contents are
correct).

**The design decision worth copying** is that `CompassController` exposes
neither its fields nor the sensor objects - only setters:

```typescript
public setAngleUpdateListener(listener: (angle: number) => void) {
  this.angleUpdateListener = listener;
}
```

with private no-op defaults, so a listener that was never registered is not a
null check at the notify site. `notifyPositionChanges()` fans one location
event out to three listeners (latitude, longitude, altitude) because that is
what one `Location` object carries. The page's job then reduces to formatting:
each callback assigns a `@State` `ResourceStr` built with `$r('app.string.x',
value)`, which keeps the units and the ° ′ ″ punctuation inside the resource
files rather than in the code.

The decision **worth avoiding** is the permission flow at the top of
`aboutToAppear`. It is asynchronous, its result branches never re-check, and
`getLocation()` is then called unconditionally at the bottom of the same
method - see `HW-15-0056`.

## Implementation steps

1. **Declare both location permissions** with `reason` and
   `usedScene.when: "inuse"`. `LOCATION` is precise, `APPROXIMATELY_LOCATION`
   approximate, and the precise one is refused unless the approximate one is
   requested with it.
2. **Check before requesting.** `checkAccessTokenSync` against the app's own
   token from `bundleManager.getBundleInfoForSelfSync(GET_BUNDLE_INFO_WITH_APPLICATION)`.
3. **Handle the three outcomes of the request** - granted, refused with a
   dialog, refused without a dialog. The last one means a permanent refusal:
   fall back to `requestPermissionOnSetting`.
4. **Subscribe to location only in the granted branch, exactly once**
   (`HW-15-0056`). The sample subscribes in that branch *and* unconditionally
   at the end of `aboutToAppear`.
5. **Subscribe the orientation sensor** at `SENSOR_INTERVAL` (10 000 000 ns =
   10 ms) and take `alpha` as the heading; `beta` and `gamma` are pitch and
   roll and are logged only.
6. **Take the heading modulo 360 before display** (`HW-15-0056`).
   `Math.round` turns anything from 359.5 upwards into 360, which no cardinal
   branch covers.
7. **Subscribe barometer and magnetometer at a much slower interval**
   (100 000 000 ns = 100 ms); neither drives an animation.
8. **Derive altitude from pressure inside the barometer callback** with
   `sensor.getDeviceAltitude(seaPressure, currentPressure, cb)`, and let a
   real GPS altitude overwrite it when one arrives.
9. **Counter-rotate the dial**, do not rotate the needle: `rotate({ z: 1,
   angle: -this.angle })` on the `Stack` of images, with a static tick drawn
   outside it.
10. **Call `destory()` from `aboutToDisappear`** (`HW-15-0055`). It exists,
    it is correct, and nothing in the sample invokes it.

## Verified snippets

All snippets are from `Compass.zip`. Corrected forms are marked.

**The dial — `entry/src/main/ets/component/CompassView.ets`** (as shipped)

```typescript
export struct CompassView {
  @Prop angle: number;

  build() {
    Stack() {
      Stack() {
        Image($r('app.media.compass_background')).width(290).height(290);
        Image($r('app.media.circle_secondary')).width(290).height(290);
        Image($r('app.media.circle_first')).width(290).height(290);
        Image($r('app.media.compass_arrow')).width(240).height(240);
      }
      .width(290).height(290)
      .rotate({
        x: 0,
        y: 0,
        z: 1,
        angle: -this.angle
      });

      Line()
        .width(3)
        .height(26)
        .backgroundColor('#254FF7')
        .borderRadius('50%')
        .margin({ bottom: 280 });
    }.scale({ x: 1, y: 1 });
  }
}
```

**The negative angle is the whole trick.** The heading reports how far the
*device* has turned from north; the dial must turn the opposite way so that
its painted north stays put in world space. Everything that belongs to the
compass rose - the four background images including the needle - sits in the
inner `Stack` and rotates as one unit; the blue `Line`, which marks where the
user is pointing, sits in the outer `Stack` and never moves. That separation
is what makes the readout and the graphic agree.

`rotate` is given an explicit axis `{x: 0, y: 0, z: 1}` rather than the bare
`angle`, which is the safe form: `z: 1` is a rotation in the screen plane.
`@Prop angle` means the parent's `@State` drives it one-way, so a re-render at
sensor rate touches only this subtree.

**The orientation subscription — `entry/src/main/ets/controller/CompassController.ets`** (corrected, see `HW-15-0056`)

```typescript
public getAngle() {
  try {
    sensor.on(
      sensor.SensorId.ORIENTATION,
      this.angleChange,
      {
        interval: Constants.SENSOR_INTERVAL      // 10_000_000 ns = 10 ms
      });
  } catch (err) {
    Logger.error('sensor error: ' + err);
  }
}

public angleChange = (data: sensor.OrientationResponse) => {
  this.angle = Math.round(data.alpha) % 360;     // FIX: sample yields 360 for alpha >= 359.5
  this.notifyAngleChanges();
};

public scalAngle(angle: number) {
  let str = parseInt(angle.toString());
  let position: ResourceStr = $r('app.string.N', str);
  if (str >= 0 && str < 23) {
    position = $r('app.string.N', str);
  } else if (str >= 23 && str < 68) {
    position = $r('app.string.EN', str);
  } // ... E, ES, S, WS, W, WN ...
  else if (str >= 338 && str < 360) {
    position = $r('app.string.N', str);
  }
  return position;
}
```

**`angleChange` is a field-initialised arrow function, not a method** - that is
deliberate and load-bearing. It is passed by reference to `sensor.on`, so a
regular method would lose `this` when the framework invoked it. Every callback
in this controller is written the same way.

The rounding bug is small and very visible: `alpha` in `[359.5, 360)` rounds
to 360, and the ladder in `scalAngle` ends at `str < 360`, so pointing almost
exactly north falls through every branch and shows the initialiser value while
the degree text reads `360°`. `% 360` after the round closes it. The bands
themselves are 45° wide and centred on the eight cardinals - north is the
split one, `[338, 360)` plus `[0, 23)`.

The interval is worth noting against its neighbours: orientation runs at 10 ms
because it drives a rotation the eye tracks; pressure and magnetic field run at
100 ms because they only feed a number in a card.

**Permissions and the subscription that should follow them — `entry/src/main/ets/pages/CompassPage.ets`** (corrected, see `HW-15-0055`, `HW-15-0056`)

```typescript
aboutToAppear(): void {
  // ... six setXxxUpdateListener registrations ...

  if (!checkPermissions()) {
    let atManager = abilityAccessCtrl.createAtManager();
    try {
      // 申请精确位置授权
      atManager.requestPermissionsFromUser(this.context,
        ['ohos.permission.LOCATION', 'ohos.permission.APPROXIMATELY_LOCATION'])
        .then((data) => {
          if (data.authResults[0] === -1 || data.authResults[1] === -1) {   // 用户拒绝授权
            if (data.dialogShownResults && (data.dialogShownResults[0] || data.dialogShownResults[1])) {
              // 有授权弹窗: the user saw the dialog and said no - leave location off
            } else {
              // 无授权弹窗: permanently refused, route to the settings sheet
              atManager.requestPermissionOnSetting(this.context,
                ['ohos.permission.LOCATION', 'ohos.permission.APPROXIMATELY_LOCATION'])
                .then((data: Array<abilityAccessCtrl.GrantStatus>) => {
                  if (data[0] === abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED) {
                    this.controller.getLocation();
                  }
                })
                .catch((err: BusinessError) => Logger.error('data:' + JSON.stringify(err)));
              return;
            }
          } else {
            this.controller.getLocation();
          }
        })
        .catch((err: BusinessError) => hilog.error(0x0000, TAG, `err: ${JSON.stringify(err)}`));
    } catch (err) {
      hilog.error(0x0000, TAG, `catch err->${JSON.stringify(err)}`);
    }
  } else {
    this.controller.getLocation();     // FIX: the sample calls this unconditionally, outside the branch
  }

  this.controller.getAngle();
  this.controller.getPressure();
  this.controller.getMagnetic();
}

aboutToDisappear(): void {
  this.controller.destory();           // FIX: absent in the sample
}
```

**`dialogShownResults` is what separates "said no" from "cannot be asked".**
After a permanent refusal `requestPermissionsFromUser` still resolves, with
`authResults` `-1` and `dialogShownResults` `false`, and shows nothing - so
without this check the app looks like it asked and was refused, when in fact
nothing appeared. `requestPermissionOnSetting` opens the settings sheet
instead.

The shipped code gets that logic right and then undoes it: line 154 calls
`this.controller.getLocation()` unconditionally after the whole `if`. On a
first run that fires before the user has answered, so
`geoLocationManager.on('locationChange', ...)` rejects with error 201 and the
rejection is unhandled; if the user then grants, the granted branch subscribes
a *second* listener over the first. Moving the call into the else branch (and
into the settings-sheet result) makes it exactly one subscription, only after
a grant.

The other half of the fix is the missing `aboutToDisappear`. `checkPermissions`
here is a free function using the synchronous APIs -
`getBundleInfoForSelfSync` and `checkAccessTokenSync` - which is legitimate in
`aboutToAppear` and avoids the async-ordering trap that the request path falls
into.

**The teardown that is never called — `CompassController.ets`** (as shipped)

```typescript
/**
 * Unsubscribe location changed.
 */
public destory() {
  geoLocationManager.off('locationChange');
  Logger.info('unsubscribe location changed');
  sensor.off(sensor.SensorId.ORIENTATION);
  sensor.off(sensor.SensorId.MAGNETIC_FIELD);
  sensor.off(sensor.SensorId.BAROMETER);
  Logger.info('unsubscribe orientation sensor');
}
```

**This method is correct and has zero callers** (`HW-15-0055`). It releases all
four streams, it is the right granularity - one teardown for one controller -
and `CompassPage` simply has no `aboutToDisappear` to call it from. Three
hardware sensors and a GPS listener therefore run until the process dies. On a
compass that is not a subtle cost: `ACCURACY` priority location plus a 10 ms
orientation stream is one of the more expensive things a phone can be doing.

This is the same defect class as `HW-15-0004` in `UTIL-23`, which is the
original anchor of the family: a sensor demo that models `on()` and not
`off()`. If you are writing one, wire the teardown first.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" },
  {
    "name": "ohos.permission.APPROXIMATELY_LOCATION",     // approximate
    "reason": "$string:getLocation",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  },
  {
    "name": "ohos.permission.LOCATION",                   // precise
    "reason": "$string:getLocation",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
]
```

- Both location permissions are `user_grant`, so `reason` and `usedScene` are
  mandatory and the reason resource must exist.
- Requesting `LOCATION` without `APPROXIMATELY_LOCATION` in the same array is
  refused outright - the pair is requested together here, correctly.
- `when: "inuse"` is right: the compass has no continuous task and must not
  hold location in the background.
- **No sensor permission is declared, and none is needed.** Orientation,
  magnetic field and barometer are all in the unrestricted set;
  `ACCELEROMETER`-class sensors would require `ohos.permission.ACCELEROMETER`.
- `INTERNET` is declared but nothing in the sample makes a network call.
- `deviceTypes` is `["phone", "tablet", "2in1"]`, which is optimistic - a 2in1
  is unlikely to carry a barometer, and the pressure and altitude cards then
  stay at their default placeholder forever.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- Requires real hardware: orientation, magnetometer and barometer are absent
  or stubbed on the emulator, and `sensor.on` throws (caught and logged) when
  the sensor does not exist. The dial then sits at 0°.
- The document notes that altitude is only obtainable by GPS in open ground;
  indoors the value shown is the pressure-derived estimate against a fixed sea
  level of 1013.2 hPa, which is weather-dependent and can be tens of metres out.
- Magnetic heading, not true heading - no declination correction is applied,
  and nearby metal or a magnetic case will skew it.
- The page has no calibration prompt and no accuracy indicator, both of which
  a shipping compass needs.
- Location uses `LocationRequestPriority.ACCURACY` with
  `distanceInterval: 100`, so the first fix can take several seconds and the
  coordinates only refresh every 100 m.

## Pitfalls

- **`HW-15-0055`** (B/medium, confirmed): `CompassController.destory()` is
  defined at `CompassController.ets:184` and has no callers anywhere in the
  sample; `CompassPage` has no `aboutToDisappear`. ORIENTATION,
  MAGNETIC_FIELD, BAROMETER and the `locationChange` listener keep sampling
  after the page is gone - a battery and privacy cost. New instance of the
  `UTIL-23` / `HW-15-0004` sensor-leak class. Fix: call `destory()` from
  `aboutToDisappear`.
- **`HW-15-0056`** (B/medium, confirmed): `CompassPage.ets:154` calls
  `getLocation()` unconditionally at the end of `aboutToAppear`, so on a
  first run or after a refusal it subscribes before permission exists and the
  resulting error-201 rejection is unhandled; when the user does grant, the
  granted branch at line 98 subscribes a second listener over the first. In the
  same finding, `CompassController.ets:126` rounds `alpha` without a modulo, so
  `alpha >= 359.5` displays `360°` and falls through every branch of
  `scalAngle` (which ends at `< 360`) to the initialiser. Fix: subscribe only
  in the granted branches, once; `% 360` before display.
- Not filed, but adjacent: `getAltitude()` is called from inside the barometer
  callback, so `sensor.getDeviceAltitude` is invoked on every pressure sample
  (every 100 ms), and its result competes with the GPS altitude for the same
  field.

## References

- `documentation/harmonyos-references/03_system/js-apis-sensor.md` -
  `sensor.on(ORIENTATION)` and `OrientationResponse.alpha/beta/gamma`,
  `MAGNETIC_FIELD`, `BAROMETER`, `sensor.getDeviceAltitude`, `sensor.off`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-sensor
- `documentation/harmonyos-references/06_application-services/js-apis-geolocationmanager.md` -
  `on('locationChange')`, `LocationRequest`, `LocationRequestPriority`, `off`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-geolocationmanager
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` -
  `checkAccessTokenSync`, `requestPermissionsFromUser`, `requestPermissionOnSetting`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-references/02_application-framework/js-apis-permissionrequestresult.md` -
  `authResults` and `dialogShownResults`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-permissionrequestresult
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-transformation.md` -
  `rotate` and its axis parameters
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-transformation
- `documentation/harmonyos-guides/07_application-services/location-guidelines.md` -
  which of the two location permissions is which, and the request flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/location-guidelines
- `documentation/harmonyos-guides/04_system/permissions-for-all.md`,
  `documentation/harmonyos-guides/04_system/permissions-for-all-user.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `UTIL-23` - the spirit level, the same sensor-leak defect at its origin
