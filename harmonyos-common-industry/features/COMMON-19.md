---
id: COMMON-19
title: BLE device scan and connect - radar scan UI, filtered BLE discovery, and a GATT client connection
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/19_search_and_connect_ble.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_and_connect_ble-0000002293947117
sample: huawei_industry_tree/19_common_technical_solutions/downloads/低功耗设备蓝牙扫描与连接示例代码.zip
kits: ["@kit.ConnectivityKit", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit", "@kit.ArkUI"]
apis: ["ble.on('BLEDeviceFind')", "ble.off('BLEDeviceFind')", "ble.startBLEScan", "ble.stopBLEScan", "ble.ScanFilter", "ble.ScanOptions", "ble.ScanDuty", "ble.MatchMode", "ble.ScanResult", "ble.createGattClientDevice", "GattClientDevice.connect", "GattClientDevice.disconnect", "GattClientDevice.close", "GattClientDevice.on('BLEConnectionStateChange')", "GattClientDevice.off('BLEConnectionStateChange')", "GattClientDevice.getServices", "GattClientDevice.getRssiValue", "GattClientDevice.readCharacteristicValue", "GattClientDevice.writeCharacteristicValue", "GattClientDevice.readDescriptorValue", "GattClientDevice.writeDescriptorValue", "GattClientDevice.setCharacteristicChangeNotification", "GattClientDevice.on('BLECharacteristicChange')", "access.enableBluetooth", "access.disableBluetooth", "access.getState", "access.on('stateChange')", "abilityAccessCtrl.createAtManager", "AtManager.requestPermissionsFromUser", IDataSource, LazyForEach]
permissions: ["ohos.permission.ACCESS_BLUETOOTH"]
min_api: 20
modules: [products/default, commons/common, features/device, features/scaneffect]
findings: [HW-19-0050, HW-19-0051, HW-19-0052, HW-19-0053, HW-19-0054, HW-19-0055, HW-19-0056, HW-19-0057, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when the application must **find nearby Bluetooth Low Energy
devices and connect to one** - a smart watch, a health monitor, a smart-home
device - showing a radar-style scan animation, a live list of discovered devices
with signal strength, and a per-device detail screen that reads and writes GATT
characteristics.

The document is explicit about the boundary: this is **BLE only**. 部分蓝牙耳机不属于
低功耗设备 ("some Bluetooth headsets are not low-power devices"); classic Bluetooth
scanning and pairing is a different guide (`bluetooth-br`).

**Read the lifecycle findings before reusing.** As shipped the scan is never
stopped (HW-19-0050) and the GATT client is never disconnected or closed
(HW-19-0052) - both are battery costs on precisely the long-running device class
this feature targets.

## Feature checklist

The application must:

- Declare `ohos.permission.ACCESS_BLUETOOTH` and **request it at runtime**, since
  it is a user-grant permission, checking `authResults` before proceeding.
- Turn Bluetooth on if it is off, after checking the current state.
- React to the Bluetooth switch being turned off while the page is open -
  through `access.on('stateChange')`, not a poll (HW-19-0054).
- Subscribe to `BLEDeviceFind`, set scan filters and options, and start the scan.
- Process **every** `ScanResult` in each report, deduplicating by address and
  refreshing RSSI (HW-19-0056).
- Stop the scan and unsubscribe when the user leaves or stops it
  (HW-19-0050, HW-19-0051).
- Create a `GattClientDevice`, subscribe to `BLEConnectionStateChange`, connect,
  and resolve once the state reaches connected.
- Enumerate services, read/write characteristics and descriptors, and subscribe
  to characteristic-change notifications.
- **Disconnect, close and unsubscribe** when done (HW-19-0052).

## Architecture

Four modules - one entry HAP plus three HARs - matching the layered practice of
COMMON-02:

| Module | Path | Contents |
| --- | --- | --- |
| `default` (entry HAP) | `products/default` | `EntryAbility`, `pages/AddDevice`, and the `PageHeader` / `PageBody` / `PageFooter` views; owns the permission request and the scan-state machine |
| `common` (HAR) | `commons/common` | `BleUtil` (scan), `BTUtil` (switch), `DeviceManager` (GATT client), `BufferUtil`, the beans (`ManDeviceBean`, `UUIDBean`), the constants (`BTState`, `BleConnectionState`, `CommonConstants`), the `ScanListener` interface and `BleScanListener`, and `ManDeviceDataSource` |
| `device` (HAR) | `features/device` | `DeviceDetails` view and `BTDetailModel` |
| `scaneffect` (HAR) | `features/scaneffect` | `RadarScanEffectForBLE`, the scan animation |

The `common` HAR exposes its API through `commons/common/Index.ets`; note that
`BTState` is missing from that barrel, which is why `PageBody` deep-imports it
(HW-19-0055).

**Scan flow.**

1. `PageBody` requests `ohos.permission.ACCESS_BLUETOOTH` when the radar is
   tapped, and checks `data.authResults[0]`.
2. On grant it calls `BTUtil.enableBTByState()`, which reads `access.getState()`
   and calls `access.enableBluetooth()` only when the switch is off - the pattern
   the reference recommends ("You are advised to call this API only when the
   Bluetooth switch status is STATE_OFF").
3. `BleUtil.startBLEScan(listener)` releases any previous `BLEDeviceFind`
   subscription, subscribes afresh, builds an empty `ScanFilter` and a
   `ScanOptions` (`interval: 100`, `SCAN_MODE_LOW_POWER`,
   `MATCH_MODE_AGGRESSIVE`), and starts the scan.
4. Reports arrive as `Array<ScanResult>`; `BleScanListener.onApply` turns them
   into `ManDeviceBean`s and pushes them into `ManDeviceDataSource`, which
   deduplicates by address and updates RSSI in place.
5. `ManDeviceDataSource` is an `IDataSource`, so `DeviceDetails` renders the live
   list with `LazyForEach`.

**Connect flow.** `DeviceManager` is a singleton holding one
`ble.GattClientDevice`. `connectDevice` wraps the connection in a promise that
resolves when `BLEConnectionStateChange` reports state 2 (connected) or 0
(disconnected), then either enumerates services and marks the bean bound, or
rejects. Characteristic access (`readCharacteristicValue`,
`writeCharacteristicValue`, `readDescriptorValue`, `writeDescriptorValue`,
`setCharacteristicChangeNotification` + `on('BLECharacteristicChange')`) all go
through the same cached client.

## Implementation steps

1. **Declare and request the permission.** `ohos.permission.ACCESS_BLUETOOTH` in
   `module.json5` with a truthful `usedScene` (HW-19-0053), then
   `atManager.requestPermissionsFromUser(context, ['ohos.permission.ACCESS_BLUETOOTH'])`
   and check `authResults` before touching any Bluetooth API.
2. **Check, then enable.** `access.getState()`; call `access.enableBluetooth()`
   only when the state is `STATE_OFF`.
3. **Track the switch by subscription**, not by polling:
   `access.on('stateChange', cb)` in `aboutToAppear`, `access.off('stateChange')`
   in `aboutToDisappear` (HW-19-0054).
4. **Subscribe before scanning.** `ble.on('BLEDeviceFind', cb)` and set your
   "subscribed" flag right there, not inside the callback (HW-19-0051).
5. **Start the scan** with a filter array and `ScanOptions`, and record that a
   scan is running (HW-19-0050).
6. **Handle the whole result set** and deduplicate by address (HW-19-0056).
7. **Stop cleanly**: `ble.stopBLEScan()` and `ble.off('BLEDeviceFind')` from the
   page's stop action and from `aboutToDisappear`.
8. **Connect**: `ble.createGattClientDevice(address)`,
   `on('BLEConnectionStateChange')`, `connect()`, resolve on state 2.
9. **Release the client**: `off('BLEConnectionStateChange')`, `disconnect()`,
   `close()` (HW-19-0052).

## Verified snippets

All snippets below come from the sample project, not from the document.

### Scan start and stop (as shipped - see HW-19-0050 and HW-19-0051)

`低功耗设备蓝牙扫描与连接示例代码.zip#SearchAndConnectBLE/commons/common/src/main/ets/utils/BleUtil.ets`

```ts
import { ble } from '@kit.ConnectivityKit';

export class BleUtil {
  public static scanDeviceList: ManDeviceDataSource = new ManDeviceDataSource();
  public static isScanning: boolean = false;
  public static isBLEDeviceFind: boolean = false;

  static async startBLEScan(scanListener: ScanListener): Promise<void> {
    try {
      if (BleUtil.isBLEDeviceFind) {
        ble.off('BLEDeviceFind');
        BleUtil.isBLEDeviceFind = false;
      }
      ble.on('BLEDeviceFind', (data) => {
        BleUtil.isBLEDeviceFind = true;   // FIX (HW-19-0051): set outside the callback
        hilog.info(0x0000, 'testTag', `scan result = ${data[0].deviceId}`);
        scanListener.onApply(data);
      });
      // filtering
      let scanFilter: ble.ScanFilter = {};
      // Setting Scan Parameters
      let scanOptions: ble.ScanOptions = {
        interval: 100,
        dutyMode: ble.ScanDuty.SCAN_MODE_LOW_POWER,
        matchMode: ble.MatchMode.MATCH_MODE_AGGRESSIVE
      };
      // Start scanning
      if (BleUtil.isScanning) {
        ble.stopBLEScan();
        BleUtil.isScanning = false;
      }
      ble.startBLEScan([scanFilter], scanOptions);
      // FIX (HW-19-0050): BleUtil.isScanning = true;
    } catch (err) {
      hilog.error(0x0000, 'testTag',
        `errCode: ${(err as BusinessError).code}, errMessage: ${(err as BusinessError).message}`);
    }
  }

  static stopScan(): void {
    try {
      if (BleUtil.isBLEDeviceFind) {
        ble.off('BLEDeviceFind');
        BleUtil.isBLEDeviceFind = false;
      }
      if (BleUtil.isScanning) {      // never true as shipped - HW-19-0050
        ble.stopBLEScan();
        BleUtil.isScanning = false;
      }
    } catch (err) {
      hilog.error(0x0000, 'testTag',
        `errCode: ${(err as BusinessError).code}, errMessage: ${(err as BusinessError).message}`);
    }
  }
}
```

### Bluetooth switch helper

`低功耗设备蓝牙扫描与连接示例代码.zip#SearchAndConnectBLE/commons/common/src/main/ets/utils/BTUtil.ets`

```ts
import { access } from '@kit.ConnectivityKit';

export class BTUtil {
  public static enableBT(): void {
    try {
      access.enableBluetooth();
    } catch (err) {
      hilog.error(0x0000, 'testTag',
        `errCode: ${(err as BusinessError).code}, errMessage: ${(err as BusinessError).message}`);
    }
  }

  public static getBTState(): Number {
    try {
      return access.getState();
    } catch (err) {
      hilog.error(0x0000, 'testTag',
        `errCode: ${(err as BusinessError).code}, errMessage: ${(err as BusinessError).message}`);
      return BTState.STATE_EXCEPTION;
    }
  }

  public static enableBTByState(): void {
    let state = BTUtil.getBTState();
    if (BTState.STATE_ON === state) {
      return;
    }
    BTUtil.enableBT();
  }
}
```

### Runtime permission request

`低功耗设备蓝牙扫描与连接示例代码.zip#SearchAndConnectBLE/products/default/src/main/ets/views/PageBody.ets`

```ts
private permissions: Array<Permissions> = ['ohos.permission.ACCESS_BLUETOOTH'];

let atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
atManager.requestPermissionsFromUser(this.context, this.permissions)
  .then((data: PermissionRequestResult) => {
    let grantStatus: Array<number> = data.authResults;
    let length: number = grantStatus.length;
    if (length <= 0) {
      return;
    }
    if (grantStatus[0] === Constants.AUTHORIZED) {
      BTUtil.enableBTByState();
      if (BTUtil.getBTState() === BTState.STATE_ON) {
        // ... start the scan animation ...
      }
    } else {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.get_permissions') });
      return;
    }
  })
  .catch((err: BusinessError) => {
    hilog.error(0x0000, 'testTag',
      `Failed to request permissions from user. Code is ${err.code}, message is ${err.message}`);
  });
```

### GATT connect (as shipped - see HW-19-0052)

`低功耗设备蓝牙扫描与连接示例代码.zip#SearchAndConnectBLE/commons/common/src/main/ets/DeviceManager.ets`

```ts
export class DeviceManager {
  private static instance = new DeviceManager();
  private clientDevice?: ble.GattClientDevice;
  private connectManDeviceBean: ManDeviceBean = new ManDeviceBean();

  public static getInstance(): DeviceManager {
    return DeviceManager.instance;
  }

  async connectDevice(manDeviceBean: ManDeviceBean): Promise<number> {
    let address = manDeviceBean.address;
    let state: number = await new Promise((resolve: Function) => {
      this.clientDevice = ble.createGattClientDevice(address);
      // Subscribe to connection status change events
      this.clientDevice.on('BLEConnectionStateChange', (bleConnectionState) => {
        //bleConnectionState.state: 0 DISCONNECTED, 1 CONNECTING, 2 STATE_CONNECTED, 3 STATE_DISCONNECTING
        let state = bleConnectionState.state;
        manDeviceBean.connectState = bleConnectionState.state;
        if (2 === state || 0 === state) {
          resolve(state);
        }
      });
      // Connecting to the GattServer Service
      this.clientDevice.connect();
    });
    return new Promise((resolve, reject) => {
      this.connectManDeviceBean = manDeviceBean;
      if (2 === state) {
        this.getServices();
        manDeviceBean.bind = true;
        resolve(state);
      } else {
        manDeviceBean.bind = false;
        reject(state);
      }
    });
  }
}
```

Missing counterpart to add (HW-19-0052):

```ts
disconnectDevice(): void {
  try {
    this.clientDevice?.off('BLEConnectionStateChange');
    this.clientDevice?.disconnect();
    this.clientDevice?.close();
  } catch (err) {
    hilog.error(0x0000, 'testTag',
      `errCode: ${(err as BusinessError).code}, errMessage: ${(err as BusinessError).message}`);
  } finally {
    this.clientDevice = undefined;
  }
}
```

### Characteristic notification subscription

`低功耗设备蓝牙扫描与连接示例代码.zip#SearchAndConnectBLE/commons/common/src/main/ets/DeviceManager.ets`

```ts
return new Promise((resolve, reject) => {
  this.clientDevice?.setCharacteristicChangeNotification(bleCharacteristicDataIn, true, callBack => {
    if (callBack === null) {
      this.clientDevice?.on('BLECharacteristicChange', (characteristicChangeReq: ble.BLECharacteristic) => {
        let value: Uint8Array = new Uint8Array(characteristicChangeReq.characteristicValue);
        let hexString = BufferUtil.uint8ArrayToHexString(value);
        hilog.info(0x0000, 'testTag', `hesString = ${hexString}`);
        resolve(JSON.stringify(value));
      });
    }
  });
});
```

### The scan-result data source

`低功耗设备蓝牙扫描与连接示例代码.zip#SearchAndConnectBLE/commons/common/src/main/ets/adapter/ManDeviceDataSource.ets`

```ts
@Observed
export class ManDeviceDataSource implements IDataSource {
  private listeners: DataChangeListener[] = [];
  private originDataArray: ManDeviceBean[] = [];

  public totalCount(): number {
    return this.originDataArray.length;
  }

  public getData(index: number): ManDeviceBean {
    return this.originDataArray[index];
  }

  public pushData(data: ManDeviceBean): void {
    let address = data.address;
    for (let manDeviceBean of this.originDataArray) {
      if (address === manDeviceBean.address) {
        manDeviceBean.rssi = data.rssi;
        return;
      }
    }
    this.originDataArray.push(data);
    this.notifyDataAdd(this.originDataArray.length - 1);
  }
}
```

### Result mapping (as shipped - see HW-19-0056)

`低功耗设备蓝牙扫描与连接示例代码.zip#SearchAndConnectBLE/commons/common/src/main/ets/listener/BleScanListener.ets`

```ts
export class BleScanListener implements ScanListener {
  public scanDeviceList: ManDeviceDataSource = new ManDeviceDataSource();

  onApply(data: Array<ble.ScanResult>): void {
    let manDeviceBean = new ManDeviceBean();
    manDeviceBean.id = data[0].deviceId;        // FIX: iterate the whole array
    manDeviceBean.name = data[0].deviceName;
    manDeviceBean.rssi = data[0].rssi;
    manDeviceBean.address = data[0].deviceId;

    let name = manDeviceBean.name;
    if (name !== undefined && name !== null && name.length !== 0) {
      this.scanDeviceList.pushData(manDeviceBean);
    }
  }
}
```

## Permissions & config

`ohos.permission.ACCESS_BLUETOOTH` is the only permission, and the document lists
exactly that one. It is a **user-grant** permission - the document links the
user-grant permission page - so declaring it is not enough; the sample correctly
requests it at runtime and inspects `authResults`.

`低功耗设备蓝牙扫描与连接示例代码.zip#SearchAndConnectBLE/products/default/src/main/module.json5`:

```json5
{
  "module": {
    "name": "default",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone"],
    "deliveryWithInstall": true,
    "installationFree": false,
    "pages": "$profile:main_pages",
    "abilities": [
      {
        "name": "EntryAbility",
        "srcEntry": "./ets/entryability/EntryAbility.ets",
        "icon": "$media:layered_image",
        "label": "$string:EntryAbility_label",
        "startWindowIcon": "$media:startIcon",
        "startWindowBackground": "$color:start_window_background",
        "exported": true,
        "skills": [
          { "entities": ["entity.system.home"], "actions": ["action.system.home"] }
        ]
      }
    ],
    "extensionAbilities": [
      {
        "name": "EntryBackupAbility",
        "srcEntry": "./ets/entrybackupability/EntryBackupAbility.ets",
        "type": "backup",
        "exported": false,
        "metadata": [
          { "name": "ohos.extension.backup", "resource": "$profile:backup_config" }
        ]
      }
    ],
    "requestPermissions": [
      {
        "name": "ohos.permission.ACCESS_BLUETOOTH",
        "reason": "$string:access_ble_reason",
        "usedScene": {
          "abilities": ["DefaultAbility"],   // FIX (HW-19-0053): "EntryAbility"
          "when": "inuse"
        }
      }
    ]
  }
}
```

Root `build-profile.json5` registers the four modules: `default`
(`./products/default`), `common` (`./commons/common`), `device`
(`./features/device`) and `scaneffect` (`./features/scaneffect`).

Note that this module declares **`deviceTypes: ["phone"]` only** - unlike most
samples in this industry.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **BLE only.** The document states that some Bluetooth headsets are not
  low-power devices and points at the classic-Bluetooth guide for those:
  "本示例适用于低功耗设备的蓝牙扫描与连接，部分蓝牙耳机不属于低功耗设备。"
- **`ohos.permission.ACCESS_BLUETOOTH` is required by essentially every API used
  here** - `ble.on('BLEDeviceFind')`, `startBLEScan`, `connect`, `disconnect`,
  `close`, `access.enableBluetooth`, `access.getState` all list it as a required
  permission, and all of them can fail with error 201 Permission denied.
- **Enable Bluetooth only when it is off**: "You are advised to call this API
  only when the Bluetooth switch status is STATE_OFF."
- **Bluetooth error codes to expect**: 201 permission denied, 801 capability not
  supported, 2900001 service stopped, 2900003 Bluetooth disabled, 2900099
  operation failed.
- **`off('BLEConnectionStateChange')` with no callback argument** unsubscribes
  every callback for that event; pass the exact callback to remove just one.
- **A closed `GattClientDevice` is unusable**: "The instance created by calling
  GattClientDevice is unavailable after being closed" - create a new one to
  reconnect.
- **Device type.** `phone` only, per this sample's `module.json5`.

## Pitfalls

- **`isScanning` is never set to `true`, which is incorrect.** The guard makes
  `ble.stopBLEScan()` unreachable, so the scan runs until the process ends.
  (HW-19-0050)
- **Setting `isBLEDeviceFind` inside the callback is incorrect.** A scan that
  finds nothing leaves the `BLEDeviceFind` listener registered, and every
  subsequent scan stacks another one. Set the flag where `ble.on` is called.
  (HW-19-0051)
- **The GATT client is never released, which is incorrect.** No `off`, no
  `disconnect()`, no `close()` exists in the project; each connect leaks a client
  and its listeners and leaves the radio link up. (HW-19-0052)
- **`usedScene.abilities: ["DefaultAbility"]` is incorrect** - the module's only
  ability is `EntryAbility`. (HW-19-0053)
- **Polling `access.getState()` every 100 ms is incorrect.** The access module
  documents `access.on('stateChange')` for exactly this. (HW-19-0054)
- **`import { BTState } from '@ohos/common/src/main/ets/constants/BTState'`
  bypasses the HAR barrel**, because `BTState` is not exported from
  `commons/common/Index.ets`. Export it and import through `@ohos/common`.
  (HW-19-0055)
- **Reading only `data[0]` of the scan report is incorrect.** The callback
  delivers "the set of scan results"; iterate it, and guard the index against an
  empty array. (HW-19-0056)
- **`connectDevice` resolves the inner promise on state 0 as well as state 2.**
  That is deliberate - it is how a failed connection unblocks the await - but it
  means the outer promise rejects with the *state number*, not a `BusinessError`;
  callers must not treat the rejection value as an error object.
- **`writeCharacteristicValue` and `writeDescriptorValue` return promises that
  never settle, which is incorrect.** Both wrap the call in
  `new Promise(() => { ... })` with no `resolve`, so awaiting them hangs forever
  and a write error is never delivered to the caller. (HW-19-0057)

## References

- `documentation/harmonyos-references/03_system/js-apis-bluetooth-ble.md` -
  `ble.on('BLEDeviceFind')` / `off`, `startBLEScan` / `stopBLEScan`,
  `ScanFilter`, `ScanOptions`, `ScanResult`, `createGattClientDevice`, and the
  `GattClientDevice` methods `connect`, `disconnect` ("Disconnects the GATT
  connection from the server"), `close` ("Closes a client instance. The instance
  ... is unavailable after being closed"), `on`/`off('BLEConnectionStateChange')`,
  plus the Bluetooth error codes.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-bluetooth-ble
- `documentation/harmonyos-references/03_system/js-apis-bluetooth-access.md` -
  `enableBluetooth`, `disableBluetooth`, `getState`, and the notes recommending
  `access.on('stateChange')` and calling enable only in `STATE_OFF`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-bluetooth-access
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` -
  requesting a user-grant permission and checking `authResults`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `documentation/harmonyos-guides/01_getting-started/har-package.md` - HAR
  packaging and the `Index.ets` entry point.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/har-package
- https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/bluetooth-br -
  classic Bluetooth, for devices that are not BLE.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/radar_scan_effect-0000002236447458 -
  the radar scan animation this sample reuses.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_and_connect_ble-0000002293947117
