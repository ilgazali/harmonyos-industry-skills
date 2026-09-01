---
id: SOCIAL-06
title: Nearby people - continuous location behind a switch, Haversine distance, and a 5 km filter
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/06_near_people.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/near_people-0000002237251858
sample: huawei_industry_tree/14_social_communication/downloads/NeiborPeople.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.CryptoArchitectureKit", "@kit.LocationKit", "@kit.PerformanceAnalysisKit"]
apis: [abilityAccessCtrl, bundleManager, common, cryptoFramework, geoLocationManager, hilog, window]
permissions: [ohos.permission.LOCATION, ohos.permission.APPROXIMATELY_LOCATION]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0011, HW-14-0012, HW-14-0087]
status: verified-with-fixes
---

## When to use

**Load this card when a feature needs the user's live position and a
distance-ranked list around it** - nearby people, nearby shops, drivers on a
map, an event feed sorted by how far away it is. The sample is the social
version: a switch that starts continuous positioning, a second switch that
turns the list on, and a row per user showing metres.

Three parts are worth lifting separately. The permission helper
(`PermissionsView`) is a self-contained request/re-request/check unit you can
drop into any user_grant flow. `OpenContinuosLocation` is the minimal
`geoLocationManager.on('locationChange', ...)` subscription. And
`calculateDistance` is a textbook Haversine that needs no map service, no
network and no key - if all you need is "how far apart are these two
coordinates", you do not need Map Kit.

**Read `HW-14-0011` before adopting the location half.** The sample subscribes
and never unsubscribes, so it is the shape of the API call that transfers, not
its lifecycle. Continuous positioning that outlives the screen it was started
for is a battery and privacy problem, and every toggle cycle here adds another
listener.

## Feature checklist

- A two-tab page; the first tab holds the whole feature.
- A 定位开关 (location switch) row: turning it on requests both location
  permissions, and on grant shows the live longitude and latitude beside the
  switch, refreshed as the device moves.
- Refusal shows a 定位未开启 toast and the switch returns to off.
- A 附近的人 (nearby people) switch that only produces a list while location
  is on; otherwise it toasts the reason.
- With both on, a list of users appears after a short delay, each row showing
  an avatar, a name, interest chips and a distance in metres.
- Only users within 5 km are listed.
- Tapping a row toasts that user's name and coordinates.
- A refresh icon clears and regenerates the list.
- Turning location off clears the list and forces the nearby switch off.

## Architecture

One `entry` module. Two pages, three utilities, no persistence.

```
entry/src/main/ets
├── entryability/EntryAbility.ets   full-screen layout, avoid areas + the UIContext into AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── pages
│  ├── NearMainPage.ets             @Entry, the tabs, the two switches, the list, the toggle handlers
│  └── NearbyUser.ets               the NearbyUser class, the coordinate interface, and the row component
└── utils
   ├── LocationUtil.ets             OpenContinuosLocation - the geoLocationManager subscription
   ├── PermissionsView.ets          request / re-request-on-setting / check, exported as a singleton
   └── ProcessData.ets              the crypto-backed RNG, generateMockData, calculateDistance
```

The documented tree matches the zip.

**The design decision worth copying** is `PermissionsView`. It is a plain
class exported as `export default new PermissionsView()`, so every page shares
one instance and the call site is one line:

```typescript
await PermissionsView.commonRequestPermissions(permissions, this.getUIContext());
this.isLocationEnabled = await PermissionsView.checkPermissions(permissions);
```

`commonRequestPermissions` encodes the whole user_grant protocol: check
first, request only if missing, and if `dialogShownResults[0]` comes back
anything but `true` - meaning the system showed no dialog because the user
already refused permanently - escalate to `requestPermissionOnSetting`. The
page then re-checks rather than trusting the request's return value. That
check-after-request is the part most implementations skip, and it is why this
page can correctly reset its own switch.

**The decision worth avoiding** is in `LocationUtil`. `OpenContinuosLocation`
takes the caller's coordinate object, hands it to the callback closure to be
mutated, and returns the same object:

```
NearMainPage:  this.currentLocation = OpenContinuosLocation(this.currentLocation)
                                          |
LocationUtil:  locationCallback --------> mutates the raw object it captured
```

The closure holds the *raw* object reference, not the `@State` proxy that
ArkUI installs on assignment, so later position updates mutate an object the
framework is not observing. Return a fresh object from the callback into a
setter the page owns, or expose the subscription as a callback the page binds
to its own state.

## Implementation steps

