# Pitfalls

> Generated from `features/*.md`. Source industry: `19_common_technical_solutions`, 52 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (3)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-19-0183` - Systematic: 7 sample projects declare permissions that no code path in the project uses

- Category D, severity medium, confidence confirmed
- Features: COMMON-18, COMMON-21, COMMON-28, COMMON-32, COMMON-36, COMMON-39, COMMON-48
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/21_agent_download_control.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/agent_download_control-0000002262990970
- Why: A permission declared in module.json5 but never referenced by any .ets or .ts file in the project cannot be exercised, so it is pure over-declaration. Over-declared permissions are a release-review rejection reason, and restricted permissions especially so. Because these module.json5 files are copied as templates, the surplus entries propagate into products that have even less claim to them.
- Fix: Delete every declared permission the code does not exercise. Where the capability is reached through a permission-free path such as SaveButton or PhotoViewPicker, no declaration is needed at all.

### `HW-19-0181` - 3 sample projects depend on third-party packages through unpinned version ranges

- Category B, severity low, confidence confirmed
- Features: COMMON-07, COMMON-08, COMMON-17
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/07_cache_clean.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cache_clean-0000002231963970
- Why: A caret or tilde range lets ohpm resolve a different version than the one the sample was verified against, so the published sample is not reproducible and can break without any change to its own source. Sample projects are reference implementations; their dependency set should be exactly the one that was tested.
- Fix: Pin exact versions in oh-package.json5 dependencies, and record the resolved set in oh-package-lock.json5.

### `HW-19-0182` - Systematic: 41 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: COMMON-07, COMMON-08, COMMON-09, COMMON-10, COMMON-11, COMMON-14, COMMON-15, COMMON-16, COMMON-17, COMMON-18, COMMON-19, COMMON-20, COMMON-21, COMMON-22, COMMON-23, COMMON-24, COMMON-25, COMMON-26, COMMON-27, COMMON-28, COMMON-29, COMMON-31, COMMON-32, COMMON-33, COMMON-34, COMMON-35, COMMON-36, COMMON-37, COMMON-38, COMMON-39, COMMON-40, COMMON-41, COMMON-42, COMMON-43, COMMON-45, COMMON-46, COMMON-47, COMMON-48, COMMON-49, COMMON-50, COMMON-51
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/25_accessibility.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/accessibility-0000002320999085
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (180)

### `HW-19-0039` - The AES key used to protect stored user passwords is hardcoded in the login page source, so the reversibly encrypted passwords in the database are effectively in the clear

- Category D, severity blocker, confidence confirmed
- Features: COMMON-17
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/17_silent_login.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/silent_login-0000002292499361
- Why: The key ships inside the application package, identical on every installation, so anyone who reads the APP file recovers it and can decrypt every stored password. Encryption with an embedded key is not a secret; it is an encoding. Passwords used only for local verification must not be recoverable at all - they should be stored as a salted one-way hash, and any key that must exist should be generated per device and held by the system key store rather than written into a source file.
- Fix: Do not store the password reversibly. Store a per-account random salt and a hash (for example PBKDF2/SM3 via Universal Keystore Kit) and compare hashes at login. If a symmetric key is genuinely required for other data, generate it per installation and keep it in the key store instead of embedding a literal.

### `HW-19-0153` - ConverterInit closes the UTF-8 converter on the GBK fallback path, so a successful fallback returns a closed converter and a failed one closes it twice

- Category B, severity blocker, confidence confirmed
- Features: COMMON-47
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/47_format_conversion.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/format_conversion-0000002516884120
- Why: Two distinct memory-safety failures on one path. Where the fallback succeeds, ConverterInit reports success and every subsequent conversion passes a freed UConverter to ICU - a use-after-free on the first call the caller makes, in both DemoConversion and Utf82Gb2312 (napi_init.cpp:32 and :99, which both proceed on a zero return). Where the fallback also fails, the same pointer is passed to ucnv_close twice, which is a double free. The comment on line 36, 如果gb2312不可用，尝试gbk ("if gb2312 is unavailable, try gbk"), shows the fallback is meant to leave the object usable - the close on line 37 is simply in the wrong function, since releasing the UTF-8 converter has nothing to do with the GB2312 one failing to open.
- Fix: Delete the ucnv_close on line 37. The close on line 41 is the correct one and covers the only path where the object is abandoned. Set converter->utf8Conv and converter->gb2312Conv to nullptr after closing so ConverterCleanup cannot close them again.

### `HW-19-0177` - A failed copy of a single file deletes the whole application files directory, including every file imported before it

- Category B, severity blocker, confidence confirmed
- Features: COMMON-51
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/51_file_persistence_permission_acquisition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/file_persistence_permission_acquisition-0000002412498917
- Why: The catch is reached by ordinary, expected failures - a uri whose persisted permission has lapsed, a source file the user has since deleted or renamed, a device out of storage, any one of which affects one file - and its response is to delete the directory that holds every file the application has ever imported, along with anything else the application keeps under filesDir. The loop then continues to the next uri and opens destination paths under a directory that no longer exists. Nothing in the catch is scoped to the file that failed: the natural cleanup would be to remove the partial destination file, path + name, which is the only artefact the failed iteration created.
- Fix: Delete only the partial destination: in the catch, close the handles first and then fs.unlinkSync(path + name) guarded by fs.accessSync, and continue the loop. Never pass a sandbox root to a recursive delete.

### `HW-19-0004` - Both cache paths are read from module-level contexts, so the four 'different' paths collapse into two duplicated ones and the application-level cache directory /data/storage/elX/base/cache is never cleared

- Category A, severity high, confidence confirmed
- Features: COMMON-07
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/07_cache_clean.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cache_clean-0000002231963970
- Why: The feature is 'clear the application cache'. Because neither context is an ApplicationContext, the application-level cache directory is never enumerated, so cache written there is never deleted - including the ArkWeb cache, which the official Web guides place at /data/storage/el2/base/cache/web/ (documentation/harmonyos-guides/03_application-framework/web-download.md:41). At the same time paths[0] equals paths[1] and paths[2] equals paths[3], so each module cache directory is scanned twice.
- Fix: Obtain the application-level cache directory from the application context and the module-level one from the module context, for both encryption areas. For example: const appCtx = this.context.getApplicationContext(); paths.push(appCtx.cacheDir); paths.push(this.context.cacheDir); then set appCtx.area and this.context.area to contextConstant.AreaMode.EL1, push both cacheDir values again, and restore both to the original area. Update the four path comments in the document accordingly.

### `HW-19-0019` - PersistentStorage.persistProp('mode') runs inside the loadContent callback, after the page's @StorageLink has already created 'mode' in AppStorage, so the theme chosen in the previous run is discarded on every launch

- Category B, severity high, confidence confirmed
- Features: COMMON-11
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/11_edition_switch.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/edition_switch-0000002284505357
- Why: The entire point of the feature is that the chosen theme survives a restart. Because loadContent's callback runs after the page content has been created, AppStorage already holds 'mode' with the hardcoded STANDARD_MODE default by the time persistProp is called, so persistProp adopts that value and overwrites what the previous run stored. The application always reopens in standard mode regardless of what the user selected.
- Fix: Call PersistentStorage.persistProp('mode', STANDARD_MODE) before loadContent - for example at the top of onWindowStageCreate, or in the ability's onCreate - so that the persisted value is placed into AppStorage before any @StorageLink or AppStorage.get touches the key.

### `HW-19-0025` - Both authentication paths pass a hardcoded six-byte challenge instead of a freshly generated random value, defeating the replay protection the challenge exists for

- Category D, severity high, confidence confirmed
- Features: COMMON-13
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/13_user_authentication.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/user_authentication-0000002290451165
- Why: The challenge is the nonce that binds the returned authentication token to this particular authentication request. A constant challenge means every successful authentication produces a token derived from the same nonce, so a token captured once can be replayed later and still verify - which is exactly the attack the reference says the field prevents. The document presents this as the identity-verification solution for banking login and payment scenarios ("登录银行账户、支付交易时需要进行身份认证", "identity authentication is required when logging in to a bank account or making a payment transaction"), where replay protection is not optional.
- Fix: Generate the challenge per request with cryptoFramework.createRandom().generateRandomSync(16).data, following the official example, and abort the authentication if generation fails. Note that the sample also never reads result.token, so if a token is later sent to a server, the random challenge must be the one the server issued.

### `HW-19-0030` - The page enables web debugging unconditionally in aboutToAppear, which the official reference advises against in released applications

- Category D, severity high, confidence confirmed
- Features: COMMON-14
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/14_upload_picture.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/upload_picture-0000002256189702
- Why: The feature is an identity-document upload form: the Web component hosts the front and back ID-card upload pages, and the selected photo URIs are handed to it. With web debugging on, anyone who can attach DevTools to the device can inspect and modify that page's internal state. The sample is presented as the reference implementation for this scenario, so the flag propagates into applications that copy it.
- Fix: Remove the call, or gate it so it can only run in a debug build, and wrap it in try/catch as the official example does. Web debugging is off by default, so nothing needs to be done for the release path.

### `HW-19-0031` - handleFileList is given the component's own display array before the new selection is written into it, so the HTML file input receives the previous values instead of the picked photo

- Category B, severity high, confidence confirmed
- Features: COMMON-14
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/14_upload_picture.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/upload_picture-0000002256189702
- Why: handleFileList is how the file the user picked is handed back to the HTML <input type="file"> that triggered the selector. Passing the component's display array means the web form is told about whatever was selected previously - and on the first use, about two app.media Resource objects rather than file URIs. It also hands both slots to a form that asked for one file. The picture that the user just chose is never given to the page.
- Fix: Pass only the new selection, after checking it: `if (photoSelectResult && photoSelectResult.photoUris.length > 0) { event?.result.handleFileList(photoSelectResult.photoUris); this.uris[this.isFront ? 0 : 1] = photoSelectResult.photoUris[0]; }` Keep the ArkUI preview array separate from the value handed to the web page.

### `HW-19-0034` - The page enables web debugging unconditionally in aboutToAppear, which the official reference advises against in released applications

- Category D, severity high, confidence confirmed
- Features: COMMON-15
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/15_webview_picker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/webview_picker-0000002257162014
- Why: This page hosts an upload form and drives three system pickers - photo, camera and document - from JavaScript that the page itself returns via runJavaScript('getFileType()'). With web debugging enabled, that page's state and the JavaScript bridge it uses to choose which picker to launch can be inspected and modified from DevTools on the device.
- Fix: Remove the call from the release path, or gate it on build mode and wrap it in try/catch as the official example does. Web debugging is disabled by default, so no code is needed for production.

### `HW-19-0040` - The RDB query column list names a column that the account table does not have, so every account query fails and the login page always falls through to the registration branch

- Category A, severity high, confidence confirmed
- Features: COMMON-17
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/17_silent_login.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/silent_login-0000002292499361
- Why: Querying a non-existent column makes the query fail, and Rdb.query's error branch logs and returns without invoking the callback. LoginPage.aboutToAppear therefore never assigns this.accountList (it stays []), so `this.accountList.findIndex(v => v.userName === this.userName)` always returns -1 and the login button always takes the else branch - insertData - even for an account that already exists. Repeated logins keep appending duplicate rows and no password is ever actually verified, which defeats the credential check the document describes in step 1 ("登录时检查数据库中用户信息是否已存在", "on login, check whether the user information already exists in the database").
- Fix: Change the columns array to ['id', 'phone', 'password'], or omit the parameter so the query applies to all columns. Also make Rdb.query report failure through the callback rather than returning silently, so the empty list is distinguishable from a failed query.

### `HW-19-0045` - The addAfter hook runs when the async download method returns its Promise, not when the download finishes, so the measured 'load time' is the promise-creation time

- Category B, severity high, confidence confirmed
- Features: COMMON-18
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/18_code_instrumentation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/code_instrumentation-0000002291963849
- Why: The document's stated purpose is 统计...图片加载时间 ("collect statistics on image loading time"). Because the hook fires as soon as instanceMethod returns its Promise - before the HTTP request has completed - end - begin measures the synchronous prologue of the async function, effectively zero, and never the network time the feature is supposed to report. The displayed 耗时 ("time taken") figure on the statistics page is therefore meaningless.
- Fix: Instrument something that completes synchronously with the work, or stop the clock in the promise: have the after hook wrap the returned promise, for example `return (ret as Promise<image.PixelMap | undefined>).then((v) => { end3 = new Date().getTime(); required3 += end3 - begin3; return v; });` and type the parameter as the promise it actually is.

### `HW-19-0046` - The document's addAfter snippet omits the return statement, so code copied from it makes the instrumented method return undefined and breaks the caller's promise chain

- Category E, severity high, confidence confirmed
- Features: COMMON-18
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/18_code_instrumentation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/code_instrumentation-0000002291963849
- Why: Because the inserted function's return value replaces the method's, dropping the return makes the instrumented instanceMethod resolve to undefined. Every call site chains off it - for example CodeInstrumentation.zip#CodeInstrumentation/entry/src/main/ets/pages/FunPage.ets:41, `new DownloadImage3().instanceMethod(this.url).then((pixelMap: image.PixelMap | undefined) => { ... })` - so a reader who copies the document snippet gets a TypeError on .then of undefined and no image at all. This is the single snippet the document offers for step 3.
- Fix: Add `return ret;` as the last statement of the addAfter callback in the document snippet, matching all four sample pages and the reference example.

### `HW-19-0048` - All four news pages pass an empty image URL to the download method, so the sample downloads nothing and the document never says a URL must be supplied

- Category E, severity high, confidence confirmed
- Features: COMMON-18
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/18_code_instrumentation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/code_instrumentation-0000002291963849
- Why: The whole feature is timing an image download. With an empty URL the request cannot succeed, downloadNetworkImage falls to its `else { return undefined; }` branch, ReUsePage renders an undefined PixelMap, and the statistics page shows counts and times for a download that never happened. A reader who builds and runs the sample as shipped sees a blank card and cannot tell whether the instrumentation works. The document must state that the URL is desensitised and has to be filled in.
- Fix: State in 实现思路 or 约束与限制 that the sample ships with the image URL removed and that @State url must be set to a reachable https image address before running; ideally have the sample read it from a constant so there is one place to change.

### `HW-19-0050` - BleUtil.isScanning is never set to true, so the guard in stopScan() always fails and ble.stopBLEScan() is never called - the BLE scan keeps running after the user leaves the page

- Category B, severity high, confidence confirmed
- Features: COMMON-19
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/19_search_and_connect_ble.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_and_connect_ble-0000002293947117
- Why: The flag is the sole condition for calling ble.stopBLEScan(), so the stop path is dead code and a scan started by the user is never stopped - it continues while the app is in the background and until the process ends. The document presents stopScan as one of its three implementation steps ("停止扫描蓝牙设备：引入蓝牙ble模块，停止蓝牙扫描。", "stop scanning for Bluetooth devices: import the ble module and stop the Bluetooth scan"), so the shipped code does not do what the document describes. A continuously running BLE scan is a direct battery cost on the exact class of long-running low-power scenarios the document targets.
- Fix: Set BleUtil.isScanning = true immediately after the successful ble.startBLEScan([scanFilter], scanOptions) call, so that the guarded stop in stopScan() and in the re-entry path of startBLEScan actually runs.

### `HW-19-0052` - The GATT client is connected and subscribed to but never disconnected, closed or unsubscribed anywhere in the project

- Category B, severity high, confidence confirmed
- Features: COMMON-19
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/19_search_and_connect_ble.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_and_connect_ble-0000002293947117
- Why: Each connectDevice call constructs a new GattClientDevice, registers a connection-state listener on it and opens a GATT link that is never released. Connecting to a second device, or reconnecting to the same one, leaks the previous client object together with its listeners, and the radio link stays up after the user leaves the page - the same battery cost as the unstopped scan, on a feature whose stated audience is 智能手表、健康监测和智能家居等长时间运行的低功耗设备 ("long-running low-power devices such as smart watches, health monitors and smart home devices"). The document's third implementation step shows the connect half of the pair and never mentions the disconnect half.
- Fix: Add a disconnect path to DeviceManager: `this.clientDevice?.off('BLEConnectionStateChange'); this.clientDevice?.disconnect(); this.clientDevice?.close(); this.clientDevice = undefined;` and call it from the page's aboutToDisappear and from the UI's disconnect action. Release the 'BLECharacteristicChange' subscription the same way.

### `HW-19-0063` - The STATUS_FINISH case in fileOperate has no break, so it falls through into STATUS_COMPLETED and deletes the file and the list entry twice

- Category B, severity high, confidence confirmed
- Features: COMMON-21
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/21_agent_download_control.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/agent_download_control-0000002262990970
- Why: STATUS_FINISH is the state a task reaches the moment its 'completed' event fires (FileItem.ets sets `this.status = Constants.STATUS_FINISH;` in the completed handler), so it is the normal state of a just-downloaded file. Tapping the delete icon in that state runs the delete path twice: cleanFileByName is called a second time on a file that no longer exists, and removeByName runs a second filter/splice over this.fileList - which, because removeByName removes the first match by index, can delete a different entry if two files share the name. Nothing about the second pass is intended; the two cases are byte-identical.
- Fix: Add `break;` after the STATUS_FINISH case, as the document's snippet shows. Since the two cases have identical bodies, the cleaner form is to let STATUS_FINISH fall through to STATUS_COMPLETED with no statements of its own: `case Constants.STATUS_FINISH: case Constants.STATUS_COMPLETED: fileUtils.cleanFileByName(this.fileName); this.removeByName(this.fileName); break;`

### `HW-19-0073` - The native scanner object is registered as a JavaScript proxy and never deleted, so it stays exposed after the web view navigates to a remote site

- Category D, severity high, confidence confirmed
- Features: COMMON-23
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/23_web_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_scan-0000002284631808
- Why: The proxy is bound to the WebviewController, not to the page, and the reference says it is exposed to all page frames. After handleScanResult navigates away from the packaged index.html, whatever the remote site serves - and any third-party frame it embeds - can call window.nativeScanner.scan(), which raises the camera permission dialog and launches the system scanner, and window.nativeScanner.handleScanResult(x), which drives the web view to a URL of the caller's choosing. Nothing in the code narrows the proxy to the local page. The missing deleteJavaScriptRegister is also the memory leak the reference names.
- Fix: Scope the proxy to the trusted local page: delete it before leaving that page, `this.webVC.deleteJavaScriptRegister('nativeScanner');` - for instance in onPageBegin when the upcoming URL is not the packaged rawfile, and in aboutToDisappear. If the remote page genuinely needs the bridge, use the permission parameter of registerJavaScriptProxy to restrict the calling origins rather than exposing it to every frame.

### `HW-19-0074` - The page enables web debugging unconditionally in aboutToAppear, on a web view that carries a native scanner bridge

- Category D, severity high, confidence confirmed
- Features: COMMON-23
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/23_web_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_scan-0000002284631808
- Why: This web view has a JavaScript proxy that reaches the camera: window.nativeScanner.scan() launches the system scanner and window.nativeScanner.handleScanResult() redirects the view. With web debugging enabled, that bridge is reachable from the DevTools console on the device, in a scenario the document scopes to 银行、购物、娱乐、支付类等应用 ("banking, shopping, entertainment and payment applications"). This is the third sample in this industry shipping the same line (see also HW-19-0030 and HW-19-0034), so it is a propagating pattern.
- Fix: Remove the call from the release path, or gate it on build mode and wrap it in try/catch as the reference example does.

### `HW-19-0081` - The document requires ohos.permission.APP_TRACKING_CONSENT, but the sample declares no permission at all and requests the ad with an empty OAID

- Category E, severity high, confidence confirmed
- Features: COMMON-26
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/26_splash_page_ad_access.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/splash_page_ad_access-0000002322822425
- Why: APP_TRACKING_CONSENT is a user_grant permission, so it needs both a declaration in module.json5 and a runtime request before the OAID can be read - and the OAID is the only reason the document names it. As shipped the sample cannot obtain an OAID, sends an empty one, and a reader who follows the 权限说明 section finds nothing in the project to copy. The two halves of the document contradict each other: the permission section describes personalised-ad tracking, the code implements the non-personalised path.
- Fix: Either add the declaration and the runtime request - declare ohos.permission.APP_TRACKING_CONSENT with reason and usedScene, request it with requestPermissionsFromUser, read the identifier through Ads Kit's identifier module and pass it as oaid - or state in the 权限说明 section that the permission is optional and that an empty oaid requests non-personalised ads.

### `HW-19-0084` - The pre-login home page is only blurred, not disabled, and closing the login prompt leaves the blurred content fully interactive

- Category D, severity high, confidence confirmed
- Features: COMMON-27
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/27_display_after_login.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/display_after_login-0000002293948070
- Why: foregroundBlurStyle is a rendering attribute: it changes how the subtree is drawn, not whether it receives touches. The document presents the blur as the gate that makes the page unusable before login - 登录后模糊效果消失，页面可正常使用 ("after login the blur disappears and the page can be used normally") - which implies it is not usable before. It is: tapping the close button on the login prompt removes the only thing covering the content, and every control underneath still responds while the page reads as locked. A user, or an automated tap, can drive the pre-login UI through a screen that looks disabled.
- Fix: Pair the blur with an interaction block on the same subtree, for example `.enabled(this.isLogin)` or `.hitTestBehavior(this.isLogin ? HitTestMode.Default : HitTestMode.Block)`, and keep the login prompt non-dismissable (or re-show it) while isLogin is false.

### `HW-19-0092` - onGestureSwipe reads an 'index' field that getTextInfo never returns, so the selected tab index is set to undefined on every swipe frame

- Category B, severity high, confidence confirmed
- Features: COMMON-30
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/30_tab_toggle_animation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/tab_toggle_animation-0000002332006785
- Why: curIndex is declared @State curIndex: number = 0 and drives the selected-tab colouring - `fontColor(index === this.curIndex ? selected : unselected)` in tabBarBuilder - and the expansion guard `if (this.curIndex >= 2)` in onWillScroll. Assigning undefined to it on every frame of a finger swipe makes the equality test false for every tab, so no tab renders as selected while the user drags, and the `>= 2` comparison is false as well. The value is only repaired when onAnimationStart or onChange fires afterwards. The correct value is already in hand: the callback's own index parameter.
- Fix: Use the callback argument: `this.curIndex = index;` - matching what onChange and onAnimationStart already do - and drop the non-existent field from both the sample and the document snippet.

### `HW-19-0100` - The font switch is applied immediately after the asynchronous FontFace load is started, so the first tap can restyle the page before the font is registered - and a failed load is permanently remembered as loaded

- Category B, severity high, confidence confirmed
- Features: COMMON-33
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/33_h5_load_custom_font_library.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5_load_custom_font_library-0000002349762829
- Why: Two defects follow. First, on the first tap the style is set to a font family that has not yet been added to document.fonts, so the text falls back to the default until something else forces a re-layout - the switch appears not to work, or works only on the second tap. Second, fontFamily.isLoaded is set to true immediately after the load is *started*, not after it succeeded; if the FontFace load rejects - a missing or corrupt .ttf, an unreadable url - the failure is invisible (the injected promise has no rejection handler) and the guard prevents any retry for the lifetime of the page.
- Fix: Chain the switch onto the load inside the injected script rather than issuing it separately, for example `font.load().then(function(){ document.fonts.add(font); changeFont(); }).catch(function(e){ /* report */ })`, and set isLoaded from the resolved callback - or use the runJavaScript promise result to confirm success before marking the family loaded.

### `HW-19-0102` - canOpenLink is called without try/catch on every non-http scheme the page emits, so any scheme not listed in querySchemes throws out of the load interceptor

- Category B, severity high, confidence confirmed
- Features: COMMON-34
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/34_app_pull_up.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_pull_up-0000002353615821
- Why: onLoadIntercept runs for every navigation the page attempts. A mailto:, tel:, about:blank or any other non-http URL - including a malformed one, which is error 17700055 - reaches canOpenLink and throws, because only hwtips is declared. The throw escapes the interceptor callback, which has no try/catch anywhere in the chain, so the navigation is neither allowed nor handled and the exception surfaces from the Web component's event dispatch. The narrower the querySchemes list is, the more URLs hit this path.
- Fix: Wrap the check as the reference example does and treat a throw as 'cannot open': `let canOpen = false; try { canOpen = bundleManager.canOpenLink(url); } catch (err) { hilog.error(0x0000, 'URL', `canOpenLink failed. Code: ${(err as BusinessError).code}, message: ${(err as BusinessError).message}`); } if (canOpen) { ... } else { ... }`. Better still, match the URL against the declared schemes before calling.

### `HW-19-0108` - Fold-status listeners are registered from inside repeatedly-invoked callbacks and are never released, so they accumulate on every resize and every page show

- Category B, severity high, confidence confirmed
- Features: COMMON-36
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/36_web_display_mode_switch.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_display_mode_switch-0000002366467641
- Why: Both hosts are called more than once. windowSizeChange fires on every fold, unfold, rotation and split-screen change - and each firing adds another foldStatusChange listener, so after n resizes a single fold event runs the handler n times. onPageShow fires every time the page returns to the foreground, adding another listener each time. On a foldable, which is the only device this feature targets, resize and fold events are exactly what the user generates. Registration belongs in a place that runs once (onWindowStageCreate directly, or aboutToAppear), and every on() needs a matching off() in the corresponding teardown.
- Fix: Move the display.on('foldStatusChange', ...) call out of the windowSizeChange handler to the same level as it, keep the callback in a field, and release both subscriptions in onWindowStageDestroy with windowClass.off('windowSizeChange') and display.off('foldStatusChange'). In the page, register in aboutToAppear and release in aboutToDisappear rather than using onPageShow.

### `HW-19-0111` - The order check is overwritten on every tap, so a wrong click order passes verification whenever the final tap happens to match

- Category B, severity high, confidence confirmed
- Features: COMMON-37
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/37_text_order_verification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/text_order_verification-0000002347027492
- Why: This is the entire correctness test of a human-verification gate that the document scopes to registration and login, and passing it sets isValidated, which unlocks sending the SMS code. A wrong order is accepted whenever the last tap lands on the right character - one in four orderings of four items ends with the correct final element, so the gate is trivially passable by guessing. The flag needs to be sticky once a mismatch is seen.
- Fix: Make the failure sticky: initialise orderIsCorrected to true when the challenge is refreshed and only ever clear it, never set it - if (curClickOrder !== textItem.clickIndex) { orderIsCorrected = false; } with no else branch. Reset it to true in refreshText alongside curClickOrder = 0.

### `HW-19-0132` - The receiver's unsubscribe lives in a method named disappear(), which is not a lifecycle callback, so the subscription is never released

- Category B, severity high, confidence confirmed
- Features: COMMON-43
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/43_customizing_common_event.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/customizing_common_event-0000002470917738
- Why: The framework calls aboutToDisappear by name; a method called disappear is an ordinary member that the framework never invokes and that no code in the project invokes either. So the receiver registers a common-event subscriber every time the page is created and releases it never. The subscriber holds a callback closing over this, which closes over the page's @Provide sessionList, so the destroyed page and its entire message list stay reachable for the process lifetime, and each re-creation of the page adds another live subscriber that will run the same handler again on the next purchaseEvent - one published event then appends the message N times. The cleanup code is fully correct; only its name is wrong, which makes the defect invisible in review.
- Fix: Rename the method to aboutToDisappear so the framework calls it: `aboutToDisappear() { if (this.subscriber) { ... } }`. Nothing inside the body needs to change.

### `HW-19-0137` - createVpnConnection runs in a field initializer before startVpnExtensionAbility and is handed a UIAbilityContext cast to VpnExtensionContext, while the VpnExtensionAbility that would supply a real one is commented out

- Category A, severity high, confidence confirmed
- Features: COMMON-44
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/44_vpn_adaptation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vpn_adaptation-0000002507518793
- Why: getHostContext() in a page returns a UIAbilityContext; the `as common.VpnExtensionContext` cast is a compile-time assertion that changes nothing at runtime, so createVpnConnection receives an object of the wrong type before the VPN function has been enabled, and every later call on the returned object - protect, create, destroy - is built on it. startVpnExtensionAbility is then pointed at EntryAbility, which is declared as a normal UIAbility, not type: vpn. As shipped, the entire VPN half of the sample cannot establish a tunnel, which matters because the document presents this project as the VPN case and only a footnote directs readers elsewhere for VPN functionality.
- Fix: Restore the VpnExtensionAbility: uncomment the extensionAbilities entry, ship ets/vpnability/VPNExtentionAbility.ets extending VpnExtensionAbility, call createVpnConnection inside its onCreate with that ability's own context, and point the Want at that ability rather than at EntryAbility. If the sample is meant to demonstrate only the long-task notification, remove VPNUtils and the VPN toggle rather than shipping a path that cannot work.

### `HW-19-0154` - Gb2312ToUtf8 lets the conversion fill the whole allocation and then writes the terminator one byte past its end

- Category B, severity high, confidence confirmed
- Features: COMMON-47
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/47_format_conversion.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/format_conversion-0000002516884120
- Why: This is a one-byte heap write past the end of a malloc block, on the path where the converted output happens to be the maximum size the buffer allows. It corrupts whatever the allocator keeps after the block - in practice heap metadata or the next allocation - and the corruption is silent, appearing later as an unrelated crash or as data damage in another buffer. The condition is reachable with ordinary input: gb2312Len * 3 is the exact worst case for GB2312 to UTF-8, since a two-byte GB2312 character becomes three UTF-8 bytes, so any input consisting entirely of Chinese characters converts to a length within one byte of the limit.
- Fix: Make the allocation one byte larger than the conversion limit, exactly as Utf8ToGb2312 does: `int32_t targetCapacity = gb2312Len * 3; char* target = (char*)malloc(targetCapacity + 1); ... char* targetLimit = target + targetCapacity;`

### `HW-19-0156` - The native module declares utf82gb2312 as returning a string while it returns a napi buffer, and the page then re-encodes the result as UTF-8 to build the GB2312 hex display

- Category A, severity high, confidence confirmed
- Features: COMMON-47
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/47_format_conversion.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/format_conversion-0000002516884120
- Why: The hexadecimal column is the only observable evidence the sample produces that a conversion happened - the document's 效果预览 shows it as the result - and it is computed by passing the value through buffer.from(str, 'utf-8'), which encodes as UTF-8. The GB2312 bytes the native layer produced are not what reaches uint8ArrayToHexStr; they are re-derived from whatever the value coerces to under a UTF-8 encoding, and GB2312 byte sequences are not valid UTF-8, so the round trip is lossy. The wrong .d.ts declaration is what allows this: it tells the ArkTS compiler the value is a string, so assigning it to `let res: string` and passing it where a string is expected both type-check, and nothing flags that a Buffer is being treated as text.
- Fix: Declare the real return type - `export const utf82gb2312: (str: string) => ArrayBuffer;` - and consume the bytes directly in the page: `let res = testNapi.utf82gb2312(this.inputMessage); let array = new Uint8Array(res); this.hexStr = this.uint8ArrayToHexStr(array);` with no UTF-8 encoding step.

### `HW-19-0159` - off('wifiScanStateChange') passes a freshly created arrow function, so it cannot remove the callback on() registered, and two return paths skip it entirely

- Category B, severity high, confidence confirmed
- Features: COMMON-48
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/48_wifi_connect.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/wifi_connect-0000002518583904
- Why: Each arrow function expression creates a new object, so the one passed to off is not the one registered by on and the framework has nothing matching to remove. getScanResult runs on every toggle-on, so listeners accumulate one per call and every scan-state change then invokes all of them. The two early returns compound it: when Wi-Fi is off - the case the function is written to handle - the listener is registered and the function returns before any attempt to remove it.
- Fix: Hold the callback in a field and pass the same reference to both calls, or call wifiManager.off('wifiScanStateChange') with no callback to remove all registrations for the event. Register once rather than on every scan, and unregister on a path that runs regardless of how the function exits.

### `HW-19-0160` - The connect path handles none of the documented outcomes, so a refused or unanswered user prompt reports 连接中 and rejects unhandled

- Category B, severity high, confidence confirmed
- Features: COMMON-48
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/48_wifi_connect.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/wifi_connect-0000002518583904
- Why: The whole point of connectToCandidateConfigWithUserAction over connectToCandidateConfig is that the user gets a prompt and can decline - the reference says so explicitly - so a rejection is the expected path, not an edge case. Neither await is guarded and neither promise carries a catch, so a decline, a timeout, a wrong pre-shared key or a disabled STA rejects into nothing. Meanwhile the two statements after the await never run on that path, leaving the password sheet open with no message, while a successful call closes the sheet and announces 连接中 ("connecting") before the connection result is known. The user is told the opposite of what happened in both directions.
- Fix: Wrap both awaits in try/catch inside connectWiFi, return the outcome to the caller, and branch the UI on it: on 2501006 or 2501005 keep the sheet open and say the connection was not confirmed; on success close the sheet and report progress.

### `HW-19-0164` - The common-event subscriber runs at module scope on import and is never unsubscribed, and the status-bar click listener has no matching off

- Category B, severity high, confidence confirmed
- Features: COMMON-49
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/49_pc_status_bar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pc_status_bar-0000002551100435
- Why: Two system-service registrations with no release path. The subscriber holds a callback that closes over globalContext and calls terminateSelf, and it is bound to the module rather than to the ability, so it outlives every EntryAbility instance and stays live for the process lifetime - in an application whose entire purpose is to keep a process alive in the background, that is the process this code runs in. statusBarIconClick is worse in kind: onStatusBarIconClick calls showAbility on globalContext, and after the ability that set globalContext has been destroyed the handler still fires on every tray click, acting on a stale context. Registering in onCreate without releasing in onDestroy also means a second creation of the ability adds a second registration.
- Fix: Move createSubscriber and subscribe into onCreate, and add onDestroy: commonEventManager.unsubscribe(subscriber) and statusBarManager.off('statusBarIconClick', onStatusBarIconClick), so each registration is released by the lifecycle hook that matches the one which created it.

### `HW-19-0169` - The migrated page path is never cleared, so onPageShow re-pushes it every time the root page becomes visible and the user cannot get back to the home page

- Category B, severity high, confidence confirmed
- Features: COMMON-50
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/50_navigation_continue.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/navigation_continue-0000002522501268
- Why: onPageShow runs whenever the page becomes visible, not only once after a migration. On the receiving device the first invocation pushes the migrated destination, which is correct; but as soon as the user presses back to return to the Navigation home page, Index becomes visible again, onPageShow fires again, pagePath still holds the same value and the same destination is pushed straight back onto the stack. The home page cannot be reached at all after a continuation, and the same happens on every return from background. The flag is consumed as though it were an event but stored as though it were state, and nothing marks it as handled.
- Fix: Clear the value immediately after acting on it: `if (this.pagePath) { const path = this.pagePath; AppStorage.setOrCreate('pagePath', ''); AppStorage.setOrCreate('pageParams', ''); this.pageStack.pushPathByName(path, params); }` - and show the same clearing step in the document snippet.

### `HW-19-0173` - Both the persist and the activate helpers swallow their failures, so the user is told authorization succeeded and the import task then runs on files the application cannot read

- Category B, severity high, confidence confirmed
- Features: COMMON-51
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/51_file_persistence_permission_acquisition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/file_persistence_permission_acquisition-0000002412498917
- Why: The two operations that grant the application access to the chosen files are the only ones whose failure matters, and both report it exclusively to the log. On the persist path the toast fires whenever the picker returned any file, so a user whose authorization failed - a device without the capability, a revoked restricted permission, a uri the system refuses - is told the files were imported and authorized. On the activate path the toast is unconditional. In both cases ImportFileTask is then launched against uris the process has no permission to open, so the visible outcome is a success message followed by files that silently fail to load. The persist path also clears the stored path list in its catch, so the next launch finds nothing and starts over, which hides the failure again.
- Fix: Have both helpers return a boolean or rethrow, and branch the UI on it: show TOAST_MSG only after persistPermission resolves, surface a failure message otherwise, and do not start ImportFileTask when authorization did not succeed.

### `HW-19-0002` - The capability overview lists 虚拟娱乐 (virtual entertainment) as a supported Huawei Pay scenario, contradicting the document's own physical-goods scope and the official non-virtual merchant category served by Payment Kit

- Category F, severity medium, confidence confirmed
- Features: COMMON-06
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/06_practice-common-huawei-pay-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-huawei-pay-v1-0000002004896593
- Why: The two statements in the same document disagree about what may be sold through the Huawei Pay checkout, and the second one contradicts the official kit split. Payment Kit merchants are activated under the explicitly non-virtual PetalPayMerchant category; virtual goods (consumables, non-consumables, subscriptions) are IAP Kit products. A developer who follows line 17 and integrates Huawei Pay for in-app virtual entertainment goods will fail merchant registration or app review.
- Fix: Remove 虚拟娱乐 from the scenario list, or qualify it - Huawei Pay covers the physical goods and real-world services of an entertainment merchant, while in-app virtual goods must go through IAP Kit. Add an explicit pointer to the IAP Kit guide for the virtual-goods case, mirroring the non-virtual category wording used in the Payment Kit configuration guide.

### `HW-19-0003` - The checkout integration section shows a client-side success branch without the official warning that the client-returned result must not be used as the user's payment result

- Category E, severity medium, confidence confirmed
- Features: COMMON-06
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/06_practice-common-huawei-pay-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-huawei-pay-v1-0000002004896593
- Why: The document is an integration practice guide and its snippet is the piece developers copy. Marking the promise resolution as 'pay success' without the accompanying caveat invites the merchant client to unlock goods on a client-side signal. The official guide requires the payment result to be taken from the server callback notification or the query API instead; the document's steps 6 and 7 describe that server callback but never state that it is the authoritative result.
- Fix: Add the official caveat next to the snippet: the client result only indicates that the checkout screen returned, and the authoritative payment state must come from the server-side result notification (payment result callback for direct merchants / for platform merchants) or the order query API, after SM2 signature verification.

### `HW-19-0005` - writeFile() swallows every stream write error in an empty catch block

- Category B, severity medium, confidence confirmed
- Features: COMMON-07
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/07_cache_clean.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cache_clean-0000002231963970
- Why: An empty catch block discards the BusinessError, so a failed write to a cache directory is indistinguishable from a successful one. Every other error path in this file logs through hilog (lines 214, 222-223, 244, 254-255), so the silent branch is the only place where a failure disappears.
- Fix: Log the error in the catch block, matching the surrounding style: catch (e) { hilog.error(0x0000, 'CacheClean', `Failed to write cache file: ${JSON.stringify(e)}`); }

### `HW-19-0006` - deleteFile() attaches no rejection handler to the fs.listFile promise, so a failure to list a cache directory is silently lost

- Category B, severity medium, confidence confirmed
- Features: COMMON-07
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/07_cache_clean.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cache_clean-0000002231963970
- Why: The inner fs.unlink call in the same method does chain a .catch, so the outer promise is the only unhandled one. If listFile rejects - for example because the directory does not exist after the encryption area was switched - the whole cache directory is skipped with no log and no user-visible failure, while the UI still reports the cache as cleared.
- Fix: Chain a rejection handler on listFile, matching the pattern the official reference example uses: fs.listFile(cacheDir).then(...).catch((err: BusinessError) => { hilog.error(0x0000, 'CacheClean', `Failed to list ${cacheDir}: ${err.code} ${err.message}`); }); Fix the same snippet in the document's step 2.

### `HW-19-0007` - build() calls SETTING_LIST.splice(0, 2), destructively mutating the exported module-level constant every time the list is rebuilt

- Category C, severity medium, confidence probable
- Features: COMMON-07
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/07_cache_clean.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cache_clean-0000002231963970
- Why: The two ListItemGroups are meant to show items 1-2 and items 3-5, but they achieve that by permanently deleting the first two entries from the shared exported constant during rendering. After the splice, SETTING_LIST holds only three items, so any later re-evaluation of the same ForEach argument removes two more entries and the 缓存管理 ("cache management") row - the row that opens the clear-cache dialog - disappears from the list. The page does drive re-rendering from state: getCache() assigns @State cache at line 225 and is called again after every clear operation (line 132).
- Fix: Use a non-destructive read and precomputed slices, for example `private readonly topItems: SettingItem[] = SETTING_LIST.slice(0, 2); private readonly restItems: SettingItem[] = SETTING_LIST.slice(2);` and render those two arrays, leaving SETTING_LIST untouched. Do not mutate module-level state from build().

### `HW-19-0009` - HomePage sets .id(COMPONENT_ID[3]) on the music-item image, but COMPONENT_ID has only three elements, so the id is undefined

- Category B, severity medium, confidence confirmed
- Features: COMMON-08
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/08_feature_guide_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/feature_guide_page-0000002271957437
- Why: The official id attribute reference states that id 'identifies a component uniquely within an application' and 'Sets a unique identifier for this component, with uniqueness guaranteed by the user'. Passing undefined sets no usable identifier, so the third guide step - the one that was meant to highlight the new-music item through 'newMusicRecommend' - can never resolve its target component. The off-by-one also leaves the declared 'newMusicRecommend' constant dead.
- Fix: Use the declared constant: .id(COMPONENT_ID[2]). Indices into COMPONENT_ID should be named constants so the mismatch is caught at review time.

### `HW-19-0010` - The highlight target ids are assigned inside ForEach loops, so the same component id is applied to every item of the banner and recommendation lists

- Category C, severity medium, confidence confirmed
- Features: COMMON-08
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/08_feature_guide_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/feature_guide_page-0000002271957437
- Why: The highlight guide resolves the region to cut out of the mask by looking the component up by id. With two or three components carrying the same id the resolution is ambiguous, so which item gets highlighted depends on lookup order rather than on the developer's intent - and the uniqueness contract the id attribute documents is broken.
- Fix: Assign the highlight id to a single component: either apply it only to the item that should be highlighted (for example `.id(index === 0 ? COMPONENT_ID[0] : '')`), or move the id onto the enclosing List, which is a single component and is what the guide should highlight.

### `HW-19-0011` - setLabel() receives a Resource object cast to string, so the guide label is not the resolved text

- Category B, severity medium, confidence probable
- Features: COMMON-08
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/08_feature_guide_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/feature_guide_page-0000002271957437
- Why: setLabel expects a string. A TypeScript type assertion is not a runtime conversion, so the library receives an object where it expects text; any use of that label as a key or as displayed text will be wrong. The correct way to turn a string resource into text is resourceManager.getStringSync, which the sibling sample in this industry uses (CacheClean.zip#CacheClean/entry/src/main/ets/pages/CacheClean.ets:282 calls resourceManager.getStringSync($r('app.string.cachesize').id)).
- Fix: Resolve the resource before passing it: `const label = this.getUIContext().getHostContext()!.resourceManager.getStringSync($r('app.string.high_light_label').id); this.builder = new HighLightGuideBuilder().setLabel(label)`. Alternatively declare labelText as a plain string.

### `HW-19-0012` - EntryAbility reads the status-bar height from the TYPE_CUTOUT avoid area and adds an unexplained 40 to it

- Category B, severity medium, confidence confirmed
- Features: COMMON-08
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/08_feature_guide_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/feature_guide_page-0000002271957437
- Why: topRectHeight is consumed as the top padding of the home page (HomePage.ets:100, px2vp(this.topRectHeight)), so it must be the status-bar inset. TYPE_CUTOUT reports the notch/punch-hole region, which is zero on devices without a cutout and never equals the status-bar height on devices with one; the hardcoded +40 is a compensation constant that only happens to look right on the demo device. On a device with no cutout the page is padded by a fixed 40 px regardless of the real status bar.
- Fix: Read the status-bar inset from the documented type and drop the magic number: `let avoidAreaTop = windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM); AppStorage.setOrCreate('topRectHeight', avoidAreaTop.topRect.height);` and keep it current by subscribing to 'avoidAreaChange', as the CacheClean sample does.

### `HW-19-0013` - The sample declares ohos.permission.INTERNET although every page it loads is a local rawfile, so the permission is unnecessary

- Category D, severity medium, confidence confirmed
- Features: COMMON-09
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/09_backpage_by_gesture.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/backpage_by_gesture-0000002237371200
- Why: The official permission principles require the opposite of what the sample does: "Request only the least required permissions for your application. Do not apply for unnecessary or obsolete permissions. Excess permission requests can cause users to worry about application security and degrade user experience" (documentation/harmonyos-guides/04_system/app-permission-mgmt-overview.md:25). A reader copying this solution into a purely local H5 flow inherits a network permission the feature does not use.
- Fix: Remove the requestPermissions block from module.json5 for the local-only case, and change the document's 权限说明 to state that ohos.permission.INTERNET is required only when the Web component loads online pages - not for $rawfile pages.

### `HW-19-0016` - The onGestureSwipe snippet in the document drops the reverse-swipe branch, so the button fade and the indicator never come back when the user swipes away from the last page

- Category E, severity medium, confidence confirmed
- Features: COMMON-10
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/10_custom_swiper_guide_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_swiper_guide_page-0000002282653913
- Why: penetrability drives both the button opacity (opacity(1 - this.penetrability), SwiperGuide.ets:153) and its hit testing (hitTestBehavior, line 152), and visible drives the custom indicator (line 143). The omitted branch is the one that runs while the user drags backwards off the last page; without it the real-time gradient described by the document's own bullet - "在滑动页面过程中，根据页面偏移量的变化，实现按钮透明度的实时渐变效果" ("during the page swipe, implement a real-time gradient of the button opacity based on the change of the page offset") - only works in one direction, and the indicator stays hidden until the swipe animation ends.
- Fix: Add the missing branch to the document snippet so both swipe directions are shown, matching the sample.

### `HW-19-0020` - The document prescribes PersistentStorage.persistProp without mentioning that the official state-management guide advises using PersistenceV2.globalConnect instead

- Category F, severity medium, confidence confirmed
- Features: COMMON-11
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/11_edition_switch.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/edition_switch-0000002284505357
- Why: The two documents disagree about which API a new implementation should use. A reader following this solution builds the theme-persistence layer on the API the platform guide explicitly steers away from, and inherits the module-coupling problem the guide describes - which matters directly for this feature, since the layered architecture practice in the same industry (02_practice-common-app-layered-v1.md) expects the theme setting to be shared across several HAR/HSP modules.
- Fix: Note the official recommendation in the 实现思路 section and either switch the sample to PersistenceV2.globalConnect or state explicitly why PersistentStorage is used here (single-module demo) and link the PersistentStorage->PersistenceV2 migration guide.

### `HW-19-0022` - The createSubWindow callback ignores the BusinessError and dereferences the returned window immediately, so a failed creation crashes the eye-protection toggle

- Category B, severity medium, confidence confirmed
- Features: COMMON-12
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/12_custom_theme_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_theme_demo-0000002290124205
- Why: createSubWindow is invoked from a UI toggle (DisplayPage.ets:220), so it can be called again while a window named 'eyeWindow' still exists - precisely the 1300002 case. On that path `data` is not a valid Window, and the very next line calls moveWindowTo on it unconditionally, turning a recoverable error into a crash inside a callback that has no try/catch around it.
- Fix: Check the error before touching the result, as the official example does: `if (error.code) { LOGGER.error(...); return; } this.eyeWindowClass = data; ...`

### `HW-19-0023` - removeSubWithEyeWindow destroys the child window but keeps the stale reference and discards the result, so a second toggle-off calls destroyWindow on an already destroyed window

- Category B, severity medium, confidence confirmed
- Features: COMMON-12
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/12_custom_theme_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_theme_demo-0000002290124205
- Why: Two concrete defects follow. First, the previous Window object is leaked whenever createSubWithEyeWindow runs twice without an intervening successful destroy, because eyeWindowClass is the only handle to it. Second, disabling eye protection twice calls destroyWindow on a destroyed window and the resulting BusinessError is swallowed by the empty callback, so neither the developer nor the user learns that the state machine has diverged.
- Fix: Handle the callback error and clear the handle: `this.eyeWindowClass.destroyWindow((err: BusinessError) => { if (err.code) { LOGGER.error(...); return; } this.eyeWindowClass = null; });` and guard createSubWithEyeWindow with an early return when this.eyeWindowClass is already set.

### `HW-19-0024` - DisplayPage converts the @StorageProp('topRectHeight') from px to vp into the same variable, and the conversion is discarded the next time the avoid area changes

- Category C, severity medium, confidence confirmed
- Features: COMMON-12
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/12_custom_theme_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_theme_demo-0000002290124205
- Why: The component stores a vp value in a variable that AppStorage refills with a px value. The first avoidAreaChange event after the page appears - rotation, entering or leaving a full-screen video, a foldable state change, all reachable on the declared phone/tablet/2in1 device types - overwrites the converted value with the raw pixel count, and the page's top padding grows by the screen density factor.
- Fix: Keep the raw @StorageProp value and convert at the point of use: `.padding({ top: this.getUIContext().px2vp(this.topRectHeight) })`, as the sibling samples in this industry do (for example BackPageByGestures.zip#BackPageByGestures/entry/src/main/ets/pages/Index.ets:63). Never assign into a @StorageProp variable a value in different units from the one AppStorage holds.

### `HW-19-0026` - The 'result' subscription on each UserAuthInstance is never released with off('result'), so every authentication attempt leaves a live callback registration behind

- Category B, severity medium, confidence confirmed
- Features: COMMON-13
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/13_user_authentication.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/user_authentication-0000002290451165
- Why: getUserAuthInstance is called on every button tap (UserAuth.pullUpLock at UserAuthPage.ets:41 and UserAuth.fingerIdentification at UserAuthPage.ets:88), and each call creates a fresh instance held only by a local variable. Because the instance is never unsubscribed, the registration outlives the local reference and accumulates one per attempt - a subscription leak in the exact shape the reference's on/off pair is designed to avoid.
- Fix: Keep the instance and unsubscribe once the result has been delivered: `userAuthInstance.off('result');` at the end of onResult (for both the success and failure paths), or hold the instance in a field and unsubscribe in aboutToDisappear.

### `HW-19-0027` - onResult only handles UserAuthResultCode.SUCCESS, so every failure, lockout or cancellation is silently discarded with no user feedback

- Category B, severity medium, confidence confirmed
- Features: COMMON-13
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/13_user_authentication.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/user_authentication-0000002290451165
- Why: The page has no other completion path: authCallback (UserAuthPage.ets:140-142) only pops the navigation stack on success, so after a failed fingerprint or a cancelled PIN entry the user is returned to a screen that looks unchanged, with nothing logged and nothing shown. Cases such as a locked-out credential or an unenrolled authentication type are indistinguishable from the user simply not having tried.
- Fix: Handle the non-success codes: log result.result, and surface an appropriate message or fall back to another authType. At minimum add `else { Logger.error('auth failed, code ' + result.result); }` and unsubscribe on that branch too (see HW-19-0026).

### `HW-19-0028` - FileUtil discards every preview error in two empty catch blocks, leaves the downloadFile promise without a rejection handler and never unsubscribes the download 'complete' listener

- Category B, severity medium, confidence confirmed
- Features: COMMON-13
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/13_user_authentication.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/user_authentication-0000002290451165
- Why: This is the whole post-authentication path: after a successful identity check the app must show the contract PDF. If the download rejects (no network, expired URL) or canPreview/openPreview rejects, nothing is logged, nothing is shown, and the user is returned to a blank screen with no indication that anything failed - while the enclosing try/catch gives the false impression that the path is guarded. The un-released 'complete' listener keeps the task callback alive after the download has finished.
- Fix: Log the BusinessError in both catch blocks (code and message), add a .catch to downloadFile, and call downloadTask.off('complete', completeCallback) inside completeCallback once the file has been handed to the preview.

### `HW-19-0032` - The @Provide and @Consume declarations of the shared uris array use different element types, so the consumer treats Resource values as strings

- Category B, severity medium, confidence confirmed
- Features: COMMON-14
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/14_upload_picture.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/upload_picture-0000002256189702
- Why: The two decorators bind the same object, so the narrower @Consume type is a false statement about its contents: until a photo has been picked, elements 0 and 1 are Resource objects created by $r(). Every consumer-side use assumes string - handleFileList takes Array<string>, and the ArkTS type checker cannot flag the mismatch because it trusts the declared @Consume type. Keeping placeholder Resources and picked file URIs in one array is also what forces the wrong argument in HW-19-0031.
- Fix: Declare the same type on both sides, and separate the concerns: keep an Array<string> of picked URIs for handleFileList, and resolve the displayed image with a fallback, for example `Image(this.uris[0] ?? $r('app.media.front'))`.

### `HW-19-0035` - onShowFileSelector returns true on every path but five of them never call handleFileList, leaving the HTML file input unresolved

- Category B, severity medium, confidence probable
- Features: COMMON-15
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/15_webview_picker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/webview_picker-0000002257162014
- Why: Returning true transfers responsibility for completing the file selection to the application. Because the selection is launched asynchronously from the runJavaScript callback, every failure path returns true first and then quietly abandons the request, so the <input type="file"> that triggered it is left waiting with no result and no cancellation - the user sees a picker that opened and closed with nothing happening, and the form cannot be retried until the page is reloaded.
- Fix: Complete the selection on every path: call event.result.handleFileList([]) in the runJavaScript error branch, in an else for the falsy result, in the default branch, and in each picker's rejection handler. Extract a helper such as `const finish = (files: string[]) => event?.result.handleFileList(files);` so no path can be missed.

### `HW-19-0038` - The header opacity accumulates scroll deltas without clamping, so once the user has scrolled past the fade distance the title stays hidden even after scrolling back to the top

- Category C, severity medium, confidence confirmed
- Features: COMMON-16
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/16_tab_bar_blur.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/tab_bar_blur-0000002257193008
- Why: Because the rendered value is clamped but titleOpacity is not, a long downward scroll drives the stored number far below 0 (for example -8 after scrolling eight header-heights). Scrolling back to the very top then adds the same total back and returns it to 1 only if the upward deltas exactly balance - but the WaterFlow inside uses nested scrolling (WaterFlowContentList.ets:44-47, PARENT_FIRST forward and SELF_FIRST backward), so the outer Scroll receives different offsets in each direction and the sum does not come back. The header then stays invisible at the top of the page.
- Fix: Derive the opacity from the absolute scroll position rather than integrating deltas, and clamp it: `const y = this.scroller.currentOffset().yOffset; this.titleOpacity = Math.min(1, Math.max(0, 1 - y / TITLE_HEIGHT));` using a Scroller bound to the Scroll component.

### `HW-19-0041` - The RDB store holding account names and passwords is created at security level S1, the level the reference reserves for non-sensitive data

- Category D, severity medium, confidence confirmed
- Features: COMMON-17
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/17_silent_login.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/silent_login-0000002292499361
- Why: A phone number and a password are not comparable to the wallpapers the reference gives as the S1 example. The same page also warns that the choice is one-way - "You cannot change the security level of an RDB store from a higher level to a lower one" - and that the level governs cross-device sync access control, so a credential store left at S1 can be synchronised to peers that should not receive it. Choosing the level too low at creation time cannot be corrected later on existing installations.
- Fix: Create the store at a level matching the data - S3 or higher for authentication credentials - and pick it before the first release, since the level cannot be lowered afterwards and existing stores keep the level they were created with.

### `HW-19-0042` - The password field is bound to the same TextInputController as the user-name field, while the second controller declared for it is never used

- Category B, severity medium, confidence confirmed
- Features: COMMON-17
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/17_silent_login.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/silent_login-0000002292499361
- Why: A TextInputController drives a single TextInput - it is what caret positioning and the keyboard-control APIs act on. Binding one controller to two inputs makes any controller call ambiguous about which field it targets, and the presence of an unused controller2 shows the second binding was intended to be distinct. This is exactly the kind of copy-paste defect that surfaces later, when someone adds a caretPosition or stopEditing call and it moves the wrong field.
- Fix: Bind the password TextInput to this.controller2.

### `HW-19-0043` - The remember-me Checkbox hardcodes select(false) and toggles the persisted flag by negation, so the box never shows the state that was actually persisted

- Category C, severity medium, confidence confirmed
- Features: COMMON-17
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/17_silent_login.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/silent_login-0000002292499361
- Why: The whole feature is that the choice persists: PersistentStorage keeps isRemembered across launches, and MainPage skips the login page when it is true. But the checkbox always renders unchecked, so a returning user whose preference is stored as true sees an unticked box; ticking it then flips the stored value to false and silently turns silent login off. Deriving the new value by negation rather than from the callback argument also lets the UI and the state diverge whenever the component's own state changes for any other reason.
- Fix: Bind the attribute to the state and use the callback value: `.select(this.isRemembered).onChange((value: boolean) => { this.isRemembered = value; })`.

### `HW-19-0047` - The document's counter and elapsed-time arithmetic does not match the sample, where the call counter is assigned 1 instead of incremented and three of four pages drop the running total

- Category E, severity medium, confidence confirmed
- Features: COMMON-18
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/18_code_instrumentation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/code_instrumentation-0000002291963849
- Why: The document teaches accumulate-and-count; the sample it links to does neither consistently. With `count = 1` the per-visit counter can never exceed 1, so a page visited several times in one session still contributes 1 to the persisted 次数 ("count") figure, and with `required = end - begin` on three pages a session's repeated loads contribute only the last measurement. The two are also inconsistent with each other across the four pages of the same sample, so a reader cannot tell which form is intended.
- Fix: Make the sample match the document: `count += 1;` and `required = end - begin + required;` in all four pages, and reset both to 0 at the point where they are folded into the persisted totals so nothing is double-counted.

### `HW-19-0049` - Each image download creates an HttpRequest that is never destroyed, which the reference names as a memory-leak cause

- Category B, severity medium, confidence confirmed
- Features: COMMON-18
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/18_code_instrumentation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/code_instrumentation-0000002291963849
- Why: downloadNetworkImage runs once per page visit across four pages, and each call leaks one HttpRequest. The reference states the consequence directly - memory leaks - and requires a per-request object precisely so that each one is released.
- Fix: Hold the request object and release it in a finally block: `const httpRequest = http.createHttp(); try { const data = await httpRequest.request(url, options); ... } finally { httpRequest.destroy(); }`

### `HW-19-0051` - The BLEDeviceFind subscription is only released when at least one device was found, because the tracking flag is set inside the callback itself

- Category B, severity medium, confidence confirmed
- Features: COMMON-19
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/19_search_and_connect_ble.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_and_connect_ble-0000002293947117
- Why: A scan that finds nothing - the common case when no device is in range - leaves the callback registered. The next startBLEScan then registers a second one, and each subsequent scan adds another, so device reports are delivered to every stale listener and the ScanListener is invoked once per accumulated registration. The flag is meant to record 'a listener is registered', which is decided at ble.on time, not at first callback.
- Fix: Set BleUtil.isBLEDeviceFind = true immediately after the ble.on('BLEDeviceFind', ...) call rather than inside the callback body.

### `HW-19-0053` - The permission's usedScene names an ability called DefaultAbility, which does not exist in the module

- Category E, severity medium, confidence confirmed
- Features: COMMON-19
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/19_search_and_connect_ble.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_and_connect_ble-0000002293947117
- Why: usedScene.abilities is meant to list the abilities that use the permission, so that the declaration describes the real usage. Naming an ability that does not exist in the module makes the declaration self-inconsistent and misleading in review, and it is the kind of stale copy-paste that silently survives a module rename.
- Fix: Change the usedScene abilities entry to ["EntryAbility"], matching the ability actually declared in this module.

### `HW-19-0054` - The Bluetooth switch state is polled on a 100 ms timer instead of being subscribed to through access.on('stateChange')

- Category C, severity medium, confidence confirmed
- Features: COMMON-19
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/19_search_and_connect_ble.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_and_connect_ble-0000002293947117
- Why: The Bluetooth switch changes rarely; polling it ten times a second runs a cross-process getState call on the UI thread for the entire lifetime of the page. The sibling performance practice in this industry states the rule directly - 避免在自定义组件的生命周期内执行高耗时操作 ("avoid executing time-consuming operations inside custom component lifecycle callbacks", 04_practice-common-app-performance-v1.md:11) - and the platform provides exactly the callback that removes the need to poll.
- Fix: Subscribe once in aboutToAppear with access.on('stateChange', (state) => { ... }) and release it with access.off('stateChange') in aboutToDisappear, dropping the interval.

### `HW-19-0056` - The scan-result handler reads only data[0] of the reported result set, so every additional device in the same report is discarded and an empty array would throw

- Category B, severity medium, confidence confirmed
- Features: COMMON-19
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/19_search_and_connect_ble.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_and_connect_ble-0000002293947117
- Why: The callback delivers a set, not a single device: a report carrying several ScanResults contributes only its first entry, so devices are missed at exactly the moment the scan is most productive. The receiving data source is already written to handle repeated and multiple devices - ManDeviceDataSource.pushData deduplicates by address and refreshes rssi (ManDeviceDataSource.ets:76-85) - so nothing but the caller prevents the full set from being processed. Indexing [0] without checking data.length also throws on an empty report.
- Fix: Iterate the array: `for (const result of data) { ... this.scanDeviceList.pushData(bean); }`, and guard with `if (!data || data.length === 0) { return; }` before any indexing.

### `HW-19-0057` - Two DeviceManager write methods return promises whose executor never calls resolve or reject, so awaiting a characteristic or descriptor write hangs forever

- Category B, severity medium, confidence confirmed
- Features: COMMON-19
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/19_search_and_connect_ble.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_and_connect_ble-0000002293947117
- Why: Both methods are declared Promise<void>, so a caller that awaits them - the natural use for a write that must complete before the next GATT operation - blocks permanently, and any code sequenced after the await never runs. The failure is silent: no rejection is ever delivered either, so a write error is invisible to the caller and appears only in the log.
- Fix: Take the executor parameters and settle the promise from the write callback: `return new Promise<void>((resolve, reject) => { try { this.clientDevice?.writeCharacteristicValue(characteristic, ble.GattWriteType.WRITE, (err: BusinessError) => { if (err) { reject(err); } else { resolve(); } }); } catch (err) { reject(err as BusinessError); } });` and the same shape for writeDescriptorValue.

### `HW-19-0058` - handelShareImage closes the same sandbox file twice - once by FD inside the try and once by File object in the finally

- Category B, severity medium, confidence confirmed
- Features: COMMON-20
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/20_screenshot_listen.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/screenshot_listen-0000002261676460
- Why: The second close acts on a descriptor that has already been released, so it either fails with a Basic File IO error - which the finally block does not guard, so it would escape the promise handler - or, worse, closes a descriptor number that the runtime has since reused for another file. The correct pattern is one close on one path; the finally exists precisely so the close does not need to be in the try body.
- Fix: Remove the fileIo.closeSync(file.fd) call from the try block and keep only the finally, so the file is closed exactly once whether or not the write succeeds: `try { fileIo.writeSync(file.fd, buffer); ... } finally { fileIo.closeSync(file); }`

### `HW-19-0059` - saveImage closes the sandbox file only on the success path, so a failed write leaks the file descriptor

- Category B, severity medium, confidence confirmed
- Features: COMMON-20
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/20_screenshot_listen.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/screenshot_listen-0000002261676460
- Why: saveImage runs on every screenshot the user takes, so a repeated failure - a full disk, a permission problem on the sandbox path - leaks one descriptor per attempt for the lifetime of the process. The catch block reports 保存失败 ("save failed") to the user, which makes the leak invisible.
- Fix: Move the close into a finally: `try { fileIo.writeSync(file.fd, buffer); ... } catch (error) { ... } finally { fileIo.closeSync(file); }`, and drop the close from the try body. Fix the same snippet in the document's step 3.

### `HW-19-0060` - The screenshot PixelMap is replaced on every screenshot and never released

- Category B, severity medium, confidence confirmed
- Features: COMMON-20
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/20_screenshot_listen.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/screenshot_listen-0000002261676460
- Why: Each screenshot allocates a new full-screen bitmap at scale 2 - on a 1260x2720 phone that is roughly 27 MB of RGBA - and the previous one is dropped without being released. Repeated screenshots while the page is open therefore accumulate native image buffers that the guide expects the application to hand back explicitly. aboutToDisappear unsubscribes the listener but does not release the last captured PixelMap either.
- Fix: Release the previous PixelMap before replacing it and on page exit: `const next = await this.screenshot(); this.pixmap?.release(); this.pixmap = next;` and in aboutToDisappear `this.pixmap?.release(); this.pixmap = undefined;` after the windowClass.off('screenshot') call.

### `HW-19-0061` - The document has no 权限说明 section although the sample declares two user-grant location permissions and requests them at runtime

- Category E, severity medium, confidence confirmed
- Features: COMMON-20
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/20_screenshot_listen.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/screenshot_listen-0000002261676460
- Why: Both permissions are user_grant and produce a runtime dialog, so a reader reproducing the sample must know they are needed and why. The omission is also misleading in the other direction: the two location permissions belong to the shopping shell the sample is built on, not to the screenshot-listening feature the document teaches, and with no permission section the reader cannot tell which of the two the feature actually requires - the answer being neither.
- Fix: Add a 权限说明 section stating that the screenshot listening, saving and sharing feature itself requires no permission, and that ohos.permission.LOCATION and ohos.permission.APPROXIMATELY_LOCATION in the sample belong to the surrounding shopping demo's address lookup.

### `HW-19-0062` - The share dialog's preview receives the PixelMap once, at controller construction, when it is still undefined - so the dialog never shows the screenshot it is previewing

- Category C, severity medium, confidence confirmed
- Features: COMMON-20
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/20_screenshot_listen.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/screenshot_listen-0000002261676460
- Why: The two callbacks work because they are closures that read this.pixmap when invoked, but the preview is not a closure - it is a plain value copied when the controller was constructed, which is at component construction time, before any screenshot exists. The dialog therefore always renders an empty preview area, defeating the purpose of the screenshot-share prompt the document describes ("应用检测到用户截屏操作，会弹出分享框提示", "when the application detects the user's screenshot action, it pops up a share prompt").
- Fix: Bind the preview to the state rather than copying it: decorate the dialog member as @Link (`@Link pixmap: image.PixelMap | undefined;`) and pass `pixmap: $pixmap` from the controller, or build the controller lazily inside the screenshot callback so the current PixelMap is captured.

### `HW-19-0064` - unitConversion labels the megabyte range as KB, so every file between 1 MB and 1 GB is displayed with the wrong unit

- Category B, severity medium, confidence confirmed
- Features: COMMON-21
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/21_agent_download_control.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/agent_download_control-0000002262990970
- Why: This is the size shown for every downloaded file: FileItem renders `fileUtils.unitConversion(this.downloadProgressNum)` during download and `fileUtils.getFileSizeByName(this.fileName)` when complete, and the file list page formats sizes the same way. A 5 MB download is displayed as '5.0KB', understating the size by a factor of 1024 across the entire range that ordinary downloads fall into.
- Fix: Change the third branch's format string to '%sMB', and correct the '@param size bit' comment to bytes.

### `HW-19-0065` - The three download-task event subscriptions are never released with off(), and the component has no aboutToDisappear

- Category B, severity medium, confidence confirmed
- Features: COMMON-21
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/21_agent_download_control.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/agent_download_control-0000002262990970
- Why: The subscriptions outlive the component. A FileItem that is scrolled out of the list or whose page is closed while a download is running keeps receiving progress events and keeps writing into the state of a component that is no longer displayed, and the closure holds the component alive. The failure is compounded by the shape of the flow: a new Task is created on every STATUS_PREPARE tap (create() -> downloadTaskInit()), so re-downloading the same file registers three more callbacks without releasing the previous three.
- Fix: Add an aboutToDisappear that releases them: `this.downloadTask?.off('progress'); this.downloadTask?.off('completed'); this.downloadTask?.off('failed');` and do the same in the 'completed' and 'failed' handlers, next to the existing request.agent.remove call.

### `HW-19-0066` - The UI is switched to the downloading state even when task creation failed, leaving a row that shows a download in progress forever

- Category B, severity medium, confidence confirmed
- Features: COMMON-21
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/21_agent_download_control.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/agent_download_control-0000002262990970
- Why: request.agent.create can fail for ordinary reasons - a malformed url, no network, the task-count quota - and the catch reports it only to the log, at info level. The row nevertheless switches to STATUS_DOWNLOADING, which swaps the icon to the downloading glyph and renders the progress bar at 0%. Because no task exists, no progress, completed or failed event will ever arrive, so the row stays at 0% permanently and the next tap is interpreted as pause. The user is shown a download that was never started.
- Fix: Make create() report success - return the created Task or a boolean - and only advance the status when it did: `const task = await this.create(fileName); if (!task) { /* toast the failure */ return; } await this.start(); this.status = Constants.STATUS_DOWNLOADING;`. Log the creation failure with hilog.error rather than hilog.info.

### `HW-19-0068` - The navDestinationSwitch handler indexes the page stack at position 0 without checking that it is non-empty

- Category B, severity medium, confidence confirmed
- Features: COMMON-22
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/22_page_tracing_point.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/page_tracing_point-0000002283641138
- Why: All three listeners are registered together in initPageTracing, so navDestinationSwitch can fire before any NavDestination has reported ON_APPEAR - and it also fires on every return to navBar, including after ON_DISAPPEAR has emptied the stack. In those cases this.pageStack[0] is undefined and getUniqueMarkId throws while reading info.name, inside an observer callback that has no try/catch, so the exception escapes the tracing layer into the framework's event dispatch. The two other listeners already guard their map lookups with `?? 0`; only this one dereferences unconditionally.
- Fix: Guard before use: `if (this.pageStack.length === 0) { return; } const rootInfo = this.pageStack[0];` in both branches, and make getUniqueMarkId tolerate an undefined argument.

### `HW-19-0069` - The page stack is mutated with removeByIndex from inside its own forEach traversal, in three places

- Category B, severity medium, confidence probable
- Features: COMMON-22
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/22_page_tracing_point.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/page_tracing_point-0000002283641138
- Why: removeByIndex shifts every later element down by one while the traversal continues from the index it was already on, so the element immediately after a removed one is skipped. With one matching entry the visible behaviour is correct, which is why the defect survives; with two entries carrying the same markId - the same page opened twice, which the accessCount logic explicitly expects - the second is never removed and the stack keeps a stale entry that then confuses the ON_PAGE_HIDE branch, whose duration accounting reads this.pageStack[index + 1].
- Fix: Collect first, then remove - or use the container's own removal by value: `const stale = this.pageStack.getIndexOf(...)` / build an index list in the forEach and remove from the highest index downwards after the traversal completes.

### `HW-19-0070` - EntryAbility derives the top inset from the TYPE_CUTOUT avoid area plus a hardcoded 20, instead of from the status-bar area

- Category B, severity medium, confidence confirmed
- Features: COMMON-22
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/22_page_tracing_point.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/page_tracing_point-0000002283641138
- Why: topRectHeight is published to AppStorage and consumed as the top padding of the pages, so it must be the status-bar inset. TYPE_CUTOUT reports the notch or punch-hole region, which is zero on devices without one - there the padding becomes a flat 20 vp regardless of the real status bar - and on devices with one it is not the status-bar height either. The +20 is an unexplained compensation constant that only happens to look right on the demo device. The same defect appears in the FeatureGuidePage sample of this industry (HW-19-0012), so it is a pattern being propagated rather than a one-off.
- Fix: Read the status-bar inset from the documented type and drop the constant: `let systemArea = mWindow.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM); AppStorage.setOrCreate('topRectHeight', uiContext.px2vp(systemArea.topRect.height));`, and subscribe to TYPE_SYSTEM rather than TYPE_CUTOUT in the avoidAreaChange handler.

### `HW-19-0071` - getTime tests the minutes boundary before the hours boundary, so the hours branch is unreachable and long sessions are reported in minutes

- Category B, severity medium, confidence confirmed
- Features: COMMON-22
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/22_page_tracing_point.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/page_tracing_point-0000002283641138
- Why: This function renders the 访问总时间 ("total visit time") column that the document names as the deliverable of step 4. A two-hour accumulated visit is 7200 seconds, which takes the minutes branch and is displayed as '120分' instead of '2小时'; the hours unit never appears at all, no matter how long the user browses. The ordering is the whole defect - the branches are individually correct.
- Fix: Test the larger boundary first: `if (second >= CommonConstant.HOURS_BOUNDARY) { ... hours ... } else if (second >= CommonConstant.MINUTES_BOUNDARY) { ... minutes ... } else { ... seconds ... }`.

### `HW-19-0072` - HUNDRED_MILLION is defined as 1,000,000,000, so counts formatted with the 亿 suffix are ten times too small

- Category B, severity medium, confidence confirmed
- Features: COMMON-22
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/22_page_tracing_point.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/page_tracing_point-0000002283641138
- Why: The constant is both the branch threshold and the divisor, so the error compounds: values between 10^8 and 10^9 are shown in 万 rather than 亿 (10^8 renders as '10000.0万'), and values above 10^9 are divided by ten times too much, so 2x10^9 renders as '2.0亿' when it is 20亿. Since the same helper formats the 访问次数 ("visit count") column the document's step 4 describes, the displayed statistic is simply wrong at that magnitude.
- Fix: Set `static readonly HUNDRED_MILLION: number = 100000000;` (10^8), matching the 亿 suffix it is paired with.

### `HW-19-0075` - scan() encodes success, failure and permission denial all as plain strings, and the caller distinguishes them by searching the scanned text for 'errCode'

- Category B, severity medium, confidence confirmed
- Features: COMMON-23
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/23_web_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_scan-0000002284631808
- Why: The barcode content is attacker-chosen. A QR code whose text contains the substring 'errCode' - or a code whose text is exactly 'No camera permission' - is classified as a failure and silently swallowed, so a legitimate scan is dropped. The classification is also not reliable in the other direction: any decoded value that happens not to contain those markers is treated as a success and drives a navigation. Encoding control flow in the same string as untrusted data is the defect; the two are not separable afterwards.
- Fix: Return a discriminated result rather than a bare string, for example `{ ok: true, value } | { ok: false, code, message } | { ok: false, denied: true }`, and branch on the flag in handleScanResult. Since the object crosses the JavaScript proxy boundary, serialise it as JSON with an explicit status field instead of relying on substring matching.

### `HW-19-0077` - The card list's key generator names its parameter 'index' but receives the data item, so the key degenerates to '[object Object]' and the framework falls back to appending the array index

- Category C, severity medium, confidence confirmed
- Features: COMMON-24
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/24_pull_back.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pull_back-0000002320205289
- Why: The intended key was the card index; what the code actually computes is a constant string, and the only thing that keeps the keys distinct is the framework's automatic index suffix - the fallback the guide names as a source of duplicate-key problems. The result is the index-as-key anti-pattern the same guide warns against, on a list whose items are the source elements of a geometryTransition: if allCardData is ever reordered or filtered, the identity of each card - and therefore the shared-element transition bound to `this.index.toString()` in CardItem.ets - follows the position rather than the data.
- Fix: Declare both parameters and key on stable data: `}, (cardData: CardData, index: number) => cardData.title);` - or any field that uniquely identifies the card. Keep the geometryTransition identifier derived from the same stable value rather than from the array position.

### `HW-19-0079` - The document's only code snippet reads the accessibility strings from CommonConstants, but the sample keeps them in a separate AccessibilityConstants class and CommonConstants.SCAN does not exist

- Category E, severity medium, confidence confirmed
- Features: COMMON-25
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/25_accessibility.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/accessibility-0000002320999085
- Why: This is the document's single snippet for the single technique it teaches, and it does not compile against the project it links to: CommonConstants.SCAN is undefined. It also loses the separation the sample deliberately makes - screen-reader strings in AccessibilityConstants, layout and display strings in CommonConstants - which is the part of the design worth copying.
- Fix: Change the snippet to use AccessibilityConstants.SEARCH and AccessibilityConstants.SCAN for the accessibilityText calls, keeping CommonConstants for the placeholder and the layout values, exactly as HomePage.ets does.

### `HW-19-0080` - The grid item container sets accessibilityText without accessibilityGroup, so its child Texts stay individually focusable and the title is announced twice

- Category C, severity medium, confidence confirmed
- Features: COMMON-25
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/25_accessibility.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/accessibility-0000002320999085
- Why: Without grouping the container and its children are separate screen-reader stops: the container announces item.title, and the Text child immediately after it announces the same item.title again, so the user hears the title twice while stepping through one tile. The second line of the tile, Text(item.component), is meanwhile excluded from the container's announcement because accessibilityText replaces rather than extends it - the reference notes "If a component has both text content and accessibility text, only the accessibility text is announced." The result is a duplicated title and a subtitle that only appears on a separate stop, in the sample whose entire purpose is to demonstrate correct screen-reader behaviour.
- Fix: Add `.accessibilityGroup(true)` to the grid item Column alongside its accessibilityText, and make the accessibility text cover both lines - for example `.accessibilityText(`${item.title} ${item.component}`)` - matching the tabBuilder and Charge.ets pattern.

### `HW-19-0082` - Both window.getLastWindow callbacks ignore the BusinessError and use the window object unconditionally

- Category B, severity medium, confidence confirmed
- Features: COMMON-26
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/26_splash_page_ad_access.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/splash_page_ad_access-0000002322822425
- Why: On the failure path the second callback argument is not a usable Window, and the very next statement calls setWindowLayoutFullScreen on it - turning a reportable error into a crash inside a lifecycle callback that has no try/catch. This runs on the application's first screen, where a window that is not yet available is exactly the plausible case, and the aboutToDisappear copy is the one that restores the orientation, so a silent failure there leaves the whole application locked in portrait.
- Fix: Check the error before using the window, as the reference example does: `if (err.code) { hilog.error(...); return; }`, in both callbacks.

### `HW-19-0085` - The bottom tab bar builds ListItem components inside a plain Row, but ListItem's parent may only be List or ListItemGroup

- Category C, severity medium, confidence confirmed
- Features: COMMON-27
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/27_display_after_login.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/display_after_login-0000002293948070
- Why: ListItem is not a generic container - it exists to participate in a List's layout, selection, swipe actions and item recycling, all of which need a List parent. Placing it under a Row puts the component outside the contract the reference defines, so its behaviour is unspecified: it contributes no list semantics, and the wrapping only obscures that the bar is really five Columns in a Row. The five tab items also carry no onClick, so the bar is decorative.
- Fix: Drop the ListItem wrapper and place the tab item directly in the Row: `ForEach(this.data, (item: TabData) => { this.tabItem(item); });` - or, if list semantics are wanted, replace the Row with a List and keep the ListItem.

### `HW-19-0087` - showAssetsCreationDialog is given the raw sandbox path instead of a file:// URI, so the authorisation dialog cannot preview the image being saved

- Category A, severity medium, confidence confirmed
- Features: COMMON-28
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/28_press_to_save_or_copy.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/press_to_save_or_copy-0000002328179937
- Why: The whole point of showAssetsCreationDialog is an authorisation dialog the user can act on; with a sandbox path instead of a URI the dialog cannot show what is about to be saved, so the user is asked to approve an unidentified image. The sibling sample in this industry does the conversion correctly - ScreenshotSharing.zip#ScreenshotSharing/entry/src/main/ets/utils/SaveImage.ets builds its srcFileUris with fileUri.getUriFromPath(...).
- Fix: Convert the sandbox path before passing it: `import { fileUri } from '@kit.CoreFileKit'; let srcFileUris: Array<string> = [fileUri.getUriFromPath(imgPath)];` and open the source file from the same URI.

### `HW-19-0088` - A failed image download is reported only to the log at info level, so the save silently does nothing from the user's point of view

- Category B, severity medium, confidence confirmed
- Features: COMMON-28
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/28_press_to_save_or_copy.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/press_to_save_or_copy-0000002328179937
- Why: The user long-presses an image, taps 保存 ("save"), the popup closes, and if the download fails nothing at all happens - no image in the gallery, no message. The failure is indistinguishable from a slow network, and it is logged at the same level as the routine progress lines from onDownloadUpdated, so it is hard to find even in the log. The error code that would explain it, getLastErrorCode(), is captured and then discarded into an info line.
- Fix: Report the failure to the user and log it at error level: `Logger.error(...); showToast(this.getUIContext(), $r('app.string.save_failure'));` inside onDownloadFailed.

### `HW-19-0089` - The Web component enables application file-system access, which nothing in this sample needs

- Category D, severity medium, confidence confirmed
- Features: COMMON-28
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/28_press_to_save_or_copy.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/press_to_save_or_copy-0000002328179937
- Why: fileAccess(true) grants the loaded page the ability to read the application's file system over the file protocol. This page has no use for it - rawfile loading is explicitly unaffected by the setting - so the capability is granted for nothing, on a component whose content includes remote images. The sibling H5 samples in this industry that do not need it set the opposite: BackPageByGestures.zip#BackPageByGestures/entry/src/main/ets/pages/WebPage.ets:34 and UploadPicture.zip#UploadPicture/entry/src/main/ets/pages/UploadPicture.ets:118 both call .fileAccess(false).
- Fix: Set .fileAccess(false), or drop the call entirely since API 12 defaults it to false. If a future version of the page really needs local files, prefer setPathAllowingUniversalAccess with an explicit path list.

### `HW-19-0090` - The user agreement and privacy policy text is hardcoded as Chinese string literals in Constants.ets, although the project ships localised en_US and zh_CN resources for every other string

- Category B, severity medium, confidence confirmed
- Features: COMMON-29
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/29_user_agreement_and_privacy_policy.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/user_agreement_and_privacy_policy-0000002331953689
- Why: This is the text the user is asked to legally consent to. Everything framing it - the sheet title, the 同意 and 取消 buttons, the two detail-page titles - is resolved through $r() and therefore switches with the device locale, while the agreement and policy bodies stay Chinese. A user on an en_US device sees an English title and English buttons above a Chinese consent text, and taps 'Agree'. Consent obtained that way is not informed consent, and the split is a maintenance hazard: legal review updates the resource files and misses Constants.ets.
- Fix: Move AGREEMENT_AND_PRIVACY_CONTENT_LIST, DETAIL and PRIVACY_DETAIL into resource string arrays (base plus each supported locale) and read them with $r()/resourceManager, as the rest of the page already does.

### `HW-19-0093` - onChange scrolls the tab bar to index - 1, which is -1 for the first tab and is therefore ignored

- Category B, severity medium, confidence confirmed
- Features: COMMON-30
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/30_tab_toggle_animation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/tab_toggle_animation-0000002332006785
- Why: The document describes this step as 将自定义TabBar滚动至当前选中页签 ("scroll the custom TabBar to the currently selected tab"). The -1 offset is there to keep one tab visible to the left of the selected one, but for index 0 it produces -1, which the reference says is abnormal and performs no scrolling at all. So switching back to the first tab - by swipe or by tap - leaves the tab bar wherever it was, and the first tab can remain scrolled off screen while its content is displayed. The failure is silent: scrollToIndex neither throws nor reports.
- Fix: Clamp the target: `this.scroller.scrollToIndex(Math.max(index - 1, 0), true);`

### `HW-19-0094` - The indicator's left margin is seeded from the tab's local position but updated from its global position, mixing two coordinate systems in one state variable

- Category C, severity medium, confidence probable
- Features: COMMON-30
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/30_tab_toggle_animation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/tab_toggle_animation-0000002332006785
- Why: indicatorLeftMargin is a margin inside a Stack, so it must be measured in the Stack's coordinate space. It happens to work in this layout because the Stack sits at the window's left edge and the page applies only safeAreaPadding({ top }), making global and local x coincide. Any left inset on an ancestor - a side padding, a two-column layout on a tablet or a foldable, an RTL mirror - makes the two disagree, and the indicator jumps by that inset the first time a tab is switched. The two readings should not be mixed in one variable regardless of whether the current layout hides the difference.
- Fix: Store and consume one coordinate space. Since the value is used as a margin inside the Stack, use newValue.position.x for the map as well: `this.tabPosWidthMap.set(index, { 'left': Number.parseFloat(newValue.position.x.toString()), 'width': width });`

### `HW-19-0095` - The interception snippet declares the callback parameter as _to but uses to in the body, so the document's code does not compile

- Category E, severity medium, confidence confirmed
- Features: COMMON-31
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/31_nav_mode_change.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/nav_mode_change-0000002298406768
- Why: This is the document's snippet for step 1, the step that implements the entire feature - switching the Navigation mode before the authentication page appears. Pasted as written it fails to resolve `to`, and the underscore prefix is a deliberate 'unused parameter' marker, so a reader is likely to keep it and rename the uses instead of the declaration. The sample shows the intended form: underscore-prefix the three parameters that really are unused and leave `to` plain.
- Fix: Change the parameter list in the document snippet to `(_from, to, _operation, _isAnimated)`, matching the sample.

### `HW-19-0096` - breakpointChange pushes the placeholder NonePage whenever the stack is empty, including at the small breakpoint the guard above it exists to protect

- Category B, severity medium, confidence confirmed
- Features: COMMON-31
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/31_nav_mode_change.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/nav_mode_change-0000002298406768
- Why: NonePage exists only to occupy the detail column in Split mode, as the document's step 3 explains - 双栏模式下...将空页压入路由栈中 ("in two-column mode, push an empty page onto the route stack"). At the SM breakpoint there is no detail column, so the placeholder covers the home page with a blank screen. The empty-stack case is reachable on the ordinary path: VerifyPage calls this.pathStack.clear() before pushing, and the SM branch's own pop(false) empties the stack, so a subsequent breakpoint notification finds length 0.
- Fix: Guard the second branch by breakpoint as well: `} else if (this.currentBreakpoint !== BreakpointConstant.BREAKPOINT_SM && pathNames.length === 0) { this.pathStack.pushPath({ name: 'NonePage' }); }`

### `HW-19-0097` - EntryAbility derives the top inset from the TYPE_CUTOUT avoid area plus a hardcoded 20, and never releases the avoidAreaChange subscription

- Category B, severity medium, confidence confirmed
- Features: COMMON-31
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/31_nav_mode_change.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/nav_mode_change-0000002298406768
- Why: topRectHeight is published to AppStorage and consumed as the top inset by HomePage and VerifyPage through @StorageProp. TYPE_CUTOUT reports the notch region, which is zero on devices without one - there the inset degenerates to a flat 20 vp regardless of the real status bar - and is not the status-bar height on devices with one. The device set includes tablets and 2in1, where cutouts are uncommon. This is the third sample in this industry shipping the identical expression (see HW-19-0012 and HW-19-0070), so it is a pattern being copied rather than a one-off.
- Fix: Read the status-bar inset from the documented type and drop the constant: `let systemArea = mWindow.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM); AppStorage.setOrCreate('topRectHeight', uiContext.px2vp(systemArea.topRect.height));`, subscribe to TYPE_SYSTEM in the change handler, and release the subscription with mWindow.off('avoidAreaChange') in onWindowStageDestroy alongside the observer.

### `HW-19-0098` - The windowSizeChange subscription is never released, and onWindowStageDestroy only logs

- Category B, severity medium, confidence confirmed
- Features: COMMON-32
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/32_app_multiplier.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_multiplier-0000002312850858
- Why: The callback closes over the ability instance and writes into AppStorage on every resize. Leaving it registered when the window stage is destroyed means the ability object cannot be released and the handler keeps running against a window the application no longer owns - on a foldable, the exact device this feature targets, fold and unfold events fire constantly. The comment in onWindowStageDestroy already states the intent ('release UI related resources'); only the call is missing.
- Fix: Release it in onWindowStageDestroy: `if (this.windowObj) { this.windowObj.off('windowSizeChange'); }`, keeping a reference to the callback if only that one registration should be removed.

### `HW-19-0101` - The font selector toggles between indices 0 and 1 only, so every font past the first one discovered by the rawfile scan is unreachable

- Category B, severity medium, confidence confirmed
- Features: COMMON-33
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/33_h5_load_custom_font_library.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5_load_custom_font_library-0000002349762829
- Why: The discovery loop is written to be general - it enumerates the directory and accepts any number of fonts - but the only consumer can address just two entries. Dropping a second .ttf into rawfile, which is exactly what the feature invites, adds a font that can never be selected. The two halves of the feature disagree about how many fonts exist.
- Fix: Cycle through the whole list: `this.fontIndex = (this.fontIndex + 1) % this.fontFamilies.length;` - or present the discovered fonts in a picker rather than a toggle.

### `HW-19-0103` - The download prompt treats any dismissal of the dialog as confirmation, because the ShowDialogSuccessResponse index is never inspected

- Category B, severity medium, confidence confirmed
- Features: COMMON-34
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/34_app_pull_up.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_pull_up-0000002353615821
- Why: showDialog resolves when the dialog closes, not only when the single button is tapped: a tap outside the dialog or a back press also settles it. Because the handler ignores the response, every one of those paths launches AppGallery, so a user who dismisses the prompt is still sent out of the application to a store page. The dialog is presented as a choice - 提示 / 下载 with a 确认 button - and behaves as if there were none.
- Fix: Inspect the response before acting: `.then((result: promptAction.ShowDialogSuccessResponse) => { if (result.index === 0) { /* launch AppGallery */ } })`, and add a cancel button so dismissal is a real option.

### `HW-19-0104` - The not-installed fallback always opens the AppGallery page of one hardcoded bundle, whatever scheme the page actually asked for

- Category B, severity medium, confidence confirmed
- Features: COMMON-34
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/34_app_pull_up.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_pull_up-0000002353615821
- Why: The feature is described as generic - 已安装则拉起，未安装则打开应用市场进行下载 ("if installed, launch it; if not installed, open the app market to download it") - but the download half only ever works for the single application the demo was built around. A page that links to any other app sends the user to the wrong store page, which is worse than doing nothing: the user installs an unrelated application and the original link still fails.
- Fix: Derive the target from the link. Keep a scheme-to-bundle map alongside querySchemes - the two lists have to be maintained together anyway - and build the store URI from it: `const bundleName = SCHEME_TO_BUNDLE.get(schemeOf(url)); if (!bundleName) { return; } uri: `store://appgallery.huawei.com/app/detail?id=${bundleName}` `.

### `HW-19-0106` - Gallery images are decoded at 30x30 while being displayed at roughly 100 vp high and 95% of a third of the screen, so every thumbnail is upscaled from a 30-pixel bitmap

- Category C, severity medium, confidence confirmed
- Features: COMMON-35
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/35_title_and_content_linkage.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/title_and_content_linkage-0000002365395897
- Why: sourceSize is the decode resolution, not a display hint. Decoding to 30x30 and then stretching that bitmap to a 100 vp cell with ImageFit.Fill produces a visibly blurred thumbnail on every card in the gallery - roughly a 3x to 10x upscale depending on screen density. The optimisation the comment describes is real, but the number has to be chosen from the rendered size (the cell's width and height in px), not picked arbitrarily.
- Fix: Set the decode size to the actual rendered size in pixels, for example `.sourceSize({ width: vp2px(cellWidth), height: vp2px(100) })`, or drop the attribute and let the component decode at natural size if the source images are already small.

### `HW-19-0109` - Two of the three branches set the custom user agent in onControllerAttached while the page is already loading from src, the ordering the reference warns breaks backward navigation

- Category A, severity medium, confidence confirmed
- Features: COMMON-36
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/36_web_display_mode_switch.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_display_mode_switch-0000002366467641
- Why: The two mobile branches are the ones a phone and a folded foldable actually take, so the defect applies to the common case: the component starts loading the src, the UA change forces a reload, and the reference says the initial history entry is replaced rather than added - leaving accessBackward false and in-page back navigation broken. The sample already contains the correct shape in its own first branch, so the fix is to make the other two match it.
- Fix: Use the same pattern in all three branches: an empty src on the Web component, and onControllerAttached calling setCustomUserAgent first and loadUrl second.

### `HW-19-0110` - getDisplayWidth swallows its exception and returns 0, which silently forces the layout into the wide branch

- Category B, severity medium, confidence confirmed
- Features: COMMON-36
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/36_web_display_mode_switch.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_display_mode_switch-0000002366467641
- Why: A returned 0 makes foldableScreenMiddleWidth 0, so the test this.winWidth > 0 is true for any real window and the first branch - the desktop user agent and the wide layout - is selected on a folded device as well, provided isFoldable is true. The failure that produced it leaves no trace: the exception is discarded, so there is nothing in the log to connect the wrong layout to a getDefaultDisplaySync failure. Every other error path in this project at least logs.
- Fix: Log the exception with hilog.error including its code and message, and treat a zero threshold as unknown at the call site rather than as everything-is-wide.

### `HW-19-0113` - The document presents the client-side challenge as human verification for registration and login without stating that the verdict is not a security boundary

- Category E, severity medium, confidence confirmed
- Features: COMMON-37
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/37_text_order_verification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/text_order_verification-0000002347027492
- Why: A human-verification challenge exists to stop automated requests reaching a server; a challenge whose question, answer and verdict are all computed on the device cannot do that, because the request the gate protects - sending an SMS verification code - is issued by the same client that decided it passed. The document does not say this, so a reader following it for the registration flow it names may treat the UI as the control. The sample is a legitimate demonstration of the interaction pattern; what is missing is one sentence scoping it.
- Fix: Add a note to 约束与限制 stating that the challenge and its verdict are client-side only and are a user-interaction demonstration, and that a production registration or login flow must have the server issue the challenge and verify the answer before it accepts the request the gate protects.

### `HW-19-0114` - The puzzle gap and its accepted slider range are fixed constants, so the challenge has the same answer on every attempt

- Category B, severity medium, confidence confirmed
- Features: COMMON-38
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/38_slide_verification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/slide_verification-0000002348282886
- Why: A slide-puzzle challenge works because the target position is unpredictable - that is the only thing distinguishing it from a plain slider. With the gap always at the same place and the accepted band always [360, 380] out of a 0-560 range, the challenge is solved by moving the slider to the same position every time, which any script or a user repeating the flow does trivially. The document names the purpose explicitly - 通过滑动滑块完成拼图以进行人机验证 ("complete the puzzle by sliding the slider to perform human verification") - and a constant answer does not perform it.
- Fix: Randomise the gap per challenge: pick a target offset at dialog open, position the select1 image at it, and derive INTERVAL_MIN/INTERVAL_MAX from that target plus a tolerance rather than from module constants. Re-randomise on the refresh button as well as on each open.

### `HW-19-0115` - The document presents the slider puzzle as human verification for login and registration without stating that the verdict is client-side only

- Category E, severity medium, confidence confirmed
- Features: COMMON-38
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/38_slide_verification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/slide_verification-0000002348282886
- Why: A human-verification challenge is meant to keep automated requests away from a server, but here the challenge, the answer and the verdict are all computed on the device that would issue the request. The document does not scope this, so a reader building the login flow it names may treat the slider as the control. The sibling document 37_text_order_verification.md has the same gap, so the omission is a pattern in this industry rather than a one-off.
- Fix: Add a note to 环境准备 or a new 约束与限制 section stating that the slider result is a client-side interaction demonstration, and that a production login or registration flow must have the server issue the challenge and verify the answer before accepting the request it protects.

### `HW-19-0117` - Privacy mode is never turned off when the login page is left, so the whole application window stays screenshot-blocked for the rest of the session

- Category D, severity medium, confidence confirmed
- Features: COMMON-39
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/39_privacy_mode.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/privacy_mode-0000002376419961
- Why: setWindowPrivacyMode applies to the application window, not to the login page, so once the H5 login page has switched it on it stays on after the user signs in and moves to NavigationPage. Every screen of the application then silently refuses screenshots and screen recording - behaviour the user did not ask for and cannot explain, and which breaks legitimate uses such as sharing a receipt or filing a support report. The document scopes the feature to 登录页面或修改密码页面 ("the login page or the change-password page"), so leaving it engaged everywhere else contradicts its own stated scope.
- Fix: Turn privacy mode off when the protected page goes away: add `this.setPrivacy(false);` to LoginPage.aboutToDisappear, alongside the existing teardown calls, and do the same in any other page that switches it on.

### `HW-19-0118` - getLastWindow has no rejection handler, so a failure to obtain the window leaves the anti-screenshot protection silently disengaged

- Category B, severity medium, confidence confirmed
- Features: COMMON-39
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/39_privacy_mode.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/privacy_mode-0000002376419961
- Why: This function is the only thing standing between the login screen and a screenshot. If getLastWindow rejects, no privacy mode is applied, nothing is logged, and the page renders normally - so the failure of a security control is indistinguishable from success. The inner error path was considered important enough to log; the outer one is the same failure with the same consequence.
- Fix: Chain a rejection handler that logs at error level, as the reference example does, and consider surfacing the failure to the caller so the page can decide whether to proceed without protection.

### `HW-19-0120` - setAppPreferredLanguage is called without try/catch although it is documented to throw on an invalid language tag

- Category B, severity medium, confidence confirmed
- Features: COMMON-40
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/40_language_switch.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/language_switch-0000002381506029
- Why: The language tag is the one thing that can go wrong here - 890001 is raised precisely when the tag fails verification - and an uncaught throw inside an onClick handler escapes into the framework's event dispatch rather than being reported. The failure is also silent in the other direction: this.isEnglish is assigned before the call, so the checkmark moves to the row the user tapped even when the language was not actually changed, leaving the UI claiming a state the system does not have.
- Fix: Wrap each call as the reference example does, and only update this.isEnglish after it returns: try { i18n.System.setAppPreferredLanguage('zh-Hans'); this.isEnglish = false; } catch (error) { hilog.error(...); }

### `HW-19-0121` - Neither the document nor the sample offers a way back to the system language, although the reference documents the default value for exactly that

- Category E, severity medium, confidence confirmed
- Features: COMMON-40
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/40_language_switch.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/language_switch-0000002381506029
- Why: The document's stated goal is 使应用内语言设置不受系统语言设置的影响 ("make the in-app language setting independent of the system language setting"), and once a preferred language is set the application stops following the system for good. A user who taps either row - including by accident - has no way to undo it, and a user whose system language is neither Chinese nor English is forced into one of the two. The reference provides the exact remedy, 'default', and neither the document nor the sample mentions it.
- Fix: Add a third option - 跟随系统 / Follow system - that calls i18n.System.setAppPreferredLanguage('default'), and note in the document that this reverts to the system language and takes effect on the next cold start.

### `HW-19-0123` - The scenario requires every page to turn gray but the solution shown is a per-component attribute; the window-level setWindowGrayScale API is not mentioned

- Category E, severity medium, confidence confirmed
- Features: COMMON-41
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/41_page_grayscale.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/page_grayscale-0000002368995430
- Why: A reader who follows this document literally has to add a grayscale state variable and a .grayscale() call to the root container of every page, and to keep them in sync - and any content the framework draws outside those containers, such as a dialog or a sheet mounted on the window rather than inside the page subtree, stays in colour. setWindowGrayScale is a single call at window level and is documented for exactly this. Omitting it from a document whose stated scenario is the app-wide mourning grayscale leaves out the correct answer to the question the document asks.
- Fix: Present window.setWindowGrayScale(1.0) as the app-wide mechanism - called on the window obtained in EntryAbility after loadContent, guarded with canIUse('SystemCapability.Window.SessionManager') and a .catch as the reference example shows - and keep the .grayscale() attribute as the option for graying part of a page only. State the API 12 floor and error codes 401/801/1300002/1300003.

### `HW-19-0124` - The setTimeout ID is never captured and the timer is never cleared, so the callback can write to state after the page is gone

- Category B, severity medium, confidence confirmed
- Features: COMMON-41
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/41_page_grayscale.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/page_grayscale-0000002368995430
- Why: setTimeout returns the only handle to the pending timer, and discarding it makes cancellation impossible. The callback closes over this and assigns to two @State variables, so if the component is destroyed inside the three-second window - the user navigates away, or the ability is backgrounded and the page released - the pending callback still runs and writes to the state of a component that is no longer mounted. In a real application, where the grayscale flag would be set on many pages rather than on the single page of this sample, every visited page leaves such a timer behind.
- Fix: Store the ID and cancel it on teardown: `private timerId: number = -1;` then `this.timerId = setTimeout(...)` in aboutToAppear and `aboutToDisappear(): void { clearTimeout(this.timerId); }`. Update the document snippet the same way, since it is the version readers copy.

### `HW-19-0127` - Configuration is imported from the raw @ohos.app.ability.Configuration module instead of @kit.AbilityKit

- Category A, severity medium, confidence confirmed
- Features: COMMON-42
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/42_dark_mode.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/dark_mode-0000002403773657
- Why: The kit is the documented import surface; the @ohos.* module path is the underlying implementation and is not what the reference tells developers to write. Mixing the two in one file means the same ability type is reached through two different module identities, which the ArkTS compiler treats as unrelated declarations, and the project builds with useNormalizedOHMUrl: true, where module identity determines resolution. There is also no reason for it here - Configuration is exported from the kit the file already imports two symbols from.
- Fix: Delete line 21 and add Configuration to the existing kit import: `import { ConfigurationConstant, UIAbility, Configuration } from '@kit.AbilityKit';`

### `HW-19-0128` - The document teaches qualifier directories for images while its own sample loads its product images from rawfile, which is documented never to match on device state

- Category E, severity medium, confidence confirmed
- Features: COMMON-42
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/42_dark_mode.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/dark_mode-0000002403773657
- Why: The document's stated subject is configuring application resources by colour mode, and the largest visual area of its own sample is deliberately outside that system without a word of explanation. A reader who stores images in rawfile - which the sample demonstrates as normal practice - and then follows step 4 by adding a dark/media directory will find that nothing happens, and the resource guide's rule is the only place that explains why. The missing rawfile entry in the project tree compounds this: the file that supplies every product record, product_detail.json, is invisible in the documented layout.
- Fix: Add the rawfile directory to the 工程目录 tree, and state in step 4 that only base and qualifier directories participate in colour-mode matching - rawfile and resfile never do - so any image that must change between light and dark has to be a $r('app.media.*') resource rather than a $rawfile one.

### `HW-19-0129` - The avoidAreaChange subscription registered in onWindowStageCreate is never unregistered

- Category B, severity medium, confidence confirmed
- Features: COMMON-42
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/42_dark_mode.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/dark_mode-0000002403773657
- Why: on('avoidAreaChange') binds a closure to the window object for the lifetime of that window. The ability's own teardown hook is the documented place to release UI resources and it releases nothing, so the registration outlives every use of it - and if the window stage is created again in the same process the handler is registered a second time, so each avoid-area change runs the callback once per registration. This is the same defect already recorded against COMMON-08, COMMON-22 and COMMON-31 in this industry, which makes it a pattern the samples propagate rather than an isolated slip.
- Fix: Keep the handler in a field and call windowClass.off('avoidAreaChange', handler) in onWindowStageDestroy, so registration and release sit in matching lifecycle hooks.

### `HW-19-0133` - The receiver instructions stop at subscribe and never mention unsubscribe, which the module reference lists as one of its three core capabilities

- Category E, severity medium, confidence confirmed
- Features: COMMON-43
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/43_customizing_common_event.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/customizing_common_event-0000002470917738
- Why: A common-event subscriber outlives the component that created it - it is registered with a system service, not with the UI tree - so a document that teaches only registration teaches a leak. The sample itself shows the consequence: its author did write the unsubscribe call but attached it to a method the framework never invokes (HW-19-0132), and because the document has no step to check it against, nothing in the material reveals that the release path is dead. Anyone following these three steps produces a page that accumulates one live subscriber per creation.
- Fix: Add a fourth receiver step: release the subscriber in the component's aboutToDisappear by calling commonEventManager.unsubscribe(subscriber) inside try/catch, and clear the stored reference. Show it as code, as the first three steps are shown.

### `HW-19-0134` - The publish failure log passes err.code to a format string that contains no format specifier, so the error code is never printed

- Category B, severity medium, confidence confirmed
- Features: COMMON-43
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/43_customizing_common_event.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/customizing_common_event-0000002470917738
- Why: err.code is the only piece of information that distinguishes the failure modes of publish - a malformed event name, a receiver bundle that is not installed, a permission problem - and it has no identifier to land in, so it is discarded. The log line that remains says only that publication failed. This is the sole error handling on the publishing side, and the same defective call is printed in the document, so every reader copies it.
- Fix: Give the value a matching identifier: hilog.error(0xFF00, 'commonEventSender', '事件发布失败: %{public}d', err.code) - and correct the snippet in the document as well.

### `HW-19-0135` - The sender opens the payment-succeeded dialog and permanently disables the button before publish reports its result

- Category B, severity medium, confidence confirmed
- Features: COMMON-43
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/43_customizing_common_event.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/customizing_common_event-0000002470917738
- Why: publish is asynchronous and its callback is the only place the result is known, so ordering the UI updates before the call means the success state is committed unconditionally. When publication fails the user is told the payment succeeded, the receiver application never gets the event and therefore never shows the deduction message, and the button stays disabled through .enabled(!this.isPurchased) so the action cannot be retried - the page has no path back to a usable state. The failure is visible only in the log, and even there the code is dropped (HW-19-0134).
- Fix: Move the two state changes into the success path of the callback and handle the failure: open the dialog and set isPurchased only when err is falsy; on error leave isPurchased false so the button stays enabled, and surface the failure to the user rather than only to the log.

### `HW-19-0138` - The WantAgent for the long-running task is built with the bundle name passed as the ability name

- Category A, severity medium, confidence confirmed
- Features: COMMON-44
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/44_vpn_adaptation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vpn_adaptation-0000002507518793
- Why: actionType is OperationType.START_ABILITY, so this WantAgent exists to launch an ability when the user taps the long-running-task notification. Its target is a name that is not an ability, so the tap has nothing to start. The failure is invisible during development because startBackgroundRunning still succeeds - the WantAgent is accepted, and only the user-facing tap does nothing. The document making the same mistake, next to an unused wantAgentInfo block that suggests the snippet was assembled rather than copied from the project, means a reader has no correct version to compare against.
- Fix: Pass the ability name: wantAgentUtil.createWantAgent(Constants.BUNDLE_NAME, Constants.ABILITY_NAME), matching NotificationPublish.ets. In the document, replace the step-1 snippet with the project's actual startContinuousTask and drop the unused wantAgentInfo declaration.

### `HW-19-0139` - The INTERNET permission declaration lists an ability named MainAbility, which does not exist in this project

- Category E, severity medium, confidence confirmed
- Features: COMMON-44
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/44_vpn_adaptation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vpn_adaptation-0000002507518793
- Why: usedScene.abilities enumerates the abilities that use the permission and is the declaration a reviewer reads to understand why an application wants network access. Naming an ability that does not exist makes that declaration meaningless - it describes a component of some other project - and leaves the ability that genuinely needs INTERNET unlisted. It is also a copy-paste marker: MainAbility is the FA-model name, and its presence in a stage-model manifest indicates the block was carried over from an older template rather than written for this module.
- Fix: Replace "MainAbility" with "EntryAbility" in the usedScene.abilities array.

### `HW-19-0141` - The createWantAgent promise in startContinuousTask has no rejection handler and the surrounding try/catch cannot catch one

- Category B, severity medium, confidence confirmed
- Features: COMMON-44
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/44_vpn_adaptation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vpn_adaptation-0000002507518793
- Why: The catch block is written specifically for this call, as its message shows, and cannot receive the failure it names: a rejected promise from an async function does not propagate to a synchronous try/catch around the call. The rejection is therefore unhandled - the long-running task is never started, no notification id is stored, and nothing is logged. The caller then proceeds regardless: the click handler that invokes startContinuousTask immediately starts a 6-second interval that calls updateContinuousTask, which publishes against notificationId 0 forever.
- Fix: Attach .catch((err: BusinessError) => Logger.error(...)) to the createWantAgent chain, and stop the interval when the task fails to start so the page does not sit publishing updates against an id that was never issued.

### `HW-19-0143` - The rotation angle reset compares against 360, a value the 50-degree increment never produces, so the angle grows without bound

- Category B, severity medium, confidence confirmed
- Features: COMMON-45
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/45_connect_animate.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/connect_animate-0000002508806245
- Why: The reset exists to keep the angle inside one revolution, and it never runs. The interval fires every 200 ms, so a connection attempt that lasts the sample's own five-second timeout leaves the angle at 1250, and a page left connecting adds 250 degrees per second indefinitely - the value passed to .rotate({ angle: this.angle }) grows for as long as the animation is on screen. Every intermediate animateTo also interpolates from the current large value, so the arithmetic the animation performs drifts further from the intended 0-360 domain the longer it runs. The same defective condition is printed in the document, so it is what readers copy.
- Fix: Use the modulus of the increment rather than an equality test: this.angle = (this.angle + 50) % 360; - which resets correctly for any step size.

### `HW-19-0144` - NetworkManager.connecting registers the socket listeners but is never called, while closeNetwork unregisters them

- Category B, severity medium, confidence confirmed
- Features: COMMON-45
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/45_connect_animate.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/connect_animate-0000002508806245
- Why: connecting is the only place the socket's message handler is installed, so as shipped the sample never receives anything from the server: the DataView decode loop on lines 63-69, which is the only code that reads socket data, cannot run. The unbalanced off calls in closeNetwork are the visible symptom - a teardown that releases subscriptions no path ever created - and they are what makes the omission look intentional to a reader. The document does not mention connecting either; its step 1 shows only connect, so nothing in the material reveals that a listener registration step is missing.
- Fix: Call this.networkManager.connecting() alongside the connect call - either at the top of NetworkManager.connect before this.tcpSocket.connect, or from openConnect in the page - so that on and off are registered and released in matching places.

### `HW-19-0145` - The page has no aboutToDisappear, so two intervals and two timeouts survive its destruction

- Category B, severity medium, confidence confirmed
- Features: COMMON-45
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/45_connect_animate.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/connect_animate-0000002508806245
- Why: The only path that stops the animation interval is a state transition inside the page; destroying the page is not such a transition. If the user leaves while a connection is in flight - which lasts at least the sample's own deliberate five-second delay - the 200 ms animation interval and the 800 ms text interval keep firing against a destroyed component, each callback closing over this and writing to @Local fields, and the two pending NetworkManager timeouts still resolve and set connected on an object nothing observes any more. Nothing in the page ever releases them, so they persist for the process lifetime.
- Fix: Add aboutToDisappear() to the page: clearInterval(this.animateTimer); clearInterval(this.valueChangeTimer); and close the socket through this.networkManager.closeNetwork(), which also clears the two pending timeouts if a clearTimeout pair is added to it.

### `HW-19-0146` - GET_WIFI_INFO is requested although no Wi-Fi API is used, is absent from the document's permission list, and its usedScene names an ability from a different bundle

- Category D, severity medium, confidence confirmed
- Features: COMMON-45
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/45_connect_animate.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/connect_animate-0000002508806245
- Why: A permission that no code exercises is a permission the application should not hold; it widens what the package can do beyond what it needs, and it is what a reviewer sees when deciding whether the request is proportionate. The evidence that it is not deliberate is in the declaration itself: the ability it names belongs to com.samples.socket, so the whole block was copied from another project. The document listing only two of the three permissions compounds this - the reader is told the sample needs INTERNET and GET_NETWORK_INFO, and the third is granted silently.
- Fix: Remove the GET_WIFI_INFO block from requestPermissions. If a later version does read Wi-Fi state, declare it with this application's own ability name and add it to the document's 权限说明.

### `HW-19-0148` - The throttle wrapper is constructed inside build(), so its guard state is discarded whenever the attribute is re-evaluated

- Category C, severity medium, confidence probable
- Features: COMMON-45
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/45_connect_animate.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/connect_animate-0000002508806245
- Why: The guard variable lives in the closure, not in the component, so it survives only as long as that particular closure is the one attached to the click handler. Any re-evaluation of the attribute installs a fresh wrapper whose window has not started, and the five-second protection the comment describes is lost for the next click. Marked probable because the exact set of state changes that re-evaluate this attribute depends on the framework's update granularity and was not measured on a device; what is certain from the code is that the throttle state is created during build rather than held by the component, which is not how the referenced FAQ solves the problem.
- Fix: Create the throttled handler once and store it as a component field - for example `private throttledConnect = throttle((e: ClickEvent) => { ... }, 5000);` - and reference this.throttledConnect from .onClick, so the guard state belongs to the component rather than to a build-time expression.

### `HW-19-0149` - scrollTo is called without error handling although it is documented to throw, and without the duration parameter that would animate the scroll

- Category A, severity medium, confidence confirmed
- Features: COMMON-46
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/46_web_scroll_to_top.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_scroll_to_top-0000002538476079
- Why: 17100001 is raised when the controller is not yet bound to a Web component, which is exactly the state this button can be in: the controller is created in the parent and handed to the child as a constructor argument, so it exists before the Web component has attached it, and the optional-chaining ?. guards only against null - it does not guard against the bound-controller check failing. An uncaught throw inside onClick escapes into event dispatch. Separately, omitting duration disables the animation, so a page scrolled hundreds of vp jumps instantly to the top; the document's subject is a scroll-to-top affordance and the reference provides the parameter that makes it a scroll rather than a jump.
- Fix: Wrap the call and give it a duration: try { this.webviewController?.scrollTo(0, 0, 300); this.isShow = false; } catch (error) { hilog.error(...); } - and update the document's step 5 snippet the same way.

### `HW-19-0150` - The floating button's initial position is two hard-coded vp values while every other position it takes is computed from the measured container

- Category C, severity medium, confidence confirmed
- Features: COMMON-46
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/46_web_scroll_to_top.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_scroll_to_top-0000002538476079
- Why: The component derives every dragged position from the container it was given but its starting position from two literals that describe one particular device. In a window narrower than 344 vp or shorter than 671 vp - a resizable window, a folded screen, a landscape orientation - the button is laid out beyond the container edge on first render. Because the clamp is attached to onActionEnd, it can only correct a position the user has already dragged, and a button rendered outside the visible area cannot receive the pan that would trigger it. The enclosing Stack is declared Alignment.BottomEnd, which suggests the intended placement, but .position() is absolute and overrides that alignment entirely.
- Fix: Derive the initial offsets from the measured container, the way the clamp does - for example set them in the parent's onAreaChange or in aboutToAppear as containerWidth - buttonWidth - buttonPadding and containerHeight - buttonHeight - buttonPadding - so the first placement and every later placement use the same source.

### `HW-19-0151` - The document has no 权限说明 section although the sample declares INTERNET and its page cannot render without it

- Category E, severity medium, confidence confirmed
- Features: COMMON-46
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/46_web_scroll_to_top.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_scroll_to_top-0000002538476079
- Why: The scenario is a scroll-to-top button, and the button only appears once the page has scrolled past 300 vp - which requires the remote images to load and give the page height. Without INTERNET the demo shows a short page and the feature never becomes visible, so a reader who recreates the project from the documented steps and omits the permission sees nothing and has no way to tell why. The permission list is also the part of a sample a reviewer reads first when deciding what a package needs, and this document does not disclose that the sample requests network access at all.
- Fix: Add a 权限说明 section listing ohos.permission.INTERNET with its purpose - loading the remote images referenced by the demo page - as the neighbouring documents in this industry do.

### `HW-19-0155` - Both converters exclude U_BUFFER_OVERFLOW_ERROR from the failure test and return truncated output as success

- Category B, severity medium, confidence confirmed
- Features: COMMON-47
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/47_format_conversion.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/format_conversion-0000002516884120
- Why: The one status that means the caller did not get the data it asked for is the one status treated as success. The caller has no way to detect it: the return value is 0, the out-parameter length is a plausible number, and the buffer contains a valid prefix of the answer. In Utf8ToGb2312 the sizing is utf8Len * 2, which is not a safe bound - a three-byte UTF-8 sequence can map to a two-byte GB2312 character, so the ratio holds for Chinese text but a string of characters that fall outside GB2312 produces substitution sequences that can exceed it, and the overflow is then silently truncated. Where the log is the only record of an error, this path does not even log.
- Fix: Treat the overflow as the failure it is: test U_FAILURE(status) alone, or handle U_BUFFER_OVERFLOW_ERROR explicitly by reallocating to the size ICU reports and converting again. Correct the snippet in the document, which is the version readers copy.

### `HW-19-0157` - Utf82Gb2312 leaks its 1024-byte input buffer on every call, truncates longer input silently, and returns a buffer whose last byte is never written

- Category B, severity medium, confidence confirmed
- Features: COMMON-47
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/47_format_conversion.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/format_conversion-0000002516884120
- Why: One 1024-byte block is lost per button press, and the button is the sample's only action, so the leak grows for as long as the page is used. The truncation is the more damaging of the two for correctness: a long input is cut at an arbitrary byte, which for Chinese text falls in the middle of a multi-byte UTF-8 sequence, and the converter is then handed malformed input with no error path back to the user. The uninitialised trailing byte matters because the extra byte was clearly allocated to hold a terminator and nothing writes one - the consumer sees a buffer one byte longer than the data with an indeterminate value at the end.
- Fix: Free the input buffer on every path, or use a std::string sized from a first napi_get_value_string_utf8 call with a null buffer; check that call's status and the reported length instead of assuming 1024 is enough; and either write the terminator into the returned buffer or allocate exactly gb2312Len bytes.

### `HW-19-0161` - The scan results are read synchronously right after startScan and the scan-complete listener that exists to prevent this only writes a log line

- Category B, severity medium, confidence confirmed
- Features: COMMON-48
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/48_wifi_connect.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/wifi_connect-0000002518583904
- Why: startScan is asynchronous - it returns as soon as the scan is requested, not when results are available - so getScanInfoList immediately afterwards returns whatever the previous system scan left behind. The listener that would tell the code when fresh results exist is registered and its callback is a log statement, so the one mechanism the reference prescribes is present in form and absent in effect. Because isStartScan latches, turning the switch off and on again re-reads the same stale list without requesting a new scan, and the user has no way to refresh.
- Fix: Move the getScanResult call into the wifiScanStateChange callback and update this.wifiList when the callback reports success, rather than reading the list synchronously after startScan. Reset isStartScan when the switch is turned off so a later toggle requests a fresh scan.

### `HW-19-0163` - The shutdown handshake uses an unrestricted common event, so any application on the device can make this one remove its tray icon and terminate

- Category D, severity medium, confidence confirmed
- Features: COMMON-49
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/49_pc_status_bar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pc_status_bar-0000002551100435
- Why: The event name is a plain string carried in the shipped binary, and the handler that reacts to it removes the tray icon and terminates the application. With no publisherBundleName on the subscriber, any process that publishes that string is accepted as the background ability; with no bundleName or subscriberPermissions on the publisher, any process can also observe the application shutting down. For a sample whose stated purpose is keeping a PC application alive in the background, a remotely triggerable shutdown defeats the feature - and the callback does not inspect the event data at all, so there is nothing else that could distinguish a genuine notification from a forged one.
- Fix: Restrict both ends: set publisherBundleName to this application's own bundle in the CommonEventSubscribeInfo, and set bundleName on the CommonEventPublishData so the event is delivered only within the application. Since both abilities belong to one bundle, an in-process channel such as emitter avoids the exposure entirely.

### `HW-19-0165` - loadStatusBar swallows an addToStatusBar failure and resolves anyway, so the whole start-up chain proceeds as though the tray icon exists

- Category B, severity medium, confidence confirmed
- Features: COMMON-49
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/49_pc_status_bar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pc_status_bar-0000002551100435
- Why: addToStatusBar is the one operation that has to succeed for any of this to mean anything - it is what puts the icon in the tray, and the background ability launched two steps later is bound to that icon through NEW_PROCESS_ATTACH_TO_STATUS_BAR_ITEM. Swallowing its failure means the click listener is still registered, the background process is still spawned, and the one-second interval still starts, all against a tray item that does not exist, with a single error line in the log as the only trace. The discarded Promise.resolve() shows the intent was to signal success explicitly; a rethrow in the catch is what would make the caller's .then mean what it appears to mean.
- Fix: Rethrow from the catch, or reject, so loadStatusBar fails when addToStatusBar fails, and give the caller a .catch that skips the rest of the chain. Delete the discarded Promise.resolve() statement.

### `HW-19-0166` - Start-up is sequenced by nested fixed delays and ends in a one-second log interval that is never cleared

- Category B, severity medium, confidence confirmed
- Features: COMMON-49
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/49_pc_status_bar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pc_status_bar-0000002551100435
- Why: The 500 ms and 200 ms delays sit inside a then() that has already waited for loadStatusBar to complete, so they are not sequencing anything the promise did not already sequence - they are guesses about how long a system operation takes, and on a slower machine the guess is wrong in the direction that breaks the feature. The interval is the more concrete defect: it logs once per second for the entire life of a process whose whole purpose is to run indefinitely in the background, with no id kept and no way to stop it. And holdStatusBar rejecting with no reason and no handler means a failure to launch the background ability - the step that actually keeps the process alive - produces an unhandled rejection with no consequence beyond one log line inside the callback.
- Fix: Drop the two nested setTimeout wrappers and chain the operations directly on the promises. Give holdStatusBar a rejection value and attach a .catch. If a heartbeat log is wanted, store the interval id and clear it in onDestroy; otherwise remove it.

### `HW-19-0167` - Step 1 points at rawfile/resources while the icons live in resources/rawfile, and the right-click menu step 2 describes is built and then commented out

- Category E, severity medium, confidence confirmed
- Features: COMMON-49
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/49_pc_status_bar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pc_status_bar-0000002551100435
- Why: Both defects mislead a reader following the steps in order. A rawfile/resources directory created as instructed does not match getRawFileContentSync('white.png'), which resolves from the rawfile root, so the first step produces a layout the first API call cannot read. The menu code is worse in a document presented as a walkthrough: twenty lines appear as step 2 content, they compile, they are dead, and the reader has no way to know the tray in the screenshot has no right-click menu. The document 常见FAQ even discusses closing the tray, a task the right-click menu would ordinarily serve.
- Fix: Correct step 1 to resources/rawfile. Either enable the right-click menu by uncommenting statusBarGroupMenu in the StatusBarItem, or delete the menu construction from both the sample and the document and stop describing it as a step.

### `HW-19-0170` - The distributed data object status listener is registered on every continuation and never unregistered

- Category B, severity medium, confidence confirmed
- Features: COMMON-50
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/50_navigation_continue.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/navigation_continue-0000002522501268
- Why: The field comment on line 27 explains why the object is held at all - 数据恢复时，分布式数据对象状态监听有延迟，需要在此处定义该对象，避免对象销毁导致未接收到对端迁移数据 ("during data restoration the distributed data object status listening is delayed, so the object must be declared here to prevent its destruction causing the migrated data from the peer not to be received") - which is exactly right, and it makes the missing release the other half of the same concern. Each repeated continuation leaves an activated session and a live callback that still writes pagePath, pageParams and continueMsg into AppStorage, so a late 'restored' event from an earlier session can overwrite the values of the current one, and the page then navigates to the previous migration target.
- Fix: Call this.distributedObject.off('status') and clear the session before creating a replacement in restoreData, and again in onDestroy, so each activated object is released when it is superseded.

### `HW-19-0171` - The migration type written into the want is produced by a ternary that maps each value to itself and disagrees with the branch immediately below it when the value is unset

- Category B, severity medium, confidence probable
- Features: COMMON-50
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/50_navigation_continue.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/navigation_continue-0000002522501268
- Why: The value announced to the peer and the branch actually taken are computed from the same variable by two different tests, so they agree only as long as the variable is a defined enum member. The key is normally created by the Index page - @StorageProp('dataMigrateType') with a local default (Index.ets:7) - so in ordinary use the value is 0 or 1 and the ternary is merely redundant; the divergence needs onContinue to run before that page has been built, which is why this is recorded as probable rather than confirmed. The redundancy itself is certain from the enum values, and it is what hides the divergence: a reader sees an inversion where none exists and has no reason to check the two tests against each other.
- Fix: Assign the value directly and test it once: `const dataMigrateType = AppStorage.get<number>('dataMigrateType') ?? DATA_MIGRATE_TYPE.DISTRIBUTED_OBJECT_TYPE; wantParam.dataMigrateType = dataMigrateType; if (dataMigrateType === DATA_MIGRATE_TYPE.DISTRIBUTED_OBJECT_TYPE) { ... }` so the announced type and the branch taken cannot disagree.

### `HW-19-0174` - getFilePathExist evicts the Preferences instance from the cache on every read, leaving the cached field the class writes through in the state the reference warns against

- Category B, severity medium, confidence confirmed
- Features: COMMON-51
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/51_file_persistence_permission_acquisition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/file_persistence_permission_acquisition-0000002412498917
- Why: The eviction and the write target the same store name through two different instance handles, which is precisely the situation the reference describes as causing data inconsistency. init caches one handle for the life of the singleton and never refreshes it, while every read discards the cache entry underneath it, so the path list written through the stale handle and the path list read through the freshly opened one are two views of the same file with no guarantee of agreement. The eviction also defeats the purpose of the cache on a path that runs on every import.
- Fix: Read through the same instance the class writes through - drop removePreferencesFromCacheSync and the second getPreferences call, and use this.dataPreferences.getSync('pathArray', []) - or, if the cache must be invalidated, reassign this.dataPreferences from the newly opened instance and null out the old one.

### `HW-19-0175` - Once any path has been stored the picker is never shown again, and each import clears the stored list first, so the batch history the scenario describes can never grow

- Category B, severity medium, confidence confirmed
- Features: COMMON-51
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/51_file_persistence_permission_acquisition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/file_persistence_permission_acquisition-0000002412498917
- Why: The two behaviours combine into a one-shot import. The first use opens the picker and stores the selection; from then on the stored list is non-empty, so every later use takes the else branch, activates the same files and returns them - the picker cannot be reached again and no second batch can ever be chosen. And even if it could, clearSync immediately before putSync discards the previous batch, so the stored list holds at most one selection. A history that can neither be added to nor replaced is not the history the scenario describes, and there is no other code path that clears the preference to recover.
- Fix: Separate the two actions: keep an explicit import action that always opens the picker, and merge its result into the stored list rather than clearing first - read the existing array, append the newly selected uris after deduplicating, then putSync the union. Activation of the stored list stays as it is, on application start.

### `HW-19-0178` - The document is an eleven-line redirect with none of the sections every other document in this industry carries, yet it is indexed as one of the industry scenarios

- Category E, severity medium, confidence confirmed
- Features: COMMON-52
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/52_practice-common-app-architecture-v1_35.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-architecture-v1_35-0000002263801938
- Why: The industry listing is how a reader chooses what to read, and this entry is indistinguishable in that listing from the fifty-one documents that contain scenarios, code and samples. Its title, 行业常见问题 ("industry FAQ"), promises the one document in the set that would answer questions the scenario documents raise, and the page delivers a single link. A reader spends a navigation step to discover there is nothing here, and nothing in the listing warns them. The destination is also outside the local documentation mirror - documentation/ contains only harmonyos-guides and harmonyos-references, with no harmonyos-faqs tree - so the content the sentence promises cannot be reached from this repository at all.
- Fix: Either fold the redirect into the industry README as an external link rather than shipping it as a document, or mark the entry in the listing as a redirect so a reader can see before opening it that the content lives elsewhere.

### `HW-19-0179` - All nineteen industries ship a byte-identical 行业常见问题 stub pointing at the same generic phone FAQ, so no industry-specific FAQ is reachable from any of them

- Category F, severity medium, confidence confirmed
- Features: COMMON-52
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/52_practice-common-app-architecture-v1_35.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-architecture-v1_35-0000002263801938
- Why: Nineteen distinct source pages, each titled for its own industry, redirect to one undifferentiated destination. Whatever industry-specific FAQ each page used to hold, a reader arriving from the automotive industry and a reader arriving from this one are sent to the same generic phone FAQ, so the industry dimension the titles promise is lost at the redirect. The redirect is also the only content each of the nineteen pages has, which means the loss is total rather than partial - there is no remaining industry-specific text to fall back on. That the nineteen bodies are byte-identical, rather than each carrying a link into its own section of the FAQ, is what identifies this as a templating shortcut rather than nineteen independent editorial decisions.
- Fix: Give each industry redirect a destination that lands on that industry's FAQ content - a per-industry page or at minimum an anchor within faq-phone - so the title and the link agree. If the industry-specific FAQs no longer exist as separate content, say so in the sentence rather than implying the reader will find them at the other end.

### `HW-19-0180` - 1 sample project swallows errors in catch blocks with an empty body

- Category B, severity medium, confidence confirmed
- Features: COMMON-47
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/47_format_conversion.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/format_conversion-0000002516884120
- Why: A catch block with an empty body discards the error object and lets execution continue as if the operation had succeeded. The failure becomes invisible: no log, no user feedback, and no way to diagnose it from a released build. In several of these cases the guarded call is the feature the sample exists to demonstrate.
- Fix: Log the error with hilog including the BusinessError code and message, and surface a user-visible result where the operation was user initiated. Never leave the body empty.

### `HW-19-0001` - Two sentences in the package-type sections are grammatically incomplete, one of them dropping the noun that identifies what the split is per

- Category E, severity low, confidence confirmed
- Features: COMMON-02
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/02_practice-common-app-layered-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-layered-v1-0000001916033058
- Why: Documentation must state the packaging rule unambiguously. Section 模块分包/共包建议 ("Module split/shared packaging recommendations") is precisely about per-device differences, so a sentence that omits the word 设备 ("device") leaves the reader unable to tell what dimension the split is along.
- Fix: Correct line 32 to "实践中，单Entry HAP包多HAR包比较常见" and line 49 to "Entry分包：针对不同设备单独规划一个Entry类型HAP包。" ("plan a separate entry-type HAP for each device type"), and remove the stray space before 应用.

### `HW-19-0008` - getPaths() restores both contexts to a hardcoded EL2 area instead of the area they had before, contradicting the document's own instruction

- Category B, severity low, confidence confirmed
- Features: COMMON-07
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/07_cache_clean.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cache_clean-0000002231963970
- Why: The document requires restoring the original area precisely because the area is process-wide state shared with the rest of the application. Writing a fixed EL2 back is only correct for an application that was already in EL2; for one that deliberately runs in EL1 the cache-clear operation silently moves every subsequent Context path lookup to EL2, which is the failure the document's own comment warns about.
- Fix: Capture both areas before switching and restore the captured values: const prevModuleArea = moduleContext.area; const prevCtxArea = this.context.area; ... moduleContext.area = prevModuleArea; this.context.area = prevCtxArea;

### `HW-19-0014` - accessStep() and backward() are called inside onBackPressed without the try/catch the official reference example uses

- Category B, severity low, confidence confirmed
- Features: COMMON-09
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/09_backpage_by_gesture.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/backpage_by_gesture-0000002237371200
- Why: onBackPressed must return a boolean telling the framework whether the back event was consumed. If accessStep throws - for example when the callback fires before the controller has been associated with the Web component - the exception escapes the callback, no value is returned, and the back gesture is left in an undefined state instead of falling through to the framework's own back handling.
- Fix: Wrap the two calls and return false on failure so the framework performs the default back navigation:
```
.onBackPressed(() => {
  try {
    if (this.controller.accessStep(-1)) {
      this.controller.backward();
      return true;
    }
  } catch (error) {
    hilog.error(0x0000, 'WebPage', `accessStep failed. Code: ${(error as BusinessError).code}, message: ${(error as BusinessError).message}`);
  }
  return false;
})
```

### `HW-19-0015` - The 工程目录 tree omits the entrybackupability directory and the route_map profile that the sample actually ships and declares

- Category E, severity low, confidence confirmed
- Features: COMMON-09
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/09_backpage_by_gesture.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/backpage_by_gesture-0000002237371200
- Why: The 工程目录 section is what a reader uses to reproduce the project layout. Omitting route_map.json is the more consequential of the two: the Index page navigates with this.pathStack.pushPath({ name: 'WebPage' }) (Index.ets:59) and that name resolves only through the routerMap profile, so a reader who recreates the tree exactly as documented gets a navigation that silently fails.
- Fix: Add the two missing entries to the tree, for example an entrybackupability/EntryBackupAbility.ets line and a resources/base/profile/route_map.json line with a comment noting it is the Navigation route table used by pushPath({ name: 'WebPage' }).

### `HW-19-0017` - Three hilog calls in SwiperGuide pass a value as the log tag or pass arguments to a format string that has no format identifiers, so the logged values never appear

- Category B, severity low, confidence confirmed
- Features: COMMON-10
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/10_custom_swiper_guide_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_swiper_guide_page-0000002282653913
- Why: The number and type of arguments must map to identifiers in the format string, so the supplied index and offset are dropped and the log lines print only the literal prefixes - exactly the values a developer would be reading these logs for. Using a per-frame varying value as the tag also defeats tag-based log filtering, and tags are truncated at 31 bytes. The same file gets it right elsewhere: lines 98-101 concatenate with '+' instead, and EntryAbility.ets:26 uses the documented '%{public}s' form.
- Fix: Use a constant tag and matching identifiers, for example `hilog.info(0x0000, 'SwiperGuide', 'index: %{public}d, current offset: %{public}f', index, extraInfo.currentOffset);`

### `HW-19-0018` - windowWidth is captured once in aboutToAppear and never refreshed, so the swipe-driven opacity is computed against a stale width after a window size change

- Category B, severity low, confidence probable
- Features: COMMON-10
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/10_custom_swiper_guide_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_swiper_guide_page-0000002282653913
- Why: The gradient maps the drag offset onto 0..1 by dividing by the page width. If the window is resized after the page appears - rotation, a foldable unfolding, or a resizable 2in1 window, all of which are in the declared deviceTypes ['phone','tablet','2in1'] - the divisor no longer matches the actual swipe distance, so penetrability leaves the 0..1 range and the button opacity and hit-test state become wrong.
- Fix: Recompute the width when it changes: keep it in AppStorage and refresh it from a window 'windowSizeChange' subscription in EntryAbility, reading it in the page with @StorageProp instead of a one-off aboutToAppear copy. Alternatively derive the ratio from the component's own measured width via onAreaChange.

### `HW-19-0021` - AppColors.ets imports the theme types from '@ohos.arkui.theme' instead of the documented '@kit.ArkUI', and disagrees with the other file in the same project

- Category A, severity low, confidence confirmed
- Features: COMMON-12
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/12_custom_theme_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_theme_demo-0000002290124205
- Why: The Kit import form is the one the reference prescribes and the one the rest of the project uses. Mixing the two import styles for the same symbol in one project is a maintenance hazard, and the @ohos.* specifier bypasses the Kit barrel that the documentation and the useNormalizedOHMUrl build option assume.
- Fix: Change the import in AppColors.ets to `import { CustomColors, CustomTheme } from '@kit.ArkUI';`, matching the reference and DisplayPage.ets.

### `HW-19-0029` - The authentication failure log interpolates a Resource object into a template string, so it prints [object Object] instead of the message

- Category B, severity low, confidence confirmed
- Features: COMMON-13
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/13_user_authentication.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/user_authentication-0000002290451165
- Why: This catch block is the only report of a failure to launch the authentication dialog - the case where getUserAuthInstance or start() throws, for example when ohos.permission.ACCESS_BIOMETRIC is denied (error 201) or no credential is enrolled. Emitting '[object Object]' in place of the message makes that failure unreadable in the log.
- Fix: Resolve the resource first, or drop it: `Logger.error('UserAuth', `startUserAuth failed: ${(e as BusinessError).code} ${(e as BusinessError).message}`);` Use Logger.error rather than Logger.info for a failure path.

### `HW-19-0033` - The picker failure path reports at info level and calls hilog.isLoggable as a statement whose return value is discarded

- Category B, severity low, confidence confirmed
- Features: COMMON-14
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/14_upload_picture.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/upload_picture-0000002256189702
- Why: This is the only report of a failed photo selection. Logging it at INFO buries it among the routine 'onShowFileSelector invoked' lines from the same tag, and the dead isLoggable call suggests a level guard that was never actually applied. The mismatched %{public}s for a numeric code is the same format/argument mismatch the hilog reference forbids: "The number and type of parameters must map to the identifier in the format string."
- Fix: Drop the isLoggable statement (or use it as a condition) and log at error level with matching identifiers: `hilog.error(0xFF00, 'UploadPicture', 'photoViewPicker.select failed, code %{public}d, message %{public}s', err.code, err.message);`

### `HW-19-0036` - The photo picker rejection is logged at info level while the two sibling picker rejections in the same switch use error level

- Category B, severity low, confidence confirmed
- Features: COMMON-15
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/15_webview_picker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/webview_picker-0000002257162014
- Why: The three branches are the same kind of failure and should be filterable together. Logging one of them at INFO hides it among the routine trace lines this same file emits at INFO (lines 73, 76, 90), so a failing photo selection is the hardest of the three to diagnose - and it is the most common path in this feature.
- Fix: Use Logger.error for the photo picker rejection, matching the other two branches and the official example.

### `HW-19-0037` - The 工程目录 tree names the constants file Constants.ets while the shipped project actually contains Contants.ets

- Category E, severity low, confidence confirmed
- Features: COMMON-16
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/16_tab_bar_blur.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/tab_bar_blur-0000002257193008
- Why: The 工程目录 section is the reference for recreating the layout. A reader who creates common/Constants.ets as documented and then copies the sample's imports gets an unresolved module, because those imports name Contants. Either the file or the document is wrong; today they disagree.
- Fix: Rename the file to Constants.ets in the sample and update the two imports, so the code matches the documented tree.

### `HW-19-0044` - PersistentStorage.persistProp('isRemembered') is called inside the loadContent callback rather than before the page is created

- Category B, severity low, confidence probable
- Features: COMMON-17
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/17_silent_login.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/silent_login-0000002292499361
- Why: The callback runs after pages/MainPage has been created. In the shipped code this happens to be harmless because MainPage reads isRemembered only inside a click handler (MainPage.ets:147) and LoginPage's @StorageLink (LoginPage.ets:34) is constructed later still, so nothing touches the key first. The ordering is nevertheless the one the guide names as incorrect, and it breaks the moment anyone binds isRemembered at page construction - which is precisely what LoginPage does once it is made the start page.
- Fix: Call PersistentStorage.persistProp('isRemembered', false) at the top of onWindowStageCreate (or in onCreate), before windowStage.loadContent, so the persisted value reaches AppStorage before any component can bind the key.

### `HW-19-0055` - PageBody deep-imports BTState from inside the common HAR instead of through its Index.ets barrel, because that constant is not exported

- Category E, severity low, confidence confirmed
- Features: COMMON-19
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/19_search_and_connect_ble.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/search_and_connect_ble-0000002293947117
- Why: Index.ets is the HAR's declared entry point, and the layered-architecture practice in this same industry describes the common-capability tier as a package consumed through its exported interface (02_practice-common-app-layered-v1.md). Reaching past the barrel into the package's internal directory layout couples the product module to the HAR's file structure, so any internal reorganisation of commons/common breaks a consumer that the barrel was supposed to insulate. It is also inconsistent within one file - BTUtil, whose return value is compared against BTState, is imported the proper way.
- Fix: Add `export { BTState } from './src/main/ets/constants/BTState';` to commons/common/Index.ets and change the import in PageBody.ets to `import { BTState } from '@ohos/common';`.

### `HW-19-0067` - The failed handler calls request.agent.show() and discards the result, with a rejection message that describes a task removal

- Category B, severity low, confidence confirmed
- Features: COMMON-21
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/21_agent_download_control.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/agent_download_control-0000002262990970
- Why: This is the only place the sample could report why a download failed. TaskInfo is what carries that answer (faults, progress, state), and it is thrown away, so a failed download produces a log line containing only the raw Progress object. The copy-pasted rejection message then misattributes any failure of the show call to a removal, which sends anyone reading the log to the wrong call.
- Fix: Consume the result and correct the message: `request.agent.show(this.downloadTask?.tid).then((info: request.agent.TaskInfo) => { hilog.error(DOMAIN_ID, TAG, `/Request ---failed, faults: ${info.faults}, state: ${info.progress.state}`); }).catch((err: BusinessError) => { hilog.error(DOMAIN_ID, TAG, `/Request ---Failed to show the download task, Code: ${err.code}, message: ${err.message}`); });`

### `HW-19-0076` - Two hilog calls use the truncated format identifier '%{public}' with no type letter, and the Web component repeats imageAccess and fileAccess twice

- Category B, severity low, confidence confirmed
- Features: COMMON-23
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/23_web_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_scan-0000002284631808
- Why: The two log lines are the only record of a scan failure or a denied camera permission, and with a malformed identifier the value they were meant to carry is not printed - the developer sees a bare marker. The duplicated attribute calls are harmless at runtime but signal a copy-paste that survived review, and they obscure which security-relevant settings the component actually has: fileAccess(true) is a deliberate choice here (the page needs resourceDir through setPathAllowingUniversalAccess) and deserves to appear exactly once.
- Fix: Use complete identifiers - `hilog.info(DOMAIN, 'Scanner', '%{public}s', scanResult);` - and delete the duplicated .imageAccess(true) / .fileAccess(true) calls.

### `HW-19-0078` - The date label is assembled with hardcoded Chinese characters while the weekday beside it comes from string resources

- Category B, severity low, confidence confirmed
- Features: COMMON-24
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/24_pull_back.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pull_back-0000002320205289
- Why: Half of one label is localisable and half is not, so any locale the application later adds renders a translated weekday next to an untranslated Chinese date - and the date format itself (month-first, character separators) is not adaptable at all. The project already has the mechanism in place for the weekday, which is what makes the inconsistency a defect rather than a scope decision.
- Fix: Externalise the date format too, for example a `date_format` string resource with placeholders used through resourceManager, or build the label with Intl.DateTimeFormat so both the order and the separators follow the device locale.

### `HW-19-0083` - The ad-load timeout timer is cleared only when the ad loads successfully, and stays armed if the splash page is left for any other reason

- Category B, severity low, confidence confirmed
- Features: COMMON-26
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/26_splash_page_ad_access.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/splash_page_ad_access-0000002322822425
- Why: Any exit from the splash page that is not a successful ad load leaves a live timer holding this.uiContext. If the user leaves within the two-second window - or if the ad request fails and the page is dismissed by something else first - the timer still fires and issues a replaceUrl on a page that is no longer displayed. onAdLoadFailure also does not clear it, so the failure path depends on the timer firing, which conflates 'no ad' with 'timed out' in the log.
- Fix: Clear the timer in aboutToDisappear and in onAdLoadFailure: `clearTimeout(this.timeOutIndex);`, and have onAdLoadFailure route home explicitly rather than waiting for the timeout.

### `HW-19-0086` - MainPage.ets declares its entry component as struct LoginPage, so two different components in the project carry the same name

- Category E, severity low, confidence confirmed
- Features: COMMON-27
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/27_display_after_login.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/display_after_login-0000002293948070
- Why: The home page - the one that shows the blurred feed and provides isLogin to the whole tree - is called LoginPage, the same name as the actual login destination and the same name as the route it pushes with pushDestinationByName('LoginPage', ...). Reading the code, there is no way to tell from a struct name which of the two components is meant, and a search for the login page returns the home page first.
- Fix: Rename the entry component in MainPage.ets to MainPage, matching its file name and the document's 工程目录 description.

### `HW-19-0091` - A one-shot delayed action is implemented with setInterval that clears itself on the first tick, is never cleared on page disappear, and repeats what the branch above it already did

- Category B, severity low, confidence confirmed
- Features: COMMON-29
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/29_user_agreement_and_privacy_policy.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/user_agreement_and_privacy_policy-0000002331953689
- Why: Three separate problems in six lines: setInterval is the wrong primitive for a single delayed action (setTimeout expresses it and needs no self-clearing); the timer outlives the component, so a page dismissed inside 500 ms still gets isShow and the persisted flag written; and the block is redundant, since on the only launch where its guard passes the else branch has already set isShow. A reader copying this pattern inherits a timer leak for no behaviour.
- Fix: Delete the block - the if/else already covers it. If a delayed re-prompt is genuinely wanted, use setTimeout, store the handle, and clear it in aboutToDisappear.

### `HW-19-0099` - windowStage.getMainWindow() is used without a rejection handler, so a failure to obtain the main window is silently lost

- Category B, severity low, confidence confirmed
- Features: COMMON-32
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/32_app_multiplier.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_multiplier-0000002312850858
- Why: If getMainWindow rejects, curScreenSizeString is never written and no resize listener is installed, so the Navigation stays in whatever mode its default produces and never adapts when the device is folded or unfolded - the entire feature silently does nothing, with no log line to explain it. An unhandled promise rejection is also a runtime warning in its own right.
- Fix: Add a rejection handler: `.catch((err: BusinessError) => { hilog.error(DOMAIN, 'testTag', `Failed to obtain the main window. Code: ${err.code}, message: ${err.message}`); });`

### `HW-19-0105` - The sample declares ohos.permission.INTERNET although the page it loads is a local rawfile with only packaged assets

- Category D, severity low, confidence confirmed
- Features: COMMON-34
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/34_app_pull_up.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_pull_up-0000002353615821
- Why: The official permission principles are explicit: "Request only the least required permissions for your application. Do not apply for unnecessary or obsolete permissions. Excess permission requests can cause users to worry about application security and degrade user experience" (documentation/harmonyos-guides/04_system/app-permission-mgmt-overview.md:25). Neither the deep-link launch nor the AppGallery fallback needs network access - both go through startAbility - so a reader copying this solution inherits a network permission the feature never uses. The same defect appears in the BackPageByGestures sample of this industry (HW-19-0013).
- Fix: Remove the requestPermissions block, and change the 权限说明 section to state that no permission is needed for the local-page case, with ohos.permission.INTERNET required only if the hosting page is loaded from the network.

### `HW-19-0107` - Both ForEach key generators declare their parameter as number while the arrays they iterate hold ShowData objects

- Category B, severity low, confidence confirmed
- Features: COMMON-35
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/35_title_and_content_linkage.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/title_and_content_linkage-0000002365395897
- Why: The two callbacks receive the same value and disagree about its type in the same expression. The mistyping is harmless at runtime only because JSON.stringify accepts any argument; it becomes a real defect the moment the key generator does anything numeric with its parameter, and it defeats the type checking that would otherwise catch such a change. The key generator also omits the index parameter, so the framework appends the index itself - the fallback the same guide warns can produce duplicate keys.
- Fix: Type the parameter as the element type and declare the index explicitly: `}, (item: ShowData, index: number) => item.groupName);` - keying on a stable field rather than on the serialised object.

### `HW-19-0112` - Two one-shot delays are implemented with setInterval that clears itself inside its own callback, and neither timer is cancelled if the dialog closes first

- Category B, severity low, confidence confirmed
- Features: COMMON-37
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/37_text_order_verification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/text_order_verification-0000002347027492
- Why: setTimeout expresses a single delayed action and needs no self-clearing, so the setInterval form is both misleading and one clearInterval away from firing forever if the guard is ever edited. More concretely, the timers outlive the dialog: intervalID2 calls DialogUtils.closeCustomDialog() 300 ms later with no check that the dialog is still open, and intervalID1 touches module-level verification state that refreshText may have reset in the meantime - for instance if the user dismisses the dialog with the close button within those 300 ms.
- Fix: Use setTimeout for both, keep the handles in module or component state, and clear them when the dialog is dismissed or the challenge is refreshed.

### `HW-19-0116` - This document uses a 环境准备 section where all fifty sibling documents in the industry use 约束与限制, and omits the HarmonyOS SDK line they all carry

- Category F, severity low, confidence confirmed
- Features: COMMON-38
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/38_slide_verification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/slide_verification-0000002348282886
- Why: The industry's documents are read as a set, and readers scan for the 约束与限制 heading to find the version floor. A differently named section is easy to miss, and the missing 本示例支持HarmonyOS x Release SDK 及以上版本 line removes the one bullet that names the runtime rather than the API level - which matters more here than anywhere else, because this is the only document in the industry that requires something newer than API 20 / HarmonyOS 6.0.0.
- Fix: Rename the section to 约束与限制 and add the HarmonyOS SDK bullet, for example 本示例支持HarmonyOS 6.1.1 Release SDK及以上版本, so the reader sees at a glance that this solution has a higher floor than the rest of the industry.

### `HW-19-0119` - Two directory names in the 工程目录 tree do not match the shipped project: components versus component, and a misspelt entrybackupablility

- Category E, severity low, confidence confirmed
- Features: COMMON-39
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/39_privacy_mode.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/privacy_mode-0000002376419961
- Why: The 工程目录 section is what a reader follows to recreate the layout, and both errors are silent: a page importing '../components/BindSheetBuilder' fails to resolve, and a module.json5 srcEntry pointing at ./ets/entrybackupablility/EntryBackupAbility.ets fails at build time with a message about a missing file rather than about a typo. The project also enables strictMode caseSensitiveCheck, so path spelling is enforced rather than forgiving.
- Fix: Correct the tree to component (singular, matching the ZIP) and entrybackupability, or rename the directory in the sample to components if the plural is intended - but make the two agree.

### `HW-19-0122` - The 工程目录 tree names the component directory in the singular while the shipped project uses the plural

- Category E, severity low, confidence confirmed
- Features: COMMON-40
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/40_language_switch.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/language_switch-0000002381506029
- Why: The 工程目录 section is the reference for recreating the layout, and this project builds with strictMode caseSensitiveCheck enabled, so a path that differs from the shipped one does not silently resolve - an import written from the documented tree fails. The inconsistency between neighbouring documents in the same industry also removes the reader's ability to infer the convention.
- Fix: Correct the tree to components/MessageTab.ets, matching the shipped project, and align the naming with the other documents in the industry.

### `HW-19-0125` - TabBar declares isGrayedOut as @Link although the child never writes it; @Prop is the documented decorator for one-way data

- Category C, severity low, confidence confirmed
- Features: COMMON-41
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/41_page_grayscale.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/page_grayscale-0000002368995430
- Why: The decorator is the declaration of intent that the next reader relies on. @Link says the child may write back and that a write is meant to reach the parent; here nothing does, so the declaration is misleading, and it also removes the compiler's ability to catch an accidental write in the child - with @Prop such a write stays local, with @Link it silently changes the parent's state and re-renders the whole page. @Link additionally forbids local initialization, so the child cannot be instantiated or previewed without a parent supplying the value.
- Fix: Declare `@Prop isGrayedOut: boolean = false;` in TabBar. The parent call site in MainPage.ets:56-58 needs no change - @Prop accepts the same `isGrayedOut: this.isGrayedOut` initialization.

### `HW-19-0126` - The 实现思路 snippet is not the shipped code: it drops the isGrayedOut assignment and its braces are misaligned

- Category E, severity low, confidence confirmed
- Features: COMMON-41
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/41_page_grayscale.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/page_grayscale-0000002368995430
- Why: The snippet is the whole of the document's implementation guidance, and as printed it produces a page that turns gray while the selected tab keeps a red-tinted highlight that grayscale renders as a dull pink - the very artefact the sample's second variable exists to avoid. The document never mentions that a selected-state highlight has to be neutralised separately, so a reader has no way to know the snippet is incomplete. The misaligned closing braces additionally make the block unreadable and, copied verbatim, wrong.
- Fix: Print the shipped aboutToAppear verbatim, including this.isGrayedOut = true, with correct indentation, and add a sentence explaining that accent-coloured selected states should be switched to a neutral asset when the page is grayed.

### `HW-19-0130` - The Logger constructor calls toUpperCase on the format string and discards the result, and onConfigurationUpdate contains an empty dark-mode branch

- Category B, severity low, confidence confirmed
- Features: COMMON-42
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/42_dark_mode.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/dark_mode-0000002403773657
- Why: Both are statements that read as behaviour and produce none. The toUpperCase call suggests the format is normalised in some way that a maintainer must preserve, when nothing happens - and had it worked, uppercasing '%{public}s' would have produced '%{PUBLIC}S', which is not a valid hilog privacy specifier. The empty branch is worse in a sample whose subject is dark mode: it announces that the status bar needs separate handling in dark mode and then ships nothing, leaving a reader to assume either that the framework handles it or that the line is unfinished, with no way to tell which.
- Fix: Delete the toUpperCase statement. Either implement the status-bar handling in the dark branch - through window.setWindowSystemBarProperties - or delete the branch and keep only the AppStorage write.

### `HW-19-0131` - The rawfile read failure is logged against a file name the code never opens

- Category B, severity low, confidence confirmed
- Features: COMMON-42
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/42_dark_mode.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/dark_mode-0000002403773657
- Why: The catch covers two distinct failures - the rawfile being absent or unreadable, and JSON.parse rejecting its contents - and the only diagnostic it emits points at a file that does not exist. Someone debugging an empty product grid in the field is sent to look for the wrong artefact, and the mismatch is invisible from the log itself. The catch also swallows both failures into an empty array, so the log line is the sole evidence that anything went wrong.
- Fix: Name the file that is actually read - Failed to read product data from rawfile product_detail.json - and keep the error code and message interpolation as it is.

### `HW-19-0136` - The dialog is configured isModal: true with a comment stating the opposite, and the receiver logs under the sender's tag

- Category B, severity low, confidence confirmed
- Features: COMMON-43
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/43_customizing_common_event.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/customizing_common_event-0000002470917738
- Why: Both defects mislead whoever reads the log or the code next. The comment claims a behaviour opposite to the one configured, so a maintainer who trusts it and needs a modal dialog will change a value that is already correct; and since true is also the documented default, the parameter as written adds nothing but the wrong comment. The misattributed tag is worse in this sample specifically: the whole point of the demo is two applications exchanging an event, and a parse failure in the receiver prints under commonEventSender, sending anyone reading a combined log to the wrong process.
- Fix: Delete the isModal line or correct the comment to 设为模态弹窗. Change the receiver's log tag to commonEventReceiver.

### `HW-19-0140` - Three modules are imported through raw @ohos.* paths instead of their kits, and notificationManager is imported twice under two names in the same file

- Category A, severity low, confidence confirmed
- Features: COMMON-44
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/44_vpn_adaptation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vpn_adaptation-0000002507518793
- Why: The kits are the documented import surface, and this project shows both styles side by side - notificationManager reaches the code through @kit.NotificationKit in MainPage and through @ohos.notificationManager in NotificationPublish, so the same type arrives under two module identities in one build. The duplicate import compounds it inside a single file: two names bound to one module, used alternately in one object literal, which reads as though notification and notificationManager were different APIs.
- Fix: Use the kits everywhere - import { notificationManager } from '@kit.NotificationKit', import { vpnExtension } from '@kit.NetworkKit', import { wantAgent } from '@kit.AbilityKit' - and keep one name per module in NotificationPublish.ets.

### `HW-19-0142` - The tunnel socket is validated with a truthiness test, which classifies file descriptor 0 as a failure

- Category B, severity low, confidence confirmed
- Features: COMMON-44
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/44_vpn_adaptation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vpn_adaptation-0000002507518793
- Why: The value being tested is a file descriptor, and 0 is a valid one - the check reports CreateTunnel Fail for a socket that was created successfully, shows the failure toast to the user, and then the tunnel descriptor is used anyway by protect(gTunnelFd) and startVpn(gTunFd, gTunnelFd), so the UI and the actual state disagree. The failure return and the valid value collide because the native side chose 0 for both meanings.
- Fix: Return -1 from the native error path and test explicitly on the ArkTS side: if (gTunnelFd >= 0) { ... } else { ... }.

### `HW-19-0147` - Two of the four source directories in the 工程目录 tree do not exist under those names in the shipped project

- Category E, severity low, confidence confirmed
- Features: COMMON-45
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/45_connect_animate.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/connect_animate-0000002508806245
- Why: The 工程目录 section is the map a reader uses to place files when recreating the project, and two of its four entries name directories that do not exist. Case and spelling both matter here: this industry's samples build with strictMode caseSensitiveCheck enabled, so a path written from the documented tree does not resolve. The Mode spelling is worth correcting in the project rather than in the document - it is the only directory in the industry named that way, and the document's model is what a reader would expect.
- Fix: Rename Mode to model and Utils to util in the project, updating the two imports in ButtonRoundPage.ets and the two in NetworkManager.ets, so the shipped layout matches the documented tree.

### `HW-19-0152` - The 工程目录 tree misspells the entrybackupability directory and marks two sibling directories as the last child

- Category E, severity low, confidence confirmed
- Features: COMMON-46
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/46_web_scroll_to_top.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_scroll_to_top-0000002538476079
- Why: The 工程目录 section is the layout a reader copies when recreating the project, and this industry's samples build with strictMode caseSensitiveCheck enabled, so a directory created from the documented spelling does not resolve. The duplicated last-child marker additionally breaks the tree's structure at exactly the point where the sample's own component lives - views/ScrollToTopButton.ets is the file that implements the feature the document is about.
- Fix: Correct the spelling to entrybackupability and change the marker before pages to ├── so that views reads as its sibling.

### `HW-19-0158` - The 工程目录 tree omits the NAPI bridge, the build file and the type declarations, which are three of the five files under cpp

- Category E, severity low, confidence confirmed
- Features: COMMON-47
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/47_format_conversion.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/format_conversion-0000002516884120
- Why: The two files the tree does list are pure C helpers with no connection to ArkTS; everything that makes them reachable from the page - the NAPI wrappers, the module registration, the CMake target that produces libentry.so, and the type declarations that let the import compile - is in the three files the tree omits. A reader recreating the project from this section builds the conversion functions and has no way to call them, and the document's 实现思路 does not cover the bridge either: it shows only the body of Utf8ToGb2312.
- Fix: List all five entries under cpp, with a note on what napi_init.cpp and types/libentry/Index.d.ts contribute, and add a step to 实现思路 covering the NAPI export and the type declaration.

### `HW-19-0162` - One of the three source files reaches wifiManager through the raw @ohos.wifiManager path while the other two use the kit

- Category A, severity low, confidence confirmed
- Features: COMMON-48
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/48_wifi_connect.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/wifi_connect-0000002518583904
- Why: WifiScanInfo values cross this boundary: Index.ets holds them as wifiManager.WifiScanInfo[] typed through the kit and passes each one into WiFiInfo, which declares the parameter through the raw module. The two identities describe the same interface but arrive through different module paths, which is the situation the kit imports exist to avoid, and it leaves one file in the project inconsistent with its two neighbours for no reason - the file needs the type only, which the kit exports.
- Fix: Change WiFiInfo.ets line 2 to `import { wifiManager } from '@kit.ConnectivityKit';`, matching the other two files.

### `HW-19-0168` - The 工程目录 tree misspells entrybackupability and marks two sibling directories as the last child

- Category E, severity low, confidence confirmed
- Features: COMMON-49
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/49_pc_status_bar.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pc_status_bar-0000002551100435
- Why: The 工程目录 is the layout a reader copies when recreating the project. entrybackupablility does not exist in any sample in this industry, and the samples build with strictMode caseSensitiveCheck enabled, so a path taken from this tree does not resolve. The duplicated marker breaks the tree exactly where utils/StatusBarUtil.ets sits - the file that holds loadStatusBar and holdStatusBar, which is to say the whole feature.
- Fix: Correct the spelling to entrybackupability and change the marker before pages so utils reads as its sibling. Apply the same correction to the shared template, since 46_web_scroll_to_top.md carries both defects too.

### `HW-19-0172` - onContinue logs the name of a different lifecycle hook, the stack-top variable is named firstPath, and the project tree misspells entrybackupability

- Category E, severity low, confidence confirmed
- Features: COMMON-50
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/50_navigation_continue.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/navigation_continue-0000002522501268
- Why: Three separate labels that describe something other than what is there. The duplicated log line is the most damaging in practice: onContinue and onWindowStageRestore are the two hooks a developer traces when a continuation misbehaves, and both emit the same text, so the log cannot distinguish the source device saving its state from the target device restoring its window. firstPath sends a reader looking for the bottom of the stack when the code takes the top. The tree misspelling is a path that does not exist in any sample in this industry, and these projects build with strictMode caseSensitiveCheck enabled.
- Fix: Change the onContinue log text to Ability onContinue, rename firstPath to topPath, and correct the tree entry to entrybackupability - in this document and in the shared template the other two documents use.

### `HW-19-0176` - The 工程目录 tree gives every child of common and model the last-child marker and writes pages with a malformed prefix

- Category E, severity low, confidence confirmed
- Features: COMMON-51
- Document: `huawei_industry_tree/19_common_technical_solutions/docs/51_file_persistence_permission_acquisition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/file_persistence_permission_acquisition-0000002412498917
- Why: The 工程目录 is the one place a reader sees the whole project shape, and in this sample it is the largest of any document in the industry - seventeen source files across six directories. Connectors that mark every child as the last one destroy the nesting the section exists to convey, and the │ │── prefix on pages leaves the reader unable to tell whether pages is inside model or beside it. The stray // Index.ets header sends anyone searching for the file by its first line to the wrong place, and the single console.error bypasses the tag and level handling every other message in the file uses.
- Fix: Use ├── for every non-final child and └── only for the last one in each directory, and give pages the same ├── prefix as its siblings. Correct the header comment in PersistPermissionUtils.ets and route its one console.error through Logger.

