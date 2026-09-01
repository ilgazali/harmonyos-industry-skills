# Pitfalls

> Generated from `features/*.md`. Source industry: `05_office`, 32 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (2)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-05-0184` - 2 sample projects swallow errors in catch blocks with an empty body

- Category B, severity medium, confidence confirmed
- Features: OFFICE-05, OFFICE-11
- Document: `huawei_industry_tree/05_office/docs/11_conference_release.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/conference_release-0000002321751725
- Why: A catch block with an empty body discards the error object and lets execution continue as if the operation had succeeded. The failure becomes invisible: no log, no user feedback, and no way to diagnose it from a released build. In several of these cases the guarded call is the feature the sample exists to demonstrate.
- Fix: Log the error with hilog including the BusinessError code and message, and surface a user-visible result where the operation was user initiated. Never leave the body empty.

### `HW-05-0185` - Systematic: 25 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: OFFICE-03, OFFICE-04, OFFICE-06, OFFICE-10, OFFICE-11, OFFICE-12, OFFICE-13, OFFICE-14, OFFICE-15, OFFICE-16, OFFICE-17, OFFICE-18, OFFICE-19, OFFICE-20, OFFICE-21, OFFICE-22, OFFICE-23, OFFICE-24, OFFICE-25, OFFICE-26, OFFICE-27, OFFICE-28, OFFICE-29, OFFICE-30, OFFICE-31
- Document: `huawei_industry_tree/05_office/docs/24_app_watermark.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_watermark-0000002353053774
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (184)

### `HW-05-0129` - The speech-recognition audio path is never connected: setDataCallback targets a second, never-started AudioCapturer, and the capturer's readData handler never invokes the stored callback, so writeAudio is never called

- Category B, severity blocker, confidence confirmed
- Features: OFFICE-23
- Document: `huawei_industry_tree/05_office/docs/23_voice_input_notes.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_input_notes-0000002351441712
- Why: writeAudio is the only place audio reaches the recognition engine, and it sits inside a callback that nothing ever runs. The engine is created, the listener is set and startListening is issued, so the UI shows a live session - but no PCM is ever fed to it, and neither generatedText nor recognitionResult can ever be populated. That is the entire feature the document describes.
- Fix: Share one AudioCapturer - inject the instance the UI starts into SpeechRecognizer rather than constructing a second one - and make the capturer actually dispatch: call 'this.dataCallBack?.(buffer);' inside readDataCallback so the registered consumer receives every buffer.

### `HW-05-0001` - The sample project calls reminderAgentManager.publishReminder but never declares the mandatory ohos.permission.PUBLISH_AGENT_REMINDER permission in module.json5

- Category D, severity high, confidence confirmed
- Features: OFFICE-01
- Document: `huawei_industry_tree/05_office/docs/01_practice-office-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1-0000001965211649
- Why: ohos.permission.PUBLISH_AGENT_REMINDER is a normal-level permission that must be declared in requestPermissions of the HAP module.json5. Without the declaration the publishReminder call fails with permission-denied, so the background reminder feature that the document advertises as its industry innovation never works.
- Fix: Add {"name": "ohos.permission.PUBLISH_AGENT_REMINDER", "reason": "$string:reason", "usedScene": {"abilities": ["EntryAbility"], "when": "inuse"}} to requestPermissions in entry/src/main/module.json5, and document it in the permission list of the guide.

### `HW-05-0023` - saveImage closes the sandbox file twice and dereferences a null File in the finally block when openSync fails

- Category B, severity high, confidence confirmed
- Features: OFFICE-05
- Document: `huawei_industry_tree/05_office/docs/05_personal_card.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/personal_card-0000002236270234
- Why: Two defects share one line. On the success path the descriptor closed at :41 is closed a second time at :73. On the failure path - openSync throwing, which is exactly what the catch exists for - 'file' is still the null value it was initialised with at :35, so 'file.fd' in the finally raises a TypeError that propagates out of saveImage and replaces the original, already-logged error. Saving the QR code is the primary function of this page, so the user sees neither the success toast at :61 nor any diagnosis.
- Fix: Close the descriptor once, in the finally only, and guard it: remove fs.closeSync(file.fd) at :41 and write 'finally { if (file) { fs.closeSync(file); } }'. The official save-user-file example closes each handle exactly once.

### `HW-05-0030` - writeToFile closes the destination file in its finally block while the asynchronous fs.copyFile is still running, so the copy writes to a closed descriptor

- Category B, severity high, confidence confirmed
- Features: OFFICE-06
- Document: `huawei_industry_tree/05_office/docs/06_document_approval.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/document_approval-0000002280673593
- Why: fs.copyFile returns a Promise, so the try block completes as soon as the copy is scheduled and the finally runs immediately - before the copy has written anything. The descriptor handed to the copy is therefore closed underneath it. Three further consequences follow from the same shape: the .then() has no .catch(), the synchronous catch can never observe an asynchronous rejection, and the download_fail toast at :68 is therefore unreachable while the download_suc toast at :65 fires regardless of whether any bytes arrived.
- Fix: Make the function async and await the copy before releasing the handle: 'try { await fs.copyFile(filePath, file.fd); showToast(download_suc); } catch (error) { showToast(download_fail); Logger.error(...); } finally { fs.closeSync(file); }'.

### `HW-05-0037` - Every diagnostic in CameraShooter.ets uses hilog domain -1, which is outside the documented [0x0, 0xFFFF] range, so none of those logs are printed

- Category A, severity high, confidence confirmed
- Features: OFFICE-07
- Document: `huawei_industry_tree/05_office/docs/07_camera_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/camera_page-0000002245028040
- Why: -1 is outside [0x0, 0xFFFF], so by the documented behaviour these lines print nothing. Every failure path of the camera pipeline - getSupportedCameras, createCameraInput, cameraInput.open, createPreviewOutput, createPhotoOutput, createSession, beginConfig, addInput, addOutput, commitConfig, start - reports exclusively through these calls and swallows its BusinessError afterwards without rethrowing, so a camera that fails to initialise produces no diagnostic at all. The same calls additionally place the message in the tag slot and an empty string in the format slot, which is the second half of the misuse.
- Fix: Use a valid domain constant and the documented argument order, e.g. 'const DOMAIN = 0x0000;' then 'hilog.error(DOMAIN, "CameraDemo", "cameraManager is undefined");' and 'hilog.error(DOMAIN, "CameraDemo", "createCameraInput error: %{public}s", JSON.stringify(err));'.

### `HW-05-0038` - releaseCamera is async and never awaited, so the camera pipeline is rebuilt while the previous session, input and outputs are still being torn down

- Category B, severity high, confidence confirmed
- Features: OFFICE-07
- Document: `huawei_industry_tree/05_office/docs/07_camera_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/camera_page-0000002245028040
- Why: cameraShooting proceeds immediately to camera.getCameraManager, createCameraInput and cameraInput.open on the next lines while the previous input close and session release are still in flight, so the new input can be opened against a device the old input has not finished closing. The same applies on page hide followed by a quick re-entry, where onHidden's teardown overlaps onShown's rebuild at CameraPage.ets:200. The five inner calls being unawaited also means each release rejection is an unhandled promise rejection.
- Fix: await each release inside releaseCamera ('await photoSession.stop(); await cameraInput.close(); await previewOutput.release(); await photoSession.release(); await photoOutPut.release();') inside a try/catch, and await releaseCamera() at CameraShooter.ets:46 before building the new pipeline.

### `HW-05-0045` - viewTXT, viewWord and viewExcel call the asynchronous myRawfileCopy and then preview the file on the very next synchronous line

- Category B, severity high, confidence confirmed
- Features: OFFICE-08
- Document: `huawei_industry_tree/05_office/docs/08_file_view.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/file_view-0000002256155688
- Why: viewFile immediately calls filePreview.canPreview on a sandbox path whose content is still being produced by the callback, so the preview races the copy. The copy also re-opens the file with fs.OpenMode.CREATE | fs.OpenMode.READ_WRITE and no TRUNC (FileViewUtil.ets:38), so a partially rewritten file can retain a stale tail from the previous copy while the previewer is reading it.
- Fix: Make myRawfileCopy return a Promise - either await context.resourceManager.getRawFileContent(filetype) in its promise form, or wrap the callback - and await it before calling viewFile. Add fs.OpenMode.TRUNC so the rewrite cannot leave a stale tail.

### `HW-05-0063` - readWriteFile is called with the bare attachment file name instead of the sandbox destination path, so the file is never written to the URI that is previewed on the next line

- Category B, severity high, confidence confirmed
- Features: OFFICE-11
- Document: `huawei_industry_tree/05_office/docs/11_conference_release.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/conference_release-0000002321751725
- Why: attachmentName is a bare file name with no directory, while attachmentUri - built two lines earlier from this.filesDir - is the sandbox path the preview call then resolves with fileUri.getUriFromPath. The copy therefore targets a different location from the one that is opened, so the previewer is handed a path that was never written. The document states the intent explicitly at line 37: "在readWriteFile函数中创建同名沙箱文件和拷贝附件内容" ("create a same-named sandbox file and copy the attachment content in the readWriteFile function").
- Fix: Pass the destination that was just computed: 'readWriteFile(srcDir, this.meetingFormData.attachmentUri);'.

### `HW-05-0071` - The stamp handler releases the document and reloads the output path even when loadDocument failed or saveDocument returned false, pointing the viewer at a file that was never written

- Category B, severity high, confidence confirmed
- Features: OFFICE-12
- Document: `huawei_industry_tree/05_office/docs/12_pdf_add_mark.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pdf_add_mark-0000002324100453
- Why: outPdfPath is reassigned to a brand-new timestamped path on line 118, before anything has been written there. If loadDocument does not return PARSE_SUCCESS the whole stamping block is skipped, yet lines 128-129 still release the document and reload the viewer from that non-existent path, so the correctly displayed original disappears. The same happens when saveDocument returns false: its result is logged but never acted on, isMarked is still set to true on line 126, and the user is told the stamp succeeded.
- Fix: Move the release and reload inside the success branch and gate them on the save result: 'if (res === PARSE_SUCCESS) { ...; const result = this.pdfDocument.saveDocument(this.outPdfPath); this.pdfDocument.releaseDocument(); if (result) { this.isMarked = true; this.reloadDocument(this.outPdfPath); } else { show an error toast } } else { this.pdfDocument.releaseDocument(); show an error toast }'.

### `HW-05-0090` - Cancelling the picker overwrites the already-selected ID photo with undefined, and the add icon has already been hidden so there is no control left to re-pick

- Category B, severity high, confidence confirmed
- Features: OFFICE-16
- Document: `huawei_industry_tree/05_office/docs/16_recommend_id_photos.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/recommend_id_photos-0000002344341185
- Why: Dismissing the system photo picker rejects the promise, so imagePicker returns undefined and the assignment clears the state. On the first attempt this erases the ic_front_id_card placeholder; after a successful pick it erases the chosen photo, and because frontAddIcon was set to ImageContent.EMPTY on that earlier success the add button is now an empty image, leaving the user with a blank card and nothing to tap. The guard only ever hides the add icon - it never restores it.
- Fix: Assign only on success and restore the affordance otherwise: 'const picked = await this.imagePicker(RecommendationType.ID_CARD); if (picked !== undefined) { this.frontIdCard = picked; this.frontAddIcon = ImageContent.EMPTY; }'. Also guard the index - 'const uris = (await photoPicker.select(option)).photoUris; result = uris.length > 0 ? uris[0] : undefined;'.

### `HW-05-0093` - The tree is built by pushing into the children arrays of exported module-level nodes, so entering the page a second time duplicates every child

- Category B, severity high, confidence confirmed
- Features: OFFICE-17
- Document: `huawei_industry_tree/05_office/docs/17_multi_level_nesting_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/multi_level_nesting_list-0000002311817336
- Why: hashMap is rebuilt per component instance but the Node objects themselves are the module-level singletons, and their children arrays are never cleared before the push. aboutToAppear therefore appends the same child a second time on the next visit, so 开发部 and 测试部 each appear twice under 研发部门, their own children appear twice, and the selection counters in checkParent - which compare selectedChildren against children.length - are computed against a doubled length.
- Fix: Reset the relationships before building them, or build the tree once outside the component: clear each node's children and parent at the top of aboutToAppear ('data.forEach((n) => { n.children = []; n.parent = null; })'), or move the tree construction into MockData so it runs once at module load.

### `HW-05-0097` - releaseCamera awaits none of the five asynchronous releases it performs, so createRecorder rebuilds the pipeline while the previous one is still tearing down

- Category B, severity high, confidence confirmed
- Features: OFFICE-18
- Document: `huawei_industry_tree/05_office/docs/18_conference_window_change.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/conference_window_change-0000002325708930
- Why: The await on an async function that never awaits its own work returns as soon as the four release calls are scheduled. createRecorder therefore opens a new camera input while the previous input is still closing and the previous session is still being released, on a device where the camera is an exclusive resource. The comment at WindowContentComponent.ets:69 shows the author already found this fragile - both the main and the sub window release once each to try to force the resource free.
- Fix: Await every release inside a try/catch: 'await this.videoSession?.stop(); await this.cameraInput?.close(); for (const preview of this.previewOutputs) { await preview.release(); } this.previewOutputs = []; await this.videoSession?.release();'.

### `HW-05-0103` - releaseCamera awaits none of its five releases and is itself called without await at the top of cameraShooting

- Category B, severity high, confidence confirmed
- Features: OFFICE-19
- Document: `huawei_industry_tree/05_office/docs/19_watermark_camera.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/watermark_camera-0000002334789536
- Why: Two omissions compound. Even if the caller awaited it, the method resolves as soon as the five releases are scheduled; and cameraShooting does not await it at all. The new camera input is therefore opened while the previous input is still closing and the previous session is still being released, on a device where the camera is an exclusive resource - which is exactly what happens on every front/rear switch, since the switch re-enters cameraShooting.
- Fix: Await each release inside a try/catch and await the call: 'await this.photoSession?.stop(); await this.cameraInput?.close(); await this.previewOutput?.release(); await this.photoSession?.release(); await this.photoOutPut?.release();' and 'await this.releaseCamera();' at CameraUtils.ets:64.

### `HW-05-0104` - createPreviewOutput and createPhotoOutput are passed profiles that find() returns undefined for whenever the hard-coded size and format are not in the device capability list

- Category A, severity high, confidence confirmed
- Features: OFFICE-19
- Document: `huawei_industry_tree/05_office/docs/19_watermark_camera.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/watermark_camera-0000002334789536
- Why: The code's own comment says the profile depends on the device model, and the declared type says the search can fail, yet the failure is never handled. On any device whose previewProfiles or photoProfiles do not contain that exact width, height and format combination, find returns undefined and it is passed as a mandatory Profile argument. The guard placed after each call cannot help, because the bad argument has already been supplied.
- Fix: Check each search result before using it and fall back to a supported entry: 'if (!previewProfile || !photoProfile) { Logger.error("no matching profile on this device"); return; }' before the two factory calls, or select from cameraOutputCap by nearest supported size rather than by exact equality.

### `HW-05-0105` - saveToFile keeps its file descriptor in a module-level variable that is never reset, guards it with a check that skips descriptor 0, and closes it without awaiting

- Category B, severity high, confidence confirmed
- Features: OFFICE-19
- Document: `huawei_industry_tree/05_office/docs/19_watermark_camera.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/watermark_camera-0000002334789536
- Why: Three defects share these lines. Because fd is module state and is never cleared, a second save whose createAsset or open throws enters the finally with the previous invocation's descriptor still in fd and closes an already-closed descriptor. Because the guard is 'if (fd)' rather than a null check, a legitimate descriptor value of 0 is treated as absent and never closed, leaking it. And because the close is not awaited, saveToFile resolves before the descriptor is released, while its caller in the SaveButton handler immediately continues.
- Fix: Make the descriptor local and null-check it explicitly: 'let file: fileIo.File | undefined = undefined; try { file = await fileIo.open(filePath, mode); await fileIo.truncate(file.fd); await fileIo.write(file.fd, imageBuffer); } catch (err) { ... } finally { if (file !== undefined) { fileIo.closeSync(file); } }'.

### `HW-05-0111` - NoteCanvas unsubscribes windowSizeChange with no callback argument, which cancels every subscription on that window including the ability's breakpoint updater

- Category B, severity high, confidence confirmed
- Features: OFFICE-20
- Document: `huawei_industry_tree/05_office/docs/20_note_with_camera.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/note_with_camera-0000002378064505
- Why: The reference is explicit that omitting the callback cancels all subscriptions to that event, and both the component and the ability subscribe to windowSizeChange on the window returned by getLastWindow. Destroying the note canvas therefore also removes EntryAbility's listener, so after that point the breakpoint stored in AppStorage under currentBreakpoint stops updating and every responsive layout in the app freezes at whatever value it last held.
- Fix: Keep the callback in a field and pass it to off so only this component's subscription is removed: 'this.sizeChangeCallback = (windowSize: window.Size) => { ... }; windowClass.on("windowSizeChange", this.sizeChangeCallback);' and 'windowClass.off("windowSizeChange", this.sizeChangeCallback);'.

### `HW-05-0116` - The recording file is closed twice - once from the capturer's STATE_RELEASED handler and again in stopAndRelease - and the fd field is initialised to 0 so a stop before any recording closes descriptor 0

- Category B, severity high, confidence confirmed
- Features: OFFICE-21
- Document: `huawei_industry_tree/05_office/docs/21_voice_insert_note.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_insert_note-0000002348459564
- Why: release() drives the capturer to STATE_RELEASED, which fires the stateChange handler and closes the file; stopAndRelease then closes the very same descriptor a second time on the next line. Separately, because fd is initialised to 0 rather than left undefined, the 'this.fd !== undefined' guard passes before any recording has been created, so calling stopAndRelease first closes file descriptor 0. Neither close is awaited in the stateChange handler either - fileIo.close returns a Promise.
- Fix: Close in one place only. Drop the close from the stateChange handler, declare the field as 'private fd?: number = undefined;', and keep the guarded close in stopAndRelease - or keep the handler and remove the close from stopAndRelease.

### `HW-05-0117` - All four AudioRenderer state guards have empty bodies, so start, pause, stop and release are issued from any state

- Category B, severity high, confidence confirmed
- Features: OFFICE-21
- Document: `huawei_industry_tree/05_office/docs/21_voice_insert_note.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_insert_note-0000002348459564
- Why: Each condition enumerates exactly the states from which the following transition is legal, and each body is empty - the throw or early return that belonged there has been removed. Every one of the four methods therefore issues its transition unconditionally: start() on an already-running renderer, pause() on a stopped one, stop() on a prepared one, release() on a running one. The play/pause toggle in the UI calls startRenderer and pauseRenderer directly on each tap, so these paths are reachable by ordinary interaction.
- Fix: Restore the guards, mirroring the STATE_INVALID branches directly above them: 'if (state !== audio.AudioState.STATE_PREPARED && state !== audio.AudioState.STATE_STOPPED && state !== audio.AudioState.STATE_PAUSED) { hilog.warn(...); return; }' and the equivalent in pauseRenderer, stopRenderer and releaseRenderer.

### `HW-05-0118` - stopRenderer closes the playback file without clearing playFile, so replaying the same note reads from a closed descriptor

- Category B, severity high, confidence confirmed
- Features: OFFICE-21
- Document: `huawei_industry_tree/05_office/docs/21_voice_insert_note.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_insert_note-0000002348459564
- Why: After stopRenderer the File object survives with the same path, so the 'this.playFile?.path !== filePath' test is false on the next startRenderer for that recording and the reopen is skipped. The writeData callback then calls readSync on a descriptor that was closed, which the catch at AudioRendererManager.ets:83-86 converts into a thrown Error from inside the audio callback. Replaying the same voice note after stopping it is the ordinary path through this feature.
- Fix: Clear the field when the file is closed: 'fileIo.closeSync(this.playFile); this.playFile = undefined;' in stopRenderer, so the next startRenderer reopens it.

### `HW-05-0122` - BreakpointSystem.unregister creates six fresh media-query handles and calls off on those, leaving the six listeners created by register subscribed

