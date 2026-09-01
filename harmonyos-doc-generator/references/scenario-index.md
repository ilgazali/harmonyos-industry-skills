# Scenario index

Every feature Huawei treats as a standalone key scenario, across all 19
industries. Generated from the industry skills; do not edit by hand.

Use it to calibrate two things when documenting your own app:

- **Granularity.** These are the units Huawei ships as one document each.
  If your candidate is much larger, split it. Much smaller, merge it.
- **Naming.** Match the phrasing style: a capability, not a screen name.

Navigational cards (scenario indexes, industry FAQs) are excluded.

## 01_auto
| ID | Scenario | Kits |
|---|---|---|
| `AUTO-01` | Automotive app | ArkUI, MapKit, ScanKit, LocationKit +3 |
| `AUTO-03` | Persistent bottom notification bar via a window-stage sub-window | ArkUI, AbilityKit, PerformanceAnalysisKit |
| `AUTO-04` | Nearby petrol stations | MapKit, LocationKit, AbilityKit, BasicServicesKit +1 |
| `AUTO-05` | One-tap dialling from a half-modal contact sheet | TelephonyKit, AbilityKit, BasicServicesKit, ArkUI +1 |
| `AUTO-06` | Custom dashboard gauge drawn on Canvas | ArkUI, ArkTS |
| `AUTO-07` | Smart autofill of owner details, with a custom licence-plate keyboard | AbilityKit, ArkUI |

## 08_children_education
| ID | Scenario | Kits |
|---|---|---|
| `KIDS-02` | Parent gate | ArkUI, AbilityKit, CryptoArchitectureKit, PerformanceAnalysisKit +1 |
| `KIDS-03` | Handwriting practice board | ArkUI, CoreFileKit, ArkTS, AbilityKit |
| `KIDS-04` | Screen-time limit | ArkUI, ArkData, AbilityKit, PerformanceAnalysisKit +1 |
| `KIDS-05` | Go board and stones | ArkUI |
| `KIDS-06` | Tangram puzzle | ArkUI, AbilityKit, PerformanceAnalysisKit, BasicServicesKit |
| `KIDS-07` | Children's drawing board | ArkUI |
| `KIDS-08` | Poem annotation template | ArkUI, AbilityKit |
| `KIDS-09` | Rubber-band shape drawing | ArkUI |
| `KIDS-10` | Sliding picture puzzle | ArkUI, CryptoArchitectureKit, PerformanceAnalysisKit |
| `KIDS-11` | Data usage monitor | NetworkKit, ArkData, ArkUI, PerformanceAnalysisKit |
| `KIDS-12` | Child location and trail | MapKit, LocationKit, ArkUI, BasicServicesKit +1 |
| `KIDS-13` | Vocabulary cards | MediaKit, ArkUI, AbilityKit, BasicServicesKit +1 |
| `KIDS-14` | Distance-from-home alarm | LocationKit, MapKit, ArkUI, AbilityKit +2 |
| `KIDS-15` | Schulte grid | ArkUI, CryptoArchitectureKit |
| `KIDS-16` | Dice roller | CryptoArchitectureKit, LocalizationKit, ArkUI, @ohos/gif-drawable |

