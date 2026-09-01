---
id: SHOP-16
title: Ticket-drop countdown with a calendar reminder - setInterval for the digits, CalendarKit for the alarm the app cannot give
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/16_booking_to_grab_tickets.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/booking_to_grab_tickets-0000002304538948
sample: huawei_industry_tree/16_shopping/downloads/CountDownDemo.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CalendarKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [abilityAccessCtrl, calendarManager, common, hilog, window, setInterval, clearInterval, "UIContext.showDatePickerDialog", "calendarManager.getCalendarManager", createCalendar, getCalendar, "Calendar.setConfig", "Calendar.addEvent", EventType, CalendarType, "Intl.DateTimeFormat", "@Link", "@StorageLink", "@StorageProp", requestPermissionsFromUser]
permissions: [ohos.permission.READ_CALENDAR, ohos.permission.WRITE_CALENDAR]
min_api: 20
modules: [entry (entry)]
findings: [HW-16-0018, HW-16-0019, HW-16-0027]
status: verified-with-fixes
---

## When to use

Load this card when a screen has to **count down to a moment the user must be
present for** - a ticket drop, a flash sale, a lottery draw, an auction close -
**and then hand the reminder off to the system** so the user is alerted even
if the app is not running.

Those are two separate jobs and the sample is right to treat them that way.
The countdown is pure UI: a one-second `setInterval` and four `@State` numbers.
The reminder is a calendar event written through `CalendarKit`, which survives
the app being killed, appears in the user's own calendar, and costs the app no
background execution quota. An app-side timer cannot do that; a calendar event
is the sanctioned way to get an alarm at a wall-clock time.

The transferable lesson is the boundary between the two. **Do not treat the
in-app countdown as the source of truth for time** - `HW-16-0018` is exactly
that mistake, and it is the single most common bug in countdown widgets: the
sample subtracts the nominal interval from a stored gap instead of asking the
clock, so after any throttle or backgrounding the digits are wrong in the one
direction that matters, showing time remaining when the sale has already
started.

## Feature checklist

- A show detail page: poster, price, date, venue, and a fixed bottom bar with
  a collect icon and a 预约抢票 (book to grab tickets) button.
- Above the button, a countdown card showing days / hours / minutes / seconds,
  each digit pair in its own rounded dark chip.
- Values are zero-padded to two characters.
- The sale time defaults to 23:59:59 tomorrow, rendered above the countdown in
  a localised zh-CN long format.
- Tapping that line opens a date-and-time picker, ranged from now to one year
  ahead; accepting restarts the countdown against the new target.
- Reaching zero stops the timer and disables the booking button.
- Tapping the booking button writes a calendar event at the sale time with a
  reminder ten minutes before.
- Calendar read/write permissions are requested once, at ability start.

## Architecture

One `entry` module, one page and one component.

```
entry/src/main/ets
├── constants/CommonConstants.ets     millisecond constants + the 23:59:59 default
├── entryability/EntryAbility.ets     avoid areas, permission request, CalendarManager -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── pages
│   ├── CountDownView.ets             the countdown card, the picker, the timer (201 lines)
│   └── Index.ets                     @Entry: the show page + writeCalendar() (344 lines)
└── utils/Logger.ets
```

The documented tree matches the zip.

**The design decision worth copying** is where the `CalendarManager` comes
from. `EntryAbility` requests the two permissions, and only inside the
resolution of that request does it construct the manager and publish it:

```typescript
atManager.requestPermissionsFromUser(mContext, permissions).then((result: PermissionRequestResult) => {
  let calendarMgr = calendarManager.getCalendarManager(mContext);
  AppStorage.setOrCreate('calendarMgr', calendarMgr);
});
```

The page then picks it up with
`@StorageLink('calendarMgr') calendarMgr: calendarManager.CalendarManager | undefined = undefined`
and calls it through the optional chain `this.calendarMgr?.createCalendar(...)`.
That gives the whole app one manager instance, created after the grant,
reachable without passing a context around, and typed `undefined` until it
exists - so a booking tap before the permission resolves is a silent no-op
rather than a crash. Copy this shape for any permission-gated service object.

The second structural choice is the split of *time* between the two files.
`Index` owns `ticketDate` and `isRunning` as `@State`; `CountDownView` takes
both as `@Link`. So the picker inside the countdown card can rewrite the
target date and the parent's booking button - `enabled(this.isRunning)` and
`startTime: this.ticketDate.getTime()` - follows automatically. One date,
two-way bound, no callbacks.

What the sample gets wrong is that the *countdown itself* keeps a third,
private copy of time: `private gap`. That field is the bug (`HW-16-0018`).

## Implementation steps

