---
id: SOCIAL-34
title: Nine-grid drag sorting - reorder post images with Grid editMode and onItemDrop
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/34_drag_image_sort.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/drag_image_sort-0000002333934506
sample: huawei_industry_tree/14_social_communication/downloads/SwitchingOrderOfImages.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [Grid, GridItem, editMode, onItemDragStart, onItemDrop, ItemDragInfo, "photoAccessHelper.PhotoViewPicker", "photoAccessHelper.getPhotoAccessHelper", getAssets, getThumbnail, "image.createImagePacker", packToData, "fileIo.openSync", "fileIo.writeSync", "fileIo.closeSync", NavPathStack, "@Link"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-14-0073, HW-14-0074, HW-14-0003, HW-14-0087]
status: verified-with-fixes
---

## When to use

Load this card for the **nine-grid image picker in a post composer**: tap a
dashed tile to pick photos from the gallery, then drag any thumbnail onto
another to swap the two. It is the standard editor for social posts, and the
same mechanics cover reordering an album, a playlist, or any small fixed grid
of items the user arranges by hand.

Everything hangs off three `Grid` members: `editMode(true)` permits dragging a
`GridItem`, `onItemDragStart` returns the floating preview, and `onItemDrop`
delivers the source and destination indices. There is no gesture code, no
`DragEvent`, and no manual hit testing - ArkUI's grid drag sorting does it.

The pattern generalises with one caveat that this sample gets wrong and you
should not: **a grid whose last cell is an "add" button is not a uniformly
draggable grid.** The add tile lives inside the same `Grid`, so it is
drag-eligible and drop-eligible, and neither the shipped guard nor the
document's guard covers dragging *from* it (`HW-14-0074`).

## Feature checklist

- An edit page with an avatar-free composer: a title row with a 发布 (post)
  button, a text area, and the image grid below.
- The grid holds up to nine thumbnails plus a trailing dashed "add" tile whose
  label reads 添加图片 when empty and 继续添加 once at least one image is in.
- Tapping the add tile opens the system photo picker (up to nine at a time).
- Picked images appear as thumbnails and are also written into the app's
  distributed files directory as JPEG.
- Long-pressing a thumbnail lifts a floating copy of it; dropping it on another
  thumbnail exchanges the two positions.
- Dropping outside the grid leaves the order unchanged.
- 发布 pushes a read-only preview page rendering the same list in the same
  order, with dragging disabled.

## Architecture

One `entry` module. The composer and the preview are two pages that share the
image array; the grid exists twice, once editable and once not.

```
entry/src/main/ets
├── components
│   ├── AddPic.ets           the editable grid + the picker + the file write (197 lines)
│   ├── ImageInfo.ets        the same grid, read-only, on the preview page
│   ├── MainTextArea.ets     the post body input
│   ├── InteractiveInfo.ets  like/comment row on the preview
│   └── UserInfo.ets         author header on the preview
├── constants
│   ├── CommonConstants.ets  MAX_ADD_PIC = 9, percentages, and dead permission constants
│   └── StyleConstants.ets   aspect ratios and layout numbers
├── entryability/EntryAbility.ets
├── entrybackupability/
├── model/ParamRes.ets       the route parameter: imageList + content
├── pages
│   ├── EditPage.ets         @Entry, owns @State imageList
│   └── ReleasePage.ets      NavDestination preview, reads ParamRes in onReady
├── utils/FileUtil.ets       fileSelect(): the PhotoViewPicker wrapper
└── viewmodel/ImageData.ets  static preview data
```

The documented tree matches the zip file for file.

**The design decision worth copying** is that `AddPic` takes the array by
`@Link`, not by value:

```typescript
@Link imageList: (PixelMap | Resource | null)[]
```

`EditPage` owns `@State imageList` and passes `AddPic({ imageList: this.imageList })`.
Every reorder and every append mutates the page's array in place, so the 发布
button can hand `new ParamRes(this.imageList, this.textContent)` to the preview
page without any synchronisation step. `ImageInfo` on the preview then takes the
same `@Link` and renders an identical `Grid` with `editMode` absent and
`draggable(this.notEdited)` - one component shape, two permission levels.

**The decision worth avoiding** is putting the add tile inside the sortable
`Grid`. `addDefaultPic()` is a `@Builder` emitting a `GridItem` after the
`ForEach`, which makes it item number `imageList.length` in the grid's own
index space - a position the data array does not have. Every drop handler then
has to defend against an index that is out of bounds by construction. Keeping
the add button outside the `Grid`, or making the grid's item count include a
sentinel entry in the array itself, removes the whole class of bug.

