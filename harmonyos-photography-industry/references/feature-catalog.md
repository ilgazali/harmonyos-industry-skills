# Feature catalog

> Generated from `features/*.md`. Source industry: `18_photography`, 32 features.
> Do not edit by hand; regenerate it in the review repository.

Kit names are shown without the `@kit.` prefix; the full module specifier is in each feature card.

| ID | Feature | Kits | Permissions | Min API | Status | Findings |
|---|---|---|---|---:|---|---:|
| `PHOTO-01` | Photography app architecture | CameraKit, ImageKit, MediaLibraryKit, ArkGraphics2D +9 | CAMERA, MICROPHONE +1 | 20 | verified-with-fixes | 18 |
| `PHOTO-02` | Key scenario index | - | - | 20 | verified | - |
| `PHOTO-03` | Gallery photo filters | ImageKit, ArkGraphics2D, MediaLibraryKit, CoreFileKit +3 | WRITE_IMAGEVIDEO | 20 | verified-with-fixes | 4 |
| `PHOTO-04` | Photo aspect-ratio switch | CameraKit, MediaLibraryKit, AbilityKit, ArkUI +6 | CAMERA, WRITE_IMAGEVIDEO +1 | 20 | verified-with-fixes | 4 |
| `PHOTO-05` | Template photo collage | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 | WRITE_IMAGEVIDEO | 20 | verified-with-fixes | 3 |
| `PHOTO-06` | Camera Kit + AVRecorder video capture | AbilityKit, ArkData, ArkUI, BasicServicesKit +5 | MICROPHONE, CAMERA | 20 | verified-with-fixes | 7 |
| `PHOTO-07` | Custom camera zoom | AbilityKit, ArkUI, BasicServicesKit, CameraKit +4 | CAMERA | 20 | verified-with-fixes | 5 |
| `PHOTO-08` | Subject segmentation cut-out | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +4 | - | 20 | verified-with-fixes | 6 |
| `PHOTO-09` | Quality-tiered image compression | AbilityKit, ArkData, ArkUI, BasicServicesKit +4 | READ_IMAGEVIDEO, WRITE_IMAGEVIDEO | 20 | verified-with-fixes | 6 |
| `PHOTO-10` | Sticker overlay editor | AbilityKit, ArkUI, CoreFileKit, ImageKit +2 | WRITE_IMAGEVIDEO | 20 | verified-with-fixes | 7 |
| `PHOTO-11` | Face detection with a focus matrix | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +4 | - | 20 | verified-with-fixes | 5 |
| `PHOTO-12` | GIF from a picture sequence | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +1 | - | 20 | verified-with-fixes | 5 |
| `PHOTO-13` | Capture timer | AbilityKit, ArkUI, BasicServicesKit, CameraKit +4 | CAMERA | 20 | verified-with-fixes | 5 |
| `PHOTO-14` | Image format conversion | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 | - | 20 | verified-with-fixes | 3 |
| `PHOTO-15` | Video trim timeline | AbilityKit, ArkData, ArkUI, BasicServicesKit +5 | - | 20 | verified-with-fixes | 6 |
| `PHOTO-16` | Colour adjustment | AbilityKit, ArkUI, CoreFileKit, ImageKit +2 | - | 20 | verified-with-fixes | 3 |
| `PHOTO-17` | Video aspect-ratio crop | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 | - | 20 | verified-with-fixes | 6 |
| `PHOTO-18` | Salt-and-pepper denoising | AbilityKit, ArkTS, ArkUI, BasicServicesKit +4 | WRITE_IMAGEVIDEO | 20 | verified-with-fixes | 7 |
| `PHOTO-19` | Picture-in-picture video edit | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +4 | - | 20 | verified-with-fixes | 7 |
| `PHOTO-20` | Rotate and flip a still image | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 | WRITE_IMAGEVIDEO | 20 | verified-with-fixes | 7 |
| `PHOTO-21` | Static video watermark | AbilityKit, ArkData, ArkTS, ArkUI +5 | READ_IMAGEVIDEO, WRITE_IMAGEVIDEO | 20 | verified-with-fixes | 11 |
| `PHOTO-22` | Video format conversion | AbilityKit, ArkUI, BasicServicesKit, MediaKit +2 | - | 20 | verified-with-fixes | 6 |
| `PHOTO-23` | Mosaic brush | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 | WRITE_IMAGEVIDEO | 20 | verified-with-fixes | 5 |
| `PHOTO-24` | Ratio crop frame | AbilityKit, ArkUI, BasicServicesKit, CoreFileKit +3 | WRITE_IMAGEVIDEO | 20 | verified-with-fixes | 7 |
| `PHOTO-25` | Delayed video recording | AbilityKit, ArkData, ArkTS, ArkUI +8 | CAMERA, MICROPHONE +2 | 20 | verified-with-fixes | 8 |
| `PHOTO-26` | Preview resolution switch | AbilityKit, ArkData, ArkGraphics2D, ArkUI +5 | CAMERA, WRITE_IMAGEVIDEO +1 | 20 | verified-with-fixes | 3 |
| `PHOTO-27` | Camera preview across background switches | AbilityKit, ArkGraphics2D, ArkUI, BasicServicesKit +3 | CAMERA | 20 | verified-with-fixes | 6 |
| `PHOTO-28` | Oil-painting and pencil-sketch filters | AbilityKit, ArkTS, ArkUI, CoreFileKit +3 | - | 20 | verified-with-fixes | 6 |
| `PHOTO-29` | Gravity-sensor orientation | AbilityKit, ArkUI, BasicServicesKit, CameraKit +5 | CAMERA, READ_IMAGEVIDEO +1 | 20 | verified-with-fixes | 9 |
| `PHOTO-30` | Keep recording in a picture-in-picture window | AbilityKit, ArkData, ArkTS, ArkUI +7 | CAMERA, MICROPHONE +2 | 20 | verified-with-fixes | 9 |
| `PHOTO-31` | Preview resolution switch | AbilityKit, ArkData, ArkGraphics2D, ArkUI +6 | CAMERA, WRITE_IMAGEVIDEO +1 | 20 | verified-with-fixes | 9 |
| `PHOTO-32` | Industry FAQ page | - | - | n/a | verified | - |

Read the full card at `features/<ID>.md` before writing any code. `verified-with-fixes` means the published HQ document contains at least one defect that the card corrects.
