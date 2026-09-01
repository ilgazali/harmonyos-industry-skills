# Pitfalls

> Generated from `features/*.md`. Source industry: `04_education`, 21 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (1)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-04-0155` - Systematic: 18 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: EDU-03, EDU-04, EDU-05, EDU-06, EDU-07, EDU-08, EDU-09, EDU-10, EDU-11, EDU-12, EDU-13, EDU-14, EDU-15, EDU-16, EDU-17, EDU-18, EDU-19, EDU-20
- Document: `huawei_industry_tree/04_education/docs/15_cascading_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cascading_selection-0000002364082736
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (154)

### `HW-04-0019` - Tabs.onContentWillChange returns false unconditionally, so the two other tabs can never be opened

- Category C, severity blocker, confidence confirmed
- Features: EDU-04
- Document: `huawei_industry_tree/04_education/docs/04_horizental_and_vertical_scrolling_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/horizental_and_vertical_scrolling_list-0000002296361950
- Why: onContentWillChange is the interception hook for a tab switch; returning false vetoes it. The page ships three tabs (主页, 消息, 课表) and starts on index 2, so tapping 主页 or 消息 changes only the highlight - the content never switches. onChange then writes currentIndex anyway, leaving the bar highlighting a tab whose content is not shown.
- Fix: Delete the onContentWillChange call. Use it only when you genuinely need to block a switch, and return true for the switches you want to allow.

### `HW-04-0048` - The rawfile path is prefixed with 'rawfile/', which resolves outside the rawfile root and throws before the PDF is ever copied to the sandbox

- Category B, severity blocker, confidence confirmed
- Features: EDU-08
- Document: `huawei_industry_tree/04_education/docs/08_pdf_to_long_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pdf_to_long_image-0000002349147473
- Why: The path argument is already relative to resources/rawfile, so prefixing it asks for resources/rawfile/rawfile/试卷1.pdf, which does not exist and raises 9001005. The call is not inside a try, and it runs in aboutToAppear on the first visit to every document - before the sandbox copy exists. The PDF is therefore never written, loadDocument fails, and 转换长图 does nothing on a clean install. Two samples in the same industry call this API with different path conventions and only one of them can be right.
- Fix: Drop the 'rawfile/' prefix, and wrap the copy in a try/catch that surfaces the failure instead of leaving the page blank.

### `HW-04-0057` - The preview page closes an undefined file descriptor in its finally block, so any open failure throws out of the error handler

- Category B, severity blocker, confidence confirmed
- Features: EDU-09
- Document: `huawei_industry_tree/04_education/docs/09_import_pdf.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/import_pdf-0000002378274393
- Why: When openSync fails, file stays null, file?.fd evaluates to undefined and fs.closeSync(undefined) raises a parameter error from inside the finally - which discards the original failure and propagates out of onReady, where nothing catches it. The empty catch above it means the first failure was already silent, so the user sees a crash rather than a message. The two files in this same sample disagree on the guard, and only ImportPdf.ets has it right.
- Fix: Null-check before closing, and replace the empty catch with a log plus a visible error state.

### `HW-04-0059` - Tabs.onContentWillChange returns false unconditionally, so three of the four bottom tabs can never be opened

- Category C, severity blocker, confidence confirmed
- Features: EDU-09
- Document: `huawei_industry_tree/04_education/docs/09_import_pdf.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/import_pdf-0000002378274393
- Why: Returning false vetoes every switch, so the three tabs beyond the home page are unreachable. They are empty TabContent blocks in this sample, so the veto hides that they were never implemented. The identical defect appears in 04_horizental_and_vertical_scrolling_list.md, so it is spreading between this industry's samples rather than being a one-off.
- Fix: Delete the onContentWillChange call.

### `HW-04-0114` - The recording file descriptor is closed in a finally block immediately after start, three seconds before the recording ends

- Category B, severity blocker, confidence confirmed
- Features: EDU-16
- Document: `huawei_industry_tree/04_education/docs/16_equipment_detection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/equipment_detection-0000002398440861
- Why: AVRecorder writes through the descriptor named by url for the whole duration of the recording. The finally runs the moment start() resolves, so the descriptor is closed while three seconds of audio are still being written to it. Wrapping start() in a try/finally is the mistake: the recording is asynchronous and outlives the function, so there is no scope whose exit corresponds to the end of the write. The playback step then opens a file that was closed under the recorder.
- Fix: Keep this.file open, and close it in stopRecordingProcess after the release; guard against double close by clearing the field.

### `HW-04-0122` - Playback closes the audio file descriptor in a finally block before the player has started reading it

- Category B, severity blocker, confidence confirmed
- Features: EDU-16
- Document: `huawei_industry_tree/04_education/docs/16_equipment_detection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/equipment_detection-0000002398440861
- Why: AVPlayer reads through the descriptor named by fdSrc for the whole of playback. prepare() and play() are asynchronous and are not awaited, so the finally runs before the player has read anything, and the descriptor is closed underneath it. This is the same defect as the recorder's, in the opposite direction, and the class already contains the correct place to do it - endAudio - which is dead code.
- Fix: Remove the finally, and close the descriptor from the completed state or from endAudio, which the completed handler should call.

### `HW-04-0139` - The word ForEach is keyed on JSON.stringify of an item whose coordinates change every seventeen milliseconds

- Category C, severity blocker, confidence confirmed
- Features: EDU-19
- Document: `huawei_industry_tree/04_education/docs/19_globe_label_animation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/globe_label_animation-0000002529698917
- Why: Every key contains the item's current 3D coordinates and opacity, all of which are rewritten sixty times a second, so every key differs on every frame and ArkUI destroys and recreates every Text node in the sphere on every frame. This is the whole animation loop running against the framework rather than with it: the @ObservedV2/@Trace decorations on x, y and z exist precisely so the position can update without rebuilding the node, and the key generator defeats them. It also serialises the word, its meaning, its part of speech, four example sentences and four translations into a string sixty times a second, per word.
- Fix: Key on item.word, which is unique and never changes.

### `HW-04-0147` - The scan result is copied only when the scanned file does not exist, because the accessSync guard is inverted

- Category B, severity blocker, confidence confirmed
- Features: EDU-20
- Document: `huawei_industry_tree/04_education/docs/20_scan_submit_homework.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/scan_submit_homework-0000002337721418
- Why: accessSync returns true when the file exists, so negating it means the whole submission branch runs only when the scanner produced nothing, and never when a scan succeeded. On a successful scan the PDF is not copied to the homework path, pdfFileExist is not set, and the page does not pop with its result - the feature the document is about does not happen. The comment on the line above states the intended condition correctly, which makes the negation a transcription slip rather than a misunderstanding. accessSync is also documented to take an "application sandbox path" while uris[0] is a URI from the scanner, so the call is questionable independently of its sense.
- Fix: Drop the negation, and gate on the result code as the document's snippet does rather than probing the file at all.

### `HW-04-0001` - The video-playback code sample calls setXComponentSurfaceSize, deprecated since API version 12

- Category A, severity high, confidence confirmed
- Features: EDU-01
- Document: `huawei_industry_tree/04_education/docs/01_practice-educate-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture-v1-0000001904563108
- Why: The document declares HarmonyOS 6.0.0 Release SDK (API 20) as the minimum, and the sample project sets compatibleSdkVersion 6.0.0(20). Publishing a deprecated API as the reference implementation for an API 20 project contradicts the deprecation notice, and setXComponentSurfaceRect is also the only one of the two that supports the offsetX/offsetY immersive layout the reference recommends.
- Fix: Replace setXComponentSurfaceSize with setXComponentSurfaceRect in the document snippet. The shipped sample already uses setXComponentSurfaceRect, so the document is the only place that is wrong.

### `HW-04-0006` - The sample declares four user_grant permissions that no code requests and no feature uses

- Category D, severity high, confidence confirmed
- Features: EDU-01
- Document: `huawei_industry_tree/04_education/docs/01_practice-educate-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture-v1-0000001904563108
- Why: All four are user_grant permissions, so they must be requested at runtime with requestPermissionsFromUser before use; declaring them without a request flow gives no capability and only enlarges the permission surface shown at release review. READ_MEDIA and WRITE_MEDIA are additionally marked "Deprecated from: 22" with the Files permission group as substitute, so a framework meant to be copied is seeding a permission that is already on its way out.
- Fix: Delete the READ_MEDIA, WRITE_MEDIA, CAMERA and MEDIA_LOCATION entries from entry/src/main/module.json5. If a later feature needs media files, use the Files permission group substitute or the picker, which needs no permission at all.

### `HW-04-0008` - EntryAbility subscribes to avoidAreaChange and never unregisters it

- Category B, severity high, confidence confirmed
- Features: EDU-01
- Document: `huawei_industry_tree/04_education/docs/01_practice-educate-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture-v1-0000001904563108
- Why: on/off must be paired. The callback closes over the window object and keeps writing to AppStorage after the window stage is gone; when the ability is recreated a second listener is added on top of the first, so the handler count grows with every relaunch.
- Fix: Store the window in a private field in onWindowStageCreate and release the subscription in onWindowStageDestroy: this.windowClass?.off('avoidAreaChange').

### `HW-04-0011` - Exam scoring reads the first answer ever given for a question, so a corrected answer is ignored

- Category B, severity high, confidence confirmed
- Features: EDU-01
- Document: `huawei_industry_tree/04_education/docs/01_practice-educate-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture-v1-0000001904563108
- Why: find returns the first match. The user can move back with 上一题 (previous question) and change a radio selection; each change pushes a new entry but scoring still reads the original one. A student who corrects a wrong answer is still marked wrong, and one who spoils a right answer is still marked right.
- Fix: Replace the array with a Map<string, number>, or in onChange remove any existing entry for the key before pushing.

### `HW-04-0012` - The TextTimer onDisAppear handler overwrites the score with 0 on the submit path the document publishes

- Category B, severity high, confidence probable
- Features: EDU-01
- Document: `huawei_industry_tree/04_education/docs/01_practice-educate-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture-v1-0000001904563108
- Why: submit(true) sets isSubmitted before it writes Score, then pushes MineTestResPage; when the timer leaves the tree the handler takes the else branch and writes 0 over the score that was just computed. The result page reads the same AppStorage key, so a timed-out exam scores 0 regardless of the answers.
- Fix: Delete the else branch. Clearing the score belongs in aboutToAppear of the next exam attempt, not in a teardown callback that races the submit path.

### `HW-04-0015` - The drag handler converts vp to score with the ratio inverted, so both handles run ahead of the finger by about 35 percent

- Category B, severity high, confidence confirmed
- Features: EDU-03
- Document: `huawei_industry_tree/04_education/docs/03_adjusting_score_interval_screening_schools.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/adjusting_score_interval_screening_schools-0000002294530662
- Why: sliderLength holds score units on a 258-wide scale (DataPanel max is 258 and the initial [33, 225, 0] sums to 258), while the track is 300 vp wide and even.offsetX is in vp. Score-to-vp is value/258*300; vp-to-score is therefore offset/300*258, the reciprocal. Using the same factor in both directions scales the drag by (300/258)^2 = 1.35: a 100 vp drag moves the handle 135 vp and changes the displayed score by 116 instead of 86, so the handle detaches from the finger and the number under it is wrong.
- Fix: Divide by the track width and multiply by the score range in both onActionUpdate handlers: (even.offsetX - this.lastOffsetX) / 300 * 258. Better, hoist TRACK_WIDTH = 300 and SCORE_RANGE = 258 into named constants so the two directions cannot drift apart again.

### `HW-04-0020` - Three ForEach key generators build the key from new Date() or a global counter, so every key changes on every rebuild

- Category C, severity high, confidence confirmed
- Features: EDU-04
- Document: `huawei_industry_tree/04_education/docs/04_horizental_and_vertical_scrolling_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/horizental_and_vertical_scrolling_list-0000002296361950
- Why: A key containing the current timestamp, or a counter that advances on every call, differs on every render pass. The framework therefore treats every one of the 56 grid cells and every weekday header as a brand-new element and destroys and recreates the whole subtree on any state change - the exact opposite of what the key is for. In a table that is scrolled continuously this is the dominant cost on the page.
- Fix: Give each ForEach a data-derived key: the weekday index for classificationNames, `${row}_${col}` for the grid cells, and the item value for classIndex/classTime.

### `HW-04-0023` - The implementation write-up omits the EdgeEffect.None step the Scroll reference makes mandatory for this pattern

- Category E, severity high, confidence confirmed
- Features: EDU-04
- Document: `huawei_industry_tree/04_education/docs/04_horizental_and_vertical_scrolling_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/horizental_and_vertical_scrolling_list-0000002296361950
- Why: The reference names this as a precondition for the technique, not a refinement: without it the bounce animation breaks the synchronisation and the header drifts out of alignment with the grid. A reader who follows the document's three steps and not the zip gets a table that desynchronises at every edge.
- Fix: Add the EdgeEffect.None requirement to 实现思路 step 2 and show it in the snippet, as the shipped code already does.

### `HW-04-0027` - The document teaches maxLines(-1), a value outside the documented range [0, INT32_MAX]

- Category A, severity high, confidence confirmed
- Features: EDU-05
- Document: `huawei_industry_tree/04_education/docs/05_spread_all_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/spread_all_text-0000002297066498
- Why: -1 is outside the documented range, so 'disabling' the attribute this way relies on unspecified behaviour rather than on the API contract. The reference documents no sentinel value for 'no limit' - the way to have no limit is not to set the attribute. A published practice document teaching an out-of-range argument propagates it into every application that copies it.
- Fix: Drive the attribute conditionally, for example .maxLines(this.isSpread ? Number.MAX_SAFE_INTEGER : MAX_LINE), or split the two Text branches so the collapsed one sets maxLines and the expanded one does not.

### `HW-04-0028` - The expand-label width is measured with fontSize '10p', which is not a valid length unit

- Category B, severity high, confidence confirmed
- Features: EDU-05
- Document: `huawei_industry_tree/04_education/docs/05_spread_all_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/spread_all_text-0000002297066498
- Why: '10p' is neither a recognised unit nor a bare number, so the value is invalid and measureText falls back to its default of 16 rather than measuring at the label's real size. labelWidth is the reserved space for the ...展开 affordance, and it is subtracted from the available width when computing secondLineEndIndex - so the truncation point is computed against the wrong reserve and the label overlaps or leaves a gap.
- Fix: Pass the same font size as the rendered Text, as a number: fontSize: 10.

