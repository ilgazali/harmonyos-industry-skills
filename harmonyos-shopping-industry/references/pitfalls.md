# Pitfalls

> Generated from `features/*.md`. Source industry: `16_shopping`, 24 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (2)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-16-0013` - Systematic pattern: abridged doc snippets are cut mid-construct and no longer parse — this doc's step-2 snippet loses the onClick closing braces

- Category E, severity medium, confidence confirmed
- Features: SHOP-01, SHOP-04, SHOP-05, SHOP-07, SHOP-10, SHOP-12, SHOP-14, SHOP-23
- Document: `huawei_industry_tree/16_shopping/docs/12_search_history_expand_and_collapse.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_history_expand_and_collapse-0000002343696461
- Why: The abridgement step that produces the published snippets removes structurally necessary code (closing braces, catch blocks, async modifiers, returns) rather than eliding only bodies, so a large share of the corpus' copy-paste snippets fail to parse. Reporting it as one systematic defect matches the shared root cause in the excerpt pipeline.
- Fix: Regenerate excerpts with brace-balanced elision (keep catch/async/return skeletons); for this doc, close the onClick handler before the Flex block.

### `HW-16-0027` - Systematic: 19 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: SHOP-01, SHOP-03, SHOP-05, SHOP-06, SHOP-07, SHOP-08, SHOP-11, SHOP-12, SHOP-13, SHOP-14, SHOP-15, SHOP-16, SHOP-17, SHOP-18, SHOP-19, SHOP-20, SHOP-21, SHOP-22, SHOP-23
- Document: `huawei_industry_tree/16_shopping/docs/23_address_manager.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/address_manager-0000002366628616
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (33)

### `HW-16-0001` - AudioRecorder closes the recording file immediately after opening it, then hands the closed fd to AVRecorder

- Category B, severity high, confidence confirmed
- Features: SHOP-01
- Document: `huawei_industry_tree/16_shopping/docs/01_practice-purchase-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-purchase-app-architecture-v1-0000002077367333
- Why: AVRecorder must write into a valid open fd for the whole recording session; recording into a closed descriptor fails or produces an empty file, so the voice-message feature cannot actually persist audio.
- Fix: Keep the fs.File open (store it on the class), remove the finally-close, and close the fd in stopRecordingProcess after release(); rename the file to 01.m4a.

### `HW-16-0014` - Thumbnail flow calls photoAccessHelper.getAssets, which requires the restricted READ_IMAGEVIDEO permission the sample never declares

- Category D, severity high, confidence confirmed
- Features: SHOP-13
- Document: `huawei_industry_tree/16_shopping/docs/13_evaluation_after_received.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/evaluation_after_received-0000002356153881
- Why: At runtime getAssets fails with a permission error, so no thumbnail is ever produced; worse, READ_IMAGEVIDEO is a restricted (ACL) permission that app-market apps generally cannot obtain for this use case — the correct pattern for picker-selected images is reading the returned URI (e.g. image.createImageSource) which needs no permission.
- Fix: Replace the getAssets/getThumbnail chain with image.createImageSource(uri) + createPixelMap with desiredSize, dropping the permission-gated API.

### `HW-16-0034` - The home page search box cannot accept input: its value is bound to an undecorated field with no onChange, and the search button always submits the carousel keyword instead of what the user typed

- Category B, severity high, confidence confirmed
- Features: SHOP-20
- Document: `huawei_industry_tree/16_shopping/docs/20_search_recommendation_word_carousel.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_recommendation_word_carousel-0000002348742090
- Why: Typing a query into the search box on the home page and pressing 搜索 runs a search for whichever recommendation the carousel happens to be showing. The typed text is discarded twice over: no onChange writes it, and the button never reads it. Because changeValue is also undecorated, an assignment would not re-render the Search either, so the box cannot even display what was typed. The document presents this page as the implementation of the feature, and the same project already shows the correct pattern on SearchPage, so the two pages in one sample contradict each other on the component they both use.
- Fix: Decorate changeValue with @State, add an onChange handler that assigns it, and in the button use this.changeValue when it is non-empty and fall back to this.placeholderList[this.index] otherwise, matching the onSubmit logic already written in SearchPage.ets:62-64.

