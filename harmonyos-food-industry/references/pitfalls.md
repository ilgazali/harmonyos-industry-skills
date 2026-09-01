# Pitfalls

> Generated from `features/*.md`. Source industry: `17_food`, 6 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (1)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-17-0029` - Systematic: 4 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: FOOD-01, FOOD-02, FOOD-04, FOOD-05
- Document: `huawei_industry_tree/17_food/docs/05_custom_refresh.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_refresh-0000002331948181
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (29)

### `HW-17-0001` - getShopByName() can return undefined and PageShopList dereferences it without a null check, crashing on first entry

- Category B, severity high, confidence confirmed
- Features: FOOD-01
- Document: `huawei_industry_tree/17_food/docs/01_practice-food-app-design-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-design-demo-v1-0000002303530072
- Why: Dereferencing a possibly-undefined return value without a guard throws a TypeError at runtime; the declared return type also lies about nullability.
- Fix: Type the function as `ShopDetail | undefined`; in PageShopList.aboutToAppear use a fallback such as `const shop = getShopByName(this.chooseShop) ?? shopDetailList[0]`.

### `HW-17-0006` - Document tells developers to declare only ohos.permission.LOCATION, which the official guide says is not declarable alone

- Category E, severity high, confidence confirmed
- Features: FOOD-01
- Document: `huawei_industry_tree/17_food/docs/01_practice-food-app-design-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-design-demo-v1-0000002303530072
- Why: Following the document verbatim produces a module.json5 that fails permission declaration rules; the doc contradicts both the official guide and its own sample project.
- Fix: Update the sentence to require both permissions, matching the sample project and the Location Kit permission guide.

### `HW-17-0014` - Full RSA-2048 merchant private key is hardcoded in client code and payment orders are signed on-device

- Category D, severity high, confidence confirmed
- Features: FOOD-02
- Document: `huawei_industry_tree/17_food/docs/02_practice-food-menu-app-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-menu-app-demo-v1-0000002300530042
- Why: A merchant signing key embedded in a shipped package can be extracted by anyone unpacking the HAP, allowing forged payment orders; payment-order signing must happen on the merchant server. A template teaches this pattern to every developer copying it.
- Fix: Move order signing behind the mock API layer and store no key material in the app; add a comment stating the key must live server-side.

### `HW-17-0022` - Sample declares no permissions at all although the document (and the static-map network call) require ohos.permission.INTERNET

- Category D, severity high, confidence confirmed
- Features: FOOD-04
- Document: `huawei_industry_tree/17_food/docs/04_map_navigation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_navigation-0000002248154336
- Why: Without the INTERNET declaration every network request from the app is denied, so the core feature (static map image) cannot work; the sample contradicts its own permission section.
- Fix: Add requestPermissions with ohos.permission.INTERNET to entry/src/main/module.json5.

### `HW-17-0030` - OrderInfoUtil.getSign sorts payment parameters by first character into a map it then discards, and signs the unsorted parameters with a hardcoded 2016 timestamp

- Category B, severity high, confidence confirmed
- Features: FOOD-02
- Document: `huawei_industry_tree/17_food/docs/02_practice-food-menu-app-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-menu-app-demo-v1-0000002300530042
- Why: Three separate defects in one signing routine. First, mapEntries holds key strings, so a[0].localeCompare(b[0]) compares only the first character of each key. Second, sortedKeyValues is built from entry[0] and entry[1], which are the first and second characters of each key string rather than a key and a value, and the map is then never read - the signature is computed from keyValues in insertion order. Third, the timestamp is a fixed literal from 2016. Alipay requires the signed parameter string to be sorted by key and rejects stale timestamps, so this signature cannot validate. The routine appears to work only because the demo never reaches a real gateway.
- Fix: Sort the entries by full key with a.localeCompare(b), build the parameter string from the sorted entries, delete the unused sortedKeyValues map, and set timestamp from the current time in the format the gateway expects. Note that the signing itself belongs on the merchant server, as already recorded in HW-17-0014.

### `HW-17-0004` - PermissionsUtil.checkPermissions honors only the last permission in the list and ignores the request result

- Category B, severity medium, confidence confirmed
- Features: FOOD-01
- Document: `huawei_industry_tree/17_food/docs/01_practice-food-app-design-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-design-demo-v1-0000002303530072
- Why: If the last permission is granted but an earlier one is denied, the denied permission is never requested; callers also cannot know whether the request succeeded.
- Fix: Use `applyResult = applyResult && (grantStatus === PERMISSION_GRANTED)` (initialized to true) or `Array.every`, and return/await the requestPermissionsFromUser promise.

