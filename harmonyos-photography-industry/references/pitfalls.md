# Pitfalls

> Generated from `features/*.md`. Source industry: `18_photography`, 32 features.
> Do not edit by hand; regenerate it in the review repository.

Every entry is a confirmed defect in the published HQ documentation or in its sample project. A naive copy of the document reproduces it.

## Systemic - repeated across features (21)

These recur in more than one feature of this industry. Fix them once in your own project template.

### `HW-18-0007` - Runtime permission request includes undeclared MEDIA_LOCATION, so the whole grant fails and the recorder cannot start (same defect in PipwindowRecorder)

- Category D, severity high, confidence confirmed
- Features: PHOTO-25, PHOTO-30
- Document: `huawei_industry_tree/18_photography/docs/25_avrecorder_timer.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/avrecorder_timer-0000002529561491
- Why: requestPermissionsFromUser rejects the entire call when any requested permission is undeclared in module.json5, so neither sample ever obtains CAMERA/MICROPHONE — the recording feature both docs demonstrate cannot start; MEDIA_LOCATION is also a restricted permission that this use case does not need.
- Fix: Remove MEDIA_LOCATION from the runtime request array in both samples (or declare it if EXIF location is truly required).

### `HW-18-0029` - Systematic: the AVRecorder target fd is closed in a finally right after prepare() — before recording starts — in three recorder samples (plus double close on stop)

- Category B, severity high, confidence confirmed
- Features: PHOTO-06, PHOTO-25, PHOTO-30
- Document: `huawei_industry_tree/18_photography/docs/06_video_recording.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_recording-0000002258016882
- Why: The recorder writes to an fd the app already closed (or, if the number was reused, to an unrelated file): recordings come out empty/corrupt on the main record path unless the native layer happens to dup the fd — the official AVRecorder guide closes the file only after stop().
- Fix: Keep the file open until avRecorder.stop() completes, close once, and guard the finally for unassigned files.

### `HW-18-0003` - getAssets is called with READ_IMAGEVIDEO declared but never requested at runtime, so the picker-URI query fails (same in VideoWatermark)

- Category B, severity medium, confidence confirmed
- Features: PHOTO-09, PHOTO-21
- Document: `huawei_industry_tree/18_photography/docs/09_compress_images.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/compress_images-0000002322173825
- Why: A declared user-grant permission still needs a runtime request before the API succeeds; without it getAssets returns a permission error and the compress/watermark pipeline that depends on the fetched asset breaks. The permission-free alternative (reading the picker URI directly via file APIs) is what these flows should use.
- Fix: Replace the getAssets lookup with fs/image APIs over the returned URI in both samples.

### `HW-18-0004` - Systematic: photography samples declare restricted READ/WRITE_IMAGEVIDEO although their code deliberately uses permission-free SaveButton and PhotoViewPicker flows

- Category D, severity medium, confidence confirmed
- Features: PHOTO-04, PHOTO-05, PHOTO-09, PHOTO-10, PHOTO-18, PHOTO-20, PHOTO-21, PHOTO-23, PHOTO-24, PHOTO-29
- Document: `huawei_industry_tree/18_photography/docs/10_picture_sticker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/picture_sticker-0000002293373244
- Why: READ/WRITE_IMAGEVIDEO are restricted (ACL) permissions ordinary apps cannot ship with; declaring them in templates that were explicitly built around the permission-free security-control flow contradicts the design being taught and hands every copying developer an app-review rejection. One shared module.json5 template is the evident root cause.
- Fix: Delete the READ/WRITE_IMAGEVIDEO requestPermissions entries from all nine module.json5 files.

### `HW-18-0010` - Systematic: fire-and-forget camera teardown races the rebuild in eight photography samples

- Category B, severity medium, confidence confirmed
- Features: PHOTO-01, PHOTO-04, PHOTO-07, PHOTO-13, PHOTO-25, PHOTO-26, PHOTO-27, PHOTO-29, PHOTO-30, PHOTO-31
- Document: `huawei_industry_tree/18_photography/docs/01_practice-photo-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1-0000002077325929
- Why: Camera switch / re-entry / resolution toggle rebuilds the session while the old input is still closing: intermittent camera-occupied errors, black preview, and unhandled rejections from the fire-and-forget release promises.
- Fix: Make releaseCamera await each call, null the fields, and have callers await it before re-init.

### `HW-18-0021` - Systematic: hardcoded camera profiles — find() can return undefined which is passed straight into createPreviewOutput/createPhotoOutput (4 samples)

- Category B, severity medium, confidence probable
- Features: PHOTO-01, PHOTO-13, PHOTO-31
- Document: `huawei_industry_tree/18_photography/docs/01_practice-photo-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1-0000002077325929
- Why: On any device that does not expose exactly the hardcoded profile, createPreviewOutput(undefined, surfaceId) throws 7400101 inside an async function no caller catches — black preview instead of a fallback.
- Fix: Guard the find() results before use and fall back to the first supported profile.

### `HW-18-0030` - Systematic: requestPermissionsFromUser result ignored — the camera/recorder pipeline is built even when the user denies (6 samples)

- Category B, severity medium, confidence confirmed
- Features: PHOTO-01, PHOTO-06, PHOTO-07, PHOTO-25, PHOTO-26, PHOTO-30, PHOTO-31
- Document: `huawei_industry_tree/18_photography/docs/06_video_recording.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_recording-0000002258016882
- Why: Deny the dialog: the code still opens the camera / enables the feature, producing raw permission errors (or recording without the gate) instead of a refused-state UI; requests' rejections are also unhandled.
- Fix: Check data.authResults before building the pipeline; add .catch.

### `HW-18-0041` - Systematic: camera init races the XComponent surface — init fired with empty surfaceId from permission callbacks/fixed timers (4 samples)

- Category B, severity medium, confidence probable
- Features: PHOTO-13, PHOTO-29, PHOTO-30, PHOTO-31
- Document: `huawei_industry_tree/18_photography/docs/13_capture_timer.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/capture_timer-0000002352218316
- Why: First run: a doomed empty-surface init opens the camera and fails on createPreviewOutput, racing the second (onLoad) init whose un-awaited teardown can strand the first open — intermittent black preview at startup.
- Fix: Gate camera init on surfaceId !== '' and drive it from one place (onLoad after grant).

### `HW-18-0073` - Local `let previewOutput` shadows the module variable, making stopRecordPreview's release dead code (2 samples)

