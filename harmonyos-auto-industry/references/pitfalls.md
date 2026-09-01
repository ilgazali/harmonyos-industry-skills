# Pitfalls

> Generated from `features/*.md`. Source industry: `01_auto`, 8 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (1)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-01-0050` - Systematic: 5 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: AUTO-03, AUTO-04, AUTO-05, AUTO-06, AUTO-07
- Document: `huawei_industry_tree/01_auto/docs/05_call_phone.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/call_phone-0000002342413398
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (49)

### `HW-01-0031` - The sample's only page is declared @Entry without @Component, which the custom-component rules forbid

- Category B, severity blocker, confidence confirmed
- Features: AUTO-05
- Document: `huawei_industry_tree/01_auto/docs/05_call_phone.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/call_phone-0000002342413398
- Why: The ArkTS custom-component guide states that 'the definition of a custom component must start with the @Component struct followed by the component name', and every @Entry example in that guide pairs the two decorators. CallPhone declares only @Entry, yet the struct defines build() and uses state-management V1 decorators - @Provide and two @StorageLink members - which the same guide ties to @Component. PhoneSheet.ets in the same project decorates its struct with @Component correctly, so this is an omission rather than a project convention. This is the sample's single page and the mainElement of the module, so nothing else in the project can compensate for it.
- Fix: Add @Component beneath @Entry on CallPhone.

### `HW-01-0001` - checkPermissions only honours the last permission in the list, so a denied LOCATION is never requested

- Category B, severity high, confidence confirmed
- Features: AUTO-01
- Document: `huawei_industry_tree/01_auto/docs/01_practice-auto-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-auto-app-architecture-v1-0000001903742656
- Why: applyResult is assigned inside the loop rather than accumulated, so every iteration discards the previous verdict and only the final permission decides. ServicePage passes ['ohos.permission.LOCATION', 'ohos.permission.APPROXIMATELY_LOCATION'] in that order. If the user has granted approximate location but denied precise location - the exact split the two-permission model exists to express, and the state left behind by choosing 'approximate only' in the system dialog - the loop ends with applyResult true and the app never asks for LOCATION again. The map then silently runs without the precise fix it was written for.
- Fix: Start from true and fail closed: let applyResult = true; ... if (grantStatus !== PERMISSION_GRANTED) { applyResult = false; break; }

### `HW-01-0002` - Five map controller listeners are registered and never released, leaking on every visit to the map pages

- Category B, severity high, confidence confirmed
- Features: AUTO-01
- Document: `huawei_industry_tree/01_auto/docs/01_practice-auto-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-auto-app-architecture-v1-0000001903742656
- Why: OptimalStation registers three listeners and ServicePage two, and a search of the whole project finds no off() call and no aboutToDisappear or onWillDisappear on either page. Both are NavDestination pages the user enters and leaves repeatedly - ServicePage is a bottom-tab destination and OptimalStation is pushed from it and popped with the back arrow. Each visit registers another set of callbacks that close over the destroyed component instance, so the handlers accumulate, keep the old page objects alive, and markerClick and myLocationButtonClick eventually fire several times per tap. The mapLoad listener is also registered with an empty body, so it leaks for no purpose at all.
- Fix: Add aboutToDisappear() to both pages and release what was registered: this.mapController?.off('mapLoad'); this.mapController?.off('markerClick'); this.mapController?.off('myLocationButtonClick'); and delete the empty mapLoad registration.

### `HW-01-0003` - getCurrentLocation() has no rejection handler and runs before the permission request has resolved

- Category B, severity high, confidence confirmed
- Features: AUTO-01
- Document: `huawei_industry_tree/01_auto/docs/01_practice-auto-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-auto-app-architecture-v1-0000001903742656
- Why: Two faults compound. checkPermissions returns a Promise that aboutToAppear neither awaits nor inspects, so the map callback proceeds while the permission dialog is still on screen. getMyLocation then calls geoLocationManager.getCurrentLocation() with a then() and no catch(). That call rejects whenever location services are switched off, the permission was refused, or no fix is obtained before the timeout - all ordinary states - and the rejection is unhandled. The user is left on a map centred on the hardcoded fallback coordinate with no error surfaced and no retry path, and the same unguarded call is the myLocationButtonClick handler, so pressing the locate button in that state does nothing at all.
- Fix: Await the permission result and gate on it, then add a rejection branch: try { const result = await geoLocationManager.getCurrentLocation(); ... } catch (err) { Logger.error(...); /* surface a retry or a settings hint */ }

### `HW-01-0013` - Sub-window is created from two uncoordinated call sites with no guard, and closing it never clears the stored handle

- Category B, severity high, confidence confirmed
- Features: AUTO-03
- Document: `huawei_industry_tree/01_auto/docs/03_global_components.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/global_components-0000002313328742
- Why: Loading is a module-level singleton (export default new Loading()) whose only state is this.win, and OpenSubWindows overwrites it unconditionally. It is invoked from aboutToAppear and again from onVisibleAreaChange every time the first tab reaches full visibility, so a cold start calls it twice and every return to that tab calls it again, with nothing checking whether a window is already open. Whenever two creations succeed, the earlier handle is overwritten and that window can never be destroyed - the close button only ever reaches the newest one. CloseSubWindows compounds it: destroyWindow() returns a Promise that is neither awaited nor given a catch, so a failed destroy is an unhandled rejection, and this.win is left pointing at a destroyed window, so the next close attempt rejects too and the next open silently discards a stale handle.
- Fix: Guard on the existing handle and clean up properly: if (this.win) { return; } in OpenSubWindows, and async CloseSubWindows() { try { await this.win?.destroyWindow(); } catch (err) { hilog.error(...); } finally { this.win = undefined; } }. Then drop one of the two call sites.

### `HW-01-0020` - Tapping a second marker never resets the first, so markers accumulate enlarged with their info windows open

- Category B, severity high, confidence confirmed
- Features: AUTO-04
- Document: `huawei_industry_tree/01_auto/docs/04_nearby_gas_station.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/nearby_gas_station-0000002342199206
- Why: The handler enlarges the tapped marker and opens its info window, then overwrites this.curMarker with it. The only code that scales a marker back down lives in the bottom sheet's onWillDismiss. Tapping a second marker while the sheet is open sets isShow = true when it is already true, so the sheet never dismisses and onWillDismiss never runs - meanwhile curMarker has already been replaced, so the first marker can no longer be reached by any reset path. After tapping three stations the map shows three markers at 1.5x with three info windows open, and dismissing the sheet shrinks only the last one. This is exactly the interaction 实现思路 step 3 presents as the feature.
- Fix: Reset before selecting: at the top of the handler, if (this.curMarker && this.curMarker !== marker) { this.curMarker.setInfoWindowVisible(false); mapUtil.imageAnimation(this.curMarker, 1); } then assign the new marker.

