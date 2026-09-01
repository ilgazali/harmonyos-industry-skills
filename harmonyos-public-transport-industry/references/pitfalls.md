# Pitfalls

> Generated from `features/*.md`. Source industry: `06_public_transport`, 9 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (1)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-06-0051` - Systematic: 6 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: TRANS-03, TRANS-04, TRANS-05, TRANS-06, TRANS-07, TRANS-08
- Document: `huawei_industry_tree/06_public_transport/docs/06_add_shortcut_to_desktop.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_shortcut_to_desktop-0000002368913597
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (51)

### `HW-06-0001` - One failed route lookup disables the search button permanently, because the loading flag is cleared only on the success path

- Category B, severity high, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: getRoutesLoading is a re-entrancy guard: the button raises it and refuses to fire again while it is set. getRoutes clears it as the very last statement of the try, and its catch block is completely empty. Any failure before that line - navi.getDrivingRoutes rejecting because AGC is not configured or the network is down, or result.routes[0] throwing when the service returns no route - jumps to the empty catch, leaves the flag raised, and reports nothing. From that point the search button is inert for the lifetime of the page: every tap sees getRoutesLoading still true and does nothing at all, with no error, no spinner state change and no log. The user is left tapping a dead control.
- Fix: Move the reset into a finally block and log the error: try { ... } catch (err) { Logger.error(`getRoutes failed: ${JSON.stringify(err)}`); } finally { this.getRoutesLoading = false; }

### `HW-06-0002` - Two independent positioning mechanisms are started for one lookup and race to write the same city field

- Category B, severity high, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: getLocationInfo starts a continuous locationChange subscription whose callback reverse-geocodes and writes this.city, this.localArea and AppStorage['firstSelectCity'], and then immediately also fires a one-shot getCurrentLocation whose then branch reverse-geocodes again and writes this.city too. Both paths run, both hit the geocoding service, and whichever resolves last wins - so the displayed city is decided by a race. The two paths are not even equivalent: only the subscription writes localArea and the AppStorage key, so if the one-shot wins the city label updates while the district and the stored city do not. Two positioning requests and two reverse-geocode calls are also billed and powered for a value the screen needs once.
- Fix: Keep the subscription with its unsubscribe-on-first-fix logic and delete the getCurrentLocation call, or keep the one-shot and delete the subscription. Whichever remains must write all three pieces of state.

### `HW-06-0003` - The location subscription is never released when the geocoder is unavailable

- Category B, severity high, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: Every off() call in this callback sits inside the if (isAvailable) branch or in a catch reached from within it. When isGeocoderAvailable() returns false the callback simply falls off the end without unsubscribing, so locationChange stays registered and keeps firing for every position update the system produces, for the lifetime of the process. Each invocation re-checks the geocoder, finds it still unavailable, and does nothing - a permanent wake-up on every location fix that also holds the component instance alive through the closure. Geocoder availability is exactly the condition a developer cannot control: it depends on the device, the region and the network.
- Fix: Unsubscribe unconditionally at the end of the callback rather than per branch, and treat the geocoder-unavailable case as a handled outcome that still stops the subscription.

### `HW-06-0018` - The router parameter the whole practice is about is only logged, and the target page is hardcoded

- Category B, severity high, confidence confirmed
- Features: TRANS-03
- Document: `huawei_industry_tree/06_public_transport/docs/03_qrcard_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/qrcard_demo-0000002328156469
- Why: The document's subject is jumping from a home-screen card to the ride-code page, and its single code sample is the FormLink whose params carry router: 'QrCodePage'. On the receiving side that value is parsed in both onCreate and onNewWant and then passed to hilog and nothing else - it never reaches loadContent, which names 'pages/QrCodePage' as a literal. The demo appears to work only because the application has exactly one page and it happens to be the one the card names. A developer who adds a second card pointing at a different screen gets the ride-code page from both, with no error and a log line that cheerfully reports the target it ignored. onNewWant has the same dead parse, so returning to an already-running app does not route either.
- Fix: Use the value: build the page path from params.router and pass it to loadContent on cold start, and push it onto the app's navigation stack in onNewWant when the ability is already running. Validate it against a known set of page names before using it.

### `HW-06-0026` - Location is fetched regardless of the permission verdict, up to three times on the denial path

- Category B, severity high, confidence confirmed
- Features: TRANS-04
- Document: `huawei_industry_tree/06_public_transport/docs/04_real_time_bus.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/real_time_bus-0000002300272510
- Why: requestPermissionFromUser calls getLocation() on the line before it returns the verdict, so the positioning request is issued while the caller has not yet looked at authResults - it runs even when the user tapped Deny. requestPermission then sees the denial and opens the settings sheet, and requestPermissionOnSetting's then branch calls getLocation() again while binding data, the array of grant statuses, only to a log line. Since that then branch resolves whenever the sheet closes rather than only on a grant, a user who declines twice still triggers a third positioning attempt. Each attempt is a real SingleLocationRequest with a ten-second timeout that can only fail without the permission, so the denial path costs three failed acquisitions and produces nothing the user can see.
- Fix: Return the verdict without side effects, and let the caller act: in requestPermission, call getLocation() once after confirming authResults[0] === 0; in requestPermissionOnSetting's then branch, check data before calling it.

### `HW-06-0031` - An unused outer loop makes every ride record appear five times in the results

