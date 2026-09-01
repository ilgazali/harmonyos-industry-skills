# Pitfalls

> Generated from `features/*.md`. Source industry: `15_utilities`, 47 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (7)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-15-0078` - Systematic: requestPermissionsFromUser results ignored — refusal treated as granted (three utilities samples)

- Category D, severity high, confidence confirmed
- Features: UTIL-37, UTIL-38, UTIL-41
- Document: `huawei_industry_tree/15_utilities/docs/37_image_position.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_position-0000002491645574
- Why: Every refusal path proceeds as if granted and fails later with raw errors instead of a handled state.
- Fix: Branch on data.authResults.

### `HW-15-0039` - Systematic: (item: string) => item key generators over object arrays — every row keys as '[object Object]' (two samples)

- Category B, severity medium, confidence confirmed
- Features: UTIL-01, UTIL-16
- Document: `huawei_industry_tree/15_utilities/docs/16_taskpool_query.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/taskpool_query-0000002292913090
- Why: All keys identical: LazyForEach/ForEach diffing and item reuse are broken at scale.
- Fix: Return item.path / a real id.

### `HW-15-0044` - Systematic: ImageReceiver pipelines leak — receiver never released/off'd on exit, and error frames skip nextImage.release() (two samples)

- Category B, severity medium, confidence confirmed
- Features: UTIL-17, UTIL-41
- Document: `huawei_industry_tree/15_utilities/docs/17_vehicle_frame_number_identification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vehicle_frame_number_identification-0000002294949328
- Why: Frame delivery stalls after a few bad frames; exit/re-entry accumulates whole camera pipelines.
- Fix: off+release the receiver in the release path; release nextImage on every path.

### `HW-15-0100` - 3 sample projects swallow errors in catch blocks with an empty body

- Category B, severity medium, confidence confirmed
- Features: UTIL-01, UTIL-19, UTIL-27
- Document: `huawei_industry_tree/15_utilities/docs/27_base64_image_save_h5.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/base64_image_save_h5-0000002334961556
- Why: A catch block with an empty body discards the error object and lets execution continue as if the operation had succeeded. The failure becomes invisible: no log, no user feedback, and no way to diagnose it from a released build. In several of these cases the guarded call is the feature the sample exists to demonstrate.
- Fix: Log the error with hilog including the BusinessError code and message, and surface a user-visible result where the operation was user initiated. Never leave the body empty.

### `HW-15-0102` - Systematic: 13 sample projects declare permissions that no code path in the project uses

- Category D, severity medium, confidence confirmed
- Features: UTIL-05, UTIL-08, UTIL-09, UTIL-11, UTIL-18, UTIL-21, UTIL-31, UTIL-32, UTIL-34, UTIL-35, UTIL-36, UTIL-40, UTIL-42
- Document: `huawei_industry_tree/15_utilities/docs/36_audio_inner_record.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio_inner_record-0000002523743763
- Why: A permission declared in module.json5 but never referenced by any .ets or .ts file in the project cannot be exercised, so it is pure over-declaration. Over-declared permissions are a release-review rejection reason, and restricted permissions especially so. Because these module.json5 files are copied as templates, the surplus entries propagate into products that have even less claim to them.
- Fix: Delete every declared permission the code does not exercise. Where the capability is reached through a permission-free path such as SaveButton or PhotoViewPicker, no declaration is needed at all.

### `HW-15-0016` - Systematic: finally-block closeSync on possibly-undefined files (three utilities samples)

- Category B, severity low, confidence confirmed
- Features: UTIL-01, UTIL-27, UTIL-46
- Document: `huawei_industry_tree/15_utilities/docs/01_practice-tools-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tools-app-architecture-v1-0000002041326514
- Why: Routine cancel/failure paths crash in cleanup code instead of returning quietly.
- Fix: if (file) closeSync(file).

### `HW-15-0101` - Systematic: 43 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: UTIL-01, UTIL-03, UTIL-04, UTIL-05, UTIL-06, UTIL-08, UTIL-09, UTIL-10, UTIL-11, UTIL-12, UTIL-13, UTIL-14, UTIL-15, UTIL-16, UTIL-17, UTIL-18, UTIL-19, UTIL-20, UTIL-21, UTIL-22, UTIL-23, UTIL-24, UTIL-25, UTIL-26, UTIL-27, UTIL-28, UTIL-30, UTIL-31, UTIL-32, UTIL-33, UTIL-34, UTIL-35, UTIL-36, UTIL-37, UTIL-38, UTIL-39, UTIL-40, UTIL-41, UTIL-42, UTIL-43, UTIL-44, UTIL-45, UTIL-46
- Document: `huawei_industry_tree/15_utilities/docs/36_audio_inner_record.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio_inner_record-0000002523743763
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (103)

### `HW-15-0010` - Whitepaper preview is doubly broken: 'rawfile/' path prefix points at a nonexistent subdirectory and the referenced PDFs are not shipped

- Category B, severity high, confidence confirmed
- Features: UTIL-01
- Document: `huawei_industry_tree/15_utilities/docs/01_practice-tools-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tools-app-architecture-v1-0000002041326514
- Why: Tapping the only two functional template items (indexes 6/7) throws an uncaught exception instead of showing the PDF — the Temp tab's working flows are dead.
- Fix: Drop the prefix; ship the PDFs (or stub the items).

### `HW-15-0027` - Module-level statusOutput is never cleared — the second check shows stale data and index-shifted fields

- Category B, severity high, confidence confirmed
- Features: UTIL-09
- Document: `huawei_industry_tree/15_utilities/docs/09_network_awareness.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/network_awareness-0000002284512886
- Why: Do-it-twice on the app's single core function: from the second click the UI永远 shows the first snapshot / shifted rows.
- Fix: statusOutput = [] at the start of each check.

### `HW-15-0034` - Float-ball drag accumulates CUMULATIVE gesture offsets per frame — and adds vp values into px state

- Category B, severity high, confidence confirmed
- Features: UTIL-12
- Document: `huawei_industry_tree/15_utilities/docs/12_app_float_tool_ball.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_float_tool_ball-0000002322067001
- Why: The ball races far ahead of the finger on every drag; on density-3 screens the units are also 3x off.
- Fix: Capture the start position; convert units.

### `HW-15-0062` - Save flow is doubly broken: MIME-encoded base64 decoded in BASIC mode (fails for real images) and the file write races the finally-close with an unconditional success toast

- Category B, severity high, confidence probable
- Features: UTIL-27
- Document: `huawei_industry_tree/15_utilities/docs/27_base64_image_save_h5.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/base64_image_save_h5-0000002334961556
- Why: The app's only feature (save the H5 image) fails for real images, and even when decode succeeds the user is told 'saved' regardless of the write outcome.
- Fix: decode(str, util.Type.MIME); await fs.write; branch the toast.

### `HW-15-0074` - Every write opens the file in APPEND and a fresh download starts at byte 0 — exit-mid-download + relaunch appends a full copy onto stale bytes

