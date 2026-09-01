# API map

> Generated from `features/*.md`. Source industry: `08_children_education`, 17 features.
> Do not edit by hand; regenerate it in the review repository.

## Feature to kits, permissions and API level

| ID | Kits | Permissions | Min API | Modules |
|---|---|---|---:|---|
| `KIDS-01` | - | - | 20 | - |
| `KIDS-02` | @kit.ArkUI, @kit.AbilityKit, @kit.CryptoArchitectureKit, @kit.PerformanceAnalysisKit, @kit.BasicServicesKit | - | 20 | entry |
| `KIDS-03` | @kit.ArkUI, @kit.CoreFileKit, @kit.ArkTS, @kit.AbilityKit | - | 20 | entry |
| `KIDS-04` | @kit.ArkUI, @kit.ArkData, @kit.AbilityKit, @kit.PerformanceAnalysisKit, @kit.BasicServicesKit | - | 20 | entry |
| `KIDS-05` | @kit.ArkUI | - | 20 | entry |
| `KIDS-06` | @kit.ArkUI, @kit.AbilityKit, @kit.PerformanceAnalysisKit, @kit.BasicServicesKit | - | 20 | entry |
| `KIDS-07` | @kit.ArkUI | - | 20 | entry |
| `KIDS-08` | @kit.ArkUI, @kit.AbilityKit | - | 20 | entry |
| `KIDS-09` | @kit.ArkUI | - | 20 | entry |
| `KIDS-10` | @kit.ArkUI, @kit.CryptoArchitectureKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `KIDS-11` | @kit.NetworkKit, @kit.ArkData, @kit.ArkUI, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `KIDS-12` | @kit.MapKit, @kit.LocationKit, @kit.ArkUI, @kit.BasicServicesKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `KIDS-13` | @kit.MediaKit, @kit.ArkUI, @kit.AbilityKit, @kit.BasicServicesKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `KIDS-14` | @kit.LocationKit, @kit.MapKit, @kit.ArkUI, @kit.AbilityKit, @kit.BackgroundTasksKit, @kit.PerformanceAnalysisKit | ohos.permission.LOCATION, ohos.permission.APPROXIMATELY_LOCATION, ohos.permission.LOCATION_IN_BACKGROUND, ohos.permission.KEEP_BACKGROUND_RUNNING, ohos.permission.GET_NETWORK_INFO | 20 | entry |
| `KIDS-15` | @kit.ArkUI, @kit.CryptoArchitectureKit | - | 20 | entry |
| `KIDS-16` | @kit.CryptoArchitectureKit, @kit.LocalizationKit, @kit.ArkUI, @ohos/gif-drawable | - | 20 | entry |
| `KIDS-17` | - | - | 20 | - |

## Kit to features

| Kit | Features |
|---|---|
| `@kit.AbilityKit` | KIDS-02, KIDS-03, KIDS-04, KIDS-06, KIDS-08, KIDS-13, KIDS-14 |
| `@kit.ArkData` | KIDS-04, KIDS-11 |
| `@kit.ArkTS` | KIDS-03 |
| `@kit.ArkUI` | KIDS-02, KIDS-03, KIDS-04, KIDS-05, KIDS-06, KIDS-07, KIDS-08, KIDS-09, KIDS-10, KIDS-11, KIDS-12, KIDS-13, KIDS-14, KIDS-15, KIDS-16 |
| `@kit.BackgroundTasksKit` | KIDS-14 |
| `@kit.BasicServicesKit` | KIDS-02, KIDS-04, KIDS-06, KIDS-12, KIDS-13 |
| `@kit.CoreFileKit` | KIDS-03 |
| `@kit.CryptoArchitectureKit` | KIDS-02, KIDS-10, KIDS-15, KIDS-16 |
| `@kit.LocalizationKit` | KIDS-16 |
| `@kit.LocationKit` | KIDS-12, KIDS-14 |
| `@kit.MapKit` | KIDS-12, KIDS-14 |
| `@kit.MediaKit` | KIDS-13 |
| `@kit.NetworkKit` | KIDS-11 |
| `@kit.PerformanceAnalysisKit` | KIDS-02, KIDS-04, KIDS-06, KIDS-10, KIDS-11, KIDS-12, KIDS-13, KIDS-14 |
| `@ohos/gif-drawable` | KIDS-16 |

