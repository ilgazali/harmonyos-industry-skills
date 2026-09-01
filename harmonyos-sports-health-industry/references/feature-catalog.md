# Feature catalog

> Generated from `features/*.md`. Source industry: `03_sports_health`, 15 features.
> Do not edit by hand; regenerate it in the review repository.

Kit names are shown without the `@kit.` prefix; the full module specifier is in each feature card.

| ID | Feature | Kits | Permissions | Min API | Status | Findings |
|---|---|---|---|---:|---|---:|
| `SPORT-01` | Sports and health app | ConnectivityKit, SensorServiceKit, AbilityKit, NotificationKit +4 | ACCESS_BLUETOOTH, ACTIVITY_MOTION | 20 | verified-with-fixes | 14 |
| `SPORT-02` | Sports and health key-scenario catalog | - | - | 20 | verified | - |
| `SPORT-03` | Hold-to-finish a workout | ArkUI, ArkTS | - | 20 | verified-with-fixes | 5 |
| `SPORT-04` | Workout plans on a custom calendar | CalendarKit, ArkData, AbilityKit, ArkUI +2 | READ_CALENDAR, WRITE_CALENDAR | 20 | verified-with-fixes | 6 |
| `SPORT-05` | Periodic health charts | LocalizationKit, ArkTS, AbilityKit, ArkUI | - | 20 | verified-with-fixes | 5 |
| `SPORT-06` | Match scorer | ArkUI | - | 20 | verified-with-fixes | 4 |
| `SPORT-07` | The three activity rings | ArkUI | - | 20 | verified-with-fixes | 4 |
| `SPORT-08` | Fit a whole route on screen | MapKit, BasicServicesKit, ArkUI, PerformanceAnalysisKit | INTERNET, GET_NETWORK_INFO | 20 | verified-with-fixes | 5 |
| `SPORT-09` | Publish a group activity | ArkUI, MediaLibraryKit, PerformanceAnalysisKit, BasicServicesKit | INTERNET | 20 | verified-with-fixes | 4 |
| `SPORT-10` | Knockout bracket | ArkUI, CryptoArchitectureKit | - | 20 | verified-with-fixes | 4 |
| `SPORT-11` | Live GPS track | LocationKit, MapKit, AbilityKit, ArkUI | LOCATION, APPROXIMATELY_LOCATION +2 | 20 | verified-with-fixes | 5 |
| `SPORT-12` | The 3-2-1 countdown | ArkUI | - | 20 | verified-with-fixes | 4 |
| `SPORT-13` | Reorder dashboard cards by drag | ArkUI, @ohos.curves | - | 20 | verified-with-fixes | 3 |
| `SPORT-14` | Scan a medicine barcode | ScanKit, ArkUI, LocalizationKit, BasicServicesKit | - | 20 | verified-with-fixes | 4 |
| `SPORT-15` | Sports and health industry FAQ | - | - | 20 | verified | - |

Read the full card at `features/<ID>.md` before writing any code. `verified-with-fixes` means the published HQ document contains at least one defect that the card corrects.