- Category B, severity high, confidence confirmed
- Features: TRANS-05
- Document: `huawei_industry_tree/06_public_transport/docs/05_ride_records.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/ride_records-0000002312400232
- Why: The loop variable i is never read inside the body, so the outer loop does nothing except run the inner filter five times and push a duplicate DataInfo for every match on each pass. INPUT_TIMES is 5 and data.json holds 155 records over 31 days, so a filter spanning the full data set returns 775 rows in which every trip is listed five consecutive times. This is the single function the document exists to explain, and 实现思路 step 1 reproduces it verbatim, so a reader copying the snippet inherits the duplication. It also multiplies the rendering cost: the list in step 2 uses ForEach with no key generator, which falls back to serialising each item, so the five-fold inflation is paid again on every render pass.
- Fix: Delete the outer loop and keep the inner one. If INPUT_TIMES was meant to bound how many records are returned, apply it as a limit rather than a repeat count.

### `HW-06-0032` - The bundled ride data falls entirely outside the default filter window, so the sample opens with an empty list

- Category B, severity high, confidence confirmed
- Features: TRANS-05
- Document: `huawei_industry_tree/06_public_transport/docs/05_ride_records.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/ride_records-0000002312400232
- Why: The view model seeds the filter with today minus seven days to today, and the constructor runs getData with those bounds, so the first thing the user sees is whatever matches the last week. Every record in data.json is dated between 31 May and 30 June 2025. The two windows have no overlap and never will again - the data is fixed while the default range follows the clock, so the gap widens every day. Anyone who builds and runs this sample sees an empty ride-history screen and has to work out for themselves that the date picker must be driven back to mid-2025 before the feature does anything. Nothing in the document mentions it.
- Fix: Generate the mock timestamps relative to the current date at startup, or seed beginTime and endTime from the earliest and latest dates present in the data instead of from the clock.

### `HW-06-0035` - Tapping the metro-map shortcut opens the home page, not the map: the handler clears the flag and navigates nowhere

- Category B, severity high, confidence confirmed
- Features: TRANS-06
- Document: `huawei_industry_tree/06_public_transport/docs/06_add_shortcut_to_desktop.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_shortcut_to_desktop-0000002368913597
- Why: The document promises 点击即可快速查看 - tap the desktop shortcut and the metro map opens. Every link in the chain is built correctly: shortcuts_config.json carries shortCutKey, the want delivers it, EntryAbility catches it in both onCreate and onNewWant and publishes it to AppStorage, MainPage links it with @StorageLink and watches it, and aboutToAppear calls the handler directly to cover the cold-start case where the watch has not yet fired. Then the handler, named jumpToMap, does nothing but assign an empty string. The only code in the project that reaches MapPage is a double-tap gesture on the image, which the shortcut path never touches. A user who adds the shortcut and taps it lands on the home page and has to discover the double-tap for themselves. The handler also writes the very variable it is registered to watch, so the assignment re-enters it once before terminating on the emptied value.
- Fix: Navigate, then clear: jumpToMap(): void { if (this.shortCutKey !== 'MainPage') { return; } this.shortCutKey = ''; this.pageInfos.pushPathByName('MapPage', null); } - clearing first avoids the re-entrant watch, and the push is what the document describes.

### `HW-06-0039` - Both reminders ship a test placeholder in additionalText, a field the notification displays to the user

- Category B, severity high, confidence confirmed
- Features: TRANS-07
- Document: `huawei_industry_tree/06_public_transport/docs/07_bus_on_off_notification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bus_on_off_notification-0000002394404617
- Why: additionalText is part of NotificationBasicContent and is rendered in the notification alongside the title and text; it is not a debug field. Both reminders in this sample fill it with a developer placeholder, so a passenger who enables the feature receives a banner reading 请上车 / 车辆即将到站 / test_additionalText, and the alighting reminder reads test_additionalText111. This is the payload of the only feature the document describes, and it reaches the lock screen and the notification shade. The document's step 3 snippet omits additionalText entirely, so a reader comparing the document with the archive would not notice the placeholder is there.
- Fix: Populate additionalText with the stop or line the reminder refers to, or drop the field from both requests.

### `HW-06-0044` - Centring the map on the user is gated behind a reverse-geocode whose result is never used

