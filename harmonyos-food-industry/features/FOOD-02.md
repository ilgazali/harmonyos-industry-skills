---
id: FOOD-02
title: Recipe app template - waterfall feed over a LazyDataSource and a two-list linked category browser, on ArkUI V2 state
industry: 17_food
doc: huawei_industry_tree/17_food/docs/02_practice-food-menu-app-demo-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-menu-app-demo-v1-0000002300530042
sample: huawei_industry_tree/17_food/downloads/Recipes.zip
kits: ["@kit.AbilityKit", "@kit.AccountKit", "@kit.AdsKit", "@kit.ArkData", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CryptoArchitectureKit", "@kit.FormKit", "@kit.MediaLibraryKit", "@kit.PaymentKit", "@kit.PerformanceAnalysisKit", "@kit.ScenarioFusionKit"]
apis: [abilityAccessCtrl, advertising, authentication, common, cryptoFramework, display, emitter, formBindingData, formInfo, fs, functionalButtonComponentManager, hilog]
permissions: [ohos.permission.INTERNET, ohos.permission.GET_NETWORK_INFO, ohos.permission.APP_TRACKING_CONSENT]
min_api: 24
modules: [shopping_basket (har), home_search (har), aggregated_login (har), upload_recipe (har), aggregated_payment (har), base_ui (har), personal_homepage (har), link_category (har)]
findings: [HW-17-0014, HW-17-0015, HW-17-0016, HW-17-0017, HW-17-0018, HW-17-0019, HW-17-0020, HW-17-0021, HW-17-0028, HW-17-0029, HW-17-0030]
status: verified-with-fixes
---

## When to use

Load this card when you are building a **content-browsing app over a catalogue**
- recipes here, but equally workouts, courses, articles or products. The two
patterns the document is built around are the ones to lift: a **waterfall feed
backed by a lazy data source**, and a **two-list category browser** where a
narrow rail on the left and a grouped list on the right drive each other.

It is also the cleanest example in this industry of the **ArkUI V2 state model
applied across HAR boundaries**: `@ComponentV2` components take `@Param` in and
give `@Event` out, view models are `@ObservedV2` singletons with `@Trace`
fields, and account state is persisted through `PersistenceV2`. Reusable
components live in `components/` and know nothing about routing; feature HARs in
`features/` own the pages and do the navigating.

**Read `HW-17-0014` before you copy anything from `features/mine/util/`.** The
sample signs payment orders on the device with a full RSA-2048 merchant private
key pasted into the source. That is the one pattern in this template you must
not reproduce.

## Feature checklist

- Launch page, optional splash ad and a privacy-policy gate before the tab
  frame appears.
- Four bottom tabs - 首页 (home), 分类 (categories), 热量计算 (calories),
  我的 (mine) - each a `wrapBuilder`ed page from its own feature HAR.
- Home: banner swiper, quick-filter chips, and a two-column waterfall of recipe
  cards that loads lazily.
- The search box collapses into the header as the feed scrolls past it.
- Search by name or by category keyword, with hot-search chips and a result
  list.
- Categories: left rail of category names, right list grouped by category with a
  three-column grid of dishes; tapping the rail scrolls the list, scrolling the
  list highlights the rail.
- Recipe detail with favourite, ingredient list and "add to basket"; the basket
  page manages what was added.
- Calories: daily and weekly bar chart, diet plans per meal slot, food search.
- Mine: Huawei ID silent login, member centre with a payment picker, my recipes,
  uploads, favourites, browsing history, notification centre, settings.
- A service widget deep-links into any of the tabs or into the favourites page,
  routing through the login page first when the target needs an account.

## Architecture

Four layers, twenty-odd modules, one entry HAP.

```
Recipes
├── commons
│   ├── commonlib/src/main/ets/
│   │   ├── components/{BaseHeader,BuildTitleBar,HeaderMenuBuilder,MenuItemBuilder}.ets
│   │   ├── constants/{CommonConstants,CommonEnums}.ets   RouterMap enum lives here
│   │   ├── types/Types.ets                               UserInfo and friends
│   │   └── utils/
│   │       ├── AccountUtil.ets      Huawei ID silent login + PersistenceV2 user model
│   │       ├── RouterModule.ets     one static NavPathStack, push/replace/pop by name
│   │       └── {PreferenceUtil,PermissionUtil,WindowUtil,DialogUtil,FormatUtil,Logger}.ets
│   └── network/src/main/ets/
│       ├── apis/{APIList,HttpRequest}.ets    axios-shaped request layer
│       ├── mocks/{AxiosMock,RequestMock}.ets + MockData/*  the entire "backend"
│       └── types/{Recipe,Calories,Member,Notice}.ets
├── components                       reusable, route-free UI HARs
│   ├── featured_recipes/            WaterFlow + LazyForEach + LazyDataSource/ObservedArray
│   ├── link_category/               the two-list linked category browser
│   ├── home_search/                 search box, hot keys, results
│   ├── shopping_basket/             basket and ingredient lists
│   ├── calorie_calculation/         BarChart, CaloriesSummary, FoodDiary
│   ├── personal_homepage/           blogger header + recipe grid, reused for self and others
│   ├── upload_recipe/ + base_ui/    upload form; BaseTabs
│   └── aggregated_{login,payment,ads}/  login panel, payment picker, splash ad + OAID
├── features                         one HAR per tab, pages + view models
│   ├── home/          HomePage, SearchPage, BloggerProfilePage, FollowersPage
│   ├── classification/ ClassificationPage, DishesPage, ShoppingBasketPage
│   ├── calories/      CaloriesPage, DietPlanPage, SearchFoodPage
│   └── mine/          MinePage + 12 sub-pages, util/{MockApi,OrderInfoUtil,SignUtils}
└── products/entry/src/main/ets/
    ├── entryability/EntryAbility.ets     want parsing for widget deep links
    ├── entryformability/EntryFormAbility.ets
    ├── pages/{Index,LaunchPage,LaunchAdPage,PrivacyPolicyPage,MainEntry}.ets
    ├── viewmodels/MainEntryVM.ets        the four tabs as wrapBuilder entries
    └── widget/pages/WidgetCard.ets
```

The documented tree is close but not exact: it writes `CommonContants.ets` for
`CommonConstants.ets`, `UploadRecipe.ets` for `mine/pages/UploadRecipePage.ets`,
and annotates `BarChart.ets` as 协议弹窗组件 (agreement dialog component) when it
is the bar chart (`HW-17-0017`).

**The design decision worth copying** is the split between `components/` and
`features/`. A `components/` HAR receives data and hands back events; it never
imports `RouterModule` and never knows a page name. `LinkCategory` takes
`recipeCategoryList` and `currentIndex` as `@Param` and emits `onRecipeClick`
and `changeCurrentIndex` as `@Event`; `ClassificationPage` in
`features/classification` supplies the view model and turns the click into
`RouterModule.push(RouterMap.DISHES, …)`. The same component could be dropped
into a shopping app unchanged. Contrast with `FOOD-01`, where the menu component
pushes routes itself and drags a routing dependency into every feature HAR.

The tab frame follows from that: `MainEntryVM` holds the four pages as
`wrapBuilder(HomePageBuilder)` values, so `products/entry` depends on the four
feature HARs and on nothing below them.

## Implementation steps

1. **Lay out four layers**: `commons` (utilities and the mock network),
   `components` (route-free reusable UI), `features` (one HAR per tab), and one
   `products/entry` HAP that composes them.
2. **Route by name through one static stack.** `commonlib/RouterModule` owns a
   single `NavPathStack`; `Index.ets` hands it to `Navigation` and replaces to
   the launch page. Every HAR that owns pages declares them in its own
   `router_map.json`.
3. **Model each page with an `@ObservedV2` singleton view model** exposing
   `@Trace` fields, and let `@ComponentV2` pages read `vm.x` directly - no
   `@State` copies, no prop drilling.
4. **Feed the waterfall from an `IDataSource`.** Implement `totalCount`/`getData`
   plus the listener bookkeeping once, then use `LazyForEach` with a stable
   `item.id.toString()` key.
5. **Build the category browser as two `List`s sharing an index.** The left one
   is a plain `ForEach` of names; the right one is `ListItemGroup` per category.
   Guard the cross-scroll with an equality check so the two `scrollToIndex`
   calls cannot ping-pong.
6. **Silent-login on start** with `forceLogin = false`, verify the returned
   `state` against the one you generated, and persist the profile through
   `PersistenceV2` so a restart does not re-prompt.
7. **Keep the same callback reference for `emitter.on` and `emitter.off`**
   (`HW-17-0015`); `MainEntry` shows the correct form, `MinePage` the wrong one.
8. **Validate every want parameter** before parsing it - widget and third-party
   launches reach `onCreate`/`onNewWant` unchecked (`HW-17-0018`).
9. **Request `APP_TRACKING_CONSENT` before the OAID,** branch on the result, and
   log the real error in each catch (`HW-17-0019`). Document the permission -
   the doc lists only `INTERNET` (`HW-17-0016`) - and drop
   `GET_NETWORK_INFO`, which nothing uses (`HW-17-0021`).
10. **Never sign a payment order on the device** (`HW-17-0014`). Ask a server for
    a pre-signed order string; the client should hold no key material.

## Verified snippets

All snippets are from `Recipes.zip`. Corrected forms are marked.

**The linked category browser — `components/link_category/src/main/ets/components/LinkCategory.ets`** (as shipped)

```typescript
@ComponentV2
export struct LinkCategory {
  @Param recipeCategoryList: RecipeCategory[] = [];
  @Param currentIndex: number = 0;
  titleItemScroller: Scroller = new Scroller();
  scroller: Scroller = new Scroller();
  @Event onRecipeClick: (recipeDetail: RecipeBriefInfo) => void = () => {};
  @Event changeCurrentIndex: (currentIndex: number) => void = () => {};

  // 下标索引处理 (index change handling)
  currentIndexChangeAction(index: number, isClassify: boolean): void {
    if (this.currentIndex !== index) {
      this.changeCurrentIndex(index);
      if (isClassify) {
        this.scroller.scrollToIndex(index);        // rail tapped -> move the content list
      } else {
        this.titleItemScroller.scrollToIndex(index); // content scrolled -> move the rail
      }
    }
  }

  build() {
    Row() {
      List({ space: 8, scroller: this.titleItemScroller }) {
        ForEach(this.recipeCategoryList, (item: RecipeCategory, index: number) => {
          ListItem() {
            this.leftListBuilder(item.name, index);
          };
        }, (item: RecipeCategory, index: number) => item.name + index);
      }
      .width(92)
      .listDirection(Axis.Vertical)
      .scrollBar(BarState.Off)
      .contentStartOffset(12).contentEndOffset(12);

      List({ space: 12, scroller: this.scroller }) {
        ForEach(this.recipeCategoryList, (item: RecipeCategory) => {
          ListItemGroup({ header: this.listTitleBuilder(item.name), space: 12 }) {
            ListItem() {
              Grid() {
                ForEach(item.recipeList, (listItem: RecipeBriefInfo) => {
                  GridItem() {
                    // card
                  }.onClick(() => {
                    this.onRecipeClick(listItem);
                  });
                }, (listItem: RecipeBriefInfo) => `${item.id}${listItem.id}`);
              }
              .columnsTemplate('1fr 1fr 1fr');
            };
          };
        }, (item: RecipeCategory) => item.id.toString());
      }
      .layoutWeight(1)
      .sticky(StickyStyle.None)
      .contentStartOffset(12).contentEndOffset(12)
      .onScrollIndex((start: number) => this.currentIndexChangeAction(start, false));
    }.layoutWeight(1);
  }
}
```

**Four details make this work.** First, `ListItemGroup` per category is what
lets `onScrollIndex` report a *category* index rather than a row index - the
right list has exactly as many top-level items as the left one, so one number
indexes both. Second, `currentIndexChangeAction` starts with
`if (this.currentIndex !== index)`: without that guard the rail's
`scrollToIndex` fires `onScrollIndex`, which scrolls the rail, which fires
again. Third, the `isClassify` flag says which side initiated, so only the
*other* list is scrolled. Fourth, `currentIndex` is a `@Param` owned by the view
model, not local state - the component asks its host to change it via
`@Event changeCurrentIndex`, and re-renders when the host does.

`contentStartOffset` / `contentEndOffset` on both lists reserve 12 vp at each
end, which keeps the highlighted rail item from sitting flush against the edge
when it is scrolled to the top.

**The waterfall feed — `components/featured_recipes/.../FeaturedRecipes.ets` and `utils/LazyDataSource.ets`** (as shipped)

```typescript
@ComponentV2
export struct FeaturedRecipes {
  @Param dishesList: LazyDataSource<RecipeBriefInfo> = new LazyDataSource();
  @BuilderParam uploadBuilderParam: () => void = this.uploadBuilder;

  build() {
    WaterFlow() {
      this.uploadBuilderParam();                    // optional first tile, e.g. "upload my recipe"
      LazyForEach(this.dishesList, (item: RecipeBriefInfo) => {
        FlowItem() {
          RecommendedCard({ recipe: item, /* … */ });
        };
      }, (item: RecipeBriefInfo) => item.id.toString());
    }
    .columnsGap(8)
    .rowsGap(8)
    .columnsTemplate('1fr 1fr');
  }
}

@ObservedV2
export default class LazyDataSource<T> extends BasicDataSource<T> {
  @Trace dataArray: T[] = [];

  public totalCount(): number {
    return this.dataArray.length;
  }

  public getData(index: number): T {
    return this.dataArray[index];
  }

  public pushArrayData(newData: ObservedArray<T>): void {
    this.clear();
    this.dataArray.push(...newData);
    this.notifyDataReload();                        // replace the page
  }

  public deleteData(index: number): void {
    this.dataArray.splice(index, 1);
    this.notifyDataDelete(index);                   // targeted, keeps siblings alive
  }
}
```

**`LazyForEach` only pays off if the data source tells the truth.**
`WaterFlow` asks for `totalCount()` and then for the items in the visible range
plus a cache margin, so a 500-recipe feed builds a couple of dozen
`RecommendedCard`s. The key function must be stable and unique -
`item.id.toString()` here - because ArkUI reuses a node when the key is
unchanged; keying on the index would make every insert re-render the tail.

The notify methods are not interchangeable. `notifyDataDelete(index)` tells the
framework exactly which node disappeared, so the rest keep their state;
`notifyDataReload()` rebuilds everything, which is right after a refresh
(`pushArrayData`) and wasteful for a single deletion. `@ObservedV2` +
`@Trace dataArray` additionally makes the array observable, so a V2 view model
can hold the source as a `@Trace` field and hand it over as a `@Param`.

**A `@BuilderParam` as the first flow item** is a small trick worth copying: the
same waterfall renders the home feed (no leading tile) and the "my recipes" grid
(leading "upload" tile) with no branch inside the component.

**Tab-to-page notification — `products/entry/.../MainEntry.ets` (as shipped) and `features/mine/.../MinePage.ets` (corrected, see `HW-17-0015`)**

```typescript
// MainEntry.ets — the correct shape: one stored reference, on and off
callBackFunc: () => void = () => {
  this.controller.changeIndex(1);
};

aboutToAppear(): void {
  emitter.on('jumpPage', this.callBackFunc);
  let url = this.vm.formCardJump.form.url;
  this.widgetInterception(url);
}

aboutToDisappear(): void {
  emitter.off('jumpPage', this.callBackFunc);
}

// …and the tab bar tells the mine page to refresh itself
.onChange((index: number) => {
  this.vm.curIndex = index;
  if (index === 3) {
    emitter.emit('refreshMinePage');
  }
});

// MinePage.ets — corrected
private refreshCb = () => {
  this.vm.refreshMinePage();
};

aboutToAppear(): void {
  this.vm.init();
  emitter.on('refreshMinePage', this.refreshCb);    // FIX: sample passes a fresh arrow function
}

aboutToDisappear(): void {
  emitter.off('refreshMinePage', this.refreshCb);   // FIX: …and a different one here, so nothing unsubscribes
}
```

**`emitter.off` matches on the callback identity.** The reference states plainly
that it takes effect only for a callback registered through `on` or `once`;
passing a newly allocated arrow function with the same body unregisters nothing.
`MinePage` lives inside a `TabContent`, so each rebuild adds one more live
subscription, all of them holding a captured `this`.

The pattern itself is sound and worth noting: pages inside a `Tabs` are built
once and not re-entered, so "refresh when my tab is selected" has to be pushed
in from the outside. `MainEntry` emits on `onChange(3)` and again from
`NavDestination.onShown`, which covers both switching tabs and returning from a
pushed page.

**Widget deep links — `products/entry/src/main/ets/entryability/EntryAbility.ets`** (corrected, see `HW-17-0018`)

```typescript
resolvePagePath(want: Want) {
  const parameters = want?.parameters;
  // FIX: sample checks only parameters?.url, then reads parameters?.params.toString() || ''
  if (parameters?.url && typeof parameters.params === 'string') {
    let formCardJump: FormCardJump = AppStorageV2.connect(FormCardJump, () => new FormCardJump())!;
    try {
      formCardJump.form = JSON.parse(parameters.params);
    } catch (err) {
      hilog.error(DOMAIN, 'testTag', 'Malformed widget params: %{public}s', JSON.stringify(err));
      return;
    }
    formCardJump.form.id = new Date().getTime();    // new id every launch, so @Monitor fires again
  }
}

onCreate(want: Want): void {
  this.resolvePagePath(want);
}

onNewWant(want: Want): void {
  this.resolvePagePath(want);
}
```

**The ability writes, the page reads - through `AppStorageV2`.** The ability has
no UI and cannot navigate; it parks the request in an `AppStorageV2`-connected
`FormCardJump` object, and `MainEntry` picks it up with
`@Monitor('vm.formCardJump.form.id', 'vm.formCardJump.form.url')`. Stamping a
fresh `id` on every launch is what makes the monitor fire when the *same* widget
is tapped twice; `MainEntry` then clears `form.url` after handling it so a later
unrelated change does not re-navigate.

Both entry points must call it: `onCreate` for a cold start, `onNewWant` for a
warm one. And both are reachable from any caller, which is why the parsing must
be defensive - `JSON.parse('')` throws, and the shipped `|| ''` fallback
guarantees exactly that input when a want carries `url` without `params`.

**Payment order signing — `features/mine/src/main/ets/util/OrderInfoUtil.ets`** (as shipped — **do not copy**, see `HW-17-0014`)

```typescript
export class OrderInfoUtil {
  static readonly APP_ID = '2014100900013222';
  // pkcs8 格式的商户私钥 (the merchant private key, in PKCS8 form) — ~1.6 KB, verbatim in the source
  static readonly RSA2_PRIVATE = 'MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQCa…';

  static async getOrderInfo(): Promise<string> {
    const keyValues = OrderInfoUtil.buildOrderParamMap();
    let orderParam = OrderInfoUtil.buildOrderParam(keyValues);
    let sign = await OrderInfoUtil.getSign(keyValues);   // signs on the device
    return orderParam + '&sign=' + sign;
  }
}
```

**This is the anti-pattern the card exists to flag.** `SignUtils.sign` feeds the
key into `cryptoFramework.createAsyKeyGenerator('RSA2048|PRIMES_2')` and
`createSign('RSA2048|PKCS1|SHA256')` - textbook use of Crypto Architecture Kit,
applied to a secret that must never be in the package. Anyone who unpacks the
HAP can lift the key and mint payment orders for that merchant.

The correct shape is already half-present: `MockApi.getWxPreOrderInfo()` and
`getHuaweiOrderInfo()` return **pre-signed** order strings, which is exactly
what a server would hand back. Route the Alipay path through the same mock API,
delete `OrderInfoUtil` and `SignUtils`, and the client holds no key material at
all.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" },
  {
    "name": "ohos.permission.APP_TRACKING_CONSENT",     // OAID for the splash ad
    "reason": "$string:app_name",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
  // GET_NETWORK_INFO is also declared and never used (HW-17-0021)
]
```

- `APP_TRACKING_CONSENT` is `user_grant` and privacy-sensitive; the doc's
  权限说明 omits it entirely (`HW-17-0016`). Its `reason` points at
  `$string:app_name`, which is a placeholder, not an explanation - replace it.
- `deviceTypes` is phone only; `installationFree` is not set, so this is an app,
  not an atomic service (unlike `FOOD-01`).
- The `client_id` metadata entry is `"xxxx"` with a `todo`; Huawei ID login and
  Ads Kit both fail until it carries the AGC value.
- `aggregated_login` and `aggregated_payment` carry WeChat wrappers with an app
  id constant that must be replaced too.

## Constraints

- API Version 24 Release or later; DevEco Studio 6.1.1 Release or later. This is
  a newer baseline than every other sample in this industry.
- There is no backend: `commons/network` is an axios-shaped mock over static
  arrays in `mocks/MockData/`. Every "request" resolves locally with
  `status: 200`, so no error path is exercised.
- Payment is a demo. The Alipay branch signs locally (`HW-17-0014`), the WeChat
  and Huawei branches return canned pre-order strings.
- The splash ad needs a real ad id and the OAID permission; on a device that
  denies tracking consent, the shipped code proceeds anyway (`HW-17-0019`).
- Images are local resources addressed as `$r('app.media.' + name)`, so adding a
  recipe means adding a media file - there is no remote image loading.
- The whole app is `@ComponentV2` / `@ObservedV2`; mixing these components with
  V1 `@State`/`@Prop` hosts is not supported, so adopt the layer wholesale.

## Pitfalls

- **`HW-17-0014`** (D/high, confirmed): a complete RSA-2048 merchant private key
  is hardcoded in `OrderInfoUtil.RSA2_PRIVATE` and used to sign payment orders on
  the device. Anyone unpacking the HAP can forge orders. Fix: request a
  pre-signed order string from the server and keep no key material in the app.
- **`HW-17-0015`** (B/medium, confirmed): `MinePage` passes a freshly allocated
  arrow function to `emitter.off`, so the `refreshMinePage` subscription is never
  removed and stacks on every rebuild. Fix: store the callback in a field and
  pass the same reference to `on` and `off`.
- **`HW-17-0016`** (E/medium, confirmed): the 权限说明 section lists only
  `INTERNET`, though the entry module declares and requests
  `APP_TRACKING_CONSENT`. Fix: document the tracking permission and its OAID use.
- **`HW-17-0017`** (E/low, confirmed): the 工程目录 tree has `CommonContants.ets`,
  `UploadRecipe.ets` (the file is `UploadRecipePage.ets`) and labels
  `BarChart.ets` as the agreement dialog. Fix: regenerate the tree from the zip.
- **`HW-17-0018`** (B/low, probable): `EntryAbility.resolvePagePath` checks
  `parameters?.url` but then reads `parameters?.params.toString() || ''` and
  feeds the result to `JSON.parse`, which throws on both `undefined` and `''`.
  Any malformed want crashes the launch. Fix: check `params` explicitly and wrap
  the parse in try/catch.
- **`HW-17-0019`** (B/low, confirmed): all three catch blocks in
  `HwAdService.requestOAIDTrackingConsentPermissions` log the constant string
  `ads[0].adType is :` and discard the error, and the permission result is never
  inspected before `identifier.getOAID()`. Fix: log `err.code` / `err.message`
  and branch on the `PermissionRequestResult`.
- **`HW-17-0020`** (E/low, probable): the doc's primary reference for the
  two-level linkage, `harmonyos-guides/arkts-common-list-flow`, does not resolve
  to any page in the crawled documentation tree. Fix: point it at the current
  list-linkage guide.
- **`HW-17-0021`** (D/low, confirmed): `ohos.permission.GET_NETWORK_INFO` is
  declared, undocumented and unused - the network layer is a local mock. Fix:
  remove it.

## References

- `huawei_industry_tree/17_food/docs/02_practice-food-menu-app-demo-v1.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-menu-app-demo-v1-0000002300530042
- `documentation/harmonyos-references/02_application-framework/ts-container-waterflow.md` - `WaterFlow`, `FlowItem`, `columnsTemplate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-waterflow
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-lazyforeach.md` - `IDataSource`, key functions, the notify methods
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-lazyforeach
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `onScrollIndex`, `contentStartOffset`, `Scroller.scrollToIndex`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-container-listitemgroup.md` - grouped lists and the index they report
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-listitemgroup
- `documentation/harmonyos-references/03_system/js-apis-emitter.md` - why `off` needs the registered callback reference
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-emitter
- `documentation/harmonyos-guides/03_application-framework/arkts-new-appstoragev2.md` - `AppStorageV2.connect` for the widget hand-off
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-appstoragev2
- `documentation/harmonyos-guides/03_application-framework/arkts-new-persistencev2.md` - persisting the logged-in user model
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-new-persistencev2
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-cross-package.md` - `router_map.json` per HAR
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-cross-package
- `documentation/harmonyos-guides/07_application-services/account-kit-guide.md` - silent login and the `state` check
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/account-kit-guide
- `documentation/harmonyos-guides/07_application-services/ads-kit-guide.md` - splash ads, OAID and tracking consent
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ads-kit-guide
- `FOOD-01` - the same layering with V1 state, a real Map Kit integration and Preferences persistence
- `FOOD-05` - custom pull-to-refresh and load-more for the waterfall this template renders statically