- Category B, severity medium, confidence confirmed
- Features: PHOTO-25, PHOTO-30
- Document: `huawei_industry_tree/18_photography/docs/25_avrecorder_timer.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/avrecorder_timer-0000002529561491
- Why: The preview output stream is never released across record/stop/switch cycles — leaked outputs accumulate and later session configs can fail.
- Fix: Drop the `let` so the module variable is assigned.

### `HW-18-0090` - 2 sample projects swallow errors in catch blocks with an empty body

- Category B, severity medium, confidence confirmed
- Features: PHOTO-06, PHOTO-29
- Document: `huawei_industry_tree/18_photography/docs/29_camera_twist.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/camera_twist-0000002552826219
- Why: A catch block with an empty body discards the error object and lets execution continue as if the operation had succeeded. The failure becomes invisible: no log, no user feedback, and no way to diagnose it from a released build. In several of these cases the guarded call is the feature the sample exists to demonstrate.
- Fix: Log the error with hilog including the BusinessError code and message, and surface a user-visible result where the operation was user initiated. Never leave the body empty.

### `HW-18-0001` - Systematic: five photography project trees do not match their zips (2 missing files, 1 renamed, 2 wrong extensions)

- Category E, severity low, confidence confirmed
- Features: PHOTO-03, PHOTO-21
- Document: `huawei_industry_tree/18_photography/docs/03_image_filter.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_filter-0000002284048625
- Why: Readers navigating by the documented structure hit files that are missing or named differently; the trees were not regenerated after the samples changed.
- Fix: Regenerate the five 工程目录 blocks from the current zips.

### `HW-18-0005` - Systematic: ImageSource instances are created but never released across eight photography sample files

- Category B, severity low, confidence confirmed
- Features: PHOTO-01, PHOTO-06, PHOTO-08, PHOTO-10, PHOTO-12, PHOTO-14
- Document: `huawei_industry_tree/18_photography/docs/14_image_converter.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_converter-0000002368877552
- Why: ImageSource holds native decoding resources; repeatedly creating instances (every conversion/sticker/filter operation) without release accumulates native memory — the official image-decoding guide's own examples release the source after createPixelMap.
- Fix: Call imageSource.release() (finally-block) after createPixelMap in the listed utilities.

### `HW-18-0008` - Two photography docs are published under wrong-topic URL slugs ('insurance', 'audio') from unrelated pipelines

- Category E, severity low, confidence confirmed
- Features: PHOTO-26, PHOTO-31
- Document: `huawei_industry_tree/18_photography/docs/26_insurance-v1_2-ts_32.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/insurance-v1_2-ts_32-0000002312678754
- Why: The document IDs are user-visible and citable; wrong-domain slugs indicate the pages were cloned from other industries' page shells and never re-keyed, which also breaks any slug-based navigation or search assumptions.
- Fix: Re-publish the two pages under topic-correct slugs with redirects from the old IDs.

### `HW-18-0022` - Systematic: files opened with fileIo.openSync are never closed in image/video utilities (5 samples)

- Category B, severity low, confidence confirmed
- Features: PHOTO-01, PHOTO-11, PHOTO-15, PHOTO-17, PHOTO-24
- Document: `huawei_industry_tree/18_photography/docs/01_practice-photo-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1-0000002077325929
- Why: Every image load / page visit leaks one fd for the process lifetime; heavy use exhausts the descriptor table and later opens fail.
- Fix: Close the files after reading / in aboutToDisappear.

### `HW-18-0023` - Systematic: pinch-zoom clamps compare against a stale or never-assigned zoomRatioRange (3 samples)

- Category B, severity low, confidence confirmed
- Features: PHOTO-01, PHOTO-30, PHOTO-31
- Document: `huawei_industry_tree/18_photography/docs/01_practice-photo-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1-0000002077325929
- Why: Front camera clamped to the rear camera's 10x range, or unclamped values passed to setZoomRatio which throws (swallowed) — pinch zoom silently stops responding at range edges.
- Fix: Assign the returned zoomRatioRange on every cameraShooting call.

### `HW-18-0024` - Systematic: photo-picker cancel path unhandled — openSync(undefined)/throw with no catch in six samples

- Category B, severity low, confidence confirmed
- Features: PHOTO-03, PHOTO-10, PHOTO-12, PHOTO-14, PHOTO-18, PHOTO-20, PHOTO-22, PHOTO-23
- Document: `huawei_industry_tree/18_photography/docs/03_image_filter.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_filter-0000002284048625
- Why: Cancelling the picker is a routine action; each cancel raises an unhandled promise rejection (and in GIFGenerator litters the cache with junk files) instead of quietly returning. ImageRotateAndFlip contains the correct empty-array guard, showing the intended pattern.
- Fix: Guard on photoUris.length === 0 before use and add .catch to the call chains.

### `HW-18-0025` - Systematic: fd closed immediately after createImageSource, before the async createPixelMap decode (3 samples)

- Category B, severity low, confidence probable
- Features: PHOTO-03, PHOTO-10, PHOTO-18
- Document: `huawei_industry_tree/18_photography/docs/03_image_filter.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_filter-0000002284048625
- Why: createImageSource(fd) requires the fd to stay valid until decoding completes; closing first can fail the decode on the startup path, leaving the main image blank.
- Fix: Move closeSync after await createPixelMap (finally).

### `HW-18-0027` - Systematic: capture() hardcodes rotation ROTATION_0 (3 samples); RatioCamera even computes the sensor rotation and then ignores it

- Category B, severity low, confidence confirmed
- Features: PHOTO-04, PHOTO-07, PHOTO-31
- Document: `huawei_industry_tree/18_photography/docs/04_ratio_camera.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/ratio_camera-0000002252528422
- Why: Photos taken with the device rotated are saved with wrong orientation and display sideways in the gallery.
- Fix: Feed the computed rotation into PhotoCaptureSetting.

### `HW-18-0036` - Systematic: ImagePacker created per save and never released in eight photography samples

- Category B, severity low, confidence confirmed
- Features: PHOTO-08, PHOTO-09, PHOTO-10, PHOTO-12, PHOTO-16, PHOTO-18, PHOTO-23, PHOTO-24, PHOTO-28
- Document: `huawei_industry_tree/18_photography/docs/08_image_segmentation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_segmentation-0000002282720108
- Why: Every save leaks a native packer instance; repeated saves accumulate native memory for the process lifetime.
- Fix: imagePacker.release() in finally.

### `HW-18-0091` - 5 sample projects depend on third-party packages through unpinned version ranges

- Category B, severity low, confidence confirmed
- Features: PHOTO-15, PHOTO-17, PHOTO-19, PHOTO-21, PHOTO-22
- Document: `huawei_industry_tree/18_photography/docs/15_video_clip.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_clip-0000002405051977
- Why: A caret or tilde range lets ohpm resolve a different version than the one the sample was verified against, so the published sample is not reproducible and can break without any change to its own source. Sample projects are reference implementations; their dependency set should be exactly the one that was tested.
- Fix: Pin exact versions in oh-package.json5 dependencies, and record the resolved set in oh-package-lock.json5.

