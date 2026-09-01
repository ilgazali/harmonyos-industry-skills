# API map

> Generated from `features/*.md`. Source industry: `17_food`, 6 features.
> Do not edit by hand; regenerate it in the review repository.

## Feature to kits, permissions and API level

| ID | Kits | Permissions | Min API | Modules |
|---|---|---|---:|---|
| `FOOD-01` | @kit.AbilityKit, @kit.AccountKit, @kit.ArkData, @kit.ArkTS, @kit.ArkUI, @kit.BasicServicesKit, @kit.FormKit, @kit.LocationKit, @kit.MapKit, @kit.PerformanceAnalysisKit, @kit.ScanKit, @kit.TelephonyKit | ohos.permission.LOCATION, ohos.permission.APPROXIMATELY_LOCATION, ohos.permission.INTERNET, ohos.permission.GET_NETWORK_INFO, ohos.permission.GYROSCOPE, ohos.permission.ACCELEROMETER | 20 | feature_home (har), feature_mine (har), feature_food (har), feature_order (har), feature_map (har), router_manage (har), common_utils (har), phone (entry) |
| `FOOD-02` | @kit.AbilityKit, @kit.AccountKit, @kit.AdsKit, @kit.ArkData, @kit.ArkTS, @kit.ArkUI, @kit.BasicServicesKit, @kit.CryptoArchitectureKit, @kit.FormKit, @kit.MediaLibraryKit, @kit.PaymentKit, @kit.PerformanceAnalysisKit, @kit.ScenarioFusionKit | ohos.permission.INTERNET, ohos.permission.GET_NETWORK_INFO, ohos.permission.APP_TRACKING_CONSENT | 24 | shopping_basket (har), home_search (har), aggregated_login (har), upload_recipe (har), aggregated_payment (har), base_ui (har), personal_homepage (har), link_category (har) |
| `FOOD-03` | - | - | n/a | - |
| `FOOD-04` | @kit.MapKit, @kit.AbilityKit, @kit.ArkUI, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `FOOD-05` | @kit.ArkUI, @kit.AbilityKit, @kit.PerformanceAnalysisKit, @kit.CoreFileKit | - | 20 | entry |
| `FOOD-06` | - | - | n/a | - |

## Kit to features

| Kit | Features |
|---|---|
| `@kit.AbilityKit` | FOOD-01, FOOD-02, FOOD-04, FOOD-05 |
| `@kit.AccountKit` | FOOD-01, FOOD-02 |
| `@kit.AdsKit` | FOOD-02 |
| `@kit.ArkData` | FOOD-01, FOOD-02 |
| `@kit.ArkTS` | FOOD-01, FOOD-02 |
| `@kit.ArkUI` | FOOD-01, FOOD-02, FOOD-04, FOOD-05 |
| `@kit.BasicServicesKit` | FOOD-01, FOOD-02 |
| `@kit.CoreFileKit` | FOOD-05 |
| `@kit.CryptoArchitectureKit` | FOOD-02 |
| `@kit.FormKit` | FOOD-01, FOOD-02 |
| `@kit.LocationKit` | FOOD-01 |
| `@kit.MapKit` | FOOD-01, FOOD-04 |
| `@kit.MediaLibraryKit` | FOOD-02 |
| `@kit.PaymentKit` | FOOD-02 |
| `@kit.PerformanceAnalysisKit` | FOOD-01, FOOD-02, FOOD-04, FOOD-05 |
| `@kit.ScanKit` | FOOD-01 |
| `@kit.ScenarioFusionKit` | FOOD-02 |
| `@kit.TelephonyKit` | FOOD-01 |

## Permission to features

| Permission | Features |
|---|---|
| `ohos.permission.ACCELEROMETER` | FOOD-01 |
| `ohos.permission.APPROXIMATELY_LOCATION` | FOOD-01 |
| `ohos.permission.APP_TRACKING_CONSENT` | FOOD-02 |
| `ohos.permission.GET_NETWORK_INFO` | FOOD-01, FOOD-02 |
| `ohos.permission.GYROSCOPE` | FOOD-01 |
| `ohos.permission.INTERNET` | FOOD-01, FOOD-02 |
| `ohos.permission.LOCATION` | FOOD-01 |

## API to features

| API | Features |
|---|---|
| `$$` | FOOD-05 |
| `@StorageProp` | FOOD-04 |
| `DataChangeListener` | FOOD-05 |
| `FlowItem` | FOOD-05 |
| `IDataSource` | FOOD-05 |
| `LazyForEach` | FOOD-05 |
| `LoadingProgress` | FOOD-05 |
| `NaviParams` | FOOD-04 |
| `NestedScrollMode` | FOOD-05 |
| `PoiDetailParams` | FOOD-04 |
| `Progress` | FOOD-05 |
| `ProgressStatus` | FOOD-05 |
| `Rating` | FOOD-04 |
| `Refresh` | FOOD-05 |
| `RefreshStatus` | FOOD-05 |
| `RoutePlanParams` | FOOD-04 |
| `Scroll` | FOOD-05 |
| `StaticMapMarker` | FOOD-04 |
| `StaticMapOptions` | FOOD-04 |
| `Tabs` | FOOD-05 |
| `UIContext` | FOOD-01 |
| `UIContext.getHostContext` | FOOD-04 |
| `UIContext.getMeasureUtils` | FOOD-05 |
| `UIContext.px2vp` | FOOD-04, FOOD-05 |
| `WaterFlow` | FOOD-05 |
| `abilityAccessCtrl` | FOOD-01, FOOD-02 |
| `advertising` | FOOD-02 |
| `authentication` | FOOD-01, FOOD-02 |
| `bundleManager` | FOOD-01 |
| `call` | FOOD-01 |
| `common` | FOOD-01, FOOD-02 |
| `cryptoFramework` | FOOD-02 |
| `deviceInfo` | FOOD-01 |
| `display` | FOOD-02 |
| `edgeEffect` | FOOD-05 |
| `emitter` | FOOD-02 |
| `formBindingData` | FOOD-01, FOOD-02 |
| `formInfo` | FOOD-01, FOOD-02 |
| `fs` | FOOD-02 |
| `functionalButtonComponentManager` | FOOD-02 |
| `geoLocationManager` | FOOD-01 |
| `hilog` | FOOD-01, FOOD-02 |
| `map` | FOOD-01 |
| `nestedScroll` | FOOD-05 |
| `onContentWillChange` | FOOD-05 |
| `onDidScroll` | FOOD-05 |
| `onOffsetChange` | FOOD-05 |
| `onReachEnd` | FOOD-05 |
| `onRefreshing` | FOOD-05 |
| `onStateChange` | FOOD-05 |
| `petalMaps.openMapNavi` | FOOD-04 |
| `petalMaps.openMapPoiDetail` | FOOD-04 |
| `petalMaps.openMapRoutePlan` | FOOD-04 |
| `staticMap.IconSize` | FOOD-04 |
| `staticMap.getMapImage` | FOOD-04 |
| `window.getWindowAvoidArea` | FOOD-04, FOOD-05 |
