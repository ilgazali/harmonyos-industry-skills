# API map

> Generated from `features/*.md`. Source industry: `12_jobs`, 5 features.
> Do not edit by hand; regenerate it in the review repository.

## Feature to kits, permissions and API level

| ID | Kits | Permissions | Min API | Modules |
|---|---|---|---:|---|
| `JOBS-01` | - | - | 17 | - |
| `JOBS-02` | @kit.NotificationKit, @kit.AbilityKit, @kit.ArkUI, @kit.BasicServicesKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `JOBS-03` | @kit.ArkUI, @kit.AbilityKit, @kit.BasicServicesKit, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `JOBS-04` | @kit.ArkUI, @kit.AbilityKit, @kit.BasicServicesKit, @kit.PerformanceAnalysisKit | - | 17 | entry |
| `JOBS-05` | - | - | 17 | - |

## Kit to features

| Kit | Features |
|---|---|
| `@kit.AbilityKit` | JOBS-02, JOBS-03, JOBS-04 |
| `@kit.ArkUI` | JOBS-02, JOBS-03, JOBS-04 |
| `@kit.BasicServicesKit` | JOBS-02, JOBS-03, JOBS-04 |
| `@kit.NotificationKit` | JOBS-02 |
| `@kit.PerformanceAnalysisKit` | JOBS-02, JOBS-03, JOBS-04 |

## Permission to features

None.

## API to features

| API | Features |
|---|---|
| `@ObservedV2` | JOBS-03 |
| `@Prop` | JOBS-04 |
| `@StorageProp` | JOBS-02, JOBS-03, JOBS-04 |
| `@Trace` | JOBS-03 |
| `Blank` | JOBS-04 |
| `Divider` | JOBS-04 |
| `ForEach` | JOBS-03, JOBS-04 |
| `GestureGroup` | JOBS-03 |
| `List` | JOBS-02, JOBS-04 |
| `ListItem` | JOBS-04 |
| `NavDestination` | JOBS-04 |
| `NavDestinationContext` | JOBS-04 |
| `NavPathStack` | JOBS-04 |
| `Navigation` | JOBS-04 |
| `PanGesture` | JOBS-03 |
| `PanGestureOptions` | JOBS-03 |
| `Search` | JOBS-02 |
| `Stack` | JOBS-03, JOBS-04 |
| `UIContext.animateTo` | JOBS-03 |
| `UIContext.getHostContext` | JOBS-02 |
| `UIContext.getPromptAction` | JOBS-02, JOBS-03 |
| `UIContext.px2vp` | JOBS-03, JOBS-04 |
| `display.getDefaultDisplaySync` | JOBS-03 |
| `notificationManager.isNotificationEnabledSync` | JOBS-02 |
| `notificationManager.openNotificationSettings` | JOBS-02 |
| `notificationManager.requestEnableNotification` | JOBS-02 |
| `offset` | JOBS-03 |
| `onReady` | JOBS-04 |
| `opacity` | JOBS-03 |
| `pop` | JOBS-04 |
| `pushPathByName` | JOBS-04 |
| `routerMap` | JOBS-04 |
| `safeAreaPadding` | JOBS-02 |
| `window.getWindowAvoidArea` | JOBS-02, JOBS-04 |
| `zIndex` | JOBS-03 |
