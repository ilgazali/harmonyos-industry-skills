---
id: PHOTO-02
title: Key scenario index - the 29 photography samples and which card covers each
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/02_practice-photo-app-architecture-v1_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1_1-0000002247958506
sample: none
kits: []
apis: []
permissions: []
min_api: 20
modules: []
findings: []
status: verified
---

## When to use

Load this card when you know **what you want the app to do but not which
sample does it** - "I need to crop a video to a different aspect ratio", "I
need a mosaic brush", "I need the camera to keep recording in a floating
window". The underlying page is Huawei's index of the photography industry's
key scenarios: a bare list of 29 links, no prose, no code.

Its value is as a routing table, and this card turns it into one. `PHOTO-01`
gives you the architecture - how the modules split, how pages in different HARs
reach each other, who owns the camera. Every entry below is one feature of that
architecture extracted into a standalone project you can build and run. The
usual workflow is: find the row here, open that card, copy the pattern back
into the `PHOTO-01` module layout.

**This is a documentation page with no sample of its own.** Nothing on it is
compile-verified, and the only thing that can be wrong with it is a broken or
mis-titled link. All 29 links resolve to pages that exist in this pack.

## Feature checklist

What the page promises, and what was checked:

- A flat list of 29 scenario links under the 拍摄美化 (photography and
  beautification) practice, in no particular order.
- Every link points at an `architecture-guides` page with its own sample zip.
- Each linked page follows the same fixed template: 场景介绍 (scenario), 效果预览
  (preview gif), 实现思路 (approach, with a short snippet), 约束与限制
  (constraints), 工程目录 (project tree), 参考文档, 代码下载.
- The list is not grouped, ordered or annotated - a scenario's kit, difficulty
  and dependency on the others are all left for the reader to work out.
- No permissions, no constraints and no code of its own.

## Architecture

The page has none - it is a navigation node in the architecture-guides tree,
sitting between `PHOTO-01` (the architecture narrative) and the 29 scenario
pages. Its structure is one Markdown list.

What it links to, mapped onto this pack's cards:

| Card | Scenario (doc title) | Sample |
| --- | --- | --- |
| `PHOTO-03` | 图库照片滤镜添加 - gallery photo filters | ImageFilter |
| `PHOTO-04` | 拍照比例自定义 - photo aspect-ratio switch | RatioCamera |
| `PHOTO-05` | 图片拼接 - image stitching | ImageStitch |
| `PHOTO-06` | 基于CameraKit和AVRecorder实现视频录制与播放 | VideoRecording |
| `PHOTO-07` | 相机焦距缩放 - camera zoom | CustomisedCamera |
| `PHOTO-08` | 图片抠图 - subject cutout | SubjectSegmentation |
| `PHOTO-09` | 图片压缩至不同质量 - compress to target quality | CompressImages |
| `PHOTO-10` | 图片添加贴纸并保存 - stickers | PictureSticker |
| `PHOTO-11` | 图片人脸位置识别 - face position detection | FaceDetection |
| `PHOTO-12` | 多张图片转Pixelmap合成GIF动图 - GIF from stills | GIFGenerator |
| `PHOTO-13` | 自定义相机定时拍摄 - self-timer capture | CaptureTimer |
| `PHOTO-14` | 图片格式转换 - format conversion | ImageConverter |
| `PHOTO-15` | 拖动时间轴剪辑视频时长 - timeline trim | VideoClip |
| `PHOTO-16` | 图片色彩调节 - colour adjustment | ImageEffect |
| `PHOTO-17` | 视频画面比例裁剪 - video aspect crop | VideoCropping |
| `PHOTO-18` | 图像降噪处理-椒盐噪声 - salt-and-pepper denoise | ImageDenoising |
| `PHOTO-19` | 视频剪辑-添加画中画 - picture-in-picture edit | VideoEditPiPWindow |
| `PHOTO-20` | 静态图片旋转与翻转 - rotate and flip | ImageRotateAndFlip |
| `PHOTO-21` | 视频静态水印添加 - static video watermark | VideoWatermark |
| `PHOTO-22` | 视频编码转换 - transcoding | VideoEncodingConvert |
| `PHOTO-23` | 图片绘制马赛克并保存 - mosaic brush | ImageDrawMosaic |
| `PHOTO-24` | 图片比例裁剪 - aspect-ratio image crop | CropRect |
| `PHOTO-25` | 自定义相机延时录像 - delayed recording | AVRecorderTimer |
| `PHOTO-26` | 自定义相机分辨率设置 - resolution selection | CameraResolution |
| `PHOTO-27` | 前后台切换相机继续预览 - resume preview on foreground | CustomCamera |
| `PHOTO-28` | 图片添加油画滤镜和铅笔画滤镜 - oil-paint and pencil filters | ImageFilterProcessing |
| `PHOTO-29` | 基于重力传感器实现相机图标和照片拍摄角度跟随设备旋转 | CameraTwist |
| `PHOTO-30` | 自定义相机退后台保持画中画录制 - background PiP recording | PipwindowRecorder |
| `PHOTO-31` | 使用OpenGL标准化设备坐标绘制矩形人脸框 - NDC face box | (see card) |