## Implementation steps

1. **Hold the images in the page and link them down.** `@State imageList:
   (PixelMap | Resource | null)[]` in `EditPage`, `@Link` in both grids.
2. **Pick with `PhotoViewPicker`,** not with a permission. `photoPicker.select`
   returns URIs the app may read without `ohos.permission.READ_IMAGEVIDEO`;
   `maxSelectNumber: 9` caps one round of picking.
3. **Resolve each URI to a thumbnail** through
   `photoAccessHelper.getPhotoAccessHelper(context)`, a
   `dataSharePredicates.equalTo('uri', uri)` fetch, and
   `asset.getThumbnail(...)`.
4. **Append the thumbnails in the order the user picked them** - await each URI
   in turn rather than starting all of the callbacks at once (`HW-14-0074`).
5. **Guard the file handle in `finally`.** `fs.closeSync(file.fd)` on a
   `null!`-initialised `fs.File` throws a `TypeError` whenever `openSync` failed,
   and that rejection is what silently loses the picked image (`HW-14-0073`).
6. **Turn on `editMode(true)` and set `draggable(false)` on the images.**
   The two are not contradictory: `draggable` governs ArkUI's generic
   drag-and-drop on the `Image`, which must be off so the grid's own item
   dragging wins.
7. **Return the drag preview from `onItemDragStart`** by returning a `@Builder`
   that renders `this.imageList[item]` at the tile's size.
8. **Validate both indices in `onItemDrop`,** not just `insertIndex`: reject
   `!isSuccess` (dropped outside), `insertIndex >= length` (dropped on the add
   tile) and `itemIndex >= length` (dragged *from* the add tile) - the last one
   is what corrupts the array today (`HW-14-0074`).
9. **Swap, do not splice.** The documented behaviour is exchange, so
   `changeIndex` writes each element into the other's slot. A move-and-shift
   implementation would be a different feature.
10. **Delete the dead `REQUEST_PERMISSIONS` location constants** from
    `CommonConstants` - nothing in an image-sorting demo needs location
    (`HW-14-0003`).

## Verified snippets

All snippets are from `SwitchingOrderOfImages.zip`. Corrected forms are marked.

**The sortable grid — `entry/src/main/ets/components/AddPic.ets`** (corrected, see `HW-14-0074`)

```typescript
Grid(this.scroller) {
  ForEach(this.imageList, (item: PixelMap | Resource | null, index: number) => {
    GridItem() {
      Image(item)
        .width($r('app.float.index_img_width'))
        .aspectRatio(StyleConstants.INDEX_IMG_ASPECTRATIO)
        .borderRadius($r('app.float.borderRadius_sixteen'))
        .draggable(false)
    }
    .width($r('app.float.index_img_width'))
    .aspectRatio(StyleConstants.INDEX_IMG_ASPECTRATIO)
  }, (item: PixelMap | Resource | null, index: number) => `${index}`)  // FIX: shipped key adds Math.random()
  this.addDefaultPic()
}
.columnsTemplate('1fr 1fr 1fr')
.rowsTemplate('1fr 1fr 1fr')
.editMode(true)                       // 设置Grid是否进入编辑模式 - permits dragging GridItems
.onItemDragStart(((event: ItemDragInfo, index: number) => {
  return this.dragItem(index)         // the floating preview during the drag
}))
.onItemDrop((event: ItemDragInfo, itemIndex: number, insertIndex: number, isSuccess: boolean) => {
  // isSuccess=false: dropped outside the grid; index >= length: the trailing add tile
  if (!isSuccess || insertIndex >= this.imageList.length) {
    return
  }
  if (itemIndex >= this.imageList.length) {   // FIX: absent - dragging FROM the add tile
    return
  }
  this.changeIndex(itemIndex, insertIndex)
})

// 交换数组位置 - exchange, not move
changeIndex(index1: number, index2: number) {
  let temp: (PixelMap | Resource | null);
  temp = this.imageList[index1];
  this.imageList[index1] = this.imageList[index2];
  this.imageList[index2] = temp;
}
```