### `HW-16-0002` - Amplitude polling interval leaks and dereferences a released AVRecorder when stop is called outside started/paused states

- Category B, severity medium, confidence confirmed
- Features: SHOP-01
- Document: `huawei_industry_tree/16_shopping/docs/01_practice-purchase-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-purchase-app-architecture-v1-0000002077367333
- Why: If stop is reached in any other state (e.g. prepare failed after the interval was scheduled, or stop called twice), the interval keeps firing forever and each tick evaluates `this.avRecorder!` on undefined, throwing a TypeError; the non-null assertion hides the hazard from the compiler.
- Fix: Move clearInterval(this.time) out of the state check (first line of stopRecordingProcess) and guard the interval callback with `if (!this.avRecorder) return;`.

### `HW-16-0003` - Restricted READ/WRITE_IMAGEVIDEO permissions are declared; WRITE is used by the photo-save flow but never requested at runtime, READ is not needed at all

- Category D, severity medium, confidence confirmed
- Features: SHOP-01
- Document: `huawei_industry_tree/16_shopping/docs/01_practice-purchase-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-purchase-app-architecture-v1-0000002077367333
- Why: Both are restricted (ACL) permissions ordinary apps cannot ship with; declaring them without requesting means the user-grant permission is never granted, so applyChanges fails and the take-photo-and-save flow cannot work — while the unused READ declaration violates least privilege. The correct pattern is the security-component/authorization-dialog save flow that needs no restricted permission.
- Fix: Remove both declarations and switch saveCameraPhoto to the authorization-dialog based save API.

### `HW-16-0006` - getDeviceStatus keeps executing after resolve(), so a device that fails verification is still marked as having collected the coupon

- Category B, severity medium, confidence confirmed
- Features: SHOP-05
- Document: `huawei_industry_tree/16_shopping/docs/05_get_coupons.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/get_coupons-0000002256272966
- Why: A promise resolves only once, so the caller sees the failure result while the side effect (marking the device as collected) happens anyway — against a real Device Security server this burns the one-per-device coupon for devices that never passed verification, and wastes the extra requests in every early-exit path.
- Fix: Add `return;` after every early resolve (or restructure as sequential `return 'X';` statements in an async method instead of the new-Promise wrapper).

### `HW-16-0012` - Skeleton shimmer mutates translateX outside the animateTo closure, so per the API contract no looping animation is produced

- Category C, severity medium, confidence confirmed
- Features: SHOP-11
- Document: `huawei_industry_tree/16_shopping/docs/11_skeleton_screen.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/skeleton_screen-0000002294856764
- Why: The documented contract animates only state changed inside the closure; with an empty closure the infinite (iterations: -1) shimmer sweep the doc promises has nothing to animate, and the highlight bar just jumps to its end position.
- Fix: this.getUIContext().animateTo({ duration: this.duration, delay: this.delay, iterations: -1, curve: Curve.EaseInOut }, () => { this.translateX = '100%'; });

### `HW-16-0015` - Packed JPEG bytes are decoded as UTF-8 text and stored as the image payload, corrupting the binary data

- Category B, severity medium, confidence confirmed
- Features: SHOP-13
- Document: `huawei_industry_tree/16_shopping/docs/13_evaluation_after_received.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/evaluation_after_received-0000002356153881
- Why: JPEG is binary; decoding it as UTF-8 replaces invalid byte sequences irreversibly, so the stored 'imageArrayBuffer' string can never be turned back into the original image — any upload/persist built on this field ships corrupted data. The doc presents this flow as the way to 'convert image data for display'.
- Fix: Store the ArrayBuffer directly in ImageInfo, or use util.Base64Helper().encodeToString(new Uint8Array(data)); fix the loop guard to `if (item)`.

### `HW-16-0017` - Browsing history grows without dedup or size cap and is persisted via PersistentStorage, against the documented 2 KB guidance

- Category B, severity medium, confidence confirmed
- Features: SHOP-15
- Document: `huawei_industry_tree/16_shopping/docs/15_offering_browsing_history.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/offering_browsing_history-0000002361003509
- Why: A browsing-history array is unbounded by nature: repeated visits store the same product many times and the array is re-serialized to disk on the UI thread on every tap — exactly the pattern the official guide tells developers to avoid; as a template it teaches PersistentStorage for a use case Huawei's own docs route to the database APIs.
- Fix: Filter out an existing entry (by id) before unshift, cap the array (e.g. 20 items), or move the history to @ohos.data.preferences / relationalStore as the guide recommends.