- Category B, severity high, confidence confirmed
- Features: UTIL-35
- Document: `huawei_industry_tree/15_utilities/docs/35_rcp_download_pause.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/rcp_download_pause-0000002473755986
- Why: The saved file becomes corrupt (partial data + full re-download); the resume math also disagrees with the on-disk state.
- Fix: TRUNC on NONE-state starts (or delete first).

### `HW-15-0083` - Use-after-free on the napi queue-failure path (reproduced verbatim in the doc), plus a leak and 'undefined undefined' failure messages

- Category B, severity high, confidence confirmed
- Features: UTIL-40
- Document: `huawei_industry_tree/15_utilities/docs/40_ping_tool.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/ping_tool-0000002506881396
- Why: Freed-memory dereference on the error path; failed pings show a nonsense message.
- Fix: Reorder the calls; reject with an Error object.

### `HW-15-0084` - Album decode flow dead: hardcoded sandbox path missing the leading '/' — copy/open always fail

- Category B, severity high, confidence confirmed
- Features: UTIL-41
- Document: `huawei_industry_tree/15_utilities/docs/41_tools-v1_2-ts_59.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/tools-v1_2-ts_59-0000002329284128
- Why: The album-pick conversion — half the sample — never works.
- Fix: Use filesDir; add catch/release.

### `HW-15-0104` - updatePixelMap does not await writeBufferToPixels before packing and saving the image, so the saved file can hold the pre-edit pixels

- Category B, severity high, confidence confirmed
- Features: UTIL-01
- Document: `huawei_industry_tree/15_utilities/docs/01_practice-tools-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tools-app-architecture-v1-0000002041326514
- Why: writeBufferToPixels returns a Promise. The call is not awaited and updatePixelMap is not async, so packToData at :215 can read the PixelMap while the worker buffer is still being copied into it. The user sees the edit applied on screen, because the UI re-renders later, but the file written to the gallery can contain the previous content. This is a race, so it fails intermittently and is easy to mistake for a worker bug.
- Fix: Make updatePixelMap async and await writeBufferToPixels, or chain the rest of the method onto its then(). Use writeBufferToPixelsSync only if the buffer is small enough to justify blocking the UI thread.

### `HW-15-0001` - Architecture template declares seven permissions including unused DISTRIBUTED_DATASYNC, restricted WRITE_IMAGEVIDEO and deprecated READ/WRITE_MEDIA

- Category D, severity medium, confidence confirmed
- Features: UTIL-01
- Document: `huawei_industry_tree/15_utilities/docs/01_practice-tools-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tools-app-architecture-v1-0000002041326514
- Why: This is the industry's flagship architecture template — its permission block gets copied first; it ships an unused multi-device permission, a restricted album permission, and a deprecated permission pair.
- Fix: Prune module.json5 to the permissions the sample uses and replace the deprecated READ/WRITE_MEDIA.

### `HW-15-0004` - Orientation sensor subscription is never unsubscribed — the sensor keeps sampling after the page exits

- Category B, severity medium, confidence confirmed
- Features: UTIL-23
- Document: `huawei_industry_tree/15_utilities/docs/23_spirit_level.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/spirit_level-0000002362176089
- Why: A continuously sampling hardware sensor left subscribed after page destruction drains battery and fires callbacks into detached state; sensor demos must model the off() lifecycle.
- Fix: Add the teardown hook with the off call.

### `HW-15-0011` - Default A4 page size is square: width and height are both the 297 mm value

- Category B, severity medium, confidence confirmed
- Features: UTIL-01
- Document: `huawei_industry_tree/15_utilities/docs/01_practice-tools-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tools-app-architecture-v1-0000002041326514
- Why: All print jobs are submitted with a wrong paper width, distorting print scaling on the main path.
- Fix: Fix the width constant.

### `HW-15-0012` - '立即打印' with no document crashes: fileList[0] used with no empty guard (and each item's copies stepper shares one page-level count)

- Category B, severity medium, confidence confirmed
- Features: UTIL-01
- Document: `huawei_industry_tree/15_utilities/docs/01_practice-tools-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tools-app-architecture-v1-0000002041326514
- Why: Tap print before adding a file → crash; with multiple documents the per-item copy counts silently diverge from the display.
- Fix: Guard fileList.length; move printCount into the item model.

### `HW-15-0013` - Restore-original in the image editor is a guaranteed no-op: savepixMap aliases the mutated PixelMap

- Category B, severity medium, confidence confirmed
- Features: UTIL-01
- Document: `huawei_industry_tree/15_utilities/docs/01_practice-tools-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tools-app-architecture-v1-0000002041326514
- Why: After any crop/rotate the original image can never be recovered.
- Fix: clonePixelMap into savepixMap at load.

### `HW-15-0018` - Sector edge-grab fails across the ±π wraparound

- Category B, severity medium, confidence confirmed
- Features: UTIL-04
- Document: `huawei_industry_tree/15_utilities/docs/04_control_sector.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/control_sector-0000002238010112
- Why: Dragging an edge near 180° — the sample's core interaction — half-fails.
- Fix: Wrap the difference before comparing.

### `HW-15-0020` - Download failure skips all cleanup: isRunning stuck at '测速中' forever and the netQos listener leaks (stale tmp.zip also yields an instant 0-Mbps report)

- Category B, severity medium, confidence confirmed
- Features: UTIL-05
- Document: `huawei_industry_tree/15_utilities/docs/05_network_speed_guage.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/network_speed_guage-0000002284206189
- Why: One failed download permanently disables the speed test button; an interrupted previous run reports a bogus 0 result.
- Fix: Run the cleanup in catch/finally; unlink tmp.zip at start.

### `HW-15-0022` - Swipe-delete is fake: files are never unlinked, so refresh/onPageShow resurrects every 'deleted' ringtone

- Category B, severity medium, confidence confirmed
- Features: UTIL-06
- Document: `huawei_industry_tree/15_utilities/docs/06_ringtone.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/ringtone-0000002284229333
- Why: Delete silently undoes itself within the same session.
- Fix: unlinkSync the sandbox file.

### `HW-15-0024` - Drawer's section tab bar is dead: tabOpacity is initialized 0 and never assigned, so the guard can never pass

- Category B, severity medium, confidence confirmed
- Features: UTIL-07
- Document: `huawei_industry_tree/15_utilities/docs/07_drawer_sliding_effect.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/drawer_sliding_effect-0000002296764169
- Why: The implemented anchor-tab navigation (tabBar + scrollToIndex) is unreachable.
- Fix: Assign tabOpacity where intended; fix the anchor id.

### `HW-15-0026` - Share flow: pack failure still opens the share sheet with an empty file; cleanup uses rmdirSync on a regular file; shareFunc fired without await/catch

- Category B, severity medium, confidence confirmed
- Features: UTIL-08
- Document: `huawei_industry_tree/15_utilities/docs/08_web_poster_produce_and_share.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_poster_produce_and_share-0000002282232164
- Why: Users can share a zero-byte image after a pack failure; the cleanup path is dead; early failures give no feedback.
- Fix: Guard the share on success; use unlink; await shareFunc.

