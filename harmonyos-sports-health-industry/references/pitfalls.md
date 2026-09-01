# Pitfalls

> Generated from `features/*.md`. Source industry: `03_sports_health`, 15 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (1)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-03-0055` - Systematic: 13 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: SPORT-01, SPORT-03, SPORT-04, SPORT-05, SPORT-06, SPORT-07, SPORT-08, SPORT-09, SPORT-10, SPORT-11, SPORT-12, SPORT-13, SPORT-14
- Document: `huawei_industry_tree/03_sports_health/docs/04_calendar_schedule_events.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calendar_schedule_events-0000002274539357
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (54)

### `HW-03-0001` - The pedometer callback opens another pedometer subscription on every reading, so subscriptions multiply without bound for as long as the user walks.

- Category B, severity blocker, confidence confirmed
- Features: SPORT-01
- Document: `huawei_industry_tree/03_sports_health/docs/01_practice-sports-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-sports-health-app-architecture-v1-0000001952522073
- Why: Each delivery of the first subscription registers one more listener, and every one of those keeps firing, so the number of live pedometer subscriptions grows with every step the user takes -- with the first registered at a 10 ms interval, this compounds within seconds. Nothing ever unregisters: the component has no teardown, so leaving the running page leaves every accumulated subscription alive against a destroyed component, writing this.stepNum. The document presents this file as the reference implementation of its 实时记录步数 (real-time step counting) feature and quotes the second, inner sensor.on as the code to copy.
- Fix: Register once and release once: delete onPedometer entirely, keep the single sensor.on in getPedometerData with the callback stored in a field, and add aboutToDisappear that calls sensor.off(sensor.SensorId.PEDOMETER, this.sensorCallback).

### `HW-03-0002` - The step-counting section names the wrong permission and the wrong sensor, contradicting the same document's own solution section two pages earlier.

- Category E, severity high, confidence confirmed
- Features: SPORT-01
- Document: `huawei_industry_tree/03_sports_health/docs/01_practice-sports-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-sports-health-app-architecture-v1-0000001952522073
- Why: Two errors in one sentence, in the section a reader turns to for the implementation. A developer who follows it declares ACCELEROMETER, which does not authorise the pedometer, so sensor.on throws error 201 Permission denied -- and because both catch blocks in the sample are empty, the step counter silently reads zero with nothing logged. Naming the accelerometer as the subscribed sensor also misdescribes the code directly beneath it.
- Fix: Correct the sentence to match the document's own 方案设计 and the sample: subscribe to the pedometer sensor (SensorId.PEDOMETER) and declare ohos.permission.ACTIVITY_MOTION.

### `HW-03-0003` - offBLEConnectionStateChange creates a brand-new GattClientDevice and unsubscribes from that, leaving the real connection's listener registered and overwriting the live client handle.

- Category B, severity high, confidence confirmed
- Features: SPORT-01
- Document: `huawei_industry_tree/03_sports_health/docs/01_practice-sports-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-sports-health-app-architecture-v1-0000001952522073
- Why: createGattClientDevice returns a new client instance, so off() is called on an object that never had a listener while the instance that does keeps its subscription. The assignment then overwrites this.gattClient, so the connected client is unreachable: disconnectServer at lines 104-111 will call disconnect() on the fresh instance instead, and the real connection is left open with a live callback writing this.connectedDeviceId, this.connectState, this.connectedDeviceMap and this.deviceList. Nothing calls close() either, which the BLE reference requires to release a client instance.
- Fix: Do not create a client to unsubscribe. Call off on the field that holds the connected client and then release it: if (this.gattClient) { this.gattClient.off('BLEConnectionStateChange'); this.gattClient.disconnect(); this.gattClient.close(); this.gattClient = undefined; }

### `HW-03-0004` - The BLE scan is stopped only from onPageHide, which never runs because the device page is a plain @Component rather than an @Entry page.

- Category C, severity high, confidence confirmed
- Features: SPORT-01
- Document: `huawei_industry_tree/03_sports_health/docs/01_practice-sports-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-sports-health-app-architecture-v1-0000001952522073
- Why: The start half runs from aboutToAppear, which does fire on a plain component; the stop half is attached to a callback that a non-@Entry component never receives. So a low-latency BLE scan is started when the add-device view appears and is never stopped -- it continues after the user leaves the screen and for the rest of the process, with SCAN_MODE_LOW_LATENCY and MATCH_MODE_STICKY, which is the most power-hungry scan configuration available. The BLEDeviceFind subscription is likewise never removed and keeps pushing into the destroyed component's @State array.
- Fix: Move the teardown to aboutToDisappear, which does fire on a custom component, and keep onPageHide only if the hosting @Entry page needs to stop the scan when the app is backgrounded -- in which case put it on that page.

### `HW-03-0014` - Constants reads uiContext out of AppStorage at module scope, but the ability only writes that key inside the loadContent callback that runs after the page and its imports are evaluated.

- Category B, severity high, confidence probable
- Features: SPORT-03
- Document: `huawei_industry_tree/03_sports_health/docs/03_indoor_run.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/indoor_run-0000002266807001
- Why: The write happens in the completion callback of loadContent, and the page whose loading triggers that callback is what imports Constants, so the read is ordered before the write: ui is undefined and ui.vp2px is a call on undefined. Every geometry constant in the file derives from RADIUS, so nothing about the button animation can be computed. Marked probable because the exact point at which ArkTS evaluates a page's transitive module imports relative to the loadContent callback is not stated in this corpus, and the sample is published as working -- but the ordering visible in the two files is the wrong way round regardless, and the pattern breaks as soon as any other entry point imports Constants first.
- Fix: Do not compute layout constants at module load. Make RADIUS a function or a lazily initialised getter that takes a UIContext from the caller -- every component already has one via this.getUIContext() -- so the conversion happens when a component needs it rather than when the module is parsed.

### `HW-03-0018` - Opening the calendar page seeds a fixed demo row into the database every time, so the same event accumulates on every visit.

- Category B, severity high, confidence confirmed
- Features: SPORT-04
- Document: `huawei_industry_tree/03_sports_health/docs/04_calendar_schedule_events.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calendar_schedule_events-0000002274539357
- Why: The table creation is guarded with IF NOT EXISTS and the seed behind it is not, so only the schema is idempotent. Every time the user opens the calendar another 平安夜 (Christmas Eve) row is written, and because the primary key autoincrements there is no constraint to stop it -- the 24 December cell fills with identical entries that the user never created and cannot distinguish. Passing 'ID': 0 into an AUTOINCREMENT column is the tell that the row was meant to be a single fixed record rather than a growing set.
- Fix: Seed only when the table is empty: query the row count after createNoteTable and call initNoteTable only when it is zero. Better, drop the demo row from the utility entirely -- a database helper should not write sample content -- and load it from a fixture the app can choose to install once.

