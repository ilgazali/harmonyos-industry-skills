---
id: SPORT-05
title: Periodic health charts - mpchart line charts over day, week, month and year
industry: 03_sports_health
doc: huawei_industry_tree/03_sports_health/docs/05_period_chart.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/period_chart-0000002280744357
sample: huawei_industry_tree/03_sports_health/downloads/PeriodChart.zip
kits: ["@kit.LocalizationKit", "@kit.ArkTS", "@kit.AbilityKit", "@kit.ArkUI"]
apis: ["@ohos/mpchart", LineChartModel, LineDataSet, LineData, XAxis, YAxis, Mode, IAxisValueFormatter, IValueFormatter, "resourceManager.getRawFileContentSync", "util.TextDecoder", "Intl.DateTimeFormat", "@Watch", "@StorageProp", "@Link", "@Prop", "@State", "UIAbility.onConfigurationUpdated"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-03-0023, HW-03-0024, HW-03-0025, HW-03-0054, HW-03-0055]
status: verified-with-fixes
---

## When to use

Load this card when a screen must **show one metric over a switchable period**
- heart rate, blood pressure, blood sugar, weight, temperature, calories -
with day, week, month and year views over the same underlying series. The
document names that range explicitly, and adds that the same shape suits
temperature and humidity outside health.

It is the corpus's only example of **`@ohos/mpchart`**, the OpenHarmony port
of MPAndroidChart, so it is the reference for using a real charting library
rather than drawing on a `Canvas` by hand. Compare `SPORT-07` and `SPORT-10`,
which draw their visuals from primitives.

The transferable design is the **separation between the series and the view**:
one raw dataset is loaded once, and four aggregation functions reduce it to
whatever the selected period needs. The chart component never sees a date.

## Feature checklist

- A line chart of heart rate with a filled gradient under the curve.
- A period selector: day, week, month, year.
- A date toolbar that steps backwards and forwards through periods.
- Day view plots every reading; week and month plot a daily average; year
  plots a monthly average.
- A headline figure above the chart - the last reading for a day, the average
  for the other periods - with the label changing to match.
- A marker bubble on touch showing the value and its period label.
- Labels follow the system language and update when it changes.

## Architecture

One `entry` module, split by responsibility rather than by screen.

```
entry/src/main/ets
├── common/CommonConstants.ets      period ids, chart colours, axis limits
├── components
│   ├── DateToolBar.ets             back/forward through periods
│   ├── HeaderBar.ets
│   ├── LineChartComponent.ets      the mpchart wrapper - takes numbers, knows no dates
│   └── PeriodSelection.ets         day / week / month / year
├── entryability/EntryAbility.ets   publishes systemLanguage, watches for changes
├── model
│   ├── ChartDataFormat.ets         DayChartData, TimeStatistic, ChartDataSet, IndicatorTimePeriod
│   ├── LabelValueFormat.ets        IValueFormatter for the marker
│   └── XAxisValueFormat.ets        IAxisValueFormatter for the x axis
├── pages/ChartPage.ets             @Entry - owns the series, the period and the two watches
└── utils
    ├── DateUtils.ets               period arithmetic and locale formatting
    ├── FileReaderUtils.ets         the rawfile read
    └── StatisticsUtils.ets         four aggregations, one per period
```

The documented tree matches the zip.

**The data path is one-way and re-derived, never mutated:**

```
heart_rate_data.json ──read once──> statistics: DayChartData[]   (raw, private, never changed)
                                            │
periodSelectedIndex ──@Watch──> updateTimePeriod() ──> timePeriod
                                                          │
                                        timePeriod ──@Watch──> updateStats()
                                                          │
                                     StatisticsUtils.getXxxDataForChart(timePeriod, statistics)
                                                          │
                                            chartData: number[] + periodData: string[]
                                                          │
                                              LineChartComponent(@Link both)
```

**Two chained `@Watch`es** is the pattern worth copying: changing the period
selector recomputes the date range, and changing the date range recomputes the
statistics. Neither handler knows about the other's trigger, so the date
toolbar can write `timePeriod` directly and the same recomputation follows.

