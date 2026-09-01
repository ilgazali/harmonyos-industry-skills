---
id: SPORT-01
title: Sports and health app - layered HARs with BLE device pairing and a pedometer
industry: 03_sports_health
doc: huawei_industry_tree/03_sports_health/docs/01_practice-sports-health-app-architecture-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-sports-health-app-architecture-v1-0000001952522073
sample: huawei_industry_tree/03_sports_health/downloads/SportsHealth_Framework_Code_V1.zip
kits: ["@kit.ConnectivityKit", "@kit.SensorServiceKit", "@kit.AbilityKit", "@kit.NotificationKit", "@kit.ArkData", "@kit.ArkTS", "@kit.CryptoArchitectureKit", "@kit.BasicServicesKit"]
apis: ["ble.startBLEScan", "ble.stopBLEScan", "ble.on('BLEDeviceFind')", "ble.off('BLEDeviceFind')", "ble.createGattClientDevice", "GattClientDevice.connect", "GattClientDevice.disconnect", "GattClientDevice.close", "GattClientDevice.on('BLEConnectionStateChange')", "ble.ScanOptions", "ble.ScanDuty", "ble.MatchMode", "constant.ProfileConnectionState", "sensor.on(SensorId.PEDOMETER)", "sensor.off", "sensor.PedometerResponse", "abilityAccessCtrl.requestPermissionsFromUser", "notificationManager.requestEnableNotification", NavPathStack, "@StorageLink", "@Consume", "preferences"]
permissions: ["ohos.permission.ACCESS_BLUETOOTH", "ohos.permission.ACTIVITY_MOTION"]
min_api: 20
modules: [phone, smartscreen, common, first, run, find, shop, mine, weightscale]
findings: [HW-03-0001, HW-03-0002, HW-03-0003, HW-03-0004, HW-03-0005, HW-03-0006, HW-03-0007, HW-03-0008, HW-03-0009, HW-03-0010, HW-03-0011, HW-03-0012, HW-03-0013, HW-03-0055]
status: verified-with-fixes
---

## When to use

Load this card when an app **pairs with a wearable or a body-metrics device
over BLE and counts the user's steps** - a fitness tracker companion, a smart
scale app, a skipping-rope or heart-rate accessory, a health dashboard.

Two capabilities carry the architecture and neither appears elsewhere in this
corpus:

- **BLE device discovery and connection** through Connectivity Kit -
  `startBLEScan`, `BLEDeviceFind`, `createGattClientDevice`,
  `BLEConnectionStateChange`.
- **The pedometer sensor** through Sensor Service Kit -
  `sensor.on(SensorId.PEDOMETER)`.

The **one-HAR-per-IoT-device** module rule is also worth taking: each physical
accessory gets its own HAR wrapping both its UI and its Bluetooth handling, so
a new device is a new module rather than a branch in shared code.

**Read the Pitfalls before copying either capability.** Thirteen findings, and
the two subscription paths - the pedometer and the BLE scan - are both broken
in ways that run down the battery or multiply without bound.

## Feature checklist

- Five bottom tabs: 首页 (home), 运动 (exercise), 发现 (discover),
  商城 (shop), 我的 (mine).
- Home: avatar, guide, device management, body metrics, weekly report,
  exercise plan, habits.
- Device management: scan for BLE devices, list what is found, connect,
  disconnect, keep a list of paired devices.
- Exercise: plans, courses, a running page with a live step count, a
  leaderboard.
- Shop: banner carousel, categories, recommendations.
- Mine: profile, health report, heart monitoring, messages, settings, privacy
  policy and data disclosure pages.
- Account flow: launcher, login, SMS-code login, register, change password,
  change phone, cancel account.

## Architecture

Three layers, nine modules, with an extra tier for accessories.

```
products/
├── phone/src/main/ets           the only HAP (type: entry)
│   ├── entryability/EntryAbility.ets
│   ├── pages/                   13 pages: Launcher, Login, MainPage, Register, Verify,
│   │                            ChangePassword, ChangeIphone, RetrievePassword,
│   │                            CancelAccount, SureCancel, Secret, Privacy, Phone
│   ├── common/utils/{GlobalContext,PreferencesUtil}.ets
│   └── utils/{Logger,PermissionsUtil}.ets
└── smartscreen/                 declared type: har, and empty (HW-03-0010)

features/                        one HAR per tab
├── first/    HomePageDemoOne.ets, CustomDialogComponent.ets
├── run/      ListDemoOne.ets            ← the pedometer
├── find/     DiscoveryPage.ets
├── shop/     SwiperDemo01.ets
└── mine/     8 pages, 4 views, 3 models

IoT/                             one HAR per accessory
└── weightscale/  DeviceListPageOne.ets

common/                          shared capability
├── common/components/CommonConstants.ets
├── common/utils/{GlobalContext,Utils}.ets
├── model/{NavItemModel,TaskInitList}.ets
└── page/{AddDeviceDemoOne,MyDeviceOne}.ets   ← the BLE scan and pairing
```