### `HW-17-0005` - Unused permissions GYROSCOPE, ACCELEROMETER and GET_NETWORK_INFO are declared in module.json5

- Category D, severity medium, confidence confirmed
- Features: FOOD-01
- Document: `huawei_industry_tree/17_food/docs/01_practice-food-app-design-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-design-demo-v1-0000002303530072
- Why: Declaring permissions the app never uses violates the least-privilege principle for app permissions and will be flagged in app review.
- Fix: Remove the GYROSCOPE, ACCELEROMETER and GET_NETWORK_INFO entries from requestPermissions.

### `HW-17-0007` - Ordering code snippet in the document does not match the shipped sample code (checkShop vs chickShop)

- Category E, severity medium, confidence confirmed
- Features: FOOD-01
- Document: `huawei_industry_tree/17_food/docs/01_practice-food-app-design-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-design-demo-v1-0000002303530072
- Why: Copy-pasting the documented snippet into the sample project does not compile; doc and sample must stay in sync.
- Fix: Regenerate the 点餐代码实现 snippet from the current FoodCategory.ets.

### `HW-17-0009` - Returning to the main page is implemented as push(PAGE_MAIN), stacking new MainPage instances instead of popping

- Category B, severity medium, confidence confirmed
- Features: FOOD-01
- Document: `huawei_industry_tree/17_food/docs/01_practice-food-app-design-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-design-demo-v1-0000002303530072
- Why: The navigation stack grows unboundedly; the system back gesture then walks through stale MainPage copies (old tab state, old cart), and each copy re-creates all four tabs, wasting memory.
- Fix: Replace these calls with RouterModule.popToName(NavStackMap.MAIN_STACK, NavRouterMap.PAGE_MAIN) (or clear + keep the root).

### `HW-17-0012` - Sample project has cross-feature HAR dependencies, contradicting the document's own layered software view

- Category E, severity medium, confidence confirmed
- Features: FOOD-01
- Document: `huawei_industry_tree/17_food/docs/01_practice-food-app-design-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-design-demo-v1-0000002303530072
- Why: The referenced layered-architecture practice defines features as high-cohesion low-coupling modules consumed by the product layer, with shared capabilities in the common layer; shared shop/food data living inside feature HARs forces feature-to-feature coupling the doc's own architecture diagram does not show.
- Fix: Move ShopModelData/FoodDetail types into common_utils (or a new common data HAR) and drop the cross-feature dependencies.

### `HW-17-0015` - MinePage passes a new arrow function to emitter.off, so the refreshMinePage subscription is never unregistered

- Category B, severity medium, confidence confirmed
- Features: FOOD-02
- Document: `huawei_industry_tree/17_food/docs/02_practice-food-menu-app-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-menu-app-demo-v1-0000002300530042
- Why: The off() call receives a different function instance than the one registered, so per the emitter API contract nothing is unregistered; every aboutToAppear stacks another live subscription that fires on a possibly-destroyed component.
- Fix: Store the callback in a field (`private refreshCb = () => this.vm.refreshMinePage();`) and pass it to both on() and off().

### `HW-17-0016` - Permission section documents only INTERNET but the sample also declares and requests APP_TRACKING_CONSENT

- Category E, severity medium, confidence confirmed
- Features: FOOD-02
- Document: `huawei_industry_tree/17_food/docs/02_practice-food-menu-app-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-menu-app-demo-v1-0000002300530042
- Why: The 权限说明 section is the authoritative permission list of the template; omitting a user-grant privacy permission (tracking consent) misleads developers and privacy reviewers.
- Fix: Add ohos.permission.APP_TRACKING_CONSENT to the 权限说明 section.

### `HW-17-0002` - Copy-paste bug: latitude state is assigned the longitude value in PageShopList shop-card click handler