**`LineChartComponent` takes `number[]` and `string[]`, nothing else.** All
date handling, locale formatting and aggregation stay in the page and the
utils, so the chart is reusable for any metric.

## Implementation steps

1. **Load the raw series once** in the page - asynchronously, and guarded
   (`HW-03-0024`).
2. **Model a day as `DayChartData`** with a `Date` and an array of
   `TimeStatistic`, so the four aggregations can all filter on the same field.
3. **Publish the system language to `AppStorage`** from the ability, and
   refresh it from `onConfigurationUpdated`.
4. **Bind `periodSelectedIndex` and `timePeriod` with `@Watch`** and let the
   chain do the recomputation.
5. **Write one aggregation per period**, all returning the same `ChartDataSet`
   shape, and guard the empty case identically in all four
   (`HW-03-0025`).
6. **Build month labels from the period's year** (`HW-03-0023`).
7. **Pass only numbers and labels to the chart component.**

## Verified snippets

All snippets are from `PeriodChart.zip`. Corrected forms are marked.

**Reading the series — `entry/src/main/ets/utils/FileReaderUtils.ets`** (corrected, see `HW-03-0024`)

```typescript
import { util } from '@kit.ArkTS';
import { resourceManager } from '@kit.LocalizationKit';

static async readChartDataFromRawfile(rm: resourceManager.ResourceManager): Promise<DayChartData[]> {
  try {
    const bytes = await rm.getRawFileContent('heart_rate_data.json');   // FIX: sample uses ...Sync
    const dataStr = FileReaderUtils.uint8ArrayToString(bytes);
    const dayData = JSON.parse(dataStr) as DayChartData[];
    dayData.forEach((value: DayChartData) => {                          // FIX: sample uses map and
      value.date = new Date(value.date);                                //      discards the result
    });
    return dayData;
  } catch (err) {
    hilog.error(0x0000, 'PeriodChart', `readChartData failed: ${(err as BusinessError).code}`);
    return [];                                                          // FIX: sample has no catch
  }
}

static uint8ArrayToString(array: Uint8Array) {
  const textDecoderOptions: util.TextDecoderOptions = { fatal: false, ignoreBOM: true };
  const decodeToStringOptions: util.DecodeToStringOptions = { stream: false };
  const textDecoder = util.TextDecoder.create('utf-8', textDecoderOptions);
  return textDecoder.decodeToString(array, decodeToStringOptions);
}
```

**`getRawFileContent` returns a `Uint8Array`, not a string**, and
`util.TextDecoder` is the conversion. `ignoreBOM: true` matters for a JSON file
that may have been saved with a byte-order mark - without it the leading
`﻿` makes `JSON.parse` throw on an otherwise valid file. `fatal: false`
keeps a malformed byte from throwing mid-decode.

Re-hydrating the dates after the parse is required: `JSON.parse` gives back
strings, and every aggregation calls `Date` methods on that field.

**The two chained watches — `entry/src/main/ets/pages/ChartPage.ets`** (as shipped)

```typescript
@StorageProp('systemLanguage') @Watch('updateTimePeriod') systemLanguage: string = 'en-Latn-US';
@State @Watch('updateTimePeriod') periodSelectedIndex: number = 0;
@State @Watch('updateStats') timePeriod: IndicatorTimePeriod =
  { periodDesc: '', beginDate: new Date(), endDate: new Date() };
private statistics: DayChartData[] = [];      // raw series: not @State, never re-rendered from

updateTimePeriod() {
  switch (this.periodSelectedIndex) {
    case PERIOD_DAY:
      this.timePeriod = {
        periodDesc: DateUtils.getDateLocaleStringLong(this.latestDate),
        beginDate: this.latestDate, endDate: this.latestDate
      };
      break;
    case PERIOD_WEEK:  this.timePeriod = DateUtils.getWeekPeriod(this.latestDate);  break;
    case PERIOD_MONTH: this.timePeriod = DateUtils.getMonthPeriod(this.latestDate); break;
    case PERIOD_YEAR:  this.timePeriod = DateUtils.getYearPeriod(this.latestDate);  break;
  }
}

updateStats() {
  let chartDataSet: ChartDataSet = { chartData: [], period: [], average: 0 };
  switch (this.periodSelectedIndex) {
    case PERIOD_DAY:
      chartDataSet = StatisticsUtils.getDayDataForChart(this.timePeriod, this.statistics);
      this.toolbarLabel = $r('app.string.label_last_measure');
      break;
    // ... week, month, year all use label_average_hr
  }
}
```

