---
id: LIFE-13
title: Three-day calendar - two synchronised horizontal Lists over a Stacked timeline, with column snap-back and a bindSheet editor
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/13_3-day_view_calendar.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/3-day_view_calendar-0000002330222589
sample: huawei_industry_tree/02_convenient_life/downloads/ThreeDayViewCalendar.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: [List, ListItem, listDirection, Scroller, "scroller.currentOffset", "scroller.scrollTo", onScrollStop, edgeEffect, Scroll, ScrollDirection, Stack, zIndex, position, LazyForEach, IDataSource, DataChangeListener, bindSheet, "$$", "@Provide", "@Watch", "@State", "@Prop", "@Link", "@LocalStorageLink", "@StorageProp", "@Observed", safeAreaPadding, Blank, Divider, "window.getWindowAvoidArea", "window.on('avoidAreaChange')"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-02-0089, HW-02-0090, HW-02-0091, HW-02-0092, HW-02-0093, HW-02-0094, HW-02-0095, HW-02-0096, HW-02-0097, HW-02-0098, HW-02-0099, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card when you need a **day-view calendar**: a vertical hour ruler with
event blocks positioned by time, and more than one day visible side by side.

The construction has three parts worth knowing:

1. **A `Stack` puts the hour rules behind the day columns.** The timeline is one
   child, the horizontally scrolling list of days is the other, and `zIndex`
   orders them. The rules therefore span the full width and do not scroll
   horizontally with the days.
2. **Two `List`s share one data source and one snap offset** - a short header
   list of dates and a tall body list of columns - kept in step by scrolling one
   whenever the other stops.
3. **Event blocks are absolutely positioned** by converting `HH:mm` into a
   vertical offset at a fixed units-per-hour scale.

That third part is the reusable idea and also where the sample is weakest: the
scale used by the cards and the scale produced by the timeline are independent
numbers (`HW-02-0091`).

Take this for day and week calendars, booking grids, shift rosters, timetables -
anything laid out as time down, resource across. If the days do not need to
scroll independently of the ruler, a single `Grid` is far less machinery.

## Feature checklist

- A horizontally scrolling strip of date headers over a horizontally scrolling
  strip of day columns, moving together.
- Releasing a swipe snaps both strips to a column boundary.
- An hour ruler behind the columns, drawn once and shared by every day.
- Event blocks positioned and sized from their start and end times.
- A multi-day event draws on its first day and on each day it spans.
- A + button opens a half-modal sheet to add a schedule; saving appends it and
  every day column re-renders.
- The page draws under the status bar, with the inset applied as safe-area
  padding.

## Architecture

One `entry` module. The date strip is a `LazyForEach` over an `IDataSource`; the
schedules are a plain array.

```
entry/src/main/ets
├── constants/CalendarConstants.ets   DEFAULT_FIRST_HOUR 8, DEFAULT_LAST_HOUR 25,
│                                     DAY_COLUMN_SPACE 0, DAY_COLUMN_LENGTH 110
├── entryability/EntryAbility.ets     full screen, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets     (doc says entrybackupability.ets - HW-02-0098)
├── model
│   ├── DayInfo.ets                   DayInfo + DayDataSource (IDataSource)
│   └── ScheduleItem.ets              @Observed - title, start/end Date, start/end "HH:mm", colour
├── pages
│   ├── CalendarView.ets              @Entry - THE CARD: the Stack, both Lists, the snap
│   ├── CalendarHeaderItem.ets        one date header
│   ├── DayColumn.ets                 one day's schedules
│   ├── ScheduleCard.ets              one positioned event block
│   ├── TimelineBar.ets               the hour ruler
│   └── SheetBuilder.ets              the add-schedule half-modal
└── utils/LayoutCalculator.ets        HH:mm -> top offset and height
```

The documented tree matches the zip except for the backup-ability filename.

**The layering of the Stack is the part to copy.**

```
Scroll (vertical)
└── Column (height '120%')                      <- HW-02-0095: unrelated to the hour count
    └── Stack
        ├── TimelineBar        .zIndex(1)       hour rules, full width, never scrolls sideways
        └── Row                .zIndex(2)       the day columns, on top
            ├── Blank('150px')                  gutter matching the ruler's label column
            └── List (horizontal, scrollController)
                └── LazyForEach(dayDataSource) -> ListItem -> DayColumn
```

The header strip above repeats the same `Blank` + horizontal `List` shape with a
second `Scroller`, so the two lists have identical geometry and can be driven by
one offset.

**How the two strips stay together.** Each list's `onScrollStop` computes the
nearest column boundary, writes it into `@State @Watch('onCountUpdated')
xOffset`, and scrolls itself; the watcher then scrolls *both*. That double path
is `HW-02-0092`, and the arithmetic that picks the boundary carries a precedence
bug (`HW-02-0089`) and a unit mismatch (`HW-02-0090`).

**How an event finds its place.** `LayoutCalculator` parses `"13:00"` into hours
plus minutes/60, subtracts the first hour, and multiplies by a fixed 52. Height
is the same conversion applied to the duration, floored at half an hour so a
short event stays tappable.

## Implementation steps

1. **Declare one hour-row constant and use it everywhere** - the timeline row
   height, the card height scale and the card top offset must all be the same
   number (`HW-02-0091`).
2. **Build the day source as an `IDataSource`** so the strip can page. Notify
   per inserted index, not once for the batch (`HW-02-0094`).
3. **Draw the ruler once, behind everything,** as a `Column` of one row per hour
   with an explicit height. Give it `zIndex(1)` and the day list `zIndex(2)`.
4. **Reserve the label gutter in both strips** with the same fixed-width
   `Blank`, so the header dates line up with the columns beneath them.
5. **Size the day column from the same constant the snap uses** - not `'30%'`
   against a snap pitch of 110 (`HW-02-0090`).
6. **Write the snap once**, as a method taking the `Scroller`, and parenthesise
   the boundary comparison (`HW-02-0089`, `HW-02-0093`).
7. **Pick one synchronisation mechanism.** Either the handler scrolls both
   controllers, or it only sets state and a `@Watch` does the scrolling - not
   both (`HW-02-0092`).
8. **Position event blocks with `.position({ x, y })` inside the day column**,
   converting `HH:mm` in one place.
9. **Render one card branch,** guarded by "is the first day or is contained by
   the range" (`HW-02-0093`).
10. **Size the vertical scroll content from the hour range,** not as a
    percentage of the viewport (`HW-02-0095`).
11. **Open the editor with `bindSheet($$this.isShow, this.SheetBuilder(), ...)`**
    where `SheetBuilder()` is a local `@Builder` that forwards the two `@Link`
    arguments.

## Verified snippets

All snippets are from `ThreeDayViewCalendar.zip`. Corrected forms are marked.

**The stacked layout - `ThreeDayViewCalendar.zip#entry/src/main/ets/pages/CalendarView.ets:179`** (as shipped)

```typescript
Scroll(this.scrollerForScroll) {
  Column() {
    Stack() {
      TimelineBar({
        firstHour: CalendarConstants.DEFAULT_FIRST_HOUR,
        lastHour: CalendarConstants.DEFAULT_LAST_HOUR
      })
        .zIndex(1);
      Row() {
        Blank().width('150px').color('rgba(255, 255, 255, 0)').zIndex(0);
        List({ space: CalendarConstants.DAY_COLUMN_SPACE, scroller: this.scrollController }) {
          LazyForEach(this.dayDataSource, (day: DayInfo) => {
            ListItem() {
              Column() {
                DayColumn({ day, schedules: this.schedules }).layoutWeight(1);
              };
            }.width('30%');            // FIX: use CalendarConstants.DAY_COLUMN_LENGTH - HW-02-0090
          });
        }
        .listDirection(Axis.Horizontal)
        .edgeEffect(EdgeEffect.None)
        .scrollBar(BarState.Off)
        .onScrollStop(() => { /* snap - see below */ });
      }
      .zIndex(2)
      .width('100%')
      .height('100%');
    };
  }.height('120%');                    // FIX: derive from the hour count - HW-02-0095
}
.scrollable(ScrollDirection.Vertical)
.edgeEffect(EdgeEffect.None)
```

**One ruler behind N columns is what the `Stack` buys.** The hour lines are drawn
once at full width and stay put while the day list scrolls horizontally over
them - so adding a fourth day costs one more column, not one more ruler, and the
lines can never disagree between columns.

**The transparent `Blank().width('150px')`** in the day layer mirrors the
timeline's label column, so column 0 starts where the rules start rather than
under the hour labels. It appears in both strips with the same width, which is
what keeps the header dates over their columns.

**`edgeEffect(EdgeEffect.None)` on the horizontal lists is deliberate:** a spring
would fight the snap-back, leaving the list settling somewhere the snap has
already computed an offset for.

**The hour ruler - `ThreeDayViewCalendar.zip#entry/src/main/ets/pages/TimelineBar.ets:22`** (corrected, see `HW-02-0091` and `HW-02-0096`)

```typescript
build() {
  Column() {
    ForEach(this.generateHours(this.firstHour, this.lastHour), (hour: number) => {
      Row() {
        Text(hour.toString() + ScheduleItemConstants.SCHEDULE_TIME)   // "8" + ":00"
          .fontSize(12)
          .fontColor('#999')
          .width('11%')
          .margin({ left: -10 });
        Divider().height(1).width('100%').backgroundColor('#ccc');
      }
      .height(CalendarConstants.HOUR_ROW_HEIGHT);   // FIX: the sample writes .padding(19)
    }, (hour: number) => hour.toString());          // FIX: the sample supplies no key
  };
}

private generateHours(start: number, end: number): number[] {
  let hours: number[] = [];
  for (let i = start; i <= end; i++) {              // inclusive at both ends
    hours.push(i);
  }
  return hours;
}
```

**The row pitch has to be the same number the cards use.** As shipped the row
height is whatever `.padding(19)` plus a 12 fp `Text` comes to, while
`LayoutCalculator` positions every card at 52 units per hour - two independent
numbers that only a hand-tuned coincidence keeps close. Setting the height
explicitly from a shared constant is the whole fix.

`generateHours` is inclusive at both ends, so `DEFAULT_LAST_HOUR = 25` produces a
`25:00` row (`HW-02-0096`).

**Time to geometry - `ThreeDayViewCalendar.zip#entry/src/main/ets/utils/LayoutCalculator.ets:28`** (as shipped)

```typescript
static getScheduleHeight(schedule: ScheduleItem): number {
  let startTimeParts = schedule.startTime.split(':');
  let startHours = Number(startTimeParts[0]);
  let startMinutes = Number(startTimeParts[1]);
  let endTimeParts = schedule.endTime.split(':');
  let endHours = Number(endTimeParts[0]);
  let endMinutes = Number(endTimeParts[1]);
  let durationHours = (endHours - startHours);
  let durationMinutes = (endMinutes - startMinutes) / 60;
  let durationTime = durationHours + durationMinutes;
  let offset = 52;                          // 每小时 52px - but consumed as vp (HW-02-0091)
  if (durationTime < 0.5) {
    return 0.5 * offset;                    // a half-hour floor keeps short events tappable
  } else {
    return durationTime * offset;
  }
}

static getScheduleTopPosition(schedule: ScheduleItem, firstHour: number): number {
  let startTimeParts = schedule.startTime.split(':');
  let hours = Number(startTimeParts[0]);
  let minutes = Number(startTimeParts[1]);
  let totalHours = hours + minutes / 60;
  return (totalHours - firstHour) * 52 + 28;    // 28 = half a row, centring on the hour line
}
```

**`hours + minutes / 60` is the whole conversion** - decimal hours make both the
offset and the height one multiplication. Subtracting `firstHour` is what makes
the ruler's first row the origin, which is why `firstHour` must be the same
value everywhere (`HW-02-0096`).

**The half-hour floor is a real UX rule,** not defensive padding: a 5-minute
event would otherwise be four units tall and impossible to hit.

The `+ 28` offsets cards to sit below their hour line rather than on it. It is
half of 52 rounded, and like the 52s it is a literal repeated rather than a
named constant.

**The snap - same file, line 153 and line 202** (corrected, see `HW-02-0089`, `HW-02-0092`, `HW-02-0093`)

```typescript
// FIX: the sample repeats this block verbatim in both onScrollStop handlers.
private snapToColumn(scroller: Scroller): void {
  const xOffset = scroller.currentOffset().xOffset;
  const yOffset = scroller.currentOffset().yOffset;
  const pitch = CalendarConstants.DAY_COLUMN_LENGTH;

  for (let i = 0; i < this.dayDataSource.totalCount(); i++) {
    if (xOffset <= i * pitch && (i * pitch - pitch / 2) <= xOffset) {
      this.xOffset = pitch * i;                  // past the midpoint: snap forward
      break;
    } else if (xOffset >= (i - 1) * pitch &&     // FIX: sample has i - 1 * pitch
               (i * pitch - pitch / 2) >= xOffset) {
      this.xOffset = pitch * (i - 1);            // before the midpoint: snap back
      break;
    } else if (xOffset === 0) {
      this.xOffset = 0;
    }
  }
  // FIX: the sample also scrolls here, and again from @Watch('onCountUpdated').
  //      Pick one. Leaving it to the watcher keeps both lists in step.
  scroller.scrollTo({ xOffset: this.xOffset, yOffset });
}
```

**`i - 1 * DAY_COLUMN_LENGTH` is `i - 110`, not `(i - 1) * 110`.** Multiplication
binds tighter than subtraction, so for every `i` in 0..9 the left operand is
between -110 and -101 and the comparison is true for any non-negative offset -
the guard asks nothing. The corrected form is already written two lines down in
the assignment. The document reprints the defect verbatim at its line 86
(`HW-02-0098`).

**The double scroll is the subtler problem.** The handler scrolls the list it
was called on, and assigning `this.xOffset` also fires
`@Watch('onCountUpdated')`, which scrolls *both* controllers. When the user
settles back on the column they started from, the assignment writes the same
number, `@Watch` does not fire, and only the touched list moves - so the date
header and the schedule body end up showing different days.

**An event block - `ThreeDayViewCalendar.zip#entry/src/main/ets/pages/ScheduleCard.ets:25`** (corrected, see `HW-02-0093`)

```typescript
build() {
  Column() {
    // FIX: the sample writes this markup twice, once under if (isFirstDay) with an
    //      always-true inner test, once under else if (isContainDay).
    if (this.schedule.isFirstDay(this.day) || this.schedule.isContainDay(this.day)) {
      Row() {
        Column() {
          Text(this.schedule.title).fontSize(12).fontWeight(700).fontColor(Color.White);
          Text(`${this.schedule.startTime}- ${this.schedule.endTime}`)
            .fontSize(12).fontWeight(400).fontColor(Color.White);
        }
        .justifyContent(FlexAlign.Center)
        .width('100%');
      }
      .borderRadius(12)
      .width(80)
      .height(LayoutCalculator.getScheduleHeight(this.schedule))
      .backgroundColor(this.schedule.color)
      .position({ x: 0, y: LayoutCalculator.getScheduleTopPosition(this.schedule, this.firstHour) });
    }
  };
}
```

**`.position()` inside the day column is what makes the card float at its time.**
`position` takes the node out of the `Column`'s flow, so the cards do not stack
- each one sits at its own computed `y` and overlapping events overlap visually,
which is what a day view wants.

Note that `DayColumn` hands **every** schedule to **every** day
(`DayColumn.ets:27`), and each `ScheduleCard` decides internally whether to draw.
With N days and M schedules that is N x M components, most rendering nothing. For
a real calendar, filter per day before the `ForEach`.

**The editor sheet - `ThreeDayViewCalendar.zip#entry/src/main/ets/pages/CalendarView.ets:82` and `:122`** (as shipped)

```typescript
import { SheetBuilder } from './SheetBuilder';     // the component

@Builder
SheetBuilder() {                                    // the local wrapper, same name
  SheetBuilder({ isShow: this.isShow, schedules: this.schedules });
}

// ...
Image($r('app.media.add'))
  .width(40)
  .height(40)
  .onClick(() => {
    this.isShow = true;
  })
  .bindSheet($$this.isShow, this.SheetBuilder(), {
    height: 467,
    showClose: false
  });
```

**`$$this.isShow` is what lets the sheet close itself.** The two-way binding
means the sheet's own cancel and save handlers - which take `isShow` as `@Link`
(`SheetBuilder.ets:28`) and set it to `false` - dismiss the sheet without the
page observing anything.

The local `@Builder` and the imported component share the name `SheetBuilder`;
inside the builder body the call resolves to the imported component. It works,
but it is worth renaming one of them. The document's snippet omits the builder
entirely, so the two `@Link` arguments never appear (`HW-02-0098`).

`schedules` is `@LocalStorageLink('schedules')` on the page and `@Link` in the
sheet, so `this.schedules.push(...)` in the sheet's save handler
(`SheetBuilder.ets:368`) reaches every `DayColumn` without any explicit refresh.

**The day source - `ThreeDayViewCalendar.zip#entry/src/main/ets/model/DayInfo.ets:148`** (corrected, see `HW-02-0094`)

```typescript
public pushData(data: DayInfo | DayInfo[]): void {
  const start = this.dataArray.length;                 // FIX
  if (Array.isArray(data)) {
    this.dataArray.push(...data);
    for (let i = 0; i < data.length; i++) {            // FIX: one notification per element
      this.notifyDataAdd(start + i);
    }
  } else {
    this.dataArray.push(data);
    this.notifyDataAdd(start);
  }
}
```

The shipped version emits a single `notifyDataAdd(this.dataArray.length - 1)`
however many items were appended, and the page appends ten at once
(`CalendarView.ets:59-63`). It survives only because that happens in
`aboutToAppear`, before either list has rendered.

## Permissions & config

None. `ThreeDayViewCalendar.zip#entry/src/main/module.json5` declares no
`requestPermissions` block.

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "pages": "$profile:main_pages",
    "abilities": [{
      "name": "EntryAbility",
      "srcEntry": "./ets/entryability/EntryAbility.ets",
      "exported": true,
      "skills": [{ "entities": ["entity.system.home"], "actions": ["action.system.home"] }]
    }],
    "extensionAbilities": [{
      "name": "EntryBackupAbility",
      "srcEntry": "./ets/entrybackupability/EntryBackupAbility.ets",
      "type": "backup",
      "exported": false,
      "metadata": [{ "name": "ohos.extension.backup", "resource": "$profile:backup_config" }]
    }]
  }
}
```

Root `build-profile.json5` targets `6.0.0(20)`.

`EntryAbility` sets full-screen layout, writes `topRectHeight` and
`bottomRectHeight` into `AppStorage`, and subscribes to `avoidAreaChange`
without releasing it (`HW-02-0097`). The page consumes only the top inset, and
does so with `safeAreaPadding` rather than `padding` (`CalendarView.ets:246`) -
the right choice for a scrolling page.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 116-118).
- The layout is tuned for a single screen width: `DAY_COLUMN_LENGTH = 110`
  against columns sized `'30%'` agrees only near 366 vp (`HW-02-0090`), while
  the module declares phone, tablet and 2in1.
- The hour range is 8 to 25 inclusive - eighteen rows including a `25:00`
  (`HW-02-0096`) - and the scrollable extent is a flat 120 % of the viewport, so
  the later hours are unreachable (`HW-02-0095`).
- `ScheduleItem` carries both a `Date` range and separate `"HH:mm"` strings, and
  they are not derived from each other: the bundled 4-day event has times
  `9:00`-`10:00`, so it draws a one-hour block on each of its days.
- `getVisibleDaysFrom` can return a negative number when the reference date is
  past `endDate`, which `getScheduleWidthPercent` multiplies by 80.
- Every schedule is passed to every day column; only `ScheduleCard`'s internal
  test stops it drawing.
- The ten days are generated once in `aboutToAppear` from `Date.now()`; there is
  no paging into the previous month and no way to reach yesterday.
- Schedules live in `LocalStorage` for the session only - nothing persists.

## Pitfalls

- **`HW-02-0089` - `i - 1 * DAY_COLUMN_LENGTH` is a precedence bug.** It
  evaluates as `i - 110`, so the snap-back guard is true for every non-negative
  offset. Parenthesise it as `(i - 1) * DAY_COLUMN_LENGTH`, in both copies - and
  note the document reprints the defect verbatim.
- **`HW-02-0090` - the snap pitch is a constant and the columns are a
  percentage.** 110 vp equals 30 % only near 366 vp; anywhere else the snap lands
  mid-column and the two strips drift apart.
- **`HW-02-0091` - the timeline row height and the card scale are unrelated
  numbers,** and the code calls the unit px while ArkUI reads a bare number as
  vp. Declare one `HOUR_ROW_HEIGHT` and set the row height explicitly instead of
  producing it with `.padding(19)`.
- **`HW-02-0092` - every snap scrolls twice,** once inline and once via
  `@Watch`; and when the offset does not change the watcher never fires, so only
  the touched list moves.
- **`HW-02-0093` - the snap block and the whole card markup are each duplicated
  verbatim,** and the card's inner condition is always true inside the branch
  that already established it.
- **`HW-02-0094` - `pushData` appends N items and notifies about one.**
- **`HW-02-0095` - the vertical scroll extent is `'120%'`** while the timeline is
  eighteen hour rows tall, so the end of the day cannot be scrolled to.
- **`HW-02-0096` - `DEFAULT_LAST_HOUR = 25` renders a `25:00` row,** and
  `DayColumn` hard-codes `firstHour: 8` where the constant already exists - the
  origin every card's position is measured from.
- **`HW-02-0097` - `on('avoidAreaChange')` has no `off()`.**
- **`HW-02-0098` - the document reprints the precedence bug,** names the backup
  ability `entrybackupability.ets` in the tree, and shows `bindSheet` without the
  `@Builder` wrapper or its two arguments.
- **`HW-02-0099` - six `@Provide` variables with no `@Consume` anywhere.**
- **Do not let the ruler scroll horizontally with the days.** Keeping it as a
  separate `Stack` child at a lower `zIndex` is what makes one ruler serve every
  column.
- **Do not give the day columns a percentage width if the snap uses a constant.**
  One of the two has to yield.
- **Do not hand every schedule to every column.** Filter by day before the
  `ForEach`; the sample builds N x M cards to render a handful.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `listDirection`, `edgeEffect`, `Scroller.currentOffset`/`scrollTo`, `onScrollStop`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-container-scroll.md` - `Scroll`, `ScrollDirection`, `friction`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scroll
- `documentation/harmonyos-guides/03_application-framework/arkts-layout-development-stack-layout.md` - `Stack` and child ordering
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-layout-development-stack-layout
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-location.md` - `position` and its removal from flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-location
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - `bindSheet`, `height`, `showClose`, and the `$$` two-way binding
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-lazyforeach.md` - `IDataSource`, the notification contract, key uniqueness
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-lazyforeach
- `documentation/harmonyos-guides/03_application-framework/arkts-localstorage.md` - `@LocalStorageLink` and mandatory local initialisation
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-localstorage
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `on`/`off('avoidAreaChange')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `LIFE-12` - the same industry's month-grid calendar, which has no sample archive
- `LIFE-08` - the schedule editor this sheet is a smaller version of
- `LIFE-01` - the industry shell this page would sit in