- Category B, severity low, confidence confirmed
- Features: FOOD-01
- Document: `huawei_industry_tree/17_food/docs/01_practice-food-app-design-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-design-demo-v1-0000002303530072
- Why: Both state fields receive the longitude, so the stored latitude is wrong; any future consumer of this state gets corrupt coordinates.
- Fix: Assign latitude from gcj02Position.latitude.

### `HW-17-0003` - animateCamera() is immediately cancelled by a redundant moveCamera() call with the same CameraUpdate

- Category B, severity low, confidence probable
- Features: FOOD-01
- Document: `huawei_industry_tree/17_food/docs/01_practice-food-app-design-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-design-demo-v1-0000002303530072
- Why: moveCamera jumps instantly to the target, so the 1000 ms animation started on the previous line never plays; the two calls are contradictory.
- Fix: Remove the moveCamera call (or the animateCamera call if no animation is wanted).

### `HW-17-0008` - Project tree in the document uses 'models/' directories that are named 'model/' in the sample zip

- Category E, severity low, confidence confirmed
- Features: FOOD-01
- Document: `huawei_industry_tree/17_food/docs/01_practice-food-app-design-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-design-demo-v1-0000002303530072
- Why: The documented project structure must match the sample project so readers can locate files.
- Fix: Update the 代码结构解读 tree to use model/ for the five feature modules.

### `HW-17-0010` - Order list is loaded only in aboutToAppear, so an already-built order tab never refreshes after a new order

- Category B, severity low, confidence probable
- Features: FOOD-01
- Document: `huawei_industry_tree/17_food/docs/01_practice-food-app-design-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-design-demo-v1-0000002303530072
- Why: Once the push-to-main navigation is fixed, completed orders would silently never appear in the order tab; data load is tied to the wrong lifecycle.
- Fix: Reload the list in Tabs.onChange for index 2, or expose a refresh triggered by an @StorageLink flag written after payment.

### `HW-17-0011` - Pay button has no double-tap guard during the 2-second simulated payment, allowing duplicate orders

- Category B, severity low, confidence probable
- Features: FOOD-01
- Document: `huawei_industry_tree/17_food/docs/01_practice-food-app-design-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-design-demo-v1-0000002303530072
- Why: Un-debounced payment actions create duplicate orders; the loading state is visual only and does not disable the trigger.
- Fix: Guard with `if (this.enableLoading) return;` at the top of onClick (or .enabled(!this.enableLoading) on the button) and await flush().

### `HW-17-0013` - The '历史工程迁移' capability is linked to two different slugs in two industry docs, neither of which exists in the documentation navigation tree

- Category E, severity low, confidence probable
- Features: FOOD-01
- Document: `huawei_industry_tree/17_food/docs/01_practice-food-app-design-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-design-demo-v1-0000002303530072
- Why: Two docs pointing the same feature at two different, apparently nonexistent targets indicates at least one (likely both) links is stale; readers cannot reach the migration guide from either practice doc.
- Fix: Verify the current slug of the project-migration guide on the live site and update both docs to it.

### `HW-17-0017` - Project tree in the document has wrong file names and a wrong comment compared with the zip

- Category E, severity low, confidence confirmed
- Features: FOOD-02
- Document: `huawei_industry_tree/17_food/docs/02_practice-food-menu-app-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-menu-app-demo-v1-0000002300530042
- Why: The documented project structure must match the sample project; wrong names and copy-pasted comments send readers to files that do not exist.
- Fix: Fix the three entries in the 工程目录 tree.

### `HW-17-0018` - EntryAbility.resolvePagePath parses want parameters without guarding, crashing on malformed form-card wants

- Category B, severity low, confidence probable
- Features: FOOD-02
- Document: `huawei_industry_tree/17_food/docs/02_practice-food-menu-app-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-menu-app-demo-v1-0000002300530042
- Why: onCreate/onNewWant run for every launch including third-party wants; unvalidated parameters can crash the ability at startup.
- Fix: Guard with `if (parameters?.url && typeof parameters.params === 'string')` and wrap JSON.parse in try/catch.

### `HW-17-0019` - Ad service catch blocks log a meaningless copy-pasted string and discard the real error

- Category B, severity low, confidence confirmed
- Features: FOOD-02
- Document: `huawei_industry_tree/17_food/docs/02_practice-food-menu-app-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-menu-app-demo-v1-0000002300530042
- Why: Failures in the permission/OAID/ad-request chain become undiagnosable; the log text refers to an unrelated variable and hides denial vs. API error.
- Fix: Log `err.code/err.message` in each catch and branch on the PermissionRequestResult before getOAID().

### `HW-17-0020` - Core reference link for the two-level linked list (arkts-common-list-flow) does not resolve to any page in the documentation tree

- Category E, severity low, confidence probable
- Features: FOOD-02
- Document: `huawei_industry_tree/17_food/docs/02_practice-food-menu-app-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-menu-app-demo-v1-0000002300530042
- Why: This link is the doc's primary how-to reference for its second implementation step; if it is dead, readers lose the underlying pattern documentation the step depends on.
- Fix: Point the link at the current guide for list linkage scenarios (verify the live slug).

