---
id: JOBS-02
title: In-app notification authorization - ask once on entry, fall back to the notification settings sheet
industry: 12_jobs
doc: huawei_industry_tree/12_jobs/docs/02_notification_authorization_popup.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/notification_authorization_popup-0000002386582149
sample: huawei_industry_tree/12_jobs/downloads/应用内授权通知弹窗示例代码.zip
kits: ["@kit.NotificationKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["notificationManager.isNotificationEnabledSync", "notificationManager.requestEnableNotification", "notificationManager.openNotificationSettings", "UIContext.getHostContext", "@StorageProp", safeAreaPadding, "window.getWindowAvoidArea", List, Search, "UIContext.getPromptAction"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-12-0001, HW-12-0005]
status: verified-with-fixes
---

## When to use

Load this card when the app **needs notification authorization and wants to ask
for it inside its own first screen**, not at some later point buried in a
settings menu. The pattern: query the current state synchronously on entry, and
only if notifications are off raise the system authorization dialog; if that
dialog can no longer appear, open the notification settings half-modal instead.

Notifications are the whole business model of a job board — a recruiter reply,
a new match, an interview invite — so the prompt sits on the very first page
the applicant sees, next to the job list it is about. That framing generalises:
any app whose value depends on a push (delivery tracking, ticket alerts, chat)
wants the same two-step ask on the first meaningful screen rather than a cold
dialog before the user has seen anything.

**Read `HW-12-0001` before adopting the shipped flow.** The two-step ladder is
correct in shape and wrong in its stopping condition: as written it re-opens
the settings sheet on every single launch once the user has refused.

## Feature checklist

- On entering the main page, notification state is checked with no visible UI
  when notifications are already enabled.
- If notifications are off, the system authorization dialog appears once.
- If the user grants, nothing further happens.
- If the user refuses — or has refused in a previous session, so the dialog
  cannot appear at all — a second, explicit path opens the app's notification
  settings sheet.
- The second step happens at most once, or behind a user action; it does not
  fire on every launch.
- The job list, search box and filter buttons behind the dialog are fully laid
  out before the prompt appears.

## Architecture

One `entry` module. A single page, a single data file, no model layer and no
service layer — the feature is one boolean owned by the system.

```
entry/src/main/ets
├── data/JobData.ets                 JobContent interface + 4 static job rows
├── entryability/EntryAbility.ets    full-screen window, avoid areas (px) -> AppStorage
├── entrybackupability/
└── pages/MainPage.ets               @Entry: the authorization flow + the whole job list UI
```

The documented 工程目录 matches the zip exactly.

**The design decision worth copying** is that the authorization flow lives in
`aboutToAppear` of the page that benefits from it, not in `EntryAbility`. The
ability only sets up the window; the page that shows jobs is the page that asks
to notify about jobs. That keeps the ask contextual — the user sees the list
behind the dialog and understands what the notifications would be for — and it
means a second page could ask for a different capability without the ability
becoming a permission dispatcher.

**The design decision worth avoiding** is the state variable that drives the
ladder. `isDialogShown` is set to `true` only when `requestEnableNotification`
*resolves*, i.e. when the user granted. Its name says "we showed a dialog"; its
value means "we are authorized". Every reader of this page has to work that out
before they can see the bug in it.

## Implementation steps

1. **Get the ability context from the UI context**, not from a stashed global:
   `this.getUIContext().getHostContext() as common.UIAbilityContext`. Both
   notification APIs take a `UIAbilityContext` because they raise UI on its
   behalf.
2. **Gate on `isNotificationEnabledSync()`** in `aboutToAppear`. The sync form
   is what lets the whole ladder be a plain `if` with no promise chain before
   the first frame.
3. **Call `requestEnableNotification(context)` first.** It shows the system
   dialog the first time only; after a refusal it rejects immediately and shows
   nothing.
4. **Branch the catch on `err.code === 1600004`** to tell "the user refused"
   apart from a real API failure (`HW-12-0001`). The guide states this
   explicitly; the sample's catch just logs.
5. **Persist the refusal** (preferences, or any store that outlives the page)
   and never run the second step again from `aboutToAppear` (`HW-12-0001`).
6. **Put `openNotificationSettings(context)` behind an explicit user action**
   for the second ask — a banner or a row in settings. The official flow marks
   this step "(Optional)".
7. **Declare nothing in `module.json5`.** Notification authorization is not a
   `requestPermissions` entry; it is granted through these APIs alone.
8. **Pad the page with the status-bar avoid area** via `@StorageProp` +
   `safeAreaPadding`, since the ability puts the window in layout-full-screen
   mode.

## Verified snippets

All snippets are from `应用内授权通知弹窗示例代码.zip`
(`NotificationAuthorizationPopup`). Corrected forms are marked.

**The authorization ladder — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
import { notificationManager } from '@kit.NotificationKit';
import { common } from '@kit.AbilityKit';

@State isDialogShown: boolean = false;
context = this.getUIContext().getHostContext() as common.UIAbilityContext;

async aboutToAppear() {
  if (!notificationManager.isNotificationEnabledSync()) {
    // 一次授权  (first ask: the system dialog)
    await this.requestPermissions();
    if (this.isDialogShown !== true) {
      // 二次授权  (second ask: the settings half-modal)
      await this.requestPermissionsOnSetting();
    }
  }
}

async requestPermissions(): Promise<void> {
  try {
    await notificationManager.requestEnableNotification(this.context);
    this.isDialogShown = true;
    hilog.info(0x0000, 'testTag', `requestEnableNotification success`);
  } catch (err) {
    hilog.error(0x0000, 'testTag',
      `requestEnableNotification failed, code is ${err.code}, message is ${err.message}`);
  }
}
```

**Two calls carry the design, and one flag breaks it.**
`isNotificationEnabledSync()` is the cheap gate: it returns the current grant
state with no dialog and no promise, so a user who already allowed
notifications never sees any of this. `requestEnableNotification` is the only
API that can show the system dialog, and it can do so exactly once per install
— after a refusal it rejects with `1600004` and draws nothing.

`isDialogShown` is the flag that breaks it. It is set inside the `try`, so it
is `true` only on a grant. On the second launch after a refusal,
`requestEnableNotification` rejects instantly, `isDialogShown` is still
`false`, and the page opens the settings sheet — unprompted, every launch. In
the very first session it is worse: the user taps Deny and the settings sheet
appears on top of the dialog they just dismissed.

**The corrected ladder — same file** (corrected, see `HW-12-0001`)

```typescript
import { preferences } from '@kit.ArkData';
import { BusinessError } from '@kit.BasicServicesKit';

@State isGranted: boolean = false;          // FIX: was isDialogShown, which meant "granted" anyway
private userRefused: boolean = false;

async aboutToAppear() {
  if (notificationManager.isNotificationEnabledSync()) {
    this.isGranted = true;
    return;
  }
  const store = await preferences.getPreferences(this.context, 'notify');
  this.userRefused = store.getSync('userRefused', false) as boolean;
  if (this.userRefused) {
    return;                                  // FIX: never auto-open the settings sheet again
  }
  await this.requestPermissions();
}

async requestPermissions(): Promise<void> {
  try {
    await notificationManager.requestEnableNotification(this.context);
    this.isGranted = true;
  } catch (error) {
    const err = error as BusinessError;
    if (err.code === 1600004) {              // FIX: the documented "user refused" code
      this.userRefused = true;
      const store = await preferences.getPreferences(this.context, 'notify');
      store.putSync('userRefused', true);
      await store.flush();
    } else {
      hilog.error(0x0000, 'testTag',
        `requestEnableNotification failed, code is ${err.code}, message is ${err.message}`);
    }
  }
}
```

`1600004` is the hinge. Without it a refusal and a genuine API failure are the
same event, and the only thing the page can conclude is "not granted" — which
is why the shipped code keeps retrying. With it, the refusal is a terminal
state the app records once and respects afterwards, and
`requestPermissionsOnSetting()` moves behind a button the user chooses to press.

Note also that the shipped `catch (err)` reads `err.code` off an untyped catch
binding; the reference flow casts to `BusinessError` first, which is what makes
the code comparison type-safe.

**The settings fallback — same file** (as shipped)

```typescript
async requestPermissionsOnSetting(): Promise<void> {
  try {
    await notificationManager.openNotificationSettings(this.context);
    hilog.info(0x0000, 'testTag', `openNotificationSettings success`);
  } catch (err) {
    hilog.error(0x0000, 'testTag',
      `openNotificationSettings failed, code is ${err.code}, message is ${err.message}`);
  }
}
```

`openNotificationSettings` opens a **half-modal sheet over the app**, not the
system Settings app, so the user stays in context and returns to the job list
by dismissing it. That is exactly why it is the sanctioned second ask: it is
cheap enough to offer from a dismissible banner, and it is the only route left
once the one-shot dialog is spent. The method itself is fine as written — the
defect is only in who calls it and how often.

**Full-screen layout and the safe area — same file plus `EntryAbility.ets`** (as shipped)

```typescript
// EntryAbility.onWindowStageCreate
let type = window.AvoidAreaType.TYPE_SYSTEM;
let avoidArea = windowClass.getWindowAvoidArea(type);
let topRectHeight = avoidArea.topRect.height;          // px
AppStorage.setOrCreate('topRectHeight', topRectHeight);
windowClass.on('avoidAreaChange', (data) => {
  if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
    AppStorage.setOrCreate('topRectHeight', data.area.topRect.height);
  } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
    AppStorage.setOrCreate('bottomRectHeight', data.area.bottomRect.height);
  }
});