### `HW-16-0018` - Countdown decrements a stored gap by the timer period instead of recomputing from the clock, so it drifts and lags after backgrounding

- Category B, severity medium, confidence probable
- Features: SHOP-16
- Document: `huawei_industry_tree/16_shopping/docs/16_booking_to_grab_tickets.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/booking_to_grab_tickets-0000002304538948
- Why: setInterval ticks are not exact and are throttled or paused while the app is in background; every missed or delayed tick makes the displayed countdown later than reality. For the exact use case the doc sells — a ticket-grab start time — the user returns to the app and sees time remaining although the sale already started.
- Fix: In the interval callback set `this.gap = this.ticketDate.getTime() - Date.now();` before calculateTime().

### `HW-16-0023` - Every product radio in the compare sheet shares value 'Radio1' and one global group, so selections are indistinguishable and clash with other radio clusters

- Category B, severity medium, confidence confirmed
- Features: SHOP-21
- Document: `huawei_industry_tree/16_shopping/docs/21_goods_pk.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/goods_pk-0000002349269256
- Why: Radios are identified by value within their group: duplicate values make the framework unable to distinguish which product is selected, and sharing one group name across independent sections means selecting a product can clear the selection of the spec radios elsewhere on the page — the selection model only appears to work because onChange writes goodsItem.id manually.
- Fix: Use value: goodsItem.id.toString() with a dedicated group like 'pkGoodsGroup', and separate group names for the other clusters.

### `HW-16-0028` - 1 sample project declares permissions that no code path in the project uses

- Category D, severity medium, confidence confirmed
- Features: SHOP-05
- Document: `huawei_industry_tree/16_shopping/docs/05_get_coupons.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/get_coupons-0000002256272966
- Why: A permission declared in module.json5 but never referenced by any .ets or .ts file in the project cannot be exercised, so it is pure over-declaration. Over-declared permissions are a release-review rejection reason, and restricted permissions especially so. Because these module.json5 files are copied as templates, the surplus entries propagate into products that have even less claim to them.
- Fix: Delete every declared permission the code does not exercise. Where the capability is reached through a permission-free path such as SaveButton or PhotoViewPicker, no declaration is needed at all.

### `HW-16-0029` - onOffsetChange writes an @State variable on every drag frame that nothing ever reads, so the documented offset listener is pure overhead

- Category C, severity medium, confidence confirmed
- Features: SHOP-06
- Document: `huawei_industry_tree/16_shopping/docs/06_pull_to_jump.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pull_to_jump-0000002282295284
- Why: onOffsetChange fires continuously while the user drags, so this writes an observed state variable dozens of times per second and makes the framework run its dependency check and re-render pass each time, for a value that reaches no component. The document presents onOffsetChange as one of the two mechanisms that implement the feature, so a reader copies it believing it is load bearing, when in this sample the pull distance is used by nothing and the jump is performed by onRefreshing.
- Fix: Delete pullOffset and the onOffsetChange handler, or make the custom refresh builder actually use the offset. Correct step 2 so it names onRefreshing as the callback that performs the jump.

### `HW-16-0031` - The avoidAreaChange listener registered in onWindowStageCreate is never released in onWindowStageDestroy

