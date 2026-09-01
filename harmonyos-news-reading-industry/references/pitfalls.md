# Pitfalls

> Generated from `features/*.md`. Source industry: `11_news_reading`, 27 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (4)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-11-0029` - 2 sample projects swallow errors in catch blocks with an empty body

- Category B, severity medium, confidence confirmed
- Features: NEWS-01, NEWS-22
- Document: `huawei_industry_tree/11_news_reading/docs/22_bookshelf_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bookshelf_demo-0000002334312212
- Why: A catch block with an empty body discards the error object and lets execution continue as if the operation had succeeded. The failure becomes invisible: no log, no user feedback, and no way to diagnose it from a released build. In several of these cases the guarded call is the feature the sample exists to demonstrate.
- Fix: Log the error with hilog including the BusinessError code and message, and surface a user-visible result where the operation was user initiated. Never leave the body empty.

### `HW-11-0028` - KEEP_BACKGROUND_RUNNING and backgroundModes are declared but the continuous-task API is never called

- Category D, severity low, confidence confirmed
- Features: NEWS-09, NEWS-26
- Document: `huawei_industry_tree/11_news_reading/docs/26_text_reader.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/text_reader-0000002525367690
- Why: The declarations promise background playback that the code does not implement — the timed-shutdown feature this sample demonstrates actually depends on a setTimeout that is throttled once the app backgrounds, so the config misleads readers into thinking background operation is handled.
- Fix: Call backgroundTaskManager.startBackgroundRunning(AUDIO_PLAYBACK) while reading (and stop it on completion), or drop the permission and backgroundModes entries.

### `HW-11-0030` - 2 sample projects depend on third-party packages through unpinned version ranges

- Category B, severity low, confidence confirmed
- Features: NEWS-01, NEWS-14
- Document: `huawei_industry_tree/11_news_reading/docs/01_practice-news-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-news-app-architecture-v1-0000001938013088
- Why: A caret or tilde range lets ohpm resolve a different version than the one the sample was verified against, so the published sample is not reproducible and can break without any change to its own source. Sample projects are reference implementations; their dependency set should be exactly the one that was tested.
- Fix: Pin exact versions in oh-package.json5 dependencies, and record the resolved set in oh-package-lock.json5.

### `HW-11-0031` - Systematic: 22 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: NEWS-03, NEWS-04, NEWS-07, NEWS-08, NEWS-09, NEWS-10, NEWS-11, NEWS-12, NEWS-13, NEWS-14, NEWS-15, NEWS-16, NEWS-17, NEWS-18, NEWS-19, NEWS-20, NEWS-21, NEWS-22, NEWS-23, NEWS-24, NEWS-25, NEWS-26
- Document: `huawei_industry_tree/11_news_reading/docs/10_ad_loading.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/ad_loading-0000002257601520
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (45)

### `HW-11-0023` - Picker-returned book URI is opened with READ_WRITE although the permission is documented as read-only — book import fails and creates empty books

- Category B, severity high, confidence confirmed
- Features: NEWS-22
- Document: `huawei_industry_tree/11_news_reading/docs/22_bookshelf_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bookshelf_demo-0000002334312212
- Why: Opening a read-only-permitted URI for write access is denied, so the sample's core feature (importing a local book) breaks, and the silent catch converts the error into corrupted state (empty book file).
- Fix: Open the picker URI with READ_ONLY, log/toast in catch, skip creating the destination when the read failed, and guard the finally close with `if (file)`.

### `HW-11-0045` - Article text is concatenated into HTML without escaping and loaded into a Web component with JavaScript and file access enabled, so authored markup executes as script

