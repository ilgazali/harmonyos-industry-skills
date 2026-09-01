---
id: MAT-04
title: Child growth curve chart
industry: 10_maternity_health
doc: huawei_industry_tree/10_maternity_health/docs/04_growth_record_curve.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/growth_record_curve-0000002281357289
sample: huawei_industry_tree/10_maternity_health/downloads/BabyGrowthRecordCurve.zip
kits: ["@kit.ArkUI", "@kit.ArkTS", "@kit.AbilityKit"]
third_party: ["@ohos/mpchart ^3.0.11"]
apis: [LineChartModel, LineData, LineDataSet, XAxis, YAxis, IAxisValueFormatter, Description, Legend, Tabs, DatePicker, bindSheet, NavPathStack, LocalStorage, util.format]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-10-0024, HW-10-0025, HW-10-0026, HW-10-0027, HW-10-0028, HW-10-0029, HW-10-0030, HW-10-0031, HW-10-0032, HW-10-0033]
status: verified-with-fixes
---

## When to use

Load this card when a screen must plot **a user-entered measurement series
against time** - child height and weight curves, maternal weight gain, blood
pressure or glucose trends, sleep duration. It covers the chart itself, the
entry form that feeds it, and how to refresh the chart when the series changes.

The charting library is third-party: `@ohos/mpchart` from ohpm. There is no
first-party line chart in this practice.

## Feature checklist

- Two tabs, one per measured quantity (height, weight), each with its own curve.
- Below each curve, the list of records backing it.
- An add-record page with a date picker and numeric inputs for both
  measurements.
- Adding a record updates the curve immediately, without leaving the page.
- Records are kept in date order; a record for an existing day replaces it.
- The x-axis is labelled with the record dates, not raw indices.
- The visible date span is shown as a caption above the chart.

## Architecture

Single `entry` module.

```
entry/src/main/ets
├── constants/CommonConstants.ets
├── pages/MainPage.ets       Tabs host, record lists, chart captions
├── pages/LineCharts.ets     reusable chart component + axis formatter
├── pages/AddRecord.ets      date picker sheet + numeric entry + save
└── viewmodel/GrowthRecord.ets  @Observed record class
```

State lives in `LocalStorage`, shared between `MainPage` and `AddRecord` through
`@LocalStorageLink`. `LineCharts` is a presentational component: it takes the
series as `@Prop trendData` and watches it.

**The sample keeps four parallel arrays** - `growthRecordList`, `heightList`,
`weightList`, `dateList` - and hand-maintains them in lockstep with a hardcoded
index offset. Do not copy that. Keep one record list and derive the series
(`HW-10-0025`).

Refresh path: a write to the series fires `@Watch('onDataChange')`, which
recomputes the x-axis maximum, rebuilds `LineData`, and calls
`notifyDataSetChanged()` then `invalidate()` on the model. mpchart **is**
data-driven through that path - the sample carries a comment claiming otherwise
(`HW-10-0030`).

## Implementation steps

1. **Add the dependency** to `oh-package.json5`, pinned to an exact version.
2. **Build the model once in `aboutToAppear`**: create `LineChartModel`,
   disable description/legend/pinch-zoom, configure both axes.
3. **Give the x-axis an `IAxisValueFormatter`** that maps the index to a date
   label, and guard the array access inside it.
4. **Set `setGranularity(1)`** so ticks land on integer indices, and set the
   axis maximum to `length - 0.7` so the last point is not clipped at the edge.
5. **Watch the series, not the derived value** (`HW-10-0029`).
6. **On save, match an existing record by calendar day** - year, month and date
   (`HW-10-0024`) - then replace, insert in order, or append.
7. **Derive the date-picker range from the clock**, never from literals
   (`HW-10-0026`).

## Verified snippets

All snippets are from `BabyGrowthRecordCurve.zip`. Corrected forms are marked.

**Chart model setup — `pages/LineCharts.ets`, `aboutToAppear`** (as shipped)