### `HW-03-0019` - The query result is assigned inside its own loop instead of being collected, so a day with several plans displays only the last one.

- Category B, severity high, confidence confirmed
- Features: SPORT-04
- Document: `huawei_industry_tree/03_sports_health/docs/04_calendar_schedule_events.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calendar_schedule_events-0000002274539357
- Why: conditionalNotesQuery returns the rows for one date, and the sheet the document is built around lets a user add as many plans to a date as they like -- so the loop exists precisely because the result is expected to hold more than one. Assigning rather than accumulating means the calendar silently shows a single plan per day: the user adds a second workout reminder, it is written to the database and to the system calendar, and it never appears in the app. A loop whose body ignores its own index is the signal that a push or a slice was intended.
- Fix: Collect instead of assigning -- give the selected day an array and fill it, this.selectedDate.matters = value, or push each row if the field must stay a list built element by element.

### `HW-03-0032` - The module declares no network permission and no client_id metadata, so the map this sample exists to show cannot load from a clean checkout.

- Category D, severity high, confidence confirmed
- Features: SPORT-08
- Document: `huawei_industry_tree/03_sports_health/docs/08_max_display_of_routes.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/max_display_of_routes-0000002335684832
- Why: Map Kit's own troubleshooting page names the missing network permission as the first cause of a blank map and prescribes adding ohos.permission.INTERNET and ohos.permission.GET_NETWORK_INFO to module.json5, and the Configuring the Client ID guide requires a client_id entry in the entry module's metadata for app identity verification. Without them no tiles are fetched, so a reader who completes every step the document lists still sees an empty screen with the trace animation running over nothing. Both permissions are system_grant and cost the user nothing.
- Fix: Add requestPermissions with ohos.permission.INTERNET and ohos.permission.GET_NETWORK_INFO, add a module-level metadata entry named client_id with a placeholder value, and add a fifth setup step to 说明 pointing at entry/src/main/module.json5.

### `HW-03-0033` - The route lookup casts a possibly absent Map entry to an array, so an unknown name crashes the page before the map is built.

- Category B, severity high, confidence confirmed
- Features: SPORT-08
- Document: `huawei_industry_tree/03_sports_health/docs/08_max_display_of_routes.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/max_display_of_routes-0000002335684832
- Why: Two unchecked casts sit in a row on the page's first three lines. getParamByName returns an array that is empty when the route carries no parameter, so [0] can be undefined; that value is then used as a Map key, which yields undefined, which the second cast presents as an array. The failure surfaces one call later as a for-of over undefined, in aboutToAppear, so the page throws before the map component is ever configured -- and nothing in between logs or guards. A route reached from anywhere other than the sample's own list, or with the parameter name mistyped, takes this path.
- Fix: Return the absence instead of casting it away: getPoints(name: string): Array<mapCommon.LatLng> { return this.routes.get(name) ?? []; }, and have RouteShow check that the parameter array is non-empty and that the resulting points array has at least one element before configuring the camera.

### `HW-03-0036` - User-authored text is concatenated into HTML without escaping and rendered in a Web component with JavaScript, file access and mixed content all enabled.

- Category D, severity high, confidence confirmed
- Features: SPORT-09
- Document: `huawei_industry_tree/03_sports_health/docs/09_outdoor_sports.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/outdoor_sports-0000002337059970
- Why: The document's scenario is 组团信息发布 (publishing group-activity information) -- content one user writes and others read -- so the text in the RichEditor is untrusted input by design. Typing a script tag into the activity description produces markup that is executed rather than displayed, because javaScriptAccess is on. fileAccess(true) is what makes that consequential rather than cosmetic: injected script runs in a document whose own images are loaded over file://, so it can reach into the app's sandbox, and mixedMode(MixedMode.All) additionally permits it to exfiltrate over plain HTTP. None of the three switches is needed to render styled text and images.
- Fix: Escape the text before interpolating it -- replace &, < and > with their entities in every branch that emits item.value -- and turn off what the document does not need: javaScriptAccess(false), and mixedMode set to MixedMode.None. Keep fileAccess only if the sandbox images cannot be inlined, and scope the loaded document's base URL rather than passing a blank one.

### `HW-03-0042` - The continuous location subscription is never cancelled and the page has no teardown, so accuracy-priority GPS keeps running after the user leaves the run screen.

- Category B, severity high, confidence confirmed
- Features: SPORT-11
- Document: `huawei_industry_tree/03_sports_health/docs/11_motion_trajectory.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/motion_trajectory-0000002351150394
- Why: The code-linter rule @performance/reasonable-gps-use-check states that apps without continuous tasks are not allowed to use location services in the background, and its Example of Incorrect Code is this exact shape -- on('locationChange', ...) with no matching off(). With ACCURACY priority, trajectory-tracking scenario and a zero distance filter, the subscription is configured to report as often as the hardware can, so leaving the page keeps GNSS at full rate for the life of the process. The retained callback also keeps writing this.point and calling animateCamera on a controller whose component is gone.
- Fix: Keep the callback in a field and add aboutToDisappear that calls geoLocationManager.off('locationChange', this.locationChange), and stop the subscription from the ability's onBackground as the linter rule's correct example does.

### `HW-03-0043` - Every location update adds another two-point trace overlay to the map and none is ever removed, so a run accumulates one overlay per GPS fix.

- Category B, severity high, confidence confirmed
- Features: SPORT-11
- Document: `huawei_industry_tree/03_sports_health/docs/11_motion_trajectory.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/motion_trajectory-0000002351150394
- Why: addTraceOverlay creates a map overlay; calling it per fix builds the track out of hundreds of separate two-point overlays instead of one growing polyline. With the request configured for continuous reporting, an hour's run leaves thousands of live overlays on the controller, each with its own animation, and the map has no way to shed them because no handle was kept. That this.points is maintained in the same callback and never used shows the single-track alternative was in reach: one overlay rebuilt or replaced from the accumulated array is what the API is for.
- Fix: Keep one overlay: store the handle returned by addTraceOverlay, remove it before adding the updated track, and build the new one from this.points rather than from the last pair. If per-segment animation is wanted, cap the number of retained overlays and remove the oldest.

### `HW-03-0049` - Horizontal drags check only the array bounds, not the row, so dragging a left-column card left moves it up to the end of the previous row.

