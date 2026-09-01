---
id: EDU-12
title: Live network status and speed - NetConnection events plus netQuality QoS sampling
industry: 04_education
doc: huawei_industry_tree/04_education/docs/12_network_monitor.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/network_monitor-0000002355490078
sample: huawei_industry_tree/04_education/downloads/MonitorNetwork.zip
kits: ["@kit.NetworkKit", "@kit.NetworkBoostKit", "@kit.ArkTS", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["connection.createNetConnection", "NetConnection.register", "NetConnection.unregister", "on('netAvailable')", "on('netLost')", "on('netCapabilitiesChange')", "on('netConnectionPropertiesChange')", "connection.getDefaultNetSync", "connection.getNetCapabilitiesSync", "connection.NetBearType", "netQuality.on('netQosChange')", "netQuality.off('netQosChange')", "netQuality.NetworkQos", linkDownRate, Decimal, toDecimalPlaces, setInterval, clearInterval]
permissions: ["ohos.permission.GET_NETWORK_INFO"]
min_api: 20
modules: [entry]
findings: [HW-04-0085, HW-04-0086, HW-04-0087, HW-04-0088, HW-04-0089, HW-04-0090, HW-04-0091, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card when the app must **show the user what their connection is doing**
- WiFi or mobile, and how fast - before or during something bandwidth-sensitive:
starting a video lesson, downloading a course pack, joining a live class.

Two different APIs are needed and they answer different questions:

- **`@kit.NetworkKit`'s `connection`** answers *what kind of network is this*.
  Events for available/lost/capabilities/properties, plus synchronous
  `getDefaultNetSync` + `getNetCapabilitiesSync` to read the bearer type.
- **`@kit.NetworkBoostKit`'s `netQuality`** answers *how fast is it right now*.
  `on('netQosChange')` pushes a `NetworkQos` array with `linkDownRate` and
  `linkUpRate` in **bit/s**.

Both need only `ohos.permission.GET_NETWORK_INFO`, which is `system_grant` - no
runtime request, no dialog. The document says `INTERNET`, which is wrong
(`HW-04-0089`).

## Feature checklist

- A WiFi/cellular indicator that follows the current default network.
- A download speed readout that updates as the QoS callback fires, with a KB/s
  or MB/s unit chosen by magnitude.
- The indicator clears when the network is lost.
- All subscriptions are released when the page goes away.

## Architecture

One `entry` module, one page. There is no model layer beyond a list of static
row data.

```
entry/src/main/ets
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── model/ListInfo.ets        STUDENTITEM / TOOLITEM - static rows for the surrounding UI
└── pages/NetMonitor.ets      @Entry - both subscriptions and the whole layout
```

The documented tree matches the zip.

**Two subscriptions with completely different lifecycles, and that is the thing
to get right:**

| | `connection.NetConnection` | `netQuality` |
| --- | --- | --- |
| subscribe | `on(...)` per event, **then** `register(cb)` | `on('netQosChange', cb)` |
| release | `off(...)` per event, **then** `unregister(cb)` | `off('netQosChange')` |
| shape | instance you create and own | module-level, one subscription per process |

`NetConnection` is a two-step subscription: `on` attaches handlers, `register`
activates them. The order is not interchangeable - the reference says "call
**on** to enable listening and then call **register**" - and the sample does it
backwards (`HW-04-0085`).

**Event handlers give you the change; the synchronous getters give you the
value.** `netConnectionPropertiesChange` tells you *that* the network changed;
it does not carry the bearer type. To learn WiFi-versus-cellular you call
`getDefaultNetSync()` then `getNetCapabilitiesSync(handle)` and read
`bearerTypes`. That two-step - event as a trigger, sync getter as the read - is
the pattern worth copying.

**The sample's teardown covers one of the two subscriptions.**
`aboutToDisappear` calls `networkSpeedMonitorOff()` only; the `NetConnection`
registration, its four handlers and the clock timer are all left running
(`HW-04-0086`, `HW-04-0087`), and `onPageShow` re-creates every one of them on
each visit.

## Implementation steps

1. **Declare `ohos.permission.GET_NETWORK_INFO`** in `module.json5`. It is
   `system_grant`, so no `reason`, no `usedScene`, no runtime request.
2. **Create the connection once**, as a field:
   `connection.createNetConnection()`.
3. **Attach handlers first, then register.** `on('netAvailable')`,
   `on('netLost')`, `on('netCapabilitiesChange')`,
   `on('netConnectionPropertiesChange')`, and only then `register(cb)`.
4. **Do this in `aboutToAppear`, not `onPageShow`.** `onPageShow` fires on every
   return to the page.
5. **Read the bearer type through the sync getters** inside the property-change
   handler, and handle *both* branches - WiFi and cellular - so the indicator can
   turn off again (`HW-04-0090`).
6. **Subscribe to `netQosChange`** and convert from bit/s. Pick the unit by
   comparing against **1** MB/s, not 1024 (`HW-04-0088`).
7. **Store every timer id** and clear it in `aboutToDisappear`.
8. **Release everything symmetrically**: `off` each event, `unregister` the
   connection, `netQuality.off('netQosChange')`, `clearInterval`.

## Verified snippets

All snippets are from `MonitorNetwork.zip`. Corrected forms are marked.

**Subscribing to network state — `entry/src/main/ets/pages/NetMonitor.ets`** (corrected, see `HW-04-0085`, `HW-04-0086`, `HW-04-0090`)

```typescript
import { connection } from '@kit.NetworkKit';
import { netQuality } from '@kit.NetworkBoostKit';

netCon: connection.NetConnection = connection.createNetConnection();
@State isWifi: boolean = false;
private clockId: number = 0;                  // FIX: sample discards the interval handle

aboutToAppear(): void {                       // FIX: sample does all of this in onPageShow
  this.clockId = setInterval(() => { this.updateTime(); }, 1000);
  this.networkMonitor();
  this.networkSpeedMonitor();
}

aboutToDisappear(): void {
  clearInterval(this.clockId);                // FIX: absent in the sample
  this.networkMonitorOff();                   // FIX: absent in the sample
  this.networkSpeedMonitorOff();
}

networkMonitor() {
  // FIX: the sample calls register() FIRST. The reference requires on() then register().
  this.netCon.on('netAvailable', (data: connection.NetHandle) => {
    hilog.info(0x0000, 'net', `netAvailable: ${JSON.stringify(data)}`);
  });
  this.netCon.on('netLost', (data: connection.NetHandle) => {
    this.isWifi = false;
  });
  this.netCon.on('netCapabilitiesChange', (data: connection.NetCapabilityInfo) => {
    hilog.info(0x0000, 'net', `netCapabilityInfo: ${JSON.stringify(data)}`);
  });
  this.netCon.on('netConnectionPropertiesChange', (data: connection.NetConnectionPropertyInfo) => {
    this.networkTypes = this.getNetworkConnectionType()?.bearerTypes;
    if (this.networkTypes?.includes(connection.NetBearType.BEARER_WIFI)) {
      this.isWifi = true;
    } else if (this.networkTypes?.includes(connection.NetBearType.BEARER_CELLULAR)) {
      this.isWifi = false;                    // FIX: this branch is in the document, not the code
    }
  });

  this.netCon.register((err: BusinessError) => {
    if (err) {
      hilog.error(0x0000, 'net', `register failed: ${JSON.stringify(err)}`);
    }
  });
}

networkMonitorOff() {                         // FIX: no equivalent in the sample
  this.netCon.off('netAvailable');
  this.netCon.off('netLost');
  this.netCon.off('netCapabilitiesChange');
  this.netCon.off('netConnectionPropertiesChange');
  this.netCon.unregister(() => {});
}
```

**`on` then `register`, in that order.** The reference is explicit: *"To listen
for a specific type of events, call **on** to enable listening and then call
**register** to register an event listener."* `register` is what activates the
subscription; handlers attached after it are attached to an already-active
registration. The same note carries the other half of the contract: *"After
using the register API, you need to call unregister to deregister the
listener."*

**`onPageShow` is the wrong hook for a subscription.** It fires on every return
to the page, so the sample accumulates a registration, four handlers and a
one-second timer per visit. `aboutToAppear`/`aboutToDisappear` bracket the
component's life; `onPageShow`/`onPageHide` bracket its visibility.

**Reading the bearer type — same file** (as shipped)

```typescript
getNetworkConnectionType(): connection.NetCapabilities | null {
  let netCapability: connection.NetCapabilities | null = null;
  try {
    const netHandle = connection.getDefaultNetSync();
    if (!netHandle || netHandle.netId === 0) {
      return null;                       // netId 0 means "no default network"
    }
    netCapability = connection.getNetCapabilitiesSync(netHandle);
  } catch (e) {
    const err = e as BusinessError;
    hilog.error(0x0000, 'net', `errCode: ${err.code}, errMessage: ${err.message}`);
  }
  return netCapability ?? null;
}
```

**`netId === 0` is the sentinel for "no network".** `getDefaultNetSync` returns a
handle regardless; the id is how you tell whether it refers to anything. Skip
that check and `getNetCapabilitiesSync` throws on every offline call.

`bearerTypes` is an **array** - a network can carry more than one bearer - so the
test is `includes(...)`, not equality.

**Sampling the speed — same file** (corrected, see `HW-04-0088`)

```typescript
networkSpeedMonitor() {
  try {
    netQuality.on('netQosChange', (info: Array<netQuality.NetworkQos>) => {
      info.forEach((qos) => {
        // linkDownRate is bit/s (netQuality reference, NetworkQos.linkDownRate)
        const mbPerSec = qos.linkDownRate / 8 / 1e6;
        // FIX: the sample tests "mbPerSec <= 1024", which is true for every real
        //      network, so the MB/s branch is dead and 100 Mbit/s reads "12800KB/s"
        if (mbPerSec < 1) {
          this.networkSpeed = `${new Decimal(mbPerSec * 1000).toDecimalPlaces(1)}KB/s`;
        } else {
          this.networkSpeed = `${new Decimal(mbPerSec).toDecimalPlaces(1)}MB/s`;
        }
      });
    });
  } catch (err) {
    hilog.error(0x0000, 'netSpeed',
      `errorCode: ${(err as BusinessError).code}, errorMessage: ${(err as BusinessError).message}`);
  }
}

networkSpeedMonitorOff() {
  try {
    netQuality.off('netQosChange');
  } catch (err) {
    hilog.error(0x0000, 'netSpeed', `off failed: ${(err as BusinessError).code}`);
  }
}
```

**`linkDownRate` is bit/s** - the reference says so plainly, and the NetworkBoost
guide's own example divides the sum of up and down by 8 to get byte/s. So
bytes/s is `/8`, MB/s is `/8/1e6`. The threshold then has to be **1**, because
the value being compared is already in MB/s. Comparing it to 1024 asks whether
the link is under 8.6 Gbit/s.

**`netQuality.on/off` is module-level, not per-instance,** unlike
`NetConnection`. `off('netQosChange')` with no callback removes the
subscription for the whole process - which is correct here, but means two
components cannot each hold their own.

**Both APIs throw synchronously**, so a `try`/`catch` around `on` is the right
error handling - error 201 is "permission denied", 801 is "capability not
supported" on a device without NetworkBoost.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.GET_NETWORK_INFO" }
]
```

- `GET_NETWORK_INFO` is `system_grant`: no `reason`, no `usedScene`, no runtime
  request. It is required by **every** API this page calls -
  `NetConnection.register`, `getDefaultNetSync`, `getNetCapabilitiesSync` and
  `netQuality.on('netQosChange')` all list it.
- **The document names `ohos.permission.INTERNET` instead** (`HW-04-0089`). That
  permission governs opening sockets, which this sample never does; following the
  document produces error 201 on every call.
- `netQuality` is `SystemCapability.Communication.NetworkBoost.Core` and is
  documented for **Phone, PC/2in1 and Tablet** only - narrower than the rest of
  the page, and a missing capability surfaces as error 801.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `netQuality` needs API 12 or later and is unavailable on device types outside
  phone/tablet/2-in-1; guard for error 801 rather than assuming it exists.
- **The speed is what the system reports, not what the app measures.**
  `netQosChange` fires when the platform's estimate changes, at its own cadence -
  there is no polling interval to tune and no guarantee of an update within any
  given period.
- Only `linkDownRate` is used; `NetworkQos` also carries `linkUpRate` and
  latency fields the sample ignores.
- The page has a hard-coded clock and static list rows (`STUDENTITEM`,
  `TOOLITEM`) that exist only to give the indicator somewhere to live.
- `@kit.TelephonyKit` is imported for an unused type (`HW-04-0091`), so the
  sample declares a dependency it does not need.

## Pitfalls

- **`HW-04-0085` — `register()` is called before the `on()` handlers,** the
  reverse of what the reference requires. The document states this wrong order as
  the procedure, so it is not a transcription slip.
- **`HW-04-0086` — nothing is ever unregistered.** The reference note on
  `register` says an `unregister` is required; the sample has none, and
  `onPageShow` adds a fresh registration plus four handlers on every visit.
- **`HW-04-0087` — the clock `setInterval` handle is discarded,** so the timer
  can never be stopped and a new one starts on each page show.
- **`HW-04-0088` — the unit threshold compares MB/s against 1024,** making the
  MB/s branch unreachable. Every speed is rendered in KB/s, and the two
  conversions mix decimal and binary bases.
- **`HW-04-0089` — the document's 权限说明 names `INTERNET`** where the code and
  all four API references say `GET_NETWORK_INFO`.
- **`HW-04-0090` — the shipped handler has no cellular branch,** so once WiFi is
  seen the indicator latches on until the network is lost entirely. Here the
  document is right and the code is wrong.
- **`HW-04-0091` — Telephony Kit is imported for a field that is never read.**
- **Do not subscribe in `onPageShow`.** Use `aboutToAppear`/`aboutToDisappear`
  for anything that must be created once and released once.
- **Do not read the bearer type from the event payload.**
  `netConnectionPropertiesChange` is a trigger; the value comes from
  `getDefaultNetSync` + `getNetCapabilitiesSync`.

## References

- `documentation/harmonyos-references/03_system/js-apis-net-connection.md` - `createNetConnection`, the `on`-then-`register` order, the `unregister` requirement, `getDefaultNetSync`, `getNetCapabilitiesSync`, `NetBearType`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-net-connection
- `documentation/harmonyos-references/03_system/networkboost-netquality.md` - `on('netQosChange')`, `NetworkQos`, `linkDownRate` in bit/s, error codes 201/401/801
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/networkboost-netquality
- `documentation/harmonyos-guides/04_system/networkboost-qoscallback.md` - the worked QoS callback, including the `/8` conversion to byte/s
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/networkboost-qoscallback
- `documentation/harmonyos-guides/04_system/networkboost-netqualityguide.md` - when to use network quality information
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/networkboost-netqualityguide
- `documentation/harmonyos-guides/04_system/net-connection-manager.md` - `ohos.permission.GET_NETWORK_INFO` and the connection-manager flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/net-connection-manager
- `EDU-01` - the online video player whose playback this indicator is meant to inform