### `HW-01-0021` - Permission result is never inspected, so the location is requested even after the user denies it

- Category B, severity high, confidence confirmed
- Features: AUTO-04
- Document: `huawei_industry_tree/01_auto/docs/04_nearby_gas_station.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/nearby_gas_station-0000002342199206
- Why: requestPermissionsFromUser resolves whenever the dialog completes, whether the user granted or refused - the verdict is carried in data.authResults, which this code does not even bind. So the then branch runs on refusal too, logs 'request Permissions success', and calls geoLocationManager.getCurrentLocation(). That call needs the permission it was just denied, so it rejects; its Promise is discarded on the spot with no await and no catch, making the refusal path an unhandled rejection rather than a handled state. The step-1 snippet in 实现思路 shows this same shape, so the document teaches it. The sibling maternity practice does this correctly - it loops over authResults and returns early on any non-zero entry.
- Fix: Bind the result and gate on it: .then((data: PermissionRequestResult) => { if (data.authResults.some(r => r !== 0)) { return; } ... }) and give the getCurrentLocation() call its own await plus catch.

### `HW-01-0022` - checkPermissions only honours the last permission in the list, so a denied LOCATION is never requested

- Category B, severity high, confidence confirmed
- Features: AUTO-04
- Document: `huawei_industry_tree/01_auto/docs/04_nearby_gas_station.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/nearby_gas_station-0000002342199206
- Why: applyResult is assigned inside the loop rather than accumulated, so each iteration discards the previous verdict and only the final permission decides. This sample needs LOCATION and APPROXIMATELY_LOCATION together; a user who granted approximate location but denied precise location - the state left behind by choosing approximate-only in the system dialog - ends the loop with applyResult true, so the app never asks again and the map runs without the precise fix the petrol-station distances are computed from. The identical defect ships in the automotive architecture practice's own PermissionsUtil, so it has been copied across samples in this industry rather than fixed once.
- Fix: Fail closed: let applyResult = true; ... if (grantStatus !== PERMISSION_GRANTED) { applyResult = false; break; }

### `HW-01-0036` - A 100 ms interval drives the gauge forever and is never cleared, with its id not even captured

- Category B, severity high, confidence confirmed
- Features: AUTO-06
- Document: `huawei_industry_tree/01_auto/docs/06_customize_dashboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/customize_dashboard-0000002369250730
- Why: Neither the setTimeout nor the setInterval handle is assigned to anything, and the page declares no aboutToDisappear, so there is no reference left with which to stop them and no lifecycle hook in which to try. The interval keeps firing after the page is destroyed, mutating @Provide state on a dead component. Each tick writes both speed and acceleration, and the Dashboard child watches both with @Watch('draw'), so one tick triggers two full canvas repaints - twenty complete redraws per second, indefinitely, each one re-querying the component rectangle and re-decoding a bitmap. speed also grows without bound; it passes maxSpeed after about twenty-four ticks and simply keeps climbing behind the clamp. The 仅作演示用 comment marks the values as demo data but does not make an uncancellable timer acceptable, and the document never mentions that the reader must add the cleanup.
- Fix: Capture both ids and release them: private timeoutId: number = -1; private intervalId: number = -1; assign in aboutToAppear, then aboutToDisappear(): void { clearTimeout(this.timeoutId); clearInterval(this.intervalId); }

### `HW-01-0037` - An ImageBitmap is allocated and decoded inside a draw routine, so it is rebuilt on every repaint

- Category B, severity high, confidence confirmed
- Features: AUTO-06
- Document: `huawei_industry_tree/01_auto/docs/06_customize_dashboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/customize_dashboard-0000002369250730
- Why: drawBigFill is called from draw(), which is bound to @Watch on both speed and acceleration. With the demo interval ticking every 100 ms and writing both values, draw() runs twenty times a second, so a file-backed ImageBitmap is constructed, decoded, drawn and closed twenty times a second on the UI thread for an image that never changes. draw() also re-queries getComponentUtils().getRectangleById('canvas') on every one of those passes to recompute a canvas size that is equally constant. Closing the bitmap on the line immediately after drawImage compounds the risk, since the decode the draw depends on is released the moment the draw is issued.
- Fix: Create the bitmap once in aboutToAppear into a member, reuse it in drawBigFill, and close it in aboutToDisappear. Cache canvasSize from onReady and on an area-change callback instead of recomputing it per frame.

### `HW-01-0043` - requestAutoSave is called without its result callback and a success toast is shown unconditionally