1. **Declare both permissions** in `module.json5` with `reason`, `usedScene`
   and `when: "inuse"`. `LOCATION` is precise, `APPROXIMATELY_LOCATION` is
   approximate, and the precise one is refused unless the approximate one is
   requested alongside it.
2. **Request from the switch's `onChange`, then re-check.** Never set the
   switch from the request's own return value - only `checkPermissions` tells
   you the real grant state.
3. **Catch the settings escalation.** `requestPermissionOnSetting` rejects
   when the user backs out of the settings sheet, and the rejection travels
   out of the `async onChange` unhandled, leaving the switch stuck on
   (`HW-14-0012`).
4. **Subscribe once, and keep the callback.** Hold the exact function
   reference you passed to `geoLocationManager.on` so you can pass it to
   `off` (`HW-14-0011`).
5. **Unsubscribe on toggle-off and in `aboutToDisappear`.** Re-enabling calls
   `on` again; without an intervening `off` the callbacks stack one per cycle
   (`HW-14-0011`).
6. **Do not build the list until a real fix has arrived.** The seed is
   `{latitude: 0, longitude: 0}` and the sample generates users 700 ms after
   the toggle, which puts them around null island on first run
   (`HW-14-0012`).
7. **Compute distance with Haversine on metres**, then filter - not the other
   way round. The filter reads the field the map step just wrote.
8. **Key the list on a stable id.** `ForEach(..., (user: NearbyUser) =>
   user.id.toString())` - the sample gets this right, unlike most of the chat
   samples in this industry.

## Verified snippets

All snippets are from `NeiborPeople.zip`. Corrected forms are marked.

**The subscription — `entry/src/main/ets/utils/LocationUtil.ets`** (corrected, see `HW-14-0011`)

```typescript
import { geoLocationManager } from '@kit.LocationKit';
import { hilog } from '@kit.PerformanceAnalysisKit';
import { GeneratedObjectLiteralInterface } from '../pages/NearbyUser';

// FIX: the sample keeps no reference, so nothing can ever be unsubscribed
let locationCallback: ((location: geoLocationManager.Location) => void) | null = null;

export function OpenContinuosLocation(myLocation: GeneratedObjectLiteralInterface): GeneratedObjectLiteralInterface {
  let request: geoLocationManager.ContinuousLocationRequest = {
    'interval': 1,
    'locationScenario': geoLocationManager.UserActivityScenario.NAVIGATION
  };
  CloseContinuosLocation();                       // FIX: never stack two subscriptions
  locationCallback = (location: geoLocationManager.Location): void => {
    if (location !== null) {
      myLocation.latitude = location.latitude;
      myLocation.longitude = location.longitude;
    }
  };
  try {
    geoLocationManager.on('locationChange', request, locationCallback);
    return myLocation;
  } catch (err) {
    hilog.error(0x0000, 'errCode:', JSON.stringify(err));
    return myLocation;
  }
}

// FIX: absent in the sample - GPS runs until the process dies
export function CloseContinuosLocation(): void {
  if (locationCallback) {
    geoLocationManager.off('locationChange', locationCallback);
    locationCallback = null;
  }
}
```

**Two request fields decide the cost of this feature.** `interval: 1` asks for
a fix every second, and `locationScenario:
UserActivityScenario.NAVIGATION` asks for the highest-accuracy, lowest-latency
mode the platform has. That pair is right for turn-by-turn navigation and
wrong for a nearby-people list, where `DAILY_LIFE_SERVICE` and an interval of
tens of seconds would give the same list at a fraction of the power. The
file's own doc comment lists all five scenarios - and lists `TRANSPORT` twice,
which is a hint that it was not read closely.

`geoLocationManager.off` matches on the callback identity, which is the whole
reason the reference has to be stored. The shipped code creates the closure
inline inside the function, so after `OpenContinuosLocation` returns there is
no way to name it again. Every time the user flips the switch off and on, a
second, third, fourth callback is added to the same event - all still writing
into the same coordinate object, all still holding the GPS awake.

**The switch — `entry/src/main/ets/pages/NearMainPage.ets`** (corrected, see `HW-14-0011`, `HW-14-0012`)

