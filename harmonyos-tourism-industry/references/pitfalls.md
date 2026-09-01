# Pitfalls

> Generated from `features/*.md`. Source industry: `09_tourism`, 13 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (1)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-09-0080` - Systematic: 9 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: TOUR-03, TOUR-04, TOUR-05, TOUR-06, TOUR-08, TOUR-09, TOUR-10, TOUR-11, TOUR-12
- Document: `huawei_industry_tree/09_tourism/docs/12_add_and_collect_map_marker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_and_collect_map_marker-0000002402495269
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (79)

### `HW-09-0002` - MapPage registers 'mapLoad' and 'markerClick' map listeners but never unregisters them, and never releases the map controller.

- Category B, severity high, confidence confirmed
- Features: TOUR-01
- Document: `huawei_industry_tree/09_tourism/docs/01_practice-tourist-park-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1-0000001965211653
- Why: The official Map Kit FAQ shows the required teardown: "aboutToDisappear(): void { this.mapEventManager?.off('mapLoad'); this.mapController?.clear(); }". Every subscription opened with on() must be closed with off(); the markerClick callback captures `this` and calls this.dialogController.open(), so the retained callback keeps the destroyed component alive and can open a dialog on a dead component.
- Fix: Store the event manager (`this.mapEventManager = this.mapController.getEventManager()`), register through it, and in aboutToDisappear call `this.mapEventManager?.off('markerClick'); this.mapEventManager?.off('mapLoad'); this.mapController?.clear();` before nulling the dialog controller.

### `HW-09-0004` - The home page starts a continuous location subscription that is only cancelled from inside its own callbacks, so it keeps running in the background when no fix ever arrives.

- Category B, severity high, confidence confirmed
- Features: TOUR-01
- Document: `huawei_industry_tree/09_tourism/docs/01_practice-tourist-park-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1-0000001965211653
- Why: The official code-linter rule @performance/reasonable-gps-use-check states: "Apps without continuous tasks are not allowed to use location services when they are running in the background", and its Example of Incorrect Code is exactly this shape -- on('locationChange', ...) with no matching off(). If the device never obtains a fix (indoors, GNSS off, permission granted but hardware unavailable) the callback never runs, so off() is never reached and GNSS keeps draining battery after the user leaves the tab or backgrounds the app.
- Fix: Keep the locationChange handler in a field and add `aboutToDisappear(): void { geoLocationManager.off('locationChange', this.locationChange); }` to the Home component; additionally stop the subscription in the ability's onBackground as the linter rule's correct example does.

### `HW-09-0006` - PreferencesUtil.getPreferencesValue reads the stored value and then returns undefined, so the preferences read path never yields data.

