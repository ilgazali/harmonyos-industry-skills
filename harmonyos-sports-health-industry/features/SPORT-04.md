---
id: SPORT-04
title: Workout plans on a custom calendar - RdbStore persistence plus Calendar Kit sync
industry: 03_sports_health
doc: huawei_industry_tree/03_sports_health/docs/04_calendar_schedule_events.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/calendar_schedule_events-0000002274539357
sample: huawei_industry_tree/03_sports_health/downloads/CalendarScheduleEvents.zip
kits: ["@kit.CalendarKit", "@kit.ArkData", "@kit.AbilityKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: ["calendarManager.getCalendarManager", "CalendarManager.getCalendar", "Calendar.addEvent", "calendarManager.Event", "relationalStore.getRdbStore", "relationalStore.StoreConfig", "relationalStore.SecurityLevel", "RdbStore.execute", "RdbStore.insert", "RdbStore.batchInsert", "RdbStore.query", "relationalStore.RdbPredicates", "relationalStore.ConflictResolution", "abilityAccessCtrl.requestPermissionsFromUser", GridRow, GridCol, bindSheet, "@State", "@Prop"]
permissions: ["ohos.permission.READ_CALENDAR", "ohos.permission.WRITE_CALENDAR"]
min_api: 20
modules: [entry]
findings: [HW-03-0018, HW-03-0019, HW-03-0020, HW-03-0021, HW-03-0022, HW-03-0055]
status: verified-with-fixes
---

## When to use

Load this card when the app **keeps its own schedule and wants it on the
user's system calendar too** - workout plans here, but the same shape covers
medication reminders, lesson timetables, appointment booking, shift rosters.

Three parts are worth taking:

- **A calendar grid built from `GridRow` with `columns: 7`** rather than the
  system `CalendarPicker`, so each day cell can carry the app's own markers.
- **`RdbStore` for the plans**, with a `StoreConfig` naming the database and
  its security level - the only relational-store example in the industries
  reviewed so far.
- **Calendar Kit for the system event**: `getCalendarManager` →
  `getCalendar()` → `addEvent(event)`.

The two stores are deliberately separate: the app's database is the source of
truth for its own UI, and the system calendar is a copy the user sees
elsewhere. Getting that division right is what lets the plan still work when
the calendar permission is refused - which is the design this sample sets up
and then undermines (`HW-03-0022`).

## Feature checklist

- A month grid, seven columns, with the selected day highlighted.
- Tapping a day opens a sheet to add a plan: title, location, start and end
  time, priority, notes.
- The plan is written to the app's own `RdbStore`.
- The same plan is added to the system calendar as an event.
- The selected day lists the plans stored for it.
- The end time defaults to one hour after the start.

## Architecture

One `entry` module. The split between the two stores runs through the whole
file layout.

```
entry/src/main/ets
├── component
│   ├── MyCalendarViewCard.ets      one day cell
│   ├── MatterItem.ets              one plan row
│   └── PhysiologicalSheet.ets      the add-plan sheet: permission, RDB write, calendar write
├── entryability/EntryAbility.ets
├── entrybackupability/
├── model
│   ├── DataModel.ets               HmCalendarItem and friends
│   └── DataSource.ets              CommonConstants + the CREATE TABLE SQL + PendingMatter
├── pages
│   ├── Index.ets
│   └── MyCalendar.ets              @Entry - owns the selected date and the query
├── utils
│   ├── AboutEvent.ets              addEvent: the Calendar Kit call
│   ├── DateUtil.ets
│   ├── RDBStoreUtil.ets            a singleton RdbStore wrapper
│   └── RequestCalendarPermission.ets
└── view
    ├── MyCalendarView.ets
    ├── MyPlan.ets
    └── Physiological.ets
```

The documented tree matches the zip.

**`RDBStoreUtil` is exported as a singleton** - `export default new RDBStoreUtil()` -
so the page and the sheet share one open store rather than each opening the
database. That is the right shape; what it does on open is not
(`HW-03-0018`).

**The write path forks after the sheet is confirmed:**

```
confirm
├── RDBStoreUtil.insertNote(...)        -> the app's own list, always
└── addEvent(userGrant, calendarMgr, event, uiContext)
                    │
                    └── userGrant false -> a toast, and nothing written
```

`addEvent` taking the grant flag as its first parameter is a good pattern: the
calendar write cannot happen without the caller having proved authorisation,
and the "no permission" path is a visible toast rather than a silent skip.

## Implementation steps

1. **Declare both calendar permissions** with their own reason strings and
   `when: "inuse"` - the sample gets this right.
2. **Request them at the point of the calendar write**, not when the form
   opens (`HW-03-0022`), and derive the answer only from `authResults`
   (`HW-03-0021`).
3. **Open the store once** with a `StoreConfig` and create the table with
   `IF NOT EXISTS`; seed sample data only when the table is empty
   (`HW-03-0018`).
4. **Build the month with `GridRow({ columns: 7 })`** and a `GridCol` per day.
5. **Open the add-plan form with `bindSheet`**, taking cancel and confirm as
   callbacks.
6. **Compute the default end time by adding an hour to the timestamp**, not to
   the hour field (`HW-03-0020`).
7. **Collect the query result**, do not assign it in a loop
   (`HW-03-0019`).

## Verified snippets

All snippets are from `CalendarScheduleEvents.zip`. Corrected forms are marked.

**Opening the store — `entry/src/main/ets/utils/RDBStoreUtil.ets`** (corrected, see `HW-03-0018`)

```typescript
import { relationalStore } from '@kit.ArkData';

export class RDBStoreUtil {
  objectiveRDB?: relationalStore.RdbStore;

  async createObjectiveRDB(context: Context) {
    const STORE_CONFIG: relationalStore.StoreConfig = {
      name: 'Objective.db',
      securityLevel: relationalStore.SecurityLevel.S1
    };
    const rdbStore = await relationalStore.getRdbStore(context, STORE_CONFIG);
    this.objectiveRDB = rdbStore;
    await this.createNoteTable();
  }

  async createNoteTable() {
    await this.objectiveRDB?.execute(CommonConstants.CREATE_NOTES_TABLE_SQL);
    // FIX: the sample calls initNoteTable() unconditionally here, seeding a demo row every open
  }
}

export default new RDBStoreUtil();     // one store for the whole app
```

with the schema kept in the model rather than the utility:

```typescript
// model/DataSource.ets
static readonly CREATE_NOTES_TABLE_SQL: string =
  'CREATE TABLE IF NOT EXISTS NOTES (ID INTEGER PRIMARY KEY AUTOINCREMENT, TITLE TEXT ' +
  'NOT NULL, LOCATION TEXT NOT NULL, STARTTIME TEXT NOT NULL, ENDTIME TEXT NOT NULL, ' +
  'DATE TEXT NOT NULL, PRIORITY TEXT NOT NULL, PLANCONTENT TEXT NOT NULL, TYPE TEXT NOT NULL)';
```

**`SecurityLevel.S1` is a real decision, not boilerplate.** It classifies how
sensitive the database is and governs how the platform protects and backs it
up; workout plans are low-sensitivity, so S1 is right, but health measurements
or anything identifying would need a higher level.

`IF NOT EXISTS` makes the schema idempotent. The seed behind it is not, which
is the finding.

**Writing a plan — same file** (as shipped)

```typescript
async insertNote(title: string, location: string, startTime: string, endTime: string,
  date: string, priority: string, planContent: string, type: string) {
  const noteData: relationalStore.ValuesBucket = {
    'TITLE': title, 'LOCATION': location, 'STARTTIME': startTime, 'ENDTIME': endTime,
    'DATE': date, 'PRIORITY': priority, 'PLANCONTENT': planContent, 'TYPE': type
  };
  await this.objectiveRDB?.insert('NOTES', noteData,
    relationalStore.ConflictResolution.ON_CONFLICT_REPLACE).then((rowId: number) => {
    hilog.info(0x0000, TAG, `Insert is successful, rowId = ${rowId}`);
  });
}
```

`ON_CONFLICT_REPLACE` on a table whose only unique column autoincrements never
fires, but it is the right habit for a table that later gains a natural key.
Note the `ID` is deliberately absent from the bucket here - it is supplied by
the database, unlike the seed row.

**Adding the system event — `entry/src/main/ets/utils/AboutEvent.ets`** (as shipped)

```typescript
import { calendarManager } from '@kit.CalendarKit';

let calendar: calendarManager.Calendar | undefined = undefined;

export function addEvent(userGrant: boolean, calendarMgr: calendarManager.CalendarManager | null,
  event: calendarManager.Event, uiContext: UIContext) {
  if (!userGrant) {
    permission(uiContext);          // a toast explaining the permission is missing
    return;                         // the plan still lives in the app's own database
  }
  calendarMgr?.getCalendar().then((data: calendarManager.Calendar) => {
    calendar = data;
    calendar.addEvent(event, (err: BusinessError, data: number): void => {
      if (err) {
        hilog.error(0x0000, 'testTag', '%{public}s', `${JSON.stringify(err)}`);
      } else {
        uiContext.getPromptAction().showToast({ message: $r('app.string.success') });
      }
    });
  }).catch((err: BusinessError) => {
    hilog.info(0x0000, 'testTag', '%{public}s', `${JSON.stringify(err)}`);
  });
}
```

**Taking `userGrant` as the first parameter is the pattern worth copying.**
The function cannot be called into the calendar without the caller having
established authorisation, and refusal produces an explanation rather than
silence. `getCalendar()` with no argument returns the app's default calendar
account, which is what an app writing its own events wants.

Note the `catch` logs at `info` rather than `error`, and the module-level
`calendar` variable is shared across every call - neither is fatal, both are
worth tightening.

**Requesting the permissions — `entry/src/main/ets/utils/RequestCalendarPermission.ets`** (corrected, see `HW-03-0021`)

```typescript
export async function requestCalendarPermission(context: Context): Promise<boolean> {
  // FIX: the sample takes a userGrant parameter and returns it unchanged on refusal
  const atManager = abilityAccessCtrl.createAtManager();
  try {
    const grantStatus = await atManager.requestPermissionsFromUser(context,
      ['ohos.permission.READ_CALENDAR', 'ohos.permission.WRITE_CALENDAR']);
    return grantStatus.authResults[0] === 0 && grantStatus.authResults[1] === 0;
  } catch (err) {
    hilog.error(0x0000, 'testTag', `requestPermissionsFromUser failed: ${(err as BusinessError).code}`);
    return false;                    // FIX: the sample has no try/catch
  }
}
```

**Requiring both results to be `0` is correct** - `READ_CALENDAR` and
`WRITE_CALENDAR` are granted independently, and writing an event needs both.
This is the one sample in these industries that checks a permission pair
properly rather than overwriting its verdict in a loop.

**The custom month grid — `entry/src/main/ets/pages/MyCalendar.ets`** (as shipped)

```typescript
GridRow({ columns: 7 }) {
  ForEach(this.monthDays, (item: HmCalendarItem) => {
    GridCol() {
      CalendarItem({ /* ... */ })
    }
  }, (item: HmCalendarItem) => JSON.stringify(item))
}
```

`GridRow` with `columns: 7` gives a week per row for free - each `GridCol`
takes one column by default, so the days flow and wrap without any index
arithmetic. That is the reason to prefer it over `Grid` here: no
`columnsTemplate`, no row count to compute for a 28-to-31 day month.

**Loading the day's plans — same file** (corrected, see `HW-03-0019`)

```typescript
selectedChange = async () => {
  const value = await RDBStoreUtil.conditionalNotesQuery(this.tmpTime);
  this.selectedDate.matters = value;      // FIX: the sample loops and assigns value[index] each pass
};

async aboutToAppear() {
  await RDBStoreUtil.createObjectiveRDB(this.context);
  this.tmpTime = this.selectedDate.year.toString() + '/' + this.selectedDate.month.toString() + '/' +
    this.selectedDate.date.toString();
  await this.selectedChange();
}
```

## Permissions & config

```json5
// entry/src/main/module.json5 - correct as shipped
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

**This is the best permission block in the three industries reviewed so far**:
two permissions, each with its own purpose-specific reason string, scoped
`inuse` rather than `always`, and nothing declared that the code does not use.
Compare `SPORT-01`, which points three permissions at the ability description.

Both are `user_grant`. The database needs no permission - it lives in the
app's own sandbox.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Two stores, two failure modes.** A plan can exist in the app and not in
  the system calendar (permission refused) but not the other way round. Any
  edit or delete added later has to keep both in step - the sample stores no
  event id, so an app-side delete cannot remove the calendar entry it created.
- `SecurityLevel.S1` suits workout plans. Raise it for anything health-
  identifying.
- Dates are stored as `yyyy/M/d` strings and times as `HH:mm` strings, with
  the day query matching on the formatted date - so the schema depends on
  every writer formatting identically, and there is no timezone information
  anywhere.
- The seeded demo row is dated 2025/12/24, so a fresh install shows a plan on
  a date in the past.

## Pitfalls

- **`HW-03-0018` — opening the page seeds a fixed demo row every time.**
  `createObjectiveRDB` → `createNoteTable` → `initNoteTable` runs on every
  `aboutToAppear`, and the autoincrement key means nothing rejects the
  duplicate. The schema is guarded with `IF NOT EXISTS`; the seed is not.
- **`HW-03-0019` — the day's plans are assigned inside their own loop,** so
  only the last row survives and a day with two plans shows one.
- **`HW-03-0020` — the default end time is `getHours() + 1` with no wrap,**
  producing `24:07` after 23:00 and an invalid `Date` from it.
- **`HW-03-0021` — `requestCalendarPermission` takes the caller's flag as its
  starting value** and returns it unchanged on refusal, so passing `true`
  reports a grant that did not happen. No `try`/`catch` either.
- **`HW-03-0022` — both calendar permissions are requested when the sheet
  opens,** before the user has said they want a calendar event at all.

## References

- `documentation/harmonyos-references/06_application-services/js-apis-calendarManager.md` - `getCalendarManager`, `getCalendar`, `addEvent`, `Event`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-calendarmanager
- `documentation/harmonyos-references/02_application-framework/arkts-apis-data-relationalstore-rdbstore.md` - `RdbStore`, `insert`, `batchInsert`, `query`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-data-relationalstore-rdbstore
- `documentation/harmonyos-guides/03_application-framework/data-persistence-by-rdb-store.md` - `StoreConfig`, `SecurityLevel`, table creation
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/data-persistence-by-rdb-store
- `documentation/harmonyos-references/02_application-framework/ts-container-gridrow.md` - `GridRow` and `columns`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-gridrow
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - `bindSheet`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` - when to raise a user_grant request
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `reason` and `usedScene`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- `SPORT-01` - the app these plans belong to, and a permission block to contrast with this one
