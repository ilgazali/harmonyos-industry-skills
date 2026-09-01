---
id: UTIL-18
title: Join Wi-Fi from a QR code - scanBarcode into a candidate WifiDeviceConfig
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/18_wifi_scan.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/wifi_scan-0000002330272949
sample: huawei_industry_tree/15_utilities/downloads/WifiScan.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.ConnectivityKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit", "@kit.ScanKit"]
apis: [hilog, scanBarcode, scanCore, wifiManager, window]
permissions: [ohos.permission.SET_WIFI_INFO, ohos.permission.GET_WIFI_INFO]
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0046, HW-15-0047, HW-15-0101, HW-15-0102]
status: verified-with-fixes
---

## When to use

Load this card when the app should **join a network from a printed or
displayed QR code** - the sticker on a router, the code on a café receipt, a
guest-network placard, a code shown by a companion device during onboarding.
The user points the camera at the code and the device connects; no SSID list,
no typing a 63-character passphrase.

The pattern is three calls deep: `scanBarcode.startScanForResult` opens the
system scan UI and resolves with the decoded string,
`wifiManager.addCandidateConfig` registers the credentials and returns a
network id, and `wifiManager.connectToCandidateConfig` joins with it. The
"candidate" wording matters - a candidate network is one the *app* proposed
rather than one the user picked in Settings, and the system keeps it
app-scoped and revocable. That is why this whole flow needs only
`ohos.permission.SET_WIFI_INFO` and `ohos.permission.GET_WIFI_INFO`, both
`normal`-level, with no user prompt.

**The generalisable half is the parsing, and it is the half the sample gets
wrong.** The `WIFI:` URI is a tagged, order-free, escaped format, and this
sample reads it by array index (`HW-15-0046`) and ignores the security tag
entirely (`HW-15-0047`). Take the connect sequence from here; write the
parser yourself.

## Feature checklist

- A WLAN settings-style page with a title bar, a scan icon, an on/off toggle
  and a list of nearby networks.
- Tapping the scan icon opens the system barcode scanner (all code types,
  multi-code and album import enabled).
- A scanned Wi-Fi QR yields an SSID and a passphrase.
- The credentials are registered as a candidate configuration and the device
  connects to that network.
- Flipping the toggle on reveals the network list and shows the currently
  connected SSID at the top when `wifiManager.isConnected()` is true.
- Scan failures and connection failures are logged, not surfaced (the sample
  has no error UI at all).

## Architecture

One `entry` module, four source files. There is no model layer and no state
container: the scan result is parked in `static` fields on the utility class.

```
entry/src/main/ets
├── common/Constants.ets            font sizes, paddings and the fixed margins the layout leans on
├── entryability/EntryAbility.ets   full screen + avoid areas -> AppStorage
├── entrybackupability/
├── pages/MainPage.ets              @Entry: title bar, toggle, the WLAN list builder
└── utils/Scan.ets                  the whole feature: scan, parse, addCandidateConfig, connect
```

The documented 工程目录 matches the zip.

**The design decision worth copying** is that `Scan.Scan(context: UIContext)`
takes the `UIContext` and derives the ability context from it with
`context.getHostContext()`. `startScanForResult` needs a real
`common.Context` to raise the system scan UI, and passing the `UIContext`
down from the component keeps the utility free of any global-context
singleton - contrast `UTIL-17`, which needs a `GlobalContext` holder for
exactly the same reason.

**The decision worth avoiding** is the state: `Scan.ssid`, `Scan.pwd`,
`Scan.arr`, `Scan.ssidArr`, `Scan.pwdArr` are all `static` on the class, and
`MainPage` renders `Text(Scan.ssid)` directly. A plain static field is not
observable, so the connected-SSID row only refreshes when something else
re-renders the builder - which is why the toggle handler writes `Scan.ssid`
and then relies on `this.isOpen` changing to force the redraw. Put the SSID
in `@State` (or `AppStorage`) and the coupling disappears.

## Implementation steps

1. **Declare `SET_WIFI_INFO` and `GET_WIFI_INFO`** in `module.json5`. Both
   are `normal`-level, so no runtime request and no `reason` string are
   needed.
