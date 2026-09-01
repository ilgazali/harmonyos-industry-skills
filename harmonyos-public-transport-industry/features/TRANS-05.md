---
id: TRANS-05
title: Ride history with a date-range filter
industry: 06_public_transport
doc: huawei_industry_tree/06_public_transport/docs/05_ride_records.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/ride_records-0000002312400232
sample: huawei_industry_tree/06_public_transport/downloads/TicketDemo.zip
kits: ["@kit.ArkUI"]
apis: [List, ListItem, ForEach, CalendarPicker, "@ObservedV2", "@Trace", BorderRadiuses, promptAction.showToast]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-06-0031, HW-06-0032, HW-06-0033, HW-06-0034, HW-06-0051]
status: verified-with-fixes
---

## When to use

Load this card for any **history list the user narrows with a date range** -
ride records, transaction history, trip logs, order history, statements. The
shape is a filter header over a `List`, with a view model that re-runs the
filter whenever either bound changes.

It is also the corpus's cleanest example of **date handling that actually
works**, which is worth having after the maternity-health samples: see the
`DateUtil` snippet below.

## Feature checklist

- A header showing the active date range, tappable to change either bound.
- A list of records inside that range.
- Validation: a begin date later than the end date is rejected with a message.
- A reset control returning the filter to its default.
- Rounded corners on the first and last rows only, so the list reads as one card.

## Architecture

Single `entry` module, small and cleanly separated.

```
entry/src/main/ets
├── common/Constants.ets
├── components/FilteringComponent.ets   the date-range header UI
├── components/FilteringViewModel.ets   the two bounds, validation, reset
├── components/MineViewModel.ets        the record model and the filter query
├── pages/Index.ets
├── pages/MinePage.ets                  the list
└── util/DateUtil.ets                   one job: format a Date as YYYY/MM/DD
    util/ToastUtils.ets
entry/src/main/resources/rawfile/data.json   155 mock records
```

State uses **V2 decorators**: `@ObservedV2` on the classes and `@Trace` on the
array the list renders. `MineViewModel` owns a `FilteringViewModel` and hands it
a callback, so changing a bound flows back as `onChangeTimeCallback(begin, end)`
and re-runs the query. The filter UI never touches the record list directly.

That callback wiring is the part worth copying: the filter component knows
nothing about records, and the record view model knows nothing about pickers.

## Implementation steps

1. **Pick one date string format and use it for both the data and the bounds.**
   Zero-padded `YYYY/MM/DD` sorts lexicographically in chronological order,
   which is what makes the whole filter a two-character comparison.
2. **Format with numeric components and `padStart`**, never with a template
   that can emit a single digit.
3. **Filter with a plain string comparison** once both sides share the format.
4. **Validate the range and revert only what the user got wrong**
   (`HW-06-0033`).
5. **Seed the default range from the data**, not from the clock, if the data is
   fixed (`HW-06-0032`).
6. **Give `ForEach` a key generator** derived from the record, not the index.

## Verified snippets

All snippets are from `TicketDemo.zip`. Corrected forms are marked.

**Date formatting — `util/DateUtil.ets`** (as shipped)

```typescript
export class DateUtil {
  static getFormattedDate(date: Date = new Date()): string {
    let year = date.getFullYear();
    let month = String(date.getMonth() + 1).padStart(2, '0'); // 补零
    let day = String(date.getDate()).padStart(2, '0');
    return `${year}/${month}/${day}`;
  }
}
```

**Copy this one as-is.** It builds the string from numeric components with
explicit zero-padding, so it never produces the single-digit forms that break
comparison, and it never parses a string back into a `Date`. Compare the
maternity-health samples, which construct dates from unpadded template strings
and are wrong in three separate places as a result.

The zero-padding is what licenses the string comparison below: for
`YYYY/MM/DD`, lexicographic order and chronological order are the same.

**The filter query — `components/MineViewModel.ets`** (corrected, see `HW-06-0031`)

```typescript
import jsonData from 'resources/rawfile/data.json';

@ObservedV2
export class DataInfo {
  timeStr: string;
  price: string = '3.00';
  amountPaidOut: string = '2.70';
  state: string = '已完成';        // FIX: belongs in string.json - see HW-06-0034

  constructor(timeStr: string) {
    this.timeStr = timeStr ?? '';
  }

  static getData(begin: string, end: string): DataInfo[] {
    let tempModelAry: DataInfo[] = [];
    // FIX: the shipped version wraps this in `for (let i = 0; i < INPUT_TIMES; i++)`
    // with i unused, so every match is pushed five times.
    for (let item of jsonData) {
      let datePart = item.time.split(' ')[0];   // "2025/05/31 08:00/10:00" -> "2025/05/31"
      if (datePart >= begin && datePart <= end) {
        tempModelAry.push(new DataInfo(item.time));
      }
    }
    return tempModelAry;
  }
}
```

`item.time.split(' ')[0]` is how one field carries both the day and the time
window: the record reads `"2025/05/31 08:00/10:00"`, the date is everything
before the space, and the rest is displayed as-is.

