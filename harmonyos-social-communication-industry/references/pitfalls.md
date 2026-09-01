# Pitfalls

> Generated from `features/*.md`. Source industry: `14_social_communication`, 45 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (15)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-14-0017` - Systematic: 'send success' toast fires before un-awaited, catch-less compression; JPEG bytes labeled data:image/png (two samples)

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-08, SOCIAL-39
- Document: `huawei_industry_tree/14_social_communication/docs/08_send_original_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/send_original_image-0000002249291536
- Why: A failed compression is an unhandled rejection with the success toast already shown — the message silently never appears; the payload MIME is mislabeled for any consumer that trusts it.
- Fix: await + .catch; use data:image/jpeg.

### `HW-14-0018` - Systematic: broken ForEach keys across chat samples — object-valued keys collapse to '[object Object]', content-only keys collide on duplicate sends (six samples)

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-08, SOCIAL-23, SOCIAL-25, SOCIAL-29, SOCIAL-39, SOCIAL-43
- Document: `huawei_industry_tree/14_social_communication/docs/08_send_original_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/send_original_image-0000002249291536
- Why: Duplicate keys break ArkUI diffing: second identical message not rendered or wrong node reused — on each sample's core send flow.
- Fix: Append index / use a message id.

### `HW-14-0028` - Systematic: dual message stores desync — timer-generated incoming messages never enter the @StorageLink('message') array (two samples sharing the MyChat template)

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-13, SOCIAL-40
- Document: `huawei_industry_tree/14_social_communication/docs/13_chat_unread_reminder.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_unread_reminder-0000002282675266
- Why: Incoming messages don't auto-scroll into view while the chat is open; unread bookkeeping runs on a store that never sees them.
- Fix: Push incoming messages into the shared array too.

### `HW-14-0085` - 3 sample projects swallow errors in catch blocks with an empty body

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-17, SOCIAL-21, SOCIAL-43
- Document: `huawei_industry_tree/14_social_communication/docs/43_chat_page_file_encryption.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_page_file_encryption-0000002424613385
- Why: A catch block with an empty body discards the error object and lets execution continue as if the operation had succeeded. The failure becomes invisible: no log, no user feedback, and no way to diagnose it from a released build. In several of these cases the guarded call is the feature the sample exists to demonstrate.
- Fix: Log the error with hilog including the BusinessError code and message, and surface a user-visible result where the operation was user initiated. Never leave the body empty.

### `HW-14-0088` - Systematic: 8 sample projects declare permissions that no code path in the project uses

- Category D, severity medium, confidence confirmed
- Features: SOCIAL-12, SOCIAL-20, SOCIAL-21, SOCIAL-22, SOCIAL-24, SOCIAL-25, SOCIAL-26, SOCIAL-44
- Document: `huawei_industry_tree/14_social_communication/docs/26_chat_page_location_navigation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_page_location_navigation-0000002322113637
- Why: A permission declared in module.json5 but never referenced by any .ets or .ts file in the project cannot be exercised, so it is pure over-declaration. Over-declared permissions are a release-review rejection reason, and restricted permissions especially so. Because these module.json5 files are copied as templates, the surplus entries propagate into products that have even less claim to them.
- Fix: Delete every declared permission the code does not exercise. Where the capability is reached through a permission-free path such as SaveButton or PhotoViewPicker, no declaration is needed at all.

### `HW-14-0001` - Systematic: four social project trees list files their zips do not contain

- Category E, severity low, confidence confirmed
- Features: SOCIAL-01, SOCIAL-04, SOCIAL-05, SOCIAL-29, SOCIAL-36, SOCIAL-44
- Document: `huawei_industry_tree/14_social_communication/docs/05_image_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_recognition-0000002236898070
- Why: Readers navigating by the documented structure hit nonexistent files; trees were not regenerated after renames.
- Fix: Regenerate the four 工程目录 blocks.

### `HW-14-0002` - Chat samples teach cleartext ws:// WebSocket endpoints (also in 44_web_socket_client2); the documented echo server is wss-only

- Category D, severity low, confidence confirmed
- Features: SOCIAL-20, SOCIAL-44
- Document: `huawei_industry_tree/14_social_communication/docs/20_weak_network_reconnection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/weak_network_reconnection-0000002288904746
- Why: An instant-messaging reliability template is exactly where transport security matters; teaching ws:// normalizes cleartext chat traffic, and the sample's own endpoint no longer accepts ws://.
- Fix: Switch the URLs to wss:// and mention certificate requirements.

### `HW-14-0003` - Systematic: copy-pasted permission config leaves unused declarations and dead location-permission constants across four social samples

- Category D, severity low, confidence confirmed
- Features: SOCIAL-01, SOCIAL-03, SOCIAL-33, SOCIAL-34, SOCIAL-42
- Document: `huawei_industry_tree/14_social_communication/docs/33_chat_multi_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_multi_selection-0000002332574910
- Why: Excess declared permissions violate least privilege, and dead location-permission constants in unrelated templates invite developers to wire them up; both stem from one copy-pasted template config.
- Fix: Delete the unused requestPermissions entries and the REQUEST_PERMISSIONS constants.

### `HW-14-0004` - Systematic: image-preview onAnimationEnd dereferences the size entry of a not-yet-loaded image (two samples)

- Category B, severity low, confidence probable
- Features: SOCIAL-02, SOCIAL-08
- Document: `huawei_industry_tree/14_social_communication/docs/02_image_preview.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_preview-0000002266277321
- Why: Fast swiping / large images crash the preview gesture handling.
- Fix: if (!this.imageListSize[index]) return;

### `HW-14-0008` - Systematic: permission-check loops return inside the first iteration (two social samples)

