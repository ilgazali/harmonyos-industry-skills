---
id: COMMON-48
title: Wi-Fi scan and connect - listing networks and joining one as a candidate network
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/48_wifi_connect.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/wifi_connect-0000002518583904
sample: huawei_industry_tree/19_common_technical_solutions/downloads/WiFIConnect.zip
kits: ["@kit.ConnectivityKit", "@kit.ArkUI", "@kit.ArkTS"]
apis: ["wifiManager.startScan", "wifiManager.getScanInfoList", "wifiManager.isWifiActive", "wifiManager.getSignalLevel", "wifiManager.addCandidateConfig", "wifiManager.connectToCandidateConfigWithUserAction", "wifiManager.on('wifiScanStateChange')", "wifiManager.off('wifiScanStateChange')", WifiScanInfo, WifiDeviceConfig, WifiSecurityType, "collections.Set", bindSheet, TextInput, "InputType.Password", "promptAction.openToast"]
permissions: ["ohos.permission.GET_WIFI_INFO", "ohos.permission.SET_WIFI_INFO"]
min_api: 21
modules: [entry]
findings: [HW-19-0159, HW-19-0160, HW-19-0161, HW-19-0162, HW-19-0182, HW-19-0183]
status: verified-with-fixes
---

## When to use

Load this card when the application must show nearby Wi-Fi networks and let the
user join one from inside the app - a device-provisioning flow, a captive-network
onboarding, a kiosk setup screen.

The mechanism is the **candidate network**: an ordinary application cannot join a
network directly, but it can register a configuration as a candidate and ask the
system to connect to it, at which point the *system* asks the user to confirm.
That confirmation is the security boundary, and it is also the outcome the code
has to handle - the user can decline (HW-19-0160).

Note the version floor: **API 21 / HarmonyOS 6.0.1**, higher than the rest of this
industry, because `connectToCandidateConfigWithUserAction` is API 20+ and the
sample's `build-profile.json5` pins `6.0.1(21)`.

## Feature checklist

- `isWifiActive()` first - the application cannot turn Wi-Fi on.
- `startScan()`, then read the list **when the scan-complete event arrives**
  (HW-19-0161).
- `getScanInfoList()` → de-duplicate by SSID, drop empty SSIDs.
- `getSignalLevel(rssi, band)` for the bars.
- `addCandidateConfig(config)` → `connectToCandidateConfigWithUserAction(networkId)`,
  both guarded (HW-19-0160).
- Register the scan listener once, and unregister it with the **same reference**
  (HW-19-0159).

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `pages/Index.ets` | the WLAN switch, the network list |
| `components/WiFiInfo.ets` | one row, plus the password sheet and connect button |
| `utils/WiFiUtil.ets` | scan, de-duplicate, map to a view model, connect |
| `models/WiFiInfoModel.ets` | icon, name, security label |

**The candidate-network flow is two calls, and the second one is a prompt.**
`addCandidateConfig` registers the SSID/password/security triple and returns a
`networkId`; `connectToCandidateConfigWithUserAction(networkId)` asks the system
to connect and, per the reference, "the system prompts the user to confirm
whether to trust and connect to the specified candidate network" and "the
connection operation is not performed before the user confirms the trust". So the
promise can reject with `2501006` (refused) or `2501005` (no response) as a
matter of course.

There is a sibling API, `connectToCandidateConfig`, which the reference notes
returns "no user response" - this sample deliberately chooses the one that asks.

**De-duplication uses `collections.Set`, not a plain `Set`.** `@kit.ArkTS`'s
`collections` provides the ArkTS-native containers; a scan typically returns one
entry per band and per AP, so the same SSID appears several times, and the sample
keeps the first occurrence of each. It also drops `ssid === ''`, which is how
hidden networks appear.

**The signal icon is chosen by string composition:**

```ts
let secType: string = securityType === `加密` ? `sec` : `open`;
let signalLevel: number = wifiManager.getSignalLevel(scanInfo.rssi, scanInfo.band);
let src: string = `app.media.wifi_${secType}_${signalLevel}`;
let wifiLabel: Resource = $r(src);
```

A compact idiom - `wifi_sec_0` … `wifi_open_4` must all exist as media resources,
and `$r()` with a computed string is resolved at runtime, so a missing
combination fails silently rather than at build time.

**The password sheet is a `bindSheet` with `$$` two-way binding.**
`bindSheet($$this.isShowSheet, this.SheetBuilder(), {...})` - the `$$` is what
lets the sheet's own dismissal write `false` back to the state.

**The scan listener is registered but not used.** `on('wifiScanStateChange')` is
the mechanism the reference prescribes for knowing when fresh results exist; its
callback here logs and nothing else, while the list is read synchronously right
after `startScan` (HW-19-0161).

