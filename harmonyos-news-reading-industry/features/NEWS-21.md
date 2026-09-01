---
id: NEWS-21
title: Reading-time dashboard - a hand-drawn Canvas histogram redrawn from @Watch when the period changes
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/21_time_statistics.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/time_statistics-0000002362798665
sample: huawei_industry_tree/11_news_reading/downloads/TimeStatistics.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [Canvas, CanvasRenderingContext2D, RenderingContextSettings, onReady, clearRect, fillRect, setLineDash, arc, fillText, "@Prop", "@Watch", "@StorageProp", ForEach, "cryptoFramework.createRandom", "resourceManager.getStringSync"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-11-0022, HW-16-0013, HW-11-0031]
status: verified-with-fixes
---

## When to use

Load this card when a screen has to **draw a chart that no built-in component
gives you** - a bar chart with rounded capsule bars, a right-hand y-axis,
dashed gridlines with headroom above the tallest bar - and the data behind it
changes as the user pages through periods. The pattern is a two-layer
component: a renderer that knows only numbers and a style description, and a
domain wrapper that turns reading minutes into those numbers.

The transferable part is the redraw trigger. `Canvas` is imperative: nothing
repaints because a state variable changed. `onReady` fires once, and every
later repaint has to be raised by hand. This sample wires `@Prop @Watch` on
the data array, so a period switch upstream travels down as a normal state
assignment and lands on a `drawChart()` call. The same wiring covers any
imperative surface hung under declarative state - `Canvas`, `XComponent`, a
drawing `Node`.

It generalises past reading statistics to any "week / month / year / all"
board: step counts, listening minutes, spend by category. What changes is the
aggregation layer; the chart underneath is domain-free.

## Feature checklist

- Four period buttons (周/月/年/总 - week, month, year, all); the active one is
  filled blue and the rest grey.
- Left and right chevrons page the current period; the title above reads
  `4月13日-4月19日`, `2025年4月` or `2025年` depending on the view.
- The "all" view hides the pager and shows a static caption instead.
- A summary card shows accumulated hours and minutes plus a daily average.
- The histogram redraws when the period or the page changes.
- Bars are capsules (semicircular caps), labelled with their value on top.
- Gridlines are dashed, evenly spaced, with one extra line above the tallest
  bar so the peak never touches the top edge.
- Week and month views count in minutes; year and all views count in hours.
- A period with no reading renders a flat chart, not a chart of `NaN`
  (`HW-11-0022`).

## Architecture

One `entry` module. Three layers - page, domain wrapper, renderer - plus a
model folder that does all the date arithmetic.

```
entry/src/main/ets
├── component/Chart.ets              BarChart: summary header, minute/hour unit choice, builds HistogramOptions
├── component/HistogramChart.ets     the renderer: option types, HistogramConfig, drawChart + four draw passes
├── entryability/EntryAbility.ets    full-screen window, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── model/DailyReadingData.ets       axis label arrays, BUTTONS, DailyReadingData, ReadingDataStatic, getReadingDataStatic
├── model/MockData.ets               six years of stub data, cryptoFramework RNG
├── model/PersonalReadingStats.ets   the aggregation: getWeeklyData / getMonthlyData / getYearlyData / getAllYearsData
└── pages/ReadingStatsPage.ets       @Entry: title bar, period buttons, pager, the chart card
```

The documented tree matches the zip exactly, file for file. The document's
**step-1 snippet does not parse**, though: it opens a `Row` and an `onClick`
and jumps into the 统计卡片 `Column` without closing either - one instance of
the corpus-wide abridgement defect `HW-16-0013`. Copy from the zip, not from
the page.

**The design decision worth copying** is that `HistogramChart` never learns
what it is drawing. Its inputs are `curDataHour: Array<number>` and an
`options: HistogramOptions` object that describes bar width, colours, fonts,
axis behaviour, gridline dash pattern and the drawing-area insets. All of the
reading domain - that a week is counted in minutes and a year in hours, that
the summary line says `3小时 20分` - lives one level up in `BarChart`. The
split means the renderer is liftable into any project unchanged, and it is why
the option interface is worth the ~90 lines it costs.

The second structural choice is that aggregation happens **once, at the page,
into a flat value object**. `getReadingDataStatic(Array<DailyReadingData>)`
walks the daily records once and returns a `ReadingDataStatic` carrying
`dailyMinutes`, `totalMinutes`, `totalBookCount`, `avgMinutes`, `maxMinutes`
and `days`. The chart never sees a `Date`. Every paging button assigns a fresh
`ReadingDataStatic` into `@State`, which is what makes the whole redraw chain a
single reference change.

## Implementation steps

1. **Model the period data as one value object** (`ReadingDataStatic`) and fill
   it in a single pass over the daily records. Reassign the whole object on a
   period change - never mutate it in place, or the `@Watch` will not fire.