- Category B, severity high, confidence confirmed
- Features: SPORT-13
- Document: `huawei_industry_tree/03_sports_health/docs/13_custom_exercise_dashboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_exercise_dashboard-0000002361577984
- Why: In a two-column grid a leftward drag from the left column has nowhere to go, and a rightward drag from the right column likewise. The bounds check admits both: dragging the left cell of row two to the left swaps it with the right cell of row one, so the card jumps up and across rather than moving in the direction of the finger, and the compensating offsetX shifts it the wrong way while it does so. The vertical helpers show the author knew the layout is two columns; only the horizontal pair was written as if the grid were a flat list.
- Fix: Test the column as well as the bounds: return early from left when index % 2 === 0 and from right when index % 2 === 1, deriving the 2 from the same constant that drives the vertical step so a three-column layout stays consistent.

### `HW-03-0051` - Which action a sheet button performs is decided by comparing its resolved display text, so the behaviour depends on translated strings being distinct.

- Category B, severity high, confidence confirmed
- Features: SPORT-14
- Document: `huawei_industry_tree/03_sports_health/docs/14_scan_to_add_medication.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/scan_to_add_medication-0000002396859981
- Why: The identity of a control is being carried by its user-visible text, so a translator can change what a button does: give two buttons the same wording in any locale and the scan button silently becomes the not-implemented toast, or the wrong branch runs. Comparing the ids themselves would be exact, locale-independent and free, and both ids are already in hand at the comparison. The indirection also costs two synchronous resource lookups on every tap, and both are optional-chained, so if resourceManager is undefined both sides evaluate to undefined, the equality holds, and the else branch runs regardless of which button was pressed.
- Fix: Compare the resource ids rather than their resolved values -- if (text.id === $r('app.string.scan').id) -- or better, pass an explicit action enum into the builder alongside the label so the control's behaviour does not depend on its wording at all.

### `HW-03-0005` - Every Bluetooth and sensor call in the framework is wrapped in a catch that discards the error, including the permission-denied case both APIs report that way.

- Category B, severity medium, confidence confirmed
- Features: SPORT-01
- Document: `huawei_industry_tree/03_sports_health/docs/01_practice-sports-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-sports-health-app-architecture-v1-0000001952522073
- Why: These five handlers cover every way the two headline features of this architecture can fail -- Bluetooth switched off, the ACCESS_BLUETOOTH or ACTIVITY_MOTION permission not granted, the sensor service unavailable -- and each converts it into silence. The step counter shows zero and the device list stays empty with nothing in the log to say which of those happened, so the first question a developer asks cannot be answered at all. The project ships a Logger at products/phone/src/main/ets/utils/Logger.ets, so the means to report is already present.
- Fix: Log the BusinessError code and message in every handler through the project's Logger, and surface the distinguishable cases to the UI -- Bluetooth disabled and permission denied both need a different prompt from a generic failure.

### `HW-03-0006` - The client_id metadata is nested inside the ability instead of the module, and it carries a real-looking numeric value rather than a placeholder.

- Category D, severity medium, confidence confirmed
- Features: SPORT-01
- Document: `huawei_industry_tree/03_sports_health/docs/01_practice-sports-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-sports-health-app-architecture-v1-0000001952522073
- Why: The service reads the client ID from the module's metadata, so an entry nested in an ability is not where it is looked for and app identity verification has nothing to verify against. The value is the second problem: every other template in this corpus ships an obvious placeholder -- "xxx", "xxxx", "xxxxxxxxxx" -- while this one ships 111264125, which reads as a real AGC project's client ID left in by accident. A developer copying this framework inherits another project's identifier and has no cue that it needs replacing.
- Fix: Move the metadata block out of the ability to the module object, as a sibling of abilities, and replace the value with an obvious placeholder plus a comment saying it must be filled in from AGC.

### `HW-03-0007` - All three permissions share $string:EntryAbility_desc as their reason and are declared when: always, including a background-running permission nothing uses.

- Category D, severity medium, confidence confirmed
- Features: SPORT-01
- Document: `huawei_industry_tree/03_sports_health/docs/01_practice-sports-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-sports-health-app-architecture-v1-0000001952522073
- Why: Three separate problems in one block. The reason string is the sentence the system shows before the user decides, and pointing all three at the ability description means the pedometer and the Bluetooth scan are justified with the same generic text -- and the user cannot tell which request is which. when: always asks for the permissions to be usable in the background, which is a stronger claim than a foreground step counter or an on-screen device scan needs; inuse is the correct scope for both. And KEEP_BACKGROUND_RUNNING has no consumer at all: the permission only takes effect together with a continuous task and a backgroundModes declaration, neither of which exists, so it is requested and unusable.
- Fix: Give each permission its own purpose-specific reason string, change when to inuse for ACCESS_BLUETOOTH and ACTIVITY_MOTION, and delete KEEP_BACKGROUND_RUNNING unless a continuous task is added with the backgroundModes it requires.

### `HW-03-0008` - The Bluetooth permission note points the reader at the Sensor Service Kit and at the wrong reference page for the permission list.

- Category E, severity medium, confidence confirmed
- Features: SPORT-01
- Document: `huawei_industry_tree/03_sports_health/docs/01_practice-sports-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-sports-health-app-architecture-v1-0000001952522073
- Why: The paragraph is the only place the document states the Bluetooth permission, and it attributes it to the wrong kit -- so a reader chasing the link lands on the sensor documentation, which says nothing about ACCESS_BLUETOOTH. It also implies the permission goes in a Sensor Service Kit file, when module.json5 is the entry module's own manifest and there is one per module. The permission named is correct; only the routing around it is wrong, which makes it the kind of error a reader cannot detect without already knowing the answer.
- Fix: Rewrite the sentence to point at Connectivity Kit and at the entry module's module.json5: the BLE scan requires ohos.permission.ACCESS_BLUETOOTH, declared in products/phone/src/main/module.json5, and link the BLE development guide rather than the sensor kit introduction.

### `HW-03-0010` - The document plans two Entry HAPs for phone and smart screen, but the smart-screen module is an empty HAR and the one HAP does not target a screen.