### `HW-04-0029` - Text is measured in vp but rendered in fp, so the truncation index is wrong at any non-default system font scale

- Category B, severity high, confidence confirmed
- Features: EDU-05
- Document: `huawei_industry_tree/04_education/docs/05_spread_all_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/spread_all_text-0000002297066498
- Why: fp scales with the user's system font-size setting and vp does not. The whole page computes where to cut the biography by comparing measured widths against the available width; measuring at a fixed 10 vp while the component draws at 10 fp means that as soon as the user enlarges the system font, the computed secondLineEndIndex is short and the visible text is cut in the wrong place - exactly the accessibility setting that makes a collapsed-text feature matter.
- Fix: Pass fontSize as a number in all four measureText calls so measurement and rendering use the same unit.

### `HW-04-0030` - The implementation write-up describes only the maxLines toggle and omits the entire measurement and truncation mechanism the page is built on

- Category E, severity high, confidence confirmed
- Features: EDU-05
- Document: `huawei_industry_tree/04_education/docs/05_spread_all_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/spread_all_text-0000002297066498
- Why: The documented technique - toggle maxLines - produces a different UI from the sample. maxLines alone cannot place an inline expand label at the end of the last visible line; that is the entire reason for the measurement code, and it is the only non-obvious thing in the sample. A reader following the document builds the trivial version and then cannot explain why the zip has 50 lines they were never told about.
- Fix: Document the measurement pass as step 2 and the maxLines toggle as step 3, and show the Stack that overlays the label on the truncated Text.

### `HW-04-0035` - The card array is an undecorated member of a @ComponentV2 struct, so the splice that deletes a card cannot re-render the ForEach

- Category C, severity high, confidence confirmed
- Features: EDU-06
- Document: `huawei_industry_tree/04_education/docs/06_stackable_word_cards.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/stackable_word_cards-0000002349040357
- Why: In state management V2 an undecorated member of a @ComponentV2 struct is not observed. The per-card animation still works, because WordCard is @ObservedV2 with @Trace on offsetX/opacityX/positionX/positionY/sizeRatio - those are property changes on an observed object. The array itself is not observed, so splice removes the element from the model while the ForEach keeps rendering the node it built for it. The removed card's Column stays in the tree with its gestures attached, and every subsequent index shifts under a stale render.
- Fix: Decorate the array: @Local arr: WordCard[]. In a @ComponentV2 component, prefer Repeat over ForEach for a mutable list - it is the V2 rendering-control API and diffs by key.

### `HW-04-0043` - The submitted flag is passed into a LazyForEach child whose key never changes, so the child is never refreshed and the right/wrong feedback never appears

- Category C, severity high, confidence confirmed
- Features: EDU-07
- Document: `huawei_industry_tree/04_education/docs/07_english_practice.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/english_practice-0000002315012362
- Why: isSubmitted lives in an @Provide array outside the data source, so writing to it re-runs ExamPage's build but leaves every LazyForEach key identical. The child is therefore not refreshed, its @Prop keeps the value captured at creation, @Watch('onSubmit') never fires and the toast that tells the student whether the answer was right never appears. The Radio's ".enabled(this.isSubmitted !== SUBMITTED)" is stale for the same reason, so a submitted question stays editable. The submit button itself does grey out, because .enabled() is evaluated in ExamPage's own build - which is what makes the defect look like it works.
- Fix: Keep the per-question state inside the Question objects, update through CommonDataSource.changeData (which already calls notifyDataChange), and make the key generator depend on the mutable field. Note the guide's companion rule: "ensure the onDataChange API of DataChangeListener generates new key values different from previous ones to trigger component re-rendering" - notifying alone is not enough while the key is constant.

### `HW-04-0049` - The sample declares the restricted system_basic permission WRITE_IMAGEVIDEO although it saves through a SaveButton, which exists to avoid it

- Category D, severity high, confidence confirmed
- Features: EDU-08
- Document: `huawei_industry_tree/04_education/docs/08_pdf_to_long_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pdf_to_long_image-0000002349147473
- Why: WRITE_IMAGEVIDEO is a restricted permission at system_basic level; a normal-APL application cannot obtain it without declaring a higher APL and passing ACL review, so the declaration buys nothing at runtime and costs a review finding at release. The SaveButton path the sample actually uses is the documented alternative to that permission, so the two are mutually exclusive by design - shipping both signals that the security component was not understood.
- Fix: Remove the WRITE_IMAGEVIDEO entry. The SaveButton plus createAsset within ten seconds of the tap is the complete flow.

### `HW-04-0050` - readPixelsToBuffer is not awaited, so each page is written into the long image from a buffer that may still be empty

- Category B, severity high, confidence confirmed
- Features: EDU-08
- Document: `huawei_industry_tree/04_education/docs/08_pdf_to_long_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pdf_to_long_image-0000002349147473
- Why: readPixelsToBuffer returns a promise. Dropping it means the copy into pagePixelBuffer is still in flight when writePixels reads that same buffer, so the page written into the long image is whatever the buffer held at that instant - typically zeros, giving a black band. Nothing catches the dropped promise either, so a read failure is silent. The surrounding try/catch cannot help: the rejection happens outside the synchronous block it guards.
- Fix: Await it, or use readPixelsToBufferSync since the loop is already sequential.

### `HW-04-0051` - No PixelMap is ever released: one per page plus the full-length composite are held for the life of the process

- Category B, severity high, confidence confirmed
- Features: EDU-08
- Document: `huawei_industry_tree/04_education/docs/08_pdf_to_long_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pdf_to_long_image-0000002349147473
- Why: These are native allocations, not JS objects. The composite alone is maxPageWidth x totalHeight x 4 bytes - for a 20-page document at page resolution that is tens of megabytes in one buffer - and each per-page PixelMap adds a page's worth on top, all retained until the process exits. Converting several documents in one session accumulates every composite. PixelMap exposes release() precisely because the garbage collector does not see this memory.
- Fix: Release each page PixelMap inside the loop, and release the previous AppStorage entry before overwriting it.

### `HW-04-0058` - The preview page reads its parameter with getParamByName, which returns an array of every matching page's parameter

- Category B, severity high, confidence confirmed
- Features: EDU-09
- Document: `huawei_industry_tree/04_education/docs/09_import_pdf.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/import_pdf-0000002378274393
- Why: The API returns every PdfPreview entry's parameter, not this page's. With one preview on the stack String(['a.pdf']) happens to yield 'a.pdf', which is why it appears to work. Open a second document without popping the first and the value becomes 'a.pdf,b.pdf', uriMap.get returns undefined, openSync(undefined) throws into the empty catch, and the finally then throws again on closeSync(undefined). NavDestinationContext.pathInfo is the accessor scoped to this page.
- Fix: Read the parameter from context.pathInfo.param, which belongs to the destination being built.

### `HW-04-0060` - The file descriptor is closed before the document is loaded, and loadDocument's result is discarded

- Category B, severity high, confidence confirmed
- Features: EDU-09
- Document: `huawei_industry_tree/04_education/docs/09_import_pdf.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/import_pdf-0000002378274393
- Why: The URI the picker returns carries a temporary read authorisation that the open descriptor represents; closing it first and then re-opening by raw path relies on ambient filesystem access to a location outside the app sandbox. Discarding the promise means a failed load leaves a blank PdfView with no message and no log - and this page has no other feedback path, because the catch above it is empty.
- Fix: await this.controller.loadDocument(...) inside the try, close the descriptor after it resolves, and render an error state in the catch.

### `HW-04-0068` - The vertical slot arithmetic places nine of twelve positions outside the comment layer, and the bottom-only mode entirely outside it

- Category B, severity high, confidence confirmed
- Features: EDU-10
- Document: `huawei_industry_tree/04_education/docs/10_video_course_bullet_comments.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_course_bullet_comments-0000002378246529
- Why: Twelve equal parts of 100% is 8.33% per part, not 33%. With 33 the slots run 0, 33, 66, 99 ... 363 percent of the layer height, so only slots 0, 1 and 2 land inside it and the remaining nine draw below the bottom edge. 底部弹幕 (bottom position mode) selects slots 8 to 11, which are 264% to 363% - that mode displays nothing at all. Random mode silently discards three quarters of the comments it generates.
- Fix: Set COMMENT_POSITION_PERCENT to 100 / POSITION_DIVISION, or reduce POSITION_DIVISION to 3 and adjust getTop/getBottom to match.

### `HW-04-0069` - The comment ForEach has no key generator, so the default key changes every frame and every node is destroyed and rebuilt sixty times a second

- Category C, severity high, confidence confirmed
- Features: EDU-10
- Document: `huawei_industry_tree/04_education/docs/10_video_course_bullet_comments.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_course_bullet_comments-0000002378246529
- Why: JSON.stringify of a BulletCommentItem includes positionX, which changes on every one of the 60 ticks per second. The default key is therefore different on every frame for every comment on screen, so ArkUI treats all of them as replaced and tears down and recreates the whole subtree each frame - the worst case the key mechanism exists to avoid, in the most frame-sensitive component on the page. The .key(item.id.toString()) at line 97 does not help: that attribute is a test identifier, not the ForEach diff key.
- Fix: Pass an explicit key generator based on a stable id, and fix the id collision noted separately so the key is also unique.

### `HW-04-0070` - Horizontal movement is driven by a 16 ms setInterval that reassigns the whole array each tick instead of by an ArkUI animation

- Category C, severity high, confidence confirmed
- Features: EDU-10
- Document: `huawei_industry_tree/04_education/docs/10_video_course_bullet_comments.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_course_bullet_comments-0000002378246529
- Why: The per-item mutation is invisible to state management because BulletCommentItem is undecorated; the only reason the screen updates is the whole-array reassignment on the last line, which is why it is unconditional. That makes the design accidental rather than intended, and it puts a forEach plus a filter plus a full re-render on the UI thread every 16 ms for the lifetime of playback. The guard "if (positionX !== item.positionX)" is also dead - speed is a non-zero constant, so the values always differ.
- Fix: Animate translate declaratively per comment, or at minimum decorate BulletCommentItem with @ObservedV2/@Trace so the position writes are observed and the array reassignment can be dropped.

### `HW-04-0076` - The microphone permission's usedScene names a FormAbility that does not exist in the project

- Category D, severity high, confidence confirmed
- Features: EDU-11
- Document: `huawei_industry_tree/04_education/docs/11_oral_english.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/oral_english-0000002349625326
- Why: usedScene is checked at application release review, and it names the ability that actually uses the permission. FormAbility appears nowhere in this project - the value is copied verbatim from the guide's example - so the declaration describes a component that does not exist while the one that records audio, EntryAbility, is not listed. The document's 权限说明 section states only the permission name and does not mention usedScene at all, so a reader has no way to catch this from the prose.
- Fix: Change the abilities array to EntryAbility, and add the usedScene requirement to the document's 权限说明 section.

### `HW-04-0077` - The recorder is released on two paths at once, and the stateChange path leaves the amplitude interval and the file descriptor behind

- Category B, severity high, confidence confirmed
- Features: EDU-11
- Document: `huawei_industry_tree/04_education/docs/11_oral_english.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/oral_english-0000002349625326
- Why: stop() drives the recorder to 'stopped', so the callback runs during the await on line 147 and releases the recorder and nulls the field. Execution then returns into stopRecordingProcess, whose narrowing from line 145 no longer holds: line 155 dereferences this.avRecorder, which is now undefined. If instead the recording ends by the TICK_COUNT_MAX timeout or the error handler, only the stateChange path runs - so the 100 ms amplitude interval is never cleared and the recording file descriptor is never closed. The interval keeps ticking, keeps incrementing tickCount and keeps re-raising the 'recording too long' toast for the rest of the session.
- Fix: Clear the interval and close the descriptor in one place, guard the release with a local copy of this.avRecorder taken before the first await, and have the stateChange handler only publish the state.

### `HW-04-0078` - AVPlayerManager.stopAudio releases the player without clearing the field, so the next release runs on an already-released instance

- Category B, severity high, confidence confirmed
- Features: EDU-11
- Document: `huawei_industry_tree/04_education/docs/11_oral_english.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/oral_english-0000002349625326
- Why: stop() drives the player to 'stopped', so the stateChange handler releases it and nulls the field; stopAudio then releases the same instance a second time. On the other path - OralTrainingPage.onPageHide calling stopAudio directly on a player that is not in 'playing' or 'paused' - stop() never runs, the handler never fires, and the field is left holding a released player, so aboutToDisappear's releasePlayer releases it again. Either way release() is called twice on one AVPlayer.
- Fix: Set this.avPlayer = undefined after releasing in stopAudio, and keep release in exactly one of the two paths.

### `HW-04-0085` - register is called before the on subscriptions, reversing the order the reference requires

- Category A, severity high, confidence confirmed
- Features: EDU-12
- Document: `huawei_industry_tree/04_education/docs/12_network_monitor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/network_monitor-0000002355490078
- Why: register is what activates the subscription with the network manager; handlers attached afterwards are not part of the registration, so events arriving between register and the last on are missed, and depending on the implementation the later handlers may never fire at all. The document does not merely reproduce the sample's ordering - it states it as the procedure, so the error propagates to every reader.
- Fix: Move the four on() calls above register(), in both the document and NetMonitor.ets.

### `HW-04-0086` - The network connection is registered on every page show and never unregistered, and none of the four event handlers is ever released

- Category B, severity high, confidence confirmed
- Features: EDU-12
- Document: `huawei_industry_tree/04_education/docs/12_network_monitor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/network_monitor-0000002355490078
- Why: onPageShow fires on every return to the page, so each visit adds a fresh registration and four more handlers to the same NetConnection object; the reference explicitly requires a matching unregister, and none exists. Every subsequent network event is then delivered once per accumulated registration, and the callbacks keep running after the page is destroyed because nothing releases them.
- Fix: Move networkMonitor to aboutToAppear, keep a flag so it runs once, and add an off/unregister teardown alongside networkSpeedMonitorOff.

### `HW-04-0087` - The clock interval id is discarded, so a new one-second timer is started on every page show and none is ever cleared

- Category B, severity high, confidence confirmed
- Features: EDU-12
- Document: `huawei_industry_tree/04_education/docs/12_network_monitor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/network_monitor-0000002355490078
- Why: setInterval returns a handle that is the only way to stop it. Discarding it means the timer runs for the life of the process, and because onPageShow fires on every return to the page, the timers accumulate - each one calling updateTime and writing two @State fields once a second, so the re-render rate grows with the number of visits.
- Fix: let this.clockId = setInterval(...) in aboutToAppear and clearInterval(this.clockId) in aboutToDisappear.

