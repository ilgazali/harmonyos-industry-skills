---
id: EDU-18
title: Study-progress line chart - two series over a week or a month with @ohos/mpchart
industry: 04_education
doc: huawei_industry_tree/04_education/docs/18_track_study_time.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/track_study_time-0000002391113388
sample: huawei_industry_tree/04_education/downloads/TrackStudyTime.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit"]
apis: [LineChart, LineChartModel, LineData, LineDataSet, ILineDataSet, EntryOhos, JArrayList, XAxis, XAxisPosition, YAxis, YAxisLabelPosition, IAxisValueFormatter, Description, Legend, setValueFormatter, setLabelCount, setVisibleXRangeMaximum, notifyDataSetChanged, invalidate, "@Watch", "@Prop", "@LocalStorageLink", "@StorageProp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-04-0131, HW-04-0132, HW-04-0133, HW-04-0134, HW-04-0135, HW-04-0136, HW-04-0137, HW-04-0138, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card when you need a **line chart** - study minutes per day, words
learned per week, scores over a term - and specifically when you need two series
on one axis with a period switch above it.

The first thing to know is that **ArkUI has no chart component.** This sample
uses `@ohos/mpchart`, a third-party OHPM package, and that is the significant
decision: it brings an external dependency into an application that otherwise
needs only the SDK. The document names the library once and never mentions that
it must be installed (`HW-04-0134`).

The second thing is the **shape of an mpchart integration**, which is not
ArkUI-idiomatic: you build a `LineChartModel` imperatively in `aboutToAppear`,
hand it to a `LineChart` component, and redraw by calling `invalidate()` on the
model rather than by changing state. Understanding where that boundary sits is
most of the work.

## Feature checklist

- Two toggle buttons switch the whole page between a weekly and a monthly view.
- Three charts per view: vocabulary learned vs revised, and study time.
- Each chart draws one or two series with no point markers and no value labels.
- The x axis is formatted as dates (`8/12`) in the weekly view and as months
  (`7月`) in the monthly view, with the last point labelled 本日 / 本月.
- A left axis with a fixed min/max supplied per chart, no right axis, no legend,
  no description, no zoom.
- A summary row below each chart showing the latest value of each series.

## Architecture

One `entry` module, three pages, no view model beyond a data interface.

```
entry/src/main/ets
├── constants/CommonConstants.ets    layout percentages
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── model/ChartsModel.ets            ChartsModel interface + CHARTS_DATA
└── pages
    ├── MainPage.ets                 @Entry
    ├── ChartsPage.ets               the period buttons and the six data arrays
    └── LineCharts.ets               the reusable chart component + two axis formatters
```

The documented tree matches the zip.

**`LineCharts` is a generic wrapper with ten `@Prop`s**, and that is the piece to
lift:

| `@Prop` | Purpose |
| --- | --- |
| `trendData`, `trendData1` | the two series |
| `duringMinValue`, `duringMaxValue` | the left-axis range |
| `timeStyle` | `'week'` or `'month'` - selects the x-axis formatter |
| `selectedIndex` | 0-3 - selects which legend labels to show |
| `unit`, `textLeft`, `numLeft`, `textRight`, `numRight` | the summary row |

**The period switch works by reconstruction, not by re-binding.** `ChartsPage`
puts the chart inside `if (this.currentIndex === 0) { LineCharts({... 'week'
...}) } else { LineCharts({... 'month' ...}) }`, so switching destroys one
component and builds the other, and `aboutToAppear` re-runs with the new
`timeStyle`. That is what makes the axis configuration - which happens only in
`aboutToAppear` - correct after a switch.

It also means the `@Watch`-driven `onDataChange` path the document presents as
step 3 never actually runs (`HW-04-0136`): the arrays are literal constants that
nothing writes, and the period switch replaces the component rather than changing
its props.

**Two `IAxisValueFormatter` implementations do the labelling,** and both have
date-arithmetic defects (`HW-04-0131`, `HW-04-0132`, `HW-04-0133`).

## Implementation steps

1. **Add the dependency to the module that uses it.** `ohpm install @ohos/mpchart`
   and declare it in `entry/oh-package.json5`, not only at the project root
   (`HW-04-0135`). Pin the version exactly.
2. **Build the model once, in `aboutToAppear`:** create a `LineChartModel`,
   disable what you do not want (description, legend, zoom, grid background),
   configure the axes, then `setData`.
3. **Convert your arrays to mpchart's collections.** `JArrayList<EntryOhos>` with
   `new EntryOhos(index, value)` per point - the library does not take plain
   arrays.
4. **Set the x-axis label count and formatter from the period,** and remember
   that this only runs at construction - so the host must reconstruct the
   component when the period changes, as `ChartsPage` does.
5. **Do the date arithmetic with `Date`, not with integers.** `setDate` borrows
   across month and year boundaries; subtracting from `getDate()` does not
   (`HW-04-0131`).
6. **Guard optional numbers with `!== undefined`,** never with truthiness - a
   zero-based month is falsy in January (`HW-04-0132`).
7. **Redraw with `invalidate()`**, after `setData` and `notifyDataSetChanged`.
   Do not expect a state change to repaint the chart.
8. **Keep the model in a plain field**, not `@State` - nothing observes it
   (`HW-04-0138`).

## Verified snippets

All snippets are from `TrackStudyTime.zip`. Corrected forms are marked.

**Building the model — `entry/src/main/ets/pages/LineCharts.ets`** (as shipped)

```typescript
import {
  JArrayList, XAxis, XAxisPosition, YAxis, Description, Legend, EntryOhos,
  YAxisLabelPosition, LineDataSet, ILineDataSet, LineData, LineChart,
  LineChartModel, IAxisValueFormatter
} from '@ohos/mpchart';

aboutToAppear(): void {
  this.model = new LineChartModel();
  this.model.setPinchZoom(false);
  this.model.setDrawGridBackground(false);
  this.model.getDescription()?.setEnabled(false);   // the library's watermark
  this.model.getLegend()?.setEnabled(false);        // we draw our own legend row

  this.xAxis = this.model.getXAxis();
  if (this.xAxis) {
    this.xAxis.setPosition(XAxisPosition.BOTTOM);
    this.xAxis.setDrawGridLines(false);
    this.xAxis.setGranularity(1);                   // one label per data point, never fractional
    this.xAxis.enableGridDashedLine(2, 2, 0);
    const today = new Date();
    if (this.timeStyle === 'week') {
      this.xAxis.setValueFormatter(new XValueFormatterWeek(today, this.context));
      this.xAxis.setLabelCount(XYNumber.X_WEEK);
    } else if (this.timeStyle === 'month') {
      this.xAxis.setValueFormatter(new XValueFormatterMonth(today.getMonth() + 1, this.context));
      this.xAxis.setLabelCount(XYNumber.X_MONTH);   // FIX: sample passes getMonth(), zero-based
    }
  }

  this.leftAxis = this.model.getAxisLeft();
  if (this.leftAxis) {
    this.leftAxis.setLabelCount(XYNumber.Y, false);       // false: do not force exact spacing
    this.leftAxis.setPosition(YAxisLabelPosition.OUTSIDE_CHART);
    this.leftAxis.setAxisMinimum(this.duringMinValue);    // fixed range, supplied per chart
    this.leftAxis.setAxisMaximum(this.duringMaxValue);
    this.leftAxis.setAxisLineColor(Color.Transparent);    // grid lines only, no axis rule
  }
  this.model.getAxisRight()?.setEnabled(false);

  this.model.setScaleEnabled(false);
  this.model.setData(this.getLineData());
  this.model.setVisibleXRangeMaximum(7);
  this.model.invalidate();
}
```

**`setAxisMinimum`/`setAxisMaximum` with values passed in per chart** is what
makes the wrapper reusable: the vocabulary chart and the time chart have
different scales, and neither wants mpchart's autoscaling, which would make two
charts on one screen incomparable.

**`setGranularity(1)`** stops the library interpolating fractional x values into
labels - essential when the x axis is a day index rather than a continuous
quantity.

**Everything is configured before `setData`,** and `invalidate()` is the paint
call. This is an imperative charting API living inside a declarative UI: state
management does not repaint it, `invalidate()` does.

**Array to chart data — same file** (corrected, see `HW-04-0137`)

```typescript
private getLineData(): LineData {
  const values: JArrayList<EntryOhos> = new JArrayList<EntryOhos>();
  let i: number = 0;
  this.trendData.forEach((value: number) => { values.add(new EntryOhos(i++, value)); });

  const values1: JArrayList<EntryOhos> = new JArrayList<EntryOhos>();
  let j: number = 0;
  this.trendData1.forEach((value: number) => { values1.add(new EntryOhos(j++, value)); });

  const dataSetList: JArrayList<ILineDataSet> = new JArrayList<ILineDataSet>();

  const dataSet: LineDataSet = new LineDataSet(values, 'Dataset1');
  dataSet.setLineWidth(2);
  dataSet.setDrawCircles(false);
  dataSet.setColorByColor(0xFF2E94FF);
  dataSet.setDrawValues(false);        // FIX: sample sets true, then setValueTextSize(0)
  dataSetList.add(dataSet);

  const dataSet1: LineDataSet = new LineDataSet(values1, 'Dataset2');
  dataSet1.setLineWidth(2);
  dataSet1.setDrawCircles(false);
  dataSet1.setColorByColor(Color.Green);
  dataSet1.setDrawValues(false);
  dataSetList.add(dataSet1);

  return new LineData(dataSetList);
}
```

**`JArrayList` and `EntryOhos` are mpchart's own types** - a plain `number[]` is
not accepted anywhere. The x value is the array index, so the axis formatter is
what turns 0-6 into dates; the data carries no dates at all.

**`setDrawCircles(false)`** is what makes it a trend line rather than a
scatter-with-line; with seven points and two series the markers would dominate.

**Redrawing on a data change — same file** (as shipped)

```typescript
@Prop @Watch('onDataChange') trendData: Array<number>;
@Prop @Watch('onDataChange') trendData1: Array<number>;

onDataChange() {
  if (this.model) {
    this.model.setData(this.getLineData());
    this.model.notifyDataSetChanged();   // recompute the library's cached min/max
    this.model.invalidate();             // repaint
  }
}
```

**The three calls are not interchangeable.** `setData` replaces the data,
`notifyDataSetChanged` makes the library recompute the ranges it caches, and
`invalidate` schedules the repaint. Omit the middle one and the new data is drawn
against the old axis bounds.

Note that in this sample the path is unreachable (`HW-04-0136`): the six arrays
are literal constants and the period switch reconstructs the component. Keep the
`@Watch` if your data is live; understand that the sample never exercises it.

**The axis formatters — same file** (corrected, see `HW-04-0131`, `HW-04-0132`, `HW-04-0133`)

```typescript
class XValueFormatterWeek implements IAxisValueFormatter {
  constructor(private today?: Date, private context?: Context) {}

  getFormattedValue(value: number): string {
    if (this.today && value < XYNumber.X_WEEK - 1) {
      // FIX: the sample computes `today.getDate() - 6 + value` and pairs it with
      //      today's month, so the first six days of any month give 8/-3, 8/0 ...
      const d = new Date(this.today);
      d.setDate(this.today.getDate() - (XYNumber.X_WEEK - 1) + value);   // setDate borrows
      return `${d.getMonth() + 1}/${d.getDate()}`;
    }
    return this.context?.resourceManager.getStringByNameSync('this_day') ?? 'Unknown';
  }
}

class XValueFormatterMonth implements IAxisValueFormatter {
  constructor(private month?: number, private context?: Context) {}   // one-based

  getFormattedValue(value: number): string {
    // FIX: the sample guards `if (this.month && ...)`, and getMonth() is 0 in January
    if (this.month !== undefined && this.context && value < XYNumber.X_MONTH - 1) {
      // FIX: the sample uses XYNumber.X_WEEK here, in the MONTH formatter
      const m = this.month - (XYNumber.X_MONTH - 2) + value;
      const label = m > 0 ? m : m + 12;                                // wrap into the previous year
      return `${label}` + this.context.resourceManager.getStringByNameSync('month');
    }
    return this.context?.resourceManager.getStringByNameSync('this_month') ?? 'Unknown';
  }
}

class XYNumber {
  public static readonly X_DAY: number = 48;
  public static readonly X_WEEK: number = 7;
  public static readonly X_MONTH: number = 7;
  public static readonly X_YEAR: number = 12;
  public static readonly Y: number = 6;
}
```

**`IAxisValueFormatter` is where the meaning lives.** The data is indices; the
formatter is the only thing that knows index 6 is today. Two of them, chosen by
`timeStyle`, is the whole period mechanism.

**The last point is deliberately not a date.** Both formatters fall through to
本日 / 本月 for the final index - anchoring the axis at "now" reads better than a
date the user has to compare against today.

**Three date bugs live in fifteen lines here**, and all three are the same
mistake in different forms: doing calendar arithmetic on integers instead of on
`Date`, and treating a zero-based month as if it were one-based.

## Permissions & config

None. `entry/src/main/module.json5` declares no `requestPermissions` block.

The dependency, which belongs in `entry/oh-package.json5` (`HW-04-0135`):

```json5
{
  "dependencies": {
    "@ohos/mpchart": "3.0.11"      // FIX: sample declares "^3.0.11" at the project root only
  }
}
```

Run `ohpm install` before opening the project; the document does not say so
(`HW-04-0134`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Requires the third-party `@ohos/mpchart` package**, which the document's
  constraints section does not mention. Everything charting-related here is that
  library's API, not the SDK's.
- **Both periods show seven points.** `X_WEEK` and `X_MONTH` are both 7 and
  `setVisibleXRangeMaximum(7)` is hard-coded, so the "monthly" view is the last
  seven months, not the days of a month.
- All six data arrays are literal constants in `ChartsPage`; nothing writes them,
  and `CHARTS_DATA` in `model/ChartsModel.ets` - three July 2025 entries with
  zero values - is not used at all.
- The period switch reconstructs the chart component, so any per-chart state is
  lost on every switch.
- `CommonConstants` carries `VERIFY_NUMBER_LIST`, `HALF_HEIGHT` and `FULL_PARENT`
  from another sample; none is used here.
- The chart is fixed at 70 % of its container's height, with the legend row above
  and the summary row below.

## Pitfalls

- **`HW-04-0131` — the weekly labels subtract six from `getDate()` with no
  borrow,** so the first six days of every month produce 8/0 and 8/-3, all
  stamped with the current month. Use `Date.setDate`.
- **`HW-04-0132` — the monthly formatter is constructed with `getMonth()`,**
  which is 0 in January, and guarded with truthiness - so all seven labels read
  本月 for a month a year.
- **`HW-04-0133` — the monthly formatter offsets by `X_WEEK`,** correct only
  because `X_WEEK` and `X_MONTH` are both 7.
- **`HW-04-0134` — the constraints omit the OHPM dependency** the sample is built
  on, its version, and the `ohpm install` step.
- **`HW-04-0135` — the dependency is declared only at the project root** while
  `entry`, the module that imports it, declares none.
- **`HW-04-0136` — the documented `@Watch` refresh never runs.** The data is
  constant and the period switch reconstructs the component.
- **`HW-04-0137` — value labels are enabled three times and hidden by
  `setValueTextSize(0)`** rather than turned off.
- **`HW-04-0138` — the chart model is `@State`** although it is only mutated in
  place and repainted by `invalidate()`.
- **Do not expect state management to repaint an mpchart.** `setData` +
  `notifyDataSetChanged` + `invalidate`, in that order, is the contract.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-watch.md` - `@Watch` and when it fires
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-watch
- `documentation/harmonyos-guides/03_application-framework/arkts-localstorage.md` - `@LocalStorageLink`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-localstorage
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-ifelse.md` - conditional rendering, which is what actually swaps the chart here
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-ifelse
- `documentation/harmonyos-guides/02_environment-setup/ide-ohpm-system-settings.md` - configuring OHPM before an external dependency will resolve
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ide-ohpm-system-settings
- https://ohpm.openharmony.cn/#/cn/detail/@ohos%2Fmpchart - the charting library's own API reference; nothing in this card's chart code is SDK API
- `EDU-14` - the other progress-visualisation card in this industry, built from ArkUI primitives with no third-party dependency
