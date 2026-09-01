---
id: FIN-03
title: Stock charts - intraday line and daily K-line on Canvas
industry: 07_finance_insurance
doc: huawei_industry_tree/07_finance_insurance/docs/03_stock_chart_x.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/stock_chart_x-0000002264336070
sample: huawei_industry_tree/07_finance_insurance/downloads/StockChartX.zip
kits: ["@kit.ArkUI", "@kit.ArkTS", "@kit.LocalizationKit"]
apis: [Canvas, CanvasRenderingContext2D, RenderingContextSettings, clearRect, resourceManager.getRawFileContent, resourceManager.getStringSync, buffer.from, "@Watch", "@Prop", setInterval, clearInterval, PinchGesture, PanGesture]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-07-0012, HW-07-0013, HW-07-0014, HW-07-0053]
status: verified-with-fixes
---

## When to use

Load this card for **financial time-series charts drawn by hand**: intraday
price lines with a volume histogram, candlestick charts, and any chart that must
support zooming, panning and live append. It also applies outside finance to any
scrolling windowed chart over a long series.

**This is the best-built sample in the corpus so far.** Three findings across
2,133 lines, no timer leak, no per-frame allocation, no hardcoded strings, a
full locale set. Where the other Canvas practice in this corpus - the automotive
dashboard - gets almost everything wrong, this one gets almost everything right.
Read it as the reference for Canvas work.

## Feature checklist

- Two chart modes: intraday time line and daily K-line, switchable.
- Intraday: price polyline plus a volume histogram beneath it.
- Daily: candlesticks plus a volume histogram.
- Live append - new points arrive on a timer and the chart follows, but only if
  the user is looking at the latest data.
- Daily chart supports zoom in/out and pan left/right through history.
- Mock data loaded from `rawfile`, ready to be swapped for a network feed.

## Architecture

Single `entry` module, cleanly split between page and chart components.

```
entry/src/main/ets
├── component/TimeLineChart.ets      intraday line + volume
├── component/DailyKLineChart.ets    candlesticks + volume, zoom and pan
├── constants/StockChartPageConstants.ets
├── constants/TimeLineChartConstants.ets
├── constants/DailyKLineChartConstants.ets
├── model/StockDataModel.ets         TimeLineData, KLineData
└── pages/StockChartPage.ets         data loading, the fetch timer, mode switch
entry/src/main/resources/rawfile/
├── MockTimeLineData.json
└── MockDailyKLineData.json
```

The page owns **all** the data and the timer; the charts are pure
presentation, taking their series through `@Prop @Watch(...)`. Each chart
carries its own constants file, so nothing about candle width or grid spacing
leaks into the page.

**The windowing model** is what makes the daily chart work: the page holds the
full series, and the chart keeps `leftOffset` and `displayCount` to describe the
visible window into it. Zooming changes `displayCount`; panning changes
`leftOffset`; `isViewingLatest` records whether the user has scrolled away from
the right edge, so live appends only scroll the view when they have not.

## Implementation steps

1. **Load the series from `rawfile`** with `getRawFileContent`, convert the
   buffer to a string with `buffer.from(...).toString()`, then `JSON.parse`.
2. **Guard the parse** with `?.` and a default, as the sample does - and log the
   failure (`HW-07-0013`).
3. **Take the canvas size in `onReady`** from `this.context.width` / `.height`,
   once, and store it.
4. **Refuse to draw until you have a size and data** - the sample's guard is the
   pattern to copy.
5. **Redraw from a `@Watch` on the series**, clearing with `clearRect` first.
6. **Keep the visible window as `leftOffset` + `displayCount`** and derive the
   drawn slice from it.
7. **Make the timer idempotent and release it** - see below.

## Verified snippets

All snippets are from `StockChartX.zip`. Corrected forms are marked.

**Timer lifecycle — `pages/StockChartPage.ets`** (as shipped)

```typescript
private dataUpdateTimer: number | null = null;
private dataFetchInterval: number = StockChartPageConstants.DATA_FETCH_INTERVAL;  // 3000

aboutToAppear() {
  this.loadInitialData();
  this.startDataFetchTimer();
}

aboutToDisappear() {
  this.stopDataFetchTimer();
}

private startDataFetchTimer() {
  this.stopDataFetchTimer();          // idempotent: never two timers
  this.dataUpdateTimer = setInterval(() => {
    this.fetchNewData();
  }, this.dataFetchInterval);
}

private stopDataFetchTimer() {
  if (this.dataUpdateTimer !== null) {
    clearInterval(this.dataUpdateTimer);
    this.dataUpdateTimer = null;      // clears the handle, so stop is idempotent too
  }
}
```

**Copy this verbatim.** It is the correct timer pattern and the only complete
one in this corpus: the handle is typed `number | null`, `start` stops first so
it can never leak a second interval, `stop` nulls the handle so calling it twice
is safe, and `aboutToDisappear` releases it. Compare `AUTO-06`, which captures
no handle at all and runs forever.

**Loading a series from rawfile — same file** (corrected, see `HW-07-0013`)

