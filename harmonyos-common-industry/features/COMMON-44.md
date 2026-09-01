---
id: COMMON-44
title: Long-running task notifications - live-view progress for a background data transfer
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/44_vpn_adaptation.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/vpn_adaptation-0000002507518793
sample: huawei_industry_tree/19_common_technical_solutions/downloads/VPNAdaptation.zip
kits: ["@kit.BackgroundTasksKit", "@kit.NotificationKit", "@kit.AbilityKit", "@kit.NetworkKit", "@kit.BasicServicesKit", "@kit.ArkTS", "@kit.ArkUI"]
apis: ["backgroundTaskManager.startBackgroundRunning", "backgroundTaskManager.stopBackgroundRunning", ContinuousTaskNotification, "wantAgent.getWantAgent", "wantAgent.WantAgentInfo", "wantAgent.OperationType.START_ABILITY", "notificationManager.publish", NotificationRequest, NotificationTemplate, "ContentType.NOTIFICATION_CONTENT_SYSTEM_LIVE_VIEW", "SlotType.LIVE_VIEW", "vpnExtension.createVpnConnection", "vpnExtension.startVpnExtensionAbility", "VpnConnection.protect", "VpnConnection.create", setInterval, clearInterval, "util.format"]
permissions: ["ohos.permission.INTERNET", "ohos.permission.KEEP_BACKGROUND_RUNNING"]
min_api: 20
modules: [entry]
findings: [HW-19-0137, HW-19-0138, HW-19-0139, HW-19-0140, HW-19-0141, HW-19-0142]
status: verified-with-fixes
---

## When to use

Load this card when the application must keep working after the user leaves it -
a transfer, a sync, a VPN session - and must show the user a **live progress
notification** while it does. On HarmonyOS the two halves are inseparable: a
continuous task is granted only against a notification, and the notification id
comes back from the grant.

The document frames this as VPN, but the transferable content is the
continuous-task-plus-live-view pattern. The VPN half of the sample does not work
as shipped (HW-19-0137), and the document's own closing note redirects readers to
a separate VPN sample for VPN functionality.

## Feature checklist

- Declare `ohos.permission.KEEP_BACKGROUND_RUNNING` and the ability's
  `backgroundModes`.
- Build a `WantAgent` with `OperationType.START_ABILITY` pointing at a **real
  ability** (HW-19-0138).
- `startBackgroundRunning(context, ['dataTransfer'], wantAgentObj)` and keep
  `res.notificationId`.
- Publish updates as `NOTIFICATION_CONTENT_SYSTEM_LIVE_VIEW` with
  `updateOnly: true`, `SlotType.LIVE_VIEW`, and the `downloadTemplate` template -
  **using the returned id**.
- `stopBackgroundRunning` and `clearInterval` when done.
- Handle rejections on every promise, including `getWantAgent` (HW-19-0141).

## Architecture

Single-module project (`entry` HAP) with a native library:

| File | Responsibility |
| --- | --- |
| `pages/MainPage.ets` | the toggle and connect button; owns the task lifecycle and the progress interval |
| `utils/WantAgentUtil.ets` | one-call `WantAgent` factory |
| `utils/NotificationPublish.ets` | builds the live-view update request and publishes the final text notification |
| `utils/VPNUtils.ets` | tunnel setup over `libentry.so` |
| `utils/Toast.ets`, `utils/Logger.ets` | UI feedback and logging |
| `constants/Constants.ets` | bundle name, ability name, time units, `INTERVAL = 6000` |
| `cpp/napi_init.cpp` | `udpConnect`, `startVpn`, `stopVpn` |