- Category B, severity low, confidence confirmed
- Features: SOCIAL-03, SOCIAL-17
- Document: `huawei_industry_tree/14_social_communication/docs/03_voice_to_text_forchat.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_to_text_forchat-0000002231520016
- Why: Generic multi-permission helpers report results based on one element only — wrong grants on any partial refusal.
- Fix: Return false in-loop, true after; AND-accumulate in the checker.

### `HW-14-0027` - Systematic: window 'keyboardHeightChange' listeners registered per component and never removed (three samples)

- Category B, severity low, confidence confirmed
- Features: SOCIAL-12, SOCIAL-16, SOCIAL-27, SOCIAL-35
- Document: `huawei_industry_tree/14_social_communication/docs/12_h5_interception.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5_interception-0000002282125390
- Why: Listeners accumulate per open/rebuild and keep firing into destroyed components for the window's lifetime.
- Fix: Store the callback; currentWindow.off.

### `HW-14-0031` - Systematic: isSelf toggled BEFORE sendMessage — the first sent message renders as the peer (two samples)

- Category B, severity low, confidence probable
- Features: SOCIAL-14, SOCIAL-42
- Document: `huawei_industry_tree/14_social_communication/docs/14_chat_reference.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_reference-0000002317503353
- Why: The user's first typed message appears as an incoming message on the left with the peer's avatar.
- Fix: Move the toggle below sendMessage().

### `HW-14-0033` - Contacts sorted by raw code points but grouped by pinyin — order inside letter groups is wrong

- Category B, severity low, confidence confirmed
- Features: SOCIAL-09, SOCIAL-15
- Document: `huawei_industry_tree/14_social_communication/docs/15_add_friends.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_friends-0000002283246714
- Why: Visible mis-ordering within groups whenever codepoint order differs from pinyin order.
- Fix: Sort on convertToPinyinString(name).

### `HW-14-0086` - 3 sample projects depend on third-party packages through unpinned version ranges

- Category B, severity low, confidence confirmed
- Features: SOCIAL-02, SOCIAL-09, SOCIAL-15
- Document: `huawei_industry_tree/14_social_communication/docs/15_add_friends.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_friends-0000002283246714
- Why: A caret or tilde range lets ohpm resolve a different version than the one the sample was verified against, so the published sample is not reproducible and can break without any change to its own source. Sample projects are reference implementations; their dependency set should be exactly the one that was tested.
- Fix: Pin exact versions in oh-package.json5 dependencies, and record the resolved set in oh-package-lock.json5.

### `HW-14-0087` - Systematic: 40 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: SOCIAL-02, SOCIAL-03, SOCIAL-04, SOCIAL-06, SOCIAL-07, SOCIAL-08, SOCIAL-09, SOCIAL-10, SOCIAL-11, SOCIAL-12, SOCIAL-13, SOCIAL-14, SOCIAL-15, SOCIAL-16, SOCIAL-18, SOCIAL-19, SOCIAL-20, SOCIAL-21, SOCIAL-22, SOCIAL-23, SOCIAL-24, SOCIAL-25, SOCIAL-26, SOCIAL-27, SOCIAL-28, SOCIAL-29, SOCIAL-31, SOCIAL-32, SOCIAL-33, SOCIAL-34, SOCIAL-35, SOCIAL-36, SOCIAL-37, SOCIAL-38, SOCIAL-39, SOCIAL-40, SOCIAL-41, SOCIAL-42, SOCIAL-43, SOCIAL-44
- Document: `huawei_industry_tree/14_social_communication/docs/15_add_friends.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_friends-0000002283246714
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (79)

### `HW-14-0009` - Preference matching is dead code: similarity is computed but never sorts or filters — the 'match' result is always the input list

- Category B, severity high, confidence confirmed
- Features: SOCIAL-04
- Document: `huawei_industry_tree/14_social_communication/docs/04_preference_search.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/preference_search-0000002236253766
- Why: The sample's core feature (根据偏好匹配好友) has no effect — changing preferences never changes the result.
- Fix: Sort userSimArray by sim desc before mapping.

### `HW-14-0016` - Deselecting one photo wipes the whole selection: pop() plus reassignment from the sent-message list

- Category B, severity high, confidence confirmed
- Features: SOCIAL-08
- Document: `huawei_industry_tree/14_social_communication/docs/08_send_original_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/send_original_image-0000002249291536
- Why: Deselect any image → imageUri becomes [] while the picker still shows others selected; Send then sends nothing (or stale base64 strings).
- Fix: Filter the selection itself.

### `HW-14-0037` - Bluetooth SPP sockets and read subscriptions are never closed or unsubscribed anywhere — plus duplicate sppRead listeners per id change

- Category B, severity high, confidence confirmed
- Features: SOCIAL-17
- Document: `huawei_industry_tree/14_social_communication/docs/17_bluetooth_spp.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bluetooth_spp-0000002318232397
- Why: Backing out leaves the RFCOMM client + listening server socket + subscription alive; re-entry stacks more (duplicate received messages); peer disconnect is never handled — socket lifecycle is the whole point of the sample.
- Fix: Add cleanup in onHidden; off before re-subscribing.

### `HW-14-0044` - The advertised weak-network auto-reconnect is dead code: `!this.ws` is never true after a drop and retryCount never resets on recovery

- Category B, severity high, confidence confirmed
- Features: SOCIAL-20
- Document: `huawei_industry_tree/14_social_communication/docs/20_weak_network_reconnection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/weak_network_reconnection-0000002288904746
- Why: Doc: '当检测到网络可用且WebSocket连接断开后，重新连接服务端' — never fires; after ~31 s of backoff the app is offline until restart.
- Fix: Track an isConnected flag; reset retryCount on recovery.

### `HW-14-0045` - Reconnect churn loop: the old socket's close handler re-arms a retry that tears down the new healthy connection, forever