```typescript
import { buffer } from '@kit.ArkTS';

private async loadInitialData(): Promise<void> {
  try {
    const timeLineValue = await this.uiContext.resourceManager.getRawFileContent('MockTimeLineData.json');
    const timeLineStr = buffer.from(timeLineValue.buffer).toString();
    this.allMockTimeLineData = JSON.parse(timeLineStr)?.timeLineData || [];

    const assumeCurrentTimeLineIndex: number = this.findCurrentTimeIndex(this.currentTime);
    if (assumeCurrentTimeLineIndex >= 0) {
      this.currentTimeLineIndex = assumeCurrentTimeLineIndex;
      this.currentTimeLineData = this.allMockTimeLineData.slice(0, assumeCurrentTimeLineIndex + 1);
    }

    const dailyKLineValue = await this.uiContext.resourceManager.getRawFileContent('MockDailyKLineData.json');
    const dailyKLineStr = buffer.from(dailyKLineValue.buffer).toString();
    this.allMockDailyKLineData = JSON.parse(dailyKLineStr)?.dailyKLineData || [];

    const assumeDailyKLineIndex: number = this.findCurrentDateIndex(this.currentDate);
    if (assumeDailyKLineIndex !== -1) {        // FIX: shipped code tests assumeCurrentTimeLineIndex
      this.currentDailyKLineIndex = assumeDailyKLineIndex;
      this.currentDailyKLineData = this.allMockDailyKLineData.slice(0, assumeDailyKLineIndex + 1);
    } else {
      this.currentDailyKLineData = [...this.allMockDailyKLineData];
    }

    this.updateRealtimeData();
  } catch (err) {
    Logger.error(`loadInitialData failed: ${JSON.stringify(err)}`);   // FIX: shipped catch is empty
  }
}
```

`getRawFileContent` returns a `Uint8Array`, not a string - `buffer.from(x.buffer).toString()`
from `@kit.ArkTS` is the conversion. The `?.field || []` on the parse result is
worth keeping: it survives a JSON payload whose shape changed.

**Simulating a live feed — same file** (as shipped)

```typescript
private fetchNewData() {
  if (this.chartType === StockChartPageConstants.STR_TIME &&
    this.currentTimeLineData.length < this.allMockTimeLineData.length) {
    this.currentTimeLineIndex++;
    this.currentTimeLineData.push(this.allMockTimeLineData[this.currentTimeLineIndex]);
  } else if (this.chartType === StockChartPageConstants.STR_DAY &&
    this.currentDailyKLineData.length < this.allMockDailyKLineData.length) {
    this.currentDailyKLineIndex++;
    this.currentDailyKLineData.push(this.allMockDailyKLineData[this.currentDailyKLineIndex]);
  }
}
```

Replace the body with your network call; the surrounding timer and the
`@Prop @Watch` plumbing stay the same.

**Following live data without fighting the user — `component/DailyKLineChart.ets`** (as shipped)

```typescript
@Prop @Watch('onDataChange') data: KLineData[];
@Prop displayCount: number;

private onDataChange() {
  const hasNewData = this.data.length > this.previousDataLength;

  // only follow the right edge if the user has not scrolled away from it
  if (this.isViewingLatest && hasNewData) {
    this.leftOffset = Math.max(0, this.data.length - this.displayCount);
  }

  this.previousDataLength = this.data.length;
  this.updateDisplayData(this.isViewingLatest, false);
}
```

**This is the detail that makes a live chart usable.** Tracking
`previousDataLength` distinguishes an append from a zoom or a pan, and
`isViewingLatest` means a user reading history is not yanked back to the right
edge every three seconds. Note also that the handler writes `leftOffset` and
`previousDataLength`, neither of which is watched - so unlike `AUTO-06`'s
dashboard it does not re-enter itself.

**The redraw guard — `component/TimeLineChart.ets`** (as shipped)

```typescript
@Prop @Watch('redrawCanvas') data: TimeLineData[];

private redrawCanvas() {
  if (!this.context || this.canvasWidth === 0 || this.canvasHeight === 0) {
    return;
  }
  this.calculatePriceRange();
  const ctx = this.context;
  ctx.clearRect(0, 0, width, height);
  this.drawMainChart(ctx);
  this.drawVolumeChart(ctx);
}
```

The daily chart adds `|| this.displayData.length === 0` to the same guard. Three
conditions - context ready, size known, data present - checked before any
drawing happens. The size itself is read once in `onReady`
(`this.canvasWidth = this.context.width`), not re-measured per frame.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

Resource directories: `base`, `dark`, `en_US`, `zh_CN`, `rawfile` - a full
locale set, and no hardcoded user-visible strings anywhere in the sample.

The "current" moment the charts open on comes from the `init_time` and
`init_date` string resources rather than from code, so it can be retargeted at
new mock data without touching the source.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Data is mock, loaded from `rawfile`. The document says so: real data needs a
  network request.
- The fetch interval is a fixed 3000 ms constant.
- The whole series lives in memory; the window is a slice. Fine for a demo
  series, but a multi-year daily history needs paging.
- Canvas drawing is manual - there is no chart library here, which is why the
  card is worth reading.

## Pitfalls

- **`HW-07-0012` — the daily K-line branch tests the intraday index.** If the
  configured date is missing from the daily data but the configured time is
  present, the candlestick chart opens empty; if the reverse, it shows candles
  from after the moment it is meant to represent. Both inputs are string
  resources, so this is exactly what a developer adapting the sample changes.
- **`HW-07-0013` — one empty catch covers both loads and both parses,** so a
  missing rawfile or malformed JSON leaves both charts blank with nothing logged
  and nothing shown.
- **`HW-07-0014` — the field named `uiContext` holds a `UIAbilityContext`** and
  is only ever used to reach `.resourceManager`.

## References

- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - the `Canvas` component
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/02_application-framework/ts-canvasrenderingcontext2d.md` - drawing, `clearRect`, paths and text
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-canvasrenderingcontext2d
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getRawFileContent`, `getStringSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `documentation/harmonyos-guides/03_application-framework/arkts-watch.md` - `@Watch`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-watch
- `AUTO-06` - the other hand-drawn Canvas practice; compare its timer and redraw handling with this one