**The notification id is issued, not chosen.** `startBackgroundRunning` resolves
with a `ContinuousTaskNotification`, and `res.notificationId` is the id of the
notification the *system* created for the task. Every subsequent update must
carry that id - the sample's own comment says so: `id: id, // 必须是申请长时任务
返回的id，否则应用更新通知失败` ("must be the id returned when applying for the
long-running task, otherwise the application's notification update fails").
Publishing with any other id creates a second notification instead of updating
the first.

**Two different notification shapes, for two different moments.**

| | during the task | at the end |
| --- | --- | --- |
| content type | `NOTIFICATION_CONTENT_SYSTEM_LIVE_VIEW` | `NOTIFICATION_CONTENT_BASIC_TEXT` |
| slot | `SlotType.LIVE_VIEW` | `SlotType.CONTENT_INFORMATION` |
| id | `res.notificationId` | `1` |
| template | `downloadTemplate` with `progressValue` | none |
| `updateOnly` | `true` | absent |

`updateOnly: true` is what makes the live view refresh in place rather than
re-alerting the user every six seconds.

**The template's fields are fixed by the platform.** `name: 'downloadTemplate'`
is the only supported template name and `typeCode: 8` the only supported
live-view type code - the sample marks both `保持不变` ("keep unchanged"). Only
`title`, `fileName`, `progressValue`, and the `systemLiveView` text are the
application's.

**Progress is simulated.** `setInterval` every 6 s increments a counter to 100,
so the demo runs ten minutes. Note the interval is started by the click handler
regardless of whether the task actually started - see HW-19-0141.

**Teardown is complete and correct.** `stopContinuousTask` clears the interval
first, then `stopBackgroundRunning`, and `.finally(() => { this.linkTime = []; })`
resets the timestamps so a second connection measures its own duration. That
`finally` is easy to overlook and is what keeps the elapsed-time calculation
right across sessions.

## Implementation steps

1. **Declare the background mode.** In `module.json5`, on the ability:
   `"backgroundModes": ["dataTransfer"]`, plus
   `ohos.permission.KEEP_BACKGROUND_RUNNING` in `requestPermissions`. Name a real
   ability in `usedScene.abilities` (HW-19-0139).
2. **Create the WantAgent** with `OperationType.START_ABILITY` and the bundle
   name **and ability name** of the ability to open when the notification is
   tapped (HW-19-0138).
3. **Start the task**: `startBackgroundRunning(context, ['dataTransfer'],
   wantAgentObj)`, store `res.notificationId`, record the start time. Attach a
   `.catch` to every link in the chain (HW-19-0141).
4. **Update progress** on a timer: build a `NotificationRequest` with
   `NOTIFICATION_CONTENT_SYSTEM_LIVE_VIEW`, `updateOnly: true`,
   `SlotType.LIVE_VIEW`, the `downloadTemplate` template carrying
   `progressValue`, and the stored id; `notificationManager.publish(request)`.
5. **Stop**: `clearInterval`, `stopBackgroundRunning`, then publish the closing
   basic-text notification with the elapsed time; reset state in `.finally`.

## Verified snippets

All snippets below come from the sample project, not from the document.

### Starting the continuous task

`VPNAdaptation.zip#VPNAdaptation/entry/src/main/ets/pages/MainPage.ets`

```ts
import { WantAgent } from '@kit.AbilityKit';
import { backgroundTaskManager } from '@kit.BackgroundTasksKit';
import { BusinessError } from '@kit.BasicServicesKit';
import { notificationManager } from '@kit.NotificationKit';

// 开启长时任务
startContinuousTask() {
  try {
    this.vpnUtils.setupVpn();
    // 通过wantAgent模块下getWantAgent方法获取WantAgent对象
    wantAgentUtil.createWantAgent(Constants.BUNDLE_NAME, Constants.BUNDLE_NAME)   // FIX (HW-19-0138): ABILITY_NAME
      .then((wantAgentObj: WantAgent) => {                                        // FIX (HW-19-0141): no .catch
        let list: string[] = ['dataTransfer'];
        backgroundTaskManager.startBackgroundRunning(this.context, list, wantAgentObj)
          .then((res: backgroundTaskManager.ContinuousTaskNotification) => {
            Logger.info('StartBackgroundRunning succeeded');
            // 使用res中返回的notificationId来更新通知
            this.notificationId = res.notificationId;
            this.linkTime.push(new Date().getTime());
          })
          .catch((error: BusinessError) => {
            Logger.error(`Failed to StartBackgroundRunning. code is ${error.code} message is ${error.message}`);
          });
      });
  } catch (error) {
    Logger.error(`Failed to Operation getWantAgent. code is ${(error as BusinessError).code} message is ${(error as BusinessError).message}`);
  }
}
```

### Driving and stopping it

`VPNAdaptation.zip#VPNAdaptation/entry/src/main/ets/pages/MainPage.ets`

