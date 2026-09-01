# API map

> Generated from `features/*.md`. Source industry: `16_shopping`, 24 features.
> Do not edit by hand; regenerate it in the review repository.

## Feature to kits, permissions and API level

| ID | Kits | Permissions | Min API | Modules |
|---|---|---|---:|---|
| `SHOP-01` | @kit.AbilityKit, @kit.ArkData, @kit.ArkTS, @kit.ArkUI, @kit.BasicServicesKit, @kit.CameraKit, @kit.CoreSpeechKit, @kit.CryptoArchitectureKit, @kit.IMEKit, @kit.ImageKit, @kit.LocalizationKit, @kit.LocationKit, @kit.MediaKit, @kit.MediaLibraryKit, @kit.PaymentKit, @kit.PerformanceAnalysisKit, @kit.ScanKit, @kit.ShareKit, @kit.VisionKit | ohos.permission.CAMERA, ohos.permission.MICROPHONE, ohos.permission.APPROXIMATELY_LOCATION, ohos.permission.READ_IMAGEVIDEO, ohos.permission.WRITE_IMAGEVIDEO | 20 | phone (entry), common (har), camera (har), commoditydetail (har), conversation (har), home (har), orderdetail (har), personal (har), search (har), shopcart (har) |
| `SHOP-02` | - | - | 20 | - |
| `SHOP-03` | @kit.AbilityKit, @kit.ArkTS, @kit.ArkUI, @kit.BasicServicesKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `SHOP-04` | @kit.AbilityKit, @kit.ArkData, @kit.ArkUI, @kit.BasicServicesKit, @kit.CoreFileKit, @kit.NotificationKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `SHOP-05` | @kit.DeviceSecurityKit, @kit.ArkData, @kit.BasicServicesKit, @kit.PerformanceAnalysisKit, @kit.AbilityKit, @kit.ArkUI | ohos.permission.INTERNET | 21 | entry |
| `SHOP-06` | @kit.ArkUI, @kit.ArkWeb, @kit.AbilityKit, @kit.BasicServicesKit, @kit.CameraKit, @kit.CoreFileKit, @kit.MediaLibraryKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `SHOP-07` | @kit.ArkUI, @kit.ImageKit, @kit.PerformanceAnalysisKit, @kit.AbilityKit, @kit.BasicServicesKit, @kit.CoreFileKit | - | 20 | entry |
| `SHOP-08` | @kit.ArkUI, @kit.AbilityKit, @kit.BasicServicesKit, @kit.PerformanceAnalysisKit, @kit.CoreFileKit | - | 20 | entry |
| `SHOP-09` | @kit.AbilityKit, @kit.BasicServicesKit | - | 20 | entry |
| `SHOP-10` | @kit.AbilityKit, @kit.ArkUI, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `SHOP-11` | @kit.AbilityKit, @kit.ArkUI, @kit.BasicServicesKit, @kit.CoreFileKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `SHOP-12` | @kit.AbilityKit, @kit.ArkUI, @kit.CoreFileKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `SHOP-13` | @kit.AbilityKit, @kit.ArkData, @kit.ArkTS, @kit.ArkUI, @kit.BasicServicesKit, @kit.CoreFileKit, @kit.ImageKit, @kit.MediaLibraryKit, @kit.PerformanceAnalysisKit | - | 20 | entry (entry) |
| `SHOP-14` | @kit.AbilityKit, @kit.ArkUI, @kit.BasicServicesKit, @kit.CoreFileKit, @kit.PerformanceAnalysisKit | - | 20 | entry (entry) |
| `SHOP-15` | @kit.AbilityKit, @kit.ArkUI, @kit.CoreFileKit, @kit.PerformanceAnalysisKit | - | 20 | entry (entry) |
| `SHOP-16` | @kit.AbilityKit, @kit.ArkUI, @kit.BasicServicesKit, @kit.CalendarKit, @kit.CoreFileKit, @kit.PerformanceAnalysisKit | ohos.permission.READ_CALENDAR, ohos.permission.WRITE_CALENDAR | 20 | entry (entry) |
| `SHOP-17` | @kit.AbilityKit, @kit.ArkUI, @kit.CoreFileKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `SHOP-18` | @kit.AbilityKit, @kit.ArkUI, @kit.BasicServicesKit, @kit.CoreFileKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `SHOP-19` | @kit.AbilityKit, @kit.ArkUI, @kit.BasicServicesKit, @kit.CoreFileKit, @kit.CryptoArchitectureKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `SHOP-20` | @kit.AbilityKit, @kit.ArkUI, @kit.BasicServicesKit, @kit.CoreFileKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `SHOP-21` | @kit.AbilityKit, @kit.ArkUI, @kit.BasicServicesKit, @kit.CoreFileKit, @kit.PerformanceAnalysisKit | - | 20 | entry (entry) |
| `SHOP-22` | @kit.AbilityKit, @kit.ArkUI, @kit.BasicServicesKit, @kit.CoreFileKit | - | 20 | entry (entry) |
| `SHOP-23` | @kit.AbilityKit, @kit.BasicServicesKit, @kit.ContactsKit, @kit.CoreFileKit, @kit.LocationKit, @kit.PerformanceAnalysisKit | ohos.permission.APPROXIMATELY_LOCATION, ohos.permission.LOCATION | 20 | entry (entry) |
| `SHOP-24` | - | - | n/a | - |