- Category B, severity medium, confidence confirmed
- Features: SHOP-06
- Document: `huawei_industry_tree/16_shopping/docs/06_pull_to_jump.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pull_to_jump-0000002282295284
- Why: The listener is bound to the main window and outlives the window stage. Its closure holds the window, and relaunching within the same process adds a second listener on top of the first, so every avoid-area change writes the AppStorage key more than once and drives a duplicate re-layout of the tab bar padding.
- Fix: Keep the callback in a class field and call windowClass.off('avoidAreaChange', callback) in onWindowStageDestroy.

### `HW-16-0032` - The unread badge appends a plus sign to every count except one, so two unread messages render as 2+

- Category B, severity medium, confidence confirmed
- Features: SHOP-18
- Document: `huawei_industry_tree/16_shopping/docs/18_clear_unread_messages.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/clear_unread_messages-0000002341399506
- Why: The condition tests for exactly one, not for a cap, so every count from 2 upward is rendered with a trailing plus: two unread messages show as 2+, five as 5+. A plus means at least this many, so the badge overstates a count it knows exactly. The seed data is the reason this looks right in the demo: it contains only 1 and 99, the two values for which the expression happens to produce sensible output. Any real message list hits the wrong branch immediately, and this sample exists specifically to demonstrate unread-count handling.
- Fix: Compare against a display cap: render item.numbers when it is at most the cap and cap + '+' when it exceeds it. Replace JUDGE_MESSAGE_IF_ONE in this expression with a MESSAGE_COUNT_MAX constant.

### `HW-16-0033` - The long-press menu labels and the message timestamps are hardcoded Chinese literals while every other user-visible string in the sample uses a resource reference

- Category B, severity medium, confidence confirmed
- Features: SHOP-18
- Document: `huawei_industry_tree/16_shopping/docs/18_clear_unread_messages.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/clear_unread_messages-0000002341399506
- Why: The four hardcoded items are the long-press menu, which is one of the two interactions this document exists to demonstrate, so the strings a reader most wants to reuse are the ones that cannot be translated. Mixing the two styles in a single file is also misleading: the surrounding $r calls suggest the sample is fully localised, so a developer copying it will not notice that four labels and every timestamp are pinned to Chinese.
- Fix: Move the four menu labels into string resources and build dialogList from $r references, and supply the timestamps as ResourceStr like the other fields of MessageInfo already are.

### `HW-16-0035` - The avoidAreaChange listener registered in onWindowStageCreate is never released in onWindowStageDestroy

- Category B, severity medium, confidence confirmed
- Features: SHOP-20
- Document: `huawei_industry_tree/16_shopping/docs/20_search_recommendation_word_carousel.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_recommendation_word_carousel-0000002348742090
- Why: The listener is bound to the main window and outlives the window stage. Its closure holds the window, and relaunching within the same process stacks a second listener on the first, so every avoid-area change writes both AppStorage keys more than once and drives duplicate re-layouts of the page padding.
- Fix: Keep the callback in a class field and call windowClass.off('avoidAreaChange', callback) in onWindowStageDestroy.

### `HW-16-0004` - Document promises a dashed coupon divider but the sample draws a solid border; project tree also omits the constants directory

- Category E, severity low, confidence confirmed
- Features: SHOP-03
- Document: `huawei_industry_tree/16_shopping/docs/03_coupons_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/coupons_page-0000002235638646
- Why: The doc's central claim for this scenario (dashed coupon edge) is not what the shipped code produces, and the incomplete tree hides a file the page imports.
- Fix: Set style: { end: BorderStyle.Dashed } (localized key to match start/end widths) or update the doc text; add the constants directory to the tree.

### `HW-16-0005` - Doc code snippet is a syntax error (try without catch) and the project tree lists a Mine.ets page that does not exist in the zip

- Category E, severity low, confidence confirmed
- Features: SHOP-04
- Document: `huawei_industry_tree/16_shopping/docs/04_sign.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sign-0000002284183885
- Why: A try block without catch/finally is invalid TypeScript/ArkTS, so copy-pasting the documented snippet fails immediately; the tree references a file readers cannot find.
- Fix: Extend the snippet through the catch block and remove (or correct) the Mine.ets entry in 工程目录.

### `HW-16-0007` - Doc snippet of getDeviceStatus does not compile and does not match the sample; tree lists CouponInfo.ts instead of .ets