- Category B, severity high, confidence confirmed
- Features: SOCIAL-20
- Document: `huawei_industry_tree/14_social_communication/docs/20_weak_network_reconnection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/weak_network_reconnection-0000002288904746
- Why: One heartbeat timeout puts the manager into permanent connect/close churn; the retry cap never engages because retryCount keeps resetting.
- Fix: clearTimeout in handleOpen; off listeners pre-close.

### `HW-14-0048` - The referenced audio worker files do not exist — all voice capture/render is dead

- Category B, severity high, confidence confirmed
- Features: SOCIAL-21
- Document: `huawei_industry_tree/14_social_communication/docs/21_voice_call.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_call-0000002328140805
- Why: Answering a call never records or plays audio, though the doc intro claims the sample implements capture/render with AudioCapturer/AudioRenderer.
- Fix: Add the worker sources or fix doc/code.

### `HW-14-0078` - QR save: fd closed in finally while the async packToFile is still writing (plus unhandled refusal path)

- Category B, severity high, confidence confirmed
- Features: SOCIAL-38
- Document: `huawei_industry_tree/14_social_communication/docs/38_customizable_qrcode.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/customizable_qrcode-0000002354787644
- Why: The saved QR image can be truncated/failed depending on timing while the late callback may still toast success; refusal gives no feedback.
- Fix: Move the close; widen the try; await.

### `HW-14-0005` - Drag-to-cancel never stops the speech recognizer — the mic stays live and text keeps arriving after cancel

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-03
- Document: `huawei_industry_tree/14_social_communication/docs/03_voice_to_text_forchat.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_to_text_forchat-0000002231520016
- Why: Cancelling a voice input leaves the engine listening; subsequent speech is transcribed into the input the user just cancelled.
- Fix: Call stop()/cancel in the cancel branch.

### `HW-14-0006` - Drag-to-cancel wipes the entire editor: deleteSpans() called with no range

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-03
- Document: `huawei_industry_tree/14_social_communication/docs/03_voice_to_text_forchat.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_to_text_forchat-0000002231520016
- Why: Type a draft, add voice, cancel — the whole draft is destroyed, not just the voice part.
- Fix: Pass the placeholder/transcription range.

### `HW-14-0007` - Stop path unconditionally deletes 3 characters to remove a '...' placeholder that is only inserted when the editor had content

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-03
- Document: `huawei_industry_tree/14_social_communication/docs/03_voice_to_text_forchat.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_to_text_forchat-0000002231520016
- Why: First use on an empty editor: releasing the talk button chops 3 characters off the head of the inserted transcription (or arbitrary text after the caret).
- Fix: Track a placeholderInserted flag.

### `HW-14-0010` - Similarity formula wrong twice: operator precedence multiplies by the second denominator, and centering subtracts the raw count instead of the mean

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-04
- Document: `huawei_industry_tree/14_social_communication/docs/04_preference_search.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/preference_search-0000002236253766
- Why: Even if the result were used, the computed value is not a normalized correlation.
- Fix: Parenthesize; center by the mean.

### `HW-14-0011` - Location subscription is never unsubscribed — GPS runs forever and each toggle cycle stacks another listener

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-06
- Document: `huawei_industry_tree/14_social_communication/docs/06_near_people.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/near_people-0000002237251858
- Why: Battery/privacy: positioning continues after toggle-off; duplicate callbacks accumulate per cycle.
- Fix: Store the callback and call geoLocationManager.off.

### `HW-14-0012` - Settings-dialog refusal leaves the location switch ON (unhandled rejection), and 'nearby users' generate around lat/lon 0 before the first fix

- Category B, severity medium, confidence probable
- Features: SOCIAL-06
- Document: `huawei_industry_tree/14_social_communication/docs/06_near_people.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/near_people-0000002237251858
- Why: Refusal boundary: switch stays ON without permission; first-run shows users positioned at null island with ~0.0x coordinates.
- Fix: catch the request; regenerate on first locationChange.

### `HW-14-0021` - Persistence built on `new UIContext()` — getPreferencesSync receives an invalid context

- Category B, severity medium, confidence probable
- Features: SOCIAL-10
- Document: `huawei_industry_tree/14_social_communication/docs/10_personal_info_upload.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/personal_info_upload-0000002290265961
- Why: The whole profile persistence (avatar/name/phone/signature/birthday) fails at the first instance() call.
- Fix: Pass the host context in from the page.

### `HW-14-0022` - Edit pages never prefill stored values — a bare Save wipes the field (and one-shot isSave blocks correction)

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-10
- Document: `huawei_industry_tree/14_social_communication/docs/10_personal_info_upload.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/personal_info_upload-0000002290265961
- Why: One accidental Save destroys existing data and the page cannot correct it in place.
- Fix: Load current values in aboutToAppear.

### `HW-14-0023` - Back gesture is dead app-wide: onBackPress/onBackPressed return true everywhere without acting

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-10
- Document: `huawei_industry_tree/14_social_communication/docs/10_personal_info_upload.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/personal_info_upload-0000002290265961
- Why: System back does nothing anywhere; sub-pages only exit via the on-screen arrow and the app cannot be left with the gesture.
- Fix: Remove the blanket true returns.

### `HW-14-0024` - Hand-rolled UTF-8 decoder drops all 4-byte sequences — every emoji vanishes from decrypted text

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-11
- Document: `huawei_industry_tree/14_social_communication/docs/11_encryp_chat.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/encryp_chat-0000002291169549
- Why: Supplementary-plane characters (all emoji) disappear — decrypt does not round-trip the plaintext. util.TextDecoder would be correct.
- Fix: Use util.TextDecoder.

### `HW-14-0025` - Decrypted JSON unwrapped with fixed slice(16, len-4) offsets — messages containing quotes/backslashes render corrupted; only span[0] is ever encrypted

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-11
- Document: `huawei_industry_tree/14_social_communication/docs/11_encryp_chat.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/encryp_chat-0000002291169549
- Why: Ordinary punctuation corrupts the displayed decryption; multi-span input is silently truncated.
- Fix: Parse instead of slicing; join span values.