- Category D, severity high, confidence confirmed
- Features: NEWS-24
- Document: `huawei_industry_tree/11_news_reading/docs/24_auto_flip_read.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/auto_flip_read-0000002384511241
- Why: item.value is whatever the author typed into the RichEditor. Angle brackets pass through untouched, so a span containing a script tag or an event-handler attribute becomes live markup in the rendered article. The Web component is configured with javaScriptAccess(true) and fileAccess(true), so that markup runs with the ability to read local files through file:// URLs, which the same generator already emits for images. The sample is presented as the template for an article publishing flow, where the author and the reader are different people, so the authored content is untrusted input by construction.
- Fix: Escape &, <, >, " and ' in item.value before concatenating, and turn off javaScriptAccess unless the rendered article genuinely needs it. Prefer building the DOM through a template that treats text as text rather than assembling a document from strings.

### `HW-11-0001` - User passwords are persisted in cleartext in the local relational database

- Category D, severity medium, confidence confirmed
- Features: NEWS-01
- Document: `huawei_industry_tree/11_news_reading/docs/01_practice-news-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-news-app-architecture-v1-0000001938013088
- Why: Even though the doc discloses the login is UI-only, the template persists a real user-typed password unencrypted on disk — the pattern developers copy into production. Credentials must never be stored in cleartext; HarmonyOS provides asset store / HUKS for secrets.
- Fix: Drop the password column from the RDB schema (keep account/phone), or store a salted hash; reference Asset Store Kit for credential storage in the doc.

### `HW-11-0002` - TTS pause button dereferences an engine that may not exist yet, and leaving the page never stops or releases the engine

- Category B, severity medium, confidence confirmed
- Features: NEWS-01
- Document: `huawei_industry_tree/11_news_reading/docs/01_practice-news-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-news-app-architecture-v1-0000001938013088
- Why: Tapping pause before the async createEngine completes (or after a create failure — flag is set true before speak() even starts) calls stop() on undefined and crashes; navigating away while playing leaves the engine speaking with no way to release it, since the shutdown path only exists behind the pause button.
- Fix: Set this.flag=true only in createEngine's success callback, guard pause with `if (ttsEngine) {...}`, and stop/shutdown in aboutToDisappear.

### `HW-11-0004` - Age filter uses strict '<' so content rated exactly at the user's age limit is hidden, contradicting the doc's own '>=' rule

- Category B, severity medium, confidence confirmed
- Features: NEWS-03
- Document: `huawei_industry_tree/11_news_reading/docs/03_new_minors_protection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/new_minors_protection-0000002266453325
- Why: Age-gating boundaries matter: a 16+ user loses all 16-rated content; prose and code in the same document teach opposite rules.
- Fix: Change both comparisons to <= in HomeView.ets and in the doc snippet.

### `HW-11-0005` - Minors-protection management password is stored in cleartext preferences

- Category D, severity medium, confidence confirmed
- Features: NEWS-03
- Document: `huawei_industry_tree/11_news_reading/docs/03_new_minors_protection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/new_minors_protection-0000002266453325
- Why: This password is the only barrier protecting the minors-mode settings; storing it unencrypted on disk (and comparing plaintext) is the exact anti-pattern security review flags — a salted hash or the system asset store should hold it.
- Fix: Hash the password before putValue and compare hashes on verification.

### `HW-11-0007` - File is closed twice on the success path — the finally block re-closes an already closed fd and throws after the save succeeded

- Category B, severity medium, confidence confirmed
- Features: NEWS-06
- Document: `huawei_industry_tree/11_news_reading/docs/06_base64_image_save.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/base64_image_save-0000002271203733
- Why: On every successful save the finally block calls closeSync on a descriptor that was just closed, which throws (bad fd) from within finally; the caller invokes saveImageToAlbum without await or catch, so each successful save produces an unhandled rejection right after the success toast.
- Fix: Remove the close inside try and keep only the finally close (or set file = undefined after the inner close).

### `HW-11-0008` - Doc's headline claim is 'no album permission needed', yet the sample declares the restricted WRITE_IMAGEVIDEO permission

- Category D, severity medium, confidence confirmed
- Features: NEWS-06
- Document: `huawei_industry_tree/11_news_reading/docs/06_base64_image_save.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/base64_image_save-0000002271203733
- Why: The declaration directly contradicts the doc's core selling point: SaveButton exists precisely to avoid this restricted (ACL) permission, and shipping the declaration anyway both misleads readers about what SaveButton achieves and adds a permission app review will reject for ordinary apps.
- Fix: Delete the requestPermissions entry from module.json5 (the SaveButton-scoped createAsset flow needs none).

### `HW-11-0013` - Global volume-key listener is not unregistered when the reader closes — keys keep flipping a released reader

- Category B, severity medium, confidence confirmed
- Features: NEWS-12
- Document: `huawei_industry_tree/11_news_reading/docs/12_volume_key_turn_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/volume_key_turn_page-0000002293620017
- Why: inputConsumer registers an app-global shortcut: leaving the reader with the toggle on keeps intercepting volume keys everywhere in the app, and each press calls flipPage on a controller whose book was just released — volume buttons appear broken outside the reading page.
- Fix: Add inputConsumer.off('keyPressed') in aboutToDisappear (and re-register on re-entry if the setting is on).

