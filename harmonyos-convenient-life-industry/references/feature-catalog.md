# Feature catalog

> Generated from `features/*.md`. Source industry: `02_convenient_life`, 31 features.
> Do not edit by hand; regenerate it in the review repository.

Kit names are shown without the `@kit.` prefix; the full module specifier is in each feature card.

| ID | Feature | Kits | Permissions | Min API | Status | Findings |
|---|---|---|---|---:|---|---:|
| `LIFE-01` | Government-service app shell | AbilityKit, ArkUI, ArkWeb, ScanKit +7 | INTERNET | 24 | verified-with-fixes | 12 |
| `LIFE-02` | Convenient-life key-scenario catalog | - | - | n/a | verified | - |
| `LIFE-03` | Chinese licence-plate keyboard | ArkUI, AbilityKit, ArkTS, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 9 |
| `LIFE-04` | Cinema seat picker | ArkUI, AbilityKit, ArkTS, PerformanceAnalysisKit +1 | - | 20 | verified-with-fixes | 8 |
| `LIFE-05` | Three-column cascading category picker | ArkUI, AbilityKit, ArkTS, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 8 |
| `LIFE-06` | Drag-to-reorder to-do list | ArkUI, AbilityKit, ArkTS, PerformanceAnalysisKit +1 | - | 20 | verified-with-fixes | 9 |
| `LIFE-07` | Expandable bill list | ArkUI, AbilityKit, ArkTS, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 9 |
| `LIFE-08` | Combined date-and-time wheel | ArkUI, AbilityKit, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 11 |
| `LIFE-09` | Infinite icon carousel | ArkUI, AbilityKit, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 7 |
| `LIFE-10` | Password vault | AssetStoreKit, UserAuthenticationKit, ArkUI, ArkTS +2 | STORE_PERSISTENT_DATA, ACCESS_BIOMETRIC | 20 | verified-with-fixes | 9 |
| `LIFE-11` | Pinch to switch card view | ArkUI, AbilityKit, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 7 |
| `LIFE-12` | Perpetual calendar template | ArkUI | INTERNET | 16 | verified-with-fixes | 7 |
| `LIFE-13` | Three-day calendar | ArkUI, AbilityKit, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 12 |
| `LIFE-14` | Map with a docked half-modal | MapKit, LocationKit, AbilityKit, ArkUI +1 | INTERNET, LOCATION +1 | 20 | verified-with-fixes | 9 |
| `LIFE-15` | Two-level ID-type picker | ArkUI, BasicServicesKit, AbilityKit, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 7 |
| `LIFE-16` | Route planning between two searched addresses | MapKit, AbilityKit, BasicServicesKit | INTERNET, APPROXIMATELY_LOCATION +1 | 20 | verified-with-fixes | 8 |
| `LIFE-17` | Two-way linked category lists | ArkUI, AbilityKit | - | 20 | verified-with-fixes | 7 |
| `LIFE-18` | Week / month / detail calendar | ArkUI, ArkTS, LocalizationKit, AbilityKit | - | 20 | verified-with-fixes | 8 |
| `LIFE-19` | Paste an address, split it into fields | NaturalLanguageKit, BasicServicesKit, IMEKit, ArkUI +2 | - | 20 | verified-with-fixes | 9 |
| `LIFE-20` | Rotate the map to the heading, then travel along it | MapKit, AbilityKit, ArkUI, BasicServicesKit +1 | INTERNET, APPROXIMATELY_LOCATION +1 | 20 | verified-with-fixes | 8 |
| `LIFE-21` | Photograph an address label and fill the form | CoreVisionKit, NaturalLanguageKit, MediaLibraryKit, CameraKit +5 | - | 20 | verified-with-fixes | 9 |
| `LIFE-22` | Scan an ID card with a live custom camera | CameraKit, ImageKit, CoreVisionKit, NaturalLanguageKit +5 | CAMERA | 20 | verified-with-fixes | 14 |
| `LIFE-23` | Real-name registration with the CardRecognition system control | VisionKit, ArkUI, PerformanceAnalysisKit, BasicServicesKit +1 | - | 20 | verified-with-fixes | 13 |
| `LIFE-24` | Book a home-service slot and write it to the calendar | CalendarKit, AbilityKit, ArkUI, ArkTS +2 | WRITE_CALENDAR | 20 | verified-with-fixes | 16 |
| `LIFE-25` | Show the commute from a listing to work | MapKit, ArkUI, BasicServicesKit, PerformanceAnalysisKit +1 | - | 20 | verified-with-fixes | 14 |
| `LIFE-26` | Find nearby service outlets and act on one | MapKit, LocationKit, TelephonyKit, AbilityKit +4 | LOCATION, APPROXIMATELY_LOCATION | 20 | verified-with-fixes | 14 |
| `LIFE-27` | Grouped photo preview with a linked category strip | ArkUI, AbilityKit, BasicServicesKit, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 12 |
| `LIFE-28` | Let an embedded H5 page pick a contact | ArkWeb, ContactsKit, ArkUI, AbilityKit +1 | - | 24 | verified-with-fixes | 13 |
| `LIFE-29` | Draw a draggable coverage circle on the map | MapKit, ArkUI, AbilityKit, BasicServicesKit +1 | INTERNET | 20 | verified-with-fixes | 9 |
| `LIFE-30` | Batch thumbnail generation off the UI thread | ArkTS, ImageKit, CoreFileKit, MediaLibraryKit +3 | - | 20 | verified-with-fixes | 13 |
| `LIFE-31` | Convenient-life industry FAQ redirect | - | - | 20 | verified-with-fixes | 2 |

Read the full card at `features/<ID>.md` before writing any code. `verified-with-fixes` means the published HQ document contains at least one defect that the card corrects.
