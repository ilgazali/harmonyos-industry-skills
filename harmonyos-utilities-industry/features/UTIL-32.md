---
id: UTIL-32
title: Live IP address - read linkAddresses from the default net and refresh on connection events
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/32_get_ip_address.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/get_ip_address-0000002482813589
sample: huawei_industry_tree/15_utilities/downloads/GetIPAddress.zip
kits: ["@kit.NetworkKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.AbilityKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: ["connection.createNetConnection", "NetConnection.register", "NetConnection.unregister", "NetConnection.on", "connection.getDefaultNet", "connection.getDefaultNetSync", "connection.getConnectionPropertiesSync", ConnectionProperties, LinkAddress, NetHandle, Canvas, CanvasRenderingContext2D, createLinearGradient, "display.getDefaultDisplaySync", px2vp, Tabs, onContentWillChange, "@StorageProp", showToast]
permissions: ["ohos.permission.GET_NETWORK_INFO"]
min_api: 20
modules: [entry]
findings: [HW-15-0071, HW-15-0101, HW-15-0102]
status: verified-with-fixes
---

## When to use

**Load this card when a screen has to show the device's current IP address and
keep it true as the network moves underneath it** - a speed-test page, a
diagnostics screen, a LAN pairing flow that tells the user which address to
type on the other device, a support panel that captures the connection state
with a bug report.

The mechanism is two-sided and that is the whole lesson. Reading the address is
one synchronous call - `getConnectionPropertiesSync(netHandle).linkAddresses` -
but a read alone gives you a value that is stale the moment Wi-Fi drops.
Keeping it true needs a `NetConnection` subscription: `on('netLost')` and
`on('netConnectionPropertiesChange')`, both re-running the same read. The
pattern generalises to anything network-derived that a user looks at for more
than a second: carrier name, connection type, metered status, signal.

**The trap the sample falls into is that the refresh path is the fragile
path.** During exactly the transitions that fire those events, there may be no
default network and `linkAddresses` may be empty - so the code that runs most
often is the code with the least guarding (`HW-15-0071`).

## Feature checklist

- A 测速 (speed test) page as the first tab, with the page title at the top.
- A circular gauge drawn on a `Canvas` with a vertical blue linear gradient,
  horizontally centred against the real window width.
- The gauge label is tappable and raises a 此功能仅供展示 (this feature is for
  display only) toast - the speed test itself is not implemented.
- A card at the bottom of the page shows the current IP address on its first
  line and a carrier label under it.
- The address appears on entry and updates when the network connection
  properties change or the network is lost.
- With no connected network the address line goes blank rather than showing a
  stale value.
- A bottom tab bar with 测速 and 我的 (mine); switching is deliberately vetoed
  and raises the same "display only" toast.
- The subscription is torn down when the page disappears.

## Architecture

One `entry` module, two pages, and no model layer - the feature is one string.

```
entry/src/main/ets
├── constants/StyleConstants.ets    the three Chinese UI literals
├── entryability/EntryAbility.ets   full-screen window, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── pages
│   ├── InternetSpeedPage.ets       the whole feature: canvas, IP card, subscription
│   └── TabsPage.ets                @Entry, the two-tab shell
└── utils/Logger.ets
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that `InternetSpeedPage` owns its
`NetConnection` as a private field and tears it down in `aboutToDisappear`:

```typescript
private netCon: connection.NetConnection = connection.createNetConnection();
```

The subscription's lifetime is the component's lifetime, expressed by where the
object lives. There is no manager, no singleton, and no way for the handle to
outlive the UI that uses it - which is the failure mode this kind of code
usually has. Compare `TOUR-01` and `TOUR-03` in the tourism pack, where an
`avoidAreaChange` listener registered in the ability is never released at all.

`connection.createNetConnection()` with no argument means "the default
network". Passing a `NetSpecifier` would scope the subscription to a particular
capability - Wi-Fi only, or unmetered only - which is what you want if the
screen is about one kind of connection rather than whatever is current.

**The tab bar's veto is intentional here**, unlike the identical-looking defect
in `TOUR-03`:

```typescript
.onContentWillChange(() => {
  this.promptAction.showToast({ message: StyleConstants.ONLY_DISPLAY });
  return false;
})
```

The second tab's `TabContent` is empty, `scrollable(false)` is set, and the
toast says so in words. Returning `false` from `onContentWillChange` vetoes the
switch; used deliberately with a message it is a legitimate "not built yet"
placeholder, used accidentally it kills the navigation. Know which one you are
writing.

## Implementation steps

1. **Create the `NetConnection` as a component field**, so its lifetime is the
   page's.
2. **Subscribe with `on(...)` before calling `register()`** (`HW-15-0071`).
   The reference for `register` is explicit: "To listen for a specific type of
   events, call `on` to enable listening and then call `register` to register
   an event listener." The sample does it the other way round and can miss the
   first event.
3. **Read once on entry** so the address is present before any event fires.
4. **Treat `netId === 0` as "no network"** and clear the displayed address.
   That is the documented signal, not an error.
5. **Guard `linkAddresses` before indexing it** (`HW-15-0071`). It is a legal
   empty array during a transition, and `linkAddresses[0].address` on an empty
   array is a `TypeError` inside a promise - an unhandled rejection that
   silently stops the display from ever updating again.
6. **Attach a `.catch` to `getDefaultNet()`** (`HW-15-0071`). It rejects when
   there is no default network, which is precisely the state `netLost`
   announces.
7. **Pick the address family deliberately.** On a dual-stack network
   `linkAddresses[0]` may be the IPv6 entry; filter by `family` if the screen
   promises an IPv4 address.
8. **Unregister in `aboutToDisappear`.** `unregister` removes every `on`
   handler attached to that connection.

## Verified snippets

All snippets are from `GetIPAddress.zip`. Corrected forms are marked.

**The subscription — `entry/src/main/ets/pages/InternetSpeedPage.ets`** (corrected, see `HW-15-0071`)

```typescript
private netCon: connection.NetConnection = connection.createNetConnection();

aboutToAppear() {
  this.windowWidth = this.uiContext.px2vp(this.display.width);
  this.circleCenterX = this.windowWidth / 2;

  this.getConnectionIPAddress();

  // FIX: the sample calls register() first; the reference requires on() before register()
  // 订阅网络连接信息变化事件
  this.netCon.on('netConnectionPropertiesChange', (data: connection.NetConnectionPropertyInfo) => {
    this.getConnectionIPAddress();
    Logger.info('Network connection changes: ' + JSON.stringify(data));
  });

  // 订阅网络丢失事件
  this.netCon.on('netLost', (data: connection.NetHandle) => {
    this.getConnectionIPAddress();
    Logger.info('Network loss:' + JSON.stringify(data));
  });

  // 注册网络状态变化事件
  this.netCon.register((error: BusinessError) => {
    if (error) {
      Logger.error('Network registration failed: ' + JSON.stringify(error));
    }
  });
}

aboutToDisappear(): void {
  // 取消订阅
  this.netCon.unregister((error: BusinessError) => {
    if (error) {
      Logger.error('Network unregistration failed: ' + JSON.stringify(error));
    }
  });
}
```

**`register` is the switch, `on` is the wiring.** Handlers attached with `on`
are inert until `register` activates the connection; `register` starts
delivering events immediately, including the initial state snapshot that the
platform pushes when the listener is added. Registering first therefore opens a
window - short, but real - in which an event is delivered to a connection with
no handlers on it. The reference pages disagree with each other on this
(`on('netLost')` says the opposite), but `register`'s own description gives the
required order and it is the one that costs nothing to follow.

Both handlers do the same thing: re-read. That is the right shape - there is
one function that computes the displayed value from current system state, and
every event just asks for it again. `netConnectionPropertiesChange` covers a
DHCP lease change or a Wi-Fi-to-cellular handover; `netLost` covers the
disconnect. Neither callback tries to derive the address from its own payload.

**The read — same file** (corrected, see `HW-15-0071`)

```typescript
getConnectionIPAddress() {
  connection.getDefaultNet().then((netHandle: connection.NetHandle) => {
    if (netHandle.netId === 0) {
      // 当前没有已连接的网络时，获取的netHandler的netid为0
      Logger.error('no connected networks');
      this.ipAddress = '';
      return;
    }
    let properties: connection.ConnectionProperties =
      connection.getConnectionPropertiesSync(netHandle);
    let addresses = properties.linkAddresses;
    if (!addresses || addresses.length === 0) {     // FIX: sample indexes [0] blindly
      this.ipAddress = '';
      return;
    }
    let v4 = addresses.find((item) => item.address.family === 1);   // FIX: dual-stack
    this.ipAddress = (v4 ?? addresses[0]).address.address;
  }).catch((err: BusinessError) => {                // FIX: absent in the sample
    Logger.error(`getDefaultNet failed: ${err.code} ${err.message}`);
    this.ipAddress = '';
  });
}
```

**Three guards, and each one covers a state the events actually produce.** The
`netId === 0` check is the documented "no connected network" signal and the
sample has it right. The empty-`linkAddresses` check covers the moment between
association and address assignment - a network exists, it has no address yet -
which is exactly when `netConnectionPropertiesChange` fires. The `.catch`
covers `getDefaultNet` rejecting outright, which is what `netLost` leads to.

Without them the failure is invisible in the worst way: the `TypeError` from
`addresses[0].address` is thrown inside a `then` callback, so it becomes an
unhandled rejection rather than a crash, and the page keeps rendering the last
good address forever. Nothing on screen says the value stopped being true.

The shipped version also reassigns the handle it was just given -
`netHandle = connection.getDefaultNetSync();` - before reading properties. It
is harmless but redundant: the promise already delivered a valid handle, and
re-fetching it synchronously inside the callback only widens the race. The
corrected form drops it and uses the handle it was handed.

`family === 1` is `AF_INET`; on a dual-stack Wi-Fi network the IPv6 entry can
sort first, so a screen that promises "your IP address" to a user reading it
aloud should prefer the v4 one and fall back to whatever exists.

**The gauge — same file** (as shipped)

```typescript
private settings: RenderingContextSettings = new RenderingContextSettings(true);
private context: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings);
private display: display.Display = display.getDefaultDisplaySync();