### `HW-15-0028` - Hex color assembled without zero-padding — malformed '#' strings for any channel < 16

- Category B, severity medium, confidence confirmed
- Features: UTIL-10
- Document: `huawei_industry_tree/15_utilities/docs/10_change_color_by_picture.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/change_color_by_picture-0000002320644189
- Why: The gradient renders a wrong color for many images — the sample's core effect.
- Fix: Pad each component.

### `HW-15-0035` - Initial float-ball position hardcoded at (1100, 2300) px — off-screen on smaller displays

- Category B, severity medium, confidence confirmed
- Features: UTIL-12
- Document: `huawei_industry_tree/15_utilities/docs/12_app_float_tool_ball.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_float_tool_ball-0000002322067001
- Why: The ball is created fully off-screen on any display under ~1148x2348 px.
- Fix: Compute from getDefaultDisplaySync() minus window size (the snapping code already does).

### `HW-15-0038` - taskpool.execute has no .catch: page-exit cancel is an unhandled rejection and a failed create-task blocks the button forever

- Category B, severity medium, confidence confirmed
- Features: UTIL-16
- Document: `huawei_industry_tree/15_utilities/docs/16_taskpool_query.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/taskpool_query-0000002292913090
- Why: Every mid-query exit throws unhandled; one failure permanently blocks task creation with the 'task running' toast.
- Fix: Add .catch mirroring the callback cleanup.

### `HW-15-0040` - File sizes displayed one unit too big: bytes labeled KB (1 MB shows as 1GB)

- Category B, severity medium, confidence confirmed
- Features: UTIL-16
- Document: `huawei_industry_tree/15_utilities/docs/16_taskpool_query.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/taskpool_query-0000002292913090
- Why: Every listed size is inflated 1024x.
- Fix: Prepend 'B' to UNITS.

### `HW-15-0042` - Manual VIN query is completely dead: the TextInput has no onChange, so typed input never reaches state

- Category B, severity medium, confidence confirmed
- Features: UTIL-17
- Document: `huawei_industry_tree/15_utilities/docs/17_vehicle_frame_number_identification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vehicle_frame_number_identification-0000002294949328
- Why: First-run user types a VIN and taps 查询车辆信息 — nothing happens.
- Fix: Use $$this.frameNo or onChange.

### `HW-15-0043` - VIN regex is mangled: first-char class contains a space and lowercase 'ens' while omitting A,B,C,D,E,K,M,N,P,U

- Category B, severity medium, confidence confirmed
- Features: UTIL-17
- Document: `huawei_industry_tree/15_utilities/docs/17_vehicle_frame_number_identification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vehicle_frame_number_identification-0000002294949328
- Why: The scanner silently never recognizes whole families of real VINs.
- Fix: Fix the class.

### `HW-15-0046` - Wi-Fi QR parsed by fixed field positions and naive splits — legal orders/escapes produce wrong credentials

- Category B, severity medium, confidence confirmed
- Features: UTIL-18
- Document: `huawei_industry_tree/15_utilities/docs/18_wifi_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/wifi_scan-0000002330272949
- Why: Any nonconforming-but-valid QR silently produces a wrong config and a failed connection.
- Fix: Parse by field tags.

### `HW-15-0047` - securityType hardcoded to WPA2 (T: field ignored); toggle handler awaits getLinkedInfo with no catch

- Category B, severity medium, confidence confirmed
- Features: UTIL-18
- Document: `huawei_industry_tree/15_utilities/docs/18_wifi_scan.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/wifi_scan-0000002330272949
- Why: Only WPA2-PSK networks can ever join; flipping the toggle without a connection throws unhandled.
- Fix: Map T: to securityType; try/catch.

### `HW-15-0048` - Both unzip paths toast success unconditionally — zlib errors are logged as info and reported as '解压成功' (with isFinish set before the work runs)

- Category B, severity medium, confidence confirmed
- Features: UTIL-19
- Document: `huawei_industry_tree/15_utilities/docs/19_unzip_compressed_file_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/unzip_compressed_file_demo-0000002331316561
- Why: Corrupt/missing zips — the primary failure mode of an unzip demo — are invisible: checkmark + success toast regardless.
- Fix: Branch on errData; move isFinish into the callback.

### `HW-15-0049` - Worker leaks whenever it fails: terminate() only lives in onmessage and no onerror/onexit is registered

- Category B, severity medium, confidence confirmed
- Features: UTIL-19
- Document: `huawei_industry_tree/15_utilities/docs/19_unzip_compressed_file_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/unzip_compressed_file_demo-0000002331316561
- Why: Any worker error strands the thread for the app's lifetime.
- Fix: Register error handlers that terminate.

### `HW-15-0050` - 0 ÷ 0 displays the literal string 'null'

- Category B, severity medium, confidence confirmed
- Features: UTIL-20
- Document: `huawei_industry_tree/15_utilities/docs/20_calculator.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calculator-0000002298744774
- Why: A fully enterable expression shows a bogus 'null' instead of the error text.
- Fix: Add Number.isNaN(result).

### `HW-15-0051` - Thousands separator is a dead feature: the RegExp is built from a string containing the literal '/'-delimiters and stripped backslashes (and its @Watch guard is always-true)

- Category B, severity medium, confidence confirmed
- Features: UTIL-20
- Document: `huawei_industry_tree/15_utilities/docs/20_calculator.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calculator-0000002298744774
- Why: 1234567 is never rendered as 1,234,567 anywhere; fixing the regex alone would then loop the watcher.
- Fix: Regex literal + && guard (strip commas before parsing).

### `HW-15-0053` - Systematic: NDEF UTF-16 text branch passes 'utf-16' — not a supported buffer encoding — and ignores the BE/BOM format (two samples sharing NdefTagModel)

- Category B, severity medium, confidence probable
- Features: UTIL-21
- Document: `huawei_industry_tree/15_utilities/docs/21_nfc_read_write.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/nfc_read_write-0000002299096544
- Why: The code path written for UTF-16 text records can never succeed; unknown URI prefix codes corrupt the parsed URL.
- Fix: Decode utf16 properly; default the prefix to ''.

### `HW-15-0055` - Compass cleanup destory() is never called — three sensors plus the location listener run for the app's lifetime

- Category B, severity medium, confidence confirmed
- Features: UTIL-22
- Document: `huawei_industry_tree/15_utilities/docs/22_compass_effect.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/compass_effect-0000002317882746
- Why: ORIENTATION/MAGNETIC_FIELD/BAROMETER and locationChange keep sampling after the page is gone (battery/privacy).
- Fix: Call destory() in aboutToDisappear.

### `HW-15-0056` - getLocation() runs unguarded before/without permission and double-subscribes on grant; heading shows 360° instead of 0°

