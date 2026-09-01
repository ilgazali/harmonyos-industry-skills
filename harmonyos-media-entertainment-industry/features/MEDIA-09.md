---
id: MEDIA-09
title: Cellular switch reminder - pause the video and ask before spending mobile data
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/09_data_network_pause_playback.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_network_pause_playback-0000002291777477
sample: huawei_industry_tree/13_media_entertainment/downloads/DataNetworkPausePlayback.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.NetworkKit", "@kit.PerformanceAnalysisKit"]
apis: [connection, "connection.createNetConnection", register, unregister, "on('netLost')", "on('netAvailable')", "connection.getDefaultNet", "connection.getNetCapabilitiesSync", NetBearType, hilog, Video, VideoController, "UIContext.showAlertDialog", window]
permissions: [ohos.permission.GET_NETWORK_INFO]
min_api: 20
modules: [entry (entry)]
findings: [HW-13-0027, HW-13-0028, HW-13-0099]
status: verified-with-fixes
---

## When to use

Load this card when playback **costs the user money the moment the network
changes underneath it**. The pattern: subscribe to network state for the
lifetime of the player page, and when the active bearer becomes cellular,
pause immediately and ask before resuming.

It is one page of code, and the shape generalises past video. Any activity
with an open, unbounded byte stream deserves the same treatment - a podcast
that is buffering ahead, a large download, an upload of camera roll footage,
a live stream. What makes it a pattern rather than a toast is the *pause
first, ask second* order: the user is asked while nothing is being spent, and
the answer is a real branch (`controller.start()` or stay paused), not an
acknowledgement.

**Read `HW-13-0027` before adopting the sample's listener verbatim.** The
event handling is written around one specific event ordering and the callback
inspects the process-wide default network rather than the handle that was
just delivered to it. The corrected form below is barely longer and does not
depend on ordering at all.

## Feature checklist

- A full-screen, landscape-locked video that autoplays and loops from
  `rawfile`.
- Tapping the video toggles pause/play.
- Registering for network-state notifications when the player page appears,
  and deregistering when it disappears.
- When the active network becomes a cellular one, playback pauses without
  the user doing anything.
- A modal dialog explains the switch and offers two buttons: cancel (stay
  paused) and confirm (resume on mobile data).
- The dialog is not auto-cancellable - a tap outside must not dismiss the
  decision.

## Architecture

One `entry` module, one page, one utility. There is no player abstraction:
the ArkUI `Video` component and its `VideoController` are the player.

```
entry/src/main/ets
├── entryability/EntryAbility.ets        full-screen window, status/navigation bars hidden
├── entrybackupability/EntryBackupAbility.ets
├── pages/HomePage.ets                   @Entry, the Video + the dialog + the callback
└── utils/NetCheck.ets                   the singleton NetConnection wrapper (On/Off)
```

The documented tree matches the zip; unlike the trees called out in
`HW-13-0003` for this industry, the names and cases here all resolve.

**The design decision worth copying** is that `NetCheck` exports an
*instance*, not a class:

```typescript
class NetCheck { private static instance: NetCheck = new NetCheck(); /* ... */ }
export default NetCheck.getInstance();
```

A `NetConnection` is an OS-level subscription with a matching
`unregister` obligation. Handing callers a singleton with exactly two methods
- `On(callback)` and `Off()` - makes the pairing hard to get wrong: the page
calls `On` in `aboutToAppear` and `Off` in `aboutToDisappear` and never sees a
`NetConnection` object. The one thing the singleton then owes you is
idempotence, which this one does not have: calling `On` twice overwrites
`this.netCon` and leaks the first subscription. With a single player page
that never happens; with two, it does.

Note also the module config: `supportWindowMode: ["fullscreen"]` and
`orientation: "auto_rotation_landscape"` are declared on the ability, not
managed in code. For a video player that is the right place for them.

## Implementation steps

1. **Declare `ohos.permission.GET_NETWORK_INFO`** in `module.json5`, with
   `usedScene.abilities` naming an ability that actually exists in the module
   (`HW-13-0028`).
2. **Create the connection and subscribe the events before registering.** The
   reference is explicit: "To listen for a specific type of events, call `on`
   to enable listening and then call `register`". The sample calls `register`
   first and its handlers still fire, but the prescribed order costs nothing.
3. **Handle `netAvailable` on its own merits**: read the capabilities of the
   `NetHandle` the callback was handed and act if its bearer is
   `BEARER_CELLULAR` (`0`). Do not gate it on a flag set by a previous
   `netLost` (`HW-13-0027`).
4. **Never inspect `getDefaultNet()` inside the handler** - by the time that
   promise resolves the default network may be a third one, and the answer has
   nothing to do with the event being handled (`HW-13-0027`).
5. **Pause the controller before showing anything.** The dialog is a
   confirmation to resume, not a warning issued while bytes keep flowing.
6. **Set `autoCancel: false`** on the dialog so the decision cannot be
   dismissed by tapping outside, and give the resume button
   `DialogButtonStyle.HIGHLIGHT`.