### `HW-04-0088` - The speed unit threshold compares a MB/s value against 1024, so the MB/s branch is unreachable and every speed is shown in KB/s

- Category B, severity high, confidence confirmed
- Features: EDU-12
- Document: `huawei_industry_tree/04_education/docs/12_network_monitor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/network_monitor-0000002355490078
- Why: linkDownRate is bit/s, so dividing by 1e6 and then by 8 yields megabytes per second. Comparing that against 1024 asks whether the link is slower than 1024 MB/s, i.e. about 8.6 Gbit/s - true for every network a phone will ever see. The MB/s branch is dead code and a 100 Mbit/s link is reported as '12800KB/s' instead of '12.5MB/s'. The conversion also mixes bases: the megabyte comes from dividing by 1e6 but the kilobyte from multiplying by 1024.
- Fix: Compare against 1, and use one consistent base for both conversions.

### `HW-04-0089` - The permission section names INTERNET while the sample declares GET_NETWORK_INFO, which is what every API used actually requires

- Category E, severity high, confidence confirmed
- Features: EDU-12
- Document: `huawei_industry_tree/04_education/docs/12_network_monitor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/network_monitor-0000002355490078
- Why: The document names the wrong permission. INTERNET governs actually using a socket, which this sample never does; GET_NETWORK_INFO governs reading network status and quality, which is all it does. A reader following the 权限说明 declares INTERNET, omits GET_NETWORK_INFO, and every call fails with error 201 Permission denied - while the shipped module.json5 they also have in front of them says the opposite.
- Fix: Correct the 权限说明 section to ohos.permission.GET_NETWORK_INFO, matching the shipped module.json5 and all four API references.

### `HW-04-0092` - The refresh path re-raises the calendar permission request on every page show, with none of the handling the same file implements a few lines below

- Category D, severity high, confidence confirmed
- Features: EDU-13
- Document: `huawei_industry_tree/04_education/docs/13_class_add_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/class_add_schedule-0000002358628536
- Why: requestPermissionsFromUser is raised every single time the user returns to the timetable. While the permission is still undecided that is a dialog on each visit; once it has been refused permanently the call resolves successfully and shows nothing, so refreshList silently takes the unpermitted branch forever with no route to the settings sheet - even though the correct recovery is implemented ten lines away. Two permission flows in one component, and the one wired to the automatic refresh is the incomplete one.
- Fix: Extract the complete flow from requestCalendarPermission into one method, add a checkAccessToken fast path, and call it from both places.

### `HW-04-0093` - Removing a reminder marks the course as removed and persists that before the calendar delete has run, and ignores its failure

- Category B, severity high, confidence confirmed
- Features: EDU-13
- Document: `huawei_industry_tree/04_education/docs/13_class_add_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/class_add_schedule-0000002358628536
- Why: The method returns before getCalendar has even called back, so the caller cannot await it and cannot know the outcome. The UI clears eventId and the added-to-calendar flag straight away, and the next refreshPreference writes -1 into preferences - so if deleteEvent fails, the event is still in the system calendar, the app has forgotten its id, and the reminder can never be removed from inside the app again. The document publishes the un-awaited call as the pattern.
- Fix: Return a promise from deleteFromCalendar, await it, and update course.eventId / isAddedToCalendar only when it resolves successfully.

### `HW-04-0099` - The chapter-complete toast fires on every arrival at the end, and a chapter shorter than the viewport is complete before it is read

- Category B, severity high, confidence confirmed
- Features: EDU-14
- Document: `huawei_industry_tree/04_education/docs/14_course_progress_display.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/course_progress_display-0000002358926170
- Why: The author guarded the array and not the toast, so scrolling up a line and back down re-announces 已完成 every time - the asymmetry shows the repeat was not intended. The same event is also the completion criterion, so any chapter whose rendered text is shorter than the 640 vp pane is at the end position as soon as it lays out: it is marked read, the toast appears, and the course progress advances without the student having read anything.
- Fix: Move the toast inside the duplicate guard, and require a minimum scroll (for example rollingProgress reaching textBuffer.length) before treating a chapter as complete.

### `HW-04-0107` - One ListScroller instance is bound to all three cascading lists

- Category C, severity high, confidence confirmed
- Features: EDU-15
- Document: `huawei_industry_tree/04_education/docs/15_cascading_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cascading_selection-0000002364082736
- Why: The reference forbids sharing a Scroller across containers. With three Lists bound to one instance the binding is undefined: at best two of the three are silently ignored, and any later call to scrollToIndex or currentOffset addresses whichever container the framework happened to associate with it. Nothing in this component ever calls a Scroller method, so all three bindings are cost without benefit - the sample teaches a construction the reference rules out, for no gain.
- Fix: Remove the scroller option from all three Lists; add one instance per List only if programmatic scrolling is needed.

### `HW-04-0108` - Schools and graduation years share one array separated by an id sentinel, so two columns iterate all rows and hide the wrong ones inside the ListItem

- Category C, severity high, confidence confirmed
- Features: EDU-15
- Document: `huawei_industry_tree/04_education/docs/15_cascading_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cascading_selection-0000002364082736
- Why: The if is inside the ListItem, so a ListItem is still created for every one of the 17 entries in both columns and the .onClick, .backgroundColor and .borderRadius stay attached to rows that render nothing. Each column therefore carries thirteen or four content-free but live rows, and the school column's handler would assign item.department - undefined for a year entry - into an Array<DepartmentItem> state field. Filtering the data instead removes the empty rows, the dead handlers and the sentinel comparison in one step.
- Fix: Split the JSON into schools and years, or derive this.schoolList and this.yearList once in getJson, and drop the id tests from the item builders.

### `HW-04-0115` - The preview hard-codes camera index 1 behind a guard that only rejects an empty camera list

- Category B, severity high, confidence confirmed
- Features: EDU-16
- Document: `huawei_industry_tree/04_education/docs/16_equipment_detection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/equipment_detection-0000002398440861
- Why: The guard proves the author knew the array can be short and then indexed past it anyway: a device with exactly one camera passes length !== 0 and getSupportedSceneModes(undefined) throws out of an async function with no try/catch. Single-camera devices are ordinary among the tablet and 2-in-1 form factors this module declares. Index 1 is also not a contract - the ordering of getSupportedCameras is not specified, so even on a multi-camera phone it is not necessarily the intended lens for a camera self-test.
- Fix: Find the device by cameraPosition rather than by index, and guard the lookup returning undefined.

### `HW-04-0116` - Support for NORMAL_PHOTO is checked and a NORMAL_VIDEO session is created

- Category B, severity high, confidence confirmed
- Features: EDU-16
- Document: `huawei_industry_tree/04_education/docs/16_equipment_detection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/equipment_detection-0000002398440861
- Why: The capability test is meaningless because it tests a different mode from the one used. On a device that supports NORMAL_PHOTO but not NORMAL_VIDEO the guard passes and createSession fails; on one that supports NORMAL_VIDEO but not NORMAL_PHOTO the function returns early and the preview never starts even though it would have worked. The as camera.VideoSession cast also hides a null return from createSession, so a failure surfaces later as a call on undefined in beginConfig.
- Fix: Test sceneModes for NORMAL_VIDEO, and check the createSession result before casting.

### `HW-04-0117` - The preview profile is hard-coded rather than chosen from the device's supported profiles, and a creation failure is cast away into addOutput(undefined)

- Category B, severity high, confidence confirmed
- Features: EDU-16
- Document: `huawei_industry_tree/04_education/docs/16_equipment_detection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/equipment_detection-0000002398440861
- Why: createPreviewOutput throws when the profile is not among the device's supported preview profiles, and 1920x1080 YUV_420_SP is not guaranteed on every device the module targets. The empty catch then swallows the error code, the function returns undefined, the as cast tells the compiler otherwise, and startPreviewOutput reaches session.addOutput(undefined) - so a device that cannot do exactly this profile fails with an unrelated error deep in the session configuration instead of a clear message.
- Fix: Select the profile from previewProfiles, log the BusinessError in the catch, and have the caller handle an undefined preview output.

### `HW-04-0118` - The camera is shut down only by the next button, and neither the preview output nor the session is ever released

- Category B, severity high, confidence confirmed
- Features: EDU-16
- Document: `huawei_industry_tree/04_education/docs/16_equipment_detection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/equipment_detection-0000002398440861
- Why: The sheet can be dismissed by swiping down or by pressing back, and the button itself does nothing when isCamera is false - on all of those paths the camera stays open and the session keeps running after the sheet is gone, holding the device's camera against every other application. Calling stopPreviewOutput before any preview started also dereferences an uninitialised module-level session and throws. Leaking the session and the preview output means the native resources are never returned even on the one path that does call stop.
- Fix: Move the shutdown into aboutToDisappear, make it idempotent, and release the preview output and the session as well as closing the input.

### `HW-04-0119` - Both permission declarations omit the mandatory when field, name unrelated placeholder reasons, and one points at an ability that does not exist

- Category D, severity high, confidence confirmed
- Features: EDU-16
- Document: `huawei_industry_tree/04_education/docs/16_equipment_detection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/equipment_detection-0000002398440861
- Why: Three defects in two declarations. FormAbility does not exist in this project - the same copy-paste from the guide's example as in the OralEnglish sample. Neither entry sets when, which the guide says cannot be empty for a user_grant permission. And the two reasons are unrelated placeholder resources - the application name for the microphone and the module description for the camera - neither of which states a scenario, which the same guide warns may cause a release application to be rejected.
- Fix: Correct the abilities array, add "when": "inuse" to both, and write a reason resource per permission.

### `HW-04-0123` - prepare is called both from the state-change handler and from the caller, and play is called while the player is still initialized

- Category B, severity high, confidence confirmed
- Features: EDU-16
- Document: `huawei_industry_tree/04_education/docs/16_equipment_detection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/equipment_detection-0000002398440861
- Why: Assigning fdSrc drives the player to initialized, so the handler's prepare() is already in flight when playAudio's own prepare() runs, and its play() runs while the player is still initialized rather than prepared - a call the AVPlayer state machine does not allow from that state. The two paths race: whichever prepare resolves second operates on a player the other has already moved on. Either the handler or the caller should own the transitions, not both.
- Fix: Delete the prepare and play calls from playAudio and keep the state-machine-driven versions in init.

### `HW-04-0125` - Advancing to the next word is gated on the pronunciation audio completing, and the audio error path never completes

- Category B, severity high, confidence confirmed
- Features: EDU-17
- Document: `huawei_industry_tree/04_education/docs/17_word_spelling.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/word_spelling-0000002368882740
- Why: A missing or corrupt audio rawfile, or any AVPlayer error, leaves showTips permanently true: the completion callback never fires, the automatic advance never runs, and the manual next button - the user's only remaining control - returns immediately because of its own guard. The learner is stuck on that word with a visibly present but inert next button and no way forward except leaving the page. Making the single failure path of an optional feature block the core interaction is the design problem; the guard on the manual button is what removes the escape hatch.
- Fix: Clear showTips on a timeout regardless of playback outcome, and remove the showTips guard from the manual next button so the user can always advance.

### `HW-04-0126` - The keyboard-height subscription is never released and the window lookup has no rejection handler

- Category B, severity high, confidence confirmed
- Features: EDU-17
- Document: `huawei_industry_tree/04_education/docs/17_word_spelling.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/word_spelling-0000002368882740
- Why: on/off must be paired. The handler closes over the component and writes this.keyboardHeight, so it keeps running - and keeps driving an animateTo - after the page has been destroyed, and a second visit to the page adds another subscription on the same window. The dropped promise also means a failure to obtain the window is silent, and the layout then never adapts to the keyboard with no indication why.
- Fix: Store the window handle in aboutToAppear, release the subscription in aboutToDisappear, and log a rejection from getLastWindow.

### `HW-04-0131` - The weekly x-axis labels subtract six from the day of the month with no borrow, producing zero and negative dates in the first six days of every month

- Category B, severity high, confidence confirmed
- Features: EDU-18
- Document: `huawei_industry_tree/04_education/docs/18_track_study_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/track_study_time-0000002391113388
- Why: getDate() returns the day within the current month, so subtracting six from it does not roll back into the previous month. On the 3rd of August the six labels read 8/-3, 8/-2, 8/-1, 8/0, 8/1, 8/2 - four of them impossible dates, and all of them stamped with the current month even though four belong to July. The month is taken from today unconditionally, so it is wrong for every label that crosses a boundary. Date arithmetic on the day component alone is the defect; Date.setDate borrows across months and years for free.
- Fix: Compute each label's date with a Date object and setDate, and read the month from that date rather than from today.

### `HW-04-0140` - Neither the rotation interval nor the inertia interval is released when the page goes away

- Category B, severity high, confidence confirmed
- Features: EDU-19
- Document: `huawei_industry_tree/04_education/docs/19_globe_label_animation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/globe_label_animation-0000002529698917
- Why: The 17 ms rotation interval starts in aboutToAppear and nothing stops it when the page is destroyed, so it keeps calling rotateX and rotateY - each a full pass over the word array writing three @Trace properties per item - sixty times a second for the rest of the process's life, on a component that is no longer rendered. The inertia interval leaks the same way when the user releases a drag and then leaves. Every setInterval in a component needs a matching clearInterval in its teardown.
- Fix: Add aboutToDisappear and clear both handles there.

### `HW-04-0141` - The rotation toggle clears the interval without clearing its handle, so auto-rotation can never be restarted

- Category B, severity high, confidence confirmed
- Features: EDU-19
- Document: `huawei_industry_tree/04_education/docs/19_globe_label_animation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/globe_label_animation-0000002529698917
- Why: clearInterval stops the timer but does not change the variable holding its id, so timerAn keeps a stale non-zero value. The next call therefore takes the same branch and clears an id that no longer refers to anything, and the else branch is unreachable for the rest of the component's life - the toggle stops the animation permanently on its first use. The function is published as the way to start and stop the animation and it can only stop it.
- Fix: Reset the handle to 0 after clearing, and do the same in the touch and gesture handlers that clear it.

