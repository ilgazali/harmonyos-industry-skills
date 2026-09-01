# Feature catalog

> Generated from `features/*.md`. Source industry: `13_media_entertainment`, 43 features.
> Do not edit by hand; regenerate it in the review repository.

Kit names are shown without the `@kit.` prefix; the full module specifier is in each feature card.

| ID | Feature | Kits | Permissions | Min API | Status | Findings |
|---|---|---|---|---:|---|---:|
| `MEDIA-01` | Music app architecture | MediaKit, AbilityKit, ArkData, ArkUI +6 | KEEP_BACKGROUND_RUNNING, MICROPHONE | 20 | verified-with-fixes | 7 |
| `MEDIA-02` | Key-scenario index | - | - | n/a | verified | - |
| `MEDIA-03` | Video metadata card | MediaLibraryKit, ImageKit, ArkUI, AbilityKit +2 | READ_IMAGEVIDEO | 20 | verified-with-fixes | 3 |
| `MEDIA-04` | Video compression | MediaKit, MediaLibraryKit, ArkUI, AbilityKit +3 | - | 20 | verified-with-fixes | 4 |
| `MEDIA-05` | Player gesture sliders | ArkUI, AbilityKit, BasicServicesKit, PerformanceAnalysisKit +1 | - | 20 | verified-with-fixes | 2 |
| `MEDIA-06` | Parallel tag filtering | ArkUI, AbilityKit, BasicServicesKit, PerformanceAnalysisKit | - | 24 | verified-with-fixes | 5 |
| `MEDIA-07` | Floating mini-player | ArkUI, AbilityKit, MediaKit, AudioKit +3 | - | 20 | verified-with-fixes | 4 |
| `MEDIA-08` | Landscape/portrait video page | ArkUI, AbilityKit, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 2 |
| `MEDIA-09` | Cellular switch reminder | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 | GET_NETWORK_INFO | 20 | verified-with-fixes | 3 |
| `MEDIA-10` | Karaoke lyrics | AbilityKit, ArkTS, ArkUI, BasicServicesKit +2 | - | 20 | verified-with-fixes | 4 |
| `MEDIA-11` | Audio output switching without AVSession | AVSessionKit, AbilityKit, ArkTS, ArkUI +5 | - | 20 | verified-with-fixes | 4 |
| `MEDIA-12` | Video playlist linkage | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 | - | 20 | verified-with-fixes | 3 |
| `MEDIA-13` | Import local music files | AbilityKit, ArkData, ArkUI, AudioKit +5 | FILE_ACCESS_PERSIST | 20 | verified-with-fixes | 7 |
| `MEDIA-14` | Video segment to GIF | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +4 | - | 20 | verified-with-fixes | 6 |
| `MEDIA-15` | Bullet comments (danmaku) | AbilityKit, ArkTS, ArkUI, CoreFileKit +1 | - | 20 | verified-with-fixes | 3 |
| `MEDIA-16` | Short-video comment sheet | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 | - | 20 | verified-with-fixes | 3 |
| `MEDIA-17` | Playback speed control | AbilityKit, ArkUI, BasicServicesKit, PerformanceAnalysisKit | - | 20 | verified-with-fixes | 3 |
| `MEDIA-18` | Auto-playing video feed | AbilityKit, ArkTS, ArkUI, BasicServicesKit +4 | - | 16 | verified-with-fixes | 5 |
| `MEDIA-19` | Moving-photo carousel | AbilityKit, ArkData, ArkUI, BasicServicesKit +3 | READ_IMAGEVIDEO | 20 | verified-with-fixes | 4 |
| `MEDIA-20` | Video concatenation | AbilityKit, ArkData, ArkUI, BasicServicesKit +4 | READ_IMAGEVIDEO | 20 | verified-with-fixes | 10 |
| `MEDIA-21` | Multi-camera video wall | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 | - | 20 | verified-with-fixes | 2 |
| `MEDIA-22` | Video like burst | AbilityKit, ArkUI, CoreFileKit, CryptoArchitectureKit +1 | - | 20 | verified-with-fixes | 3 |
| `MEDIA-23` | Extract the audio track from a picked video | AbilityKit, ArkData, ArkTS, ArkUI +5 | - | 20 | verified-with-fixes | 4 |
| `MEDIA-24` | Auto-pause and resume around an audio interruption | AbilityKit, ArkUI, AudioKit, BasicServicesKit +4 | - | 20 | verified-with-fixes | 4 |
| `MEDIA-25` | Navigation splash page | AbilityKit, ArkData, ArkUI, BasicServicesKit +4 | - | 20 | verified-with-fixes | 9 |
| `MEDIA-26` | Seamless list-to-detail video | AbilityKit, ArkUI, BasicServicesKit, MediaKit +1 | - | 20 | verified-with-fixes | 3 |
| `MEDIA-27` | Scrolling audio waveform | AbilityKit, ArkTS, ArkUI, AudioKit +3 | MICROPHONE | 20 | verified-with-fixes | 4 |
| `MEDIA-28` | Buffered progress bar | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 | INTERNET | 20 | verified-with-fixes | 6 |
| `MEDIA-29` | Speed lock | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 | - | 20 | verified-with-fixes | 2 |
| `MEDIA-30` | Download a track over HTTP and play it through a native OHAudio renderer | AbilityKit, ArkUI, AudioKit, BasicServicesKit +2 | INTERNET | 20 | verified-with-fixes | 9 |
| `MEDIA-31` | Drum pad | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 | VIBRATE | 20 | verified-with-fixes | 4 |
| `MEDIA-32` | Video screenshot with frame stepping | AVSessionKit, AbilityKit, ArkUI, BackgroundTasksKit +4 | INTERNET | 20 | verified-with-fixes | 8 |
| `MEDIA-33` | Mirror video playback | AVSessionKit, AbilityKit, ArkUI, BackgroundTasksKit +3 | KEEP_BACKGROUND_RUNNING | 20 | verified-with-fixes | 5 |
| `MEDIA-34` | Background m3u8 download | AbilityKit, ArkTS, ArkUI, BackgroundTasksKit +6 | INTERNET, KEEP_BACKGROUND_RUNNING | 20 | verified-with-fixes | 6 |
| `MEDIA-35` | PCM audio editing | AbilityKit, ArkTS, ArkUI, AudioKit +2 | MICROPHONE | 20 | verified-with-fixes | 5 |
| `MEDIA-36` | Native PCM transcode | AbilityKit, ArkTS, ArkUI, AudioKit +4 | MICROPHONE | 20 | verified-with-fixes | 5 |
| `MEDIA-37` | Offscreen video render | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 | - | 20 | verified-with-fixes | 5 |
| `MEDIA-38` | Metronome | AbilityKit, ArkTS, ArkUI, AudioKit +4 | - | 20 | verified-with-fixes | 5 |
| `MEDIA-39` | Scrub preview | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 | - | 20 | verified-with-fixes | 3 |
| `MEDIA-40` | Image + text compositing | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +2 | - | 17 | verified-with-fixes | 7 |
| `MEDIA-41` | Automatic video subtitles | AbilityKit, ArkTS, ArkUI, BasicServicesKit +4 | - | 17 | verified-with-fixes | 6 |
| `MEDIA-42` | Pause on headset disconnect | AVSessionKit, AbilityKit, ArkUI, CoreFileKit +3 | - | 20 | verified-with-fixes | 7 |
| `MEDIA-43` | Industry FAQ page | - | - | n/a | verified | 3 |

Read the full card at `features/<ID>.md` before writing any code. `verified-with-fixes` means the published HQ document contains at least one defect that the card corrects.
