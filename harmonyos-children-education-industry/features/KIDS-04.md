---
id: KIDS-04
title: Screen-time limit - a timed lock overlay with a forced rest countdown
industry: 08_children_education
doc: huawei_industry_tree/08_children_education/docs/04_control_usage_time.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/control_usage_time-0000002277224065
sample: huawei_industry_tree/08_children_education/downloads/ControlUsageTime.zip
kits: ["@kit.ArkUI", "@kit.ArkData", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: [setTimeout, clearTimeout, TextTimer, TextTimerController, isCountDown, onTimer, onVisibleAreaChange, "preferences.getPreferencesSync", getSync, put, flush, deleteSync, "@Provide", "@Consume", "@StorageLink", NavPathStack, NavDestination, pushPathByName, TextPicker, RelativeContainer, Stack]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-08-0023, HW-08-0024, HW-08-0025, HW-08-0026, HW-08-0027, HW-08-0028, HW-08-0029, HW-08-0030, HW-08-0031, HW-08-0032, HW-08-0033, HW-08-0034, HW-08-0120]
status: verified-with-fixes
---

## When to use

Load this card for **an app that limits how long it may be used and then makes
the user stop** - a children's app with an anti-addiction requirement, a
homework timer, a break reminder, any product that has to enforce a pause
rather than suggest one.

The design is worth taking; almost none of the arithmetic is.

- **The lock is an overlay in the same page, not a route.** A `Stack` holds
  the content and, when `isShow` goes true, a translucent `Column` on top of
  it. Nothing navigates, so there is no back gesture to escape through.
- **Two timers with two different jobs.** A `setTimeout` fires once, at the
  moment the allowance runs out, and puts the overlay up; a `TextTimer` in
  countdown mode then shows the child how long the rest has left. Using a
  repeating tick for the first job would burn cycles for nothing.
- **The deadline is stored, not the remaining time.** `futureRest` is an
  absolute moment, so the state can in principle be reconstructed after a
  restart rather than being lost with the process.

**This is the most broken sample in the corpus so far - twelve findings, one
of them a blocker.** A limit set as "40 minutes" locks after forty seconds,
and nothing written to preferences can ever be read back. Take the shape from
this card and the numbers from nowhere.

## Feature checklist

- A content page (a mock news article) that the child uses normally.
- A settings page with two pickers: duration of use, duration of rest.
- Saving computes the moment the rest will end and stores it.
- When the allowance expires the content is covered by a lock panel.
- The panel shows a countdown of the remaining rest.
- When the rest ends, a link on the panel becomes active and dismisses it.

## Architecture

One `entry` module, one page, one destination, two components, three utils.

```
entry/src/main/ets
├── components
│   ├── Lock.ets           the lock panel and its (gated) dismiss link
│   └── RestTimer.ets      the TextTimer countdown, plus a module-level controller
├── entryability/EntryAbility.ets
├── entrybackupability/
├── pages
│   ├── MainPage.ets       the content, the overlay, the setTimeout, all the state
│   └── SetTimePage.ets    the two pickers and the save
└── utils
    ├── GetTimeUtils.ets       seconds since midnight
    ├── ListOptionUtils.ets    the three preference key names
    └── PreferenceUtils.ets    getStore / save / delete / get
```

The documented tree matches the zip.

**All shared state is `@Provide` on `MainPage` and `@Consume` everywhere
else** - `isShow`, `isClick`, `useTime`, `restTime`, `futureRest`,
`returnTimerId`, `restReal`. Seven providers is a lot, but it is the right
call here: `Lock` and `RestTimer` are nested two levels down inside a
`@Builder`, so passing parameters would mean threading them through the
builder by hand.

**The time model, as intended:**

```
save ──> futureRest = now + useTime + restTime          (an absolute moment)
              │
setTime() ──> setTimeout(lock, futureRest - restTime - now)   (fires when use runs out)
              │
           lock ──> restReal = restTime ──> TextTimer counts it down
              │
        countdown ends ──> isClick = true ──> the panel can be dismissed
```

**On relaunch `aboutToAppear` reconstructs the state** from the stored
`futureRest`: still more than a rest away, carry on; inside the rest window,
re-lock with the remaining time. That is the correct instinct for an
anti-addiction control - and it does not work, because the read never returns
what was written (`HW-08-0024`).

## Implementation steps