- Category E, severity medium, confidence confirmed
- Features: SPORT-01
- Document: `huawei_industry_tree/03_sports_health/docs/01_practice-sports-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-sports-health-app-architecture-v1-0000001952522073
- Why: The two-HAP split is the first architectural claim the document makes and the premise of its 应用健康应用接续 innovation section, which describes handing a video or a workout page from the phone to the screen. Neither端 exists in the code: there is no second entry to continue to, and the single HAP would not install on a screen. A reader planning a multi-device build from this template finds the module scaffolded and hollow, with no indication in the document that the smart-screen half is unimplemented.
- Fix: Either state in 简介 and 软件视图设计 that only the phone HAP is provided and the smartscreen module is a placeholder, or ship the second entry: change the module type to entry, give it an ability and pages, and add the screen device type.

### `HW-03-0011` - PermissionsUtil.checkPermissions overwrites its verdict on every loop pass, so only the last permission decides whether the request is raised.

- Category B, severity medium, confidence confirmed
- Features: SPORT-01
- Document: `huawei_industry_tree/03_sports_health/docs/01_practice-sports-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-sports-health-app-architecture-v1-0000001952522073
- Why: ACTIVITY_MOTION and ACCESS_BLUETOOTH are granted independently, so with Bluetooth allowed and motion denied the loop ends on the granted one, applyResult is true and no request is raised -- leaving the step counter, the feature the document devotes a whole section to, permanently unauthorised. The defect is currently latent because MainPage calls requestPermissions directly and checkPermissions has no call site, but the method is the util's public entry point and is what a developer adding a third permission would reach for. The identical loop appears in two other industry templates in this corpus, so it is a pattern being copied between practices rather than a one-off.
- Fix: Accumulate instead of assigning, and stop at the first missing permission: let missing = false; for (const p of permissions) { if (await this.checkAccessToken(p) !== abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED) { missing = true; break; } } if (missing) { this.requestPermissions(permissions, context); }

### `HW-03-0012` - The main page fires the runtime permission request and the notification authorization request together in aboutToAppear, and never checks whether either was granted.

- Category B, severity medium, confidence confirmed
- Features: SPORT-01
- Document: `huawei_industry_tree/03_sports_health/docs/01_practice-sports-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-sports-health-app-architecture-v1-0000001952522073
- Why: Three problems compound. The two calls are unsequenced, so the user is shown a runtime permission dialog and a notification authorization dialog stacked on top of each other the first time the app's main page appears, before anything has explained why either is needed. Neither result is read: PermissionsUtil.requestPermissions cannot report back, and the notification handler discards the outcome -- so the app cannot tell that the pedometer is unauthorised and cannot fall back. Typing the notification error as Error rather than BusinessError is what hides the refusal code: the notification guide states that error code 1600004 means the authorization was rejected, and distinguishing it from a genuine failure is the only way to know whether to offer openNotificationSettings.
- Fix: Sequence the two requests and read both results: await the permission request, which should return its authResults, and only then request notification authorization, typing its catch as BusinessError and branching on err.code === 1600004. Move both out of aboutToAppear to the point where each capability is first needed.

### `HW-03-0013` - Thirty-four strings were added to the Chinese resource file and are used almost nowhere, while 89 Chinese literals sit inline and the English file holds only generated boilerplate.

- Category B, severity medium, confidence confirmed
- Features: SPORT-01
- Document: `huawei_industry_tree/03_sports_health/docs/01_practice-sports-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-sports-health-app-architecture-v1-0000001952522073
- Why: The localisation work was started and abandoned: someone populated zh_CN with 34 application strings, and the pages then bypass them. So the resource file is a second, stale copy of text that lives inline, and the en_US file it was meant to be translated against was never filled in -- leaving the app Chinese-only despite shipping the directory structure that says otherwise. For a framework template whose stated purpose is to be copied and re-skinned, every one of those 89 literals is a line a downstream developer has to find and replace by hand.
- Fix: Move the inline text into resources/base/element/string.json, reference it with $r, and populate en_US with the translations. Reconcile or delete the 34 unused zh_CN entries so there is one source of truth per string.

### `HW-03-0015` - The path commands are assembled with the %d integer specifier, so every fractional coordinate of the curve is truncated to a whole pixel.

- Category A, severity medium, confidence confirmed
- Features: SPORT-03
- Document: `huawei_industry_tree/03_sports_health/docs/03_indoor_run.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/indoor_run-0000002266807001
- Why: Path commands accept fractional coordinates, and this animation is built on a square root sampled every 4 ms in half-percent steps -- precisely the case where sub-pixel positions matter. Truncating each endpoint to an integer quantises the curve onto the pixel grid, so the sweep advances in visible jumps instead of gliding, and consecutive progress values that differ by less than one pixel produce an identical path string and no movement at all. The effect is worst near the semicircles, where the derivative of the square root is largest.
- Fix: Use the floating-point specifier throughout formatPathCommands: replace every %d with %f in the M, A and L commands.

### `HW-03-0016` - The long-press interval is cleared only from the gesture's own end and completion paths, and the component defines no teardown, so a press interrupted by leaving the page leaves a 4 ms timer running.

- Category B, severity medium, confidence confirmed
- Features: SPORT-03
- Document: `huawei_industry_tree/03_sports_health/docs/03_indoor_run.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/indoor_run-0000002266807001
- Why: Both clears live on the gesture. If the component is destroyed while a press is in flight -- a navigation, a call arriving, the ability being backgrounded -- neither runs, and a callback firing 250 times a second survives against a destroyed component, writing this.progress, this.pathCommands and four colour fields through changeProgressNum and changeColor. The zero seed compounds it: zero is a legal timer id, so a guard of the form if (this.interval) would skip a real handle, and clearInterval is called unconditionally here on a value that may be a stale id from a previous press.
- Fix: Declare the handle as number | undefined initialised to undefined, clear it through one helper that checks for undefined and resets the field, and call that helper from aboutToDisappear as well as from onActionEnd and the completion branch.

### `HW-03-0017` - Every control on the running screen is placed at an absolute pixel offset, so the layout is pinned to one screen size.

- Category C, severity medium, confidence confirmed
- Features: SPORT-03
- Document: `huawei_industry_tree/03_sports_health/docs/03_indoor_run.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/indoor_run-0000002266807001
- Why: position() takes the element out of the layout and fixes it, so a top of 675 vp assumes a screen taller than that and a left of 118 assumes a particular width. The module declares deviceTypes for phone, tablet and 2in1, and on any of them other than the reference phone the finish button and its label land in the wrong place or off screen entirely -- the negative x on the header image shows the values were tuned against one device. The two inline numbers, 178 and 93, are the same kind of value as the seven in Constants and were not collected with them, so the geometry cannot even be retuned from one place.
- Fix: Lay the three buttons out with a Stack or a Row and alignment rules rather than absolute positions, keep only the sizes as constants, and let the safe-area padding the page already applies handle the vertical placement. If absolute positioning is kept for the animation, derive the offsets from the measured container via onAreaChange instead of hardcoding them.

### `HW-03-0020` - The default end time is built by adding one to the current hour without wrapping, so any plan created after 23:00 gets an invalid time.

- Category B, severity medium, confidence confirmed
- Features: SPORT-04
- Document: `huawei_industry_tree/03_sports_health/docs/04_calendar_schedule_events.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calendar_schedule_events-0000002274539357
- Why: getHours() returns 0-23, so between 23:00 and midnight the expression yields 24 and the sheet opens showing an end time of 24:07 -- a string no clock uses. Parsing '2025/12/24  24:07' produces an invalid Date, so this.endTime is unusable and whatever is written to the calendar event and to the database carries it. The hour is also the only field rolled: the date is still today's, so a plan spanning midnight is expressed as ending after it began on the same day.
- Fix: Add the hour to the Date rather than to its hour field, so the roll-over is handled: const end = new Date(date.getTime() + 60 * 60 * 1000), then format end.getHours() and end.getMinutes() -- and carry end's date, not tmpTime's, when the hour wraps past midnight.