- Category B, severity high, confidence confirmed
- Features: TRANS-08
- Document: `huawei_industry_tree/06_public_transport/docs/08_map_rotation_lock.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_rotation_lock-0000002396727869
- Why: The coordinates the camera needs are already in hand as result.latitude and result.longitude the moment getCurrentLocation resolves. The code nevertheless issues a reverse-geocode, and puts the entire camera move inside the success branch of that callback - while doing nothing with the address it fetched beyond logging it. Two ordinary conditions therefore break the feature: if isGeocoderAvailable() returns false the callback is never entered and the map silently stays where it was, and if the geocode fails, err is logged and the move is skipped even though nothing about it depended on the address. Reverse geocoding is device- and region-dependent and needs the network; map centring needs neither. The document's step 2 describes this as 通过getLocation方法获取当前位置经纬度，通过moveCamera方法将地图相机移动至当前位置 with no mention of a geocode in between.
- Fix: Move the camera directly from result in the then branch, and drop the getAddressesFromLocation call unless the address is actually displayed - in which case run it separately and let it fail without blocking the move.

### `HW-06-0052` - PayPage subscribes to the cityChange event inside onAppear and never unsubscribes, so handlers accumulate on every visit

- Category B, severity high, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: onAppear fires every time the component becomes visible, not once per instance, so navigating away from the pay page and back registers an additional handler each time. After n visits a single cityChange event runs the handler n times and pushes n duplicate routes onto pageStack1. The handler closure also captures this, so the component and its page stack are retained after the page is destroyed. This is the entry architecture sample for the industry, so the pattern gets copied.
- Fix: Move emitter.on into aboutToAppear and call emitter.off('cityChange') in aboutToDisappear, keeping a reference to the handler so the correct one is removed.

### `HW-06-0004` - getLocationInfo is called once per granted permission, so a normal grant starts two full positioning cycles

- Category B, severity medium, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: The call sits inside the loop body rather than after it, so it runs once for each granted entry in authResults. The permissions array holds two entries and the system grants both together in the ordinary case, so accepting the dialog invokes getLocationInfo twice. Since getLocationInfo itself already starts both a locationChange subscription and a getCurrentLocation call, one tap on Allow produces two subscriptions and two one-shot requests - four positioning operations and up to four reverse-geocode calls for a single city label. The second subscription is also unreachable by the first callback's off(), because off is passed the specific closure instance.
- Fix: Accumulate the verdict in the loop and act once: for (...) { if (grantStatus[i] !== 0) { return; } } this.getLocationInfo();

### `HW-06-0005` - checkPermissions inspects only the first entry of the permissions array

- Category B, severity medium, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: The array declares two permissions and the manifest requests two, but only index 0 is ever examined. Whenever the first is granted and the second is not, the branch proceeds straight to getLocationInfo without asking for the missing one. This is the third variant of the same defect encountered in this review: the automotive architecture practice and the petrol-station practice both ship a checkPermissions that loops but overwrites its verdict each iteration, and this one does not loop at all. The reverse-geocoding this screen depends on needs the precise permission, so a partial grant produces a page that silently shows no city.
- Fix: Check all entries and request whenever any is missing: for (const p of this.permissions) { if (await this.checkAccessToken(p) !== PERMISSION_GRANTED) { this.reqPermissionsFromUser(...); return; } } this.getLocationInfo();

### `HW-06-0006` - Place search is confined to a 50-metre radius where the documented default is 50000 metres

- Category A, severity medium, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: The reference defines radius as 'Location search radius, in meters. The value ranges from 1 to 50000. The default value is 50000.' The sample sets 50 on both the origin and the destination search, narrowing the search area by a factor of a thousand from the default with no comment explaining why. This is the origin and destination lookup of a trip planner: the user types the name of a station or a street and expects it to be found anywhere in the city. A 50-metre circle will return nothing for almost every query, and because the result is not length-checked the code then falls back to coordinate zero. Nothing in the document mentions the radius at all.
- Fix: Drop the radius parameter to take the documented 50000 default, or set it to a city-scale value and state the intent in a comment.

### `HW-06-0007` - An empty search result places the trip marker at coordinate zero instead of reporting that nothing was found

- Category B, severity medium, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: The guard tests that sites exists, not that it contains anything, so an empty array passes. The optional chaining then yields undefined and the || 0 fallbacks turn it into latitude 0, longitude 0 - a point in the Gulf of Guinea, roughly ten thousand kilometres from any Chinese transit network. A marker is placed there and, because getRoutes reads the marker positions, the subsequent route request is issued from or to that point. Given the 50-metre search radius recorded separately, an empty result is the common case rather than an edge case, so this fallback is on the main path. The user sees a marker vanish off the map with no explanation.
- Fix: Check rsp.sites.length before building the position, and surface a not-found message instead of substituting zero.

### `HW-06-0008` - Reverse-geocoding results are indexed without a length check and the one-shot path has no rejection handler

- Category B, severity medium, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: getAddressesFromLocation resolves with an array that can be empty - the coordinates may have no civic address, or the service may return nothing. Both call sites index element zero immediately, so an empty result is a TypeError rather than a handled outcome. On the callback path the throw happens inside the framework's callback with no surrounding handler; on the promise path there is no catch at all, so both the empty-array TypeError and any rejection from getCurrentLocation itself - location services off, permission revoked between the check and the call, no fix before timeout - become unhandled rejections. The surrounding try/catch does not help: it wraps the synchronous call, not the asynchronous resolution.
- Fix: Guard both sites with a length check and add a catch to the promise chain: if (!data || data.length === 0) { return; } and .catch((err) => Logger.error(...)).

### `HW-06-0009` - All eight HAR modules redeclare the app's permissions and each names an EntryAbility that none of them contains

- Category E, severity medium, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: Nine modules ship the same three permission declarations: the entry HAP product/phone, which legitimately owns them, and eight HAR modules that do not. A HAR declares no abilities at all - EntryAbility is defined only in product/phone - so every usedScene.abilities entry in those eight manifests points at an ability that is not in the module making the declaration. The result is eight copies of a permission request to keep in step by hand, each with a dangling scope reference, and no single place that states what the application actually needs. The maternity and automotive practices reviewed earlier both declare permissions only in their entry HAP and leave every feature HAR's manifest without a requestPermissions block, so this sample also breaks the convention the rest of the corpus follows.
- Fix: Remove requestPermissions from all eight HAR manifests and keep the single declaration in product/phone/src/main/module.json5.

### `HW-06-0010` - The shipped manifest hardcodes the sample author's AppGallery Connect client_id and app_id, and the document never says to replace them

- Category E, severity medium, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: Map Kit resolves the app against the Client ID declared in this metadata block. The archive ships concrete values belonging to whoever built the sample, so a developer who follows the document, completes the AGC setup it links to, and then builds the project still has someone else's identifiers in the manifest - the map fails to authorise and nothing in the document points at the metadata block as the thing to change. 方案设计 says only 开发者先登录AGC，完成地图相关的配置 and links map-config-agc, which covers the console side and not the manifest side. The inline comment in the manifest is the only hint, and it is in the file the reader has no reason to open. Publishing concrete project identifiers in a downloadable sample is also poor practice in its own right.
- Fix: Replace both values with obvious placeholders such as YOUR_CLIENT_ID, and add a step to 路径规划技术方案 naming the metadata block in product/phone/src/main/module.json5 as the place to paste the Client ID from AGC.

### `HW-06-0011` - Three-way disagreement about which devices the app supports

- Category E, severity medium, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: The document restricts the practice to phones twice, in the introduction and again in the software view. The entry HAP adds tablet, so the app installs and runs on tablets whose layout the document never discusses. The eight HAR manifests declare a third, different set - default plus tablet - where default is the catch-all qualifier rather than phone. A reader therefore has three answers to a single question, and the module that decides distribution disagrees with the prose that describes it. The sample does ship a BreakpointSystem utility in common, which suggests some multi-device intent, but no document text covers it.
- Fix: Decide the scope, state it once in 简介, and make all nine manifests match it.

### `HW-06-0012` - The calendarManager link leaves the HarmonyOS documentation for the OpenHarmony community repository on gitcode.com

- Category E, severity medium, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: Every other API link in this document points at developer.huawei.com. This one points at a markdown file in the OpenHarmony documentation repository on gitcode.com - a different product, maintained separately, whose API surface and version numbering do not match HarmonyOS. The reader is dropped out of the documentation set the rest of the practice is written against, into a raw file view on a code-hosting site. The HarmonyOS reference does document this API: the page js-apis-calendarmanager exists at developer.huawei.com, and there are three calendarmanager guides in the HarmonyOS guides as well, so there is no gap the external link is filling. The two links either side of it, for Live View Kit and Form Kit, both point at developer.huawei.com.
- Fix: Point the link at https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-calendarmanager.

### `HW-06-0013` - Five empty catch blocks swallow failures across route planning, place search and preferences

- Category B, severity medium, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: Five catch blocks in this sample contain nothing at all. Three of them sit on the trip-planning path - the route request and both place searches - so a failed lookup produces no message, no log and no visible change; the route one additionally strands the loading flag, recorded separately. The two in PreferencesUtil mean that a failure to open the preferences file or to write a value is invisible, and since loadPreferences swallows its own failure the map of instances silently stays empty and every later read and write no-ops. The project ships a Logger in common/src/main/ets/utils and uses it elsewhere, including in the permission code in Home.ets, so the tooling to do this properly is already imported and in use a few files away.
- Fix: Log in all five with the existing Logger, and let the two trip-planning ones set an error state the UI can render.

### `HW-06-0017` - The public transport scenario index is published under the finance and insurance URL family

- Category E, severity medium, confidence confirmed
- Features: TRANS-02
- Document: `huawei_industry_tree/06_public_transport/docs/02_financial-insurance-v1-6_1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/financial-insurance-v1-6_1-0000002293996400
- Why: Every industry publishes a 关键场景示例 page listing its feature practices. Seventeen of the nineteen name their own industry in the slug. This one is served from financial-insurance-v1-6_1, so the canonical URL of the public transport scenario index declares it a finance and insurance page; only the trailing 6 distinguishes it from the finance industry's own index at financial-insurance-v1-7_1. The page content is entirely correct - the title is 关键场景示例 and all six links resolve to the transit practices - so the defect is purely in identity: the address, the breadcrumb it implies and anything keyed on the slug all point at the wrong industry. The sibling architecture document for this same industry is practice-bus-app-architecture-v1, which is the family this page should have joined. It reads as a copy of the finance index with only the number changed.
- Fix: Republish the page under a transit slug consistent with its own architecture document, and redirect the current address.