**The design decision worth noting** is that this list is flat where the
material is not. The 29 scenarios fall into four groups that barely overlap,
and knowing which group you are in tells you which APIs and which failure modes
to expect:

- **Camera pipeline** (`PHOTO-04`, `07`, `13`, `26`, `27`, `29`, and the camera
  half of `06`, `25`, `30`): Camera Kit, `XComponent` surfaces, session
  rebuilds. These share a single copy-pasted `CameraShooter`/`CameraUtils`
  file, and therefore share its defects - `HW-18-0010` (un-awaited teardown)
  hits nine of them, `HW-18-0021` (hardcoded profiles) four.
- **Still-image processing** (`PHOTO-03`, `08`, `09`, `10`, `14`, `16`, `18`,
  `20`, `23`, `24`, `28`): Image Kit plus effectKit or a hand-written pixel
  kernel. These share the picker-in / `SaveButton`-out envelope, and with it
  `HW-18-0024` (unhandled picker cancel, six samples), `HW-18-0005` (unreleased
  `ImageSource`, eight files) and `HW-18-0036` (unreleased `ImagePacker`, eight
  samples).
- **Video** (`PHOTO-15`, `17`, `19`, `21`, `22`): AVPlayer, AVRecorder,
  `AVCodec`, `AVTranscoder`. Longest-running and least forgiving of missing
  teardown.
- **Graphics** (`PHOTO-31`): OpenGL through a native XComponent - the only
  scenario that leaves ArkTS.

If you are picking a starting point, start in the group, not the list: the
second sample you read from the same group will be 80% familiar, and the same
finding will usually apply to it.

## Implementation steps

There is nothing to implement from this page. The steps are how to use it:

1. **Read `PHOTO-01` first** if the goal is an app rather than a feature. It
   defines the module split, the `NavPathStack` + routing-table wiring and the
   camera ownership model that every scenario below assumes.
2. **Find the scenario** in the table above and open that card, not the Huawei
   page - the cards carry the review findings and the corrected snippets.
3. **Check the group** the scenario belongs to and read its shared systematic
   findings before writing code. They are the defects you would otherwise
   inherit by copying.
4. **Expect the template.** Every scenario page has the same seven sections;
   the useful ones are 实现思路 (a partial snippet) and 工程目录 (the file
   tree). Five of the trees do not match their zips (`HW-18-0001`), so trust
   the zip.
5. **Fold the feature back into the `PHOTO-01` layout**: a scenario sample is a
   single-`entry` project, and moving it into a feature HAR means adding a
   barrel `Index.ets`, a `route_map.json`, and a `@Builder` entry point.

## Verified snippets

There is no sample. The page's entire content is a link list; this is its shape
(from the doc — no sample shipped; not compile-verified):

```markdown
- **[图库照片滤镜添加](https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_filter-0000002284048625)**
- **[拍照比例自定义](https://developer.huawei.com/consumer/cn/doc/architecture-guides/ratio_camera-0000002252528422)**
- **[图片拼接](https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_stitch-0000002287473193)**
...
- **[自定义相机退后台保持画中画录制](https://developer.huawei.com/consumer/cn/doc/architecture-guides/pipwindow_recorder-0000002522526388)**
- **[使用OpenGL标准化设备坐标绘制矩形人脸框](https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio-v1_2-ts_50-0000002407536358)**
```

Two things are worth reading off it. First, the **URL slugs are the reliable
identifier**, not the titles: `image_filter`, `ratio_camera`, `capture_timer`
name the sample zip directly, and this pack's doc filenames follow them. Second,
three slugs do not: `insurance-v1_2-ts_32`, `photo-v1_2-ts_12` and
`audio-v1_2-ts_50` are recycled ids from unrelated industries, so those three
scenarios cannot be identified from the URL at all - `insurance-v1_2-ts_32` is
the camera resolution sample and `audio-v1_2-ts_50` is the OpenGL face box.
Match those by title.

## Pitfalls

- **The list is the only place these 29 scenarios are enumerated**, and it is
  unordered and ungrouped. Do not read the order as a difficulty or dependency
  sequence; there is none.
- **Three link slugs are recycled from other industries**
  (`insurance-v1_2-ts_32`, `photo-v1_2-ts_12`, `audio-v1_2-ts_50`), so a slug
  is not a reliable scenario name for those three. Match by title.
- **`HW-18-0001`** (E/low, confirmed) applies to five of the linked pages: their
  工程目录 blocks were not regenerated after the samples changed, listing files
  that are missing, renamed, or carry a different extension. Navigate by the
  zip, not the tree.
- No defect was found in this index page itself: all 29 links resolve, and each
  title matches the page it points at.

## References

- `huawei_industry_tree/18_photography/docs/02_practice-photo-app-architecture-v1_1.md` - the page itself
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-photo-app-architecture-v1_1-0000002247958506
- `PHOTO-01` - the architecture these 29 scenarios are features of
- `huawei_industry_tree/18_photography/docs/index.json` - the machine-readable form of the same list, with local paths
