---
id: OFFICE-01
title: Integrated office app shell - one entry HAP over six HARs, with system-calendar and agent-powered reminders
industry: 05_office
doc: huawei_industry_tree/05_office/docs/01_practice-office-app-architecture-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1-0000001965211649
sample: huawei_industry_tree/05_office/downloads/IntegratOffice.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.ArkData", "@kit.ArkTS", "@kit.BasicServicesKit", "@kit.CalendarKit", "@kit.PerformanceAnalysisKit", "@kit.ScanKit", "@kit.TelephonyKit", "@kit.BackgroundTasksKit", "@kit.ArkWeb"]
apis: [UIAbility, "windowStage.loadContent", "windowStage.getMainWindowSync", "window.getWindowAvoidArea", "AppStorage.setOrCreate", "AppStorage.get", Navigation, NavPathStack, NavDestination, "NavPathStack.pushPathByName", "NavPathStack.pop", Tabs, TabContent, LazyForEach, IDataSource, "abilityAccessCtrl.createAtManager", "atManager.requestPermissionsFromUser", "calendarManager.getCalendarManager", "CalendarManager.getCalendar", "CalendarManager.createCalendar", "Calendar.setConfig", "Calendar.addEvent", "reminderAgentManager.publishReminder", "reminderAgentManager.cancelReminder", "notificationManager.isNotificationEnabled", "notificationManager.requestEnableNotification", "preferences.getPreferencesSync", "Preferences.putSync", "Preferences.flush", "call.makeCall", "scanBarcode.startScanForResult", "UIContext.setKeyboardAvoidMode", "web_webview.WebviewController"]
permissions: ["ohos.permission.READ_CALENDAR", "ohos.permission.WRITE_CALENDAR", "ohos.permission.PUBLISH_AGENT_REMINDER"]
min_api: 20
modules: [entry, common, dfx, business, mail, message, mine]
findings: [HW-05-0001, HW-05-0002, HW-05-0003, HW-05-0004, HW-05-0005, HW-05-0006, HW-05-0007, HW-05-0008, HW-05-0009, HW-05-0010, HW-05-0011, HW-05-0012]
status: verified-with-fixes
---

## When to use

Load this card when you are building an **enterprise office application** that
ships as a single entry HAP and needs four bottom-tab domains - conversations,
mail, corporate contacts and business services - plus **time-critical reminders
that must fire even when the app is in the background or the network is weak**.

The concrete product is an integrated office app: message list and single-chat
detail, an inbox with mail detail and compose, a company address book with
region filtering and one-tap dialling, and a business tab holding attendance
check-in, to-do approval, an in-app web page and a calendar/schedule page.

The reusable part is the shell: **one `entry` HAP, two common HARs (`common`,
`dfx`), and one HAR per business domain**, joined by a single `Navigation` +
`NavPathStack` router that is provided from `Index.ets` and consumed by every
`NavDestination` inside the feature HARs. The distinguishing capability is the
reminder layer: server push is unreliable in the background, so schedules are
written into the **system calendar (Calendar Kit)** and to-do alarms are
published through **agent-powered reminders (reminderAgentManager)**, both of
which the system - not the app - delivers.

## Feature checklist

The application must be able to:

- Boot into a four-tab home (消息 / 收件箱 / 通讯录 / 业务) whose title and overflow
  menu change with the selected tab.
- Route to every sub-page by name through a single shared `NavPathStack`, with no
  router table in `module.json5`.
- Ask the user to enable notifications at first launch, before any reminder is
  published.
- Request `READ_CALENDAR` / `WRITE_CALENDAR`, **verify the grant result**, then
  obtain or create a local calendar account and write schedules into it.
- Publish an agent-powered alarm reminder for to-do items so the system fires it
  while the app is in the background or closed.
- Persist locally created schedules through `Preferences` so they survive a
  restart.
- Show the schedule list filtered by the date selected in the calendar component,
  with an explicit empty state.
- Dial a contact number from the address book by handing the number to the system
  dialler.
- Offer a scan entry (Scan Kit default UI) from every tab's overflow menu.
- Render long lists (conversations, mail, to-dos) with `LazyForEach` over a
  shared `IDataSource` implementation.

## Architecture