### `HW-06-0019` - A resource key is concatenated into the message parameter as a literal string

- Category B, severity medium, confidence confirmed
- Features: TRANS-03
- Document: `huawei_industry_tree/06_public_transport/docs/03_qrcard_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/qrcard_demo-0000002328156469
- Why: 'app.string.widget_mine' is the name of a string resource, not a string value. Concatenating it onto this.message produces the literal 'add detailapp.string.widget_mine', which is what the card actually sends to the ability. The intent was evidently to append the localised label, and the correct way to reference that same resource appears two lines further down in the same build method as $r('app.string.widget_mine'). So the file demonstrates the right and the wrong way to reach one resource within a few lines of each other, and the document reproduces the wrong one as its only code sample. Nothing catches it because the receiving side merely logs the params.
- Fix: Drop the concatenation, or resolve the resource before sending: getContext().resourceManager.getStringSync($r('app.string.widget_mine').id).

### `HW-06-0020` - JSON.parse of the incoming want parameters has no error handling, in the ability's startup path

- Category B, severity medium, confidence confirmed
- Features: TRANS-03
- Document: `huawei_industry_tree/06_public_transport/docs/03_qrcard_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/qrcard_demo-0000002328156469
- Why: want.parameters.params arrives from outside the application - the launcher passes through whatever the card put there, and the value is cast to string without checking its type. JSON.parse throws on anything that is not valid JSON, and the guard above it only tests that the field is present. A throw here happens inside onCreate, the first lifecycle callback of the ability, so a malformed or unexpected payload takes down the launch rather than degrading to a normal start. The same unguarded parse is repeated in onNewWant. The file handles errors carefully everywhere else - setWindowLayoutFullScreen has a catch, loadContent checks err.code - which makes the omission on the one path fed by external input stand out.
- Fix: Wrap both parses: try { const params = JSON.parse(String(want.parameters.params)); ... } catch (e) { hilog.error(DOMAIN, 'testTag', '%{public}s', `bad card params: ${e}`); }

### `HW-06-0021` - The avoidAreaChange listener is registered on the main window and never released

- Category B, severity medium, confidence confirmed
- Features: TRANS-03
- Document: `huawei_industry_tree/06_public_transport/docs/03_qrcard_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/qrcard_demo-0000002328156469
- Why: The listener is attached to the main window in onWindowStageCreate and closes over the ability. onWindowStageDestroy exists, carries a comment saying it is where UI resources are released, and then releases nothing. A UIAbility launched from a home-screen card is destroyed and recreated more often than a normally-launched one, because each card tap may reach a fresh instance, so every cycle attaches another listener while the previous one remains registered and every avoid-area change fans out to a growing set of handlers writing the same two AppStorage keys. The identical omission ships in the automotive industry's bottom-sheet practice, so this is a pattern being copied rather than a one-off.
- Fix: Keep the window reference on the ability and release it: onWindowStageDestroy(): void { this.windowClass?.off('avoidAreaChange'); }

### `HW-06-0022` - The document calls the widget static in one paragraph and dynamic in the next, and the configuration says static

- Category E, severity medium, confidence confirmed
- Features: TRANS-03
- Document: `huawei_industry_tree/06_public_transport/docs/03_qrcard_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/qrcard_demo-0000002328156469
- Why: The two sentences are eight lines apart and say opposite things about what kind of widget this is. The shipped form_config.json settles it: isDynamic is false, so the card is static. The wrong sentence is the one in 实现思路, which is the paragraph that introduces the only code sample and which a reader takes as the statement of what is being built. Static and dynamic widgets differ in what they may do - update policy, interactivity, whether an ArkTS card runtime is involved - so telling the reader it is dynamic sends them to the wrong half of the widget documentation.
- Fix: Correct 实现思路 to say 静态卡片, matching 场景介绍 and the isDynamic setting.