## 19_common_technical_solutions
| ID | Scenario | Kits |
|---|---|---|
| `COMMON-01` | Key scenario catalogue | - |
| `COMMON-02` | Layered modular architecture | - |
| `COMMON-03` | Navigation design practice | ArkUI |
| `COMMON-04` | Performance optimisation practice | ArkUI, ArkTS, ArkWeb |
| `COMMON-05` | HarmonyOS porting analysis | ArkTS, ArkWeb |
| `COMMON-06` | Huawei Pay checkout | PaymentKit, AbilityKit, BasicServicesKit |
| `COMMON-07` | Application cache clearing | CoreFileKit, AbilityKit, BasicServicesKit, PerformanceAnalysisKit +1 |
| `COMMON-08` | Feature guide overlay | ArkUI, AbilityKit, PerformanceAnalysisKit, BasicServicesKit |
| `COMMON-09` | H5 side-swipe back interception | ArkWeb, ArkUI, AbilityKit, PerformanceAnalysisKit +1 |
| `COMMON-10` | Personalised launch guide | ArkUI, AbilityKit, PerformanceAnalysisKit, BasicServicesKit |
| `COMMON-11` | App theme edition switch | ArkUI, AbilityKit, PerformanceAnalysisKit, BasicServicesKit |
| `COMMON-12` | Custom theme colours | ArkUI, AbilityKit, PerformanceAnalysisKit, BasicServicesKit |
| `COMMON-13` | User identity authentication | UserAuthenticationKit, PreviewKit, CoreFileKit, BasicServicesKit +3 |
| `COMMON-14` | Picture upload and preview | ArkWeb, MediaLibraryKit, BasicServicesKit, PerformanceAnalysisKit +1 |
| `COMMON-15` | H5 image and file upload | ArkWeb, MediaLibraryKit, CameraKit, CoreFileKit +3 |
| `COMMON-16` | TabBar background blur | ArkUI |
| `COMMON-17` | Silent login | ArkData, ArkUI, AbilityKit, PerformanceAnalysisKit +1 |
| `COMMON-18` | Load-time and call-count instrumentation | ArkTS, NetworkKit, ArkData, ImageKit +3 |
| `COMMON-19` | BLE device scan and connect | ConnectivityKit, AbilityKit, BasicServicesKit, PerformanceAnalysisKit +1 |
| `COMMON-20` | Screenshot detection and sharing | ArkUI, ImageKit, MediaLibraryKit, CoreFileKit +5 |
| `COMMON-21` | File download management | BasicServicesKit, CoreFileKit, ArkTS, ArkUI +2 |
| `COMMON-22` | Page tracing points | ArkUI, ArkTS, AbilityKit, PerformanceAnalysisKit |
| `COMMON-23` | H5 scan integration | ScanKit, ArkWeb, AbilityKit, ArkUI +2 |
| `COMMON-24` | Card-to-fullscreen shared element transition | ArkUI |
| `COMMON-25` | Accessibility screen reading | ArkUI |
| `COMMON-26` | Splash screen ad integration | AdsKit, ArkUI, AbilityKit, ArkData +2 |
| `COMMON-27` | Blur the home page until login | ArkUI |
| `COMMON-28` | H5 long-press on an image | ArkWeb, MediaLibraryKit, CoreFileKit, ArkUI +2 |
| `COMMON-29` | User agreement and privacy policy consent | ArkUI, AbilityKit |
| `COMMON-30` | Custom TabBar switch animation | ArkUI |
| `COMMON-31` | Full-application page inside a two-column layout | ArkUI, AbilityKit, PerformanceAnalysisKit |
| `COMMON-32` | Foldable split-column adaptation | ArkUI, AbilityKit, ArkTS, PerformanceAnalysisKit |
| `COMMON-33` | Custom font in an H5 page | ArkWeb, ArkUI, BasicServicesKit, PerformanceAnalysisKit |
| `COMMON-34` | Launch another app from an H5 page by URL scheme | ArkWeb, AbilityKit, ArkUI, BasicServicesKit +1 |
| `COMMON-35` | Category title to content linkage | ArkUI |
| `COMMON-36` | Foldable web layout switching | ArkWeb, ArkUI, AbilityKit, PerformanceAnalysisKit |
| `COMMON-37` | Tap-the-characters-in-order human verification | ArkUI, BasicServicesKit |
| `COMMON-38` | Slide-puzzle human verification | ArkUI |
| `COMMON-39` | Privacy mode for an H5 login page | ArkWeb, ArkUI, AbilityKit, BasicServicesKit +1 |
| `COMMON-40` | In-app language switch | LocalizationKit, ArkUI |
| `COMMON-41` | Automatic page grayscale | ArkUI, ArkTS |
| `COMMON-42` | Dark and light mode | AbilityKit, ArkUI, ArkTS, LocalizationKit +2 |
| `COMMON-43` | Custom common events | BasicServicesKit, ArkUI, PerformanceAnalysisKit |
| `COMMON-44` | Long-running task notifications | BackgroundTasksKit, NotificationKit, AbilityKit, NetworkKit +3 |
| `COMMON-45` | Connection animation | NetworkKit, ArkUI, ArkTS, BasicServicesKit +1 |
| `COMMON-46` | Scroll-to-top for H5 pages | ArkWeb, ArkUI |
| `COMMON-47` | Charset conversion with ICU4C | ArkTS, ArkUI, IMEKit |
| `COMMON-48` | Wi-Fi scan and connect | ConnectivityKit, ArkUI, ArkTS |
| `COMMON-49` | PC system tray | DeskTopExtensionKit, AbilityKit, ImageKit, BasicServicesKit +2 |
| `COMMON-50` | Navigation continuation | AbilityKit, ArkData, ArkUI, BasicServicesKit +1 |
| `COMMON-51` | Persisted file permissions | CoreFileKit, ArkData, ArkTS, ArkUI +1 |

## 02_convenient_life
| ID | Scenario | Kits |
|---|---|---|
| `LIFE-01` | Government-service app shell | AbilityKit, ArkUI, ArkWeb, ScanKit +7 |
| `LIFE-03` | Chinese licence-plate keyboard | ArkUI, AbilityKit, ArkTS, PerformanceAnalysisKit |
| `LIFE-04` | Cinema seat picker | ArkUI, AbilityKit, ArkTS, PerformanceAnalysisKit +1 |
| `LIFE-05` | Three-column cascading category picker | ArkUI, AbilityKit, ArkTS, PerformanceAnalysisKit |
| `LIFE-06` | Drag-to-reorder to-do list | ArkUI, AbilityKit, ArkTS, PerformanceAnalysisKit +1 |
| `LIFE-07` | Expandable bill list | ArkUI, AbilityKit, ArkTS, PerformanceAnalysisKit |
| `LIFE-08` | Combined date-and-time wheel | ArkUI, AbilityKit, PerformanceAnalysisKit |
| `LIFE-09` | Infinite icon carousel | ArkUI, AbilityKit, PerformanceAnalysisKit |
| `LIFE-10` | Password vault | AssetStoreKit, UserAuthenticationKit, ArkUI, ArkTS +2 |
| `LIFE-11` | Pinch to switch card view | ArkUI, AbilityKit, PerformanceAnalysisKit |
| `LIFE-12` | Perpetual calendar template | ArkUI |
| `LIFE-13` | Three-day calendar | ArkUI, AbilityKit, PerformanceAnalysisKit |
| `LIFE-14` | Map with a docked half-modal | MapKit, LocationKit, AbilityKit, ArkUI +1 |
| `LIFE-15` | Two-level ID-type picker | ArkUI, BasicServicesKit, AbilityKit, PerformanceAnalysisKit |
| `LIFE-16` | Route planning between two searched addresses | MapKit, AbilityKit, BasicServicesKit |
| `LIFE-17` | Two-way linked category lists | ArkUI, AbilityKit |
| `LIFE-18` | Week / month / detail calendar | ArkUI, ArkTS, LocalizationKit, AbilityKit |
| `LIFE-19` | Paste an address, split it into fields | NaturalLanguageKit, BasicServicesKit, IMEKit, ArkUI +2 |
| `LIFE-20` | Rotate the map to the heading, then travel along it | MapKit, AbilityKit, ArkUI, BasicServicesKit +1 |
| `LIFE-21` | Photograph an address label and fill the form | CoreVisionKit, NaturalLanguageKit, MediaLibraryKit, CameraKit +5 |
| `LIFE-22` | Scan an ID card with a live custom camera | CameraKit, ImageKit, CoreVisionKit, NaturalLanguageKit +5 |
| `LIFE-23` | Real-name registration with the CardRecognition system control | VisionKit, ArkUI, PerformanceAnalysisKit, BasicServicesKit +1 |
| `LIFE-24` | Book a home-service slot and write it to the calendar | CalendarKit, AbilityKit, ArkUI, ArkTS +2 |
| `LIFE-25` | Show the commute from a listing to work | MapKit, ArkUI, BasicServicesKit, PerformanceAnalysisKit +1 |
| `LIFE-26` | Find nearby service outlets and act on one | MapKit, LocationKit, TelephonyKit, AbilityKit +4 |
| `LIFE-27` | Grouped photo preview with a linked category strip | ArkUI, AbilityKit, BasicServicesKit, PerformanceAnalysisKit |
| `LIFE-28` | Let an embedded H5 page pick a contact | ArkWeb, ContactsKit, ArkUI, AbilityKit +1 |
| `LIFE-29` | Draw a draggable coverage circle on the map | MapKit, ArkUI, AbilityKit, BasicServicesKit +1 |
| `LIFE-30` | Batch thumbnail generation off the UI thread | ArkTS, ImageKit, CoreFileKit, MediaLibraryKit +3 |