- Category B, severity high, confidence confirmed
- Features: AUTO-07
- Document: `huawei_industry_tree/01_auto/docs/07_car_maintenance.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/car_maintenance-0000002404899097
- Why: The reference gives the signature as requestAutoSave(context: UIContext, callback?: AutoSaveCallback): void and states that the callback is triggered when the auto-save request is complete; AutoSaveCallback carries onSuccess and onFailure. The sample passes no callback, so it has no way of learning the outcome, yet it shows 绑定信息成功 - binding successful - on the very next line. requestAutoSave raises a system prompt the user can dismiss, and it can fail outright, so the message is asserted before anything has happened and stays wrong whenever the save does not occur. The reference example additionally wraps the call in try/catch because it throws synchronously on a parameter error; the sample has neither. The user is told their owner details were saved to their Huawei account when they may not have been.
- Fix: Pass the callback and move the toast into it: try { autoFillManager.requestAutoSave(this.getUIContext(), { onSuccess: () => showToast(success), onFailure: () => showToast(failure) }); } catch (error) { Logger.error(...); }

### `HW-01-0004` - The permission and persistence layers swallow every error in empty catch blocks

- Category B, severity medium, confidence confirmed
- Features: AUTO-01
- Document: `huawei_industry_tree/01_auto/docs/01_practice-auto-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-auto-app-architecture-v1-0000001903742656
- Why: Six catch blocks across the two utility classes contain nothing but a '// ...' comment. In PermissionsUtil this means a failed getBundleInfoForSelf leaves tokenId at 0, checkAccessToken(0, permission) is then called with an invalid token, and its own failure is swallowed too, so an infrastructure error is reported to the caller as an ordinary denial. The requestPermissionsFromUser promise discards both its result and its error, so nothing in the app ever learns whether the user granted anything. In PreferencesUtil a failure to open the preferences file leaves this.pref null and every later read and write silently no-ops. The project ships a LoggerUtil in the same folder and uses it elsewhere, so the tooling to do this properly is already present.
- Fix: Log in every catch with the existing Logger and propagate: have checkPermissions resolve to a boolean, have requestPermissions inspect data.authResults, and have PreferencesUtil surface failures to the caller instead of returning silently.

### `HW-01-0005` - getPreferencesValue returns false instead of the caller's default, and writes are never flushed to disk

- Category B, severity medium, confidence confirmed
- Features: AUTO-01
- Document: `huawei_industry_tree/01_auto/docs/01_practice-auto-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-auto-app-architecture-v1-0000001903742656
- Why: Three defects in one small class. First, the caller passes defValue and gets false back when this.pref is null - the store failing to open silently inverts the answer for any caller whose default is true, such as a first-launch or privacy-accepted flag. Second, putPreferencesValue is declared async so callers await it, but flush() returns a Promise that is never awaited, so the call resolves before the data reaches the sandbox file and a write followed by an immediate exit is lost. Third, the read casts unconditionally to boolean even though the signature accepts the full ValueType, so any non-boolean value read back is mistyped. Since loadPreferences swallows its own failure, none of this is observable at the call site.
- Fix: Return defValue rather than false, await this.pref.flush(), and make the class generic over ValueType instead of casting to boolean. Also drop the unused name parameter from hasPreferenceskey.

### `HW-01-0006` - Project tree disagrees with the shipped sample in six places, including a misspelled directory and three files that do not exist

- Category E, severity medium, confidence confirmed
- Features: AUTO-01
- Document: `huawei_industry_tree/01_auto/docs/01_practice-auto-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-auto-app-architecture-v1-0000001903742656
- Why: The sample ships its shared component folder as compontents, a misspelling the document silently corrects to components, so every import path a reader copies from the tree fails to resolve. Three files the tree advertises are simply not in the archive: the buying-car swiper item, the explore real-time information component, and the second splash screen. One is renamed, and the buyingCar module directory differs in case, which breaks on case-sensitive build hosts. A file the sample does ship, BrandModel.ets, is missing from the tree. 工程目录 is the section a developer uses to navigate an unfamiliar sample, and six divergences make it unusable for that.
- Fix: Regenerate the tree from the archive, and fix the misspelled compontents directory and the buyingcar case in the sample itself rather than papering over them in the document.

### `HW-01-0007` - The code snippets in the document are an older revision than the sample and will not compile against it

- Category E, severity medium, confidence confirmed
- Features: AUTO-01
- Document: `huawei_industry_tree/01_auto/docs/01_practice-auto-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-auto-app-architecture-v1-0000001903742656
- Why: Both featured snippets have drifted from the archive they claim to quote. The scan example still passes the global getContext(this); the sample has already migrated to this.getUIContext().getHostContext(), the UIContext form the ArkUI guide recommends. The document also decorates OptimalStation with @Entry, which the sample does not, and @Entry on a struct that is only ever rendered as a NavDestination inside a Navigation makes it a standalone page entry as well. Worst of all, the SheetTransition call in the document uses StationList and Station where the sample exports stationList and station, so that snippet fails to compile verbatim - and these are exactly the two blocks 行业关键技术方案 points readers at as 代码参考.
- Fix: Regenerate both snippets from the shipped source, and keep them regenerated when the sample is updated.

### `HW-01-0008` - Internationalisation is listed as a common-module responsibility but 64 UI strings are hardcoded Chinese and the en_US resources are near-empty

- Category E, severity medium, confidence confirmed
- Features: AUTO-01
- Document: `huawei_industry_tree/01_auto/docs/01_practice-auto-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-auto-app-architecture-v1-0000001903742656
- Why: Every module in the sample ships both a zh_CN and an en_US resource directory, so the localisation scaffolding is deliberately in place and the module table promises it as a common-layer capability. The scaffolding is then almost entirely unused: features/service declares one en_US string while ServicePage hardcodes five Chinese literals, and features/buyingCar declares one while BuyingCarDetailPage hardcodes fifteen. Across the sample 64 Text() calls carry Chinese literals in code. An English-locale device therefore renders a mostly Chinese interface, and a developer adapting this framework has to hunt those strings out of twelve files by hand - which is precisely the work the 通用 module was said to have absorbed.
- Fix: Move the 64 literals into each module's string.json, provide the en_US translations next to the zh_CN ones, and reference them with $r('app.string.*') as LoginPage already does for part of its own text.

### `HW-01-0009` - Document scopes the practice to phones but the sample declares phone, tablet and 2in1 device types