### `HW-14-0026` - URL whitelist is a substring match — 'https://evil.com/www.huawei' bypasses the interception the sample exists to demonstrate

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-12
- Document: `huawei_industry_tree/14_social_communication/docs/12_h5_interception.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5_interception-0000002282125390
- Why: The doc's claim (only whitelisted sites accessible) is trivially defeated by embedding the token in a hostile URL's path/query.
- Fix: Parse the URL and match the host suffix.

### `HW-14-0038` - Receive path decodes with TextDecoder.create('"utf-8"') — quotes inside the encoding name

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-17
- Document: `huawei_industry_tree/14_social_communication/docs/17_bluetooth_spp.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bluetooth_spp-0000002318232397
- Why: Message reception throws a parameter error on the sample's receive path.
- Fix: Fix the literal.

### `HW-14-0039` - getPairedDevices called OUTSIDE its try/catch — refusal/Bluetooth-off throws uncaught from the toggle handler

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-17
- Document: `huawei_industry_tree/14_social_communication/docs/17_bluetooth_spp.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bluetooth_spp-0000002318232397
- Why: Enabling 'server listening' with Bluetooth off crashes out of onChange instead of the intended catch path.
- Fix: Move the call into the try.

### `HW-14-0040` - Image draft is saved but never restored — and the saved payload is irreversibly corrupted by UTF-8 decoding JPEG bytes

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-18
- Document: `huawei_industry_tree/14_social_communication/docs/18_save_draft_on_exit.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/save_draft_on_exit-0000002284059456
- Why: After a restart the image draft is gone; even a restore implementation could not reconstruct the bytes.
- Fix: Store base64; read it back in aboutToAppear.

### `HW-14-0041` - String draft's default value is the number 0 — first run renders '0' in the TextArea

- Category B, severity medium, confidence probable
- Features: SOCIAL-18
- Document: `huawei_industry_tree/14_social_communication/docs/18_save_draft_on_exit.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/save_draft_on_exit-0000002284059456
- Why: First run / after discard: the field shows '0' instead of the placeholder.
- Fix: Use '' as the default.

### `HW-14-0046` - Offline resend cache never drains: slice(index, index) is a no-op where splice(index, 1) was intended

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-20
- Document: `huawei_industry_tree/14_social_communication/docs/20_weak_network_reconnection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/weak_network_reconnection-0000002288904746
- Why: Every once-failed message is re-sent on EVERY reconnect — duplicates accumulate per reconnection.
- Fix: Copy + clear, or filter after send.

### `HW-14-0049` - Floating-window feature is fully dead: init never called and the target page doesn't exist

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-21
- Document: `huawei_industry_tree/14_social_communication/docs/21_voice_call.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_call-0000002328140805
- Why: The documented 悬浮窗控制类 can never run; minimize-to-float assets are unused.
- Fix: Call initWindowStage in onWindowStageCreate; add the page.

### `HW-14-0050` - Leaving without hanging up leaks the looping ringtone, timer, event subscriber and socket

- Category B, severity medium, confidence probable
- Features: SOCIAL-21
- Document: `huawei_industry_tree/14_social_communication/docs/21_voice_call.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_call-0000002328140805
- Why: Back-gesture during ringing keeps the ringtone looping from the background process.
- Fix: Run the hangUp cleanup from aboutToDisappear.

### `HW-14-0053` - onDestroy shuts down a fresh recognizer instance — the real ASR engine leaks (and CoreSpeechKit engines are exclusive)

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-23
- Document: `huawei_industry_tree/14_social_communication/docs/23_voice_to_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_to_text-0000002296107966
- Why: Guaranteed no-op cleanup; the actual engine leaks on ability destruction and can block re-creation (engine busy).
- Fix: Route shutdown to the real instance.

### `HW-14-0054` - Playing a second voice bubble stops the WRONG renderer: stopAndRelease acts on whatever instance is current

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-23
- Document: `huawei_industry_tree/14_social_communication/docs/23_voice_to_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_to_text-0000002296107966
- Why: Tap bubble A then B: both play simultaneously (A leaked); when A's file ends its callback stops/releases B mid-playback.
- Fix: Stop the old renderer before creating; capture the instance in the callback.

### `HW-14-0055` - Quick press-release leaves the microphone recording forever: capturer is created after the stop already ran

- Category B, severity medium, confidence probable
- Features: SOCIAL-23
- Document: `huawei_industry_tree/14_social_communication/docs/23_voice_to_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_to_text-0000002296107966
- Why: Cancel-mid-op: mic left capturing indefinitely, pcm file growing, fd never closed.
- Fix: Check a cancelled flag in the create callback.

### `HW-14-0056` - Recognition results routed through mutable cIndex — text can land on the wrong bubble (the correct eIndex is dead code)

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-23
- Document: `huawei_industry_tree/14_social_communication/docs/23_voice_to_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_to_text-0000002296107966
- Why: Convert bubble A, tap bubble B before callbacks finish → A's transcription appears on B.
- Fix: Use eIndex (capture the index at submission).

### `HW-14-0057` - Barcode decode pipeline: convertPixelFormat un-awaited AND NV12 converted but NV21 declared

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-24
- Document: `huawei_industry_tree/14_social_communication/docs/24_long_press_scan_code.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/long_press_scan_code-0000002297010246
- Why: The decoder receives a buffer matching neither declared size nor format — nondeterministic recognition failures surfaced only as a generic toast.
- Fix: await + align NV12/NV21.

### `HW-14-0059` - Image-emoji messages store time as 'H:MM' where ISO dates are parsed — Invalid Date breaks the time headers

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-25
- Document: `huawei_industry_tree/14_social_communication/docs/25_emoji_pack_recommended.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/emoji_pack_recommended-0000002330999945
- Why: Time headers around emoji messages are silently suppressed (NaN comparisons), or render '星期undefined NaN:NaN' when taken.
- Fix: Store toISOString() and format at render.