### `HW-04-0142` - Lifting a finger after a drag starts two rotation intervals and leaks one of them

- Category B, severity high, confidence confirmed
- Features: EDU-19
- Document: `huawei_industry_tree/04_education/docs/19_globe_label_animation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/globe_label_animation-0000002529698917
- Why: A pan ends with both a TouchType.Up event and onActionEnd, so two intervals are created for one lift and only one id survives in this.timerAn; the other runs forever with no handle. The order of the two callbacks decides which one leaks, so the outcome is not even deterministic. Every subsequent drag adds another orphan, and each orphan is a full 60 Hz pass over the word array - so the sphere accelerates as the user interacts with it, which is exactly the symptom that makes this defect hard to attribute.
- Fix: Remove the interval creation from the onTouch handler and let onActionStart/onActionEnd own the timer, or vice versa.

### `HW-04-0148` - The file-copy helper closes two null handles in its finally block when the source cannot be opened

- Category B, severity high, confidence confirmed
- Features: EDU-20
- Document: `huawei_industry_tree/04_education/docs/20_scan_submit_homework.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/scan_submit_homework-0000002337721418
- Why: Both handles are initialised with a non-null assertion on null, so if the first openSync throws - a missing or unreadable scanner output, which is exactly the failure this function must survive - the catch swallows it and the finally then calls fs.close(null) twice. That raises a parameter error from inside the finally, discarding the original failure and propagating out of the scanner's onResult callback where nothing catches it. When only the source opens, destFile is still null and the second close throws the same way. The same defect appears in UploadPdf's PdfPreview (HW-04-0057), so it is recurring across this industry.
- Fix: Null-check both handles before closing, and log the caught error instead of discarding it.

### `HW-04-0149` - The destination file is deleted with the asynchronous unlink and immediately reopened, so the delete races the create

- Category B, severity high, confidence confirmed
- Features: EDU-20
- Document: `huawei_industry_tree/04_education/docs/20_scan_submit_homework.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/scan_submit_homework-0000002337721418
- Why: fs.unlink returns a promise; in a synchronous void function nothing awaits it, so the deletion is still pending when openSync recreates the path immediately afterwards. The two can complete in either order, and when the unlink lands second the freshly written homework PDF is deleted. The intent - clear a previous submission before writing the new one - is right; only the API variant is wrong, and the surrounding code shows the author knew which variants to use. The dropped promise also makes a failed delete silent.
- Fix: Use fs.unlinkSync, matching every other call in the function.

### `HW-04-0150` - The documented result handler bears no resemblance to the shipped one

- Category E, severity high, confidence confirmed
- Features: EDU-20
- Document: `huawei_industry_tree/04_education/docs/20_scan_submit_homework.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/scan_submit_homework-0000002337721418
- Why: The two versions differ in their success condition, in their handling of cancellation, and in what they do afterwards. The document omits the pop(this.pageParam) that is the only way the result reaches the detail page - so a reader who implements the documented handler gets a scan that copies the file and then leaves the user stranded on the scanner. The document also never mentions the -1 cancel code, which is the only code the shipped handler actually distinguishes.
- Fix: Reprint the handler from DocScan.ets and describe the result codes and the pop-with-result return path.

### `HW-04-0002` - Both the document and the sample build the XComponent with the deprecated libraryname constructor

- Category A, severity medium, confidence confirmed
- Features: EDU-01
- Document: `huawei_industry_tree/04_education/docs/01_practice-educate-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture-v1-0000001904563108
- Why: The XComponentOptions overload replaces this one from API 12, and the deprecated form additionally suppresses onLoad when libraryname is set: the reference says "The callback is triggered only when the libraryname parameter is not set for the XComponent." The sample depends on onLoad to obtain the surface id, so keeping libraryname in the parameter list is a hazard with no purpose - no native library exists in this project.
- Fix: Use the API 12 XComponentOptions form and drop the empty id and libraryname fields, which are meaningless for an ArkTS-side surface.

### `HW-04-0003` - Every source path in the code samples omits the src/main segment and points at directories that do not exist in the zip

- Category E, severity medium, confidence confirmed
- Features: EDU-01
- Document: `huawei_industry_tree/04_education/docs/01_practice-educate-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture-v1-0000001904563108
- Why: The project tree printed earlier in the same document is rooted at features/train/src/main/ets, so the document contradicts itself, and a reader following the comment cannot find the file. src/ets is not a valid HarmonyOS module layout - hvigor only compiles src/main/ets.
- Fix: Insert the missing main/ segment in all four path comments.

### `HW-04-0005` - The module description list omits the login HAR although the tree, build-profile and oh-package all contain it

- Category E, severity medium, confidence confirmed
- Features: EDU-01
- Document: `huawei_industry_tree/04_education/docs/01_practice-educate-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture-v1-0000001904563108
- Why: The document states the list is exhaustive and it is not. Login is the module the same document singles out as needing developer completion ("框架代码中登录验证模块，只是UI能力" - the login validation module is UI only), so leaving it out of the module map is the opposite of helpful.
- Fix: Add the login module to the bullet list, and to 表1 模块划分, where the 我的 row currently absorbs 账号注册登录.

### `HW-04-0007` - Every user_grant permission in the sample uses the app name as its reason string and declares when: always

- Category D, severity medium, confidence confirmed
- Features: EDU-01
- Document: `huawei_industry_tree/04_education/docs/01_practice-educate-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture-v1-0000001904563108
- Why: $string:app_name resolves to the application label, which names no scenario and, per the same guide, "may cause the application release application to be rejected". "when": "always" declares background use; this application has no background task or continuous task, so the wider claim is unjustified.
- Fix: Add per-permission reason strings to resources/base/element/string.json and set "when": "inuse". Better still, remove the permissions entirely - see the finding above.

### `HW-04-0009` - AVPlayerDemo registers five AVPlayer callbacks but unregisters only four

- Category B, severity medium, confidence confirmed
- Features: EDU-01
- Document: `huawei_industry_tree/04_education/docs/01_practice-educate-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture-v1-0000001904563108
- Why: Every on must have a matching off before release(). The speedDone handler survives until the AVPlayer object is collected and holds a reference to the disposed component.
- Fix: Add this.avPlayer.off('speedDone'); to aboutToDisappear, or drop the empty speedDone registration, which does nothing.

### `HW-04-0010` - The AVPlayer error handler calls reset, which re-sets the same URL, producing an unbounded retry loop

- Category B, severity medium, confidence confirmed
- Features: EDU-01
- Document: `huawei_industry_tree/04_education/docs/01_practice-educate-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture-v1-0000001904563108
- Why: reset() drives the player back to idle, the stateChange handler immediately re-assigns the same URL, and a permanently failing source (no network, 404, expired link) fails again. Nothing counts attempts and nothing tells the user, so the component spins on the media pipeline for as long as the page is open, and the error callback takes no argument so the cause is never logged.
- Fix: Accept the BusinessError argument, log it, and guard the reset with a retry counter; after the limit, leave the player in error state and show a retry affordance.

### `HW-04-0013` - clearCache concatenates the cache directory and the file name without a separator, so it never deletes anything

- Category B, severity medium, confidence confirmed
- Features: EDU-01
- Document: `huawei_industry_tree/04_education/docs/01_practice-educate-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture-v1-0000001904563108
- Why: Every fs.statSync on the malformed path throws, the catch swallows it into a log line, and the 清除缓存 (clear cache) menu item silently does nothing while appearing to succeed. The outer fs.listFile promise has no catch either, so a failure to list is an unhandled rejection.
- Fix: Join with a path separator and add a catch to the listFile promise.

### `HW-04-0017` - The sample never filters anything: the confirm button has no handler and the school list ignores the selected range

- Category E, severity medium, confidence confirmed
- Features: EDU-03
- Document: `huawei_industry_tree/04_education/docs/03_adjusting_score_interval_screening_schools.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/adjusting_score_interval_screening_schools-0000002294530662
- Why: The document states the filtering scenario as the point of the sample, and the sample implements only the slider widget. A reader copying this gets a range picker whose value goes nowhere, and has to discover the missing half themselves.
- Fix: Either wire the button - lift leftValue/rightValue into FilterPage (they already live there) and have SchoolPage take them as @Prop and filter its list - or narrow the 场景介绍 to say the sample demonstrates the range control only.

### `HW-04-0018` - This EntryAbility reads the avoid areas once while the industry architecture sample subscribes to avoidAreaChange

- Category F, severity medium, confidence confirmed
- Features: EDU-03
- Document: `huawei_industry_tree/04_education/docs/03_adjusting_score_interval_screening_schools.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/adjusting_score_interval_screening_schools-0000002294530662
- Why: Two documents in the same industry publish the same insets boilerplate in two different shapes, and both shapes are incomplete in opposite ways. Here the values are captured once, so a rotation, a fold or a switch between gesture and button navigation leaves ScreenPage padding by a stale topRectHeight and the content sliding under the status bar. In EDU-01 the subscription exists but is never released (HW-04-0008).
- Fix: Adopt the subscribing form from EDU-01 here, and add the matching off() there, so both samples show the complete pattern.

### `HW-04-0021` - The weekday key generator types its parameter as string while the array holds Resource, so every key stringifies to the same prefix

- Category C, severity medium, confidence confirmed
- Features: EDU-04
- Document: `huawei_industry_tree/04_education/docs/04_horizental_and_vertical_scrolling_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/horizental_and_vertical_scrolling_list-0000002296361950
- Why: The element is a Resource object, not a string. Concatenating it yields the same '[object Object]' prefix for all seven days, so the only thing distinguishing the keys is the timestamp - and once two days are generated inside the same millisecond the keys collide outright. The declared parameter type also disagrees with the array type, which the itemGenerator on the line above gets right (item: Resource).
- Fix: Key by index, which is stable and unique for a fixed seven-element header.

### `HW-04-0022` - The open-on-today scroll never fires on Sunday because Date.getDay returns 0 for it

- Category B, severity medium, confidence confirmed
- Features: EDU-04
- Document: `huawei_industry_tree/04_education/docs/04_horizental_and_vertical_scrolling_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/horizental_and_vertical_scrolling_list-0000002296361950
- Why: Date.getDay() returns 0 for Sunday, not 7. Sunday therefore fails the > 1 guard and the table opens on Monday - the column furthest from today - which is precisely the case the feature exists to handle. Saturday works only by coincidence: getDay() 6 maps to column index 5, which is Saturday under this Monday-first ordering.
- Fix: Convert to a Monday-first index once, at the declaration, and scroll to `col * 120` with no guard other than col > 0.

### `HW-04-0025` - The layout sketch puts the weekday header and the two left rails in one Row, while the shipped page splits them across two absolutely positioned Rows

- Category E, severity medium, confidence confirmed
- Features: EDU-04
- Document: `huawei_industry_tree/04_education/docs/04_horizental_and_vertical_scrolling_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/horizental_and_vertical_scrolling_list-0000002296361950
- Why: The 42 vp offset and the absolute positioning are what keep the weekday strip pinned above a grid that scrolls under it, and the sketch shows neither. A reader building from the sketch gets the weekday strip scrolling away with the content, which is the defect the design exists to avoid.
- Fix: Redraw the snippet from the shipped CourseTablePage.ets, keeping the .position() calls and the zIndex on the header.

### `HW-04-0031` - The line-counting loop cannot terminate when a single character is wider than the available width

- Category B, severity medium, confidence confirmed
- Features: EDU-05
- Document: `huawei_industry_tree/04_education/docs/05_spread_all_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/spread_all_text-0000002297066498
- Why: The only exit from the outer while (true) is the labelled break, reached when the remaining text fits on one line. When the very first character already exceeds textWidth the branch fires with i === 1, so startIndex advances by i - 1 === 0, the inner loop restarts from the same index and the condition is identical. The loop then spins on the UI thread with lineNum incrementing without bound - the application hangs on page entry. It is reachable whenever textWidth is small or zero, for example in a narrow floating window or on a foldable where display.width is not the window width.
- Fix: Clamp the advance to at least one character and add a guard for textWidth <= 0. Better, use measureTextSize with constraintWidth and maxLines, which performs the line breaking natively instead of by hand.

### `HW-04-0032` - The biography is truncated by string index, which the MeasureUtils reference warns splits multi-code-point characters

- Category B, severity medium, confidence confirmed
- Features: EDU-05
- Document: `huawei_industry_tree/04_education/docs/05_spread_all_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/spread_all_text-0000002297066498
- Why: The sample text is CJK and Latin only, so the defect is invisible here - but this is a template for arbitrary user-supplied biographies. An emoji or any astral-plane character landing on the cut point is split into a lone surrogate, which renders as a replacement glyph and throws the measurement off.
- Fix: Walk the string with Array.from or a code-point-aware iterator and cut on code-point boundaries.

### `HW-04-0033` - This page converts the avoid areas into its own @StorageProp fields, while the rest of the industry converts at the point of use

- Category F, severity medium, confidence confirmed
- Features: EDU-05
- Document: `huawei_industry_tree/04_education/docs/05_spread_all_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/spread_all_text-0000002297066498
- Why: @StorageProp is a one-way link from AppStorage into the component: the framework overwrites the local copy whenever the key changes. Storing vp in a field bound to a px key means the field silently reverts to px on the next AppStorage write, and the padding jumps by the pixel-density factor. Four documents in one industry publish the same avoid-area boilerplate and this is the only one that mutates the bound field.
- Fix: Drop the two assignments in aboutToAppear and wrap the two usages at lines 331-332 in px2vp.

### `HW-04-0036` - The avoidAreaChange handler passes a fresh RectHeight to AppStorageV2.connect, which returns the stored object and discards it

- Category B, severity medium, confidence confirmed
- Features: EDU-06
- Document: `huawei_industry_tree/04_education/docs/06_stackable_word_cards.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/stackable_word_cards-0000002349040357
- Why: The third argument is a default constructor, used only when the key is absent. onWindowStageCreate has already stored a RectHeight under both keys, so every later connect returns that stored object and throws the newly constructed one away. The subscription therefore does nothing: the avoid-area heights are frozen at their launch values, and the page padding does not follow a rotation or a fold.
- Fix: Connect once, keep the returned instance, and assign to its @Trace height property inside the callback - which is what makes the change propagate to the pages.