- Category E, severity medium, confidence confirmed
- Features: AUTO-01
- Document: `huawei_industry_tree/01_auto/docs/01_practice-auto-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-auto-app-architecture-v1-0000001903742656
- Why: The introduction, the software view and the hardware requirements all restrict this practice to a phone, and the software view goes as far as 只涉及手机端 - the phone end only. The shipped manifest nonetheless declares tablet and 2in1, so the app is distributed to and installed on devices whose layout the document never discusses and whose breakpoints the sample never handles - unlike the maternity practice, this project has no breakpoint utility at all. A reader cannot tell whether the wider declaration is intentional.
- Fix: Narrow deviceTypes to ['phone'] to match the stated scope, or extend the document to cover the tablet and 2in1 layouts and add the responsive handling the sample lacks.

### `HW-01-0010` - PreferencesUtil imports through legacy @ohos.* module paths while the rest of the sample uses @kit.* entry points

- Category A, severity medium, confidence confirmed
- Features: AUTO-01
- Document: `huawei_industry_tree/01_auto/docs/01_practice-auto-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-auto-app-architecture-v1-0000001903742656
- Why: Every other file in this sample imports through Kit entry points - ServicePage pulls @kit.MapKit, @kit.ScanKit, @kit.LocationKit, @kit.AbilityKit and @kit.BasicServicesKit, and PermissionsUtil in the very same utils folder imports @kit.AbilityKit for the same 'common' namespace this file reaches for through @ohos.app.ability.common. The project targets compatibleSdkVersion 6.0.0(20), well past the point where the Kit entry points became the documented form. The result is one shared utility folder demonstrating two contradictory import conventions for the same namespace, which is what a developer copies first.
- Fix: Replace both imports with their Kit equivalents: import { common } from '@kit.AbilityKit'; import { preferences } from '@kit.ArkData';

### `HW-01-0014` - Sub-window vertical position is a magic pixel offset frozen at module load, ignoring the navigation-bar inset the app already measures

- Category B, severity medium, confidence confirmed
- Features: AUTO-03
- Document: `huawei_industry_tree/01_auto/docs/03_global_components.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/global_components-0000002313328742
- Why: 370 is an unexplained literal, and subtracting it from the raw display height assumes one specific device geometry. The same application already does the correct measurement: EntryAbility reads window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR, publishes it as AppStorage 'bottomRectHeight', and even subscribes to avoidAreaChange to keep it current - and MainPage consumes it for its own padding. The sub-window ignores all of that. Worse, SUB_WINDOW_Y is a static readonly initialised from display.getDefaultDisplaySync() when the module is first loaded, so it is computed once per process and never reflects a rotation, a fold, or a multi-window resize, while the width passed to resize() on the line below is re-read from the display at every open. The notification therefore floats at the wrong height on any device whose bottom system area differs from the one this constant was tuned on.
- Fix: Compute the position at open time from the measured inset: const bottom = AppStorage.get<number>('bottomRectHeight') ?? 0; windowClass.moveWindowTo(0, display.getDefaultDisplaySync().height - bottom - Constants.SUB_WINDOW_HEIGHT); and re-position on avoidAreaChange.

### `HW-01-0015` - avoidAreaChange listener is registered on the main window and never released

- Category B, severity medium, confidence confirmed
- Features: AUTO-03
- Document: `huawei_industry_tree/01_auto/docs/03_global_components.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/global_components-0000002313328742
- Why: The listener is attached to the main window in onWindowStageCreate and holds a closure over the ability. onWindowStageDestroy exists in the file but does nothing except log, so the subscription outlives the window stage it was created for. On a UIAbility that is destroyed and recreated - the ordinary consequence of a configuration change or of the ability being restarted from a cold background - each cycle attaches another listener while the previous one is still registered, so every avoid-area change fans out to a growing set of handlers all writing the same AppStorage keys. This is the documented lifecycle slot for the cleanup and it is empty.
- Fix: Keep the window reference on the ability and release in the matching lifecycle hook: onWindowStageDestroy(): void { this.windowClass?.off('avoidAreaChange'); }

### `HW-01-0016` - createSubWindow is called without the surrounding try/catch and null check the reference example requires

- Category A, severity medium, confidence confirmed
- Features: AUTO-03
- Document: `huawei_industry_tree/01_auto/docs/03_global_components.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/global_components-0000002313328742
- Why: The official createSubWindow example wraps the call in try { windowStage.createSubWindow(...) } catch (exception) { ... }, because the API throws synchronously on a parameter error rather than reporting it through the callback. The sample puts its try inside the callback instead, so a synchronous throw escapes OpenSubWindows entirely and, since neither call site awaits or guards it, propagates out of aboutToAppear. The reference example also checks 'if (!windowClass)' before using the window; the sample assigns this.win = windowClass and dereferences it immediately, so a callback that reports success with no window produces a null dereference rather than the logged message the reference shows.
- Fix: Move the try to enclose the createSubWindow call, keep the inner try for the configuration chain, and null-check the callback argument before assigning this.win.

### `HW-01-0017` - The document's snippet does not compile and strips both error logs, teaching an empty catch

- Category E, severity medium, confidence confirmed
- Features: AUTO-03
- Document: `huawei_industry_tree/01_auto/docs/03_global_components.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/global_components-0000002313328742
- Why: Two separate faults. The document's version takes no parameter and reads this.context, but the class it belongs to declares only 'win: window.Window | undefined' - there is no context member, so the snippet as printed does not compile. And both hilog.error calls present in the sample have been removed, leaving a bare return on the failure branch and a completely empty catch block. 实现思路 is the section a reader copies from, so the document teaches silent swallowing of exactly the two failures the sample was careful to log. Notably the same document set already carries this defect elsewhere in this industry, where the architecture practice's snippets are likewise an older revision than the archive.
- Fix: Regenerate the snippet from entry/src/main/ets/utils/GlobalComponents.ets, keeping the parameter and both error logs.

### `HW-01-0023` - One screen triggers three separate GNSS fixes, and the latitude and longitude are taken from two different ones

