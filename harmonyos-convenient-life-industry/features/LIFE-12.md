---
id: LIFE-12
title: Perpetual calendar template - a layered 20-module calendar and almanac application, documented without a sample archive
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/12_perpetual_calendar.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/perpetual_calendar-0000002319250845
sample: none (the 代码下载 link is a Gitee source tree, not a zip - see HW-02-0082)
kits: ["@kit.ArkUI"]
apis: [Swiper, SwiperController, Grid, GridItem, columnsTemplate, maxCount, LazyForEach, cachedCount, indicator, loop, itemSpace, effectMode, EdgeEffect]
permissions: ["ohos.permission.INTERNET"]
min_api: 16
modules: [commons/common, commons/router_module, components/base_apis, components/base_calendar, components/base_almanac, components/base_events, components/date_calculation, components/festival_solar, components/login_info, components/today_history, components/traffic_restriction, components/vip_center, components/yiji_query, features/almanac, features/mine, features/perpetual, sdk/aggregated_payment, product/entry]
findings: [HW-02-0082, HW-02-0083, HW-02-0084, HW-02-0085, HW-02-0086, HW-02-0087, HW-02-0088]
status: verified-with-fixes
---

## When to use

Load this card for two distinct reasons.

**As an architecture reference:** it is the largest layered application in this
industry - eighteen modules across four layers - and the only one that
demonstrates the full commons / components / features / product split that
`LIFE-01` only sketches. If you are laying out an application big enough that
`LIFE-01`'s single-HAP-plus-HARs shape stops scaling, this is the next size up.

**As a month-grid recipe:** the `Swiper` of `Grid`s with a computed row count is
the standard way to build a scrollable calendar in ArkUI, and the date algorithm
in the document is complete enough to reimplement.

**What this card cannot do** is verify either against code. Unlike every other
scenario in this industry, this document ships no sample archive - its 代码下载
link points at a live Gitee branch (`HW-02-0082`). Everything below is read from
the document, and the snippets are quoted as published, with the defects that
are provable from the page itself marked.

## Feature checklist

The document describes three top-level modules:

- **万年历 (perpetual calendar)** - calendar service lookup, auspicious-day
  tools, date calculation, festival and solar-term lookup.
- **黄历 (almanac)** - per-date almanac content, with 今日宜 / 今日忌 (today's
  auspicious / inauspicious activities) and date switching.
- **我的 (mine)** - personal information, Huawei ID one-tap sign-in, theme
  switching.

Around them the tree describes further component modules that the prose does not
mention: traffic restriction, today-in-history, VIP centre, aggregated payment,
and a calendar widget (`EntryFormAbility` + `CalendarWidgetCard`).