### `HW-06-0027` - The permission status array is dereferenced on the very path where the guard admits it may be absent

- Category B, severity medium, confidence confirmed
- Features: TRANS-04
- Document: `huawei_industry_tree/06_public_transport/docs/04_real_time_bus.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/real_time_bus-0000002300272510
- Why: requestPermissionFromUser returns requestStatus?.authResults, and the optional chaining is there precisely because requestStatus may be absent - so the declared Promise<number[]> can resolve to undefined. The guard in requestPermission acknowledges that by testing permissionStatus and its length before reading index zero. The very next line, reached only when that guard failed, indexes permissionStatus[0] unconditionally. When the array is missing or empty - the two cases the guard was written for - this throws a TypeError inside an async method whose rejection nobody handles, so the settings sheet on the following line never opens and the user is left with no permission and no prompt.
- Fix: Guard the log or drop it: Logger.info(`permission status: ${permissionStatus?.[0] ?? 'none'}`); and widen the declared return type to Promise<number[] | undefined> so the possibility is visible at the call site.

### `HW-06-0028` - The reverse-geocoding result is indexed without checking that it returned anything

- Category B, severity medium, confidence confirmed
- Features: TRANS-04
- Document: `huawei_industry_tree/06_public_transport/docs/04_real_time_bus.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/real_time_bus-0000002300272510
- Why: getAddressesFromLocation resolves with an array of addresses that can legitimately be empty - coordinates with no civic address, or a geocoding service that returns no match. The else branch treats a non-error result as guaranteed to contain at least one entry and reads data[0] directly. The ?? '' on the end guards the field, not the index, so an empty array is a TypeError before the coalescing operator is ever reached, thrown inside a framework callback with nothing to catch it. The document reproduces this exact shape as step 3, so it is the version a reader copies. The same unchecked indexing appears in this industry's architecture practice, which suggests it travels with the snippet rather than being written fresh.
- Fix: Check first: if (!data || data.length === 0) { return; } before reading data[0].

### `HW-06-0029` - A sub-administrative area is stored and displayed as the province

- Category B, severity medium, confidence probable
- Features: TRANS-04
- Document: `huawei_industry_tree/06_public_transport/docs/04_real_time_bus.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/real_time_bus-0000002300272510
- Why: GeoAddress exposes both administrativeArea and subAdministrativeArea, and the names state their relationship: the sub-administrative area is a division below the administrative one. In the Chinese address hierarchy that puts administrativeArea at province level and subAdministrativeArea at prefecture or district level. The code reads the sub level and assigns it to a field named province, which is then passed to SearchBar and rendered as the location label in the header, so the user is shown a district under a province heading. Note also that this industry's own architecture practice reads locality for the same purpose, so two samples in one industry pick two different GeoAddress fields to answer 'where am I'.

RECORDED AS PROBABLE: the exact semantics of administrativeArea versus subAdministrativeArea could not be confirmed against the local corpus - documentation/harmonyos-references/06_application-services/js-apis-geolocationmanager.md is an eleven-line stub with no GeoAddress field table, so the reasoning rests on the field names and the address hierarchy rather than on a quoted definition. Confirm against the live reference for GeoAddress before reporting.
- Fix: If a province is wanted, read administrativeArea. If the district is wanted, rename the state, the prop and the label accordingly. Either way, align this with the field the architecture practice uses so the industry is consistent.

### `HW-06-0030` - The permission declares usedScene against a FormAbility that the module does not contain

- Category E, severity medium, confidence confirmed
- Features: TRANS-04
- Document: `huawei_industry_tree/06_public_transport/docs/04_real_time_bus.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/real_time_bus-0000002300272510
- Why: The module declares one ability, EntryAbility, and one backup extension ability. FormAbility appears nowhere in the manifest, nowhere in the source and nowhere in the project - this sample ships no widget at all. usedScene is the field that tells the system which abilities need the permission and underpins how the request is scoped and presented, so naming a non-existent ability leaves the declaration describing something that cannot occur. The location is in fact requested from MainPage, which runs under EntryAbility. The document devotes a 权限说明 section to this single permission and does not mention the scope. The automotive industry's petrol-station practice carries the identical defect with a different phantom ability name, so this is a copy-paste pattern across samples rather than a one-off slip.
- Fix: Replace FormAbility with EntryAbility in the usedScene block.

### `HW-06-0033` - Reset collapses the date range to a single day instead of restoring the default, and it is also the recovery path for an invalid range

- Category B, severity medium, confidence confirmed
- Features: TRANS-05
- Document: `huawei_industry_tree/06_public_transport/docs/05_ride_records.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/ride_records-0000002312400232
- Why: The constructor opens the screen on a seven-day window, but reSet sets both ends to today, so pressing reset does not return the user to the state they started in - it narrows the range to a single day. The same method is the recovery path in dateComplianceCheck: when the user picks a begin date later than the end date, the toast fires and reSet collapses both fields to today, silently discarding the end date they had chosen as well as the one they got wrong. A reset control that lands somewhere other than the initial state, and an error handler that throws away valid input alongside invalid input, are both surprising in a filter the user is expected to drive back and forth.
- Fix: Factor the constructor's default into a method and call it from both places; in dateComplianceCheck, revert only the field that was just set rather than calling reSet.

### `HW-06-0036` - The bundle name is hardcoded in code, duplicating the shortcut configuration with nothing keeping the two in step

- Category B, severity medium, confidence confirmed
- Features: TRANS-06
- Document: `huawei_industry_tree/06_public_transport/docs/06_add_shortcut_to_desktop.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_shortcut_to_desktop-0000002368913597
- Why: checkPinShortcutPermitted requires the want passed in code to match the one declared in shortcuts_config.json, so the same four values exist in two files with no mechanism tying them together. The bundle name is the one a developer is certain to change - it is the first thing edited when the sample is adapted - and changing it in AppScope/app.json5 and the profile while missing the literal in MainPage.ets produces a mismatch that surfaces only as a rejected shortcut request at runtime. The application already knows its own identity: this.context.abilityInfo.bundleName gives it without duplication. The document reproduces the literal in step 1 and its comment says only 对应shortcuts标签中配置的want, without warning that the value must be kept in step with the profile and with the app's real bundle name.
- Fix: Read the bundle name from the context - let ctx = this.getUIContext().getHostContext() as common.UIAbilityContext; bundleName: ctx.abilityInfo.bundleName - and note in the document that the want must match shortcuts_config.json exactly.