## 04_education
| ID | Scenario | Kits |
|---|---|---|
| `EDU-01` | Education app framework | AbilityKit, ArkUI, MediaKit, BasicServicesKit +4 |
| `EDU-03` | Score-range dual slider | ArkUI, AbilityKit, PerformanceAnalysisKit, BasicServicesKit |
| `EDU-04` | Two-axis timetable | ArkUI, AbilityKit, PerformanceAnalysisKit, BasicServicesKit |
| `EDU-05` | Expand and collapse a biography | ArkUI, AbilityKit, PerformanceAnalysisKit |
| `EDU-06` | Stacked word cards | ArkUI, AbilityKit, PerformanceAnalysisKit, BasicServicesKit |
| `EDU-07` | Swipeable question practice | ArkUI, AbilityKit, ArkTS, PerformanceAnalysisKit +1 |
| `EDU-08` | PDF to one long image | PDFKit, ImageKit, MediaLibraryKit, CoreFileKit +3 |
| `EDU-09` | Import and open a PDF | CoreFileKit, PDFKit, AbilityKit, ArkUI |
| `EDU-10` | Bullet comments over a course video | ArkUI, AbilityKit, CryptoArchitectureKit, ArkTS +1 |
| `EDU-11` | Read-aloud practice | MediaKit, AbilityKit, CoreFileKit, LocalizationKit +3 |
| `EDU-12` | Live network status and speed | NetworkKit, NetworkBoostKit, ArkTS, BasicServicesKit +1 |
| `EDU-13` | Course reminders in the system calendar | CalendarKit, ArkData, AbilityKit, BasicServicesKit +1 |
| `EDU-14` | Course reading progress | ArkUI, ArkTS, AbilityKit, LocalizationKit |
| `EDU-15` | Three-column cascading picker | ArkUI, ArkTS, AbilityKit |
| `EDU-16` | Camera and microphone self-check | CameraKit, MediaKit, AbilityKit, CoreFileKit +2 |
| `EDU-17` | Word spelling drill | ArkUI, MediaKit, LocalizationKit, ArkTS +1 |
| `EDU-18` | Study-progress line chart | ArkUI, AbilityKit |
| `EDU-19` | Spherical word cloud | ArkUI |
| `EDU-20` | Scan and submit homework | VisionKit, CoreFileKit, PDFKit, ArkUI +2 |

## 07_finance_insurance
| ID | Scenario | Kits |
|---|---|---|
| `FIN-01` | Insurance app | ArkUI, VisionKit, AbilityKit, LocationKit +1 |
| `FIN-03` | Stock charts | ArkUI, ArkTS, LocalizationKit |
| `FIN-04` | Daily income and expenditure calendar | ArkUI |
| `FIN-05` | Stock-code keyboard | ArkUI |
| `FIN-06` | App lock | ArkUI, CryptoArchitectureKit, ArkData, ArkTS |
| `FIN-07` | Loan calculator | ArkUI, PerformanceAnalysisKit |
| `FIN-08` | Bank card number | BasicServicesKit, ArkUI, IMEKit, AbilityKit |
| `FIN-09` | Randomised-order custom keyboard for password entry | ArkUI, CryptoArchitectureKit, BasicServicesKit, AbilityKit |
| `FIN-10` | Expenditure distribution pie chart with mpchart | ArkUI |

## 17_food
| ID | Scenario | Kits |
|---|---|---|
| `FOOD-01` | Food ordering atomic service | AbilityKit, AccountKit, ArkData, ArkTS +8 |
| `FOOD-02` | Recipe app template | AbilityKit, AccountKit, AdsKit, ArkData +9 |
| `FOOD-04` | Store address and route | MapKit, AbilityKit, ArkUI, PerformanceAnalysisKit |
| `FOOD-05` | Custom pull-to-refresh and load-more | ArkUI, AbilityKit, PerformanceAnalysisKit, CoreFileKit |

## 12_jobs
| ID | Scenario | Kits |
|---|---|---|
| `JOBS-02` | In-app notification authorization | NotificationKit, AbilityKit, ArkUI, BasicServicesKit +1 |
| `JOBS-03` | Stacked job cards | ArkUI, AbilityKit, BasicServicesKit, PerformanceAnalysisKit |
| `JOBS-04` | Job detail to recruiter to company job list | ArkUI, AbilityKit, BasicServicesKit, PerformanceAnalysisKit |

## 10_maternity_health
| ID | Scenario | Kits |
|---|---|---|
| `MAT-01` | Maternity and child health app | ArkUI, ArkTS, AbilityKit, MediaLibraryKit +3 |
| `MAT-03` | Baby growth record timeline | ArkUI, MediaLibraryKit, CoreFileKit, AbilityKit +1 |
| `MAT-04` | Child growth curve chart | ArkUI, ArkTS, AbilityKit |