### `HW-18-0092` - Systematic: 29 sample projects in this industry ship with release obfuscation explicitly disabled

- Category D, severity low, confidence confirmed
- Features: PHOTO-01, PHOTO-03, PHOTO-05, PHOTO-06, PHOTO-07, PHOTO-08, PHOTO-09, PHOTO-10, PHOTO-11, PHOTO-12, PHOTO-13, PHOTO-14, PHOTO-15, PHOTO-16, PHOTO-17, PHOTO-18, PHOTO-19, PHOTO-20, PHOTO-21, PHOTO-22, PHOTO-23, PHOTO-24, PHOTO-25, PHOTO-27, PHOTO-28, PHOTO-29, PHOTO-30, PHOTO-31
- Document: `huawei_industry_tree/18_photography/docs/25_avrecorder_timer.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/avrecorder_timer-0000002529561491
- Why: These samples are published as templates and are copied wholesale into real products. A release buildOptionSet that sets obfuscation.ruleOptions.enable to false, while still shipping an obfuscation-rules.txt, reads as a deliberate configuration rather than an omission, so a developer copying the module has no signal that release builds are unprotected. ArkTS source names and structure remain readable in the released HAP.
- Fix: Set arkOptions.obfuscation.ruleOptions.enable to true in the release entry of buildOptionSet for every module, and keep the existing obfuscation-rules.txt. HARs should also declare consumerFiles so their rules reach consumers.

## Per feature (74)

### `HW-18-0009` - CAMERA permission requested fire-and-forget while onShown starts the camera immediately; no retry after grant

- Category B, severity high, confidence probable
- Features: PHOTO-01
- Document: `huawei_industry_tree/18_photography/docs/01_practice-photo-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1-0000002077325929
- Why: First run: the grant dialog is still open (or denied) when onShown opens the camera; cameraInput.open() fails without ohos.permission.CAMERA as an unhandled rejection and the preview stays black; the camera never retries until the page is left and re-entered.
- Fix: Await the permission promise, check authResults, then start cameraShooting; re-run on grant.

### `HW-18-0050` - Median denoise filter swaps the red and blue channels in its output

- Category B, severity high, confidence confirmed
- Features: PHOTO-18
- Document: `huawei_industry_tree/18_photography/docs/18_image_denoising.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_denoising-0000002419864905
- Why: Every denoised image comes out with red and blue exchanged on the sample's main path.
- Fix: targetIdx=R gets neighborsR, targetIdx+2 gets neighborsB.

### `HW-18-0076` - Module-level `let context: Context;` is never assigned in any of the three pages — getCameraManager(undefined) throws and the preview never starts

- Category B, severity high, confidence confirmed
- Features: PHOTO-27
- Document: `huawei_industry_tree/18_photography/docs/27_photo-v1_2-ts_12.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/photo-v1_2-ts_12-0000002298869489
- Why: Every camera entry point in all three demo scenarios passes undefined as Context — the sample's core feature (preview) never works.
- Fix: Assign context = this.getUIContext().getHostContext() before use (or pass this.context).

### `HW-18-0011` - Stitch drag-swap exchanges only the PixelMaps; leftInfo/rightInfo keep pre-swap dimensions used by the crop

- Category B, severity medium, confidence confirmed
- Features: PHOTO-01
- Document: `huawei_industry_tree/18_photography/docs/01_practice-photo-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1-0000002077325929
- Why: Two images of different heights, swap, save: the taller-image branch crops the wrong bitmap with the other image's width/height — out-of-bounds or wrong crop, wrong stitched output.
- Fix: Swap leftInfo/rightInfo together with the pixel maps (or re-getImageInfo before cropping).

### `HW-18-0012` - Crop pinch multiplies the canvas transform by the cumulative gesture scale every frame and never resets it

- Category B, severity medium, confidence probable
- Features: PHOTO-01
- Document: `huawei_industry_tree/18_photography/docs/01_practice-photo-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1-0000002077325929
- Why: A steady 1.1x pinch becomes 1.1^n zoom within a single gesture and the polluted matrix persists into all later drawImage calls, mis-scaling the crop preview.
- Fix: Use setTransform(scale,0,0,scale,0,0) or resetTransform() + single transform.

### `HW-18-0013` - Rotate mutates the PixelMap in place but width/height state is never swapped, corrupting later crop math

- Category B, severity medium, confidence probable
- Features: PHOTO-01
- Document: `huawei_industry_tree/18_photography/docs/01_practice-photo-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1-0000002077325929
- Why: Rotate a portrait image 90°, then crop: the crop region is computed from transposed (stale) dimensions and can exceed the rotated bitmap — cropImage fails or crops the wrong area; centering is also off.
- Fix: Swap the width/height fields (and *_Trans) after each 90°/270° rotation.

### `HW-18-0014` - Doc claims the camera works '无需申请相机权限' while the sample declares and runtime-requests ohos.permission.CAMERA

- Category E, severity medium, confidence confirmed
- Features: PHOTO-01
- Document: `huawei_industry_tree/18_photography/docs/01_practice-photo-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1-0000002077325929
- Why: A developer trusting the doc's claim and omitting the declaration/request gets error 12100001 / camera open failure; the stated constraint contradicts the shipped code.
- Fix: Reword to state Camera Kit requires ohos.permission.CAMERA (the no-permission claim applies only to cameraPicker).

### `HW-18-0037` - Default state mismatch: 'high quality' mode is pre-selected but zipQuality starts at 0, so the default compression runs at worst quality

- Category B, severity medium, confidence confirmed
- Features: PHOTO-09
- Document: `huawei_industry_tree/18_photography/docs/09_compress_images.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/compress_images-0000002322173825
- Why: Open app → pick photos → start zip with the pre-highlighted mode: packToFile runs with quality 0 (maximum compression), contradicting the selected 'high' UI state.
- Fix: Initialize zipQuality to the high-mode value (40).

### `HW-18-0040` - Un-awaited fs.rmdir/fs.mkdir race on cacheDir can leave the cache directory deleted, breaking GIF generation

- Category B, severity medium, confidence confirmed
- Features: PHOTO-12
- Document: `huawei_industry_tree/18_photography/docs/12_gif_generator.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/gif_generator-0000002330170016
- Why: With the race resolved the wrong way the app has no cacheDir, so GifGenerator's fs.openSync(cacheDir + '/result_*.gif') throws ENOENT and the feature fails.
- Fix: await fs.rmdir then await fs.mkdir (or rmdirSync/mkdirSync), with error handling.