### `HW-06-0037` - Only one of the failure codes reaches the user; every other outcome is logged and nothing else

- Category B, severity medium, confidence confirmed
- Features: TRANS-06
- Document: `huawei_industry_tree/06_public_transport/docs/06_add_shortcut_to_desktop.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_shortcut_to_desktop-0000002368913597
- Why: The error handling here is otherwise the best in this industry - both promises have a catch, the whole block sits in a try, and one specific code is mapped to a friendly toast. But 1006620003, shortcut already added, is the only outcome the user is ever told about. A refusal by the launcher, an unavailable service, a want that does not match the profile, or any failure inside requestNewPinShortcut all end at a Logger.error the user cannot see, so tapping Add to desktop appears to do nothing at all and the user has no way to distinguish it from a slow success. Adding a shortcut is a deliberate, explicit user action; silent failure is the one outcome it should not have.
- Fix: Add a generic failure toast alongside the specific one in both catch blocks, keeping the specific message for 1006620003.

### `HW-06-0040` - The boarding and alighting reminders share one notification id, so the second silently replaces the first

- Category B, severity medium, confidence confirmed
- Features: TRANS-07
- Document: `huawei_industry_tree/06_public_transport/docs/07_bus_on_off_notification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bus_on_off_notification-0000002394404617
- Why: NotificationRequest.id identifies the notification within the application: publishing a second request with the same id updates the existing notification rather than adding one. Both reminders here use 1, so the alighting reminder overwrites the boarding reminder wherever the boarding one is still on screen - which is the normal case on a short hop, where the two stops are close together. It also removes any way to cancel one without the other, and it means a second journey started while the first reminder is showing collides with it. Two reminders that mean different things and can coexist need two identities; a constant literal in two places is the least maintainable way to express that.
- Fix: Give each a distinct id from a named constant, for example ON_BUS_NOTIFICATION_ID = 1 and OFF_BUS_NOTIFICATION_ID = 2, and derive per-journey ids if more than one trip can be tracked.

### `HW-06-0041` - The guard for an unset alighting stop tests undefined while the unset value is null, so it never guards anything

- Category B, severity medium, confidence confirmed
- Features: TRANS-07
- Document: `huawei_industry_tree/06_public_transport/docs/07_bus_on_off_notification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bus_on_off_notification-0000002394404617
- Why: The owning page declares offPoint as number | null and initialises it to null, and BottomContent resets it to null when the user clears the choice, so null is the project's own sentinel for 'no alighting stop'. The guard tests against undefined instead, and null !== undefined is true, so the condition passes on every tick whether or not a stop has been chosen. Its comment, 如果设置了下车点, describes a check the code does not perform. The two comparisons inside happen to be harmless today because comparing a number with null yields false, so the defect is masked - but the block also assigns this.ifOnBus, and any future condition added inside it would run unguarded. The document reproduces the same guard in step 1.
- Fix: Test the sentinel the state uses: if (this.offPoint !== null). If both null and undefined must be tolerated, use if (this.offPoint != null), which covers each.

### `HW-06-0042` - The document empties the publish callback and drops the field that carries the placeholder

- Category E, severity medium, confidence confirmed
- Features: TRANS-07
- Document: `huawei_industry_tree/06_public_transport/docs/07_bus_on_off_notification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bus_on_off_notification-0000002394404617
- Why: Two omissions in one short snippet. The callback is the only channel notificationManager.publish reports failure on, and the document prints it with an empty body while the sample checks err and logs both branches - so the version a reader copies silently swallows a failed reminder, which for this feature means the passenger is simply not warned. Separately, the document's content block drops additionalText, the field the sample fills with a test placeholder, so the document conceals the defect recorded against this same practice. This is the sixth document in this review whose code block has been stripped of error handling present in the archive.
- Fix: Regenerate the snippet from entry/src/main/ets/utils/BusNotification.ets, keeping the err check and the complete normal block.

### `HW-06-0045` - Zoom stepping has no upper bound and its lower bound admits values below the documented minimum

- Category A, severity medium, confidence confirmed
- Features: TRANS-08
- Document: `huawei_industry_tree/06_public_transport/docs/08_map_rotation_lock.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_rotation_lock-0000002396727869
- Why: CameraPosition.zoom is documented as 'Zoom level near the center of the screen. The value ranges from 2 to 20.' The increase branch has no guard at all, so repeated taps carry zoomSize past 20 without limit and every subsequent moveCamera is issued with an out-of-range value. The decrease branch is guarded, but at the wrong boundary: this.zoomSize >= 1 permits a decrement from 1 to 0, so both 1 and 0 - two values below the documented minimum - are reachable. Unlike bearing, whose entry in the same table states that out-of-range values are adjusted back into range, the zoom entry promises no such correction, so the behaviour past the bounds is unspecified. The default is 16, so a user only has to tap zoom-in five times to leave the documented range.
- Fix: Clamp both directions against named constants: if (enlarge && this.zoomSize < MAP_ZOOM_MAX) { this.zoomSize++; } else if (!enlarge && this.zoomSize > MAP_ZOOM_MIN) { this.zoomSize--; } with MAP_ZOOM_MIN = 2 and MAP_ZOOM_MAX = 20.

### `HW-06-0046` - The permission request is not awaited before the location lookup it exists to authorise

- Category B, severity medium, confidence confirmed
- Features: TRANS-08
- Document: `huawei_industry_tree/06_public_transport/docs/08_map_rotation_lock.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_rotation_lock-0000002396727869
- Why: requestPermissions is declared async and awaits two things internally - the token check and, when needed, the user dialog. The click handler calls it without await and immediately calls mapVM.init(), which calls getLocation, which calls geoLocationManager.getCurrentLocation. So on a first run the positioning request is issued while the permission dialog is still on screen and the permission has not been granted, and it fails; the user grants the permission a moment later and nothing retries, so the map stays where it was until the button is pressed a second time. The same handler shape is used in aboutToAppear a hundred lines above, where the call is correctly awaited - so the file contains both forms.
- Fix: Make the handler async and await the request, then call init only when the permission was granted.

