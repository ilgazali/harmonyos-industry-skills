---
id: FIN-04
title: Daily income and expenditure calendar
industry: 07_finance_insurance
doc: huawei_industry_tree/07_finance_insurance/docs/04_daily_revenue_and_expenditure_calendar_chart.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/daily_revenue_and_expenditure_calendar_chart-0000002294643216
sample: huawei_industry_tree/07_finance_insurance/downloads/CalendarOfIncomeAndExpense.zip
kits: ["@kit.ArkUI"]
apis: [Flex, UIContext.showDatePickerDialog, "@Local", ForEach, Date]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-07-0015, HW-07-0016, HW-07-0017, HW-07-0018, HW-07-0053]
status: verified-with-fixes
---

## When to use

Load this card for a **month grid whose cells carry per-day aggregates** -
income and expenditure per day, steps per day, bookings per day, anything where
the calendar itself is the chart. Tapping a day opens its detail list; the grid
pads with adjacent-month days so the weeks line up.

The layout half is sound. The data-placement half has two high-severity defects,
so read the pitfalls before reusing `calcFlows`.

## Feature checklist

- A month grid, weeks starting Sunday, padded at both ends with greyed
  adjacent-month days.
- Each cell shows the day number plus that day's income and expenditure totals.
- Tapping a cell selects it and shows the day's transactions beneath.
- Tapping a padding cell moves to that adjacent month.
- A date picker for jumping to an arbitrary month.

## Architecture

Single `entry` module, one page.

```
entry/src/main/ets
├── constants/CommonConstants.ets
├── model/SourceDataModel.ets    SourceData records + BillCell (one grid cell)
└── pages/FlowCalendar.ets       calcFlows builds the grid; Flex renders it
```

State uses **V2 decorators** (`@Local`). `BillCell` carries the day number, an
`isSupplemental` flag for padding cells, the two totals, and the day's records;
`addDetails` folds a record into a cell and updates its totals.

**The grid maths** is the reusable part:

```
startMissingDays = new Date(y, m - 1, 1).getDay()      // 0..6, leading pad
daysOfThisMonth  = new Date(y, m, 0).getDate()         // day 0 of next month
daysOfLastMonth  = new Date(y, m - 1, 0).getDate()     // day 0 of this month
endMissingDays   = 6 - new Date(y, m - 1, daysOfThisMonth).getDay()
cellIndex(day)   = day + startMissingDays - 1
```

`new Date(y, m, 0)` giving the last day of month `m-1` is the idiom worth
remembering - it avoids leap-year special cases entirely.

## Implementation steps

1. **Build the cell array in three passes**: leading pad from the previous
   month, the real days, then the trailing pad.
2. **Filter the records to the displayed month before placing them**
   (`HW-07-0015`).
3. **Bounds-check every cell index** before dereferencing (`HW-07-0016`).
4. **Mark padding cells** with a flag and restyle them in one pass afterwards -
   the sample does this cleanly.
5. **Build every `Date` from numeric components** (`HW-07-0017`).

## Verified snippets

From `CalendarOfIncomeAndExpense.zip`. Corrected forms are marked.

**Building the month grid — `pages/FlowCalendar.ets`** (corrected, see `HW-07-0015`, `HW-07-0016`)