### `HW-11-0018` - Font size is putSync'd but never flushed, so the promised persistence does not survive an app restart

- Category B, severity medium, confidence confirmed
- Features: NEWS-16
- Document: `huawei_industry_tree/11_news_reading/docs/16_h5_fontsize.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5_fontsize-0000002289377882
- Why: putSync only updates the in-memory instance; when the process is killed the setting silently reverts, defeating the exact feature the doc claims to demonstrate.
- Fix: Call this.dataPreferences?.flush() after each putSync (or once in aboutToDisappear).

### `HW-11-0027` - A page-turning reader sample declares LOCATION, APPROXIMATELY_LOCATION and LOCATION_IN_BACKGROUND with zero location code

- Category D, severity medium, confidence confirmed
- Features: NEWS-25
- Document: `huawei_industry_tree/11_news_reading/docs/25_automatic_page_turn.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/automatic_page_turn-0000002415970301
- Why: Three user-grant privacy permissions — including background location, the most privacy-sensitive of all — ride along in an unrelated sample, evidently copy-pasted module.json5 config; any developer scaffolding from this template inherits an app-review rejection and a privacy red flag.
- Fix: Remove the three requestPermissions entries.

### `HW-11-0032` - 1 sample project declares permissions that no code path in the project uses

- Category D, severity medium, confidence confirmed
- Features: NEWS-01
- Document: `huawei_industry_tree/11_news_reading/docs/01_practice-news-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-news-app-architecture-v1-0000001938013088
- Why: A permission declared in module.json5 but never referenced by any .ets or .ts file in the project cannot be exercised, so it is pure over-declaration. Over-declared permissions are a release-review rejection reason, and restricted permissions especially so. Because these module.json5 files are copied as templates, the surplus entries propagate into products that have even less claim to them.
- Fix: Delete every declared permission the code does not exercise. Where the capability is reached through a permission-free path such as SaveButton or PhotoViewPicker, no declaration is needed at all.

### `HW-11-0033` - The document never mentions the configuration profile that blocks system font scaling, although the scene introduction promises that half of the feature

- Category E, severity medium, confidence confirmed
- Features: NEWS-05
- Document: `huawei_industry_tree/11_news_reading/docs/05_set_app_font_size.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_app_font_size-0000002235941282
- Why: fontSizeScale is the only mechanism in the sample that delivers the advertised system-font blocking. A developer who follows the document reproduces the slider and the preference store but not the profile, so their app still follows the system font size and the headline half of the feature is silently missing. The official app.json5 guide documents fontSizeScale under configuration with nonFollowSystem meaning the font size does not follow the system, confirming this is the intended mechanism rather than an incidental file.
- Fix: Add a third step to 实现思路 showing the configuration profile and the fontSizeScale: nonFollowSystem value, and add the AppScope/resources/base/profile/configuration.json entry to the project tree.

### `HW-11-0034` - Avoid-area heights are measured once in onWindowStageCreate and cached in AppStorage with no avoidAreaChange subscription, so the layout padding goes stale

- Category B, severity medium, confidence confirmed
- Features: NEWS-05
- Document: `huawei_industry_tree/11_news_reading/docs/05_set_app_font_size.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_app_font_size-0000002235941282
- Why: getWindowAvoidArea returns the avoid area at the moment it is called. The values are read once, during window-stage creation, and then drive the top and bottom padding of every page for the life of the app. Rotating the device, unfolding a foldable, or any change to the navigation indicator leaves the padding computed for the previous geometry, so content is clipped by the cutout or overlaps the indicator. The app is an immersive full-screen reader, which is exactly the case where the padding is doing real work.
- Fix: Subscribe with windowClass.on('avoidAreaChange', callback) after setWindowLayoutFullScreen resolves, update the two AppStorage keys from the callback, and call windowClass.off('avoidAreaChange') in onWindowStageDestroy.

### `HW-11-0037` - The avoidAreaChange listener registered in onWindowStageCreate is never released in onWindowStageDestroy

- Category B, severity medium, confidence confirmed
- Features: NEWS-08
- Document: `huawei_industry_tree/11_news_reading/docs/08_move_to_top.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/move_to_top-0000002283118621
- Why: The listener is registered against the main window every time a window stage is created and is never removed. The callback closure keeps a reference alive after the window stage is destroyed, and on a relaunch within the same process a second listener is added on top of the first, so each avoid-area change runs the AppStorage writes more than once.
- Fix: Store the callback in a class field, register it in onWindowStageCreate, and call windowClass.off('avoidAreaChange', callback) in onWindowStageDestroy.

