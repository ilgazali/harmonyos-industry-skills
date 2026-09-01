---
id: TOUR-04
title: Date range picker - hotel check-in and check-out calendar on Grid and Stack
industry: 09_tourism
doc: huawei_industry_tree/09_tourism/docs/04_data_selection.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_selection-0000002240198078
sample: huawei_industry_tree/09_tourism/downloads/DateSelection.zip
kits: ["@kit.ArkTS", "@kit.PerformanceAnalysisKit", "@kit.ArkUI"]
apis: [Grid, GridItem, Stack, List, ListItemGroup, LazyForEach, IDataSource, Navigation, NavPathStack, NavDestination, pushPathByName, pop, PopInfo, "util.format", "@Provide", "@Consume", "@StorageProp", safeAreaPadding, columnsTemplate, rowsTemplate]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-09-0022, HW-09-0023, HW-09-0024, HW-09-0025, HW-09-0026, HW-09-0027, HW-09-0080]
status: verified-with-fixes
---

## When to use

Load this card for a **two-endpoint date range picked from a scrolling
calendar** - hotel check-in and check-out, a car rental period, a multi-day
ticket, a leave request. The distinguishing requirement is that the range is
selected by two taps on a month grid and the days between the endpoints must
be visibly filled, which is what makes it different from an ordinary
`DatePicker` or `CalendarPicker`.

It is also a compact, readable example of three things worth reading on their
own: laying a month out on a `Grid` with leading and trailing blanks,
overlaying selection state with `Stack` instead of restyling the day cell, and
returning a result from a `NavDestination` through `NavPathStack.pop`.

Do not reach for this when a single date is enough - the system
`CalendarPicker` is less code. Reach for it when the range itself is the
product.

## Feature checklist

- A room detail page showing check-in and check-out slots, empty at first.
- Tapping the slots opens the calendar as a `NavDestination`.
- The calendar shows the current month and the next, scrolling, with a sticky
  year-month header per month.
- A fixed weekday header row, Sunday first.
- First tap sets the start; second tap sets the end - unless it falls before
  the start, in which case it re-anchors the start.
- A third tap clears the range and starts over.
- The two endpoints are badged 入住 (check in) and 离店 (check out); the days
  between them are tinted.
- Leading and trailing cells belonging to the neighbouring months are laid out
  but invisible.
- The footer shows both dates with their weekday and a confirm button
  labelled with the number of nights.
- Confirming pops back and writes both dates into the room page.

## Architecture

One `entry` module. Two pages, a lazy data source and a date utility - no
network, no storage.

```
entry/src/main/ets
├── entryability/EntryAbility.ets
├── model
│   ├── DataType.ets           TabBarDataType
│   ├── GetDate.ets            getMonthDate(month, year) -> number[] with 0 padding
│   ├── MonthDataSource.ets    IDataSource over Month[]
│   └── TabBarData.ets         static bottom bar entries
├── pages
│   ├── MainPage.ets           @Entry, Navigation host, owns the two result strings
│   └── CalendarPage.ets       the NavDestination, owns the whole selection state
└── util/DateUtils.ets         isDayLaterThan, isDayBetween, calculateDays, formatMonth
```

The documented tree matches the zip.

**The state model is the thing to copy.** `CalendarPage` holds the range as
six independent `@State` numbers - `startYear` / `startMonth` / `startDay` and
the three `end*` counterparts - with `-1` as the "not chosen" sentinel on the
month and day. There is no `Date` object in the state and no selected-day set;
every cell decides its own appearance by comparing itself against those six
numbers during layout. That is what keeps the tap handler to a three-branch
`if` and makes the whole feature reviewable in one screen.

The trade-off is that the comparison has to be written out at each render
site, and the sample gets it inconsistent: the endpoints compare month and
day, the middle days compare year, month and day (`HW-09-0023`).

**The result travels by `pop`,** not by shared storage: `MainPage` pushes with
an `onPop` callback and `CalendarPage` pops with a `string[]`. Note from the
`Navigation` reference that `onPop` "is triggered only when the **result**
parameter is set in pop" - a system back gesture pops without a result and
never calls the callback, so `MainPage` does not need a guard for it.

## Implementation steps

1. **Build the month array** with `getMonthDate(month, year)`: leading zeros
   for the weekday offset of the 1st, then 1..totalDays, then trailing zeros
   up to Saturday. `new Date(year, month, 0).getDate()` gives the month length
   with a 1-based month.
