# Pitfalls

> Generated from `features/*.md`. Source industry: `02_convenient_life`, 31 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (1)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-02-0269` - Systematic: 23 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: LIFE-04, LIFE-05, LIFE-07, LIFE-08, LIFE-10, LIFE-11, LIFE-13, LIFE-14, LIFE-15, LIFE-17, LIFE-18, LIFE-19, LIFE-20, LIFE-21, LIFE-22, LIFE-23, LIFE-24, LIFE-25, LIFE-26, LIFE-27, LIFE-28, LIFE-29, LIFE-30
- Document: `huawei_industry_tree/02_convenient_life/docs/19_parcel_address_text_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_text_recognition-0000002326921238
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (270)

### `HW-02-0007` - The login page hardcodes the AES-CBC key and IV used to encrypt the user account, in a document whose stated focus is on-device data privacy

- Category D, severity blocker, confidence confirmed
- Features: LIFE-01
- Document: `huawei_industry_tree/02_convenient_life/docs/01_practice-convenient-life-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1-0000001952539489
- Why: Section D of the review forbids hardcoded secrets. The key and IV are compiled into the HAP, so the ciphertext stored in AppStorage under 'login' gives no protection at all: anyone who unpacks the application recovers the key. The sample's own inline comment states the rule it is breaking, and a fixed IV additionally makes CBC encryption deterministic, so identical accounts produce identical ciphertext.
- Fix: Replace the literals with a key handle managed by @kit.UniversalKeystoreKit (HUKS) or a secret persisted through @kit.AssetStoreKit, and derive a per-call IV from cryptoFramework.createRandom().generateRandomSync(16); persist the IV with the ciphertext rather than embedding it in the source.

### `HW-02-0061` - aboutToDisappear clears the wrong field, so the 10 ms auto-scroll timer is never stopped and keeps calling scrollTo on a destroyed component

- Category B, severity blocker, confidence confirmed
- Features: LIFE-09
- Document: `huawei_industry_tree/02_convenient_life/docs/09_easylife_loopscroll.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_loopscroll-0000002284598853
- Why: Section B of the review flags lifecycle cleanup problems. The interval survives the component, so after the page is left the callback keeps firing a hundred times a second, mutating the data source and calling scrollTo on a Scroller whose component is gone. The timer can never be reclaimed because its handle is held only in a field of the destroyed component, and every remount starts another one.
- Fix: Delete the unused intervalID field and change aboutToDisappear to clearInterval(this.intervalNum);. Setting this.intervalNum = CommonConstants.INIT_ZERO afterwards makes a double teardown harmless.

### `HW-02-0142` - The sample requests two location permissions at runtime that module.json5 never declares, so the request is rejected as invalid and no dialog is ever shown

- Category D, severity blocker, confidence confirmed
- Features: LIFE-20
- Document: `huawei_industry_tree/02_convenient_life/docs/20_confirm_direction_in_map_to_rotate_and_move.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/confirm_direction_in_map_to_rotate_and_move-0000002335234358
- Why: Section D of the review covers missing permission configuration. Both permissions are user-grant, so they must be declared before they can be requested; without the declaration the request resolves immediately with 2 for every entry and no dialog is shown. The .then does not inspect authResults, so it proceeds to enable the my-location layer and the my-location button as though the user had agreed - and the button, when tapped, has no permission behind it. The feature the whole page is built on cannot work from this zip.
- Fix: Add a requestPermissions block to entry/src/main/module.json5 with { "name": "ohos.permission.APPROXIMATELY_LOCATION", "reason": "$string:...", "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" } } and the same for ohos.permission.LOCATION, plus ohos.permission.INTERNET for Map Kit itself. MapBindSheet.zip#entry/src/main/module.json5 is the shape to copy.

### `HW-02-0211` - Every field the search produces is commented out and replaced with placeholder x strings, so the shipped sample displays no real outlet data at all

- Category B, severity blocker, confidence confirmed
- Features: LIFE-26
- Document: `huawei_industry_tree/02_convenient_life/docs/26_list_of_nearby_outlets.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/list_of_nearby_outlets-0000002347931784
- Why: The feature is a list of nearby outlets with their names, addresses and phone numbers. With the four bindings commented out, every row of that list shows the same eight or sixteen letter x characters, so nothing the location lookup and the place search compute is ever visible. A reader who builds and runs the sample to see what it does sees placeholder text, and the one control that still works on real data - tapping the phone row, which calls jumpToCallPage(item.phone) at :339 - dials a number that was never shown.
- Fix: Delete the four literal Text components and uncomment the four bindings directly above them. Nothing else has to change; the data is already in place.

### `HW-02-0001` - The documented project tree lists features/mine/.../SecurityCenter.ets (Security Center), but no such file exists anywhere in Life_Framework_Code_V1.zip

- Category E, severity high, confidence confirmed
- Features: LIFE-01
- Document: `huawei_industry_tree/02_convenient_life/docs/01_practice-convenient-life-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1-0000001952539489
- Why: Section E of the review requires the project tree shown in the document to match the actual zip. A reader following the tree will look for a Security Center implementation that was never shipped, and the module-division table promises a capability the sample project does not contain.
- Fix: Remove the SecurityCenter.ets entry from the 代码结构解读 tree (line 228) and add a note next to 表1 that 安全中心 is a placeholder in the framework code, or add the missing file to the sample project.

### `HW-02-0004` - The CardRecognition snippet in the document uses a callback property and a CallbackParam type that the sample project does not use; the shipped code uses onResult and CardRecognitionResult

- Category E, severity high, confidence confirmed
- Features: LIFE-01
- Document: `huawei_industry_tree/02_convenient_life/docs/01_practice-convenient-life-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1-0000001952539489
- Why: Section E of the review requires code snippets in the document to match the actual zip implementation. Both the property name and the callback parameter type differ from the shipped code, so the snippet does not compile against the @kit.VisionKit surface that the sample project itself imports.
- Fix: Replace 'callback:' with 'onResult:' and 'CallbackParam' with 'CardRecognitionResult' in the 证件识别代码实现 snippet, and add the '@kit.VisionKit' import line so the snippet is self-contained.

### `HW-02-0005` - In the card-recognition callback the optional chaining guards only cardInfo, so params.cardInfo.back being undefined throws a TypeError, and the call sits outside the try block

- Category B, severity high, confidence confirmed
- Features: LIFE-01
- Document: `huawei_industry_tree/02_convenient_life/docs/01_practice-convenient-life-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1-0000001952539489
- Why: The ?. operator short-circuits only the property access it is attached to. When cardInfo is present but back is undefined - which happens for a single-sided scan, and the code checks front for undefined two lines further down for exactly that reason - evaluating .originalImageUri on undefined raises a TypeError. Because the statement is outside the try block, nothing catches it and the recognition callback aborts before any field is filled in.
- Fix: Guard the whole read: const backUri = params.cardInfo?.back?.originalImageUri; if (backUri) { let file = fs.openSync(backUri, fs.OpenMode.READ_ONLY); try { ... } finally { fs.closeSync(file); } } and move the front-side field extraction out of the back-image branch so it still runs when only the front was scanned.

### `HW-02-0006` - EntryAbility subscribes to the window avoidAreaChange event but never unsubscribes, leaking the listener when the window stage is destroyed

- Category B, severity high, confidence confirmed
- Features: LIFE-01
- Document: `huawei_industry_tree/02_convenient_life/docs/01_practice-convenient-life-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1-0000001952539489
- Why: Section B of the review flags subscriptions that are opened but never closed. The listener holds a closure that writes into AppStorage; keeping it registered after the window stage is torn down is a resource leak and can produce writes against a stale window.
- Fix: Hold the window handle and the callback as fields, then in onWindowStageDestroy call this.windowClass?.off('avoidAreaChange', this.avoidAreaCallback);

### `HW-02-0010` - The documented on-device secure storage scheme is not wired into the sample: AssetStore.ets is imported by no file and runs asset.add as an unconditional module side effect

- Category E, severity high, confidence confirmed
- Features: LIFE-01
- Document: `huawei_industry_tree/02_convenient_life/docs/01_practice-convenient-life-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1-0000001952539489
- Why: Section E of the review requires the document to describe what the sample project actually does. The document claims the framework code demonstrates the Asset Store Kit scheme for exactly the data the login flow handles, but the module is dead code; and even if it were imported, the top-level asset.add would fire on import rather than at a point the caller controls, and would add a demo password rather than the user's data.
- Fix: Wrap the asset.add call in an exported async function, export it from features/home/Index.ets, and call it from LoginPage.login() instead of AppStorage.setAndRef; alternatively add a note to the 端侧数据安全存储方案 section stating that AssetStore.ets is a standalone snippet.

### `HW-02-0013` - The bottom safe-area inset is read from the TYPE_SYSTEM avoid area, which is the status bar; the bottom navigation inset requires TYPE_NAVIGATION_INDICATOR

- Category A, severity high, confidence confirmed
- Features: LIFE-03
- Document: `huawei_industry_tree/02_convenient_life/docs/03_vehicle_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vehicle_keyboard-0000002236257106
- Why: Section A of the review requires API parameters to be used as the reference defines them. TYPE_SYSTEM does not describe the bottom navigation bar, so its bottomRect.height is not the inset the page needs. The page is drawn with setWindowLayoutFullScreen(true), so with a zero bottom padding the footer - anchored with .position({ bottom: ... }) at MainPage.ets:124 - sits underneath the system navigation bar.
- Fix: Replace the single query with two: AppStorage.setOrCreate('topAvoid', windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM).topRect.height); AppStorage.setOrCreate('bottomAvoid', windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR).bottomRect.height);

### `HW-02-0020` - Seat hit-testing converts screen coordinates by hand instead of using ClickEvent.x/.y, which already give the position inside the clicked Canvas

- Category C, severity high, confidence confirmed
- Features: LIFE-04
- Document: `huawei_industry_tree/02_convenient_life/docs/04_canvas_cinema.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_cinema-0000002272398929
- Why: Section C of the review covers UI defects. Because displayX/displayY are screen coordinates, the code has to subtract the canvas margin, the canvas offset and the status-bar height by hand, and StyleConstants.CANVAS_Y = 161 is a magic number derived from six unrelated resource floats (margin_ten 10 + title_height 26 + margin_thirty 30 + title2_height 25 + margin_ten 10 + title3_height 50, plus the page Column space 10). Any change to the header layout, any scroll, any split-screen or floating-window placement shifts every seat by one or more rows with no compile-time signal. ClickEvent.x/.y are already relative to the Canvas and need none of that.
- Fix: Replace e.displayX/e.displayY with e.x/e.y and delete CANVAS_Y, CANVAS_LEFT_MARGIN and the px2vp(topRectHeight) term from the hit-test. Update the document's step-1 snippet to match.

### `HW-02-0026` - The payment dialog receives the seat list through @Prop, which the CustomDialogController reference says does not track data changes in a builder

- Category C, severity high, confidence probable
- Features: LIFE-04
- Document: `huawei_industry_tree/02_convenient_life/docs/04_canvas_cinema.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_cinema-0000002272398929
- Why: Section C of the review covers state handling. If the dialog component is instantiated with the controller rather than on each open(), the @Prop deep copy is taken from the initial empty array, aboutToAppear then builds mess from an empty list, and the confirmation reads 您已选中以下座位：，确认支付吗？ ("you have selected the following seats: , confirm payment?") with no seats named - while the payment still goes through, because confirm() reads the parent's array directly.
- Fix: Change the declaration to @Link selectedSeats: Array<Array<number>>; and, because aboutToAppear sorts the array in place, sort a local copy instead: const ordered = this.selectedSeats.slice().sort((a, b) => a[1] === b[1] ? a[0] - b[0] : a[1] - b[1]); then build mess from ordered.

### `HW-02-0027` - The menu component indexes its arrays with a menu id instead of an array index, at eleven call sites

- Category B, severity high, confidence confirmed
- Features: LIFE-05
- Document: `huawei_industry_tree/02_convenient_life/docs/05_cascading_menu_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cascading_menu_selection-0000002274600681
- Why: Section B of the review covers null and out-of-range risks. The expressions only resolve because the bundled data happens to number every level 0,1,2,... in array order - entry/src/main/resources/rawfile/config/menu_item_list.json gives the four primary items ids 0-3 and each secondMenu ids starting at 0. The moment the data carries real category ids, or a category is filtered out, list[id] is undefined and the following property write throws. The two lists are also different collections: primaryMenuItemList is the whole tree while secondLevelMenuItemList is one branch, so a second-level id from a long branch can index past the end of a short one.
- Fix: Store the selected object rather than its id - @State selectedSecond: SecondMenuItem | undefined - or replace each subscript with a find() guarded by an undefined check. Keeping ids as subscripts silently couples the component to the ordering of the JSON file.

### `HW-02-0034` - The drag step is 50 vp while the rendered row pitch is 60 vp, so every swap moves the dragged note ten vp out of alignment

- Category C, severity high, confidence confirmed
- Features: LIFE-06
- Document: `huawei_industry_tree/02_convenient_life/docs/06_sticky_note.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sticky_note-0000002275644713
- Why: Section C of the review covers layout defects. After a swap the code subtracts one ITEM_HEIGHT from the visual offset to compensate for the item having moved one row in the data. Because the item actually moves 60 vp but the offset is corrected by 50 vp, the dragged note is left 10 vp below the finger after the first swap, 20 vp after the second, and so on; the swap threshold also fires 5 vp early because it is half of the wrong pitch.
- Fix: Delete both ITEM_HEIGHT literals and derive one constant from the two that already exist: export const ITEM_PITCH = Constants.STICKY_NOTE_VIEW_HEIGHT + Constants.STICKY_NOTE_LIST_SPACE; then import it in StickyNotePage.ets instead of re-declaring it.

### `HW-02-0035` - The swap bounds check runs after the offset has already been adjusted and tests the wrong limits, so dragging the first note up or the last note down shifts it by one row without reordering anything

- Category B, severity high, confidence confirmed
- Features: LIFE-06
- Document: `huawei_industry_tree/02_convenient_life/docs/06_sticky_note.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sticky_note-0000002275644713
- Why: Section B of the review covers out-of-range and state-drift defects. Dragging the top note upward past half a row makes target -1: the guard blocks the reorder, but offsetY and dragRefOffset have already been shifted by a full row, so the note jumps one row up on screen while its data position is unchanged. Dragging the bottom note downward makes target === length, which passes the guard; changeItem then does splice(index, 1) followed by splice(length, 0, item), and because the first splice already shortened the array the insert is clamped back to the end - the note does not move, but the offset has again been shifted. Both ends of the list therefore produce a visible one-row jump with no reorder.
- Fix: Hoist the test: const target = index + direction; if (target < 0 || target >= this.modifier.length) { return; } and only then enter animateTo to adjust offsetY, dragRefOffset and call changeItem.

### `HW-02-0036` - The ForEach key is JSON.stringify of a mutable note, so ticking a note's checkbox changes its key and forces the framework to destroy and rebuild that list item

- Category C, severity high, confidence confirmed
- Features: LIFE-06
- Document: `huawei_industry_tree/02_convenient_life/docs/06_sticky_note.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sticky_note-0000002275644713
- Why: Section C of the review covers rendering defects. Every checkbox tick rewrites the key of that row, so ArkUI tears the ListItem down and builds a new one instead of updating it: the .transition(TransitionEffect.OPACITY) replays, the .attributeModifier binding is re-established, and any drag or swipe state on that row is lost mid-interaction. The item already has the unique id the guidance asks for, so the cost is paid for nothing.
- Fix: Replace the key generator with item.id.toString(). Consider also replacing Math.random() in the StickyNoteInfo constructor with a monotonically increasing counter, so two notes created in the same tick can never collide.

### `HW-02-0043` - The expanded detail rows are a second List nested inside a List item with no main-axis size, the case the List reference tells you to solve with ListItemGroup

- Category C, severity high, confidence confirmed
- Features: LIFE-07
- Document: `huawei_industry_tree/02_convenient_life/docs/07_collapse_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/collapse_list-0000002248773270
- Why: Section C of the review covers UI construction defects. This is the exact configuration the reference warns about, three deep: the inner List loads every detail row, the outer List loads every bill row, and neither can virtualise. The scenario is a bill history, which grows without bound, so the page materialises the entire dataset on open. A two-level list with a header row and a body is precisely what ListItemGroup exists for.
- Fix: Replace the ListItem/nested-List pair with ListItemGroup({ header: this.dayHeader(item) }) { ForEach(item.children, ...) } inside the outer List, and drop the inner List entirely. Expansion then becomes a matter of rendering the ForEach conditionally inside the group rather than nesting a scroll container.

### `HW-02-0051` - The dialog's ComponentContent is never disposed, and dismissing by tapping the mask bypasses the cleanup entirely, so a module-scope Map grows for the life of the process

- Category B, severity high, confidence confirmed
- Features: LIFE-08
- Document: `huawei_industry_tree/02_convenient_life/docs/08_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule-0000002284438545
- Why: Section B of the review covers resource leaks. Two leaks compound here. Even on the happy path the backend node is never released, because dispose() is never called anywhere in the zip. And because autoCancel defaults to true, tapping outside the dialog closes it without running closeDialog at all, so the mapContext entry - which holds the UIContext and the ComponentContent - survives forever. mapContext is module scope, so nothing ever collects it.
- Fix: Chain the close: config?.context.getPromptAction().closeCustomDialog(config.content).then(() => { config.content.dispose(); mapContext.delete(this.node); }).catch((err: BusinessError) => hilog.error(...)); and pass onWillDismiss (or onDidDisappear) in the BaseDialogOptions so a mask dismissal runs the same cleanup. Replacing mapContext with a field on the component would remove the module-scope map altogether.

### `HW-02-0052` - One TextInputController is bound to two different TextInput components while a second controller is declared and never used

- Category B, severity high, confidence confirmed
- Features: LIFE-08
- Document: `huawei_industry_tree/02_convenient_life/docs/08_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule-0000002284438545
- Why: Section B of the review covers component wiring. A TextInputController drives exactly one TextInput; binding the same instance to two of them leaves the association ambiguous, so any controller call - caretPosition, setTextSelection, stopEditing - acts on whichever component the controller ended up attached to rather than the one the caller meant. The presence of an unused controller2 shows the intent was one controller per field.
- Fix: Change line 141 to controller: this.controller2. If neither field is ever driven programmatically, delete both controllers instead - TextInput does not require one.

### `HW-02-0063` - The LazyForEach key generator is typed as string but receives a DataSource object, so every one of the sixteen entries produces the same key

- Category C, severity high, confidence confirmed
- Features: LIFE-09
- Document: `huawei_industry_tree/02_convenient_life/docs/09_easylife_loopscroll.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_loopscroll-0000002284598853
- Why: Section C of the review covers rendering defects. This is the exact failure the LazyForEach guidance describes, in the component most sensitive to it: the carousel constantly recycles items as they leave and re-enter the viewport, and with one key for sixteen entries the framework has no way to tell which cached component belongs to which datum. Icons can appear against the wrong labels and stale cells can be reused as the strip scrolls.
- Fix: Give DataSource an id assigned when it is built, and key on it - (item: DataSource) => item.id - or, since the data source already renumbers itself as it rotates, key on a per-entry counter stored alongside the item. Do not fall back to the index: the front-remove/back-append rotation changes every index on each cycle.

### `HW-02-0064` - The second seeding loop indexes the first eight elements again, so the data source holds each object twice and eight freshly allocated items are left unconfigured and unused

- Category C, severity high, confidence confirmed
- Features: LIFE-09
- Document: `huawei_industry_tree/02_convenient_life/docs/09_easylife_loopscroll.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_loopscroll-0000002284598853
- Why: Section C of the review covers state handling. The carousel needs sixteen entries so that the front-remove/back-append rotation at TimeScrollView.ets:131-145 always has a full screen of items ahead of the viewport, and it does get sixteen - but only eight distinct objects, each present twice. Two positions in the strip therefore share one object identity, which is what makes the LazyForEach key collision unavoidable, and mutating one visually mutates the other. The eight wasted allocations also show the loop was meant to build a second, independent copy.
- Fix: Replace the three this.dataSource[i] references in the second loop with this.dataSource[i + CommonConstants.ITEM_NUM], or drop the second loop and build all sixteen in one loop that reads its resources with i % CommonConstants.ITEM_NUM.

### `HW-02-0065` - The shared item component has a GridItem root and is reused inside a Scroll, where the reference says GridItem cannot be placed

- Category C, severity high, confidence confirmed
- Features: LIFE-09
- Document: `huawei_industry_tree/02_convenient_life/docs/09_easylife_loopscroll.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_loopscroll-0000002284598853
- Why: Section C of the review covers UI construction defects. The auto-scrolling strip is the headline feature of this document, and its cells are built from a component whose root is a container the reference forbids outside a Grid. The layout attributes GridItemComponent sets on that root - .height(CommonConstants.GRID_ITEM_HEIGHT) - are applied to a node the parent Row does not know how to lay out.
- Fix: Split the shared component: keep GridItemComponent with its GridItem root for MenuItemComponent, and extract the inner Column (image over text) into a ScrollItemComponent that TimeScrollView renders directly, without the GridItem wrapper.

### `HW-02-0068` - postQuerySync runs only on the authentication success path, so a cancelled or failed authentication leaves the pre-query session open

- Category A, severity high, confidence confirmed
- Features: LIFE-10
- Document: `huawei_industry_tree/02_convenient_life/docs/10_asset_verification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/asset_verification-0000002257074702
- Why: Section A of the review requires paired APIs to be used as the reference defines them. preQuerySync allocates a challenge and an authentication session in the asset service; postQuerySync is what releases it. Cancelling the fingerprint prompt - the most common outcome after a mistaken tap - leaves that session allocated with no handle left in the application to close it, and every subsequent cancelled attempt allocates another.
- Fix: Move the call out of the success branch: Asset.getAuthToken(this.challenge, this.context, async (isSuccess, authToken) => { try { if (isSuccess) { ...query and navigate... } } finally { Asset.postQueryAssetPromise(this.challenge); } }); and make sure the callback is invoked for every authentication outcome (see the result-code finding).

### `HW-02-0069` - The authentication result handler covers only two of the documented outcomes, so a failed or cancelled authentication invokes no callback at all and the query stalls silently

- Category B, severity high, confidence confirmed
- Features: LIFE-10
- Document: `huawei_industry_tree/02_convenient_life/docs/10_asset_verification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/asset_verification-0000002257074702
- Why: Section B of the review covers unhandled paths. A wrong fingerprint (12500001), a tap on cancel (12500003), a timeout or a locked-out sensor all reach onResult and match neither branch, so nothing happens: no toast, no navigation, no postQuerySync. From the user's point of view the query button is dead, and the asset session opened by preQuerySync is orphaned.
- Fix: Replace the else-if chain with a default: if (result.result === userAuth.UserAuthResultCode.SUCCESS) { callback(true, result.token); } else { Asset.promptAction(messageFor(result.result), context); callback(false, new Uint8Array(0)); } and use the userAuth.UserAuthResultCode enum members rather than numeric literals.

### `HW-02-0082` - The 代码下载 entry points at a Gitee source tree instead of a sample-code archive, so this is the only document in the industry with no downloadable sample project

- Category E, severity high, confidence confirmed
- Features: LIFE-12
- Document: `huawei_industry_tree/02_convenient_life/docs/12_perpetual_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/perpetual_calendar-0000002319250845
- Why: Section E of the review requires a correct sample-project mapping. The section is titled 代码下载 ("code download") in every document and everywhere else yields a versioned archive that matches the 工程目录 tree printed above it. Here it yields a branch of a live repository, so there is nothing pinned to the document: the tree in section 工程目录 cannot be checked against anything, the 实现思路 snippets cannot be traced to a file, and the contents can change under the document at any time. It also means this scenario alone cannot be reviewed against its code.
- Fix: Publish the Calendar template as a zip under alliance-communityfile-drcn.dbankcdn.com like the sibling documents and link that. If the Gitee repository must remain the source of truth, pin the link to a tag or commit rather than tree/main, and state the commit in 约束与限制 so the printed tree and snippets can be matched to a fixed revision.

### `HW-02-0089` - The snap-back comparison has an operator-precedence bug: i - 1 * DAY_COLUMN_LENGTH evaluates as i - 110, not (i - 1) * 110

- Category B, severity high, confidence confirmed
- Features: LIFE-13
- Document: `huawei_industry_tree/02_convenient_life/docs/13_3-day_view_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/3-day_view_calendar-0000002330222589
- Why: Section B of the review covers logic defects. For every i in 0..9 the expression i - 110 is between -110 and -101, so xOffset >= i - 110 is true for any non-negative scroll offset and the branch degenerates into its second condition alone. The guard that was supposed to ask whether the finger stopped past the previous column boundary asks nothing, so the else-if fires on offsets it should not claim and the list snaps to the wrong column.
- Fix: Parenthesise both occurrences - CalendarView.ets:164 and :213 - as (i - 1) * CalendarConstants.DAY_COLUMN_LENGTH, and correct the same line in the 实现思路 step 3 snippet. Extracting the whole block into one shared method would prevent the fix being applied to only one of the two copies.

### `HW-02-0090` - The snap arithmetic uses a fixed 110 vp column pitch while the columns are sized as 30 percent of the list, so the alignment is correct only on one screen width

- Category C, severity high, confidence confirmed
- Features: LIFE-13
- Document: `huawei_industry_tree/02_convenient_life/docs/13_3-day_view_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/3-day_view_calendar-0000002330222589
- Why: Section C of the review covers layout defects. 30 percent of a list equals 110 vp only when the list is about 366 vp wide, which is roughly one phone. On a wider device each column is wider than 110 vp, so after a swipe the computed offset lands part-way into a column and the three-day view is permanently misaligned - and because the same constant drives both the header list and the body list, the two also drift apart from each other by a growing multiple of the error.
- Fix: Give the ListItem a fixed width of CalendarConstants.DAY_COLUMN_LENGTH so the layout and the arithmetic share one number, or capture the real column width with onAreaChange on the first ListItem and snap to multiples of that. Fold the 32 vp header margin and the 150px leading Blank into the same measurement rather than leaving them out of it.

### `HW-02-0091` - The hour-row height of the timeline and the 52-unit-per-hour scale of the schedule cards are independent magic numbers, and the code calls the unit px while ArkUI reads it as vp

- Category C, severity high, confidence confirmed
- Features: LIFE-13
- Document: `huawei_industry_tree/02_convenient_life/docs/13_3-day_view_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/3-day_view_calendar-0000002330222589
- Why: Section C of the review covers layout defects. A day-view calendar is correct exactly when the card offsets and the hour rules use the same scale, and here the two are produced by unrelated expressions that only a hand-tuned coincidence keeps close. Any change to the timeline font size, the divider thickness or the padding moves the rules without moving the cards. The px-versus-vp confusion compounds it: the comment states a pixel scale while the value reaches the layout as vp, so on any device whose density is not 1 the intended and actual scales differ by the density factor.
- Fix: Add static readonly HOUR_ROW_HEIGHT = 52 to CalendarConstants, set the timeline row height explicitly with .height(CalendarConstants.HOUR_ROW_HEIGHT) instead of .padding(19), and replace the three literal 52s in LayoutCalculator with that constant. Correct the two comments to say vp, or convert with vp2px if a pixel scale is genuinely intended.

### `HW-02-0100` - The location permission check returns success as soon as either permission is granted, so precise location is never requested when only the approximate one was allowed

- Category B, severity high, confidence confirmed
- Features: LIFE-14
- Document: `huawei_industry_tree/02_convenient_life/docs/14_map_bind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_bind-0000002297622922
- Why: Section B of the review covers logic defects and missing authorization flows. The user-facing dialog lets the user grant approximate location while refusing precise location, which is the common case on a map screen. The loop then sees APPROXIMATELY_LOCATION granted, returns true, and the application never asks for LOCATION - so geoLocationManager.getCurrentLocation() at MapHome.ets:90 and the my-location button silently work at reduced accuracy or fail, with nothing in the code able to tell the difference.
- Fix: Invert the loop so a single denial fails the whole check: for (let permission of permissions) { if (await this.checkAccessToken(permission) !== abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED) { return false; } } return true;

### `HW-02-0101` - The runtime permission request discards authResults and logs success even when the user denies, so the map proceeds as if it had been authorised

- Category B, severity high, confidence confirmed
- Features: LIFE-14
- Document: `huawei_industry_tree/02_convenient_life/docs/14_map_bind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_bind-0000002297622922
- Why: Section B of the review covers missing authorization flows. The message logged on a flat refusal is the same one logged on a grant, and nothing downstream re-checks: MapHome.ets:81-82 goes on to call setMyLocationEnabled(true) and setMyLocationStyle regardless. The user is left with a my-location button that does nothing and no explanation, and the application has no way to route them to Settings.
- Fix: Take the result and test it: .then((data: PermissionRequestResult) => { const denied = data.authResults.some((r: number) => r !== 0); if (denied) { /* tell the user and offer atManager.requestPermissionOnSetting(this.context, permissions) */ } }) and only enable the my-location features once every entry is 0.

