---
id: OFFICE-27
title: Batch-sync to-dos into the system calendar - two-stage permission, duplicate check, Calendar Kit addEvent
industry: 05_office
doc: huawei_industry_tree/05_office/docs/27_multi_schedule.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/multi_schedule-0000002356954036
sample: huawei_industry_tree/05_office/downloads/MultiSchedule.zip
kits: ["@kit.CalendarKit", "@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["calendarManager.getCalendarManager", "CalendarManager.getCalendar", "Calendar.addEvent", "Calendar.getEvents", "calendarManager.Event", "calendarManager.EventType", "abilityAccessCtrl.createAtManager", "AtManager.requestPermissionsFromUser", "AtManager.requestPermissionOnSetting", PermissionRequestResult, "relationalStore", LazyForEach, IDataSource, Swiper, "resourceManager.getRawFileContent", "UIContext.getPromptAction", "@Provide", "@Consume", "@State"]
permissions: ["ohos.permission.READ_CALENDAR", "ohos.permission.WRITE_CALENDAR"]
min_api: 20
modules: [entry]
findings: [HW-05-0144, HW-05-0145, HW-05-0146, HW-05-0147, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when an app keeps its **own to-do or task list** and the user
should be able to **push a selection of them into the system calendar in one
action**, so the phone reminds them even when the app is closed.

Three concerns make this more than a loop over `addEvent`:

- **Two-stage permission.** `READ_CALENDAR` and `WRITE_CALENDAR` are
  `user_grant`. The sample raises the ordinary dialog, inspects `authResults`,
  and only escalates to the Settings dialog when `dialogShownResults` says no
  dialog was shown - the correct reading of that pair.
- **Idempotence.** Re-syncing must not create duplicates, so the existing events
  are read back and matched before each insert.
- **Batching.** The user ticks several to-dos and presses sync once.

Compare OFFICE-11, which writes a *single* meeting through
`CalendarManager.editEvent`; here the writes go through
`Calendar.addEvent` and there are many of them.

## Feature checklist

The application must be able to:

- Load the to-do list from bundled JSON and persist it in an RDB store.
- Show a calendar month view with a marker on days that have to-dos.
- Let the user enter a selection mode and tick several to-dos.
- Request both calendar permissions, handle a first-time denial and a
  "don't ask again" denial differently, and report a refusal.
- Stop the sync when the permission was refused.
- Convert each selected to-do into a `calendarManager.Event`.
- Skip a to-do that is already in the calendar and say so.
- Insert the rest and confirm.

## Architecture

Single `entry` HAP:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | ability entry |
| `pages/Index.ets` | `@Entry`; the month view, the to-do list, the selection state and **the batch-sync click handler** |
| `view/MyCalendarView.ets`, `component/MyCalendarViewCard.ets` | the calendar grid and one day cell |
| `utils/AboutEvent.ets` | `addEvent` - resolves the calendar and inserts one event |
| `utils/RequestCalendarPermission.ets` | the two-stage permission ladder |
| `utils/JsonDataUtils.ets` | reads `rawfile/schedule.json` into `PendingMatter[]` |
| `utils/RDBStoreUtil.ets` | relational store for the to-dos |
| `utils/CommonDataSource.ets` | `IDataSource` for `LazyForEach` |
| `utils/DateUtil.ets`, `model/DataSource.ets` | date helpers, `PendingMatter` and `HmCalendarItem` |

The to-do record and the calendar event are deliberately different shapes; the
conversion happens at the sync site:

```ts
// PendingMatter (app side)        ->   calendarManager.Event (system side)
title                             ->   title
date + ' ' + startTime            ->   startTime  (ms since epoch)
date + ' ' + endTime              ->   endTime
                                       type: EventType.NORMAL
// location, priority, planContent, type have no counterpart and are dropped
```

The bundled data uses `"date": "2026/01/30"` and `"startTime": "11:50"`, so the
conversion is a string concatenation fed to `new Date(...)`.

Sync flow:

```
sync button onClick
  requestCalendarPermission(...)              -> boolean
    requestPermissionsFromUser([READ, WRITE])
    authResults        all 0      -> userGrant
    dialogShownResults any false  -> not first time
    if (!firstTime && !granted) requestPermissionOnSetting(...)
    else if (firstTime && !granted) toast
  calendarManager.getCalendarManager(context) -> calendarMgr

  for each ticked index
    build Event
    calendarMgr.getCalendar(cb)
      calendar.getEvents(cb)                  -> duplicate check by title + startTime
        addEvent(...)  ->  getCalendar().then(calendar.addEvent(event, cb))
```

## Implementation steps

1. **Declare both calendar permissions** with a complete `usedScene` including
   `"when": "inuse"` - the sample does this correctly and it matches the
   document's 权限说明 section.
2. **Run the two-stage ladder before anything else.** Request both permissions,
   check every entry of `authResults`, and read `dialogShownResults` to decide
   whether a refusal was a first-time "no" (toast and stop) or a standing "no"
   (escalate to `requestPermissionOnSetting`).
3. **Stop when the permission was refused.** Return before the loop rather than
   letting an optional chain swallow the whole batch (HW-05-0144).
4. **Resolve the calendar once per batch** and enumerate the existing events once,
   building a lookup set - not once per selected item (HW-05-0145).
5. **Convert each to-do** into a `calendarManager.Event` with
   `type: EventType.NORMAL` and 13-digit millisecond `startTime` / `endTime`.
6. **Skip duplicates** by matching on a stable pair - title plus start time here -
   and tell the user which ones were already present.
7. **Insert the rest** with `calendar.addEvent(event, cb)`, checking `err` in the
   callback (the sample's helper does this correctly).
8. **Check `err` in every AsyncCallback** before touching `data` (HW-05-0146).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### The two-stage permission ladder

`MultiSchedule.zip#entry/src/main/ets/utils/RequestCalendarPermission.ets`

```ts
export async function requestCalendarPermission(userGrant: boolean,
  uiContext: UIContext): Promise<boolean> {
  let permissions: Permissions[] = ['ohos.permission.READ_CALENDAR', 'ohos.permission.WRITE_CALENDAR'];
  let isFirstTime: boolean = true;
  let atManager = abilityAccessCtrl.createAtManager();
  let grantStatus = await atManager.requestPermissionsFromUser(uiContext.getHostContext() as Context, permissions);

  userGrant = true;                                  // parameter discarded - HW-05-0147
  for (let element of grantStatus.authResults) {
    if (element !== 0) {
      userGrant = false;
      break;
    }
  }

  if (grantStatus.dialogShownResults) {
    for (let element of grantStatus.dialogShownResults) {
      if (!element) {
        isFirstTime = false;
        break;
      }
    }
  }

  if (!isFirstTime && !userGrant) {
    const data: Array<abilityAccessCtrl.GrantStatus> =
      await atManager.requestPermissionOnSetting(uiContext.getHostContext() as Context, permissions);
    userGrant = true;
    for (let element of data) {
      if (element !== 0) {
        userGrant = false;
      }
    }
    if (!userGrant) {
      uiContext.getPromptAction().showToast({
        message: $r('app.string.need_permission'),
        duration: 2000
      });
    }
  } else if (isFirstTime && !userGrant) {
    uiContext.getPromptAction().showToast({
      message: $r('app.string.need_permission'),
      duration: 2000
    });
  }
  return userGrant;
}
```

This is the best permission ladder in the industry for this particular decision.
It reads **both** result arrays and uses them for what they mean: `authResults`
for whether the permission was granted, `dialogShownResults` for whether the
system actually showed a dialog. A first-time refusal gets a toast; a refusal
where no dialog appeared - the "already answered, don't ask again" case - escalates
to `requestPermissionOnSetting`. Compare OFFICE-01 (HW-05-0002), OFFICE-11
(HW-05-0065) and OFFICE-21 (HW-05-0123), which each get part of this wrong.

The one flaw is the dead `userGrant` parameter (HW-05-0147); the escalation also
returns the Settings result without a `catch`.

### The batch sync handler - as shipped

`MultiSchedule.zip#entry/src/main/ets/pages/Index.ets`

```ts
.onClick(async () => {
  let flag = await requestCalendarPermission(this.userGrant, this.uiContext);
  if (flag) {
    this.userGrant = true;
    let context = this.context as common.UIAbilityContext;
    this.calendarMgr = calendarManager.getCalendarManager(context);
  }
  for (let i = 0; i < this.index.length; i++) {          // runs even when flag is false - HW-05-0144
    if (this.index[i]) {
      let event: calendarManager.Event = {
        type: calendarManager.EventType.NORMAL,
        startTime: new Date(this.mattersData[i].date + '  ' + this.mattersData[i].startTime).getTime(),
        endTime: new Date(this.mattersData[i].date + '  ' + this.mattersData[i].endTime).getTime(),
        title: this.mattersData[i].title
      };

      let calendar: calendarManager.Calendar | undefined = undefined;
      this.calendarMgr?.getCalendar((err: BusinessError, data: calendarManager.Calendar) => {
        calendar = data;                                  // err unchecked - HW-05-0146
        calendar.getEvents((err: BusinessError, data: calendarManager.Event[]) => {
          let isPush = true;
          for (let j = 0; j < data.length; j++) {         // full calendar scan, per item - HW-05-0145
            if (data[j].title === event.title && data[j].startTime === event.startTime) {
              isPush = false;
            }
          }
          if (isPush) {
            addEvent(this.userGrant, this.calendarMgr, event, this.uiContext);
          } else {
            this.uiContext.getPromptAction().showToast({
              message: $r('app.string.alreadyAdd'),
              textColor: '#0A59F7',
            });
          }
        });
      });
    }
    this.editVisibility = Visibility.None;
    this.addVisibility = Visibility.None;
  }
});
```

Corrected shape - guard, resolve once, index once:

```ts
.onClick(async () => {
  const granted = await requestCalendarPermission(this.uiContext);
  if (!granted) {
    return;                                   // the ladder already toasted
  }
  this.userGrant = true;
  this.calendarMgr = calendarManager.getCalendarManager(this.context as common.UIAbilityContext);

  try {
    const calendar = await this.calendarMgr.getCalendar();
    const existing = await calendar.getEvents();
    const seen = new Set(existing.map((e) => `${e.title}|${e.startTime}`));

    let added = 0;
    let skipped = 0;
    for (let i = 0; i < this.index.length; i++) {
      if (!this.index[i]) {
        continue;
      }
      const matter = this.mattersData[i];
      const event: calendarManager.Event = {
        type: calendarManager.EventType.NORMAL,
        startTime: new Date(`${matter.date} ${matter.startTime}`).getTime(),
        endTime: new Date(`${matter.date} ${matter.endTime}`).getTime(),
        title: matter.title
      };
      if (seen.has(`${event.title}|${event.startTime}`)) {
        skipped++;
        continue;
      }
      await calendar.addEvent(event);
      added++;
    }
    this.uiContext.getPromptAction().showToast({ message: `${added} added, ${skipped} already present` });
  } catch (err) {
    hilog.error(0x0000, 'MultiSchedule', 'sync failed: %{public}s', JSON.stringify(err));
  } finally {
    this.editVisibility = Visibility.None;
    this.addVisibility = Visibility.None;
  }
});
```

One summary toast instead of one per item is part of the fix: the shipped code
raises a toast inside each per-item callback, so ticking ten to-dos queues ten
toasts.

### The single-event helper

`MultiSchedule.zip#entry/src/main/ets/utils/AboutEvent.ets`

```ts
let calendar: calendarManager.Calendar | undefined = undefined;

function permission(uiContext: UIContext) {
  uiContext.getPromptAction().showToast({
    message: $r('app.string.permission')
  });
}

export function addEvent(userGrant: boolean, calendarMgr: calendarManager.CalendarManager | null,
  event: calendarManager.Event, uiContext: UIContext) {
  if (!userGrant) {
    permission(uiContext);
    return;
  }
  calendarMgr?.getCalendar().then((data: calendarManager.Calendar) => {
    calendar = data;
    calendar.addEvent(event, (err: BusinessError, data: number): void => {
      if (err) {
        hilog.error(0x0000, 'testTag', '%{public}s', `${JSON.stringify(err)}`);
      } else {
        hilog.info(0x0000, 'testTag', '%{public}s', `${data}`);
        uiContext.getPromptAction().showToast({
          message: $r('app.string.success'),
          textColor: '#0A59F7',
        });
      }
    });
  }).catch((err: BusinessError) => {
    hilog.info(0x0000, 'testTag', '%{public}s', `${JSON.stringify(err)}`);
  });
}
```

The `addEvent` callback here is the shape to copy - `if (err)` first, the new
event id in `data` on success - and the promise carries a `.catch`. Note that
`calendar` is a **module-level** variable reused across calls, and that the helper
resolves the calendar again even though the caller has just done so.

`Calendar.addEvent` returns the created event id, which is what a real
implementation would store on the `PendingMatter` so the to-do can later be
updated or removed from the calendar; this sample logs it and discards it.

## Permissions & config

`MultiSchedule.zip#entry/src/main/module.json5`

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.READ_CALENDAR",
    "reason": "$string:reason_read_calendar",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  },
  {
    "name": "ohos.permission.WRITE_CALENDAR",
    "reason": "$string:reason_write_calendar",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
]
```

Notes on the config:

- The declared set matches the document's 权限说明 section exactly, both entries
  carry `"when": "inuse"`, and each has its **own** reason string - better than
  the samples that reuse `EntryAbility_desc` for every permission.
- `READ_CALENDAR` is genuinely needed here, not just `WRITE_CALENDAR`: the
  duplicate check calls `Calendar.getEvents`.
- Both are `user_grant`, so the runtime ladder is mandatory.
- **No storage permission** - the to-dos live in an app-private RDB store seeded
  from `rawfile/schedule.json`.
- `build-profile.json5` pins the SDK to `6.0.0(20)`.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **`Event.startTime` / `endTime` are 13-digit millisecond timestamps**, so the
  app's own date and time strings must be parsed before insertion. The sample
  concatenates `date` and `startTime` with a **double space**
  (`date + '  ' + startTime`) and does not check the result for `NaN` - worth
  normalising to a single space and validating when adapting.
- **`getEvents()` with no filter returns the whole calendar.** It is the wrong
  tool for a per-item existence check; call it once per batch.
- **The duplicate key is title + startTime.** Two genuinely distinct to-dos that
  share a title and a start minute will be treated as one.
- **`getCalendar()` resolves the default calendar account.** Unlike OFFICE-01,
  this sample does not create a named account, so the events land in whatever the
  system considers default.
- **`dialogShownResults` is what distinguishes the two refusals.** `false` means
  the system did not show a dialog because the permission had already been
  answered - that is the only case where `requestPermissionOnSetting` is
  appropriate.
- **Nothing links the to-do back to its calendar event.** `addEvent` returns an
  event id that the sample discards, so a to-do edited or deleted later leaves its
  calendar entry behind.

## Pitfalls

- **Running the batch loop regardless of the permission result is incorrect.**
  When the user refuses, `calendarMgr` stays `null`, the `?.` short-circuits for
  every item, the selection UI closes, and nothing is inserted or reported - the
  helper's own permission toast is unreachable because the optional chain returns
  first. Return early instead. (HW-05-0144)
- **Resolving the calendar and enumerating every event once per selected item is
  incorrect** for a batch operation: N to-dos means N full calendar reads and 2N
  calendar handle resolutions. Resolve once, build a `Set`, then loop.
  (HW-05-0145)
- **Ignoring `err` in the `getCalendar` and `getEvents` callbacks is incorrect** -
  `data` is undefined on failure and both callbacks dereference it immediately;
  the inner callback also shadows the outer `err` and `data`. The project's own
  `addEvent` helper shows the right shape. (HW-05-0146)
- **`requestCalendarPermission(userGrant, ...)` taking a grant flag it overwrites
  on line one is incorrect** - the signature implies the current state matters,
  and it does not. Drop the parameter, or honour it with a `checkAccessToken`
  pre-check. (HW-05-0147)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/06_application-services/js-apis-calendarmanager.md` -
  the Calendar Kit reference the document names for `getCalendarManager`,
  `getCalendar`, `addEvent`, `getEvents` and `Event`; it is a stub in this
  repository, so the API shapes above were read from the sample and cross-checked
  against the Calendar Kit usage in OFFICE-01 and OFFICE-11.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-calendarmanager
- `documentation/harmonyos-references/02_application-framework/js-apis-permissionrequestresult.md` -
  the meaning of `authResults` and `dialogShownResults`, which this sample reads
  correctly.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-permissionrequestresult
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` -
  `requestPermissionsFromUser` and `requestPermissionOnSetting12+` with its
  precondition that the ordinary dialog must have run first.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` and
  `request-user-authorization-second.md` - the two-stage flow this ladder
  implements.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization-second
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `usedScene`
  and `when` for user_grant permissions.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