### `HW-03-0021` - requestCalendarPermission takes the caller's current flag as its starting value and returns it unchanged on refusal, so passing true reports a grant that never happened.

- Category B, severity medium, confidence confirmed
- Features: SPORT-04
- Document: `huawei_industry_tree/03_sports_health/docs/04_calendar_schedule_events.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calendar_schedule_events-0000002274539357
- Why: The function decides a permission outcome and lets the caller pre-load the answer. Today this.userGrant is false at the only call site so the result is correct, but the contract is inverted: once the sheet is reopened after a successful grant -- or any caller passes true -- a subsequent refusal returns true and the app proceeds to write calendar events it is no longer authorised to write. The parameter serves no purpose the return value does not, and the missing try/catch is the second half: requestPermissionsFromUser can reject, and an unhandled rejection here leaves the sheet with neither a grant nor a failure path.
- Fix: Drop the parameter and derive the answer solely from the result: return grantStatus.authResults[0] === 0 && grantStatus.authResults[1] === 0, wrapped in try/catch returning false on rejection with the BusinessError logged.

### `HW-03-0022` - Both calendar permissions are requested the moment the plan sheet opens, before the user has indicated they want the plan on the system calendar.

- Category D, severity medium, confidence confirmed
- Features: SPORT-04
- Document: `huawei_industry_tree/03_sports_health/docs/04_calendar_schedule_events.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calendar_schedule_events-0000002274539357
- Why: READ_CALENDAR and WRITE_CALENDAR are user_grant permissions covering the user's whole calendar, and the sheet that triggers them is the ordinary way to add a workout plan -- a plan the app stores in its own database whether or not the calendar is involved. So a user who only wants an in-app reminder is asked for full calendar access before typing anything, with no context for the request beyond the reason string. Asking at the point of the confirm tap, or behind an explicit sync-to-calendar control, gives the user the connection between the request and what they were doing.
- Fix: Move the request to the confirm handler, immediately before addEvent, or gate it behind a visible toggle in the sheet. Keep the in-app plan working when the permission is refused, which the database path already allows.

### `HW-03-0023` - The year chart builds its month labels from a Date whose year argument is a month number, so every label is dated to 1900.

- Category B, severity medium, confidence confirmed
- Features: SPORT-05
- Document: `huawei_industry_tree/03_sports_health/docs/05_period_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/period_chart-0000002280744357
- Why: The bug is masked because the only consumer asks for the month alone -- DateUtils.ets:214-221, "let dateFormat: Intl.DateTimeFormat = new Intl.DateTimeFormat(locale, { month: 'short' }); return dateFormat.format(date);" -- so the label reads correctly today by coincidence. It stops being harmless the moment the axis wants a year, a month-and-year label, or any locale whose short month form varies by era, and a reader copying this line into a chart that shows the year gets 1900 with no clue why.
- Fix: Pass the year: new Date(timePeriod.beginDate.getFullYear(), monthIndex).

### `HW-03-0024` - An 82 KB data file is read and parsed synchronously on the UI thread in aboutToAppear, with no error handling.

- Category C, severity medium, confidence confirmed
- Features: SPORT-05
- Document: `huawei_industry_tree/03_sports_health/docs/05_period_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/period_chart-0000002280744357
- Why: aboutToAppear runs on the UI thread before the first frame, so the read, the UTF-8 decode, the parse of 82 KB and the per-record Date construction all happen before anything is drawn -- the page opens late by the cost of all four. The resource manager offers an asynchronous getRawFileContent for exactly this, and a chart already has a natural empty state to show while it loads. The missing error handling is the second half: getRawFileContentSync throws when the asset is absent and JSON.parse throws on malformed content, and either would take the page down rather than showing an empty chart.
- Fix: Use the asynchronous getRawFileContent and await it in an async aboutToAppear, wrap the read and the parse in try/catch, and render an empty or error state when statistics stays empty. For a file this size consider parsing it off the UI thread with a TaskPool task.

### `HW-03-0025` - The day view reads its headline figure by indexing past the end of an empty array, returning undefined where the other three periods return zero.

- Category B, severity medium, confidence confirmed
- Features: SPORT-05
- Document: `huawei_industry_tree/03_sports_health/docs/05_period_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/period_chart-0000002280744357
- Why: The two ternaries immediately above prove the empty case was anticipated, and then the average is taken without the same guard. A day with no readings is the ordinary case in a health chart -- the user opens the app on a day they did not wear the device -- and the page renders that value: ChartPage assigns it to measuredHeartRate, so the headline reads undefined instead of a dash or a zero. The inconsistency between the four period functions is what makes it hard to spot: three are safe, the one most often shown first is not.
- Fix: Guard the same way as the other three, and decide what an absent reading should display: average: dataSet.length ? dataSet[dataSet.length - 1] : 0, with the page rendering a placeholder rather than 0 when there is no measurement.

### `HW-03-0026` - The scoring buttons track their pressed state from Down and Up only, so a touch that ends in Cancel leaves the button highlighted for good.

- Category C, severity medium, confidence confirmed
- Features: SPORT-06
- Document: `huawei_industry_tree/03_sports_health/docs/06_match_scorer.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/match_scorer-0000002329513545
- Why: A touch sequence does not always end in Up: pressing a button and then dragging off it, or having the gesture pre-empted by a scroll or an incoming dialog, terminates the sequence with Cancel instead. The score is correctly not awarded in that case, but isClick is never cleared, so the button stays blue and looks pressed until something else re-renders it -- and because each of the three buttons has its own index, several can be stuck at once. This is exactly the case a scorer hits at the courtside, where taps are hurried and often slide.
- Fix: Clear the flag on both terminal types and award the score only on Up: else if (event.type === TouchType.Up) { award; this.isClick[index] = false; } else if (event.type === TouchType.Cancel) { this.isClick[index] = false; }