## Implementation steps

1. **Check Wi-Fi is on.** `wifiManager.isWifiActive()`; if not, tell the user to
   enable it - an application cannot.
2. **Register the scan-state listener once**, keeping the callback in a variable
   so it can be removed later (HW-19-0159).
3. **Start a scan**: `wifiManager.startScan()` inside `try/catch`.
4. **Read the list when the scan completes** - from the `wifiScanStateChange`
   callback, not immediately after `startScan` (HW-19-0161).
5. **De-duplicate by SSID** and drop empty ones.
6. **Map to a view model**: security label from `WifiSecurityType`, bars from
   `getSignalLevel(rssi, band)`.
7. **On tap**, open a sheet with an `InputType.Password` field.
8. **Connect**: `addCandidateConfig(config)` then
   `connectToCandidateConfigWithUserAction(networkId)`, both in `try/catch`, and
   branch the UI on the result including refusal (HW-19-0160).
9. **Unregister** the listener with the same callback reference, on a path that
   runs however the function exits.

## Verified snippets

All snippets below come from the sample project, not from the document.

### Scanning and de-duplicating

`WiFIConnect.zip#WiFIConnect/entry/src/main/ets/utils/WiFiUtil.ets`

```ts
import { wifiManager } from '@kit.ConnectivityKit';
import { collections } from '@kit.ArkTS';
import { promptAction } from '@kit.ArkUI';
import { WiFiInfoModel } from '../models/WiFiInfoModel';

class WiFiUtil {
  getScanResult(): wifiManager.WifiScanInfo[] {
    try {
      // 获取扫描状态 0：扫描失败；1：扫描成功。
      wifiManager.on("wifiScanStateChange", (result: number) => {
        console.info(`Receive Wifi scan state change event: ${result}`);
      });                                    // FIX (HW-19-0161): use this event to refresh

      let isWifiActive = wifiManager.isWifiActive();
      if (!isWifiActive) {
        promptAction.openToast({
          message: '请先手动打开WiFi'
        });
        console.error(`wifi not enable`);
        return [];                           // FIX (HW-19-0159): returns before off()
      }

      let scanInfoList: wifiManager.WifiScanInfo[] = wifiManager.getScanInfoList();
      let scanInfoListSet: wifiManager.WifiScanInfo[] = [];
      let wifiSet = new collections.Set<string>();
      for (let wifi of scanInfoList) {
        if (wifiSet.has(wifi.ssid) === false && wifi.ssid !== '') {
          scanInfoListSet.push(wifi);
          wifiSet.add(wifi.ssid);
        }
      }

      // 取消注册，停止获取扫描状态。
      wifiManager.off("wifiScanStateChange", (result: number) => {
        console.info(`Receive Wifi scan state change event: ${result}`);
      });                                    // FIX (HW-19-0159): a different function object

      return scanInfoListSet;

    } catch (error) {
      console.error(`WiFi scan fail. ${error.message}`);
    }
    return [];
  }
```

The corrected registration:

```ts
private onScanStateChange = (result: number) => {
  if (result === 1) {
    this.refreshList();
  }
};
// register once
wifiManager.on('wifiScanStateChange', this.onScanStateChange);
// remove the same object
wifiManager.off('wifiScanStateChange', this.onScanStateChange);
// or remove every registration for the event
wifiManager.off('wifiScanStateChange');
```

### Mapping a scan result to the view model

`WiFIConnect.zip#WiFIConnect/entry/src/main/ets/utils/WiFiUtil.ets`

```ts
getWiFiInfoModel(scanInfo: wifiManager.WifiScanInfo): WiFiInfoModel {
  let wifiName: string = scanInfo.ssid;
  let securityType: string =
    scanInfo.securityType === wifiManager.WifiSecurityType.WIFI_SEC_TYPE_OPEN ? `开放` : `加密`;
  let secType: string = securityType === `加密` ? `sec` : `open`;
  let signalLevel: number = wifiManager.getSignalLevel(scanInfo.rssi, scanInfo.band);
  let src: string = `app.media.wifi_${secType}_${signalLevel}`;
  let wifiLabel: Resource = $r(src);
  return new WiFiInfoModel(wifiLabel, wifiName, securityType);
}
```

Note the security test collapses every non-open type into 加密 ("encrypted") -
WEP, WPA, WPA2, SAE and OWE are not distinguished, which is fine for a label but
would not be for a security decision.

### Connecting

`WiFIConnect.zip#WiFIConnect/entry/src/main/ets/utils/WiFiUtil.ets`