**Watching `systemLanguage` with the same handler as the period selector** is
the neat part: a language change re-runs `updateTimePeriod`, which rebuilds
`timePeriod` with a freshly formatted `periodDesc`, which triggers
`updateStats` and re-labels the axis. One locale change repaints the whole
chart through the existing chain, with no separate refresh path.

Keeping `statistics` as a plain private field rather than `@State` is
deliberate - it never changes, so there is nothing to observe and no reason to
pay for observation on an 82 KB array.

**The four aggregations — `entry/src/main/ets/utils/StatisticsUtils.ets`** (corrected, see `HW-03-0023`, `HW-03-0025`)

```typescript
static getDayDataForChart(timePeriod: IndicatorTimePeriod, statistics: DayChartData[]): ChartDataSet {
  const dayStats = statistics.filter((d: DayChartData) => DateUtils.isBetweenDate(timePeriod, d.date));
  const dataSet = dayStats.length ? dayStats[0].data.map((t: TimeStatistic) => t.value) : [];
  const period  = dayStats.length ? dayStats[0].data.map((t: TimeStatistic) => t.time) : [];
  return {
    chartData: dataSet, period: period,
    average: dataSet.length ? dataSet[dataSet.length - 1] : 0    // FIX: sample indexes unguarded
  };
}

static getYearDataForChart(timePeriod: IndicatorTimePeriod, statistics: DayChartData[]): ChartDataSet {
  const yearStats = statistics.filter((d: DayChartData) => DateUtils.isBetweenDate(timePeriod, d.date));
  const dataSet: number[] = [];
  const period: string[] = [];
  for (let monthIndex = 0; monthIndex < 12; monthIndex++) {
    const byMonth = yearStats.filter((d: DayChartData) => d.date.getMonth() === monthIndex)
      .map((d: DayChartData) => StatisticsUtils.calculateAverageStats(d.data));
    if (!byMonth.length) {
      continue;                                                  // months with no data are skipped,
    }                                                            // not plotted as zero
    dataSet.push(Math.round(byMonth.reduce((p: number, c: number) => p + c) / byMonth.length));
    period.push(DateUtils.getMonthLocaleString(
      new Date(timePeriod.beginDate.getFullYear(), monthIndex)));  // FIX: sample passes getMonth()
  }
  return { chartData: dataSet, period: period, average: StatisticsUtils.calculateAverage(dataSet) };
}

static calculateAverage(nums: number[]): number {
  if (!nums.length) {
    return 0;
  }
  return Math.round(nums.reduce((prev: number, curr: number) => prev + curr) / nums.length);
}
```

**All four return the same `ChartDataSet`,** so `updateStats` is a switch that
assigns rather than four different code paths. **Skipping empty months rather
than plotting zero** is the right call for a health series - a month with no
readings is absent, not a month of zero heart rate - and it is why `chartData`
and `period` must be pushed together.

`calculateAverage` guarding the empty case is the pattern the day function
should have followed.

**Locale-aware labels — `entry/src/main/ets/utils/DateUtils.ets`** (as shipped)

```typescript
static getMonthLocaleString(date: Date): string {
  let locale = AppStorage.get<string>('systemLanguage');
  if (!locale) {
    locale = 'en-Latn-US';
  }
  const dateFormat: Intl.DateTimeFormat = new Intl.DateTimeFormat(locale, { month: 'short' });
  return dateFormat.format(date);
}
```

with the ability keeping that key current:

