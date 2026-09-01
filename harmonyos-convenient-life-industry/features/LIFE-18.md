---
id: LIFE-18
title: Week / month / detail calendar - one Swiper whose data source swaps per view, with a vertical PanGesture switching between them
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/18_calendar_swiper.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/calendar_swiper-0000002355470245
sample: huawei_industry_tree/02_convenient_life/downloads/CalendarSwiper.zip
kits: ["@kit.ArkUI", "@kit.ArkTS", "@kit.LocalizationKit", "@kit.AbilityKit"]
apis: [Swiper, SwiperController, "swiper.index", onAnimationStart, onAnimationEnd, loop, indicator, PanGesture, PanDirection, GestureEvent, "UIContext.animateTo", Curve, "@Watch", "@State", "@Link", "@Prop", "@StorageProp", "$$", ForEach, "resourceManager.getRawFileContentSync", "util.TextDecoder", "util.DecodeToStringOptions", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.on('avoidAreaChange')"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-02-0127, HW-02-0128, HW-02-0129, HW-02-0130, HW-02-0131, HW-02-0132, HW-02-0133, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card for a calendar that **collapses from a month grid to a single
week and expands back**, with a horizontal `Swiper` paging through periods in
either mode - the shape of every system calendar app.

Two ideas carry it:

1. **One `Swiper`, two data sources.** The page holds `monthRanges` and
   `weekRanges`, both derived from the same flat array of weeks. Switching view
   swaps which array the `ForEach` iterates, and a `@Watch` on the view type
   re-points `Swiper.index` at the page containing the selected day - so the
   collapse looks like the month folding down onto its own week rather than a
   jump.
2. **The collapse is a height animation, not a re-layout.** Each page renders
   *all* the weeks of its month at all times; week view is the same grid with
   `calendarHeight` animated down to one row and `topOffset` translated so the
   selected week is the one left visible.

That second point is the reusable trick: because nothing is added or removed,
`animateTo` has a continuous property to interpolate and the transition is free.

Take this for calendars, date-range pickers, any period browser that has both a
compact and an expanded form. The range-derivation code (`CalendarUtil`) is the
part that needs the most care - two of its behaviours are wrong
(`HW-02-0127`, `HW-02-0128`).

## Feature checklist

- Month view: a grid of the month's weeks, paged horizontally.
- Swipe up on the month grid to collapse to a single-week strip; swipe down to
  expand back.
- Swipe down again from month view to open a taller detail view with agenda dots
  and shift markers.
- The selected day survives every switch; switching to another month moves the
  selection to the same day-of-month.
- Paging in week view advances the selected week.
- Data comes from a bundled rawfile of 14 weeks.

## Architecture

One `entry` module. The calendar is a page plus one component, with all the
period arithmetic in a util class.

```
entry/src/main/ets
├── common/CommonConstant.ets      grid sizes, PAN_GESTURE_OFFSET 5, animation duration 300
├── components/CalendarView.ets    one Swiper page: the grid, the height animation, the gesture
├── entryability/EntryAbility.ets  full screen, avoid areas -> AppStorage
├── model
│   ├── CalendarViewData.ets       CalendarViewType enum, CalendarViewDay, CalendarAgenda
│   └── CalendarViewRange.ets      CalendarViewRange { startWeekIndex, endWeekIndex, weekViewIndex, month }
├── pages/CalendarPage.ets         THE CARD: both range arrays, the Swiper, the view-switch watch
└── utils
    ├── CalendarUtil.ets           getCalendarRange, findSameDay, adjustMonthRange, isWeekInMonthRange
    └── RawfileUtil.ets            rawfile -> CalendarViewDay[][]
resources/rawfile/calendar_data.json   14 weeks x 7 days, 2025-04-27 to 2025-08-02
```

The documented tree matches the zip exactly.

**Everything is an index into one flat array.** `data: CalendarViewDay[][]` is
weeks-by-weekdays, and every other structure is a pair of bounds into it:

```
CalendarViewRange { startWeekIndex, endWeekIndex, weekViewIndex, month }

month page   -> weeks [startWeekIndex, endWeekIndex)   weekViewIndex = startWeekIndex
week page    -> the same bounds, but weekViewIndex names the one week to show
selection    -> selectedWeekIndex + selectedDayIndex, indices into data
```

A week page therefore carries its whole month's bounds *and* the single week to
display, which is what lets the collapse animate: the grid is unchanged, only
`calendarHeight` and `topOffset` move.

**The switch cycle:**

```
PanGesture (vertical, >5 vp)  in CalendarView
   MONTH + down  -> changeToDetailView()     grid 24 -> 40, height grows
   MONTH + up    -> changeToWeekView()       height -> one row, topOffset scrolls to the week
   WEEK  + down  -> changeToMonthView()
   DETAIL + up   -> changeToMonthView()
        each animateTo(300ms) sets @Link viewType in onFinish

@Watch('refreshCalendarData') on viewType in CalendarPage
   pick the range array for the new view
   find the page holding the selected day  -> currentIndex
   Swiper.index is $$-bound, so it moves
```

Note the ordering: the **animation runs first and the view type flips in
`onFinish`**. The height is already where the new view wants it by the time the
data source swaps, so the swap is invisible.

## Implementation steps

1. **Flatten the calendar to weeks-by-days once** and derive everything else as
   index ranges over it. That is what makes the two views two projections of one
   array rather than two datasets.
2. **Build the month ranges by scanning for month transitions - and flush the
   last one after the loop** (`HW-02-0127`). A push that only happens on a
   transition never emits the final period.
3. **Derive the week ranges from the month ranges,** one entry per week, each
   keeping its month's bounds so the expansion animation has somewhere to grow
   to.
4. **Give each range a stable id** and key the `Swiper`'s `ForEach` on it, not
   on the serialised object (`HW-02-0129`).
5. **Never mutate the precomputed ranges** (`HW-02-0128`). Hold any transient
   expansion bounds in separate state.
6. **Bind `Swiper.index` with `$$`** so the `@Watch` can re-point it and a user
   swipe can write back to it through the same variable.
7. **Animate `calendarHeight` and `topOffset`, and flip the view type in
   `onFinish`.** Guard re-entry with a `canAnimate` flag so a second gesture
   during the transition is ignored.
8. **Read the target week from the page being swiped to,** not from the page
   delta (`HW-02-0130`).
9. **Guard the rawfile load** and do not index into the data before it is
   populated (`HW-02-0131`).

## Verified snippets

All snippets are from `CalendarSwiper.zip`. Corrected forms are marked.

**Deriving the month pages - `CalendarSwiper.zip#entry/src/main/ets/utils/CalendarUtil.ets:27`** (corrected, see `HW-02-0127`)

```typescript
static getCalendarRange(data: CalendarViewDay[][], viewType: CalendarViewType): CalendarViewRange[] {
  let res: CalendarViewRange[] = [];
  if (data.length > 0) {
    let currentMonth = data[0][SATURDAY_INDEX].month;
    let startWeekIndex = 0;
    for (let i = 0; i < data.length; i++) {
      if (data[i][SATURDAY_INDEX].month !== currentMonth) {
        if (data[i][SUNDAY_INDEX].month !== currentMonth) {
          res.push({ startWeekIndex, endWeekIndex: i, weekViewIndex: startWeekIndex, month: currentMonth });
        } else {
          // the boundary week straddles two months: give it to both
          res.push({ startWeekIndex, endWeekIndex: i + 1, weekViewIndex: startWeekIndex, month: currentMonth });
        }
        currentMonth = data[i][SATURDAY_INDEX].month;
        startWeekIndex = i;
      }
    }
    // FIX: the sample stops here, so the month still open when the loop ends never gets a page
    res.push({ startWeekIndex, endWeekIndex: data.length, weekViewIndex: startWeekIndex, month: currentMonth });
  }

  if (viewType === CalendarViewType.WEEK) {
    let weekRes: CalendarViewRange[] = [];
    let lastWeekIndex = -1;
    for (let i = 0; i < res.length; i++) {
      for (let j = 0; j < (res[i].endWeekIndex - res[i].startWeekIndex); j++) {
        if (lastWeekIndex !== res[i].startWeekIndex + j) {     // dedup the shared boundary week
          weekRes.push({
            startWeekIndex: res[i].startWeekIndex,             // the month's bounds, for the expansion
            endWeekIndex: res[i].endWeekIndex,
            weekViewIndex: res[i].startWeekIndex + j,          // the one week this page shows
            month: res[i].month
          });
          lastWeekIndex = res[i].startWeekIndex + j;
        }
      }
    }
    return weekRes;
  }
  return res;
}
```

**The `SUNDAY_INDEX` test is the boundary-week rule.** When a week's Saturday
has already crossed into the next month but its Sunday has not, the week belongs
to both, so the outgoing month's range is extended by one (`endWeekIndex: i + 1`)
and the incoming month starts at the same index. That overlap is deliberate and
is what `isWeekInMonthRange` later disambiguates by also comparing the month.

**A week page carries its month's bounds, not just its own week.** That is not
redundancy - `changeToMonthView` grows `calendarHeight` to
`(gridGap + gridSize) * (endWeekIndex - startWeekIndex)`, so the page already
knows how tall it has to become.

**The missing flush is the defect.** With the bundled data - Saturday months
`5,5,5,5,5,6,6,6,6,7,7,7,7,8` - transitions at weeks 5, 9 and 13 emit pages for
May, June and July, and August never gets one. Week 13's August days are
rendered inside the page labelled July.

**Both views from one Swiper - `CalendarSwiper.zip#entry/src/main/ets/pages/CalendarPage.ets:137`** (corrected, see `HW-02-0129` and `HW-02-0130`)

```typescript
Swiper(this.swiperController) {
  ForEach(this.viewType === CalendarViewType.WEEK ? this.weekRanges : this.monthRanges,
    (range: CalendarViewRange, index: number) => {
      CalendarView({
        viewType: this.viewType,
        monthData: this.data,
        range: range,
        selectedDayIndex: this.selectedDayIndex,
        selectedWeekIndex: this.selectedWeekIndex,
        currentIndex: this.currentIndex,
        swiperIndex: index,
        swiperController: this.swiperController
      });
    }, (range: CalendarViewRange) => range.id);      // FIX: sample uses JSON.stringify(range) + index
}
.index($$this.currentIndex)
.loop(false)
.indicator(false)
.onAnimationStart((index, targetIndex) => {
  if (this.viewType === CalendarViewType.WEEK) {
    // FIX: sample does this.selectedWeekIndex += targetIndex - index
    this.selectedWeekIndex = this.weekRanges[targetIndex].weekViewIndex;
  } else if (targetIndex !== index &&
    this.monthRanges[targetIndex].month !== this.data[this.selectedWeekIndex][this.selectedDayIndex].month) {
    // 月视图切换时，切换到同日期的日子
    let pos = CalendarUtil.findSameDay(this.data, this.selectedWeekIndex, this.selectedDayIndex,
      this.monthRanges[targetIndex]);
    this.selectedWeekIndex = pos[0];
    this.selectedDayIndex = pos[1];
  }
})
.onAnimationEnd(() => {
  this.refreshCalendarData();
});
```

**The ternary in the `ForEach` is the whole two-view design.** Changing
`viewType` changes which array is iterated, so the Swiper's page count and
contents both switch with one `@State` write - no second Swiper, no conditional
rendering of two trees.

**`$$this.currentIndex` has to be two-way.** The `@Watch` writes it to re-point
the Swiper after a view switch, and a user swipe writes it back - a one-way bind
would make the programmatic jump stick and the manual swipe snap back.

**`onAnimationStart` versus `onAnimationEnd`.** The selection is moved at the
*start* of the page transition so the incoming page renders with the right day
already highlighted; the range/index reconciliation runs at the *end*, once the
Swiper has settled.

**`findSameDay` keeps the day-of-month across a month swipe** - swiping from
15 May to June selects 15 June, falling back to the last matching day when the
target month is shorter.

**Re-pointing after a view switch - same file, line 36** (corrected, see `HW-02-0128`)

```typescript
@State @Watch('refreshCalendarData') viewType: CalendarViewType = CalendarViewType.MONTH;

refreshCalendarData() {
  let ranges = this.viewType === CalendarViewType.WEEK ? this.weekRanges : this.monthRanges;
  for (let i = 0; i < ranges.length; i++) {
    let range = ranges[i];
    if (this.viewType === CalendarViewType.WEEK) {
      if (this.selectedWeekIndex === range.weekViewIndex) {
        this.currentIndex = i;
        // FIX: the sample calls CalendarUtil.adjustMonthRange(range, selectedDay) here,
        //      permanently rewriting this element of weekRanges. Hold the expansion
        //      bounds in separate state instead.
        break;
      }
    } else if (CalendarUtil.isWeekInMonthRange(this.selectedWeekIndex,
      this.data[this.selectedWeekIndex][this.selectedDayIndex], range)) {
      this.currentIndex = i;
      break;
    }
  }
  if (this.currentIndex >= ranges.length) {
    this.currentIndex = ranges.length - 1;         // the two arrays have different lengths
  }
}
```

**The clamp at the end is necessary, not defensive.** `weekRanges` is far longer
than `monthRanges`, so a `currentIndex` valid in week view is usually past the
end of the month array; without the clamp the Swiper would be asked for a page
that does not exist.

**`isWeekInMonthRange` compares the month as well as the bounds** precisely
because the boundary week appears in two ranges - the bounds alone would match
both and the search would stop at the wrong page.

The shipped `adjustMonthRange` call rewrites `startWeekIndex`, `endWeekIndex`
and `month` on an element of the array the Swiper is rendering from. It is
intended to widen the week page's bounds to the selected day's month so the
expansion animation has the right target height - but it does so by editing
reference data that is never rebuilt.

**The collapse animation - `CalendarSwiper.zip#entry/src/main/ets/components/CalendarView.ets:130`** (as shipped)

```typescript
changeToWeekView() {
  if (this.canAnimate) {
    this.canAnimate = false;                      // re-entry guard for the 300 ms window
    let weekViewIndex =
      this.selectedWeekIndex >= this.range.startWeekIndex &&
        this.selectedWeekIndex < this.range.endWeekIndex ? this.selectedWeekIndex : this.range.weekViewIndex;
    this.getUIContext().animateTo({
      duration: VIEW_SWITCH_ANIMATION_DURATION,
      curve: Curve.Linear,
      onFinish: () => {
        this.viewType = CalendarViewType.WEEK;    // the data source swaps only after the animation
        this.canAnimate = true;
      }
    }, () => {
      this.calendarHeight = this.gridGap + this.gridSize;                       // one row
      this.topOffset = -(this.gridGap + this.gridSize) * (weekViewIndex - this.range.startWeekIndex);
    });
  }
}

changeToMonthView() {
  if (this.canAnimate) {
    this.canAnimate = false;
    this.getUIContext().animateTo({
      duration: VIEW_SWITCH_ANIMATION_DURATION,
      curve: Curve.Linear,
      onFinish: () => {
        this.viewType = CalendarViewType.MONTH;
        this.canAnimate = true;
      }
    }, () => {
      this.gridSize = GRID_SMALL_SIZE;
      this.calendarHeight =
        (this.gridGap + this.gridSize) * (this.range.endWeekIndex - this.range.startWeekIndex);
      this.topOffset = 0;
      this.tipOffset = SHIFT_TIP_TOP_OFFSET_SMALL;
    });
  }
}
```

**Height plus offset, not add and remove.** The page always renders every week
of its month; collapsing sets the container height to one row and translates the
content up by `(rowHeight) * (weekToShow - firstWeekOfMonth)`. `animateTo` then
has two plain numbers to interpolate, and the weeks that scroll out of view are
clipped rather than destroyed - which is why the transition is smooth and why the
week page needs to know its month's bounds.

**Flipping `viewType` in `onFinish` is the ordering that makes it work.** The
data source swap - and with it the Swiper's page count - happens after the height
is already correct, so no frame shows a month grid at week height or vice versa.

**`canAnimate` is a re-entry guard, not a flag.** A `PanGesture` can fire again
inside the 300 ms window; without it two overlapping `animateTo` calls would
fight over `calendarHeight`.

**The gesture - same file, line 296** (as shipped)

```typescript
PanGesture({ direction: PanDirection.Vertical })
  .onActionStart((event: GestureEvent) => {
    if (this.viewType === CalendarViewType.MONTH) {
      if (event.offsetY > PAN_GESTURE_OFFSET) {
        this.changeToDetailView();
      } else if (event.offsetY < -PAN_GESTURE_OFFSET) {
        this.changeToWeekView();
      }
    } else if ((this.viewType === CalendarViewType.DETAIL && event.offsetY < -PAN_GESTURE_OFFSET) ||
      (this.viewType === CalendarViewType.WEEK && event.offsetY > PAN_GESTURE_OFFSET)) {
      this.changeToMonthView();
    }
  })
```

**`onActionStart`, not `onActionUpdate`.** The switch is a discrete decision, so
it is taken once at recognition from the initial direction rather than tracked
per frame - the animation, not the finger, drives the height. `PAN_GESTURE_OFFSET`
is 5 vp, a direction threshold rather than a distance the user must drag.

The three-state machine is complete in one expression: month goes to either
neighbour, and both neighbours return to month.

**Loading the data - `CalendarSwiper.zip#entry/src/main/ets/utils/RawfileUtil.ets`** (corrected, see `HW-02-0131`)

```typescript
static readDataFromRawfile(resourceManager: resourceManager.ResourceManager): CalendarViewDay[][] {
  try {                                                        // FIX: the sample has no guard
    const dataStr = RawfileUtil.uint8ArrayToString(
      resourceManager.getRawFileContentSync('calendar_data.json'));
    return JSON.parse(dataStr) as CalendarViewDay[][];
  } catch (error) {
    hilog.error(0x0000, 'RawfileUtil', 'calendar_data.json failed: %{public}s', JSON.stringify(error));
    return [];
  }
}

static uint8ArrayToString(array: Uint8Array) {
  let textDecoderOptions: util.TextDecoderOptions = { fatal: false, ignoreBOM: true };
  let decodeToStringOptions: util.DecodeToStringOptions = { stream: false };
  let textDecoder = util.TextDecoder.create('utf-8', textDecoderOptions);
  return textDecoder.decodeToString(array, decodeToStringOptions);
}
```

`fatal: false` plus `ignoreBOM: true` is the tolerant pair: a stray byte becomes
a replacement character rather than a throw, and a UTF-8 BOM does not end up in
front of the opening brace. Both `getRawFileContentSync` and `JSON.parse` still
throw, which is what the added `try` covers.

## Permissions & config

None. `CalendarSwiper.zip#entry/src/main/module.json5` declares no
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
    }]
  }
}
```

Root `build-profile.json5` targets `6.0.0(20)`. The dataset lives at
`entry/src/main/resources/rawfile/calendar_data.json`.

`EntryAbility` sets `setWindowLayoutFullScreen(true)`, reads both avoid areas
**inside** the promise's `then` - the correct ordering, unlike `LIFE-03` and
`LIFE-11` - and subscribes to `avoidAreaChange` without ever releasing it
(`HW-02-0132`). The page consumes only the top inset, with `@StorageProp`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 100-102).
- The data is a fixed 14-week rawfile (2025-04-27 to 2025-08-02). There is no
  generation, no paging beyond it, and no notion of today (`HW-02-0133`).
- `CalendarViewDay` carries `monthStartWeekIndex` and `monthEndWeekIndex`
  per day, so the dataset must be internally consistent - the code trusts those
  fields rather than recomputing them.
- Week and month index spaces line up 1:1 only because the week-range builder
  emits exactly one entry per data week; the `+=` in `onAnimationStart` depends
  on that (`HW-02-0130`).
- The boundary week belongs to two month ranges by design, which every lookup
  has to disambiguate by month.
- Each page renders its whole month at all times, so the collapse is a clip, not
  a virtualisation - fine at a month's 5-6 rows, not a strategy for a year view.
- `loop(false)` is required: a looping Swiper would wrap from the last month to
  the first, and `findSameDay` has no notion of that discontinuity.

## Pitfalls

- **`HW-02-0127` - `getCalendarRange` never emits the last month.** The push
  happens only on a month transition and there is no flush after the loop, so
  the final period has no page and its weeks appear under the previous month's
  label. Add the flush.
- **`HW-02-0128` - switching to week view rewrites an element of `weekRanges` in
  place.** `adjustMonthRange` overwrites `startWeekIndex`, `endWeekIndex` and
  `month` on precomputed reference data that is never rebuilt, and the local
  variable is even named `monthRange` while the branch is indexing `weekRanges`.
- **`HW-02-0129` - the `ForEach` key is `JSON.stringify(range)`,** whose every
  field the previous defect mutates - so each adjustment tears down and rebuilds
  the page, discarding its animation state. The two defects mask each other:
  `range` is a `@Prop`, so without the key churn the child would never see the
  mutation at all.
- **`HW-02-0130` - the week selection is advanced by the page delta**
  (`selectedWeekIndex += targetIndex - index`) rather than read from
  `weekRanges[targetIndex].weekViewIndex`, which the model already carries. It
  works only while the two arrays happen to be the same length.
- **`HW-02-0131` - the rawfile read has no `try` and no fallback,** while
  `build()` indexes `data[2][3]` unconditionally - so a missing or malformed
  file throws during render rather than degrading.
- **`HW-02-0132` - `on('avoidAreaChange')` has no `off()`.**
- **`HW-02-0133` - the initial selection is the hard-coded pair (2, 3)** rather
  than today; nothing in the zip calls `new Date()`.
- **Do not flip the view type before the animation.** Setting it in `onFinish`
  is what keeps the data-source swap out of the animated frames.
- **Do not drop the `canAnimate` guard.** A second vertical pan inside the
  300 ms window starts a competing `animateTo` on the same height.
- **Do not give a week page only its own week's bounds.** The expansion needs
  the month's bounds to know how tall to grow.
- **Do not forget that the boundary week is in two ranges.** Every lookup must
  compare the month as well as the indices.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-swiper.md` - `Swiper`, `index` with `$$`, `loop`, `indicator`, `onAnimationStart`/`onAnimationEnd`, `SwiperController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-swiper
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - `PanGesture`, `PanDirection.Vertical`, `onActionStart`, `GestureEvent.offsetY`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `animateTo` and its `onFinish`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - key generation and the "use a unique id" rule
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-guides/03_application-framework/arkts-watch.md` - `@Watch` on a state variable
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-watch
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getRawFileContentSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `documentation/harmonyos-references/02_application-framework/js-apis-util.md` - `TextDecoder`, `TextDecoderOptions.fatal`/`ignoreBOM`, `DecodeToStringOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-util
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `on`/`off('avoidAreaChange')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `LIFE-13` - the same industry's day-view calendar, with the hour-ruler problem instead
- `LIFE-12` - the month-grid calendar documented without a sample archive
- `LIFE-01` - the industry shell this page would sit in