### `HW-02-0102` - The my-location icon is set to a string path that must resolve under resources/rawfile, but the project has no rawfile directory and no such file

- Category A, severity high, confidence confirmed
- Features: LIFE-14
- Document: `huawei_industry_tree/02_convenient_life/docs/14_map_bind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_bind-0000002297622922
- Why: Section A of the review requires API parameters to be used as the reference defines them. The path is resolved against a folder that does not exist, so the custom my-location marker cannot load - the one visual difference this style object was written to produce is silently absent, and the surrounding anchorU/anchorV/radiusFillColor settings that only make sense with a custom icon are wasted.
- Fix: Add the asset under entry/src/main/resources/rawfile/ and keep the relative path, or switch to the typed form - icon: $r('app.media.my_location') - which the reference has accepted since version 5.0.0(12) and which the compiler can check.

### `HW-02-0107` - The safe-area padding is read with a plain AppStorage.get during build, but EntryAbility writes those values inside loadContent's callback, so the page is padded with zero and never corrected

- Category C, severity high, confidence confirmed
- Features: LIFE-15
- Document: `huawei_industry_tree/02_convenient_life/docs/15_select_type_document.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/select_type_document-0000002340295529
- Why: Section C of the review covers state handling and rendering. AppStorage.get returns a snapshot and creates no dependency, so the first build reads undefined - DisplayUtil converts that to 0 - and nothing schedules a rebuild when the real insets arrive a moment later. The application is in full-screen layout mode, so the registration form is left drawn underneath the status bar and the navigation bar for the whole session, with the first field and the agree button partly obscured.
- Fix: Declare them on the component - @StorageProp('topRectHeight') topRectHeight: number = 0; and @StorageProp('bottomRectHeight') bottomRectHeight: number = 0; - and use those in the padding expression. DisplayUtil's two AppStorage helpers can then be deleted; its display-size helpers are unaffected.

### `HW-02-0108` - The sheet is reopened without awaiting the close that precedes it, and the shared ComponentContent is updated while the old sheet may still be on screen

- Category B, severity high, confidence confirmed
- Features: LIFE-15
- Document: `huawei_industry_tree/02_convenient_life/docs/15_select_type_document.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/select_type_document-0000002340295529
- Why: Section B of the review flags promises that should be awaited but are not. closeBindSheet and openBindSheet both act on the same ComponentContent node, so the second call is issued while the first is still in flight and the node's content is mutated in between. The ordering the code depends on - the old sheet is fully gone before the new content is attached and shown - is never established, and the failure mode is a sheet that shows the previous level's list or does not appear at all.
- Fix: Have both helpers return their promises - static closeBindSheet(): Promise<void> { return PromptActionClass.ctx.closeBindSheet(...); } - and sequence the callers: await PromptActionClass.closeBindSheet(); this.contentNode.update(...); PromptActionClass.setOptions(...); await PromptActionClass.openBindSheet();

### `HW-02-0113` - The two user-grant location permissions are declared in module.json5 and documented, but the sample never requests them at runtime

- Category D, severity high, confidence confirmed
- Features: LIFE-16
- Document: `huawei_industry_tree/02_convenient_life/docs/16_direction_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/direction_demo-0000002317152262
- Why: Section D of the review covers missing permission request flows. Both permissions are user-grant, so declaring them grants nothing - the reference makes the runtime step explicit: "The application calls requestPermissionsFromUser() to display a dialog box to request the foreground location permission from the user." Nothing in this sample ever shows that dialog, so the two location permissions are permanently denied, and the empty catch blocks around every Map Kit call (see the separate finding) hide the resulting failures.
- Fix: Add a checkAccessToken pass over this.permissions in aboutToAppear and, when any is not granted, call requestPermissionsFromUser and inspect data.authResults - every entry must be 0 before searchByText and the navi calls are attempted. MapBindSheet.zip#entry/src/main/ets/pages/MapHome.ets:128-152 is the shape to copy, with the AND fix noted there.

### `HW-02-0114` - Every network call in the sample is wrapped in an empty catch, and the document republishes one of them

- Category B, severity high, confidence confirmed
- Features: LIFE-16
- Document: `huawei_industry_tree/02_convenient_life/docs/16_direction_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/direction_demo-0000002317152262
- Why: Section B of the review flags empty catch blocks. These are not incidental - they sit around every remote call the feature makes. A geocoding lookup that finds nothing, a routing request that is rejected because Map Kit was not enabled in AppGallery Connect, or a call that fails because the location permissions were never requested all produce the same outcome: nothing is drawn, nothing is logged and nothing is shown to the user. Given that the same sample never requests its permissions, the empty catches are what make that defect invisible.
- Fix: Log in each - hilog.error(DOMAIN, TAG, 'searchByText failed: code %{public}d, message %{public}s', (error as BusinessError).code, (error as BusinessError).message); - and in getRoutes also surface the failure through getUIContext().getPromptAction().showToast so the empty route list is explained. Correct the snippet at document lines 37-38 at the same time.

### `HW-02-0115` - Markers and polylines are added on every route request and never removed, so overlays accumulate on the map for the life of the page

- Category B, severity high, confidence confirmed
- Features: LIFE-16
- Document: `huawei_industry_tree/02_convenient_life/docs/16_direction_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/direction_demo-0000002317152262
- Why: Section B of the review covers resource leaks. Switching between driving, cycling and walking is the sample's central interaction, and each switch adds two more markers and one more polyline while the previous ones stay on the map. After a few taps the start and end pins are stacked several deep and every previously requested route is still drawn, so the map shows a tangle of overlapping lines and the user cannot tell which one is current. The very first start marker is worse: it is created in mapLoad at the hard-coded default position 2.922865, 101.58584 and is never moved or removed, so a pin sits in Malaysia regardless of what the user searched for.
- Fix: Remove the previous overlay before every add - the handles are already stored in this.startMarker, this.endMarker and this.mapPolyline, so guard each add with a release of the existing value and clear the field. Drop the addMarker in the mapLoad handler entirely: it places a pin at a default coordinate that has nothing to do with the user's search.

### `HW-02-0121` - The feedback loop between the two lists is broken with a fixed 500 ms timeout against a smooth scroll of unknown duration, so a long jump lets the left list snap to a category the scroll is merely passing

- Category C, severity high, confidence confirmed
- Features: LIFE-17
- Document: `huawei_industry_tree/02_convenient_life/docs/17_dual_list_linkage.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/dual_list_linkage-0000002319372516
- Why: Section C of the review covers UI defects. scrollToIndex with animation reports no completion and its duration depends on the distance travelled - the sample's own list spans eight categories, each carrying a nine-cell grid, so a jump from the first to the last is far longer than a jump to the next. When the animation outlasts 500 ms, isScrollUpdate is restored while the list is still moving, onScrollIndex fires for an intermediate index, and the left selection lands on a category the user did not choose. Below 500 ms the flag stays false long after the scroll has settled, so a genuine user scroll in that window is ignored.
- Fix: Clear the flag from the list's own onScrollStop rather than a timer: keep this.isScrollUpdate = false in the tap handler, and set it back to true in .onScrollStop() on the level-2 List. Delete the setTimeout and the dead this.isScrollUpdate = true at line 168.

### `HW-02-0127` - getCalendarRange emits a range only when the month changes, so the final month in the data never gets a page and its weeks are shown under the previous month's label

- Category B, severity high, confidence confirmed
- Features: LIFE-18
- Document: `huawei_industry_tree/02_convenient_life/docs/18_calendar_swiper.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calendar_swiper-0000002355470245
- Why: Section B of the review covers logic defects. The month Swiper is built directly from this array - CalendarSwiper.zip#entry/src/main/ets/pages/CalendarPage.ets:138 iterates this.monthRanges - so the last month of any dataset has no page of its own and its days appear under the preceding month's heading. With a dataset ending in several complete August weeks rather than one partial one, all of them would be absorbed into July's page.
- Fix: Add a flush after the for loop: if (data.length > 0) { res.push({ startWeekIndex: startWeekIndex, endWeekIndex: data.length, weekViewIndex: startWeekIndex, month: currentMonth }); } before the week-view conversion at line 55.

### `HW-02-0128` - Switching to week view rewrites an entry of the precomputed range array in place, permanently corrupting the data source both Swipers render from

- Category C, severity high, confidence confirmed
- Features: LIFE-18
- Document: `huawei_industry_tree/02_convenient_life/docs/18_calendar_swiper.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calendar_swiper-0000002355470245
- Why: Section C of the review covers state handling. The array is precomputed reference data describing the calendar's structure, and the view-switch path edits one of its elements to carry a different month's bounds. Nothing restores it, so after a few week/month switches on different dates several entries of weekRanges describe months rather than weeks. The variable is even named monthRange while the WEEK branch that mutates it is indexing weekRanges, which is what disguises the mistake.
- Fix: Do not mutate the array element. Keep a separate @State expandRange: CalendarViewRange for the animation, computed from the selected day, and pass that to the CalendarView when a week-to-month expansion is in progress. If in-place adjustment is kept, recompute both arrays with CalendarUtil.getCalendarRange after each switch so the corruption cannot accumulate.

### `HW-02-0134` - The page dismisses the keyboard from an onTouch on its root container, which touch bubbling fires for every phase of every tap - including taps on the form fields themselves

- Category C, severity high, confidence confirmed
- Features: LIFE-19
- Document: `huawei_industry_tree/02_convenient_life/docs/19_parcel_address_text_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_text_recognition-0000002326921238
- Why: Section C of the review covers UI defects. Because the event bubbles, tapping a field to type in it also runs the ancestor's handler, so the input session is ended at the moment the user is trying to start one. The handler fires again on every TouchType.Move and again on TouchType.Up, so a single tap ends the session three times. The whole point of the screen is filling in five text fields, which is the interaction this breaks.
- Fix: Take the event and test the phase - .onTouch((event: TouchEvent) => { if (event.type === TouchType.Down) { inputMethod.getController().stopInputSession().catch(...); } }) - and stop the propagation from the inputs themselves, or move the handler onto a background sibling that the fields do not sit inside.

### `HW-02-0135` - One TextInputController is bound to all five TextInput components on the form

- Category B, severity high, confidence confirmed
- Features: LIFE-19
- Document: `huawei_industry_tree/02_convenient_life/docs/19_parcel_address_text_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_text_recognition-0000002326921238
- Why: Section B of the review covers component wiring. A TextInputController drives exactly one TextInput; binding one instance to five leaves the association ambiguous, so any controller call - caretPosition, setTextSelection, stopEditing - acts on whichever component the controller ended up attached to rather than the one the caller meant. On a five-field address form, which is the whole screen, that is the difference between a working focus flow and an arbitrary one.
- Fix: Declare five controllers and pass each to its own field, or - since nothing calls into the controller - drop the parameter entirely from all five TextInputs. TextInput does not require one.

### `HW-02-0136` - Neither the pasteboard read nor the entity extraction has a rejection handler, and the try that surrounds the extraction can only catch a synchronous throw

- Category B, severity high, confidence confirmed
- Features: LIFE-19
- Document: `huawei_industry_tree/02_convenient_life/docs/19_parcel_address_text_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_text_recognition-0000002326921238
- Why: Section B of the review flags promises whose rejections are not handled. Both calls are the feature: the pasteboard read fails when the clipboard is empty or holds no text, and getEntity is a model-backed service that can fail for its own reasons. As written the failure is silent - the fields have already been cleared at Index.ets:143-148 before the read is attempted, so the user is left with an empty form, no message, and nothing in the log.
- Fix: Chain a catch onto each - systemPasteboard.getData().then(...).catch((err: BusinessError) => { hilog.error(...); showToast(...); }) and textProcessing.getEntity(...).then(...).catch((err: BusinessError) => { ... }) - and drop the try, which cannot see either rejection.

### `HW-02-0143` - The document has no 权限说明 section although the sample requests location permissions, unlike both other Map Kit documents in this industry

- Category E, severity high, confidence confirmed
- Features: LIFE-20
- Document: `huawei_industry_tree/02_convenient_life/docs/20_confirm_direction_in_map_to_rotate_and_move.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/confirm_direction_in_map_to_rotate_and_move-0000002335234358
- Why: Section E of the review covers missing mandatory sections and incorrect permission lists. A reader following this document has no indication that the feature needs any permission at all, and the zip's module.json5 does not tell them either - so the omission in the document and the omission in the configuration reinforce each other. Every other scenario in this industry that touches location documents it.
- Fix: Add the section after 说明, in the same form as 16_direction_demo.md lines 72-76, and note that the two location permissions are user-grant and therefore need both a module.json5 declaration and a runtime request.

### `HW-02-0144` - Three nested setInterval timers are held only in local variables, and the innermost polls a flag that the page's own teardown stops updating - so leaving the page mid-move leaves a 100 ms timer running forever

- Category B, severity high, confidence confirmed
- Features: LIFE-20
- Document: `huawei_industry_tree/02_convenient_life/docs/20_confirm_direction_in_map_to_rotate_and_move.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/confirm_direction_in_map_to_rotate_and_move-0000002335234358
- Why: Section B of the review flags lifecycle cleanup problems. Leaving the page while the camera is animating removes the cameraIdle listener, so cameraMoveFinished can never become true, the inner interval never reaches its clearInterval, and it keeps firing every 100 ms for the life of the process - touching this.targetMarker and this.startMarker on a destroyed component. Because the handle is a local of a callback that has already returned, nothing can stop it. The outer 1000 ms timer and the 3000 ms one from onPageShow leak the same way if the page is left before they fire.
- Fix: Store the three handles in private fields, clear each in aboutToDisappear before releaseMapEventListener, and give the inner poll a bounded retry count so it cannot spin when the completion signal stops arriving. The onPageShow timer clears itself on its first tick, so it should be a setTimeout rather than a setInterval.

### `HW-02-0150` - Every image selection leaks a file descriptor, an ImageSource and the previous PixelMap - loadImage opens and decodes but releases nothing

- Category B, severity high, confidence confirmed
- Features: LIFE-21
- Document: `huawei_industry_tree/02_convenient_life/docs/21_parcel_address_image_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_image_recognition-0000002343383049
- Why: Section B of the review flags resource leaks. fileIo.open returns a File holding a native descriptor that must be closed explicitly, and both ImageSource and PixelMap hold decoded image memory. The screen invites the user to try several photos, and each attempt adds one descriptor plus one decoded bitmap that nothing can reclaim - a full-resolution camera photo is several megabytes, so a handful of retries is enough to matter, and the process eventually reaches its descriptor limit.
- Fix: Release first, then open in a try/finally: this.chooseImage?.release(); this.imageSource?.release(); const fileSource = await fileIo.open(name, fileIo.OpenMode.READ_ONLY); try { this.imageSource = image.createImageSource(fileSource.fd); this.chooseImage = await this.imageSource.createPixelMap(); } finally { fileIo.closeSync(fileSource); } and release both again in the page teardown.

### `HW-02-0151` - The text-recognition callback handles only the success code and the try around it cannot see an asynchronous failure, so a failed OCR is completely silent

- Category B, severity high, confidence confirmed
- Features: LIFE-21
- Document: `huawei_industry_tree/02_convenient_life/docs/21_parcel_address_image_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_image_recognition-0000002343383049
- Why: Section B of the review covers unhandled paths and error handling. An OCR failure - an unreadable photo, an uninitialised engine, an unsupported image - leaves this.recognizeText holding whatever the previous run produced, with no toast, no log and no visual change. The catch that was written to cover exactly that case can never fire, because a callback-style API signals failure through its first callback argument, not by throwing.
- Fix: Handle both outcomes in the callback - if (error && error.code !== 0) { hilog.error(0x0000, 'OCR', 'recognizeText failed: %{public}d %{public}s', error.code, error.message); showToast($r('app.string.recognition_fail')); return; } this.recognizeText = data.value; - and drop the surrounding try, which cannot observe the failure.

### `HW-02-0152` - aboutToDisappear is declared async and awaits inside, which the lifecycle guidance says keeps the component alive in the promise closure

- Category B, severity high, confidence confirmed
- Features: LIFE-21
- Document: `huawei_industry_tree/02_convenient_life/docs/21_parcel_address_image_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_image_recognition-0000002343383049
- Why: Section B of the review covers lifecycle cleanup. The await captures the component in the closure of the release promise, so the node that ArkUI has just de-referenced is held alive until the OCR engine finishes shutting down - and this page also holds an undisposed PixelMap and ImageSource, so what is being kept alive is not small. The paired aboutToAppear at Index.ets:58-61 is async for the same reason and has the same effect on startup.
- Fix: Drop the async and the await: aboutToDisappear(): void { textRecognition.release().then(() => { hilog.info(...); }).catch((err: BusinessError) => { hilog.error(...); }); } - the promise then closes over nothing but the logger rather than over the component.

### `HW-02-0158` - The scan page builds the camera preview with the XComponent constructor that was deprecated in API 12, while the project targets API 20

- Category A, severity high, confidence confirmed
- Features: LIFE-22
- Document: `huawei_industry_tree/02_convenient_life/docs/22_id_card_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/id_card_scan-0000002338766518
- Why: The sample is the reference implementation developers copy for custom camera preview. Shipping the deprecated id-based overload on an API 20 project propagates a construct the platform has already superseded, and the id parameter it forces developers to invent has no effect on an ArkTS-side surface.
- Fix: Drop the id field and use the XComponentOptions overload introduced in API 12: XComponent({ type: XComponentType.SURFACE, controller: this.mXComponentController }). Nothing else in the sample reads the component id.

### `HW-02-0159` - A full-resolution PixelMap is decoded for every preview frame and none of them is ever released

- Category B, severity high, confidence confirmed
- Features: LIFE-22
- Document: `huawei_industry_tree/02_convenient_life/docs/22_id_card_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/id_card_scan-0000002338766518
- Why: recognizeText is started on every frame delivered by the ImageReceiver (CameraService.ets:352-361), so the page decodes a new full-resolution bitmap several times per second and hands none of them back. The PixelMap is never attached to an Image component, so nothing else can free it either; the memory grows for as long as the scan page stays open.
- Fix: Release the bitmap once the recognition chain is finished: attach .finally(() => pixelMap.release()) to the recognizeText promise, or await recognizeText and call pixelMap.release() afterwards so both the success and the failure path free it.

### `HW-02-0160` - The preview buffer is decoded without ever comparing the row stride with the image width, which the ImageReceiver guide requires

- Category B, severity high, confidence confirmed
- Features: LIFE-22
- Document: `huawei_industry_tree/02_convenient_life/docs/22_id_card_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/id_card_scan-0000002338766518
- Why: The producer of the preview stream is free to pad every row to a hardware alignment, so on those devices stride is larger than width. Decoding the padded buffer as if it were tightly packed shears the image progressively, and the OCR step the whole feature depends on then reads a skewed frame and never matches the ID card.
- Fix: Follow the guide: read imgComponent.rowStride, and when it differs from the frame width copy each row into a tightly packed Uint8Array before creating the image source (method 1 of the guide), or decode at width = stride and crop with pixelMap.cropSync (method 2).

### `HW-02-0161` - Two early returns in the frame callback skip nextImage.release(), so failing frames are never handed back to the ImageReceiver

- Category B, severity high, confidence confirmed
- Features: LIFE-22
- Document: `huawei_industry_tree/02_convenient_life/docs/22_id_card_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/id_card_scan-0000002338766518
- Why: The receiver is created with a capacity of eight buffers (CameraService.ets:140, image.createImageReceiver(this.previewProfileObj.size, image.ImageFormat.JPEG, 8)). Each frame that leaves through one of the two early returns keeps its buffer checked out permanently; after eight such frames the receiver has no free buffer left and imageArrival stops firing, so scanning dies silently with the preview still running.
- Fix: Call nextImage.release() before each early return, or restructure the callback so the release happens once in a tail that every path reaches.

### `HW-02-0162` - releaseCamera never releases the ImageReceiver nor unsubscribes imageArrival, so every visit to the scan page leaks a receiver and a listener

- Category B, severity high, confidence confirmed
- Features: LIFE-22
- Document: `huawei_industry_tree/02_convenient_life/docs/22_id_card_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/id_card_scan-0000002338766518
- Why: A user who backs out of the scan page without a successful read - the ordinary failure case - leaves behind a receiver holding eight full-resolution buffers plus a live imageArrival subscription that still points at the old CameraService closure. Opening the scan page again stacks another one on top, so the leak grows with every retry.
- Fix: Extend releaseCamera with the same try/catch/finally shape used for the other four handles: unsubscribe with this.imageReceiver?.off('imageArrival'), await this.imageReceiver?.release(), then set this.imageReceiver = undefined.

### `HW-02-0163` - The recognised ID card text, the extracted entities and the final name plus ID number are all written to hilog as public plaintext

- Category D, severity high, confidence confirmed
- Features: LIFE-22
- Document: `huawei_industry_tree/02_convenient_life/docs/22_id_card_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/id_card_scan-0000002338766518
- Why: This is the one sample in the industry whose entire subject is a national identity document. Marking the payload {public} defeats the exact mechanism the platform provides for personal data, so the citizen's full name and ID number end up in the device log in clear text where any log capture, bug report or crash dump picks them up.
- Fix: Remove the three payload logs, or reduce them to non-identifying facts (whether a name and a number were found, and their lengths). If a value must be traced during development, route it through a format string that uses the default {private} identifier so the platform filters it.

### `HW-02-0171` - The sample wires CardRecognition through the callback parameter that its own source comment marks as deprecated since API 19, on a project that targets API 20

- Category A, severity high, confidence confirmed
- Features: LIFE-23
- Document: `huawei_industry_tree/02_convenient_life/docs/23_card_information_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_information_recognition-0000002377837925
- Why: The sample ships the superseded parameter while carrying a comment that tells the reader it is superseded, so the code and its own documentation disagree on the page a developer is meant to copy. The industry's other CardRecognition sample already uses onResult, so the two official samples now teach two different call shapes for the same control.
- Fix: Rename the parameter to onResult and delete the comment. The callback body does not change; only the property name does.

### `HW-02-0172` - A comment forbidding the printing of personal information is followed six lines later by two log calls that print the whole recognised card

- Category D, severity high, confidence confirmed
- Features: LIFE-23
- Document: `huawei_industry_tree/02_convenient_life/docs/23_card_information_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_information_recognition-0000002377837925
- Why: The document's opening scenario is real-name authentication with an ID card, a passport or a residence permit. Serialising the whole recognition result writes the holder's name, document number and image URI into the device log in clear text, and the sample states in its own comment that this must not happen - so the defect is not an oversight about what is sensitive, it is the code contradicting its own stated rule on the page every reader copies.
- Fix: Delete both log lines. If the callback needs tracing, log only params.code and params.cardType, which lines 59 and 63 already do.

### `HW-02-0173` - The confirmation page shows a hardcoded name placeholder and a hardcoded 18-digit number instead of the recognised values it was handed

- Category B, severity high, confidence confirmed
- Features: LIFE-23
- Document: `huawei_industry_tree/02_convenient_life/docs/23_card_information_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_information_recognition-0000002377837925
- Why: The confirmation page is the whole point of the feature - the user is meant to check what was recognised before submitting it. It shows a fixed asterisk string and a fixed eighteen-digit number that belongs to no one, identically for every document scanned, so the recognition result is never actually presented and the submit button confirms fabricated data. Anyone copying this page ships a real-name flow that verifies nothing.
- Fix: Bind the two Text components to the consumed model: Text(this.cardData.name) and Text(this.cardData.id). If a masked display is intended, mask this.cardData.id at render time instead of substituting a literal.

### `HW-02-0174` - The avoidAreaChange listener registered in onWindowStageCreate is never unsubscribed

- Category B, severity high, confidence confirmed
- Features: LIFE-23
- Document: `huawei_industry_tree/02_convenient_life/docs/23_card_information_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_information_recognition-0000002377837925
- Why: The subscription outlives the window stage it was created for, and its closure keeps writing into AppStorage after the UI that reads those keys is gone. The ability can be recreated within the same process, and each recreation registers another listener on top of the ones that were never removed.
- Fix: Keep the window reference on the ability and call windowClass.off('avoidAreaChange') from onWindowStageDestroy, matching the subscription made in onWindowStageCreate.

### `HW-02-0183` - Every order calls createCalendar with the same account name and never queries first, which the Calendar Kit guide says creates the calendar repeatedly

- Category B, severity high, confidence confirmed
- Features: LIFE-24
- Document: `huawei_industry_tree/02_convenient_life/docs/24_appointment_service_remind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/appointment_service_remind-0000002378138101
- Why: addCalendarEvent runs once per placed order, so the guide's warning applies on the second order and every order after it. The user's calendar app accumulates 'Appointment' accounts, and each new reminder lands in whichever one the latest create returned, so the events the application wrote earlier are no longer in the calendar it is now writing to.
- Fix: Query first and fall back to creation: 'let calendar = await this.calendarMgr.getCalendar(account).catch(() => this.createCalendar(account));' - or keep the resolved Calendar on the utility after the first call instead of re-resolving it per event.

### `HW-02-0184` - The today slot list is built by matching the current hour plus two against an exact interval start hour, so before 06:00 and after 18:00 the whole day collapses to one row

- Category B, severity high, confidence confirmed
- Features: LIFE-24
- Document: `huawei_industry_tree/02_convenient_life/docs/24_appointment_service_remind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/appointment_service_remind-0000002378138101
- Why: Between 00:00 and 05:59 the entire 08:00-20:45 service day is still ahead of the user, and the picker offers none of it - only the 'within two hours' row, which at 03:00 means a 03:00-05:00 pickup. Between 19:00 and 20:44 the same collapse happens. The bug is the exact-equality match: an hour that is not itself an interval boundary can never set the found flag, so the loop pushes nothing.
- Fix: Replace the equality with a comparison: 'NORMAL_TIME_INTERVAL.forEach((timeInterval) => { if (timeInterval.startTime.hour >= startHour) { this.timePreview.push(timeInterval); } });'. Clamp startHour to EARLIEST_TIME.hour so early-morning openings show the full day.

### `HW-02-0185` - A @Link declared without undefined is bound to a @State that is declared and initialised as undefined, which the state guidance says must match

- Category C, severity high, confidence confirmed
- Features: LIFE-24
- Document: `huawei_industry_tree/02_convenient_life/docs/24_appointment_service_remind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/appointment_service_remind-0000002378138101
- Why: The child declares that the value is always present and then immediately guards against it being absent, so the declared type is known to be false where it is written. Every read of this.selectedParam in the child is unchecked apart from that one guard, and the sample is the page a developer copies for the parent-child two-way binding pattern.
- Fix: Declare the union on the child to match the source - '@Link selectedParam: SelectedTimeParam | undefined;' - or initialise the parent's @State with 'new SelectedTimeParam('', 0, 0)' so the non-optional type is true on both sides and the guard can go.

### `HW-02-0186` - The avoidAreaChange listener registered in onWindowStageCreate is never unsubscribed

- Category B, severity high, confidence confirmed
- Features: LIFE-24
- Document: `huawei_industry_tree/02_convenient_life/docs/24_appointment_service_remind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/appointment_service_remind-0000002378138101
- Why: The listener closure captures mWindow and the UIContext and keeps writing into AppStorage after the window stage it belongs to has been destroyed. Because the subscription is made inside the loadContent callback, the window handle it uses is not stored on the ability either, so there is nothing left to unsubscribe with once onWindowStageCreate has returned.
- Fix: Keep the window on the ability - 'this.mWindow = windowStage.getMainWindowSync();' - and call this.mWindow?.off('avoidAreaChange') from onWindowStageDestroy.

### `HW-02-0198` - The avoidAreaChange listener is registered on a window handle that is local to a callback, and is never unsubscribed

- Category B, severity high, confidence confirmed
- Features: LIFE-25
- Document: `huawei_industry_tree/02_convenient_life/docs/25_commuting_calculation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/commuting_calculation-0000002347813672
- Why: The subscription outlives the window stage and keeps writing into AppStorage after the pages that read topRectHeight and bottomRectHeight are gone. Because the handle is a local inside the getMainWindow callback, it is unreachable from onWindowStageDestroy, so there is nothing left to unsubscribe with even if the call were added there.
- Fix: Store the window on the ability - 'this.windowClass = data;' - and unsubscribe in onWindowStageDestroy with this.windowClass?.off('avoidAreaChange').