### `HW-04-0037` - Both pages copy the height number out of the connected object, which severs the @Trace link that would deliver updates

- Category C, severity medium, confidence confirmed
- Features: EDU-06
- Document: `huawei_industry_tree/04_education/docs/06_stackable_word_cards.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/stackable_word_cards-0000002349040357
- Why: @Trace propagates changes through the object that holds the property. Reading .height at field-initialisation time copies a number, and the copy has no link back to the stored object - the whole point of wrapping a plain number in an @ObservedV2 class is lost. Combined with the connect defect above, the avoid-area values are doubly frozen: the store is never updated, and the pages could not see it if it were.
- Fix: Hold the connected object in the @Local field and read .height where it is consumed.

### `HW-04-0038` - Deleting the last card leaves left at -1 and computes right modulo zero, producing NaN indices

- Category B, severity medium, confidence confirmed
- Features: EDU-06
- Document: `huawei_industry_tree/04_education/docs/06_stackable_word_cards.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/stackable_word_cards-0000002349040357
- Why: When the last card is removed arr.length is 0, so left becomes -1 and right becomes (n + 1 + 0) % 0, which is NaN. Any later read of this.arr[this.left] or this.arr[this.right] is undefined, and assigning .opacityX on it throws. The arrow buttons guard against this; the two pan handlers do not, and they are still reachable because the ForEach did not drop the deleted card's node (see the finding above).
- Fix: After the splice, bail out when the array is empty and reset left/middle/right to 0; add the same length guard the buttons use to both pan handlers.

### `HW-04-0039` - The initial deck is laid out at 52 vp per card and every refresh at 50 vp, so the stack shifts on the first interaction

- Category B, severity medium, confidence confirmed
- Features: EDU-06
- Document: `huawei_industry_tree/04_education/docs/06_stackable_word_cards.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/stackable_word_cards-0000002349040357
- Why: The two step sizes disagree by 2 vp per card, so the deck is drawn at 0/52/104 on entry and re-laid out at 0/50/100 the first time the user swipes or deletes - a visible 4 vp jump on the third card that no animation covers. Nothing else in the three layout formulas differs, which makes the 52 look like a typo rather than a deliberate initial offset.
- Fix: Extract CARD_OFFSET = 50 and use it in the constructor and both loops.

### `HW-04-0041` - route_map.json registers the word-card page under the name PageOne while the code pushes and matches MemorizeWordsPage

- Category E, severity medium, confidence confirmed
- Features: EDU-06
- Document: `huawei_industry_tree/04_education/docs/06_stackable_word_cards.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/stackable_word_cards-0000002349040357
- Why: Navigation resolves pushPathByName against the in-component navDestination builder first, so the page happens to open - but the route map, which is the fallback and the cross-module contract, names a route that nothing pushes and would never fire. A NavDestination-rooted component listed in main_pages is also loadable by windowStage.loadContent, where it would render with no Navigation parent. Both entries are copy-paste leftovers, down to the description "this is PageOne" duplicated on the EntryPage entry.
- Fix: Rename the route to MemorizeWordsPage, fix the two descriptions, and drop pages/MemorizeWordsPage from main_pages.json.

### `HW-04-0044` - The answer arrays are provided empty, so the guards written against NO_ANSWER read undefined and one of them assigns undefined into a number field

- Category B, severity medium, confidence confirmed
- Features: EDU-07
- Document: `huawei_industry_tree/04_education/docs/07_english_practice.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/english_practice-0000002315012362
- Why: For an unanswered question myAnswer[index] is undefined, not 0. The guard "!== 0" is therefore true on first entry to every question and assigns undefined into a field declared number, which the Radio then compares with ".checked(this.selected === index1 + 1)". It happens to render nothing checked, so the defect is silent - but selected is typed number and holds undefined, and NO_ANSWER, the constant written for exactly this test, is never the value present. The same mismatch makes ExamPage.ets:123's "this.myAnswer[this.index] === 0" branch dead.
- Fix: Size both arrays when the questions load: this.myAnswer = new Array(listData.length).fill(NO_ANSWER), and the same for isSubmitted.

### `HW-04-0045` - The submit button's enable condition reads 'no answer given' as a reason to enable it

- Category B, severity medium, confidence confirmed
- Features: EDU-07
- Document: `huawei_industry_tree/04_education/docs/07_english_practice.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/english_practice-0000002315012362
- Why: The first clause enables 提交答案 (submit answer) precisely when the student has selected nothing, which is when it should be disabled. The condition only behaves acceptably today because myAnswer is an empty array and the element is undefined rather than 0 - fix the initialisation (see the finding above) and the submit button becomes tappable on an unanswered question, submitting a selection of undefined and incrementing the day's counter.
- Fix: State the condition positively: enabled while the question is unsubmitted and an option has been chosen.

### `HW-04-0052` - The save page packs the image without awaiting, so tapping Save early writes undefined to the gallery file

- Category B, severity medium, confidence confirmed
- Features: EDU-08
- Document: `huawei_industry_tree/04_education/docs/08_pdf_to_long_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pdf_to_long_image-0000002349147473
- Why: Packing a full-length PNG takes long enough for the page to render and the user to tap Save. Until it resolves, arrayBuffer is undefined, so createAsset has already created an empty asset in the gallery and fileIo.write receives undefined - the user gets a success toast and a zero-byte image. The dropped promise also swallows a packing failure, and packToData is called with a possibly-undefined pixelMap because AppStorage.get returns T | undefined.
- Fix: Await the pack, guard on the pixelMap being present, catch failures, and gate the SaveButton on the buffer being ready.

### `HW-04-0053` - A failure before the page loop leaves isConverting true and the convert button stuck on the loading state

- Category B, severity medium, confidence confirmed
- Features: EDU-08
- Document: `huawei_industry_tree/04_education/docs/08_pdf_to_long_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pdf_to_long_image-0000002349147473
- Why: image.createPixelMap throws on an out-of-range size, and maxPageWidth x totalHeight x 4 for a long document is exactly the value that goes out of range. When it does, isConverting is never cleared, the button is permanently replaced by a spinner and the page must be left and re-entered. The user is given no indication that anything failed.
- Fix: Wrap the conversion in try/finally, clear isConverting in the finally, and toast the failure.

### `HW-04-0055` - The step-2 snippet is not compilable and the step-1 and step-3 snippets contradict the shipped code

- Category E, severity medium, confidence confirmed
- Features: EDU-08
- Document: `huawei_industry_tree/04_education/docs/08_pdf_to_long_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pdf_to_long_image-0000002349147473
- Why: The step-2 listing cannot be pasted into an editor and compiled - the reader has to reconstruct the block structure and re-add the error handling. The two smaller mismatches point the reader at the wrong option values: SaveDescription.SAVE and SAVE_TO_GALLERY render different button captions, and showScroll flips the PDF scrollbar.
- Fix: Regenerate all three snippets from PdfPage.ets and SavePage.ets.

### `HW-04-0056` - The whole page-by-page conversion runs in the click handler on the UI thread

- Category C, severity medium, confidence confirmed
- Features: EDU-08
- Document: `huawei_industry_tree/04_education/docs/08_pdf_to_long_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pdf_to_long_image-0000002349147473
- Why: getPagePixelMap rasterises a PDF page; doing that for every page of a document inside a click handler blocks frames on the UI thread, so the LoadingProgress the code puts up to signal work in progress is itself unable to animate smoothly. It also means a large document can trip the watchdog rather than merely being slow.
- Fix: Run the conversion on a worker and drive the progress indicator from posted updates, so the spinner the design already calls for actually spins.

### `HW-04-0061` - The import counter is incremented before the file is opened, so a failed import shifts every later file name by one slot

- Category B, severity medium, confidence confirmed
- Features: EDU-09
- Document: `huawei_industry_tree/04_education/docs/09_import_pdf.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/import_pdf-0000002378274393
- Why: index and uriPrint advance unconditionally while fileName and recentDocList advance only on success. After one failed open, fileName has a hole at the skipped slot, uriPrint holds a URI with no corresponding entry, and index no longer equals recentDocList.length - so this.fileName[this.index-1] writes past the end and the two lists that are supposed to be parallel are not.
- Fix: Move this.index++ and the uriPrint push inside the success branch, or replace the manual index with fileName.push(file.name).

### `HW-04-0062` - Documents are keyed in the URI map by file name, so two files with the same name from different folders collide

- Category B, severity medium, confidence confirmed
- Features: EDU-09
- Document: `huawei_industry_tree/04_education/docs/09_import_pdf.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/import_pdf-0000002378274393
- Why: File names are not unique across directories, and 试卷.pdf is exactly the kind of name a student picks twice from different folders. The second import silently overwrites the first entry in uriMap while both appear in the recent list, so two visibly different rows open the same document and the first file becomes unreachable.
- Fix: Use the URI as the map key or as the navigation parameter directly; display the name, address by URI.

### `HW-04-0063` - Both file-open sites swallow the exception in an empty catch, and the picker promise has no rejection handler

- Category B, severity medium, confidence confirmed
- Features: EDU-09
- Document: `huawei_industry_tree/04_education/docs/09_import_pdf.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/import_pdf-0000002378274393
- Why: Every failure path in this feature is silent. A cancelled or failed picker rejects and nothing handles it; a file that cannot be opened produces no log, no message and no visual change, so the user taps 导入 and nothing happens. Publishing empty catch blocks in sample code teaches the pattern.
- Fix: Replace both empty catches with hilog.error plus user-visible feedback, and add a .catch to the picker promise.

### `HW-04-0065` - The step-2 snippet is presented as free-standing code although it only runs inside the destination's onReady, and step 1 omits the navigation it performs

- Category E, severity medium, confidence confirmed
- Features: EDU-09
- Document: `huawei_industry_tree/04_education/docs/09_import_pdf.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/import_pdf-0000002378274393
- Why: onReady is where the destination first has its NavPathStack and parameter, and it is the only reason the load can happen after the component is declared - the snippet as printed suggests the controller can be loaded before the PdfView exists. Omitting the pushPath hides that a successful import navigates away, which is what the isVisible flag and the empty-state artwork exist for.
- Fix: Reprint both snippets from the shipped files, including the onReady wrapper and the navigation.

### `HW-04-0066` - The document list uses the shared @Consume import counter as its for-loop variable, re-rendering the tree on every iteration and resetting the importer's state

- Category C, severity medium, confidence confirmed
- Features: EDU-09
- Document: `huawei_industry_tree/04_education/docs/09_import_pdf.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/import_pdf-0000002378274393
- Why: @Consume writes back to the @Provide and marks every dependent component dirty, so each of the two assignments per iteration schedules a re-render of both pages while the loop is still running - N documents cause 2N re-render requests for one list build. It also destroys the provider's meaning: index is the importer's running count, and the loop leaves it at uriPrint.length, so the next import writes fileName at a slot derived from the list length rather than the import count.
- Fix: Use local variables for the loop counter and the per-item page count; leave the shared index alone.

### `HW-04-0071` - Comments start off-screen by the display height rather than its width, so each one is invisible for several seconds after it is created

- Category B, severity medium, confidence confirmed
- Features: EDU-10
- Document: `huawei_industry_tree/04_education/docs/10_video_course_bullet_comments.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_course_bullet_comments-0000002378246529
- Why: The value is a horizontal start offset, so it must be the display width. On a portrait phone the height is roughly 2.2 times the width, so every comment begins about 1.2 screen widths beyond the right edge and, at MOVEMENT_SPEED 2 vp per 16 ms tick, spends roughly three seconds travelling before it becomes visible. The generator's 500-2000 ms interval is tuned for comments that appear promptly, so the on-screen density is far lower than intended.
- Fix: Use display.getDefaultDisplaySync().width, or better the measured width of the comment layer.

### `HW-04-0072` - Comment ids are millisecond timestamps, so a user comment sent in the same millisecond as a generated one gets a duplicate id

- Category B, severity medium, confidence confirmed
- Features: EDU-10
- Document: `huawei_industry_tree/04_education/docs/10_video_course_bullet_comments.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_course_bullet_comments-0000002378246529
- Why: getTime() has millisecond resolution and is not unique. Two comments created in the same millisecond share an id, so the .key attribute is ambiguous, and any attempt to fix the missing ForEach key generator by keying on id would inherit the collision - which the LazyForEach and ForEach guides both warn causes the framework to reuse the wrong cached component.
- Fix: Replace the timestamp with a static incrementing counter, which is unique, cheaper, and stable.

### `HW-04-0073` - endGenerator clears the comment array by assigning to length, which @Trace does not observe

- Category C, severity medium, confidence confirmed
- Features: EDU-10
- Document: `huawei_industry_tree/04_education/docs/10_video_course_bullet_comments.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_course_bullet_comments-0000002378246529
- Why: Assigning to length is a property write, not an array API call, so @Trace does not report it and the UI is not re-rendered. Stopping the video therefore empties the model while the comments stay on screen until the next tick of the animation interval happens to reassign the array - and that interval returns early when isPlaying is false, so after a stop they stay frozen on screen indefinitely. Three different mutation styles in one class, and only this one is unobservable.
- Fix: Assign a new empty array, or use splice.

### `HW-04-0074` - The same generator function is printed twice with different bodies and neither listing has balanced braces