## 13_media_entertainment
| ID | Scenario | Kits |
|---|---|---|
| `MEDIA-01` | Music app architecture | MediaKit, AbilityKit, ArkData, ArkUI +6 |
| `MEDIA-03` | Video metadata card | MediaLibraryKit, ImageKit, ArkUI, AbilityKit +2 |
| `MEDIA-04` | Video compression | MediaKit, MediaLibraryKit, ArkUI, AbilityKit +3 |
| `MEDIA-05` | Player gesture sliders | ArkUI, AbilityKit, BasicServicesKit, PerformanceAnalysisKit +1 |
| `MEDIA-06` | Parallel tag filtering | ArkUI, AbilityKit, BasicServicesKit, PerformanceAnalysisKit |
| `MEDIA-07` | Floating mini-player | ArkUI, AbilityKit, MediaKit, AudioKit +3 |
| `MEDIA-08` | Landscape/portrait video page | ArkUI, AbilityKit, PerformanceAnalysisKit |
| `MEDIA-09` | Cellular switch reminder | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `MEDIA-10` | Karaoke lyrics | AbilityKit, ArkTS, ArkUI, BasicServicesKit +2 |
| `MEDIA-11` | Audio output switching without AVSession | AVSessionKit, AbilityKit, ArkTS, ArkUI +5 |
| `MEDIA-12` | Video playlist linkage | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `MEDIA-13` | Import local music files | AbilityKit, ArkData, ArkUI, AudioKit +5 |
| `MEDIA-14` | Video segment to GIF | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +4 |
| `MEDIA-15` | Bullet comments (danmaku) | AbilityKit, ArkTS, ArkUI, CoreFileKit +1 |
| `MEDIA-16` | Short-video comment sheet | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `MEDIA-17` | Playback speed control | AbilityKit, ArkUI, BasicServicesKit, PerformanceAnalysisKit |
| `MEDIA-18` | Auto-playing video feed | AbilityKit, ArkTS, ArkUI, BasicServicesKit +4 |
| `MEDIA-19` | Moving-photo carousel | AbilityKit, ArkData, ArkUI, BasicServicesKit +3 |
| `MEDIA-20` | Video concatenation | AbilityKit, ArkData, ArkUI, BasicServicesKit +4 |
| `MEDIA-21` | Multi-camera video wall | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `MEDIA-22` | Video like burst | AbilityKit, ArkUI, CoreFileKit, CryptoArchitectureKit +1 |
| `MEDIA-23` | Extract the audio track from a picked video | AbilityKit, ArkData, ArkTS, ArkUI +5 |
| `MEDIA-24` | Auto-pause and resume around an audio interruption | AbilityKit, ArkUI, AudioKit, BasicServicesKit +4 |
| `MEDIA-25` | Navigation splash page | AbilityKit, ArkData, ArkUI, BasicServicesKit +4 |
| `MEDIA-26` | Seamless list-to-detail video | AbilityKit, ArkUI, BasicServicesKit, MediaKit +1 |
| `MEDIA-27` | Scrolling audio waveform | AbilityKit, ArkTS, ArkUI, AudioKit +3 |
| `MEDIA-28` | Buffered progress bar | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `MEDIA-29` | Speed lock | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `MEDIA-30` | Download a track over HTTP and play it through a native OHAudio renderer | AbilityKit, ArkUI, AudioKit, BasicServicesKit +2 |
| `MEDIA-31` | Drum pad | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `MEDIA-32` | Video screenshot with frame stepping | AVSessionKit, AbilityKit, ArkUI, BackgroundTasksKit +4 |
| `MEDIA-33` | Mirror video playback | AVSessionKit, AbilityKit, ArkUI, BackgroundTasksKit +3 |
| `MEDIA-34` | Background m3u8 download | AbilityKit, ArkTS, ArkUI, BackgroundTasksKit +6 |
| `MEDIA-35` | PCM audio editing | AbilityKit, ArkTS, ArkUI, AudioKit +2 |
| `MEDIA-36` | Native PCM transcode | AbilityKit, ArkTS, ArkUI, AudioKit +4 |
| `MEDIA-37` | Offscreen video render | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `MEDIA-38` | Metronome | AbilityKit, ArkTS, ArkUI, AudioKit +4 |
| `MEDIA-39` | Scrub preview | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `MEDIA-40` | Image + text compositing | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `MEDIA-41` | Automatic video subtitles | AbilityKit, ArkTS, ArkUI, BasicServicesKit +4 |
| `MEDIA-42` | Pause on headset disconnect | AVSessionKit, AbilityKit, ArkUI, CoreFileKit +3 |

## 11_news_reading
| ID | Scenario | Kits |
|---|---|---|
| `NEWS-01` | News app skeleton | AbilityKit, ArkData, ArkTS, ArkUI +6 |
| `NEWS-03` | Minors mode | AbilityKit, AccountKit, ArkData, ArkUI +2 |
| `NEWS-04` | Channel subscription editor | AbilityKit, ArkUI, PerformanceAnalysisKit |
| `NEWS-05` | In-app font size | AbilityKit, ArkData, ArkUI, BasicServicesKit +1 |
| `NEWS-06` | Save a Base64 image to the gallery | AbilityKit, ArkTS, ArkUI, MediaLibraryKit +1 |
| `NEWS-07` | Three page-turn modes in one reader | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `NEWS-08` | Back to top on a long list | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `NEWS-09` | AI read-aloud | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `NEWS-10` | Ads in a lazy feed | AbilityKit, ArkTS, ArkUI, BasicServicesKit +2 |
| `NEWS-11` | Hot-search board | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `NEWS-12` | Volume-key page turning | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 |
| `NEWS-13` | Text highlight marking | AbilityKit, ArkData, ArkTS, ArkUI +3 |
| `NEWS-14` | Preset database | AbilityKit, ArkData, ArkUI, BasicServicesKit +3 |
| `NEWS-15` | Typewriter effect | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `NEWS-16` | H5 font size | AbilityKit, ArkData, ArkUI, ArkWeb +2 |
| `NEWS-17` | Regex keyword highlight | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `NEWS-18` | Reading magnifier | AbilityKit, ArkUI, CoreFileKit, PerformanceAnalysisKit |
| `NEWS-19` | Smear to recognise | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 |
| `NEWS-20` | Reading-progress widget | AbilityKit, ArkData, ArkUI, BasicServicesKit +3 |
| `NEWS-21` | Reading-time dashboard | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `NEWS-22` | Bookshelf with local import | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `NEWS-23` | Per-paragraph comments | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `NEWS-24` | Rich-text post editor | AbilityKit, ArkTS, ArkUI, ArkWeb +4 |
| `NEWS-25` | Auto page turn | AbilityKit, ArkTS, ArkUI, BasicServicesKit +3 |
| `NEWS-26` | Sleep timer for text-to-speech | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |

## 05_office
| ID | Scenario | Kits |
|---|---|---|
| `OFFICE-01` | Integrated office app shell | AbilityKit, ArkUI, ArkData, ArkTS +7 |
| `OFFICE-03` | Attendance check-in location | AbilityKit, LocationKit, ArkUI, BasicServicesKit +1 |
| `OFFICE-04` | Secure online PDF preview | ArkWeb, ArkUI, AbilityKit, BasicServicesKit +1 |
| `OFFICE-05` | Personal card page | ScanKit, ImageKit, MediaLibraryKit, CoreFileKit +5 |
| `OFFICE-06` | Document approval | ArkUI, CoreFileKit, PreviewKit, BasicServicesKit +2 |
| `OFFICE-07` | ID photo capture with a mask overlay | CameraKit, MediaLibraryKit, AbilityKit, ArkUI +2 |
| `OFFICE-08` | File download and preview | PreviewKit, BasicServicesKit, AbilityKit, ArkUI +1 |
| `OFFICE-09` | Visitor management | ContactsKit, ScanKit, ArkUI, AbilityKit +2 |
| `OFFICE-10` | Mail attachments | MediaLibraryKit, CameraKit, CoreFileKit, PreviewKit +4 |
| `OFFICE-11` | Publish a meeting from custom dialogs and write it into the system calendar | CalendarKit, CoreFileKit, PreviewKit, AbilityKit +4 |
| `OFFICE-12` | Electronic seal on a PDF | PDFKit, CoreFileKit, AbilityKit, ArkUI +2 |
| `OFFICE-13` | Open a file with another app | AbilityKit, CoreFileKit, ArkUI, BasicServicesKit +1 |
| `OFFICE-14` | Marquee banner notice | ArkUI, PerformanceAnalysisKit |
| `OFFICE-15` | Handle a message later | ArkUI, ArkData, AbilityKit, BasicServicesKit +1 |
| `OFFICE-16` | ID-photo recommendation | MediaLibraryKit, ArkUI, AbilityKit, BasicServicesKit +1 |
| `OFFICE-17` | Multi-level corporate directory | ArkUI, ArkTS, PerformanceAnalysisKit |
| `OFFICE-18` | Online meeting main/sub window swap | ArkUI, CameraKit, AbilityKit, ArkData +2 |
| `OFFICE-19` | Watermark camera | CameraKit, LocationKit, ImageKit, MediaLibraryKit +4 |
| `OFFICE-20` | Insert and edit an image in a note | ArkUI, MediaLibraryKit, CameraKit, ImageKit +3 |
| `OFFICE-21` | Voice note record and playback | AudioKit, CoreFileKit, ArkUI, AbilityKit +2 |
| `OFFICE-22` | Add special-attention contacts | ArkUI |
| `OFFICE-23` | Live speech-to-text notes | CoreSpeechKit, AudioKit, CoreFileKit, AbilityKit +3 |
| `OFFICE-24` | Application background watermark | ArkUI |
| `OFFICE-25` | Pinned group announcement | ArkUI |
| `OFFICE-26` | Multi-level organisation menu | ArkUI |
| `OFFICE-27` | Batch-sync to-dos into the system calendar | CalendarKit, AbilityKit, ArkData, ArkUI +2 |
| `OFFICE-28` | Adding and deleting annotations in PDF preview | PDFKit, ArkData, CoreFileKit, AbilityKit +3 |
| `OFFICE-29` | Showing enterprise employee details on the incoming-call screen | CallServiceKit, ArkData, AbilityKit, ArkUI +2 |
| `OFFICE-30` | Warning the candidate when an exam app is switched to the background | ArkUI, NotificationKit, MediaKit, AudioKit +4 |
| `OFFICE-31` | Collaborative schedule management | CalendarKit, ShareKit, AbilityKit, ArkUI +4 |