- Category B, severity high, confidence confirmed
- Features: TOUR-01
- Document: `huawei_industry_tree/09_tourism/docs/01_practice-tourist-park-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1-0000001965211653
- Why: The method is declared as Promise<dataPreferences.ValueType | undefined> and the local `value` is computed and logged but discarded, so every caller receives undefined even when the key exists and even when defValue was supplied. The document lists this file in the code structure ("PreferencesUtil.ets // 工具类" / utility class), so it is presented as reusable framework code.
- Fix: Return the value: `return value;` on the success path, and return `defValue` (or rethrow) in the catch path instead of a bare return.

### `HW-09-0007` - TravelTips loads $rawfile('userIndex.html') from the ParkService HAR, which has no rawfile directory, so the travel-guide Web view renders blank.

- Category B, severity high, confidence confirmed
- Features: TOUR-01
- Document: `huawei_industry_tree/09_tourism/docs/01_practice-tourist-park-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1-0000001965211653
- Why: $rawfile resolves against the resources of the module that contains the calling code, not against another HAR. ParkService ships no rawfile folder, so the resource cannot be resolved and the 旅游攻略 (travel guide) page shows an empty Web component.
- Fix: Add features/ParkService/src/main/resources/rawfile/userIndex.html to the ParkService module (the home module's copy can be duplicated), or move the shared HTML into the common HAR and load it from there.

### `HW-09-0016` - The main page's Tabs returns false from onContentWillChange, which permanently blocks switching to the other three bottom tabs.

- Category C, severity high, confidence confirmed
- Features: TOUR-03
- Document: `huawei_industry_tree/09_tourism/docs/03_location_permission_prompt.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/location_permission_prompt-0000002235220070
- Why: The Tabs reference defines the return value of OnTabsContentWillChangeCallback as: "The return value true means that the tab can switch to the new page. The value false means that the tab cannot switch to the new page and will remain on the current page." A constant false therefore vetoes every switch -- tap, swipe, index change and controller changeIndex alike -- so three of the four tabs are unreachable and only the tab bar highlight moves.
- Fix: Remove the onContentWillChange call, or return true; use the callback only when a switch must genuinely be vetoed (an unsaved form, an unmet login), and return true otherwise.

### `HW-09-0017` - checkPermissions overwrites its own result on every loop pass, so the bubble is decided by the last permission alone and stays hidden when only the precise permission is missing.

- Category B, severity high, confidence confirmed
- Features: TOUR-03
- Document: `huawei_industry_tree/09_tourism/docs/03_location_permission_prompt.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/location_permission_prompt-0000002235220070
- Why: The document's stated behaviour is 检查是否开启定位权限，没有开启则左上角地址弹出气泡 (check whether the location permission is enabled; if it is not, a bubble pops up at the address in the top left corner). ohos.permission.LOCATION and ohos.permission.APPROXIMATELY_LOCATION are granted independently, so with APPROXIMATELY_LOCATION granted and LOCATION denied the loop ends on the granted permission, handlePopup is false and the user is never asked -- exactly the state the feature exists to catch. The loop bound 2 is also a hardcoded literal that silently ignores any third permission added to the array.
- Fix: Never clear the flag inside the loop -- accumulate instead: `let missing = false; for (const p of permissions) { if (await this.checkAccessToken(p) !== AbilityAccessCtrl.GrantStatus.PERMISSION_GRANTED) { missing = true; break; } } this.handlePopup = missing;` and iterate over permissions.length rather than the literal 2.

### `HW-09-0034` - The entry module declares no requestPermissions at all, so a sample whose only two features are a map and a POI search has no network permission.

- Category D, severity high, confidence confirmed
- Features: TOUR-06
- Document: `huawei_industry_tree/09_tourism/docs/06_obtain_destination_map.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/obtain_destination_map-0000002318736725
- Why: The Map Kit troubleshooting page Map Not Displayed names this as the first cause of a blank map: "Check whether the 'get network status error, code: 201, message:Permission denied' log entry exists. If the log exists, the app does not have the permission to obtain the network status. Add the permission to obtain the network status in the module.json5 file of your app", with a snippet declaring ohos.permission.INTERNET and ohos.permission.GET_NETWORK_INFO. Both are system_grant, so they cost the user nothing and there is no reason to omit them. Without them the map tiles never load and searchByText fails, which means neither of the two things this document teaches can work from a clean checkout.
- Fix: Add to entry/src/main/module.json5: requestPermissions with ohos.permission.INTERNET and ohos.permission.GET_NETWORK_INFO, each with usedScene.when set to always.

### `HW-09-0036` - site.searchByText is awaited with no try/catch in aboutToAppear, so a network or authentication failure rejects unhandled and leaves the nearby list silently empty.

- Category B, severity high, confidence confirmed
- Features: TOUR-06
- Document: `huawei_industry_tree/09_tourism/docs/06_obtain_destination_map.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/obtain_destination_map-0000002318736725
- Why: searchByText is a network call against a service that can fail for reasons this sample makes likely: no INTERNET permission is declared, and Map Kit app identity verification needs a client_id the sample does not carry. Both produce a rejected promise, which in an async aboutToAppear becomes an unhandled rejection rather than a caught error. Everything after the await is skipped, so this.metros stays empty, this.closestOne stays undefined, the address row renders the fallback single space at RentingPage.ets:202, and the map page it pushes receives an empty array -- with nothing logged to say why.
- Fix: Wrap the search in try/catch, log the BusinessError code and message, and render an explicit empty or retry state instead of a blank row.

### `HW-09-0037` - The map listeners are registered on a process-lifetime singleton and never unregistered, so every visit to the map page leaves another mapLoad and markerClick subscription behind.

- Category B, severity high, confidence confirmed
- Features: TOUR-06
- Document: `huawei_industry_tree/09_tourism/docs/06_obtain_destination_map.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/obtain_destination_map-0000002318736725
- Why: The official Map Kit FAQ shows the required teardown: "aboutToDisappear(): void { this.mapEventManager?.off('mapLoad'); this.mapController?.clear(); }". Here the field holding the event manager belongs to a singleton, so it is overwritten -- not released -- each time MapComponent re-initialises after the user leaves the map page and returns. The previous manager and its two closures stay alive for the life of the process along with the map controller they captured, and the singleton also keeps this.mapController pointing at a controller whose component has been destroyed, which is what moveToCenter and moveCamera would then drive.
- Fix: Give the view model a release() that calls off('markerClick') and off('mapLoad') on the event manager, then clear() on the controller, nulls both fields, and is called from MapPage.aboutToDisappear.

### `HW-09-0041` - The marker click listener is registered in the map init callback and never removed, and the page defines no aboutToDisappear at all.

- Category B, severity high, confidence confirmed
- Features: TOUR-07
- Document: `huawei_industry_tree/09_tourism/docs/07_map_marker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_marker-0000002332603737
- Why: The official Map Kit FAQ pairs the two: "aboutToDisappear(): void { this.mapEventManager?.off('mapLoad'); this.mapController?.clear(); }". The closure registered here captures this and writes this.clickedShop, so the destroyed page is kept alive by the subscription and a later marker tap writes state on a component that is gone. The document repeats the same unbalanced pattern in 实现思路 step 3, where on('markerClick') is shown with no matching off.
- Fix: Obtain the event manager once (this.mapEventManager = mapController.getEventManager()), register through it, and add aboutToDisappear that calls off('markerClick') and then clear() on the controller.

### `HW-09-0042` - The marker's zIndex is used as the shop identity, so the drawing order of the pins decides which shop card is shown.

- Category B, severity high, confidence confirmed
- Features: TOUR-07
- Document: `huawei_industry_tree/09_tourism/docs/07_map_marker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_marker-0000002332603737
- Why: zIndex is a rendering property -- it decides which overlay is painted on top when two pins collide -- not an identity. Overloading it means the two concerns cannot be tuned independently: raising a pin so it is not hidden behind another silently changes which shop the detail card and the enlarged info window show, and two pins that should legitimately share a z level cannot, because they would then resolve to the same shop. The filter in onValueChange has no fallback either, so any zIndex that does not match a SHOPS index leaves currentShop as undefined. The sample already carries the correct identifier in the marker title and simply ignores it.
- Fix: Read the identity from the title the sample already sets -- Number(marker.getTitle()) -- or better, keep a local array of the added markers and use indexOf, then free zIndex for its actual purpose. Guard the SHOPS lookup with a fallback for an unmatched index.

### `HW-09-0045` - The module declares no network permission and no client_id metadata, so the map this sample exists to demonstrate cannot load from a clean checkout.

- Category D, severity high, confidence confirmed
- Features: TOUR-07
- Document: `huawei_industry_tree/09_tourism/docs/07_map_marker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_marker-0000002332603737
- Why: Map Kit's own troubleshooting page names the missing network permission as the first cause of a blank map: "the app does not have the permission to obtain the network status. Add the permission to obtain the network status in the module.json5 file of your app", with ohos.permission.INTERNET and ohos.permission.GET_NETWORK_INFO. The Configuring the Client ID guide adds the second requirement: "In the module.json5 file of the entry module in the project, add the metadata section, in which you should set name to client_id". Without both, app identity verification fails and no tiles are fetched, so a reader who completes every step the document lists still sees an empty screen. Both permissions are system_grant and cost the user nothing.
- Fix: Add requestPermissions with ohos.permission.INTERNET and ohos.permission.GET_NETWORK_INFO to entry/src/main/module.json5, add a metadata entry named client_id with a placeholder value, and add a sixth setup step to 说明 pointing at that file.

### `HW-09-0047` - The publish button pops the page immediately and then runs the submit one second later on a destroyed component, from a timer nobody can cancel.

- Category B, severity high, confidence confirmed
- Features: TOUR-08
- Document: `huawei_industry_tree/09_tourism/docs/08_hotel_check_in_review.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/hotel_check_in_review-0000002364291253
- Why: pop() runs synchronously, so the NavDestination is gone roughly a second before the callback fires; the callback then dereferences this.promptAction and this.submit2Remote on a component that has been destroyed. The timer handle is not stored, so there is no clearTimeout in any teardown -- leaving the page, backgrounding the app or popping the whole stack all leave it armed. The ordering is also backwards for the flow the document describes: the order is marked 已评价 (reviewed) by the pop callback in Index.ets:138-141 before the simulated request has even started, so a failed submit could never be reflected.
- Fix: Submit first and pop from the result: make submitReview async, await it, show the toast, then pop -- keeping the button disabled while it runs. If a timer is genuinely wanted for the demo, store its id and clear it in aboutToDisappear.

### `HW-09-0052` - MediaPlayer.initialize assigns the newly created AVPlayer to its own parameter instead of the field, so the fallback branch leaves this.avPlayer null.

- Category B, severity high, confidence confirmed
- Features: TOUR-09
- Document: `huawei_industry_tree/09_tourism/docs/09_attraction_talk_sample.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/attraction_talk_sample-0000002335049696
- Why: The whole purpose of the fallback is to obtain a player when none was supplied, and it drops it: the handlers are wired up to an instance the singleton has no reference to, this.avPlayer stays null, and every subsequent play(), pause(), prepare() and reset() hits its own null guard and silently returns. The player itself is left alive, subscribed and unreachable. The first branch is the only one that works, so the bug is masked today purely because EntryAbility.ets:40 always passes a player in.
- Fix: Assign to the field: this.avPlayer = await media.createAVPlayer(); this.setupCallbacks(context, this.avPlayer); Better still, delete initialize entirely and keep only getAVPlayer, which already creates the player and wires it up exactly once.

### `HW-09-0053` - The ability wires the AVPlayer callbacks twice by passing getAVPlayer's already-configured player straight into initialize.

- Category B, severity high, confidence probable
- Features: TOUR-09
- Document: `huawei_industry_tree/09_tourism/docs/09_attraction_talk_sample.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/attraction_talk_sample-0000002335049696
- Why: The redundancy itself is certain from the code -- the argument expression configures the player and the call then configures it again -- and the initialize call has no other effect, since its parameter is never null on this path. If AVPlayer.on appends rather than replaces, every state transition is then handled twice, which in this handler is not idempotent: the initialized branch calls avPlayer.prepare() twice, completed and stopped and error each call avPlayer.reset() twice, and the idle branch assigns fdSrc twice, re-entering initialized. Marked probable because the AVPlayer reference page in this corpus is empty and does not state whether a repeated on replaces the previous callback.
- Fix: Call one or the other, not both: EntryAbility should do await instance.getAVPlayer(this.context) alone. Remove initialize, or make setupCallbacks idempotent by guarding on a flag and calling off before re-registering.

### `HW-09-0066` - An invalid ID card number turns the name field red, because the ID card input has no error flag of its own and reuses isNameCorrect.

- Category C, severity high, confidence confirmed
- Features: TOUR-11
- Document: `huawei_industry_tree/09_tourism/docs/11_check_information.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/check_information-0000002360422162
- Why: The page's whole purpose is to tell the user which field is wrong. With one flag serving two inputs, a bad ID card paints the name red as well, so the user is pointed at a field that is fine and that the app never validates at all. The line above it makes the coupling worse: setting isPhoneNoCorrect = true inside the ID card branch silently clears a phone error the user has not fixed yet, so the phone field goes black again while still holding a bad number.
- Fix: Add an isIdCardCorrect flag, bind the ID card input and its divider to it, and leave isNameCorrect for a real name check. Drop the isPhoneNoCorrect = true assignment from the ID card branch -- each validator should only write its own flag.

### `HW-09-0067` - The agreement checkbox is toggled twice by a single tap, because it two-way binds isAgree while its parent Row also flips isAgree on click.

- Category C, severity high, confidence confirmed
- Features: TOUR-11
- Document: `huawei_industry_tree/09_tourism/docs/11_check_information.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/check_information-0000002360422162
- Why: Click events "follow the touch event dispatch mechanism, supporting custom behaviors like event blocking and bubbling", so a tap on the Checkbox also reaches the Row's onClick. The two handlers then flip the same flag in the same gesture: the two-way binding sets it, the Row's handler sets it back. Tapping the checkbox itself therefore does nothing, while tapping the label works -- and the confirm button is gated on isAgree (line 193), so the user's most natural gesture leaves the form permanently unsubmittable.
- Fix: Pick one owner. Either drop the Row's onClick and let the Checkbox's own binding handle it, or keep the Row handler and stop the child from writing the flag -- give the Checkbox .select(this.isAgree) without $$ and .hitTestBehavior(HitTestMode.None) so only the Row responds.

### `HW-09-0072` - The module declares no network permission and no client_id metadata, so neither the map nor the walking-route call can work from a clean checkout.

- Category D, severity high, confidence confirmed
- Features: TOUR-12
- Document: `huawei_industry_tree/09_tourism/docs/12_add_and_collect_map_marker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_and_collect_map_marker-0000002402495269
- Why: Map Kit's own troubleshooting page names the missing network permission as the first cause of a blank map: "the app does not have the permission to obtain the network status. Add the permission to obtain the network status in the module.json5 file of your app", with ohos.permission.INTERNET and ohos.permission.GET_NETWORK_INFO. The Configuring the Client ID guide adds the second requirement, that the entry module carry a metadata entry named client_id. Without both, app identity verification fails, no tiles load, and the route request that fills in the walking time and distance fails too -- so a reader who completes every step the document lists still sees an empty screen. Both permissions are system_grant and cost the user nothing.
- Fix: Add ohos.permission.INTERNET and ohos.permission.GET_NETWORK_INFO to requestPermissions, add a module-level metadata entry named client_id with a placeholder value, and add a sixth setup step to 说明 pointing at entry/src/main/module.json5.

### `HW-09-0075` - Three map event listeners are registered in the init callback and none is ever released; the page defines no aboutToDisappear.

- Category B, severity high, confidence confirmed
- Features: TOUR-12
- Document: `huawei_industry_tree/09_tourism/docs/12_add_and_collect_map_marker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_and_collect_map_marker-0000002402495269
- Why: The official Map Kit FAQ pairs them: "aboutToDisappear(): void { this.mapEventManager?.off('mapLoad'); this.mapController?.clear(); }". Both retained callbacks here capture this and write @Provide state -- markerClickCallback sets this.currmarker and this.showAddressDialog, mapLongClickcallback replaces this.currmarker entirely -- so the destroyed page is kept alive by the subscriptions, and a long press delivered afterwards mutates provided state that no consumer is bound to any more.
- Fix: Hold mapLoadCallback in a field like the other two, and add aboutToDisappear calling off('mapLongClick'), off('markerClick'), off('mapLoad') and then clear() on the controller.

### `HW-09-0001` - The navigation snippet in the document calls getContext(this), which has been deprecated since API version 18, while the document targets API version 24.

- Category A, severity medium, confidence confirmed
- Features: TOUR-01
- Document: `huawei_industry_tree/09_tourism/docs/01_practice-tourist-park-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1-0000001965211653
- Why: The API reference states: "This API is supported since API version 9 and deprecated since API version 18. You are advised to use getHostContext instead." Publishing a deprecated call in a document that declares API 24 teaches the wrong pattern and produces a deprecation warning in DevEco Studio.
- Fix: Replace the document snippet with the form already used in the sample: `let context = this.getUIContext().getHostContext() as common.UIAbilityContext;`

### `HW-09-0003` - The map initialization callback handles only the success branch, so a map load failure is swallowed with no log and no user feedback.

- Category B, severity medium, confidence confirmed
- Features: TOUR-01
- Document: `huawei_industry_tree/09_tourism/docs/01_practice-tourist-park-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1-0000001965211653
- Why: The official Map Kit guide's initialization callback always handles the error branch: "} else { console.error(`Failed to initialize the map. Code: ${err.code}; message: ${err.message}`); }". The document itself lists the common causes of a blank map (no network, missing Client ID, unfinished AGC setup, app identity verification failure) -- exactly the errors delivered through this parameter and here discarded, leaving a developer with a blank screen and an empty log.
- Fix: Add the else branch and log err.code/err.message, and surface a retry or an error state in the UI.

### `HW-09-0005` - topRectHeight and bottomRectHeight are provided as number but consumed as string in ten components, so every @Provide/@Consume pair in the app is a type mismatch.

- Category C, severity medium, confidence confirmed
- Features: TOUR-01
- Document: `huawei_industry_tree/09_tourism/docs/01_practice-tourist-park-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1-0000001965211653
- Why: The @Provide/@Consume guide states: "@Provide and @Consume are bound using variable names or aliases. They must be of the same type. Otherwise, implicit type conversion occurs, causing application exceptions." The consumed value is a vp number at runtime but is typed string, so arithmetic on it silently concatenates and, from API version 20, a BuilderNode-crossing pair with mismatched types raises a runtime error.
- Fix: Change every consumer to `@Consume('topRectHeight') topRectHeight: number;` and `@Consume('bottomRectHeight') bottomRectHeight: number;` to match the provider.

### `HW-09-0008` - The entry ability is exported as an NFC tag and HCE handler for nine tag technologies although the project contains no NFC code at all.

- Category D, severity medium, confidence confirmed
- Features: TOUR-01
- Document: `huawei_industry_tree/09_tourism/docs/01_practice-tourist-park-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1-0000001965211653
- Why: Per the NFC Tag Read/Write guide these skills are what registers an app as a candidate handler for tag discovery, and the HCE guide does the same for card emulation. Declaring them widens the app's exported attack surface and makes the tourism app appear in the system's handler list on every scan of those tag types, where it can do nothing. The declaration is copy-paste residue from an NFC demo, not a feature of this sample.
- Fix: Delete the three NFC actions and the nine tag-tech uris from the EntryAbility skills block, leaving only entity.system.home / action.system.home.

### `HW-09-0009` - The document's permission block declares only ohos.permission.LOCATION, which is not enough for the precise positioning it promises and does not match the sample's module.json5.

- Category E, severity medium, confidence confirmed
- Features: TOUR-01
- Document: `huawei_industry_tree/09_tourism/docs/01_practice-tourist-park-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1-0000001965211653
- Why: The Map Kit guide is explicit: "Apply for the ohos.permission.LOCATION and ohos.permission.APPROXIMATELY_LOCATION permissions. You need to declare the required permissions in the module.json5 configuration file." A developer who copies only the document's block gets no location fix and no map tiles either, because INTERNET is missing as well.
- Fix: Extend the document snippet to the three entries the sample actually ships (ohos.permission.INTERNET, ohos.permission.APPROXIMATELY_LOCATION, ohos.permission.LOCATION), each with its reason and usedScene, and state that LOCATION must be requested together with APPROXIMATELY_LOCATION.

### `HW-09-0013` - EntryAbility subscribes to avoidAreaChange in onWindowStageCreate and never unsubscribes in onWindowStageDestroy.

- Category B, severity medium, confidence confirmed
- Features: TOUR-01
- Document: `huawei_industry_tree/09_tourism/docs/01_practice-tourist-park-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1-0000001965211653
- Why: A window listener registered on the main window outlives the window stage unless it is removed with off('avoidAreaChange'). The callback writes into AppStorage, so it keeps firing against a stage that no longer exists after the ability's window stage is torn down and recreated, and each recreation adds another listener.
- Fix: Keep the window handle in a field and add `this.windowClass?.off('avoidAreaChange')` to onWindowStageDestroy, which is where the file's own comment already says UI resources should be released.

### `HW-09-0014` - getRoutes indexes result.routes[0] without checking the result and appends to the routes and points arrays without clearing them, so a second route request duplicates the whole path.

- Category B, severity medium, confidence confirmed
- Features: TOUR-01
- Document: `huawei_industry_tree/09_tourism/docs/01_practice-tourist-park-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1-0000001965211653
- Why: getDrivingRoutes can legitimately return zero routes (no drivable path, out-of-service region), and result.routes[0].steps then throws a TypeError that the surrounding catch converts into a silent log with no route drawn. The unbounded accumulation is the second half of the same defect: the 路线管理 button that calls getRoutes is tappable repeatedly, so on the second tap this.points already holds the previous path and the polyline is built from both routes concatenated -- and because @State points is what the overviewPolyline fallback reads, the drawn line grows every time.
- Fix: Guard the result (`if (!result.routes || result.routes.length === 0) { return; }`) and reset the accumulators at the top of getRoutes (`this.routes = []; this.points = [];`). Drop the pointless `async` on the forEach callback, which returns a promise nobody awaits.

### `HW-09-0015` - BreakPointType.getValue is stubbed to return an empty string and is still exported from the common HAR as public API.

- Category B, severity medium, confidence confirmed
- Features: TOUR-01
- Document: `huawei_industry_tree/09_tourism/docs/01_practice-tourist-park-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1-0000001965211653
- Why: The document presents the common HAR as the 公共能力层 (common capability layer) that upper-layer business components reference, so anything Index.ets exports is API a developer is meant to consume. A responsive helper that discards its options and returns '' for every breakpoint fails silently: numeric consumers get '' where a vp value was expected, and the layout collapses instead of erroring.
- Fix: Implement getValue against the current breakpoint held in AppStorage -- read BreakpointConstants.CURRENT_BREAKPOINT, index this.options with it and fall back to the sm option -- or drop BreakPointType from common/Index.ets until it works.

### `HW-09-0018` - The permissions section swaps the two location permissions: it labels LOCATION as the approximate one and APPROXIMATELY_LOCATION as the precise one.

- Category E, severity medium, confidence confirmed
- Features: TOUR-03
- Document: `huawei_industry_tree/09_tourism/docs/03_location_permission_prompt.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/location_permission_prompt-0000002235220070
- Why: The names are self-describing and the mapping is fixed: APPROXIMATELY_LOCATION is the approximate permission and LOCATION is the precise one. A developer who trusts this section will request the wrong permission for the accuracy their feature needs, and requesting LOCATION without APPROXIMATELY_LOCATION is refused outright, which is the failure this whole sample exists to prevent. TOUR-01's document states the correct mapping for the same pair of permissions, so the corpus contradicts itself.
- Fix: Swap the two lines: 获取精确位置权限 is ohos.permission.LOCATION, 获取模糊位置权限 is ohos.permission.APPROXIMATELY_LOCATION, and add the note that LOCATION must always be requested together with APPROXIMATELY_LOCATION.

### `HW-09-0019` - The bubble is never re-evaluated after the user grants the permission, so it stays on screen although the document promises it disappears.

- Category B, severity medium, confidence confirmed
- Features: TOUR-03
- Document: `huawei_industry_tree/09_tourism/docs/03_location_permission_prompt.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/location_permission_prompt-0000002235220070
- Why: handlePopup is the boolean bound to bindPopup, so once the grant succeeds the bubble keeps offering 去打开 (go and enable) for a permission that is already enabled. The promised behaviour only takes effect on the next launch of the page, because that is the only time aboutToAppear runs again.
- Fix: Re-check after the request resolves: `await atManager.requestPermissionsFromUser(context, permissions); await this.checkPermissions();` -- checkPermissions already writes handlePopup, so a single call closes the bubble on success and leaves it open on refusal.

### `HW-09-0020` - checkAccessToken asserts non-null on two variables that stay unassigned when their try blocks throw, so a failure returns undefined typed as GrantStatus.

- Category B, severity medium, confidence confirmed
- Features: TOUR-03
- Document: `huawei_industry_tree/09_tourism/docs/03_location_permission_prompt.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/location_permission_prompt-0000002235220070
- Why: If getBundleInfoForSelf rejects, tokenID is never assigned and `tokenID!` passes undefined into checkAccessToken; that call then throws, is swallowed, and `grantStatus!` returns undefined to a caller whose type signature promises a GrantStatus. The two `!` assertions are what let this compile -- they suppress the exact check that would have caught it. The caller compares with !== PERMISSION_GRANTED, so the undefined happens to fall through as denied, but any caller that switches on the enum gets an unhandled value.
- Fix: Initialise both to a safe default (`let grantStatus: AbilityAccessCtrl.GrantStatus = AbilityAccessCtrl.GrantStatus.PERMISSION_DENIED; let tokenID: number = 0;`), return early from the first catch, and drop both non-null assertions.

### `HW-09-0021` - EntryAbility subscribes to avoidAreaChange without ever unsubscribing and fires setWindowLayoutFullScreen without handling its promise.

- Category B, severity medium, confidence confirmed
- Features: TOUR-03
- Document: `huawei_industry_tree/09_tourism/docs/03_location_permission_prompt.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/location_permission_prompt-0000002235220070
- Why: A window listener registered on the main window survives the window stage unless removed with off('avoidAreaChange'), so each stage recreation stacks another listener writing into AppStorage. The unhandled promise is the second half: if the window manager rejects setWindowLayoutFullScreen the app silently keeps a non-full-screen layout while every page still pads itself by the avoid areas, leaving a double inset with no log to explain it.
- Fix: Keep the window in a field, add `this.windowClass?.off('avoidAreaChange')` to onWindowStageDestroy, and attach a `.catch((err: BusinessError) => hilog.error(...))` to setWindowLayoutFullScreen.

### `HW-09-0022` - calculateDays computes the night count from month and day only, forcing both ends into the current year even though the page already tracks startYear and endYear.

- Category B, severity medium, confidence confirmed
- Features: TOUR-04
- Document: `huawei_industry_tree/09_tourism/docs/04_data_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_selection-0000002240198078
- Why: The year is what decides February's length, so any range that crosses 29 February in a leap year is counted in the wrong year and comes out one night short: 2028-02-28 to 2028-03-01 is two nights, but evaluated against 2026 it is one. The cross-year branch compounds this -- it adds exactly one year to a date already anchored to the wrong year, so it cannot recover the real span. The night count is what the confirm button shows and what a booking would be priced on.
- Fix: Take the years as parameters: `calculateDays(startYear, startMonth, startDay, endYear, endMonth, endDay)`, build both Date objects from them, and delete the cross-year heuristic -- with real years the plain difference is already correct.

### `HW-09-0023` - The range endpoints are matched on day and month only while the days between them are matched on year, month and day, so the two halves of the same highlight use different keys.

- Category C, severity medium, confidence confirmed
- Features: TOUR-04
- Document: `huawei_industry_tree/09_tourism/docs/04_data_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_selection-0000002240198078
- Why: monthItem.year is available at both sites and the click handler stores startYear / endYear, so the omission is not a data problem. It is invisible today only because the calendar renders exactly two consecutive months, which can never share a month number; the moment a developer extends the LazyForEach to a full year of Month entries -- the obvious first customisation of a hotel date picker -- December 2026 and December 2027 both match monthItem.month === 12 and the check-in badge is painted on both.
- Fix: Add the year to the endpoint comparisons, matching the middle-range check: `day === this.startDay && monthItem.month === this.startMonth && monthItem.year === this.startYear` in both the font-colour ternary and the badge condition, and likewise for the end date.

### `HW-09-0024` - MonthDataSource.pushData accepts an array but emits a single onDataAdd, so appending N months tells LazyForEach about one.

- Category C, severity medium, confidence confirmed
- Features: TOUR-04
- Document: `huawei_industry_tree/09_tourism/docs/04_data_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_selection-0000002240198078
- Why: LazyForEach maintains its own view of the item count from the notifications it receives, so pushing N items and notifying one add leaves it N-1 items behind totalCount(). The current sample survives this because the only pushData call happens in aboutToAppear, before any LazyForEach has registered a listener, so the notification is dropped and the first render reads totalCount() directly -- but MonthDataSource is written as a reusable data source, and loading further months on scroll, the natural next step for a calendar, breaks immediately.
- Fix: Notify once per appended element, or use the batch API: `const start = this.dataArray.length; this.dataArray.push(...data); data.forEach((_, i) => this.notifyDataAdd(start + i));`

### `HW-09-0028` - CalendarPickerDialog.show is called as a bare global API instead of inside UIContext.runScopedTask, which the UIContext guide requires because this dialog has no UIContext substitute.

- Category A, severity medium, confidence confirmed
- Features: TOUR-05
- Document: `huawei_industry_tree/09_tourism/docs/05_address_exchange.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/address_exchange-0000002283895117
- Why: The UIContext guide lists CalendarPickerDialog in its global-API replacement table with the substitute column reading "Not supported", and prescribes the workaround: "If the UIContext object does not provide an alternative API (for example, CalendarPickerDialog) ... you can use the runScopedTask method of the UIContext object to execute the closure." Its recommended example wraps exactly this call. The CalendarPickerDialog reference adds that "the APIs of this module cannot be used where the UI context is ambiguous". A bare call resolves the UI instance from the call chain, so in a multi-window or multi-instance app the picker can open on the wrong instance or fail to open at all.
- Fix: Wrap the call: `const uiContext = this.getUIContext(); uiContext.runScopedTask(() => { CalendarPickerDialog.show({ selected: this.selectedDate, onAccept: (value) => { ... } }); });`

### `HW-09-0029` - The swap is purely visual - the two city values are never exchanged - so after a swap every reader of cityNameStart still gets the original origin.

- Category B, severity medium, confidence confirmed
- Features: TOUR-05
- Document: `huawei_industry_tree/09_tourism/docs/05_address_exchange.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/address_exchange-0000002283895117
- Why: After a swap, screen position and state disagree: the label reading 出发地 (departure) sits above the text held in cityNameEnd. Any code that submits the search, records the pair, or navigates on reads the pre-swap values and silently books the reverse trip. The inverted assignment in the history handler is the tell -- every future consumer of these two fields will need the same `if (this.swap)` correction, and the first one that forgets it is a wrong-direction booking. The document's own framing, 出发地与目的地地址左右交换 (swapping the departure and destination addresses left and right), describes a data swap, not a slide.
- Fix: Exchange the values inside the animation callback and reset the offset, so state and view agree: `const from = this.cityNameStart; this.cityNameStart = this.cityNameEnd; this.cityNameEnd = from;` with translateX animated out and back to zero. Then drop this.swap and the crossed assignment in the history handler.

### `HW-09-0030` - The document animates the rotation inside animateTo, but the sample changes the angle outside it and drives the icon with the component animation attribute instead.

- Category E, severity medium, confidence confirmed
- Features: TOUR-05
- Document: `huawei_industry_tree/09_tourism/docs/05_address_exchange.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/address_exchange-0000002283895117
- Why: These are two different animation mechanisms with different behaviour: an explicit animateTo closure drives the rotation on the springMotion curve together with the slide, while the attribute animation drives it independently on an EaseOut curve. The document's snippet is also missing the else branch, so a reader who copies it gets an offset that only ever grows and never returns -- one press slides the addresses and every later press does nothing.
- Fix: Replace the 实现思路 snippet with the shipped handler, including the if/else on this.swap, and describe the attribute animation on the rotate as the mechanism the sample actually uses for the icon.

### `HW-09-0035` - The setup note lists only the AGC and signing steps and never mentions the client_id metadata that app identity verification needs, which the sample also omits.

- Category E, severity medium, confidence confirmed
- Features: TOUR-06
- Document: `huawei_industry_tree/09_tourism/docs/06_obtain_destination_map.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/obtain_destination_map-0000002318736725
- Why: The Configuring the Client ID guide states the requirement plainly: "In the module.json5 file of the entry module in the project, add the metadata section, in which you should set name to client_id and value to the client ID obtained in the previous step." Without it the Map Kit app identity verification fails with error 1002600003 and the map stays blank -- the symptom the sibling document TOUR-01 warns about but this one does not. A reader who follows all four listed steps still gets an empty screen and has nothing in the document pointing at the cause.
- Fix: Add a fifth step to 说明: copy the Client ID from AGC into a metadata entry named client_id in entry/src/main/module.json5, and ship that block, with a placeholder value, in the sample.

### `HW-09-0038` - setMarkers empties the marker array without removing the markers from the map and re-adds the destination marker, so a second call duplicates every pin.

- Category B, severity medium, confidence confirmed
- Features: TOUR-06
- Document: `huawei_industry_tree/09_tourism/docs/06_obtain_destination_map.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/obtain_destination_map-0000002318736725
- Why: map.Marker inherits remove() from BaseOverlay, documented as "Removes an overlay from a map" with the example marker.remove(). Truncating the array with searchMarkers.length = 0 drops the only handles to the markers already on the map, so those markers stay drawn and can never be removed afterwards. Since setMarkers is the mapLoad callback on a singleton whose listeners are never released, a second mapLoad -- returning to the map page, a style reload -- paints a second full set of subway pins plus a second destination pin over the first, and markerClick then fires on whichever of the stacked pins is hit. The destination marker created in init() is additionally unreachable from the start, because its promise result is discarded.
- Fix: Remove before clearing: iterate searchMarkers calling remove() on each, then truncate. In init(), keep the destination marker in a field, remove the previous one, and await the addMarker call from setMarkers.

### `HW-09-0039` - The map initialization callback handles only the success branch, in both the sample and the document's own snippet.

- Category B, severity medium, confidence confirmed
- Features: TOUR-06
- Document: `huawei_industry_tree/09_tourism/docs/06_obtain_destination_map.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/obtain_destination_map-0000002318736725
- Why: The official map-presenting guide always pairs the branch: "} else { console.error(`Failed to initialize the map. Code: ${err.code}; message: ${err.message}`); }". This sample makes the error branch matter more than usual, because two configuration gaps it ships with -- no INTERNET permission and no client_id metadata -- both surface as an err on exactly this callback. Discarding it leaves a developer with a blank map, an empty log, and no way to tell an authentication failure from a network one.
- Fix: Add the else branch and log err.code and err.message, in the document snippet as well as in MarkerViewModel.

### `HW-09-0040` - SearchNearby overwrites the shared markerClickCallback field after the fact, so the list highlight works only because the map happens to register that field later than the list mounts.

- Category B, severity medium, confidence confirmed
- Features: TOUR-06
- Document: `huawei_industry_tree/09_tourism/docs/06_obtain_destination_map.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/obtain_destination_map-0000002318736725
- Why: on() captures the function reference at registration time, so reassigning the field afterwards has no effect on an existing subscription. The sample works only because MapComponent's mapCallback is asynchronous and therefore runs after SearchNearby.aboutToAppear -- a mount-ordering coincidence, not a contract. Make the list conditional, mount it after the map has loaded, or return to the page (where the singleton still holds the leaked subscription from the previous visit, bound to a destroyed SearchNearby), and taps on a pin stop highlighting the list row. The reassignment is also invisible coupling: nothing in MarkerViewModel says a component may replace its callbacks.
- Fix: Do not swap the callback. Have MarkerViewModel expose the tapped marker as observable state - a selectedIndex that markerClickCallback sets - and let SearchNearby read it through @Link or an @ObjectLink on the observed view model.

### `HW-09-0043` - The annotation options are written as raw integers instead of the mapCommon enums, and fontStyle 1 selects BOLD where the sample intends plain text.

- Category A, severity medium, confidence confirmed
- Features: TOUR-07
- Document: `huawei_industry_tree/09_tourism/docs/07_map_marker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_marker-0000002332603737
- Why: Both fields are declared as enum types, so a bare 2 and a bare 1 are readable only against the reference table -- and the reader of this document, whose whole subject is the name label under a marker, is given no way to tell that 2 means BOTTOM. The fontStyle literal is the concrete cost: 1 is BOLD, not REGULAR, so all three shop labels render bold although nothing in the sample or the document asks for that; a developer who reads 1 as 'plain, the first option' gets the wrong weight.
- Fix: Use the enums: annotationPosition: mapCommon.TextPosition.BOTTOM, and fontStyle: mapCommon.FontStyle.REGULAR (or BOLD, stated deliberately).

### `HW-09-0044` - The marker labels - the subject of this document - are hardcoded Chinese literals, while the same three shop names already exist as string resources used by the detail card.

- Category B, severity medium, confidence confirmed
- Features: TOUR-07
- Document: `huawei_industry_tree/09_tourism/docs/07_map_marker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_marker-0000002332603737
- Why: The document's title is 地图指定位置名称标记 (name label marker at a specified map position), so the label text is the feature, and it is the one string in the sample that cannot be translated. The result is a split source of truth: rename a shop in string.json and the detail card updates while its map pin still shows the old name. The project ships en_US and zh_CN resource directories, so the mechanism is present and only these three strings bypass it.
- Fix: Resolve the names from the same resources the model uses -- read them with getUIContext().getHostContext().resourceManager.getStringSync($r('app.string.shop1').id) before building MarkerOptions -- or drive the whole marker set from the SHOPS array so pin and card cannot diverge.

### `HW-09-0046` - This document registers map listeners on the MapComponentController while the sibling tourism document registers them on the MapEventManager, which is the form the official guide teaches.

- Category F, severity medium, confidence confirmed
- Features: TOUR-07
- Document: `huawei_industry_tree/09_tourism/docs/07_map_marker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_marker-0000002332603737
- Why: The official Event Interaction guide states one form: "The following table describes APIs for map event listening, which are mainly provided by MapEventManager. You can obtain MapEventManager using the getEventManager method", and every example in it, in map-presenting and in map-faq-4 goes through the manager. Two neighbouring documents in the same industry teaching two different subscription surfaces for the same event leaves a reader unable to tell which is current, and it matters practically: off() is documented on the event manager, so the controller-based form has no obvious counterpart and is the shape that leaks in this sample.
- Fix: Align this document and its sample with the guide and with 06_obtain_destination_map.md: obtain MapEventManager via getEventManager(), register with on there, and release with off in aboutToDisappear.

### `HW-09-0048` - The quick-description chips mutate a property of a plain interface inside an @Local array, which V2 cannot observe, so the file compensates by copying the whole array on every toggle.

- Category C, severity medium, confidence confirmed
- Features: TOUR-08
- Document: `huawei_industry_tree/09_tourism/docs/08_hotel_check_in_review.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/hotel_check_in_review-0000002364291253
- Why: The @Local guide states the rule: "When @Local is used to decorate a nested class or object array, changes of lower-level object properties cannot be observed. Observation of these lower-level object properties requires use of @ObservedV2 and @Trace decorators." The shallow copy is a workaround for that gap, and it costs correctness as well as work: the ForEach over quickItems has no key generator, so replacing the array rebuilds all four chips on every tap, and the copy is shallow, so the new array still holds the same mutated objects and only happens to re-render because the array identity changed. Two state models for two identical problems in one file is also what makes the page hard to extend.
- Fix: Make QuickDescItem an @ObservedV2 class with @Trace isSelected, exactly as ReviewScore is, and delete the array copy from the toggle handler.

### `HW-09-0049` - Every user-visible string on both pages is a hardcoded Chinese literal and the module ships no locale resource directory, although colours and floats are properly resourced.

- Category B, severity medium, confidence confirmed
- Features: TOUR-08
- Document: `huawei_industry_tree/09_tourism/docs/08_hotel_check_in_review.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/hotel_check_in_review-0000002364291253
- Why: The sample demonstrates the mechanism for colours and dimensions and then bypasses it for all text, so the pages cannot be translated at all. It also splits the model: the rating words and chip labels are UI content that belongs beside the data they annotate, and burying them in the component makes the review form impossible to reconfigure without editing ArkTS. The neighbouring tourism sample AddressExchange.zip resources everything user-visible and ships en_US and zh_CN directories, so the corpus is inconsistent on this point.
- Fix: Move the text into resources/base/element/string.json, reference it with $r, and add en_US and zh_CN directories. Keep the chip labels and score descriptions as string-array resources so the sets can be changed without touching the page.

### `HW-09-0054` - The AVPlayer error callback takes the BusinessError and discards it, so every playback failure is a silent reset.

- Category B, severity medium, confidence confirmed
- Features: TOUR-09
- Document: `huawei_industry_tree/09_tourism/docs/09_attraction_talk_sample.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/attraction_talk_sample-0000002335049696
- Why: reset() drives the player back to idle, whose branch re-assigns fdSrc from MediaPlayer.audioUrl and starts the whole cycle again. A permanently bad asset -- a missing rawfile, an unsupported codec, a truncated mp3 -- therefore loops error, reset, idle, initialized, prepared, error indefinitely, with the only trace in the log being the word 'error'. The BusinessError in hand carries the code that distinguishes those cases and is thrown away.
- Fix: Log the error before resetting -- Logger.error with err.code and err.message -- and stop the retry cycle: clear MediaPlayer.audioUrl so the idle branch does not immediately re-arm the same failing source, and surface the failure to the page.

### `HW-09-0055` - The detail page polls the player's static fields on a 500 ms interval although the AVPlayer already pushes timeUpdate, durationUpdate and stateChange events.

- Category C, severity medium, confidence confirmed
- Features: TOUR-09
- Document: `huawei_industry_tree/09_tourism/docs/09_attraction_talk_sample.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/attraction_talk_sample-0000002335049696
- Why: The data is pushed and then polled, so the page pays a wakeup every 500 ms for the life of the screen while showing a progress readout that can lag half a second behind the audio and a play/pause icon that can lag half a second behind the user's own tap. The fields being polled are plain public statics on MediaPlayer, not observable state, which is the reason the timer exists at all -- and it is also why the timer must be created and destroyed in step with the page, adding the leak surface that the -1 sentinel guard then gets wrong.
- Fix: Make the shared values observable instead of static: hold them on an @ObservedV2 MediaPlayer with @Trace on state, time and duration, or publish them through AppStorage with AppStorage.setOrCreate from the existing timeUpdate and stateChange handlers, and bind the page with @StorageProp. Then delete the interval.

### `HW-09-0056` - An audio-guide player registers no audioInterrupt handler, so a call or another app's audio leaves the commentary in an inconsistent state.

- Category B, severity medium, confidence confirmed
- Features: TOUR-09
- Document: `huawei_industry_tree/09_tourism/docs/09_attraction_talk_sample.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/attraction_talk_sample-0000002335049696
- Why: This is a spoken-commentary player that a user runs while walking around a site, so interruptions are the normal case rather than the edge case: an incoming call, a navigation prompt, another media app. Without an audioInterrupt handler the system pauses or ducks the stream without the app knowing, so MediaPlayer.state and MediaPlayer.isPlaying keep saying playing, the page's play/pause icon keeps showing pause, and the 500 ms poll faithfully reports a position that is no longer advancing. Nothing resumes the commentary afterwards either.
- Fix: Register avPlayer.on('audioInterrupt', ...) in setupCallbacks, update MediaPlayer.state and isPlaying from the InterruptEvent, and resume on the matching resume hint. Set avPlayer.audioRendererInfo while the player is in the initialized state, before the first prepare(), choosing the stream usage that suits spoken commentary.

### `HW-09-0058` - VoicePlayerComponent defines onPageShow and onPageHide, which never run because it is a plain @Component and not an @Entry page.

- Category C, severity medium, confidence confirmed
- Features: TOUR-09
- Document: `huawei_industry_tree/09_tourism/docs/09_attraction_talk_sample.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/attraction_talk_sample-0000002335049696
- Why: The custom component lifecycle reference is explicit about the scope of both callbacks: onPageShow is "Invoked each time a router-managed page (a custom component decorated with @Entry) is displayed", and onPageHide the same for hiding. On a non-@Entry component neither is ever called, so the two methods are dead code that reads as working intent -- the clearing of the highlighted-track id when the page is shown or backgrounded simply never happens, and a developer maintaining the file has no signal that it does not.
- Fix: Delete both methods from the component. If the highlight really must be cleared when the page appears or the app backgrounds, put the callbacks on the @Entry page that hosts the list, or drive it from the ability's onForeground and onBackground, which the sample already uses for the isForeGround flag.

### `HW-09-0059` - Each list item resets the shared singleton player in its own aboutToAppear and aboutToDisappear, so creating or destroying any one track row tears down playback for all of them.

- Category B, severity medium, confidence confirmed
- Features: TOUR-09
- Document: `huawei_industry_tree/09_tourism/docs/09_attraction_talk_sample.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/attraction_talk_sample-0000002335049696
- Why: Per-item components own no app-wide resource, but these two hooks reach into the global player: every row that mounts blanks MediaPlayer.audioUrl, which the stateChange idle branch reads to re-arm the source, and every row that unmounts calls reset() on the live player. Whenever the list is rebuilt or the album page is left, the last row to be destroyed stops audio that a different screen may be playing, and the id driving the highlight is cleared underneath it. It is also order-dependent: with N rows the reset runs N times.
- Fix: Move ownership to the screen. Let the page that owns the list decide whether playback should stop, and leave the row responsible only for its own view state. If a row must signal a selection, have it write its own id and read the shared state, never write or reset it.

### `HW-09-0060` - OrderStatus is a string enum whose members are Chinese display labels, and the card renders the enum value directly, so order state and its user-visible text are the same string.

- Category B, severity medium, confidence confirmed
- Features: TOUR-10
- Document: `huawei_industry_tree/09_tourism/docs/10_travel_checkin_order.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/travel_checkin_order-0000002386506221
- Why: One string is doing two jobs. As a display label it cannot be localised: the module has en_US and zh_CN directories and every other piece of visible text uses $r, but the status chip is Chinese in both. As a state token it makes the twelve comparisons a string match against text a translator is free to change, so localising the label silently breaks the filtering and the button rules that drive the whole screen.
- Fix: Make OrderStatus a plain enum of state tokens and map to text at the point of display: a statusLabel(status): Resource helper returning $r('app.string.order_status_closed') and so on, with the strings in base, en_US and zh_CN. Move the hotel and room names into resources the same way.

### `HW-09-0061` - filterOrder aliases the shared master list into the view's own field on one branch and calls clear() on that same field on the other, so one changed tab type would wipe every order in the app.

- Category C, severity medium, confidence confirmed
- Features: TOUR-10
- Document: `huawei_industry_tree/09_tourism/docs/10_travel_checkin_order.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/travel_checkin_order-0000002386506221
- Why: After the ALL branch runs once, this.orderList and ORDER_SAMPLES are the same object, and the else branch six lines below calls clear() on it. Today the two branches can never both run in one instance, because orderTabType is assigned once when the TabContent builds it and never changes -- so the defect is latent rather than live. But filterOrder is called from three places (aboutToAppear, orderChange, tabChange), the ALL view re-runs it on every add and delete, and any change that lets a view see a second tab type -- reusing one OrderView across tabs, a filter chip, a search box -- empties the application's only data store on the first non-ALL filter. Nothing in the method signals that this.orderList is sometimes borrowed and sometimes owned.
- Fix: Never alias: copy on the ALL branch as well. Replace the assignment with a rebuild -- this.orderList.clear(); ORDER_SAMPLES.forEach((order) => this.orderList.add(order)); -- or drop the intermediate ArrayList entirely and build this.orderArray directly with a single filter over ORDER_SAMPLES.

### `HW-09-0062` - The stay dates are formatted with a hardcoded zh-CN locale, so the date order and separators ignore the device setting.

- Category B, severity medium, confidence confirmed
- Features: TOUR-10
- Document: `huawei_industry_tree/09_tourism/docs/10_travel_checkin_order.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/travel_checkin_order-0000002386506221
- Why: Intl.DateTimeFormat exists precisely to render a date the way the reader's locale expects; pinning the tag to 'zh-CN' throws that away and produces 2025/05/30 for a user whose system is set to English, where 05/30/2025 is expected. Every other locale-dependent value on the screen already follows the system, so the check-in and check-out dates are the one element that does not. The formatter is also built once per OrderContent instance, so a list of N orders constructs N identical formatters.
- Fix: Omit the locale argument so the system locale is used -- new Intl.DateTimeFormat(undefined, { year: 'numeric', month: '2-digit', day: '2-digit' }) -- or take it from i18n.System.getSystemLocale(). Hoist the formatter to module scope so the list shares one instance.

### `HW-09-0065` - EntryAbility subscribes to avoidAreaChange in onWindowStageCreate and never unsubscribes in onWindowStageDestroy.

- Category B, severity medium, confidence confirmed
- Features: TOUR-10
- Document: `huawei_industry_tree/09_tourism/docs/10_travel_checkin_order.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/travel_checkin_order-0000002386506221
- Why: A listener registered on the main window survives the window stage unless it is removed with off, so each recreation of the stage adds another callback writing into AppStorage, and the callbacks from destroyed stages keep running. This is the same defect the other tourism samples carry, so it is boilerplate propagating across the corpus rather than a one-off.
- Fix: Keep the window in a field and call off('avoidAreaChange') in onWindowStageDestroy, which is where the generated comment already says UI resources should be released.

### `HW-09-0068` - The phone-number regex printed in the document drops the end anchor that the shipped code has, so the documented version accepts any trailing text.

- Category E, severity medium, confidence confirmed
- Features: TOUR-11
- Document: `huawei_industry_tree/09_tourism/docs/11_check_information.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/check_information-0000002360422162
- Why: Without the trailing $ the pattern only requires a valid number at the start of the string, so RegExp.test returns true for 13800138000abc, for a 20-digit run of numbers, and for a pasted line containing a phone number anywhere after position zero. That is precisely the class of input a validation routine exists to reject, and the document is the copy a reader takes: the code block is the whole of step 2 and is presented as the function to use.
- Fix: Add the missing $ to the document's regex so it matches the shipped CheckMethods.ets exactly.

### `HW-09-0069` - isValidDate parses the birth date as UTC and then reads its components in local time, so west of Greenwich every valid ID card is rejected.

- Category B, severity medium, confidence confirmed
- Features: TOUR-11
- Document: `huawei_industry_tree/09_tourism/docs/11_check_information.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/check_information-0000002360422162
- Why: A date-only string in YYYY-MM-DD form is parsed as UTC midnight, while getFullYear, getMonth and getDate return the local calendar date. In any timezone behind UTC the local date is the previous day, so the round-trip comparison this function is built on fails for every input: 1990-01-01 comes back as 1989-12-31 and checkIDCardNo returns false before the checksum is even computed. The routine works only because the sample is exercised in UTC+8; the same code in a device set to any American timezone rejects every genuine ID card.
- Fix: Parse the parts explicitly and build a local date: split the string first, then new Date(Number(y), Number(m) - 1, Number(d)), and compare the components back against the parsed numbers. That is timezone-independent and also rejects overflow dates such as 1990-02-31, which is what the round-trip check is for.

### `HW-09-0073` - Both location permissions give their reason as $string:EntryAbility_desc, whose value is the literal word description, so the system dialog would justify the request with placeholder text.

- Category D, severity medium, confidence confirmed
- Features: TOUR-12
- Document: `huawei_industry_tree/09_tourism/docs/12_add_and_collect_map_marker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_and_collect_map_marker-0000002402495269
- Why: reason is mandatory for a user_grant permission precisely because it is the sentence the user reads before deciding, and the Declaring Permissions guide requires it for exactly these two permissions: "ohos.permission.APPROXIMATELY_LOCATION and ohos.permission.LOCATION are user_grant permissions, and reason and usedScene are mandatory". Pointing it at the ability description satisfies the schema and defeats the purpose: the prompt reads description, which tells the user nothing and would not survive store review. Reusing one string for two different permissions also means the approximate and precise requests can never be explained differently.
- Fix: Add two purpose-specific strings to resources/base/element/string.json -- one explaining that the precise location is used to measure the walking distance to a marked address, one for the approximate location -- and point each permission's reason at its own.

### `HW-09-0074` - Both location permissions are declared but never requested at runtime, and the my-location the walking route measures from is a hardcoded constant.

- Category D, severity medium, confidence confirmed
- Features: TOUR-12
- Document: `huawei_industry_tree/09_tourism/docs/12_add_and_collect_map_marker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_and_collect_map_marker-0000002402495269
- Why: A user_grant permission that is declared but never requested is never granted, so it buys nothing and only widens what the app appears to want; the Declaring Permissions guide's rule is to request the minimum the app actually uses. Here the gap is also a functional one that the document hides: the screen shows a walking time and distance to the tapped address as though from the user's position, and it is really from a fixed point near Nanjing. A developer adapting this sample will not find the permission request they need, because the sample never wrote one.
- Fix: Either use the permissions -- request them with requestPermissionsFromUser as TOUR-03 does, take a fix with geoLocationManager, and pass it as the route origin -- or remove both declarations and state in the document that the origin is a fixed demo coordinate.

### `HW-09-0076` - Both reverse-geocode callbacks dereference data[0] after testing only that data exists, so an empty result array throws inside the callback.

- Category B, severity medium, confidence confirmed
- Features: TOUR-12
- Document: `huawei_industry_tree/09_tourism/docs/12_add_and_collect_map_marker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_and_collect_map_marker-0000002402495269
- Why: getAddressesFromLocation returns an array, and an empty array is truthy: a long press in the sea, in a desert, or anywhere the geocoder has no entry returns [] rather than an error, and data[0].placeName then throws on undefined. The throw happens inside an asynchronous callback, so the surrounding try/catch at lines 94 and 107 -- which only guards the synchronous call -- does not catch it. The four fields concatenated for simpleAddress are each optional in the reverse-geocode result, so even a non-empty result can render the string undefinedundefined... when a level of the administrative hierarchy is missing.
- Fix: Guard on length and return on error: if (err) { hilog.error(...); return; } if (!data || data.length === 0) { return; } and build simpleAddress from the parts that are present, filtering out undefined before joining.

### `HW-09-0077` - The marker id arrives asynchronously into a field the next long press replaces, so a favourite can be saved with an empty id or with the id of a different marker.

- Category B, severity medium, confidence confirmed
- Features: TOUR-12
- Document: `huawei_industry_tree/09_tourism/docs/12_add_and_collect_map_marker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_and_collect_map_marker-0000002402495269
- Why: Two windows are open here. The address sheet is shown immediately, so a user who taps the save star before addMarker resolves stores markerId '' -- and every later tap on that pin computes a real id, finds no match, shows the address as un-saved, and pushes a second copy. Worse, this.currmarker is replaced wholesale by the next long press, so a pending then from the previous press writes its marker's id into the new currmarker, attaching one pin's identity to another pin's address.
- Fix: Await the marker before showing the sheet: const marker = await this.mapController?.addMarker(markerOptions); then set the id, the address and showAddressDialog together. If the sheet must open first, capture the MarkerType instance in a local and write the id into that local rather than into this.currmarker.

### `HW-09-0010` - The project tree printed in the document does not match the sample: the feature root, two module directory names and one file name are all spelled differently.

- Category E, severity low, confidence confirmed
- Features: TOUR-01
- Document: `huawei_industry_tree/09_tourism/docs/01_practice-tourist-park-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1-0000001965211653
- Why: The documented tree is the map a developer uses to navigate the download. HarmonyOS module paths are case sensitive in build-profile.json5 ("srcPath": "./features/home"), so the names in the document cannot be used as written, and ParkingVistingData.ets does not exist under any spelling.
- Fix: Regenerate the tree from the shipped zip: feature -> features, Home -> home, personalcenter -> PersonalCenter, ParkingVistingData.ets -> ParkingVisitingData.ets, and add the entrybackupability entry.

### `HW-09-0011` - The map setup note tells the reader to edit the application ID in model.json5, a file that does not exist; the setting lives in module.json5.

- Category E, severity low, confidence confirmed
- Features: TOUR-01
- Document: `huawei_industry_tree/09_tourism/docs/01_practice-tourist-park-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1-0000001965211653
- Why: There is no model.json5 in a HarmonyOS project; the configuration file is module.json5. Since a wrong or missing Client ID is the top cause of the blank map that the same note warns about, the misspelling sends the reader looking for a file that is not there.
- Fix: Change model.json5 to module.json5 and name the exact keys to fill in: the metadata entries client_id and app_id in phone/src/main/module.json5.

### `HW-09-0012` - The document states the app targets phones only, but the sample's entry module declares phone, tablet and 2in1 device types.

- Category E, severity low, confidence confirmed
- Features: TOUR-01
- Document: `huawei_industry_tree/09_tourism/docs/01_practice-tourist-park-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1-0000001965211653
- Why: deviceTypes is what decides which devices the HAP installs on. The document's stated scope and the shipped manifest disagree, and the sample additionally carries breakpoint handling (BreakpointSystem, GridRow with six breakpoints in LoginPage) that only makes sense for the wider set -- so it is the document, not the manifest, that is behind.
- Fix: Either state in 简介 that the entry HAP targets phone, tablet and 2in1 as the sample does, or trim deviceTypes in module.json5 to ["phone"] to match the stated scope.

### `HW-09-0025` - endWeekDay is recomputed from the -1 sentinels on every tap, building a Date from month -2 and day -1 whenever no end date is selected.

- Category B, severity low, confidence confirmed
- Features: TOUR-04
- Document: `huawei_industry_tree/09_tourism/docs/04_data_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_selection-0000002240198078
- Why: With the sentinels in place the expression is new Date(0, -2, -1), which JavaScript normalises to a date in 1899 rather than raising -- so endWeekDay silently holds the weekday of an unrelated day. It is masked only because the two render sites guard on `this.endDay === -1` before indexing this.week[this.endWeekDay]; remove or reorder either guard and the calendar prints a weekday for a date the user has not chosen.
- Fix: Guard the assignment: compute endWeekDay only inside the branch that already tests `this.endMonth !== -1 && this.endDay !== -1`, and reset it to 0 on the clear branch.

### `HW-09-0026` - The calendar's user-visible strings are hardcoded Chinese in the source and the module ships no locale resource directory.

- Category B, severity low, confidence confirmed
- Features: TOUR-04
- Document: `huawei_industry_tree/09_tourism/docs/04_data_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_selection-0000002240198078
- Why: Every other string on the page goes through $r('app.string....'), so the file already demonstrates the right mechanism and then bypasses it for exactly the strings a calendar must localise: weekday abbreviations, the year-month header, the date format and the pluralised night count. With no locale directories the app cannot be translated at all, and the date order (month before day) is baked into a format string rather than taken from the locale.
- Fix: Move the weekday names into a string array resource, the header and the button into $r placeholders filled with util.format or resource parameters, and add en_US and zh_CN resource directories so the base values can be overridden.

### `HW-09-0027` - The 实现思路 code block is labelled ts but does not compile, and it throws away the return value that actually drives the date-range logic.

- Category E, severity low, confidence confirmed
- Features: TOUR-04
- Document: `huawei_industry_tree/09_tourism/docs/04_data_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/data_selection-0000002240198078
- Why: The block is fenced as ts, not as pseudocode, so a reader is entitled to paste it. backgroundColor() without a ResourceColor is a compile error, and calling isDayLaterThan for its side effects teaches the opposite of what the sample does -- the whole point of the comparison is the boolean it returns, which is how a second tap before the start date re-anchors the range instead of ending it.
- Fix: Replace the block with the real branch from CalendarPage.ets, showing the if on the return value and a concrete colour argument, or fence it as pseudocode and say so.

### `HW-09-0031` - The initial date state is internally inconsistent: the page opens showing 4月23日 周日 (23 April, Sunday) although 23 April 2025 is a Wednesday.

- Category B, severity low, confidence confirmed
- Features: TOUR-05
- Document: `huawei_industry_tree/09_tourism/docs/05_address_exchange.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/address_exchange-0000002283895117
- Why: weekDay, monthDay and day are three independent literals that are only kept in step by the CalendarPickerDialog's onAccept handler (lines 222-226); nothing derives the initial weekday from the initial date, so the first screen the user sees -- and the one in the document's 效果预览 screenshot -- shows a weekday that does not belong to the date beside it. The hardcoded ISO string is a second hazard: new Date('2025-04-23') is parsed as UTC midnight, so in any timezone west of Greenwich it resolves to 22 April locally and the picker opens on the previous day.
- Fix: Derive all three from one source in aboutToAppear: `const d = this.selectedDate; this.monthDay = d.getMonth() + 1; this.day = d.getDate(); this.weekDay = this.weekDays[d.getDay()];` and construct the seed date with numeric arguments, `new Date(2025, 3, 23)`, so it is local time.

### `HW-09-0032` - The history list's ForEach key generator declares its parameter as string while the array elements are HistoryClass objects.

- Category C, severity low, confidence confirmed
- Features: TOUR-05
- Document: `huawei_industry_tree/09_tourism/docs/05_address_exchange.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/address_exchange-0000002283895117
- Why: The item builder and the key generator receive the same element, so the two parameter types must agree; declaring the key generator's parameter as string misrepresents what JSON.stringify is being handed. It happens to produce distinct keys here only because each entry pairs different $r() Resource ids, but the annotation is a lie about the data and hides the real question -- whether two history entries with the same origin and destination would collide, which they would.
- Fix: Type the key generator to match the list and key on something stable and unique: `}, (item: HistoryClass, index: number) => index.toString())`, or give HistoryClass an explicit id field and key on that.

### `HW-09-0033` - The rotation constant is 360 degrees under a comment that says 180, so each press spins the swap icon a full turn back to where it started.

- Category B, severity low, confidence confirmed
- Features: TOUR-05
- Document: `huawei_industry_tree/09_tourism/docs/05_address_exchange.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/address_exchange-0000002283895117
- Why: A swap control conventionally rotates 180 degrees so its resting orientation encodes the current direction -- which is what the comment describes and what would let the icon show, at a glance, that the addresses are reversed. Adding 360 returns the icon to exactly its previous orientation, so the animation reads as a spin with no state and the icon looks identical whether the addresses are swapped or not. Since the addresses themselves are only slid rather than exchanged (see the swap finding for this document), the icon is the only remaining cue and it carries none.
- Fix: Either set rotateAddAngle to 180 to match the comment and give the icon a resting state per direction, or correct the comment to say 360 if the full spin is the intended effect.