- Category E, severity medium, confidence confirmed
- Features: EDU-10
- Document: `huawei_industry_tree/04_education/docs/10_video_course_bullet_comments.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_course_bullet_comments-0000002378246529
- Why: Neither listing compiles, and a reader who takes step 3 as the authoritative version builds a generator with no stop flag - the setTimeout chain then keeps producing comments after the video is paused and after the page is destroyed. Splitting one function across two steps and eliding a different part in each also hides that the position choice and the comment fill happen in the same tick.
- Fix: Print the function once, in full, and describe the position logic and the fill logic as two parts of it.

### `HW-04-0079` - The example-audio path never records the rawfile descriptor it opens, so that descriptor is never released

- Category B, severity medium, confidence confirmed
- Features: EDU-11
- Document: `huawei_industry_tree/04_education/docs/11_oral_english.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/oral_english-0000002349625326
- Why: Two methods open a descriptor and only one of them records it, so the entire cleanup machinery built around this.fileFd is inert for example audio. Every time the student plays the model sentence a new rawfile descriptor is opened and never released; with one sentence per exercise and repeated listening that accumulates for the life of the page. Rawfile descriptors also need closeRawFdSync rather than fileIo.close, so the existing cleanup would be wrong for them even if the field were set.
- Fix: Record which kind of descriptor is open and close it with the matching API - fileIo.close for the sandbox recording, resourceManager.closeRawFdSync for the rawfile.

### `HW-04-0080` - The page-hide guard mixes && and || so the player branch fires on a state the recorder branch owns

- Category B, severity medium, confidence confirmed
- Features: EDU-11
- Document: `huawei_industry_tree/04_education/docs/11_oral_english.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/oral_english-0000002349625326
- Why: && binds tighter than ||, so the condition parses as (manager && PLAYING_RECORD) || PLAYING_EXAMPLE - the null check applies to only one of the two states. The call is written with optional chaining, so it does not crash today, but the guard says something different from what it is meant to say, and the optional chaining is what is masking it. Any later edit that drops the ? turns this into a null dereference on the example-playback path.
- Fix: Parenthesise the two state comparisons.

### `HW-04-0081` - The waveform bar heights are drawn from a random source inside build, so the render is not a function of state

- Category C, severity medium, confidence confirmed
- Features: EDU-11
- Document: `huawei_industry_tree/04_education/docs/11_oral_english.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/oral_english-0000002349625326
- Why: build must be a pure function of state: ArkUI re-runs it whenever any bound value changes, and each run here produces different bar heights even when the amplitude has not moved. The waveform therefore jitters on unrelated re-renders - a sibling state change, a layout pass - rather than only on new amplitude samples, and the animateTo wrapped around the amplitude assignment cannot interpolate a value that is re-randomised underneath it. The dead "+ i * 0" also signals that per-bar variation was meant to come from the index rather than from an RNG call per bar.
- Fix: Derive the bar heights in the amplitude callback into a @State number[] and render that; drop the "+ i * 0".

### `HW-04-0082` - The microphone permission is re-requested on every record tap with no handling of a permanent refusal

- Category D, severity medium, confidence confirmed
- Features: EDU-11
- Document: `huawei_industry_tree/04_education/docs/11_oral_english.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/oral_english-0000002349625326
- Why: After the user has refused permanently, requestPermissionsFromUser still resolves successfully but displays nothing - authResults is -1 and dialogShownResults is false. The student then taps 录音, sees a 未授权 toast, and has no route to fix it from inside the app; the feature is permanently dead with no explanation. Re-raising the request on every tap also means an already-granted permission goes through the full request path each time instead of a cheap token check.
- Fix: Add a checkAccessToken fast path and a requestPermissionOnSetting fallback keyed on dialogShownResults, so a permanently refused permission opens the settings sheet.

### `HW-04-0083` - The documented startRecordingProcess signature is missing its first parameter and the call site passes the wrong number of arguments

- Category E, severity medium, confidence confirmed
- Features: EDU-11
- Document: `huawei_industry_tree/04_education/docs/11_oral_english.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/oral_english-0000002349625326
- Why: The snippet as printed does not compile against the shipped class, and the missing parameter is not incidental - the UIContext is how the method reaches the ability context for requestPermissionsFromUser and the PromptAction for its two toasts. The same snippet also drops the .catch on getAudioCapturerMaxAmplitude and the whole TICK_COUNT_MAX timeout that the real method has, so the documented version records without a bound and swallows amplitude failures.
- Fix: Reprint step 2 from AVRecorderManager.ets:106-142, including the context parameter, the catch and the tick-count cutoff.

### `HW-04-0090` - The document's handler resets the WiFi flag on a cellular bearer but the shipped code omits that branch

- Category E, severity medium, confidence confirmed
- Features: EDU-12
- Document: `huawei_industry_tree/04_education/docs/12_network_monitor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/network_monitor-0000002355490078
- Why: isWifi drives the connection-type indicator. Without the cellular branch it latches: once WiFi has been seen, switching to mobile data leaves the indicator claiming WiFi until the network is lost entirely. The document is right and the code is wrong, which is the reverse of the usual direction and means a reader who trusts the zip over the prose ships the defect.
- Fix: Add the else-if branch to NetMonitor.ets:69-75.

### `HW-04-0094` - getCalendar's promise overload is ignored and the callback overload is hand-wrapped in new Promise twice

- Category A, severity medium, confidence confirmed
- Features: EDU-13
- Document: `huawei_industry_tree/04_education/docs/13_class_add_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/class_add_schedule-0000002358628536
- Why: The promise overload exists precisely so callers do not have to do this. The hand-rolled wrapper is twenty lines repeated twice, it loses the typing that the promise overload provides, and in addToCalendar it also forces an awkward "let calendar: Calendar | undefined" assigned from inside the callback and then re-cast at line 99. It is also what makes deleteFromCalendar impossible to await, since that one was written without the wrapper at all.
- Fix: Call the promise overload directly in all three places and delete the wrappers.

### `HW-04-0095` - The first-run branch writes the event-id array with putSync and never flushes it

- Category B, severity medium, confidence confirmed
- Features: EDU-13
- Document: `huawei_industry_tree/04_education/docs/13_class_add_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/class_add_schedule-0000002358628536
- Why: putSync writes to the in-memory preference set; flush is what persists it to the sandbox file. On the very first launch this branch is the one that creates the 'eventId' key, so if the application is killed before some later flush happens to run, the key is absent again on the next start and every stored reminder id is lost - the app then shows no course as added to the calendar while the events still exist in the system calendar. Three writes in one file, and only this one omits the flush.
- Fix: Flush after the putSync, as the other two write sites do.

### `HW-04-0096` - refreshPreference's second branch ignores the courseShow it was passed and writes an array of -1

- Category B, severity medium, confidence confirmed
- Features: EDU-13
- Document: `huawei_industry_tree/04_education/docs/13_class_add_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/class_add_schedule-0000002358628536
- Why: The method's contract, per its own comment 'Refresh preference based on the change of CourseShow', is to persist the ids the caller holds. When the key is absent it instead discards them and stores -1 everywhere, so any reminder added before the key exists is not recorded. It is masked today only because refreshCourseShow happens to create the key first on every path - which makes this a latent defect that will surface the moment the call order changes.
- Fix: Hoist the array-building loop out of the branch and keep only the deleteSync inside the has-key case.

### `HW-04-0098` - Class start times are offsets from a UTC midnight, so the calendar event lands at the labelled hour only in UTC+8

- Category B, severity medium, confidence confirmed
- Features: EDU-13
- Document: `huawei_industry_tree/04_education/docs/13_class_add_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/class_add_schedule-0000002358628536
- Why: UTC midnight plus thirty minutes is 08:30 only where the offset is +08:00. The offsets silently encode Beijing time into what is presented as a generic timetable: on a device set to any other zone the first period is written into the system calendar at the wrong local hour - eight hours out in UTC, and off by a fraction of an hour in the half-hour zones - while the app's own list still displays 08:30. The reminder then fires at the wrong time, which is the one thing this feature exists to get right.
- Fix: Build dates with the local-time Date constructor and set the class hour with setHours, so the offsets are expressed in the device's own zone.

### `HW-04-0100` - The reading pane is a fixed 640 vp height that also serves as the numerator of every progress calculation

- Category C, severity medium, confidence confirmed
- Features: EDU-14
- Document: `huawei_industry_tree/04_education/docs/14_course_progress_display.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/course_progress_display-0000002358926170
- Why: 640 vp is taller than the content area of a compact phone and much shorter than a tablet's, so on one the reader overflows its card and on the other it leaves the page half empty. Because the same constant is the numerator of the progress ratio, the two cannot be decoupled: fixing the layout by changing the height silently changes what the progress counter means, and sizing the pane responsively breaks the arithmetic entirely.
- Fix: Measure the Scroll with onSizeChange into a viewportHeight field and use that in place of the constant in both ratios.

### `HW-04-0101` - A TextController is constructed and bound to the Text but never used, on a Text that is given both a value and a Span child

- Category C, severity medium, confidence confirmed
- Features: EDU-14
- Document: `huawei_industry_tree/04_education/docs/14_course_progress_display.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/course_progress_display-0000002358926170
- Why: TextController exists to drive styled-string content or to read the layout manager; constructing one and passing it without ever calling a method on it is dead weight that suggests to a reader copying this page that the controller is required for Span content, which it is not. The empty string first argument is also inert - when a Text has child Spans the string value is not rendered - so the line carries two pieces of misleading ceremony.
- Fix: Drop the controller and the empty string value.

### `HW-04-0102` - The chapter text is read with the error argument discarded, so a failed read shows an empty chapter and a 0/0 counter

- Category B, severity medium, confidence confirmed
- Features: EDU-14
- Document: `huawei_industry_tree/04_education/docs/14_course_progress_display.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/course_progress_display-0000002358926170
- Why: getRawFileContent reports failure through its first callback argument, which is the only signal available - the call itself does not throw. Discarding it means a missing or unreadable content.txt produces a blank reading pane and a progress counter reading 0/0, with nothing in the log and no indication to the user that anything went wrong. The chapter also becomes instantly 'complete', because an empty pane is at its end position.
- Fix: Inspect the error argument, log it with hilog, and set a visible failure state.

### `HW-04-0103` - The chapter index is read with getParamByName and cast to number[], repeating the plural-parameter defect from the PDF import sample

- Category F, severity medium, confidence confirmed
- Features: EDU-14
- Document: `huawei_industry_tree/04_education/docs/14_course_progress_display.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/course_progress_display-0000002358926170
- Why: The array is one entry per ReadPage on the stack, not the parameters of this page, so params[0] is the oldest ReadPage's chapter - correct only while exactly one is pushed. The cast to number[] also misdescribes the return type, which is Array<unknown>, so the compiler cannot warn when the pushed parameter shape changes. Two documents in this industry publish the same misuse of this API, which makes it a pattern rather than a slip.
- Fix: Take the parameter from NavDestinationContext.pathInfo.param in onReady, as the Navigation reference intends.

### `HW-04-0109` - None of the three ForEach loops has a key generator, so each school's key is a JSON serialisation of its whole department subtree

- Category C, severity medium, confidence confirmed
- Features: EDU-15
- Document: `huawei_industry_tree/04_education/docs/15_cascading_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cascading_selection-0000002364082736
- Why: Every Item carries its department array, so the default key for one school row is the JSON of that school and all of its departments - recomputed on every render pass, for every row, in three lists at once. The data already has a natural unique key: id. The guide singles out exactly this shape as the reason to supply a key generator.
- Fix: Pass item.id.toString() as the key generator in all three loops.

### `HW-04-0110` - The data file is read and parsed in aboutToAppear with no error handling at all

- Category B, severity medium, confidence confirmed
- Features: EDU-15
- Document: `huawei_industry_tree/04_education/docs/15_cascading_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cascading_selection-0000002364082736
- Why: Two of these three calls throw: getRawFileContentSync raises 9001005 on a bad path and JSON.parse raises on malformed content. Both run inside aboutToAppear, where an exception propagates out of the component's construction rather than being reported, so a corrupt or missing data.json takes the page down instead of showing an empty selector. The project already ships a Logger utility that is never used on this path.
- Fix: Guard getJson with try/catch, log through the existing Logger, and leave schoolList empty on failure.

### `HW-04-0111` - Item.department is declared non-optional but is absent from thirteen of the seventeen entries in the data file

- Category A, severity medium, confidence confirmed
- Features: EDU-15
- Document: `huawei_industry_tree/04_education/docs/15_cascading_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cascading_selection-0000002364082736
- Why: The parsed objects do not match the declared interface, and the cast at line 211 - "JSON.parse(result) as ItemModel" - suppresses any complaint, so the compiler believes department is always present. Because the year rows are in the same array the school column iterates, the click handler that assigns item.department is attached to rows for which it is undefined, and undefined would be written into a field typed Array<DepartmentItem>. The interface should describe the data.
- Fix: Mark department optional and guard the assignment, or model schools and years as distinct types.

### `HW-04-0112` - The implementation write-up shows a DepartmentView component that does not exist and never mentions the third cascade level

- Category E, severity medium, confidence confirmed
- Features: EDU-15
- Document: `huawei_industry_tree/04_education/docs/15_cascading_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cascading_selection-0000002364082736
- Why: A reader cannot locate DepartmentView in the sample because it does not exist, and the two documented steps describe a two-level cascade while the scenario paragraph and the code both describe three. The document also omits the one detail that makes the cascade work - that the second and third columns are gated by isSelectedSchool and isSelectedDepartment and that every click resets the levels below it.
- Fix: Reprint both snippets from SelectionView.ets, add the third level, and describe the downward reset performed by each column's handler.

### `HW-04-0120` - The XComponent, the surface rectangle and the preview profile are all set to different aspect ratios

- Category C, severity medium, confidence confirmed
- Features: EDU-16
- Document: `huawei_industry_tree/04_education/docs/16_equipment_detection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/equipment_detection-0000002398440861
- Why: Three numbers describe the same picture and no two of them agree: the surface is 1170:2080 (about 9:16), the component is 1200:1920 (5:8) and the profile is 1920:1080 (16:9). The guide makes matching ratios a requirement, not a preference, so the preview is stretched, and the negative 200 vp top margin on the component suggests the mismatch was compensated for by eye rather than fixed. The constant name RESOLUTION_WIGHT is also misspelled, which makes the pair harder to find.
- Fix: Derive all three from the chosen preview profile, or use .renderFit(RenderFit.RESIZE_CONTAIN) as the guide suggests instead of setting the component size by hand.

### `HW-04-0124` - A throwaway AVPlayUtil instance is constructed to call an instance method, orphaning the file handle the class exists to hold