## 18_photography
| ID | Scenario | Kits |
|---|---|---|
| `PHOTO-01` | Photography app architecture | CameraKit, ImageKit, MediaLibraryKit, ArkGraphics2D +9 |
| `PHOTO-03` | Gallery photo filters | ImageKit, ArkGraphics2D, MediaLibraryKit, CoreFileKit +3 |
| `PHOTO-04` | Photo aspect-ratio switch | CameraKit, MediaLibraryKit, AbilityKit, ArkUI +6 |
| `PHOTO-05` | Template photo collage | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 |
| `PHOTO-06` | Camera Kit + AVRecorder video capture | AbilityKit, ArkData, ArkUI, BasicServicesKit +5 |
| `PHOTO-07` | Custom camera zoom | AbilityKit, ArkUI, BasicServicesKit, CameraKit +4 |
| `PHOTO-08` | Subject segmentation cut-out | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +4 |
| `PHOTO-09` | Quality-tiered image compression | AbilityKit, ArkData, ArkUI, BasicServicesKit +4 |
| `PHOTO-10` | Sticker overlay editor | AbilityKit, ArkUI, CoreFileKit, ImageKit +2 |
| `PHOTO-11` | Face detection with a focus matrix | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +4 |
| `PHOTO-12` | GIF from a picture sequence | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `PHOTO-13` | Capture timer | AbilityKit, ArkUI, BasicServicesKit, CameraKit +4 |
| `PHOTO-14` | Image format conversion | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 |
| `PHOTO-15` | Video trim timeline | AbilityKit, ArkData, ArkUI, BasicServicesKit +5 |
| `PHOTO-16` | Colour adjustment | AbilityKit, ArkUI, CoreFileKit, ImageKit +2 |
| `PHOTO-17` | Video aspect-ratio crop | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 |
| `PHOTO-18` | Salt-and-pepper denoising | AbilityKit, ArkTS, ArkUI, BasicServicesKit +4 |
| `PHOTO-19` | Picture-in-picture video edit | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +4 |
| `PHOTO-20` | Rotate and flip a still image | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 |
| `PHOTO-21` | Static video watermark | AbilityKit, ArkData, ArkTS, ArkUI +5 |
| `PHOTO-22` | Video format conversion | AbilityKit, ArkUI, BasicServicesKit, MediaKit +2 |
| `PHOTO-23` | Mosaic brush | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 |
| `PHOTO-24` | Ratio crop frame | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 |
| `PHOTO-25` | Delayed video recording | AbilityKit, ArkData, ArkTS, ArkUI +8 |
| `PHOTO-26` | Preview resolution switch | AbilityKit, ArkData, ArkGraphics2D, ArkUI +5 |
| `PHOTO-27` | Camera preview across background switches | AbilityKit, ArkGraphics2D, ArkUI, BasicServicesKit +3 |
| `PHOTO-28` | Oil-painting and pencil-sketch filters | AbilityKit, ArkTS, ArkUI, CoreFileKit +3 |
| `PHOTO-29` | Gravity-sensor orientation | AbilityKit, ArkUI, BasicServicesKit, CameraKit +5 |
| `PHOTO-30` | Keep recording in a picture-in-picture window | AbilityKit, ArkData, ArkTS, ArkUI +7 |
| `PHOTO-31` | Preview resolution switch | AbilityKit, ArkData, ArkGraphics2D, ArkUI +6 |

## 06_public_transport
| ID | Scenario | Kits |
|---|---|---|
| `TRANS-01` | Transit and navigation app | MapKit, LocationKit, AbilityKit, ArkData +2 |
| `TRANS-03` | Home-screen widget that opens the ride code | ArkUI, AbilityKit, FormKit, PerformanceAnalysisKit +1 |
| `TRANS-04` | Real-time bus arrivals | LocationKit, AbilityKit, BasicServicesKit, ArkUI |
| `TRANS-05` | Ride history with a date-range filter | ArkUI |
| `TRANS-06` | Pin an in-app shortcut to the home screen | StoreKit, AbilityKit, ArkUI |
| `TRANS-07` | Boarding and alighting reminders with Notification Kit | NotificationKit, ArkUI, BasicServicesKit |
| `TRANS-08` | Map rotation lock, zoom stepping and locate-me | MapKit, LocationKit, AbilityKit, ArkUI +1 |

## 16_shopping
| ID | Scenario | Kits |
|---|---|---|
| `SHOP-01` | Shopping mall app skeleton | AbilityKit, ArkData, ArkTS, ArkUI +15 |
| `SHOP-03` | Coupon wallet | AbilityKit, ArkTS, ArkUI, BasicServicesKit +1 |
| `SHOP-04` | Daily check-in and points | AbilityKit, ArkData, ArkUI, BasicServicesKit +3 |
| `SHOP-05` | One coupon per device | DeviceSecurityKit, ArkData, BasicServicesKit, PerformanceAnalysisKit +2 |
| `SHOP-06` | Pull-down to navigate | ArkUI, ArkWeb, AbilityKit, BasicServicesKit +4 |
| `SHOP-07` | Scratch card | ArkUI, ImageKit, PerformanceAnalysisKit, AbilityKit +2 |
| `SHOP-08` | Long-press to mark a product | ArkUI, AbilityKit, BasicServicesKit, PerformanceAnalysisKit +1 |
| `SHOP-09` | Editable search history | AbilityKit, BasicServicesKit |
| `SHOP-10` | Sticky mall home page | AbilityKit, ArkUI, PerformanceAnalysisKit |
| `SHOP-11` | Skeleton screen | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `SHOP-12` | Expand/collapse a wrapped history block | AbilityKit, ArkUI, CoreFileKit, PerformanceAnalysisKit |
| `SHOP-13` | Order status tabs and the post-receipt review page | AbilityKit, ArkData, ArkTS, ArkUI +5 |
| `SHOP-14` | Self-sizing quick-entry menu | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `SHOP-15` | Browsing history that survives a restart | AbilityKit, ArkUI, CoreFileKit, PerformanceAnalysisKit |
| `SHOP-16` | Ticket-drop countdown with a calendar reminder | AbilityKit, ArkUI, BasicServicesKit, CalendarKit +2 |
| `SHOP-17` | Red envelope rain | AbilityKit, ArkUI, CoreFileKit, PerformanceAnalysisKit |
| `SHOP-18` | Clearing unread badges | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `SHOP-19` | Rolling sales counters | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `SHOP-20` | Carousel placeholder search | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `SHOP-21` | Product comparison | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `SHOP-22` | Pull-down second floor | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit |
| `SHOP-23` | Delivery address book | AbilityKit, BasicServicesKit, ContactsKit, CoreFileKit +2 |