### `HW-14-0060` - All location cards share one MapComponent controller/options — camera and marker land on the wrong card's map

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-26
- Document: `huawei_industry_tree/14_social_communication/docs/26_chat_page_location_navigation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_page_location_navigation-0000002322113637
- Why: With 2+ location cards, earlier cards re-center to the newest location and a new card can stay stuck at the hardcoded TEMP coordinates.
- Fix: Build mapOptions per item; keep a controller per card.

### `HW-14-0061` - Keyboard listeners nest and multiply: a NEW keyboardHeightChange listener is registered inside every avoidAreaChange event, with conflicting offset math and no off anywhere

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-27
- Document: `huawei_industry_tree/14_social_communication/docs/27_chat_bubbles.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_bubbles-0000002332591285
- Why: Multiple callbacks race writing conflicting 'keyboardHeight' values — the input row's bottom padding depends on nondeterministic callback order, and listeners leak for the window's lifetime.
- Fix: Register once; off in aboutToDisappear.

### `HW-14-0064` - Tapping '+' wipes the sent photos BEFORE the picker opens, and cancel pops without a result — the photos are gone

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-29
- Document: `huawei_industry_tree/14_social_communication/docs/29_custom_album_style.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_album_style-0000002333520485
- Why: Open picker, press back: every photo previously in the chat is permanently gone.
- Fix: Move the clear into the onPop callback.

### `HW-14-0067` - Phone-number segmentation uses split() instead of entity offsets — repeated numbers corrupt the text (doc reproduces the same code); the destructive pass also breaks re-editing

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-31
- Document: `huawei_industry_tree/14_social_communication/docs/31_auto_phone_number_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/auto_phone_number_recognition-0000002317048484
- Why: A repeated phone number (normal chat pattern) corrupts rendering; done→re-edit→done loses the message body.
- Fix: Use result[i] position info; keep originalText immutable.

### `HW-14-0070` - Multi-select forward sorts numeric indices lexicographically — messages ≥ index 10 forward out of order

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-33
- Document: `huawei_industry_tree/14_social_communication/docs/33_chat_multi_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_multi_selection-0000002332574910
- Why: [2,10] sorts to [10,2]; the merged forwarded chat shows wrong chronological order.
- Fix: sort((a,b) => a-b).

### `HW-14-0071` - multiSelectIndex never cleared on exit — re-entering multi-select forwards stale, unchecked selections

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-33
- Document: `huawei_industry_tree/14_social_communication/docs/33_chat_multi_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_multi_selection-0000002332574910
- Why: Do-it-twice: users forward messages they never (visibly) selected.
- Fix: multiSelectIndex.clear() on mode exit.

### `HW-14-0073` - closeSync on a null-asserted file in finally kills the add-image path on any open failure

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-34
- Document: `huawei_industry_tree/14_social_communication/docs/34_drag_image_sort.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/drag_image_sort-0000002333934506
- Why: Open failure = unhandled rejection + lost image.
- Fix: if (file) closeSync.

### `HW-14-0074` - Multi-select order nondeterministic and the draggable 'add' tile corrupts the grid on drop

- Category B, severity medium, confidence probable
- Features: SOCIAL-34
- Document: `huawei_industry_tree/14_social_communication/docs/34_drag_image_sort.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/drag_image_sort-0000002333934506
- Why: The nine-grid starts in the wrong order and a stray drag of the add tile corrupts it.
- Fix: await per uri in order; guard itemIndex < length.

### `HW-14-0075` - Scroll thresholds compared against an accumulated delta that can never be positive — auto-scroll and auto-hide branches are dead (doc contradicted)

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-35
- Document: `huawei_industry_tree/14_social_communication/docs/35_latest_message.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/latest_message-0000002384193145
- Why: A user at the bottom gets the 'new message' pill instead of the documented auto-scroll; the pill only dismisses by clicking it.
- Fix: Track offset from the list end (or use scroller state).

### `HW-14-0076` - Tabs guards confuse child index with JSON index — selecting the first rendered tab breaks highlight and snaps content back

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-36
- Document: `huawei_industry_tree/14_social_communication/docs/36_quick_reply_in_chat.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/quick_reply_in_chat-0000002385043665
- Why: Switching back to the first tab leaves the highlight on the previous tab and content reverts after tapping a phrase.
- Fix: Drop the >0 guards (indices already shifted).

### `HW-14-0081` - Unsupported-share toast is invisible: the singleton captured 'currentUIContext' before EntryAbility stored it

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-41
- Document: `huawei_industry_tree/14_social_communication/docs/41_receive_image_share.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/receive_image_share-0000002405380589
- Why: Sharing an unsupported file type does nothing at all — no toast, no navigation.
- Fix: Fetch AppStorage inside the method.

### `HW-14-0082` - Picker cancel crashes the select flow, and the copy destination is opened without TRUNC (stale bytes on re-send)

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-43
- Document: `huawei_industry_tree/14_social_communication/docs/43_chat_page_file_encryption.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_page_file_encryption-0000002424613385
- Why: Cancel faults the flow; re-sent files can be silently corrupted by stale trailing bytes.
- Fix: Guard length; add OpenMode.TRUNC.

