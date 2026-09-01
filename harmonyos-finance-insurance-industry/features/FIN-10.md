---
id: FIN-10
title: Expenditure distribution pie chart with mpchart
industry: 07_finance_insurance
doc: huawei_industry_tree/07_finance_insurance/docs/10_expenditure_pie_chart.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/expenditure_pie_chart-0000002483692785
sample: huawei_industry_tree/07_finance_insurance/downloads/ExpenditurePieChart.zip
kits: ["@kit.ArkUI"]
apis: [PieChart, PieChartModel, PieData, PieDataSet, PieEntry, ValuePosition, ChartColor, JArrayList, setValueFormatter, setHoleRadius, setMinAngleForSlices, Progress, Tabs, List, ListItem]
permissions: []
min_api: 20
modules: [entry]
dependencies: ["@ohos/mpchart"]
findings: [HW-07-0036, HW-07-0037, HW-07-0038, HW-07-0039, HW-07-0050, HW-07-0051, HW-07-0052, HW-07-0053]
status: verified-with-fixes
---

## When to use

Load this card for a **donut or pie chart with outside labels and a legend list
beneath it** - spending breakdowns, portfolio allocation, storage usage, poll
results. It is the same `@ohos/mpchart` package that `MAT-04` uses for growth
curves, so read both before choosing a charting approach for a project.

The chart setup here is sound and reusable; the data plumbing around it is the
part to rewrite (four independent copies of the same numbers - see the pitfalls).

## Feature checklist

- A donut chart of expenditure by category, holed out in the middle.
- Each slice labelled outside the ring with its category and percentage.
- No legend and no description drawn by the chart itself.
- A composition list beneath the chart: icon, label, bar, amount.
- Rotation disabled so the chart cannot be dragged out of alignment.

## Architecture

Two files do the work.

```
entry/src/main/ets/
├── components/PieChartComponent.ets   the chart: model, data, colours, formatter
├── constants/CommonConstants.ets      sizes, margins, font sizes
└── pages/MainPage.ets                 @Entry - tab bar, header, chart, composition list
```

`PieChartComponent` builds a `PieChartModel` in `aboutToAppear` and hands it to
the `PieChart` component. **Configuration is imperative and happens once**; the
`build()` is a single line. That split - model assembled in the lifecycle hook,
component rendered from it - is the mpchart idiom worth internalising.

## Implementation steps

1. **Add `@ohos/mpchart` to `oh-package.json5`**, pinned to an exact version
   (`HW-07-0039`).
2. **Compute the total once** from the values, and pass both down (`HW-07-0037`,
   `HW-07-0050`).
3. **Build a `JArrayList<PieEntry>`** - value plus label per slice.
4. **Configure the `PieDataSet`**: colours, label placement, leader lines, text.
5. **Set a value formatter** to render `label percentage%` rather than the raw
   number.
6. **Configure the model**: hole radius, no legend, no description, extra
   offsets for the outside labels, rotation off.
7. **Render the legend list separately** with the same denominator the chart
   uses (`HW-07-0038`).

## Verified snippets

From `ExpenditurePieChart.zip`. Corrected forms are marked.

**Model configuration — `components/PieChartComponent.ets`** (corrected, see `HW-07-0036`)

```typescript
private model: PieChartModel = new PieChartModel();

aboutToAppear(): void {
  let pieData: PieData = this.getPieData();
  this.model.setData(pieData);
  this.model.setHoleRadius(80);                  // % of radius - 80 makes it a thin ring
  this.model.getDescription()?.setEnabled(false);
  this.model.getLegend()?.setEnabled(false);     // the list below is the legend
  this.model.setExtraOffsets(50, 0, 50, 0);      // room for the outside labels
  // FIX: shipped code calls this.model.setMinAngleForSlices(10). On a chart whose
  // stated purpose is 比重, a floor of 10 degrees inflates every category below
  // 2.78% of the total and compresses all the others to make room. With the six
  // bundled values the smallest slice is 20.9 degrees, so the distortion is
  // invisible during development and appears only on real data.
  this.model.setRotationEnabled(false);
  this.model.setDrawEntryLabels(false);          // labels come from the formatter instead
}

build() {
  Column() {
    PieChart({ model: this.model })
      .width(CommonConstants.FULL_WIDTH)
      .height(CommonConstants.FULL_HEIGHT)
      .backgroundColor(Color.White);
  };
}
```

**Data set and formatter — same file** (corrected, see `HW-07-0050`)

```typescript
private getPieData(): PieData {
  let entries: JArrayList<PieEntry> = new JArrayList<PieEntry>();
  for (let i = 0; i < this.labels.length; i++) {
    entries.add(new PieEntry(this.values[i], this.labels[i]));
  }

  let dataSet: PieDataSet = new PieDataSet(entries, 'Pie Chart');
  dataSet.setSliceSpace(1);                              // hairline gap between slices
  dataSet.setSelectionShift(5);                          // how far a tapped slice pops out

  let colors: JArrayList<number> = new JArrayList();
  colors.add(ChartColor.argb(255, 32, 112, 243));        // one opaque lead colour ...
  colors.add(ChartColor.argb(204, 10, 89, 247));         // ... then one hue at falling alpha
  colors.add(ChartColor.argb(178, 10, 89, 247));
  colors.add(ChartColor.argb(153, 10, 89, 247));
  colors.add(ChartColor.argb(127, 10, 89, 247));
  colors.add(ChartColor.argb(102, 10, 89, 247));
  dataSet.setColorsByList(colors);

  dataSet.setDrawValues(true);
  dataSet.setXValuePosition(ValuePosition.OUTSIDE_SLICE);
  dataSet.setYValuePosition(ValuePosition.OUTSIDE_SLICE);
  dataSet.setValueLinePart1Length(0.3);                  // leader line, inner segment
  dataSet.setValueLinePart2Length(0.2);                  // leader line, outer segment
  dataSet.setValueLineColor(Color.Gray);
  dataSet.setValueLineWidth(1);
  dataSet.setValueTextColor(Color.Black);
  dataSet.setValueTextSize(12);

  dataSet.setValueFormatter({
    getFormattedValue: (value: number, entry: PieEntry) => {
      // FIX: shipped code divides by `this.totalValue`, a hardcoded 3174.3 that
      // happens to equal the sum of the bundled figures. Derive it.
      const total = this.values.reduce((a: number, b: number) => a + b, 0);
      const percentage = (value / total) * 100;
      return `${entry.getLabel()} ${percentage.toFixed(1)}%`;
    }
  });
  return new PieData(dataSet);
}
```