- Category E, severity low, confidence confirmed
- Features: SHOP-05
- Document: `huawei_industry_tree/16_shopping/docs/05_get_coupons.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/get_coupons-0000002256272966
- Why: The snippet is the doc's core teaching content; as printed it neither compiles nor reflects the sample's sequential logic.
- Fix: Replace the snippet with the zip's awaited implementation and fix the tree entry.

### `HW-16-0008` - getPixmapFromMedia snippet has broken syntax and documents an async API the sample does not use

- Category E, severity low, confidence confirmed
- Features: SHOP-07
- Document: `huawei_industry_tree/16_shopping/docs/07_scratch_effect.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/scratch_effect-0000002320935777
- Why: The snippet is the doc's step-1 teaching content; as printed it is a syntax error and it names a different API (async getMediaContent) than the shipped sample (getMediaContentSync).
- Fix: Replace the call with `getMediaContentSync(resource.id)` (or a valid `getMediaContent(resource.id)` await form) in the doc.

### `HW-16-0009` - Doc links the util Stack data structure instead of the ArkUI Stack layout component

- Category E, severity low, confidence confirmed
- Features: SHOP-08
- Document: `huawei_industry_tree/16_shopping/docs/08_product_card_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/product_card_demo-0000002324599805
- Why: A reader following the link lands on an unrelated collections API; the sample never imports @ohos.util.Stack.
- Fix: Replace both links with the ts-container-stack reference.

### `HW-16-0010` - History deduplication splices the array while iterating it with forEach, skipping elements after a removal

- Category B, severity low, confidence confirmed
- Features: SHOP-09
- Document: `huawei_industry_tree/16_shopping/docs/09_search_history.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_history-0000002328895045
- Why: Removing elements from the array being forEach-iterated shifts the remaining items left, so the element after any removed entry is never visited — if the history ever contains two matching entries (e.g. seeded data or a prior missed dedup), consecutive duplicates survive, and the pattern silently corrupts iteration for anyone copying the template.
- Fix: this.historyWords = this.historyWords.filter(item => item.word !== wordModel.word); then unshift.

### `HW-16-0011` - Doc snippet names the page MainMallPage.ets while the tree and the zip use MallMainPage.ets

- Category E, severity low, confidence confirmed
- Features: SHOP-10
- Document: `huawei_industry_tree/16_shopping/docs/10_scroll_celling_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/scroll_celling_demo-0000002331639969
- Why: The two names in the same document contradict each other; readers searching for MainMallPage.ets in the sample find nothing.
- Fix: Correct the snippet comment to MallMainPage.ets.

### `HW-16-0016` - Off-by-one in last-page line count: formula subtracts 6 although the first page holds 5 items

- Category B, severity low, confidence confirmed
- Features: SHOP-14
- Document: `huawei_industry_tree/16_shopping/docs/14_auto_height_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/auto_height_list-0000002359342093
- Why: With the shipped 15 items both formulas coincidentally give 2, but any data size where (n-5) crosses a multiple of 5 (e.g. 16 items → 3 real rows vs computed 2) makes containerMaxHeight one row too small, clipping the last menu row when expanded.
- Fix: Change the constant 6 to 5 (and derive both 5s from a shared PAGE_ITEM_COUNT constant).

### `HW-16-0019` - Every booking tap creates the 'MyCalendar' calendar account again instead of reusing an existing one

- Category B, severity low, confidence probable
- Features: SHOP-16
- Document: `huawei_industry_tree/16_shopping/docs/16_booking_to_grab_tickets.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/booking_to_grab_tickets-0000002304538948
- Why: Repeat bookings either fail at createCalendar (account already exists) so no event is added, or pile up duplicate calendar accounts — either way the second and later reminders are unreliable; the discarded eventId also makes later updates/deletions impossible.
- Fix: Try getCalendar(calendarAccount) first and fall back to createCalendar; persist eventId for update/delete.

### `HW-16-0020` - Project tree mislabels Index.ets as the 'coupon page' and lists Constant.ets with wrong letter case

- Category E, severity low, confidence confirmed
- Features: SHOP-17
- Document: `huawei_industry_tree/16_shopping/docs/17_red_envelope_rain.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/red_envelope_rain-0000002370873497
- Why: Wrong comment misdescribes the sample's main page; the case difference breaks the reference on case-sensitive builds (the sample projects enable caseSensitiveCheck).
- Fix: Rename entry to constant.ets and change the comment to 红包雨页面.