- Category B, severity high, confidence confirmed
- Features: OFFICE-21
- Document: `huawei_industry_tree/05_office/docs/21_voice_insert_note.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_insert_note-0000002348459564
- Why: matchMediaSync returns a listening handle for the condition; unregister discards the handles that register subscribed on and calls off on brand-new ones, which have no callback attached. The six subscriptions created by register therefore survive, and because the fields are reassigned there is no longer any reference through which they could be removed. Every register/unregister cycle adds six more live media-query listeners, each holding the BreakpointSystem instance through its bound arrow function.
- Fix: Unsubscribe from the stored handles and clear them: 'this.smListener?.off("change", this.isBreakpointSM); this.smListener = null;' and the same for the other five, with no matchMediaSync call in unregister at all.

### `HW-05-0123` - The microphone permission is only ever requested through requestPermissionOnSetting, which the reference says may not be called before requestPermissionsFromUser

- Category D, severity high, confidence confirmed
- Features: OFFICE-21
- Document: `huawei_industry_tree/05_office/docs/21_voice_insert_note.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_insert_note-0000002348459564
- Why: ohos.permission.MICROPHONE is a user_grant permission and the reference states plainly that the settings dialog may only be raised after the ordinary authorization dialog has been shown. By skipping the first stage entirely the sample never presents the standard prompt, so on a first run the user is sent to the Settings dialog for a permission they were never asked for - and the recording feature, which is the whole scenario, cannot be authorised through the normal flow. Every sibling sample in this industry performs the first-stage request first; see the two-stage ladder in ConferenceWindowChange.zip#entry/src/main/ets/utils/PermissionsUtils.ets.
- Fix: Add the first stage and only escalate afterwards: call atManager.requestPermissionsFromUser(context, ['ohos.permission.MICROPHONE']), inspect data.authResults, and call requestPermissionOnSetting only when the system reports that no dialog was shown.

### `HW-05-0124` - Each contact checkbox hard-codes select(false) instead of deriving its state from the selection model, and the select-all checkbox has no handler at all

- Category C, severity high, confidence confirmed
- Features: OFFICE-22
- Document: `huawei_industry_tree/05_office/docs/22_special_following.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/special_following-0000002382515945
- Why: The checkbox state is write-only: onChange pushes into selectedData, but nothing ever reads selectedData back into the control. Re-entering the page after confirming a set of follows shows every row unchecked even though myFollows holds them, so the user cannot see or amend an existing selection. The select-all checkbox is rendered next to its label and does nothing when tapped, which is worse than omitting it.
- Fix: Bind the state to the model - 'Checkbox().select(this.selectedData.includes(data))' or a $$ two-way binding onto a per-contact selected flag - seed selectedData from myFollows in aboutToAppear, and give the select-all checkbox an onChange that sets selectedData to all contacts or clears it.

### `HW-05-0130` - stop() shuts the recognition engine down without clearing the reference or finishing the session, so a second recording starts on a destroyed engine

- Category B, severity high, confidence confirmed
- Features: OFFICE-23
- Document: `huawei_industry_tree/05_office/docs/23_voice_input_notes.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_input_notes-0000002351441712
- Why: shutdown destroys the engine, and the sample keeps using the same reference afterwards. The second recording therefore calls startListening on a destroyed engine and lands on exactly the error the file documents at :156-159, with the failure reaching only the onError log. The session is also never finished, so whatever the engine had buffered is discarded rather than flushed as a final result.
- Fix: Finish the session and drop the reference: 'stop() { this.asrEngine?.finish(this.sessionId); this.asrEngine?.shutdown(); this.asrEngine = undefined; }', and have startRecording re-create the engine through createByCallback when asrEngine is undefined.

### `HW-05-0131` - The recording file is closed twice - from the capturer's STATE_RELEASED handler and again in stopAndRelease - and the fd field is initialised to 0 so a stop before any recording closes descriptor 0

- Category B, severity high, confidence confirmed
- Features: OFFICE-23
- Document: `huawei_industry_tree/05_office/docs/23_voice_input_notes.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_input_notes-0000002351441712
- Why: release() drives the capturer to STATE_RELEASED, which fires the stateChange handler and closes the file; stopAndRelease then closes the very same descriptor again two lines later. Separately, because fd is initialised to 0 rather than left undefined, the 'this.fd !== undefined' guard passes before any recording exists, so calling stopAndRelease first closes file descriptor 0. Neither close is awaited in the stateChange handler - fileIo.close returns a Promise.
- Fix: Close in one place only: drop the close from the stateChange handler, declare 'private fd?: number = undefined;', and keep the guarded close in stopAndRelease.

### `HW-05-0144` - The batch-sync loop runs even when the calendar permission was refused, and the optional chain then discards every event with no feedback

- Category B, severity high, confidence confirmed
- Features: OFFICE-27
- Document: `huawei_industry_tree/05_office/docs/27_multi_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/multi_schedule-0000002356954036
- Why: When the user refuses the calendar permission, flag is false, calendarMgr stays null, and 'this.calendarMgr?.getCalendar(...)' short-circuits for every selected item - so the whole batch is silently dropped. The addEvent helper does contain a !userGrant branch that shows a permission toast, but control never reaches it because the optional chain returns first. The user presses sync, the selection UI is dismissed at Index.ets:228-229, and nothing tells them the events were not created.
- Fix: Return early when the permission was refused, and let the existing helper report it: 'if (!flag) { this.uiContext.getPromptAction().showToast({ message: $r("app.string.need_permission") }); return; } this.userGrant = true; this.calendarMgr = calendarManager.getCalendarManager(context);' before the loop.

### `HW-05-0145` - Syncing N to-dos performs N full calendar reads: the loop resolves the calendar and enumerates every existing event once per selected item, then resolves the calendar again inside addEvent

- Category C, severity high, confidence confirmed
- Features: OFFICE-27
- Document: `huawei_industry_tree/05_office/docs/27_multi_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/multi_schedule-0000002356954036
- Why: getEvents with no filter returns every event in the calendar, and the sample calls it once per selected to-do purely to test for a duplicate - so selecting ten items reads the entire calendar ten times and resolves the calendar handle twenty times. The document presents this as the batch-synchronisation scenario (待办事项批量同步至日历), which is exactly the case where the per-item cost matters. The duplicate scan is also O(N x M) over the JavaScript side on top of the I/O.
- Fix: Resolve the calendar once and enumerate once before the loop: 'const calendar = await this.calendarMgr.getCalendar(); const existing = await calendar.getEvents(); const seen = new Set(existing.map((e) => `${e.title}|${e.startTime}`));' then inside the loop test the Set and call calendar.addEvent directly, passing the resolved calendar into the helper instead of the manager.

### `HW-05-0148` - Delete-all-annotations reads counts that only the Save button writes, so freshly added annotations cannot be deleted and the app reports that there are none

- Category B, severity high, confidence confirmed
- Features: OFFICE-28
- Document: `huawei_industry_tree/05_office/docs/28_add_delete_annotations_in_pdf_preview.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_delete_annotations_in_pdf_preview-0000002384668918
- Why: Add a highlight and press 删除所有批注 without pressing 保存 first: the database still holds manualAnnNumber = 0 for every page, delAllAnn returns false, nothing is deleted, and the user is told there is nothing to delete while the highlight is plainly visible on screen. The in-memory manualAnnNumRecord is also left untouched in that path (the reset at PdfKitAbility.ets:195 sits inside the `manualAnnNumber !== 0` branch), so a later Save inflates the stored count by annotations that are still present, and the next delete pass removes the wrong number of annotations.
- Fix: In delAllAnn, compute the number of annotations to remove on each page as `dbManualAnnNumber + manualAnnNumRecord[pageIndex]`, delete that many starting at rawAnnNumber, then write manualAnnNumber = 0 and reset manualAnnNumRecord[pageIndex] = 0 for every page that had a non-zero total. Alternatively persist the count from registerAnnotationChangedListener as soon as an annotation is added, so that the database always reflects what is on screen.

### `HW-05-0149` - Four of the five relationalStore ResultSet instances are never closed, leaking file descriptors and memory on every page render, save and startup

- Category B, severity high, confidence confirmed
- Features: OFFICE-28
- Document: `huawei_industry_tree/05_office/docs/28_add_delete_annotations_in_pdf_preview.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_delete_annotations_in_pdf_preview-0000002384668918
- Why: Every save and every startup opens result sets that are never released. The reference explicitly names FD leaks as a consequence, and file descriptors are a per-process limit: a long editing session that repeatedly saves a multi-page document will keep accumulating cursors until queries start failing. The sample proves the author knew the rule, because delAllAnn closes its cursor with an explanatory comment while the four other query sites do not.
- Fix: Wrap each query in try/finally and call resultSet.close() in the finally block, for example in getAnnNumOfEachPage: `let resultSet = await store.query(predicates, [...]); try { if (resultSet.goToNextRow()) { return; } } finally { resultSet.close(); }`. Apply the same to printResultSet, to the inner query at line 93 and to the query at line 220 in saveAllAnnotation.

### `HW-05-0155` - The caller-info extension keeps its query result in instance fields that are never reset, so a call from an unknown number can be answered with the employee identity matched for the previous call

- Category B, severity high, confidence probable
- Features: OFFICE-29
- Document: `huawei_industry_tree/05_office/docs/29_call_identity_delivery_employee_info.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/call_identity_delivery_employee_info-0000002419483593
- Why: onQueryCallerInfo is a method on a long-lived ExtensionAbility instance, so two calls arriving while the extension is still alive share the same fields. The first call from a number that is in the EMPLOYEE table sets isSuccess = true and fills the four strings; the next call from a number that is not in the table finds no rows, changes nothing, and the promise still resolves - the system dialer then labels the unknown caller with the previous employee's name, staff number, department and position. That is both a wrong result on the incoming-call screen and a disclosure of one employee's details on another person's call. Marked probable because the reference page for CallerInfoQueryExtensionAbility (documentation/harmonyos-references/06_application-services/callservicekit-callerinfoquery-extension-ability.md) is a stub in this repository and does not state how long the extension instance is retained; the defect in the code - result state that is written but never cleared - is not in question.
- Fix: Make queryData return the row it found instead of writing to fields: `private async queryEmployee(phoneNumber: string): Promise<CallerInfo | undefined> { ... }`, then `const info = await this.queryEmployee(phoneNumber); if (info) { return info; } throw { code: 10001, msg: 'query fail' };`. If the fields must stay, clear all five at the top of onQueryCallerInfo before querying.

### `HW-05-0156` - The CREATE TABLE promise is not awaited before the EMPLOYEE table is queried and inserted into, and the query rejection is never caught

- Category B, severity high, confidence confirmed
- Features: OFFICE-29
- Document: `huawei_industry_tree/05_office/docs/29_call_identity_delivery_employee_info.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/call_identity_delivery_employee_info-0000002419483593
- Why: On a fresh install nothing guarantees that the table exists by the time the query runs. If the query wins the race it rejects with a no-such-table error that nobody catches, the async callback aborts before the seeding insert at EntryAbility.ets:106, and the app starts with an empty EMPLOYEE table - at which point the whole feature silently does nothing, because the extension can only answer calls for numbers this insert was supposed to add. The failure is invisible: no toast, no log at the call site, only an unhandled rejection.
- Fix: Replace the fire-and-forget chain with `try { await store.executeSql(SQL_CREATE_TABLE); await this.queryData(store); } catch (err) { hilog.error(DOMAIN, 'testTag', 'Failed to init store. Code:%{public}d, message:%{public}s', (err as BusinessError).code, (err as BusinessError).message); return; }` and keep the seeding insert after it.

### `HW-05-0170` - The error path of ScheduleUtils.addEvent returns a promise whose executor never resolves, so a failed system-calendar write hangs the save flow forever instead of reporting an error

- Category B, severity high, confidence confirmed
- Features: OFFICE-31
- Document: `huawei_industry_tree/05_office/docs/31_schedule_share.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule_share-0000002573150445
- Why: Any error thrown by Calendar.addEvent - a revoked permission, a malformed event, a full calendar - takes this branch. The .then callback then never runs: no failure toast, no local save, and RouterModule.closeDialog is never reached, so the new-schedule dialog stays on screen with no feedback and the user can only press it again. Because the returned promise is never rejected either, no upstream catch can recover. Note that the log message in this catch also says 'Failed to update event' inside addEvent, so the one diagnostic that does fire names the wrong operation.
- Fix: Return the value directly instead of a hand-built promise: `} catch (e) { hilog.error(0x0000, TAG, 'Failed to add event. Code: %{public}d, message: %{public}s', (e as BusinessError).code, (e as BusinessError).message); return undefined; }` - an async function wraps that in a resolved promise automatically.

### `HW-05-0171` - initSysCalender is declared async but never awaits the permission and getCalendar chain, and its only caller does not await it either, so the first saved schedule is written before the system calendar handle exists

- Category B, severity high, confidence confirmed
- Features: OFFICE-31
- Document: `huawei_industry_tree/05_office/docs/31_schedule_share.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule_share-0000002573150445
- Why: On the first run the user has to answer a permission dialog between aboutToAppear and pressing Save, and the getCalendar round trip follows it, so ScheduleUtils.calendar is undefined for the whole of that window. `calendar?.addEvent` then short-circuits to undefined instead of writing anything, addEvent resolves undefined, and NewSchedule.save at NewSchedule.ets:75-84 treats it as a failed sync but still stores the schedule locally - the schedule exists in the app and not in the system calendar, which is the one thing 场景介绍 promises ("实现应用日程与系统日历无损同步，确保日程提醒零遗漏" - "lossless synchronisation between app schedules and the system calendar, ensuring that no reminder is missed"). The same handle is also required by deleteEvent and updateEvent, which optional-chain it away just as silently.
- Fix: Rewrite with await: `public static async initSysCalender(): Promise<void> { if (ScheduleUtils.calendar) { return; } const result = await atManager.requestPermissionsFromUser(GlobalRegister.getContext(), permissions); if (!result.authResults.every(r => r === abilityAccessCtrl.GrantStatus.PERMISSION_GRANTED)) { return; } const calendarMgr = calendarManager.getCalendarManager(GlobalRegister.getContext()); ScheduleUtils.calendar = await calendarMgr.getCalendar(); }` and have addEvent do `await ScheduleUtils.initSysCalender();` before using the handle, throwing a typed error when it is still missing.

### `HW-05-0002` - Reminder.ets ignores the authResults returned by requestPermissionsFromUser and performs calendar read/write even when the user denies the permission

- Category D, severity medium, confidence confirmed
- Features: OFFICE-01
- Document: `huawei_industry_tree/05_office/docs/01_practice-office-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1-0000001965211649
- Why: requestPermissionsFromUser resolves successfully even when the user denies the request; the grant state is carried only in PermissionRequestResult.authResults. Skipping that check makes the app run the whole Calendar Kit chain (getCalendar / createCalendar / addEvent) against a denied permission, producing unexplained error logs instead of a user-visible authorization prompt.
- Fix: Inspect result.authResults after the promise resolves and only continue with calendarManager.getCalendarManager when every entry equals 0 (PERMISSION_GRANTED); otherwise guide the user to Settings as the official sample does.

### `HW-05-0003` - The call.makeCall callback dereferences err.message without checking whether err is set, so a successful call crashes the callback

- Category B, severity medium, confidence confirmed
- Features: OFFICE-01
- Document: `huawei_industry_tree/05_office/docs/01_practice-office-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1-0000001965211649
- Why: In an AsyncCallback the error argument is undefined when the operation succeeds. Reading err.message unconditionally throws a TypeError inside the callback on the normal success path, which is exactly why the official reference guards the access with if (err).
- Fix: Guard the callback: call.makeCall(this.phone.toString(), (err: BusinessError) => { if (err) { hilog.error(...); } }); or switch to the promise overload and handle the failure in .catch().

### `HW-05-0004` - BusinessPage computes an alarm hour that can reach 24, which is outside the [0, 23] range required by ReminderRequestAlarm

- Category B, severity medium, confidence confirmed
- Features: OFFICE-01
- Document: `huawei_industry_tree/05_office/docs/01_practice-office-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1-0000001965211649
- Why: hours comes from Date.getHours() and can be 23; Math.floor(minutes / 58) is 1 when the current minute is 58 or 59. Between 23:58 and 23:59 the computed hour is 24, which violates the documented [0, 23] range, so publishReminder rejects the request and the to-do reminder is silently lost.
- Fix: Normalise the roll-over, e.g. const total = hours * 60 + minutes + 2; addReminder({ id: 10, hour: Math.floor(total / 60) % 24, minute: total % 60 }).

### `HW-05-0005` - Five empty catch blocks in the framework code swallow NavDestination parameter and startAbility errors without any logging

- Category B, severity medium, confidence confirmed
- Features: OFFICE-01
- Document: `huawei_industry_tree/05_office/docs/01_practice-office-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1-0000001965211649
- Why: WebPage.ets:88 parses the navigation parameter with JSON.parse(this.params.params) and ConversationDetail.ets:78 casts ctx.pathInfo.param; when the parameter is missing or malformed the page silently keeps its default title and no diagnostic is produced. The same file set already uses hilog elsewhere (e.g. CalendarPage.ets:228), so the silent variants are an inconsistency inside the framework code that this document ships as the reference implementation.
- Fix: Log the caught error with hilog.error and fall back to a defined default, mirroring the CalendarPage.ets:227-229 pattern already used in the same project.

### `HW-05-0007` - All three LazyForEach lists build their keys with JSON.stringify(item), the exact pattern the official Code Linter performance rule forbids

- Category C, severity medium, confidence confirmed
- Features: OFFICE-01
- Document: `huawei_industry_tree/05_office/docs/01_practice-office-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1-0000001965211649
- Why: JSON.stringify runs on the UI thread for every visible and cached item on every rebuild. The official performance rule marks it as the cause of frame loss while scrolling and prescribes a stable unique id instead. The document presents this project as the office-industry reference framework, so the anti-pattern propagates.
- Fix: Give ConversationDataInterface, ToDoInfo and EmailInfo a stable id field and use it as the key, e.g. '(item: ConversationDataInterface) => item.id'. Only fall back to index when no natural id exists.

### `HW-05-0008` - The 日历项 code snippet in the document uses identifiers that do not exist (this.List, CalendarInfoList) and contradicts both its own declarations and the sample project

- Category E, severity medium, confidence confirmed
- Features: OFFICE-01
- Document: `huawei_industry_tree/05_office/docs/01_practice-office-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1-0000001965211649
- Why: ArkTS is case sensitive. As printed, the snippet references an undeclared array CalendarInfoList and writes to a member this.List that the following ForEach never reads, so a developer who copies the document verbatim gets a compile error and an empty list.
- Fix: Correct the snippet to 'this.list = calendarInfoList.filter(v => v.date === this.currentDate);' and add the '@State list: CalendarInfo[] = [];' declaration, matching CalendarPage.ets:31 and :57.

### `HW-05-0012` - The 后台提醒服务 section documents only the Calendar Kit route, while the sample additionally implements reminders through reminderAgentManager, which the document never mentions

- Category E, severity medium, confidence confirmed
- Features: OFFICE-01
- Document: `huawei_industry_tree/05_office/docs/01_practice-office-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1-0000001965211649
- Why: The 实现方案 subsection is the document's complete statement of how the innovation is built and which permissions it needs. Because it omits the agent-powered reminder path, the permission list is incomplete (see HW-05-0001) and a reader cannot reproduce the to-do alarm that the sample fires on entering the business tab.
- Fix: Extend the 实现方案 subsection with the agent-powered reminder path: describe reminderAgentManager.publishReminder, the ohos.permission.PUBLISH_AGENT_REMINDER declaration, and the notificationManager.requestEnableNotification prerequisite that EntryAbility.ets:35-38 already performs.

### `HW-05-0013` - requestPermissionOnSetting is called with the raw result of getHostContext(), which is typed Context | undefined, while the API documents context as a mandatory Context

- Category A, severity medium, confidence confirmed
- Features: OFFICE-03
- Document: `huawei_industry_tree/05_office/docs/03_location_permissions.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/location_permissions-0000002231804582
- Why: getHostContext() may return undefined, so its raw result does not satisfy the mandatory Context parameter. The document reproduces the same uncast call at line 39, so a reader copying either the guide or the sample writes an argument whose type does not match the published signature.
- Fix: Cast the context exactly as the first call in the same file and the official guide do: 'await atManager.requestPermissionOnSetting(context.getHostContext() as common.UIAbilityContext, permissions);'.

### `HW-05-0014` - The check-out record copies this.address before the asynchronous reverse-geocoding callback has written it, so it stores the check-in address or an empty string

- Category B, severity medium, confidence confirmed
- Features: OFFICE-03
- Document: `huawei_industry_tree/05_office/docs/03_location_permissions.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/location_permissions-0000002231804582
- Why: getLocation() returns immediately; the position arrives through a callback and the place name through a further promise. Reading this.address on the next statement therefore captures the value from the previous check-in - and on the very first tap, the empty initial value declared at CheckInPage.ets:32. The check-out row rendered at CheckInPage.ets:166 then shows the wrong location, which defeats the purpose of an attendance record.
- Fix: Make the address resolution awaitable (return the promise chain from getLocation/convertLatToPosition) and assign this.lastAddress inside its resolution handler, after this.address has been updated.

