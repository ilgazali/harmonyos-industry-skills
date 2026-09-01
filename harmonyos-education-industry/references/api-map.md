# API map

> Generated from `features/*.md`. Source industry: `04_education`, 21 features.
> Do not edit by hand; regenerate it in the review repository.

## Feature to kits, permissions and API level

| ID | Kits | Permissions | Min API | Modules |
|---|---|---|---:|---|
| `EDU-01` | @kit.AbilityKit, @kit.ArkUI, @kit.MediaKit, @kit.BasicServicesKit, @kit.PerformanceAnalysisKit, @kit.ArkWeb, @kit.CoreFileKit, @kit.ArkData | ohos.permission.INTERNET, ohos.permission.PRIVACY_WINDOW | 20 | entry, common/basic, features/home, features/login, features/mine, features/online, features/train |
| `EDU-02` | - | - | 20 | - |
| `EDU-03` | @kit.ArkUI, @kit.AbilityKit, @kit.PerformanceAnalysisKit, @kit.BasicServicesKit | - | 20 | entry |
| `EDU-04` | @kit.ArkUI, @kit.AbilityKit, @kit.PerformanceAnalysisKit, @kit.BasicServicesKit | - | 20 | entry |
| `EDU-05` | @kit.ArkUI, @kit.AbilityKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `EDU-06` | @kit.ArkUI, @kit.AbilityKit, @kit.PerformanceAnalysisKit, @kit.BasicServicesKit | - | 20 | entry |
| `EDU-07` | @kit.ArkUI, @kit.AbilityKit, @kit.ArkTS, @kit.PerformanceAnalysisKit, @kit.LocalizationKit | - | 20 | entry |
| `EDU-08` | @kit.PDFKit, @kit.ImageKit, @kit.MediaLibraryKit, @kit.CoreFileKit, @kit.AbilityKit, @kit.ArkUI, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `EDU-09` | @kit.CoreFileKit, @kit.PDFKit, @kit.AbilityKit, @kit.ArkUI | - | 20 | entry |
| `EDU-10` | @kit.ArkUI, @kit.AbilityKit, @kit.CryptoArchitectureKit, @kit.ArkTS, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `EDU-11` | @kit.MediaKit, @kit.AbilityKit, @kit.CoreFileKit, @kit.LocalizationKit, @kit.CryptoArchitectureKit, @kit.ArkUI, @kit.ArkTS | ohos.permission.MICROPHONE | 20 | entry |
| `EDU-12` | @kit.NetworkKit, @kit.NetworkBoostKit, @kit.ArkTS, @kit.BasicServicesKit, @kit.PerformanceAnalysisKit | ohos.permission.GET_NETWORK_INFO | 20 | entry |
| `EDU-13` | @kit.CalendarKit, @kit.ArkData, @kit.AbilityKit, @kit.BasicServicesKit, @kit.ArkUI | ohos.permission.READ_CALENDAR, ohos.permission.WRITE_CALENDAR | 20 | entry |
| `EDU-14` | @kit.ArkUI, @kit.ArkTS, @kit.AbilityKit, @kit.LocalizationKit | - | 20 | entry |
| `EDU-15` | @kit.ArkUI, @kit.ArkTS, @kit.AbilityKit | - | 20 | entry |
| `EDU-16` | @kit.CameraKit, @kit.MediaKit, @kit.AbilityKit, @kit.CoreFileKit, @kit.ArkUI, @kit.PerformanceAnalysisKit | ohos.permission.CAMERA, ohos.permission.MICROPHONE | 20 | entry |
| `EDU-17` | @kit.ArkUI, @kit.MediaKit, @kit.LocalizationKit, @kit.ArkTS, @kit.AbilityKit | - | 20 | entry |
| `EDU-18` | @kit.ArkUI, @kit.AbilityKit | - | 20 | entry |
| `EDU-19` | @kit.ArkUI | - | 20 | entry |
| `EDU-20` | @kit.VisionKit, @kit.CoreFileKit, @kit.PDFKit, @kit.ArkUI, @kit.ArkTS, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `EDU-21` | - | - | 20 | - |

## Kit to features