### `HW-09-0050` - The review page declares two @Local fields, reviewText and uploadedImages, that duplicate comment and selectedImageUris and are never read or written anywhere.

- Category B, severity low, confidence confirmed
- Features: TOUR-08
- Document: `huawei_industry_tree/09_tourism/docs/08_hotel_check_in_review.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/hotel_check_in_review-0000002364291253
- Why: Two names for the review body and two for the picked images, with only one of each live, is exactly the shape that produces a submit sending empty data: submit2Remote is a stub today, and whoever fills it in has a fifty-fifty chance of reading the dead pair. @Local also registers each of them with the V2 observation machinery for nothing.
- Fix: Delete reviewText and uploadedImages, or rename comment and selectedImageUris to them and delete the duplicates -- one name per concept.

### `HW-09-0051` - List is used as a plain layout container in both pages - nested inside a Scroll for the order groups, and around five static stars - so a scrolling component is created where a Column or Row is meant.

- Category C, severity low, confidence confirmed
- Features: TOUR-08
- Document: `huawei_industry_tree/09_tourism/docs/08_hotel_check_in_review.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/hotel_check_in_review-0000002364291253
- Why: A List is a scroll container with lazy layout; sized to its content inside another scroll container it can neither scroll nor virtualise, so it costs the machinery and gives none of it back, while nesting two scrollables means the gesture arbitration has to resolve a conflict that does not need to exist. The five-star row is the clearer case: five fixed children in a 180 vp box are a Row, and making them a horizontal List means a stray drag can scroll the stars. Neither List uses LazyForEach, so nothing about the lazy path is being demonstrated either.
- Fix: Replace the inner List in Index.myTab with a Column({ space: 12 }), and the star List with a Row({ space: 4 }). If the order list is ever meant to grow, invert the nesting instead: make the outer container the List, with one ListItemGroup per booking date, and drop the Scroll.