### `HW-05-0015` - EntryAbility subscribes to window avoidAreaChange but never unsubscribes in onWindowStageDestroy

- Category B, severity medium, confidence confirmed
- Features: OFFICE-03
- Document: `huawei_industry_tree/05_office/docs/03_location_permissions.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/location_permissions-0000002231804582
- Why: The callback keeps a reference to the window object and writes into AppStorage for the lifetime of the process. Because on() is called every time a window stage is created and off() is never called, repeated create/destroy cycles accumulate live subscriptions on the same window - the register/unregister pair the reference defines is left incomplete.
- Fix: Keep the callback in a field and call windowClass.off('avoidAreaChange', callback) in onWindowStageDestroy, or call windowClass.off('avoidAreaChange') there to drop every registration.

### `HW-05-0016` - The 实现思路 text says the settings dialog is opened when the user refuses authorization, but the sample opens it when no dialog was shown at all

- Category E, severity medium, confidence confirmed
- Features: OFFICE-03
- Document: `huawei_industry_tree/05_office/docs/03_location_permissions.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/location_permissions-0000002231804582
- Why: The two conditions are not the same. When the user denies the first dialog, dialogShownResults[0] is true and the sample does not open the settings dialog; it opens it only when the system showed no dialog because the permission was already set. Following the document's wording produces a settings dialog on the very first denial, which is not the behaviour the sample implements and not what the reference prescribes.
- Fix: Rewrite the step as: call requestPermissionsFromUser first; if data.dialogShownResults[0] is false - meaning the system did not show a dialog because the permission has already been set - call requestPermissionOnSetting to guide the user to Settings. Also mention the checkPermissions/checkAccessToken pre-check that PermissionsRequest.ets:25 performs before either request.

### `HW-05-0018` - Two hilog calls in EntryAbility pass the message as the tag and the serialized error as the format string, so the placeholder is never resolved

- Category A, severity medium, confidence confirmed
- Features: OFFICE-04
- Document: `huawei_industry_tree/05_office/docs/04_load_display_pdf.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/load_display_pdf-0000002270470565
- Why: With only three arguments the message text lands in the tag slot - where it is truncated at 31 bytes - and JSON.stringify(err) becomes the format string. The %{public}s placeholder therefore never expands, and any % character inside the serialized error is interpreted as a format specifier. The failure path of getMainWindow, which is the entry point of the whole anti-screenshot feature, becomes unreadable in the log.
- Fix: Pass all four arguments, e.g. hilog.error(0x0000, 'testTag', 'Failed to obtain the main window. Cause: %{public}s', JSON.stringify(err)); matching EntryAbility.ets:58 in the same file.

### `HW-05-0019` - setWindowPrivacyMode is called without try/catch and its promise is discarded, so a failed anti-screenshot activation is completely silent

- Category B, severity medium, confidence confirmed
- Features: OFFICE-04
- Document: `huawei_industry_tree/05_office/docs/04_load_display_pdf.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/load_display_pdf-0000002270470565
- Why: This call is the entire anti-screenshot control for a page whose stated purpose is secure preview. Because neither the synchronous exception nor the promise rejection is handled, a 201 permission failure or a 1300002 abnormal-window state leaves the PDF fully screenshot-able while the UI gives no indication at all. The document reproduces the same unguarded call at line 56.
- Fix: Follow the reference example: wrap the call in try/catch and handle the promise, e.g. this.windowClass.setWindowPrivacyMode(true).then(() => {...}).catch((err: BusinessError) => hilog.error(...)); and surface the failure to the user before showing the document.

### `HW-05-0022` - ohos.permission.CAMERA is declared and documented although the sample's only camera use is the Scan Kit default UI, for which the guide states the permission must not be requested

- Category D, severity medium, confidence confirmed
- Features: OFFICE-05
- Document: `huawei_industry_tree/05_office/docs/05_personal_card.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/personal_card-0000002236270234
- Why: Requesting a user_grant permission that the feature does not need is an over-declaration: it adds an authorization prompt the user cannot relate to any function, and the declare-permissions rules require the reason to match the actual permission function. Declaring it with "when": "always" widens the request further, since the scan UI is a foreground-only interaction.
- Fix: Remove the ohos.permission.CAMERA entry from entry/src/main/module.json5 and delete the 权限说明 section of the document, or replace it with a note that the default scanning UI needs no application-side camera permission. If a custom scanning UI is ever adopted (scan-customscan-api), declare CAMERA then with "when": "inuse".

### `HW-05-0024` - The destination media file is also closed twice - once by fd inside the try and once by File object in the finally

- Category B, severity medium, confidence confirmed
- Features: OFFICE-05
- Document: `huawei_industry_tree/05_office/docs/05_personal_card.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/personal_card-0000002236270234
- Why: closeSync accepts either the FD or the File object, so the two calls at :60 and :66 target the same open descriptor. The guard 'if (imageFile1)' cannot help because imageFile1 is always truthy once openSync returned - it only protects against openSync having thrown, which happens outside this try block anyway.
- Fix: Drop fileIo.closeSync(imageFile1.fd) at :60 and keep only the finally-block close, so the descriptor is released exactly once on both the success and the failure path.

### `HW-05-0025` - showAssetsCreationDialog is chained with .then() and no .catch(), so a user who dismisses the save dialog produces an unhandled rejection

- Category B, severity medium, confidence confirmed
- Features: OFFICE-05
- Document: `huawei_industry_tree/05_office/docs/05_personal_card.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/personal_card-0000002236270234
- Why: The dialog is the user-authorization step of the save flow, and dismissing it is an ordinary outcome rather than an exceptional one. Without a rejection handler that outcome surfaces as an unhandled promise rejection, the user gets no feedback at all, and the sandbox file written moments earlier is left behind.
- Fix: Follow the official example: await showAssetsCreationDialog inside the existing try/catch, or attach a .catch((err: BusinessError) => ...) that logs the code and shows the failure toast the page already defines as $r('app.string.save_error').

### `HW-05-0026` - The barcode-scanning failure path is three empty blocks, so every scan error including user cancellation is discarded silently

- Category B, severity medium, confidence confirmed
- Features: OFFICE-05
- Document: `huawei_industry_tree/05_office/docs/05_personal_card.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/personal_card-0000002236270234
- Why: Scan Kit reports outcomes such as the user cancelling the default scanning UI through this rejection path, and the code explicitly branches on scanCore.ScanErrorCode.INTERNAL_ERROR only to do nothing in either branch. The scan button therefore appears dead whenever anything goes wrong, and no code or message reaches the log.
- Fix: Call the existing showError(error, this.uiContext) helper in the catch and log the code with hilog.error, mirroring what MainPage.ets:124-127 already does for createBarcode.

### `HW-05-0027` - EntryAbility subscribes to avoidAreaChange without a matching off in onWindowStageDestroy

- Category B, severity medium, confidence confirmed
- Features: OFFICE-05
- Document: `huawei_industry_tree/05_office/docs/05_personal_card.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/personal_card-0000002236270234
- Why: The subscription is created for each window stage and is never released, so the register/unregister pair defined by the reference is left open and the callback keeps writing into AppStorage for the whole process lifetime.
- Fix: Store the callback in a field and call windowClass.off('avoidAreaChange', callback) in onWindowStageDestroy.

### `HW-05-0031` - Both request.downloadFile calls omit the .catch() that the official example attaches, so a failed download request is an unhandled rejection

- Category B, severity medium, confidence confirmed
- Features: OFFICE-06
- Document: `huawei_industry_tree/05_office/docs/06_document_approval.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/document_approval-0000002280673593
- Why: downloadFile rejects when the request itself cannot be created - no network, a malformed URL, an unwritable sandbox path. In ApprovalPage the call sits inside a try/catch (ApprovalPage.ets:50-79), but a synchronous catch cannot observe an asynchronous rejection, so the Logger.error at :78 never runs. In FileComponent there is no try at all. The user taps the document, nothing happens, and nothing is logged.
- Fix: Attach '.catch((err: BusinessError) => Logger.error(...))' to both downloadFile chains, exactly as the reference example does, and surface a failure toast.

### `HW-05-0032` - The download task subscribes to 'complete' without ever unsubscribing and never subscribes to 'fail', so a failed download is silent

- Category B, severity medium, confidence confirmed
- Features: OFFICE-06
- Document: `huawei_industry_tree/05_office/docs/06_document_approval.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/document_approval-0000002280673593
- Why: Two defects share the same registration. Without on('fail') a download that starts but does not finish - a server error, a dropped connection - produces no callback at all, so the preview never opens and the user gets no feedback. Without a matching off('complete') each tap of the document row adds another live subscription that captures the component and is never released.
- Fix: Register a failure handler alongside the completion handler, and release both when the page or component is torn down: 'downloadTask.on("fail", (err: number) => Logger.error(...));' plus 'downloadTask.off("complete", completeCallback)' and 'downloadTask.off("fail", failCallback)' in aboutToDisappear.

### `HW-05-0033` - The file-preview chain has no rejection handler on either canPreview or openPreview, and openPreview's then body is empty

- Category B, severity medium, confidence confirmed
- Features: OFFICE-06
- Document: `huawei_industry_tree/05_office/docs/06_document_approval.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/document_approval-0000002280673593
- Why: canPreview is documented as checking "only whether the file exists and whether the file format is supported", so a false result and a rejection are both ordinary outcomes for a file that has just been downloaded. As written, an unsupported or missing file makes the approval document silently fail to open with no log line and no user message, and an openPreview rejection becomes an unhandled promise rejection.
- Fix: Attach .catch((err: BusinessError) => Logger.error(...)) to both promises as the reference examples do, and add an else branch for result === false that tells the user the file cannot be previewed.

### `HW-05-0034` - The download snippet in 实现思路 is not valid ArkTS: a malformed arrow function, a missing DocumentSaveOptions and an unbound result variable

- Category E, severity medium, confidence confirmed
- Features: OFFICE-06
- Document: `huawei_industry_tree/05_office/docs/06_document_approval.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/document_approval-0000002280673593
- Why: This is the only code the document gives for the download half of the scenario. As printed it does not parse, it omits the DocumentSaveOptions that supplies newFileNames - without which the save dialog has no proposed file name - and it drops the empty-result guard that protects documentSaveResult[0] when the user cancels the picker.
- Fix: Replace the snippet with the shipped implementation: construct DocumentSaveOptions, set newFileNames, pass it to save(), bind the result in a proper arrow function, guard against an empty array, and attach a .catch().

### `HW-05-0039` - Four camera listeners are registered on every cameraShooting call and none is ever unregistered, and the page calls cameraShooting twice per appearance

- Category B, severity medium, confidence confirmed
- Features: OFFICE-07
- Document: `huawei_industry_tree/05_office/docs/07_camera_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/camera_page-0000002245028040
- Why: Camera Kit exposes an off counterpart for each of these events; without it the callbacks accumulate on every page entry and on every camera switch. Because photoAssetAvailable is the handler that writes the captured asset and updates AppStorage 'photoUri' (CameraShooter.ets:387-397), duplicate registrations mean one shutter press runs the media-library save request more than once.
- Fix: Keep each callback in a module-level field, call the matching off ('error' for cameraInput/previewOutput/photoSession and 'photoAssetAvailable' for photoOutput) inside releaseCamera before releasing the objects, and give the page a single initialisation path rather than calling cameraShooting from both onLoad and onShown.

### `HW-05-0040` - The 权限说明 section lists only the camera permission while the sample declares four user_grant permissions

- Category E, severity medium, confidence confirmed
- Features: OFFICE-07
- Document: `huawei_industry_tree/05_office/docs/07_camera_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/camera_page-0000002245028040
- Why: All four are user_grant permissions, so each one that stays declared but unrequested is an authorization the store review will ask about and the user will never be prompted for. The document, which is the specification a reader implements from, describes only one of them, so the reader cannot tell whether the other three are required by the scenario or left over from a template.
- Fix: Reconcile the two: keep in module.json5 only the permissions this scenario actually exercises, request each of them at runtime through PermissionsView.commonRequestPermissions, and list the final set in the 权限说明 section with the reason each one is needed.

### `HW-05-0041` - The four user_grant permissions are declared with a usedScene that omits the mandatory when field

- Category D, severity medium, confidence confirmed
- Features: OFFICE-07
- Document: `huawei_industry_tree/05_office/docs/07_camera_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/camera_page-0000002245028040
- Why: CAMERA, MICROPHONE, READ_IMAGEVIDEO and WRITE_IMAGEVIDEO are all user_grant permissions, so usedScene is mandatory and its when parameter carries the foreground/background scope that release verification checks. Every other sample in this industry supplies it - for example LocationPermissions.zip#entry/src/main/module.json5:43 '"when": "inuse"'.
- Fix: Add '"when": "inuse"' to each usedScene block, since all four permissions are used only while the capture page is in the foreground.

### `HW-05-0042` - PermissionsView passes the raw getHostContext() result to requestPermissionOnSetting, whose context parameter is documented as a mandatory Context

- Category A, severity medium, confidence confirmed
- Features: OFFICE-07
- Document: `huawei_industry_tree/05_office/docs/07_camera_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/camera_page-0000002245028040
- Why: getHostContext() may return undefined, so its raw result does not satisfy the mandatory Context parameter. This is the second-stage authorization path for the camera permission, which is the only permission this page requests at runtime, so it is on the critical path of the scenario.
- Fix: Cast the argument as the same file already does for requestPermissionsFromUser: 'await atManager.requestPermissionOnSetting(context.getHostContext() as common.UIAbilityContext, permissions);'.

### `HW-05-0043` - NavDestination.onShown starts the camera with the module-level surfaceId, which is still an empty string until the XComponent onLoad callback assigns it

- Category B, severity medium, confidence probable
- Features: OFFICE-07
- Document: `huawei_industry_tree/05_office/docs/07_camera_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/camera_page-0000002245028040
- Why: On the first appearance of the page the destination is shown before any surface exists, so an onShown that runs ahead of onLoad hands an empty surface id to createPreviewOutput. The 500 ms setTimeout in onLoad is itself an acknowledgement that the surface is not ready immediately. The exact ordering of onShown against onLoad is not established here, which is why this is recorded as probable, but the variable is provably empty until onLoad executes and no guard prevents its use.
- Fix: Guard the start on a non-empty surface id and drive initialisation from a single place: keep the start inside onLoad after getXComponentSurfaceId returns, and in onShown call cameraShooting only when surfaceId is truthy. Replace the fixed 500 ms delay with the surface-ready callback.

### `HW-05-0046` - myRawfileCopy dereferences a null File in its finally block and closes the descriptor with an unawaited asynchronous fs.close

- Category B, severity medium, confidence confirmed
- Features: OFFICE-08
- Document: `huawei_industry_tree/05_office/docs/08_file_view.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/file_view-0000002256155688
- Why: Two defects on one line. When fs.openSync throws - the case the catch exists for - 'file' is still the null it was initialised with at :36, so 'file.fd' in the finally raises a TypeError that escapes the getRawFileContent callback and hides the original failure. On the success path the promise returned by fs.close is neither awaited nor given a rejection handler, so the descriptor release is unobserved and a close failure is silent. The catch block also logs at info level with a fixed string and drops the BusinessError entirely.
- Fix: Guard and use the synchronous close: 'finally { if (file) { fs.closeSync(file); } }', and log the caught error with hilog.error including err.code and err.message.

### `HW-05-0047` - The .xlsx branch sets the legacy .xls MIME type instead of the OOXML spreadsheet type documented for xlsx

- Category A, severity medium, confidence confirmed
- Features: OFFICE-08
- Document: `huawei_industry_tree/05_office/docs/08_file_view.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/file_view-0000002256155688
- Why: Reading the official table position by position, xls maps to application/vnd.ms-excel and xlsx maps to application/vnd.openxmlformats-officedocument.spreadsheetml.sheet. The sample declares the xls type for an xlsx file in the PreviewInfo it hands to filePreview.openPreview, so the previewer is told the wrong format for one of the five file types this scenario exists to demonstrate.
- Fix: Change the xlsx branch to mType = 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet', matching the official mapping and the pattern the docx branch already follows.

### `HW-05-0048` - The download progress callback raises a toast on every progress event, and neither download listener is released nor is a failure listener registered

- Category B, severity medium, confidence confirmed
- Features: OFFICE-08
- Document: `huawei_industry_tree/05_office/docs/08_file_view.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/file_view-0000002256155688
- Why: The progress event fires repeatedly for the duration of a transfer, so putting showToast inside it queues one toast per chunk rather than reporting the download once. Separately, without on('fail') a transfer that starts and then fails never calls back at all - the preview simply never opens and nothing is logged - and without the matching off calls both subscriptions survive every download the user starts.
- Fix: Move the toast out of the progress callback and use the received/total values to drive a progress indicator instead; register on('fail') to report failures; and call off('progress', progressCallback), off('complete', completeCallback) and off('fail', failCallback) once the task has finished or the page is torn down.

### `HW-05-0051` - The full contact record returned by selectContacts is serialized into a hilog call, writing the contact's name and phone numbers to the system log

- Category D, severity medium, confidence confirmed
- Features: OFFICE-09
- Document: `huawei_industry_tree/05_office/docs/09_guest_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/guest_demo-0000002257167990
- Why: Two problems compound. The call passes only three arguments, so '%s' lands in the tag slot and the serialized contact becomes the format string itself - which means the privacy identifier mechanism that would have filtered it as {private} never applies and the contact data is emitted verbatim. Contact names and phone numbers taken from the system address book are personal data and must not be written to the log at all.
- Fix: Remove the log line, or log only a non-identifying fact such as the number of contacts returned: hilog.info(0x0001, 'GuestDemo', 'selectContacts returned %{public}d entries', data.length).

### `HW-05-0052` - The contact.selectContacts promise has no rejection handler in either the document or the sample

- Category B, severity medium, confidence confirmed
- Features: OFFICE-09
- Document: `huawei_industry_tree/05_office/docs/09_guest_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/guest_demo-0000002257167990
- Why: selectContacts opens the system contact picker, so a user who dismisses it or a picker that fails to start produces a rejection. With no handler the rejection is unhandled, the name and phone fields silently stay empty, and the user is given no indication that the import did not happen.
- Fix: Attach a rejection handler that logs the code and message and leaves the manually typed values intact: 'promise.then((data) => { ... }).catch((err: BusinessError) => hilog.error(0x0001, "GuestDemo", "selectContacts failed: %{public}s", JSON.stringify(err)));'.

### `HW-05-0053` - The top inset is read from AvoidAreaType.TYPE_CUTOUT although the code comments it as the status-bar height, so on a device without a cutout the padding collapses to a hard-coded 15

- Category A, severity medium, confidence confirmed
- Features: OFFICE-09
- Document: `huawei_industry_tree/05_office/docs/09_guest_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/guest_demo-0000002257167990
- Why: The window is put into immersive layout by setWindowLayoutFullScreen(true) at EntryAbility.ets:41, so the pages must pad by the status-bar inset, which is TYPE_SYSTEM. TYPE_CUTOUT reports only the notch or punch-hole region and is zero on a device that has none, leaving every page - GuestListPage.ets:155, AddViewPage.ets:317, AddressPage.ets:96 and GuestViewPage.ets:145 all consume topRectHeight - with only the hard-coded 15 vp between its title row and the top of the screen.
- Fix: Read the status-bar inset from window.AvoidAreaType.TYPE_SYSTEM as the sibling samples in this industry do, and handle TYPE_SYSTEM rather than TYPE_CUTOUT in the avoidAreaChange branch. Keep TYPE_CUTOUT only if a separate notch-specific adjustment is genuinely needed.

### `HW-05-0054` - The guest list @State is bound directly to the exported module constant and then sorted and pushed in place, permanently mutating shared module data

- Category C, severity medium, confidence confirmed
- Features: OFFICE-09
- Document: `huawei_industry_tree/05_office/docs/09_guest_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/guest_demo-0000002257167990
- Why: The assignment copies the reference, not the array, so @State and the exported constant are the same object. sort() and push() mutate it, so the module-level SAMPLE_GUEST_LIST that other code imports as a constant is reordered and grows every time a guest is added. A second component that reads SAMPLE_GUEST_LIST would observe the first component's edits, and the sample data can no longer be relied on as a fixed baseline.
- Fix: Copy the seed data into the state instead of aliasing it: '@State dataArr: GuestData[] = [...SAMPLE_GUEST_LIST];'. If the seed is meant to be immutable, freeze it or expose it through a factory function that returns a fresh array.

### `HW-05-0056` - EntryAbility subscribes to avoidAreaChange without a matching off and discards the setWindowLayoutFullScreen promise

- Category B, severity medium, confidence confirmed
- Features: OFFICE-09
- Document: `huawei_industry_tree/05_office/docs/09_guest_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/guest_demo-0000002257167990
- Why: The avoid-area subscription is created for each window stage and never released, so the register/unregister pair the reference defines stays open and the callback keeps writing into AppStorage for the process lifetime. Separately, setWindowLayoutFullScreen returns a promise whose rejection is unhandled, and the immersive layout it establishes is the precondition for the avoid-area padding read two lines later.
- Fix: Keep the callback in a field and call mainWindow.off('avoidAreaChange', callback) in onWindowStageDestroy, and await setWindowLayoutFullScreen inside a try/catch before reading getWindowAvoidArea.

