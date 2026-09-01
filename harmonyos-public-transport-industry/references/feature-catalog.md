# Feature catalog

> Generated from `features/*.md`. Source industry: `06_public_transport`, 9 features.
> Do not edit by hand; regenerate it in the review repository.

Kit names are shown without the `@kit.` prefix; the full module specifier is in each feature card.

| ID | Feature | Kits | Permissions | Min API | Status | Findings |
|---|---|---|---|---:|---|---:|
| `TRANS-01` | Transit and navigation app | MapKit, LocationKit, AbilityKit, ArkData +2 | INTERNET, LOCATION +1 | 24 | verified-with-fixes | 18 |
| `TRANS-02` | Key scenario index for the public transport industry | - | - | 24 | verified-with-fixes | 1 |
| `TRANS-03` | Home-screen widget that opens the ride code | ArkUI, AbilityKit, FormKit, PerformanceAnalysisKit +1 | - | 20 | verified-with-fixes | 9 |
| `TRANS-04` | Real-time bus arrivals | LocationKit, AbilityKit, BasicServicesKit, ArkUI | APPROXIMATELY_LOCATION | 20 | verified-with-fixes | 6 |
| `TRANS-05` | Ride history with a date-range filter | ArkUI | - | 20 | verified-with-fixes | 5 |
| `TRANS-06` | Pin an in-app shortcut to the home screen | StoreKit, AbilityKit, ArkUI | - | 20 | verified-with-fixes | 5 |
| `TRANS-07` | Boarding and alighting reminders with Notification Kit | NotificationKit, ArkUI, BasicServicesKit | - | 20 | verified-with-fixes | 6 |
| `TRANS-08` | Map rotation lock, zoom stepping and locate-me | MapKit, LocationKit, AbilityKit, ArkUI +1 | LOCATION, APPROXIMATELY_LOCATION | 20 | verified-with-fixes | 6 |
| `TRANS-09` | Industry FAQ | - | - | 24 | verified-with-fixes | 1 |

Read the full card at `features/<ID>.md` before writing any code. `verified-with-fixes` means the published HQ document contains at least one defect that the card corrects.
