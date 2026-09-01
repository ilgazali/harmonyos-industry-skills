# Feature catalog

> Generated from `features/*.md`. Source industry: `17_food`, 6 features.
> Do not edit by hand; regenerate it in the review repository.

Kit names are shown without the `@kit.` prefix; the full module specifier is in each feature card.

| ID | Feature | Kits | Permissions | Min API | Status | Findings |
|---|---|---|---|---:|---|---:|
| `FOOD-01` | Food ordering atomic service | AbilityKit, AccountKit, ArkData, ArkTS +8 | LOCATION, APPROXIMATELY_LOCATION +4 | 20 | verified-with-fixes | 14 |
| `FOOD-02` | Recipe app template | AbilityKit, AccountKit, AdsKit, ArkData +9 | INTERNET, GET_NETWORK_INFO +1 | 24 | verified-with-fixes | 11 |
| `FOOD-03` | Key-scenario index | - | - | n/a | verified | - |
| `FOOD-04` | Store address and route | MapKit, AbilityKit, ArkUI, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 5 |
| `FOOD-05` | Custom pull-to-refresh and load-more | ArkUI, AbilityKit, PerformanceAnalysisKit, CoreFileKit | - | 20 | verified-with-fixes | 3 |
| `FOOD-06` | Food industry FAQ page | - | - | n/a | verified | - |

Read the full card at `features/<ID>.md` before writing any code. `verified-with-fixes` means the published HQ document contains at least one defect that the card corrects.