### `HW-11-0038` - currentYOffset is decorated @State although no build method reads it, so every scroll frame performs a state write for a value used only in a comparison

- Category C, severity medium, confidence confirmed
- Features: NEWS-08
- Document: `huawei_industry_tree/11_news_reading/docs/08_move_to_top.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/move_to_top-0000002283118621
- Why: onDidScroll fires once per frame while the list moves. Every one of those frames writes an observed state variable, so the framework runs its dependency bookkeeping and re-render check for a value the UI never displays. The visible output is the boolean iconIsShow, which changes twice per scroll session at most. On the UI thread during scrolling this is exactly the per-frame overhead HQ's own performance practice tells developers to remove.
- Fix: Drop the @State decorator from currentYOffset and keep it as a private field, or read the offset from this.scrollerController.currentOffset().yOffset inside the callback instead of accumulating it.

### `HW-11-0039` - The back-to-top pattern is demonstrated on an eagerly rendered ForEach list, the construction HQ's own performance practice rules out for long lists

- Category C, severity medium, confidence confirmed
- Features: NEWS-08
- Document: `huawei_industry_tree/11_news_reading/docs/08_move_to_top.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/move_to_top-0000002283118621
- Why: A back-to-top affordance only exists because the list is long enough that the user cannot scroll back by hand, and the document explicitly offers the pattern for news, video and commodity feeds. HQ's own performance practice document (19_common_technical_solutions/docs/04_practice-common-app-performance-v1.md) states that long lists must use LazyForEach plus component reuse plus list-item caching together. Reproducing this sample at the scale it advertises constructs every item up front, which is the cost those three mechanisms exist to avoid.
- Fix: Convert newsInfoList to an IDataSource and use LazyForEach, mark the item component @Reusable, and set cachedCount on the List. The 12-item demo data set can stay as it is; the construction is what gets copied.

### `HW-11-0040` - The immersive avoid-area plumbing is dead: EntryAbility maintains topRectHeight and bottomRectHeight in AppStorage but no component ever reads them, and the page hardcodes a 50 vp top margin instead

- Category C, severity medium, confidence confirmed
- Features: NEWS-11
- Document: `huawei_industry_tree/11_news_reading/docs/11_hot_search.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/hot_search-0000002258742170
- Why: The window is put into full-screen layout, which means the app is responsible for its own insets. The sample builds the correct machinery for that and then does not connect it, substituting a magic 50. On any device whose status bar is not 50 vp, and after any change the avoidAreaChange listener reports, the search bar sits at the wrong offset. The dead AppStorage keys also make the sample look correct to a reader skimming the ability, which is the file the document points at for immersive behaviour.
- Fix: Declare @StorageProp('topRectHeight') and @StorageProp('bottomRectHeight') in Index, convert with px2vp, and replace the literal 50 margin with the converted top inset.

### `HW-11-0041` - The avoidAreaChange listener registered in onWindowStageCreate is never released in onWindowStageDestroy

- Category B, severity medium, confidence confirmed
- Features: NEWS-11
- Document: `huawei_industry_tree/11_news_reading/docs/11_hot_search.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/hot_search-0000002258742170
- Why: The listener outlives the window stage it was registered for. Its closure holds the window reference, and a relaunch within the same process stacks a second listener on the first, so every avoid-area change runs the AppStorage writes more than once. The comment in onWindowStageDestroy already says UI related resources are released there; the listener is the one resource the ability actually holds.
- Fix: Keep the callback in a class field and call windowClass.off('avoidAreaChange', callback) in onWindowStageDestroy.

