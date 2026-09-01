---
id: UTIL-09
title: Network status panel - query the default net and push the result to the page over eventHub
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/09_network_awareness.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/network_awareness-0000002284512886
sample: huawei_industry_tree/15_utilities/downloads/NetworkAwareness.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.NetworkKit", "@kit.PerformanceAnalysisKit"]
apis: [connection, hilog, window]
permissions: [ohos.permission.GET_NETWORK_INFO, ohos.permission.INTERNET]
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0027, HW-15-0101, HW-15-0102]
status: verified-with-fixes
---

## When to use

**Load this card when the app has to show what the network actually is** -
bearer type, interface name, MTU, addresses, routes - rather than merely react
to online/offline. Diagnostics screens, "why is this slow" panels, VPN and
tethering tools, and enterprise support screens all need this shape.

The pattern is a three-part chain: `connection.getDefaultNet()` to obtain a
`NetHandle`, `connection.getNetCapabilities` and
`connection.getConnectionPropertiesSync` to expand it into facts, and an
`eventHub` emit to hand the finished snapshot to whatever page is showing.
The utility knows nothing about the UI; the page knows nothing about the
network stack.

It generalises to any query whose result arrives through a callback that is
not a promise the caller can await - the eventHub hop is the standard ArkTS
answer when the producer lives in a static utility class and the consumer is
a `@Entry` component. **Read `HW-15-0027` before adopting the sample's
transport**: the snapshot travels as a bare `string[]` read by fixed index,
and the array is module-level and never cleared, which is the one defect that
breaks the app on its second use.

## Feature checklist

- The page opens on an empty state: an illustration and a "please tap to
  check" hint.
- A full-width button at the bottom triggers the query.
- After the query the page lists ten labelled rows: up/down bandwidth,
  network type, interface name, MTU, link addresses, route interface,
  destination address, gateway address, has-gateway, is-default-route.
- The bearer type is rendered in words (蜂窝网络 / WIFI / 以太网 / VPN), not
  as the raw enum.
- With no network connected the page shows a single `error：` row reading
  `No network connected`.
- A failed `getNetCapabilities` shows the error message in the same single
  row.
- Tapping the button a second time replaces the previous snapshot with a
  fresh one.

## Architecture

One `entry` module, four files. No model layer - the snapshot is an array of
strings.

```
entry/src/main/ets
├── constants/Constants.ets       row labels, bearer-type names, layout numbers
├── entryability/EntryAbility.ets full-screen window, avoid areas -> AppStorage
├── pages/MainPage.ets            @Entry, empty state / list, the check button
└── utils/NetworkUtil.ets         the whole connection query + the eventHub emit
```

The documented 工程目录 matches the zip exactly - four directories, four
files, no backup ability.

**The design decision worth copying** is the eventHub hop.
`connection.getNetCapabilities` is a callback API nested inside the promise
from `getDefaultNet`, so by the time the last route is appended the caller's
stack is long gone. Rather than passing the component in, or making the
utility a stateful singleton the page subscribes to, `NetworkUtil` takes a
plain `Context` and emits one named event:

```typescript
context.eventHub.emit('netStatusChange', statusOutput);
```

`MainPage` registers `eventHub.on('netStatusChange', ...)` in
`aboutToAppear`. The utility stays a pure static class, and any other page
could show the same snapshot by listening to the same event.

**The part worth avoiding** is what travels over that event. The payload is a
module-scope `string[]` that the page decodes by position -
`statusOutput[0]` is bandwidth, `[1]` type, `[2]` interface name, and so on
to `[9]`. Two things follow. The array is never reset, so the second query
appends behind the first and every index is wrong (`HW-15-0027`). And the
no-network path writes `statusOutput[0]` in place while the ten-row path
pushes, so a first failed run permanently shifts every later field by one.
A small class with named fields - the sample already has `StatusItem` in
`MainPage` - removes both problems at once.