### `HW-17-0021` - GET_NETWORK_INFO is declared but neither documented nor used by any code

- Category D, severity low, confidence confirmed
- Features: FOOD-02
- Document: `huawei_industry_tree/17_food/docs/02_practice-food-menu-app-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-menu-app-demo-v1-0000002300530042
- Why: Least-privilege violation: a permission with no consumer in code and no mention in the doc's own permission section.
- Fix: Remove the GET_NETWORK_INFO entry from module.json5.

### `HW-17-0023` - Reusable StaticMap component hardcodes destinationPoiId, so every store's map click opens the same POI

- Category B, severity low, confidence confirmed
- Features: FOOD-04
- Document: `huawei_industry_tree/17_food/docs/04_map_navigation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_navigation-0000002248154336
- Why: The component is parameterized by coordinates but pins the POI detail to one fixed ID; used for any other store it silently shows the wrong place.
- Fix: Accept poiId as a @Prop and pass it from StoreDetailPage.

### `HW-17-0024` - Map Kit calls have no error handling although the API documents a 'Map permission is not enabled' error

- Category B, severity low, confidence confirmed
- Features: FOOD-04
- Document: `huawei_industry_tree/17_food/docs/04_map_navigation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_navigation-0000002248154336
- Why: The most likely first-run failure (map service not enabled, the exact setup step the doc warns about) is swallowed: the image stays empty with no feedback and the promise rejection is unhandled.
- Fix: Add .catch/try-catch around getMapImage and the petalMaps calls, logging the code and showing a toast.

### `HW-17-0025` - UIContext instance is passed into a child component as @Prop, which deep-copies the value

- Category C, severity low, confidence probable
- Features: FOOD-04
- Document: `huawei_industry_tree/17_food/docs/04_map_navigation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_navigation-0000002248154336
- Why: @Prop is designed for data values; deep-copying a framework handle like UIContext is at best wasted work and at worst yields a clone whose getHostContext() no longer matches the live window context.
- Fix: Drop the decorator (private context: UIContext) or call this.getUIContext() inside StaticMap.

### `HW-17-0026` - Header title opacity accumulates unclamped scroll deltas and drifts far outside [0,1]

- Category B, severity low, confidence confirmed
- Features: FOOD-05
- Document: `huawei_industry_tree/17_food/docs/05_custom_refresh.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_refresh-0000002331948181
- Why: Opacity is a [0,1] value; because the accumulated state can reach large negative numbers, scrolling back up must first 'pay back' the overshoot before the title becomes visible again, so the title stays invisible long after returning to the top.
- Fix: this.titleOpacity = Math.min(1, Math.max(0, this.titleOpacity - yOffset / Constants.TWENTY)); or compute from Scroll.currentOffset().

### `HW-17-0027` - Pull-to-refresh reload() only re-notifies the unchanged data set, so refreshing visibly does nothing

- Category B, severity low, confidence probable
- Features: FOOD-05
- Document: `huawei_industry_tree/17_food/docs/05_custom_refresh.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_refresh-0000002331948181
- Why: A refresh implementation that neither resets nor refetches data demonstrates the gesture but not the feature; developers copying the template get a no-op refresh.
- Fix: In reload(), rebuild dataArray with the first ONCE_COUNT items and then call notifyDataReload().

### `HW-17-0028` - 1 sample project depends on third-party packages through unpinned version ranges

- Category B, severity low, confidence confirmed
- Features: FOOD-02
- Document: `huawei_industry_tree/17_food/docs/02_practice-food-menu-app-demo-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-menu-app-demo-v1-0000002300530042
- Why: A caret or tilde range lets ohpm resolve a different version than the one the sample was verified against, so the published sample is not reproducible and can break without any change to its own source. Sample projects are reference implementations; their dependency set should be exactly the one that was tested.
- Fix: Pin exact versions in oh-package.json5 dependencies, and record the resolved set in oh-package-lock.json5.