// MainPage.build()
@StorageProp('topRectHeight') topRectHeight: number = 0;
// ...
.safeAreaPadding({ top: this.getUIContext().px2vp(this.topRectHeight) })
```

The ability stores **px** and the page converts with `px2vp` at the point of
use. Keeping the conversion on the read side is the right split here: the
avoid-area API speaks px, `AppStorage` is a dumb bus, and only the component
knows it needs vp. Compare `JOBS-03`, which stores px the same way but then
tries to convert *into* its own `@StorageProp` — a local write that does not
propagate and gets stamped back to px by the next `avoidAreaChange`.

`safeAreaPadding` rather than plain `padding` is deliberate: it insets the
content while letting the background colour keep running under the status bar,
which is what full-screen layout mode is for.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions` array at all.
Notification authorization is not an `ohos.permission.*` grant — it is held per
bundle by the notification service and is only ever changed through
`requestEnableNotification` or the settings sheet. Adding a permission entry
for it would be a no-op.

`deviceTypes` is `phone`, `tablet`, `2in1`. `compatibleSdkVersion` and
`targetSdkVersion` are both `6.0.0(20)`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `requestEnableNotification` shows its dialog **once per install**. There is
  no API to reset that, so the flow must be correct the first time — you cannot
  test the happy path twice without reinstalling.