### `HW-02-0212` - The outlet list key generator is annotated as taking a string but receives an OutletInfo object and returns it as the key

- Category C, severity high, confidence confirmed
- Features: LIFE-26
- Document: `huawei_industry_tree/02_convenient_life/docs/26_list_of_nearby_outlets.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/list_of_nearby_outlets-0000002347931784
- Why: The annotation is false - at runtime the parameter is an OutletInfo, not a string - so the key handed to the framework is an object rather than the unique string the key generation contract requires, and the type annotation hides the mismatch from the compiler. The list is rebuilt from scratch on every search (:251 sets nearbyOutlets to []) and grows by push inside two loops, which is exactly the mutable-data-source case the guidance says needs a stable unique id.
- Fix: Carry site.siteId - which the place search reference defines as the mandatory 'Unique ID of a place' - into OutletInfo, and key on it: '}, (item: OutletInfo) => item.siteId);'.

### `HW-02-0224` - One SwiperController is bound to two Swipers with different page counts, and changeIndex is called with an index from the wrong one

- Category C, severity high, confidence confirmed
- Features: LIFE-27
- Document: `huawei_industry_tree/02_convenient_life/docs/27_easylife_demo_vrhouse.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_demo_vrhouse-0000002349301462
- Why: changeIndex is called with a photo index but the same controller also addresses the category Swiper, whose maximum page index is smaller - with displayCount(5) and three categories that Swiper has a single page, so any photoId above 0 is out of range and the documented fallback silently sends it to page 0. Tapping the third category tab, which computes photoId 3 for the first house, therefore cannot reliably land on the third category's first image, and the two Swipers are being told to move by one call that means different things to each.
- Fix: Give each Swiper its own controller - a photoCtr bound to the image Swiper and an indicatorCtr bound to the tab Swiper - and call changeIndex on the photo controller only. The category tab already updates categoryId directly, so the indicator does not need programmatic paging at all.

### `HW-02-0225` - The tab index is bound two-way with $$ and read for the tab bar highlight, but it is not decorated as state

- Category C, severity high, confidence confirmed
- Features: LIFE-27
- Document: `huawei_industry_tree/02_convenient_life/docs/27_easylife_demo_vrhouse.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_demo_vrhouse-0000002349301462
- Why: The guidance ties the UI update to the variable being decorated, and this one is not. Switching tabs writes the new index back through $$, but the tab bar that reads it is built from an undecorated field, so the blue highlight and the bold label have no reason to move off the first tab. This is the whole visible state of the bottom bar, and the sample is the page a reader copies for the $$ pattern.
- Fix: Decorate it: '@State tabId: number = 0;'. Nothing else has to change - the $$ binding and the three reads are already correct.

### `HW-02-0226` - The provided UIContext is declared as possibly undefined while all three consumers declare it as always present

- Category C, severity high, confidence confirmed
- Features: LIFE-27
- Document: `huawei_industry_tree/02_convenient_life/docs/27_easylife_demo_vrhouse.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_demo_vrhouse-0000002349301462
- Why: The provider's type says the value may be absent and the consumers' types say it never is, which is exactly the mismatch the guidance warns produces application exceptions. Every consumer then dereferences it without a check - HouseImagePage.ets:157, HouseInfoPage.ets:218 and HomeComponent.ets:296 all call this.uiContext.px2vp(...) directly in their build methods.
- Fix: Make the two sides agree. The cleanest fix is to drop the union on the provider by assigning at declaration where the context is available, or to declare the union on all three consumers and guard the px2vp calls.

### `HW-02-0227` - The avoidAreaChange listener registered in onWindowStageCreate is never unsubscribed

- Category B, severity high, confidence confirmed
- Features: LIFE-27
- Document: `huawei_industry_tree/02_convenient_life/docs/27_easylife_demo_vrhouse.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_demo_vrhouse-0000002349301462
- Why: The listener outlives the window stage it was created for and keeps writing topRectHeight and bottomRectHeight into AppStorage after the pages that read them through @StorageLink are gone. Because the window handle is a local in onWindowStageCreate, it is unreachable from onWindowStageDestroy, so the unsubscribe cannot even be added without first storing the window on the ability.
- Fix: Keep the window on the ability - 'this.windowClass = windowStage.getMainWindowSync();' - and call this.windowClass?.off('avoidAreaChange') from onWindowStageDestroy.

### `HW-02-0235` - javaScriptProxy is registered and never unregistered, which the Web reference says is required to prevent memory leaks

- Category B, severity high, confidence confirmed
- Features: LIFE-28
- Document: `huawei_industry_tree/02_convenient_life/docs/28_h5rechargeplatform.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5rechargeplatform-0000002387265697
- Why: The registration creates a bridge between the web engine and an ArkTS object, and the reference states plainly that the pairing with deleteJavaScriptRegister is what prevents the leak. This page is the whole sample - it is the one file a reader copies for the javaScriptProxy pattern - and it demonstrates only half of the pair the documentation defines.
- Fix: Add the teardown to the page: 'aboutToDisappear(): void { this.webViewController.deleteJavaScriptRegister('contacts'); }', using the same name string passed to javaScriptProxy.

### `HW-02-0236` - The contact result is indexed twice without any length check, behind a non-null assertion, inside the method exposed to the web page

- Category B, severity high, confidence confirmed
- Features: LIFE-28
- Document: `huawei_industry_tree/02_convenient_life/docs/28_h5rechargeplatform.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5rechargeplatform-0000002387265697
- Why: Two ordinary outcomes throw. If the user dismisses the picker without choosing anyone the array is empty and [0] is undefined, so reading .phoneNumbers off it raises a TypeError; if the chosen contact has a name but no stored number, phoneNumbers is an empty array and [0].phoneNumber does the same. The method is registered through javaScriptProxy, so the throw surfaces inside the web page's await with no ArkTS-side log, and the trailing ! is what stops the compiler from pointing any of this out.
- Fix: Destructure and check: 'let contacts = await contact.selectContacts(contactSelectionOptions); let number = contacts?.[0]?.phoneNumbers?.[0]?.phoneNumber; this.phoneNumber = number ?? '';' and wrap the whole method body in try/catch so a picker failure returns an empty string rather than rejecting into the page.

### `HW-02-0237` - The avoidAreaChange listener registered in onWindowStageCreate is never unsubscribed

- Category B, severity high, confidence confirmed
- Features: LIFE-28
- Document: `huawei_industry_tree/02_convenient_life/docs/28_h5rechargeplatform.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5rechargeplatform-0000002387265697
- Why: The listener outlives the window stage and keeps writing into AppStorage after the page that reads those keys with @StorageProp is gone. The handle is a local inside onWindowStageCreate, so it is not reachable from onWindowStageDestroy and the unsubscribe cannot be added without first storing the window on the ability.
- Fix: Store the window - 'this.windowClass = windowStage.getMainWindowSync();' - and call this.windowClass?.off('avoidAreaChange') from onWindowStageDestroy.

### `HW-02-0247` - Three map listeners are registered and the page has no teardown of any kind

- Category B, severity high, confidence confirmed
- Features: LIFE-29
- Document: `huawei_industry_tree/02_convenient_life/docs/29_set_coverage_area.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_coverage_area-0000002547690109
- Why: Each callback closes over the page instance and over this.mapCircle, this.pointAnnotation, this.marker and this.mapController, so the whole page graph stays reachable from the map's listener registry after the page is gone. The markerDrag callback in particular fires per frame and calls back into the page to add and remove map overlays, so a stale registration keeps mutating a map on behalf of a destroyed page.
- Fix: Add the teardown the sibling sample demonstrates: 'aboutToDisappear(): void { this.mapEventManager?.off('markerDragStart'); this.mapEventManager?.off('markerDrag'); this.mapEventManager?.off('mapLongClick'); }', registering all three through the event manager as the guide shows.

### `HW-02-0248` - The circle and annotation are redrawn by an unawaited async pair, so overlapping drag frames leave orphaned overlays on the map

- Category B, severity high, confidence confirmed
- Features: LIFE-29
- Document: `huawei_industry_tree/02_convenient_life/docs/29_set_coverage_area.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_coverage_area-0000002547690109
- Why: A second call entering while the first is still awaiting addCircle reads this.mapCircle before the first has assigned to it, so it removes the already-removed overlay and then overwrites the field with its own. Whatever the first call finally assigns is drawn on the map and no longer referenced by anything, so it can never be removed - during a drag, which fires per frame, discarded circles and annotations accumulate. The remove calls at :104 and :127 also sit outside the try/catch that wraps only the add, so removing an already-removed overlay throws where nothing catches it.
- Fix: Serialise the redraw: keep the previous overlay in a local, await the add, assign the field, then remove the local - and guard re-entry with an in-flight flag so a drag frame arriving mid-redraw is skipped rather than interleaved.

### `HW-02-0254` - The executor attaches no rejection handler, and because the success handler is the only place a task leaves the running list, a failed task permanently consumes a concurrency slot

- Category B, severity high, confidence confirmed
- Features: LIFE-30
- Document: `huawei_industry_tree/02_convenient_life/docs/30_thumbnail_taskpool.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/thumbnail_taskpool-0000002338966908
- Why: Every rejection leaves its task in runningList forever, so the executor loses one concurrency slot each time. The sample runs the thumbnail pool with a maximum of three (FileItem.ets:47, 'TaskManager.getInstance().execute(task, 3, 1000)'), so three failed or cancelled thumbnails are enough for canExecute to return false permanently and no thumbnail is ever generated again for the life of the process. The same handler is also the only caller of startTask after the first admission, so a rejection additionally stops the queue from draining.
- Fix: Add the rejection branch and share the cleanup: '.catch((err: BusinessError) => { Logger.error(TAG, `task failed. Code: ${err.code}`); }).finally(() => { this.removeTaskFromRunningList(task); this.startTask(); });' - moving the removal and the restart out of the success path so both outcomes free the slot.

### `HW-02-0255` - The wait-list overflow rule uses slice with a count where an end index belongs, and discards the task it has just accepted

- Category B, severity high, confidence confirmed
- Features: LIFE-30
- Document: `huawei_industry_tree/02_convenient_life/docs/30_thumbnail_taskpool.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/thumbnail_taskpool-0000002338966908
- Why: The queue is last-in-first-out by design, so the entry at the end is the one that would have run next - and that is exactly the entry the overflow rule removes. Every task submitted once the queue is full is accepted, counted and then silently dropped, while the oldest entries that the comment says should be discarded stay in the list. The truncation also loses one more item than intended, since it takes 999 of 1000 slots.
- Fix: Drop the second argument so the tail is kept: 'this.waitList = this.waitList.slice(startIndex);' - or, equivalently, 'this.waitList = this.waitList.slice(-this.maxWaitLimit);'.

### `HW-02-0256` - The LazyForEach key generator is annotated as taking a string but receives a FileModel and returns it as the key

- Category C, severity high, confidence confirmed
- Features: LIFE-30
- Document: `huawei_industry_tree/02_convenient_life/docs/30_thumbnail_taskpool.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/thumbnail_taskpool-0000002338966908
- Why: The annotation is false at runtime, which is what stops the compiler from catching it, and the value handed to the framework is an object rather than the unique persistent string the key contract requires. This is the list the whole sample exists to render, its contents are replaced wholesale on every query, and each row also carries a lazily generated thumbnail that the framework matches to a component by key.
- Fix: Key on the file path, which is unique per row and stable across reloads: '}, (item: FileModel) => item.path);'.

### `HW-02-0257` - The thumbnail worker decodes a full-size PixelMap and two ImageSources per image and releases none of them

- Category B, severity high, confidence confirmed
- Features: LIFE-30
- Document: `huawei_industry_tree/02_convenient_life/docs/30_thumbnail_taskpool.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/thumbnail_taskpool-0000002338966908
- Why: This function is the entire point of the feature and it runs once per file in a batch - the document's scenario is 'a long list containing a large number of images'. Each run holds a full-resolution bitmap plus two image sources until the worker's garbage collector happens to reclaim them, and the intermediate full-size pixelmap is provably dead the moment packToData returns. The kept thumbnail is displayed through an Image component and needs no manual release, but nothing else in the function is.
- Fix: Release each object as soon as it is finished with: the first ImageSource after createPixelMap, the full-size pixelmap after packToData, and the second ImageSource after the thumbnail is decoded - in a finally so a failed pack does not skip them.

### `HW-02-0002` - Three directory names in the documented project tree do not match the directories that actually exist in the sample zip

- Category E, severity medium, confidence confirmed
- Features: LIFE-01
- Document: `huawei_industry_tree/02_convenient_life/docs/01_practice-convenient-life-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1-0000001952539489
- Why: Section E of the review requires the documented project tree to match the actual zip. HarmonyOS module resolution is case-sensitive when strictMode.caseSensitiveCheck is enabled, and entry/src/main/ets/pages/Index.ets:22 imports the deep path 'mine/src/main/ets/page/MyPage', so a reader who follows the documented spelling gets an unresolved-module build failure rather than a cosmetic difference.
- Fix: Correct lines 188, 232 and 241 of the 代码结构解读 tree to the exact directory names used in Life_Framework_Code_V1.zip.

### `HW-02-0003` - The CardRecognition snippet in the document opens a file descriptor with fs.openSync and never closes it, leaking one fd per scan

- Category B, severity medium, confidence confirmed
- Features: LIFE-01
- Document: `huawei_industry_tree/02_convenient_life/docs/01_practice-convenient-life-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1-0000001952539489
- Why: fs.openSync returns a File object holding a native file descriptor that must be released explicitly; without fs.closeSync every card scan leaks one descriptor, and the process eventually hits its fd limit. The sample project itself demonstrates the correct pattern, so the document is publishing a strictly worse version of its own code.
- Fix: Wrap the body after fs.openSync in try { ... } finally { fs.closeSync(file); }, exactly as CredentialsPage.ets:178-189 does.

### `HW-02-0008` - The login flow swallows encryption failures in an empty catch block, so a failed login silently leaves the user record unset

- Category B, severity medium, confidence confirmed
- Features: LIFE-01
- Document: `huawei_industry_tree/02_convenient_life/docs/01_practice-convenient-life-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1-0000001952539489
- Why: Section B of the review flags empty catch blocks. When encryption fails, nothing is logged, no message reaches the user, and AppStorage.setAndRef is never reached, so the 'login' entry is never written - yet line 216 still clears the navigation stack and dismisses the login page. entry/src/main/ets/pages/Index.ets:163-167 then reads AppStorage.ref('login') on the next tap of the 我的 tab, finds nothing, and pushes the login page again with no explanation.
- Fix: Log the error inside the catch (hilog.error(DOMAIN, 'LoginPage', 'encrypt failed: %{public}s', err.message)), show a toast, and move RouterModule.clear into the success branch so the page is dismissed only after the login state has actually been written.

### `HW-02-0009` - hilog.info is called with the wrong argument order in AssetStore.ets, so the error payload is used as the format string and no log tag is emitted

- Category A, severity medium, confidence confirmed
- Features: LIFE-01
- Document: `huawei_industry_tree/02_convenient_life/docs/01_practice-convenient-life-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1-0000001952539489
- Why: Section A of the review requires API calls to match the official signature. Here the tag slot receives the literal '%{public}s' and the format slot receives the serialized BusinessError, so the placeholder is gone, the JSON is printed verbatim with no privacy annotation, and the log carries no usable tag - the failure of asset.add becomes untraceable.
- Fix: Move the tag into the second argument and keep the format string with its placeholder in the third: hilog.error(DOMAIN, 'AssetStore', 'asset.add failed: %{public}s', JSON.stringify(err)). At EntryAbility.ets:27-28, use one placeholder per argument, e.g. hilog.info(DOMAIN, 'EntryAbility', 'onCreate want=%{public}s', JSON.stringify(want)).

### `HW-02-0011` - The documented project tree names the entry page Index.ets, but the sample ships pages/MainPage.ets

- Category E, severity medium, confidence confirmed
- Features: LIFE-03
- Document: `huawei_industry_tree/02_convenient_life/docs/03_vehicle_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vehicle_keyboard-0000002236257106
- Why: Section E of the review requires the project tree in the document to match the actual zip. The page route string is what EntryAbility and main_pages.json both key on, so a reader who creates pages/Index.ets by following the tree gets a blank window at startup.
- Fix: Change line 93 of 工程目录 to read MainPage.ets, and consider renaming the struct in MainPage.ets from Index to MainPage so the file and the struct agree.

### `HW-02-0012` - The step-1 snippet claims to hide the TextInput but omits every attribute that actually hides it

- Category E, severity medium, confidence confirmed
- Features: LIFE-03
- Document: `huawei_industry_tree/02_convenient_life/docs/03_vehicle_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vehicle_keyboard-0000002236257106
- Why: Section E of the review requires code snippets to match the zip. The omitted lines are the entire mechanism the surrounding sentence promises: copy the snippet as printed and a full-size, fully visible system TextInput sits under the licence-plate boxes. The reader has no way to derive the four attributes from the text.
- Fix: Extend the 实现思路 step 1 snippet with the four hiding attributes and maxLength(7), and note that the visible plate is drawn by the Row of Stack cells above it, not by the TextInput.

### `HW-02-0014` - setWindowLayoutFullScreen returns a promise that is neither awaited nor caught, and the avoid area is read on the next line before the immersive layout has been applied

- Category B, severity medium, confidence confirmed
- Features: LIFE-03
- Document: `huawei_industry_tree/02_convenient_life/docs/03_vehicle_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vehicle_keyboard-0000002236257106
- Why: Section B of the review flags promises that should be awaited but are not. Dropping the promise means a rejection becomes an unhandled rejection with no log, and the avoid-area query on the following line races the layout change it depends on. The same file's sibling sample in this industry does chain a .catch (LIFE-01, EntryAbility.ets:51-56).
- Fix: windowClass.setWindowLayoutFullScreen(true).then(() => { /* read both avoid areas into AppStorage here */ }).catch((err: BusinessError) => { hilog.error(0x0000, 'EntryAbility', 'setWindowLayoutFullScreen failed: %{public}s', JSON.stringify(err)); });

### `HW-02-0016` - The delete key is placed with a hard-coded pixel offset inside a fractional-column Grid, so it misaligns on any screen width other than the one it was tuned for

- Category C, severity medium, confidence confirmed
- Features: LIFE-03
- Document: `huawei_industry_tree/02_convenient_life/docs/03_vehicle_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vehicle_keyboard-0000002236257106
- Why: Section C of the review covers layout defects. .position() takes the GridItem out of the grid flow and pins it at a fixed vp offset, so the rowStart/rowEnd/columnStart/columnEnd values on the same item are dead. Every other key scales with the '1fr' columns; the delete key does not. On a width other than the phone the numbers were tuned for - and the HAR declares tablet and 2in1 in its deviceTypes - the delete key lands away from its cell. The two keyboards even need different magic numbers (153 vs 164) because they have 10 and 7 columns.
- Fix: Drop the offset field from the three delete-key entries and remove .position(item.offset) from Keyboard.ets:39; give the delete key the grid span it should occupy - for the 10-column province and numeric keyboards that is position: [3, 3, 8, 9] on the last row - and let the fractional columns place it.

### `HW-02-0021` - Both 实现思路 snippets drop the OFFSET_X/OFFSET_Y compensation, and the drawing snippet also drops the seat outline, so the printed code paints seats two vp off and with no border

- Category E, severity medium, confidence confirmed
- Features: LIFE-04
- Document: `huawei_industry_tree/02_convenient_life/docs/04_canvas_cinema.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_cinema-0000002272398929
- Why: Section E of the review requires code snippets to match the zip. The two-vp inset is not decoration: the canvas is deliberately four vp larger than the seat grid so the 0.5 vp outline is not clipped at the edges. The printed drawing code has no outline at all, so a 可售 (available) seat - which is white on a white canvas - would be invisible, and the printed hit-test disagrees with the printed drawing code by two vp.
- Fix: Copy lines 77-86 of CinemaInfo.ets verbatim into the step-2 snippet, and add - StyleConstants.OFFSET_X to the step-1 snippet.

### `HW-02-0022` - A new ImageBitmap is constructed on every seat tap and never closed, leaking one decoded image per selection

- Category B, severity medium, confidence confirmed
- Features: LIFE-04
- Document: `huawei_industry_tree/02_convenient_life/docs/04_canvas_cinema.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_cinema-0000002272398929
- Why: Section B of the review flags resource leaks. Every tap decodes the same PNG again and drops the handle with its graphics resources still held; the seat grid is 14 x 10, so a user filling a row leaks a dozen bitmaps in a few seconds, and the page holds them until it is destroyed.
- Fix: Declare it as a component field - private selectedImg: ImageBitmap = new ImageBitmap('resources/base/media/selected.png'); - use this.selectedImg in drawImage, and close it in aboutToDisappear alongside the existing dialogController cleanup.

### `HW-02-0023` - EntryAbility subscribes to avoidAreaChange but never unsubscribes

- Category B, severity medium, confidence confirmed
- Features: LIFE-04
- Document: `huawei_industry_tree/02_convenient_life/docs/04_canvas_cinema.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_cinema-0000002272398929
- Why: Section B of the review flags subscriptions that are opened but never closed. The callback holds a closure that writes into AppStorage and keeps the window object referenced after the stage is torn down.
- Fix: Store the window handle and the callback in fields of EntryAbility and call this.windowClass?.off('avoidAreaChange', this.avoidAreaCallback) in onWindowStageDestroy.

### `HW-02-0024` - The 清空 (clear) action resets confirmed and paid seats back to available, contradicting the app's own rule that a confirmed seat cannot be chosen

- Category C, severity medium, confidence confirmed
- Features: LIFE-04
- Document: `huawei_industry_tree/02_convenient_life/docs/04_canvas_cinema.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_cinema-0000002272398929
- Why: Section C of the review covers incorrect state mutation. Seats in state 2 represent completed purchases, not the current selection; clearing them repaints them white and lets the same seat be sold twice. The button's own label 清空 and its position next to the selected-seat list place it in the selection area, so the user has no way to know it also erases their paid seats.
- Fix: Replace the double loop with a pass over the selection: this.selectedSeats.forEach((s: number[]) => { this.drawRect(s[0], s[1], 0); this.seats[s[0]][s[1]] = 0; }); this.count = 0; this.selectedSeats = [];

### `HW-02-0028` - The two select-all handlers decrement the running total once per item when deselecting, without the guard they apply when selecting

- Category B, severity medium, confidence confirmed
- Features: LIFE-05
- Document: `huawei_industry_tree/02_convenient_life/docs/05_cascading_menu_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cascading_menu_selection-0000002274600681
- Why: Section B of the review covers state that can drift. totalCount is the number the bottom bar renders (BottomBar.ets:19) and it is maintained by hand rather than derived from selectedItems.length. Because the decrement is unconditional while removeByName is conditional, the counter and the list can only stay in step as long as every reachable path guarantees that the whole subtree was selected - an invariant nothing in the code enforces, and one that any new entry point into the deselect branch breaks silently and irreversibly, since the counter never resynchronises.
- Fix: Guard both deselect loops - if (thirdItem.isSelected === Constants.SELECTED) { thirdItem.isSelected = Constants.NOT_SELECTED; this.removeByName(thirdItem.name); this.totalCount += Constants.COUNT_DECREASE; } - or drop the totalCount field entirely and render this.selectedItems.length, which cannot drift.

### `HW-02-0029` - The first menu level is exempted from the @Observed/@ObjectLink pattern the document says the sample demonstrates: PrimaryMenuItem is a plain interface and the child binds it with @State

- Category C, severity medium, confidence confirmed
- Features: LIFE-05
- Document: `huawei_industry_tree/02_convenient_life/docs/05_cascading_menu_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cascading_menu_selection-0000002274600681
- Why: Section C of the review covers state handling. isAll on a first-level item is written but can never be observed, and @State in a child that is initialised from the parent takes the value once and does not resynchronise - so the first level is wired differently from the two levels below it for no stated reason. The sample gets away with it only because PrimaryMenu renders nothing but name; adding the same check mark the second and third levels have would produce a control that never updates.
- Fix: Convert PrimaryMenuItem from an interface to an @Observed class with a constructor, build it in DataManager alongside SecondMenuItem and ThirdMenuItem, and change PrimaryMenuItem.ets to @ObjectLink primaryMenu: PrimaryMenuItem;.

### `HW-02-0037` - The two animation snippets call a bare getUIContext(), a module-local helper that reads the UIContext out of AppStorage, without showing or naming it

- Category E, severity medium, confidence confirmed
- Features: LIFE-06
- Document: `huawei_industry_tree/02_convenient_life/docs/06_sticky_note.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sticky_note-0000002275644713
- Why: Section E of the review requires snippets to be usable against the zip. As printed, the snippets look like they call a framework global. A reader who copies onMove and onDrop into a plain class gets an unresolved identifier; a reader who assumes it is the component method gets a compile error, because ListExchangeCtrl is not a component and has no getUIContext.
- Fix: Add the four-line helper and the aboutToAppear seeding to the 实现思路 section, or - better - give ListExchangeCtrl a uiContext: UIContext constructor parameter and call this.uiContext.animateTo(...), which removes the AppStorage round trip entirely.

### `HW-02-0038` - The add-note dialog initialises its own controller field with a self-referential CustomDialogController instead of leaving it optional for the framework to inject

- Category C, severity medium, confidence probable
- Features: LIFE-06
- Document: `huawei_industry_tree/02_convenient_life/docs/06_sticky_note.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sticky_note-0000002275644713
- Why: Section C of the review covers component wiring. The 取消 and 确认 buttons at StickyNotePage.ets:207-219 both call this.controller.close(); if the framework does not overwrite an already-initialised field, that call closes a dialog that was never opened and the visible one stays on screen. Even where the injection does win, the initializer allocates a second controller and a second TextInputDialog builder closure per dialog instance, for nothing.
- Fix: Change the declaration to controller?: CustomDialogController; and guard both handlers: if (this.controller !== undefined) { this.controller.close(); ... }. Also clear this.content in the confirm handler, otherwise the next open shows the previous note's text.

### `HW-02-0039` - EntryAbility subscribes to avoidAreaChange but never unsubscribes

- Category B, severity medium, confidence confirmed
- Features: LIFE-06
- Document: `huawei_industry_tree/02_convenient_life/docs/06_sticky_note.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sticky_note-0000002275644713
- Why: Section B of the review flags subscriptions that are opened but never closed. The callback holds a closure writing into AppStorage and keeps the window referenced after the stage is torn down.
- Fix: Keep the window handle and the callback in fields of EntryAbility and release the listener in onWindowStageDestroy.

### `HW-02-0040` - The page parks the live UIContext in AppStorage under a global key so that a plain model class can reach animateTo

- Category B, severity medium, confidence confirmed
- Features: LIFE-06
- Document: `huawei_industry_tree/02_convenient_life/docs/06_sticky_note.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sticky_note-0000002275644713
- Why: Section B of the review covers lifecycle cleanup and resource leaks. AppStorage lives for the whole application, so the UIContext of this page - and through it the window and the entire component tree it belongs to - is kept reachable after the page is destroyed. The key is also global and untyped, so a second page doing the same thing overwrites it and every ListExchangeCtrl in the application starts animating against the wrong window.
- Fix: Add a uiContext: UIContext parameter to the ListExchangeCtrl constructor, pass this.getUIContext() from the page, and replace every getUIContext()?.animateTo with this.uiContext.animateTo. If the AppStorage route is kept, delete the key in the page's aboutToDisappear.

### `HW-02-0044` - None of the five ForEach calls supplies a keyGenerator, so the default key embeds the mutable isExpand flag and the index