1. **Request `READ_CALENDAR` and `WRITE_CALENDAR` in
   `onWindowStageCreate`,** and build the `CalendarManager` in the `then`, not
   before it; publish it through `AppStorage`.
2. **Check the grant result** before constructing the manager -
   `requestPermissionsFromUser` resolves on refusal too, with `-1` in
   `authResults`.
3. **Own the target date in the page** as `@State ticketDate: Date`, and pass
   it plus a `running` flag to the countdown as `@Link`s.
4. **Seed the target** from `CommonConstants.SAMPLE_HOUR/MINUTE/SEC` on the
   next day, and format it for display with `Intl.DateTimeFormat('zh-CN', ...)`
   rather than string concatenation.
5. **Recompute the gap from the wall clock on every tick**:
   `this.gap = this.ticketDate.getTime() - Date.now();` - never
   `gap -= DURATION` (`HW-16-0018`).
6. **Stop before you format.** `stopCountDown()` must be the last thing the
   tick does when the gap is non-positive, or `calculateTime()` will render
   negative digits over the zeros it just wrote.
7. **Clear the interval in `aboutToDisappear`** as well as at zero; the sample
   leaks the timer if the component goes away first.
8. **Open the picker through the `UIContext`**:
   `this.getUIContext().showDatePickerDialog({...})` with `showTime: true`,
   `start: now`, `end: now + one year`, and write the accepted value back to
   both `selectedDate` and the `@Link`.
9. **Reuse the calendar account.** Try `getCalendar(calendarAccount)` and only
   `createCalendar` when the lookup fails; keep the returned `eventId` so the
   reminder can later be updated or deleted (`HW-16-0019`).
10. **Give the event a non-zero duration.** The sample sets `endTime` equal to
    `startTime`.

## Verified snippets

All snippets are from `CountDownDemo.zip`. Corrected forms are marked.

**The timer — `entry/src/main/ets/pages/CountDownView.ets`** (corrected, see `HW-16-0018`)

```typescript
@Link ticketDate: Date;
@Link isRunning: boolean;
private nowDate = new Date();
private intervalID: number | null = null;
private gap = 0;
private DURATION: number = 1000;

aboutToAppear(): void {
  this.ticketDate.setDate(this.nowDate.getDate() + 1);
  this.ticketDate.setHours(CommonConstants.SAMPLE_HOUR);      // 23
  this.ticketDate.setMinutes(CommonConstants.SAMPLE_MINUTE);  // 59
  this.ticketDate.setSeconds(CommonConstants.SAMPLE_SEC);     // 59
  this.nowDate = new Date();
  this.gap = this.ticketDate.getTime() - this.nowDate.getTime();
  this.ticketDayString = this.getTicketDayString();

  this.calculateTime();
  this.startCountDown();
}

startCountDown() {
  this.isRunning = true;
  this.intervalID = setInterval(() => {
    // FIX: shipped code is `this.gap = this.gap - this.DURATION;`
    this.gap = this.ticketDate.getTime() - Date.now();
    if (this.gap <= 0) {
      this.stopCountDown();
      return;                     // FIX: shipped code falls through into calculateTime()
    }                             //      and re-renders the zeros as negative digits
    this.calculateTime();
  }, this.DURATION);
}

stopCountDown() {
  this.days = 0;
  this.hours = 0;
  this.minutes = 0;
  this.seconds = 0;
  clearInterval(this.intervalID);
  this.isRunning = false;
}

private calculateTime() {
  this.days = Math.floor(this.gap / (CommonConstants.ONE_DAY_TOTAL_MILLS));
  this.hours =
    Math.floor((this.gap % (CommonConstants.ONE_DAY_TOTAL_MILLS)) / (CommonConstants.ONE_HOUR_TOTAL_MILLS));
  this.minutes =
    Math.floor((this.gap % (CommonConstants.ONE_HOUR_TOTAL_MILLS)) / (CommonConstants.ONE_MINUTE_TOTAL_MILLS));
  this.seconds =
    Math.floor((this.gap % (CommonConstants.ONE_MINUTE_TOTAL_MILLS)) / (CommonConstants.ONE_SECOND_TOTAL_MILLS));
}
```

**The interval is a repaint trigger, not a clock.** `setInterval` guarantees
*at least* the requested delay, never exactly it, and the system throttles or
suspends timers while the app is in the background. The shipped
`gap -= DURATION` therefore accumulates every missed and every late tick as
permanent error, always in the direction of showing more time than remains -
which for a ticket drop means the user is told the sale has not started when
it has. Deriving `gap` from `Date.now()` on every tick makes the drift
structurally impossible: a late tick simply skips a number.

