---
id: SHOP-04
title: Daily check-in and points - a self-closing ComponentContent dialog, a preferences-backed reminder switch, and a TextTimer browse task
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/04_sign.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/sign-0000002284183885
sample: huawei_industry_tree/16_shopping/downloads/Sign.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.NotificationKit", "@kit.PerformanceAnalysisKit"]
apis: [ComponentContent, wrapBuilder, "UIContext.getPromptAction", openCustomDialog, closeCustomDialog, DialogAlignment, "preferences.getPreferences", put, flush, get, "notificationManager.publish", requestEnableNotification, NotificationRequest, TextTimer, TextTimerController, onTimer, Progress, ProgressType, Toggle, ToggleType, Navigation, NavPathStack, NavDestination, pushPathByName, "@Provide", "@Consume", "@StorageLink", "UIContext.px2vp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-16-0005, HW-16-0013]
status: verified-with-fixes
---

## When to use

Load this card when you are building a **daily engagement loop**: a check-in
strip that pays points, a reminder the user opts into, and a timed task that
completes while the user browses. It is the loyalty furniture of shopping,
news, fitness and reading apps, and this sample carries all three pieces at
once.

Three techniques are worth taking away, and they are independent of the loyalty
theme. **A dialog opened as `ComponentContent` and closed on a timer** is the
right shape for any transient confirmation the user must not have to dismiss.
**A `Toggle` whose truth lives in `preferences` and whose *enabledness* lives
in a system permission state** is the correct pattern for every "remind me"
switch - the toggle must not lie about a notification the OS will refuse to
show. And **a `TextTimer` with `isCountDown` plus a `Progress` bar** is the
cheapest honest way to run a foreground dwell timer.

The load-bearing constraint the sample gets right is that the reminder switch
is *disabled*, not merely off, when notifications are refused, and tapping it
explains why. Copy that. A switch that flips but does nothing is worse than one
that cannot flip.

## Feature checklist

- A 我的 (mine) tab showing a profile header and two rows, 今日任务 (today's
  tasks) and 签到排名 (check-in ranking).
- Tapping 今日任务 pushes the sign-in page onto a `NavPathStack`.
- The sign-in page shows a points counter, a 积分规则 (points rules) chip, and
  a large round 签到 button.
- Tapping 签到 opens a "signed in successfully" dialog that closes itself after
  500 ms, adds the day's points, and flips the button to 已签到.
- A seven-segment strip shows the reward per day; segments before the current
  index are filled, the rest are pale.
- The heading switches from 今日签到可得10积分 to 明日签到可得20积分, and the
  streak counter from 0 to 1 day.
- A 连续签到提醒 (streak reminder) switch persists through `preferences`, and
  publishes a text notification on the next entry when it is on.
- The switch is disabled when notification permission was refused; tapping it
  then raises a 无通知权限！(no notification permission) toast.
- A 浏览我的页面 (browse my page) task with a 去完成 button; pressing it
  returns to the mine tab and reveals a floating capsule with a `Progress` bar
  and a five-second countdown.
- When the countdown reaches zero the task shows 已完成 and 10 points are
  added.

## Architecture

One `entry` module, two pages, two small components, three models. The whole
feature is ~880 lines of ArkUI with no service layer - the only I/O is
`preferences` and one notification publish.

```
entry/src/main/ets
├── component
│   ├── CustomSlider.ets          one segment of the 7-day strip (+points over a date)
│   └── DialogBuilder.ets         @Builder function for the check-in success dialog
├── constants/Constants.ets       sizes, indices, notification text, db name
├── datasource/MineData.ets       the two rows of the mine tab
├── entryability/EntryAbility.ets full screen, avoid areas -> AppStorage, loads MainPage
├── entrybackupability/
├── model
│   ├── DialogClass.ets           the dialog's single payload field
│   ├── MineModel.ets             Physiological { image, title, icon }
│   └── SliderModel.ets           SliderModel { value, day }
└── pages
    ├── MainPage.ets              @Entry: Navigation + 3 tabs + the browse-task capsule
    └── SignIn.ets                the sign-in NavDestination
```

**The documented tree does not match the zip**: doc line 101 lists
`pages/Mine.ets // 我的页面`, which does not exist - the mine tab is the third
`TabContent` inside `MainPage.ets` (`HW-16-0005`).

**The design decision worth copying** is that the countdown task is owned by
`MainPage`, not by `SignIn`, even though the user starts it from `SignIn`.
`MainPage` `@Provide`s `isTask`, `isFinished` and `point`; `SignIn` `@Consume`s
them and only *sets* `isTask = true` before popping itself off the stack. The
timer, the interval and the `Progress` all live on the page that stays
on screen, which is the only page that can honestly measure "time spent
browsing".

The decision **worth avoiding** is the pairing of a `TextTimer` with a
`setInterval` that ticks `progressValue` independently. Two clocks with no
common source drift; the sample papers over it by slamming
`progressValue = fullValue` when `onTimer` reports five seconds. Derive the bar
from the timer's own `elapsedTime` instead and delete the interval.

## Implementation steps

1. **Build the dialog content as an exported `@Builder` function**, not a
   `@CustomDialog` struct, so it can be wrapped by `wrapBuilder` and handed to
   `ComponentContent` with a typed payload object.
2. **Construct `ComponentContent` fresh on each open**, passing
   `this.getUIContext()`, the wrapped builder, and a `DialogClass` carrying the
   points for that day. Do not reuse a node you have already closed.
3. **Open through `UIContext.getPromptAction().openCustomDialog(node, options)`,
   inside a `try/catch`** - `openCustomDialog` throws on bad arguments, and the
   document's snippet cuts off before the `catch`, so pasting it does not
   compile (`HW-16-0005`, `HW-16-0013`).
4. **Close on a timer, not on a button**: a `setTimeout` of 500 ms that calls
   `closeCustomDialog(this.contentNode)`, credits the points and advances the
   day index.
5. **Read the reminder switch's stored value in `aboutToAppear`** with
   `preferences.getPreferences` then `get('switch', 0)`; write it in `onChange`
   with `put` + `flush`. `flush` is what makes it survive a kill.
6. **Ask for notification permission from the host page's `onPageShow`** with
   `notificationManager.requestEnableNotification`, and publish the result -
   `'success'` or `'fail'` - into `AppStorage`, so any page can bind its UI to
   it with `@StorageLink`.
7. **Bind the `Toggle`'s `isOn` to `stored === 1 && permission === 'success'`
   and its `enabled` to the permission alone**, then put the "why is this
   greyed out" toast on the *wrapping* `Row`, because a disabled `Toggle`
   receives no taps.
8. **Publish the reminder once per app run**, guarding on a `@Provide`d `play`
   flag that is set to `false` on first publish - otherwise every navigation
   back into the page fires another notification.
9. **Run the browse task with `TextTimer({ isCountDown: true, count })` and a
   controller**, start it from `onNavBarStateChange` / `onPageShow`, pause it
   in `onPageHide`, and award the points in `onTimer`.

## Verified snippets

All snippets are from `Sign.zip`. Corrected forms are marked.

**The self-closing dialog — `entry/src/main/ets/pages/SignIn.ets`** (as shipped)

```typescript
import { ComponentContent } from '@kit.ArkUI';
import { SignInSuccessDialogBuilder } from '../component/DialogBuilder';
import { DialogClass } from '../model/DialogClass';

uiContext = this.getUIContext();
promptAction = this.uiContext.getPromptAction();
private contentNode: ComponentContent<DialogClass> | null = null;
private currentSliderModel: SliderModel | null = null;

// ... inside the 签到 button's onClick:
.onClick(() => {
  //签到之后，短暂弹窗，自动关闭
  this.currentSliderModel =
    this.arr[this.index === 7 ? this.arr.length - 1 : this.index] as SliderModel;
  this.contentNode = new ComponentContent((this.uiContext), wrapBuilder(SignInSuccessDialogBuilder),
    new DialogClass(this.currentSliderModel.value));
  try {
    this.promptAction.openCustomDialog(this.contentNode, {
      alignment: DialogAlignment.Center,
    });
    setTimeout(() => {
      this.closeDialog();
      this.isClick = true;
    }, 500);
  } catch (error) {
    let message = (error as BusinessError).message;
    let code = (error as BusinessError).code;
    hilog.error(DOMAIN_NUMBER, TAG, `OpenCustomDialog args error code is ${code}, message is ${message}`);
  }
});

closeDialog = () => {
  this.promptAction.closeCustomDialog(this.contentNode);
  this.point += this.currentSliderModel!.value;
  if (this.index <= 6) {
    this.index++;
  }
};
```

**`ComponentContent` is the imperative escape hatch from `@CustomDialog`.** The
three constructor arguments are the whole point: a `UIContext` (so the node
attaches to the right window instance), a `wrapBuilder` around a *global*
`@Builder` function (a member builder will not do - `wrapBuilder` only accepts
a globally declared one), and one payload object. Because the builder is
generic over `DialogClass`, the dialog can render `+${dialogClass.data}` for
whatever the day's reward happens to be without any shared state.

The `try` guards `openCustomDialog`, which throws error 100001 on an invalid
node. The document's snippet ends at the closing brace of the `try` block and
never shows the `catch` - a `try` with no `catch` and no `finally` is a syntax
error, so the printed code cannot compile (`HW-16-0005`); it is one instance of
a corpus-wide abridgement defect (`HW-16-0013`).

Note that `closeDialog` credits the points and advances `index`, so the dialog's
dismissal - not the tap - is the transaction boundary. And note the day index
clamp: `this.index === 7 ? this.arr.length - 1 : this.index`, needed because
`index` is allowed to reach 7 while `arr` holds seven entries.

**The reminder switch — same file** (as shipped)

```typescript
@StorageLink('notification') permission: string = '';
@State isNoticed: number = 0;

async aboutToAppear() {
  await this.getPreferencesFromStorage();
  this.isNoticed = await this.getPreference();
  if (this.isNoticed === 1 && this.play) {
    this.play = false;                                  // publish once per run
    notificationManager.publish(notificationRequest, (err: BusinessError) => {
      if (err) {
        hilog.error(DOMAIN_NUMBER, TAG,
          `Failed to publish notification. Code is ${err.code}, message is ${err.message}`);
      }
    });
  }
}

async putPreference(data: number) {
  if (preferenceTheme !== null) {
    await preferenceTheme.put('switch', data);
    await preferenceTheme.flush();
  }
}

// in build():
Row() {
  Toggle({
    type: ToggleType.Switch,
    isOn: this.isNoticed === 1 && this.permission === Constants.SUCCESS
  })
    .enabled(this.permission === Constants.SUCCESS)
    .onChange((isOn: boolean) => {
      if (isOn) {
        this.putPreference(1);
      } else {
        this.putPreference(0);
      }
    });
}
.onClick(() => {
  if (this.permission !== Constants.SUCCESS) {
    this.getUIContext().getPromptAction().showToast({
      message: $r('app.string.can_not_notice')       // 无通知权限！
    });
  }
});
```

**Two sources of truth, deliberately kept apart.** `isNoticed` is what the user
asked for and lives in a `preferences` db (`notification.db`, key `switch`);
`permission` is what the system allows and arrives through `AppStorage` from
`MainPage.onPageShow`'s `requestEnableNotification`. `isOn` is the *conjunction*
- the switch shows on only when both agree - while `enabled` tracks the
permission alone. That is what stops the UI from promising a notification the
OS will drop.

The explanatory toast hangs on the wrapping `Row`, not the `Toggle`: a disabled
component does not receive click events, so the only way to explain the grey
switch is to catch the tap on its container. `put` followed by `flush` is the
persistence pair - `put` writes to the in-memory cache and `flush` commits it
to disk.

**The browse-task countdown — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
@State progressValue: number = 0;
@State time: number = 5000;
@State fullValue: number = 50;
@State intervalID: number = 0;
countDownTextTimerController: TextTimerController = new TextTimerController();

onPageShow(): void {
  if (this.isHide && this.isTask) {
    this.countDownTextTimerController.start();
    this.intervalID = setInterval(() => {
      this.progressValue += 1;
    }, 100);
    this.isHide = false;
  }
}

onPageHide(): void {
  this.countDownTextTimerController.pause();
  clearInterval(this.intervalID);
  this.isHide = true;
}

// in the floating capsule builder:
Progress({ value: 0, total: this.fullValue, type: ProgressType.Capsule })
  .value(this.progressValue);

TextTimer({ isCountDown: true, count: this.time, controller: this.countDownTextTimerController })
  .format('ss')
  .visibility(this.isFinished ? Visibility.None : Visibility.Visible)
  .onTimer((_utc: number, elapsedTime: number) => {
    if (elapsedTime === 5) {
      clearInterval(this.intervalID);
      this.progressValue = this.fullValue;
      this.isFinished = true;
      this.point += 10;
    }
  });
```

**`isCountDown: true` with `count: 5000` is a five-second countdown in
milliseconds**, and `format('ss')` renders it as two digits next to the
秒后完成 ("done in N seconds") label. The controller - not a state flag - is
what starts and pauses it, which is why `onPageHide` can suspend the task when
the user leaves the tab and `onPageShow` can resume it: dwell time only counts
while the page is visible.

`onTimer`'s second parameter is elapsed time in **seconds**, so
`elapsedTime === 5` is the completion edge. The `Progress` runs on its own
`setInterval` at 100 ms over a total of 50, which coincidentally also lands at
five seconds - two clocks that agree by arithmetic rather than by construction.
Deriving `progressValue` from `elapsedTime` inside `onTimer` removes the
interval, the `intervalID`, and both `clearInterval` calls.

## Permissions & config

**None declared.** `entry/src/main/module.json5` has no `requestPermissions`
block. Publishing a notification needs no `user_grant` permission on
HarmonyOS - it needs the user to have **enabled notifications for the app**,
which is what `notificationManager.requestEnableNotification(context)` asks
for, and it is called from `MainPage.onPageShow`, not from a permission
manager.

The module declares an `EntryBackupAbility` with `$profile:backup_config`, and
`main_pages.json` lists only `pages/MainPage` - `SignIn` is a `NavDestination`
reached through the `NavPathStack`, so it is deliberately not a routed page.

`EntryAbility.onCreate` pins light mode with `setColorMode(COLOR_MODE_LIGHT)`
and `onWindowStageCreate` writes `topRectHeight` / `bottomRectHeight` into
`AppStorage` as **px**; every consumer converts with `px2vp`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **The check-in state is not persisted.** `index`, `isClick` and `point` are
  plain `@State` / `@Provide` fields, so relaunching the app resets the streak
  to zero and lets the user check in again immediately. Only the *reminder
  switch* survives a restart. Any real implementation needs the check-in date
  in `preferences` (or on a server) and a comparison against today.
- The seven day labels come from `calculateTomorrowDate(n)`, which is
  `new Date()` plus n days formatted `M.D` - correct, but computed once in the
  field initialiser, so a session that spans midnight shows yesterday's labels.
- `MainPage` declares `@StorageLink('bottomRectHeight') bottomRectHeight: string`
  while `EntryAbility` stores a **number** into that key, and then calls
  `Number.parseInt` on it. The two other consumers type the same key as
  `number`. Type the link to match the writer.
- `MainPage`'s `Tabs` returns `false` from `onContentWillChange`, which vetoes
  every tab switch; the first two `TabContent` blocks are empty anyway, so the
  sample only ever shows the third tab (it opens on `index: 2`). Remove the
  callback if you add real content to the other tabs.
- The floating browse-task capsule is positioned with `margin({ top: 340 })`
  inside a full-height column - a fixed offset that will not survive a
  different screen height or a tablet.
- The notification title and body are hardcoded Chinese strings in
  `Constants.ets` (`签到提醒` / `今天不要忘记签到哦`) rather than string
  resources, so they do not localise with the rest of the UI.

## Pitfalls

- **`HW-16-0005`** (E/low, confirmed): **the document's `openCustomDialog`
  snippet does not compile and its project tree names a file that does not
  exist.** Doc lines 31-39 print a `try { ... }` with no `catch` or `finally`,
  which is invalid ArkTS; the zip's `SignIn.ets:214` has the `catch` the
  excerpt cut off. Doc line 101 lists `pages/Mine.ets`, but the zip's `pages`
  directory holds only `MainPage.ets` and `SignIn.ets`. Fix: extend the snippet
  through the catch block and regenerate the tree from the zip.
- **`HW-16-0013`** (E/medium, confirmed): **systematic — abridged doc snippets
  across this corpus are cut mid-construct and no longer parse.** This doc's
  try-without-catch is one of the catalogued instances (alongside
  `16_shopping/01`, `05`, `07`, `10`, `14`, `23` and docs in five other
  industries). The zip source is valid in every case; only the published
  excerpt is mangled. Fix: regenerate excerpts with brace-balanced elision -
  elide bodies with comments, never the `catch`, `async`, `return` or closing
  braces.

## References

- `huawei_industry_tree/16_shopping/docs/04_sign.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sign-0000002284183885
- `documentation/harmonyos-references/02_application-framework/js-apis-arkui-componentcontent.md` - `ComponentContent` and its constructor
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-arkui-componentcontent
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-promptaction.md` - `openCustomDialog`, `closeCustomDialog`, `showToast`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-promptaction
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-texttimer.md` - `isCountDown`, `count`, `format`, `onTimer`, `TextTimerController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-texttimer
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-progress.md` - `ProgressType.Capsule` and `value`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-progress
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-toggle.md` - `ToggleType.Switch`, `isOn`, `switchStyle`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-toggle
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` - `getPreferences`, `put`, `get`, `flush`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `documentation/harmonyos-references/06_application-services/js-apis-notificationmanager.md` - `publish`, `requestEnableNotification`, `NotificationRequest`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-notificationmanager
- `documentation/harmonyos-guides/07_application-services/text-notification.md` - the basic-text notification flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/text-notification
- `documentation/harmonyos-guides/03_application-framework/data-persistence-by-preferences.md` - when `preferences` is the right store
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/data-persistence-by-preferences
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - `Navigation` / `NavPathStack` / `NavDestination`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `SHOP-16` - the other countdown in this industry (ticket grabbing), with a persisted deadline
- `SHOP-05` - coupon collection, the sibling engagement loop