### `HW-03-0027` - The ten-minute countdown reaches zero with nothing observing it, so the end of the period is never signalled.

- Category B, severity medium, confidence confirmed
- Features: SPORT-06
- Document: `huawei_industry_tree/03_sports_health/docs/06_match_scorer.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/match_scorer-0000002329513545
- Why: The document's first stated feature is 比赛倒计时 (the match countdown), and a countdown that no one observes cannot end anything: at zero the display simply stops and the app carries on as though play continued -- no sound, no visual change, and the plus-score buttons stay live. For a basketball scoreboard the buzzer is the point of the clock. onTimer is the only way to learn the remaining time, since TextTimer keeps the count internally and the controller exposes only start, pause and reset.
- Fix: Attach onTimer and act when the remaining time reaches zero: .onTimer((utc: number, elapsedTime: number) => { if (elapsedTime >= Constants.TIME_COUNT) { this.textTimerController.pause(); this.isPause = false; /* signal the end of the period */ } }).

### `HW-03-0029` - All twelve ring and badge colours are inline hex literals, so the screen ignores the dark resource directory it ships and renders identically in dark mode.

- Category C, severity medium, confidence confirmed
- Features: SPORT-07
- Document: `huawei_industry_tree/03_sports_health/docs/07_sport_three_ring.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sport_three_ring-0000002345359673
- Why: The dark directory exists and is wired up -- it is what flips the splash background -- so the mechanism is in place and every colour on the actual screen bypasses it. The pale ring backgrounds are the concrete problem: #d5e5ff, #feefcd and #fbd4d3 are near-white tints chosen to sit on a white page, and on a dark background they become three bright halos, while the page's own text switches to the system's dark palette around them. A fitness ring screen is mostly colour, so this is the sample where the omission is most visible.
- Fix: Move the six ring colours and the three badge fills into resources/base/element/color.json, add dark variants to resources/dark/element/color.json, and reference them with $r('app.color....'). Keep the hue and adjust the tint for the dark surface rather than reusing the light one.

### `HW-03-0034` - getExtremum returns out-of-range sentinels for an empty track, and the page averages them into a camera target in the Gulf of Guinea.

- Category B, severity medium, confidence confirmed
- Features: SPORT-08
- Document: `huawei_industry_tree/03_sports_health/docs/08_max_display_of_routes.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/max_display_of_routes-0000002335684832
- Why: With no points the four sentinels average to latitude 0 and longitude 0, so the map opens on the Atlantic off West Africa at zoom 16, and the LatLngBounds built from the raw values has its northeast corner south and west of its southwest corner -- an inverted rectangle handed to map.newLatLngBounds. Neither is reported: there is no guard on points.length anywhere between the model and the camera. A single-point track is the second case the sentinels do not cover, since northeast and southwest then coincide and the bounds have zero extent.
- Fix: Make the empty case explicit rather than encoding it in sentinels: have getExtremum return undefined for an empty array, and have the page fall back to a default camera position and skip addTraceOverlay when there is no track. Handle the single-point case by expanding the bounds by a small delta before calling newLatLngBounds.

### `HW-03-0035` - The mapLoad listener and the trace overlay are never released; the page defines no aboutToDisappear.

- Category B, severity medium, confidence confirmed
- Features: SPORT-08
- Document: `huawei_industry_tree/03_sports_health/docs/08_max_display_of_routes.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/max_display_of_routes-0000002335684832
- Why: The official Map Kit FAQ pairs registration with release: "aboutToDisappear(): void { this.mapEventManager?.off('mapLoad'); this.mapController?.clear(); }". Here the listener is registered on a page reached by pushing onto a NavPathStack, so leaving and re-entering the route view adds another subscription each time while the previous ones stay bound to destroyed components. Discarding the return of addTraceOverlay is the second half: the trace is the page's main visual and there is no handle with which to remove or replace it, so re-entering the page redraws over the previous animation rather than replacing it.
- Fix: Keep the callback and the overlay in fields, and add aboutToDisappear that calls off('mapLoad') on the event manager, removes the trace overlay, and then clear() on the controller.

### `HW-03-0037` - The only permission is scoped to an ability the module does not contain, and its reason string is a button label.

- Category D, severity medium, confidence confirmed
- Features: SPORT-09
- Document: `huawei_industry_tree/03_sports_health/docs/09_outdoor_sports.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/outdoor_sports-0000002337059970
- Why: usedScene.abilities names which of the module's abilities use the permission, so an ability that does not exist describes nothing and the declaration cannot be validated against the code -- it is copy-paste residue from a project that had a form ability. The reason is the second half: it exists to explain the request to the user, and pointing it at a button caption produces the words Continue Add. Neither is fatal because INTERNET is system_grant and never prompts, which is precisely why the errors survive: nothing exercises them. They ship in a template developers copy, where the same block will be reused for a permission that does prompt.
- Fix: Correct the scope to the ability that makes the network calls -- "abilities": ["EntryAbility"] -- and either give the permission a purpose-specific reason string or drop reason and usedScene entirely, since neither is required for a system_grant permission.

### `HW-03-0038` - The same AppStorage key is bound one-way in the component that writes it and two-way in the components that only read it, so the writer has to push and then re-publish by hand.

- Category C, severity medium, confidence confirmed
- Features: SPORT-09
- Document: `huawei_industry_tree/03_sports_health/docs/09_outdoor_sports.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/outdoor_sports-0000002337059970
- Why: The bindings are the wrong way round. The AppStorage guide states that a @StorageProp variable's local changes are not written back and are overwritten the next time the stored value changes, which is why the push alone would not reach the other two components and the explicit setOrCreate has to follow it. That extra call is easy to forget on a second write path, and it is the only thing making the feature work. Meanwhile the two read-only lists take a two-way binding they never use, so nothing in the code records which component owns the data.
- Fix: Swap them: give the publishing component @StorageLink so its push propagates and the manual setOrCreate can go, and give the two display components @StorageProp to make it explicit that they only read.

### `HW-03-0039` - A drawn tie leaves both competitors marked as losers, and the advancement code then promotes the second one anyway.