- Category C, severity medium, confidence confirmed
- Features: LIFE-07
- Document: `huawei_industry_tree/02_convenient_life/docs/07_collapse_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/collapse_list-0000002248773270
- Why: Section C of the review covers rendering defects. The default key is both index-based - which the guidance rules out - and content-based over an object whose content changes, so any operation that does re-run the ForEach diff (the sort at BillPage.ets:54, a filter, an appended month) rebuilds every row instead of moving it, discarding each child's aboutToAppear-computed totals. Note the obvious fix is a trap: ListModel.ets:30 declares id?: string, but spend_data.json and income_data.json both number their entries 1, 3, 7, 10..., so item.id alone is ambiguous in the merged billList built at BillPage.ets:53.
- Fix: Use (item: ListModelData) => `${item.type}_${item.id}` for the three bill lists and (item: ChildrenObject) => item.id for the children, or renumber the two rawfiles so ids are unique across both.

### `HW-02-0045` - The expansion flag is held twice - as a local @State and on the @Observed model - and the copy is seeded only once, so the declared @ObjectLink binding drives nothing

- Category C, severity medium, confidence confirmed
- Features: LIFE-07
- Document: `huawei_industry_tree/02_convenient_life/docs/07_collapse_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/collapse_list-0000002248773270
- Why: Section C of the review covers state handling. Because build() never reads this.item.isExpand, the @Observed/@ObjectLink pair the 场景介绍 relies on contributes nothing to the expansion behaviour, and the model field is write-only bookkeeping. aboutToAppear runs once, so anything that changes item.isExpand from outside the row - a collapse-all control, restoring a saved state, the sort at BillPage.ets:54 reordering rows - leaves the arrow and the detail list showing the stale local value with no way to resynchronise.
- Fix: Delete the @State isExpand field and the aboutToAppear assignment; change the tap handler to this.item.isExpand = !this.item.isExpand; and both readers to this.item.isExpand. The same edit applies to SpendDataItem.ets.

### `HW-02-0046` - The only code snippet in the document omits the aboutToAppear line that makes the expansion state survive, and shows an empty Row body for the row it is explaining

- Category E, severity medium, confidence confirmed
- Features: LIFE-07
- Document: `huawei_industry_tree/02_convenient_life/docs/07_collapse_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/collapse_list-0000002248773270
- Why: Section E of the review requires snippets to reflect the zip well enough to be followed. Document line 23 tells the reader to "在父列表项和子列表项中分别设置isExpand变量并初始化为false" ("set an isExpand variable in both the parent list item and the child list item and initialise it to false") - which, taken literally with the printed snippet, produces a row whose expansion is lost every time the list is rebuilt, because nothing reads the value back off the model.
- Fix: Extend the 实现思路 snippet with IncomeDataItem.ets lines 23-28, and state that the model field is what carries the expansion across rebuilds.

### `HW-02-0047` - EntryAbility subscribes to avoidAreaChange but never unsubscribes

- Category B, severity medium, confidence confirmed
- Features: LIFE-07
- Document: `huawei_industry_tree/02_convenient_life/docs/07_collapse_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/collapse_list-0000002248773270
- Why: Section B of the review flags subscriptions that are opened but never closed. The callback holds a closure writing into AppStorage and keeps the window referenced after the stage is torn down.
- Fix: Keep the window handle and the callback in fields of EntryAbility and release the listener in onWindowStageDestroy.

### `HW-02-0053` - Choosing an out-of-order date erases the previously valid value, and the rejected date is written to the model before it is validated

- Category C, severity medium, confidence confirmed
- Features: LIFE-08
- Document: `huawei_industry_tree/02_convenient_life/docs/08_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule-0000002284438545
- Why: Section C of the review covers state handling. The user has already chosen a valid start time; picking a second, invalid one throws the first away, so the only recovery is to re-enter it. The model is left inconsistent as well: startDate holds the rejected value while isStartNull says no start has been chosen, so the two disagree about what the page knows.
- Fix: Test first and return early: if (!this.isEndNull && selectedDate.getTime() >= this.endDate.getTime()) { showToast(...); return; } then assign this.startDate and the two display strings. The same edit applies to the end-date branch.

### `HW-02-0054` - The result callback is routed through @Prop, whose deep copy the reference says preserves only primitives, Map, Set, Date and Array

- Category C, severity medium, confidence probable
- Features: LIFE-08
- Document: `huawei_industry_tree/02_convenient_life/docs/08_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule-0000002284438545
- Why: Section C of the review covers state handling. A function is none of the types the reference lists as surviving the copy, so passing the only channel by which the picker returns its result through @Prop is outside what the decorator guarantees. The failure mode is silent and total - the confirm button would throw or do nothing - and it would appear only on an ArkUI version whose copy implementation drops function properties.
- Fix: Split the callback out of the copied object: declare resultCb as a plain member of DateTimePicker and DateTimePickerDialogComponent - resultCb: Callback<Date> = () => {}; - and pass it alongside data rather than inside it. Keep @Prop for the plain-data fields (startYear, endYear, selectedDate), which the reference does guarantee.

### `HW-02-0055` - The year range lives in a module-scope singleton that every picker instance overwrites, so two pickers with different ranges cannot coexist

- Category A, severity medium, confidence confirmed
- Features: LIFE-08
- Document: `huawei_industry_tree/02_convenient_life/docs/08_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule-0000002284438545
- Why: Section A of the review covers incorrect use of an API surface. The HAR advertises the year span as a per-picker parameter but stores it in shared module state, so mounting a second picker with a different span silently reconfigures the first: its year column still shows the old strings while selected is computed from the new start year, putting the wheel on the wrong entry. The picker is exported from a reusable HAR (dateTimePicker/Index.ets), which is exactly the setting where two instances are likely.
- Fix: Make DateTimeRangeClass instantiable per component - private range: DateTimeRangeClass = new DateTimeRangeClass(); in DateTimePicker - and drop the exported DATE_TIME_RANGE constant, or pass startYear/endYear to yearRange() and getStartYear() as arguments so the helper holds no state at all.

### `HW-02-0056` - setWindowLayoutFullScreen is called without awaiting it inside an async method that awaits everything else, and the avoid areas are read on the next lines

- Category B, severity medium, confidence confirmed
- Features: LIFE-08
- Document: `huawei_industry_tree/02_convenient_life/docs/08_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule-0000002284438545
- Why: Section B of the review flags promises that should be awaited but are not. The dropped promise turns any failure into an unhandled rejection with no log, and the two getWindowAvoidArea calls that follow race the layout change they depend on. Because the values are read once and never refreshed, a wrong reading at startup - or any later rotation or fold - leaves the page padded incorrectly for the rest of the session.
- Fix: Add the await with a try/catch that logs the BusinessError, and register WIN.on('avoidAreaChange', cb) after the initial reads, paired with WIN.off('avoidAreaChange', cb) in onWindowStageDestroy.

### `HW-02-0057` - The document's single snippet is three disconnected fragments with empty bodies, and it shows dateTimePickerDialog as a no-argument function that it is not

- Category E, severity medium, confidence confirmed
- Features: LIFE-08
- Document: `huawei_industry_tree/02_convenient_life/docs/08_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule-0000002284438545
- Why: Section E of the review requires snippets to reflect the zip. The printed dateTimePickerDialog does not compile and, more importantly, hides the two things a reader must get right to use PromptAction for a custom dialog: wrapping the builder in a ComponentContent, and keeping a handle to that node so it can be closed and disposed later. 实现思路 step 2 claims the dialog is shown "通过PromptAction展示" ("displayed through PromptAction") and step 3 that the value comes back "通过传入的resultCb返回" ("returned through the resultCb that was passed in"), but the snippet demonstrates neither end to end.
- Fix: Replace the 实现思路 snippet with DateTimePickerDialog.ets lines 133-143 and NewSchedule.ets lines 47-72 verbatim, and add a fourth step covering closeCustomDialog plus dispose().

### `HW-02-0062` - startAutoRoll overwrites the interval handle without clearing the previous timer, so any scroll-stop without a matching scroll-start leaks a timer

- Category B, severity medium, confidence confirmed
- Features: LIFE-09
- Document: `huawei_industry_tree/02_convenient_life/docs/09_easylife_loopscroll.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_loopscroll-0000002284598853
- Why: Section B of the review covers resource leaks. The code assumes onScrollStart and onScrollStop always alternate. Any path that reaches onScrollStop without a preceding onScrollStart - a programmatic scrollTo settling, a fling that ends after the component has re-entered the tree, a rapid second gesture - starts a second timer while the first is still running, and the handle to the first is lost. Two timers then advance rollOffset twice per period and the carousel visibly doubles speed, with no way to recover short of leaving the page.
- Fix: Begin startAutoRoll with if (this.intervalNum !== CommonConstants.INIT_ZERO) { clearInterval(this.intervalNum); this.intervalNum = CommonConstants.INIT_ZERO; } before assigning the new handle.

### `HW-02-0066` - The document's only snippet is four fragments with comment-filled bodies, and the Swiper it shows omits the ForEach that supplies the pages

- Category E, severity medium, confidence confirmed
- Features: LIFE-09
- Document: `huawei_industry_tree/02_convenient_life/docs/09_easylife_loopscroll.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_loopscroll-0000002284598853
- Why: Section E of the review requires snippets to reflect the zip. As printed, the Swiper renders one page with no data, so the manual-scroll half of the feature cannot be reproduced at all; and the two bodies replaced by comments are exactly the parts a reader cannot derive - how many items to rotate per cycle, why rollOffset is decremented by itemWidth * ITEM_NUM_HALF afterwards, and what offsetRemain has to return to make onScrollFrameBegin wrap the strip.
- Fix: Replace the 实现思路 snippet with HomePage.ets lines 26-38, TimeScrollView.ets lines 128-151 and TimeScrollView.ets lines 202-215 verbatim.

### `HW-02-0070` - The user-authentication result listener is registered but never unsubscribed, and the instance that owns it is dropped when the function returns

- Category B, severity medium, confidence confirmed
- Features: LIFE-10
- Document: `huawei_industry_tree/02_convenient_life/docs/10_asset_verification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/asset_verification-0000002257074702
- Why: Section B of the review flags subscriptions that are opened but never closed. Because the only reference to the instance is the dropped local, the subscription can never be released once the function has returned - the reference requires the same instance to unsubscribe. Every query therefore leaves a live authentication callback holding the UIContext and the caller's closure.
- Fix: Keep the instance in a variable the handler can close over, and unsubscribe as the first thing the handler does: const instance = userAuth.getUserAuthInstance(authParam, widgetParam); const handler: userAuth.IAuthCallback = { onResult: (result) => { instance.off('result', handler); ... } }; instance.on('result', handler); instance.start();

### `HW-02-0071` - A failed or empty query still navigates to the result page, which then renders three undefined fields

- Category B, severity medium, confidence confirmed
- Features: LIFE-10
- Document: `huawei_industry_tree/02_convenient_life/docs/10_asset_verification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/asset_verification-0000002257074702
- Why: Section B of the review covers null and undefined risk. The declared return type Map<string, string> is not honoured on either path, and the caller navigates regardless of the outcome, so a query that failed shows the user a result screen with three blank rows - which reads as "this account has an empty password" rather than "the query failed" - while an empty result set throws inside convertMapArrayToMapItem.
- Fix: Change the return type to Promise<Map<string, string> | undefined> and return accountList.length > 0 ? accountList[0] : undefined, returning undefined from the catch as well. At the call site, guard the navigation: if (!account) { return; } after the postQuery cleanup.

### `HW-02-0072` - The result page seeds its state by reading a parameter from a NavPathStack it just constructed itself, and casts the undefined result to an array

- Category C, severity medium, confidence confirmed
- Features: LIFE-10
- Document: `huawei_industry_tree/02_convenient_life/docs/10_asset_verification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/asset_verification-0000002257074702
- Why: Section C of the review covers state handling. getParamByIndex(0) on a stack with no entries has no parameter to return, and the as Array<AssetItem> cast hides that from the compiler, so this.data is typed non-nullable while holding undefined until onReady replaces it. The initializer is dead work whose only effect is to make the declared type a lie, and any future path that renders before onReady - or a push that carries no parameter - reaches ForEach with undefined.
- Fix: Initialise it as @State data: Array<AssetItem> = []; and keep the assignment in onReady, guarded: const param = context.pathStack.getParamByIndex(0); this.data = (param as Array<AssetItem>) ?? [];

### `HW-02-0076` - Both onActionEnd handlers discard the GestureEvent the reference hands them and branch on a @State that only onActionUpdate writes, which both stales the decision and re-renders the page on every pinch frame