### `HW-05-0057` - The MIME type is read from getTypeDescriptor without the null check and try/catch that the official examples both use

- Category B, severity medium, confidence confirmed
- Features: OFFICE-10
- Document: `huawei_industry_tree/05_office/docs/10_email_attachment.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/email_attachment-0000002319115245
- Why: Both UTD calls run in aboutToAppear, so any failure aborts the attachment row before it renders. getTypeDescriptor returns null for a type that does not exist, and reading .mimeTypes on null throws; the mimeTypes array can also be empty, in which case this.mimeType becomes undefined and is then handed to filePreview.openPreview as the PreviewInfo mimeType. A file whose extension is unusual - which is exactly what a document picker can deliver - therefore breaks the attachment list rather than degrading to a generic preview.
- Fix: Wrap both calls in try/catch as the reference examples do, check 'if (typeObj && typeObj.mimeTypes.length > 0)' before indexing, and fall back to mimeType: '' - the Preview Kit reference states that an empty string makes the system infer the type from the URI suffix.

### `HW-05-0058` - The attachment size is computed from an unguarded lstatSync whose optional-chained result is then used as if it were always a number

- Category B, severity medium, confidence confirmed
- Features: OFFICE-10
- Document: `huawei_industry_tree/05_office/docs/10_email_attachment.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/email_attachment-0000002319115245
- Why: The author wrote '?.size', which acknowledges that the stat result may be absent, but then compares and concatenates 'size' without handling the undefined case - so an unresolvable path yields the label 'undefinedB' instead of a size. Worse, lstatSync throws rather than returning undefined for a path it cannot reach, and the URIs here come from three different pickers (photo, camera, document) whose returned URIs are not all plain sandbox paths; an exception in aboutToAppear aborts the attachment row entirely.
- Fix: Wrap the stat in try/catch, default the size to 0 on failure, and branch on a definite number: 'let size = 0; try { size = fs.lstatSync(fileUriObject.path).size; } catch (e) { Logger.error(...); }'.

### `HW-05-0059` - filePreview.openPreview is called as a bare statement in both the document and the sample, so a failed preview is an unhandled rejection

- Category B, severity medium, confidence confirmed
- Features: OFFICE-10
- Document: `huawei_industry_tree/05_office/docs/10_email_attachment.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/email_attachment-0000002319115245
- Why: openPreview returns a Promise, so a file that Preview Kit refuses - an unsupported type, an unreadable URI, or the undefined mimeType that HW-05-0057 can produce - rejects with no handler. Tapping the attachment then does nothing at all, with no toast and no log line. The sample also never calls canPreview first, unlike the sibling samples in this industry.
- Fix: Attach .then()/.catch() as the reference example does, or await the call inside the already-async handler within a try/catch, and report the failure to the user. Optionally gate on filePreview.canPreview first.

### `HW-05-0060` - The attachment hint popup binds a plain private field that is filled asynchronously, so the popup opens empty and never updates

- Category C, severity medium, confidence confirmed
- Features: OFFICE-10
- Document: `huawei_industry_tree/05_office/docs/10_email_attachment.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/email_attachment-0000002319115245
- Why: Only @State-tracked members drive a re-render. addTip is a plain field, so the value assigned when the resource promise resolves never reaches the already-built bindPopup parameter. The popup is additionally opened from onAppear, which fires as soon as the attach icon is laid out - before the asynchronous getStringValue has resolved - so the hint is shown with the initial empty string.
- Fix: Declare the field as '@State addTip: string = "";' so the resolved value triggers a rebuild, and prefer the synchronous getStringSync (or a $r() resource passed straight to message) so the text is available before the popup opens.

### `HW-05-0064` - The MIME lookup table spells the Word OOXML extension 'docs' instead of 'docx' and uses a non-standard csv type

- Category A, severity medium, confidence confirmed
- Features: OFFICE-11
- Document: `huawei_industry_tree/05_office/docs/11_conference_release.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/conference_release-0000002321751725
- Why: Reading the official table position by position, the wordprocessingml type belongs to the extension docx, and csv maps to text/csv. Because the switch has no 'docx' case, a real Word document falls through to the default branch at FileUtils.ets:50-51 and receives an empty mimeType, and every csv is announced with a type the table does not list. findMimeType is the only source of PreviewInfo.mimeType at FileUtils.ets:113, so both errors reach filePreview.openPreview directly.
- Fix: Rename the case to 'docx' and change the csv branch to return 'text/csv'. The sibling extensions in the same switch (xls/xlsx, ppt/pptx) already follow the official mapping.

### `HW-05-0065` - EntryAbility ignores the authResults of the calendar permission request and creates the CalendarManager even when the user denies

- Category D, severity medium, confidence confirmed
- Features: OFFICE-11
- Document: `huawei_industry_tree/05_office/docs/11_conference_release.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/conference_release-0000002321751725
- Why: requestPermissionsFromUser resolves successfully even when the user denies; the grant state lives only in PermissionRequestResult.authResults. Because the manager is published into AppStorage unconditionally, the publish button at PublishMeetingPage.ets:702 calls editEvent on a manager the app is not authorised to use, and the only feedback is a hilog line - the user sees the meeting silently fail to reach the system calendar, which is the whole point of the scenario.
- Fix: Bind the result and gate the manager creation: 'requestPermissionsFromUser(mContext, permissions).then((data: PermissionRequestResult) => { if (data.authResults.every((r) => r === 0)) { AppStorage.setOrCreate("calendarMgr", calendarManager.getCalendarManager(mContext)); } else { /* direct the user to Settings */ } })'.

### `HW-05-0066` - Four empty error handlers cover the whole attachment pipeline, so every file and preview failure is discarded

- Category B, severity medium, confidence confirmed
- Features: OFFICE-11
- Document: `huawei_industry_tree/05_office/docs/11_conference_release.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/conference_release-0000002321751725
- Why: These four handlers are the complete error surface of the upload-and-preview flow: opening the sandbox file, copying the bytes, opening the previewer, and choosing the file. With all of them empty, the misrouted copy recorded as HW-05-0063 produces no symptom whatsoever - no toast, no log line - which is precisely why that defect can ship unnoticed.
- Fix: Log each caught error with hilog.error including code and message, and surface the copy and preview failures to the user with the promptAction the page already holds at PublishMeetingPage.ets:158.

### `HW-05-0067` - The attachment copy opens both source and destination READ_WRITE|CREATE without TRUNC, so a re-attached shorter file keeps a stale tail and a missing source is silently created empty

- Category B, severity medium, confidence confirmed
- Features: OFFICE-11
- Document: `huawei_industry_tree/05_office/docs/11_conference_release.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/conference_release-0000002321751725
- Why: Two consequences follow from the mode. The copy loop at FileUtils.ets:91-99 writes from offset 0 forward but never shortens the destination, so re-attaching a smaller file over an existing same-named sandbox file leaves the previous file's trailing bytes in place. And because the source is opened with CREATE, a srcDir that does not exist is created as an empty file rather than raising an error - the copy then silently produces a zero-byte attachment. The source is only ever read (FileUtils.ets:90 and :98), so requesting read/write access to the user-picked document is unnecessary as well.
- Fix: Open the source read-only without CREATE - 'fs.openSync(srcDir, fs.OpenMode.READ_ONLY)' - and truncate the destination - 'fs.openSync(destFileUri, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE | fs.OpenMode.TRUNC)'. The separate createFile pre-pass then becomes unnecessary.

### `HW-05-0069` - The zero-padding of the meeting date is immediately discarded by wrapping the padded string in Number(), so selectedDate is never zero-padded

- Category B, severity medium, confidence confirmed
- Features: OFFICE-11
- Document: `huawei_industry_tree/05_office/docs/11_conference_release.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/conference_release-0000002321751725
- Why: Number('08') evaluates to the number 8, so the leading zero the ternary exists to add is thrown away before the value is concatenated. selectedDate therefore reads '2026-8-5' rather than '2026-08-05'. That string is stored in the shared FormData and is later concatenated and parsed as a date at ConferenceRelease.zip#entry/src/main/ets/pages/PublishMeetingPage.ets:695-698 'new Date(this.meetingFormData.selectedDate + "-00:00").getTime()', which feeds startTime and endTime of the calendar Event.
- Fix: Keep the padded value as a string, exactly as CustomTimePickerDialog does: 'const hm = m < 10 ? "0" + m : `${m}`; const hd = d < 10 ? "0" + d : `${d}`; date = `${y}-${hm}-${hd}`;'.

### `HW-05-0070` - The weekday label array starts at Monday but is indexed with Date.getDay(), which returns 0 for Sunday, so every weekday shown in the date dialog is wrong

- Category B, severity medium, confidence confirmed
- Features: OFFICE-11
- Document: `huawei_industry_tree/05_office/docs/11_conference_release.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/conference_release-0000002321751725
- Why: Date.getDay() is defined to return 0 for Sunday and 6 for Saturday, while this array places Monday at index 0. Every label is therefore shifted by one: a Sunday is titled 星期一 (Monday), a Monday is titled 星期二 (Tuesday), and a Saturday - getDay() 6 - is titled 星期日 (Sunday). The title is the only confirmation the user gets of which day the meeting is being scheduled on.
- Fix: Order the array from Sunday so it matches getDay(): "const WEEK_DAYS = ['星期日', '星期一', '星期二', '星期三', '星期四', '星期五', '星期六'];", or keep the Monday-first array and index it with '(value.getDay() + 6) % 7'.

### `HW-05-0072` - The save flow copies to uriSave outside its own guard and shows the success toast unconditionally, ignoring both a cancelled picker and the copy's return value

- Category B, severity medium, confidence confirmed
- Features: OFFICE-12
- Document: `huawei_industry_tree/05_office/docs/12_pdf_add_mark.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pdf_add_mark-0000002324100453
- Why: The null/undefined check does not cover an empty array, which is what a cancelled picker yields, so uriSave keeps its previous value - '' on the first cancel. copyFile2Document is then called with that value and its false return is discarded, and the success toast fires regardless. The user is told the stamped PDF was saved when nothing was written anywhere.
- Fix: Return early on an empty result and branch on the copy result: 'if (!documentSaveResult || documentSaveResult.length === 0) { return; } this.uriSave = documentSaveResult[0]; if (FileUtils.copyFile2Document(this.outPdfPath, this.uriSave)) { showToast(save_success); } else { showToast(save_failed); }'.

### `HW-05-0073` - copyFile2Document opens two file handles with no try/finally, so any failure leaks both descriptors

- Category B, severity medium, confidence confirmed
- Features: OFFICE-12
- Document: `huawei_industry_tree/05_office/docs/12_pdf_add_mark.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pdf_add_mark-0000002324100453
- Why: If opening the destination throws - the picker URI is unwritable, for instance - srcFile is never closed; if copyFileSync throws, neither handle is closed, and the exception escapes into the picker's then-handler where the caller has no catch of its own for it. The source is only read, so opening it READ_WRITE also requests more access than the operation needs.
- Fix: Wrap the body in try/catch/finally like copyResourceFile2Cache in the same class, close both handles in the finally, open the source with fileIo.OpenMode.READ_ONLY, and return false from the catch.

### `HW-05-0076` - The startAbility snippet omits the showDefaultPicker parameter that forces the file-opening dialog the scenario is named after

- Category E, severity medium, confidence confirmed
- Features: OFFICE-13
- Document: `huawei_industry_tree/05_office/docs/13_intergratoffice_demo_startability.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/intergratoffice_demo_startability-0000002329091989
- Why: The document is titled 选择文件打开方式 ("choose how to open a file"), and the dialog that lets the user choose only appears unconditionally when showDefaultPicker is true. Without it the default is false and, per the reference, the system may go straight to the default application without ever offering a choice. A reader copying the document's snippet therefore does not get the behaviour the page demonstrates.
- Fix: Add the parameter to the snippet, matching the sample: 'parameters: { "ohos.ability.params.showDefaultPicker": true },' and explain in the step text that it is what forces the selection dialog.

### `HW-05-0077` - Implementation steps 2 and 3 describe a target-application side that the shipped sample does not contain, and the document never says so

- Category E, severity medium, confidence confirmed
- Features: OFFICE-13
- Document: `huawei_industry_tree/05_office/docs/13_intergratoffice_demo_startability.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/intergratoffice_demo_startability-0000002329091989
- Why: The three numbered steps read as one implementation, but the download contains only step 1. A reader who follows the document and then opens the ZIP to check steps 2 and 3 finds no corresponding code, and cannot tell whether the target side was omitted deliberately or is missing by mistake. The scenario is only demonstrable end to end when some installed application declares the matching skills.
- Fix: State explicitly that the sample implements the caller side only, and that steps 2 and 3 describe what a separate target application must declare and implement; alternatively ship a second module that declares the viewData skill so the flow can be run end to end.

### `HW-05-0078` - Both ForEach key generators declare their first parameter as a number although the framework passes the data item there

- Category C, severity medium, confidence confirmed
- Features: OFFICE-13
- Document: `huawei_industry_tree/05_office/docs/13_intergratoffice_demo_startability.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/intergratoffice_demo_startability-0000002329091989
- Why: The keyGenerator's first parameter is the data item and the second is the index. Declaring it as 'index: number' means a File object is bound to a parameter typed number, and JSON.stringify then serialises the whole file record rather than the intended index - so the identifier is misleading and the declared type does not match what the framework supplies. The same guide also warns against using the index as a key at all: "Do not use the data item index as the key, as this can cause unexpected rendering results and reduced rendering performance."
- Fix: Give each File a stable id and key on it: '}, (file: File) => file.id;'. If no id exists, at minimum name the parameters correctly - '(file: File, index: number) => ...' - so the declaration matches what ForEach passes.

### `HW-05-0079` - Every file row opens the same hard-coded 1.txt as plain text, so tapping the PDF or JPG entries requests a handler for a file unrelated to the row

- Category B, severity medium, confidence confirmed
- Features: OFFICE-13
- Document: `huawei_industry_tree/05_office/docs/13_intergratoffice_demo_startability.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/intergratoffice_demo_startability-0000002329091989
- Why: The list advertises three different file types, but each tap writes and opens the same one-byte 1.txt and always declares general.plain-text. The reference warns that the declared type must match the file behind the URI or no application will be matched - so a reader adapting this to a real file list, where the type does have to follow the row, gets no guidance from the sample at all.
- Fix: Pass the row to the helper and derive both values from it: 'Utils.startAbility(file)' with the path taken from the row and the type derived from its extension (the sibling samples use uniformTypeDescriptor.getUniformDataTypeByFilenameExtension for exactly this).

### `HW-05-0080` - EntryAbility subscribes to avoidAreaChange without a matching off in onWindowStageDestroy

- Category B, severity medium, confidence confirmed
- Features: OFFICE-13
- Document: `huawei_industry_tree/05_office/docs/13_intergratoffice_demo_startability.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/intergratoffice_demo_startability-0000002329091989
- Why: The subscription is created for each window stage and never released, so the register/unregister pair the reference defines is left open and the callback keeps writing into AppStorage for the lifetime of the process.
- Fix: Keep the callback in a field and call windowClass.off('avoidAreaChange', callback) in onWindowStageDestroy.

### `HW-05-0081` - The linearGradient snippet references a CommonConstants class and a NOTIFICATION_HEIGHT member that do not exist in the sample, and shows only one of the two gradient overlays

- Category E, severity medium, confidence confirmed
- Features: OFFICE-14
- Document: `huawei_industry_tree/05_office/docs/14_text_marquee.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/text_marquee-0000002331949121
- Why: The 场景介绍 at document line 15 promises 两端渐变的跑马灯横幅效果 ("a marquee banner with a gradient at both ends"), but the snippet shows a single left-direction gradient and names a constants class the project does not have. A reader copying it gets a compile error on the identifiers and, once those are fixed, a fade on one side only.
- Fix: Replace the snippet with the shipped pair: a Flex with FlexAlign.SpaceBetween holding one GradientDirection.Right column and one GradientDirection.Left column, both sized with $r('app.float.notification_height'), and drop the non-existent CommonConstants references.

### `HW-05-0082` - The message list uses ForEach with no key generator while deleting items by index, the pattern the ForEach guide warns against

- Category C, severity medium, confidence confirmed
- Features: OFFICE-14
- Document: `huawei_industry_tree/05_office/docs/14_text_marquee.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/text_marquee-0000002331949121
- Why: The default key embeds the index, so removing one row through swipeAction shifts the index of every following row and changes all of their keys. The framework then treats them as new data and rebuilds each one instead of reusing it - the reduced rendering performance the guide names - and the default key additionally runs JSON.stringify over each message item on every rebuild.
- Fix: Supply a keyGenerator over the id the model already carries: '}, (item: MessageItem) => item.id.toString());'.

### `HW-05-0083` - The consumed session list is assigned the exported module constant and then spliced in place, permanently mutating shared module data

- Category C, severity medium, confidence confirmed
- Features: OFFICE-14
- Document: `huawei_industry_tree/05_office/docs/14_text_marquee.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/text_marquee-0000002331949121
- Why: The assignment copies the reference, not the array, so the provided state and the exported constant become the same object. Every swipe-to-delete splices the module-level MESSAGE_DATA, so the seed data shrinks for the lifetime of the process and any other consumer of that export sees the deletions. Re-entering the page cannot restore the original list because aboutToAppear re-assigns the already-mutated array.
- Fix: Copy the seed data instead of aliasing it: 'this.sessionList = [...MESSAGE_DATA];', or expose the seed through a factory that returns a fresh array.

### `HW-05-0085` - isKeyExist returns its local flag before the asynchronous Preferences.has() resolves, so it always reports false

- Category B, severity medium, confidence confirmed
- Features: OFFICE-15
- Document: `huawei_industry_tree/05_office/docs/15_later_items.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/later_items-0000002301534190
- Why: The return statement executes synchronously, before the promise callback that assigns isKeyExist can run, so the method returns the initial false on every call regardless of whether the key exists. The sibling methods in the same class already use the synchronous Preferences API - putSync at PreferenceUtils.ets:33 and getSync at PreferenceUtils.ets:44 - so the async call here is inconsistent as well as wrong.
- Fix: Use the synchronous counterpart to match the rest of the class: 'isKeyExist(): boolean { return this.preference?.hasSync(KEY_APP_LATER_ITEMS) ?? false; }', or make the method async and return the promise.

### `HW-05-0086` - The top inset is read from AvoidAreaType.TYPE_CUTOUT although the surrounding comment and usage mean the status bar

- Category A, severity medium, confidence confirmed
- Features: OFFICE-15
- Document: `huawei_industry_tree/05_office/docs/15_later_items.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/later_items-0000002301534190
- Why: The window is switched to immersive layout at EntryAbility.ets:50 with setWindowLayoutFullScreen(true), so the page must pad by the status-bar inset, which is TYPE_SYSTEM. TYPE_CUTOUT reports only the notch or punch-hole region and is zero on a device without one, leaving the header row with only the hard-coded 15 vp between it and the top of the screen. The navigation inset on the following lines correctly uses TYPE_NAVIGATION_INDICATOR, which makes the cutout choice for the top inset an inconsistency within the same block.
- Fix: Read the top inset from window.AvoidAreaType.TYPE_SYSTEM, keeping TYPE_CUTOUT only if a separate notch-specific adjustment is genuinely required.

### `HW-05-0087` - The getMainWindow callback dereferences data without checking err, and setWindowLayoutFullScreen is called without handling its promise

- Category B, severity medium, confidence confirmed
- Features: OFFICE-15
- Document: `huawei_industry_tree/05_office/docs/15_later_items.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/later_items-0000002301534190
- Why: When getMainWindow fails, data is undefined and 'data.getUIContext()' raises a TypeError inside the callback, so neither avoidTop nor avoidBottom is ever published and every page loses its inset padding with no diagnostic. Separately, setWindowLayoutFullScreen returns a promise whose rejection is unhandled, and the immersive layout it establishes is the precondition for the avoid-area values read on the next four lines.
- Fix: Check the error first, as the enclosing loadContent callback already does: 'if (err.code) { hilog.error(...); return; }', and await or chain the layout call before reading getWindowAvoidArea.

### `HW-05-0088` - The long-press menu dispatches its action by comparing the displayed title against a hard-coded Chinese literal

- Category B, severity medium, confidence confirmed
- Features: OFFICE-15
- Document: `huawei_industry_tree/05_office/docs/15_later_items.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/later_items-0000002301534190
- Why: The only behaviour this scenario implements - adding a message to the to-do list - is selected by string-matching the label shown to the user. Moving the menu titles into string resources so the app can be localized, or editing the label, silently disables the feature and the blue highlight with no compile error. The menu item has no stable key to dispatch on.
- Fix: Give menuItem a stable discriminator - an enum or id field - and dispatch on it: 'if (item.action === MenuAction.LATER) { ... }', then move the titles into $r('app.string.*') resources like the rest of the project.