### `HW-16-0021` - Animators are never cancelled when the page is left mid-rain

- Category B, severity low, confidence probable
- Features: SHOP-17
- Document: `huawei_industry_tree/16_shopping/docs/17_red_envelope_rain.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/red_envelope_rain-0000002370873497
- Why: Leaving the page during the rain keeps 20 onFrame callbacks firing on destroyed UI state and leaks the interval until its own countdown ends; frame-animation samples should demonstrate cancellation as part of the lifecycle.
- Fix: Add aboutToDisappear(): iterate redEnvs calling fallAnimation?.cancel(), set them undefined, and clearInterval the countdown timer.

### `HW-16-0022` - Project tree names the directory 'component' while the zip uses 'components'

- Category E, severity low, confidence confirmed
- Features: SHOP-19
- Document: `huawei_industry_tree/16_shopping/docs/19_sales_update_roll.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/sales_update_roll-0000002380510193
- Why: The documented structure must match the sample project.
- Fix: Rename the tree entry to components.

### `HW-16-0024` - Second-floor height is hardcoded to 760vp and the fade formula to 690/70, ignoring the actual window height the sample itself stores

- Category B, severity low, confidence probable
- Features: SHOP-22
- Document: `huawei_industry_tree/16_shopping/docs/22_second_floor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/second_floor-0000002386145197
- Why: On windows shorter or taller than 760vp (foldables, tablets, split screen) the second floor either overflows or leaves a gap, and the search-bar fade band no longer aligns with the pull range; the constants are silently coupled and break together if either changes.
- Fix: Set floorHeight from screenHeight (px2vp) and compute the fade as (Math.abs(this.offsetY) - (this.floorHeight - FADE_BAND)) / FADE_BAND.

### `HW-16-0025` - getCurrentLocation snippet uses await in a non-async callback and drops the permission check; Pasteboard links point to the C API

- Category E, severity low, confidence confirmed
- Features: SHOP-23
- Document: `huawei_industry_tree/16_shopping/docs/23_address_manager.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/address_manager-0000002366628616
- Why: The snippet neither compiles nor reflects the sample's permission handling, and the reference link sends ArkTS readers to NDK documentation.
- Fix: Update the step-4 snippet from DetailAddressView.ets and fix both Pasteboard links.

### `HW-16-0026` - 1 sample project depends on third-party packages through unpinned version ranges

- Category B, severity low, confidence confirmed
- Features: SHOP-01
- Document: `huawei_industry_tree/16_shopping/docs/01_practice-purchase-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-purchase-app-architecture-v1-0000002077367333
- Why: A caret or tilde range lets ohpm resolve a different version than the one the sample was verified against, so the published sample is not reproducible and can break without any change to its own source. Sample projects are reference implementations; their dependency set should be exactly the one that was tested.
- Fix: Pin exact versions in oh-package.json5 dependencies, and record the resolved set in oh-package-lock.json5.

### `HW-16-0030` - The refresh state is compared against the literal 2 instead of the RefreshStatus enum member it is typed as

- Category B, severity low, confidence confirmed
- Features: SHOP-06
- Document: `huawei_industry_tree/16_shopping/docs/06_pull_to_jump.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pull_to_jump-0000002282295284
- Why: The callback already receives a typed enum value, and the sample discards that by comparing against a bare ordinal. The reader cannot tell which state 2 is without opening the enum, the compiler cannot catch a wrong ordinal, and any future renumbering of RefreshStatus silently changes which state shows the release prompt. This is the only branch that decides what the pull-down affordance tells the user, so getting it silently wrong means the control invites a release at the wrong moment.
- Fix: Compare against the named RefreshStatus member instead of the literal.