### `HW-14-0083` - The error handler cancels the just-scheduled reconnect — auto-reconnect defeats itself

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-44
- Document: `huawei_industry_tree/14_social_communication/docs/44_web_socket_client2.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_socket_client2-0000002550213863
- Why: On server-unreachable or mid-session drops the advertised 断线重连 never happens.
- Fix: Don't clear the timer in the error path.

### `HW-14-0089` - ChatPage has no aboutToDisappear, so the pending 200 ms timeout and the frame animator both outlive the page

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-39
- Document: `huawei_industry_tree/14_social_communication/docs/39_recent_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/recent_image-0000002355555898
- Why: The timeout is armed from the RecentPhotoComponent callback, which fires when the gallery reports a recent photo, and it is cleared only if the user then closes the function area. Navigating away from the chat within those 200 ms leaves the callback to run and assign to a destroyed component's state. The animator is worse: its onFrame closure holds both the component and the list scroller, so an animation still running when the page is popped keeps them alive and calls scrollEdge on a detached scroller. Both are the ordinary exit path from a chat screen, not an edge case.
- Fix: Add aboutToDisappear to ChatPage that calls clearTimeout(this.timeoutID) unconditionally and this.menuAnimator?.cancel(), and clear the onFrame reference.

### `HW-14-0090` - The avoidAreaChange listener registered in onWindowStageCreate is never released in onWindowStageDestroy

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-39
- Document: `huawei_industry_tree/14_social_communication/docs/39_recent_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/recent_image-0000002355555898
- Why: The listener is bound to the main window and outlives the window stage. Its closure holds the window, and relaunching within the same process registers another listener on top, so each avoid-area change runs the handler more than once. The value it maintains, bottomRectHeight, feeds the animator interpolation bounds in ChatPage, so duplicated updates are not inert.
- Fix: Keep the callback in a class field and call windowClass.off('avoidAreaChange', callback) in onWindowStageDestroy.

### `HW-14-0091` - The scroll-to-bottom timeout discards its handle and the page has no aboutToDisappear, so it cannot be cancelled and stacks on repeated focus

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-42
- Document: `huawei_industry_tree/14_social_communication/docs/42_chat_send_conntact.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_send_conntact-0000002412138797
- Why: onEditing runs from a @Watch on isFocus, so it fires every time the input gains focus. Each call schedules another 200 ms callback that captures the component and its ListScroller, and nothing ever cancels them. Focusing and unfocusing the message box a few times leaves several pending callbacks that all scroll the list, and leaving the chat within 200 ms runs scrollEdge against a scroller whose list is gone. Neither the id nor a cancellation path exists, so this cannot be fixed by the caller.
- Fix: Store the id in a private field, clearTimeout it at the top of onEditing before scheduling, and clear it again in a new aboutToDisappear.

### `HW-14-0092` - The avoidAreaChange listener registered in onWindowStageCreate is never released in onWindowStageDestroy

- Category B, severity medium, confidence confirmed
- Features: SOCIAL-42
- Document: `huawei_industry_tree/14_social_communication/docs/42_chat_send_conntact.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_send_conntact-0000002412138797
- Why: The listener outlives the window stage and its closure holds the window. Relaunching within the same process adds a second listener on top of the first, so every avoid-area change writes the AppStorage key more than once. Unlike some samples in this corpus the value here is genuinely consumed by the chat layout, so duplicated writes drive real re-layouts.
- Fix: Keep the callback in a class field and call windowClass.off('avoidAreaChange', callback) in onWindowStageDestroy.

### `HW-14-0093` - The contact book builds an alphabet-indexed address book from nested eager ForEach loops, with no LazyForEach, reuse or item caching

- Category C, severity medium, confidence confirmed
- Features: SOCIAL-42
- Document: `huawei_industry_tree/14_social_communication/docs/42_chat_send_conntact.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_send_conntact-0000002412138797
- Why: An AlphabetIndexer exists only because the list is too long to scroll by hand, so the construction is being demonstrated at a scale where eager building is the wrong choice. Nested ForEach constructs every group and every contact in every group up front, which is the cost HQ's own performance practice document (19_common_technical_solutions/docs/04_practice-common-app-performance-v1.md) requires long lists to avoid by combining LazyForEach, component reuse and list-item caching. scrollToIndex over an eagerly built list also has to walk constructed nodes rather than jumping.
- Fix: Convert listItemArr to an IDataSource and use LazyForEach for the outer grouping, mark the contact row component @Reusable, and set cachedCount on the List.

### `HW-14-0013` - Vote total incremented on selection clicks and never reflected in the rendered denominator

- Category B, severity low, confidence confirmed
- Features: SOCIAL-07
- Document: `huawei_industry_tree/14_social_communication/docs/07_vote_result_display.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vote_result_display-0000002274416885
- Why: Vote math drifts arbitrarily with selection clicks while the progress ratios never change denominator.
- Fix: Move the increment to the vote action and update candidate.total.

### `HW-14-0014` - Six attributes receive raw resource-name strings instead of $r() — width/fontWeight silently invalid

- Category B, severity low, confidence confirmed
- Features: SOCIAL-07
- Document: `huawei_industry_tree/14_social_communication/docs/07_vote_result_display.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vote_result_display-0000002274416885
- Why: Header columns lose their full-width/center layout; the settings are ignored.
- Fix: Add $r().

### `HW-14-0015` - Doc snippet calls this.dialogControllerList.open() — no such dialog exists anywhere in the sample

- Category D, severity low, confidence confirmed
- Features: SOCIAL-07
- Document: `huawei_industry_tree/14_social_communication/docs/07_vote_result_display.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/vote_result_display-0000002274416885
- Why: Readers look for a result dialog the sample never opens.
- Fix: Fix the snippet.

### `HW-14-0019` - pickerOptions.preselectedUris is dead: the object is never passed to the PhotoPickerComponent (and copies the wrong list)

- Category B, severity low, confidence confirmed
- Features: SOCIAL-08
- Document: `huawei_industry_tree/14_social_communication/docs/08_send_original_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/send_original_image-0000002249291536
- Why: The preselection feature has no effect; even wired up it would inject invalid uris.
- Fix: Pass this.pickerOptions to the component; copy imageUri.