2. **Hold one state object per view** (`curWeekData`, `curMonthData`,
   `curYearData`, `curAllData`) and switch between four `BarChart` instances
   in an `if / else if` chain, so each view keeps its own paging offset.
3. **Translate domain to chart in the wrapper**: divide minutes by a
   `threshold` (1 for week/month, 60 for year/all) and pick the matching unit
   resource in `aboutToAppear`, before the first `createOption()`.
4. **Declare `@Prop @Watch('onDataChange')` on the number array** in the
   renderer. `@Prop` gives it a one-way copy; `@Watch` is the only thing that
   turns a data change into a repaint.
5. **Paint from two places**: `onReady` for the first frame (the canvas has no
   size before it fires) and `onDataChange` for every later one. Re-apply the
   options before drawing, since the wrapper hands down a new options object
   with the new unit.
6. **Guard the y-scale against a zero maximum** before any bar geometry is
   computed (`HW-11-0022`): an empty week makes `MAX_DATA_VALUE` zero, the
   division `Infinity`, and every bar coordinate `NaN`.
7. **Let the gridline pass own the final scale.** It computes a display
   maximum with one interval of headroom and returns the scale it used; the
   axis and bar passes must use that returned value, not the raw one, or the
   bars will not line up with the lines behind them.
8. **Reset the context per frame** with `CTX.reset()` and `clearRect`, and
   wrap each pass in `save()` / `restore()` so a dash pattern or a text
   alignment set by one pass does not leak into the next.

## Verified snippets

All snippets are from `TimeStatistics.zip`. Corrected forms are marked.

**The redraw trigger - `entry/src/main/ets/component/HistogramChart.ets`** (as shipped)

```typescript
@Component
export struct HistogramChart {
  private settings = new RenderingContextSettings(true);
  private canvas: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings);
  @Prop xAxisData: Array<string>;
  @Prop @Watch('onDataChange') curDataHour: Array<number>;   // the only repaint trigger
  @Prop options: HistogramOptions;
  private context = this.getUIContext().getHostContext() as common.UIAbilityContext;
  @State config: HistogramConfig = new HistogramConfig();

  aboutToAppear() {
    this.applyOptions();          // config only - the canvas has no size yet
  }

  onDataChange() {
    this.applyOptions();          // pick up the new unit and labels
    this.drawChart();
  }

  build() {
    Column() {
      Stack() {
        Canvas(this.canvas)
          .backgroundColor('#ffffff')
          .onReady(() => {
            this.drawChart();     // first paint: onReady is the earliest safe moment
          });
      };
    };
  }
}
```

**Two entry points, and both are necessary.** `onReady` is the first moment
`CTX.width` and `CTX.height` are meaningful, so it owns the initial frame;
`aboutToAppear` deliberately only prepares `config`. After that the canvas is
inert - the ArkUI diff never repaints it - so `@Watch` on `curDataHour`
carries every subsequent redraw. `applyOptions()` runs on both paths because
the wrapper rebuilds `HistogramOptions` alongside the data (the unit resource
changes between the minute views and the hour views).

`@Prop` rather than `@Link` is right here: the renderer copies the array and
never writes back. Note that `xAxisData` is declared but never passed by
`BarChart` - the labels the chart actually draws arrive inside
`options.labels`. Drop the unused `@Prop` when lifting this component.

**The scale, and the empty-period guard - same file** (corrected, see `HW-11-0022`)

```typescript
private drawChart() {
  const CTX = this.canvas;
  if (CTX === undefined) {
    return;
  }
  CTX.reset();
  CTX.save();

  const WIDTH = CTX.width;
  const HEIGHT = CTX.height;
  const DRAWING_AREA = this.config.drawingArea;

  CTX.clearRect(0, 0, WIDTH, HEIGHT);

  const CHART_AREA: CHART_AREA = {
    x: DRAWING_AREA.padding.left + DRAWING_AREA.margin.left,
    y: DRAWING_AREA.padding.top + DRAWING_AREA.margin.top,
    width: WIDTH - DRAWING_AREA.padding.left - DRAWING_AREA.padding.right -
    DRAWING_AREA.margin.left - DRAWING_AREA.margin.right,
    height: HEIGHT - DRAWING_AREA.padding.top - DRAWING_AREA.padding.bottom -
    DRAWING_AREA.margin.top - DRAWING_AREA.margin.bottom,
  };

  const MAX_DATA_VALUE = Math.max(...this.config.data, 0);
  const MAX_ALLOWED_HEIGHT = CHART_AREA.height * DRAWING_AREA.maxBarHeightRatio;
  // FIX: shipped code divides unconditionally -> Infinity, then 0 * Infinity = NaN
  let yScale = MAX_DATA_VALUE > 0 ? MAX_ALLOWED_HEIGHT / MAX_DATA_VALUE : 0;

  if (this.config.gridLineStyle.show) {
    yScale = this.drawGridLines(CTX, CHART_AREA.x, CHART_AREA.y, CHART_AREA.width, CHART_AREA.height,
      MAX_DATA_VALUE);                    // the gridline pass returns the scale everyone else uses
  }

  this.drawYAxis(CTX, CHART_AREA.x, CHART_AREA.y, CHART_AREA.width, CHART_AREA.height, MAX_DATA_VALUE, yScale);
  this.drawXAxis(CTX, CHART_AREA.x, CHART_AREA.y, CHART_AREA.width, CHART_AREA.height);
  this.drawBars(CTX, CHART_AREA.x, CHART_AREA.y, CHART_AREA.width, CHART_AREA.height, yScale);
}
```