### `HW-09-0057` - The interval handle is guarded with a truthiness test, so a timer id of 0 would never be cleared, and the field is seeded with -1 rather than undefined.

- Category B, severity low, confidence confirmed
- Features: TOUR-09
- Document: `huawei_industry_tree/09_tourism/docs/09_attraction_talk_sample.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/attraction_talk_sample-0000002335049696
- Why: The field's declared type already says undefined is the absent value, and the initialiser then uses -1 instead, so two sentinels describe the same state. The guard tests neither: it tests truthiness, which is false for 0 as well as for undefined. A timer id of 0 is a legal return from setInterval, and it would leave the interval running for the life of the process, ticking every 500 ms against a destroyed page and writing its @State fields. The same guard also lets a second aboutToAppear stack a second interval, because nothing stops the first.
- Fix: Type the field number | undefined, initialise it to undefined, guard with if (this.timer !== undefined), and clear any existing interval at the top of aboutToAppear before creating a new one.

### `HW-09-0063` - The order list's ForEach key mixes the stable order id with the array index, so deleting or adding one order invalidates the key of every order after it.

- Category C, severity low, confidence confirmed
- Features: TOUR-10
- Document: `huawei_industry_tree/09_tourism/docs/10_travel_checkin_order.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/travel_checkin_order-0000002386506221
- Why: The whole point of the key generator is to let ForEach match an existing child to an item across re-renders. order.orderId alone does that; appending the index breaks it exactly when it matters, because delete-order and rebook both change the array and, through isAddOrDelete, re-sort it by creation time. Every item that shifts position gets a new key, so ForEach destroys and rebuilds those OrderContent components instead of updating them -- losing their state and doing work proportional to the list rather than to the change.
- Fix: Key on the identifier alone: (order: Order) => order.orderId. Concatenating with the index is only a workaround for duplicate ids, and these are unique by construction.