### `HW-18-0043` - Clip failure leaves isClipping true — the full-screen loading overlay locks the UI permanently

- Category B, severity medium, confidence confirmed
- Features: PHOTO-15
- Document: `huawei_industry_tree/18_photography/docs/15_video_clip.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_clip-0000002405051977
- Why: Any ffmpeg/copy failure leaves the app stuck behind a modal loading overlay with no recovery except killing the app, plus an unhandled rejection.
- Fix: try/finally around the await; reset isClipping in finally, toast on error.

### `HW-18-0047` - deleteFile uses fileIo.rmdir on a regular file, so the old output survives; without ffmpeg -y the second export fails and the button stays disabled

- Category B, severity medium, confidence confirmed
- Features: PHOTO-17
- Document: `huawei_industry_tree/18_photography/docs/17_video_cropping.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_cropping-0000002385796378
- Why: 'Do it twice': the stale output blocks ffmpeg, the crop callback never fires, saveToFile never runs, and the Export button stays permanently disabled.
- Fix: Use fileIo.unlink for files (or add -y), restore the flag on failure.

### `HW-18-0048` - First-run race: XComponent.onLoad plays the sandbox video before the async rawfile copy finishes, and the AVPlayer error path releases the player for good

- Category B, severity medium, confidence probable
- Features: PHOTO-17
- Document: `huawei_industry_tree/18_photography/docs/17_video_cropping.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_cropping-0000002385796378
- Why: On the very first launch the player opens an empty/partial file, errors, and self-releases — the video never plays until the app is restarted.
- Fix: Await the copy (or gate onLoad playback on the copy promise).

### `HW-18-0052` - PiP thumbnail extraction loops over the BASIC video's duration, not the PiP video's

- Category B, severity medium, confidence confirmed
- Features: PHOTO-19
- Document: `huawei_industry_tree/18_photography/docs/19_video_edit_pip_window.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_edit_pip_window-0000002395437788
- Why: PiP shorter than main: frames queried past its end (failed thumbnails, loop aborts via catch); PiP longer: strip truncated — the PiP thumbnail row never reflects the PiP file's own length.
- Fix: Fetch the PiP file's duration from its own metadata before looping.

### `HW-18-0053` - A failed merge never resets outputPath, so Export permanently skips merging and re-saves a nonexistent file

- Category B, severity medium, confidence confirmed
- Features: PHOTO-19
- Document: `huawei_industry_tree/18_photography/docs/19_video_edit_pip_window.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_edit_pip_window-0000002395437788
- Why: After one failed merge every Export tap tries to save a file ffmpeg never produced — unhandled rejection, export dead for the session.
- Fix: Reset outputPath in the ffmpeg failure callback; catch the save.

### `HW-18-0054` - AVPlayer instances and their fds are never released; teardown even unlinks the files the orphaned players still hold

- Category B, severity medium, confidence confirmed
- Features: PHOTO-19
- Document: `huawei_industry_tree/18_photography/docs/19_video_edit_pip_window.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_edit_pip_window-0000002395437788
- Why: Each edit session leaks a player + fd; exit mid-merge can throw in teardown and corrupt the next session's state.
- Fix: release() both players, close fds, guard unlinkSync with access().

### `HW-18-0056` - rotate/flip mutate the PixelMap in place and reassign the same reference — @State sees no change and the preview does not refresh

- Category B, severity medium, confidence probable
- Features: PHOTO-20
- Document: `huawei_industry_tree/18_photography/docs/20_image_rotate_and_flip.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_rotate_and_flip-0000002396643788
- Why: Same-reference assignment is a no-op for state observation and ArkUI does not watch in-place PixelMap content — the preview does not reliably re-render after rotate/flip.
- Fix: Use clonePixelMap (already present) before/after the sync transform.

### `HW-18-0060` - ffmpeg result code ignored: failure still copies the output to the gallery and toasts success (and Loading.close lives only on the success path)

- Category B, severity medium, confidence confirmed
- Features: PHOTO-21
- Document: `huawei_industry_tree/18_photography/docs/21_video_water_mark.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_water_mark-0000002408013854
- Why: Any ffmpeg failure saves a corrupt/empty 'watermarked' video with a success message; a copy failure additionally leaves the loading mask up forever.
- Fix: Accept the code param, branch on it, close Loading in finally.

### `HW-18-0061` - addWaterMark called without await inside try/catch — rejections escape and the non-cancelable loading mask never closes

- Category B, severity medium, confidence confirmed
- Features: PHOTO-21
- Document: `huawei_industry_tree/18_photography/docs/21_video_water_mark.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_water_mark-0000002408013854
- Why: Any pre-ffmpeg failure = full-screen spinner forever; the error alert never shows.
- Fix: await the call (make onClick async).

### `HW-18-0070` - CropView measures its own root as the container, so the max-width clamp is 85% of the current frame — the crop can only shrink, never widen

- Category B, severity medium, confidence confirmed
- Features: PHOTO-24
- Document: `huawei_industry_tree/18_photography/docs/24_image_cropping.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_cropping-0000002426210646
- Why: First outward drag of an edge snaps the frame ~15% narrower; repeated drags shrink it progressively — documented drag-to-resize is broken in one direction.
- Fix: Measure the parent image container id instead of the view's own root.

### `HW-18-0074` - Every preview rebuild creates a new empty gallery asset, and getThumbnail picks the newest asset — preview points at the empty placeholder

- Category B, severity medium, confidence probable
- Features: PHOTO-25
- Document: `huawei_industry_tree/18_photography/docs/25_avrecorder_timer.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/avrecorder_timer-0000002529561491
- Why: The shown thumbnail/preview is the freshly created empty placeholder, not the recorded video; every rebuild (foreground, camera switch, fold) also litters the gallery with empty video files.
- Fix: Create the asset in startRecord; pass the recorded uri to getThumbnail.

### `HW-18-0075` - Fold-status change rebuilds the whole pipeline without stopping the running session

- Category B, severity medium, confidence probable
- Features: PHOTO-25
- Document: `huawei_industry_tree/18_photography/docs/25_avrecorder_timer.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/avrecorder_timer-0000002529561491
- Why: Folding/unfolding while previewing creates a second session while the first still holds the camera — conflict errors or leaked sessions on foldables.
- Fix: await stopRecordPreview() first.

### `HW-18-0077` - commonRequestPermissions un-awaited; checkPermissions races the dialog — first tap always bails

