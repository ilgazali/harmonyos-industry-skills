---
id: OFFICE-31
title: Collaborative schedule management - Calendar Kit sync, team/colleague visibility switches, and Tap to Transfer sharing over a Deep Link
industry: 05_office
doc: huawei_industry_tree/05_office/docs/31_schedule_share.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule_share-0000002573150445
sample: huawei_industry_tree/05_office/downloads/ScheduleShare.zip
kits: ["@kit.CalendarKit", "@kit.ShareKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.ArkData", "@kit.ArkTS", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["calendarManager.getCalendarManager", "CalendarManager.getCalendar", "Calendar.addEvent", "Calendar.updateEvent", "Calendar.deleteEvent", "calendarManager.Event", "calendarManager.EventType", "abilityAccessCtrl.createAtManager", "AtManager.requestPermissionsFromUser", PermissionRequestResult, "harmonyShare.on('knockShare')", "harmonyShare.off('knockShare')", "harmonyShare.SharableTarget", "SharableTarget.share", "systemShare.SharedData", "utd.UniformDataType.HYPERLINK", canIUse, "UIAbility.onCreate", "UIAbility.onNewWant", "Want.uri", "url.URL.parseURL", "util.Base64Helper", "util.TextEncoder", "util.TextDecoder", "Context.eventHub", "pasteboard.getSystemPasteboard", NavDestinationMode, "@ComponentV2", "@Provider", "@Consumer", "@Monitor", "@CustomDialog"]
permissions: ["ohos.permission.READ_CALENDAR", "ohos.permission.WRITE_CALENDAR"]
min_api: 20
modules: [entry, schedule]
findings: [HW-05-0170, HW-05-0171, HW-05-0172, HW-05-0173, HW-05-0174, HW-05-0175, HW-05-0176, HW-05-0177, HW-05-0178, HW-05-0179, HW-05-0180, HW-05-0181, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card for a **team calendar**: personal schedules that also live in the
system calendar, colleagues' and teams' schedules overlaid on the same month
view with per-source visibility switches, and face-to-face sharing of a single
schedule by tapping two phones together.

The three scenarios are independent and can be lifted separately:

1. **System-calendar sync** - Calendar Kit `addEvent` / `updateEvent` /
   `deleteEvent`, keeping the returned event id on the local record.
2. **Visibility switches** - a `NavDestinationMode.DIALOG` bottom sheet with a
   checkbox per team and per colleague, driving a `noShow` flag on each
   schedule.
3. **Tap to Transfer** - `harmonyShare.on('knockShare')` on the share dialog,
   sending a Deep Link whose query string carries the schedule as URL-safe
   Base64; the receiver's exported ability picks it up from `want.uri`.

Compare OFFICE-11 (one meeting, `editEvent`) and OFFICE-27 (batch sync of
to-dos). This card is the only one in `05_office` that covers Share Kit's
knock-share, and the only two-module project.

## Feature checklist

The application must be able to:

- Request `READ_CALENDAR` / `WRITE_CALENDAR` and hold a single `Calendar` handle.
- Create a schedule from a form and mirror it into the system calendar, storing
  the returned event id.
- Keep team schedules local and personal schedules synced.
- Show teams' and colleagues' schedules in a colour-coded month view.
- Toggle each team's and colleague's visibility from a bottom-sheet dialog and
  refresh the calendar immediately.
- Encode a personal schedule into a Deep Link and share it by tapping two
  phones.
- Copy the same link to the clipboard as a fallback.
- Receive the link in `onCreate` / `onNewWant`, decode it, and offer a one-tap
  import.

## Architecture

Two modules - an `entry` HAP and a `schedule` HAR:

```
entry (HAP)                                schedule (HAR, type: "har")
  EntryAbility.ets                           Index.ets            <- public barrel
    onCreate / onNewWant -> want.uri         schedule/            <- the five pages
    knock-share receive                        ScheduleView       <- month view + list
    deep-link Action=Create                    NewSchedule / EditSchedule / ScheduleDetail
  pages/Index.ets -> ScheduleView             SettingSchedule    <- visibility switches
  module.json5                                 components/
    skills: scheme "link", host "shareSchedule"  ReminderDialog.ets  <- 3 dialogs, see below
    requestPermissions: READ/WRITE_CALENDAR    utils/
                                                 ScheduleUtils    <- Calendar Kit
                                                 ShareUtil        <- link build/parse + Base64
                                               mock/DataManager   <- personal/team/colleague maps
                                               viewmodel/{RouterModule, MainEntryVM, GlobalRegister}
```

`ReminderDialog.ets` holds **three** `@CustomDialog` structs, not one:
`ReminderDialog`, `ShareConfirmDialog` (the knock-share sender) and
`ReceiveShareDialog` (the importer). The document's project tree names only the
first (HW-05-0180).

The share round trip:

```
sender: ShareConfirmDialog.aboutToAppear
          canIUse('SystemCapability.Collaboration.HarmonyShare')
          harmonyShare.on('knockShare', shareCallback)
        [phones tapped]
          shareCallback(sharableTarget)
            ShareUtil.shareSchedule -> canShare? -> creatShareScheduleData
            buildShareText -> JSON -> Base64 -> URL-safe -> link://shareSchedule?data=...
            new systemShare.SharedData({ utd: HYPERLINK, content: link, title, description })
            sharableTarget.share(shareData)
        aboutToDisappear -> harmonyShare.off('knockShare', shareCallback)

receiver: EntryAbility.onCreate / onNewWant
            want.uri startsWith 'link://'
            AppStorage 'isFromKnockShare' + 'knockShareData'
            context.eventHub.emit('knockShareDataUpdated', ...)
          ScheduleView.aboutToAppear
            eventHub.on('knockShareDataUpdated')  (hot start)
            AppStorage.get(...)                    (cold start)
            checkKnockShareData -> parse -> ReceiveShareDialog
              DataManager.addOtherSchedule(colleague, schedule)
              eventHub.emit('scheduleDataChanged') -> ScheduleView.load()
```

Writing to **both** `AppStorage` and the `eventHub` is deliberate: on a cold
start the emit happens in `onCreate`, before any page has subscribed, so the
page reads the stored copy instead.

## Implementation steps

1. **Declare both calendar permissions** in the entry module's `module.json5`
   with `reason` and `usedScene`.
2. **Initialise the calendar handle once**, awaiting the permission result and
   `getCalendar` before anything can call `addEvent` (HW-05-0171), and complete
   the permission ladder with `requestPermissionOnSetting` plus user feedback on
   refusal (HW-05-0174).
3. **Convert the app model to `calendarManager.Event`** with
   `EventType.NORMAL`, millisecond timestamps and `reminderTime` in minutes;
   add `ONE_DAY` to the end of an all-day event.
4. **Store the returned event id** on the local record, and stop the save when
   there is none rather than reporting success (HW-05-0173). Return `undefined`
   from the error path, never a promise that cannot settle (HW-05-0170).
5. **Guard update and delete on the system event id**, not the app-local id
   (HW-05-0177).
6. **Build the settings sheet** as a `NavDestination` with
   `.mode(NavDestinationMode.DIALOG)` and toggle `noShow` on each schedule.
7. **Register the knock-share listener on the share dialog only**, behind
   `canIUse('SystemCapability.Collaboration.HarmonyShare')`, and unregister it
   in `aboutToDisappear`.
8. **Encode the payload as URL-safe Base64** and build
   `link://<host>?data=<payload>`, with the scheme and host matching the
   receiver's `skills.uris` exactly.
9. **Declare the link in the receiver's `module.json5`** and read `want.uri` in
   both `onCreate` and `onNewWant`.
10. **Validate everything that arrives over the link, and confirm with the user
    before writing to the system calendar** (HW-05-0172).
11. **Unregister the window listener** in `onWindowStageWillDestroy`
    (HW-05-0175).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### Declaring the link and the permissions

`ScheduleShare.zip#entry/src/main/module.json5`

```json5
"skills": [
  {
    "entities": ["entity.system.home"],
    "actions": ["ohos.want.action.home"]
  },
  {
    "actions": [
      "ohos.want.action.sendData",
      "ohos.want.action.viewData",
    ],
    "uris": [
      {
        "scheme": "link",
        "host": "shareSchedule",
        //监听链接请根据实际情况进行更换
      },
    ]
  }
],
"requestPermissions": [
  {
    "name": "ohos.permission.READ_CALENDAR",
    "reason": "$string:read_calendar_reason",
    "usedScene": { "when": "always" }
  },
  {
    "name": "ohos.permission.WRITE_CALENDAR",
    "reason": "$string:write_calendar_reason",
    "usedScene": { "when": "always" }
  }
]
```

The second `skills` entry is what makes `link://shareSchedule?...` reach this
ability. `scheme` and `host` must match the sender byte for byte - the sample's
own comment on the callback says so. Note that `usedScene` here carries only
`when` and no `abilities` list, and declares `always` rather than `inuse`;
OFFICE-27's `{"abilities": ["EntryAbility"], "when": "inuse"}` is the tighter
form for a foreground-only calendar app.

### The knock-share sender

`ScheduleShare.zip#ablility/schedule/src/main/ets/schedule/components/ReminderDialog.ets`
(inside `ShareConfirmDialog`)

```ts
// 碰一碰分享回调事件，此处使用DeepLinking作为示例，如使用AppLinking需要应用在AGC平台配置请参考官网指南
// 在示例中需要在打开分享弹框后，再去碰一碰才能生效
private shareCallback = (sharableTarget: harmonyShare.SharableTarget) => {
  // 1.构建链接，链接的scheme和host必须与接收方配置的完全一致
  const linking: string = ShareUtil.shareSchedule(this.schedule, this.getUIContext());

  // 2.构造要分享的数据，此处仅作示例，实际可替换为文件、图片或自定义的Want参数
  const shareData: systemShare.SharedData = new systemShare.SharedData({
    utd: utd.UniformDataType.HYPERLINK,
    content: linking,
    title: `日程分享`, // 卡片标题
    description: `日程: ${this.schedule.title}`, // 卡片描述
    // 如果有预览图，可设置thumbnailUri或thumbnail字段
  });

  // 3.调用接口，执行分享
  sharableTarget.share(shareData);
};

aboutToAppear() {
  this.shareLink = ShareUtil.shareSchedule(this.schedule, this.getUIContext());

  // 页面出现时注册监听，监听碰一碰
  if (canIUse('SystemCapability.Collaboration.HarmonyShare')) {
    harmonyShare.on('knockShare', this.shareCallback);
  }
}

aboutToDisappear(): void {
  // 页面消失时取消监听，避免内存泄漏
  if (canIUse('SystemCapability.Collaboration.HarmonyShare')) {
    harmonyShare.off('knockShare', this.shareCallback);
  }
}
```

This is the model to copy: the `canIUse` capability check the Share Kit overview
prescribes, registration scoped to the page that owns the shareable item, and
`off` with the **same callback reference** on the way out. The link is built
inside the callback, when the target device is already known, and again in
`aboutToAppear` for the copy-to-clipboard fallback.

### Encoding the payload

`ScheduleShare.zip#ablility/schedule/src/main/ets/utils/ShareUtil.ets`

```ts
export function encodeScheduleInfo(shareScheduleData: ShareScheduleData): string {
  try {
    // 1.将整个对象转为JSON字符串
    const jsonStr = JSON.stringify(shareScheduleData);
    // 2.Base64编码（解决URL传输中的特殊字符问题）
    const base64Str = encodeToBase64(jsonStr);
    // 3.URL安全的Base64编码（替换+/=等特殊字符）
    const urlSafeBase64 = base64Str
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
      .replace(/=+$/, '');
    return urlSafeBase64;
  } catch (error) {
    console.error('编码ScheduleInfo失败:', error);
    return '';
  }
}

function encodeToBase64(str: string): string {
  try {
    const textEncoder = new util.TextEncoder();
    const uint8Array = textEncoder.encodeInto(str);
    const base64Helper = new util.Base64Helper();
    return base64Helper.encodeToStringSync(uint8Array);
  } catch (error) {
    console.error('Base64编码失败:', error);
    return '';
  }
}

export function buildShareText(schedule: ShareScheduleData): string {
  const encodedSchedule = encodeScheduleInfo(schedule);
  const shareUrl = `link://shareSchedule?data=${encodedSchedule}`;
  return shareUrl;
}
```

The `+/=` to `-_` substitution and the stripped padding are the standard
URL-safe Base64 alphabet, and the decoder restores both:

```ts
let base64Str = encodedStr.replace(/-/g, '+').replace(/_/g, '/');
const padding = base64Str.length % 4;
if (padding) {
  base64Str += '='.repeat(4 - padding);
}
```

Base64 here is **transport encoding, not protection** - the payload is trivially
readable by anything that sees the link.

### The receiver

`ScheduleShare.zip#entry/src/main/ets/entryability/EntryAbility.ets`

```ts
private handleKnockShareData(want: Want): void {
  // 获取分享的内容
  const link = want.uri;
  if (link && link.startsWith('link://')) {
    this.isFromKnockShare = true;
    this.knockShareData = link;

    // 将数据存储到AppStorage，全局UI都可以访问
    AppStorage.setOrCreate('isFromKnockShare', this.isFromKnockShare);
    AppStorage.setOrCreate('knockShareData', this.knockShareData);

    // 通过EventHub发送事件通知页面
    this.context.eventHub.emit('knockShareDataUpdated', {
      isFrom: true,
      data: link
    });
  }
}
```

Called from **both** `onCreate` and `onNewWant`, which is what the Share Kit
guide prescribes: "the target app can obtain the want data passed through the
onCreate or onNewWant callback. In the want data, want.uri is the link".

`ScheduleShare.zip#ablility/schedule/src/main/ets/schedule/ScheduleView.ets`

```ts
aboutToAppear(): void {
  this.load();

  // 监听EventHub事件
  this.context.eventHub.on('knockShareDataUpdated', (data: ESObject) => {
    this.isFromKnockShare = true;
    this.knockShareData = data.data;
    this.checkKnockShareData();
  });

  // 添加监听日程数据变化事件
  this.context.eventHub.on('scheduleDataChanged', () => {
    this.load(); // 重新加载数据
  });

  // 检查是否有已存在的数据（应用冷启动时）
  const existingFlag = AppStorage.get<boolean>('isFromKnockShare');
  const existingData = AppStorage.get<string>('knockShareData');
  if (existingFlag && existingData) {
    this.isFromKnockShare = existingFlag;
    this.knockShareData = existingData;
    this.checkKnockShareData();
  }

  this.initDialogController();
}

// 组件销毁时移除事件监听
aboutToDisappear(): void {
  this.context.eventHub.off('knockShareDataUpdated');
  this.context.eventHub.off('scheduleDataChanged');
}
```

Both the subscription and the cold-start fallback, and both listeners
unregistered - the pattern the window listener in `EntryAbility` should have
followed (HW-05-0175).

### The Calendar Kit layer - as shipped

`ScheduleShare.zip#ablility/schedule/src/main/ets/utils/ScheduleUtils.ets`

```ts
public static async initSysCalender(): Promise<void> {
  if (ScheduleUtils.calendar) {
    return;
  }
  let calendarMgr: calendarManager.CalendarManager | null = null;
  const permissions: Permissions[] = [
    'ohos.permission.READ_CALENDAR',
    'ohos.permission.WRITE_CALENDAR'
  ];
  let atManager = abilityAccessCtrl.createAtManager();
  atManager.requestPermissionsFromUser(GlobalRegister.getContext(), permissions)   // not awaited - HW-05-0171
    .then((result: PermissionRequestResult) => {
      const allGranted = result.authResults.every(
        r => r === abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED
      );
      if (!allGranted) {
        hilog.warn(0x0000, TAG, 'Calendar permission denied');   // no second stage, no UI - HW-05-0174
        return;
      }
      calendarMgr = calendarManager.getCalendarManager(GlobalRegister.getContext());
      calendarMgr.getCalendar().then((data: calendarManager.Calendar) => {
        ScheduleUtils.calendar = data; // 保存全局实例
      }).catch((err: BusinessError) => { /* ... */ });
    })
    .catch((error: BusinessError) => { /* ... */ });
}

public static async addEvent(schedule: ScheduleInfo): Promise<number | undefined> {
  const event = ScheduleUtils.initEvent(schedule);
  try {
    return await ScheduleUtils.calendar?.addEvent(event);
  } catch (e) {
    console.error(`Failed to update event. Code: ${e.code}, message: ${e.message}`);
    return new Promise<undefined>(() => undefined);       // never settles - HW-05-0170
  }
}
```

Corrected:

```ts
public static async initSysCalender(): Promise<void> {
  if (ScheduleUtils.calendar) {
    return;
  }
  const permissions: Permissions[] = ['ohos.permission.READ_CALENDAR', 'ohos.permission.WRITE_CALENDAR'];
  const atManager = abilityAccessCtrl.createAtManager();
  const context = GlobalRegister.getContext();

  let result = await atManager.requestPermissionsFromUser(context, permissions);
  let granted = result.authResults.every(r => r === abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED);
  if (!granted && result.dialogShownResults?.some(shown => !shown)) {
    const statuses = await atManager.requestPermissionOnSetting(context, permissions);
    granted = statuses.every(s => s === abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED);
  }
  if (!granted) {
    // tell the user the calendar sync is off until the permission is granted
    return;
  }
  ScheduleUtils.calendar = await calendarManager.getCalendarManager(context).getCalendar();
}

public static async addEvent(schedule: ScheduleInfo): Promise<number | undefined> {
  await ScheduleUtils.initSysCalender();
  if (!ScheduleUtils.calendar) {
    return undefined;
  }
  try {
    return await ScheduleUtils.calendar.addEvent(ScheduleUtils.initEvent(schedule));
  } catch (e) {
    hilog.error(0x0000, TAG, 'Failed to add event. Code: %{public}d, message: %{public}s',
      (e as BusinessError).code, (e as BusinessError).message);
    return undefined;
  }
}
```

### Model conversion

`ScheduleShare.zip#ablility/schedule/src/main/ets/utils/ScheduleUtils.ets`

```ts
private static initEvent(schedule: ScheduleInfo) {
  const event: calendarManager.Event = {
    title: schedule.title,
    description: schedule.desc,
    // 日程类型，不推荐三方开发者使用calendarManager.EventType.IMPORTANT，重要日程类型不支持一键服务跳转功能及无法自定义提醒时间
    type: calendarManager.EventType.NORMAL, // 普通日程类型
    isAllDay: schedule.isAllDay,
    startTime: new Date(schedule.startTime).getTime(),
    endTime: ScheduleUtils.getEndTime(schedule),
    reminderTime: schedule.reminderTime, // 提前提醒时间（分钟）
  };
  if (schedule.location) {
    event.location = {
      location: schedule.location,
      longitude: schedule.longitude,
      latitude: schedule.latitude
    };
  }
  return event;
}

private static getEndTime(schedule: ScheduleInfo) {
  return schedule.isAllDay ? new Date(schedule.endTime).getTime() + CommonConstants.ONE_DAY :
    new Date(schedule.endTime).getTime();
}
```

Two things worth keeping: the note that third-party apps should not use
`EventType.IMPORTANT` - it blocks one-tap service jumps and custom reminder
times - and the all-day `+ ONE_DAY` on the end time, since the system treats an
all-day range as exclusive of the final day.

### The visibility sheet

`ScheduleShare.zip#ablility/schedule/src/main/ets/schedule/SettingSchedule.ets` -
the dialog is a `NavDestination` in `DIALOG` mode, the same shape
`NewSchedule.ets` uses:

```ts
.mode(NavDestinationMode.DIALOG)
.onBackPressed(() => {
  return false;
})
.expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.TOP, SafeAreaEdge.BOTTOM])
.hideTitleBar(true);
```

A dialog that is a route rather than a `CustomDialogController` keeps the back
gesture, the navigation stack and the parameter passing uniform with the rest of
the app - that is the reusable idea in 实现思路 step 2.

## Permissions & config

| Permission | Type | Why |
| --- | --- | --- |
| `ohos.permission.READ_CALENDAR` | user_grant | `getCalendar`, reading back events |
| `ohos.permission.WRITE_CALENDAR` | user_grant | `addEvent` / `updateEvent` / `deleteEvent` |

Both are requested at runtime from `ScheduleUtils.initSysCalender`, whose own
comment says the request is placed there only for readability: `// 定义需要的权限，这里就近申请权限仅作示例，方便理解，推荐是在EntryAbility.ets文件中进行操作`
("this near-the-use-site request is only an example for clarity; doing it in
EntryAbility.ets is recommended"). That matches the Calendar Kit guide, which
says "You are advised to perform managements in the EntryAbility.ets file."

Share Kit needs **no permission** - the user initiates the transfer physically,
and the system mediates it.

`ScheduleShare.zip#build-profile.json5`

```json5
"targetSdkVersion": "6.0.0(20)",
"compatibleSdkVersion": "6.0.0(20)",
"runtimeOS": "HarmonyOS",
"buildOption": { "strictMode": { "caseSensitiveCheck": true, "useNormalizedOHMUrl": true } }
```

```json5
"modules": [
  { "name": "entry",    "srcPath": "./entry",             ... },
  { "name": "schedule", "srcPath": "./ablility/schedule", ... }   // misspelt - HW-05-0179
]
```

`deviceTypes` is `["phone"]` for both modules, which is consistent with a
feature built on phone-to-phone tapping.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later - and the project agrees.
- **Tap to Transfer needs both devices unlocked with the screen on**, and Huawei
  Share enabled. If either device does not support it, tapping does nothing at
  all; if the user disabled Huawei Share, they get a system notification asking
  them to turn it on.
- **Share Kit does not report the result to the app.** "Share Kit does not send
  sharing results to host apps. Instead, Share Kit notifies the users who
  initiate sharing of the sharing result (accepted or rejected) through system
  notifications." Do not build UI that waits for a confirmation.
- **Register the listener on the page that owns the shareable item**, and
  unregister it when that page goes away - the sample's comment adds that in
  this demo you must open the share dialog *before* tapping the phones.
- **Guard with `canIUse('SystemCapability.Collaboration.HarmonyShare')`** - this
  is the check the Share Kit overview prescribes for the capability.
- **The link's scheme and host must match the receiver's `skills.uris`
  exactly**, or the want never arrives.
- **`want.uri` is readable in `onCreate` and `onNewWant` only.** On a cold start
  `onCreate` runs before any page exists, which is why the payload is parked in
  `AppStorage` as well as emitted on the `eventHub`.
- **`EventType.IMPORTANT` is not for third-party apps** - the sample's comment
  states it blocks one-tap service jumps and custom reminder times. Use
  `EventType.NORMAL`.
- **All-day events need `+ ONE_DAY` on the end time.**
- **The backend is a mock.** `RequestProxy` and `DataManager` are in-memory; the
  document's own tree labels them "（Mock示例）". Nothing survives a restart, and
  no colleague data is ever fetched.
- **Base64 is encoding, not encryption** - anything that can read the link can
  read the schedule's title, times and location.
- **The reference pages for Share Kit are stubs in this repository**
  (`documentation/harmonyos-references/06_application-services/share-harmony-share.md`
  is 11 lines), so the exact `harmonyShare.on/off('knockShare')` overloads were
  verified against the guide's group-sharing sample rather than the reference.

## Pitfalls

- **Returning `new Promise<undefined>(() => undefined)` from a catch block is
  incorrect** - the executor never resolves or rejects, so every awaiting caller
  hangs: no failure toast, no local save, and the new-schedule dialog never
  closes. Return `undefined` and let `async` wrap it. (HW-05-0170)
- **Declaring `initSysCalender` async while leaving the permission and
  `getCalendar` chain un-awaited is incorrect.** The calendar handle is still
  undefined when the user presses Save, `calendar?.addEvent` short-circuits, and
  the schedule exists only in the app - the opposite of the "lossless
  synchronisation" the document promises. (HW-05-0171)
- **Writing a system-calendar event straight from an incoming Deep Link is
  incorrect.** The ability is exported with a `link://shareSchedule` skill, so
  any app can trigger `Action=Create` with a title it chooses and no user
  confirmation, and the missing validation - which the document explicitly
  claims exists - turns an absent parameter into an event titled `"undefined"`
  with `NaN` timestamps. (HW-05-0172)
- **Showing the failure toast and then the success toast is incorrect**: there
  is no `return` between them, so a failed calendar write is reported as both
  failed and successful, and the record is stored with `eventId === undefined`
  and can never be reconciled with the system calendar afterwards. (HW-05-0173)
- **Checking only `authResults` is incorrect** for a `user_grant` permission
  that the whole feature depends on. Read `dialogShownResults`, escalate with
  `requestPermissionOnSetting`, and tell the user - the sibling MultiSchedule
  sample in this industry shows the complete ladder. (HW-05-0174)
- **Registering `windowSizeChange` without a matching `off` is incorrect**; the
  lifecycle guide names `onWindowStageWillDestroy` as the place to unsubscribe,
  and this project's own eventHub and knock-share registrations already do it
  properly. (HW-05-0175)
- **Handling the deep link from `onCreate` is incorrect** when the handler needs
  a `UIContext`: the field is assigned later, inside an async `getMainWindow`
  callback, so every toast on the cold-start path is silently dropped. Defer the
  user-visible part until the content is loaded. (HW-05-0176)
- **`if (schedule.id)` guarding a call that sends `schedule.eventId` is
  incorrect** - the two ids are different fields, and the Calendar API is asked
  to update an event with `id: undefined` whenever the original write returned
  nothing. `deleteEvent` guards on the right one. (HW-05-0177)
- **`import url from '@ohos.url'` is incorrect** for a project that uses
  `@kit.*` everywhere else and builds with `useNormalizedOHMUrl`; the reference
  documents `import { url } from '@kit.ArkTS';`. (HW-05-0178)
- **The document's tree says `ability/schedule`, which is incorrect** - the
  directory is `ablility/schedule`, and both `build-profile.json5`'s `srcPath`
  and `entry/oh-package.json5`'s `file:` dependency point at the misspelling.
  (HW-05-0179)
- **The tree says `ReminderDialog.ets` is the reminder-time dialog, which is
  incomplete** - it also contains `ShareConfirmDialog` and `ReceiveShareDialog`,
  the whole of scenario 3 - **and says `ScheduleAbility.ets` is the module's
  UIAbility, which is incorrect**: the schedule module is a HAR that declares no
  abilities, and nothing references that class. (HW-05-0180)
- **Telling the user the imported schedule went to 团队日程 is incorrect** - the
  code adds it to the colleague group via `DataManager.addOtherSchedule`, which
  is a separate section of the same settings sheet. (HW-05-0181)

## References

Reference and guide pages used to verify this card:

- `documentation/harmonyos-guides/07_application-services/knock-share-between-phones-overview.md` -
  the Tap to Transfer service process, the unlocked/screen-on restriction, the
  fact that Share Kit does not return the sharing result to the app, and the
  `canIUse('SystemCapability.Collaboration.HarmonyShare')` check.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/knock-share-between-phones-overview
- `documentation/harmonyos-guides/07_application-services/knock-share-between-phones-group.md` -
  the `harmonyShare.on('knockShare', ...)` / `off` pair in
  `aboutToAppear` / `aboutToDisappear`, the `SharedData` with
  `utd.UniformDataType.HYPERLINK`, and reading the link from `want.uri` in
  `onCreate` / `onNewWant`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/knock-share-between-phones-group
- `documentation/harmonyos-guides/07_application-services/calendarmanager-calendar-developer.md` -
  the Calendar Kit API table, the permission declaration requirement, and the
  advice to manage the calendar from `EntryAbility.ets`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/calendarmanager-calendar-developer
- `documentation/harmonyos-guides/04_system/request-user-authorization-second.md` -
  the second-stage `requestPermissionOnSetting` flow this sample omits.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization-second
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `usedScene`
  with its `abilities` and `when` parameters, mandatory for `user_grant`
  permissions.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- `documentation/harmonyos-guides/03_application-framework/uiability-lifecycle.md` -
  `onWindowStageWillDestroy` as the place to unsubscribe from WindowStage
  events.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/uiability-lifecycle
- `documentation/harmonyos-references/02_application-framework/js-apis-url.md` -
  "Modules to Import: `import { url } from '@kit.ArkTS';`".
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-url
- `documentation/harmonyos-references/02_application-framework/js-apis-permissionrequestresult.md` -
  `authResults` and `dialogShownResults`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-permissionrequestresult
- `documentation/harmonyos-references/06_application-services/share-harmony-share.md` - the share entry point
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/share-harmony-share
