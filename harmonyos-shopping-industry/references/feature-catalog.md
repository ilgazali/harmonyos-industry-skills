# Feature catalog

> Generated from `features/*.md`. Source industry: `16_shopping`, 24 features.
> Do not edit by hand; regenerate it in the review repository.

Kit names are shown without the `@kit.` prefix; the full module specifier is in each feature card.

| ID | Feature | Kits | Permissions | Min API | Status | Findings |
|---|---|---|---|---:|---|---:|
| `SHOP-01` | Shopping mall app skeleton | AbilityKit, ArkData, ArkTS, ArkUI +15 | CAMERA, MICROPHONE +3 | 20 | verified-with-fixes | 6 |
| `SHOP-02` | Key scenario index | - | - | 20 | verified | - |
| `SHOP-03` | Coupon wallet | AbilityKit, ArkTS, ArkUI, BasicServicesKit +1 | - | 20 | verified-with-fixes | 2 |
| `SHOP-04` | Daily check-in and points | AbilityKit, ArkData, ArkUI, BasicServicesKit +3 | - | 20 | verified-with-fixes | 2 |
| `SHOP-05` | One coupon per device | DeviceSecurityKit, ArkData, BasicServicesKit, PerformanceAnalysisKit +2 | INTERNET | 21 | verified-with-fixes | 5 |
| `SHOP-06` | Pull-down to navigate | ArkUI, ArkWeb, AbilityKit, BasicServicesKit +4 | - | 20 | verified | 4 |
| `SHOP-07` | Scratch card | ArkUI, ImageKit, PerformanceAnalysisKit, AbilityKit +2 | - | 20 | verified-with-fixes | 3 |
| `SHOP-08` | Long-press to mark a product | ArkUI, AbilityKit, BasicServicesKit, PerformanceAnalysisKit +1 | - | 20 | verified-with-fixes | 2 |
| `SHOP-09` | Editable search history | AbilityKit, BasicServicesKit | - | 20 | verified-with-fixes | 1 |
| `SHOP-10` | Sticky mall home page | AbilityKit, ArkUI, PerformanceAnalysisKit | - | 20 | verified | 2 |
| `SHOP-11` | Skeleton screen | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 | - | 20 | verified-with-fixes | 2 |
| `SHOP-12` | Expand/collapse a wrapped history block | AbilityKit, ArkUI, CoreFileKit, PerformanceAnalysisKit | - | 20 | verified | 2 |
| `SHOP-13` | Order status tabs and the post-receipt review page | AbilityKit, ArkData, ArkTS, ArkUI +5 | - | 20 | verified-with-fixes | 3 |
| `SHOP-14` | Self-sizing quick-entry menu | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 | - | 20 | verified-with-fixes | 3 |
| `SHOP-15` | Browsing history that survives a restart | AbilityKit, ArkUI, CoreFileKit, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 2 |
| `SHOP-16` | Ticket-drop countdown with a calendar reminder | AbilityKit, ArkUI, BasicServicesKit, CalendarKit +2 | READ_CALENDAR, WRITE_CALENDAR | 20 | verified-with-fixes | 3 |
| `SHOP-17` | Red envelope rain | AbilityKit, ArkUI, CoreFileKit, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 3 |
| `SHOP-18` | Clearing unread badges | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 | - | 20 | verified | 3 |
| `SHOP-19` | Rolling sales counters | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 | - | 20 | verified-with-fixes | 2 |
| `SHOP-20` | Carousel placeholder search | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 | - | 20 | verified | 3 |
| `SHOP-21` | Product comparison | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 | - | 20 | verified-with-fixes | 2 |
| `SHOP-22` | Pull-down second floor | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit | - | 20 | verified-with-fixes | 2 |
| `SHOP-23` | Delivery address book | AbilityKit, BasicServicesKit, ContactsKit, CoreFileKit +2 | APPROXIMATELY_LOCATION, LOCATION | 20 | verified-with-fixes | 3 |
| `SHOP-24` | Shopping industry FAQ node | - | - | n/a | verified | - |

Read the full card at `features/<ID>.md` before writing any code. `verified-with-fixes` means the published HQ document contains at least one defect that the card corrects.