- Category B, severity medium, confidence confirmed
- Features: AUTO-04
- Document: `huawei_industry_tree/01_auto/docs/04_nearby_gas_station.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/nearby_gas_station-0000002342199206
- Why: getMyLocation() wraps geoLocationManager.getCurrentLocation(), a one-shot positioning request. moveToMyLocation calls it once internally, then the next two lines call it again, once per coordinate - three GNSS acquisitions for a single page load, each with its own battery and latency cost. Worse, the two awaited calls are independent fixes taken seconds apart, so currentLatitude comes from one position and currentLongitude from another. For a stationary user the difference is GPS jitter, but for a user in a moving vehicle - the entire premise of an automotive petrol-station finder - the two halves describe different places, and every station distance computed from that pair is wrong. moveToMyLocation is also fired without await while the lines below depend on the same underlying fix.
- Fix: Take the fix once and reuse it: const location = await mapUtil.getMyLocation(); this.currentLatitude = location.latitude; this.currentLongitude = location.longitude; then pass it to the camera move instead of letting moveToMyLocation acquire its own.

### `HW-01-0024` - Markers are added from an async callback passed to forEach, so nothing awaits them and their failures are unhandled

- Category B, severity medium, confidence confirmed
- Features: AUTO-04
- Document: `huawei_industry_tree/01_auto/docs/04_nearby_gas_station.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/nearby_gas_station-0000002342199206
- Why: Array.prototype.forEach ignores the value its callback returns, so the promise each async callback produces is dropped. The map initialisation callback therefore finishes while the marker additions are still in flight, and any rejection from addMarker - an invalid coordinate in the mock data, a controller torn down while the user navigates away mid-load - becomes an unhandled rejection with no place to report it. The await inside the callback also gives a false impression of sequencing: the iterations do not run one after another, they all start at once. This is the marker-adding step the document highlights, and it is the pattern a reader will copy for their own station list.
- Fix: Use map() plus Promise.all and wrap it in try/catch so a single bad station is reported rather than silently lost.

### `HW-01-0025` - Three event listeners are registered and never released

- Category B, severity medium, confidence confirmed
- Features: AUTO-04
- Document: `huawei_industry_tree/01_auto/docs/04_nearby_gas_station.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/nearby_gas_station-0000002342199206
- Why: A search of the whole sample finds three on() registrations and no off() call at all; GasStationPage has no aboutToDisappear and EntryAbility's onWindowStageDestroy does not release the window listener. GasStationPage is a NavDestination reached from MainPage and left with the back arrow, so every visit attaches another pair of map callbacks that close over the destroyed component instance. The markerClick handler in particular mutates this.curMarker and this.isShow, so stale handlers keep writing into dead component state. The Map Kit guide documents off('myLocationButtonClick') alongside the on() it pairs with, so the counterpart API is in plain view.
- Fix: Add aboutToDisappear(): void { this.mapController?.off('markerClick'); this.mapController?.off('myLocationButtonClick'); } to the page, and release avoidAreaChange in onWindowStageDestroy.

### `HW-01-0026` - All three permissions declare usedScene against ShopAbility, an ability that does not exist in the module

- Category E, severity medium, confidence confirmed
- Features: AUTO-04
- Document: `huawei_industry_tree/01_auto/docs/04_nearby_gas_station.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/nearby_gas_station-0000002342199206
- Why: The module declares exactly one ability, EntryAbility, plus a backup extension ability. ShopAbility appears nowhere in the manifest, nowhere in the source, and nowhere in the project - it is a copy-paste remnant from a different sample. usedScene is the field that tells the system which abilities need each permission and underpins how the permission is presented and scoped, so pointing all three at a non-existent ability leaves the declarations describing something that cannot happen. The document devotes a whole 权限说明 section to these three permissions and never mentions the scope, so a reader copying the manifest inherits the dangling reference.
- Fix: Replace ShopAbility with EntryAbility in all three usedScene blocks.

### `HW-01-0027` - The permission snippet does not compile and strips the error logging, leaving an empty catch

- Category E, severity medium, confidence confirmed
- Features: AUTO-04
- Document: `huawei_industry_tree/01_auto/docs/04_nearby_gas_station.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/nearby_gas_station-0000002342199206
- Why: The printed method takes only permissions, yet passes context to requestPermissionsFromUser - there is no such parameter, no such class member, and no import that would put a context in scope, so the snippet does not compile. The sample's signature takes context as a second parameter. The document also deletes the Logger.error call, leaving a catch body that is a bare comment, so the one code block a reader copies for permission handling teaches silent swallowing. Its then branch additionally binds data: PermissionRequestResult and then ignores it, which is what makes the denial path proceed as if granted. This is the third document in this industry whose snippets have drifted from the archive they quote.
- Fix: Regenerate the snippet from entry/src/main/ets/utils/PermissionsUtil.ets, keeping the context parameter and the Logger calls, and fix the underlying authResults omission in both places at once.

### `HW-01-0032` - The sheet's cancel and confirm callbacks are wired up but never invoked, so the confirm handler is dead code

- Category B, severity medium, confidence confirmed
- Features: AUTO-05
- Document: `huawei_industry_tree/01_auto/docs/05_call_phone.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/call_phone-0000002342413398
- Why: Step 1 of 实现思路 presents the cancel and confirm closures as how the sheet reports back to the page, and the page duly defines sheetCancel() and sheetConfirm() to receive them. But PhoneSheetItem only ever assigns params - the identifier appears exactly twice in the file, at the constructor call and at its own declaration, and neither params.cancel() nor params.confirm() is called anywhere. The sheet closes by mutating the @Consume'd isShowSheet directly, and cancellation happens to still work because the page also hooks onDisappear. sheetConfirm() therefore never runs at all: it is referenced only at the point where it is handed to a slot nothing reads. A developer copying this structure gets a callback contract that silently does nothing.
- Fix: Invoke the callbacks from the sheet - this.params.cancel() on the close control and this.params.confirm() on the confirm path - or drop the Tmp interface and let the @Consume'd flag be the only channel.

### `HW-01-0033` - Phone numbers are resolved with a synchronous resource lookup inside build(), once per row per render