2. **Configure `ScanOptions` once** as a static: `scanTypes`,
   `enableMultiMode`, `enableAlbum`. Narrow `scanTypes` to
   `scanCore.ScanType.QR_CODE` if the feature only ever accepts Wi-Fi codes.
3. **Call `scanBarcode.startScanForResult(context.getHostContext(), options)`**
   and read `result.originalValue` - the raw decoded string, not a parsed
   payload; the Scan Kit does not interpret `WIFI:` URIs for you.
4. **Validate the `WIFI:` prefix and parse by tag, not by position**
   (`HW-15-0046`). Handle the `\;` `\:` `\,` `\\` escapes.
5. **Map the `T:` tag onto `WifiSecurityType`** instead of hardcoding `3`
   (`HW-15-0047`). `nopass`/absent is `WIFI_SEC_TYPE_OPEN` (1), `WPA` is
   `WIFI_SEC_TYPE_PSK` (3), `SAE`/WPA3 is `WIFI_SEC_TYPE_SAE` (4).
6. **`addCandidateConfig(config)` resolves to a network id**; pass that id -
   not the config - to `connectToCandidateConfig`.
7. **Wrap every `wifiManager` await in a `catch`.** `getLinkedInfo` rejects
   with `2501001` (*Wi-Fi STA disabled*) whenever WLAN is off, which is the
   normal state on the toggle-off path (`HW-15-0047`).
8. **Publish the SSID through observable state** so the connected row
   actually re-renders.

## Verified snippets

All snippets are from `WifiScan.zip`. Corrected forms are marked.

**The connect sequence - `entry/src/main/ets/utils/Scan.ets`** (as shipped)

```typescript
static options: scanBarcode.ScanOptions = {
  scanTypes: [scanCore.ScanType.ALL],
  enableMultiMode: true,
  enableAlbum: true
};

static Scan(context: UIContext) {
  try {
    scanBarcode.startScanForResult(context.getHostContext(), Scan.options).then((result: scanBarcode.ScanResult) => {
      // 收到正确的扫码返回
      Scan.arr = result.originalValue.split(';');
      Scan.ssidArr = Scan.arr[1].split(':');
      Scan.pwdArr = Scan.arr[2].split(':');
      Scan.ssid = Scan.ssidArr[1];
      Scan.pwd = Scan.pwdArr[1];
      // 获取WifiDeviceConfig
      let config: wifiManager.WifiDeviceConfig = {
        ssid: Scan.ssid,        // wifi名称
        preSharedKey: Scan.pwd, // 自定义wifi密码
        securityType: 3         // WLAN加密类型
      };
      Scan.wifiDeviceConfigs.push(config);

      wifiManager.addCandidateConfig(config).then(result => {
        // 获取ID
        wifiManager.connectToCandidateConfig(result);
      }).catch((err: number) => {
        console.error('failed:' + JSON.stringify(err));
      });
    }).catch((error: BusinessError) => {
      console.error('failed:' + JSON.stringify(error));
    });
  } catch (error) {
    console.error('failed:' + JSON.stringify(error));
  }
}
```

**The last four lines are the part to keep.** `addCandidateConfig` returns a
`Promise<number>`, and that number is the network id the system assigned to
*your* candidate; `connectToCandidateConfig(networkId)` is synchronous and
returns nothing, so success is observed through
`wifiManager.isConnected()` / `getLinkedInfo()`, not through a resolved
promise. Note also the double error handling: `startScanForResult` can both
throw synchronously (bad options, no scan capability) and reject
asynchronously (user cancelled), so the `try` and the `.catch` are both
needed.

Everything above those lines is the defect. `result.originalValue` for a
Wi-Fi code looks like `WIFI:S:mynet;T:WPA;P:secret;H:false;;`. Splitting on
`;` and taking `arr[1]`/`arr[2]` only works for the single field order
`S`,`P`; a code written `S`,`T`,`P` - which is the order the format's own
examples use - puts `T:WPA` into `arr[1]`, so the SSID becomes `WPA`.
Splitting each field on `:` and taking `[1]` truncates any passphrase
containing a colon. Nothing checks the `WIFI:` prefix, so a URL or a vCard
produces a garbage config instead of a message.