// in aboutToAppear:
this.windowWidth = this.uiContext.px2vp(this.display.width);
this.circleCenterX = this.windowWidth / 2;

Canvas(this.context)
  .width('100%')
  .height('100%')
  .onReady(() => {
    this.context.beginPath();
    this.context.arc(this.circleCenterX, 81, 81, 0, 6.28);
    this.context.stroke();
    let grad = this.context.createLinearGradient(this.circleCenterX, 0, this.circleCenterX, 162);
    grad.addColorStop(0.0, 'rgba(46, 119, 244, 0.79)');
    grad.addColorStop(1.0, 'rgb(21, 92, 216)');
    this.context.fillStyle = grad;
    this.context.fill();
  });
```

**`onReady` is the only correct place to draw.** It fires once the canvas is
laid out and has a backing surface; drawing from `aboutToAppear` targets a
context with no size. `RenderingContextSettings(true)` turns on antialiasing,
which a 162 vp circle needs badly.

The centre X comes from `px2vp(display.width)`, not from a literal, so the
circle stays centred on any phone. That is the right instinct executed one
level too high: `display.width` is the *screen*, not the component, so in a
floating or split-screen window the circle drifts off centre. Reading the
component's own width via `onAreaChange` or a `size` callback would be
window-safe.

`arc(x, y, r, 0, 6.28)` is a full circle written as an approximation of 2π -
6.28 leaves a hairline gap that the `fill()` closes anyway. `Math.PI * 2` is
the honest form.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.GET_NETWORK_INFO",
    "reason": "$string:EntryAbility_desc",
    "usedScene": {
      "abilities": ["EntryAbility"],
      "when": "inuse"
    }
  }
]
```