The document is explicit that this is a template, not a product
(line 133): 本示例均提供的是模拟数据 ("this sample provides only simulated
data"), with mock pages, a simulated verification-code flow, and simulated login
when Huawei ID one-tap sign-in is not configured.

## Architecture

Four layers, named by directory rather than by `module.json5` type:

```
commons/                      公共能力层  - shared, depends on nothing above it
├── common                    components, constants, models, styles, CalculateDate, CalendarVM, WebView
└── router_module             RouterModule.ets

components/                   reusable UI + domain modules, one per capability
├── base_apis                 ActionSheet, Toast, SliderSwitch, BaseCell, BaseDatePicker,
│                             BaseLunarCard, BaseTextPicker, BaseTimerPicker, YiJiCell
├── base_calendar             CalendarDataSource, DateModel, HolidayModel, LunarModel,
│                             CalendarVM, HolidayVM, Https, DataMock, views/Calendar.ets
├── base_almanac              FiveElementsClashElements, TaboosAccordPengZu, CalendarAlmanac
├── base_events               schedules, todos, birthday reminders, EventDispatcher, Countdown
├── date_calculation          DateCalculation, DateInterval, YinYangConversion
├── festival_solar            PublicFestival, PublicHolidays, SolarTerms
├── login_info                LoginWithHuaweiID, BaseLogin, PersonInfo, privacy/terms pages
├── today_history             TodayHistoryCard, HistoryMore
├── traffic_restriction       LocationPermission.ets, TrafficRestriction view
├── vip_center                VipOpen, VipPackage, VipBenefits, OrderInfoUtil, SignUtils
└── yiji_query                JiDayDetail, JiDayDetailCard

features/                     基础特性层 - the three product tabs
├── almanac                   AlmanacView.ets            黄历入口
├── mine                      MinePage.ets               我的
└── perpetual                 PerpetualCalendar.ets      万年历组件入口

sdk/aggregated_payment        WXApiWrap, AggregatedPaymentPicker
product/entry                 EntryAbility, EntryFormAbility, Index, TabContainer, CalendarWidgetCard
```

**The layering rule this demonstrates.** `commons` is depended on by everything
and depends on nothing; `components` are self-contained capabilities that expose
one view each; `features` compose components into the three tabs; `product`
holds the ability, the tab container and the widget. That is one more layer than
`LIFE-01`'s three (product / basic-feature / common-capability), and the extra
one - `components` - is what lets a capability like `yiji_query` be developed and
tested without knowing which tab hosts it.

**Every component module is listed twice in the tree** - once for
`<module>/src/main/ets` and once for its root (`BuildProfile.ets`, `Index.ets`).
That pairing is the convention; three entries break it (`HW-02-0087`).

**The calendar's data flow** is the one part the document spells out:

```
CalendarVM.getDateList(param)     param = -1, 0, +1 (previous, current, next month)
  -> dayjs().add(param,'month')
  -> pad to the week start, compute rows, emit totalRows * 7 DateModel cells
  -> vm.dateListSource : IDataSource of DateModelList
Swiper > LazyForEach(dateListSource) > Grid(7 columns) > ForEach > GridItem
CalendarController.setSelectDate(date) -> vm.changeDate(date)   the public handle
```

## Implementation steps

Reconstructed from the document's four snippets. Corrections are marked.

1. **Compute the visible window from the month, not from a fixed 42 cells.**
   Take the first and last day of the target month, pad backwards to the
   configured week start, and derive the row count from the total span.
2. **Emit whole weeks.** `totalCells = totalRows * 7` so the `Grid` never has a
   ragged last row, and mark each cell with `isCurrentMonth` so the padding days
   can be greyed.
3. **Clamp the row count at both ends.** The document's algorithm has a floor of
   five and no ceiling; the arithmetic tops out at six, not the seven the prose
   claims (`HW-02-0083`).
4. **Attach the lunar data per cell** while you are already iterating, rather
   than in a second pass - the snippet calls `getLunarInfo(lunarDate)` twice per
   day, once for the label and once for the colour, which is worth collapsing
   into one call.
5. **Page the months with a `Swiper` over a `LazyForEach`,** one child per month,
   `loop(false)` so the ends are hard stops.
6. **Give the `LazyForEach` a real key** - a month identifier, not the default
   index (`HW-02-0084`). The window slides as the user pages, so the item at a
   given index changes month.
7. **Do not set `maxCount` on a `Grid` that has `columnsTemplate`** - it is inert
   in that mode (`HW-02-0083`).
8. **Expose the calendar as a controller object,** not as a component reference,
   so the hosting page can drive it without reaching into the view.
9. **Declare `dayjs` and the lunar library in `oh-package.json5`.** The
   document's central algorithm needs both and names neither properly
   (`HW-02-0088`).

## Verified snippets

There is no archive to verify against (`HW-02-0082`), so these are quoted from
the document with its own line numbers. Corrections are marked and derived from
the referenced HarmonyOS documentation or from arithmetic on the snippet itself.

**The month pager - document lines 30-52** (as published; two corrections marked)

```typescript
Swiper(this.swiperController) {
  // 使用懒加载渲染数据
  LazyForEach(this.vm.dateListSource, (item: DateModelList) => {
    Grid() {
      ForEach(item, (item: DateModel) => {
        GridItem() {
          // 根据数据渲染日期
        }
      });
    }
    .columnsTemplate('1fr 1fr 1fr 1fr 1fr 1fr 1fr')
    // .maxCount(7)                       FIX: inert when columnsTemplate is set
    .columnsGap(0)
    .rowsGap(0)
    .padding({ bottom: 20 })
  }, (item: DateModelList) => item.monthKey);   // FIX: the document supplies no key generator
}
.cachedCount(0)
.indicator(false)
.loop(false)
.itemSpace(0)
.effectMode(EdgeEffect.None)
```

**`columnsTemplate` with seven `1fr` and no `rowsTemplate`** is the correct mode
for a calendar: the Grid arranges the cells in seven columns, wraps to as many
rows as the data needs, and - per the reference - "the main axis size of the grid
is the maximum value of the main axis sizes of all grid items", so the month
sizes itself. Setting `rowsTemplate` as well would switch to the fixed,
non-scrolling mode and pin the row count.

**`.maxCount(7)` does nothing here.** The Grid reference lists three layout modes
and states, for the one in use: "In this mode, the following attributes do not
take effect: layoutDirection, **maxCount**, minCount, and cellLength." The row
cap the prose promises has to come from the data instead (`HW-02-0083`).

**`cachedCount(0)` with `loop(false)`** keeps only the visible month alive - a
deliberate memory choice for a pager whose neighbours are cheap to rebuild, and
the opposite of the usual advice. It pairs with step 2's "fetch two months
either side" to trade memory for a rebuild on every swipe.

**`effectMode(EdgeEffect.None)`** removes the rubber-band at the ends. The
reference notes it "takes effect only when `loop` is set to `false` or all child
nodes are displayed on one screen", which `loop(false)` on the line above
satisfies.

**The date algorithm - document lines 56-100** (as published)

```typescript
getDateList(param: number) {
  const baseDate = dayjs().add(param, 'month');
  const firstDate = baseDate.startOf('month');
  const afterDate = baseDate.endOf('month');

  // pad backwards to the configured week start
  const firstDateDay = firstDate.day();          // 0..6, Sunday = 0
  const weekStart = this.weekStart % 7;
  const frontPadding = (firstDateDay - weekStart + 7) % 7;
  const showFirstDay = firstDate.subtract(frontPadding, 'day');

  const baseDays = afterDate.diff(showFirstDay, 'day') + 1;
  const rowsNeeded = Math.ceil(baseDays / 7);
  const totalRows = Math.max(5, rowsNeeded);     // floor only - see below
  const totalCells = totalRows * 7;

  let dateList: DateModel[] = [];
  for (let i = 0; i < totalCells; i++) {
    const dayjsObj = showFirstDay.add(i, 'day');
    const isCurrentMonth = dayjsObj.isSame(firstDate, 'month');
    const lunarDate = Lunar.fromDate(dayjsObj.toDate());
    let lunarDay = this.getLunarInfo(lunarDate)[0];
    let lunarColor = this.getLunarInfo(lunarDate)[1];
    const dateModel = new DateModel(
      dayjsObj, dayjsObj.date(), lunarDay, isCurrentMonth, lunarColor,
      dayjsObj.format('YYYY-M-D')
    );
    if (dayjsObj.isSame(this.selectDate, 'day')) {
      this.todayInfo = dateModel;
    }
    dateList.push(dateModel);
  }
  return dateList;
}
```

**`(firstDateDay - weekStart + 7) % 7` is the whole week-start abstraction.**
`+ 7` before the modulo is what keeps the result non-negative when the month
starts before the configured week start - the single most commonly botched line
in a calendar. Setting `weekStart` to 1 gives a Monday-first grid with no other
change.

**The row count can never reach the seven the prose claims.** `frontPadding` is
at most 6 and a month is at most 31 days, so `baseDays` is at most 37 and
`rowsNeeded = ceil(37 / 7) = 6`. `Math.max(5, ...)` raises the floor and never
the ceiling, so the range is five to six rows - which is why `.maxCount(7)` looks
like a cap and is not one (`HW-02-0083`).

**`isCurrentMonth` is computed against `firstDate`, not `baseDate`.** Both are in
the same month so the result is identical, but comparing against the normalised
first-of-month is the safer habit - `baseDate` is "today, shifted by N months",
which `dayjs` clamps when the source day does not exist in the target month.

**Overshooting to a whole number of rows is deliberate.** `totalCells` runs past
the end of the month into the next one, and those cells are marked
`isCurrentMonth: false` rather than being omitted - which is what keeps the Grid
rectangular.

`getLunarInfo(lunarDate)` is called twice per cell for two elements of the same
returned array; one call and a destructure would halve the lunar work in the
hot loop.

**The controller handle - document lines 107-115 and 122** (as published)

```typescript
// 日历日期切换以及获取今日宜今日忌信息
class CalendarController {
  public static vm: CalendarVM = CalendarVM.instance;

  public setSelectDate(date: Date) {
    CalendarController.vm.changeDate(date);
  }

  public getTodayYiJi() {
    CalendarController.vm.getTodayYiJi();
  }
}

// at the call site
this.calendarController.setSelectDate(new Date(date));
```

**A controller object is the right export for a calendar in a HAR.** The hosting
page gets a stable handle it can call without importing the view's state, and the
view stays free to re-render. This is the same shape as ArkUI's own
`SwiperController` / `TabsController`.

Note the static `vm` bound to a **singleton** `CalendarVM.instance`: every
`CalendarController` instance drives the same view model, so two calendars on one
screen would fight - the same shared-singleton hazard as `LIFE-08`'s
`DATE_TIME_RANGE` (`HW-02-0055`). `getTodayYiJi()` also returns `void` while its
name implies a value; the result must be arriving through the view model.

## Permissions & config

The document declares one permission, at line 139:

```
ohos.permission.INTERNET
```

That is consistent with the `Https.ets` in `base_calendar` and the mock-backed
services. It is a system-grant permission, so no runtime request is needed.

The module set the tree describes suggests the list is incomplete
(`HW-02-0085`): `components/traffic_restriction/utils/LocationPermission.ets`
implies a location permission, which would be user-grant and would need both a
`module.json5` declaration and a runtime request. `sdk/aggregated_payment` and
`login_info/LoginWithHuaweiID.ets` are likewise unreflected in the section.

Third-party dependencies are used but never declared (`HW-02-0088`): `dayjs`
appears in nine lines of the step-2 snippet and is not mentioned anywhere in
prose; the lunar library is named only in passing inside the mock-data note at
line 133.

No `module.json5` content is quoted anywhere in the document, and there is no
archive to read one from.

## Constraints

- The document declares **API Version 16 Release / HarmonyOS 5.0.4 Release SDK /
  DevEco Studio 5.0.4 Release** (lines 127-129) - four API levels behind the
  other 26 scenarios in this industry (`HW-02-0086`).
- **There is no sample archive.** The 代码下载 link is a Gitee `tree/main` URL,
  so the code behind this document is unpinned and can change (`HW-02-0082`).
- Everything is mock data (line 133): the service pages are local mocks, the
  almanac content comes from the lunar library, the verification-code flow is
  simulated (line 134), and login falls back to a simulated user when Huawei ID
  one-tap sign-in is not configured (line 135).
- The calendar month grid is five or six rows, never seven.
- `cachedCount(0)` means every swipe rebuilds a month; the algorithm runs
  `Lunar.fromDate` plus two `getLunarInfo` calls for each of 35-42 cells on each
  rebuild.
- `CalendarController.vm` is static and bound to `CalendarVM.instance`, so one
  calendar per process.
- The tree includes an `EntryFormAbility` and a `CalendarWidgetCard`, so the
  template also ships a widget - which the prose never mentions.

## Pitfalls

- **`HW-02-0082` - there is no sample zip.** The 代码下载 section links a live
  Gitee branch instead of an archive on Huawei's CDN, uniquely among this
  industry's 29 scenario documents. Nothing in this document can be checked
  against code, and the code can change under it.
- **`HW-02-0086` - the API floor is API 16 / SDK 5.0.4** while 26 sibling
  documents say API 20 / SDK 6.0.0. Do not assume the two generations of sample
  compose.
- **`HW-02-0083` - `.maxCount(7)` on a `Grid` that has `columnsTemplate` does
  nothing,** and the "maximum 7 rows" in the prose is one more than the printed
  algorithm can produce. The real range is five to six.
- **`HW-02-0087` - three entries in the project tree are copy-paste errors:**
  `base_apis` appears a third time where `base_calendar`'s root entry belongs,
  `yiji_query`'s view is listed as `VipCenter.ets`, and one entry is a `utils`
  directory whose only child is `utils`. With no archive, the tree is the only
  description of the layout there is.
- **`HW-02-0088` - `dayjs` and the lunar library are undeclared.** The central
  algorithm will not compile without both, and the document gives neither package
  name nor version.
- **`HW-02-0084` - the `LazyForEach` has no key generator,** so it falls back to
  the index-based default the guidance rules out - in a pager whose window slides
  by month, which is the case where that default misbehaves.
- **`HW-02-0085` - the permission section lists only `INTERNET`** although the
  tree includes a `LocationPermission.ets`, a payment SDK and a Huawei ID login.
- **Do not drop the `+ 7` from the front-padding modulo.** Without it the
  expression goes negative for any month starting before the configured week
  start, and `subtract(-n, 'day')` walks the wrong way.
- **Do not omit the trailing next-month cells.** Filling to `totalRows * 7` and
  flagging them `isCurrentMonth: false` is what keeps the grid rectangular.
- **Do not call `getLunarInfo` twice per cell.** It is inside a 35-42 iteration
  loop that runs on every month swipe.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` - the three `Grid` layout modes and the note that `maxCount` does not take effect when `columnsTemplate` is set
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `documentation/harmonyos-references/02_application-framework/ts-container-swiper.md` - `Swiper`, `loop`, `cachedCount`, `indicator`, `itemSpace`, `effectMode` and its `loop(false)` precondition
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-swiper
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-lazyforeach.md` - `IDataSource` and the key-uniqueness rule
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-lazyforeach
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.INTERNET` and the user-grant location permissions
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `documentation/harmonyos-guides/01_getting-started/har-package.md` - HAR constraints for the eighteen-module split
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/har-package
- `LIFE-01` - the industry's other layered-architecture document, with a three-layer split and an actual sample archive
- `LIFE-13` - the three-day calendar view, which does ship a zip and covers the same month-grid problem
- `LIFE-08` - the shared-singleton hazard `CalendarController.vm` repeats (`HW-02-0055`)