### `HW-14-0020` - ContactDetail reads its navigation param from stack index 0 instead of the top entry

- Category B, severity low, confidence probable
- Features: SOCIAL-09
- Document: `huawei_industry_tree/14_social_communication/docs/09_contracts.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/contracts-0000002289306481
- Why: Any second push (rapid double-tap) shows the first entry's contact on the top page.
- Fix: getParamByIndex(size()-1) or getParamByName.

### `HW-14-0029` - Tab bar neutralized: empty onClick consumes taps while scrollable(false) blocks swipes (ChatFoldTop variant: only tab 0 clickable)

- Category B, severity low, confidence probable
- Features: SOCIAL-13
- Document: `huawei_industry_tree/14_social_communication/docs/13_chat_unread_reminder.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_unread_reminder-0000002282675266
- Why: Tab switching is impossible (or limited to tab 0); visible buttons give no response.
- Fix: Wire changeIndex(button.index) unconditionally.

### `HW-14-0030` - 'Mark unread' state is overwritten by the next recompute — the manual badge silently disappears/miscounts

- Category B, severity low, confidence confirmed
- Features: SOCIAL-13
- Document: `huawei_industry_tree/14_social_communication/docs/13_chat_unread_reminder.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_unread_reminder-0000002282675266
- Why: After marking unread, one new message shows badge 1 instead of 2 and recomputes clear the manual mark.
- Fix: Flip the last message to SENT (or preserve the flag).

### `HW-14-0032` - closeSelectionMenu called on a RichEditorController never attached to any component — the reference menu is not dismissed

- Category B, severity low, confidence probable
- Features: SOCIAL-14
- Document: `huawei_industry_tree/14_social_communication/docs/14_chat_reference.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_reference-0000002317503353
- Why: After tapping 引用 the custom selection menu stays on screen.
- Fix: Use a TextController bound to that Text.

### `HW-14-0034` - Submit sends empty bubbles: no content guard in onSubmit

- Category B, severity low, confidence confirmed
- Features: SOCIAL-16
- Document: `huawei_industry_tree/14_social_communication/docs/16_drop_to_send_image_and_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/drop_to_send_image_and_text-0000002283583388
- Why: Pressing enter on an empty input litters the chat with empty bubbles.
- Fix: Guard on trimmed text.

### `HW-14-0035` - @StorageProp px values converted to vp once and then clobbered back to raw px on the next store write

- Category B, severity low, confidence confirmed
- Features: SOCIAL-16
- Document: `huawei_industry_tree/14_social_communication/docs/16_drop_to_send_image_and_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/drop_to_send_image_and_text-0000002283583388
- Why: After the first avoid-area event the padding silently becomes ~3x too large.
- Fix: Convert in the binding expression, not by overwriting the prop.

### `HW-14-0036` - onDrop consumes only record[0] — the rest of a multi-item drag is silently discarded

- Category B, severity low, confidence confirmed
- Features: SOCIAL-16
- Document: `huawei_industry_tree/14_social_communication/docs/16_drop_to_send_image_and_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/drop_to_send_image_and_text-0000002283583388
- Why: Dragging several images sends only the first.
- Fix: Loop over getRecords().

### `HW-14-0042` - deepCopy's object branch calls obj.length() — throws on any non-array object

- Category B, severity low, confidence confirmed
- Features: SOCIAL-19
- Document: `huawei_industry_tree/14_social_communication/docs/19_group_solitaire.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/group_solitaire-0000002320400581
- Why: Any reuse with objects crashes the advertised generic helper.
- Fix: Rewrite the object branch.

### `HW-14-0043` - onReady hardcodes isChange = true — Send is active before any edit and can publish the '用户' placeholder

- Category B, severity low, confidence confirmed
- Features: SOCIAL-19
- Document: `huawei_industry_tree/14_social_communication/docs/19_group_solitaire.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/group_solitaire-0000002320400581
- Why: The user can immediately publish a chain whose only new entry is the literal placeholder.
- Fix: Drop the hardcoded true.

### `HW-14-0047` - Heartbeat control messages fall through into chat-message parsing and are emitted as bogus events

- Category B, severity low, confidence confirmed
- Features: SOCIAL-20
- Document: `huawei_industry_tree/14_social_communication/docs/20_weak_network_reconnection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/weak_network_reconnection-0000002288904746
- Why: Every heartbeat emits a garbage 'websocketMessage' (msgID 'ping'/'pong'); any consumer beyond Chat.ets would process corrupted data.
- Fix: Add the return; catch the send.

### `HW-14-0051` - lastConnected never resets — one disconnect re-fires the whole teardown every 3 s; call timer mixes UTC and local time

- Category B, severity low, confidence confirmed
- Features: SOCIAL-21
- Document: `huawei_industry_tree/14_social_communication/docs/21_voice_call.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_call-0000002328140805
- Why: Repeated teardown storms after a drop; wrong call duration in half-hour-offset timezones.
- Fix: Set lastConnected=false in onDisconnected; use getUTCMinutes/Seconds.

### `HW-14-0052` - HomePage initializes a detached `new LocalStorage()` that no @LocalStorageLink ever reads

- Category B, severity low, confidence confirmed
- Features: SOCIAL-22
- Document: `huawei_industry_tree/14_social_communication/docs/22_chat_web_float.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_web_float-0000002329723933
- Why: Any non-default initial value (or future writes through this.localStorage) silently never reaches the UI.
- Fix: Bind via @Entry(storage) or drop the private instance.

### `HW-14-0058` - No-code-detected path: scanResults[0] unchecked — leaks the PixelMap/ImageSource and shows a raw error