### `HW-11-0043` - getPreferences evicts the Preferences instance from the cache on every call, the pattern the official reference warns causes data inconsistency

- Category B, severity medium, confidence confirmed
- Features: NEWS-20
- Document: `huawei_industry_tree/11_news_reading/docs/20_read_card.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/read_card-0000002351196153
- Why: The official reference states that after removePreferencesFromCacheSync 'calling getPreferences again will read data from the persistent file and create a Preferences instance', and warns: 'Avoid using a removed Preferences instance to perform data operations, which may cause data inconsistency. Instead, set the removed Preferences instance to null.' This helper does exactly what the warning forbids. EntryFormAbility.onAddForm holds an instance across its body while ReadPage.saveHistory can call getPreferences concurrently and evict it, leaving the form ability writing through a removed instance. It also forces a file read on every access, on a path that runs once per chapter change and once per card update.
- Fix: Drop the removePreferencesFromCacheSync call from getPreferences and let the cache serve the instance. Keep a separate explicit method for the rare case where the store must be reloaded, and null out any instance after removing it.

### `HW-11-0044` - Three string-typed preference keys are read with the number 0 as their default, so a card added before the reading page has ever run binds numbers into Text components

- Category B, severity medium, confidence confirmed
- Features: NEWS-20
- Document: `huawei_industry_tree/11_news_reading/docs/20_read_card.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/read_card-0000002351196153
- Why: getSync returns the supplied default when the key is absent, so on a device where the card is added from the app-icon long-press menu before ReadPage has ever run, all three string keys return the number 0 and the as string cast hides it from the compiler. ReadCard then passes a number to Text(), which is outside the declared string | Resource type. ReadPage.aboutToAppear seeds the store, but onAddForm can run first, which is the ordinary way a user adds a widget.
- Fix: Pass '' as the default for the three string keys, and in onAddForm fall back to the same seed values ReadPage.saveHistory(0) writes when the store is empty.

### `HW-11-0046` - fontColor is dereferenced with toString() three lines before the guard that checks whether it is undefined

- Category B, severity medium, confidence confirmed
- Features: NEWS-24
- Document: `huawei_industry_tree/11_news_reading/docs/24_auto_flip_read.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/auto_flip_read-0000002384511241
- Why: The guard on line 42 states that fontColor can be undefined. Line 38 has already called toString() on it unconditionally, so in exactly the case the guard exists for the expression throws a TypeError and the whole span conversion fails, taking the article publish with it. The later check can never do its job because control never reaches it.
- Fix: Move the argb2Rgba call inside the conditional, or compute it as item.textStyle.fontColor !== undefined ? argb2Rgba(item.textStyle.fontColor.toString()) : ''.

### `HW-11-0047` - The page is published under an auto_flip_read URL slug although its entire subject is rich-text article editing, and a separate document covers auto page turning

- Category E, severity medium, confidence confirmed
- Features: NEWS-24
- Document: `huawei_industry_tree/11_news_reading/docs/24_auto_flip_read.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/auto_flip_read-0000002384511241
- Why: The slug is the stable identifier for the page: it is what search results, bookmarks and cross-document links resolve to. A reader who follows a link promising auto page turning lands on a rich-text editor, and the industry's real auto-page-turn document is harder to find because the obvious slug is taken by an unrelated page. The mismatch is between the URL and the page, not inside the page, so no amount of reading the content reveals it.
- Fix: Republish the page under a slug that names its subject, and keep a redirect from the current one.

### `HW-11-0003` - Project tree omits the features/service HAR that exists in the zip

- Category E, severity low, confidence confirmed
- Features: NEWS-01
- Document: `huawei_industry_tree/11_news_reading/docs/01_practice-news-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-news-app-architecture-v1-0000001938013088
- Why: The documented module structure is the map readers navigate the sample by; a whole missing module suggests the tree was not regenerated after the sample changed.
- Fix: Regenerate the 代码结构解读 tree from the current zip.

### `HW-11-0006` - Doc snippet opens the sheet by setting a variable (showEditSheet) that does not exist in the sample

- Category E, severity low, confidence confirmed
- Features: NEWS-04
- Document: `huawei_industry_tree/11_news_reading/docs/04_channel_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/channel_selection-0000002270325497
- Why: Copied as printed, the snippet's click handler writes an undeclared variable and the sheet never opens; the doc invents a third name absent from the sample.
- Fix: Correct the onClick line to the sample's canShowEditSheet @Link flow.