## 14_social_communication
| ID | Scenario | Kits |
|---|---|---|
| `SOCIAL-02` | Full-screen image preview | AbilityKit, ArkUI, BasicServicesKit, PerformanceAnalysisKit |
| `SOCIAL-03` | Hold-to-talk voice input | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +4 |
| `SOCIAL-04` | Friend-match reveal | AbilityKit, ArkUI, PerformanceAnalysisKit |
| `SOCIAL-05` | Recommended photo picking | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `SOCIAL-06` | Nearby people | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 |
| `SOCIAL-07` | In-chat vote card | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `SOCIAL-08` | Send original or compressed | AbilityKit, ArkTS, ArkUI, BasicServicesKit +4 |
| `SOCIAL-09` | Contacts list grouped by pinyin initial | AbilityKit, ArkUI, PerformanceAnalysisKit, TelephonyKit |
| `SOCIAL-10` | Avatar crop and profile edit | AbilityKit, ArkData, ArkTS, ArkUI +5 |
| `SOCIAL-11` | SM2 message encryption in a chat | AbilityKit, ArkTS, ArkUI, BasicServicesKit +3 |
| `SOCIAL-12` | Chat link interception | AbilityKit, ArkUI, ArkWeb, BasicServicesKit +1 |
| `SOCIAL-13` | Unread badges on a chat list | AbilityKit, ArkTS, ArkUI, BasicServicesKit +2 |
| `SOCIAL-14` | Quoting a chat message | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `SOCIAL-15` | Adding friends from the address book | AbilityKit, ArkUI, BasicServicesKit, ContactsKit +1 |
| `SOCIAL-16` | Drag-and-drop send | AbilityKit, ArkData, ArkUI, BasicServicesKit +3 |
| `SOCIAL-17` | Bluetooth SPP chat | AbilityKit, ArkTS, ArkUI, BasicServicesKit +2 |
| `SOCIAL-18` | Post draft autosave | AbilityKit, ArkData, ArkTS, ArkUI +6 |
| `SOCIAL-19` | Group solitaire | AbilityKit, ArkTS, ArkUI, CoreFileKit +1 |
| `SOCIAL-20` | Weak-network chat | AbilityKit, ArkTS, ArkUI, BasicServicesKit +3 |
| `SOCIAL-21` | Voice call screen | AbilityKit, AudioKit, BasicServicesKit, CoreFileKit +1 |
| `SOCIAL-22` | History float bubble | AbilityKit, ArkUI, ArkWeb, BasicServicesKit +3 |
| `SOCIAL-23` | Voice message to text | AbilityKit, ArkUI, AudioKit, BasicServicesKit +3 |
| `SOCIAL-24` | Long-press a chat image to read its QR code | AbilityKit, ArkUI, ArkWeb, BasicServicesKit +3 |
| `SOCIAL-25` | Emoji pack association | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `SOCIAL-26` | Location card in chat | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `SOCIAL-27` | Chat bubble backgrounds | AbilityKit, ArkUI, CoreFileKit, IMEKit +1 |
| `SOCIAL-28` | App message centre | AbilityKit, ArkTS, ArkUI, BasicServicesKit +2 |
| `SOCIAL-29` | In-app photo picker | AbilityKit, ArkData, ArkUI, BasicServicesKit +3 |
| `SOCIAL-30` | Inertial long-image scrolling | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 |
| `SOCIAL-31` | Phone numbers inside a message | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `SOCIAL-32` | Keyword search in chat history | AbilityKit, ArkUI, BasicServicesKit, PerformanceAnalysisKit |
| `SOCIAL-33` | Chat multi-select and forward | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `SOCIAL-34` | Nine-grid drag sorting | AbilityKit, ArkData, ArkTS, ArkUI +5 |
| `SOCIAL-35` | Back-to-latest pill | AbilityKit, ArkTS, ArkUI, BasicServicesKit +2 |
| `SOCIAL-36` | Quick-reply panel | AbilityKit, ArkTS, ArkUI, BasicServicesKit +2 |
| `SOCIAL-37` | Emoji reactions on a message | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `SOCIAL-38` | Personalised QR code | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 |
| `SOCIAL-39` | Send the most recent photo | AbilityKit, ArkTS, ArkUI, BasicServicesKit +3 |
| `SOCIAL-40` | Pinned chats and a fold row | AbilityKit, ArkTS, ArkUI, BasicServicesKit +2 |
| `SOCIAL-41` | Receive a shared image | AbilityKit, ArkTS, ArkUI, BasicServicesKit +2 |
| `SOCIAL-42` | Send a contact card | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `SOCIAL-43` | Send an encrypted file in chat | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 |
| `SOCIAL-44` | WebSocket chat client | AbilityKit, ArkUI, CoreFileKit, PerformanceAnalysisKit |

## 03_sports_health
| ID | Scenario | Kits |
|---|---|---|
| `SPORT-01` | Sports and health app | ConnectivityKit, SensorServiceKit, AbilityKit, NotificationKit +4 |
| `SPORT-03` | Hold-to-finish a workout | ArkUI, ArkTS |
| `SPORT-04` | Workout plans on a custom calendar | CalendarKit, ArkData, AbilityKit, ArkUI +2 |
| `SPORT-05` | Periodic health charts | LocalizationKit, ArkTS, AbilityKit, ArkUI |
| `SPORT-06` | Match scorer | ArkUI |
| `SPORT-07` | The three activity rings | ArkUI |
| `SPORT-08` | Fit a whole route on screen | MapKit, BasicServicesKit, ArkUI, PerformanceAnalysisKit |
| `SPORT-09` | Publish a group activity | ArkUI, MediaLibraryKit, PerformanceAnalysisKit, BasicServicesKit |
| `SPORT-10` | Knockout bracket | ArkUI, CryptoArchitectureKit |
| `SPORT-11` | Live GPS track | LocationKit, MapKit, AbilityKit, ArkUI |
| `SPORT-12` | The 3-2-1 countdown | ArkUI |
| `SPORT-13` | Reorder dashboard cards by drag | ArkUI, @ohos.curves |
| `SPORT-14` | Scan a medicine barcode | ScanKit, ArkUI, LocalizationKit, BasicServicesKit |