- Category B, severity medium, confidence confirmed
- Features: UTIL-22
- Document: `huawei_industry_tree/15_utilities/docs/22_compass_effect.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/compass_effect-0000002317882746
- Why: Refusal path throws unhandled; grant path stacks listeners; pointing north can display 360°.
- Fix: Move the call into the granted branch; % 360 before display.

### `HW-15-0057` - Countdown centiseconds invert once elapsed exceeds 100 s

- Category B, severity medium, confidence confirmed
- Features: UTIL-24
- Document: `huawei_industry_tree/15_utilities/docs/24_custom_start_time_timer.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_start_time_timer-0000002330585486
- Why: Any countdown longer than 100 s shows wrong centiseconds for its whole tail.
- Fix: Compute from remaining, not (100 - elapsed).

### `HW-15-0060` - First-run marquee renders '[object Object]': the default content is a Resource concatenated as a string

- Category B, severity medium, confidence probable
- Features: UTIL-26
- Document: `huawei_industry_tree/15_utilities/docs/26_led_scroll_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/led_scroll_text-0000002366978977
- Why: Before any user edit the full-screen LED shows '[object Object]'.
- Fix: getStringSync the resource in aboutToAppear.

### `HW-15-0063` - Save depends on restricted WRITE_IMAGEVIDEO (auto-denied without ACL) and the helper 'components' are new-ed structs whose getUIContext() has no UI instance

- Category B, severity medium, confidence probable
- Features: UTIL-27
- Document: `huawei_industry_tree/15_utilities/docs/27_base64_image_save_h5.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/base64_image_save_h5-0000002334961556
- Why: On a normally signed build the feature cannot succeed, and the refusal feedback itself is broken.
- Fix: Use showAssetsCreationDialog; pass UIContext/host context as parameters.

### `HW-15-0065` - Hand-rolled UTF-8 decoder drops 4-byte sequences — emoji silently deleted from RSA1/RSA3 decryptions

- Category B, severity medium, confidence confirmed
- Features: UTIL-29
- Document: `huawei_industry_tree/15_utilities/docs/29_rsa_encryption_and_decryption.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/rsa_encryption_and_decryption-0000002465072997
- Why: Messages containing emoji encrypt fine but decrypt with those characters missing — a broken round trip.
- Fix: Use buffer.from(...).toString('utf-8').

### `HW-15-0066` - Error arguments ignored twice: OAEP doFinal dereferences data on failure, and the encrypt buttons render 'undefined' on oversize input

- Category B, severity medium, confidence confirmed
- Features: UTIL-29
- Document: `huawei_industry_tree/15_utilities/docs/29_rsa_encryption_and_decryption.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/rsa_encryption_and_decryption-0000002465072997
- Why: Oversize input crashes the OAEP path silently and displays 'undefined' in the basic paths.
- Fix: Check err before data.

### `HW-15-0068` - HMAC key validated as 32 characters but 'HMAC|SHA256' needs 32 BYTES — non-ASCII keys reject unhandled (and the result flag flips before the await)

- Category B, severity medium, confidence probable
- Features: UTIL-30
- Document: `huawei_industry_tree/15_utilities/docs/30_calculation_and_verification_of_hmac.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calculation_and_verification_of_hmac-0000002431485178
- Why: A legal-looking key fails silently with no feedback.
- Fix: Validate byte length; try/catch and gate the flag.

### `HW-15-0069` - One failed download group deadlocks the batch forever: no .catch on taskpool.execute or showAssetsCreationDialog, and LongTasks are never terminated on failure

- Category B, severity medium, confidence confirmed
- Features: UTIL-31
- Document: `huawei_industry_tree/15_utilities/docs/31_batch_download_images_and_videos.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/batch_download_images_and_videos-0000002464972285
- Why: A single group failure hangs the whole batch UI until restart; exits leave LongTasks downloading.
- Fix: .catch marking the group done; call cancelTask in aboutToDisappear.

### `HW-15-0070` - Sandbox filenames miss the dot before the extension ('image-<ts>jpeg') and are timestamp-only (concurrent collisions overwrite)

- Category B, severity medium, confidence confirmed
- Features: UTIL-31
- Document: `huawei_industry_tree/15_utilities/docs/31_batch_download_images_and_videos.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/batch_download_images_and_videos-0000002464972285
- Why: Extension-less files feed the album-save validation; same-millisecond completions overwrite each other's content.
- Fix: Add the dot and an index/url-hash.

### `HW-15-0071` - IP refresh: no .catch on getDefaultNet and linkAddresses[0] dereferenced blindly — exactly during the change events that trigger it (subscriptions also attached after register())

- Category B, severity medium, confidence confirmed
- Features: UTIL-32
- Document: `huawei_industry_tree/15_utilities/docs/32_get_ip_address.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/get_ip_address-0000002482813589
- Why: The displayed IP silently freezes on the very transitions the sample demonstrates.
- Fix: Check length; add .catch; reorder.

### `HW-15-0072` - Second selection re-uploads the ENTIRE previous batch: uploadFiles always queues the whole imageList

- Category B, severity medium, confidence confirmed
- Features: UTIL-34
- Document: `huawei_industry_tree/15_utilities/docs/34_pc_upload_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pc_upload_image-0000002459568386
- Why: The same images upload twice to the server.
- Fix: Track flushed uris (or queue the delta).

### `HW-15-0073` - Server-name round trip never matches: the preserve-old-entry logic is dead so every refresh resets all file times ('.jpg.jpg' names feed it)

- Category B, severity medium, confidence confirmed
- Features: UTIL-34
- Document: `huawei_industry_tree/15_utilities/docs/34_pc_upload_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pc_upload_image-0000002459568386
- Why: Displayed modification times reset on every refresh; server filenames are corrupted/mislabeled.
- Fix: Normalize one canonical name format.

### `HW-15-0075` - Pause rejects the awaited fetch with no try/catch — every pause is an unhandled rejection (and rapid re-taps start concurrent fetches)

- Category B, severity medium, confidence confirmed
- Features: UTIL-35
- Document: `huawei_industry_tree/15_utilities/docs/35_rcp_download_pause.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/rcp_download_pause-0000002473755986
- Why: The sample's core pause action throws unhandled; double-taps corrupt the download.
- Fix: Catch the cancel rejection; guard on current state.

### `HW-15-0076` - BackgroundTask module: BusinessError used without import (does not compile), notification want targets 'entryAbility' (case mismatch), and the invalid-context guard doesn't return

- Category B, severity medium, confidence confirmed
- Features: UTIL-36
- Document: `huawei_industry_tree/15_utilities/docs/36_audio_inner_record.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio_inner_record-0000002523743763
- Why: The background-recording support module is broken three ways.
- Fix: Import BusinessError; fix the name; return.

### `HW-15-0077` - Exit mid-recording never stops the inner-audio capture or the 40 ms waveform interval

- Category B, severity medium, confidence probable
- Features: UTIL-36
- Document: `huawei_industry_tree/15_utilities/docs/36_audio_inner_record.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio_inner_record-0000002523743763
- Why: Native capture keeps running after the UI is gone; the recording file is never finalized.
- Fix: Stop capture and clear the interval in aboutToDisappear.

### `HW-15-0079` - MEDIA_LOCATION declared but never requested — EXIF GPS is redacted so the core feature fails for real photos

- Category D, severity medium, confidence confirmed
- Features: UTIL-37
- Document: `huawei_industry_tree/15_utilities/docs/37_image_position.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_position-0000002491645574
- Why: Gallery photos' GPS EXIF returns redacted values without the grant; markers never reflect real photo positions.
- Fix: Add it to the requested list.