**Three lines carry the geometry.** `CHART_AREA` subtracts both margin and
padding on each side; the sample uses margin for the space the axis labels
need outside the plot and padding for breathing room inside, which is why both
appear in the same subtraction. `Math.max(...data, 0)` seeds the spread with
`0` so an empty array does not produce `-Infinity` - but it is exactly that
seed that makes `MAX_DATA_VALUE === 0` reachable, and the shipped division
then yields `Infinity`. `drawBars` computes `BAR_VALUE * yScale`, so `0 *
Infinity` is `NaN`, and a first-run user with no reading history gets a chart
of nothing rather than a flat baseline.

Note the reassignment of `yScale` from `drawGridLines`. The raw scale would
put the tallest bar flush against the top edge; the gridline pass recomputes
against `maxValue + oneInterval` and hands back that smaller scale, so the
peak bar always sits one gridline below the ceiling.

**The gridline pass that owns the scale - same file** (as shipped)

```typescript
private drawGridLines(ctx: CanvasRenderingContext2D, chartX: number, chartY: number,
  chartWidth: number, chartHeight: number, maxValue: number) {
  const GRID_STYLE = this.config.gridLineStyle;
  ctx.save();

  ctx.strokeStyle = GRID_STYLE.strokeStyle;
  ctx.lineWidth = GRID_STYLE.lineWidth;
  if (GRID_STYLE.lineDash.length > 0) {
    ctx.setLineDash(GRID_STYLE.lineDash);          // [2, 2]
  }

  const GRID_LINE_COUNT = GRID_STYLE.GRID_LINE_COUNT;
  const VALUE_INTERVAL = maxValue / GRID_LINE_COUNT;
  const DISPLAY_MAX_VALUE = maxValue + VALUE_INTERVAL;   // one interval of headroom
  const DISPLAY_Y_SCALE = chartHeight / DISPLAY_MAX_VALUE;

  for (let i = 0; i <= GRID_LINE_COUNT; i++) {
    const VALUE = i * VALUE_INTERVAL;
    const Y = chartY + chartHeight - (VALUE * DISPLAY_Y_SCALE);
    if (Y >= chartY && Y <= chartY + chartHeight) {
      ctx.beginPath();
      ctx.moveTo(chartX, Y);
      ctx.lineTo(chartX + chartWidth + 36, Y);      // overshoot so lines run under the right-hand labels
      ctx.stroke();
    }
  }

  ctx.restore();
  return DISPLAY_Y_SCALE;
}
```

**`save()` / `restore()` around a pass that sets a dash pattern is not
optional.** `setLineDash` is context state, not per-path state: without the
restore, the axis lines and the bar outlines drawn afterwards would come out
dashed too. The same bracket appears around every other pass in the file for
the same reason - each one sets `textAlign`, `textBaseline`, `fillStyle` and
`font`.

The `+ 36` on the line end is deliberate. The y-axis is configured
`position: 'right'`, so its labels sit outside the plot on the right; running
the gridlines past the plot edge puts the line under its own label instead of
stopping short of it.

**Domain to chart, in the wrapper - `entry/src/main/ets/component/Chart.ets`** (as shipped)

```typescript
@Component
export struct BarChart {
  @Prop label: string;                                   // 'week' | 'month' | 'year' | 'all'
  @Prop xAxisData: Array<string>;
  @Prop @Watch('onDataChange') curData: ReadingDataStatic;
  @State curDataHour: Array<number> = [];
  @State defOption: HistogramOptions = this.createOption();
  threshold: number = 1;
  unit: Resource = $r('app.string.minute');

  aboutToAppear() {
    if (this.label === 'year' || this.label === 'all') {
      this.threshold = 60;                               // 1 hour = 60 minutes
      this.unit = $r('app.string.hour');
    }
    this.onDataChange();                                 // build options with the right unit before first paint
  }

  onDataChange() {
    this.curDataHour = this.curData.dailyMinutes.map(item => Math.floor(item / this.threshold));
    this.defOption = this.createOption();
  }
}
```

