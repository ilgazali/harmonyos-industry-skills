---
id: OFFICE-30
title: Warning the candidate when an exam app is switched to the background - windowStage lifecycle events, banner notification, SoundPool alert and a blur overlay
industry: 05_office
doc: huawei_industry_tree/05_office/docs/30_exam_cut_backstage_tips.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/exam_cut_backstage_tips-0000002431623674
sample: huawei_industry_tree/05_office/downloads/Note4ScreenSwitch.zip
kits: ["@kit.ArkUI", "@kit.NotificationKit", "@kit.MediaKit", "@kit.AudioKit", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit", "@kit.ArkTS"]
apis: ["windowStage.on('windowStageEvent')", "window.WindowStageEventType", "window.on('windowStatusChange')", "window.WindowStatusType", "window.getWindowStatus", "notificationManager.isNotificationEnabled", "notificationManager.requestEnableNotification", "notificationManager.openNotificationSettings", "notificationManager.publish", "notificationManager.cancel", "notificationManager.cancelAll", "notificationManager.SlotType", "notificationManager.ContentType", "wantAgent.getWantAgent", "wantAgent.OperationType", "media.createSoundPool", "SoundPool.load", "SoundPool.play", "SoundPool.stop", "SoundPool.unload", "SoundPool.release", "SoundPool.on('loadComplete')", "resourceManager.getRawFd", "UIAbility.onNewWant", "@StorageLink", "@StorageProp", "@Watch", NavPathStack, "NavDestination.onShown", foregroundBlurStyle]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-05-0162, HW-05-0163, HW-05-0164, HW-05-0165, HW-05-0166, HW-05-0167, HW-05-0168, HW-05-0169, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when the app must **notice that it left the foreground and react
to it** - an online exam that forbids switching away, a signing session, a
supervised form. The pattern also covers the softer version: pause a timer, blur
sensitive content, or warn the user.

Three separate signals are combined, and that combination is the point:

| Signal | Catches |
| --- | --- |
| `windowStage.on('windowStageEvent')` | leaving via the task switcher, home, another app |
| `window.on('windowStatusChange')` | split-screen, floating window, minimise |
| `foregroundBlurStyle` on the page | hides the paper the instant either fires |

Neither listener alone is enough: entering split-screen keeps the window stage
SHOWN, and being pushed to the background does not always change the window
status.

## Feature checklist

The application must be able to:

- Ask for notification permission and open the banner-notification settings
  panel on first entry to the exam page.
- Track foreground/background state in `AppStorage` from the ability.
- Treat split-screen, floating and minimised as "left the exam" too.
- Blur the exam page whenever either condition holds.
- On each switch: decrement the allowance, publish a banner notification saying
  how many are left, and play a recorded voice warning.
- Make tapping the banner return to the exam page (cold and hot start).
- Auto-submit once the allowance runs out, and cancel the notification.
- Only warn on the exam page, never on the main page.
- Cancel all notifications when the app exits.

## Architecture

Single `entry` HAP, one `Navigation` stack with two `NavDestination` pages:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup, `windowStageEvent` + `windowStatusChange` listeners, `onNewWant` deep-link handling |
| `pages/NavigationPage.ets` | `@Entry`; owns the `NavPathStack`, maps names to pages, handles the "go to ExamPage" request |
| `pages/MainPage.ets` | exam home - progress, start button, notification permission request |
| `pages/ExamPage.ets` | the paper; `@Watch` on both state flags, the counter, the notification and the sound |
| `utils/NoteUtils.ets` | `ServerInteraction` (mock backend), `NotificationUtil`, `SoundPoolUtil` |
| `utils/DisplayUtil.ets` | avoid-area heights out of `AppStorage` |
| `common/*.ets`, `viewmodel/NoteData.ets` | constants and the exam data model |

State flows entirely through `AppStorage`, which is what lets an ability-level
listener drive a component-level reaction:

```
EntryAbility                         AppStorage            ExamPage
  on('windowStageEvent')  ---->  'isOnForeground'  ---->  @StorageLink @Watch
     SHOWN/RESUMED  -> true                                 isOnForegroundChanged()
     HIDDEN/PAUSED  -> false                                  counter--
                                                              notification + sound
  on('windowStatusChange') ---> 'windowStatusType' --->  @StorageLink @Watch
     FLOATING/SPLIT_SCREEN/                                 isWindowStatusTypeChanged()
     MINIMIZE/UNDEFINED                                       -> writes isOnForeground = false
  onNewWant(targetPage)   ---->  'nameForNavi'     ---->  NavigationPage.onPageShow
                                                            popToName / pushPath
```

The second watcher deliberately writes into the first watcher's state instead of
duplicating the notification logic - one place decides what "in the background"
means, and the guard `if (this.isOnForeground) { this.isOnForeground = false; }`
keeps a single switch from being counted twice.

Returning from the banner works for both start modes: cold start reads
`want.parameters.targetPage` in `onWindowStageCreate`, hot start reads it in
`onNewWant`, and both write `'nameForNavi'` for `NavigationPage.onPageShow` to
act on.

## Implementation steps

1. **Publish the foreground flag from the ability.** Register
   `windowStageEvent` and map SHOWN/RESUMED to `true`, HIDDEN/PAUSED to `false`.
   On an API 20 project prefer `windowStageLifecycleEvent`, which is the event
   the window guide recommends when transition order matters (HW-05-0162).
2. **Register `windowStatusChange` as well** and publish the status, so
   split-screen and floating are detected.
3. **Unregister both in `onWindowStageWillDestroy`** (HW-05-0163).
4. **Request notification permission on the main page**, and follow it with
   `openNotificationSettings` so the user can turn banners on - a plain
   permission grant does not give you a banner.
5. **Build the `WantAgent` once**, targeting `EntryAbility` with
   `parameters: { targetPage: 'ExamPage' }`, and attach it to the notification.
6. **Watch both flags on the exam page.** Only act when `examParam.isInExam` is
   true, so the main page never warns.
7. **On each switch**: decrement, submit when the allowance is exhausted -
   counting so that the fifth switch submits if you promised five (HW-05-0164) -
   otherwise persist, play the sound and publish the notification.
8. **Create the SoundPool before it can be needed**, awaiting it, and only play
   after `loadComplete` (HW-05-0165).
9. **Blur the page** with `foregroundBlurStyle` driven by both flags.
10. **Release the SoundPool in `aboutToDisappear`** and cancel notifications in
    `onWindowStageDestroy`.

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### Publishing the foreground flag

`Note4ScreenSwitch.zip#entry/src/main/ets/entryability/EntryAbility.ets`

```ts
try {
  windowStage.on('windowStageEvent', (data) => {
    // 根据事件状态类型选择进行相应的处理
    if (data === window.WindowStageEventType.SHOWN) {
      // 应用进入前台，默认为可交互状态
      AppStorage.setOrCreate<boolean>('isOnForeground', true);
    } else if (data === window.WindowStageEventType.HIDDEN) {
      // 应用进入后台，默认为不可交互状态
      AppStorage.setOrCreate<boolean>('isOnForeground', false);
    } else if (data === window.WindowStageEventType.PAUSED) {
      // 前台应用进入多任务，转为不可交互状态
      AppStorage.setOrCreate<boolean>('isOnForeground', false);
    } else if (data === window.WindowStageEventType.RESUMED) {
      // 进入多任务后又继续返回前台时，恢复可交互状态
      AppStorage.setOrCreate<boolean>('isOnForeground', true);
    }
  });
} catch (exception) {
  hilog.error(DOMAIN, 'testTag', 'Failed to enable the listener for window stage event changes. Cause: %{public}s',
    JSON.stringify(exception));
}
```

The four-state mapping is right, and PAUSED - the multitask view - is correctly
treated as "gone". What is missing is the matching `off`, and on an API 20
project the newer event is the recommended one:

```ts
onWindowStageCreate(windowStage: window.WindowStage): void {
  this.windowStage = windowStage;
  windowStage.on('windowStageLifecycleEvent', (data: window.WindowStageLifecycleEventType) => { /* ... */ });
}

onWindowStageWillDestroy(windowStage: window.WindowStage): void {
  try {
    windowStage.off('windowStageLifecycleEvent');
    this.windowClass?.off('windowStatusChange');
  } catch (err) {
    hilog.error(DOMAIN, 'testTag', 'Failed to unsubscribe. Code is %{public}d', (err as BusinessError).code);
  }
}
```

### Publishing the window status

`Note4ScreenSwitch.zip#entry/src/main/ets/entryability/EntryAbility.ets`

```ts
try {
  windowStage.getMainWindow().then((windowClass) => {
    AppStorage.setOrCreate<WindowStatusType>('windowStatusType', windowClass.getWindowStatus());
    windowClass.on('windowStatusChange', (status) => {
      hilog.info(DOMAIN, 'testTag', 'current window status is %{public}d', status);
      AppStorage.setOrCreate<WindowStatusType>('windowStatusType', status);
    });
  }).catch((err: BusinessError) => {
    hilog.error(DOMAIN, 'testTag', 'Failed to get main window. Cause: %{public}s', JSON.stringify(err));
  });
} catch (exception) {
  hilog.error(DOMAIN, 'testTag', 'Failed to enable the listener for window status changes. Cause: %{public}s',
    JSON.stringify(exception));
}
```

The initial value is seeded before the listener is attached, which is the right
order - but it is seeded from an asynchronous `then`, which is why the exam
page's initialiser for the same key can still read `undefined` (HW-05-0167).

### The two watchers

`Note4ScreenSwitch.zip#entry/src/main/ets/pages/ExamPage.ets`

```ts
@StorageLink('isOnForeground') @Watch('isOnForegroundChanged') isOnForeground: boolean = true;
@StorageLink('windowStatusType') @Watch('isWindowStatusTypeChanged') windowStatusType: window.WindowStatusType =
  AppStorage.get('windowStatusType') as window.WindowStatusType;      // may be undefined - HW-05-0167
@State isWindowStatusTypeNormal: boolean = true;

// 应用屏幕状态，当分屏或浮窗时算作切后台
isWindowStatusTypeChanged() {
  if (this.windowStatusType === window.WindowStatusType.FLOATING ||
    this.windowStatusType === window.WindowStatusType.SPLIT_SCREEN ||
    this.windowStatusType === window.WindowStatusType.MINIMIZE ||
    this.windowStatusType === window.WindowStatusType.UNDEFINED
  ) {
    this.isWindowStatusTypeNormal = false;
    if (this.isOnForeground) {
      this.isOnForeground = false;
    }
  } else {
    this.isWindowStatusTypeNormal = true;
    if (!this.isOnForeground) {
      this.isOnForeground = true;
    }
  }
}

// 应用进入前后台监听处理逻辑
isOnForegroundChanged() {
  if (!this.isOnForeground && this.examParam.isInExam) { // 处于考试进行中时，若切后台则发送通知
    this.examParam.leftSwitchScreenCounts = this.examParam.leftSwitchScreenCounts - CommonConstant.NUM_1;
    if (this.examParam.leftSwitchScreenCounts < CommonConstant.NUM_0) { // 当切后台剩余次数小于0，则自动提交考试结束
      this.submitExam();
      return;
    }
    ServerInteraction.sendExamParamToServer(this.examParam); // 向服务端更新剩余切后台次数
    this.notificationInputParam.text = '剩余' + this.examParam.leftSwitchScreenCounts.toString() + '次';
    SoundPoolUtil.PlaySoundPool(); // 播放自定义录制提示音
    NotificationUtil.setNotificationWithBanner(this.notificationInputParam); // 弹出用户通知
  }
}
```

The `if (this.isOnForeground)` / `if (!this.isOnForeground)` guards inside the
status watcher are what stop one physical switch from being counted twice when
both listeners fire - copy them.

The counting itself is off by one against what the candidate is told (`< 0`
after the decrement allows six switches, not five). Corrected:

```ts
this.examParam.leftSwitchScreenCounts--;
if (this.examParam.leftSwitchScreenCounts <= CommonConstant.NUM_0) {
  this.submitExam();
  return;
}
```

### Blurring the page

`Note4ScreenSwitch.zip#entry/src/main/ets/pages/ExamPage.ets`

```ts
.foregroundBlurStyle(!this.isOnForeground || !this.isWindowStatusTypeNormal ? BlurStyle.Thin : BlurStyle.NONE, {
  colorMode: ThemeColorMode.LIGHT,
  adaptiveColor: AdaptiveColor.DEFAULT
});
```

Driving it off **both** flags rather than only `isOnForeground` is deliberate:
in split-screen the app is still technically foreground.

### Notification permission, then the banner settings panel

`Note4ScreenSwitch.zip#entry/src/main/ets/utils/NoteUtils.ets`

```ts
static requestEnableNotification() {
  let context = NotificationUtil.ctx.getHostContext() as common.UIAbilityContext;
  notificationManager.isNotificationEnabled().then((data: boolean) => {
    if (!data) {
      notificationManager.requestEnableNotification(context).then(() => {
        // 弹出设置横幅通知的半模态面板。
        notificationManager.openNotificationSettings(context).then(() => {
          hilog.info(DOMAIN, TAG, 'openNotificationSettings success');
        }).catch((err: BusinessError) => {
          hilog.error(DOMAIN, TAG,
            'openNotificationSettings failed, code is %{public}d, message is %{public}s', err.code, err.message);
        });
      }).catch((err: BusinessError) => {
        if (NotificationConstant.NOTIFICATION_DISABLE_CODE === err.code) {
          hilog.error(DOMAIN, TAG, 'requestEnableNotification refused, code is %{public}d, message is %{public}s',
            err.code, err.message);
        } else { /* ... */ }
      });
    }
  }).catch((err: BusinessError) => { /* ... */ });
}
```

The three-step shape - check `isNotificationEnabled`, request only if it is off,
then open the settings panel so the user can pick *banner* style - is the part
worth reusing, along with treating error code **1600004** (notification
disabled) as a user refusal rather than a failure.

### Publishing the warning

`Note4ScreenSwitch.zip#entry/src/main/ets/utils/NoteUtils.ets`

```ts
static setNotificationWithBanner(inputParam: NotificationParam) {
  let notificationRequest: notificationManager.NotificationRequest = {
    id: inputParam.id,
    updateOnly: inputParam.updateOnly,
    content: {
      notificationContentType: inputParam.notificationContentType,
      normal: {
        title: inputParam.title,
        text: inputParam.text,
        additionalText: inputParam.additionalText,
      }
    },
    notificationSlotType: inputParam.notificationSlotType,
    wantAgent: inputParam.wantAgent,
  };
  notificationManager.publish(notificationRequest, (err: BusinessError) => {
    if (err) {
      hilog.error(DOMAIN, TAG, 'Failed to publish notification. Code is %{public}d, message is %{public}s', err.code,
        err.message);
      return;
    }
    hilog.info(DOMAIN, TAG, 'Succeeded in publishing notification.');
  });
}
```

A fixed `id` (`NOTIFICATION_ID = 20250825`) means each new warning replaces the
previous one instead of stacking, and it is the id `cancelById` uses on submit.
`SlotType.SERVICE_INFORMATION` is the slot that can raise a banner.

### The WantAgent that brings the candidate back

`Note4ScreenSwitch.zip#entry/src/main/ets/pages/ExamPage.ets`

```ts
let wantAgentInfo: wantAgent.WantAgentInfo = {
  wants: [
    {
      deviceId: '',
      bundleName: this.bundleName,
      abilityName: 'EntryAbility',
      action: '',
      entities: [],
      uri: '',
      parameters: {
        targetPage: 'ExamPage' // 添加目标页面参数
      }
    }
  ],
  actionType: wantAgent.OperationType.START_ABILITY,
  requestCode: CommonConstant.NUM_0,
  wantAgentFlags: [wantAgent.WantAgentFlags.CONSTANT_FLAG]
};
wantAgent.getWantAgent(wantAgentInfo, (err: BusinessError, data: WantAgent) => {
  if (err) { /* ... */ return; }
  this.notificationInputParam.wantAgent = data;
});
```

`bundleName` comes from `AppStorage`, seeded in `EntryAbility.onCreate` with
`this.context.abilityInfo.bundleName` - not hardcoded.

### Routing the tap back to the exam page

`Note4ScreenSwitch.zip#entry/src/main/ets/entryability/EntryAbility.ets`

```ts
// 热启动
onNewWant(want: Want): void {
  const targetPage = want.parameters?.targetPage;
  if (targetPage === 'ExamPage') {
    AppStorage.setOrCreate<string>('nameForNavi', 'ExamPage');
  }
}

onWindowStageCreate(windowStage: window.WindowStage): void {
  // 冷启动跳转目标页面
  if (this.funcAbilityWant?.parameters?.targetPage === 'ExamPage') {
    AppStorage.setOrCreate<string>('nameForNavi', 'ExamPage');
  }
  // ...
}
```

`Note4ScreenSwitch.zip#entry/src/main/ets/pages/NavigationPage.ets`

```ts
onPageShow(): void {
  let somePage = AppStorage.get<string>('nameForNavi');
  if (somePage) {
    if (this.pageInfos.popToName(somePage) === PageConstant.POP_NOT_MATCH) { // pop到栈底第一个匹配名称的页面
      this.pageInfos.pushPath({ name: somePage }); // 若栈内没有该页面，则入栈该页面
    }
    AppStorage.delete('nameForNavi');
  }
}
```

`popToName` first, `pushPath` only when it reports no match - that avoids
stacking a second copy of the exam page each time the banner is tapped. Deleting
the key afterwards keeps a later unrelated `onPageShow` from re-navigating.

### SoundPool lifecycle

`Note4ScreenSwitch.zip#entry/src/main/ets/utils/NoteUtils.ets`

```ts
static async create() {
  try {
    let audioRendererInfo: audio.AudioRendererInfo = {
      usage: audio.StreamUsage.STREAM_USAGE_UNKNOWN,
      rendererFlags: SoundPoolConstant.RENDER_FLAGS
    };
    SoundPoolUtil.soundPool = await media.createSoundPool(SoundPoolConstant.MAX_STREAMS, audioRendererInfo);
    SoundPoolUtil.loadCallback();
    SoundPoolUtil.finishPlayCallback();
    SoundPoolUtil.setErrorCallback();

    let context = SoundPoolUtil.ctx.getHostContext();
    let fileDescriptor = await context!.resourceManager.getRawFd(SoundPoolConstant.NOTE_AUDIO_FILE);
    SoundPoolUtil.soundId =
      await SoundPoolUtil.soundPool!.load(fileDescriptor.fd, fileDescriptor.offset, fileDescriptor.length);
  } catch (error) { /* ... */ }
}

static async release() {
  try {
    await SoundPoolUtil.soundPool!.stop(SoundPoolUtil.streamId);
    await SoundPoolUtil.soundPool!.unload(SoundPoolUtil.soundId);
    SoundPoolUtil.setOffCallback();
    await SoundPoolUtil.soundPool!.release();
  } catch (error) { /* ... */ }
}

static async setOffCallback() {
  SoundPoolUtil.soundPool!.off('loadComplete');
  SoundPoolUtil.soundPool!.off('playFinished');
  SoundPoolUtil.soundPool!.off('error');
}
```

The teardown is the best in this industry: stop the stream, unload the sound,
unregister all three callbacks, then release the pool - in that order, each
awaited. The gap is on the other side: `create()` is called without `await` and
`play` is never gated on `loadComplete` (HW-05-0165).

## Permissions & config

**No permissions.** `Note4ScreenSwitch.zip#entry/src/main/module.json5` declares
no `requestPermissions` array. Notifications are not a permission here - they
are an enablement the user grants through
`notificationManager.requestEnableNotification`, which is why error code
**1600004** appears in `NoteConstants.ets` instead of an `authResults` check.
The alert sound is a bundled rawfile played through SoundPool, so no media
permission is involved either.

`Note4ScreenSwitch.zip#build-profile.json5`

```json5
"targetSdkVersion": "6.0.0(20)",
"compatibleSdkVersion": "6.0.0(20)",
"runtimeOS": "HarmonyOS",
"buildOption": { "strictMode": { "caseSensitiveCheck": true, "useNormalizedOHMUrl": true } }
```

`deviceTypes` is `["phone"]` only - narrower than most samples in this industry,
and consistent with a feature about a single foreground window.

Required asset:

```
entry/src/main/resources/rawfile/audio_demo_no_switch_screen.mp3
```

The document's 说明 item 1 says to re-record it and keep it under 2 s; the
SoundPool guide's own limit is 1 MB after decoding, about 5.6 s at 44.1 kHz
16-bit stereo.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later - and the project agrees, unlike
  OFFICE-29.
- **`on('windowStageEvent')` does not guarantee transition order.** The window
  guide states this outright and recommends `on('windowStageLifecycleEvent')`
  from API 20 for logic that depends on order - with the caveat that the newer
  event does not report focus gain/loss, for which `on('windowEvent')` is
  needed.
- **Background transitions differ by device.** On phones, moving the window to
  the background always transitions the UIAbility; on tablets and 2-in-1 devices
  it depends on whether the app supports phones. This sample ships
  `deviceTypes: ["phone"]`, which is the case where the behaviour is simple.
- **Notification enablement is not a permission.** `requestEnableNotification`
  can only be raised once meaningfully; a refusal returns **1600004**, and the
  banner *style* still has to be chosen by the user in the panel that
  `openNotificationSettings` opens.
- **SoundPool requires a fixed call sequence.** "During application development,
  you must subscribe to playback state changes and call the APIs in the defined
  sequence. Otherwise, an exception or undefined behavior may occur", and "call
  play after the loadComplete callback is received".
- **SoundPool decodes at most 1 MB.** Longer audio is truncated to the first
  1 MB - roughly 5.6 seconds at 44.1 kHz / 16-bit / stereo.
- **`ServerInteraction` is a mock.** Its static fields stand in for a backend;
  in a real invigilation system the remaining-switch count must be authoritative
  on the server, or the candidate can reset it by reinstalling.
- **Everything resets on restart.** 说明 item 8 says so explicitly: once the exam
  has ended, restarting the app is the only way to start over - because the
  "backend" is process memory.

## Pitfalls

- **Using `on('windowStageEvent')` on an API 20 project whose logic depends on
  transition order is incorrect.** The window guide says this API "does not
  ensure the order of lifecycle state transitions and is not recommended for use
  when the order of states matters", and recommends
  `on('windowStageLifecycleEvent')` from API 20 - which is the version both the
  document and `build-profile.json5` require. (HW-05-0162)
- **Registering `windowStageEvent` and `windowStatusChange` without ever
  unregistering them is incorrect.** The lifecycle guide names
  `onWindowStageWillDestroy` as the place to "unsubscribe from WindowStage
  events"; the sample does not implement that callback at all, so stale
  listeners from a destroyed stage keep writing `isOnForeground` and the exam
  page's `@Watch` decrements the counter once per leaked listener. (HW-05-0163)
- **`leftSwitchScreenCounts < 0` after the decrement is incorrect** when the
  document and the on-screen banner both promise five switches: the counter
  passes through 0 on the fifth, so the paper is only submitted on the sixth.
  Test `<= 0` instead. (HW-05-0164)
- **Calling `SoundPoolUtil.create()` without awaiting it, and `play` without
  waiting for `loadComplete`, is incorrect** - the guide requires the documented
  sequence and warns of "an exception or undefined behavior" otherwise. The
  sample's own comment on the `play` line says the same thing. (HW-05-0165)
- **Setting `progressNum = 100` for a finished exam and recomputing it on the
  next line is incorrect** - the assignment is dead, and the finished exam shows
  whatever share of questions was answered. (HW-05-0166)
- **`AppStorage.get('windowStatusType') as window.WindowStatusType` as a
  `@StorageLink` default is incorrect**: the key is first written from an async
  `getMainWindow().then`, so the cast can assert a type onto `undefined`, and the
  overlay is then not applied on a cold start into split-screen. `DisplayUtil` in
  the same project shows the correct guarded form. (HW-05-0167)
- **The document says to watch the AppStorage state variable
  "isOnForegroundChanged", which is incorrect** - that is the `@Watch` callback;
  the AppStorage key is `isOnForeground`, as the document's own snippet and
  step 1 both show. (HW-05-0169)
- **The document's project tree spells the directory `entrybackupablility`,
  which is incorrect** - it is `entrybackupability`, and `module.json5`'s
  `srcEntry` points at that spelling under `caseSensitiveCheck: true`.
  (HW-05-0168)

## References

Reference and guide pages used to verify this card:

- `documentation/harmonyos-guides/03_application-framework/window-overview.md` -
  "Listening for Lifecycle State Changes": the ordering caveat on
  `on('windowStageEvent')` and the API 20 recommendation of
  `on('windowStageLifecycleEvent')`, plus the per-device differences in when a
  backgrounded window transitions the UIAbility.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/window-overview
- `documentation/harmonyos-guides/03_application-framework/uiability-lifecycle.md` -
  `onWindowStageWillDestroy` as the place to "release resources obtained through
  the WindowStage and unsubscribe from WindowStage events", with the
  `off('windowStageEvent')` sample.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/uiability-lifecycle
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-windowstage.md` -
  `off('windowStageEvent')9+` and `on('windowStageLifecycleEvent')20+` /
  `off('windowStageLifecycleEvent')20+`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-windowstage
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `off('windowStatusChange')11+`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-guides/05_media/using-soundpool-for-playback.md` - the
  mandatory call sequence, the loadComplete-before-play rule, and the 1 MB
  decoded-size limit.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/using-soundpool-for-playback
- `documentation/harmonyos-references/03_system/js-apis-hilog.md` - the privacy
  identifier rules that both files here follow correctly.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