```typescript
aboutToAppear(): void {
  this.model = new LineChartModel();
  this.model.setPinchZoom(false);
  this.model.setDrawGridBackground(false);

  const description: Description | null = this.model.getDescription();
  if (description) { description.setEnabled(false); }
  const legend: Legend | null = this.model.getLegend();
  if (legend) { legend.setEnabled(false); }

  this.xAxis = this.model.getXAxis();
  if (this.xAxis) {
    this.xAxis.setPosition(XAxisPosition.BOTTOM);
    this.xAxis.setDrawGridLines(false);
    this.xAxis.setAxisLineColor(0xff979797);
    this.xAxis.setTextColor(0xff979797);
    this.xAxis.setTextSize(10);
    this.xAxis.setAxisMinimum(-0.3);
    this.xAxis.setAxisMaximum(this.max);
    this.xAxis.setGranularity(1);                 // one tick per index
    this.xAxis.setLabelCount(XYNumber.X_WEEK);
    this.xAxis.setValueFormatter(new XValueFormatterMonth(this.dateList));
  }

  this.leftAxis = this.model.getAxisLeft();
  if (this.leftAxis) {
    this.leftAxis.setXOffset(10);
    this.leftAxis.setLabelCount(XYNumber.Y);
    this.leftAxis.setDrawGridLines(true);
    this.leftAxis.setPosition(YAxisLabelPosition.OUTSIDE_CHART);
    this.leftAxis.setAxisMinimum(this.duringMinValue);
    this.leftAxis.setAxisMaximum(this.duringMaxValue);
    this.leftAxis.setAxisLineColor(Color.Transparent);
  }
}
```

Note `setAxisMinimum(-0.3)` and `max = length - 0.7`: the half-unit padding at
both ends is what keeps the first and last markers from being clipped by the
plot border. Every getter is null-checked because mpchart returns `T | null`.

**Axis label formatter — same file** (as shipped)

```typescript
class XValueFormatterMonth implements IAxisValueFormatter {
  private dateList: Date[];

  constructor(dateList: Date[]) {
    this.dateList = dateList;
  }

  getFormattedValue(value: number): string {
    let date = this.dateList[value];
    if (!date || !(date instanceof Date) || isNaN(date.getTime())) {
      return '--';
    }
    let day = this.dateList[value].getDate();
    let month = this.dateList[value].getMonth() + 1;
    return `${month}/${day}`;
  }
}
```

The guard matters: mpchart calls the formatter with tick values that can fall
outside the data range, so an unguarded `this.dateList[value].getDate()` would
throw. This is the one place in the maternity samples where an array access is
properly defended - keep it.

**Refreshing the chart — same file** (corrected, see `HW-10-0029`)

```typescript
// FIX: the shipped version also puts @Watch('onDataChange') on `max`, which
// onDataChange assigns to on its first line - a watch that triggers itself.
@Prop @Watch('onDataChange') trendData: Array<number>;
@LocalStorageLink('dateList') @Watch('onDataChange') dateList: Date[] = [];
private max: number = 0;

onDataChange() {
  if (this.model) {
    this.max = this.dateList.length - 0.7;
    if (this.xAxis) {
      this.xAxis.setAxisMaximum(this.max);
    }
    this.model.setData(this.getLineData());
    this.model.notifyDataSetChanged();
    this.model.invalidate();
  }
}
```

`setData` → `notifyDataSetChanged` → `invalidate` is the required order, and it
is the whole refresh mechanism. No resize hack is needed.

**Using the chart — `pages/MainPage.ets`** (corrected, see `HW-10-0027`)

```typescript
LineCharts({
  topText: $r('app.string.ohos_id_elements_content_topText1'),
  rightText: $r('app.string.ohos_id_elements_content_rightText1'),
  unit: $r('app.string.unit1'),
  duringMinValue: 0,
  duringMaxValue: 80,
  duringTime: this.spanCaption(),   // FIX: shipped code indexes dateList[0] inline
  trendData: this.heightList,
})
  .height(240)
  .margin({ top: 12, bottom: 12 });

private spanCaption(): string {
  if (this.dateList.length === 0) { return ''; }
  const first = this.dateList[0];
  const last = this.dateList[this.dateList.length - 1];
  return `${first.getMonth() + 1}/${first.getDate()}-${last.getMonth() + 1}/${last.getDate()}`;
}
```

The shipped version builds that caption by indexing `dateList[0]` and
`dateList[length - 1]` directly in the component call, which throws the moment
the list is empty - and the seeded demo data is the only reason it is not.

**Inserting a record in date order — `pages/AddRecord.ets`** (corrected, see `HW-10-0024` and `HW-10-0025`)

```typescript
// FIX: shipped code compares getDay() (weekday!) and offsets growthRecordList
// by a hardcoded -6. Keep one list, match on the calendar day.
private sameDay(a: Date, b: Date): boolean {
  return a.getFullYear() === b.getFullYear()
    && a.getMonth() === b.getMonth()
    && a.getDate() === b.getDate();
}

const d = this.newGrowthRecord.dateTime;
const existing = this.growthRecordList.findIndex(r => this.sameDay(r.dateTime, d));
if (existing >= 0) {
  this.growthRecordList[existing] = this.newGrowthRecord;
} else {
  const after = this.growthRecordList.findIndex(r => r.dateTime > d);
  if (after >= 0) {
    this.growthRecordList.splice(after, 0, this.newGrowthRecord);
  } else {
    this.growthRecordList.push(this.newGrowthRecord);
  }
}
this.pathStack.pop();
```