## Kit to features

| Kit | Features |
|---|---|
| `@kit.AbilityKit` | SHOP-01, SHOP-03, SHOP-04, SHOP-05, SHOP-06, SHOP-07, SHOP-08, SHOP-09, SHOP-10, SHOP-11, SHOP-12, SHOP-13, SHOP-14, SHOP-15, SHOP-16, SHOP-17, SHOP-18, SHOP-19, SHOP-20, SHOP-21, SHOP-22, SHOP-23 |
| `@kit.ArkData` | SHOP-01, SHOP-04, SHOP-05, SHOP-13 |
| `@kit.ArkTS` | SHOP-01, SHOP-03, SHOP-13 |
| `@kit.ArkUI` | SHOP-01, SHOP-03, SHOP-04, SHOP-05, SHOP-06, SHOP-07, SHOP-08, SHOP-10, SHOP-11, SHOP-12, SHOP-13, SHOP-14, SHOP-15, SHOP-16, SHOP-17, SHOP-18, SHOP-19, SHOP-20, SHOP-21, SHOP-22 |
| `@kit.ArkWeb` | SHOP-06 |
| `@kit.BasicServicesKit` | SHOP-01, SHOP-03, SHOP-04, SHOP-05, SHOP-06, SHOP-07, SHOP-08, SHOP-09, SHOP-11, SHOP-13, SHOP-14, SHOP-16, SHOP-18, SHOP-19, SHOP-20, SHOP-21, SHOP-22, SHOP-23 |
| `@kit.CalendarKit` | SHOP-16 |
| `@kit.CameraKit` | SHOP-01, SHOP-06 |
| `@kit.ContactsKit` | SHOP-23 |
| `@kit.CoreFileKit` | SHOP-04, SHOP-06, SHOP-07, SHOP-08, SHOP-11, SHOP-12, SHOP-13, SHOP-14, SHOP-15, SHOP-16, SHOP-17, SHOP-18, SHOP-19, SHOP-20, SHOP-21, SHOP-22, SHOP-23 |
| `@kit.CoreSpeechKit` | SHOP-01 |
| `@kit.CryptoArchitectureKit` | SHOP-01, SHOP-19 |
| `@kit.DeviceSecurityKit` | SHOP-05 |
| `@kit.IMEKit` | SHOP-01 |
| `@kit.ImageKit` | SHOP-01, SHOP-07, SHOP-13 |
| `@kit.LocalizationKit` | SHOP-01 |
| `@kit.LocationKit` | SHOP-01, SHOP-23 |
| `@kit.MediaKit` | SHOP-01 |
| `@kit.MediaLibraryKit` | SHOP-01, SHOP-06, SHOP-13 |
| `@kit.NotificationKit` | SHOP-04 |
| `@kit.PaymentKit` | SHOP-01 |
| `@kit.PerformanceAnalysisKit` | SHOP-01, SHOP-03, SHOP-04, SHOP-05, SHOP-06, SHOP-07, SHOP-08, SHOP-10, SHOP-11, SHOP-12, SHOP-13, SHOP-14, SHOP-15, SHOP-16, SHOP-17, SHOP-18, SHOP-19, SHOP-20, SHOP-21, SHOP-23 |
| `@kit.ScanKit` | SHOP-01 |
| `@kit.ShareKit` | SHOP-01 |
| `@kit.VisionKit` | SHOP-01 |