```typescript
Toggle({ type: ToggleType.Switch, isOn: this.isLocationEnabled })
  .onChange(async (isOn) => {
    this.handleLocationToggle(isOn);
    this.isLocationEnabled = isOn;
    if (!isOn) {
      CloseContinuosLocation();                   // FIX: absent - the subscription outlives the switch
      return;
    }
    let permissions: Array<Permissions> =
      ['ohos.permission.LOCATION', 'ohos.permission.APPROXIMATELY_LOCATION'];
    try {
      await PermissionsView.commonRequestPermissions(permissions, this.getUIContext()); // 通用权限申请示例
    } catch (err) {
      hilog.error(0x0000, 'testTag', `permission request rejected: ${JSON.stringify(err)}`);
    }                                             // FIX: an un-caught refusal left the switch ON
    // 示例，权限申请成功后，进行对应业务处理
    this.isLocationEnabled = await PermissionsView.checkPermissions(permissions);
    if (this.isLocationEnabled) {
      this.getUIContext().getPromptAction().showToast({
        message: $r('app.string.location_is_on'),
        bottom: 68,
      });
      this.currentLocation = OpenContinuosLocation(this.currentLocation);
      if (this.isNearbyEnabled) {
        this.loadNearbyUsers();
      }
    } else {
      this.getUIContext().getPromptAction().showToast({
        message: '定位未开启',
        bottom: 68,
      });
    }
  })
```

**The `checkPermissions` line after the request is the load-bearing one.**
`requestPermissionsFromUser` resolving does not mean anything was granted -
after a permanent refusal it resolves having shown no dialog at all. So the
page throws the request's result away and asks the token manager, and assigns
that answer to `isLocationEnabled`. The switch is therefore driven by the
permission state, not by the gesture, which is exactly right.

What breaks it is the missing `try`. `requestPermissionsOnSetting` inside
`commonRequestPermissions` is awaited with no handler; when the user closes
the settings sheet without granting, the rejection escapes an `async`
`onChange` with nobody to catch it, the two lines after it never run, and
`isLocationEnabled` keeps the `true` that was assigned at the top - a switch
showing "on" with no permission behind it (`HW-14-0012`).

The `!isOn` early return is also new. The shipped handler does nothing on
toggle-off beyond clearing the list, so positioning continues
(`HW-14-0011`).

**Mock users and the distance filter — `entry/src/main/ets/utils/ProcessData.ets`** (as shipped)

```typescript
// 生成模拟数据
export function generateMockData(isLocationEnabled: boolean,
  currentLocation: GeneratedObjectLiteralInterface | null, uiContext: UIContext | undefined): NearbyUser[] {
  const mockNames = ['Alice', 'Bob', 'Charlie', 'David', 'Eva', 'Yegli'];
  const mockAvatarUrl: Resource[] = [$r('app.media.Alice'), $r('app.media.Bob'),
    $r('app.media.Charlie'), $r('app.media.David')];

  if (!isLocationEnabled) {
    uiContext?.getPromptAction().showToast({
      message: $r('app.string.location_is_not_on'),
      bottom: 68,
    });
    return [];
  }

  return mockNames.map((name, index) => {
    let userLat = currentLocation!.latitude + (doRandBySync() * 0.01);
    let userLon = currentLocation!.longitude + (doRandBySync() * 0.01);

    const newUser = new NearbyUser(index + 1, name, -1, userLat, userLon, mockAvatarUrl[index % 4]);

    newUser.distance = calculateDistance(currentLocation !== null ? currentLocation.latitude : 0,
      currentLocation !== null ? currentLocation.longitude : 0, userLat, userLon);
    return newUser;

  }).filter(newUser => newUser.distance <= 5000); // 筛选5公里内用户
}

// 新增Haversine距离计算函数
export function calculateDistance(lat1: number, lon1: number, lat2: number, lon2: number): number {
  const R = 6371e3; // 地球半径（米）
  const φ1 = lat1 * Math.PI / 180;
  const φ2 = lat2 * Math.PI / 180;
  const pφ = (lat2 - lat1) * Math.PI / 180;
  const pλ = (lon2 - lon1) * Math.PI / 180;

  const a = Math.sin(pφ / 2) * Math.sin(pφ / 2) +
    Math.cos(φ1) * Math.cos(φ2) *
    Math.sin(pλ / 2) * Math.sin(pλ / 2);
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  return R * c; // 返回米数
}
```

**Haversine is the whole distance service.** `R = 6371e3` is the mean Earth
radius in metres, so the result is metres and the `<= 5000` filter reads as
five kilometres with no unit conversion anywhere. Using `atan2(√a, √(1-a))`
rather than `asin(√a)` is the numerically stable form - it does not lose
precision for nearly-antipodal points. The Greek identifiers (`φ1`, `pφ`,
`pλ`) come straight from the standard formulation; keep them or rename them,
but do not reshuffle the terms.