2. **Wrap the months in an `IDataSource`** so the calendar can grow. Notify
   one add per appended element (`HW-09-0024`).
3. **One `ListItemGroup` per month** inside a `LazyForEach`, with the
   year-month text as its `header` - that gives sticky headers for free.
4. **One `Grid` per month**, `columnsTemplate` of seven `1fr`, and a
   `rowsTemplate` chosen from the array length: six rows when the padded array
   exceeds 35 cells, five otherwise.
5. **Hide the padding cells with `opacity`**, not by omitting them - the grid
   positions depend on them being there.
6. **Stack the selection over the day**, do not restyle the day: `Text(day)`
   with `zIndex(999)` on top, and the badge or tint `Text` beneath it in the
   same `Stack`.
7. **Three branches in the tap handler**: no start yet -> set start; start but
   no end -> if the tap is earlier than the start, move the start, else set
   the end; both set -> clear and set a new start.
8. **Compare with the year included** at every site (`HW-09-0023`).
9. **Compute the nights from full dates**, years included (`HW-09-0022`).
10. **Pop with the result** and read it in the pusher's `onPop`.

## Verified snippets

All snippets are from `DateSelection.zip`. Corrected forms are marked.

**Laying a month on a seven-column grid — `entry/src/main/ets/model/GetDate.ets`** (as shipped)

```typescript
const SATURDAY = 6; // index of Saturday when the week starts on Sunday (0..6)

export function getMonthDate(specifiedMonth: number, specifiedYear: number) {
  let currentAllDay: number[] = [];
  // day 0 of the next month is the last day of this one
  let totalDays = new Date(specifiedYear, specifiedMonth, 0).getDate();
  let currentFirstWeekDay = new Date(specifiedYear, specifiedMonth - 1, 1).getDay();
  let currentLastWeekDay = new Date(specifiedYear, specifiedMonth - 1, totalDays).getDay();

  for (let item = 0; item < currentFirstWeekDay; item++) {
    currentAllDay[item] = 0;              // blanks before the 1st
  }
  for (let item = 1; item <= totalDays; item++) {
    currentAllDay.push(item);
  }
  for (let item = 0; item < SATURDAY - currentLastWeekDay; item++) {
    currentAllDay.push(0);                // blanks after the last day
  }
  return currentAllDay;
}
```

**`new Date(year, month, 0).getDate()` is the idiom worth memorising** - with
a 1-based month it yields that month's length, leap years included, without a
table. The padded array is then a flat list the `Grid` can consume in order,
and `0` marks a cell that exists for layout but shows nothing.

**The month grid — `entry/src/main/ets/pages/CalendarPage.ets`** (as shipped)

```typescript
const ROW_NUMBER = 35;

List() {
  LazyForEach(this.contentData, (monthItem: Month) => {
    ListItemGroup({ header: this.itemHead(monthItem.yearWithMonth) }) {
      ListItem() {
        Stack() {
          Grid() {
            ForEach(monthItem.days, (day: number) => {
              GridItem() {
                Stack() { /* the day cell, see below */ }
              }
              .opacity(day === 0 ? 0 : 1)     // padding cells occupy space, show nothing
            })
          }
          .columnsTemplate('1fr 1fr 1fr 1fr 1fr 1fr 1fr')
          // 6 rows only when the padded month needs them
          .rowsTemplate(monthItem.days.length > ROW_NUMBER ? '1fr 1fr 1fr 1fr 1fr 1fr' : '1fr 1fr 1fr 1fr 1fr')
          .height(monthItem.days.length > ROW_NUMBER ? 360 : 300)
        }
      }
    }
  })
}
.edgeEffect(EdgeEffect.None)
.scrollBar(BarState.Off)
```

`ListItemGroup` with a `header` is what makes the year-month label stick while
its month scrolls - a plain `List` of months would need that built by hand.
Switching `rowsTemplate` **and** `height` together off the same 35-cell test
keeps every day cell the same size whether the month spans five weeks or six.

**Selection overlaid with Stack — same file** (corrected, see `HW-09-0023`)