7. **Mirror playback into a `videoState` boolean** so the tap-to-toggle
   handler and the dialog's two buttons agree on what the player is doing.
8. **Call `Off()` from `aboutToDisappear`** - `register` and `unregister` are
   a pair, and the subscription outlives the page otherwise.

## Verified snippets

All snippets are from `DataNetworkPausePlayback.zip`. Corrected forms are marked.

**The network listener - `entry/src/main/ets/utils/NetCheck.ets`** (as shipped)

```typescript
import { BusinessError } from '@kit.BasicServicesKit';
import { connection } from '@kit.NetworkKit';
import { hilog } from '@kit.PerformanceAnalysisKit';

class NetCheck {
  private static instance: NetCheck = new NetCheck();
  private netCon: connection.NetConnection | undefined
  private signal = 0

  On(callback: Callback<boolean>) {
    try {
      // 注册监听
      this.netCon = connection.createNetConnection();
      this.netCon.register((error: BusinessError) => {
        hilog.info(0x0000, 'testTag', 'load the content: %{public}s', JSON.stringify(error));
      });

      // 监听网络丢失事件，标记为切换网络
      this.netCon.on('netLost', (data: connection.NetHandle) => {
        connection.getDefaultNet().then((netHandle: connection.NetHandle) => {
          hilog.info(0x0000, 'testTag', 'netLost to get data: %{public}s', JSON.stringify(data));
        });
        this.signal = 1;
      });

      // 监听网络重连，确认重连的为数据网络则回调
      this.netCon.on('netAvailable', (data: connection.NetHandle) => {
        if (this.signal) {
          connection.getDefaultNet().then((netHandle: connection.NetHandle) => {
            let getNetCapabilitiesSync = connection.getNetCapabilitiesSync(netHandle);
            if (getNetCapabilitiesSync.bearerTypes[0] === 0) {   // 0 === BEARER_CELLULAR
              callback(true)
            }
          });
        }
        this.signal = 0;
      });
    } catch (err) {
      hilog.error(0x0000, 'testTag', 'errCode: %{public}s, errMessage: %{public}s',
        (err as BusinessError).code, (err as BusinessError).message);
    }
  }
}
```

**`this.signal` is the whole design, and it is the whole problem.** The
listener is a two-event state machine: `netLost` raises a flag meaning "a
switch is in progress", `netAvailable` consumes it. The reference does
document the ordering the sample banks on - "when the network transitions from
Wi-Fi to cellular, a **netLost** event is first triggered ... and then a
**netAvailable** event" - so on a clean break-before-make handover the sample
works. It is the other paths that fail: a make-before-break handover where the
cellular network is already up when Wi-Fi goes, or an app foregrounded onto a
cellular network it did not previously have, deliver `netAvailable` without a
preceding `netLost` and produce no dialog. And a `netLost` that is followed by
nothing leaves `signal` stuck at `1`, so the *next* unrelated network event
raises a spurious dialog.

The second defect in the same eight lines is `connection.getDefaultNet()`.
The callback is handed `data`, the handle that just became available, and then
ignores it to ask the system asynchronously which network is now the default.
Those are two different questions, and by the time the promise resolves the
answer can have changed again.

**The corrected listener - same file** (corrected, see `HW-13-0027`)

```typescript
On(callback: Callback<boolean>) {
  try {
    this.netCon = connection.createNetConnection();

    // FIX: no signal flag - judge each handle on its own capabilities
    this.netCon.on('netAvailable', (data: connection.NetHandle) => {
      const capabilities = connection.getNetCapabilitiesSync(data);   // FIX: the delivered handle
      if (capabilities.bearerTypes[0] === connection.NetBearType.BEARER_CELLULAR) {
        callback(true);
      }
    });

    this.netCon.on('netLost', (data: connection.NetHandle) => {
      hilog.info(0x0000, 'testTag', 'netLost: %{public}s', JSON.stringify(data));
    });

    this.netCon.register((error: BusinessError) => {   // FIX: register after on(), per the reference
      hilog.info(0x0000, 'testTag', 'register: %{public}s', JSON.stringify(error));
    });
  } catch (err) {
    hilog.error(0x0000, 'testTag', 'errCode: %{public}s, errMessage: %{public}s',
      (err as BusinessError).code, (err as BusinessError).message);
  }
}
```

`getNetCapabilitiesSync` is synchronous and takes the handle, so the whole
decision happens inside the callback with no window for the state to move
under it. Using the named enum instead of the literal `0` also documents what
the check means - `bearerTypes` is declared to hold exactly one element, and
`BEARER_CELLULAR` is `0`, `BEARER_WIFI` is `1`.

If the app must distinguish "was on Wi-Fi, now on cellular" from "was already
on cellular", keep the last seen bearer in a field and compare - that is a
state you own, unlike an event ordering you do not control.

**The pause and the dialog - `entry/src/main/ets/pages/HomePage.ets`** (as shipped)