- Category C, severity medium, confidence confirmed
- Features: AUTO-05
- Document: `huawei_industry_tree/01_auto/docs/05_call_phone.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/call_phone-0000002342413398
- Why: getStringSync is a blocking resource-manager call, and both helpers are invoked from inside build(), so every render pass of the sheet performs two synchronous lookups on the UI thread. The rest of the same builder passes Resource objects directly - $r('app.media.ic_green_phone') and $r('app.string.store_phone') go through untouched - so the file demonstrates both the declarative and the blocking way to read a resource, side by side, for no stated reason. The values are also constant for the lifetime of the component, so the repeated work buys nothing. The same anti-pattern appears in the maternity-health growth-curve sample, so it is spreading between practices.
- Fix: Resolve once into @State members in aboutToAppear, or keep the numbers as plain string constants and use Resource only for the labels.

### `HW-01-0034` - The makeCall snippet reduces the error callback to an empty body, deleting the handling the sample actually has

- Category E, severity medium, confidence confirmed
- Features: AUTO-05
- Document: `huawei_industry_tree/01_auto/docs/05_call_phone.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/call_phone-0000002342413398
- Why: makeCall reports its outcome exclusively through this AsyncCallback - the reference gives the signature as makeCall(phoneNumber: string, callback: AsyncCallback<void>): void, so the callback is the only place a failure can surface. The document prints it with an empty body, which is not shorthand for the sample's code but the opposite of it: the sample checks err and logs both branches. 步骤 2 is a two-line snippet whose entire content is that call, so the omission is the whole lesson. This is the fourth document in this industry whose code blocks have been stripped of the error handling present in the archive, alongside the architecture practice, the bottom-sheet practice and the petrol-station practice.
- Fix: Regenerate the snippet from entry/src/main/ets/component/PhoneSheet.ets, keeping the err check and both log branches.

### `HW-01-0038` - drawCircleInside overwrites the shared start and end angles that the two arcs drawn before it depend on

- Category B, severity medium, confidence confirmed
- Features: AUTO-06
- Document: `huawei_industry_tree/01_auto/docs/06_customize_dashboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/customize_dashboard-0000002369250730
- Why: startAngle and endAngle are plain instance fields read by drawCircleBig and drawCircleMiddle, both of which run before drawCircleInside in the same pass. On the very first draw they see the declared 70 and 20; drawCircleInside then assigns 69 and 21, and every later draw - and the gauge redraws twenty times a second - renders the outer track and the coloured speed arc one degree off from where the first frame put them. The shift is permanent because nothing ever restores the original values. The same field, this.radius, is likewise reused as a scratch variable by six different draw methods, which is what makes this class of defect easy to introduce here.
- Fix: Make the angles local to each draw method, or give the inner ring its own named constants: const innerStart = 69; const innerEnd = 21; and leave the component fields read-only.

### `HW-01-0039` - Two arc calls pass degrees where the API takes radians, in a file that converts correctly everywhere else

- Category C, severity medium, confidence confirmed
- Features: AUTO-06
- Document: `huawei_industry_tree/01_auto/docs/06_customize_dashboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/customize_dashboard-0000002369250730
- Why: CanvasRenderingContext2D.arc takes its start and end angles in radians. Passing 360 asks for an arc spanning 360 radians, about fifty-seven full turns. The rendered result still looks like a complete circle because the sweep exceeds a full revolution, which is exactly why the mistake survives - it is invisible until someone reuses the line for a partial arc, where 90 would draw a full circle instead of a quarter. The component defines degToRad and applies it to every other angle it passes, including the visually identical full circle in drawCircleSmall, so the file demonstrates both the correct and the incorrect form for the same operation without distinguishing them.
- Fix: Replace both bare 360 values with this.degToRad(360), matching drawCircleSmall.

### `HW-01-0040` - Gauge text is positioned with canvas-relative x but hardcoded absolute y, so the labels detach from the dial on other screen sizes

- Category C, severity medium, confidence confirmed
- Features: AUTO-06
- Document: `huawei_industry_tree/01_auto/docs/06_customize_dashboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/customize_dashboard-0000002369250730
- Why: Every arc, radius and gradient in this component is expressed as a fraction of canvasSize, which is measured from the actual laid-out canvas. The text is only half converted: the x values follow canvasSize while the y values - 140, 220, 290, 310 - are fixed numbers, and the 'km/h' label has both coordinates hardcoded at 175 and 140. Canvas height is set with .height('50%') and the width comes from the parent, so canvasSize varies with the device; on anything wider or narrower than the reference device the dial scales while the readouts stay put, and the speed value drifts out of the ring it belongs to. The 200 passed as fillText's maxWidth is a fixed pixel budget for the same reason.
- Fix: Express the y coordinates and the maxWidth as fractions of canvasSize the way the x coordinates already are.

### `HW-01-0044` - This practice requires API 24 while every other document in the industry requires API 20, and it renames the constraints section

- Category E, severity medium, confidence confirmed
- Features: AUTO-07
- Document: `huawei_industry_tree/01_auto/docs/07_car_maintenance.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/car_maintenance-0000002404899097
- Why: Every other document in this industry - the architecture practice and the four other feature practices - states API Version 20 and DevEco Studio 6.0.0 under a heading called 约束与限制. This one states API Version 24 and DevEco Studio 6.1.1 under a differently named heading, 环境准备, and drops the HarmonyOS SDK line entirely. The manifest confirms the sample really is built against 6.1.1(24), so the requirement is genuine, not a typo. The consequence is that the industry's own scenario index presents five feature practices as peers while one of them silently raises the whole project's toolchain floor by four API versions. A developer following 关键场景示例 to assemble the features has no signal about that until the build fails, because the requirement is stated only inside the one document and under a heading the reader is not looking for.
- Fix: Use 约束与限制 with the same three-line shape as the sibling documents, and note in 关键场景示例 which practices carry a higher API floor.

### `HW-01-0045` - Two of three field placeholders and both toasts are hardcoded Chinese although the project ships en_US resources