### `HW-09-0064` - The tab bar scrolls to index - 1 after every switch, which is a documented no-op when the first tab is selected.

- Category B, severity low, confidence confirmed
- Features: TOUR-10
- Document: `huawei_industry_tree/09_tourism/docs/10_travel_checkin_order.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/travel_checkin_order-0000002386506221
- Why: The Scroller reference states the contract: "If the value set is a negative value or greater than the maximum index of the items in the container, the value is deemed abnormal, and no scrolling will be performed." So selecting the first tab is the one case where the bar is not repositioned, which is exactly the case where a bar scrolled to the right needs to come back to the start. The intent -- keep one tab visible ahead of the selected one -- is right; the expression just has no floor.
- Fix: Clamp the target: this.scroller.scrollToIndex(Math.max(0, index - 1), true).

### `HW-09-0070` - The documented project tree names the validation file utils/CheckInformation.ets, but the sample ships utils/CheckMethods.ets.

- Category E, severity low, confidence confirmed
- Features: TOUR-11
- Document: `huawei_industry_tree/09_tourism/docs/11_check_information.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/check_information-0000002360422162
- Why: The tree is what a reader uses to find the two functions the document spends both of its code steps on, and the name it gives leads nowhere: there is no CheckInformation.ets anywhere in the sample. The confusion is compounded by CheckInformation being the top-level folder name, so a search for it lands on the project root rather than the file.
- Fix: Rename the entry in 工程目录 to CheckMethods.ets.