- `openNotificationSettings` needs API 13 or later;
  `isNotificationEnabledSync` needs API 12 or later.
- The job list is four static entries in `JobData.ets`; there is no network
  layer and no actual notification is ever published. This sample demonstrates
  the ask, not the send.
- The filter buttons (推荐 / 附近 / 最新) share one `onClick` on their parent
  `Row` and all raise the same toast — they are decoration, not filters.
- `EntryAbility` registers `avoidAreaChange` and never releases it in
  `onWindowStageDestroy`, and drops the promise from
  `setWindowLayoutFullScreen`. Same boilerplate weakness as the other two
  samples in this industry.

## Pitfalls

- **`HW-12-0001`** (B/medium, confirmed): the authorization ladder re-opens the
  notification settings sheet on every launch once the user has refused —
  `isDialogShown` is only set on a grant, so the `!== true` branch is taken
  forever, and in the first session the sheet appears immediately after the
  user tapped Deny. The catch blocks ignore error codes entirely, so the
  documented `1600004` refusal is indistinguishable from an API failure. Fix:
  detect `err.code === 1600004`, persist a `userRefused` flag, open
  `openNotificationSettings` at most once or only from a user action, and
  rename `isDialogShown` to `isGranted`.

## References

- `huawei_industry_tree/12_jobs/docs/02_notification_authorization_popup.md` — the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/notification_authorization_popup-0000002386582149
- `documentation/harmonyos-references/06_application-services/js-apis-notificationmanager.md` — `isNotificationEnabledSync`, `requestEnableNotification`, `openNotificationSettings`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-notificationmanager
- `documentation/harmonyos-guides/07_application-services/notification-enable.md` — the three-step official flow and the `1600004` refusal code
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/notification-enable
- `documentation/harmonyos-references/02_application-framework/js-apis-window.md` — `getWindowAvoidArea`, `avoidAreaChange`, `setWindowLayoutFullScreen`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-window
- `JOBS-03` — the same avoid-area boilerplate, consumed incorrectly
- `JOBS-01` — the index page linking this scenario to its siblings