- Category E, severity medium, confidence confirmed
- Features: AUTO-07
- Document: `huawei_industry_tree/01_auto/docs/07_car_maintenance.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/car_maintenance-0000002404899097
- Why: The owner-info card renders three fields with the same structure. The first takes its placeholder from a string resource; the two below it hardcode Chinese literals, so the file demonstrates both conventions three lines apart for identical widgets. Both user-facing toasts on the bind button are hardcoded too. The module ships base, dark, en_US and zh_CN resource directories and uses $r for every label in the same component, so the localisation path exists and is deliberately in place - these five strings simply bypass it. On an English-locale device the form shows an English name placeholder above two Chinese ones, and both confirmation messages come back in Chinese.
- Fix: Move the two placeholders and the two toast messages into string.json alongside name_placeholder and reference them with $r.

### `HW-01-0011` - Nullish coalescing applied to a comparison that can never be null, masking the intended verification-code length check

- Category B, severity low, confidence confirmed
- Features: AUTO-01
- Document: `huawei_industry_tree/01_auto/docs/01_practice-auto-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-auto-app-architecture-v1-0000001903742656
- Why: ?? binds looser than >, so the expression parses as (this.password.length > 5) ?? false. The left operand is a boolean and can never be null or undefined, so the ?? false branch is unreachable and the operator does nothing. It reads as a guard against an absent password, which it is not - password is declared @State password: string = '' and is never nullable. A reader copying this line inherits a defensive-looking construct that provides no defence, and the real intent, guarding a six-digit verification code, is obscured.
- Fix: Delete the ?? false. If a length floor for the code path is what was meant, name it: this.password.length >= CommonConstants.VERIFY_CODE_LENGTH.

### `HW-01-0012` - Industry FAQ page has no content and redirects to the unfiltered phone FAQ index instead of the automotive FAQs

- Category E, severity low, confidence confirmed
- Features: AUTO-08
- Document: `huawei_industry_tree/01_auto/docs/08_practice-auto-app-architecture-v1_2.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-auto-app-architecture-v1_2-0000002297842985
- Why: The page is a migration stub whose entire body is one sentence and one link. The link target is faq-phone, the general phone FAQ index for all of HarmonyOS, with no anchor, category filter or search term for the automotive industry. A developer who opens 行业常见问题 from the automotive architecture guide is handed an unfiltered list and has no way to reach the industry FAQ content the sentence promises. This is byte-for-byte the same stub and the same target as the maternity health industry's FAQ page, which confirms the redirect is a shared boilerplate rather than a per-industry pointer.
- Fix: Point the link at the automotive-filtered FAQ view, or list the migrated questions inline with direct links so the industry context survives the migration.

### `HW-01-0018` - The not-visible branch of onVisibleAreaChange logs that the sub window is visible

- Category B, severity low, confidence confirmed
- Features: AUTO-03
- Document: `huawei_industry_tree/01_auto/docs/03_global_components.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/global_components-0000002313328742
- Why: The second branch fires precisely when the content has become invisible, and logs 'sub window is visible'. Anyone reading the log to understand the sub-window lifecycle - which is the whole subject of this practice - is told the opposite of what occurred. The branch also does nothing else, so it reads as a placeholder where the matching CloseSubWindows call was meant to go, which would have prevented the duplicate-creation defect recorded separately for this document.
- Fix: Either log 'tab content hidden' or, better, call Loading.CloseSubWindows() there so creation and destruction are symmetric.

### `HW-01-0019` - Four of the five tabs render the same page, including the ones the project tree labels home and service

- Category E, severity low, confidence confirmed
- Features: AUTO-03
- Document: `huawei_industry_tree/01_auto/docs/03_global_components.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/global_components-0000002313328742
- Why: The tree presents three page files as home, car selection and mine, but the tab host wires four of five tabs - home, shop, select car and service - to BuyingCarPage. There is no home page content anywhere in the sample, even though MainPage.ets is annotated 首页 in the tree; MainPage is the tab host, not a home screen. A developer using the tree to find where the home tab is implemented has nowhere to land. The duplication is harmless to the bottom-sheet mechanism this document is about, but it is undocumented and the tree actively points the wrong way.
- Fix: Note in 工程目录 that the non-mine tabs are placeholders sharing BuyingCarPage, and correct the MainPage.ets comment to describe it as the tab host rather than 首页.

### `HW-01-0028` - Project tree names the backup ability file EntryBackAbility.ets but the sample ships EntryBackupAbility.ets

- Category E, severity low, confidence confirmed
- Features: AUTO-04
- Document: `huawei_industry_tree/01_auto/docs/04_nearby_gas_station.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/nearby_gas_station-0000002342199206
- Why: The tree drops the 'up' from Backup. The manifest's extensionAbilities entry points at EntryBackupAbility.ets, which is what the archive contains, so the document is the side that is wrong. The neighbouring practice in this same industry prints the same file correctly as EntryBackupAbility.ets, so the two trees disagree with each other as well as with the code.
- Fix: Correct the tree entry to EntryBackupAbility.ets.

### `HW-01-0029` - MapUtil carries a comment describing code it does not contain, an @Observed decorator with nothing to observe, and a pointless async

- Category B, severity low, confidence confirmed
- Features: AUTO-04
- Document: `huawei_industry_tree/01_auto/docs/04_nearby_gas_station.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/nearby_gas_station-0000002342199206
- Why: Three small inconsistencies in one utility class that a reader is meant to copy. The comment warns about a synchronous conversion method on the line above an awaited asynchronous call, so it describes an earlier revision and now misleads about which API is in use. @Observed makes a class's property changes trackable by @ObjectLink, but MapUtil declares no properties at all - it is a stateless collection of methods exported as a singleton, so the decorator adds observation machinery around nothing. imageAnimation is declared async and returns Promise<void> while containing no await, so every caller either awaits a promise that was already resolved or, as in the markerClick handler, ignores a promise that never needed to exist.
- Fix: Delete the stale comment, drop @Observed from MapUtil, and make imageAnimation a plain void method.

### `HW-01-0030` - Sample declares wearable support with no responsive handling, and disagrees with the device set of the industry's own architecture practice

- Category F, severity low, confidence confirmed
- Features: AUTO-04
- Document: `huawei_industry_tree/01_auto/docs/04_nearby_gas_station.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/nearby_gas_station-0000002342199206
- Why: This sample adds wearable to the three device types the industry's architecture practice declares, so the two projects in the same industry disagree about where an automotive app runs. Nothing in the sample adapts to a watch: there is no breakpoint utility, no GridRow, no deviceType branch anywhere in the source, and the resource directory carries only base, dark, rawfile and zh_CN with no device-specific qualifiers. A full-screen MapComponent with a bottom sheet and a station list is therefore installed unchanged on a watch face. 约束与限制 lists three tool versions and says nothing about devices at all, so the reader has no statement to check the manifest against.
- Fix: Drop wearable unless a watch layout is actually provided, and state the supported device types in 约束与限制 so the manifest can be verified against the document.

