---
id: SOCIAL-17
title: Bluetooth SPP chat - one RFCOMM UUID, both roles at once, and the socket lifecycle the sample forgets
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/17_bluetooth_spp.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/bluetooth_spp-0000002318232397
sample: huawei_industry_tree/14_social_communication/downloads/蓝牙通信示例代码.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.ConnectivityKit", "@kit.PerformanceAnalysisKit"]
apis: [abilityAccessCtrl, buffer, bundleManager, common, connection, hilog, socket, util, window]
permissions: [ohos.permission.ACCESS_BLUETOOTH]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0037, HW-14-0038, HW-14-0039, HW-14-0008, HW-14-0085]
status: verified-with-fixes
---

## When to use

Load this card when two devices must **exchange data over classic Bluetooth
with no network in between** - a phone-to-phone chat, a handset talking to a
peripheral, a debugging channel between a tablet and a companion device. SPP
(Serial Port Profile) is the RFCOMM stream socket: you agree on a UUID, one
side listens, the other connects, and after that it is a byte pipe in both
directions.

The pattern here is *symmetric*: every instance of the app both dials out and
listens, so whichever device the user taps first becomes the client and the
other one is already accepting. That is the right shape for peer-to-peer
sharing, and it generalises past chat - file handoff, remote control, sensor
streaming all use the same four calls (`sppConnect`, `sppListen` + `sppAccept`,
`sppWrite`, `on('sppRead')`).

**Read `HW-14-0037` before adopting it.** The sample opens sockets and
subscriptions and never closes any of them. Bluetooth sockets are a scarce
system resource held by the Bluetooth subsystem, not by your page, so a leak
here survives the page and outlives the user's patience. Treat this card's
corrected snippets as the shipping form, not the zip's.

## Feature checklist

- On launch, request `ohos.permission.ACCESS_BLUETOOTH`; if it is refused,
  toast and hide the switch.
- A 服务端监听 (server listening) toggle. Turning it on reveals the list of
  already-paired devices.
- Each row shows the paired device's MAC address and a chevron.
- Tapping a row pushes a chat page and starts an SPP connection to that MAC.
- The chat page simultaneously listens on the same UUID so the peer can dial in.
- Typed text is sent with `sppWrite`; received bytes are decoded and appended
  to the same list, right-aligned for sent and left-aligned for received.
- Leaving the chat page closes both sockets and removes the read listener.
  (**Not implemented in the sample** - see `HW-14-0037`.)

## Architecture

One `entry` module, two pages, two utility singletons, and a one-line
constants class.

```
entry/src/main/ets
├── common/Constants.ets           the single SPP UUID, shared by both roles
├── entryability/EntryAbility.ets  full-screen window, avoid areas -> AppStorage
├── model/MsgShow.ets              @Observed { message, flag } - flag = sent-by-me
├── pages/MainPage.ets             @Entry: permission, toggle, paired list, Navigation
├── pages/Communication.ets        NavDestination: connect + listen + read + write + bubbles
└── utils
    ├── PermissionsUtil.ets        check / request helper (broken - HW-14-0008)
    └── Util.ets                   ArrayBuffer <-> string conversions
```

The documented tree matches the zip exactly, including the absence of an
`entrybackupability` directory.

**The design decision worth copying** is that the UUID is a single constant
shared by both roles and both devices:

```typescript
export default class Constants {
  static readonly UUID: string = '00001810-0000-1000-8000-00805F9B34FB';
};
```

`sppListen` registers a service record under that UUID and `sppConnect`
resolves the peer's service record by the same UUID. Because it is one
constant in one file, the two halves of the handshake cannot drift apart. If
you fork this for a real product, change this value to your own generated UUID
- `00001810-…` is the assigned Blood Pressure service base and is being reused
here only as an arbitrary 128-bit token.

**The design decision worth avoiding** is arming the read listener from a
`@Watch`. `clientNumber` is written from two places - `sppConnect`'s callback
and `sppAccept`'s callback - so the watch is a neat way to say "whenever we
acquire a socket id, start reading from it, whichever role produced it". It is
also why the sample stacks duplicate `sppRead` subscriptions: the watch fires
again on the next id, and nothing ever unsubscribes the previous one.

## Implementation steps

1. **Declare `ohos.permission.ACCESS_BLUETOOTH`** in `module.json5` with
   `reason` and `usedScene`. It is `user_grant`, so it must also be requested
   at runtime.
2. **Request it in `aboutToAppear` and re-check in `onPageShow`,** so a grant
   made in Settings while the app was backgrounded is picked up on return.