### `HW-06-0047` - Step 2 calls the permission helper with no argument while step 1 on the same page declares it takes a context

- Category E, severity medium, confidence confirmed
- Features: TRANS-08
- Document: `huawei_industry_tree/06_public_transport/docs/08_map_rotation_lock.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_rotation_lock-0000002396727869
- Why: The two snippets are on the same page, seventeen lines apart, and contradict each other: the first declares requestPermissions(context: UIContext), the second calls requestPermissions() with nothing. The parameter is not optional in the sample and the body dereferences it, so the step 2 snippet does not compile and, if it did, would fail at the first use of context. The archive passes this.context correctly. A reader working through 实现思路 in order takes step 2 as the call site for the function step 1 just defined, so the omission lands exactly where it is most likely to be copied. This is the seventh document in this review whose code block diverges from the archive it quotes.
- Fix: Regenerate the step 2 snippet from entry/src/main/ets/pages/MainPage.ets so the call carries this.context, and await it as the fix for the related runtime defect requires.

### `HW-06-0014` - The destination search is locked to one city while the origin search is not, and the two guards are written differently

- Category F, severity low, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: The two handlers do the same job for the two ends of one journey, but only the destination passes cityId, hardcoded to 025 - the city code for Nanjing, which the document names as the sample city for the ride code. The origin has no city filter. So the user can set an origin anywhere but can only ever reach a destination in Nanjing, which is not a restriction any trip planner would want and which the document never mentions. The two blocks also express the same null guard in two different styles, one truthiness-based and one explicit undefined comparison, three lines apart in the same file. The city code is a bare literal with no constant and no comment.
- Fix: Use the same parameters for both endpoints. If a city filter is wanted, derive it from the city the home screen already resolves into AppStorage rather than hardcoding 025, and apply it to both searches.

### `HW-06-0015` - The location permission request is labelled as a microphone permission request

- Category B, severity low, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: 申请麦克风权限 means request microphone permission. The call beneath it requests this.permissions, which holds ohos.permission.LOCATION and ohos.permission.APPROXIMATELY_LOCATION, and the sample neither declares nor uses any audio permission anywhere. The comment is a leftover from a microphone sample that the permission block was copied from. It sits in the middle of the permission-handling code that 获取城市位置代码实现 points readers at, which is precisely the code a developer is reading closely, and it tells them the wrong thing about what the branch does.
- Fix: Replace the comment with one describing the location request, for example 未授权时动态申请定位权限.

### `HW-06-0016` - Route planning ships four unrelated hardcoded locations, including a default destination in the Irish Sea

- Category B, severity low, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: Four different places are baked into one screen of a Nanjing transit app. routeParams.destination defaults to 54.216608, -4.66529, which is the Isle of Man in the Irish Sea. drivingRouteParams defaults both its origin and its destination to 31.98, 120.28, a point near Nantong rather than Nanjing. The map centre and the two route fallbacks use 39.9, 116.4, which is Beijing. Meanwhile the destination place search is pinned to city code 025, Nanjing. None of these are commented and none relate to each other, so the file reads as an assembly of fragments from different samples. The fallbacks are reachable: getRoutes substitutes the Beijing coordinate whenever a marker has no position, so a user whose place search returned nothing gets a route computed to Beijing. language is likewise pinned to zh_CN rather than following the device locale.
- Fix: Initialise the params with empty origins and no destination, refuse to plan until both markers are set instead of substituting a coordinate, centre the map on the city the home screen resolves into AppStorage, and take language from the system locale.

### `HW-06-0023` - The widget configures an update schedule while updates are switched off

- Category B, severity low, confidence confirmed
- Features: TRANS-03
- Document: `huawei_industry_tree/06_public_transport/docs/03_qrcard_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/qrcard_demo-0000002328156469
- Why: updateEnabled false switches off periodic refresh for the widget, which makes scheduledUpdateTime and updateDuration inert - the card will never refresh at 10:30 and never on a one-unit interval. Leaving both in place states an intent the configuration then cancels, so a reader adapting this file cannot tell whether the refresh was meant to be on and was disabled for the demo, or was never wanted. For a ride-code card the distinction matters: a code that must stay current is exactly the case where a developer would look at these fields for guidance.
- Fix: Remove scheduledUpdateTime and updateDuration while updates are disabled, or enable updates if the card is meant to refresh and say so in the document.

### `HW-06-0024` - Parameter data is concatenated into hilog's format argument instead of being passed as a value

- Category B, severity low, confidence confirmed
- Features: TRANS-03
- Document: `huawei_industry_tree/06_public_transport/docs/03_qrcard_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/qrcard_demo-0000002328156469
- Why: hilog treats its third argument as a format string and applies the %{public} and %{private} markers to the arguments that follow. Here the value taken from external want parameters is concatenated into the format string itself, so it is interpreted as format text rather than as data: any percent sequence inside it is read as a directive, and the value carries no public or private marking, which is how hilog decides whether it is redacted in released logs. Every other hilog call in this file uses the format-plus-argument form correctly, including one on the line four lines below. The cast is also misplaced - as string binds to the whole concatenation, which is already a string.
- Fix: Move the value out of the format string and mark it: hilog.info(DOMAIN, 'testTag', 'Target Page %{public}s', String(params.router));

### `HW-06-0025` - The industry's architecture practice requires four API versions more than every feature practice under it

- Category F, severity low, confidence confirmed
- Features: TRANS-03
- Document: `huawei_industry_tree/06_public_transport/docs/03_qrcard_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/qrcard_demo-0000002328156469
- Why: Seven documents in this industry declare two different toolchains. The architecture practice - the one a developer starts from and the one the scenario index points back to - requires API 24 and DevEco Studio 6.1.1, while all six feature practices require API 20 and DevEco Studio 6.0.0. The effective floor for anyone combining them is therefore 24, which no feature document states. The architecture practice also uses a differently named section, 环境准备 rather than 约束与限制, and drops the HarmonyOS SDK line the features carry, so the two baselines are not even presented in comparable form. The automotive industry has the mirror image of this defect, with one feature above its architecture, which suggests the version lines are maintained per document rather than per industry.
- Fix: State the industry baseline once, in the architecture practice, under the same 约束与限制 heading the features use, and note there which practices raise it.