- `GET_NETWORK_INFO` is `system_grant`: it is granted at install time and no
  dialog is ever shown. The official guides declare it as a bare `{ "name": ... }`,
  so the `reason` and `usedScene` here are surplus - harmless, but they suggest
  a runtime prompt that does not exist.
- It is required by `register`, `getDefaultNet` and
  `getConnectionPropertiesSync` alike; without it those calls fail with 201.
- `ohos.permission.INTERNET` is **not** declared and is not needed - reading
  connection metadata is not network traffic.
- `deviceTypes` includes `"wearable"` alongside phone, tablet and 2in1, but the
  layout is built from phone-sized literals (a 162 vp gauge, 26 vp title) and
  has no watch adaptation.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The address shown is the default network's**, not "the device's". A device
  on Wi-Fi and cellular at once has more than one; `getDefaultNet` picks the
  one the system routes through.
- `linkAddresses` can hold several entries per network (IPv4 plus one or more
  IPv6). Index 0 has no defined meaning.
- The speed gauge is decorative. There is no measurement anywhere in the
  sample - the label, the arrow and the second tab all raise the same
  此功能仅供展示 toast.
- The carrier line under the IP address is the static resource
  `app.string.china_mobile`; it is not read from the SIM.
- `aboutToDisappear` unregisters, but the page is the first tab of a
  non-scrollable `Tabs` whose switching is vetoed, so in practice it is only
  reached when the page itself is destroyed.

## Pitfalls

- **`HW-15-0071`** (B/medium, confirmed): the refresh path is unguarded exactly
  where it runs most. `getConnectionIPAddress` has no `.catch` on
  `getDefaultNet` and dereferences `linkAddresses[0].address.address` blindly
  (`InternetSpeedPage.ets:75-88`), while both callers are the `netLost` and
  `netConnectionPropertiesChange` handlers - the transitions during which the
  array can be empty and the promise can reject. The resulting `TypeError`
  becomes an unhandled rejection and the display silently freezes on the last
  value. On dual-stack networks index 0 may also be the IPv6 entry.
  Additionally, `register()` is called before the two `on()` handlers
  (`:46-57`), against the ordering `register`'s own reference prescribes, so
  the initial snapshot event can be missed. Fix: guard the array length, add a
  `.catch`, prefer the IPv4 entry, and move `register()` after the `on()`
  calls.

## References

- `documentation/harmonyos-references/03_system/js-apis-net-connection.md` -
  `createNetConnection`, `register`/`unregister` and their required ordering,
  `on('netLost')`, `on('netConnectionPropertiesChange')`,
  `getDefaultNet`, `getConnectionPropertiesSync`, `ConnectionProperties`,
  `LinkAddress`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-net-connection
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` -
  `onContentWillChange` and what returning `false` means
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-guides/04_system/networkboost-preparations.md` -
  `ohos.permission.GET_NETWORK_INFO` declared without `reason`/`usedScene`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/networkboost-preparations
- `UTIL-31` - the other network sample in this pack; `INTERNET` versus
  `GET_NETWORK_INFO` and why they are unrelated
- `TOUR-03` - the same `onContentWillChange` construct used by accident
