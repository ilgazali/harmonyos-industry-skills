---
id: FOOD-01
title: Food ordering atomic service - feature HARs on one Navigation stack, cart in LocalStorage, orders in Preferences
industry: 17_food
doc: huawei_industry_tree/17_food/docs/01_practice-food-app-design-demo-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-design-demo-v1-0000002303530072
sample: huawei_industry_tree/17_food/downloads/Food.zip
kits: ["@kit.AbilityKit", "@kit.AccountKit", "@kit.ArkData", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.FormKit", "@kit.LocationKit", "@kit.MapKit", "@kit.PerformanceAnalysisKit", "@kit.ScanKit", "@kit.TelephonyKit"]
apis: [UIContext, abilityAccessCtrl, authentication, bundleManager, call, common, deviceInfo, formBindingData, formInfo, geoLocationManager, hilog, map]
permissions: [ohos.permission.LOCATION, ohos.permission.APPROXIMATELY_LOCATION, ohos.permission.INTERNET, ohos.permission.GET_NETWORK_INFO, ohos.permission.GYROSCOPE, ohos.permission.ACCELEROMETER]
min_api: 20
modules: [feature_home (har), feature_mine (har), feature_food (har), feature_order (har), feature_map (har), router_manage (har), common_utils (har), phone (entry)]
findings: [HW-17-0001, HW-17-0002, HW-17-0003, HW-17-0004, HW-17-0005, HW-17-0006, HW-17-0007, HW-17-0008, HW-17-0009, HW-17-0010, HW-17-0011, HW-17-0012, HW-17-0013, HW-17-0029]
status: verified-with-fixes
---

## When to use

Load this card when you are building a **single-HAP ordering atomic service**:
one product module, a bottom tab bar, and each business area (home, menu,
orders, mine, store map) shipped as its own HAR. It is the food template, but
the shape is the same for coffee chains, pharmacy pickup, cinema snacks or any
"pick a branch, fill a cart, pay, look at the order" flow.

The load-bearing parts are three: a **routing HAR** that lets every feature HAR
push pages onto one shared `NavPathStack` without importing the pages; a **cart
that lives in `LocalStorage`**, not in a component, so the menu page, the detail
page and the checkout page all mutate the same map; and a **store picker driven
by Map Kit and Location Kit**, which picks the nearest branch on first launch
and then remembers the user's choice.

This is a template, not a product. Payment is a `setTimeout`, the menu and the
store list are hardcoded arrays, and there is no backend at all. Copy the module
layout, the router and the state plumbing; do not copy the navigation
(`HW-17-0009`), the permission utility (`HW-17-0004`) or the shop lookup
(`HW-17-0001`) unchanged - all three are broken in ways that only show up once
real data flows through them.

## Feature checklist

- Four bottom tabs - 首页 (home), 点餐 (order food), 订单 (orders), 我的 (mine)
  - built once inside a single `NavDestination`.
- Home offers 外卖 (delivery) and 堂食 (dine-in) entries; delivery jumps to the
  address list, dine-in switches to the menu tab.
- The menu page resolves the current location, preselects the nearest store and
  shows the distance in metres; a segmented button switches dine-in/delivery,
  and choosing delivery with no address opens the address page.
- Left category rail and right dish list scroll in step; add/remove buttons
  update a cart badge and a running total, and the cart sheet can be emptied in
  one tap. A dish detail page adds several units at once.
- The store page shows a map with a marker per branch, a collapsible map area,
  a call button, a "route in Petal Maps" button and an "order here" button.
- Checkout writes the order into Preferences and lands the user on the order
  tab, where completed orders are listed with their items.
- 我的 offers Huawei ID login and a QR scan that jumps straight into the menu of
  the scanned store.

## Architecture

Eight modules: one entry HAP under `product/`, five feature HARs, two common
HARs. All in one DevEco project, wired by relative `file:` dependencies.

```
Food
├── common
│   ├── common_utils/src/main/ets/
│   │   ├── model/OrderType.ets             DINE_IN | TAKEOUT enum, shared by three HARs
│   │   └── utils/{Logger,MathUtil,PermissionsUtil,PreferenceUtil}.ets
│   │                                       money maths, permissions, Preferences, toasts
│   └── router_manage/src/main/ets/
│       ├── commons/{NavRouterMap,NavStackMap,AnimateMap}.ets  page/stack name enums
│       ├── models/NavRouterInfo.ets        {stackName, url, param, animateSwitch}
│       └── models/RouterModule.ets         static Map<string, NavPathStack> + push/replace/pop
├── feature
│   ├── feature_food/…/components/FoodCategory.ets   the menu, the cart and the location gate
│   ├── feature_food/…/model/FoodModelData.ts        682 lines of hardcoded dishes per store
│   ├── feature_food/…/pages/PageFoodDetail.ets      dish detail, add N to cart, pop
│   ├── feature_home/…/components/HomeComponent.ets  banner + delivery/dine-in tiles
│   ├── feature_home/…/pages/{PageTakeOut,PageAddAddress}.ets  delivery addresses
│   ├── feature_map/…/model/ShopModelData.ts         four branches with WGS84 coordinates
│   ├── feature_map/…/utils/MapUtil.ets              WGS84 -> GCJ02 conversion
│   ├── feature_map/…/pages/PageShopList.ets         MapComponent + markers + branch cards
│   ├── feature_mine/…/components/MineComponent.ets  Huawei ID login + scan-to-store
│   ├── feature_order/…/components/PageOrderList.ets order history read from Preferences
│   └── feature_order/…/pages/PageOrderPreview.ets   checkout, simulated payment, order write
└── product/phone/src/main/ets/
    ├── phoneability/PhoneAbility.ets       full screen + avoid areas -> AppStorage
    ├── phoneformability/PhoneFormAbility.ets  service-widget entry
    ├── food/pages/FoodCard.ets             the widget's own page
    ├── pages/Index.ets                     @Entry: creates the stack, replaces to PageMain
    └── pages/Main.ets                      the four tabs, registered as PageMain
```

The documented tree does **not** match the zip: the doc writes `models/` for all
five feature modules where the zip has `model/` (`HW-17-0008`). Everything else
in the tree lines up.

**The design decision worth copying** is `RouterModule` plus one
`route_map.json` per HAR. Each feature HAR declares its own pages in
`src/main/resources/base/profile/route_map.json` (`PageShopList` in
`feature_map`, `PageOrderPreview` in `feature_order`, and so on) and exports a
`@Builder` function named in that file. The entry HAP declares only `PageMain`.
Navigation is then done by **name**, never by import:

```typescript
RouterModule.push({
  stackName: NavStackMap.MAIN_STACK,
  url: NavRouterMap.PAGE_SHOP_LIST,
  animateSwitch: AnimatedMap.ON
});
```

`feature_food` can send the user to a page that lives in `feature_map` without a
compile-time dependency on it, because `pushPathByName` resolves through the
system's merged router map - that is what keeps the feature HARs independently
buildable. The sample then throws part of it away: `feature_food` depends on
`feature_map`, `feature_order` on `feature_food`, `feature_mine` on
`feature_map`, purely to reach the `ShopDetail` and `FoodDetail` **types**
(`HW-17-0012`). Move the shared models into a common data HAR and the horizontal
edges disappear; the router already handles the rest.

## Implementation steps

1. **Create the project as one entry HAP plus HARs.** `product/phone` depends on
   every feature and common HAR; feature HARs depend only on `common_utils` and
   `router_manage`. Keep shared data types in the common layer, not in a feature
   HAR (`HW-17-0012`).
2. **Register the stack once.** `Index.ets` creates a `NavPathStack`, hands it to
   `RouterModule.createStack(NavStackMap.MAIN_STACK, …)` in `aboutToAppear`,
   then `replace`s to `PageMain` so the tab page is the stack root.
3. **Give every HAR a `route_map.json`** and export the matching
   `@Builder function XxxBuilder()`; reference the profile from the HAR's
   `module.json5` as `"routerMap": "$profile:route_map"`.
4. **Publish window insets through `AppStorage`.** `PhoneAbility` reads the
   navigation-indicator and cutout avoid areas after `loadContent` resolves and
   writes `bottomRectHeight` / `topRectHeight`; pages read them with
   `AppStorage.get<number>()` and apply them as padding.
5. **Put the cart in `LocalStorage`,** not in a component: `buyFoods`
   (`Map<string, FoodDetail>`), `choosePrice`, `orderType`, `chooseShop` and
   `chooseTakeOutAddr` are all `@LocalStorageLink`, and every price change goes
   through `MathUtil.addition` / `subtraction` so a running ￥ total stays exact.
6. **Declare both location permissions** in `module.json5`.
   `ohos.permission.LOCATION` cannot be declared alone; the doc's instruction to
   add only it is wrong (`HW-17-0006`). Declare nothing else you do not call
   (`HW-17-0005`). Accumulate the check across the array instead of overwriting
   it per iteration, and await the request (`HW-17-0004`).
7. **Convert coordinates before they touch Map Kit.** The store list is WGS84;
   `map.convertCoordinateSync(WGS84, GCJ02, …)` is applied for the camera target
   and for every marker.
8. **Pick the nearest store only when none is chosen,** using
   `map.calculateDistance`; once `chooseShop` is set, recompute only that store's
   distance so a user's explicit choice survives. Guard the lookup -
   `getShopByName` can return `undefined` (`HW-17-0001`).
9. **Persist the order with `putSync` + `flush`** under a named store
   (`food.order`), keyed by timestamp, `await` the flush, and disable the pay
   button while the payment is in flight (`HW-17-0011`).
10. **Return to the tab page with `popToName`, never `push`** (`HW-17-0009`), and
    reload the order list when the order tab becomes visible rather than only in
    `aboutToAppear` (`HW-17-0010`).

## Verified snippets

All snippets are from `Food.zip`. Corrected forms are marked.

**The cart mutation — `feature/feature_food/src/main/ets/components/FoodCategory.ets`** (as shipped)

```typescript
@LocalStorageLink('choosePrice') choosePrice: number = 0;
@LocalStorageLink('buyFoods') buyFoods: Map<string, FoodDetail> = new Map<string, FoodDetail>();
@State chickShop: boolean = false;

@Builder
buyNumBuilder(foodDetail: FoodDetail) {
  if (this.buyFoods.get(foodDetail.id)) {
    Image($r('app.media.ic_public_remove'))
      .width(25).height(25).objectFit(ImageFit.Fill)
      .onClick(() => {
        let item = this.buyFoods.get(foodDetail.id);
        if (item) {
          if (item.buyNum > 1) {
            item.buyNum--;
            this.buyFoods.set(foodDetail.id, item);
            this.choosePrice = subtraction(this.choosePrice, foodDetail.price);
            if (this.choosePrice === 0) {
              this.chickShop = false;              // close the cart sheet when it empties
            }
          } else if (item.buyNum === 1) {
            item.buyNum--;
            this.buyFoods.delete(foodDetail.id);     // last unit: drop the entry, not a zero row
            this.choosePrice = subtraction(this.choosePrice, foodDetail.price);
            if (this.choosePrice === 0) {
              this.chickShop = false;
            }
          }
        }
      });
  }

  if (this.buyFoods.get(foodDetail.id)) {
    Text(this.buyFoods.get(foodDetail.id)?.buyNum + '').fontSize(16).margin({ left: 10, right: 10 });
  }

  Image($r('app.media.ic_public_add_norm'))
    .width(25).height(25).objectFit(ImageFit.Fill)
    .onClick(() => {
      this.choosePrice = addition(this.choosePrice, foodDetail.price);
      let item = this.buyFoods.get(foodDetail.id);
      if (item) {
        item.buyNum++;
        this.buyFoods.set(foodDetail.id, item);
      } else {
        foodDetail.buyNum = 1;
        this.buyFoods.set(foodDetail.id, foodDetail);
      }
    });
}
```

**Three things carry the design.** The builder is the *whole* stepper - minus
button, count, plus button - and it is called from the dish row, the cart-sheet
row and the detail page, so one definition governs every place a quantity can
change. The cart is a `@LocalStorageLink` map keyed by dish id, so
`PageFoodDetail` in the same HAR and `PageOrderPreview` in another see the same
instance with no parameter passing. And every price change goes through
`addition` / `subtraction` from `common_utils`, which scale both operands to
integers first - the only reliable way to keep a running ￥ total exact.

Note `this.buyFoods.set(...)` after mutating `item`: a `Map` in `LocalStorage`
does not observe in-place field writes, so re-setting the key is what triggers
the re-render. The document's copy of this builder references `this.checkShop`
and `item.buyNum == 1`, neither of which exists in the sample (`HW-17-0007`).

The same file's `getRecentShop()` ranks the four branches with
`map.calculateDistance(fromLatLng, toLatLng)` - plain `LatLng` in, metres out,
no map instance needed - but only when `chooseShop` is still `''`. Afterwards it
recomputes the distance for the chosen branch alone, so a user who picked a
store by hand or by QR scan is never silently moved. The menu itself is derived
from the store (`getCategoryListByShop`), which is why switching stores should
also revalidate the cart; the sample leaves a `todo` there and does not.

**The store page — `feature/feature_map/src/main/ets/pages/PageShopList.ets`** (corrected, see `HW-17-0001`, `HW-17-0002`, `HW-17-0003`)

```typescript
aboutToAppear(): void {
  this.preChooseShop = this.chooseShop;
  this.shopList = shopDetailList;

  // FIX: getShopByName returns ShopDetail | undefined and chooseShop starts as ''
  const shop: ShopDetail = getShopByName(this.chooseShop) ?? shopDetailList[0];
  this.latitude = shop.latitude;
  this.longitude = shop.longitude;
  let gcj02Position = MapUtil.convertToGcj02(this.latitude, this.longitude);

  this.mapOptions = {
    position: {
      target: { latitude: gcj02Position.latitude, longitude: gcj02Position.longitude },
      zoom: 16,
    },
    myLocationControlsEnabled: true
  };

  this.callback = async (err, mapController) => {
    if (!err) {
      this.mapController = mapController;
      for (let i = 0; i < this.shopList.length; i++) {
        let gcj02 = MapUtil.convertToGcj02(this.shopList[i].latitude, this.shopList[i].longitude);
        let markerOptions: mapCommon.MarkerOptions = {
          position: { latitude: gcj02.latitude, longitude: gcj02.longitude },
          anchorU: 0.5, anchorV: 1,                    // pin tip sits on the coordinate
          rotation: 0, visible: true, zIndex: 0, alpha: 1, clickable: true, draggable: true, flat: false
        };
        this.mapController.addMarker(markerOptions);   // only valid inside this callback
      }
    }
  };
  // build(): MapComponent({ mapOptions: this.mapOptions, mapCallback: this.callback });
}

// branch card tap
.onClick(() => {
  this.preChooseShop = detail.name;
  let gcj02Position = MapUtil.convertToGcj02(detail.latitude, detail.longitude);
  this.latitude = gcj02Position.latitude;           // FIX: sample assigns longitude here
  this.longitude = gcj02Position.longitude;
  let cameraUpdate: map.CameraUpdate = map.newCameraPosition({
    target: { latitude: gcj02Position.latitude, longitude: gcj02Position.longitude },
    zoom: 16
  });
  this.mapController?.animateCamera(cameraUpdate, 1000);
  // FIX: the sample also calls moveCamera(cameraUpdate) here, which cancels the animation
});
```

**`MapComponent` is initialised by options plus a callback, not by an imperative
"create map" call.** `mapOptions` fixes the first camera position - which is why
it must be computed before the first build, and why the unguarded
`getShopByName` there is a crash on entry - and `mapCallback` hands you the
`MapComponentController` once the surface exists. Markers can only be added
inside that callback.

Coordinates must be converted exactly once: `shopDetailList` holds WGS84 (the
frame the GNSS chip reports), `mapCommon.CoordinateType.GCJ02` is what Map Kit
draws in, and a missing conversion puts every pin a few hundred metres off in
mainland China. The sample gets that right everywhere it draws - and then
corrupts its own `latitude` state with a longitude on the card tap
(`HW-17-0002`).

**Permission check — `common/common_utils/src/main/ets/utils/PermissionsUtil.ets`** (corrected, see `HW-17-0004`)

```typescript
async checkPermissions(permissions: Array<Permissions>, context: Context): Promise<boolean> {
  let applyResult: boolean = true;                       // FIX: sample starts false
  for (let permission of permissions) {
    let grantStatus: abilityAccessCtrl.GrantStatus = await this.checkAccessToken(permission);
    // FIX: sample assigns per iteration, so only the last permission counts
    applyResult = applyResult && (grantStatus === abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED);
  }
  if (!applyResult) {
    // FIX: sample fires and forgets, so callers never learn the outcome
    const result: PermissionRequestResult = await this.requestPermissions(permissions, context);
    return result.authResults.every((code: number) => code === 0);
  }
  return applyResult;
}

async checkAccessToken(permission: Permissions): Promise<abilityAccessCtrl.GrantStatus> {
  let atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
  let grantStatus: abilityAccessCtrl.GrantStatus = abilityAccessCtrl.GrantStatus.PERMISSION_DENIED;
  let tokenId: number = 0;
  try {
    let bundleInfo: bundleManager.BundleInfo =
      await bundleManager.getBundleInfoForSelf(bundleManager.BundleFlag.GET_BUNDLE_INFO_WITH_APPLICATION);
    tokenId = bundleInfo.appInfo.accessTokenId;
  } catch (error) {
    let err: BusinessError = error as BusinessError;
    Logger.error(`Failed to get bundle info for self. Code is ${err.code}, message is ${err.message}`);
    return grantStatus;                                  // FIX: sample falls through with tokenId 0
  }
  return await atManager.checkAccessToken(tokenId, permission);
}
```

**Two location permissions are granted independently,** so the loop must be an
AND over all of them. The shipped version writes `applyResult = true` or
`false` on each pass; with the approximate permission granted and the precise
one denied - the ordinary state after a user taps "approximate only" - the last
iteration wins and the app concludes it has everything it needs.

`checkAccessToken` needs the app's own token id, and the only route to it is
`getBundleInfoForSelf` with `GET_BUNDLE_INFO_WITH_APPLICATION`; without that
flag `appInfo` is not populated. The menu page adds the permanent-refusal path:
`atManager.requestPermissionOnSetting(context, permissions)` behind the 开启定位
(turn on location) button, which opens the settings sheet when the ordinary
request would no longer show a dialog.

**Checkout — `feature/feature_order/src/main/ets/pages/PageOrderPreview.ets`** (corrected, see `HW-17-0009`, `HW-17-0011`)

```typescript
.onClick(async () => {
  if (this.enableLoading) {
    return;                                       // FIX: absent - a second tap writes a second order
  }
  this.enableLoading = true;

  setTimeout(async () => {
    this.enableLoading = false;
    let key = Date.parse(new Date().toString()) + '';
    let value: FoodDetail[] = [];
    this.buyFoods.forEach((detail: FoodDetail) => value.push(detail));

    let dataPreferences: preferences.Preferences | null =
      PreferenceUtil.getPreference(this.context, 'food.order');
    dataPreferences.putSync(key,
      JSON.stringify(new OrderDetail('FD' + key, this.choosePrice, this.orderType, this.chooseShop,
        this.chooseTakeOutAddr, key, OrderStatus.COMMITED, value)));
    await dataPreferences.flush();                // FIX: sample drops the promise

    this.choosePrice = 0;
    this.buyFoods.clear();
    this.selectedIndex = 2;                       // land on the order tab
    // FIX: sample pushes PAGE_MAIN, stacking another copy of the tab page
    RouterModule.popToName(NavStackMap.MAIN_STACK, NavRouterMap.PAGE_MAIN, true);

    PreferenceUtil.showToastMessage('下单成功', this.getUIContext());
  }, 2000);
});
```

**`putSync` writes to the in-memory cache; only `flush` reaches the file.**
Awaiting it before navigating is what makes the order survive an immediate kill.

**`popToName` is the correct return.** `MainPage` is already the bottom of this
stack - `Index.ets` put it there with `replace` - so pushing it again builds a
second, third, fourth copy, each with its own four tabs and its own stale cart,
and the back gesture then walks the user through those ghosts. `popToName`
unwinds to the existing instance, and because `selectedIndex` is an
`@StorageLink`, setting it to 2 before popping lands on the order tab.

One consequence to plan for: once you stop pushing a fresh `MainPage`, the order
tab no longer rebuilds, and `PageOrderList` only reads Preferences in
`aboutToAppear` (`HW-17-0010`). Reload it from `Tabs.onChange` or
`NavDestination.onShown`.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.LOCATION",                  // precise, metre-level
    "reason": "$string:permission_reason",
    "usedScene": { "abilities": ["PhoneAbility"], "when": "inuse" } },
  { "name": "ohos.permission.APPROXIMATELY_LOCATION",    // required alongside LOCATION
    "reason": "$string:permission_reason",
    "usedScene": { "abilities": ["PhoneAbility"], "when": "inuse" } },
  { "name": "ohos.permission.INTERNET" }
  // GET_NETWORK_INFO, GYROSCOPE and ACCELEROMETER are declared but never used (HW-17-0005)
]
```

- Both location permissions are `user_grant`, so `reason` and `usedScene` are
  mandatory and the reason string resource must exist. `LOCATION` is not
  declarable on its own; the document says to add only it (`HW-17-0006`).
- `when: "inuse"` is right: the app resolves a position when the menu tab
  appears and holds nothing in the background.
- The module also declares `"installationFree": true` (atomic service) and a
  `client_id` metadata entry that must carry the AGC value before Huawei ID
  login or Map Kit will work.
- Every HAR declares its own `route_map.json` profile; the entry HAP declares
  only `PageMain`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `deviceTypes` is phone/tablet/2in1 but
  the layouts are phone-only.
- Map Kit needs the map service switched on in AppGallery Connect and a manual
  signing profile; without it `MapComponent` renders empty. The location gate
  also refuses the emulator (`deviceInfo.productModel === 'emulator'` shows a
  请使用真机运行 "run on a real device" toast).
- Menu and store data are hardcoded (`FoodModelData.ts`, `ShopModelData.ts`);
  there is no network layer, and payment is a 2-second `setTimeout`.
- The scan-to-store flow matches the raw QR text against a store **name**, so
  any real deployment needs App Linking URLs instead (see the doc's 扫码拉起元服务
  section).
- Preferences is the only persistence: orders are keyed by timestamp in the
  `food.order` store and are never pruned.

## Pitfalls

- **`HW-17-0001`** (B/high, confirmed): `getShopByName()` is typed `: ShopDetail`
  but returns `undefined` on no match, and `PageShopList.aboutToAppear`
  dereferences it while `chooseShop` is still `''` - entering the store page
  before a store is picked throws. Fix: type it `ShopDetail | undefined` and fall
  back to `shopDetailList[0]`.
- **`HW-17-0002`** (B/low, confirmed): the branch-card tap assigns
  `this.latitude = gcj02Position.longitude`. Fix: assign the latitude.
- **`HW-17-0003`** (B/low, probable): `animateCamera(cameraUpdate, 1000)` is
  immediately followed by `moveCamera(cameraUpdate)`, which jumps to the target
  and cancels the animation. Fix: keep one of the two.
- **`HW-17-0004`** (B/medium, confirmed): `PermissionsUtil.checkPermissions`
  overwrites its result on every loop pass, so only the last permission counts,
  and the follow-up request is fire-and-forget. Fix: accumulate with `&&` and
  await the request.
- **`HW-17-0005`** (D/medium, confirmed): `GYROSCOPE`, `ACCELEROMETER` and
  `GET_NETWORK_INFO` are declared but no source file uses a sensor or
  network-info API. Fix: remove them - unused permissions fail app review.
- **`HW-17-0006`** (E/high, confirmed): the document instructs readers to declare
  only `ohos.permission.LOCATION`, which the Location Kit permission guide lists
  as not declarable alone. Fix: document the pair, as the sample already does.
- **`HW-17-0007`** (E/medium, confirmed): the 点餐代码实现 snippet uses
  `this.checkShop` and `item.buyNum == 1` where the sample has `chickShop` and
  `===`, so the documented code does not compile. Fix: regenerate it from
  `FoodCategory.ets`.
- **`HW-17-0008`** (E/low, confirmed): the documented tree says `models/` for all
  five feature HARs; the zip uses `model/`. Fix: regenerate the tree.
- **`HW-17-0009`** (B/medium, confirmed): four call sites (`PageOrderPreview`,
  `PageShopList`, `PageTakeOut`, `MineComponent`) return to the tab page with
  `push(PAGE_MAIN)` while a `MainPage` already sits at the bottom of the same
  stack, so the stack grows without bound and back walks through stale copies.
  Fix: `RouterModule.popToName(MAIN_STACK, PAGE_MAIN)`.
- **`HW-17-0010`** (B/low, probable): `PageOrderList` reads Preferences only in
  `aboutToAppear`, inside a `TabContent` built once - it looks fresh today only
  because of the bug above. Fix: reload on tab change or `onShown`.
- **`HW-17-0011`** (B/low, probable): the pay button stays clickable during the
  2-second simulated payment and `flush()` is not awaited, so a double tap
  writes two orders. Fix: guard on `enableLoading` and await the flush.
- **`HW-17-0012`** (E/medium, confirmed): the feature HARs depend on each other
  (`feature_food` → `feature_map`, `feature_order` → `feature_food`,
  `feature_mine` → `feature_map`) to reach shared types, contradicting the doc's
  own layered software view. Fix: move `ShopDetail` / `FoodDetail` into the
  common layer.
- **`HW-17-0013`** (E/low, probable): the 历史工程迁移 link points at
  `harmonyos-guides-V14/ide-integrated-project-migration-V14`, a slug absent from
  the crawled navigation trees, and a sibling industry doc links the same
  capability elsewhere. Fix: verify the live slug and align both docs.

## References

- `huawei_industry_tree/17_food/docs/01_practice-food-app-design-demo-v1.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-design-demo-v1-0000002303530072
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-cross-package.md` - `route_map.json` per HAR and `pushPathByName` across modules
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-cross-package
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `NavPathStack`, `popToName`, `replacePathByName`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-guides/03_application-framework/data-persistence-by-preferences.md` - `putSync` / `flush` semantics
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/data-persistence-by-preferences
- `documentation/harmonyos-guides/07_application-services/map-kit-guide.md` - `MapComponent`, markers, camera updates
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-kit-guide
- `documentation/harmonyos-guides/07_application-services/map-calculation-tool.md` - `calculateDistance` and `convertCoordinateSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-calculation-tool
- `documentation/harmonyos-guides/07_application-services/location-kit.md` - `geoLocationManager.getCurrentLocation`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/location-kit
- `documentation/harmonyos-guides/07_application-services/location-permission-guidelines.md` - why `LOCATION` cannot be declared alone
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/location-permission-guidelines
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` - `checkAccessToken`, `requestPermissionsFromUser`, `requestPermissionOnSetting`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-guides/05_media/scan-directservice.md` - scan-to-atomic-service, the production form of the QR flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/scan-directservice
- `documentation/harmonyos-guides/03_application-framework/app-linking-startup.md` - the links a real QR code should carry
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/app-linking-startup
- `FOOD-04` - the store-detail map and Petal Maps handoff this template stops short of
- `FOOD-02` - the same layering done with V2 state and a mock network layer