```ts
.onClick(() => {
  this.getUIContext().animateTo({ duration: Constants.ANIMATION_DURATION }, () => {
    this.isLinking = !this.isLinking;
  });
  let process: number = 0;
  if (this.isLinking) {
    this.startContinuousTask();
    this.intervalId = setInterval(() => {
      process++;
      this.updateContinuousTask(process);
    }, Constants.INTERVAL);
  } else {
    this.stopContinuousTask();
  }
});

// 应用更新进度
updateContinuousTask(process: number) {
  if (process >= 100) {
    this.isLinking = false;
    this.stopContinuousTask();
    return;
  }
  let request = this.notificationPublish.updateNotification(this.notificationId, process);
  notificationManager.publish(request)
    .then(() => {
      Logger.info(`Succeeded in updating notification.`);
    })
    .catch((err: BusinessError) => {
      Logger.error(`Update fail. code is ${err.code} message is ${err.message}`);
    });
}

// 停止长时任务
stopContinuousTask() {
  clearInterval(this.intervalId);

  backgroundTaskManager.stopBackgroundRunning(this.context)
    .then(() => {
      Logger.info(`StopBackgroundRunning.`);
      this.vpnUtils.destroy();
      this.linkTime.push(new Date().getTime());
      this.notificationPublish.publishNotification(this.linkTime);
    })
    .catch((err: BusinessError) => {
      Logger.error(`Failed to operation stopBackgroundRunning. Code is ${err.code}, message is ${err.message}`);
    })
    .finally(() => {
      this.linkTime = [];
    });
}
```

### The live-view progress request

`VPNAdaptation.zip#VPNAdaptation/entry/src/main/ets/utils/NotificationPublish.ets`

```ts
// 更新任务通知
updateNotification = (id: number, process: number) => {
  // 应用定义下载类通知模版
  let downLoadTemplate: notificationManager.NotificationTemplate = {
    name: 'downloadTemplate', // 当前只支持downloadTemplate，保持不变
    data: {
      title: 'vpn数据传输', // 必填
      fileName: '数据传输中', // 必填
      progressValue: process, // 应用更新进度值，自定义
    }
  };
  let request: notificationManager.NotificationRequest = {
    id: id, // 必须是申请长时任务返回的id，否则应用更新通知失败
    content: {
      // 系统实况类型，保持不变
      notificationContentType: notificationManager.ContentType.NOTIFICATION_CONTENT_SYSTEM_LIVE_VIEW,
      systemLiveView: {
        typeCode: 8, // 上传下载类型需要填写 8，当前仅支持此类型。保持不变
        title: `VPN Connect`, // 应用自定义
        text: `Connecting`, // 应用自定义
      }
    },
    updateOnly: true,
    notificationSlotType: notificationManager.SlotType.LIVE_VIEW, // 实况窗类型，保持不变
    template: downLoadTemplate // 应用需要设置的模版名称
  };
  return request;
};
```

### The closing text notification

`VPNAdaptation.zip#VPNAdaptation/entry/src/main/ets/utils/NotificationPublish.ets`

```ts
// 发布通知
publishNotification = async (linkTime: number[]) => {
  try {
    if (this.context !== undefined && this.context !== null) {
      let duration = linkTime[1] - linkTime[0];
      let timeStr = this.formatTime(duration);
      let notificationWantAgent = await wantAgentUtil.createWantAgent(Constants.BUNDLE_NAME, Constants.ABILITY_NAME);
      let request: notificationManager.NotificationRequest =
        {
          notificationSlotType: notification.SlotType.CONTENT_INFORMATION,
          id: 1, // 通知id，默认为1
          content: {
            notificationContentType: notification.ContentType.NOTIFICATION_CONTENT_BASIC_TEXT,
            normal: {
              title: 'vpn连接断开',
              text: '连接时长: ' + timeStr + ' 网络速度: 50 KB/s',
            }
          },
          wantAgent: notificationWantAgent
        };
      notificationManager.publish(request)
        .then(() => {
          Logger.info(`Succeeded in publishing notification.`);
        })
        .catch((err: BusinessError) => {
          Logger.info(`publish fail. code is ${err.code} message is ${err.message}`);
        });
    }
  } catch (error) {
    Logger.info(`publishNotificationWithWantAgent error, error = ${JSON.stringify(error)}`);
  }
};
```

This is the call site that passes `Constants.ABILITY_NAME` correctly - compare it
with `startContinuousTask` above, which passes `BUNDLE_NAME` twice.