1. **Store the durations in the same unit the clock uses** (`HW-08-0023`).
2. **Read preferences with a default of the stored type** (`HW-08-0024`).
3. **Compute an absolute deadline** - and make it an instant, not a time of
   day (`HW-08-0029`).
4. **Arm the `setTimeout` when the page becomes visible** and clear it in
   `aboutToDisappear` and when it goes away.
5. **Raise the overlay from a flag inside a `Stack`**, so no navigation is
   involved.
6. **Count the rest down with `TextTimer`**, and release the lock on a
   threshold rather than an exact tick (`HW-08-0026`).
7. **Recompute on resume**, since the tick does not run in the background.

## Verified snippets

All snippets are from `ControlUsageTime.zip`. Corrected forms are marked.

**The overlay — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
@Provide isShow: boolean = false;
@Provide isClick: boolean = false;
@Provide returnTimerId: number = 0;

@Builder
myBuilder() {
  Column() {
    RestTimer().margin({ top: this.uiContext.px2vp(this.topRectHeight) });
    Lock().margin({ top: 165 });
  }
  .width('100%').height('100%')
  .backgroundColor('#55000000');        // translucent scrim over the content
}

build() {
  Navigation(this.pageInfos) {
    Stack() {
      Column() { /* the article, and the settings button */ }
        .onVisibleAreaChange([0.0, 1.0], (isVisible: boolean, currentRatio: number) => {
          if (isVisible && currentRatio >= 1.0) {
            this.setTime();                     // arm when fully shown
          }
          if (!isVisible && currentRatio <= 0.0) {
            clearTimeout(this.returnTimerId);   // disarm when fully hidden
          }
        });

      if (this.isShow) {
        this.myBuilder();                       // the lock, on top, same page
      }
    }
  }
  .hideTitleBar(true)
  .hideBackButton(true)
  .hideToolBar(true);
}
```

**A full-size `Column` with a background is what makes the lock a lock.** It
is hit-testable, so it swallows every touch aimed at the content and at the
settings button underneath. Nothing needs to be disabled individually.

**`onVisibleAreaChange([0.0, 1.0], ...)` is the right hook for arming a
timer.** The two ratios mean the callback fires only at fully-visible and
fully-hidden, so the timer is armed once when the page is really in front of
the child and cleared when it is not - covering navigation to settings as well
as backgrounding. The sample's own comment says exactly this.

**The timer is cleared in three places** - here, in `aboutToDisappear`, and
before pushing the settings page. That is the discipline the document asks for
in step 1, and this sample keeps it.

**Arming the lock — same file** (corrected, see `HW-08-0023`)

```typescript
setTime(): void {
  this.currentTime = GetDateTime.getDateStringWithTimeStamp();
  if (this.useTime !== 0) {
    if ((this.futureRest - this.currentTime) > this.restTime) {
      this.returnTimerId = setTimeout(() => {
        this.restReal = this.restTime;
        this.isShow = true;
      }, (this.futureRest - this.restTime - this.currentTime) * 1000);
    }
  }
}

aboutToDisappear() {
  clearTimeout(this.returnTimerId);
}
```

**`futureRest - restTime` is the moment use runs out**, since `futureRest` is
the end of the *rest*. The subtraction is the only place that relationship is
written down, which is why the two must be stored in the same unit - and the
picker supplies minutes while this line assumes seconds.

**`aboutToAppear` is the restart path — same file** (as shipped)

```typescript
aboutToAppear() {
  this.currentTime = GetDateTime.getDateStringWithTimeStamp();
  if ((this.futureRest - this.currentTime) > this.restTime) {
    this.isShow = false;                                  // outside the window: free
    this.restReal = this.restTime;
  } else if ((this.futureRest - this.currentTime) > 0) {
    this.restReal = this.futureRest - this.currentTime;   // mid-rest: re-lock, partial
    this.isShow = true;
  }
}
```

**Reconstructing the lock from a stored deadline is the correct design** and
the one thing that separates a real anti-addiction control from a decoration:
force-quitting must not clear the limit. The two branches are right; the value
they read is not.

**The rest countdown — `entry/src/main/ets/components/RestTimer.ets`** (corrected, see `HW-08-0026`)

```typescript
export const textTimerController: TextTimerController = new TextTimerController();
export const FORMAT: string = 'mm:ss';