## Permission to features

| Permission | Features |
|---|---|
| `ohos.permission.APPROXIMATELY_LOCATION` | SHOP-01, SHOP-23 |
| `ohos.permission.CAMERA` | SHOP-01 |
| `ohos.permission.INTERNET` | SHOP-05 |
| `ohos.permission.LOCATION` | SHOP-23 |
| `ohos.permission.MICROPHONE` | SHOP-01 |
| `ohos.permission.READ_CALENDAR` | SHOP-16 |
| `ohos.permission.READ_IMAGEVIDEO` | SHOP-01 |
| `ohos.permission.WRITE_CALENDAR` | SHOP-16 |
| `ohos.permission.WRITE_IMAGEVIDEO` | SHOP-01 |

## API to features

| API | Features |
|---|---|
| `$$` | SHOP-20 |
| `@Builder` | SHOP-20 |
| `@BuilderParam` | SHOP-11 |
| `@ComponentV2` | SHOP-10, SHOP-11 |
| `@Consume` | SHOP-01, SHOP-04, SHOP-13, SHOP-15, SHOP-21 |
| `@CustomDialog` | SHOP-18 |
| `@Extend` | SHOP-23 |
| `@Link` | SHOP-08, SHOP-16, SHOP-18, SHOP-19, SHOP-22, SHOP-23 |
| `@Local` | SHOP-10, SHOP-11 |
| `@LocalStorageLink` | SHOP-23 |
| `@ObservedV2` | SHOP-17, SHOP-18 |
| `@Param` | SHOP-10, SHOP-11 |
| `@Prop` | SHOP-12 |
| `@Provide` | SHOP-01, SHOP-04, SHOP-13, SHOP-15, SHOP-21 |
| `@State` | SHOP-08, SHOP-12, SHOP-19 |
| `@StorageLink` | SHOP-03, SHOP-04, SHOP-15, SHOP-16, SHOP-18, SHOP-22 |
| `@StorageProp` | SHOP-01, SHOP-05, SHOP-06, SHOP-08, SHOP-09, SHOP-15, SHOP-16, SHOP-19, SHOP-20, SHOP-21, SHOP-22 |
| `@Trace` | SHOP-17, SHOP-18 |
| `@Watch` | SHOP-10 |
| `AnimateParam` | SHOP-11 |
| `AnimatorOptions` | SHOP-17 |
| `AnimatorResult` | SHOP-17, SHOP-22 |
| `AnimatorResult.onFrame` | SHOP-22 |
| `AppStorage.get` | SHOP-08, SHOP-15 |
| `AppStorage.setOrCreate` | SHOP-08, SHOP-09, SHOP-11, SHOP-15 |
| `BarMode` | SHOP-09 |
| `Blank` | SHOP-12 |
| `BorderStyle` | SHOP-03 |
| `BottomTabBarStyle` | SHOP-15 |
| `Calendar.addEvent` | SHOP-16 |
| `Calendar.setConfig` | SHOP-16 |
| `CalendarType` | SHOP-16 |
| `Canvas` | SHOP-07 |
| `CanvasRenderingContext2D` | SHOP-07 |
| `Checkbox` | SHOP-23 |
| `ComponentContent` | SHOP-04 |
| `Curve` | SHOP-19 |
| `CustomDialogController` | SHOP-18, SHOP-23 |
| `DataChangeListener` | SHOP-10 |
| `DialogAlignment` | SHOP-04 |
| `DismissSheetAction` | SHOP-21 |
| `EdgeEffect` | SHOP-14 |
| `EventType` | SHOP-16 |
| `Flex` | SHOP-09, SHOP-12 |
| `FlexWrap` | SHOP-09, SHOP-12 |
| `FlowItem` | SHOP-15 |
| `ForEach` | SHOP-03, SHOP-09, SHOP-11, SHOP-12, SHOP-17, SHOP-18, SHOP-19, SHOP-20, SHOP-23 |
| `GestureGroup` | SHOP-08, SHOP-18 |
| `GestureMode` | SHOP-08, SHOP-18 |
| `Grid` | SHOP-08, SHOP-10, SHOP-13, SHOP-19 |
| `GridItem` | SHOP-08, SHOP-13, SHOP-19 |
| `HitTestMode` | SHOP-09 |
| `IDataSource` | SHOP-01, SHOP-10 |
| `ImagePacker.packToData` | SHOP-13 |
| `ImageSource.createPixelMap` | SHOP-07 |
| `ImageSource.release` | SHOP-07 |
| `Intl.DateTimeFormat` | SHOP-16 |
| `LazyForEach` | SHOP-01, SHOP-10 |
| `LengthMetrics` | SHOP-03 |
| `List` | SHOP-03, SHOP-11, SHOP-14, SHOP-18, SHOP-19, SHOP-23 |
| `List.lanes` | SHOP-01, SHOP-14 |
| `ListItem` | SHOP-03, SHOP-11, SHOP-14, SHOP-18, SHOP-23 |
| `ListScroller` | SHOP-14 |
| `ListScroller.currentOffset` | SHOP-14 |
| `ListScroller.scrollToIndex` | SHOP-14 |
| `LocalStorage` | SHOP-23 |
| `LongPressGesture` | SHOP-08, SHOP-18 |
| `MIMETYPE_TEXT_PLAIN` | SHOP-23 |
| `MediaAssetChangeRequest` | SHOP-01 |
| `NavDestination` | SHOP-04, SHOP-06, SHOP-08, SHOP-13, SHOP-15, SHOP-20, SHOP-21, SHOP-23 |
| `NavDestination.onReady` | SHOP-21 |
| `NavDestinationContext` | SHOP-20 |
| `NavPathInfo` | SHOP-23 |
| `NavPathStack` | SHOP-01, SHOP-04, SHOP-08, SHOP-13, SHOP-15, SHOP-20, SHOP-21, SHOP-23 |
| `NavPathStack.pushPath` | SHOP-06 |
| `Navigation` | SHOP-04, SHOP-06, SHOP-13, SHOP-15, SHOP-20, SHOP-21, SHOP-23 |
| `NestedScrollMode` | SHOP-10 |
| `NotificationRequest` | SHOP-04 |
| `PersistentStorage.persistProp` | SHOP-15 |
| `PhotoAccessHelper.getAssets` | SHOP-13 |
| `PhotoAsset.getThumbnail` | SHOP-13 |
| `PhotoViewPicker.select` | SHOP-13 |
| `Progress` | SHOP-04 |
| `ProgressType` | SHOP-04 |
| `Radio` | SHOP-21 |
| `RadioStyle` | SHOP-21 |
| `Rating` | SHOP-13 |
| `Refresh` | SHOP-06, SHOP-10 |
| `RefreshStatus` | SHOP-06 |
| `RenderingContextSettings` | SHOP-07 |
| `ReverseGeoCodeRequest` | SHOP-23 |
| `SafeAreaEdge` | SHOP-12 |
| `SafeAreaType` | SHOP-12 |
| `Scroll` | SHOP-10, SHOP-14 |
| `ScrollSnapAlign` | SHOP-14 |
| `Scroller` | SHOP-10 |
| `Search` | SHOP-12, SHOP-20 |
| `SearchController` | SHOP-20 |
| `Span` | SHOP-09 |
| `Stack` | SHOP-07, SHOP-08, SHOP-09, SHOP-12, SHOP-20 |
| `SubTabBarStyle` | SHOP-13 |
| `Swiper` | SHOP-20, SHOP-22 |
| `SwiperController` | SHOP-20 |
| `TabContent` | SHOP-09, SHOP-10, SHOP-13 |
| `Tabs` | SHOP-06, SHOP-09, SHOP-10, SHOP-13, SHOP-15, SHOP-22 |
| `TabsController` | SHOP-10, SHOP-22 |
| `TapGesture` | SHOP-08, SHOP-18 |
| `TextArea` | SHOP-13, SHOP-23 |
| `TextInput` | SHOP-09 |
| `TextInputController` | SHOP-09 |
| `TextTimer` | SHOP-04 |
| `TextTimerController` | SHOP-04 |
| `Toggle` | SHOP-04 |
| `ToggleType` | SHOP-04 |
| `TouchEvent` | SHOP-22 |
| `TouchType` | SHOP-22 |
| `UIContext.animateTo` | SHOP-10, SHOP-11, SHOP-19, SHOP-22 |
| `UIContext.createAnimator` | SHOP-17, SHOP-22 |
| `UIContext.getFocusController` | SHOP-20 |
| `UIContext.getPromptAction` | SHOP-03, SHOP-04, SHOP-05, SHOP-08, SHOP-18 |
| `UIContext.px2vp` | SHOP-03, SHOP-04, SHOP-18, SHOP-19, SHOP-20 |
| `UIContext.showDatePickerDialog` | SHOP-16 |
| `UIContext.vp2px` | SHOP-07 |
| `Visibility` | SHOP-12 |
| `WaterFlow` | SHOP-15 |
| `Web` | SHOP-06 |
| `XComponent` | SHOP-01 |
| `XComponentController` | SHOP-01 |
| `abilityAccessCtrl` | SHOP-01, SHOP-16 |
| `abilityAccessCtrl.createAtManager` | SHOP-23 |
| `aboutToAppear` | SHOP-12 |
| `arc` | SHOP-07 |
| `autoFillManager` | SHOP-01 |
| `barHeight` | SHOP-10 |
| `base` | SHOP-01 |
| `beginPath` | SHOP-07 |
| `bindSheet` | SHOP-21 |
| `border` | SHOP-03 |
| `borderRadius` | SHOP-11 |
| `buffer.from` | SHOP-03 |
| `bundleManager.getBundleInfoForSelf` | SHOP-23 |
| `calendarManager` | SHOP-16 |
| `calendarManager.getCalendarManager` | SHOP-16 |
| `camera` | SHOP-01 |
| `cameraPicker` | SHOP-01 |
| `cameraPicker.pick` | SHOP-06 |
| `checkAccessToken` | SHOP-23 |
| `clearFocus` | SHOP-20 |
| `clearInterval` | SHOP-16, SHOP-19 |
| `clip` | SHOP-11, SHOP-12, SHOP-19 |
| `closeCustomDialog` | SHOP-04 |
| `closePath` | SHOP-07 |
| `columnsTemplate` | SHOP-08 |
| `common` | SHOP-01, SHOP-13, SHOP-16 |
| `componentUtils` | SHOP-14, SHOP-22 |
| `componentUtils.getRectangleById` | SHOP-10 |
| `constraintSize` | SHOP-12 |
| `contact.selectContacts` | SHOP-23 |
| `createCalendar` | SHOP-16 |
| `cryptoFramework` | SHOP-01 |
| `cryptoFramework.createRandom` | SHOP-19 |
| `customScan` | SHOP-01 |
| `customScan.init` | SHOP-01 |
| `customScan.release` | SHOP-01 |
| `customScan.start` | SHOP-01 |
| `customScan.stop` | SHOP-01 |
| `dataSharePredicates` | SHOP-13 |
| `defaultFocus` | SHOP-20 |
| `deleteSync` | SHOP-05 |
| `detectBarcode` | SHOP-01 |
| `detectBarcode.decode` | SHOP-01 |
| `deviceCertificate.getDeviceToken` | SHOP-05 |
| `deviceInfo` | SHOP-01 |
| `display` | SHOP-01 |
| `display.getDefaultDisplaySync` | SHOP-18 |
| `draggable` | SHOP-08 |
| `drawImage` | SHOP-07 |
| `expandSafeArea` | SHOP-03, SHOP-07, SHOP-10, SHOP-12 |
| `fill` | SHOP-07 |
| `flush` | SHOP-04 |
| `flushSync` | SHOP-05 |
| `friction` | SHOP-14 |
| `fs` | SHOP-01 |
| `generateRandomSync` | SHOP-19 |
| `geoLocationManager.getAddressesFromLocation` | SHOP-23 |
| `geoLocationManager.getCurrentLocation` | SHOP-23 |
| `get` | SHOP-04 |
| `getAudioCapturerMaxAmplitude` | SHOP-01 |
| `getCalendar` | SHOP-16 |
| `getParamByName` | SHOP-15, SHOP-21 |
| `getPromptAction` | SHOP-21 |
| `getRectangleById` | SHOP-22 |
| `getSync` | SHOP-05 |
| `globalCompositeOperation` | SHOP-07 |
| `hasSync` | SHOP-05 |
| `hilog` | SHOP-03, SHOP-05, SHOP-09, SHOP-11, SHOP-12, SHOP-13, SHOP-14, SHOP-15, SHOP-16, SHOP-17, SHOP-19, SHOP-21, SHOP-22 |
| `hitTestBehavior` | SHOP-09 |
| `image` | SHOP-13 |
| `image.DecodingOptions` | SHOP-07 |
| `image.createImageSource` | SHOP-07 |
| `iterations` | SHOP-11 |
| `linearGradient` | SHOP-09, SHOP-11 |
| `listDirection` | SHOP-14 |
| `media.createAVRecorder` | SHOP-01 |
| `mediaquery.matchMediaSync` | SHOP-01 |
| `nestedScroll` | SHOP-10 |
| `notificationManager.publish` | SHOP-04 |
| `offset` | SHOP-22 |
| `onAnimationStart` | SHOP-10 |
| `onAppear` | SHOP-17 |
| `onAreaChange` | SHOP-09, SHOP-10, SHOP-14, SHOP-17 |
| `onChange` | SHOP-09, SHOP-20 |
| `onContentWillChange` | SHOP-06 |
| `onFinish` | SHOP-17 |
| `onFrame` | SHOP-17 |
| `onOffsetChange` | SHOP-06 |
| `onReachEnd` | SHOP-10 |
| `onReady` | SHOP-07, SHOP-20 |
| `onRefreshing` | SHOP-06, SHOP-10 |
| `onScrollIndex` | SHOP-14 |
| `onShowFileSelector` | SHOP-06 |
| `onStateChange` | SHOP-06 |
| `onSubmit` | SHOP-20 |
| `onTimer` | SHOP-04 |
| `onTouch` | SHOP-07, SHOP-22 |
| `onWillScroll` | SHOP-10, SHOP-14 |
| `opacity` | SHOP-22 |
| `openCustomDialog` | SHOP-04 |
| `parallelGesture` | SHOP-08 |
| `pasteboard.createData` | SHOP-23 |
| `pasteboard.getSystemPasteboard` | SHOP-23 |
| `photoAccessHelper` | SHOP-13 |
| `photoAccessHelper.PhotoViewPicker` | SHOP-01, SHOP-06 |
| `picker.DocumentViewPicker` | SHOP-06 |
| `pop` | SHOP-20 |
| `position` | SHOP-17, SHOP-22 |
| `preferences.getPreferences` | SHOP-04 |
| `preferences.getPreferencesSync` | SHOP-05 |
| `pullToRefresh` | SHOP-06, SHOP-10 |
| `pushPath` | SHOP-20 |
| `pushPathByName` | SHOP-04, SHOP-08, SHOP-15, SHOP-21 |
| `put` | SHOP-04 |
| `putSync` | SHOP-05 |
| `px2vp` | SHOP-22 |
| `refreshOffset` | SHOP-06 |
| `requestEnableNotification` | SHOP-04 |
| `requestPermissionOnSetting` | SHOP-23 |
| `requestPermissionsFromUser` | SHOP-16, SHOP-23 |
| `resourceManager.getMediaContentSync` | SHOP-07 |
| `resourceManager.getRawFileContent` | SHOP-03 |
| `rotate` | SHOP-22 |
| `routerMap` | SHOP-01, SHOP-20, SHOP-21 |
| `rowsTemplate` | SHOP-10 |
| `safeAreaPadding` | SHOP-05 |
| `scale` | SHOP-17, SHOP-22 |
| `scrollSnapAlign` | SHOP-14 |
| `searchButton` | SHOP-12 |
| `searchIcon` | SHOP-12 |
| `setInterval` | SHOP-16, SHOP-17, SHOP-19 |
| `setTimeout` | SHOP-11, SHOP-19 |
| `setWindowLayoutFullScreen` | SHOP-03, SHOP-17, SHOP-18, SHOP-19 |
| `showToast` | SHOP-03, SHOP-05, SHOP-08, SHOP-18, SHOP-21 |
| `tabBar` | SHOP-09 |
| `translate` | SHOP-11, SHOP-19 |
| `util` | SHOP-13 |
| `visibility` | SHOP-12, SHOP-20 |
| `window` | SHOP-12, SHOP-13, SHOP-14, SHOP-15, SHOP-16, SHOP-21, SHOP-22 |
| `window.getLastWindow` | SHOP-17 |
| `window.getWindowAvoidArea` | SHOP-03, SHOP-05, SHOP-06, SHOP-09, SHOP-11, SHOP-18, SHOP-19, SHOP-20 |
| `window.on('avoidAreaChange')` | SHOP-20 |
| `wrapBuilder` | SHOP-04 |