**A tag-based parser - same file** (corrected, see `HW-15-0046`, `HW-15-0047`)

```typescript
// FIX: replaces the index-based Scan.arr/ssidArr/pwdArr splits
static parseWifiQr(raw: string): wifiManager.WifiDeviceConfig | undefined {
  if (!raw.startsWith('WIFI:')) {
    return undefined;                                   // not a Wi-Fi code at all
  }
  let fields = new Map<string, string>();
  let body = raw.substring('WIFI:'.length);
  let key = '';
  let value = '';
  let seenColon = false;
  for (let i = 0; i < body.length; i++) {
    let ch = body.charAt(i);
    if (ch === '\\') {                                  // \; \: \, \\ are literal characters
      i++;
      value += body.charAt(i);
    } else if (ch === ':' && !seenColon) {
      seenColon = true;
      key = value;
      value = '';
    } else if (ch === ';') {
      if (seenColon) {
        fields.set(key, value);
      }
      key = '';
      value = '';
      seenColon = false;
    } else {
      value += ch;
    }
  }
  let ssid = fields.get('S');
  if (ssid === undefined || ssid.length === 0) {
    return undefined;
  }
  let type = fields.get('T');
  let securityType: number = wifiManager.WifiSecurityType.WIFI_SEC_TYPE_PSK;
  if (type === undefined || type === 'nopass') {
    securityType = wifiManager.WifiSecurityType.WIFI_SEC_TYPE_OPEN;
  } else if (type === 'SAE') {
    securityType = wifiManager.WifiSecurityType.WIFI_SEC_TYPE_SAE;
  }
  return {
    ssid: ssid,
    preSharedKey: fields.get('P') ?? '',
    securityType: securityType
  };
}
```

**Four rules define this format and the loop implements all four.** Fields
are `TAG:value;` pairs; the tags may appear in any order; a backslash escapes
the next character so `;` `:` `,` `\` can appear inside an SSID or a
passphrase; and only the *first* colon of a field separates tag from value,
which is why `seenColon` exists - `P:a:b` is the passphrase `a:b`, not a
malformed field.

The security mapping is the second half of the fix. `securityType: 3` is
`WIFI_SEC_TYPE_PSK`, so as shipped an open network gets a PSK config with an
empty passphrase and a WPA3-only network gets a PSK config the AP will
reject - in both cases the join fails with no explanation. Note that
`WIFI_SEC_TYPE_WEP` (2) is deliberately unreachable above: the reference
states WEP "is not supported for the candidate network configuration added by
`addCandidateConfig`", so a `T:WEP` code cannot be honoured by this API at
all and deserves an explicit message rather than a silent failure.

**The toggle handler - `entry/src/main/ets/pages/MainPage.ets`** (corrected, see `HW-15-0047`)

```typescript
Toggle({ type: ToggleType.Switch, isOn: false })
  .onChange(async (isOn: boolean) => {
    this.isOpen = isOn;
    if (!isOn) {
      return;                                   // FIX: shipped code queries on the OFF flip too
    }
    try {
      let wifiLinkedInfo = await wifiManager.getLinkedInfo();
      Scan.ssid = wifiLinkedInfo.ssid;
    } catch (err) {                             // FIX: 2501001 (Wi-Fi STA disabled) was unhandled
      console.error(`getLinkedInfo failed: ${JSON.stringify(err)}`);
    }
  })
  .margin({ left: Constants.MARGIN_240 });
```

`getLinkedInfo()` rejects rather than resolving with an empty object when
there is no connection, and `2501001` - *Wi-Fi STA disabled* - is not an
exceptional case here: it is what happens every single time the user turns
WLAN off, because the shipped handler runs the same query on both edges of
the toggle. An unhandled rejection inside an `async` UI callback is an
uncaught promise error, so the sample's own primary control is the fastest
way to crash it.

**Avoid areas into padding - same file** (as shipped)

```typescript
@StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
@StorageProp('topRectHeight') topRectHeight: number = 0;