```typescript
Stack() {
  Text(day.toString())
    .zIndex(999)                       // the number stays above whatever is painted under it
    .fontColor(
      day === this.startDay && monthItem.month === this.startMonth && monthItem.year === this.startYear ||
      day === this.endDay && monthItem.month === this.endMonth && monthItem.year === this.endYear
        ? Color.White : $r('app.color.date_font_color'))
    .width('100%').height('100%').textAlign(TextAlign.Center).borderRadius(10)
    .onClick(() => { /* the three branches, below */ })

  // endpoint: a solid block carrying the 入住 / 离店 label
  if (day === this.startDay && monthItem.month === this.startMonth && monthItem.year === this.startYear) {
    Column() {
      Text()
      Text($r('app.string.check_in')).fontSize(12).fontColor(Color.White).margin({ top: 40 })
    }
    .borderRadius(5)
    .backgroundColor($r('app.color.selected_background_color'))
  }
  // between the endpoints: a flat tint
  else if (this.endMonth !== -1 &&
    DateUtils.isDayBetween(monthItem.year, monthItem.month, day,
      this.startYear, this.startMonth, this.startDay,
      this.endYear, this.endMonth, this.endDay)) {
    Text()
      .backgroundColor($r('app.color.middle_background_color'))
      .width('100%').height('95%')
  }
}
```

The `year` terms in the endpoint conditions are the fix; the sample compares
month and day only. **The overlay approach is the point of the card**: the day
number is one component with a fixed style and the selection is a sibling
painted underneath it, so adding a state - a blocked date, a price tag, a
"today" ring - means adding a branch to the `Stack`, not another ternary
inside `Text`.

**The three-branch tap handler — same file** (corrected, see `HW-09-0025`)

```typescript
.onClick(() => {
  if (this.startMonth === -1) {                    // 1. nothing chosen yet
    this.startYear = monthItem.year;
    this.startMonth = monthItem.month;
    this.startDay = day;
  } else if (this.endMonth === -1) {               // 2. start chosen, end open
    if (DateUtils.isDayLaterThan(this.startYear, this.startMonth, this.startDay,
      monthItem.year, monthItem.month, day)) {
      this.startYear = monthItem.year;             //    tapped before the start: re-anchor
      this.startMonth = monthItem.month;
      this.startDay = day;
    } else {
      this.endYear = monthItem.year;               //    tapped after: close the range
      this.endMonth = monthItem.month;
      this.endDay = day;
    }
  } else {                                         // 3. range complete: start over
    this.startYear = monthItem.year;
    this.startMonth = monthItem.month;
    this.startDay = day;
    this.endYear = 0;
    this.endMonth = -1;
    this.endDay = -1;
  }

  this.startWeekDay = new Date(this.startYear, this.startMonth - 1, this.startDay).getDay();
  if (this.endMonth !== -1 && this.endDay !== -1) {          // FIX: shipped code computes both
    this.endWeekDay = new Date(this.endYear, this.endMonth - 1, this.endDay).getDay();
    this.stayNights = DateUtils.calculateDays(
      this.startYear, this.startMonth, this.startDay,        // FIX: shipped call drops both years
      this.endYear, this.endMonth, this.endDay);
  } else {
    this.endWeekDay = 0;
    this.stayNights = 0;
  }
})
```

**Branch 2 is the detail that makes the picker feel right.** Without it, a
user who taps a date earlier than their start gets an inverted range or a
rejected tap; with it, the earlier tap simply becomes the new start, which is
what every booking calendar does.

**Date comparison — `entry/src/main/ets/util/DateUtils.ets`** (corrected, see `HW-09-0022`)

```typescript
export class DateUtils {
  static isDayBetween(currentYear: number, currentMonth: number, currentDay: number,
    startYear: number, startMonth: number, startDay: number,
    endYear: number, endMonth: number, endDay: number): boolean {
    return DateUtils.isDayLaterThan(currentYear, currentMonth, currentDay, startYear, startMonth, startDay) &&
      DateUtils.isDayLaterThan(endYear, endMonth, endDay, currentYear, currentMonth, currentDay);
  }

  static isDayLaterThan(currentYear: number, currentMonth: number, currentDay: number,
    targetYear: number, targetMonth: number, targetDay: number): boolean {
    return new Date(currentYear, currentMonth - 1, currentDay) >
           new Date(targetYear, targetMonth - 1, targetDay);
  }

  // FIX: the shipped signature is (startMonth, startDay, endMonth, endDay) and
  // anchors both ends to new Date().getFullYear(), which miscounts across a leap day
  static calculateDays(startYear: number, startMonth: number, startDay: number,
    endYear: number, endMonth: number, endDay: number): number {
    const startDate = new Date(startYear, startMonth - 1, startDay);
    const endDate = new Date(endYear, endMonth - 1, endDay);
    return Math.ceil((endDate.getTime() - startDate.getTime()) / (1000 * 60 * 60 * 24));
  }
}
```

