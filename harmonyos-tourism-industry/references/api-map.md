# API map

> Generated from `features/*.md`. Source industry: `09_tourism`, 13 features.
> Do not edit by hand; regenerate it in the review repository.

## Feature to kits, permissions and API level

| ID | Kits | Permissions | Min API | Modules |
|---|---|---|---:|---|
| `TOUR-01` | @kit.MapKit, @kit.LocationKit, @kit.AbilityKit, @kit.BasicServicesKit, @kit.ArkData, @kit.ScanKit, @kit.CryptoArchitectureKit, @kit.PerformanceAnalysisKit, @kit.ArkUI | ohos.permission.INTERNET, ohos.permission.APPROXIMATELY_LOCATION, ohos.permission.LOCATION | 24 | phone, common, home, ParkService, MapService, PersonalCenter, AccountCenter |
| `TOUR-02` | - | - | 24 | - |
| `TOUR-03` | @kit.AbilityKit, @kit.PerformanceAnalysisKit, @kit.BasicServicesKit, @kit.ArkUI | ohos.permission.LOCATION, ohos.permission.APPROXIMATELY_LOCATION, ohos.permission.INTERNET | 20 | entry |
| `TOUR-04` | @kit.ArkTS, @kit.PerformanceAnalysisKit, @kit.ArkUI | - | 20 | entry |
| `TOUR-05` | @kit.ArkUI, @ohos.curves | - | 20 | entry |
| `TOUR-06` | @kit.MapKit, @kit.BasicServicesKit, @kit.ArkUI, @kit.AbilityKit, @kit.PerformanceAnalysisKit | ohos.permission.INTERNET, ohos.permission.GET_NETWORK_INFO | 20 | entry |
| `TOUR-07` | @kit.MapKit, @kit.BasicServicesKit, @kit.ArkUI | ohos.permission.INTERNET, ohos.permission.GET_NETWORK_INFO | 20 | entry |
| `TOUR-08` | @kit.MediaLibraryKit, @kit.ArkUI, @kit.BasicServicesKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `TOUR-09` | @kit.MediaKit, @kit.AbilityKit, @kit.BasicServicesKit, @kit.PerformanceAnalysisKit, @kit.ArkTS | - | 20 | entry |
| `TOUR-10` | @kit.ArkTS, @kit.BasicServicesKit, @kit.ArkUI, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `TOUR-11` | @kit.ArkUI | - | 20 | entry |
| `TOUR-12` | @kit.MapKit, @kit.LocationKit, @kit.BasicServicesKit, @kit.ArkUI, @kit.PerformanceAnalysisKit | ohos.permission.INTERNET, ohos.permission.GET_NETWORK_INFO | 20 | entry |
| `TOUR-13` | - | - | 24 | - |

## Kit to features

| Kit | Features |
|---|---|
| `@kit.AbilityKit` | TOUR-01, TOUR-03, TOUR-06, TOUR-09 |
| `@kit.ArkData` | TOUR-01 |
| `@kit.ArkTS` | TOUR-04, TOUR-09, TOUR-10 |
| `@kit.ArkUI` | TOUR-01, TOUR-03, TOUR-04, TOUR-05, TOUR-06, TOUR-07, TOUR-08, TOUR-10, TOUR-11, TOUR-12 |
| `@kit.BasicServicesKit` | TOUR-01, TOUR-03, TOUR-06, TOUR-07, TOUR-08, TOUR-09, TOUR-10, TOUR-12 |
| `@kit.CryptoArchitectureKit` | TOUR-01 |
| `@kit.LocationKit` | TOUR-01, TOUR-12 |
| `@kit.MapKit` | TOUR-01, TOUR-06, TOUR-07, TOUR-12 |
| `@kit.MediaKit` | TOUR-09 |
| `@kit.MediaLibraryKit` | TOUR-08 |
| `@kit.PerformanceAnalysisKit` | TOUR-01, TOUR-03, TOUR-04, TOUR-06, TOUR-08, TOUR-09, TOUR-10, TOUR-12 |
| `@kit.ScanKit` | TOUR-01 |
| `@ohos.curves` | TOUR-05 |

## Permission to features

| Permission | Features |
|---|---|
| `ohos.permission.APPROXIMATELY_LOCATION` | TOUR-01, TOUR-03 |
| `ohos.permission.GET_NETWORK_INFO` | TOUR-06, TOUR-07, TOUR-12 |
| `ohos.permission.INTERNET` | TOUR-01, TOUR-03, TOUR-06, TOUR-07, TOUR-12 |
| `ohos.permission.LOCATION` | TOUR-01, TOUR-03 |