- Category C, severity medium, confidence confirmed
- Features: LIFE-11
- Document: `huawei_industry_tree/02_convenient_life/docs/11_card_pinch_scale.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_pinch_scale-0000002282825670
- Why: Section C of the review covers state handling. Two separate problems follow from routing the ratio through @State. First, correctness: onActionUpdate is documented as "Triggered when the user moves the finger in the pinch gesture on the screen", so a gesture that is recognised at the 5 vp threshold and released without further movement reaches onActionEnd with scaleValue still holding the previous gesture's ratio - and the branch that decides between the card view and the list view then acts on a stale number. Second, cost: @State writes re-render the owning component, so every frame of every pinch rebuilds all of CardIndex, including the five-item horizontal List with chainAnimation, to update a value that nothing displays.
- Fix: Take the parameter and drop the field: .onActionEnd((event: GestureEvent) => { if (event.scale < 1) { this.isSingleTodo = false; } }) on the page gesture and .onActionEnd((event: GestureEvent) => { if (event.scale > 1) { this.isSingleTodo = true; this.selectedItem = item; } }) on the list item, then delete @State scaleValue and both onActionUpdate handlers, which exist only to feed it.

### `HW-02-0077` - The card list is rendered by a ForEach with no key generator over five byte-identical entries, so the framework falls back to an index-based key the guidance rules out

- Category C, severity medium, confidence confirmed
- Features: LIFE-11
- Document: `huawei_industry_tree/02_convenient_life/docs/11_card_pinch_scale.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_pinch_scale-0000002282825670
- Why: Section C of the review covers rendering defects. Because the serialised content of all five entries is the same string, the index is the only thing distinguishing one key from another - which is exactly the case the guidance warns about. Inserting, removing or reordering a card would rebuild every card from that position on rather than moving it, discarding each one's PinchGesture recognition state mid-interaction, and TodoItemType carries no id to key on instead.
- Fix: Add id: string to TodoItemType (CardPinchScale.zip#entry/src/main/ets/common/Model.ets:21-25), give each entry in TODO_LIST a distinct value, and close the ForEach with (item: TodoItemType) => item.id.

### `HW-02-0078` - setWindowLayoutFullScreen is not awaited inside an async method that awaits the line above, and the avoid areas are read immediately after and never refreshed

- Category B, severity medium, confidence confirmed
- Features: LIFE-11
- Document: `huawei_industry_tree/02_convenient_life/docs/11_card_pinch_scale.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_pinch_scale-0000002282825670
- Why: Section B of the review flags promises that should be awaited but are not. The dropped promise turns a failure into an unhandled rejection with no log, and the two getWindowAvoidArea reads on the following lines race the layout change they depend on. Because the values are captured once and never refreshed, a wrong reading at startup - or any later rotation, fold, or switch between gesture and three-button navigation - leaves the page padded incorrectly for the rest of the session, with the content drawn under the system bars.
- Fix: Add the await with a try/catch that logs the BusinessError, and register WIN.on('avoidAreaChange', cb) after the initial reads, paired with WIN.off('avoidAreaChange', cb) in onWindowStageDestroy.

### `HW-02-0079` - Both 实现思路 snippets have empty bodies and omit the three things that make the interaction work: the shared state, where each gesture is attached, and the list attributes that produce the card-deck effect

- Category E, severity medium, confidence confirmed
- Features: LIFE-11
- Document: `huawei_industry_tree/02_convenient_life/docs/11_card_pinch_scale.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_pinch_scale-0000002282825670
- Why: Section E of the review requires snippets to reflect the zip. As printed, this.scaleValue is an undeclared identifier in both fragments, so neither compiles; the empty Column hides that the zoom-out gesture is page-wide while the zoom-in gesture is per card, which is the whole design; and the four List attributes that turn a horizontal list into a snapping card deck appear nowhere in the document, although the 效果预览 animation is entirely about that effect.
- Fix: Replace the two 实现思路 snippets with MainPage.ets lines 59-124 verbatim, and add a sentence naming scrollSnapAlign(ScrollSnapAlign.CENTER) and chainAnimation(true) as the source of the card-deck behaviour.

### `HW-02-0083` - The calendar snippet sets Grid maxCount alongside columnsTemplate, where the reference says it has no effect, and the prose's seven-row maximum is not what the printed algorithm can produce

- Category E, severity medium, confidence confirmed
- Features: LIFE-12
- Document: `huawei_industry_tree/02_convenient_life/docs/12_perpetual_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/perpetual_calendar-0000002319250845
- Why: Section E of the review requires the document's snippets and prose to agree with each other and with the referenced APIs. Two things are wrong at once: the attribute a reader would take as the row cap is inert in the configuration it is printed in, and the number the prose quotes is one more than the algorithm on the same page can ever produce. A reader who trusts either statement will size the calendar container for a row that never appears.
- Fix: Delete .maxCount(7) from the 实现思路 step 1 snippet, and change 最大7行 on line 27 to 最大6行 ("maximum 6 rows"). If a hard cap is genuinely wanted, express it in the data - const totalRows = Math.min(6, Math.max(5, rowsNeeded)) - rather than through a Grid attribute that the columnsTemplate mode ignores.

### `HW-02-0086` - This document declares API Version 16 and SDK 5.0.4 while every other scenario in the industry declares API Version 20 and SDK 6.0.0

- Category F, severity medium, confidence confirmed
- Features: LIFE-12
- Document: `huawei_industry_tree/02_convenient_life/docs/12_perpetual_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/perpetual_calendar-0000002319250845
- Why: Section F of the review covers the same topic being described differently across documents. A reader assembling an application from this industry's scenarios has to satisfy every scenario's floor at once, and this one is four API levels behind the rest with no explanation. Either the sample genuinely predates the others - in which case the document should say so, because its APIs and its project layout are then from a different generation than the other 28 - or the constraint block was not refreshed when the rest of the industry moved to API 20.
- Fix: Re-verify the template against API Version 20 and update lines 127-129, or add a sentence to 约束与限制 stating that this template is maintained on the 5.0.4 baseline and is not aligned with the rest of the industry's samples.

### `HW-02-0087` - The project tree carries three copy-paste defects: base_apis is listed a third time in base_calendar's place, yiji_query's view is named VipCenter.ets, and one entry is a directory containing itself

- Category E, severity medium, confidence confirmed
- Features: LIFE-12
- Document: `huawei_industry_tree/02_convenient_life/docs/12_perpetual_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/perpetual_calendar-0000002319250845
- Why: Section E of the review requires the documented project tree to match the actual project. All three are the signature of block-copying one module's entry into the next: a reader mapping the tree onto the repository finds no base_calendar root, looks for a VIP centre inside the auspicious-day query module, and cannot tell what the nested utils entry is meant to be. With no sample archive attached to this document, the tree is the only description of the layout there is, so an error in it cannot be resolved by looking at the code.
- Fix: Correct the three entries in the 工程目录 tree, and re-derive the tree from the repository rather than editing it by hand - the same block-copy pattern is what produced all three.

### `HW-02-0088` - The core date snippet depends on two third-party libraries that the document never declares, names inconsistently, or versions

- Category E, severity medium, confidence confirmed
- Features: LIFE-12
- Document: `huawei_industry_tree/02_convenient_life/docs/12_perpetual_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/perpetual_calendar-0000002319250845
- Why: Section E of the review covers missing mandatory information. The snippet the document presents as its central algorithm cannot be compiled without adding both packages to oh-package.json5, and the document supplies neither the package names as ohpm would resolve them nor the versions the code was verified against. The lunar mention is also buried in a note about mock data, where a reader looking for dependencies would not find it - and dayjs, which appears in nine of the snippet's lines, is never mentioned at all.
- Fix: Add a 依赖说明 ("dependencies") section listing the ohpm package name and version for dayjs and for the lunar library, and show the corresponding oh-package.json5 dependencies block. Move the lunar mention out of the mock-data note in 说明.

### `HW-02-0092` - Each snap scrolls twice - once inline and once through a @Watch - and when the computed offset repeats, the watcher does not fire and the two lists desynchronise

- Category B, severity medium, confidence confirmed
- Features: LIFE-13
- Document: `huawei_industry_tree/02_convenient_life/docs/13_3-day_view_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/3-day_view_calendar-0000002330222589
- Why: Section B of the review covers state that can drift. @Watch fires only when the value changes, so the design has two different behaviours for the same gesture. When the user swipes to a new column the offset changes, the watcher runs, and both lists are scrolled - twice, since the inline call already moved one of them. When the user swipes and settles back on the column they started from, the assignment writes the same number, the watcher does not fire, and only the list that was touched is repositioned - leaving the date header and the schedule body showing different days until the next successful snap.
- Fix: Delete the inline scrollTo from both onScrollStop handlers and let onCountUpdated own the scrolling, or drop the @Watch and have a single shared snapTo(offset) method scroll both controllers directly. If the watcher is kept, the same-value case still needs handling - reset xOffset to -1 after each snap so the next assignment is always a change.

### `HW-02-0093` - The twenty-line snap block and the whole schedule-card markup are each duplicated verbatim, so every fix has to be applied twice

- Category C, severity medium, confidence confirmed
- Features: LIFE-13
- Document: `huawei_industry_tree/02_convenient_life/docs/13_3-day_view_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/3-day_view_calendar-0000002330222589
- Why: Section C of the review covers construction defects. Both duplicates are load-bearing: the snap logic carries a real bug that a maintainer would naturally fix in one copy and miss in the other, and the card markup means any styling change has to be made twice or the first day of a multi-day event silently diverges from its continuation days. The always-true inner condition also misleads - it reads as though single-day and multi-day events are handled differently there, when the branch has already excluded everything but the first day.
- Fix: Extract private snapToColumn(scroller: Scroller) from the two onScrollStop bodies and call it from both. In ScheduleCard, replace the two branches with a single if (this.schedule.isFirstDay(this.day) || this.schedule.isContainDay(this.day)) around one copy of the markup, and delete the always-true inner test.

### `HW-02-0094` - DayDataSource.pushData appends an array of days but notifies LazyForEach of a single insertion at the last index

- Category B, severity medium, confidence confirmed
- Features: LIFE-13
- Document: `huawei_industry_tree/02_convenient_life/docs/13_3-day_view_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/3-day_view_calendar-0000002330222589
- Why: Section B of the review covers state that can drift. IDataSource notifications are the only channel by which LazyForEach learns what changed; telling it that one item arrived at index 9 when ten arrived at indices 0 through 9 leaves its internal map inconsistent with totalCount(). The sample gets away with it because the single call happens in aboutToAppear before either list has rendered, but any later batch append - which is exactly what a calendar paging into the next month would do - would add rows the lists never build.
- Fix: Emit one notification per element - const start = this.dataArray.length; ... for (let i = 0; i < data.length; i++) { this.notifyDataAdd(start + i); } - or, for a bulk load, call notifyDataReload() once instead.

### `HW-02-0095` - The vertical scroll extent is hard-coded at 120 percent while the timeline renders eighteen hour rows, so most of the day cannot be reached

- Category C, severity medium, confidence confirmed
- Features: LIFE-13
- Document: `huawei_industry_tree/02_convenient_life/docs/13_3-day_view_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/3-day_view_calendar-0000002330222589
- Why: Section C of the review covers layout defects. The scroll extent is expressed as a fraction of the viewport and the content length is a function of the configured hour range; nothing ties the two together. On a viewport of about 600 vp the user can reach roughly 720 vp of a 936 vp timeline, so the last four hours of the day - and any schedule card positioned in them - are unreachable. Widening the hour range makes it worse with no other symptom.
- Fix: Size the Column from the same constants the timeline uses: .height((CalendarConstants.DEFAULT_LAST_HOUR - CalendarConstants.DEFAULT_FIRST_HOUR + 1) * CalendarConstants.HOUR_ROW_HEIGHT), or drop the explicit height and let the Column take its content height.

### `HW-02-0096` - EntryAbility subscribes to avoidAreaChange but never unsubscribes

- Category B, severity medium, confidence confirmed
- Features: LIFE-13
- Document: `huawei_industry_tree/02_convenient_life/docs/13_3-day_view_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/3-day_view_calendar-0000002330222589
- Why: Section B of the review flags subscriptions that are opened but never closed. The callback holds a closure writing into AppStorage and keeps the window referenced after the stage is torn down.
- Fix: Keep the window handle and the callback in fields of EntryAbility and release the listener in onWindowStageDestroy.

### `HW-02-0098` - The document reprints the precedence bug verbatim, names the backup ability file wrongly in the project tree, and shows a snippet whose bindSheet builder cannot resolve as written

- Category E, severity medium, confidence confirmed
- Features: LIFE-13
- Document: `huawei_industry_tree/02_convenient_life/docs/13_3-day_view_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/3-day_view_calendar-0000002330222589
- Why: Section E of the review requires the document to match the zip and to be followable. Republishing the precedence defect propagates it to every reader who copies the snippet, which is the stated purpose of the 实现思路 section. The tree entry breaks a path that module.json5 resolves literally. And the bindSheet snippet omits the builder wrapper and its two arguments, so a reader who copies it gets an unresolved reference rather than the sheet the section promises.
- Fix: Fix line 86 to (i - 1) * CalendarConstants.DAY_COLUMN_LENGTH, change line 129 to EntryBackupAbility.ets, and extend the step 4 snippet with CalendarView.ets lines 82-85 so the builder and its parameters are visible.

### `HW-02-0103` - A TextInputController is passed to the Search component, whose controller parameter is typed SearchController

- Category A, severity medium, confidence confirmed
- Features: LIFE-14
- Document: `huawei_industry_tree/02_convenient_life/docs/14_map_bind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_bind-0000002297622922
- Why: Section A of the review requires parameters to match the official signature. SearchController and TextInputController are distinct types with different method sets - SearchController adds caretPosition, stopEditing and setTextSelection for the search box specifically - so the field is bound to a controller that cannot drive the component it is attached to. It reads as though the search box is programmatically controllable when it is not.
- Fix: Change the declaration to searchController: SearchController = new SearchController(); and pass that, or drop the controller argument entirely - Search does not require one.

### `HW-02-0104` - module.json5 ships the sample author's AGC client ID as metadata, which the Map Kit setup guide says is no longer required for the version this sample targets

- Category E, severity medium, confidence confirmed
- Features: LIFE-14
- Document: `huawei_industry_tree/02_convenient_life/docs/14_map_bind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_bind-0000002297622922
- Why: Section E of the review covers configuration that does not match the documentation. Two things follow. The entry is obsolete for the declared SDK level, so it is dead configuration a reader will copy forward. And the value is a concrete AGC project identifier belonging to whoever built the sample: a reader who builds this project unchanged is shipping an application that identifies itself with someone else's client ID, and the document gives no instruction to substitute their own.
- Fix: Delete the metadata block from entry/src/main/module.json5. If an older baseline must be supported, keep it but set the value to a placeholder and add a step to the 说明 block telling the reader to replace it with the client ID from their own AGC project.

### `HW-02-0105` - The map height is computed from a window height that is filled asynchronously, so the map is laid out at zero height and can be given a negative one

- Category B, severity medium, confidence confirmed
- Features: LIFE-14
- Document: `huawei_industry_tree/02_convenient_life/docs/14_map_bind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_bind-0000002297622922
- Why: Section B of the review covers promises that are not sequenced with the code that depends on them. Nothing orders the window query before the first detent callback. Until the promise resolves the map is laid out at height 0 - the component is present but invisible - and if the detent callback lands first the height becomes 0 - heightVal + 10, a negative number for any detent above 10 vp. The guard at MapHome.ets:206, if (this.heightVal > 0) { return; }, makes it permanent: the block runs exactly once, so a value computed against an unresolved window height is never recomputed.
- Fix: Make getWindowHeight return the promise and await it in aboutToAppear before anything can consume windowHeight, or set mapWindowHeight inside the same then callback using the current heightVal. Dropping the heightVal > 0 guard and recomputing on every detent change would also make the height self-correcting.

### `HW-02-0106` - The document's snippet binds the sheet to a $$this.isShow state variable that does not exist in the sample, and omits the nested sheet that renders the toolbar

- Category E, severity medium, confidence confirmed
- Features: LIFE-14
- Document: `huawei_industry_tree/02_convenient_life/docs/14_map_bind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_bind-0000002297622922
- Why: Section E of the review requires code snippets to match the zip implementation. The two differ in the one argument that decides whether the sheet is dismissible: a literal true means the sheet is always shown and its visibility is not application state, which is the whole point of a permanently docked map panel, whereas $$this.isShow implies a toggle the sample does not have. Copying the document as printed produces an unresolved identifier. Omitting the nested sheet also hides the technique that produces the effect in 效果预览 - a fixed 60 vp toolbar sheet layered under a multi-detent content sheet, both with enableOutsideInteractive.
- Fix: Replace the 实现思路 snippet with MapHome.ets lines 194-215 verbatim and add MapHome.ets lines 338-349 with a sentence explaining that the toolbar is a second, single-detent sheet nested inside the first.

### `HW-02-0109` - The ComponentContent backing the sheet is never disposed, so its backend node outlives the page

- Category B, severity medium, confidence confirmed
- Features: LIFE-15
- Document: `huawei_industry_tree/02_convenient_life/docs/15_select_type_document.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/select_type_document-0000002340295529
- Why: Section B of the review covers resource leaks. The node holds the builder closure and, through it, the page component. PromptActionClass also keeps static references to the same node and to the UIContext (PromptActionClass.ets:28-29), and neither is cleared on teardown, so every visit to the registration page leaves one ComponentContent and one UIContext reachable for the life of the process.
- Fix: Chain the disposal onto the close - PromptActionClass.closeBindSheet().then(() => { this.contentNode.dispose(); }) once closeBindSheet returns its promise - and null out PromptActionClass.contentNode and PromptActionClass.ctx in the same handler.

### `HW-02-0110` - The sheet helper guards its uninitialised statics against null, which is never the value they hold before they are set

- Category B, severity medium, confidence confirmed
- Features: LIFE-15
- Document: `huawei_industry_tree/02_convenient_life/docs/15_select_type_document.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/select_type_document-0000002340295529
- Why: Section B of the review covers null and undefined risk. The guard was written to make the helper safe to call before setContentNode has run, and it does the opposite: the branch is entered, and openBindSheet is invoked on an undefined UIContext with an undefined node and undefined options. The class is a module-scope singleton reachable from anywhere, so any call ordered before the page's aboutToAppear - or after another page has torn down and cleared it - takes that path.
- Fix: Type the statics as optional - static ctx?: UIContext; static contentNode?: ComponentContent<Object>; static options?: SheetOptions; - and guard with a truthiness test that covers all three: if (!PromptActionClass.ctx || !PromptActionClass.contentNode || !PromptActionClass.options) { return; }

### `HW-02-0111` - The row that opens the second-level sheet is identified by its array index rather than by its document-type code

- Category C, severity medium, confidence confirmed
- Features: LIFE-15
- Document: `huawei_industry_tree/02_convenient_life/docs/15_select_type_document.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/select_type_document-0000002340295529
- Why: Section C of the review covers state handling. The index is a property of the current ordering of idTypeMainData, not of the document type. Inserting, removing or reordering a type - which is the one thing a list of national ID document types is likely to need - silently moves the disclosure arrow and the sub-sheet trigger onto a different row, with no compile-time or runtime signal. The four sub-types in idTypeSubData are numbered 040 to 043, so the data already encodes the parent-child relationship the index is standing in for.
- Fix: Replace the constant with static readonly ID_TYPE_CODE_JWRYSFZM: string = '04'; and both tests with item.idTypeCode === Constant.ID_TYPE_CODE_JWRYSFZM and value.selectedIDType.idTypeCode === Constant.ID_TYPE_CODE_JWRYSFZM. Deriving the sub-list as the entries whose code starts with that prefix would remove the second hard-coded relationship as well.

### `HW-02-0112` - Six form fields are declared as state but only one is ever written; the other five TextInputs have neither a two-way binding nor an onChange, so nothing the user types reaches the page

- Category C, severity medium, confidence confirmed
- Features: LIFE-15
- Document: `huawei_industry_tree/02_convenient_life/docs/15_select_type_document.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/select_type_document-0000002340295529
- Why: Section C of the review covers state handling. Declaring the fields as @State states an intent that the bindings do not implement: the values the user types live only inside the component and the page's model stays empty, so any future validation, submit or restore reads blanks. The literal .select(false) has a visible consequence today - the agreement checkbox is re-applied as unchecked on every rebuild of the page, so checking it and then opening the document-type sheet clears it.
- Fix: Use the two-way form on each editable input - TextInput({ text: $$this.idNumber, ... }) - and give the checkbox a state variable: @State isAgreed: boolean = false; with Checkbox().select($$this.isAgreed). Keep idType one-way, since it is driven by the sheet and is intentionally not focusable.

### `HW-02-0116` - The marker icon is given as a bare filename that must resolve under resources/rawfile, but the file is in resources/base/media and the project has no rawfile directory

- Category A, severity medium, confidence confirmed
- Features: LIFE-16
- Document: `huawei_industry_tree/02_convenient_life/docs/16_direction_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/direction_demo-0000002317152262
- Why: Section A of the review requires API parameters to be used as the reference defines them. The path is resolved against a folder that does not exist, so all three markers fall back to the default pin and the asset the sample ships for them is never used. The same mistake appears in the sibling map sample in this industry, where the file is missing altogether.
- Fix: Move icon.png to entry/src/main/resources/rawfile/ and keep the relative path, or change the option to icon: $r('app.media.icon'), which the compiler can check and which resolves the file where it already is.

### `HW-02-0117` - The routing result is indexed at routes[0] with no length check, and the resulting throw is swallowed by the empty catch

- Category B, severity medium, confidence confirmed
- Features: LIFE-16
- Document: `huawei_industry_tree/02_convenient_life/docs/16_direction_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/direction_demo-0000002317152262
- Why: Section B of the review covers null and out-of-range risk. A routing request legitimately returns no routes - no walking path across water, no driving route between the two coordinates, or the 0, 0 fallback the geocoder produces when the search text matches nothing. routes[0] is then undefined, .steps throws, the empty catch discards it, and the user sees an unchanged map with no list and no message. The fallback coordinate makes it reachable from ordinary input rather than only from a service error.
- Fix: Guard before use: if (!result.routes || result.routes.length === 0) { showToast(no route found); return; } and hoist const ROUTE = result.routes[0]; for the two reads. In getStartPosition and getEndPosition, treat an empty sites array as a failure instead of defaulting the latitude and longitude to 0.

### `HW-02-0118` - An async callback is passed to Array.forEach, which discards the promises it returns

- Category B, severity medium, confidence confirmed
- Features: LIFE-16
- Document: `huawei_industry_tree/02_convenient_life/docs/16_direction_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/direction_demo-0000002317152262
- Why: Section B of the review covers promises that should be awaited but are not. The sample works only because the callback body happens to contain no await, so an async function runs synchronously to completion; adding a single awaited call inside the loop would silently move this.points and this.routes to being filled after line 162 has already read them, and forEach would report nothing. The async keyword announces an asynchronous body that the surrounding code has no way to wait for.
- Fix: Drop the async keyword: result.routes[0].steps.forEach((step) => { ... }). If asynchronous work is added later, convert to for (const step of ROUTE.steps) { await ... } inside getRoutes, which is already async.

### `HW-02-0119` - module.json5 ships the sample author's AGC client ID as metadata, which the Map Kit setup guide says is no longer required for the version this sample targets

- Category E, severity medium, confidence confirmed
- Features: LIFE-16
- Document: `huawei_industry_tree/02_convenient_life/docs/16_direction_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/direction_demo-0000002317152262
- Why: Section E of the review covers configuration that does not match the documentation. The entry is obsolete for the declared SDK level, so it is dead configuration a reader will copy forward; and the value is a concrete AGC project identifier belonging to whoever built the sample, so a reader who builds this project unchanged ships an application identifying itself with someone else's client ID. The identical defect appears in the sibling map sample in this industry with a different literal value, which shows it is a template artefact rather than an isolated slip.
- Fix: Delete the metadata block from entry/src/main/module.json5. If an older baseline must be supported, replace the value with a placeholder and add a step to the 说明 block telling the reader to substitute the client ID from their own AGC project.

### `HW-02-0122` - The left list puts the ListItem inside the custom component and attaches the click handler to the custom component, both of which the List reference advises against

- Category C, severity medium, confidence confirmed
- Features: LIFE-17
- Document: `huawei_industry_tree/02_convenient_life/docs/17_dual_list_linkage.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/dual_list_linkage-0000002319372516
- Why: Section C of the review covers UI construction. Both halves of the recommendation are inverted here: the List's direct child is a custom component rather than a ListItem, and .onClick is set on that custom component. The two lists in the same builder are therefore built to two different rules, and the left one relies on behaviour the reference explicitly declines to recommend - which is also why the click target is the whole component rather than the list row.
- Fix: Move the ListItem out of LeftListChild into the ForEach body and attach onClick to it, leaving LeftListChild to render only the Text. That also makes the left list match the right one, which is already built this way.

### `HW-02-0123` - Each row of the right-hand list contains an unsized Grid, the nested-scrollable case the List reference tells you to solve with ListItemGroup

- Category C, severity medium, confidence confirmed
- Features: LIFE-17
- Document: `huawei_industry_tree/02_convenient_life/docs/17_dual_list_linkage.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/dual_list_linkage-0000002319372516
- Why: Section C of the review covers UI construction defects. This is the configuration the reference warns about, and the shape it recommends instead is exactly what the screen is: a category header followed by that category's items. As shipped, all eight categories and all seventy-two grid cells are built up front, and the scrollToIndex linkage that is the point of the sample has to work against a list that cannot virtualise.
- Fix: Replace the ListItem/Grid pair with ListItemGroup({ header: this.categoryHeader(level1data) }) { ForEach(level1data.level2List, ...) } inside the level-2 List, laying the services out with a Flex or a three-column row rather than a nested scrollable.

### `HW-02-0124` - None of the four ForEach calls supplies a key generator, so the default key embeds the isSelect flag that the linkage toggles

- Category C, severity medium, confidence confirmed
- Features: LIFE-17
- Document: `huawei_industry_tree/02_convenient_life/docs/17_dual_list_linkage.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/dual_list_linkage-0000002319372516
- Why: Section C of the review covers rendering defects. Level1Data carries no identifier, so the default key is the only thing available and it changes on every selection change. The repaint currently arrives through @ObjectLink in LeftListChild rather than through a ForEach diff, so the churn is latent - but any operation that does re-run the diff, such as reordering or filtering the categories, rebuilds every row from that point on and discards the level-2 list's scroll position, which is the one piece of state the whole feature depends on.
- Fix: Add id: string to both classes in ListData.ets, assign it while seeding in aboutToAppear, and close the ForEach calls with (level1data: Level1Data) => level1data.id and (level2data: Level2Data) => level2data.id. For the tab row, key on the resource index string.

### `HW-02-0129` - The Swiper key serialises the range object that the view-switch path mutates, so every adjustment destroys and rebuilds the calendar page

- Category C, severity medium, confidence confirmed
- Features: LIFE-18
- Document: `huawei_industry_tree/02_convenient_life/docs/18_calendar_swiper.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calendar_swiper-0000002355470245
- Why: Section C of the review covers rendering defects. Adjusting a range therefore tears down the CalendarView for that page and builds a new one mid-transition, discarding the @State it animates with - gridSize, topOffset, tipOffset, calendarHeight and the canAnimate re-entry guard at CalendarSwiper.zip#entry/src/main/ets/components/CalendarView.ets:42-46. The sample happens to rely on that rebuild, because CalendarView takes the range as @Prop (CalendarView.ets:40) and a @Prop copy would not otherwise see the mutation - so two defects mask each other, and fixing either one alone changes the behaviour.
- Fix: Give CalendarViewRange an id assigned in getCalendarRange (for example `${month}-${weekViewIndex}`) and key on it. Fix this together with the in-place mutation: with the ranges immutable, a stable key is correct and the CalendarView keeps its animation state across the transition.

### `HW-02-0130` - In week view the selected week is advanced by the Swiper's page delta instead of being read from the target page's own week index

- Category B, severity medium, confidence confirmed
- Features: LIFE-18
- Document: `huawei_industry_tree/02_convenient_life/docs/18_calendar_swiper.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calendar_swiper-0000002355470245
- Why: Section B of the review covers logic that is correct only by coincidence. The two index spaces line up 1:1 only because the week-range builder emits exactly one entry per data week and dedups the boundary week with lastWeekIndex (CalendarUtil.ets:57-68). Any change that makes weekRanges shorter or longer than data - the missing final-month range in the sibling finding is one such change - desynchronises the two, and the calendar then highlights a different week from the one on screen with no error. selectedWeekIndex is also used to index this.data directly at CalendarPage.ets:105, so a drift is an out-of-range read.
- Fix: Read it from the range: this.selectedWeekIndex = this.weekRanges[targetIndex].weekViewIndex; which is exactly the value refreshCalendarData matches against, and needs no assumption about the two arrays having equal length.

### `HW-02-0131` - The rawfile read has no error handling and no fallback, and the page indexes into the resulting array unconditionally during build

- Category B, severity medium, confidence confirmed
- Features: LIFE-18
- Document: `huawei_industry_tree/02_convenient_life/docs/18_calendar_swiper.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calendar_swiper-0000002355470245
- Why: Section B of the review covers null risk and empty error handling. this.data starts as an empty array, so any path that leaves it empty - a missing or malformed rawfile, a JSON.parse throw, or getHostContext returning undefined - makes data[2] undefined and the header's property read throws during build, taking the whole page down rather than degrading. getRawFileContentSync and JSON.parse both throw, and neither is guarded.
- Fix: Wrap the read in try/catch in RawfileUtil, log the BusinessError and return an empty array on failure. In CalendarPage, guard the header and the Swiper behind if (this.data.length > 0) so an empty dataset renders an empty state instead of throwing.

### `HW-02-0132` - EntryAbility subscribes to avoidAreaChange but never unsubscribes

- Category B, severity medium, confidence confirmed
- Features: LIFE-18
- Document: `huawei_industry_tree/02_convenient_life/docs/18_calendar_swiper.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calendar_swiper-0000002355470245
- Why: Section B of the review flags subscriptions that are opened but never closed. The callback holds a closure writing into AppStorage and keeps the window referenced after the stage is torn down.
- Fix: Keep the window handle and the callback in fields of EntryAbility and release the listener in onWindowStageDestroy.

### `HW-02-0137` - The address classifier substring-matches a serialised object instead of reading its fields, so any address whose text happens to contain the key names is misclassified

- Category C, severity medium, confidence confirmed
- Features: LIFE-19
- Document: `huawei_industry_tree/02_convenient_life/docs/19_parcel_address_text_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_text_recognition-0000002326921238
- Why: Section C of the review covers logic that is correct only for particular inputs. A substring search over a serialised object cannot distinguish a key from a value, so an entity whose address text or attributes contain any of those six words - which is ordinary for a romanised or English-language address, and for a Chinese address containing 城市 rendered into the payload - takes a different branch. The condition is also eight separate toString calls on the same object inside a loop over every entity, rebuilding the string each time.
- Fix: Parse it once per entity - const attrs: Record<string, string> = JSON.parse(entity.jsonObject.toString()); - and test the properties directly: attrs.adornLocation === undefined, attrs.coreLocation !== undefined, and so on. That removes both the ambiguity and the repeated serialisation.

### `HW-02-0138` - Splitting the address on the characters 区 and 县 takes only the second fragment, which is undefined when the character ends the string and discards everything after a second occurrence

- Category B, severity medium, confidence confirmed
- Features: LIFE-19
- Document: `huawei_industry_tree/02_convenient_life/docs/19_parcel_address_text_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_text_recognition-0000002326921238
- Why: Section B of the review covers out-of-range and undefined risk. String.split returns one more fragment than there are separators, so addressList[1] is the empty string when the character is last and is undefined only if the character is absent - which the includes guard prevents - but the real defect is the third fragment onward: an address containing 区 twice, which is common where a district name itself ends in 区 followed by a road name containing it, silently loses everything after the second occurrence. Splitting on a character that also occurs inside proper nouns is not a safe way to cut a Chinese address.
- Fix: Use the index rather than split: const at = entity.text.indexOf(Constants.ENTRY_TEXT_SPLIT_ZONE); if (at >= 0) { this.provinceAddress = entity.text.slice(0, at + 1); this.detailedAddress = entity.text.slice(at + 1); }. Better still, take the administrative prefix from the parsed jsonObject fields rather than cutting the display text at all.

### `HW-02-0139` - formatEntityResult is named and typed as a formatter but is the function that fills the whole form, and the debug string it builds is only logged

- Category C, severity medium, confidence confirmed
- Features: LIFE-19
- Document: `huawei_industry_tree/02_convenient_life/docs/19_parcel_address_text_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_text_recognition-0000002326921238
- Why: Section C of the review covers state handling. The name and the return type both describe a pure formatter, while the function's actual purpose is to mutate the page. A reader following the document would reasonably call it to build a debug string and be surprised that it rewrites the form; and because the two responsibilities share one loop, the classification logic cannot be tested or reused without also writing to component state.
- Fix: Split it - parseEntities(entities): AddressFields returning { name, phone, province, detail } with no side effects, and a caller that assigns those four fields to the @State variables. Keep the util.format debug string in a separate, clearly named helper, or drop it and log the parsed object instead.

### `HW-02-0145` - The permission check returns success on the first granted permission and enables the location layer from inside a function named check

- Category B, severity medium, confidence confirmed
- Features: LIFE-20
- Document: `huawei_industry_tree/02_convenient_life/docs/20_confirm_direction_in_map_to_rotate_and_move.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/confirm_direction_in_map_to_rotate_and_move-0000002335234358
- Why: Section B of the review covers logic defects. The loop is an OR over two permissions that must both be held, so granting only approximate location makes the check report success and the precise-location request is never made. Enabling the my-location layer inside the check also means the function cannot be called to ask a question - every caller that only wants to know the status gets a side effect on the map.
- Fix: Invert the loop - for (let permission of permissions) { if (await this.checkAccessToken(permission) !== abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED) { return false; } } return true; - and move the two setMyLocation calls to the call site that acts on the result.

### `HW-02-0146` - The permission request logs its result object and then enables the location layer regardless of whether anything was granted

- Category B, severity medium, confidence confirmed
- Features: LIFE-20
- Document: `huawei_industry_tree/02_convenient_life/docs/20_confirm_direction_in_map_to_rotate_and_move.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/confirm_direction_in_map_to_rotate_and_move-0000002335234358
- Why: Section B of the review covers missing authorization handling. A flat refusal resolves successfully, so the my-location layer and button are enabled for a user who said no, and the only diagnostic written is [object Object]. Combined with the permissions not being declared at all, every run of this sample takes exactly this path.
- Fix: Test the result - .then((data: PermissionRequestResult) => { LOGGER.info(`authResults: ${JSON.stringify(data.authResults)}`); if (data.authResults.every((r: number) => r === 0)) { mapController.setMyLocationEnabled(true); mapController.setMyLocationControlsEnabled(true); } else { /* tell the user, or offer requestPermissionOnSetting */ } }) - and use JSON.stringify in the log so the values are readable.

### `HW-02-0147` - moveCameraToPositionWithMarker is declared async and returns a promise that resolves before the camera animation it delegates to has begun

- Category B, severity medium, confidence confirmed
- Features: LIFE-20
- Document: `huawei_industry_tree/02_convenient_life/docs/20_confirm_direction_in_map_to_rotate_and_move.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/confirm_direction_in_map_to_rotate_and_move-0000002335234358
- Why: Section B of the review flags promises that should be awaited but are not. The wrapper advertises a Promise<void> that completes when the move is done and delivers one that completes immediately, so any caller that awaits it proceeds while the camera is still animating - and a rejection from animateCameraWithMarker becomes an unhandled rejection rather than reaching the caller. The polled flag exists precisely because this return value is useless, so fixing the await would let the whole nested-interval construction be deleted.
- Fix: Return or await the delegated call. The caller can then write await MAP_UTIL.moveCameraToPositionWithMarker(...) followed directly by the marker update, replacing the inner setInterval and the cameraMoveFinished flag entirely.

### `HW-02-0149` - The two map event listeners are registered and never unsubscribed, although MapEventManager exposes off

- Category B, severity medium, confidence confirmed
- Features: LIFE-14
- Document: `huawei_industry_tree/02_convenient_life/docs/14_map_bind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_bind-0000002297622922
- Why: Section B of the review flags subscriptions that are opened but never closed. Both callbacks are closures over the page - myLocationButtonClick calls this.mapController?.setMyLocation(location) - so they keep the component and its UIContext reachable after the page is destroyed, and the geolocation request they start can still fire against it.
- Fix: Keep the two callbacks in fields so they can be passed back, and add aboutToDisappear(): void { this.mapEventManager?.off('mapLoad', this.mapLoad); this.mapEventManager?.off('myLocationButtonClick', this.myLocationButtonClick); }. ConfirmDirectionInMap.zip#entry/src/main/ets/utils/MapUtil.ets:148-157 is the pattern to copy.

### `HW-02-0153` - Recognition is deferred by two seconds using the toast-duration constant as a timer delay

- Category C, severity medium, confidence confirmed
- Features: LIFE-21
- Document: `huawei_industry_tree/02_convenient_life/docs/21_parcel_address_image_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_image_recognition-0000002343383049
- Why: Section C of the review covers UI defects. A single constant now means two unrelated things, so shortening the toast would shorten the delay before OCR starts and lengthening it would delay recognition further. The delay itself has no stated purpose - recognizeText is asynchronous and reports through a callback, so nothing needs to wait for the toast - and it makes every recognition feel two seconds slower than it is.
- Fix: Remove the setTimeout and call textRecognition.recognizeText directly; the toast shown by the caller at Index.ets:82-85 and :109-112 already covers the wait. If a delay is genuinely required, give it its own constant such as OCR_START_DELAY with a comment saying why.

### `HW-02-0154` - With no recognised text the page substitutes a hard-coded demo address, so pressing the button with no image fills the form with fabricated personal data

- Category C, severity medium, confidence confirmed
- Features: LIFE-21
- Document: `huawei_industry_tree/02_convenient_life/docs/21_parcel_address_image_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_image_recognition-0000002343383049
- Why: Section C of the review covers state handling. The user taps the recognise button before choosing an image and the form fills with a name, a phone number and an address that came from nowhere - indistinguishable, on screen, from a real recognition result, and ready to be saved by the 保存地址 button. The placeholder is also not a valid phone number: 1888888888888 is thirteen digits where a Chinese mobile number has eleven, so it is not even usable as a demonstration.
- Fix: Replace the substitution with a guard: if (!this.recognizeText) { showToast($r('app.string.image_recognition_fail')); return; }. If a demo seed is wanted, put it in the TextArea's placeholder or behind an explicit 示例 button, so it can never be mistaken for a recognition result.

### `HW-02-0155` - The OCR engine's initialisation result is logged but never checked, and recognition proceeds regardless

- Category B, severity medium, confidence confirmed
- Features: LIFE-21
- Document: `huawei_industry_tree/02_convenient_life/docs/21_parcel_address_image_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_image_recognition-0000002343383049
- Why: Section B of the review covers unhandled paths. init is the call that loads the recognition model, and it is the one place a device that cannot run OCR would say so. Discarding its result means the failure surfaces later, inside a callback whose error branch does not exist either, so a device without the capability shows the user an image, a toast saying recognition has started, and then nothing at all.
- Fix: Keep the result in a field and gate on it - this.ocrReady = (initResult === 0); - disabling the 图片识别 and camera buttons and showing a message when it is false, and skipping textRecognition.release() in the teardown when initialisation never succeeded.

### `HW-02-0156` - Cutting the address on 区 with split keeps only the second fragment, so an address containing the character twice loses its tail

- Category C, severity medium, confidence confirmed
- Features: LIFE-21
- Document: `huawei_industry_tree/02_convenient_life/docs/21_parcel_address_image_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_image_recognition-0000002343383049
- Why: Section C of the review covers logic that is correct only for particular inputs. 区 occurs inside ordinary Chinese addresses beyond the administrative district - in compound and estate names such as 小区 - so a text like 杭州市西湖区某小区3号楼 splits into three fragments and everything after the second occurrence is silently dropped from the detailed address. Splitting on a character that also appears inside proper nouns is not a safe way to cut an address.
- Fix: Use the index instead of split: const at = entity.text.indexOf(DISTRICT); if (at >= 0) { this.address = entity.text.slice(0, at + 1); this.detailAddress = entity.text.slice(at + 1); } else { this.detailAddress = entity.text; }. Taking the administrative prefix from the entity's structured jsonObject rather than cutting the display text would remove the guesswork entirely.

### `HW-02-0157` - The document's address-splitting snippet drops the 区 character that the shipped code re-appends, so the region field loses its last character

- Category E, severity medium, confidence confirmed
- Features: LIFE-21
- Document: `huawei_industry_tree/02_convenient_life/docs/21_parcel_address_image_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_image_recognition-0000002343383049
- Why: Section E of the review requires code snippets to match the zip implementation. split consumes the separator, so the printed version leaves the region as 浙江省杭州市西湖 where the shipped one produces 浙江省杭州市西湖区 - a truncated administrative name in the field the form saves. A reader copying the snippet gets a subtly wrong result that looks plausible enough to ship.
- Fix: Correct document line 125 to this.address = addresses[0] + '区';, and use the named constant as the sample does. Better still, replace the whole snippet with the indexOf/slice form so the document does not teach the split-based cut at all.

### `HW-02-0164` - Leaving the scan page fires releaseCamera twice concurrently because it is wired to both onHidden and onWillHide

- Category B, severity medium, confidence confirmed
- Features: LIFE-22
- Document: `huawei_industry_tree/02_convenient_life/docs/22_id_card_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/id_card_scan-0000002338766518
- Why: Neither callback awaits the other, so the second invocation reads this.previewOutput, this.session and this.cameraInput while the first is still awaiting their release and has not reached its finally block. Release is then called a second time on handles that are already being torn down, and the resulting errors are swallowed by the surrounding catch blocks, so the double teardown is invisible in the log.
- Fix: Keep the release in one place. Bind it to onWillHide only (the earlier of the two) and let onHidden do nothing more than reset isInit, or guard releaseCamera with a boolean so a second concurrent call returns immediately.

### `HW-02-0165` - Every arriving preview frame starts a new recognition with no in-flight guard, so recognitions and their bitmaps pile up

- Category C, severity medium, confidence confirmed
- Features: LIFE-22
- Document: `huawei_industry_tree/02_convenient_life/docs/22_id_card_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/id_card_scan-0000002338766518
- Why: The ImageReceiver delivers frames at preview rate while a single recognizeText round trip takes far longer, so the callback keeps launching new recognitions on top of the ones still running. Each one decodes and holds its own full-resolution PixelMap, so the concurrency multiplies the leak reported at CameraService.ets:385 instead of merely repeating it.
- Fix: Add an isRecognizing field, return early from the imageArrival handler while it is true, set it before calling recognizeText and clear it in a finally on the recognizeText promise chain.

### `HW-02-0168` - The first implementation snippet obtains the surface id with getXComponentSurfaceId, which is not how the shipped sample does it and not what the component reference advises

- Category E, severity medium, confidence confirmed
- Features: LIFE-22
- Document: `huawei_industry_tree/02_convenient_life/docs/22_id_card_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/id_card_scan-0000002338766518
- Why: Step 1 is the entry point of the whole walkthrough and it teaches the pattern the sample deliberately avoided. A developer who follows the document instead of the ZIP places the call where the reference warns the surface id may not be valid yet, and then cannot find this.surfaceId or a getXComponentSurfaceId call anywhere in the downloadable code to compare against.
- Fix: Replace the step 1 snippet with the shipped code: a controller subclass whose onSurfaceCreated receives the surface id and starts CameraService.initCamera, and keep setXComponentSurfaceRect in onLoad where the sample puts it.

### `HW-02-0169` - The constraints section lists only version requirements and omits that the entity extraction step does not work on the emulator

- Category E, severity medium, confidence confirmed
- Features: LIFE-22
- Document: `huawei_industry_tree/02_convenient_life/docs/22_id_card_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/id_card_scan-0000002338766518
- Why: The feature has two device-only dependencies - the camera and the entity extraction service - and the reader is told about neither. Someone running the sample on the emulator gets a preview that never produces a result, and nothing in the constraints explains why.
- Fix: Add the emulator restriction to 约束与限制, citing the entity extraction guide, alongside a note that the custom camera preview also requires a physical device.

### `HW-02-0170` - The two convenient-life OCR samples disagree on whether textRecognition has to be initialised and released around recognizeText

- Category F, severity medium, confidence confirmed
- Features: LIFE-22
- Document: `huawei_industry_tree/02_convenient_life/docs/22_id_card_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/id_card_scan-0000002338766518
- Why: A developer reading the two industry documents side by side cannot tell whether init and release are mandatory, optional or harmful. One sample implies the recognizer is stateless and can be called ad hoc; the other implies it owns a session that must be opened and closed. Only one of the two can be the pattern the platform intends.
- Fix: Decide which lifecycle is correct for textRecognition, state it in both documents, and align the two samples on it. If init is optional but recommended for repeated recognition, say so explicitly in the step 4 text of this document, since this sample calls recognizeText on every preview frame.

### `HW-02-0175` - The avoid areas are read synchronously right after setWindowLayoutFullScreen, whose promise has not resolved yet

- Category B, severity medium, confidence confirmed
- Features: LIFE-23
- Document: `huawei_industry_tree/02_convenient_life/docs/23_card_information_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_information_recognition-0000002377837925
- Why: The two reads race the layout change they depend on: the API returns a promise precisely because the immersive layout is not in effect when the call returns, and an immersive layout is what makes the status bar and navigation bar become avoid areas the application must pad around. Whichever value the first read happens to catch is written into AppStorage and consumed as the padding by MainPage.ets:300-305.
- Fix: Move the getWindowAvoidArea calls and the two AppStorage writes into the .then() branch of setWindowLayoutFullScreen, so the areas are read after the immersive layout is in effect. The avoidAreaChange listener then only has to handle later changes.

### `HW-02-0176` - ListItem is used inside a Row, which the component reference allows only under List or ListItemGroup

- Category C, severity medium, confidence confirmed
- Features: LIFE-23
- Document: `huawei_industry_tree/02_convenient_life/docs/23_card_information_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_information_recognition-0000002377837925
- Why: The reference states the allowed parent as an absolute, and a Row is not one of them, so the layout depends on unspecified behaviour. The wrapper also buys nothing here: the row is a fixed strip of four static hint images, not a list, and the @Builder inside it already produces a complete RelativeContainer.
- Fix: Remove the ListItem wrapper: 'Row() { ForEach(this.data, (item: ListData) => { this.listItem(item.img, item.name, item.qualified); }); }'.

### `HW-02-0177` - The canIUse guard around CardRecognition has no fallback, so on a device without the capability the page shows a submit form with nothing to scan

- Category C, severity medium, confidence confirmed
- Features: LIFE-23
- Document: `huawei_industry_tree/02_convenient_life/docs/23_card_information_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_information_recognition-0000002377837925
- Why: The guard correctly recognises that the control may be absent, then leaves the user on a dead page: a real-name form whose fields never fill and a submit button that pops back and marks the account verified. A guard whose negative branch is worse than a clean failure is the wrong shape to publish as a sample.
- Fix: Add an else branch that surfaces the unsupported state - a message plus this.pageStack.pop() - so the flow ends instead of continuing into an unusable submit form.

### `HW-02-0178` - The failure branch pops the page but does not return, so the rest of the callback runs on a page that is already gone

- Category B, severity medium, confidence confirmed
- Features: LIFE-23
- Document: `huawei_industry_tree/02_convenient_life/docs/23_card_information_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_information_recognition-0000002377837925
- Why: On a non-success code the page is popped and the callback then keeps writing into the @Consume state that MainPage owns, so a failed scan can still populate the confirmation model. The user meanwhile is returned to the home page with no explanation, which for a real-name flow is indistinguishable from the app silently deciding not to work.
- Fix: Return immediately after popping, and show the failure before doing so: 'if (params.code !== 200) { this.getUIContext().getPromptAction().showToast({ message: $r('app.string.recognition_failed') }); this.pageStack.pop(); return; }'.

### `HW-02-0179` - A Resource is popped as the navigation result and cast to string on the receiving side, so a state field declared string holds a Resource object

- Category B, severity medium, confidence confirmed
- Features: LIFE-23
- Document: `huawei_industry_tree/02_convenient_life/docs/23_card_information_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_information_recognition-0000002377837925
- Why: The union type on the sending side shows the author knew the value is a Resource, and the cast on the receiving side erases that. isRealName then only works because every use is a truthiness test (:152, :189, :214, :222, :230); the moment anyone renders or compares it as the string its declaration promises, they get a resource object instead. MainPage.ets:214 already feeds it into a ternary that selects an image source, so the field is one edit away from being used as a real string.
- Fix: Pop a value whose type matches what the caller declares - a boolean flag, or the plain selected card type string - and drop the 'as string' cast. Render the localized 'already verified' text from $r('app.string.haven_real_name') at the point of display, as MainPage.ets:153 already does.

### `HW-02-0182` - The document's only code snippet copies the deprecated callback parameter and drops the deprecation comment that the sample carries

- Category E, severity medium, confidence confirmed
- Features: LIFE-23
- Document: `huawei_industry_tree/02_convenient_life/docs/23_card_information_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_information_recognition-0000002377837925
- Why: The snippet is the entire technical content of the document - there is no other code and no API table. Stripping the deprecation warning makes the document strictly worse than the ZIP it links to, and a reader who copies from the page has no way to learn what the reader who opens the ZIP is told in the first comment they see.
- Fix: Update the snippet to onResult, keep the canIUse guard around the control, and state the minimum API version in 约束与限制 alongside the existing version bullets.

### `HW-02-0187` - setWindowLayoutFullScreen is called with its promise dropped entirely and the avoid areas are read on the next line

- Category B, severity medium, confidence confirmed
- Features: LIFE-24
- Document: `huawei_industry_tree/02_convenient_life/docs/24_appointment_service_remind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/appointment_service_remind-0000002378138101
- Why: Two defects in three lines. The dropped promise means a failure to enter the immersive layout is never observed - the reference notes the call 'neither takes effect nor returns an error' in the freeform window state, so the silent path is a real one. And the two reads race the layout change they depend on, because the immersive layout is precisely what turns the status bar and navigation bar into areas the content must pad around. ServicePage.ets:356-357 consumes both stored values as the page padding.
- Fix: Chain the reads: 'mWindow.setWindowLayoutFullScreen(true).then(() => { /* read and store both avoid areas, then subscribe */ }).catch((err: BusinessError) => { hilog.error(...); });'.

### `HW-02-0188` - When the reminder is not created because the calendar permission is missing, the order button gives the user no feedback at all

- Category B, severity medium, confidence confirmed
- Features: LIFE-24
- Document: `huawei_industry_tree/02_convenient_life/docs/24_appointment_service_remind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/appointment_service_remind-0000002378138101
- Why: Declining the calendar permission is the ordinary case this code path exists for. The user taps the order button, the settings request is fired without being awaited, and whatever they do next the button produces no toast, no error and no state change - indistinguishable from the button not working. The application already has an appoint_failed_tip string for exactly this message.
- Fix: Handle the false branch: '.then((res: boolean) => { showToast(res ? $r('app.string.appoint_success_tip') : $r('app.string.appoint_failed_tip'), this.getUIContext()); })'.

### `HW-02-0189` - The within-two-hours slot is built straight from the wall clock and is never bounded by the service closing time

- Category B, severity medium, confidence confirmed
- Features: LIFE-24
- Document: `huawei_industry_tree/02_convenient_life/docs/24_appointment_service_remind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/appointment_service_remind-0000002378138101
- Why: The slot is the default first row of the today tab, and it is the one the user is most likely to pick. Selected at 20:30 it produces a 20:30-22:30 appointment; selected at 03:00 - which the collapse reported at ModalSelectWindow.ets:33 makes the only option - it produces a 03:00-05:00 appointment. Both are outside the 08:00-20:45 service window the rest of the page enforces, and both are written straight into the calendar event as startTime and endTime.
- Fix: Bound the end against LATEST_TIME on the same day and drop the row when now is already past LATEST_TIME minus the slot length, so the two-hour option can never fall outside the service window the interval list is built from.

### `HW-02-0190` - The item count the user adjusts with the stepper is discarded when the page hands its result back

- Category B, severity medium, confidence confirmed
- Features: LIFE-24
- Document: `huawei_industry_tree/02_convenient_life/docs/24_appointment_service_remind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/appointment_service_remind-0000002378138101
- Why: A stepper that visibly changes a number and then has no effect on the order is worse than no stepper: the user sets a count of three, confirms, and the home page shows the same summary as a count of one. The price the home page then displays is a fixed '10' (ServicePage.ets:263), so nothing downstream notices either.
- Fix: Include it in the summary - 'this.itemInfo = `${ITEM_TYPES[this.selectedIndex]} ${this.weight}${CommonConstant.WEIGHT_KG} ${this.count}${CommonConstant.COUNT_UNIT}`;' - or drop the count stepper from the page.

### `HW-02-0191` - The logger imports hilog through the @ohos module path while the rest of the same project uses the kit path the reference documents

- Category A, severity medium, confidence confirmed
- Features: LIFE-24
- Document: `huawei_industry_tree/02_convenient_life/docs/24_appointment_service_remind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/appointment_service_remind-0000002378138101
- Why: The two files in one sample import the same module two different ways, and the one the utility class uses is not the form the API reference documents. A reader copying the logging utility - the file most likely to be lifted wholesale - carries the non-kit import into their own project.
- Fix: Change Logger.ets:15 to the named kit import: import { hilog } from '@kit.PerformanceAnalysisKit';

### `HW-02-0197` - The permission section lists only WRITE_CALENDAR although the Calendar Kit guide instructs declaring both calendar permissions

- Category E, severity medium, confidence confirmed
- Features: LIFE-24
- Document: `huawei_industry_tree/02_convenient_life/docs/24_appointment_service_remind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/appointment_service_remind-0000002378138101
- Why: Three sources disagree in one feature: the kit guide says declare both, the industry document names one, and the sample's own comment claims it obtains both while the line below it requests one. A reader cannot tell whether the read permission was deliberately omitted because the sample only writes, or forgotten.
- Fix: State in 权限说明 which permissions Calendar Kit requires and which subset this sample needs, and correct the misleading comment at EntryAbility.ets:71 so it matches the single permission the next line actually requests.

### `HW-02-0199` - The avoid areas are read outside the setWindowLayoutFullScreen promise chain, so they are read before the immersive layout is applied

- Category B, severity medium, confidence confirmed
- Features: LIFE-25
- Document: `huawei_industry_tree/02_convenient_life/docs/25_commuting_calculation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/commuting_calculation-0000002347813672
- Why: The promise exists because the immersive layout is not in effect when the call returns, and the immersive layout is precisely what turns the status bar and the navigation bar into areas the content must pad around. The two heights are consumed as layout offsets by HomePage.ets:42 and :143, so a stale first read shifts the header image and the bottom action bar.
- Fix: Move the four statements into the existing .then() branch, which currently only logs, and register the avoidAreaChange listener there as well.

### `HW-02-0200` - The getMainWindow callback ignores its error parameter and dereferences the window on the next line

- Category B, severity medium, confidence confirmed
- Features: LIFE-25
- Document: `huawei_industry_tree/02_convenient_life/docs/25_commuting_calculation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/commuting_calculation-0000002347813672
- Why: On the failure path data is not a window, so windowClass.getUIContext() on the following line throws inside an asynchronous callback where nothing catches it. Everything the callback sets up - the immersive layout, both avoid-area heights and the avoidAreaChange subscription - is lost with it, and the two AppStorage keys the pages read stay at their defaults with no diagnostic beyond the crash. The file already demonstrates the correct guard nine lines earlier.
- Fix: Open the callback with the same guard used for loadContent: 'if (err.code) { hilog.error(DOMAIN, 'testTag', `getMainWindow failed. Cause: ${JSON.stringify(err)}`); return; }'.

### `HW-02-0201` - The route result is indexed at routes[0].steps[0] with no length check, although the reference documents an empty array as the no-route outcome

- Category B, severity medium, confidence confirmed
- Features: LIFE-25
- Document: `huawei_industry_tree/02_convenient_life/docs/25_commuting_calculation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/commuting_calculation-0000002347813672
- Why: An empty routes array is a documented, expected outcome - the reference says so in the field description - not an exceptional one. Indexing it raises a TypeError that the surrounding try/catch happens to swallow and log as a route-planning failure, so the one case the API tells you to handle is the one the code handles by accident and mislabels in the log.
- Fix: Guard before reading: 'if (!result.routes.length || !result.routes[0].steps.length) { return; }' and give the caller a way to show that no route was found.

### `HW-02-0202` - When route planning fails the page renders its sentences with the numbers missing instead of showing any failure state

- Category B, severity medium, confidence confirmed
- Features: LIFE-25
- Document: `huawei_industry_tree/02_convenient_life/docs/25_commuting_calculation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/commuting_calculation-0000002347813672
- Why: Both lookups happen in aboutToAppear and both can fail - the reference lists error codes for them, and this sample additionally requires Map Service to be enabled in AppGallery Connect and the build to be manually signed, which the document itself spells out. On any of those failures the user sees the literal text 'About kilometres to Confucius Temple' and 'About by car, about on foot', with no indication that anything went wrong.
- Fix: Add a status field set in the catch blocks, and render either a retry affordance or a plain 'commute time unavailable' line when it is set, instead of interpolating empty strings into a fixed sentence.

### `HW-02-0203` - getMapImage is called with a then and no catch, so a failed map image is an unhandled rejection and a blank component

- Category B, severity medium, confidence confirmed
- Features: LIFE-25
- Document: `huawei_industry_tree/02_convenient_life/docs/25_commuting_calculation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/commuting_calculation-0000002347813672
- Why: The static map is also the only control that opens Petal Maps - its onClick at :49-61 is where openMapRoutePlan lives - so a failed image leaves a 167vp blank strip that still swallows taps, and nothing anywhere records why. The document states that this sample needs Map Service enabled in AppGallery Connect and a manually signed build, which makes a failed image the expected first-run outcome for anyone who has not finished that setup.
- Fix: Add the rejection handler and a fallback: '.catch((err: BusinessError) => { hilog.error(0x0000, 'StaticMap', `getMapImage failed. Code: ${err.code}`); })', and render a placeholder while this.image is undefined.

### `HW-02-0204` - openMapRoutePlan is awaited inside an async click handler with no try/catch and with a context that may be undefined

- Category B, severity medium, confidence confirmed
- Features: LIFE-25
- Document: `huawei_industry_tree/02_convenient_life/docs/25_commuting_calculation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/commuting_calculation-0000002347813672
- Why: This call leaves the application - it launches Petal Maps - and it is the payoff of the entire feature. If the map application is absent or the launch is refused, the rejection lands in an async click handler where nothing observes it: the user taps the map, nothing happens, and no log line is written. The neighbouring route-planning calls in HousingResourcesPage.ets:58-68 and :86-94 both wrap their awaits in try/catch, so the sample is inconsistent with itself.
- Fix: Wrap it: 'try { await petalMaps.openMapRoutePlan(this.getUIContext().getHostContext(), params); } catch (error) { let err = error as BusinessError; hilog.error(0x0000, 'StaticMap', `openMapRoutePlan failed. Code: ${err.code}`); }' and show a prompt on failure.

### `HW-02-0205` - onContentWillChange returns false unconditionally, so none of the five tabs on the home page can ever be selected

- Category C, severity medium, confidence confirmed
- Features: LIFE-25
- Document: `huawei_industry_tree/02_convenient_life/docs/25_commuting_calculation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/commuting_calculation-0000002347813672
- Why: The page renders a five-entry tab bar with indicators and then vetoes every switch, so the four extra tabs are unreachable by construction rather than merely unimplemented. A reader copying this Tabs block inherits a switching veto with no comment explaining it, and will spend the debugging time working out why taps on the tab bar do nothing.
- Fix: Drop the onContentWillChange veto and remove the four empty TabContent entries, or keep the veto and add a comment stating that the other tabs are intentionally out of scope for this sample.

### `HW-02-0206` - The button labelled route shows the placeholder toast while the real route-planning entry point is an undecorated tap on the map image

- Category C, severity medium, confidence confirmed
- Features: LIFE-25
- Document: `huawei_industry_tree/02_convenient_life/docs/25_commuting_calculation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/commuting_calculation-0000002347813672
- Why: The document's second implementation step is exactly this call, so a reader arrives looking for it, presses the only control that says route, and is told the feature is for display only. The working entry point is a picture that gives no sign it can be tapped. The two are in different files, which is why the mismatch survives reading either one alone.
- Fix: Move the openMapRoutePlan handler onto the route button - or call the same function from both - so the labelled control performs the action it is labelled with.

### `HW-02-0209` - The reference-documents entry for the route planning API links to the Petal Maps page instead

- Category E, severity medium, confidence confirmed
- Features: LIFE-25
- Document: `huawei_industry_tree/02_convenient_life/docs/25_commuting_calculation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/commuting_calculation-0000002347813672
- Why: 参考文档 is the section a reader uses to find the API contract for the code they just copied, and the entry for the API that does the actual work - getDrivingRoutes, getWalkingRoutes, RouteResult, RouteStep - takes them to the Petal Maps launcher page instead. Nothing on that page describes the fields the sample reads.
- Fix: Point the navi entry at map-navi-api, the URL the same document already uses in its 场景介绍 paragraph.

### `HW-02-0213` - The pagination loop derives its page count from the total match count and can request page indexes the search API rejects

- Category A, severity medium, confidence confirmed
- Features: LIFE-26
- Document: `huawei_industry_tree/02_convenient_life/docs/26_list_of_nearby_outlets.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/list_of_nearby_outlets-0000002347931784
- Why: A keyword search for a common query in a dense area returns a totalCount well above 500, and the loop then asks for pageIndex 26 and upwards, which the documented constraint pageIndex * pageSize <= 500 forbids. Every one of those requests is awaited in sequence inside the search path, so the failure - or the wasted round trips - lands directly in the user-visible search.
- Fix: Clamp the loop: 'const MAX_PAGE = Math.floor(500 / PAGE_SIZE); for (let i = 2; i <= Math.min(Math.ceil(count / PAGE_SIZE), MAX_PAGE); i++)', with PAGE_SIZE declared once and passed explicitly as pageSize rather than relying on the default.

### `HW-02-0214` - The reverse geocoding block ends in a completely empty catch

- Category B, severity medium, confidence confirmed
- Features: LIFE-26
- Document: `huawei_industry_tree/02_convenient_life/docs/26_list_of_nearby_outlets.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/list_of_nearby_outlets-0000002347931784
- Why: The try covers isGeocoderAvailable and the construction and dispatch of the reverse geocoding request. Anything those throw disappears without a trace, and the method still returns coordinates as if nothing happened, so the page continues to the place search with placeName left at its 'fetching location' placeholder and no record anywhere of why.
- Fix: Log the caught error at minimum: 'catch (error) { let err = error as BusinessError; hilog.error(0x0000, 'NearbyOutlets', `reverse geocode failed. Code: ${err.code}, message: ${err.message}`); }' and set a state field the UI can show.

### `HW-02-0215` - getCurrentLocation is awaited outside the try block, so a failed fix leaves the page showing searching forever

- Category B, severity medium, confidence confirmed
- Features: LIFE-26
- Document: `huawei_industry_tree/02_convenient_life/docs/26_list_of_nearby_outlets.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/list_of_nearby_outlets-0000002347931784
- Why: Obtaining a fix is the failure-prone step of this feature - the user can grant the permission and still have location services off, be indoors, or time out. When it rejects, the promise is unhandled, the search never starts, and the page is left permanently displaying the searching message with no way back except leaving and re-entering.
- Fix: Move the await inside the existing try, and have the caller handle the failure: wrap :253-254 in try/catch, and in the catch set isBeginToSearch back to false and show a prompt.

### `HW-02-0216` - The place name is derived by splitting the address on a separator that is empty whenever the district field is missing

- Category B, severity medium, confidence confirmed
- Features: LIFE-26
- Document: `huawei_industry_tree/02_convenient_life/docs/26_list_of_nearby_outlets.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/list_of_nearby_outlets-0000002347931784
- Why: subLocality is an optional field of the reverse geocoding result, and the code already acknowledges that by defaulting it to ''. That default then becomes a separator, so the ordinary missing-district case does not fall back to the full name - it produces a one-character place name. The comment on the line above (:51) shows the author was choosing between subLocality and premises for granularity, not considering the absent case.
- Fix: Return the address unchanged when the separator is empty: 'if (!subLocality) { return address; }' before splitting, and fall back to address when the split yields nothing useful.

### `HW-02-0217` - The opening-hours formatter labels the start of each range with the loop index instead of the weekday

- Category B, severity medium, confidence confirmed
- Features: LIFE-26
- Document: `huawei_industry_tree/02_convenient_life/docs/26_list_of_nearby_outlets.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/list_of_nearby_outlets-0000002347931784
- Why: The two halves of the same range label are read from different things: the end from the period's actual weekday, the start from where the period happens to sit in the array. They coincide only when the array is a complete, sorted, one-entry-per-day week starting at the right offset. Any venue that is closed one day, or opens twice on one day, produces a label naming days it is not open, and the array is longer than seven entries as soon as any day has two periods.
- Fix: Read the weekday from the data on both sides: replace this.getDayInChinese(i) at :178 with this.getDayInChinese(periods[i]?.open?.week ?? -1), matching the call fifteen lines above.

### `HW-02-0218` - openMapRoutePlan is awaited with no try/catch, so a failed handoff to the map application is silent

- Category B, severity medium, confidence confirmed
- Features: LIFE-26
- Document: `huawei_industry_tree/02_convenient_life/docs/26_list_of_nearby_outlets.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/list_of_nearby_outlets-0000002347931784
- Why: This is the navigate half of the feature named in the document title, and it leaves the application. If the map application is absent or the launch is refused, the rejection lands in an async method nobody awaits: the user taps 到这里 ("go here") and nothing at all happens, with no toast and no log line.
- Fix: Wrap it: 'try { await petalMaps.openMapRoutePlan(this.getUIContext().getHostContext(), params); } catch (error) { let err = error as BusinessError; hilog.error(0x0000, 'NearbyOutlets', `openMapRoutePlan failed. Code: ${err.code}`); this.promptAction.showToast({ message: $r('app.string.map_unavailable') }); }'.

### `HW-02-0223` - The first implementation snippet does not compile and uses a different address field from the shipped code

- Category E, severity medium, confidence confirmed
- Features: LIFE-26
- Document: `huawei_industry_tree/02_convenient_life/docs/26_list_of_nearby_outlets.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/list_of_nearby_outlets-0000002347931784
- Why: This is the document's only explanation of how the current location is turned into a place name, and as printed it cannot be pasted into a project: a try without a catch is a syntax error and isAvailable is undeclared. The third difference is quieter and worse - a reader following the document produces a different place name from the sample they downloaded, and the comment that explains the choice exists only in the ZIP.
- Fix: Reproduce the shipped block: keep the isGeocoderAvailable assignment, keep the catch, and use the same address field the sample uses - or state in the prose why premises is shown instead.

### `HW-02-0228` - The avoid areas are read outside the setWindowLayoutFullScreen promise chain, so they are read before the immersive layout applies

- Category B, severity medium, confidence confirmed
- Features: LIFE-27
- Document: `huawei_industry_tree/02_convenient_life/docs/27_easylife_demo_vrhouse.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_demo_vrhouse-0000002349301462
- Why: The API returns a promise because the immersive layout is not in effect when the call returns, and it is the immersive layout that turns the status bar and navigation bar into areas the content must pad around. All four pages consume the two stored values as their top and bottom padding through @StorageLink, so a stale first read offsets the whole application.
- Fix: Move the two getWindowAvoidArea blocks and the avoidAreaChange registration into the existing .then() branch, which currently only logs.

### `HW-02-0229` - The navigation parameter guard reads length before it checks for undefined and null

- Category B, severity medium, confidence confirmed
- Features: LIFE-27
- Document: `huawei_industry_tree/02_convenient_life/docs/27_easylife_demo_vrhouse.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_demo_vrhouse-0000002349301462
- Why: The two null checks exist precisely because the author considered that getParamByName may return nothing, and their position makes them unreachable in the case they were written for. The page then aborts through the catch, house stays undefined, and the reader sees an empty preview with a log line that blames parameter retrieval rather than the ordering bug.
- Fix: Reorder the conditions so the existence checks come first, and drop the redundant cast that hides the possible undefined from the compiler.

### `HW-02-0230` - px2vp is called with values typed as possibly undefined in every page's build method

- Category B, severity medium, confidence confirmed
- Features: LIFE-27
- Document: `huawei_industry_tree/02_convenient_life/docs/27_easylife_demo_vrhouse.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_demo_vrhouse-0000002349301462
- Why: The declared type says the value may be undefined and the call site passes it to px2vp, which takes a number - the union is declared and then ignored three times in three different files. The default is undefined rather than 0, so the very first build of each page converts a value the declaration itself says may not be there.
- Fix: Default the two @StorageLink fields to 0 and drop the union, which matches what EntryAbility always writes, or guard: 'top: this.topRectHeight ? this.uiContext.px2vp(this.topRectHeight) : 0'.

### `HW-02-0231` - Every ForEach key generator names its item parameter index and serialises the whole object, which is what a custom key generator exists to avoid

- Category C, severity medium, confidence confirmed
- Features: LIFE-27
- Document: `huawei_industry_tree/02_convenient_life/docs/27_easylife_demo_vrhouse.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_demo_vrhouse-0000002349301462
- Why: The three custom generators do exactly what the default one does - serialise the whole item - so they pay the memory cost the guidance describes while adding a false type annotation that stops the compiler from flagging the mistake. A reader copying the line believes it keys on an index, which is a different rule with different consequences. The fourth site inherits the same serialisation on the largest objects in the project.
- Fix: Key on a short unique field: '(category: Category) => category.name', '(detail: Detail) => detail.name' and '(house: House) => house.name', adding an id to the models where the name is not unique.

### `HW-02-0232` - The contact bar is placed at a hardcoded absolute y coordinate of 718

- Category C, severity medium, confidence confirmed
- Features: LIFE-27
- Document: `huawei_industry_tree/02_convenient_life/docs/27_easylife_demo_vrhouse.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_demo_vrhouse-0000002349301462
- Why: 718 is a fixed offset in vp from the top of the parent, chosen for one screen height, inside a layout that is otherwise expressed in percentages of the available space. On any device whose content area is not that height the contact bar overlaps the images above it or floats away from the bottom edge, and the pages already apply variable safe-area padding on top of that, which shifts the content but not this absolute offset.
- Fix: Remove the .position and let the bar sit at the end of the Column flow, or anchor it with a Stack aligned to Alignment.Bottom, and drop the negative top margins that compensate for the fixed placement.

### `HW-02-0234` - Both implementation snippets use identifiers that do not exist in the sample, so neither can be compiled as printed

- Category E, severity medium, confidence confirmed
- Features: LIFE-27
- Document: `huawei_industry_tree/02_convenient_life/docs/27_easylife_demo_vrhouse.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_demo_vrhouse-0000002349301462
- Why: These two snippets are the entire technical content of the document, and both misspell an identifier the reader has to type. Neither compiles as printed, and the misspellings are of exactly the kind a reader will not notice while reading - a case difference and a missing letter - so the failure surfaces only at build time with an error naming a symbol that appears nowhere in the sample they downloaded.
- Fix: Correct both snippets to swiperCtr and categories so they match the shipped HouseImagePage.ets and HouseModel.ets.

### `HW-02-0238` - When the contacts capability is unavailable the empty initial value is returned to the web page as though it were a phone number

- Category B, severity medium, confidence confirmed
- Features: LIFE-28
- Document: `huawei_industry_tree/02_convenient_life/docs/28_h5rechargeplatform.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5rechargeplatform-0000002387265697
- Why: The guard correctly recognises that the contacts component may be absent - the module declares phone, tablet and 2in1 - and then makes that case indistinguishable from a successful selection of an empty number. The web handler converts the empty string to Number('') which is 0, so the input silently fills with a zero and the user is given no reason why the picker did not open.
- Fix: Return a sentinel the page can test for, or throw so the web-side await can catch it, and add an else that reports the unsupported capability.

### `HW-02-0239` - setWindowLayoutFullScreen is given a then and no catch, so a failure to enter the immersive layout is an unhandled rejection

- Category B, severity medium, confidence confirmed
- Features: LIFE-28
- Document: `huawei_industry_tree/02_convenient_life/docs/28_h5rechargeplatform.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5rechargeplatform-0000002387265697
- Why: Reading the avoid areas inside .then() is the correct ordering and this sample gets that right - but with no rejection handler a failure silently skips both AppStorage writes as well, so the page renders with zero top and bottom padding under the system bars and nothing anywhere records why. The other lifecycle callback in the same file handles its error properly (:67-73).
- Fix: Add the rejection branch: '.catch((err: BusinessError) => { hilog.error(DOMAIN, 'testTag', 'Failed to set the window layout to full-screen mode. Cause: %{public}s', JSON.stringify(err)); });'.

### `HW-02-0240` - The selected phone number is converted to a Number before being written into the tel input, which destroys leading zeros

- Category B, severity medium, confidence confirmed
- Features: LIFE-28
- Document: `huawei_industry_tree/02_convenient_life/docs/28_h5rechargeplatform.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5rechargeplatform-0000002387265697
- Why: A phone number is a string of digits, not a quantity. Number() drops any leading zero, so a stored landline such as 01012345678 is written into the field as 1012345678, and a value longer than fifteen digits loses precision. The regular expression on the line above has already removed everything that is not a digit, so there is nothing left for the conversion to fix.
- Fix: Assign the cleaned string directly - the input is type tel and maxlength 11, so no numeric coercion is wanted at any point.

### `HW-02-0241` - The web page awaits the injected ArkTS method with no try/catch, so a rejected bridge call is silent on both sides

- Category B, severity medium, confidence confirmed
- Features: LIFE-28
- Document: `huawei_industry_tree/02_convenient_life/docs/28_h5rechargeplatform.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5rechargeplatform-0000002387265697
- Why: This is the only bridge call in the sample and neither end handles failure. When the ArkTS method throws - which the unchecked indexing at Recharge.ets:73 makes the ordinary outcome of cancelling the picker - the rejection lands in an async event listener where nothing observes it: the user taps the contacts icon, the picker closes, the field stays empty and no message appears anywhere.
- Fix: Wrap the call: 'try { const response = await contacts.selectContacts(); if (!response) { return; } document.getElementById('phone').value = response.replace(/\\D/g, ''); } catch (e) { alert('无法获取联系人'); }'.

### `HW-02-0245` - The document reproduces the unguarded contact expression without the error handling the API reference itself demonstrates

- Category E, severity medium, confidence confirmed
- Features: LIFE-28
- Document: `huawei_industry_tree/02_convenient_life/docs/28_h5rechargeplatform.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5rechargeplatform-0000002387265697
- Why: This snippet is one of only two in the document and it is the one a reader will copy into their own bridge method. As printed it turns two ordinary user actions - cancelling the picker, and choosing a contact with no stored number - into a thrown TypeError, and the trailing ! ensures the compiler will not warn them. The API's own reference page shows the failure branch that the document omits.
- Fix: Show the guarded form in the document: resolve the array into a variable, check it and the phone number list, and wrap the call in try/catch, matching the error handling the contact reference demonstrates.

### `HW-02-0249` - The marker drag events are subscribed on the map controller while the guide documents them on the event manager the sample already holds

- Category A, severity medium, confidence confirmed
- Features: LIFE-29
- Document: `huawei_industry_tree/02_convenient_life/docs/29_set_coverage_area.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_coverage_area-0000002547690109
- Why: One callback body subscribes to three events through two different objects, twenty lines apart, with no comment explaining why. The two that use the controller are exactly the two the guide documents on the event manager, and the event manager is already assigned to a field on the line the sample obtains it. It also leaves the teardown ambiguous: a reader adding the missing unsubscribe cannot tell which object to call off on for which event.
- Fix: Register all three through this.mapEventManager, as the marker guide's example does, so the subscriptions and their eventual unsubscriptions go through one object.

### `HW-02-0253` - The drag snippet calls changeRadius without this, and the circle snippet shows a fill colour the sample does not use

- Category E, severity medium, confidence confirmed
- Features: LIFE-29
- Document: `huawei_industry_tree/02_convenient_life/docs/29_set_coverage_area.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_coverage_area-0000002547690109
- Why: changeRadius is a method of the component, so the bare call in the snippet does not resolve - and it sits two lines above a correctly written this.movePointAnnotation call, which makes it read as a deliberate distinction rather than a slip. The colour difference is quieter but means the reader who follows the document sees a visibly more opaque overlay than the screenshot above it.
- Fix: Add the missing receiver in step 4 and align the fill colour in step 2 with Index.ets:112.

### `HW-02-0258` - The thumbnail is scaled to a fixed 500 by 500, so every image is stretched to a square

- Category B, severity medium, confidence confirmed
- Features: LIFE-30
- Document: `huawei_industry_tree/02_convenient_life/docs/30_thumbnail_taskpool.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/thumbnail_taskpool-0000002338966908
- Why: The comment calls the two values a compression ratio, but a ratio is one number; using a separate factor per axis is what turns the scale into a resize to fixed dimensions. A landscape photo is squeezed horizontally and stretched vertically before it is ever displayed, and because the distortion is baked into the packed thumbnail no display-side objectFit can undo it. It also wastes work on portrait and landscape images alike, since one dimension is enlarged past what the 48vp frame needs.
- Fix: Compute a single factor and pass it twice: 'let scale = Math.min(500 / imageInfo.size.width, 500 / imageInfo.size.height); await pixelmap.scale(scale, scale);', and set objectFit on the Image so the square frame crops rather than distorts.

### `HW-02-0259` - TaskManager exposes a second entry point that bypasses the whole concurrency layer, and the document's first step uses it

- Category B, severity medium, confidence confirmed
- Features: LIFE-30
- Document: `huawei_industry_tree/02_convenient_life/docs/30_thumbnail_taskpool.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/thumbnail_taskpool-0000002338966908
- Why: The document's subject is managing multi-threaded tasks and allocating resources sensibly, and its first step routes work through the method that does none of that. The two names differ by a suffix, both take a BaseTask, and nothing in either signature or the document distinguishes them - so a reader picking executeTask believes they are using the pool manager the sample is about. The import task copies up to fifty files chosen by the picker (ThumbnailPage.ets:120, 'photoSelectOptions.maxSelectNumber = 50'), which is precisely the workload the cap exists for.
- Fix: Route executeTask through an executor with a default cap, or rename it to make the difference explicit and have the import path call execute like the thumbnail path does.

### `HW-02-0260` - The import worker swallows every per-file failure and the caller reports success unconditionally

- Category B, severity medium, confidence confirmed
- Features: LIFE-30
- Document: `huawei_industry_tree/02_convenient_life/docs/30_thumbnail_taskpool.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/thumbnail_taskpool-0000002338966908
- Why: The user picks up to fifty images and is told the import succeeded even when every single copy failed - a full disk, a revoked URI or an unreadable source all produce the same green toast. The failures are visible only in the log, and because the task resolves normally the executor also treats the run as a success.
- Fix: Return a result from the worker - the number of files copied and the number that failed - and have the callback choose between the success string and a failure string based on it.

### `HW-02-0261` - Appended query results are published with notifyDataReload although the data source provides the incremental pushData path

- Category C, severity medium, confidence confirmed
- Features: LIFE-30
- Document: `huawei_industry_tree/02_convenient_life/docs/30_thumbnail_taskpool.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/thumbnail_taskpool-0000002338966908
- Why: The worker sends results in batches of a thousand while the list is already on screen, and each batch tells LazyForEach to refresh every child component rather than to add the new ones. Rows that are already rendered - and whose thumbnails have already been generated by a separate task each - are put through the reload comparison on every batch, which is the opposite of what the sample is demonstrating. The data source's own pushData, written correctly, is never called.
- Fix: Append through the data source's pushData, or use notifyDatasetChange with an add operation for the batch, so only the new indexes are created.

### `HW-02-0265` - The document states an overflow policy that the shipped executor implements backwards

- Category E, severity medium, confidence confirmed
- Features: LIFE-30
- Document: `huawei_industry_tree/02_convenient_life/docs/30_thumbnail_taskpool.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/thumbnail_taskpool-0000002338966908
- Why: The sentence is the document's only explanation of what the maxWaitLimit argument buys, and it makes a specific promise - that requests are buffered so tasks are not lost, and that overflow costs you the oldest entries. The code loses the newest task in exactly the situation the sentence is describing, so a reader who trusts the document ships a queue that silently drops the work the user just asked for.
- Fix: Fix the slice in TaskExecutor.ets:145 so the tail is kept, which makes the code match the documented policy, and keep the sentence as written.

### `HW-02-0266` - The industry FAQ page is a single redirect line whose target has no industry anchor, so the content it names cannot be reached from it

- Category E, severity medium, confidence confirmed
- Features: LIFE-31
- Document: `huawei_industry_tree/02_convenient_life/docs/31_practice-convenient-life-app-architecture-v1_2.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1_2-0000002263237450
- Why: The page is titled 行业常见问题 ("Industry FAQ") and is the last numbered document of the convenient-life industry, so it is where a reader goes for the industry's known issues. What it hands them is the front door of a general phone FAQ with no way to tell which of its entries were the industry ones. A redirect that drops the very qualifier its title promises leaves the migrated content effectively unreachable from here.
- Fix: Point the link at the migrated convenient-life section - an anchor or a filtered view of the FAQ - or list the migrated questions with their individual destinations, so the industry qualifier survives the migration.

### `HW-02-0267` - Ten industries ship a byte-identical industry FAQ page pointing at the same generic destination

- Category F, severity medium, confidence confirmed
- Features: LIFE-31
- Document: `huawei_industry_tree/02_convenient_life/docs/31_practice-convenient-life-app-architecture-v1_2.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1_2-0000002263237450
- Why: The word 行业 ("industry") in the title is the only industry-specific thing about the page, and it is not reflected in the body, the link or the destination. A reader who follows the convenient-life FAQ and the automotive FAQ arrives at the same generic page, which means either the migrated FAQs are no longer separated by industry - in which case ten identically titled industry pages are misleading - or they are, and none of the ten links reaches them.
- Fix: Decide which it is: if the migrated FAQ is still organised by industry, give each of the ten pages its own destination; if it is not, replace the per-industry pages with a single cross-industry entry so the titles stop promising a division that no longer exists.

### `HW-02-0270` - 1 sample project declares permissions that no code path in the project uses

- Category D, severity medium, confidence confirmed
- Features: LIFE-29
- Document: `huawei_industry_tree/02_convenient_life/docs/29_set_coverage_area.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_coverage_area-0000002547690109
- Why: A permission declared in module.json5 but never referenced by any .ets or .ts file in the project cannot be exercised, so it is pure over-declaration. Over-declared permissions are a release-review rejection reason, and restricted permissions especially so. Because these module.json5 files are copied as templates, the surplus entries propagate into products that have even less claim to them.
- Fix: Delete every declared permission the code does not exercise. Where the capability is reached through a permission-free path such as SaveButton or PhotoViewPicker, no declaration is needed at all.

### `HW-02-0271` - CredentialsPage releases the file descriptor of a scanned ID card but never releases the ImageSource built from it

- Category B, severity medium, confidence confirmed
- Features: LIFE-01
- Document: `huawei_industry_tree/02_convenient_life/docs/01_practice-convenient-life-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1-0000001952539489
- Why: The finally block correctly closes the descriptor but leaves the ImageSource allocated. image.createImageSource holds a native decoder that the ArkTS garbage collector does not reclaim, so every card scan leaks one. The callback runs once per recognition and the page is the ID-card capture screen, so repeated scans accumulate. This is distinct from HW-02-0003, which concerns the descriptor in the document own snippet rather than the ImageSource in the shipped code.
- Fix: Add imageSource.release() to the finally block, before or after fs.closeSync(file).

### `HW-02-0015` - The avoid areas are read once at window-stage creation and never refreshed, so the page padding is stale after a rotation or a keyboard show

- Category B, severity low, confidence confirmed
- Features: LIFE-03
- Document: `huawei_industry_tree/02_convenient_life/docs/03_vehicle_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vehicle_keyboard-0000002236257106
- Why: Section B of the review covers lifecycle cleanup and stale state. The avoid area changes when the device rotates, when a fold opens, or when the status bar height changes; a value captured once at startup silently stops matching the window, and the immersive page then overlaps the system bars.
- Fix: Add the subscription in onWindowStageCreate, writing topAvoid for TYPE_SYSTEM and bottomAvoid for TYPE_NAVIGATION_INDICATOR, and pair it with an off() call in onWindowStageDestroy.

### `HW-02-0017` - The key handler is passed to the child as an unbound method reference, so it runs with the Keyboard component as this and works only because Keyboard happens to declare identically named @Link fields

- Category C, severity low, confidence confirmed
- Features: LIFE-03
- Document: `huawei_industry_tree/02_convenient_life/docs/03_vehicle_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vehicle_keyboard-0000002236257106
- Why: Section C of the review covers state handling. The handler mutates the state of whichever component it is called on, not of the component that defined it. Rename either @Link in Keyboard, or reuse the handler from a component without those two fields, and the writes land on undefined instead of on the licence-plate string - a silent no-op rather than a compile error.
- Fix: Pass an arrow function that closes over the owner: Keyboard({ ..., onKeyboardEvent: (item: IKeyAttribute) => this.onKeyboardEvent(item) }). The @Link pair on Keyboard can then be narrowed to whatever the child actually renders.

### `HW-02-0018` - The vehicleKeyboard HAR does not declare the phone device type that the entry HAP targets, and uses default, which the reference says cannot be released

- Category E, severity low, confidence confirmed
- Features: LIFE-03
- Document: `huawei_industry_tree/02_convenient_life/docs/03_vehicle_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vehicle_keyboard-0000002236257106
- Why: Section E of the review covers configuration that does not match the feature. The application ships only for phone, yet the HAR that contains the entire licence-plate keyboard lists tablet and 2in1 - which the entry HAP does not support - and substitutes the build-only default value for the one device type that is actually targeted.
- Fix: Change features/vehicleKeyboard/src/main/module.json5 to deviceTypes: ["phone"]; if tablet and 2in1 really are supported, add them to the entry HAP as well and verify the hard-coded keyboard offsets on those widths first (see the delete-key finding).

### `HW-02-0019` - PaymentInfo declares a @Consume binding to the licence plate that it never reads

- Category B, severity low, confidence confirmed
- Features: LIFE-03
- Document: `huawei_industry_tree/02_convenient_life/docs/03_vehicle_keyboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vehicle_keyboard-0000002236257106
- Why: Section B of the review covers dead state. A @Consume registers the component as a subscriber of the @Provide in the ancestor, so every keystroke on the plate re-renders PaymentInfo even though nothing it draws depends on the plate. It also reads as if the plate were the display source, hiding that the dotted string is computed once, only when the confirm button is pressed.
- Fix: Delete the @Consume('licensePlate') declaration from PaymentInfoComponent.ets.

### `HW-02-0025` - drawRect mixes drawing with state mutation, so the selection count and list are updated only on one of its three branches

- Category C, severity low, confidence confirmed
- Features: LIFE-04
- Document: `huawei_industry_tree/02_convenient_life/docs/04_canvas_cinema.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_cinema-0000002272398929
- Why: Section C of the review covers state handling. A function named drawRect that also owns part of the selection state has no single caller contract: three of its four call sites must remember to finish the job, and the fourth must remember not to. That is why count and selectedSeats.length are maintained as two independent counters that can only be kept in step by hand.
- Fix: Move this.seats[x][y] = 1, this.count++ and this.selectedSeats.push([x, y]) out of drawRect into handleClick, and derive the displayed count from this.selectedSeats.length instead of maintaining a separate count field.

### `HW-02-0030` - A divider width is fetched with resourceManager.getStringSync inside build(), from a possibly undefined context, where the same value is available as a resource reference two lines above

- Category B, severity low, confidence confirmed
- Features: LIFE-05
- Document: `huawei_industry_tree/02_convenient_life/docs/05_cascading_menu_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cascading_menu_selection-0000002274600681
- Why: Section B of the review covers null risk and needless work in the render path. The expression performs a synchronous resource-manager lookup on every rebuild of the component, and when the context is undefined the optional chain silently yields undefined, so the divider quietly loses its width instead of failing. Nothing here needs the resource manager - the component tree already resolves $r() references directly.
- Fix: Replace the expression with .strokeWidth($r('app.integer.integer_1')) and delete the context field if nothing else uses it.

### `HW-02-0031` - Removing one chip walks the whole three-level menu tree twice, because the watcher resets its own trigger variable

- Category B, severity low, confidence confirmed
- Features: LIFE-05
- Document: `huawei_industry_tree/02_convenient_life/docs/05_cascading_menu_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cascading_menu_selection-0000002274600681
- Why: Section B of the review covers work done in the wrong place. The reset is needed so that deleting the same name twice in a row still triggers the watcher, but it makes the handler re-enter itself on every deletion, doubling a full O(primary x second x third) tree walk. It is also fragile: any future early exit or any state written before the reset runs twice.
- Fix: Guard the entry point: onDelete(): void { if (this.deleteItem === Constants.NULL_STRING) { return; } ... }. The tree walk can also stop at the first match, since the names are unique across the 54 items in menu_item_list.json.

### `HW-02-0032` - The document's 实现思路 snippet shows a select-all placeholder it never explains and omits the ListItem contents, so it cannot be followed on its own

- Category E, severity low, confidence confirmed
- Features: LIFE-05
- Document: `huawei_industry_tree/02_convenient_life/docs/05_cascading_menu_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cascading_menu_selection-0000002274600681
- Why: Section E of the review requires snippets to reflect the zip. The comment points the reader at a select-all button on the first list that does not exist there, the empty ListItem bodies hide the PrimaryMenu/SecondLevelMenu/ThirdLevelMenu components that carry the whole @ObjectLink story the 场景介绍 advertises, and the abbreviated onClick omits the two assignments that keep the downstream select-all state correct.
- Fix: Replace the 实现思路 snippet with WholeMenu.ets lines 46-77 and 103-108 verbatim, and move the 全选 comment onto the second list where the button actually lives.

### `HW-02-0033` - The navigation-bar inset is read once inside the loadContent callback and never refreshed, so the dialog padding is stale after a rotation

- Category B, severity low, confidence confirmed
- Features: LIFE-05
- Document: `huawei_industry_tree/02_convenient_life/docs/05_cascading_menu_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cascading_menu_selection-0000002274600681
- Why: Section B of the review covers stale state and lifecycle. The navigation-bar inset changes with rotation, folding and the user's gesture-versus-three-button setting; a value captured once at startup stops matching the window, and the selected-items dialog - which is anchored to the bottom with DialogAlignment.Bottom - then overlaps the navigation bar. Note that the rest of the page avoids this by using expandSafeArea (SelectionPage.ets:97, BottomBar.ets:63) instead of a stored inset, so the two halves of the same screen use two different strategies.
- Fix: Add the subscription next to the initial read and pair it with windowClass.off('avoidAreaChange', cb) in onWindowStageDestroy; alternatively drop the AppStorage value and give the dialog content .expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.BOTTOM]).

### `HW-02-0041` - Two directory and file names in the documented project tree do not match the zip

- Category E, severity low, confidence confirmed
- Features: LIFE-06
- Document: `huawei_industry_tree/02_convenient_life/docs/06_sticky_note.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sticky_note-0000002275644713
- Why: Section E of the review requires the documented project tree to match the zip. Module resolution is case-sensitive when strictMode.caseSensitiveCheck is on, and srcEntry in module.json5 is matched literally, so both spellings from the document fail to build rather than merely reading oddly.
- Fix: Correct lines 125 and 128 of the 工程目录 tree. Renaming the directory to the conventional lower-case plural constants/ would also be reasonable, but then the import in StickyNotePage.ets must change with it.

### `HW-02-0042` - The note count is kept in a hand-maintained field that duplicates the array length, and it is updated in three places

- Category B, severity low, confidence confirmed
- Features: LIFE-06
- Document: `huawei_industry_tree/02_convenient_life/docs/06_sticky_note.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sticky_note-0000002275644713
- Why: Section B of the review covers state that can drift. The decrement happens inside an animation callback, so between the tap on the delete icon and the end of the 300 ms plus 500 ms animation chain the counter and the rendered list disagree; and any future path that splices deductionData without going through addItem or deleteItem leaves the counter permanently wrong, because nothing ever resynchronises it.
- Fix: Delete deductionDataSize and expose a getter - get size(): number { return this.deductionData.length; } - or make the page read this.appInfoList.length, which is already @Provide-decorated and therefore observed.

### `HW-02-0048` - Each budget is written twice - once as a constant that drives the Progress bar and the colour threshold, once as a literal in the label beside it

- Category B, severity low, confidence confirmed
- Features: LIFE-07
- Document: `huawei_industry_tree/02_convenient_life/docs/07_collapse_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/collapse_list-0000002248773270
- Why: Section B of the review covers duplicated state. Changing the budget requires editing two places in each file; miss one and the bar fills to a different number than the label claims, with no compile-time signal. The literal is also untranslatable, unlike every other string on the row, which goes through $r('app.string...').
- Fix: Replace Text('10000') with Text(`${BUDGET}`) and Text('500') with Text(`${BUDGET}`). The hand-tuned .margin({ left: 96 }) and .margin({ left: 114 }) that align the two labels should then become a justifyContent(FlexAlign.SpaceBetween) on the Row, since the label width is no longer fixed.

### `HW-02-0049` - util is imported through the raw module path while the same file imports common through its kit, contradicting the import form the util reference documents

- Category A, severity low, confidence confirmed
- Features: LIFE-07
- Document: `huawei_industry_tree/02_convenient_life/docs/07_collapse_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/collapse_list-0000002248773270
- Why: Section A of the review requires import paths to match the official reference. Mixing the two forms in one file also defeats the kit-based dependency analysis the build uses to decide which modules to package.
- Fix: Replace line 20 with import { util } from '@kit.ArkTS';. The two call sites, util.TextDecoderOptions and util.TextDecoder.create, are unchanged.

### `HW-02-0050` - The merged bill list is sorted by day only, so it will interleave months as soon as the data covers more than one

- Category B, severity low, confidence confirmed
- Features: LIFE-07
- Document: `huawei_industry_tree/02_convenient_life/docs/07_collapse_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/collapse_list-0000002248773270
- Why: Section B of the review covers logic that is correct only for the sample data. A bill history spans months by definition; with a second month present, day 13 of the older month sorts above day 12 of the newer one, and the page presents an interleaved history with no indication that anything is wrong. The month is also a display string rather than a number, so the comparator cannot simply be extended without changing the model.
- Fix: Add a numeric year and month to ListModelData (or a single ISO date string), populate them in the two rawfiles, and sort on the composite: this.billList.sort((a, b) => b.year - a.year || b.monthIndex - a.monthIndex || b.day - a.day). Keep the 月 suffix as a display-only field.

### `HW-02-0058` - Six @Provide variables are declared on the page but no @Consume exists anywhere in the project

- Category C, severity low, confidence confirmed
- Features: LIFE-08
- Document: `huawei_industry_tree/02_convenient_life/docs/08_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule-0000002284438545
- Why: Section C of the review covers state handling. @Provide registers each variable in the component's provider map so descendants can bind to it, which costs more than @State and signals to a reader that some child depends on the value. Here nothing does; the page is the only consumer of its own data.
- Fix: Replace @Provide with @State on lines 33-38. Nothing else changes, because every read is already this.<name> inside the same component.

### `HW-02-0059` - PickerClass carries @Observed and @Track but is never instantiated - the builder is always called with a plain object literal, so both decorators are inert

- Category C, severity low, confidence confirmed
- Features: LIFE-08
- Document: `huawei_industry_tree/02_convenient_life/docs/08_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule-0000002284438545
- Why: Section C of the review covers state handling. @Observed installs its proxy when an instance is constructed, and @Track narrows which of that instance's properties trigger updates; neither applies to a structurally typed literal. The class therefore reads as a state-management mechanism that is doing work, when it is only a type annotation - and the by-reference parameter passing that actually keeps the six pickers up to date comes from the builder receiving a single object literal, which is a different rule entirely.
- Fix: Replace the class with export interface PickerParam { range: string[]; selected: number; dateIndex: DateIndex; option?: DateTimePickerOption; callback: (dateIndex: DateIndex, value: string | string[], index: number | number[]) => void; } and type the builder against it. The six call sites are unchanged.

### `HW-02-0060` - The displayed time pads minutes to two digits but not hours, so a morning appointment reads 9:05 instead of 09:05

- Category B, severity low, confidence confirmed
- Features: LIFE-08
- Document: `huawei_industry_tree/02_convenient_life/docs/08_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule-0000002284438545
- Why: Section B of the review covers logic defects. The two halves of one timestamp are formatted by different rules, so the field is ragged for the first ten hours of every day and the two rows of the form do not line up. The pad is already written two terms away, which makes the omission an oversight rather than a choice.
- Fix: Extract a helper - pad(n: number): string { return (n < Constants.TEN ? '0' : '') + n; } - and build the string as pad(hours) + ':' + pad(minutes), used by both the start and the end branch.

### `HW-02-0067` - nextNum is incremented and wrapped on every rotation but is never read

- Category B, severity low, confidence confirmed
- Features: LIFE-09
- Document: `huawei_industry_tree/02_convenient_life/docs/09_easylife_loopscroll.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_loopscroll-0000002284598853
- Why: Section B of the review covers dead state. Five lines run inside a timer that fires every 10 ms to maintain a counter nothing consumes, and its presence implies the rotation is tracking a position that some other part of the component depends on - which it is not.
- Fix: Delete the nextNum field and lines 137-141. If a rotation counter is wanted for diagnostics, log it rather than storing it.

### `HW-02-0073` - The query button does nothing at all when no account is entered - no query, no message

- Category B, severity low, confidence confirmed
- Features: LIFE-10
- Document: `huawei_industry_tree/02_convenient_life/docs/10_asset_verification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/asset_verification-0000002257074702
- Why: Section B of the review covers unhandled paths. Tapping 查询 with the account field empty produces no query, no toast and no visual change, so the control appears broken. Both helper functions already support the account-less bulk query the comment describes, so the missing branch is the only thing standing between the sample and the feature it documents.
- Fix: Add the else branch: else { Asset.promptAction($r('app.string.account_prompt'), this.context); } as the minimum, or route it through the same preQuery/auth/query/postQuery sequence with an empty alias to list every stored asset - which is what queryAuthAssetPromise would need extending to support, since it currently sets RETURN_TYPE only inside the same length check.

### `HW-02-0074` - preQueryAssetPromise swallows its error and returns an empty array, so the caller cannot tell a missing asset from a failed call

- Category B, severity low, confidence confirmed
- Features: LIFE-10
- Document: `huawei_industry_tree/02_convenient_life/docs/10_asset_verification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/asset_verification-0000002257074702
- Why: Section B of the review flags catch blocks that discard information. preQuerySync fails for distinguishable reasons - 24000002 for an asset that does not exist, 24000005 for an unsupported authentication type, and general service errors - and the user is told the same thing for all of them. Nothing is logged either, so the cause is not recoverable from a device log.
- Fix: Log in the catch - hilog.error(0x00, 'Asset', 'preQuerySync failed: %{public}s', JSON.stringify(error)); - and either return the code to the caller or show the not-found message directly for 24000002, matching removeAssetPromise.

### `HW-02-0075` - The avoid areas are read once at window-stage creation and never refreshed

- Category B, severity low, confidence confirmed
- Features: LIFE-10
- Document: `huawei_industry_tree/02_convenient_life/docs/10_asset_verification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/asset_verification-0000002257074702
- Why: Section B of the review covers stale state. The page is drawn under the system bars with setWindowLayoutFullScreen(true) at EntryAbility.ets:41, so the stored insets are the only thing keeping the content clear of them. Those insets change on rotation, on a fold, and when the user switches between gesture and three-button navigation; a value captured once at startup silently stops matching the window.
- Fix: Add the subscription next to the two initial reads, writing topRectHeight for TYPE_SYSTEM and bottomRectHeight for TYPE_NAVIGATION_INDICATOR, and pair it with windowClass.off('avoidAreaChange', cb) in onWindowStageDestroy.

### `HW-02-0080` - TodoItemType.isListView is stored on every data entry but never read; the view mode comes from literals at the two call sites

- Category C, severity low, confidence confirmed
- Features: LIFE-11
- Document: `huawei_industry_tree/02_convenient_life/docs/11_card_pinch_scale.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_pinch_scale-0000002282825670
- Why: Section C of the review covers state handling. The field makes the data look as though each card decides its own presentation, when in fact the presentation is a property of the containing view and the data value - false on every entry - would be wrong for the list branch if it ever were read. It also lands in the ForEach's default serialised key, adding a field that must stay constant for the key to stay stable.
- Fix: Delete isListView from Model.ets:21-25 and from all five entries in CardData.ets. The @Prop on CardView and the two literal call sites are unchanged.

### `HW-02-0081` - A TextInputController is constructed on the page although the page has no text input

- Category B, severity low, confidence confirmed
- Features: LIFE-11
- Document: `huawei_industry_tree/02_convenient_life/docs/11_card_pinch_scale.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_pinch_scale-0000002282825670
- Why: Section B of the review covers dead code. A controller is allocated for every instance of the page and bound to nothing, and its presence implies an editable field that does not exist.
- Fix: Delete line 26 and the now-unused TextInputController reference.

### `HW-02-0084` - The LazyForEach in the published snippet supplies no key generator, so it falls back to the index-based default the guidance rules out

- Category E, severity low, confidence confirmed
- Features: LIFE-12
- Document: `huawei_industry_tree/02_convenient_life/docs/12_perpetual_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/perpetual_calendar-0000002319250845
- Why: Section E of the review covers snippets that teach a pattern the referenced documentation contradicts. This one is a month-paging calendar, which is precisely the case where an index-based key misbehaves: 实现思路 step 2 says the source holds 前后2个月的数据 ("the data for the two months either side"), so the window slides as the user swipes and the item at a given index changes month. With the default key the framework treats that as the same item and can reuse the previous month's cells.
- Fix: Close the LazyForEach with a key generator over a stable field - LazyForEach(this.vm.dateListSource, (item: DateModelList) => { ... }, (item: DateModelList) => item.monthKey) - adding that field to DateModelList if it does not already carry one, and do the same for the inner ForEach using the dateModel's YYYY-M-D string, which document line 91 shows is already computed.

### `HW-02-0085` - The 权限说明 section lists only the network permission although the documented module set includes a traffic-restriction feature with a file named LocationPermission.ets

- Category E, severity low, confidence probable
- Features: LIFE-12
- Document: `huawei_industry_tree/02_convenient_life/docs/12_perpetual_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/perpetual_calendar-0000002319250845
- Why: Section E of the review requires the permission list to match the feature set the document describes. Location permissions are user-grant and must be both declared in module.json5 and requested at runtime, so a reader building from this document would find the traffic-restriction module failing with no indication from the document that a second permission was ever needed. The uncertainty is only in which permission: the file name establishes that the module handles one, but with no sample archive attached there is no way to read which constant it requests.
- Fix: Audit the template's module.json5 for its full requestPermissions block and list each entry in 权限说明. If the traffic-restriction module requests a location permission, add it with a note that it is user-grant and needs a runtime request, unlike ohos.permission.INTERNET.

### `HW-02-0097` - The timeline is configured to end at hour 25 and renders a 25:00 row, and the first hour is repeated as a literal where the constant already exists

- Category C, severity low, confidence confirmed
- Features: LIFE-13
- Document: `huawei_industry_tree/02_convenient_life/docs/13_3-day_view_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/3-day_view_calendar-0000002330222589
- Why: Section C of the review covers UI defects and duplicated configuration. 25:00 is not a time; a day view should end at 24:00 (or 23:00 with an inclusive range). More importantly, firstHour is the origin of every card's vertical position - getScheduleTopPosition subtracts it - so the literal 8 in DayColumn and the constant 8 in CalendarConstants must agree or every card shifts by the difference times 52, with the timeline still drawn from the constant.
- Fix: Set DEFAULT_LAST_HOUR to 24, or to 23 if generateHours should stay inclusive of both ends. Replace the literal at DayColumn.ets:28 with CalendarConstants.DEFAULT_FIRST_HOUR, importing the constant there.

### `HW-02-0099` - Six @Provide variables are declared on the page but no @Consume exists anywhere in the project

- Category C, severity low, confidence confirmed
- Features: LIFE-13
- Document: `huawei_industry_tree/02_convenient_life/docs/13_3-day_view_calendar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/3-day_view_calendar-0000002330222589
- Why: Section C of the review covers dead state. @Provide registers each variable in the component's provider map so descendants can bind to it, which costs more than @State and signals to a reader that some child depends on the value. Here nothing does, and the parallel DayInfo path that the header actually uses makes the six look like a superseded design that was left behind.
- Fix: Delete lines 28-35 and the corresponding assignments at lines 51-58. Nothing reads them, and the values they hold are all recoverable from dayDataSource.getData(0).

### `HW-02-0120` - The travel-mode index starts at -1, a value no branch tests for, so an unselected mode silently plans a walking route

- Category C, severity low, confidence confirmed
- Features: LIFE-16
- Document: `huawei_industry_tree/02_convenient_life/docs/16_direction_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/direction_demo-0000002317152262
- Why: Section C of the review covers state handling. The initial value is outside the domain of the three modes but is not treated as such, so any path that reaches getRoutes before a mode has been chosen produces a walking route while none of the three mode buttons is highlighted - the icon comparison at DirectionDemo.ets:349 matches no index. The reader cannot tell from the code that -1 was meant to mean nothing selected, because the else branch consumes it as though it meant walking.
- Fix: Give the mode its own enum and default it to the driving mode, or guard the entry point: if (this.activeRouterClassIndex < 0) { return; } at the top of getRoutes, so an unselected state cannot silently become walking.

### `HW-02-0125` - The back arrow pops the navigation stack and then terminates the ability, so the pop can never be observed

- Category B, severity low, confidence confirmed
- Features: LIFE-17
- Document: `huawei_industry_tree/02_convenient_life/docs/17_dual_list_linkage.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/dual_list_linkage-0000002319372516
- Why: Section B of the review covers logic defects. terminateSelf destroys the UIAbility, so whatever the pop leaves on screen is discarded a moment later - the two statements express two different intentions and only the second takes effect. It also means the two back affordances behave identically, closing the application, when the presence of a NavPathStack implies the arrow was meant to return to the previous destination.
- Fix: Drop the terminateSelf from the arrow handler and keep this.pageInfos.pop(); if closing the application really is intended when the stack is empty, test it first - if (this.pageInfos.size() <= 1) { this.mContext.terminateSelf(); } else { this.pageInfos.pop(); }.

### `HW-02-0126` - The tab ForEach declares its item as string while the array it iterates is Resource[]

- Category C, severity low, confidence confirmed
- Features: LIFE-17
- Document: `huawei_industry_tree/02_convenient_life/docs/17_dual_list_linkage.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/dual_list_linkage-0000002319372516
- Why: Section C of the review covers type correctness in the render path. ForEach's item generator is typed loosely enough that the mismatch compiles, so the declaration is a silent lie: anything written against it that expects string semantics - a comparison, a template interpolation, a .length - would compile and then misbehave on a Resource object at runtime. It also makes the tab list read as though its titles were plain strings when every other title in the project is a resource reference.
- Fix: Correct the parameter type at line 224 to Resource. No other change is needed - Text already accepts it.

### `HW-02-0133` - The initially selected date is a hard-coded pair of array indices rather than today

- Category C, severity low, confidence confirmed
- Features: LIFE-18
- Document: `huawei_industry_tree/02_convenient_life/docs/18_calendar_swiper.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calendar_swiper-0000002355470245
- Why: Section C of the review covers state handling. A calendar that opens on a fixed cell rather than on today reads as an oversight rather than a demo simplification, because everything else in the sample - the shift markers, the agenda dots, the month labels - is driven from real dates in the data. It also means the two indices are not validated against the data at all, so changing the rawfile to a different date range silently changes which day is preselected.
- Fix: After loading the data in aboutToAppear, scan for the entry whose year, month and day match new Date() and set the two indices from it; keep 2 and 3 only as the fallback for a dataset that does not contain today.

### `HW-02-0140` - The project tree misspells the backup-ability directory, and the paste snippet omits the state reset that makes a second paste correct

- Category E, severity low, confidence confirmed
- Features: LIFE-19
- Document: `huawei_industry_tree/02_convenient_life/docs/19_parcel_address_text_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_text_recognition-0000002326921238
- Why: Section E of the review requires the documented tree and snippets to match the zip. The directory name is resolved literally by module.json5, so the spelling in the tree does not build. The omitted reset is not cosmetic either: the extraction only ever assigns fields it finds, so without clearing first a second paste of an address with no company name leaves the previous one in place, and the RichEditor accumulates spans.
- Fix: Correct line 105 to entrybackupability, and replace the step-1 snippet with Index.ets lines 141-171 so the reset, the editor update and the extraction call are all visible.

### `HW-02-0141` - The system pasteboard handle is acquired at module load rather than when it is used

- Category B, severity low, confidence confirmed
- Features: LIFE-19
- Document: `huawei_industry_tree/02_convenient_life/docs/19_parcel_address_text_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/parcel_address_text_recognition-0000002326921238
- Why: Section B of the review covers resource lifetime. Acquiring a system service handle as an import side effect means it is created when the module is first loaded, whether or not the page is ever shown, and held for the life of the process. It also makes the dependency invisible at the call site and untestable, since there is no seam at which a different pasteboard could be supplied.
- Fix: Move it into the handler - const systemPasteboard = pasteboard.getSystemPasteboard(); as the first line of the onClick - or make it a private field of Index initialised in aboutToAppear.

### `HW-02-0148` - MyLocationStyle is given an empty icon string, which is not a valid value for the field's documented string form

- Category A, severity low, confidence confirmed
- Features: LIFE-20
- Document: `huawei_industry_tree/02_convenient_life/docs/20_confirm_direction_in_map_to_rotate_and_move.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/confirm_direction_in_map_to_rotate_and_move-0000002335234358
- Why: Section A of the review requires API parameters to be used as the reference defines them. An empty string is neither a relative rawfile path nor a data URL, so the field is set to a value with no documented meaning rather than left out. The reference states that omitting icon uses the default marker, so the correct way to say "no custom icon" is not to set the property at all - and the surrounding anchorU/anchorV/radiusFillColor settings, which only shape a custom icon, are then also questionable.
- Fix: Drop the icon property from the object literal, or add an asset under entry/src/main/resources/rawfile/ and pass its relative path. The Resource form - icon: $r('app.media.my_location') - has been accepted since 5.0.0(12) and is checked at build time.

### `HW-02-0166` - The logger declares two format identifiers but passes a single array argument, so the second identifier never receives a value

- Category B, severity low, confidence confirmed
- Features: LIFE-22
- Document: `huawei_industry_tree/02_convenient_life/docs/22_id_card_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/id_card_scan-0000002338766518
- Why: Two identifiers are declared and exactly one argument is supplied, and that argument is an array rather than a string. Every call site in the sample passes a tag and a message expecting them to land in the two slots; instead the pair reaches hilog as one value, so the declared contract between the format string and the arguments is broken.
- Fix: Spread the rest parameter into the call: hilog.info(this.domain, this.prefix, this.format, ...args), and do the same in debug, warn and error.

### `HW-02-0167` - Four user-visible strings are hardcoded in the pages while every other string in the same files comes from the resource file

- Category C, severity low, confidence confirmed
- Features: LIFE-22
- Document: `huawei_industry_tree/02_convenient_life/docs/22_id_card_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/id_card_scan-0000002338766518
- Why: The sample sets up the resource mechanism and then bypasses it for the permission-denied toast, the primary action button, the submit toast and the scanner instruction - the four strings a reader is most likely to reuse. Anyone copying the page inherits text that cannot be localized or reviewed with the rest of the copy.
- Fix: Move the four literals into entry/src/main/resources/base/element/string.json and reference them with $r('app.string.<name>') as the surrounding text does.

### `HW-02-0180` - cardDataSource is declared @Provide although no component in the project consumes it

- Category C, severity low, confidence confirmed
- Features: LIFE-23
- Document: `huawei_industry_tree/02_convenient_life/docs/23_card_information_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_information_recognition-0000002377837925
- Why: @Provide sets up an inherited-state channel and pays the observation cost for it. With no consumer the decorator only misleads the reader of a sample page into thinking the recognition buffer is shared with the confirmation component, which is exactly the assumption that makes the hardcoded fields in CardData.ets look intentional.
- Fix: Declare it as a private field: 'private cardDataSource: Record<string, string>[] = [];'.

### `HW-02-0181` - The cardType state is computed when the picker is confirmed and then never read

- Category C, severity low, confidence confirmed
- Features: LIFE-23
- Document: `huawei_industry_tree/02_convenient_life/docs/23_card_information_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/card_information_recognition-0000002377837925
- Why: The page resolves the enum, discards it, ships a display label across the navigation boundary, and makes the destination resolve the same enum again from that label. A reader following the flow sees the lookup twice and cannot tell which one is authoritative, and the display label is now load-bearing routing data.
- Fix: Remove the unused @State cardType and the assignment at :123, or push the resolved CardType as the navigation parameter and have CardRecognitionPage use it directly instead of looking the label up a second time.

### `HW-02-0192` - The logger constructor calls toUpperCase on the format string and throws the result away

- Category B, severity low, confidence confirmed
- Features: LIFE-24
- Document: `huawei_industry_tree/02_convenient_life/docs/24_appointment_service_remind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/appointment_service_remind-0000002378138101
- Why: A statement in a sample that looks deliberate and does nothing. Had it been written as an assignment it would be actively harmful, because uppercasing turns the privacy identifiers into '%{PUBLIC}S' and the hilog reference requires the identifiers in the format string to be the documented ones.
- Fix: Delete line 28.

### `HW-02-0193` - The logger declares two format identifiers but passes a single array argument, so the second identifier never receives a value

- Category B, severity low, confidence confirmed
- Features: LIFE-24
- Document: `huawei_industry_tree/02_convenient_life/docs/24_appointment_service_remind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/appointment_service_remind-0000002378138101
- Why: Two identifiers are declared and exactly one argument is supplied, and that argument is an array rather than a string, so the pair of values every call site passes reaches hilog as a single value and the declared contract between the format string and the arguments is broken.
- Fix: Spread the rest parameter in all four methods: hilog.info(this.domain, this.prefix, this.format, ...args).

### `HW-02-0194` - The permission-on-setting result is logged as success in the branch that runs only when it failed

- Category B, severity low, confidence confirmed
- Features: LIFE-24
- Document: `huawei_industry_tree/02_convenient_life/docs/24_appointment_service_remind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/appointment_service_remind-0000002378138101
- Why: The only diagnostic the permission flow leaves behind states the opposite of what happened. Anyone reading the log while investigating why no calendar reminder was created sees 'requestPermissions2 success' on the run where the user denied the permission.
- Fix: Invert the message and the level: Logger.error('requestPermissionOnSetting denied', JSON.stringify(res)); and raise the rejection log at :125 to error as well.

### `HW-02-0195` - The confirm button silently does nothing when no item type has been selected

- Category B, severity low, confidence confirmed
- Features: LIFE-24
- Document: `huawei_industry_tree/02_convenient_life/docs/24_appointment_service_remind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/appointment_service_remind-0000002378138101
- Why: A capsule button rendered in its enabled style that does nothing when pressed reads as a broken application. The sibling pages behave the same way by accident rather than design - PaySelectPage pops whatever is selected, including an empty string - so the sample never shows the reader how to handle an incomplete sheet.
- Fix: Add an else that shows a prompt, or disable the button while selectedIndex is -1 with '.enabled(this.selectedIndex >= 0)'.

### `HW-02-0196` - The send-or-receive tab index drives a conditional background colour but is a plain field rather than @State

- Category C, severity low, confidence confirmed
- Features: LIFE-24
- Document: `huawei_industry_tree/02_convenient_life/docs/24_appointment_service_remind.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/appointment_service_remind-0000002378138101
- Why: An undecorated field is read to decide a rendered colour, which is the exact shape that produces a silent no-op the first time somebody wires the tabs up: assigning selectedIndex from a click handler will change the value and leave the UI as it was. Placing it unmarked in the middle of a decorated field block also makes the omission look deliberate.
- Fix: Decorate it - '@State selectedIndex: number = 0;' - and add '.onClick(() => { this.selectedIndex = index; })' to the tab labels, or remove the conditional colour if the strip is meant to be static.

### `HW-02-0207` - The two sentences the feature exists to produce are hardcoded Chinese literals with the destination name baked in

- Category C, severity low, confidence confirmed
- Features: LIFE-25
- Document: `huawei_industry_tree/02_convenient_life/docs/25_commuting_calculation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/commuting_calculation-0000002347813672
- Why: Changing the destination means changing two constants in one file and a Chinese string literal in another, with nothing linking them - the next reader who updates the coordinates gets a page that confidently names the wrong landmark. The surrounding labels already come from string.json, so the literals are an inconsistency inside a single build method rather than a deliberate simplification.
- Fix: Move both sentences into string.json with placeholders, and add a name field next to the coordinates in StyleConstants so the label and the position change together.

### `HW-02-0208` - The home page carries a Navigation stack nothing pushes to and computes a window height nothing reads

- Category C, severity low, confidence confirmed
- Features: LIFE-25
- Document: `huawei_industry_tree/02_convenient_life/docs/25_commuting_calculation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/commuting_calculation-0000002347813672
- Why: Both are scaffolding left behind from a template. The Navigation is the more misleading of the two: the document's project tree presents HomePage, housingResourcesPage and StaticMap as three pages, and the Navigation on the home page suggests a routing structure that does not exist - the other two are components in a tab and a child component respectively.
- Fix: Drop the Navigation wrapper and the pathStack field if no route is ever pushed, and delete windowHeight together with the aboutToAppear that computes it.

### `HW-02-0210` - The project tree spells the housing detail page with a lowercase initial while the shipped file is capitalised

- Category E, severity low, confidence confirmed
- Features: LIFE-25
- Document: `huawei_industry_tree/02_convenient_life/docs/25_commuting_calculation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/commuting_calculation-0000002347813672
- Why: The project builds with case-sensitivity checking switched on - CommutingCalculation.zip#build-profile.json5:13-14 sets "strictMode": { "caseSensitiveCheck": true } - and a reader reconstructing the tree from the document creates a file whose name does not match the import in HomePage.ets. It is also the only one of the three page entries that does not match the ZIP.
- Fix: Correct the entry to HousingResourcesPage.ets so all three page names in 工程目录 match the shipped files.

### `HW-02-0219` - The reverse geocoding result is indexed at element zero without checking that the array has any entries

- Category B, severity low, confidence confirmed
- Features: LIFE-26
- Document: `huawei_industry_tree/02_convenient_life/docs/26_list_of_nearby_outlets.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/list_of_nearby_outlets-0000002347931784
- Why: A reverse geocoding call that succeeds but resolves no address returns an empty array rather than an error, so the else branch runs and geoAddress[0] is undefined. Reading .placeName from it throws inside the callback, where the surrounding try has already returned and nothing catches it.
- Fix: Guard the array before use, and leave placeName at its 'fetching location' default when no address comes back.

### `HW-02-0220` - On a device with no voice capability the phone row does nothing and says nothing

- Category B, severity low, confidence confirmed
- Features: LIFE-26
- Document: `huawei_industry_tree/02_convenient_life/docs/26_list_of_nearby_outlets.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/list_of_nearby_outlets-0000002347931784
- Why: The sample declares device types phone, tablet and 2in1 in its module.json5, and on the two that have no cellular voice the tap is a complete no-op - no dialler, no message, not even a log line. The guard is correct; what is missing is any outcome on the branch it exists to protect.
- Fix: Add the else: 'this.promptAction.showToast({ message: $r('app.string.no_voice_capability') });', and hide or disable the phone row entirely when hasVoiceCapability() is false.

### `HW-02-0221` - A hilog call passes the error message in the tag slot and the rest of the message in the format slot

- Category A, severity low, confidence confirmed
- Features: LIFE-26
- Document: `huawei_industry_tree/02_convenient_life/docs/26_list_of_nearby_outlets.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/list_of_nearby_outlets-0000002347931784
- Why: The sentence that identifies the failure is placed where the tag belongs, so it is subject to the documented 31-byte truncation and it destroys the tag that log filtering relies on - every permission-check failure arrives under a different tag. What is left in the format slot is a fragment beginning with a space.
- Fix: Move the whole message into the format parameter and give the tag a stable value, matching the pattern the same file already uses at :41 and :44.

### `HW-02-0222` - The status line, the business-hours label and the navigate label are hardcoded Chinese while the rest of the page uses resource strings

- Category C, severity low, confidence confirmed
- Features: LIFE-26
- Document: `huawei_industry_tree/02_convenient_life/docs/26_list_of_nearby_outlets.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/list_of_nearby_outlets-0000002347931784
- Why: The four literals are the entire status and action vocabulary of the screen - what is happening, how many results there are, what the times mean and what the button does. Everything around them already comes from string.json, so the inconsistency sits inside a single build method rather than reflecting a decision not to localize the sample.
- Fix: Move the four strings into string.json and reference them with $r, using a format placeholder for the count as the surrounding labels do.

### `HW-02-0233` - The project ships a string resource file and uses it for nothing, hardcoding every user-visible string instead

- Category C, severity low, confidence confirmed
- Features: LIFE-27
- Document: `huawei_industry_tree/02_convenient_life/docs/27_easylife_demo_vrhouse.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_demo_vrhouse-0000002349301462
- Why: The resource mechanism is set up and then used for nothing, so the sample demonstrates the opposite of the practice its own project structure implies. Media and colour resources are referenced through $r throughout the same files, which makes the omission specific to text rather than a deliberate simplification of the whole sample.
- Fix: Move the user-visible strings into entry/src/main/resources/base/element/string.json and reference them with $r, as the media and colour resources in the same files already are.

### `HW-02-0242` - Two dialog buttons are used through the implicit element-id global while the eight other elements are looked up explicitly

- Category B, severity low, confidence confirmed
- Features: LIFE-28
- Document: `huawei_industry_tree/02_convenient_life/docs/28_h5rechargeplatform.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5rechargeplatform-0000002387265697
- Why: Two of the ten elements in one script are reached a different way from the other eight, and the way they are reached depends on an implicit binding rather than on the lookup the rest of the file demonstrates. A reader copying the block sees eight explicit lookups and assumes the remaining two identifiers were declared somewhere they cannot find.
- Fix: Add the two missing getElementById declarations alongside the other eight at the top of the script.

### `HW-02-0243` - The success dialog looks up a phone-number element it never fills in

- Category B, severity low, confidence confirmed
- Features: LIFE-28
- Document: `huawei_industry_tree/02_convenient_life/docs/28_h5rechargeplatform.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5rechargeplatform-0000002387265697
- Why: The recharge confirmation shows the amount but never the number being topped up, although the element and its lookup exist for exactly that. The dead declaration is the visible half of a missing assignment rather than harmless leftover code.
- Fix: Set it in the same handler that sets successAmount: 'successPhone.textContent = document.getElementById('phone').value;', or delete the unused lookup.

### `HW-02-0244` - Two hilog calls declare one format identifier and pass two arguments, so the serialised payload is dropped

- Category B, severity low, confidence confirmed
- Features: LIFE-28
- Document: `huawei_industry_tree/02_convenient_life/docs/28_h5rechargeplatform.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5rechargeplatform-0000002387265697
- Why: One identifier is declared and two arguments are supplied, so the launch parameters and the want - the only two values worth logging in onCreate - never appear in the output. The try/catch wrapped around the two calls (:23-28) suggests they were expected to carry real payloads.
- Fix: Declare a second identifier in the format string, or fold the serialised value into a single interpolated argument.

### `HW-02-0246` - This document replaces the industry's constraints section with an environment section and drops the supported SDK line

- Category E, severity low, confidence confirmed
- Features: LIFE-28
- Document: `huawei_industry_tree/02_convenient_life/docs/28_h5rechargeplatform.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5rechargeplatform-0000002387265697
- Why: This is the one document in the industry whose sample needs a newer API level, and it is the one that omits the SDK line its siblings all carry. A reader comparing documents sees a different heading and two bullets instead of three, and has to open build-profile.json5 to learn which HarmonyOS SDK the higher API version corresponds to.
- Fix: Use the 约束与限制 heading and add the SDK bullet naming the HarmonyOS release that provides API Version 24, so the requirement is visible without opening the ZIP.

### `HW-02-0250` - The circle-drawing failure is logged as a marker icon builder test error

- Category B, severity low, confidence confirmed
- Features: LIFE-29
- Document: `huawei_industry_tree/02_convenient_life/docs/29_set_coverage_area.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_coverage_area-0000002547690109
- Why: The only diagnostic emitted when the coverage circle fails to draw names a different feature and calls itself a test. Anyone searching the log for why no circle appeared will not find this line, and anyone who does find it will look for a marker icon builder that does not exist in the project.
- Fix: Correct the message to name the operation, matching the pattern the annotation handler already uses at :158.

### `HW-02-0251` - The drag start and drag handlers are two identical six-line blocks

- Category B, severity low, confidence confirmed
- Features: LIFE-29
- Document: `huawei_industry_tree/02_convenient_life/docs/29_set_coverage_area.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_coverage_area-0000002547690109
- Why: The duplication is the reason the two handlers can drift apart, and it doubles the surface of the redraw problem reported for Index.ets:103 - both blocks fire the same unawaited pair. Calling getPosition() six times per frame for one value compounds it on the hottest path in the sample, since markerDrag fires per frame.
- Fix: Extract one callback that reads the position once into a local and register it for both events: 'let onDrag = (marker: map.Marker) => { let p = marker.getPosition(); this.markerLatLng = p; this.changeRadius(p.latitude, p.longitude); this.movePointAnnotation(p.latitude, p.longitude, this.radius); };'.

### `HW-02-0252` - The only text the feature renders is a hardcoded Chinese literal built by concatenation

- Category C, severity low, confidence confirmed
- Features: LIFE-29
- Document: `huawei_industry_tree/02_convenient_life/docs/29_set_coverage_area.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_coverage_area-0000002547690109
- Why: The radius label is the entire textual output of the feature - it is what the point annotation exists to display - and it is assembled from two Chinese literals around a number. The same file already reaches resources through $r for its icon, so the omission is specific to text.
- Fix: Move the label into string.json with a placeholder and format it through the resource manager, keeping the numeric value as the only thing computed at runtime.

### `HW-02-0262` - Executors are cached in a static map that is never pruned, and cancelling a pool leaves its entry behind

- Category B, severity low, confidence confirmed
- Features: LIFE-30
- Document: `huawei_industry_tree/02_convenient_life/docs/30_thumbnail_taskpool.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/thumbnail_taskpool-0000002338966908
- Why: The map is static, so it outlives every page that used it. Leaving the executor behind also keeps its configuration: a pool created with maxRunningTask 3 keeps that cap forever, so a later caller passing a different value to execute silently gets the original executor and the original limits, since :90 only constructs one when the lookup misses.
- Fix: Remove the entry after cancelling - 'TaskManager.executorMap.remove(poolName);' - so the next execute for that pool name builds an executor with the caller's parameters.

### `HW-02-0263` - The logger declares two format identifiers and passes one array, and one call site supplies a single argument

- Category B, severity low, confidence confirmed
- Features: LIFE-30
- Document: `huawei_industry_tree/02_convenient_life/docs/30_thumbnail_taskpool.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/thumbnail_taskpool-0000002338966908
- Why: Two identifiers are declared and exactly one argument is supplied, and that argument is an array rather than a string, so the declared contract between the format string and the arguments is broken for every call. The single-argument call site compounds it: the one message the thumbnail worker emits when generation fails arrives with no tag identifying where it came from.
- Fix: Spread the rest parameter in all four methods, and give the thumbnail worker's error call the same tag-first shape the other call sites use.

### `HW-02-0264` - The onReceiveData failure is logged as a cancel task error

- Category B, severity low, confidence confirmed
- Features: LIFE-30
- Document: `huawei_industry_tree/02_convenient_life/docs/30_thumbnail_taskpool.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/thumbnail_taskpool-0000002338966908
- Why: onReceiveData is how the sandbox query task streams its batches back to the page - QuerySandBoxFile.ets:33 registers the page callback through it - so a failure here means the file list never arrives. The one log line that would explain it names cancellation instead, and both messages come from the same tag, so they are indistinguishable in the log.
- Fix: Correct the message to name the operation the catch actually guards.

### `HW-02-0268` - 1 sample project depends on third-party packages through unpinned version ranges

- Category B, severity low, confidence confirmed
- Features: LIFE-01
- Document: `huawei_industry_tree/02_convenient_life/docs/01_practice-convenient-life-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1-0000001952539489
- Why: A caret or tilde range lets ohpm resolve a different version than the one the sample was verified against, so the published sample is not reproducible and can break without any change to its own source. Sample projects are reference implementations; their dependency set should be exactly the one that was tested.
- Fix: Pin exact versions in oh-package.json5 dependencies, and record the resolved set in oh-package-lock.json5.