- Category B, severity medium, confidence confirmed
- Features: EDU-16
- Document: `huawei_industry_tree/04_education/docs/16_equipment_detection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/equipment_detection-0000002398440861
- Why: The class exists to own the file handle, and every call site throws the owner away immediately. The instance that opened the descriptor is unreachable the moment playAudio returns, so endAudio can never be called on it; the completed handler's fresh instance has file set to null! and would throw if it did try to close. Constructing an object to call what is effectively a static function also hides that the player itself is passed in from outside, so the class owns nothing it can manage.
- Fix: Keep a single AVPlayUtil field on the sheet, pass it the player once, and route completed through that instance's endAudio.

### `HW-04-0127` - Two setTimeout handles are discarded, and the one that pops the navigation stack can fire after the user has already left

- Category B, severity medium, confidence confirmed
- Features: EDU-17
- Document: `huawei_industry_tree/04_education/docs/17_word_spelling.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/word_spelling-0000002368882740
- Why: The second timer calls pathInfos.pop() 1500 ms after the last word. If the learner presses back during the toast the page pops immediately and the timer still fires, popping a second entry and taking them one page further back than they asked. Both timers also hold the component alive and write to its state after it is gone; the tips timer additionally re-triggers the @Watch on showTips, which can call goToNextWord on a destroyed page.
- Fix: Assign both timer ids to private fields and clearTimeout them in aboutToDisappear.

### `HW-04-0128` - Two samples in this industry ship divergent copies of AVPlayerManager, and only this one releases the rawfile descriptor

- Category F, severity medium, confidence confirmed
- Features: EDU-17
- Document: `huawei_industry_tree/04_education/docs/17_word_spelling.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/word_spelling-0000002368882740
- Why: The same class, in the same industry, exists in two versions where one has a resource leak the other has fixed - and nothing in either document says which is current. A reader who takes the OralEnglish copy because they met it first inherits a descriptor leak that was already solved two documents later. Both copies also still omit "this.avPlayer = undefined" after the release in stopAudio, so the divergence is partial rather than a clean supersession.
- Fix: Publish one AVPlayerManager across the industry's samples, with the closeRawFdSync handling from this copy and the avPlayer field cleared after every release.

### `HW-04-0129` - The word Swiper uses ForEach with no key generator, so every card is built up front and keyed by a JSON serialisation

- Category C, severity medium, confidence confirmed
- Features: EDU-17
- Document: `huawei_industry_tree/04_education/docs/17_word_spelling.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/word_spelling-0000002368882740
- Why: WordDetail carries the word, its translation, its pronunciation and its audio path, so the default key is a JSON serialisation of all of it, recomputed per render pass per card. Because it is ForEach rather than LazyForEach, every card in the list is also constructed at page entry - including a TextInput each - when only one is ever visible. EDU-07 in this same industry demonstrates the LazyForEach-in-Swiper form for exactly this shape of screen.
- Fix: Supply a key generator now, and move to LazyForEach with cachedCount if the word list grows beyond a demo size.

### `HW-04-0132` - The monthly formatter guards on a zero-based month, so every label falls back to a placeholder throughout January

- Category B, severity medium, confidence confirmed
- Features: EDU-18
- Document: `huawei_industry_tree/04_education/docs/18_track_study_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/track_study_time-0000002391113388
- Why: Date.getMonth() returns 0 for January, which is falsy, so for the whole of January the first condition fails on every call and all seven x-axis labels render as 本月 (this month). The value is otherwise treated as one-based inside the branch - the label text is the number followed by 月 - so the constructor is passing the wrong convention and the guard then hides it for one month a year. Guarding an optional number with truthiness rather than an undefined check is the underlying mistake.
- Fix: Construct with today.getMonth() + 1 and change the guard to this.month !== undefined.

### `HW-04-0133` - The monthly formatter computes its offsets from the weekly constant, and is correct only because the two happen to be equal

- Category C, severity medium, confidence confirmed
- Features: EDU-18
- Document: `huawei_industry_tree/04_education/docs/18_track_study_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/track_study_time-0000002391113388
- Why: The month formatter is a copy of the week formatter with the constant only half renamed. It produces correct labels today purely because X_WEEK and X_MONTH are both 7; raising X_MONTH to 12 to show a year - which the class anticipates, since it also declares X_YEAR = 12 - would extend the axis while leaving the labels computed from a seven-point window, so the first months would repeat. The mixed constants also make the intent unreadable: nothing in the month formatter should refer to a week.
- Fix: Replace X_WEEK with X_MONTH in the three offset expressions.

### `HW-04-0134` - The constraints section omits the third-party OHPM dependency the whole sample is built on

- Category E, severity medium, confidence confirmed
- Features: EDU-18
- Document: `huawei_industry_tree/04_education/docs/18_track_study_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/track_study_time-0000002391113388
- Why: Every other sample in this industry builds from the SDK alone, so a reader has no reason to expect an external dependency. Opening this zip without running ohpm install produces a wall of unresolved imports from a package the constraints never mention, and the caret range ^3.0.11 means two developers can resolve different versions of a charting library whose axis and formatter APIs the sample uses directly.
- Fix: Add the dependency and the ohpm install step to 约束与限制, and pin the version exactly rather than with a caret range.

### `HW-04-0135` - The chart library is declared in the project-level package file while the only module that imports it declares no dependencies

- Category C, severity medium, confidence confirmed
- Features: EDU-18
- Document: `huawei_industry_tree/04_education/docs/18_track_study_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/track_study_time-0000002391113388
- Why: A module's own oh-package.json5 is the manifest of what that module needs; putting the dependency only at the project level means the entry module's manifest claims it has none, so extracting or reusing the module carries no record of the requirement. The other samples in this industry that ship multi-module structures - Education_Framework_Code_V1 - declare each module's dependencies in that module's own file, so this is a deviation within the industry as well as from the convention.
- Fix: Move the dependency into entry/oh-package.json5.

### `HW-04-0136` - The documented @Watch refresh path can never run in the shipped sample

- Category E, severity medium, confidence confirmed
- Features: EDU-18
- Document: `huawei_industry_tree/04_education/docs/18_track_study_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/track_study_time-0000002391113388
- Why: onDataChange fires only when trendData changes on a mounted component, and neither of the two things that could cause that happens: the arrays are constant literals, and the period switch replaces the component rather than re-binding it. The document's step 3 therefore describes a path the reader cannot exercise, while the mechanism that does drive the redraw - full reconstruction through the if/else, with the axis formatters and label counts chosen in aboutToAppear - is not described at all.
- Fix: Describe the if/else reconstruction and the aboutToAppear configuration, and keep the @Watch step only if the sample is changed so the data can actually change under a live chart.

### `HW-04-0143` - The depth-fade opacity is the one animated property that is not @Trace

- Category B, severity medium, confidence confirmed
- Features: EDU-19
- Document: `huawei_industry_tree/04_education/docs/19_globe_label_animation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/globe_label_animation-0000002529698917
- Why: alpha is the property that produces the near-large-far-faint illusion the document describes as the point of the effect, and it is the only animated one the framework is not told to observe - so a change to it cannot re-render the bound opacity. It appears to work only because the ForEach key is JSON.stringify(item), which changes every frame and rebuilds the node from scratch. Fix that key, as it must be fixed, and the depth fade stops updating; the two defects are hiding each other.
- Fix: Decorate alpha with @Trace, so the opacity binding updates on its own once the key is corrected.

### `HW-04-0144` - A purely horizontal or vertical drag changes neither rotation angle because the two are guarded with a conjunction

- Category B, severity medium, confidence confirmed
- Features: EDU-19
- Document: `huawei_industry_tree/04_education/docs/19_globe_label_animation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/globe_label_animation-0000002529698917
- Why: A drag straight across the screen produces offsetY of zero, so changeY is zero, the conjunction fails and neither angle is updated - the sphere keeps its previous rotation instead of following the finger. The same happens for a vertical drag. Horizontal and vertical are the two most natural gestures for spinning a globe, and both are the ones that do nothing; the conjunction should be two independent guards because the two axes are independent.
- Fix: Split the condition into two separate assignments.

### `HW-04-0145` - The inertia decay runs on a 500 ms interval against a 17 ms animation, so it steps once every thirty frames

- Category C, severity medium, confidence confirmed
- Features: EDU-19
- Document: `huawei_industry_tree/04_education/docs/19_globe_label_animation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/globe_label_animation-0000002529698917
- Why: A factor of 0.7 applied every 500 ms means the rotation speed holds constant for thirty frames and then drops by 30 percent in one frame, which reads as a series of jerks rather than as inertia. The whole point of a release-and-coast animation is a continuous decay, and the file already establishes 17 ms as its frame budget. Assigning null to speedTimer, declared as number, is also a type violation that the strict null checking would reject.
- Fix: Run the decay inside the existing 17 ms rotation tick with a per-frame factor, and type the timer handles as number with 0 as the empty value.

### `HW-04-0151` - Only the cancel code is distinguished, and every other outcome indexes uris[0] without checking the array

- Category B, severity medium, confidence confirmed
- Features: EDU-20
- Document: `huawei_industry_tree/04_education/docs/20_scan_submit_homework.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/scan_submit_homework-0000002337721418
- Why: The handler assumes the only two outcomes are cancel and success. A scanner failure that reports some other code takes the success path, where an empty uris array makes uris[0] undefined and accessSync is called on undefined. Nothing tells the user that the scan failed either - the page simply stays on the scanner with no feedback, which is indistinguishable from the inverted-guard defect above.
- Fix: Switch on the code: 0 for success, -1 for cancel, anything else to a toast plus pop; and guard uris.length > 0 before indexing.

### `HW-04-0152` - A multi-megabyte PDF is copied synchronously in 4 KB chunks on the UI thread

- Category B, severity medium, confidence confirmed
- Features: EDU-20
- Document: `huawei_industry_tree/04_education/docs/20_scan_submit_homework.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/scan_submit_homework-0000002337721418
- Why: Thirty scanned pages is a PDF of several megabytes, copied 4096 bytes at a time with two synchronous syscalls per chunk - on the order of a thousand blocking round trips - all inside a UI callback. The page freezes for the duration with no indication that anything is happening, and the user is looking at the scanner UI while it does. fs.copyFile exists for exactly this and does not need the chunk loop at all.
- Fix: Replace the loop with fs.copyFile, and move the call off the UI thread if the file can be large.

### `HW-04-0153` - The copy helper's catch block is empty, marked only by a comment promising error handling

- Category B, severity medium, confidence confirmed
- Features: EDU-20
- Document: `huawei_industry_tree/04_education/docs/20_scan_submit_homework.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/scan_submit_homework-0000002337721418
- Why: readWriteFile returns void and swallows every failure, so DocScan.ets:63-66 sets pdfFileExist = true and pops with a success result whether or not the file was actually written. A disk-full or permission failure therefore presents to the student as a submitted homework that does not exist. Publishing an empty catch with a comment saying error handling belongs here teaches the shape without the substance.
- Fix: Log the error and return a boolean, and have DocScan mark the submission only when the copy reports success.

### `HW-04-0004` - The project tree lists ResManager.ets but both real files are named 'ResManager .ets' with an embedded space

- Category E, severity low, confidence confirmed
- Features: EDU-01
- Document: `huawei_industry_tree/04_education/docs/01_practice-educate-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture-v1-0000001904563108
- Why: The documented tree does not match the shipped project. A space before the extension propagates into every import specifier - three files under features/login import '../common/resources/ResManager ' - so a developer who renames the file to the documented name breaks four import statements with no obvious cause.
- Fix: Rename both files to ResManager.ets and update common/basic/Index.ets:20, LogoutView.ets:16, LogoutNextView.ets:16 and ModifyPasswordView.ets:16.

### `HW-04-0014` - The play-when-ready interval in AVPlayerDemo is never cleared if the component leaves before the video loads

- Category B, severity low, confidence confirmed
- Features: EDU-01
- Document: `huawei_industry_tree/04_education/docs/01_practice-educate-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture-v1-0000001904563108
- Why: clearInterval runs only inside the success branch. If the user leaves the course page while the video is still preparing, the timer keeps firing every 100 ms against a released AVPlayer for the lifetime of the process.
- Fix: Store the handle in a private field and clear it in aboutToDisappear; better, drive playback from the 'prepared' branch of the stateChange handler, which already sets this.flag.

### `HW-04-0016` - The confirm button sets alignRules inside a Column and anchors to an id that no component declares

- Category C, severity low, confidence confirmed
- Features: EDU-03
- Document: `huawei_industry_tree/04_education/docs/03_adjusting_score_interval_screening_schools.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/adjusting_score_interval_screening_schools-0000002294530662
- Why: The universal-attributes reference states for alignRules: "This attribute only takes effect when the parent container is RelativeContainer." The parent is a Column, and the anchor id does not exist, so the attribute is dead code that misleads anyone copying this component into a RelativeContainer.
- Fix: Remove the alignRules line from the Button.

### `HW-04-0024` - The project tree lists the entryability directory twice, the second time holding EntryBackupAbility.ets

- Category E, severity low, confidence confirmed
- Features: EDU-04
- Document: `huawei_industry_tree/04_education/docs/04_horizental_and_vertical_scrolling_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/horizental_and_vertical_scrolling_list-0000002296361950
- Why: The tree names two different directories identically, so it does not describe the shipped project; a reader creating the directories from it puts both files in one place and the module.json5 srcEntry "./ets/entrybackupability/EntryBackupAbility.ets" no longer resolves.
- Fix: Rename the second tree entry to entrybackupability.

### `HW-04-0026` - The tab bar modifier applies alignRules, an attribute that only takes effect inside a RelativeContainer

- Category C, severity low, confidence confirmed
- Features: EDU-04
- Document: `huawei_industry_tree/04_education/docs/04_horizental_and_vertical_scrolling_list.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/horizental_and_vertical_scrolling_list-0000002296361950
- Why: The Tabs bar is not a RelativeContainer child, and the rule object is empty in any case, so the whole @State field, the aboutToAppear call and the barModifier binding do nothing. Shipped as sample code it suggests barModifier is required for a custom tab bar, which it is not.
- Fix: Remove tabBarModifier and the barModifier option.

### `HW-04-0034` - backup_config.json ships with no EntryBackupAbility and no extensionAbilities entry to reference it