## 09_tourism
| ID | Scenario | Kits |
|---|---|---|
| `TOUR-01` | Tourist park app | MapKit, LocationKit, AbilityKit, BasicServicesKit +5 |
| `TOUR-03` | Location permission bubble | AbilityKit, PerformanceAnalysisKit, BasicServicesKit, ArkUI |
| `TOUR-04` | Date range picker | ArkTS, PerformanceAnalysisKit, ArkUI |
| `TOUR-05` | Origin and destination swap | ArkUI, @ohos.curves |
| `TOUR-06` | Destination map with nearby POIs | MapKit, BasicServicesKit, ArkUI, AbilityKit +1 |
| `TOUR-07` | Named map markers | MapKit, BasicServicesKit, ArkUI |
| `TOUR-08` | Hotel stay review | MediaLibraryKit, ArkUI, BasicServicesKit, PerformanceAnalysisKit |
| `TOUR-09` | Attraction audio guide | MediaKit, AbilityKit, BasicServicesKit, PerformanceAnalysisKit +1 |
| `TOUR-10` | Hotel order list | ArkTS, BasicServicesKit, ArkUI, PerformanceAnalysisKit |
| `TOUR-11` | Traveller details form | ArkUI |
| `TOUR-12` | Long-press to drop a marker, reverse-geocode it and save the address | MapKit, LocationKit, BasicServicesKit, ArkUI +1 |

## 15_utilities
| ID | Scenario | Kits |
|---|---|---|
| `UTIL-01` | Print app skeleton | AbilityKit, ArkData, ArkTS, ArkUI +8 |
| `UTIL-03` | Radar scan animation | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `UTIL-04` | Azimuth and included-angle dial | AbilityKit, ArkUI, CoreFileKit, PerformanceAnalysisKit |
| `UTIL-05` | Canvas speed gauge | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `UTIL-06` | Ringtone setting | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `UTIL-07` | Bottom sheet into side drawer | AbilityKit, ArkUI, BasicServicesKit, PerformanceAnalysisKit |
| `UTIL-08` | Web page poster | AbilityKit, ArkData, ArkUI, ArkWeb +5 |
| `UTIL-09` | Network status panel | AbilityKit, ArkUI, BasicServicesKit, NetworkKit +1 |
| `UTIL-10` | Banner-driven background | AbilityKit, ArkGraphics2D, ArkUI, BasicServicesKit +4 |
| `UTIL-11` | Device-info home-screen widgets | AbilityKit, ArkData, ArkTS, ArkUI +5 |
| `UTIL-12` | In-app floating tool ball | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `UTIL-13` | Immersive keyboard | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `UTIL-14` | Rate this app | AbilityKit, AppGalleryKit, ArkUI, BasicServicesKit +2 |
| `UTIL-15` | Auto-advancing form | AbilityKit, ArkUI, BasicServicesKit, PerformanceAnalysisKit |
| `UTIL-16` | TaskPool file scan | AbilityKit, ArkTS, ArkUI, CoreFileKit +1 |
| `UTIL-17` | VIN scanner | AbilityKit, ArkTS, ArkUI, BasicServicesKit +5 |
| `UTIL-18` | Join Wi-Fi from a QR code | AbilityKit, ArkUI, BasicServicesKit, ConnectivityKit +3 |
| `UTIL-19` | Unzip a file two ways | AbilityKit, ArkTS, ArkUI, BasicServicesKit +2 |
| `UTIL-20` | Calculator | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `UTIL-21` | NFC tag read and write | AbilityKit, ArkTS, ArkUI, BasicServicesKit +3 |
| `UTIL-22` | Compass | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 |
| `UTIL-23` | Spirit level | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `UTIL-24` | Custom-start timer | AbilityKit, ArkUI, CoreFileKit, PerformanceAnalysisKit |
| `UTIL-25` | Drag-to-rearrange note board | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 |
| `UTIL-26` | LED banner | AbilityKit, ArkUI, CoreFileKit, PerformanceAnalysisKit |
| `UTIL-27` | Save a Base64 image out of an H5 page | AbilityKit, ArkTS, ArkUI, ArkWeb +4 |
| `UTIL-28` | Date calculator | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 |
| `UTIL-29` | Three RSA modes | CryptoArchitectureKit, ArkTS, AbilityKit, ArkUI +1 |
| `UTIL-30` | HMAC compute and verify | CryptoArchitectureKit, ArkTS, AbilityKit, ArkUI +3 |
| `UTIL-31` | Batch media download | ArkTS, NetworkKit, MediaLibraryKit, CoreFileKit +4 |
| `UTIL-32` | Live IP address | NetworkKit, ArkUI, BasicServicesKit, AbilityKit +2 |
| `UTIL-33` | TOTP dynamic password | AbilityKit, ArkUI, CoreFileKit, CryptoArchitectureKit +1 |
| `UTIL-34` | Background photo upload on 2in1 | AbilityKit, ArkData, ArkUI, BasicServicesKit +4 |
| `UTIL-35` | Pausable download with rcp | AbilityKit, ArkTS, ArkUI, CoreFileKit +2 |
| `UTIL-36` | Inner audio recording in native code | AbilityKit, ArkTS, ArkUI, BasicServicesKit +3 |
| `UTIL-37` | Photo geotag map | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +5 |
| `UTIL-38` | Geotagged custom camera | AbilityKit, ArkData, ArkGraphics2D, ArkUI +7 |
| `UTIL-39` | Four-up meeting grid | AbilityKit, ArkTS, ArkUI, CoreFileKit +1 |
| `UTIL-40` | Native ping | AbilityKit, ArkTS, ArkUI, BasicServicesKit +2 |
| `UTIL-41` | Decode an image into a chosen pixel format | AbilityKit, ArkUI, BasicServicesKit, CameraKit +5 |
| `UTIL-42` | HCE app link | AbilityKit, ArkTS, ArkUI, BasicServicesKit +3 |
| `UTIL-43` | Local persistence side by side | AbilityKit, ArkData, ArkUI, BasicServicesKit +2 |
| `UTIL-44` | Orbit camera for a loaded 3D model | AbilityKit, ArkGraphics3D, ArkUI, BasicServicesKit +2 |
| `UTIL-45` | Keyboard candidate strip | AbilityKit, ArkUI, IMEKit |
| `UTIL-46` | Poster generator | AbilityKit, ArkGraphics2D, ArkTS, ArkUI +6 |