```typescript
calcFlows(y: number, m: number, day: number) {
  const thisFlows: BillCell[] = [];
  const startDay = new Date(y, m - 1, 1);
  const daysOfThisMonth = new Date(y, m, 0).getDate();
  const startMissingDays = startDay.getDay();          // 0 = Sunday

  // lead-in from the previous month
  if (startMissingDays > 0) {
    const daysOfLastMonth = new Date(startDay.getFullYear(), startDay.getMonth(), 0).getDate();
    for (let i = daysOfLastMonth - startMissingDays + 1; i <= daysOfLastMonth; i++) {
      thisFlows.push(new BillCell(i, true, 0, 0, []));   // true = supplemental
    }
  }
  // this month
  for (let i = 1; i <= daysOfThisMonth; i++) {
    thisFlows.push(new BillCell(i, false, 0, 0, []));
  }

  // FIX: the shipped loop reads only getDate(), so records from any month land
  // in the displayed one. Filter on year and month first.
  for (const record of this.sourceData) {
    const d = record.payDate;
    if (d.getFullYear() !== y || d.getMonth() !== m - 1) {
      continue;
    }
    const idx = d.getDate() + startMissingDays - 1;
    if (idx < 0 || idx >= thisFlows.length) {           // FIX: shipped code indexes blind
      continue;
    }
    thisFlows[idx].addDetails(record);
  }

  // trailing pad, only when the month shown is not the current one
  const endMissingDays = 6 - new Date(y, m - 1, daysOfThisMonth).getDay();
  if (y !== this.today.getFullYear() || m !== this.today.getMonth() + 1) {
    for (let i = 1; i <= endMissingDays; i++) {
      thisFlows.push(new BillCell(i, true, 0, 0, []));
    }
  }

  // restyle every padding cell in one pass
  for (const cell of thisFlows) {
    if (cell.isSupplemental) {
      cell.changeCellStyle($r('app.color.background_color'),
        $r('app.color.supplemental_date_font_color'));
    }
  }
  this.flowCells = thisFlows;

  const showIndex = day + startMissingDays - 1;
  if (showIndex >= 0 && showIndex < this.flowCells.length) {   // FIX: shipped code does not check
    this.selectedDayFlow = this.flowCells[showIndex];
    this.selectedDayFlow.changeCellStyle($r('app.color.selected_date_cell_background'),
      $r('app.color.selected_date_font_color'));
  }
  this.selectedDate = new Date(y, m - 1, day);
}
```

Separating "is this a padding cell" from "how does a padding cell look" - a flag
at build time, a restyle pass afterwards - keeps the three build loops simple.

**The date picker — same file** (corrected, see `HW-07-0017`)

```typescript
this.getUIContext().showDatePickerDialog({
  start: new Date(2000, 0, 1),        // FIX: shipped code uses new Date('2000-1-1')
  end: new Date(),
  selected: this.selectedDate,
  onDateAccept: (value: Date) => {
    this.year = value.getFullYear();
    this.month = value.getMonth() + 1;
    this.calcFlows(this.year, this.month, value.getDate());
  }
});
```

`showDatePickerDialog` on `UIContext` is the current form - it needs no
`CustomDialogController` and no component-scoped builder.

**Month switching from a padding cell — same file** (as shipped, plus a clamp)

```typescript
.onClick(() => {
  if (item.isSupplemental) {
    // ... step year/month forward or back ...
  }
  // FIX: item.day belongs to the adjacent month and may exceed the new month's
  // length - clamp before passing it in
  const maxDay = new Date(this.year, this.month, 0).getDate();
  this.calcFlows(this.year, this.month, Math.min(item.day, maxDay));
})
```

## Permissions & config

**None.**

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Records are six hardcoded entries in `SourceDataModel.ets`, all June 2025.
- The whole record set is re-scanned on every month change; fine for six
  records, not for a real ledger.
- Weeks start on Sunday - `getDay()` returns 0 for Sunday, and the pad maths
  assumes it.

## Pitfalls

- **`HW-07-0015` — records are placed by day-of-month alone.** Year and month
  are never compared, so every month displays the same entries. The bundled data
  is June 2025 and the calendar opens on today, so every figure on screen is
  wrong out of the box.
- **`HW-07-0016` — two cell lookups have no bounds check.** A record dated the
  31st while February is shown, or a tap on a padding cell at a month boundary,
  indexes past the end and dereferences `undefined`.
- **`HW-07-0017` — `new Date('2000-1-1')`** is the one string-built date among
  twelve numeric ones in the same file.
- **`HW-07-0018` — the `year` and `month` fields carry dead initial values** that
  are overwritten in `aboutToAppear` and that disagree with the bundled data's
  month anyway.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-flex.md` - `Flex`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-flex
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `showDatePickerDialog`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-guides/03_application-framework/arkts-new-observedv2-and-trace.md` - the V2 state decorators
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-observedv2-and-trace
- `TRANS-05` - the other date-handling practice in this corpus; its `DateUtil` is the reference for date formatting