- Category B, severity medium, confidence confirmed
- Features: PHOTO-27
- Document: `huawei_industry_tree/18_photography/docs/27_photo-v1_2-ts_12.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/photo-v1_2-ts_12-0000002298869489
- Why: First run: user taps a scenario, grants the dialog, nothing happens; only a second tap works.
- Fix: await the request (or branch on its authResults).

### `HW-18-0078` - emitter.on subscriptions never removed — dead pages keep creating camera sessions on every app foreground

- Category B, severity medium, confidence confirmed
- Features: PHOTO-27
- Document: `huawei_industry_tree/18_photography/docs/27_photo-v1_2-ts_12.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/photo-v1_2-ts_12-0000002298869489
- Why: After leaving the page every foreground event still runs cameraCreate from the dead page (stale surfaceId); N visits → N conflicting session builds per event.
- Fix: emitter.off(eventId) with the stored callbacks.

### `HW-18-0079` - Leaving EmitterPage never releases the camera — the only teardown is the app-background emitter event

- Category B, severity medium, confidence confirmed
- Features: PHOTO-27
- Document: `huawei_industry_tree/18_photography/docs/27_photo-v1_2-ts_12.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/photo-v1_2-ts_12-0000002298869489
- Why: Navigating back leaves the camera session running/leaked while an unrelated page shows.
- Fix: Call releaseCamera in onHidden/aboutToDisappear.

### `HW-18-0080` - Oil-painting filter crashes on any pure-white pixel: bucket index reaches count (off-by-one)

- Category B, severity medium, confidence confirmed
- Features: PHOTO-28
- Document: `huawei_industry_tree/18_photography/docs/28_image_filter_processing.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_filter_processing-0000002520461010
- Why: Images with highlights/white background (extremely common) crash the taskpool task — combined with the missing .catch the app hangs behind the loading overlay.
- Fix: Math.min(count-1, floor(gray/255*count)).

### `HW-18-0081` - Oil-painting filter writes red and blue swapped (little-endian channel confusion)

- Category B, severity medium, confidence confirmed
- Features: PHOTO-28
- Document: `huawei_industry_tree/18_photography/docs/28_image_filter_processing.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_filter_processing-0000002520461010
- Why: Filter output visibly wrong on every use: oranges turn blue, sky turns orange.
- Fix: Write R to byte 0 and B to byte 2 (swap indexes 0/2).

### `HW-18-0082` - taskpool.execute has no .catch — a filter failure leaves the loading overlay blocking the UI forever

- Category B, severity medium, confidence confirmed
- Features: PHOTO-28
- Document: `huawei_industry_tree/18_photography/docs/28_image_filter_processing.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_filter_processing-0000002520461010
- Why: Filter task failure = unhandled rejection + permanent full-screen LoadingProgress; restart required.
- Fix: .catch(() => { isProcessEnd = true; toast })

### `HW-18-0084` - Every shutter press registers another permanent GRAVITY sensor listener; sensor.off never called anywhere (3 subscription sites)

- Category B, severity medium, confidence confirmed
- Features: PHOTO-29
- Document: `huawei_industry_tree/18_photography/docs/29_camera_twist.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/camera_twist-0000002552826219
- Why: Shutter spam / component rebuilds stack full-rate sensor subscriptions forever — battery drain, N callbacks per event, callbacks mutating destroyed components.
- Fix: Subscribe once; sensor.off in aboutToDisappear/release.

### `HW-18-0085` - First capture reads this.rotation before any sensor callback has fired — the headline rotate-with-device feature fails on the first shot

- Category B, severity medium, confidence confirmed
- Features: PHOTO-29
- Document: `huawei_industry_tree/18_photography/docs/29_camera_twist.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/camera_twist-0000002552826219
- Why: First photo in landscape saved with rotation 0; later captures use whatever earlier listeners left behind.
- Fix: Maintain rotation from one long-lived listener; read it directly in capture.

### `HW-18-0088` - Doc claims closing the PiP window stops recording and saves the video; the sample's onStateChange only logs

- Category D, severity medium, confidence confirmed
- Features: PHOTO-30
- Document: `huawei_industry_tree/18_photography/docs/30_pipwindow_recorder.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pipwindow_recorder-0000002522526388
- Why: Record → background (PiP) → close PiP: the recorder keeps running, the asset is never finalized, the promised save never happens; the recording is lost when the process dies.
- Fix: Call stopRecord() in the STOPPED branch.

### `HW-18-0089` - EntryAbility.onForeground unconditionally rebuilds the camera — first-run and dialog transitions trigger overlapping doomed inits

- Category B, severity medium, confidence probable
- Features: PHOTO-31
- Document: `huawei_industry_tree/18_photography/docs/31_audio-v1_2-ts_50.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio-v1_2-ts_50-0000002407536358
- Why: First launch: failed init + racing rebuilds → intermittent black preview; the permission dialog itself causes a foreground transition that re-triggers init.
- Fix: Guard fromBack on permission/surfaceId and serialize inits.

### `HW-18-0093` - The published detectFace snippet drops the try/catch the sample has, so as documented a detection failure skips release and leaves the detector initialised

- Category E, severity medium, confidence confirmed
- Features: PHOTO-11
- Document: `huawei_industry_tree/18_photography/docs/11_face_detection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/face_detection-0000002328775849
- Why: The abridgement removes the one thing that keeps the two-line init/release pair balanced. In the published form a rejection from detect propagates out of detectFace before release is reached, so the detector stays initialised for the life of the process and the next init call is made against an already-initialised engine. The real sample gets this right, so the document is strictly worse than the code it documents, and the reader has no way to know a guard was removed.
- Fix: Restore the try/catch in the snippet, or use try/finally with release in the finally, which is the shape a reader should copy.

### `HW-18-0094` - loadImage creates an ImageSource for every selected picture and never releases it

- Category B, severity medium, confidence confirmed
- Features: PHOTO-11
- Document: `huawei_industry_tree/18_photography/docs/11_face_detection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/face_detection-0000002328775849
- Why: image.createImageSource allocates a native decoder that the ArkTS garbage collector does not reclaim. loadImage runs once per picture the user picks, and picking another picture is the normal way to use this sample, so each selection leaks one decoder plus the descriptor. The local variable is even declared as image.ImageSource | undefined = undefined on the line before, which reads like the author intended a try/finally and then did not write one.
- Fix: Wrap the body in try/finally, call imageSource.release() and fileIo.closeSync(fileSource) in the finally, and return the pixel map from a variable assigned inside the try.

### `HW-18-0095` - The face thumbnail grid derives its ForEach key by JSON.stringify of the detection result, which is both expensive and collision-prone