### `HW-15-0080` - DMS parsing truncates fractional seconds with parseInt, and photos without GPS drop a marker at (0,0)

- Category B, severity medium, confidence confirmed
- Features: UTIL-37
- Document: `huawei_industry_tree/15_utilities/docs/37_image_position.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_position-0000002491645574
- Why: Systematic coordinate error plus a misleading (0,0) marker for any non-geotagged photo.
- Fix: Use parseFloat and guard the no-GPS case.

### `HW-15-0081` - Location-tagged capture: lat/lon/alt read undefined before the first fix (or after refusal), and the whole camera teardown/init pipeline races

- Category B, severity medium, confidence confirmed
- Features: UTIL-38
- Document: `huawei_industry_tree/15_utilities/docs/38_convenient-life-v1_2-ts_133.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/convenient-life-v1_2-ts_133-0000002429890325
- Why: Shots before a fix (or after refusal) carry no/broken location; camera switch races teardown against re-open.
- Fix: Guard on the fix; await releases; set isFront.

### `HW-15-0085` - readerModeCb error branch falls through into tagInfo.technology — crash instead of return

- Category B, severity medium, confidence confirmed
- Features: UTIL-42
- Document: `huawei_industry_tree/15_utilities/docs/42_hce_link_app.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/hce_link_app-0000002550216593
- Why: A callback error becomes a TypeError crash.
- Fix: Add the return.

### `HW-15-0086` - NfcUtil.ndef is never assigned — the writable/read-only rows always show 'undefined' and the record list is always empty

- Category B, severity medium, confidence confirmed
- Features: UTIL-42
- Document: `huawei_industry_tree/15_utilities/docs/42_hce_link_app.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/hce_link_app-0000002550216593
- Why: A whole documented read-page section is dead.
- Fix: Construct NdefTagModel in readerModeCb.

### `HW-15-0087` - hceCmd subscribed on every tab show, never off'd — N callbacks transmit N responses per APDU; the response also lacks SW1SW2

- Category B, severity medium, confidence confirmed
- Features: UTIL-42
- Document: `huawei_industry_tree/15_utilities/docs/42_hce_link_app.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/hce_link_app-0000002550216593
- Why: Tab switching stacks duplicate responders; standards-conformant readers reject the SW-less response.
- Fix: off in onWillHide; append SW1SW2 and branch on the command.

### `HW-15-0088` - KVStore result sets are never closed — the 8-open-result-set limit exhausts on repeated searches

- Category B, severity medium, confidence confirmed
- Features: UTIL-43
- Document: `huawei_industry_tree/15_utilities/docs/43_educate-v1_1-ts_23.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/educate-v1_1-ts_23-0000002309706602
- Why: Repeated searches/refreshes make getResultSet start failing.
- Fix: Close in a finally.

### `HW-15-0089` - KVManager configured with the wrong bundleName 'com.example.datatest' (actual: com.example.localdatabase) — doc repeats it

- Category B, severity medium, confidence confirmed
- Features: UTIL-43
- Document: `huawei_industry_tree/15_utilities/docs/43_educate-v1_1-ts_23.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/educate-v1_1-ts_23-0000002309706602
- Why: createKVManager/closeKVStore fail with invalid-args on a real install.
- Fix: Use the app's bundleName (or context-derived).

### `HW-15-0090` - RDB init: err ignored and CREATE TABLE not awaited before the first query (first-run race); duplicate inserts reject unhandled while the UI pretends success

- Category B, severity medium, confidence confirmed
- Features: UTIL-43
- Document: `huawei_industry_tree/15_utilities/docs/43_educate-v1_1-ts_23.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/educate-v1_1-ts_23-0000002309706602
- Why: Fresh installs can fail the initial load; duplicate names silently 'succeed' in the UI only.
- Fix: await executeSql; then/catch the insert.

### `HW-15-0092` - IME switcher loops over EVERY input method instead of the clicked one

- Category B, severity medium, confidence confirmed
- Features: UTIL-45
- Document: `huawei_industry_tree/15_utilities/docs/45_kikainput.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/kikainput-0000002535498198
- Why: The user ends on an arbitrary input method, not the one clicked.
- Fix: Fix the condition and break.

### `HW-15-0093` - Immersive mode is dead: the page's `panel` field is never assigned, so setImmersiveMode is always a no-op

- Category B, severity medium, confidence confirmed
- Features: UTIL-45
- Document: `huawei_industry_tree/15_utilities/docs/45_kikainput.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/kikainput-0000002535498198
- Why: The documented immersive-mode feature never applies.
- Fix: Route through KeyboardController's panel.

### `HW-15-0094` - Scene2 resizes the keyboard panel to a fixed 100x100 px and hardcodes hot-zone rects for one display (RK branch also assigns a 0.35 RATIO where a pixel height belongs)

- Category B, severity medium, confidence probable
- Features: UTIL-45
- Document: `huawei_industry_tree/15_utilities/docs/45_kikainput.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/kikainput-0000002535498198
- Why: On any other resolution the input region is wrong and a 100x100 panel is unusable.
- Fix: Use the computed dWidth/keyHeight; fix the constant.

### `HW-15-0095` - IME listener on/off pairing broken: cursorContextChange off'd only in debug, off('inputStop') passed a fresh callback, several listeners never off'd

- Category B, severity medium, confidence confirmed
- Features: UTIL-45
- Document: `huawei_industry_tree/15_utilities/docs/45_kikainput.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/kikainput-0000002535498198
- Why: Destroy/recreate cycles stack handlers (resizePanel/inputStart run multiple times).
- Fix: Store and remove each callback.

### `HW-15-0096` - Typed characters double-counted in the preview pipeline: text read AFTER insert then appended again

- Category B, severity medium, confidence probable
- Features: UTIL-45
- Document: `huawei_industry_tree/15_utilities/docs/45_kikainput.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/kikainput-0000002535498198
- Why: The 'hello world' preview triggers on wrong input with an out-of-range setPreviewText.
- Fix: Drop the re-append; use real indices.

### `HW-15-0097` - Second poster loses the QR code: the pixelMap is released after each render and regeneration is async and un-awaited (and the QR is skipped entirely under the wrong guard)

- Category B, severity medium, confidence confirmed
- Features: UTIL-46
- Document: `huawei_industry_tree/15_utilities/docs/46_poster_gen.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/poster_gen-0000002569022355
- Why: First poster OK; every later one (and any no-background poster) silently omits the QR.
- Fix: Keep or await the QR; fix the guard.

### `HW-15-0098` - No-background poster is white-on-white: canvas filled white and title/content brushes are also pure white

