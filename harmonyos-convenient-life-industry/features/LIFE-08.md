---
id: LIFE-08
title: Combined date-and-time wheel - six non-looping TextPickers in one Row, shown through PromptAction.openCustomDialog
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/08_schedule.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule-0000002284438545
sample: huawei_industry_tree/02_convenient_life/downloads/Schedule.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: [TextPicker, canLoop, onChange, ComponentContent, wrapBuilder, "UIContext.getPromptAction", "promptAction.openCustomDialog", "promptAction.closeCustomDialog", "ComponentContent.dispose", "promptAction.BaseDialogOptions", autoCancel, "@Builder", "@Prop", "@Require", "@Watch", "@State", "@Provide", "@Observed", "@Track", "@Extend", TextInputController, Toggle, ToggleType, Visibility, "window.getMainWindow", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "ConfigurationConstant.ColorMode"]
permissions: []
min_api: 20
modules: [entry, dateTimePicker]
findings: [HW-02-0051, HW-02-0052, HW-02-0053, HW-02-0054, HW-02-0055, HW-02-0056, HW-02-0057, HW-02-0058, HW-02-0059, HW-02-0060, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card when you need **date and time on one wheel** - year, month, day,
hour, minute, second visible together, scrolling independently, not looping -
which neither `DatePicker` nor `TimePicker` gives you, and which
`DatePickerDialog` cannot combine.

The construction is six `TextPicker`s in a `Row`, each fed a string array
(`['2024年', '2025年', ...]`, `['1月', ...]`, `['1日', ...]`), each reporting
through one shared handler that rebuilds a `Date`. Day counts recompute when the
year or month changes, so February shows 28 or 29. Columns are hidden rather
than removed when the caller asks for less precision.

The dialog around it is the second half of the card: `ComponentContent` +
`UIContext.getPromptAction().openCustomDialog()`, which is how you show an
arbitrary component as a dialog from a HAR that has no `@CustomDialog` host.

Take this for scheduling, booking, reminder and deadline pickers. If you only
need a date, `DatePickerDialog` is one call.

## Feature checklist

- Six independent wheels - year, month, day, hour, minute, second - side by side.
- No looping: the first year and the last year are hard stops.
- The day column resizes to the selected month, leap years included.
- `timeFormat` trims the wheel from the right: date only, +hours, +minutes,
  +seconds.
- The picker reports every change through a callback as a complete `Date`.
- It is presented as a bottom dialog with a title, a cancel button and a confirm
  button; only confirm commits the value.
- The host form shows start and end rows that open the same dialog with
  different titles, and rejects an end that is not after the start.

## Architecture

Two modules: an `entry` HAP and a `dateTimePicker` HAR that is genuinely
reusable - it exports the component, the dialog entry point and the parameter
interfaces, and depends on nothing but the SDK.

```
dateTimePicker (HAR, "@ohos/datetimepicker")
├── Index.ets                              re-exports pages/, dialog/, interface/
└── src/main/ets
    ├── interface/DateTimePickerInterface.ets   DateIndex, TimeFormat, ...Data/Option/Style/Param
    ├── common/Constants.ets
    ├── utils
    │   ├── DateTimeBase.ets                range(), hour/minute/second ranges, isLeapYear
    │   ├── DateTimeSolar.ets               extends Base: year/month/day ranges
    │   └── DateTimeRange.ets               DATE_TIME_RANGE singleton  (HW-02-0055)
    ├── pages/DateTimePicker.ets            THE CARD: six TextPickers + the Date reducer
    └── dialog/DateTimePickerDialog.ets     ComponentContent + openCustomDialog + mapContext

entry
├── common/Constants.ets                    START_YEAR 1980, END_YEAR 2050
├── entryability/EntryAbility.ets           light mode, full screen, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
└── pages/NewSchedule.ets                   @Entry - the form, the two date rows, validation
```

The documented tree matches the zip exactly.

**Two enums carry the whole configuration.**

```
DateIndex   YEAR 0  MONTH 1  DAY 2  HOURS 3  MINUTES 4  SECOND 5   -- which wheel is this
TimeFormat  NONE 0  HOUR 1   MINUTES 2  SECOND 3                   -- how far right to show
```

`isVisibility` maps one to the other through a four-element lookup:
`[DAY, HOURS, MINUTES, SECOND][timeFormat]` is the last visible `DateIndex`.
`TimeFormat.NONE` therefore stops at `DAY`, and the default `MINUTES` hides only
the seconds wheel.

**The reduction is one-way and total.** Every wheel change routes to one handler
which updates a scalar (`currentYear` / `currentMonth` / `currentDay`) or the
`Date` directly, then rebuilds `selectDate` from all six components. A `@Watch`
on `selectDate` fires the caller's callback. There is no partial state: any
change produces a complete `Date`.

```
TextPicker.onChange(value, index)
  -> picker.callback(dateIndex, value, index)      the shared handler, per wheel
  -> dealSelectedYear/Month/Day  -> recompute dependent ranges -> updateSolarDate()
     dealSelectedTime            -> selectDate.setHours/Minutes/Seconds in place
  -> @Watch('onChangedSelectDate') -> data.resultCb(selectDate)
```

Note the asymmetry: date changes construct a **new** `Date`; time changes mutate
the existing one. Both are observed - the `@Prop`/`@State` `Date` guidance lists
`setHours`, `setMinutes` and `setSeconds` among the observed mutators - but only
the first path re-runs `updateSolarDate`.

**Dialog lifetime is tracked in a module-scope `Map` keyed by `Date.now()`,**
so the component can close the node that is showing it. That map is where the
leak lives (`HW-02-0051`).

## Implementation steps

1. **Define the parameter surface first** - `data` (range, initial value,
   callback), `option` (loop, precision), `style` - so the HAR has one entry
   shape. Keep the callback **out** of any `@Prop`-decorated object
   (`HW-02-0054`).
2. **Build the range helpers as pure functions of their inputs,** not as a
   singleton (`HW-02-0055`). `range(start, end, unit)` returning
   `['0时', '1时', ...]` is the whole of it; `dayRange(year, month)` needs the
   leap-year test.
3. **Write one `@Builder` that renders a single wheel** and takes an object
   literal - `{ range, selected, dateIndex, callback, option }`. Passing a single
   object literal to a global `@Builder` is what gives by-reference parameter
   passing and therefore dynamic updates.
4. **Set `canLoop(false)`** from `option.isLoop ?? false` - that is the whole of
   the document's step 1, and it is what makes 12月 not wrap to 1月.
5. **Hide, do not omit, the trailing wheels.** `.visibility(...)` keyed off
   `timeFormat` keeps the six-column layout stable and the indices constant.
6. **Recompute dependent ranges inside the handlers,** not in `build()`:
   changing the year refreshes `dayRange` (February), changing the month
   refreshes it too.
7. **Rebuild the `Date` from the scalars** in one place, carrying the time
   components across: `new Date(y, m - 1, d, h, min, s)`.
8. **Wrap the component for the dialog** with `new ComponentContent(uiContext,
   wrapBuilder(builderFn), params)` and open it with
   `getPromptAction().openCustomDialog(node, options)`.
9. **Keep the node handle, and dispose it** (`HW-02-0051`). `closeCustomDialog`
   returns a promise; chain `dispose()` onto it, and handle the mask-dismissal
   path too, because `autoCancel` defaults to `true`.
10. **Validate before mutating** in the host form (`HW-02-0053`), so a rejected
    date does not destroy the previously accepted one.

## Verified snippets

All snippets are from `Schedule.zip`. Corrected forms are marked.

**One wheel - `Schedule.zip#dateTimePicker/src/main/ets/pages/DateTimePicker.ets:29`** (corrected, see `HW-02-0059`)

```typescript
function isVisibility(dateIndex: DateIndex, timeFormat?: TimeFormat): Visibility {
  let format: DateIndex[] = [DateIndex.DAY, DateIndex.HOURS, DateIndex.MINUTES, DateIndex.SECOND];
  return (dateIndex <= format[timeFormat ?? TimeFormat.MINUTES]) ? Visibility.Visible : Visibility.None;
}

// FIX: the sample declares this as @Observed class PickerClass with @Track fields,
//      but never constructs one - every call site passes an object literal.
export interface PickerParam {
  range: string[];
  selected: number;
  dateIndex: DateIndex;
  option?: DateTimePickerOption;
  callback: (dateIndex: DateIndex, value: string | string[], index: number | number[]) => void;
}

@Builder
function customPicker(picker: PickerParam) {
  TextPicker({ range: picker.range, selected: picker.selected })
    .layoutWeight(Constants.LAYOUT_WEIGHT)
    .canLoop(picker.option?.isLoop ?? false)
    .visibility(isVisibility(picker.dateIndex, picker.option?.timeFormat))
    .onChange((value: string | string[], index: number | number[]) => {
      picker.callback(picker.dateIndex, value, index);
    });
}
```

**`isVisibility` is a lookup, not a chain of comparisons.** The array
`[DAY, HOURS, MINUTES, SECOND]` maps a `TimeFormat` to the rightmost `DateIndex`
that should still be shown, so adding a precision level is one array entry
rather than a new branch in six places.

**`Visibility.None` rather than a conditional `if`** keeps all six wheels in the
tree. That matters here because `layoutWeight(1)` divides the `Row` between the
*visible* children, and because the wheels' `selected` indices are computed
against a fixed column order.

**`canLoop` is a per-wheel attribute, not a picker option** - the same
`option.isLoop` is threaded to all six so they agree.

**The Date reducer - same file, line 99** (as shipped)

```typescript
updateSelectedDate = (dateIndex: DateIndex, value: string | string[], index: number | number[]) => {
  if (Array.isArray(value) || Array.isArray(index)) {
    return;                                    // multi-column TextPicker - not this layout
  }
  switch (dateIndex) {
    case DateIndex.YEAR:  this.dealSelectedYear(value, index);  break;
    case DateIndex.MONTH: this.dealSelectedMonth(value, index); break;
    case DateIndex.DAY:   this.dealSelectedDay(value, index);   break;
    default:              this.dealSelectedTime(dateIndex, value, index); break;
  }
};

dealSelectedYear(value: string, index: number) {
  this.currentYear = Number(value.slice(Constants.ZERO, -Constants.ONE));   // '2025年' -> 2025
  this.monthRange = DATE_TIME_RANGE.monthRange(this.currentYear);
  this.dayRange = DATE_TIME_RANGE.dayRange(this.currentYear, this.currentMonth);   // Feb 28/29
  this.updateDateTime();
}

dealSelectedDay(value: string, index: number) {
  this.currentDay = index + Constants.ONE;     // index, not the parsed label
  this.dayIndex = index;
  this.updateDateTime();
}

dealSelectedTime(dateIndex: DateIndex, value: string, index: number) {
  switch (dateIndex) {
    case DateIndex.HOURS:   this.selectDate.setHours(index);   break;
    case DateIndex.MINUTES: this.selectDate.setMinutes(index); break;
    case DateIndex.SECOND:  this.selectDate.setSeconds(index); break;
    default: break;
  }
}

updateSolarDate() {
  this.selectDate = new Date(
    this.currentYear, this.currentMonth - Constants.ONE, this.currentDay,
    this.selectDate.getHours(), this.selectDate.getMinutes(), this.selectDate.getSeconds()
  );
}
```

**`value.slice(0, -1)` strips the unit suffix.** The wheel shows `'2025年'`
because `TextPicker` renders strings; the number has to come back out. The day
handler avoids the parse entirely by using `index + 1`, which is safer - and is
the pattern the year and month handlers should follow, since `slice(0, -1)`
breaks the moment a locale uses a two-character unit.

**The time wheels mutate `selectDate` in place** while the date wheels replace
it. Both are observed - the state-management guidance lists `setHours`,
`setMinutes` and `setSeconds` among the `Date` mutators that trigger an update -
but the in-place path deliberately skips `updateSolarDate`, because rebuilding
from `currentYear/Month/Day` would be a no-op for a time-only change.

**`Array.isArray` guard first.** `TextPicker.onChange` is typed for both
single-column and multi-column pickers; this layout is six single-column
pickers, so an array argument means something has been reconfigured and the
handler bails rather than guessing.

**Range recomputation - `Schedule.zip#dateTimePicker/src/main/ets/utils/DateTimeSolar.ets:44`** (as shipped)

```typescript
public dayRange(year: number, month: number): string[] {
  let mapDays = new Map<number, number>([
    [1, 31], [2, this.isLeapYear(year) ? 29 : 28], [3, 31], [4, 30], [5, 31], [6, 30],
    [7, 31], [8, 31], [9, 30], [10, 31], [11, 30], [12, 31]
  ]);
  return this.range(Constants.ONE, mapDays.get(month) ?? Constants.CURRENT_DAY, '日');
}

// DateTimeBase
public isLeapYear(year: number): boolean {
  if ((year % 400 === 0) || (year % 4 === 0 && year % 100 !== 0)) {
    return true;
  }
  return false;
}
```

**The leap-year rule is the full one** - divisible by 400, or by 4 but not by
100 - which matters for 1900 and 2100, both inside the sample's 1980-2050 range
in the 2100 direction. The `?? 30` fallback covers a month index outside 1..12.

Rebuilding the `Map` on every call is wasteful but harmless at this scale; a
module-level array would be the cheap fix.

**The dialog - `Schedule.zip#dateTimePicker/src/main/ets/dialog/DateTimePickerDialog.ets:133`** (corrected, see `HW-02-0051`)

```typescript
export function dateTimePickerDialog(uiContext: UIContext, param: DateTimePickerParam,
  txt: ResourceStr, dialogOptions?: promptAction.BaseDialogOptions) {
  let now: number = Date.now();
  let contentNode = new ComponentContent(uiContext, wrapBuilder(dateTimePickerDialogBuilder), {
    node: now,
    param: param,
    txt: txt
  } as DateTimePickerDialogParam);
  uiContext.getPromptAction().openCustomDialog(contentNode, dialogOptions);
  mapContext.set(now, { context: uiContext, content: contentNode });
}

closeDialog() {
  if (!mapContext.has(this.node)) {
    return;
  }
  const config = mapContext.get(this.node)!;
  config.context.getPromptAction().closeCustomDialog(config.content)
    .then(() => {
      config.content.dispose();          // FIX: the sample never disposes
      mapContext.delete(this.node);
    })
    .catch((err: BusinessError) => {
      hilog.error(0x0000, 'DateTimePicker', 'closeCustomDialog failed: %{public}s', err.message);
    });
}
```

**`ComponentContent` + `openCustomDialog` is how a HAR shows a dialog** without
owning a `@CustomDialog` struct or a `CustomDialogController` field - both of
which must live inside an `@Component`. `wrapBuilder` turns the global `@Builder`
into a value the `ComponentContent` constructor accepts, and the third argument
is that builder's parameter object.

**The node handle must outlive the call.** `openCustomDialog` does not return
anything you can close later, so the caller has to keep `contentNode` - here in
a module `Map` keyed by the timestamp that was also passed into the component,
so the component can find its own node.

**`dispose()` is required, not optional.** The reference is explicit: "If the
frontend object ComponentContent cannot be released, memory leaks may occur. To
avoid this, be sure to call dispose() on the ComponentContent object when you no
longer need it." The sample never calls it.

**And `autoCancel` defaults to `true`,** so a tap on the mask closes the dialog
without any button handler running - `closeDialog` never fires, and the map
entry and its node stay forever. Pass `onWillDismiss` (or `onDidDisappear`) in
the `BaseDialogOptions` to route that path through the same cleanup.

**Opening it from the form - `Schedule.zip#entry/src/main/ets/pages/NewSchedule.ets:47`** (as shipped)

```typescript
openDateTimePicker = (isStart: boolean, onConfirm: (selectedDate: Date) => void) => {
  dateTimePickerDialog(this.getUIContext(), {
    data: {
      startYear: Constants.START_YEAR,          // 1980
      endYear: Constants.END_YEAR,              // 2050
      selectedDate: isStart ? this.startDate : this.endDate,
      resultCb: (selectedDate: Date) => {
        onConfirm(selectedDate);
        this.updateSelectData(selectedDate);
      },
    },
    option: { timeFormat: TimeFormat.MINUTES },
    style: {
      backgroundColor: Color.White,
      borderRadius: Constants.BORDER_RADIUS,
      borderColor: Color.Orange,
      borderWidth: Constants.BORDER_WIDTH,
      margin: Constants.MARGIN_ONE
    }
  } as DateTimePickerParam, this.txt, {
    alignment: DialogAlignment.Center,
    offset: ({ dx: Constants.DX, dy: Constants.DY } as GeneratedObjectLiteralInterface_1),
  });
};
```

**One opener, two call sites, distinguished by a boolean and a callback.** The
start row and the end row differ only in which `Date` seeds the picker and what
the confirm handler does with the result - so the row `onClick` sets the dialog
title into `@State txt` and passes its own closure.

`as DateTimePickerParam` on an object literal is the ArkTS idiom for satisfying
an interface parameter; ArkTS will not infer it.

**Validation in the host - same file, line 180** (corrected, see `HW-02-0053`)

```typescript
.onClick(() => {
  this.txt = $r('app.string.titleStart');
  this.openDateTimePicker(true, (selectedDate: Date) => {
    // FIX: the sample assigns this.startDate before testing, and blanks the
    //      previously accepted value in the else branch.
    if (!this.isEndNull && selectedDate.getTime() >= this.endDate.getTime()) {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.selectagain') });
      return;                                    // keep whatever was already valid
    }
    this.startDate = selectedDate;
    this.selectDate = selectedDate;
    this.startMd = (selectedDate.getMonth() + 1) + '月' + selectedDate.getDate() + '日 ';
    this.startTime = pad(selectedDate.getHours()) + ':' + pad(selectedDate.getMinutes());  // HW-02-0060
    this.isStartNull = false;
  });
});
```

The shipped version writes `this.startDate = selectedDate` first and then, on
rejection, clears `startMd`/`startTime` and sets `isStartNull = true` - so the
model holds the rejected date while the flag claims nothing is set, and the
user's earlier valid choice is gone.

**The immersive setup - `Schedule.zip#entry/src/main/ets/entryability/EntryAbility.ets:30`** (corrected, see `HW-02-0056`)

```typescript
async onWindowStageCreate(windowStage: window.WindowStage): Promise<void> {
  this.context.getApplicationContext().setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT);
  const WIN = await windowStage.getMainWindow();
  await WIN.setWindowLayoutFullScreen(true);          // FIX: sample drops this promise
  AppStorage.setOrCreate('topHeight',
    WIN.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM).topRect.height);
  AppStorage.setOrCreate('bottomHeight',
    WIN.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR).bottomRect.height);
  WIN.on('avoidAreaChange', this.avoidAreaCallback);  // FIX: sample never subscribes
  windowStage.loadContent('pages/NewSchedule', (err) => { /* ... */ });
}
```

Worth copying from the sample: it uses `await windowStage.getMainWindow()`
rather than `getMainWindowSync()`, and it takes the **right** avoid-area type
for each edge - `TYPE_SYSTEM` for the top, `TYPE_NAVIGATION_INDICATOR` for the
bottom - which `LIFE-03` gets wrong.

## Permissions & config

None. Neither `module.json5` declares a `requestPermissions` block.

`Schedule.zip#entry/src/main/module.json5`

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "deliveryWithInstall": true,
    "installationFree": false,
    "pages": "$profile:main_pages",
    "abilities": [{
      "name": "EntryAbility",
      "srcEntry": "./ets/entryability/EntryAbility.ets",
      "exported": true,
      "skills": [{ "entities": ["entity.system.home"], "actions": ["action.system.home"] }]
    }],
    "extensionAbilities": [{
      "name": "EntryBackupAbility",
      "srcEntry": "./ets/entrybackupability/EntryBackupAbility.ets",
      "type": "backup",
      "exported": false,
      "metadata": [{ "name": "ohos.extension.backup", "resource": "$profile:backup_config" }]
    }]
  }
}
```

The HAR is wired by package name, and the two spellings agree - unlike
`LIFE-03`:

```json5
// entry/oh-package.json5
"dependencies": { "@ohos/datetimepicker": "file:../dateTimePicker" }