- Category C, severity medium, confidence confirmed
- Features: PHOTO-11
- Document: `huawei_industry_tree/18_photography/docs/11_face_detection.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/face_detection-0000002328775849
- Why: The key generator runs for every item on every render pass, so each pass serialises every detection result, including its rectangle and probability fields, to build a string used only for identity. It is also unstable in the wrong direction: two faces that the detector reports with identical geometry serialise to the same string and collide, so the grid drops one of them, while any floating-point jitter in a re-detection produces a different key for the same face and forces a rebuild. Building a key by serialising the item is the specific anti-pattern HQ's own performance practice warns against for keyed iteration.
- Fix: Use the index, or a compact composite such as `${face.rect.left},${face.rect.top}`, as the key generator.

### `HW-18-0002` - Systematic: project trees list 'EntryBackAbility.ets' — a truncation of EntryBackupAbility.ets — across four docs

- Category E, severity low, confidence confirmed
- Features: PHOTO-08
- Document: `huawei_industry_tree/18_photography/docs/08_image_segmentation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_segmentation-0000002282720108
- Why: The recurring identical typo indicates the trees are hand-edited from a shared, wrong template rather than generated from the zips.
- Fix: Correct all four trees; drop the entry for ratio_camera.

### `HW-18-0006` - Progress-refresh interval started in aboutToAppear is never cleared

- Category B, severity low, confidence confirmed
- Features: PHOTO-19
- Document: `huawei_industry_tree/18_photography/docs/19_video_edit_pip_window.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_edit_pip_window-0000002395437788
- Why: Leaving the edit page keeps a periodic timer polling player.currentTime on a detached component for the process lifetime.
- Fix: Add aboutToDisappear(): clearInterval(this.proFreshTimer).

### `HW-18-0015` - Watermark font string '宋体 ${...}px}' is malformed (stray brace, family-before-size), so the computed size never applies — doc ships the same snippet

- Category B, severity low, confidence confirmed
- Features: PHOTO-01
- Document: `huawei_industry_tree/18_photography/docs/01_practice-photo-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1-0000002077325929
- Why: The watermark ignores imageScale and renders at the default size on every crop/edit render.
- Fix: Use `${100 / this.imageScale}px 宋体` (drop the stray '}').

### `HW-18-0016` - joinImages helper breaks its own contract: string-array overload always throws, 3+ images silently dropped, raw-buffer ImageSources are dead code

- Category B, severity low, confidence confirmed
- Features: PHOTO-01
- Document: `huawei_industry_tree/18_photography/docs/01_practice-photo-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1-0000002077325929
- Why: Passing URIs (the declared type) silently yields null; joinImages([a,b,c]) drops c without error; every stitch leaks two useless native ImageSources.
- Fix: Remove the string overload (or implement it), size the canvas from all inputs, drop the dead ImageSource creation.

### `HW-18-0017` - Falsy-zero pan guard: a purely horizontal or vertical drag frame is discarded and the image snaps to center

- Category B, severity low, confidence confirmed
- Features: PHOTO-01
- Document: `huawei_industry_tree/18_photography/docs/01_practice-photo-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1-0000002077325929
- Why: Pan straight down (offsetX===0): the drag is ignored for that frame; any axis crossing zero mid-gesture makes the image jump to the centered position.
- Fix: Use `if (offsetX !== undefined && offsetY !== undefined)`.

### `HW-18-0018` - Reset button force-casts rawPixelMap which is undefined until the async load completes — crash on early tap

- Category B, severity low, confidence confirmed
- Features: PHOTO-01
- Document: `huawei_industry_tree/18_photography/docs/01_practice-photo-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1-0000002077325929
- Why: Tapping 重置 immediately after entering the page (image still decoding) calls getImageInfoSync() on undefined and crashes.
- Fix: Guard on rawPixelMap before cloning.

### `HW-18-0019` - PRIVACY_WINDOW is a system_basic permission a normal app cannot hold; setWindowPrivacyMode rejects with no catch

- Category B, severity low, confidence probable
- Features: PHOTO-01
- Document: `huawei_industry_tree/18_photography/docs/01_practice-photo-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1-0000002077325929
- Why: In a normally signed build setWindowPrivacyMode rejects with 201 — unhandled rejection and the advertised anti-screenshot feature silently does nothing.
- Fix: Add a .catch and document the ACL/signing requirement.

### `HW-18-0020` - Doc promises a banner carousel on the home page; the sample renders a single static image and the banner data is dead code

- Category D, severity low, confidence confirmed
- Features: PHOTO-01
- Document: `huawei_industry_tree/18_photography/docs/01_practice-photo-app-architecture-v1.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1-0000002077325929
- Why: Readers looking for the described carousel implementation find dead data and a static image.
- Fix: Either implement the Swiper carousel or fix the doc's layout description.

### `HW-18-0026` - Permission-check loop returns inside the first iteration, so only the first permission is ever evaluated

- Category B, severity low, confidence confirmed
- Features: PHOTO-04
- Document: `huawei_industry_tree/18_photography/docs/04_ratio_camera.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/ratio_camera-0000002252528422
- Why: With CAMERA granted but a later permission denied the helper still reports success; the generic helper is broken for any multi-permission use.
- Fix: Return false inside the loop, true after it.

### `HW-18-0028` - Stencil utils: readPixelsToBuffer un-awaited and its raw RGBA buffer fed to createImageSource — the PixelMap input path can never decode

- Category B, severity low, confidence confirmed
- Features: PHOTO-05
- Document: `huawei_industry_tree/18_photography/docs/05_image_stitch.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_stitch-0000002287473193
- Why: Any caller passing PixelMaps gets a dead path: the ImageSource can never decode raw RGBA, so the branch silently produces nothing.
- Fix: Use the PixelMaps directly (writePixels/canvas) or pack to PNG before createImageSource; await the read.

### `HW-18-0031` - Falsy-zero guard `if (cameraPosition)` skips prepareCamera for the back camera (position 0) on the recorder-recovery path

- Category B, severity low, confidence confirmed
- Features: PHOTO-06
- Document: `huawei_industry_tree/18_photography/docs/06_video_recording.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_recording-0000002258016882
- Why: When startRecordingProcess has to rebuild the recorder (avRecorder undefined) with the back camera active, prepareCamera is skipped and recording starts with no camera session.
- Fix: Use `if (cameraPosition !== undefined)`.

### `HW-18-0032` - VideoPlay's `seeking` flag is initialized true and never toggled — the drag-conflict guard it implements is dead

