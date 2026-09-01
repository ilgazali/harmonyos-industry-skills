---
id: MEDIA-03
title: Video metadata card - pick a video with PhotoViewPicker, then read size, duration and thumbnail
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/03_get_video_info.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/get_video_info-0000002273254857
sample: huawei_industry_tree/13_media_entertainment/downloads/PageMediaMeta.zip
kits: ["@kit.MediaLibraryKit", "@kit.ImageKit", "@kit.ArkUI", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["photoAccessHelper.PhotoViewPicker", PhotoSelectOptions, PhotoViewMIMETypes, "photoAccessHelper.getPhotoAccessHelper", getAssets, FetchOptions, PhotoKeys, "FetchResult.getFirstObject", "PhotoAsset.get", "PhotoAsset.getThumbnail", "dataSharePredicates.DataSharePredicates", "image.PixelMap", Grid, GridItem, "window.getWindowAvoidArea", "@StorageLink", expandSafeArea]
permissions: ["ohos.permission.READ_IMAGEVIDEO"]
min_api: 20
modules: [entry]
findings: [HW-13-0002, HW-13-0013, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when the user picks a video (or an image) from the gallery and
your UI has to **show something about it before playing it** - a card with a
thumbnail, a duration badge, a file size and a name. Upload flows, share
sheets, editors and "recent clips" grids all start here.

The pattern is two steps: `PhotoViewPicker` returns a temporary read URI for
the chosen item, then the metadata is read from that URI. The second step is
where the design decision is, and this sample makes the expensive choice: it
resolves the URI back into a `PhotoAsset` through `photoAccessHelper.getAssets`,
which needs `ohos.permission.READ_IMAGEVIDEO` - a restricted, user-grant
permission - to return anything.

**Read `HW-13-0002` before adopting it.** The permission is declared and never
requested, so the query returns nothing and the grid stays empty. Either
request it properly, or - better for most apps - stay on the picker URI, which
grants file access with **no permission at all**. The picker guide is explicit
that it "allows an application to access the image/video selected by the user
without the permission for reading images/videos".

## Feature checklist

- A title bar with a 加 (add) button on the right.
- Tapping it opens the system photo picker filtered to videos, one item
  maximum.
- On selection, the app reads the chosen video's size, duration, display name
  and a thumbnail sized to the video's own resolution.
- The result is appended to a two-column grid; each cell shows the thumbnail,
  a 视频 (video) tag, the duration top-right, the file size bottom-left and
  the file name underneath.
- Repeated picks accumulate in the grid rather than replacing it.
- Duration renders as `HH:MM:SS`; size renders as `B` / `KB` / `MB`.
- Portrait videos (`ORIENTATION` 90 or 270) get their thumbnail dimensions
  swapped so the aspect ratio is right.
- The page draws under the status bar and the navigation indicator, with the
  title bar pushed down by the measured avoid area.

## Architecture

One `entry` module, three ArkTS files. Everything - picking, querying,
formatting and layout - is in the one page component.

```
entry/src/main/ets
├── entryability/EntryAbility.ets          full-screen window, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets   template stub
└── pages/GetVideoData.ets                 the whole feature, 272 lines
```

The documented tree matches the zip exactly (unlike the three media docs in
`HW-13-0003`). `resources/` holds the strings, the add icon and the layered
launcher icon.

**The design decision worth avoiding** is the double `getAssets`. The page
queries the media library twice for the same item: once in `uriGetAssets` for
`SIZE`, `DURATION`, `WIDTH`, `HEIGHT` and `TITLE`, and again inside
`getThumbnailByUrl` for `WIDTH`, `HEIGHT` and `ORIENTATION`, each building its
own `DataSharePredicates` on the same URI. One fetch with the union of the
columns would serve both, and passing the already-fetched `PhotoAsset` into
the thumbnail helper - instead of passing the `PhotoAccessHelper` and
re-querying - removes the second round trip entirely.

The larger point is that neither query is needed for most of what the card
shows. `fs.statSync` on the picker URI gives the size, and
`media.AVImageGenerator` gives a frame, both without touching the media
library and without `READ_IMAGEVIDEO` - that is exactly the route `MEDIA-04`
takes on the same kind of URI. Only `displayName` and `DURATION` genuinely
come from the library, and duration is available from
`media.AVMetadataExtractor` too.

## Implementation steps

1. **Build `PhotoSelectOptions`** with `MIMEType = PhotoViewMIMETypes.VIDEO_TYPE`
   and `maxSelectNumber = 1`, and call `select` on a `PhotoViewPicker`.
   Nothing so far needs a permission.
2. **Keep the returned `photoUris`** in state; they are the app's handle on
   the file for the rest of the session.
3. **Decide the metadata route before writing anything else.** File-API route:
   no permission, `fs` + `AVImageGenerator`/`AVMetadataExtractor`. Media-library
   route: `getAssets`, and then `READ_IMAGEVIDEO` must be *requested* at
   runtime, not merely declared (`HW-13-0002`).
4. **If you query, query once.** Put every column both consumers need in one
   `fetchColumns` array and pass the resulting `PhotoAsset` down.
5. **Include `SIZE` and `DURATION` in `fetchColumns`** - `PhotoAsset.get`
   throws for a column that was not fetched, which is why the sample's own
   comment shouts 重要 (important) at that line.
6. **Size the thumbnail from the asset's own `WIDTH`/`HEIGHT`,** swapped when
   `ORIENTATION` is 90 or 270.
7. **Append to the grid immutably** (`[...this.thumbnails, item]`) so the
   `@State` array change is observed.
8. **Keep the byte formatter's threshold and divisor consistent**
   (`HW-13-0013`).
9. **Read the avoid areas in the ability**, publish them through `AppStorage`,
   and pad the page - and unregister `avoidAreaChange` when the window goes
   away, which this sample does not do.

## Verified snippets

All snippets are from `PageMediaMeta.zip`. Corrected forms are marked.

**Picking the video - `entry/src/main/ets/pages/GetVideoData.ets`** (as shipped)

```typescript
@State uris: Array<string> = [];
context = this.getUIContext().getHostContext();

// 调用PhotoViewPicker.select选择视频
async photoPickerGetUri() {
  try {
    let photoSelectOptions = new photoAccessHelper.PhotoSelectOptions; //选择设置
    photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.VIDEO_TYPE; //视频类型
    photoSelectOptions.maxSelectNumber = 1; //最大选择数量
    let photoPicker = new photoAccessHelper.PhotoViewPicker();
    //选择过程
    photoPicker.select(photoSelectOptions).then((PhotoSelectResult: photoAccessHelper.PhotoSelectResult) => {
      this.uris = PhotoSelectResult.photoUris; //资源uris
      //根据资源查询对应的信息
      this.uriGetAssets();
    }).catch((err: BusinessError) => {
      hilog.error(-1, 'PhotoViewPicker.select failed with err: ', JSON.stringify(err));
    });
  } catch (error) {
    let err: BusinessError = error as BusinessError;
    hilog.error(-1, 'PhotoViewPicker failed with err: ', JSON.stringify(err));
  }
}
```

**Two options carry the design.** `MIMEType` filters the picker itself, so the
user cannot select a photo and reach code that assumes a video;
`maxSelectNumber = 1` keeps `photoUris[0]` a safe indexing. The picker runs in
its own process and returns URIs the app is temporarily authorised to read -
that authorisation is granted by the *selection*, not by a permission, which
is the whole point of the component.

Note `this.getUIContext().getHostContext()` at the top: the correct way to
reach the ability context from a page. `HW-13-0032` records four sibling
samples that write `new UIContext()` here instead and get a detached instance
whose `getHostContext()` returns nothing usable.

**Reading the metadata - same file** (corrected, see `HW-13-0002`)

```typescript
async uriGetAssets() {
  try {
    // FIX: getAssets requires ohos.permission.READ_IMAGEVIDEO, which the sample
    // declares in module.json5 but never requests. Either request it here first:
    //   const atManager = abilityAccessCtrl.createAtManager();
    //   await atManager.requestPermissionsFromUser(
    //     this.context as Context, ['ohos.permission.READ_IMAGEVIDEO' as Permissions]);
    // or drop this query and read the picker URI directly with fs/AVImageGenerator.
    let phAccessHelper = photoAccessHelper.getPhotoAccessHelper(this.context);
    let predicates: dataSharePredicates.DataSharePredicates = new dataSharePredicates.DataSharePredicates();
    // 配置查询条件，使用PhotoViewPicker选择图片返回的uri进行查询
    predicates.equalTo('uri', this.uris[0]);
    let fetchOption: photoAccessHelper.FetchOptions = {
      fetchColumns: [photoAccessHelper.PhotoKeys.WIDTH, photoAccessHelper.PhotoKeys.HEIGHT,
        photoAccessHelper.PhotoKeys.TITLE, photoAccessHelper.PhotoKeys.SIZE, photoAccessHelper.PhotoKeys.DURATION],
      predicates: predicates
    };

    let fetchResult: photoAccessHelper.FetchResult<photoAccessHelper.PhotoAsset> =
      await phAccessHelper.getAssets(fetchOption);

    // 得到uri对应的PhotoAsset对象，读取文件的部分信息
    const asset: photoAccessHelper.PhotoAsset = await fetchResult.getFirstObject();
    // 必须确保fetchOptions中包含SIZE和DURATION字段（重要！）
    const rawSize = asset.get(photoAccessHelper.PhotoKeys.SIZE);       // 单位：字节
    const rawDuration = asset.get(photoAccessHelper.PhotoKeys.DURATION); // 单位：毫秒
    this.videoSize = this.formatFileSize(Number(rawSize));
    this.videoTime = this.formatDuration(Number(rawDuration));
    this.audioMap = await this.getThumbnailByUrl(phAccessHelper);
    this.name = asset.displayName;

    if (this.audioMap !== null) {
      let thumbNail: ThumbnailItem = {
        pixelMap: this.audioMap, duration: this.videoTime, size: this.videoSize, name: this.name
      };
      this.thumbnails = [...this.thumbnails, thumbNail];
    }
  } catch (error) {
    hilog.error(0x0000, 'uriGetAssets failed with err: ', JSON.stringify(error));
  }
}
```

**A picker URI is a file handle, not a media-library key - and this function
treats it as both.** `predicates.equalTo('uri', this.uris[0])` asks the media
library for the asset whose URI matches, which is a legitimate lookup, but it
is gated by `READ_IMAGEVIDEO` and the sample never asks the user for it. The
`try/catch` then swallows the failure into a log line, so the symptom is not
an error dialog but a grid that stays empty after every pick. The same defect
class is confirmed in five other samples across four industries, which is why
it is worth naming as a rule: *a declared user-grant permission is not a
granted one.*

The guarded append is the other detail worth keeping. `getThumbnail` can
return `null`, and building the `ThumbnailItem` only when it did not keeps a
half-populated card out of the grid entirely - the cell has no fallback image.

**The thumbnail, sized from the asset - same file** (as shipped)

```typescript
async getThumbnailByUrl(phAccessHelper: photoAccessHelper.PhotoAccessHelper): Promise<PixelMap | null> {
  try {
    let predicates: dataSharePredicates.DataSharePredicates = new dataSharePredicates.DataSharePredicates();
    predicates.equalTo('uri', this.uris[0]);
    let videoFetchResult: photoAccessHelper.FetchResult<photoAccessHelper.PhotoAsset> =
      await phAccessHelper.getAssets({
        fetchColumns: ['width', 'height', 'orientation'],
        predicates: predicates
      });
    let asset: photoAccessHelper.PhotoAsset = await videoFetchResult.getFirstObject();

    // 配置缩略图参数
    let thumbnailSize: Size = { width: 0, height: 0 };
    if (asset.get(photoAccessHelper.PhotoKeys.ORIENTATION) === 90 ||
      asset.get(photoAccessHelper.PhotoKeys.ORIENTATION) === 270) {
      thumbnailSize.width = asset.get(photoAccessHelper.PhotoKeys.HEIGHT) as number;
      thumbnailSize.height = asset.get(photoAccessHelper.PhotoKeys.WIDTH) as number;
    } else {
      thumbnailSize.width = asset.get(photoAccessHelper.PhotoKeys.WIDTH) as number;
      thumbnailSize.height = asset.get(photoAccessHelper.PhotoKeys.HEIGHT) as number;
    }
    return asset.getThumbnail(thumbnailSize);
  } catch (error) {
    hilog.error(0x0000, '', `get acquireThumbnail failed, error: ${JSON.stringify(error)}`);
    return null;
  }
}
```

**`ORIENTATION` is stored separately from `WIDTH`/`HEIGHT`, and the swap is
the part everyone forgets.** A phone-shot portrait clip is commonly recorded
as a landscape frame plus a 90-degree rotation flag; asking for a thumbnail at
the stored width and height then yields a sideways image squeezed into the
wrong box. Swapping the two for 90 and 270 is the correct and complete fix -
0 and 180 need no change.

Note that `fetchColumns` here uses the raw strings `'width'`, `'height'`,
`'orientation'` while the sibling query uses the `PhotoKeys` enum. They are
the same values; mixing the two forms in one file is the kind of drift worth
cleaning up before the code is copied.

**The size formatter - same file** (corrected, see `HW-13-0013`)

```typescript
// 文件大小格式化（支持B/KB/MB自动转换）
private formatFileSize(bytes: number): string {
  const units = ['B', 'KB', 'MB'];
  let size = bytes;
  let unitIndex = 0;

  while (size >= 1024 && unitIndex < units.length - 1) {   // FIX: shipped threshold is 1000
    size /= 1024;
    unitIndex++;
  }
  return `${size.toFixed(2)}${units[unitIndex]}`;
}
```

**A decimal threshold with a binary divisor produces values below 1 in the
larger unit.** The shipped loop enters at 1000 bytes but divides by 1024, so a
1000-byte file renders as `0.98KB` and every unit boundary is off by 2.4%.
Pick one system and use it for both numbers: 1024/1024 for the KiB-style
labels this code uses, or 1000/1000 if the labels are meant to be decimal.
`MEDIA-04`'s `fileSizeConversion` gets this right with explicit
`1024`-multiple constants and is the better model.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.READ_IMAGEVIDEO",
    "reason": "$string:reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
]
```

- `READ_IMAGEVIDEO` is `user_grant` **and restricted**: declaring it is not
  enough, the user must grant it through `requestPermissionsFromUser`, and on
  a released app the restricted class also needs an ACL entry in the signing
  profile. Nothing in this sample requests it (`HW-13-0002`).
- Everything up to and including `photoPicker.select` works without it. Only
  `getAssets` / `getThumbnail` are gated.
- `reason` and `usedScene` are correctly filled in, and `when: "inuse"` is the
  right choice - the app reads gallery data only while in the foreground.
- No `INTERNET`, no other declarations; the sample is entirely local.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`
  and matches the document's 约束与限制 - the three docs in `HW-13-0004` do
  not.
- `deviceTypes`: `phone`, `tablet`, `2in1`. The grid is fixed at
  `columnsTemplate('1fr 1fr')` with 170vp cells, so a 2in1 window shows two
  very wide columns; make the template width-dependent for real use.
- `caseSensitiveCheck` and `useNormalizedOHMUrl` are both on in
  `build-profile.json5`.
- The colour mode is forced to light in `EntryAbility.onCreate` (and again in
  `onWindowStageCreate`); the hardcoded `#F1F3F5` background and white cards
  assume it.
- The thumbnail is requested at the video's full resolution, which is
  wasteful for a 170vp cell - `getThumbnail()` with no argument returns the
  system default size and is cheaper.
- `windowClass.on('avoidAreaChange', ...)` is registered in
  `onWindowStageCreate` and never released in `onWindowStageDestroy`, the same
  boilerplate omission seen across these samples.
- There is no video playback here at all - this card ends at the metadata. For
  a picked-video player see `MEDIA-04`, which reuses the same picker.

## Pitfalls

- **`HW-13-0002` (B/medium, confirmed) - `getAssets` is called with
  `READ_IMAGEVIDEO` declared but never requested.** Both queries fail, the
  catch logs and returns, and the grid never fills. Fix: request the
  permission at runtime, or drop the media-library lookup and read the
  picker-returned URI directly with `fs` and `AVImageGenerator`, which needs
  no permission. The same defect is confirmed in `20_merge_video`,
  `19_dynamic_photo` and three samples in other industries.
- **`HW-13-0013` (B/low, confirmed) - `formatFileSize` mixes a decimal
  threshold (`>= 1000`) with a binary divisor (`/1024`).** A 1000-byte file
  displays as `0.98KB`. Fix: match threshold and divisor.
- **The media library is queried twice per pick** for overlapping columns, and
  `getThumbnailByUrl` receives the helper rather than the asset it needs.
  Fetch once with the union of the columns and pass the `PhotoAsset` down.
- **`avoidAreaChange` is never unregistered,** and
  `setWindowLayoutFullScreen`'s promise is handled but the listener outlives
  the window stage.
- Related systematics that this sample avoids: `HW-13-0032` (`new UIContext()` -
  this page correctly uses `this.getUIContext().getHostContext()`) and
  `HW-13-0003` (documented tree disagreeing with the zip - this one matches).

## References

- `huawei_industry_tree/13_media_entertainment/docs/03_get_video_info.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/get_video_info-0000002273254857
- `documentation/harmonyos-guides/05_media/component-guidelines-photoviewpicker.md` - the picker component and its no-permission guarantee
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/component-guidelines-photoviewpicker
- `documentation/harmonyos-guides/05_media/photoaccesshelper-photoviewpicker.md` - `PhotoSelectOptions`, `PhotoSelectResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-photoviewpicker
- `documentation/harmonyos-guides/05_media/photoaccesshelper-resource-guidelines.md` - fetching image and video thumbnails
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-resource-guidelines
- `documentation/harmonyos-guides/05_media/photoaccesshelper-preparation.md` - which photoAccessHelper calls need which permission
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-preparation
- `documentation/harmonyos-guides/05_media/medialibrary-request-photouris-permission.md` - obtaining access to a picked URI
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/medialibrary-request-photouris-permission
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoasset.md` - `get`, `getThumbnail`, `displayName`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoasset
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-fetchresult.md` - `getFirstObject`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-fetchresult
- `MEDIA-04` - the permission-free route to the same data on a picked video URI
