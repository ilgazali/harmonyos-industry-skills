# API map

> Generated from `features/*.md`. Source industry: `06_public_transport`, 9 features.
> Do not edit by hand; regenerate it in the review repository.

## Feature to kits, permissions and API level

| ID | Kits | Permissions | Min API | Modules |
|---|---|---|---:|---|
| `TRANS-01` | @kit.MapKit, @kit.LocationKit, @kit.AbilityKit, @kit.ArkData, @kit.ArkUI, @kit.BasicServicesKit | ohos.permission.INTERNET, ohos.permission.LOCATION, ohos.permission.APPROXIMATELY_LOCATION | 24 | product/phone, common, features/account, features/home, features/pay, features/personal, features/travel, rideCode/cityCode, rideCode/otherCityCode |
| `TRANS-02` | - | - | 24 | - |
| `TRANS-03` | @kit.ArkUI, @kit.AbilityKit, @kit.FormKit, @kit.PerformanceAnalysisKit, @kit.BasicServicesKit | - | 20 | entry |
| `TRANS-04` | @kit.LocationKit, @kit.AbilityKit, @kit.BasicServicesKit, @kit.ArkUI | ohos.permission.APPROXIMATELY_LOCATION | 20 | entry |
| `TRANS-05` | @kit.ArkUI | - | 20 | entry |
| `TRANS-06` | @kit.StoreKit, @kit.AbilityKit, @kit.ArkUI | - | 20 | entry |
| `TRANS-07` | @kit.NotificationKit, @kit.ArkUI, @kit.BasicServicesKit | - | 20 | entry |
| `TRANS-08` | @kit.MapKit, @kit.LocationKit, @kit.AbilityKit, @kit.ArkUI, @kit.BasicServicesKit | ohos.permission.LOCATION, ohos.permission.APPROXIMATELY_LOCATION | 20 | entry |
| `TRANS-09` | - | - | 24 | - |

## Kit to features

| Kit | Features |
|---|---|
| `@kit.AbilityKit` | TRANS-01, TRANS-03, TRANS-04, TRANS-06, TRANS-08 |
| `@kit.ArkData` | TRANS-01 |
| `@kit.ArkUI` | TRANS-01, TRANS-03, TRANS-04, TRANS-05, TRANS-06, TRANS-07, TRANS-08 |
| `@kit.BasicServicesKit` | TRANS-01, TRANS-03, TRANS-04, TRANS-07, TRANS-08 |
| `@kit.FormKit` | TRANS-03 |
| `@kit.LocationKit` | TRANS-01, TRANS-04, TRANS-08 |
| `@kit.MapKit` | TRANS-01, TRANS-08 |
| `@kit.NotificationKit` | TRANS-07 |
| `@kit.PerformanceAnalysisKit` | TRANS-03 |
| `@kit.StoreKit` | TRANS-06 |

## Permission to features

| Permission | Features |
|---|---|
| `ohos.permission.APPROXIMATELY_LOCATION` | TRANS-01, TRANS-04, TRANS-08 |
| `ohos.permission.INTERNET` | TRANS-01 |
| `ohos.permission.LOCATION` | TRANS-01, TRANS-08 |

## API to features

| API | Features |
|---|---|
| `@Link` | TRANS-07 |
| `@ObservedV2` | TRANS-05 |
| `@StorageLink` | TRANS-06 |
| `@Trace` | TRANS-05 |
| `@Watch` | TRANS-06 |
| `BorderRadiuses` | TRANS-05 |
| `CalendarPicker` | TRANS-05 |
| `ConfigurationConstant.ColorMode` | TRANS-03 |
| `ForEach` | TRANS-05 |
| `FormExtensionAbility` | TRANS-03 |
| `FormLink` | TRANS-03 |
| `List` | TRANS-05 |
| `ListItem` | TRANS-05 |
| `MapComponent` | TRANS-01, TRANS-08 |
| `Matrix4` | TRANS-06 |
| `NavPathStack.pushPathByName` | TRANS-06 |
| `Slider` | TRANS-07 |
| `Tabs` | TRANS-01 |
| `TapGesture` | TRANS-06 |
| `Toggle` | TRANS-08 |
| `UIAbility.onCreate` | TRANS-03 |
| `UIAbility.onNewWant` | TRANS-03 |
| `Want.parameters` | TRANS-03, TRANS-06 |
| `Window.on avoidAreaChange` | TRANS-03 |
| `WindowStage.loadContent` | TRANS-03 |
| `abilityAccessCtrl.AtManager` | TRANS-01 |
| `abilityAccessCtrl.requestPermissionOnSetting` | TRANS-04, TRANS-08 |
| `abilityAccessCtrl.requestPermissionsFromUser` | TRANS-04, TRANS-08 |
| `addMarker` | TRANS-01 |
| `addPolyline` | TRANS-01 |
| `clearInterval` | TRANS-07 |
| `form_config.json` | TRANS-03 |
| `geoLocationManager.LocatingPriority` | TRANS-04 |
| `geoLocationManager.SingleLocationRequest` | TRANS-04 |
| `geoLocationManager.getAddressesFromLocation` | TRANS-01, TRANS-04 |
| `geoLocationManager.getCurrentLocation` | TRANS-01, TRANS-04, TRANS-08 |
| `geoLocationManager.isGeocoderAvailable` | TRANS-01, TRANS-04 |
| `geoLocationManager.off` | TRANS-01 |
| `geoLocationManager.on locationChange` | TRANS-01 |
| `getCameraPosition` | TRANS-08 |
| `map.MapComponentController` | TRANS-01, TRANS-08 |
| `map.convertCoordinate` | TRANS-08 |
| `map.newCameraPosition` | TRANS-08 |
| `mapCommon.CameraPosition` | TRANS-08 |
| `moveCamera` | TRANS-08 |
| `navi.getCyclingRoutes` | TRANS-01 |
| `navi.getDrivingRoutes` | TRANS-01 |
| `navi.getWalkingRoutes` | TRANS-01 |
| `notificationManager.ContentType` | TRANS-07 |
| `notificationManager.NotificationRequest` | TRANS-07 |
| `notificationManager.SlotType` | TRANS-07 |
| `notificationManager.isNotificationEnabled` | TRANS-07 |
| `notificationManager.publish` | TRANS-07 |
| `notificationManager.requestEnableNotification` | TRANS-07 |
| `preferences.getPreferencesSync` | TRANS-01 |
| `productViewManager.CheckShortcutResult` | TRANS-06 |
| `productViewManager.checkPinShortcutPermitted` | TRANS-06 |
| `productViewManager.requestNewPinShortcut` | TRANS-06 |
| `promptAction.showToast` | TRANS-05 |
| `setInterval` | TRANS-07 |
| `setRotateGesturesEnabled` | TRANS-08 |
| `shortcuts_config.json` | TRANS-06 |
| `site.searchByText` | TRANS-01 |