- Category B, severity medium, confidence confirmed
- Features: SPORT-10
- Document: `huawei_industry_tree/03_sports_health/docs/10_tournament_advancement_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/tournament_advancement_chart-0000002381782357
- Why: The ternary treats "competitor1 did not win" as "competitor2 won", so an equal aggregate silently advances the second team into the next round while the bracket renders neither of them as the winner -- the chart and the progression disagree. The sample's data avoids the case because penalties are folded into the aggregate at model/SourceDataModel.ets:26, "this.totalScore += (score + additionalScore)", and a shoot-out cannot end level. That makes the defect latent rather than absent: any dataset where a tie is broken by away goals, seeding or a rule the aggregate does not encode reaches it, and the document offers the sample for 篮球、网球、电竞 (basketball, tennis, esports) too, which resolve ties differently.
- Fix: Make the draw explicit rather than implicit: compute the winner once, and when the totals are equal either apply the tie-break rule or leave the next round's slot empty instead of promoting a team no comparison selected.

### `HW-03-0040` - The bracket slot is computed from an optionally-chained team index, so a missing opponent yields NaN and the truncation silently maps it to slot 0.

- Category B, severity medium, confidence confirmed
- Features: SPORT-10
- Document: `huawei_industry_tree/03_sports_health/docs/10_tournament_advancement_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/tournament_advancement_chart-0000002381782357
- Why: The ?. says the author considered opponentA absent, and nothing acts on that possibility -- the result flows straight into index arithmetic. A fixture with a bye, a walkover, or a not-yet-drawn opponent therefore writes into match group 0 instead of being skipped, overwriting the first round-of-16 tie with data from another round. Because ~~ turns NaN into a valid-looking 0 rather than throwing, the corruption is silent: the bracket renders, with one group showing the wrong teams. The same truncation idiom is used four times in the file, so the pattern spreads.
- Fix: Guard before computing: if (curSourceData.opponentA === undefined) { continue; } and type idx as number afterwards, or make opponentA non-optional in the model if a source record must always carry both sides.

### `HW-03-0044` - The map module declares no network permission, and both location permissions are justified with the ability description.

- Category D, severity medium, confidence confirmed
- Features: SPORT-11
- Document: `huawei_industry_tree/03_sports_health/docs/11_motion_trajectory.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/motion_trajectory-0000002351150394
- Why: Map Kit's own troubleshooting page names the missing network permission as the first cause of a blank map and prescribes ohos.permission.INTERNET and ohos.permission.GET_NETWORK_INFO in module.json5, so the tiles this sample draws its track over cannot load. The reason strings are the second problem: both are user_grant permissions whose reason is the sentence shown before the user decides, and pointing both at the ability description means a request for continuous precise location during a run is explained with generic boilerplate -- for the one permission in this sample that most needs a specific justification. The client_id block shows the author knew where module-level configuration goes, which makes the omission an oversight rather than a misunderstanding.
- Fix: Add ohos.permission.INTERNET and ohos.permission.GET_NETWORK_INFO, and give each location permission its own reason string explaining that the precise location is used to record the running route.

### `HW-03-0045` - The first recorded segment is drawn from a hardcoded placeholder coordinate in the South China Sea to wherever the user actually is.

- Category B, severity medium, confidence confirmed
- Features: SPORT-11
- Document: `huawei_industry_tree/03_sports_health/docs/11_motion_trajectory.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/motion_trajectory-0000002351150394
- Why: On the first callback lastPoint is still 22N 113E, a point in the sea near the Pearl River estuary, so the track opens with a segment stretching from there to the runner's actual position -- potentially thousands of kilometres of line drawn as part of the run. The same literal is the map's initial camera target at lines 100-104, so the map also opens there before the first fix arrives. Nothing distinguishes the seeded value from a real one, because lastPoint carries no not-yet-set state.
- Fix: Type lastPoint as mapCommon.LatLng | undefined, start it undefined, and skip the overlay on the first callback: if (this.lastPoint === undefined) { this.lastPoint = this.point; return; }. Seed the camera from the single-shot getCurrentLocation the page already performs rather than from a literal.

### `HW-03-0046` - The countdown is three animateTo calls nested by hand, so the loop count is fixed in the code shape while the number it counts from is a constant.

- Category B, severity medium, confidence confirmed
- Features: SPORT-12
- Document: `huawei_industry_tree/03_sports_health/docs/12_countdown_to_run.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/countdown_to_run-0000002394156713
- Why: The number of decrements is expressed by how deeply the calls are nested, and the number to count from is expressed by a constant, so the two can disagree with nothing to catch it: raising COUNTDOWN_NUMBER to 6 still runs three animations and the countdown ends showing 3. The nesting also makes the animation parameters impossible to vary per step without editing four blocks, and it is why the same two-line closure is written out three times.
- Fix: Drive the sequence from the constant with one recursive step: a method that decrements once inside animateTo and, from onFinish, calls itself while countdownNumber is above zero, then runs the shrink animation. The loop count then follows COUNTDOWN_NUMBER.

### `HW-03-0047` - The 3.5-second animation chain cannot be cancelled, and its final callback writes four @Link fields that belong to the parent.

- Category C, severity medium, confidence confirmed
- Features: SPORT-12
- Document: `huawei_industry_tree/03_sports_health/docs/12_countdown_to_run.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/countdown_to_run-0000002394156713
- Why: animateTo's onFinish fires when the animation completes, not when the component that started it still exists, so a user who backs out mid-countdown leaves a chain that keeps running and, three and a half seconds later, flips clickStart, countdownFinished and buttonState on the parent -- driving the run screen into the started state for a run the user cancelled. Starting the chain from onAppear rather than from an explicit call compounds it: onAppear fires whenever the view becomes visible, so a countdown can also be restarted by a re-entry while a previous chain is still in flight, and the two then interleave their decrements on the same shared counter.
- Fix: Guard the writes and the restart: keep a cancelled flag set in aboutToDisappear and checked at the top of every onFinish before touching state, and a running flag so changeText returns early if a chain is already in progress. Trigger the countdown from an explicit call by the parent rather than from onAppear.

### `HW-03-0050` - The card model carries its user-visible name and description as hardcoded Chinese literals rather than resources.

- Category B, severity medium, confidence confirmed
- Features: SPORT-13
- Document: `huawei_industry_tree/03_sports_health/docs/13_custom_exercise_dashboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_exercise_dashboard-0000002361577984
- Why: The interface makes the inconsistency explicit: the image is declared ResourceStr and resolves through the resource system, and the two strings printed next to it on the same card are not. Every card's title and subtitle is therefore untranslatable, in a model file that exists precisely to be the list a product team edits. ResourceStr accepts both a literal and a $r reference, so declaring the fields that way costs nothing and leaves the choice to the data.
- Fix: Type name and description as ResourceStr, as image already is, and fill exerciseCards with $r references backed by entries in string.json.

### `HW-03-0052` - The medication list keys each row on the barcode, so adding the same medicine twice gives two rows the same key, and an unavailable resource manager makes every key undefined.

- Category C, severity medium, confidence confirmed
- Features: SPORT-14
- Document: `huawei_industry_tree/03_sports_health/docs/14_scan_to_add_medication.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/scan_to_add_medication-0000002396859981
- Why: A barcode identifies a product, not an entry in the user's list, and the document's own scenario is 记录个人用药历史或保留购药记录 (recording a personal medication history or keeping purchase records) -- a history in which the same medicine legitimately appears more than once. Two rows then share a key, which is what ForEach uses to tell children apart, so the framework cannot distinguish them and the second insert does not render correctly. The optional chain is the second half: if resourceManager is undefined the key generator returns undefined for every row rather than a string, leaving the whole list unkeyed.
- Fix: Key on something unique to the entry rather than to the product: give MedicationScanInfo an id assigned when the row is created, or fall back to the array index combined with the code. Resolve resourceManager once and fail loudly if it is unavailable rather than optional-chaining it at every use.

