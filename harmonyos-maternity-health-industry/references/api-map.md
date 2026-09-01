# API map

> Generated from `features/*.md`. Source industry: `10_maternity_health`, 5 features.
> Do not edit by hand; regenerate it in the review repository.

## Feature to kits, permissions and API level

| ID | Kits | Permissions | Min API | Modules |
|---|---|---|---:|---|
| `MAT-01` | @kit.ArkUI, @kit.ArkTS, @kit.AbilityKit, @kit.MediaLibraryKit, @kit.ArkData, @kit.BasicServicesKit, @kit.PerformanceAnalysisKit | ohos.permission.READ_IMAGEVIDEO | 20 | entry, common, features/recommend, features/cloudCircle, features/mine, features/record |
| `MAT-02` | - | - | 20 | - |
| `MAT-03` | @kit.ArkUI, @kit.MediaLibraryKit, @kit.CoreFileKit, @kit.AbilityKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `MAT-04` | @kit.ArkUI, @kit.ArkTS, @kit.AbilityKit | - | 20 | entry |
| `MAT-05` | - | - | 20 | - |

## Kit to features

| Kit | Features |
|---|---|
| `@kit.AbilityKit` | MAT-01, MAT-03, MAT-04 |
| `@kit.ArkData` | MAT-01 |
| `@kit.ArkTS` | MAT-01, MAT-04 |
| `@kit.ArkUI` | MAT-01, MAT-03, MAT-04 |
| `@kit.BasicServicesKit` | MAT-01 |
| `@kit.CoreFileKit` | MAT-03 |
| `@kit.MediaLibraryKit` | MAT-01, MAT-03 |
| `@kit.PerformanceAnalysisKit` | MAT-01, MAT-03 |

## Permission to features

| Permission | Features |
|---|---|
| `ohos.permission.READ_IMAGEVIDEO` | MAT-01 |

## API to features

| API | Features |
|---|---|
| `DatePicker` | MAT-03, MAT-04 |
| `Description` | MAT-04 |
| `GridRow` | MAT-01 |
| `IAxisValueFormatter` | MAT-04 |
| `IDataSource` | MAT-01 |
| `Intl.DateTimeFormat` | MAT-03 |
| `LazyForEach` | MAT-01 |
| `Legend` | MAT-04 |
| `LineChartModel` | MAT-04 |
| `LineData` | MAT-04 |
| `LineDataSet` | MAT-04 |
| `List` | MAT-03 |
| `ListItemGroup` | MAT-03 |
| `LocalStorage` | MAT-04 |
| `NavPathStack` | MAT-01, MAT-04 |
| `NavPathStack.pop` | MAT-03 |
| `NavPathStack.pushPathByName` | MAT-03 |
| `Navigation` | MAT-01 |
| `PromptAction.showToast` | MAT-03 |
| `Tabs` | MAT-01, MAT-04 |
| `UIContext.getMeasureUtils` | MAT-01 |
| `XAxis` | MAT-04 |
| `YAxis` | MAT-04 |
| `abilityAccessCtrl.requestPermissionsFromUser` | MAT-01 |
| `bindSheet` | MAT-03, MAT-04 |
| `cameraPicker.pick` | MAT-01 |
| `photoAccessHelper.PhotoSelectOptions` | MAT-03 |
| `photoAccessHelper.PhotoViewPicker` | MAT-03 |
| `photoAccessHelper.getAssets` | MAT-01 |
| `photoAccessHelper.getPhotoAccessHelper` | MAT-01 |
| `util.format` | MAT-04 |
| `window.getLastWindow` | MAT-01 |