- Category B, severity low, confidence probable
- Features: SOCIAL-24
- Document: `huawei_industry_tree/14_social_communication/docs/24_long_press_scan_code.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/long_press_scan_code-0000002297010246
- Why: Every no-barcode attempt leaks native objects and leaves the sheet open with an internal message.
- Fix: Check length; release in finally.

### `HW-14-0062` - List initialIndex computed from msgNums which is still 0 — initialIndex is -1 on first build

- Category B, severity low, confidence confirmed
- Features: SOCIAL-27
- Document: `huawei_industry_tree/14_social_communication/docs/27_chat_bubbles.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_bubbles-0000002332591285
- Why: The documented start-at-last-message wiring is invalid (-1).
- Fix: Use textDetailData.totalCount() - 1.

### `HW-14-0063` - Empty-state view is dead code: isShowMessage unconditionally true; bulk add notifies only one item

- Category B, severity low, confidence confirmed
- Features: SOCIAL-28
- Document: `huawei_industry_tree/14_social_communication/docs/28_app_message_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_message_list-0000002332602789
- Why: First-run/empty boundary shows a blank list instead of the designed view; any LazyForEach listener sees N-1 missing items on bulk adds.
- Fix: Derive from totalCount(); notifyDataAdd per index.

### `HW-14-0065` - Album picker checks the box immediately while the async asset query has no .catch (and FetchResult never closed)

- Category B, severity low, confidence confirmed
- Features: SOCIAL-29
- Document: `huawei_industry_tree/14_social_communication/docs/29_custom_album_style.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_album_style-0000002333520485
- Why: UI and data diverge on transient media-library failures; native resources leak per pick.
- Fix: then/catch around createSelectPhotoInfo; fetchResult.close().

### `HW-14-0066` - Inertial-scroll clamp inverts when the image is shorter than the view: minTranslateY not clamped to <= 0

- Category B, severity low, confidence probable
- Features: SOCIAL-30
- Document: `huawei_industry_tree/14_social_communication/docs/30_inertial_sliding.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/inertial_sliding-0000002308946264
- Why: The first pan shoves the image downward off its anchored top position.
- Fix: Clamp the minimum to zero.

### `HW-14-0068` - Popup borderRadius set to the string resource 'description'

- Category B, severity low, confidence confirmed
- Features: SOCIAL-31
- Document: `huawei_industry_tree/14_social_communication/docs/31_auto_phone_number_recognition.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/auto_phone_number_recognition-0000002317048484
- Why: Invalid Length; the intended rounded corners are silently dropped.
- Fix: Point at the correct resource.

### `HW-14-0069` - Search results render an invisible full-width clickable Row for EVERY message; isSearched latches on the first keystroke

- Category B, severity low, confidence probable
- Features: SOCIAL-32
- Document: `huawei_industry_tree/14_social_communication/docs/32_seek_chat_history.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/seek_chat_history-0000002351789373
- Why: Tapping blank space below the results jumps the chat to a message the user never selected.
- Fix: Filter the data before ForEach.

### `HW-14-0072` - Delete popup item and enabled trash icon have no onClick — inert UI presented as functional

- Category B, severity low, confidence confirmed
- Features: SOCIAL-33
- Document: `huawei_industry_tree/14_social_communication/docs/33_chat_multi_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_multi_selection-0000002332574910
- Why: Tapping delete does nothing and the popup does not close.
- Fix: Wire delete or drop the controls.

### `HW-14-0077` - Three emitter.on subscriptions never off'd — page rebuilds double-process emoji clicks

- Category B, severity low, confidence confirmed
- Features: SOCIAL-37
- Document: `huawei_industry_tree/14_social_communication/docs/37_emoji_comment.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/emoji_comment-0000002351840732
- Why: Emoji comments counted twice after a relaunch/back-navigation rebuild.
- Fix: emitter.off for the three events.

### `HW-14-0079` - Doc's pinned-list visibility condition is inverted relative to the shipped code

- Category D, severity low, confidence confirmed
- Features: SOCIAL-40
- Document: `huawei_industry_tree/14_social_communication/docs/40_chat_fold_top.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_fold_top-0000002405205461
- Why: Copying the doc yields inverted fold behavior.
- Fix: Fix the snippet.

### `HW-14-0080` - Deleting a session never stops its message timer — the deleted contact keeps generating messages forever

- Category B, severity low, confidence confirmed
- Features: SOCIAL-40
- Document: `huawei_industry_tree/14_social_communication/docs/40_chat_fold_top.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_fold_top-0000002405205461
- Why: Unbounded growth and stale lastMessage/reminders for deleted contacts.
- Fix: clearInterval for the contact's timer.

### `HW-14-0084` - Client/server heartbeat protocol mismatch — the handshake works only by side effect; UI logs messages as sent when the socket is null

- Category B, severity low, confidence confirmed
- Features: SOCIAL-44
- Document: `huawei_industry_tree/14_social_communication/docs/44_web_socket_client2.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_socket_client2-0000002550213863
- Why: The designed request/response keepalive never runs; after a drop users see 'sent' messages that were never transmitted.
- Fix: Answer heartbeat_request with heartbeat_response; gate the log on send success.

### `HW-14-0094` - The published URL slug misspells contact as conntact

- Category E, severity low, confidence confirmed
- Features: SOCIAL-42
- Document: `huawei_industry_tree/14_social_communication/docs/42_chat_send_conntact.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_send_conntact-0000002412138797
- Why: The slug is the stable identifier the page is linked and searched by, and it is the one place the word is misspelled: the title, the sample project name and the body all spell it correctly. A reader searching for the correct spelling does not reach the page, and any hand-written cross reference to it will be typed correctly and break. This is the third slug defect found in this corpus, after 11_news_reading/docs/24_auto_flip_read.md and 13_media_entertainment/docs/40_audio-v1_2-ts_64.md, both of which carry slugs describing a different subject.
- Fix: Republish under chat_send_contact and redirect the misspelled slug.