### `HW-05-0091` - EntryAbility subscribes to avoidAreaChange without a matching off in onWindowStageDestroy

- Category B, severity medium, confidence confirmed
- Features: OFFICE-16
- Document: `huawei_industry_tree/05_office/docs/16_recommend_id_photos.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/recommend_id_photos-0000002344341185
- Why: The subscription is created for each window stage and never released, so the register/unregister pair the reference defines stays open and the callback keeps writing into AppStorage for the lifetime of the process.
- Fix: Keep the callback in a field and call windowClass.off('avoidAreaChange', callback) in onWindowStageDestroy.

### `HW-05-0094` - Every recursion level nests a new List inside the parent List with no main-axis size, the arrangement the List reference says defeats lazy loading

- Category C, severity medium, confidence confirmed
- Features: OFFICE-17
- Document: `huawei_industry_tree/05_office/docs/17_multi_level_nesting_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/multi_level_nesting_list-0000002311817336
- Why: The reference describes exactly this arrangement - a vertically scrolling List inside a vertically scrolling List with no main-axis height - and states that the outer List then loads every child, so no lazy loading occurs anywhere in the tree. Because nestedScroll defaults to SELF_ONLY, the inner lists also do not hand scrolling back to the outer one, so a deep branch scrolls independently instead of continuing the page. Each level additionally allocates a Scroller that nothing ever calls.
- Fix: Use a single List at the page level and render the recursive levels with ListItemGroup or plain Columns, as the reference advises, so only the outermost container scrolls; drop the per-node Scroller instances, or set an explicit main-axis size and nestedScroll on the inner lists if independent scrolling is genuinely wanted.

### `HW-05-0095` - The checkbox selection is handled in onClick instead of the documented onChange, so the propagation logic depends on an unstated ordering against the $$ two-way binding

- Category A, severity medium, confidence confirmed
- Features: OFFICE-17
- Document: `huawei_industry_tree/05_office/docs/17_multi_level_nesting_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/multi_level_nesting_list-0000002311817336
- Why: onClick is a universal touch event; the event the reference defines for a selection change is onChange, which also delivers the new value as its callback argument. The commented-out line at NodeItem.ets:111 records the assumption the code rests on - that the $$ binding has already flipped item.selected by the time the click handler runs. Nothing in the reference guarantees that ordering, and if it does not hold every branch in itemSelected propagates the previous state: children are set to the old value and the parent counter moves the wrong way.
- Fix: Use the documented event and take the value from the callback rather than re-reading the bound field: '.onChange((value: boolean) => { this.itemSelected(this.item, value); })', and pass that boolean into checkChildren / checkParent instead of item.selected.

### `HW-05-0096` - The HashMap lookup result is assigned to a non-optional Node and dereferenced without a check while building the tree

- Category B, severity medium, confidence confirmed
- Features: OFFICE-17
- Document: `huawei_industry_tree/05_office/docs/17_multi_level_nesting_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/multi_level_nesting_list-0000002311817336
- Why: The nodes come from a flat MockData array in which every pid is expected to name another entry, but nothing enforces that. A typo in a pid, or a child listed before its parent is inserted into a differently ordered data source, makes get() return undefined; 'parent.children.push(item)' then raises a TypeError inside aboutToAppear and the page fails to build. The same applies to the 'root' lookup, whose result is asserted non-null at the render site rather than validated.
- Fix: Check both lookups and skip or report an orphan: 'const parent = this.hashMap.get(item.pid); if (!parent) { Logger.error(`missing parent ${item.pid} for ${item.id}`); return; } item.parent = parent; parent.children.push(item);', and guard rootItem before rendering instead of using the ! assertion.

### `HW-05-0098` - Each window pushes its surfaceId into a shared AppStorage array that is never pruned, so stale surfaces accumulate and receive preview outputs

- Category B, severity medium, confidence confirmed
- Features: OFFICE-18
- Document: `huawei_industry_tree/05_office/docs/18_conference_window_change.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/conference_window_change-0000002325708930
- Why: The array is the registry of live preview surfaces, but only additions are recorded. Any reload of a window's content - the sub windows are re-created whenever the page is entered again, since nothing destroys them either - appends a second id for the same window while the dead one stays, and the next createRecorder builds a preview output against a surface that no longer exists and adds it to the session.
- Fix: Remove the id in aboutToDisappear before releasing the camera: 'const i = this.surfaceIds.indexOf(this.xComponentSurfaceId); if (i >= 0) { this.surfaceIds.splice(i, 1); }', and reassign the array so the @StorageLink observes the change.

### `HW-05-0099` - The two sub-windows are created on every page appearance and never destroyed, and the window-configuration promises around them are discarded

- Category B, severity medium, confidence confirmed
- Features: OFFICE-18
- Document: `huawei_industry_tree/05_office/docs/18_conference_window_change.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/conference_window_change-0000002325708930
- Why: The reference provides destroyWindow specifically for application child windows, and the sample creates two of them from a lifecycle hook that runs on every appearance of the page. Nothing releases them, so re-entering the meeting leaves the previous pair alive - which is also what makes the stale surfaceIds recorded in HW-05-0098 reachable. Separately, resize, moveWindowTo and showWindow all return promises whose rejections are unhandled, so a window that fails to size or show does so silently.
- Fix: Keep references to both sub-windows and call destroyWindow on each in aboutToDisappear, and await (or attach .catch to) setUIContent, resize, moveWindowTo and showWindow so a configuration failure is reported.

### `HW-05-0100` - The getAllDisplays callback ignores its error argument and indexes element 0 of the result unconditionally

- Category B, severity medium, confidence confirmed
- Features: OFFICE-18
- Document: `huawei_industry_tree/05_office/docs/18_conference_window_change.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/conference_window_change-0000002325708930
- Why: On failure data is undefined and 'DISPLAY_CONFIG[0].width' raises a TypeError inside the callback, so neither sub-window is ever created and the meeting screen stays blank with no diagnostic. The correct guard is written twice in the same function for the window callbacks, which makes the omission on the display callback an inconsistency rather than a deliberate simplification.
- Fix: Guard it the same way: 'if (err.code > 0 || !data || data.length === 0) { Logger.error(`getAllDisplays error: ${JSON.stringify(err)}`); return; }' before reading data[0].

### `HW-05-0101` - The sub-window creation snippet does not parse and omits the showWindow ordering the sample documents as mandatory

- Category E, severity medium, confidence confirmed
- Features: OFFICE-18
- Document: `huawei_industry_tree/05_office/docs/18_conference_window_change.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/conference_window_change-0000002325708930
- Why: Step 1 is the only description the document gives of how the two windows are created, and as printed it is not valid ArkTS. More importantly it hides the two ordering rules the sample calls out in its own comments - setRaiseByClickEnabled must be called after showWindow has completed to take effect, and the guest window must be created after the moderator window has been shown because showWindow raises the window level. A reader following the snippet nests the two creations with no awaits at all and gets the window-layering confusion the sample was written to avoid.
- Fix: Replace the snippet with the shipped sequence: create the moderator window, setUIContent, resize, moveWindowTo, 'await subWindow.showWindow()', then 'await subWindow.setRaiseByClickEnabled(false)', and only then create the guest window - closing each callback with '});' and giving the second callback its own parameter name.

### `HW-05-0102` - EntryAbility subscribes to avoidAreaChange on the main window without a matching off in onWindowStageDestroy

- Category B, severity medium, confidence confirmed
- Features: OFFICE-18
- Document: `huawei_industry_tree/05_office/docs/18_conference_window_change.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/conference_window_change-0000002325708930
- Why: The subscription is created for each window stage and never released, so the register/unregister pair the reference defines is left open and the callback keeps holding the main window and writing into AppStorage for the lifetime of the process. This sample is more exposed than most because it also creates and abandons two sub-windows (HW-05-0099).
- Fix: Keep the callback in a field and call mWindow.off('avoidAreaChange', callback) in onWindowStageDestroy.

### `HW-05-0106` - The ImageSource and both PixelMaps created on every capture are never released, contrary to the image-decoding guide's explicit release step

- Category B, severity medium, confidence confirmed
- Features: OFFICE-19
- Document: `huawei_industry_tree/05_office/docs/19_watermark_camera.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/watermark_camera-0000002334789536
- Why: Every shutter press decodes a full-resolution image into a PixelMap, draws it onto an OffscreenCanvas and produces a second full-resolution PixelMap, and none of the three native objects is released. The previous watermarked PixelMap is simply overwritten in a module-level variable, so its backing memory is dropped without a release call. The guide lists the release as a required step precisely because these objects hold native buffers.
- Fix: Release each object once it is no longer needed, as the guide's helper does: release the ImageSource after createPixelMap returns, release the decoded camera PixelMap after drawImage, and release the previous addedWatermarkPixelMap before assigning a new one.

### `HW-05-0107` - The photoAssetAvailable listener is registered on every camera start and never unregistered

- Category B, severity medium, confidence confirmed
- Features: OFFICE-19
- Document: `huawei_industry_tree/05_office/docs/19_watermark_camera.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/watermark_camera-0000002334789536
- Why: Camera Kit exposes an off counterpart for this event. Without it the callback set stays attached to each photo output for as long as the object lives, and because the registration happens inside the pipeline setup rather than once, every camera restart adds another handler. Since this handler is what decodes the captured photo, applies the watermark and opens the preview dialog, a duplicate registration means one shutter press runs the whole decode-and-watermark chain more than once.
- Fix: Unregister before releasing the output: call photoOutput.off('photoAssetAvailable', callback) at the start of releaseCamera, keeping the callback in a field so it can be passed to off.

### `HW-05-0108` - The shutter abandons the capture entirely when the location fix fails, logging at info level and giving the user no feedback

- Category B, severity medium, confidence confirmed
- Features: OFFICE-19
- Document: `huawei_industry_tree/05_office/docs/19_watermark_camera.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/watermark_camera-0000002334789536
- Why: photoOutPut.capture is only reached from inside the location callback's success path, so any condition that prevents a fix - the location switch being off, the permission being denied, or no fix within the timeout - makes the shutter button do nothing at all. The only trace is a Logger.info line, and the location error is not surfaced to the user, so the camera appears broken rather than unable to stamp a position.
- Fix: Separate the two concerns: always capture, and treat the location as optional metadata. Report the failure to the user, and either capture without the location field or offer to retry: 'if (err) { Logger.error(TAG, `location failed: ${err.code} ${err.message}`); showToast(location_unavailable); this.photoOutPut?.capture({ quality, rotation, mirror: isFront }); return; }'.

### `HW-05-0109` - The watermarked image is published one line after an un-awaited async call that produces it, so the handoff works only because addWatermark happens to contain no await

- Category B, severity medium, confidence confirmed
- Features: OFFICE-19
- Document: `huawei_industry_tree/05_office/docs/19_watermark_camera.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/watermark_camera-0000002334789536
- Why: The read succeeds today only because addWatermark's body contains no await, so an async function with a fully synchronous body runs to completion before returning its promise. That is an implementation detail, not a contract: introducing any await into addWatermark - to release the previous PixelMap, or to load a font - would make AppStorage publish the previous capture's watermark, or null on the first shot, with no error anywhere. The neighbouring call on the line above is correctly awaited, which makes the omission an inconsistency rather than a deliberate choice.
- Fix: Await it and take the result from the return value rather than a module-level variable: 'const watermarked = await this.addWatermark(cameraImage, text, this.uiContext); AppStorage.setOrCreate("locationUri", watermarked);' with addWatermark returning the PixelMap it creates.

### `HW-05-0112` - The breakpoint updater only ever produces sm or lg, so the md branch of every BreakpointType lookup is unreachable

- Category C, severity medium, confidence confirmed
- Features: OFFICE-20
- Document: `huawei_industry_tree/05_office/docs/20_note_with_camera.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/note_with_camera-0000002378064505
- Why: The document presents this sample as a responsive-layout adaptation for phones and tablets ("适配手机及平板设备"), and the three-value BreakpointType is the mechanism. Because the producer collapses everything at or above 600 vp into lg, the md value passed at every call site is dead code and a window in the 600-840 vp range - the medium tablet and split-screen case the md breakpoint exists for - receives the large layout.
- Fix: Emit all three breakpoints from updateBreakpoint: 'if (windowWidthVp < 600) { curBp = Constants.BREAKPOINT_SM; } else if (windowWidthVp < 840) { curBp = Constants.BREAKPOINT_MD; } else { curBp = Constants.BREAKPOINT_LG; }', or drop the md value from BreakpointType and its call sites so the API matches what the producer can emit.

### `HW-05-0113` - Both image loaders close a File that may be undefined and never release the ImageSource they create

- Category B, severity medium, confidence confirmed
- Features: OFFICE-20
- Document: `huawei_industry_tree/05_office/docs/20_note_with_camera.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/note_with_camera-0000002378064505
- Why: Two problems on the same lines. The finally runs even when openSync throws - the only reason a try/finally is there - and at that point fileSource is still the undefined it was initialised to, so closeSync receives undefined where the signature requires a File or a number. And imageSource is declared with an explicit '| undefined' type as though it were going to be released in the finally, but no release call exists anywhere in either file, so every imported photo leaks its ImageSource.
- Fix: Guard the close and release the source: 'finally { imageSource?.release(); if (fileSource !== undefined) { fileIo.closeSync(fileSource); } }'.

### `HW-05-0114` - Three getLastWindow callbacks dereference data without checking err, and the ability's two window listeners are never released

- Category B, severity medium, confidence confirmed
- Features: OFFICE-20
- Document: `huawei_industry_tree/05_office/docs/20_note_with_camera.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/note_with_camera-0000002378064505
- Why: On a getLastWindow failure data is undefined and 'windowClass.getWindowProperties()' raises a TypeError inside the callback, so the initial breakpoint is never computed and the size listener is never attached - the responsive layout silently never starts. Separately the reference defines off counterparts for both events, and neither is called, so both subscriptions outlive the window stage.
- Fix: Check the error before using data in all three callbacks - 'if (err.code) { hilog.error(...); return; }' - and release both listeners in onWindowStageDestroy with off('windowSizeChange', callback) and off('avoidAreaChange', callback).

### `HW-05-0115` - drawImage is passed a value typed PixelMap | undefined with no guard, and the discarded PixelMap is never released

- Category A, severity medium, confidence confirmed
- Features: OFFICE-20
- Document: `huawei_industry_tree/05_office/docs/20_note_with_camera.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/note_with_camera-0000002378064505
- Why: drawImage's first parameter is a mandatory ImageBitmap or PixelMap, and the variable handed to it is explicitly typed as possibly undefined - the surrounding code sets it to undefined on both the insert and the delete path, so the undefined state is reachable rather than theoretical. Separately, setting tmpPix to undefined drops a full-resolution PixelMap decoded from the picked photo without calling release on it, so every image inserted into the note leaks its backing buffer.
- Fix: Guard the draw and release the source: 'drawImage() { const bitmap = this.tmpPix; if (!bitmap) { return; } this.myPen.beginPath(); this.myPen.drawImage(bitmap, ...); this.myPen.closePath(); bitmap.release(); this.tmpPix = undefined; }' and release on the delete path too.

### `HW-05-0119` - readSync and closeSync are passed this.playFile?.fd, whose type is number | undefined, where the fd parameter is mandatory

- Category A, severity medium, confidence confirmed
- Features: OFFICE-21
- Document: `huawei_industry_tree/05_office/docs/21_voice_insert_note.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_insert_note-0000002348459564
- Why: The optional chain yields undefined whenever playFile has not been opened, and both call sites pass that straight into a parameter the reference marks mandatory. The writeData callback at AudioRendererManager.ets:74 is registered in initRenderer, before any file has been opened by startRenderer, so the renderer can invoke it with playFile still undefined.
- Fix: Guard the field once and use the concrete descriptor: 'const file = this.playFile; if (!file) { return; } const actualReadLen = fileIo.readSync(file.fd, buffer, { offset: this.readOffset, length: readLen });' and 'if (this.playFile) { fileIo.closeSync(this.playFile); this.playFile = undefined; }'.

### `HW-05-0120` - None of the four audio event subscriptions is ever unregistered, although Audio Kit documents an off counterpart for each

- Category B, severity medium, confidence confirmed
- Features: OFFICE-21
- Document: `huawei_industry_tree/05_office/docs/21_voice_insert_note.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_insert_note-0000002348459564
- Why: Audio Kit defines an off counterpart for every one of these events, and both createOn and initRenderer can run more than once - createOn is called for each new recording. The register/unregister pair is therefore left open on every cycle. The readData and writeData handlers are the data paths themselves, so a duplicate registration would write or read the same buffer twice.
- Fix: Keep each callback in a field and release it before the corresponding release() call: 'this.audioCapturer?.off("readData", this.readDataCallback); this.audioCapturer?.off("stateChange", this.stateCallback);' in stopAndRelease, and 'this.renderer?.off("writeData", this.writeDataCallback); this.renderer?.off("stateChange", this.rendererStateCallback);' in releaseRenderer.

### `HW-05-0121` - The document says the audio data is supplied through setDataCallback, but that function writes a field the sample never reads

- Category E, severity medium, confidence confirmed
- Features: OFFICE-21
- Document: `huawei_industry_tree/05_office/docs/21_voice_insert_note.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_insert_note-0000002348459564
- Why: Step 2 is the document's account of how recording works, and it names a function that is dead code. A reader who calls setDataCallback to receive the PCM buffers gets nothing: the assignment is never consulted, and the buffers go straight to the file inside the on('readData') handler. The mechanism to document is on('readData').
- Fix: Describe the real path - 'audioCapturer.on("readData", callback)' writing each buffer to the sandbox .pcm file - and either delete setDataCallback and the dataCallBack field, or make the readData handler invoke this.dataCallBack so the setter has an effect.

### `HW-05-0125` - The contact card filters the whole contact array twice per row inside the build method

- Category C, severity medium, confidence confirmed
- Features: OFFICE-22
- Document: `huawei_industry_tree/05_office/docs/22_special_following.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/special_following-0000002382515945
- Why: Both filters run inside a @Builder, so they are re-evaluated on every rebuild of the card, and the second one runs once per contact row rather than once per group. For a real corporate directory that is a full scan of the contact array per rendered row, on the UI thread, to compute a value that does not change during the build.
- Fix: Filter once and reuse the result: 'const group = this.totalData.filter((item) => item.firstLetter === firstLetter);' before the ForEach, then use 'group' as the source and 'index !== group.length - 1' for the divider - or precompute a Map<string, ContactInfo[]> in aboutToAppear alongside contactGroupArray.

### `HW-05-0126` - None of the four ForEach loops declares a key generator, including the contact rows the checkboxes belong to

- Category C, severity medium, confidence confirmed
- Features: OFFICE-22
- Document: `huawei_industry_tree/05_office/docs/22_special_following.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/special_following-0000002382515945
- Why: With no key generator the framework falls back to a key that embeds the index and a JSON.stringify of the item, which for the contact rows means serialising a ContactInfo - including its Resource - on every rebuild of every row. Because the checkbox state is not part of the model either (HW-05-0124), any key change rebuilds the row and its checkbox from scratch.
- Fix: Supply a key for each loop: '}, (letter: string) => letter);' for the groups, '}, (item: string) => item);' for the alphabet index, and '}, (data: ContactInfo) => data.name);' for the contact rows - or add an id field to ContactInfo if names can repeat.

### `HW-05-0128` - EntryAbility subscribes to avoidAreaChange without a matching off in onWindowStageDestroy

- Category B, severity medium, confidence confirmed
- Features: OFFICE-22
- Document: `huawei_industry_tree/05_office/docs/22_special_following.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/special_following-0000002382515945
- Why: The subscription is created for each window stage and never released, so the register/unregister pair the reference defines stays open and the callback keeps holding the window and writing into AppStorage for the lifetime of the process.
- Fix: Keep the callback in a field and call windowClass.off('avoidAreaChange', callback) in onWindowStageDestroy.

### `HW-05-0132` - Neither capturer subscription is unregistered, although Audio Kit documents an off counterpart for both events

- Category B, severity medium, confidence confirmed
- Features: OFFICE-23
- Document: `huawei_industry_tree/05_office/docs/23_voice_input_notes.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/voice_input_notes-0000002351441712
- Why: Audio Kit defines an off counterpart for both events and createOn runs once per recording, so the register/unregister pair is left open on every cycle. readData is the data path itself, so a duplicate registration would write each captured buffer to the file twice and double-count the level samples.
- Fix: Keep both callbacks in fields and release them before the capturer is released: 'this.audioCapturer?.off("readData", this.readDataCallback); this.audioCapturer?.off("stateChange", this.stateChangeCallback);' at the start of stopAndRelease.