The documented tree spells `components` as `compontents`, renames
`LoginPage.ets` to `LoginPage2.ets` and omits `PrivacyPage.ets`
(`HW-03-0009`).

**The IoT tier is the idea worth copying.** The document states the rule -
每个智能设备打包一个HAR包，封装设备的UI及蓝牙连接等内聚的功能 (package each
smart device as one HAR, encapsulating its UI and Bluetooth connection) - so
adding a skipping rope alongside the weight scale means adding
`IoT/skippingrope/`, not editing a shared device file.

Note where the BLE code actually lives, though: the scan and pairing sit in
`common/page/AddDeviceDemoOne.ets`, not in the IoT tier, so the generic
discovery is shared and only the device-specific screens are per-HAR.

**The intended BLE sequence**, which the document draws as a timing diagram:

```
startBLEScan ──> on('BLEDeviceFind') ──> user picks a device
                                              │
                              createGattClientDevice(deviceId)
                                              │
                        connect() ──> on('BLEConnectionStateChange')
                                              │
                                    STATE_CONNECTED ──> record the device
                                              │
                        off(...) ──> disconnect() ──> close()
```

The sample implements everything down to the last row and gets the teardown
wrong (`HW-03-0003`).

## Implementation steps

1. **Declare `ACCESS_BLUETOOTH` and `ACTIVITY_MOTION`** with per-permission
   reasons and `when: "inuse"` (`HW-03-0007`), and put `client_id` in the
   module's own metadata (`HW-03-0006`).
2. **Request the permissions before the features need them**, sequenced, and
   read the results (`HW-03-0012`).
3. **Start the scan from `aboutToAppear` and stop it in `aboutToDisappear`** -
   not `onPageHide`, which a non-`@Entry` component never receives
   (`HW-03-0004`).
4. **Filter discoveries** on a name, connectability and an id you have not
   seen before, so the list does not thrash.
5. **Keep the `GattClientDevice` in a field**, subscribe on that instance, and
   unsubscribe, disconnect and `close()` that same instance (`HW-03-0003`).
6. **Subscribe to the pedometer exactly once** and release it on teardown
   (`HW-03-0001`).
7. **Log every `BusinessError`** - both APIs report permission denial as an
   exception (`HW-03-0005`).

## Verified snippets

All snippets are from `SportsHealth_Framework_Code_V1.zip`. Corrected forms
are marked.

**Scanning for devices — `common/src/main/ets/page/AddDeviceDemoOne.ets`** (corrected, see `HW-03-0004`, `HW-03-0005`)

```typescript
import { ble, constant } from '@kit.ConnectivityKit';

aboutToAppear(): void {
  this.startScan();
  this.onBLEDeviceFind();
}

aboutToDisappear(): void {          // FIX: the sample puts this in onPageHide, which never fires
  this.stopScan();
  this.offBLEDeviceFind();
}

startScan() {
  try {
    const scanFilter: ble.ScanFilter = {};                 // empty: accept everything
    const scanOptions: ble.ScanOptions = {
      interval: 1000,
      dutyMode: ble.ScanDuty.SCAN_MODE_LOW_LATENCY,        // fastest discovery, highest drain
      matchMode: ble.MatchMode.MATCH_MODE_STICKY           // only report strong, stable devices
    };
    ble.startBLEScan([scanFilter], scanOptions);
  } catch (err) {
    Logger.error(`startBLEScan failed: ${(err as BusinessError).code}`);   // FIX: sample's catch is empty
  }
}
```

**`SCAN_MODE_LOW_LATENCY` with `MATCH_MODE_STICKY` is a deliberate pair.** Low
latency finds an accessory the user has just switched on within a second;
sticky matching suppresses the weak, transient advertisements that would
otherwise fill the list in a crowded room. It is the right configuration for a
short, user-initiated pairing scan - and the wrong one to leave running, which
is exactly what `HW-03-0004` causes.

**Filtering discoveries — same file** (as shipped)

```typescript
onBLEDeviceFind() {
  ble.on('BLEDeviceFind', (data: Array<ble.ScanResult>) => {
    if (data[0].deviceName !== '' && data[0].connectable &&
        this.deviceIds.getIndexOf(data[0].deviceId) < 0) {
      this.deviceIds.add(data[0].deviceId);
      // ... push into the displayed list, capped at 5
    }
  });
}
```

**Three conditions, each earning its place.** A blank `deviceName` is an
advertisement with nothing to show the user; `connectable` false is a beacon
that cannot be paired; and the `deviceId` set is what stops the same device
being appended on every advertising interval - BLE devices advertise several
times a second, so without it the list grows continuously.