### `HW-11-0009` - Doc snippet references CONFIGURATION.PAGEFLIPPAGECOUNT, a constant name that does not exist in the sample

- Category E, severity low, confidence confirmed
- Features: NEWS-07
- Document: `huawei_industry_tree/11_news_reading/docs/07_page_flip_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/page_flip_page-0000002271210553
- Why: Copying the documented snippet against the sample's constants file fails to compile; doc and sample use different identifiers for the same constant.
- Fix: Rename the constant in the snippet to CONFIGURATION.PAGE_FLIP_PAGE_COUNT.

### `HW-11-0010` - requestMore listener is a no-op (returns a function reference without calling it) and TextReader is never released

- Category B, severity low, confidence confirmed
- Features: NEWS-09
- Document: `huawei_industry_tree/11_news_reading/docs/09_ai_recitation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/ai_recitation-0000002290573329
- Why: When the reader panel requests more content the app silently does nothing (dead listener), and the initialized TextReader service plus its three registered listeners are never released on exit — the pattern developers copy leaves the speech service running.
- Fix: Replace the callback with a real handler and add cleanup in aboutToDisappear / onDestroy.

### `HW-11-0011` - Ad node controllers are cached in a module-level map that is never cleaned up

- Category B, severity low, confidence probable
- Features: NEWS-10
- Document: `huawei_industry_tree/11_news_reading/docs/10_ad_loading.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/ad_loading-0000002257601520
- Why: The module-level map outlives the pages: every recreation of the list builds new controllers whose FrameNode/BuilderNode trees are retained by the old map entries, so ad view hierarchies accumulate in memory for the life of the process — a leak pattern in the exact API (NodeController) whose docs emphasize dispose management.
- Fix: Add a cleanup (NODE_MAP.get(id)?.dispose(); NODE_MAP.delete(id)) invoked from the ad card's aboutToDisappear / page teardown.

### `HW-11-0012` - Project tree lists IconModels.ets; the zip file is IconModel.ets

- Category E, severity low, confidence confirmed
- Features: NEWS-10
- Document: `huawei_industry_tree/11_news_reading/docs/10_ad_loading.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/ad_loading-0000002257601520
- Why: Tree/zip filename mismatch; the referenced file does not exist under the documented name.
- Fix: Rename the tree entry to IconModel.ets.

### `HW-11-0014` - Mark merge algorithm changes text outside the user's selection: partial overlaps are recolored wholesale and un-marking erases the entire overlapping highlight

- Category B, severity low, confidence probable
- Features: NEWS-13
- Document: `huawei_industry_tree/11_news_reading/docs/13_text_marker_ability.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/text_marker_ability-0000002283796046
- Why: Two user-visible wrong results: (1) highlighting green over half of an existing yellow mark repaints the untouched yellow half green; (2) clearing a few words inside a long highlight sets the whole union range transparent, deleting the entire highlight instead of splitting it into left/right remnants. Both alter text the user never selected.
- Fix: When an overlap is found, push the non-overlapping sub-ranges of the existing mark (with its original color) into leftMarks/rightMarks instead of extending currentSelection.

### `HW-11-0015` - Project tree lists pages/Index.ets although the zip's page is PresetBookPage.ets

- Category E, severity low, confidence confirmed
- Features: NEWS-14
- Document: `huawei_industry_tree/11_news_reading/docs/14_preset_book.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/preset_book-0000002284947236
- Why: The tree and the snippet within the same doc name different files; the tree's file does not exist in the sample.
- Fix: Replace Index.ets with PresetBookPage.ets in 工程目录.

### `HW-11-0016` - Typewriter interval is only cleared when the text finishes — leaving the page mid-animation leaks the 50 ms timer