### `HW-05-0133` - The keyboardHeightChange subscription is never released and the getLastWindow promise has no rejection handler; the page declares no aboutToDisappear at all

- Category B, severity medium, confidence confirmed
- Features: OFFICE-24
- Document: `huawei_industry_tree/05_office/docs/24_app_watermark.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_watermark-0000002353053774
- Why: The reference defines an off counterpart for this event, and the subscription is created in aboutToAppear with nothing to remove it, so the callback outlives the page and keeps writing into its @State keyboardHeight - which in turn drives the @Watch handler that scrolls the message list. Separately the getLastWindow rejection is unhandled, so a failure there silently leaves both the keyboard tracking and the watermark text unset.
- Fix: Keep the window and the callback in fields, add an aboutToDisappear that calls currentWindow.off('keyboardHeightChange', callback), and attach a .catch((err: BusinessError) => ...) to the getLastWindow chain.

### `HW-05-0134` - The page switches the keyboard avoid mode to RESIZE in aboutToAppear and never restores it

- Category B, severity medium, confidence probable
- Features: OFFICE-24
- Document: `huawei_industry_tree/05_office/docs/24_app_watermark.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_watermark-0000002353053774
- Why: setKeyboardAvoidMode is called on the UIContext, which is scoped to the window rather than to this component, so the RESIZE behaviour this chat page wants stays in force for every other page shown in the same window after it. The precise blast radius depends on how the application routes between pages, which is why this is recorded as probable; the missing restore is unambiguous.
- Fix: Record the previous mode with getKeyboardAvoidMode() in aboutToAppear and restore it in aboutToDisappear, or set the mode once at ability start-up if RESIZE is wanted application-wide.

### `HW-05-0135` - The chat message ForEach declares no key generator although the message list grows as the user sends

- Category C, severity medium, confidence confirmed
- Features: OFFICE-24
- Document: `huawei_industry_tree/05_office/docs/24_app_watermark.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_watermark-0000002353053774
- Why: The default key embeds both the index and a JSON.stringify of the ChatFormat item, so every append re-serialises every message in the list on the UI thread, and the index component makes the keys of later messages depend on their position. A chat transcript is exactly the case the guide's warning is about - it grows at the end and is re-rendered on each send.
- Fix: Give ChatFormat a stable id and key on it: '}, (messages: ChatFormat) => messages.id);' - or, if the messages are guaranteed distinct, key on the message text itself.

### `HW-05-0137` - The ComponentContent is removed from the OverlayManager but never disposed, which the reference names as a memory-leak risk

- Category B, severity medium, confidence confirmed
- Features: OFFICE-25
- Document: `huawei_industry_tree/05_office/docs/25_pinned_notice.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pinned_notice-0000002355181168
- Why: removeComponentContent detaches the node from the overlay, but the ComponentContent object still holds its reference to the backend entity node - which is exactly the state the reference tells developers to avoid by calling dispose(). The Params object handed to the builder additionally captures the page's NavPathStack, the OverlayManager and the overlayContent array, so the undisposed node keeps that whole graph reachable after the page is gone.
- Fix: Dispose after removing: 'aboutToDisappear(): void { const componentContent = this.overlayContent.pop(); if (componentContent) { this.overlayNode.removeComponentContent(componentContent); componentContent.dispose(); } }'.

### `HW-05-0139` - EntryAbility subscribes to avoidAreaChange without a matching off in onWindowStageDestroy

- Category B, severity medium, confidence confirmed
- Features: OFFICE-25
- Document: `huawei_industry_tree/05_office/docs/25_pinned_notice.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pinned_notice-0000002355181168
- Why: The subscription is created for each window stage and never released, so the register/unregister pair the reference defines stays open and the callback keeps holding the window and writing into AppStorage for the lifetime of the process.
- Fix: Keep the callback in a field and call windowClass.off('avoidAreaChange', callback) in onWindowStageDestroy.

### `HW-05-0140` - The breadcrumb snippet drops the guard and the list assignment the sample has, so tapping the current level pops a trail entry that is still displayed

- Category E, severity medium, confidence confirmed
- Features: OFFICE-26
- Document: `huawei_industry_tree/05_office/docs/26_organization_structure.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/organization_structure-0000002388806441
- Why: A do-while executes its body before testing, so without the surrounding if the last breadcrumb - where index already equals selectedStructure.length - 1 - still pops once. The trail then loses the level whose children the lower list is showing, and every subsequent index comparison in the same ForEach is off by one. The snippet also omits 'this.menuItemList = item.sonStructure', which is what actually moves the lower list back to the chosen level, so as printed the tap changes the breadcrumb and nothing else.
- Fix: Show the shipped form: assign this.menuItemList = item.sonStructure first, then wrap the do-while in 'if (index !== this.selectedStructure.length - 1) { ... }'.

### `HW-05-0141` - The drill-down snippet gates on this.level, a field that exists nowhere in the sample

- Category E, severity medium, confidence confirmed
- Features: OFFICE-26
- Document: `huawei_industry_tree/05_office/docs/26_organization_structure.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/organization_structure-0000002388806441
- Why: Step 3 is the document's only description of the drill-down, and as printed it does not compile - this.level is undeclared - and it describes the wrong rule. Whether a row can be entered depends on whether that row has children, which is also what makes the chevron appear; a page-level depth counter would let the user tap a leaf and replace the list with an empty array. The snippet also omits the push onto selectedStructure, so the breadcrumb would never grow.
- Fix: Replace the condition with the shipped one and include the trail push: 'if (item.sonStructure.length > 0) { this.menuItemList = item.sonStructure; this.selectedStructure.push(item); }'.

### `HW-05-0142` - Neither of the two linked lists declares a ForEach key generator although the model carries an id

- Category C, severity medium, confidence confirmed
- Features: OFFICE-26
- Document: `huawei_industry_tree/05_office/docs/26_organization_structure.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/organization_structure-0000002388806441
- Why: The default key runs JSON.stringify over each Structure, and Structure is recursive - sonStructure holds the entire subtree - so serialising one root item walks the whole organisation on every rebuild, on the UI thread. Both lists rebuild on every navigation step, since selectedStructure is pushed and popped and menuItemList is reassigned. The index component of the default key also makes the breadcrumb keys shift on every pop, which is precisely the case the guide warns about.
- Fix: Key both loops on the model's id: '}, (item: Structure) => item.id.toString());'.

### `HW-05-0146` - The getCalendar and getEvents callbacks declare an error parameter and never read it, dereferencing the data argument instead

- Category B, severity medium, confidence confirmed
- Features: OFFICE-27
- Document: `huawei_industry_tree/05_office/docs/27_multi_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/multi_schedule-0000002356954036
- Why: In an AsyncCallback the data argument is undefined when the call failed, so 'calendar = data; calendar.getEvents(...)' raises a TypeError inside the callback and 'data.length' does the same in the inner one. Because both run inside the per-item loop, one failure aborts that item silently - the batch continues and the user is told nothing. The correct shape is written twice in the very helper this code calls.
- Fix: Check the error first in both callbacks, as addEvent already does: 'if (err) { hilog.error(...); return; }' before using data, and rename the inner parameters so they do not shadow the outer ones.

### `HW-05-0150` - A single RdbPredicates object is reused across every iteration of the initialisation loop, so the accumulated AND conditions make the per-page lookup impossible to satisfy from the second page onwards

- Category B, severity medium, confidence confirmed
- Features: OFFICE-28
- Document: `huawei_industry_tree/05_office/docs/28_add_delete_annotations_in_pdf_preview.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_delete_annotations_in_pdf_preview-0000002384668918
- Why: On page 0 the predicate is `pageIndex = 0`; on page 1 it becomes `pageIndex = 0 AND pageIndex = 1`, which no row can satisfy, and it keeps growing. The existence check at line 94 therefore always takes the else branch from page 1 onwards, so the update path is unreachable and the code would insert a duplicate row for any page that already had one. It is masked today only because the guard at lines 68-72 returns early whenever the table is non-empty, so the loop runs on an empty table; a partially initialised table - an insert that failed at line 99, or a document whose page count grew - produces duplicate ANNOTATION rows, after which delAllAnn and saveAllAnnotation act on whichever duplicate the cursor reaches first.
- Fix: Move the construction inside the loop: `let pagePredicates = new relationalStore.RdbPredicates('ANNOTATION'); pagePredicates.equalTo('pageIndex', pageIndex); let resultSet = await store.query(pagePredicates, [...]);` and use pagePredicates for the update as well, leaving the unconditioned predicate at line 68 for the initial emptiness check only.

### `HW-05-0151` - The pdfService.PdfDocument opened to count the original annotations is never released, so the PDF stays loaded for the whole life of the page while a second copy of it is loaded into the PdfView controller

- Category B, severity medium, confidence probable
- Features: OFFICE-28
- Document: `huawei_industry_tree/05_office/docs/28_add_delete_annotations_in_pdf_preview.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_delete_annotations_in_pdf_preview-0000002384668918
- Why: The document is only needed for the one-off count in aboutToAppear - after that, every read goes through the controller. Holding a second parsed copy of the same PDF in native memory for the whole life of the page doubles the document's footprint for no benefit, and it is exactly the pattern the official guides avoid by calling releaseDocument once the work is finished. Marked probable because the reference page for pdfService (documentation/harmonyos-references/06_application-services/pdf-arkts-pdfservice.md) is a stub in this repository and does not state the lifetime rule in words; the evidence is the consistent behaviour of five official guide samples.
- Fix: Call pdfDocument.releaseDocument() at the end of getAnnNumOfEachPage, in a finally block so it also runs when the loop throws, and drop the pdfDocument field from Index.ets since nothing else uses it: `try { ... } finally { pdfDocument.releaseDocument(); }`.

### `HW-05-0153` - The document credits @ohos.file.fs with reading the original PDF byte stream, but the sample reads it with ResourceManager, and neither the reference list nor the project tree mentions the mandatory rawfile/a.pdf asset

- Category E, severity medium, confidence confirmed
- Features: OFFICE-28
- Document: `huawei_industry_tree/05_office/docs/28_add_delete_annotations_in_pdf_preview.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_delete_annotations_in_pdf_preview-0000002384668918
- Why: A reader following 实现思路 step 1 will look for a file.fs API that yields the bytes of a bundled document and will not find one, because the bundled document is reached through context.resourceManager.getRawFileContentSync. The omission of rawfile/a.pdf compounds it: the sample cannot run at all without that asset, and the only place this prerequisite is stated is a source-code comment inside the ZIP, not the document.
- Fix: Reword step 1 as "通过context.resourceManager.getRawFileContentSync读取resources/rawfile/a.pdf的字节流，再通过@ohos.file.fs写入应用沙箱" ("Read the byte stream of resources/rawfile/a.pdf with context.resourceManager.getRawFileContentSync, then write it into the application sandbox with @ohos.file.fs"), add the rawfile/a.pdf line to 工程目录, and add https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager to 参考文档.

### `HW-05-0157` - Employee name, staff number, department and position are interpolated into the hilog message string, which prints identifiable personal data in plaintext and bypasses the privacy filter

- Category D, severity medium, confidence confirmed
- Features: OFFICE-29
- Document: `huawei_industry_tree/05_office/docs/29_call_identity_delivery_employee_info.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/call_identity_delivery_employee_info-0000002419483593
- Why: In the extension the log line is written on every incoming call, and the record it prints was selected by that caller's phone number - so the log associates a phone number with a named employee, their staff number, their department and their job title. String interpolation happens before hilog sees the text, so there is no way to mark those values private; anyone who can read the device log reads the corporate directory entry in the clear. The rest of both files uses the argument form correctly (for example EntryAbility.ets:62 `'Failed to load the content. Cause: %{public}s', JSON.stringify(err)`), so the privacy-sensitive lines are the only ones that bypass the mechanism.
- Fix: Drop the personal fields from the log, for example `hilog.info(DOMAIN, 'testTag', 'caller info matched, id=%{public}d', id);`. If a value must appear, pass it as an argument with the default privacy identifier: `hilog.info(DOMAIN, 'testTag', 'contactName=%{private}s', this.contactName);`.

### `HW-05-0159` - 约束与限制 requires API Version 20 and the HarmonyOS 6.0.0 SDK, but the sample project is configured for compatibleSdkVersion 5.0.3(15) and declares no targetSdkVersion

- Category E, severity medium, confidence confirmed
- Features: OFFICE-29
- Document: `huawei_industry_tree/05_office/docs/29_call_identity_delivery_employee_info.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/call_identity_delivery_employee_info-0000002419483593
- Why: A reader who trusts 约束与限制 will assume the project targets API 20 and may raise compatibleSdkVersion to match, or will fail to understand why the project as downloaded builds against a three-release-older SDK. The missing targetSdkVersion is the more concrete problem: without it the product has no declared target API level, so the build falls back to a default rather than the value the document promises.
- Fix: Either update CallerInfoProviderDemo.zip#build-profile.json5 to `"targetSdkVersion": "6.0.0(20)", "compatibleSdkVersion": "6.0.0(20)"` to match 约束与限制, or correct 约束与限制 to state the version the project actually builds against, noting that CallerInfoQueryExtensionAbility is available since 5.0.2(14).

### `HW-05-0160` - The document never says that the test phone number has to be hardcoded into EntryAbility.ets, so the sample as downloaded seeds an employee record for the literal number 'xxx' and can never match a real call

- Category E, severity medium, confidence confirmed
- Features: OFFICE-29
- Document: `huawei_industry_tree/05_office/docs/29_call_identity_delivery_employee_info.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/call_identity_delivery_employee_info-0000002419483593
- Why: Nothing in the document tells the reader which file and which line to edit, and the only hint is a Chinese comment inside the ZIP. A reader who follows 实现思路 and 工程目录 to the letter installs the app, enables the caller-ID switch as instructed in step 10, places a test call and sees nothing happen, with no error anywhere - the query simply finds no row for the calling number.
- Fix: Add to step 2: "运行前需将entry/src/main/ets/entryability/EntryAbility.ets中的phoneNumber字段（默认值'xxx'）改为用于测试的真实来电号码，该号码同时作为初始化数据的主键写入EMPLOYEE表。" ("Before running, change the phoneNumber field in entry/src/main/ets/entryability/EntryAbility.ets - default value 'xxx' - to the real number used for testing; this number is also written to the EMPLOYEE table as the key of the seeded record.")

### `HW-05-0162` - The document and the sample both target API 20 yet use on('windowStageEvent'), whose lifecycle ordering is explicitly not guaranteed, in a feature that counts ordered foreground/background transitions

- Category A, severity medium, confidence confirmed
- Features: OFFICE-30
- Document: `huawei_industry_tree/05_office/docs/30_exam_cut_backstage_tips.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/exam_cut_backstage_tips-0000002431623674
- Why: This is precisely a case where the order of states matters. Note4ScreenSwitch.zip#entry/src/main/ets/pages/ExamPage.ets:87-100 decrements a counter on every SHOWN->HIDDEN/PAUSED edge and ends the exam once it runs out, so a state pair delivered out of order either misses a switch the invigilator must see or charges the candidate for one that did not happen. The API the guide recommends for exactly this situation is available in the version the sample targets.
- Fix: Replace the registration in EntryAbility.ets with `windowStage.on('windowStageLifecycleEvent', (data: window.WindowStageLifecycleEventType) => { ... })` and unregister with `windowStage.off('windowStageLifecycleEvent')` in onWindowStageWillDestroy. Note that the newer event does not report focus gain/loss; if that is needed, add on('windowEvent') as the guide directs. Update the code block in 实现思路 step 1 to match.

### `HW-05-0163` - Neither the windowStageEvent listener nor the windowStatusChange listener is ever unregistered, and onWindowStageWillDestroy is not implemented at all

- Category B, severity medium, confidence confirmed
- Features: OFFICE-30
- Document: `huawei_industry_tree/05_office/docs/30_exam_cut_backstage_tips.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/exam_cut_backstage_tips-0000002431623674
- Why: Both callbacks capture the ability and write into AppStorage. They outlive the window stage they were registered on, so after the ability is destroyed and recreated the app accumulates listeners that all fire on the next foreground/background transition and all write 'isOnForeground' - the exam page's @Watch then runs once per stale listener, decrementing the remaining-switch counter several times for a single switch and ending the exam early. The guide names onWindowStageWillDestroy as the place to undo both registrations, and the sample does not implement it.
- Fix: Keep references to the window stage and the main window, then add `onWindowStageWillDestroy(windowStage: window.WindowStage): void { try { windowStage.off('windowStageEvent'); this.windowClass?.off('windowStatusChange'); } catch (err) { hilog.error(DOMAIN, 'testTag', 'Failed to unsubscribe. Code is %{public}d', (err as BusinessError).code); } }`.

### `HW-05-0164` - Off-by-one in the switch-to-background allowance: the document and the on-screen warning both promise that the paper is submitted after 5 switches, but the code submits on the 6th

- Category B, severity medium, confidence confirmed
- Features: OFFICE-30
- Document: `huawei_industry_tree/05_office/docs/30_exam_cut_backstage_tips.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/exam_cut_backstage_tips-0000002431623674
- Why: Counting through it: switches 1 to 5 take the counter 5,4,3,2,1,0 - all of them pass the `< 0` test, and the fifth one notifies the candidate with '剩余0次' ("0 times remaining"). Only the sixth switch reaches -1 and submits. The candidate is therefore allowed one more switch than both the document and the warning banner say, and a real invigilation policy of exactly five would be silently violated.
- Fix: Change the guard to `if (this.examParam.leftSwitchScreenCounts < CommonConstant.NUM_1)` before the decrement, or keep the decrement and test `<= CommonConstant.NUM_0` after it: `this.examParam.leftSwitchScreenCounts--; if (this.examParam.leftSwitchScreenCounts <= 0) { this.submitExam(); return; }`.

### `HW-05-0165` - SoundPool creation is fired without await and playback is not gated on the loadComplete callback, against the sequence the SoundPool guide and the sample's own comment require

- Category B, severity medium, confidence confirmed
- Features: OFFICE-30
- Document: `huawei_industry_tree/05_office/docs/30_exam_cut_backstage_tips.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/exam_cut_backstage_tips-0000002431623674
- Why: The window between entering the exam page and the sound being decoded is exactly when a candidate is most likely to leave the app, and nothing in the code prevents play from running in it. If the switch happens before create() resolves, `SoundPoolUtil.soundPool` is still undefined and the non-null assertion at NoteUtils.ets:241 throws inside an async static with no try/catch; if it happens after createSoundPool but before load resolves, `soundId` is undefined and play is called out of the documented sequence, which the guide says produces an exception or undefined behaviour. Either way the audible warning the whole feature exists to deliver is the part that fails.
- Fix: Make the page await the setup - `await SoundPoolUtil.create();` in the async aboutToAppear - and have loadCallback record readiness: `SoundPoolUtil.soundPool.on('loadComplete', (soundId_: number) => { SoundPoolUtil.soundId = soundId_; SoundPoolUtil.isLoaded = true; })`, then guard playback with `if (!SoundPoolUtil.isLoaded) { return; }` at the top of PlaySoundPool.

### `HW-05-0172` - The exported deep link writes an event straight into the system calendar with no validation and no user confirmation, contradicting the document's own 参数合法性校验 claim

- Category D, severity medium, confidence confirmed
- Features: OFFICE-31
- Document: `huawei_industry_tree/05_office/docs/31_schedule_share.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule_share-0000002573150445
- Why: Any app or web page on the device can open link://shareSchedule?Action=Create&title=...&startTime=...&endTime=... and silently add an event to the user's system calendar, with a title it controls, without the user ever seeing an import prompt. The absence of validation makes it worse rather than better: `urlObject.params.get('title')?.toString() + ''` yields the literal string 'undefined' when the parameter is missing, and `new Date('undefined').getTime()` in ScheduleUtils.initEvent yields NaN for startTime and endTime, so a malformed link produces an event titled 'undefined' with invalid timestamps. The routing decision is equally weak: EntryAbility.ets:108-110 `isGroupSchedule(scheduleName: string): boolean { return scheduleName.includes('团队'); }` classifies the schedule by searching the attacker-supplied title for a substring.
- Fix: In OnServiceCardRouterMessage, reject the want unless title, startTime and endTime are all present and parse to finite timestamps - `const title = urlObject.params.get('title'); const start = Date.parse(urlObject.params.get('startTime') ?? ''); if (!title || Number.isNaN(start)) { return; }` - and then route the parsed schedule into the existing ReceiveShareDialog so the user confirms the import instead of writing straight through ScheduleUtils.addEvent.