- Category B, severity low, confidence confirmed
- Features: PHOTO-06
- Document: `huawei_industry_tree/18_photography/docs/06_video_recording.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_recording-0000002258016882
- Why: While dragging the slider, onUpdate keeps overwriting currentTime — exactly the conflict the flag was meant to prevent; the comment documents intent the code does not implement.
- Fix: Toggle seeking in Slider onChange begin/end (or via onTouch).

### `HW-18-0033` - previewPhoto launches the gallery via a hardcoded internal bundle name instead of a public action

- Category B, severity low, confidence confirmed
- Features: PHOTO-07
- Document: `huawei_industry_tree/18_photography/docs/07_customised_camera.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/customised_camera-0000002279772340
- Why: Hardcoding a system app's internal bundle/ability is fragile (fails where the bundle differs or the interface changes) and is not a public API contract; the thumbnail tap then silently does nothing.
- Fix: Use an implicit want with action/uri only, or the media library's preview capability.

### `HW-18-0034` - SaveImage error handling broken three ways: dialog refusal unhandled, double close of the gallery fd, closeSync on a null! file

- Category B, severity low, confidence confirmed
- Features: PHOTO-08
- Document: `huawei_industry_tree/18_photography/docs/08_image_segmentation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_segmentation-0000002282720108
- Why: Refusing the save dialog or any open failure produces unhandled rejections/TypeErrors instead of a clean failure toast; even the success path throws EBADF in the finally.
- Fix: Close each fd once (finally only), guard for null, add .catch on the dialog chain.

### `HW-18-0035` - Picker cancel leaves the page in a broken state: dead `URI === undefined` check and isSelected set before the pick

- Category B, severity low, confidence confirmed
- Features: PHOTO-08
- Document: `huawei_industry_tree/18_photography/docs/08_image_segmentation.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_segmentation-0000002282720108
- Why: Cancel (routine action) logs a bogus 'Image loading failed' error and leaves a half-active UI where crop appears to work but renders blank.
- Fix: Check photoUris.length, set isSelected only after a successful load.

### `HW-18-0038` - Completion flag reset by a fixed 500 ms timer instead of the compression promises

- Category B, severity low, confidence confirmed
- Features: PHOTO-09
- Document: `huawei_industry_tree/18_photography/docs/09_compress_images.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/compress_images-0000002322173825
- Why: Compressing 9 large photos takes >500 ms: the start button re-enables mid-batch, and a second tap starts an overlapping batch (duplicate outputs, duplicate toasts, extra deletions).
- Fix: Promise.all(the compressImg promises).then(() => isLastZipFin = true).

### `HW-18-0039` - isPixelMapChange flips before the snapshot exists, so the image binds to an undefined pixel map (and save can pack undefined)

- Category B, severity low, confidence confirmed
- Features: PHOTO-10
- Document: `huawei_industry_tree/18_photography/docs/10_picture_sticker.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/picture_sticker-0000002293373244
- Why: First 'finished' tap: blank image flash; snapshot failure leaves a permanently blank image and a broken save flow.
- Fix: Move isPixelMapChange=true into the then; reset on catch.

### `HW-18-0042` - Early returns after cameraInput.open() leave the opened camera never closed

- Category B, severity low, confidence confirmed
- Features: PHOTO-13
- Document: `huawei_industry_tree/18_photography/docs/13_capture_timer.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/capture_timer-0000002352218316
- Why: On a device failing the capability query the hardware camera stays open, orphaned, blocking other camera clients until app exit.
- Fix: Reorder the checks before open(), or call close() before each early return.

### `HW-18-0044` - Save-dialog refusal path: closeSync on undefined files in finally masks the real error and leaks the opened fd

- Category B, severity low, confidence confirmed
- Features: PHOTO-15
- Document: `huawei_industry_tree/18_photography/docs/15_video_clip.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_clip-0000002405051977
- Why: Refusing the save dialog (routine) produces a TypeError from the finally instead of a clean failure, with an fd leak.
- Fix: `if (srcFile) closeSync(srcFile)` etc.

### `HW-18-0045` - Preview pauses a full second before the selected end time ('Video play to the end position' comment notwithstanding)

- Category B, severity low, confidence probable
- Features: PHOTO-15
- Document: `huawei_industry_tree/18_photography/docs/15_video_clip.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_clip-0000002405051977
- Why: For any trimmed range the last ~1 s of the selection is never previewed (endTime=10 s pauses at 9 s), so the preview does not match the export.
- Fix: Compare against endTime/1000 without the -1 (or use ms).

### `HW-18-0046` - JPEG bytes saved into a .png gallery asset (2 samples), with createAsset outside the failure toast

- Category B, severity low, confidence confirmed
- Features: PHOTO-16
- Document: `huawei_industry_tree/18_photography/docs/16_image_effect.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_effect-0000002408004045
- Why: Exported photos carry a wrong extension/MIME, confusing format-sensitive consumers; save failures surface as unhandled rejections with no user feedback.
- Fix: Pack as png or create the asset as jpg; widen the try; add .catch.

### `HW-18-0049` - onBackPress unconditionally returns true and the on-screen back arrow has no handler — no way to leave the page

- Category B, severity low, confidence confirmed
- Features: PHOTO-17
- Document: `huawei_industry_tree/18_photography/docs/17_video_cropping.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_cropping-0000002385796378
- Why: System back gesture/button does nothing and the UI offers no alternative exit.
- Fix: Remove the blanket true / wire the back arrow.

### `HW-18-0051` - Taskpool split has no overlap rows, so block boundaries are filtered as if they were image edges (seam artifacts)

- Category B, severity low, confidence confirmed
- Features: PHOTO-18
- Document: `huawei_industry_tree/18_photography/docs/18_image_denoising.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_denoising-0000002419864905
- Why: Visible seam lines / incompletely denoised rows across the output at each block boundary.
- Fix: Pass one extra row on each side of every block and drop it after filtering.

### `HW-18-0055` - showBasic/showPip are only set when the frame loop reaches exactly t=1 s — videos of ≤1 s never display

- Category B, severity low, confidence confirmed
- Features: PHOTO-19
- Document: `huawei_industry_tree/18_photography/docs/19_video_edit_pip_window.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_edit_pip_window-0000002395437788
- Why: A short picked video silently cannot be used at all.
- Fix: Set the flag at timeUs===0 (first iteration).

### `HW-18-0057` - Always-true validation: `uri !== '' || uri !== undefined || uri !== null` (|| instead of &&)

- Category B, severity low, confidence confirmed
- Features: PHOTO-20
- Document: `huawei_industry_tree/18_photography/docs/20_image_rotate_and_flip.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_rotate_and_flip-0000002396643788
- Why: The intended validation never happens; empty/undefined URIs proceed into openSync and only the try/catch saves it.
- Fix: Join with && (or `if (!uri) return undefined`).

### `HW-18-0058` - WRITE_IMAGEVIDEO's usedScene references a nonexistent 'FormAbility'