Derive the chart series from that single list rather than maintaining three more
arrays beside it.

**Date picker sheet — `pages/AddRecord.ets`** (corrected, see `HW-10-0026`)

```typescript
DatePicker({
  start: this.earliestAllowed,   // FIX: shipped code hardcodes new Date('2025-8-5')
  end: new Date(),               // FIX: shipped code hardcodes new Date('2025-12-31')
  selected: this.selectedDate
})
  .lunar(this.isLunar)
  .onDateChange((value: Date) => {
    this.selectedDate = value;
  });
```

`.lunar()` is worth knowing about in this industry - Chinese users often record
birth and milestone dates on the lunar calendar.

**Getting the stack inside a NavDestination — same file** (as shipped)

```typescript
.onReady((context: NavDestinationContext) => {
  this.pathStack = context.pathStack;
});
```

This is the correct way for a destination to reach the stack that pushed it;
do not construct a second `NavPathStack` (the shipped file also declares an
unused one - `HW-10-0030`).

## Permissions & config

**None.** `entry/src/main/module.json5` declares no `requestPermissions`.

`oh-package.json5` (root):

```json5
{
  "modelVersion": "6.0.0",
  "dependencies": {
    "@ohos/mpchart": "3.0.11"   // pin exactly; the sample ships "^3.0.11"
  }
}
```

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Requires the third-party ohpm package `@ohos/mpchart`. The document does not
  list this in its constraints (`HW-10-0032`).
- State lives in `LocalStorage` only - it is process-lifetime, not persisted.
  Nothing survives a cold start.
- The y-axis bounds are passed in as fixed `duringMinValue` / `duringMaxValue`
  (0-80 for height), so out-of-range measurements are clipped rather than
  rescaling the axis.
- mpchart returns `null` from its getters; every `getXAxis()`, `getAxisLeft()`,
  `getDescription()` and `getLegend()` call needs a null check.

## Pitfalls

- **`HW-10-0024` — `getDay()` is the weekday, `getDate()` is the day of month.**
  The shipped duplicate check uses `getDay()`, so two records in the same month
  that share a weekday overwrite each other, and the year is never compared at
  all. This destroys user data on ordinary input.
- **`HW-10-0025` — never hardcode an offset between parallel arrays.** The
  sample writes `growthRecordList[index - 6]` because three of its arrays carry
  six seed values and one does not. When `index < 6` that is a negative index:
  the assignment silently creates a non-array property and the record disappears,
  and the splice inserts from the wrong end.
- **`HW-10-0026` — the date picker range has already expired.** It is pinned to
  5 Aug - 31 Dec 2025, and the default selection is today, outside that window.
  Anyone running the sample now cannot add a record at all.
- **`HW-10-0027` — non-ISO date literals.** `'2025-3-5'` parses as local time,
  `'2025-12-31'` as UTC, and the two are mixed in the same file. Verified: in
  `America/New_York`, `new Date('2025-12-31')` resolves to 30 December.
- **`HW-10-0028` — do not call `getStringByNameSync` in a render path.** The
  shipped list rows perform six synchronous resource lookups each, per render,
  while the same file uses `$r()` declaratively thirty lines above.
- **`HW-10-0029` — a `@Watch` must not write the state it watches.** The
  document's own step 3 snippet does exactly that; it terminates only because
  ArkUI suppresses no-op assignments.
- **`HW-10-0030` — dead state and a comment that misleads.** `weight`,
  `nowWeight` and a second `pageStack` are unused, and a comment claims mpchart
  cannot be refreshed by data - contradicted by the `invalidate()` call the same
  document teaches.
- **`HW-10-0031` — validation only rejects exactly zero,** so negative
  measurements enter the series and wreck the axis scale. `MAT-03`'s form in the
  same industry bounds its inputs; this one does not.
- **`HW-10-0032` — the constraints omit the charting dependency** and the
  version is a floating caret range while the SDK beside it is pinned exactly.

## References

- [`@ohos/mpchart` on ohpm](https://ohpm.openharmony.cn/#/cn/detail/@ohos%2Fmpchart) - the charting library; no local mirror in `documentation/`
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-datepicker.md` - `DatePicker`, `start` / `end` / `selected`, `lunar`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-datepicker
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `NavDestinationContext`, `onReady`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-guides/03_application-framework/arkts-localstorage.md` - `@LocalStorageLink` sharing between pages
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-localstorage
- `MAT-03` - the sibling record-entry practice in this industry; compare its validation and its date handling