Product-customisation layer - `entry` (HAP):
`entry/src/main/ets/pages/Index.ets` owns the `Navigation`, creates the single
`NavPathStack` and `@Provide`s it under the key `pageInfos`. `PagesMap` is the
`navDestination` builder: it maps a route-name string to the component exported
by a feature HAR. `entry/src/main/ets/entryability/EntryAbility.ets` loads that
page, requests notification enablement, and publishes the status-bar and
navigation-indicator heights into `AppStorage` so every HAR can pad its own
header without importing `window`.

Basic-feature layer - four HARs under `feature/`:

| HAR | Responsibility | Exported entry points |
| --- | --- | --- |
| `message` | conversation list, single chat, group creation, meeting publish, personal settings | `ConversationPage`, `ConversationDetail`, `CreateChatPage`, `MeetingPublish`, `PersonalSet` |
| `mail` | inbox, mail detail, compose | `Email`, `EmailDetail`, `EmailPage`, `InboxPage` |
| `mine` | corporate address book, contact detail, region selection | `InformationPage`, `SelectRegionPage` |
| `business` | business home, calendar/schedule, attendance check-in, to-do approval, in-app web page | `BusinessPage`, `CalendarPage`, `SignInPage`, `ToDoPage`, `WebPage` |

Common-capability layer - two HARs under `common/`: `common` exports `PageHead`,
`PageSearch` and the generic `LazyDataSource` (an `IDataSource` implementation
shared by all lazy lists); `dfx` exports `Logger`. Note the dependency
direction: `business` depends on `dfx`
(`feature/business/oh-package.json5`), and `entry` depends on `common` plus the
four feature HARs. Feature HARs never depend on each other.

Data flow for a reminder:

1. `BusinessPage.aboutToAppear` asks `ReminderService` to publish an alarm
   reminder for the nearest to-do; `ReminderService` wraps
   `reminderAgentManager.publishReminder` and returns the reminder id.
2. The user opens `CalendarPage`; the `Reminder` component embedded in its
   `.menus()` builder requests the calendar permissions, then resolves a
   `calendarManager.Calendar` (get, else create) and calls `setConfig` with
   `enableReminder: true`.
3. Adding a schedule writes a `calendarManager.Event` through
   `calendar.addEvent`, appends a `ScheduleInfo` to the in-memory array, and
   persists that array with `Preferences.putSync` + `flush`.
4. The system - Notification service for the alarm, Calendar for the event -
   raises the reminder independently of the app process.

## Implementation steps

1. **Create the module skeleton.** One `entry` HAP plus six HAR modules
   registered in the root `build-profile.json5`; give each HAR an `Index.ets`
   barrel and set `"main": "Index.ets"` in its `oh-package.json5`. Add the HARs to
   `entry/oh-package.json5` as `file:` dependencies.
2. **Provide the router once.** In `Index.ets` declare
   `@Provide('pageInfos') pageInfos: NavPathStack = new NavPathStack()`, wrap the
   `Tabs` in `Navigation(this.pageInfos)`, and pass a `@Builder PagesMap(name)` to
   `.navDestination()`. Every page inside a HAR declares
   `@Consume('pageInfos') pageInfos: NavPathStack` and navigates with
   `pushPathByName(name, param)` / `pop()`.
3. **Publish window insets at startup.** In `onWindowStageCreate`, read the avoid
   areas from `windowStage.getMainWindowSync()` and store `statusHeight` and
   `bottomHeight` in `AppStorage`; HAR headers read them with
   `AppStorage.get('statusHeight')`.
4. **Enable notifications before reminders.** Call
   `notificationManager.isNotificationEnabled()` and, when it resolves `false`,
   `notificationManager.requestEnableNotification(this.context)`. The official
   reference states that `publishReminder` "can be called only after the
   notificationManager.requestEnableNotification permission is obtained", so this
   must run before step 6.
5. **Declare every permission.** `entry/src/main/module.json5` must list
   `READ_CALENDAR`, `WRITE_CALENDAR` **and `PUBLISH_AGENT_REMINDER`**. The
   shipped sample omits the last one (HW-05-0001); without it every
   `publishReminder` call fails.