### `HW-01-0035` - An async arrow is assigned to a void-returning callback slot, discarding a promise for a synchronous body

- Category B, severity low, confidence confirmed
- Features: AUTO-05
- Document: `huawei_industry_tree/01_auto/docs/05_call_phone.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/call_phone-0000002342413398
- Why: Tmp declares confirm as () => void, and an async arrow evaluates to Promise<void>. Assigning it means whatever the function returns is dropped at the call site, so any rejection inside it would be unobservable. Here the body is one synchronous assignment - sheetConfirm() only sets a boolean - so the async keyword adds a promise allocation and an unhandled-rejection surface in exchange for nothing. The cancel handler immediately above it is written correctly as a plain arrow, so the two halves of the same object literal disagree. The document reproduces this asymmetry verbatim in step 1.
- Fix: Drop the async keyword so confirm matches its declared type and its sibling.

### `HW-01-0041` - Clamp branches contain expressions that cancel to a constant, and the document reproduces one of them

- Category B, severity low, confidence confirmed
- Features: AUTO-06
- Document: `huawei_industry_tree/01_auto/docs/06_customize_dashboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/customize_dashboard-0000002369250730
- Why: Both else branches were produced by copying the if branch and substituting the maximum for the current value, which leaves the maximum multiplied and then divided by itself. this.maxSpeed * 310 / this.maxSpeed is 310 and this.maxAcceleration * 360 / this.maxAcceleration is 360, for any non-zero maximum. The expressions are not merely redundant: they read as though the clamp still depends on maxSpeed, so a reader tuning the gauge will look for a relationship that is not there, and they introduce a division by a value that is only guaranteed non-zero by convention. 实现思路 prints the first of the two as the document's single code sample.
- Fix: Write the saturated branches as the constants they evaluate to, and hoist 310 and 360 into named constants describing the sweep of each dial.

### `HW-01-0042` - Within one sentence, two canvas APIs link to the HarmonyOS reference and the third links to the atomic-service documentation

- Category E, severity low, confidence confirmed
- Features: AUTO-06
- Document: `huawei_industry_tree/01_auto/docs/06_customize_dashboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/customize_dashboard-0000002369250730
- Why: The three APIs are described in a single sentence as three steps of one operation, and all three are ordinary CanvasRenderingContext2D methods documented in the HarmonyOS reference. The stroke link alone points into atomic-ascf, the atomic service API set, which is a different documentation product with a different scope and a different navigation tree. A reader following the sentence is dropped out of the HarmonyOS references between step two and step three, and the 参考文档 section at the foot of the same document links CanvasRenderingContext2D to harmonyos-references, so the document contradicts itself about where these APIs are documented.
- Fix: Point the stroke link at the same harmonyos-references page as its two neighbours.

### `HW-01-0046` - Project tree omits the backup ability and labels EntryAbility as style constants

- Category E, severity low, confidence confirmed
- Features: AUTO-07
- Document: `huawei_industry_tree/01_auto/docs/07_car_maintenance.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/car_maintenance-0000002404899097
- Why: Two defects in four lines. The comment on EntryAbility.ets reads 样式常量 - style constants - which is the comment belonging to StyleConstants.ets on the line above it, copied down one entry; EntryAbility is the application entry point and has nothing to do with styling. And the entrybackupability directory, which the archive contains and which module.json5 registers as an extension ability, is missing from the tree entirely, even though the four sibling practices in this industry all list it. A reader using 工程目录 to understand the project is told the wrong thing about one file and nothing at all about another.
- Fix: Correct the EntryAbility comment to describe the entry point and add the entrybackupability entry, matching the sibling documents.

### `HW-01-0047` - Type assertion is applied to px2vp's return value instead of its argument, inside a padding computed on every render

- Category B, severity low, confidence confirmed
- Features: AUTO-07
- Document: `huawei_industry_tree/01_auto/docs/07_car_maintenance.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/car_maintenance-0000002404899097
- Why: AppStorage.get returns the stored value as possibly undefined, and px2vp takes a number, so the argument is the thing that needs narrowing. Writing the assertion outside the call asserts px2vp's return type, which is already number, and leaves the untyped value flowing into the parameter unchecked - the assertion is in the one position where it does nothing. EntryAbility does set both keys in onWindowStageCreate, so the values are present in practice and the mistake is invisible, but the same expression is also evaluated inside build()'s padding, so both AppStorage lookups and both conversions run on every render pass of the page rather than being read once into state.
- Fix: Move the assertion inside the call, and read the two values once into @StorageProp members as the other practices in this industry do.

### `HW-01-0048` - OwnerInfoOption interface is declared in the model and never used anywhere

- Category B, severity low, confidence confirmed
- Features: AUTO-07
- Document: `huawei_industry_tree/01_auto/docs/07_car_maintenance.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/car_maintenance-0000002404899097
- Why: The interface appears exactly once in the project, at its own declaration - nothing imports it and nothing constructs a value of it. Its three members describe precisely the three things that vary between the form's rows, so it was clearly written to let OwnerInfo.ets render the fields from a list. Instead OwnerInfo.ets spells out three near-identical Row blocks by hand, which is what allowed two of the three placeholders to drift into hardcoded Chinese while the first stayed a resource. The dead interface therefore documents the design the file was meant to have and does not.
- Fix: Build the rows with ForEach over an array of OwnerInfoOption, which also removes the duplication that produced the inconsistent placeholders, or delete the interface.

### `HW-01-0049` - 1 sample project depends on third-party packages through unpinned version ranges

- Category B, severity low, confidence confirmed
- Features: AUTO-01
- Document: `huawei_industry_tree/01_auto/docs/01_practice-auto-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-auto-app-architecture-v1-0000001903742656
- Why: A caret or tilde range lets ohpm resolve a different version than the one the sample was verified against, so the published sample is not reproducible and can break without any change to its own source. Sample projects are reference implementations; their dependency set should be exactly the one that was tested.
- Fix: Pin exact versions in oh-package.json5 dependencies, and record the resolved set in oh-package-lock.json5.