```typescript
aboutToAppear(): void {
  NetCheck.On((data : boolean)=>{
    if (data) {
      this.controller.pause()
      this.videoState = false
      this.getUIContext().showAlertDialog(
        {
          title: $r('app.string.title'),
          message: $r('app.string.sec_title'),
          autoCancel: false,
          alignment: DialogAlignment.Center,
          width: 328,
          height: 195,
          gridCount: 4,
          primaryButton: {
            value: $r('app.string.cancel'),
            action: () => {
              this.controller.pause()
              this.videoState = false
            },
            backgroundColor: $r('app.color.button_background'),
          },
          secondaryButton: {
            enabled: true,
            defaultFocus: true,
            style: DialogButtonStyle.HIGHLIGHT,
            value: $r('app.string.ok'),
            action: () => {
              this.controller.start()
              this.videoState = true
            }
          },
          backgroundColor: $r('app.color.dialog_background')
        }
      )
    }
  });
}

aboutToDisappear(): void {
  NetCheck.Off();
}
```

**Three things here are worth copying.** The pause happens on the line before
the dialog is built, so no data is spent while the user reads it.
`autoCancel: false` means a tap outside cannot dismiss the question - for a
choice that costs money, an accidental dismissal must not be an implicit
"yes". And the dialog is raised through `this.getUIContext().showAlertDialog`
rather than the global `AlertDialog.show`, which is the form the UIContext
guide prescribes and which binds the dialog to the right window instance; this
sample gets that right where several others in the same industry do not
(`HW-13-0032`).

The `aboutToAppear` / `aboutToDisappear` pairing is the other half. A
`NetConnection` registered and never unregistered keeps delivering events into
a destroyed page's closure.

## Permissions & config

```json5
"requestPermissions":[
  {
    "name" : "ohos.permission.GET_NETWORK_INFO",
    "reason": "$string:reason",
    "usedScene": {
      "abilities": [
        "FormAbility"                      // does not exist - should be EntryAbility
      ],
      "when":"inuse"
    }
  }
]
```

- `GET_NETWORK_INFO` is `system_grant`, so there is no runtime request - it is
  granted at install time from this declaration alone. `register` requires it.
- `usedScene.abilities` names `FormAbility`, which this module does not
  declare; it ships only `EntryAbility` plus the backup extension
  (`HW-13-0028`). The same copy-paste appears in the photography industry's
  `ImageRotateAndFlip`.
- The ability declares `supportWindowMode: ["fullscreen"]` (no split screen)
  and `orientation: "auto_rotation_landscape"`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` and
  `targetSdkVersion` are both `6.0.0(20)`, so unlike the three docs in
  `HW-13-0004` this constraint section is accurate.
- `bearerTypes` holds exactly one element - the array is not a set of
  simultaneous bearers, and indexing `[0]` is correct.
- The sample only detects the switch *to* cellular. Going back to Wi-Fi does
  not resume playback; if that is wanted, handle `BEARER_WIFI` in the same
  callback.
- `NetCheck.On` is not idempotent: a second call overwrites `netCon` and the
  previous subscription is never unregistered. Guard it if more than one page
  can subscribe.
- The dialog text is a fixed string resource. Real players usually name the
  size of the remaining download in it; nothing in this sample knows it.
- `EntryAbility` registers no `avoidAreaChange` listener and drops the
  promises from `setWindowLayoutFullScreen` and the system-bar calls into
  `catch` handlers that only log - acceptable here because the page is a
  black full-screen video with no avoid-area padding.

## Pitfalls

- **`HW-13-0027`** (B/medium, probable): `netAvailable` acts only `if (this.signal)`,
  a flag set by a prior `netLost`, so a make-before-break Wi-Fi to cellular
  handover shows no dialog and leaves `signal` stuck at `1` for a later
  unrelated event; the handler also inspects `getDefaultNet()` instead of the
  `NetHandle` it was given. Fix: check the delivered handle's bearer directly
  in `netAvailable` and drop the flag.
- **`HW-13-0028`** (B/low, confirmed): `GET_NETWORK_INFO`'s `usedScene`
  references a `FormAbility` that the module does not declare. Fix: name
  `EntryAbility`.
- **The doc's snippet does not type-match the sample**: the document writes
  `this.signal = true` / `false` where `NetCheck.ets` declares
  `private signal = 0` and assigns `1` / `0`. Copying the document's step 2
  and 3 into the shipped class does not compile.

## References

- `documentation/harmonyos-references/03_system/js-apis-net-connection.md` - `createNetConnection`, `register`/`unregister`, `on('netLost')`, `on('netAvailable')`, `getNetCapabilitiesSync`, `NetBearType`, and the NOTE on event ordering
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-net-connection
- `documentation/harmonyos-guides/04_system/net-connection-manager.md` - the subscribe/register lifecycle
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/net-connection-manager
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.GET_NETWORK_INFO`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `usedScene` and what `abilities` must name
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- `documentation/harmonyos-references/02_application-framework/ts-media-components-video.md` - `Video` and `VideoController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-media-components-video
- `documentation/harmonyos-references/02_application-framework/ts-methods-alert-dialog-box.md` - `showAlertDialog`, `autoCancel`, `DialogButtonStyle`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-methods-alert-dialog-box
- `MEDIA-12` - the same `Video` + `VideoController` pair, driven by a list instead of the network