6. **Publish the agent-powered alarm.** Build a
   `reminderAgentManager.ReminderRequestAlarm` and pass it to `publishReminder`.
   Normalise the time before you do: `hour` must stay in `[0, 23]` and `minute` in
   `[0, 59]` (HW-05-0004). Keep the returned reminder id so `cancelReminder(id)`
   can revoke it.
7. **Request the calendar permissions and check the result.** Call
   `atManager.requestPermissionsFromUser(context, ['ohos.permission.READ_CALENDAR', 'ohos.permission.WRITE_CALENDAR'])`
   and inspect `result.authResults`; only continue to
   `calendarManager.getCalendarManager(context)` when every entry is `0`
   (HW-05-0002).
8. **Resolve the calendar account.** `getCalendar(account)` first; on rejection
   `createCalendar(account)`. Apply `setConfig({ enableReminder: true, color })`
   on whichever succeeds, and only then call `addEvent`.
9. **Persist and reload.**
   `preferences.getPreferencesSync(context, { name: 'mySchedule' })`,
   `putSync('schedule', scheduleData)`, then `await flush()` and show the success
   toast in the resolution handler, not before it (HW-05-0006).
10. **Filter the schedule list by the selected date.** The calendar component
    reports the picked day through a callback; format it to `yyyy-MM-dd` and
    filter the schedule array into an `@State` list, rendering an empty component
    when the result is empty. Use the corrected identifiers - the document's
    snippet is not compilable (HW-05-0008).
11. **Dial from the address book.** `call.makeCall(phone.toString(), cb)` and
    guard the callback with `if (err)` before touching `err.message`
    (HW-05-0003).
12. **Back the long lists with a lazy data source.** Reuse `common`'s
    `LazyDataSource<T>` and give each item a stable id to use as the
    `LazyForEach` key - do not call `JSON.stringify` in the key generator
    (HW-05-0007).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### Router provider and route table

`IntegratOffice.zip#entry/src/main/ets/pages/Index.ets`

```ts
@Entry
@Component
struct Index {
  @State message: string = '消息';
  @State currentTabIndex: number = 0;
  @Provide('pageInfos') pageInfos: NavPathStack = new NavPathStack();

  @Builder
  PagesMap(name: string) {
    if (name === 'InformationPage') {
      InformationPage();
    } else if (name === 'CalendarPage') {
      CalendarPage();
    } else if (name === 'ClockInPage') {
      SignInPage();
    } else if (name === 'EmailDetail') {
      EmailDetail();
    } else if (name === 'ConversationDetail') {
      ConversationDetail();
    } else if (name === 'ToDoPage') {
      ToDoPage();
    } else if (name === 'WebPage') {
      WebPage();
    }
  }

  build() {
    Navigation(this.pageInfos) {
      Column() {
        Tabs({ barPosition: BarPosition.End }) {
          TabContent() { ConversationPage(); }
          .tabBar(this.BuildTabs($r('app.media.home'), $r('app.string.tab_bar_home'), 0));
          TabContent() { Email(); }
          .tabBar(this.BuildTabs($r('app.media.fortune'), $r('app.string.tab_bar_fortunes'), 1));
          TabContent() { SelectRegionPage(); }
          .tabBar(this.BuildTabs($r('app.media.life_filled'), $r('app.string.tab_bar_life'), 2));
          TabContent() { BusinessPage(); }
          .tabBar(this.BuildTabs($r('app.media.cards_filled'), $r('app.string.tab_bar_mine'), 3));
        }
        .scrollable(false)
        .onChange((index: number) => {
          this.currentTabIndex = index;
        });
      }
      .height('100%');
    }
    .hideToolBar(true)
    .navDestination(this.PagesMap)
    .title(this.message)
    .menus(this.NavigationMenus);
  }
}
```

### Notification enablement and window insets at startup

`IntegratOffice.zip#entry/src/main/ets/entryability/EntryAbility.ets`

