---
id: LIFE-24
title: Book a home-service slot and write it to the calendar - dialog NavDestinations wrapping bindSheet, plus Calendar Kit reminders
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/24_appointment_service_remind.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/appointment_service_remind-0000002378138101
sample: huawei_industry_tree/02_convenient_life/downloads/AppointmentServiceRemind.zip
kits: ["@kit.CalendarKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.ArkTS", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["calendarManager.getCalendarManager", "calendarManager.CalendarManager.createCalendar", "calendarManager.CalendarManager.getCalendar", "calendar.setConfig", "calendar.addEvent", "calendarManager.CalendarAccount", "calendarManager.CalendarConfig", "calendarManager.Event", "calendarManager.EventType", "calendarManager.CalendarType", "abilityAccessCtrl.createAtManager", "atManager.requestPermissionsFromUser", "atManager.checkAccessToken", "atManager.requestPermissionOnSetting", "bundleManager.getBundleInfoForSelf", Navigation, NavPathStack, "pathStack.pushPath", "pathStack.pop", PopInfo, NavDestination, NavDestinationMode, bindSheet, DismissSheetAction, SheetType, Tabs, TabsController, TabContent, List, ListItem, ForEach, Provide, Consume, Link, StorageProp, "AppStorage.setOrCreate", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.on('avoidAreaChange')", "UIContext.px2vp", "UIContext.animateTo", "PromptAction.showToast", "util.format", TextPicker, Toggle, LengthMetrics]
permissions: ["ohos.permission.WRITE_CALENDAR"]
min_api: 20
modules: [entry]
findings: [HW-02-0183, HW-02-0184, HW-02-0185, HW-02-0186, HW-02-0187, HW-02-0188, HW-02-0189, HW-02-0190, HW-02-0191, HW-02-0192, HW-02-0193, HW-02-0194, HW-02-0195, HW-02-0196, HW-02-0197, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card for **book a slot, then remind me**: the user picks a time
window from a bottom sheet, fills in the rest of an order form, and the
confirmation writes a calendar event with a reminder.

Two patterns are worth taking from it, and they are independent:

**1. A bottom sheet that behaves like a navigation destination.** Each picker is
its own `NavDestination` in `NavDestinationMode.DIALOG` whose entire content is
an empty `Text()` with a `bindSheet` attached. Pushing the route opens the
sheet; dismissing the sheet pops the route and carries the selection back
through `onPop`:

```
ServicePage (Navigation + NavPathStack)
   pushPath({ name: 'TimeSelectPage', onPop: (result) => ... })
        TimeSelectPage: NavDestination(DIALOG) { Text().bindSheet(...) }
           onWillDismiss -> dismissAction.dismiss(); pathStack.pop(param, true)
   <- onPop receives the param
```

The payoff is that the sheet gets a route, a back-gesture, and a typed return
value without the parent owning any sheet state.

**2. A calendar reminder from Calendar Kit.** One permission, one account, one
event:

```
EntryAbility  requestPermissionsFromUser(['ohos.permission.WRITE_CALENDAR'])
              -> calendarManager.getCalendarManager(context) -> AppStorage['calendarMgr']
CalendarUtils getCalendar(account) or createCalendar(account)
              -> calendar.setConfig({ enableReminder: true, color })
              -> calendar.addEvent({ title, type, startTime, endTime, reminderTime })
```

`reminderTime` is an array of minutes before the start; `[0]` means "at the
appointment time".

## Feature checklist

- [ ] `ohos.permission.WRITE_CALENDAR` declared and requested; see pitfall 12 on
      whether you also need `READ_CALENDAR`.
- [ ] The calendar **queried** before it is created (HW-02-0183).
- [ ] Slot generation using comparisons, not exact hour equality
      (HW-02-0184).
- [ ] Every relative slot bounded by the service window (HW-02-0189).
- [ ] `@Link` types matching their `@State` source exactly (HW-02-0185).
- [ ] Avoid areas read inside the `setWindowLayoutFullScreen` promise
      (HW-02-0187), and `off('avoidAreaChange')` in `onWindowStageDestroy`
      (HW-02-0186).
- [ ] A visible result when the reminder is **not** created (HW-02-0188).
- [ ] Every value the sheet collects actually included in what it pops
      (HW-02-0190).

## Architecture

| File | Role |
| --- | --- |
| `pages/ServicePage.ets` | `@Entry`. The order form: address block, the three select rows, the price bar and the order button. Owns the `NavPathStack` as `@Provide('pathStack')`. |
| `pages/TimeSelectPage.ets` | Dialog `NavDestination`. A vertical day tab strip (today / tomorrow / the day after) driving a `Tabs` whose pages are `ModalSelectWindow`. |
| `pages/ItemInfoSelectPage.ets` | Dialog `NavDestination`. Item type grid plus weight and count steppers. |
| `pages/PaySelectPage.ets` | Dialog `NavDestination`. A list of payment methods. |
| `component/ModalSelectWindow.ets` | One day's time-slot list. Builds the selectable intervals and turns a tap into a `SelectedTimeParam`. |
| `utils/CalendarUtils.ets` | Permission check, calendar resolution, event creation. Module singleton. |
| `model/DataModel.ets` | Interfaces plus the generated `NORMAL_TIME_INTERVAL` table and the static option lists. |
| `constant/CommonConstant.ets` | Durations, thresholds and the calendar title. |

The time table is generated once at module load, not per render:

```ts
export const EARLIEST_TIME: TimeModel = { hour: 8, minute: 0 };
export const LATEST_TIME: TimeModel = { hour: 20, minute: 45 };
export const NORMAL_TIME_INTERVAL: TimeInterval[] = [];

for (let start = EARLIEST_TIME.hour; start <= LATEST_TIME.hour; start++) { ... }
```

so the only interval start hours that exist are 8 through 20. Everything the
today tab does has to be written against that fact - see pitfall 2.

Routing: `main_pages.json` lists only `pages/ServicePage`; the three pickers are
`routerMap` entries with `@Builder` export functions.

## Implementation steps

Where the shipped code is wrong, the step gives the corrected version and names
the finding.

1. **Declare and request the calendar permission, then publish the manager.**

   ```json5
   "requestPermissions": [
     {
       "name" : "ohos.permission.WRITE_CALENDAR",
       "reason": "$string:reason",
       "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
     }
   ]
   ```

   ```ts
   const PERMISSIONS: Permissions[] = ['ohos.permission.WRITE_CALENDAR'];
   let atManager = abilityAccessCtrl.createAtManager();
   atManager.requestPermissionsFromUser(mContext, PERMISSIONS).then((result: PermissionRequestResult) => {
     if (!result.authResults.every((value) => value === abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED)) {
       return;
     }
     let calendarMgr = calendarManager.getCalendarManager(mContext);
     AppStorage.setOrCreate('calendarMgr', calendarMgr);
   }).catch((error: BusinessError) => { ... });
   ```

   `AppStorage` is the transport because `CalendarUtils` is a module singleton
   with no access to the ability context.

2. **Set the immersive layout and read the avoid areas in the right order**
   (HW-02-0187, HW-02-0186):

   ```ts
   this.mWindow = windowStage.getMainWindowSync();
   let uiContext = this.mWindow.getUIContext();
   this.mWindow.setWindowLayoutFullScreen(true).then(() => {
     let systemArea = this.mWindow.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM);
     AppStorage.setOrCreate('topRectHeight', uiContext.px2vp(systemArea.topRect.height));
     let navArea = this.mWindow.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR);
     AppStorage.setOrCreate('bottomRectHeight', uiContext.px2vp(navArea.bottomRect.height));
     this.mWindow.on('avoidAreaChange', (area) => { /* later changes */ });
   }).catch((err: BusinessError) => { hilog.error(...); });
   ```

   ```ts
   onWindowStageDestroy(): void {
     this.mWindow?.off('avoidAreaChange');
   }
   ```

   The sample converts px to vp at the point of storage, so the pages consume
   plain vp numbers.

3. **Generate the interval table from the service window.** Keep the bounds in
   one place so every slot rule can be written against them:

   ```ts
   for (let start = EARLIEST_TIME.hour; start <= LATEST_TIME.hour; start++) {
     if (start === LATEST_TIME.hour) {
       NORMAL_TIME_INTERVAL.push({ startTime: { hour: start, minute: 0 }, endTime: LATEST_TIME });
     } else if (start === EARLIEST_TIME.hour) {
       NORMAL_TIME_INTERVAL.push({ startTime: EARLIEST_TIME, endTime: { hour: start + 1, minute: 0 } });
     } else {
       NORMAL_TIME_INTERVAL.push({ startTime: { hour: start, minute: 0 }, endTime: { hour: start + 1, minute: 0 } });
     }
   }
   ```

4. **Drop the today tab once the service day is over.**

   ```ts
   .onReady(() => {
     let nowTime = new Date();
     if (nowTime.getHours() > LATEST_TIME.hour ||
       (nowTime.getHours() === LATEST_TIME.hour && nowTime.getMinutes() >= LATEST_TIME.minute)) {
       this.availableDay = APPOINT_DAYS.filter((appointDay) => appointDay.id !== 'today');
     } else {
       this.availableDay = APPOINT_DAYS;
     }
   })
   ```

5. **Build today's remaining slots with a comparison** (HW-02-0184). The shipped
   version matches an exact start hour and collapses whenever the current hour
   plus two is not itself a boundary:

   ```ts
   aboutToAppear() {
     if (this.dayParam.id === 'today') {
       this.timePreview.push({ startTime: { hour: -1, minute: -1 }, endTime: { hour: -1, minute: -1 } });
       const startHour = Math.max(this.time.getHours() + CommonConstant.IN_TWO_HOURS_COUNT, EARLIEST_TIME.hour);
       NORMAL_TIME_INTERVAL.forEach((timeInterval) => {
         if (timeInterval.startTime.hour >= startHour) {     // shipped: === startHour, with a found flag
           this.timePreview.push(timeInterval);
         }
       });
     } else {
       this.timePreview = NORMAL_TIME_INTERVAL;
     }
   }
   ```

   The `{ hour: -1 }` sentinel is the "within two hours" row; the render checks
   `item.startTime.hour === -1` to swap in that label.

6. **Declare the two-way binding with matching types** (HW-02-0185):

   ```ts
   // parent
   @State selectedParam: SelectedTimeParam | undefined = undefined;
   // child - must be the same type
   @Link selectedParam: SelectedTimeParam | undefined;
   ```

7. **Turn a tap into absolute epoch milliseconds.** Two paths, and both must
   respect the service window (HW-02-0189):

   ```ts
   let startDate = new Date();
   let endDate = new Date();
   startDate.setSeconds(0); startDate.setMilliseconds(0);
   endDate.setSeconds(0);   endDate.setMilliseconds(0);

   if (item.startTime.hour === -1) {
     // "within two hours" - clamp the end to LATEST_TIME instead of adding two hours blindly
     this.selectedParam.timeStr = `${this.dayParam.title} ${CommonConstant.IN_TWO_HOUR}`;
     this.selectedParam.startTime = startDate.getTime();
     this.selectedParam.endTime = endDate.getTime() + CommonConstant.TWO_HOUR_TIME;
   } else {
     if (this.dayParam.id !== 'today') {
       let newTime = startDate.getTime() + CommonConstant.ONE_DAY_TIME;
       if (this.dayParam.id === 'dayAfterTomorrow') {
         newTime += CommonConstant.ONE_DAY_TIME;
       }
       startDate = new Date(newTime);
       endDate = new Date(newTime);
     }
     startDate.setHours(item.startTime.hour);
     startDate.setMinutes(item.startTime.minute);
     endDate.setHours(item.endTime.hour);
     endDate.setMinutes(item.endTime.minute);
     this.selectedParam.startTime = startDate.getTime();
     this.selectedParam.endTime = endDate.getTime();
   }
   ```

8. **Wrap each picker in a dialog NavDestination.** The whole page is one empty
   `Text()` carrying the sheet:

   ```ts
   build() {
     NavDestination() {
       Text()
         .bindSheet($$this.isShow, this.timeSelectPanel, {
           height: $r('app.float.day_select_modal_window_height'),
           preferType: SheetType.BOTTOM,
           showClose: false,
           onWillDismiss: (dismissAction: DismissSheetAction) => {
             dismissAction.dismiss();
             this.pathStack.pop(this.selectedParam, true);
           }
         });
     }.mode(NavDestinationMode.DIALOG)
     .hideTitleBar(true)
   }
   ```

   `showClose: false` removes the system close icon; the panel draws its own,
   which sets `isShow = false` and pops. `onWillDismiss` covers the interactive
   dismissals - pull-down, side swipe, mask tap - which is the only hook that
   can pop the route when the user closes the sheet by gesture.

9. **Collect the result through `onPop`.**

   ```ts
   this.pathStack.pushPath({
     name: 'TimeSelectPage', onPop: (result: PopInfo) => {
       if (!result || !result.result) {
         return;
       }
       this.timeParam = result.result as SelectedTimeParam;
       this.appointTime = this.timeParam.timeStr;
     }
   });
   ```

   Whatever the sheet pops must contain everything it collected - the shipped
   item sheet drops its count (HW-02-0190).

10. **Resolve the calendar by querying first** (HW-02-0183):

    ```ts
    const CALENDAR_ACCOUNT: calendarManager.CalendarAccount = {
      name: 'Appointment',
      type: calendarManager.CalendarType.LOCAL,
      displayName: 'Appointment'
    };
    let calendar = await this.calendarMgr.getCalendar(CALENDAR_ACCOUNT)
      .catch(() => this.createCalendar(CALENDAR_ACCOUNT));   // shipped: createCalendar every time
    ```

    ```ts
    async createCalendar(account: calendarManager.CalendarAccount) {
      const CONFIG: calendarManager.CalendarConfig = { enableReminder: true, color: Color.Orange };
      let calendar = await this.calendarMgr.createCalendar(account);
      await calendar.setConfig(CONFIG);
      return calendar;
    }
    ```

    `setConfig` is not optional in practice: the guide notes a newly created
    calendar is black by default and may display poorly in dark mode.

11. **Add the event.**

    ```ts
    const EVENT: calendarManager.Event = {
      title: title,
      type: calendarManager.EventType.NORMAL,
      startTime: startTime,
      endTime: endTime,
      reminderTime: notifyTime      // CommonConstant.NORMAL_REMIND_TIME = [0]
    };
    await calendar?.addEvent(EVENT);
    ```

12. **Report both outcomes at the call site** (HW-02-0188). `addCalendarEvent`
    resolves `false` when the permission is missing or the context is gone, and
    the shipped caller only reacts to `true`:

    ```ts
    CalendarUtils.addCalendarEvent(CommonConstant.CALENDAR_TITLE, this.timeParam.startTime,
      this.timeParam.endTime, CommonConstant.NORMAL_REMIND_TIME)
      .then((res: boolean) => {
        showToast(res ? $r('app.string.appoint_success_tip') : $r('app.string.appoint_failed_tip'),
          this.getUIContext());
      })
      .catch((err: BusinessError) => {
        showToast($r('app.string.appoint_failed_tip'), this.getUIContext());
        Logger.error('add calendar event error', `${err.code} ${err.message}`);
      });
    ```

## Verified snippets

All snippets below are copied from the ZIP, not from the document.

`AppointmentServiceRemind.zip#entry/src/main/ets/model/DataModel.ets:27-45` -
the service window and the interval table generated from it:

```ts
export const EARLIEST_TIME: TimeModel = {
  hour: 8, minute: 0
};

export const LATEST_TIME: TimeModel = {
  hour: 20, minute: 45
};

export const NORMAL_TIME_INTERVAL: TimeInterval[] = [];

for (let start = EARLIEST_TIME.hour; start <= LATEST_TIME.hour; start++) {
  if (start === LATEST_TIME.hour) {
    NORMAL_TIME_INTERVAL.push({ startTime: { hour: start, minute: 0 }, endTime: LATEST_TIME });
  } else if (start === EARLIEST_TIME.hour) {
    NORMAL_TIME_INTERVAL.push({ startTime: EARLIEST_TIME, endTime: { hour: start + 1, minute: 0 } });
  } else {
    NORMAL_TIME_INTERVAL.push({ startTime: { hour: start, minute: 0 }, endTime: { hour: start + 1, minute: 0 } });
  }
}
```

`AppointmentServiceRemind.zip#entry/src/main/ets/pages/TimeSelectPage.ets:107-135` -
the dialog NavDestination whose whole body is a sheet, and the today-tab rule:

```ts
  build() {
    NavDestination() {
      Text()
        .bindSheet($$this.isShow, this.timeSelectPanel, {
          height: $r('app.float.day_select_modal_window_height'),
          preferType: SheetType.BOTTOM,
          showClose: false,
          onWillDismiss: (dismissAction: DismissSheetAction) => {
            dismissAction.dismiss();
            this.pathStack.pop(this.selectedParam, true);
          }
        });
    }.mode(NavDestinationMode.DIALOG)
    .hideTitleBar(true)
    .onReady(() => {
      let nowTime = new Date();
      // Determine whether the "Today" tab is needed based on the current time.
      if (nowTime.getHours() > LATEST_TIME.hour ||
        (nowTime.getHours() === LATEST_TIME.hour && nowTime.getMinutes() >= LATEST_TIME.minute)) {
        this.availableDay = APPOINT_DAYS.filter((appointDay) => {
          if (appointDay.id === 'today') {
            return false;
          }
          return true;
        });
      } else {
        this.availableDay = APPOINT_DAYS;
      }
    });
  }
```

`AppointmentServiceRemind.zip#entry/src/main/ets/pages/TimeSelectPage.ets:64-96` -
a day strip driving a barless vertical `Tabs` through a `TabsController`:

```ts
        Row() {
          Flex({ direction: FlexDirection.Column, alignContent: FlexAlign.Start }) {
            ForEach(this.availableDay, (day: AppointDayModel, index: number) => {
              Text(day.title)
                .width('100%')
                .height($r('app.float.day_select_row_height'))
                .fontColor(this.selectedIndex === index ? $r('app.color.normal_blue_color') : Color.Black)
                .backgroundColor(this.selectedIndex === index ? Color.White : Color.Transparent)
                .textAlign(TextAlign.Center)
                .onClick(() => {
                  this.selectedIndex = index;
                  this.tabsController.changeIndex(this.selectedIndex);
                });
            }, (day: AppointDayModel, index: number) => day.id + index);
          }.width($r('app.float.day_select_tab_width'));

          Tabs({ controller: this.tabsController }) {
            ForEach(this.availableDay, (day: AppointDayModel) => {
              TabContent() {
                ModalSelectWindow({
                  dayParam: day,
                  selectedParam: this.selectedParam,
                  currentSelectedDayId: this.currentSelectedDayId
                });
              };
            }, (day: AppointDayModel, index: number) => day.id + index);
          }
          .scrollable(false)
          .barHeight(0)
          .barWidth(0)
          .layoutWeight(1)
          .vertical(true)
          .animationDuration(0);
        }
```

`AppointmentServiceRemind.zip#entry/src/main/ets/component/ModalSelectWindow.ets:98-126` -
a tap turned into absolute epoch milliseconds, with the day offset applied
before the hours are set:

```ts
              let startDate = new Date();
              let endDate = new Date();
              startDate.setSeconds(0);
              startDate.setMilliseconds(0);

              endDate.setSeconds(0);
              endDate.setMilliseconds(0);
              if (item.startTime.hour === -1) {
                this.selectedParam.timeStr = `${this.dayParam.title} ${CommonConstant.IN_TWO_HOUR}`;
                this.selectedParam.startTime = startDate.getTime();
                this.selectedParam.endTime = endDate.getTime() + CommonConstant.TWO_HOUR_TIME;
              } else {
                let timeStr = this.getTimeIntervalStr(item);
                this.selectedParam.timeStr = `${this.dayParam.title} ${timeStr}`;
                if (this.dayParam.id !== 'today') {
                  let newTime = startDate.getTime() + CommonConstant.ONE_DAY_TIME;
                  if (this.dayParam.id === 'dayAfterTomorrow') {
                    newTime += CommonConstant.ONE_DAY_TIME;
                  }
                  startDate = new Date(newTime);
                  endDate = new Date(newTime);
                }
                startDate.setHours(item.startTime.hour);
                startDate.setMinutes(item.startTime.minute);
                endDate.setHours(item.endTime.hour);
                endDate.setMinutes(item.endTime.minute);
                this.selectedParam.startTime = startDate.getTime();
                this.selectedParam.endTime = endDate.getTime();
              }
```

`AppointmentServiceRemind.zip#entry/src/main/ets/utils/CalendarUtils.ets:36-70` -
permission gate, lazy manager recovery, account, event:

```ts
  async addCalendarEvent(title: string, startTime: number, endTime: number, notifyTime: number[]) {
    const PERMISSIONS: Permissions[] = ['ohos.permission.WRITE_CALENDAR'];
    if (!await this.checkPermissions(PERMISSIONS)) {
      this.checkAndOpenPermissionsOnSetting(PERMISSIONS);
      return false;
    }
    let ctx = this.uiContext?.getHostContext();
    if (!ctx) {
      return false;
    }
    /* If calendarMgr is undefined, retrieve it again to prevent subsequent operations from failing due to the undefined
     state.*/
    if (!this.calendarMgr) {
      if (AppStorage.has('calendarMgr')) {
        this.calendarMgr = AppStorage.get('calendarMgr') as calendarManager.CalendarManager;
      } else {
        this.calendarMgr = calendarManager.getCalendarManager(ctx);
      }
    }
    const CALENDAR_ACCOUNT: calendarManager.CalendarAccount = {
      name: 'Appointment',
      type: calendarManager.CalendarType.LOCAL,
      displayName: 'Appointment'
    };
    let calendar: calendarManager.Calendar = await this.createCalendar(CALENDAR_ACCOUNT);
    const EVENT: calendarManager.Event = {
      title: title,
      type: calendarManager.EventType.NORMAL,
      startTime: startTime,
      endTime: endTime,
      reminderTime: notifyTime,
    };
    await calendar?.addEvent(EVENT);
    return true;
  }
```

The `createCalendar` on the second-to-last line is HW-02-0183 - query first.

`AppointmentServiceRemind.zip#entry/src/main/ets/utils/CalendarUtils.ets:95-110` -
the token check, which is the correct way to test a permission without prompting:

```ts
  async checkAccessToken(permission: Permissions): Promise<abilityAccessCtrl.GrantStatus> {
    let atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
    let grantStatus: abilityAccessCtrl.GrantStatus = abilityAccessCtrl.GrantStatus.PERMISSION_DENIED;
    let tokenId: number = 0;
    try {
      let bundleInfo: bundleManager.BundleInfo =
        await bundleManager.getBundleInfoForSelf(bundleManager.BundleFlag.GET_BUNDLE_INFO_WITH_APPLICATION);
      let appInfo: bundleManager.ApplicationInfo = bundleInfo.appInfo;
      tokenId = appInfo.accessTokenId;
      grantStatus = await atManager.checkAccessToken(tokenId, permission);
    } catch (error) {
      let err: BusinessError = error as BusinessError;
      Logger.error(`Failed to check access token  ${err.code}, message is ${err.message}`);
    }
    return grantStatus;
  }
```

`AppointmentServiceRemind.zip#entry/src/main/ets/pages/ServicePage.ets:240-253` -
one click handler per select row, returning a closure that pushes the matching
route:

```ts
  getClickFunc(info: InfoSelectBar) {
    switch (info.id) {
      case SelectBarType.TIME_BAR:
        return () => {
          this.pathStack.pushPath({
            name: 'TimeSelectPage', onPop: (result: PopInfo) => {
              if (!result || !result.result) {
                return;
              }
              this.timeParam = result.result as SelectedTimeParam;
              this.appointTime = this.timeParam.timeStr;
            }
          });
        };
```

## Permissions & config

One permission, user-grant, requested at ability start:

```json5
// entry/src/main/module.json5
"requestPermissions": [
  {
    "name" : "ohos.permission.WRITE_CALENDAR",
    "reason": "$string:reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
]
```

The Calendar Kit guide asks for both calendar permissions; this sample declares
only the write one - see pitfall 12.

Also in `module.json5`:

```json5
"deviceTypes": ["phone", "tablet", "2in1"],
"pages": "$profile:main_pages",
"routerMap": "$profile:route_map",
```

Build target:

```json5
"targetSdkVersion": "6.0.0(20)",
"compatibleSdkVersion": "6.0.0(20)",
```

`oh-package.json5` has no dependencies - Calendar Kit is an SDK kit. The project
ships `base`, `dark`, `en_US` and `zh_CN` resource qualifiers.

## Constraints

- **API 20 / HarmonyOS 6.0.0 Release and later**, DevEco Studio 6.0.0 Release
  and later.
- **Calendar writes need the permission at the moment of the write.**
  `CalendarUtils` re-checks with `checkAccessToken` on every call rather than
  trusting the grant obtained at start-up, because the user can revoke it in
  Settings while the application is running.
- **`requestPermissionOnSetting` is the re-request path.** After a denial the
  system dialog will not appear again; this API opens the settings entry
  instead. It resolves with an array of `GrantStatus`, not a
  `PermissionRequestResult`.
- **A calendar account is not idempotent.** Calling `createCalendar` twice with
  the same account creates it twice; `getCalendar` is the query
  (HW-02-0183).
- **`reminderTime` is minutes before the start.** `[0]` fires at the
  appointment time itself.
- **`startTime` and `endTime` are epoch milliseconds**, so every slot has to be
  resolved to an absolute `Date` before it becomes an event.
- **Only interval start hours 8 to 20 exist** in the generated table, which is
  what makes the exact-equality match in the today path fail outside 06:00-18:00
  (HW-02-0184).
- **`getWindowAvoidArea` returns px.** This sample converts with `px2vp` before
  storing, so the pages consume vp directly.
- **`onWillDismiss` only fires on interactive dismissals** - pull-down, side
  swipe, mask tap, and the system close icon. Setting the `bindSheet` binding to
  `false` in code closes the sheet without it, which is why each panel's own
  close button has to pop the route itself.

## Pitfalls

1. **HW-02-0183 - the calendar is created again on every order.**
   `CalendarUtils.ets:60` calls `this.createCalendar(CALENDAR_ACCOUNT)`
   unconditionally, and `:77` inside it calls
   `this.calendarMgr.createCalendar(account)` with the fixed account name
   `'Appointment'`. `getCalendar` is never called anywhere in the ZIP. The
   Calendar Kit guide is explicit: "Query the account information and create a
   calendar when an exception indicating that the calendar does not exist is
   thrown. Otherwise, the calendar may be created repeatedly." Query first.

2. **HW-02-0184 - today's slot list collapses to one row outside 06:00-18:00.**
   `ModalSelectWindow.ets:33-42` computes `startHour = getHours() + 2` and then
   looks for `timeInterval.startTime.hour === startHour`, setting a `found` flag
   that gates every push. The generated table only contains start hours 8
   through 20 (`DataModel.ets:37-45`), so the equality can only succeed for
   `getHours()` between 6 and 18. At 03:00 or at 19:30 nothing after the
   sentinel row is pushed, and the today tab is still shown until 20:45
   (`TimeSelectPage.ets:124-131`). Use `>=` instead of `===` and drop the flag.

3. **HW-02-0189 - the "within two hours" slot ignores the service window.**
   `ModalSelectWindow.ets:105-108` sets `startTime` to now and `endTime` to
   now + `TWO_HOUR_TIME`, consulting neither `EARLIEST_TIME` nor `LATEST_TIME`
   although both are exported from the model file it already imports. Picked at
   20:30 it books 20:30-22:30; picked at 03:00 - which pitfall 2 makes the only
   option - it books 03:00-05:00. Both values go straight into the calendar
   event.

4. **HW-02-0185 - the `@Link` type does not match its `@State` source.**
   `ModalSelectWindow.ets:24` declares `@Link selectedParam: SelectedTimeParam;`
   while the source at `TimeSelectPage.ets:29` is
   `@State selectedParam: SelectedTimeParam | undefined = undefined;`. The
   guidance is unambiguous: "The type of the variable decorated by @Link must be
   the same as the data source type. Otherwise, an error is reported during
   compilation." The child's own first line of its click handler
   (`:92-94`, `if (!this.selectedParam) { this.selectedParam = new
   SelectedTimeParam('', 0, 0); }`) proves the value really is undefined at
   runtime. Declare the union on both sides, or initialise the parent's state.

5. **HW-02-0186 - `on('avoidAreaChange')` is never unsubscribed.**
   `EntryAbility.ets:58` subscribes inside the `loadContent` callback;
   `onWindowStageDestroy` (`:87-90`) only logs, and `off(` appears nowhere in
   the file. Because the window handle is a local inside that callback, there is
   nothing left to unsubscribe with. Keep the window on the ability.

6. **HW-02-0187 - the immersive-layout promise is dropped and the avoid areas
   are read immediately.** `EntryAbility.ets:51` is
   `mWindow.setWindowLayoutFullScreen(true);` with no `then`, no `catch` and no
   `await`, and `:52-57` read both areas on the following lines. The reference
   gives the signature as returning `Promise<void>`, and notes the call "neither
   takes effect nor returns an error" in the freeform window state - so the
   failure path is silent as well as unordered. Chain the reads.

7. **HW-02-0188 - a failed reminder produces no feedback.**
   `CalendarUtils.addCalendarEvent` resolves `false` on two paths (`:38-41`
   permission missing, `:42-45` no context) and the caller at
   `ServicePage.ets:323-327` only shows a toast when the result is `true`. The
   `appoint_failed_tip` string exists and its toast is wired to `.catch()`,
   which those two paths never reach. Declining the calendar permission - the
   ordinary case - makes the order button look broken.

8. **HW-02-0190 - the item count is collected and thrown away.**
   `ItemInfoSelectPage.ets:31` declares `@State count`, `:173-200` give it a
   working stepper and `:180-183` render it, but `:227` builds the popped
   summary from the type and the weight only:
   `` this.itemInfo = `${ITEM_TYPES[this.selectedIndex]} ${this.weight}${CommonConstant.WEIGHT_KG}` ``.
   `count` appears nowhere else in the ZIP.

9. **HW-02-0196 - the confirm button on the item sheet is inert with no
   selection.** `ItemInfoSelectPage.ets:222-231` wraps the handler in
   `if (this.selectedIndex >= 0 && ...)` with no `else`, and `selectedIndex`
   starts at `-1`. Pressing the enabled-looking capsule does nothing at all.
   Prompt, or disable with `.enabled(this.selectedIndex >= 0)`.

10. **HW-02-0197 - the send/receive tab index is not `@State`.**
    `ServicePage.ets:42` declares `selectedIndex: number = 0;` with no
    decorator, sitting in a block where every neighbouring field is decorated,
    and `:164` reads it to choose a background colour. Nothing assigns it and
    the labels have no `onClick`, so today the strip is simply fixed - but the
    first person to wire the tabs up will assign it and watch nothing happen.

11. **HW-02-0191, HW-02-0192, HW-02-0193 - three defects in the logging
    utility.** `Logger.ets:15` imports `hilog` as
    `import hilog from '@ohos.hilog';` while `EntryAbility.ets:22` in the same
    project uses the form the reference documents,
    `import { hilog } from '@kit.PerformanceAnalysisKit';`. `Logger.ets:28` is
    `this.format.toUpperCase();` - a discarded no-op that would break the
    privacy identifiers if it were ever assigned. And `Logger.ets:23` declares
    `'%{public}s, %{public}s'` while all four methods pass `args` as one array,
    against the reference's rule that "the number and type of parameters must
    map to the identifier in the format string". Also `CalendarUtils.ets:120-122`
    logs `'requestPermissions2 success'` inside the branch guarded by
    `if (!res.every(...PERMISSION_GRANTED))` - that is, only when it failed.

12. **HW-02-0194 - the permission story does not agree with itself.** The
    document's 权限说明 names only
    `ohos.permission.WRITE_CALENDAR`; the Calendar Kit guide says "When using
    Calendar Kit, declare the ohos.permission.READ_CALENDAR and
    ohos.permission.WRITE_CALENDAR permissions in the module.json5 file"; and
    `EntryAbility.ets:71` comments "Obtain read and write permissions for the
    calendar" directly above `:73`, which requests write only. This sample never
    reads, so write alone may be right - but nothing in the material says so,
    and the comment says otherwise.

## References

- Document:
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/appointment_service_remind-0000002378138101
- Calendar Management guide (permissions, create vs query, setConfig):
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/calendarmanager-calendar-developer
- @ohos.calendarManager reference:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-calendarmanager
- Sheet transition / `bindSheet` (`onWillDismiss` semantics):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
- Navigation:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-basic-components-navigation
- @Link (type must match the data source):
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-link
- Window (`setWindowLayoutFullScreen`, `getWindowAvoidArea`,
  `on`/`off('avoidAreaChange')`):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- hilog (import path, format-argument mapping):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
- ohos.permission.WRITE_CALENDAR:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/permissions-for-all-user#ohospermissionwrite_calendar