// dateTimePicker/oh-package.json5
"name": "@ohos/datetimepicker",
"packageType": "har",
"main": "Index.ets"
```

`dateTimePicker/Index.ets` re-exports all three public surfaces, so the entry
imports everything from the package name:

```typescript
export * from './src/main/ets/pages/DateTimePicker';
export * from './src/main/ets/dialog/DateTimePickerDialog';
export * from './src/main/ets/interface/DateTimePickerInterface';
```

Root `build-profile.json5` targets `6.0.0(20)`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 49-51).
- The year span is 1980-2050 in the host (`entry/common/Constants.ets`), but it
  is stored in a module singleton, so **two pickers with different spans cannot
  coexist** (`HW-02-0055`).
- The wheel labels carry Chinese unit suffixes (年月日时分秒) built into the
  range functions, and the year and month handlers parse them back with
  `slice(0, -1)`. The component is not localisable as written.
- `TimeFormat` trims only from the right. There is no date-less time picker.
- `DateTimePickerData.selectedDate` is optional, but the dialog asserts it with
  `this.param.data.selectedDate!` - omit it and `selectDate` is `undefined` at
  runtime.
- The dialog's node key is `Date.now()`; two dialogs opened in the same
  millisecond would share a key.
- The 保存 (save) button on the form has no `onClick`; the sample ends at
  selection.
- Light mode is forced application-wide.

## Pitfalls

- **`HW-02-0051` - the `ComponentContent` is never `dispose()`d,** and because
  `autoCancel` defaults to `true`, a mask tap closes the dialog without running
  `closeDialog()` at all. The module-scope `mapContext` then holds the
  `UIContext` and the node for the life of the process. Chain
  `dispose()` onto `closeCustomDialog`'s promise and add an `onWillDismiss`.
- **`HW-02-0052` - `controller1` is bound to both `TextInput`s** while
  `controller2` is declared and unused. A `TextInputController` drives one field.
- **`HW-02-0053` - an out-of-order date wipes the previously valid one.** The
  handler assigns before it validates and blanks the display strings on
  rejection, leaving `startDate` holding the rejected value while
  `isStartNull` says nothing is set.
- **`HW-02-0054` - the result callback travels inside a `@Prop` object.** The
  `@Prop` guidance says the deep copy loses "all types, except primitive types,
  Map, Set, Date, and Array". Pass callbacks as undecorated members.
- **`HW-02-0055` - the year range is a module singleton.** `setYearRange` is
  called from every picker's `aboutToAppear` and read back during render, so a
  second picker silently reconfigures the first. This is a HAR: two instances
  are the expected case.
- **`HW-02-0056` - `setWindowLayoutFullScreen` is not awaited** in a method that
  awaits `getMainWindow` two lines earlier, and the avoid areas are read
  immediately after. There is also no `avoidAreaChange` subscription.
- **`HW-02-0057` - the document's snippet does not compile.** It prints
  `dateTimePickerDialog()` with no parameters and three unbound identifiers, and
  omits `ComponentContent`, `wrapBuilder` and the node bookkeeping - the three
  things that make the `PromptAction` route work.
- **`HW-02-0058` - six `@Provide` variables with no `@Consume` anywhere.** They
  should be `@State`.
- **`HW-02-0059` - `PickerClass` carries `@Observed` and `@Track` but is never
  constructed.** All six call sites pass object literals, so both decorators are
  inert; the dynamic updates come from the single-object-literal `@Builder`
  rule, not from `@Observed`.
- **`HW-02-0060` - minutes are zero-padded and hours are not,** in the same
  expression. Mornings read `9:05`.
- **Do not use `if` to hide the trailing wheels.** `Visibility.None` keeps the
  column count and the `layoutWeight` division stable.
- **Do not parse the day label.** `index + 1` is exact; `slice(0, -1)` on
  `'15日'` only works while the unit is one character.
- **Do not forget that `TextPicker.onChange` can hand you arrays.** Guard with
  `Array.isArray` before treating `value`/`index` as scalars.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textpicker.md` - `TextPicker`, `canLoop`, `selected`, `onChange` and its `string | string[]` / `number | number[]` parameters
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textpicker
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-promptaction.md` - `openCustomDialog(ComponentContent, BaseDialogOptions)`, `closeCustomDialog`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-promptaction
- `documentation/harmonyos-references/02_application-framework/js-apis-promptaction.md` - `BaseDialogOptions`, `autoCancel` (default `true`), `alignment`, `offset`, `onWillDismiss`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-promptaction
- `documentation/harmonyos-references/02_application-framework/js-apis-arkui-componentcontent.md` - `ComponentContent`, `wrapBuilder`, and the `dispose()` leak warning
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-arkui-componentcontent
- `documentation/harmonyos-guides/03_application-framework/arkts-prop.md` - what `@Prop`'s deep copy preserves, and the observed `Date` mutators
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-prop
- `documentation/harmonyos-guides/03_application-framework/arkts-builder.md` - by-reference parameter passing for a global `@Builder` with one object literal
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-builder
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `getMainWindow`, `setWindowLayoutFullScreen`, `getWindowAvoidArea`, `on`/`off('avoidAreaChange')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-guides/01_getting-started/har-package.md` - HAR constraints and package naming
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/har-package
- `LIFE-03` - the same industry's other HAR-plus-entry sample, where the avoid-area types are taken wrongly
- `LIFE-01` - the industry shell this page would sit in