- Category B, severity medium, confidence confirmed
- Features: UTIL-46
- Document: `huawei_industry_tree/15_utilities/docs/46_poster_gen.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/poster_gen-0000002569022355
- Why: First-run posters without an uploaded image show invisible text (and no QR per the guard bug).
- Fix: Dark default brushes (or a default background).

### `HW-15-0099` - Mutable class statics used as constants: a long title permanently shrinks TEXT_SIZE and IMG_HEIGHT keeps the last image's height

- Category B, severity medium, confidence confirmed
- Features: UTIL-46
- Document: `huawei_industry_tree/15_utilities/docs/46_poster_gen.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/poster_gen-0000002569022355
- Why: Generate a long-titled poster then a short one: the short title renders at the shrunken size; canvas height carries over.
- Fix: Copy the statics into locals per generation.

### `HW-15-0103` - Systematic: LogTemp creates ImagePacker and ImageSource objects in six files and releases none of them

- Category B, severity medium, confidence confirmed
- Features: UTIL-01
- Document: `huawei_industry_tree/15_utilities/docs/01_practice-tools-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tools-app-architecture-v1-0000002041326514
- Why: image.createImageSource and image.createImagePacker allocate native decoder and encoder objects that the ArkTS garbage collector does not reclaim; the official Image Kit guide releases both explicitly. encode() and getPixelMap() sit on the print path and run once per page the user prints or edits, so the leak grows with normal use. DecodeUtil additionally leaks the file descriptor, which is a bounded per-process resource.
- Fix: Wrap each creation in try/finally and call release() on the ImageSource and ImagePacker in the finally. In DecodeUtil close the descriptor returned by getResourceFd in the same finally. This is the same defect class already recorded for the photography samples as HW-18-0005.

### `HW-15-0105` - The avoidAreaChange listener registered in onWindowStageCreate is never released in onWindowStageDestroy

- Category B, severity medium, confidence confirmed
- Features: UTIL-13
- Document: `huawei_industry_tree/15_utilities/docs/13_input_method_application_immersive_mode.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/input_method_application_immersive_mode-0000002283469114
- Why: The listener is bound to the main window and outlives the window stage; its closure holds the window reference, and a relaunch within the same process stacks another listener on the first so each avoid-area change runs the handler more than once. This sample is specifically about immersive layout, where the avoid-area values drive the page padding, so the listener is load bearing rather than incidental.
- Fix: Keep the callback in a class field and call windowClass.off('avoidAreaChange', callback) in onWindowStageDestroy.

### `HW-15-0106` - The search box width is computed once from the display size in aboutToAppear and never recomputed, so it is wrong after any rotation or fold

- Category C, severity medium, confidence confirmed
- Features: UTIL-13
- Document: `huawei_industry_tree/15_utilities/docs/13_input_method_application_immersive_mode.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/input_method_application_immersive_mode-0000002283469114
- Why: getAllDisplays reports the display size at the moment of the call, in aboutToAppear. Rotating the device or unfolding a foldable changes the width, but screenWidth and the derived searchWidth keep the value measured at page construction, so the search box stays sized for the previous geometry while its container resizes. The same page already demonstrates immersive layout, where the app owns its own insets, so a stale width is visible immediately. Computing a fixed width by subtracting two margin constants from the screen width also reimplements what the layout system does for free.
- Fix: Give the Search component a percentage width or layoutWeight inside its row and delete screenWidth and searchWidth. If an absolute value is genuinely needed, recompute it from an onAreaChange handler on the container rather than from the display.

### `HW-15-0108` - The validity countdown is decremented by an assumed 50 ms per tick instead of being recomputed from the clock, so the displayed window and the refresh moment drift away from the real TOTP window

- Category B, severity medium, confidence confirmed
- Features: UTIL-33
- Document: `huawei_industry_tree/15_utilities/docs/33_dynamic_password_generation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/dynamic_password_generation-0000002489836613
- Why: setInterval does not guarantee its period: ticks are delayed by main-thread work and coalesced when the app is backgrounded, so subtracting a constant 50 accumulates error in one direction only, never catching up. Over a 30 second window the counter therefore lags real time, and because the refresh is triggered by that same lagging counter rather than by the clock, the app keeps displaying a code after its window has closed. For a two-factor token that is the failure that matters: the user reads a six-digit code the server has already rejected. The correct value is one function call away and is already used at refresh time.
- Fix: In the interval callback set this.countDown = HMACUtil.getRemainingMilliseconds() and trigger refreshToken when the computed time window index changes, rather than when a decremented counter reaches zero.

### `HW-15-0109` - A 50 ms interval writes an observed state variable twenty times a second to drive a countdown that is only meaningful to the second

- Category C, severity medium, confidence confirmed
- Features: UTIL-33
- Document: `huawei_industry_tree/15_utilities/docs/33_dynamic_password_generation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/dynamic_password_generation-0000002489836613
- Why: Every assignment to an @State variable makes the framework run its dependency bookkeeping and re-render the components that read it. At a 50 ms period that is twenty state writes and twenty re-render passes per second, sustained for as long as the page is on screen, to move a countdown that a user reads in seconds. The page is the app's only screen, so this runs continuously in the foreground. This is the per-frame state churn HQ's own performance practice tells developers to remove.
- Fix: Raise the interval to 1000 ms and set countDown from the clock on each tick. If a smooth progress arc is wanted, drive it with an animation on a non-state value rather than by re-rendering twenty times a second.

### `HW-15-0110` - The implementation section skips the Base32 decode between the shared secret and the HMAC key, although the prose says the secret is Base32 and the sample performs the decode

- Category E, severity medium, confidence confirmed
- Features: UTIL-33
- Document: `huawei_industry_tree/15_utilities/docs/33_dynamic_password_generation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/dynamic_password_generation-0000002489836613
- Why: The document moves from a Base32 string to a Uint8Array without saying how, and the decode is not incidental: TOTP interoperability depends on the key being the decoded bytes, not the characters of the encoding. A developer following the four documented steps naturally converts the string to bytes directly, producing codes that are self-consistent but that no standard authenticator server accepts. The failure looks like a clock-synchronisation problem, which step 2 explicitly warns about, so it is easy to misdiagnose.
- Fix: Add the decodeBase32 helper to step 1, or state explicitly that the secret must be Base32-decoded to bytes before being converted into an HMAC key.

### `HW-15-0002` - Reference link 'subwindow-guide' does not resolve to any page in the documentation navigation tree

- Category E, severity low, confidence probable
- Features: UTIL-12
- Document: `huawei_industry_tree/15_utilities/docs/12_app_float_tool_ball.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_float_tool_ball-0000002322067001
- Why: The link is the doc's how-to reference for its core mechanism (sub-windows); a dead target strands readers.
- Fix: Point the link at the current sub-window development guide.

### `HW-15-0003` - Constraints claim API 20 while the sample targets 5.0.5(17)

- Category E, severity low, confidence confirmed
- Features: UTIL-23
- Document: `huawei_industry_tree/15_utilities/docs/23_spirit_level.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/spirit_level-0000002362176089
- Why: The constraint overstates the requirement by three API versions against the sample's own configuration.
- Fix: Align the constraint with 5.0.5(17) or bump the sample.

