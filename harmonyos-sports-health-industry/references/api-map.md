# API map

> Generated from `features/*.md`. Source industry: `03_sports_health`, 15 features.
> Do not edit by hand; regenerate it in the review repository.

## Feature to kits, permissions and API level

| ID | Kits | Permissions | Min API | Modules |
|---|---|---|---:|---|
| `SPORT-01` | @kit.ConnectivityKit, @kit.SensorServiceKit, @kit.AbilityKit, @kit.NotificationKit, @kit.ArkData, @kit.ArkTS, @kit.CryptoArchitectureKit, @kit.BasicServicesKit | ohos.permission.ACCESS_BLUETOOTH, ohos.permission.ACTIVITY_MOTION | 20 | phone, smartscreen, common, first, run, find, shop, mine, weightscale |
| `SPORT-02` | - | - | 20 | - |
| `SPORT-03` | @kit.ArkUI, @kit.ArkTS | - | 20 | entry |
| `SPORT-04` | @kit.CalendarKit, @kit.ArkData, @kit.AbilityKit, @kit.ArkUI, @kit.PerformanceAnalysisKit, @kit.BasicServicesKit | ohos.permission.READ_CALENDAR, ohos.permission.WRITE_CALENDAR | 20 | entry |
| `SPORT-05` | @kit.LocalizationKit, @kit.ArkTS, @kit.AbilityKit, @kit.ArkUI | - | 20 | entry |
| `SPORT-06` | @kit.ArkUI | - | 20 | entry |
| `SPORT-07` | @kit.ArkUI | - | 20 | entry |
| `SPORT-08` | @kit.MapKit, @kit.BasicServicesKit, @kit.ArkUI, @kit.PerformanceAnalysisKit | ohos.permission.INTERNET, ohos.permission.GET_NETWORK_INFO | 20 | entry |
| `SPORT-09` | @kit.ArkUI, @kit.MediaLibraryKit, @kit.PerformanceAnalysisKit, @kit.BasicServicesKit | ohos.permission.INTERNET | 20 | entry |
| `SPORT-10` | @kit.ArkUI, @kit.CryptoArchitectureKit | - | 20 | entry |
| `SPORT-11` | @kit.LocationKit, @kit.MapKit, @kit.AbilityKit, @kit.ArkUI | ohos.permission.LOCATION, ohos.permission.APPROXIMATELY_LOCATION, ohos.permission.INTERNET, ohos.permission.GET_NETWORK_INFO | 20 | entry |
| `SPORT-12` | @kit.ArkUI | - | 20 | entry |
| `SPORT-13` | @kit.ArkUI, @ohos.curves | - | 20 | entry |
| `SPORT-14` | @kit.ScanKit, @kit.ArkUI, @kit.LocalizationKit, @kit.BasicServicesKit | - | 20 | entry |
| `SPORT-15` | - | - | 20 | - |

## Kit to features

| Kit | Features |
|---|---|
| `@kit.AbilityKit` | SPORT-01, SPORT-04, SPORT-05, SPORT-11 |
| `@kit.ArkData` | SPORT-01, SPORT-04 |
| `@kit.ArkTS` | SPORT-01, SPORT-03, SPORT-05 |
| `@kit.ArkUI` | SPORT-03, SPORT-04, SPORT-05, SPORT-06, SPORT-07, SPORT-08, SPORT-09, SPORT-10, SPORT-11, SPORT-12, SPORT-13, SPORT-14 |
| `@kit.BasicServicesKit` | SPORT-01, SPORT-04, SPORT-08, SPORT-09, SPORT-14 |
| `@kit.CalendarKit` | SPORT-04 |
| `@kit.ConnectivityKit` | SPORT-01 |
| `@kit.CryptoArchitectureKit` | SPORT-01, SPORT-10 |
| `@kit.LocalizationKit` | SPORT-05, SPORT-14 |
| `@kit.LocationKit` | SPORT-11 |
| `@kit.MapKit` | SPORT-08, SPORT-11 |
| `@kit.MediaLibraryKit` | SPORT-09 |
| `@kit.NotificationKit` | SPORT-01 |
| `@kit.PerformanceAnalysisKit` | SPORT-04, SPORT-08, SPORT-09 |
| `@kit.ScanKit` | SPORT-14 |
| `@kit.SensorServiceKit` | SPORT-01 |
| `@ohos.curves` | SPORT-13 |

## Permission to features

| Permission | Features |
|---|---|
| `ohos.permission.ACCESS_BLUETOOTH` | SPORT-01 |
| `ohos.permission.ACTIVITY_MOTION` | SPORT-01 |
| `ohos.permission.APPROXIMATELY_LOCATION` | SPORT-11 |
| `ohos.permission.GET_NETWORK_INFO` | SPORT-08, SPORT-11 |
| `ohos.permission.INTERNET` | SPORT-08, SPORT-09, SPORT-11 |
| `ohos.permission.LOCATION` | SPORT-11 |
| `ohos.permission.READ_CALENDAR` | SPORT-04 |
| `ohos.permission.WRITE_CALENDAR` | SPORT-04 |