**Connecting and releasing — same file** (corrected, see `HW-03-0003`)

```typescript
private gattClient?: ble.GattClientDevice;

getConnect(device: ble.ScanResult) {
  this.gattClient = ble.createGattClientDevice(device.deviceId);
  this.gattClient.on('BLEConnectionStateChange', (state: ble.BLEConnectionChangeState) => {
    this.connectState = state.state;
    if (state.state === constant.ProfileConnectionState.STATE_CONNECTED) {
      this.connectedDeviceMap.set(device.deviceName, device);
      if (this.deviceList.indexOf(device.deviceName) < 0) {
        this.deviceList.push(device.deviceName);
      }
    }
  });
  this.gattClient.connect();          // subscribe before connecting, or the first transition is missed
}

release() {
  // FIX: the sample's off path calls createGattClientDevice again and unsubscribes from the new one
  if (this.gattClient) {
    this.gattClient.off('BLEConnectionStateChange');
    this.gattClient.disconnect();
    this.gattClient.close();          // FIX: the sample never closes the client
    this.gattClient = undefined;
  }
}
```

**Subscribe before `connect()`.** The sample calls `connect()` and then
registers the listener, so the `STATE_CONNECTED` transition can arrive before
anything is listening for it - a fast local device is exactly when that
happens.

`close()` is not optional: the BLE reference states that an instance created
by `createGattClientDevice` "is unavailable after being closed", which is the
signal that it holds a native resource until then.

**The pedometer — `features/run/src/main/ets/pages/ListDemoOne.ets`** (corrected, see `HW-03-0001`)

```typescript
import { sensor } from '@kit.SensorServiceKit';

private sensorCallback: (data: sensor.PedometerResponse) => void =
  (data: sensor.PedometerResponse) => {
    this.stepNum = data.steps ? data.steps : 0;
    // FIX: the sample's callback ends by calling onPedometer(), which registers ANOTHER sensor.on
  };

aboutToAppear(): void {
  try {
    sensor.on(sensor.SensorId.PEDOMETER, this.sensorCallback, { interval: 100000000 });
  } catch (err) {
    Logger.error(`sensor.on failed: ${(err as BusinessError).code}`);   // 201 = permission denied
  }
}

aboutToDisappear(): void {                    // FIX: absent in the sample
  sensor.off(sensor.SensorId.PEDOMETER, this.sensorCallback);
}
```

**The permission is `ohos.permission.ACTIVITY_MOTION`,** not `ACCELEROMETER` -
the sensor reference states it under `PEDOMETER on()`, and the document's
implementation section says the wrong one (`HW-03-0002`).

`interval` is in **nanoseconds** and defaults to 200,000,000 (200 ms). The
sample's first subscription asks for 10,000,000 - 10 ms - for a step count
that changes at most twice a second, which is what makes its runaway
subscription compound so fast.

**One HAR per accessory — `build-profile.json5`** (as shipped)

```json5
"modules": [
  { "name": "phone",       "srcPath": "./products/phone", "targets": [ /* ... */ ] },
  { "name": "smartscreen", "srcPath": "./products/smartscreen" },
  { "name": "common",      "srcPath": "./common" },
  { "name": "first",       "srcPath": "./features/first" },
  { "name": "run",         "srcPath": "./features/run" },
  { "name": "find",        "srcPath": "./features/find" },
  { "name": "shop",        "srcPath": "./features/shop" },
  { "name": "mine",        "srcPath": "./features/mine" },
  { "name": "weightscale", "srcPath": "./IoT/weightscale" }     // one per accessory
]
```

with the HAP depending on each by alias:

```json5
// products/phone/oh-package.json5
"dependencies": {
  "@ohos/common": "file:../../common",
  "@ohos/mine": "file:../../features/mine",
  "@ohos/first": "file:../../features/first",
  "@ohos/run": "file:../../features/run",
  "@ohos/weightscale": "file:../../IoT/weightscale",
  "@ohos/find": "file:../../features/find",
  "@ohos/shop": "file:../../features/shop"
}
```

## Permissions & config

Entry module (`products/phone/src/main/module.json5`), corrected:

```json5
"metadata": [
  { "name": "client_id", "value": "<your AGC Client ID>" }   // FIX: sample nests this in the ability
],                                                            //      with a real-looking value
"requestPermissions": [
  {
    "name": "ohos.permission.ACCESS_BLUETOOTH",
    "reason": "$string:bluetooth_permission_reason",   // FIX: sample uses EntryAbility_desc for all three
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }   // FIX: sample says always
  },
  {
    "name": "ohos.permission.ACTIVITY_MOTION",
    "reason": "$string:motion_permission_reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
  // FIX: delete ohos.permission.KEEP_BACKGROUND_RUNNING - nothing claims a continuous task
]
```