| Kit | Features |
|---|---|
| `@kit.AbilityKit` | EDU-01, EDU-03, EDU-04, EDU-05, EDU-06, EDU-07, EDU-08, EDU-09, EDU-10, EDU-11, EDU-13, EDU-14, EDU-15, EDU-16, EDU-17, EDU-18 |
| `@kit.ArkData` | EDU-01, EDU-13 |
| `@kit.ArkTS` | EDU-07, EDU-10, EDU-11, EDU-12, EDU-14, EDU-15, EDU-17, EDU-20 |
| `@kit.ArkUI` | EDU-01, EDU-03, EDU-04, EDU-05, EDU-06, EDU-07, EDU-08, EDU-09, EDU-10, EDU-11, EDU-13, EDU-14, EDU-15, EDU-16, EDU-17, EDU-18, EDU-19, EDU-20 |
| `@kit.ArkWeb` | EDU-01 |
| `@kit.BasicServicesKit` | EDU-01, EDU-03, EDU-04, EDU-06, EDU-12, EDU-13 |
| `@kit.CalendarKit` | EDU-13 |
| `@kit.CameraKit` | EDU-16 |
| `@kit.CoreFileKit` | EDU-01, EDU-08, EDU-09, EDU-11, EDU-16, EDU-20 |
| `@kit.CryptoArchitectureKit` | EDU-10, EDU-11 |
| `@kit.ImageKit` | EDU-08 |
| `@kit.LocalizationKit` | EDU-07, EDU-11, EDU-14, EDU-17 |
| `@kit.MediaKit` | EDU-01, EDU-11, EDU-16, EDU-17 |
| `@kit.MediaLibraryKit` | EDU-08 |
| `@kit.NetworkBoostKit` | EDU-12 |
| `@kit.NetworkKit` | EDU-12 |
| `@kit.PDFKit` | EDU-08, EDU-09, EDU-20 |
| `@kit.PerformanceAnalysisKit` | EDU-01, EDU-03, EDU-04, EDU-05, EDU-06, EDU-07, EDU-08, EDU-10, EDU-12, EDU-16, EDU-20 |
| `@kit.VisionKit` | EDU-20 |

## Permission to features

| Permission | Features |
|---|---|
| `ohos.permission.CAMERA` | EDU-16 |
| `ohos.permission.GET_NETWORK_INFO` | EDU-12 |
| `ohos.permission.INTERNET` | EDU-01 |
| `ohos.permission.MICROPHONE` | EDU-11, EDU-16 |
| `ohos.permission.PRIVACY_WINDOW` | EDU-01 |
| `ohos.permission.READ_CALENDAR` | EDU-13 |
| `ohos.permission.WRITE_CALENDAR` | EDU-13 |

## API to features