3. **Accumulate the permission check across the array; never return from
   inside the first iteration** (`HW-14-0008`). Both helpers in
   `PermissionsUtil` get this wrong in different ways.
4. **List paired devices with `connection.getPairedDevices()` inside the try
   block** - it is the only call in that method that can throw (`HW-14-0039`).
5. **Give both roles the same `SppOptions`**: `{ uuid, secure: false, type: 0 }`,
   where `type: 0` is `SppType.SPP_RFCOMM`. `secure: false` selects an
   insecure RFCOMM socket - no man-in-the-middle protection - which is
   acceptable for a demo between already-paired devices and is not acceptable
   for anything carrying user content.
6. **Start `sppAccept` from inside the `sppListen` callback,** not beside it:
   the server socket id does not exist until the listen callback fires.
7. **Remove the previous `sppRead` listener before registering the next one**
   (`HW-14-0037`).
8. **Decode with the exact encoding name `'utf-8'`** - the sample's literal
   carries embedded double quotes and throws (`HW-14-0038`).
9. **Close both sockets and unsubscribe in `onHidden`** - `sppCloseClientSocket`
   for the connection, `sppCloseServerSocket` for the listener
   (`HW-14-0037`). The reference is explicit that the server must close its
   socket so the Bluetooth subsystem deletes the registered service record.

## Verified snippets

All snippets are from `蓝牙通信示例代码.zip` (`BluetoothSPP`). Corrected forms
are marked.

**Both roles from one `onReady` — `entry/src/main/ets/pages/Communication.ets`** (as shipped)

```typescript
@State serverNumber: number = -1;
@State @Watch('onSppRead') clientNumber: number = -1;

sppConnect() {
  let clientSocket = (code: BusinessError, number: number) => {
    if (code) {
      hilog.error(0x0000, 'sppListen error, code is ', `${code}`);
      return;
    } else {
      // 获取的clientNumber用作客户端后续读/写操作socket的id。
      this.clientNumber = number;
      this.connectSwitch = true;
    }
  };
  let sppOption: socket.SppOptions = { uuid: Constants.UUID, secure: false, type: 0 };
  try {
    socket.sppConnect(this.mac, sppOption, clientSocket);
  } catch (err) {
    hilog.error(0x0000, 'errCode: ', (err as BusinessError).code.toString());
  }
}

sppListen() {
  let sppOption: socket.SppOptions = { uuid: Constants.UUID, secure: false, type: 0 };
  try {
    socket.sppListen('BT_SOCKET', sppOption, (code: BusinessError, number: number) => {
      if (code) {
        return;
      } else {
        this.serverNumber = number;
        this.socketListen = true;
        this.sppAccept();            // the server id only exists here
      }
    });
  } catch (err) {
    hilog.error(0x0000, 'errCode: ', (err as BusinessError).code.toString());
  }
}

// ...
.onReady((context: NavDestinationContext) => {
  this.pageInfos = context.pathStack;
  this.mac = context.pathInfo.param as string;
  this.sppConnect();                 // dial the peer we tapped
  this.sppListen();                  // and accept if the peer dials us
});
```

**Three things carry the design.** The MAC address arrives as
`context.pathInfo.param`, so the chat page is addressed purely by the route -
no shared state between the two pages. `sppAccept` is nested inside the
`sppListen` callback because the server socket id is produced by that
callback; calling it from `onReady` would find `serverNumber === -1` and toast
instead. And both roles are started unconditionally, which is what makes the
demo work no matter which of the two devices the user taps first.

Note the copy-paste in `sppConnect`: its error log says `sppListen error`.
Harmless, but it tells you the two methods were written from one template.

**Arming and disarming the read listener — same file** (corrected, see `HW-14-0037`)

```typescript
private readArmed: number = -1;

/**
 * 订阅spp读请求事件,需要在客户端连接后打开监听
 */
onSppRead() {
  if (this.clientNumber === -1) {
    this.getUIContext().getPromptAction().showToast({ message: $r('app.string.Connection_status') });
    return;
  }

  try {
    if (this.readArmed !== -1) {
      socket.off('sppRead', this.readArmed);   // FIX: absent in the sample
    }
    socket.on('sppRead', this.clientNumber, (dataBuffer: ArrayBuffer) => {
      let data = Utils.ArrayBuffer2String(dataBuffer);
      this.msgList.push(new MsgShow(data, false));
    });
    this.readArmed = this.clientNumber;
    this.msgReadSwitch = true;
  } catch (err) {
    hilog.error(0x0000, 'errCode: ', (err as BusinessError).code.toString());
  }
}

// FIX: the sample has no teardown of any kind
onHidden(): void {
  if (this.readArmed !== -1) {
    socket.off('sppRead', this.readArmed);
    this.readArmed = -1;
  }
  if (this.clientNumber !== -1) {
    socket.sppCloseClientSocket(this.clientNumber);
    this.clientNumber = -1;
  }
  if (this.serverNumber !== -1) {
    socket.sppCloseServerSocket(this.serverNumber);
    this.serverNumber = -1;
  }
}
```

