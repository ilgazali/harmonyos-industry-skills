# Feature catalog

> Generated from `features/*.md`. Source industry: `05_office`, 32 features.
> Do not edit by hand; regenerate it in the review repository.

Kit names are shown without the `@kit.` prefix; the full module specifier is in each feature card.

| ID | Feature | Kits | Permissions | Min API | Status | Findings |
|---|---|---|---|---:|---|---:|
| `OFFICE-01` | Integrated office app shell | AbilityKit, ArkUI, ArkData, ArkTS +7 | READ_CALENDAR, WRITE_CALENDAR +1 | 20 | verified-with-fixes | 12 |
| `OFFICE-02` | Key-scenario index for the office industry | - | - | n/a | verified | - |
| `OFFICE-03` | Attendance check-in location | AbilityKit, LocationKit, ArkUI, BasicServicesKit +1 | LOCATION, APPROXIMATELY_LOCATION | 20 | verified-with-fixes | 6 |
| `OFFICE-04` | Secure online PDF preview | ArkWeb, ArkUI, AbilityKit, BasicServicesKit +1 | INTERNET, PRIVACY_WINDOW | 20 | verified-with-fixes | 5 |
| `OFFICE-05` | Personal card page | ScanKit, ImageKit, MediaLibraryKit, CoreFileKit +5 | - | 20 | verified-with-fixes | 9 |
| `OFFICE-06` | Document approval | ArkUI, CoreFileKit, PreviewKit, BasicServicesKit +2 | INTERNET | 20 | verified-with-fixes | 8 |
| `OFFICE-07` | ID photo capture with a mask overlay | CameraKit, MediaLibraryKit, AbilityKit, ArkUI +2 | CAMERA | 20 | verified-with-fixes | 8 |
| `OFFICE-08` | File download and preview | PreviewKit, BasicServicesKit, AbilityKit, ArkUI +1 | INTERNET | 20 | verified-with-fixes | 7 |
| `OFFICE-09` | Visitor management | ContactsKit, ScanKit, ArkUI, AbilityKit +2 | - | 20 | verified-with-fixes | 6 |
| `OFFICE-10` | Mail attachments | MediaLibraryKit, CameraKit, CoreFileKit, PreviewKit +4 | - | 20 | verified-with-fixes | 7 |
| `OFFICE-11` | Publish a meeting from custom dialogs and write it into the system calendar | CalendarKit, CoreFileKit, PreviewKit, AbilityKit +4 | READ_CALENDAR, WRITE_CALENDAR | 20 | verified-with-fixes | 10 |
| `OFFICE-12` | Electronic seal on a PDF | PDFKit, CoreFileKit, AbilityKit, ArkUI +2 | - | 20 | verified-with-fixes | 6 |
| `OFFICE-13` | Open a file with another app | AbilityKit, CoreFileKit, ArkUI, BasicServicesKit +1 | - | 20 | verified-with-fixes | 6 |
| `OFFICE-14` | Marquee banner notice | ArkUI, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 5 |
| `OFFICE-15` | Handle a message later | ArkUI, ArkData, AbilityKit, BasicServicesKit +1 | - | 20 | verified-with-fixes | 6 |
| `OFFICE-16` | ID-photo recommendation | MediaLibraryKit, ArkUI, AbilityKit, BasicServicesKit +1 | - | 20 | verified-with-fixes | 4 |
| `OFFICE-17` | Multi-level corporate directory | ArkUI, ArkTS, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 5 |
| `OFFICE-18` | Online meeting main/sub window swap | ArkUI, CameraKit, AbilityKit, ArkData +2 | CAMERA | 20 | verified-with-fixes | 7 |
| `OFFICE-19` | Watermark camera | CameraKit, LocationKit, ImageKit, MediaLibraryKit +4 | CAMERA, LOCATION +2 | 20 | verified-with-fixes | 9 |
| `OFFICE-20` | Insert and edit an image in a note | ArkUI, MediaLibraryKit, CameraKit, ImageKit +3 | - | 20 | verified-with-fixes | 6 |
| `OFFICE-21` | Voice note record and playback | AudioKit, CoreFileKit, ArkUI, AbilityKit +2 | MICROPHONE | 20 | verified-with-fixes | 9 |
| `OFFICE-22` | Add special-attention contacts | ArkUI | - | 20 | verified-with-fixes | 6 |
| `OFFICE-23` | Live speech-to-text notes | CoreSpeechKit, AudioKit, CoreFileKit, AbilityKit +3 | MICROPHONE | 20 | verified-with-fixes | 5 |
| `OFFICE-24` | Application background watermark | ArkUI | - | 20 | verified-with-fixes | 5 |
| `OFFICE-25` | Pinned group announcement | ArkUI | - | 20 | verified-with-fixes | 4 |
| `OFFICE-26` | Multi-level organisation menu | ArkUI | - | 20 | verified-with-fixes | 5 |
| `OFFICE-27` | Batch-sync to-dos into the system calendar | CalendarKit, AbilityKit, ArkData, ArkUI +2 | READ_CALENDAR, WRITE_CALENDAR | 20 | verified-with-fixes | 5 |
| `OFFICE-28` | Adding and deleting annotations in PDF preview | PDFKit, ArkData, CoreFileKit, AbilityKit +3 | - | 24 | verified-with-fixes | 8 |
| `OFFICE-29` | Showing enterprise employee details on the incoming-call screen | CallServiceKit, ArkData, AbilityKit, ArkUI +2 | - | 15 | verified-with-fixes | 8 |
| `OFFICE-30` | Warning the candidate when an exam app is switched to the background | ArkUI, NotificationKit, MediaKit, AudioKit +4 | - | 20 | verified-with-fixes | 9 |
| `OFFICE-31` | Collaborative schedule management | CalendarKit, ShareKit, AbilityKit, ArkUI +4 | READ_CALENDAR, WRITE_CALENDAR | 20 | verified-with-fixes | 13 |
| `OFFICE-32` | Industry FAQ placeholder for the office industry | - | - | n/a | verified-with-fixes | 2 |

Read the full card at `features/<ID>.md` before writing any code. `verified-with-fixes` means the published HQ document contains at least one defect that the card corrects.