### `HW-05-0173` - When the system-calendar write fails the save flow shows the failure toast and the success toast one after the other, and stores the schedule with an undefined eventId that can never be reconciled

- Category B, severity medium, confidence confirmed
- Features: OFFICE-31
- Document: `huawei_industry_tree/05_office/docs/31_schedule_share.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule_share-0000002573150445
- Why: The user is told 创建系统日程失败 ("failed to create the system schedule") and immediately afterwards 新建成功 ("created successfully"), and the dialog closes as though nothing were wrong. The schedule is then persisted with eventId undefined, so updateEvent skips it and deleteEvent's `if (eventId)` guard skips it too - the app has a schedule that can never be pushed to, edited in, or removed from the system calendar, and no path ever retries the write. Given that this is the state the sample lands in whenever the calendar handle is not ready yet, it is the normal outcome of the first save rather than a rare one.
- Fix: Return after the failure toast and leave the dialog open so the user can retry: `if (!eventId) { this.getUIContext().getPromptAction().showToast({ message: $r('app.string.create_sys_schedule_fail') }); return; }`. If saving locally without the system event is intentional, mark the record as unsynced and retry the write when the calendar handle becomes available, instead of showing the success toast.

### `HW-05-0174` - The calendar permission flow ignores dialogShownResults, never escalates to requestPermissionOnSetting, and gives the user no feedback when the permission is refused

- Category D, severity medium, confidence confirmed
- Features: OFFICE-31
- Document: `huawei_industry_tree/05_office/docs/31_schedule_share.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule_share-0000002573150445
- Why: READ_CALENDAR and WRITE_CALENDAR are user_grant permissions, so a refusal is permanent for every subsequent requestPermissionsFromUser call - the system stops showing the dialog and returns the refusal immediately. From that point the app can never sync anything, and the user is never told why: no toast, no banner, only 'Calendar permission denied' in the log. The feature the whole document is built on silently stops working with no route to recovery, which is exactly the situation requestPermissionOnSetting exists for.
- Fix: Follow the ladder used elsewhere in this industry: check `result.authResults` for the grant, then `if (result.dialogShownResults?.some(shown => !shown))` call `await atManager.requestPermissionOnSetting(context, permissions)` and re-check the returned statuses; on a final refusal raise a toast explaining that calendar sync is unavailable until the permission is granted in Settings.

### `HW-05-0175` - The windowSizeChange listener registered on the main window is never unregistered and onWindowStageDestroy is empty

- Category B, severity medium, confidence confirmed
- Features: OFFICE-31
- Document: `huawei_industry_tree/05_office/docs/31_schedule_share.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule_share-0000002573150445
- Why: The callback closes over the ability instance and over this.uiContext, so it keeps both alive after the window stage is gone. When the ability is destroyed and recreated the app accumulates one listener per lifetime, each writing breakpoint state into the shared WindowUtil singleton on every resize, and each holding a UIContext belonging to a destroyed window. The project's own share and eventHub registrations show the author knew the rule; the window listener is the one that was missed.
- Fix: Keep the window handle on the ability and unregister it: `private windowClass?: window.Window;` assigned in the getMainWindow callback, then `onWindowStageWillDestroy(): void { try { this.windowClass?.off('windowSizeChange', this.onWindowSizeChange); } catch (err) { hilog.error(DOMAIN, 'testTag', 'Failed to unsubscribe windowSizeChange. Code is %{public}d', (err as BusinessError).code); } }`.

### `HW-05-0176` - The deep-link handler is invoked from onCreate, where this.uiContext is still undefined, so every toast it raises is silently dropped on a cold start

- Category B, severity medium, confidence confirmed
- Features: OFFICE-31
- Document: `huawei_industry_tree/05_office/docs/31_schedule_share.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule_share-0000002573150445
- Why: onCreate always precedes onWindowStageCreate, and the assignment happens later still, inside an asynchronous window callback. On a cold start from a link - the case the feature exists for - the optional chain short-circuits and the user is told nothing at all, whether the schedule was created or the system-calendar write failed. The same call also invokes `RouterModule.closeDialog({})` at EntryAbility.ets:155 and :166 before any page has been loaded. On a hot start through onNewWant the field happens to be set, so the behaviour differs between the two launch paths for no stated reason.
- Fix: Keep the want in onCreate and process it once the content is loaded - store it in a field, and in the windowStage.loadContent callback (or after this.uiContext is assigned) run OnServiceCardRouterMessage with the saved want. The knock-share path already shows the pattern the router path should follow: it stores the payload in AppStorage and emits an eventHub event that ScheduleView subscribes to at ScheduleShare.zip#ablility/schedule/src/main/ets/schedule/ScheduleView.ets:87.

### `HW-05-0177` - updateEvent guards on the app-local schedule id but sends the system event id, so it calls the Calendar API with id undefined whenever the original write did not return one

- Category B, severity medium, confidence confirmed
- Features: OFFICE-31
- Document: `huawei_industry_tree/05_office/docs/31_schedule_share.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule_share-0000002573150445
- Why: Every local save calls updateEvent immediately afterwards - NewSchedule.ets:80-82 and EntryAbility.ets:162-164 both do `RequestProxy.saveOne(this.schedule).then(() => { ScheduleUtils.updateEvent(this.schedule); });`. When the app-local id is set but the system event id is not, the guard passes and the call goes out with `id: undefined`, so the Calendar API is asked to update an unidentified event; the rejection is logged with console.error and nothing else. Guarding on the identifier that is actually sent would skip the call instead.
- Fix: Change the guard to the system id and make the absence explicit: `public static updateEvent(schedule: ScheduleInfo) { if (!schedule.eventId) { hilog.warn(0x0000, TAG, 'Skip updateEvent: schedule has no system event id'); return; } const updateEvent = ScheduleUtils.initEvent(schedule); updateEvent.id = schedule.eventId; ... }`.

### `HW-05-0179` - 工程目录 shows the schedule module under 'ability/', but the directory in the sample is misspelt 'ablility/', and both build-profile.json5 and entry/oh-package.json5 depend on that spelling

- Category E, severity medium, confidence confirmed
- Features: OFFICE-31
- Document: `huawei_industry_tree/05_office/docs/31_schedule_share.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule_share-0000002573150445
- Why: 工程目录 is the map a reader uses to recreate the project or to navigate the ZIP. A reader who creates ability/schedule as the document shows, then copies build-profile.json5 and oh-package.json5 from the sample, gets a module whose srcPath and file: dependency point at a directory that does not exist, and the build fails to resolve the schedule HAR. The reverse is just as bad: a reader searching the extracted ZIP for the path the document gives finds nothing.
- Fix: Rename the directory in the sample to ability/schedule and update ScheduleShare.zip#build-profile.json5 to `"srcPath": "./ability/schedule"` and ScheduleShare.zip#entry/oh-package.json5 to `"schedule": "file:../ability/schedule"`.

### `HW-05-0180` - 工程目录 describes ReminderDialog.ets only as the reminder-time dialog although it also contains the entire share and receive flow, and lists ScheduleAbility.ets as the module's UIAbility although the module is a HAR that declares no abilities

- Category E, severity medium, confidence confirmed
- Features: OFFICE-31
- Document: `huawei_industry_tree/05_office/docs/31_schedule_share.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule_share-0000002573150445
- Why: 工程目录 is the only index the document gives, and both entries send the reader to the wrong place. Someone looking for the carbon copy of the share code quoted in 实现思路 step 3 will look under a file the tree calls a reminder-time picker and will not think to open it; someone reading the tree will believe the schedule module hosts its own UIAbility and will look for a module.json5 entry that does not exist. The ScheduleAbility file is dead code in a HAR, which is a design statement the tree presents as fact.
- Fix: Change line 377 to `├── ReminderDialog.ets  // 提醒时间选择弹窗、分享确认弹窗、接收导入弹窗` ("reminder-time dialog, share-confirmation dialog and receive-import dialog"), or split the three structs into separate files matching the tree. Delete scheduleability/ScheduleAbility.ets together with line 383, since a HAR cannot register a UIAbility.

### `HW-05-0182` - The 行业常见问题 page of the office industry has no content at all - one redirect line pointing at the same generic URL every other industry redirects to, so there is no office-specific FAQ anywhere in the set

- Category E, severity medium, confidence confirmed
- Features: OFFICE-32
- Document: `huawei_industry_tree/05_office/docs/32_practice-office-app-architecture-v1_2.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1_2-0000002263504048
- Why: This is the last node of the 05_office industry tree and the only place a reader is told to look for the industry's known problems and their answers. Following it yields a shared, industry-agnostic FAQ index, so a developer working through the 29 office scenarios has nowhere to look up an office-specific question. Reviewers are equally stuck: the page carries no statement that can be checked against any sample, which is why this document is reviewable on the documentation and consistency axes only.
- Fix: Replace the redirect with a link to the office subsection of the HarmonyOS FAQ, or list the office-relevant FAQ entries inline - the questions raised repeatedly across this industry's samples are the two-stage user_grant permission ladder, which system pickers need no permission at all, unregistering window and share listeners in onWindowStageWillDestroy, and closing every relationalStore ResultSet.

### `HW-05-0186` - 1 sample project declares permissions that no code path in the project uses

- Category D, severity medium, confidence confirmed
- Features: OFFICE-08
- Document: `huawei_industry_tree/05_office/docs/08_file_view.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/file_view-0000002256155688
- Why: A permission declared in module.json5 but never referenced by any .ets or .ts file in the project cannot be exercised, so it is pure over-declaration. Over-declared permissions are a release-review rejection reason, and restricted permissions especially so. Because these module.json5 files are copied as templates, the surplus entries propagate into products that have even less claim to them.
- Fix: Delete every declared permission the code does not exercise. Where the capability is reached through a permission-free path such as SaveButton or PhotoViewPicker, no declaration is needed at all.

### `HW-05-0006` - Reminder.ets calls preferences flush() without awaiting it or attaching a rejection handler

- Category B, severity low, confidence confirmed
- Features: OFFICE-01
- Document: `huawei_industry_tree/05_office/docs/01_practice-office-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1-0000001965211649
- Why: flush() returns a Promise. Calling it without await, .then() or .catch() leaves the rejection unhandled, so a failure to persist the schedule list is invisible and the UI still shows the 'added' toast issued two lines later at Reminder.ets:306.
- Fix: Await the promise inside an async handler, or chain .then()/.catch() and only show the success toast after the flush resolves.

### `HW-05-0009` - The document states the app is deployed on phones only, but the sample entry module declares phone, tablet and 2in1 device types

- Category E, severity low, confidence confirmed
- Features: OFFICE-01
- Document: `huawei_industry_tree/05_office/docs/01_practice-office-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1-0000001965211649
- Why: The project tree and the deployment scope are part of the architecture contract the document defines. A reader who follows the phone-only statement will not test the tablet and 2in1 layouts that the shipped module.json5 actually enables, and the extra device types also change the store distribution scope.
- Fix: Either narrow entry/src/main/module.json5 deviceTypes to ["phone"] so it matches the stated architecture, or update the document to state that the framework ships for phone, tablet and 2in1.

### `HW-05-0010` - The module description of the mine HAR does not match its content in the sample project

- Category E, severity low, confidence confirmed
- Features: OFFICE-01
- Document: `huawei_industry_tree/05_office/docs/01_practice-office-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1-0000001965211649
- Why: The four bullets under 代码结构解读 are the document's mapping between the module names and their responsibilities. Describing mine with the message list and the web page contradicts both the tree printed immediately below it (which lists BookPage/InformationPage/SelectRegionPage under mine) and the shipped ZIP, so the module split cannot be reproduced from the text.
- Fix: Rewrite the bullet as: mine is the contacts/personal-information HAR; it contains the organisation contact list (BookPage), the contact detail page (InformationPage) and the region selection page (SelectRegionPage).

### `HW-05-0011` - ConversationDetail switches the window-wide keyboard avoid mode to RESIZE in aboutToAppear and never restores it

- Category B, severity low, confidence probable
- Features: OFFICE-01
- Document: `huawei_industry_tree/05_office/docs/01_practice-office-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1-0000001965211649
- Why: setKeyboardAvoidMode is called on the UIContext of the main window, so it applies to the whole window and not to this NavDestination. After the chat page is popped, every other page of the app keeps the RESIZE behaviour that only the chat page needs.
- Fix: Record the previous mode with getKeyboardAvoidMode() in aboutToAppear and restore it in aboutToDisappear, or set the mode once in EntryAbility if RESIZE is intended application-wide.

### `HW-05-0017` - Both 实现思路 snippets swallow their exceptions while the sample project logs them

- Category E, severity low, confidence confirmed
- Features: OFFICE-03
- Document: `huawei_industry_tree/05_office/docs/03_location_permissions.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/location_permissions-0000002231804582
- Why: geoLocationManager.getCurrentLocation throws synchronously for invalid parameters and for a disabled location switch. A reader who copies the document's snippet loses those error codes entirely and has no way to distinguish 'no permission' from 'location service off'. The sample already shows the correct handling, so the guide is strictly worse than the code it documents.
- Fix: Bring the two document snippets in line with the sample by logging the caught error with hilog.error before returning.

### `HW-05-0020` - setWindowLayoutFullScreen is called inside an async function without await and without a rejection handler

- Category B, severity low, confidence confirmed
- Features: OFFICE-04
- Document: `huawei_industry_tree/05_office/docs/04_load_display_pdf.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/load_display_pdf-0000002270470565
- Why: The surrounding function is already async and awaits getMainWindow on the previous line, so the missing await is an omission rather than a constraint. Two consequences follow: the promise rejection is unhandled, and getWindowAvoidArea on the next line reads the avoid area before the full-screen layout switch has necessarily been applied, which is exactly the value the pages consume through AppStorage 'topHeight'.
- Fix: await the call - 'await WIN.setWindowLayoutFullScreen(true);' - inside a try/catch, then read the avoid area.

### `HW-05-0021` - The 防截屏 snippet in the document drops the getMainWindow error check that the sample performs

- Category E, severity low, confidence confirmed
- Features: OFFICE-04
- Document: `huawei_industry_tree/05_office/docs/04_load_display_pdf.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/load_display_pdf-0000002270470565
- Why: When getMainWindow fails, data is undefined and the document's snippet stores undefined under the AppStorage key 'windowClass'. Preview.ets:32 then reads that key and its guard at Preview.ets:38 silently skips the privacy-mode call, so the anti-screenshot protection is dropped with no diagnostic. The sample already handles this; the guide should not teach the weaker version.
- Fix: Add the error branch to the document snippet so it matches EntryAbility.ets:46-55: check err.code, log it and return before writing to AppStorage.

### `HW-05-0028` - The project tree describes CustomTextBox.ets as the component that displays the scan result, but the sample never imports it

- Category E, severity low, confidence confirmed
- Features: OFFICE-05
- Document: `huawei_industry_tree/05_office/docs/05_personal_card.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/personal_card-0000002236270234
- Why: The project tree is the document's map of the sample. Naming a file as the display component for the scan result when the shipped code uses a toast instead sends a reader looking for a wiring that does not exist, and leaves CustomTextBox as dead code that no reviewer would otherwise question.
- Fix: Either wire CustomTextBox({ inputValue: this.inputValue, scanType: String(this.scanType) }) into MainPage where the scan result is presented, or drop the file and its tree entry and describe the toast as the result display.

### `HW-05-0029` - Two hilog.info calls in EntryAbility pass a serialized object as the format string instead of as a variadic argument

- Category A, severity low, confidence confirmed
- Features: OFFICE-05
- Document: `huawei_industry_tree/05_office/docs/05_personal_card.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/personal_card-0000002236270234
- Why: The serialized Want lands in the format-string slot, so any % sequence it contains is interpreted as a format specifier, and because the content carries no {public} identifier it is treated as private data and filtered in the log output. The launch parameters that these two lines exist to record are therefore not reliably readable.
- Fix: Use the variadic form: hilog.info(0x0000, 'testTag', 'want: %{public}s', JSON.stringify(want)); and likewise for launchParam.

### `HW-05-0035` - The signature-board snippet dereferences the optional onTouch event without a guard and omits the Path2D creation the sample performs

- Category E, severity low, confidence confirmed
- Features: OFFICE-06
- Document: `huawei_industry_tree/05_office/docs/06_document_approval.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/document_approval-0000002280673593
- Why: Two problems follow from copying the snippet as printed. Reading event.type on a parameter typed 'TouchEvent | undefined' does not satisfy the declared type, and pushing this.tempPath without assigning a new Path2D first means every stroke appends the same object to pathArray - so the undo button described in the next paragraph removes an entry that is still being drawn into, and the redraw loop repeats one accumulating path.
- Fix: Show the guarded form from the sample: test 'event?.touches.length === 1' before use, and on TouchType.Down call beginPath(), assign 'this.tempPath = new Path2D()', push it, then moveTo the first point.

### `HW-05-0036` - FileComponent calls fs.accessSync outside any try/catch while the identical call in ApprovalPage is guarded

- Category B, severity low, confidence confirmed
- Features: OFFICE-06
- Document: `huawei_industry_tree/05_office/docs/06_document_approval.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/document_approval-0000002280673593
- Why: accessSync throws rather than returning false for conditions such as a permission failure on the path. Because the call sits directly in the click handler with no guard, that exception escapes the event callback, and the download button silently does nothing. The correct pattern already exists two files away in the same project.
- Fix: Wrap the accessSync call and the branch that follows it in a try/catch that logs through the project's Logger, mirroring ApprovalPage.ets:50-79.

### `HW-05-0044` - The mask snippet in 实现思路 references undeclared identifiers and puts an image resource inside a Span

- Category E, severity low, confidence confirmed
- Features: OFFICE-07
- Document: `huawei_industry_tree/05_office/docs/07_camera_page.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/camera_page-0000002245028040
- Why: This is the only code the document offers for step 1, the mask construction that the whole scenario is named after. As printed it does not compile: the identifiers do not exist, the field name does not match the sample, and a media resource cannot serve as Span text.
- Fix: Replace the snippet with the shipped expressions - '$r("app.media.front_photo_frame")' / '$r("app.media.reverse_photo_frame")' for the Image and '$r("app.string.ID_card_Front")' / '$r("app.string.ID_card_Reverse_side")' for the Span - and use the sample's field name isIDCardFront.

### `HW-05-0049` - The scenario description advertises png while the sample's file list offers jpg and repeats the txt entry twice

- Category E, severity low, confidence confirmed
- Features: OFFICE-08
- Document: `huawei_industry_tree/05_office/docs/08_file_view.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/file_view-0000002256155688
- Why: The 场景介绍 paragraph is the document's statement of what the sample demonstrates. It names a format the UI never offers, omits the one it does offer, and the sample itself has a duplicated txt row whose file name prefix ('png' + time + '文件.jpg') contradicts its own extension. A reader checking the five advertised formats against the running app cannot reconcile them.
- Fix: State the formats the sample actually exposes - pdf, txt, docx, xlsx and jpg - remove the duplicated .txt row from FileMainView, and rename the downloaded image to a 'jpg' prefix so the file name matches its extension.

### `HW-05-0050` - The project tree omits the entrybackupability directory that the sample ships and module.json5 declares

- Category E, severity low, confidence confirmed
- Features: OFFICE-08
- Document: `huawei_industry_tree/05_office/docs/08_file_view.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/file_view-0000002256155688
- Why: The project tree is the document's map of the sample, and this module contributes a declared ExtensionAbility plus a backup_config profile that a reader reproducing the project would otherwise leave out. The same industry documents the same directory elsewhere, so the omission is an inconsistency rather than a deliberate simplification.
- Fix: Add the entrybackupability/EntryBackupAbility.ets entry to the 工程目录 block, matching how 07_camera_page.md presents the same directory.

### `HW-05-0055` - One shared select field feeds the selected index of two different TextPicker dialogs, so choosing a reason changes the preselected area code