**`@Watch` fires per assignment, `socket.on` accumulates per call.** Those two
facts together are the whole defect. `clientNumber` is assigned by
`sppConnect`'s callback and again by `sppAccept`'s callback, and it will be
assigned once more on every reconnect; each assignment re-enters `onSppRead`
and adds another subscription to the same event, so one inbound message is
pushed into `msgList` two, three, N times. Tracking the id that is currently
armed and `off`-ing it first collapses that to exactly one listener.

The teardown belongs in `onHidden` rather than `aboutToDisappear` because
`NavDestination` components are not necessarily destroyed when popped. The
reference is unambiguous about why it matters on the server side: if the
server socket is not closed, the Bluetooth subsystem keeps the service record
registered, and a peer that connects to it afterwards fails.

**The receive-path decoder — `entry/src/main/ets/utils/Util.ets`** (corrected, see `HW-14-0038`)

```typescript
import { buffer, util } from '@kit.ArkTS';

class Utils {
  string2ArrayBuffer(value: string) {
    let buf = buffer.from(value).buffer;
    return buf;
  }

  ArrayBuffer2String(buf: ArrayBuffer) {
    const decoder = util.TextDecoder.create('utf-8');   // FIX: sample passes '"utf-8"'
    const str = decoder.decodeToString(new Uint8Array(buf));
    return str;
  }
}

export default new Utils();
```

The shipped literal is nine characters long - `"utf-8"` *including* the double
quotes - and `TextDecoder.create` rejects it as an unknown encoding. Because
`ArrayBuffer2String` is called from inside the `sppRead` callback with no
`try`/`catch` around it, that throw escapes the callback rather than showing
up as a failed message. Every inbound message on the sample's receive path
dies there.

The send direction is fine and worth noting for its asymmetry: `buffer.from`
handles the encode, so only the decode side needed a hand-written helper.
`Util.ets` also carries three unused conversion helpers (`string2Buffer`,
`buffer2String`, `string2ArrayBuffer2`, `charString2ArrayBuffer`) - dead code
inherited from a larger template.

**Paired devices and the permission helper — `MainPage.ets` / `PermissionsUtil.ets`** (corrected, see `HW-14-0039`, `HW-14-0008`)

```typescript
// MainPage.ets
getPairedDevices(): string[] {
  try {
    let devices: Array<string> = connection.getPairedDevices();   // FIX: was above the try
    for (const device of devices) {
      hilog.info(0x0000, 'getPairState: ', JSON.stringify(device)); // FIX: sample logs devices[0]
    }
    return devices;
  } catch (err) {
    hilog.error(0x0000, 'errCode: ', `${(err as BusinessError).code}`);
    return [];
  }
}

// PermissionsUtil.ets
async checkPermissions(permissions: Array<Permissions>): Promise<boolean> {
  for (let permission of permissions) {                            // FIX: sample overwrites
    let grantStatus = await this.checkAccessToken(permission);     //      applyResult per pass
    if (grantStatus !== abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED) {
      return false;
    }
  }
  return true;
}

async requestPermission(permissions: Permissions[], context: common.UIAbilityContext): Promise<boolean> {
  let atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
  let data = await atManager.requestPermissionsFromUser(context, permissions);
  let grantStatus: Array<number> = data.authResults;
  for (let i = 0; i < grantStatus.length; i++) {
    if (grantStatus[i] !== 0) {
      return false;                                                // FIX: sample also returns
    }                                                              //      true from inside the
  }                                                                //      first iteration
  return true;
}
```

**`connection.getPairedDevices()` is the only statement in that method that
can throw** - `201` when the permission is missing and `2900003` when
Bluetooth is switched off - and in the shipped code it sits one line above the
`try`. The `try` instead wraps a loop that cannot throw, and whose body
ignores its own loop variable to re-read `devices[0]` on every pass. Flipping
a Bluetooth-off device's 服务端监听 toggle therefore throws out of `onChange`
rather than landing in the catch that was written for it.