```ts
onWindowStageCreate(windowStage: window.WindowStage): void {
  notificationManager.isNotificationEnabled().then((data: boolean) => {
    if (!data) {
      notificationManager.requestEnableNotification(this.context).then(() => {
        hilog.info(DOMAIN_NUMBER, TAG, `[ANS] requestEnableNotification success`);
      }).catch((err: Base.BusinessError) => {
        if (1600004 === err.code) {
          hilog.info(DOMAIN_NUMBER, TAG, '[ANS] requestEnableNotification refused');
        } else {
          hilog.error(DOMAIN_NUMBER, TAG,
            `[ANS] requestEnableNotification failed, code is ${err.code}, message is ${err.message}`);
        }
      });
    }
  }).catch((err: Base.BusinessError) => {
    hilog.error(DOMAIN_NUMBER, TAG, `isNotificationEnabled fail: ${JSON.stringify(err)}`);
  });

  windowStage.loadContent('pages/Index', (err, data) => {
    if (err.code) {
      hilog.error(0x0000, 'testTag', 'Failed to load the content. Cause: %{public}s', JSON.stringify(err) ?? '');
      return;
    }
  });

  let windowClass = windowStage.getMainWindowSync();
  let statusHeight = windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM).topRect.height;
  let bottomHeight = windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR).bottomRect.height;
  AppStorage.setOrCreate('bottomHeight', bottomHeight + 'px');
  AppStorage.setOrCreate('statusHeight', statusHeight + 'px');
  AppStorage.setOrCreate('windowStage', windowStage);
}
```

### Calendar account resolution (get, else create)

`IntegratOffice.zip#feature/business/src/main/ets/common/Reminder.ets`

```ts
private calendarMgr: calendarManager.CalendarManager | null = null;
private calendar: calendarManager.Calendar | undefined = undefined;
private config: calendarManager.CalendarConfig = {
  enableReminder: true,
  color: Color.Green
};
private myCalendarAccount: calendarManager.CalendarAccount = {
  name: 'test',
  type: calendarManager.CalendarType.LOCAL,
  displayName: 'test'
};

aboutToAppear() {
  const permissions: Permissions[] = ['ohos.permission.READ_CALENDAR', 'ohos.permission.WRITE_CALENDAR'];
  let atManager = abilityAccessCtrl.createAtManager();
  atManager.requestPermissionsFromUser(this.context, permissions).then((result: PermissionRequestResult) => {
    // FIX (HW-05-0002): inspect result.authResults here and return early when any entry is not 0.
    this.calendarMgr = calendarManager.getCalendarManager(this.context);
    this.calendarMgr.getCalendar(this.myCalendarAccount).then((data: calendarManager.Calendar) => {
      this.calendar = data;
      this.calendar.setConfig(this.config).catch((err: BusinessError) => {
        hilog.error(0X0000, TAG, `Failed to set config. Code: ${err.code}, message: ${err.message}`);
      });
    }).catch(() => {
      this.calendarMgr?.createCalendar(this.myCalendarAccount).then((data: calendarManager.Calendar) => {
        this.calendar = data;
        this.calendar?.setConfig(this.config);
      }).catch((error: BusinessError) => {
        hilog.error(0X0000, TAG, `Failed to create calendar. Code: ${error.code}, message: ${error.message}`);
      });
    });
  }).catch((error: BusinessError) => {
    hilog.error(0X0000, TAG, `get Permission error, error. Code: ${error.code}, message: ${error.message}`);
  });
}
```

### Writing a schedule into the system calendar

`IntegratOffice.zip#feature/business/src/main/ets/common/Reminder.ets`

```ts
const EVENT_NOT_REPEATED: calendarManager.Event = {
  title: this.title,
  location: { location: this.location },
  type: calendarManager.EventType.NORMAL,
  // 13-digit timestamps
  startTime: this.startTime.getTime(),
  endTime: this.endTime.getTime(),
  // minutes before startTime; a negative value delays the reminder
  reminderTime: this.reminderTimeArray
};
this.calendar?.addEvent(EVENT_NOT_REPEATED).then((data: number) => {
  hilog.info(0X0000, TAG, `Succeeded in adding event, id -> ${data}`);
}).catch((err: BusinessError) => {
  hilog.error(0X0000, TAG, `Failed to addEvent. Code: ${err.code}, message: ${err.message}`);
});

let options: preferences.Options = { name: 'mySchedule' };
this.dataPreferences = preferences.getPreferencesSync(this.context, options);
this.dataPreferences.putSync('schedule', this.scheduleData);
this.dataPreferences.flush(); // FIX (HW-05-0006): await this promise before the success toast
```

### Agent-powered alarm reminder