- Category B, severity low, confidence confirmed
- Features: PHOTO-20
- Document: `huawei_industry_tree/18_photography/docs/20_image_rotate_and_flip.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_rotate_and_flip-0000002396643788
- Why: Copy-paste config: the scene binding is meaningless and misleads permission audits.
- Fix: Point usedScene at EntryAbility (or drop the declaration per PHOTO-10_001).

### `HW-18-0059` - Doc's flip signature returns Promise<image.PixelMap> while the (identical-body) sample function is synchronous

- Category D, severity low, confidence confirmed
- Features: PHOTO-20
- Document: `huawei_industry_tree/18_photography/docs/20_image_rotate_and_flip.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_rotate_and_flip-0000002396643788
- Why: Copying the documented signature yields a type error or a pointless await on a plain value.
- Fix: Fix the doc's return type.

### `HW-18-0062` - Cleanup finally uses rmdirSync on regular files — first call throws, aborting cleanup and leaking both fds

- Category B, severity low, confidence probable
- Features: PHOTO-21
- Document: `huawei_industry_tree/18_photography/docs/21_video_water_mark.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_water_mark-0000002408013854
- Why: Every save: cache files survive, both fds leak, and the throw propagates out of the callback.
- Fix: fileIo.unlinkSync + close first (or nested try).

### `HW-18-0063` - Square video (aspect ratio exactly 1) hits neither scale branch — watermark rendered at 1×1 px

- Category B, severity low, confidence confirmed
- Features: PHOTO-21
- Document: `huawei_industry_tree/18_photography/docs/21_video_water_mark.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_water_mark-0000002408013854
- Why: For 1:1 videos (common on social platforms) every watermark is an invisible single pixel.
- Fix: Use >= (or an explicit ===1 branch).

### `HW-18-0064` - timeFormat lacks a modulo on minutes: one hour renders as 01:61:01

- Category B, severity low, confidence confirmed
- Features: PHOTO-21
- Document: `huawei_industry_tree/18_photography/docs/21_video_water_mark.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_water_mark-0000002408013854
- Why: Progress/duration labels wrong for any video ≥1 h (3661 s → 01:61:01).
- Fix: minutes = floor(time/60) % 60.

### `HW-18-0065` - All text watermarks share the component id 'textMark' — the snapshot always captures the first one

- Category B, severity low, confidence probable
- Features: PHOTO-21
- Document: `huawei_industry_tree/18_photography/docs/21_video_water_mark.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_water_mark-0000002408013854
- Why: Add two different text marks: the second mark's cached PNG contains the first mark's text — the exported video shows the wrong content.
- Fix: Append the mark index to the id.

### `HW-18-0066` - Pagination flag leak: listVideos returning undefined leaves isLoading true, permanently blocking onReachEnd

- Category B, severity low, confidence confirmed
- Features: PHOTO-22
- Document: `huawei_industry_tree/18_photography/docs/22_video_encoding_convert.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_encoding_convert-0000002409157670
- Why: One empty page and all further pagination is dead for the session (onReachEnd guards on isLoading).
- Fix: Reset isLoading before the early return / in finally.

### `HW-18-0067` - Dedup guard dead: videoSet stores fileName but the check tests filePath

- Category B, severity low, confidence confirmed
- Features: PHOTO-22
- Document: `huawei_industry_tree/18_photography/docs/22_video_encoding_convert.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_encoding_convert-0000002409157670
- Why: Every listed file is re-stat'ed and re-parsed (metadata + thumbnail) on each call; the intended skip never fires.
- Fix: Use filePath (or fileName) consistently.

### `HW-18-0068` - AVMetadataExtractor/AVImageGenerator released twice; ParsePage closes the fd before the async player teardown

- Category B, severity low, confidence confirmed
- Features: PHOTO-22
- Document: `huawei_industry_tree/18_photography/docs/22_video_encoding_convert.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_encoding_convert-0000002409157670
- Why: Double release rejections with no handler; player teardown briefly reads a closed fd on page exit.
- Fix: Drop the duplicate releases; await stop/release then close.

### `HW-18-0069` - Mosaic CanvasPattern created once in the constructor before any canvas attaches; failure silently degrades strokes to black

- Category B, severity low, confidence probable
- Features: PHOTO-23
- Document: `huawei_industry_tree/18_photography/docs/23_image_draw_mosaic.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_draw_mosaic-0000002413444896
- Why: A failed early createPattern permanently downgrades the core mosaic effect to solid black lines with no error surfaced.
- Fix: Create the pattern in onReady; log/toast on failure.

### `HW-18-0071` - unregisterDataChangeListener uses `pos > 0` — the first-registered listener can never be removed

- Category B, severity low, confidence confirmed
- Features: PHOTO-24
- Document: `huawei_industry_tree/18_photography/docs/24_image_cropping.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_cropping-0000002426210646
- Why: LazyForEach's listener at index 0 stays registered after the consumer detaches — leaked listener notifying a dead component.
- Fix: Fix the comparison.

### `HW-18-0072` - Doc teaches initImageInfo() which is dead code in the sample; the real init path is initImageInfoWithActualSize()

- Category D, severity low, confidence confirmed
- Features: PHOTO-24
- Document: `huawei_industry_tree/18_photography/docs/24_image_cropping.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_cropping-0000002426210646
- Why: Readers implement an abandoned initialization that breaks the drag logic.
- Fix: Replace the doc snippet with initImageInfoWithActualSize.

### `HW-18-0083` - Success toast created via `new UIContext()` — an unbound context whose showToast fails silently

- Category B, severity low, confidence probable
- Features: PHOTO-28
- Document: `huawei_industry_tree/18_photography/docs/28_image_filter_processing.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_filter_processing-0000002520461010
- Why: The save-succeeded toast never appears — no user feedback on the save flow.
- Fix: Pass the page's getUIContext() into saveImage.

### `HW-18-0086` - Timed-capture feature is dead: timerShooting is never assigned anywhere

- Category B, severity low, confidence confirmed
- Features: PHOTO-29
- Document: `huawei_industry_tree/18_photography/docs/29_camera_twist.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/camera_twist-0000002552826219
- Why: The countdown branch is unreachable — the shutter always fires immediately despite the timer UI plumbing.
- Fix: Wire the timer selection UI to timerShooting.

### `HW-18-0087` - Doc snippet assigns string states 'ROTATION_90'/'ROTATION_270' and feeds them into rotate()/PhotoCaptureSetting — non-compiling and opposite to the sample's numeric mapping

- Category D, severity low, confidence confirmed
- Features: PHOTO-29
- Document: `huawei_industry_tree/18_photography/docs/29_camera_twist.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/camera_twist-0000002552826219
- Why: Following the doc yields invalid angle values and contradicts the shipped sample.
- Fix: Sync the doc snippet with ToolsComponents.ets.