**`editMode(true)` and `draggable(false)` are the pair that makes this work.**
`editMode` is a `Grid` attribute that enables the container's own item
reordering and is what causes `onItemDragStart` / `onItemDrop` to fire at all.
`draggable(false)` on the child `Image` disables the universal drag-and-drop
that would otherwise start a system drag of the image data and steal the
gesture. The document's snippet ties both to a `notEdited` flag; the shipped
`AddPic` hardcodes them, and the flag only survives in `ImageInfo` on the
preview page where dragging is off.

**Two index spaces meet in `onItemDrop`.** `itemIndex` and `insertIndex` are
positions among the grid's *children*, which number `imageList.length + 1`
because of `addDefaultPic()`. The shipped guard checks only the destination, so
dropping onto the add tile is handled but dragging the add tile onto a photo is
not: `changeIndex(length, target)` writes `imageList[length] = imageList[target]`,
extending the array with a phantom entry, and sets the target slot to
`undefined` - an empty hole in the grid. One extra bounds check closes it.

The shipped key generator is `JSON.stringify(item) + Math.random()`, which
produces a fresh key for every item on every diff pass and defeats `ForEach`
reuse entirely - each re-render rebuilds all nine tiles. `PixelMap` also does
not stringify to anything distinguishing. Keying on the index is honest here:
the array is short, bounded at nine, and reordered in place.

**Picking and appending — same file** (corrected, see `HW-14-0074`)

```typescript
async selectImage(): Promise<void> {
  const uris: Array<string> = await fileSelect();     // FIX: shipped code is .then + forEach
  for (const uri of uris) {                           // sequential: preserves the picked order
    await this.getThumbnail(uri);
  }
}

async getThumbnail(uri: string) {
  let phAccessHelper = photoAccessHelper.getPhotoAccessHelper(this.context);
  let predicates: dataSharePredicates.DataSharePredicates = new dataSharePredicates.DataSharePredicates();
  predicates.equalTo('uri', uri);
  let fetchOption: photoAccessHelper.FetchOptions = { fetchColumns: [], predicates: predicates };
  let fetchResult: photoAccessHelper.FetchResult<photoAccessHelper.PhotoAsset> =
    await phAccessHelper.getAssets(fetchOption);
  let asset: photoAccessHelper.PhotoAsset = await fetchResult.getFirstObject();
  asset.getThumbnail((err, pixelMap) => {
    if (err === undefined) {
      let imageName = asset.displayName.substring(0, (asset.displayName).indexOf('.'));
      this.pixelMapToBuffer(pixelMap, imageName)
        .then(() => { this.imageList.push(pixelMap); })
        .catch((e: BusinessError) => {               // FIX: shipped .then has no .catch
          hilog.error(0x0000, '[AddPic]', `pack/write failed: ${e.code} ${e.message}`);
          this.imageList.push(pixelMap);             // the thumbnail is still valid
        });
    }
  });
}
```

**The URI is not the image.** `PhotoViewPicker.select` hands back media-library
URIs that the app is granted to read, and the thumbnail behind one is reached
only through a `PhotoAccessHelper` fetch keyed on that URI - hence the
`DataSharePredicates.equalTo('uri', uri)` round trip for every single pick. That
is also why the shipped `forEach` is a correctness problem and not just a style
one: it launches N independent fetch-plus-callback chains and each one pushes
whenever it finishes, so the nine-grid of an *ordering* demo starts in whatever
order the media library happened to answer.

The `.catch` matters because of the next snippet: `pixelMapToBuffer` also
performs a file write, and any failure there currently takes the thumbnail down
with it.

**The file write — same file** (corrected, see `HW-14-0073`)

```typescript
writeDistributedFile(buf: ArrayBuffer, displayName: string): void {
  let distributedDir: string = this.context.distributedFilesDir;
  let filePath: string = distributedDir + '/' + displayName;
  let file: fs.File | undefined = undefined;      // FIX: shipped code is `let file: fs.File = null!`
  try {
    file = fs.openSync(filePath, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
    hilog.info(0x0000, '[AddPic]', 'Create file success.');
    fs.writeSync(file.fd, buf);
  } catch (error) {
    let err: BusinessError = error as BusinessError;
    hilog.error(0x0000, '[AddPic]',
      `Failed to openSync / writeSync / closeSync. Code: ${err.code}, message: ${err.message}`);
  } finally {
    if (file) {                                    // FIX: shipped code is bare fs.closeSync(file.fd)
      fs.closeSync(file.fd);
    }
  }
}
```