### `HW-15-0005` - Upload helper's constructor starts a 2-second interval that is never cleared

- Category B, severity low, confidence confirmed
- Features: UTIL-34
- Document: `huawei_industry_tree/15_utilities/docs/34_pc_upload_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pc_upload_image-0000002459568386
- Why: Every Upload instance leaves a permanent 2 s heartbeat running for the process lifetime, polling background-task state even when no upload exists; multiple instances stack timers.
- Fix: Store the id, start on first upload, clearInterval when the queue empties or in a dispose() called from the page teardown.

### `HW-15-0006` - AVPlayer is created but release() is never called anywhere in the sample

- Category B, severity low, confidence confirmed
- Features: UTIL-36
- Document: `huawei_industry_tree/15_utilities/docs/36_audio_inner_record.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio_inner_record-0000002523743763
- Why: The player's native resources are never freed; the inner-record demo teaches an incomplete AVPlayer lifecycle.
- Fix: Release the player in the page's aboutToDisappear/onDestroy.

### `HW-15-0007` - Permission section lists GET_NETWORK_INFO the sample never declares, while the sample declares INTERNET the doc never mentions; tree lists Entryability.ets

- Category E, severity low, confidence confirmed
- Features: UTIL-37
- Document: `huawei_industry_tree/15_utilities/docs/37_image_position.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_position-0000002491645574
- Why: Doc and sample disagree in both directions about the permission set, and the tree entry does not resolve on a case-sensitive build.
- Fix: Sync the permission list with module.json5, drop the unused INTERNET declaration, fix the filename.

### `HW-15-0008` - Project tree lists NfcUtils.ets; the zip file is NfcUtil.ets

- Category E, severity low, confidence confirmed
- Features: UTIL-42
- Document: `huawei_industry_tree/15_utilities/docs/42_hce_link_app.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/hce_link_app-0000002550216593
- Why: Tree/zip filename mismatch.
- Fix: Rename the entry to NfcUtil.ets.

### `HW-15-0009` - Doc snippet misspells the constant KVSTORE_SHEET_HEIGHT as KVSORE_SHEET_HEIGHT

- Category E, severity low, confidence confirmed
- Features: UTIL-43
- Document: `huawei_industry_tree/15_utilities/docs/43_educate-v1_1-ts_23.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/educate-v1_1-ts_23-0000002309706602
- Why: Copying the documented snippet fails to compile against the sample's constants; the identifier is a typo of the shipped name.
- Fix: Correct the identifier in the doc.

### `HW-15-0014` - MIME mapping branch order makes the docx/xlsx branches unreachable; continuation-file copy is corrupted by a 1024x allocation and full-buffer writes

- Category B, severity low, confidence confirmed
- Features: UTIL-01
- Document: `huawei_industry_tree/15_utilities/docs/01_practice-tools-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tools-app-architecture-v1-0000002041326514
- Why: Preview Kit gets wrong MIME types for docx/xlsx; the persisted continuation copy is padded/corrupted.
- Fix: Reorder branches (docx before doc); write {length: readLen}.

### `HW-15-0015` - Print camera page ships unfinished: the switch-camera button is an empty function and the captured photo is mis-decoded then discarded

- Category B, severity low, confidence confirmed
- Features: UTIL-01
- Document: `huawei_industry_tree/15_utilities/docs/01_practice-tools-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tools-app-architecture-v1-0000002041326514
- Why: Camera switch does nothing; the shutter's result is both garbage and unused; one of the two preview outputs leaks per exit.
- Fix: Finish or remove the stubs.

### `HW-15-0017` - Stop-scan only stops the counter: sweep/ripple animations keep running and found results stay on screen

- Category B, severity low, confidence probable
- Features: UTIL-03
- Document: `huawei_industry_tree/15_utilities/docs/03_radar_scan_effect.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/radar_scan_effect-0000002236447458
- Why: After 停止扫描 the page still animates and shows stale results, contradicting the button state.
- Fix: Gate the animations on isScanning; reset scannedNum.

### `HW-15-0019` - Doc snippet references Constants.LINE_ONE_CHANGE which does not exist in the sample

- Category D, severity low, confidence confirmed
- Features: UTIL-04
- Document: `huawei_industry_tree/15_utilities/docs/04_control_sector.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/control_sector-0000002238010112
- Why: Copying the documented snippet fails to compile.
- Fix: Fix the snippet.

### `HW-15-0021` - Copy-paste: ctx.font assigned a COLOR string for the 'Mbps' unit label

- Category B, severity low, confidence confirmed
- Features: UTIL-05
- Document: `huawei_industry_tree/15_utilities/docs/05_network_speed_guage.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/network_speed_guage-0000002284206189
- Why: The unit renders with the wrong color/font.
- Fix: Assign to fillStyle.

### `HW-15-0023` - copyRawfile() targets context.resourceDir (resfile) which the project does not ship — it throws on every launch

- Category B, severity low, confidence confirmed
- Features: UTIL-06
- Document: `huawei_industry_tree/15_utilities/docs/06_ringtone.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/ringtone-0000002284229333
- Why: An always-failing dead path executed on every aboutToAppear.
- Fix: Drop the resfile branch.

### `HW-15-0025` - Drawer dead-code cluster: misspelled '__container' anchors, inert positionX slide animation, bare Stack as a List child

- Category B, severity low, confidence confirmed
- Features: UTIL-07
- Document: `huawei_industry_tree/15_utilities/docs/07_drawer_sliding_effect.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/drawer_sliding_effect-0000002296764169
- Why: Invalid anchors, a dead slide effect and a non-conforming list child in the sample's showcase page.
- Fix: Fix the keyword; wire or delete positionX; wrap the Stack.

### `HW-15-0029` - Indicator misconfigured: 8 dots for 4 pages, TWELVE=8 kills the selected-dot emphasis, and dots are vertical over a horizontal Swiper

- Category B, severity low, confidence confirmed
- Features: UTIL-10
- Document: `huawei_industry_tree/15_utilities/docs/10_change_color_by_picture.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/change_color_by_picture-0000002320644189
- Why: The page indicator misrepresents count, orientation and selection.
- Fix: Use imgData.length; fix TWELVE; vertical(false).

### `HW-15-0030` - Doc says the sample takes the picture's AVERAGE color; the code uses getMainColorSync (dominant color)

- Category E, severity low, confidence confirmed
- Features: UTIL-10
- Document: `huawei_industry_tree/15_utilities/docs/10_change_color_by_picture.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/change_color_by_picture-0000002320644189
- Why: The stated mechanism contradicts the implementation.
- Fix: Use getAverageColor or fix the doc.

### `HW-15-0031` - unsubscribe guard tests `!== null` on an optional field whose unset value is undefined

- Category B, severity low, confidence confirmed
- Features: UTIL-11
- Document: `huawei_industry_tree/15_utilities/docs/11_device_info_card.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/device_info_card-0000002286708232
- Why: The guard never protects the failure path it was written for.
- Fix: if (this.subscriber).

### `HW-15-0032` - Default formIdList is [''] — first-run battery events call updateForm('') every time

- Category B, severity low, confidence confirmed
- Features: UTIL-11
- Document: `huawei_industry_tree/15_utilities/docs/11_device_info_card.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/device_info_card-0000002286708232
- Why: Before any card is added, every battery event issues a guaranteed-failing updateForm('').
- Fix: Use an empty array.