- Category B, severity low, confidence confirmed
- Features: NEWS-15
- Document: `huawei_industry_tree/11_news_reading/docs/15_typewriter_effect.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/typewriter_effect-0000002322974369
- Why: A 20-per-second timer keeps firing after the component is gone, mutating @Observed items of a detached tree until the whole text length is exhausted; the doc's snippet teaches the same unguarded pattern.
- Fix: Add aboutToDisappear(): clearInterval(this.intervalId).

### `HW-11-0017` - Constraints section requires API 20 / HarmonyOS 6.0.0 while the sample project targets compatibleSdkVersion 5.0.5(17)

- Category E, severity low, confidence confirmed
- Features: NEWS-15
- Document: `huawei_industry_tree/11_news_reading/docs/15_typewriter_effect.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/typewriter_effect-0000002322974369
- Why: The doc and its own sample disagree by three API versions: either the constraint overstates the requirement (the code needs nothing above API 17) or the sample config is outdated — readers on 5.0.5 are told they cannot run a sample that is configured for exactly their version.
- Fix: Align the 约束与限制 section with the sample's 5.0.5(17), or bump the sample's compatibleSdkVersion to 6.0.0(20).

### `HW-11-0019` - Project tree lists EntryBackAbility.ets; the zip file is EntryBackupAbility.ets

- Category E, severity low, confidence confirmed
- Features: NEWS-17
- Document: `huawei_industry_tree/11_news_reading/docs/17_regular_highlight.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/regular_highlight-0000002328562941
- Why: Tree/zip filename mismatch ('Backup' truncated to 'Back').
- Fix: Correct the entry to EntryBackupAbility.ets.

### `HW-11-0020` - Project tree lists 'view' directory while the zip uses 'views'; magnifier math is tied to hardcoded device-specific offsets

- Category E, severity low, confidence confirmed
- Features: NEWS-18
- Document: `huawei_industry_tree/11_news_reading/docs/18_magnifier.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/magnifier-0000002296851262
- Why: The tree entry doesn't match the sample, and the unexplained magic offsets make the core effect break silently on other window sizes without the doc warning about it.
- Fix: Rename the tree entry to views and derive the lens offsets from the magnifier dimensions instead of literals.

### `HW-11-0021` - Pasteboard references point to the native C API although the sample uses the ArkTS pasteboard module

- Category E, severity low, confidence confirmed
- Features: NEWS-19
- Document: `huawei_industry_tree/11_news_reading/docs/19_erase_recognize.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/erase_recognize-0000002313854866
- Why: Readers following the reference land on C/NDK documentation that does not match a line of the ArkTS sample.
- Fix: Replace both capi-pasteboard links with the ArkTS pasteboard reference.

### `HW-11-0022` - Histogram scale divides by zero when all readings are 0, producing NaN bar coordinates

- Category B, severity low, confidence confirmed
- Features: NEWS-21
- Document: `huawei_industry_tree/11_news_reading/docs/21_time_statistics.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/time_statistics-0000002362798665
- Why: A brand-new user or an empty week is the normal first-run state; the chart draws with NaN coordinates instead of an empty baseline.
- Fix: let yScale = MAX_DATA_VALUE > 0 ? MAX_ALLOWED_HEIGHT / MAX_DATA_VALUE : 0;

### `HW-11-0024` - dealHistoryData splices the array inside its own forEach — the identical defective function is copy-pasted from the shopping SearchHistory sample

- Category B, severity low, confidence confirmed
- Features: NEWS-22
- Document: `huawei_industry_tree/11_news_reading/docs/22_bookshelf_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bookshelf_demo-0000002334312212
- Why: Same mutation-during-iteration defect, propagated across industries by copy-paste — evidence that a fix must be applied at the shared snippet source, not just one sample.
- Fix: this.historyWords = this.historyWords.filter(item => item.word !== wordModel.word); then unshift (fix both samples).

### `HW-11-0025` - keyboardHeightChange window listener is registered but never unregistered

- Category B, severity low, confidence confirmed
- Features: NEWS-23
- Document: `huawei_industry_tree/11_news_reading/docs/23_novel_read_review.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/novel_read_review-0000002347649374
- Why: Every opened comment sheet adds another window-level keyboard listener that survives the component; listeners accumulate per sheet open and keep firing into destroyed state.
- Fix: Call win.off('keyboardHeightChange') in aboutToDisappear (keep the callback reference).

### `HW-11-0026` - Auto-page-turn interval is not stopped in aboutToDisappear — it keeps flipping a released reader