```ts
async connectWiFi(wifiScanInfo: wifiManager.WifiScanInfo, psk: string) {
  let config: wifiManager.WifiDeviceConfig = {
    ssid: wifiScanInfo.ssid,
    preSharedKey: psk,
    securityType: wifiScanInfo.securityType
  };
  let networkId = await wifiManager.addCandidateConfig(config);       // FIX (HW-19-0160)
  await wifiManager.connectToCandidateConfigWithUserAction(networkId); // FIX (HW-19-0160)
}
```

The corrected form:

```ts
async connectWiFi(wifiScanInfo: wifiManager.WifiScanInfo, psk: string): Promise<boolean> {
  const config: wifiManager.WifiDeviceConfig = {
    ssid: wifiScanInfo.ssid,
    preSharedKey: psk,
    securityType: wifiScanInfo.securityType
  };
  try {
    const networkId = await wifiManager.addCandidateConfig(config);
    await wifiManager.connectToCandidateConfigWithUserAction(networkId);
    return true;
  } catch (error) {
    const err = error as BusinessError;
    // 2501005 no response, 2501006 refused, 2501001 STA disabled
    console.error(`connect failed: ${err.code} ${err.message}`);
    return false;
  }
}
```

Note what the config does **not** carry: no `isHiddenSsid`, no `bssid`. For a
visible network chosen from a scan that is sufficient, and the security type is
taken from the scan result rather than guessed.

### The password sheet and the connect button

`WiFIConnect.zip#WiFIConnect/entry/src/main/ets/components/WiFiInfo.ets`

```ts
@Component
export struct WiFiInfo {
  @Prop wifiScanInfo: wifiManager.WifiScanInfo;
  @State wifiInfoModel?: WiFiInfoModel = undefined;
  @State isShowSheet: boolean = false;
  @State password: string = '';

  aboutToAppear(): void {
    this.wifiInfoModel = WiFiUtil.getWiFiInfoModel(this.wifiScanInfo);
  }

  @Builder
  SheetBuilder() {
    Column() {
      TextInput({
        placeholder: `密码`
      })
        .onChange((value: string) => {
          this.password = value;
        })
        .height(56)
        .padding({ top: 16, bottom: 16 })
        .borderRadius(0)
        .type(InputType.Password)
        .margin({ top: 30, left: 16, right: 16 })
        .backgroundColor(`#FAFAFA`);
      Divider()
        .margin({ left: 16, right: 16, bottom: 24 })
        .width('100%');
      Button(`连接`)
        .fontColor(`#0A59F7`)
        .width(328)
        .height(40)
        .borderRadius(20)
        .backgroundColor(`rgba(0, 0, 0, 0.05)`)
        .onClick(async () => {
          await WiFiUtil.connectWiFi(this.wifiScanInfo, this.password);
          this.isShowSheet = false;                     // FIX (HW-19-0160): runs regardless
          this.getUIContext().getPromptAction().showToast({
            message: `连接中`
          });
        });
    }
    .backgroundColor(`#FAFAFA`);
  }
```

`.type(InputType.Password)` is correct and important - it masks the field and
opts the input out of the system's text-prediction and clipboard history. The
password is held in `@State` and passed straight to `addCandidateConfig`; it is
never logged, which is the right call and worth preserving.

### The row and its sheet binding

`WiFIConnect.zip#WiFIConnect/entry/src/main/ets/components/WiFiInfo.ets`

```ts
.onClick(() => {
  this.isShowSheet = true;
})
.bindSheet($$this.isShowSheet, this.SheetBuilder(), {
  title: {
    title: this.wifiInfoModel?.wifiName
  },
  height: 600,
  backgroundColor: `#FAFAFA`
});
```

### The switch

`WiFIConnect.zip#WiFIConnect/entry/src/main/ets/pages/Index.ets`

```ts
@Entry
@Component
struct Index {
  @State wifiList: wifiManager.WifiScanInfo[] = [];
  password: string = '';
  isStartScan: boolean = false;
  @State isWifiSwitchOn: boolean = false;

  aboutToAppear(): void {
    let isWifiActive = wifiManager.isWifiActive();
    if (!isWifiActive) {
      promptAction.openToast({ message: '请先手动打开WiFi' });
      console.error("wifi not enable");
    }
  }

  // ...
  Toggle({ type: ToggleType.Switch, isOn: false })
    .id('wifi_Switch')
    .margin({ right: 12 })
    .onChange((isOn: boolean) => {
      this.isWifiSwitchOn = isOn;
      if (isOn) {
        if (this.isStartScan === false) {
          try {
            // 获取扫描结果前，先发起一次扫描，由于系统本身也会定期扫描，这里发起一次扫描即可。
            wifiManager.startScan();
            this.isStartScan = true;      // FIX (HW-19-0161): never reset
          } catch (error) {
            console.error(`startScan failed, ${JSON.stringify(error)}`);
          }
        }
        this.wifiList = WiFiUtil.getScanResult();   // FIX (HW-19-0161): reads stale results
      } else {
        this.wifiList = [];
      }
    });
```

