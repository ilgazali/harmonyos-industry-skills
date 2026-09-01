# Feature catalog

> Generated from `features/*.md`. Source industry: `09_tourism`, 13 features.
> Do not edit by hand; regenerate it in the review repository.

Kit names are shown without the `@kit.` prefix; the full module specifier is in each feature card.

| ID | Feature | Kits | Permissions | Min API | Status | Findings |
|---|---|---|---|---:|---|---:|
| `TOUR-01` | Tourist park app | MapKit, LocationKit, AbilityKit, BasicServicesKit +5 | INTERNET, APPROXIMATELY_LOCATION +1 | 24 | verified-with-fixes | 16 |
| `TOUR-02` | Tourism key-scenario catalog | - | - | 24 | verified | - |
| `TOUR-03` | Location permission bubble | AbilityKit, PerformanceAnalysisKit, BasicServicesKit, ArkUI | LOCATION, APPROXIMATELY_LOCATION +1 | 20 | verified-with-fixes | 7 |
| `TOUR-04` | Date range picker | ArkTS, PerformanceAnalysisKit, ArkUI | - | 20 | verified-with-fixes | 7 |
| `TOUR-05` | Origin and destination swap | ArkUI, @ohos.curves | - | 20 | verified-with-fixes | 7 |
| `TOUR-06` | Destination map with nearby POIs | MapKit, BasicServicesKit, ArkUI, AbilityKit +1 | INTERNET, GET_NETWORK_INFO | 20 | verified-with-fixes | 8 |
| `TOUR-07` | Named map markers | MapKit, BasicServicesKit, ArkUI | INTERNET, GET_NETWORK_INFO | 20 | verified-with-fixes | 6 |
| `TOUR-08` | Hotel stay review | MediaLibraryKit, ArkUI, BasicServicesKit, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 6 |
| `TOUR-09` | Attraction audio guide | MediaKit, AbilityKit, BasicServicesKit, PerformanceAnalysisKit +1 | - | 20 | verified-with-fixes | 9 |
| `TOUR-10` | Hotel order list | ArkTS, BasicServicesKit, ArkUI, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 7 |
| `TOUR-11` | Traveller details form | ArkUI | - | 20 | verified-with-fixes | 7 |
| `TOUR-12` | Long-press to drop a marker, reverse-geocode it and save the address | MapKit, LocationKit, BasicServicesKit, ArkUI +1 | INTERNET, GET_NETWORK_INFO | 20 | verified-with-fixes | 8 |
| `TOUR-13` | Tourism industry FAQ | - | - | 24 | verified | - |

Read the full card at `features/<ID>.md` before writing any code. `verified-with-fixes` means the published HQ document contains at least one defect that the card corrects.