// ...
.padding({
  top: this.getUIContext().px2vp(this.topRectHeight),
  bottom: this.getUIContext().px2vp(this.bottomRectHeight)
})
```

`EntryAbility` sets `setWindowLayoutFullScreen(true)`, reads
`TYPE_SYSTEM`/`TYPE_NAVIGATION_INDICATOR` avoid areas into `AppStorage`, and
registers `avoidAreaChange` to keep them current. The page reads them back
with `@StorageProp` (read-only, defaults to `0`) and converts px to vp at the
point of use. This is the correct shape of the boilerplate and worth
copying verbatim - note that the `avoidAreaChange` subscription is never
released in `onWindowStageDestroy`, the same omission seen across this
industry's samples.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.SET_WIFI_INFO",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" } },
  { "name": "ohos.permission.GET_WIFI_INFO",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" } }
]
```

- Both are `normal`-level: granted at install, no runtime dialog, no `reason`
  string required. That is why this sample has no permission code at all.
- `SET_WIFI_INFO` covers `addCandidateConfig` and
  `connectToCandidateConfig`; `GET_WIFI_INFO` covers `isConnected` and
  `getLinkedInfo`.
- No camera permission is declared and none is needed: `startScanForResult`
  raises the *system* scan UI, which holds the camera in its own process.
  Requesting `ohos.permission.CAMERA` for a default-UI scan is a common and
  unnecessary mistake.
- `getLinkedInfo().macAddress` would additionally need
  `ohos.permission.GET_WIFI_LOCAL_MAC`; the sample reads only `ssid`, so it
  does not.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Only the connected-network row is real. The two 可用WLAN entries below it
  are static string resources (`wifi_name_1`, `wifi_name_2`) - there is no
  `wifiManager.scan()` / `getScanResults()` anywhere in the sample, so
  "scan" in the feature name refers to the QR code, not to a Wi-Fi scan.
- The 刷新 (refresh) label is a `Text` with no `onClick`.
- The layout positions its columns with fixed margins (`MARGIN_200`,
  `MARGIN_210`, `MARGIN_240`) rather than with `layoutWeight` or
  `justifyContent`, so it only lines up at one width. On a tablet or a
  resized 2in1 window the toggle and the info icons drift.
- `Scan.wifiDeviceConfigs` accumulates every scanned config for the app's
  lifetime and is never read.
- `addCandidateConfig` cannot express WEP; see the reference note above.

## Pitfalls

- **`HW-15-0046`** (B/medium, confirmed): the Wi-Fi QR is parsed by fixed
  field positions and naive splits - `arr[1]` is assumed to be `S:` and
  `arr[2]` to be `P:`. The `WIFI:` format allows any field order and escaped
  `;`/`:`, so `WIFI:S:mynet;T:WPA;P:pw;;` yields `ssid = 'WPA'`, a passphrase
  containing `:` is truncated, and a non-Wi-Fi QR falls into a generic
  failure because nothing validates the `WIFI:` prefix. Fix: parse by field
  tag with escape handling, as in the corrected snippet.
- **`HW-15-0047`** (B/medium, confirmed): two defects in one. `securityType`
  is hardcoded to `3` (`WIFI_SEC_TYPE_PSK`) and the `T:` tag is ignored, so
  open, WEP and WPA3/SAE networks all get a wrong candidate config and can
  never join. And `MainPage`'s toggle handler `await`s
  `wifiManager.getLinkedInfo()` with no `catch`, on both edges of the switch,
  so flipping it while WLAN is off or nothing is connected rejects with
  `2501001` unhandled. Fix: map `T:` onto `WifiSecurityType`, and guard the
  await (and skip it entirely on the off edge).

## References

- `documentation/harmonyos-references/04_media/scan-scanbarcode-api.md` -
  `startScanForResult`, `ScanOptions`, `ScanResult.originalValue`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/scan-scanbarcode-api
- `documentation/harmonyos-references/03_system/js-apis-wifimanager.md` -
  `WifiDeviceConfig`, `WifiSecurityType`, `addCandidateConfig`,
  `connectToCandidateConfig`, `getLinkedInfo`, error code `2501001`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-wifimanager
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` -
  `ohos.permission.SET_WIFI_INFO` and `ohos.permission.GET_WIFI_INFO`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `UTIL-17` - the alternative when you must own the camera yourself rather
  than delegate to the system scan UI