### Elapsed-time formatting

`VPNAdaptation.zip#VPNAdaptation/entry/src/main/ets/utils/NotificationPublish.ets`

```ts
formatTime(num: number): string {
  if (num < Constants.MILLISECOND) {
    return '00:00';
  }
  let seconds = Math.floor(num / Constants.MILLISECOND);
  let minutes = Math.floor(num / (Constants.MILLISECOND * Constants.MINUTE_SECOND));
  let hours = Math.floor(num / (Constants.MILLISECOND * Constants.HOUR_SECOND));
  minutes = minutes - hours * Constants.MINUTE_SECOND;
  seconds = seconds - hours * Constants.HOUR_SECOND - minutes * Constants.MINUTE_SECOND;
  if (hours === 0) {
    return util.format('%s:%s', minutes.toString().padStart(2, '0'), seconds.toString().padStart(2, '0'));
  } else {
    return util.format('%s:%s:%s', hours.toString().padStart(2, '0'), minutes.toString().padStart(2, '0'),
      seconds.toString().padStart(2, '0'));
  }
}
```

Each unit is computed from the total and then reduced by the larger units - a
correct decomposition, verified against `MILLISECOND = 1000`,
`MINUTE_SECOND = 60`, `HOUR_SECOND = 3600`.

### The WantAgent factory

`VPNAdaptation.zip#VPNAdaptation/entry/src/main/ets/utils/WantAgentUtil.ets`

```ts
import wantAgent from '@ohos.app.ability.wantAgent';   // FIX (HW-19-0140): '@kit.AbilityKit'

class WantAgentUtil {
  async createWantAgent(bundleName: string, abilityName: string) {
    let wantAgentInfo: wantAgent.WantAgentInfo = {
      wants: [
        {
          bundleName: bundleName,
          abilityName: abilityName
        }
      ],
      actionType: wantAgent.OperationType.START_ABILITY,
      requestCode: 0 // WantAgentInfo的请求码，默认为0
    };
    return await wantAgent.getWantAgent(wantAgentInfo);
  }
}

export const wantAgentUtil = new WantAgentUtil();
```

### The VPN connection (does not work as shipped)

`VPNAdaptation.zip#VPNAdaptation/entry/src/main/ets/utils/VPNUtils.ets`

```ts
import vpn_client from 'libentry.so';
import vpnExt from '@ohos.net.vpnExtension';

let want: Want = {
  bundleName: Constants.BUNDLE_NAME,
  abilityName: Constants.ABILITY_NAME,     // EntryAbility - a UIAbility, not a vpn extension
};

export class VPNUtils {
  private ctx: common.VpnExtensionContext;
  private vpnConnection: vpnExt.VpnConnection;
  private vpnServerIp = 'x.x.x.x'; // 示例服务端地址
  private tunIp: string = 'x.x.x.x'; // 示例地址
  private blockedAppName: string = 'com.example.xxx'; // 示例

  constructor(ctx: Context) {
    this.ctx = ctx as common.VpnExtensionContext;          // FIX (HW-19-0137): a UIAbilityContext
    this.vpnConnection = vpnExt.createVpnConnection(this.ctx);  // FIX: before startVpnExtensionAbility
  }

  createTunnel() {
    gTunnelFd = vpn_client.udpConnect(this.vpnServerIp, 8888);
    if (gTunnelFd) {                                       // FIX (HW-19-0142): fd 0 is valid
      Toast.showToast('CreateTunnel Success');
      Logger.info('CreateTunnel Success');
    } else {
      Toast.showToast('CreateTunnel Fail');
      Logger.error('CreateTunnel Fail');
    }
  }
```

The three `x.x.x.x` / `com.example.xxx` placeholders are marked `示例`
("example") and must be replaced before the tunnel can connect to anything - the
document never says so.

## Permissions & config

`VPNAdaptation.zip#VPNAdaptation/entry/src/main/module.json5`:

```json5
"abilities": [
  {
    "name": "EntryAbility",
    "backgroundModes": ["dataTransfer"],
    ...
  }
],
"extensionAbilities": [
  // the VPN extension entry is present but commented out
],
"requestPermissions": [
  {
    "name": "ohos.permission.INTERNET",
    "reason": "$string:grant_internet",
    "usedScene": {
      "abilities": ["MainAbility"],     // FIX (HW-19-0139): no such ability
      "when": "inuse"
    }
  },
  { "name": "ohos.permission.KEEP_BACKGROUND_RUNNING" }
]
```