Note also that `getNetworkStatus` declares `: string[]` and returns the module
array synchronously, before any callback has run. `MainPage` assigns that
return value to `this.statusOutput` and it is always the stale array; the
value that actually matters arrives later over the event. The return type is
decorative and should be `void`.

## Implementation steps

1. **Declare `GET_NETWORK_INFO` and `INTERNET`** in `module.json5`. Both are
   `system_grant`, so no `reason`/`usedScene` and no runtime request.
2. **Reset the output before every query** (`HW-15-0027`). Whether it stays an
   array or becomes a struct, the first statement of the query must clear the
   previous snapshot.
3. **Call `connection.getDefaultNet()`** and treat `netHandle.netId === 0` as
   "no network connected" - that is the documented sentinel, not an error.
4. **Expand capabilities in the callback**: `linkUpBandwidthKbps` /
   `linkDownBandwidthKbps` (0 means "cannot be evaluated", not "zero
   bandwidth") and `bearerTypes[0]` mapped to a display name.
5. **Read link properties synchronously** with
   `connection.getConnectionPropertiesSync(netHandle)` - inside the
   capabilities callback the handle is known good, so the sync form is safe
   and keeps the nesting flat.
6. **Join the link addresses yourself**; `linkAddresses` is an array and the
   UI wants one line.
7. **Emit once, at the end of each terminal path** - success, no-network, and
   both error paths. Every path in the sample calls `statusListening`, which
   is why the empty state never sticks.
8. **Unsubscribe in `aboutToDisappear`** with `eventHub.off('netStatusChange')`
   (`HW-15-0027`); the sample registers but never releases.
9. **Gate the list on a separate `isChecked` flag**, so the illustration shows
   until the user has actually asked, independent of whether the snapshot
   happens to be empty.

## Verified snippets

All snippets are from `NetworkAwareness.zip`. Corrected forms are marked.

**The query — `entry/src/main/ets/utils/NetworkUtil.ets`** (corrected, see `HW-15-0027`)

```typescript
import { connection } from '@kit.NetworkKit';
import { BusinessError } from '@kit.BasicServicesKit';
import { Constants } from '../constants/Constants';

let statusOutput: string[] = [];

export class NetworkUtil {
  static statusListening(context: Context) {
    // 触发网络状态变化
    context.eventHub.emit('netStatusChange', statusOutput);
  }

  static getNetworkStatus(context: Context): void {   // FIX: was `: string[]`, returning a stale array
    statusOutput = [];                                // FIX: absent in the sample - the defect of HW-15-0027
    connection.getDefaultNet().then((netHandle: connection.NetHandle) => {
      if (netHandle.netId === 0) {
        // 当前没有已连接的网络时，获取的 netHandler 的 netid 为 0
        statusOutput[0] = Constants.NOT_CONNECTED;
        NetworkUtil.statusListening(context);
        return;
      }
      connection.getNetCapabilities(netHandle, (error: BusinessError, netCapabilities: connection.NetCapabilities) => {
        if (error) {
          statusOutput[0] = `${error.message}`;
          NetworkUtil.statusListening(context);
          return;
        }
        // 上/下行带宽，单位(kb/s)，0表示无法评估当前网络带宽。
        statusOutput.push(`${netCapabilities.linkUpBandwidthKbps}/${netCapabilities.linkDownBandwidthKbps} kb/s`);

        let netBearType: connection.NetBearType = netCapabilities.bearerTypes[0];
        statusOutput.push(`${GET_NET_BEAR_TYPE(netBearType)}`);

        let properties: connection.ConnectionProperties = connection.getConnectionPropertiesSync(netHandle);
        statusOutput.push(`${properties.interfaceName}`);
        statusOutput.push(`${properties.mtu}`);

        let currentNetInfoTmp = '';
        properties.linkAddresses.forEach((address, index, arr) => {
          currentNetInfoTmp += address.address.address;
          if (index < arr.length - 1) {
            currentNetInfoTmp += ',';
          }
        });
        statusOutput.push(currentNetInfoTmp);

        properties.routes.forEach((route) => {
          statusOutput.push(route.interface);
          statusOutput.push(route.destination.address.address);
          statusOutput.push(route.gateway.address);
          statusOutput.push(route.hasGateway ? Constants.TRUE : Constants.FALSE);
          statusOutput.push(route.isDefaultRoute ? Constants.TRUE : Constants.FALSE);
        });
        NetworkUtil.statusListening(context);
      });
    }).catch((error: BusinessError) => {
      statusOutput[0] = `${JSON.stringify(error)}`;
      NetworkUtil.statusListening(context);
    });
  }
}
```

**Three lines carry the design.** `netHandle.netId === 0` is the documented
"nothing connected" signal - `getDefaultNet` resolves successfully, so
checking the promise is not enough. `getNetCapabilities` is the async
callback form and `getConnectionPropertiesSync` the synchronous one; mixing
them deliberately is right here, because inside the capabilities callback the
handle has already proven valid and the sync call avoids a third nesting
level. And `statusOutput = []` - the one line the sample is missing - is what
makes the button idempotent.

`linkUpBandwidthKbps` returning `0` means the bearer cannot evaluate its
bandwidth, which is the normal case on Wi-Fi. Rendering it as `0/0 kb/s` is
misleading; a real diagnostics panel should print a dash.

**The label mapping — same file** (as shipped)

```typescript
const GET_NET_BEAR_TYPE = (type: number): string => {
  switch (type) {
    case connection.NetBearType.BEARER_CELLULAR:
      return Constants.CELLULAR_NETWORK;   // 蜂窝网络
    case connection.NetBearType.BEARER_WIFI:
      return Constants.WIFI;
    case connection.NetBearType.BEARER_ETHERNET:
      return Constants.ETHERNET;           // 以太网
    case connection.NetBearType.BEARER_VPN:
      return Constants.VPN;
    default:
      return '';
  }
};
```

Only `bearerTypes[0]` is read. A net handle can carry more than one bearer -
a VPN over Wi-Fi is the everyday case - and the first entry is not promised
to be the interesting one. If the panel is meant to be diagnostic, map the
whole array and join it; the `default: ''` branch also silently produces a
blank row for any bearer the switch does not know.

**The listener and the fixed-index decode — `entry/src/main/ets/pages/MainPage.ets`** (corrected, see `HW-15-0027`)

```typescript
interface StatusItem {
  title: string;
  value: string;
}

@Entry
@Component
struct MainPage {
  @State statusOutput: string[] = [];
  @State isChecked: boolean = false;
  @State physiological: StatusItem[] = [];
  private context = this.getUIContext().getHostContext() as Context;

  aboutToAppear(): void {
    this.context.eventHub.on('netStatusChange', (statusOutput: string[]) => {
      this.statusOutput = statusOutput;
      if (this.statusOutput.length > 1) {
        this.physiological = [
          { title: Constants.BANDWIDTH, value: this.statusOutput[0] },
          { title: Constants.TYPE, value: this.statusOutput[1] },
          { title: Constants.NAME, value: this.statusOutput[2] },
          { title: Constants.UNIT, value: this.statusOutput[3] },
          { title: Constants.MSG, value: this.statusOutput[4] },
          { title: Constants.ROUTING_NAME, value: this.statusOutput[5] },
          { title: Constants.DESTINATION_ADDRESS, value: this.statusOutput[6] },
          { title: Constants.GATEWAY_ADDRESS, value: this.statusOutput[7] },
          { title: Constants.GATEWAY, value: this.statusOutput[8] },
          { title: Constants.DEFAULT_ROUTE, value: this.statusOutput[9] }
        ];
      } else {
        this.physiological = [{ title: Constants.ERROR, value: this.statusOutput[0] }];
      }
    });
  }

  aboutToDisappear(): void {
    this.context.eventHub.off('netStatusChange');   // FIX: absent in the sample
  }
}
```

**`StatusItem` is the shape the payload should have had.** The page already
declares a `{title, value}` record and builds ten of them by hand from
positions 0-9; moving that structure into `NetworkUtil` would delete the
index arithmetic, the `length > 1` heuristic that stands in for "did it
succeed", and the whole class of shifted-row bugs. As shipped, one route with
two entries also produces fifteen strings and the extra five are silently
dropped.

`eventHub.off` matters more here than in a single-page demo: the handler
closes over `this`, so a page that is destroyed and re-entered leaves the old
component alive and writing state nobody renders.

**The empty state and the trigger — same file** (as shipped)

```typescript
if (this.isChecked) {
  Column() {
    ForEach(this.physiological, (item: StatusItem) => {
      Row() {
        Text(item.title).fontColor($r('sys.color.mask_secondary'));
        Text(item.value).fontColor($r('sys.color.mask_secondary'));
      }
      .width(Constants.FULL_WIDTH)
      .justifyContent(FlexAlign.Start)
    });
  }
  Blank();
} else {
  Column() {
    Image($r('app.media.empty')).width($r('app.float.image_empty_width'));
    Text($r('app.string.empty'));
    Text($r('app.string.please_click')).fontColor($r('app.color.text_please_click_font_color'));
  }
}

Row() {
  Button($r('app.string.check'))
    .onClick(() => {
      this.isChecked = true;
      NetworkUtil.getNetworkStatus(this.context);   // FIX: sample assigns the (stale) return value
    })
    .width(Constants.FULL_WIDTH)
}
```

The gate is `isChecked`, not `physiological.length`, which is the right
choice: the error path legitimately produces one row, and an empty result
should still count as "answered". The surrounding `Column` uses
`justifyContent(FlexAlign.SpaceBetween)` with a `Blank()` after the list, so
the button stays pinned to the bottom in both states without a stack or
absolute positioning.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.GET_NETWORK_INFO" },
  { "name": "ohos.permission.INTERNET" }
]
```

- Both are `system_grant`. No `reason`, no `usedScene`, no
  `requestPermissionsFromUser` call - declaring them is the whole job.
- `GET_NETWORK_INFO` is what `getDefaultNet`, `getNetCapabilities` and
  `getConnectionPropertiesSync` need. `INTERNET` is not used by this sample's
  queries at all; it is declared for completeness.
- `deviceTypes` is `phone`, `tablet`, `2in1`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The panel is a snapshot on demand. Despite `statusListening`'s name nothing
  registers a `NetConnection` observer, so a Wi-Fi drop while the page is open
  leaves the stale rows on screen. For live updates add
  `connection.createNetConnection()` and its `netAvailable` /
  `netCapabilitiesChange` / `netLost` handlers, and unregister them in
  `aboutToDisappear`.
- Only the first bearer and all routes of the default net are shown; other
  active nets (`connection.getAllNets`) are ignored.
- Row labels are hardcoded Chinese strings in `Constants.ets`, not string
  resources, so the panel cannot be localised without touching code.

## Pitfalls

- **`HW-15-0027`** (B/high, confirmed): module-level `statusOutput` is never
  cleared, so every check appends behind the previous snapshot while the page
  reads fixed indexes 0-9 - from the second tap on, the UI shows the first
  result, and a first "no network" run shifts every later field by one
  (bandwidth rendered as network type, and so on). The same event handler is
  registered in `aboutToAppear` and never `off`'d.
  Fix: `statusOutput = []` at the start of each query, and
  `eventHub.off('netStatusChange')` in `aboutToDisappear`.

## References

- `documentation/harmonyos-references/03_system/js-apis-net-connection.md` - `getDefaultNet`, `NetHandle`, `getNetCapabilities`, `NetCapabilities`, `getConnectionPropertiesSync`, `ConnectionProperties`, `NetBearType`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-net-connection
- `documentation/harmonyos-references/02_application-framework/js-apis-inner-application-eventhub.md` - `eventHub.emit` / `on` / `off`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inner-application-eventhub
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.GET_NETWORK_INFO`, `ohos.permission.INTERNET`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `huawei_industry_tree/15_utilities/docs/09_network_awareness.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/network_awareness-0000002284512886