**`null!` is the whole defect.** The non-null assertion silences the compiler
about a variable that is genuinely null until `openSync` succeeds, and the
`finally` block runs on the failure path too - so an open failure (a name with a
`/` in it, a full disk, a display name whose `indexOf('.')` returned `-1` and
produced an empty file name) is converted from a logged `BusinessError` into a
`TypeError` thrown out of `finally`. That rejects `pixelMapToBuffer`, and with
no `.catch` on the caller the image never reaches `imageList`: the user picks a
photo and nothing appears, with nothing in the log to say why.

Note that the file write is incidental to the sorting feature - it demonstrates
`distributedFilesDir` - but it sits directly in the add-image path, so its
failure mode is the feature's failure mode.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions` at all, and that is
correct: `PhotoViewPicker` is a system picker, so the URIs it returns come with
a temporary read grant and `ohos.permission.READ_IMAGEVIDEO` is not needed.
`"routerMap": "$profile:route_map"` is declared for the `EditPage` →
`ReleasePage` navigation.

The sample nevertheless ships this, in `constants/CommonConstants.ets`:

```typescript
static readonly REQUEST_PERMISSIONS: Array<Permissions> = [
  'ohos.permission.APPROXIMATELY_LOCATION',
  'ohos.permission.LOCATION'
];
```

No code references it. It is template residue from a moments/feed sample that
did tag posts with a location, and `HW-14-0003` records it together with the
same residue in the sibling samples. Leaving it in place invites the next
developer to wire it up and request two location permissions a photo sorter has
no use for; the neighbouring `END_SM_LOCAL_INFO` / `START_MD_LOCAL_INFO`
constants are dead for the same reason.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The grid is fixed at `columnsTemplate('1fr 1fr 1fr')` and
  `rowsTemplate('1fr 1fr 1fr')` - exactly nine cells. With nine images the add
  tile becomes the tenth child and is pushed out of the visible template; the
  `onClick` guard `imageList.length < MAX_ADD_PIC` has an empty `else`, so
  reaching the cap gives the user no feedback at all.
- Images are held as `PixelMap` thumbnails in memory, not as URIs. Nine
  thumbnails are cheap; the same array holding full-resolution decodes would not
  be.
- Drag sorting requires a long press to lift an item; there is no explicit
  "edit" affordance telling the user the grid is rearrangeable.
- `ReleasePage` renders the array read-only - `ImageInfo` has no `editMode`, so
  the order fixed in the composer is final.

## Pitfalls

- **`HW-14-0073`** (B/medium, confirmed): `writeDistributedFile` declares
  `let file: fs.File = null!` and closes `file.fd` in `finally`, so any
  `openSync` failure throws a `TypeError` that rejects `pixelMapToBuffer`; the
  caller's `.then` has no `.catch`, and the picked image silently never appears.
  Fix: guard the close with `if (file)` and add a `.catch`.
- **`HW-14-0074`** (B/medium, probable): two ordering defects. The picker's
  `uri.forEach(...)` starts one asynchronous thumbnail chain per URI and pushes
  in completion order, so a multi-pick lands in random order in a sample about
  order; and `onItemDrop` validates only `insertIndex`, so dragging the trailing
  add tile onto a photo writes `imageList[length]` and leaves the target
  `undefined` - a phantom entry plus an empty slot. Fix: await each URI in turn,
  and reject `itemIndex >= this.imageList.length`.
- **`HW-14-0003`** (D/low, confirmed): systematic copy-pasted permission config -
  this sample carries a `CommonConstants.REQUEST_PERMISSIONS` array of
  `LOCATION` / `APPROXIMATELY_LOCATION` that no code ever passes to
  `requestPermissionsFromUser`. Fix: delete the constant.

## References

- `huawei_industry_tree/14_social_communication/docs/34_drag_image_sort.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/drag_image_sort-0000002333934506
- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` - `editMode`, `onItemDragStart`, `onItemDrop`, `ItemDragInfo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `documentation/harmonyos-references/02_application-framework/ts-container-griditem.md` - `GridItem` and its participation in drag sorting
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-griditem
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-drag-sorting.md` - the drag-sorting attributes and their index semantics
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-drag-sorting
- `documentation/harmonyos-references/02_application-framework/ts-universal-events-drag-drop.md` - `draggable` and why it must be off inside an editable grid
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-events-drag-drop
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` - `PhotoViewPicker.select` and `PhotoSelectOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-fetchresult.md` - `getAssets` / `getFirstObject` / `getThumbnail`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-fetchresult
- `SOCIAL-33` - the sibling sample carrying the other half of `HW-14-0003`
