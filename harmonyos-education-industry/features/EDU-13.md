---
id: EDU-13
title: Course reminders in the system calendar - two-way sync between the timetable and Calendar Kit
industry: 04_education
doc: huawei_industry_tree/04_education/docs/13_class_add_schedule.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/class_add_schedule-0000002358628536
sample: huawei_industry_tree/04_education/downloads/ClassAddSchedule.zip
kits: ["@kit.CalendarKit", "@kit.ArkData", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.ArkUI"]
apis: ["calendarManager.getCalendarManager", "CalendarManager.getCalendar", "Calendar.addEvent", "Calendar.deleteEvent", "Calendar.getEvents", "calendarManager.Event", "calendarManager.EventType", reminderTime, "preferences.getPreferencesSync", putSync, getSync, hasSync, deleteSync, flush, "abilityAccessCtrl.createAtManager", requestPermissionsFromUser, requestPermissionOnSetting, dialogShownResults, "@Observed", "@Watch", "@Link"]
permissions: ["ohos.permission.READ_CALENDAR", "ohos.permission.WRITE_CALENDAR"]
min_api: 20
modules: [entry]
findings: [HW-04-0092, HW-04-0093, HW-04-0094, HW-04-0095, HW-04-0096, HW-04-0097, HW-04-0098, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card when the app should **put something into the user's system
calendar and keep it in step** - a class reminder, an exam date, a homework
deadline, a live-lesson slot.

The interesting part is the word *two-way*. Writing an event is one call; the
hard problem is that the user can delete it from the Calendar app, where your
code is not running. There is no deletion callback, so the only workable
approach is the one this sample uses:

1. Store the returned `eventId` in `preferences`, indexed by the slot it came
   from.
2. On every entry to the page, list the ids that still exist in the calendar
   (`Calendar.getEvents`) and intersect with the stored ones.
3. Anything stored but no longer present was deleted externally - reset that
   slot to "not added".

That reconciliation-on-resume pattern is the transferable idea here.

## Feature checklist

- A 20-week timetable; tapping a class opens a sheet with its details.
- The sheet offers 加入日历 with a reminder-offset picker (at time, 15, 30,
  60 minutes before).
- Adding writes a `calendarManager.Event` and stores the returned `eventId`.
- Removing deletes the event and clears the stored id.
- Deleting the event from the system Calendar app clears the in-app flag on the
  next return to the page.
- The calendar permissions are requested with a settings-sheet fallback when the
  user has refused permanently.

## Architecture

One `entry` module, one page, one list component, four utility classes.

```
entry/src/main/ets
├── common/Contants.ets            class times, durations, CALENDAR_PERMISSIONS
│                                  (documented as Constants.ets - HW-04-0097)
├── component
│   ├── ClassCard.ets              one cell in the grid
│   └── ClassList.ets              THE COMPONENT: permissions, calendar, preferences, the sheet
├── dataModel
│   ├── Course.ets                 the static timetable entry
│   └── CourseShow.ets             @Observed: course + isAddedToCalendar + eventId
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── pages/MainPage.ets             @Entry - week picker, sets shouldRefresh on page show
└── utils
    ├── CalendarUtils.ets          Course -> Event, add / delete / list ids
    ├── CourseUtils.ets            Course[] -> (CourseShow | undefined)[][]
    ├── DateUtils.ets              the 20-week date grid
    ├── PreferencesUtils.ets       the eventId array <-> CourseShow reconciliation
    ├── RawfileUtils.ets
    └── Logger.ets
```

**The state that matters is one flat number array in `preferences`.** The key
`eventId` holds `DAYS_IN_A_TERM * CLASS_IN_A_DAY` entries - one slot per class
period across the whole term - with `-1` meaning "not in the calendar". The
index arithmetic `day * CLASS_IN_A_DAY + period` is what links a grid cell to
its stored id, and it appears in four places.

**`CourseShow` is the join.** It is `@Observed` and carries the static `Course`
plus the two mutable facts: `eventId` and `isAddedToCalendar`. The 2-D
`courseShow` array is rebuilt from the raw course list on every refresh and then
re-hydrated from preferences - the array is derived, the preference is the
record.

**Two permission flows, and the sample only got one of them right.**
`requestCalendarPermission()` (tapped from the sheet) reads `dialogShownResults`
and falls back to `requestPermissionOnSetting`. `refreshList()` (fired by
`@Watch` on every page show) does none of that and simply re-requests
(`HW-04-0092`).

## Implementation steps

1. **Declare both calendar permissions** with `reason` and `usedScene`. Both are
   `user_grant`.
2. **Get the manager once per grant**:
   `calendarManager.getCalendarManager(context)`. The guide suggests holding it
   in `EntryAbility`; this sample holds it in the component.
3. **Check, then request, then fall back.** `checkAccessToken` first;
   `requestPermissionsFromUser` when missing; and when `dialogShownResults`
   reports no dialog was shown, `requestPermissionOnSetting`. Use **one** flow
   for both the automatic refresh and the explicit button (`HW-04-0092`).
4. **Use the promise overload of `getCalendar`** - `await
   calendarMgr.getCalendar()` - not the callback form wrapped in a hand-rolled
   `new Promise` (`HW-04-0094`).
5. **Build the `Event` from local time.** Construct dates with the local-time
   `Date` constructor and set the class hour with `setHours`; do not add a fixed
   millisecond offset to a UTC midnight (`HW-04-0098`).
6. **Map the reminder checkboxes to `reminderTime`**, an array of "minutes
   before" values - `[0, 15, 30, 60]`.
7. **Keep `addEvent`'s return value.** It is the `eventId`, and it is the only
   handle you will ever have on that event.
8. **Await the delete before updating the model** (`HW-04-0093`).
9. **Persist with `putSync` + `flush`,** every time (`HW-04-0095`).
10. **Reconcile on entry**: `getEvents` → ids still present → any stored id not
    in that set becomes `-1`.

## Verified snippets

All snippets are from `ClassAddSchedule.zip`. Corrected forms are marked.

**Course to calendar event — `entry/src/main/ets/utils/CalendarUtils.ets`** (as shipped)

```typescript
import { calendarManager } from '@kit.CalendarKit';

static async CourseToCalendar(course: Course | undefined, date: Date, classTime: number,
    remindTime: boolean[], calendarMgr: calendarManager.CalendarManager | null): Promise<number> {
  let eventId = -1;
  if (course) {
    const classEvent: calendarManager.Event = {
      type: calendarManager.EventType.NORMAL,
      title: course.name,
      location: { location: course.location },
      startTime: date.getTime() + CalendarUtils.classTimeToDateTime(classTime),
      endTime: date.getTime() + CalendarUtils.classTimeToDateTime(classTime) + Constants.CLASS_DURATION,
      reminderTime: CalendarUtils.setReminderTime(remindTime)   // minutes BEFORE the start
    };
    eventId = await CalendarUtils.addToCalendar(classEvent, calendarMgr);
  }
  return eventId;
}

// boolean[4] from the checkboxes -> the minutes-before array the Event wants
static setReminderTime(remindTime: boolean[]): number[] {
  const reminder: number[] = [];
  if (remindTime[0]) { reminder.push(0); }    // at the start
  if (remindTime[1]) { reminder.push(15); }
  if (remindTime[2]) { reminder.push(30); }
  if (remindTime[3]) { reminder.push(60); }
  return reminder;
}
```

**`reminderTime` is an array of offsets in minutes before `startTime`,** so one
event can carry several reminders - which is why the UI is a set of checkboxes
rather than a single choice. `0` means "at the moment the class starts".

**`EventType.NORMAL` is the right type for a timed class.** The alternative,
`IMPORTANT`, is for all-day anniversary-style entries and ignores the time
component.

**The time arithmetic is where this goes wrong.** `date` comes from
`DateUtil.getFullDate()`, which builds every day as `new Date('2025-09-01')` - a
date-only ISO string, which JavaScript parses as **UTC** midnight. Adding
`30 * 60 * 1000` therefore yields 00:30 UTC, which is 08:30 only at UTC+8. The
label the app shows for that slot is `08:30-09:15` (`HW-04-0098`).

**Adding and listing — same file** (corrected, see `HW-04-0094`)

```typescript
// FIX: the sample wraps the CALLBACK overload of getCalendar in a hand-rolled
// new Promise, twice. The promise overload exists:
//   getCalendar(calendarAccount?: CalendarAccount): Promise<Calendar>
static async addToCalendar(event: calendarManager.Event,
    calendarMgr: calendarManager.CalendarManager | null): Promise<number> {
  if (!calendarMgr) { return -1; }
  try {
    const calendar = await calendarMgr.getCalendar();     // the default calendar
    const eventId = await calendar.addEvent(event);
    return eventId;
  } catch (err) {
    Logger.error(`Failed to add event. Error: ${err}`);
    return -1;
  }
}

static async getEventsFromCalendar(
    calendarMgr: calendarManager.CalendarManager | null): Promise<number[]> {
  if (!calendarMgr) { return []; }
  try {
    const calendar = await calendarMgr.getCalendar();
    const events: calendarManager.Event[] = await calendar.getEvents();
    return events.map(e => e.id).filter((id): id is number => id !== undefined);
  } catch (error) {
    Logger.error(`Error in getEventsFromCalendar: ${error}`);
    return [];
  }
}
```

**`getCalendar()` with no argument returns the application's default calendar,**
created on first use - so there is no `createCalendar` step, and every event this
app writes lands in a calendar it owns. That matters for `getEvents()`: it lists
only *this* calendar's events, so the reconciliation cannot be confused by the
user's own appointments.

**Deleting — same file** (corrected, see `HW-04-0093`)

```typescript
// FIX: the sample's version returns void, calls the getCalendar CALLBACK overload,
// and fires deleteEvent(...).then().catch() inside it - so the caller cannot await
// it and cannot know whether it worked.
static async deleteFromCalendar(eventId: number,
    calendarMgr: calendarManager.CalendarManager | null): Promise<boolean> {
  if (!calendarMgr || eventId < 0) { return false; }
  try {
    const calendar = await calendarMgr.getCalendar();
    await calendar.deleteEvent(eventId);
    return true;
  } catch (err) {
    Logger.error(`Failed to delete event. Code: ${(err as BusinessError).code}`);
    return false;
  }
}

// caller, ClassList.ets:246-253 (corrected)
if (await CalendarUtils.deleteFromCalendar(course.eventId, this.calendarMgr)) {
  course.eventId = -1;                    // FIX: sample clears these unconditionally,
  course.isAddedToCalendar = false;       //      before the delete has even started
}
```

**Clearing `eventId` before the delete succeeds is unrecoverable.** The id is
the only handle on the event; once the app has forgotten it and persisted `-1`,
a failed delete leaves an orphan reminder in the user's calendar that the app can
no longer see or remove.

**The reconciliation — `entry/src/main/ets/utils/PreferencesUtils.ets`** (as shipped)

```typescript
static async refreshCourseShow(courseShow: (CourseShow | undefined)[][],
    dataPreferences: preferences.Preferences | null,
    calendarMgr: calendarManager.CalendarManager | null,
    isPermitted: boolean): Promise<(CourseShow | undefined)[][]> {
  // rebuild the grid with every cell reset to "not added"
  const returnCourseShow: (CourseShow | undefined)[][] = [];
  for (let i = 0; i < Constants.DAYS_IN_A_TERM; i++) {
    returnCourseShow.push([]);
    for (let j = 0; j < Constants.CLASS_IN_A_DAY; j++) {
      const tmp = courseShow[i][j];
      returnCourseShow[i].push(tmp ? new CourseShow(tmp.course, false) : undefined);
    }
  }
  if (dataPreferences?.hasSync('eventId')) {
    const idSavedInApp = dataPreferences.getSync('eventId', 'default') as number[];
    if (isPermitted) {
      const idSavedInCalendar = await CalendarUtils.getEventsFromCalendar(calendarMgr);
      // anything we remember that the calendar no longer has was deleted externally
      for (let i = 0; i < idSavedInApp.length; i++) {
        if (!idSavedInCalendar.includes(idSavedInApp[i])) {
          idSavedInApp[i] = -1;
        }
      }
    }
    for (let j = 0; j < returnCourseShow.length; j++) {
      for (let l = 0; l < returnCourseShow[j].length; l++) {
        const tmp = returnCourseShow[j][l];
        if (tmp) {
          tmp.eventId = idSavedInApp[j * Constants.CLASS_IN_A_DAY + l];
          if (tmp.eventId !== -1) {
            tmp.isAddedToCalendar = true;
          }
        }
      }
    }
  }
  return returnCourseShow;
}
```

**This is the whole two-way sync, and it is one set intersection.** There is no
deletion event from Calendar Kit, so the app compares what it remembers against
what `getEvents` currently returns and drops the difference. Doing it on every
page show is what makes an external deletion appear to propagate.

**The `isPermitted` branch matters**: without the calendar permission the stored
ids are re-applied *without* verification, so the UI keeps showing the last known
state rather than falsely resetting everything to "not added".

**`day * CLASS_IN_A_DAY + period` is the index contract** between the flat
preference array and the 2-D grid. It appears in four places across two files; a
single named helper would be safer.

**Persisting — same file** (corrected, see `HW-04-0095`, `HW-04-0096`)

```typescript
static refreshPreference(courseShow: (CourseShow | undefined)[][],
    dataPreferences: preferences.Preferences | null) {
  if (!dataPreferences) { return; }
  // FIX: the sample builds this array only in the has-key branch; the else branch
  // ignores courseShow entirely and stores -1 for every slot
  const eventIdArray: number[] =
    new Array<number>(Constants.DAYS_IN_A_TERM * Constants.CLASS_IN_A_DAY).fill(-1);
  for (let j = 0; j < courseShow.length; j++) {
    for (let l = 0; l < courseShow[j].length; l++) {
      const tmp = courseShow[j][l];
      if (tmp) {
        eventIdArray[j * Constants.CLASS_IN_A_DAY + l] = tmp.eventId;
      }
    }
  }
  if (dataPreferences.hasSync('eventId')) {
    dataPreferences.deleteSync('eventId');
  }
  dataPreferences.putSync('eventId', eventIdArray);
  dataPreferences.flush((err: BusinessError) => {        // putSync alone does not persist
    if (err) {
      Logger.error(`Failed to flush. Code:${err.code}, message:${err.message}`);
    }
  });
}
```

**`putSync` writes to memory; `flush` writes to disk.** The sample pairs them in
two places and omits the `flush` in the third - the first-run branch of
`refreshCourseShow`, which is the one that creates the key (`HW-04-0095`).

## Permissions & config

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

- Both are `user_grant`, so `reason` and `usedScene` are mandatory. This sample
  gets both right - including naming the real `EntryAbility`, unlike `EDU-11`.
- The Calendar Management guide states both are required when using Calendar Kit:
  `READ_CALENDAR` for `getEvents`, `WRITE_CALENDAR` for `addEvent`/`deleteEvent`.
- Request them together - `Constants.CALENDAR_PERMISSIONS` is the pair - so the
  user sees one dialog.
- `when: "inuse"` is correct: the reminders are fired by the system Calendar, not
  by this app, so there is no background need.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The term is hard-coded**: 20 weeks from `2025-09-01`, 7 days, 8 periods -
  `DAYS_IN_A_TERM` is 140 and the preference array is sized from it. Changing the
  term length invalidates every stored index.
- **Class times are Beijing-relative** (`HW-04-0098`): the offsets in
  `CLASS_TIME_NUMS` are measured from UTC midnight.
- Every class is 45 minutes (`CLASS_DURATION`); there is no per-course duration.
- Events are written as single occurrences, not recurring - a course taught every
  week produces one event per week that must be added individually.
- The reconciliation detects **deletion** from the system calendar, not editing:
  if the user moves an event, the id still exists and the app reports it as
  synced at the original time.
- `WEEK_DAYS` in `Contants.ets` is hard-coded Chinese, unlike every other user
  string in the project, which is a resource.

## Pitfalls

- **`HW-04-0092` — the automatic refresh re-requests the permissions on every
  page show,** with no `checkAccessToken`, no `dialogShownResults` check and no
  `requestPermissionOnSetting` fallback - all of which are implemented correctly
  thirty lines below in the same file.
- **`HW-04-0093` — `deleteFromCalendar` is fire-and-forget.** The model is
  cleared and persisted before the delete runs, and its failure is only logged,
  so a failed delete orphans the event permanently.
- **`HW-04-0094` — `getCalendar`'s promise overload is ignored** and the callback
  overload is hand-wrapped in `new Promise` twice, which is also why the delete
  path could not be awaited.
- **`HW-04-0095` — the first-run write is `putSync` with no `flush`,** so the key
  that everything else depends on may never reach disk.
- **`HW-04-0096` — `refreshPreference`'s else branch ignores its argument** and
  stores `-1` everywhere.
- **`HW-04-0097` — the constants file is `Contants.ets`,** documented as
  `Constants.ets`, with the misspelling propagated into four imports.
- **`HW-04-0098` — class times are offsets from a UTC midnight,** so the calendar
  entry matches its label only in UTC+8.
- **Do not assume Calendar Kit will tell you about external deletions.** There is
  no such callback; reconcile with `getEvents` on resume.
- **Do not lose the `eventId`.** It is the only handle on the event, and once it
  is `-1` the app cannot find or remove what it created.

## References

- `documentation/harmonyos-guides/07_application-services/calendarmanager-calendar-developer.md` - `getCalendarManager`, the `getCalendar` promise overload, and the two required permissions
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/calendarmanager-calendar-developer
- `documentation/harmonyos-guides/07_application-services/calendarmanager-event-developer.md` - `Event`, `EventType`, `reminderTime`, `addEvent`, `deleteEvent`, `getEvents`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/calendarmanager-event-developer
- `documentation/harmonyos-references/06_application-services/js-apis-calendarmanager.md` - the Calendar Kit API surface
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-calendarmanager
- `documentation/harmonyos-guides/03_application-framework/data-persistence-by-preferences.md` - `putSync` versus `flush`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/data-persistence-by-preferences
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` - `checkAccessToken`, `requestPermissionsFromUser`, `requestPermissionOnSetting`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-references/02_application-framework/js-apis-permissionrequestresult.md` - `authResults` and `dialogShownResults`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-permissionrequestresult
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `READ_CALENDAR` and `WRITE_CALENDAR`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `EDU-11` - the same permission API used with the opposite defect: a complete request flow but a wrong `usedScene`
- `EDU-04` - the timetable grid this reminder attaches to
