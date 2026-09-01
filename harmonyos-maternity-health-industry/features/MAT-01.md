---
id: MAT-01
title: Maternity and child health app - layered application architecture
industry: 10_maternity_health
doc: huawei_industry_tree/10_maternity_health/docs/01_practice-health-app-architecture-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-health-app-architecture-v1-0000001938172424
sample: huawei_industry_tree/10_maternity_health/downloads/MYDemo.zip
kits: ["@kit.ArkUI", "@kit.ArkTS", "@kit.AbilityKit", "@kit.MediaLibraryKit", "@kit.ArkData", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [NavPathStack, Navigation, LazyForEach, IDataSource, photoAccessHelper.getPhotoAccessHelper, photoAccessHelper.getAssets, cameraPicker.pick, abilityAccessCtrl.requestPermissionsFromUser, window.getLastWindow, UIContext.getMeasureUtils, GridRow, Tabs]
permissions: [ohos.permission.READ_IMAGEVIDEO]
min_api: 20
modules: [entry, common, features/recommend, features/cloudCircle, features/mine, features/record]
findings: [HW-10-0001, HW-10-0002, HW-10-0003, HW-10-0004, HW-10-0005, HW-10-0006, HW-10-0007, HW-10-0008, HW-10-0009, HW-10-0010, HW-10-0011, HW-10-0012, HW-10-0013]
status: verified-with-fixes
---

## When to use

Load this card when building a maternity, pregnancy-tracking, baby-care or
general consumer health app that combines a **social feed**, a **health
calendar** and a **personal profile** area behind a bottom tab bar. It defines
the module split and the navigation skeleton the rest of the `MAT-` cards plug
into. It is an architecture card, not a single feature: start here, then load
the feature card for the screen you are actually building.

## Feature checklist

A maternity health app built to this architecture covers:

- **Home / recommend** - in-app message entry, search box, post-composer entry,
  quick-action cards, and a scrolling image-and-text post feed.
- **Record** - a custom month calendar that renders three mutually exclusive
  tracking modes: menstrual cycle (period days, predicted period, ovulation),
  pregnancy (early / mid / late trimester bands), and parenting. Plus list rows
  for antenatal appointments, baby weight and height, and the vaccination book.
- **Community** - a four-tab area (square, Q&A, experience, activities) with
  share / comment / like / favourite actions on each post.
- **Mine** - settings, profile, account, nickname, and mode switching.
- **Post composer** - a `TextArea` plus a system-album grid and a camera entry,
  with runtime permission handling before the composer opens.
- **Splash** - a skippable countdown screen ahead of the tab shell.

## Architecture

Three layers, each shipped as HAR packages inside one IDE project:

| Layer | Contents | Module |
|---|---|---|
| Product customisation | Page shell, navigation, splash, tab host, phone-specific resources | `entry` |
| Basic feature | Recommend, record, community, mine - one HAR each | `features/*` |
| Common capability | Shared components, models, lazy data source, breakpoint and log utils | `common` |

Data flow: `EntryAbility` loads `pages/EntryPage`. `EntryPage` owns a single
`NavPathStack`, publishes it with `@Provide('pageInfos')`, and maps route names
to `NavDestination` builders. Every feature HAR consumes the stack with
`@Consume('pageInfos')` and pushes by name, so feature modules never import each
other - they only know route strings. Status-bar and navigation-bar heights are
measured once in `EntryPage` and published through `AppStorage`, which is how
every page gets its top padding.

> The document's software view also claims routing, network and database HARs in
> the common layer. They are not in the sample - see `HW-10-0013`. Treat routing
> as "raw NavPathStack in entry" and plan your own network and persistence
> modules.

## Implementation steps

1. **Register every module in `build-profile.json5`.** Only `entry` gets a
   `targets` block bound to the product; the HARs are listed with `name` and
   `srcPath` only.
2. **Set up one navigation stack in the entry page.** Provide it, map route
   names to destinations in an `@Builder`, hide the nav bar, and use
   `NavigationMode.Stack`.
3. **Publish system inset heights to `AppStorage`** from `aboutToAppear` so
   every feature page can pad itself without re-querying the window.
4. **Implement one shared `IDataSource`** in `common` and use it for every feed.
   Get the notification calls right - this is where the sample is weakest
   (`HW-10-0001`, `HW-10-0005`, `HW-10-0009`).
5. **Give every `LazyForEach` a stable key generator** derived from an
   identifying field. Never `JSON.stringify` the item, never fold in the index
   (`HW-10-0006`).
6. **Build the calendar from numeric date components**, never from formatted
   strings (`HW-10-0004`).
7. **Request `ohos.permission.READ_IMAGEVIDEO` before routing to the composer**,
   not inside it, so a denied grant never leaves the user on a dead page.

## Verified snippets

All snippets are taken from `MYDemo.zip`. Where the shipped code carries a
finding, the corrected form is given and marked.

**Single navigation stack — `entry/src/main/ets/pages/EntryPage.ets`** (as shipped)

```typescript
@Entry
@Component
struct EntryPage {
  @Provide('pageInfos') pageInfos: NavPathStack = new NavPathStack();
  uiContext: UIContext = this.getUIContext();
  context: Context = this.uiContext.getHostContext() as Context;

  @Builder
  PagesMap(name: string) {
    if (name === 'flashScreenPage') {
      FlashScreenPage()
    } else if (name === 'tabPage') {
      TabPage()
    } else if (name === 'postingPage') {
      PostingPage()
    } else if (name === 'detail') {
      FriendCircleDetailPage()
    }
  }

  build() {
    Navigation(this.pageInfos) {
    }
    .hideNavBar(true)
    .mode(NavigationMode.Stack)
    .navDestination(this.PagesMap)
  }
}
```

**Publishing system inset heights — same file, `aboutToAppear`** (as shipped)

```typescript
window.getLastWindow(this.context).then((windowStage: window.Window) => {
  let area = windowStage.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM);
  let topHeight = this.uiContext.px2vp(area.topRect.height);
  let bottomArea = windowStage.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR);
  let bottom = this.uiContext.px2vp(bottomArea.bottomRect.height);
  if (topHeight > 0) {
    AppStorage.setOrCreate('statusBarHeight', topHeight);
    AppStorage.setOrCreate('navigationBarHeight', bottom);
  }
});
```

Consume it anywhere with `@StorageProp('statusBarHeight') statusBarHeight: number = 0;`.

**Permission gate before the composer — `common/src/main/ets/utils/DropPostingPageUtil.ets`** (as shipped)

```typescript
import { abilityAccessCtrl, PermissionRequestResult, Permissions } from '@kit.AbilityKit';
import { BusinessError } from '@kit.BasicServicesKit';

export function dropPositingPage(context: Context, pageInfos: NavPathStack) {
  let permissions: Array<Permissions> = ['ohos.permission.READ_IMAGEVIDEO'];
  let atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
  atManager.requestPermissionsFromUser(context, permissions).then((data: PermissionRequestResult) => {
    let grantStatus: Array<number> = data.authResults;
    for (let i = 0; i < grantStatus.length; i++) {
      if (grantStatus[i] != 0) {
        return;   // denied: stay put, guide the user to Settings
      }
    }
    pageInfos.pushPath({ name: 'postingPage' });
  }).catch((err: BusinessError) => {
    Logger.error(`Failed to request permissions. Code ${err.code}, message ${err.message}`);
  })
}
```

The shape worth copying: request first, push the route only inside the granted
branch. The composer page never has to handle a permission-less state.

**Reading the system album — `features/recommend/src/main/ets/pages/PostingPage.ets`** (corrected, see `HW-10-0003`)

```typescript
async initAlbumDataResource() {
  try {
    this.albumData.clear();            // FIX: shipped code omits this and duplicates on every onShown
    let phAccessHelper = photoAccessHelper.getPhotoAccessHelper(this.context);
    let photoPredicates: dataSharePredicates.DataSharePredicates = new dataSharePredicates.DataSharePredicates();
    photoPredicates.orderByDesc(photoAccessHelper.PhotoKeys.DATE_ADDED);
    photoPredicates.equalTo(photoAccessHelper.PhotoKeys.PHOTO_TYPE, photoAccessHelper.PhotoType.IMAGE);
    let photoFetchOptions: photoAccessHelper.FetchOptions = {
      fetchColumns: [],
      predicates: photoPredicates
    };
    let photoFetchResult = await phAccessHelper.getAssets(photoFetchOptions);
    if (photoFetchResult) {
      let photoAssets = await photoFetchResult.getAllObjects();
      for (let i = 0; i < photoAssets.length; i++) {
        this.albumData.addData(i, [{ index: i, uri: photoAssets[i].uri }]);
      }
      photoFetchResult.close();        // always close the fetch result
    }
  } catch (err) {
    Logger.error('initDataResource failed with err: ' + err)
  }
}
```

**Calendar month grid — `features/record/src/main/ets/components/CalendarComp.ets`** (corrected, see `HW-10-0004`)

```typescript
// Leading blank columns for the first day of the displayed month.
// FIX: shipped code builds this from `${currYear}-${currMonth}-01`, which is
// parsed as UTC for months 10-12 and as local time for months 1-9.
getOffsetColumn(): number {
  return new Date(this.currYear, this.currMonth - 1, 1).getDay();
}

GridRow({ columns: this.weeksList.length, gutter: CommonConstants.PADDING / 2 }) {
  ForEach(this.dayData, (item: CalendarData, index: number) => {
    GridCol({ offset: index === 0 ? this.getOffsetColumn() : 0 }) {
      // day cell
    }
  })
}
```

The mode switch is driven by `@Consume @Watch('changeCurrPattern') currPattern`
and the month by `@Link @Watch('changeDayData') currYear / currMonth`, so the
parent page owns the month and the component only renders it.

**Month paging by pan gesture — same file** (as shipped)

```typescript
.parallelGesture(
  PanGesture(this.panOption)
    .onActionUpdate((event: GestureEvent) => { this.offsetX = event.offsetX; })
    .onActionEnd((event: GestureEvent) => {
      this.uiContext.animateTo({ curve: Curve.Smooth }, () => {
        if (Number(event.offsetX) > Constants.CALENDAR_OFFSET_X) {
          this.startDay.setMonth(this.startDay.getMonth() - 1)
        }
        if (event.offsetX < -Constants.CALENDAR_OFFSET_X) {
          this.startDay.setMonth(this.startDay.getMonth() + 1)
        }
        this.currYear = this.startDay.getFullYear();
        this.currMonth = this.startDay.getMonth() + 1;
        this.offsetX = 0;
      })
    })
)
```

## Permissions & config

`entry/src/main/module.json5`:

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.READ_IMAGEVIDEO",
    "reason": "$string:permission_reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
  }
]
```

Declared in the entry HAP only - the feature HARs declare no permissions.
`READ_IMAGEVIDEO` is needed because the composer enumerates the whole album with
`photoAccessHelper.getAssets()`. If you only need the user to pick a few images,
use `PhotoViewPicker` instead and drop the permission entirely.

`build-profile.json5` products block:

```json5
"products": [{
  "name": "default",
  "signingConfig": "default",
  "compatibleSdkVersion": "6.0.0(20)",
  "runtimeOS": "HarmonyOS",
  "targetSdkVersion": "6.0.0(20)"
}]
```

## Constraints

- DevEco Studio 6.0.0 Release or later; HarmonyOS 6.0.0 Release SDK or later.
- The document scopes the practice to Huawei phones, but the shipped manifest
  declares `["phone", "tablet", "2in1"]` (`HW-10-0012`). Decide which you mean
  before you ship - the sample does contain `sm` / `md` / `lg` breakpoint
  handling, so multi-device is workable.
- All content in the sample is hardcoded local arrays. There is no network
  layer, no database and no cloud sync; the document says to replace post
  content with device-cloud data yourself.
- The calendar's period, ovulation and trimester data are constants keyed by
  day-of-month, not real cycle maths.

## Pitfalls

Each item is a defect found in this document's sample. Fix it before shipping.

- **`HW-10-0001` — `LazyDataSource.addData()` breaks the list on bulk append.**
  The document ships an `addData(index, data[])` that `concat`s N items but
  fires one `notifyDataAdd`. Do not use it as-is: insert at the index and notify
  once per element. The sample's own load-more path passes a six-element array.
- **`HW-10-0002` — the composer's remove-photo handler deletes the wrong photo.**
  The shipped predicate is `item.uri === item.uri`, always true, so tapping any
  delete badge removes the first selected photo. Bind the row item and compare
  against it.
- **`HW-10-0003` — the album grid duplicates itself.** The shipped page reloads
  the album from `onShown` without clearing the data source, so every return to
  the page appends the entire album again. Clear first, or load from
  `aboutToAppear`.
- **`HW-10-0004` — never build calendar dates from template strings.** The
  shipped code uses `` `${year}-${month}-01` ``, which is valid ISO 8601 only for
  months 10-12; those get parsed as UTC while months 1-9 are parsed as local
  time. West of UTC the entire October, November and December grid shifts one
  weekday column. Use `new Date(year, month - 1, day)`.
- **`HW-10-0005` — `findDataIndex()` always returns -1.** The shipped predicate
  wrapper has a block body with no `return`. Rewrite it before you call it.
- **`HW-10-0006` — do not `JSON.stringify` in a `LazyForEach` key generator.**
  The shipped code does it twice, against the official
  `@performance/hp-arkui-no-stringify-in-lazyforeach-key-generator` rule, and one
  of the two also concatenates the index so keys of unchanged rows shift on
  insert. Key on an identifying field.
- **`HW-10-0007` — measure text with the weight you actually render.** The tab
  underline is measured at weight 700 but the label renders at 500, so the
  indicator sits right of centre. The demo hides this because all four labels are
  two Chinese characters; the moment you change the labels it shows.
- **`HW-10-0008` — gate every calendar marker on the displayed month.** The
  shipped code guards `isMenes` but not `isForecastMenes` or `isOvulation`, so
  the same predicted period and ovulation days are painted in every month of the
  year.
- **`HW-10-0009` — `appendArrayData` triggers a full list rebuild.** It calls
  `notifyDataReload()` for a single append, and `pushArrayData` fires n+2
  notifications with two full rebuilds. This directly contradicts the long-list
  performance practice the document itself points you at.
- **`HW-10-0010` — pick one import convention.** The sample mixes deprecated
  `@ohos.*` paths with `@kit.*` in the same file, including a dead
  `import measure from '@ohos.measure'` whose API has been deprecated since API
  18. Use `@kit.*` throughout.
- **`HW-10-0011` — the shipped file is `SerachInputComp.ets`, not
  `SearchInputComp.ets`.** The document prints the corrected spelling, so import
  paths copied from the document will not resolve.
- **`HW-10-0013` — the common capability layer is thinner than advertised.**
  The document describes routing, network and database HARs; none of them exist
  in the sample. Budget for building them.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-lazyforeach.md` - key generation rules, data update semantics
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-lazyforeach
- `documentation/harmonyos-guides/12_coding-and-debugging/ide_hp-arkui-no-stringify-lazyforeach-key.md` - the key generator rule broken by `HW-10-0006`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ide_hp-arkui-no-stringify-lazyforeach-key
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-measureutils.md` - `measureText`, return unit is px
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-measureutils
- `documentation/harmonyos-references/02_application-framework/js-apis-measure.md` - deprecation of `MeasureText.measureText`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-measure
- `documentation/harmonyos-guides/04_system/request-app-permissions.md` - runtime permission request flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-app-permissions
- Document's own pointer: `practice-common-app-layered-v1` (layered modularisation) and `bpta-best-practices-long-list` (long list loading)