The permission helpers are the second instance of a defect that also affects
`SOCIAL-03`'s `VoiceToTextForChatDemo` (`HW-14-0008`): a loop over an array of
permissions that decides the whole result from element zero. With a
single-element array like this one it happens to behave; copy either helper
into a feature that needs two permissions and it reports the wrong answer on
any partial refusal.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.ACCESS_BLUETOOTH",
    "reason": "$string:EntryAbility_desc",
    "usedScene": {
      "abilities": ["EntryAbility"],
      "when": "always"
    }
  }
]
```

- `ACCESS_BLUETOOTH` is `user_grant`, so declaring it is not enough - the
  runtime request in `aboutToAppear` is mandatory.
- `reason` points at `EntryAbility_desc`, the generic module description. A
  real app should carry a purpose-specific string: this is the text the user
  reads in the grant dialog.
- `when: "always"` is declared although the sample has no background task and
  no continuous task; `"inuse"` would be the honest value for a foreground-only
  chat page.
- `deviceTypes` are `phone`, `tablet`, `2in1`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Two physical devices are required.** There is no discovery UI - the list
  is `connection.getPairedDevices()`, so the peer must already be paired in
  system Bluetooth settings. The doc suggests unpairing everything else first,
  because the list shows raw MAC addresses with no friendly names.
- `secure: false` selects an insecure RFCOMM socket. Anything shipping user
  content should use `secure: true` and accept the pairing prompt.
- The sample never reacts to the peer disconnecting: there is no error
  subscription and no `socket.isConnected` poll (available from API 22), so a
  peer that walks out of range leaves the UI looking connected.
- `msgList` is in-memory only. Nothing is persisted across page exits.

## Pitfalls

- **`HW-14-0037` (B/high, confirmed) — no socket is ever closed and no
  listener is ever removed,** and `@Watch('onSppRead')` adds a duplicate
  `sppRead` subscription on every `clientNumber` change. Backing out of the
  chat leaves the client socket, the listening server socket and the
  subscription alive; re-entering stacks more, so messages arrive duplicated.
  Fix: track the armed id, `socket.off` before re-subscribing, and close both
  sockets in `onHidden`.
- **`HW-14-0038` (B/medium, confirmed) — `TextDecoder.create('"utf-8"')`
  carries the quotes inside the encoding name.** It is not a valid encoding,
  and the call sits in the `sppRead` callback with no `try`/`catch`, so every
  inbound message throws. Fix: `'utf-8'`.
- **`HW-14-0039` (B/medium, confirmed) — `connection.getPairedDevices()` is
  called outside its own `try`.** Error 201 (no permission) and 2900003
  (Bluetooth off) escape `onChange` instead of reaching the catch. The `try`
  currently wraps a loop that cannot throw and ignores its loop variable. Fix:
  move the call inside.
- **`HW-14-0008` (B/low, confirmed, systematic) — permission helpers decide
  from the first array element.** `requestPermission` returns from inside
  iteration zero; `checkPermissions` overwrites `applyResult` each pass so the
  *last* permission decides. `BluetoothSPP` is the second instance of this
  shape in the industry, after `SOCIAL-03`'s `VoiceToTextForChatDemo`. Fix:
  return `false` in-loop and `true` after it.
- **Not filed, but worth knowing:** the paired-device rows render the raw MAC
  string, and the whole `Navigation` has an `onClick` that duplicates the
  toggle row's permission-on-setting handler verbatim - two copies of the same
  15 lines in one file.

## References

- `huawei_industry_tree/14_social_communication/docs/17_bluetooth_spp.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bluetooth_spp-0000002318232397
- `documentation/harmonyos-references/03_system/js-apis-bluetooth-socket.md` - `sppConnect`, `sppListen`, `sppAccept`, `sppWrite`, `on/off('sppRead')`, `sppCloseClientSocket`, `sppCloseServerSocket`, `SppOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-bluetooth-socket
- `documentation/harmonyos-references/03_system/js-apis-bluetooth-connection.md` - `getPairedDevices` and its error codes
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-bluetooth-connection
- `documentation/harmonyos-guides/04_system/bluetooth-br.md` - the classic-Bluetooth (BR/EDR) connection flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/bluetooth-br
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` - `checkAccessToken`, `requestPermissionsFromUser`, `requestPermissionOnSetting`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` - the `user_grant` request flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `documentation/harmonyos-references/02_application-framework/js-apis-util.md` - `TextDecoder.create` and its accepted encoding names
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-util
- `SOCIAL-03` - the other instance of the first-element permission loop (`HW-14-0008`)