| API | Features |
|---|---|
| `@ComponentV2` | EDU-06, EDU-19 |
| `@Consume` | EDU-07, EDU-09, EDU-14 |
| `@Consumer` | EDU-19 |
| `@Link` | EDU-03, EDU-13, EDU-15, EDU-17 |
| `@Local` | EDU-06, EDU-19 |
| `@LocalStorageLink` | EDU-18 |
| `@ObjectLink` | EDU-20 |
| `@Observed` | EDU-13, EDU-20 |
| `@ObservedV2` | EDU-06, EDU-10, EDU-19 |
| `@Prop` | EDU-07, EDU-17, EDU-18 |
| `@Provide` | EDU-07, EDU-09, EDU-14 |
| `@Provider` | EDU-19 |
| `@State` | EDU-15 |
| `@StorageLink` | EDU-17 |
| `@StorageProp` | EDU-01, EDU-03, EDU-04, EDU-05, EDU-07, EDU-09, EDU-11, EDU-14, EDU-18 |
| `@Trace` | EDU-06, EDU-10, EDU-19 |
| `@Watch` | EDU-07, EDU-13, EDU-17, EDU-18 |
| `AVFileDescriptor` | EDU-11 |
| `AVPlayer.on('stateChange')` | EDU-01 |
| `AVPlayer.on('timeUpdate')` | EDU-01 |
| `AVPlayer.seek` | EDU-01 |
| `AVPlayer.setSpeed` | EDU-01 |
| `AVRecorderConfig` | EDU-11, EDU-16 |
| `AVRecorderProfile` | EDU-11 |
| `AppStorage.setOrCreate` | EDU-01 |
| `AppStorageV2` | EDU-06 |
| `AppStorageV2.connect` | EDU-06 |
| `AudioSourceType.AUDIO_SOURCE_TYPE_MIC` | EDU-11 |
| `Calendar.addEvent` | EDU-13 |
| `Calendar.deleteEvent` | EDU-13 |
| `Calendar.getEvents` | EDU-13 |
| `CalendarManager.getCalendar` | EDU-13 |
| `CodecMimeType.AUDIO_AAC` | EDU-11 |
| `CommonModifier` | EDU-04 |
| `ContainerFormatType.CFT_MPEG_4A` | EDU-11 |
| `DataChangeListener` | EDU-07 |
| `DataPanel` | EDU-03 |
| `DataPanelOptions` | EDU-03 |
| `DataPanelType.Line` | EDU-03 |
| `Decimal` | EDU-12 |
| `Description` | EDU-18 |
| `Divider` | EDU-15 |
| `DocumentScanner` | EDU-20 |
| `DocumentScannerConfig` | EDU-20 |
| `EdgeEffect` | EDU-15 |
| `EdgeEffect.None` | EDU-04 |
| `EntryOhos` | EDU-18 |
| `FilterId` | EDU-20 |
| `FinishCallbackType` | EDU-06 |
| `ForEach` | EDU-04, EDU-10, EDU-15, EDU-19 |
| `GestureEvent` | EDU-19 |
| `GestureGroup` | EDU-06 |
| `GestureMode` | EDU-06 |
| `IAxisValueFormatter` | EDU-18 |
| `IDataSource` | EDU-07 |
| `ILineDataSet` | EDU-18 |
| `JArrayList` | EDU-18 |
| `KeyframeState` | EDU-17 |
| `LazyForEach` | EDU-07 |
| `Legend` | EDU-18 |
| `LineBreakStrategy` | EDU-14 |
| `LineChart` | EDU-18 |
| `LineChartModel` | EDU-18 |
| `LineData` | EDU-18 |
| `LineDataSet` | EDU-18 |
| `List` | EDU-04, EDU-15 |
| `ListItem` | EDU-15 |
| `ListScroller` | EDU-15 |
| `MeasureUtils` | EDU-05 |
| `NavDestination` | EDU-01, EDU-06, EDU-07, EDU-09, EDU-14, EDU-19 |
| `NavDestinationContext` | EDU-09 |
| `NavPathStack` | EDU-01, EDU-06, EDU-09, EDU-14, EDU-15, EDU-19, EDU-20 |
| `NavPathStack.pop` | EDU-15 |
| `Navigation` | EDU-01, EDU-06, EDU-07, EDU-09, EDU-14, EDU-15, EDU-19, EDU-20 |
| `NetConnection.register` | EDU-12 |
| `NetConnection.unregister` | EDU-12 |
| `PanDirection` | EDU-03 |
| `PanGesture` | EDU-03, EDU-06, EDU-19 |
| `PanGestureOptions` | EDU-06 |
| `PdfView` | EDU-08, EDU-09 |
| `PixelMap.release` | EDU-08 |
| `Progress` | EDU-14 |
| `PromptAction` | EDU-06, EDU-07, EDU-11, EDU-14, EDU-15, EDU-17 |
| `Radio` | EDU-07 |
| `RadioIndicatorType` | EDU-07 |
| `ReadOptions` | EDU-20 |
| `RelativeContainer` | EDU-07 |
| `SaveButton` | EDU-08 |
| `SaveButtonOnClickResult` | EDU-08 |
| `SaveDescription` | EDU-08 |
| `SaveOption` | EDU-20 |
| `Scroll` | EDU-04, EDU-14 |
| `ScrollDirection` | EDU-04 |
| `Scroller` | EDU-04, EDU-14 |
| `Session.start` | EDU-16 |
| `ShootingMode` | EDU-20 |
| `SizeOptions` | EDU-14 |
| `Span` | EDU-14 |
| `Stack` | EDU-05, EDU-06, EDU-10 |
| `StyledString` | EDU-05 |
| `Swiper` | EDU-07, EDU-17 |
| `SwiperController` | EDU-07, EDU-17 |
| `TabContent` | EDU-04 |
| `Tabs` | EDU-04, EDU-09 |
| `Text` | EDU-05, EDU-14 |
| `TextController` | EDU-14 |
| `TextDecoderOptions` | EDU-15 |
| `TextInput` | EDU-17 |
| `TextTimer` | EDU-01 |
| `TextTimerController` | EDU-01 |
| `Toggle` | EDU-10 |
| `TouchType` | EDU-19 |
| `Video` | EDU-10 |
| `VideoController` | EDU-10 |
| `WriteOptions` | EDU-20 |
| `XAxis` | EDU-18 |
| `XAxisPosition` | EDU-18 |
| `XComponent` | EDU-01, EDU-16 |
| `XComponentController` | EDU-01, EDU-16 |
| `YAxis` | EDU-18 |
| `YAxisLabelPosition` | EDU-18 |
| `abilityAccessCtrl.createAtManager` | EDU-11, EDU-13, EDU-16 |
| `addInput` | EDU-16 |
| `addOutput` | EDU-16 |
| `alignRules` | EDU-07 |
| `animateTo` | EDU-06, EDU-10, EDU-11, EDU-17 |
| `barModifier` | EDU-04 |
| `beginConfig` | EDU-16 |
| `bindSheet` | EDU-16 |
| `bundleManager.getBundleInfoForSelf` | EDU-16 |
| `cachedCount` | EDU-07 |
| `calendarManager.Event` | EDU-13 |
| `calendarManager.EventType` | EDU-13 |
| `calendarManager.getCalendarManager` | EDU-13 |
| `camera.Profile` | EDU-16 |
| `camera.SceneMode` | EDU-16 |
| `camera.getCameraManager` | EDU-16 |
| `changeIndex` | EDU-17 |
| `checkAccessToken` | EDU-11, EDU-16 |
| `clearInterval` | EDU-10, EDU-11, EDU-12, EDU-19 |
| `clearTimeout` | EDU-10 |
| `commitConfig` | EDU-16 |
| `connection.NetBearType` | EDU-12 |
| `connection.createNetConnection` | EDU-12 |
| `connection.getDefaultNetSync` | EDU-12 |
| `connection.getNetCapabilitiesSync` | EDU-12 |
| `copyOption` | EDU-17 |
| `createAsset` | EDU-08 |
| `createCameraInput` | EDU-16 |
| `createPreviewOutput` | EDU-16 |
| `createSession` | EDU-16 |
| `cryptoFramework.createRandom` | EDU-10 |
| `currentOffset` | EDU-14 |
| `dataPreferences.getPreferences` | EDU-01 |
| `defaultFocus` | EDU-17 |
| `deleteSync` | EDU-13 |
| `dialogShownResults` | EDU-13 |
| `disableSwipe` | EDU-17 |
| `display.getDefaultDisplaySync` | EDU-05, EDU-10 |
| `fdSrc` | EDU-11, EDU-16, EDU-17 |
| `fileIo` | EDU-08 |
| `fileIo.accessSync` | EDU-20 |
| `fileIo.close` | EDU-11 |
| `fileIo.closeSync` | EDU-09 |
| `fileIo.copyFile` | EDU-20 |
| `fileIo.lstatSync` | EDU-20 |
| `fileIo.open` | EDU-11 |
| `fileIo.openSync` | EDU-09, EDU-20 |
| `fileIo.readSync` | EDU-20 |
| `fileIo.unlinkSync` | EDU-20 |
| `fileIo.writeSync` | EDU-20 |
| `fileSuffixFilters` | EDU-09 |
| `flush` | EDU-13 |
| `fs.File` | EDU-09 |
| `fs.listFile` | EDU-01 |
| `generateRandomSync` | EDU-10 |
| `getAudioCapturerMaxAmplitude` | EDU-11 |
| `getPage` | EDU-08 |
| `getPageCount` | EDU-08, EDU-09 |
| `getPagePixelMap` | EDU-08 |
| `getParamByName` | EDU-09, EDU-14 |
| `getSupportedCameras` | EDU-16 |
| `getSupportedOutputCapability` | EDU-16 |
| `getSupportedSceneModes` | EDU-16 |
| `getSync` | EDU-13 |
| `getXComponentSurfaceId` | EDU-01, EDU-16 |
| `hasSync` | EDU-13 |
| `image.InitializationOptions` | EDU-08 |
| `image.PositionArea` | EDU-08 |
| `image.createImagePacker` | EDU-08 |
| `image.createPixelMap` | EDU-08 |
| `invalidate` | EDU-18 |
| `isGallerySupported` | EDU-20 |
| `isShareable` | EDU-20 |
| `keyframeAnimateTo` | EDU-17 |
| `linkDownRate` | EDU-12 |
| `listDirection` | EDU-04 |
| `loadDocument` | EDU-08, EDU-09 |
| `maxLines` | EDU-05 |
| `maxShotCount` | EDU-20 |
| `measureText` | EDU-05 |
| `measureTextSize` | EDU-05 |
| `media.createAVPlayer` | EDU-01, EDU-11, EDU-16, EDU-17 |
| `media.createAVRecorder` | EDU-11, EDU-16 |
| `netQuality.NetworkQos` | EDU-12 |
| `netQuality.off('netQosChange')` | EDU-12 |
| `netQuality.on('netQosChange')` | EDU-12 |
| `notifyDataChange` | EDU-07 |
| `notifyDataReload` | EDU-07 |
| `notifyDataSetChanged` | EDU-18 |
| `on('keyboardHeightChange')` | EDU-17 |
| `on('netAvailable')` | EDU-12 |
| `on('netCapabilitiesChange')` | EDU-12 |
| `on('netConnectionPropertiesChange')` | EDU-12 |
| `on('netLost')` | EDU-12 |
| `onActionEnd` | EDU-19 |
| `onActionStart` | EDU-03, EDU-19 |
| `onActionUpdate` | EDU-03, EDU-19 |
| `onAnimationStart` | EDU-17 |
| `onAreaChange` | EDU-10 |
| `onChange` | EDU-17 |
| `onContentWillChange` | EDU-04, EDU-09 |
| `onDidScroll` | EDU-14 |
| `onFinish` | EDU-10 |
| `onPop` | EDU-20 |
| `onPrepared` | EDU-10 |
| `onReachEnd` | EDU-14 |
| `onReady` | EDU-06 |
| `onResult` | EDU-20 |
| `onScrollFrameBegin` | EDU-04 |
| `onSizeChange` | EDU-14 |
| `onTouch` | EDU-19 |
| `onUpdate` | EDU-10 |
| `onWillChange` | EDU-17 |
| `opacity` | EDU-06, EDU-19 |
| `originalUris` | EDU-20 |
| `packToData` | EDU-08 |
| `pathInfo` | EDU-09 |
| `pdfService.PageFit` | EDU-08, EDU-09 |
| `pdfService.ParseResult` | EDU-08 |
| `pdfService.PdfDocument` | EDU-08 |
| `pdfViewManager.PdfController` | EDU-08, EDU-09 |
| `photoAccessHelper.getPhotoAccessHelper` | EDU-08 |
| `picker.DocumentSelectOptions` | EDU-09 |
| `picker.DocumentViewPicker` | EDU-09 |
| `position` | EDU-03, EDU-04, EDU-10, EDU-19 |
| `preferences.getPreferencesSync` | EDU-13 |
| `prepare` | EDU-11 |
| `pushPath` | EDU-20 |
| `pushPathByName` | EDU-09, EDU-14 |
| `putSync` | EDU-13 |
| `px2vp` | EDU-05 |
| `readPixelsToBuffer` | EDU-08 |
| `readPixelsToBufferSync` | EDU-08 |
| `release` | EDU-11 |
| `reminderTime` | EDU-13 |
| `requestPermissionOnSetting` | EDU-11, EDU-13 |
| `requestPermissionsFromUser` | EDU-11, EDU-13, EDU-16 |
| `resourceManager.closeRawFdSync` | EDU-17 |
| `resourceManager.getRawFdSync` | EDU-11, EDU-17 |
| `resourceManager.getRawFileContent` | EDU-14 |
| `resourceManager.getRawFileContentSync` | EDU-07, EDU-08, EDU-15 |
| `routerMap` | EDU-01 |
| `saveDocument` | EDU-09 |
| `scrollBy` | EDU-04 |
| `scrollTo` | EDU-04 |
| `select` | EDU-09 |
| `setInterval` | EDU-10, EDU-11, EDU-12, EDU-19 |
| `setLabelCount` | EDU-18 |
| `setTimeout` | EDU-10 |
| `setValueFormatter` | EDU-18 |
| `setVisibleXRangeMaximum` | EDU-18 |
| `setWindowPrivacyMode` | EDU-01 |
| `setWindowSystemBarProperties` | EDU-03 |
| `setXComponentSurfaceRect` | EDU-01, EDU-16 |
| `showNext` | EDU-07 |
| `showPrevious` | EDU-07 |
| `showToast` | EDU-06, EDU-07 |
| `start` | EDU-11 |
| `stop` | EDU-11 |
| `subStyledString` | EDU-05 |
| `textIndent` | EDU-14 |
| `textOverflow` | EDU-05 |
| `toDecimalPlaces` | EDU-12 |
| `translate` | EDU-06, EDU-10, EDU-17 |
| `util.TextDecoder` | EDU-14, EDU-15 |
| `util.generateRandomUUID` | EDU-20 |
| `valueColors` | EDU-03 |
| `visibility` | EDU-05 |
| `window.getLastWindow` | EDU-17 |
| `window.getWindowAvoidArea` | EDU-01, EDU-03 |
| `window.on('avoidAreaChange')` | EDU-01 |
| `window.setPreferredOrientation` | EDU-10 |
| `window.setWindowLayoutFullScreen` | EDU-01, EDU-03 |
| `writePixels` | EDU-08 |
| `zIndex` | EDU-04, EDU-06 |