### `HW-06-0034` - The record status is a hardcoded Chinese literal in the model, and the module ships no non-Chinese resources

- Category B, severity low, confidence confirmed
- Features: TRANS-05
- Document: `huawei_industry_tree/06_public_transport/docs/05_ride_records.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/ride_records-0000002312400232
- Why: 已完成 - completed - is the status shown on every row of the ride list, written as a literal inside the data model rather than as a resource. The project does use $r elsewhere, including for the invalid-range toast in FilteringViewModel, so the resource path exists and is in use a file away. The resources directory holds only base, dark and rawfile, with no en_US, so unlike several other samples in this corpus there is not even an empty English shell to fill. Every row of the list is therefore Chinese regardless of device locale. Note also that price and amountPaidOut are fixed defaults never overwritten by getData, so all 155 records display the same fare.
- Fix: Move the status into string.json and reference it with $r, add an en_US resource directory, and carry the real fare fields through from the data instead of defaulting them.

### `HW-06-0038` - The document names the variable uiContext while it holds a UIAbilityContext

- Category E, severity low, confidence confirmed
- Features: TRANS-06
- Document: `huawei_industry_tree/06_public_transport/docs/06_add_shortcut_to_desktop.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_shortcut_to_desktop-0000002368913597
- Why: getUIContext() returns a UIContext and getHostContext() returns the ability's Context, so the cast produces a common.UIAbilityContext - a different type from UIContext, obtained by calling through one to reach the other. Naming it uiContext makes the snippet read as though the API takes a UIContext, which is the type a developer would then try to pass from elsewhere. The whole point of this first step is to show what to hand checkPinShortcutPermitted, and the name is the only clue the snippet gives about the parameter's type. The same misleading name is carried through into the call on the next line and into the nested requestNewPinShortcut call.
- Fix: Rename to abilityContext in both the document snippet and the sample, so the type the API expects is legible from the code.

### `HW-06-0043` - Notification title and body are hardcoded Chinese literals and the module ships no locale resources

- Category B, severity low, confidence confirmed
- Features: TRANS-07
- Document: `huawei_industry_tree/06_public_transport/docs/07_bus_on_off_notification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bus_on_off_notification-0000002394404617
- Why: 请上车 and 车辆即将到站 are the entire user-visible payload of this feature, written as literals inside a utility class. The resources directory holds only base and dark, with no locale qualifier at all, so there is not even a shell to move them into - unlike several other samples in this corpus which ship an en_US directory and simply leave it thin. A notification is the one place where localisation matters most, because it is read outside the app with no surrounding context. NotificationBasicContent takes plain strings rather than Resource objects, so the values must be resolved through resourceManager before the request is built, which is the step the sample skips entirely.
- Fix: Add locale resource directories, move the four strings into string.json, and resolve them with resourceManager.getStringSync before constructing the NotificationRequest.

### `HW-06-0048` - Locating the user discards the zoom level they had chosen

- Category B, severity low, confidence confirmed
- Features: TRANS-08
- Document: `huawei_industry_tree/06_public_transport/docs/08_map_rotation_lock.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_rotation_lock-0000002396727869
- Why: The component tracks the live zoom in this.zoomSize, refreshes it from getCameraPosition when the map settles, and honours it in moveCamera along with the bearing. The locate path ignores all of that and passes the constant 16, so a user who has zoomed in to read street names and then taps locate is thrown back to the default level - and the bearing they set is dropped too, since the locate path omits the field entirely. The two camera moves in one small class therefore disagree about whether the user's view state survives. Recentring and re-zooming are separate intentions and the button only claims the first.
- Fix: Use this.zoomSize and this.bearing in the locate path as well, or route it through moveCamera() after updating this.center.

### `HW-06-0049` - Industry FAQ page has no content and redirects to the unfiltered phone FAQ index instead of the transit FAQs

- Category E, severity low, confidence confirmed
- Features: TRANS-09
- Document: `huawei_industry_tree/06_public_transport/docs/09_practice-bus-app-architecture-v1_2.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1_2-0000002591551329
- Why: The page is a migration stub whose entire body is one sentence and one link. The link target is faq-phone, the general phone FAQ index for all of HarmonyOS, with no anchor, category filter or search term for public transport. A developer who opens 行业常见问题 from the transit architecture guide is handed an unfiltered list and has no way to reach the industry FAQ content the sentence promises. This is byte-for-byte the same stub and the same target already recorded in the maternity health and automotive industries, so the redirect is shared boilerplate rather than a per-industry pointer - three of the three industries reviewed so far carry it.
- Fix: Point the link at the public-transport-filtered FAQ view, or list the migrated questions inline with direct links so the industry context survives the migration.

### `HW-06-0050` - 1 sample project depends on third-party packages through unpinned version ranges

- Category B, severity low, confidence confirmed
- Features: TRANS-01
- Document: `huawei_industry_tree/06_public_transport/docs/01_practice-bus-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-bus-app-architecture-v1-0000001938172420
- Why: A caret or tilde range lets ohpm resolve a different version than the one the sample was verified against, so the published sample is not reproducible and can break without any change to its own source. Sample projects are reference implementations; their dependency set should be exactly the one that was tested.
- Fix: Pin exact versions in oh-package.json5 dependencies, and record the resolved set in oh-package-lock.json5.