TextTimer({
  isCountDown: true,
  count: (this.restReal) * 1000,       // count is in milliseconds
  controller: textTimerController
})
  .format(FORMAT)
  .onTimer((utc: number, elapsedTime: number) => {
    // FIX: the sample tests `elapsedTime === this.restReal`. onTimer does not
    // fire in the background or with the screen locked, so a missed tick
    // leaves the panel undismissable.
    if (elapsedTime >= this.restReal) {
      this.isClick = true;
    }
  })
  .onAppear(() => {
    textTimerController.start();
  });
```

**`count` is milliseconds, `elapsedTime` is not.** The reference defines
`elapsedTime` as being "in the minimum unit of the format" - with `'mm:ss'`
that is seconds. Two different units on the same component, three lines apart,
and both have to be right.

**`isCountDown: true` plus `count` set at construction** is the whole
countdown. `count` is only the *initial* value, so re-arming means recreating
the component - which the `if (this.isShow)` around the builder does for free.

**Saving the settings — `entry/src/main/ets/pages/SetTimePage.ets`** (corrected, see `HW-08-0025` and `HW-08-0030`)

```typescript
private useArr: string[] = ['20', '40', '60', '80'];
private restArr: string[] = ['10', '20', '30', '40'];

TextPicker({ range: this.useArr, selected: 1 })
  .canLoop(false)
  .onChange((value: string | string[]) => {
    this.useText = value + '';
    this.setUse = true;
    // FIX: the sample looks the value up in a `checkedList` Record whose
    // '20' entry is 10 and whose '80' entry is 90.
    this.useTime = Number.parseInt(value + '', 10) * 60;   // FIX: minutes -> seconds
  });

Button($r('app.string.Saved'))
  .enabled((this.setRest && this.setUse) ? true : false)
  .onClick(async () => {
    this.now = GetDateTime.getDateStringWithTimeStamp();
    this.futureRest = this.now + this.useTime + this.restTime;
    // FIX: the sample pops first and never awaits the save
    await PreferencesClass.savePreferenceInfo(this.context, LstOptionClass.futureRestInfo,
      PreferencesClass.defaultStore, this.futureRest);
    this.pageInfos.pop();
  });
```

**`.enabled((this.setRest && this.setUse))` is a real guard** - the deadline
cannot be written until both pickers have been touched, so a half-configured
limit is impossible. The `onVisibleAreaChange` handler on the same page
computes the same deadline with no such guard, which is `HW-08-0027`.

**Reading a preference — `entry/src/main/ets/utils/PreferenceUtils.ets`** (corrected, see `HW-08-0024` and `HW-08-0031`)

```typescript
import { preferences } from '@kit.ArkData';
import { common } from '@kit.AbilityKit';   // FIX: the sample imports Context from
                                            // '@ohos.abilityAccessCtrl'

static getStore(content: common.Context, storeName: string) {
  return preferences.getPreferencesSync(content, {
    name: storeName || PreferencesClass.defaultStore
  });
}

static getNumber(content: common.Context, storeName: string, storeKey: string): number {
  // FIX: the sample calls getSync(storeKey, 'No info') for keys holding numbers.
  // "If the value is null or is not of the default value type, defValue is returned."
  return PreferencesClass.getStore(content, storeName).getSync(storeKey, 0) as number;
}
```

**`getSync` matches on the default's type, not just on absence.** That one
sentence in the reference is what makes the shipped version return `'No info'`
for a number that is sitting in the store - and it is why every read in this
sample fails silently rather than loudly.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`; the limit is
enforced entirely inside the app.

The routing profile is named `route_map.json` here (other samples in this
corpus use `router_map.json`); the key inside it is `routerMap` either way:

```json
{
  "routerMap": [
    {
      "name": "SetTimePage",
      "pageSourceFile": "src/main/ets/pages/SetTimePage.ets",
      "buildFunction": "setTimePageBuilder",
      "data": { "description": "this is SetTimePage" }
    }
  ]
}
```

`main_pages.json` lists `pages/MainPage` alone, and `SetTimePage` is a
`NavDestination` reached through `pushPathByName` - the layering `KIDS-03`
gets wrong and this sample gets right.

**This is the first sample in the industry with real localisation**:
`resources/en_US/element/string.json` and `resources/zh_CN/element/string.json`
both exist, every visible label goes through `$r`, and the durations are
formatted with a parameterised string, `$r('app.string.mint', this.useText)`
against `"%s minutes"`. `KIDS-02` and `KIDS-03` should have done this and did
not.