**`calculateTime` is a pure formatter over `gap`,** using cascading modulo -
`gap % ONE_DAY / ONE_HOUR` gives the hours *within* the day, and so on. That
is why it must never run with a negative gap: `Math.floor` on a negative
quotient rounds away from zero, so the shipped code's fall-through renders
`-1` in each chip and `formatTimeNumber` then zero-pads it to `0-1`. The
`return` after `stopCountDown()` is the whole fix.

`isRunning` being a `@Link` is what disables the booking button on the parent
the moment the timer stops - no callback, no event.

**Re-targeting from the picker — same file** (as shipped)

```typescript
Text() {
  Span(this.ticketDayString);
  Span($r('app.string.get_show_ticket_text'));    // 开抢
}
.onClick(() => {
  this.getUIContext().showDatePickerDialog({
    start: this.nowDate,
    end: new Date(this.nowDate.getTime() + CommonConstants.ONE_YEAR_TOTAL_MILLS),
    selected: this.selectedDate,
    showTime: true,
    useMilitaryTime: false,
    onDateAccept: (value: Date) => {
      // 通过Date的setFullYear方法设置按下确定按钮时的日期，这样当弹窗再次弹出时显示选中的是上一次确定的日期
      this.selectedDate = value;
      this.ticketDate = value;
      this.nowDate = new Date();
      this.ticketDayString = this.getTicketDayString();
      this.gap = this.ticketDate.getTime() - this.nowDate.getTime();
      if (this.gap <= 0) {
        this.stopCountDown();
      } else {
        this.stopCountDown();
        this.calculateTime();
        this.startCountDown();
      }
    }
  });
});
```

**`showDatePickerDialog` on the `UIContext` is the correct call form.** The
bare global `DatePickerDialog.show()` has no way to know which UI instance it
belongs to in a multi-window or multi-instance app; the UIContext method is
its sanctioned replacement, and unlike `CalendarPickerDialog` it does not need
a `runScopedTask` wrapper. `showTime: true` is what turns a date picker into
the date-and-time picker this feature needs - a ticket drop is a minute, not a
day.

`onDateAccept` writes the value to **three** places on purpose:
`selectedDate` so the dialog reopens on the last choice rather than the seed,
`ticketDate` (the `@Link`) so the parent's calendar event targets the new time,
and `ticketDayString` for the label. The `stopCountDown()` in both branches is
the guard against two intervals running at once - re-targeting while a timer
is live would otherwise leave the old one ticking forever.

**The calendar event — `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-16-0019`)

```typescript
@StorageLink('calendarMgr') calendarMgr: calendarManager.CalendarManager | undefined = undefined;
calendar: calendarManager.Calendar | undefined = undefined;

private async writeCalendar() {
  const calendarAccount: calendarManager.CalendarAccount = {
    name: 'MyCalendar',
    type: calendarManager.CalendarType.LOCAL,
    // 日历账户显示名称，该字段如果不填，创建的日历账户在界面显示为空字符串。
    displayName: 'MyCalendar'
  };
  const config: calendarManager.CalendarConfig = {
    enableReminder: true,          // 打开日程提醒
    color: '#aabbcc'
  };

  // FIX: shipped code calls createCalendar() unconditionally on every tap
  try {
    this.calendar = await this.calendarMgr?.getCalendar(calendarAccount);
  } catch (err) {
    logger.info(`No existing calendar, creating one. Code: ${(err as BusinessError).code}`);
  }
  if (!this.calendar) {
    this.calendar = await this.calendarMgr?.createCalendar(calendarAccount);
  }
  if (!this.calendar) {
    return;
  }

  await this.calendar.setConfig(config);
  const event: calendarManager.Event = {
    title: this.calendarTitle,
    // 不推荐三方开发者使用 EventType.IMPORTANT，重要日程不支持一键服务跳转且无法自定义提醒时间
    type: calendarManager.EventType.NORMAL,
    startTime: this.ticketDate.getTime(),
    endTime: this.ticketDate.getTime(),
    reminderTime: [10]             // 距开始时间提前10分钟提醒
  };
  const eventId: number = await this.calendar.addEvent(event);
  logger.info(`Succeeded in adding event, id -> ${eventId}`);
}
```

**Three fields decide whether the reminder actually fires.**
`CalendarConfig.enableReminder` must be `true` or the account is silent no
matter what the event says. `EventType.NORMAL` rather than `IMPORTANT` is what
the sample's own comment insists on: important events cannot carry a custom
reminder time and do not support the one-tap service jump, so `reminderTime`
would be ignored. And `reminderTime: [10]` is an array of **minutes before
`startTime`** - it is a list because an event may carry several reminders.