Both are `user_grant`, so `reason` and `usedScene` are mandatory.

Resource directories: `base`, `en_US`, `zh_CN`, `rawfile`. `zh_CN` holds 37
strings and `en_US` three; almost nothing on screen resolves through either
(`HW-03-0013`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` and
  `targetSdkVersion` are both `6.0.0(20)`.
- **Hardware required.** BLE cannot be exercised on the emulator, and the
  pedometer needs a device with the sensor - the sensor reference notes that
  "the step counter sensor's data reporting is subject to some delay, and the
  delay is determined by specific product implementations."
- **The smart-screen half does not exist.** The document plans two Entry HAPs;
  the sample ships one, plus an empty HAR (`HW-03-0010`). The 应用接续
  continuation scenario it describes has no second endpoint.
- **The login is UI only.** The document states it -
  任意用户名和密码均可登录 (any username and password will log in) - and the
  code confirms it: `LoginPage.ets` enables the button when both fields are
  non-empty and navigates without any check.
- The weight-scale HAR contains one page and no Bluetooth protocol work, so
  the per-accessory tier is scaffolding rather than a worked example.
- The scan list is capped at five devices and reversed on each insert, so the
  order is newest-first by construction.

## Pitfalls

- **`HW-03-0001` — the pedometer callback opens another pedometer
  subscription on every reading.** Subscriptions multiply for as long as the
  user walks, the first at a 10 ms interval, and nothing is ever released.
  This is the single most damaging line in the sample.
- **`HW-03-0002` — the step-counting section names `ACCELEROMETER` and says
  the accelerometer is subscribed,** contradicting the same document's own
  solution section and the sensor reference, which require
  `ACTIVITY_MOTION` and `SensorId.PEDOMETER`.
- **`HW-03-0003` — the BLE unsubscribe creates a fresh `GattClientDevice`**
  and calls `off` on that, leaving the real connection's listener registered
  and overwriting the live handle. `close()` is never called.
- **`HW-03-0004` — the scan is stopped only from `onPageHide`** on a component
  that is not `@Entry`, so a low-latency scan starts and never stops.
- **`HW-03-0005` — five empty catch blocks** cover every Bluetooth and sensor
  failure, including the permission-denied case both APIs report as an
  exception.
- **`HW-03-0006` — `client_id` is nested inside the ability** instead of the
  module, and carries `111264125` rather than a placeholder.
- **`HW-03-0007` — all three permissions share one reason string,** are scoped
  `when: always`, and one of them (`KEEP_BACKGROUND_RUNNING`) has no consumer.
- **`HW-03-0008` — the Bluetooth permission note points at Sensor Service
  Kit,** two sections after the document correctly attributes the BLE work to
  Connectivity Kit.
- **`HW-03-0009` — the documented tree has `compontents`, `LoginPage2.ets` and
  no `PrivacyPage.ets`.**
- **`HW-03-0010` — two Entry HAPs are described, one exists;** the
  smart-screen module is an empty HAR.
- **`HW-03-0011` — `PermissionsUtil.checkPermissions` overwrites its verdict
  per loop pass,** the same defect as in two other industry templates in this
  corpus.
- **`HW-03-0012` — the permission request and the notification request fire
  together in `aboutToAppear`,** stacking two system dialogs, and neither
  result is read.
- **`HW-03-0013` — 89 inline Chinese literals against 9 resourced strings,**
  with 34 unused entries sitting in `zh_CN` and an empty `en_US`.

## References

- `documentation/harmonyos-guides/04_system/ble-development-guide.md` - scanning, `BLEDeviceFind`, and the `off` pattern
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ble-development-guide
- `documentation/harmonyos-references/03_system/js-apis-bluetooth-ble.md` - `ScanOptions`, `GattClientDevice`, `close()`, the error codes
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-bluetooth-ble
- `documentation/harmonyos-references/03_system/js-apis-bluetooth-constant.md` - `ProfileConnectionState`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-bluetooth-constant
- `documentation/harmonyos-references/03_system/js-apis-sensor.md` - `SensorId.PEDOMETER`, its required permission, `Options.interval`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-sensor
- `documentation/harmonyos-guides/04_system/motion-guidelines.md` - declaring `ohos.permission.ACTIVITY_MOTION`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/motion-guidelines
- `documentation/harmonyos-references/02_application-framework/ts-custom-component-lifecycle.md` - why `onPageHide` needs `@Entry`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-custom-component-lifecycle
- `documentation/harmonyos-guides/04_system/configuration_client_id.md` - where `client_id` belongs
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/configuration_client_id
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `reason`, `usedScene`, and requesting the minimum
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- `documentation/harmonyos-guides/07_application-services/notification-enable.md` - the notification authorization flow and code 1600004
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/notification-enable