### `HW-03-0009` - The documented project tree misspells a directory, renames the login page and omits a page the sample ships.

- Category E, severity low, confidence confirmed
- Features: SPORT-01
- Document: `huawei_industry_tree/03_sports_health/docs/01_practice-sports-health-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-sports-health-app-architecture-v1-0000001952522073
- Why: The tree is the map a reader uses to find the files the technical sections quote, and three of its entries do not resolve: compontents is not a directory, LoginPage2.ets is not a file, and the page that actually implements the privacy screen is absent from the map entirely. The duplicated 修改密码 comment compounds it by giving three different files the same description.
- Fix: Regenerate the tree from the shipped zip: compontents -> components, LoginPage2.ets -> LoginPage.ets, add PrivacyPage.ets, and give SecretPage.ets its own description.

### `HW-03-0028` - The documented project tree misspells the backup ability directory as entrybackupablility.

- Category E, severity low, confidence confirmed
- Features: SPORT-06
- Document: `huawei_industry_tree/03_sports_health/docs/06_match_scorer.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/match_scorer-0000002329513545
- Why: The tree is a five-entry list and one of them does not resolve. The misspelling transposes letters in the middle of a long lowercase word, which is exactly the kind that survives review and then costs a reader a search -- and the directory name is load-bearing, because module.json5 points at the extension ability by path.
- Fix: Correct the entry to entrybackupability.

### `HW-03-0030` - The four measurement units are concatenated into the display as Chinese literals while the labels beside them come from string resources.

- Category B, severity low, confidence confirmed
- Features: SPORT-07
- Document: `huawei_industry_tree/03_sports_health/docs/07_sport_three_ring.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sport_three_ring-0000002345359673
- Why: The units are the one part of each readout that has to change with the locale -- kilocalories, minutes and hours all have different abbreviations and different placement relative to the number in other languages, and some locales put the unit before it. Concatenating them into the string also fixes the order at build time, so a translation cannot move the unit even if the words were resourced. The row is half localisable and half not, split between two adjacent Text components.
- Fix: Move the four units into string.json and compose them with the number through a resource that carries the placeholder, so both the wording and the ordering come from the locale: $r('app.string.calories_of_target', this.exerciseCalories, this.targetCalories).

### `HW-03-0031` - The three ring icons are positioned by hardcoded offsets tuned to the ring diameters, so changing a ring size silently misplaces its icon.

- Category C, severity low, confidence confirmed
- Features: SPORT-07
- Document: `huawei_industry_tree/03_sports_health/docs/07_sport_three_ring.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sport_three_ring-0000002345359673
- Why: The icon has to sit on its own ring, so its offset is a function of that ring's radius and stroke width. Writing both as unrelated literals means the invariant is only in the author's head: an adaptation that changes a ring from 150 to 160 moves the arc and leaves the icon behind, with nothing failing and no comment recording why -55 was correct. The document's own step 3 describes the rings as 半径等差 (radii in equal steps), which is precisely the relationship the code does not encode.
- Fix: Derive both from one set of numbers: keep the radii and the stroke width as named constants and compute each width as 2 * radius and each icon offset as -(radius - strokeWidth / 2 + iconHeight / 2), so a change to a radius carries the icon with it.

### `HW-03-0041` - The documented project tree describes the match data model as a font-loading entity class.

- Category E, severity low, confidence confirmed
- Features: SPORT-10
- Document: `huawei_industry_tree/03_sports_health/docs/10_tournament_advancement_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/tournament_advancement_chart-0000002381782357
- Why: The tree is the only index into a seven-file project, and the one entry a reader most needs -- the data structure the document devotes its entire step 1 and a schematic figure to -- is labelled as something unrelated. The description is copy-paste residue from a different sample, so a reader skimming the tree for the model skips past it.
- Fix: Change the comment to describe what the file contains, for example 比赛数据模型 (the match data model).

### `HW-03-0048` - The button colour field is typed as Length and initialised with a colour resource.

- Category C, severity low, confidence confirmed
- Features: SPORT-12
- Document: `huawei_industry_tree/03_sports_health/docs/12_countdown_to_run.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/countdown_to_run-0000002394156713
- Why: The annotation was copied from the height field on the previous line and never corrected. It compiles because a Resource satisfies both aliases, so nothing catches it, and the cost is that the type no longer documents anything: a reader looking for what buttonColor accepts is told it takes a dimension, and assigning a plain vp number to it would type-check while producing an invalid colour. The correct alias, ResourceColor, would accept the resource, a Color enum member and a hex string, and reject a length.
- Fix: Type it as the colour it is: @State buttonColor: ResourceColor = $r('app.color.button_color_green');

### `HW-03-0053` - The note telling the reader where the test barcodes are misspells the directory as /sceenshots; the project directory is screenshots.

- Category E, severity low, confidence confirmed
- Features: SPORT-14
- Document: `huawei_industry_tree/03_sports_health/docs/14_scan_to_add_medication.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/scan_to_add_medication-0000002396859981
- Why: These two images are the only way to exercise the feature: the catalogue in MockData.ets holds two stubbed products, so any barcode other than the ones in these files returns undefined and the page falls back to a not-found toast. A reader following the note looks for a directory that does not exist, and a search for the misspelling finds nothing.
- Fix: Correct the note to read screenshots, matching the directory shipped in the sample.

### `HW-03-0054` - 1 sample project depends on third-party packages through unpinned version ranges

- Category B, severity low, confidence confirmed
- Features: SPORT-05
- Document: `huawei_industry_tree/03_sports_health/docs/05_period_chart.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/period_chart-0000002280744357
- Why: A caret or tilde range lets ohpm resolve a different version than the one the sample was verified against, so the published sample is not reproducible and can break without any change to its own source. Sample projects are reference implementations; their dependency set should be exactly the one that was tested.
- Fix: Pin exact versions in oh-package.json5 dependencies, and record the resolved set in oh-package-lock.json5.