### `HW-15-0033` - Doc registers the callee as 'updateBatteryInfoCard' while the widget posts 'updateBatteryInfoCall'

- Category E, severity low, confidence confirmed
- Features: UTIL-11
- Document: `huawei_industry_tree/15_utilities/docs/11_device_info_card.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/device_info_card-0000002286708232
- Why: Following the doc, the battery-card refresh call is never dispatched.
- Fix: Fix the doc.

### `HW-15-0036` - Doc promises three comment flows; two (DeepLink, in-app dialog) are dead code with no UI trigger

- Category E, severity low, confidence confirmed
- Features: UTIL-14
- Document: `huawei_industry_tree/15_utilities/docs/14_comment.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/comment-0000002292394860
- Why: Two of the three documented flows are unreachable in the running sample.
- Fix: Wire the two dead functions or fix the doc.

### `HW-15-0037` - Last field's Next key requests focus on the empty key (fails), and both password fields render unmasked

- Category B, severity low, confidence confirmed
- Features: UTIL-15
- Document: `huawei_industry_tree/15_utilities/docs/15_jump_to_next_input_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/jump_to_next_input_text-0000002326705889
- Why: The final Next press errors instead of Done; passwords display in the clear on a password form.
- Fix: Special-case the last field and password fields.

### `HW-15-0041` - Dialog texts chain .fontWeight twice, the second passing a LINE_HEIGHT constant

- Category B, severity low, confidence confirmed
- Features: UTIL-16
- Document: `huawei_industry_tree/15_utilities/docs/16_taskpool_query.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/taskpool_query-0000002292913090
- Why: Title/subtitle weights are never applied.
- Fix: Rename the second call.

### `HW-15-0045` - Flash support checks are dead: results queried, logged and ignored before an unconditional setFlashMode

- Category B, severity low, confidence confirmed
- Features: UTIL-17
- Document: `huawei_industry_tree/15_utilities/docs/17_vehicle_frame_number_identification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vehicle_frame_number_identification-0000002294949328
- Why: On flashless devices setFlashMode errors instead of being skipped.
- Fix: Branch on the results.

### `HW-15-0052` - '%' allowed directly after +, × and ÷ — '5×%' silently evaluates to 0

- Category B, severity low, confidence confirmed
- Features: UTIL-20
- Document: `huawei_industry_tree/15_utilities/docs/20_calculator.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/calculator-0000002298744774
- Why: Inconsistent guard yields silently wrong results instead of rejecting the keystroke.
- Fix: Extend the guard.

### `HW-15-0054` - Tag sessions opened with connect() are never resetConnection()'d anywhere

- Category E, severity low, confidence confirmed
- Features: UTIL-21
- Document: `huawei_industry_tree/15_utilities/docs/21_nfc_read_write.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/nfc_read_write-0000002299096544
- Why: The tag-kit contract is connect→operate→resetConnection; open sessions block re-detection (read the same tag twice) and leak the channel.
- Fix: Add the calls.

### `HW-15-0058` - 24-hour cap silently disabled on first run: undefined + number = NaN

- Category B, severity low, confidence confirmed
- Features: UTIL-24
- Document: `huawei_industry_tree/15_utilities/docs/24_custom_start_time_timer.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_start_time_timer-0000002330585486
- Why: The maximum-time protection is off until a custom start time is set once.
- Fix: Add the fallback.

### `HW-15-0059` - Drag ghost and drop zones ignore the 60vp page header — everything is shifted down a band

- Category B, severity low, confidence probable
- Features: UTIL-25
- Document: `huawei_industry_tree/15_utilities/docs/25_drag_and_change.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/drag_and_change-0000002364921921
- Why: Drop classification and ghost position disagree with the visual layout.
- Fix: Measure the grid's global origin.

### `HW-15-0061` - Color panel reports a one-touch-stale color: getColor() runs before sat/val update

- Category B, severity low, confidence confirmed
- Features: UTIL-26
- Document: `huawei_industry_tree/15_utilities/docs/26_led_scroll_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/led_scroll_text-0000002366978977
- Why: The selected color always lags the tracker by one touch event.
- Fix: Reorder the statements.

### `HW-15-0064` - Estimation tab first-run shows an empty result, and the solar branch lacks the range clamp its lunar sibling has

- Category B, severity low, confidence confirmed
- Features: UTIL-28
- Document: `huawei_industry_tree/15_utilities/docs/28_date_conversion.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/date_conversion-0000002402521613
- Why: First entry shows a dangling sentence; large solar gaps silently leave the app's declared date domain.
- Fix: Compute in aboutToAppear; clamp the solar branch.

### `HW-15-0067` - RSA2's key pair is regenerated per app launch — ciphertext can't be decrypted after a restart

- Category B, severity low, confidence confirmed
- Features: UTIL-29
- Document: `huawei_industry_tree/15_utilities/docs/29_rsa_encryption_and_decryption.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/rsa_encryption_and_decryption-0000002465072997
- Why: Restart-then-decrypt fails with the generic error, inconsistent with the sibling mode.
- Fix: Persist the pair or document the scope.

### `HW-15-0082` - Tapping any participant deletes the LAST entry — down to deleting '我' while the UI still renders her and fakes the count

- Category B, severity low, confidence probable
- Features: UTIL-39
- Document: `huawei_industry_tree/15_utilities/docs/39_swiper_conference_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/swiper_conference_page-0000002538460457
- Why: Delete-to-empty leaves a self-contradictory meeting page.
- Fix: Splice by the tapped index; fix the empty branch.

### `HW-15-0091` - Pinch end sets the gesture state to 1 ('pinch') instead of -1 ('none') — rotation stays blocked until the finger re-presses (initial camera z also jumps 5→3)

- Category B, severity low, confidence confirmed
- Features: UTIL-44
- Document: `huawei_industry_tree/15_utilities/docs/44_rotate_model.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/rotate_model-0000002556688551
- Why: The state machine contradicts its documented meaning; a visible zoom jump on first interaction.
- Fix: Fix the constant; align z with orbitDistance_.

### `HW-15-0107` - The UIContext handle is decorated @State although it is a framework object that is never reassigned

- Category C, severity low, confidence confirmed
- Features: UTIL-13
- Document: `huawei_industry_tree/15_utilities/docs/13_input_method_application_immersive_mode.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/input_method_application_immersive_mode-0000002283469114
- Why: @State asks the framework to observe a variable for changes and to re-render the components that read it. UIContext is a handle onto the UI instance, not application state: it never changes, and there is nothing meaningful to observe inside it. Decorating it adds the observation wrapper and the dependency bookkeeping for a value that can never trigger an update, and it invites a reader to treat framework handles as state elsewhere.
- Fix: Change the decorator to a plain private field.