The `password: string = ''` field on line 10 is undecorated and unused - the
password lives in the child component.

## Permissions & config

`WiFIConnect.zip#WiFIConnect/entry/src/main/module.json5`:

```json5
"requestPermissions": [
  { "name": "ohos.permission.GET_WIFI_INFO" },
  { "name": "ohos.permission.SET_WIFI_INFO" }
]
```

Both are normal, install-time permissions, and both are genuinely needed - the
declaration matches the APIs exactly:

| API | Required permission |
| --- | --- |
| `getScanInfoList`, `isWifiActive`, `off('wifiScanStateChange')` | `GET_WIFI_INFO` |
| `startScan`, `addCandidateConfig`, `connectToCandidateConfigWithUserAction` | `SET_WIFI_INFO` |

Nothing beyond these two is requested - notably no location permission, which
older Wi-Fi scanning APIs on other platforms require. The document's 权限说明
lists exactly these two.

`build-profile.json5` pins `targetSdkVersion` and `compatibleSdkVersion` to
`6.0.1(21)`.

## Constraints

- **API level.** API Version 21 Release or later, HarmonyOS 6.0.1 Release SDK or
  later, DevEco Studio 6.0.1 Release or later.
  `connectToCandidateConfigWithUserAction` is API 20+; `getScanInfoList` is API
  10+; `startScan` is API 21+.
- **The application cannot enable Wi-Fi.** `isWifiActive()` is a check, not a
  switch; the user must turn it on in system settings.
- **`startScan` is asynchronous.** Results are not available when it returns -
  subscribe to `wifiScanStateChange`.
- **`connectToCandidateConfigWithUserAction` always prompts** and does not connect
  until the user confirms. `2501005` and `2501006` are normal outcomes.
- **`off(type, callback)` removes only the identical callback object;** omit the
  callback to remove all.
- **Scanning changes channel** and disturbs latency-sensitive traffic - the
  document's reason for scanning at most once.
- **`$r()` with a computed resource name is resolved at runtime**, so every
  `wifi_{sec,open}_{0..4}` media resource must exist.
- **Devices.** Per `module.json5`.

## Pitfalls

- **`off('wifiScanStateChange', <new arrow function>)` cannot remove the
  registered callback, which is incorrect** - and two return paths in the same
  function skip the `off` entirely, so listeners accumulate one per scan.
  (HW-19-0159)
- **Neither awaited connect call is guarded and the UI commits regardless, which
  is incorrect** - a user who declines the system's trust prompt sees the sheet
  close and a 连接中 toast, and the rejection goes unhandled. (HW-19-0160)
- **The scan list is read synchronously after `startScan`, which is incorrect** -
  the reference prescribes reading it when `wifiScanStateChange` reports
  completion, and the sample registers that listener only to log. `isStartScan`
  also never resets, so the list can never refresh. (HW-19-0161)
- **One file imports `wifiManager` from `@ohos.wifiManager` while its two
  neighbours use `@kit.ConnectivityKit`, which is incorrect** - and
  `WifiScanInfo` values cross that boundary. (HW-19-0162)
- **Do not log the pre-shared key.** The sample does not, and that is worth
  keeping - `console.error(\`${JSON.stringify(config)}\`)` while debugging would
  put it in the device log.
- **Do not treat "not open" as "secure".** The security label collapses every
  non-open type into one bucket; use the actual `WifiSecurityType` where the
  distinction matters.
- **Do not expect the connection to be up when the promise resolves.** It resolves
  once the user has confirmed and the request is accepted; association and DHCP
  follow.

## References

- `documentation/harmonyos-references/03_system/js-apis-wifimanager.md` -
  `startScan` (and its scan-state guidance), `getScanInfoList`,
  `addCandidateConfig`, `connectToCandidateConfigWithUserAction` (the user-prompt
  behaviour and error codes 2501000-2501007), `on`/`off('wifiScanStateChange')`
  and the callback-matching rule, `getSignalLevel`, `WifiSecurityType`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-wifimanager
- https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/permissions-for-all -
  `ohos.permission.GET_WIFI_INFO`, `ohos.permission.SET_WIFI_INFO`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-attributes-sheet-transition -
  `bindSheet` and its `$$` two-way binding.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/wifi_connect-0000002518583904