- Category B, severity low, confidence confirmed
- Features: OFFICE-09
- Document: `huawei_industry_tree/05_office/docs/09_guest_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/guest_demo-0000002257167990
- Why: The comment on both onAccept handlers states the intent: remember the chosen index so the dialog reopens on the previous selection. Because one field serves both dialogs, picking reason index 3 makes the area-code picker open on area index 3, and vice versa - and if the reason list is longer than the area list, the index passed as selected is out of range for the other picker.
- Fix: Give each dialog its own index field, for example 'private areaSelect: number | number[] = 0;' and 'private reasonSelect: number | number[] = 0;', and use each one only in its own showTextPickerDialog call.

### `HW-05-0061` - The Context type is imported from @ohos.abilityAccessCtrl, a module whose reference documents no such export

- Category A, severity low, confidence probable
- Features: OFFICE-10
- Document: `huawei_industry_tree/05_office/docs/10_email_attachment.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/email_attachment-0000002319115245
- Why: abilityAccessCtrl is the permission-management module; Context belongs to the ability framework. Importing an application type through an unrelated permission module is undocumented, ties this page to a module it otherwise never uses, and diverges from the convention used by every sibling sample. The specific compile behaviour was not verified here, which is why this is recorded as probable.
- Fix: Import the documented type instead: 'import { common } from "@kit.AbilityKit";' and declare 'private context: common.UIAbilityContext = this.getUIContext().getHostContext() as common.UIAbilityContext;'.

### `HW-05-0062` - The resourceManager.getStringValue promise in aboutToAppear has no rejection handler

- Category B, severity low, confidence confirmed
- Features: OFFICE-10
- Document: `huawei_industry_tree/05_office/docs/10_email_attachment.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/email_attachment-0000002319115245
- Why: getStringValue returns a Promise that rejects when the resource id cannot be resolved. Because this is the only place the attachment hint text is loaded, a rejection leaves the hint empty and produces an unhandled promise rejection instead of a log line. The same file already demonstrates the correct shape three times.
- Fix: Attach '.catch((err: BusinessError) => Logger.error(`getStringValue failed, code is ${err.code}, message is ${err.message}`))', matching the picker chains in the same file.

### `HW-05-0068` - utfFileNameToString splits the file name on the first dot, so any attachment whose name contains more than one dot is truncated and given the wrong extension

- Category B, severity low, confidence confirmed
- Features: OFFICE-11
- Document: `huawei_industry_tree/05_office/docs/11_conference_release.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/conference_release-0000002321751725
- Why: split('.')[0] keeps only the text before the first dot and split('.')[1] takes the second segment as the extension. For a picked file named 'meeting.2025.q3.pdf' the function returns 'meeting.2025' - the true extension pdf is dropped, the name is truncated, and the sandbox path built from it therefore has no usable extension, which in turn makes findMimeType return the default empty string.
- Fix: Take the extension from the last segment and the stem from everything before it, e.g. 'const dot = utfFileName.lastIndexOf("."); const stem = dot > 0 ? utfFileName.substring(0, dot) : utfFileName; const ext = dot > 0 ? utfFileName.substring(dot) : "";' and decode only the stem.

### `HW-05-0074` - The project tree names the wrong module root and mis-cases the page file relative to the shipped project

- Category E, severity low, confidence confirmed
- Features: OFFICE-12
- Document: `huawei_industry_tree/05_office/docs/12_pdf_add_mark.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pdf_add_mark-0000002324100453
- Why: The project tree is the document's map of the sample. The root path 'ets/src/main/ets' does not exist in the project - the module is 'entry' - and the file name differs in case from the shipped file while the build explicitly turns on case-sensitive path checking, so a reader reproducing the tree literally creates a path the project cannot resolve.
- Fix: Correct the first line to '├──entry/src/main/ets' and the page entry to 'AddMarkPage.ets', matching the shipped project.

### `HW-05-0075` - The 实现思路 snippets differ from the sample in the PdfView page-fit mode and drop the error logging the sample performs

- Category E, severity low, confidence confirmed
- Features: OFFICE-12
- Document: `huawei_industry_tree/05_office/docs/12_pdf_add_mark.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pdf_add_mark-0000002324100453
- Why: The two snippets are the document's only description of how the preview is configured and how the save is guarded. FIT_WIDTH and FIT_NONE produce visibly different layouts, so a reader cannot reproduce the screenshot from the text; and the empty handlers teach the weaker version of code the sample already gets right, hiding exactly the picker failures that HW-05-0072 depends on.
- Fix: Align the snippets with the shipped code: use pdfService.PageFit.FIT_NONE, and keep the Logger.error calls in both the .catch() and the enclosing catch.

### `HW-05-0084` - Two hilog calls pass an empty format string, so the value they exist to record is never printed

- Category A, severity low, confidence confirmed
- Features: OFFICE-14
- Document: `huawei_industry_tree/05_office/docs/14_text_marquee.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/text_marquee-0000002331949121
- Why: The format string carries the placeholders that consume the variadic arguments. With an empty format there is no identifier for the argument to map to, so the sending time and the tapped index are dropped and each call emits an empty log line. Both calls are the only instrumentation on the delete and avatar-tap handlers.
- Fix: Supply a format with a placeholder, e.g. hilog.info(0x0000, 'testTag', 'delete item sent at %{public}s', item.sendingTime); and hilog.info(0x0000, 'testTag', 'avatar tapped at index %{public}d', index);

### `HW-05-0089` - The message-span ForEach builds its key with two JSON.stringify calls plus the index

- Category C, severity low, confidence confirmed
- Features: OFFICE-15
- Document: `huawei_industry_tree/05_office/docs/15_later_items.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/later_items-0000002301534190
- Why: Both stringify calls are pure overhead on a value that is already a primitive, and they run for every span of every message bubble on each rebuild. Combining the serialised content with the index also reintroduces the index dependency the ForEach guide warns about, so inserting a span invalidates the keys of all the spans after it.
- Fix: Key on the content directly: '}, (content: string) => content;' - or add a stable id to the content model if duplicate strings within one bubble are possible.

### `HW-05-0092` - The step-3 snippet discards the picker result instead of binding it to the state the Image renders

- Category E, severity low, confidence confirmed
- Features: OFFICE-16
- Document: `huawei_industry_tree/05_office/docs/16_recommend_id_photos.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/recommend_id_photos-0000002344341185
- Why: Step 3 is titled 将用户选中的图片URI，在Image组件中展示 ("display the URI of the image selected by the user in the Image component"), and the assignment is exactly the link that makes that happen. As printed the snippet calls the picker, throws the URI away, and then shows an Image bound to a variable nothing ever set - so it does not demonstrate the step it heads.
- Fix: Show the assignment and the @State declaration it targets: '@State frontIdCard: (ResourceStr | undefined) = $r("app.media.ic_front_id_card");' and 'this.frontIdCard = await this.imagePicker(photoAccessHelper.RecommendationType.ID_CARD);'.

### `HW-05-0110` - The project tree names the component directory components while the sample ships it as component, under a case-sensitive build

- Category E, severity low, confidence confirmed
- Features: OFFICE-19
- Document: `huawei_industry_tree/05_office/docs/19_watermark_camera.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/watermark_camera-0000002334789536
- Why: The project tree is the document's map of the sample, and this entry names a directory that does not exist in the download. With caseSensitiveCheck enabled the distinction is enforced by the toolchain, so a reader who reproduces the tree literally and then copies the sample's import paths gets an unresolvable module.
- Fix: Correct the tree entry to '├──component', matching the shipped directory name.

### `HW-05-0127` - The project tree names a page directory and a component file that do not exist under the shipped case-sensitive build

- Category E, severity low, confidence confirmed
- Features: OFFICE-22
- Document: `huawei_industry_tree/05_office/docs/22_special_following.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/special_following-0000002382515945
- Why: The project tree is the document's map of the sample, and two of its entries name paths the download does not contain - one differing in case, one in the directory name. With caseSensitiveCheck enabled the toolchain enforces both, so a reader who reproduces the tree literally and then copies the sample's import statements gets unresolvable modules.
- Fix: Correct the two entries to '└──Title.ets' and '└──pages', matching the shipped project.

### `HW-05-0136` - The project tree names the date-utility directory utils while the sample ships it as util, under a case- and path-sensitive build

- Category E, severity low, confidence confirmed
- Features: OFFICE-24
- Document: `huawei_industry_tree/05_office/docs/24_app_watermark.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_watermark-0000002353053774
- Why: The project tree is the document's map of the sample, and this entry names a directory the download does not contain. With caseSensitiveCheck enabled the toolchain enforces the exact path, so a reader who reproduces the tree literally and then copies the sample's import statement gets an unresolvable module.
- Fix: Correct the tree entry to '└──util', matching the shipped directory and the import in WatermarkPage.ets.

### `HW-05-0138` - The project tree names a constants directory and a page directory that the sample does not contain

- Category E, severity low, confidence confirmed
- Features: OFFICE-25
- Document: `huawei_industry_tree/05_office/docs/25_pinned_notice.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pinned_notice-0000002355181168
- Why: The project tree is the document's map of the sample, and two of its directory entries name paths the download does not contain. With caseSensitiveCheck enabled the toolchain enforces the exact paths, so a reader who reproduces the tree literally and then copies the sample's import statements gets unresolvable modules.
- Fix: Correct the two entries to '├──common' and '├──pages', matching the shipped project and its imports.

### `HW-05-0143` - The project tree names the page directory page while the sample ships pages, under a path-sensitive build

- Category E, severity low, confidence confirmed
- Features: OFFICE-26
- Document: `huawei_industry_tree/05_office/docs/26_organization_structure.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/organization_structure-0000002388806441
- Why: The project tree is the document's map of the sample, and this entry names a directory the download does not contain. With caseSensitiveCheck enabled the toolchain enforces the exact path, so a reader who reproduces the tree literally gets a layout the project's own page profile cannot resolve.
- Fix: Correct the entry to '└──pages', matching the shipped project.

### `HW-05-0147` - requestCalendarPermission takes the current grant state as a parameter and overwrites it on its first statement, so the argument is dead

- Category B, severity low, confidence confirmed
- Features: OFFICE-27
- Document: `huawei_industry_tree/05_office/docs/27_multi_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/multi_schedule-0000002356954036
- Why: The signature suggests the caller's current grant state matters - a reader would expect an already-granted state to short-circuit the dialog - but the first statement discards it and the function always raises the authorization dialog. The parameter is dead weight that makes the contract misleading, and the function performs no checkAccessToken pre-check that would make an early exit possible.
- Fix: Drop the parameter - 'export async function requestCalendarPermission(uiContext: UIContext): Promise<boolean>' - or honour it by checking the current grant with atManager.checkAccessToken first and returning true without prompting when both permissions are already held.

### `HW-05-0152` - The hilog format strings that trace the annotation counts use %d and %s with no {public} identifier, so every page index and annotation count is printed as <private>

- Category B, severity low, confidence confirmed
- Features: OFFICE-28
- Document: `huawei_industry_tree/05_office/docs/28_add_delete_annotations_in_pdf_preview.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_delete_annotations_in_pdf_preview-0000002384668918
- Why: The privacy identifier defaults to {private}, so these lines reach the log as 'current page is <private>, raw_annotations_len is <private>.'. Those are precisely the values a developer needs to debug the annotation bookkeeping, and none of them is personal data. The project already writes the identifier correctly where it matters least - the plain string messages at Index.ets:83 and :141 carry no arguments at all - so the traces that would actually help are the ones that are blanked out.
- Fix: Add the identifier to every placeholder that carries a page index, a count or a control type, for example `hilog.info(0x0000, 'testTag', 'current page is %{public}d, raw_annotations_len is %{public}d.', pageIndex, annotations.length);`.

### `HW-05-0154` - The four user-facing toasts are hardcoded Chinese literals while every button label in the same file goes through the resource system

- Category B, severity low, confidence confirmed
- Features: OFFICE-28
- Document: `huawei_industry_tree/05_office/docs/28_add_delete_annotations_in_pdf_preview.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_delete_annotations_in_pdf_preview-0000002384668918
- Why: The four messages that report the outcome of the two operations the sample exists to demonstrate are the only ones that cannot be translated or edited without touching ArkTS code. Adding any locale directory to this project leaves the buttons localised and the result toasts in Chinese, and a reader copying this page into a shipping app inherits that split.
- Fix: Declare saveSuccess, saveFailed, delAllAnnSuccess and delAllAnnSaveFailed in entry/src/main/resources/base/element/string.json and use them, for example `showToast({ message: $r('app.string.saveSuccess') })`.

### `HW-05-0158` - The two flag traces use %s with no {public} identifier, so the only diagnostics that report whether the lookup succeeded print as <private>

- Category B, severity low, confidence confirmed
- Features: OFFICE-29
- Document: `huawei_industry_tree/05_office/docs/29_call_identity_delivery_employee_info.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/call_identity_delivery_employee_info-0000002419483593
- Why: These two booleans are the only signal a developer has for the two decisions the sample makes - whether the startup seeding is skipped and whether the incoming number was matched - and both reach the log as 'isQuery: <private>' and 'isSuccess: <private>'. Neither is personal data. The same files use the identifier correctly elsewhere, for example EntryAbility.ets:30 `'%{public}s', 'Ability onCreate'`.
- Fix: Write `hilog.info(DOMAIN, 'testTag', 'isQuery: %{public}s', String(this.isQuery));` and the same for isSuccess.

### `HW-05-0161` - 实现思路 step 10 asks the user to find the caller-ID switch by hand and omits the two switch-state APIs and the deep link documented on the Call Service Kit page the document links to

- Category E, severity low, confidence confirmed
- Features: OFFICE-29
- Document: `huawei_industry_tree/05_office/docs/29_call_identity_delivery_employee_info.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/call_identity_delivery_employee_info-0000002419483593
- Why: The feature is inert until the user turns the switch on, and the document offers no way for the app to notice that. A real deployment cannot ask every employee to walk a four-step settings path blind - it needs to query the switch state on startup and, when it is off, send the user straight to the page with openLink('callsetting://number_identity'). Leaving both out makes the sample undebuggable for anyone who misses the manual step and gives no clue why no caller information is displayed.
- Fix: Add a step after step 10: "应用可通过numberIdentify.queryNumberIdentifySwitchState(context)查询全局与应用级号码识别开关状态，通过isSupportEnterpriseNumberIdentify(context)判断企业号码识别是否可用；开关关闭时可使用context.openLink('callsetting://number_identity', { appLinkingOnly: false })跳转到号码识别设置页。" ("An app can query the global and app-level caller-ID switch states with numberIdentify.queryNumberIdentifySwitchState(context) and check whether enterprise caller ID is available with isSupportEnterpriseNumberIdentify(context); when the switch is off, use context.openLink('callsetting://number_identity', { appLinkingOnly: false }) to jump to the caller-ID settings page.") and add the numberIdentify reference to 参考文档.

### `HW-05-0166` - The main page sets the answer progress to 100% when the exam is over and overwrites it on the very next line, so the branch is dead

- Category B, severity low, confidence confirmed
- Features: OFFICE-30
- Document: `huawei_industry_tree/05_office/docs/30_exam_cut_backstage_tips.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/exam_cut_backstage_tips-0000002431623674
- Why: checkIfExam runs from aboutToAppear and from the page's onShown at MainPage.ets:484-487, so the assignment is executed and discarded every time. A candidate whose paper was auto-submitted after too many background switches returns to a main page reading '考试结束' next to a progress bar showing however few questions they had answered - never the 100% the code intends. Either the intent is wrong or the line is, and as written the reader cannot tell which.
- Fix: Move the recomputation into the two live branches and leave the finished branch alone: `} else { this.examButtonText = '考试结束'; this.progressNum = CommonConstant.NUM_100; this.progressText = ...; this.currentQuestionSn = this.examParam.currentQuestionSn; return; }`, or delete `this.progressNum = CommonConstant.NUM_100;` if showing the real answered share is intended.

### `HW-05-0167` - The @StorageLink default for windowStatusType is AppStorage.get of the same key cast to a non-nullable type, which is undefined until the ability's asynchronous getMainWindow resolves

- Category B, severity low, confidence confirmed
- Features: OFFICE-30
- Document: `huawei_industry_tree/05_office/docs/30_exam_cut_backstage_tips.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/exam_cut_backstage_tips-0000002431623674
- Why: The cast asserts a value the call may not have. When the key is absent the decorated variable is initialised to undefined while its declared type says it is a WindowStatusType, and isWindowStatusTypeChanged - which aboutToAppear invokes directly at ExamPage.ets:176 - compares that undefined against four enum members, matches none, and takes the else branch that declares the window normal and forces isOnForeground back to true. On a cold start into the exam page in split-screen or floating mode the blur overlay is therefore not applied.
- Fix: Give the link a concrete default and let AppStorage overwrite it when the ability publishes the real value: `@StorageLink('windowStatusType') @Watch('isWindowStatusTypeChanged') windowStatusType: window.WindowStatusType = window.WindowStatusType.FULL_SCREEN;`, and have EntryAbility seed the key synchronously with getMainWindowSync().getWindowStatus() before loadContent.

### `HW-05-0168` - 工程目录 spells the backup ability directory 'entrybackupablility'; the directory in the sample is 'entrybackupability'

- Category E, severity low, confidence confirmed
- Features: OFFICE-30
- Document: `huawei_industry_tree/05_office/docs/30_exam_cut_backstage_tips.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/exam_cut_backstage_tips-0000002431623674
- Why: The project tree is what a reader recreates by hand when building the sample from the document rather than from the ZIP. A directory named entrybackupablility does not match the srcEntry path in module.json5, and the backup extension fails to resolve.
- Fix: Correct line 99 of 工程目录 to `│  ├──entrybackupability`.

### `HW-05-0169` - 实现思路 step 2 names the AppStorage state variable 'isOnForegroundChanged', which is the @Watch callback, not the key - the key is 'isOnForeground'

- Category E, severity low, confidence confirmed
- Features: OFFICE-30
- Document: `huawei_industry_tree/05_office/docs/30_exam_cut_backstage_tips.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/exam_cut_backstage_tips-0000002431623674
- Why: The two names differ by a suffix and do completely different jobs: 'isOnForeground' is the AppStorage key that @StorageLink binds to, while isOnForegroundChanged is the method @Watch invokes when it changes. A reader following the prose rather than the snippet will write @StorageLink('isOnForegroundChanged'), which binds to a key nothing ever writes, and the page will never learn that the app went to the background.
- Fix: Correct step 2 to name the key 'isOnForeground' and, if the callback is worth mentioning, say that the change is handled in the @Watch callback isOnForegroundChanged.

### `HW-05-0178` - The URL parser is imported from the legacy @ohos.url path as a default export instead of the documented @kit.ArkTS named import

- Category A, severity low, confidence confirmed
- Features: OFFICE-31
- Document: `huawei_industry_tree/05_office/docs/31_schedule_share.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule_share-0000002573150445
- Why: This is the only module in the project reached through the legacy @ohos.* path, and it is imported as a default binding rather than the named `url` export the reference documents. The project builds with `"useNormalizedOHMUrl": true` (ScheduleShare.zip#build-profile.json5), so mixing the two module-resolution forms in one file is exactly the kind of inconsistency that form is meant to eliminate, and a reader copying this line into a new project gets an import shape the reference does not describe.
- Fix: Replace EntryAbility.ets:24 with `import { url } from '@kit.ArkTS';`. The call site needs no change.

### `HW-05-0181` - The import confirmation tells the user the schedule was added to team schedules while the code adds it to colleague schedules, which are two separate groups in the same UI

- Category B, severity low, confidence confirmed
- Features: OFFICE-31
- Document: `huawei_industry_tree/05_office/docs/31_schedule_share.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/schedule_share-0000002573150445
- Why: A received schedule lands in the colleague group, under the sharer's name, and can be shown or hidden from the 同事日程 section of the settings dialog. The user is told it went to 团队日程 instead, so they look for it under the wrong heading, and when they toggle the team group off the schedule they just imported does not disappear. The comment and the log line in the same method already say colleague; only the user-facing string is wrong.
- Fix: Change the toast at ReminderDialog.ets:423 to name the colleague group, and move it into the string resource file with the rest of the user-facing text.

### `HW-05-0183` - The office architecture series is numbered v1 / v1-5_1 / v1_2 while eleven other industries use v1 / v1_1 / v1_2, so in 05_office the slug that reads as part two of the series is the empty FAQ stub

- Category F, severity low, confidence confirmed
- Features: OFFICE-32
- Document: `huawei_industry_tree/05_office/docs/32_practice-office-app-architecture-v1_2.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-office-app-architecture-v1_2-0000002263504048
- Why: The slug is what a reader sees in the URL and what a tool sorts on. In every other industry the sequence v1 -> v1_1 -> v1_2 walks from the architecture case study through the scenario index to the FAQ. In 05_office the same walk lands on v1_2 immediately after v1, because the scenario index - the single most useful page in the industry, listing all 29 scenario documents - sits behind the non-standard v1-5_1 slug and sorts outside the series. Anyone navigating the office industry by URL pattern reaches the empty FAQ stub and concludes the series has two parts.
- Fix: Renumber the office scenario index to the v1_1 slug used by every other industry, or document why the office series skips it.