`IntegratOffice.zip#feature/business/src/main/ets/common/ReminderService.ets`

```ts
public addReminder(alarmItem: ReminderItem, callback?: (reminderId: number) => void) {
  let reminder = this.initReminder(alarmItem);
  reminderAgent.publishReminder(reminder, (err, reminderId: number) => {
    if (callback != null) {
      callback(reminderId);
    }
    if (err) {
      Logger.error('publishReminder message ' + JSON.stringify(err));
    }
  });
}

public deleteReminder(reminderId: number) {
  reminderAgent.cancelReminder(reminderId);
}

private initReminder(item: ReminderItem): reminderAgent.ReminderRequestAlarm {
  return {
    reminderType: 2, // ReminderType.REMINDER_TYPE_ALARM
    hour: item.hour, // must be within [0, 23] - see HW-05-0004
    minute: item.minute, // must be within [0, 59]
    ringDuration: 300,
    snoozeTimes: 3,
    daysOfWeek: [],
    actionButton: [
      { title: '关闭', type: reminderAgent.ActionButtonType.ACTION_BUTTON_TYPE_CLOSE },
      { title: '稍后提醒', type: reminderAgent.ActionButtonType.ACTION_BUTTON_TYPE_SNOOZE },
    ],
    wantAgent: {
      pkgName: 'com.example.temp_demo',
      abilityName: 'EntryAbility'
    },
    title: '待办',
    content: '您有一个待办事项',
    expiredContent: 'this reminder has expired',
    snoozeContent: 'remind later',
    notificationId: 99,
    slotType: notification.SlotType.SOCIAL_COMMUNICATION
  };
}
```

### Date-filtered schedule list

`IntegratOffice.zip#feature/business/src/main/ets/mainPage/CalendarPage.ets`

```ts
@State currentDate: string = '';
@State selectItem: CJDateItem = new CJDateItem(new Date());
@State list: CalendarInfo[] = [];

getDate() {
  this.currentDate = this.getCurrentDate(); // 'yyyy-MM-dd'
  this.list = calendarInfoList.filter(v => v.date === this.currentDate);
}

getSelectItem(item: CJDateItem) {
  this.selectItem = item;
  this.getDate();
}

build() {
  NavDestination() {
    Column() {
      VariableCalendar({
        getSelectItem: (item: CJDateItem): void => this.getSelectItem(item),
        getYearAndMonth: (title: string): void => this.getTitle(title)
      });

      if (this.list.length > 0) {
        List() {
          ForEach(this.list, (item: CalendarInfo) => {
            ListItem() {
              Column() {
                Row() {
                  Image($r('app.media.time')).width(16).height(16).margin({ right: 8 });
                  Text(`${item.startTime}-${item.endTime}`);
                };
                Text(`${item.title}`).margin({ top: 8 });
              }
              .padding(10)
              .alignItems(HorizontalAlign.Start);
            };
          }, (item: CalendarInfo) => JSON.stringify(item));
        }
        .layoutWeight(1);
      } else {
        EmptyComponent({ message: '暂无数据' }).layoutWeight(1);
      }
    };
  }
  .menus(this.NavigationMenus)
  .title(this.title);
}
```

### Shared lazy data source

`IntegratOffice.zip#common/common/src/main/ets/utils/LazyDataSource.ets`

```ts
@Observed
export class LazyDataSource<T> extends BasicDataSource<T> {
  dataArray: T[] = [];

  public totalCount(): number {
    return this.dataArray.length;
  }

  public getData(index: number): T {
    return this.dataArray[index];
  }

  public pushData(data: T): void {
    this.dataArray.push(data);
    this.notifyDataAdd(this.dataArray.length - 1);
  }

  public pushArrayData(newData: ObservedArray<T>): void {
    this.clear();
    this.dataArray.push(...newData);
    this.notifyDataReload();
  }

  public deleteData(index: number): void {
    this.dataArray.splice(index, 1);
    this.notifyDataDelete(index);
  }
}
```

### Dialling from the address book

`IntegratOffice.zip#feature/mine/src/main/ets/mainPage/InformationPage.ets`