The offsets are `doRandBySync() * 0.01` degrees, roughly 0-1.1 km of latitude,
so every generated user lands inside the filter and it never actually rejects
anyone. That is fine for a demo and worth knowing before you conclude the
filter is tested.

`doRandBySync` reaches for `cryptoFramework.createRandom()` and converts four
bytes to a float - cryptographic randomness for decorative jitter. It is
harmless, but `Math.random()` is the honest call here; the interesting part is
the conversion (`view.getUint32(0, true) / 4294967296`) if you ever do need a
uniform float from secure bytes.

Note the shape: `map` fills `distance`, then `filter` reads it. Each user is
constructed with `-1` for distance and corrected one line later - a two-step
that only works because `map` runs to completion per element before `filter`
sees it.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.LOCATION",                    // PRECISE location
    "reason": "$string:location_permission",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  },
  {
    "name": "ohos.permission.APPROXIMATELY_LOCATION",      // APPROXIMATE location
    "reason": "$string:fuzzy_location_permission",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
]
```

- Both are `user_grant`, so `reason` and `usedScene` are mandatory and both
  reason strings must exist in `resources/base/element/string.json`.
- `LOCATION` is rejected at request time unless `APPROXIMATELY_LOCATION` is in
  the same array; the sample requests them together, which is correct.
- `when: "inuse"` is right - there is no continuous task here, so the app must
  not hold position in the background. That makes the missing `off`
  (`HW-14-0011`) a correctness problem as well as a battery one: the
  declaration promises foreground-only use that the code does not honour.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The nearby list is synthetic.** Six names, four avatars, coordinates
  jittered around your own position. There is no server, no presence, no
  refresh-on-move: the refresh icon simply regenerates the same random set.
- `interval: 1` with `UserActivityScenario.NAVIGATION` is a navigation-grade
  request used for a browse list - expect the power draw to match navigation,
  not a chat app.
- **Unfiled observation - the second tab is unreachable.**
  `.onContentWillChange(() => { return false; })` on the `Tabs` vetoes every
  switch, so only the bar highlight moves. Remove the callback or return
  `true`. The same defect appears in `09_tourism`'s `LocationPermissionPrompt`
  (`HW-09-0016`), which suggests a shared page template.
- **Unfiled observation - the live coordinates may stop updating.** The
  `locationChange` closure mutates the object it captured, while the page
  reads through the `@State` proxy created when that object was assigned; the
  longitude/latitude labels are not guaranteed to re-render on later fixes.
- Distance is rendered with `toFixed(0)` and always suffixed `m`, so a 4.8 km
  neighbour reads as `4800m`.

## Pitfalls

- **`HW-14-0011` (B/medium, confirmed) — the location subscription is never
  released.** `LocationUtil.ets:30-48` calls
  `geoLocationManager.on('locationChange', ...)` and there is no `off`
  anywhere in the project; `handleLocationToggle` only flips UI state, and
  re-enabling calls `OpenContinuosLocation` again, so each cycle stacks
  another callback while GPS keeps running after toggle-off. Fix: store the
  callback and call `geoLocationManager.off`.
- **`HW-14-0012` (B/medium, probable) — a refused settings dialog leaves the
  switch on, and the first list is generated at null island.**
  `PermissionsView.ets:54-57` awaits `requestPermissionOnSetting` with no
  try/catch, so a refusal rejects out of the async `onChange` and
  `isLocationEnabled` keeps the `true` set at the top
  (`NearMainPage.ets:94-119`); separately, `loadNearbyUsers` fires 700 ms
  after the toggle while `currentLocation` is still `{0, 0}`
  (`ProcessData.ets:45-55`), and nothing regenerates when the first real fix
  arrives. Fix: catch the request, and regenerate the list on the first
  `locationChange`.

## References

- `documentation/harmonyos-guides/07_application-services/location-guidelines.md` - `ContinuousLocationRequest`, `UserActivityScenario`, and the on/off pairing
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/location-guidelines
- `documentation/harmonyos-references/06_application-services/js-apis-geolocationmanager.md` - `geoLocationManager.on` / `off` and the `Location` payload
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-geolocationmanager
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - which of the two location permissions is which, and why they are requested together
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` - the user_grant request flow and `dialogShownResults`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` - `checkAccessToken`, `requestPermissionsFromUser`, `requestPermissionOnSetting`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-toggle.md` - `Toggle` and the `isOn` / `onChange` contract
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-toggle