- **`ohos.permission.INTERNET`** - normal, granted at install; the network access
  the transfer needs.
- **`ohos.permission.KEEP_BACKGROUND_RUNNING`** - normal, granted at install;
  required by `startBackgroundRunning`. Declared with no `reason`/`usedScene`,
  which is permitted for a normal permission.
- **`backgroundModes: ["dataTransfer"]`** must be declared on the ability and must
  match the list passed to `startBackgroundRunning`.

`deviceTypes: ["default", "tablet"]`. `build-profile.json5` pins `6.0.0(20)`.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **The continuous task requires a WantAgent** - `startBackgroundRunning` will
  not grant one without it, and the WantAgent determines what the notification
  tap opens.
- **Updates must use the returned `notificationId`.** Any other id creates a new
  notification instead of updating the live view.
- **`downloadTemplate` is the only template name** and `typeCode: 8` the only
  live-view type code currently supported.
- **`title` and `fileName` in the template data are mandatory.**
- **`createVpnConnection` needs a `VpnExtensionContext`** obtained inside a
  `VpnExtensionAbility`, and `startVpnExtensionAbility` must be called first.
- **The task must be stopped explicitly.** `stopBackgroundRunning` plus
  `clearInterval`; the system does not stop it when the page goes away.
- **Devices.** `default` and `tablet` per `module.json5`; `createVpnConnection`
  is documented for phone, PC/2in1, tablet and wearable.

## Pitfalls

- **`createVpnConnection` is called in a field initializer with a
  `UIAbilityContext` force-cast to `VpnExtensionContext`, before
  `startVpnExtensionAbility`, and the `VpnExtensionAbility` is commented out of
  `module.json5` - all of which is incorrect.** The VPN path cannot establish a
  tunnel as shipped. (HW-19-0137)
- **`createWantAgent(BUNDLE_NAME, BUNDLE_NAME)` passes the bundle name as the
  ability name, which is incorrect** - the notification tap has no ability to
  start. Use `Constants.ABILITY_NAME`, as `NotificationPublish` does.
  (HW-19-0138)
- **`usedScene.abilities: ["MainAbility"]` names an ability that does not exist,
  which is incorrect** - the module declares only `EntryAbility`. `MainAbility`
  is the FA-model name and is a leftover from an older template. (HW-19-0139)
- **`@ohos.notificationManager` is imported twice under two names in one file, and
  three modules bypass their kits, which is incorrect** - the same project imports
  `notificationManager` from `@kit.NotificationKit` elsewhere. (HW-19-0140)
- **The `createWantAgent` promise has no `.catch`, and the `try/catch` around it
  cannot catch an async rejection, which is incorrect** - the failure is silent and
  the interval then publishes forever against `notificationId` 0. (HW-19-0141)
- **`if (gTunnelFd)` tests a file descriptor for truthiness, which is incorrect** -
  fd 0 is valid and would be reported as a failure. (HW-19-0142)
- **Do not publish the progress update with your own id.** The sample's own
  comment is the warning: the update silently fails and you get a second
  notification.
- **Do not start the progress timer before the task is granted.** The click
  handler here does, so a failed `startBackgroundRunning` leaves a live interval
  with nothing behind it.
- **Replace the `x.x.x.x` placeholders.** They are marked `示例` in the source and
  nowhere in the document.

## References

- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-resourceschedule-backgroundtaskmanager -
  `startBackgroundRunning`, `stopBackgroundRunning`, `ContinuousTaskNotification`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-notificationmanager -
  `NotificationRequest`, `NotificationTemplate`, `ContentType`, `SlotType`,
  `updateOnly`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-app-ability-wantagent -
  `getWantAgent`, `WantAgentInfo`, `OperationType`.
- `documentation/harmonyos-references/03_system/js-apis-net-vpnextension.md` -
  `createVpnConnection` (the ordering note, the required context type, and the
  `VpnExtensionAbility` example), `startVpnExtensionAbility`, `VpnConnection`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-net-vpnextension
- `documentation/harmonyos-references/05_common-capabilities/js-apis-timer.md` -
  `setInterval` / `clearInterval`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-timer
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/vpn_adaptation-0000002507518793