### `HW-09-0071` - The three TextInputs use the default keyboard and no length limit, so a form that validates an 11-digit phone and an 18-character ID card does nothing to constrain either at entry.

- Category C, severity low, confidence confirmed
- Features: TOUR-11
- Document: `huawei_industry_tree/09_tourism/docs/11_check_information.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/check_information-0000002360422162
- Why: The document's subject is 有效性的核对 (validity checking), and every constraint is deferred to the confirm button. The user gets an alphabetic keyboard for a phone number, can type any number of characters into both fields, and only learns at submit time that the input was the wrong shape. TextInput already carries the tools for this -- InputType.PhoneNumber raises the numeric keypad, and maxLength stops overlong entry -- so the validators would be left to catch what only they can catch, a wrong checksum or a bad province code.
- Fix: Set .type(InputType.PhoneNumber) and .maxLength(14) on the phone field, and .maxLength(18) with an inputFilter of [0-9Xx] on the ID card field. Keep the checksum validation on submit.

### `HW-09-0078` - The distance and duration strings are assembled from hardcoded Chinese units in code, and the module ships no locale resource directory.

- Category B, severity low, confidence confirmed
- Features: TOUR-12
- Document: `huawei_industry_tree/09_tourism/docs/12_add_and_collect_map_marker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_and_collect_map_marker-0000002402495269
- Why: These are the four strings that would have to change first for any non-Chinese locale, and they are the only user-visible text in the sample that cannot: units, and the imperial or metric choice behind them, are locale decisions, not code constants. The fallback branch is a second problem of the same kind -- a walk of under a minute is reported as 1分钟 rather than as less than a minute, which is a rounding decision baked into a template literal.
- Fix: Move the units into string resources with placeholders and format with util.format, add en_US and zh_CN directories, and use a measurement formatter so the unit system follows the locale.

### `HW-09-0079` - 1 sample project depends on third-party packages through unpinned version ranges

- Category B, severity low, confidence confirmed
- Features: TOUR-01
- Document: `huawei_industry_tree/09_tourism/docs/01_practice-tourist-park-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1-0000001965211653
- Why: A caret or tilde range lets ohpm resolve a different version than the one the sample was verified against, so the published sample is not reproducible and can break without any change to its own source. Sample projects are reference implementations; their dependency set should be exactly the one that was tested.
- Fix: Pin exact versions in oh-package.json5 dependencies, and record the resolved set in oh-package-lock.json5.