## Permission to features

| Permission | Features |
|---|---|
| `ohos.permission.APPROXIMATELY_LOCATION` | KIDS-14 |
| `ohos.permission.GET_NETWORK_INFO` | KIDS-14 |
| `ohos.permission.KEEP_BACKGROUND_RUNNING` | KIDS-14 |
| `ohos.permission.LOCATION` | KIDS-14 |
| `ohos.permission.LOCATION_IN_BACKGROUND` | KIDS-14 |

## API to features

| API | Features |
|---|---|
| `@Consume` | KIDS-02, KIDS-03, KIDS-04, KIDS-07, KIDS-09 |
| `@CustomDialog` | KIDS-02, KIDS-06 |
| `@Link` | KIDS-10, KIDS-13, KIDS-15 |
| `@Observed` | KIDS-06 |
| `@ObservedV2` | KIDS-15 |
| `@Prop` | KIDS-06, KIDS-10, KIDS-13 |
| `@Provide` | KIDS-02, KIDS-03, KIDS-04, KIDS-07, KIDS-09, KIDS-10 |
| `@State` | KIDS-05, KIDS-08, KIDS-11, KIDS-15, KIDS-16 |
| `@StorageLink` | KIDS-02, KIDS-04, KIDS-10 |
| `@StorageProp` | KIDS-06, KIDS-07, KIDS-09, KIDS-10, KIDS-12 |
| `@Trace` | KIDS-15 |
| `@Watch` | KIDS-02, KIDS-10, KIDS-15 |
| `BlurStyle` | KIDS-08 |
| `Canvas` | KIDS-03, KIDS-05, KIDS-07, KIDS-09 |
| `CanvasRenderingContext2D` | KIDS-03, KIDS-05, KIDS-07, KIDS-09 |
| `Circle` | KIDS-07 |
| `ContinuousLocationRequest` | KIDS-14 |
| `CustomDialogController` | KIDS-02, KIDS-06 |
| `DismissReason` | KIDS-06 |
| `Divider` | KIDS-08 |
| `Flex` | KIDS-10 |
| `FlexWrap` | KIDS-10 |
| `ForEach` | KIDS-08, KIDS-10, KIDS-13, KIDS-15 |
| `GIFComponent.ControllerOptions` | KIDS-16 |
| `GIFComponent.ScaleType` | KIDS-16 |
| `GestureEvent` | KIDS-05, KIDS-06 |
| `Grid` | KIDS-02, KIDS-06, KIDS-07, KIDS-15 |
| `GridItem` | KIDS-06, KIDS-07, KIDS-15 |
| `Intl.DateTimeFormat` | KIDS-11 |
| `LoadingProgress` | KIDS-16 |
| `MapCircleOptions` | KIDS-14 |
| `MapComponent` | KIDS-12, KIDS-14 |
| `MeasureUtils.measureTextSize` | KIDS-05 |
| `NavDestination` | KIDS-02, KIDS-03, KIDS-04, KIDS-07, KIDS-09 |
| `NavPathStack` | KIDS-02, KIDS-03, KIDS-04, KIDS-07, KIDS-10 |
| `Navigation` | KIDS-02 |
| `PanDirection` | KIDS-13 |
| `PanGesture` | KIDS-06, KIDS-13 |
| `Path` | KIDS-06 |
| `Path2D` | KIDS-07, KIDS-09 |
| `PopupOptions` | KIDS-08 |
| `Progress` | KIDS-11 |
| `ProgressType` | KIDS-11 |
| `Random.generateRandom` | KIDS-10 |
| `Random.generateRandomSync` | KIDS-02 |
| `Rect` | KIDS-07 |
| `Refresh` | KIDS-11 |
| `RelativeContainer` | KIDS-04 |
| `RenderingContextSettings` | KIDS-03, KIDS-05, KIDS-07, KIDS-09 |
| `ResourceLoader.loadMedia` | KIDS-16 |
| `ReverseGeoCodeRequest` | KIDS-12 |
| `SafeAreaEdge` | KIDS-08 |
| `SafeAreaType` | KIDS-08 |
| `SingleLocationRequest` | KIDS-14 |
| `Slider` | KIDS-07, KIDS-09 |
| `Stack` | KIDS-04, KIDS-06, KIDS-13, KIDS-15 |
| `Swiper` | KIDS-02 |
| `TabContent` | KIDS-02 |
| `Tabs` | KIDS-02 |
| `TapGesture` | KIDS-05 |
| `Text` | KIDS-08 |
| `TextInput` | KIDS-11 |
| `TextPicker` | KIDS-04 |
| `TextTimer` | KIDS-04, KIDS-15 |
| `TextTimerController` | KIDS-04, KIDS-15 |
| `TouchObject` | KIDS-03 |
| `TouchType` | KIDS-03, KIDS-07, KIDS-09 |
| `UIAbilityContext.terminateSelf` | KIDS-08 |
| `UIContext.animateTo` | KIDS-06, KIDS-13 |
| `UIContext.getPromptAction` | KIDS-07, KIDS-16 |
| `UIContext.px2vp` | KIDS-02, KIDS-05, KIDS-07, KIDS-10 |
| `abilityAccessCtrl.requestPermissionsFromUser` | KIDS-14 |
| `addMarker` | KIDS-12 |
| `addTraceOverlay` | KIDS-12 |
| `alignRules` | KIDS-08 |
| `arc` | KIDS-05, KIDS-09 |
| `avPlayer.on('error')` | KIDS-13 |
| `avPlayer.on('stateChange')` | KIDS-13 |
| `backgroundImage` | KIDS-10 |
| `backgroundImagePosition` | KIDS-10 |
| `backgroundTaskManager.startBackgroundRunning` | KIDS-14 |
| `beginPath` | KIDS-03, KIDS-05 |
| `bindPopup` | KIDS-08 |
| `bindSheet` | KIDS-07, KIDS-09, KIDS-10, KIDS-11, KIDS-12 |
| `blur` | KIDS-13 |
| `buffer.from` | KIDS-03 |
| `clearRect` | KIDS-03, KIDS-07, KIDS-09 |
| `clearTimeout` | KIDS-04 |
| `closePath` | KIDS-03, KIDS-09 |
| `columnsGap` | KIDS-15 |
| `columnsTemplate` | KIDS-15 |
| `commands` | KIDS-06 |
| `cryptoFramework.createRandom` | KIDS-02, KIDS-10, KIDS-15, KIDS-16 |
| `deleteSync` | KIDS-04, KIDS-11 |
| `destroy` | KIDS-16 |
| `detents` | KIDS-12 |
| `display.getAllDisplays` | KIDS-05 |
| `expandSafeArea` | KIDS-08 |
| `fdSrc` | KIDS-13 |
| `fileIo.closeSync` | KIDS-03 |
| `fileIo.openSync` | KIDS-03 |
| `fileIo.writeSync` | KIDS-03 |
| `fill` | KIDS-05, KIDS-09 |
| `fillRect` | KIDS-09 |
| `fillStyle` | KIDS-09 |
| `fillText` | KIDS-05 |
| `fingerList` | KIDS-05 |
| `flush` | KIDS-04 |
| `flushSync` | KIDS-11 |
| `font` | KIDS-05 |
| `generateRandomSync` | KIDS-15, KIDS-16 |
| `geoLocationManager.getAddressesFromLocation` | KIDS-12 |
| `geoLocationManager.off('locationChange')` | KIDS-14 |
| `geoLocationManager.on('locationChange')` | KIDS-14 |
| `getCurrentLocation` | KIDS-14 |
| `getSync` | KIDS-04, KIDS-11 |
| `getWindowProperties` | KIDS-06 |
| `isCountDown` | KIDS-04 |
| `letterSpacing` | KIDS-08 |
| `lineCap` | KIDS-07 |
| `lineTo` | KIDS-03, KIDS-05, KIDS-09 |
| `lineWidth` | KIDS-07, KIDS-09 |
| `loadBuffer` | KIDS-16 |
| `map.MapComponentController` | KIDS-12 |
| `map.PlayImageAnimation` | KIDS-12 |
| `map.calculateDistance` | KIDS-14 |
| `map.convertCoordinate` | KIDS-14 |
| `mapCommon.MapOptions` | KIDS-12 |
| `mapCommon.MarkerOptions` | KIDS-12 |
| `mapCommon.TraceOverlayParams` | KIDS-12 |
| `media.AVFileDescriptor` | KIDS-13 |
| `media.AVPlayer` | KIDS-13 |
| `media.createAVPlayer` | KIDS-13 |
| `messageOptions` | KIDS-08 |
| `moveTo` | KIDS-03, KIDS-05, KIDS-09 |
| `offset` | KIDS-13 |
| `onActionEnd` | KIDS-06 |
| `onActionStart` | KIDS-06 |
| `onActionUpdate` | KIDS-06 |
| `onAreaChange` | KIDS-03 |
| `onReady` | KIDS-03, KIDS-05 |
| `onRefreshing` | KIDS-11 |
| `onStateChange` | KIDS-08 |
| `onTimer` | KIDS-04 |
| `onTouch` | KIDS-03, KIDS-07, KIDS-09 |
| `onVisibleAreaChange` | KIDS-04 |
| `onWillDismiss` | KIDS-12 |
| `pause` | KIDS-13 |
| `play` | KIDS-13 |
| `position` | KIDS-06 |
| `preferences.getPreferencesSync` | KIDS-04, KIDS-11 |
| `prepare` | KIDS-13 |
| `pushPathByName` | KIDS-02, KIDS-03, KIDS-04 |
| `put` | KIDS-04 |
| `putSync` | KIDS-11 |
| `rect` | KIDS-09 |
| `release` | KIDS-13 |
| `reset` | KIDS-13 |
| `resourceManager.getRawFdSync` | KIDS-13 |
| `rotate` | KIDS-06 |
| `rowsGap` | KIDS-15 |
| `rowsTemplate` | KIDS-15 |
| `safeAreaPadding` | KIDS-06, KIDS-13 |
| `setAnimation` | KIDS-12 |
| `setInfoWindowVisible` | KIDS-14 |
| `setInterval` | KIDS-12 |
| `setLoopFinish` | KIDS-16 |
| `setMyLocationStyle` | KIDS-14 |
| `setOpenHardware` | KIDS-16 |
| `setScaleType` | KIDS-16 |
| `setSpeedFactor` | KIDS-16 |
| `setTimeout` | KIDS-04 |
| `showToast` | KIDS-07, KIDS-14, KIDS-16 |
| `startAnimation` | KIDS-12 |
| `statistics.getIfaceRxBytes` | KIDS-11 |
| `statistics.getIfaceTxBytes` | KIDS-11 |
| `stop` | KIDS-13 |
| `stroke` | KIDS-03, KIDS-05, KIDS-07, KIDS-09 |
| `strokeRect` | KIDS-05 |
| `strokeStyle` | KIDS-07, KIDS-09 |
| `textAlign` | KIDS-05 |
| `textBaseline` | KIDS-05 |
| `toDataURL` | KIDS-03 |
| `visibility` | KIDS-10 |
| `window.getLastWindow` | KIDS-06 |
| `window.getWindowAvoidArea` | KIDS-02 |
| `window.on('avoidAreaChange')` | KIDS-02 |
| `window.setSpecificSystemBarEnabled` | KIDS-03 |
| `window.setWindowLayoutFullScreen` | KIDS-02, KIDS-03 |
| `zIndex` | KIDS-06, KIDS-13 |