**Range state, validation and reset — `components/FilteringViewModel.ets`** (corrected, see `HW-06-0033`)

```typescript
export type DateType = 'begin' | 'end';
type ChangeTimeCallback = (begin: string, end: string) => void;

export class FilteringViewModel {
  onChangeTimeCallback: ChangeTimeCallback;
  beginTime: string;
  endTime: string;

  constructor(callback: ChangeTimeCallback) {
    this.applyDefaultRange();       // FIX: shipped code inlines this, and reSet
    this.onChangeTimeCallback = callback;   //      sets a different range
  }

  private applyDefaultRange() {
    const nowDate = new Date();
    nowDate.setDate(nowDate.getDate() - 7);
    this.beginTime = DateUtil.getFormattedDate(nowDate);
    this.endTime = DateUtil.getFormattedDate();
  }

  setTime(type: DateType, value: Date, uiContext: UIContext) {
    const previous = type === 'begin' ? this.beginTime : this.endTime;
    if (type === 'begin') {
      this.beginTime = DateUtil.getFormattedDate(value);
    } else {
      this.endTime = DateUtil.getFormattedDate(value);
    }
    if (this.beginTime > this.endTime) {
      ToastUtils.showToast($r('app.string.time_not_allowed'), uiContext);
      // FIX: shipped code calls reSet(), collapsing both bounds to today.
      // Revert only the field that was just changed.
      if (type === 'begin') { this.beginTime = previous; } else { this.endTime = previous; }
      return;
    }
    this.onChangeTimeCallback(this.beginTime, this.endTime);
  }

  reSet() {
    this.applyDefaultRange();
    this.onChangeTimeCallback(this.beginTime, this.endTime);
  }
}
```

`this.beginTime > this.endTime` as the validation is only correct because both
strings share the padded format - the same property the filter relies on.

**The list with card-shaped corners — `pages/MinePage.ets`** (as shipped, plus a key)

```typescript
List() {
  ForEach(this.vm.dataAry, (item: DataInfo, index: number) => {
    ListItem() {
      this.buildItem(item);
    }
    .clip(true)
    .borderRadius(this.vm.getBorderRadius(index));
  }, (item: DataInfo, index: number) => `${item.timeStr}-${index}`)   // FIX: shipped code has no key
}
.clip(true)
.divider({
  strokeWidth: Constants.LIST_DIVIDER_STROKEWIDTH,
  color: Constants.LIST_DIVIDER_COLOR,
  startMargin: Constants.LIST_DIVIDER_START_MARGIN
})
```

```typescript
// components/MineViewModel.ets
getBorderRadius(index: number): BorderRadiuses | number {
  if (index === 0) {
    return { topLeft: Constants.SET_BORDER_RADIUS, topRight: Constants.SET_BORDER_RADIUS };
  }
  if (index === this.dataAry.length - 1) {
    return { bottomLeft: Constants.SET_BORDER_RADIUS, bottomRight: Constants.SET_BORDER_RADIUS };
  }
  return 0;
}
```

Rounding only the first and last rows, with `.clip(true)` on both the item and
the list, is the standard way to make a `List` read as a single grouped card.
Note it returns `BorderRadiuses | number` - `0` for the middle rows.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

Data lives at `entry/src/main/resources/rawfile/data.json` and is pulled in with
a module import (`import jsonData from 'resources/rawfile/data.json'`), so it is
resolved at build time rather than read through `resourceManager`.

Resource directories: `base`, `dark`, `rawfile` - **no `en_US`**
(`HW-06-0034`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. The industry's architecture practice
  requires API 24, so a combined project sits at 24.
- All 155 records are mock, dated 31 May to 30 June 2025, with a fixed fare on
  every row.
- The whole data set is filtered in memory on every bound change. Fine for 155
  rows; a real history needs the range pushed down to the query.
- The date comparison depends on the padded `YYYY/MM/DD` format. Change the
  format on either side and the filter silently returns wrong results.

## Pitfalls

- **`HW-06-0031` — every record is returned five times.** `getData` wraps its
  filter in `for (let i = 0; i < INPUT_TIMES; i++)` with `i` unused, and
  `INPUT_TIMES` is 5. The document's step 1 prints the loop verbatim.
- **`HW-06-0032` — the sample opens empty.** The default range is the last seven
  days from today; every record is from mid-2025. The two windows have no
  overlap and the gap grows daily.
- **`HW-06-0033` — reset does not restore the default.** The constructor opens
  on a seven-day window; `reSet` sets both bounds to today. It is also the
  recovery path for an invalid range, so one bad pick discards the good bound
  too.
- **`HW-06-0034` — the row status is a hardcoded Chinese literal** in the model,
  and the module ships no `en_US` directory at all. `price` and `amountPaidOut`
  are also fixed defaults, so every row shows the same fare.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `divider`, `ListItem`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - `ForEach` default key generation and why to supply one
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-guides/03_application-framework/arkts-new-observedv2-and-trace.md` - `@ObservedV2` / `@Trace`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-observedv2-and-trace
- `TRANS-01` - the industry architecture; ride history is one of its Mine-tab entries