- Category E, severity low, confidence confirmed
- Features: EDU-05
- Document: `huawei_industry_tree/04_education/docs/05_spread_all_text.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/spread_all_text-0000002297066498
- Why: A profile resource that nothing references is packaged into the HAP for no reason and suggests a backup extension exists when it does not. The document's project tree does not mention it either way, so a reader comparing the tree to the zip has to work out which of the two is right.
- Fix: Delete entry/src/main/resources/base/profile/backup_config.json, or restore the EntryBackupAbility to match the other samples in this industry.

### `HW-04-0040` - The horizontal pan handler starts a 500 ms animation on every frame, including the frames where nothing changes

- Category B, severity low, confidence confirmed
- Features: EDU-06
- Document: `huawei_industry_tree/04_education/docs/06_stackable_word_cards.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/stackable_word_cards-0000002349040357
- Why: onActionUpdate fires once per frame for the whole drag. Every one of those frames opens a 500 ms animation whose closure usually changes nothing, so the framework schedules and tears down an animation per frame for no effect. The two handlers in the same file disagree on where the guard belongs, and the vertical one is right.
- Fix: Move the offsetX checks out of the closure and call animateTo only when a card actually moves.

### `HW-04-0042` - The project tree lists the entryability directory twice, the second time holding EntryBackupAbility.ets

- Category E, severity low, confidence confirmed
- Features: EDU-06
- Document: `huawei_industry_tree/04_education/docs/06_stackable_word_cards.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/stackable_word_cards-0000002349040357
- Why: The tree names two different directories identically, so it does not describe the shipped project. The same defect appears verbatim in 04_horizental_and_vertical_scrolling_list.md, so it is a template error propagating across this industry's documents rather than a one-off typo.
- Fix: Rename the second tree entry to entrybackupability, here and in the other documents that reuse this tree template.

### `HW-04-0046` - The LazyForEach key is the whole question stem, which a real question bank cannot guarantee to be unique

- Category C, severity low, confidence confirmed
- Features: EDU-07
- Document: `huawei_industry_tree/04_education/docs/07_english_practice.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/english_practice-0000002315012362
- Why: The ten sample stems happen to be distinct, so the sample runs. A question bank routinely repeats a stem across variants - the same sentence with different distractors is a standard item-writing technique - and two identical keys in a Swiper with cachedCount 2 make the framework reuse the wrong cached component. Using a 200-character sentence as a hash key is also needlessly expensive per diff.
- Fix: Add an id field to the question JSON and key on it, or fold the index into the key as the guide's corrected example does.

### `HW-04-0047` - This page binds the avoid areas with @StorageLink while the rest of the industry uses the read-only @StorageProp

- Category F, severity low, confidence confirmed
- Features: EDU-07
- Document: `huawei_industry_tree/04_education/docs/07_english_practice.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/english_practice-0000002315012362
- Why: @StorageLink is a two-way binding: a write in the component propagates back into AppStorage and out to every other consumer. Nothing here writes, so the extra capability buys nothing and creates a hazard - a later edit that assigns to topRectHeight would silently corrupt the shared avoid-area value for the whole application. @StorageProp expresses the read-only intent and cannot do that. Five documents in this industry publish this boilerplate and this is the only one that makes it writable.
- Fix: Change both declarations to @StorageProp.

### `HW-04-0054` - A DecodingOptions object is built per page purely to read back the size that is already in hand, and its pixel format is never applied

- Category B, severity low, confidence confirmed
- Features: EDU-08
- Document: `huawei_industry_tree/04_education/docs/08_pdf_to_long_image.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pdf_to_long_image-0000002349147473
- Why: DecodingOptions configures image.createPixelMap from an ImageSource. Here the PixelMap comes from page.getPagePixelMap(), so nothing consumes these options and the declared BGRA_8888 is never applied - readPixelsToBuffer returns the PixelMap's own format. The object exists only so its desiredSize can be read back through non-null assertions, and it advertises a pixel-format conversion the code does not perform, next to a composite declared RGBA_8888.
- Fix: Delete singleOpts and size the buffer from pageWidth and pageHeight directly.

### `HW-04-0064` - The import action is dispatched by comparing two localized display strings

- Category C, severity low, confidence confirmed
- Features: EDU-09
- Document: `huawei_industry_tree/04_education/docs/09_import_pdf.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/import_pdf-0000002378274393
- Why: Control flow depends on two resolved strings matching, so behaviour is tied to the active locale and to two resource entries staying textually identical. Adding a translation that differs by whitespace disables the import button with no error. Resolving a string resource on every list-item tap and every tab-bar rebuild is also needless work for a comparison that should be an integer.
- Fix: Add an action field to the list model and compare that; keep the resource for display only.

### `HW-04-0067` - Every imported document is copied into the sandbox with a doubled extension

- Category B, severity low, confidence confirmed
- Features: EDU-09
- Document: `huawei_industry_tree/04_education/docs/09_import_pdf.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/import_pdf-0000002378274393
- Why: fs.File.name is the full base name including the suffix, so a picked 试卷1.pdf is written to the sandbox as 试卷1.pdf.pdf. The copy is invisible in this sample because nothing reads it back, but it is the file a real application would list, share or re-open, and the name it shows would not match the one the recent-document list displays.
- Fix: Append the extension only when file.name lacks it.

### `HW-04-0075` - Cosmetic randomness is drawn from the cryptographic RNG, with a new Random object created for every value

- Category C, severity low, confidence confirmed
- Features: EDU-10
- Document: `huawei_industry_tree/04_education/docs/10_video_course_bullet_comments.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_course_bullet_comments-0000002378246529
- Why: These values choose which of twelve rows a decorative comment lands in and how long to wait before the next one; nothing about them is security relevant. createRandom allocates a new crypto context on every call, several times per second for the whole of playback. Drawing a single byte and taking it modulo 12 also biases the low slots, since 256 is not a multiple of 12.
- Fix: Use Math.random() for these, or hold one cryptoFramework.Random instance as a field if a CSPRNG is wanted anyway.

### `HW-04-0084` - The module declares a router map whose profile is an empty array, and names the file differently from every other sample in the industry

- Category E, severity low, confidence confirmed
- Features: EDU-11
- Document: `huawei_industry_tree/04_education/docs/11_oral_english.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/oral_english-0000002349625326
- Why: An empty router map is dead configuration that is nevertheless packaged and parsed, and the file name differs from the industry's convention by one character - which is exactly the kind of difference that makes a developer copying between these samples produce a routerMap pointing at a profile that does not exist. The document's project tree lists neither file, so nothing signals which name is intended.
- Fix: Remove the routerMap entry and router_map.json; if a route map is added later, name it route_map.json to match the rest of the industry.

### `HW-04-0091` - Telephony Kit is imported and a signal-type field declared, neither of which is ever used

- Category B, severity low, confidence confirmed
- Features: EDU-12
- Document: `huawei_industry_tree/04_education/docs/12_network_monitor.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/network_monitor-0000002355490078
- Why: The import is the only reason this application depends on Telephony Kit at all, and a dependency that exists solely for an unused type annotation is dead weight in a sample meant to be copied. It also hints at an intended feature - showing the cellular network generation next to the speed - that is not implemented, so a reader cannot tell whether it was dropped or forgotten.
- Fix: Delete the import and the field.

### `HW-04-0097` - The project tree names the constants file Constants.ets but it ships as Contants.ets

- Category E, severity low, confidence confirmed
- Features: EDU-13
- Document: `huawei_industry_tree/04_education/docs/13_class_add_schedule.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/class_add_schedule-0000002358628536
- Why: The documented tree does not match the shipped project, and the misspelling has propagated into four import specifiers. A developer who creates the file under the documented name, or renames it to match, breaks all four imports; a developer who trusts the zip carries a typo into their own project's public structure.
- Fix: Rename the file to Constants.ets and update the four import statements.

### `HW-04-0104` - Read chapters are recorded by title text, so two chapters with the same name count as one

- Category B, severity low, confidence confirmed
- Features: EDU-14
- Document: `huawei_industry_tree/04_education/docs/14_course_progress_display.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/course_progress_display-0000002358926170
- Why: Course outlines repeat titles - 练习, 小结 and 复习 recur once per unit in most syllabi. Two chapters sharing a title are one entry in readChapterArray, so reading either ticks both in the contents list and the percentage stops at less than 100 even when every chapter has been read. Using the display string as the identity also couples progress tracking to localisation.
- Fix: Store chapter indices in readChapterArray and compare on index.

### `HW-04-0105` - The project tree names the page directory page while the zip ships pages

- Category E, severity low, confidence confirmed
- Features: EDU-14
- Document: `huawei_industry_tree/04_education/docs/14_course_progress_display.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/course_progress_display-0000002358926170
- Why: The documented tree does not match the shipped project, and the directory name is not cosmetic here: main_pages.json and the route map both address the pages by path, so a project laid out from the document has an entry point that does not resolve. Every other sample in this industry uses pages.
- Fix: Correct the tree entry to pages.

### `HW-04-0106` - An unused @State field named index11 is declared on the reading page

- Category C, severity low, confidence confirmed
- Features: EDU-14
- Document: `huawei_industry_tree/04_education/docs/14_course_progress_display.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/course_progress_display-0000002358926170
- Why: An @State field is registered with the state-management framework and participates in dependency tracking, so an unused one is not free. The placeholder name also signals leftover scaffolding in a sample published as a reference implementation - a reader has to determine that it is dead before they can safely delete it.
- Fix: Delete the declaration.

### `HW-04-0113` - The department handler clears the graduation-year selection twice on the same path

- Category B, severity low, confidence confirmed
- Features: EDU-15
- Document: `huawei_industry_tree/04_education/docs/15_cascading_selection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/cascading_selection-0000002364082736
- Why: The inner assignment is redundant, and because graduationYearSelectedId is a @Link the duplicate write propagates to the parent twice for one tap. More importantly it obscures the actual rule the cascade depends on - selecting or deselecting a department always resets the year - by making it look conditional in one branch and unconditional overall.
- Fix: Delete line 118.

### `HW-04-0121` - The project tree names the permission utility PermissionsView.ets while the zip ships PermissionsRequest.ets, and an unlisted patch.json is packaged

- Category E, severity low, confidence confirmed
- Features: EDU-16
- Document: `huawei_industry_tree/04_education/docs/16_equipment_detection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/equipment_detection-0000002398440861
- Why: The documented tree does not describe the shipped project. The name is not arbitrary either - the file is a request helper, not a view, and the document's own comment 权限验证工具类 (permission verification utility) matches the real name better than the one it prints. patch.json is a quick-fix artefact that has no place in a published sample and is invisible in the documented structure.
- Fix: Correct the tree entry and delete entry/patch.json from the distributed zip.

### `HW-04-0130` - Every card in the Swiper declares defaultFocus on its TextInput

- Category C, severity low, confidence confirmed
- Features: EDU-17
- Document: `huawei_industry_tree/04_education/docs/17_word_spelling.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/word_spelling-0000002368882740
- Why: defaultFocus designates the component that takes focus when the page opens; declaring it on every card in the list means the whole word list competes for it, and which one wins is not something the component can control. Combined with the eager ForEach construction, the page opens with N off-screen text inputs all claiming default focus.
- Fix: Pass the card its index and set defaultFocus(index === currentWordIndex), or use a FocusController to focus the active card after onAnimationEnd.

### `HW-04-0137` - Value labels are enabled three times and then defeated by setting their text size to zero

- Category C, severity low, confidence confirmed
- Features: EDU-18
- Document: `huawei_industry_tree/04_education/docs/18_track_study_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/track_study_time-0000002391113388
- Why: Three statements decide one thing and the last one wins by making the text invisible rather than by turning it off, so the library still lays out and renders a label per point at size zero. A reader cannot tell whether the values are meant to be shown - the code says yes twice and then hides them - and turning them back on means finding and removing a setValueTextSize call rather than flipping the flag that names the behaviour.
- Fix: Call setDrawValues(false) on both data sets and delete the setValueTextSize calls and the loop in aboutToAppear.

### `HW-04-0138` - The chart model is held in @State although it is only ever mutated in place and redrawn by invalidate

- Category C, severity low, confidence confirmed
- Features: EDU-18
- Document: `huawei_industry_tree/04_education/docs/18_track_study_time.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/track_study_time-0000002391113388
- Why: The reference never changes after construction, so @State observes an object that is only mutated internally and can never report a change; the redraw comes from the library's own invalidate(), not from state management. Meanwhile the framework wraps a large third-party object in an observation proxy for no benefit. The same file already treats the axis objects as plain fields, so the decorator here is inconsistent as well as inert.
- Fix: Declare model as a private field, matching leftAxis, rightAxis and xAxis.

### `HW-04-0146` - The project tree misspells the backup ability directory as entrybackupablility

- Category E, severity low, confidence confirmed
- Features: EDU-19
- Document: `huawei_industry_tree/04_education/docs/19_globe_label_animation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/globe_label_animation-0000002529698917
- Why: The documented tree does not match the shipped project, and the directory name is referenced from module.json5's srcEntry, so a project laid out from the document has an extension ability that does not resolve. This is the third distinct way this industry's documents get this one directory wrong - 04 and 06 list it under the name entryability, and this one misspells it.
- Fix: Correct the tree entry to entrybackupability.

### `HW-04-0154` - The industry FAQ chapter is a single redirect line that is byte-identical across all nineteen industries and points at the generic phone FAQ

- Category F, severity low, confidence confirmed
- Features: EDU-21
- Document: `huawei_industry_tree/04_education/docs/21_practice-educate-app-architecture-v1_2.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-educate-app-architecture-v1_2-0000002298147617
- Why: The chapter is titled 行业常见问题 - questions specific to this industry - and every industry's copy sends the reader to the same general phone FAQ, which is organised by device and capability rather than by industry. So the promise in the title is not kept anywhere: there is no education-specific content at the destination, and no way to tell from the link which of the FAQ's entries were meant. The page also has none of the sections its siblings have - no 场景介绍, no 约束与限制, no 工程目录 - so it contributes nothing to the industry's documentation set beyond an outbound link.
- Fix: Point the link at an industry-filtered FAQ view, or fold the chapter into the architecture document and remove the placeholder page from all nineteen industries.

