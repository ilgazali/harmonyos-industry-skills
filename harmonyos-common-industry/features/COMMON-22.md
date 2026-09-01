---
id: COMMON-22
title: Page tracing points - count page visits and dwell time with the ArkUI observer, without touching page code
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/22_page_tracing_point.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/page_tracing_point-0000002283641138
sample: huawei_industry_tree/19_common_technical_solutions/downloads/PageTracingPoint.zip
kits: ["@kit.ArkUI", "@kit.ArkTS", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: ["UIContext.getUIObserver", "UIObserver.on('navDestinationSwitch')", "UIObserver.on('navDestinationUpdate')", "UIObserver.on('routerPageUpdate')", "UIObserver.off", "uiObserver.NavDestinationSwitchInfo", "uiObserver.NavDestinationInfo", "uiObserver.NavDestinationState", "uiObserver.RouterPageInfo", "uiObserver.RouterPageState", ArrayList, Navigation, NavPathStack, NavDestination, routerMap, "window.getWindowAvoidArea", "window.on('avoidAreaChange')", "AvoidAreaType.TYPE_SYSTEM"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0068, HW-19-0069, HW-19-0070, HW-19-0071, HW-19-0072, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when the product needs **analytics on page usage - how often each
page is opened and how long the user stays - without instrumenting the pages
themselves**. The document's framing: 通过页面埋点监听用户对各个页面的访问频次、浏览
时间及页面操作习惯，从而为用户提供个性化推荐设置 ("track how often the user visits
each page, how long they browse and their interaction habits, in order to provide
personalised recommendations").

The mechanism is ArkUI's 无感监听 ("transparent observation") - `@ohos.arkui.observer`
- registered once at ability level. No page in the sample contains a single line
of tracking code.

## Feature checklist

The application must:

- Register the observer once, from `onWindowStageCreate`, and release it in
  `onWindowStageDestroy`.
- Subscribe to all three page-lifecycle sources: `navDestinationUpdate` (per
  NavDestination state), `routerPageUpdate` (per Router page state), and
  `navDestinationSwitch` (transitions to and from the Navigation home content).
- Increment a visit count on the appear state.
- Start a stopwatch on the shown state and accumulate the elapsed time on the
  hidden state.
- Drop per-page bookkeeping on the disappear state.
- Key everything by a stable page identity that survives across visits.
- Render the accumulated counts and durations on a report page, with correct
  unit thresholds (HW-19-0071, HW-19-0072).

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup and insets; obtains the `UIObserver` and calls `initPageTracing`; calls `unsubscribe` on stage destroy |
| `util/PageTracingManager.ets` | the singleton tracer: three maps, one page stack, three listener registrations |
| `util/CustomInfoUtil.ets` | normalises `NavDestinationInfo` and `RouterPageInfo` into one `CustomPageInfo`, and builds the identity key |
| `model/PageTracingModel.ets` | `CustomPageInfo` and the `NEWS` list that drives both the home page and the report |
| `constant/CommonConstant.ets` | thresholds and unit labels for the report |
| `pages/HomePage.ets` | the `Navigation` host - note `.id('TracingNavigation')`, which becomes part of every key |
| `pages/CurAffairPage.ets`, `EntertainmentPage.ets`, `FashionPage.ets` | the tracked content pages |
| `pages/TracingResultPage.ets` | the report |
| `views/NewsView.ets`, `views/OtherView.ets`, `component/CommonComponent.ets` | shared UI |

**State held by the tracer.** Three `Map<string, number>` plus one stack:

| Field | Meaning |
| --- | --- |
| `accessCountMap` | visits per page, incremented on appear |
| `accessTimeMap` | accumulated milliseconds per page |
| `startTimeMap` | the open stopwatch per page, set on shown |
| `pageStack` | `ArrayList<CustomPageInfo>` of live pages, used to resolve the Navigation home page and to decide whether a hidden Router page really lost focus |

**The identity key** is `` `${info.name}-${info.uniqueId}` ``, where `uniqueId`
is the `navigationId` for a NavDestination and the literal `'Router'` for a
Router page. That is why the report looks its rows up as
`news.pageName + '-TracingNavigation'` - `TracingNavigation` is the `id` set on
the `Navigation` component in `HomePage`. Rename that `id` and every stored
statistic becomes unreachable.

**Why three listeners rather than one.** `navDestinationUpdate` and
`routerPageUpdate` cover the two routing frameworks; `navDestinationSwitch` is
needed because the Navigation *home content* (`navBar`) is not a NavDestination
and so never appears in the first stream - the switch event is the only signal
that the user entered or left it.

**Event to bookkeeping mapping:**

| Event state | Action |
| --- | --- |
| `ON_APPEAR` / `ABOUT_TO_APPEAR` | count++, push onto `pageStack` |
| `ON_SHOWN` / `ON_PAGE_SHOW` | `startTimeMap[key] = Date.now()` |
| `ON_HIDDEN` / `ON_PAGE_HIDE` | `accessTimeMap[key] += Date.now() - startTimeMap[key]` |
| `ON_DISAPPEAR` / `ABOUT_TO_DISAPPEAR` | delete the stopwatch, remove from `pageStack` |
| `navDestinationSwitch` from/to `navBar` | same stopwatch accounting for `pageStack[0]` |

## Implementation steps

1. **Make the tracer a singleton** so the maps survive navigation.
2. **Initialise once, from the ability**:
   `PageTracingManager.getInstance().initPageTracing(uiContext)` inside the
   `loadContent` callback, where a `UIContext` is available.
3. **Get the observer from the UI context**: `uiContext.getUIObserver()`.
4. **Register the three listeners** and normalise both info shapes into a single
   `CustomPageInfo` so one set of maps serves both routing frameworks.
5. **Guard every stack access.** `pageStack[0]` is undefined before the first
   appear and after the last disappear (HW-19-0068).
6. **Do not remove from the stack while traversing it** (HW-19-0069).
7. **Release the subscriptions** in `onWindowStageDestroy` via
   `observer.off('navDestinationUpdate' | 'navDestinationSwitch' |
   'routerPageUpdate')`.
8. **Build the report** from `getAccessCountMap()` / `getAccessTimeMap()`,
   formatting with the largest matching unit first (HW-19-0071) and correct
   magnitude constants (HW-19-0072).

## Verified snippets

All snippets below come from the sample project, not from the document.

### Initialising the observer

`PageTracingPoint.zip#PageTracingPoint/entry/src/main/ets/util/PageTracingManager.ets`

```ts
import { UIObserver, uiObserver } from '@kit.ArkUI';
import { CustomPageInfo } from '../model/PageTracingModel';
import { ArrayList } from '@kit.ArkTS';
import { CustomInfoUtil } from './CustomInfoUtil';

export class PageTracingManager {
  private static instance: PageTracingManager | undefined;
  private accessCountMap: Map<string, number> = new Map();
  private accessTimeMap: Map<string, number> = new Map();
  private startTimeMap: Map<string, number> = new Map();
  private pageStack: ArrayList<CustomPageInfo> = new ArrayList();

  public static getInstance() {
    if (PageTracingManager.instance === undefined) {
      PageTracingManager.instance = new PageTracingManager();
    }
    return PageTracingManager.instance;
  }

  /**
   * 初始化无感监听
   */
  public initPageTracing(uiContext: UIContext) {
    let uiObserver = uiContext.getUIObserver();
    this.registerSwitchListener(uiObserver);
    this.registerNavUpdateListener(uiObserver);
    this.registerRouterUpdateListener(uiObserver);
  }

  public unsubscribe(uiObserver: UIObserver) {
    if (uiObserver) {
      uiObserver.off('navDestinationUpdate');
      uiObserver.off('navDestinationSwitch');
      uiObserver.off('routerPageUpdate');
    }
  }
}
```

### NavDestination state accounting

`PageTracingPoint.zip#PageTracingPoint/entry/src/main/ets/util/PageTracingManager.ets`

```ts
public registerNavUpdateListener(observer: UIObserver) {
  observer.on('navDestinationUpdate', (info: uiObserver.NavDestinationInfo) => {
    let navDestinationInfo: CustomPageInfo = CustomInfoUtil.navInfo(info);
    let markId = CustomInfoUtil.getUniqueMarkId(navDestinationInfo);
    if (info.state === uiObserver.NavDestinationState.ON_APPEAR) {
      let accessCount = this.accessCountMap.get(markId) ?? 0;
      accessCount++;
      this.accessCountMap.set(markId, accessCount);
      this.pageStack.add(navDestinationInfo);
    }

    if (info.state === uiObserver.NavDestinationState.ON_DISAPPEAR) {
      this.startTimeMap.delete(markId);
      this.pageStack.forEach((value, index) => {          // FIX (HW-19-0069)
        if (CustomInfoUtil.getUniqueMarkId(value) === markId) {
          this.pageStack.removeByIndex(index);
        }
      });
    }

    if (info.state === uiObserver.NavDestinationState.ON_SHOWN) {
      this.startTimeMap.set(markId, Date.now());
    }

    if (info.state === uiObserver.NavDestinationState.ON_HIDDEN) {
      let startTime = this.startTimeMap.get(markId) ?? Date.now();
      let accessTime = this.accessTimeMap.get(markId) ?? 0;
      this.accessTimeMap.set(markId, Date.now() - startTime + accessTime);
    }
  });
}
```

### The navBar switch listener (as shipped - see HW-19-0068)

`PageTracingPoint.zip#PageTracingPoint/entry/src/main/ets/util/PageTracingManager.ets`

```ts
public registerSwitchListener(observer: UIObserver) {
  observer.on('navDestinationSwitch', (info: uiObserver.NavDestinationSwitchInfo) => {
    if (info.from === 'navBar') {
      let rootInfo = this.pageStack[0];        // FIX: guard pageStack.length === 0
      let markId = CustomInfoUtil.getUniqueMarkId(rootInfo);
      let startTime = this.startTimeMap.get(markId) ?? 0;
      let preTime = this.accessTimeMap.get(markId) ?? 0;
      let duration = startTime === 0 ? 0 : Date.now() - startTime;
      this.accessTimeMap.set(markId, duration + preTime);
    }
    if (info.to === 'navBar') {
      let rootInfo = this.pageStack[0];        // FIX: same guard
      let markId = CustomInfoUtil.getUniqueMarkId(rootInfo);
      this.startTimeMap.set(markId, Date.now());
    }
  });
}
```

### Router page accounting, with the focus check

`PageTracingPoint.zip#PageTracingPoint/entry/src/main/ets/util/PageTracingManager.ets`

```ts
if (info.state === uiObserver.RouterPageState.ON_PAGE_HIDE) {
  this.pageStack.forEach((value, index) => {
    if (CustomInfoUtil.getUniqueMarkId(value) === markId) {
      if (index === this.pageStack.length - 1 || this.pageStack[index as number + 1].uniqueId === 'Router') {
        let startTime = this.startTimeMap.get(markId) ?? Date.now();
        let accessTime = this.accessTimeMap.get(markId) ?? 0;
        this.accessTimeMap.set(markId, Date.now() - startTime + accessTime);
      }
    }
  });
}
```

The inner condition is the interesting part: a Router page is only credited with
having lost focus if it is the top of the stack, or if the entry above it is
another Router page - otherwise a NavDestination merely opened on top of it and
the Router page is still the active page.

### Normalising the two info shapes

`PageTracingPoint.zip#PageTracingPoint/entry/src/main/ets/util/CustomInfoUtil.ets`

```ts
export class CustomInfoUtil {
  public static routerInfo(info: uiObserver.RouterPageInfo) {
    return new CustomPageInfo(info.pageId, info.name, 'Router');
  }

  public static navInfo(info: uiObserver.NavDestinationInfo) {
    return new CustomPageInfo(info.navDestinationId, info.name.toString(), `${info.navigationId}`);
  }

  public static getUniqueMarkId(info: CustomPageInfo) {
    return `${info.name}-${info.uniqueId}`;
  }
}
```

### The report (as shipped - see HW-19-0071 and HW-19-0072)

`PageTracingPoint.zip#PageTracingPoint/entry/src/main/ets/pages/TracingResultPage.ets`

```ts
aboutToAppear() {
  let tracingIns = PageTracingManager.getInstance();
  let accessCountMap = tracingIns.getAccessCountMap();
  let accessTimeMap = tracingIns.getAccessTimeMap();
  NEWS.forEach((news) => {
    let key = news.pageName + '-TracingNavigation';
    let pageCount = accessCountMap.get(key) ?? 0;
    let pageTime = accessTimeMap.get(key) ?? 0;
    this.tracingItems.push({ news: news, accessCount: pageCount, accessTime: pageTime });
  });
}

getTime(mills: number): string {
  let second = Math.round(mills / CommonConstant.MILLS_SECOND_MAGNITUDE);
  if (second >= CommonConstant.MINUTES_BOUNDARY) {         // FIX (HW-19-0071): test hours first
    second = Math.floor(second / CommonConstant.MINUTES_BOUNDARY);
    return `${second}${CommonConstant.MINUTES_TEXT}`;
  } else if (second >= CommonConstant.HOURS_BOUNDARY) {    // unreachable
    second = Math.floor(second / CommonConstant.HOURS_BOUNDARY);
    return `${second}${CommonConstant.HOURS_TEXT}`;
  } else {
    return `${second}${CommonConstant.SECOND_TEXT}`;
  }
}

getCounts(origin: number): string {
  if (origin < CommonConstant.TEN_THOUSAND) {
    return origin.toString();
  } else if (origin < CommonConstant.HUNDRED_MILLION) {
    return `${(origin / CommonConstant.TEN_THOUSAND).toFixed(1)}${CommonConstant.TEN_THOUSAND_TEXT}`;
  } else {
    return `${(origin / CommonConstant.HUNDRED_MILLION).toFixed(1)}${CommonConstant.HUNDRED_MILLION_TEXT}`;
  }
}
```

### The magnitude constants (as shipped - see HW-19-0072)

`PageTracingPoint.zip#PageTracingPoint/entry/src/main/ets/constant/CommonConstant.ets`

```ts
static readonly TEN_THOUSAND_TEXT: string = '万';
static readonly HUNDRED_MILLION_TEXT: string = '亿';
static readonly TEN_THOUSAND: number = 10000;
static readonly HUNDRED_MILLION: number = 1000000000;   // FIX: 亿 is 100000000
static readonly MILLS_SECOND_MAGNITUDE: number = 1000;
static readonly MINUTES_BOUNDARY: number = 60;
static readonly HOURS_BOUNDARY: number = 3600;
```

### Registering and releasing from the ability

`PageTracingPoint.zip#PageTracingPoint/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
windowStage.loadContent('pages/HomePage', (err) => {
  if (err.code) { /* ... */ return; }
  let mWindow = windowStage.getMainWindowSync();
  let uiContext = mWindow.getUIContext();
  // FIX (HW-19-0070): TYPE_CUTOUT + 20 is not the status-bar inset
  let cutoutArea = mWindow.getWindowAvoidArea(window.AvoidAreaType.TYPE_CUTOUT);
  let topRectHeight = uiContext.px2vp(cutoutArea.topRect.height) + 20;
  let navArea = mWindow.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR);
  AppStorage.setOrCreate('topRectHeight', topRectHeight);
  AppStorage.setOrCreate('bottomRectHeight', uiContext.px2vp(navArea.bottomRect.height));
  mWindow.on('avoidAreaChange', (data) => { /* ... */ });
  mWindow.setWindowLayoutFullScreen(true);
  this.uiObserver = uiContext.getUIObserver();

  PageTracingManager.getInstance().initPageTracing(uiContext);
});

onWindowStageDestroy(): void {
  PageTracingManager.getInstance().unsubscribe(this.uiObserver as UIObserver);
  hilog.info(DOMAIN, 'testTag', '%{public}s', 'Ability onWindowStageDestroy');
}
```

## Permissions & config

**No permissions are required** and none are declared - the observer reports the
application's own page transitions and nothing leaves the device. Statistics live
only in memory: the maps are fields of a singleton, so they reset on every
application restart. A production analytics layer would persist or upload them,
and that is where a permission would first appear.

`PageTracingPoint.zip#PageTracingPoint/entry/src/main/module.json5` declares
`"deviceTypes": ["phone", "tablet", "2in1"]`, the usual `EntryAbility`, and a
`routerMap` profile for the Navigation destinations. There is no
`requestPermissions` block.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **The observer is obtained from a `UIContext`**, so registration cannot happen
  before the window content is loaded - hence the call inside the `loadContent`
  callback.
- **`navDestinationUpdate` never reports the Navigation home content.** The
  `navBar` is not a NavDestination; `navDestinationSwitch` is the only source for
  it, which is why the sample needs all three listeners.
- **The identity key embeds the Navigation component's `id`.** `HomePage` sets
  `.id('TracingNavigation')`, and `TracingResultPage` hardcodes
  `news.pageName + '-TracingNavigation'` when reading the maps - the two must be
  kept in step.
- **Statistics are in-memory only** and are lost on restart.
- **`off(type)` with no callback removes every callback for that event** -
  acceptable here because the tracer owns all three registrations, but not if
  another component subscribes to the same events.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **`this.pageStack[0]` is read without an emptiness check, which is
  incorrect.** `navDestinationSwitch` can fire when the stack is empty, and
  `getUniqueMarkId` then dereferences `undefined.name` inside an observer
  callback with no try/catch. (HW-19-0068)
- **`removeByIndex` is called from inside `pageStack.forEach`, which is
  incorrect.** Removing shifts the later indices while the traversal continues,
  so an element immediately after a removed one is skipped - visible as soon as
  two live entries share a key. (HW-19-0069)
- **The top inset is taken from `TYPE_CUTOUT` plus a hardcoded 20, which is
  incorrect.** `TYPE_SYSTEM` is the status-bar area; on a device with no cutout
  the padding degenerates to a flat 20 vp. The same mistake appears in COMMON-08.
  (HW-19-0070)
- **`getTime` tests minutes before hours, which is incorrect** - the hours branch
  is unreachable and a two-hour session reads as '120分'. (HW-19-0071)
- **`HUNDRED_MILLION = 1000000000` is incorrect** - 亿 is 10^8. The constant is
  both the threshold and the divisor, so counts above 10^8 are shown in the wrong
  unit and then divided by ten times too much. (HW-19-0072)
- **`ON_HIDDEN` accumulates but does not clear `startTimeMap`.** A second
  `ON_HIDDEN` without an intervening `ON_SHOWN` would add the interval again;
  only `ON_DISAPPEAR` deletes the entry.
- **The `?? Date.now()` fallback in the hidden branches hides a missing
  stopwatch** by producing a zero duration instead of an error. That is
  deliberate, but it means a lost `ON_SHOWN` silently under-counts rather than
  showing up as a bug.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-arkui-observer.md` -
  `UIObserver.on`/`off` for `navDestinationUpdate`, `navDestinationSwitch` and
  `routerPageUpdate`; `NavDestinationInfo`, `NavDestinationState`,
  `NavDestinationSwitchInfo`, `RouterPageInfo`, `RouterPageState`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-arkui-observer
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` -
  `UIContext.getUIObserver()`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-references/02_application-framework/js-apis-arraylist.md` -
  `ArrayList`, `add`, `removeByIndex`, `forEach` and its callback parameters.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-arraylist
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-e.md` -
  `AvoidAreaType.TYPE_SYSTEM` vs `TYPE_CUTOUT`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-e
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` -
  `Navigation` / `NavDestination`, whose lifecycle states the observer reports.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/page_tracing_point-0000002283641138