`isDayBetween` composed from two strict `isDayLaterThan` calls gives an
exclusive range, which is exactly right here: the endpoints get their own
badge, so the tint must cover only the days strictly between them. With real
years the night count needs no cross-year special case at all - the shipped
version has one only because it threw the years away first.

**Returning the range — `entry/src/main/ets/pages/MainPage.ets` and `CalendarPage.ets`** (as shipped)

```typescript
// MainPage: push with a callback
.onClick(() => {
  this.pathStack.pushPathByName('calendarPage', null, (info: PopInfo) => {
    let date = info.result as string[];
    this.startDate = date[0];
    this.endDate = date[1];
  });
})

// CalendarPage: pop with the result
Button(`确认${this.stayNights}晚`)
  .onClick(() => {
    this.pathStack.pop([
      this.startMonth !== -1 ? util.format('%s月%s日(周%s)',
        this.startMonth.toString(), this.startDay.toString(), this.week[this.startWeekDay]) : '',
      this.endMonth !== -1 ? util.format('%s月%s日(周%s)',
        this.endMonth.toString(), this.endDay.toString(), this.week[this.endWeekDay]) : ''
    ]);
  })
```

`onPop` fires **only** when `pop` is given a result, so a back gesture leaves
`MainPage` untouched with no guard needed. The literal `确认N晚` and the
`%s月%s日` format are the strings that should have been resources
(`HW-09-0026`).

## Permissions & config

**None.** The sample declares no `requestPermissions` and its
`extensionAbilities` array is empty.

Routing is by `routerMap`, entry module:

```json5
// entry/src/main/module.json5
"routerMap": "$profile:route_map"
```

```json
// entry/src/main/resources/base/profile/route_map.json
{
  "routerMap": [
    {
      "name": "calendarPage",
      "pageSourceFile": "src/main/ets/pages/CalendarPage.ets",
      "buildFunction": "calendarPageBuilder"
    }
  ]
}
```

Resource directories: `base` and `dark` only - no locale set (`HW-09-0026`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` and
  `targetSdkVersion` are both `6.0.0(20)`.
- **Exactly two months are loaded**, the current one and the next, in
  `aboutToAppear`. There is no infinite scroll and no way to reach a date
  further out; extending the range is the first change most products need, and
  it is the change that exposes `HW-09-0023` and `HW-09-0024`.
- The week starts on **Sunday**, hardcoded in both the header array and
  `getMonthDate`'s padding maths. A Monday-first locale needs both changed.
- No past-date guard: yesterday is selectable.
- The room, price and images on `MainPage` are static resources; the picker
  returns formatted display strings, not dates, so a real booking flow needs
  the six numbers rather than the two strings.

## Pitfalls

- **`HW-09-0022` — `calculateDays` takes no years** and anchors both ends to
  the current year, so a range crossing 29 February is counted one night
  short. Both years are already in state; pass them.
- **`HW-09-0023` — endpoint highlighting compares month and day, the middle
  tint compares year, month and day.** Harmless while only two consecutive
  months render, wrong the moment the calendar spans more than a year.
- **`HW-09-0024` — `MonthDataSource.pushData` notifies a single add for an
  N-element array,** which is invisible at init only because no listener has
  registered yet. It breaks on the first append after render.
- **`HW-09-0025` — `endWeekDay` is computed from the `-1` sentinels** on every
  tap, producing `new Date(0, -2, -1)`; masked only by the render-site guards.
- **`HW-09-0026` — the weekday names, the month header, the date format and
  the confirm button are hardcoded Chinese,** and there is no locale
  directory, although everything else on the page uses `$r`.
- **`HW-09-0027` — the document's 实现思路 snippet does not compile**
  (`backgroundColor()` with no argument) and discards the `isDayLaterThan`
  return value that the real handler branches on.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` - `Grid`, `columnsTemplate`, `rowsTemplate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `documentation/harmonyos-references/02_application-framework/ts-container-griditem.md` - `GridItem`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-griditem
- `documentation/harmonyos-references/02_application-framework/ts-container-stack.md` - `Stack` and stacking order
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-stack
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `pushPathByName`, `pop`, `PopInfo`, and when `onPop` fires
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-lazyforeach.md` - `IDataSource` and the notification contract
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-lazyforeach
- `TOUR-11` - the next step in the booking funnel, confirming the traveller details
- `TOUR-05` - the other half of the search form, swapping origin and destination