- Category B, severity low, confidence confirmed
- Features: NEWS-25
- Document: `huawei_industry_tree/11_news_reading/docs/25_automatic_page_turn.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/automatic_page_turn-0000002415970301
- Why: Exiting the reader while auto-turn is active leaks a periodic timer that operates on a released controller for the rest of the process lifetime; the doc's snippet shows start/stop but never ties stop to the lifecycle.
- Fix: Add this.stopAutoScroll(); as the first line of aboutToDisappear.

### `HW-11-0035` - setWindowLayoutFullScreen is called without await or a rejection handler, and the avoid area is read before it can have taken effect

- Category B, severity low, confidence confirmed
- Features: NEWS-05
- Document: `huawei_industry_tree/11_news_reading/docs/05_set_app_font_size.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_app_font_size-0000002235941282
- Why: setWindowLayoutFullScreen returns a Promise. Discarding it means a failure to enter full-screen layout is silent, and the two getWindowAvoidArea calls on the next lines run before the layout change is guaranteed to have been applied, so they can return the pre-full-screen avoid area. An unhandled rejection also surfaces as an unhandled promise rejection at runtime.
- Fix: Make onWindowStageCreate async and await setWindowLayoutFullScreen before reading the avoid areas, with a catch that logs the BusinessError.

### `HW-11-0036` - The published Slider onChange snippet guards on a font size of 0 that the sample can never produce, so the branch is dead code

- Category B, severity low, confidence confirmed
- Features: NEWS-05
- Document: `huawei_industry_tree/11_news_reading/docs/05_set_app_font_size.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/set_app_font_size-0000002235941282
- Why: The preference key is always populated with 16 before any page is shown, and the provider initialises to 16, so changeFontSize is never 0 and the early return can never execute. Carrying the branch into the published snippet suggests to a reader that the first slider movement needs special handling, and copying it into a project where the default is genuinely 0 would silently discard the user's first drag.
- Fix: Delete the zero branch from both the sample and the snippet in the document, and initialise changeFontSize from preferences in aboutToAppear instead.

### `HW-11-0042` - The project tree names the resource directory entry/src/main/resource, but the shipped directory is resources

- Category E, severity low, confidence confirmed
- Features: NEWS-11
- Document: `huawei_industry_tree/11_news_reading/docs/11_hot_search.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/hot_search-0000002258742170
- Why: Resource directory names are fixed by the build system: $r references resolve against resources, and a directory named resource is not compiled at all. A reader recreating the project from the tree gets a build where no resource resolves.
- Fix: Change resource to resources in the project tree.

### `HW-11-0048` - getSandbox closes the source file with the asynchronous fs.close inside a finally, so the descriptor is not closed before the function returns

- Category B, severity low, confidence confirmed
- Features: NEWS-24
- Document: `huawei_industry_tree/11_news_reading/docs/24_auto_flip_read.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/auto_flip_read-0000002384511241
- Why: fs.close returns a Promise. The function is synchronous and the return value is discarded, so the close is scheduled rather than performed and any failure is an unhandled rejection. getSandbox is called once per image span while generating the article HTML, so a multi-image article opens several descriptors whose closure is deferred, and file descriptors are a bounded per-process resource.
- Fix: Replace fs.close(file) with fs.closeSync(file) so the descriptor is released before the function returns.

### `HW-11-0049` - The generated img tag separates its attributes with a comma, which is not valid HTML attribute syntax

- Category B, severity low, confidence confirmed
- Features: NEWS-24
- Document: `huawei_industry_tree/11_news_reading/docs/24_auto_flip_read.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/auto_flip_read-0000002384511241
- Why: HTML attributes are whitespace separated. The comma is parsed as part of the attribute list and produces a bogus attribute named , so the markup the sample emits is malformed. Browsers recover from it, which is why the demo looks correct, but the generated document does not validate and any downstream consumer that parses it strictly rejects it. The article HTML is persisted and re-rendered, so the defect ships with every published article.
- Fix: Remove the comma between the width and height attributes.


## Cross-industry references (1)

Referenced by a feature card here but recorded in another industry's workbook, usually a corpus-wide defect. Read the detail in the owning industry's pitfalls: `HW-16-0013`