*(the shipped `aboutToAppear` spells the same decision out as a five-branch
`if/else if` over `'day' | 'week' | 'month' | 'year' | 'all'`; the two hour
branches are the only ones that differ from the defaults.)*

**The `@Watch` chain is three links long and each one is a plain assignment.**
The page reassigns `curWeekData`; `BarChart.onDataChange` recomputes
`curDataHour` and rebuilds `defOption`; `HistogramChart.onDataChange` re-applies
the config and repaints. Nothing subscribes to anything - each hop is
`@Prop @Watch` on the value the level above just replaced. That is why the
aggregation must produce a *new* `ReadingDataStatic` rather than mutating the
old one.

`Math.floor(item / threshold)` also explains a rough edge worth knowing about:
in the year and all views a day under an hour floors to `0`, so a year of light
reading can legitimately produce an all-zero array - which is the exact input
`HW-11-0022` mishandles.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

`deviceTypes` is `["phone"]` only - narrower than the rest of this industry's
samples, which usually list `phone`, `tablet` and `2in1`. The chart geometry is
consistent with that: `Chart.ets` fixes the histogram host at `368 x 228` vp
and the font sizes inside `HistogramOptions` (28-40) are canvas pixels tuned
for a phone-density surface.

The y-axis unit is resolved imperatively, because canvas text takes a string,
not a `Resource`:

```typescript
ctx.fillText(VALUE.toString() + this.context.resourceManager.getStringSync(Y_AXIS.unit.id), LABEL_X, Y);
```

That is the standard escape hatch for drawing localised text on a canvas -
`getStringSync` off the ability context's `resourceManager`, with the
`Resource`'s `.id`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` in the zip is
  `6.0.0(20)`, matching the document.
- The data is entirely synthetic. `MockData.ets` fills 2020-2025 with random
  daily minutes (20-100) and book counts (1-4) drawn from
  `cryptoFramework.createRandom()`. There is no persistence layer:
  `PersonalReadingStats.loadData()` is an empty method with a comment where a
  database or network call would go.
- The mock generator's month lengths are approximate - the day loop runs
  `0..maxDay` with `maxDay` of 30/29/28/27, and `new Date(year, month, 0)` is
  the last day of the *previous* month, so each month's first record lands in
  the month before it. Harmless for a demo, wrong as a template for real
  aggregation.
- The month view buckets by count, not by date: `getMonthlyData` accumulates
  records and flushes a bucket every fifth one, which yields the seven buckets
  that `MONTH_AXIS` (`1, 5, 10, 15, 20, 25, 30`) labels - but the labels are
  fixed strings, so a short month still reads "30".
- `getStartOfWeek` builds the week from the first weekday of the month, and the
  page derives the week number with `Math.ceil((date + firstDayOfWeek) / 7)`.
  Paging by `weekOffset * 7` days therefore drifts across month boundaries.
- The back arrow in the title bar has no `onClick`, and the list below the
  chart is a single static image (`ic_list`), not a component. This is a chart
  sample, not a screen.
- `HistogramChart` declares `@Prop xAxisData` that no caller passes; the x
  labels travel through `options.labels`.

## Pitfalls

- **`HW-11-0022`** (B/low, confirmed): the histogram scale divides by zero when
  every reading in the period is `0` - `MAX_DATA_VALUE` is `0`, `yScale`
  becomes `Infinity`, and `drawBars` produces `NaN` coordinates for every bar.
  A brand-new user, or any week with no reading, hits it on first paint. Fix:
  `let yScale = MAX_DATA_VALUE > 0 ? MAX_ALLOWED_HEIGHT / MAX_DATA_VALUE : 0;`
  The document reproduces the same unguarded division at line 105.
- **`HW-16-0013`** (E/medium, confirmed): systematic - the document's abridged
  snippets are cut mid-construct. This page's step-1 snippet (line 24) opens a
  `Row`, an `if` and an `onClick` and never closes them before jumping to the
  统计卡片 `Column`, so it does not parse. The zip source is valid; the
  abridgement pipeline is what is broken, and the same defect recurs across
  `16_shopping`, `13_media_entertainment`, `18_photography` and others. Fix:
  regenerate excerpts with brace-balanced elision.

## References

- `documentation/harmonyos-references/02_application-framework/ts-canvasrenderingcontext2d.md` - `clearRect`, `setLineDash`, `arc`, `fillText`, `save`/`restore`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-canvasrenderingcontext2d
- `documentation/harmonyos-guides/03_application-framework/arkts-watch.md` - `@Watch` and when the callback fires
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-watch
- `documentation/harmonyos-guides/03_application-framework/arkts-prop.md` - `@Prop` one-way copy semantics
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-prop
- `huawei_industry_tree/11_news_reading/docs/21_time_statistics.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/time_statistics-0000002362798665