## API to features

| API | Features |
|---|---|
| `@Builder` | SPORT-06 |
| `@Consume` | SPORT-01, SPORT-08 |
| `@Link` | SPORT-05, SPORT-06, SPORT-12, SPORT-13 |
| `@Prop` | SPORT-04, SPORT-05, SPORT-10 |
| `@State` | SPORT-03, SPORT-04, SPORT-05, SPORT-06, SPORT-07, SPORT-10, SPORT-12, SPORT-13, SPORT-14 |
| `@StorageLink` | SPORT-01, SPORT-09 |
| `@StorageProp` | SPORT-03, SPORT-05, SPORT-07, SPORT-08, SPORT-09, SPORT-12 |
| `@Watch` | SPORT-05 |
| `@ohos/mpchart` | SPORT-05 |
| `Alignment` | SPORT-07 |
| `AnimateParam` | SPORT-12 |
| `Calendar.addEvent` | SPORT-04 |
| `CalendarManager.getCalendar` | SPORT-04 |
| `Column` | SPORT-10 |
| `Curve` | SPORT-12 |
| `ForEach` | SPORT-09, SPORT-10, SPORT-13, SPORT-14 |
| `GattClientDevice.close` | SPORT-01 |
| `GattClientDevice.connect` | SPORT-01 |
| `GattClientDevice.disconnect` | SPORT-01 |
| `GattClientDevice.on('BLEConnectionStateChange')` | SPORT-01 |
| `GestureEvent` | SPORT-03, SPORT-13 |
| `GestureGroup` | SPORT-13 |
| `GestureMode` | SPORT-13 |
| `Grid` | SPORT-13 |
| `GridCol` | SPORT-04 |
| `GridItem` | SPORT-13 |
| `GridRow` | SPORT-04 |
| `HitTestMode` | SPORT-03 |
| `IAxisValueFormatter` | SPORT-05 |
| `IValueFormatter` | SPORT-05 |
| `ImageInterpolation` | SPORT-06 |
| `Intl.DateTimeFormat` | SPORT-05 |
| `Length` | SPORT-12 |
| `LengthMetrics` | SPORT-09 |
| `LineChartModel` | SPORT-05 |
| `LineData` | SPORT-05 |
| `LineDataSet` | SPORT-05 |
| `List` | SPORT-09, SPORT-14 |
| `LocationRequestPriority` | SPORT-11 |
| `LocationRequestScenario` | SPORT-11 |
| `LongPressGesture` | SPORT-03, SPORT-13 |
| `MapComponent` | SPORT-08, SPORT-11 |
| `MixedMode` | SPORT-09 |
| `Mode` | SPORT-05 |
| `NavPathStack` | SPORT-01, SPORT-08, SPORT-09, SPORT-13 |
| `PanGesture` | SPORT-13 |
| `Path` | SPORT-03 |
| `PhotoSelectOptions` | SPORT-09 |
| `PlayMode` | SPORT-12 |
| `Polyline` | SPORT-10 |
| `Progress` | SPORT-07 |
| `ProgressStyleOptions` | SPORT-07 |
| `ProgressType` | SPORT-07 |
| `RdbStore.batchInsert` | SPORT-04 |
| `RdbStore.execute` | SPORT-04 |
| `RdbStore.insert` | SPORT-04 |
| `RdbStore.query` | SPORT-04 |
| `ResourceColor` | SPORT-12 |
| `RichEditor` | SPORT-09 |
| `RichEditorController` | SPORT-09 |
| `RichEditorImageSpanResult` | SPORT-09 |
| `RichEditorOptions` | SPORT-09 |
| `RichEditorTextSpanResult` | SPORT-09 |
| `Row` | SPORT-10 |
| `SheetSize` | SPORT-14 |
| `SingleLocationRequest` | SPORT-11 |
| `Stack` | SPORT-07 |
| `TabContent` | SPORT-10 |
| `Tabs` | SPORT-10 |
| `TextInput` | SPORT-06 |
| `TextTimer` | SPORT-03, SPORT-06 |
| `TextTimerController` | SPORT-03, SPORT-06 |
| `TouchEvent` | SPORT-06 |
| `TouchType` | SPORT-06 |
| `UIAbility.onConfigurationUpdated` | SPORT-05 |
| `UIContext.animateTo` | SPORT-03, SPORT-12, SPORT-13 |
| `UIContext.getHostContext` | SPORT-14 |
| `UIContext.getPromptAction` | SPORT-14 |
| `UIContext.px2vp` | SPORT-03 |
| `UIContext.vp2px` | SPORT-03 |
| `Web` | SPORT-09 |
| `WebLayoutMode` | SPORT-09 |
| `WebviewController` | SPORT-09 |
| `XAxis` | SPORT-05 |
| `YAxis` | SPORT-05 |
| `abilityAccessCtrl.requestPermissionsFromUser` | SPORT-01, SPORT-04, SPORT-11 |
| `addMarker` | SPORT-08 |
| `addTraceOverlay` | SPORT-08, SPORT-11 |
| `animateCamera` | SPORT-11 |
| `animation attribute` | SPORT-03 |
| `animationCallback` | SPORT-08 |
| `aspectRatio` | SPORT-12 |
| `bindSheet` | SPORT-04, SPORT-09, SPORT-14 |
| `ble.MatchMode` | SPORT-01 |
| `ble.ScanDuty` | SPORT-01 |
| `ble.ScanOptions` | SPORT-01 |
| `ble.createGattClientDevice` | SPORT-01 |
| `ble.off('BLEDeviceFind')` | SPORT-01 |
| `ble.on('BLEDeviceFind')` | SPORT-01 |
| `ble.startBLEScan` | SPORT-01 |
| `ble.stopBLEScan` | SPORT-01 |
| `calendarManager.Event` | SPORT-04 |
| `calendarManager.getCalendarManager` | SPORT-04 |
| `clearInterval` | SPORT-03 |
| `columnsTemplate` | SPORT-13 |
| `commands` | SPORT-03 |
| `constant.ProfileConnectionState` | SPORT-01 |
| `count` | SPORT-06 |
| `cryptoFramework.createRandom` | SPORT-10 |
| `curves.interpolatingSpring` | SPORT-13 |
| `fadingEdge` | SPORT-09 |
| `fileAccess` | SPORT-09 |
| `format` | SPORT-06 |
| `geoLocationManager.LocationRequest` | SPORT-11 |
| `geoLocationManager.getCurrentLocation` | SPORT-11 |
| `geoLocationManager.isLocationEnabled` | SPORT-11 |
| `geoLocationManager.off('locationChange')` | SPORT-11 |
| `geoLocationManager.on('locationChange')` | SPORT-11 |
| `getParamByName` | SPORT-08 |
| `getSpans` | SPORT-09 |
| `hitTestBehavior` | SPORT-03 |
| `isCountDown` | SPORT-06 |
| `javaScriptAccess` | SPORT-09 |
| `loadData` | SPORT-09 |
| `map.MapComponentController` | SPORT-08 |
| `map.MapEventManager` | SPORT-08 |
| `map.convertCoordinateSync` | SPORT-11 |
| `map.newLatLng` | SPORT-11 |
| `map.newLatLngBounds` | SPORT-08 |
| `mapCommon.LatLngBounds` | SPORT-08 |
| `mapCommon.MapType` | SPORT-08 |
| `mapCommon.TraceOverlayParams` | SPORT-08, SPORT-11 |
| `metaViewport` | SPORT-09 |
| `mixedMode` | SPORT-09 |
| `moveCamera` | SPORT-08 |
| `notificationManager.requestEnableNotification` | SPORT-01 |
| `offset` | SPORT-07 |
| `onAction` | SPORT-03 |
| `onActionEnd` | SPORT-03, SPORT-13 |
| `onActionStart` | SPORT-13 |
| `onActionUpdate` | SPORT-13 |
| `onAppear` | SPORT-12 |
| `onFinish` | SPORT-12 |
| `onSubmit` | SPORT-06 |
| `onTimer` | SPORT-06 |
| `onTouch` | SPORT-06 |
| `pause` | SPORT-06 |
| `photoAccessHelper.PhotoViewPicker` | SPORT-09 |
| `points` | SPORT-10 |
| `position` | SPORT-03 |
| `preferences` | SPORT-01 |
| `relationalStore.ConflictResolution` | SPORT-04 |
| `relationalStore.RdbPredicates` | SPORT-04 |
| `relationalStore.SecurityLevel` | SPORT-04 |
| `relationalStore.StoreConfig` | SPORT-04 |
| `relationalStore.getRdbStore` | SPORT-04 |
| `reset` | SPORT-06 |
| `resourceManager.getRawFileContentSync` | SPORT-05 |
| `resourceManager.getStringSync` | SPORT-14 |
| `scale` | SPORT-13 |
| `scaleControlsEnabled` | SPORT-08 |
| `scanBarcode.ScanOptions` | SPORT-14 |
| `scanBarcode.ScanResult` | SPORT-14 |
| `scanBarcode.startScanForResult` | SPORT-14 |
| `scanCore.ScanType` | SPORT-14 |
| `sensor.PedometerResponse` | SPORT-01 |
| `sensor.off` | SPORT-01 |
| `sensor.on(SensorId.PEDOMETER)` | SPORT-01 |
| `setInterval` | SPORT-03 |
| `setTypingStyle` | SPORT-09 |
| `showToast` | SPORT-14 |
| `start` | SPORT-06 |
| `stroke` | SPORT-10 |
| `strokeWidth` | SPORT-07, SPORT-10 |
| `translate` | SPORT-13 |
| `util.TextDecoder` | SPORT-05 |
| `util.format` | SPORT-03 |
| `zIndex` | SPORT-03, SPORT-13 |