## API to features

| API | Features |
|---|---|
| `$$` | TOUR-11 |
| `@BuilderParam` | TOUR-07 |
| `@ComponentV2` | TOUR-08 |
| `@Consume` | TOUR-01, TOUR-04, TOUR-12 |
| `@Extend` | TOUR-10 |
| `@Link` | TOUR-10 |
| `@Local` | TOUR-08 |
| `@ObjectLink` | TOUR-10 |
| `@Observed` | TOUR-06, TOUR-10 |
| `@ObservedV2` | TOUR-08 |
| `@Prop` | TOUR-06, TOUR-07, TOUR-12 |
| `@Provide` | TOUR-01, TOUR-04, TOUR-12 |
| `@State` | TOUR-10, TOUR-11 |
| `@StorageLink` | TOUR-09 |
| `@StorageProp` | TOUR-03, TOUR-04, TOUR-07, TOUR-09, TOUR-10, TOUR-11 |
| `@Trace` | TOUR-08 |
| `@Track` | TOUR-06 |
| `@Watch` | TOUR-07, TOUR-09, TOUR-10 |
| `AnimateParam` | TOUR-05 |
| `ArrayList` | TOUR-10 |
| `BaseOverlay.remove` | TOUR-06 |
| `Button` | TOUR-11 |
| `CalendarPickerDialog` | TOUR-05 |
| `Checkbox` | TOUR-11 |
| `Date` | TOUR-11 |
| `Divider` | TOUR-11 |
| `Flex` | TOUR-05 |
| `FlexWrap` | TOUR-05 |
| `ForEach` | TOUR-05, TOUR-08, TOUR-10 |
| `Grid` | TOUR-04, TOUR-08 |
| `GridItem` | TOUR-04, TOUR-08 |
| `IDataSource` | TOUR-04 |
| `InputType` | TOUR-11 |
| `Intl.DateTimeFormat` | TOUR-10 |
| `LazyForEach` | TOUR-01, TOUR-04 |
| `List` | TOUR-04, TOUR-10 |
| `ListItemGroup` | TOUR-04 |
| `ListScroller` | TOUR-06 |
| `MapComponent` | TOUR-01, TOUR-06, TOUR-07, TOUR-12 |
| `Marker.getId` | TOUR-12 |
| `NavDestination` | TOUR-04, TOUR-08 |
| `NavDestinationContext` | TOUR-06, TOUR-08 |
| `NavPathStack` | TOUR-01, TOUR-04, TOUR-06, TOUR-08, TOUR-09 |
| `Navigation` | TOUR-04, TOUR-08 |
| `PanGesture` | TOUR-06 |
| `PhotoSelectOptions` | TOUR-08 |
| `PhotoViewMIMETypes` | TOUR-08 |
| `Placement` | TOUR-03 |
| `PopInfo` | TOUR-04 |
| `PopupOptions` | TOUR-03 |
| `RegExp` | TOUR-11 |
| `Scroller` | TOUR-10 |
| `Stack` | TOUR-04, TOUR-05 |
| `TabContent` | TOUR-03 |
| `Tabs` | TOUR-03, TOUR-10 |
| `TabsAnimationEvent` | TOUR-10 |
| `TabsController` | TOUR-10 |
| `TextArea` | TOUR-08 |
| `TextInput` | TOUR-11 |
| `Toggle` | TOUR-08 |
| `ToggleType` | TOUR-08 |
| `UIAbilityContext.startAbility` | TOUR-01 |
| `UIContext.animateTo` | TOUR-05, TOUR-10 |
| `UIContext.getPromptAction` | TOUR-05, TOUR-08, TOUR-11, TOUR-12 |
| `UIContext.runScopedTask` | TOUR-05 |
| `WaterFlow` | TOUR-01 |
| `abilityAccessCtrl.createAtManager` | TOUR-01, TOUR-03 |
| `addMarker` | TOUR-01, TOUR-06, TOUR-07, TOUR-12 |
| `addPolygon` | TOUR-01 |
| `addPolyline` | TOUR-01 |
| `animateCamera` | TOUR-06, TOUR-12 |
| `animation` | TOUR-05 |
| `avPlayer.fdSrc` | TOUR-09 |
| `bindPopup` | TOUR-03 |
| `bundleManager.getBundleInfoForSelf` | TOUR-01, TOUR-03 |
| `changeIndex` | TOUR-10 |
| `checkAccessToken` | TOUR-01, TOUR-03 |
| `clearInterval` | TOUR-09 |
| `columnsTemplate` | TOUR-04 |
| `curves.springMotion` | TOUR-05 |
| `customInfoWindow` | TOUR-07 |
| `dialogShownResults` | TOUR-03 |
| `geoLocationManager.ReverseGeoCodeRequest` | TOUR-12 |
| `geoLocationManager.getAddressesFromLocation` | TOUR-01, TOUR-12 |
| `geoLocationManager.isGeocoderAvailable` | TOUR-01 |
| `geoLocationManager.on('locationChange')` | TOUR-01 |
| `inputFilter` | TOUR-11 |
| `map.MapComponentController` | TOUR-01, TOUR-06, TOUR-07, TOUR-12 |
| `map.MapEventManager` | TOUR-01, TOUR-06, TOUR-12 |
| `map.MarkerDelegate` | TOUR-07 |
| `map.convertCoordinate` | TOUR-01 |
| `map.newCameraPosition` | TOUR-06, TOUR-12 |
| `mapCommon.CameraPosition` | TOUR-06 |
| `mapCommon.FontStyle` | TOUR-07 |
| `mapCommon.MarkerOptions` | TOUR-07 |
| `mapCommon.Text` | TOUR-07 |
| `mapCommon.TextPosition` | TOUR-07 |
| `maxLength` | TOUR-11 |
| `media.AVFileDescriptor` | TOUR-09 |
| `media.AVPlayer` | TOUR-09 |
| `media.createAVPlayer` | TOUR-09 |
| `navi.DrivingRouteParams` | TOUR-12 |
| `navi.getDrivingRoutes` | TOUR-01 |
| `navi.getWalkingRoutes` | TOUR-12 |
| `on('durationUpdate')` | TOUR-09 |
| `on('error')` | TOUR-09 |
| `on('mapLoad')` | TOUR-12 |
| `on('mapLongClick')` | TOUR-12 |
| `on('markerClick')` | TOUR-07, TOUR-12 |
| `on('seekDone')` | TOUR-09 |
| `on('stateChange')` | TOUR-09 |
| `on('timeUpdate')` | TOUR-09 |
| `onAnimationEnd` | TOUR-10 |
| `onAnimationStart` | TOUR-10 |
| `onAreaChange` | TOUR-10 |
| `onContentWillChange` | TOUR-03 |
| `onFocus` | TOUR-11 |
| `onScrollFrameBegin` | TOUR-10 |
| `openCustomDialog` | TOUR-12 |
| `pause` | TOUR-09 |
| `photoAccessHelper.PhotoViewPicker` | TOUR-08 |
| `play` | TOUR-09 |
| `pop` | TOUR-04, TOUR-08 |
| `prepare` | TOUR-09 |
| `pushPath` | TOUR-09 |
| `pushPathByName` | TOUR-04, TOUR-06, TOUR-08 |
| `relationalStore.getRdbStore` | TOUR-01 |
| `release` | TOUR-09 |
| `requestPermissionOnSetting` | TOUR-03 |
| `requestPermissionsFromUser` | TOUR-01, TOUR-03 |
| `reset` | TOUR-09 |
| `resourceManager.getRawFdSync` | TOUR-09 |
| `rotate` | TOUR-05 |
| `rowsTemplate` | TOUR-04 |
| `safeAreaPadding` | TOUR-04 |
| `scanBarcode.startScanForResult` | TOUR-01 |
| `scrollToIndex` | TOUR-06, TOUR-10 |
| `setClickable` | TOUR-07 |
| `setInfoWindowAnchor` | TOUR-07 |
| `setInfoWindowVisible` | TOUR-07 |
| `setInterval` | TOUR-09 |
| `setMyLocationEnabled` | TOUR-06 |
| `setTitle` | TOUR-07 |
| `setZoomControlsEnabled` | TOUR-06 |
| `showCounter` | TOUR-08 |
| `showToast` | TOUR-05, TOUR-08, TOUR-11 |
| `site.SearchByTextParams` | TOUR-06 |
| `site.searchByText` | TOUR-06 |
| `stop` | TOUR-09 |
| `translate` | TOUR-05 |
| `util.format` | TOUR-04 |
| `window.getWindowAvoidArea` | TOUR-01, TOUR-03 |
| `windowClass.on('avoidAreaChange')` | TOUR-01 |