**The account must be looked up, not created.** `createCalendar` is a
create-only call; firing it on every booking tap either fails because
`MyCalendar` already exists, so no event is added at all, or accumulates
duplicate accounts. The corrected form tries `getCalendar` first, which is
also the only way to reach an account the user has already got from a previous
run of the app. The shipped code additionally assigns `addEvent`'s result to a
local `eventId` and throws it away; keep it, because updating or deleting the
reminder later needs it - and a user who re-books after moving the sale time
should get one updated reminder, not two.

`endTime` equal to `startTime` gives a zero-length event. The calendar accepts
it, but it renders as a point in the day grid; a fifteen-minute window reads
better and is what the drop actually occupies.

## Permissions & config

Declared in `entry/src/main/module.json5`:

```json5
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
  }
]
```

- Both are `user_grant`, so `reason` and `usedScene` are mandatory and the
  `reason` string resource must exist.
- `when: "inuse"` is right: the event is written while the user is looking at
  the page; the reminder afterwards is the system's job, not the app's.
- `WRITE_CALENDAR` alone is not enough - `getCalendar` and the account lookup
  need `READ_CALENDAR`, which is a further reason the corrected flow declares
  both.
- The sample requests the pair unconditionally at `onWindowStageCreate`, so
  the dialog appears on first launch before the user has expressed any
  interest in a reminder. Requesting on the first booking tap would be the
  better manners; requesting here is the simpler demo.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `EntryAbility` never inspects `PermissionRequestResult.authResults`. The
  promise resolves on refusal too, so the manager is built either way and the
  first booking tap simply fails inside `CalendarKit`.
- The interval is cleared only when the countdown reaches zero or the picker
  restarts it. There is no `aboutToDisappear`, so navigating away with a live
  countdown leaks the timer.
- `clearInterval(this.intervalID)` is called with a `number | null`; it is
  only safe because `stopCountDown` is never reached before `startCountDown`.
- The countdown displays days without a cap, so a target a year out renders a
  three-digit day chip in a fixed-width container.
- `EntryAbility` writes the avoid-area heights to `AppStorage` **already
  converted to vp**, which is why `Index.ets` applies `this.topRectHeight`
  directly with no `px2vp`. Do not add one.
- Nothing persists the target time: relaunching returns to 23:59:59 tomorrow,
  while the calendar event the user booked stays at the old time.

## Pitfalls

- **`HW-16-0018`** (B/medium, probable): `startCountDown` reduces the stored
  `gap` by the nominal `DURATION` on each tick and never re-derives it from
  the clock, so the display drifts with every late or throttled tick and lags
  badly after the app has been in the background - the user comes back and
  sees time remaining although the drop has begun. Fix: set
  `this.gap = this.ticketDate.getTime() - Date.now();` at the top of the
  interval callback.
- **`HW-16-0019`** (B/low, probable): `writeCalendar` calls
  `calendarMgr.createCalendar({ name: 'MyCalendar', ... })` unconditionally on
  every booking tap, with no `getCalendar` lookup and no cached account, and
  discards the `eventId` `addEvent` returns. Repeat bookings either fail
  outright or pile up duplicate accounts, and the reminder can never be
  updated or removed. Fix: look the account up first and fall back to
  `createCalendar`; keep the `eventId`.
- **The tick renders negative digits at zero.** The interval calls
  `stopCountDown()` and then `calculateTime()` unconditionally, so the zeros
  that `stopCountDown` just wrote are immediately overwritten from a negative
  `gap`; `formatTimeNumber` zero-pads the result to strings like `0-1`. Return
  after stopping.
- **No `aboutToDisappear`.** The `setInterval` outlives the component unless
  the countdown happens to reach zero first.
- **The grant is never checked.** `requestPermissionsFromUser` resolving is
  not the same as being granted; read `authResults` before building the
  `CalendarManager`.
- **`endTime === startTime`** produces a zero-length calendar entry.

## References

- `documentation/harmonyos-references/05_common-capabilities/js-apis-timer.md` - `setInterval`/`clearInterval` and their timing guarantees
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-timer
- `documentation/harmonyos-references/06_application-services/js-apis-calendarmanager.md` - `getCalendarManager`, `getCalendar`, `createCalendar`, `CalendarConfig.enableReminder`, `Event.reminderTime`, `EventType`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-calendarmanager
- `documentation/harmonyos-references/02_application-framework/ts-methods-datepicker-dialog.md` - `showDatePickerDialog`, `showTime`, `onDateAccept`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-methods-datepicker-dialog
- `documentation/harmonyos-guides/03_application-framework/arkts-global-interface.md` - why the picker is opened through the `UIContext`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-global-interface
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `READ_CALENDAR` / `WRITE_CALENDAR`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` - the `user_grant` request flow and `authResults`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `huawei_industry_tree/16_shopping/docs/16_booking_to_grab_tickets.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/booking_to_grab_tickets-0000002304538948