```typescript
// entry/src/main/ets/entryability/EntryAbility.ets
systemLanguage = this.context.config.language;
AppStorage.setOrCreate('systemLanguage', systemLanguage);

onConfigurationUpdated(newConfig: Configuration): void {
  if (systemLanguage !== newConfig.language) {
    systemLanguage = newConfig.language;
    AppStorage.setOrCreate('systemLanguage', systemLanguage);
  }
}
```

**Reading the locale from `AppStorage` with a fallback, instead of hardcoding
a tag,** is what `TOUR-10` gets wrong in the tourism industry. Handling
`onConfigurationUpdated` is what makes it live: change the phone's language
and the axis relabels without a restart.

**The chart wrapper — `entry/src/main/ets/components/LineChartComponent.ets`** (as shipped)

```typescript
import { LineChartModel, LineDataSet, LineData, XAxis, YAxis, Mode, XAxisPosition, YAxisLabelPosition }
  from '@ohos/mpchart';

@Link @Watch('onDataChange') data: number[];
@Link @Watch('onPeriodChange') period: string[];
@Prop label: Resource;
@State model: LineChartModel = new LineChartModel();
@State dataSet: LineDataSet = new LineDataSet(null, '');
@State customUiInfo: CustomUiInfo = new CustomUiInfo(LABEL_UI_WIDTH, LABEL_UI_HEIGHT);   // the marker
```

**`@Link` on both inputs, `@Prop` on the label.** The two arrays are large and
change together, so a reference binding avoids copying them on every period
switch; the label is a small immutable resource and is copied. The two
`@Watch`es let the component rebuild its `LineDataSet` when either changes,
which is what a charting library needs - the model is imperative and has to be
told.

## Permissions & config

**None.** The sample declares no `requestPermissions` - the data is bundled.

The chart library is declared at project level:

```json5
// oh-package.json5 (project root)
"dependencies": {
  "@ohos/mpchart": "^3.0.21"
}
```

Data file: `entry/src/main/resources/rawfile/heart_rate_data.json`, 82 KB.

Resource directories carry the localised labels; the locale itself comes from
`AppStorage`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **`@ohos/mpchart` is a third-party OHPM package**, not part of the SDK, so
  it is versioned separately and is not covered by the HarmonyOS references -
  its API is documented at ohpm.openharmony.cn only.
- The series is a **bundled rawfile**, so the chart's date range is fixed by
  the asset: `getLatestDateInStats` picks the newest record as the starting
  view rather than using today's date.
- Aggregation is **linear over the whole series** on every period change -
  four `filter` passes for a year view. Fine for 82 KB; a multi-year series
  needs indexing by date.
- One metric only. Adding blood pressure or weight means another rawfile and
  another set of aggregations - the chart component itself needs no change.

## Pitfalls

- **`HW-03-0023` — the year chart's month labels are built from
  `new Date(beginDate.getMonth(), monthIndex)`,** passing a month where the
  year belongs, so every label date is in 1900. Masked because only the short
  month name is formatted.
- **`HW-03-0024` — the 82 KB file is read and parsed synchronously in
  `aboutToAppear`**, on the UI thread, with no `try`/`catch` around either the
  read or the parse.
- **`HW-03-0025` — the day view returns `dataSet[dataSet.length - 1]` without
  a length guard,** so a day with no readings yields `undefined` where the
  other three periods yield `0`.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getRawFileContent` and `getRawFileContentSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `documentation/harmonyos-references/02_application-framework/js-apis-util.md` - `util.TextDecoder`, `TextDecoderOptions`, `decodeToString`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-util
- `documentation/harmonyos-guides/03_application-framework/arkts-watch.md` - `@Watch` and chained handlers
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-watch
- `documentation/harmonyos-guides/03_application-framework/arkts-appstorage.md` - `@StorageProp`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-appstorage
- `documentation/harmonyos-references/02_application-framework/js-apis-app-ability-configuration.md` - `Configuration.language` and `onConfigurationUpdated`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-app-ability-configuration
- `documentation/harmonyos-guides/03_application-framework/i18n-time-date.md` - `Intl.DateTimeFormat` and locale-aware dates
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/i18n-time-date
- `SPORT-07`, `SPORT-10` - the two charts in this industry drawn from primitives instead of a library