```ts
import call from '@ohos.telephony.call';
import { BusinessError } from '@ohos.base';

// as shipped - see HW-05-0003; err is undefined on success
call.makeCall((this.phone).toString(), (err: BusinessError) => {
  hilog.error(0x0000, 'testTag', '%{public}s', err.message);
});

// corrected form
call.makeCall((this.phone).toString(), (err: BusinessError) => {
  if (err) {
    hilog.error(0x0000, 'testTag', '%{public}s', err.message);
  }
});
```

## Permissions & config

`IntegratOffice.zip#entry/src/main/module.json5` - as shipped, plus the missing
reminder permission (HW-05-0001):

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "requestPermissions": [
      {
        "name": "ohos.permission.READ_CALENDAR",
        "reason": "$string:reason",
        "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
      },
      {
        "name": "ohos.permission.WRITE_CALENDAR",
        "reason": "$string:reason",
        "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
      },
      {
        // MISSING in the shipped sample - required by reminderAgentManager.publishReminder
        "name": "ohos.permission.PUBLISH_AGENT_REMINDER",
        "reason": "$string:reason",
        "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
      }
    ],
    "deviceTypes": ["phone", "tablet", "2in1"],
    "pages": "$profile:main_pages",
    "abilities": [
      {
        "name": "EntryAbility",
        "srcEntry": "./ets/entryability/EntryAbility.ets",
        "exported": true,
        "skills": [
          { "entities": ["entity.system.home"], "actions": ["action.system.home"] }
        ]
      }
    ]
  }
}
```

Notes on the config:

- `READ_CALENDAR` / `WRITE_CALENDAR` are user_grant permissions and must still be
  requested at runtime with `requestPermissionsFromUser`.
- `PUBLISH_AGENT_REMINDER` is declared only; there is no runtime dialog for it.
  It does, however, require notifications to be enabled first - see the
  `requestEnableNotification` step.
- `call.makeCall` needs no permission: it hands the number to the system dialler
  and the user presses the call button.
- `scanBarcode.startScanForResult` uses the Scan Kit default UI, so the app
  declares no camera permission of its own.
- The HAR modules carry no `requestPermissions`; all permissions live in the HAP
  `module.json5`.
- Root `build-profile.json5`: `compatibleSdkVersion` and `targetSdkVersion` are
  both `6.0.0(20)`, and the six HAR modules are registered under `modules` with
  their `srcPath` (`./common/common`, `./common/dfx`, `./feature/*`).

## Constraints

- **Build environment.** DevEco Studio 6.0.0 Release or later and HarmonyOS
  6.0.0 Release SDK or later; the sample pins `6.0.0(20)`, so API level 20 is the
  effective floor.
- **Devices.** The document scopes the app to Huawei phones, but the shipped
  `module.json5` declares `phone`, `tablet` and `2in1` (HW-05-0009). Decide which
  one you want before you publish - the two are not interchangeable.
- **Reminder agent.** `reminderAgentManager.publishReminder` is available on
  Phone, PC/2in1, Tablet, TV and Wearable, requires
  `ohos.permission.PUBLISH_AGENT_REMINDER`, and per the official reference "can
  be called only after the notificationManager.requestEnableNotification
  permission is obtained". The same reference notes an anti-abuse control
  mechanism on agent-powered reminders, so a production app must also follow the
  constraints section of the agent-powered-reminder guide.
- **Alarm time range.** `ReminderRequestAlarm.hour` is limited to `[0, 23]` and
  `minute` to `[0, 59]`.
- **Calendar account.** `getCalendar` rejects when the named account does not
  exist; `createCalendar` must complete before any `addEvent` call, which is why
  the sample nests the calls rather than running them in parallel.
- **Framework code only.** The document states explicitly that the download is
  desensitised framework code, not the full application: the mail, message and
  business pages are backed by in-memory arrays, and several overflow-menu
  entries only show a 可自行添加模板 ("add your own template") toast.
- **`ConversationViewModel`** starts `AppletAbility` and `DocumentAbility`, which
  are not declared in the sample's `module.json5`; the class is never
  instantiated, so treat it as a stub rather than a working example.

## Pitfalls

- **The document lists only the calendar permissions.** It says
  "需要获取读取或写入日历、日程的权限：ohos.permission.READ_CALENDAR和ohos.permission.WRITE_CALENDAR"
  ("Permissions to read or write calendars and events are required:
  ohos.permission.READ_CALENDAR and ohos.permission.WRITE_CALENDAR"), which is
  incomplete; the sample also publishes agent-powered reminders, so
  `ohos.permission.PUBLISH_AGENT_REMINDER` must be declared as well.
  (HW-05-0001, HW-05-0012)
- **The sample treats a resolved permission promise as a granted permission,
  which is incorrect.** `requestPermissionsFromUser` resolves even on denial;
  read `result.authResults` and only continue when every entry is `0`.
  (HW-05-0002)
- **The `call.makeCall` callback reads `err.message` unconditionally, which is
  incorrect.** On success `err` is `undefined`; guard with `if (err)` as the
  official reference sample does. (HW-05-0003)
- **`hour: hours + Math.floor(minutes / 58)` is incorrect.** Between 23:58 and
  23:59 it yields `24`, outside the documented `[0, 23]` range, and the reminder
  is rejected; normalise through total minutes and take `% 24`. (HW-05-0004)
- **Five `catch` blocks in the framework code are empty, which is incorrect.**
  `WebPage.ets:89`, `ToDoPage.ets:151`, `SignInPage.ets:411`,
  `ConversationDetail.ets:79` and `ConversationViewModel.ets:128` silently drop
  navigation-parameter and `startAbility` failures; log with `hilog.error` and
  fall back to a defined default. (HW-05-0005)
- **`Preferences.flush()` is fired and forgotten, which is incorrect.** It
  returns a `Promise<void>`; await it (or chain `.catch`) and only then show the
  success toast. (HW-05-0006)
- **All three `LazyForEach` key generators call `JSON.stringify(item)`, which is
  incorrect.** The official Code Linter rule
  `@performance/hp-arkui-no-stringify-in-lazyforeach-key-generator` names that
  exact expression as its incorrect example; use a stable id such as
  `(item) => item.id.toString()`. (HW-05-0007)
- **The document's 日历项 snippet does not compile, which is incorrect.** It
  declares `calendarInfoList` but then writes
  `this.List = CalendarInfoList.filter(...)` and reads `this.list`; the working
  form is `this.list = calendarInfoList.filter(v => v.date === this.currentDate)`
  with `@State list: CalendarInfo[] = []`. (HW-05-0008)
- **The document says the app is phone-only, which does not match the sample.**
  `entry/src/main/module.json5` declares `["phone", "tablet", "2in1"]`; align one
  side or the other before release. (HW-05-0009)
- **The document describes the `mine` HAR as holding the message list and the
  detail web page, which is incorrect.** `mine` holds `BookPage`,
  `InformationPage` and `SelectRegionPage`; the web page is
  `business/.../WebPage.ets` and the message list is
  `message/.../ConversationPage.ets`. (HW-05-0010)
- **`ConversationDetail` changes the window-wide keyboard avoid mode on entry and
  never restores it, which is incorrect.** `setKeyboardAvoidMode` applies to the
  whole main window, so restore the previous mode in `aboutToDisappear` or set it
  once in `EntryAbility`. (HW-05-0011)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/02_application-framework/js-apis-reminderagentmanager.md` -
  `publishReminder` required permission, notification prerequisite, and the
  `ReminderRequestAlarm` field ranges.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-reminderagentmanager
- `documentation/harmonyos-references/03_system/js-apis-call.md` - `call.makeCall`
  signature and the `if (err)` callback pattern.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-call
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` -
  `flush(): Promise<void>` and the `ValueType` union.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` -
  checking `authResults` after `requestPermissionsFromUser`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `documentation/harmonyos-guides/12_coding-and-debugging/ide_hp-arkui-no-stringify-lazyforeach-key.md` -
  the `LazyForEach` key-generator performance rule.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ide_hp-arkui-no-stringify-lazyforeach-key
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-lazyforeach.md` -
  `keyGenerator` semantics and the default key function.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-lazyforeach
- `documentation/harmonyos-guides/02_media/scan-scanbarcode.md` -
  `startScanForResult(this.getUIContext().getHostContext(), options)` usage.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/scan-scanbarcode
- `documentation/harmonyos-references/02_application-framework/js-apis-bundleManager-applicationInfo.md` -
  `ApplicationInfo.name` corresponds to `bundleName`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-bundlemanager-applicationinfo