`EntryAbility.onCreate` pins light mode with
`setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT)`, so
`resources/dark` is dead.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The lock only exists while this page is alive.** It is an overlay inside
  `MainPage`, so it covers this app and nothing else; it is not a device-level
  control and it does not survive being force-quit (`HW-08-0024`).
- **The clock is the device's local time**, so moving the system clock forward
  clears the limit. Any anti-addiction feature that must resist a motivated
  child needs a source the child cannot set.
- **The durations are a fixed menu** - four use values, four rest values, both
  hardcoded in the page. There is no free entry and no per-day schedule.
- **The rest cannot be ended early and the lock cannot be ended by an adult.**
  There is no parent override, so the only way past an active lock is to wait
  or to reinstall. Pairing this with `KIDS-02`'s arithmetic gate is the
  obvious combination and neither sample makes it.
- **After the rest the child is sent to the settings page**, not back to the
  content: the dismiss link both clears the lock and pushes `SetTimePage`.
- `TextPicker` is always constructed with `selected: 1`, so reopening settings
  shows the second option rather than the saved one.

## Pitfalls

- **`HW-08-0023` — every duration is labelled in minutes and computed in
  seconds.** `"%s minutes"` in the resource file, `hours*3600 + minutes*60 +
  seconds` in the clock, and no conversion anywhere between them. A 40-minute
  allowance locks after 40 seconds and the 20-minute rest lasts 20. This is
  the blocker: the feature does not do the thing its own UI describes.
- **`HW-08-0024` — every preference is read with the string default
  `'No info'` although all stored values are numbers.** The reference says a
  default-type mismatch returns the default, so nothing saved is ever read
  back, `as number` hides it, and the lock does not survive a restart - which
  for an anti-addiction control is the whole point. The sample's own
  first-launch check tests against `'No info'`, which is how the defect is
  visible from inside the code.
- **`HW-08-0025` — the picker-to-number table is wrong in two of six
  entries:** `'20'` maps to 10 and `'80'` maps to 90, while the label keeps
  showing what was picked.
- **`HW-08-0026` — the unlock hangs on `elapsedTime === restReal`,** and
  `onTimer` is documented not to fire in the background or with the screen
  locked - exactly where a device spends a forced rest. Miss the tick and the
  panel can never be dismissed.
- **`HW-08-0027` — closing the settings page rewrites the deadline in memory
  without persisting it,** and does so even when nothing was edited, so merely
  opening and closing settings grants the child another full use period.
- **`HW-08-0029` — the clock is seconds since midnight,** so a period crossing
  midnight either schedules a lock a day out or drops it entirely. Evening use
  is the case the feature exists for.
- **`HW-08-0030` — `savePreferenceInfo` is async and awaited at none of its
  four call sites,** and the save button pops the page before calling it.
- **`HW-08-0028` — `EntryAbility` computes a start page through a first-launch
  branch, then loads a hardcoded one;** the discarded default names
  `pages/RestPage`, which does not exist.
- **`HW-08-0031` — `Context` is imported from `@ohos.abilityAccessCtrl`,** the
  permission-checking module, in a sample that requests no permissions.
- **`HW-08-0032` — the `avoidAreaChange` listener is never released.** Unlike
  the neighbouring samples the keys here are genuinely used, so only the
  teardown is missing.
- **`HW-08-0033` — the preference helpers take the store name and the key in
  opposite orders,** and the (unused) delete helper never flushes.
- **`HW-08-0034` — the document's code block lists bare identifiers**, showing
  `setTimeout()` with no callback and `TextTimer.onTimer()` as a static call.

## References

- `documentation/harmonyos-references/05_common-capabilities/js-apis-timer.md` - `setTimeout`, `clearTimeout`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-timer
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-texttimer.md` - `isCountDown`, `count` in ms, `onTimer` and its units and its background behaviour
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-texttimer
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` - `getSync` and its default-type rule, `put`, `flush`, `deleteSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` - the module `Context` was wrongly taken from, and the import its own examples use
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-references/02_application-framework/ts-universal-component-visible-area-change-event.md` - `onVisibleAreaChange` and its ratio list
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-component-visible-area-change-event
- `documentation/harmonyos-guides/03_application-framework/arkts-provide-and-consume.md` - the seven-provider state tree
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-provide-and-consume
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navdestination.md` - the settings destination
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navdestination
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `on`/`off('avoidAreaChange')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `KIDS-02` - the arithmetic parent gate, the natural override this sample lacks
- `KIDS-11` - the other parental-monitoring sample, which uses the same `preferences` helper shape
