# API map

> Generated from `features/*.md`. Source industry: `01_auto`, 8 features.
> Do not edit by hand; regenerate it in the review repository.

## Feature to kits, permissions and API level

| ID | Kits | Permissions | Min API | Modules |
|---|---|---|---:|---|
| `AUTO-01` | @kit.ArkUI, @kit.MapKit, @kit.ScanKit, @kit.LocationKit, @kit.AbilityKit, @kit.BasicServicesKit, @kit.ArkData | ohos.permission.LOCATION, ohos.permission.APPROXIMATELY_LOCATION | 20 | entry, common, features/explore, features/buyingCar, features/shoppingMall, features/service, features/mine |
| `AUTO-02` | - | - | 20 | - |
| `AUTO-03` | @kit.ArkUI, @kit.AbilityKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `AUTO-04` | @kit.MapKit, @kit.LocationKit, @kit.AbilityKit, @kit.BasicServicesKit, @kit.ArkUI | ohos.permission.LOCATION, ohos.permission.APPROXIMATELY_LOCATION, ohos.permission.INTERNET | 20 | entry |
| `AUTO-05` | @kit.TelephonyKit, @kit.AbilityKit, @kit.BasicServicesKit, @kit.ArkUI, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `AUTO-06` | @kit.ArkUI, @kit.ArkTS | - | 20 | entry |
| `AUTO-07` | @kit.AbilityKit, @kit.ArkUI | - | 24 | entry, features/vehicleKeyboard |
| `AUTO-08` | - | - | 20 | - |

## Kit to features

| Kit | Features |
|---|---|
| `@kit.AbilityKit` | AUTO-01, AUTO-03, AUTO-04, AUTO-05, AUTO-07 |
| `@kit.ArkData` | AUTO-01 |
| `@kit.ArkTS` | AUTO-06 |
| `@kit.ArkUI` | AUTO-01, AUTO-03, AUTO-04, AUTO-05, AUTO-06, AUTO-07 |
| `@kit.BasicServicesKit` | AUTO-01, AUTO-04, AUTO-05 |
| `@kit.LocationKit` | AUTO-01, AUTO-04 |
| `@kit.MapKit` | AUTO-01, AUTO-04 |
| `@kit.PerformanceAnalysisKit` | AUTO-03, AUTO-05 |
| `@kit.ScanKit` | AUTO-01 |
| `@kit.TelephonyKit` | AUTO-05 |

## Permission to features

| Permission | Features |
|---|---|
| `ohos.permission.APPROXIMATELY_LOCATION` | AUTO-01, AUTO-04 |
| `ohos.permission.INTERNET` | AUTO-04 |
| `ohos.permission.LOCATION` | AUTO-01, AUTO-04 |

## API to features

| API | Features |
|---|---|
| `@Consume` | AUTO-05 |
| `@Extend` | AUTO-07 |
| `@ObjectLink` | AUTO-07 |
| `@Observed` | AUTO-07 |
| `@Provide` | AUTO-05 |
| `@StorageLink` | AUTO-05 |
| `AutoSaveCallback` | AUTO-07 |
| `Canvas` | AUTO-06 |
| `CanvasRenderingContext2D` | AUTO-06 |
| `ContentType.ID_CARD_NUMBER` | AUTO-07 |
| `ContentType.PERSON_FULL_NAME` | AUTO-07 |
| `ContentType.PHONE_NUMBER` | AUTO-07 |
| `Decimal` | AUTO-06 |
| `ImageBitmap` | AUTO-06 |
| `LengthMetrics` | AUTO-07 |
| `MapComponent` | AUTO-01, AUTO-04 |
| `NavPathStack` | AUTO-01 |
| `Navigation` | AUTO-01 |
| `PromptAction.showToast` | AUTO-05, AUTO-07 |
| `RenderingContextSettings` | AUTO-06 |
| `TextInput` | AUTO-07 |
| `Window.destroyWindow` | AUTO-03 |
| `Window.getWindowAvoidArea` | AUTO-03 |
| `Window.moveWindowTo` | AUTO-03 |
| `Window.on avoidAreaChange` | AUTO-03 |
| `Window.resize` | AUTO-03 |
| `Window.setUIContent` | AUTO-03 |
| `Window.setWindowBackgroundColor` | AUTO-03 |
| `Window.showWindow` | AUTO-03 |
| `WindowStage.createSubWindow` | AUTO-03 |
| `abilityAccessCtrl.AtManager` | AUTO-01 |
| `addMarker` | AUTO-04 |
| `animateCamera` | AUTO-04 |
| `arc` | AUTO-06 |
| `autoFillManager.requestAutoSave` | AUTO-07 |
| `beginPath` | AUTO-06 |
| `bindSheet` | AUTO-04, AUTO-05 |
| `bundleManager.getBundleInfoForSelf` | AUTO-01 |
| `call.makeCall` | AUTO-05 |
| `clearRect` | AUTO-06 |
| `contentType` | AUTO-07 |
| `createConicGradient` | AUTO-06 |
| `createRadialGradient` | AUTO-06 |
| `display.getDefaultDisplaySync` | AUTO-03 |
| `drawImage` | AUTO-06 |
| `fill` | AUTO-06 |
| `fillText` | AUTO-06 |
| `geoLocationManager.getCurrentLocation` | AUTO-01, AUTO-04 |
| `getComponentUtils` | AUTO-06 |
| `getRectangleById` | AUTO-06 |
| `map.MapComponentController` | AUTO-01, AUTO-04 |
| `map.Marker` | AUTO-04 |
| `map.ScaleAnimation` | AUTO-04 |
| `map.calculateDistance` | AUTO-04 |
| `map.convertCoordinate` | AUTO-01, AUTO-04 |
| `map.newCameraPosition` | AUTO-04 |
| `map.newLatLng` | AUTO-01 |
| `mapCommon.MapOptions` | AUTO-01, AUTO-04 |
| `mapCommon.MarkerOptions` | AUTO-04 |
| `on markerClick` | AUTO-04 |
| `on myLocationButtonClick` | AUTO-04 |
| `onVisibleAreaChange` | AUTO-03 |
| `preferences.getPreferencesSync` | AUTO-01 |
| `resourceManager.getStringSync` | AUTO-05 |
| `scanBarcode.startScanForResult` | AUTO-01 |
| `scanCore.ScanType` | AUTO-01 |
| `setAnimation` | AUTO-04 |
| `setInfoWindowVisible` | AUTO-04 |
| `setMyLocation` | AUTO-04 |
| `setMyLocationEnabled` | AUTO-04 |
| `site.searchByText` | AUTO-01 |
| `startAnimation` | AUTO-04 |
| `stroke` | AUTO-06 |
| `window.getLastWindow` | AUTO-01 |
