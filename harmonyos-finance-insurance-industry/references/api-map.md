# API map

> Generated from `features/*.md`. Source industry: `07_finance_insurance`, 11 features.
> Do not edit by hand; regenerate it in the review repository.

## Feature to kits, permissions and API level

| ID | Kits | Permissions | Min API | Modules |
|---|---|---|---:|---|
| `FIN-01` | @kit.ArkUI, @kit.VisionKit, @kit.AbilityKit, @kit.LocationKit, @kit.PerformanceAnalysisKit | ohos.permission.PRIVACY_WINDOW, ohos.permission.APPROXIMATELY_LOCATION | 20 | products/phone, commons/basic, commons/dfx, commons/jsbridge, commons/router, features/account, features/insurance, features/mine, features/service, features/tools |
| `FIN-02` | - | - | 20 | - |
| `FIN-03` | @kit.ArkUI, @kit.ArkTS, @kit.LocalizationKit | - | 20 | entry |
| `FIN-04` | @kit.ArkUI | - | 20 | entry |
| `FIN-05` | @kit.ArkUI | - | 20 | entry, keyboard, common |
| `FIN-06` | @kit.ArkUI, @kit.CryptoArchitectureKit, @kit.ArkData, @kit.ArkTS | - | 20 | entry |
| `FIN-07` | @kit.ArkUI, @kit.PerformanceAnalysisKit | - | 20 | entry |
| `FIN-08` | @kit.BasicServicesKit, @kit.ArkUI, @kit.IMEKit, @kit.AbilityKit | ohos.permission.PRIVACY_WINDOW | 20 | entry |
| `FIN-09` | @kit.ArkUI, @kit.CryptoArchitectureKit, @kit.BasicServicesKit, @kit.AbilityKit | ohos.permission.PRIVACY_WINDOW | 20 | entry |
| `FIN-10` | @kit.ArkUI | - | 20 | entry |
| `FIN-11` | - | - | 20 | - |

## Kit to features

| Kit | Features |
|---|---|
| `@kit.AbilityKit` | FIN-01, FIN-08, FIN-09 |
| `@kit.ArkData` | FIN-06 |
| `@kit.ArkTS` | FIN-03, FIN-06 |
| `@kit.ArkUI` | FIN-01, FIN-03, FIN-04, FIN-05, FIN-06, FIN-07, FIN-08, FIN-09, FIN-10 |
| `@kit.BasicServicesKit` | FIN-08, FIN-09 |
| `@kit.CryptoArchitectureKit` | FIN-06, FIN-09 |
| `@kit.IMEKit` | FIN-08 |
| `@kit.LocalizationKit` | FIN-03 |
| `@kit.LocationKit` | FIN-01 |
| `@kit.PerformanceAnalysisKit` | FIN-01, FIN-07 |
| `@kit.VisionKit` | FIN-01 |

## Permission to features

| Permission | Features |
|---|---|
| `ohos.permission.APPROXIMATELY_LOCATION` | FIN-01 |
| `ohos.permission.PRIVACY_WINDOW` | FIN-01, FIN-08, FIN-09 |

## API to features

| API | Features |
|---|---|
| `@Consume` | FIN-01 |
| `@Local` | FIN-04 |
| `@Prop` | FIN-03 |
| `@Provide` | FIN-01 |
| `@Watch` | FIN-03 |
| `Canvas` | FIN-03 |
| `CanvasRenderingContext2D` | FIN-03 |
| `CardRecognition` | FIN-01 |
| `CardRecognitionResult` | FIN-01 |
| `CardSide` | FIN-01 |
| `CardType` | FIN-01 |
| `ChartColor` | FIN-10 |
| `Cipher.doFinal` | FIN-06 |
| `Cipher.init` | FIN-06 |
| `Date` | FIN-04 |
| `Flex` | FIN-04 |
| `ForEach` | FIN-04, FIN-05, FIN-07 |
| `Grid` | FIN-01, FIN-05, FIN-09 |
| `GridItem` | FIN-01, FIN-05 |
| `GridLayoutOptions` | FIN-09 |
| `InsertValue` | FIN-07 |
| `JArrayList` | FIN-10 |
| `List` | FIN-07, FIN-10 |
| `ListItem` | FIN-10 |
| `NavDestination` | FIN-07 |
| `NavPathStack` | FIN-07 |
| `NavPathStack.popToName` | FIN-01 |
| `NavPathStack.pushPathByName` | FIN-01 |
| `Navigation` | FIN-07 |
| `PanGesture` | FIN-03 |
| `PatternLock` | FIN-06 |
| `PatternLockController` | FIN-06 |
| `PieChart` | FIN-10 |
| `PieChartModel` | FIN-10 |
| `PieData` | FIN-10 |
| `PieDataSet` | FIN-10 |
| `PieEntry` | FIN-10 |
| `PinchGesture` | FIN-03 |
| `Preferences.flush` | FIN-06 |
| `Preferences.get` | FIN-06 |
| `Preferences.put` | FIN-06 |
| `Progress` | FIN-10 |
| `PromptAction.openCustomDialog` | FIN-08 |
| `RenderingContextSettings` | FIN-03 |
| `Scroller` | FIN-07 |
| `Tabs` | FIN-05, FIN-10 |
| `TextInput.customKeyboard` | FIN-01, FIN-05, FIN-09 |
| `TextInput.inputFilter` | FIN-07 |
| `TextInputController` | FIN-05, FIN-09 |
| `TextInputController.stopEditing` | FIN-01 |
| `TextPicker` | FIN-07 |
| `UIContext.showDatePickerDialog` | FIN-04 |
| `ValuePosition` | FIN-10 |
| `Window.setWindowPrivacyMode` | FIN-01 |
| `attach` | FIN-08 |
| `bindSheet` | FIN-07 |
| `buffer.from` | FIN-03 |
| `caretPosition` | FIN-05, FIN-09 |
| `clearInterval` | FIN-03 |
| `clearRect` | FIN-03 |
| `convertKey` | FIN-06 |
| `createCipher` | FIN-06 |
| `cryptoFramework.createAsyKeyGenerator` | FIN-06 |
| `cryptoFramework.createRandom` | FIN-09 |
| `detach` | FIN-08 |
| `emitter.off` | FIN-09 |
| `emitter.on` | FIN-09 |
| `generateRandomSync` | FIN-09 |
| `geoLocationManager.getAddressesFromLocation` | FIN-01 |
| `geoLocationManager.getCurrentLocation` | FIN-01 |
| `inputMethod.getController` | FIN-08 |
| `on deleteLeft` | FIN-08 |
| `on insertText` | FIN-08 |
| `onChange` | FIN-05 |
| `onPaste` | FIN-05 |
| `onPatternComplete` | FIN-06 |
| `onReady` | FIN-07 |
| `onWillInsert` | FIN-07 |
| `pasteboard.createData` | FIN-08 |
| `pasteboard.getSystemPasteboard` | FIN-08 |
| `preferences.getPreferences` | FIN-06 |
| `resourceManager.getRawFileContent` | FIN-03 |
| `resourceManager.getStringSync` | FIN-03 |
| `setData` | FIN-08 |
| `setHoleRadius` | FIN-10 |
| `setInterval` | FIN-03 |
| `setMinAngleForSlices` | FIN-10 |
| `setValueFormatter` | FIN-10 |
| `setWindowPrivacyMode` | FIN-08 |
| `window.getLastWindow` | FIN-01, FIN-08 |
| `window.setWindowPrivacyMode` | FIN-09 |
