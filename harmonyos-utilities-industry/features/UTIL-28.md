---
id: UTIL-28
title: Date calculator - solar/lunar conversion, day interval and day offset over i18n.Calendar
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/28_date_conversion.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/date_conversion-0000002402521613
sample: huawei_industry_tree/15_utilities/downloads/DateConversion.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.LocalizationKit", "@kit.PerformanceAnalysisKit"]
apis: ["i18n.getCalendar", "Calendar.setTime", "Calendar.get", "UIContext.showDatePickerDialog", DatePickerDialogOptions, onDateAccept, "UIContext.getPromptAction", showToast, Tabs, TabContent, TabsController, "resourceManager.getStringSync", "@Link", "$$" ]
min_api: 20
modules: [entry]
findings: [HW-15-0064, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card when the app has to **do arithmetic on dates and show the result
in a calendar system the platform does not format for you**. The three tabs here
are the three questions users actually ask: what is this date in the other
calendar, how many days between these two dates, and what date is N days from
this one.

The transferable mechanism is `i18n.getCalendar('zh-Hans', 'chinese')`. It gives
a Chinese lunisolar calendar object whose `setTime(Date)` accepts a Gregorian
instant and whose `get('year' | 'month' | 'date')` returns the lunar fields -
so conversion is a two-line operation and the only work left is rendering the
numbers as 甲子年 / 正月 / 初一. The same call with a different calendar name
covers Hebrew, Islamic, Indian and the rest, so this card is really "how to
build a converter over `i18n.Calendar`".

The interval and offset tabs need no i18n at all: they are millisecond
arithmetic on `Date.getTime()`, with `MILLISECONDS_PER_DAY` as the only
constant. What makes them non-trivial is the input handling - a free-text day
count that must reject leading zeros, empty input and values that would walk off
the end of the supported date range.

## Feature checklist

- Three tabs across the top - 日期转换 (conversion), 日期间隔 (interval),
  日期推算 (estimation) - as pill buttons, with the content non-swipeable.
- Conversion: a solar date row that opens the system date picker, and a read-only
  lunar row that updates with it.
- Interval: two solar date rows and a sentence reading
  "公历 A 和 公历 B 间隔 N 天".
- Estimation: a solar/lunar segmented switch, a chosen date, an editable day
  count, and a sentence reading "N 天后是 公历 X".
- Choosing a date or editing the day count recomputes the sentence immediately.
- The day count strips leading zeros, and an emptied field becomes `0`.
- All pickers are bounded to 1900-01-31 … 2100-12-31.
- On the lunar side, a result beyond 2100-12-31 is clamped back to the range and
  a toast explains the limit.

## Architecture

One `entry` module, one page, three views, one shared component, one conversion
utility.

```
entry/src/main/ets
├── components/SlideButton.ets      the solar/lunar segmented control - one animated pill over a ForEach
├── constants/CommonConstants.ets   tab indices, MILLISECONDS_PER_DAY, the gap-string defaults
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── pages/DateConversion.ets        @Entry - the title row and the three Tabs
├── utils
│  ├── Logger.ets
│  ├── ResourceToStr.ets            getResourceString(resource, context) -> string
│  └── SwitchDate.ets               solarToLunar, getLunarDay, HeavenlyStems, MonthSwitch, getTime
└── views
   ├── DateEstimation.ets           the offset tab, 384 lines - both calendars
   ├── DateInterval.ets             the interval tab
   └── DateTranslation.ets          the conversion tab
```

The documented tree matches the zip, except that the document spells the
constants folder `constrants`; the zip has `constants`.

**The design decision worth copying** is that dates are carried in two
representations at once and each has one job. `selectSolarTime: Date` is the
value handed to and received from the picker - it is the truth. `solarDate:
string` is the `YYYY/MM/DD` rendering, and it is what gets *parsed* by
`solarToLunar` and `calculationInterval`. Formatting through
`SwitchDate.getTime()` and re-splitting on `/` looks redundant, but it is what
normalises away the time-of-day component that the picker leaves on the `Date`,
so a conversion never straddles a day boundary because of an inherited
`14:37:02`.

The related trick is the **08:00 anchor**. Every `Date` built for conversion is
`new Date(y, m - 1, d, 8, 0, 0, 0)`. Eight in the morning is far enough from
both midnights that no timezone offset within ±8 hours can move the date, which
is the cheapest available defence against the classic off-by-one-day bug. Copy
it.

**Worth avoiding**, on the other hand, is the shape of `DateEstimation`: the
solar and lunar halves are two near-identical 130-line blocks of layout and
handler inline in one `build()`, and they have already drifted apart -
`HW-15-0064` is exactly that drift. Two instances of one parameterised
sub-component would have made the divergence impossible.

## Implementation steps

1. **Get the lunar calendar once per conversion**:
   `i18n.getCalendar('zh-Hans', 'chinese')`.
2. **Feed it a normalised `Date` anchored at 08:00**, built from the formatted
   string rather than from the picker's raw value.
3. **Read `year`, `month`, `date` back with `get()`.** `month` is 0-based, like
   the Gregorian `Date`.
4. **Render the sexagenary year from the numeric lunar year** with `% 10` and
   `% 12` into rotated stem and branch arrays.
5. **Bound every picker** with `start`/`end` from the `start_time` /
   `end_time` string resources (1900-01-31 and 2100-12-31), so the input can
   never leave the domain the converter supports.
6. **Compute the initial result in `aboutToAppear`**, in every tab, not only in
   the picker callbacks (`HW-15-0064`).
7. **Sanitise the day count on every change**: strip leading zeros with
   `value.replace(/^0+/, '')`, fall back to `'0'` when the field is emptied, and
   only then recompute.
8. **Clamp the computed result to the supported range on both branches** and
   toast when you do (`HW-15-0064`).
9. **Resolve string resources with a helper**, not inline, when they must be
   concatenated with a value - `getResourceString` wraps `getStringSync` in a
   try/catch that logs and returns `''`.
10. **Give the tab bar its own `TabsController`** and set `.scrollable(false)`
    so the pills are the only way to switch: three unrelated calculators should
    not be swipe-adjacent.

## Verified snippets

All snippets are from `DateConversion.zip`. Corrected forms are marked.

**The conversion — `entry/src/main/ets/utils/SwitchDate.ets`** (as shipped)

```typescript
import { i18n } from '@kit.LocalizationKit';

export class SwitchDate {
  private lunarUnits = ['初', '十', '廿'];
  private lunarDigits = ['十', '一', '二', '三', '四', '五', '六', '七', '八', '九'];
  private lunarMonths = ['正', '二', '三', '四', '五', '六', '七', '八', '九', '十', '冬', '腊'];
  private heavenlyStems = ['癸', '甲', '乙', '丙', '丁', '戊', '己', '庚', '辛', '壬'];
  private earthlyBranches = ['亥', '子', '丑', '寅', '卯', '辰', '巳', '午', '未', '申', '酉', '戌'];

  HeavenlyStems(year: number): string {
    let heavenlyStemsIndex = year % 10;
    let earthlyBranchesIndex = year % 12;
    return this.heavenlyStems[heavenlyStemsIndex] + this.earthlyBranches[earthlyBranchesIndex] + '年';
  }

  solarToLunar(solarDate: string, selectedTime: Date): string {
    let calendar: i18n.Calendar = i18n.getCalendar('zh-Hans', 'chinese');
    let dateArray: number[] = solarDate.split('/').map(Number);
    calendar.setTime(new Date(dateArray[0], dateArray[1] - 1, dateArray[2], 8, 0, 0, 0));
    let year: number = calendar.get('year');
    let month: number = calendar.get('month');
    let day: number = calendar.get('date');
    let lunarYear = this.HeavenlyStems(year);
    let lunarMonth = this.MonthSwitch(month);
    let lunarDay = this.getLunarDay(day);
    let lastDay = selectedTime.getTime() - 1000 * 60 * 60 * 24 * (day + 1);
    calendar.setTime(lastDay);
    let lastMonth = calendar.get('month');
    let lunarDate: string = '';
    if (month === lastMonth) {
      lunarDate = lunarYear + '闰' + lunarMonth + lunarDay;
    } else {
      lunarDate = lunarYear + lunarMonth + lunarDay;
    }
    return lunarDate;
  }
}
```

**Three lines do the conversion; the rest is typography.** `getCalendar`,
`setTime`, `get` - that is the whole platform surface, and it handles leap
months, month lengths and the epoch without the app owning a lookup table.
Everything else in this file exists because ICU returns *numbers* and Chinese
lunar dates are read as *characters*.

**The two cyclical arrays are rotated on purpose.** `heavenlyStems` begins 癸
rather than 甲 and `earthlyBranches` begins 亥 rather than 子, which is what lets
`year % 10` and `year % 12` index them directly with no offset term - 甲 lands at
index 1, and a year ≡ 1 (mod 10) is indeed a 甲 year. It reads like an error and
is not; leave a comment if you copy it.

The leap-month detection is the weak part: it steps back `day + 1` days from the
selected instant and calls it a leap month if the lunar month number is
unchanged. That is a heuristic over `calendar.get('month')`, which does not
itself distinguish a leap month from its base month, and it uses `selectedTime`
- the caller's raw `Date` - rather than the normalised one the conversion was
performed on. Prefer checking whether the day count exceeds the month length, or
carry the leap flag from a calendar that exposes it.

**The interval — `entry/src/main/ets/views/DateInterval.ets`** (as shipped)

```typescript
calculationInterval(dateOne: string, dateTwo: string): string {
  let dateArrayOne: number[] = dateOne.split('/').map(Number);
  let dateArrayTwo: number[] = dateTwo.split('/').map(Number);
  let firstDate: Date = new Date(dateArrayOne[0], dateArrayOne[1] - 1, dateArrayOne[2], 8, 0, 0, 0);
  let lastDate: Date = new Date(dateArrayTwo[0], dateArrayTwo[1] - 1, dateArrayTwo[2], 8, 0, 0, 0);
  let timeDifference: number = Math.abs(lastDate.getTime() - firstDate.getTime());
  let result: string = Math.floor(timeDifference / CommonConstants.MILLISECONDS_PER_DAY).toString();
  return result;
}

aboutToAppear(): void {
  this.solarDateOne = SwitchDate.getTime(new Date());
  this.solarDateTwo = SwitchDate.getTime(new Date());
  this.gapDay = this.calculationInterval(this.solarDateOne, this.solarDateTwo);   // computed up front
}
```

**Both dates are rebuilt at 08:00 before subtracting**, which is what makes the
`Math.floor` safe: two instants that are both eight in the morning differ by a
whole number of days plus at most one DST hour, and flooring absorbs that hour.
Subtracting the raw picker values instead would give 0 days for a pair that
spans a midnight by minutes.

Note the shape of `aboutToAppear`: seed the inputs, then compute the result. The
estimation tab is missing that last line, which is half of `HW-15-0064`.

**Initial state for the estimation tab — `entry/src/main/ets/views/DateEstimation.ets`** (corrected, see `HW-15-0064`)

```typescript
aboutToAppear(): void {
  this.solarDate = SwitchDate.getTime(new Date());
  this.lunarDate = SwitchDate.getTime(new Date());
  this.lunarDate = SwitchDate.solarToLunar(this.lunarDate, this.selectLunarTime);
  this.selectLunarTime =
    new Date(this.selectSolarTime.getFullYear(), this.selectSolarTime.getMonth(), this.selectSolarTime.getDate(), 8,
      0, 0, 0);
  // FIX: both result strings are left empty by the sample
  this.estimationSolarDate = SwitchDate.getTime(new Date(this.selectSolarTime.getTime() +
    CommonConstants.MILLISECONDS_PER_DAY * Number.parseInt(this.solarGapDay)));
  this.estimationLunarDate = SwitchDate.solarToLunar(
    SwitchDate.getTime(new Date(this.selectLunarTime.getTime() +
      CommonConstants.MILLISECONDS_PER_DAY * Number.parseInt(this.lunarGapDay))),
    this.selectLunarTime);
}
```

`solarGapDay` and `lunarGapDay` both default to `CommonConstants.INITIAL_GAP`
(`'1'`), and the banner at the bottom of the tab renders
`solarGapDay + '天后' + '是' + '公历' + estimationSolarDate`. With
`estimationSolarDate` still `''`, the first thing a user sees on this tab is the
dangling sentence "1天后是 公历" with nothing after it. Every handler that could
fill it in requires the user to touch something first. Computing in
`aboutToAppear` - exactly as `DateInterval` and `DateTranslation` already do -
is the whole fix.

**The range clamp, and the branch that lacks it — same file** (as shipped, lunar branch)

```typescript
this.estimationLunarDate = SwitchDate.getTime(new Date(this.selectLunarTime.getTime() +
  CommonConstants.MILLISECONDS_PER_DAY * Number.parseInt(this.lunarGapDay)));
if (new Date(this.estimationLunarDate) >
  new Date(getResourceString($r('app.string.end_time'), this.context))) {
  let timeDifference: number =
    Math.abs(new Date(getResourceString($r('app.string.end_time'), this.context)).getTime() -
    this.selectLunarTime.getTime());
  this.lunarGapDay = Math.floor(timeDifference / CommonConstants.MILLISECONDS_PER_DAY).toString();
  this.estimationLunarDate = SwitchDate.getTime(new Date(this.selectLunarTime.getTime() +
    CommonConstants.MILLISECONDS_PER_DAY * Number.parseInt(this.lunarGapDay)));
  this.estimationLunarDate =
    SwitchDate.solarToLunar(this.estimationLunarDate, new Date(this.estimationLunarDate));
  this.promptAction.showToast({
    message: $r('app.string.no_lunar_date')     // 本示例农历最大日期为庚申年腊月初一
  });
} else {
  this.estimationLunarDate =
    SwitchDate.solarToLunar(this.estimationLunarDate, new Date(this.estimationLunarDate));
}
```

**This is the behaviour the solar branch should have and does not.** When the
offset would push the result past `end_time` (2100-12-31), the lunar branch
recomputes the largest legal gap, rewrites the input field to it, recomputes the
result and toasts the reason. The solar branch at the top of the same file runs
the identical `estimationSolarDate = …` line with no comparison, so typing
`99999` days there silently produces a date outside the range every picker in
the app enforces - and outside the range the converter is documented for. Wrap
the solar assignment in the same comparison; the toast can reuse a solar-specific
string.

The solar input has a second asymmetry: it is bound as
`TextInput({ text: this.solarGapDay, placeholder: '' })` while the lunar one uses
`TextInput({ text: $$this.lunarGapDay })`. Without `$$`, the binding is one-way,
so the sanitising assignment `this.solarGapDay = processedValue` updates the
state but not the visible field - a pasted `007` stays `007` on screen while the
calculation uses `7`.

**The segmented control — `entry/src/main/ets/components/SlideButton.ets`** (as shipped)

```typescript
@Link tabSelectedIndex: number;

build() {
  Stack() {
    Row()
      .height(this.selectedButtonHeight)
      .width('50%')
      .backgroundColor($r('sys.color.comp_background_primary_contrary'))
      .borderRadius(this.selectedButtonHeight / 2)
      .offset({ x: `${CommonConstants.PERCENT_50 * this.tabSelectedIndex}%` })
      .animation({ duration: 250, curve: Curve.EaseInOut })
      .shadow({ radius: 3, color: $r('app.color.slide_button_shadow') });
    Row() {
      ForEach(this.categoryNames, (name: ResourceStr, index: number) => {
        Text(name)
          .zIndex(100)
          .width('50%')
          .fontColor(this.tabSelectedIndex === index ? this.selectedFontColor : this.fontColor)
          .onClick(() => {
            this.tabSelectedIndex = index;
          });
      }, (name: ResourceStr) => JSON.stringify(name));
    };
  }
  .backgroundColor('#0D000000')
  .alignContent(Alignment.Start);
}
```

**One pill, one percentage, one declarative animation.** The moving highlight is
a single `Row` at 50% width whose `offset.x` is `50 * index` percent, with
`.animation()` attached to it - so the slide is a consequence of the index
changing rather than something a handler drives. The labels sit above it in a
sibling `Row` with `zIndex(100)`, which is what lets the pill pass under the text
instead of covering it. This is the cheapest correct segmented control in ArkUI
and it is worth lifting wholesale.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

Configuration lives in string resources rather than constants, which is the
right call for two of them: `start_time` (`1900-01-31`) and `end_time`
(`2100-12-31`) are the supported date domain and are read back through
`getResourceString` wherever a picker or a clamp needs them. `no_lunar_date`
carries the user-facing explanation of the upper bound
(本示例农历最大日期为庚申年腊月初一 - "the maximum lunar date in this sample is
the first day of the twelfth month of the 庚申 year").

All strings are Chinese, in `base` only.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The supported range is 1900-01-31 … 2100-12-31**, enforced by every picker's
  `start`/`end` and by the lunar clamp. The lunar side additionally stops at
  庚申年腊月初一, slightly earlier than the Gregorian bound.
- `new Date(getResourceString(...))` parses `'1900-01-31'` as an ISO date, i.e.
  **UTC midnight**. West of Greenwich that is 30 January locally - a one-day
  discrepancy between the declared bound and the enforced one.
- The leap-month rule in `solarToLunar` is a heuristic (see above) and is fed
  the caller's un-normalised `Date`; treat 闰 output as indicative.
- `getLunarDay` maps 1–30 only, via `lunarUnits[floor((day-1)/10)]` and
  `lunarDigits[day % 10]`. It is correct for a lunar month but has no defence if
  `get('date')` ever returned something else.
- The day-count field is `maxLength(5)` and `InputType.Number`, so the offset is
  bounded at 99999 days (~273 years) - which is why the solar branch's missing
  clamp is reachable in one paste.
- Nothing is persisted and there is no history; the 加 (add) button in the title
  row raises a 仅供展示 (for display only) toast.

## Pitfalls

- **`HW-15-0064` (B/low, confirmed) — the estimation tab opens on an empty
  result, and its solar branch lacks the range clamp its lunar sibling has.**
  `aboutToAppear` seeds the dates but never computes `estimationSolarDate` /
  `estimationLunarDate`, so the banner reads "1天后是 公历" with nothing after
  it until the user touches something; and the solar offset is written with no
  comparison against `end_time`, so a large gap silently leaves the app's
  declared date domain while the lunar path clamps and toasts. Fix: compute both
  results in `aboutToAppear`, and mirror the lunar clamp onto the solar branch.
  While there, bind the solar day-count field with `$$` as the lunar one is.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-i18n.md` - `getCalendar`, `Calendar.setTime`, `Calendar.get` and the supported calendar names
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-i18n
- `documentation/harmonyos-references/02_application-framework/ts-methods-datepicker-dialog.md` - `showDatePickerDialog`, `DatePickerDialogOptions`, `lunar`, `onDateAccept`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-methods-datepicker-dialog
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `showDatePickerDialog` and `getPromptAction` off the UIContext
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` - `Tabs`, `TabsController`, custom `tabBar` builders, `scrollable`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md` - `$$` two-way binding, `InputType.Number`, `maxLength`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