**Two things here are the reason to read this sample.** First, the colour ramp:
one saturated lead colour for the largest category, then a single hue at
descending alpha for the rest - it reads as one family, needs no palette, and
extends to any number of slices. Second, `setDrawEntryLabels(false)` combined
with `setValueFormatter` and `OUTSIDE_SLICE`: the label and the percentage are
composed into one string by the formatter, so there is exactly one text run per
slice on the leader line instead of two competing ones.

**The composition list — `pages/MainPage.ets`** (corrected, see `HW-07-0038`, `HW-07-0051`)

```typescript
// FIX: shipped code is Progress({ value: value, total: this.values[0] }) - each bar
// is scaled against the largest category while the pie above is scaled against the
// sum, so the same row reads 52.5% in the bar and 20.7% in the slice. Index 0 also
// assumes the array is sorted descending.
Progress({ value: value, total: this.total })
  .width(CommonConstants.PROGRESS_WIDTH)
  .layoutWeight(1)
  .backgroundColor($r('app.color.start_window_background'));

// FIX: shipped code puts the row *and* a conditional Divider inside one ListItem,
// then positions the Divider with a 50vp top margin that duplicates the row height.
// ListItem "can contain a single child component"; List draws separators itself.
List() {
  ForEach(this.labels, (item: string, index: number) => {
    ListItem() {
      this.compositionListItem(this.icons[index], this.labels[index],
        this.values[index], '￥' + this.values[index]);
    };
  }, (item: string) => item);
}
.divider({ strokeWidth: CommonConstants.STROKE_WIDTH, startMargin: CommonConstants.MARGIN_FORTY })
.scrollBar(BarState.Off);
```

**Owning the data in one place** (corrected, see `HW-07-0037`, `HW-07-0050`)

```typescript
// FIX: as shipped the same six figures are declared in MainPage.ets AND in
// PieChartComponent.ets, their sum is hardcoded a third time as
// `totalValue: number = 3174.3`, and a fourth time as a string resource
// { "name": "amount", "value": "3174.30" } rendered as the 总支出 headline.
// Nothing recomputes any of them.
@State values: number[] = [1254, 658.3, 551, 320, 207, 184];
get total(): number {
  return this.values.reduce((a: number, b: number) => a + b, 0);
}

PieChartComponent({ values: this.values, labels: this.labels, totalValue: this.total })
  .height(CommonConstants.CHART_HEIGHT);
Text(this.total.toFixed(2))            // not $r('app.string.amount')
```

## Permissions & config

**None.**

`entry/oh-package.json5` (corrected, see `HW-07-0039`):

```json5
"dependencies": {
  "@ohos/mpchart": "3.0.23"     // FIX: shipped as "^3.0.23" - a floating range
                                // beside an exactly pinned SDK
}
```

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Requires the ohpm package `@ohos/mpchart` - **not listed in the document's
  约束与限制** (`HW-07-0039`).
- `setExtraOffsets` must leave room for the outside labels or they are clipped;
  50vp left and right suits six categories at 12fp.
- Colour count should match slice count - mpchart cycles the list otherwise.
- `setMinAngleForSlices` trades proportional accuracy for legibility. On a
  proportion chart, do not.
- The tab bar on this page cannot switch (`HW-07-0052`); only the 统计 tab is
  implemented.

## Pitfalls

- **`HW-07-0036` — `setMinAngleForSlices(10)` inflates any category below 2.78%
  of the total** and shrinks the rest, on a chart introduced as showing 比重.
- **`HW-07-0037` — the values array is declared twice,** once in the page and
  once in the chart component, each rendering from its own copy.
- **`HW-07-0050` — the total is hardcoded twice more,** as `totalValue = 3174.3`
  (the divisor for every percentage label) and as the string resource `amount`.
- **`HW-07-0038` — the bars use `total: this.values[0]`,** so a row reads 52.5%
  in the bar and 20.7% in the pie beside it.
- **`HW-07-0051` — two children per `ListItem`,** with the divider placed by a
  50vp margin that silently duplicates the row height.
- **`HW-07-0039` — `"^3.0.23"` floating range,** and the dependency is missing
  from the constraints. `MAT-04` pins the same package at `^3.0.11`.
- **`HW-07-0052` — the tab bar is inert:** `onContentWillChange` always returns
  `false` and `selectedIndex` is `@State` but never assigned.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-progress.md` - `Progress`, `value`, `total`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-progress
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `divider`, `startMargin`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-container-listitem.md` - the single-child rule
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-listitem
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` - `Tabs`, `onContentWillChange`, `barMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - `ForEach` keys
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `MAT-04` - growth curves with the same `@ohos/mpchart` package (line chart side)
- `FIN-03` - the K-line/intraday chart, the other charting practice in this industry
- `FIN-04` - the income/expense calendar, the other spending-visualisation page
